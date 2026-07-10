# Argo 任务管理实现方案

## 1. 目标和整体思路

本次改造要让系统在每次提交 Argo Workflow 后，都能在本系统保存一条任务记录，并通过 Argo 回调维护任务状态。页面以后只需要查本系统的任务表，就能看到任务列表、任务详情、失败原因，并能对失败任务重新提交。

整体链路如下：

1. 业务代码准备 Argo 参数，例如 `data_id`、`s3_path`、`files`、`slice_id`。
2. 业务代码调用 `ArgoWorkflowClient` 提交 Argo。
3. Argo 返回真实 Workflow 名称，也就是 `metadata.name`。
4. 系统把 `workflowName`、模板名、任务类型、业务 ID、参数 JSON 保存到 `argo_task` 表。
5. WorkflowTemplate 增加 `onExit`，Workflow 结束时调用 `POST /argo/tasks/callback`。
6. 回调接口根据 `workflowName` 找到任务记录，更新为 `SUCCEEDED` 或 `FAILED`。
7. 页面查询 `/argo/tasks/page`、`/argo/tasks/{id}` 展示任务，失败任务调用 `/argo/tasks/{id}/retry` 重新提交。

关键原则：重跑必须新增一条任务记录，不能覆盖旧记录。旧记录保留失败原因，新记录通过 `retryFromTaskId` 指向旧记录。

## 2. 当前项目中要改的位置

后端代码入口：

- `sacp-dms-data-service/src/main/java/com/seres/sacp/dms/client/ArgoWorkflowClient.java`
- `sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/info/mq/DataUploadConsumer.java`
- `sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/event/mq/EventCompleteConsumer.java`
- `sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/info/service/impl/DataLogicSliceServiceImpl.java`
- `sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/common/controller/IndexController.java`
- `sacp-dms-data-service/src/main/java/com/seres/sacp/dms/config/WebMvcConfig.java`

Argo 模板参考：

- `doc/mywork/argo-task-management/workflow-template/sacp-dms-data-process-with-qc.yaml`
- `doc/mywork/argo-task-management/workflow-template/sacp-dms-event-parse.yaml`
- `doc/mywork/argo-task-management/workflow-template/sacp-dms-manual-qc.yaml`

## 3. 第一步：新增数据库表

新增文件：`sacp-dms-data-service/src/main/resources/db/changelog/sql/sacp-dms-data-server-0048-argo-task.sql`

```sql
-- liquibase formatted sql
-- changeset sxw:0048
-- comment: add argo task management table

CREATE TABLE IF NOT EXISTS `argo_task` (
                                           `id` BIGINT NOT NULL COMMENT '主键ID',
                                           `workflow_name` VARCHAR(128) NOT NULL COMMENT 'Argo Workflow 名称',
    `template_name` VARCHAR(128) NOT NULL COMMENT 'WorkflowTemplate 名称',
    `task_type` VARCHAR(32) NOT NULL COMMENT '任务类型：DATA_PROCESS、EVENT_PROCESS、MANUAL_QC、CUSTOM',
    `biz_id` VARCHAR(64) DEFAULT NULL COMMENT '业务对象ID，例如 dataId、eventInfoId、sliceId',
    `data_id` BIGINT DEFAULT NULL COMMENT '原始数据ID',
    `data_name` VARCHAR(255) DEFAULT NULL COMMENT '原始数据名称',
    `parameters` TEXT NOT NULL COMMENT '提交给 Argo 的参数 JSON',
    `generate_name` VARCHAR(128) DEFAULT NULL COMMENT '提交 Argo 时使用的 generateName',
    `status` VARCHAR(32) NOT NULL COMMENT '任务状态：SUBMITTED、SUCCEEDED、FAILED、ERROR、RETRY_SUBMITTED',
    `message` TEXT DEFAULT NULL COMMENT '失败原因或回调说明',
    `start_time` DATETIME NOT NULL COMMENT '提交时间',
    `finish_time` DATETIME DEFAULT NULL COMMENT '完成时间',
    `retry_from_task_id` BIGINT DEFAULT NULL COMMENT '重跑来源任务ID',
    `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    `deleted` TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0-未删除，1-已删除',
    `creator` VARCHAR(64) DEFAULT NULL COMMENT '创建人',
    `updater` VARCHAR(64) DEFAULT NULL COMMENT '更新人',
    `creator_employee_no` VARCHAR(64) DEFAULT NULL COMMENT '创建人员工编号',
    `updater_employee_no` VARCHAR(64) DEFAULT NULL COMMENT '更新人员工编号',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_argo_task_workflow_name` (`workflow_name`),
    KEY `idx_argo_task_type_status` (`task_type`, `status`),
    KEY `idx_argo_task_data_id` (`data_id`),
    KEY `idx_argo_task_biz_id` (`biz_id`),
    KEY `idx_argo_task_retry_from` (`retry_from_task_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Argo任务记录表';
```
注释：项目的 `db.changelog-master.yaml` 已经使用 `includeAll` 扫描 `db/changelog/sql/`，所以新增 `0048` SQL 后会自动执行。`workflow_name` 必须唯一，因为回调只拿 workflow 名称来定位任务。

## 4. 第二步：新增 argo 模块基础类

### 4.1 任务类型枚举

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/enums/ArgoTaskTypeEnum.java`

```java
package com.seres.sacp.dms.module.argo.enums;

import lombok.Getter;
import lombok.RequiredArgsConstructor;

@Getter
@RequiredArgsConstructor
public enum ArgoTaskTypeEnum {

    DATA_PROCESS("DATA_PROCESS", "原始数据解析"),
    EVENT_PROCESS("EVENT_PROCESS", "事件解析"),
    MANUAL_QC("MANUAL_QC", "手动质检"),
    CUSTOM("CUSTOM", "手动提交");

    private final String code;
    private final String desc;
}
```
注释：`taskType` 用字符串保存，比数字更适合排查问题；数据库里看到 `DATA_PROCESS` 就知道是哪类任务。

### 4.2 任务状态枚举

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/enums/ArgoTaskStatusEnum.java`

```java
package com.seres.sacp.dms.module.argo.enums;

import lombok.Getter;
import lombok.RequiredArgsConstructor;

@Getter
@RequiredArgsConstructor
public enum ArgoTaskStatusEnum {

    SUBMITTED("SUBMITTED", "已提交"),
    RETRY_SUBMITTED("RETRY_SUBMITTED", "重跑已提交"),
    SUCCEEDED("SUCCEEDED", "成功"),
    FAILED("FAILED", "失败"),
    ERROR("ERROR", "异常");

    private final String code;
    private final String desc;

