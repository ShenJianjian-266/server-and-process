# Argo 任务列表增强方案

## 1. 需求和最终效果

本次改造包含两部分：

1. 在 `argo_task` 表中新增 `slice_name` 字段，用来保存逻辑切片名称。
2. Argo 任务分页接口返回数据库记录中的以下字段：
   - `creator`：创建人用户名。
   - `creator_employee_no`：创建人员工编号。
   - `slice_name`：切片名称。
   - `create_time`：任务记录创建时间。
3. 前端任务列表增加“详情”按钮。点击后打开对应的 Argo Workflow 页面，例如：

```text
https://siop-dev.seres.cn/argo/workflows/sacp-dms-argo/data-process-adc-12345-abcde
```

其中最后一段不是本系统任务 ID，而是分页接口返回的 `workflowName`。

> 重要说明：数据库字段使用下划线命名，Java 和接口 JSON 使用驼峰命名。因此数据库中的 `creator_employee_no`、`slice_name`、`create_time`，在分页响应中分别叫 `creatorEmployeeNo`、`sliceName`、`createTime`。这是项目现有接口的统一风格。

## 2. 先理解当前数据流

当前创建任务的主要链路是：

```text
业务代码
  └─ 构造 ArgoTaskSubmitDTO
       └─ ArgoTaskServiceImpl.submitAndCreateTask()
            ├─ 调用 Argo 创建 Workflow
            ├─ 将任务信息插入 argo_task
            └─ 转成 ArgoTaskVO 返回

分页查询
  └─ POST /argo/tasks/page
       └─ ArgoTaskMapper.selectPageList()
            └─ 查询 argo_task
                 └─ ArgoTaskServiceImpl.toVO()
                      └─ 返回 ArgoTaskVO
```

要让 `sliceName` 从业务代码进入数据库并最终返回页面，需要把整条链路补齐：

```text
DataLogicSlice.sliceName
  → ArgoTaskSubmitDTO.sliceName
  → ArgoTask.sliceName
  → argo_task.slice_name
  → ArgoTaskVO.sliceName
  → 分页响应 data.list[].sliceName
```

`creator`、`creatorEmployeeNo`、`createTime` 已经存在于 `DmsBaseDO`/`BaseDO` 中，不需要在 `ArgoTask` 实体里重复声明。它们目前没有出现在接口响应中，是因为 `ArgoTaskVO` 没有对应字段，并且 `toVO()` 没有复制这些值。

## 3. 字段规则

### 3.1 字段对应关系

| 含义 | 数据库字段 | Java 字段 | 分页 JSON 字段 | 可空 |
| --- | --- | --- | --- | --- |
| 创建人 | `creator` | `creator` | `creator` | 是 |
| 创建人员工编号 | `creator_employee_no` | `creatorEmployeeNo` | `creatorEmployeeNo` | 是 |
| 切片名称 | `slice_name` | `sliceName` | `sliceName` | 是 |
| 创建时间 | `create_time` | `createTime` | `createTime` | 否，正常数据应有值 |

### 3.2 不同任务类型的 `sliceName`

| 任务类型 | 是否有切片名称 | 赋值方式 |
| --- | --- | --- |
| `MANUAL_QC` | 有 | 使用本次逻辑切片的 `DataLogicSlice.sliceName` |
| `DATA_PROCESS` | 没有 | 不赋值，数据库保存 `NULL` |
| `EVENT_PROCESS` | 没有 | 不赋值，数据库保存 `NULL` |
| `CUSTOM` | 当前没有 | 不赋值；以后有业务需要时可通过 DTO 传入 |
| 重跑任务 | 与原任务一致 | 从原任务复制 `sliceName` |

不要把 `dataName` 当成 `sliceName`。前者是原始数据名称，后者是逻辑切片名称，两者业务含义不同。

### 3.3 创建人返回规则

本次需求是“返回表记录中的创建人”，所以接口只原样返回 `argo_task.creator` 和 `argo_task.creator_employee_no`：

