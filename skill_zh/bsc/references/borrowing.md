
# BiSheng C Borrowing 技能

## 子集横幅 — 先阅读

本项目 BSC 子集禁用了：泛型、成员函数、traits、libcbs、`_Owned struct`。借用检查器和本技能中的所有规则**不变地**适用于普通结构体 + 自由函数代码。当你在正文中看到使用已禁用功能的旧示例时，使用下表进行翻译：

| 本技能中的旧模式 | 使用此替代 |
|---|---|
| `Vec<int> *_Borrow this` | 在普通结构体上的自由函数，例如 `int_vec_len(const struct IntVec *_Borrow v)` |
| `obj->method(...)` / `val.method(...)` 成员函数调用 | 自由函数调用，例如 `obj_set_string(obj, name, value)` |
| `val.get_object()->set_string(...)` 链式成员调用 | 自由函数链或命名临时借用：`set_string(get_object(&_Mut val), name, value)` |
| 泛型类型参数限制（例如 "不允许 `_Borrow` 作为泛型参数"） | 不适用 — 泛型被禁用 |

其他所有内容（借用语法、NLL、冻结、类型转换、`_Borrow _ArrayElem`、可空性、设计模式）按原样适用。

## 默认选择 `_Borrow`，而非 `_Owned`，用于指针参数

一个通过指针**读取或修改**值但**不获取所有权**的函数必须使用 `_Borrow`，绝不能使用 `_Owned`。

- 函数释放指针、将其传递出去或长期存储 → `T *_Owned`
- 函数只通过指针读取 → `const T *_Borrow`
- 函数通过指针修改但不释放 → `T *_Borrow`
- 函数进行指针算术/迭代 → 原始 `T *`

`_Owned` 参数消耗调用者的变量。大多数 C 函数不这样做。默认到处使用 `_Owned` 会使每个调用点变成一次性移动，并在调用点强制使用丑陋的重新分配模式 — 这是错误的。

## 关键：`_Borrow` 语法

**`_Borrow` 放在 `*` 之后**，而不是类型之前。它是一个指针限定符。

```c
// 正确
const int *_Borrow r = &_Const x;
int *_Borrow mr = &_Mut x;

// 错误（不能编译）
_Borrow int* r = &_Const x;
```

## 1. 概述

临时的、非拥有引用。编译器在编译时强制借用规则 — 没有悬空引用，没有别名修改。

## 2. 创建借用

```c
int x = 42;
const int *_Borrow cr = &_Const x;   // 不可变借用 — 只读
int *_Borrow mr = &_Mut x;           // 可变借用 — 读/写
```

从 `_Owned` 指针：`&_Const *owned_ptr` 或 `&_Mut *owned_ptr`。

## 3. 语义

| 类型 | 访问 | 别名 |
|---|---|---|
| `const T *_Borrow` | 只读 | 同时允许多个 |
| `T *_Borrow` | 读 + 写 | 同时只能有一个 |

可变借用隐式转换为不可变：`T *_Borrow` -> `const T *_Borrow`（编译器插入 `&_Const *`）。这在声明、赋值、函数参数和返回中都有效。

## 4. 非词法生命周期（NLL）

借用生命周期使用 **NLL**：一个借用从其创建（或重新赋值）到其**最后一次使用**是活跃的，而不是到词法作用域的结束。NLL 可以被分段 — 如果一个借用变量被重新赋值，它会创建不相交的活跃范围。

```c
void use(int *_Borrow p) {}

void foo() {
    int local1 = 1, local2 = 2;
    int *_Borrow p = &_Mut local1;  // NLL 段 1 开始
    use(p);                          // NLL 段 1 结束（最后一次使用）
    // local1 在此处解冻 — p 的借用已结束
    local1 = 10;                     // 可以：p 不再活跃
    p = &_Mut local2;                // NLL 段 2 开始（并结束 — 无后续使用）
}
```

扩展 NLL 的使用：函数调用 `use(p)`、`return p`、解引用 `*p`、成员访问 `p->field`。

## 5. 规则

