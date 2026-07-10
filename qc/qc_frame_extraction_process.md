# sacp-dms-data-process 质量检查 QC 抽帧流程讲解

本文讲解 `/home/c64586/server-and-process/sacp-dms-data-process` 项目里的质量检查（QC）抽帧流程。目标是让新手能从入口、采样点、文件下载、抽帧、输出命名、上传上报几个层面完整理解这套代码。

## 1. 先看结论

QC 抽帧做的事情可以概括为一句话：

> 给定一批上传到 S3 的数据文件，先计算若干个质检采样时间点 checkpoint，再在每个 checkpoint 附近从相机、点云、标定等 topic 中抽取连续帧，最后把抽出来的 `.jpeg`、`.pcd`、`.json` 上传到 S3 并调用后端接口登记。

主流程文件是：

| 文件 | 作用 |
| --- | --- |
| `data_parse/pipeline_qc.py` | QC 总入口：选择 ADC 或 Truth 分支、汇总输出、上传并上报 |
| `data_parse/pipeline_qc_adc.py` | ADC 分支：相机 JPEG、前激光 PCD、标定/底盘 JSON |
| `data_parse/pipeline_qc_truth.py` | Truth 分支：pandar 和 qt_* 激光 PCD |
| `data_parse/qc_utils.py` | 通用工具：checkpoint、切片时间索引、点云解析、PCD 写出 |
| `common/api_client.py` | 调用 checkpoint 接口和 QC 文件上报接口 |

重要提醒：当前 `app.py` 启动的 Kafka 消费者注册的是普通数据解析 topic，不是 QC 抽帧入口。QC 代码主要通过 `pipeline_qc.py` 的 CLI 或函数调用触发。

## 2. 入口在哪里

### 2.1 服务入口 app.py 不直接注册 QC

`app.py` 的职责是启动健康检查 HTTP 服务和 Kafka consumer：

```python
# sacp-dms-data-process/app.py  # 注释：标明这段示例代码来自哪个源码文件
from common.kafka_utils import run_consumer  # 注释：从公共 Kafka 工具中导入消费循环启动函数

def main():  # 注释：定义程序主入口函数，启动健康检查和 Kafka 消费
    health_thread = threading.Thread(target=_start_health_server, daemon=True, name="health-server")  # 注释：创建健康检查线程，daemon=True 表示主进程退出时线程自动结束
    health_thread.start()  # 注释：启动健康检查 HTTP 服务线程

    from kafka_consumer import TOPIC_HANDLERS  # 注释：在函数内部导入 topic 处理器映射，避免启动前过早加载依赖

    run_consumer(TOPIC_HANDLERS)  # 注释：启动 Kafka 消费循环，并把消息分发给 TOPIC_HANDLERS 中配置的函数
```

而 `data_parse/kafka_consumer.py` 当前注册的是：

```python
# sacp-dms-data-process/data_parse/kafka_consumer.py  # 注释：标明这段示例代码来自哪个源码文件
TOPIC_DATA_PARSE = "sacp-dms-data-parse-start"  # 注释：定义普通数据解析 Kafka topic 名称；当前不是 QC 入口

TOPIC_HANDLERS = {  # 注释：开始定义 Kafka topic 到处理函数的映射字典
    TOPIC_DATA_PARSE: handle_data_parse,  # 注释：把普通解析 topic 映射到 handle_data_parse 处理函数
}  # 注释：结束上方的多行结构
```

`handle_data_parse()` 里调用的是普通解析流水线：

```python
if truth_flag:  # 注释：如果消息标记为 Truth 数据，进入 Truth 普通解析分支
    logger.info(f"运行 truth 流水线")  # 注释：输出日志，说明即将处理 Truth 普通解析流程
    # run_pipeline_truth(truth_s3_path=s3_path, data_id=data_id, vis_enable=vis_enable)  # 注释：这是一行源码里的原注释，用来解释下面代码的意图
else:  # 注释：否则进入 ADC 普通解析分支
    logger.info(f" 运行 adc 流水线")  # 注释：输出日志，说明即将处理 ADC 普通解析流程
    run_pipeline_adc(s3_path=s3_path, data_id=data_id, vis_enable=vis_enable)  # 注释：调用 ADC 普通解析流水线；注意它不是 QC 抽帧入口
```

所以新手不要把 `python app.py` 理解成“自动跑 QC 抽帧”。QC 的主入口是下面这个文件。

### 2.2 QC 主入口 pipeline_qc.py

`data_parse/pipeline_qc.py` 里最核心的函数是 `run_pipeline_qc()`：

```python
def run_pipeline_qc(  # 注释：定义 QC 总入口函数，负责选择 ADC 或 Truth 分支
        data_id: str,  # 注释：参数 data_id 是后端数据唯一标识，贯穿 checkpoint、输出目录和上报
        s3_path_adc: str | None,  # 注释：参数 s3_path_adc 是 ADC 数据 S3 目录；有值时走 ADC 分支
        s3_path_truth: str | None,  # 注释：参数 s3_path_truth 是 Truth 数据 S3 目录；ADC 路径为空时使用
) -> None:  # 注释：函数签名结束，并声明返回值类型
    set_trace_id(generate_trace_id())  # 注释：设置本次任务日志 trace_id，方便按一次任务串联日志
    start_time = time.time()  # 注释：记录流水线开始时间，用于最后统计耗时

    from datetime import datetime  # 注释：导入时间工具，用当前日期生成 QC 输出目录
    data_date = datetime.now()  # 注释：获取当前日期时间，用来生成本地输出目录和 S3 日期路径
    output_base = WORK_DIR / data_id / "qc" / data_date.strftime("%Y/%m/%d")  # 注释：拼出本地 QC 输出根目录：workspace/data_id/qc/yyyy/mm/dd
    output_base.mkdir(parents=True, exist_ok=True)  # 注释：创建目录；parents=True 会连父目录一起创建，exist_ok=True 表示已存在也不报错

    all_outputs: list[Path] = []  # 注释：准备总输出列表，收集 ADC 或 Truth 分支生成的所有文件

    s3_path_for_name = s3_path_adc or s3_path_truth or ""  # 注释：优先选 ADC 路径，没有时选 Truth 路径，用于解析 dataName
    data_name = _extract_data_name_from_s3_path(s3_path_for_name)  # 注释：从 S3 路径中提取 dataName，后续上报给后端

    if s3_path_adc:  # 注释：如果传入 ADC S3 路径，就走 ADC QC 分支
        adc_out_dir = output_base / "adc"  # 注释：拼出 ADC 分支输出目录
        adc_out_dir.mkdir(parents=True, exist_ok=True)  # 注释：创建目录；parents=True 会连父目录一起创建，exist_ok=True 表示已存在也不报错
        adc_outputs = run_qc_adc(s3_path_adc, data_id, adc_out_dir)  # 注释：执行 ADC QC 抽帧，并返回生成的 JPEG、PCD、JSON 文件列表
        all_outputs.extend(adc_outputs)  # 注释：把 ADC 分支输出追加到总输出列表
    else:  # 注释：当前条件不满足时进入另一条分支
        truth_out_dir = output_base / "truth"  # 注释：拼出 Truth 分支输出目录
        truth_out_dir.mkdir(parents=True, exist_ok=True)  # 注释：创建目录；parents=True 会连父目录一起创建，exist_ok=True 表示已存在也不报错
        truth_outputs = run_qc_truth(s3_path_truth, data_id, truth_out_dir)  # 注释：执行 Truth QC 抽帧，并返回生成的 PCD 文件列表
        all_outputs.extend(truth_outputs)  # 注释：把 Truth 分支输出追加到总输出列表

    _upload_qc_files_and_report(all_outputs, data_id, data_name, data_date)  # 注释：统一上传所有抽帧文件，并把文件清单上报后端
```

这段代码说明了几个关键点：

1. 输出目录放在 `data_parse/workspace/<data_id>/qc/yyyy/mm/dd/`。
2. 如果传了 `s3_path_adc`，走 ADC 分支。
3. 否则走 Truth 分支。
4. 分支抽帧结束后统一调用 `_upload_qc_files_and_report()` 上传并上报。

当前 CLI 参数实际是：

```bash
cd /home/c64586/server-and-process/sacp-dms-data-process/data_parse

# ADC QC
python pipeline_qc.py --s3_path "s3://bucket/path/to/adc_xxx" --data_id "20567100000303"

# Truth QC
python pipeline_qc.py --s3_path "s3://bucket/path/to/truth_xxx" --truth_flag true --data_id "20567100000303"
```

## 3. 最核心的概念：checkpoint

QC 抽帧不是把所有数据都抽出来，而是先确定一些采样点。每个采样点叫 checkpoint，长这样：

```json
{
  "seq_no": "001",
  "ts": "1781187318000",
  "short_name": "003",
  "count": 3
}
```

字段含义：

| 字段 | 含义 |
| --- | --- |
| `seq_no` | checkpoint 序号，例如第 1 个采样点是 `001` |
| `ts` | 采样时间，单位是毫秒 |
| `short_name` | 采样点所在文件片段的短序号，通常来自文件名末尾的 `003`、`004` |
| `count` | 从该采样点开始连续取多少帧，当前默认是 3 |

代码里有一个非常重要的单位转换：

```python
ts_ns = int(cp["ts"]) * 1_000_000  # 注释：把 checkpoint 的毫秒时间转换为纳秒，才能和 MCAP log_time 或 PCD 时间戳比较
```

因为 checkpoint 的 `ts` 是毫秒，而 MCAP 消息的 `log_time` 是纳秒，所以抽帧前必须乘以 `1_000_000`。

## 4. checkpoint 怎么计算

### 4.1 旧的按全局时间均匀采样逻辑

`qc_utils.py` 中 `_compute_sample_seconds()` 是一个更基础的按时间范围均匀采样函数：

```python
def _compute_sample_seconds(t_start_ns: int, t_end_ns: int) -> list[int]:  # 注释：定义全局均匀采样函数，输入 PTP 纳秒范围，输出采样秒
    t_start_s = t_start_ns // 1_000_000_000  # 注释：把纳秒级开始时间向下取整为秒
    t_end_s   = t_end_ns   // 1_000_000_000  # 注释：把纳秒级结束时间向下取整为秒
    duration_s = t_end_s - t_start_s  # 注释：计算数据时长，单位是秒

    if duration_s < 2:  # 注释：数据不足 2 秒时不抽帧，避免首尾边界占满全部数据
        logger.warning(f"  [QC] duration {duration_s}s < 2s, giving up")  # 注释：输出警告日志，说明数据太短所以放弃采样
        return []  # 注释：返回空采样点列表，表示本段数据不参与 QC 抽帧

    if duration_s >= 7:  # 注释：根据条件决定是否执行下面缩进代码
        return [t_start_s + i * duration_s // 7 for i in range(1, 6)]  # 注释：长数据按 7 等份取中间 5 个采样秒
    else:  # 注释：当前条件不满足时进入另一条分支
        return list(range(t_start_s + 1, t_end_s))  # 注释：短数据逐秒采样，但去掉第一秒和最后一秒
```

这段代码的逻辑：

1. 把纳秒时间转成整数秒。
2. 数据小于 2 秒，不抽。
3. 数据大于等于 7 秒，把整体切成 7 份，取中间 5 个点。
4. 数据在 2 到 6 秒之间，逐秒取样，但去掉首尾秒。

