# Argo 任务管理前端接口文档

本文档面向任务管理页面的前端开发与联调，内容以当前后端实现为准。

## 1. 快速索引

| 场景 | 方法 | 接口 | 前端调用 |
| --- | --- | --- | --- |
| 查询任务列表 | `POST` | `/argo/tasks/page` | 是 |
| 查询任务详情 | `GET` | `/argo/tasks/{id}` | 是 |
| 重跑失败任务 | `POST` | `/argo/tasks/{id}/retry` | 是 |
| 接收任务结束回调 | `POST` | `/argo/tasks/callback` | 否，仅供 Argo 调用 |

列表默认按任务记录创建时间倒序，最新任务排在最前面。

## 2. 请求约定

### 2.1 基础地址

- 本地默认地址：`http://localhost:20001/sacp-dms-data-server`
- 本地列表接口完整示例：`http://localhost:20001/sacp-dms-data-server/argo/tasks/page`
- 生产环境是否包含 context-path 由网关配置决定，前端应使用环境变量中的服务基础地址，不要硬编码。

### 2.2 请求头

业务接口使用 JSON：

```http
Content-Type: application/json
Authorization: Bearer <token>
```

当前服务不会因缺少或解析失败的 `Authorization` 头直接拒绝请求，但浏览器端仍应沿用项目统一的登录态请求头。`/argo/tasks/callback` 已排除 JWT 拦截，仅供 Argo 调用。

### 2.3 统一响应结构

成功响应：

```json
{
  "code": 0,
  "data": {},
  "msg": ""
}
```

失败响应示例：

```json
{
  "code": 400,
  "data": null,
  "msg": "请求参数不正确:workflowName不能为空"
}
```

前端应以响应体的 `code === 0` 判断业务成功，不要只依赖 HTTP 状态码；失败时优先展示 `msg`。

### 2.4 ID、时间和 JSON 字符串

- `id`、`dataId`、`retryFromTaskId` 均为 Long。超过 JavaScript 安全整数范围时，服务会序列化为字符串，前端类型建议统一声明为 `string | null`，不得使用 `Number()` 转换后再传回。
- `createTime`、`startTime`、`finishTime` 当前返回 Unix 毫秒时间戳。`finishTime` 在任务未结束时为 `null`。
- `parameters` 是 JSON 字符串，不是 JSON 对象。详情页格式化展示前需要 `JSON.parse`，并做好解析失败兜底。

## 3. 枚举

### 3.1 任务类型 `taskType`

| 值 | 页面文案 |
| --- | --- |
| `DATA_PROCESS` | 原始数据解析 |
| `EVENT_PROCESS` | 事件解析 |
| `MANUAL_QC` | 手动质检 |
| `CUSTOM` | 手动提交 |

### 3.2 任务状态 `status`

| 值 | 页面文案 | 是否终态 | 是否允许重跑 |
| --- | --- | --- | --- |
| `SUBMITTED` | 已提交 | 否 | 否 |
| `RETRY_SUBMITTED` | 重跑已提交 | 否 | 否 |
| `SUCCEEDED` | 成功 | 是 | 否 |
| `FAILED` | 失败 | 是 | 是 |
| `ERROR` | 异常 | 是 | 是 |

页面仅应在 `FAILED` 或 `ERROR` 状态展示“重跑”按钮。当前任务模型没有 `RUNNING` 状态，Argo 执行期间仍显示为 `SUBMITTED` 或 `RETRY_SUBMITTED`。

## 4. 公共任务对象 `ArgoTaskVO`

列表项、详情和重跑成功响应中的任务对象结构相同。

| 字段                | 类型 | 可空 | 说明                       |
|-------------------| --- | --- |--------------------------|
| `id`              | string | 否 | 本系统任务记录 ID               |
| `workflowName`    | string | 否 | Argo Workflow 的真实名称      |
| `templateName`    | string | 否 | WorkflowTemplate 名称      |
| `taskType`        | string | 否 | 任务类型，见任务类型枚举             |
| `bizId`           | string | 是 | 业务对象 ID，可能是数据、事件或逻辑切片 ID |
| `dataId`          | string | 是 | 关联的原始数据 ID               |
| `dataName`        | string | 是 | 关联的原始数据名称                |
| `sliceName`       | string | 是 | 逻辑切片名称；非切片任务为 `null`      |
| `creator`         | string | 是 | 任务记录创建人；非 HTTP 场景可能为 `null` |
| `creatorEmployeeNo` | string | 是 | 创建人员工编号；非 HTTP 场景可能为 `null` |
| `createTime`      | number | 否 | 任务记录创建时间，Unix 毫秒时间戳       |
| `parameters`      | string | 否 | 提交给 Argo 的参数 JSON 字符串    |
| `generateName`    | string | 是 | 提交 Workflow 时使用的名称前缀     |
| `status`          | string | 否 | 任务状态，见任务状态枚举             |
| `message`         | string | 是 | Argo 回调消息或失败原因           |
| `startTime`       | number | 否 | 任务提交时间，Unix 毫秒时间戳        |
| `finishTime`      | number | 是 | 任务完成时间，Unix 毫秒时间戳        |
| `retryFromTaskId` | string | 是 | 重跑来源任务 ID；首次提交时为 `null`  |

