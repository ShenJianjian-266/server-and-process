# Argo 任务管理需求梳理

## 1. 背景

当前项目中有多个业务流程会调用 `ArgoWorkflowClient` 拉起 Argo Workflow。Argo 任务被提交后，系统只拿到了 Argo 返回的 workflow 名称，但项目内没有统一的任务记录，也没有页面能查看任务执行状态。

导师希望新增一个“任务管理”能力：每次本系统调起 Argo 任务时，都在本系统记录一条任务，并持续维护它的状态。后续 Web 页面可以展示这些任务，也可以在任务失败后重新发起同一个任务。

## 2. 新手先理解的概念

Argo 可以理解为一个外部任务平台。本项目向 Argo 发 HTTP 请求，告诉 Argo 使用哪个 `WorkflowTemplate`、带哪些参数，Argo 就会创建一个真实运行的 `Workflow`。这个 `Workflow` 会跑脚本或容器任务，例如原始数据解析、事件解析、手动质检。

本项目的入口类是：

`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/client/ArgoWorkflowClient.java`

它会调用 Argo 的 submit API，并从响应里取出 `metadata.name`。这个值就是后续要展示和追踪的 Argo 任务名称。

## 3. 当前项目中调用 Argo 的地方

目前需要重点看这些调用点：

- `IndexController.submitArgo(...)`：测试或手动触发任意 Argo 模板。
- `DataUploadConsumer.dispatchParseStart(...)`：原始数据上传入库后，触发数据解析 Argo 任务。
- `EventCompleteConsumer.onMessage(...)`：事件完成消息入库后，触发事件解析 Argo 任务。
- `DataLogicSliceServiceImpl.create(...)`：手动创建逻辑切片且需要质检时，触发质检 Argo 任务。
- `DataUploadConsumerFullFlowTest`：测试里 mock 了 `ArgoWorkflowClient`，说明后续改造要同步考虑测试。

## 4. 要做的核心需求

### 4.1 提交 Argo 时记录任务

每次本系统成功调起 Argo Workflow 后，需要新增一条任务记录。记录至少要能回答：

- 这是哪一次 Argo 任务？
- 它属于哪类业务？
- 它关联了哪条原始数据或业务对象？
- 它当前是什么状态？
- 它失败时为什么失败？
- 它是否可以重新执行？

建议记录字段包括：

| 字段 | 说明 |
| --- | --- |
| id | 本系统任务记录主键 |
| workflowName | Argo 返回的 workflow 名称，即 `metadata.name` |
| templateName | 使用的 WorkflowTemplate 名称 |
| taskType | 任务类型，例如 DATA_PROCESS、EVENT_PROCESS、MANUAL_QC |
| bizId | 业务对象 ID，例如 dataId、eventInfoId、sliceId |
| dataId | 原始数据 ID；不是所有任务都有，可为空 |
| dataName | 原始数据名称；用于页面展示 |
| parameters | 提交给 Argo 的参数 JSON |
| status | 本系统维护的任务状态 |
| message | 失败原因或回调说明 |
| startTime | 任务提交时间 |
| finishTime | 任务完成时间 |
| retryFromTaskId | 如果是重跑任务，记录来源任务 ID |

### 4.2 维护任务状态

状态初步可以设计为：

- `SUBMITTED`：本系统已成功提交 Argo。
- `SUCCEEDED`：Argo 执行成功。
- `FAILED`：Argo 执行失败。
- `ERROR`：本系统提交或回调处理异常。
- `RETRY_SUBMITTED`：由失败任务重新发起的新任务。

### 4.3 Web 端任务管理页面

后续页面需要展示任务列表，至少支持：

- 按任务类型、状态、原始数据 ID、任务名称查询。
- 展示 Argo workflow 名称、模板名、业务 ID、原始数据名称、状态、开始时间、结束时间。
- 点击查看详情，能看到提交参数、失败原因、回调信息。
- 重跑按钮：对失败任务执行“重新解析”或“重新提交”。

### 4.4 支持失败后重跑

重跑不是简单更新旧记录，而是应该重新提交一次 Argo，并生成一条新的任务记录。新记录通过 `retryFromTaskId` 关联旧任务，方便页面展示“这次重跑来自哪一次失败任务”。

重跑时需要使用旧任务保存的 `templateName` 和 `parameters`。所以第一次提交时必须完整保存参数。

### 4.5 接收 Argo 回调

要让 Argo 调本项目的 HTTP 接口。本项目需要提供一个回调接口，例如：

`POST /argo/tasks/callback`

这个接口接收 Argo 返回的 workflow 名称、状态、完成时间、错误信息等，然后更新任务表。

回调接口不需要鉴权，要考虑：

- 如何根据 `workflowName` 找到本系统任务记录。
- 回调重复发送时，接口应保持幂等。
- 失败状态要保存失败原因。
- 回调失败处理：不重试，本项目主动轮询 Argo 状态

## 5. Argo 回调实现调研方向

根据 Argo 官方文档，可以重点调研：

- LifecycleHook：可在 Workflow 或模板生命周期满足条件时触发动作。
- Exit Handler：Workflow 结束时必定执行，适合统一上报成功或失败。
- HTTP Template：Argo Workflow 内可以发 HTTP 请求，适合作为回调本系统的步骤。

参考官方文档：

- https://argo-workflows.readthedocs.io/en/latest/lifecyclehook/
- https://argo-workflows.readthedocs.io/en/latest/walk-through/exit-handlers/
- https://argo-workflows.readthedocs.io/en/latest/http-template/

## 6. 可能的后端改造范围

建议新增一个独立模块，例如：

`com.seres.sacp.dms.module.argo`

可能包含：

- `entity/ArgoTask.java`：任务表实体。
- `mapper/ArgoTaskMapper.java` 和 XML：任务表查询与更新。
- `dto/ArgoTaskCallbackDTO.java`：接收 Argo 回调。
- `dto/ArgoTaskRetryDTO.java`：重跑请求。
- `query/ArgoTaskQuery.java`：页面分页查询条件。
- `vo/ArgoTaskVO.java`：页面展示对象。
- `service/ArgoTaskService.java`：创建任务、更新状态、重跑任务。
- `controller/ArgoTaskController.java`：任务分页、详情、重跑、回调接口。

同时需要改造 `ArgoWorkflowClient`：提交成功拿到 `workflowName` 后，通知任务服务保存记录。让业务调用方传入任务上下文，例如任务类型、dataId、dataName、bizId，而不是只传 Argo 参数。
