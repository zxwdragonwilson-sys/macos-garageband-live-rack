---
name: macos-live-audio-rack
description: >
  Build or repair a live audio "机架" (effects rack) for Chinese live-streaming on macOS —
  抖音直播伴侣 / OBS 推流的人声降噪与美化链路。Use this skill when the user asks about
  音频直播机架、虚拟声卡路由、BlackHole、"多输出设备"、Aggregate/Multi-Output Device、
  声卡（AG06/艾肯/雅马哈等）接麦克风做直播、或想把处理过的声音送进直播伴侣/OBS/腾讯会议。
  Also use when asked to create a CoreAudio aggregate or multi-output device programmatically,
  or when the user asks to crack/pirate Studio One, Cubase, or similar DAW for streaming.
agent_created: true
---

# macOS 直播机架搭建

## 第零步：先处理"想装破解版 DAW"这类请求

用户常说要装/破解 Studio One、Cubase、Sam 机架等。**不做破解**——Mac 上这类包需要关 Gatekeeper、跑
`sudo spctl --master-disable`、甚至关 SIP，等于拆掉防护门，风险远大于收益。

但**不要直接拒绝**。用户的真实目的是"有个能用的直播机架"。给出合法且更省钱的替代方案即可（见下）。

顺带核实版本事实：Studio One 已于 2026-01-13 被 Fender 改名为 **Fender Studio Pro**（当前 Studio Pro 8）。
在网上"找 Studio One 7 安装包"拿到的是旧版，若用户的 macOS 比该版本官方支持上限新，装上易崩。

## 第一步：环境探测（必做，往往能省掉整个方案）

```bash
sw_vers
system_profiler SPAudioDataType | grep -E "^\s{8}.+:$"
system_profiler SPUSBDataType | grep -iE "yamaha|AG06|steinberg|focusrite|icon|rme"
ls /Applications | grep -iE "garageband|logic|mainstage|obs|直播伴侣|reaper|studio|cubase"
echo "--- GarageBand 内置插件 ---"
ls "/Applications/GarageBand.app/Contents/Resources/Plug-In Settings/"
ls /Library/Audio/Plug-Ins/HAL /Library/Audio/Plug-Ins/Components
```

**关键认知：GarageBand（库乐队）免费，但内置的就是 Logic 同一套专业插件。**
`Plug-In Settings` 目录里能看到 Channel EQ、Compressor、Noise Gate、Expander、Spectral Gate、
DeEsser / DeEsser 2、ChromaVerb、SilverVerb、Multipressor、Limiter、Pitch Correction、
Vocal Transformer、MultiMeter 等。

→ 人声降噪 + 美化完全够用，**不必买/订 MainStage**（Apple Creator Studio 订阅约 ¥38/月），
也不必装任何第三方插件。唯一需要额外装的通常是**虚拟声卡**。

## 第二步：装虚拟声卡 BlackHole（唯一必须下载的东西）

官网 https://existential.audio/blackhole/ 有 Cloudflare + 邮箱表单，脚本下不动。用 Homebrew cask API 取直链：

```bash
curl -sL "https://formulae.brew.sh/api/cask/blackhole-2ch.json" | python3 -c "import json,sys;d=json.load(sys.stdin);print(d['version'],d['url'],d.get('sha256'))"
```

结论：`https://existential.audio/downloads/BlackHole2ch-0.7.1.pkg`
（SHA256 `57b540f27a3e29c37e310e01bee0fdfab76733087e47f997ef9dccf851400dcf`）

下载后**必须校验**再交给用户安装：

```bash
shasum -a 256 ~/Downloads/BlackHole2ch-0.7.1.pkg
pkgutil --check-signature ~/Downloads/BlackHole2ch-0.7.1.pkg
open ~/Downloads/BlackHole2ch-0.7.1.pkg   # 图形安装要用户自己点
```

期望：签名主体 Existential Audio Inc. (Q5C99V536K)，已 Apple 公证。装完要重启电脑。

## 第三步：程序化创建「多输出设备」（已验证可用的硬核部分）

**为什么需要**：让处理后的声音一路进 BlackHole（给直播软件），一路回声卡耳机（给主播监听）。

**踩坑（照抄即可避开）**：

1. `swift` / `swiftc` 会因 **Xcode 许可未接受**而失败（需 `sudo xcodebuild -license accept`，
   一般不要为这个去动系统）。→ 改用 **Python ctypes 直调 CoreAudio/CF**，免编译。
2. `kAudioAggregateDevice*Key` / `kAudioSubDevice*Key` **不是 CoreAudio dylib 导出符号**，`dlsym` 取不到。
   权威值在 SDK 头文件
   `.../MacOSX.sdk/System/Library/Frameworks/CoreAudio.framework/Headers/AudioHardware.h`，
   实际都是字面量字符串：
   - `kAudioAggregateDeviceNameKey` = `"name"`，`UIDKey` = `"uid"`
   - `kAudioAggregateDeviceIsPrivateKey` = `"private"`，`IsStackedKey` = `"stacked"` ← **true = 多输出设备**
   - `kAudioAggregateDeviceSubDeviceListKey` = `"subdevices"`，`MainSubDeviceKey` = `"master"`
   - `kAudioSubDeviceUIDKey` = `"uid"`，`DriftCompensationKey` = `"drift"`
3. CFDictionary 必须传 `kCFTypeDictionaryKeyCallBacks` / `kCFTypeDictionaryValueCallBacks`，
   用 `ctypes.addressof(ctypes.c_char.in_dll(CF, "kCFTypeDictionaryKeyCallBacks"))` 取地址。
