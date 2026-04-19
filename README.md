# compressvideos

>这是个整活仓库。

把以下提示词发给你的AI以获取源代码：

```
你是一位拥有 20 年经验的资深 Linux 系统架构师和 Bash 脚本大师。请根据以下极其详尽的架构设计文档，使用 Bash 编写一款名为 `compressvideos` 的纯命令行批量视频压缩与转码工具。

# 核心约束
1. 语言：纯 Bash 脚本，首行必须为 `#!/usr/bin/env bash`，并在其后声明 `set -u` 以严格控制未定义变量。
2. 架构：必须完全遵循以下定义的模块、函数名映射规则及控制流。
3. 执行：按照单线程串行模式处理多文件队列。

---

## 第一部分：总体系统架构与配置状态机

**1. 默认配置变量**：
系统在初始化时需声明以下默认变量：
- `DEFAULT_CODEC="h264"`
- `DEFAULT_DEVICE="auto"`
- `DEFAULT_QUALITY=5`
并将其赋值给工作变量 `CODEC`, `DEVICE`, `QUALITY`。

**2. 标准化彩色日志器**：
定义四个输出函数，需使用 ANSI 转义码：
- `log_info` (蓝色 [INFO])
- `log_warn` (黄色 [WARN])
- `log_err`  (红色 [ERROR]，需重定向到 stderr `>&2`)
- `log_ok`   (绿色 [OK])

---

## 第二部分：模块化详细设计与底层实现规范

### 模块 1：参数解析与 CLI 路由
使用 `while [[ $# -gt 0 ]]` 和 `case` 进行参数解析。输入文件使用数组 `FILES=()`，输出文件使用数组 `OUT_FILES=()` 收集。
- **可选带参/无参混合判定**：对于 `-c|--codec` 和 `-d|--accel-device`，必须判断下一个参数 `$2` 是否存在且不以 `-` 开头（`! "$2" =~ ^-`）。如果是，则赋值；否则视为“无参查询行为”，设置对应的 ACTION 标志位（如 `ACTION_LIST_CODECS=1`），并在解析完成后立即执行查询并 `exit 0`。
- `-i|--input`：追加到 `FILES` 数组。不支持通配符自动展开时，未加选项的孤立参数（`*` 分支）也直接追加到 `FILES`。
- `-o|--output`：追加到 `OUT_FILES`。
- `-q|--quality`：限制参数只能是 1~10 的正则匹配 (`^([1-9]|10)$`)。
- `-p|--pause`：设置布尔值 `PAUSE_ON_EXIT=1`。
- **校验边界**：执行处理前，若 `OUT_FILES` 不为空，必须校验 `${#FILES[@]}` 是否等于 `${#OUT_FILES[@]}`，不等则报错退出。必须前置检查 `ffmpeg` 命令是否存在。

### 模块 2：硬件探测与挂载映射 (list_accel_devices / get_drm_device)
必须通过读取 Linux 底层文件节点来侦测物理设备：
- 遍历 `/sys/class/drm/renderD*`，读取 `device/driver/module` 软链接的 basename。
- 映射规则：`nvidia` / `nvidia_drm` -> nvidia 节点；`amdgpu` / `radeon` -> amdgpu 节点；`i915` / `xe` -> intel 节点。
- 构建 `/dev/dri/renderD[X]` 绝对路径以备挂载。如果找不到，需平滑回退。

### 模块 3：编码器兼容性探测引擎
调用 `ffmpeg -hide_banner -encoders 2>/dev/null`：
- `get_nvenc_codecs`：用 awk 匹配 `nvenc` 及 `^ V`，用 sed 去除 `_nvenc`。
- `get_vaapi_codecs`：匹配 `vaapi`，去除 `_vaapi`。
- `get_cpu_codecs`：直接 grep 检查 `libx264` (映射为 h264), `libx265` (hevc), `libvpx-vp9` (vp9), `libsvtav1` / `libaom-av1` (av1)。
提供 `is_codec_supported(device, codec)` 进行准入校验。

### 模块 4：核心处理管道与 FFmpeg 参数拼装规范
所有 ffmpeg 调用必须包含 `-y` 且通过 `</dev/null` 重定向标准输入，防止吞噬终端按键。

**4.0 辅助映射**：
- `get_crf_value(quality)`：用 case 语法将 1~10 映射为 38~11（步长 3，如 5->26）。

**4.1 智能媒体分流 (run_audio_only)**：
- 扩展名忽略大小写匹配 `mp3|wav|m4a|flac|aac|ogg|wma|opus`。
- 参数：`-vn`，直接复制音频，无其他滤镜。

**4.2 分支 A：NVENC (run_nvenc)**：
- 准入：用户设备为 auto 或 nvidia，且支持该 codec。需前置验证 `nvidia-smi` 或 `/dev/nvidia*` 存在。
- 参数：`-hwaccel cuda`，视频滤镜 `-vf "scale=trunc(iw/2)*2:trunc(ih/2)*2"`。
- 视频编码器：`${CODEC}_nvenc`。
- 质量：`-preset p4 -cq <CRF_VAL> -pix_fmt yuv420p`。绑定音频 `-c:a aac -b:a 128k`。

**4.3 分支 B：VAAPI (run_vaapi)**：
- 准入：NVENC 失败后，设备为 auto/amdgpu/intel 且探测到 `/dev/dri/` 节点。
- 参数：`-hwaccel vaapi -hwaccel_device <节点> -hwaccel_output_format vaapi`。
- 视频滤镜：`-vf "scale_vaapi=w=trunc(iw/2)*2:h=trunc(ih/2)*2"`。
- 视频编码器：`${CODEC}_vaapi`。质量：`-qp <CRF_VAL>`。绑定音频。

**4.4 分支 C：CPU 软编 (run_cpu)**：
- 准入：硬件加速全失败，设备为 auto/cpu。
- 视频滤镜：同 NVENC。质量：`-crf <CRF_VAL> -preset fast -pix_fmt yuv420p`。

### 模块 5：系统级进程保护与主循环控制
- **防脏数据机制 (trap cleanup)**：注册 `SIGINT` 和 `SIGTERM` 监听。回调函数需检查工作变量 `$CURRENT_OUT` 对应的文件是否存在，若存在则立刻 `rm -f`，并 `exit 130`。
- **主处理循环**：使用 `for i in "${!FILES[@]}"` 遍历：
  1. 判断源文件是否是标准文件。
  2. 动态生成 `$CURRENT_OUT`：如果提供了 `OUT_FILES`，则取对应索引的值，无扩展名自动补 `.mp4`；未提供则生成 `${f%.*}_compressed.mp4`。
  3. 执行 `NVENC -> VAAPI -> CPU` 的三级状态机尝试，使用 `success=0|1` 标志位控制流转。
  4. 每级回退前必须清理失败产生的遗留文件 `rm -f "$CURRENT_OUT"`。
- **结束暂停**：执行完毕且 `PAUSE_ON_EXIT=1` 时，使用 `read -n 1 -s -r -p` 等待用户按任意键。

---
严格按照上述架构、变量名设计和降级调度流程输出完整且可直接执行的 Bash 代码，不需要分段，只需提供最终的一体化脚本。
```