### 4.2 当前 ADC/Truth 更常用的按片段采样逻辑

当前 ADC 和 Truth 分支主要调用 `_compute_checkpoints_from_segments()`。这个函数不只看全局起止时间，还会结合文件片段数量，尽量避开首尾片段。

核心代码：

```python
def _compute_checkpoints_from_segments(  # 注释：定义按文件片段生成 checkpoint 的函数
    segments: list[str],  # 注释：输入片段文件名列表，按文件顺序排列
    t_start_ns: int,  # 注释：整批数据开始 PTP 时间，单位纳秒
    t_end_ns: int,  # 注释：整批数据结束 PTP 时间，单位纳秒
    n: int = 5,  # 注释：给变量赋值，保存后续步骤要使用的中间结果
    seg_ptp_lookup: callable | None = None,  # 注释：给变量赋值，保存后续步骤要使用的中间结果
    count: int = 3,  # 注释：给变量赋值，保存后续步骤要使用的中间结果
) -> list[dict]:  # 注释：函数签名结束，并声明返回值类型
    if not segments:  # 注释：如果没有片段文件，就无法计算 checkpoint
        return []  # 注释：返回空采样点列表，表示本段数据不参与 QC 抽帧

    total = len(segments)  # 注释：统计片段总数
    span_ns = (t_end_ns - t_start_ns) / total if total else 0  # 注释：用总时长除以片段数，估算每个片段的平均时间跨度
    seg_starts_ns = [int(t_start_ns + i * span_ns) for i in range(total)]  # 注释：按平均跨度估算每个片段的起始 PTP 时间

    def _make_cps(seg_idx: int, cp_count: int, start_seq: int) -> list[dict]:  # 注释：定义内部 helper，为指定片段生成 checkpoint 列表
        seg_start, seg_end = _seg_range_ns(seg_idx)  # 注释：取得当前片段的 PTP 起止时间范围
        short_name = _extract_short_name(Path(segments[seg_idx]))  # 注释：从片段文件名提取 short_name，例如 003，用于后续少下载文件

        seg_start_s = seg_start // 1_000_000_000  # 注释：把片段开始时间从纳秒转换为整秒
        seg_end_s = seg_end // 1_000_000_000  # 注释：把片段结束时间从纳秒转换为整秒

        if cp_count == 1:  # 注释：如果当前片段只需要生成一个 checkpoint
            base_s = seg_start_s + 2  # 注释：采样点放在片段开始后 2 秒，避开切片首帧附近
            return [{  # 注释：返回包含 checkpoint 字典的列表
                "seq_no": f"{start_seq:03d}",  # 注释：checkpoint 序号，格式化成三位字符串，例如 001
                "ts": str(base_s * 1000),  # 注释：checkpoint 时间，单位毫秒，后续抽帧时会再转纳秒
                "short_name": short_name,  # 注释：checkpoint 对应的文件片段短序号
                "count": count,  # 注释：从该 checkpoint 开始连续抽取的帧数
            }]  # 注释：结束 checkpoint 字典和列表
```

继续看分支策略：

```python
if total == 1:  # 注释：只有一个片段时，在这个片段内部均匀取点
    # 1 个片段：两端各留 2s，剩余范围均匀取 n 个整数秒  # 注释：这是一行源码里的原注释，用来解释下面代码的意图
    ...  # 注释：省略此处代码，文档只展示关键流程
elif total == 2:  # 注释：只有两个片段时，没有中间片段可跳过
    # 2 个片段：没有"中间"可去掉，两段各取 3 个，共 6 个  # 注释：这是一行源码里的原注释，用来解释下面代码的意图
    for seg_idx in range(2):  # 注释：遍历第 0 和第 1 个片段
        result.extend(_make_cps(seg_idx, 3, seq))  # 注释：为当前片段生成 3 个 checkpoint 并追加到结果
        seq += 3  # 注释：输出序号向后移动 3 位，为下一批 checkpoint 留位置
else:  # 注释：当前条件不满足时进入另一条分支
    # 去掉首尾，处理中间片段  # 注释：这是一行源码里的原注释，用来解释下面代码的意图
    middle_indices = list(range(1, total - 1))  # 注释：生成中间片段索引，跳过首片段和尾片段
    t_prime = len(middle_indices)  # 注释：统计中间片段数量
    if t_prime <= n:  # 注释：如果中间片段数量不超过目标 checkpoint 数
        cp_count = (n + t_prime - 1) // t_prime  # 注释：计算每个中间片段分配几个 checkpoint，使用向上取整
        for idx in middle_indices:  # 注释：遍历每个被选中的中间片段
            result.extend(_make_cps(idx, cp_count, seq))  # 注释：为当前中间片段生成 checkpoint 并追加到结果
            seq += cp_count  # 注释：根据当前片段生成的 checkpoint 数推进输出序号
    else:  # 注释：当前条件不满足时进入另一条分支
        step = t_prime // n  # 注释：当中间片段很多时，计算隔多少片段取一个 checkpoint
        for i in range(0, t_prime, step):  # 注释：按 step 间隔遍历中间片段
            if len(result) >= n:  # 注释：如果已经生成目标数量的 checkpoint，就停止继续生成
                break  # 注释：提前结束循环，不再生成更多 checkpoint
            result.extend(_make_cps(middle_indices[i], 1, seq))  # 注释：为选中的中间片段生成 1 个 checkpoint
            seq += 1  # 注释：输出序号向后移动 1 位
```

这段逻辑可以用白话理解：

1. 只有 1 个片段：就在这个片段内部均匀取 5 个 checkpoint。
2. 有 2 个片段：两个片段都取，每段取 3 个，一共 6 个。
3. 多于 2 个片段：去掉首片段和尾片段，只在中间片段抽，避免边界数据不稳定。
4. 每个 checkpoint 默认 `count=3`，所以一个 checkpoint 会连续输出 3 帧。

### 4.3 checkpoint 要先查服务端

QC checkpoint 会写到后端服务里。抽帧前会先查，如果已经存在就复用，避免 ADC 和 Truth 各算各的导致不对齐。

API 客户端代码：

```python
class QcCheckpointClient:  # 注释：定义 checkpoint API 客户端类，封装查询和写入接口
    def update(self, data_id: str, checkpoints: list[dict]) -> bool:  # 注释：定义写入 checkpoint 的方法，成功返回 True
        url = f"{self.base_url}/qc/checkpoint/update"  # 注释：拼出 checkpoint 写入接口地址
        payload = {  # 注释：开始组装请求体 payload
            "dataId": data_id,  # 注释：请求体中携带当前数据 ID
            "checkpoint": _json.dumps(checkpoints, separators=(",", ":"))  # 注释：把 checkpoint 列表序列化成紧凑 JSON 字符串传给后端
        }  # 注释：结束上方的多行结构
        resp = requests.post(url, json=payload, timeout=_TIMEOUT)  # 注释：发送 POST 请求保存 checkpoint
        ...  # 注释：省略此处代码，文档只展示关键流程

    def get(self, data_id: str) -> list[dict] | None:  # 注释：定义查询 checkpoint 的方法，查不到或异常时返回 None
        url = f"{self.base_url}/qc/data/{data_id}"  # 注释：拼出 checkpoint 查询接口地址
        resp = requests.get(url, timeout=_TIMEOUT)  # 注释：发送 GET 请求查询服务端已有 checkpoint
        ...  # 注释：省略此处代码，文档只展示关键流程
```

ADC 分支里使用方式如下：

```python
client = get_qc_checkpoint_client()  # 注释：创建 checkpoint API 客户端
checkpoints = client.get(data_id)  # 注释：优先读取服务端已有 checkpoint

if checkpoints:  # 注释：服务端已有 checkpoint 时直接复用
    logger.info(f"[QC-ADC] checkpoint exists on server, seq_count={len(checkpoints)}, matching short_names")  # 注释：给变量赋值，保存后续步骤要使用的中间结果
else:  # 注释：当前条件不满足时进入另一条分支
    local_cps = _compute_checkpoints_from_segments(  # 注释：本地按片段和 PTP 时间计算 checkpoint
        video_names, t_start_ns, t_end_ns, seg_ptp_lookup=_video_ptp_lookup  # 注释：用 video 片段名、全局 PTP 起止时间和真实片段 PTP 回调计算 checkpoint
    )  # 注释：结束上方的多行结构
    ok = client.update(data_id, local_cps)  # 注释：把本地 checkpoint 写入后端，成功时其他分支可复用
    if ok:  # 注释：PCD 写出成功后才加入输出列表
        checkpoints = local_cps  # 注释：写入成功后使用本地计算结果
    else:  # 注释：当前条件不满足时进入另一条分支
        fallback = client.get(data_id)  # 注释：写入失败时再次查询，处理并发任务已写入的情况
        checkpoints = fallback or local_cps  # 注释：优先用服务端返回值；服务端仍没有时使用本地结果兜底
```

## 5. 文件片段怎么匹配

S3 上的数据是分片文件，例如：

```text
pointcloud_202606160710_003.mcap
struct_202606160710_003.mcap
video_202606160710_003.mcap
qt_front_202606160710_003.pcap
pandar_202606160710_003.pcap
```

代码会从文件名里提取 `003` 作为 `short_name`：

```python
def _extract_short_name(path: Path) -> str:  # 注释：定义从文件名提取片段短序号的函数，例如 003
    stem = path.stem  # 注释：取文件名去掉扩展名后的主体
    if stem.endswith("_0"):  # 注释：兼容以 _0 结尾的文件名
        stem = stem[:-2]  # 注释：去掉末尾 _0，保留真实片段名
    parts = stem.rsplit("_", 1)  # 注释：从右侧最后一个下划线切开文件名
    if len(parts) == 2 and parts[-1].isdigit():  # 注释：如果最后一段是数字，就把它当作片段序号
        return parts[-1]  # 注释：返回片段序号，例如 003
    return stem  # 注释：无法提取数字序号时，返回整个文件主体名兜底
```

然后根据 checkpoint 中的 `short_name` 只下载需要的分片，而不是把整个目录下载下来：

```python
def _filter_keys_by_short_names(  # 注释：定义按 short_name 过滤 S3 key 的函数，只保留要抽帧的片段
        keys: list[str],  # 注释：待筛选的 S3 key 列表
        short_names: set[str],  # 注释：目标片段短序号集合
) -> list[str]:  # 注释：函数签名结束，并声明返回值类型
    return [key for key in keys if _match_by_short_name(key, short_names)]  # 注释：只保留 short_name 命中的 S3 key
```

ADC 分支下载逻辑：

