# Argo 任务管理能力面试亮点

总览：纳入本系统任务生命周期    Argo回调    回调幂等    定时轮询补偿   重跑新建任务复用参数    观测workflow、模板、参数、失败原因

## 简历一行

设计并落地 Argo 异步任务管理闭环，将原本不可追踪的 Workflow 改造成可观测、可补偿、可重跑的任务生命周期系统。

## 结论

这个技术实现可以作为项目亮点介绍。

它满足“项目亮点”的两个条件：

1. 面试官容易感兴趣：它不是普通 CRUD，而是围绕外部异步任务平台 Argo 做任务生命周期管理，涉及异步调用、状态一致性、失败补偿、幂等、重跑、可观测性等后端常见高频话题。
2. 有很多可深挖点：可以继续问数据库设计、回调幂等、为什么需要轮询补偿、重跑为什么新建任务、如何避免重复更新、Argo WorkflowTemplate 怎么改、定时任务并发问题、接口鉴权边界等。

注意：面试时不要把它说成“我做了一个任务列表页面”。应该说成“我把原来不可追踪的 Argo 异步任务，改造成了本系统内可查询、可补偿、可重跑的任务管理闭环”。

## 面试亮点具体介绍

总览：独立模块    传模板名、参数、完整任务上下文    argo_task表    onExit exit-handler    workflow结束回调    定时补偿任务    任务重跑新建任务记录

### 1. 一句话版本

我负责设计并落地了 Argo 异步任务管理能力。原来系统提交 Argo Workflow 后，只拿到一个 workflowName，后续任务执行状态、失败原因、是否能重跑都无法在本系统追踪。我通过新增 `argo_task` 任务表、统一提交上下文、Argo 结束回调、幂等状态更新、失败重跑和定时轮询补偿，把外部 Argo Workflow 纳入了本系统的任务生命周期管理。

### 2. 两分钟版本

项目里有多个业务流程会拉起 Argo Workflow，比如原始数据解析、事件解析、手动质检。原来的问题是：系统只负责把任务提交给 Argo，拿到 `metadata.name` 后就结束了。本系统没有任务记录，也不知道任务后面是成功还是失败，失败后也不能基于原参数重新提交。

我做的改造是把 Argo 任务抽象成一个独立模块 `module.argo`。每次业务提交 Argo 时，不再只传模板名和参数，而是传一个完整的任务上下文，包括 `templateName`、`taskType`、`bizId`、`dataId`、`dataName`、`parameters`、`generateName`。Argo 返回真实 `workflowName` 后，系统把这些信息写入 `argo_task` 表，状态初始化为 `SUBMITTED`。

状态维护上，我在 WorkflowTemplate 里增加 `onExit` exit handler，通过 HTTP Template 在 Workflow 结束时回调本系统 `/argo/tasks/callback`。后端根据 `workflowName` 找到任务记录，把状态更新为 `SUCCEEDED` 或 `FAILED`，同时保存失败原因。回调接口做了幂等处理，如果任务已经是终态，重复回调会直接忽略。

考虑到回调可能失败，我又设计了定时补偿任务，扫描长时间处于 `SUBMITTED`、`RETRY_SUBMITTED` 的任务，调用 Argo API 查询真实 Workflow 状态，再补偿更新本地任务状态。这样即使 Argo 回调因为网络、服务重启或接口异常失败，本系统最终也能修正任务状态。

另外，失败任务支持重跑。重跑时不会覆盖旧任务，而是读取旧任务保存的 `templateName` 和 `parameters` 重新提交 Argo，并新增一条任务记录，用 `retryFromTaskId` 关联旧任务。这样页面上既能看到失败历史，也能看到重跑链路。

### 3. 可以强调的技术点

- 外部异步任务平台接入：把 Argo Workflow 纳入本系统任务生命周期。
- 状态一致性：本地状态由 Argo 回调和定时轮询共同维护。
- 幂等设计：重复回调不会重复修改终态任务。
- 失败补偿：回调失败后通过主动轮询 Argo API 修正状态。
- 可重跑设计：重跑新建任务，不覆盖旧任务，保留完整历史。
- 可观测性：保存 `workflowName`、模板、参数、业务 ID、失败原因和时间。
- 扩展性：任务类型抽象为 `DATA_PROCESS`、`EVENT_PROCESS`、`MANUAL_QC`、`CUSTOM`，后续新 Argo 模板可以复用同一套任务管理能力。

