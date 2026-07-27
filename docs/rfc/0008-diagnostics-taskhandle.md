# RFC 0008 · P0-3 诊断协议 + 错误码分类法 + TaskHandle 错误态—— 结构化诊断 / AILxxxx 编号空间 / Task Failed 闭合

| | |
|---|---|
| **状态** | 草案（Draft v1）—— 待 review（已跑对抗式 workflow pass-1/pass-2/pass-3/pass-4/pass-5/pass-6/pass-7/pass-8，pass-6 报 4 confirmed [0H/2M/2L] 已全部修正；pass-7 报 6 confirmed [0H/6M/0L]（§6.3 exit-sweep「仅扫 detached」收窄回归 3 维度——引入「已出口」标记 + sweep 恢复 catch-all 根治 / handle-drop 补「未已出口」前置 / §7+§10 OQ#7 stale「P0-3」→P1+ / §4 ICE span→Optional）已全部修正；**pass-8 报 2 confirmed [0H/1M/1L]**（同根——§6.3 #3 exit-sweep 把 scoped 子 of 永不终结父（①）与 root/入口 fn main() 非 Task（②）误归 sweep sound 覆盖：① sweep 需进程退出而卡死父保活致进程可能永不退出、根本无法 sound 覆盖 → 诚实登记为残留 gap §10 OQ#8 + 缓解 `spawn detached`；② fn main() 非 Task/不在注册表/自身 panic→abort §17、移出 sweep 场景）已全部修正；**pass-9 报 2 confirmed [0H/0M/2L]**（① §8 自洽表 line 326 AIL7xxx↔§17 panic 命名「✅ 一一映射」与 §7 显式 1:N 声明矛盾〔pass-5 双射→满射精确化遗漏的相邻映射〕→ 改「✅ 全覆盖（4 具名符号 1:1→AIL7001-7004 + `AssertionFailure` 1:N 拆分→AIL7005/7006/7007 + 显式 panic→AIL7010 + requires/ensures→AIL7008/7009；详见 §7 映射表 + 三条非双射诚实声明）」/ ② §9 落地映射 line 353 误指「§34 line 1783 `TaskHandle` 方法表」〔冻结 §34 line 1783 实为散文列举、仅同节 `Optional` line 1721 / `Result` line 1735 有方法签名表〕→ 改「**新建** `TaskHandle` 方法签名表——同节 `Optional`/`Result` 方法表格式先例；codegen 同步更新 §77 line 3104」）已全部修正；**pass-10 报 2 confirmed [0H/0M/2L]**（① diagnostic-protocol-soundness：§4.1 Diagnostic/RelatedSpan 内部 Rust struct〔§9 line 347 明示「§69.3 内部 struct 扩展 Part IV 编译器实现层」、冻结 §69.3 line 2774-2779 全文 Rust `Option<T>` + `pub struct`/`String`/`Vec`〕同体混用 ailang 记法 `Optional<Span>`〔line 65 RelatedSpan.span、line 74 Diagnostic.span——pass-4 F1 + pass-7 F6 编辑漂移〕与 Rust `Option<Suggestion>`〔line 76〕——记法回归〔Rust 无 `Optional<T>`、ailang `Optional<T>` 不与 `pub struct` 同形〕→ §4.1 line 65/74 + §4.2 line 106/107 引用统一 `Optional<Span>`→`Option<Span>`、与冻结 §69.3 同构〔§6.2 TaskFailure ```ail 块 `Optional<FileSpan>` 不动——ailang 运行期类型、记法正确〕 / ② frozen-boundary：AIL7008/7009 落地映射 §9 line 350 列 §17 line 736 + §20 line 990 + §27 line 1480 三处冻结位置更新、遗漏冻结 §34 line 1756-1764「已知 panic 触发原因→符号→指针」表〔Part II normative core、与 §17 line 736 平行同 5 因；requires/ensures 为新 panic 因、合并后不同步将致两处 normative frozen 位置口径不一致〕→ §9 line 350 补 §34 表对称补注〔加 requires/ensures 行或声明非穷尽〕+ §7 line 311「§34 panic 符号表不变」收紧为「§34 既有因的符号分类立场不变」）已全部修正、待 pass-11 复验；**pass-11 报 0008 clean（0 confirmed）**〔pass-10 F1 Optional→Option §4.1/§4.2 + F2 §9 line350 §34 表补注 + §7 line311 收紧 两 fix 均 hold、无回归〕；**pass-12 报 2 confirmed [0H/1M/1L]**（① [L] taskhandle-failed-state-closure：§6.3 #1「首因」选择规则全文未定义〔多子并发 Failed 时父 panic 的 slug 随实现迭代序而异——违 RFC 0007 §2 line 34 显式登记 MUST 原则、削弱 §5.2 line 174 稳定 slug 契约〕→ §6.3 #1 line 259 显式定义「首因」= **spawn 序**〔源码静态出现序、可免费确定、确定性跨实现一致、服务稳定 slug 契约〕；② [M] join-failure-propagation：F4〔§6.3 #3 line 268〕「handler 从 runtime 上下文被调用」前提对 **#1 多子次级投递路径**失效〔次级投递于父 Task join 上下文同步、非 runtime 上下文 → handler 此路径 panic→父 Failed〔非 abort〕与 F4 矛盾 / double-panic / 截断静默丢失三后果〕→ 澄清**次级投递由 executor 在 runtime 上下文异步调度**〔非父 Task 栈同步、触发在 join / 执行在 executor 分离〕、F4 对**全部投递路径**一致成立、三后果一并消解；§6.3 #1 line 259 + #3 line 268 F4 普适性注 + 设计说明 line 274 三处同步）已全部修正、待 pass-13 复验；**pass-13 报 1 confirmed [0H/1M/0L]**（[M] join-failure-propagation：pass-12 把 §6.3 #1 多子次级投递由父 Task 栈同步改为 executor 异步调度以闭合 F4 普适性、却引入回归——root scoped 父 Task〔父为 `fn main()`〕多子同时 Failed 时、首因 panic 经 #1 传播至 `fn main()` → abort、抢占已 enqueue 给 executor 的次级投递、次级 TaskFailure 诊断信息〔slug/span〕可能永不被全局处理器观测、违反 §6.3 #1 line 259「次级失败不静默丢失」无限制断言；sweep〔需进程退出而 abort 非 graceful 不触发〕/ §10 OQ#8〔登记 scoped 子 of 永不终结父、非 root abort〕/ F4 普适性注〔论 handler 被调用后 panic 的目标、非 handler 未调用即被抢占〕/ changelog ③〔仅覆盖 handler panic 截断投递链、与 handler 是否 panic 无关〕均不覆盖此「首因传播抢占 executor 异步投递」路径）→ §6.3 #3 line 267 加「abort 前 drain 保证」normative 条款〔runtime 须在语言语义触发的 abort〔root scoped→fn main()→abort / fn main() 自身 panic→abort / 全局处理器 panic→abort F4 / extern 帧 panic→abort §17〕前 drain 已 enqueue 的全局处理器投递队列、先完成 pending 投递 + handler 标记次级「已出口」再终止进程；OS 强制终止 SIGKILL/SIGTERM/OOM/hardware fault 为进程外事件、非语言语义保证范围、同 OQ#8 诚实排除〕+ §6.3 #1 line 259 末句「不静默丢失」断言加 drain 保证 cross-ref 闭合 + 设计说明 line 274 同步 drain 保证注 三处一致；选 suggested_fix option (b)〔runtime 层真闭合、不破坏「不隐藏错误」哲学〕非 (a)〔不削弱断言、而是闭合之〕、保 F4 普适性〔handler 仍 runtime 上下文、panic 仍→abort、仅推迟到 drain 之后〕）已全部修正、已全部修正；**pass-14 报 1 confirmed [0H/1M/0L]**（[M] taskhandle-failed-state-closure：pass-12/13 把「首因」选择规则基准定为「spawn 序 = 源码静态出现序」、并把 `parallel for` 列入「多子 Task 同时 Failed」适用情形——但同源码静态 `spawn` 位点产生多个并发子 Task 时〔`parallel for` 的每次迭代、或循环 / 递归体内累计存活的 spawn 句柄〕所有子 Task 共享同一源码静态出现序、「取首个未 ack Failed 子 Task」于同位点集合无确定指称——tie-break 未定义、且「可免费确定」论证于 `parallel for` 失效〔其迭代调度依赖 §21 work-stealing〕、违 RFC 0007 §2 line 34 MUST；更甚冻结 §21.10 / §58 / §87 #9 把 `parallel for` 定义为表达式〔返回 `List<R>`、body 须 `R`、无 TaskHandle 暴露 / 无独立 Task 身份〕——列入多子 Failed 即隐含「迭代 = Task」模型却无依据）→ §6.3 #1 line 259〔从多子裁定适用情形移除 `parallel for`——明示其为非 spawn 表达式、body panic 经表达式单一出口一次性传播；+ spawn 序由「源码静态出现序」扩展为良定义全序：不同静态位点按源码静态出现序、同静态位点的多次 spawn 调用〔循环 / 递归体内 spawn〕按运行时 spawn 调用序〔父 Task 控制流顺序执行、不依赖 work-stealing〕——闭合同位点 tie、保「确定性 / 可免费确定」声称真成立、闭合 RFC 0007 §2 line 34 MUST〕已全部修正、待 pass-15 复验；目标收敛 0H/0M/0L，对齐 RFC 0001 v6 / 0002 v8 / 0003 v5 / 0004 v5 / 0005 v1 / 0006 v1 / 0007 v1）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94 语义决断、56 关键字、110 决议均不变；TaskHandle 新查询方法 `is_failed`/`status`/`failure` 为 `std.async` 方法非关键字——同 `cancelled`/`yield` 先例；`AILxxxx` 错误码为登记表标识符非关键字；零新关键字、零新文法产生式）|
| **日期** | 2026-07-27 |
| **分级** | **P0-3**（综合判断 [`docs/research/synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 第四优先级；[`deep-review-2026-07.md`](../research/deep-review-2026-07.md) 可观测维度 4.0「运行期近零」+ P1「Task 失败语义闭环（RFC 0003 已依赖）」；[RFC 0003](./0003-pbt-fuzzer.md) PBT 每 trial 独立 Task 捕获 panic 的契约前置）|
| **承接** | [`deep-review-2026-07.md`](../research/deep-review-2026-07.md) 可观测 4.0 + AI 友好 6.0「meaning 不可验证」；[`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §4 交叉印证 #11（结构化诊断未形式化）+ #6（运行期可观测缺失）；[RFC 0005](./0005-spec-governance.md) §3 Conformance「诊断信息**必须**（MUST）结构化（§69.3）」——本 RFC 给该引用补上**协议定义**；[RFC 0003](./0003-pbt-fuzzer.md) §「每 trial 独立 Task 捕获 panic = 唯一可观测 unwind 边界」——本 RFC 定义该边界的可观测契约；[ailang-ai-friendliness 存档](../research/ai-friendliness-2026-07.md) §9「结构化诊断一等公民」（5 个 AI-era 加法之一）|

---

## 1. 动机

deep-review 把 **可观测维度评为 4.0**——全场最弱项之一，判决「运行期近乎零」。三个具体空洞：

- **结构化诊断未形式化**——§69.3 的 `Diagnostic` 是 ail-ast 内部 Rust struct（3 级 severity、自由 `code: Option<&str>`），**无机器可读的发射协议**。AILang 的存在理由是「代码表达意图 → `.ailmeta` → AI 零猜测」，但编译器**吐给 AI 的诊断**却是非结构化的人读文本——AI 自修复（RFC 0003 fuzz self-repair、RFC 0004 结构化结果）恰恰依赖结构化诊断作输入。
- **错误码体系未建**——§18.2 仅一句「`AIL2xxx` = 所有权/内存类……统一错误码体系待后续设计阶段细化」。运行期 panic 用散落命名（`ArithmeticOverflow`/`DivideByZero`/`IndexOutOfBounds`/`ConstraintViolation`），**无稳定标识符、无分类法、无版本化**——AI 无法跨编译器版本稳定匹配诊断。
- **TaskHandle 错误态未闭合**——§17 line 736 与 §3119 都写「Task 内 panic → TaskHandle 错误态」，但 §21.8 生命周期图**只有 `Cancelled` 一个非 `Completed` 终态，无 `Failed`**；`TaskHandle` 仅有 `cancelled()` 查询，**无失败查询、无 join 失败传播规则**。这是 RFC 0003 PBT「每 trial 独立 Task 捕获 panic = 唯一可观测 unwind 边界」直接依赖却未定义的契约。

**三者为何同批**。它们是**同一主题「让失败可被观测」的编译期 / 标识 / 运行期三面**：诊断协议 = 编译期失败如何结构化吐出；错误码 = 失败如何稳定命名；TaskHandle 错误态 = 运行期 Task 失败如何可观测。合在一起把「可观测 4.0」从结构上抬起，并闭合 RFC 0003 的前置依赖。

**本 RFC 的性质**：纯**工具链 / 运行时可观测层**——零新关键字、零新文法产生式；`TaskHandle` 查询方法为 `std.async` 方法（同 `cancelled`/`yield` 先例，§87 已确认此类不计入 56）；`AILxxxx` 为登记表标识符。不修改 §17 **已列** panic 触发的语义、§87 #4 spawn/cancel、§89 #8 并发错误表任一冻结决议；为 §20 `requires`/`ensures` 运行期违约（冻结 §20 line 990 / §27 line 1480 仅标「运行期断言」、**未定失败行为**——与 §92 #2 显式写明 T(expr) 违约→panic ConstraintViolation 形成对比）显式定义为 panic 并分配 AIL7008/7009（与 §92 #2 T(expr) 违约→panic 同族）——属 **gap-fill 非 overturn**。仅**扩展填补**「错误态未定义」「诊断未协议化」「错误码未分类」三个被前轮决议引用却悬空的缝隙。

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
//   Hint = 指向可能的改进（非错误，如 unused）；Help = 非 error 的「如何修」prose 诊断（经 message 给出修法文字）
//   ⚠️ 机器可应用补丁独立由 `suggestion` 字段承载（§4.1 Suggestion），与 severity 正交——
//      任一 severity（含 Help）均可携带 suggestion，Help 不蕴含 suggestion、suggestion 有无不取决于 severity

pub enum DiagnosticCategory {
    Lex, Parse, Names, Type, Borrow, Effect, Visibility, Contract, Concurrent, Runtime, Ffi, Codegen, Internal
}
//   Runtime = 运行期 panic（§5 AIL7xxx）；Ffi = FFI / ABI / 布局违规（§5 AIL8xxx）；Codegen = lowering / 代码生成失败；Internal = 编译器内部错误 ICE（§5 AIL0xxx）——ICE 是编译器自身的元失败、非任一 pass 的「跨类」诊断，故独立成类（`Internal`↔AIL0xxx）
//   **无 `Other` 兜底变体**——每条诊断**必须**归入 13 类之一（若诊断跨类，归属主类别）；这维持 `category`→AILxxxx 段的**满射**（每段至少 1 个 category 归宿、覆盖 AIL0xxx–AIL8xxx 全 9 段；**非双射**——category→段为多对一，如 Lex/Parse→AIL1xxx、Effect/Contract→AIL5xxx、Ffi/Codegen→AIL8xxx）、闭合 §5「每个诊断携带稳定 DiagnosticCode」契约（AI 按 slug 匹配）。若实现遇到确无法归类的新型诊断，属诊断分类法缺陷、应扩类别而非兜底。

pub struct RelatedSpan { pub label: String, pub span: Option<Span> }      // 「定义于此」「前一次借用于此」；Option<Span> 承载 span-less 子注记（如「此为已知问题」「以 debug 模式编译」）——恢复冻结 §69.3 `notes: Vec<(String, Option<Span>)>` 的 span-less 子注记能力（`related` 取代 `notes`、不应丢失该能力）；`label` 保持必需

pub struct Suggestion { pub span: Span, pub replacement: String } // 机器可应用补丁（span → 替换文本）

pub struct Diagnostic {
    pub severity: Severity,                 // Error | Warning | Note | Hint | Help（原 3 级 → 5 级）
    pub code: DiagnosticCode,               // 原 Option<&str> → 稳定码（§5），杜绝自由串
    pub category: DiagnosticCategory,       // 新增：产诊断的 pass 或运行期段（AI 按类别聚类；运行期 panic → Runtime）
    pub message: String,                    // 人读文本（可跨版本改）
    pub span: Option<Span>,               // 主定位；Option<Span> 承载 span-less 诊断（ICE AIL0xxx / 编译器元失败无用户源码上下文，见 §4.2）
    pub related: Vec<RelatedSpan>,          // 原 notes → 结构化多定位（带 label + span）
    pub suggestion: Option<Suggestion>,     // 新增：机器可应用修复建议
}
```

**设计说明**：
- `code` 由自由 `Option<&str>` 收紧为 `DiagnosticCode`（§5）——杜绝每个 pass 各自发明 code 字符串。
- `related` 取代原 `notes: Vec<(String, Option<Span>)>`——结构化（label + span），让 AI 能精确定位「定义处」「冲突处」等多定位。
- `suggestion` 是**机器可应用**修复（span + 精确替换文本），区别于 `Help` 文本提示——AI 自修复可直接套用（对齐 RFC 0003 fuzz 反例→self-repair 的机器闭环）。
- `Hint` / `Help` 分离 + `suggestion` 正交：Hint 是「可改进」建议（如 unused）；Help 是非 error 的「如何修」prose 诊断（经 `message` 文字给出修法）；**机器可应用补丁**（`span` + 精确替换文本）**独立**由 `suggestion` 字段承载——与 severity **正交**（Error/Warning/Note/Hint/Help 任一均可携带 `suggestion`，`suggestion` 的有无不取决于 severity）。此正交性使「机器修复」有**唯一权威通道**（`suggestion`），severity 仅表达人读紧迫度。对齐 LSP 的分层：`Diagnostic.severity` 表达紧迫度（Error/Warning/Information/Hint）、独立的 `CodeAction`/fix 表达「修复」——二者解耦（LSP 无「Help」severity，本 RFC 的 Help 为其「prose 提示」层的命名，不与 `suggestion` 的机器补丁通道重叠）。

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
| `category` | enum string：`"lex"\|"parse"\|"names"\|"type"\|"borrow"\|"effect"\|"visibility"\|"contract"\|"concurrent"\|"runtime"\|"ffi"\|"codegen"\|"internal"` | 是 | §4.1 DiagnosticCategory 的 JSON 串形（小写）；`"runtime"` 对应 AIL7xxx 运行期 panic、`"internal"` 对应 AIL0xxx ICE（无 `"other"`——对齐 §4.1 无兜底变体，维持 category→AILxxxx 段满射、覆盖 AIL0xxx–AIL8xxx 全 9 段） |
| `message` | string | 是 | 人读文本（可跨版本演进，非契约） |
| `span` | `Span`（见下） | 否 | 主定位；**可缺省**承载 span-less 诊断（ICE AIL0xxx / 编译器元失败无用户源码上下文——对齐 §4.1 `Diagnostic.span: Option<Span>` 与 `RelatedSpan.span: Option<Span>` 同构；AI 消费方可凭 `span` 是否为 null 区分「真实源码定位」vs「编译器元失败无定位」）。有源码定位时五字段全必需、1-based 不变量不变 |
| `related` | array of `{label, span?}` | 否 | 结构化多定位；空时可省；`span` 可选（对齐 §4.1 `RelatedSpan.span: Option<Span>`，承载 span-less 子注记——恢复冻结 §69.3 `notes: Vec<(String, Option<Span>)>` 能力） |
| `suggestion` | object `{span, replacement}` | 否 | 机器可应用补丁；无建议时省略 |

**`Span` 规范形状**（本协议 JSON 的诊断定位形状——**非声称「全规范统一」**：§69.1 编译器内部 `Span = {start, end, line, col}`（无 `file`，文件由编译会话上下文持有）与 §24 `.ailmeta` `span = {file, line_start, col_start, line_end, col_end}`（schema 字段名不同、含 `file`、双端点）**各自保持既有形状不变**，本协议**不**改写二者；JSON `Span` 为诊断发射协议的**专用形状**，含 `file` 因诊断须跨文件定位。与既有形状的映射见下「形状映射」。）：

| 字段 | 类型 | 必需 | 语义 |
|---|---|---|---|
| `file` | string | 是 | 源文件路径（相对包根或绝对） |
| `line` | int（1-based） | 是 | 起始行 |
| `col` | int（1-based） | 是 | 起始列（**按 Unicode 标量值计字符数**，1-based；统一为字符列、对齐编辑器 / AI 可读性——非字节偏移、非「由实现文档化」以消除实现定义行为） |
| `start` | int | 是 | 文件字节偏移起点（0-based） |
| `end` | int | 是 | 文件字节偏移终点（exclusive） |

**形状映射**（JSON `Span` 与既有形状的关系，**不修改既有形状**）：
- **§69.1 内部 `Span {start, end, line, col}`**：编译器内部表示，无 `file`（文件经编译会话上下文持有）。发射 JSON 时，发射器从会话上下文补 `file`，取 `start`/`end`（文件字节偏移，§69.1 既有语义）+ `line`/`col`（本协议定义为 Unicode 标量值字符列）。**§69.1 struct 不变**（不加 `file`——`file` 由会话上下文提供，非进内部 Span）。
- **§24 `.ailmeta` `span {file, line_start, col_start, line_end, col_end}`**：`.ailmeta` schema 字段，双端点（`line_start`/`line_end`）、字段名与本协议 JSON `Span`（单 `line`/`col` 起点 + `start`/`end` 字节偏移）不同。二者信息可互转但**形状不同、各自保持**——本协议**不**改写 §24 schema（§24 字段名属冻结 `.ailmeta` schema，改须经 Authorized RFC）。诊断 JSON 的单点定位用本协议 `Span`；`.ailmeta` 的源码定位用 §24 `span`。
- **RFC 0003 PBT/fuzz 反例 `span`**：归 RFC 0003 §9.6 自身 schema 决定（现行 RFC 0003 §9.6 line 164/175 采用 §24 形 `{file, line_start, col_start, line_end, col_end}`），**不**经本协议 JSON `Span`——本 RFC 仅定义**编译器诊断发射**（`ailc --diagnostics-format=json`）的 JSON `Span` 形状，不扩展到 PBT runner 输出。若 RFC 0003 未来决定改用 JSON `Span` 形，由 RFC 0003 自行登记其 schema 变更（本 RFC §7 交叉更新表**不**代为登记）。

**`suggestion` 应用语义**（机器补丁）：消费方（AI / 编辑器）应用建议 = 将 `[span.start, span.end)` 区间替换为 `replacement` 字符串。`replacement` 为字面替换文本（非 diff / 非 AST 变换），应用后**应**重新编译校验（建议非权威修复，可能引入新诊断）。`suggestion` 的 `span` 必落在 `Diagnostic.span` 或其 `related[].span` 之一内（建议须指向被诊断的代码区域）。

- `code.slug` 为**跨版本稳定标识**（AI 工具按 slug 匹配，不按可能重编的 number）。
- `ailc --diagnostics-format=json` 退出码：有 Error 级诊断 → 非零；仅 Warning/Note/Hint/Help → 零（不阻断）。
- **发射流**：JSON Lines 输出到 **stderr**（对齐 rustc/clang 诊断惯例）；**stdout** 保留给 `--print` 类编译产物输出（如 `ailc --print=ast`）。human 格式（默认）与 json 格式**共享同一 stderr 流**——诊断与编译产物分流，使 `ailc --diagnostics-format=json 2>&1 | jq` 管道与 CI / 编辑器集成无歧义（闭合 RFC 0005 §3 #4「诊断必须结构化」MUST 条款对发射流的 normative 歧义）。
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
| `AIL5xxx` | **编译期**契约 / 约束 / 效果诊断 | `AIL5001 contract-requires-static-reject`（**预留码**——`requires` 静态可证伪编译期拒收，激活须待 [RFC 0002](./0002-contracts.md) Tier 2 decidable refinement；v0.2 `requires` 为纯运行期断言 §20 / §83 #3，运行期违约落 AIL7008，故 Tier 2 落地前不触发——占位登记、为 Tier 2 预留稳定 slug）、`AIL5002 constraint-bound-check`（编译期约束 bound / 类型检查——slug 刻意用 `bound-check` 以与运行期 AIL7004 `constraint-violation-runtime` 区分，消解 §34 `ConstraintViolation` 同名碰撞）、`AIL5003 effect-violation` |
| `AIL6xxx` | 并发 / Send-Sync / Task | `AIL6001 not-send`、`AIL6002 not-sync`、`AIL6003 spawn-bound-check` |
| `AIL7xxx` | **运行期 panic** | `AIL7001 arithmetic-overflow`、`AIL7002 divide-by-zero`、`AIL7003 index-out-of-bounds`、`AIL7004 constraint-violation-runtime`（约束 `T(expr)` 运行期断言违约）、`AIL7005 unwrap-on-none`（`Option::unwrap` on `None`）、`AIL7006 unwrap-on-err`（`Result::unwrap` on `Err`）、`AIL7007 assertion-failed`（`assert` / 断言宏）、`AIL7008 contract-requires-violation-runtime`（§20 `requires` 运行期违约）、`AIL7009 contract-ensures-violation-runtime`（§20 `ensures` 运行期违约）、`AIL7010 explicit-panic`（显式 `std.core.panic(msg)`，§17） |
| `AIL8xxx` | **FFI / ABI / 布局 / codegen** | `AIL8001 ffi-unsafe-type`（非 FFI-safe 类型入 `extern` 签名，[RFC 0009](./0009-ffi-abi-supply-chain.md) §4）、`AIL8002 abi-mismatch`（调用约定越封闭集合，0009 §5）、`AIL8003 layout-mismatch`（`layout` 组合非法，0009 §7.2）、`AIL8004 ffi-marshalling`（封送不可表示）、`AIL8005 codegen-lowering`（lowering / 代码生成失败） |

> **`AIL0xxx`（ICE）的 `DiagnosticCode` 归属**（闭合 §4.1 `Internal`↔AIL0xxx 与 §5.2 码条目的衔接）：`AIL0001 internal-compiler-error` 的 `DiagnosticCode` 条目 `category = Internal`（§4.1 第 13 变体）、`default_severity = Error`——ICE 是编译器自身的元失败（如 invariant 违背、unreachable 命中、跨 pass 协议破裂）、独立成类，归 `Internal`（非任一编译期 pass 的「跨类」诊断、不借 `Other` 兜底）。ICE 经 §69.3 `Diagnostic` 结构吐出（一切走 Diagnostic、§69.3 + RFC 0005 §3 #4 诊断 MUST），其 `code.category = "internal"`、`slug = internal-compiler-error` 跨版本稳定——闭合 §4.1「每诊断归入 13 类」对 ICE 段（AIL0xxx）的覆盖（满射不漏段）。

**段边界不变量**（消除 5xxx 与 7xxx 的口径重叠、闭合 codegen/FFI 无家可归）：
- **5xxx = 编译期**契约 / 约束 / 效果诊断（在静态分析 pass 拒收）；**7xxx = 运行期 panic**（程序运行时触发）。`requires` / `ensures`（§20）为**运行期**前后置断言——其运行期违约落 **AIL7008 / 7009**（7xxx），**不**入 5xxx；5xxx 的 `AIL5001`（`requires` 静态可证伪编译期拒收）为**预留码**——激活须待 [RFC 0002](./0002-contracts.md) Tier 2 decidable refinement 落地；**v0.2 `requires` 为纯运行期断言**（§20 / §83 #3 显式排除自动证明），运行期违约一律落 AIL7008，故 AIL5001 在 Tier 2 落地前不触发（占位登记、不废弃——为 Tier 2 预留稳定 slug）。
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

§17 已列的 panic 命名（`ArithmeticOverflow`/`DivideByZero`/`IndexOutOfBounds`/`ConstraintViolation`/`unwrap`/`assert`/显式 `std.core.panic(msg)`）**映射为 `AIL7xxx`**——运行期 panic 现在携带 `DiagnosticCode`（而非裸命名串）。panic payload（§6 `TaskFailure`）含该 code，使运行期失败有稳定标识、可被 AI / 监控稳定匹配——闭合「可观测 运行期近零」的核心空洞。**§17 已列 panic 触发（含 §34「无独立符号」的 `unwrap`/`assert`/`std.core.panic`）均有对应 AIL7xxx 码**（AIL7005–7007 / AIL7010）；此外 §20 `requires`/`ensures` 运行期违约（冻结 §20 line 990 / §27 line 1480 未定失败行为、本 RFC §1 显式 gap-fill 为 panic）亦有 AIL7008 / AIL7009。故 §6.2 `TaskFailure.code` 为非 Optional（`DiagnosticCode` 而非 `Optional<DiagnosticCode>`）：任一 `Failed` 态 Task 的 panic——无论源出 §17 已列触发还是 §20 契约违约——必有稳定 code。

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
   │          │          ├──► Cancelled          // 既有：协作式取消（§21.8，Waiting → Cancelled，仅在 await 点生效；CPU-bound task 须 task.yield() 让出以检查 cancelled()，§21.8 line 1212 prose）
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
    code: DiagnosticCode,        // panic 码（§5），如 AIL7001 arithmetic-overflow；非 Optional——§17 全部 panic 触发均有 AIL7xxx 码（含显式 panic AIL7010），故 Failed 必有稳定 code
    message: string,             // panic 人读消息
    span: Optional<FileSpan>,    // 运行期源码定位（若可恢复）；FileSpan 见下
}

// FileSpan（std.core 运行期定位类型，§34 落地）——{ file: string, line: uint32, col: uint32 }
//   非 §69.1 编译器内部 Rust Span（Part IV 资料性、ail 不可名、且无 file 字段），
//   亦非 §4.2 JSON 发射形状（序列化形状、非内存类型）。
//   TaskFailure.span: Optional<FileSpan> 使全局未处理任务失败处理器（§6.3 #3）能定位运行期 panic 的源文件——闭合 Fail-time 投递的可观测目标。
struct FileSpan {
    file: string,                // 源文件路径（相对包根或绝对）
    line: uint32,                // 行号（1-based）
    col: uint32,                 // 列号（1-based，按 Unicode 标量值计字符数，对齐 §4.2 JSON Span col 语义）
}

// DiagnosticCode（AILang 运行期可见投影，std.core、§34 落地）——TaskFailure.code 的字段类型
//   §5.2 的 DiagnosticCode 为编译器内部 Rust 注册 struct（`&'static str`、含 default_severity/category 编译期元数据）；
//   AILang 运行期层（TaskFailure.code、全局处理器 fn(TaskFailure)）只见其【稳定投影】{ number, slug }——
//   default_severity/category 为编译期元数据、不进运行期值。这使全局处理器能在 AILang 层析取 .code.slug（AI 匹配键、
//   与 §4.2 JSON 通道同源），闭合 TaskFailure 在 AILang 层的类型完备性（字段类型须在 AILang 类型集内定义、非直接引用编译器内部 Rust 类型）。
struct DiagnosticCode { number: string, slug: string }   // 运行期投影；编译期完整注册 struct 见 §5.2
```

> `TaskStatus` 枚举为 §21.8 生命周期的**忠实投影**（`Created`/`Running`/`Waiting` 为非终态、三个终态收尾）——与 §6.1 生命周期图一一对应，不留「blocked / unscheduled Task 无 `status()` 返回值」的缺口。`is_failed()` ≡ `status() == Failed` 的便捷查询。

> `is_failed` / `status` / `failure` 为 `std.async` 方法，同 `cancelled` / `yield` / `task.yield()` 先例（§87 已确认此类上下文方法不计入 56 全局关键字）。`TaskStatus` / `TaskFailure` 为 std 类型（同 `TaskHandle` 自身）。

### 6.3 join 失败传播规则（核心新决断）

延续 §21.9 / §87 #1「不隐藏错误」哲学，定**结构化并发失败传播**：

1. **scoped task（默认，§87 #4）**：作用域末隐式 join 时，对处于 `Failed` 的子 Task 逐一裁定出口（每条出口命中即标记**「已出口」**、其余出口据此跳过——单一非重复机制，详见 design 说明）：
   - **父已取回失败**（调过 `h.failure()` 得到 `Some`）→ 视为**已确认（ack）**、标记「已出口」，**不**向父传播（父已知情并处理）；
   - **父未取回**（未观测的 `Failed`）→ **panic 传播至父**（父以该 `TaskFailure` panic）、标记「已出口」——失败**绝不**静默丢失；
   - **多子 Task 同时 `Failed`**（§87 #4 作用域可含多个 scoped 子 Task、经多个独立 `spawn` 语句〔含循环 / 递归体内同一静态 `spawn` 位点的多次调用〕并发，panic 为一次性 unwind、父仅 panic 一次）→ 父以**首因** `TaskFailure` panic（标记「已出口」）。**`parallel for` 非 `spawn`、不落入本条多子裁定**（冻结 §21.10 / §58 / §87 #9：`parallel for` 为表达式、返回 `List<R>`、body 须 `R`、**无 TaskHandle 暴露 / 无独立 Task 身份**），其 body `panic` 经**表达式单一出口**传播（一次性 panic、非多子 `Failed`）。**「首因」选择规则**：按作用域内 **spawn 序**取**首个未 ack `Failed` 子 Task**——**spawn 序为良定义全序**：**不同源码静态 `spawn` 位点按源码静态出现序**；**同静态位点的多次 `spawn` 调用（循环 / 递归体内 `spawn`）按运行时 `spawn` 调用序**（父 Task 控制流顺序执行 `spawn` 调用——调用本身顺序发生、**不依赖 §21 M:N work-stealing 对子 Task 执行的调度**）。**确定性、跨实现 / 跨运行一致**（spawn 序〔静态出现序 + 同位点运行时调用序〕均不依赖 §21 work-stealing 执行序非确定性、故**可免费确定**；对齐 RFC 0007 §2 line 34「不能免费确定的须显式登记」原则——此处既可免费确定即定义为 spawn 序、而非留 unspecified；服务 §5.2 line 174 稳定 slug 契约——父 panic 的 `DiagnosticCode`/slug 不随实现执行序而异）。**其余未 ack `Failed` 子 Task** 在 join 退出时**触发、由 executor 在 runtime 上下文投递**（**非父 Task 栈上同步执行**——与 detached Fail-time〔#3 line 264〕/ handle-drop〔line 263〕/ exit-sweep〔line 265〕投递**同上下文**，使全局处理器 panic 传播目标条款〔#3 line 268 F4〕对全部投递路径一致成立、无 join-上下文例外）、各自标记「已出口」——次级失败不静默丢失、亦不与首因重复（**「不静默丢失」于 abort 路径由 #3 line 267「abort 前 drain 保证」闭合**：次级投递虽由 executor 异步调度，但父作用域经 #1 传播链触发 abort 前 runtime 须 drain 投递队列，避免 abort 抢占 executor 致次级失败静默丢失；OS 强制终止除外、同 §10 OQ#8 诚实排除）。
2. **显式 `h.join()`（§77 line 3104 codegen 表列 `join` 为 `TaskHandle` 方法 / §87 #4 spawn 返 TaskHandle + 结构化作用域——显式 `h.join()` 的精确语义为本 RFC gap-fill，§87 #4 仅定作用域末隐式 join；与隐式 join 并存）**：`h.join()` **阻塞调用方至目标 Task 到达终态**（`Completed` / `Cancelled` / `Failed`），**不**传播 panic——panic 传播**仅**属上述 #1 隐式作用域末 join 对未 ack `Failed` 的行为；显式 `join` 仅同步等待、不引入二次 panic。失败观测统一经 join 后 `h.failure()` 取回 ack。**PBT runner 规范用法序列**（闭合 RFC 0003 §4.2 步 2-3「先 join 后观测」）：`h.join()`（阻塞至终态）→ `h.is_failed()`（查询 `Failed`）→ `h.failure()`（取回 `Some<TaskFailure>` = ack）→ 作用域末隐式 join 见已 ack → 不传播。此序列为 v0.3.0 规范定义的非传播失败观测路径（不依赖 §10 OQ #2 的 P1+ `try_join` 等高阶 API），使 PBT trial 捕获契约可落地。
3. **`spawn detached`（显式分离，§87 #4）**：无作用域父可传播 → 失败仅经 `h.failure()` 显式查询可观测。**未被 ack 的 detached `Failed` 的投递时机与互斥**（闭合「detached 失败可能永不报告」+「ack 后又投递全局处理器」+「handle 早于失败 drop → 孤儿失败」+「immortal handle 永不投递」四重漏洞）：
   - **ack 互斥（仅 scoped 有效）**：`h.failure()` 返回 `Some` 即 ack 点——已 ack 的失败**不得**再投递全局处理器（`failure()` 取回与全局处理器投递**互斥**，杜绝同一失败被双重处理）。**scoped/detached 不对称**（消除「可经 `failure()` 抑制 detached 全局投递」误读）：对 **scoped** TaskHandle，作用域末隐式 join 之前存在 ack 窗口——scope 内调 `h.failure()` 取回 `Some` 即 ack、抑制向父传播；对 **detached** TaskHandle，Fail-time 投递锚定在「Task 转入 `Failed` 瞬间且当时未 ack」，而 `failure()` 仅在 `Failed` 后返回 `Some`（§6.2）——故 detached 的 Fail-time 必先于任何用户 `failure()` 调用而触发，**detached 的 `failure()` 永远是 informational 查询、非 ack**（detached 失败的 canonical 出口是全局处理器，不可经 `failure()` 抑制其投递）。ack 机制的实际作用域 = scoped only。
   - **handle drop 触发（失败已发生）**：detached `TaskHandle` 被 drop **且 Task 已处 `Failed`** 而 `failure()` 从未返回 `Some`（未 ack）**且该 `TaskFailure` 未标记「已出口」**时，runtime 在 drop 点把该 `TaskFailure` 投递全局处理器 `std.async.on_unhandled_task_failure: fn(TaskFailure) -> void`、标记「已出口」——失败**不**因 handle 被丢弃而丢失、亦不与 Fail-time 投递重复（Fail-time 在转入 `Failed` 瞬间即触发、时间上必先于任何后续 handle drop，故 drop 触发实为 Fail-time 遗漏时的二次兜底）。
   - **Fail-time 投递（失败晚于 handle drop 或 immortal handle）**：runtime 同时持有 Task 注册表（知 Task 身份 / 当前态）与 handle 存活状态——当 detached Task **转入 `Failed` 的瞬间**，若其 handle 已 drop（孤儿：失败晚于 drop）**或** handle 虽存活但从未被外部 `failure()` 取回过（immortal handle 未观测），runtime **立即**把该 `TaskFailure` 投递全局处理器，**不**推迟到进程退出。此重锚消除「孤儿 detached + 永不退出进程」与「immortal handle 未 ack」两种静默丢失——失败的可观测出口锚定于 **Task Fail 事件本身**，不取决于调用方是否保留 handle、亦不取决于进程是否退出。投递后该 `TaskFailure` 标记为「已出口」；若 handle 后续 `failure()` 查询（ack 窗口已过），返回 `Some` 但**不再抑制**已发生的全局投递（双重处理风险由 ack 互斥条款闭合——一旦 Fail-time 已投递，后续 `failure()` 视为 informational 查询、非 ack）。
   - **进程退出 sweep（最终 best-effort 兜底）**：上述 Fail-time 投递已使 **detached** 失败的可观测出口锚定于 **Task Fail 事件本身**（不依赖进程退出）；进程退出 sweep 退为**最终 best-effort 兜底**，其 **sound 作用仅一项**——**runtime 实现层防御性兜底**：若 Fail-time 投递 / handle drop 投递 / #1 向父传播 / #1 多子次级投递因 runtime 实现缺陷未能命中某 `Failed` Task，sweep 在进程退出时补投该残留件（**正常路径不依赖、仅兜实现 bug**）。sweep 遍历注册表中**所有未 ack 且未标记「已出口」的 `Failed` Task**（**不限定 detached**——含 scoped 子），逐一投递全局处理器、各自标记「已出口」。**「已出口」标记为单一非重复机制**：#1 向父传播、Fail-time 投递、handle drop 投递、ack 取回均设置该标记，sweep 据此跳过已出口者——故 sweep 扫全部 `Failed` **不会**与既有出口双重观测。pass-6 曾以「避免双重观测」把 sweep 收窄至「仅扫 detached」，但双重观测风险已由「已出口」标记根治——收窄属过度防御（runtime 缺陷可发生于任何 Task、不限 detached），故 sweep 恢复 catch-all（扫全部未出口 `Failed`）。**sweep 为 best-effort、仅在进程退出时触发——不保证进程必退出、亦不保证所有未出口 `Failed` 必被观测**：进程是否终止取决于全局处理器实现（默认继续）与是否存在长驻 detached Task（见下「残留 gap」——scoped 子 of 永不终结父为 sweep **无法 sound 覆盖**的边界）。
   - **残留 gap（诚实登记，§10 开放问题 #8）**：scoped 子 Task 的 `Failed` 在其父作用域**永不终结**时**无 sound 可观测出口**——#1 隐式 join 永不触发（父卡死、不到达作用域末）、Fail-time 投递仅 detached（scoped 子不在其列、见上）、ack 需父调 `h.failure()` 而父卡死无法调、sweep 需进程退出而该卡死父（若为 detached 长驻）保持进程存活致进程可能永不退出。此为**结构化并发的固有边界**：scoped 模型要求父到达 join，父自身 stuck（如卡于 CPU-bound 无限循环——§21.8 协作式取消仅 await 点生效、不可强制终结）属编程错误，runtime 无机制在不破坏 scoped 契约（父负责在 join 处理子失败）的前提下强制观测——把 Fail-time 投递对称延伸至 scoped 子会消除父的 ack 窗口、使 scoped 退化为 detached 语义。**缓解**：长驻服务 / supervisor 需保证失败必可观测时应 `spawn detached`（享 Fail-time 投递、不依赖父 join）而非 scoped 子、或确保父作用域可达 join；sweep 在进程最终退出时 best-effort 补投此类残留件（**若**进程退出）。该 gap 的彻底闭合（scoped Fail-time 投递 / 可取消 CPU-bound task）留 §10 开放问题 #8 待运行期错误模型 RFC（P1+）。pass-7 曾把 scoped 子 of 永不终结父误归 sweep 兜底——本条诚实改为残留 gap。
   - **root / 入口（非 sweep 场景）**：`fn main()` 为同步函数、**非 Task**（§17 / §21.2）、**不在 Task 注册表**——其自身 panic 无 Task 边界吸收 ⟹ 进程 **abort**（§17 同族、定义行为、非 UB）；root 级 scoped Task（父为 `fn main()`）的 #1 传播至 `fn main()` ⟹ 亦 abort（`fn main()` 无吸收边界）。**abort 前 drain 保证**（闭合 §6.3 #1 line 259「次级失败不静默丢失」于 abort 路径——pass-12 把多子次级投递由父 Task 栈同步改为 executor 异步调度后引入的竞态窗口）：若父作用域经 #1 传播链最终触发进程 abort（本条 root scoped 父→`fn main()`→abort / `fn main()` 自身 panic→abort / 全局处理器 panic→abort〔F4 line 268〕/ `extern` 帧 panic→abort〔§17〕——均属「语言语义触发的 abort」），已 enqueue 给 executor 异步投递的次级 `TaskFailure` 可能被 abort 抢占、永不被全局处理器观测（worker 线程随 abort 死亡、无 atexit、exit-sweep 不触发——abort 非 graceful 退出）——违反「次级失败不静默丢失」声称。**runtime 须在触发上述 abort 前 drain 已 enqueue 的全局处理器投递队列**（先完成 pending 投递、次级失败达全局处理器、handler 标记「已出口」、再终止进程）——既保 F4 普适性〔handler 仍从 runtime 上下文被调用、panic 仍→abort、仅 abort 时机推迟到 drain 之后〕又保次级必达、不破坏「不隐藏错误」哲学。**OS 强制终止（SIGKILL / SIGTERM / OOM / hardware fault）为进程外事件、runtime 无控制权、非语言语义保证范围**（同 §10 OQ#8 scoped 子 of 永不终结父的诚实排除风格——语言语义仅保证其能控制的 abort 路径）。二者均**非** sweep 扫描对象（不构成存活 `Failed` Task）——pass-7 曾把「root/入口 Failed」误归 sweep 兜底，本条澄清为 abort 路径、移出 sweep 场景。
   - 全局处理器默认实现：记录到诊断日志 + 继续；可由用户覆写为 panic / 上报 / 忽略。**全局处理器 panic 的传播目标**（闭合与 §17 panic 展开目标穷举的张力）：全局处理器从 **runtime（executor）上下文**被调用——此时失败 Task 已 unwind 至 Task 边界、进入 `Failed`，handler 执行**不在任何 Task 体内**。§17（line 736 / §3119）把 panic 展开目标穷举为「最近的 Task 边界或 `extern` 帧」，runtime 上下文**既非 Task 边界亦非 `extern` 帧**——故全局处理器自身 panic **无 Task 边界可吸收** ⇒ **转进程 abort**（与 §17 `extern` 帧 panic→abort 同族：runtime 上下文无吸收边界 ⇒ abort，**非 UB**）；§17「v0.2 无 `catch_unwind`」亦禁止 runtime 经 catch_unwind 吞掉。即：用户把全局处理器覆写为 panic = 把「日志 + 继续」退化为「abort 整个进程」（退出进程是 sound 的、明确定义的，非未指定）。**F4 前提的普适性（消除「handler 于 join 上下文同步调用致 F4 失效」的解读歧义）**：上述「handler 从 runtime（executor）上下文被调用、不在任何 Task 体内」对全局处理器的**全部投递路径**一致成立——detached Fail-time〔line 264〕、handle-drop〔line 263〕、exit-sweep〔line 265〕、**以及 §6.3 #1 多子 Task 次级投递**〔line 259：次级投递虽于父 join 退出时**触发**，但其**执行**由 executor 在 runtime 上下文异步调度、**非**于父 Task 栈帧上同步执行——即触发时机在 join、执行上下文在 executor、二者分离〕；故 #1 多子次级路径上 handler panic 同样无 Task 边界可吸收 ⇒ 转进程 abort，F4 结论对该路径一致成立、**无 join-上下文例外**。
4. **actor 死信（§17 / §21.7）**：actor `on` handler panic → actor 进入死信态（既有），其失败信息同样以 `TaskFailure` 形式可经 `ActorHandle` 查询（与 TaskHandle 对称，细节留开放问题 #4）。

**设计说明**：
- 「取回即确认」把 `failure()` 定为**唯一消费式 ack**——对齐 §87 #1 / §21.9「不隐藏错误」哲学（错误须有可观测出口、不得静默吞）：你看了就是你的（ack、不传播），没看就崩你（scoped 传播至父）或经 drop / Fail-time / exit-sweep 进全局处理器（detached）。简单、sound、AI 可预测。**不援引「Result 必须 handle」**——冻结规范无此规则（`Result<T,E>` 经 `throw`/`try`/`match` 消费、无强制 handle 义务，§17 / §89 #6）；本规则的「失败必可观测」义务源于 §87 #1 并发不隐藏错误，非 Result 消费规则。
- **显式 vs 隐式 join 的语义二分**：显式 `h.join()` 仅同步等待、不传播（让调用方主动观测失败）；隐式作用域末 join 是「不隐藏错误」的兜底 panic 传播点（对未 ack 失败）。二者并存而非冲突——显式 join + `failure()` ack 是「主动处理」路径，隐式 join 传播是「被动兜底」路径，调用方选其一即可避免二次 panic。PBT runner / 显式错误处理走前者；默认 scoped task 走后者。
- **「已出口」标记为单一非重复机制**：一个 `TaskFailure` 的出口分四类——**ack**（`failure()` 取回）/ **向父传播**（scoped #1，含多子 Task 首因）/ **全局处理器投递**（detached 的 drop·Fail-time + scoped 多子 Task 次级失败经 #1 多子条款在 join 退出触发、由 executor 在 runtime 上下文投递——非父 Task 栈同步）/ **sweep best-effort 兜底**（全部未出口 `Failed`、runtime 实现层防御性补投；scoped 子 of 永不终结父为残留 gap 见 §6.3 #3 + §10 OQ #8、root/入口走 abort 非 sweep）。**四类出口互斥、单一命中、不重复、不丢失**——每条出口命中即标记「已出口」、其余出口据此跳过。**abort 前 drain 保证**（§6.3 #3 line 267）：全局处理器投递类（含 scoped 多子 Task 次级失败经 #1 多子条款的 executor 异步投递）虽于 handler 被调用时标记「已出口」，但若父作用域经 #1 传播链最终触发进程 abort，runtime 须在 abort 前 drain 已 enqueue 的投递队列——避免 abort 抢占 executor 异步投递致次级失败静默丢失（pass-12 sync→async 引入的竞态窗口、由 drain 保证闭合）；OS 强制终止（SIGKILL 等）非语言语义保证范围、同 §10 OQ#8 诚实排除。**exit-sweep 扫描全部未出口 `Failed`（不限定 detached）**：这是 **best-effort catch-all**——其 sound 作用仅一项：**runtime 实现层防御性兜底**（Fail-time / handle drop / #1 传播 / #1 多子次级投递若因 runtime 缺陷漏投，sweep 在进程退出时补投）。**已由前述出口 sound 覆盖、不靠 sweep**：scoped 子若父作用域正常终结经 #1 向父传播（标记已出口、sweep 跳过）；多子 Task 次级失败经 #1 多子条款在 join 退出时触发、由 executor 在 runtime 上下文投递全局处理器（标记已出口、sweep 跳过）；root/入口——`fn main()` 非 Task（§17 / §21.2）、不在注册表、自身 panic ⟹ abort（§17）、非 sweep 场景。**sweep 无法 sound 覆盖的残留 gap**：scoped 子 of 永不终结父（父卡死、#1 永不触发、Fail-time 仅 detached、ack 需父调用而父卡死、sweep 需进程退出而卡死父保活致进程可能永不退出）——结构化并发固有边界、见 §6.3 #3 残留 gap 条款 + §10 OQ #8、缓解 = `spawn detached`（sweep 在进程退出时 best-effort 补投、进程可能永不退出）。pass-6 曾把 sweep 收窄至「仅 detached」以避免双重观测，但双重观测风险已由「已出口」标记根治——故 sweep 恢复为 catch-all（扫全部未出口 `Failed`）；pass-7 曾把 scoped 子 of 永不终结父 + root/入口误归 sweep sound 覆盖——pass-8 诚实改判：前者为残留 gap（§10 OQ #8）、后者走 abort 路径。**Fail-time 投递与 ack 互斥的细节**：Fail-time 投递发生在「转入 `Failed` 瞬间且当时未 ack」时，一旦投递即标记「已出口」、后续 `failure()` 查询不构成抑制投递的 ack（避免「先投递再查询」造成双重处理）。
- 默认传播保证「不隐藏错误」——一个 panic 的子 Task 不会被静默吞掉，要么父显式处理，要么父跟着 panic（scoped）/ 进全局处理器（detached，且 Fail-time 投递使长驻进程亦能及时报告）。
- detached 的全局处理器是 fire-and-forget-adjacent 模式的逃生舱（日志 / 上报），但不静默——有可观测出口；Fail-time 投递使「immortal handle 未 ack」与「永不退出进程」亦能及时报告（不再依赖 exit-sweep 兜底）。

> 此规则闭合 RFC 0003 PBT「每 trial 独立 Task 捕获 panic = 唯一可观测 unwind 边界」——该边界现为**规范契约**：trial Task 的 panic 确定地成为 `TaskFailure`，runner 经显式 `h.join()` 阻塞至终态后 `h.failure()` 可靠取回 ack（不传播、确定性、可复现），支撑 PBT shrinking / fuzz 反例的稳定捕获（§6.3 #2 序列）。

---

## 7. 与 RFC 0005 / RFC 0003 的交叉更新

本 RFC 落地后，既有 RFC 的引用点更新（合并时同步）：

| 既有引用 | 现状 | 本 RFC 后 |
|---|---|---|
| RFC 0005 §3 Conformance「诊断信息**必须**（MUST）结构化」 | RFC 0005 §3 #4 **已**指向本 RFC §4 的 JSON Lines 协议（并显式标注 §69.3 仅为 Part IV 资料性参考）——合并时此引用生效 | 合并后 RFC 0005 §3 #4 的引用与本 RFC §4 自洽（§69.3 不升格为合规义务来源） |
| RFC 0005 §1.5.4 行为分类「panic 是定义行为」 | panic 定义但失败可观测性未提 | 补：panic 经 TaskHandle `Failed` + `TaskFailure` 可观测（§6），运行期失败有稳定 `DiagnosticCode`（§5） |
| RFC 0005 §6「资源耗尽（栈溢出/OOM）+ double-panic」终止语义（RFC 0005 §6 标注「留后续运行期错误模型 RFC P1+ 收紧」） | RFC 0005 §6 把二者归未指定行为、终止语义（abort / 进程终止模型）**显式留给后续运行期错误模型 RFC（P1+，待立项）** | 本 RFC（= P0-3）的主题为诊断协议 / 错误码 / TaskHandle 错误态，**不**覆盖进程级资源耗尽终止语义——**显式声明 out-of-scope**，登记 §10 开放问题 #7 待后续运行期错误模型 RFC（P1+）收紧；本 RFC 维持 RFC 0005 §6 现状（二者归未指定行为、非 UB、不开 §17 外新例外） |
| RFC 0003 PBT「每 trial 独立 Task 捕获 panic」 | 依赖未定义的「Task 错误态」 | 现为规范契约（§6.3 join 传播 + `failure()` 取回），trial 捕获确定可靠 |
| RFC 0003 `panic_symbol`（PascalCase）vs 本 RFC `code.slug`（kebab-case） | 两套标识符命名风格、映射**非**简单 PascalCase 别名（见下「`panic_symbol` ↔ `code.slug` 映射」） | 映射含 1:1（具名符号）与 1:N（`AssertionFailure` → 三 slug）两类、并消解 `ConstraintViolation` 同名碰撞；合并时 RFC 0003 统一改引 `slug`（`panic_symbol` 作人读聚合名保留或移除，留 RFC 0003 review）。详见下「`panic_symbol` ↔ `code.slug` 映射」表 + 三条非双射诚实声明 |
| RFC 0004 结构化结果六态 | `failed` 反例为人读 | 可引用 `DiagnosticCode` slug 作机器稳定键（与本 RFC §5 slug 体系一致） |

**`panic_symbol` ↔ `code.slug` 映射**（合并时 RFC 0003 统一改引 `slug`；映射**非**全局 1:1，分两类）：

| §17 / §34 panic 来源 | panic_symbol（RFC 0003 PascalCase，人读） | AIL7xxx code.slug（本 RFC，机器稳定键） | 备注 |
|---|---|---|---|
| 整数溢出（debug） | `ArithmeticOverflow` | AIL7001 `arithmetic-overflow` | 1:1（§34 具名符号） |
| 除零 / 取模零 | `DivideByZero` | AIL7002 `divide-by-zero` | 1:1（具名符号） |
| 越界访问 | `IndexOutOfBounds` | AIL7003 `index-out-of-bounds` | 1:1（具名符号） |
| 约束违约 `T(expr)`（运行期） | `ConstraintViolation` | AIL7004 `constraint-violation-runtime` | 1:1（§34 具名符号；slug 加 `-runtime` 后缀**与编译期 AIL5002 区分**——见下） |
| `unwrap` on `None` | （`AssertionFailure`） | AIL7005 `unwrap-on-none` | **1:N**——RFC 0003 单一 `AssertionFailure` 覆盖 unwrap/assert 三态，本 RFC 拆为 3 个 slug（一码一触发） |
| `unwrap` on `Err` | （`AssertionFailure`） | AIL7006 `unwrap-on-err` | 同上（1:N 第二支） |
| `assert` 失败 | （`AssertionFailure`） | AIL7007 `assertion-failed` | 同上（1:N 第三支） |
| 显式 `std.core.panic(msg)` | （无 panic_symbol——§34「无独立符号」） | AIL7010 `explicit-panic` | 显式 panic 在 §34/§17 无具名符号，本 RFC 补**诊断码**（非补 panic 符号） |

**非双射的诚实声明**：
- **1:N（AssertionFailure）**：RFC 0003 的单一 `AssertionFailure` panic_symbol 对应本 RFC 三个 slug（AIL7005/7006/7007）——这是**有意的细化**（一码一触发，AI 可精确区分 unwrap-None / unwrap-Err / assert）。合并时 RFC 0003 的 `AssertionFailure` 保留作人读聚合名，机器匹配改引具体 slug。
- **命名碰撞消解（ConstraintViolation）**：§34 运行期 `ConstraintViolation` 与本 RFC 编译期 `AIL5002` 若同名 `constraint-violation` 会碰撞。**消解**：AIL5002 slug 改为 `constraint-bound-check`（编译期约束 bound / 类型检查），AIL7004 保留 `constraint-violation-runtime`（运行期 `T(expr)` 断言违约）——二者 slug 不再碰撞（§5.1 已改）。
- **「无独立符号」≠「无诊断码」**：§34「`unwrap`/`assert` 失败 = 直接 panic、无独立符号」指 panic **展开期**不携带具名 unwind 符号（§89 #3）；本 RFC 的 AIL7xxx 是**诊断码**（稳定标识符，供 AI / 监控匹配），与 panic 展开符号是**不同命名空间**。故为 unwrap/assert 补 AIL7005/7006/7007（及显式 panic 补 AIL7010）**不**修订 §34「无独立符号」——§34 **既有因的符号分类立场**不变（unwrap/assert/explicit-panic 仍无独立符号；requires/ensures 运行期违约为 §34 line 1756-1764 表中无行的新因、由 §9 落地映射对该表对称补注处理），仅诊断码空间新增条目。

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
| §5 `AIL7xxx` panic 段 | §17 panic 命名（ArithmeticOverflow 等） | ✅ 全覆盖（映射**非**全局 1:1：4 具名符号 1:1→AIL7001-7004 + `AssertionFailure` 1:N 拆分→AIL7005/7006/7007 + 显式 `panic` 无符号补码→AIL7010 + requires/ensures gap-fill→AIL7008/7009；详见 §7 映射表 + 三条非双射诚实声明） |
| §5 `AIL8xxx` FFI/ABI/布局/codegen 段 | §25.4 `layout(C)` / [RFC 0009](./0009-ffi-abi-supply-chain.md) §4 FFI-safe / §5 ABI 封闭集 | ✅ 闭合 codegen/FFI 诊断无家可归（DiagnosticCategory `Ffi`/`Codegen` 各有 8xxx 码） |
| §5 用户 error 不在 AILxxxx | §17 `error` 枚举 + §89 #10 `errors[]` 单一真源 | ✅ 命名空间分离 |
| §6 `Failed` 终态（Running/Waiting 均可转入） | §17 panic 展开至 Task 边界（不传染进程） | ✅ 一致（边界吸收为 Failed） |
| §6 `TaskStatus` 忠实投影（Created/Running/Waiting + 三终态） | §21.8 生命周期全集 | ✅ 一一对应（无「blocked Task 无 status 返回」缺口） |
| §6 scoped 隐式 join 传播 + 显式 `h.join()` 不传播 | §21.9 / §87 #1「不隐藏错误」 + §77 line 3104 codegen 表列 `join` 为 `TaskHandle` 方法 / §87 #4 spawn+作用域（显式 join 语义本 RFC gap-fill） | ✅ 哲学一致；显式 join（同步等待、不传播）与隐式 join（未 ack 兜底传播）二分共存；PBT runner 经 `join → is_failed → failure` 序列实现非传播观测（v0.3 落地、不依赖 OQ #2 P1+ `try_join`） |
| §6 `failure()` 取回即确认（唯一 ack） | §87 #1 / §21.9「不隐藏错误」（失败必有可观测出口） | ✅ 哲学一致（**不**援引不存在的「Result 必须 handle」规则——§89 #1 为 try/catch 文法、无 Result 强制 handle；本义务源于并发不隐藏错误） |
| §6 detached ack/drop/**Fail-time**/exit-sweep 互斥（Task 级、Fail-time 投递锚定 Task Fail 事件） | §87 #4 detached 非作用域、cancel 可达 | ✅ 单一出口、不重复、不丢失；Fail-time 投递在「Task 转入 `Failed` 瞬间且未 ack」时立即触发（覆盖孤儿 drop + immortal handle + 永不退出进程），exit-sweep 退为兜底——「永不报告」空洞在长驻服务进程下亦闭合 |
| §6 RFC 0003 trial 捕获契约（显式 `h.join()` + `failure()` ack） | RFC 0003 §4.2 步 2-3「先 join 后观测」 + 「唯一可观测 unwind 边界」 | ✅ 定义该边界并提供非传播观测序列（§6.3 #2） |
| §4.2 JSON `Span` 为**编译器诊断发射协议**专用形状（非声称全规范统一、非声称覆盖 PBT runner 输出） | §69.1 内部 `Span`（无 file）/ §24 `.ailmeta` span（字段名不同）/ RFC 0003 §9.6 PBT 反例 span（§24 形、归 RFC 0003 自身 schema） | ✅ 既有形状各自不变；本协议 JSON `Span` 含 `file`、`col` 定义为 Unicode 字符列；映射关系显式给出（不覆写 §69.1/§24 冻结形状、不引入实现定义行为）；**PBT 反例 span 形状不**经本协议——本 RFC 仅定义编译器诊断 JSON `Span` |
| §4.1 DiagnosticCategory 含 `Runtime` 变体 | §5 AIL7xxx 运行期 panic 段 | ✅ 运行期 panic 有忠实 category（`category:"runtime"`）；Ffi/Codegen 对应 8xxx，三段各有归宿 |
| §4.1 DiagnosticCategory 13 变体（含 `Internal`↔AIL0xxx ICE）无 `Other` 兜底 | §5.1 9 段（AIL0xxx–AIL8xxx，category→段多对一、`Internal`→AIL0xxx） | ✅ `category`→AILxxxx 段**满射**完备（覆盖 AIL0xxx–AIL8xxx 全 9 段、无孤儿 category、无死区；**非双射**——多对一如 Lex/Parse→AIL1xxx、Effect/Contract→AIL5xxx、Ffi/Codegen→AIL8xxx）；无 Other 兜底；新型诊断应扩类别而非兜底 |
| 零新关键字 | §9（56）；`is_failed`/`status`/`failure` 为 std.async 方法（§87 先例）、`TaskStatus`/`TaskFailure`/`DiagnosticCode` 为类型词 | ✅ 已核查 |
| 零新产生式 | §27 不动；§69.3 为编译器内部 struct、§21.8 为 std API + 生命周期图 | ✅ 已核查 |

---

## 9. 落地映射（合并进 AILANG.md 的位置）

| 本 RFC 内容 | 落地位置 | 性质 |
|---|---|---|
| §4.1 Diagnostic struct 扩展（Severity 5 级 / `code` / `related` / `suggestion` / `category`） | §69.3 内部 struct 扩展（Part IV 编译器实现层） | **资料性**（Part IV 实现 detail；§69.3 按 RFC 0005 §5 归 Part IV informative，不升格为合规义务来源） |
| §4.2 诊断发射协议（JSON Lines schema + `Span` 专用形状 + 形状映射 + `suggestion` 应用语义） | 新增「诊断发射协议」小节（独立于 §69.3） | **规范性**（工具链合规契约：MUST 支持 `--diagnostics-format=json` + 字段表 + `Span` 专用形状及其与 §69.1/§24 的映射——非声称全规范统一、不覆写冻结形状） |
| §5 AILxxxx 编号空间 + 登记 | §18.2 展开 + 附录 E（错误码登记表） | 规范性（编号空间 + 段边界不变量）+ 资料性（登记表种子码） |
| §5 AIL7008/7009（§20 requires/ensures 运行期违约→panic） | §17 line 736「已知 panic 原因」清单补注（**声明非穷尽**）+ §20 line 990 / §27 line 1480 补违约→panic 条目 + §34 line 1756-1764「已知 panic 触发原因 → 符号 → 指针」表对称补注（加 `requires/ensures` 运行期违约行〔符号列「（直接 panic，无独立符号）」、指针列「§20 / §5 AIL7008/7009」〕、或与 §17 line 736 对称声明该表非穷尽） | 规范性（**gap-fill**：冻结 §20/§27 仅标「运行期断言」、未定失败行为；本 RFC 显式定义为 panic 并分配 AIL7008/7009——与 §92 #2 T(expr) 违约→panic ConstraintViolation 同族。合并后 §17 清单加「requires/ensures 运行期违约（AIL7008/7009）」一项或显式声明原清单非穷尽、由 §5 AIL7xxx 段闭合） |
| §6 TaskHandle 错误态 | §21.8 生命周期图 + §21.9 并发错误表补注 + §87 #4 补注 | 规范性（Part I 并发） |
| §6 `TaskStatus` / `TaskFailure` / `FileSpan` 类型 + `DiagnosticCode` 运行期投影 | §34 std.core（std.async lang item 子节 §34 line 1783——`TaskStatus` / `TaskFailure` / `FileSpan` / `DiagnosticCode` 运行期投影类型表） | 规范性（std API；`FileSpan` 为运行期定位类型、`DiagnosticCode` 运行期投影为 `{number, slug}`——二者**非** §69.1/§5.2 编译器内部类型（后者无 `file` 字段 / 为 Rust `&'static str` 注册项、ail 不可名）） |
| §6 `is_failed` / `status` / `failure` 查询方法 + `on_unhandled_task_failure` 全局处理器 | §34 std.async lang item 子节（line 1783，**新建** `TaskHandle` 方法签名表——同节 `Optional` line 1721 / `Result` line 1735 方法表格式先例；codegen 同步更新 §77 line 3104）+ §87 #4 补注 | 规范性（std.async 方法 + 全局处理器、均非关键字——§87 已确认此类上下文方法不计 56） |
| §7 交叉更新 | RFC 0005 §3 / §1.5.4 + RFC 0003（`panic_symbol`→`slug`）/ 0004 引用更新 | 同步 |

---

## 10. 开放问题

1. **`failure()` 取回即 ack 的精确语义**——§6.3 定为「调 `h.failure()` 得 `Some` = 确认、不传播」。备选：需显式 `h.ack_failure()` 分离查询与确认。推荐取回即确认（同 Result handle 同构、更简洁），留 review。
2. **join 传播 vs 父级 Result**——§6.3 #1 scoped 子 Task `Failed` 默认 panic 传播至父（隐式作用域末 join）；§6.3 #2 已为 v0.3 定义**显式 `h.join()` = 阻塞至终态、不传播**（runner 经 join 后 `failure()` 取回 ack，非传播观测路径）。是否再提供「父以 Result 接收子失败」的高阶 API（如 `try_join` 返 `Result<T, TaskFailure>`）？v0.2 task 体 void + CSP 通道，倾向于经通道传失败消息而非新 join 算子；非 panic 高阶 join 留 std.async 设计（P1+）。
3. **`--diagnostics-format=sarif`**——§4 定 json 为 v0.3 必备；SARIF（OASIS 标准，CI/IDE 通用）作为未来格式留 review。
4. **Actor 死信可观测对称性**——§6.3 提 actor handler panic → 死信经 `ActorHandle` 查询 `TaskFailure`；其精确 API（与 TaskHandle 对称的方法集）留 std.async 设计。
5. **诊断码登记的版本化**——§5 slug 一经发布不变；number 跨 major 可重编。major 版本切分点（v1.0？）与重编策略留治理（RFC 0005 §9 lifecycle）。
6. **全局未处理失败处理器的默认行为与触发时机**——§6.3 detached `Failed` 默认「日志 + 继续」，触发时机定为 handle drop（未 ack）+ 进程退出 sweep（Task 级，含 handle 已 drop 的孤儿）；是否默认 panic（更严格、更「不隐藏」）？推荐日志（detached 本意解耦、不应一个 detached 失败崩进程），留 review。
7. **资源耗尽（栈溢出 / OOM）与 double-panic 的终止语义**——RFC 0005 §6 将二者归入「未指定行为」并把终止语义（abort / 进程终止模型）的收紧显式留给后续运行期错误模型 RFC（P1+，待立项）。本 RFC（= P0-3）的主题为诊断协议 / 错误码 / TaskHandle 错误态，**不**覆盖进程级资源耗尽终止语义——该项**显式 deferred** 至后续运行期错误模型 RFC（P1+）。本 RFC 维持 RFC 0005 §6 现状（二者归未指定、非 UB、不开 §17 外新例外）。
8. **scoped 子 Task of 永不终结父的失败可观测性 gap**——§6.3 #3 残留 gap 条款诚实登记：scoped 子 `Failed` 在父作用域永不终结（父卡于 CPU-bound 无限循环、§21.8 协作式取消仅 await 点生效不可强制终结）时**无 sound 可观测出口**（#1 隐式 join 永不触发 / Fail-time 投递仅 detached / ack 需父调用而父卡死 / sweep 需进程退出而卡死父保活致进程可能永不退出）。此为结构化并发的固有边界——把 Fail-time 投递对称延伸至 scoped 子会消除父的 ack 窗口、使 scoped 退化为 detached 语义，不在本 RFC 闭合范围。**v0.3 缓解**：长驻服务 / supervisor 需保证失败必可观测时用 `spawn detached`（享 Fail-time 投递、不依赖父 join）而非 scoped 子、或确保父作用域可达 join；sweep 在进程最终退出时 best-effort 补投（进程可能永不退出）。该 gap 的彻底闭合（scoped Fail-time 投递 / 可取消 CPU-bound task）**显式 deferred** 至后续运行期错误模型 RFC（P1+，同 OQ #7 同一 RFC）。pass-7 曾把此 gap 误归 sweep sound 覆盖、pass-8 诚实改判为残留 gap。

---

## 11. 收敛轨迹

**Draft v1 → pass-1 → pass-2 → 修正后待 pass-3**。pass-1（与 RFC 0009 并行）报 **0H / 9M / 3L = 12 条**（RFC 0008 部分），已逐条修正（pass-1 摘要见下）。**pass-2 报 0H / 15M / 3L = 18 条**（更深探测，含 pass-1 回归），已逐条修正：

- **Span 虚假统一**：§4.2 撤回「全规范统一」过度声明——JSON `Span` 改为诊断发射协议**专用形状**（§69.1 内部 `Span` 无 file / §24 `.ailmeta` span 字段名不同，**各自保持不变**、显式给出映射，不覆写冻结形状）；`col` 由「字节/字符由实现文档化」（实现定义行为）收紧为**按 Unicode 标量值计字符列**（定义行为）。
- **DiagnosticCategory 缺 `Runtime`**：补 `Runtime` 变体（AIL7xxx 运行期 panic 有忠实 category，`category:"runtime"`）。
- **`Help`(severity) 与 `suggestion`(field) 语义重叠**：定义二者**正交**（机器补丁由 `suggestion` 独立承载、与 severity 无关；任一 severity 均可携带 suggestion），修正 rust-analyzer/clang-tidy 引用为 LSP severity+CodeAction 分层。
- **显式 `std.core.panic` 无 AIL7xxx 码**：补 **AIL7010 `explicit-panic`**（§17 全部 panic 触发均有码）；§6.2 `TaskFailure.code` 非 Optional 据此坐实。
- **AIL5001 `requires` 静态拒收无 v0.2 依据**：标为**预留码**（激活须待 RFC 0002 Tier 2；v0.2 `requires` 纯运行期 §20/§83 #3）。
- **`panic_symbol`↔`slug` 非双射**：§7 撤回「PascalCase 别名」过度声明，补显式映射表 + 三条非双射诚实声明（1:N AssertionFailure→三 slug / ConstraintViolation 同名碰撞消解：AIL5002 改 `constraint-bound-check` / 「无独立符号」≠「无诊断码」）。
- **detached 孤儿失败**：exit-sweep **重锚 Task 身份**（非 handle 存活）——闭合「handle 早 drop + 晚 panic」静默丢失。
- **`failure()`-ack 误引 §89 #1「Result 必须 handle」**：改引 §87 #1 / §21.9「不隐藏错误」（冻结规范无 Result 强制 handle 规则）。
- **栈溢出/OOM/double-panic handoff**：§7 显式声明 **out-of-scope**（属进程终止模型、非本 RFC 三主题），§10 OQ #7 deferred 至后续错误模型 RFC。
- **§3 #4 应→必须**、**§5.2 段边界 AIL5001 Tier-2 标注**、**§8 自洽表同步**。

pass-1 摘要：① 5xxx(编译期)/7xxx(运行期) 边界 + AIL7008/7009 + AIL7005/7006 拆分；② AIL8xxx FFI/ABI/布局/codegen 段 + DiagnosticCategory `Ffi`；③ §4.2 JSON 字段表 + suggestion 应用语义；④ §6.1 Failed 由 Running/Waiting 转入；⑤ §6.2 TaskStatus 忠实投影；⑥ §6.3 detached ack/drop/exit-sweep 互斥；⑦ §7 panic_symbol→slug；⑧ §9 normative/informative 拆分。待 pass-3 复验收敛 + regression。

**pass-3 报 0H / 5M / 1L = 6 条 confirmed**（refuted 3、共 raw 9），已逐条修正、待 pass-4 复验：

- **F1（M，ailxxxx-code-space）**：§1 + §5.3 + §9 协同——§1 把「不修改 §17 panic 模型」收紧为「不修改 §17 **已列** panic 触发的语义；为 §20 requires/ensures 运行期违约（冻结 §20/§27 仅标「运行期断言」、**未定失败行为**）显式定义为 panic 并分配 AIL7008/7009（gap-fill 非 overturn，与 §92 #2 T(expr) 同族）」；§5.3 非 Optional 论证纳入 AIL7008/7009（§20 源）；§9 落地映射加 §17 补注行（声明清单非穷尽）。
- **F2（M，taskhandle-failed-state-closure）**：§6.3 新增 #2 显式 `h.join()` 条款——阻塞至终态、**不**传播 panic（传播仅属 #1 隐式作用域末 join 对未 ack Failed），并给出 PBT runner 规范序列 `join → is_failed → failure`（v0.3 落地、不依赖 OQ #2 P1+ `try_join`）；§10 OQ #2 同步标注显式 join 已 v0.3 定义。
- **F3（M，taskhandle-failed-state-closure）**：§6.3 #3 把孤儿兜底从「进程退出 sweep」提前为 **Fail-time 投递**——runtime 在 detached Task 转入 `Failed` 瞬间且未 ack 时立即投递全局处理器（覆盖孤儿 drop + 永不退出进程）；exit-sweep 退为兜底（应对 Fail-time 投递遗漏 / 全局处理器自身失败等退化）。
- **F4（L，taskhandle-failed-state-closure）**：§6.1 Cancelled 注释从「Running/Waiting 任一」收紧为冻结 §21.8 的「Waiting → Cancelled，仅在 await 点生效；CPU-bound 须 task.yield() 让出」——图与注释、注释与 §21.8 三者自洽。
- **F5（M，join-failure-propagation）**：与 F3 同根——Fail-time 投递一并闭合 immortal handle 未 ack 静默漏洞；§6.3 设计说明弱化「最终必经全局处理器出口」的过强声称（改为 Fail-time 已使长驻进程亦能及时报告，exit-sweep 退为兜底）。
- **F6（M，ai-consumability-0008）**：§4.2 撤回「RFC 0003 PBT 反例 span 经本协议 JSON `Span` 发射」单方面断言——本 RFC 仅定义编译器诊断发射 JSON `Span`；PBT 反例 span 归 RFC 0003 §9.6 自身 schema（现行 §24 形、跨 RFC 边界不代为登记）。§8 自洽表同步。

**pass-4 报 0H / 4M / 0L = 4 条 confirmed**（default-refute workflow，raw 全部 confirm 无 refute），已逐条修正、待 pass-5 复验：

- **F1（M，diagnostic-protocol-soundness）**：§4.1 `RelatedSpan.span` 由 `Span`（必需）改为 `Optional<Span>`；§4.2 `related` JSON 字段 `{label, span}` 改 `{label, span?}`（span 可选）——恢复冻结 §69.3 `notes: Vec<(String, Option<Span>)>` 的 span-less 子注记能力（`related` 取代 `notes` 时丢失的能力）；`label` 保持必需。
- **F2（M，diagnostic-protocol-soundness）**：§4.2 加「发射流」条款——JSON Lines 输出到 **stderr**（对齐 rustc/clang）、stdout 留给 `--print` 编译产物；human / json 共享同一 stderr 流——闭合 RFC 0005 §3 #4「诊断必须结构化」MUST 条款对发射流的 normative 歧义（CI / 编辑器集成无 stdout/stderr 歧义）。
- **F3（M，ai-consumability-0008）**：§4.1 `DiagnosticCategory` 删 `Other` 兜底变体（12 变体留）；§4.2 JSON `category` 删 `"other"`——维持 `category`↔AILxxxx 段双射、闭合 §5「每诊断一稳定 code」契约（AIL0xxx ICE 与 Other「真实但跨类」语义不同、不可复用）；§4.1 补无兜底 rationale；§8 自洽表加 12 变体无 Other 核查行。
- **F4（M，ai-consumability-0008）**：§6.2 新增 `FileSpan`（std.core 运行期定位类型 `{file, line, col}`、§34 落地）、`TaskFailure.span` 改 `Optional<FileSpan>`——非 §69.1 编译器内部 Span（无 file、ail 不可名）、非 §4.2 JSON 发射形状；闭合全局失败处理器（§6.3 #3）定位运行期 panic 源文件的可观测目标；§9 落地映射同步加 FileSpan。

**pass-5 报 0H / 4M / 3L = 7 条 confirmed**（default-refute workflow，refuted 2、共 raw 9），已逐条修正、待 pass-6 复验：

- **F1（聚簇·4 findings，覆盖 ailxxxx-code-space / diagnostic-protocol-soundness，根因 = AIL0xxx ICE 段无 DiagnosticCategory 归宿 + 「双射」措辞数学错误[实为多对一满射、非双射]）**：§4.1 `DiagnosticCategory` 加第 13 变体 `Internal`（JSON `"internal"`、↔AIL0xxx ICE）+ 把全文「`category`↔AILxxxx 段双射」措辞改为「`category`→AILxxxx 段**满射**（非双射、category→段多对一，如 Lex/Parse→AIL1xxx、Effect/Contract→AIL5xxx、Ffi/Codegen→AIL8xxx）」；§4.2 `category` JSON 枚举加 `"internal"` + 双射→满射；§5.1 加 `AIL0001 internal-compiler-error` 的 DiagnosticCode 归属注（`category=Internal`、`default_severity=Error`、经 §69.3 `Diagnostic` 吐出）；§8 自洽表「12 变体无 Other」行→「13 变体（含 Internal↔AIL0xxx）」+ 双射→满射 + 覆盖 AIL0xxx–AIL8xxx 全 9 段——闭合 ICE 段无 category 归宿 + 「双射」实为满射两个根因（原 pass-4 删 Other 兜底后 ICE 段（AIL0xxx）确无对应 category，本次补 Internal 变体补齐）。
- **F2（M，taskhandle-failed-state-closure）**：§6.2 新增 `DiagnosticCode` AILang 运行期投影 `struct DiagnosticCode { number: string, slug: string }`（std.core、§34 落地）——区分 §5.2 编译器内部 Rust 注册 struct（`&'static str`、含 `default_severity`/`category` 编译期元数据）与 AILang 运行期层投影（仅 `{number, slug}`、编译期元数据不进运行期值），闭合 `TaskFailure.code: DiagnosticCode` 字段类型在 AILang 层的类型完备性（字段类型须在 AILang 类型集内定义、非直接引用编译器内部 Rust 类型）；§9 落地映射同步加 DiagnosticCode 投影。
- **F3（M，join-failure-propagation）**：§6.3 #3 ack 互斥条款显式声明 **scoped/detached 不对称**——对 scoped，作用域末隐式 join 前有 ack 窗口（`failure()` 取回 `Some` 即 ack、抑制向父传播）；对 detached，Fail-time 投递锚定「转入 `Failed` 瞬间且当时未 ack」、必先于任何用户 `failure()` 调用触发，故 detached 的 `failure()` 永远是 informational 查询非 ack——消除「可经 `failure()` 抑制 detached 全局投递」误读、闭合 ack 机制作用域 = scoped only。
- **F4（L，join-failure-propagation）**：§6.3 #3 全局处理器条款显式定义 **panic 的传播目标**——handler 从 runtime（executor）上下文被调用、不在任何 Task 体内；§17（line 736 / §3119）panic 展开目标穷举为「Task 边界或 `extern` 帧」、runtime 上下文二者皆非 ⇒ 无吸收边界 ⇒ 转进程 abort（与 `extern` 帧 panic→abort 同族、非 UB；§17 无 `catch_unwind` 禁吞）——闭合与 §17 panic 展开目标穷举的张力、全局处理器 panic→abort 为定义行为。

**pass-6 报 0H / 2M / 2L = 4 条 confirmed**（default-refute workflow，refuted 2、共 raw 6），已逐条修正、待 pass-7 复验：

- **F1（M，join-failure-propagation）**：§6.3 #3 exit-sweep 条件「遍历注册表中**所有**未投递且未 ack 的 `Failed` Task」**遗漏 scoped join 传播这一终态**——scoped 子 Task 处 `Failed` 时作用域末隐式 join 必先于进程退出触发（#1 panic 传播至父），未 ack 的 scoped `Failed` 不可能存活到 sweep，故 sweep 应**仅扫 detached**（否则与 #1 出口构成同一失败双重观测）。修正：exit-sweep 条件收窄为「**所有 detached 且**未投递、未 ack 的 `Failed` Task」+ 显式「仅扫 detached（不扫 scoped）」注（scoped 未 ack 出口已由 #1 兜底）；§6.3 设计说明 lifecycle 模型同步扩展为按 Task 归属分两类出口（detached：ack ∨ 全局处理器投递；scoped：ack ∨ 向父传播）——三类出口互斥单一命中、闭合「scoped 失败被双重观测」漏洞。
- **F2（L，join-failure-propagation）** + **F3（M，frozen-boundary-0008）**（同根——系统性误引 §37 作 std.async 类型/方法落地点）：§37（line 1852）冻结为 `std.fs`（文件系统）、**非** std.async；`join` 列为 `TaskHandle` 方法处在 §77 line 3104 codegen 表、std.async 类型（`Future`/`TaskHandle`/`ActorHandle`）落在 §34 std.core std.async lang item 子节（§34 line 1783）。修正三处误引：§6.3 #2（line 259）+ §8 自洽表（line 328）的「§37 line 3104」→「§77 line 3104 codegen 表」并补「§87 #4 仅定隐式 join、显式 join 语义为本 RFC gap-fill」；§9 落地映射（line 349）的「§37 std.async（`TaskStatus`）」→「§34 std.core std.async lang item 子节（§34 line 1783）」。
- **F4（L，frozen-boundary-0008）**：§9 落地映射**遗漏新 std.async API 面**（`is_failed`/`status`/`failure` 查询方法 + `on_unhandled_task_failure` 全局处理器）的落地行——仅有类型行（`TaskStatus`/`TaskFailure`/`FileSpan`/`DiagnosticCode`）、缺方法 + 处理器行。修正：§9 新增「§6 `is_failed`/`status`/`failure` + `on_unhandled_task_failure` → §34 std.async lang item 子节（§34 line 1783 `TaskHandle` 方法表，同 `cancelled`/`cancel`/`join` 先例 §77 line 3104）+ §87 #4 补注」行，闭合新 API 落地映射完备性。

**pass-7 报 0H / 6M / 0L = 6 条 confirmed**（default-refute workflow，refuted 2、共 raw 8），已逐条修正、待 pass-8 复验：

- **F1（M，taskhandle-failed-state-closure）+ F2（M，join-failure-propagation）+ F4（M，join-failure-propagation）**（同根——pass-6 F1 把 exit-sweep 收窄至「仅扫 detached」的论证在三类合法情形下不成立）：① scoped 父作用域永不终结（detached 父 Task 卡于 CPU-bound 无限循环——§21.8 协作式取消仅 await 点生效、不可取消）→ #1 隐式 join 永不触发、scoped 子 `Failed` 四条出口全落空；② root / 入口 Failed（`fn main()` 同步非 Task）无父可传、非 detached、「达进程根 ⟹ 进程终止」无冻结依据且与 §17「不传染进程」张力；③ 多子 Task 同时 `Failed`（panic 一次性 unwind、父仅 panic 一次）→ 次级失败无出口。**统一修正**：引入**「已出口」标记单一非重复机制**——ack / #1 传播 / Fail-time / drop / sweep 任一出口命中即标记、其余跳过；exit-sweep **恢复为 catch-all**（遍历全部未 ack 且未出口 `Failed`、**不限定 detached**、含 scoped 与 root）；#1 补**多子 Task**裁定（首因 panic、其余经全局处理器投递、各自标记已出口）；撤销「传播链必达进程根 ⟹ 进程终止」绝对声称（进程终止取决于全局处理器实现、sweep 不依赖该前提）——双重观测风险由「已出口」标记根治（pass-6 收窄所欲避免者已由标记闭合）、三类漏掉情形由 catch-all 兜底。
- **F3（M，join-failure-propagation）**：「handle drop 触发」条款缺「未已出口」前置条件——与 Fail-time 投递（转 `Failed` 瞬间即触发、必先于任何后续 drop）构成双重投递路径、违反「不重复」。修正：handle-drop 补「且该 `TaskFailure` 未标记「已出口」」前置条件 + 标记「已出口」、并注明 drop 触发实为 Fail-time 遗漏时的二次兜底（已并入上 F1「已出口」标记机制）。
- **F5（M，cross-rfc-update-0008）**：§7 line 287 + §10 OQ#7 仍把 RFC 0005 §6 资源耗尽终止语义指针陈述为「留给 P0-3」——但 RFC 0005 §6 已 pass-19（commit 9b315e2）改为「后续运行期错误模型 RFC（P1+，待立项）」。修正：两处「留给 P0-3」→「留给后续运行期错误模型 RFC（P1+，待立项）」（本 RFC = P0-3 的自身定位不变、仅同步对 RFC 0005 现状的描述）。
- **F6（M，ai-consumability-0008）**：ICE（AIL0001）经 §69.3 Diagnostic 吐出但「无用户源码上下文」（linker / codegen / 跨 pass 协议层元失败）时 span 无处安放——§4.2 Span 五字段全必需 + 1-based 不变量禁止 line=0/col=0 占位。修正：§4.1 `Diagnostic.span` 改 `Optional<Span>` + §4.2 JSON `span` 必需「是」→「否」（可缺省承载 span-less 诊断）——与既有 `RelatedSpan.span: Optional<Span>` 同构，AI 消费方可凭 span 是否 null 区分「真实源码定位」vs「编译器元失败无定位」；有源码定位时五字段全必需、1-based 不变量不变。

**pass-8 报 0H / 1M / 1L = 2 条 confirmed**（default-refute workflow，refuted 3、共 raw 5），已逐条修正、待 pass-9 复验：

- **F1（M，taskhandle-failed-state-closure）+ F2（L，taskhandle-failed-state-closure）**（同根——pass-7 F1 把 exit-sweep 恢复 catch-all 时、把「scoped 子 of 永不终结父（①）」与「root/入口 fn main() 非 Task（②）」两类情形误归 sweep sound 覆盖）：① **sweep 无法 sound 覆盖 scoped 子 of 永不终结父**——sweep 仅在进程退出时触发，而该卡死父（如 detached 长驻卡于 CPU-bound 无限循环、§21.8 协作式取消仅 await 点生效不可强制终结）保持进程存活致进程可能永不退出 ⟹ sweep 可能永不触发；Fail-time 投递仅 detached（scoped 子不在其列）、ack 需父调 `h.failure()` 而父卡死无法调——四条出口全落空。pass-7 曾以「catch-all 兜底三类情形」声称覆盖，实为**过强声称**。② **root/入口 fn main() 非 sweep 场景**——`fn main()` 为同步函数、非 Task（§17 / §21.2）、不在 Task 注册表、自身 panic 无 Task 边界吸收 ⟹ 进程 abort（§17 同族、定义行为非 UB）；pass-7 曾把「root/入口 Failed 无父可传」误归 sweep 兜底，实为**概念错误**（fn main() 不构成存活 `Failed` Task、非 sweep 扫描对象）。**统一修正（option b 诚实残留 gap 声明、不延伸 Fail-time 至 scoped）**：§6.3 #3 sweep 条款重写为 **best-effort 兜底**（sound 作用仅一项 = runtime 实现层防御性兜底、正常路径不依赖）+ 新增**残留 gap 条款**（① scoped 子 of 永不终结父诚实登记为结构化并发固有边界、缓解 = `spawn detached` 享 Fail-time 投递 / 确保父可达 join、sweep 进程退出时 best-effort 补投但进程可能永不退出）+ **root/入口条款**（② 移出 sweep、归 abort 路径）；设计说明同步（四类出口枚举的 sweep 项 + exit-sweep 段均去除「三类 sound 覆盖」声称）；新增 §10 开放问题 #8（该 gap 彻底闭合 deferred 至运行期错误模型 RFC P1+，同 OQ #7）。**不选 option a（Fail-time 延伸 scoped）**——会消除父的 ack 窗口、使 scoped 退化为 detached 语义、破坏 scoped 契约（父负责在 join 处理子失败）。

**pass-9 报 0H / 0M / 2L = 2 条 confirmed**（default-refute workflow，refuted 1、共 raw 3），已逐条修正、待 pass-10 复验：

- **F1（L，ailxxxx-code-space-completeness）**：§8 自洽核查表 line 326「`AIL7xxx` panic 段 ↔ §17 panic 命名 ✅ 一一映射」——与 §7 显式 1:N 声明矛盾（§7 line 292/295/308 明示映射**非**全局 1:1：4 具名符号 1:1→AIL7001-7004、`AssertionFailure` 1:N 拆分→AIL7005/7006/7007、显式 `panic` 无符号补码→AIL7010、requires/ensures gap-fill→AIL7008/7009）。pass-5 曾把相邻的「`category`↔AILxxxx 段双射」精确化为「满射（非双射）」、但遗漏此 AIL7xxx↔§17 panic 命名行的「一一映射」措辞。修正：§8 line 326「✅ 一一映射」→「✅ 全覆盖（映射**非**全局 1:1：4 具名符号 1:1→AIL7001-7004 + `AssertionFailure` 1:N 拆分→AIL7005/7006/7007 + 显式 panic→AIL7010 + requires/ensures gap-fill→AIL7008/7009；详见 §7 映射表 + 三条非双射诚实声明）」——与 §7 数学精确化标准一致。
- **F2（L，taskhandle-failed-state-closure）**：§9 落地映射 line 353「§34 line 1783 `TaskHandle` 方法表，同 `cancelled` / `cancel` / `join` 先例 §77 line 3104」——冻结 §34 line 1783-1784 实为散文列举段（`Future<T>`...`TaskHandle`...`ActorHandle`...均为 `std.async` lang item）、**无** TaskHandle 方法签名表（§34 仅同节 `Optional` line 1721 / `Result` line 1735 两表有方法签名表）；原措辞误指为既有表、且把 §77 codegen 表的 `cancelled`/`cancel`/`join` 先例错缀于 §34。修正：line 353 改「§34 std.async lang item 子节（line 1783，**新建** `TaskHandle` 方法签名表——同节 `Optional` line 1721 / `Result` line 1735 方法表格式先例；codegen 同步更新 §77 line 3104）」——明确新建表、以同节 Optional/Result 方法表为格式先例、并区分 std API（§34）与 codegen（§77）两层落地。

**pass-10 报 0H / 0M / 2L = 2 条 confirmed**（default-refute workflow，refuted 0、共 raw 2），已逐条修正、待 pass-11 复验：

- **F1（L，diagnostic-protocol-soundness）**：§4.1 Diagnostic/RelatedSpan 内部 Rust struct（§9 line 347 明示「§69.3 内部 struct 扩展（Part IV 编译器实现层）」、冻结 §69.3 line 2774-2779 全文 Rust `Option<T>` + `pub struct`/`String`/`Vec`）同体混用 ailang 记法 `Optional<Span>`（line 65 RelatedSpan.span、line 74 Diagnostic.span——pass-4 F1 + pass-7 F6 编辑引入的记法漂移、未同步 suggestion 字段 `Option<Suggestion>` 也未回改两 span 为 Rust）与 Rust `Option<Suggestion>`（line 76）——记法回归（Rust 无 `Optional<T>`、ailang `Optional<T>` 不与 `pub struct` 同形；行内注释 line 65 引冻结 §69.3 `Option<Span>` 用 Rust、紧邻字段却写 ailang `Optional<Span>` 自相矛盾）。修正：§4.1 line 65/74 + §4.2 line 106/107 引用统一 `Optional<Span>`→`Option<Span>`（含行内注释 prose）——与冻结 §69.3 `Option<T>` 同构、同 struct 三字段 `Option<Span>` ×2 + `Option<Suggestion>` ×1 一致（§6.2 TaskFailure ```ail 块的 `Optional<FileSpan>` 不动——为 ailang 运行期类型、记法正确；changelog pass-4 F1〔line 399〕+ pass-7 F6〔line 422〕历史记录的 `Optional<Span>` 为当时编辑事实、不改写历史）。
- **F2（L，frozen-boundary）**：AIL7008/7009 落地映射（§9 line 350）列 §17 line 736 + §20 line 990 + §27 line 1480 三处冻结位置更新、但遗漏冻结 §34 line 1756-1764「已知 panic 触发原因 → 符号 → 指针」表——该表为 Part II normative core（RFC 0005 §5 划 §28-§42）、与 §17 line 736 prose **完全平行**（同 5 panic 因）；requires/ensures 运行期违约为**新 panic 因**（非既有因诊断码补登），合并后若不同步加行/声明非穷尽将与被补注的 §17 line 736 形成两处 normative frozen 位置间口径不一致。修正：§9 line 350 落地位置列补「+ §34 line 1756-1764 表对称补注（加 `requires/ensures` 运行期违约行〔符号列「（直接 panic，无独立符号）」、指针列「§20 / §5 AIL7008/7009」〕、或与 §17 line 736 对称声明该表非穷尽）」+ §7 line 311「§34 panic 符号表不变」收紧为「§34 **既有因的符号分类立场**不变（unwrap/assert/explicit-panic 仍无独立符号；requires/ensures 为表中无行的新因、由 §9 落地映射对称补注处理）」——避免被误读为整张 §34 cause 表不变。

**pass-11 报 0008 clean（0 confirmed）**（default-refute workflow、refuted 0、共 raw 0）——pass-10 两 fix〔F1 Optional→Option §4.1 line 65/74 + §4.2 line 106/107 + F2 §9 line 350 §34 表对称补注 + §7 line 311 收紧「既有因的符号分类立场不变」〕均 hold、无回归、无新 gap。0009 同 pass 报 1L〔ailmeta-registry-authority〕、见 0009 changelog。

**pass-12 报 0H / 1M / 1L = 2 条 confirmed**（default-refute workflow、refuted 0、共 raw 2），已逐条修正、待 pass-13 复验：

- **F1（L，taskhandle-failed-state-closure）**：§6.3 #1 多子 Task 同时 Failed 时「首因」被当 normative 术语使用却全文未定义其选择规则（grep「首因」仅 line 259 / 274 / 419 changelog、无任一处定义；冻结 AILANG.md grep 亦空——为 RFC 0008 新引入术语）。后果：多子并发 Failed 时 runtime 须从多个未 ack Failed 子挑一个让父 panic（其余投全局处理器），挑选顺序未定（spawn 序？注册表迭代序？转入 Failed 时序？）→ 父 panic 的 DiagnosticCode/slug（如 AIL7001 vs AIL7003）随实现而异、跨实现/跨运行不稳定——违 **RFC 0007 §2 line 34**「不能免费确定的（Map 迭代序）必须显式登记为未指定」MUST 原则（首因恰属「不能免费确定」的并发固有非确定性 §21 M:N work-stealing、却既未定义亦未声明 unspecified 亦未登记 OQ）、削弱 **§5.2 line 174**「slug 一经发布不得改名/改义」稳定 slug 契约。soundness 不受影响（「已出口」标记保证不丢/不重、finding 诚实承认）。修正（finding suggested_fix **option ① 首选**）：§6.3 #1 line 259 显式定义「首因」= **spawn 序**（源码静态出现序、取首个未 ack `Failed` 子 Task）——**确定性、跨实现/跨运行一致**（spawn 静态序不依赖 §21 M:N work-stealing 执行序非确定性、故**可免费确定**；对齐 RFC 0007 §2 line 34——此处既可免费确定即定义为 spawn 序、而非留 unspecified；服务 §5.2 line 174 稳定 slug 契约——父 panic 的 DiagnosticCode/slug 不随实现迭代序而异）。
- **F2（M，join-failure-propagation）**：F4（§6.3 #3 line 268）soundness 论证「全局处理器从 runtime（executor）上下文被调用——handler 不在任何 Task 体内 ⟹ panic 无 Task 边界可吸收 ⟹ abort」的绝对前提对 **§6.3 #1 多子次级投递路径**不成立：次级投递「在 join 退出时经全局处理器投递」（line 259 + 设计说明 line 274 两处均明确「在 join 退出时」、非「推迟到 executor 异步调度」）；隐式 join 为编译器插入父 Task 作用域末的代码、在**父 Task 栈帧上执行**（冻结 §21 line 1034/1087 + §87 #4 line 3299 作用域末隐式 join），故次级投递于**父 Task 上下文同步发生、非 runtime 上下文**。若 handler 此路径 panic：① 展开处于父 Task 栈、§17 line 736 / §3119 最近 Task 边界 = 父 Task 边界 → 父进 `Failed`（**非** F4 声称的 abort）——与 F4 矛盾；② 与首因 panic 时序/合并未定 → double-panic（落 RFC 0005 §6 / §10 OQ#7 未指定、未由本 RFC 闭合）；③ handler panic 截断次级投递 → 未及投递的剩余次级 Failed 子既未标「已出口」、父已 panic 无法处理、仅靠 sweep 进程退出补投 → 长驻进程可能永不退出致静默丢失（**新**丢失路径、非 §10 OQ#8 登记的 scoped 子 of 永不终结父）。F4 前提仅对 Fail-time（detached、runtime 上下文）/ exit-sweep（进程退出、runtime 上下文）成立；handle-drop 为 moot（Fail-time 必先于 drop 触发 line 264）。**唯一真问题 = #1 多子次级路径**。修正（finding suggested_fix **option ①**）：澄清**次级投递由 executor 在 runtime 上下文异步调度**（非父 Task 栈同步执行——触发时机在 join、执行上下文在 executor、二者分离），使 F4「handler 不在任何 Task 体内」前提对**全部投递路径**一致成立（detached Fail-time〔line 264〕/ handle-drop〔line 263〕/ exit-sweep〔line 265〕/ #1 多子次级〔line 259〕）——§6.3 #1 line 259〔次级投递改为「触发、由 executor 在 runtime 上下文投递」〕+ #3 line 268〔追加 F4 普适性注、显式覆盖 #1 多子次级路径、消除「handler 于 join 上下文同步调用致 F4 失效」解读歧义〕+ 设计说明 line 274〔两处「在 join 退出投递」同步为 executor 上下文〕三处一致；三后果〔handler panic→abort 而非父 Failed / 无 double-panic / 无截断静默丢失——abort 为定义的可观测出口、非静默继续〕一并消解。
- **F1（M，join-failure-propagation，pass-13）**：pass-12 F2 把 §6.3 #1 多子次级投递由父 Task 栈同步改为 executor 异步调度（闭合 F4 普适性）后、引入新回归——root scoped 父 Task〔父为 `fn main()`〕多子同时 Failed 时、首因 panic 经 #1 传播至 `fn main()` → abort〔§6.3 #3 line 267 root/入口条款〕、抢占已 enqueue 给 executor 异步投递的次级 TaskFailure〔worker 线程随 abort 死亡、无 atexit、exit-sweep 不触发——abort 非 graceful 退出〕、次级诊断信息〔slug/span〕可能永不被全局处理器观测、违反 §6.3 #1 line 259 末句「次级失败不静默丢失」**无限制断言**。pre-pass-12 同步投递〔在父 Task 栈上、父 panic 之前完成〕无此回归。逐项排除既有覆盖：sweep〔需进程退出、abort 非 sweep 场景 line 267 明定〕/ §10 OQ#8〔登记 scoped 子 of 永不终结父〔父卡死、进程可能永不退出〕、非 root abort〔父经 abort 终止、进程确定退出〕、不同场景〕/ changelog ③〔「无截断静默丢失」上下文为 handler panic 截断投递链、仅覆盖 handler 自身 panic、与 finding 场景〔首因传播抢占 executor、handler 可能正常不 panic〕无关〕/ F4 普适性注〔论 handler 被调用后 panic 的传播目标、非 handler 尚未被调用即因首因传播 abort 的抢占路径〕——均不覆盖。修正〔suggested_fix option (b)、非 (a)〕：§6.3 #3 line 267 加「**abort 前 drain 保证**」normative 条款——runtime 须在「语言语义触发的 abort」〔root scoped 父→`fn main()`→abort / `fn main()` 自身 panic→abort / 全局处理器 panic→abort〔F4 line 268〕/ `extern` 帧 panic→abort〔§17〕〕前 **drain 已 enqueue 的全局处理器投递队列**〔先完成 pending 投递、次级失败达全局处理器、handler 标记「已出口」、再终止进程〕；**OS 强制终止**〔SIGKILL / SIGTERM / OOM / hardware fault〕为进程外事件、runtime 无控制权、**非语言语义保证范围**〔同 §10 OQ#8 诚实排除风格〕。+ §6.3 #1 line 259 末句「不静默丢失」断言加 drain 保证 cross-ref〔名副其实、闭合于 abort 路径〕+ 设计说明 line 274 同步 drain 保证注〔四类出口的 abort 前 drain〕——三处一致。选 (b) 因：真闭合「不静默丢失」声称〔非削弱为「非 abort 路径」option (a)〕、保 F4 普适性〔handler 仍 runtime 上下文被调用、panic 仍→abort、仅 abort 时机推迟到 drain 之后〕、不破坏「不隐藏错误」哲学。

**pass-14 报 1 confirmed [0H/1M/0L]**（[M] taskhandle-failed-state-closure）已修正、待 pass-15 复验：

- **F1（M，taskhandle-failed-state-closure，pass-14）**：pass-12 把「首因」选择规则基准定为「spawn 序 = 源码静态出现序」〔pass-13 补 drain 保证未触此〕、并把 `parallel for` 列入「多子 Task 同时 Failed」适用情形——但同源码静态 `spawn` 位点产生多个并发子 Task 时〔`parallel for` 的每次迭代、或循环 / 递归体内累计存活的 spawn 句柄〕所有子 Task 共享同一源码静态出现序、「取首个未 ack Failed 子 Task」于同位点集合无确定指称（tie-break 未定义）；且「可免费确定」论证于 `parallel for` 失效〔其迭代调度依赖 §21 M:N work-stealing——line 259 自己承认的非确定性源、与「spawn 静态序不依赖 work-stealing」前提矛盾〕、违 RFC 0007 §2 line 34「不能免费确定的须显式登记」MUST；更甚冻结 §21.10 / §58 / §87 #9 把 `parallel for` 定义为表达式〔返回 `List<R>`、body 须 `R`、body 须 pure 或仅 io.read、无 TaskHandle 暴露 / 无独立 Task 身份〕——列入多子 Failed 即隐含「迭代 = Task」模型却无依据。soundness 不受影响（「已出口」标记保不丢 / 不重）；缺陷纯在确定性契约的完备性。修正（suggested_fix option (a) 最干净、对齐冻结规范）：§6.3 #1 line 259〔从多子裁定适用情形移除 `parallel for`——明示其为非 spawn 表达式、body panic 经表达式单一出口一次性传播〔非多子 Failed〕；+ spawn 序由「源码静态出现序」扩展为**良定义全序**：不同静态位点按源码静态出现序、同静态位点的多次 spawn 调用（循环 / 递归体内 spawn）按运行时 spawn 调用序〔父 Task 控制流顺序执行 spawn 调用、调用本身顺序发生、不依赖 §21 work-stealing 对子 Task 执行的调度〕——闭合同位点 tie、保「确定性 / 跨实现一致 / 可免费确定」声称真成立、闭合 RFC 0007 §2 line 34 MUST〕。

> 本批是「可观测」主题，验证重心在**协议/契约的完备性与 soundness**（尤其 join 失败传播无静默吞、诊断 JSON schema 对 AI 稳定可消费），比 0006 的形式化、0007 的确定性更偏「契约正确性 + 不变量守住」。

---

*本 RFC 由综合判断 [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 P0-3 + [`deep-review-2026-07.md`](../research/deep-review-2026-07.md) 可观测 4.0 驱动。落地后将闭合「编译期诊断非结构化」「错误码未分类」「TaskHandle 错误态未定义」三个被前轮决议引用却悬空的缝隙，把可观测维度从「运行期近零」向「失败全程可观测、可机器消费」推进，并为 RFC 0003 / 0004 的机器闭环补上共同的诊断输入契约。*