- **`_Borrow` 放在 `*` 之后**：写 `int *_Borrow`，而不是 `_Borrow int*`
- **不可变借用**活跃时：原始对象只读，不允许可变借用
- **可变借用**活跃时：原始对象**被冻结** — 不允许读取、写入、移动或借用
- **结构体字段借用**：`&_Mut e.field` 仅冻结**该字段**（阻止整个结构体的修改）；其他字段保持可访问。`&_Const e.field` 将字段置于只读状态
- 借用生命周期 <= 被借用值的生命周期（编译器强制）
- 不能创建借用的借用：`int *_Borrow *_Borrow` 是非法的
- 借用变量必须在声明时初始化
- 不能是全局变量或联合体成员
- `_Owned` 和 `_Borrow` 不能共存于同一指针：`int *_Owned _Borrow` 是非法的
- 不允许对**普通**借用指针进行索引（`p[i]`）和指针算术（`p + n`、`p++`）。对于数组元素的借用，使用 `_Borrow _ArrayElem`（见 §11）— 该变体支持 `[]`、`+`、`-`、`+=`、`-=`、`++`、`--`
- 不能对包含借用成员的结构体本身进行借用
- 全局变量：在安全区域中，只允许不可变借用（`&_Const`）；不允许对全局变量进行可变借用
- 字符串字面量：`&_Mut "hello"` 被禁止；`&_Mut * "hello"` 被禁止

### 重新赋值规则
- 借用变量的重新赋值要求相同类型，新源的生命周期必须 >= 借用变量的剩余生命周期

## 6. 解引用和成员访问

### 解引用（`*p`）
| 操作 | 不可变借用 | 可变借用（T 是 Copy） | 可变借用（T 是 Move） |
|---|---|---|---|
| 读取 `*p` | 可以 | 可以 | 可以 |
| 赋值给 `*p` | 错误 | 可以 | 错误（不能移动赋值） |

### 成员访问（`p->field`）
- 不可变借用：可以读取字段，不能修改
- 可变借用：可以读取和修改字段（对于 Copy 类型）
- 不可变借用不能调用可变方法（`T *_Borrow this` 方法）

## 7. 类型转换

- **`T *_Borrow` → `void *_Borrow`**：当 `T` 是普通数据类型（没有指针字段）时**隐式**允许；否则该转换**被禁止**，即使使用显式强制转换
  ```c
  struct S { int *ptr; };
  int a = 0; struct S s = {.ptr = nullptr};
  _Safe {
      int *_Borrow p1 = &_Mut a;
      void *_Borrow p2 = p1;                       // 可以：int 是普通类型
      struct S *_Borrow p3 = &_Mut s;
      void *_Borrow p4 = (void *_Borrow)p3;        // 错误：S 有指针字段
  }
  ```
- **`void *_Borrow` → `T *_Borrow`**：隐式转换**被禁止**；显式强制转换必须在 `_Unsafe` 中
- **`_Borrow` 和原始指针之间**：在 `_Safe` 区域中被禁止；在 `_Safe` 之外，强制转换通过原始 `T *` 中间类型进行，从不直接进行
- **`_Owned` 和 `_Borrow` 之间**：任一方向的 C 风格强制转换都被**禁止**
- **可变和不可变借用之间**：显式强制转换被禁止。可变→不可变是隐式的（编译器插入 `&_Const *`）；反向永远不允许
- **`T *_Borrow _ArrayElem` → `T *_Borrow`**：允许作为隐式转换（相当于"在当前元素处重新借用，放弃数组迭代语义"）。反向 `T *_Borrow` → `T *_Borrow _ArrayElem` 被禁止
- **隐式 `_Bool` 转换**：`_Borrow _Nullable` 指针可以在条件中使用

## 8. 附加规则

- 借用指针可以在 `if`/`while`/`do-while`/`for`/三元运算符条件中使用，但**不能**在 `switch` 中
- **普通**借用指针上禁止的运算符：`-`、`~`、`[]`、`++`、`--`、`*`（算术）、`/`、`%`、`&`（位运算）、`|`、`<<`、`>>`、二元 `+`、二元 `-`。`_Borrow _ArrayElem` 重新启用 `[]`、二元 `+`/`-`、`+=`/`-=`、`++`/`--`（见 §11）。其余仍然禁止
- 相同类型的借用之间允许比较（`==`、`!=`、`<`、`<=`、`>`、`>=`）。指向类型上的顶层 `const`/`volatile`/`restrict` 在比较时被忽略 — `int *_Borrow` 和 `const int *_Borrow` 是可比较的
- `sizeof(T *_Borrow) == sizeof(T *)`，`_Alignof` 相同。`_Borrow _ArrayElem` 与 `T *` 具有相同的大小/对齐
- 字符串字面量在传递给 `const char *_Borrow` 参数时自动借用为 `const char *_Borrow`（编译器插入 `&_Const *`）

