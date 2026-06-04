
# BiSheng C 错误码技能

## 1. 错误码方案

BSC 错误使用 `BSC-Exxxx`（错误）和 `BSC-Wxxxx`（警告）格式，按类别分组。**在子集中？** 列告诉你该类别是否适用于本项目 — 与禁用特性相关的类别不会在符合子集的代码中触发，看到它们意味着你意外使用了已禁用的特性。

| 范围 | 类别 | 描述 | 在子集中？ |
|---|---|---|---|
| BSC-E01xx | Ownership | 移动后使用、未初始化、赋值、强制转换、内存泄漏 | 是 |
| BSC-E02xx | 借用检查 | 生命周期、多个借用、借用时赋值 | 是 |
| BSC-E03xx | 安全区域 | 在 `_Safe` 函数/块中禁止的操作 | 是 |
| BSC-E04xx | 可空性 | 解引用/传递/返回/强制转换为空指针 | 是 |
| BSC-E05xx | Traits | 未定义的 trait、未实现的函数、类型冲突 | **否** — 此子集中 `_Trait` 禁用 |
| BSC-E06xx | 拥有的结构体 / 析构函数 | 结构体标签、析构函数、成员问题 | **否** — `_Owned struct` 禁用；使用普通结构体 + `name_free`（`ownership.md` §7.5） |
| BSC-E07xx | 类型系统 | `_Owned`/`_Borrow` 限定符冲突、不兼容的强制转换 | 是 |
| BSC-E08xx | 异步 | `_Async`/`_Await` 使用 | **否** — 此子集中协程禁用 |
| BSC-E09xx | 泛型 / Constexpr | 泛型函数问题、constexpr 限制 | 部分 — 仅 constexpr 部分；泛型禁用 |
| BSC-E10xx | 解析级别 | BSC 特定语法错误 | 是 |
| BSC-E11xx | 成员函数 | 实例成员、属性 | **否** — 此子集中成员函数禁用 |
| BSC-E12xx | 运算符重载 | 重载限制 | **否** — 此子集中运算符重载禁用 |

## 2. 诊断抑制标志

使用命令行上的 `-Eno-<identifier>` 或源码中的 `#pragma GCC diagnostic ignored "-E<identifier>"` 抑制：

```
bsc-safety-check              # 所有 BSC 安全检查
├── ownership.md             # 所有所有权检查
│   ├── use-moved-owned       #   移动后使用
│   ├── use-uninit-owned      #   使用未初始化
│   ├── assign-owned          #   赋值给拥有的
│   ├── cast-owned            #   强制转换拥有的
│   └── check-memory-leak     #   内存泄漏
├── bsc-borrow                # 所有借用检查
│   ├── assign-borrowed       #   借用时赋值
│   ├── move-borrowed         #   借用时移动
│   ├── use-mutably-borrowed  #   可变借用时使用
│   ├── repeated-borrow       #   多个可变借用
│   ├── return-local-borrow   #   返回局部引用
│   └── short-life-borrow     #   生命周期太短
└── nullability.md           # 所有可空性检查
    ├── deref-nullable        #   解引用可为空
    ├── pass-nullable         #   传递可为空
    └── return-nullable       #   返回可为空
```

> 完整层级和使用示例见 `compile.md` 技能。

## 3. AI 友好数据库

对于 LLM/IDE 集成，`errors/ai/` 目录包含：

- `errors.md.json` — 结构化 JSON 数据库，包含所有错误的成因、修复策略、代码示例和关键字
- `errors.md.schema.json` — JSON 模式
- `error-code-mapping.json` — 将诊断名称映射到错误码

## 4. 常见错误快速参考

### Ownership（BSC-E01xx）

| 代码 | 消息 | 修复 |
|---|---|---|
| E0101 | 使用了已移动的值 | 在转移所有权之前使用变量 |
| E0104 | 使用了未初始化的值 | 在使用前初始化 |
| E0106 | 赋值给 _Owned 值 | 先释放旧值，或使用新变量 |
| E0115 | _Owned 值的无效强制转换 | 不要强制转换仍持有所有权的 _Owned 指针 |
| E0118 | 内存泄漏（拥有的未释放） | 在作用域结束前释放、返回或转移 |
| E0119 | 字段内存泄漏 | 在作用域结束前释放 _Owned 结构体字段 |
| E0126 | 临时变量内存泄漏 | 捕获返回 _Owned 的函数的返回值 |

### Borrow（BSC-E02xx）

| 代码 | 消息 | 修复 |
|---|---|---|
| E0201 | 不能赋值给已借用的变量 | 在修改前结束借用 |
| E0202 | 不能移出已借用的变量 | 在移动前结束借用 |
| E0203 | 不能使用已被可变借用的变量 | 改为通过借用指针读取 |
| E0204 | 不能多次借用为可变 | 一次只能有一个 `&_Mut` |
| E0207 | 不能返回对局部变量的引用 | 按值返回或使用 `_Owned` 堆分配 |
| E0208 | 值的存活时间不够长 | 确保借用的值的生命周期超过借用 |

### Safe Zone（BSC-E03xx）

| 代码 | 消息 | 修复 |
|---|---|---|
| E0301 | 在安全区域中禁止的不安全操作 | 包装在 `_Unsafe { }` 块中或使用安全替代方案 |
| E0302 | 无效的 _Safe/_Unsafe 声明位置 | 将限定符放在函数声明或块语句上 |
| E0303 | 安全区域中的联合体成员访问 | 将联合体访问包装在 `_Unsafe { }` 块中 |
| E0304 | 安全区域中禁止的强制转换 | 将强制转换包装在 `_Unsafe { }` 中或传递正确类型 |
| E0307 | _Safe 函数中禁止的操作 | 将操作移到 `_Unsafe` 块中 |
| E0308 | 安全区域中的可变全局变量 | 使全局变量为 `const` 或在安全区域外定义 |

### Nullability（BSC-E04xx）

| 代码 | 消息 | 修复 |
|---|---|---|
| E0401 | 不能解引用可为空的指针 | 在解引用前添加空检查 |
| E0402 | 不能传递可为空的指针参数 | 在调用前用空检查保护 |
| E0403 | 不能返回可为空的指针类型 | 添加空检查或将返回类型改为 `_Nullable` |
| E0406 | Nonnull 被可为空赋值 | 在赋值前用空检查保护 |
| E0407 | _Nonnull 指针必须初始化 | 在声明时初始化 |

> 详细的按类别文档见 `errors/` 目录。
> 常见修复见 `bsc-common-mistakes` 技能。
