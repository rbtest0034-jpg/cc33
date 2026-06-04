
# BiSheng C 安全区域技能

## 内部有 `_Unsafe { ... }` 的函数仍然是 `_Safe`

`_Safe` 和 `_Unsafe { ... }` 并不冲突；`_Unsafe` 块的整个意义在于给函数一个局部逃逸。不要因为函数体需要一个或两个逃逸行就将其从 `_Safe` 降级为 `_Unsafe`。`_Unsafe` 具有传染性 — 使函数 `_Unsafe` 会强制每个调用者也成为 `_Unsafe`。

## `_Unsafe` 块必须最小化

只包装真正需要逃逸的语句，不要多也不要少。重新检查你放入 `_Unsafe { ... }` 的每一行：如果它在 `_Safe` 中编译良好，就把它移出来。

```c
// 糟糕 — 臃肿的 _Unsafe 块，只有赋值需要逃逸
_Safe void f(uint8_t *buf, size_t i, uint8_t v) {
    _Unsafe {
        if (i >= cap) return;
        validate(v);
        buf[i] = v;
        log("wrote byte");
    }
}

// 好 — 只有原始下标是 _Unsafe
_Safe void f(uint8_t *buf, size_t i, uint8_t v) {
    if (i >= cap) return;
    validate(v);
    _Unsafe buf[i] = v;
    log("wrote byte");
}
```

`_Unsafe stmt;`（单个语句，无花括号）通常是正确的形式。

## 1. 概述

指定编译器强制内存安全的代码区域。默认上下文是 `_Unsafe`（标准 C 兼容性）。使用 `_Safe` 选择性启用严格检查。

## 2. 语法

`_Safe` / `_Unsafe` 可以修饰：函数声明、函数定义、函数签名、函数指针、语句和带括号的表达式。

```c
_Safe int add(int a, int b) { return a + b; }  // 安全函数

void example(void) {
    _Safe { int x = 10; }         // 安全块
    _Safe int y = 1;              // 安全语句
}

_Safe void process(const int *_Borrow v) {
    _Unsafe { printf("%d\n", *v); }  // 不安全逃逸（printf 是可变参数）
    _Unsafe int c = 1;               // 不安全语句
    char d = _Unsafe((char)c);       // 不安全表达式
}
```

不能在以下位置使用 `_Safe`/`_Unsafe`：全局变量、函数外部的类型声明或 `typedef`（函数指针 typedef 除外）。

## 3. 限制（编译器强制）

在 `_Safe` 区域中：

### 指针操作
- **不允许 `&` 取地址** — 使用 `&_Const` 或 `&_Mut` 来获取借用。例外：取函数地址是允许的。
- **不允许原始指针解引用**（`*rawptr`、`rawptr->field`）— `_Owned` 和 `_Borrow` 指针解引用是可以的
- **不允许指针类别转换** — 不允许在 `_Owned`/`_Borrow`/原始指针之间转换，不允许指针到整数或整数到指针的转换。例外：`T *_Owned` 可以显式转换为 `void *_Owned`。
- **不允许指向不同类型之间的强制转换**
- **需要使用 `nullptr`** — 在安全区域中禁止使用 `NULL`；使用 `nullptr` 初始化或比较指针

#### 指针转换矩阵

