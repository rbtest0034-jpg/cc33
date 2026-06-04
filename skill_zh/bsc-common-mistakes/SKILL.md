---
name: bsc-common-mistakes
description: 受限子集中 BiSheng C 的常见错误和修复。当遇到与 ownership、borrowing、safe zones 或 nullability 相关的编译错误或调试陷阱时使用。
---

# BiSheng C 常见错误技能

## 子集横幅 — 先阅读

本项目子集禁用了 `_Owned struct`、泛型、成员函数、`_Trait` 和 libcbs。下面提到 `safe_malloc<T>`、`_Owned struct`、`Vec<T>`、`String`、`bishengc_safety.hbs` 或 trait 分发的示例是底层错误的**遗留插图**；相同的问题在子集中也存在，只需替换为等效的子集模式（见 `/bsc`（references/overview.md）"不在本子集中"）。本技能中的诊断消息和核心修复仍然适用。

## 1. 安全区域错误

### 1.1 在安全区域中使用 `&` 而不是 `&_Const`/`&_Mut`
```c
_Safe void f(void) {
    int x = 42;
    // int* p = &x;                    // 错误：'&' 在安全区域中禁止
    const int *_Borrow p = &_Const x;  // 正确
}
```

### 1.2 将 `++`/`--` 结果用作表达式
`++`/`--` 允许作为语句，但返回 `void` — 不能使用其结果。
```c
_Safe void f(void) {
    int i = 0;
    i++;                               // 可以：仅副作用
    for (int j = 0; j < 10; j++) {}    // 可以：迭代子句
    // int x = i++;                    // 错误：结果类型为 void
}
```

### 1.3 空参数不带 `void`
```c
// _Safe int f() { return 42; }     // 错误
_Safe int f(void) { return 42; }    // 正确
```

### 1.4 带指针字段的结构体部分初始化
只有带指针字段的结构体才需要完整初始化器。仅基本类型的结构体可以部分初始化。
```c
_Safe {
    struct HasPtr hp = {nullptr, 0};  // 可以：完整初始化（有指针字段）
    // struct HasPtr hp2 = {0};       // 错误：部分初始化
    struct NoPtr { int a; int b; };
    struct NoPtr np = {0};            // 可以：没有指针字段
}
```

### 1.5 安全区域中的联合体成员访问
联合体可以声明/初始化/传递，但成员访问被禁止。
```c
_Safe {
    union U { int i; float f; } u = {.i = 1};
    // int x = u.i;                   // 错误
    _Unsafe { int x = u.i; }          // 可以：不安全逃逸
}
```

### 1.6 安全区域中禁止的强制类型转换
不允许跨类别指针转换（`_Owned`/`_Borrow`/raw）、指针-整数转换、浮点-整数转换。例外：`T *_Owned` 到 `void *_Owned` 是允许的。

### 1.7 安全区域中对全局变量的可变借用
只允许对全局变量使用 `&_Const`；禁止对全局变量使用 `&_Mut`。
```c
int g = 10;
_Safe void f(void) {
    // int *_Borrow p = &_Mut g;          // 错误
    const int *_Borrow p = &_Const g;     // 可以
}
```

## 2. Ownership 错误

### 2.1 `_Owned`/`_Borrow` 限定符位置错误
**#1 LLM 错误。** `_Owned`/`_Borrow` 放在 **`*` 之后**，而不是类型之前。
```c
// 错误                              // 正确
// _Owned char* msg                   char *_Owned msg = safe_malloc('\0');
// _Borrow int* r                     const int *_Borrow r = &_Const x;
```
模式：始终是 `T *_Owned` 和 `T *_Borrow`，绝不能用 `_Owned T*` 或 `_Borrow T*`。

### 2.2 所有权转移后使用变量
```c
int *_Owned p = safe_malloc(42);
int *_Owned q = p;        // p 被转移
// printf("%d\n", *p);    // 错误：使用了已转移的值
```

### 2.3 忘记在作用域结束前释放 `_Owned`
```c
void f(void) {
    int *_Owned p = safe_malloc(42);
    safe_free((void *_Owned)p);   // 必须释放：free、传递或返回
}
```

