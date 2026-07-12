# AILang — White Paper（反映 v0.2.1 语法）

> **AILang：面向 AI 时代设计的高性能系统级编程语言。**
>
> 本白皮书是 AILang 的整合性概览：解释它为何存在、它长什么样、如何实现。
> 权威规范见 [SPEC.md](./SPEC.md)；设计动因见 [PHILOSOPHY.md](./PHILOSOPHY.md)；编译器实现见 [DESIGN.md](./DESIGN.md)。

---

## 摘要（Abstract）

软件开发正在进入一个新阶段：AI 不再只是辅助工具，而正在成为代码的主要生产者之一。然而，现有编程语言几乎全部围绕「机器」与「人类程序员」两个对象设计——它们对人友好、对编译器友好，却没有为「让 AI 精确理解程序」这一目标做过优化。结果是 AI 在生成代码时大量依赖猜测：猜测参数含义、猜测副作用、猜测错误可能、猜测资源归属。猜测是错误的来源。

**AILang 是第一代面向 AI 理解的系统级编程语言。** 它在源码层面强制程序主动表达**意图**——通过语义类型（Semantic Type，可带 constraint）、效果系统（Effect System）、显式所有权（Ownership）与结构化错误模型——并由编译器将这些语义编译为机器可读的元数据（`.ailmeta`），供 AI Agent、IDE 与自动化工具直接消费。同时，AILang 通过 LLVM 后端与所有权分析达到接近 Rust 的原生性能，是一个真正的系统级语言，而非脚本语言。

AILang 的目标是：**让人类与 AI 共同创造可靠、高性能的软件系统。**

---

## 1. 引言（Introduction）

### 1.1 AI 成为代码生产者

过去几十年的语言设计围绕两个对象：机器（C）、人类程序员（Java/Python/Go）、两者兼顾（Rust）。AI 编程时代引入第三个对象：**作为代码生产者的 AI**。这带来一个新问题：

> 如何让 AI **精确理解**程序，而不是通过猜测生成代码？

### 1.2 现有语言对 AI 不够友好

```ail
fn add(a: int, b: int) -> int {
    return a + b
}
```

编译器完全理解，人类大致理解，但 AI 缺少关键信息：`a`、`b` 的现实含义？是否允许负数？是否有副作用？是否可能失败？是否取得输入的所有权？这些信息在人脑中靠命名约定与经验补全，但对 AI 而言是搜索空间里的未知，只能靠猜。

### 1.3 AILang 的主张

> 优秀的代码不应该隐藏信息，而应该主动表达意图。

---

## 2. 设计哲学（Design Philosophy）

AILang 建立在四大原则之上（完整论述见 [PHILOSOPHY.md](./PHILOSOPHY.md)）：

1. **Rust 级安全** —— Ownership / Borrow / 静态类型，无追踪 GC（COW/RC 引用计数仍存，确定性释放），但**不复制 Rust 的复杂语法**：生命周期由编译器推导，不暴露 `'a`。
2. **Rust 级性能** —— 编译为原生二进制，经 LLVM，无虚拟机/解释器。
3. **Python 级简洁 + C 系大括号结构** —— 表达简洁如 Python，代码结构用 `{ }`（AST 稳定、IDE/AI 友好、复制不破坏结构）。
4. **AI First 类型系统** —— 类型 = 数据 + 含义 + 约束 + 行为；这是 AILang 与所有现有语言的最大区别。

> Rust 让程序员告诉编译器安全规则；AILang 让编译器自动推导安全规则，让 AI 更容易生成正确代码。

---

## 3. AILang 语言导览（AILang at a Glance）

下面是一个完整的 AILang 程序（`.ail`，大括号 + 可选分号），集中展示语义类型、结构体、错误枚举、效果标注与 `Result`：

```ail
package app

type UserId = int
meaning UserId:
    "unique identifier of a user"

struct User {
    id: UserId
    name: string
}

error UserError {
    NotFound
    PermissionDenied
}

fn get_user(id: UserId) -> Result<User, UserError>
effects {
    database.read
}
{
    // 简化示例；完整签名（含 db: borrow Database 参数）与实现见 EXAMPLES §19
    ...
}

fn main() -> void {
    let user = get_user(1)
    print(user)
}
```

### 编译它，会同时产出两样东西

**(a) 原生可执行文件** `app`（经 LLVM）。

**(b) 机器可读的语义元数据** `app.ailmeta`，这就是 AI 直接消费的东西：

