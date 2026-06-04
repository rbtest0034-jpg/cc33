---
name: bsc
description: 在编写、编辑、编译、设计、调试或移植本项目受限子集中的 BiSheng C（BSC）代码时使用 — 文件为 .cbs/.hbs，或用 -x bsc 编译的 .c/.h。涵盖 _Owned 所有权和移动语义、_Borrow/&_Const/&_Mut 借用、_Safe/_Unsafe 安全区域、_Nullable 可空性、初始化分析、constexpr、编译和诊断标志、BSC-Exxxx 错误码、LSP/clangd 所有权查询、子集设计模式、INVALID 哨兵 OOM 模式以及 C↔BSC 仅注释双编译。路由到 references/ 下的按主题参考文档。
---

# BiSheng C（受限子集）

本项目使用 BiSheng C 的刻意受限子集：**仅 ownership + borrowing + safe zones**。泛型、traits、成员函数、运算符重载、协程、`_Owned struct` 和 libcbs 都**被禁止** — 每个的替换模式见 `references/overview.md` §3。

本技能是一个路由器。阅读 `references/` 下与你的任务匹配的按主题文档 — 不要一次性加载所有文档。

## 关键语法规则（每次阅读）

`_Owned` 和 `_Borrow` 是**指针限定符** — 它们放在 **`*` 之后**，绝不能放在类型之前。把它们想象成 `const`。

- 正确：`char *_Owned p = ...;`
- 正确：`const int *_Borrow r = &_Const x;`
- 错误：`_Owned char* p = ...;` — 不能编译（"未知类型名 '_Owned'"）
- 错误：`_Borrow int* r = ...;` — 不能编译

在返回 BSC 代码前自我检查：如果你写了 `_Owned T*` 或 `_Borrow T*`，重写为 `T *_Owned`。模式始终是 `T *_Owned`。

## 路由表 — 任务 / 症状 → 文档

| 当你…… | 阅读 |
|---|---|
| 获取可用与禁用子集特性的完整列表 | `references/overview.md` |
| 处理 `_Owned` 指针、移动语义、`_Owned _ArrayElem` 缓冲区、普通结构体 RAII、通过借用的释放与替换 | `references/ownership.md` |
| 引入借用 — `_Borrow`、`&_Const`、`&_Mut`、生命周期、冻结、NLL、`_Borrow _ArrayElem` | `references/borrowing.md` |
| 编写 `_Safe`/`_Unsafe` 代码、安全区域限制、libc 的混合模式 `_Safe` 重声明 | `references/safe-zone.md` |
| 处理 `_Nullable`/`_Nonnull`、空安全检查、`nullptr`、`-Wnullability-completeness` | `references/nullability.md` |
| 处理未初始化变量/字段初始化分析、`ensure_init`、`__assume_initialized` | `references/initialization.md` |
| 使用 `constexpr` / `_Static_assert`（此子集中无类型特性） | `references/constexpr.md` |
| 编译 .cbs、编译器路径/包含配置、诊断压制标志、双编译 | `references/compile.md` |
| 解码 `BSC-Exxxx` 编译器错误及其修复类别 | `references/errors.md` |
| 通过 LSP（clangd）检查所有权流/活跃范围/借用 | `references/lsp.md` |
| 设计新的 API、结构体或所有权流（在规划前加载） | `references/design.md` |
| 设计带有 `_Owned _ArrayElem _Nullable` 缓冲区的 OOM 容忍普通结构体 | `references/invalid-sentinel.md` |
| 注释现有 C 使其在 BSC 和标准 C 下都能编译 | `references/c-to-bsc-annotation-only.md` |

## 同级技能（单独维护 — 直接调用）

- **`bsc-common-mistakes`** — 当遇到所有权、借用、安全区域或可空性的编译错误或调试陷阱时自动触发。
- **`subset-skill-curation`** — 将此技能集合适配到受限子集的维护方法论（不是编码技能）。
