# _extract_json_topic 注释版

## 方法作用

`_extract_json_topic` 的作用：

```text
从 ADC 的 struct_*.mcap 切片里，按 checkpoint 抽取指定 topic 的消息，
把消息反序列化成 JSON 文件，并返回写出的 JSON 路径列表。
```

它主要处理：

```python
JsonTopicDef(topic="/calib/calib_param", label="calib")
JsonTopicDef(topic="/veh/chassis_report", label="chassis")
```

其中 `/calib/calib_param` 有特殊逻辑：

```text
找到第一帧有效标定后，后续 checkpoint 复用这一帧标定数据，但文件序号继续递增。
```

## 主方法：_extract_json_topic（被调用方法都在后面）

```python
def _extract_json_topic(  # 定义函数：从 struct MCAP 中抽取某个 JSON 类 topic
        topic_def: JsonTopicDef,  # 参数：目标 topic 配置，包含 topic 名称和日志 label
        struct_slices: list[Path],  # 参数：struct_*.mcap 文件列表
        checkpoints: list[dict],  # 参数：质检采样点列表，每个包含 seq_no、ts，可能包含 count
        output_dir: Path,  # 参数：JSON 文件输出目录
) -> list[Path]:  # 返回值：成功写出的 JSON 文件路径列表
    """对单个 JsonTopicDef，按 checkpoint 各取 count 帧并序列化为 JSON 文件。  # 方法说明：按 checkpoint 抽消息并写 JSON

    对于 /calib/calib_param topic，定位到第一帧有效数据后，后续所有  # 特殊说明：标定 topic 只需找到一次有效数据
    checkpoint 都复用这同一帧数据，但上报数据序号保持递增。  # 后续 checkpoint 复用缓存，但 seq 继续增长

    Args:  # 参数说明开始
        topic_def:      目标 topic 定义（topic 名称 + 日志标签）  # topic_def 的含义
        struct_slices:  包含该 topic 的 struct MCAP 切片列表  # struct_slices 的含义
        checkpoints:    质检采样点列表，每项含 seq_no 和 ts（毫秒），可选 count  # checkpoints 的含义
        output_dir:     输出目录  # output_dir 的含义

    Returns:  # 返回值说明开始
        成功写出的 Path 列表  # 返回写出的文件路径
    """  # 文档字符串结束
    outputs: list[Path] = []  # 准备结果列表，用来保存成功写出的 JSON 文件路径
    label = topic_def.label  # 取日志标签，例如 calib 或 chassis
    is_calib = topic_def.topic == "/calib/calib_param"  # 判断当前 topic 是否是标定 topic

    # 在 struct 切片中找到实际存在的 topic 名（可能含命名空间前缀变体）  # 注释：先确认 MCAP 里真实 topic 名
    matched_topic: str | None = None  # 初始化匹配到的真实 topic，暂时为空
    for sl in struct_slices:  # 遍历每个 struct MCAP 切片
        try:  # 捕获坏文件或读取异常，避免单个切片影响整体流程
            with open(sl, "rb") as f:  # 以二进制方式打开 MCAP 文件
                reader = make_reader(f)  # 创建 MCAP reader，用来读取 summary 和消息
                summary = reader.get_summary()  # 读取 MCAP 摘要，里面有 channel/topic 信息
                for ch in summary.channels.values():  # 遍历所有 channel
                    if topic_def.topic in ch.topic:  # 如果目标 topic 片段包含在真实 topic 名里
                        matched_topic = ch.topic  # 记录真实 topic 名
                        break  # 找到后退出 channel 遍历
        except Exception as e:  # 捕获读取当前切片时的异常
            logger.warning(  # 打印 warning，不中断整个流程
                f"  [qc] skip corrupted struct slice {sl.name}: {type(e).__name__}: {e}"  # 日志：跳过损坏切片
            )  # logger.warning 调用结束
        if matched_topic:  # 如果已经找到真实 topic
            break  # 不再继续扫描后续切片

    if matched_topic is None:  # 如果所有切片都没找到目标 topic
        logger.warning(  # 打印 warning
            f"  [QC-ADC] topic '{topic_def.topic}' not found in struct slices"  # 日志：目标 topic 不存在
        )  # logger.warning 调用结束
        return outputs  # 返回空列表，表示没有输出

    logger.info(f"  [{label}] {matched_topic}")  # 打印当前处理的 topic
    topic_safe = _topic_to_filename_prefix(matched_topic)  # 把 topic 转成安全文件名前缀
    topic_index = _build_slice_time_index(struct_slices)  # 为 struct 切片建立时间索引，方便按时间找文件
    cp_start_seq = _expand_checkpoints(checkpoints)  # 计算每个 checkpoint 的输出起始序号

    # calib topic 缓存：找到第一帧有效数据后，后续 checkpoint 复用  # 注释：标定数据通常低频，可复用
    calib_cached: tuple[dict, int] | None = None  # 缓存结构：(标定 dict, 该帧 log_time)

    for cp in checkpoints:  # 遍历每个 checkpoint
        count = _cp_count(cp)  # 读取当前 checkpoint 要抽多少帧，默认 1
        seq_no = cp["seq_no"]  # 读取 checkpoint 原始编号，例如 "001"
        start_seq = cp_start_seq.get(seq_no, int(seq_no))  # 获取当前 checkpoint 的输出起始序号
        ts_ns = int(cp["ts"]) * 1_000_000  # checkpoint 时间戳是毫秒，转换成纳秒

        # ── calib topic 且有缓存：直接复用，序号递增 ──────────────  # 注释：标定已解析过则不再读 MCAP
        if is_calib and calib_cached is not None:  # 如果是标定 topic，并且已有缓存
            cached_dict, cached_log_time = calib_cached  # 取出缓存的标定 JSON 内容和原始 log_time
            for frame_offset in range(count):  # 按 count 生成对应数量的输出文件
                seq_idx = start_seq + frame_offset  # 计算当前输出文件的最终序号
                out_name = f"{topic_safe}_{cached_log_time}_{seq_idx:03d}.json"  # 生成 JSON 文件名
                out_path = output_dir / out_name  # 拼出完整输出路径
                out_path.parent.mkdir(parents=True, exist_ok=True)  # 确保输出目录存在
                out_path.write_text(  # 写 JSON 文件
                    json.dumps(cached_dict, ensure_ascii=False, indent=2),  # 把 dict 格式化成 JSON 字符串，保留中文
                    encoding="utf-8",  # 使用 UTF-8 编码
                )  # write_text 调用结束
                out_path.with_suffix(".topic").write_text(matched_topic, encoding="utf-8")  # 写同名 .topic 文件，保存真实 topic
                outputs.append(out_path)  # 把输出 JSON 路径加入结果列表
                logger.info(f"    [{label}] seq={seq_idx:03d} (cached) -> {out_name}")  # 打印缓存输出日志
            continue  # 当前 checkpoint 已处理完，进入下一个 checkpoint

        slice_path = _find_slice_for_ns(topic_index, ts_ns)  # 根据 checkpoint 时间找到包含该时间的 struct 切片
        if slice_path is None:  # 如果没有任何切片覆盖该时间
            logger.warning(f"    [{label}] seq={start_seq:03d} ts={cp['ts']}: no slice found")  # 打印找不到切片日志
            continue  # 跳过当前 checkpoint

        frames = _extract_frames_after(slice_path, matched_topic, ts_ns, count)  # 从切片中取 log_time >= ts_ns 的连续帧
        if not frames:  # 如果 checkpoint 之后没有取到帧
            # Fallback: 所有 struct 切片中向前查找最近一帧  # 注释：低频 topic 可能 checkpoint 后没有新消息
            frame = _find_last_frame_before(struct_slices, matched_topic, ts_ns)  # 在所有切片中找 checkpoint 前最近一帧
            if frame is None:  # 如果向前也找不到
                logger.warning(  # 打印 warning
                    f"    [{label}] seq={start_seq:03d} ts={cp['ts']}: no frame "  # 日志前半段：没有帧
                    f"(slice={slice_path.name})"  # 日志后半段：当前切片名
                )  # logger.warning 调用结束
                continue  # 跳过当前 checkpoint
            logger.info(  # 如果 fallback 找到了帧，打印 info
                f"    [{label}] seq={start_seq:03d}: fallback used "  # 日志前半段：使用 fallback
                f"(matched slice had no forward frame)"  # 日志后半段：匹配切片中没有向后帧
            )  # logger.info 调用结束
            frames = [frame]  # 把 fallback 找到的一帧包装成列表，统一后续处理

        # 低频 topic 帧数不足 count 时，用最后一帧重复填充  # 注释：保证输出数量满足 count
        found = len(frames)  # 记录实际找到的帧数
        if found < count:  # 如果找到的帧数少于需要的 count
            last_frame = frames[-1]  # 取最后一帧
            frames = list(frames) + [last_frame] * (count - found)  # 用最后一帧重复补齐
            logger.info(  # 打印补齐日志
                f"    [{label}] seq={start_seq:03d}: only {found} frame(s) found, "  # 日志前半段：实际找到帧数
                f"repeating last frame for {count} total"  # 日志后半段：重复最后一帧补齐
            )  # logger.info 调用结束

        for frame_offset, (msg_data, schema, log_time) in enumerate(frames):  # 遍历每一帧消息数据、schema、真实 log_time
            seq_idx = start_seq + frame_offset  # 计算当前输出文件的最终序号
            calib_dict = _deserialize_calib_msg(msg_data, schema)  # 把 CDR 消息反序列化成 Python dict
            if calib_dict is None:  # 如果反序列化失败
                logger.warning(f"    [{label}] seq={seq_idx:03d}: deserialization failed")  # 打印失败日志
                continue  # 跳过当前帧

            # calib topic：缓存第一帧有效数据  # 注释：供后续 checkpoint 复用
            if is_calib and calib_cached is None:  # 如果当前是标定 topic，且还没有缓存
                calib_cached = (calib_dict, log_time)  # 缓存当前标定内容和 log_time
                logger.info(f"    [{label}] cached first valid frame (seq={seq_idx:03d})")  # 打印缓存日志

            out_name = f"{topic_safe}_{log_time}_{seq_idx:03d}.json"  # 生成 JSON 文件名
            out_path = output_dir / out_name  # 拼出完整输出路径
            out_path.parent.mkdir(parents=True, exist_ok=True)  # 确保输出目录存在
            out_path.write_text(  # 写 JSON 文件
                json.dumps(calib_dict, ensure_ascii=False, indent=2),  # 把 dict 转成格式化 JSON 字符串
                encoding="utf-8",  # 使用 UTF-8 编码写入
            )  # write_text 调用结束
            # sidecar: 保存原始 topic，供上报时还原使用  # 注释：旁路文件记录真实 topic
            out_path.with_suffix(".topic").write_text(matched_topic, encoding="utf-8")  # 写同名 .topic 文件
            outputs.append(out_path)  # 把成功输出的 JSON 路径加入结果列表
            logger.info(f"    [{label}] seq={seq_idx:03d} -> {out_name}")  # 打印输出成功日志

    return outputs  # 返回所有成功写出的 JSON 文件路径
```

