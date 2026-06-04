
# BiSheng C Ownership（子集）

## 关键：`_Owned` 语法

**`_Owned` 放在 `*` 之后**，而不是类型之前。它是一个指针限定符，就像 `const`。

```c
// 正确 — _Owned 在 * 之后
int *_Owned p = ...;

// 错误 — _Owned 在类型之前（不能编译）
_Owned int* p = ...;
```

相同的规则适用于 `_Borrow`：写 `int *_Borrow`，而不是 `_Borrow int*`。

## 1. 概述

用于编译时内存安全的移动语义。防止编译时的释放后使用和重复释放。一旦所有权转移，原始变量就失效了。

在此子集中，`_Owned` 指针通过项目本地的包装器（例如 `safe_calloc_chars`）分配，这些包装器内部使用原始的 `calloc`/`malloc` + `__take_array_from_raw`，在一个 `_Unsafe` 块中完成。释放使用成对的包装器（`safe_free_chars`），这些包装器按值消耗 `_Owned`。

## 2. 分配模式（自行编写）

此子集中没有 `safe_malloc<T>` / `safe_free`（libcbs 被禁用）。为每个元素类型编写一个小 `_Safe` 包装器：

```c
_Safe T *_Owned _ArrayElem _Nullable safe_calloc_T(size_t n) {
    if (n == 0) n = 1;
    T *_Owned _ArrayElem _Nullable out;
    _Unsafe {
        T *raw = (T *)calloc(n, sizeof(T));
        out = __take_array_from_raw(raw);   /* nullptr 直接通过 */
    }
    return out;
}

_Safe void safe_free_T(T *_Owned _ArrayElem _Nullable buf) {
    if (buf == nullptr) return;
    T *_Owned _ArrayElem nonnull_buf = buf;   /* 通过重新赋值缩小范围 */
    _Unsafe {
        T *raw = __move_array_to_raw(nonnull_buf);
        free(raw);
    }
}
```

分配器返回 `_Nullable` 而不是中止；调用者通过 INVALID 哨兵模式处理 OOM（见 `invalid-sentinel.md`）。

## 3. 拥有的指针

```c
int *_Owned create(int val) {
    /* ... 分配、初始化、返回 */
    return ...;  // 所有权转移给调用者
}

void consume(int *_Owned p) {
    int val = *p;
    safe_free_int(p);     /* 消耗 p */
}

_Safe int main(void) {
    int *_Owned p = create(21);
    int *_Owned p2 = p;        // 所有权转移；p 现在失效
    consume(p2);               // p2 现在失效
    int *_Owned _Nullable maybe = nullptr;  // 可为空的拥有指针
    return 0;
}
```

## 4. 多级指针释放（从内到外）

对于多级指针，从内到外释放。对于带有 `_Owned` 指针成员的普通结构体，先释放所有 `_Owned` 成员，然后释放结构体本身（或通过值将结构体传递给消费者函数 — 见 §7.5）。

```c
struct S { int *_Owned p; int *_Owned q; };

_Safe void free_s(struct S s) {
    safe_free_int(s.p);   /* 先释放内部的 _Owned 成员 */
    safe_free_int(s.q);
    /* s 本身没有 _Owned 限定符；作用域结束即可 */
}
```

## 5. 规则

- **`_Owned` 放在 `*` 之后**：写 `int *_Owned`，而不是 `_Owned int*`
- `_Owned` 只能修饰**指针类型**，不能修饰非指针类型
- `_Owned` 类型具有**移动语义**：赋值、传递、返回**转移**所有权
- 转移后，原始变量**失效** — 任何使用都是编译错误
- 作用域结束前，`_Owned` 变量**必须**释放所有权
- 释放方式：(a) 传递给接受按值 `_Owned` 的函数，(b) 调用项目本地的 `safe_free_*` 包装器，(c) 返回，(d) 赋值给另一个 `_Owned` 变量
- 不允许对 `_Owned` 指针进行指针算术运算（不允许 `+`、`-`、`[]`、`++`、`--`）。对于拥有数组且需要 `[]` 的 `_Owned` 指针，使用 `_Owned _ArrayElem` — 见 §8。
- 比较运算符（`==`、`!=`、`<` 等）是允许的
- `_Owned` 和原始指针之间不允许隐式转换 — 严格的类型匹配
- `_Owned` 和原始指针之间的显式强制转换需要 `_Unsafe` 上下文
- 例外：在安全上下文中允许 `T *_Owned` -> `void *_Owned`