### 2.4 对 `_Owned` 进行指针算术
```c
int *_Owned p = safe_malloc(42);
// p++;                    // 错误：不允许对 _Owned 进行算术运算
// p[3] = 0;               // 错误：不允许对普通 _Owned 使用 []
```

对于希望使用下标的堆**数组**，请使用 `T *_Owned _ArrayElem`（用 `safe_malloc_array` 分配，用 `safe_free_array` 释放）。它支持 `p[i]` 但仍禁止算术运算——`_Owned _ArrayElem` **不是** C 风格数组指针的自由升级。见 `/bsc`（references/ownership.md）§8。

```c
int *_Owned _ArrayElem arr = safe_malloc_array(10, 0);
arr[3] = 3;                            // 可以：_ArrayElem 允许下标
// arr += 1;                           // 错误：仍然不允许算术运算
safe_free_array((void *_Owned _ArrayElem)arr);
```

## 3. Borrowing 错误

### 3.1 在不可变借用活跃时进行可变借用
```c
int x = 42;
{
    const int *_Borrow r = &_Const x;
    printf("%d\n", *r);
}
int *_Borrow mr = &_Mut x;  // 可以：不可变借用已结束
```

### 3.2 借用比源对象存活更久
```c
// const int *_Borrow dangling(void) {
//     int x = 42;
//     return &_Const x;  // 错误：借用比局部变量存活更久
// }
```

### 3.3 返回依赖于局部辅助值的借用

最常见的所有权模型冲突之一。你计算一个局部 `String`（或其他拥有的值）用作键，然后想返回通过它查找得到的借用。

```c
// 错误 — `key` 在作用域结束时销毁，但返回的借用传递性地依赖于它
_Safe JSON_Value* _Borrow lookup(Obj* _Borrow this, const char* k) {
    String key = String::from(k);
    JSON_Value* _Borrow child = this->get_value(&_Const key);
    return child;   // 错误：`key` 存活时间不够长
}
```

**三种合法的修复方式，按偏好顺序：**

**修复 A — 从调用者那里以借用方式获取键**（最佳）。调用者拥有 `String`，其生命周期覆盖调用：
```c
_Safe JSON_Value* _Borrow lookup(Obj* _Borrow this, const String* _Borrow key) {
    return this->get_value(key);
}
```

**修复 B — 递归**，当辅助值在遍历的每一步产生时使用。将路径的剩余部分传递给递归调用；辅助值的生命周期只限于一个递归层级：
```c
_Safe static T* _Borrow navigate(Parent* _Borrow p, const Path* _Borrow path, size_t pos) {
    String segment = path->slice(pos, ...);   // 局部变量；返回时销毁
    T* _Borrow child = p->get(&_Const segment);
    // 尾递归携带 child 的生命周期，而非 segment 的
    return navigate_further(child, path, next_pos);
}
```

**修复 C — 最小 `_Unsafe` 块**，当查找确实有一个检查器无法建模的生命周期时使用（例如，哈希表命中保证返回指向现有数据的指针）。通过原始指针进行强制转换：
```c
_Safe JSON_Value* _Borrow lookup(Obj* _Borrow this, const char* k) {
    String key = String::from(k);
    _Unsafe {
        JSON_Value* _Borrow child = (JSON_Value* _Borrow)this->get_value(&_Const key);
        // `key` 在块结束时销毁，但返回的借用指向 `this`，
        // 而非 `key` — 检查器看不到这一点。
        return child;
    }
}
```

仅当修复 A 和 B 在结构上不可能时使用修复 C。记录**原因**——下一个阅读代码的人会问。这是 parson 中 `dotget_value` 使用的模式。

### 3.4 如何阅读 BSC 借用检查器诊断

BSC 编译器的借用诊断比初看起来信息更丰富。**始终阅读 `note:` 行，而不仅仅是 `error:` 行**——`note:` 才是提示所在。

**借用冲突（别名）：**
```
file.cbs:22:29: error: cannot borrow `b` as mutable more than once at a time
file.cbs:21:29: note: first mut borrow occurs here
```
`note:` 确切地告诉你哪一行之前开始了冲突的借用。修复：缩小第一个借用的作用域（将其包装在一个在第二个借用之前结束的块中），或重新组织代码，使每次只有一个借用存活。

