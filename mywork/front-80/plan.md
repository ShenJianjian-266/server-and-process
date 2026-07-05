# 前方 80m 点云裁剪完整实现方案

## 1. 目标与原则

目标：在 QC 点云产物中仅保留车辆前方 80m 点云。已确认“前方一定是 `x` 正方向”，因此统一过滤条件为：

```python
0.0 <= x <= 80.0  # 判断一个点的 x 坐标是否在车辆前方 0 到 80 米之间，左右边界都算保留。
```

实现原则：

- 只影响 QC ADC 与 QC Truth，不误伤 `event_parse` 等复用 `extract_pcd_from_message()` 的流程。
- ADC 在 PointCloud2 解析并完成 cm→m 转换后过滤。
- Truth 在 `pcl_tool` 生成 PCD 后过滤，再输出到 QC 目录，禁止失败后直接 copy 原文件。
- 支持 `DATA ascii` 与 `DATA binary` PCD；第一版明确拒绝 `binary_compressed`。
- 过滤为空时写合法空 PCD，保证后续上传和上报流程稳定。

## 2. 改动文件

- `sacp-dms-data-process/data_parse/qc_utils.py`
  - 新增前方过滤常量。
  - 新增点列表过滤函数。
  - 新增 PCD header 解析、ASCII/Binary PCD 过滤与原子写入函数。
  - 扩展 `extract_pcd_from_message()`，增加 `front_x_filter` 显式开关，默认 `False`。
- `sacp-dms-data-process/data_parse/pipeline_qc_adc.py`
  - 调用 `extract_pcd_from_message(..., front_x_filter=True)`。
- `sacp-dms-data-process/data_parse/pipeline_qc_truth.py`
  - 从 `qc_utils` import `filter_pcd_file_front_x_range`。
  - 用过滤写出替换 `shutil.copyfile()`。

## 3. `qc_utils.py` 新增 import

在现有 import 区域补充：

```python
import math  # 引入数学工具，后面用 math.isfinite 判断 x 是否不是 NaN/Inf。
import tempfile  # 引入临时文件工具，后面先写临时 PCD，再替换目标文件。
from dataclasses import dataclass  # 引入 dataclass，用少量代码定义只保存数据的元信息类。

import numpy as np  # 引入 numpy，并命名为 np，用于高效处理 binary PCD 的大量点数据。
```

说明：

- `math.isfinite()` 用于过滤 `NaN/Inf` 坐标。
- `tempfile.NamedTemporaryFile()` 用于先写临时文件再原子替换，避免半文件。
- `numpy` 已在 `requirements.txt` 中存在，用于 Truth binary PCD 向量化过滤。

## 4. `qc_utils.py` 新增常量与元数据结构

建议放在 `_PCD_TYPE_MAP` 附近：

```python
_QC_FRONT_X_MIN_M = 0.0  # 定义前方裁剪的最小 x 值，单位是米，0 表示车身当前位置。
_QC_FRONT_X_MAX_M = 80.0  # 定义前方裁剪的最大 x 值，单位是米，只保留 80 米以内的点。


@dataclass(frozen=True)  # 自动生成初始化等方法，并禁止创建后修改字段，避免元数据被误改。
class _PcdMeta:  # 定义 PCD 文件头的元数据结构，方便后续过滤函数统一使用。
    """PCD header 的结构化信息，便于 ASCII/Binary 共用校验逻辑。"""  # 说明这个类保存解析后的 PCD 头信息。

    header_lines: list[str]  # 保存原始 header 每一行文本，后面重写 WIDTH/HEIGHT/POINTS 时会复用。
    fields: list[str]  # 保存字段名列表，例如 x、y、z、intensity。
    sizes: list[int]  # 保存每个字段占用的字节数，例如 float32 是 4 字节。
    types: list[str]  # 保存每个字段的数据类型标记，例如 F 表示浮点数，I/U 表示有符号/无符号整数。
    counts: list[int]  # 保存每个字段包含几个值，普通 x/y/z 通常都是 1。
    width: int  # 保存 PCD header 中的 WIDTH，通常表示一行有多少个点。
    height: int  # 保存 PCD header 中的 HEIGHT，1 表示非组织点云。
    points: int  # 保存 PCD header 中声明的点数量。
    data: str  # 保存 DATA 格式，例如 ascii 或 binary。
    data_start: int  # 保存 DATA 行结束后的字节偏移，也就是点数据 payload 的起始位置。
```

## 5. ADC 点列表过滤代码

新增函数：