### `_Owned` 禁止的用法
- **全局变量**（包括函数本地的 `static`）
- **联合体成员**：`_Owned` 不能修饰联合体类型成员
  ```c
  union U { int *_Owned p; };  // 错误
  ```
- **数组元素**：不能在数组中存储 `_Owned` 指针，包括带有 `_Owned` 成员的结构体作为数组元素。同样的限制适用于 `T *_Owned _ArrayElem` 的指向类型（内部元素类型不能有 `_Owned` 成员）

### 可为空的 `_Owned` 指针
- `int *_Owned _Nullable p = nullptr;` 允许空的拥有指针
- `__take_from_raw` 和 `__move_to_raw` 保留可空性

### 逻辑和条件运算符
- 逻辑运算符（`!`、`&&`、`||`）可用于 `_Owned` 指针（空检查，不消耗所有权）
- `_Owned` 指针可以是 `if`/`while`/`do-while`/`for`/三元运算符的条件，但**不能**是 `switch`
- 允许隐式 `_Owned` -> `_Bool` 转换（不消耗所有权）

### 函数指针匹配
- 函数指针类型必须与 `_Owned` 注释完全匹配 — 不能将带有 `_Owned` 参数的函数赋值给不带 `_Owned` 参数的指针，反之亦然

### 显式所有权转移接口
- `__move_to_raw(p)` — 移出所有权，返回原始指针
- `__take_from_raw(p)` — 从原始指针取得所有权，返回 `_Owned` 指针
- 两者都保留可空性

### 转换顺序很重要
在 `T *_Owned` 和 `void *` 之间转换时：
- **顺序 1**：`T *_Owned` → `void *_Owned` → `void *`（内部的 `_Owned` 指针必须不拥有所有权）
- **顺序 2**：`T *_Owned` → `T *` → `void *`（内部的 `_Owned` 指针**保持**所有权）

反向转换 `void *_Owned` → `T *_Owned` 是允许的（在 `_Unsafe` 中），当变量仍然拥有内存时，**但转换后结果结构体内部的 `_Owned` 成员不拥有它们的指向对象**。在读取之前，你必须重新赋值它们或将它们视为原始指针。示例：`struct S *_Owned sp = _Unsafe((struct S *_Owned)memAlloc(...));` 之后，在 `sp->p` 被重新赋值之前，将 `sp->p` 作为 `int *_Owned` 读取是编译错误。

## 6. _Safe / _Unsafe 上下文

- `_Safe` 函数使用项目本地的 `safe_calloc_*` / `safe_free_*` 包装器；原始的 `malloc`/`free` 和 `__take_array_from_raw` / `__move_array_to_raw` 在那些包装器**内部**使用，在一个最小的 `_Unsafe` 块中
- 没有 `_Safe` 或 `_Unsafe` 的函数默认是非安全的

## 7. 缓冲区所有权模式（子集 RAII）

此子集禁止 `_Owned struct`。替代方案是一个持有 `T *_Owned _ArrayElem _Nullable` 缓冲区字段的普通结构体，配以一个按值接受结构体的释放函数。以下两个操作 — 普通结构体析构函数模式（§7.5）和通过借用的缓冲区替换（§7.6）— 完全替代了 `_Owned struct` 析构函数过去提供的功能。

### 7.5 带有 `_Owned _ArrayElem` 字段的普通结构体