| 转换 | 在 `_Safe` 中 | 在 `_Unsafe` 中 |
|---|---|---|
| `T *_Borrow` → `void *_Borrow`（T 是普通数据：没有指针字段） | **可以**（隐式） | 可以 |
| `T *_Borrow` → `void *_Borrow`（T 有指针字段） | **禁止**（即使使用显式强制转换） | 禁止 |
| `void *_Borrow` → `T *_Borrow` | **禁止** — 需要显式强制转换，必须在 `_Unsafe` 中 | 可以（显式强制转换） |
| `T *_Borrow _ArrayElem` → `T *_Borrow` | 可以（隐式） | 可以 |
| `T *_Borrow` → `T *_Borrow _ArrayElem` | 禁止 | 禁止 |
| `T *_Borrow` → 原始 `T *` | **任何地方都禁止** | 任何地方都禁止 |
| `T *_Owned` → `void *_Owned` | 可以（显式强制转换） | 可以 |
| `void *_Owned` → `T *_Owned` | 在 `_Safe` 中禁止；在 `_Unsafe` 中允许，但结果结构体内部的 `_Owned` 成员不拥有任何内容 | 可以（显式强制转换，相同注意事项） |
| `T *_Owned` ↔ `T *_Owned _ArrayElem`（C 强制转换） | 禁止 — 这些是不同的类别；使用 `__take_array_from_raw` / `__move_array_to_raw` | 禁止 |
| `T *_Owned` → 原始 `T *` | 禁止 — 使用 `__move_to_raw` | 禁止 — 使用 `__move_to_raw` |
| `T *_Owned _ArrayElem` → 原始 `T *` | 禁止 — 使用 `__move_array_to_raw` | 禁止 — 使用 `__move_array_to_raw` |
| `fn(A *_Borrow)` → `fn(B *_Borrow)`（函数指针强制转换） | 在异构 `_Safe`/`_Unsafe` 声明中禁止 — 使用蹦床模式 | 在异构声明中禁止 — 使用蹦床模式 |

**void-borrow 两步转换**是在 `_Safe` 函数内部在不相关的结构体指针类型之间转换的规范模式（例如，在桥接 FFI 风格的多态指针时）：

```c
/* 目标：public_t *_Borrow → private_t *_Borrow */
private_t *_Borrow _Nonnull this = _Unsafe(
    (private_t *_Borrow _Nonnull)(void *_Borrow _Nonnull)self);
```

- 步骤 1：`public_t *_Borrow → void *_Borrow` — 隐式，在 `_Safe` 中允许。
- 步骤 2：`void *_Borrow → private_t *_Borrow` — 显式强制转换，包装在 `_Unsafe(expr)` 中。
- 直接 `public_t *_Borrow → private_t *_Borrow` **被禁止**（不同的指向类型）。

### 签名中允许原始指针参数/返回
`_Safe` 函数**可以**有原始指针参数/返回，也可以有带指针成员或联合体类型的结构体作为参数/返回。限制适用于安全区域内对指针的*操作*，而不是它们在签名中的存在。

### 初始化规则
- **指针类型**（原始、`_Owned`、`_Borrow`、函数指针）**必须**初始化（如果需要使用 `nullptr`）
- **包含指针字段的结构体/联合体**必须使用**完整**初始化器列表进行初始化（不允许部分初始化）
- **基本类型**（`int`、`float`、`char`、`_Bool`）和不**带**指针字段的结构体/联合体**可以**保持未初始化或部分初始化

```c
_Safe {
    int a;                            // 可以：基本类型
    int *p;                           // 错误：指针必须初始化
    int *p1 = nullptr;                // 可以
    struct HasPtr hp = {nullptr, 0};  // 可以：完整初始化
    struct HasPtr hp2 = {0};          // 错误：部分初始化（有指针字段）
}
```

#### 带有 `_Owned _ArrayElem` 字段的普通结构体必须使用聚合指定初始化

一个包含一个或多个 `_Owned _ArrayElem`（或 `_Owned`）字段的普通结构体需要在声明点使用完整的指定初始化器——裸声明符在安全区域中被拒绝：

```c
/* 错误 — "安全区域中禁止未初始化的声明符" */
struct String s;

/* 正确 — 所有字段通过聚合指定初始化提供 */
struct String s = {
    .data = safe_calloc_chars(1),
    .len  = 0,
    .cap  = 1
};
```

所有 `_Owned` 字段必须在大括号列表内通过移动初始化。

### 递增/递减（`++`/`--`）
`++` 和 `--` 是**允许的**，但它们的结类型是 `void`。你可以将它们用作独立语句，但不能使用表达式的值。

```c
_Safe void foo(void) {
    int a = 0;
    a++;                // 可以：仅副作用
    int x = a++;        // 错误：结果为 void
    for (int i = 0; i < 10; i++) {}  // 可以：迭代子句
}
```