示例：

```json
{
  "id": "2079410529841000449",
  "workflowName": "data-process-adc-2079409991000000001-x7k2m",
  "templateName": "sacp-dms-data-process-with-qc",
  "taskType": "DATA_PROCESS",
  "bizId": "2079409991000000001",
  "dataId": "2079409991000000001",
  "dataName": "ADC_20260710_001",
  "sliceName": null,
  "creator": null,
  "creatorEmployeeNo": null,
  "createTime": 1783648800000,
  "parameters": "{\"data_id\":\"2079409991000000001\",\"s3_path\":\"s3://bucket/path\"}",
  "generateName": "data-process-adc-2079409991000000001-",
  "status": "FAILED",
  "message": "workflow finished with status Failed",
  "startTime": 1783648800000,
  "finishTime": 1783648920000,
  "retryFromTaskId": null
}
```

## 5. 分页查询任务

### `POST /argo/tasks/page`

用于任务管理列表和筛选。

### 请求体

所有字段均可选；请求体至少传 `{}`。

| 字段 | 类型 | 默认值 | 匹配方式 | 说明 |
| --- | --- | --- | --- | --- |
| `pageNum` | number | `1` | — | 页码，从 1 开始 |
| `pageSize` | number | `10` | — | 每页条数 |
| `taskType` | string | — | 精确 | 任务类型枚举值 |
| `status` | string | — | 精确 | 任务状态枚举值 |
| `dataId` | string | — | 精确 | 原始数据 ID |
| `workflowName` | string | — | 模糊 | Argo Workflow 名称，前后包含匹配 |
| `templateName` | string | — | 模糊 | WorkflowTemplate 名称，前后包含匹配 |

请求示例：

```json
{
  "pageNum": 1,
  "pageSize": 10,
  "taskType": "DATA_PROCESS",
  "status": "FAILED",
  "dataId": "2079409991000000001",
  "workflowName": "data-process"
}
```

### 成功响应

`data.list` 为任务数组。`PageInfo` 还会返回导航字段，前端列表主要使用下表字段即可。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `data.list` | ArgoTaskVO[] | 当前页任务列表 |
| `data.total` | number | 总记录数 |
| `data.pageNum` | number | 当前页码 |
| `data.pageSize` | number | 每页条数 |
| `data.pages` | number | 总页数 |

```json
{
  "code": 0,
  "data": {
    "total": 1,
    "list": [
      {
        "id": "2079410529841000449",
        "workflowName": "data-process-adc-2079409991000000001-x7k2m",
        "templateName": "sacp-dms-data-process-with-qc",
        "taskType": "DATA_PROCESS",
        "bizId": "2079409991000000001",
        "dataId": "2079409991000000001",
        "dataName": "ADC_20260710_001",
        "sliceName": null,
        "creator": null,
        "creatorEmployeeNo": null,
        "createTime": 1783648800000,
        "parameters": "{\"data_id\":\"2079409991000000001\"}",
        "generateName": "data-process-adc-2079409991000000001-",
        "status": "FAILED",
        "message": "workflow finished with status Failed",
        "startTime": 1783648800000,
        "finishTime": 1783648920000,
        "retryFromTaskId": null
      }
    ],
    "pageNum": 1,
    "pageSize": 1,
    "size": 1,
    "startRow": 0,
    "endRow": 0,
    "pages": 1,
    "prePage": 0,
    "nextPage": 0,
    "isFirstPage": true,
    "isLastPage": true,
    "hasPreviousPage": false,
    "hasNextPage": false,
    "navigatePages": 8,
    "navigatepageNums": [1],
    "navigateFirstPage": 1,
    "navigateLastPage": 1
  },
  "msg": ""
}
```

## 6. 查询任务详情

### `GET /argo/tasks/{id}`