## 推荐面试表达结构

### 背景

我们系统会调用 Argo Workflow 做数据解析、事件解析和质检，这些任务都是异步执行的。原来系统提交任务后只拿到了 workflowName，没有统一任务记录，页面也无法展示执行状态。

### 问题

主要问题有三个：

1. 不可观测：不知道每次 Argo 任务当前是运行中、成功还是失败。
2. 不可追溯：失败时不知道当时提交了哪些参数、失败原因是什么。
3. 不可恢复：失败后不能基于原参数重跑，只能人工重新构造请求。

### 方案

我新增了 `argo_task` 任务表和 `module.argo` 模块。提交 Argo 成功后记录任务；Workflow 结束时通过 Argo `onExit` 回调本系统；回调失败时由定时任务主动查询 Argo 状态补偿；失败任务重跑时复用旧参数重新提交，并生成一条新任务记录。

### 结果

改造后，业务侧所有 Argo 任务都能在本系统查询状态、参数、失败原因和重跑来源。失败任务支持一键重跑，并且即使回调丢失，也能通过轮询补偿把任务状态修正回来。

## 可以深挖的问题和对应回答

### 1. 为什么要新增 `argo_task` 表，不能直接查 Argo 吗？

不能只依赖 Argo。

Argo 只知道 Workflow 本身，不知道这个任务在业务系统里属于哪个业务类型、关联哪个 `dataId`、哪个 `eventInfoId`、哪个 `sliceId`，也不知道页面要展示的原始数据名称。我们需要把 Argo 的 `workflowName` 和业务上下文绑定起来，所以要有本地任务表。

另外，本地表可以保存提交参数和重跑来源，这些对失败排查和重跑非常关键。

### 2. `workflowName` 为什么要唯一？

因为回调时 Argo 只会告诉我们当前结束的是哪个 Workflow，最稳定的关联字段就是 `workflowName`。

所以 `argo_task.workflow_name` 建唯一索引，保证一个 Argo Workflow 只能对应一条本地任务记录。这样回调更新时不会出现一条回调命中多条任务的问题。

### 3. 为什么任务主键不用数据库自增？

项目里已有实体使用 MyBatis-Plus 的 `IdType.ASSIGN_ID`，也就是插入前由 MyBatis-Plus 通过雪花算法生成 Long 类型 ID。

所以 `argo_task.id` 不需要 `AUTO_INCREMENT`，实体里配置：

```java
@TableId(type = IdType.ASSIGN_ID)
private Long id;
```

这和项目现有 `DataInfo` 等实体风格保持一致。

### 4. 回调接口如何保证幂等？

回调接口根据 `workflowName` 查询本地任务。如果任务已经是终态，比如 `SUCCEEDED`、`FAILED`、`ERROR`，就直接返回成功，不再重复更新。

这样即使 Argo 或网络层重复发送回调，也不会导致状态反复变化。

```java
if (ArgoTaskStatusEnum.isFinished(task.getStatus())) {
    log.info("Argo 任务已是终态，忽略重复回调, taskId={}, workflowName={}, currentStatus={}",
             task.getId(), task.getWorkflowName(), task.getStatus());
    return;
}
String normalizedStatus = normalizeStatus(dto.getStatus());
task.setStatus(normalizedStatus);
task.setMessage(dto.getMessage());
task.setFinishTime(dto.getFinishTime() == null ? LocalDateTime.now() : dto.getFinishTime());
argoTaskMapper.updateById(task);
```



### 5. 如果回调失败怎么办？

我没有把系统正确性完全依赖在回调上，而是增加了主动补偿机制。

定时任务会扫描长时间处于 `SUBMITTED`、`RETRY_SUBMITTED` 的任务，比如提交超过 5 分钟还没收到回调，就调用 Argo API 查询真实 Workflow 状态。如果 Argo 显示已经 `Succeeded`、`Failed` 或 `Error`，就复用回调的状态更新逻辑补偿本地任务。

这样即使回调因为网络抖动、服务重启、接口 500 等原因失败，最终状态也能被修正。

