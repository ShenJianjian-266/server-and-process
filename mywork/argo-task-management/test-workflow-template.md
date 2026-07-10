# Argo UI 测试用 WorkflowTemplate

本文提供 2 个测试用 WorkflowTemplate：

1. `sacp-dms-test-success-3min`：任务运行 3 分钟后成功。
2. `sacp-dms-test-fail-3min`：任务运行 3 分钟后失败。

这两个模板可以用于测试：

1. Argo WorkflowTemplate 创建。
2. Argo 任务提交。
3. `onExit` 回调。
4. 后端任务状态从 `SUBMITTED` 更新为 `SUCCEEDED` 或 `FAILED`。
5. 定时轮询补偿逻辑。

## 1. 成功模板

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: sxw-test-001
  namespace: sacp-dms-argo
spec:
  arguments:
    parameters:
      - name: user
        value: "hh"
  entrypoint: main
  onExit: exit-handler
  templates:
    - name: main
      container:
        image: hb.seres.cn/sacp-cicd-common/sacp-dms-data-process:0.0.11.11
        command:
          - sh
          - -c
        args:
          - |
            echo "test success workflow started"
            sleep 180
            echo "test success workflow finished"
            exit 0

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

## 2. 失败模板

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: sacp-dms-test-fail-3min
  namespace: sacp-dms-argo
spec:
  arguments:
  parameters:
    - name: test_user
      value: "hh"
  entrypoint: main
  onExit: exit-handler
  templates:
    - name: main
      container:
        image: busybox:1.36
        command:
          - sh
          - -c
        args:
          - |
            echo "test fail workflow started"
            sleep 180
            echo "test fail workflow finished"
            exit 1

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

## 3. 使用说明

在 Argo UI 中创建这两个 WorkflowTemplate 后，可以分别提交它们。

预期结果：

1. `sacp-dms-test-success-3min` 提交后，Workflow 运行约 3 分钟，最终状态为 `Succeeded`。
2. `sacp-dms-test-fail-3min` 提交后，Workflow 运行约 3 分钟，最终状态为 `Failed`。
3. Workflow 结束后都会执行 `exit-handler`。
4. `exit-handler` 会调用：

```text
http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback
```

## 4. 注意事项
### 4.1 镜像可用性

模板中使用的镜像是：

```text
busybox:1.36
```

如果集群不能访问公网镜像，或者没有权限拉取该镜像，需要替换成你们内网镜像仓库中带 `sh` 和 `sleep` 命令的基础镜像。

例如：

```yaml
image: 你们内网仓库/busybox:1.36
```

### 4.2 回调地址

当前回调地址是：

```text
http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback
```

这个地址会回调 Kubernetes 集群内的 `data-server` 服务。

如果你本地启动 `data-server`，这个地址不会回调到本地。

如果要回调本地服务，需要把 URL 改成 Argo 集群能访问到的本地调试地址。

### 4.3 message 字段

这里没有使用：

```text
{{workflow.failures}}
```

而是使用：

```text
workflow finished with status {{workflow.status}}
```

原因是 `workflow.failures` 可能包含双引号、换行或复杂 JSON，直接放进 JSON 字符串里可能导致回调 body 非法。

### 4.4 运行时间

任务运行 3 分钟是由下面这行控制的：

```bash
sleep 180
```

如果想改成 1 分钟，可以改成：

```bash
sleep 60
```

如果想改成 5 分钟，可以改成：

```bash
sleep 300
```

### 若http template不可用
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: sxw-test-002
  namespace: sacp-dms-argo
spec:
  arguments:
    parameters:
      - name: user
        value: "hh"
  entrypoint: main
  onExit: exit-handler
  templates:
    - name: main
      container:
        image: hb.seres.cn/sacp-cicd-common/sacp-dms-data-process:0.0.11.11
        command:
          - sh
          - -c
        args:
          - |
            echo "test success workflow started"
            sleep 180
            echo "test success workflow finished"
            exit 0

    - name: exit-handler
      container:
        image: hb.seres.cn/sacp-cicd-common/sacp-dms-data-process:0.0.11.11
        command: [sh, -c]
        args:
          - |
            curl -sS -X POST "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback" \
              -H "Content-Type: application/json" \
              -d "{\"workflowName\":\"{{workflow.name}}\",\"status\":\"{{workflow.status}}\",\"message\":\"workflow finished with status {{workflow.status}}\"}"
```