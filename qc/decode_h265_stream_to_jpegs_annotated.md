# _decode_h265_stream_to_jpegs 注释版

## 方法作用

`_decode_h265_stream_to_jpegs` 的作用：

```text
把一段 H265 视频字节流交给 ffmpeg 解码，并输出 JPEG 图片字节。
```

它主要用于相机抽帧：

```text
MCAP 消息里的 H265 数据 -> 临时 stream.h265 文件 -> ffmpeg 解码 -> frame_xxxxxx.jpg -> 读取 JPEG bytes
```

## 完整注释版代码

```python
def _decode_h265_stream_to_jpegs(# 定义函数：把 H265 字节流解码成 JPEG 图片列表
        h265_data: bytes,# 参数：H265 原始字节数据，类型是 bytes
        tmp_dir: Path,# 参数：临时目录，用来存 H265 文件和解码出的 JPEG 文件
        select_indices: list[int] | None = None,# 参数：要抽取的帧序号列表，None 表示不指定帧
) -> list[tuple[int, bytes]]:# 返回值：列表；每项是 (帧输出编号, JPEG 字节)
    """将 H265 字节流用 ffmpeg 解码，只输出 select_indices 指定的帧（保持原始分辨率）
    Args:
        h265_data:      完整 H265 NAL 字节流（单切片即可，首帧为 IDR）
        tmp_dir:        临时工作目录
        select_indices: 0-based 帧序号列表；None 时输出全部（应避免）# select_indices：从 0 开始的帧下标
    Returns:
        [(output_order_1based_idx, jpeg_bytes), ...] 按序号升序
    """
    import subprocess# 导入 subprocess，用来调用外部命令 ffmpeg
    import glob as _glob# 导入 glob，并起别名 _glob，用来查找 frame_*.jpg 文件
    if not shutil.which("ffmpeg"):# 检查系统 PATH 中是否能找到 ffmpeg
        raise RuntimeError("ffmpeg not found in PATH; required for H265 decoding.")
    tmp_dir.mkdir(parents=True, exist_ok=True)# 创建临时目录；父目录不存在也一起创建；已存在不报错
    h265_path = tmp_dir / "stream.h265"# 拼出临时 H265 文件路径，例如 tmp/stream.h265
    h265_path.write_bytes(h265_data)# 把内存中的 H265 字节写入 stream.h265 文件
    cmd = [# 开始构造 ffmpeg 命令参数列表
        "ffmpeg", "-y",# ffmpeg：执行程序；-y：输出文件已存在时直接覆盖
        "-c:v", "hevc",# 指定视频解码器类型为 HEVC，也就是 H265
        "-i", str(h265_path),# -i：指定输入文件，也就是刚写出的 stream.h265
        "-q:v", "1",# 设置 JPEG 输出质量，1 表示高质量
        "-threads", str(os.cpu_count() or 4),# 设置线程数；优先用 CPU 核数，取不到时用 4
    ]# ffmpeg 基础命令构造结束
    if select_indices:# 如果指定了要抽取的帧下标
        expr = "+".join(f"eq(n\\,{i})" for i in sorted(set(select_indices)))  # 生成 ffmpeg select 表达式，例如 eq(n\,3)+eq(n\,5)
        cmd += ["-vf", f"select='{expr}'", "-vsync", "0"] # 添加视频过滤器：只选择指定帧；-vsync 0 防止补帧或改帧
    cmd.append(str(tmp_dir / "frame_%06d.jpg")) # 添加输出文件模板，例如 frame_000001.jpg
    result = subprocess.run(cmd, capture_output=True, check=False) # 执行 ffmpeg；捕获 stdout/stderr；失败时不自动抛异常
    if result.returncode != 0: # 如果 ffmpeg 返回码不是 0，表示执行过程中有错误或警告
        stderr_msg = result.stderr.decode("utf-8", errors="replace").strip()[:300]  # 把 stderr 解码成文本，只取前 300 个字符
        logger.warning(f"  [qc] ffmpeg warn: {stderr_msg}") # 记录 ffmpeg 警告，便于排查解码问题
    frames: list[tuple[int, bytes]] = [] # 创建结果列表，用来保存解码出的 JPEG
    for fpath in sorted(_glob.glob(str(tmp_dir / "frame_*.jpg"))): # 查找所有输出 JPEG，并按文件名排序
        p = Path(fpath) # 把字符串路径转成 Path 对象，方便后续操作
        if p.stat().st_size == 0: # 如果文件大小是 0，说明是空文件或异常输出
            continue # 跳过这个无效 JPEG 文件
        idx = int(p.stem.split("_")[-1]) # 从文件名 frame_000001 中取出编号 1
        frames.append((idx, p.read_bytes())) # 读取 JPEG 文件字节，并连同编号加入结果列表
    return frames # 返回所有有效 JPEG，格式是 [(编号, 图片字节), ...]
```

## 关键变量

```text
h265_data
```

内存里的 H265 视频字节。

```text
tmp_dir
```

临时目录。函数会在里面生成：

```text
stream.h265
frame_000001.jpg
frame_000002.jpg
...
```

```text
select_indices
```

要抽哪几帧。

例如：

```python
select_indices = [3, 5]
```

表示只抽第 4 帧和第 6 帧。因为它是 `0-based`，也就是从 0 开始数。

## ffmpeg 命令大致长这样

如果只抽第 3 帧和第 5 帧：

```python
select_indices = [3, 5]
```

生成的核心命令类似：

```bash
ffmpeg -y -c:v hevc -i stream.h265 -q:v 1 -threads 8 -vf "select='eq(n\,3)+eq(n\,5)'" -vsync 0 frame_%06d.jpg
```

含义：

```text
-y                     覆盖已有输出文件
-c:v hevc              按 H265/HEVC 解码
-i stream.h265         输入文件
-q:v 1                 高质量输出 JPEG
-threads 8             使用 8 个线程
-vf select=...         只选择指定帧
-vsync 0               不自动补帧
frame_%06d.jpg         输出文件名模板
```

## 返回值示例

假设输出了两个文件：

```text
frame_000001.jpg
frame_000002.jpg
```

函数会返回：

```python
[
    (1, b"...jpeg bytes..."),
    (2, b"...jpeg bytes..."),
]
```

其中：

```text
1                  来自文件名 frame_000001.jpg
b"...jpeg bytes..." JPEG 图片二进制内容
```

## 一句话总结

```text
这个方法负责把 H265 视频数据落盘成临时文件，再调用 ffmpeg 解码成 JPEG，最后把 JPEG 文件读回内存返回。
```