## 被调用方法：JsonTopicDef

```python
@dataclasses.dataclass(frozen=True)  # 使用 dataclass 自动生成初始化方法；frozen=True 表示创建后不可修改
class JsonTopicDef:  # 定义 JSON topic 配置类
    """描述一类"struct 消息 → JSON 文件"数据的配置。  # 类说明：描述某类 struct 消息如何导出 JSON

    Attributes:  # 属性说明开始
        topic:       ROS2 topic 名称，用于 MCAP channel 匹配  # topic：目标 topic 片段
        label:       日志中的简短标签（如 "calib"、"chassis"）  # label：日志展示名称
    """  # 文档字符串结束

    topic: str  # 目标 topic 名称或关键片段
    label: str  # 日志标签
```

## 被调用方法：_topic_to_filename_prefix

```python
def _topic_to_filename_prefix(topic: str) -> str:  # 定义函数：把 topic 转成适合文件名的前缀
    """将 topic 路径转换为安全文件名前缀。  # 函数说明：去掉斜杠并替换路径分隔符

    /sensor/camera_front_image         → sensor_camera_front_image  # 示例：相机 topic 转文件名前缀
    /sensor/lidar_pandar_pointcloud    → sensor_lidar_pandar_pointcloud  # 示例：激光 topic 转文件名前缀
    """  # 文档字符串结束
    return topic.strip("/").replace("/", "_")  # 去掉首尾 /，再把中间 / 替换成 _
```