```python
short_names = {_normalize_short_name(cp["short_name"]) for cp in checkpoints if "short_name" in cp}  # 注释：checkpoint 对应的文件片段短序号

need_pc = _filter_keys_by_short_names(_s3_pc_keys, short_names)  # 注释：筛选需要下载的 pointcloud MCAP 文件
need_struct = _filter_keys_by_short_names(_s3_struct_keys, short_names)  # 注释：筛选需要下载的 struct MCAP 文件
need_video = _filter_keys_by_short_names(_s3_video_keys, short_names)  # 注释：筛选需要下载的 video MCAP 文件
need_keys = need_pc + need_struct + need_video  # 注释：合并三类待下载文件 key

_download_files_by_keys(need_keys, adc_dir)  # 注释：按 key 把需要的文件下载到 ADC 临时目录

pc_slices = sorted(adc_dir.glob("pointcloud_*.mcap"), key=lambda p: p.name)  # 注释：找出本地已下载的点云 MCAP 切片并排序
struct_slices = sorted(adc_dir.glob("struct_*.mcap"), key=lambda p: p.name)  # 注释：找出本地已下载的 struct MCAP 切片并排序
video_slices = sorted(adc_dir.glob("video_*.mcap"), key=lambda p: p.name)  # 注释：找出本地已下载的 video MCAP 切片并排序
```

## 6. 切片时间索引：怎么知道一个 checkpoint 落在哪个文件里

抽帧时必须先找到 `ts_ns` 对应的本地切片。工具函数是：

```python
def _build_slice_time_index(slices: list[Path]) -> list[tuple[Path, int, int]]:  # 注释：定义构建切片时间索引的函数，结果是 路径、开始时间、结束时间
    result = []  # 注释：准备保存函数输出或时间索引
    for p in slices:  # 注释：遍历输入的每个切片文件
        tr = get_ptp_time_range_from_mcap(p)  # 注释：读取当前 MCAP 切片的 PTP 起止时间
        if tr:  # 注释：只有成功读到时间范围才加入索引
            result.append((p, tr[0], tr[1]))  # 注释：保存切片路径、开始时间和结束时间
    return sorted(result, key=lambda x: x[1])  # 注释：按切片开始时间排序后返回索引

def _find_slice_for_ns(  # 注释：定义按纳秒时间查找所在切片的函数
    index: list[tuple[Path, int, int]],  # 注释：准备 PCD 时间索引，元素是 PCD 路径和纳秒时间戳
    target_ns: int,  # 注释：要定位的目标纳秒时间
) -> Path | None:  # 注释：函数签名结束，并声明返回值类型
    for path, s, e in index:  # 注释：遍历每个切片及其开始、结束时间
        if s <= target_ns <= e:  # 注释：判断目标时间是否落在当前切片范围内
            return path  # 注释：命中后返回当前切片路径
    return None  # 注释：没有任何切片命中时返回 None
```

`_build_slice_time_index()` 会得到这样的结构：

```text
[
  (video_..._003.mcap, 1781187316000000000, 1781187345000000000),
  (video_..._004.mcap, 1781187346000000000, 1781187375000000000),
]
```

然后 `_find_slice_for_ns()` 逐个判断：

```text
start_ns <= checkpoint_ns <= end_ns
```

命中的那个文件就是后续抽帧要打开的文件。

## 7. ADC 分支完整流程

ADC 分支函数是 `run_qc_adc()`，总体注释已经写得很清楚：

```python
def run_qc_adc(  # 注释：定义 ADC QC 主函数，负责 ADC 相机、点云和 JSON 抽取
        s3_path_adc: str,  # 注释：参数 s3_path_adc 是 ADC 数据 S3 目录；有值时走 ADC 分支
        data_id: str,  # 注释：参数 data_id 是后端数据唯一标识，贯穿 checkpoint、输出目录和上报
        output_dir: Path,  # 注释：参数 output_dir 是本地 QC 输出目录
) -> list[Path]:  # 注释：函数签名结束，并声明返回值类型
    """  # 注释：开始或结束函数说明字符串
    1. GET checkpoint API -> 已存在则直接用，跳转到步骤 4  # 注释：先查询服务端 checkpoint，已有就复用
    2. List S3 目录（不下载），对文件名分组（struct 为基准分组）  # 注释：只列举 S3 文件并按类型分组
    3. 按片段数计算 checkpoint，POST 上报  # 注释：本地计算采样点并写入后端
    4. 按 checkpoint 的 short_name 筛选各分组文件并下载  # 注释：只下载采样点所在片段，减少 IO
    5. 抽取:  # 注释：开始按 topic 抽出 QC 文件
       - 11 路 camera: 每路按 checkpoint ts 取第一帧 JPEG  # 注释：相机数据输出 JPEG
       - 前激光 lidar_front_pointcloud: 每 checkpoint ts 取第一帧 PCD  # 注释：前激光数据输出 PCD
       - _ADC_JSON_TOPICS 中的每个 topic: 每 checkpoint ts 取第一帧 JSON  # 注释：标定和底盘等结构化数据输出 JSON
    """  # 注释：开始或结束函数说明字符串
```

ADC 目录里的文件被分成三类：

```python
def _classify_adc_keys(keys: list[str]) -> tuple[list[str], list[str], list[str]]:  # 注释：定义 ADC S3 文件分类函数，把文件分成 pointcloud、struct、video 三类
    pc_keys: list[str] = []  # 注释：准备三个列表分别保存 pointcloud、struct、video 文件 key
    struct_keys: list[str] = []  # 注释：准备三个列表分别保存 pointcloud、struct、video 文件 key
    video_keys: list[str] = []  # 注释：准备三个列表分别保存 pointcloud、struct、video 文件 key
    for key in keys:  # 注释：遍历 S3 目录中列出的每个 key
        name = Path(key).name  # 注释：只取 S3 key 的文件名部分
        if name.startswith("pointcloud_") and name.endswith(".mcap"):  # 注释：如果文件名表示 pointcloud MCAP
            pc_keys.append(key)  # 注释：把当前 key 加入点云文件列表
        elif name.startswith("struct_") and name.endswith(".mcap"):  # 注释：如果文件名表示 struct MCAP
            struct_keys.append(key)  # 注释：把当前 key 加入 struct 文件列表
        elif name.startswith("video_") and name.endswith(".mcap"):  # 注释：如果文件名表示 video MCAP
            video_keys.append(key)  # 注释：把当前 key 加入 video 文件列表
    return (  # 注释：返回三类按文件名排序后的 key 列表
        sorted(pc_keys, key=lambda k: Path(k).name),  # 注释：按文件名排序 pointcloud key，保证处理顺序稳定
        sorted(struct_keys, key=lambda k: Path(k).name),  # 注释：按文件名排序 struct key，保证处理顺序稳定
        sorted(video_keys, key=lambda k: Path(k).name),  # 注释：按文件名排序 video key，保证处理顺序稳定
    )  # 注释：结束上方的多行结构
```

### 7.1 ADC 相机 JPEG 抽帧

相机 topic 是从 `video_*.mcap` 的 summary 里发现的：

```python
def _discover_camera_topics(video_slices: list[Path]) -> list[str]:  # 注释：定义从 video MCAP 中发现所有相机 topic 的函数
    topics: set[str] = set()  # 注释：用集合保存相机 topic，自动去重
    for vslice in video_slices:  # 注释：遍历每个 video 切片
        with open(vslice, "rb") as f:  # 注释：以二进制方式打开 MCAP 文件
            reader = make_reader(f)  # 注释：创建 MCAP reader，用来读取 summary 或消息
            summary = reader.get_summary()  # 注释：读取 MCAP summary，其中包含 channel 和 topic 信息
            for ch in summary.channels.values():  # 注释：遍历 summary 中的所有 channel
                tl = ch.topic.lower()  # 注释：把 topic 转成小写，便于关键词匹配
                if any(kw in tl for kw in _ADC_CAMERA_KEYWORDS) and "calibration" not in tl:  # 注释：topic 包含 camera/image 且不是 calibration 时认为是相机 topic
                    topics.add(ch.topic)  # 注释：把发现的相机 topic 加入集合
    return sorted(topics)  # 注释：返回排序后的相机 topic 列表
```

然后每个 `topic × checkpoint` 组成一个任务：

```python
tasks = [  # 注释：开始构造相机抽帧任务列表
    (topic, cp)  # 注释：一个任务由一个相机 topic 和一个 checkpoint 组成
    for topic in camera_topics  # 注释：外层遍历所有相机 topic
    for cp in checkpoints  # 注释：内层遍历所有 checkpoint
]  # 注释：结束列表结构
max_workers = min(len(tasks), _CAMERA_MAX_WORKERS)  # 注释：线程数取任务数和最大并发限制中的较小值
```

单个任务的关键逻辑：

```python
def _camera_task(topic: str, cp: dict) -> list[tuple[Path | None, str]]:  # 注释：定义单个相机 topic 在单个 checkpoint 上的抽帧任务
    topic_safe = _topic_to_filename_prefix(topic)  # 注释：把 topic 转为文件名前缀，例如 /a/b 变成 a_b
    seq_no = cp["seq_no"]  # 注释：读取 checkpoint 的原始序号，例如 001
    start_seq = cp_start_seq.get(seq_no, int(seq_no))  # 注释：计算该 checkpoint 输出文件的起始序号  cp_start_seq = _expand_checkpoints(checkpoints)
"""
def _expand_checkpoints(checkpoints: list[dict]) -> dict[str, int]:
    result: dict[str, int] = {}
    running_seq = 1
    for cp in checkpoints:
        result[cp["seq_no"]] = running_seq  # 记录当前 checkpoint 的输出起始序号
        running_seq += _cp_count(cp) # 当前 checkpoint 可能抽多帧，所以后一个 checkpoint 的起始序号要往后跳
    return result
"""
    count = _cp_count(cp)  # 注释：读取 checkpoint 的 count，决定连续抽几帧
    ts_ns = int(cp["ts"]) * 1_000_000  # 注释：把 checkpoint 的毫秒时间转换为纳秒，才能和 MCAP log_time 或 PCD 时间戳比较

    slice_path = _find_slice_for_ns(video_index, ts_ns)  # 注释：根据 checkpoint 时间在 video 时间索引中找切片
    if slice_path is None:  # 注释：根据条件决定是否执行下面缩进代码
        return [None, f"seq={start_seq:03d} {topic}: no slice"]  # 注释：如果找不到 video 切片，返回失败信息

    frames = _extract_camera_jpegs_from_slice(  # 注释：调用相机抽帧函数，得到 JPEG 字节和真实 log_time
        slice_path, topic, ts_ns, tmp_dec, count=count  # 注释：传入切片、topic、checkpoint 时间、临时目录和连续帧数
    )  # 注释：结束上方的多行结构

    for frame_offset, (jpeg_bytes, log_time) in enumerate(frames):  # 注释：遍历抽到的 JPEG 帧
        seq_idx = start_seq + frame_offset  # 注释：按 checkpoint 起始序号和帧偏移计算最终 seq
        out_name = f"{topic_safe}_{log_time}_{seq_idx:03d}.jpeg"  # 注释：生成 JPEG 输出文件名：topic_safe_log_time_seq.jpeg
        out_path = output_dir / out_name  # 注释：拼出本地输出文件完整路径
        out_path.write_bytes(jpeg_bytes)  # 注释：把 JPEG 二进制数据写入本地文件
        out_path.with_suffix(".topic").write_text(topic, encoding="utf-8")  # 注释：写同名 .topic 文件保存原始 topic，便于上报
```

这段代码做了四件事：

1. 把 checkpoint 毫秒时间转成纳秒。
2. 找到对应 `video_*.mcap`。
3. 从切片里解码 H265，取 `log_time >= ts_ns` 的连续 `count` 帧。
4. 写出 JPEG，并额外写一个同名 `.topic` 文件保存原始 topic。