- HTTP 请求携带有效 JWT 时，MyBatis 自动填充器会将当前操作人写入这两个字段。
- Kafka 等非 HTTP 场景没有 `CurrentUser`，当前实现下这两个字段可能为 `NULL`。
- 本次改造不把空创建人替换为 `SYSTEM`，也不从其他表临时查询创建人。
- 重跑任务会新建记录。有效 JWT 场景下，新记录的创建人是本次重跑操作人，不复制原任务创建人。

## 4. 后端改造步骤

### 4.1 新增数据库变更文件

不要修改已经执行过的 `0048` 文件。新增文件：

```text
sacp-dms-data-service/src/main/resources/db/changelog/sql/sacp-dms-data-server-0049-argo-task-slice-name.sql
```

内容如下：

```sql
-- liquibase formatted sql
-- changeset sxw:0049-1
-- comment: add slice name to argo task

ALTER TABLE `argo_task`
    ADD COLUMN `slice_name` VARCHAR(128) DEFAULT NULL COMMENT '逻辑切片名称'
        AFTER `data_name`;

-- changeset sxw:0049-2
-- comment: backfill slice name for existing manual qc argo tasks

UPDATE `argo_task` AS task
    INNER JOIN `data_logic_slice` AS logic_slice
        ON task.`task_type` = 'MANUAL_QC'
        AND task.`biz_id` = CAST(logic_slice.`id` AS CHAR)
SET task.`slice_name` = logic_slice.`slice_name`
WHERE task.`slice_name` IS NULL;
```

说明：

- 使用 `VARCHAR(128)`，与 `data_logic_slice.slice_name` 的长度保持一致。
- `NULL` 合法，因为原始数据解析和事件解析任务没有切片名称。
- 第一段 SQL 给表增加字段。
- 第二段 SQL 给历史 `MANUAL_QC` 任务补齐切片名称。该类任务的 `biz_id` 保存逻辑切片 ID，因此可以关联 `data_logic_slice.id`。
- `db.changelog-master.yaml` 已使用 `includeAll` 扫描 SQL 目录，不需要手工增加 include。
- 如果生产库的 `argo_task` 数据量很大，应先让 DBA 评估 `ALTER TABLE` 和历史回填的执行时间；必要时把回填拆成运维脚本分批执行。

### 4.2 修改任务实体 `ArgoTask`

修改文件：

```text
sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/entity/ArgoTask.java
```

在 `dataName` 后增加：

```java
private String dataName;

private String sliceName;

private String parameters;
```

项目已经开启数据库下划线到 Java 驼峰的映射，因此 `sliceName` 会自动对应 `slice_name`，不需要写 `@TableField("slice_name")`。

也不要在这个类中重复增加 `creator`、`creatorEmployeeNo`、`createTime`。`ArgoTask` 继承了 `DmsBaseDO`，这些属性已经从父类继承。

### 4.3 修改提交 DTO

修改文件：

```text
sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/dto/ArgoTaskSubmitDTO.java
```

在 `dataName` 后增加：

```java
private String dataName;

private String sliceName;

private Long retryFromTaskId;
```

增加 DTO 字段的原因是：`ArgoTaskService` 是统一的任务创建入口，它不能依赖具体的逻辑切片 Service。由调用方把已经知道的切片名称传进来，可以避免模块循环依赖和额外数据库查询。

### 4.4 创建任务时保存 `sliceName`

修改文件：

```text
sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/service/impl/ArgoTaskServiceImpl.java
```

在 `submitAndCreateTask()` 中增加一行：

```java
task.setDataId(dto.getDataId());
task.setDataName(dto.getDataName());
task.setSliceName(dto.getSliceName());
task.setParameters(JSONUtil.toJsonStr(dto.getParameters()));
```

这样 MyBatis 执行 `argoTaskMapper.insert(task)` 时，就会把值保存到 `argo_task.slice_name`。

### 4.5 手动质检任务传入切片名称

修改文件：

```text
sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/info/service/impl/DataLogicSliceServiceImpl.java
```

创建 `MANUAL_QC` 类型任务时，`entity` 就是刚创建的逻辑切片。在现有赋值代码后增加：