**跨冲突修改的借用：**
```
error: cannot use `X` because it was mutably borrowed
error: cannot borrow `*X` as mutable more than once at a time
```
同样的思路 — `note:` 指向第一个借用。典型修复：缩小第一个借用的作用域，或以不同的顺序执行操作。

**转移后使用（目前没有 `note:` — 已知空白）：**
```
file.cbs:12:13: error: use of moved value: `b`
```
编译器目前不打印 `note:` 说明 `b` 在哪里被转移。使用 LSP `hover` 在 `b` 的声明上查看其完整的所有权流（它会显示 `Moved into foo()` 并附带确切行号）。见 `/bsc`（references/compile.md）§5 了解 LSP 配置。

**生命周期 / 借用返回：**
```
error: no _Borrow qualified type found in the function parameters,
       the return type is not allowed to be _Borrow qualified
```
这直接解释了规则 — 函数返回 `_Borrow` 但没有 `_Borrow` 参数可以将返回值的生命周期绑定到它。修复：添加一个生命周期与返回的借用匹配的 `_Borrow` 参数，或改为返回一个拥有的值。

**使人困惑的不明显消息：**
- `does not live long enough` — 编译器对生命周期比上下文要求短的局部变量的称呼（通常是返回的指向局部变量的 `_Borrow`）。见 §3.3 了解三种修复方式。
- `cannot cast between _Owned and raw pointer` — 你需要使用 `__take_from_raw` / `__move_to_raw` 进行所有权转移，或使用 `(T *)&_Mut *p` / `(T *)&_Const *p` 进行非转移强制转换。

**当诊断仍然不够时，使用 LSP hover：**

悬浮在抱怨的变量上 — BSC 的 LSP（配置后；见 `/bsc`（references/compile.md）§5）报告其**完整的所有权时间线，包括活跃范围**：

```
Ownership Flow:
line 21: Declared
line 21: Mut borrow of b
Live range: lines 21-21
```

对于拥有的值，hover 还显示 `Moved into foo()` 并附带行号。这是转移后使用诊断应该打印但没有打印的信息 — LSP 填补了这个空白。

### 3.5 在同一个调用中读取结构体字段同时对结构体进行可变借用

**问题：** 函数接受 `struct T *_Borrow`（对结构体的可变借用），其中一个整数参数是从同一结构体的字段计算出来的。借用检查器在可变借用活跃期间冻结整个结构体——包括其所有普通字段。即使是读取一个 `size_t` 字段也会被拒绝。

```c
/* str__ensure_cap 接受 struct String *_Borrow 作为第一个参数 */
static _Safe void str__ensure_cap(struct String *_Borrow s, size_t want);

_Safe void str_reserve(struct String *_Borrow s, size_t extra) {
    /* 错误 — 在同一个调用中 s->len 被读取时 s 正被可变借用 */
    str__ensure_cap(s, s->len + extra + 1);
    /*              ^  ^^^^^^^
     * error: cannot use (*s).len because it was mutably borrowed
     * （对第一个参数的 s 的可变借用冻结了 s->len） */
}
```

**修复：** 在执行接受可变借用的调用**之前**，将任何字段读取提取到局部变量中。

```c
_Safe void str_reserve(struct String *_Borrow s, size_t extra) {
    size_t want = s->len + extra + 1;   /* 先读取 s->len，在任何借用之前 */
    str__ensure_cap(s, want);           /* 可变借用从这里开始；不读取 s->len */
}
```

**为什么会发生：** 在函数调用 `f(a, b)` 中，编译器在调用开始之前评估所有参数（包括获取借用）。当参数 0 对 `s` 产生 `*_Borrow` 时，参数 1 对 `s->len` 的访问就冲突了——在读取 `s->len` 时借用已经活跃，即使该读取在逻辑上早于 `f` 内部的写入。将字段读取提取到调用表达式之前的语句中，就将其移到了借用生命周期之外。

**要识别的诊断：** `cannot use (*s).<field> because it was mutably borrowed`。如果字段读取和 `*_Borrow` 参数在同一个调用表达式中，这总是原因。

