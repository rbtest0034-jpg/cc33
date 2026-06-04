
# `constexpr` 在受限子集中

**可用：**

- `constexpr` 变量和函数用于编译时常量
- `_Static_assert` 用于编译时不变性

```c
constexpr size_t STR_MIN_CAP = 8;
_Static_assert(STR_MIN_CAP >= 1, "最小容量太小");
```

**不在本子集中**（这些需要泛型，已被禁用）：

- 类型特性（`is_integral`、`is_pointer`、`is_borrow`、`is_trivial_data` 等）
- 基于类型特性的 `constexpr if`
- 条件类型别名

使用普通的 `#if`/`#ifdef` 和具体类型代替。