## 9. 函数签名

在此子集中，自由函数将普通结构体借用作为第一个参数（没有 `this`，没有成员函数语法）。约定：按照结构体命名第一个参数（`v`、`s` 等）。

```c
struct IntVec {
    int *_Owned _ArrayElem _Nullable data;   /* INVALID 哨兵 = nullptr */
    size_t len;
    size_t cap;
};

// 只读访问 — &_Const 借用
_Safe size_t int_vec_len(const struct IntVec *_Borrow v) {
    return v->len;
}

// 可变访问 — &_Mut 借用
_Safe void int_vec_push(struct IntVec *_Borrow v, int value) { /* ... */ }

// 返回借用 — 生命周期绑定到输入借用
_Safe int *_Borrow int_vec_first(struct IntVec *_Borrow v) { /* ... */ }
```

- 如果返回是 `_Borrow` 且有一个参数是 `_Borrow`：返回生命周期 = 参数生命周期
- 如果有多个 `_Borrow` 参数：返回生命周期 = 所有参数生命周期的并集
- 如果没有 `_Borrow` 参数但返回是 `_Borrow`：**编译错误** — 借用返回需要至少一个借用参数

## 10. 设计模式

当借用检查器拒绝"显而易见"的 C 风格代码时，这些惯用法反复出现。

### 10.1 迭代遍历 → 转换为递归

**问题：** 你想通过 while 循环遍历一个链式结构，每一步都重新赋值游标。借用检查器禁止这样做，因为第 N+1 步的借用源自第 N 步的借用，重新赋值使链失效。

```c
// 错误 — 无法在借用通过 current 活跃时重新赋值
_Safe struct Node *_Borrow walk(struct Node *_Borrow root, const struct Path *_Borrow p) {
    struct Node *_Borrow current = root;
    for (size_t i = 0; i < path_length(p); i++) {
        current = node_child(current, i);   // 重新赋值与派生借用冲突
    }
    return current;
}
```

**修复：** 转换为尾递归。每个递归调用有自己的新栈帧，因此借用链保持线性。

```c
_Safe static struct Node *_Borrow walk_rec(struct Node *_Borrow current,
                                            const struct Path *_Borrow p, size_t i) {
    if (i >= path_length(p)) { return current; }
    return walk_rec(node_child(current, i), p, i + 1);
}
_Safe struct Node *_Borrow walk(struct Node *_Borrow root, const struct Path *_Borrow p) {
    return walk_rec(root, p, 0);
}
```

编译器通常在 `-O1` 及以上优化级别将尾递归优化回循环，因此没有运行时成本。

### 10.2 链式调用，不要命名中间借用

**问题：** 将中间值提取到命名变量中可能会使其借用超出所需的范围。

```c
// 可能很尴尬 — `obj` 通过整个块从 `val` 借用
struct JsonObject *_Borrow obj = json_get_object(&_Mut val);
json_object_set_string(obj, name, value);
// ... 大量代码 ...
// 对 `val` 的任何其他借用都与 `obj` 冲突
```

**修复：** 当借用只在一个调用中使用时，内联链式调用自由函数：

```c
json_object_set_string(json_get_object(&_Mut val), name, value);
// 没有命名借用 — 没有生命周期扩展
```

仅当你多次使用借用时才命名它；否则让它成为临时值。

### 10.3 严格限定借用的作用域

如果你必须命名一个借用但不需要它用于整个函数，将其包装在一个块中，使其生命周期显式结束：

```c
{
    struct JsonObject *_Borrow obj = json_get_object(&_Mut val);
    json_object_set_string(obj, name, value);
}   // <-- `obj` 在此处释放
json_validate(&_Mut val, &_Mut schema);   // 现在可以自由地对 `val` 进行可变借用
```

### 10.4 提取工作，而非借用

**问题：** 你需要来自借用的数据，但之后也需要自由地使用被借用者。

```c
// 错误 — `key` 传递性地扩展 `parent` 的借用
const struct String *_Borrow key = json_get_name(parent, 0);
json_set_value(&_Mut *parent, key, /* ... */);   // 可变借用与 key 冲突
```

**修复：** 克隆你需要的数据，丢弃借用，然后执行修改。