```json
{
  "schema_version": "0.2.0",
  "language": "AILang",
  "package": "app",
  "functions": [
    {
      "identity": { "name": "get_user", "description": null },
      "span": { "file": "src/main.ail", "line_start": 10, "col_start": 1, "line_end": 14, "col_end": 1 },
      "input": [
        { "name": "id", "type": "UserId", "meaning": "unique identifier of a user", "mode": "copy" }
      ],
      "output": { "type": "Result<User, UserError>", "meaning": null },
      "effects": ["database.read"],
      "errors": ["NotFound", "PermissionDenied"],
      "ownership": { "output": "new" },
      "is_pure": false,
      "is_async": false
    }
  ],
  "types": [
    { "name": "UserId", "kind": "semantic", "base": "int", "meaning": "unique identifier of a user", "span": { "file": "src/main.ail", "line_start": 4, "col_start": 1, "line_end": 4, "col_end": 18 } },
    { "name": "User", "kind": "struct", "fields": [
        { "name": "id", "type": "UserId" },
        { "name": "name", "type": "string" }
    ], "span": { "file": "src/main.ail", "line_start": 8, "col_start": 1, "line_end": 11, "col_end": 1 } }
  ],
  "errors": [
    { "name": "UserError", "variants": ["NotFound", "PermissionDenied"], "span": { "file": "src/main.ail", "line_start": 13, "col_start": 1, "line_end": 16, "col_end": 1 } }
  ],
  "declarations": []
}
```

#### `.ailmeta` 正式字段规范（v0.2，权威；SPEC §25 #2）

**顶层**（`schema_version` / `language` / `package` required；`functions` / `types` / `errors` / `declarations` 按需）。`schema_version` 为**独立 semver**，与语言版本解耦。

**`function` 条目**：

| 字段 | 类型 | req | 说明 |
|------|------|-----|------|
| `identity` | `{name, description?}` | ✓ | 函数身份（description 来自 `///`） |
| `span` | `{file,line_start,col_start,line_end,col_end}` | ✓ | 源码定位（DESIGN §4.1） |
| `input[]` | 见下 | | 入参 |
| `output` | `{type, meaning?}` | | 返回类型 + 含义 |
| `effects[]` | `string[]` | | effect 词表（§6、SPEC §25 #6） |
| `errors[]` | `string[]` \| `{name, payload?}[]` | | **单一真源**：≡ 签名 `E`（error 枚举）变体集；裸变体为 `string`，带 payload 变体为 `{name, payload:[{name,type}]}`。非 `Result` 返回 → `[]` |
| `ownership` | `{output}` | | 见枚举（**不含** `inputs`——mode 已在 input[]） |
| `is_pure` | `bool` | ✓ | 无任何 effect（含 `alloc`）+ 不改入参；与 `errors[]` 正交 |
| `is_async` | `bool` | ✓ | `async fn` |

**`input` 条目**：`{name, type, meaning?, mode}`。

**`type` 条目**：`{name, kind, span, …}`；`kind` 决定附加字段——`semantic`(+base/meaning/constraint?)、`struct`(+fields[])、`enum`(+variants[])、`interface`(+sigs[])、`trait`(+sigs[])、`error`(+variants[])、`alias`(+base)。`type X = Y` 无 `meaning` → `alias`（透明同型）；附 `meaning` → `semantic`（名义，`constraint` 为可选字段，谓词如 `["value >= 0", "value <= 150"]`）。input / output / field 的 meaning 按名义类型附着传递；bare 基础类型 / alias = null。（类型层语义详见 SPEC §7、SPEC §29。）

**`error` 条目**：`{name, variants[], span}`；`variants[]` 项可为裸名或 `{name, payload:[{name,type}]}`（带数据变体，SPEC §26 #7）。

**`declaration` 条目**（顶级声明统一容器，SPEC §25 #4）：`{kind, name, span, …}`——`kind` ∈ {`agent`(+goal/tools)、`tool`(+fns，派生 OpenAI Schema)、`actor`(+state/messages/on-handler/ActorHandle)、`server`(+routes)、`task`(+sig)}。

**枚举汇总**：
- `mode`（input）：`move` / `borrow` / `borrow_mut` / `copy`（SPEC §25 #2 / #3）。
- `ownership.output`：`new`（新建）/ `borrowed`（借用返回）/ `move`（转移入参）。
- `kind`（type）：`semantic` / `struct` / `enum` / `interface` / `trait` / `error` / `alias`。

**这一份 `.ailmeta` 是 AILang 存在的全部理由。** AI 不需要「阅读源码并猜测」——它读到的是一份结构化、可验证、由编译器背书的程序语义说明书。

---

## 4. 核心语言（Core Language）