## 4. Trait 错误

### 4.1 非指针 trait 变量
```c
// _Trait Printable obj;       // 错误：只能使用指针形式
_Trait Printable* obj = &val;  // 正确
```

### 4.2 缺少 `struct` 关键字
```c
// void S::method(S* this) { ... }               // 错误（除非使用 typedef）
void struct S::method(struct S* this) { ... }     // 正确
```

### 4.3 方法定义前使用 `_Impl`
```c
// 先定义方法，再使用 _Impl
void struct Circle::print(struct Circle* this) { ... }
_Impl _Trait Printable for struct Circle;
```

## 5. Nullability 错误

### 5.1 `_Owned` 指针可以为 null 但不带 `_Nullable`
```c
// int *_Owned p = nullptr;               // 错误：_Owned 默认为 Nonnull
int *_Owned _Nullable p = nullptr;        // 正确
```

### 5.2 不进行空检查就解引用可为空的指针
```c
_Safe void f(int *_Borrow _Nullable p) {
    // *p = 10;                           // 错误：可为空的指针
    if (p != nullptr) { *p = 10; }       // 正确：先进行空检查
}
```

### 5.3 将可为空的指针传递给非空参数
```c
_Safe void bar(int *_Borrow p) {}     // 非空参数
_Safe void f(int *_Borrow _Nullable p) {
    // bar(p);                          // 错误：可为空传递给非空
    if (p != nullptr) { bar(p); }      // 正确：已检查
}
```

## 6. 初始化错误

### 6.1 在初始化之前使用变量
```c
_Safe void f(void) {
    int x;
    // int y = x;                       // 错误：未初始化
    x = 42;
    int y = x;                          // 正确
}
```

### 6.2 逐个元素赋值数组不计为初始化
```c
_Safe void f(void) {
    int arr[3];
    arr[0] = 1; arr[1] = 2; arr[2] = 3;
    // int x = arr[0];                  // 错误：arr 不被认为已初始化
    // 修复：使用初始化列表或 __assume_initialized
    int arr2[3] = {1, 2, 3};
    int x = arr2[0];                    // 正确
}
```

### 6.3 取未初始化变量的地址
```c
_Safe void f(void) {
    int x;
    // int *_Borrow p = &_Mut x;       // 错误：x 未初始化
    int x2 = 0;
    int *_Borrow p = &_Mut x2;         // 正确
}
```

## 7. 异步错误

### 7.1 二元表达式中的 `_Await`
```c
// int result = _Await compute(1) + _Await compute(2);  // 错误
int a = _Await compute(1);
int b = _Await compute(2);
int result = a + b;
```

### 7.2 同一参数列表中的多个 `_Await`
```c
// f(_Await g(), _Await h());          // 错误：同一级别的多个 _Await
int a = _Await g();
f(a, _Await h());                      // 正确：预先评估一个
```

## 8. 调试运行时内存错误

当 BSC 代码编译通过但在运行时**重复释放**或**使用已释放的内存**时，在怀疑编译器之前按照此分类流程进行。

### 8.1 使用 valgrind 而非 gdb 定位重复释放

glibc 的 `free(): double free detected in tcache 2` 在 gdb 下只提供第二次释放的堆栈跟踪。Valgrind 显示两次释放和原始的 `malloc`，这才是你需要的查找别名根本原因的信息。

```bash
valgrind --error-exitcode=1 --leak-check=no ./binary
```

查找"Invalid read"/"Invalid free"报告。"Address X is N bytes inside a block of size M free'd"行告诉你同一地址在何处被更早释放，以及那次更早释放时的堆栈。

### 8.2 在指责编译器之前排除测试顺序污染

如果函数独立工作时正常（独立的可重现二进制文件），但在更大套件中在其他测试之后调用时失败，那么 bug 是**状态污染**，而非代码生成：

- **可变全局变量**（例如，解析器游标、分配计数器）被先前的测试留在了非零状态
- **库中的静态缓存**在调用之间没有重置
- **堆布局敏感性** — 测试 N 中的泄漏仅在分配大小恰好发生别名时，在测试 N+M 中表现为重复释放

