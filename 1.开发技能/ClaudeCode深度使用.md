---
type: 原子
status: 整理中
mastery: 2
tags:
  - 技术/工具
created: "2026-06-26"
updated: "2026-06-26"
---

++++++++# API

**月之暗面：**
网址：[Kimi API 开放平台](https://platform.kimi.com/console/api-keys)
账号：18355202496
密钥：sk-TaKzLojsJBy00uIf1QSuugfIiXPFKWQoBYzp2HnQb6tuCCqI


**LongCat:**
网址：[LongCat AI](https://longcat.chat/platform/api_keys)
账号：18355202496
密钥：ak_2X54509TV6qZ2Le3BY1Tc9GJ9bP6d


# ClaudeCode深度使用

开VPN，管理员身份运行PowerShell：

```bash
irm https://daheiai.com/cc.ps1 | iex
```

**配置参考网站：**
[cc-switch/docs/user-manual/zh/1-getting-started/1.4-quickstart.md at main · farion1231/cc-switch](https://github.com/farion1231/cc-switch/blob/main/docs/user-manual/zh/1-getting-started/1.4-quickstart.md)

[cc-switch/docs/user-manual/zh/1-getting-started/1.2-installation.md at main · farion1231/cc-switch](https://github.com/farion1231/cc-switch/blob/main/docs/user-manual/zh/1-getting-started/1.2-installation.md)

[Release CC Switch v3.14.1 · farion1231/cc-switch](https://github.com/farion1231/cc-switch/releases/tag/v3.14.1)

[LongCat AI](https://longcat.chat/platform/api_keys)

[LongCat API开放平台快速开始 | API 文档](https://longcat.chat/platform/docs/zh/#anthropic-api-%E6%A0%BC%E5%BC%8F)

[Claude Code 配置 | API 文档](https://longcat.chat/platform/docs/zh/ClaudeCode.html#%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95)






## 📅2026-06-17


- 不用询问许可
```bash
claude --dangerously-skip-permissions
```



- 现在的AI，需要使用者十分清晰自己在干什么，AI做了什么；
- 我了解结构的MES项目，我的指令词十分准确到位，AI可以很快解决
- 我想要对一个未知的skills项目进行优化，因为我完全不知道其中的代码逻辑，所以纯自然语言描述修改；
- 因为描述方向太宽泛，AI自由修改debug、找方法花大量token以及时间，出一点错误拉不回来，越改越错；


- 新修改一个B站视频自动生成文稿的skills
