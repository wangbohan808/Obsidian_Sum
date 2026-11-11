---
type: 操作手册
status: 草稿
mastery: 1
tags:
  - 内容/编曲
  - 工具/LogicPro
created: "2026-11-11"
updated: "2026-11-11"
up: "[[00_项目MOC]]"
---

# Logic Pro 13 · 功能全图与上手路径

> **一句话核心：Logic 只需要先理解四件事——Track 是通道、Region 是内容、Event 是最小颗粒、声音要流到 Output 才听得见；其余两百个功能都是在给这四个概念加开关。**

> **本文定位**：功能地图 + 首周路径。不是教程替代品，是用来回答"这个功能属于哪一类、什么时候才值得学"。
> **环境实测**（2026-11-11）：Logic Pro 13.0（`/Applications/Logic Pro Creator Studio.app`，bundle id `com.apple.mobilelogic`）· Apple M5 / 16GB / 341GB 可用 · macOS 26.6.2 · 音色库**分两处**：本机 1.8GB（最小安装）+ 外接盘 62GB（完整），**Logic 当前只读本机那份** · 工程存于外接盘 `Logic_Projects`（6 个）。

---

## 零、开箱前必须处理的四件事

| # | 事项 | 为什么现在就做 |
|---|---|---|
| 1 | **把外接盘的完整音色库接上** | 完整库已在外接盘（62GB），但 Logic 读的是本机 1.8GB 精简版。**不接上，六成乐器点开是空的**。做法见 0.1 |
| 2 | **打开高级功能** | `Logic Pro → Settings → Advanced → Show Advanced Tools`，勾选 **Complete Features**。不勾，一半菜单是隐藏的，任何教程都对不上 |
| 3 | **音频设备与缓冲** | `Settings → Audio`：Output/Input 选你的声卡；I/O Buffer **录制 64–128，混音 256–512**。16GB M5 不需要为性能牺牲延迟 |
| 4 | **不要放 iCloud / 同步盘** | `.logicx` 是**包目录**（内含音频文件），同步工具会破坏它。工程一律存本地，如 `~/Music/Logic Projects/` |

### 0.1 音色库：完整版在外接盘，但 Logic 没在用它

**实测对比（2026-11-11）**

| 项 | 外接盘 `Logic_Library/` | 本机 `~/Music/` | 倍数 |
|---|---|---|---|
| 总大小 | **62 GB** | 1.8 GB | 34× |
| Apple Loops | 6.8 GB / **16253** 个 | 421 MB / 1588 个 | 10× |
| Samples | 52 GB / **36721** 个 | 1.1 GB / 1635 个 | 22× |
| Patches 预设 | 1.1 GB / **6770** 个 | 289 MB / 1850 个 | 3.7× |
| Impulse Responses | 897 MB | 14 MB | 64× |

**根因**：Logic 只从 **`~/Music/Logic Pro Library.bundle`** 这一个固定路径读库。把库放到 `/Volumes/Logic_FCP/Logic_Library/` 不会自动生效——那里现在是"囤着"，不是"在用"。

#### 方案 A：符号链接（推荐）

```bash
# 1. 本机精简库改名备份（不删）
mv ~/Music/"Logic Pro Library.bundle" ~/Music/"Logic Pro Library.bundle.mini.bak"

# 2. 建链接指向外接盘完整库
ln -s "/Volumes/Logic_FCP/Logic_Library/Logic Pro Library.bundle" ~/Music/"Logic Pro Library.bundle"

# 3. 验证（应显示 -> /Volumes/Logic_FCP/...）
ls -l ~/Music/"Logic Pro Library.bundle"
```

完成后**退出并重开 Logic**，再让它重建素材索引（Unified Browser 里刷新，首次几分钟）。

> 这与 Apple 官方 `Relocate Sound Library` 的机制等价：官方功能也是"移动库 + 原位留符号链接"。手工做的区别只是"指向已有的库"而不是"再搬一次"。

#### 五个必须知道的代价

| 风险 | 说明 | 应对 |
|---|---|---|
| **必须先挂盘** | 盘没挂载时链接悬空，工程打开报"找不到采样" | 你的工程本来就在外接盘，这个习惯已经有了 |
| **卷名不能变** | 出现同名卷时会挂成 `Logic_FCP 1`，链接失效 | 不要在磁盘工具里改卷名 |
| **运行中别拔盘** | 会崩，或素材索引损坏 | 退出 Logic 再推出磁盘 |
| **首次要重建索引** | Apple Loops / 素材索引基于旧库 | Browser 里刷新，等它扫完 |
| **备份先别删** | `Logic Pro Library.bundle.mini.bak` 占 1.8GB | 跑通一周、确认不缺东西再删 |