### 类型转换规则
- **不允许缩小隐式转换**（`long` 到 `int`、`double` 到 `float`、`int` 到 `float`）— 需要显式强制转换
- 适合目标类型的**编译时常量**不受缩小限制的约束
- **不允许浮点到整数的强制转换**（即使是显式的）
- 安全区域中允许显式的**整数到浮点**转换
- **枚举转换**：隐式枚举到枚举被禁止；显式仅当目标枚举包含源的所有值时允许。隐式枚举到底层整数允许
- **比较/逻辑运算符**（`==`、`!=`、`>=`、`<=`、`>`、`<`、`&&`、`||`、`!`）：结果为 `int` 类型（0 或 1），可以隐式转换为其他整数类型
- **if/while 条件**：允许任何算术类型（遵循 C 规则）

### 联合体规则
- **不允许联合体成员访问**（`.` 读取或写入）— 但联合体可以声明、初始化和作为参数传递

### 其他限制
- **不允许内联汇编**
- **不允许调用不安全函数** — 必须包装在 `_Unsafe {}` 中
- **空参数需要 `void`**：`_Safe void f(void)` — 而不是 `_Safe void f()`
- **不允许可变参数** — 除非函数有 `__attribute__((format(...)))`：
  ```c
  _Safe int foo(int a, ...);  // 错误
  __attribute__((format(printf, 1, 2)))
  _Safe int bar(const char *fmt, ...);  // 可以（但 va_start/va_arg/va_end 在函数体内部仍然被禁止）
  ```
- **Switch**：`case`/`default` 只能在 switch 后的第一级块中；不能在该第一级块中声明变量

## 5. 混合模式声明（_Safe/_Unsafe 重载）

### 5.0 为什么这有效：`_Safe` 有两个正交的含义

`_Safe` 根据出现的位置具有两个**独立**的含义。混合模式只是两种含义分别配置的情况。

| 位置 | `_Safe` 控制什么 |
|---|---|
| 在**声明**上 | **调用方契约**：任何带有 `_Safe` 的声明使该函数可以从 `_Safe` 上下文中调用。 |
| 在**定义**上 | **函数体检查**：`_Safe` 定义的函数体必须通过安全区域规则（不允许返回产生原始的 `&` C 取地址、不允许原始解引用、不允许在 `_Unsafe` 块外调用非 `_Safe` 函数等）。 |

四种组合在实践中都能看到：

| 声明 `_Safe`？ | 定义 `_Safe`？ | 可从 `_Safe` 调用？ | 函数体被安全检查？ |
|---|---|---|---|
| ✓ | ✓ | 是 | 是 — 标准 `_Safe` 函数 |
| ✓ | （否） | **是** | **否** — 用户代码混合模式（§5.2） |
| （否） | ✓ | 是（定义也是声明） | 是 |
| （否） | （否） | 否 | 否 — 普通 C 函数 |

**签名规则**：同一函数的声明/定义之间只能有 `_Safe` 关键字本身的差异。所有其他限定符（`_Owned`、`_Borrow`、`_ArrayElem`、`_Nullable`、`const`、`volatile`）都是类型的一部分，必须一致 — 下面的标准混合模式规则（§5.1）描述了一个细微的例外，即 `_Safe` 声明可以相对于 `_Unsafe` 声明**添加**所有权/借用限定符到原始指针参数/返回。

### 5.1 libc 重声明（经典应用）

同一函数可以有 `_Safe` 和 `_Unsafe` 两种声明：

```c
_Unsafe int* foo(int* p);              // 不安全版本
_Safe int* _Owned foo(int* _Owned p);  // 安全版本：添加 _Owned
```