## 被调用方法：_build_slice_time_index

```python
def _build_slice_time_index(slices: list[Path]) -> list[tuple[Path, int, int]]:  # 定义函数：为 MCAP 切片建立时间索引
    """预建切片时间索引：[(path, start_ns, end_ns), ...] 按 start_ns 升序。"""  # 返回每个切片的路径、开始纳秒、结束纳秒
    result = []  # 准备索引列表
    for p in slices:  # 遍历每个 MCAP 切片路径
        tr = get_ptp_time_range_from_mcap(p)  # 读取该切片的 PTP 时间范围
        if tr:  # 如果成功读取到时间范围
            result.append((p, tr[0], tr[1]))  # 加入索引：(路径, 开始时间, 结束时间)
    return sorted(result, key=lambda x: x[1])  # 按开始时间升序排序后返回
```

## 被调用方法：_find_slice_for_ns

```python
def _find_slice_for_ns(  # 定义函数：根据目标纳秒时间找对应切片
    index: list[tuple[Path, int, int]],  # 参数：切片时间索引
    target_ns: int,  # 参数：目标纳秒时间戳
) -> Path | None:  # 返回值：匹配到的切片路径，找不到返回 None
    """在切片时间索引中找到包含 target_ns 的切片路径。"""  # 函数说明：判断 start <= target <= end
    for path, s, e in index:  # 遍历每个切片索引项
        if s <= target_ns <= e:  # 如果目标时间落在该切片时间范围内
            return path  # 返回这个切片路径
    return None  # 所有切片都不包含目标时间，返回 None
```