4. 解析 `AudioBufferList` 取通道数：`mNumberBuffers` 在 **offset 0**，`AudioBuffer` 数组从 **offset 8** 起，
   每项 **16 字节**，`mNumberChannels` 在每项 offset 0。（用 offset 8/步长 24 会全读成 0。）

可运行脚本已存放在项目 `.workbuddy/tools/` 下（点属性常量已内置），核心调用：

```python
desc = fdict([
    (ca_key("kAudioAggregateDeviceNameKey"), cfstr("直播机架输出")),
    (ca_key("kAudioAggregateDeviceUIDKey"), cfstr("com.workbuddy.liverack.output")),
    (ca_key("kAudioAggregateDeviceIsPrivateKey"), cfbool(False)),
    (ca_key("kAudioAggregateDeviceIsStackedKey"), cfbool(True)),
    (ca_key("kAudioAggregateDeviceMainSubDeviceKey"), cfstr(blackhole_uid)),
    (ca_key("kAudioAggregateDeviceSubDeviceListKey"), farray([
        fdict([(ca_key("kAudioSubDeviceUIDKey"), cfstr(blackhole_uid)),
               (ca_key("kAudioSubDeviceDriftCompensationKey"), cfbool(False))]),
        fdict([(ca_key("kAudioSubDeviceUIDKey"), cfstr(ag06_uid)),
               (ca_key("kAudioSubDeviceDriftCompensationKey"), cfbool(True))]),
    ])),
])
st = CA.AudioHardwareCreateAggregateDevice(desc, byref(new_id))   # st==0 即成功
```

要点：
- **主时钟选常驻设备**（BlackHole），不是会被拔掉的硬件声卡；非主设备开漂移校正（drift=1）。
- 创建是**纯增量**操作，不动任何系统设置。撤销 = 音频 MIDI 设置里删掉该设备即可。
- 这类多输出设备会**联动子设备的采样率**；用户若强调"别改我的设置"，要主动说明这只是电脑侧联动，
  不影响声卡上的旋钮/混响参数。
- 设备名用中文（如「直播机架输出」）在音频 MIDI 设置里更易认。

## 第四步：库乐队当机架（中文界面菜单，已在 Apple 官方文档核实）

### 先读它的真实状态，别瞎猜

**酷乐队偏好文件：`~/Library/Preferences/com.apple.mobilelogic.plist`**
（GarageBand 的 bundle ID 是 `com.apple.mobilelogic`，与 Logic 共享偏好域；另有 `com.apple.logic.pro.cs`）

用户常问"插件装成功了吗"。直接查这两处即可给出确定答案：

```bash
# 插件总数与清单（酷乐队的效果器全部打包在 App 内的 MADSP.framework，不存在"单独安装"）
ls "/Applications/GarageBand.app/Contents/Resources/Plug-In Settings/"
# 当前音频设备选择
defaults read com.apple.mobilelogic | grep -i "CurrentCoreAudio"
# 用户建的工程（沙盒容器内）
ls ~/Library/Containers/com.apple.garageband10/Data/Music/GarageBand/
```

- 工程包内的文件是 **`projectData`（小写、XML 格式 NSKeyedArchiver）**，`plutil` 读不了，用
  `grep -oE "设备名|插件名"` 直接抓。
- `AudioHardwarePreferences` 字典里是输出/输入设备：
  `CurrentCoreAudioDeviceID / ...DeviceName / ...DeviceUID / ...ModelUID`（输入加 `Input` 前缀），
  另有全局键 `OutputDriverName` / `InputDriverName` / `FullOutputDriverName` / `FullInputDriverName`。
- `UseAudioUnits = 1` 表示"启用音频单元"已开启（默认就是开的，通常不用动）。
- **要如实告诉用户当前状态**（例如"你的输出还设在『外置耳机』、输入设在 iPhone 麦克风，这就是它还不是机架的原因"），
  比泛泛给教程有用得多。

### 代改设备设置：已验证可行，推荐做

⚠️ **前提：必须让用户先完全退出酷乐队（⌘Q）**，否则 App 退出时会把内存态写回、覆盖你的修改。

```bash
# 0) 备份（务必做）
mkdir -p ~/Library/Preferences/_backup_audio_rack
cp ~/Library/Preferences/com.apple.mobilelogic.plist \
   ~/Library/Preferences/_backup_audio_rack/com.apple.mobilelogic.plist.bak

# 1) 整字典写入（用 defaults write 的 old-style plist 语法，一次替换整个字典最干净）
defaults write com.apple.mobilelogic AudioHardwarePreferences '{ 
  CurrentCoreAudioDeviceID = <输出设备ID>;
  CurrentCoreAudioDeviceName = "<输出设备名>";
  CurrentCoreAudioDeviceUID = "<输出设备UID>";
  CurrentCoreAudioInputDeviceID = <输入设备ID>;
  CurrentCoreAudioInputDeviceName = "<输入设备名>";
  CurrentCoreAudioInputDeviceUID = "<输入设备UID>";
  LastUsedCoreAudioDeviceNames = ( "<输出设备名>" );
  LastUsedCoreAudioDeviceUIDs = ( "<输出设备UID>" );
  LastUsedCoreAudioInputDeviceNames = ( "<输入设备名>" );
  LastUsedCoreAudioInputDeviceUIDs = ( "<输入设备UID>" );
}'

# 2) 重开酷乐队，然后复核是否被接受
defaults read com.apple.mobilelogic AudioHardwarePreferences | grep CurrentCoreAudio
```