`.topic` 文件很重要，因为输出文件名会把 `/` 替换成 `_`，如果不保存原始 topic，后面上报时可能无法可靠还原。

相机真正的 H265 到 JPEG 解码在这里：

```python
def _extract_camera_jpegs_from_slice(  # 注释：定义从一个 video MCAP 切片中解码相机 JPEG 的函数
        slice_path: Path,  # 注释：输入 MCAP 切片路径
        topic: str,  # 注释：参数 topic 是要处理的 ROS topic 名称
        ts_ns: int,  # 注释：参数 ts_ns 是 checkpoint 纳秒时间，用来和消息时间比较
        tmp_dir: Path,  # 注释：ffmpeg 解码时使用的临时目录
        count: int = 1,  # 注释：给变量赋值，保存后续步骤要使用的中间结果
) -> list[tuple[bytes, int]] | None:  # 注释：函数签名结束，并声明返回值类型
    chunks: list[bytes] = []  # 注释：保存从消息中提取出的 H265 数据块
    target_indices: list[int] = []  # 注释：保存目标帧在 H265 流中的帧序号
    target_log_times: list[int] = []  # 注释：保存目标帧对应的真实 log_time
    frame_count = 0  # 注释：从 0 开始记录当前扫描到第几帧

    with open(slice_path, "rb") as f:  # 注释：以二进制方式打开 MCAP 文件
        reader = make_reader(f, decoder_factories=[DecoderFactory()] if use_decoder else [])  # 注释：创建 MCAP reader，用来读取 summary 或消息
        for schema, channel, message, decoded in reader.iter_decoded_messages(topics=[topic]):  # 注释：遍历指定相机 topic 的已解码消息
            h265 = _extract_h265_payload(message.data, decoded)  # 注释：从当前 MCAP 消息中提取 H265 编码数据
            if h265 is None:  # 注释：无法提取 H265 时跳过当前消息
                continue  # 注释：跳过当前无法处理的记录，继续下一条
            chunks.append(h265)  # 注释：把当前帧 H265 数据追加到连续 H265 流
            if len(target_indices) < count and message.log_time >= ts_ns:  # 注释：如果当前帧已经到达 checkpoint 且还没取够帧数
                target_indices.append(frame_count)  # 注释：记录当前帧序号，后面让 ffmpeg 只解码这些帧
                target_log_times.append(message.log_time)  # 注释：保存目标帧对应的真实 log_time
            frame_count += 1  # 注释：处理完当前帧后，帧计数加 1

    h265_data = b"".join(chunks)  # 注释：把多个 H265 数据块拼成一个连续字节流
    jpegs = _decode_h265_stream_to_jpegs(  # 注释：调用 ffmpeg 解码函数，只解码目标帧
        h265_data, tmp_dir, select_indices=target_indices  # 注释：传入 H265 字节流、临时目录和目标帧序号
    ) # 详细注释版 ./decode_h265_stream_to_jpegs_annotated.md
```

这里的思路是：

1. 顺序扫描 MCAP 中某一路相机 topic。
2. 把 H265 数据块收集到 `chunks`。
3. 记录满足 `message.log_time >= ts_ns` 的目标帧在 H265 流里的序号。
4. 调用 ffmpeg 只解码目标帧，避免把全部帧都落盘。

ffmpeg 命令由 `_decode_h265_stream_to_jpegs()` 组装：

```python
cmd = [  # 注释：开始组装 ffmpeg 命令参数
    "ffmpeg", "-y",  # 注释：调用 ffmpeg，-y 表示允许覆盖临时输出文件
    "-c:v", "hevc",  # 注释：指定输入视频编码为 HEVC/H265
    "-i", str(h265_path),  # 注释：指定 ffmpeg 输入文件路径
    "-q:v", "1",  # 注释：设置 JPEG 输出质量，1 表示高质量
    "-threads", str(os.cpu_count() or 4),  # 注释：设置 ffmpeg 解码线程数，默认使用 CPU 核数
]  # 注释：结束列表结构
if select_indices:  # 注释：如果指定了目标帧，就只解码这些帧
    expr = "+".join(f"eq(n\\,{i})" for i in sorted(set(select_indices)))  # 注释：生成 ffmpeg select 表达式，例如 eq(n,3)+eq(n,4)
    cmd += ["-vf", f"select='{expr}'", "-vsync", "0"]  # 注释：把 select 过滤器追加到 ffmpeg 参数中，避免输出非目标帧
cmd.append(str(tmp_dir / "frame_%06d.jpg"))  # 注释：指定 JPEG 输出文件名模板
```

所以运行 ADC 相机抽帧的机器必须安装 `ffmpeg`。

### 7.2 ADC 前激光 PCD 抽帧

QC 抽帧流程中，ADC 分支只从 pointcloud_*.mcap 中抽 /sensor/lidar_front_pointcloud。不会抽 rear、left、right、pandar、qt 等其他激光 PCD。Truth 分支才会处理 pandar、qt_front、qt_left 等多类激光。

ADC 点云文件来自 `pointcloud_*.mcap`，目标 topic 包含：

```python
_ADC_LIDAR_FRONT_TOPIC = "/sensor/lidar_front_pointcloud"  # 注释：定义 ADC 前激光 topic 关键词
```

抽帧核心代码：

```python
pc_index = _build_slice_time_index(pc_slices)  # 注释：为点云 MCAP 切片建立时间索引

for cp in checkpoints:  # 注释：遍历每个 checkpoint 进行点云抽帧
    count = _cp_count(cp)  # 注释：读取 checkpoint 的 count，决定连续抽几帧
    seq_no = cp["seq_no"]  # 注释：读取 checkpoint 的原始序号，例如 001
    start_seq = cp_start_seq.get(seq_no, int(seq_no))  # 注释：计算该 checkpoint 输出文件的起始序号
    ts_ns = int(cp["ts"]) * 1_000_000  # 注释：把 checkpoint 的毫秒时间转换为纳秒，才能和 MCAP log_time 或 PCD 时间戳比较

    slice_path = _find_slice_for_ns(pc_index, ts_ns)  # 注释：根据 checkpoint 时间在点云切片索引中找文件
    if slice_path is None:  # 注释：根据条件决定是否执行下面缩进代码
        continue  # 注释：跳过当前无法处理的记录，继续下一条

    frames = _extract_frames_after(slice_path, lidar_topic, ts_ns, count)  # 注释：从点云切片中读取 checkpoint 后的连续消息帧 紧接着有详解

    for frame_offset, (msg_data, schema, log_time) in enumerate(frames):  # 注释：遍历抽到的点云消息、schema 和真实 log_time
        seq_idx = start_seq + frame_offset  # 注释：按 checkpoint 起始序号和帧偏移计算最终 seq
        out_name = f"{lidar_safe}_{log_time}_{seq_idx:03d}.pcd"  # 注释：生成 PCD 输出文件名：topic_safe_log_time_seq.pcd
        out_path = output_dir / out_name  # 注释：拼出本地输出文件完整路径
        ok = extract_pcd_from_message(msg_data, schema, out_path)  # 注释：把 PointCloud2 消息解析并写成 PCD 文件 紧接着有详解
        if ok:  # 注释：PCD 写出成功后才加入输出列表
            outputs.append(out_path)  # 注释：把输出文件路径加入列表，后面统一上传
            out_path.with_suffix(".topic").write_text(lidar_topic, encoding="utf-8")  # 注释：写同名 .topic 文件保存点云原始 topic
```

`_extract_frames_after()` 是通用 MCAP 抽帧函数：

```python
def _extract_frames_after(  # 定义从 MCAP 切片中读取 checkpoint 后连续消息帧的通用函数
    slice_path: Path,  # 输入 MCAP 切片路径
    topic: str,  # 参数 topic 是要处理的 ROS topic 名称
    ts_ns: int,  # 参数 ts_ns 是 checkpoint 纳秒时间，用来和消息时间比较
    count: int = 1,  # 给变量赋值，保存后续步骤要使用的中间结果
) -> list[tuple[bytes, object, int]]:  # 注释：函数签名结束，并声明返回值类型
    result: list[tuple[bytes, object, int]] = []  # 注释：准备保存抽到的消息帧
    with open(slice_path, "rb") as f:  # 注释：以二进制方式打开 MCAP 文件
        reader = make_reader(f)  # 注释：创建 MCAP reader，用来读取 summary 或消息
        summary = reader.get_summary()  # 注释：读取 MCAP summary，其中包含 channel 和 topic 信息
        for _, channel, message in reader.iter_messages(  # 注释：开始组装多行结构
            topics=[topic],  # 注释：只读取指定 topic 的消息
            start_time=ts_ns, # 从 `start_time=ts_ns` 开始取消息
        ): 
            result.append((message.data, summary.schemas.get(channel.schema_id), message.log_time))  # 注释：保存消息字节、schema 和真实 log_time
            if len(result) >= count: 
                break
    return result
```

它的含义非常直接：

1. 打开一个 MCAP 切片。
2. 只遍历指定 topic。
3. 从 `start_time=ts_ns` 开始取消息。
4. 取够 `count` 帧就停止。

点云消息转 PCD 的代码：

```python
# 从 MCAP 消息中提取点云并可选写出 PCD 的函数。
def extract_pcd_from_message(  
    message_data: bytes,  # 输入参数 message_data：PointCloud2 CDR 的原始二进制消息。
    schema,  # 输入参数 schema：这条消息对应的 MCAP schema，用来辅助解析字段结构。
    output_path: Path | None = None,  # 输入参数 output_path：输出 PCD 路径；为 None 时只解析、不写文件。
    *, 
    front_x_filter: bool = False,  # 输入参数 front_x_filter：是否启用前方 0 到 80 米裁剪，默认不启用以保持兼容。
) -> bool:  # 返回 bool：True 表示解析/写出成功，False 表示失败。
    """从一帧 PointCloud2 CDR 消息解析点云，按需写出 PCD ASCII 文件。
    Args:  # 参数说明开始。
        message_data: PointCloud2 CDR 原始字节。  # message_data 是未解析的原始消息内容。
        schema: 对应 MCAP schema 对象。  # schema 告诉解析函数如何理解原始字节。
        output_path: 输出 PCD 路径；None 时只解析不落盘。  # 控制是否生成 PCD 文件。
        front_x_filter: True 时仅保留 0<=x<=80m 的点。默认 False，保证老调用方行为不变。
    """  
    result = _parse_pc2_to_points(message_data, schema)  # 调用已有解析逻辑，把二进制 PointCloud2 消息转成 points 和 fields。 ./qc中有注释版
    if result is None:  # 如果解析失败，内部约定会返回 None。
        return False  # 解析失败时直接返回 False，告诉调用方本帧没有成功处理。
    points, fields = result  # 解包解析结果：points 是点列表，fields 是字段定义列表。

    if _needs_unit_conversion(fields):  # 判断当前点云坐标是否需要从厘米转换成米。 紧接着有详解
        points = _convert_cm_to_m(points)  # 如果需要转换，就把所有点的 x/y/z 从 cm 转成 m。
        fields = [  # 同步重建字段定义，确保 x/y/z 的数据类型也改成 float32。
            {**f, "datatype": 7} if f["name"] in ("x", "y", "z") else f  # 如果字段是 x/y/z，就复制原字段并把 datatype 改为 7；其他字段保持原样。
            for f in fields  # 遍历原始 fields 列表中的每个字段定义。
        ]  # 字段列表重建结束。

    # 必须放在 cm->m 之后，否则 ADC INT16 cm 会把 80m 误判为 80cm。
    if front_x_filter:  # 只有调用方显式传 front_x_filter=True 时才执行前方裁剪。
        points = filter_front_x_points(points)  # 对已经转换成米的点列表应用 0 到 80 米过滤。 紧接着有详细讲解

    if output_path is not None:  # 如果调用方提供了输出路径，就需要把点云写成 PCD 文件。
        _write_pcd_ascii(output_path, points, fields)  # 调用已有 ASCII PCD 写入函数，把过滤后的 points 写到 output_path。
    return True  # 能走到这里说明处理成功，返回 True。
```