    public static boolean isFinished(String status) {
        return SUCCEEDED.code.equals(status) || FAILED.code.equals(status) || ERROR.code.equals(status);
    }
}
```
注释：`isFinished` 用于回调幂等。如果任务已经是终态，重复回调时直接返回成功，不重复改数据。

### 4.3 任务实体

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/entity/ArgoTask.java`

```java
package com.seres.sacp.dms.module.argo.entity;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.seres.sacp.framework.mybatis.core.dataobject.DmsBaseDO;
import lombok.Data;
import lombok.EqualsAndHashCode;

import java.time.LocalDateTime;

@Data
@EqualsAndHashCode(callSuper = true)
@TableName("argo_task")
public class ArgoTask extends DmsBaseDO {

    @TableId(type = IdType.ASSIGN_ID)//IdType.ASSIGN_ID表示插入前由 MyBatis-Plus 使用雪花算法生成 Long 类型 ID，然后再写入数据库
    private Long id;

    private String workflowName;

    private String templateName;

    private String taskType;

    private String bizId;

    private Long dataId;

    private String dataName;

    private String parameters;

    private String generateName;

    private String status;

    private String message;

    private LocalDateTime startTime;

    private LocalDateTime finishTime;

    private Long retryFromTaskId;
}
```
注释：字段名使用驼峰，MyBatis-Plus 会按项目配置映射到下划线字段，例如 `workflowName` 对应 `workflow_name`。

### 4.4 Mapper

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/mapper/ArgoTaskMapper.java`

```java
package com.seres.sacp.dms.module.argo.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.seres.sacp.dms.module.argo.entity.ArgoTask;
import com.seres.sacp.dms.module.argo.query.ArgoTaskQuery;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

@Mapper
public interface ArgoTaskMapper extends BaseMapper<ArgoTask> {