## 被调用方法：_cp_count

```python
def _cp_count(cp: dict) -> int:  # 定义函数：读取 checkpoint 要抽取的帧数
    """读取 checkpoint 的 count 字段，默认 1（兼容无 count 的旧 checkpoint）。"""  # 函数说明：没有 count 就按 1 处理
    return int(cp.get("count", 1))  # 从 cp 中读取 count，缺省值为 1，并转成 int
```

## 被调用方法：_expand_checkpoints

```python
def _expand_checkpoints(checkpoints: list[dict]) -> dict[str, int]:  # 定义函数：计算每个 checkpoint 的输出起始序号
    """计算每个 checkpoint 的输出起始序号（连续递增）。  # 函数说明：处理 count>1 时的连续编号

    5 个 checkpoint × count=3 → 输出序号 001→015。  # 示例：5 个点，每个 3 帧，共 15 个序号
    旧 checkpoint 无 count 字段时，按 count=1 处理。  # 兼容旧数据

    Args:  # 参数说明开始
        checkpoints: [{"seq_no": "001", "ts": "...", "count": 3}, ...]  # checkpoint 列表示例

    Returns:  # 返回值说明开始
        {seq_no: start_seq_idx, ...}  如 {"001": 1, "002": 4, "003": 7, ...}  # 返回 seq_no 到起始序号的映射
    """  # 文档字符串结束
    result: dict[str, int] = {}  # 准备结果字典
    running_seq = 1  # 当前可用的输出序号，从 1 开始
    for cp in checkpoints:  # 遍历 checkpoint
        result[cp["seq_no"]] = running_seq  # 当前 checkpoint 的起始输出序号
        running_seq += _cp_count(cp)  # 下一个 checkpoint 的起始序号向后移动 count 位
    return result  # 返回映射表
```

