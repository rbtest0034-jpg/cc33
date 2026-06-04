
# BiSheng C 可空性技能

## 1. 概述

BSC 在编译时跟踪指针可空性。编译器防止解引用或通过可能为空的指针访问成员。

- `_Nonnull` — 指针保证非空（`_Owned` 和 `_Borrow` 的默认值）
- `_Nullable` — 指针可能为空（原始指针的默认值）
- `nullptr` — 空指针字面量（在安全区域中替换 `NULL`）

## 2. 默认可空性

| 指针类型 | 默认 | 覆盖 |
|---|---|---|
| 原始指针（`int *`） | Nullable | `int *_Nonnull` |
| `_Owned` 指针（`int *_Owned`） | Nonnull | `int *_Owned _Nullable` |
| `_Borrow` 指针（`int *_Borrow`） | Nonnull | `int *_Borrow _Nullable` |

```c
// 可为空指针：
int *_Nullable p1 = nullptr;
int *_Borrow _Nullable p2 = nullptr;
int *_Owned _Nullable p3 = nullptr;
int *p4 = nullptr;                    // 原始指针默认为 Nullable

// 非空指针：
int *_Nonnull p5 = &a;
int *_Borrow p6 = &_Mut a;           // _Borrow 默认为 Nonnull
int *_Owned p7 = safe_calloc_int(1); // _Owned 默认为 Nonnull
```

### 实用注释规则

对于 `_Owned` 和 `_Borrow` 指针，**只写 `_Nullable`** — 非空是默认值，因此 `_Nonnull` 是多余的，应该省略。

对于原始指针，当参数或字段必须非空时**只写 `_Nonnull`** — 默认为 `_Nullable`，因此没有注释就意味着可为空。

```c
/* _Owned / _Borrow — 只有例外需要注释 */
char *_Owned _ArrayElem _Nullable data;   /* 显式可为空：标注 */
char *_Owned _ArrayElem buf;              /* 默认非空：无注释 */
const struct String *_Borrow s;           /* 默认非空：无注释 */

/* 原始指针 — 只有例外需要注释 */
const char *_Nonnull src;   /* 必须非空：标注 */
const char *dst;            /* 默认可为空：无注释 */
```

**自我检查**：如果你写了 `*_Borrow _Nonnull` 或 `*_Owned _Nonnull`，移除 `_Nonnull`。如果你写了一个原始的 `T *` 参数，且被调用者将无条件解引用它，添加 `_Nonnull`。

## 3. 可跟踪的指针

编译器只能跟踪满足以下条件的指针的可空性：
1. 是**左值**（有内存地址）
2. 不是 `volatile`
3. 不是通过数组下标（`[]`）获得的

```c
_Safe void test(int *_Borrow _Nullable p, int *_Borrow _Nullable volatile vp) {
    int *_Borrow _Nullable q = nullptr;
    int *_Borrow _Nullable arr[2] = {nullptr, &_Mut local};

    if ((q = p) != nullptr) {
        *q = 1;  // 可以：q 是可跟踪的左值
    }
    if (identity(p) != nullptr) {
        *identity(p) = 3;  // 错误：函数返回值不是左值
    }
    if (vp != nullptr) {
        *vp = 4;  // 错误：volatile 指针不被跟踪
    }
    if (arr[1] != nullptr) {
        *arr[1] = 5;  // 错误：数组下标不被跟踪
    }
}
```

### 变通方法：通过可跟踪的局部变量复制

当你需要解引用一个不可跟踪的可空指针时，首先将其绑定到一个**新的局部变量**（因为它是一个普通左值而可跟踪），然后对局部变量进行空检查和解引用：

```c
_Safe void use_untrackable(int *_Borrow _Nullable p, int *_Borrow _Nullable arr[]) {
    int *_Borrow _Nullable t1 = identity(p);  // 可跟踪的临时变量
    if (t1 != nullptr) { *t1 = 3; }           // 可以

    int *_Borrow _Nullable t3 = arr[1];       // 可跟踪的临时变量
    if (t3 != nullptr) { *t3 = 5; }           // 可以
}
```

临时变量必须是局部变量 — 不能是结构体成员，不能是另一个数组元素。这是你在面对不可跟踪的可空指针时推荐使用的模式。

## 4. 可空性状态变化

对于**标记为 Nonnull** 的指针：状态始终是 Nonnull。如果进行了空检查，空分支会将其视为空（对 `_Owned` 泄漏预防有用）：

