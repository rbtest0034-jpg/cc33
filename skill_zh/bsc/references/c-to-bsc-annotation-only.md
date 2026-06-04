
# 仅注释双构建（受限子集）

## 何时使用此技能

**仅注释模式**意味着：你添加的每个 BSC 注释必须可通过充满 `#define` 的头文件擦除为空白（或同一性直通），使得相同的 `.c` / `.h` 源文件以两种方式编译：

- BSC clang 使用 `-x bsc` → 关键字保持其含义；所有权、借用和安全区域检查都运行。
- 标准 `clang -x c -std=c11` 或 `gcc -x c -std=c11` → 关键字通过 `bsc_compat.h` 擦除；文件是普通的 C99/C11。

在以下情况下使用此技能：

- 现有 C 项目已经以普通 C 方式构建，你想添加 BSC 安全检查，**而无需分叉代码库或使 BSC 工具链成为硬性构建依赖**。
- 你希望 CI 独立验证标准 C 路径。
- 你正在增量地采用 BSC，需要在迁移过程中每个文件继续作为普通 C 编译。

在此子集中，仅注释双构建是主要的移植工作流 — 没有单独的单向 `.c`/`.h` → `.cbs`/`.hbs` 翻译轨道，因为被禁用的特性（`_Owned struct`、泛型、成员函数、traits、libcbs）留下的 BSC 专属语法表面都已经不会双编译。

## 顶级规则

1. **同一源代码，两个编译器。** 一组 `.c` / `.h` 文件；唯一变化的是编译器调用。
2. **保留 `.c` / `.h` 扩展名。** 不要重命名为 `.cbs` / `.hbs` — 仅注释模式正是关于保持现有 C 文件名，使标准 C 构建不会遇到意外。
3. **只添加注释。** 不要重构代码，不要更改算法。添加 `_Owned` / `_Borrow` 是可以的；将结构体重写为 `_Owned struct` 是不行的 — 在此子集中被禁用，而且也不会双编译。
4. **你添加的每个注释必须可以宏擦除。** 如果一个构造不能通过 `bsc_compat.h` 变成无操作或同一性直通，你不能在这里使用它。
5. **受限子集已经禁止了那些不易擦除的模式。** 泛型（`<T>`）、成员函数（`T::m`）、`_Owned struct`、析构函数、traits、libcbs — 全部被禁用，全部在普通 C 中不可表示。子集和双构建模式相互加强。

## 编写注释的核心规则

以下两条规则指导每个注释决策。在扫描函数之前先内化它们。

**核心规则 1 — `_Safe` 是默认选择；`_Unsafe` 由编译器驱动。**

将每个函数标记为 `_Safe`。像编写完全 `_Safe` 代码一样编写函数体。仅当编译器拒绝你不能以其他方式修复的行时才添加 `_Unsafe`。准确包装违规语句（不带花括号的 `_Unsafe stmt;`，或可能最小的 `_Unsafe { ... }` 块）。`_Unsafe` 具有传染性 — 标记为 `_Unsafe` 的函数强制每个调用者进入 `_Unsafe` 上下文，因此将包装器放在函数体周围，而不是函数本身。内部有 `_Unsafe { ... }` 块的函数仍然是 `_Safe` 的；这正是此块的全部意义。

禁止的捷径：因为一行需要逃逸而标记整个函数为 `_Unsafe`；"以防万一"扩大 `_Unsafe` 块；在编译前防御性地添加 `_Unsafe`；使用 `_Unsafe` 静默借用检查器错误（借用错误意味着注释有误 — 修复注释）。

**核心规则 2 — 函数签名中的每个指针必须被注释。**

出现在函数声明、定义或参数列表中的每个指针必须带有 `_Owned`、`_Borrow`，或有意地保持原始并有结构性理由。签名中的裸 `T *` 永远不是"默认"或"待办" — 它是一个积极的主张，声称该指针是原始的。

决策树（应用于每个签名中的每个指针）：

1. 函数是否释放指针、将其传递出去或存储在长期拥有者中？ → `T *_Owned`。
2. 函数是否通过指针读取而不修改？ → `const T *_Borrow`。
3. 函数是否通过指针修改但不释放？ → `T *_Borrow`。
4. 它是输出双指针（`T **out`）、函数指针、不透明转换中介或用于算术的游标？ → 原始 `T *` 是正确的，附带一行注释解释原因。