```java
submitDTO.setBizId(String.valueOf(entity.getId()));
submitDTO.setDataId(entity.getDataId());
submitDTO.setDataName(entity.getDataName());
submitDTO.setSliceName(entity.getSliceName());
```

其他任务入口不需要伪造切片名称：

- `DataUploadConsumer` 创建 `DATA_PROCESS` 任务，不增加 `setSliceName()`。
- `EventCompleteConsumer` 创建 `EVENT_PROCESS` 任务，不增加 `setSliceName()`。
- `IndexController` 创建测试 `CUSTOM` 任务，当前不增加 `setSliceName()`。

它们插入数据库时，`slice_name` 自然为 `NULL`。

### 4.6 重跑时复制原任务的切片名称

仍然修改 `ArgoTaskServiceImpl.retry()`。在构造 `retryDto` 时增加：

```java
retryDto.setDataId(oldTask.getDataId());
retryDto.setDataName(oldTask.getDataName());
retryDto.setSliceName(oldTask.getSliceName());
retryDto.setRetryFromTaskId(oldTask.getId());
```

必须复制该字段，否则原 `MANUAL_QC` 任务有切片名称，重跑后新任务的 `slice_name` 却会变成 `NULL`。

这里只复制切片名称，不复制 `creator` 和 `creatorEmployeeNo`。创建人仍由插入时的自动填充器写入，因此重跑新记录表示本次重跑操作人。

### 4.7 扩展返回 VO

修改文件：

```text
sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/vo/ArgoTaskVO.java
```

增加以下字段：

```java
private String dataName;

/** 逻辑切片名称；非切片任务为空。 */
private String sliceName;

/** 任务记录创建人；非 HTTP 场景可能为空。 */
private String creator;

/** 创建人员工编号；非 HTTP 场景可能为空。 */
private String creatorEmployeeNo;

/** 任务记录写入本系统的时间。 */
private LocalDateTime createTime;

private String parameters;
```

字段在类中的具体顺序不影响 JSON，但建议将业务展示字段放在 `parameters` 前，便于阅读。

因为列表、详情和重跑成功响应目前共用 `ArgoTaskVO`，增加字段后以下响应都会包含这些字段：

- `POST /argo/tasks/page`
- `GET /argo/tasks/{id}`
- `POST /argo/tasks/{id}/retry`

这不会改变已有字段，只是向响应中增加字段，属于向后兼容变更。

### 4.8 在 `toVO()` 中复制字段

只在 VO 中声明字段还不够。当前代码手工完成实体到 VO 的转换，因此必须同步修改 `ArgoTaskServiceImpl.toVO()`：

```java
vo.setDataId(task.getDataId());
vo.setDataName(task.getDataName());
vo.setSliceName(task.getSliceName());
vo.setCreator(task.getCreator());
vo.setCreatorEmployeeNo(task.getCreatorEmployeeNo());
vo.setCreateTime(task.getCreateTime());
vo.setParameters(task.getParameters());
```

这是最容易漏掉的一步。如果只改数据库和实体，SQL 能查到字段，但分页 JSON 仍然不会返回它们。

### 4.9 Mapper XML 是否需要修改

当前 `ArgoTaskMapper.xml` 的分页 SQL 是：

```xml
SELECT *
FROM argo_task
```

因此新增的 `slice_name` 以及已有的 `creator`、`creator_employee_no`、`create_time` 都会被查询出来，并自动映射到 `ArgoTask`，本次不需要修改 Mapper XML。

如果后续将 `SELECT *` 改成显式字段列表，必须把以下字段一起加入：

```sql
creator,
creator_employee_no,
slice_name,
create_time
```

## 5. 分页接口改造后的响应

接口不变：

```http
POST /argo/tasks/page
Content-Type: application/json
Authorization: Bearer <token>
```

请求示例：

```json
{
  "pageNum": 1,
  "pageSize": 10,
  "taskType": "MANUAL_QC"
}
```

响应示例：