ADC 前激光有一个特殊点：如果点云字段是 `INT16`，坐标单位按厘米处理，需要转成米：

```python
def _needs_unit_conversion(fields: list[dict]) -> bool:  # 注释：定义判断点云坐标是否要从厘米转米的函数
    xyz = [f for f in fields if f["name"] in ("x", "y", "z")]  # 注释：找出字段定义中的 x、y、z 坐标字段
    return bool(xyz) and xyz[0]["datatype"] == 3  # INT16  # 注释：x/y/z 字段改成 datatype=7 即 FLOAT32，其它字段保持不变

def _convert_cm_to_m(points: list[dict]) -> list[dict]:  # 注释：定义把点云 x/y/z 坐标从厘米转换为米的函数
    for pt in points:  # 注释：遍历每一个点
        for axis in ("x", "y", "z"):  # 注释：依次处理 x、y、z 三个坐标轴
            if axis in pt:  # 注释：只有当前点包含该坐标轴时才转换
                pt[axis] = float(pt[axis]) / 100.0  # 注释：厘米除以 100 得到米
    return points  # 注释：返回转换后的点列表
```

```python
# 过滤已经解析成 Python 字典列表的点云点
def filter_front_x_points(  
    points: list[dict],  # 输入参数 points：每个点是一个 dict，里面应包含 x、y、z 等字段。
    min_x_m: float = _QC_FRONT_X_MIN_M,  # 输入参数 min_x_m：允许保留的最小 x，默认使用全局常量 0 米。
    max_x_m: float = _QC_FRONT_X_MAX_M,  # 输入参数 max_x_m：允许保留的最大 x，默认使用全局常量 80 米。
) -> list[dict]:  # 返回值类型是 list[dict]，也就是过滤后的点列表。
    """保留 x 正方向指定距离内的点。

    该函数用于已解析出的 PointCloud2 点列表。调用前必须确保坐标单位已经是米。
    缺失 x、x 无法转 float、x 为 NaN/Inf 的点都丢弃，避免写出不可判断的点。
    """
    filtered: list[dict] = []  # 创建空列表，用来收集通过 x 范围检查的点。
    for pt in points:  # 逐个遍历输入点列表中的每一个点。
        try:  # 尝试读取并转换当前点的 x 坐标。
            x = float(pt["x"])  # 从点字典中取出 x 字段，并转成浮点数，方便做数值比较。
        except (KeyError, TypeError, ValueError):  # 如果没有 x、点不是合法字典、或 x 不能转成数字，就进入这里。
            continue  # 跳过这个异常点，继续处理下一个点。

        # 非有限值会破坏后续可视化和校验，直接丢弃。
        if math.isfinite(x) and min_x_m <= x <= max_x_m:
            filtered.append(pt)  # 当前点满足条件，把它加入过滤后的结果列表。

    logger.debug(  # 打一条 debug 日志，方便排查过滤前后点数是否符合预期。
        "  [qc-front-x] points before=%d after=%d range=[%.2f, %.2f]",  # 日志模板，依次打印过滤前数量、过滤后数量、最小 x、最大 x。
        len(points),  # 第一个占位符：输入点总数。
        len(filtered),  # 第二个占位符：过滤后剩余点数。
        min_x_m,  # 第三个占位符：本次使用的最小 x。
        max_x_m,  # 第四个占位符：本次使用的最大 x。
    )
    return filtered  # 返回过滤后的点列表。
```



### 7.3 ADC JSON 抽取：标定和底盘

ADC 还会从 struct MCAP 中抽 JSON 类数据：

```python
_ADC_JSON_TOPICS: tuple[JsonTopicDef, ...] = (  # 注释：定义 ADC 需要导出 JSON 的 topic 配置
    JsonTopicDef(topic="/calib/calib_param", label="calib"),  # 注释：配置标定参数 topic，输出标签为 calib
    JsonTopicDef(topic="/veh/chassis_report", label="chassis"),  # 注释：配置底盘报文 topic，输出标签为 chassis
)
```

调用位置：

```python
if struct_slices:  # 如果存在 struct 切片，就可以抽 JSON 类 topic
    for topic_def in _ADC_JSON_TOPICS:  # 遍历每个 JSON topic 配置
        json_outputs = _extract_json_topic(  # 调用 JSON topic 抽取函数
            topic_def, struct_slices, checkpoints, output_dir  # 传入 topic 配置、struct 切片、checkpoint 和输出目录
        )  # 在./qc中有详解
        outputs.extend(json_outputs)  # 注释：把 JSON 输出追加到总输出列表
```

`_extract_json_topic()` 会做几件事：

1. 在 struct 切片里找到真实 topic 名。
2. 按 checkpoint 找到对应切片。
3. 取 `log_time >= ts_ns` 的连续帧。
4. 反序列化 ROS2 CDR 消息。
5. 写成 JSON 文件。

对于 `/calib/calib_param`，代码还有缓存逻辑：找到第一帧有效标定后，后续 checkpoint 复用这帧标定。这是因为标定数据低频，不一定每个 checkpoint 后面都有新消息。

## 8. Truth 分支完整流程

Truth 分支和 ADC 的差异很大。它的输入主要是 `.pcap`，不是 MCAP 点云消息。当前流程是：

```text
pcap -> pcl_tool 转 PCD 目录 -> 从 PCD 文件名读 PTP 时间 -> 按 checkpoint 找 PCD -> 前方 0~80m 裁剪 -> 写 QC 输出
```

主函数注释：

```python
def run_qc_truth(  # 注释：定义 Truth QC 主函数，负责 pcap 转 PCD 后按 checkpoint 抽帧
    s3_path_truth: str,  # 注释：参数 s3_path_truth 是 Truth 数据 S3 目录；ADC 路径为空时使用
    data_id: str,  # 注释：参数 data_id 是后端数据唯一标识，贯穿 checkpoint、输出目录和上报
    output_dir: Path,  # 注释：参数 output_dir 是本地 QC 输出目录
) -> list[Path]:
    """
    1. GET checkpoint API -> 已存在则复用，并用 qt_front 精确匹配本地 short_name 
    2. 如果没有 checkpoint，则 list S3 Truth 目录并按 topic 分组 
    3. 用 qt_front 首尾/目标片段的 PTP 计算 checkpoint，并 POST 上报 
    4. 按 checkpoint.short_name 筛选并下载各 topic 对应 pcap
    5. 各 topic 多进程并行；topic 内按 pcap seq 串行：pcap -> PCD -> 抽帧 -> 前方裁剪 -> 删除临时目录  # 不同雷达并行，单雷达内部保持时间顺序
    # 一个 pcap 经 pcl_tool 转换成一批 PCD 文件，通常每个点云帧对应一个 PCD。每个 pcap 会对应一个独立的临时 PCD 目录。不过，这批临时 PCD 不会全部成为最终 QC 输出。后续只会按照 checkpoint 选取匹配的帧，裁剪后写入 QC 输出目录，其余临时 PCD 会随临时目录一起删除。
    """
```

如果服务端已经有 checkpoint，当前代码会额外做 Truth 本地片段匹配：

```python
if checkpoints:  # 注释：如果服务端已经存在 checkpoint，就进入复用分支
    all_keys = _list_s3_dir(s3_path_truth)  # 列举 Truth S3 目录下的文件 key
    all_keys = [k for k in all_keys if Path(k).suffix == ".pcap"]  # 只保留 pcap 文件
    topic_key_map: dict[str, list[str]] = defaultdict(list)  # 注释：准备 topic -> pcap key 列表的映射
    for key in all_keys:  # 注释：遍历每个 pcap key
        topic = _pcap_stem_to_topic(Path(key).stem)  # 注释：根据文件名推断 topic  下面有详解
        if topic:  # 注释：如果 topic 推断成功
            topic_key_map[topic].append(key)  # 注释：把该 key 放入对应 topic 分组
    base_keys = topic_key_map.get(_TRUTH_BASE_TOPIC, [])  # 注释：取 qt_front pcap，作为 Truth 时间和片段匹配基准
    ptp_cache: dict[str, tuple[int, int]] = {}  # 注释：缓存 pcap 的 PTP 范围，避免重复下载和转换
    ptp_cache_lock = threading.Lock()  # 注释：多线程访问缓存时加锁，避免并发写冲突
```

并发匹配 checkpoint 的代码：

```python
with ThreadPoolExecutor(max_workers=min(5, len(checkpoints))) as pool:  # 注释：最多 5 个线程并发匹配 checkpoint
    futures = {  # 注释：准备 future -> checkpoint 的映射
        pool.submit(  # 注释：提交一个 checkpoint 匹配任务
            _match_checkpoint_to_segment,  #按 checkpoint 时间和 short_name 匹配本地 qt_front 片段 在./{x}中有详解
            cp, _normalize_short_name(cp.get("short_name", "")),  # 传入 checkpoint 和规范化后的 short_name
            base_keys, truth_dir, _download_fn,  #传入 qt_front 候选文件、下载目录、PTP 读取函数
        ): cp  #future 完成后能找回对应 checkpoint
        for cp in checkpoints  # 遍历服务端返回的所有 checkpoint
        if _normalize_short_name(cp.get("short_name", ""))  # 只处理带 short_name 的 checkpoint
    }  # future 字典构造结束
    for fut in as_completed(futures):  # 注释：按任务完成顺序处理结果
        cp = futures[fut]  # 注释：取回当前 future 对应的 checkpoint
        local_sn = fut.result()  # 注释：得到匹配到的 Truth 本地 short_name
        if local_sn:  # 注释：如果匹配成功
            cp["short_name"] = local_sn  # 注释：把 checkpoint 的 short_name 改成 Truth 本地片段号
```

### 8.1 Truth topic 从文件名推断

Truth 的 pcap 文件名决定 topic：

```python
def _pcap_stem_to_topic(stem: str) -> str | None:  # 注释：定义从 Truth pcap 文件名推断 ROS topic 的函数
    parts = stem.rsplit("_", 3)  # 注释：从右侧最后一个下划线切开文件名
    if len(parts) >= 4:  # 注释：qt_front、qt_right 等文件名会切出至少 4 段
        lidar_type = f"{parts[0]}_{parts[1]}"   # qt_front / qt_right / ...  # 注释：组合 qt 和方位，得到 lidar_type，例如 qt_front
    elif len(parts) >= 3:  # 注释：pandar 文件名会进入这个分支
        lidar_type = parts[0]                    # pandar  # 注释：pandar 的 lidar_type 就是第一段 pandar
    else:  # 注释：当前条件不满足时进入另一条分支
        return None  # 注释：没有任何切片命中时返回 None
    topic = f"/sensor/lidar_{lidar_type}_pointcloud"  # 注释：按项目 topic 命名规范拼出点云 topic
    if topic == _TRUTH_PANDAR_TOPIC or topic == _TRUTH_BASE_TOPIC or topic.startswith(_TRUTH_QT_PREFIX):  # 注释：只接受 pandar、qt_front 或 qt_* 合法 Truth topic
        return topic  # 注释：返回推断出的 topic
    return None  # 注释：没有任何切片命中时返回 None
```