普通结构体可以直接持有 `T *_Owned _ArrayElem` 字段。所有权规则在字段级别应用：当结构体通过值传递或返回时，字段随结构体一起移动，编译器强制要求字段在作用域结束前被消耗。将类型与 `name_free(struct Name s)` 函数配对，该函数通过值接受结构体并消耗 `_Owned` 字段 — 这就是析构函数。

```c
/* 普通结构体 — 不是 _Owned struct */
struct String {
    char *_Owned _ArrayElem data;   /* _Owned _ArrayElem，不是原始 char* */
    size_t len;
    size_t cap;
};

/* 构造函数 — 通过值返回结构体；调用者接收所有权 */
_Safe struct String str_new(void) {
    char *_Owned _ArrayElem buf = safe_calloc_chars(1);
    struct String s = { .data = buf, .len = 0, .cap = 1 };
    return s;
}

/* 析构函数 — 通过值接受结构体，消耗 _Owned _ArrayElem 字段 */
_Safe void str_free(struct String s) {
    safe_free_chars(s.data);   /* 消耗 s.data；s 本身没有 _Owned 限定符 */
}
```

此模式的关键特性：

- **按值移动**：`struct String a = str_new(); struct String b = a;` 将 `a.data` 移动到 `b`；`a` 失效。函数调用和返回也是如此。
- **强制消耗**：编译器拒绝任何代码路径，其中带有活跃 `_Owned _ArrayElem` 字段的 `struct String` 在作用域结束前未被传递给消耗函数。
- **作用域结束时的显式释放**：每个代码路径必须显式调用结构体上的 `name_free`。没有自动调用。
- **通过 `*_Borrow` 进行修改**：要通过借用修改缓冲区，接受 `struct Name *_Borrow s` 作为参数。`_Owned _ArrayElem` 字段通过借用访问，可以被替换（见 §7.6 的替换惯用法）。
- **通过借用的直接下标有效**：`s->data[i]` 通过 `*_Borrow` 读写单个字节，无需 `_Unsafe`。`_Owned _ArrayElem` 限定符允许 `[]`（见 §8），且该许可通过结构体借用传播。仅在你需要连续的原始视图时才使用 `_Unsafe` 原始强制转换（用于 `memcpy`、`memcmp`、`memmove`）。

### 7.6 通过可变借用替换 `_Owned _ArrayElem` 字段（交换后释放）

要重新分配仅通过 `*_Borrow` 访问的结构体中的缓冲区，使用二元交换原语。在任何时候都不会有任何槽悬空。

```c
/* 原语 — 编写一次，随处使用 */
_Safe void safe_swap_arr(char *_Owned _ArrayElem *_Borrow left,
                         char *_Owned _ArrayElem *_Borrow right) {
    _Unsafe {
        /* 在不消耗所有权的情况下读取每个槽的原始值 */
        char *raw_l = (char *)&_Mut **left;
        char *raw_r = (char *)&_Mut **right;
        /* 交叉安装：每个槽现在拥有对方以前拥有的内容 */
        *left  = __take_array_from_raw(raw_r);
        *right = __take_array_from_raw(raw_l);
    }
}
```

用法 — 用 `donor.data` 替换借用的 `struct String` 内部的缓冲区，然后释放落入 `donor.data` 的内容（即旧的缓冲区）：

```c
static _Safe void str__swap_in(struct String *_Borrow s, struct String donor) {
    safe_swap_arr(&_Mut s->data, &_Mut donor.data);
    safe_free_chars(donor.data);   /* 交换后旧缓冲区落在这里 */
    s->cap = donor.cap;
    /* donor.data 被 safe_free_chars 消耗；donor 本身退出作用域 */
}
```

使之工作的三个新的语法事实：

