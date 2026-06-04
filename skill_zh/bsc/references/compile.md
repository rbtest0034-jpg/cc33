
# BiSheng C 编译技能

## 子集横幅

本项目子集不链接 `libstdcbs` 或不包含 `bishengc_safety.hbs`。下面显示有 `-L/path/to/libcbs -lstdcbs` 的编译命令仅当那些特性可用时适用。在此子集中：

- 没有 `-lstdcbs` 链接步骤。
- 没有 `bishengc_safety.hbs` 包含。
- 通过 `bsc_compat.h` 支持双编译（见 `c-to-bsc-annotation-only.md`）。

## 1. 编译器路径设置

BSC 编译器是定制构建的 clang — 它**不是**系统自带的 `clang`。在编译之前，你需要 BSC clang 可执行文件的路径。

**如何找到路径：** 检查项目的 `CLAUDE.md`（或 `.cursorrules` / `AGENTS.md`）中是否有类似以下内容：

```
BSC 编译器路径：/path/to/bsc/bin/clang
```

如果没有配置路径，询问用户。然后所有编译命令使用完整路径：

```bash
/path/to/bsc/bin/clang file.cbs -o output
```

**推荐的项目设置：** 将编译器路径添加到 `CLAUDE.md`，以便 AI 始终知道在哪里找到它：

```markdown
## BSC 编译器
- 路径：/home/user/bsc/build/bin/clang
- libcbs 包含：/home/user/bsc/libcbs/src
```

## 2. 基本用法

```bash
# 编译为可执行文件
/path/to/bsc/bin/clang file.cbs -o output

# 使用优化
/path/to/bsc/bin/clang file.cbs -O2 -o output

# 仅语法检查（无二进制输出）
/path/to/bsc/bin/clang -fsyntax-only file.cbs

# 带警告
/path/to/bsc/bin/clang -Wall -Wextra file.cbs -o output

# 带调试信息
/path/to/bsc/bin/clang -g file.cbs -o output
/path/to/bsc/bin/clang -g -gdwarf-4 file.cbs -o output  # 更好的 gdb 兼容性

# 将 .c / .h 文件编译为 BSC（覆盖基于扩展名的检测）
/path/to/bsc/bin/clang -x bsc file.c -o output       # 驱动器形式，空格分隔
/path/to/bsc/bin/clang -cc1 -xbsc file.c             # cc1 形式，无空格
```

**`-x bsc` 使用场景**：增量 C→BSC 迁移，无需重命名每个文件；保留原始 `.c`/`.h` 扩展名以与外部工具兼容。只有 `.cbs`/`.hbs` 会自动检测为 BSC — 其他所有内容需要此标志。大多数编辑器/LSP 工具依赖于 `.cbs`/`.hbs`，因此保留 `.c`/`.h` 意味着你会失去 BSC 感知的编辑器特性。

## 3. 包含路径

BSC 有两个可能需要 `-I` 标志的包含目录：

| 路径 | 包含内容 |
|---|---|
| `libcbs/src/` | 标准库头文件（`vec.hbs`、`string.hbs` 等） |
| `clang/lib/Headers/bsc_include/` | 内置头文件（`bsc_type_traits.hbs`、`future.hbs` 等） |

```bash
/path/to/bsc/bin/clang -I/path/to/libcbs/src -I/path/to/bsc_include file.cbs -o output
```

### 链接 libcbs（`String`、`Vec` 等**不是纯头文件**）

`String`、`Vec`、`LinkedList`、`Option`、`Result` 和所有其他 libcbs 类型在 `libstdcbs.a` 中有已编译的实现。包含它们的 `.hbs` 头文件而不链接库会在链接时产生"未定义的引用"（如 `struct_String_new`）。使用任何 libcbs 类型时，始终添加 `-L<install>/lib -lstdcbs`：

```bash
/path/to/bsc/bin/clang file.cbs \
    -I/path/to/install/include/libcbs \
    -L/path/to/install/lib -lstdcbs \
    -o output
```

如果程序也使用 pthreads（例如线程池），也添加 `-lpthread`。`bishengc_safety.hbs` 原语（`safe_malloc`、`safe_free`、`safe_swap`）也是在 `libstdcbs` 中实现的 — 它们不是宏。

## 4. 特殊模式

```bash
# 源到源重写（BSC → C）
/path/to/bsc/bin/clang -rewrite-bsc file.cbs
# 生成 file.c，BSC 特性降低为普通 C

# 带包含路径的重写（标准库类型需要）
/path/to/bsc/bin/clang -rewrite-bsc file.cbs -I/path/to/libcbs/src

# 带显式输出的重写
/path/to/bsc/bin/clang -rewrite-bsc file.cbs -o output.c

# 重写多个文件
/path/to/bsc/bin/clang -rewrite-bsc foo.cbs bar.cbs

# 带行号映射的重写（用于调试）
/path/to/bsc/bin/clang -rewrite-bsc -line file.cbs -o file.c

# AST 转储
/path/to/bsc/bin/clang -Xclang -ast-dump -fsyntax-only file.cbs
```