例子：

| 文件名 | topic |
| --- | --- |
| `pandar_202606160710_003.pcap` | `/sensor/lidar_pandar_pointcloud` |
| `qt_front_202606160710_003.pcap` | `/sensor/lidar_qt_front_pointcloud` |
| `qt_left_202606160710_003.pcap` | `/sensor/lidar_qt_left_pointcloud` |

`qt_front` 是 Truth 分支计算时间范围和 checkpoint 的基准 topic：

```python
_TRUTH_BASE_TOPIC = "/sensor/lidar_qt_front_pointcloud"  # 定义 Truth 时间基准 topic，使用 qt_front 计算或匹配 checkpoint
```

### 8.2 pcap 转 PCD

Truth 用 `pcl_tool` 把 pcap 转成一批 PCD 文件：

```python
def _run_pcap_to_pcd_dir(  # 注释：定义调用 pcl_tool 把单个 pcap 转成 PCD 目录的函数
    pcap_file: Path,  # 注释：参数 pcap_file 是输入 pcap 文件路径
    topic: str,  # 注释：参数 topic 是要处理的 ROS topic 名称
    pcd_dir: Path,  # 注释：参数 pcd_dir 是 pcl_tool 输出 PCD 的临时目录
    idle_timeout: int = 30,  # 注释：pcl_tool 无输出时的超时时间，默认 30 秒
    max_frames: int = 0,  # 最多转换多少帧；0 表示不限制
) -> list[tuple[Path, int]]: 
    from util.pcap_to_mcap import run_pcl_tool  # 导入 pcl_tool 封装函数，用来把 Truth pcap 转成 PCD

    pcd_dir.mkdir(parents=True, exist_ok=True)  # 注释：创建目录；parents=True 会连父目录一起创建，exist_ok=True 表示已存在也不报错
    pcl_lidar_type = "pandar" if "pandar" in topic else "qt"  # 注释：根据 topic 判断 pcl_tool 用 pandar 还是 qt 解析模式

    with tempfile.NamedTemporaryFile(suffix=".ini", delete=False) as tmp:  # 注释：创建临时 ini 配置文件，供 pcl_tool 使用
        cfg_path = tmp.name  # 注释：保存临时配置文件路径
    try:  # 注释：确保无论转换成功失败，最后都会删除临时配置文件
        run_pcl_tool(  # 注释：开始调用 pcl_tool 转换 pcap
            str(pcap_file), str(pcd_dir),  # 注释：传入 pcap 输入路径和 PCD 输出目录
            max_frames=max_frames,  # 注释：限制最多转换的帧数；0 表示不限制
            lidar_type=pcl_lidar_type,  # 注释：告诉 pcl_tool 当前雷达类型
            cfg_path=cfg_path,  # 注释：传入临时配置文件路径
            idle_timeout=idle_timeout,  # 注释：传入无输出超时时间
        ) 
        if not list(pcd_dir.glob("*.pcd")):  # 注释：如果 pcl_tool 没生成任何 PCD，认为转换失败
            raise RuntimeError(  # 注释：主动抛错，避免后续误以为成功
                f"pcl_tool 退出码正常但未生成 PCD 文件: pcap={pcap_file}, pcd_dir={pcd_dir}"  # 注释：错误信息包含输入和输出目录
            )  # 注释：RuntimeError 构造结束
    finally:  # 注释：无论是否异常都执行清理
        Path(cfg_path).unlink(missing_ok=True)  # 注释：删除临时 ini 配置文件

    index: list[tuple[Path, int]] = []  # 注释：准备 PCD 时间索引，元素是 PCD 路径和纳秒时间戳
    for pcd_path in sorted(pcd_dir.glob("*.pcd"), key=lambda p: p.name):  #按文件名遍历 pcl_tool 生成的 PCD
        ts = _read_pcd_first_ts(pcd_path)  # 注释：从 PCD 文件名中读取该帧 PTP 时间 紧接着有详解
        if ts:  # 注释：如果成功解析出时间戳
            index.append((pcd_path, ts))  # 注释：把 PCD 路径和时间戳加入索引
        else: 
            pcd_path.unlink(missing_ok=True)  # 注释：删除无法解析时间戳的无效 PCD 文件
            logger.debug(f"  [pcl_tool] Deleted invalid PCD (no timestamp): {pcd_path.name}")  #记录被删除的无效 PCD
    logger.info(f"  [pcl_tool] {pcap_file.name} → {len(index)} frames (dir={pcd_dir.name})")  # 注释：打印当前 pcap 转出的有效帧数
    if index:  # 注释：如果至少有一帧
        logger.info(f"  [pcl_tool] first frame PTP: {index[0][1]} ns = {index[0][1] / 1e9:.6f} s")  # 打印首帧 PTP 时间
    return index  # 注释：返回 PCD 时间索引
```

返回值 `index` 是：

```text
[
  (PointCloudFrame0_1782571256.225666_bin.pcd, 1782571256225666000),
  (PointCloudFrame1_1782571256.325666_bin.pcd, 1782571256325666000),
]
```

时间戳来自 PCD 文件名：

```python
def _read_pcd_first_ts(pcd_path: Path) -> int | None:  # 注释：定义从 PCD 文件名中解析 PTP 纳秒时间戳的函数
    name = pcd_path.name  # 注释：取 PCD 文件名
    m = _PCD_TS_RE.search(name)  # 注释：用正则匹配文件名里的秒和微秒
    if m:  # 注释：如果文件名匹配成功
        sec = int(m.group(1))  # 注释：提取秒字段并转成整数
        microsec = int(m.group(2))  # 注释：提取微秒字段并转成整数
        return sec * 1_000_000_000 + microsec * 1000  # 注释：把秒和微秒合成纳秒时间戳
    return None  # 注释：没有任何切片命中时返回 None
```

### 8.3 Truth 如何定位 checkpoint 对应的 PCD

Truth 使用二分查找找到 `timestamp_ns >= checkpoint_ns` 的第一帧：

```python
def _find_pcd_for_ts(  # 注释：定义在 PCD 时间索引中查找 checkpoint 后第一帧的函数
    index: list[tuple[Path, int]],  # 注释：准备 PCD 时间索引，元素是 PCD 路径和纳秒时间戳
    ts_ns: int,  # 注释：参数 ts_ns 是 checkpoint 纳秒时间，用来和消息时间比较
    timestamps: list[int] | None = None,  # 注释：给变量赋值，保存后续步骤要使用的中间结果
) -> tuple[Path, int] | None:
    if timestamps is None:  # 注释：如果调用方没有传入纯时间戳列表
        timestamps = [ts for _, ts in index]  # 注释：从 PCD 索引中提取所有时间戳
    pos = bisect_left(timestamps, ts_ns)  # 注释：二分查找第一个大于等于 checkpoint 时间的位置
    if pos < len(index):  # 注释：如果找到的位置没有超过索引长度
        return index[pos]  # 注释：返回命中的 PCD 路径和时间戳
    return None  # 注释：没有任何切片命中时返回 None
```

单 topic 处理流程在 `_process_topic()`：

```python
def _process_topic(  # 定义 Truth 单个 topic 的处理函数，内部按 pcap 顺序转换和抽帧
    topic: str,  # 参数 topic 是要处理的 ROS topic 名称
    pcap_files: list[Path],  # 当前 topic 下按序排列的 pcap 文件列表
    checkpoints: list[dict],  # 注释：待抽取的 checkpoint 列表
    pcd_tmp_dir: Path,  # 注释：pcl_tool 输出 PCD 时使用的临时目录
    output_dir: Path,  # 注释：参数 output_dir 是本地 QC 输出目录
    preconverted: dict[Path, list[tuple[Path, int]]],  # 注释：预转换缓存，避免重复转换同一个 pcap
    idle_timeout: int = 30,  # 注释：pcl_tool 无输出时的超时时间，默认 30 秒
) -> list[Path]:  # 注释：函数签名结束，并声明返回值类型
    topic_safe = _topic_to_filename_prefix(topic)  # 注释：把 topic 转为文件名前缀，例如 /a/b 变成 a_b
    outputs: list[Path] = []  # 注释：给变量赋值，保存后续步骤要使用的中间结果
    pending = list(checkpoints)  # 注释：复制 checkpoint 列表，pending 表示还没匹配到帧的采样点
    cp_start_seq = _expand_checkpoints(checkpoints)  # 计算每个 checkpoint 的输出起始序号 上面有详解

    for pcap_file in pcap_files:  # 注释：按顺序处理当前 topic 下的每个 pcap 文件
        if not pending:  # 注释：如果所有 checkpoint 都已经抽到，就可以提前结束
            break  # 注释：提前结束循环，不再生成更多 checkpoint

        pcd_dir = pcd_tmp_dir / pcap_file.stem  # 注释：为当前 pcap 准备独立的 PCD 临时目录
        if pcap_file in preconverted:  # 如果探测阶段已经转换过这个 pcap
            idx = preconverted[pcap_file]  # 直接复用已有 PCD 时间索引
        else:  # 注释：如果没有预转换结果
            idx = _run_pcap_to_pcd_dir(pcap_file, topic, pcd_dir, idle_timeout=idle_timeout)  # 把当前 pcap 转成 PCD，并得到 PCD 时间索引

        if idx:  # 注释：如果当前 pcap 转出了有效 PCD
            _idx_timestamps = [ts for _, ts in idx]  # 注释：预提取时间戳列表，避免循环里重复构造
        else:  # 注释：如果当前 pcap 没有有效 PCD
            _idx_timestamps = []  # 注释：使用空时间戳列表，后续查找会失败

        still_pending: list[dict] = []  # 保存当前 pcap 仍未命中的 checkpoint，后续 pcap 继续找
        for cp in pending:  # 注释：遍历当前仍未匹配到帧的 checkpoint
            count = _cp_count(cp)  # 注释：读取 checkpoint 的 count，决定连续抽几帧
            seq_no = cp["seq_no"]  # 注释：读取 checkpoint 原始序号，例如 "001"
            start_seq = cp_start_seq.get(seq_no, int(seq_no))  # 注释：取得当前 checkpoint 的输出起始序号
            ts_ns = int(cp["ts"]) * 1_000_000  # 注释：把 checkpoint 的毫秒时间转换为纳秒，才能和 MCAP log_time 或 PCD 时间戳比较

            matched_frames: list[tuple[Path, int]] = []  # 注释：准备保存当前 checkpoint 找到的连续 PCD 帧
            search_ts = ts_ns  # 注释：第一次从 checkpoint 时间开始查找
            for _ in range(count):  # 注释：最多查找 count 帧
                result = _find_pcd_for_ts(idx, search_ts, _idx_timestamps)  # 注释：在 PCD 时间索引中找第一帧时间不早于 search_ts 的帧
                if result is not None:  # 注释：如果找到了符合条件的 PCD 帧
                    pcd_path, frame_ts = result  # 注释：拆出 PCD 文件路径和该帧时间戳
                    matched_frames.append(result)  # 注释：把命中的 PCD 帧加入当前 checkpoint 的结果
                    search_ts = frame_ts + 1  # 注释：下一次从当前帧之后开始找，避免重复命中
                else:  # 注释：当前条件不满足时进入另一条分支
                    break  # 注释：提前结束循环，不再生成更多 checkpoint

            if matched_frames:  # 注释：如果这一 checkpoint 找到至少一帧
                for frame_offset, (pcd_path, frame_ts) in enumerate(matched_frames):  # 注释：遍历匹配到的连续 PCD 帧
                    seq_idx = start_seq + frame_offset  # 注释：按 checkpoint 起始序号和帧偏移计算最终 seq
                    out_name = f"{topic_safe}_{frame_ts}_{seq_idx:03d}.pcd"  # 注释：生成 PCD 输出文件名
                    out_path = output_dir / out_name  # 注释：拼出本地输出文件完整路径
                    out_path.parent.mkdir(parents=True, exist_ok=True)  # 注释：确保输出目录存在
                    ok = filter_pcd_file_front_x_range(pcd_path, out_path)  # 注释：把源 PCD 裁剪到前方 0~80m 后写入输出路径
                    if not ok:  # 注释：如果裁剪失败
                        logger.warning(f"    [{topic}] seq={seq_idx:03d}: front-x filter failed, skip output")  # 注释：记录失败并跳过该输出
                        continue  # 注释：不写 .topic、不加入 outputs，避免无效文件上传
                    out_path.with_suffix(".topic").write_text(topic, encoding="utf-8")  #写同名 .topic 文件保存原始 topic，便于上报
                    outputs.append(out_path)  # 注释：把输出文件路径加入列表，后面统一上传
                    logger.info(f"    [{topic}] seq={seq_idx:03d} → {out_name}")  # 注释：打印成功输出日志
            else:  # 注释：当前条件不满足时进入另一条分支
                still_pending.append(cp)  # 注释：保存当前 pcap 仍未命中的 checkpoint，后续 pcap 继续找
        pending = still_pending  # 注释：复制 checkpoint 列表，pending 表示还没匹配到帧的采样点

        _cleanup_dir(pcd_dir)  # 注释：删除当前 pcap 的 PCD 临时目录，释放磁盘 下面有详解

    return outputs  # 注释：把结果返回给调用方
```