如果 (1)–(4) 都不明显适用，你尚未完成对函数的分析。不要将指针作为占位符留空。注释必须在 `.h` 声明和 `.c` 定义中完全相同地出现 — 不匹配会产生类型错误或静默编码错误的所有权模型。

## 垫片：`bsc_compat.h`

项目的 `bsc_compat.h` 是仅注释模式的规范构件。在每个源文件/头文件的顶部包含它：

```c
#include "bsc_compat.h"
```

它基于 `__bishengc`（由 BSC clang 预定义）进行守卫：

- 在 BSC 下：头文件是无操作；每个 BSC 关键字保持其含义。
- 在标准 C 下：每个 BSC 限定符变为空；所有权转移内置函数（`__take_from_raw`、`__move_to_raw`、`__take_array_from_raw`、`__move_array_to_raw`）变为同一性宏；`nullptr` 为 pre-C23 编译器定义。

不要随意编辑垫片 — 它设定了此项目可以使用的 BSC 构造的契约。代码中引入的任何新 BSC 关键字必须添加到垫片中，否则标准 C 构建会崩溃。

## 仅注释 + 此子集中允许的内容

| 注释 / 构造 | 标准 C 下的宏行为 |
|---|---|
| `_Owned`、`_Borrow`、`_ArrayElem`、`_Nullable`、`_Nonnull`（在 `*` 之后） | 擦除为空；`T *_Owned _ArrayElem _Nullable` → `T *` |
| 函数声明/块/语句/表达式上的 `_Safe`、`_Unsafe` | 擦除；`_Safe void f(...)` → `void f(...)`；`_Unsafe { ... }` → `{ ... }` |
| `&_Mut x`、`&_Const x` | `_Mut` / `_Const` 擦除 → `& x`（普通取地址） |
| `nullptr` | 对于 pre-C23 定义为 `((void *)0)` |
| `__take_from_raw(p)`、`__move_to_raw(p)`、`__take_array_from_raw(p)`、`__move_array_to_raw(p)` | 同一性宏：`(p)` |
| 带有 `_Owned _ArrayElem _Nullable` 缓冲区字段的普通 C 结构体 | 字段限定符擦除 → 普通 C 结构体 |
| 按类型分配器包装器（`safe_calloc_T`、`safe_free_T`） | 注释擦除后的常规 C 函数 |
| `_Static_assert(...)` | 标准 C11 — 两种模式相同 |
| 参数上的 `__attribute__((ensure_init))` | Clang 属性；gcc / 普通 clang 未知但通常忽略。如果标准 C 编译器报错，在垫片中添加 `#define`。 |

## 仅注释 + 此子集中禁止的内容

| 构造 | 为什么禁止 |
|---|---|
| `safe_malloc<T>(val)`、`Vec<T>`、`Option<T>`、任何 `<T>` | 泛型语法 — 标准 C 无法解析，此子集中禁用。 |
| `_Owned struct S { ... }`、析构函数 `~S(S this)` | 此子集中禁用（`overview.md` §3）；普通 C 没有自动销毁的等价物。 |
| `TypeName::method(this, ...)` | 成员函数在此子集中禁用；标准 C 无法解析。 |
| `_Trait`、`_Impl` | 此子集中禁用；普通 C 没有等价物。 |
| `_Async`、`_Await`、`Future`、`Scheduler` | 此子集中禁用；普通 C 没有等价物。 |
| `#include "bishengc_safety.hbs"` | libcbs 在此子集中禁用；头文件在标准 C 下不存在。 |
| 运算符重载（`__attribute__((operator OP))`） | 此子集中禁用；标准 C 无法解析。 |
| `constexpr`（C23 关键字） | 仅当标准 C 构建也针对 C23 时才安全。使用 `-std=c11` 会被拒绝。改用 `static const` + `_Static_assert`。 |

## 按类型分配器包装器（libcbs 替代方案）

由于 `safe_malloc<T>` 被禁止，为每个缓冲区元素类型编写一对包装器。函数体包装原始的 `calloc` / `free` 加上两个内置函数，所有这些都正确双编译：