重现方式：构建一个最小的 `main`，在全新的进程中**只调用失败的函数**。如果通过，则 bug 是上下文相关的。

### 8.3 在声称是编译器 bug 之前检查脱糖后的 AST（驱动器模式）

大多数关于析构函数重复释放的"编译器 bug"假设最终被证明是错误的。使用以下方式验证：

```bash
clang -Xclang -ast-dump -fsyntax-only file.cbs -I./include
```

查找 `varname_is_moved` 标记和 `if (!varname_is_moved) ~Type(varname)` IfStmt。如果编译器的机制完整，那么 bug 在于你库中的堆指针别名。

**不要为此使用 `clang -cc1 -fsyntax-only`** — 它缺少系统包含，这使得在 AST 中每个 `_Owned struct` 都虚假地"无效"。见 `/bsc`（references/compile.md）技能 §6 了解详情。

### 8.4 重复释放的常见库级别原因

当编译器行为正确时，真正原因通常是以下之一：

- **作为联合体的标签结构体**：每个实例为每个变体携带堆指针，因此如果构造无意中共享指针，两个实例可能产生别名。见 `/bsc`（references/design.md）规则 7 和 §2 "替换 C 联合体"。
- **别名指针之间的 `safe_swap`**：交换使双方指向重叠的所有权。
- **返回底层拥有的值已被调用者转移的借用**：借用变成了悬空指针。
- **手动 `_Unsafe` 字节赋值覆盖 `_Owned` 槽而不析构先前内容**：旧的堆指针泄漏（单次复制）或别名（如果它们刚刚被 `memmove` 移入）。

### 8.5 分类流程总结

1. 在 valgrind 下重现 → 获取两个释放位置和 malloc 来源。
2. 独立测试 → 确认/否认测试顺序污染。
3. 驱动器模式 AST 转储 → 确认编译器析构函数插入正确。
4. **然后**才考虑编译器级别 bug。在实践中，BSC 项目中 >90% 的重复释放可追溯到库级别的别名，而非代码生成。

### 8.6 已知编译器 bug：`if`/`while`/`for` 条件中的 `_Owned` 参数不进行转移跟踪

BSC 析构函数脱糖过程（`SemaBSCDestructor.cpp::VisitCompoundStmt`）只对**外部**类型为 `BinaryOperator`、`DeclStmt` 或 `CallExpr` 的语句运行转移跟踪。当消耗 `_Owned` 参数的函数调用出现在 **`if`/`while`/`for`/`switch` 条件内部**时，外部语句是 `IfStmt` 等，因此转移跟踪被完全跳过。`is_moved` 标记永远不会被设置为 1，析构函数在作用域结束时对已释放的内存触发 → 重复释放。

**症状：** 在使用 `_Owned` 参数且包装在 `if` 条件中的函数调用之后立即出现 `free(): double free detected in tcache 2`。

**最小的可重现代码（29 行）：**

```c
#include "bishengc_safety.hbs"
#include <stdlib.h>

_Owned struct Box {
_Public:
    int *_Owned data;
    ~Box(Box this) {
        _Unsafe { free((void*)__move_to_raw(this.data)); }
    }
};

_Safe Box Box::new(void) {
    Box b = { .data = safe_malloc(0) };
    return b;
}

_Safe int consume(Box b) { return 1; }   /* 获取所有权 */

int main(void) {
    Box b = Box::new();
    if (consume(b) == 1) { }  /* BUG：b 未被标记为已转移 → 在 } 处重复释放 */
    return 0;
}
```

**AST 证据** — 使用 `clang -Xclang -ast-dump -fsyntax-only` 转储：
- 工作模式（`int r = consume(b); if (r == 1) {}`）：AST 在调用后显示 `BinaryOperator '=' b_is_moved = 1`。
- 有 bug 的模式（`if (consume(b) == 1) {}`）：没有 `b_is_moved = 1` 赋值，只有初始的 `b_is_moved = 0` 和析构函数 `if (!b_is_moved)` 检查。

**变通方法** — 在测试前将调用结果存储到局部变量中：

