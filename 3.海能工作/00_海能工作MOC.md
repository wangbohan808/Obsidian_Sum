---
type: MOC
status: 整理中
up: "[[00_总索引]]"
tags:
  - 系统/索引
created: "2026-10-12"
updated: "2026-10-12"
---

# 海能工作 · 索引

> [!info] 本区定位
> 公司日常：**设备台账**（机型/固件/测试数据）、常用操作、待办与问题汇总。
> 机器相关的零散文件统一挂到 [[设备台账]]，用数据库视图汇总，不再靠文件名找。

## 固定入口

- [[设备台账]] —— 机型 × 固件 × 测试数据（数据库视图）
- [[0.海能常用操作]] —— 平台账号、三元组、OTA 等操作手册（**含明文口令，注意外泄**）
- [[1.海尔待办事项]] —— 未完成事项
- [[海尔项目问题汇总]] —— 问题与结论
- [[关键事件备忘录]] —— 时间线

## 全部笔记（自动）

```dataview
TABLE type AS 类型, status AS 状态, mastery AS 熟练度, updated AS 更新
FROM "3.海能工作"
WHERE file.name != this.file.name
SORT file.name ASC
```

## 待补

```dataview
LIST
FROM "3.海能工作"
WHERE status = "空壳"
```