- `_Safe` 声明可以向原始指针参数/返回**添加** `_Owned`、`_Borrow`、`_Owned _ArrayElem` 或 `_Borrow _ArrayElem`。`_Owned _ArrayElem` 和 `_Borrow _ArrayElem` 作为**完整单元**添加 — 你不能在声明之间将普通 `_Owned` 升级为 `_Owned _ArrayElem`。
- 不能**移除** `_Unsafe` 声明中存在的限定符，也不能将 `_Owned` 换成 `_Borrow`（反之亦然）。**返回类型**上的标准 C 限定符（`const`、`volatile` 等）也必须保留；**参数类型**上的为了兼容性会被剥离。
- 在安全上下文中，只有 `_Safe` 重载是可调用的。在不安全上下文中，当类型匹配时优先使用 `_Safe` 版本。
- **泛型函数不支持混合模式**
- 如果函数有多个相同安全级别的声明，它们必须一致

### `void *` 参数 — 使用 `void *_Borrow`，而不是原始 `void *`

对于参数为 `void *` 的 libc 函数（memcpy、memset、memcmp、memmove 等）— **原始 `void *` 重声明不起作用，但 `void *_Borrow` 重声明有效** — 当 T 是普通类型（没有指针字段）时，在安全区域中允许隐式 `T *_Borrow → void *_Borrow` 转换，因此调用者可以直接传递类型化的 `_Borrow` 参数。

```c
/* 不工作 — 原始 void * 重声明 */
_Safe void memcpy(void *dst, const void *src, size_t n);

_Safe void bad_example(char *dst, const char *src, size_t n) {
    memcpy(dst, src, n);
    /* 错误：在安全区域中禁止隐式转换 char* -> void* */
}

/* 工作 — void *_Borrow 重声明 */
_Safe void *_Borrow memcpy(void *_Borrow dst, const void *_Borrow src, size_t n);
_Safe void *_Borrow memset(void *_Borrow s, int c, size_t n);
_Safe int           memcmp(const void *_Borrow a, const void *_Borrow b, size_t n);

_Safe void good_example(char *_Borrow _ArrayElem dst,
                        const char *_Borrow _ArrayElem src, size_t n) {
    memcpy(dst, src, n);   /* 隐式 char *_Borrow _ArrayElem → void *_Borrow */
    memset(dst, 0, n);
}
```

**为什么这有效**：拒绝 `char * → void *` 在 `_Safe` 中的规则适用于**原始**指针（其中生命周期/别名未被跟踪）。对于 `_Borrow` 指针，借用检查器通过 `void *_Borrow` 重声明继续跟踪源生命周期，因此转换是合理的 — 并且编译器自动阻止危险情况（指向类型为带指针字段的结构体，这会让 memcpy 别名所有权）：

```c
struct S { int *p; };

_Safe void blocked(struct S *_Borrow dst, const struct S *_Borrow src) {
    memcpy(dst, src, sizeof(struct S));
    /* 错误：从 'struct S *_Borrow' 到 'void *_Borrow' 的转换被禁止
       注意：源指向类型 'struct S' 不是普通数据类型 */
}
```

这正是借用检查器应该做的：普通字节复制通过，所有权别名复制在类型级别被拒绝。无需手动过滤。

**何时仍然需要按类型包装器**：唯一常见的需要手工编写的 `_Safe` 包装器的情况是在单个缓冲区内的 `memmove`（例如，通过 `buf[dst_off..] <- buf[src_off..]` 进行修剪/移动）。两个到同一缓冲区的 `&_Mut buf[i]` 借用会在调用点失败借用检查，因此包装器接受一个 `*_Borrow _ArrayElem` + 两个 `size_t` 偏移量，并在一个 `_Unsafe` 块内部进行原始算术：

```c
_Safe void safe_memmove_within(char *_Borrow _ArrayElem buf,
                               size_t dst_off, size_t src_off, size_t n) {
    _Unsafe {
        char *raw = (char *)&_Mut *buf;
        memmove(raw + dst_off, raw + src_off, n);
    }
}
```

**通过单行重声明救活的函数**（不需要包装器体）：