## 被调用方法：_extract_frames_after

```python
def _extract_frames_after(  # 定义函数：从某个 MCAP 切片中抽取指定 topic 的连续帧
    slice_path: Path,  # 参数：MCAP 切片路径
    topic: str,  # 参数：目标 topic
    ts_ns: int,  # 参数：起始纳秒时间戳
    count: int = 1,  # 参数：最多抽取多少帧，默认 1
) -> list[tuple[bytes, object, int]]:  # 返回值：[(消息原始 bytes, schema, log_time_ns), ...]
    """从切片中取出 log_time >= ts_ns 的连续帧（最多 count 帧）。  # 函数说明：从 checkpoint 时间之后取帧

    当 count=1 时，等价于 _extract_first_frame_after 但返回单元素列表。  # 说明 count=1 的行为

    Args:  # 参数说明开始
        slice_path: MCAP 切片路径  # slice_path 的含义
        topic:      要提取的 topic  # topic 的含义
        ts_ns:      起始纳秒时间戳  # ts_ns 的含义
        count:      需要提取的连续帧数（默认 1）  # count 的含义

    Returns:  # 返回值说明开始
        [(message_data, schema, log_time_ns), ...] 最多 count 个元素  # 返回消息数据、schema 和真实日志时间
    """  # 文档字符串结束
    result: list[tuple[bytes, object, int]] = []  # 准备结果列表
    if count <= 0:  # 如果请求帧数小于等于 0
        return result  # 直接返回空列表
    try:  # 捕获读取 MCAP 时的异常
        with open(slice_path, "rb") as f:  # 以二进制方式打开 MCAP 文件
            reader = make_reader(f)  # 创建 MCAP reader
            summary = reader.get_summary()  # 获取 summary，后面用 channel.schema_id 找 schema
            for _, channel, message in reader.iter_messages(  # 遍历指定 topic 的消息
                topics=[topic],  # 只读取目标 topic
                start_time=ts_ns,  # 只读取 log_time >= ts_ns 的消息
            ):  # iter_messages 参数结束
                result.append((message.data, summary.schemas.get(channel.schema_id), message.log_time))  # 保存消息 bytes、schema、log_time
                if len(result) >= count:  # 如果已经达到需要的帧数
                    break  # 停止读取
    except Exception as e:  # 捕获异常
        logger.warning(f"  [qc] skip corrupted slice {slice_path.name}: {type(e).__name__}: {e}")  # 记录坏切片 warning
    return result  # 返回抽到的帧列表
```

## 被调用方法：_find_last_frame_before

