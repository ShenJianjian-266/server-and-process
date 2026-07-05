# _parse_pc2_to_points 注释版

## 方法作用

`_parse_pc2_to_points` 的作用：

```text
把一条 ROS2 PointCloud2 的 CDR 二进制消息，解析成 Python 能处理的点云列表。
```

输入：

```text
message_data: 原始二进制点云消息
schema:       这条消息对应的 MCAP schema
```

输出：

```text
(points, fields)
```

其中：

```text
points: 每个点的数据列表，例如 [{"x": 1.2, "y": 0.3, "z": 5.6, ...}, ...]
fields: 点云字段定义，例如 x/y/z/intensity 的 offset、datatype 等
```

## 完整注释版代码

```python
def _parse_pc2_to_points(message_data: bytes, schema) -> tuple[list[dict], list[dict]] | None:  # 定义函数：把 PointCloud2 原始字节解析成点列表和字段列表；失败返回 None
    """将 PointCloud2 CDR 消息解析为 (points, fields)。Returns None on failure.""" 
    try:
        from util.cdr_pc2 import parse_pc2_cdr# 导入 PointCloud2 CDR 解析函数
        pc = parse_pc2_cdr(message_data, schema) # 调用解析函数，把原始 bytes 按 schema 解析成结构化字典 紧接着有讲解
        fields = pc["fields"] # 取出字段定义列表，例如 x、y、z、intensity 等字段信息
        point_step = pc["point_step"]                                               # 取出单个点占用的字节数
        point_data = pc["point_data"]                                               # 取出真正的点云二进制数据区
        width = pc["width"]                                                         # 取出点的数量；这里代码按 width 遍历点
        if width == 0 or point_step == 0:                                           # 如果点数量为 0，或单点字节数为 0，说明数据无效
            return None                                                             # 返回 None，表示解析失败或没有有效点

        field_offsets = [(f["name"], f["offset"], f["datatype"]) for f in fields]   # 把每个字段提炼成三元组：(字段名, 字段在单点内的字节偏移, 字段数据类型)
        data_bytes = bytes(point_data) if not isinstance(point_data, (bytes, bytearray)) else point_data  # 确保点云数据是 bytes 或 bytearray，方便按偏移读取

        points: list[dict] = []                                                     # 准备结果列表；每个元素代表一个点
        for i in range(width):                                                      # 遍历每个点；i 是第几个点，从 0 开始
            base = i * point_step                                                   # 计算当前点在 point_data 中的起始字节位置
            pt: dict = {}                                                           # 准备当前点的字典，例如 {"x": 1.0, "y": 2.0}
            for fname, foff, fdtype in field_offsets:                               # 遍历当前点里的每个字段
                fmt = _DTYPE_STRUCT_FMT.get(fdtype, "f")                            # 根据 ROS PointField datatype 找 struct 解包格式；未知类型默认按 float
                fsize = struct.calcsize(fmt)                                        # 计算该字段占多少字节，例如 float32 是 4 字节
                byte_off = base + foff                                              # 计算该字段在整段 point_data 中的绝对字节偏移
                if byte_off + fsize <= len(data_bytes):                             # 确认读取范围没有超过 point_data 总长度
                    pt[fname] = struct.unpack_from("<" + fmt, data_bytes, byte_off)[0]  # 按小端格式从指定偏移解出字段值，并放入当前点字典
            points.append(pt)                                                       # 当前点解析完成，加入 points 列表
        return points, fields                                                       # 返回点列表和字段定义列表
    except Exception as e:                                                          # 捕获所有异常，避免单帧坏数据中断整个 QC 流程
        logger.warning(f"  [qc] parse_pc2 failed: {e}", exc_info=False)             # 记录解析失败原因，但不打印完整异常栈
        return None                                                                 # 返回 None，告诉调用方本帧解析失败
```

## 相关数据类型映射

函数中用到了 `_DTYPE_STRUCT_FMT`：

```python
_DTYPE_STRUCT_FMT = {
    1: "b",  # INT8
    2: "B",  # UINT8
    3: "h",  # INT16
    4: "H",  # UINT16
    5: "i",  # INT32
    6: "I",  # UINT32
    7: "f",  # FLOAT32
    8: "d",  # FLOAT64
}
```

它的作用：

```text
把 ROS2 PointField 的 datatype 转成 Python struct 能识别的解包格式。
```

例如：

```text
datatype = 7  ->  "f"  ->  FLOAT32
datatype = 3  ->  "h"  ->  INT16
```

## 核心理解

PointCloud2 的点云数据不是一行一行文本，而是一大段二进制：

```text
point_data = 点1字节 + 点2字节 + 点3字节 + ...
```

`point_step` 表示一个点占多少字节。

例如：

```text
point_step = 16
```

那么：

```text
第 0 个点起始位置 = 0 * 16 = 0
第 1 个点起始位置 = 1 * 16 = 16
第 2 个点起始位置 = 2 * 16 = 32
```

字段 `offset` 表示字段在单个点内部的位置。

例如：

```text
x offset = 0
y offset = 4
z offset = 8
```

如果当前是第 2 个点：