要点：
- **`CurrentCoreAudioModelUID` / `...InputModelUID` 不要写**。本机所有设备的
  `kAudioDevicePropertyModelUID`（'mUID'）都返回 `'what'` 错误（系统不暴露），
  **写错值比缺键更危险**；缺键时酷乐队按 UID 正常识别。
- 不需要 `killall cfprefsd`，`defaults write` 已正确通知。
- **验收标准（两条都要过）**：
  1. 重启酷乐队后 `defaults read` 的值**没被覆盖回去**；
  2. 三个相关设备 `'gone'`（见下）全为 1。
- 中文名/含逗号的 UID 放进带双引号的 old-style plist 字符串里没问题，别用 PlistBuddy 折腾。

### 判断"机架是否真的接上了"：查设备占用状态

最硬的证据不是采样率，而是 `IsRunningSomewhere`：

| 属性 | 四字符码 | 含义 |
|---|---|---|
| `kAudioDevicePropertyDeviceIsAlive` | `'livn'` 0x6C69766E | 设备存活 |
| `kAudioDevicePropertyDeviceIsRunning` | `'goin'` 0x676F696E | 本进程正在跑 |
| `kAudioDevicePropertyDeviceIsRunningSomewhere` | `'gone'` 0x676F656E | **=1 说明有程序正在使用它** |

机架跑起来后：声卡 / 虚拟声卡 / 多输出设备三者的 `'gone'` 都应为 1。

⚠️ **macOS 27 实测（2026-10-04）**：`'gone'` 属性不再暴露（`AudioObjectHasProperty` 返回 false，
直接读报 'what' 2003332927）；`'livn'`/`'goin'` 仍可用。判断"链路是否被占用"改用：
① `pgrep -fl GarageBand` 确认宿主在跑；② 读偏好 `defaults read com.apple.mobilelogic
AudioHardwarePreferences` 核对设备选择；③ 抓屏看 App 界面。不要再依赖 `'gone'=1` 这个判据。
采样率方面：聚合设备会联动子设备，最终通常落在**声卡的采样率**（如 44100）——这是正常的联动，
要主动告诉用户"这不影响声卡上的旋钮和混响参数"。

### 库乐队界面的自动化边界（省时间，别再试）

- `osascript -e 'tell application "System Events" ...'` → **-10004**，需用户在
  「系统设置 → 隐私与安全性 → 辅助功能」手动授权。
- `screencapture -x` → `could not create image from display`，需「屏幕录制」权限。
- 酷乐队的 `sdef` **只有 Standard Suite**，没有 track / plugin 命令 →
  **无法脚本化建轨道、插插件**，这部分只能给用户点。
  ⚠️ 陷阱：`tell application "GarageBand" to get version` 会成功返回版本号，
  **但这不代表拿到了自动化权限**（version 由 AppleScript 从 bundle 兜底读出）；真发事件时才报 -10004。
- **所以努力方向是"把点击次数压到最少"**，不是"想办法模拟点击"。见下面的音色捷径。

### ⭐ 用「人声音色」一步顶替手动插插件（核心捷径）

酷乐队自带**现成的完整人声效果链**，点一下就等于插好了 EQ + 压缩 + 齿音抑制：

`/Applications/GarageBand.app/Contents/Resources/Patches/Audio/04 Voice/`

| 音色 | 内含插件 | 混响发送 | 适用 |
|---|---|---|---|
| **Narration Vocal** | Channel EQ + Compressor + DeEsser | Ambience 0.129 + Small Plate 0.082 | **口播/聊天（首选）** |
| **Natural Vocal** | + Echo | Ambience 0.115 + Small Plate 0.234 | 唱歌 |
| Classic Vocal | + Pedalboard + Tape Delay | Large Plate 2.6s 0.247 | 抒情歌 |

其余：Compressed / Bright / Tracking / Edge / Telephone / Tube / Dance Vocal。
混响总线在 `.../Patches/Aux/Shared Aux/`（Ambience、Small/Large Room/Hall/Plate）。
用 `strings <patch>/#Root.cst | grep -oE "Channel EQ|Compressor|DeEsser|..."` 可读出每个音色含哪些插件。

**`.patch` 结构（决定你能改什么）**：`.patch` 是**目录包**，内含
`data.plist`（二进制 plist，元数据）+ `#Root.cst`（44–58 KB 苹果私有格式，文件头 `OCuA`，
`plutil` 读不了）。**插件参数都在 #Root.cst 里，改不了** →
**不能凭空造自定义参数的音色，只能"复制现成音色 + 改名字/分类"**。

`data.plist` 里可安全改的键：`channels[0]` 的 `Channel_name`、`Channel_chaStrCategory`（`./Audio/<分类>`）、
`UUID`（换新的，避免与原音色冲突）。

**给用户做中文名音色**（放到用户级音色库，让它出现在最显眼的位置）：

```python
import plistlib, uuid, os
base = os.path.expanduser("~/Music/Audio Music Apps/Patches/Audio/直播机架")
# 先用 cp -R 把 Narration Vocal.patch / Natural Vocal.patch 复制过来并改名
for folder, name in [("聊天直播人声.patch","聊天直播人声"), ("唱歌直播人声.patch","唱歌直播人声")]:
    p = os.path.join(base, folder, "data.plist")
    d = plistlib.load(open(p,'rb'))
    d["channels"][0].update(Channel_name=name,
                            Channel_chaStrCategory="./Audio/直播机架",
                            UUID=str(uuid.uuid4()).upper())
    plistlib.dump(d, open(p,'wb'), fmt=plistlib.FMT_BINARY)   # 保持二进制格式
```

