---
type: 原子
status: 整理中
mastery: 2
tags:
  - 技术/工具
created: "2026-08-23"
updated: "2026-10-12"
up: "[[00_开发技能MOC]]"
---

# Agent工具

## 使用Agent的局限

- 如果有些操作必须进行断网处理：
1. agent本身操作是需要网络的，如果断开网络，agent操作直接终止
2. 操作者进行远程软件控制会断开

## 我的Agent工具使用

- Cursor处理代码生成任务
1. 默认打开Agent Windows
2. 在"New Agent"可以切换为"Editor Windows"

- workbuddy处理日常电脑任务->调用腾讯提供的工具
1. 如果处理指定位置的文件夹，需要选择文件夹作为工作空间，这个文件夹名称需要明确，与后续总结为skills功能需要对应
2. 如果不需要指定电脑文件，直接创建工作空间，总结skills

- trae work相当于workbuddy平替
1. trae work可以将工作流放在云端，这样一个账号，两台电脑可以共享工作记录、储存的skills

- deepseek处理开发板以及虚拟机任务
1. 开发板上安装cursor以及workbuddy不方便，使用codex + deepseek完成日常任务

## Skill 跨电脑迁移（A→B，经手机中转）

- 工具：TRAE Work（与 workbuddy 平替的 agent）
- 使用场景：
  - Skill 里含本地脚本、Python 环境等私有配置，不适合放云端共享（trae work 默认云端存 skill）
  - 只有电脑 A 装有该 Skill，B 也想用，且两台电脑账号同步不覆盖本地依赖
  - 想让 B 结合自己电脑环境重新安装，而非照搬 A 的配置
- 处理：用手机做中转，不必走云端

步骤：

1. 电脑 A：确认 Skill 已完成打包（含完整安装流程）
2. 电脑 A：在本机会话中打包发送（只能本地发送，无法跨机）
3. 电脑 A → 手机：把 Skill 文件下载到手机
4. 手机：切换到控制电脑 B
5. 电脑 B：选择"本地执行"
6. 电脑 B：把 Skill 文件加进会话命令框
7. 电脑 B：指示安装该 Skill，并让它结合 B 自身环境来装

## Claude Code

### 常用指令

- 给claude自动执行权限
```bash
claude --dangerously-skip-permissions
```

- 流畅安装claude一个可选项（开VPN，给命令行管理员权限）
```bash
irm https://daheiai.com/cc.ps1 | iex
```

### Skills 的使用

- 查看当前安装了哪些 Skills：在 Claude 中直接调用 `/skills`
- 调用 Skills：使用 `/skill 技能名 内容` 的写法，或直接说出对应 Skill 的触发关键词

```bash
/skill 技能名 这里写你要让技能处理的内容
```

## AI 使用复盘：视频号抓流（2026-08-27）

### 抓流原理（登录态 + 动态渲染平台的通用方法）
- 静态抓 HTML 拿不到地址（WebFetch 不执行 JS、无登录态）；视频流是前端 JS 请求 + 一次性签名直链
- 正确路径：让视频在"已登录的浏览器"里真正播放起来 → 浏览器必把真实地址写进 `<video>` → JS 读 DOM 即可
- 读地址：`document.querySelectorAll('video').map(v=>v.currentSrc)`
- 下载：`curl` 带 `Referer` + 正常 UA 直下；`ffprobe` 校验；`ffmpeg -c:a copy` 无损抽音频

### AI Agent 的局限
- 看不到视觉真值：只能读 DOM/文本，看画面需截图且须被允许读图
- 对登录态/加密/签名/动态渲染，静态抓取无能为力，必须借用户登录态或人工配合
- 强于确定性脚本，弱于异常状态排障（如账号切换死循环、第三方报错）
- 缺乏止损意识，倾向重复试错而非停下来澄清
- 上下文会丢，会重复做已排除的事

### 怎么用好（对操作者的要求）
- 懂一点技术：能及时纠错，也能在非常规场景下引导 AI 换方法
- 语言描述要精准；有歧义时 AI 应先澄清再动手
- AI 打转时果断打断纠偏
- 快速补上 AI 做不到的环节：扫码/登录/切账号/GUI 授权
- 给 AI 感官：允许读截图、装必要工具（完整 ffmpeg / python）
- 给出验收标准，别只给模糊目标
- 复杂/非常规需求，先与「优质模型」聊清方案，再让执行模型干
- 按任务选模型：需读页面视觉信息/看图时，别用无视觉模型（如 deepseek）

## 软件使用技巧

- Edge 浏览器无法进行数据同步——电脑开启 VPN，会导致一直停留在加载同步
- Obsidian 笔记无法观看目录——关闭文件后重新打开即可