#### 方案 B：搬回本机（不想依赖外接盘时）

本机还有 339GB 可用，62GB 放得下。用 `ditto` 合并而不是覆盖：

```bash
ditto "/Volumes/Logic_FCP/Logic_Library/Logic Pro Library.bundle" "$HOME/Music/Logic Pro Library.bundle"
```

代价是占掉 62GB 内置空间，换来"永远不用插盘"。**如果 Logic 是你的主力工具，这个选择更省心。**

#### 方案 C：只把素材夹加进 Browser（补充，不替代 A/B）

13.0 的 Unified Browser 支持把任意文件夹加为搜索位置。可把 `/Volumes/Logic_FCP/Sono_Music` 加进去，让自制素材与官方库同框搜索。**但它不能替代 A/B**——Apple Loops、Patches、Samples 仍需走 bundle 路径。

> **不要用 `Relocate Sound Library` 处理这个场景**：它是"把当前库移到新位置"，而当前库是 1.8GB 精简版，移过去会和已有的 62GB 冲突。

### 关于 13.0 的 AI 助手（Intelligent Assistant）

**你现在用不了**，四个条件同时满足才行：Creator Studio 订阅 + Apple silicon + **macOS 27 或更高** + 英文环境。你的系统是 26.6.2，卡在第三条。

> 不要花时间找这个菜单——找不到是正常的。等系统升级后它会出现，且**它会改变你学 Logic 的顺序**（部分编排工作变成对话），现在先按手动路径学，不影响任何基础能力。

---

## 一、四个概念：整个软件的地基

### 1. 层级关系

```
Project（.logicx 包 = 一个文件夹）
└── Track（轨道 = 通道，决定"送到哪、用什么乐器、走什么效果"）
    └── Region（片段 = 时间轴上的矩形块，决定"什么时候响、响多久"）
        └── Event（音符 / 采样 = Region 内部的最小单位，在编辑器里改）
```

**必须分清的一句话**：
- **Track 管"声音从哪来"**（乐器、效果、输出目标）
- **Region 管"什么时候响"**（位置、长度、循环）
- 两者**解耦**——同一个 Region 可以拖到同类型的另一条轨上，音色立刻变但内容不变。这是 Logic 和剪映思维差异最大的地方：剪映里片段自带全部属性，Logic 里属性在轨道上。

### 2. 两套坐标（这是 Logic 独有的）

| 坐标 | 单位 | 用在哪 |
|---|---|---|
| **音乐时间** | 小节 : 拍 : tick（1/3840 拍） | 编曲、量化、和弦、循环长度 |
| **绝对时间** | 时 : 分 : 秒 : 帧（SMPTE） | 对画面、配乐、视频同步 |

顶部 LCD 显示区可切换；`File → Project Settings → Synchronization` 里设帧率。**做视频配乐时，把帧率设成和成片一致（25/30/60）**，否则对点会飘。

### 3. 信号链：声音怎么从轨道走到耳朵

```
音源（乐器/音频文件）
  → Insert 插入效果（串联，这条轨专属）
  → Fader 推子（音量）
  → Send 发送（并联，抽一份送到 Aux）
        → Aux 轨（通常挂混响/延迟，多轨共享）
  → Output（输出，通常 Stereo Out）
        → Master（主推子 + 总线效果 + 电平表）
```

**两个必须记住的判断**：
- **Insert** = 只改这条轨 → 压缩、EQ、失真放这里
- **Send → Aux** = 多条轨共享同一个空间 → 混响、延迟放这里（放 Insert 会让每条轨各算一遍，浪费且不融合）
- **Bus 压缩**：把鼓、贝斯都输出到同一个 Bus，在 Bus 上加压缩 → 它们会"粘"在一起。这是混音最有效的一招

### 4. 三个"看见就懂"的常识

- 轨道**从左到右是时间，从上到下是层**（无优先级概念，只有发声顺序）
- 播放头 = 你现在在哪；**Locator**（左右定位符）= 你要处理/循环的区间
- 红色 = 录音中，黄色 = Solo，蓝色高亮 = 选中

---

## 二、界面六大区域（13.0 布局）