## 5. 阅读 BSC 编译器诊断

### 始终与 `error:` 一起阅读 `note:` 行

BSC 编译器发出带有 `note:` 源代码位置的借用检查器错误，告诉你**冲突的先前借用开始的位置**：

```
file.cbs:22:29: error: cannot borrow `b` as mutable more than once at a time
file.cbs:21:29: note: first mut borrow occurs here
```

不阅读 `note:`，错误看起来不透明。阅读它，修复变得明显 — 限定第 21 行的借用在第 22 行之前结束。

**已知空白**（截至当前编译器）：
- `error: use of moved value: X` 没有伴随的 `note:` 指向移动位置。使用 LSP `hover` 在 `X` 上查看移动位置。
- 没有 `help:` 建议修复模式 — 你必须知道惯用法（见 `/bsc-common-mistakes` §3.3 了解"局部借用存活时间太长"的修复方法）。

### LSP 是活跃范围信息的权威来源

在任何 `_Owned` 或 `_Borrow` 变量上 `hover` 返回完整的所有权时间线 AND 显式的活跃范围：

```
Ownership Flow:
line 21: Declared
line 21: Mut borrow of b
Live range: lines 21-21
```

对于拥有的值：hover 显示 `Moved into foo()` / `Freed` / `Dropped` 事件，带有确切的行号。

当编译器诊断简短时，**始终检查错误命名的变量上的 LSP hover** — 它比诊断打印的信息更详细。见下面 §6 中通过 LSP + AST 转储调试析构函数 bug 的内容。

## 6. 检查析构函数插入（调试内存错误）

当你遇到运行时 `double free` 或 `use after free` 并怀疑编译器的析构函数插入错误时，真相来源是**驱动器模式下的脱糖后 AST**。

### 关键：使用驱动器模式，而不是直接使用 `-cc1`

```bash
# 错误 — 缺少系统包含，使每个 _Owned struct 虚假地"无效"，
# 产生误导性的 RecoveryExpr / <dependent type> AST 节点
clang -cc1 -fsyntax-only -ast-dump file.cbs -I./include

# 正确 — 驱动器模式拾取系统包含（stdlib.h、string.h 等）
clang -Xclang -ast-dump -fsyntax-only file.cbs -I./include
```

在没有系统包含的 `-cc1` 模式下，找不到 `stdlib.h` → `bishengc_safety.hbs` 解析失败 → 每个使用 `String`/`Vec` 的 `_Owned struct` 被标记为 `referenced invalid struct ... definition` → 通过它们的每个成员访问变成 `RecoveryExpr` / `CXXDependentScopeMemberExpr`。这个 AST 看起来坏了，但不反映实际代码生成产生的结果。**始终使用驱动器模式（`-Xclang -ast-dump`）进行与析构函数相关的调试。**

### 正确的析构函数插入看起来像什么

对于每个 `_Owned struct` 局部变量或参数，脱糖后的 AST 应显示：

```
# 在函数/参数入口处：
DeclStmt
  VarDecl 'varname_is_moved' 'bool' cinit
    IntegerLiteral 'int' 0

# 在使用该变量的 CallExpr 或 BinaryOperator 之后：
BinaryOperator 'bool' '='
  DeclRefExpr 'varname_is_moved' 'bool'
  IntegerLiteral 'int' 1

# 在作用域退出处（或 ReturnStmt）：
IfStmt
  UnaryOperator '!' → DeclRefExpr 'varname_is_moved'
  CallExpr → BSCMethod '~TypeName' → DeclRefExpr 'varname'
```

如果三个部分都存在且对于你怀疑的变量格式良好，**编译器的插入是正确的**，bug 在别处（通常是你结构体布局中堆指针字段之间的别名）。

### 泛型模板转储不同

`Vec<T>::insert_at` 在其**模板形式**中对通过 `T` 的每个成员访问显示 `<dependent type>` — 这是正常的。析构函数脱糖**按实例化运行**，因此检查 `Vec<ConcreteType>::insert_at`（在 AST 转储中搜索 mangled/实例化形式）以查看真实的代码生成。

不要让模板形式的 `<dependent type>` 标记吓到你，导致错误的"编译器 bug"诊断。

### 调试析构函数相关重复释放的分类流程

