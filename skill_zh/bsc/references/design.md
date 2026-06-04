
# BiSheng C 设计技能

BSC 不仅仅是"C 带注释" — 所有权系统改变了 API 的塑造方式。本技能涵盖当你**从空白文件开始**时面临的设计决策，而不是在移植现有 C 时面临的问题。

## 1. 经验法则（先阅读此部分）

设计任何 BSC API 时要参考的八条规则。如果你只记得一件事，记住**规则 2**。

### 规则 1 — 返回值，借用参数
按值返回 `_Owned`；将 `_Borrow` 作为参数。仅当函数真正**消耗**（移动）值时才接受 `_Owned` 参数（例如 `Vec::push`、`set_string`）。

```c
// 好：调用者保留所有权，函数借用
_Safe String format_greeting(const String* _Borrow name);

// 坏：调用者必须移动一个它可能还想使用的值
_Safe String format_greeting(String name);
```

### 规则 2 — 使 `_Unsafe` 表面尽可能小
不要因为一行代码需要就标记整个函数为 `_Unsafe`。用 `_Unsafe { ... }` 包装最小值，并保持函数的接口为 `_Safe`。`_Unsafe` 接口具有**传染性** — 每个调用者都要付出代价。

```c
/* 好：接口 _Safe，只有 libc 调用是 _Unsafe */
_Safe size_t strlen_via_libc(const char *_Borrow _ArrayElem s) {
    size_t n = 0;
    _Unsafe { n = strlen(s); }
    return n;
}

/* 坏：整个函数因为一行代码而成为 _Unsafe */
_Unsafe size_t strlen_via_libc(const char *s) {
    return strlen(s);
}
```

### 规则 3 — 有意识地选择 `T` 与 `T *_Owned`

这是一个真正的设计决策，不是一个默认项。两者都有效；基于类型的大小、生命周期和使用场景选择。

**按值返回 `struct T` 当：**
- 类型**小到中等**（~< 几百字节）— 移动成本低。
- 调用者将在本地使用它，并在作用域退出时让 `name_free` 消耗它。
- 你想要零分配构造。

```c
_Safe struct String str_new(void);                       /* 小，本地使用 */
```

**返回 `T *_Owned`（原始拥有指针）当：**
- 类型**很大** — 指针移动是 `O(1)`，值移动复制字节。
- 结果必须**显式可为空**，无需 INVALID 哨兵。
- 调用者的生命周期模型需要显式的指针交接。

对于此子集中的大多数类型，值返回 + INVALID 哨兵（见 `invalid-sentinel.md`）涵盖了可为空的需求，并避免了额外的间接层。

### 规则 4 — 多个 `_Const` 读取者 OR 一个 `_Mut` 写入者（不能同时）

`_Const` 借用可组合：任意数量的不可变借用可以在同一值上共存。检查器禁止的是**将 `_Mut` 与任何其他借用重叠** — 另一个 `_Mut` 或任何 `_Const`。

```c
// 好：多个读取者，没有写入者
const T* _Borrow r1 = &_Const x;
const T* _Borrow r2 = &_Const x;
const T* _Borrow r3 = &_Const x;
read(r1); read(r2); read(r3);   // 全部可以

// 坏：写入者与读取者重叠
const T* _Borrow r = &_Const x;
T* _Borrow w = &_Mut x;          // 错误：r 仍然活跃
modify(w);

// 坏：两个写入者重叠
T* _Borrow w1 = &_Mut x;
T* _Borrow w2 = &_Mut x;          // 错误：w1 仍然活跃
```

**设计含义：** 如果你的 API 需要多个读取者，接受 `const T* _Borrow` 并自由传递。如果需要修改，接受 `T* _Borrow`（可变），并确保在调用期间没有其他借用活跃。如果你发现自己需要重叠的 `_Mut` 借用，设计有误 — 重新组织。

```c
// 常见设计错误：尝试对 `val` 进行两次可变借用
JSON_Object* _Borrow obj = val.get_object();   // 可变借用 #1
val.validate(&_Mut schema);                     // 可变借用 #2 — 冲突

// 修复：严格限定第一个借用的作用域
{
    JSON_Object* _Borrow obj = val.get_object();
    obj->set_string(...);
}   // 借用 #1 在此处结束
val.validate(&_Mut schema);   // 现在可以
```

### 规则 5 — 按值析构函数模式
对于每个持有一个或多个 `T *_Owned _ArrayElem` 字段的普通结构体，编写一个按值接受结构体并通过项目的 `safe_free_*` 包装器消耗每个 `_Owned` 字段的析构函数函数。

```c
struct String {
    char *_Owned _ArrayElem _Nullable data;
    size_t len;
    size_t cap;
};

_Safe void str_free(struct String s) {
    safe_free_chars(s.data);   /* 消耗 s.data；s 干净地退出 */
}
```