```c
_Safe int test(void) {
    int *_Owned _ArrayElem _Nullable a = safe_calloc_int(1);  // 检查前为 Nullable
    if (a != nullptr) {
        safe_free((void *_Owned)a);
        return 1;
    }
    return 0;  // 可以：无泄漏错误（此分支中 a 被视为空）
}
```

对于**标记为 Nullable** 的指针，状态通过以下方式变化：
1. **使用非空表达式赋值** → 变为 Nonnull
2. **控制流中的空检查** → 在非空分支中变为 Nonnull

```c
_Safe void test(void) {
    int *_Borrow _Nullable p1 = nullptr;    // Nullable
    *p1 = 10;                                // 错误！

    int local = 10;
    p1 = &_Mut local;                        // → Nonnull（非空赋值）
    *p1 = 20;                                // 可以

    p1 = foo(&_Mut local);                   // → Nullable（foo 返回 _Nullable）
    *p1 = 20;                                // 错误！

    p1 = bar(&_Mut local);                   // → Nonnull（bar 返回 Nonnull）
    *p1 = 20;                                // 可以

    int *_Borrow _Nullable p2 = foo(&_Mut local);
    if (p2 != nullptr)
        *p2 = 10;  // 可以：true 分支中为 Nonnull
    else
        *p2 = 20;  // 错误：else 分支中为 Nullable
}
```

### 特例：`_Owned _ArrayElem _Nullable` — 缩小需要重新赋值

对于普通的 `_Owned _Nullable` 或 `_Borrow _Nullable` 指针，`if (p != nullptr)` 分支将 `p` 缩小为 `_Nonnull`，允许解引用和强制转换。对于 `_Owned _ArrayElem _Nullable`，这种缩小**不会**传播到 `__move_array_to_raw` 或 `(T *)&_Mut *p` 原始视图强制转换：

```c
_Safe void bad(char *_Owned _ArrayElem _Nullable buf) {
    if (buf != nullptr) {
        /* 在 if 分支内部仍然被拒绝： */
        _Unsafe { char *raw = __move_array_to_raw(buf); free(raw); }
        /* 错误：不能对 _Nullable 指针调用 __move_array_to_raw */
    }
}
```

**修复**：在空检查后立即重新赋值给一个新声明的 `_Nonnull` 局部变量。赋值本身执行缩小，后续对局部变量的数组操作成功：

```c
_Safe void safe_free_chars(char *_Owned _ArrayElem _Nullable buf) {
    if (buf == nullptr) return;
    char *_Owned _ArrayElem nonnull_buf = buf;   /* 通过重新赋值缩小 */
    _Unsafe {
        char *raw = __move_array_to_raw(nonnull_buf);
        free(raw);
    }
}
```

同样的规则适用于用于借用（非所有权转移）的原始视图强制转换：

```c
/* 如果你只需要原始视图（非所有权转移）： */
_Safe void use_buf(char *_Owned _ArrayElem _Nullable buf) {
    if (buf == nullptr) return;
    char *_Owned _ArrayElem nonnull = buf;   /* 先缩小 */
    _Unsafe { char *raw = (char *)&_Mut *nonnull; memcpy(dst, raw, n); }
}
```

**注意**：`(T *)&_Mut *buf` 的编译器错误会建议使用 `__move_array_to_raw`。接下来尝试这个也会由于同样的原因被拒绝。两种形式都需要 `_Nonnull` 局部变量缩小 — 没有捷径。

**总结规则**：对于 `_Owned _ArrayElem _Nullable`，在空检查后始终通过 `T *_Owned _ArrayElem local = nullable_ptr;` 缩小。不要依赖 if 检查缩小来进行数组强制转换或移动操作。

## 5. 空检查模式

在 `if`/`while`/三元运算符的条件中，以下模式构成空检查：

1. **直接指针**：`if (p)` / `while (s.p)`
2. **逻辑运算符**：`if (!p)` / `if (p && q)` / `if (p || q)`
3. **显式比较**：`if (p != nullptr)` / `if (nullptr == p)`
4. **带赋值**：`if ((q = p) != nullptr)` — 只检查 `q`
5. **在逗号表达式中**：`if ((x, p != nullptr))` — 只检查最后一项
6. **括号**：以上任何都可以嵌套在括号中

状态更新：
- `if (e)` / `if (e != nullptr)`：`e` 在 true 分支中为 Nonnull
- `if (!e)` / `if (e == nullptr)`：`e` 在 false/else 分支中为 Nonnull
- `if (p && q)`：两者都在 true 分支中为 Nonnull
- `if (!p || !q)`：两者都在 else 分支中为 Nonnull