- **`char *_Owned _ArrayElem *_Borrow` 是有效的参数类型** — 对 `_Owned _ArrayElem` 槽的借用。当你需要修改字段本身（交换、替换）而不仅仅是其字节时使用此类型。
- **`*left = __take_array_from_raw(raw)` 将新的 `_Owned _ArrayElem` 值写入借用的槽。** 这与通过 `(char *)&_Mut **left` 读取槽的原始视图是对称的，并且是通过借用安装新的拥有值的唯一方法。
- **来自不同存储位置的两个 `&_Mut` 借用不会冲突。** `&_Mut s->data`（来自 `struct String *_Borrow`）和 `&_Mut donor.data`（来自栈上的 `struct String donor`）指向不同的内存，因此在同一个调用中将两者传递给 `safe_swap_arr` 是被借用检查器允许的。两个指向**同一**缓冲区的 `&_Mut` 借用仍然被禁止。

使用此 `safe_swap_arr` + `safe_free_chars` 对作为规范的"替换 `_Owned _ArrayElem` 字段"操作。donor 是新分配的替代品；交换后，旧缓冲区落入 donor 的槽中，并通过按值传递 donor 来释放。

#### 可为空字段变体：双路径交换后释放

当结构体字段是 `_Owned _ArrayElem _Nullable` 时，`safe_swap_arr` 要求两侧都不为空（它接受 `char *_Owned _ArrayElem *_Borrow`，默认不可空）。调用者必须分成两条路径，并且必须在**每条**路径中都消耗 donor — donor 按值传递，持有一个编译器要求释放的 `_Owned` 字段。

```c
/* str__swap_in：用 donor.data 替换 s->data。
 * donor 按值消耗 — 每条路径都必须释放 donor.data。 */
static _Safe void str__swap_in(struct String *_Borrow s, struct String donor) {
    if (s->data == nullptr || donor.data == nullptr) {
        /* 一侧或两侧为空：跳过交换，只释放 donor 的缓冲区
         *（safe_free_chars 是空安全的 — 也能处理空的 donor.data） */
        safe_free_chars(donor.data);   /* 在此路径上消耗 donor.data */
        return;
    }
    /* 两侧都不为空：通过重新赋值缩小，然后交换 */
    safe_swap_arr(&_Mut s->data, &_Mut donor.data);
    safe_free_chars(donor.data);   /* 旧缓冲区落在这里；消耗它 */
    s->cap = donor.cap;
    /* donor.data 在上面被消耗；donor 退出作用域 */
}
```

**关键约束：**

- 提前返回的每个分支仍然必须调用 `safe_free_chars(donor.data)` 来消耗 donor 中的 `_Owned _ArrayElem _Nullable` 字段。在任何分支上省略它都是编译错误（"拥有的字段在作用域结束前未释放"）。
- `safe_free_chars` 必须接受 `_Nullable`（空安全），以便提前返回路径即使在 `donor.data` 本身为空时也能工作。
- 正常路径（两侧都不为空）在传递给 `safe_swap_arr` 之前不需要缩小，因为此时两个字段都已经通过了 `!= nullptr` 检查。如果检查器仍然将它们拒绝为 `_Nullable`，通过局部变量缩小每个（见 §7.5 的缩小规则和 `nullability.md` §4 特例）。

### 7.7 通过借用释放 + `*slot = nullptr` 陷阱

当你持有一个 `T *_Owned _Nullable *_Borrow slot`（或其 `_ArrayElem` 变体）时，你不能仅仅释放其内容 — 更糟糕的是，三种显而易见的形状中有两种是错误的：

| 形状 | 编译器 | 运行时 |
|---|---|---|
| `type_free(*slot);` | **拒绝**：`_Borrow type does not allow move ownership` | — |
| `*slot = nullptr;` | **编译通过** | **静默泄漏**：旧的拥有值被覆盖而不释放，调用者的 `_Owned _Nullable` 现在为 `nullptr`，因此其自己的释放路径提前返回。 |
| 原始救援 + 赋空（下文） | 编译通过 | 正确 |