析构函数必须在每个代码路径上显式调用 — 作用域结束时没有自动调用。

### 规则 6 — 仅限自由函数
每个操作都是一个自由函数，接受 `struct T *_Borrow s`（读取/修改）或 `struct T s`（消耗/销毁）。语言级别不存在类似方法的组织方式；选择清晰的 `name_op` 前缀（`str_push`、`str_find`、`str_free`），使调用点自然可读。

```c
_Safe size_t str_len(const struct String *_Borrow s);
_Safe void   str_push(struct String *_Borrow s, char c);
_Safe void   str_free(struct String s);
```

### 规则 7 — 仔细选择不安全的接缝
每个 BSC 库至少有一个 `_Unsafe` 接缝。将其放在 BSC 模型与现实不匹配的**边界**处：

- **FFI / 系统调用**（`fopen`、`read`、缓冲区上的指针算术）
- **原始分配器包装器**（每个元素类型一个 `_Unsafe` 块内的 `calloc`/`free` + `__take_array_from_raw` / `__move_array_to_raw`）
- **性能关键的热路径**（检查器会强制不必要的克隆）

可能失败的查找**不是**接缝 — 返回 `T *_Borrow _Nullable`（编译时可空性跟踪使调用点保持 `_Safe`）。见规则 8。

永远不要将接缝放在业务逻辑的中间。

### 规则 8 — 优先使用 `_Nonnull`；仅当"不存在"是真实的情况时才使用 `_Nullable`

`_Owned` 和 `_Borrow` 默认为 `_Nonnull`。保持它们这样，除非 API 确实有无结果/可选/不存在的情况需要表示。`_Nullable` 强制每个使用点在进行解引用前进行空检查 — 当指针实际上从不为空时选择它只会给调用者增加无益的负担。

**保持默认 `_Nonnull`** 当：
- 参数是必需的（没有"输入不存在"的情况）
- 当函数成功时返回值始终产生
- 结构体字段在构造后始终被填充

**选择 `_Nullable`** 当：
- 分配可能失败并且你暴露这一点：`T *_Owned _Nullable try_alloc(size_t)`
- 清理路径持有可能尚未设置的指针：`Node *_Owned _Nullable item = nullptr;`
- 结构体字段确实是可选的
- setter 接受"清除"作为有效输入：`void set_cache(Cache *_Owned _Nullable c)`

```c
// 好 — 默认 Nonnull；调用者不需要空检查
_Safe void render(const Canvas *_Borrow c, const Frame *_Borrow f);

// 好 — 分配可能失败，可为空暴露它
_Safe Buffer *_Owned _Nullable try_create_buffer(size_t n);

// 坏 — 出于谨慎声明为 _Nullable；调用者无理由进行空检查
_Safe void render(const Canvas *_Borrow _Nullable c, const Frame *_Borrow _Nullable f);
```

更改现有 API 上指针的可空性（Nonnull ↔ Nullable）是一个**破坏性更改** — 每个调用者的空检查假设都会改变。在设计时决定，而不是修补时。

> 关于编译时可空性跟踪机制和空检查模式，见 `nullability.md` 技能。

### 规则 9 — 将所有 `_Unsafe` 集中到命名的语义辅助函数中；公共 API 是零 `_Unsafe`

规则 2 适用于单个函数。在**模块级别**，将其应用高一层：将每个原始内存操作收集到专用的 `static _Safe` 辅助函数中，每个函数包含恰好一个用于一个语义操作的 `_Unsafe` 块（原始视图、原始复制、原始比较、通过借用释放等）。每个公共函数完全由这些辅助函数组成，没有自己的 `_Unsafe`。

通过 `*_Borrow` 的逐字节读/写根本不需要辅助函数 — `_Owned _ArrayElem` 下标 `s->data[i]` 在 `_Safe` 代码中直接工作（见 `ownership.md` §8）。只有接触安全区域禁止的原始内存（memcpy/memcmp/memmove、底层缓冲区的 `free`、原始指针算术）的操作才需要辅助函数。

最清晰的分层使用两层分割：

- **底层**（`bsc_str_safe.cbs`）：`<string.h>` / `<stdlib.h>` 的包装器，接受 `char *_Borrow _ArrayElem` 参数，每个包含一个最小的 `_Unsafe`。它们独立于任何特定的结构体。
- **顶层**（`bsc_str.cbs`）：`static _Safe` 辅助函数，通过 `&_Mut s->data[offset]` / `&_Const s->data[offset]` 委托给底层。**在借用点将偏移量嵌入到下标中**，以便包装器不需要显式的偏移参数。