- 具体指向类型 libc：`strlen`、`strcmp`、`strncmp`、`strchr`、`strerror`、`abort`、`exit` — 声明 `_Safe`，向原始指针参数添加 `_Borrow` / `_Borrow _ArrayElem`。
- `void *` 参数 libc：`memcpy`、`memset`、`memmove`、`memcmp` — 使用 `void *_Borrow` / `const void *_Borrow` 声明 `_Safe`。`_Safe` 中的调用者直接传递类型化的 `T *_Borrow` / `T *_Borrow _ArrayElem`；当 T 是普通类型时编译器允许转换，当 T 有指针字段时拒绝。

**关键：当调用者传递 `_Owned _ArrayElem` 字段时，使用 `_Borrow _ArrayElem`，而不是裸 `_Borrow` 或原始 `const char *`。**

在此子集中，字符串缓冲区作为 `char *_Owned _ArrayElem` 字段持有。调用者通过下标借用访问它们：`&_Const s->data[0]` 产生 `const char *_Borrow _ArrayElem`。保持参数为普通 `const char *`（甚至 `const char *_Borrow`）的 `_Safe` 重声明将在这些调用点被拒绝：

> "在安全区域中禁止从 `const char *_Borrow _ArrayElem` 到 `const char *_Borrow` 的转换"

正确的形式 — 已验证可编译和运行 — 是：

```c
_Safe size_t strlen(const char *_Borrow _ArrayElem s);
_Safe int strcmp(const char *_Borrow _ArrayElem a,
                 const char *_Borrow _ArrayElem b);
```

字符串字面量（`"hello"`）在 `_Borrow _ArrayElem` 参数处自动降级为 `const char *_Borrow _ArrayElem` — 无需强制转换即可接受。只有裸的 `const char *` 变量在调用者期望 `_Borrow _ArrayElem` 时需要 `_Unsafe` 桥接。

**经验法则**：如果函数的概念输入是"被解释为字符串的整个数组"（`strlen`、`strcmp`、`strchr`、memcmp 风格变体），将参数声明为 `const char *_Borrow _ArrayElem`。如果函数的概念输入是"按引用传递的单个字符"，使用普通 `const char *_Borrow`。

### 非内存资源：使用 `_Owned` 语义重声明

相同的机制适用于参数是具体非 `void *` 类型的 libc 资源函数。将创建者重声明为带有 `_Owned _Nullable` 返回，销毁者重声明为带有 `_Owned` 参数 — 然后借用检查器就像处理堆缓冲区一样精确地跟踪句柄：

```c
/* path 参数：_Borrow _ArrayElem 以便字符串字面量调用者直接使用 */
_Safe FILE *_Owned _Nullable fopen(const char *_Borrow _ArrayElem path,
                                   const char *_Borrow _ArrayElem mode);
_Safe int fclose(FILE *_Owned fp);
```

在这些重声明之后：

- `_Safe` 代码中的 `fopen` 调用产生 `FILE *_Owned _Nullable` — 编译器强制在作用域结束前消耗它。
- `fclose` 按值消耗 `FILE *_Owned` — 重复关闭和关闭后使用是编译时错误。
- 调用点不需要 `_Unsafe` 块 — 句柄像任何其他 `_Owned` 指针一样流过 `_Safe` 代码。

唯一的要求是资源类型（`FILE`、套接字 fd 包装器、锁句柄）是具体的指针或整数类型 — `void *` 参数如上所述仍然被阻止。

> 完整的设计模式（结构体字段 + 构造函数 + 析构函数函数）见 `design.md` §2 "析构函数（消费者函数）设计"。

### 5.2 用户代码混合模式：非 `_Safe` 定义上的 `_Safe` 声明

上面的 libc 应用将现有的非 `_Safe` 符号重声明为带有 `_Safe` 重载。相同的正交性（§5.0）允许你以相反的方向编写用户代码：从** `_Safe` 声明**开始，用于调用方契约，但**定义函数时不带 `_Safe`**，使函数体在安全检查之外运行。

```c
static int g_x = 42;

/* 声明：调用者的 _Safe 契约。 */
_Safe const int *get_g(void);

/* 定义：没有 _Safe → 函数体是普通 C；&g_x 只是 C 的取地址。 */
const int *get_g(void) {
    return &g_x;             /* 不需要 _Unsafe { } 块 */
}
```

