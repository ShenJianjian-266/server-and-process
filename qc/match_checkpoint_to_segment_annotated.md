# _match_checkpoint_to_segment 注释版

## 方法作用

`_match_checkpoint_to_segment` 的作用：

```text
把一个 checkpoint 匹配到当前分支真实存在的文件片段 short_name。
```

为什么需要它：

```text
ADC 和 Truth 的文件片段编号可能不完全一致。
服务端 checkpoint 可能来自另一条分支，所以要根据 checkpoint 的 ts，
在当前分支候选文件里找到真正覆盖这个时间的片段。
```

尝试顺序：

```text
candidate -> candidate±1 -> candidate±2
```

例如候选 `003`：

```text
先试 003
再试 002、004
最后兜底试 001、005
```

## 主方法：_match_checkpoint_to_segment

```python
def _match_checkpoint_to_segment(  # 定义函数：把 checkpoint 匹配到当前分支真实文件片段
    checkpoint: dict,  # 参数：checkpoint 字典，至少包含 seq_no 和 ts
    short_name_candidate: str,  # 参数：候选 short_name，可能来自另一分支
    all_keys: list[str],  # 参数：当前分支所有 S3 文件 key
    local_dir: Path,  # 参数：文件下载到本地的目录
    download_and_read_ptp,  # 参数：下载文件并读取 PTP 范围的回调函数
    offset: int = 0,  # 参数：当前递归偏移层级，0 表示先试原候选
) -> str | None:  # 返回值：匹配到的 short_name；失败返回 None
    """对单个 checkpoint 尝试匹配本地文件段。  # 方法说明：根据 checkpoint 时间匹配文件片段

    尝试顺序：candidate → candidate±1 → candidate±2（±2 作为边界兜底）  # 匹配策略说明

    Args:  # 参数说明开始
        checkpoint:        checkpoint dict，含 seq_no / ts 
        short_name_candidate: 候选 short_name（来自另一分支）
        all_keys:         所有本地 S3 key 列表
        local_dir:        下载目标目录（供 download_and_read_ptp 内部使用）
        download_and_read_ptp: 下载并读 PTP 范围的函数，签名 (key, local_dir) -> (start_ns, end_ns) | None
        offset:           当前尝试的偏移（内部递归用

    Returns:
        匹配到的本地 short_name，或 None  # 成功返回片段号，失败返回 None
    """  # 文档字符串结束
    if offset > 2:  # 如果偏移已经超过 2
        return None  # 超出兜底范围，返回 None

    ts_ns = int(checkpoint["ts"]) * 1_000_000  # checkpoint 的 ts 是毫秒，这里转成纳秒

    for delta in (0, -1, 1) if offset == 0 else (1, -1) if offset == 1 else (-1, 1):  # 根据 offset 决定本轮尝试哪些偏移
        candidate_num = int(_normalize_short_name(short_name_candidate)) + delta  # 把候选 short_name 转数字并加偏移
        candidate_sn = f"{candidate_num:03d}"  # 把数字格式化成三位字符串，例如 3 -> "003"
        # 在 all_keys 中找匹配的 key  # 注释：从当前分支文件列表中找对应片段
        matched_key = None  # 初始化匹配到的 S3 key
        for key in all_keys:  # 遍历当前分支全部候选文件 key
            if _extract_short_name(Path(key)) == candidate_sn:  # 如果该文件的 short_name 等于候选片段号
                matched_key = key  # 记录这个文件 key
                break  # 找到后退出循环
        if not matched_key:  # 如果当前候选片段号没有对应文件
            continue  # 换下一个 delta 继续尝试

        ptp_range = download_and_read_ptp(matched_key, local_dir)  # 下载该文件并读取它的 PTP 时间范围
        if ptp_range is None:  # 如果读取 PTP 失败
            continue  # 换下一个候选继续尝试
        start_ns, end_ns = ptp_range  # 解包 PTP 范围：开始纳秒和结束纳秒

        if start_ns <= ts_ns <= end_ns:  # 如果 checkpoint 时间落在该文件时间范围内
            logger.info(  # 打印匹配成功日志
                f"  [match] seq={checkpoint['seq_no']} ts={ts_ns}: "  # 日志前半段：checkpoint 序号和纳秒时间
                f"matched {candidate_sn} (delta={delta}, range=[{start_ns},{end_ns}])"  # 日志后半段：匹配片段、偏移和范围
            )  # logger.info 调用结束
            return candidate_sn  # 返回匹配到的 short_name
        elif ts_ns > end_ns and offset < 2:  # 如果 checkpoint 时间晚于当前文件结束时间，且还能继续兜底
            # ts 超出当前段，尝试下一段  # 注释：当前段太早，递归试相邻段
            return _match_checkpoint_to_segment(  # 递归调用本函数
                checkpoint, candidate_sn, all_keys, local_dir,  # 传入 checkpoint、当前候选片段、全部 key、本地目录
                download_and_read_ptp, offset=offset + 1  # 传入同一个 PTP 读取回调，并把 offset 加 1
            )  # 递归调用结束
        elif ts_ns < start_ns and offset < 2:  # 如果 checkpoint 时间早于当前文件开始时间，且还能继续兜底
            # ts 早于当前段，尝试上一段  # 注释：当前段太晚，递归试相邻段
            return _match_checkpoint_to_segment(  # 递归调用本函数
                checkpoint, candidate_sn, all_keys, local_dir,  # 传入 checkpoint、当前候选片段、全部 key、本地目录
                download_and_read_ptp, offset=offset + 1  # 传入同一个 PTP 读取回调，并把 offset 加 1
            )  # 递归调用结束

    return None  # 所有候选都没有匹配成功，返回 None
```