```c
/* 底层 — 在确切位置接受借用，不需要偏移参数 */
_Safe void safe_memcpy_into(char *_Borrow _ArrayElem dst, const char *src, size_t n) {
    _Unsafe { char *raw = (char *)&_Mut *dst; memcpy(raw, src, n); }
}
_Safe int safe_memcmp_arr(const char *_Borrow _ArrayElem a,
                          const char *_Borrow _ArrayElem b, size_t n) {
    int r;
    _Unsafe {
        const char *ra = (const char *)&_Const *a;
        const char *rb = (const char *)&_Const *b;
        r = memcmp(ra, rb, n);
    }
    return r;
}
/* 例外 — 单个缓冲区内的 memmove 接受一个借用 + 两个偏移量，
 * 因为借用检查器拒绝两个同时指向同一缓冲区的 &_Mut */
_Safe void safe_memmove_within(char *_Borrow _ArrayElem buf,
                               size_t dst_off, size_t src_off, size_t n) {
    _Unsafe { char *raw = (char *)&_Mut *buf; memmove(raw + dst_off, raw + src_off, n); }
}

/* 顶层 — 零 _Unsafe；偏移量进入下标 */
static _Safe void str__write_raw(struct String *_Borrow s, size_t offset,
                                 const char *src, size_t n) {
    safe_memcpy_into(&_Mut s->data[offset], src, n);
}
static _Safe int str__cmpn(const struct String *_Borrow a,
                           const struct String *_Borrow b, size_t n) {
    return safe_memcmp_arr(&_Const a->data[0], &_Const b->data[0], n);
}

/* 公共 API — 零 _Unsafe；下标读/写内联 */
_Safe void str_to_upper(struct String *_Borrow s) {
    for (size_t i = 0; i < s->len; i++) {
        char c = s->data[i];                                   /* 直接下标 */
        if (c >= 'a' && c <= 'z') s->data[i] = (char)(c - 'a' + 'A');
    }
}
```

这种分解的好处：

- **可审计性**：模块的完整 `_Unsafe` 表面就是辅助函数列表。审计员可以阅读约 8 个短函数，覆盖所有不安全代码。
- **可测试性**：每个辅助函数有一个职责，可以通过调用它的公共 API 进行测试。
- **组合安全性**：公共函数完全是 `_Safe` 的 — 它们可以从任何安全上下文中调用，它们的调用者不会仅仅因为实现接触了原始内存而变成 `_Unsafe`。
- **易于扩展**：添加新的公共函数意味着只编写安全代码；原始操作集已经枚举完毕。

---

## 2. 设计类型

### 选择类型形状

| 你的类型有…… | 使用 |
|---|---|
| 纯数据（无堆指针，无清理） | 只有标量字段的普通 `struct` |
| 单个元素类型的一个堆缓冲区，支持 `[]` | 带有 `T *_Owned _ArrayElem _Nullable` 字段的普通 `struct`（见 `ownership.md` §7.5）+ `name_free` 消费者 |
| 多个堆缓冲区 | 带有几个 `T *_Owned _ArrayElem` 字段的普通结构体；`name_free` 消费者必须释放每个 |
| 一组封闭的变体 | 带有标签枚举 + 每个变体一个字段的普通 `struct`（所有变体始终初始化；析构函数释放所有） |
| **大型内联数据**（≥ 几百字节的内联字段/数组） | 堆装箱整个结构体 — 见下面的"大型结构体" |

### 大型结构体 — 堆装箱而非按值传递

按值句柄模式是为**小型句柄结构体**设计的（通常 3-6 个标量/指针字段，约 24-64 字节）。`_Owned _ArrayElem` 缓冲区在堆上；句柄只携带一个指针和一些簿记信息，因此按值传递和 `name_free` 消耗成本低。clang 的 RVO/NRVO 通常将其折叠为单个存储。

当结构体本身有大型内联数据（`int meta[1024]`、几个内联子结构体、数十个字段）时，按值句柄模式不再是免费的。将整个结构体堆装箱，改为传递拥有的指针。

```c
struct Big {
    int    meta[64];
    char   name[128];
    size_t counts[16];
    /* ... 大型主体 ... */
};

_Safe struct Big *_Owned _Nullable big_new(void) {
    struct Big *raw;
    _Unsafe { raw = (struct Big *)calloc(1, sizeof(struct Big)); }
    if (raw == nullptr) return nullptr;
    _Unsafe { return __take_from_raw(raw); }
}

_Safe void big_free(struct Big *_Owned _Nullable b) {
    if (b == nullptr) return;
    struct Big *_Owned nonnull = b;                 /* 缩小 */
    _Unsafe { free(__move_to_raw(nonnull)); }
}

/* 操作接受借用；所有权仅在构造/析构时移动。 */
_Safe int  big_get(const struct Big *_Borrow b, size_t i) { return b->meta[i]; }
_Safe void big_set(struct Big *_Borrow b, size_t i, int v) { b->meta[i] = v; }
```