| 区域                  | 位置      | 管什么                                                      | 快捷键           |
| ------------------- | ------- | -------------------------------------------------------- | ------------- |
| **Control Bar 控制条** | 顶部      | 播放/录音/循环/节拍器、LCD 时间与速度、总音量                               | —             |
| **Inspector 检查器**   | 左       | 选中 Track / Region 后在这里改参数                                | —             |
| **Unified Browser** | 左（可折叠）  | **13.0 新增**：Loops、乐器、效果、插件预设、Splice 素材统一搜索               | `Y`           |
| **Tracks Area 轨道区** | 中       | 主战场：排 Region、剪、拖、循环                                      | —             |
| **Editors 编辑器**     | 下方或独立窗  | Piano Roll（MIDI）、Audio、Score、Step Sequencer、Drum Machine | `Cmd+K`（钢琴卷帘） |
| **Mixer 混音台**       | 下 / 独立窗 | 通道条、推子、Send、Bus、Master                                   | `X`           |

外加两个**随时会用到**的：
- **Smart Controls**（`B`）：把当前音色最该调的 8 个参数抽成一屏宏旋钮，不会用合成器时的救命面板
- **Loop Browser**：**13.0 已并入 Unified Browser**，看旧教程找不到它是正常的

> **13.0 的 Unified Browser 怎么用**：支持自然语言搜索（输入"atmospheric pad""lo-fi kick"）、按标签筛选、找相似音色、也能把本机任意文件夹加为搜索位置。对配乐最实用的是：**把你自己攒的音效库文件夹加进去**，以后一个搜索框搞定。

---

## 三、功能全景：按流程分组，附"什么时候才学"

### A. 音源（声音从哪来）

| 功能 | 是什么 | 什么时候学 |
|---|---|---|
| **Software Instrument 轨** | 虚拟乐器轨，接收 MIDI | **Day 1** |
| **Audio 轨** | 录真实声音（人声、吉他） | **Day 1** |
| **Drummer** | 虚拟鼓手，选风格+复杂度自动生成，可编辑 | **Day 1**（最快出成品） |
| **Apple Loops** | 现成乐句，拖入自动匹配 BPM 与调性 | **Day 1** |
| **Session Player** | 键盘/贝斯/鼓手的"虚拟乐手"，比 Drummer 更懂和声进行 | Week 2 |
| **Quick Sampler** | 拖一个音频进去就变成可演奏乐器（做音效、采样） | Week 3 |
| **Sampler (EXS24)** | 多层采样器，映射多个音区与力度层 | 需要时 |
| **Alchemy** | 重量级合成器（加法/波表/颗粒/采样），Logic 最强音源 | Month 2+ |
| **Retro Synths** | ES2 / Retro Synth / Sculpture 等经典合成器 | Month 2+ |

### B. 编排

| 功能 | 是什么 | 什么时候学 |
|---|---|---|
| **Region 编辑** | 切割、合并、循环、复制、量化 | **Day 1** |
| **Folder / Folder Stack** | 把多轨装进一个"文件夹轨"做分组（可折叠、可整体处理） | Week 2（超过 10 轨就必需） |
| **Arrangement Marker** | 给段落打标记（Intro / Verse / Chorus），可整体拖拽改结构 | Week 2 |
| **Chord Track** | 和弦轨：写和弦进行，自动喂给 Session Player 与 Chord Pad | Week 3 |
| **Live Loops** | 格子式现场演出/拼接，适合快速试组合 | 需要快速实验时 |
| **Tempo Track / Tempo Set** | 速度自动化，可做渐变、突变 | 影视配乐会用到 |
| **Transposition / Signature** | 全局移调、拍号变化 | 需要时 |

### C. MIDI 编辑

| 功能 | 是什么 | 什么时候学 |
|---|---|---|
| **Piano Roll** | 钢琴卷帘，改音符音高/时值/力度 | **Day 1** |
| **Step Sequencer** | 步进音序器，做鼓组与节奏型最快（13.0 支持多选、复制、行选择、nudge） | Week 2 |
| **Drum Machine Designer** | 把多个采样拼成一套鼓，配 Step Sequencer 用 | Week 3 |
| **Score** | 五线谱视图 | 需要写谱时 |
| **Quantize / Groove Template** | 对齐到网格 / 借用某段录音的律动 | Week 2 |
| **Transform** | 批量变换 MIDI（移调、随机化、压缩力度） | 进阶 |
| **MIDI Draw / Automation** | 在钢琴卷帘里画控制器曲线（CC11 表情、CC1 调制） | Week 3 |