## 核心变量解释

```text
checkpoint["ts"]
```

checkpoint 时间，单位是毫秒。

```text
ts_ns = int(checkpoint["ts"]) * 1_000_000
```

转成纳秒，因为文件 PTP 范围是纳秒。

```text
short_name_candidate
```

候选片段号，可能是：

```text
003
202606240228_003
```

会通过 `_normalize_short_name` 统一成：

```text
003
```

## 被调用方法：_extract_short_name

```python
def _extract_short_name(path: Path) -> str:  # 定义函数：从文件路径中提取片段 short_name
    """从文件名提取 short_name（纯序号部分）。  # 函数说明：统一提取文件末尾的片段号

    S3 上的文件名可能带 _0 后缀，本函数统一处理两种形式：  # 说明：兼容 S3 key 和本地文件名
        struct_202606160710_003.mcap      → 003  # 示例：struct 文件
        pandar_202606160710_003.pcap      → 003  # 示例：pandar 文件
        qt_front_202606160710_003.pcap    → 003  # 示例：qt_front 文件
        video_202606160710_003.mcap       → 003  # 示例：video 文件
        pointcloud_202606160710_003_0.mcap → 003  (S3 上带 _0)  # 示例：pointcloud S3 文件
    """  # 文档字符串结束
    stem = path.stem  # 取文件名去掉扩展名后的部分
    # 去掉 S3 命名的 _0 后缀（下载到本地时 download_dir 已去掉，但 S3 key 仍带）  # 注释：兼容 pointcloud_..._003_0
    if stem.endswith("_0"):  # 如果文件名主体以 _0 结尾
        stem = stem[:-2]  # 去掉末尾两个字符 "_0"
    parts = stem.rsplit("_", 1)  # 从右侧按最后一个下划线切成两段
    if len(parts) == 2 and parts[-1].isdigit():  # 如果切出了两段，且最后一段全是数字
        return parts[-1]  # 返回最后一段作为 short_name
    return stem  # 如果无法提取数字片段，就返回整个 stem
```

## 被调用方法：_extract_seq_suffix / _normalize_short_name

`_normalize_short_name` 是 `_extract_seq_suffix` 的别名：

```python
def _extract_seq_suffix(short_name: str) -> str:  # 定义函数：从 short_name 中提取纯序号
    """从 short_name 提取序号后缀（如下划线后的数字部分）。  # 函数说明：兼容新旧 short_name

    新格式: "012" → "012"  # 示例：新格式本来就是纯序号
    旧格式: "202606240228_012" → "012"  # 示例：旧格式需要取最后的 012
    """  # 文档字符串结束
    parts = short_name.rsplit("_", 1)  # 从右侧按最后一个下划线切分
    if len(parts) == 2 and parts[-1].isdigit():  # 如果最后一段是数字
        return parts[-1]  # 返回最后一段数字
    # 已经是纯序号或其他格式，直接返回  # 注释：例如 "012" 会直接返回
    return short_name  # 返回原始 short_name

_normalize_short_name = _extract_seq_suffix  # 给函数起别名，外部统一用 _normalize_short_name
```

## 被调用方法：download_and_read_ptp 回调

`_match_checkpoint_to_segment` 本身不关心文件是 ADC 的 `struct_*.mcap`，还是 Truth 的 `qt_front_*.pcap`。

它只要求调用方传入一个函数：

```text
(key, local_dir) -> (start_ns, end_ns) | None
```

### ADC 回调：_download_and_read_struct_ptp

```python
def _download_and_read_struct_ptp(key: str, local_dir: Path):  # 定义函数：下载 ADC struct 文件并读取 PTP 范围
    """下载 struct mcap 文件并读取 PTP 时间范围（MCAP summary 秒回）。"""  # 函数说明：用于 ADC 分支匹配 short_name
    paths = _download_files_by_keys([key], local_dir)  # 按 S3 key 下载指定 struct 文件
    if not paths:  # 如果下载失败或没有返回路径
        return None  # 返回 None 表示无法读取 PTP
    return get_ptp_time_range_from_mcap(paths[0])  # 从下载到的 MCAP 文件读取 PTP 起止时间
```

