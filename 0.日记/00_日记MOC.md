---
type: MOC
status: 整理中
up: "[[00_总索引]]"
tags:
  - 系统/索引
created: "2026-10-12"
updated: "2026-10-12"
---

# 日记 · 索引

> [!info] 本区定位
> 按日期命名的流水记录：当天待办、公司/个人任务、想法碎片。
> 日记是"原料"，不是"成品"——沉淀下来的结论应另起原子笔记（进 [[00_开发技能MOC]] 等分区），日记里留一个链接即可。

## 最近 30 天

```dataview
TABLE status AS 状态, updated AS 更新
FROM "0.日记"
WHERE file.name != this.file.name
SORT file.name DESC
LIMIT 30
```

## 全部日记

```dataview
LIST
FROM "0.日记"
WHERE file.name != this.file.name
SORT file.name ASC
```

## 从这里出发

- [[00_开发技能MOC]] —— 日记里"今天学到的"应该沉淀成技能笔记
- [[00_项目MOC]] —— 日记里的公司/个人任务属于哪个项目
- [[1.海尔待办事项]] —— 未完成的公司任务汇总