```json
{
  "code": 0,
  "data": {
    "total": 1,
    "list": [
      {
        "id": "2080000000000000001",
        "workflowName": "sacp-dms-manual-qc-2079999999999999999-abcde",
        "templateName": "sacp-dms-manual-qc",
        "taskType": "MANUAL_QC",
        "bizId": "2079999999999999999",
        "dataId": "2079000000000000001",
        "dataName": "ADC_20260710_001",
        "sliceName": "ADC_20260710_001_103000_103030",
        "creator": "张三",
        "creatorEmployeeNo": "10001234",
        "createTime": 1783650000000,
        "parameters": "{\"slice_id\":\"2079999999999999999\"}",
        "generateName": "sacp-dms-manual-qc-2079999999999999999-",
        "status": "SUBMITTED",
        "message": null,
        "startTime": 1783650000000,
        "finishTime": null,
        "retryFromTaskId": null
      }
    ],
    "pageNum": 1,
    "pageSize": 10,
    "pages": 1
  },
  "msg": ""
}
```

注意：项目会按照现有 Jackson 配置序列化 `LocalDateTime`。当前前端接口文档约定时间为 Unix 毫秒时间戳，所以示例中的 `createTime` 是数字。实际联调时应以项目运行后的 JSON 为准，并和 `startTime` 的处理方式保持一致。

## 6. 前端跳转到 Argo Workflow 详情页

### 6.1 URL 组成规则

Argo 页面地址由三部分组成：

```text
Argo UI 基础地址 + /workflows/ + namespace + / + workflowName
```

开发环境示例：

```text
基础地址：https://siop-dev.seres.cn/argo
namespace：sacp-dms-argo
workflowName：sacp-dms-manual-qc-2079999999999999999-abcde

最终地址：
https://siop-dev.seres.cn/argo/workflows/sacp-dms-argo/sacp-dms-manual-qc-2079999999999999999-abcde
```

分页接口已经返回真实的 `workflowName`，所以为了实现页面跳转，不需要再增加后端接口，也不需要把 Argo Token 返回给浏览器。

### 6.2 前端增加环境配置

不同环境的 Argo 域名可能不同，不要在组件中直接写死 `https://siop-dev.seres.cn/argo`。

如果前端使用 Vite，可以在开发环境配置中增加：

```dotenv
VITE_ARGO_UI_BASE_URL=https://siop-dev.seres.cn/argo
VITE_ARGO_NAMESPACE=sacp-dms-argo
```

测试、预生产和生产环境分别填写各自的地址。如果项目使用 Vue CLI，可将前缀改成 `VUE_APP_`；如果使用其他构建工具，应按照该工具读取环境变量的方式调整。

### 6.3 封装 URL 构造函数

建议集中封装，避免每个页面重复拼接：

```ts
const ARGO_UI_BASE_URL = import.meta.env.VITE_ARGO_UI_BASE_URL;
const ARGO_NAMESPACE = import.meta.env.VITE_ARGO_NAMESPACE;

export function buildArgoWorkflowUrl(workflowName: string): string {
  if (!workflowName) {
    throw new Error('workflowName 不能为空');
  }
  if (!ARGO_UI_BASE_URL || !ARGO_NAMESPACE) {
    throw new Error('Argo UI 地址或 namespace 未配置');
  }

  const baseUrl = ARGO_UI_BASE_URL.replace(/\/+$/, '');
  return `${baseUrl}/workflows/${encodeURIComponent(ARGO_NAMESPACE)}/${encodeURIComponent(workflowName)}`;
}
```

这里有两个细节：

- `replace(/\/+$/, '')` 会删除基础地址末尾多余的 `/`，避免拼出双斜线。
- `encodeURIComponent()` 会对路径参数编码，防止特殊字符破坏 URL。

### 6.4 详情按钮点击事件

列表每一行使用该行的 `workflowName`：

```ts
function openWorkflowDetail(row: ArgoTaskVO) {
  if (!row.workflowName) {
    // 替换成项目统一的消息提示组件
    console.warn('该任务没有 workflowName，无法查看 Argo 详情');
    return;
  }

  const url = buildArgoWorkflowUrl(row.workflowName);
  window.open(url, '_blank', 'noopener,noreferrer');
}
```