```c
// 糟糕 — 重复释放
if (consume(owned_value) == 1) { ... }

// 好 — 提取调用
int result = consume(owned_value);
if (result == 1) { ... }

// 也好 — 作为独立语句调用
consume(owned_value);
if (some_other_check) { ... }
```

**此 bug 适用于**：
- `if (call(owned) ...)` 和 `if (... call(owned) ...)`
- `while (call(owned) ...)`、`for (...; call(owned); ...)`
- `switch (call(owned))`
- 任何包含消耗 `_Owned` 参数的 `CallExpr` 的条件表达式

**此 bug 不适用于**：
- `T result = call(owned);` 后跟 `if (result ...)` — DeclStmt 被检查
- `call(owned);` 作为独立语句 — CallExpr 被检查
- `lhs = call(owned);` 作为顶级赋值 — BinaryOperator 被检查

**当你怀疑此 bug 时：** 将一个可疑调用站点转换为变通形式。如果重复释放消失，就确认了。对同一作用域中的所有同级站点应用相同的变通方法。

### 8.7 已知编译器 bug：`_Owned` 局部变量的析构函数在返回借用它的 return 表达式之前触发

析构函数脱糖过程为每个 `_Owned` 局部变量在作用域末尾作为**在 `ReturnStmt` 之前**的同级语句发出析构函数 `IfStmt`。如果返回表达式是通过该局部变量的借用进行读取的调用（`return f(&_Const b, ...)`），则该调用在**已被析构的内存**上运行，并返回错误的值。没有崩溃，没有警告——只是静默地返回损坏的值。

这与 §8.6 不同：那是在 `if`/`while` 条件中缺少移动标记赋值。这个是析构函数插入与 `ReturnStmt` 操作数评估之间的顺序错误，即使不涉及移动也存在。

**症状：** 包含 `_Owned` 局部变量和 `return f(...borrow of that local...);` 的返回 `_Bool` 或 `int` 的 `_Safe` 函数返回错误的值。在调用和返回之间添加任何 `_Unsafe { printf(...); }` 会使其"开始正常工作"——因为打印操作将值强制放入一个命名临时变量中，其生命周期跨越析构函数。

**最小的可重现代码（30 行）：**

```c
#include "string.hbs"
#include <stdio.h>

_Safe static String make(void) { _Unsafe { return String::from("x"); } }

_Safe static _Bool buggy(const String* _Borrow a) {
    String b = make();
    return a->equals(&_Const b);   /* BUG：即使 a == b 也返回 0 */
}

_Safe static _Bool stashed(const String* _Borrow a) {
    String b = make();
    _Bool r = a->equals(&_Const b);
    return r;                       /* 正确：返回 1 */
}

int main(void) {
    String a = make();
    _Unsafe {
        printf("buggy   = %d\n", (int)buggy  (&_Const a));  /* 打印 0 */
        printf("stashed = %d\n", (int)stashed(&_Const a));  /* 打印 1 */
    }
    return 0;
}
```

**AST 证据** — 使用 `clang -Xclang -ast-dump -fsyntax-only` 转储。

有 bug 模式的 `CompoundStmt`：
```
├── DeclStmt: b = make()
├── DeclStmt: b_is_moved = 0
├── IfStmt: if (!b_is_moved) ~String(b)     ← 析构函数在此处触发
└── ReturnStmt
    └── CallExpr: a->equals(&_Const b)      ← 在析构函数之后读取 b
```

正确模式的 `CompoundStmt`：
```
├── DeclStmt: b = make()
├── DeclStmt: b_is_moved = 0
├── DeclStmt: r = a->equals(&_Const b)      ← 在析构函数之前读取 b
├── IfStmt:  if (!b_is_moved) ~String(b)
└── ReturnStmt: return r
```

相同的节点集合，不同的顺序。该过程在语句列表末尾插入析构函数 `IfStmt`，不管尾部的 `ReturnStmt` 的操作数是否仍在借用拥有的局部变量。

**触发条件** — 必须同时满足三个条件：
1. 函数有一个 `_Owned` 局部变量（`String`、任何 `_Owned struct`）在返回时仍在作用域中。
2. 返回表达式是一个 `CallExpr`，接受指向该局部变量的 `*_Borrow`（直接或通过其他参数传递）。
3. 被调用者实际**解引用**该借用以计算返回值。忽略借用参数的被调用者不触发此 bug。