    List<ArgoTask> selectPageList(@Param("query") ArgoTaskQuery query);
}
```
注释：简单增删改查用 `BaseMapper`，列表页有多条件模糊查询，所以单独写 XML。

新增文件：`sacp-dms-data-service/src/main/resources/mapper/argo/ArgoTaskMapper.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.seres.sacp.dms.module.argo.mapper.ArgoTaskMapper">

    <select id="selectPageList" resultType="com.seres.sacp.dms.module.argo.entity.ArgoTask">
        SELECT *
        FROM argo_task
        <where>
            deleted = 0
            <if test="query.taskType != null and query.taskType != ''">
                AND task_type = #{query.taskType}
            </if>
            <if test="query.status != null and query.status != ''">
                AND status = #{query.status}
            </if>
            <if test="query.dataId != null">
                AND data_id = #{query.dataId}
            </if>
            <if test="query.workflowName != null and query.workflowName != ''">
                AND workflow_name LIKE CONCAT('%', #{query.workflowName}, '%')
            </if>
            <if test="query.templateName != null and query.templateName != ''">
                AND template_name LIKE CONCAT('%', #{query.templateName}, '%')
            </if>
        </where>
        ORDER BY create_time DESC
    </select>
</mapper>
```
注释：列表页默认按创建时间倒序，最新任务显示在最前面。`workflowName` 和 `templateName` 使用模糊匹配，方便页面搜索。

## 5. 第三步：新增 DTO、Query、VO

### 5.1 提交上下文 DTO

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/dto/ArgoTaskSubmitDTO.java`

```java
package com.seres.sacp.dms.module.argo.dto;

import lombok.Data;

import java.util.Map;

@Data
public class ArgoTaskSubmitDTO {

    private String templateName;

    private Map<String, String> parameters;

    private String generateName;

    private String taskType;

    private String bizId;

    private Long dataId;

    private String dataName;

    private Long retryFromTaskId;
}
```
注释：这个类把“提交 Argo 需要的信息”和“任务管理需要记录的信息”放在一起，业务代码以后只传一个对象。

### 5.2 回调 DTO

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/dto/ArgoTaskCallbackDTO.java`

```java
package com.seres.sacp.dms.module.argo.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.Data;

import java.time.LocalDateTime;

@Data
public class ArgoTaskCallbackDTO {

    @NotBlank(message = "workflowName不能为空")
    private String workflowName;

    @NotBlank(message = "status不能为空")
    private String status;

    private String message;

    private LocalDateTime finishTime;
}
```
注释：Argo 回调只需要传 workflow 名称、状态和错误信息。`finishTime` 可以由 Argo 传，也可以后端没收到时用当前时间兜底。

### 5.3 重跑 DTO

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/dto/ArgoTaskRetryDTO.java`

```java
package com.seres.sacp.dms.module.argo.dto;

import lombok.Data;

@Data
public class ArgoTaskRetryDTO {

    private String message;
}
```
注释：当前重跑不需要额外参数，保留 `message` 是为了页面可传“人工重跑原因”，以后也方便扩展。

### 5.4 查询对象

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/query/ArgoTaskQuery.java`

```java
package com.seres.sacp.dms.module.argo.query;

import com.seres.sacp.dms.module.info.query.PageParam;
import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = true)
public class ArgoTaskQuery extends PageParam {

    private String taskType;

    private String status;

    private Long dataId;

    private String workflowName;

    private String templateName;
}
```
注释：项目已有 `PageParam`，这里复用它的 `pageNum` 和 `pageSize`，列表接口返回 `PageInfo`。

### 5.5 页面 VO

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/vo/ArgoTaskVO.java`

```java
package com.seres.sacp.dms.module.argo.vo;

import lombok.Data;

import java.time.LocalDateTime;

@Data
public class ArgoTaskVO {

    private Long id;

    private String workflowName;

    private String templateName;

    private String taskType;

    private String bizId;

    private Long dataId;

    private String dataName;

    private String parameters;

    private String generateName;

    private String status;

    private String message;

    private LocalDateTime startTime;

    private LocalDateTime finishTime;

    private Long retryFromTaskId;
}
```
注释：VO 先和实体字段保持一致，页面任务列表和详情都能直接使用。以后如果要隐藏 `parameters`，可以新增列表专用 VO。

## 6. 第四步：新增任务 Service

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/service/ArgoTaskService.java`

```java
package com.seres.sacp.dms.module.argo.service;

import com.github.pagehelper.PageInfo;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskCallbackDTO;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskSubmitDTO;
import com.seres.sacp.dms.module.argo.query.ArgoTaskQuery;
import com.seres.sacp.dms.module.argo.vo.ArgoTaskVO;

public interface ArgoTaskService {

    ArgoTaskVO submitAndCreateTask(ArgoTaskSubmitDTO dto);

    void handleCallback(ArgoTaskCallbackDTO dto);

    ArgoTaskVO retry(Long id, String message);

    PageInfo<ArgoTaskVO> page(ArgoTaskQuery query);

    ArgoTaskVO getById(Long id);
}
```
注释：所有和任务记录有关的能力集中在 `ArgoTaskService`，业务代码不要直接操作 `argo_task` 表。

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/service/impl/ArgoTaskServiceImpl.java`

```java
package com.seres.sacp.dms.module.argo.service.impl;

import cn.hutool.json.JSONUtil;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.pagehelper.PageHelper;
import com.github.pagehelper.PageInfo;
import com.seres.sacp.dms.client.ArgoWorkflowClient;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskCallbackDTO;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskSubmitDTO;
import com.seres.sacp.dms.module.argo.entity.ArgoTask;
import com.seres.sacp.dms.module.argo.enums.ArgoTaskStatusEnum;
import com.seres.sacp.dms.module.argo.mapper.ArgoTaskMapper;
import com.seres.sacp.dms.module.argo.query.ArgoTaskQuery;
import com.seres.sacp.dms.module.argo.service.ArgoTaskService;
import com.seres.sacp.dms.module.argo.vo.ArgoTaskVO;
import com.seres.sacp.framework.common.exception.ServiceException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.Assert;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class ArgoTaskServiceImpl implements ArgoTaskService {

    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    private final ArgoWorkflowClient argoWorkflowClient;
    private final ArgoTaskMapper argoTaskMapper;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public ArgoTaskVO submitAndCreateTask(ArgoTaskSubmitDTO dto) {
        String workflowName = argoWorkflowClient.submitRaw(
                dto.getTemplateName(), dto.getParameters(), dto.getGenerateName());

        ArgoTask task = new ArgoTask();
        task.setWorkflowName(workflowName);
        task.setTemplateName(dto.getTemplateName());
        task.setTaskType(dto.getTaskType());
        task.setBizId(dto.getBizId());
        task.setDataId(dto.getDataId());
        task.setDataName(dto.getDataName());
        task.setParameters(JSONUtil.toJsonStr(dto.getParameters()));
        task.setGenerateName(dto.getGenerateName());
        task.setStatus(dto.getRetryFromTaskId() == null
                ? ArgoTaskStatusEnum.SUBMITTED.getCode()
                : ArgoTaskStatusEnum.RETRY_SUBMITTED.getCode());
        task.setStartTime(LocalDateTime.now());
        task.setRetryFromTaskId(dto.getRetryFromTaskId());
        argoTaskMapper.insert(task);

        log.info("Argo 任务记录创建成功, taskId={}, workflowName={}, taskType={}",
                task.getId(), task.getWorkflowName(), task.getTaskType());
        return toVO(task);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void handleCallback(ArgoTaskCallbackDTO dto) {
        ArgoTask task = argoTaskMapper.selectOne(Wrappers.<ArgoTask>lambdaQuery()
                .eq(ArgoTask::getWorkflowName, dto.getWorkflowName())
                .eq(ArgoTask::getDeleted, false)
                .last("LIMIT 1"));
        if (task == null) {
            log.warn("收到未知 Argo Workflow 回调, workflowName={}, status={}",
                    dto.getWorkflowName(), dto.getStatus());
            return;
        }
        if (ArgoTaskStatusEnum.isFinished(task.getStatus())) {
            log.info("Argo 任务已是终态，忽略重复回调, taskId={}, workflowName={}, currentStatus={}",
                    task.getId(), task.getWorkflowName(), task.getStatus());
            return;
        }

        String normalizedStatus = normalizeStatus(dto.getStatus());
        task.setStatus(normalizedStatus);
        task.setMessage(dto.getMessage());
        task.setFinishTime(dto.getFinishTime() == null ? LocalDateTime.now() : dto.getFinishTime());
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

        log.info("Argo 任务状态更新成功, taskId={}, workflowName={}, status={}",
                task.getId(), task.getWorkflowName(), normalizedStatus);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public ArgoTaskVO retry(Long id, String message) {
        ArgoTask oldTask = argoTaskMapper.selectById(id);
        Assert.isTrue(oldTask != null, "任务不存在");
        Assert.isTrue(ArgoTaskStatusEnum.FAILED.getCode().equals(oldTask.getStatus())
                || ArgoTaskStatusEnum.ERROR.getCode().equals(oldTask.getStatus()), "只有失败或异常任务允许重跑");

        Map<String, String> parameters;
        try {
            parameters = OBJECT_MAPPER.readValue(oldTask.getParameters(), new TypeReference<Map<String, String>>() {});
        } catch (Exception e) {
            throw new ServiceException(200, "任务参数解析失败，无法重跑: " + e.getMessage());
        }

        ArgoTaskSubmitDTO retryDto = new ArgoTaskSubmitDTO();
        retryDto.setTemplateName(oldTask.getTemplateName());
        retryDto.setParameters(parameters);
        retryDto.setGenerateName(oldTask.getGenerateName());
        retryDto.setTaskType(oldTask.getTaskType());
        retryDto.setBizId(oldTask.getBizId());
        retryDto.setDataId(oldTask.getDataId());
        retryDto.setDataName(oldTask.getDataName());
        retryDto.setRetryFromTaskId(oldTask.getId());

        ArgoTaskVO newTask = submitAndCreateTask(retryDto);
        log.info("Argo 任务重跑成功, oldTaskId={}, newTaskId={}, message={}",
                oldTask.getId(), newTask.getId(), message);
        return newTask;
    }
    
    /**
     * 分页查询 Argo 任务列表。
     *
     * @param query 查询条件和分页参数
     * @return 保留完整分页元数据的 Argo 任务分页结果
     */
    @Override
    public PageInfo<ArgoTaskVO> page(ArgoTaskQuery query) {
        // PageHelper 只会拦截 startPage 后紧接着执行的第一条 MyBatis 查询。
        PageHelper.startPage(query.getPageNum(), query.getPageSize());

        // 该 List 在运行时实际是 Page<ArgoTask>，内部保存了 total、pageNum、pages 等信息。
        List<ArgoTask> entityList = argoTaskMapper.selectPageList(query);

        // 必须先构造 PageInfo，保留 Page 中的分页元数据。
        PageInfo<ArgoTask> entityPageInfo = new PageInfo<>(entityList);

        // convert 只转换列表元素，同时会复制全部分页元数据。
        return entityPageInfo.convert(this::toVO);
    }

    @Override
    public ArgoTaskVO getById(Long id) {
        ArgoTask task = argoTaskMapper.selectById(id);
        Assert.isTrue(task != null, "任务不存在");
        return toVO(task);
    }

    private String normalizeStatus(String argoStatus) {
        if ("Succeeded".equalsIgnoreCase(argoStatus) || "SUCCEEDED".equalsIgnoreCase(argoStatus)) {
            return ArgoTaskStatusEnum.SUCCEEDED.getCode();
        }
        if ("Failed".equalsIgnoreCase(argoStatus) || "FAILED".equalsIgnoreCase(argoStatus)
                || "Error".equalsIgnoreCase(argoStatus) || "ERROR".equalsIgnoreCase(argoStatus)) {
            return ArgoTaskStatusEnum.FAILED.getCode();
        }
        return ArgoTaskStatusEnum.ERROR.getCode();
    }

    private ArgoTaskVO toVO(ArgoTask task) {
        ArgoTaskVO vo = new ArgoTaskVO();
        vo.setId(task.getId());
        vo.setWorkflowName(task.getWorkflowName());
        vo.setTemplateName(task.getTemplateName());
        vo.setTaskType(task.getTaskType());
        vo.setBizId(task.getBizId());
        vo.setDataId(task.getDataId());
        vo.setDataName(task.getDataName());
        vo.setParameters(task.getParameters());
        vo.setGenerateName(task.getGenerateName());
        vo.setStatus(task.getStatus());
        vo.setMessage(task.getMessage());
        vo.setStartTime(task.getStartTime());
        vo.setFinishTime(task.getFinishTime());
        vo.setRetryFromTaskId(task.getRetryFromTaskId());
        return vo;
    }
}
```
注释：这里故意让 `ArgoTaskService` 调 `ArgoWorkflowClient.submitRaw`，而不是让 Client 再回调 Service，避免循环依赖。`submitAndCreateTask` 先提交 Argo，成功拿到 workflowName 后再入库；如果提交失败，不会产生 `SUBMITTED` 任务。若你希望提交失败也落库，需要在业务侧捕获异常后额外写 `ERROR` 记录。

## 7. 第五步：改造 ArgoWorkflowClient

修改文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/client/ArgoWorkflowClient.java`

核心改法：把原来的 `submit` 方法保留给测试接口使用，但新增 `submitRaw`，任务服务调用 `submitRaw`。然后把三个业务便捷方法改成返回 `ArgoTaskSubmitDTO` 或删除便捷方法，推荐由业务方直接创建 DTO 调 `ArgoTaskService`。

```java
public String submitRaw(String template, Map<String, String> parameters, String generateName) {
    String url = StrUtil.removeSuffix(argoProperties.getEndpoint(), "/")
            + "/api/v1/workflows/" + argoProperties.getNamespace() + "/submit";

    JSONArray parameterArray = new JSONArray();
    if (parameters != null) {
        parameters.forEach((name, value) -> parameterArray.put(name + "=" + value));
    }

    JSONObject submitOptions = new JSONObject().set("parameters", parameterArray);
    if (StrUtil.isNotBlank(generateName)) {
        submitOptions.set("generateName", generateName);
    }

    JSONObject body = new JSONObject()
            .set("namespace", argoProperties.getNamespace())
            .set("resourceKind", "WorkflowTemplate")
            .set("resourceName", template)
            .set("submitOptions", submitOptions);

    String bodyStr = body.toString();
    try (HttpResponse response = HttpRequest.post(url)
            .header("Authorization", "Bearer " + argoProperties.getToken())
            .header("Content-Type", "application/json")
            .body(bodyStr)
            .timeout(TIMEOUT_MS)
            .execute()) {
        if (response.isOk()) {
            String workflowName = new JSONObject(response.body())
                    .getByPath("metadata.name", String.class);
            log.info("Argo 工作流提交成功, template={}, parameters={}, workflow={}",
                    template, parameters, workflowName);
            return workflowName;
        }
        log.error("Argo 工作流提交失败, template={}, status={}, body={}",
                template, response.getStatus(), response.body());
        throw new RuntimeException("Argo 工作流提交失败, status=" + response.getStatus()
                + ", body=" + response.body());
    }
}

public String submit(String template, Map<String, String> parameters) {
    return submitRaw(template, parameters, null);
}

public String submit(String template, Map<String, String> parameters, String generateName) {
    return submitRaw(template, parameters, generateName);
}
```
注释：这段代码基本复用原 `submit` 逻辑，只是把真正 HTTP 提交方法命名为 `submitRaw`。这样 `ArgoTaskServiceImpl` 可以提交 Argo，但 `IndexController` 的旧测试接口也不会立刻编译失败。

然后建议删除或停止使用这三个旧方法：

```java
// submitEventProcess(...)
// submitQc(...)
// submitDataProcess(...)
```
注释：旧方法只有参数，没有 `taskType`、`bizId`、`dataName` 等任务上下文。保留它们也可以，但业务方不能继续调用它们，否则不会记录任务。

## 8. 第六步：新增 Controller

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/controller/ArgoTaskController.java`

```java
package com.seres.sacp.dms.module.argo.controller;

import com.github.pagehelper.PageInfo;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskCallbackDTO;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskRetryDTO;
import com.seres.sacp.dms.module.argo.query.ArgoTaskQuery;
import com.seres.sacp.dms.module.argo.service.ArgoTaskService;
import com.seres.sacp.dms.module.argo.vo.ArgoTaskVO;
import com.seres.sacp.framework.common.pojo.CommonResult;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Tag(name = "Argo任务管理")
@RestController
@RequestMapping("/argo/tasks")
@RequiredArgsConstructor
public class ArgoTaskController {

    private final ArgoTaskService argoTaskService;

    @PostMapping("/page")
    @Operation(summary = "分页查询 Argo 任务")
    public CommonResult<PageInfo<ArgoTaskVO>> page(@Valid @RequestBody ArgoTaskQuery query) {
        return CommonResult.success(argoTaskService.page(query));
    }

    @GetMapping("/{id}")
    @Operation(summary = "查询 Argo 任务详情")
    public CommonResult<ArgoTaskVO> getById(@PathVariable Long id) {
        return CommonResult.success(argoTaskService.getById(id));
    }

    @PostMapping("/{id}/retry")
    @Operation(summary = "重跑失败的 Argo 任务")
    public CommonResult<ArgoTaskVO> retry(@PathVariable Long id,
                                          @RequestBody(required = false) ArgoTaskRetryDTO dto) {
        String message = dto == null ? null : dto.getMessage();
        return CommonResult.success(argoTaskService.retry(id, message));
    }

    @PostMapping("/callback")
    @Operation(summary = "Argo Workflow 结束回调")
    public CommonResult<Boolean> callback(@Valid @RequestBody ArgoTaskCallbackDTO dto) {
        log.info("开始处理 Argo Workflow 结束回调, workflowName={}", dto.getWorkflowName());
        argoTaskService.handleCallback(dto);
        return CommonResult.success(true);
    }
}
```
注释：回调接口也放在这个 Controller 内，路径是 `/argo/tasks/callback`。Argo 重复调用时也返回成功，避免 Argo 端一直认为回调失败。

## 9. 第七步：放开回调接口鉴权

修改文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/config/WebMvcConfig.java`

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(jwtAuthInterceptor)
            .addPathPatterns("/**")
            .excludePathPatterns("/actuator/**", "/error", "/argo/tasks/callback");
    registry.addInterceptor(traceIdInterceptor)
            .addPathPatterns("/**")
            .excludePathPatterns("/actuator/**", "/error");
}
```
注释：需求要求回调接口不鉴权。这里只有 JWT 拦截器排除 `/argo/tasks/callback`，TraceId 拦截器仍然保留，方便查日志。

## 10. 第八步：改造业务调用点

### 10.1 数据解析任务

修改文件：`DataUploadConsumer.java`

新增依赖：

```java
private final ArgoTaskService argoTaskService;
private final ArgoProperties argoProperties;
```
注释：`ArgoTaskService` 负责提交并记录任务，`ArgoProperties` 提供模板名称。改完构造函数后，相关测试的构造参数也要同步更新。

替换 `dispatchParseStart` 中提交 Argo 的代码：

```java
Map<String, String> parameters = new LinkedHashMap<>();
parameters.put("s3_path", parseMsg.getS3Path());
parameters.put("data_id", parseMsg.getDataId());
parameters.put("vis_enable", String.valueOf(parseMsg.getVisEnable()));
parameters.put("truth_flag", String.valueOf(parseMsg.isTruthFlag()));

String generateName = parseMsg.isTruthFlag()
        ? "data-process-truth-" + parseMsg.getDataId() + "-"
        : "data-process-adc-" + parseMsg.getDataId() + "-";

ArgoTaskSubmitDTO submitDTO = new ArgoTaskSubmitDTO();
submitDTO.setTemplateName(argoProperties.getTemplate());
submitDTO.setParameters(parameters);
submitDTO.setGenerateName(generateName);
submitDTO.setTaskType(ArgoTaskTypeEnum.DATA_PROCESS.getCode());
submitDTO.setBizId(parseMsg.getDataId());
submitDTO.setDataId(dataInfoVO.getId());
submitDTO.setDataName(dataInfoVO.getDataName());

try {
    argoTaskService.submitAndCreateTask(submitDTO);
} catch (Exception e) {
    log.error("数据解析 Argo 任务提交异常, dataId={}", parseMsg.getDataId(), e);
}
```
注释：这段替代原来的 `argoWorkflowClient.submitDataProcess(...)`。原始数据解析需要保存 `dataId` 和 `dataName`，后续列表页才能按原始数据查任务。

需要补充 import：

```java
import com.seres.sacp.dms.config.ArgoProperties;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskSubmitDTO;
import com.seres.sacp.dms.module.argo.enums.ArgoTaskTypeEnum;
import com.seres.sacp.dms.module.argo.service.ArgoTaskService;
import java.util.LinkedHashMap;
import java.util.Map;
```
注释：`LinkedHashMap` 可以保持参数顺序，虽然 Argo 不要求顺序，但排查日志时更清楚。

### 10.2 事件解析任务

修改文件：`EventCompleteConsumer.java`

新增依赖：

```java
private final ArgoTaskService argoTaskService;
private final ArgoProperties argoProperties;
```
注释：事件解析任务没有现成的原始 `dataId`，所以 `bizId` 使用 `eventInfoId`，`dataId` 可以留空。

替换原来的 `argoWorkflowClient.submitEventProcess(...)`：

```java
Map<String, String> parameters = new LinkedHashMap<>();
parameters.put("event_info_id", String.valueOf(event.getEventInfoId()));
parameters.put("files", filesJson);

ArgoTaskSubmitDTO submitDTO = new ArgoTaskSubmitDTO();
submitDTO.setTemplateName(argoProperties.getEventTemplate());
submitDTO.setParameters(parameters);
submitDTO.setGenerateName("event-process-" + event.getEventInfoId() + "-");
submitDTO.setTaskType(ArgoTaskTypeEnum.EVENT_PROCESS.getCode());
submitDTO.setBizId(String.valueOf(event.getEventInfoId()));

try {
    argoTaskService.submitAndCreateTask(submitDTO);
} catch (Exception e) {
    log.error("事件解析 Argo 任务提交异常, eventInfoId={}", event.getEventInfoId(), e);
}
```
注释：事件任务的参数必须完整保存，尤其是 `files` JSON；失败重跑时会直接复用这份 JSON。

需要补充 import：

```java
import com.seres.sacp.dms.config.ArgoProperties;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskSubmitDTO;
import com.seres.sacp.dms.module.argo.enums.ArgoTaskTypeEnum;
import com.seres.sacp.dms.module.argo.service.ArgoTaskService;
import java.util.LinkedHashMap;
import java.util.Map;
```
注释：改完后可以删除 `ArgoWorkflowClient` 字段和 import。

### 10.3 手动质检任务

修改文件：`DataLogicSliceServiceImpl.java`

新增依赖：

```java
private final ArgoTaskService argoTaskService;
private final ArgoProperties argoProperties;
```
注释：这里原来注入了 `ArgoWorkflowClient`，改造后建议删除它，统一由 `ArgoTaskService` 提交并记录任务。

替换原来的 `argoWorkflowClient.submitQc(...)`：

```java
Map<String, String> parameters = new LinkedHashMap<>();
parameters.put("slice_id", String.valueOf(entity.getId()));
parameters.put("start_time_ns", String.valueOf(absStartNs));
parameters.put("end_time_ns", String.valueOf(absEndNs));
parameters.put("file_list", fileListJson);

ArgoTaskSubmitDTO submitDTO = new ArgoTaskSubmitDTO();
submitDTO.setTemplateName(argoProperties.getManualQcTemplate());
submitDTO.setParameters(parameters);
submitDTO.setGenerateName("sacp-dms-manual-qc-" + entity.getId() + "-");
submitDTO.setTaskType(ArgoTaskTypeEnum.MANUAL_QC.getCode());
submitDTO.setBizId(String.valueOf(entity.getId()));
submitDTO.setDataId(entity.getDataId());
submitDTO.setDataName(entity.getDataName());

try {
    argoTaskService.submitAndCreateTask(submitDTO);
    qc.setParseStatus(1);
    dataLogicSliceQcMapper.updateById(qc);
} catch (Exception e) {
    qc.setParseStatus(3);
    dataLogicSliceQcMapper.updateById(qc);
    log.error("手动质检 Argo 任务提交异常, sliceId={}", entity.getId(), e);
}
```
注释：原代码不管提交是否成功都会把 `parseStatus` 改成解析中。这里改成提交成功才置为 `1`，提交失败置为 `3`，页面状态更准确。

需要补充 import：

```java
import com.seres.sacp.dms.config.ArgoProperties;
import com.seres.sacp.dms.module.argo.dto.ArgoTaskSubmitDTO;
import com.seres.sacp.dms.module.argo.enums.ArgoTaskTypeEnum;
import com.seres.sacp.dms.module.argo.service.ArgoTaskService;
```
注释：`DataLogicSliceServiceImpl` 已经有 `LinkedHashMap`、`Map` import，不需要重复加。

### 10.4 手动测试提交接口

修改文件：`IndexController.java`

新增依赖：

```java
private final ArgoTaskService argoTaskService;
```
注释：`IndexController.submitArgo` 是测试入口，也要记录任务，否则“每次本系统调起 Argo 都记录任务”不完整。

替换 `/index/argo/submit` 方法内部：

```java
ArgoTaskSubmitDTO submitDTO = new ArgoTaskSubmitDTO();
submitDTO.setTemplateName(template);
submitDTO.setParameters(parameters);
submitDTO.setGenerateName("sxw-test-task-status-");
submitDTO.setTaskType(ArgoTaskTypeEnum.CUSTOM.getCode());
ArgoTaskVO task = argoTaskService.submitAndCreateTask(submitDTO);
return CommonResult.success(task.getWorkflowName());
```
注释：测试接口不知道具体业务 ID，所以任务类型用 `CUSTOM`，`bizId` 和 `dataId` 可以为空。

需要补充 import：

```java
import com.seres.sacp.dms.module.argo.dto.ArgoTaskSubmitDTO;
import com.seres.sacp.dms.module.argo.enums.ArgoTaskTypeEnum;
import com.seres.sacp.dms.module.argo.service.ArgoTaskService;
import com.seres.sacp.dms.module.argo.vo.ArgoTaskVO;
```
注释：改完后可以删除 `ArgoWorkflowClient` 字段和 import，或者保留给其他测试方法用。

## 11. 第九步：改造 WorkflowTemplate 增加回调

当前三个模板都没有 `onExit`。建议每个 WorkflowTemplate 增加一个统一的 `exit-handler`，Workflow 结束时由 HTTP Template 调本系统。

### 11.1 数据解析模板

参考文件：`workflow-template/sacp-dms-data-process-with-qc.yaml`

在 `spec` 下增加：

```yaml
spec:
  entrypoint: data-process
  onExit: exit-handler
  templates:
    - name: exit-handler
      container:
        image: hb.seres.cn/sacp-cloud-sdi/net-tool:v0.1
        command: [sh, -c]
        args:
          - |
            curl -sS -X POST "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback" \
              -H "Content-Type: application/json" \
              -d "{\"workflowName\":\"{{workflow.name}}\",\"status\":\"{{workflow.status}}\",\"message\":\"workflow finished with status {{workflow.status}}\"}"
```
注释：`onExit` 表示无论主流程成功还是失败，最后都会执行 `exit-handler`。`{{workflow.name}}` 必须传回后端，因为后端靠它查 `argo_task.workflow_name`。

注意：实际文件里已经有 `entrypoint: data-process`，不要重复写两个 `entrypoint`。正确做法是在现有 `spec` 下加一行 `onExit: exit-handler`，再把 `exit-handler` 作为一个新模板放进 `templates` 列表。

### 11.2 事件解析模板

参考文件：`workflow-template/sacp-dms-event-parse.yaml`

```yaml
spec:
  entrypoint: event-parse
  onExit: exit-handler
  templates:
    - name: exit-handler
      container:
        image: hb.seres.cn/sacp-cloud-sdi/net-tool:v0.1
        command: [sh, -c]
        args:
          - |
            curl -sS -X POST "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback" \
              -H "Content-Type: application/json" \
              -d "{\"workflowName\":\"{{workflow.name}}\",\"status\":\"{{workflow.status}}\",\"message\":\"workflow finished with status {{workflow.status}}\"}"
```
注释：事件解析模板只有一个主模板 `event-parse`，回调模板和数据解析一致即可。

### 11.3 手动质检模板

参考文件：`workflow-template/sacp-dms-manual-qc.yaml`

```yaml
spec:
  entrypoint: manual-qc
  onExit: exit-handler
  templates:
    - name: exit-handler
      container:
        image: hb.seres.cn/sacp-cloud-sdi/net-tool:v0.1
        command: [sh, -c]
        args:
          - |
            curl -sS -X POST "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback" \
              -H "Content-Type: application/json" \
              -d "{\"workflowName\":\"{{workflow.name}}\",\"status\":\"{{workflow.status}}\",\"message\":\"workflow finished with status {{workflow.status}}\"}"
```
注释：`url` 使用集群内 Service 地址，和模板里已有的 `DMS_DATA_MANAGER_URL=http://sacp-dms-data-server.sacp-dms:80/` 保持一致。

### 11.4 Argo 版本注意事项

如果当前 Argo 版本的 HTTP Template 不支持 `http:`，可以退化成容器回调：

```yaml
    - name: exit-handler
      container:
        image: hb.seres.cn/sacp-cloud-sdi/net-tool:v0.1
        command: [sh, -c]
        args:
          - |
            curl -sS -X POST "http://sacp-dms-data-server.sacp-dms:80/argo/tasks/callback" \
              -H "Content-Type: application/json" \
              -d "{\"workflowName\":\"{{workflow.name}}\",\"status\":\"{{workflow.status}}\",\"message\":\"workflow finished with status {{workflow.status}}\"}"
```
注释：需求文档要求参考 HTTP Template，首选上面的 `http:` 写法。只有集群 Argo 版本不支持 HTTP Template 时，才用 curl 容器兜底。

## 12. 第十步：接口使用示例

### 12.1 分页查询任务

```bash
curl -X POST 'http://localhost:8080/argo/tasks/page' \
  -H 'Content-Type: application/json' \
  -d '{
    "pageNum": 1,
    "pageSize": 10,
    "taskType": "DATA_PROCESS",
    "status": "FAILED",
    "dataId": 2067209677984182273
  }'
```
注释：页面筛选条件可以只传其中几个，不传表示不过滤。后端返回 `PageInfo<ArgoTaskVO>`。

### 12.2 查看详情

```bash
curl -X GET 'http://localhost:8080/argo/tasks/1'
```
注释：详情会返回 `parameters`，页面可以格式化 JSON 展示，方便排查当时提交给 Argo 的参数。

### 12.3 手动模拟回调

```bash
curl -X POST 'http://localhost:8080/argo/tasks/callback' \
  -H 'Content-Type: application/json' \
  -d '{
    "workflowName": "data-process-adc-2067209677984182273-abcde",
    "status": "Succeeded",
    "message": ""
  }'
```
注释：本地调试时可以先用 curl 模拟 Argo 回调，确认任务状态能从 `SUBMITTED` 变成 `SUCCEEDED`。

### 12.4 重跑失败任务

```bash
curl -X POST 'http://localhost:8080/argo/tasks/1/retry' \
  -H 'Content-Type: application/json' \
  -d '{
    "message": "人工确认后重新提交"
  }'
```
注释：重跑成功后返回新任务。旧任务状态不变，新任务的 `retryFromTaskId` 等于旧任务 ID。

## 13. 第十一步：测试和验证

### 13.1 编译测试

```bash
mvn test -pl sacp-dms-data-service
```
注释：这是项目要求的服务模块测试命令。当前 `DataUploadConsumerFullFlowTest` 里手动 new `DataUploadConsumer` 的构造参数已经和现有代码不一致；本次改造新增构造参数后，也要同步修正这个测试，否则编译阶段会失败。

### 13.2 推荐新增单元测试

新增测试：`ArgoTaskServiceImplTest`

重点覆盖：

1. `submitAndCreateTask`：mock `ArgoWorkflowClient.submitRaw` 返回 workflowName，断言数据库插入 `SUBMITTED`。
2. `handleCallback`：传 `Succeeded`，断言状态更新为 `SUCCEEDED`，`finishTime` 不为空。
3. `handleCallback` 重复调用：任务已是 `SUCCEEDED` 时再次回调，不报错、不改变状态。
4. `retry`：旧任务为 `FAILED` 时，重跑生成新任务，并且 `retryFromTaskId` 指向旧任务。

```java
when(argoWorkflowClient.submitRaw(anyString(), anyMap(), any())).thenReturn("workflow-001");
```
注释：测试里只 mock `submitRaw`，不要真的请求 Argo。这样测试稳定，不依赖外部网络和 Argo 集群。

### 13.3 联调检查清单

1. 提交数据解析任务后，`argo_task` 出现一条 `DATA_PROCESS`、`SUBMITTED` 记录。
2. Argo 页面能看到 workflow 名称，且和 `argo_task.workflow_name` 一致。
3. Workflow 成功结束后，`argo_task.status` 变成 `SUCCEEDED`。
4. Workflow 失败后，`argo_task.status` 变成 `FAILED`，`message` 有失败信息。
5. 对失败任务点重跑后，新增一条任务，旧任务不被覆盖。
6. `/argo/tasks/callback` 不带登录态也能访问，其它业务接口仍需要鉴权。

## 14. 常见坑

1. 不要只保存 `data_id`，一定要保存完整 `parameters` JSON。重跑靠它复原原始 Argo 请求。
2. 不要用旧任务记录改状态表示重跑。重跑必须新建任务。
3. 不要忘记放开 `/argo/tasks/callback` 鉴权，否则 Argo 回调会被 JWT 拦截。
4. 不要在 WorkflowTemplate 里写两个 `entrypoint`。只保留原来的 entrypoint，再增加 `onExit`。
5. 不要让业务方继续调用 `submitDataProcess`、`submitEventProcess`、`submitQc` 这类旧便捷方法，否则任务表不会有记录。
6. 如果 Argo 回调成功但状态没更新，先检查回调传的 `workflowName` 是否和 `argo_task.workflow_name` 完全一致。

## 15. 回调失败兜底：定时轮询 Argo 状态

### 15.1 为什么需要主动轮询

Argo 的 `onExit` 回调不是 100% 可靠的。比如本系统重启、网络抖动、Service DNS 异常、接口临时 500，都会导致 Argo 没能成功调用 `/argo/tasks/callback`。需求里明确说“回调失败处理：不重试，本项目主动轮询 Argo 状态”，所以后端必须增加一个补偿定时任务。

补偿逻辑如下：

1. 定时扫描本地 `argo_task` 表。
2. 只查长时间处于 `SUBMITTED`、`RETRY_SUBMITTED` 的任务。
3. 调用 Argo API 查询真实 Workflow 状态。
4. 如果 Argo 已经结束，把本地任务补偿更新为 `SUCCEEDED`、`FAILED` 或 `ERROR`。
5. 如果 Argo 仍在运行，不更新本地状态，等待下一轮扫描。

### 15.2 增加配置项

修改文件：`sacp-dms-data-service/src/main/resources/application*.yml`

```yaml
argo:
  status-sync:
    enabled: true
    fixed-delay-ms: 60000
    submitted-timeout-minutes: 5
    batch-size: 50
```
注释：`fixed-delay-ms` 表示每 60 秒扫描一次。`submitted-timeout-minutes` 表示任务提交超过 5 分钟还没收到回调，才会主动查 Argo，避免刚提交就频繁查询。

修改文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/config/ArgoProperties.java`

```java
private StatusSync statusSync = new StatusSync();

@Data
public static class StatusSync {

    private Boolean enabled = true;

    private Long fixedDelayMs = 60000L;

    private Integer submittedTimeoutMinutes = 5;

    private Integer batchSize = 50;
}
```
注释：这个内部类会绑定 `argo.status-sync` 下的配置。默认值保证即使配置中心暂时没配，也能按 60 秒、5 分钟、50 条的规则运行。

### 15.3 ArgoWorkflowClient 增加查询 Workflow 状态方法

修改文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/client/ArgoWorkflowClient.java`

```java
public JSONObject getWorkflow(String workflowName) {
    String url = StrUtil.removeSuffix(argoProperties.getEndpoint(), "/")
            + "/api/v1/workflows/" + argoProperties.getNamespace() + "/" + workflowName;

    try (HttpResponse response = HttpRequest.get(url)
            .header("Authorization", "Bearer " + argoProperties.getToken())
            .timeout(TIMEOUT_MS)
            .execute()) {
        if (response.isOk()) {
            return new JSONObject(response.body());
        }
        log.error("查询 Argo Workflow 失败, workflowName={}, status={}, body={}",
                workflowName, response.getStatus(), response.body());
        throw new RuntimeException("查询 Argo Workflow 失败, status=" + response.getStatus()
                + ", body=" + response.body());
    }
}
```
注释：Argo 查询接口路径是 `/api/v1/workflows/{namespace}/{workflowName}`。返回 JSON 里重点读取 `status.phase`、`status.message`、`status.finishedAt`。

再增加一个轻量 VO，避免 Service 直接解析复杂 JSON。

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/vo/ArgoWorkflowStatusVO.java`

```java
package com.seres.sacp.dms.module.argo.vo;

import lombok.Data;

import java.time.LocalDateTime;

@Data
public class ArgoWorkflowStatusVO {

    private String workflowName;

    private String phase;

    private String message;

    private LocalDateTime finishedAt;
}
```
注释：`phase` 是 Argo 原始状态，常见值是 `Pending`、`Running`、`Succeeded`、`Failed`、`Error`。

也可以把 Client 方法直接封装成返回 VO：

```java
public ArgoWorkflowStatusVO getWorkflowStatus(String workflowName) {
    JSONObject workflow = getWorkflow(workflowName);
    ArgoWorkflowStatusVO vo = new ArgoWorkflowStatusVO();
    vo.setWorkflowName(workflowName);
    vo.setPhase(workflow.getByPath("status.phase", String.class));
    vo.setMessage(workflow.getByPath("status.message", String.class));

    String finishedAt = workflow.getByPath("status.finishedAt", String.class);
    if (StrUtil.isNotBlank(finishedAt)) {
        vo.setFinishedAt(java.time.OffsetDateTime.parse(finishedAt).toLocalDateTime());
    }
    return vo;
}
```
注释：Argo 的 `finishedAt` 通常是 ISO-8601 字符串，例如 `2026-07-07T08:36:13Z`。这里用 `OffsetDateTime.parse` 转成 `LocalDateTime`。

### 15.4 Mapper 增加扫描待补偿任务

修改文件：`ArgoTaskMapper.java`

```java
List<ArgoTask> selectPendingTimeoutTasks(@Param("deadline") LocalDateTime deadline,
                                         @Param("limit") Integer limit);
```
注释：`deadline` 是“当前时间减去超时分钟数”。只扫描提交时间早于 deadline 的任务，避免刚提交的任务被立即轮询。

需要补充 import：

```java
import java.time.LocalDateTime;
```
注释：Mapper 方法参数里使用了 `LocalDateTime`。

修改文件：`ArgoTaskMapper.xml`

```xml
<select id="selectPendingTimeoutTasks" resultType="com.seres.sacp.dms.module.argo.entity.ArgoTask">
    SELECT *
    FROM argo_task
    WHERE deleted = 0
      AND status IN ('SUBMITTED', 'RETRY_SUBMITTED')
      AND start_time &lt;= #{deadline}
    ORDER BY start_time ASC
    LIMIT #{limit}
</select>
```
注释：按 `start_time ASC` 扫描，优先补偿最早提交的任务。`LIMIT` 控制单次扫描数量，避免一次查太多影响数据库和 Argo。

### 15.5 Service 增加补偿更新方法

修改文件：`ArgoTaskService.java`

```java
int syncTimeoutTasksFromArgo();
```
注释：返回本轮实际补偿更新的任务数量，定时任务日志可以直接打印。

修改文件：`ArgoTaskServiceImpl.java`

```java
@Override
@Transactional(rollbackFor = Exception.class)
public int syncTimeoutTasksFromArgo() {
    ArgoProperties.StatusSync config = argoProperties.getStatusSync();
    LocalDateTime deadline = LocalDateTime.now().minusMinutes(config.getSubmittedTimeoutMinutes());
    List<ArgoTask> tasks = argoTaskMapper.selectPendingTimeoutTasks(deadline, config.getBatchSize());

    int updatedCount = 0;
    for (ArgoTask task : tasks) {
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
        }
    }
    return updatedCount;
}