```c
/* 在头文件（.h）中。 */
_Safe char *_Owned _ArrayElem _Nullable safe_calloc_chars(size_t n);
_Safe void                              safe_free_chars  (char *_Owned _ArrayElem _Nullable buf);

/* 在源文件（.c）中。 */
_Safe char *_Owned _ArrayElem _Nullable safe_calloc_chars(size_t n) {
    char *raw = (char *)calloc(n, sizeof(char));
    if (raw == nullptr) return nullptr;
    _Unsafe { return __take_array_from_raw(raw); }
}

_Safe void safe_free_chars(char *_Owned _ArrayElem _Nullable buf) {
    if (buf == nullptr) return;
    _Unsafe {
        char *raw = __move_array_to_raw(buf);
        free(raw);
    }
}
```

在标准 C 构建下，经过垫片处理后：

```c
char * safe_calloc_chars(size_t n) {
    char *raw = (char *)calloc(n, sizeof(char));
    if (raw == ((void *)0)) return ((void *)0);
    { return (raw); }
}
```

— 一个完全有效的普通 C 函数。见 `ownership.md` §2 了解更广泛的分配器模式，以及 `invalid-sentinel.md` 了解这些包装器所服务的 INVALID 哨兵惯用法。

## `__bishengc` 守卫 — 仅当两条路径必须不同时

如果特定块确实需要在 BSC 和标准 C 下有不同的代码，显式守卫它：

```c
#ifdef __bishengc
    /* BSC 专属路径 — 例如 libc 函数的额外 _Safe 重声明。 */
    _Safe size_t strlen(const char *_Borrow _ArrayElem s);
#else
    /* 标准 C 路径 — 回退到 libc 声明。 */
#endif
```

仅当需要时才使用 `#ifdef __bishengc`；目标是尽可能保持两条路径相同。大多数代码根本不需要此守卫，因为垫片已经中和了每个 BSC 关键字。

## 项目布局

```
project/
├── bsc_compat.h        # 垫片
├── module.h            # 声明 — #include "bsc_compat.h"
├── module.c            # 定义 — #include "module.h"
├── test_module.c       # 执行两条路径的测试
└── Makefile            # 目标：BSC 构建 AND 标准 C 构建
```

每个 `.c` / `.h` 以 `#include "bsc_compat.h"` 开头。

## Makefile 模式

关键目标 — 两者编译相同的 `.c` / `.h` 源文件，一个用 BSC 编译器，一个用标准 C 编译器：

```makefile
BSC      := /path/to/bsc/bin/clang
BSCFLAGS := -x bsc -I.
CC       ?= clang
CFLAGS   := -x c -std=c11 -Wno-nullability-completeness -I.

check:                              # 每个源文件的 BSC 语法检查
	$(BSC) $(BSCFLAGS) -fsyntax-only file.c

test:                               # BSC 构建 + 运行 — 验证所有权 / 借用
	$(BSC) $(BSCFLAGS) $(SRCS) $(TEST_SRC) -o test.bin
	./test.bin

test-stdc:                          # 标准 C 构建 + 运行 — 证明相同源码也可编译
	$(CC) $(CFLAGS) $(SRCS) $(TEST_SRC) -o test-stdc.bin
	./test-stdc.bin                 # 源码也是有效的 C
```

`test` 和 `test-stdc` 在更改完成前必须都通过。`-Wno-nullability-completeness` 静默 clang 来自 BSC 限定符擦除为空的"缺少注释"警告 — 这是垫片的无害产物，而非真正的问题。

`-x bsc` 和 `-x c` 都会覆盖基于文件扩展名的语言检测，因此普通的 `.c` / `.h` 文件在两个目标中都能工作 — 不需要重命名为 `.cbs` / `.hbs`。

## 向现有 C 代码库添加注释（快速指南）

1. 将 `bsc_compat.h` 放入项目根目录。
2. 在每个 `.c` / `.h` 顶部添加 `#include "bsc_compat.h"`。
3. 为你分配的缓冲区元素类型编写按类型分配器包装器（`safe_calloc_T`、`safe_free_T`）。见 `ownership.md` §2。
4. 对于每个函数签名：按核心规则 2 对每个指针进行分类（`_Owned` / `_Borrow` / 带理由的原始）。应用注释。
5. 按核心规则 1 将每个函数标记为 `_Safe`。根据 BSC 编译器报告的错误，逐一在最少必需的语句上包装 `_Unsafe`。永远不要预先扩大 `_Unsafe`。
6. 对于从 `malloc` / `calloc` 赋值的每个结构体字段：注释为 `T *_Owned _ArrayElem _Nullable` 并应用 INVALID 哨兵模式（`invalid-sentinel.md`）。编写 `name_free(struct S s)` 消费者函数。
7. 运行 `make check && make test && make test-stdc`。依次修复每个的诊断。只有三个全部干净时才完成。

