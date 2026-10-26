---
type: 原子
status: 整理中
mastery: 2
tags:
  - 技术/工具
  - 方法/AI
created: "2026-10-23"
updated: "2026-10-23"
up: "[[00_开发技能MOC]]"
---

# AI 编码工具的规则文件机制（层级记忆）

> [!info] 一句话结论
> 三家（Claude Code / Codex / Cursor）都有"多层规则文件 + 自动注入上下文"，骨架相同：**组织 → 团队 → 目录 → 个人，由宽到窄拼接，越靠后越优先**。
> 真正分歧的不是文件名，是**合并语义**：Claude Code 全量累加、Codex 就近覆盖 + 32 KiB 截断、Cursor 按 glob 条件挂载。同一个文件树在不同工具里的行为可以不一样。
>
> **核心判断：层级记忆是分发机制，不是一致性机制。** 它保证规则能到场，不保证规则正确、无冲突、被遵守。

## 零、先记住这张骨架

```
组织级   管理员强制（企业策略 / Team Rules）      不可被覆盖
   ↓
团队级   仓库根，进 git，走 PR 评审              团队共享
   ↓
目录级   子包 / 子目录，就近覆盖上层同类项        子域作用域
   ↓
个人级   ~/.claude、~/.codex、CLAUDE.local.md   只本机，不进库
   ↓
     合并成一份 system prompt
```

共同点只有一条：**所有层同时在场，靠"出现在 prompt 更靠后"来表达优先级**，没有机械的合并算法。

## 一、三家对比总表