1. **首先在 valgrind 下重现** — `valgrind --leak-check=no ./binary`。"Invalid read"堆栈跟踪显示失败的释放和先前的释放位置，以及 `malloc` 来源。这比在 abort 上使用 gdb 更好地定位 bug。
2. **检查测试顺序污染** — 失败的函数在只有该代码的新二进制文件中是否能正常工作？如果能，之前的测试留下了全局状态（解析器全局变量、分配计数器），稍后触发了 bug。
3. **转储驱动器模式 AST** 并验证可疑变量的 `_is_moved` 标志机制。
4. **在完成 1-3 之后**才考虑编译器级别的 bug 假设。已知的编译器 bug 确实存在（见 common-mistakes §8.6 了解 `if` 条件移动跟踪 bug），但库级别的别名是更常见的原因。

### AST 转储上有用的 grep 模式

```bash
# 保存完整 AST 供 grep 使用（驱动器模式！）
clang -Xclang -ast-dump -fsyntax-only file.cbs -I./include 2>/dev/null > ast.txt

# 查找 `varname` 的移动标记 VarDecl + 每个赋值 + 析构函数 IfStmt
grep -B1 -A2 "varname_is_moved" ast.txt

# 查找非模板代码上的所有 RecoveryExpr / 依赖类型标记（真正的 bug 指示器）
grep -B2 "RecoveryExpr\|<dependent type> contains-errors" ast.txt | grep -v "<T>"

# 只显示特定函数的函数体
sed -n '/FunctionDecl.*funcname/,/^|-/p' ast.txt

# 查找缺失的移动跟踪（声明但没有赋值给 1）
# 如果你看到 VarDecl '...is_moved' 但没有设置它的 BinaryOperator '='，
# 编译器错过了一个移动事件 — 很可能是 if 条件 bug。
```

### 从 AST 中发现 if 条件编译器 bug

如果 AST 对变量 `b` 显示：
- 一个 `VarDecl 'b_is_moved'`（已声明，初始化为 0）
- 一个检查 `!b_is_moved` 的析构函数 `IfStmt`
- 但两者之间**没有** `BinaryOperator '=' b_is_moved = 1`

…并且源代码中存在消耗 `b` 的函数调用 — 检查该调用是否出现在 `if`/`while`/`for`/`switch` 条件内部。如果是，你遇到了已知的编译器 bug。见 common-mistakes §8.6 了解变通方法。

## 7. 诊断抑制

BSC 安全检查默认是错误。要抑制特定诊断，在命令行使用 `-Eno-<identifier>` 或在源码中使用 `#pragma`。

### 命令行抑制：`-Eno-<identifier>`

```bash
# 抑制重复借用错误
/path/to/bsc/bin/clang -Eno-repeated-borrow file.cbs

# 抑制所有借用检查
/path/to/bsc/bin/clang -Eno-bsc-borrow file.cbs

# 抑制所有 BSC 安全检查
/path/to/bsc/bin/clang -Eno-bsc-safety-check file.cbs
```

### 源码内抑制：`#pragma`

```c
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Eassign-borrowed"
*p1 = 2;  // 此错误现在被抑制
#pragma GCC diagnostic pop
```

### 诊断标识符层级

```
bsc-safety-check                    # 所有 BSC 安全检查
├── nullability.md                 # 所有可空性检查
│   ├── deref-nullable              #   解引用可为空指针
│   ├── pass-nullable               #   传递可为空作为参数
│   ├── return-nullable             #   返回可为空指针
│   ├── cast-nullable               #   将可为空强制转换为非空
│   ├── assign-nullable             #   通过可为空访问成员
│   └── assign-nonnull              #   将可为空赋值给非空
├── ownership.md                   # 所有所有权检查
│   ├── use-owned                   #   移动后使用/未初始化组
│   │   ├── use-moved-owned         #     移动后使用
│   │   └── use-uninit-owned        #     使用未初始化
│   ├── assign-owned                #   赋值给拥有的组
│   │   ├── assign-moved-owned      #     赋值给已移动的值
│   │   └── assign-uninit-owned     #     赋值给部分未初始化的
│   ├── cast-owned                  #   强制转换为 void *_Owned
│   │   └── cast-moved-owned        #     强制转换已移动的值
│   ├── check-memory-leak           #   内存泄漏检测
│   ├── init-nonnull                #   _Nonnull 指针未初始化
│   ├── destruct-owned-struct       #   析构函数不正确
│   └── partially-moved-struct      #   作用域结束部分移动
└── bsc-borrow                      # 所有借用检查
    ├── assign-borrowed             #   赋值给借用的值
    ├── move-borrowed               #   移动借用的值
    ├── use-mutably-borrowed        #   可变借用时使用
    ├── repeated-borrow             #   多个可变借用
    ├── return-local-borrow         #   返回对局部的引用
    └── short-life-borrow           #   生命周期太短
```

> 关于错误消息和代码，见 `errors.md` 技能