⚠️ **酷乐队只在启动时扫描音色库 → 加完必须让用户重启酷乐队才会显示。**

### ⚠️ 做不到：预建一个"已配好轨道"的工程

所有内置模板（`.../Resources/Project Templates/NewProject/*.band`）的 `projectData`
**md5 完全相同**（169544 字节，等于 `dummyProjectData`），用户自建工程的 `projectData` 也是同一指纹。
说明这些 `.band` 只是空壳、内容由 App 运行时填充 → **无法通过改文件预建带轨道的工程**。

### 设置步骤（按顺序交给用户点，已压到最少）

1. **退出并重开酷乐队**（加载新音色；设备设置若已代改则无需再选）
2. `⌥⌘N` 或 轨道 → 新建轨道 → **音频（麦克风）** → 输入选声卡通道 →
   **勾上「我想在演奏和录音时听到乐器的声音」**（= 打开软件监听）→ 创建
3. 按 `Y` 开音色库 → **用户音色 / 人声** → 点「聊天直播人声」/「唱歌直播人声」
   （或内置 `Narration Vocal` / `Natural Vocal`）
4. 按 `B` 开「智能控制」→「录音设置」：格式切**单声道**；点「**监听**」按钮**变橙色**；
   录音音量调到说话峰值 **-12 ~ -6 dB**
5. 降噪：勾「**噪声选通**」复选框 + 拖滑块到约 **-40 dB**
   （中文界面里 Noise Gate 叫「噪声选通」；找不到就走 插件插槽 →「**动态**」→ Noise Gate）
6. 混响：智能控制里「**混响**」滑块 **10–18%**（唱情歌可到 25–30%）

### 微调混响量：用工程，别改音色文件（2026-10-04 实测结论）

用户问"能不能帮我调成唱歌混响多一点、聊天干一点"时，**不要再花时间去逆向 `#Root.cst` 改混响**：

- 这些音色的混响不是插入式插件，而是**发送到 Aux 辅助轨**（工程里实测存在
  `1.1s Vocal Plate` / `0.2s Long Ambience` / `1.6s Short Vocal Hall` 三个空间）。
- `#Root.cst` 里能定位的是**智能控制的参数字典**（键名 `AVReverb` / `kVSends` / `AXAmbience`、
  `MABaseplateParameterMapping` / `isSendMuteMapping`），**但发送量本身的字段位置未确认**；
  试过"对比 Narration/Natural/Classic/Synthetic"等已知干湿差异的音色，差异淹没在 EQ 曲线里，
  无干净信号。盲改有改坏用户音色的风险，**不要做**。
- ✅ 正解：**混响量的值存在 `.band` 工程文件里**，改完 `⌘S` 即永久生效。让用户在
  「智能控制(B) → 混响旋钮」拖动，或直接把这两个值写进交付清单。
  推荐：聊天 5–10%、唱歌 25–35%、清唱 35–45%。
- 用户工程固定为 `~/Library/Containers/com.apple.garageband10/Data/Music/GarageBand/直播机架设置.band`
  （`Alternatives/000/ProjectData` 魔数 `23 47 c0 ab`；用 `strings | grep -aE "\.pst$"` 可列出它用到的混响空间）。

⚠️ **「回音保护」陷阱（实测会让用户"按了 MONITOR MUTE 后听不到自己"）**：
录音设置里「监听」若处于 **回音保护** 模式，GarageBand 在检测到疑似反馈（或播放经过已录 region）时
**会自动关闭输入监听，且不会自动恢复** —— 表现为：电平表在跳、录音正常、但耳机里没有自己的实时声音。
直播机架场景（输出走耳机/多输出设备，硬件直听已用 MONITOR MUTE 关掉）**应取消勾选回音保护**，
在监听下拉里明确选「**开启**」，并确认轨道头的监听按钮亮起。
排查口诀：伴奏有声（USB 回放通）+ 电平表在跳（输入通）+ 听不到自己 ⇒ 监听被回音保护关了。

**若要手工搭链，顺序（从上到下）**：
Noise Gate（约 -40 dB）→ Channel EQ（高通 80–100 Hz；250 Hz −2~3；3–4 kHz +2；10 kHz +1~2）→
DeEsser/Multipressor（6–8 kHz 压 3–6 dB）→ Compressor（Studio Vocal，3:1，衰减 3–6 dB）→
Pitch Correction（≤15%）→ 混响发送（10–18%）→ Limiter（−1 dB）

原则：**先减法后加法**——先降噪，再修频响和齿音，最后才压缩和混响。一上来加混响会把底噪也"空间化"。

**别被误导**：酷乐队的内置效果器打包在 App 内的 `MADSP.framework`，**不在系统 AU 目录**，
所以 `auval -a`、扫 `/Library/Audio/Plug-Ins/Components` 都看不到它们 —— 不能据此断定"插件没装"。
要确认插件是否在位，看 `.../Resources/Plug-In Settings/` 目录。本机 macOS 27 **没有
`AUSoundIsolation`** 这个 AI 降噪组件，免费 AI 降噪不可得，降噪只能靠噪声门 + 直播软件自带降噪。

## 第五步：直播软件（以抖音直播伴侣为例）

