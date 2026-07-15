# RFC 0002 · 契约进 .ailmeta —— Contracts 元数据化

| | |
|---|---|
| **状态** | 草案（Draft v8）—— 待 review（收敛轨迹 v1=7L→v2=4L→v3=1L→v4=4L[2 缺陷：§27/§71 误标 + span/effects 张力]→v5=1L→v6=1L→v7=4L[**实为 1 缺陷被 4 维同报**：§7#2 public 闸门 bullet「public：同在 / 杜绝 contracts 有、examples 无 的非对称条目」混淆**闸门层**(eligibility)与**数据层**(presence)——gate 只保证过闸资格对称、不保证字段共存]→v8 全修。v5→v6 修 §7#5「§17 panic 表」→§34；v6→v7 修 §7#2(a)「根本不同」伪区分（重构为单一维度 public 闸门 + effect/async 执行层对比）+ §10#6 交叉引用 §7.2(b)→§7.2；**v7→v8 修**：§7#2 public 闸门 bullet 显式区分**闸门层**（contracts/examples 一致放行阻断、杜绝闸门层非对称）与**数据层**（二字段各自 absent-if-none、§3；有契约无 doctest 的 public 函数可 contracts 在 examples 缺——此数据层非对称合法、内容驱动非闸门所致）。**全量 section 边界自审**（v6）：所有 `§X line N` 引用均落入正确 section 跨度，无残留误标）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94、56 关键字、110 决议不变）|
| **日期** | 2026-07-15 |
| **承接** | [`docs/research/ai-friendliness-2026-07.md`](../research/ai-friendliness-2026-07.md) §3 Tier 1c（「一切的基础」「关键缺口」）+ [RFC 0001](./0001-doctest.md)（共享 schema bump）|

---

## 1. 动机

研究存档 §3 定性 Tier 1c「契约进 .ailmeta」为**「一切的基础」「关键缺口」**。三轮代码审计确认根因：

- **§20 契约（`requires` / `ensures` / `effects` 块）在 v0.2.1 已存在**——`fn` 产生式（§27，line 1473）含 `contract*`，契约是现有文法的一部分（非本 RFC 引入）。
- **但 §24 .ailmeta schema（line 1330-1358）不输出 `requires` / `ensures`**——function 条目有 `identity`/`input`/`output`/`effects`/`errors`/`is_pure` 等字段，**唯独缺契约**。
- **§75（line 3036）AI 分析器把 `requires`/`ensures` 当数据源消费**，构建程序知识图谱——却**不写进 `.ailmeta` 字段**。即：编译器「读」得到契约，AI/外部工具「读」不到。

**后果**：AILang 核心卖点「代码主动表达意图 → 编译器产出 `.ailmeta` → AI 零猜测」**在契约层断裂**。函数的前置/后置条件是人类可读的运行时断言，却不是机器可读的元数据——AI 无法从 `.ailmeta` 得知「调用此函数需要满足什么、保证返回什么」。

本 RFC **不发明新语言特性**（契约已有），只填这个输出缺口，并为跨函数契约机械传播（研究存档 §4 #4 核心指令：堵 DAFNYCOMP 3.69% 组合性 gap）**准备结构化数据**。1c 是 Tier 1b PBT（`ensures`→属性、`requires`→生成器守卫）与 Tier 3 SMT side-car（传播引擎）的共同地基。

**四层层级**（承接 RFC 0001 §1 表格，补全契约与 constraint 两层）：

| 层 | 表达力 | 形式 | AILang 载体 | `.ailmeta` 槽位 |
|---|---|---|---|---|
| **doctest**（RFC 0001） | exists（具体实例） | `///` 内可运行块 | DocComment | `examples[]` |
| **契约（本 RFC）** | for-all（运行时前置/后置断言） | `requires` / `ensures` 块（§20） | 契约块 + 函数签名 | **`contracts`（本 RFC 新增）** |
| **constraint**（§15.4） | 类型级谓词（ refinement） | `constraint T { expr* }` | 独立声明 | `constraint` 字段（已存在，line 1350） |
| **meaning**（§15.3） | 自然语言语义 | `meaning T: "..."` | 独立声明 | `meaning`（input/output/field 槽） |

---

## 2. 设计目标

1. **零新关键字**（保 F6 的 56 不变）—— `requires`/`ensures`/`effects` 是 v0.2.1 **已有的硬关键字**（§9 line 384「AI 语义」类别——`meaning`/`effects`/`pure`/`requires`/`ensures`/`agent`/`tool`，**已计入 56**；区别于 line 393 的上下文关键字 `where`/`constraint`/…，三者不在其列）；本 RFC 只在 `.ailmeta` 侧输出其元数据，不引入任何新词法，56 不变。
2. **不触动 §20 契约语法**—— 契约块（签名与函数体之间，§14 line 482 / §27 line 1480-1481）的语法、语义、运行时行为**完全不变**；本 RFC 只在 `.ailmeta` 侧**镜像**它们。
3. **与 RFC 0001 共享 schema bump**—— 一次 `schema_version` 0.2.0 → 0.3.0 同时承载 RFC 0001 `examples[]` + 本 RFC `contracts`（+ 将来 1b `properties[]`），避免碎片化。
4. **为 Tier 3 side-car 预留机器可解析表示**—— 契约以结构化字段输出，使外部验证器能机器读取 callee 的 postcondition，为跨函数传播**备料**（传播引擎本身属 Tier 3，不在本 RFC）。

---

## 3. `.ailmeta` schema 扩展（核心）

`function` 条目新增**可选** `contracts` 字段（`schema_version` 0.2.0 → 0.3.0，与 RFC 0001 `examples[]` 共享）：

```json
{
  "identity": { "name": "withdraw", "description": "从账户取款。" },
  "input": [
    { "name": "account", "type": "Account", "mode": "borrow_mut" },
    { "name": "amount", "type": "int", "mode": "copy" }
  ],
  "output": { "type": "Result<void, BankError>" },
  "is_pure": false,
  "is_async": false,
  "effects": ["database.write"],
  "errors": ["InsufficientFunds", "AccountLocked"],
  "contracts": {
    "requires": [
      { "expr": "amount > 0", "span": { "file": "bank.ail", "line_start": 42, "col_start": 13, "line_end": 42, "col_end": 23 } },
      { "expr": "account.balance >= amount", "span": { "file": "bank.ail", "line_start": 42, "col_start": 25, "line_end": 42, "col_end": 48 } }
    ],
    "ensures": [
      { "expr": "account.balance == old(account.balance) - amount", "span": { "file": "bank.ail", "line_start": 43, "col_start": 13, "line_end": 43, "col_end": 57 }, "olds": ["account.balance"] }
    ],
    "span": { "file": "bank.ail", "line_start": 41, "col_start": 1, "line_end": 44, "col_end": 1 }
  }
}
```

**`contracts` 条目**：`{requires: [expr_entry], ensures: [expr_entry], span}`。
- `requires` / `ensures`（可选数组）：对应 §20 `requires { expr* }` / `ensures { expr* }` 块内各表达式，**按源码顺序**。
- `span`（必填）：整个契约区段的源码定位（§69.1）——即签名与函数体之间**全部契约子句**（`requires`/`ensures`/`effects` 块，§27 `contract*` 产生式 line 1480-1481）的**连续外包络**。`effects` 块虽其内容不进 `contracts`（见 §4），但其源码区间**计入 `span`**——`span` 纯为定位器（标记「此函数的契约区在源码何处」），非「contracts 数据所覆盖的块」。

**`expr_entry`**：`{expr, span, olds?}`。
- `expr`（必填，string）：契约表达式的**原始字符串**（与 §15.4 `constraint` 谓词字段 line 1350 同风格——如 `["value >= 0", "value <= 150"]`）。
- `span`（必填）：该表达式的源码定位。
- `olds`（可选，string[]）：该 `ensures` 表达式中**每个 `old(e)` 的实参 `e` 的原始源码字符串**，**按源码顺序、不去重**（与 `expr` 同为原始字符串表示）。例：`old(account.balance)` → `["account.balance"]`；`old(f(x))` → `["f(x)"]`（§7#3 允许 `old()` 调 pure fn）；同一 ensures 含 `old(x)` 与 `old(y)` → `["x", "y"]`；嵌套 `old(old(x))` → 外层实参 `"old(x)"`、内层实参 `"x"` 各计一项 → `["old(x)", "x"]`。仅 `ensures` 条目可能有；`requires` 无 `old()`（§20：`old()` 仅后置条件合法，line 990）。**取值规则钉死为「实参源码串」**（非「变量名」）——使 Tier 3 消费者可机器可靠解析 `old()` 实参，无需重解析 `expr`（兑现 §6 机器可读承诺）。

**`contracts` 字段的存在规则**：`contracts` 当且仅当函数含**至少一个非空 `requires`/`ensures` 块**（即至少一条布尔断言）时出现——
- 仅有 `effects { }` 块（如 §75 `login` 只有 `effects { database.read }`、无 requires/ensures——常见形态）、空 `requires { }`/`ensures { }` 块（`expr*` 允许零表达式）、或无任何契约块的函数 → **省略 `contracts` 字段**（同 RFC 0001 `examples[]` 的 absent-if-none 先例，保 `.ailmeta` 紧凑）。
- 出现的 `contracts` 内，`requires`/`ensures` 子数组**仅在非空时写该键**：函数只有 ensures 无 requires → 仅写 `ensures` 键、**省略 `requires` 键**（不用空 `[]`）。

**表示粒度 = 原始表达式字符串**（非规范化 AST）。理由：(a) 与 §15.4 `constraint` 谓词字段一致，保 schema 风格统一；(b) 最小实现，无需定义表达式 AST schema；(c) 规范化 AST 留 Tier 3 side-car 真正需要传播/求解时引入（见 §9 备选）。

**schema_version 协调**：0.2.0 → 0.3.0 是一次**向后兼容**的 semver minor bump——`contracts` 为可选字段，旧 reader 忽略之即可。先例：§88 #4/#5（line 3318-3319）在深审 VI 中以**可选字段**形式加 `declarations[]` 和 per-entry `span`，未破坏冻结状态。RFC 0001 的 `examples[]` 与本 RFC 的 `contracts` 共享这同一次 bump，避免每个特性单独 bump 导致 schema 版本碎片化。

---

## 4. 语义

- **元数据镜像，非新执行语义**：`contracts` 是 §20 运行时断言在 `.ailmeta` 的**只读镜像**。它**不改变**契约的运行时行为——`requires` 仍于函数入口断言、`ensures` 仍于出口断言（含 `old()` 快照，§85 #5 要求 `borrow_mut` 接收者以便调用者观察变更）、违约仍触发 panic（§89 #3 确定性栈展开 + Drop）。
- **纯描述性**：`contracts` 字段的存在与否不影响编译、不影响代码生成、不影响运行时契约检查。它是给 AI / 外部验证器 / 文档生成器消费的元数据。
- **`old()` 语义保留**：`olds[]` 记录的是每个 `old(e)` 实参的**原始源码串**（快照点表达式，取值规则见 §3），其语义（「调用前的值」）仍由 §20 line 990 定义；本 RFC 只把「这个 ensures 引用了哪些 old 快照点、各自快照了什么表达式」结构化，便于 Tier 3 传播消费（见 §6）。
- **`effects` 块不进 `contracts`**：§20 的第三种契约块 `effects { }` 是**效果注解**（约束推断集），其内容已由 `effects[]` 字段（§24 line 1342）输出，不重复进 `contracts`。`contracts` 只记 `requires`/`ensures` 两类**布尔断言**。**注**：不进 `contracts` 的是 `effects` 块的**数据内容**；其源码区间仍计入 `contracts.span` 的外包络（见 §3）。

---

## 5. 与现有机制的衔接（四槽位厘清）

一个 `function` 条目现在可携带四种「意图表达」元数据，层次分明、互不冲突：

| 槽位 | 来源 | 表达 | 机器验证 | 本 RFC |
|---|---|---|---|---|
| `examples[]` | RFC 0001 doctest | 具体输入→输出（exists） | 执行（`ail test`） | 已定 |
| **`contracts`** | §20 `requires`/`ensures` | 函数级前置/后置断言（for-all） | 运行时断言（违约 panic） | **本 RFC 新增** |
| `constraint`（在 `input[]`/`output` 类型上） | §15.4 `constraint` 声明 | 类型级谓词（refinement） | 构造时检查（违约 `ConstraintViolation`） | 已存在（line 1350） |
| `meaning`（在 `input[]`/`output`/`field`） | §15.3 `meaning` 声明 | 自然语言语义 | 仅验证存在与结构 | 已存在（line 533） |

**关键区分**：
- `contracts`（**函数级**，运行时断言）vs `constraint`（**类型级**，构造时谓词）：前者约束「函数调用前后的状态关系」，后者约束「类型的合法取值域」。一个函数可同时有两者（如 `withdraw` 有 `requires account.balance >= amount` 契约 + `amount: int where value > 0` 的 constraint 类型）。
- `contracts` vs `meaning`：前者是**机器可检查**的布尔表达式，后者是**自然语言**描述（编译器只验证其存在，不验证其正确性，§15.3 关键限制）。两者互补：`meaning` 给 AI 语义提示，`contracts` 给机器可验证事实。
- **§75 升级**：AI 分析器（line 3036）原本从源码「数据源」读 `requires`/`ensures` 构建知识图谱；本 RFC 后，它可直接从 `.ailmeta` 的 `contracts` 字段读——**消费路径从「源码」迁移到「元数据」**，与其他所有语义槽位一致。

---

## 6. 为跨函数契约机械传播备料（不实现引擎）

研究存档 §4 #4 把「跨函数契约机械传播」列为**核心设计指令**——让 callee 的 postcondition 成一等公民、可机器传递（堵 DAFNYCOMP 3.69% 组合性 gap）。

**本 RFC 的定位：备料，不传播。**
- `ensures` 的 `old()` 快照点（`olds[]`）与纯布尔表达式（`expr`）已结构化记录，使 Tier 3 side-car 验证器能**机器读取 `functions[]` 条目中的 callee 的 postcondition**（trait/interface 方法经 `sigs[]` 的契约输出留 v0.4+，见 §7.1 / §10 #7）。
- 但**传播逻辑本身**（caller 如何从 callee 的 `ensures` 推导出自己的可用信息）属 Tier 3 SMT side-car（研究存档 §3，opt-in、永不作默认 typechecker）。本 RFC 不实现传播引擎、不定义验证器协议。
- 即：1c 让契约**可被外部消费**（数据就位），传播**如何发生**留给 Tier 3。

**为何不在此实现传播**：(a) 传播依赖未实现的验证器基建；(b) 研究存档停止规则要求核心类型系统保持 decidable，传播引擎属 opt-in 上层；(c) 1c 作为「基础」先让数据可用，传播作为「上层」在 Tier 3 消费这些数据——分层清晰。

---

## 7. 限制（首版 v0.3.0）

1. **输出范围限于 `functions[]` 条目**：`contracts` 只附在 `function` 条目上——
   - **IN**：自由函数 + **struct 方法**。struct 方法为内联 `fn` item（§15.5 / §27 line 1486 `struct := … "{" (field | fn)* "}"`）；§24 struct type 条目仅含 `fields[]`、无方法容器，故 struct 方法作为 `fn` 序列化为 `functions[]` 条目（与自由函数同形）→ `contracts` 覆盖。
   - **OUT（v0.4+）**：
     - **trait / interface 方法**：§24 type 条目 `interface`(+`sigs[]`) / `trait`(+`sigs[]`) 将其方法序列化为 type 条目下的 `sigs[]`（**非** `functions[]` 条目）。interface 方法为 `fn_sig`（抽象、无 body，§27 line 1489）；trait 方法可带默认 body（§27 line 1490 `(fn_sig | fn)*`、§86 #6）。`sigs[]` 条目是否/如何附 `contracts` 字段、trait 默认方法 `fn` 的落点，**v0.2.1 §24 未定义** → 留 v0.4+（待 `sigs[]` 表示模型明确）。
     - **`agent`/`tool`/`actor`/`server`/`task`**：在 `declarations[]`（§24 line 1354，§88 #4），其契约语义在 v0.2.1 尚未完全定义 → 留 v0.4+。
2. **输出范围 = item 级 `public` 闸门（单一维度，与 RFC 0001 `examples[]` 同一判据）**——`contracts` 进不进 `.ailmeta` **仅由可见性决定**：
   - **`public` 闸门（与 RFC 0001 `examples[]` 共享同一 item 级可见性判据）**：仅 `public` 函数的 `contracts` 进 `.ailmeta`；非 `public` 函数的契约仍存在于源码、运行时仍检查，但**不入 `.ailmeta`**（内部实现细节，§91 默认 `private`）。闸门对 `contracts` 与 `examples[]` **一致放行/阻断**——杜绝**闸门层**的非对称（不会出现 `contracts` 因 `public` 过闸入 meta、`examples[]` 却被可见性闸门拦下，反之亦然）。**但二字段的实际出现各自独立**遵循 absent-if-none（§3）：`contracts` 当且仅当函数含非空 requires/ensures 块、`examples[]` 当且仅当函数有 doctest——故一个有契约无 doctest 的 `public` 函数，其条目可有 `contracts` 无 `examples[]`（此**数据层**非对称合法、由内容驱动，非闸门所致；闸门只保证「过闸资格」对称，不保证「字段共存」）。
   - **effect/async 不影响输出（与 RFC 0001 `examples[]` 在输出层一致）**：`contracts` 是**纯声明性元数据**（无执行概念），effect-ful 函数（`database.write`/`network`/含 `unsafe` 块/FFI）、async 函数的契约与 pure 函数**同样随 `public` 输出**。这与 RFC 0001 `examples[]` 在**输出层**行为一致——RFC 0001 §7.1/§7.3 中 effect-ful/async **`public`** 函数的 doctest **仍入 `.ailmeta`**（文档价值），其 `skip` 仅指**执行**（`ail test` 标记 `[skipped: needs fixture]` / `[skipped: async harness TBD]` 不跑；§7.1「`public` 且 skip：入 meta + skip 执行」）。**contracts 与 examples 的真实差异仅在执行层**：doctest 有执行模型，故 effect-ful/async 函数的 doctest 执行被 skip（`examples[]` 仍输出）；contracts 无执行概念，故无 skip 机制、规范一律随 `public` 输出。契约是函数的**规范声明**，无论函数能否被测试执行，其规范都应进 `.ailmeta` 供 AI 读取。
3. **契约表达式必须 `pure`**：`requires`/`ensures` 表达式不得有副作用/IO（同 §15.4 `constraint` 谓词规则，§92 #4 line 548——`old()` 是只读快照、可调用 `pure fn`，禁写/禁 IO）。编译器在 §73 typecheck 阶段校验。
4. **`invariant` 不引入**：v0.2.1 无 `invariant`（grep 零命中，只有 `requires`/`ensures`/`effects`）。struct/数据结构不变式（如平衡树平衡谓词）留 v0.4——本 RFC 只元数据化现有两种契约块。
5. **契约违约 panic 符号另案**：当前契约违约是普通 `assert` 风格 panic（§89 #3），**无独立 panic 符号**（§34 `panic` 符号表 line 1758-1764 只有 `ConstraintViolation` 来自 §15.4 constraint 构造子；§17 错误模型 line 736 的 panic 原因 prose 亦仅列 `ConstraintViolation`，无契约违约专属符号）。是否给契约违约引入独立符号（如 `ContractViolation`，区分 requires/ensures 违约）是 §17 增强，**不在本 RFC**（冻结规范另案决议）。

---

## 8. 诊断与容错

- **契约表达式编译期检查**：契约块内的表达式经 §70/§71（lex+parse）→ §73（typecheck，含 pure 校验）管线（与正式代码、与 RFC 0001 doctest body 同一管线）。类型错误、非 pure 表达式、非法 `old()`（如出现在 `requires`）均照常报错。
- **`.ailmeta` 输出错误不阻断编译**：若 `contracts` 字段序列化失败（如 span 缺失），降级为 warn（契约仍存在于源码、运行时仍检查），不阻断编译——`contracts` 是元数据，其输出完整性不影响代码正确性。
- **诊断 span**：契约表达式的 span 精确到源码字符位置（同 §69.1），便于 AI 闭环修复「契约与实现不符」。

---

## 9. 备选方案（及为何不选）

- **A. 规范化 AST 表示**（`expr` 存结构化 AST 而非字符串）：更利于机器消费/传播，但 (a) 需定义完整表达式 AST schema（重）；(b) 与 §15.4 `constraint` 谓词字符串字段不一致（schema 风格分裂）；(c) Tier 3 side-car 真正需要传播/求解时再引入更合时宜。**不选**——首版用原始字符串，AST 留 Tier 3（见 §10 #4）。
- **B. 自动传播引擎**（编译器从 `.ailmeta` 读 callee `ensures` 推导 caller）：直接实现研究存档 §4 #4 核心指令，但 (a) 依赖未实现的验证器基建；(b) 越本 RFC「备料」范围；(c) 违研究存档分层（传播属 opt-in Tier 3）。**不选**——1c 备料，Tier 3 传播。
- **C. 引入 `invariant`**：契约语言更完整（struct 不变式），但 v0.2.1 无此概念，引入即新增语法形式、扩 RFC 范围。**不选**——留 v0.4。
- **D. 契约违约 panic 符号**（`ContractViolation`）：诊断更精确，但属 §17 panic 模型增强，非 `.ailmeta` 输出问题。**不选**——冻结规范另案。

---

## 10. 未决问题（待 review）

1. **契约违约 panic 符号**：是否给 `requires`/`ensures` 违约引入独立符号（如 `ContractViolation`，与 §15.4 `ConstraintViolation` 区分）？倾向：另案 §17 增强，不在此 RFC。
2. **struct `invariant`（v0.4）**：数据结构不变式（平衡树、有序表）如何进 `.ailmeta`？是否复用 `contracts` 槽位扩展？
3. **契约 → PBT 属性机械推导接口（v0.4，衔接 1b）**：`ensures`→PBT 断言、`requires`→生成器守卫、`constraint`→类型生成器不变式——三者的机械推导协议（研究存档 §9.1 互补）留 PBT RFC。
4. **规范化 AST 引入时机**：当 Tier 3 side-car 需要真正传播/求解 `ensures` 时，`expr` 是否从字符串升级为 AST？触发条件 = 第一个消费 `contracts` 的验证器出现。
5. **`declarations[]`（agent/tool/actor/server/task）契约输出（v0.4+）**：这些声明的契约语义在 v0.2.1 尚未完全定义，其 `.ailmeta` 输出待对应契约语义明确后扩展。
6. **非 `public` 函数契约与 Tier 3 传播**：§7.2 的 `public` 闸门使 private 函数契约不入 `.ailmeta`；但 Tier 3 跨函数传播（研究存档 §4 #4）可能需要 private callee 的 `ensures`（public caller 调 private helper 时）。届时是否为 opt-in side-car 扩展一个含 private 契约的**数据视图**（非默认 `.ailmeta`，保公开面信噪比），留 Tier 3 RFC。
7. **trait/interface 方法契约 × Tier 3 传播**：§27 `fn_sig`（line 1475）含 `contract*`，即 trait/interface 方法签名语法上已可携带 `requires`/`ensures`（多态调用的核心契约源——trait 方法的 `ensures` 约束所有实现者），但 §7.1 OUT 将其 `.ailmeta` 输出推迟到 v0.4+（`sigs[]` 表示模型未定）。v0.4 `sigs[]` 契约表示明确前，Tier 3 跨函数传播无法机器读取 trait 方法 postcondition——跨 trait 边界的传播补料路径待 v0.4。

---

## 11. 不触动 v0.2.1（合规声明）

- **不改 §1–§94 任何条款**（含 §20 契约语法、§24 `.ailmeta` schema、§15.3/§15.4、§17 panic 模型——均只读引用）。
- **不增关键字**（56 不变；`requires`/`ensures`/`effects` 是 v0.2.1 **已有**的硬关键字，§9 line 384「AI 语义」类别、**已计入 56**，本 RFC 零新词法）。
- **不改 110 项决议**。
- 契约进 `.ailmeta` 是 **v0.3+ 特性**；本 RFC 为**提案**，经 review + 批准后进 v0.3 规范（届时 AILANG.md §24 schema 新增 `contracts` 字段描述、`schema_version` bump 0.2.0 → 0.3.0）。
- 本 RFC 本身不修改 `docs/AILANG.md`（同 RFC 0001 §12 定位）。

**与 RFC 0001 的 schema 协调**：RFC 0001 已声明 `schema_version` v0.2 → v0.3（精确为 0.2.0 → 0.3.0）。本 RFC 的 `contracts` 字段与 RFC 0001 的 `examples[]` 共享这同一次 bump——一个 0.3.0 schema 同时携带 `examples[]`（RFC 0001）+ `contracts`（本 RFC），向后兼容（两者皆可选字段）。