**中间一行就是陷阱。** 在函数内部看起来像是一个干净的"清空槽"；调用者在调用后看到 `p == nullptr`，以为旧内容已经消失。但它们没有消失 — 它们泄漏了。借用检查器无法看到赋值前 `*slot` 中的内容。

**正确模式 — 与 `safe_swap_arr` 对称**：获取槽的原始视图，将旧值重建为 `_Owned _Nullable` 局部变量，将 `nullptr` 安装到槽中，然后通过标准路径释放局部变量。为每个元素类型编写一次，作为小的原语：

```c
/* 单个对象的通过借用释放原语。 */
_Safe void int_slot_clear(int *_Owned _Nullable *_Borrow slot) {
    int *_Owned _Nullable local;
    _Unsafe {
        int *raw = (int *)&_Mut **slot;            /* 原始视图；槽仍然拥有 */
        local = __take_from_raw(raw);              /* local 取得旧值 */
        *slot = __take_from_raw((int *)nullptr);   /* 将 null 安装到槽中 */
    }
    int_free(local);                               /* 标准释放，空安全 */
}
```

对于 `_Owned _ArrayElem _Nullable` 字段槽，将内置函数换成 `__take_array_from_raw` / `__move_array_to_raw`，并将原始视图换成针对 `T *_Owned _ArrayElem _Nullable *_Borrow` 的 `(T *)&_Mut **slot`。

**为什么这有效**：`(int *)&_Mut **slot` 产生旧值的原始别名，而不告诉借用检查器所有权已被消耗。随后的 `local = __take_from_raw(raw)` 构建一个检查器跟踪的新 `_Owned _Nullable`；`*slot = __take_from_raw(nullptr)` 用已知为空的拥有值覆盖槽。编译器看到：槽被一个跟踪值覆盖；local 现在拥有一个跟踪值；local 必须在作用域结束前被释放 — 这正是 `int_free(local)` 所做的。

### `_Owned *_Borrow` 何时应该出现在 API 中

`T *_Owned *_Borrow` 刻意设计得别扭 — 它只在两种特定场景中占有一席之地：

1. **交换后释放**（§7.6）：替换通过结构体借用访问的 `_Owned _ArrayElem` 字段。
2. **槽清空**（上面 §7.7）：释放在结构体借用中访问的拥有槽的内容。

对于其他任何情况，选择更扁平的签名：

| 意图 | 使用这个，而不是 `T *_Owned *_Borrow` |
|---|---|
| 消耗一个拥有的值 | 按值参数：`void fn(T *_Owned p)` |
| 初始化我指向的对象 | `ensure_init` + 普通借用：`void init(T *_Borrow __attribute__((ensure_init)) out)` |
| 返回一个新分配的对象 | 返回值：`T *_Owned _Nullable fn(void)` |
| 修改字段而不改变所有权 | 普通借用：`void fn(T *_Borrow self)` |
| 替换调用者的整个对象 | 让调用者自己做 `type_free(old); old = type_new();` — 不要接受槽。 |

如果 `T *_Owned _Nullable *_Borrow` 出现在签名中，问问它是否可以被扁平化。通常可以。

## 8. `_ArrayElem`：索引到数组中的拥有指针

第二个限定符 `_ArrayElem` 可以与 `_Owned`（或 `_Borrow` — 见 `borrowing.md`）组合，形成一个指向 **T 的堆分配数组**的指针。

```
T *_Owned _ArrayElem p;   // 拥有 T 的堆数组，支持 p[i]、p == q
```

`_Owned _ArrayElem` 遵循普通 `_Owned` 的所有规则（移动语义、必须释放、不允许与原始类型隐式转换等），有以下区别：

- **允许 `[]` 下标**（`p[i] = ...`、`int x = p[i];`）
- 指针**算术仍然禁止**（`p += 1` 是错误 — 即使对 `_Owned _ArrayElem`）
- 比较（`==`、`!=`）是允许的
- 使用项目本地的元素特定包装器分配/释放（见 §2）。示例：`safe_calloc_chars(n)` 和 `safe_free_chars(buf)`。