## 6. 赋值、传递和返回规则

```c
// 不能将可为空的值赋值给 Nonnull 指针：
int *_Borrow p1 = nullptr;          // 安全区域中的错误
int *_Borrow p2 = foo(&_Mut local); // 如果 foo 返回 _Nullable 则为错误

// 不能将可为空的参数传递给 Nonnull 参数：
_Safe void bar(int *_Borrow p) {}
bar(nullable_ptr);  // 错误

// 不能从 Nonnull 返回类型返回可为空：
_Safe int *_Borrow return_nonnull(int *_Borrow p) {
    int *_Borrow _Nullable q = nullptr;
    return q;  // 错误
}
```

### 原始 `const char*` 参数和 C 字符串返回类型

原始指针参数默认为 `_Nullable`。在 `_Safe` 函数中对可为空的指针进行下标访问或解引用是编译错误：

```c
// 错误 — s 默认为 _Nullable；s[i] 是空不安全的解引用
_Safe size_t my_strlen(const char* s) {
    size_t n = 0;
    while (s[n]) { n++; }  // 错误：可为空的指针不能被解引用
    return n;
}

// 正确 — 声明为 _Nonnull；调用者在调用点保证非空
_Safe size_t my_strlen(const char* _Nonnull s) {
    size_t n = 0;
    while (s[n]) { n++; }  // 可以
    return n;
}
```

相同的问出现在**返回位置**，当可为空的返回值送入 `_Nonnull` 参数时：

```c
// 默认返回可为空
const char* mime_for_ext(const char* _Nonnull ext);

// 参数是 _Nonnull
_Safe Response make_response(String body, const char* _Nonnull ct);

// 错误 — "不能将可为空的指针参数传递给非空参数"
_Safe Response f(const char* _Nonnull path) {
    return make_response(body, mime_for_ext(path));
}

// 修复 — 当函数总是返回有效指针时声明返回为 _Nonnull
const char* _Nonnull mime_for_ext(const char* _Nonnull ext);
```

**实用规则**：任何对 `const char*` 参数进行下标访问的 `_Safe` 函数必须将其声明为 `const char* _Nonnull`。任何返回值被无条件传递给 `_Nonnull` 参数的函数应将其返回声明为 `const char* _Nonnull` — 前提是它确实从不返回空值。

### 与 `_Nonnull` 返回类型的初始化死锁

当返回值只能在 `_Unsafe` 块内部计算时（因为它需要原始指针强制转换），安全区域初始化规则强制你在 `_Unsafe` 块之前声明变量 — 使用 `nullptr` 占位符。`_Nonnull` 类型的变量不能使用 `nullptr` 初始化，因此声明的类型必须是原始类型（隐式为 `_Nullable`），这随后使 return 语句无法通过 `_Nonnull` 返回类型检查：

```c
/* 死锁 */
_Safe const char *_Nonnull cstr_view(const char *_Borrow _ArrayElem buf) {
    const char *_Nonnull p = nullptr;     /* 错误：nullptr 不是 _Nonnull */
    _Unsafe { p = (const char *)&_Const *buf; }
    return p;
}
/* --- 或者，如果声明为原始类型： --- */
_Safe const char *_Nonnull cstr_view(const char *_Borrow _ArrayElem buf) {
    const char *p = nullptr;              /* 原始/可为空：可以用 nullptr 初始化 */
    _Unsafe { p = (const char *)&_Const *buf; }
    return p;                             /* 错误：不能从 _Nonnull 返回可为空 */
}
```

**解决方案**：使用原始（隐式 `_Nullable`）返回类型，并在函数注释中记录非空契约。在调用点，需要 `_Nonnull` 类型的调用者可以在空检查后通过局部重新赋值缩小 — 但实际上，输出总是非空的 `_Borrow` 输入函数只会按原样使用，基于 `_Borrow`（非空输入）产生非空输出的理解。

```c
/* 可以 — 原始返回类型；契约通过文档记录 */
/* 返回 buf 的原始视图。当 buf 是有效的 _Borrow
 *（默认非空）时，结果始终非空。 */
_Safe const char *safe_cstr_view(const char *_Borrow _ArrayElem buf) {
    const char *p = nullptr;
    _Unsafe { p = (const char *)&_Const *buf; }
    return p;
}
```

每当需要 `_Nonnull` 返回类型但安全区域的初始化规则使其在机械上不可实现时，这是标准的解决方案。

## 7. 类型强制转换

