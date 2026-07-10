# 本地启动 data-server 时，Argo 是否会回调本地服务

## 1. 结论

本地启动 `data-server` 后：

1. **是否连接 Argo，取决于你有没有触发提交 Argo 的代码。**
2. **Argo 默认不会回调你本地启动的服务。**

原因是当前 WorkflowTemplate 里的回调地址是 Kubernetes 集群内 Service 地址：

```yaml
url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
```

这个地址指向集群内的 `sacp-dms-data-server` 服务，不是你电脑上的 `localhost:8080`。

## 2. 本地启动会不会连接 Argo

本地配置里已经配置了 Argo：

```yaml
argo:
  endpoint: ${ARGO_ENDPOINT:https://siop-dev.seres.cn/argo}
  namespace: ${ARGO_NAMESPACE:sacp-dms-argo}
  template: ${ARGO_TEMPLATE:sacp-dms-data-process-with-qc}
  event-template: ${ARGO_EVENT_TEMPLATE:sacp-dms-event-parse}
  manual-qc-template: ${ARGO_QC_TEMPLATE:sacp-dms-manual-qc}
```

也就是说，本地启动的 `data-server` 具备连接 Argo 的能力。

如果本地触发了这些逻辑，就会请求 Argo：

1. 调用 `/index/argo/submit` 手动提交 Argo。
2. 数据上传流程里 `dynamicConfig.getArgoFlag() != 0`，触发数据解析 Argo。
3. 改造后的 `ArgoTaskService.submitAndCreateTask(...)`。
4. 手动质检流程提交 Argo。
5. 事件解析流程提交 Argo。

请求目标默认是：

```text
https://siop-dev.seres.cn/argo
```

但是，如果只是本地启动项目，不触发提交任务的接口或消息消费，一般不会主动提交 Argo Workflow。

## 3. Argo 为什么不会回调本地服务

WorkflowTemplate 中的回调地址是：

```yaml
url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
```

这个地址的含义是：

```text
服务名：sacp-dms-data-server
命名空间：sacp-dms
端口：80
路径：/argo/tasks/callback
```

Argo Workflow 运行在 Kubernetes 集群里。Workflow 结束时，`exit-handler` 会从集群内部访问：

```text
http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback
```

它不会访问你本地电脑上的：

```text
http://localhost:8080/argo/tasks/callback
```

所以，如果本地启动服务后提交 Argo，实际流程通常是：

1. 本地 `data-server` 调 Argo submit API。
2. Argo 在集群里创建 Workflow。
3. Workflow 运行结束。
4. Argo 调用 `http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback`。
5. 被回调的是集群里的 `data-server`，不是本地启动的 `data-server`。

## 4. 可能出现的问题

如果本地服务和集群服务使用的不是同一个数据库，就可能出现状态不同步。

典型情况：

1. 本地 `data-server` 提交 Argo 成功。
2. 本地数据库插入一条 `argo_task` 记录。
3. Argo Workflow 结束后，回调打到集群里的 `data-server`。
4. 集群里的 `data-server` 查询自己的数据库。
5. 如果集群数据库里没有这条 `workflowName`，回调无法更新本地那条任务。
6. 本地 `argo_task` 状态会一直停留在 `SUBMITTED` 或 `RETRY_SUBMITTED`。

这也是方案里设计“定时轮询 Argo 状态”的原因之一。

本地服务如果能访问 Argo API，就可以自己定时查询 Workflow 状态，然后补偿更新本地数据库。

## 5. 如果想让 Argo 回调本地，怎么办

核心原则：

**回调 URL 必须是 Argo 集群能访问到的地址。**

不能写：

```yaml
url: "http://localhost:8080/argo/tasks/callback"
```

因为这个 `localhost` 对 Argo Pod 来说，是 Argo Pod 自己，不是你的电脑。

可以选择以下方式。

### 方式一：使用集群能访问到的本机 IP

如果 Kubernetes 集群网络能访问你的电脑 IP，可以把 URL 改成：

```yaml
url: "http://你的电脑IP:8080/argo/tasks/callback"
```

前提：

1. 你的电脑 IP 对集群可达。
2. 本地服务监听端口对外开放。
3. 防火墙没有拦截。
4. 公司网络策略允许集群访问你的电脑。

这个方式不一定总能成功，取决于网络环境。

### 方式二：使用内网穿透

可以使用 ngrok、frp、NAT 隧道等方式，把本地端口暴露成 Argo 能访问的 URL。

例如把本地：

```text
http://localhost:8080
```

映射成：

```text
https://xxx.ngrok.example
```

然后 WorkflowTemplate 里写：

```yaml
url: "https://xxx.ngrok.example/argo/tasks/callback"
```

优点是适合本地临时调试。

缺点是要注意安全，不能把没有鉴权的回调接口长期暴露到公网。

### 方式三：在测试集群部署自己的 data-server 实例

可以在测试集群中部署一个自己的 `data-server` 实例，例如：

```text
sacp-dms-data-server-yourname.sacp-dms
```

然后 WorkflowTemplate 回调到：

```yaml
url: "http://sacp-dms-data-server-yourname.sacp-dms:80/argo/tasks/callback"
```

优点：

1. 网络路径和真实环境一致。
2. 不依赖本地电脑在线。
3. 调试更接近实际部署。

缺点：

1. 需要部署权限。
2. 需要独立配置数据库或测试数据。
3. 要避免影响公共测试环境。

### 方式四：临时修改 WorkflowTemplate callback URL