调用点 — `&_Mut *b` / `&_Const *b` 馈送给接受 `_Borrow` 的辅助函数：

```c
struct Big *_Owned _Nullable b = big_new();
if (b == nullptr) return -1;
big_set(&_Mut *b, 0, 42);
int x = big_get(&_Const *b, 0);
big_free(b);
```

句柄现在是一个指针（8 字节）。移动语义仍然保持 — 移动 `struct Big *_Owned` 是一个 8 字节的指针移动，无论 `Big` 的大小如何。失败仍然可以通过 `_Nullable + nullptr` 表达（指针级别的 INVALID 哨兵，而非结构体级别 — 见 `invalid-sentinel.md`）。

#### 变体：将拥有的指针包装在小型句柄结构体中

如果你希望与其他 `String`/`Vec` 风格类型（按值返回、`name_free(name h)` 消费者）保持表面一致性，将拥有的指针包装在单字段句柄中：

```c
struct BigHandle {
    struct Big *_Owned _Nullable inner;
};
_Safe struct BigHandle bigh_new(void)        { /* ... */ }
_Safe void           bigh_free(struct BigHandle h) { /* 按值消费者 */ }
```

运行时成本与裸指针版本相同（句柄仍然是 8 字节）。好处是 `struct BigHandle` 和任意 `struct Big *_Owned` 之间的类型级别消歧，以及小型句柄获得的相同编译时字段泄漏检测。

#### 反模式：大型结构体的 `ensure_init` 输出参数

一种诱人的"无按值复制"替代方案是给调用者在栈上的 `struct Big big;`，并让函数通过 `__attribute__((ensure_init))` + `*_Borrow` 填充它。**不要对大型结构体这样做。** 调用者支付完整的 `sizeof(struct Big)` 在栈帧中 — 这正是你试图避免的成本。`ensure_init` 输出参数适用于小型 POD 类型（`int`、`struct Point`），其中栈分配确实比堆调用更便宜，不适用于大型结构体。

### 决策规则（大 vs. 小）

| 结构体形状 | 使用 |
|---|---|
| 句柄（`_Owned _ArrayElem` 字段 + 几个标量），总计 ≤ ~64 B | 按值句柄。不要优化。 |
| 许多字段但每个都很小，总计仍 ≤ ~128 B | 按值句柄；信任 RVO。 |
| 大型内联数组 / 嵌套的大结构体，总计 ≥ 几百字节 | 堆装箱：`struct Big *_Owned _Nullable`（可选包装在单字段句柄结构体中） |
| 在许多借用调用链中大量使用，按值返回罕见 | 堆装箱，即使大小在边界附近。热路径基于借用，因此堆分配成本只在构造时支付一次。 |

### 变体类型的标签结构体成本

带有 `{tag, var_a, var_b, var_c}` 的普通结构体同时保持每个变体的存储存活。对于持有堆缓冲区的变体，构造函数必须分配每个变体（即使活跃标签只使用一个），以便析构函数可以统一释放所有变体。内存成本随变体数量增长；接受成本或重新组织以避免联合体模式。

### 析构函数（消费者函数）设计

析构函数是一个自由函数，按值接受结构体并消耗每个 `_Owned` 字段。它不会自动调用；每个代码路径必须调用它。

对于非内存资源（文件句柄、套接字、锁），将资源字段声明为 `_Owned`，并将 libc 创建者/销毁者重新声明为带有所有权限定符的 `_Safe`（混合模式 `_Safe` 重声明 — 见 `safe-zone.md` §5）。然后借用检查器像处理堆缓冲区一样精确地跟踪句柄，在编译时防止泄漏和重复关闭。

```c
/* libc 的混合模式 _Safe 重声明 — 告诉借用检查器
 * fopen 产生拥有的资源，fclose 消耗一个。 */
_Safe FILE *_Owned _Nullable fopen(const char *_Borrow _ArrayElem path,
                                    const char *_Borrow _ArrayElem mode);
_Safe int fclose(FILE *_Owned fp);

struct FileHandle {
    FILE *_Owned _Nullable fp;   /* 作为拥有的跟踪；关闭状态时为 null */
};

_Safe struct FileHandle filehandle_open(const char *_Borrow _ArrayElem path) {
    return (struct FileHandle){ .fp = fopen(path, "r") };
}

_Safe void filehandle_close(struct FileHandle h) {
    if (h.fp == nullptr) return;
    FILE *_Owned fp = h.fp;          /* 缩小 nullable → nonnull */
    (void)fclose(fp);                /* 消耗 fp；不需要 _Unsafe */
}
```