```c
struct String key_owned;
{
    const struct String *_Borrow key = json_get_name(parent, 0);
    key_owned = string_clone(key);
}
json_set_value(&_Mut *parent, &_Const key_owned, /* ... */);   // parent 再次自由
```

## 11. 完整示例

```c
#include <stdio.h>

void printValue(const int *_Borrow ref) {
    printf("value = %d\n", *ref);
}

void doubleValue(int *_Borrow ref) {
    *ref = *ref * 2;
}

const int *_Borrow getFirst(const int *_Borrow arr, int len) {
    return arr;
}

int main() {
    int x = 21;

    // 不可变借用
    const int *_Borrow cr = &_Const x;
    printValue(cr);
    printValue(&_Const x);   // 内联借用

    // 可变借用
    int *_Borrow mr = &_Mut x;
    doubleValue(mr);
    // NLL：mr 的生命周期在最后一次使用（doubleValue 调用）结束
    printf("x = %d\n", x);  // 可以：x 不再被冻结

    // 多个不可变借用是可以的
    const int *_Borrow r1 = &_Const x;
    const int *_Borrow r2 = &_Const x;
    printf("r1=%d, r2=%d\n", *r1, *r2);

    // 结构体字段借用 — 只有被借用的字段被冻结
    struct Point { int x; int y; };
    struct Point p = {.x = 10, .y = 20};
    int *_Borrow px = &_Mut p.x;  // 只有 p.x 被冻结
    p.y = 30;                      // 可以：p.y 是不同字段
    *px = 50;
    // px 的 NLL 在此处结束（上面的最后一次使用）

    // 可变到不可变的隐式转换
    int val = 5;
    int *_Borrow mp = &_Mut val;
    const int *_Borrow ip = mp;  // 可以：隐式转换

    // 字符串字面量自动借用
    void print_str(const char *_Borrow s);
    print_str("hello");  // 编译器自动插入 &_Const *

    return 0;
}
```

## 11. `_Borrow _ArrayElem`：借用进入数组

使用 `&_Mut arr[i]` 或 `&_Const arr[i]` 获取数组元素的地址会产生 `_Borrow _ArrayElem` 指针。这是一个**记住它指向数组内部**的借用，因此下标和算术运算是允许的（与普通 `_Borrow` 不同）。

```c
_Safe int foo(void) {
    int arr[4] = {1, 2, 3, 4};
    int *_Borrow _ArrayElem p = &_Mut arr[0];
    p = p + 1;            // 可以：_Borrow _ArrayElem 支持 +
    p += 1;               // 可以：+=
    ++p;                  // 可以：++
    int x = p[0];         // 可以：下标

    int *_Borrow q = p;   // 可以：隐式降级为普通借用
    // q + 1;             // 错误：普通借用禁止算术运算
    // q[0];              // 错误：普通借用禁止下标
    return x;
}
```

`_Borrow _ArrayElem` 特有的规则：

- 允许的运算符：`[]`、二元 `+`、`-`、`+=`、`-=`、`++`、`--`。所有其他普通 `_Borrow` 的限制仍然适用（没有 `*` 算术、没有位运算等）。
- **隐式降级** `T *_Borrow _ArrayElem` → `T *_Borrow` 是允许的（相当于"在当前元素处获取一个新的普通借用"）。反向显式强制转换被**禁止**。
- 对于混合 `_Safe`/`_Unsafe` 声明，`_Borrow _ArrayElem` 是一个单一的限定符单元 — `_Safe` 重声明可以向裸指针 `_Unsafe` 参数添加 `_Borrow _ArrayElem`，但一旦声明了一个，就不能在 `_Borrow` 和 `_Borrow _ArrayElem` 之间切换。
- `sizeof(T *_Borrow _ArrayElem) == sizeof(T *)`。

除了上述运算符和这些转换规则外，**适用于普通 `_Borrow` 的所有其他规则（生命周期、冻结、NLL、可空性、类型兼容性、不允许借用之借用等）同样适用于 `_Borrow _ArrayElem`**。大多数时候你只通过 `&_Mut arr[i]` 隐式看到此类型，并立即将其绑定到普通 `_Borrow`；仅当你需要下标或步进时才显式命名它。

> 关于 ownership（拥有指针），见 `ownership.md` 技能
> 关于需要借用才能使用的安全区域，见 `safe-zone.md` 技能
> 关于借用的可空性，见 `nullability.md` 技能
> 关于借用错误（BSC-E02xx），见 `errors.md` 技能