```python
def _cleanup_dir(directory: Path) -> None:
    if directory.exists():
        shutil.rmtree(directory)
        logger.info(f"Cleaned up: {directory}")
```

这段代码有几个关键点：

1. 某个 topic 内按 pcap 顺序处理，处理完一个 pcap 就删掉对应 PCD 临时目录。
2. 如果所有 checkpoint 都匹配到了，就提前退出，不再转换后面的 pcap。
3. 如果探测阶段已经转换过首尾 pcap，会通过 `preconverted` 复用索引。
4. 输出前会调用 `filter_pcd_file_front_x_range()`，只保留前方 `0 <= x <= 80m` 的点。

Truth 分支最后用多进程并行处理多个 topic：

```python
with ProcessPoolExecutor(max_workers=len(topics)) as pool:  # 创建多进程池，让不同 topic 并行处理
    futures = {  # 创建 future 到 topic 的映射，便于任务完成后定位 topic
        pool.submit(  # 注释：向进程池提交一个 topic 处理任务
            _process_topic,  # 子进程要执行的函数是 _process_topic 上面有代码
            topic,  # 传入当前 topic
            topic_pcaps[topic],  # 传入当前 topic 对应的 pcap 文件列表 上面有代码
            checkpoints,  # 传入全局 checkpoint 列表
            pcd_tmp_dir,  # 传入 PCD 临时目录
            output_dir,  # 传入 QC 输出目录
            preconverted,  # 传入预转换缓存，避免重复运行 pcl_tool
            idle_timeout=30,  # 传入无输出超时时间
        ): topic  # 把 future 映射回 topic 名称
        for topic in topics  # 注释：为每个 topic 都提交一个并行任务
    }
    for fut in as_completed(futures):  # 按任务完成顺序收集结果
        topic = futures[fut]  # 取回当前 future 对应的 topic
        try:  # 捕获单个 topic 失败，避免影响其他 topic
            outputs.extend(fut.result())  # 注释：把该 topic 输出的 PCD 路径追加到总输出列表
        except Exception as e:  # 如果子进程处理失败
            logger.error(f"  [QC-Truth] topic {topic} failed: {e}")  # 注释：记录失败 topic 和异常
```

## 9. 输出文件命名规则

QC 输出文件名统一是：

```text
<topic_safe>_<log_time_or_frame_ts_ns>_<seq:03d>.<ext>
```

例子：

```text
sensor_camera_front_image_1781187318123456789_001.jpeg
sensor_lidar_front_pointcloud_1781187318223456789_001.pcd
calib_calib_param_1781187000000000000_001.json
sensor_lidar_qt_front_pointcloud_1781187318333333000_001.pcd
```

`topic_safe` 来自：

```python
def _topic_to_filename_prefix(topic: str) -> str:  # 注释：定义把 ROS topic 转成安全文件名前缀的函数
    return topic.strip("/").replace("/", "_")  # 注释：去掉首尾斜杠，再把路径斜杠替换成下划线
```

同时每个输出文件旁边会写一个 `.topic` sidecar 文件，例如：

```text
sensor_camera_front_image_1781187318123456789_001.jpeg
sensor_camera_front_image_1781187318123456789_001.topic
```

`.topic` 文件内容是原始 topic：

```text
/sensor/camera_front_image
```

这样后面上报接口能拿到准确 topic。

## 10. 上传 S3 和上报后端

所有分支输出都会回到 `pipeline_qc.py` 的 `_upload_qc_files_and_report()`：

```python
def _upload_qc_files_and_report(  # 定义上传 QC 文件并调用后端 batch-save 接口的函数
        all_outputs: list[Path],  # 注释：准备总输出列表，收集 ADC 或 Truth 分支生成的所有文件
        data_id: str,  # 注释：参数 data_id 是后端数据唯一标识，贯穿 checkpoint、输出目录和上报
        data_name: str,  # 注释：数据名称，用于上报后端
        data_date,  # 注释：本次任务日期，用于生成 S3 日期路径
) -> bool:  # 注释：函数签名结束，返回布尔值表示成功或失败
    uploader = get_default_uploader()  # 注释：创建默认 S3 上传客户端
    client = get_default_client()  # 注释：创建数据管理后端 API 客户端

    yyyy_mm_dd = data_date.strftime("%Y/%m/%d")  # 注释：把任务日期格式化成 yyyy/mm/dd，用于 S3 key
    file_list: list[dict] = []  # 注释：准备后端 batch-save 要提交的文件记录列表

    for file_path in all_outputs:  # 注释：遍历每个本地 QC 输出文件
        filename = file_path.name  # 注释：只取文件名，不包含目录

        sidecar = file_path.with_suffix(".topic")  # 注释：找到同名 .topic sidecar 文件路径
        if sidecar.exists():  # 注释：如果 sidecar 存在，就用它读取真实 topic
            topic = sidecar.read_text(encoding="utf-8").strip()  # 注释：读取原始 topic 并去掉首尾空白
            parts = filename.rsplit("_", 2)  # 注释：从文件名右侧切分出 topic 前缀、时间戳和序号
            ts_ns = parts[1]  # 注释：把 checkpoint 的毫秒时间转换为纳秒，才能和 MCAP log_time 或 PCD 时间戳比较
            seq_str = parts[2].rsplit(".", 1)[0]  # 注释：取文件名中的序号，并去掉扩展名

        s3_key = f"qc/{yyyy_mm_dd}/{data_id}/{filename}"  # 注释：拼出 QC 文件在 S3 中的 key

        uploader.s3_client.upload_file(  # 注释：调用 S3 SDK 上传文件
            str(file_path),  # 注释：本地待上传文件路径
            uploader.bucket,  # 注释：目标 S3 bucket 名称
            s3_key,  # 注释：目标 S3 key
        )  # 注释：结束上方的多行结构

        file_list.append({  # 注释：准备后端 batch-save 要提交的文件记录列表
            "topic": topic,  # 注释：记录当前文件对应的原始 topic
            "storagePath": s3_key,  # 注释：记录文件在 S3 中的存储路径
            "ts": ts_ns,  # 注释：checkpoint 时间，单位毫秒，后续抽帧时会再转纳秒
            "seq": int(seq_str),  # 注释：记录抽帧序号，转为整数
        })  # 注释：结束字典并关闭函数调用

    payload = {  # 注释：开始组装请求体 payload
        "dataId": data_id,  # 注释：请求体中携带当前数据 ID
        "dataName": data_name or data_id,  # 注释：记录数据名称；没有解析出 dataName 时用 data_id 兜底
        "dataSource": 1,  # 注释：记录数据来源，当前代码固定传 1
        "fileList": file_list,  # 注释：把所有文件记录放入请求体
    }  # 注释：结束上方的多行结构

    ok = client.batch_save_qc_file(payload)  # 注释：调用后端接口批量保存 QC 文件记录
    return ok  # 注释：返回后端上报是否成功
```

上报接口是：

```python
def batch_save_qc_file(self, payload: dict) -> bool:  # 注释：定义批量保存 QC 文件记录的 API 方法
    url = f"{self.base_url}/qc-file/batch-save"  # 注释：拼出 QC 文件记录批量保存接口地址
    resp = requests.post(url, json=payload, timeout=_TIMEOUT)  # 注释：发送 POST 请求保存 checkpoint
    ...  # 注释：省略此处代码，文档只展示关键流程
```

最终上报 payload 形态大致是：

```json
{
  "dataId": "20567100000303",
  "dataName": "20260630_194626_019f185a-1e1e-7725-9988-33830cc20d75",
  "dataSource": 1,
  "fileList": [
    {
      "topic": "/sensor/camera_front_image",
      "storagePath": "qc/2026/07/03/20567100000303/sensor_camera_front_image_1781187318123456789_001.jpeg",
      "ts": "1781187318123456789",
      "seq": 1
    }
  ]
}
```

注意：代码注释里曾写过 `storagePath` 格式类似 `dms/data/...`，但当前实际代码传的是 `qc/yyyy/mm/dd/<data_id>/<filename>`。

## 11. 从零串起来：一次 ADC QC 的完整执行路径

以 ADC 数据为例，完整路径是：

