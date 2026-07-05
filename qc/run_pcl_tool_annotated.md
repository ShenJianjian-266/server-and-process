# run_pcl_tool 注释版

## 方法作用

`run_pcl_tool` 的作用：

```text
修改 Hesai SDK 的 pcl_tool 配置文件，然后启动 pcl_tool，
把一个 pcap 点云文件转换成一批 PCD 文件。
```

在 Truth QC 中，它负责：

```text
pcap -> pcl_tool -> PCD 目录
```

后续 QC 再从这些 PCD 文件中按 checkpoint 抽帧。

## 相关常量

```python
_HERE = Path(__file__).parent.parent  # 项目 data_parse 目录
PCL_TOOL = str(_HERE / "lib" / "pcl_tool")  # pcl_tool 可执行文件路径
SDK_CONFIG = str(_HERE / "lib" / "pcl_tool_config.ini")  # SDK 默认配置文件路径
PANDAR_FIRETIME = str(_HERE / "reference" / "Pandar128E3X_Firetime Correction File.csv")  # pandar 火时间修正文件
PANDAR_CORRECTION = str(_HERE / "reference" / "Pandar128_Angle_Correction_File-2.csv")  # pandar 角度修正文件
QT_FIRETIME = str(_HERE / "reference" / "QT128C2X_Firetime Correction File.csv")  # qt 火时间修正文件
QT_CORRECTION = str(_HERE / "reference" / "QT128C2X_Angle Correction File.csv")  # qt 角度修正文件
```

## 完整注释版代码

```python
def run_pcl_tool(pcap_path: str, pcd_dir: str, max_frames: int, lidar_type: str = "qt", cfg_path: str | None = None, idle_timeout: int = 60, startup_timeout: int = 120):  # 定义函数：运行 pcl_tool，把 pcap 转成 PCD
    """修改 SDK 自带 config.ini 的关键字段后运行 pcl_tool，PCD 输出到 pcd_dir。
    Args:
        cfg_path:       如果为 None，则写入默认 SDK_CONFIG（仅用于串行场景）。  # cfg_path：配置文件路径  并行调用时传入独立临时文件路径以避免竞争。
        idle_timeout:   pcl_tool 读完 PCAP 后无新输出超过此秒数则 kill（默认 60s）。
        startup_timeout: pcl_tool 启动后首次输出前的最大等待时间（默认 120s），  # 启动阶段超时
                         用于覆盖首次加载 SDK 的较长初始化时间。
    """ 
    cfg = configparser.RawConfigParser()  # 创建配置解析器，用来读取和修改 ini 配置
    from io import StringIO  # 导入 StringIO，把字符串当成文件给 configparser 读取
    try:  # 优先读取缓存的配置模板
        cfg.read_file(StringIO(_get_config_template()))  # 从内存中的 SDK 配置模板读取配置 后面有详解
    except configparser.ParsingError:  # 如果配置模板解析失败
        cfg.read_file(open(SDK_CONFIG))  # 回退：直接从默认 SDK_CONFIG 文件读取 SDK_CONFIG也就是data_parse\lib\pcl_tool_config.ini

    firetime = PANDAR_FIRETIME if lidar_type == "pandar" else QT_FIRETIME  # 根据雷达类型选择火时间修正文件
    correction = PANDAR_CORRECTION if lidar_type == "pandar" else QT_CORRECTION  # 根据雷达类型选择角度修正文件

    cfg.set("pcap", "pcap_path", pcap_path)  # 设置输入 pcap 文件路径
    cfg.set("pcap", "firetimes_path", firetime)  # 设置火时间修正文件路径
    cfg.set("pcap", "correction_file_path", correction)  # 设置角度修正文件路径
    cfg.set("pcl", "output_dir", pcd_dir)  # 设置 PCD 输出目录
    cfg.set("pcl", "save_pcd_ascii", "false")  # 不输出 ASCII PCD
    cfg.set("pcl", "save_pcd_binary", "true")  # 输出 binary PCD
    cfg.set("pcl", "save_pcd_binary_compressed", "false" )  # 不输出 binary_compressed PCD
    cfg.set("pcl", "max_frames", str(max_frames) if max_frames > 0 else "0")  # 设置最多输出帧数；0 表示不限
    cfg.set("decoder", "pcap_play_synchronization", "false")  # 关闭按真实时间同步播放，尽快处理 pcap

    target_cfg = cfg_path or SDK_CONFIG  # 如果传了 cfg_path 就写临时配置，否则写默认配置
    with open(target_cfg, "w") as f:  # 打开目标配置文件准备写入
        cfg.write(f)  # 把修改后的配置写入文件

    if cfg_path is None:  # 如果使用默认配置文件
        for section in cfg.sections():  # 遍历配置文件中的每个 section
            for key, val in cfg.items(section):  # 遍历 section 下的每个配置项
                log.info("cfg [%s] %s = %s", section, key, val)  # 打印配置内容，便于排查

    log.info("运行 pcl_tool，最多 %d 帧，PCD 输出: %s", max_frames, pcd_dir)
    env = os.environ.copy()  # 复制当前进程环境变量
    pcl_libs = "/opt/pcl-libs"  # PCL 动态库目录
    if os.path.isdir(pcl_libs):  # 如果该动态库目录存在
        env["LD_LIBRARY_PATH"] = pcl_libs + (":" + env["LD_LIBRARY_PATH"] if env.get("LD_LIBRARY_PATH") else "")  # 把 PCL 库目录加入 LD_LIBRARY_PATH
    proc = subprocess.Popen(  # 启动外部 pcl_tool 进程
        [PCL_TOOL, target_cfg],  # 命令参数：pcl_tool 可执行文件 + 配置文件路径
        stdout=subprocess.PIPE,  # 捕获标准输出
        stderr=subprocess.STDOUT,  # 把标准错误合并到标准输出
        text=True,  # 以文本模式读取输出
        env=env,  # 使用修改后的环境变量
        cwd=pcd_dir,  # 每个进程独立工作目录，避免并发时 ./log.log 冲突导致 SIGABRT
    )  # subprocess.Popen 调用结束

    # pcl_tool 读完 PCAP 后不会自动退出，用线程监控：   # 需要 watchdog 主动结束进程
    # - 首次输出前：使用 startup_timeout（覆盖 SDK 加载时间）  # 启动阶段可能很慢
    # - 首次输出后：使用 idle_timeout（处理完毕后快速退出）  # 处理阶段长时间无输出就认为完成
    last_output = [time.monotonic()]  # 保存最近一次输出时间；用 list 是为了内部函数能修改
    first_output_seen = [False]  # 标记是否已经看到 pcl_tool 的首次输出

    def _watchdog():  # 定义监控线程函数
        while proc.poll() is None:  # 只要 pcl_tool 进程还没退出，就持续检查
            current_timeout = idle_timeout if first_output_seen[0] else startup_timeout  # 根据是否已有输出选择超时时间
            if time.monotonic() - last_output[0] > current_timeout:  # 如果距离上次输出已经超过超时时间
                phase = "处理完毕" if first_output_seen[0] else "启动超时"  # 根据阶段生成日志文案
                log.info("pcl_tool %s，无新输出 %ds，自动结束...", phase, current_timeout)  # 打印自动结束原因
                proc.kill()  # 杀掉 pcl_tool 进程
                return  # 退出 watchdog 线程
            time.sleep(0.5)  # 每 0.5 秒检查一次

    t = threading.Thread(target=_watchdog, daemon=True)  # 创建守护线程，用来监控 pcl_tool 是否卡住
    t.start()  # 启动 watchdog 线程

    line_count = [0]  # 统计 pcl_tool 输出行数；用 list 便于可变
    for line in proc.stdout:  # 持续读取 pcl_tool 输出
        if not first_output_seen[0]:  # 如果这是第一次读到输出
            first_output_seen[0] = True  # 标记已经看到首次输出
            log.info("[pcl_tool] 首次输出，后续空闲超时: %ds", idle_timeout)
        last_output[0] = time.monotonic()  # 更新最近输出时间
        line_count[0] += 1  # 输出行数加 1
        if line_count[0] % 100 == 0 or line_count[0] == 1:  # 只打印第 1 行和每 100 行，避免日志过多
            log.info("[pcl_tool] %s (line %d)", line.rstrip(), line_count[0])  # 打印 pcl_tool 输出内容

    proc.wait()  # 等待 pcl_tool 进程彻底退出
    t.join()  # 等待 watchdog 线程结束
    if proc.returncode not in (0, -9):  # 如果退出码不是正常退出 0，也不是被 kill 的 -9
        raise RuntimeError(f"pcl_tool 退出码 {proc.returncode}")  # 抛异常，告诉调用方pcl_tool 执行失败
```