- **程序结构**：`package <路径>` 开头（点分模块路径，如 `package app` / `package std.http`），可跟 `import` / `from ... import` / `import ... as`（**全路径**，如 `import std.http`；SPEC §28）。
- **变量**：`let`（不可变）/ `var`（可变）/ `const`（编译期常量）。
- **函数**：`fn name(params) -> Type { }`；`pure` / `async` 为前缀修饰；`where` / `effects` / `requires` / `ensures` 契约块置于签名与函数体之间。
- **控制流**：`if` / `else` / `match` / `for` / `while` / `loop` / `break` / `continue`，统一 `{ }`。
- **关键字**：共 56 个，十一类（完整表见 [SPEC](./SPEC.md) §2）。

---

## 5. 类型系统（Type System）

> AILang 的类型 = 数据 + 含义 + 约束 + 行为。完整定义见 [SPEC](./SPEC.md) §7。

- **基础类型**：`int`（默认 i64）/ `uint`（默认 u64）/ `float`（默认 f64）/ `bool` / `string` / `byte` / `void`；需要时显式 `int8…int64` / `uint8…uint64` / `float32` / `float64`。
- **类型推断**：局部可推断；**函数参数与返回必须显式**（公共边界）。
- **Semantic Type（可带 constraint）**：`type UserId = int` + `meaning UserId: "..."` ——**名义新类型**，零成本（裸 `type X = Y` 无 `meaning` 为透明别名 `alias`，SPEC §29 #1）。附 `constraint` 即 Constraint Type（特化，非独立 kind，SPEC §29 #6）：`type Age = int` + `constraint Age: { value >= 0; value <= 150 }` ——字面量编译期检查 + 运行期断言（违约 panic `ConstraintViolation`）。
- **Optional\<T>**：`Some` / `None`，**禁止 null**。
- **Result\<T, E>**：错误安全核心。
- **复合类型**：`List` / `Array` / `Map` / `Set` / `Tuple`。
- **泛型**：`fn f<T>(...)`，trait bound 用 `where { Bound<T> }`（编译期，单态化）；`requires { }` 仅装运行期前置断言（SPEC §7.10、SPEC §23 #2）。
- **interface（服务能力） vs trait（类型能力）**：`struct X implements Trait { }`。

---

## 6. 效果系统（Effect System）

AI 生成代码最大的风险不是「不会写」，而是「不知道代码会产生什么影响」。AILang 让每个函数的能力边界可被静态分析：

```ail
fn save_user(user: User) -> Result<void, Error>
effects {
    database.write
    network
}
{
    ...
}
```

**推断为主，标注为约束**：编译器从 callee 效果并集推断完整集合；用户写的 `effects { }` 校验「推断集 ⊆ 标注集」。`pure` = 无任何效果（含堆分配 `alloc`）且不修改入参（SPEC §22 #9）。`.ailmeta` 永远输出真实、完整的效果集。

---

## 7. 所有权模型（Ownership Model）

Rust 级性能，但 AI 友好。**默认 move**，Copy 类型自动复制，`borrow` / `borrow_mut` / `move` / `copy` 显式控制：

```ail
let user = create_user()
process(borrow user)   // 只读借用，调用后 user 仍可用
process(user)          // move，之后 user 不可用
```

**关键设计：borrow 不逃逸 → 无生命周期标注。** `borrow` / `borrow_mut` 不能存入 struct 字段、不能返回、不能跨 `await` 存活，生命周期恒等于当前语句。编译器内部自动推导生命周期，程序员与 AI 都无需写 `'a`。线程安全（类 `Send + Sync`）也由编译器自动分析。

---

## 8. 错误模型（Error Model）

`Result<T, E>` 是核心机制；错误用 `error` 显式定义：

```ail
error UserError {
    NotFound
    PermissionDenied
}

fn get_user(id: UserId) -> Result<User, UserError> { ... }
```

**错误从返回类型派生**：返回 `Result<T, E>` 且 `E` 为 `error` 枚举时，其 variant 自动成为函数的 `errors`，进入 `.ailmeta`。`try/catch/throw` 是 `Result` 的语法糖（精确语义见 SPEC §9 / SPEC §20 #4）。

---

## 9. AI 元数据（AI Metadata）

传统编译器只产出二进制。AILang 的编译器在编译同时产出结构化的语义元数据 `*.ailmeta`，提供：身份与意图（`identity`）、输入/输出语义（含 `meaning` 与所有权 `mode`）、行为边界（`effects`）、失败可能（`errors`）、所有权签名、源码定位。

