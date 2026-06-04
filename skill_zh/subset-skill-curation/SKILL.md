---
name: subset-skill-curation
description: 将 BSC 技能集合适配到受限子集的方法论。当 BSC 项目禁用某些特性（如无 _Owned struct、无泛型、无 libcbs）且技能文件需要反映这些限制时使用。不是 BSC 技术技能——而是技能维护流程。
---

# 子集技能策划

当 BSC 项目使用受限的语言子集时，技能集合必须反映该限制，以免 AI 提议已禁用的特性。本技能描述如何审计和适配技能文件。

## 0. 集合的结构

BSC 知识存储在一个 **`/bsc` 技能加上几个独立的同级技能**中：

- `bsc/SKILL.md` — 一个轻量级的**路由器**：内联的关键语法规则 + 一个**路由表**（任务/症状 → `references/<topic>.md`）。
- `bsc/references/*.md` — 每个主题一个**参考文档**（ownership、borrowing、safe-zone、nullability 等）。这些文档**没有 frontmatter 和 `description`**；技能选择器不会直接选中它们。只能通过阅读路由表访问。
- 独立的同级技能（如 `bsc-common-mistakes`、`subset-skill-curation`）— 单独的目录，有自己的 `description`，放在 `/bsc` 之外是因为它们基于不同的信号触发（编译错误；维护任务）。

这种结构改变了策划方式：每个主题单元通常是一个**参考文档 + 它的路由表行 + 它的 `references/overview.md` 行**，而不是选择器会落地的独立技能。必须保持同步的两个锚点是 `references/overview.md`（特性地图）和 `bsc/SKILL.md` 中的路由表。

## 1. 分类参考文档

对于每个 `references/<topic>.md`，选择最能消除混淆且侵入性最小的层级：

| 层级 | 条件 | 操作 |
|---|---|---|
| **删除** | 该主题的整个内容在子集中被禁用 | 删除 `references/<topic>.md`，删除其在 `bsc/SKILL.md` 中的路由表行，并确保在 `references/overview.md` §3 中列出被禁用的特性，附上一行替换说明。不需要"存根文档"—参考文档没有 `description`，所以 §3 + 缺失的路由行本身就承担了重定向功能。 |
| **横幅** | 文档有有用的核心内容，但某些示例依赖于已禁用的特性 | 保留正文；在顶部添加子集横幅（SUBSET BANNER），包含转换表（§3）。 |
| **外科手术式** | 只有个别示例或句子引用了已禁用的特性 | 仅编辑那些行；保持其他内容不变。 |

不要删除有有用核心内容的文档；也不要对大部分示例对子集不适用的文档进行外科手术式编辑。

## 2. 独立的同级技能

同级技能*确实*有自己的 `description`，所以存根（Stub）层级同样适用于它：

| 层级 | 条件 | 操作 |
|---|---|---|
| **存根** | 同级技能的整个内容被禁用，但你希望选择器落地在"不可用"答案上并附带替换指引 | 重写为存根：一个重定向段落；`description` 以"在受限子集中不可用。"开头 |
| **保留 / 横幅 / 外科手术式** | 与参考文档相同 | 如上所述 |

一个主题是作为独立技能（有自己的选择器触发器）还是折叠到 `/bsc` 中作为参考文档，这本身就是一个策划决策：只有当它在 `/bsc` 描述无法捕获的**不同信号**上触发（原始编译错误、会话结束时的维护任务）时才保持独立。否则将其折叠到 `references/` 中并添加一个路由行。

## 3. 子集横幅模板

放在横幅级参考文档的顶部，紧随其标题之后：

```markdown
## 子集横幅（SUBSET BANNER）— 先阅读

本项目 BSC 子集禁用了：<逗号分隔列表>。
当你在此文档中看到使用已禁用特性的模式时，请使用右侧列代替：

| 本文档中的旧模式 | 使用此替代 |
|---|---|
| `safe_malloc<T>(val)` | 项目本地的 `safe_calloc_T(n)` 包装器（见 `references/ownership.md` §2） |
| `_Owned struct S { ~S() {...} }` | 普通 `struct S` + `name_free(struct S s)` 消费者函数（见 `references/ownership.md` §7.5） |
| `Vec<T>` / `String` / `Option<T>` | 普通 struct + `_Owned _ArrayElem` 字段 + INVALID 哨兵模式（见 `references/invalid-sentinel.md`） |
| `TypeName::method(this, ...)` | 自由函数 `name_op(struct T *_Borrow s, ...)` |
| `_Trait` 分发 | 标签联合 + switch，或每个具体类型一个函数 |
| `#include "bishengc_safety.hbs"` | 围绕 `calloc`/`free` + `__take_array_from_raw` 自行编写包装器 |

本文档中所有其他内容（底层的 BSC 规则、诊断、安全区域机制、借用检查器行为）不变。
```

同级技能正文使用相同的横幅。在 `references/` 内部，同级链接使用相同目录（`references/ownership.md`）；从独立同级技能链接，则写为 `/bsc` `references/ownership.md`。只列出正文中实际出现的行。

## 4. 选择器表面审计

在合并的结构中，技能选择器只看到 `/bsc` 和每个独立同级技能的 `description`。`bsc/SKILL.md` 中的**路由表**是 `references/` 下所有内容的内部分发器。任何内容变更后，审计所有三个表面：

- **`/bsc` 描述** — 必须枚举足够的触发器（ownership、borrowing、safe zones、nullability、init、constexpr、compile、errors、LSP、design、dual-build），使得选择器为参考文档涵盖的每个任务加载它。不得宣传已禁用的特性。
- **路由表** — 每个现有的 `references/*.md` 恰好对应一行；没有指向已删除文档的行；没有针对已禁用特性的行（这些通过 `references/overview.md` §3 重定向，而不是路由表）。
- **同级 `description`** — 移除对禁用特性的引用，或标记"不在子集中"；存根同级以"在受限子集中不可用。"开头。

## 5. `references/overview.md` 作为特性地图锚点

`references/overview.md` 必须保持最新，作为"此子集中有什么、没有什么"的权威参考。它有两个表格：

- **可用**（§2）：关键字、特性以及涵盖每个特性的 `references/<topic>.md` 文档。
- **不在本子集中**（§3）：已禁用的特性和替换模式。

任何变更后，检查 §2 或 §3 是否需要更新行。已移除的主题应将其禁用特性列在 §3 中，附上一行重定向说明。发现的新能力（例如新的混合模式重声明模式）应出现在 §2 中，指向记录它的文档。

## 6. 变更的顺序

当在一次会话中适配整个集合时：

1. 首先更新 `references/overview.md` §2/§3 — 特性地图锚点。
2. 更新 `bsc/SKILL.md` 中的路由表以匹配参考文档集合（为新增文档添加行，为已移除文档删除行）。
3. 移除完全禁用的参考文档，然后为混合内容的文档添加子集横幅（SUBSET BANNER）。
4. 对参考文档进行外科手术式编辑。
5. 处理独立同级技能（为完全禁用的技能创建存根）。
6. 审计选择器表面（§4）— `/bsc` 描述、路由表、同级描述 — 作为最终步骤。

不要在同一个编辑过程中混合移除和外科手术式编辑——上下文切换会增加在横幅级文档中留下禁用示例或在路由表中留下无效行的可能性。
