
# BiSheng C — 受限子集概述

本项目使用 BiSheng C 的刻意受限子集。可用特性列在 §2 中；其他所有内容（泛型、traits、成员函数、`_Owned struct`、libcbs）都被禁止。

## 1. 什么是 BiSheng C（子集）

BiSheng C（BSC）是 **C 的超集**，通过 ownership + borrowing + safe zones 提供编译时内存安全。在此子集中，BSC 在普通 C 之上添加这些检查，而不引入额外的语言抽象。

- 源文件：`.cbs`（或使用 `-x bsc` 编译的 `.c`）
- 头文件：`.hbs`（或使用 `-x bsc` 编译的 `.h`）
- 编译器：clang 分支 — `clang -x bsc file.cbs -o output`

## 关键语法规则

`_Owned` 和 `_Borrow` 是**指针限定符** — 放在 **`*` 之后**，绝不能放在类型之前：

- 正确：`char *_Owned p = ...;`
- 正确：`const int *_Borrow r = &_Const x;`
- 错误：`_Owned char* p = ...;` — 不能编译
- 错误：`_Borrow int* r = ...;` — 不能编译

## 2. 此子集中可用

| 特性 | 关键字 | 技能 |
|---|---|---|
| Ownership | `_Owned`、`_ArrayElem`、`_Nullable`、`nullptr`、`__take_array_from_raw`、`__move_array_to_raw` | `ownership.md` |
| Borrowing | `_Borrow`、`_ArrayElem`、`&_Const`、`&_Mut` | `borrowing.md` |
| Safe zones | `_Safe`、`_Unsafe`（块/语句/表达式/声明） | `safe-zone.md` |
| Nullability | `_Nullable`、`_Nonnull`、`nullptr`、`-Wnullability-completeness` | `nullability.md` |
| 初始化分析 | `__attribute__((ensure_init))`、`__assume_initialized` | `initialization.md` |
| constexpr | `constexpr`、`_Static_assert`（无类型特性） | `constexpr.md` |
| libc 的混合模式 `_Safe` 重声明 | `_Safe size_t strlen(const char *_Borrow _ArrayElem);` | `safe-zone.md` §5 |
| 此子集下的设计模式 | 交换后释放、辅助函数分层、大结构体堆装箱、全局返回决策树 | `design.md` |
| OOM 容忍 API 模式 | 普通结构体上的 VALID/INVALID 状态 + `_Owned _ArrayElem _Nullable` 字段 | `invalid-sentinel.md` |
| BSC ↔ 标准 C 双编译（仅注释） | `bsc_compat.h` 宏垫片、`__bishengc` 守卫、按类型分配器包装器 | `c-to-bsc-annotation-only.md` |

## 3. 不在本子集中

以下 BSC 特性对此项目**禁用**。不要在计划、代码或设计建议中引入它们。

| 禁用 | 说明 |
|---|---|
| `_Owned struct` | 使用带有 `T *_Owned _ArrayElem _Nullable` 字段的普通结构体 + `name_free(struct T s)` 消费者函数。见 `ownership.md` §7.5。 |
| 泛型（`<T>`、`<T, int N>`、泛型结构体/联合体/类型定义） | 每个元素类型编写一个特化函数。 |
| 成员函数（`TypeName::method`、`this`、`This`） | 编写接受 `struct T *_Borrow s` 的自由函数。 |
| `_Trait` / `_Impl` / trait 指针 / vtable 分发 | 使用标签枚举 + switch，或每个具体类型一个函数。 |
| 运算符重载（`__attribute__((operator OP))`） | 使用命名的自由函数（`str_eq`、`str_cmp`）。 |
| 协程（`_Async`、`_Await`、`Future`、`Scheduler`） | 编写同步代码。 |
| libcbs（`bishengc_safety.hbs`、`Vec<T>`、`String`、`Option`、`Result`、`Rc`、`Cell` 等） | 围绕原始 `malloc`/`free` 自行编写包装器。 |
| 类型特性 / 基于特性的 `constexpr if` / 条件类型别名 | 使用 `#if`/`#ifdef` 和具体类型。 |

## 4. 技能索引

> 关于 ownership，见 `ownership.md` 技能
> 关于 borrowing，见 `borrowing.md` 技能
> 关于 nullability，见 `nullability.md` 技能
> 关于 safe zones，见 `safe-zone.md` 技能
> 关于初始化分析，见 `initialization.md` 技能
> 关于 constexpr（子集），见 `constexpr.md` 技能
> 关于此子集下的设计模式，见 `design.md` 技能
> 关于 OOM 容忍的 INVALID 哨兵模式，见 `invalid-sentinel.md` 技能
> 关于编译，见 `compile.md` 技能
> 关于常见陷阱，见 `bsc-common-mistakes` 技能
> 关于错误码和诊断，见 `errors.md` 技能
> 关于 LSP / clangd 查询，见 `lsp.md` 技能
> 关于双构建模式下的 C → BSC 注释（此子集中的主要移植工作流），见 `c-to-bsc-annotation-only.md` 技能

被禁用的特性（泛型、traits、成员函数、运算符重载、协程、`_Owned struct`、libcbs）没有专门的技能 — 每个的替换模式见上面的 §3。