private boolean isArgoFinished(String phase) {
    return "Succeeded".equalsIgnoreCase(phase)
            || "Failed".equalsIgnoreCase(phase)
            || "Error".equalsIgnoreCase(phase);
}
```
注释：这里复用 `handleCallback` 更新本地状态，这样回调更新和轮询补偿的状态转换规则完全一致。单个任务查询失败只打日志，不影响下一条任务。

`ArgoTaskServiceImpl` 需要新增依赖：

```java
private final ArgoProperties argoProperties;
```
注释：Service 需要读取 `statusSync` 配置，决定扫描超时时间和批量大小。

需要补充 import：

```java
import com.seres.sacp.dms.config.ArgoProperties;
import com.seres.sacp.dms.module.argo.vo.ArgoWorkflowStatusVO;
```
注释：`ArgoProperties` 提供轮询配置，`ArgoWorkflowStatusVO` 承载 Argo 查询结果。

### 15.6 新增定时任务

新增文件：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/argo/scheduler/ArgoTaskStatusSyncScheduler.java`

```java
package com.seres.sacp.dms.module.argo.scheduler;

import com.seres.sacp.dms.config.ArgoProperties;
import com.seres.sacp.dms.module.argo.service.ArgoTaskService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class ArgoTaskStatusSyncScheduler {

    private final ArgoTaskService argoTaskService;
    private final ArgoProperties argoProperties;

    @Scheduled(fixedDelayString = "${argo.status-sync.fixed-delay-ms:60000}")
    public void syncTimeoutTasks() {
        if (!Boolean.TRUE.equals(argoProperties.getStatusSync().getEnabled())) {
            return;
        }
        int updatedCount = argoTaskService.syncTimeoutTasksFromArgo();
        if (updatedCount > 0) {
            log.info("Argo 超时任务状态补偿完成, updatedCount={}", updatedCount);
        }
    }
}
```
注释：`fixedDelay` 表示上一轮执行结束后再等待 60 秒。这样即使某轮查询 Argo 较慢，也不会并发堆积多轮扫描。

