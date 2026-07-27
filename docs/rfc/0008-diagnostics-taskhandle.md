# RFC 0008 · P0-3 诊断协议 + 错误码分类法 + TaskHandle 错误态—— 结构化诊断 / AILxxxx 编号空间 / Task Failed 闭合

| | |
|---|---|
| **状态** | 草案（Draft v1）—— 待 review（尚未跑对抗式 workflow；目标收敛 0H/0M/0L，对齐 RFC 0001 v6 / 0002 v8 / 0003 v5 / 0004 v5 / 0005 v1 / 0006 v1 / 0007 v1）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94 语义决断、56 关键字、110 决议均不变；TaskHandle 新查询方法 `is_failed`/`status`/`failure` 为 `std.async` 方法非关键字——同 `cancelled`/`yield` 先例；`AILxxxx` 错误码为登记表标识符非关键字；零新关键字、零新文法产生式）|
| **日期** | 2026-07-27 |
| **分级** | **P0-3**（综合判断 [`docs/research/synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 第四优先级；[`deep-review-2026-07.md`](../research/deep-review-2026-07.md) 可观测维度 4.0「运行期近零」+ P1「Task 失败语义闭环（RFC 0003 已依赖）」；[RFC 0003](./0003-pbt-fuzzer.md) PBT 每 trial 独立 Task 捕获 panic 的契约前置）|
| **承接** | [`deep-review-2026-07.md`](../research/deep-review-2026-07.md) 可观测 4.0 + AI 友好 6.0「meaning 不可验证」；[`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §4 交叉印证 #11（结构化诊断未形式化）+ #6（运行期可观测缺失）；[RFC 0005](./0005-spec-governance.md) §3 Conformance「诊断信息应结构化（§69.3）」——本 RFC 给该引用补上**协议定义**；[RFC 0003](./0003-pbt-fuzzer.md) §「每 trial 独立 Task 捕获 panic = 唯一可观测 unwind 边界」——本 RFC 定义该边界的可观测契约；[ailang-ai-friendliness 存档](../research/ai-friendliness-2026-07.md) §9「结构化诊断一等公民」（5 个 AI-era 加法之一）|

---

## 1. 动机

deep-review 把 **可观测维度评为 4.0**——全场最弱项之一，判决「运行期近乎零」。三个具体空洞：

- **结构化诊断未形式化**——§69.3 的 `Diagnostic` 是 ail-ast 内部 Rust struct（3 级 severity、自由 `code: Option<&str>`），**无机器可读的发射协议**。AILang 的存在理由是「代码表达意图 → `.ailmeta` → AI 零猜测」，但编译器**吐给 AI 的诊断**却是非结构化的人读文本——AI 自修复（RFC 0003 fuzz self-repair、RFC 0004 结构化结果）恰恰依赖结构化诊断作输入。
- **错误码体系未建**——§18.2 仅一句「`AIL2xxx` = 所有权/内存类……统一错误码体系待后续设计阶段细化」。运行期 panic 用散落命名（`ArithmeticOverflow`/`DivideByZero`/`IndexOutOfBounds`/`ConstraintViolation`），**无稳定标识符、无分类法、无版本化**——AI 无法跨编译器版本稳定匹配诊断。
- **TaskHandle 错误态未闭合**——§17 line 736 与 §3119 都写「Task 内 panic → TaskHandle 错误态」，但 §21.8 生命周期图**只有 `Cancelled` 一个非 `Completed` 终态，无 `Failed`**；`TaskHandle` 仅有 `cancelled()` 查询，**无失败查询、无 join 失败传播规则**。这是 RFC 0003 PBT「每 trial 独立 Task 捕获 panic = 唯一可观测 unwind 边界」直接依赖却未定义的契约。

**三者为何同批**。它们是**同一主题「让失败可被观测」的编译期 / 标识 / 运行期三面**：诊断协议 = 编译期失败如何结构化吐出；错误码 = 失败如何稳定命名；TaskHandle 错误态 = 运行期 Task 失败如何可观测。合在一起把「可观测 4.0」从结构上抬起，并闭合 RFC 0003 的前置依赖。

**本 RFC 的性质**：纯**工具链 / 运行时可观测层**——零新关键字、零新文法产生式；`TaskHandle` 查询方法为 `std.async` 方法（同 `cancelled`/`yield` 先例，§87 已确认此类不计入 56）；`AILxxxx` 为登记表标识符。不修改 §17 panic 模型、§87 #4 spawn/cancel、§89 #8 并发错误表任一冻结决议——仅**扩展填补**「错误态未定义」「诊断未协议化」「错误码未分类」三个被前轮决议引用却悬空的缝隙。

---

## 2. 设计目标

1. **不触动冻结语义决断**——§1–§94 语义、56 关键字、§83–§94 的 110 决议不变。本 RFC 是工具链 / 运行时可观测层扩展（§4 协议化 §69.3、§5 建 AILxxxx 登记、§6 闭合 TaskHandle 错误态）。
2. **零新关键字 / 零新文法产生式**——`is_failed`/`status`/`failure` 为 `std.async` 方法、`TaskStatus`/`TaskFailure`/`DiagnosticCode` 为 std/编译器类型、`AILxxxx` 为登记标识符，均非关键字、不进 §27。
3. **AI 可消费优先**——诊断协议**必须**有机器可读发射（JSON Lines），错误码**必须**有跨版本稳定的 slug，让 AI 工具能稳定匹配诊断并自修复（对齐「结构化诊断一等公民」AI-era 加法）。
4. **与既有自洽**——§6 TaskHandle 错误态与 §17 panic（确定性栈展开至 Task 边界）、§87 #4（spawn 返 TaskHandle + 结构化作用域）、§89 #8（并发错误统一表「不隐藏错误」）完全对齐；§5 错误码覆盖 §17 已列的全部 panic 命名（§11 自洽核查表）。
5. **失败必可观测、不静默吞**——延续 §21.9 / §87 #1「不隐藏错误」哲学：Task 失败**必须**有可观测出口（join 传播或显式查询），不得 fire-and-forget 静默丢失。

---

## 3. 现状与缺口诊断

| 子项 | 现状（v0.2.1） | 缺口 | 决断（本 RFC） |
|---|---|---|---|
| 诊断协议 | §69.3 `Diagnostic`（3 级 severity、`code: Option<&str>` 自由串、单 span、无 suggestion）；「所有阶段输出 `Vec<Diagnostic>`」有人读渲染、**无机器可读发射** | severity 扩展 + 多定位 + 机器可应用建议 + JSON 发射协议 | **§4 结构化诊断协议** |
| 错误码 | §18.2 仅「`AIL2xxx` = 所有权/内存，待细化」；运行期 panic 散落命名（`ArithmeticOverflow` 等）；编译诊断 code 自由串 | 统一编号空间 + 分类法 + 稳定 slug + 登记 | **§5 AILxxxx 错误码分类法** |
| TaskHandle 错误态 | §17 / §3119 引用「→ TaskHandle 错误态」；§21.8 生命周期图**仅 `Cancelled`**、无 `Failed`；仅 `cancelled()` 查询 | `Failed` 终态 + 失败查询 + join 传播规则 | **§6 TaskHandle 错误态闭合** |

---

## 4. 结构化诊断协议（扩展 §69.3）

落地形态：§69.3 `Diagnostic` 扩展 + 新增「诊断发射协议」小节。

### 4.1 Diagnostic 扩展

```rust
pub enum Severity { Error, Warning, Note, Hint, Help }
//   Hint = 指向可能的改进（非错误）；Help = 机器可应用的修复建议

pub enum DiagnosticCategory {
    Lex, Parse, Names, Type, Borrow, Effect, Visibility, Contract, Concurrent, Ffi, Codegen, Other
}
//   Ffi = FFI / ABI / 布局违规（§5 AIL8xxx）；Codegen = lowering / 代码生成失败

pub struct RelatedSpan { pub label: String, pub span: Span }      // 「定义于此」「前一次借用于此」

pub struct Suggestion { pub span: Span, pub replacement: String } // 机器可应用补丁（span → 替换文本）

pub struct Diagnostic {
    pub severity: Severity,                 // Error | Warning | Note | Hint | Help（原 3 级 → 5 级）
    pub code: DiagnosticCode,               // 原 Option<&str> → 稳定码（§5），杜绝自由串
    pub category: DiagnosticCategory,       // 新增：产诊断的 pass（AI 按类别聚类）
    pub message: String,                    // 人读文本（可跨版本改）
    pub span: Span,                         // 主定位（不变）
    pub related: Vec<RelatedSpan>,          // 原 notes → 结构化多定位（带 label + span）
    pub suggestion: Option<Suggestion>,     // 新增：机器可应用修复建议
}
```

**设计说明**：
- `code` 由自由 `Option<&str>` 收紧为 `DiagnosticCode`（§5）——杜绝每个 pass 各自发明 code 字符串。
- `related` 取代原 `notes: Vec<(String, Option<Span>)>`——结构化（label + span），让 AI 能精确定位「定义处」「冲突处」等多定位。
- `suggestion` 是**机器可应用**修复（span + 精确替换文本），区别于 `Help` 文本提示——AI 自修复可直接套用（对齐 RFC 0003 fuzz 反例→self-repair 的机器闭环）。
- `Hint` / `Help` 分离：Hint 是「可改进」建议（如 unused），Help 是「如何修」——对应 rust-analyzer / clang tidy 的 hint vs fix 区分。

### 4.2 诊断发射协议（AI 可消费契约）

合规编译器**必须**支持两种诊断发射格式：

| 格式 | 触发 | 用途 |
|---|---|---|
| **human**（默认） | `ail build` / `ailc`（无 flag） | 彩色终端、上下文截取、多定位渲染（§69.1 现状） |
| **json** | `ailc --diagnostics-format=json` | **JSON Lines**：每行一条 `Diagnostic`，稳定 schema，机器消费（AI 工具 / 编辑器 / CI） |

**JSON schema**（独立 `diagnostics_schema_version` semver，与 `.ailmeta.schema_version` 解耦）。

**规范性字段表**（v0.3 `diagnostics_schema_version = 0.1.0`；JSON Lines 每行一条对象，字段如下）：

| 字段 | 类型 | 必需 | 语义 |
|---|---|---|---|
| `diagnostics_schema_version` | string（semver） | 是 | 本行 schema 版本（顶层，与 `Diagnostic` 同对象） |
| `severity` | enum string：`"error"\|"warning"\|"note"\|"hint"\|"help"` | 是 | §4.1 Severity 5 级的 JSON 串形（小写） |
| `code` | object `{number, slug}` | 是 | `{number:"AILxxxx", slug:"kebab-case"}`；slug 跨版本稳定（见下） |
| `category` | enum string：`"lex"\|"parse"\|"names"\|"type"\|"borrow"\|"effect"\|"visibility"\|"contract"\|"concurrent"\|"ffi"\|"codegen"\|"other"` | 是 | §4.1 DiagnosticCategory 的 JSON 串形（小写） |
| `message` | string | 是 | 人读文本（可跨版本演进，非契约） |
| `span` | `Span`（见下） | 是 | 主定位 |
| `related` | array of `{label, span}` | 否 | 结构化多定位；空时可省 |
| `suggestion` | object `{span, replacement}` | 否 | 机器可应用补丁；无建议时省略 |

**`Span` 规范形状**（**全规范统一**——§69.1 人读渲染、§24 `.ailmeta` 引用、RFC 0003 PBT 反例定位、本协议 JSON 全部使用同一 `Span`，消除三处各自表述的冲突）：

| 字段 | 类型 | 必需 | 语义 |
|---|---|---|---|
| `file` | string | 是 | 源文件路径（相对包根或绝对） |
| `line` | int（1-based） | 是 | 起始行 |
| `col` | int（1-based） | 是 | 起始列（字节偏移或字符偏移由实现文档化，二者择一须全项目一致） |
| `start` | int | 是 | 文件字节偏移起点（0-based） |
| `end` | int | 是 | 文件字节偏移终点（exclusive） |

**`suggestion` 应用语义**（机器补丁）：消费方（AI / 编辑器）应用建议 = 将 `[span.start, span.end)` 区间替换为 `replacement` 字符串。`replacement` 为字面替换文本（非 diff / 非 AST 变换），应用后**应**重新编译校验（建议非权威修复，可能引入新诊断）。`suggestion` 的 `span` 必落在 `Diagnostic.span` 或其 `related[].span` 之一内（建议须指向被诊断的代码区域）。

- `code.slug` 为**跨版本稳定标识**（AI 工具按 slug 匹配，不按可能重编的 number）。
- `ailc --diagnostics-format=json` 退出码：有 Error 级诊断 → 非零；仅 Warning/Note/Hint/Help → 零（不阻断）。
- 未来可扩 `sarif`（开放问题 #3），但 `json` 为 v0.3 必备。

> 此协议是「结构化诊断一等公民」AI-era 加法（研究存档 §9）的落地——把编译器**吐给 AI 的失败**从人读文本提升为稳定 schema，使 AI 自修复 / fuzz 反例闭环（RFC 0003）/ 验证器结构化结果（RFC 0004）有共同的机器输入。

---

## 5. 统一错误码分类法（AILxxxx 编号空间）

落地形态：§18.2「错误码方案」展开为完整编号空间 + 登记（附录 E 或扩附录 B）。

### 5.1 编号空间

把 §18.2 的 `AIL2xxx`（所有权/内存）扩展为全分类法（按编译期 pass + 运行期 panic 分段）：

| 段 | 类别 | 种子码（示例，非穷尽） |
|---|---|---|
| `AIL0xxx` | 编译器内部 / 不该发生（ICE） | `AIL0001 internal-compiler-error` |
| `AIL1xxx` | 词法 / 语法（lex / parse） | `AIL1001 unexpected-token`、`AIL1002 unterminated-string` |
| `AIL2xxx` | 所有权 / 借用 / 内存（**§18.2 既有**） | `AIL2001 use-after-move`、`AIL2002 borrow-conflict` |
| `AIL3xxx` | 类型系统（推断 / 泛型 / trait / constraint 构造） | `AIL3001 type-mismatch`、`AIL3002 missing-trait-impl` |
| `AIL4xxx` | 名字解析 / 可见性 / 模块 | `AIL4001 unresolved-name`、`AIL4002 unresolved-member`、`AIL4003 import-conflict` |
| `AIL5xxx` | **编译期**契约 / 约束 / 效果诊断 | `AIL5001 contract-requires-static-reject`（`requires` 静态可证伪时编译期拒收）、`AIL5002 constraint-violation`（编译期约束 bound 检查）、`AIL5003 effect-violation` |
| `AIL6xxx` | 并发 / Send-Sync / Task | `AIL6001 not-send`、`AIL6002 not-sync`、`AIL6003 spawn-bound-check` |
| `AIL7xxx` | **运行期 panic** | `AIL7001 arithmetic-overflow`、`AIL7002 divide-by-zero`、`AIL7003 index-out-of-bounds`、`AIL7004 constraint-violation-runtime`（约束 `T(expr)` 运行期断言违约）、`AIL7005 unwrap-on-none`（`Option::unwrap` on `None`）、`AIL7006 unwrap-on-err`（`Result::unwrap` on `Err`）、`AIL7007 assertion-failed`（`assert` / 断言宏）、`AIL7008 contract-requires-violation-runtime`（§20 `requires` 运行期违约）、`AIL7009 contract-ensures-violation-runtime`（§20 `ensures` 运行期违约） |
| `AIL8xxx` | **FFI / ABI / 布局 / codegen** | `AIL8001 ffi-unsafe-type`（非 FFI-safe 类型入 `extern` 签名，[RFC 0009](./0009-ffi-abi-supply-chain.md) §4）、`AIL8002 abi-mismatch`（调用约定越封闭集合，0009 §5）、`AIL8003 layout-mismatch`（`layout` 组合非法，0009 §7.2）、`AIL8004 ffi-marshalling`（封送不可表示）、`AIL8005 codegen-lowering`（lowering / 代码生成失败） |

**段边界不变量**（消除 5xxx 与 7xxx 的口径重叠、闭合 codegen/FFI 无家可归）：
- **5xxx = 编译期**契约 / 约束 / 效果诊断（在静态分析 pass 拒收）；**7xxx = 运行期 panic**（程序运行时触发）。`requires` / `ensures`（§20）为**运行期**前后置断言——其运行期违约落 **AIL7008 / 7009**（7xxx），**不**入 5xxx；5xxx 的 `AIL5001` 仅覆盖编译器能**静态证明** `requires` 必假而提前拒收的罕见情形（静态拒收，非运行期）。
- **8xxx = FFI / ABI / 布局 / codegen**——§25.4 `layout(C)` 违规、`extern` 签名 FFI-safe 校验（[RFC 0009](./0009-ffi-abi-supply-chain.md) §4/§5）、调用约定封闭集合校验、lowering 失败等**无既归宿**的诊断统一落此段（DiagnosticCategory `Ffi` / `Codegen`）。RFC 0009 的 FFI 违规**接入 8xxx**（非 6xxx——6xxx 保持纯并发）。

### 5.2 码条目结构

```rust
pub struct DiagnosticCode {
    pub number: &'static str,     // "AILxxxx"，人读；major 版本内 append-only，跨 major 可重编
    pub slug: &'static str,       // 稳定 kebab-case 标识（AI 匹配键），一旦发布不变
    pub default_severity: Severity,
    pub category: DiagnosticCategory,
}
```

- `number`：人读编号，major 版本内仅 append（不回收、不改义）；跨 major 可重编。
- `slug`：**机器稳定标识**——一经发布**不得**改名 / 改义（AI 工具、CI、文档锚点依赖）；这是 AI 可消费的真正契约。
- `message` 模板可跨版本演进（人读文本非契约）。

### 5.3 运行期 panic 接入错误码

§17 已列的 panic 命名（`ArithmeticOverflow`/`DivideByZero`/`IndexOutOfBounds`/`ConstraintViolation`/`unwrap`/`assert`）**映射为 `AIL7xxx`**——运行期 panic 现在携带 `DiagnosticCode`（而非裸命名串）。panic payload（§6 `TaskFailure`）含该 code，使运行期失败有稳定标识、可被 AI / 监控稳定匹配——闭合「可观测 运行期近零」的核心空洞。

### 5.4 与用户 `error` 枚举的边界

**用户定义的 `error` 枚举（§17）不在 `AILxxxx` 空间**——它们是**应用错误**（经 `Result<T,E>` 传播、进 `.ailmeta` `errors[]`），不是语言 / 工具链诊断。`AILxxxx` 仅覆盖**编译期诊断 + 运行期 panic**（语言 / 工具链自身吐出的失败）。此分离避免应用错误污染语言诊断码空间。

> 登记表（附录）为种子码 + 增长机制；本 RFC 定义空间 + 初始种子，完整登记随编译器 pass 实现增量补全（每个 pass 落地时登记其码）。

---

## 6. TaskHandle 错误态闭合（扩展 §21.8 / §87 #4）

落地形态：§21.8 生命周期图加 `Failed` 终态 + `TaskHandle` 查询方法 + join 失败传播规则。

### 6.1 生命周期扩展

```
Created → Running → Waiting → Completed
   │          │          │
   │          │          ├──► Cancelled          // 既有：协作式取消（§21.8，可发生于 Running/Waiting 任一）
   │          │          │
   │          ├──┐       └──► Failed             // 新增：Task 内 panic 在边界被吸收为失败态
   │          │  │
   └──────────┘  └──► Failed                    // panic 可发生在 Running（CPU-bound）或 Waiting（await/recv 时）任一非终态
```

- `Failed` 为**新增终态**：Task 内 panic（§17）可发生于 `Running`（CPU-bound 执行中）或 `Waiting`（阻塞于通道收发 / await 时）任一**非终态**，展开至 Task 边界（§89 #3）→ 该 Task 进入 `Failed`（而非 `Completed`/`Cancelled`），**不传染进程**（§17 既有立场不变）。`Created`（尚未调度）不 panic——panic 仅在 Task 实际执行 / 阻塞期间发生。
- panic 的 `DiagnosticCode` + message + span 由 runtime 捕获为 `TaskFailure`（§6.2），挂在该 TaskHandle 上。

### 6.2 TaskHandle 查询 API（std.async 方法，非关键字）

| 方法 | 签名 | 说明 |
|---|---|---|
| `cancelled()` | `borrow self -> bool` | 既有（§21.8） |
| `is_failed()` | `borrow self -> bool` | **新增**：Task 是否处于 `Failed` 态 |
| `status()` | `borrow self -> TaskStatus` | **新增**：忠实反映 §21.8 生命周期当前态（见下枚举全集） |
| `failure()` | `borrow self -> Optional<TaskFailure>` | **新增**：`None` 除非 `Failed`；**取回即确认（ack）**（§6.3） |

```ail
enum TaskStatus { Created, Running, Waiting, Completed, Cancelled, Failed }
//   Created  = 已 spawn 未调度（§21.8 初始态）
//   Running  = CPU-bound 执行中
//   Waiting  = 阻塞于通道收发 / await（§21 / §87）
//   Completed/Cancelled/Failed = 三个终态（Failed 见 §6.1，可由 Running/Waiting 转入）

struct TaskFailure {
    code: DiagnosticCode,        // panic 码（§5），如 AIL7001 arithmetic-overflow
    message: string,             // panic 人读消息
    span: Optional<Span>,        // 源码定位（若可恢复）
}
```

> `TaskStatus` 枚举为 §21.8 生命周期的**忠实投影**（`Created`/`Running`/`Waiting` 为非终态、三个终态收尾）——与 §6.1 生命周期图一一对应，不留「blocked / unscheduled Task 无 `status()` 返回值」的缺口。`is_failed()` ≡ `status() == Failed` 的便捷查询。

> `is_failed` / `status` / `failure` 为 `std.async` 方法，同 `cancelled` / `yield` / `task.yield()` 先例（§87 已确认此类上下文方法不计入 56 全局关键字）。`TaskStatus` / `TaskFailure` 为 std 类型（同 `TaskHandle` 自身）。

### 6.3 join 失败传播规则（核心新决断）

延续 §21.9 / §87 #1「不隐藏错误」哲学，定**结构化并发失败传播**：

1. **scoped task（默认，§87 #4）**：作用域末隐式 join 时，若子 Task 处于 `Failed`：
   - **父已取回失败**（调过 `h.failure()` 得到 `Some`）→ 视为**已确认（ack）**，**不**向父传播（父已知情并处理）；
   - **父未取回**（未观测的 `Failed`）→ **panic 传播至父**（父以该 `TaskFailure` panic）——失败**绝不**静默丢失。
2. **`spawn detached`（显式分离，§87 #4）**：无作用域父可传播 → 失败仅经 `h.failure()` 显式查询可观测。**未被 ack 的 detached `Failed` 的投递时机与互斥**（闭合「detached 失败可能永不报告」+「ack 后又投递全局处理器」双重漏洞）：
   - **ack 互斥**：`h.failure()` 返回 `Some` 即**唯一 ack 点**——已 ack 的失败**不得**再投递全局处理器（`failure()` 取回与全局处理器投递**互斥**，杜绝同一失败被双重处理）。
   - **handle drop 触发**：detached `TaskHandle` 被 drop 且 `failure()` 从未返回 `Some`（未 ack）时，runtime 在 drop 点把该 `TaskFailure` 投递全局处理器 `std.async.on_unhandled_task_failure: fn(TaskFailure) -> void`——失败**不**因 handle 被丢弃而丢失。
   - **进程退出 sweep**：被**永久持有**（immortal handle，从不 drop）且未 ack 的 detached `Failed`，runtime 在**进程退出前** sweep 全部此类未观测失败，逐一投递全局处理器——保证「从未被观测」的 detached 失败最终必经全局处理器出口。
   - 全局处理器默认实现：记录到诊断日志 + 继续；可由用户覆写为 panic / 上报 / 忽略。
3. **actor 死信（§17 / §21.7）**：actor `on` handler panic → actor 进入死信态（既有），其失败信息同样以 `TaskFailure` 形式可经 `ActorHandle` 查询（与 TaskHandle 对称，细节留开放问题 #4）。

**设计说明**：
- 「取回即确认」把 `failure()` 定为**唯一消费式 ack**——与 `Result` 必须 handle 的语义同构：你看了就是你的，没看就崩你（scoped）或经 drop/exit-sweep 进全局处理器（detached）。简单、sound、AI 可预测。
- **ack 与全局处理器互斥**：一个 `TaskFailure` 的生命周期是「未观测 →（ack 经 `failure()` 取回 ⟹ 终止）∨（未 ack ⟹ drop 或 exit-sweep 时投递全局处理器 ⟹ 终止）」——单一出口、不重复、不丢失。
- 默认传播保证「不隐藏错误」——一个 panic 的子 Task 不会被静默吞掉，要么父显式处理，要么父跟着 panic（scoped）/ 进全局处理器（detached）。
- detached 的全局处理器是 fire-and-forget-adjacent 模式的逃生舱（日志 / 上报），但不静默——有可观测出口，且 immortal handle 由 exit-sweep 兜底。

> 此规则闭合 RFC 0003 PBT「每 trial 独立 Task 捕获 panic = 唯一可观测 unwind 边界」——该边界现为**规范契约**：trial Task 的 panic 确定地成为 `TaskFailure`，runner 经 `h.failure()` 可靠取回（确定性、可复现），支撑 PBT shrinking / fuzz 反例的稳定捕获。

---

## 7. 与 RFC 0005 / RFC 0003 的交叉更新

本 RFC 落地后，既有 RFC 的引用点更新（合并时同步）：

| 既有引用 | 现状 | 本 RFC 后 |
|---|---|---|
| RFC 0005 §3 Conformance「诊断信息应结构化」 | RFC 0005 §3 #4 **已**指向本 RFC §4 的 JSON Lines 协议（并显式标注 §69.3 仅为 Part IV 资料性参考）——合并时此引用生效 | 合并后 RFC 0005 §3 #4 的引用与本 RFC §4 自洽（§69.3 不升格为合规义务来源） |
| RFC 0005 §1.5.4 行为分类「panic 是定义行为」 | panic 定义但失败可观测性未提 | 补：panic 经 TaskHandle `Failed` + `TaskFailure` 可观测（§6），运行期失败有稳定 `DiagnosticCode`（§5） |
| RFC 0003 PBT「每 trial 独立 Task 捕获 panic」 | 依赖未定义的「Task 错误态」 | 现为规范契约（§6.3 join 传播 + `failure()` 取回），trial 捕获确定可靠 |
| RFC 0003 `panic_symbol`（PascalCase）vs 本 RFC `code.slug`（kebab-case） | 两套标识符命名风格、无映射 | RFC 0003 的 `panic_symbol` **对齐**本 RFC `slug`（同一机器稳定键）：`panic_symbol` 为 `slug` 的 PascalCase 别名（如 `ArithmeticOverflow` ≡ slug `arithmetic-overflow`）——合并时 RFC 0003 统一改为引 `slug`，`panic_symbol` 作为人读别名保留或移除（留 RFC 0003 review） |
| RFC 0004 结构化结果六态 | `failed` 反例为人读 | 可引用 `DiagnosticCode` slug 作机器稳定键（与本 RFC §5 slug 体系一致） |

> 即：本 RFC 把 deep-review 可观测 4.0 的两大空洞（编译期诊断非结构化、运行期失败不可观测）从结构上补齐，并为 RFC 0003 / 0004 的机器闭环提供共同诊断输入。

---

## 8. 不变量与自洽核查

| 本 RFC 条款 | 对齐的既有规范 | 自洽性 |
|---|---|---|
| §4 Diagnostic severity 5 级 | §69.3 现状（3 级扩展，非替换语义） | ✅ 超集 |
| §4 `code: DiagnosticCode` | §5 AILxxxx（替换自由 `Option<&str>`） | ✅ 收紧 |
| §5 `AIL2xxx` 所有权段 | §18.2 既有「`AIL2xxx` = 所有权/内存」 | ✅ 完全对齐（展开） |
| §5 `AIL5xxx`(编译期)/`AIL7xxx`(运行期) 边界 | §20 `requires`/`ensures` 为运行期断言（→ 7xxx 的 7008/7009）；编译期约束/效果留 5xxx | ✅ 消除 5xxx↔7xxx 口径重叠（运行期 vs 编译期二分） |
| §5 `AIL7005`/`7006` 拆分（unwrap-on-none / unwrap-on-err） | §17 `unwrap` 两种触发（Option vs Result） | ✅ 一码一 slug（消除一码两 slug 歧义） |
| §5 `AIL7xxx` panic 段 | §17 panic 命名（ArithmeticOverflow 等） | ✅ 一一映射 |
| §5 `AIL8xxx` FFI/ABI/布局/codegen 段 | §25.4 `layout(C)` / [RFC 0009](./0009-ffi-abi-supply-chain.md) §4 FFI-safe / §5 ABI 封闭集 | ✅ 闭合 codegen/FFI 诊断无家可归（DiagnosticCategory `Ffi`/`Codegen` 各有 8xxx 码） |
| §5 用户 error 不在 AILxxxx | §17 `error` 枚举 + §89 #10 `errors[]` 单一真源 | ✅ 命名空间分离 |
| §6 `Failed` 终态（Running/Waiting 均可转入） | §17 panic 展开至 Task 边界（不传染进程） | ✅ 一致（边界吸收为 Failed） |
| §6 `TaskStatus` 忠实投影（Created/Running/Waiting + 三终态） | §21.8 生命周期全集 | ✅ 一一对应（无「blocked Task 无 status 返回」缺口） |
| §6 join 失败传播 | §21.9 / §87 #1「不隐藏错误」 | ✅ 哲学一致 |
| §6 `failure()` 取回即确认（唯一 ack） | §89 #1 try/catch + Result 必须 handle 同构 | ✅ |
| §6 detached ack/drop/exit-sweep 互斥 | §87 #4 detached 非作用域、cancel 可达 | ✅ 单一出口、不重复、不丢失（immortal handle 由 exit-sweep 兜底） |
| §6 RFC 0003 trial 捕获契约 | RFC 0003「唯一可观测 unwind 边界」 | ✅ 定义该边界 |
| §4.2 `Span` 全规范统一 | §69.1 / §24 / RFC 0003 三处 Span 表述 | ✅ 统一为单一 `Span` 形状（消除三 schema 冲突） |
| 零新关键字 | §9（56）；`is_failed`/`status`/`failure` 为 std.async 方法（§87 先例）、`TaskStatus`/`TaskFailure`/`DiagnosticCode` 为类型词 | ✅ 已核查 |
| 零新产生式 | §27 不动；§69.3 为编译器内部 struct、§21.8 为 std API + 生命周期图 | ✅ 已核查 |

---

## 9. 落地映射（合并进 AILANG.md 的位置）

| 本 RFC 内容 | 落地位置 | 性质 |
|---|---|---|
| §4.1 Diagnostic struct 扩展（Severity 5 级 / `code` / `related` / `suggestion` / `category`） | §69.3 内部 struct 扩展（Part IV 编译器实现层） | **资料性**（Part IV 实现 detail；§69.3 按 RFC 0005 §5 归 Part IV informative，不升格为合规义务来源） |
| §4.2 诊断发射协议（JSON Lines schema + `Span` 形状 + `suggestion` 应用语义） | 新增「诊断发射协议」小节（独立于 §69.3） | **规范性**（工具链合规契约：MUST 支持 `--diagnostics-format=json` + 字段表 + `Span` 统一） |
| §5 AILxxxx 编号空间 + 登记 | §18.2 展开 + 附录 E（错误码登记表） | 规范性（编号空间 + 段边界不变量）+ 资料性（登记表种子码） |
| §6 TaskHandle 错误态 | §21.8 生命周期图 + §21.9 并发错误表补注 + §87 #4 补注 | 规范性（Part I 并发） |
| §6 `TaskStatus` / `TaskFailure` 类型 | §34 std.core / §37 std.async 类型表 | 规范性（std API） |
| §7 交叉更新 | RFC 0005 §3 / §1.5.4 + RFC 0003（`panic_symbol`→`slug`）/ 0004 引用更新 | 同步 |

---

## 10. 开放问题

1. **`failure()` 取回即 ack 的精确语义**——§6.3 定为「调 `h.failure()` 得 `Some` = 确认、不传播」。备选：需显式 `h.ack_failure()` 分离查询与确认。推荐取回即确认（同 Result handle 同构、更简洁），留 review。
2. **join 传播 vs 父级 Result**——§6.3 scoped 子 Task `Failed` 默认 panic 传播至父。是否提供「父以 Result 接收子失败」的非 panic 路径（如 `try_join`）？v0.2 task 体 void + CSP 通道，倾向于经通道传失败消息而非新 join 算子；非 panic join 留 std.async 高阶 API（P1+）。
3. **`--diagnostics-format=sarif`**——§4 定 json 为 v0.3 必备；SARIF（OASIS 标准，CI/IDE 通用）作为未来格式留 review。
4. **Actor 死信可观测对称性**——§6.3 提 actor handler panic → 死信经 `ActorHandle` 查询 `TaskFailure`；其精确 API（与 TaskHandle 对称的方法集）留 std.async 设计。
5. **诊断码登记的版本化**——§5 slug 一经发布不变；number 跨 major 可重编。major 版本切分点（v1.0？）与重编策略留治理（RFC 0005 §9 lifecycle）。
6. **全局未处理失败处理器的默认行为与触发时机**——§6.3 detached `Failed` 默认「日志 + 继续」，触发时机定为 handle drop（未 ack）+ 进程退出 sweep（immortal handle）；是否默认 panic（更严格、更「不隐藏」）？推荐日志（detached 本意解耦、不应一个 detached 失败崩进程），留 review。

---

## 11. 收敛轨迹

**Draft v1 → pass-1（已完成）→ 修正后待 pass-2**。pass-1 对抗式 workflow（与 RFC 0009 并行）报 **0H / 9M / 3L = 12 条**（RFC 0008 部分），已逐条修正：① 5xxx(编译期)/7xxx(运行期) 边界澄清 + `requires`/`ensures` 运行期违约落 AIL7008/7009 + AIL7005/7006 拆分；② 新增 **AIL8xxx FFI/ABI/布局/codegen 段** + DiagnosticCategory `Ffi`（闭合 codegen/FFI 无家可归，RFC 0009 FFI 违规接入 8xxx 非 6xxx）；③ §4.2 补规范性 JSON 字段表 + `Span` 全规范统一 + `suggestion` 应用语义；④ §6.1 生命周期 Failed 由 Running/Waiting 任一转入；⑤ §6.2 `TaskStatus` 加 Created/Waiting 忠实投影；⑥ §6.3 detached ack/drop/exit-sweep 互斥（消除「永不报告」+「双重处理」漏洞）；⑦ §7 RFC 0003 `panic_symbol`→`slug` 映射 + §3 引用现状校正；⑧ §9 normative/informative 标注拆分（§69.3 struct 资料性 / 发射协议规范性）。待 pass-2 复验收敛 + regression。

> 本批是「可观测」主题，验证重心在**协议/契约的完备性与 soundness**（尤其 join 失败传播无静默吞、诊断 JSON schema 对 AI 稳定可消费），比 0006 的形式化、0007 的确定性更偏「契约正确性 + 不变量守住」。

---

*本 RFC 由综合判断 [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 P0-3 + [`deep-review-2026-07.md`](../research/deep-review-2026-07.md) 可观测 4.0 驱动。落地后将闭合「编译期诊断非结构化」「错误码未分类」「TaskHandle 错误态未定义」三个被前轮决议引用却悬空的缝隙，把可观测维度从「运行期近零」向「失败全程可观测、可机器消费」推进，并为 RFC 0003 / 0004 的机器闭环补上共同的诊断输入契约。*