### D. 音频处理

| 功能 | 是什么 | 什么时候学 |
|---|---|---|
| **Flex Time** | 不改变音高的时间伸缩（修节奏） | Week 3 |
| **Flex Pitch** | 改音高（修音准，可手动或一键） | Week 3 |
| **Stem Splitter** | 把成品音频拆成人声/鼓/贝斯/其他（13.0 可把鼓再拆成 kick/snare/hihat/toms/cymbals） | 需要采样或重混时 |
| **Audio File Editor** | 破坏性编辑（去杂音、剪掉开头、反转） | 需要时 |
| **Pitch Shifter / Varispeed** | 变调 / 变速 | 需要时 |
| **Bounce in Place** | 把一条轨"烤"成音频，释放 CPU | Week 2（工程卡了就靠它） |

### E. 效果器（按用途记，别按名字记）

| 类别 | 主力插件 | 用它解决什么 |
|---|---|---|
| **EQ** | Channel EQ / Linear Phase EQ | 给每个乐器分配频段，去掉不需要的低频 |
| **动态** | Compressor / Limiter / Multipressor / DeEsser | 控制起伏、让声音"实"、压住齿音 |
| **混响** | ChromaVerb / Space Designer / PlatinumVerb | 造空间与远近（**放 Aux，用 Send 送**） |
| **延迟** | Stereo Delay / Tape Delay / Echo | 造宽度与重复，Tape Delay 更"暖" |
| **调制** | Chorus / Flanger / Phaser / Modulation Delay | 加宽、加动感 |
| **失真/箱模** | Overdrive / Amp Designer / Pedalboard | 吉他与"脏"的音色 |
| **音高** | Pitch Correction / Vocal Transformer | 修音、变声 |
| **总线/母带** | Compressor / Limiter / Loudness Meter | 最后一步，把整体做响、做稳 |

> **学习顺序原则**：**EQ → Compressor → Reverb（Send）→ Delay**。这四个能覆盖 80% 的混音需求，其余等具体问题出现再学。

### F. 混音

| 功能 | 说明 | 什么时候学 |
|---|---|---|
| **Channel Strip** | 一条通道的完整链路（Insert → Send → Pan → Fader → Output） | Week 2 |
| **Bus / Aux** | 多条轨汇到一条，统一处理 | Week 2 |
| **Send（Pre/Post Fader）** | 送多少信号去 Aux；Post 跟随推子，Pre 不跟随 | Week 3 |
| **Automation** | 参数随时间变化，5 种模式：**Off / Read / Touch / Latch / Write** | Week 3 |
| **VCA / Group** | 用一个推子控一组轨 | 轨多时 |
| **Metering** | 电平表：峰值、响度（LUFS） | 导出前 |

**Automation 五种模式的意义**（这是很多人卡住的点）：

| 模式 | 行为 | 用在哪 |
|---|---|---|
| Off | 记录但不起作用 | 临时关掉 |
| **Read** | 播放已记录的自动化 | 日常 |
| **Touch** | 按下去才覆盖，松手恢复原值 | 微调 |
| **Latch** | 按下去后一直保持新值 | 整体改 |
| **Write** | 一播放就覆盖（危险） | 精细重写 |

### G. 导出与交付

| 功能 | 用途 |
|---|---|
| **Bounce（File → Bounce → Project or Section）** | 导出成品：选范围（整工程 / 定位符区间） |
| **Export Selected Tracks** | 分轨导出（给混音师、给视频剪辑用） |
| **Export All Tracks as Audio Files** | 一次性导出所有轨 |
| **Project Alternatives** | 同一工程的多版本（`File → Project Alternatives`），**做版本管理不用复制文件** |
| **Import / Export XML（AAF/OMF）** | 与 Pro Tools / 剪辑软件交换 |

**导出参数（配乐交付用）**：

| 项 | 值 | 理由 |
|---|---|---|
| 格式 | WAV / AIFF，**24-bit** | 视频后期要无损 |
| 采样率 | **48kHz**（配视频）· 44.1kHz（纯音乐） | 视频行业标准是 48k |
| Normalize | **Off**（自己控电平）或 On（省事） | 开 Normalize 会改变你混好的动态 |
| Dither | 降到 16-bit 时才开 | 保持 24-bit 时关掉 |
| 范围 | 定位符区间 | 别把前后空白导进去 |