```python
def filter_front_x_points(  # 定义一个函数，用来过滤已经解析成 Python 字典列表的点云点。
    points: list[dict],  # 输入参数 points：每个点是一个 dict，里面应包含 x、y、z 等字段。
    min_x_m: float = _QC_FRONT_X_MIN_M,  # 输入参数 min_x_m：允许保留的最小 x，默认使用全局常量 0 米。
    max_x_m: float = _QC_FRONT_X_MAX_M,  # 输入参数 max_x_m：允许保留的最大 x，默认使用全局常量 80 米。
) -> list[dict]:  # 返回值类型是 list[dict]，也就是过滤后的点列表。
    """保留 x 正方向指定距离内的点。  # 函数说明：只保留 x 在指定范围内的点。

    该函数用于已解析出的 PointCloud2 点列表。调用前必须确保坐标单位已经是米。  # 提醒调用者：这里按米判断，不负责单位转换。
    缺失 x、x 无法转 float、x 为 NaN/Inf 的点都丢弃，避免写出不可判断的点。  # 说明异常点的处理策略：不能可靠判断就不要保留。
    """  # 文档字符串结束，真正代码从下一行开始执行。
    filtered: list[dict] = []  # 创建空列表，用来收集通过 x 范围检查的点。
    for pt in points:  # 逐个遍历输入点列表中的每一个点。
        try:  # 尝试读取并转换当前点的 x 坐标。
            x = float(pt["x"])  # 从点字典中取出 x 字段，并转成浮点数，方便做数值比较。
        except (KeyError, TypeError, ValueError):  # 如果没有 x、点不是合法字典、或 x 不能转成数字，就进入这里。
            continue  # 跳过这个异常点，继续处理下一个点。

        # 非有限值会破坏后续可视化和校验，直接丢弃。
        if math.isfinite(x) and min_x_m <= x <= max_x_m:  # 同时要求 x 不是 NaN/Inf，并且落在 [min_x_m, max_x_m] 区间内。
            filtered.append(pt)  # 当前点满足条件，把它加入过滤后的结果列表。

    logger.debug(  # 打一条 debug 日志，方便排查过滤前后点数是否符合预期。
        "  [qc-front-x] points before=%d after=%d range=[%.2f, %.2f]",  # 日志模板，依次打印过滤前数量、过滤后数量、最小 x、最大 x。
        len(points),  # 第一个占位符：输入点总数。
        len(filtered),  # 第二个占位符：过滤后剩余点数。
        min_x_m,  # 第三个占位符：本次使用的最小 x。
        max_x_m,  # 第四个占位符：本次使用的最大 x。
    )  # logger.debug 调用结束。
    return filtered  # 返回过滤后的点列表。
```

## 6. ADC 接入方式

修改 `extract_pcd_from_message()` 签名，增加关键字参数，默认不裁剪，避免影响非 QC 调用方：

```python
def extract_pcd_from_message(  # 定义从 MCAP 消息中提取点云并可选写出 PCD 的函数。
    message_data: bytes,  # 输入参数 message_data：PointCloud2 CDR 的原始二进制消息。
    schema,  # 输入参数 schema：这条消息对应的 MCAP schema，用来辅助解析字段结构。
    output_path: Path | None = None,  # 输入参数 output_path：输出 PCD 路径；为 None 时只解析、不写文件。
    *,  # 这里的 * 表示后面的参数必须用关键字传入，避免调用时传错位置。
    front_x_filter: bool = False,  # 输入参数 front_x_filter：是否启用前方 0 到 80 米裁剪，默认不启用以保持兼容。
) -> bool:  # 返回 bool：True 表示解析/写出成功，False 表示失败。
    """从一帧 PointCloud2 CDR 消息解析点云，按需写出 PCD ASCII 文件。  # 函数用途说明。

    Args:  # 参数说明开始。
        message_data: PointCloud2 CDR 原始字节。  # message_data 是未解析的原始消息内容。
        schema: 对应 MCAP schema 对象。  # schema 告诉解析函数如何理解原始字节。
        output_path: 输出 PCD 路径；None 时只解析不落盘。  # 控制是否生成 PCD 文件。
        front_x_filter: True 时仅保留 0<=x<=80m 的点。默认 False，保证老调用方行为不变。  # 控制是否启用本次新增的前方裁剪。
    """  # 文档字符串结束。
    result = _parse_pc2_to_points(message_data, schema)  # 调用已有解析逻辑，把二进制 PointCloud2 消息转成 points 和 fields。
    if result is None:  # 如果解析失败，内部约定会返回 None。
        return False  # 解析失败时直接返回 False，告诉调用方本帧没有成功处理。
    points, fields = result  # 解包解析结果：points 是点列表，fields 是字段定义列表。

    if _needs_unit_conversion(fields):  # 判断当前点云坐标是否需要从厘米转换成米。
        points = _convert_cm_to_m(points)  # 如果需要转换，就把所有点的 x/y/z 从 cm 转成 m。
        fields = [  # 同步重建字段定义，确保 x/y/z 的数据类型也改成 float32。
            {**f, "datatype": 7} if f["name"] in ("x", "y", "z") else f  # 如果字段是 x/y/z，就复制原字段并把 datatype 改为 7；其他字段保持原样。
            for f in fields  # 遍历原始 fields 列表中的每个字段定义。
        ]  # 字段列表重建结束。

    # 必须放在 cm->m 之后，否则 ADC INT16 cm 会把 80m 误判为 80cm。
    if front_x_filter:  # 只有调用方显式传 front_x_filter=True 时才执行前方裁剪。
        points = filter_front_x_points(points)  # 对已经转换成米的点列表应用 0 到 80 米过滤。

    if output_path is not None:  # 如果调用方提供了输出路径，就需要把点云写成 PCD 文件。
        _write_pcd_ascii(output_path, points, fields)  # 调用已有 ASCII PCD 写入函数，把过滤后的 points 写到 output_path。
    return True  # 能走到这里说明处理成功，返回 True。
```