```text
用户/任务调用 pipeline_qc.py
  -> run_pipeline_qc(data_id, s3_path_adc, None)
    -> 创建输出目录 workspace/<data_id>/qc/yyyy/mm/dd/adc
    -> run_qc_adc(s3_path_adc, data_id, adc_out_dir)
      -> GET /qc/data/{dataId} 查询 checkpoint
      -> 如果没有 checkpoint:
         -> list S3 ADC 目录
         -> 分类 pointcloud / struct / video 文件
         -> 下载首尾 struct 读 PTP 时间范围
         -> 下载 video 片段读取真实 PTP
         -> _compute_checkpoints_from_segments() 计算 checkpoint
         -> POST /qc/checkpoint/update 保存 checkpoint
      -> 根据 checkpoint.short_name 下载需要的 pointcloud/struct/video 分片
      -> 从 video 分片抽 camera JPEG
      -> 从 pointcloud 分片抽 lidar_front PCD
      -> 从 struct 分片抽 calib/chassis JSON
    -> _upload_qc_files_and_report()
      -> 上传到 s3://bucket/qc/yyyy/mm/dd/<data_id>/
      -> POST /qc-file/batch-save 上报文件列表
```

## 12. 从零串起来：一次 Truth QC 的完整执行路径

Truth 数据路径是：

```text
用户/任务调用 pipeline_qc.py --truth_flag true
  -> run_pipeline_qc(data_id, None, s3_path_truth)
    -> 创建输出目录 workspace/<data_id>/qc/yyyy/mm/dd/truth
    -> run_qc_truth(s3_path_truth, data_id, truth_out_dir)
      -> GET /qc/data/{dataId} 查询 checkpoint
      -> 如果已有 checkpoint:
         -> list S3 Truth 目录
         -> 只保留 .pcap
         -> 按文件名推断 topic 并分组
         -> 取 qt_front pcap 作为匹配基准
         -> 多线程调用 _match_checkpoint_to_segment()
         -> 下载候选 qt_front pcap
         -> pcl_tool 转 PCD 并读取 PTP 范围
         -> 把 checkpoint.short_name 修正为 Truth 本地片段号
         -> 预转换首尾 qt_front pcap，缓存 PCD 索引
      -> 如果没有 checkpoint:
         -> list S3 Truth 目录
         -> 只保留 .pcap
         -> 按文件名推断 topic 并分组
         -> 取 qt_front pcap 作为时间基准
         -> 下载首尾 qt_front pcap
         -> 首文件只取首帧，得到 t_start
         -> 如果 qt_front 文件数 > 2:
            -> last 文件也只取首帧，近似得到 t_end
            -> 并发下载目标 qt_front 片段
            -> 每个目标片段 max_frames=1 读取真实首帧 PTP
            -> 构造 seg_ptp_map，减少 checkpoint 线性估算误差
         -> 如果 qt_front 文件数 <= 2:
            -> last 文件完整解析，得到更准确 t_end
         -> _compute_checkpoints_from_segments() 计算 checkpoint
         -> POST /qc/checkpoint/update 保存 checkpoint
         -> 如果 POST 失败:
            -> 再 GET 服务端 checkpoint
            -> GET 也失败则使用本地计算 checkpoint
      -> 根据 checkpoint.short_name 下载需要的 pcap
      -> 按 topic 分组
      -> 多进程处理每个 topic:
         -> topic 内按 pcap seq 串行处理
         -> 如 pcap 在 preconverted 中，则复用 PCD 索引
         -> 否则调用 pcl_tool 把 pcap 转 PCD
         -> 从 PCD 文件名读时间戳
         -> 二分查找 checkpoint 后的连续帧
         -> 调用 filter_pcd_file_front_x_range() 做前方 0~80m 裁剪
         -> 写为 QC 输出 PCD
         -> 写同名 .topic sidecar
         -> 删除临时 PCD 目录
      -> 删除 pcd_tmp_dir
    -> _upload_qc_files_and_report()
      -> 上传 QC PCD
      -> POST /qc-file/batch-save 上报文件列表
```

对应的下载与分组核心代码：

```python
short_names = {_normalize_short_name(cp["short_name"]) for cp in checkpoints if "short_name" in cp}  # 注释：从 checkpoint 中取出需要下载的片段号
if not short_names:  # 注释：如果 checkpoint 没有 short_name
    _download_directory(s3_path_truth, truth_dir)  # 注释：兜底下载整个 Truth 目录
else:  # 注释：如果 checkpoint 有 short_name
    if not all_keys:  # 注释：如果前面还没有列举过 S3 文件
        all_keys = _list_s3_dir(s3_path_truth)  # 注释：列举 Truth S3 目录
        all_keys = [k for k in all_keys if Path(k).suffix == ".pcap"]  # 注释：只保留 pcap 文件
    need_keys = [k for k in all_keys if _match_by_short_name(k, short_names)]  # 注释：只筛选 checkpoint 命中的片段
    _download_files_by_keys(need_keys, truth_dir)  # 注释：下载命中的 pcap 文件
```

对应的 topic 分组代码：

```python
pcap_files = sorted(truth_dir.glob("*.pcap"))  # 注释：读取本地已下载的 pcap 文件
topic_pcaps: dict[str, list[Path]] = defaultdict(list)  # 注释：准备 topic -> pcap 文件列表
for pcap_file in pcap_files:  # 注释：遍历每个本地 pcap
    topic = _pcap_stem_to_topic(pcap_file.stem)  # 注释：根据文件名推断 topic
    if topic:  # 注释：如果 topic 合法
        topic_pcaps[topic].append(pcap_file)  # 注释：加入对应 topic 分组
for topic in topic_pcaps:  # 注释：遍历每个 topic
    topic_pcaps[topic].sort(key=_pcap_seq)  # 注释：按文件末尾 seq 排序，保证时间顺序处理
```

对应的并行处理代码：

```python
with ProcessPoolExecutor(max_workers=len(topics)) as pool:  # 注释：创建进程池，每个 topic 一个并行任务
    futures = {  # 注释：准备 future -> topic 映射
        pool.submit(  # 注释：提交单 topic 处理任务
            _process_topic,  # 注释：子进程执行 _process_topic
            topic,  # 注释：传入当前 topic
            topic_pcaps[topic],  # 注释：传入该 topic 的 pcap 文件列表
            checkpoints,  # 注释：传入 checkpoint 列表
            pcd_tmp_dir,  # 注释：传入 PCD 临时目录
            output_dir,  # 注释：传入 QC 输出目录
            preconverted,  # 注释：传入预转换缓存
            idle_timeout=30,  # 注释：传入 pcl_tool 无输出超时时间
        ): topic  # 注释：把 future 映射回 topic
        for topic in topics  # 注释：每个 topic 提交一个任务
    }  # 注释：future 字典结束
    for fut in as_completed(futures):  # 注释：按完成顺序收集结果
        topic = futures[fut]  # 注释：取回当前 topic
        outputs.extend(fut.result())  # 注释：把该 topic 的输出 PCD 路径合并到总输出
```

## 13. 新手最容易混淆的点

### 13.1 `ts` 是毫秒，`log_time` 是纳秒

checkpoint 里的 `ts` 单位是毫秒：

```json
{"ts": "1781187318000"}
```

MCAP message 的 `log_time` 是纳秒：

```python
message.log_time  # 注释：MCAP 消息的真实纳秒时间，抽帧输出文件名和上报 ts 都依赖它
```

所以抽帧时一定会看到：

```python
ts_ns = int(cp["ts"]) * 1_000_000  # 注释：把 checkpoint 的毫秒时间转换为纳秒，才能和 MCAP log_time 或 PCD 时间戳比较
```

### 13.2 `seq_no` 和输出文件名最后的序号不是永远相等

因为 checkpoint 有 `count`。如果有：

```json
[
  {"seq_no": "001", "count": 3},
  {"seq_no": "002", "count": 3}
]
```

`_expand_checkpoints()` 会算出：

```python
{"001": 1, "002": 4}  # 注释：示例映射：001 从输出序号 1 开始，002 从输出序号 4 开始
```

所以第一个 checkpoint 输出 `001/002/003`，第二个 checkpoint 输出 `004/005/006`。

代码：

```python
def _expand_checkpoints(checkpoints: list[dict]) -> dict[str, int]:  # 注释：定义把 checkpoint 展开成连续输出序号映射的函数
    result: dict[str, int] = {}  # 注释：准备 seq_no 到输出起始序号的映射
    running_seq = 1  # 注释：输出文件序号从 1 开始计数
    for cp in checkpoints:  # 注释：遍历每个 checkpoint 进行点云抽帧
        result[cp["seq_no"]] = running_seq  # 注释：记录当前 checkpoint 对应的输出起始序号
        running_seq += _cp_count(cp)  # 注释：输出文件序号从 1 开始计数
    return result  # 注释：把结果返回给调用方
```

### 13.3 `short_name` 是为了少下载文件

checkpoint 里带 `short_name` 后，只需要下载对应片段，节省 S3 下载和本地磁盘。

### 13.4 `.topic` 文件不是多余文件

输出文件名里的 topic 已经被替换为安全文件名前缀，不能 100% 还原原始 topic。`.topic` 文件用于上报时读取真实 topic。

### 13.5 ADC 相机需要 ffmpeg

相机原始数据是 H265，需要 ffmpeg 解码成 JPEG。如果机器没有 ffmpeg，`_decode_h265_stream_to_jpegs()` 会报错：

```python
if not shutil.which("ffmpeg"):  # 注释：调用 ffmpeg，-y 表示允许覆盖临时输出文件
    raise RuntimeError("ffmpeg not found in PATH; required for H265 decoding.")  # 注释：没有 ffmpeg 就报错，因为 H265 相机帧无法解码
```

### 13.6 Truth 依赖 pcl_tool

Truth 的 pcap 需要通过 `data_parse/lib/pcl_tool` 转成 PCD。没有这个工具或配置不正确，Truth 抽帧会失败。

## 14. 排查问题时看哪些日志

常见日志关键字：

| 日志关键字 | 说明 |
| --- | --- |
| `[QC] data_id` | 总入口开始运行 |
| `[QC-ADC] Listing ADC directory on S3` | ADC 开始列举 S3 |
| `[QC-Truth] Listing Truth directory on S3` | Truth 开始列举 S3 |
| `checkpoint exists on server` | 服务端已有 checkpoint |
| `checkpoint uploaded` | 本地计算的 checkpoint 已写入服务端 |
| `short_names to download` | 将按这些片段号下载文件 |
| `no slice found` | checkpoint 时间没有落在任何切片范围内 |
| `no frame` | 找到了切片，但该 topic 在 checkpoint 后没有消息 |
| `ffmpeg not found` | ADC 相机抽帧环境缺 ffmpeg |
| `pcl_tool` | Truth pcap 转 PCD 相关日志 |
| `[QC-Report] Uploaded` | 文件上传 S3 成功 |
| `batch-save QC file` | QC 文件记录上报接口结果 |

## 15. 修改代码时的建议阅读顺序

如果你要改 QC 抽帧，建议按下面顺序读：

1. `pipeline_qc.py`：先理解入口、分支选择、上传上报。
2. `qc_utils.py`：理解 checkpoint、时间单位、切片索引、PCD 写出。
3. `pipeline_qc_adc.py`：理解 ADC 三类输出：JPEG、PCD、JSON。
4. `pipeline_qc_truth.py`：理解 pcap 转 PCD、二分定位、并行处理。
5. `common/api_client.py`：确认后端接口路径和 payload。

只要抓住这条主线，整套 QC 抽帧就不复杂：

```text
S3 数据 -> checkpoint -> 定位分片 -> 找 checkpoint 后第一帧/连续帧 -> 写本地输出 -> 上传 S3 -> 上报接口
```
