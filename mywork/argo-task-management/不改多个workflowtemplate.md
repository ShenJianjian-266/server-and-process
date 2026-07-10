# 不分别修改多个 WorkflowTemplate，能否实现 Argo 回调

## 结论

可以有替代方案，但要先明确一点：

**只新增一个普通 template 本身不够。**

在 Argo 里，一个 template 只有被 `entrypoint`、`steps`、`dag`、`onExit`、`hooks` 等引用时才会执行。也就是说，即使新增了一个公共 `callback` template，如果没有任何 Workflow 引用它，它不会自动运行，也不会自动回调后端。

因此，要实现 Workflow 结束时回调，本质上必须让回调逻辑挂到 Workflow 生命周期上。常见方式有：

1. 每个 WorkflowTemplate 分别加 `onExit`。
2. 做公共 WorkflowTemplate，再由业务 WorkflowTemplate 的 `onExit` 引用它。
3. 修改 Argo controller 的 `workflowDefaults`，让所有 Workflow 默认带 `onExit`。
4. 不做 Argo 回调，完全由后端轮询 Argo 状态。

## 方案一：分别修改 3 个 WorkflowTemplate

这是当前方案的做法：

```yaml
spec:
  onExit: exit-handler
  templates:
    - name: exit-handler
      http:
        url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
        method: "POST"
        headers:
          - name: "Content-Type"
            value: "application/json"
        body: |
          {
            "workflowName": "{{workflow.name}}",
            "status": "{{workflow.status}}",
            "message": "workflow finished with status {{workflow.status}}"
          }
```

优点：

1. 最直接。
2. 最容易理解。
3. 影响范围只限当前 3 个 WorkflowTemplate。
4. 出问题时容易定位。

缺点：

1. 有几个 WorkflowTemplate，就要改几个。
2. 如果以后回调 URL 或 body 变了，需要多个模板同步修改。

当前只有 3 个模板：数据解析、事件解析、手动质检。因此这个方案仍然是最稳妥、最可控的。

## 方案二：新增公共 WorkflowTemplate，但业务模板仍然要引用

可以新增一个公共回调模板，例如：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: dms-callback-template
  namespace: sacp-dms-argo
spec:
  templates:
    - name: callback
      http:
        url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
        method: "POST"
        headers:
          - name: "Content-Type"
            value: "application/json"
        body: |
          {
            "workflowName": "{{workflow.name}}",
            "status": "{{workflow.status}}",
            "message": "workflow finished with status {{workflow.status}}"
          }
```

然后业务 WorkflowTemplate 里仍然要加一个 `onExit`，只不过这个 `onExit` 不直接写 HTTP 回调，而是引用公共模板：

```yaml
spec:
  onExit: exit-handler
  templates:
    - name: exit-handler
      steps:
        - - name: callback
            templateRef:
              name: dms-callback-template
              template: callback
```

注意：

**公共 WorkflowTemplate 不会自动执行。**

业务 WorkflowTemplate 里仍然必须有：

```yaml
onExit: exit-handler
```

否则 Workflow 结束时不会触发回调。

这个方案的优点：

1. 回调 URL、header、body 集中在一个公共模板里。
2. 以后回调逻辑变化时，只需要改公共模板。
3. 业务模板里只保留很薄的一层 `exit-handler` 引用。

这个方案的缺点：

1. 首次接入时，3 个业务 WorkflowTemplate 仍然要分别加 `onExit` 和引用壳。
2. 比直接写 HTTP Template 多了一层间接引用，新手理解成本更高。
3. 要确认当前 Argo 版本支持 `templateRef` 引用 WorkflowTemplate。

适用场景：

如果未来会有很多 WorkflowTemplate 都要接入同一套回调逻辑，可以考虑这个方案。

如果当前只有 3 个模板，直接分别加 `exit-handler` 更简单。

## 方案三：修改 workflowDefaults，一处全局生效

Argo 支持在 workflow-controller 的 ConfigMap 里配置 `workflowDefaults`。它可以给该 controller 管理的 Workflow 设置默认 spec。

理论上可以配置类似：

```yaml
workflowDefaults: |
  spec:
    onExit: dms-global-exit-handler
    templates:
      - name: dms-global-exit-handler
        http:
          url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
          method: "POST"
          headers:
            - name: "Content-Type"
              value: "application/json"
          body: |
            {
              "workflowName": "{{workflow.name}}",
              "status": "{{workflow.status}}",
              "message": "workflow finished with status {{workflow.status}}"
            }