修改 `pipeline_qc_adc.py` 中写 PCD 的调用：

```python
ok = extract_pcd_from_message(  # 调用点云提取函数，并把成功/失败结果保存到 ok。
    msg_data,  # 传入当前 MCAP 消息的原始二进制内容。
    schema,  # 传入当前消息对应的 schema，供解析函数识别字段。
    out_path,  # 传入要写出的 PCD 文件路径。
    front_x_filter=True,  # QC ADC 明确启用前方 80m 裁剪。
)  # 函数调用结束，ok 为 True 表示本帧 PCD 生成成功。
```

## 7. PCD Header 解析代码

新增通用解析函数：

```python
def _parse_pcd_header(header_lines: list[str], data_start: int) -> _PcdMeta:  # 定义 PCD header 解析函数，输入 header 行和 payload 起始偏移。
    """解析 PCD header，并做必要一致性校验。"""  # 函数说明：把文本 header 转成结构化对象，并检查格式是否可靠。
    values: dict[str, list[str]] = {}  # 创建字典保存 header 键值，例如 {"FIELDS": ["x", "y", "z"]}。
    for line in header_lines:  # 遍历 header 的每一行文本。
        stripped = line.strip()  # 去掉当前行首尾空白，方便判断和拆分。
        if not stripped or stripped.startswith("#"):  # 如果是空行或注释行，就不参与字段解析。
            continue  # 跳过这一行，继续处理下一行。
        parts = stripped.split()  # 按空白拆分当前行，例如 "WIDTH 5" 会变成 ["WIDTH", "5"]。
        values[parts[0].upper()] = parts[1:]  # 用大写 key 保存字段名，value 保存后面的参数列表。

    def require(name: str) -> list[str]:  # 定义内部小函数，用来读取必填 header 字段。
        if name not in values or not values[name]:  # 如果字段不存在，或字段后面没有任何值，就说明 header 不合法。
            raise ValueError(f"PCD header missing {name}")  # 抛出异常，明确告诉调用方缺少哪个 PCD header 字段。
        return values[name]  # 返回这个字段对应的值列表。

    fields = require("FIELDS")  # 读取字段名列表，例如 ["x", "y", "z"]。
    sizes = [int(v) for v in require("SIZE")]  # 读取每个字段的字节大小，并把字符串转成整数。
    types = [v.upper() for v in require("TYPE")]  # 读取每个字段的数据类型，并统一转成大写方便后续比较。
    counts = [int(v) for v in values.get("COUNT", ["1"] * len(fields))]  # 读取 COUNT；如果 header 没写 COUNT，则按 PCD 约定每个字段默认 1。
    width = int(require("WIDTH")[0])  # 读取 WIDTH，并转成整数。
    height = int(values.get("HEIGHT", ["1"])[0])  # 读取 HEIGHT；如果 header 没写 HEIGHT，则默认按 1 处理。
    points = int(values.get("POINTS", [str(width * height)])[0])  # 读取 POINTS；如果没写，则用 WIDTH * HEIGHT 推导点数。
    data = require("DATA")[0].lower()  # 读取 DATA 类型，并转成小写，例如 ascii 或 binary。

    if not (len(fields) == len(sizes) == len(types) == len(counts)):  # 校验字段名、大小、类型、数量四组配置长度必须一致。
        raise ValueError("PCD FIELDS/SIZE/TYPE/COUNT length mismatch")  # 长度不一致时无法正确解析每个点，直接报错。
    if len(set(fields)) != len(fields):  # 把字段列表转成 set 后如果长度变小，说明存在重复字段名。
        raise ValueError("PCD duplicate field names are not supported")  # 重复字段会导致 numpy structured dtype 歧义，直接拒绝。
    if points < 0 or width < 0 or height < 0:  # 点数、宽、高都不能是负数。
        raise ValueError("PCD negative WIDTH/HEIGHT/POINTS")  # 出现负数说明 header 明显不合法。
    if "x" not in fields:  # 前方裁剪必须依赖 x 坐标，所以 header 必须包含 x 字段。
        raise ValueError("PCD missing x field")  # 没有 x 字段就无法判断前方 80 米，直接失败。

    return _PcdMeta(header_lines, fields, sizes, types, counts, width, height, points, data, data_start)  # 把解析出的所有信息打包成 _PcdMeta 返回。
```