### H. 协作与外部

| 功能 | 说明 | 你要不要 |
|---|---|---|
| **Logic Remote（iPad/iPhone）** | 无线遥控推子、走带、乐器演奏 | 有 iPad 就装，很好用 |
| **Sidecar 控制（13.0 新增）** | iPad 作多点触控控制器：触发 Live Loops、调通道、控制乐器 | 有 iPad 再试 |
| **Play Surface（13.0 新增）** | 可调整大小的演奏面板，含带力度/调制/滑音的音乐键盘、鼓垫、note-repeat、跟随 Chord Track 的 Chord Pad | 没有 MIDI 键盘时的替代品，**值得先试** |

---

## 四、快捷键：只给高频且稳定的

> **完整列表不要背**，用 `Option+K` 打开 **Key Commands** 窗口，顶部有搜索框，输入英文动作名（如 `split`、`bounce`）就能查到并自定义。这是 Logic 最该先学会的一个元技能。

### 播放与定位

| 键 | 动作 |
|---|---|
| `Space` | 播放 / 停止 |
| `Return` | 回到工程开头 |
| `R` | 录音 |
| `C` | 循环（Cycle）开关 |
| `Z` | 缩放到适应窗口 |
| `,` / `.` | 后退 / 前进一格（拍） |

### Region 编辑

| 键 | 动作 |
|---|---|
| `Cmd + T` | 在播放头分割 Region |
| `Cmd + J` | 合并选中的 Region |
| `Cmd + R` | 重复（Repeat，按原长度接在后面） |
| `Cmd + D` | 复制（Duplicate） |
| `Option + 拖动` | 复制并移动 |
| `Option + 拖 Region 右下角` | 循环延展 |
| `Cmd + 点击空白` | 拉出 Marquee 选区（只处理一段） |
| `Delete` | 删除 |

### 工具与窗口

| 键 | 动作 |
|---|---|
| `T` | 工具菜单（选剪刀、铅笔、橡皮等） |
| `Esc` | 回到指针工具 |
| `Y` | Unified Browser |
| `X` | Mixer |
| `B` | Smart Controls |
| `Cmd + K` | 编辑器（Piano Roll） |
| `Option + K` | Key Commands 搜索 |

### 工程

| 键 | 动作 |
|---|---|
| `Cmd + S` | 保存 |
| `Cmd + Shift + S` | 另存为 |
| `Cmd + Z` | 撤销 |
| `Cmd + A` | 全选 |

---

## 五、13.0 新增功能与实际可用性

| 功能 | 内容 | 你能否用 |
|---|---|---|
| **Intelligent Assistant** | 自然语言搭鼓点、建人声轨、修工程 | ❌ 需 macOS 27+ 且订阅 |
| **Unified Browser** | 素材/乐器/效果/预设统一搜索，自然语言 + 自动打标签 | ✅ **优先用** |
| **Enhanced Stem Splitter** | 鼓组拆成 kick/snare/hihat/toms/cymbals | ✅ |
| **Liquid Glass 界面** | 半透明菜单、更高对比 | ✅ |
| **Sidecar 控制** | iPad 当多点触控控制器 | ✅（需 iPad） |
| **Play Surface** | 可缩放演奏面板：键盘/鼓垫/Chord Pad | ✅ **没 MIDI 键盘时先用它** |
| **Step Sequencer 多选** | 拖选/Shift 点选批量改力度、note repeat | ✅ Week 2 用 |
| **Splice 接入** | Browser 内直接搜 Splice 素材 | ✅（需账号） |

> ⚠️ **看教程的坑**：13.0 界面变了（Liquid Glass + Browser 合并），2024–2025 年的视频教程在视觉上会对不上。**功能逻辑没变**，跟着做没问题，只是找不到按钮。优先看 2026 年 11 月之后的内容。

---

## 六、第一次上手：90 分钟跑通一首 30 秒

目标不是好听，是**打通全链路**：音源 → 编排 → 编辑 → 混音 → 导出。

