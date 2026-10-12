---
type: MOC
status: 整理中
up: "[[00_总索引]]"
tags:
  - 系统/索引
created: "2026-10-12"
updated: "2026-10-12"
---

# 身体健康 · 索引

> [!info] 本区定位
> 运动、饮食、以及对身体这件事本身的探索。目标是"托底"——活得久、状态稳。

## 手动分组

### 训练
[[健身]] · [[骑行]]

### 饮食
[[烹饪探索]]

### 观察与探索
[[身体探索]] · [[人生探索]]

## 全部笔记（自动）

```dataview
TABLE status AS 状态, mastery AS 熟练度, updated AS 更新
FROM "5.身体健康"
WHERE file.name != this.file.name
SORT file.name ASC
```

## 待补

```dataview
LIST
FROM "5.身体健康"
WHERE status = "空壳"
```