读取文件时按二进制查找 `DATA` 行，确保 binary payload 不被文本解码破坏：

```python
def _read_pcd_file(src_path: Path) -> tuple[_PcdMeta, bytes]:  # 定义读取 PCD 文件的函数，返回元数据和点数据字节。
    """读取 PCD 文件，返回 header 元数据和 DATA 后的原始 payload。"""  # 函数说明：header 用文本解析，payload 保留原始 bytes。
    data = src_path.read_bytes()  # 一次性按二进制读取整个 PCD 文件，避免 binary 数据被错误解码。
    offset = 0  # offset 表示当前读取到文件中的哪个字节位置，初始从 0 开始。
    header_lines: list[str] = []  # 创建列表保存 header 文本行，直到遇到 DATA 行为止。

    while offset < len(data):  # 只要还没有读到文件末尾，就继续按行扫描 header。
        newline = data.find(b"\n", offset)  # 从当前 offset 开始寻找下一个换行符位置。
        if newline < 0:  # 如果找不到换行符，说明 header 没有正常结束。
            raise ValueError("PCD header missing DATA line")  # 没有 DATA 行就不知道 payload 从哪里开始，直接报错。

        line_bytes = data[offset:newline + 1]  # 切出当前这一行的原始字节，包含末尾换行符。
        line = line_bytes.decode("ascii", errors="strict").rstrip("\r\n")  # header 必须是 ASCII；解码后去掉行尾换行。
        header_lines.append(line)  # 把当前 header 行保存起来，后面会解析和重写。
        offset = newline + 1  # 移动 offset 到下一行的起始位置。

        if line.strip().upper().startswith("DATA "):  # 如果当前行是 DATA 行，说明 header 到这里结束。
            meta = _parse_pcd_header(header_lines, offset)  # 解析已经收集到的 header 行，并记录 payload 起始 offset。
            return meta, data[offset:]  # 返回 header 元数据，以及 DATA 行后面的原始 payload 字节。

    raise ValueError("PCD header missing DATA line")  # 如果循环结束仍没遇到 DATA 行，说明 PCD 文件格式不完整。
```

## 8. Header 重写与原子写入

```python
def _rewrite_pcd_header(header_lines: list[str], point_count: int) -> bytes:  # 定义 header 重写函数，用裁剪后的点数更新 header。
    """重写点数相关字段。裁剪后统一写成一维非组织点云。"""  # 函数说明：更新 WIDTH/HEIGHT/POINTS，其他 header 行尽量保持原样。
    rewritten: list[str] = []  # 创建新列表，逐行保存重写后的 header。
    for line in header_lines:  # 遍历原始 header 的每一行。
        key = line.strip().split(maxsplit=1)[0].upper() if line.strip() else ""  # 取出当前行的第一个单词作为 header key，空行则 key 为空。
        if key == "WIDTH":  # 如果当前行是 WIDTH。
            rewritten.append(f"WIDTH {point_count}")  # 把 WIDTH 改成裁剪后的点数量。
        elif key == "HEIGHT":  # 如果当前行是 HEIGHT。
            rewritten.append("HEIGHT 1")  # 裁剪后统一写成非组织点云，所以 HEIGHT 固定为 1。
        elif key == "POINTS":  # 如果当前行是 POINTS。
            rewritten.append(f"POINTS {point_count}")  # 把 POINTS 改成裁剪后的点数量。
        else:  # 其他 header 行不需要改。
            rewritten.append(line)  # 原样保留当前 header 行。
    return ("\n".join(rewritten) + "\n").encode("ascii")  # 用换行拼回 header 文本，并编码成 ASCII bytes。


def _atomic_write_bytes(dst_path: Path, content: bytes) -> None:  # 定义原子写入函数，负责安全写出二进制内容。
    """先写同目录临时文件，再 replace，避免失败时留下半截 PCD。"""  # 函数说明：先写临时文件，成功后再替换正式文件。
    dst_path.parent.mkdir(parents=True, exist_ok=True)  # 确保目标文件所在目录存在，不存在就递归创建。
    tmp_name: str | None = None  # 记录临时文件路径；初始化为 None，方便异常时判断是否需要清理。
    try:  # 开始执行可能失败的文件写入流程。
        with tempfile.NamedTemporaryFile(  # 在目标目录创建一个具名临时文件。
            "wb",  # 以二进制写模式打开临时文件。
            dir=dst_path.parent,  # 临时文件放在目标文件同一个目录，保证 replace 通常是同文件系统原子操作。
            prefix=f".{dst_path.name}.",  # 临时文件名前缀带上目标文件名，方便排查。
            suffix=".tmp",  # 临时文件名后缀使用 .tmp，表示这是临时产物。
            delete=False,  # 关闭文件时不自动删除，因为后面还要用 Path.replace 替换目标文件。
        ) as tmp:  # 把打开的临时文件对象命名为 tmp。
            tmp.write(content)  # 把完整 PCD 内容写入临时文件。
            tmp.flush()  # 刷新 Python 缓冲，尽快把内容交给操作系统。
            tmp_name = tmp.name  # 保存临时文件路径，供后面 replace 或异常清理使用。
        Path(tmp_name).replace(dst_path)  # 用写好的临时文件替换目标文件；成功后目标路径就是完整新文件。
    except Exception:  # 如果写入、刷新或替换过程中发生任何异常，就进入清理逻辑。
        if tmp_name:  # 只有已经创建过临时文件时才需要删除。
            Path(tmp_name).unlink(missing_ok=True)  # 删除临时文件；如果文件已不存在也不报错。
        raise  # 重新抛出原异常，让上层知道写入失败。
```