```text
base = 2 * point_step = 32
x 的位置 = 32 + 0 = 32
y 的位置 = 32 + 4 = 36
z 的位置 = 32 + 8 = 40
```

代码里的这一行就是在算这个位置：

```python
byte_off = base + foff
```

## 返回值示例

假设解析出两个点：

```python
points = [
    {"x": 1.0, "y": 2.0, "z": 3.0, "intensity": 120},
    {"x": 1.5, "y": 2.5, "z": 3.5, "intensity": 118},
]
```

字段定义可能是：

```python
fields = [
    {"name": "x", "offset": 0, "datatype": 7},
    {"name": "y", "offset": 4, "datatype": 7},
    {"name": "z", "offset": 8, "datatype": 7},
    {"name": "intensity", "offset": 12, "datatype": 2},
]
```

最终返回：

```python
return points, fields
```

## 一句话总结

```text
_parse_pc2_to_points 先用 schema 解析 PointCloud2 外壳，再按 point_step、offset、datatype 从二进制点云区逐点读取字段值，最终生成 Python 字典形式的点云数据。
```

## 调用链

`_parse_pc2_to_points` 里最关键的一行是：

```python
pc = parse_pc2_cdr(message_data, schema)
```

调用链如下：

```text
_parse_pc2_to_points
  -> parse_pc2_cdr
     -> _typestore_for_schema
        -> _normalize_type_name
```

`parse_pc2_cdr` 位于：

```text
sacp-dms-data-process/data_parse/util/cdr_pc2.py
```

## parse_pc2_cdr 注释版

```python
def parse_pc2_cdr(data: bytes, schema=None) -> dict:                    # 定义函数：把 ROS2 CDR 二进制点云消息解析成 dict
    """Deserialize a ROS2 CDR PointCloud2 (standard or PointCloud2V2) into a dict.  # 函数说明：反序列化标准 PointCloud2 或自定义 PointCloud2V2

    Parsing is driven entirely by schema.data, so any field/struct change in the   # 说明：解析逻辑依赖 schema.data，不硬编码具体字段结构
    message definition is handled without code changes. schema=None assumes a      # 说明：消息定义变化时，只要 schema 正确，代码不用改
    standard sensor_msgs/msg/PointCloud2 (convenience for hand-built/standard data).  # schema 为 None 时，按标准 PointCloud2 处理

    Returns the historical dict keys (sec, nanosec, frame_id, height, width,        # 返回值说明：返回常用点云字段
    fields[name/offset/datatype/count], is_bigendian, point_step, row_step,         # 返回 fields、字节序、点大小、行大小等
    point_data (bytes), is_dense) PLUS hidden keys (_msg, _typestore, _typename)    # 还返回隐藏字段，后续重新序列化时使用
    used by serialize_pc2_cdr for byte-exact, structure-preserving re-serialization. # 隐藏字段用于保持结构不变地重新序列化
    """                                                                            # 文档字符串结束
    if schema is None:                                                             # 如果调用方没有传 schema
        ts, typename = _STD_TS, _STD_PC2                                            # 使用内置标准 ROS2 typestore 和标准 PointCloud2 类型名
    else:                                                                          # 如果调用方传了 schema
        ts, typename = _typestore_for_schema(schema)                                # 根据 MCAP schema 构建或获取 typestore 和真实类型名
    msg = ts.deserialize_cdr(data, typename)                                        # 使用 rosbags typestore 把 CDR bytes 反序列化成消息对象
    return {                                                                       # 开始组装返回字典
        "sec": msg.header.stamp.sec,                                               # 消息时间戳：秒
        "nanosec": msg.header.stamp.nanosec,                                       # 消息时间戳：纳秒部分
        "frame_id": msg.header.frame_id,                                           # 坐标系 ID，例如 lidar 坐标系名称
        "height": msg.height,                                                      # 点云高度；普通无组织点云通常是 1
        "width": msg.width,                                                        # 点云宽度；这里通常表示点的数量
        "fields": [                                                                # 开始转换字段定义列表
            {"name": f.name, "offset": f.offset,                                   # 字段名和字段在单点内的字节偏移
             "datatype": f.datatype, "count": f.count}                             # 字段数据类型和字段数量
            for f in msg.fields                                                    # 遍历消息对象里的每个 PointField
        ],                                                                         # fields 列表结束
        "is_bigendian": int(msg.is_bigendian),                                     # 是否大端字节序，转成 int 保存
        "point_step": msg.point_step,                                              # 每个点占多少字节
        "row_step": msg.row_step,                                                  # 每一行点云占多少字节
        "point_data": msg.data.tobytes(),                                          # 点云数据区转成 bytes
        "is_dense": int(msg.is_dense),                                             # 是否无无效点，转成 int 保存
        "_msg": msg,                                                               # 保存原始消息对象，供后续 serialize_pc2_cdr 复用
        "_typestore": ts,                                                          # 保存 typestore，供后续重新序列化
        "_typename": typename,                                                     # 保存类型名，供后续重新序列化
    }                                                                              # 返回字典结束
```

## _typestore_for_schema 注释版

