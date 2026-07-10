# WorkflowTemplate 修改中的两个常见问题

本文说明两个问题：

1. 为什么 `{{workflow.failures}}` 直接放进 JSON body 里可能导致回调失败。
2. 为什么直接 `kubectl apply` 时建议清理 `metadata.uid`、`resourceVersion`、`generation`、`creationTimestamp`、`managedFields` 等运行时字段。

## 1. `workflow.failures` 为什么可能把 JSON 弄坏

当前 WorkflowTemplate 回调 body 类似这样：

```yaml
body: |
  {
    "workflowName": "{{workflow.name}}",
    "status": "{{workflow.status}}",
    "message": "{{workflow.failures}}"
  }
```

Argo 发送 HTTP 请求前，会先把 `{{workflow.name}}`、`{{workflow.status}}`、`{{workflow.failures}}` 这些变量替换成真实值。

如果变量值很简单，例如：

```text
{{workflow.name}} = data-process-123
{{workflow.status}} = Failed
{{workflow.failures}} = container failed
```

替换后是合法 JSON：

```json
{
  "workflowName": "data-process-123",
  "status": "Failed",
  "message": "container failed"
}
```

但是如果 `workflow.failures` 里包含双引号，例如：

```text
step "data-parse" failed
```

替换后会变成：

```json
{
  "workflowName": "data-process-123",
  "status": "Failed",
  "message": "step "data-parse" failed"
}
```

这里的 `"data-parse"` 会提前结束 `message` 字符串，导致整个 JSON 非法。后端收到后可能在 JSON 解析阶段直接失败，Controller 方法都进不去。

换行也有类似问题。比如失败信息是多行：

```text
step data-parse failed
exit code 1
```

替换后可能变成：

```json
{
  "message": "step data-parse failed
exit code 1"
}
```

普通 JSON 字符串里不能直接放未转义换行，也可能解析失败。

复杂 JSON 更明显。比如 `workflow.failures` 渲染出来是：

```json
[{"displayName":"data-parse","message":"failed"}]
```

如果外面又套了一层字符串：

```json
"message": "[{"displayName":"data-parse","message":"failed"}]"
```

里面的双引号会破坏外层 JSON。

所以问题本质是：把一个内容不确定的变量直接塞进 JSON 字符串里，但没有做 JSON 转义。

## 2. 怎么解决 `workflow.failures` 的 JSON 风险

### 方案一：回调只传安全字段，失败详情由后端主动查 Argo

推荐把 body 改成：

```yaml
body: |
  {
    "workflowName": "{{workflow.name}}",
    "status": "{{workflow.status}}"
  }
```

后端收到失败状态后，再通过 Argo API 查询这个 Workflow 的详情，拿 `status.message`、`status.nodes` 或 failures 信息。

优点是回调请求体最稳定，不容易因为失败信息格式导致回调失败。

### 方案二：`message` 只传可控的简单字符串

也可以写成：

```yaml
body: |
  {
    "workflowName": "{{workflow.name}}",
    "status": "{{workflow.status}}",
    "message": "workflow finished with status {{workflow.status}}"
  }
```

这里的 `message` 是可控字符串，不直接塞复杂失败详情，生成非法 JSON 的概率更低。

### 方案三：确认当前 Argo 版本是否支持 JSON 转义函数

有些模板引擎支持类似 `toJson`、`jsonpath`、表达式转义等能力。如果当前 Argo 版本支持把变量安全编码成 JSON 字符串，可以使用它。

这个方案要以实际 Argo 集群版本为准，不能直接假设可用。

### 方案四：后端接口做宽容接收

后端可以不强依赖 `message`，即使 `message` 为空也能更新状态。

但要注意：如果整个 body 已经不是合法 JSON，Spring Controller 连 DTO 都解析不了，这种宽容处理也帮不上忙。

所以更推荐从 Argo body 侧避免生成非法 JSON。

## 3. 推荐当前改法

对稳定性和新手实现难度来说，建议先改成：

```yaml
body: |
  {
    "workflowName": "{{workflow.name}}",
    "status": "{{workflow.status}}",
    "message": "workflow finished with status {{workflow.status}}"
  }
```

这样能保证回调 JSON 基本稳定。

如果后端需要详细失败原因，就在收到 `FAILED` 后主动调用 Argo 查询详情。任务管理方案里已经设计了 Argo 查询 API 和定时补偿逻辑，所以后端具备主动查 Argo 的基础。

## 4. 为什么建议清理 Kubernetes 运行时字段

当前 YAML 开头可能有这些字段：

```yaml
metadata:
  name: sacp-dms-data-process-with-qc
  namespace: sacp-dms-argo
  uid: 3451c6e2-e4b5-4c81-937b-b87391743479
  resourceVersion: "34633810"
  generation: 44
  creationTimestamp: "2026-06-15T06:13:59Z"
  managedFields:
    - manager: argo
      operation: Update
```

其中通常应该手写保留的是：

```yaml
metadata:
  name: sacp-dms-data-process-with-qc
  namespace: sacp-dms-argo
```

下面这些字段：

```text
uid
resourceVersion
generation
creationTimestamp
managedFields
```

是 Kubernetes 在资源创建或更新后自动生成的运行时信息。

可以理解为：开发者写 YAML 创建资源，Kubernetes 接收后会给这个资源分配唯一 ID、版本号、创建时间、管理记录。这些字段是集群内部用来管理资源版本和变更历史的，不应该由开发者手动维护。

## 5. 不清理这些字段会怎样

如果 YAML 只是拿来阅读、备份、对照方案，一般没事。

但如果拿这个 YAML 直接执行：

```bash
kubectl apply -f sacp-dms-data-process-with-qc.yaml
```

可能出现问题：

1. `resourceVersion` 是旧版本，集群可能认为你基于过期版本更新。
2. `uid` 是已有资源的唯一 ID，新建资源时不应该手动指定。
3. `managedFields` 很长，而且是 Kubernetes 自动维护的字段，手动带上没有意义。
4. 跨环境应用时，测试环境的 `uid`、`resourceVersion` 到生产环境完全不适用。

所以用于部署的 YAML 应该更像这样：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: sacp-dms-data-process-with-qc
  namespace: sacp-dms-argo
spec:
  entrypoint: data-process
  onExit: exit-handler
  templates:
    ...
```

注意：如果要 `kubectl apply`，通常还必须有：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
```

如果 YAML 是从 Argo UI 或接口导出的片段，有时顶部可能没带 `apiVersion` 和 `kind`。这种文件可以用于阅读参考，但不一定能直接部署。

## 6. 最终建议

如果这些 YAML 只是给人看、做方案验证：当前形式可以接受。

如果这些 YAML 要作为正式部署文件，建议做两件事：

1. 清理运行时字段，只保留 `metadata.name`、`metadata.namespace`，以及确实需要的业务 label。
2. 补齐 `apiVersion` 和 `kind`，确保 Kubernetes 能正确识别为 Argo `WorkflowTemplate`。