## 9. ASCII PCD 过滤代码

```python
def _filter_ascii_pcd_payload(meta: _PcdMeta, payload: bytes) -> bytes:  # 定义 ASCII PCD 点数据过滤函数。
    """过滤 DATA ascii payload，并返回完整 PCD 文件字节。"""  # 函数说明：输入 header 元数据和 payload，输出完整的新 PCD bytes。
    x_idx = meta.fields.index("x")  # 找到 x 字段在每一行数据中的列下标，例如 FIELDS x y z 时 x_idx 是 0。
    kept_rows: list[str] = []  # 创建列表，保存过滤后仍然要保留的 ASCII 数据行。

    text = payload.decode("ascii", errors="strict")  # 把 DATA ascii 的 payload 从 bytes 解码成普通字符串。
    for line in text.splitlines():  # 按行遍历每一条点数据，一行通常就是一个点。
        if not line.strip():  # 如果当前行去掉空白后为空，说明是空行。
            continue  # 空行不代表有效点，直接跳过。
        parts = line.split()  # 按空白拆分当前点的数据列，例如 "1 2 3" 变成 ["1", "2", "3"]。
        if len(parts) < len(meta.fields):  # 如果数据列数量少于 header 里声明的字段数量，说明这一行格式不完整。
            raise ValueError("PCD ascii row has fewer columns than FIELDS")  # 格式不完整时直接报错，避免生成错误 PCD。

        try:  # 尝试读取并转换当前行的 x 值。
            x = float(parts[x_idx])  # 按 x_idx 取出 x 列，并转成浮点数。
        except ValueError:  # 如果 x 字符串不是合法数字。
            continue  # 无法判断位置的点直接丢弃，继续处理下一行。

        # 边界值 0 和 80 都保留；NaN/Inf 丢弃。
        if math.isfinite(x) and _QC_FRONT_X_MIN_M <= x <= _QC_FRONT_X_MAX_M:  # 只保留有限数字，并且 x 落在 0 到 80 米闭区间内的点。
            kept_rows.append(line.rstrip())  # 保存当前原始数据行，去掉行尾多余空白但不改变字段值。

    header = _rewrite_pcd_header(meta.header_lines, len(kept_rows))  # 根据保留下来的行数重写 header 中的点数量。
    body = ("\n".join(kept_rows) + ("\n" if kept_rows else "")).encode("ascii")  # 把保留的数据行重新拼成 ASCII bytes；非空时补最后一个换行。
    return header + body  # 返回完整的新 PCD 文件内容：header 加过滤后的 body。
```

## 10. Binary PCD 过滤代码