### 6. 为什么查询 Argo 失败时不立刻把本地任务改成 `ERROR`？

因为查询失败不一定代表任务失败，可能只是本系统到 Argo 的网络临时异常，或者 Argo API 短暂不可用。

如果一查失败就把任务标记为 `ERROR`，会产生误判。所以第一版设计是查询失败只打日志，等待下一轮定时任务继续补偿。只有连续多次 404 或确认 Workflow 已不存在时，才考虑标记为 `ERROR`。

### 7. 为什么重跑要新增任务，而不是更新旧任务？

重跑本质上是一次新的 Argo Workflow，Argo 会生成新的 `workflowName`，执行结果也可能不同。

如果覆盖旧任务，就会丢失第一次失败的现场，包括失败原因、提交时间、旧 workflowName。新增任务并用 `retryFromTaskId` 关联旧任务，可以保留完整历史，也方便页面展示“这次重跑来自哪次失败任务”。

### 8. 重跑时参数从哪里来？

第一次提交 Argo 时，系统会把完整参数保存到 `argo_task.parameters`，比如 `s3_path`、`data_id`、`files`、`slice_id`、`file_list`。

重跑时读取旧任务的 `templateName` 和 `parameters`，重新调用 Argo submit API。这样不需要用户重新拼参数，也避免人工构造参数出错。

### 9. 为什么把任务类型设计成字符串枚举？

`taskType` 用 `DATA_PROCESS`、`EVENT_PROCESS`、`MANUAL_QC` 这种字符串，排查问题时更直观。

如果用数字，数据库里看到 `1`、`2`、`3` 还要查枚举含义。字符串枚举虽然稍微占空间，但对运维排查、页面展示和接口调试更友好。

### 10. Argo WorkflowTemplate 做了什么改造？

在 WorkflowTemplate 里增加 `onExit: exit-handler`。

`onExit` 的特点是 Workflow 成功或失败都会执行，所以适合做最终状态上报。`exit-handler` 使用 HTTP Template 调本系统：

```yaml
http:
  url: "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback"
  method: "POST"
```

请求体里传 `workflowName`、`status` 和失败信息。本系统根据 `workflowName` 更新任务状态。

### 11. 为什么不用业务容器自己回调，而是在 WorkflowTemplate 里加 `onExit`？

业务容器只代表某一个步骤，不能保证整个 Workflow 结束时一定能上报。

比如 Workflow 里有多个 DAG 节点，某个节点失败后，业务代码可能没有机会执行回调。`onExit` 是 Workflow 级别的退出处理，更适合统一上报最终状态。

### 12. 为什么还需要 `generateName`？

`generateName` 用来控制 Argo 生成 Workflow 名称的前缀。

比如数据解析可以生成 `data-process-adc-{dataId}-xxxxx`，事件解析可以生成 `event-process-{eventInfoId}-xxxxx`。这样在 Argo 页面和日志里看到 workflowName 时，就能大致知道它属于哪个业务对象，排查更方便。

### 13. 本地状态和 Argo 状态如何映射？

Argo 常见状态有 `Succeeded`、`Failed`、`Error`、`Running`、`Pending`。

本地只把终态落成最终状态：

- `Succeeded` -> `SUCCEEDED`
- `Failed` -> `FAILED`
- `Error` -> `FAILED` 或 `ERROR`，具体可以根据业务定义区分
- `Running`、`Pending` -> 本地保持 `SUBMITTED` 或 `RETRY_SUBMITTED`

这样本地任务表主要表达业务关心的状态，不需要完全复制 Argo 的所有内部状态。

### 14. 定时任务会不会和回调并发更新同一条任务？

有可能，所以更新逻辑要幂等。

无论是回调还是定时补偿，最终都走同一个 `handleCallback` 状态更新方法。这个方法先判断任务是否已经是终态，如果已经是终态就直接返回。这样即使回调和定时任务同时处理同一个 Workflow，也不会造成错误状态覆盖。

如果后续并发压力更高，可以进一步在 SQL 更新时加状态条件，例如只允许从 `SUBMITTED`、`RETRY_SUBMITTED` 更新到终态。