- 原始指针转换使用专用的内置函数（不是 `__move_to_raw` / `__take_from_raw`）：
  - `__move_array_to_raw(p)` — 将 `_Owned _ArrayElem` 移出为原始指针
  - `__take_array_from_raw(p)` — 将原始指针作为 `_Owned _ArrayElem` 取得
- **`T *`、`T *_Owned` 和 `T *_Owned _ArrayElem` 之间的 C 风格强制转换被禁止** — 这三个是不同的类别。使用数组内置函数。
- **指向类型不能包含 `_Owned` 成员**（与适用于普通 `_Owned` 数组的限制相同）。
- `_ArrayElem` 不能修饰**原始**指针；它只能附加到 `_Owned` 或 `_Borrow`。

```c
_Safe int main(void) {
    char *_Owned _ArrayElem _Nullable p = safe_calloc_chars(10);  // [0]..[9] = 0
    if (p == nullptr) return 1;
    char *_Owned _ArrayElem buf = p;        /* 缩小 nullable → nonnull */
    buf[3] = 'x';                           // 可以：下标
    // buf += 1;                            // 错误：不允许算术运算
    safe_free_chars(buf);
    return 0;
}
```

**`_Safe` / `_Unsafe` 互操作**：兼容性规则将 `_Owned _ArrayElem` 和 `_Borrow _ArrayElem` 视为**完整的限定符** — `_Safe` 重声明可以向未注释的 `_Unsafe` 参数添加 `_Owned _ArrayElem`，但你不能在声明之间从 `_Owned` "升级"为 `_Owned _ArrayElem`，或在 `_Owned` 和 `_Owned _ArrayElem` 之间移动。

### 陷阱：不能通过借用使用 `__move_array_to_raw`

在 `s` 是 `struct T *_Borrow` 的情况下调用 `__move_array_to_raw(s->field)` 即使在 `_Unsafe` 内部也会被拒绝。借用不授予移动权限；转移字段会使结构体的拥有槽悬空，而编译器不知道。

```c
_Safe void bad(struct String *_Borrow s) {
    _Unsafe {
        /* 错误：不能通过借用移动 _Owned _ArrayElem 字段 */
        char *raw = __move_array_to_raw(s->data);
    }
}
```

`(T *)&_Mut *s->data`（原始视图转换）的编译器错误会建议"使用 `__move_array_to_raw`"。接下来尝试这个也会由于同样的原因被拒绝。两种形式都通过借用被阻止。

**三种替代方案：**

1. **通过 `_Nonnull` 局部变量的原始视图转换** — 用于读/写访问而不转移所有权（memcpy、memcmp 参数）。将可为空的字段缩小到局部变量，然后使用 `(T *)&_Mut *local`：

   ```c
   if (s->data == nullptr) return;
   char *_Owned _ArrayElem local = s->data;   /* 缩小 */
   _Unsafe { char *raw = (char *)&_Mut *local; memcpy(dst, raw, n); }
   ```

2. **交换模式**（§7.6）— 通过 `safe_swap_arr` 原子地替换字段的内容。旧缓冲区落入 donor 的槽中，可以在那里释放。不需要将内容移出结构体。

3. **按值结构体移动** — 当函数按值接受结构体时（消耗它），`s.data` 是一个具有所有权的左值；`__move_array_to_raw` 随后被允许：

   ```c
   _Safe void str_free(struct String s) {   /* s 是局部副本 — 拥有 data */
       _Unsafe { char *raw = __move_array_to_raw(s.data); free(raw); }
   }
   ```

   这是按值析构函数中使用的模式。

> 另见：`borrowing.md`、`safe-zone.md`、`nullability.md`、`design.md`、`errors.md` 技能