### 路径参数

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | string | 是 | 本系统任务记录 ID，不是 workflowName |

请求示例：

```http
GET /argo/tasks/2079410529841000449
```

成功时 `data` 为完整的 `ArgoTaskVO`：

```json
{
  "code": 0,
  "data": {
    "id": "2079410529841000449",
    "workflowName": "data-process-adc-2079409991000000001-x7k2m",
    "templateName": "sacp-dms-data-process-with-qc",
    "taskType": "DATA_PROCESS",
    "bizId": "2079409991000000001",
    "dataId": "2079409991000000001",
    "dataName": "ADC_20260710_001",
    "sliceName": null,
    "creator": null,
    "creatorEmployeeNo": null,
    "createTime": 1783648800000,
    "parameters": "{\"data_id\":\"2079409991000000001\"}",
    "generateName": "data-process-adc-2079409991000000001-",
    "status": "FAILED",
    "message": "workflow finished with status Failed",
    "startTime": 1783648800000,
    "finishTime": 1783648920000,
    "retryFromTaskId": null
  },
  "msg": ""
}
```

任务不存在时，当前实现返回非成功业务码，前端统一按 `code !== 0` 处理并展示 `msg`。

## 7. 重跑任务

### `POST /argo/tasks/{id}/retry`

使用旧任务保存的 `templateName`、`parameters` 和业务信息重新提交 Argo。旧任务不会被覆盖，成功后会新增一条任务记录。

### 可重跑条件

- 旧任务存在。
- 旧任务状态为 `FAILED` 或 `ERROR`。

### 路径参数

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | string | 是 | 要重跑的旧任务记录 ID |

### 请求体

请求体可省略，也可传空对象。

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `message` | string | 否 | 重跑说明；当前版本仅写入后端日志，不会保存到新任务或返回给前端 |

```json
{
  "message": "人工确认后重新提交"
}
```

### 成功响应

`data` 是新任务，不是旧任务。新任务具有以下特征：

- `id` 和 `workflowName` 都是新值。
- `status` 为 `RETRY_SUBMITTED`。
- `retryFromTaskId` 等于请求路径中的旧任务 ID。
- 旧任务仍保持原来的 `FAILED` 或 `ERROR` 状态。

```json
{
  "code": 0,
  "data": {
    "id": "2079410999000000001",
    "workflowName": "data-process-adc-2079409991000000001-r8p4s",
    "templateName": "sacp-dms-data-process-with-qc",
    "taskType": "DATA_PROCESS",
    "bizId": "2079409991000000001",
    "dataId": "2079409991000000001",
    "dataName": "ADC_20260710_001",
    "sliceName": null,
    "creator": "张三",
    "creatorEmployeeNo": "10001234",
    "createTime": 1783649100000,
    "parameters": "{\"data_id\":\"2079409991000000001\"}",
    "generateName": "data-process-adc-2079409991000000001-",
    "status": "RETRY_SUBMITTED",
    "message": null,
    "startTime": 1783649100000,
    "finishTime": null,
    "retryFromTaskId": "2079410529841000449"
  },
  "msg": ""
}
```

重跑会实际提交新的 Argo Workflow。前端应防止按钮重复点击，并在请求完成前显示 loading。请求成功后可刷新列表，或直接把返回的新任务插入列表顶部。

## 8. Argo 回调接口（前端无需调用）

### `POST /argo/tasks/callback`

该接口由 Argo Workflow 结束处理器调用，用 workflowName 更新本地任务状态。前端只需通过列表或详情接口读取更新结果。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `workflowName` | string | 是 | Argo Workflow 名称 |
| `status` | string | 是 | Argo 状态；`Succeeded` 映射为成功，`Failed`/`Error` 映射为失败，其他值映射为异常 |
| `message` | string | 否 | 回调说明或失败原因 |
| `finishTime` | number | 否 | 完成时间，Unix 毫秒时间戳；不传时使用服务端当前时间 |

```json
{
  "workflowName": "data-process-adc-2079409991000000001-x7k2m",
  "status": "Succeeded",
  "message": "workflow finished with status Succeeded",
  "finishTime": 1783648920000
}
```

成功响应：

```json
{
  "code": 0,
  "data": true,
  "msg": ""
}
```

回调具有以下幂等行为：

- 任务已经处于 `SUCCEEDED`、`FAILED` 或 `ERROR` 时，重复回调不再修改任务。
- workflowName 未匹配到任务时，接口仍返回成功，仅记录警告日志。

服务还会定时查询长期停留在提交状态的 Workflow，补偿回调丢失的情况。因此前端不需要主动请求 Argo。

