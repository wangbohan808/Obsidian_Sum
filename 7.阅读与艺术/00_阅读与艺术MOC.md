---
type: MOC
status: 整理中
up: "[[00_总索引]]"
tags:
  - 系统/索引
created: "2026-10-12"
updated: "2026-10-12"
---

# 阅读与艺术 · 索引

> [!info] 本区定位
> 经典阅读（思想）与音乐/演奏（艺术）。阅读类先记"原文—理解—行动"三层，空壳笔记作为**待写种子**保留。

## 手动分组

### 思想 · 经典
[[道德经]] · [[论语]] · [[易经]] · [[金刚经]] · [[鬼谷子]]

### 艺术 · 音乐
[[乐理]] · [[钢琴-天空之城]]

## 全部笔记（自动）

```dataview
TABLE type AS 类型, status AS 状态, mastery AS 熟练度, updated AS 更新
FROM "7.阅读与艺术"
WHERE file.name != this.file.name
SORT file.name ASC
```

## 待补（待写种子）

```dataview
LIST
FROM "7.阅读与艺术"
WHERE status = "空壳"
```