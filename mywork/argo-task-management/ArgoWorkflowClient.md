# ArgoWorkflowClient 注释版文档

> 源码位置：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/client/ArgoWorkflowClient.java`

## 1. 这个类解决什么问题

`ArgoWorkflowClient` 是项目里统一提交 Argo Workflow 的 HTTP 客户端。业务代码不直接拼 Argo REST API，而是调用这个类的几个方法：

- `submit(...)`：通用提交入口，按指定 `WorkflowTemplate` 提交工作流。
- `submitEventProcess(...)`：事件完成后，提交事件解析工作流。
- `submitQc(...)`：人工创建逻辑切片并要求质检时，提交手动质检工作流。
- `submitDataProcess(...)`：数据上传入库后，提交数据解析工作流。

它被 Spring 管理为 Bean，其他业务类通过构造器注入后调用。

核心流程是：

1. 从配置 `ArgoProperties` 读取 Argo 地址、Token、命名空间和模板名。
2. 把业务参数转换成 Argo submit API 需要的 JSON。
3. 使用 Hutool `HttpRequest.post(...)` 向 Argo Server 发起请求。
4. 成功时从响应 JSON 的 `metadata.name` 取出工作流名称。
5. 失败时记录日志；通用 `submit(...)` 会抛异常，业务封装方法会捕获异常并只记日志。

## 2. 完整注释版源码

```java
// 当前类所在包：client 包通常放外部系统客户端，例如 Argo、SOM、ASR 等。
package com.seres.sacp.dms.client;