需要显式析构函数体的情况：

- 由 `T *_Owned _ArrayElem` 持有的堆缓冲区 → 调用 `safe_free_T(s.field)`。
- 外来资源（`FILE *`、套接字、锁）→ 声明字段为 `_Owned` 并使用消耗 `_Owned` 值的 `_Safe` 重声明销毁器。
- 释放前的副作用逻辑（日志、通知）。

### 头文件中的单行直通包装器 — `static inline`

对于函数体是单个调用或表达式的按类型包装器 — 通常是 `name_free`、`name_is_valid`、`name_len`、按类型 `safe_calloc_T` / `safe_free_T` 分配器对以及类似的简单访问器 — 将定义放在头文件中作为 `static inline`：

```c
/* 在 .hbs / .h 头文件中 */
static inline _Safe void str_free(struct String s) {
    safe_free_chars(s.data);
}

static inline _Safe _Bool str_is_valid(const struct String *_Borrow s) {
    return s->data != nullptr;
}

static inline _Safe size_t str_len(const struct String *_Borrow s) {
    return s->len;
}
```

为什么是 `static inline` 而不是简单的 `inline` 或普通的非内联函数：

- **`static inline` 是头文件定义函数的安全 C99/C11 惯用法。** 如果调用未内联，每个翻译单元获得自己的副本；编译器在任何优化级别内联。不需要链接步骤协调。
- **普通的 `inline`（无 `static`）** 根据 C99/C11 规则需要在某处有外部定义 — 在双构建模式下很容易出错，标准 C 路径可能需要但它。避免。
- **.c 中的非内联函数**：工作正常，但对于单行代码，包装器的 call/ret + 24 字节按值复制在 `-O0` 下是纯粹的开销，即使在 `-O2` 下你也依赖于跨 TU 的内联或 LTO。`static inline` 消除了这种依赖。

何时**不**内联：

- 函数体 ≥ 3 行，有分支，或包含超过单个语句的 `_Unsafe { ... }` 块。内联会在每个调用点增加二进制大小而不会带来有意义的收益；让编译器通过 -O 决定。
- 包装器有外部链接需求（取地址、函数指针使用等）。将其保持为 .c 中的常规 `_Safe` 函数。

注意这是 **API 卫生，而非性能优化**。在现代 CPU 上，每次调用包装器的 call/ret 成本在实际工作负载中是看不见的；使用 `static inline` 的原因是包装器是一种符号便利，应该编译为与手动内联函数体相同的代码。经验证可与 `_Safe` 编译（`_Safe static inline` 和 `static inline _Safe` 都被接受）；所有权/移动/字段泄漏跟踪不受内联影响。

---

## 3. 设计 API

### 值进值出 vs. 借用进借用出

```c
// 模式 A：所有权转移
_Safe JSON_Object JSON_Object::new(void);            // 产生拥有的值
_Safe void Obj::set_value(Obj* _Borrow this, JSON_Value v);  // 消耗 `v`

// 模式 B：只读访问
_Safe const String* _Borrow Obj::get_name(const Obj* _Borrow this, size_t i);
_Safe size_t Obj::get_count(const Obj* _Borrow this);

// 模式 C：通过借用的修改
_Safe void Obj::clear(Obj* _Borrow this);
```

**规则：** 读取 → `_Borrow this` + `_Borrow` 返回。修改 → `_Borrow this`（可变）+ `void` 返回或状态码。构造 → `_Owned` 返回，无 `this`。

### 返回依赖于局部的 `_Borrow`

每个人最终都会遇到这个问题。如果你计算一个局部 `String` 并想返回通过它查找的东西的 `_Borrow`，检查器拒绝它，因为局部变量在作用域结束时死亡。

**三种合法的修复：**

1. **重新组织** — 从调用者那里借用一个键：
   ```c
   _Safe T* _Borrow lookup(Table* _Borrow t, const String* _Borrow key);
   ```

2. **递归** — 如果局部变量是输入的转换，将转换后的值传递给递归调用，在其内部生命周期是自包含的。

3. **最小 `_Unsafe` 接缝** — 对于检查器无法建模的查找（哈希表命中），在紧凑的 `_Unsafe` 块内通过原始指针转换。记录原因。（Parson 的 `dotget_value` 就是这样做的。）

### 返回指向全局变量的指针 — `_Borrow` 不直接可用

两条 BSC 规则结合使"返回指向全局变量的 `_Borrow`"无法直接表达：