```python
_PCD_NUMPY_TYPE_MAP = {  # 定义 PCD 类型到 numpy dtype 字符串的映射表。
    ("F", 4): "<f4",  # PCD 浮点 F、4 字节，对应小端 float32。
    ("F", 8): "<f8",  # PCD 浮点 F、8 字节，对应小端 float64。
    ("I", 1): "i1",  # PCD 有符号整数 I、1 字节，对应 int8；单字节不需要大小端。
    ("U", 1): "u1",  # PCD 无符号整数 U、1 字节，对应 uint8；单字节不需要大小端。
    ("I", 2): "<i2",  # PCD 有符号整数 I、2 字节，对应小端 int16。
    ("U", 2): "<u2",  # PCD 无符号整数 U、2 字节，对应小端 uint16。
    ("I", 4): "<i4",  # PCD 有符号整数 I、4 字节，对应小端 int32。
    ("U", 4): "<u4",  # PCD 无符号整数 U、4 字节，对应小端 uint32。
    ("I", 8): "<i8",  # PCD 有符号整数 I、8 字节，对应小端 int64。
    ("U", 8): "<u8",  # PCD 无符号整数 U、8 字节，对应小端 uint64。
}  # 映射表结束；不在表里的类型会被显式拒绝。


def _build_pcd_dtype(meta: _PcdMeta) -> np.dtype:  # 定义函数，把 PCD header 里的字段描述转换成 numpy 可理解的 dtype。
    """根据 PCD SIZE/TYPE/COUNT 构造 numpy structured dtype。"""  # 函数说明：构造结构化数组类型，每个点是一条记录。
    dtype_fields = []  # 创建列表，后面逐个追加 numpy 字段定义。
    for name, size, type_char, count in zip(meta.fields, meta.sizes, meta.types, meta.counts):  # 同时遍历字段名、字节大小、类型和 COUNT。
        if count <= 0:  # COUNT 表示字段中值的个数，必须是正数。
            raise ValueError(f"PCD invalid COUNT for field {name}: {count}")  # COUNT 小于等于 0 时 header 不合法，直接报错。
        key = (type_char, size)  # 把 PCD 类型和字节数合成查表 key，例如 ("F", 4)。
        if key not in _PCD_NUMPY_TYPE_MAP:  # 如果映射表里没有这个类型组合，说明当前代码不支持它。
            raise ValueError(f"PCD unsupported field type: field={name} type={type_char} size={size}")  # 明确报出不支持的字段、类型和大小。

        base_dtype = np.dtype(_PCD_NUMPY_TYPE_MAP[key])  # 根据映射表创建 numpy 基础类型，例如 float32 或 uint16。
        if count == 1:  # 如果字段只有一个值，这是最常见的 x/y/z 情况。
            dtype_fields.append((name, base_dtype))  # 追加普通字段定义，例如 ("x", float32)。
        else:  # 如果字段包含多个值，例如某些特征向量字段。
            # 非 x 字段允许 COUNT>1，以 subarray 方式完整保留。
            dtype_fields.append((name, base_dtype, (count,)))  # 追加数组字段定义，保留这个字段的所有 count 个值。

    dtype = np.dtype(dtype_fields)  # 把字段定义列表转换成 numpy structured dtype。
    if meta.counts[meta.fields.index("x")] != 1:  # x 字段必须只有一个值，否则无法作为单个坐标比较。
        raise ValueError("PCD x field COUNT must be 1")  # x 的 COUNT 不是 1 时直接拒绝。
    return dtype  # 返回构造好的 dtype，供 np.frombuffer 解析 binary payload。


def _filter_binary_pcd_payload(meta: _PcdMeta, payload: bytes) -> bytes:  # 定义 binary PCD 点数据过滤函数。
    """过滤 DATA binary payload，并返回完整 PCD 文件字节。"""  # 函数说明：输入原始二进制点数据，输出完整的新 PCD bytes。
    dtype = _build_pcd_dtype(meta)  # 根据 header 构造每个点的 numpy 数据结构。
    expected_size = meta.points * dtype.itemsize  # 计算理论 payload 大小：点数乘以每个点占用字节数。
    if len(payload) != expected_size:  # 校验实际 payload 字节数是否和 header 声明一致。
        raise ValueError(  # 不一致时抛异常，避免按错误结构读取导致数据错位。
            f"PCD binary payload size mismatch: expected={expected_size}, actual={len(payload)}"  # 异常信息同时给出期望和实际字节数。
        )  # ValueError 构造结束。

    arr = np.frombuffer(payload, dtype=dtype, count=meta.points)  # 不复制数据，直接把 payload 解释成 structured array，每个元素是一个点。
    x_values = arr["x"].astype(np.float64, copy=False)  # 取出所有点的 x 列，并转成 float64 方便统一做 isfinite 和范围比较。

    # 使用 numpy 向量化过滤，避免 Truth 大点云逐点 Python 循环。
    mask = (  # 构造布尔掩码数组，mask 中每个 True/False 对应一个点是否保留。
        np.isfinite(x_values)  # 条件 1：x 必须是有限数字，不能是 NaN 或 Inf。
        & (x_values >= _QC_FRONT_X_MIN_M)  # 条件 2：x 必须大于等于最小前方距离 0 米。
        & (x_values <= _QC_FRONT_X_MAX_M)  # 条件 3：x 必须小于等于最大前方距离 80 米。
    )  # 三个条件用 & 做逐元素“并且”组合。
    filtered = arr[mask]  # 用布尔掩码筛选原数组，只留下满足条件的点。

    header = _rewrite_pcd_header(meta.header_lines, len(filtered))  # 根据过滤后的点数重写 PCD header。
    return header + filtered.tobytes()  # 返回完整新文件内容：重写后的 header 加过滤后点数组的二进制 bytes。
```

## 11. Truth PCD 对外过滤函数