// Hutool 字符串工具：用于删除 URL 末尾斜杠、判断字符串是否非空。
import cn.hutool.core.util.StrUtil;
// Hutool HTTP 请求工具：用于构造 POST 请求。
import cn.hutool.http.HttpRequest;
// Hutool HTTP 响应对象：用于读取状态码、响应体，并自动关闭连接资源。
import cn.hutool.http.HttpResponse;
// Hutool JSON 数组：用于构造 submitOptions.parameters。
import cn.hutool.json.JSONArray;
// Hutool JSON 对象：用于构造请求体和解析响应体。
import cn.hutool.json.JSONObject;
// Argo 配置类：绑定 application*.yml 中 argo.* 配置。
import com.seres.sacp.dms.config.ArgoProperties;
// Lombok：为 final 字段生成构造函数，Spring 通过构造函数注入 ArgoProperties。
import lombok.RequiredArgsConstructor;
// Lombok：生成 log 日志对象。
import lombok.extern.slf4j.Slf4j;
// Spring：把该类注册为 Bean，供 Controller、Service、Consumer 注入。
import org.springframework.stereotype.Component;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * 通过 Argo REST API 基于 WorkflowTemplate 提交工作流。
 *
 * Argo 中 WorkflowTemplate 可以理解为“任务模板”：
 * 项目只需要提交模板名和参数，Argo 根据模板创建一次具体的 Workflow 运行实例。
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class ArgoWorkflowClient {
    private static final int TIMEOUT_MS = 10_000; // HTTP 请求超时时间：10 秒。超过后 Hutool 会抛异常，避免业务线程一直卡住。
    private final ArgoProperties argoProperties; // Argo 配置，由 Spring 从 argo.* 配置项绑定而来。
    /**
     * 基于 WorkflowTemplate 提交工作流，使用模板默认 generateName。
     * 这是一个重载便捷方法：调用方不关心工作流名称前缀时，用这个方法即可。
     */
    public String submit(String template, Map<String, String> parameters) {
        // 复用三参数 submit，把 generateName 传 null，表示不覆盖模板默认命名前缀。
        return submit(template, parameters, null);
    }
    /**
     * 基于 WorkflowTemplate 提交工作流。
     * @param template WorkflowTemplate 名称，例如 sacp-dms-data-process-with-qc
     * @param parameters 业务入参，Map 中 key 是 Argo 参数名，value 是参数值
     * @param generateName 工作流名称前缀，末尾通常带 "-"，Argo 会追加随机后缀
     * @return Argo 创建出的 Workflow 名称
     */
    public String submit(String template, Map<String, String> parameters, String generateName) {
        // endpoint 可能配置为 https://xxx/argo/，removeSuffix 保证不会出现双斜杠。
        // 最终 URL 形如：
        // https://siop-dev.seres.cn/argo/api/v1/workflows/sacp-dms-argo/submit
        String url = StrUtil.removeSuffix(argoProperties.getEndpoint(), "/")
                + "/api/v1/workflows/" + argoProperties.getNamespace() + "/submit";
        // Argo submit API 要求 parameters 是 ["k1=v1", "k2=v2"] 这种字符串数组。
        JSONArray parameterArray = new JSONArray();
        if (parameters != null) {
            parameters.forEach((name, value) -> parameterArray.put(name + "=" + value));
        }
        // submitOptions 是 Argo submit API 的参数区。
        JSONObject submitOptions = new JSONObject().set("parameters", parameterArray);
        if (StrUtil.isNotBlank(generateName)) {
            // generateName 会影响最终 Workflow 名称，例如 data-process-adc-123-xxxxx。
            submitOptions.set("generateName", generateName);
        }
        // 构造完整请求体。
        // resourceKind 固定为 WorkflowTemplate，表示本次提交来自一个模板。
        JSONObject body = new JSONObject()
                .set("namespace", argoProperties.getNamespace())
                .set("resourceKind", "WorkflowTemplate")
                .set("resourceName", template)
                .set("submitOptions", submitOptions);

        String bodyStr = body.toString();
        try (HttpResponse response = HttpRequest.post(url)
                // Argo API 使用 Bearer Token 鉴权。
                .header("Authorization", "Bearer " + argoProperties.getToken())
                .header("Content-Type", "application/json")
                .body(bodyStr)
                .timeout(TIMEOUT_MS)
                .execute()) {
            if (response.isOk()) {
                // 成功响应中 metadata.name 是本次 Workflow 实例名。
                String workflowName = new JSONObject(response.body())
                        .getByPath("metadata.name", String.class);
                log.info("Argo 工作流提交成功, template={}, parameters={}, workflow={}",
                        template, parameters, workflowName);
                return workflowName;
            }

            // 非 2xx 响应视为失败，记录状态码和响应体，便于排查模板名、权限、参数等问题。
            log.error("Argo 工作流提交失败, template={}, status={}, body={}",
                    template, response.getStatus(), response.body());
            throw new RuntimeException("Argo 工作流提交失败, status=" + response.getStatus()
                    + ", body=" + response.body());
        }
    }

    /**
     * 提交事件解析工作流。
     *
     * 这个方法是“尽力而为”：Argo 失败只记录日志，不向上抛异常。
     * 原因是事件入库已经完成，不能因为后续工作流提交失败而影响 Kafka 消息消费主流程。
     */
    public void submitEventProcess(String eventInfoId, String filesJson) {
        Map<String, String> parameters = new LinkedHashMap<>();
        parameters.put("event_info_id", eventInfoId);
        parameters.put("files", filesJson);

        // 工作流名称前缀，例如 event-process-10001-xxxxx。
        String generateName = "event-process-" + eventInfoId + "-";
        try {
            submit(argoProperties.getEventTemplate(), parameters, generateName);
        } catch (Exception e) {
            log.error("事件 Argo 工作流提交异常, eventInfoId={}", eventInfoId, e);
        }
    }

    /**
     * 提交质检工作流。
     *
     * @param sliceId 逻辑切片 ID
     * @param startTimeNs 切片开始时间，单位是纳秒
     * @param endTimeNs 切片结束时间，单位是纳秒
     * @param fileListJson 需要质检的文件列表 JSON
     */
    public void submitQc(String sliceId, String startTimeNs, String endTimeNs, String fileListJson) {
        Map<String, String> parameters = new LinkedHashMap<>();
        parameters.put("slice_id", sliceId);
        parameters.put("start_time_ns", startTimeNs);
        parameters.put("end_time_ns", endTimeNs);
        parameters.put("file_list", fileListJson);

        // 工作流名称前缀，例如 sacp-dms-manual-qc-20001-xxxxx。
        String generateName = "sacp-dms-manual-qc-" + sliceId + "-";
        try {
            submit(argoProperties.getManualQcTemplate(), parameters, generateName);
        } catch (Exception e) {
            log.error("质检 Argo 工作流提交异常, sliceId={}", sliceId, e);
        }
    }

    /**
     * 提交数据解析工作流。
     *
     * @param s3Path 数据在 S3 上的路径
     * @param dataId 数据主表 ID
     * @param visEnable 是否生成可视化结果
     * @param truthFlag true 表示真值数据，false 表示 ADC 原始数据
     */
    public void submitDataProcess(String s3Path, String dataId, Boolean visEnable, boolean truthFlag) {
        Map<String, String> parameters = new LinkedHashMap<>();
        parameters.put("s3_path", s3Path);
        parameters.put("data_id", dataId);
        parameters.put("vis_enable", String.valueOf(visEnable));
        parameters.put("truth_flag", String.valueOf(truthFlag));

        // 真值和 ADC 用不同前缀，方便在 Argo 页面按名称识别工作流来源。
        String generateName = truthFlag
                ? "data-process-truth-" + dataId + "-"
                : "data-process-adc-" + dataId + "-";

        try {
            submit(argoProperties.getTemplate(), parameters, generateName);
        } catch (Exception e) {
            log.error("Argo 工作流提交异常, dataId={}", dataId, e);
        }
    }
}
```

## 3. 被调用的项目内方法与配置类

`ArgoWorkflowClient` 调用了 `ArgoProperties` 的多个 getter。源码中没有手写 getter，因为 `@Data` 由 Lombok 在编译期生成。

```java
package com.seres.sacp.dms.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Data
@Component
@RefreshScope
@ConfigurationProperties(prefix = "argo")
public class ArgoProperties {