1. 不带 `_Borrow` 参数的 `_Safe T *_Borrow f(void)` 被**拒绝** — "借用返回需要至少一个借用参数"（生命周期需要在输入中有锚点）。
2. `_Safe` 内部的 `&_Mut global` 被**拒绝** — 在安全区域中不能对全局变量进行可变借用。
3. 在 `_Safe` 中从 `&_Const global` 返回原始 `T *` 被**拒绝** — 借用→原始转换在 `_Safe` 中被禁止。

因此即使是只读情况 `_Safe const T *_Borrow get_g(void)` 也不能编译。决策树（优先顺序）：

```
需要暴露指向全局变量的指针？
├─ 读取者只想要值，T 很小/可复制
│       → 按值返回：_Safe T get_g(void);
├─ 全局变量是大型 const 查找表
│       → 暴露按值返回的按字段访问器：
│         _Safe int  table_field(unsigned i);
├─ "全局变量"实际上是模块私有状态，为程序生命周期持有
│       → 重构消除全局变量：调用者（main）构造一个上下文
│         结构体，将其作为 Ctx *_Borrow 通过每个 API 传递。
│         借用现在锚定在参数上 — 类型系统和现实一致，
│         不需要逃生口。
└─ 以上都不适用：见下面的"逃生口"。
```

#### 逃生口 — 混合模式：`_Safe` 声明 + 非 `_Safe` 定义

当重构不可行时（大型以读为主的全局变量，许多需要借用形状的调用点），使用 BSC 的混合模式重声明：声明函数为 `_Safe`，使调用者可以从 `_Safe` 上下文调用它，但**定义时不带 `_Safe`**，使函数体在非安全上下文中运行，可以使用普通 C 语法返回全局变量的原始地址。这适用于用户定义的函数，不仅限于 libc 包装器 — 见 `safe-zone.md` §5 了解通用规则。

两种形状；按调用点数量选择。

**形状 A — 返回原始值，每个调用者在 `_Unsafe` 中转换。** 对于 ≤ 3 个调用点，编写成本低。

```c
static int g_x = 42;

/* 声明：_Safe 以便安全区域中的调用者可以调用它。 */
_Safe const int *get_g(void);

/* 定义：非 _Safe；函数体是普通 C，&g_x 只是 C 的取地址。 */
const int *get_g(void) {
    return &g_x;
}

_Safe int caller(void) {
    const int *_Borrow b;
    _Unsafe { b = (const int *_Borrow)get_g(); }
    return use_borrow(b);                        /* b 馈送 _Safe API */
}
```

**形状 B — 集中的已经返回 `_Borrow` 的 `_Safe` 辅助函数。** 调用者保持零 `_Unsafe`。对于 > 5 个调用点值得。相同的混合模式技巧 — 声明是 `_Safe`，定义是非 `_Safe` 并使用普通 C 转换。

```c
static const int g_anchor_dummy = 0;             /* 从不读取，仅锚点 */

/* 声明：_Safe + _Borrow 返回 + _Borrow 参数（生命周期锚点） */
_Safe const int *_Borrow g_get(const int *_Borrow anchor);

/* 定义：非 _Safe；函数体是普通 C，原始到借用的转换只是 C 转换 */
const int *_Borrow g_get(const int *_Borrow anchor) {
    (void)anchor;
    return (const int *_Borrow)&g_x;
}

_Safe int caller(void) {
    const int *_Borrow b = g_get(&_Const g_anchor_dummy);
    return use_borrow(b);                        /* 调用点无 _Unsafe */
}
```

为什么这比将函数体包装在 `_Unsafe { ... }` 中更清晰：函数体读起来像普通 C — 没有额外的花括号，没有限定符处理。混合模式声明承载了整个"这是一个安全可调用但内部不通过安全区域检查的函数"的声明。阅读者看到一个地方（定义上缺少 `_Safe`）而不是两个地方（函数上的 `_Safe` 加上内部的 `_Unsafe` 块）。

关于这背后的语言机制 — `_Safe` 的两个正交含义（声明 = 调用者契约，定义 = 函数体检查）以及何时使用此模式与默认的 `_Safe` 定义 + 最小 `_Unsafe` 块 — 见 `safe-zone.md` §5.0 和 §5.2。

**两种形状都有相同的漏洞**：`_Borrow` 是通过原始转换构造的，因此借用检查器不知道它与全局变量别名。

- **不会冻结源。** 在"借用"活跃时对全局变量的外部写入不会被诊断。验证套件中的测试 9.2 确认了这一点。
- **不会检测多借用冲突。** 两个调用者可以同时持有指向 `g_x` 的 "`*_Borrow`" 和一个 `*_Mut` 形状的借用，而检查器没有注意到。
- **NLL 是在借用变量上计算的，但底层值可能已更改。** 对 `b` 的晚使用读取的是全局变量中的当前内容，而不是 `b` 被创建时的内容。

仅当**所有**以下条件成立时才使用此逃生口：