```python
def _find_last_frame_before(  # 定义函数：跨多个切片查找指定时间前最近一帧
    slices: list[Path],  # 参数：MCAP 切片路径列表
    topic: str,  # 参数：目标 topic
    ts_ns: int,  # 参数：checkpoint 纳秒时间
) -> tuple[bytes, object, int] | None:  # 返回值：一帧消息，找不到返回 None
    """跨所有切片，找到 log_time < ts_ns 的最新一帧（fallback 用）。  # 函数说明：向前找最近帧

    当 _extract_first_frame_after 返回 None 时调用。适用于低频 topic  # 使用场景：checkpoint 后没有帧
    （如 /calib/calib_param），其 checkpoint 后可能没有消息。  # 低频 topic 例子

    Args:  # 参数说明开始
        slices: MCAP 切片路径列表  # slices 的含义
        topic:  目标 topic 名称  # topic 的含义
        ts_ns:  checkpoint 纳秒时间戳  # ts_ns 的含义

    Returns:  # 返回值说明开始
        (message_data, schema, log_time_ns) 或 None  # 返回最近一帧或 None
    """  # 文档字符串结束
    best: tuple[bytes, object, int] | None = None  # 保存目前找到的最佳帧，也就是最接近 ts_ns 的前一帧
    for sp in slices:  # 遍历所有切片
        try:  # 捕获单个切片读取异常
            with open(sp, "rb") as f:  # 以二进制方式打开切片
                reader = make_reader(f)  # 创建 MCAP reader
                summary = reader.get_summary()  # 获取 summary，用来拿 schema
                for _, channel, message in reader.iter_messages(  # 遍历目标 topic 的消息
                    topics=[topic],  # 只读取目标 topic
                    end_time=ts_ns,  # 只读取 log_time < ts_ns 的消息
                    log_time_order=True,  # 按 log_time 排序
                    reverse=True,  # 倒序读取，越新的消息越先出现
                ):  # iter_messages 参数结束
                    candidate = (  # 组装候选帧
                        message.data,  # 消息原始 bytes
                        summary.schemas.get(channel.schema_id),  # 消息 schema
                        message.log_time,  # 消息真实 log_time
                    )  # candidate 组装结束
                    if best is None or message.log_time > best[2]:  # 如果还没有 best，或当前帧更接近 ts_ns
                        best = candidate  # 更新 best
                    break  # 当前切片倒序第一条就是该切片中最新的前置帧，直接跳出
        except Exception as e:  # 捕获当前切片异常
            logger.warning(f"  [qc-fallback] slice={sp.name}: {e}")  # 记录 fallback 读取失败日志
    return best  # 返回跨切片找到的最近前置帧；可能为 None
```

## 被调用方法：_deserialize_calib_msg

```python
def _deserialize_calib_msg(data: bytes, schema) -> dict | None:  # 定义函数：把 CDR 消息反序列化成 Python dict
    """将 SensorCalibration CDR 消息反序列化为 Python dict，失败返回 None。"""  # 函数说明：失败返回 None
    try:  # 捕获反序列化异常
        ts, type_name = _build_typestore(schema)  # 根据 schema 构建 typestore，并拿到类型名
        msg = ts.deserialize_cdr(data, type_name)  # 用 rosbags 把 CDR bytes 反序列化成消息对象
        return _ros_to_dict(msg)  # 把消息对象递归转成 JSON 可序列化的 dict
    except Exception as e:  # 捕获异常
        logger.warning(f"  [qc] calib deserialize failed: {e}")  # 打印反序列化失败日志
        return None  # 返回 None 表示失败
```

## 被调用方法：_build_typestore

```python
def _build_typestore(schema):  # 定义函数：从 MCAP schema 构建 rosbags typestore
    """从 MCAP schema 构建 rosbags typestore（含所有依赖类型）。"""  # 函数说明：注册 root 类型和依赖类型
    raw = schema.data.decode() if isinstance(schema.data, bytes) else schema.data  # schema.data 可能是 bytes，先转成字符串
    name = schema.name.strip()  # 取 schema 类型名，并去掉首尾空白
    if "/msg/" not in name and name.count("/") == 1:  # 如果类型名是 pkg/Type，而不是 pkg/msg/Type
        pkg, typ = name.split("/")  # 拆出包名和类型名
        name = f"{pkg}/msg/{typ}"  # 转成 rosbags 需要的 pkg/msg/Type
    parts = re.split(r"={20,}\nMSG: ", raw)  # 按 ROS msg 依赖分隔符拆分 schema 文本
    add_types: dict = {}  # 准备保存待注册类型
    add_types.update(get_types_from_msg(parts[0], name))  # 解析主消息类型
    for part in parts[1:]:  # 遍历依赖消息类型
        nl = part.index("\n")  # 找到第一行结束位置
        dep = part[:nl].strip()  # 第一行是依赖类型名
        if "/msg/" not in dep and dep.count("/") == 1:  # 如果依赖类型名也是 pkg/Type 格式
            pkg2, typ2 = dep.split("/")  # 拆出依赖包名和类型名
            dep = f"{pkg2}/msg/{typ2}"  # 转成 pkg/msg/Type 格式
        add_types.update(get_types_from_msg(part[nl + 1:], dep))  # 解析依赖消息体并加入待注册类型
    ts = get_typestore(Stores.ROS2_HUMBLE)  # 创建基于 ROS2 Humble 的 typestore
    ts.register(add_types)  # 注册主类型和依赖类型
    return ts, name  # 返回 typestore 和主类型名
```