`typestore` 可以简单理解为：

```text
rosbags 用来理解某种 ROS 消息结构的“类型字典”。
```

没有它，程序只拿到一段 bytes，不知道里面哪些字节是 `x`，哪些字节是 `y`。

```python
def _typestore_for_schema(schema):                                                 # 定义函数：根据 MCAP schema 构建 rosbags typestore
    """Build (and cache) a rosbags typestore for an mcap PointCloud2 schema.       # 函数说明：为 MCAP 点云 schema 构建并缓存 typestore

    schema.data is a concatenated ros2msg blob (root type, then '====\nMSG: pkg/Type'  # schema.data 是多个 ROS msg 定义拼在一起的文本
    blocks). Register every type not already bundled in ROS2_HUMBLE. Cached by     # 注册 ROS2_HUMBLE 里没有的自定义类型
    (schema.data, schema.name), since the pipeline replays one schema across many  # 用 schema.data 和 schema.name 作为缓存 key
    messages and the cached value carries the schema-specific typename.            # 同一个 schema 会被很多消息复用，缓存能减少重复解析

    Returns (typestore, typename).                                                 # 返回 typestore 和类型名
    """                                                                            # 文档字符串结束
    cache_key = (schema.data, schema.name)                                         # 生成缓存 key：schema 内容 + schema 名称
    cached = _TS_CACHE.get(cache_key)                                              # 从全局缓存里查是否已经构建过
    if cached is not None:                                                         # 如果缓存存在
        return cached                                                              # 直接返回缓存结果，避免重复构建

    text = schema.data.decode("utf-8")                                             # 把 schema 二进制内容解码成 UTF-8 文本
    typename = schema.name                                                         # 取 schema 的根类型名，例如 sdk_msg/msg/PointCloud2V2
    blocks = _MSG_SPLIT.split(text)                                                # 按 MSG 分隔符拆成多个 msg 定义块
    types: dict = {}                                                               # 准备保存解析出的类型定义
    types.update(get_types_from_msg(blocks[0], typename))                         # 解析第一个块，也就是根消息类型
    for blk in blocks[1:]:                                                         # 遍历后续依赖类型块
        head, _, body = blk.partition("\n")                                        # 第一行是类型名，后面是该类型的字段定义
        types.update(get_types_from_msg(body, _normalize_type_name(head)))         # 解析依赖类型，并把类型名规范化

    ts = get_typestore(_BASE_STORE)                                                # 基于 ROS2_HUMBLE 创建 typestore
    new_types = {n: t for n, t in types.items() if n not in ts.types}              # 只保留 typestore 中还没有的类型
    if new_types:                                                                  # 如果存在自定义类型或新类型
        ts.register(new_types)                                                     # 注册这些类型，让 rosbags 能解析它们

    result = (ts, typename)                                                        # 组装返回值
    _TS_CACHE[cache_key] = result                                                  # 写入缓存，下次同 schema 直接复用
    return result                                                                  # 返回 typestore 和类型名
```

## _normalize_type_name 注释版

这个函数解决类型名格式不统一的问题。

有些 schema 里写：

```text
sdk_msg/PointFieldV2
```

但 rosbags 需要：

```text
sdk_msg/msg/PointFieldV2
```

代码：

```python
def _normalize_type_name(name: str) -> str:                                        # 定义函数：把 schema 类型名转换成 rosbags 需要的格式
    """Normalize 'pkg/Type' from an mcap schema block to rosbags 'pkg/msg/Type'.""" # 函数说明：把 pkg/Type 规范化为 pkg/msg/Type
    name = name.strip()                                                            # 去掉类型名前后的空格和换行
    if "/msg/" not in name and name.count("/") == 1:                               # 如果还没有 /msg/，并且格式是 pkg/Type
        pkg, typ = name.split("/")                                                 # 拆成包名 pkg 和类型名 typ
        return f"{pkg}/msg/{typ}"                                                  # 拼成 rosbags 需要的 pkg/msg/Type 格式
    return name                                                                    # 如果本来就是规范格式，直接返回
```

## 外部库方法说明

这些方法来自 `rosbags`，不是本项目自己实现的：

```python
get_types_from_msg(...)
```

作用：

```text
把 .msg 文本定义解析成 rosbags 可注册的类型定义。
```

```python
get_typestore(...)
```

作用：

```text
创建一个 typestore，里面包含 ROS2 标准消息类型。
```

```python
ts.deserialize_cdr(data, typename)
```

作用：

```text
根据 typename 对应的消息结构，把 CDR bytes 反序列化成 Python 消息对象。
```

## 追加后的整体流程

```text
1. _parse_pc2_to_points 收到 message_data 和 schema
2. 调用 parse_pc2_cdr
3. parse_pc2_cdr 根据 schema 获取 typestore
4. typestore 调用 deserialize_cdr，把 bytes 变成 msg 对象
5. parse_pc2_cdr 从 msg 对象中取出 width、fields、point_step、point_data
6. _parse_pc2_to_points 再按 point_step 和 fields 逐点解析字段值
7. 最终得到 Python list[dict] 格式的点云
```
