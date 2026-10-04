# macOS 直播机架 Skill（库乐队方案）🎙️

一个 [Agent Skill](https://agentskills.io)（SKILL.md 格式），让 AI 编程助手（WorkBuddy / Claude Code / Cursor / Gemini CLI 等）帮你在 **macOS 上零成本搭建一套中文直播机架**——人声降噪、美化、伴奏路由、抖音直播伴侣/OBS 对接，全部基于 **免费自带的库乐队（GarageBand）**，不需要破解 DAW、不需要付费插件。

> 适用场景：抖音直播伴侣 / OBS 推流、语音厅式聊天或唱歌直播、声卡用户（以雅马哈 AG06/AG03 为例，其他声卡同理）。

## 为什么用库乐队？

GarageBand 免费预装，但内置的是 **Logic 同一套专业插件**（Channel EQ、Compressor、Noise Gate、DeEsser、ChromaVerb、Multipressor 等）。做直播人声链路完全够用，本 skill 不依赖任何第三方付费插件，唯一需要额外安装的是开源虚拟声卡 [BlackHole](https://existential.audio/blackhole/)。

## 这个 Skill 能做什么

| 阶段 | 内容 |
|---|---|
| 环境探测 | 自动识别声卡型号、已装软件、GarageBand/插件状态，避免瞎折腾 |
| 虚拟声卡 | BlackHole 2ch 的直链下载、SHA256 与签名校验、安装指引 |
| 多输出设备 | 用 Python ctypes 直调 CoreAudio **程序化创建聚合（多输出）设备**，含踩坑记录（聚合设备键名、AudioBufferList 解析等） |
| 库乐队机架 | 读取/代写酷乐队设备偏好；内置「人声音色」捷径（Narration Vocal 等）一步搭好 EQ+压缩+齿音链；程序化生成中文名自定义音色 |
| 降噪美化参数 | 噪声选通 -40dB、高通 80–100Hz、DeEsser 6–8kHz、压缩 3:1、混响发送分场景参数表（聊天/唱歌/清唱） |
| 直播软件对接 | 抖音直播伴侣（Douyin Webcast Mate）的麦克风/扬声器正确选法、伴奏经系统默认输出进 BlackHole 的路由方案 |
| 排障指纹库 | LOOPBACK 啸叫、双重人声、回音保护吞监听、「人声在伴奏没了」、全民K歌引擎静音等实测症状→根因对照表 |
| 硬件开关 | AG06 的 [TO PC] 三档开关（DRY CH 1-2 / INPUT MIX / LOOPBACK）取舍与辨认陷阱、MONITOR MUTE 行为解释 |
| 透明审计 | 交付后一键「我没动你什么」审计命令，列出全部改动与备份位置 |

## 安装

把 `skills/macos-live-audio-rack/` 放进你的技能目录即可：

```bash
# WorkBuddy
cp -R skills/macos-live-audio-rack ~/.workbuddy/skills/

# Claude Code
mkdir -p ~/.claude/skills
cp -R skills/macos-live-audio-rack ~/.claude/skills/
```

然后对 AI 说一句：

> 帮我用库乐队搭一个直播机架，人声降噪美化，对接抖音直播伴侣

## 信号链路图

```
麦克风 ──> 声卡 [TO PC]=DRY CH 1-2 ──> USB ──> 库乐队（机架）
                                              │  Noise Gate → EQ → DeEsser → Compressor → 混响
                                              ▼
                                        多输出设备（聚合）
                                        ├─> 声卡 → 主播耳机（监听）
                                        └─> BlackHole 2ch ──> 直播伴侣「麦克风」
                                                
播放器（QQ音乐等）──> 系统默认输出 = 多输出设备 ──> BlackHole ──> 直播间（伴奏）
```

## 重要说明

- **不涉及任何破解**：本 skill 明确拒绝破解版 DAW（Studio One/Cubase/Sam 机架），并给出合法替代方案。
- **改动最小化**：纯增量操作（新增 1 个虚拟声卡 + 1 个聚合设备 + 库乐队偏好），附完整撤销方法与改动前备份流程。
- **已知边界**：库乐队不支持 AppleScript 深度自动化（无法脚本化建轨道/插插件），skill 中已给出把手动点击压到最少的分步清单；GarageBand 的 Compressor 无侧链输入，人声自动避让伴奏在本方案内做不到（skill 给出三个替代做法）。
- **实测环境**：macOS 27（Tahoe 之后版本），苹果 API 行为变化（如 `'gone'` 设备占用属性不再暴露）已记录在 skill 内。

## License

MIT
