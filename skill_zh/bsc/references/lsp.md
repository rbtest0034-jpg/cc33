
# LSP 工具 — BSC 代码智能

LSP 工具（`hover`、`findReferences`、`goToDefinition`、`goToImplementation`、`documentSymbol`、`workspaceSymbol`、`prepareCallHierarchy`、`incomingCalls`、`outgoingCalls`）在 clangd 正在运行并配置为 `.cbs`/`.hbs` 时能感知 BSC。

## 重要：先检查 LSP 可用性

在依赖 LSP 之前，用一个调用进行探查（例如在已知符号上 `hover`）。如果返回 `No LSP server available for file type: .cbs`，则 BSC 感知的 LSP 未在此环境中注册。回退到手动分析（Read + Grep）并告知用户。

## 在推理所有权时主动使用 LSP

当你即将对涉及 `_Owned` 或 `_Borrow` 指针的 BSC 代码进行非平凡修改时，先使用 LSP，而不是从局部上下文猜测。具体来说：

1. **在添加/删除接受 `_Owned` 参数的函数调用之前**：在变量上 `hover` 以查看它是否已被移动。
2. **在引入新的借用之前**（`&_Const x` 或 `&_Mut x`）：在 `x` 上 `hover` 查看现有的借用者 — 重叠的可变借用 = 编译错误。
3. **当借用检查器错误提到一个变量时**：在该变量的声明上 `hover`。所有权流会告诉你它确切地在何处被移动/借用/释放。

## 在 `_Owned` / `_Borrow` 指针变量上 hover

返回变量在其函数内的完整所有权生命周期：

- **所有权流** — 带行号和代码片段的时间顺序事件：`Declared`、`Moved in from x`、`Moved out to y`、`Moved into foo()`、`Copied in`、`Mut borrow of x`、`Immut borrow of x`、`Mut reborrow of x`、`Init null`、`Freed`、`Dropped`
- **活跃范围** — 第一个事件行到最后一个事件行

示例输出：
```
Ownership Flow:
line 10: Declared
  int *_Owned p = safe_malloc(42);
line 18: Moved into consume()
  consume(p);
Live range: lines 10-18
```

## 已知限制 — 何时回退到 Grep

此子集只使用自由函数（成员函数被禁用 — 见 `overview.md` §3），因此 clangd 在 `obj->method(...)` 解析上的历史弱点在此不适用。剩下 Grep 更有优势的情况：

- **跨文件调用点枚举**：`findReferences` 在此子集中对普通自由函数是可靠的，但对于跨许多 `.cbs`/`.hbs` 文件的大规模重命名，在删除符号前用 `Grep` 交叉检查。
- **宏生成的调用点**：任何通过 `#define` 包装器调用的内容（例如分配器宏）对 LSP 是不可见的 — 使用 `Grep`。

## 净经验法则

- 使用 **LSP** 解决特定函数中特定变量的所有权/类型问题。
- 使用 **Grep** 进行跨文件发现和调用点枚举。
- 它们互为补充，而非替代。