### 15.7 启用 Spring 定时任务

检查启动类：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/DmsDataServerApplication.java`

如果启动类还没有 `@EnableScheduling`，需要加上：

```java
import org.springframework.scheduling.annotation.EnableScheduling;

@EnableScheduling
@SpringBootApplication
public class DmsDataServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(DmsDataServerApplication.class, args);
    }
}
```
注释：没有 `@EnableScheduling` 时，`@Scheduled` 方法不会执行。项目里已有其他 scheduler 类，但仍要确认启动类或配置类是否已经开启调度。

### 15.8 补偿逻辑验证

```sql
UPDATE argo_task
SET status = 'SUBMITTED',
    start_time = DATE_SUB(NOW(), INTERVAL 10 MINUTE),
    workflow_name = '替换成真实存在的 workflow 名称'
WHERE id = 1;
```
注释：本地联调时可以把一条任务改成“10 分钟前提交且仍是 SUBMITTED”，等待定时任务扫描，看它是否会根据 Argo 真实状态补偿更新。

```bash
curl -X GET 'http://你的-argo-server/api/v1/workflows/sacp-dms-argo/替换成真实workflow名称' \
  -H 'Authorization: Bearer 替换成token'
```
注释：如果补偿任务一直失败，先用 curl 确认 Argo 查询接口、namespace、token、workflowName 是否正确。

### 15.9 补偿逻辑注意事项

1. 只补偿 `SUBMITTED` 和 `RETRY_SUBMITTED`，不要扫描 `SUCCEEDED`、`FAILED`、`ERROR`。
2. 查询 Argo 失败时不要把本地任务立即改成 `ERROR`，因为可能只是网络临时问题。
3. 如果同一个 Workflow 已被 Argo 删除，连续多次查询 404 后可以再考虑把本地任务标记为 `ERROR`，第一版先只记录日志。
4. 扫描批量不要太大，建议从 50 开始，避免 Argo API 压力过高。
5. 补偿更新必须复用 `handleCallback`，避免回调链路和轮询链路状态规则不一致。