## 常见陷阱

1. **忘记在新文件顶部 `#include "bsc_compat.h"`。** 在 BSC 下编译通过；在标准 C 下失败，出现"未知类型名 `_Owned`"或"使用了未声明的标识符 `nullptr`"。
2. **在应该双编译的代码中使用仅 libcbs 的 API。** `safe_malloc<T>`、`Vec<T>`、`String`（libcbs 版本）、`Option<T>` — 在标准 C 中都不存在，在此子集中也都不存在。
3. **向代码添加新的 BSC 关键字而不扩展 `bsc_compat.h`。** 如果需要新的注释，先向 `bsc_compat.h` 添加相应的 `#define`，然后使用它。
4. **在 `CFLAGS` 为 `-std=c11` 时使用 `constexpr`。** `constexpr` 是 C23 的。要么提高 C 标准，要么避免此关键字。
5. **只运行 `make test` 和 `make test-stdc` 中的其中之一。** 两者编译相同的文件，但强制执行不同的不变量 — BSC 捕获所有权错误，标准 C 捕获意外的 BSC 专属构造。
6. **`_Owned` 和原始之间的直接 C 强制转换**（例如 `(T *_Owned)malloc(...)`）。即使在 `_Unsafe` 中 BSC 也禁止。使用 `__take_from_raw`（raw → `_Owned`）和 `__move_to_raw`（`_Owned` → raw）。这些在标准 C 下是同一性宏，因此相同的代码双编译。
7. **将 `.c` / `.h` 重命名为 `.cbs` / `.hbs`。** 这违背了仅注释模式的目的 — 许多 C 构建系统和外部工具依赖于扩展名。保留 `.c` / `.h`；使用 `-x bsc` / `-x c` 切换语言。

## 清单

- [ ] 所有源文件/头文件保留 `.c` / `.h` 扩展名 — 没有重命名为 `.cbs` / `.hbs`
- [ ] `bsc_compat.h` 存在于项目根目录，被每个源文件/头文件包含，且仅通过添加新的 BSC 关键字 `#define` 进行扩展
- [ ] 没有任何地方 `#include "bishengc_safety.hbs"`
- [ ] 没有泛型语法 `<T>` — `safe_malloc<T>` 被按类型的 `safe_calloc_T` 包装器替代
- [ ] 没有 `_Owned struct` / 析构函数 — 带有 `_Owned _ArrayElem` 缓冲区字段的普通结构体 + 手动 `name_free` 消费者（见 `ownership.md` §7.5）
- [ ] 没有成员函数语法（`T::m`）；只有自由函数
- [ ] 没有 `_Trait`、`_Impl`、`_Async`、`_Await`、运算符重载
- [ ] 如果 `CFLAGS` 是 `-std=c11` 或更旧，没有 `constexpr`
- [ ] 每个函数签名中的每个指针都被注释为 `_Owned` / `_Borrow` / 带理由的原始 — 见上面的核心规则 2
- [ ] 每个函数都是 `_Safe`，除非调用者确实需要它是 `_Unsafe`；每个 `_Unsafe { ... }` 块都是最小的 — 见上面的核心规则 1
- [ ] 在 `_Safe` 内部使用 `&_Mut x` / `&_Const x` 而不是普通的 `&x`
- [ ] 在 BSC 路径中使用 `nullptr` 而不是 `NULL`
- [ ] 在每个 raw → `_Owned` 转移处使用 `__take_from_raw` / `__take_array_from_raw`（而不是 C 强制转换）
- [ ] `make check` 通过（BSC 语法检查）
- [ ] `make test` 通过（BSC 构建 + 运行 — 所有权检查有效）
- [ ] `make test-stdc` 通过（标准 C 构建 + 运行 — 证明双构建可行）

> 关于所有权决策和按类型分配器模式，见 `ownership.md` 技能。
> 关于此双构建所依赖的 INVALID 哨兵设计模式，见 `invalid-sentinel.md` 技能。
> 关于受限子集特性列表和禁用特性，见 `overview.md` 技能。
> 关于常见陷阱和注释期间的修复方法，见 `bsc-common-mistakes` 技能。
