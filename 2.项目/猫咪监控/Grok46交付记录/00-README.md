---
type: 项目
status: 整理中
tags:
  - 技术/Linux
project: 猫咪监控
created: "2026-08-23"
updated: "2026-08-23"
---

# 虚拟机 → 宿主机：SSH 传输与猫咪监控交付记录

- **日期**：2026-08-23
- **Linux 虚拟机**：`wbh-robot` / `wbh808@`（Cursor Grok 4.6 工作区 `/home/wbh808/cat-monitor-grok46`）
- **Windows 宿主机**：`wangbohan\wangb`，VMnet8 `192.168.203.1`
- **本目录位置**：`D:\Github_Code\项目规划\猫咪饮食监控\Grok46-SSH传输与操作记录\`

| 文件 | 内容 |
| --- | --- |
| `01-SSH传输完整流程与原理.md` | 为何不用共享文件夹、密钥、authorized_keys、两种 Windows 管理员修法、scp/rsync |
| `02-猫咪监控Grok46交付摘要.md` | 文档 00–17 落地与 Arch 移植实产物 |

验证免密（虚拟机）：`ssh winhost "echo ok"` → 应打印 `ok`。