```

这样看起来就不用分别修改 3 个 WorkflowTemplate。

但是这个方案要非常谨慎。

风险：

1. 可能影响同一个 Argo controller 管理的所有 Workflow，不只是 DMS 的 3 个模板。
2. 需要 Argo 管理员权限修改 workflow-controller ConfigMap。
3. 如果其他业务也使用同一个 Argo controller，可能会给它们也加上 DMS 回调。
4. 如果某些 Workflow 已经自己定义了 `onExit`，要确认默认 `onExit` 和自定义 `onExit` 的覆盖或合并行为。
5. 要确认当前 Argo 版本对 `workflowDefaults.spec.templates` 数组的合并行为是否符合预期。

适用场景：

1. Argo 平台由你们团队统一维护。
2. 这个 controller 只服务 DMS 或明确允许所有 Workflow 都走同一个回调。
3. 已经在测试环境验证过 `workflowDefaults` 的合并和覆盖行为。

不适合的场景：

1. 你只是业务系统开发者，没有 Argo 平台管理员权限。
2. 同一个 Argo controller 被多个业务团队共享。
3. 当前只是为了接入 3 个固定模板。

因此，`workflowDefaults` 更适合平台级统一治理，不太适合作为当前需求的首选方案。

## 方案四：不做 Argo 回调，完全由后端轮询

还有一种方式是不改任何 WorkflowTemplate。

因为后端提交 Argo 成功后已经拿到了 `workflowName`，所以后端可以定时调用 Argo API 查询状态：

```text
GET /api/v1/workflows/{namespace}/{workflowName}
```

重点读取：

```text
status.phase
status.message
status.finishedAt
```

然后更新本地 `argo_task` 表。

优点：

1. 不需要修改任何 WorkflowTemplate。
2. 不依赖 Argo 回调能否访问本系统。
3. 没有 `workflow.failures` 破坏 JSON body 的风险。
4. 所有状态同步逻辑都在后端，可控性更强。

缺点：

1. 状态不是实时的，取决于定时任务执行周期。
2. 会增加 Argo API 查询压力。
3. 需要处理查询失败次数、长期 Running、Workflow 被删除等异常情况。
4. 如果任务很多，需要控制批量大小和扫描频率。

适用场景：

1. 不能修改 Argo WorkflowTemplate。
2. Argo 无法访问后端回调接口。
3. 可以接受分钟级状态延迟。

如果需求只是“本系统最终能维护 Argo 任务状态”，这个方案可以成立。

但如果需求明确要求“接收 Argo 回调”，那它只能作为兜底方案，不能完全替代回调。

## 推荐选择

当前只有 3 个 WorkflowTemplate：

1. `sacp-dms-data-process-with-qc`
2. `sacp-dms-event-parse`
3. `sacp-dms-manual-qc`

最推荐：

```text
分别给 3 个 WorkflowTemplate 加 onExit + exit-handler；
后端保留定时轮询作为回调失败兜底。
```

原因：

1. 改动范围小。
2. 风险可控。
3. 不影响其他 Workflow。
4. 不需要平台级 ConfigMap 权限。
5. 出问题时容易定位到具体模板。

如果未来 WorkflowTemplate 数量变多，可以再演进成：

```text
公共 callback WorkflowTemplate + 每个业务模板薄壳引用
```

如果未来由平台团队统一治理所有 Workflow，可以再考虑：

```text
workflowDefaults 全局 onExit
```

## 补充：有没有在某个 namespace 中全局生效的方案

有，但严格说不是“在某个 namespace 下新增一个 WorkflowTemplate 就自动全局生效”，而是要通过 Argo controller 或 Kubernetes admission 层做统一注入。

### 方案 A：namespace 专用 Argo Controller + workflowDefaults

这是最接近“某个 namespace 全局生效”的 Argo 原生方案。

Argo 的 `workflowDefaults` 配在 `workflow-controller-configmap` 上，作用范围是“这个 controller 管理的所有 Workflow”。如果要把影响限制在 `sacp-dms-argo` 这个 namespace，关键是让这个 controller 只管理这个 namespace，而不是使用多业务共享的集群级 controller。

示例思路：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
data:
  workflowDefaults: |
    spec:
      onExit: dms-global-exit-handler
      templates:
        - name: dms-global-exit-handler
          http:
            url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
            method: "POST"
            headers:
              - name: "Content-Type"
                value: "application/json"
            body: |
              {
                "workflowName": "{{workflow.name}}",
                "status": "{{workflow.status}}",
                "message": "workflow finished with status {{workflow.status}}"
              }
```

配套 controller 启动参数大致是：

```yaml
args:
  - --configmap
  - workflow-controller-configmap
  - --namespaced
  - --managed-namespace
  - sacp-dms-argo
```

这样 `workflowDefaults` 的实际影响范围就是这个 controller 管理的 namespace，而不是整个集群。

风险点：

1. 如果当前 Argo Controller 是集群级共享的，直接改 `workflowDefaults` 会影响它管理的所有 Workflow。
2. 如果某个 WorkflowTemplate 已经定义了自己的 `onExit`，Workflow 自己的值会优先，默认回调可能不会生效。
3. `workflowDefaults.spec.templates` 和业务模板合并要在测试环境验证，尤其是模板名冲突、已有 `onExit`、已有 `templates` 的场景。
4. 这需要 Argo/Kubernetes 管理权限，不是普通业务服务改代码能完成的。

### 方案 B：Kubernetes MutatingAdmissionWebhook / Kyverno 注入

如果平台侧允许，也可以在 Kubernetes admission 层对某个 namespace 的 `Workflow` CR 做变更注入：当创建 `argoproj.io/Workflow` 时，如果 namespace 是 `sacp-dms-argo`，自动补 `spec.onExit` 和回调 template。

这个方案是真正的 namespace 维度策略，不依赖业务 WorkflowTemplate 改动。但它不是 Argo 原生配置，属于 Kubernetes 平台治理方案，复杂度和权限要求更高。

### 补充结论

推荐顺序：

1. 如果 `sacp-dms-argo` 能由独立 Argo Controller 管理：用 `workflowDefaults`，这是最贴近 Argo 原生能力的方案。
2. 如果当前 Argo Controller 是多业务共享：不要直接改全局 `workflowDefaults`，除非能通过独立 controller / managed namespace 隔离影响面。
3. 如果平台团队能做 admission 策略：可以用 MutatingAdmissionWebhook/Kyverno 做 namespace 注入，但这比改 3 个 WorkflowTemplate 重得多。

结合本项目当前只有 3 个模板，“分别加 `onExit` + 后端轮询兜底”仍然是工程风险最低的方案；namespace 全局方案更适合平台侧统一治理。

## 一句话总结

只新增一个 template 不会自动回调；必须有 `onExit` 或全局默认机制把它挂到 Workflow 生命周期上。当前只有 3 个模板，分别加 `onExit` 是最稳妥的方案。
