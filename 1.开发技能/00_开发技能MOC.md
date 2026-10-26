---
type: MOC
status: 整理中
up: "[[00_总索引]]"
tags:
  - 系统/索引
created: "2026-10-12"
updated: "2026-10-23"
---

# 开发技能 · 索引

> [!info] 本区定位
> 跨项目可复用的**原子技能**：环境搭建、工具使用、语言/原理笔记。
> 判定标准：**换一个项目还用得上**。只服务于某个具体项目的，放 [[00_项目MOC]]。

## 手动分组

### 环境与系统
[[Docker与WSL2环境]] · [[VMware Workstation Pro]] · [[Arch系统]] · [[Macbook探索使用]] · [[OrangePi4A标准工程]] · [[网络代理与换源]]

### 版本控制与协作
[[Github]]

### AI 与工具
[[Agent工具]] · [[ClaudeCode深度使用]] · [[TraeWork使用指南]] · [[AI编码工具的规则文件机制]]（CLAUDE.md / AGENTS.md / .cursor/rules 层级与合并语义） · [[AI视频工作流]] · [[AI作曲]]

### 端侧 AI 与异构计算
[[端侧AI双栈工具链]]（ROCK 5C · RKNN/RKLLM ↔ Dragon Q8B · QAIRT/QNN/Genie）

### 嵌入式与硬件
[[有刷直流电机驱动]] · [[Mac编译MCU]]

### 计算机基础与工具链
[[编译链接与库文件格式]]（.o/.a/.so/.lib/.dll/.elf/.hex/.bin · 链接期踩坑 · 裸机链接） · [[计算机网络]]

## 全部笔记（自动）

```dataview
TABLE type AS 类型, status AS 状态, mastery AS 熟练度, updated AS 更新
FROM "1.开发技能"
WHERE file.name != this.file.name
SORT mastery DESC, file.name ASC
```

## 待补

```dataview
LIST
FROM "1.开发技能"
WHERE status = "空壳"
```