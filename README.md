# compressvideos

>这是个整活仓库。

把以下提示词发给你的AI以获取源代码：

```
# 角色设定
你是一位拥有 20 年经验的资深 Linux 系统架构师和 Bash 脚本大师。请根据以下极其详尽的架构设计文档，使用 Bash 编写一款名为 `compressvideos` 的纯命令行批量视频压缩与转码工具。

# 核心约束
1. 语言：纯 Bash 脚本，首行必须为 `#!/usr/bin/env bash`，并在其后声明 `set -u` 以严格控制未定义变量。
2. 架构：必须完全遵循以下定义的模块、变量命名（特别是本地化字符串）、函数名映射规则及控制流。
3. 执行：按照单线程串行模式处理多文件队列。

---

## 第一部分：总体系统架构与核心引擎

**1. 智能多语言与环境嗅探引擎 (i18n & TTY Check)**：
脚本必须在顶部实现轻量级的多语言支持（中/英）：
- **基础判定**：检测 `LANG`, `LANGUAGE`, `LC_ALL`, `LC_MESSAGES` 环境变量是否包含 `zh_CN`，若是则默认设为 `LANG_ID="zh"`，否则为 `en`。
- **TTY 防乱码降级**：必须侦测当前终端是否为底层纯文本控制台（不支持 CJK 字体）。若 `$(tty 2>/dev/null)` 匹配 `/dev/tty[0-9]*`，或者 `${TERM:-}` 为 `linux`，则强制覆盖 `LANG_ID="en"`。
- **字符串字典树**：根据 `$LANG_ID` 的值，使用一个大的 `if-else` 块定义所有日志输出的字符串模板。变量名必须以 `L_` 开头（如 `L_NO_FFMPEG`, `L_START_NVENC`），并使用 `%s`, `%d` 预留占位符，以便后续使用 `printf` 格式化输出。

**2. 默认配置与状态机变量**：
- `DEFAULT_CODEC="h264"`, `DEFAULT_DEVICE="auto"`, `DEFAULT_QUALITY=5`
- 赋值给工作变量 `CODEC`, `DEVICE`, `QUALITY`。
- 其他控制变量：`PAUSE_ON_EXIT=0`, `CURRENT_OUT=""`, `FILES=()`, `OUT_FILES=()`, `ACTION_LIST_DEVICES=0`, `ACTION_LIST_CODECS=0`。

**3. 标准化彩色日志器**：
定义四个输出函数，需使用 ANSI 转义码（蓝 INFO, 黄 WARN, 红 ERROR(重定向至>&2), 绿 OK）。所有日志内容的拼接必须通过 `printf "$L_xxx"` 动态读取对应语言的格式化字符串。

---

## 第二部分：模块化详细设计与底层实现规范

### 模块 1：参数解析与 CLI 路由
使用 `while [[ $# -gt 0 ]]` 和 `case` 进行参数解析。输入文件存入 `FILES=()`，输出文件存入 `OUT_FILES=()`。
- **可选带参判定**：对于 `-c|--codec` 和 `-d|--accel-device`，必须判断 `$2` 是否存在且不以 `-` 开头（`! "$2" =~ ^-`）。如果是，则赋值；否则视为“无参查询”，设置 `ACTION_LIST_CODECS=1` 或 `ACTION_LIST_DEVICES=1`。
- `-i|--input` 和未绑定的孤立参数（`*`）均追加到 `FILES`。
- `-o|--output`：追加到 `OUT_FILES`。
- `-q|--quality`：使用正则 `^([1-9]|10)$` 限制。
- `-p|--pause`：设置 `PAUSE_ON_EXIT=1`。
- `-h|--help`：提供中/英双语帮助文档，并且必须包含 4 个典型的使用示例 (Examples)。
- **校验拦截**：循环结束后，前置检查 `ffmpeg` 命令是否存在；处理 ACTION 列表退出；校验输入输出文件数组长度是否对等 (`${#FILES[@]} -ne ${#OUT_FILES[@]}`)。