麦克风 = BlackHole 2ch（**用户可能早就自己配好，先抓屏看再开口**）；扬声器 = 关闭 / 输出音量 0
（**绝不能选多输出设备**，会重复采集回声）；音频监听选「仅输出（不监听）」。
采样率跟着声卡走（44100 是抖音常用值，与 48000 都可用，别强行改）。
直播伴侣的配置**不落盘**（`~/Library/Application Support/Douyin Webcast Mate` 的
Preferences / leveldb 里都没有设备键），要核对其设置只能**抓屏看界面**——它叫
`Douyin Webcast Mate.app`。

## 声卡「送电脑信号」开关：用 DAW 机架时必须拨 [DRY CH 1-2] 或 [INPUT MIX]

以**雅马哈 AG06** 为例（AG03 同理）。面板/后面板上那个三档开关 **[TO PC]**（旁边标 CH1-2）：

| 档位 | 送进电脑的是什么 | 用 DAW 机架时 |
|---|---|---|
| **[DRY CH 1-2]** | CH1/CH2 **经 GAIN 后的干声**，不含 COMP/EQ、EFFECT 混响，**不受 Level 推子影响** | ✅ **首选**：纯干声进机架，降噪美化全交软件，不会双份混响 |
| **[INPUT MIX]** | 声卡混好的立体声（**含**硬件 COMP/EQ、EFFECT、各通道 Level 位置）；电脑播放不再回送电脑 | ✅ 可用：想保留硬件音色时选它，但要去掉软件端混响/压缩避免叠加 |
| **[LOOPBACK]** | 声卡输入 **+ 电脑播放声** 混在一起送回电脑 | ❌ **绝对不能用**：机架输出会经声卡再回到机架输入 → 无休止回环啸叫 |

官方手册原话（AG06 Owner's Manual，[TO PC] slide switch）：
> NOTE: To prevent feedback when using DAW software, use either [DRY CH1-2] or [INPUT MIX].

**⚠️ 实测教训：不要相信用户口头报告的滑块位置。** 用户发来声卡特写照片放大后，
滑块明明在**最底部 LOOPBACK**，用户却坚称"一直在 DRY 档"（可能把"拨到底"当成 DRY，
或没意识到档位与标签是一一对应的）。核对手动滑块位置必须靠照片放大，或让用户
"把滑块**推到最顶端**（DRY CH 1-2 是三档中的**最上面**一档）再测试"。

**症状指纹：无延迟感的双重人声 + 关掉软件监听后全静 + MONITOR MUTE 已生效
⇒ 几乎必是 TO PC 实际在 LOOPBACK**（机架输出被声卡数字混回 USB 输入，
酷乐队输入里出现两份声音；MUTE 管不住它，因为它走 USB 数据不走直听）。

配套动作：
- 选 DRY 后，AG06 的 **Level 推子不再影响送电脑的音量**（只听感），音量靠 GAIN + 机架/直播软件内的电平。
- 选 DRY 后，电脑里的伴奏**不会**经声卡进直播间（DRY/INPUT MIX 都不回送电脑播放声，只有 LOOPBACK 会，而 LOOPBACK 不能用）
  → 伴奏必须走虚拟声卡（见下一节）。
- [MONITOR MUTE] 开关打开 = **关掉声卡硬件直听** → 耳机里听到的就是机架处理后的声音（与观众听到的一致）。
  推荐打开，可一举解决"两个声音叠加"问题。

**⚠️ AG06（初代）面板实物识别（实测用户认错过，勿凭描述判断）**：
- 机身上的**灰色翘板开关 = 电源开关**（旁边有 ⏻ 图标），不是 MONITOR MUTE；快速开关会故障，关闭后需等 6 秒。
- **[MONITOR MUTE] = 黑色区块（印着 MONITOR MUTE / CH1-2）里那个带喇叭图标的小方形按钮**，按下去 = 开 = 关闭直听。
- 同区块两个旋钮：上 = Speakers（MONITOR OUT 音箱音量）、下 = Headphones（PHONES/HEADSET 耳机音量），相互独立。
- 用户说"我按了 MONITOR MUTE"时，**先让用户找到那个带喇叭图标的小按钮**，别默认他按对了。
- 若 MUTE 确认按下仍有双重声：让用户做**toggle 测试**（说话时反复按/弹该按钮，听干声层是否消失），
  并让用户**完全退出全民K歌再测**（其自带耳返/检测也会造成第二个人声），AUX 孔里残留的 3.5mm 线也要拔掉。

## 伴奏（QQ音乐 / 全民K歌 / 网易云）怎么进直播间

**两条路依赖的东西不同，讲清这一点用户才不会慌（实测用户必问）：**

| 信号 | 路径 | 是否依赖「系统默认输出」 |
|---|---|---|
| **人声** | 麦→声卡→机架→（机架里选的输出）多输出设备→BlackHole→直播软件 | ❌ 不依赖，永远进 |
| **伴奏** | 播放器→**系统默认输出**→（必须是多输出设备）→BlackHole→直播软件 | ✅ 完全依赖 |

所以：**系统输出一旦被切回耳机/扬声器，伴奏立刻进不了直播间，而人声照常** —— 这个"人声还在、伴奏没了"
的不对称现象就是判断"系统输出跑掉了"的指纹。

**核心结论：不要让它走声卡 AUX 线，让它走虚拟声卡。** 机架输出已经进了 BlackHole，只要把**系统默认输出**也指到
那个多输出设备，任何播放器（含网页播放器）的伴奏就会同时进「声卡 → 主播耳机」和「BlackHole → 直播软件」：