```python
def filter_pcd_file_front_x_range(src_path: Path, dst_path: Path) -> bool:  # 定义对外函数：读取一个 PCD，裁剪后写到另一个路径。
    """将 PCD 文件裁剪为前方 80m 并写到目标路径。  # 函数说明：这是 Truth 流程调用的入口。

    返回 True 表示已生成可信目标文件；False 表示失败，调用方不得 fallback copy 原文件。  # 返回值约定：False 时不能上传未裁剪原文件。
    """  # 文档字符串结束。
    try:  # 捕获过滤过程中的异常，统一转成 False 返回。
        meta, payload = _read_pcd_file(src_path)  # 读取源 PCD 文件，得到 header 元数据和原始点数据。

        if meta.data == "ascii":  # 如果 PCD 的 DATA 类型是 ascii。
            content = _filter_ascii_pcd_payload(meta, payload)  # 使用 ASCII 过滤逻辑生成新的完整 PCD 内容。
        elif meta.data == "binary":  # 如果 PCD 的 DATA 类型是 binary。
            content = _filter_binary_pcd_payload(meta, payload)  # 使用 binary 过滤逻辑生成新的完整 PCD 内容。
        else:  # 如果既不是 ascii 也不是 binary。
            # 当前 pcl_tool 已关闭 binary_compressed；遇到未知格式要显式失败。
            raise ValueError(f"unsupported PCD DATA type: {meta.data}")  # 明确拒绝 unsupported 或 binary_compressed，避免偷偷写错文件。

        _atomic_write_bytes(dst_path, content)  # 将过滤后的完整 PCD 内容安全写入目标路径。
        logger.info(  # 写一条 info 日志，记录本次过滤成功。
            "  [qc-front-x] filtered PCD %s -> %s data=%s points_before=%d",  # 日志模板：源文件、目标文件、格式、过滤前点数。
            src_path.name,  # 第一个占位符：源 PCD 文件名。
            dst_path.name,  # 第二个占位符：目标 PCD 文件名。
            meta.data,  # 第三个占位符：PCD DATA 类型，例如 ascii 或 binary。
            meta.points,  # 第四个占位符：header 中声明的过滤前点数。
        )  # logger.info 调用结束。
        return True  # 成功写出目标文件，返回 True。
    except Exception as e:  # 捕获任意异常，保证调用方拿到 False 而不是进程崩溃。
        logger.warning(  # 写 warning 日志，方便排查过滤失败原因。
            "  [qc-front-x] failed to filter PCD src=%s dst=%s: %s",  # 日志模板：源路径、目标路径、异常信息。
            src_path,  # 第一个占位符：源文件完整路径。
            dst_path,  # 第二个占位符：目标文件完整路径。
            e,  # 第三个占位符：异常对象，日志里会显示异常消息。
            exc_info=False,  # 不打印完整堆栈，避免日志过长；需要时可改成 True。
        )  # logger.warning 调用结束。
        return False  # 过滤失败，返回 False，调用方必须跳过该输出。
```

## 12. Truth 接入代码

在 `pipeline_qc_truth.py` 的 `from qc_utils import (...)` 中加入：

```python
filter_pcd_file_front_x_range,  # 从 qc_utils 导入 Truth PCD 前方 80m 裁剪函数。
```

将 `_process_topic()` 中的：

```python
shutil.copyfile(pcd_path, out_path)  # 旧逻辑：直接把 pcl_tool 生成的 PCD 原样复制到 QC 输出目录，没有做前方 80m 裁剪。
# sidecar: 保存原始 topic，供上报时还原使用  # 写 .topic 辅助文件，用于后续上报时找回原始 topic。
out_path.with_suffix(".topic").write_text(topic, encoding="utf-8")  # 在 PCD 同目录写一个同名 .topic 文件，内容是原始 topic 字符串，编码使用 UTF-8。
outputs.append(out_path)  # 把这个 PCD 输出路径加入 outputs，后续流程会上传或上报它。
logger.info(f"    [{topic}] seq={seq_idx:03d} → {out_name}")  # 打印当前 topic、序号和输出文件名，方便查看处理进度。
```

替换为：

```python
ok = filter_pcd_file_front_x_range(pcd_path, out_path)  # 调用前方 80m 裁剪函数，把源 PCD 过滤后写到目标路径，并保存成功状态。
if not ok:  # 如果裁剪失败，ok 会是 False。
    # 失败时不写 .topic、不加入 outputs，避免未裁剪 PCD 被上传或上报。
    logger.warning(f"    [{topic}] seq={seq_idx:03d}: front-x filter failed, skip output")  # 记录当前 topic 和序号裁剪失败，并说明会跳过输出。
    continue  # 跳过当前这帧/这个文件，继续处理下一个，避免未裁剪文件进入后续流程。

# sidecar: 保存原始 topic，供上报时还原使用。  # 只有裁剪成功后才写 sidecar，保证 sidecar 和有效 PCD 一一对应。
out_path.with_suffix(".topic").write_text(topic, encoding="utf-8")  # 写出 .topic 文件，内容是原始 topic，供上报阶段恢复 topic。
outputs.append(out_path)  # 把裁剪成功的 PCD 路径加入 outputs，允许后续上传和上报。
logger.info(f"    [{topic}] seq={seq_idx:03d} → {out_name}")  # 打印成功输出日志，包含 topic、序号和文件名。
```