### 模块 2：硬件探测与挂载映射 (list_accel_devices / get_drm_device)
必须通过读取 Linux 底层文件节点来侦测物理设备：
- 遍历 `/sys/class/drm/renderD*`，读取 `device/driver/module` 软链接的 basename。
- 映射规则：`nvidia`/`nvidia_drm` -> nvidia 节点；`amdgpu`/`radeon` -> amdgpu 节点；`i915`/`xe` -> intel 节点。
- 构建 `/dev/dri/renderD[X]` 绝对路径以备硬件加速引擎挂载。

### 模块 3：编码器兼容性探测引擎
调用 `ffmpeg -hide_banner -encoders 2>/dev/null`：
- `get_nvenc_codecs`：awk 匹配 `nvenc` 及 `^ V`，sed 剥除 `_nvenc`。
- `get_vaapi_codecs`：同理剥除 `_vaapi`。
- `get_cpu_codecs`：grep 提取 `libx264` (映射 h264), `libx265` (hevc), `libvpx-vp9` (vp9), `libsvtav1`|`libaom-av1` (av1)。
提供 `is_codec_supported(device, codec)` 进行准入校验。

### 模块 4：核心处理管道与 FFmpeg 参数拼装
所有 ffmpeg 调用必须包含 `-y` 且通过 `</dev/null` 重定向标准输入，防止吞噬终端输入流。提供 `get_crf_value(quality)` 将 1~10 映射为 38~11（步长 3）。
- **4.1 智能分流 (run_audio_only)**：扩展名忽略大小写匹配音频格式，仅附加 `-vn` 参数直接提取/转换。
- **4.2 NVENC (run_nvenc)**：用户指定 auto/nvidia 且准入支持。依赖 `-hwaccel cuda`，视频滤镜 `-vf "scale=trunc(iw/2)*2:trunc(ih/2)*2"`。质量参数 `-preset p4 -cq <CRF_VAL> -pix_fmt yuv420p`。绑定音频 `-c:a aac -b:a 128k`。
- **4.3 VAAPI (run_vaapi)**：依赖 `-hwaccel vaapi -hwaccel_device <节点> -hwaccel_output_format vaapi`。视频滤镜 `-vf "scale_vaapi=w=trunc(iw/2)*2:h=trunc(ih/2)*2"`。质量参数 `-qp <CRF_VAL>`。
- **4.4 CPU (run_cpu)**：无硬件前置。质量参数 `-crf <CRF_VAL> -preset fast -pix_fmt yuv420p`。

### 模块 5：生命周期与防脏数据机制 (Trap & Fallback Loop)
- **信号注册**：必须精准设置两种 Trap 行为以防止产生残缺视频文件：
  1. `trap 'cleanup; exit 130' SIGINT SIGTERM SIGHUP SIGQUIT`
  2. `trap cleanup EXIT`
- **清理逻辑**：`cleanup` 函数内检查 `[ -n "$CURRENT_OUT" ] &&[ -f "$CURRENT_OUT" ]`，存在则 `rm -f "$CURRENT_OUT"` 并输出警告日志。
- **主处理循环**：使用 `for i in "${!FILES[@]}"` 遍历数组。
  1. 动态获取/生成输出名（无扩展名补全 `.mp4`）。
  2. 使用 `success=0` 标志位。
  3. 执行 `纯音频逻辑 -> NVENC -> VAAPI -> CPU` 降级链。
  4. 每一级失败或回退前，若生成了无效文件，必须先 `rm -f "$CURRENT_OUT"` 清除痕迹。
- **结束暂停**：完成全队列后，若 `PAUSE_ON_EXIT=1`，使用 `read -n 1 -s -r -p` 并通过 `%s` 读取对应语言的提示语。

---
请仔细体会 i18n 字典映射与 FFmpeg 底层硬解参数的结合，直接输出最终一体化的高健壮性 Bash 脚本代码。
```