使用 `-nullability-check=all`，在非安全区域中从 Nullable 到 Nonnull 的强制转换也会被检查：

```c
void foo() {
    int *p1 = nullptr;
    int *p2 = (int *_Nonnull)p1;   // 错误：nullable 到 nonnull 的强制转换
    int *_Owned p3 = (int *_Owned)p1; // 错误
}
```

在空检查之后，允许强制转换：

```c
void foo() {
    int *p1 = nullptr;
    if (p1 != nullptr) {
        int *_Nonnull p2 = (int *_Nonnull)p1;  // 可以
        int *_Owned p3 = (int *_Owned)p1;       // 可以
    }
}
```

## 8. 结构体成员

```c
struct Data { int *_Borrow _Nullable value; };

_Safe void test(void) {
    int local = 10;
    // 初始化列表：从初始化器推断可空性
    struct Data data1 = {.value = bar(&_Mut local)};  // Nonnull
    *data1.value = 10;  // 可以

    // 非初始化列表：默认为 Nullable
    struct Data data2 = init_data(&_Mut local);  // Nullable
    *data2.value = 10;  // 错误

    // 修复：重新赋值或空检查
    data2.value = bar(&_Mut local);  // → Nonnull
    *data2.value = 10;               // 可以

    if (data3.value != nullptr)
        *data3.value = 10;           // 可以
}
```

## 9. 编译器选项

`-nullability-check=<mode>`：

| 模式 | 行为 |
|---|---|
| `safeonly`（默认） | 仅在 `_Safe` 区域中检查 |
| `all` | 在所有代码中检查（安全和非安全） |

不带此选项时，行为等同于 `-nullability-check=safeonly`。

## 10. `-Wnullability-completeness` — 全有或全无的注释规则

一旦头文件中的任何指针带有显式的可空性注释，Clang 就会对同一作用域中缺少注释的每个其他指针发出警告。规则是**每文件全有或全无**：要么注释每个指针，要么不注释任何指针。

**什么会触发它：**

```c
/* 一个 _Nullable 字段就足以触发对同一头文件中所有其他未注释指针的警告。 */
struct String {
    char *_Owned _ArrayElem _Nullable data;   /* 显式 — 触发规则 */
    size_t len;                               /* 不是指针 — 没问题 */
};

_Safe size_t str_len(const struct String *_Borrow s);
/*                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^
 *  警告：指针缺少可空性类型说明符
 *  （_Borrow 默认非空，但完整性检查现在
 *   要求所有已经存在注释的地方都显式注释） */
```

**错误的修复 — `#pragma clang assume_nonnull begin/end`：**

该 pragma 通过使区域内每个未注释的指针变为 `_Nonnull`（包括你打算使其为 `_Nullable` 的原始指针，例如可选参数）来静默警告。该 pragma 更改了语义，而不仅仅是警告。

**正确的修复 — 显式注释每个指针**，即使注释与默认值相同：

```c
/* 在向一个字段添加 _Nullable 后，显式注释每个其他
   _Borrow 和 _Owned（即使它们的默认值是非空）。
   有意为 _Nullable 的原始返回类型不需要注释，
   因为原始指针的默认值已经是 _Nullable。 */
_Safe char *_Owned _ArrayElem _Nullable safe_calloc_chars(size_t n);
_Safe void  safe_free_chars(char *_Owned _ArrayElem _Nullable buf);
_Safe void  safe_swap_arr(char *_Owned _ArrayElem *_Borrow left,
                          char *_Owned _ArrayElem *_Borrow right);
_Safe void  safe_memcpy_into(char *_Borrow _ArrayElem dst,
                             const char *_Nonnull src, size_t n);
_Safe const char *safe_cstr_view(const char *_Borrow _ArrayElem buf);
/*           ^^^^ 原始返回；默认为 _Nullable — 不需要注释 */
```

**经验法则**：一旦头文件有任何 `_Nullable`，遍历该头文件中的每个指针，添加其所需的显式注释。对于必须非空的 `_Borrow`/`_Owned` 参数，不添加任何内容（默认）。对于必须非空的原始参数，添加 `_Nonnull`。对于所有 `_Nullable` 的，添加 `_Nullable`。

> 关于 ownership 和 `_Owned _Nullable`，见 `ownership.md` 技能
> 关于 borrowing 和 `_Borrow _Nullable`，见 `borrowing.md` 技能
> 关于安全区域，见 `safe-zone.md` 技能
> 关于可空性错误（BSC-E04xx），见 `errors.md` 技能