| 步 | 动作 | 时间 |
|---|---|---|
| 1 | `File → New`（选 Empty Project，采样率 48k）→ 设 BPM 与调号 | 3 min |
| 2 | 加一条 **Drummer 轨**：选风格（如 Pop/Rock 的某个鼓手），把复杂度/响度拉到中等 | 10 min |
| 3 | 加一条 **Software Instrument 轨**，选一个 Pad 或 Piano 音色 | 5 min |
| 4 | 打开 `Cmd+K`，用铅笔（`T`）画 4–8 小节的简单和弦（哪怕每小节一个音） | 15 min |
| 5 | 打开 Unified Browser（`Y`），搜 Apple Loops 里的 Bass，拖进来，**它会自动跟 BPM 和调** | 10 min |
| 6 | 循环播放，听三个层是否各在各的位置（低频/中频/高频别打架） | 10 min |
| 7 | 加一条 Aux 混响：某轨 Send 拧到 -12dB 左右，听"远近"变化 | 10 min |
| 8 | 拉 Locator 定义 8 小节 → `File → Bounce → Project or Section` → 24-bit WAV | 10 min |
| 9 | 导出后**用手机外放听一遍**——这是最有效的审美检查 | 5 min |

> **硬判据：第一次做超过 2 小时，说明在翻菜单而不是在做事——停下来，去看第七节的"不做什么"。**

---

## 七、现在不要碰的东西（防掉坑）

| 不要碰 | 为什么 |
|---|---|
| Alchemy / Sculpture / 复杂合成器 | 音色设计是 Month 2 的事，先用预设 |
| 母带链（多段压缩 + 立体声增强） | 先把平衡做对，做响留到最后 |
| 上百轨的模板 | 5–10 轨足够练判断力 |
| 第三方插件（Waves / FabFilter 等） | Logic 自带插件完全够用，买插件是逃避学习 |
| 抄"专业工程模板" | 别人的总线设置对你不成立，因为音源和审美都不同 |
| 追求"录音棚级"监听 | 16GB M5 + 一副 300 元以上的封闭耳机，足够练到"能交付" |

---

## 八、三周路径（每周只加一件事）

| 周 | 目标 | 具体 |
|---|---|---|
| **W1** | 能独立做完一条 30–60 秒 | 音源 + Region 编辑 + Bounce。每天做一条，不管好坏 |
| **W2** | 能改好一条 | Mixer + Bus + Send + Folder Stack + Step Sequencer + Bounce in Place |
| **W3** | 能交付 | Automation + Flex Pitch/Flex Time + 分轨导出 + Project Alternatives 版本管理 |

**每周一次的检验**：把成品给一个不听这类音乐的人，问"哪一段让你想跳过"。**记原话**，不要记"挺好"。

---

## 九、常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 点乐器没声音 | 外接盘没挂载（符号链接悬空）/ 库未接上 / 轨道没选中 / 没按 R | 先看 `ls -l ~/Music/"Logic Pro Library.bundle"` 是否指向外接盘且有效 |
| 播放时爆音、卡顿 | I/O Buffer 太小 / 插件太多 | 调到 256–512；用 Bounce in Place 释放 |
| 拖 Loop 进去速度和音乐不一致 | 工程 BPM 被 Loop 改了 | 关掉 `Smart Tempo` 的自动改速，或先锁 BPM |
| 找不到 Loop Browser | 13.0 已并入 Unified Browser | 按 `Y` |
| 导出的音频比工程里小 | Normalize 关闭 + 主输出没加 Limiter | 主输出加 Limiter，看 LUFS |
| 工程打不开 / 文件损坏 | 放在 iCloud 或同步盘 | 移回本地 |
| 快捷键和教程不一样 | 教程是旧版或不同键盘布局 | `Option+K` 自查 |

---

## 十、与库里笔记的关系

| 笔记 | 层次 |
|---|---|
| [[00_视听创作总览]] | 认知层：为什么学、审美与分层思想 |
| **本篇** | **工具层：Logic 的功能地图与首周路径** |
| [[04_切片剪辑最小操作链]] | 执行层：剪映/必剪切片流程（另一工具链） |
| [[05_剪映图层与渲染原理]] | 技术层：剪映渲染原理 |

---

*关联：[[00_视听创作总览]] · [[00_项目MOC]]*

*来源：2026-11-11 对话 + 本机实测（Logic Pro 13.0 / M5 / 16GB / macOS 26.6.2 / 音色库 1.8GB）+ Apple 2026-10-27 公告与 13.0 发布说明。*
*关键事实：13.0 于 2026-11-05 推送；Intelligent Assistant 需 macOS 27+ 与 Creator Studio 订阅；Loop Browser 与 Library 已合并为 Unified Browser；Stem Splitter 支持鼓组细分。*