- 全局变量是 `const` 或按约定被视为以读为主。
- 修改点（如果有）被显式记录；读取者不得在它们之间持有转换后的借用。
- 在你的模型中并发修改是不可能的（没有线程接触此全局变量，或外部同步已经覆盖了它）。
- 更简单的替代方案（按值返回、上下文结构体重构）已被考虑并排除。

如果那些条件不全部成立，改为重构 — 借用的虚假生命周期最终会掩盖真正的 bug。

#### 反模式：完全 `_Safe` 定义返回 `&_Const global`

这能编译，看起来诚实，但比形状 B 隐藏了更多问题：

```c
/* 不要 — _Safe 定义 + &_Const global 隐藏了差距 */
_Safe const int *_Borrow get_g(const int *_Borrow anchor) {
    (void)anchor;
    return &_Const g_x;
}
```

函数体看起来像普通的安全区域借用代码。审查者扫描函数看到 `_Safe`，看到 `&_Const`，并将结果推理为指向 `anchor` 生命周期的真实借用。它不是 — 源是 `static` 的，借用的冻结/别名保证不成立。

上面的形状 B 使用混合模式（声明 `_Safe`，定义不带 `_Safe`）和函数体中的显式 C 风格转换表达了相同的思想：

```c
/* 形状 B 定义，为比较再次展示 */
const int *_Borrow g_get(const int *_Borrow anchor) {
    (void)anchor;
    return (const int *_Borrow)&g_x;
}
```

两个信号告诉读者"借用检查器没有完全建模这个"：定义行上缺少 `_Safe`，以及显式的指针类型转换。两者在反模式中都缺失，这就是为什么反模式隐藏了差距而形状 B 没有。

### 构造函数模式

使用 `name_new` / `name_with_capacity` / `name_from_cstr` 作为约定：

```c
_Safe struct String str_new(void);
_Safe struct String str_with_capacity(size_t cap);
_Safe struct String str_from_cstr(const char *_Borrow _ArrayElem s);
```

构造函数按值返回结构体。分配失败时返回 INVALID 实例（见 `invalid-sentinel.md`）而不是中止。

---

## 4. 设计所有权流

### 所有权在哪里

对于每个堆分配，决定：**谁拥有它，拥有多久，如何释放？** 然后让类型反映答案。

```c
/* 拥有者：结构体。生命周期：直到调用 str_free(s)。 */
struct String { char *_Owned _ArrayElem _Nullable data; size_t len, cap; };

/* 拥有者：str_push 的调用者。字符被复制；没有转移。 */
_Safe void str_push(struct String *_Borrow s, char c);

/* 拥有者：结构体。作为借用返回供只读使用。 */
_Safe const char *_Nullable str_as_cstr(const struct String *_Borrow s);
```

### API 边界处的移动语义

当函数接受 `T *_Owned` 或按值接受 `struct T` 时，调用者在调用后失去对变量的访问。设计使得调用者要么：

- **有意地移动它**（例如 `str_free(s)` — `s` 被消耗）
- **先克隆**，如果他们需要继续使用该值（`str_clone(&_Const s)`）

不要有"有时消耗，有时不"的函数 — 选择一个。

### 共享

只有单一所有权。如果两个地方需要读取相同的值，给每个传递一个 `*_Borrow`。如果两个地方需要写入，重新设计数据流，使得一次只有一个写入者。此子集中没有共享所有权原语。

---

## 5. 设计 `_Safe` / `_Unsafe` 边界

### 在设计时选择边界，而不是在调试期间

在编写代码之前，列出哪些函数是 `_Safe` 哪些是 `_Unsafe`。常见的库结构：

```
公共 API（.hbs）
  ├── 90% 的表面是 _Safe
  └── _Unsafe 用于：
        - 接受原始 C 字符串（const char*）的构造函数
        - 与现有 C 库互操作的函数
        - 关闭系统资源的析构函数

内部实现（.cbs）
  ├── 模型允许的地方用 _Safe
  └── _Unsafe 块（不是整个函数）用于：
        - 指针算术
        - 检查器过度拒绝的别名
```

### 何时在 `_Safe` 函数内部使用 `_Unsafe` 块

完全合法当：

- 你需要 `printf`/`puts`（格式化输出在 BSC 中被认为不安全）
- 你在已知安全的缓冲区上进行指针算术（`string->get(i)` 返回原始字符指针）
- 你在 `_Owned` 和原始之间进行强制转换以进行 `__take_from_raw` / `__move_to_raw`（C FFI）

不合法用于：

- 因为"烦人"而跳过借用检查器
- 绕过你不理解的诊断 — 先弄清楚为什么它触发

---

## 6. 反模式