**变通方法** — 先将调用结果存入一个命名的局部变量中：

```c
// 糟糕 — 返回值错误
_Safe _Bool check(const String* _Borrow a) {
    String b = make();
    return a->equals(&_Const b);
}

// 好 — 相同代码，结果存入局部变量
_Safe _Bool check(const String* _Borrow a) {
    String b = make();
    _Bool r = a->equals(&_Const b);
    return r;
}
```

存入局部变量会将对 `b` 的使用强制放入更早的 `DeclStmt` 中，该语句在析构函数插槽之前完成。`ReturnStmt` 随后只读取 `r`（一个普通的 `_Bool`，没有借用），因此析构函数的顺序不再重要。

**如何在实践中注意到这个 bug：** 来自辅助函数的值看起来颠倒或荒谬，但辅助函数自身的逻辑是正确的。在最后的借用使用和返回之间插入一个 `printf` — 如果 bug 消失，你正在遇到 §8.7。应用存储变通方法。

**此 bug 适用于任何返回表达式形状，其中返回值依赖于被调用者通过借用读取作用域中的 `_Owned` 局部变量**，包括：
- `return owned_local.method(...);` 其中方法读取 `this`
- `return helper(&_Const owned_local);`
- `return outer(inner(&_Const owned_local));`

**此 bug 不适用当：**
- 返回表达式完全不引用 `_Owned` 局部变量
- 返回值是在 return 语句之前已经计算好的普通副本
- `_Owned` 局部变量在返回前已被转移出去（那么析构函数被 `b_is_moved=1` 跳过，不会读取已释放的内存）

## 9. 快速参考

| 错误 | 修复 |
|---------|-----|
| `_Owned char*` | `char *_Owned` — 限定符在 `*` 之后 |
| `_Borrow int*` | `int *_Borrow` — 限定符在 `*` 之后 |
| `_Safe` 中的 `&` | 使用 `&_Const` 或 `&_Mut` |
| `_Safe` 中的 `int x = i++` | `i++` 作为语句可以；结果为 `void` |
| `_Safe int f()` | `_Safe int f(void)` |
| 安全区域中的部分初始化 | 仅对**带指针字段的**结构体需要 |
| 安全区域中的联合体 `.member` | 禁止 — 使用 `_Unsafe {}` 逃逸 |
| 安全区域中的强制转换类别 | 不允许跨类别转换（`T *_Owned` 到 `void *_Owned` 除外） |
| 安全区域中 `&_Mut` 全局变量 | 禁止 — 只能对全局变量使用 `&_Const` |
| `_Trait T var` | `_Trait T* ptr` |
| 缺少 `struct` | `struct S`，除非使用 typedef |
| 转移后使用 | 在转移所有权之前使用变量 |
| Owned 未释放 | `safe_free`、传递或返回在作用域结束前 |
| 表达式中的 `_Await` | 先赋值给变量 |
| 参数中的多个 `_Await` | 预先评估一个：`int a = _Await g(); f(a, _Await h());` |
| `_Owned _Borrow` | 非法 — 选择一个 |
| 借用的借用 | `T *_Borrow *_Borrow` — 限制（选择单个级别） |
| `_Owned` 不带 `_Nullable` | 如果指针可能为 null，添加 `_Nullable` |
| 解引用可为空 | 先进行空检查：`if (p != nullptr) { *p = ... }` |
| 通过元素初始化数组 | 使用初始化列表 `{1,2,3}` 或 `__assume_initialized` |
| 未初始化的局部变量 | 使用前初始化；字段级跟踪适用 |
| 来自拥有局部变量的借用的返回值错误 | 先存入局部变量；见 §8.7 |

> 详细错误码见 `/bsc`（references/errors.md）技能
> 安全区域规则见 `/bsc`（references/safe-zone.md）技能
> Ownership 规则见 `/bsc`（references/ownership.md）技能
> Nullability 规则见 `/bsc`（references/nullability.md）技能
> 初始化分析见 `/bsc`（references/initialization.md）技能