按钮示例，具体组件名按前端项目调整：

```html
<button
  :disabled="!row.workflowName"
  @click="openWorkflowDetail(row)"
>
  详情
</button>
```

推荐新标签页打开，这样用户查看 Workflow 后仍能保留当前任务列表的页码和筛选条件。如果产品明确要求当前页跳转，可改为：

```ts
window.location.href = buildArgoWorkflowUrl(row.workflowName);
```

### 6.5 前端类型同步

前端的任务类型增加字段：

```ts
export interface ArgoTaskVO {
  id: string;
  workflowName: string;
  // 其他已有字段省略
  creator: string | null;
  creatorEmployeeNo: string | null;
  sliceName: string | null;
  createTime: number;
}
```

如果项目的时间字段实际被生成成字符串，应把 `createTime` 调整为 `string`，并复用现有 `startTime` 的格式化函数。

### 6.6 登录和权限说明

- 浏览器跳转 Argo UI 不涉及 AJAX，因此通常没有跨域请求问题。
- 不要把后端使用的 Argo Bearer Token拼到 URL，也不要保存在前端代码中。
- 用户能否看到 Workflow 由 Argo UI 自身的登录态和 RBAC 决定。
- 如果跳转后显示 Argo 登录页，这是正常的认证流程；如果登录后仍提示无权限，需要平台侧给用户开通 `sacp-dms-argo` namespace 的查看权限。

## 7. 文档同步

代码实现完成后同步更新：

1. `doc/mywork/argo-task-management/argo-task-frontend-api.md`
   - 在 `ArgoTaskVO` 字段表中增加 `creator`、`creatorEmployeeNo`、`sliceName`、`createTime`。
   - 更新分页响应示例。
   - 增加 Argo Workflow 详情页跳转规则。
2. `.hermes/api.md`
   - 更新 `POST /argo/tasks/page` 响应字段说明。
   - 因为共用 `ArgoTaskVO`，同时说明详情和重跑响应也增加了字段。
3. `.hermes/mcp.md`
   - 接口路径和入参没有变化，保留原有五列格式。
   - 可把分页接口的功能描述补充为“返回创建人、员工编号、切片名称和创建时间”，不要改变 params 列结构。

本次只是新增响应字段，没有新增接口路径，也没有破坏性变更，通常不需要更新 `.hermes/changelog.md`。

## 8. 验证步骤

### 8.1 编译检查

在仓库根目录执行：

```bash
mvn -pl sacp-dms-data-service -am -DskipTests compile
```

编译失败时优先检查：

- `ArgoTaskSubmitDTO` 是否已经增加 `sliceName`。
- `ArgoTask` 是否已经增加 `sliceName`。
- `ArgoTaskVO` 是否已经增加四个返回字段。
- `toVO()` 调用的 getter/setter 名称是否为驼峰形式。

### 8.2 数据库验证

服务启动、Liquibase 执行完成后检查字段：

```sql
SHOW COLUMNS FROM argo_task LIKE 'slice_name';
```

检查历史手动质检任务回填结果：

```sql
SELECT id,
       task_type,
       biz_id,
       creator,
       creator_employee_no,
       slice_name,
       create_time
FROM argo_task
WHERE task_type = 'MANUAL_QC'
ORDER BY create_time DESC
LIMIT 20;
```

预期：能够关联到 `data_logic_slice` 的历史 `MANUAL_QC` 任务具有 `slice_name`。

### 8.3 新建手动质检任务验证

通过现有逻辑切片创建接口创建一条 `needQc=true` 且带 `sliceName` 的切片，然后执行：

```sql
SELECT id, biz_id, data_name, slice_name
FROM argo_task
WHERE task_type = 'MANUAL_QC'
ORDER BY create_time DESC
LIMIT 1;
```

预期：`slice_name` 等于创建逻辑切片时使用的名称。

### 8.4 分页响应验证

```bash
curl -X POST 'http://localhost:20001/sacp-dms-data-server/argo/tasks/page' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer 替换成有效token' \
  -d '{"pageNum":1,"pageSize":10,"taskType":"MANUAL_QC"}'
```