对于返回借用的辅助函数的等效形式（其中 `_Safe` 借用返回规则需要一个 `_Borrow` 参数，因此添加了一个虚拟锚点）：

```c
static const int g_anchor_dummy = 0;

_Safe const int *_Borrow g_get(const int *_Borrow anchor);

const int *_Borrow g_get(const int *_Borrow anchor) {
    (void)anchor;
    return (const int *_Borrow)&g_x;   /* C 风格强制转换，无 _Unsafe */
}
```

**何时使用这种模式：**

- 函数体**大部分或完全在安全区域规则之外**（全局状态访问、FFI 适配器、底层基础设施）。在定义上贴上 `_Safe` 然后将每个其他语句包装在 `_Unsafe { }` 中，比删除 `_Safe` 关键字并让函数体成为普通 C 更难读。
- 函数是安全区域代码和不可检查数据源之间的**狭窄接缝**。在接缝处放置一个信号（定义上缺少 `_Safe`）比在函数体中散布 `_Unsafe` 块更清晰。

**何时不使用：**

- 函数体大部分是安全的，只有少数不可建模的语句 → 使用 `_Safe` 定义 + 最小的 `_Unsafe { }` 块。这是默认做法，应保持为默认；`_Safe` 的正交性是为接缝情况存在的，不是作为通用的"跳过检查"旋钮。
- 你因为函数体有一个你不想修复的借用检查器错误而想删除 `_Safe` 定义 → 那是在绕过检查器。修复注释。

**两个视觉信号**区分合法的用户代码混合模式函数和绕过 `_Safe` 的代码：

1. 定义行缺少 `_Safe`。
2. 函数体包含显式的指针类型强制转换（`(T *_Borrow)`、`(T *)`），而不是隐式传递类型。

两者都应该存在且明显。一个定义缺少 `_Safe` 但其函数体未明显执行原始/强制转换操作的函数是可疑的 — 它几乎肯定应该是一个正常的 `_Safe` 定义。

> 关于主要应用之一 — 向安全区域调用者暴露指向全局变量的指针 — 见 `design.md` §3 "返回指向全局变量的指针"。

## 6. 函数指针规则

- `_Safe` 函数指针只能从具有 `_Safe` 声明的函数赋值
- `_Unsafe` 函数指针可以从 `_Safe` 或 `_Unsafe` 函数赋值（如果类型兼容）

```c
_Safe void safe_fn(void);
_Unsafe void unsafe_fn(void);

_Safe void (*sp)(void) = nullptr;
sp = safe_fn;    // 可以
sp = unsafe_fn;  // 错误：没有可用的 _Safe 声明
```

## 7. 完整示例

```c
#include <stdio.h>

_Safe int add(int a, int b) { return a + b; }

_Safe int readBorrow(const int *_Borrow ref) {
    return *ref;  // 可以：_Borrow 解引用是安全的
}

void mixedFunction(void) {
    int *raw = (int *)malloc(sizeof(int));
    *raw = 100;
    _Safe {
        int x = 42;
        int y;  // 可以：基本类型，不需要初始化
        const int *_Borrow ref = &_Const x;
        y = readBorrow(ref);
        _Unsafe {
            printf("from safe zone: %d\n", y);
            free(raw);
        }
    }
}

_Safe int main(void) {
    int sum = add(10, 20);
    const int *_Borrow r = &_Const sum;
    _Unsafe { printf("sum = %d\n", *r); }
    return 0;
}
```

> 关于初始化分析（字段级跟踪），见 `initialization.md` 技能
> 关于借用（安全区域中需要），见 `borrowing.md` 技能
> 关于安全上下文中的 ownership，见 `ownership.md` 技能
> 关于空检查，见 `nullability.md` 技能
> 关于安全区域错误（BSC-E03xx），见 `errors.md` 技能
> 关于常见安全区域错误，见 `bsc-common-mistakes` 技能