```
会出现“多个请求都查询到任务不是终态，都执行了SQL更新”的情况：先查状态 -> 判断不是终态 -> 再 updateById 不是严格并发安全的。两个请求可能同时读到 SUBMITTED，然后都执行更新：
  - 回调线程读到 SUBMITTED
  - 定时补偿线程也读到 SUBMITTED
  - 两边都认为可以更新
  - 两边都执行 SQL
如果两边更新成同一个终态，比如都是 SUCCEEDED，问题不大，只是重复更新。但如果一个更新成 SUCCEEDED，另一个因为查询时机或异常信息更新成 FAILED，就可能出现终态被覆盖。
解决方式：不要只靠 Java 层判断，要把状态条件放进 SQL 的 WHERE 里，做条件更新。
推荐写法：
  int updated = argoTaskMapper.update(null,
          Wrappers.<ArgoTask>lambdaUpdate()
                  .eq(ArgoTask::getWorkflowName, dto.getWorkflowName())
                  .in(ArgoTask::getStatus,
                          ArgoTaskStatusEnum.SUBMITTED.getCode(),
                          ArgoTaskStatusEnum.RETRY_SUBMITTED.getCode())
                  .set(ArgoTask::getStatus, normalizedStatus)
                  .set(ArgoTask::getMessage, dto.getMessage())
                  .set(ArgoTask::getFinishTime,
                          dto.getFinishTime() == null ? LocalDateTime.now() : dto.getFinishTime()));

  if (updated == 0) {
      log.info("Argo 任务已被其他线程更新，忽略本次回调, workflowName={}", dto.getWorkflowName());
  }
核心点是：
	WHERE workflow_name = ?
    	AND status IN ('SUBMITTED', 'RETRY_SUBMITTED')
这样只有第一个请求能更新成功。第二个请求即使之前查到不是终态，执行 SQL 时也会因为状态已经变成终态而 updated = 0，不会覆盖结果。
它确保的是：多个 SQL 即使并发执行，也只有一个能真正更新成功。过程是：
  1. 请求 A 执行 UPDATE，命中 status = SUBMITTED，拿到这行的行锁。
  2. 请求 B 也执行 UPDATE，但同一行被 A 锁住，只能等待。
  3. A 更新成功，把状态改成 SUCCEEDED，提交事务，释放锁。
  4. B 继续执行时，会重新检查 WHERE 条件。
  5. 此时状态已经是 SUCCEEDED，不满足 status IN ('SUBMITTED', 'RETRY_SUBMITTED')。
  6. B 更新 0 行。
```



### 15. 定时任务扫描范围怎么控制？

只扫描：

- `status IN ('SUBMITTED', 'RETRY_SUBMITTED')`
- `start_time <= 当前时间 - 超时分钟数`
- 单批限制 `LIMIT 50`

这样不会扫描已经结束的任务，也不会扫描刚提交的任务，并且能控制对数据库和 Argo API 的压力。

### 16. 回调接口为什么不鉴权？安全吗？

Argo 在集群内调用本系统接口，通常不方便带用户登录态，所以回调接口需要从 JWT 鉴权里排除。

但这不代表完全不做安全控制。可以通过以下方式增强：

1. 只暴露集群内 Service 地址，不对公网暴露。
2. 增加内部 token header，例如 `X-Argo-Callback-Token`。
3. 校验 `workflowName` 必须存在于本地任务表。
4. 只允许从非终态更新到终态。

第一版方案里至少做到不鉴权但按 `workflowName` 精确匹配任务，并且幂等更新。

### 17. 如果先提交 Argo 成功，但插入本地任务失败怎么办？

这是一个需要注意的一致性问题。

当前方案是先提交 Argo，拿到 `workflowName` 后插入本地任务。如果插入数据库失败，就会出现 Argo 已经运行但本地没有记录的问题。

可以优化为：

1. 提交失败直接返回，不插入任务。
2. 提交成功但插入失败时，记录错误日志并告警。
3. 更进一步，可以先创建本地 `CREATED` 任务，再提交 Argo，成功后回填 `workflowName` 和 `SUBMITTED`。但这样要处理本地创建成功、Argo 提交失败的状态。

面试中可以说明：第一版为了保证只有真实提交成功的 Workflow 才进入任务表，采用先提交后入库；如果对一致性要求更高，可以演进成“本地预创建 + 提交后回填”的模式。

