
# INVALID 哨兵模式（子集中的 OOM 容忍类型）

受限子集禁用了 `_Owned struct` 及其析构函数，并禁用了 libcbs 中的 `Option<T>` / `Result<T>`。要表达"此类型可能构造失败"而不中止或崩溃，使用**空缓冲区字段**作为哨兵将失败编码到结构体本身中。

这是此子集中每个带有 `_Owned _ArrayElem _Nullable` 缓冲区的普通结构体的规范形状 — `String`、`Vec` 风格容器、文件句柄、任何拥有可能失败的堆分配的对象。

## 1. 双状态模型

遵循此模式的每个结构体恰好有两个状态：

| 状态 | 条件 | 语义 |
|---|---|---|
| **VALID** | `data != nullptr` 且大小不变量成立 | 正常操作；查询返回真实数据，修改器更新状态。 |
| **INVALID** | `data == nullptr`，所有大小/计数字段为零 | 失败状态；查询返回安全的默认值，修改器静默无操作。 |

没有第三种状态。一个构造函数要么返回 VALID 实例，要么返回 INVALID 实例 — 永远不会是部分初始化的结构体。

## 2. 类型声明

```c
struct String {
    char *_Owned _ArrayElem _Nullable data;  /* nullptr → INVALID */
    size_t len;                              /* INVALID 时为 0 */
    size_t cap;                              /* INVALID 时为 0 */
};
```

缓冲区字段上的 `_Nullable` 是强制性的 — 这就是使 INVALID 状态可表示的原因。没有它，结构体无法编码失败。

## 3. 接口规则（按函数类别）

| 类别 | 规则 |
|---|---|
| 构造函数 | 在 OOM 时返回 `{.data = nullptr, .len = 0, .cap = 0}`。绝不中止。绝不断言。 |
| 有效性测试 | 提供检查 `s->data != nullptr` 的 `_Safe _Bool name_is_valid(const struct T *_Borrow s)`。 |
| 查询 | 当 `s->data == nullptr` 时，返回安全默认值（`0`、`false`、哨兵、`nullptr`）。绝不断言。 |
| 修改器 | 当 `s->data == nullptr` 时静默无操作。在内部分配失败时也进行无操作（允许 VALID → INVALID 转换；记录是否可以）。 |
| 析构函数 | 按值消费者（`void name_free(struct T s)`）。底层的 `safe_free_*` 包装器是空安全的，因此析构函数在 INVALID 上是无操作，无需特殊处理。 |

```c
_Safe _Bool          str_is_valid(const struct String *_Borrow s);
_Safe struct String  str_new(void);                       /* 可能返回 INVALID */
_Safe struct String  str_from_cstr(const char *_Borrow _ArrayElem s); /* 可能返回 INVALID */
_Safe void           str_free(struct String s);           /* 析构函数；消耗 */
_Safe size_t         str_len(const struct String *_Borrow s);
_Safe void           str_push(struct String *_Borrow s, char c);
```

## 4. 辅助函数中强制性的防御性空保护

一个注明 `/* PRECOND: s is VALID */` 的辅助函数仍然不能对 `s->data[i]` 进行下标访问，如果 `data` 的类型是 `_Nullable` — 编译器拒绝该解引用，无论任何文档中的前置条件。每个触及 `data` 的辅助函数都需要防御性保护：

```c
static _Safe void str__write_raw(struct String *_Borrow s, size_t offset,
                                 const char *_Borrow _ArrayElem src, size_t n) {
    if (s->data == nullptr) return;   /* 满足类型检查器 */
    safe_memcpy_into(&_Mut s->data[offset], src, n);
}
```

当调用者尊重前置条件时，该保护在运行时永远不会触发，但它是编译所必需的。这是使用 `_Nullable` 字段不可避免的成本 — 接受它。替代方案（非可空字段）会失去编码 INVALID 的能力。

在 `nullability.md` §4 中有一条关于 `_Owned _ArrayElem _Nullable` 缩小以进行原始指针转移操作（`__move_array_to_raw`）的相关规则；当你需要通过非可空原始视图释放或移交缓冲区时，请参阅该部分。

## 5. 销毁总是强制性的

即使 INVALID 实例也必须传递给析构函数 — 编译器跟踪 `_Owned _ArrayElem _Nullable` 槽，无论它当前是否持有 `nullptr`。因为按类型释放包装器是空安全的，析构函数不需要特殊处理：

```c
_Safe void str_free(struct String s) {
    safe_free_chars(s.data);   /* 空安全：data == nullptr 时无操作 */
}
```

即使在构造函数返回 INVALID 之后，调用者也必须调用 `str_free`。`ownership.md` §7.5 中的字段泄漏检测将捕获任何丢弃 INVALID 实例而不消耗它的路径。

## 6. 何时使用此模式

当**所有**以下条件成立时使用：

- 该类型拥有可能失败的堆分配。
- 代码库不想在 OOM 时中止。
- 调用者可以有意义地响应"构造失败"（跳过、重试、记录、传播）。

当以下情况时不要使用：

- 分配不能有意义地失败（小、不频繁、OOM 时中止是可以的）。无条件返回结构体；让底层分配器中止。
- 该类型是没有"大小"或"有效性"维度的单指针句柄。只需返回 `T *_Owned _Nullable` 并在调用点检查 `nullptr`。
- 该类型足够大，需要进行堆装箱（见 `design.md` "大型结构体"）。然后 INVALID 存在于指针级别：`struct Big *_Owned _Nullable` 在失败时返回 `nullptr`。

## 7. 此模式替代了什么

| 在 libcbs / 标准 BSC 中 | 在此子集中 |
|---|---|
| 带自动析构函数的 `_Owned struct S { ... ~S() { ... } }` | 普通 `struct S { ..._Owned _ArrayElem _Nullable buf; }` + 手动 `name_free` 消费者 |
| 用于"可能失败"返回的 `Option<S>` | 普通 `struct S`，其中 `is_valid(s) == false` 表示 None 情况 |
| 用于"可能因原因失败"的 `Result<S, Err>` | 如果错误情况不携带有效载荷，直接使用 INVALID。如果携带，返回一个标签枚举结构体及结果。 |

## 8. 交叉引用

> 关于底层的字段级所有权跟踪，见 `ownership.md` §7.5。
> 关于通过 `*_Borrow` 替换缓冲区（交换后释放），见 `ownership.md` §7.6。
> 关于通过 `*_Borrow` 释放缓冲区（槽清空 + `*slot = nullptr` 陷阱），见 `ownership.md` §7.7。
> 关于 `_Owned _ArrayElem _Nullable` 原始强制转换的 `_Nullable` 缩小规则，见 `nullability.md` §4。
> 关于大型类型的堆装箱替代方案，见 `design.md` "大型结构体 — 堆装箱而非按值传递"。