如文件中其他位置仍使用 `shutil`，不要删除 `import shutil`。

## 13. 测试建议

新增最小单元测试文件，例如：

`sacp-dms-data-process/test_qc_front_x_filter.py`

核心覆盖：

```python
from pathlib import Path  # 引入 Path，用面向对象的方式拼接和读写临时文件路径。

from data_parse.qc_utils import filter_pcd_file_front_x_range  # 导入要测试的前方 80m PCD 裁剪函数。


def test_ascii_front_x_filter(tmp_path: Path):  # 定义 pytest 测试函数，tmp_path 是 pytest 提供的临时目录。
    """验证边界：x=0 和 x=80 保留，x<0 与 x>80 删除。"""  # 测试目的：确认过滤区间是闭区间 [0, 80]。
    src = tmp_path / "in.pcd"  # 构造输入 PCD 文件路径，放在临时目录中。
    dst = tmp_path / "out.pcd"  # 构造输出 PCD 文件路径，也放在临时目录中。
    src.write_text(  # 写入一个最小可用的 ASCII PCD 文件，作为测试输入。
        "# .PCD v0.7 - Point Cloud Data file format\n"  # PCD 文件格式说明行，属于 header 注释。
        "VERSION 0.7\n"  # PCD 版本号。
        "FIELDS x y z\n"  # 声明每个点有 x、y、z 三个字段。
        "SIZE 4 4 4\n"  # 声明三个字段各占 4 字节。
        "TYPE F F F\n"  # 声明三个字段都是浮点数 F。
        "COUNT 1 1 1\n"  # 声明每个字段都只有 1 个值。
        "WIDTH 5\n"  # 声明一共有 5 个点。
        "HEIGHT 1\n"  # 声明这是非组织点云，高度为 1。
        "POINTS 5\n"  # 再次声明点数量为 5。
        "DATA ascii\n"  # 声明后面的点数据是 ASCII 文本格式。
        "-1 0 0\n"  # 第 1 个点：x=-1，小于 0，应该被删除。
        "0 0 0\n"  # 第 2 个点：x=0，刚好在左边界，应该保留。
        "10 0 0\n"  # 第 3 个点：x=10，在 0 到 80 之间，应该保留。
        "80 0 0\n"  # 第 4 个点：x=80，刚好在右边界，应该保留。
        "80.1 0 0\n",  # 第 5 个点：x=80.1，大于 80，应该被删除。
        encoding="ascii",  # 指定用 ASCII 编码写文件，和 PCD ascii 格式匹配。
    )  # 测试输入文件写入完成。

    assert filter_pcd_file_front_x_range(src, dst)  # 执行过滤函数，并断言它返回 True，表示输出文件生成成功。
    text = dst.read_text(encoding="ascii")  # 读取输出 PCD 文件内容，方便检查 header 和点数据。
    assert "POINTS 3" in text  # 过滤后应该只剩 3 个点：0、10、80。
    assert "-1 0 0" not in text  # 确认 x=-1 的点不在输出中。
    assert "80.1 0 0" not in text  # 确认 x=80.1 的点不在输出中。
```

Binary PCD 测试建议用 `numpy` 构造 `x y z intensity ring timestamp` structured array，写入 header + `arr.tobytes()`，再校验输出 `POINTS` 与 payload 长度。

## 14. 验证命令

```bash
cd sacp-dms-data-process
python test_qc_front_x_filter.py
python test_event_pipeline_local.py
```

端到端验证：

1. 跑一段 ADC QC，抽查所有输出 PCD 的 `x` 均满足 `0.0 <= x <= 80.0`。
2. 跑一段 Truth QC，确认输出文件不是原始 copy，`.topic` sidecar 正常生成。
3. 注入缺失 `x`、`binary_compressed`、payload 长度不一致的 PCD，确认失败可见且不会生成目标产物。

## 15. 回滚与兼容性

- `extract_pcd_from_message()` 默认 `front_x_filter=False`，非 QC 调用方行为保持不变。
- ADC 回滚只需移除 `front_x_filter=True`。
- Truth 回滚可恢复 `shutil.copyfile()`，但不建议在生产回退到未裁剪输出；更推荐修复过滤失败原因。
- 若后续前方距离可配置，只需将 `_QC_FRONT_X_MAX_M` 改为环境变量读取，并在日志中打印生效值。