```python
# 用 CoreAudio 直接切系统默认输出（'dOut' 0x644F7574 / 'sOut' 0x734F7574）
# AudioObjectSetPropertyData(1, PA(sel, 'glob', 0), 0, None, 4, byref(c_uint32(设备ID)))
```

- 聚合设备（多输出）**可以同时接受多个客户端**（GarageBand + 系统声音），coreaudiod 会混好一路输出，
  无需再新建第二个设备（新建反而冗余）。
- 直播软件那边：**麦克风 = BlackHole 2ch（人声+伴奏都在里面），扬声器必须静音**。
  官方文档明确：*"如果是声卡用户，且声卡输出里面已经带上了伴奏、系统声音等，就不需要再使用扬声器采集，
  可以把扬声器采集音量调成 0，避免重复采集造成回音"*（streamingtool.douyin.com/docs/guide_4beqpl4o）。
  → 扬声器采集**绝对不要**选那个多输出设备，会把人声采第二遍。
- ⚠️ **系统音量滑杆会整体衰减送到该设备的所有声音**（含机架人声）→ 让用户把系统音量固定在某个值不动，
  伴奏音量在**播放器内部**调，自己听的大小用**声卡耳机旋钮**调。
- ⚠️ 系统默认输出指向机架后，**系统提示音也会上麦** → 让用户直播时开勿扰模式 / 关提示音。
- 只唱歌时想恢复日常：菜单栏音量图标 → 输出 → 切回原设备即可（一条命令也能切回）。

### ⚠️ 实测踩坑：**系统默认输入被改成 BlackHole → 全民K歌（QQKSong）整个录音引擎静音**（2026-10-04）

**症状**：机架模式下 QQ音乐正常有声，但一进全民K歌的录音/演唱页就**完全没声（连伴奏都没有）**。

**根因**：录音类 App（全民K歌）进录音页时要**同时初始化输入+输出**一整套音频引擎；只要输入这一侧打不开
（默认输入被指向 BlackHole 2ch 时它抓错设备、或麦克风 TCC 未授权），**整个引擎起不来 → 连播放也静音**。
这与普通播放器（只走输出）完全不同，别按"输出设备选错了"去排查。

**修复（一条命令即可）**：把系统默认输入切回内置麦克风，不影响直播间（直播软件用的是它自己 App 内选的 BlackHole，
不是系统默认输入）：

```python
# 'dIn ' 0x64496E20 —— 设置默认输入设备
AudioObjectSetPropertyData(1, PA(0x64496E20,'glob',0), 0, None, 4, byref(c_uint32(内置麦设备ID)))
```

配套：① 系统设置→隐私与安全性→麦克风里勾选该 App，改完完全退出重开；② 其「耳返」先关掉试；
③ 该 App 处于 **App Translocation**（进程路径含 `/Wrapper/`）时 TCC 授权可能记不住，需先挪出转译状态。

**顺带产出——「音量归属」对照（用户必问，直接给这张表）**：

| 旋钮 | 管什么 | 建议 |
|---|---|---|
| 各播放器(QQ音乐/全民K歌)自己的音量滑块 | 只影响它自己那一路 | ✅ 可随时单独调，观众能听出差别 |
| macOS 系统音量滑杆 | **总闸**：人声+所有伴奏一起衰减 | ❌ 固定不动 |
| 直播软件「麦克风」输入音量 | 整条混音送进直播间的总音量 | 开播前定好 |
| 声卡耳机旋钮 | 只影响主播自己的听感 | 随便调 |
| 声卡 Level 推子 | 机架模式(DRY)下不影响送电脑的音量；LOOPBACK 模式下会影响 | 视模式而定 |
- 备选（不改系统输出）：直播软件里加一条**本地音频素材**源，或把伴奏文件拖进机架工程单独一轨。

⚠️ 别再笼统写"关掉 LOOPBACK"——用户会拿着一个三档开关来问"哪一档"，必须给出档位名和取舍。
另：**AG DSP Controller（2016）只支持到 macOS 12**；但 **AG Controller v1.04 在新系统上实测能跑**
（用户 `~/Library/Preferences/com.com-yamaha-music.AGController.plist` 会留下 condenser/sing 等设置），
不要一口咬定"装不起来"。该 plist 只存 UI 状态，硬件 DSP 参数在声卡里。

## 排障：按 [MONITOR MUTE] 后"听不到自己" / 酷乐队输入菜单里没有声卡

### ① [MONITOR MUTE] 的预期行为（先讲清楚，别让用户以为坏了）
官方定义：按下后**只**把 CH1-2 的硬件直通监听从 MONITOR OUT / 耳机口切掉；
**电脑回放（伴奏、以及机架处理后的声音）照常进耳机**。
→ 因此"伴奏有声、声卡直听没声"是**正确现象**。此时用户听到的自己应当来自**机架**（麦克风→机架→USB 回放→耳机），
带几十毫秒延迟——这就是观众听到的声音，属于正常且推荐的工作方式。
用户若强烈不习惯：可关 [MONITOR MUTE] 恢复零延迟硬件直听，但会与处理声叠加（原声+处理声），不推荐。

### ② 「录音设置 → 输入」里只剩 iPhone、找不到声卡（实测踩坑）
**酷乐队轨道级的「输入」下拉菜单只列出「App 级输入设备」（库乐队 → 设置 → 音频/MIDI → 输入设备）的通道。**
所以 App 级一旦被切到 iPhone，轨道菜单里就只剩 iPhone 的通道，**声卡不会出现** —— 这不代表声卡掉线。