检查 `data.list[0]` 是否包含：

```json
{
  "creator": "张三",
  "creatorEmployeeNo": "10001234",
  "sliceName": "切片名称",
  "createTime": 1783650000000
}
```

这里验证的是“接口值与表记录一致”。如果数据库中的创建人本来就是 `NULL`，接口返回 `null` 是当前设计下的正确结果。

### 8.5 重跑验证

找一条状态为 `FAILED` 或 `ERROR`、并且 `slice_name` 不为空的任务，调用：

```bash
curl -X POST 'http://localhost:20001/sacp-dms-data-server/argo/tasks/替换成任务ID/retry' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer 替换成有效token' \
  -d '{"message":"验证切片名称复制"}'
```

预期：

- 新任务的 `retryFromTaskId` 是旧任务 ID。
- 新任务的 `sliceName` 与旧任务一致。
- 新任务的 `creator` 是本次重跑操作人，而不是旧任务创建人。

### 8.6 前端跳转验证

在浏览器中完成以下检查：

1. 任务列表每一行显示“详情”按钮。
2. 点击某条记录后，打开的新地址符合：

```text
https://siop-dev.seres.cn/argo/workflows/sacp-dms-argo/{workflowName}
```

3. URL 中的 `{workflowName}` 与该行分页数据的 `workflowName` 完全一致。
4. 返回任务列表后，原来的页码和筛选条件仍然保留。
5. `workflowName` 为空时按钮不可用或给出明确提示。
6. 未登录 Argo 时进入登录页；登录且有权限后能看到对应 Workflow。

## 9. 常见错误

### 9.1 修改旧的 `0048` SQL

已经被 Liquibase 执行过的 changeset 不应直接修改，否则可能产生 checksum 校验失败。必须新增 `0049` 文件。

### 9.2 只改 VO，不改 `toVO()`

当前项目不是自动 Bean Copy。只给 `ArgoTaskVO` 增加字段会导致响应字段存在但值始终为 `null`，必须手工调用对应 setter。

### 9.3 重跑没有复制 `sliceName`

重跑会创建新记录，不会自动复制旧实体的所有字段。必须显式执行：

```java
retryDto.setSliceName(oldTask.getSliceName());
```

### 9.4 把数据库字段名直接当 JSON 字段名

前端应读取：

```ts
row.creatorEmployeeNo
row.sliceName
row.createTime
```

不要读取：

```ts
row.creator_employee_no
row.slice_name
row.create_time
```

除非后端显式使用 `@JsonProperty` 改变 JSON 命名。本方案不建议这样做，因为会破坏项目现有驼峰风格。

### 9.5 使用本系统任务 ID 拼 Argo URL

错误示例：

```text
/argo/workflows/sacp-dms-argo/2080000000000000001
```

正确做法是使用 `workflowName`：

```text
/argo/workflows/sacp-dms-argo/sacp-dms-manual-qc-2079999999999999999-abcde
```

### 9.6 在前端硬编码 Token

前端只负责打开 Argo UI 页面。后端调用 Argo API 使用的 Token 属于服务端密钥，严禁放到浏览器代码、环境文件或跳转 URL 中。

## 10. 完成标准

以下条件全部满足，才算本次需求完成：

- `argo_task` 表存在可空的 `slice_name VARCHAR(128)` 字段。
- 新建 `MANUAL_QC` 任务能够保存逻辑切片名称。
- 重跑任务能够保留原任务的切片名称。
- 分页响应包含 `creator`、`creatorEmployeeNo`、`sliceName`、`createTime`。
- 响应中的四个字段与 `argo_task` 表对应记录一致。
- 原始数据解析和事件解析任务允许 `sliceName=null`。
- 前端使用该行的 `workflowName` 构造 Argo 详情地址。
- Argo UI 地址和 namespace 通过环境配置提供，没有硬编码服务端 Token。
- 后端编译通过，数据库、分页、重跑和前端跳转均完成验证。
- 前端接口文档和 `.hermes/` 接口知识库已经同步更新。