### 反模式：`_Unsafe` 一路到底
症状：每个函数都是 `_Unsafe`。你在编写带 .cbs 扩展名的 C。
修复：将公共 API 标记为 `_Safe`，将 `_Unsafe` 部分推入每个函数内部的块中。检查器仍然在边界处帮助你。

### 反模式：上帝结构体
症状：一个有 15 个字段的普通结构体，大多数在大多数代码路径中未使用。
修复：分解为通过 `_Owned _ArrayElem` 字段组合的更小的结构体。每个子结构体有自己的 `name_free` 消费者。

### 反模式：返回原始 `T*` 而不是 `T *_Borrow`
症状：查找返回 `T*`（原始）而不是 `T *_Borrow`，强制调用者进入 `_Unsafe` 才能使用它。
修复：如果函数总是成功，返回 `T *_Borrow`。如果可能失败，返回 `T *_Borrow _Nullable` — 编译时可空性跟踪保持调用者 `_Safe`，同时表示缺失情况。

### 反模式：做太多事的析构函数
症状：析构函数深入到不相关的子系统、调用日志记录、修改全局状态。
修复：析构函数应该只释放 `this` 拥有的资源。副作用属于调用者调用的显式方法。

### 反模式：先为 C 设计，然后"移植"
症状：你编写了 C 风格的 API（`Buffer* parse(const char*)`），现在正在与借用检查器斗争。
修复：从 BSC 类型开始（`struct Buffer parse(const char *_Borrow _ArrayElem)`），让签名驱动实现。

---

## 7. 一个完整的示例

一个简单的字符串库，以子集优先的方式设计：

```c
/* bsc_str.hbs */
struct String {
    char *_Owned _ArrayElem _Nullable data;   /* nullptr → INVALID */
    size_t len;
    size_t cap;
};

_Safe _Bool         str_is_valid(const struct String *_Borrow s);
_Safe struct String str_new(void);
_Safe struct String str_from_cstr(const char *_Borrow _ArrayElem s);
_Safe void          str_free(struct String s);            /* 析构函数 */
_Safe size_t        str_len(const struct String *_Borrow s);
_Safe void          str_push(struct String *_Borrow s, char c);
_Safe _Bool         str_eq(const struct String *_Borrow a,
                           const struct String *_Borrow b);
```

设计说明：

- **规则 1：** 构造函数按值返回；修改器接受 `*_Borrow`；读取器接受 `const *_Borrow`。
- **规则 5：** `str_free(s)` 是消费者析构函数 — 显式的。
- **规则 6：** 每个操作都是自由函数 — 没有成员函数语法。
- **规则 9：** 所有原始 `memcpy`/`memcmp`/`free` 调用都存在于各自 `_Unsafe` 块内的 `static _Safe` 辅助函数中；公共 API 有零个 `_Unsafe`。
- **INVALID 哨兵**（见 `invalid-sentinel.md`）：在 OOM 时，构造函数返回 `{nullptr, 0, 0}`；对 INVALID 的修改器静默无操作；`str_is_valid` 是有效性测试。

---

## 8. 快速参考

| 问题 | 答案 |
|---|---|
| 我应该返回 `struct T` 还是 `T *_Owned`？ | 小/中型用值 + INVALID 哨兵用于可为空；`*_Owned` 用于大型或需要显式指针交接时 |
| 我应该接受 `struct T`、`T *_Owned` 还是 `T *_Borrow`？ | `_Borrow` 用于读取，可变 `_Borrow` 用于修改，按值用于消耗/销毁 |
| 这应该是 `_Safe` 还是 `_Unsafe`？ | `_Safe` 除非不能 — 然后内部包装 `_Unsafe` 块 |
| 我需要析构函数体吗？ | 是 — 每个 `_Owned _ArrayElem` 字段需要在 `name_free(struct T s)` 消费者中显式释放 |
| 如何表示"不存在" / OOM？ | INVALID 哨兵模式（见 `invalid-sentinel.md`）— 返回 `data = nullptr` 的结构体 |

在你写一行代码之前：勾勒类型，注释谁拥有什么，决定 `_Unsafe` 接缝放在哪里。在那之后代码几乎自己就写出来了。

## 9. OOM 容忍 API 设计

INVALID 哨兵模式（带有 `_Owned _ArrayElem _Nullable` 缓冲区字段的普通结构体，其中 `data == nullptr` 编码失败状态）是此子集中传播分配失败而不是中止的规范方式。它在整个本技能中被引用（§3 构造函数模式、§7 完整示例、§8 快速参考、"大型结构体"）。关于完整模式 — VALID/INVALID 模型、构造函数/查询/修改器/析构函数契约、辅助函数中强制性的防御性空保护 — 见专门的 `invalid-sentinel.md` 技能。