ADC 调用方式：

```python
local_sn = _match_checkpoint_to_segment(  # 调用跨分支片段匹配函数
    cp, sn, _s3_struct_keys, adc_dir, _download_and_read_struct_ptp  # 传入 checkpoint、候选序号、struct key 列表、下载目录、PTP 读取函数
)  # 调用结束，返回匹配到的 ADC 本地 short_name
```

### Truth 回调：_download_and_read_qtfront_ptp

```python
def _download_and_read_qtfront_ptp(  # 定义函数：下载 qt_front pcap 并读取 PTP 范围
    key: str,  # 参数：S3 key
    local_dir: Path,  # 参数：下载目录
    max_frames: int = 0,  # 参数：最多转换帧数，0 表示不限制
    idle_timeout: int = 30,  # 参数：pcl_tool 无新输出时的超时时间
) -> tuple[int, int] | None:  # 返回值：PTP 起止纳秒，失败返回 None
    """下载 qt_front pcap 文件，通过 pcl_tool 转 PCD 并读取 PTP 时间范围。"""  # 函数说明：Truth pcap 需要先转 PCD 才能拿时间
    pcap_paths = _download_files_by_keys([key], local_dir)  # 下载指定 pcap 文件
    if not pcap_paths:  # 如果下载失败
        return None  # 返回 None
    pcap_path = pcap_paths[0]  # 取下载到的本地 pcap 路径

    pcd_tmp = local_dir / f"ptp_{pcap_path.stem}"  # 为本次 PTP 探测准备临时 PCD 目录
    pcd_tmp.mkdir(parents=True, exist_ok=True)  # 创建临时 PCD 目录

    try:  # 确保最后清理临时目录
        idx = _run_pcap_to_pcd_dir(  # 调用 pcl_tool 把 pcap 转成 PCD，并返回时间索引
            pcap_path, _TRUTH_BASE_TOPIC, pcd_tmp,  # 传入 pcap、qt_front topic、临时目录
            idle_timeout=idle_timeout,  # 传入无输出超时时间
            max_frames=max_frames,  # 传入最大转换帧数
        )  # _run_pcap_to_pcd_dir 调用结束
        if idx:  # 如果成功生成 PCD 时间索引
            return idx[0][1], idx[-1][1]  # 返回首帧和末帧的 PTP 时间
    finally:  # 无论成功失败都清理
        _cleanup_dir(pcd_tmp)  # 删除临时 PCD 目录
    return None  # 没有得到有效索引，返回 None
```

Truth 里还包了一层缓存：

```python
def _download_and_read_qtfront_ptp_cached(key: str, _ldir: Path) -> tuple[int, int] | None:  # 定义带缓存的 PTP 读取函数
    """带缓存的 PTP 读取：相同 key 只下载+转换一次。"""  # 函数说明：避免重复转换同一个 pcap
    with ptp_cache_lock:  # 加锁访问缓存
        if key in ptp_cache:  # 如果这个 key 已经读过 PTP
            logger.debug(f"  [match] cache hit: {Path(key).name}")  # 打印缓存命中日志
            return ptp_cache[key]  # 直接返回缓存结果
    result = _download_and_read_qtfront_ptp(key, truth_dir)  # 缓存未命中，真正下载并读取 PTP
    if result is not None:  # 如果读取成功
        with ptp_cache_lock:  # 加锁写缓存
            ptp_cache[key] = result  # 保存该 key 的 PTP 范围
    return result  # 返回读取结果
```

## 被调用方法：_download_files_by_keys

```python
def _download_files_by_keys(keys: list[str], local_dir: Path) -> list[Path]:  # 定义函数：按 S3 key 下载文件
    """按 key 列表从 S3 下载指定文件。"""  # 函数说明：只下载指定 key，不下载整个目录
    from common.s3_utils import get_default_uploader  # 导入默认 S3 上传/下载工具
    uploader = get_default_uploader()  # 创建默认 uploader
    return uploader.download_files_by_keys(keys, local_dir)  # 调用 uploader 下载文件并返回本地路径列表
```

## 整体执行例子

假设：

```python
checkpoint = {"seq_no": "001", "ts": "1781187318500", "short_name": "003"}
short_name_candidate = "003"
```

执行逻辑：

```text
1. ts 从毫秒转纳秒：1781187318500000000
2. 先找 short_name=003 的文件
3. 下载该文件并读取 PTP 范围
4. 如果 003 覆盖 ts，返回 "003"
5. 如果 ts 晚于 003，则递归试 004，再试 005
6. 如果 ts 早于 003，则递归试 002，再试 001
7. 全部失败则返回 None
```

## 一句话总结

```text
_match_checkpoint_to_segment 用 checkpoint 时间戳校验候选片段是否真的覆盖该时间；如果不覆盖，就在候选片段前后最多 2 个片段内递归兜底查找。
```