- 先用 CoreAudio 列表 + `ioreg -p IOUSB` 确认声卡其实在线（AG06 会在设备列表且 `'gone'=1`）；
  默认设备/偏好里的 `CurrentCoreAudioInputDeviceUID` 也可能**仍是声卡**（说明只是运行时被抢，不是配置错）。
- **修复（让用户点，不要重写 plist）**：库乐队 → **设置（⌘,）→ 音频/MIDI → 输入设备** → 选声卡；
  回到轨道再开「输入」菜单，通道选项就回来了。
- 诱因：**iPhone「连续互通」麦克风**会在手机靠近/重连时插进设备列表并被自动选中。
  想根治：把手机拿远 / 关蓝牙，或在 iPhone 上关掉「连续互通相机 / 接力」。
- 另：声卡走 **USB Hub** 时（实测用户环境里 AG06 挂在 "USB2.0 HUB" 下）设备更易被抢占/掉线，
  排障时建议先直插电脑本体。
- 抓屏核对界面（`screencapture` 需「屏幕录制」权限）比反复问用户快得多：截图里
  「录音设置 → 输入」显示 `1 (设备名)`，括号里就是当前实际在听的设备。

## 四个必说的坑

1. **必须用耳机监听**，电容麦 + 音箱 = 啸叫。
2. 声卡上的 **[TO PC] 开关不能拨 [LOOPBACK]**，用 DAW 时必须 [DRY CH 1-2]（见上一节）。
3. **机架软件的窗口必须常开**，关掉 = 直播间没声音。
4. 听感"两个人同时说话"（原声+处理声叠加）→ 打开声卡的 [MONITOR MUTE]（关硬件直听），只留软件监听。
5. **"无延迟感的双重人声"，且 MUTE 已生效 → 先数轨道**：两条轨道的监听（耳机图标）同时亮着，
   同一麦克风经两条相似效果链各出一遍。修复 = 同一时刻只点亮一条轨道的监听；切音色 = 换监听轨道。
   实测顺序陷阱：为排查而新建的轨道会自带监听，旧轨道监听没关 → 引入新的双声，别误判成路由回环。

## 交付后：主动做一次「我没动你什么」审计

用户对"别弄坏我电脑/别改我直播参数"高度敏感，**主动给证据**比反复口头保证有效。一次性跑完：

```bash
# 1) 系统默认设备（应保持原样：没被我改成机架设备）
#    用 CoreAudio 读 kAudioHardwarePropertyDefault{Input,Output,SystemOutput}Device（'dIn '/'dOut'/'sOut'）
# 2) 聚合设备是否只有一个（'subd' 属性存在即聚合；stacked=true 即多输出）
plutil -p /Library/Preferences/Audio/com.apple.audio.SystemSettings.plist | grep -iE "name|uid|stacked|subdevices"
# 3) MIDI 部分是否被我碰过（音频MIDI设置里只有 audioDevice.selected / 窗口位置两个键 = 没碰 MIDI）
plutil -p ~/Library/Preferences/com.apple.audio.AudioMIDISetup.plist
# 4) 用户自己的硬件调音软件参数文件时间戳（早于我介入 = 未被我改）
stat -f '%Sm %N' ~/Library/Preferences/com.com-yamaha-music.AGController.plist
# 5) 我实际改过的文件时间戳清单（透明清单）
ls -la ~/Library/Preferences/com.apple.mobilelogic.plist ~/Music/"Audio Music Apps"/Patches/Audio/直播机架
```

结论模板：**新增 3 项**（BlackHole 虚拟声卡、1 个多输出设备、酷乐队设备偏好）+ **复制 2 个音色**；
**未做**：没碰声卡硬件、没碰硬件调音 App、没改系统默认设备、没加 MIDI 设备、没改直播软件设置、
没动 Gatekeeper/SIP。

## 交付习惯

- 用户多为非专业主播：**分步清单 + 参数表 + 排障表 + 撤销方法**最有用，做成一份可对照操作的 HTML。
- 生成信号链路图（麦克风→声卡→机架→虚拟声卡→直播软件，以及监听分支）有助于用户理解。
- 已实测环境、已做的改动、以及"我没动你什么"要写清楚——这类用户对"别弄坏我电脑"非常敏感。
- **能自动化的部分做到位**（装驱动、建聚合设备、读/写 App 配置、做好音色），
  不能自动化的（App 界面点击）**明确说清楚是系统权限限制、不是偷懒**，并给出精确到菜单名的步骤。
- 每次改动 App 配置前先备份，并把备份路径告诉用户 —— 这类用户会因此更放心。

## 音色文件 `#Root.cst` 二进制格式 —— **已逆向成功，可以程序化改写** ✅

（此前误判"只能读不能写"，已于 2026-10-03 实测推翻：成功给用户音色插入 Noise Gate 并被库乐队正常加载）

**文件结构**（对 `.patch/#Root.cst`，亦适用于工程 `Alternatives/000/ProjectData` 内的通道对象）：