## 参数解释

```text
pcap_path
```

输入 pcap 文件路径。

```text
pcd_dir
```

输出 PCD 文件目录。

```text
max_frames
```

最多输出多少帧：

```text
0 表示不限
1 表示只取第一帧，常用于快速读取 PTP 起始时间
```

```text
lidar_type
```

雷达类型：

```text
pandar -> 使用 Pandar 修正文件
qt     -> 使用 QT 修正文件
```

```text
cfg_path
```

配置文件路径。

并行运行时必须传独立临时配置文件，避免多个进程同时写同一个 `pcl_tool_config.ini`。

## 关键逻辑

### 1. 改配置

```text
设置输入 pcap
设置火时间修正文件
设置角度修正文件
设置 PCD 输出目录
设置最多输出帧数
关闭同步播放
```

### 2. 启动 pcl_tool

实际启动命令类似：

```bash
/path/to/pcl_tool /path/to/config.ini
```

### 3. watchdog 防卡死

`pcl_tool` 处理完 pcap 后可能不会主动退出，所以代码用线程监控输出：

```text
首次输出前超过 startup_timeout -> 认为启动超时，kill
首次输出后超过 idle_timeout   -> 认为处理完毕，kill
```

### 4. 允许 -9

```python
if proc.returncode not in (0, -9):
```

`-9` 表示被 `proc.kill()` 杀掉。这里是预期行为，因为 watchdog 会在空闲超时后主动结束 `pcl_tool`。

## 一句话总结

```text
run_pcl_tool 负责临时改好 Hesai SDK 配置，启动 pcl_tool 把 pcap 转成 PCD，并用 watchdog 线程在工具无输出时自动结束进程。
```

# pcap_to_mcap.py `_get_config_template` 注释版

```python
def _get_config_template() -> str:  # 定义函数，返回 SDK 配置模板字符串
    """线程安全地获取 SDK 配置模板内容（只读一次）。"""
    global _SDK_CONFIG_TEMPLATE  # 声明使用模块级缓存变量
    if _SDK_CONFIG_TEMPLATE is None:  # 缓存为空时才准备读取配置文件
        with _SDK_CONFIG_TEMPLATE_LOCK:  # 获取线程锁，避免多线程重复读取
            if _SDK_CONFIG_TEMPLATE is None:  # 加锁后再次检查，防止其他线程已完成初始化
                with open(SDK_CONFIG) as f:  # 打开 SDK 配置文件
                    _SDK_CONFIG_TEMPLATE = f.read()  # 读取文件内容并写入缓存
    return _SDK_CONFIG_TEMPLATE  # 返回已缓存的配置模板内容
```