    // Argo Server 地址，对应配置 argo.endpoint，也可通过 ARGO_ENDPOINT 覆盖。
    private String endpoint;

    // Argo API Bearer Token，对应配置 argo.token，也可通过 ARGO_TOKEN 覆盖。
    private String token;

    // WorkflowTemplate 所在 Kubernetes/Argo 命名空间，对应 argo.namespace。
    private String namespace;

    // 数据解析 WorkflowTemplate 名称，对应 argo.template。
    private String template;

    // 事件解析 WorkflowTemplate 名称，对应 argo.event-template。
    private String eventTemplate;

    // 手动质检 WorkflowTemplate 名称，对应 argo.manual-qc-template。
    private String manualQcTemplate;
}
```

配置来源在：

- `sacp-dms-data-service/src/main/resources/application-dev.yml`
- `sacp-dms-data-service/src/main/resources/application-local.yml`

关键配置项：

```yaml
argo:
  endpoint: ${ARGO_ENDPOINT:https://siop-dev.seres.cn/argo}
  token: ${ARGO_TOKEN:...}
  namespace: ${ARGO_NAMESPACE:sacp-dms-argo}
  template: ${ARGO_TEMPLATE:sacp-dms-data-process-with-qc}
  event-template: ${ARGO_EVENT_TEMPLATE:sacp-dms-event-parse}
  manual-qc-template: ${ARGO_QC_TEMPLATE:sacp-dms-manual-qc}
```

## 4. 关键外部方法说明

这些方法来自 Hutool 或 JDK，不在本项目源码中，但理解它们能看懂主流程：

- `StrUtil.removeSuffix(text, "/")`：如果字符串以 `/` 结尾则删除，避免拼 URL 出现 `//api`。
- `StrUtil.isNotBlank(text)`：判断字符串不是 `null`、不是空串、不是纯空白。
- `new JSONArray().put(value)`：向 JSON 数组追加一个元素。
- `new JSONObject().set(key, value)`：向 JSON 对象设置字段，并返回当前对象，支持链式调用。
- `JSONObject#getByPath("metadata.name", String.class)`：按路径读取嵌套字段。
- `HttpRequest.post(url)`：创建 POST 请求。
- `header(...)`、`body(...)`、`timeout(...)`：设置请求头、请求体和超时时间。
- `execute()`：真正发送 HTTP 请求，返回 `HttpResponse`。
- `HttpResponse#isOk()`：状态码为 2xx 时通常返回 true。
- `HttpResponse#body()`：读取响应体字符串。
- `HttpResponse#getStatus()`：读取 HTTP 状态码。
- `try (HttpResponse response = ...)`：try-with-resources 语法，请求结束后自动关闭响应资源。

## 5. 项目中哪些地方调用了这个类

### 5.1 IndexController：手动触发 Argo

位置：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/common/controller/IndexController.java`

```java
@PostMapping("/argo/submit")
public CommonResult<String> submitArgo(@RequestParam String template,
                                       @RequestBody Map<String, String> parameters) {
    String workflowName = argoWorkflowClient.submit(template, parameters);
    return CommonResult.success(workflowName);
}
```

用途：提供一个 HTTP 接口 `/index/argo/submit`，让调用方手动传入模板名和参数，直接提交 Argo 工作流。这里调用的是通用 `submit(template, parameters)`，如果 Argo 失败，异常会向上抛给 Web 层。

### 5.2 EventCompleteConsumer：事件完成后触发事件解析

位置：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/event/mq/EventCompleteConsumer.java`

```java
if (message.getFiles() != null && !message.getFiles().isEmpty()) {
    String filesJson = objectMapper.writeValueAsString(message.getFiles());
    argoWorkflowClient.submitEventProcess(String.valueOf(event.getEventInfoId()), filesJson);
}
```

调用链：

1. Kafka 收到 `sdi.event.complete` 事件完成消息。
2. 消息落库为事件记录。
3. 如果消息里有文件列表，把文件列表序列化为 JSON。
4. 调用 `submitEventProcess(...)`，提交事件解析工作流。

### 5.3 DataUploadConsumer：数据上传后触发数据解析

位置：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/info/mq/DataUploadConsumer.java`

```java
if (dynamicConfig.getArgoFlag() == 0) {
    kafkaTemplate.send(TOPIC_DATA_PARSE_START, JSONUtil.toJsonStr(parseMsg));
} else {
    argoWorkflowClient.submitDataProcess(parseMsg.getS3Path(), parseMsg.getDataId(),
            parseMsg.getVisEnable(), parseMsg.isTruthFlag());
}
```

调用链：

1. Kafka 收到 `sacp-dms-data-upload` 数据上传消息。
2. 消息转换为 `DataInfoCreateDTO` 并写入数据库。
3. 根据 `dynamicConfig.getArgoFlag()` 判断解析触发方式。
4. `argoFlag == 0` 时发 Kafka 消息；否则直接调用 Argo。
5. 调用 `submitDataProcess(...)` 时会传入 S3 路径、数据 ID、是否可视化、是否真值。

### 5.4 DataLogicSliceServiceImpl：手动逻辑切片需要质检时触发质检

位置：`sacp-dms-data-service/src/main/java/com/seres/sacp/dms/module/info/service/impl/DataLogicSliceServiceImpl.java`

```java
if (needQc) {
    String fileListJson = buildFileListJson(dto.getDataId(), dto.getDataTypes(), absStartNs, absEndNs);
    argoWorkflowClient.submitQc(
            String.valueOf(entity.getId()),
            String.valueOf(absStartNs),
            String.valueOf(absEndNs),
            fileListJson);
    qc.setParseStatus(1);
    dataLogicSliceQcMapper.updateById(qc);
}
```

调用链：

1. 用户创建逻辑切片。
2. 如果 `needQc == true`，先构建切片对应的文件列表 JSON。
3. 调用 `submitQc(...)` 提交手动质检 Argo 工作流。
4. 随后把质检解析状态更新为 `1`，表示已触发解析。

### 5.5 DataUploadConsumerFullFlowTest：测试中 Mock 掉 Argo

位置：`sacp-dms-data-service/src/test/java/com/seres/sacp/dms/module/info/mq/DataUploadConsumerFullFlowTest.java`

```java
@MockBean
private ArgoWorkflowClient argoWorkflowClient;
```

用途：测试完整数据上传流程时，不希望真的调用外部 Argo 服务，因此用 Mockito/Spring `@MockBean` 替换真实 Bean。

## 6. 重要设计点

### 通用方法会抛异常，业务封装方法不抛

`submit(...)` 是底层通用方法，调用失败会抛 `RuntimeException`。这样手动接口等场景可以明确知道失败。

`submitEventProcess(...)`、`submitQc(...)`、`submitDataProcess(...)` 都捕获异常，只打日志。这类调用通常发生在 Kafka 消费、数据入库、切片创建之后，项目希望主业务不要被 Argo 临时故障打断。

### 参数最终都会变成字符串

Argo submit API 中的参数格式是 `name=value` 字符串数组，所以代码里会把 `Boolean`、`Long`、纳秒时间等都转成字符串。

例如 `submitDataProcess(...)` 生成：

```json
{
  "submitOptions": {
    "parameters": [
      "s3_path=raw/xxx/adc_xxx",
      "data_id=10001",
      "vis_enable=true",
      "truth_flag=false"
    ],
    "generateName": "data-process-adc-10001-"
  }
}
```

### generateName 只是前缀

`generateName` 不是最终完整名称。Argo 会在前缀后追加随机后缀，避免名称冲突。代码根据不同业务设置不同前缀，方便在 Argo 页面排查：

- `event-process-{eventInfoId}-`
- `sacp-dms-manual-qc-{sliceId}-`
- `data-process-truth-{dataId}-`
- `data-process-adc-{dataId}-`

## 7. 新手阅读建议

先看 `submit(...)`，理解“如何把 Java 参数变成 Argo HTTP 请求”；再看三个业务封装方法，理解“不同业务传了哪些参数”；最后看第 5 节调用点，串起数据上传、事件完成、手动质检三条真实业务链路。