## 被调用方法：_ros_to_dict

```python
def _ros_to_dict(obj) -> object:  # 定义函数：把 rosbags 消息对象递归转成 JSON 可序列化对象
    """将 rosbags 反序列化对象递归转换为 JSON 可序列化的 dict/list/scalar。"""  # 函数说明：处理 dataclass、数组、numpy 类型、bytes
    if dataclasses.is_dataclass(obj) and not isinstance(obj, type):  # 如果 obj 是 dataclass 实例
        return {  # 返回字典
            f.name: _ros_to_dict(getattr(obj, f.name))  # 字段值继续递归转换
            for f in dataclasses.fields(obj)  # 遍历 dataclass 的所有字段
            if f.name != "__msgtype__"  # 跳过内部消息类型字段
        }  # 字典推导结束
    if isinstance(obj, (list, tuple)):  # 如果是 list 或 tuple
        return [_ros_to_dict(x) for x in obj]  # 逐个元素递归转换成 list
    if isinstance(obj, np.ndarray):  # 如果是 numpy 数组
        return obj.tolist()  # 转成 Python list，JSON 才能序列化
    if isinstance(obj, np.integer):  # 如果是 numpy 整数
        return int(obj)  # 转成 Python int
    if isinstance(obj, np.floating):  # 如果是 numpy 浮点数
        return float(obj)  # 转成 Python float
    if isinstance(obj, bytes):  # 如果是 bytes
        return obj.hex()  # 转成十六进制字符串，JSON 才能保存
    return obj  # 普通类型直接返回，例如 str、int、float、bool、None
```

## 外部方法说明

这些是外部库或标准库方法，文档不展开源码：

```text
make_reader(f)
```

创建 MCAP reader，用来读取 summary 和消息。

```text
reader.get_summary()
```

读取 MCAP 摘要，里面有 channels、schemas 等元信息。

```text
reader.iter_messages(...)
```

遍历 MCAP 消息，可按 topic、start_time、end_time 过滤。

```text
get_ptp_time_range_from_mcap(p)
```

读取 MCAP 切片的 PTP 起止时间。

```text
get_types_from_msg(...)
```

把 ROS `.msg` 文本解析成 rosbags 可注册的类型定义。

```text
get_typestore(Stores.ROS2_HUMBLE)
```

创建 ROS2 Humble 类型库。

```text
ts.deserialize_cdr(data, type_name)
```

按 type_name 对应的消息结构，把 CDR bytes 反序列化成 Python 对象。

## 整体流程

```text
1. 传入一个 JsonTopicDef，例如 /calib/calib_param
2. 扫描 struct_*.mcap 的 summary，找到真实 topic 名
3. 为 struct 切片建立时间索引
4. 遍历 checkpoint
5. 根据 checkpoint 时间找到对应切片
6. 从切片中取 checkpoint 后的连续帧
7. 如果后面没有帧，就 fallback 向前找最近一帧
8. 如果帧数不足 count，用最后一帧补齐
9. 把 CDR 消息反序列化成 dict
10. 写出 JSON 文件和 .topic sidecar 文件
11. 返回所有 JSON 输出路径
```

## 一句话总结

```text
_extract_json_topic 负责把 ADC struct MCAP 中指定 topic 的消息，按 checkpoint 时间抽出来，反序列化成 JSON，并处理低频 topic 的复用和补帧问题。
```