**关键：`.ailmeta` 的内容不是注释或约定，而是编译器强制验证后的事实。** 若一个函数声称 `pure`，编译器已校验它确实无副作用；若声称 `effects { database.read }`，编译器已校验它不会写库。AI 可以信任 `.ailmeta`，如同信任类型签名。

---

## 10. 编译器架构（Compiler Architecture）

> 完整实现设计见 [DESIGN.md](./DESIGN.md)。本节给出概貌。

```
main.ail
   │  Lexer        词法
   ▼
Tokens  +  trivia（doc 注释）
   │  Parser       语法
   ▼
raw AST                    只有结构 + doc 注释 + 显式注解
   │  Semantic Analysis   类型 / 约束 / 效果 / 所有权 / 契约
   ▼
AILang IR (HIR)            富语义 IR —— .ailmeta 的数据源
   ├──► meta               ──►  program.ailmeta (JSON)
   └──► codegen            ──►  LLVM IR ──►  native binary
```

**关键架构理念：AST 分层。** Parser 只能看到源码字面；语义信息（类型、效果、所有权）是分析出来的，不是解析出来的。因此 raw AST 与富语义 AILang IR（HIR）是两套独立类型——**HIR 就是 AI 真正读取的东西**。编译器以 Rust 实现，按阶段分 crate，用 Cargo 强制阶段边界。

---

## 11. 与现有语言的关系（Comparison）

AILang 吸收各所长处，补上 AI 原生语义这一缺失维度：

| 语言 | AILang 吸收 | AILang 的不同 |
|------|-------------|---------------|
| **Rust** | 性能、内存安全、所有权 | 无生命周期标注；borrow 不逃逸；产出 AI 元数据 |
| **TypeScript** | 类型表达、接口 | 编译型系统语言；名义语义类型 + 约束类型 |
| **Python** | 简洁表达 | 静态类型、零运行时开销 |
| **Go** | 大括号工程结构、少而明确 | 显式效果与所有权 |

---

## 12. 路线图与现状（Roadmap & Status）

AILang 按阶段交付，权威路线见 [DESIGN.md §15](./DESIGN.md)：

| Phase | 时间 | 产出 | 验证标准 |
|-------|------|------|----------|
| 设计 | — | 哲学 + 规范(v0.2.1) + 架构 + 白皮书 + 内存/并发模型 | ✅ 完成 |
| **1** | 1–3 月 | Lexer / Parser / AST / 基础类型 / 函数 / struct / **最小 LLVM 输出** | `fn main() { print("hello") }` 编译为原生二进制并运行 |
| **2** | 3–6 月 | `Result` / `enum` / 泛型 / `interface` / **`.ailmeta`** | 端到端：源码 → 可信 AI 元数据 |
| **3** | 6–12 月 | **Ownership + Borrow Checker** / 并发 / 标准库核心 | 所有权错误用例全部报出 |
| **4** | 1 年+ | 包管理 / IDE / Debugger / AI Agent 生态 | 生产级 |

**关键洞察**：`.ailmeta` 由 HIR 直接生成，**不依赖 LLVM codegen**——即便在 Phase 1 最小 codegen 阶段，也能独立验证「源码 → 可信 AI 元数据」这一 AILang 核心主张。Phase 2 的 `.ailmeta` 闸门即据此设计。

---

## 13. 结语（Conclusion）

> 当 AI 写软件成为主流时，现有语言是否仍然适合？

AILang 的回答：**未来的语言必须不仅让人理解，也必须让 AI 精确理解。**

语义类型（可带约束）让概念显式、边界可验，效果系统让行为透明，所有权让资源归属清晰，AI 元数据让这一切可被机器直接消费——而这一切都不以牺牲原生性能为代价。AILang 希望成为人类与人工智能之间的新一代软件表达语言。

---

## 延伸阅读

- [SPEC.md](./SPEC.md) —— 权威语言规范（关键字 / 文法 / 类型 / 所有权，v0.2.1）
- [PHILOSOPHY.md](./PHILOSOPHY.md) —— 设计哲学（四原则）
- [DESIGN.md](./DESIGN.md) —— 编译器架构与各模块实现设计
- [EXAMPLES.md](./EXAMPLES.md) —— 可运行样例集（Hello World → AI Agent，对齐 v0.2.1）
- [STDLIB.md](./STDLIB.md) —— 标准库 API 规范 + `ail` 包管理器与包生态
- [CONCURRENCY.md](./CONCURRENCY.md) —— 并发模型（Task / spawn / async-await / Channel / select / Actor）
- [MEMORY.md](./MEMORY.md) —— 内存模型（Ownership / Borrow / 生命周期推导 / Drop / Unsafe / FFI）