如果只是临时验证回调，可以把 WorkflowTemplate 中的：

```yaml
url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
```

临时改成你可访问的调试地址：

```yaml
url: "http://你的可访问地址/argo/tasks/callback"
```

验证完成后再改回正式集群内地址。

注意：不要在多人共用的 WorkflowTemplate 上随意改回调地址，否则可能影响其他人提交的任务。

## 6. 本地调试建议

本地开发时，不建议强依赖 Argo 真实回调到本地。

更推荐下面两种方式。

### 方式一：手动 curl 模拟回调

本地启动 `data-server` 后，手动调用本地回调接口：

```bash
curl -X POST 'http://localhost:8080/argo/tasks/callback' \
  -H 'Content-Type: application/json' \
  -d '{
    "workflowName": "实际workflowName",
    "status": "Succeeded",
    "message": "local test"
  }'
```

这样可以验证：

1. 回调接口是否能正常接收请求。
2. 是否能根据 `workflowName` 找到本地 `argo_task`。
3. 是否能把状态更新为 `SUCCEEDED` 或 `FAILED`。
4. 幂等逻辑是否正常。

### 方式二：启用后端定时轮询补偿

如果本地服务能访问 Argo API，可以启用定时任务，让本地服务主动查询 Argo 状态。

流程：

1. 本地服务提交 Argo。
2. 本地数据库写入 `argo_task`。
3. Argo 结束后即使没有回调本地，本地定时任务也会扫描长时间处于 `SUBMITTED` 的任务。
4. 本地服务调用 Argo API 查询 Workflow 状态。
5. 查询到终态后，本地服务补偿更新自己的数据库。

这个方式更接近真实容错逻辑，也不用反复修改 Argo WorkflowTemplate 的回调地址。

## 7. 最终建议

如果只是开发和验证后端逻辑：

```text
本地启动 data-server + curl 模拟回调
```

如果要验证 Argo 任务真实状态同步：

```text
本地启动 data-server + 提交 Argo + 使用定时轮询补偿
```

如果必须验证 Argo 真实回调本地：

```text
把 callback URL 改成 Argo 集群能访问到的本地暴露地址
```

默认情况下，当前 WorkflowTemplate 会回调集群内服务：

```text
http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback
```

不会回调本地：

```text
http://localhost:8080/argo/tasks/callback
```

## 8. 如何知道自己的调试地址

如果要临时修改 WorkflowTemplate callback URL，让 Argo 回调本地服务，那么这个“调试地址”必须满足一个条件：

**Argo 集群里的 Pod 能访问到它。**

所以不能简单写：

```text
http://localhost:8080/argo/tasks/callback
```

因为对 Argo Pod 来说，`localhost` 是 Argo Pod 自己，不是你的电脑。

### 8.1 先确认本地服务端口

本地启动 `data-server` 后，通常是：

```text
http://localhost:8080
```

可以先在本机测试：

```bash
curl http://localhost:8080/actuator/health
```

或者直接测试回调接口：

```bash
curl -X POST 'http://localhost:8080/argo/tasks/callback' \
  -H 'Content-Type: application/json' \
  -d '{
    "workflowName": "test",
    "status": "Succeeded",
    "message": "local test"
  }'
```

如果本地都访问不了，Argo 更不可能访问。

### 8.2 找本机局域网 IP

Linux/macOS：

```bash
ip addr
```

或者：

```bash
hostname -I
```

Windows：

```bat
ipconfig
```

假设你的 IP 是：

```text
10.10.20.30
```

那候选调试地址就是：

```text
http://10.10.20.30:8080/argo/tasks/callback
```

注意：这只是“候选地址”，不代表 Argo 一定能访问。

### 8.3 判断 Argo 集群能否访问这个 IP

最直接的方法是在 Argo 所在 Kubernetes 集群里启动一个临时 Pod，或者找一个已有 Pod，执行：

```bash
curl -X POST 'http://10.10.20.30:8080/argo/tasks/callback' \
  -H 'Content-Type: application/json' \
  -d '{
    "workflowName": "test",
    "status": "Succeeded",
    "message": "from k8s"
  }'
```

如果返回成功，说明这个地址可以作为 callback URL。

如果访问失败，常见原因包括：

1. 你的电脑 IP 对集群不可达。
2. 公司网络隔离。
3. 本地防火墙拦截了 8080。
4. `data-server` 只监听了 `127.0.0.1`，没有监听 `0.0.0.0`。
5. 你连的是 VPN，集群访问不到你的 VPN 地址。

### 8.4 如果集群访问不到本机，使用内网穿透

可以使用 frp、ngrok、公司内部网关等方式，把本地端口暴露成一个 Argo 能访问的 URL。

例如把本地：

```text
http://localhost:8080
```

暴露成：

```text
https://abc-debug.example.com
```

那 callback URL 就是：

```text
https://abc-debug.example.com/argo/tasks/callback
```

### 8.5 最可靠的调试地址

如果有权限在测试集群部署自己的 `data-server`，最可靠的是集群内 Service 地址，例如：

```text
http://sacp-dms-data-server-yourname.sacp-dms:80/argo/tasks/callback
```

这种地址和 Argo 在同一个 Kubernetes 网络里，通常比回调本地电脑更稳定。

### 8.6 一句话判断

你的调试地址不是“本地浏览器能访问的地址”，而是：

```text
从 Argo Workflow 所在集群能 curl 通的 /argo/tasks/callback 地址
```

能从集群里 curl 通，就可以填进 WorkflowTemplate 的 callback URL。
