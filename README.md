# ~/bin 媒体工具箱

一组基于 **fzf + ffmpeg** 的命令行媒体处理脚本，编码参数数据库移植自 [FFmpegFreeUI (3FUI)](https://github.com/onlyvk/FFmpegFreeUI) v6，全部支持文件名含空格、ESC 取消、错误日志落盘。

## 目录

- [依赖](#依赖)
- [核心架构](#核心架构)
- [转码工具](#转码工具)
- [字幕工具](#字幕工具)
- [音频与图片](#音频与图片)
- [合并 / 混流 / 抽流](#合并--混流--抽流)
- [质量与分析](#质量与分析)
- [预设系统](#预设系统)
- [滤镜函数库速查](#滤镜函数库速查)
- [其他脚本](#其他脚本)
- [状态与日志](#状态与日志)
- [环境变量](#环境变量)

## 依赖

必备：`bash ffmpeg ffprobe fzf fd`

按需：`whisper-cli`（生成字幕）、`ollama`（翻译字幕）、`libplacebo/zscale`（超分与 HDR，编译进 ffmpeg）、`chafa`（预览）、`yay`（fzf-pkg）。

运行 `fzf-fftools` 主菜单选 `P. 环境诊断` 可一键检查全部依赖与编码器。

## 核心架构

所有媒体脚本共享一个函数库 [`fzf-fftools-lib`](fzf-fftools-lib)，提供：

| 模块 | 内容 |
|---|---|
| 选择器 | `pick_video / pick_videos / pick_audio / pick_image / pick_media / pick_files / pick_subtitle(s)`，全部带预览 |
| 编码器数据库 | `PSMAP`（预设参数）/ `QMAP`（质量参数），19 个编码器：x264/x265/SVT-AV1/aom-av1/rav1e/VP9/VP8/VVC(nvenc、qsv、vaapi、vulkan 硬件全家)、FFv1、ProRes |
| 编码器辅助 | `fftools_pick_encoder`（自动隐藏本机未编译的编码器）、`fftools_encoder_setup`、`pix_fmt`、`encoder_supports_2pass`、`fftools_advanced_args`（tune/无损）、`encode_2pass` |
| 滤镜库 | `fftools_denoise / sharpen / interpolate / tmix / grain / deband / deinterlace / transpose / mpdecimate / eq / superres`（参数源自 3FUI 滤镜面板） |
| 质量评测 | `fftools_metric_filter`（PSNR/SSIM/XPSNR/VMAF，支持 VMAF 模型与 pooling）、`fftools_vmaf_models`（自动发现本地模型） |
| 翻译引擎 | `translate_timed / translate_lrc / translate_txt`（Ollama 批量请求 + 逐条回退，`fzf-generate-subs` 与 `fzf-translate-subs` 共用） |
| 通用工具 | 进度条、运行日志、输出路径防覆盖、字幕路径转义、HDR 检测、硬件加速探测 |

## 转码工具

| 脚本 | 功能 |
|---|---|
| `fzf-fftools` | 工具箱主菜单，聚合下面所有脚本（A~R 子命令） |
| `fzf-convert-media` | 万能格式转换：视频/音频/图片自动识别，目标格式菜单（mp4/mkv/hevc/webm-vp9/avif/jxl/opus/flac…），输出重名自动追加序号 |
| `fzf-ffmpeg-batch` | 批量转码 + 编码队列：多选文件、统一编码器与硬件加速、HDR 自动 10-bit、重名输出防覆盖；`--queue` 排队逐个确认（可跳过/重试/全部自动），中途退出 `--resume` 继续，队列文件带 flock 锁 |
| `fzf-speed-video` | 变速（atempo 链，自动处理 0.5~2.0 范围外倍速） |
| `fzf-trim-video` | 剪辑：指定开始+持续，或**掐头去尾**（自动算保留区间）；无损拷贝或精确重编码 |
| `fzf-ffmpeg-builder` | 手动拼 ffmpeg 命令：裁剪/缩放/帧率 + 全部滤镜菜单 + tune/无损高级选项 + 硬件解码 + EBU R128 音量标准化 |
| `fzf-ffmpeg-color` | 色彩转换：HDR10→SDR（hable tonemap）、SDR→HDR10（含 master-display 写入）、bt709↔bt2020、仅写 HDR 元数据（bsf） |
| `fzf-ffmpeg-preset` | 预设快速压制：内置 3FUI 官方配方 + 用户预设目录，见[预设系统](#预设系统) |

## 字幕工具

| 脚本 | 功能 |
|---|---|
| `fzf-generate-subs` | Whisper 生成字幕：模型选择、VAD、翻译成目标语言、双语输出；支持 SRT/VTT/LRC/TXT；生成后可接烧录或封装 |
| `fzf-translate-subs` | AI 翻译已有字幕（本地 Ollama）：批量请求、逐条回退保证正确性、SRT/VTT/LRC/TXT、可选双语保留原文 |
| `fzf-burn-subs` | 烧录字幕进画面：字体库扫描推荐（中文字体加权评分）、字号选择、force_style |

## 音频与图片

| 脚本 | 功能 |
|---|---|
| `fzf-image-batch` | 批量图片（类 Converseen）：多选图片统一转换格式（jpg/webp/png/avif/jxl/tiff/bmp）+ 缩放（百分比/最长边/指定宽高），质量分档、输出防覆盖 |
| `fzf-audio-tools` | 批量：EBU R128 音量标准化（输出 m4a）/转 Opus/AAC/提取封面 |
| `fzf-mute-video` | 批量去音轨（流复制，不重编码） |
| `fzf-video-to-gif` | 高质量 GIF：palettegen/paletteuse 双通道、dither=bayer、尺寸与帧率可选 |
| `fzf-add-watermark` | 文字水印（drawtext，字体库推荐）或图片水印，多输出格式 |

## 合并 / 混流 / 抽流

| 脚本 | 功能 |
|---|---|
| `fzf-merge-av` | 自动扫描"视频+音频"分离文件配对合并：三级兜底（流复制→音频转码→完全重编码），临时文件防半成品误判 |
| `fzf-concat-video` | 多视频拼接：参数一致走无损 concat demuxer，不一致可选重编码 |
| `fzf-mux-media` | 多文件混流进一个容器：逐文件勾选流，章节/元数据独立控制 |
| `fzf-extract-streams` | 抽流：视频/音轨/字幕/附件（字体、封面），按编码器自动定扩展名，`|` 分隔解析不怕标题含逗号 |

## 质量与分析

| 脚本 | 功能 |
|---|---|
| `fzf-quality-eval` | 压缩前后对比打分：PSNR / SSIM / VMAF / XPSNR 或全测；VMAF 支持模型选择（自动发现 `/usr/share/model` 等目录的 .json）与 pooling（mean/调和平均/min）；支持起止时间限制 |
| `fzf-validate-media` | 批量验证文件完整性 |
| `fzf-preview-video` | fzf 预览钩子：视频抽帧、图片、音频封面（ffmpegthumbnailer + 内容哈希缓存） |

## 预设系统

内置配方（参数源自 3FUI 开发者内置预设）：

- **AV1 高压缩（N卡）** — av1_nvenc p7 cq36（RTX50 常规答案）
- **AV1 UHQ 超压（N卡）** — tune uhq + multipass fullres + spatial/temporal AQ
- **AV1 层次化B帧（新驱动）** — bf 31 + b_ref_mode hierarchical（再省 30%）
- **H.265 UHQ（N卡）** — hevc_nvenc p7 uhq cq28
- **SVT-AV1-HDR 平衡点** — crf36 + variance-boost + film-grain
- **H.265 通用推荐 / H.264 最兼容** — 软编日常档
- **FDK AAC M4A / AVIF 图片 / Windows 多尺寸 ICO / H.265 二次编码**

用户预设：`fzf-ffmpeg-preset --save` 查看格式说明并生成示例，然后编辑
`~/.config/fzf-fftools/presets/*.preset`：

```ini
desc=我的压制方案
args=-c:v libx265 -preset slow -crf 24 -pix_fmt yuv420p10le -c:a aac -b:a 192k
passes=1        # 或 2 = 二次编码
ext=mp4
decode=         # 可选，如 -hwaccel cuda
```

## 滤镜函数库速查

所有函数可直接 `source fzf-fftools-lib` 后在其他脚本里调用：

```bash
fftools_denoise    <hqdn3d|nlmeans|atadenoise|bm3d> <轻|中|强>   # 降噪
fftools_sharpen    <unsharp|cas> <轻|中|强>                      # 锐化
fftools_deband     <轻|中|强|gradfun>                            # 去色带
fftools_deinterlace <yadif|yadif_field|bwdif|ivtc>               # 去隔行/反胶卷
fftools_transpose  <顺时针|逆时针|180|水平|垂直>                  # 翻转旋转
fftools_mpdecimate <轻|强>                                       # 抽帧去重
fftools_eq         "亮度:对比度:饱和度:Gamma"                     # 调色，任意项可缺省
fftools_superres   <W:H> [算法] [shader路径]                     # libplacebo 超分
fftools_interpolate <fps> [mci|mcn]                             # minterpolate 补帧
fftools_tmix       <帧数>                                        # 动态模糊
fftools_grain      <轻|中|强>                                    # 胶片颗粒
build_atempo_chain <倍速>                                        # 变速滤镜链
```

## 其他脚本

| 脚本 | 功能 |
|---|---|
| `fzf-pkg` | 包管理工具箱：pacman/AUR 安装卸载、新闻、缓存清理、统计（yay） |
| `battery-monitor` | 电池监控：低电/满电通知（阈值可调，flag 文件在 XDG_RUNTIME_DIR） |
| `set-wallpapers` | swaybg 多显示器壁纸轮换 |
| `omarchy-show-done` | 任务完成提示 |

## 状态与日志

- 运行日志：`~/.local/state/fzf-fftools/logs/`（每次 ffmpeg 调用完整落盘）
- 任务日志：`~/.local/state/fzf-fftools/tasks.log`
- 编码队列：`~/.local/state/fzf-fftools/queue.tsv`（`fzf-ffmpeg-batch --list` 查看 / `--clear` 清空）
- 预览缓存：`~/.cache/fzf-media-preview`（主菜单 `Q` 可清理/按 30 天修剪）
- 字体缓存：`~/.cache/fzf-fftools/fonts.tsv`

## 环境变量

| 变量 | 说明 |
|---|---|
| `FZF_SUBTITLE_FONT_SIZE` / `FZF_SUBTITLE_FONT_NAME` / `FZF_SUBTITLE_FONTS_DIR` | 字幕默认样式 |
| `FZF_TRANSLATE_MODEL` | Ollama 翻译模型（默认 translategemma:12b） |
| `FZF_TRANSLATE_BATCH` | 批量翻译条数（默认 20） |
| `FZF_VMAF_THREADS` | libvmaf 线程数（0=自动） |
| `FZF_VMAF_MODEL` / `FZF_VMAF_POOL` | VMAF 模型路径 / pooling 方式（菜单会自动设置） |