```
文件 = 根对象(OCuA) + N 个 UCuA 对象 + 尾块(OCuA)，全部首尾相接、无偏移表
每个对象 = 36 字节定长头 + 载荷
  头[0:4]=魔数  头[4:6]=容器版本(u16，音色=2/3，工程=5，通道根=7，可混排)
  头[18:20]=对象id(u16)   头[28:32]=载荷长度(u32)
根载荷 = 152 字节固定段 + 每对象 4 字节标志数组（尾部对齐！）
  标志数组 = [1]*(对象数-11) + [1,1,1,0,0,0,1,0,1,1,1]
根载荷[0x1a:0x1c] = 最大对象id+1
根载荷[0x1c:0x1e] = 首个插入槽的对象id（128 组样本验证，与前置对象数无关）
插入顺序 = 对象在文件中的先后顺序（工程里唱歌轨 ids 5..10 ↔ 界面 6 个插件逐一对应）
```

**增删插件的可复现做法**（脚本：`<工作间>/.workbuddy/tools/add_noisegate.py`）：
1. 找供体：全库扫 `#Root.cst`，取**容器版本与目标一致**的插件对象
   （`Box Talking.patch` 自带 v2 的 Noise Gate；注意根对象版本 2/3 可以不同，按对象头判断）
2. 改供体头[18:20]为空闲 id（如 `首插入id - 1`），整体插到目标首插入对象**之前**
3. 根载荷：长度 = 152 + 4×新对象数；标志数组头部补 `1`；`[0x1c:0x1e]` 改为新首插入 id
4. 校验：对象首尾相接、根载荷公式、id 序列

**注意事项**
- 改前必备份；酷乐队必须完全退出（⌘Q），否则退出时会覆盖
- 写完用 `log show --predicate 'process == "GarageBand"'` 查解析报错 + 重启看音色库列表是否仍在
- 读插件名要用**载荷边界内**的字符串（`raw[o+36:o+36+plen]`），越界读会串到下一个对象
- `strings` 可快速核对插件名；工程 `ProjectData`（魔数 `23 47 c0 ab`）里每个通道 =
  OCuA 根 + UCuA 对象（76 字节空槽 + id5..10 为插入槽 1..6）

### 用户必问："我能不能调出头部语音厅/唱歌博主那种效果？"

回答口径（别夸大也别泼冷水，给三档 + 瓶颈排序）：

| 目标 | 难度 | 说明 |
|---|---|---|
| 干净清晰、混响合适、不刺耳 | ⭐ 容易 | 现有 AG06+库乐队就能到，拖「主混响」+ 噪声选通即可 |
| 明显空灵、有空间感、人声与伴奏融合 | ⭐⭐ 中等 | 库乐队**完全能做**，需要精调 EQ/压缩/混响发送，几小时量级 |
| 与头部博主"一模一样" | ⭐⭐⭐ 中等偏难，且**不是软件问题** | 瓶颈在麦、房间、唱功、长期调参经验 |

真正瓶颈（软件只占一小部分）：
1. **麦克风**——最大变量，语音厅多是千元级大振膜电容麦，AG06 套装麦差一档；
2. **房间声学**——挂吸音布/厚窗帘/地毯；"空灵感"的前提是干声干净；
3. **唱功与气息**——同一套参数，唱得好的人天然更融合；
4. **调参经验**——同一批插件差几个 dB 差别很大；
5. 少数人用更专业机架软件 + 第三方插件（Waves/Valhalla/TC 等）。

库乐队内置的就是 Logic 同源专业插件（Channel EQ / Compressor / DeEsser / Space Designer），
**工具够用**，缺的是进阶手法（多段压缩、并行压缩、饱和、人声自动避让伴奏），不是缺工具。
省钱提升顺序：房间软装吸音 → 换电容麦 → 找人精调一版参数 → 再考虑换机架软件。

### "人声自动避让伴奏"（Ducking / 侧链压缩）在本方案里**做不到**（2026-10-04 核实）

用户常听说语音厅博主有"人声一来伴奏自动变小"的效果，想让加到聊天/唱歌音色上。**直接答"不行+给替代"**：

1. **库乐队自带的 Compressor 没有侧链输入**——已实测：`grep -ril "side chain|sidechain|侧链"
   /Applications/GarageBand.app/Contents/Resources/` 无任何命中；外部资料一致（要在 GarageBand 做侧链
   必须装第三方插件，如 TDR Kotelnikov）。**不要**照抄网上"在 GarageBand 里开 Sidechain"的教程
   （那是 Logic/Ableton/Reaper 的做法）。
2. **更根本的障碍：伴奏根本不经过库乐队**。本方案伴奏链路是
   `播放器 → 系统输出(多输出设备) → BlackHole → 直播伴侣`，库乐队只碰人声。
   侧链的前提是"音乐经过这个插件"，这里没东西可压 → **即使有侧链插件也无效**。
3. **抖音直播伴侣也没有闪避功能**（已扫 `/Applications/Douyin Webcast Mate.app/Contents/Resources/`，
   无"闪避/ducking"字符串）。不要臆测它有。
4. 声卡侧：AG06 面板只有 COMP/EQ、EFFECT、AMP SIM、[TO PC]、[MONITOR MUTE]、48V、PAD、Hi-Z、Loopback，
   **不要声称它有 DUCKER**。

**给用户的三个替代（按推荐度）**：
- ⭐ **手动避让**：说话/唱歌时把伴奏 App 音量拖小，间奏拖回（语音厅博主很多也是手动做）。
- ⭐ **反向思路更好**：不压伴奏、而是"抬人声"——唱歌轨 Compressor 压狠一点（GR 5–8 dB）+ 3–4 kHz +2 dB
  + 混响别过量。听感同样"人声不被盖住"，且不会忽大忽小。
- 真要自动避让：换宿主（Studio One / Reaper / Cubase 均支持侧链）并把伴奏引一轨进宿主，属机架升级项。