## 9. 前端接入建议

### 9.1 TypeScript 类型

```ts
export type ArgoTaskType =
  | 'DATA_PROCESS'
  | 'EVENT_PROCESS'
  | 'MANUAL_QC'
  | 'CUSTOM'

export type ArgoTaskStatus =
  | 'SUBMITTED'
  | 'RETRY_SUBMITTED'
  | 'SUCCEEDED'
  | 'FAILED'
  | 'ERROR'

export interface CommonResult<T> {
  code: number
  data: T
  msg: string
}

export interface ArgoTask {
  id: string
  workflowName: string
  templateName: string
  taskType: ArgoTaskType
  bizId: string | null
  dataId: string | null
  dataName: string | null
  sliceName: string | null
  creator: string | null
  creatorEmployeeNo: string | null
  createTime: number
  parameters: string
  generateName: string | null
  status: ArgoTaskStatus
  message: string | null
  startTime: number
  finishTime: number | null
  retryFromTaskId: string | null
}

export interface ArgoTaskPageQuery {
  pageNum?: number
  pageSize?: number
  taskType?: ArgoTaskType
  status?: ArgoTaskStatus
  dataId?: string
  workflowName?: string
  templateName?: string
}
```

### 9.2 参数展示兜底

```ts
export function formatTaskParameters(parameters: string): string {
  try {
    return JSON.stringify(JSON.parse(parameters), null, 2)
  } catch {
    return parameters
  }
}
```

### 9.3 页面操作规则

- 列表筛选项直接使用本文档中的任务类型和状态枚举，当前没有单独的枚举查询接口。
- 详情抽屉中建议展示 `parameters`、`message`、`workflowName` 和重跑来源 ID。
- 重跑按钮只对 `FAILED`、`ERROR` 显示，并增加二次确认和请求 loading。
- 重跑成功后以响应中的新任务 ID 跳转或刷新，不要继续轮询旧任务。
- 如需自动刷新执行状态，可周期性重新请求列表或新任务详情；建议页面不可见时停止轮询。

### 9.4 跳转 Argo Workflow 详情页

任务列表中的“详情”按钮应使用该行的 `workflowName` 打开 Argo Workflow 页面，不要使用本系统任务 `id`。开发环境地址格式为：

```text
https://siop-dev.seres.cn/argo/workflows/sacp-dms-argo/{workflowName}
```

Argo UI 地址和 namespace 应按环境配置，不能在页面组件中写死。Vite 项目示例：

```dotenv
VITE_ARGO_UI_BASE_URL=https://siop-dev.seres.cn/argo
VITE_ARGO_NAMESPACE=sacp-dms-argo
```

统一封装 URL 构造和跳转方法：

```ts
const argoUiBaseUrl = import.meta.env.VITE_ARGO_UI_BASE_URL
const argoNamespace = import.meta.env.VITE_ARGO_NAMESPACE

export function buildArgoWorkflowUrl(workflowName: string): string {
  if (!workflowName) {
    throw new Error('workflowName 不能为空')
  }
  if (!argoUiBaseUrl || !argoNamespace) {
    throw new Error('Argo UI 地址或 namespace 未配置')
  }

  const baseUrl = argoUiBaseUrl.replace(/\/+$/, '')
  return `${baseUrl}/workflows/${encodeURIComponent(argoNamespace)}/${encodeURIComponent(workflowName)}`
}

export function openArgoWorkflowDetail(task: ArgoTask): void {
  const url = buildArgoWorkflowUrl(task.workflowName)
  window.open(url, '_blank', 'noopener,noreferrer')
}
```

前端实现要求：

- `workflowName` 为空时禁用“详情”按钮或给出明确提示。
- 推荐新标签页打开，以保留任务列表的页码和筛选条件。
- 不要将后端访问 Argo API 的 Bearer Token 放入前端配置或跳转 URL。
- 跳转后的登录和查看权限由 Argo UI 自身的登录态及 RBAC 控制。

## 10. 当前联调注意事项

1. 重跑请求的 `message` 当前只记录日志，不会持久化。
2. 任务执行中没有独立的 `RUNNING` 状态。
3. 大整数 ID 可能以字符串返回，前端必须按字符串保存和回传。
4. 响应提示字段名是 `msg`，不是 `message`；任务自身的回调说明字段才叫 `message`。
5. 数据库字段 `creator_employee_no`、`slice_name`、`create_time` 在 JSON 中分别使用驼峰字段 `creatorEmployeeNo`、`sliceName`、`createTime`。