### 18. 为什么没有直接把 Argo 状态实时同步到本地？

没必要实时同步所有中间态。

页面最关心的是任务是否已提交、是否成功、是否失败、失败原因和能否重跑。`Running`、`Pending` 这类状态可以统一显示为已提交或执行中。

如果未来页面需要展示更细粒度进度，可以扩展字段保存 Argo 原始 phase，或者新增任务状态历史表。

### 19. 这个设计后续怎么扩展？

可以从几个方向扩展：

1. 增加任务状态历史表，记录每次状态变化。
2. 增加任务操作日志，记录谁触发了重跑。
3. 增加回调 token，提升内部接口安全性。
4. 增加 Argo 日志链接，页面可直接跳到 Argo Workflow。
5. 增加重跑参数编辑能力，允许用户基于旧参数微调后重跑。
6. 增加连续查询失败次数，超过阈值再标记 `ERROR`。

```
增加连续查询失败次数，超过阈值再标记 `ERROR`。
    在 argo_task 表增加字段：
    ALTER TABLE argo_task
        ADD COLUMN `sync_fail_count` INT NOT NULL DEFAULT 0 COMMENT '状态补偿连续查询失败次数',
        ADD COLUMN `last_sync_error` TEXT DEFAULT NULL COMMENT '最近一次状态补偿失败原因';
    每次定时任务查询 Argo：
      - 查询成功：把 sync_fail_count 清零。
      - 查询失败：sync_fail_count + 1，保存 last_sync_error。
      - 如果失败次数达到阈值，比如 10 次，才把任务标记为 ERROR。
```

```java
    for (ArgoTask task : tasks) {//sync_fail_count小于10才会进入tasks
        try {
            ArgoWorkflowStatusVO workflowStatus = argoWorkflowClient.getWorkflowStatus(task.getWorkflowName());
            if (!isArgoFinished(workflowStatus.getPhase())) {
                continue;
            }

            ArgoTaskCallbackDTO callbackDTO = new ArgoTaskCallbackDTO();
            callbackDTO.setWorkflowName(task.getWorkflowName());
            callbackDTO.setStatus(workflowStatus.getPhase());
            callbackDTO.setMessage(workflowStatus.getMessage());
            callbackDTO.setFinishTime(workflowStatus.getFinishedAt());
            handleCallback(callbackDTO);
            updatedCount++;
        } catch (Exception e) {
            log.error("补偿同步 Argo 任务状态失败, taskId={}, workflowName={}",
                    task.getId(), task.getWorkflowName(), e);
            //增加sync_fail_count,若为10,则标记ERROR
        }
    }
```



### 20. 这个亮点体现了哪些后端能力？

它体现的不只是会写接口，而是后端系统设计能力：

- 能识别外部异步系统接入后的状态追踪问题。
- 能设计本地任务模型承接外部任务生命周期。
- 能处理回调重复、回调丢失、外部 API 查询失败等异常场景。
- 能保留失败现场并支持重跑。
- 能在业务可观测性和系统复杂度之间做取舍。

## 面试时不建议这么说

不要说：

“我做了一个 Argo 任务管理页面，可以查询任务和重跑。”

这个说法太像普通后台管理 CRUD。

建议说：

“我把原来只提交不追踪的 Argo 异步任务，设计成了本系统内可观测、可补偿、可重跑的任务生命周期闭环。核心是本地任务表承接业务上下文，Argo onExit 回调更新终态，定时轮询补偿回调丢失，失败重跑新建任务并保留历史链路。”

## 30 秒精简背诵版

我做过一个 Argo 异步任务管理改造。原来系统提交 Argo 后只拿到 workflowName，后续状态、失败原因和重跑都无法在本系统管理。我新增了 `argo_task` 任务表，把 workflowName、模板名、任务类型、业务 ID、提交参数和状态都记录下来；WorkflowTemplate 通过 `onExit` 回调本系统更新成功或失败；回调接口做了幂等；如果回调失败，定时任务会扫描长时间未完成的任务，主动查 Argo API 补偿状态；失败任务重跑时不会覆盖旧记录，而是复用旧参数重新提交并生成新任务，用 `retryFromTaskId` 保留重跑链路。这个改造解决了外部异步任务不可观测、不可追溯、不可恢复的问题。