| | Claude Code | Codex | Cursor |
|---|---|---|---|
| **团队文件** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md`，另有 `.claude/rules/*.md` | `AGENTS.md` | `.cursor/rules/*.mdc`（同时原生读 `AGENTS.md`） |
| **层数** | 6 处 | 3 段 | 4 类来源 |
| **合并语义** | **串联累加**，全部在场，无覆盖，冲突靠模型判断 | **串联 + 就近覆盖**，`AGENTS.override.md` 整文件替换，不合并 | **条件挂载**，不命中等于不在场 |
| **兄弟目录** | 子目录 `CLAUDE.md` **按需加载**（读到那里的文件才加载） | **只加载 根→cwd 这条链**，兄弟目录不加载 | 按 glob 匹配当前上下文里的文件 |
| **硬约束** | 单文件建议 ≤200 行 | **合计 32 KiB 上限**，超了停止加载 | `.cursor/rules/` 下 `.md` 后缀被忽略 |
| **读不读 AGENTS.md** | **不读**（要一行桥接） | 它就是 AGENTS.md | 原生读 |

## 二、Claude Code

### 六处位置

| 层 | 位置 | 共享给谁 |
|---|---|---|
| 企业策略（最高，管理员强制） | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md` · Linux `/etc/claude-code/CLAUDE.md` · Windows `C:\Program Files\ClaudeCode\CLAUDE.md` | 全组织 |
| 项目 | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队（进 git） |
| 项目 rules | `./.claude/rules/*.md`，可用 frontmatter `paths: src/api/**/*.ts` 按路径限定 | 团队（进 git） |
| 用户 | `~/.claude/CLAUDE.md` | 只自己 |
| 用户 rules | `~/.claude/rules/*.md` | 只自己 |
| 项目本地 | `./CLAUDE.local.md`（要自己加 `.gitignore`） | 只本机 |

⚠️ 官方文档内部对"enterprise 第一 vs local 最具体"的表述不完全一致。实操口径：**企业层是硬约束不可违背；其余按"越具体越优先"，`CLAUDE.local.md` 最具体**。真实顺序以 `/memory` 打印的已加载清单为准。

### 合并语义：串联，不是覆盖

从当前目录**向上遍历**，把所有找到的 `CLAUDE.md` 串联起来，上层先出现、靠近项目的后出现（优先级更高）。**不是覆盖**——所以两条规则矛盾时它不会报错，会自己挑一条，结果不可预测。

子目录的 `CLAUDE.md` 属于 Project scope（不是独立层级），**按需加载**：Claude 真去读那个子目录的文件时才加载，省 token。

### 导入

```md
# 项目记忆
概览见 @README.md

@AGENTS.md          ← 跨工具桥接就靠这一行

@~/.claude/my-prefs.md
```

- 支持相对/绝对/`@~/` 家目录路径
- 递归最多 4~5 跳（不同版本文档有 4/5 两种说法）
- 代码块和行内代码里**不解析**
- 注意：`@path` 导入是**全量加载**，只减少重复、不减少 token

### 两个坑

**坑一：不读 `AGENTS.md`。** 官方原话 “Claude Code reads CLAUDE.md, not AGENTS.md”。桥接方式二选一：

```md
# CLAUDE.md 第一行
@AGENTS.md
```

```bash
ln -s AGENTS.md CLAUDE.md     # Windows 需要管理员或开发者模式，建议改用上行 import
```

**坑二：`CLAUDE.local.md` 会被算作一个 `CLAUDE.md`。** 所以仓库里只有 `AGENTS.md` 时，某个同事随手建了个 local 文件存私人笔记 → 他就**再也收不到 `AGENTS.md`** 了。修法：`/config` 里把 Project instructions 设成 `claude-md-and-agents-md`。

## 三、Codex

### 指令链怎么拼

Codex 在**每次运行开始时**（TUI 里约等于每次新建会话）构建一次指令链：

| 范围 | 读什么 |
|---|---|
| 全局 | `$CODEX_HOME`（默认 `~/.codex`）下：有 `AGENTS.override.md` 就用它，否则 `AGENTS.md`。**只用第一个非空文件** |
| 项目 | 从 Git 根**向下走到 cwd**。每级目录按 `AGENTS.override.md` → `AGENTS.md` → `project_doc_fallback_filenames` 里列的名字查，**每目录最多一个文件** |
| 合并 | 根 → cwd 串联，空行分隔。**越靠后越优先** |

```toml
# ~/.codex/config.toml
project_doc_max_bytes = 65536            # 默认 32768（32 KiB）
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
```

### 三个特性（都很关键）

**`AGENTS.override.md` 是整文件替换，不是合并。** 同目录下只要 override 存在，`AGENTS.md` 根本不会被读。

**32 KiB 封顶，且静默。** 合计大小到了上限就不再追加文件，不报警。嵌套越深、写得越多，越容易把后半截规则写丢。

**链锚定在启动目录。** 在仓库根启动 Codex，`services/payments/AGENTS.md` 不在这条路径上——**根本不会加载**。"写了没 cd 下去 = 没写"。

其他：空文件静默跳过（0 字节的 `AGENTS.md` 视同不存在）；每次都重建，无缓存，改完必须重启会话。

## 四、Cursor

### 四类规则来源

| 来源 | 位置 | 作用范围 |
|---|---|---|
| Project Rules | `.cursor/rules/*.mdc`，进仓库 | 团队 |
| User Rules | Cursor 设置（Customize → Rules） | 个人，跨项目 |
| Team Rules | dashboard 配置，Team/Enterprise 计划 | 组织，优先级最高 |
| AGENTS.md | 仓库里的 markdown | 团队，纯 markdown 的简版方案 |

**`.cursor/rules/` 下的普通 `.md` 会被直接忽略**——因为它没有 frontmatter 可以表达 description/globs/alwaysApply。想要纯 markdown，就去用 `AGENTS.md`。

### `.mdc` 的三个字段就是四种模式

**没有 `type:` 字段，字段的组合就是类型**，这是最容易搞错的地方：

| alwaysApply | description | globs | 模式 | 什么时候生效 |
|---|---|---|---|---|
| `true` | — | — | Always | 每个会话都带上 |
| `false` | 空 | 有 | Apply to Specific Files | 匹配的文件进入上下文时 |
| `false` | 有 | 空 | Apply Intelligently | agent 读描述后自己判断相关 |
| `false` | 空 | 空 | **Manual** | **只有 `@规则名` 才生效** |

⚠️ 高亮最后一行：**写完正文、frontmatter 基本留空 = 永不自动触发**。规则写得再好、评审过了，也完全不起作用，而且没有任何提示。团队误写出这种组合的频率高于其他所有情况。

```md
---
globs: src/components/**/*.tsx
alwaysApply: false
---
组件用命名导出，不用 default。
```

Glob 语法：`*` 单段、`**` 递归、多模式逗号分隔（`docs/**/*.md, docs/**/*.mdx`）。

### 一个反直觉点

规则里写 `@migration-template.sql` **不会内联文件内容**，它只是个指针，agent 要用工具去读。**必须一直在场的东西，直接写进规则正文。**

### 嵌套 AGENTS.md

Cursor 的行为是**与父目录合并，更具体的优先**——和 Codex 的"就近覆盖 + 锚定 cwd"不同。

## 五、AGENTS.md 标准的裂痕

agents.md 官网 FAQ 说得很干净：「最近的 AGENTS.md 赢；用户对话指令覆盖一切」。但各家实际实现并不一致：

| 工具 | 实际行为 |
|---|---|
| Codex | 从根串联到 **cwd**，后面的覆盖前面的；`AGENTS.override.md` 整文件替换 |
| Cursor | 根 + 子目录**合并**，更具体的优先 |
| Amp / Warp | 按 agent **实际读到的文件**触发对应子树（惰性） |
| Zed | 只挑一个项目文件，无嵌套规则 |

**同一个文件树，不同工具能给出不同行为。** 还有一个细节分歧：Warp 要求全大写 `AGENTS.md`，Factory 同时接受 `agents.md`/`Agents.md`。稳妥写法统一用全大写。

## 六、三个静默失效与验证法

| 工具 | 静默失效 | 对策 |
|---|---|---|
| Claude Code | 只建了 `AGENTS.md` 没桥接；或 `CLAUDE.local.md` 顶掉了 AGENTS.md | `/memory` 看已加载清单 |
| Codex | 32 KiB 截断；启动目录不对导致子树规则没加载 | 让它「复述你当前收到的全部指令」 |
| Cursor | `.md` 后缀；三字段全空变成 Manual | 检查 `.mdc` 后缀 + 确认应用模式 |

**通用判据：规则生效与否，只信"它复述出来的"，不信"我写下去了"。**

## 七、边界：层级记忆为什么挡不住例外污染

见 [[规格漂移与例外管理]]。这里只记那条最关键的：

**层级记忆给的是空间绑定的作用域，不是文本绑定的。**

在 `stations/A-3/AGENTS.md` 里写「取料高度 10mm」，「仅限 A-3」这个限定是靠**文件位置**表达的，不是靠文字。位置在复制、迁移、或让 AI「参考 A-3」的时候**不会被一起带走**——那行一旦进到新工位，它头顶「我在 A-3 目录里」的前提就蒸发了，剩下只有一句不带条件的绝对陈述。

所以它**提供了看起来的作用域，但它是隐式的、不可搬运的**。位置不是作用域，文字才是。

层级记忆解决不了的三件事：

1. 不知道哪条是标准、哪条是例外 —— 没有 authority 字段，`alwaysApply: true` 和「A-3 特殊」在格式上平等
2. 不知道规则有没有真加载 —— 上面三个坑全是静默的
3. 不知道规则之间是否矛盾 —— 没有冲突检测

## 八、分工：什么放层级，什么放例外表

| 内容 | 放哪 | 理由 |
|---|---|---|
| 命令、路径、依赖、禁区 | 目录层级规则文件 | 是本目录**特有的事实**，天然绑定位置 |
| "这里为什么偏离标准" | `EXCEPTIONS.md` + `DEVIATION` 注释 | 需要**文字锚定**，不能靠位置 |
| 标准本身 | 仓库根一份，+ 契约测试 | 单一真相源 |

一句话：**层级只放"这个目录特有的事实"，绝不放"为什么这里不一样"。**

## 九、一页速查

```
Claude Code   ./CLAUDE.md | .claude/rules/*.md | ~/.claude/CLAUDE.md | CLAUDE.local.md
              串联累加·不覆盖·子目录按需加载·不读 AGENTS.md（用 @AGENTS.md 桥接）
              验证：/memory         初始化：/init

Codex         ~/.codex/AGENTS.md → Git根 → cwd 逐级（每级最多1个文件）
              AGENTS.override.md 整文件替换·32KiB 静默截断·锚定启动目录
              验证：让它复述指令链

Cursor        .cursor/rules/*.mdc（必须 .mdc）| User Rules | Team Rules | AGENTS.md
              frontmatter 组合决定四种模式·三字段全空=只能手动@
              验证：确认后缀 + 应用模式

通用          规则文件 ≤200 行；短期规则别进常驻文件；改完要重启会话
```

## 关联

- 上级：[[00_开发技能MOC]]
- 配套：[[规格漂移与例外管理]] —— 那篇讲"例外为什么会污染标准"（判断），本篇讲"规则怎么送到 AI 面前"（机制）
- 相关：[[ClaudeCode深度使用]] · [[Agent工具]] · [[TraeWork使用指南]]
- 相关：[[AI协作与能力增长]] §二「路线必须外化」—— 规则文件就是这句话的具体载体
- 来源：2026-10-23 关于"层级记忆机制"的整理。工具细节会随版本变，日期口径 **2026-10**
