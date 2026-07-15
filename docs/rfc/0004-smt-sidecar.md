# RFC 0004 · Tier 3 SMT 可选验证器（side-car）—— 逻辑证明层

| | |
|---|---|
| **状态** | 草案（Draft v5）—— 待 review（收敛轨迹：v1=0H/13M/10L（9 维）→ v2=0H/4M/15L（9 维，19 条确认聚 11 根因）→ v3=0H/3M/6L（9 维，9 条确认聚 8 根因，[A]–[K] 11 修复全验落地干净）→ v4=0H/2M/2L（9 维，8 原始→确认 4 否决 4，聚 2 根因，[F1]/[F3] 等 8 修复全验落地干净）→ v5 修复；多轮对抗式 workflow 验证目标 0H/0M/0L，对齐 RFC 0001 v6 / 0002 v8 / 0003 v5）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94、56 关键字、110 决议不变）|
| **日期** | 2026-07-15 |
| **承接** | [`docs/research/ai-friendliness-2026-07.md`](../research/ai-friendliness-2026-07.md) §3 Tier 3（「SMT 验证器作可选 side-car（SPARK Gold 式 opt-in），永不作默认 typechecker；外部验证器消费 `.ailmeta`；初期不烤进 `ailc`」）+ §1（DAFNYCOMP 3.69%）+ §2（停止规则）+ §4#3（反 spec-hacking 58%→31%）+ §4#4（跨函数契约机械传播）+ §6①（结构化验证器协议，非 Z3 opaque）/ §6③（SPARK 分层）/ §9.2（SMT 降级）；[RFC 0002](./0002-contracts.md) §6（传播引擎 = Tier 3，备料不传播）/ §10#6（非 public 函数契约 × Tier 3）/ §10#7（trait 契约 × Tier 3 → v0.4）；[RFC 0003](./0003-pbt-fuzzer.md) line 22/24（Tier 3 = for-all 逻辑证明，证 ∀x·requires(x)⇒ensures(x)）/ §9#4（自动 SMT discharge = Tier 3）|

---

## 1. 动机

研究存档 §1 锚定 AILang 的立身命题 = **DAFNYCOMP 3.69%**：LLM 能验证**单函数**（60–90%）却验证不了**多函数系统**（<7%）——根因是组合性。验证 Tier 阶梯（研究 §3 line 36-42）把压缩这条 gap 的能力分到五层：1a doctest（RFC 0001）、1b PBT+fuzzer（RFC 0003）、1c 契约（RFC 0002）、2 窄 decidable refinement（未启动）、**3 SMT 作可选 side-car（本 RFC）**。前三层把「意图」结构化进 `.ailmeta`（`examples[]`/`contracts`/`properties[]`），但都停留在**运行期断言 / 经验检查**——无一做**逻辑证明**。Tier 3 是阶梯顶端的**可选**逻辑证明层。

**冻结缝隙（Tier 3 填补的精确位置）**。规范两处把 SMT 明确「留 v0.3+」，本 RFC 填之：
- **§82 #3（line 3204，「遗留（工程实现层）」第 3 项）**：「Constraint / 契约证明的实现深度：语言层已分级锁定（§83 #3，v0.2 仅断言）；**实现层待定 v0.3+ 是否集成 SMT**。」——把 SMT 归为**工程实现层开放问题，非语言决议**。
- **§83 #3（line 3216）**：「自动 SMT 证明留 v0.3+。」
- **§20 line 990**：「v0.2：运行期断言。**自动证明不在范围（§83 #3）**。」
- **§15.4 line 546**：「完整 refinement 证明不在 v0.2（§83 #3）。」

关键：§82 #3 把 SMT 归为**工程实现层** ⇒ Tier 3 可**完全不触动 §1–§94 的语言语义**落地——以「可选**外部**验证器消费 `.ailmeta`」的形态，而非「默认 typechecker」的形态（研究 §3 line 42：**永不作默认 typechecker；外部验证器消费 `.ailmeta`；初期不烤进 `ailc`**）。

**Tier 阶梯全表**（承接 RFC 0001 §1 / RFC 0002 §1 / RFC 0003 §1 层级表，补全逻辑证明顶层）：

| 层 | 表达力 | 形式 | AILang 载体 | `.ailmeta` 槽位 | 验证方式 |
|---|---|---|---|---|---|
| **doctest**（RFC 0001） | exists（具体实例） | `///` 内可运行块 | DocComment | `examples[]` | 执行（`ail test`） |
| **constraint**（§15.4） | 类型级谓词（refinement） | `constraint T { expr* }` | 独立声明 | `constraint` 字段（line 1350） | 构造时运行期断言 |
| **契约**（RFC 0002） | for-all（运行时前/后置断言） | `requires`/`ensures` 块（§20） | 契约块 | `contracts` | 入口/出口运行期断言 |
| **PBT / fuzz**（RFC 0003） | for-all（经验量化属性） | `test` 块 + `std.testing` generator | `test` + `std.testing` | `properties[]` | N 次随机输入执行 |
| **SMT side-car（本 RFC）** | **for-all（逻辑证明）** | **外部验证器消费 `contracts`/`constraint`/`properties`** | **`ail verify` + `.ailmeta.verify`** | **`verification`（compact）+ `.ailmeta.verify`** | **SMT 求解器静态证明** |

**for-all 的两层分裂**（承 RFC 0003 line 24）：**契约 = 逻辑 for-all**（本 RFC 用 SMT **证明** `∀x·requires(x)⇒ensures(x)`）、**PBT = 经验 for-all**（RFC 0003 在生成 `x` 上**检查** `ensures`）。`ensures` 是共享桥——同一表达式串（RFC 0002 `contracts.ensures[].expr`）喂两个消费者（Tier 3 逻辑证明 + RFC 0003 经验检查）。本 RFC 兑现 RFC 0003 line 22/24 的 Tier 3 移交。

**跨函数传播 = DAFNYCOMP gap-closer**（研究 §4 #4 line 53）。前三层是孤立检查（每个函数自己）。Tier 3 引入**跨函数契约机械传播引擎**（§5，level≥2）：被调函数的 `ensures` 后置条件成为调用点的一等、可机械传播的假设——使 caller 不必「重新发现」callee 的保证。这是把单函数证明能力组合成系统能力、压窄 3.69% gap 的核心机制。RFC 0002 §6「备料不传播」把 `ensures`/`old()` 结构化进 `.ailmeta` 正是为这一步**备料**；本 RFC 兑现传播引擎。

---

## 2. 设计目标

1. **永不默认（分离公理）**—— `ailc` 的编译成功与验证结果**完全独立**。`ail verify` 是 `ailc` 输出的下游消费者（Mode A：消费 `.ailmeta`+`.ailmeta.verify` 两工件、不重解析源码；Mode B 兜底：外加自带前端重解析 `.ail` 源，§3.2）。即使 `ail verify` 报告 48 个 failed VC，`ail build` 产出**相同**的二进制。验证结果只影响 (a) `ail verify` 自身退出码、(b) `.ailmeta.verify` 伴随工件、(c) 主 `.ailmeta` 的 compact `verification` marker（`--write-meta` opt-in 写入）。**`ail build` 管线不调用验证器**（研究 §3 line 42「永不作默认 typechecker」、RFC 0002 §6 line 125/128）。停止规则（研究 §2 line 20-28）：Tier 3 **正因为**跨过 SMT 停止线（启发式 solver / 超多项式 / 标注:代码比），才必须留在 opt-in 默认路径之外——分离公理是其形式化。
2. **solver-agnostic**—— 后端抽象到 **SMT-LIB v2** 标准 interchange，任何说 SMT-LIB 的 solver（Z3 / CVC5 / Alt-Ergo）均可（`--solver` 配置，默认 z3）。换 solver 只换后端、不动 VC IR；用户可按理论片段选最强 backend 缓解「SMT 漂移」（研究 §4 #2 line 51）。
3. **结构化结果，非 opaque**—— solver 结果映射为**六态**结构化分类（proved / failed-with-counterexample / unknown+reason / timeout / unsupported-theory / solver-error），反例**按参数名结构化**、非 SMT model 原文。直击研究 §9.2 line 163「杜绝 Z3 不透明 `unknown`」与 §6① line 80「结构化验证器协议喂 self-repair」。
4. **反 spec-hacking（架构硬要求）**—— 验证器**必配测试夹具**（研究 §4 #3 line 52：RLVR 模型 `ensures true`+`assume false` 骗 solver，58%→过滤后 31% honest）。trivial-spec 检测 + fixture-coverage gate 强制验证**不替代** Tier 1a/1b（§10）。
5. **零新关键字 / 零新文法产生式**—— 外部 side-car 的全部表面 = CLI（`ail verify` 子命令 + flags）+ `.ailmeta`/`.ailmeta.verify` 字段。`verify`/`proof`/`solver`/`refine`/`invariant`/`ghost`/`level` 均**非关键字**（§9 line 373-393 的 56 不含其一，grep 确认）；`loop invariant` 在规范零命中。源码内任何注解（如 loop invariant）**显式推迟 v0.4**。**不触动 §27 任何产生式**。
6. **兑现全部显式 handoff**—— RFC 0002 §6（传播引擎）/§10#6（非 public 函数契约）/§10#7（trait 契约 → v0.4）；RFC 0003 line 22/24/§9#4（逻辑 for-all 证明 + SMT discharge）。每条在节内可追溯（§12 handoff 表）。

---

## 3. 架构与集成

### 3.1 `ail verify` = 独立 opt-in 子命令

`ail verify` 是与 `ail build`/`ail test`/`ail doc` 并列的**独立子命令**，**不**进入 `ailc` 编译路径、**不**进入 `ail build` 管线、**不**进入 `ail test` 执行管线。这最忠实地体现 §82 #3「工程实现层」定性 + 研究 §3 line 42「初期不烤进 `ailc`」+「永不作默认 typechecker」。

| 命令 | 行为 |
|---|---|
| `ail verify` | 默认 level 1（per-function）跑验证，文本摘要输出 |
| `ail verify --level 0\|1\|2\|3` | 验证等级（§9.2；默认 1，`0`=no-op） |
| `ail verify --solver z3\|cvc5\|alt-ergo` | SMT 后端（默认 `z3`） |
| `ail verify --timeout <ms>` | 单 VC 超时预算（默认如 5000ms） |
| `ail verify --json` | 结构化结果 JSON 输出（喂 self-repair / CI） |
| `ail verify --strict` | `unknown`/`timeout`/`solver-error` 也算失败（默认仅 `failed` 算，§9.3） |
| `ail verify --require-fixture` | 强制 fixture-coverage gate（§10） |
| `ail verify --write-meta` | 把 compact `verification` marker 写回主 `.ailmeta`（§8） |
| `ailc --emit verify-bundle` | `ailc` opt-in 产出 `.ailmeta.verify` 伴随工件（§3.2） |

`verify` 是子命令 ident（如 `build`/`test`/`doc`），非关键字；`--level`/`--solver`/`--timeout`/`--json`/`--strict`/`--require-fixture`/`--write-meta`/`--emit` 为 CLI flag。

### 3.2 数据流：`ailc` 备料，`ail verify` 消费

**三层分工**（承 RFC 0002 §6「备料不传播」分裂的 Tooling 化）：

1. `ailc` 默认产出 `.ailmeta`（public-gated 语义元数据，§24 line 1330-1361，含 `contracts`/`constraint`/`properties`/`examples`）。**properties[]**（RFC 0003，emission 闸门 = public，§9.1）与 **examples[]**（RFC 0001，public-gated）皆在主 `.ailmeta`——验证器读主 `.ailmeta` 获取 public 函数的 property/doctest 夹具存在性。
2. `ailc --emit verify-bundle`（opt-in flag）额外产出 **`.ailmeta.verify` 伴随工件**（与 `.ailmeta` 同目录的**独立文件**，非 §24 `.ailmeta` 本体）——**自包含的验证输入**，含 **public + private** 全函数的 contracts、constraint 谓词、**函数签名类型**（input[]/output，VC SMT sort 派生所需）、**调用图**、每函数的**受限 body IR**（带 span）、**归属手写属性的断言 IR**（handwritten property VC 生成所需，§4.1）。**不携带** examples[]/properties[] **元数据**（二者 public-only、读主 `.ailmeta` 即可）；但携带手写属性的**断言 IR**（test 块断言体不在主 `.ailmeta` properties[] 元数据内、且非函数体，须由 verify-bundle 提供，否则 Mode A 无法生成 handwritten property VC）。
3. `ail verify` 读 `.ailmeta` + `.ailmeta.verify` → 生成 VC（§4）→ 传播（§5）→ SMT 求解（§6）→ emit 结构化结果（§7）。

```
source.ail ─► [ailc] ─► .ailmeta            （默认, public-gated, §24 line 1330-1361）
                     ↘
                       .ailmeta.verify       （opt-in 伴随, 全 contracts[含 private] + 签名类型 + 调用图 + 受限 body IR）
                                   │
                                   ▼
                          [ail verify] ◄── 读两者 ──► VC IR ──► SMT-LIB v2 ──► solver
                                   │
                                   ▼
                          结构化结果 (stdout --json) + (--write-meta) compact verification → 主 .ailmeta
```

**Mode A（主）**：`ailc --emit verify-bundle` 已产出 `.ailmeta.verify`，`ail verify` 为纯消费者（不重解析源码）。
**Mode B（兜底）**：未产出 `.ailmeta.verify` 时，`ail verify` 可回退到自带前端「重解析 `.ail` 源 + 读 `.ailmeta`」模式。Mode A 为主、Mode B 为兜底；两者产出相同的结构化结果。

**为何需要 body IR**：function-contract VC 必须有函数体（证 body 在 `requires` 下确能建立 `ensures`，§4）——光有 contracts 不够。**为何需要手写属性断言 IR**：handwritten property VC（source:"test"，§4.1）需要该 test 块的断言体作 `holds`——test 块是顶级 §27 item（非函数体、不在主 `.ailmeta` properties[] 元数据中），须由 verify-bundle 携带其断言 IR，否则 Mode A 无法生成此类 VC。**为何需要 private 函数 contracts**：兑现 RFC 0002 §10#6（line 172）——跨函数传播时 public caller 调 private helper，传播引擎需要 private callee 的 `ensures`；但 RFC 0002 §7#2 的 public 闸门使 private 契约**不入默认 `.ailmeta`**（保公开面信噪比），`.ailmeta.verify` 伴随工件正是 RFC 0002 §10#6 预期的「opt-in 含 private 契约的数据视图」。

**受限 body IR 是设计契约，非最终序列化格式**：本 RFC 定义「`.ailmeta.verify` 须携带哪些信息」——表达式 DAG（运算/调用/字面量/变量）、CFG 节点（控制流）、调用边（callee 标识 + span）、每个节点 span——**不规定**具体序列化（JSON/text/binary）。此信息需求契约覆盖两类「可计算内容」：(a) 函数体（function-contract VC 所需）、(b) 归属手写属性的断言体（handwritten property VC 所需，§4.1）。理由：ailc 本身 TBD（仓库尚无编译器），其内部 IR 待 ailc 落地后定；本 RFC 只锁死验证器对这两类「可计算内容」的**信息需求契约**。

### 3.3 解耦边界

`ail verify` 是 `ailc` 输出的下游消费者（Mode A：`.ailmeta` + `.ailmeta.verify` 两工件、不重解析源码；Mode B 兜底：外加自带前端重解析 `.ail` 源，§3.2），**不**耦合 ailc 内部 HIR/IR 内存表示——只消费**序列化文本工件**（`.ailmeta` / `.ailmeta.verify` / `.ail` 源）。`ailc` 对验证结果**零反馈**（分离公理，§2 #1）。核心不变量——「不耦合 ailc 内部 HIR/IR 内存表示」——在两模式下均成立（Mode B 用验证器**自带前端**重解析，仍不读 ailc 内存 IR）。验证器复用：
- **§69.1 Span**（line 2759，已序列化进 `.ailmeta` per-entry span，§88 #5 line 3319）。
- **§69.3 Diagnostic 结构**（line 2772-2782，`{severity, code, message, span, notes}` 形）——但走**独立通道**（§11），非编译期管线。

---

## 4. VC（验证条件）生成

### 4.1 四类 VC

| VC 种类 | 形式 | 输入源 | 冻结依据 |
|---|---|---|---|
| **function-contract**（level≥1） | `∀ σ_pre · R_f(σ_pre) ⇒ wp(B_f, E_f(σ_pre, σ_post))` | `contracts.requires[]`/`ensures[]`（RFC 0002）+ body IR | §20 line 974-994；RFC 0003 line 24 |
| **constraint-invariant**（level≥1） | 在**合格函数内**每个 `T(expr)` 构造点：`P_T(expr_val)` | type `constraint` 谓词数组（§24 line 1350）+ 构造点 | §15.4 line 546-547；§92 #2 line 3385 |
| **property**（level≥1） | `∀ inputs · requires(inputs) ⇒ holds(p, inputs)`（逻辑版；derived 与 handwritten 两源见下） | `properties[]`（RFC 0003） | RFC 0003 line 22 |
| **call-site-precondition**（level≥2） | 在调用点 `f(args)`：`R_f(args)` | callee `contracts.requires[]` + 调用点上下文 | RFC 0002 §6 line 125 |

记号：`σ_pre`=预态（参数 + `old()` 快照目标）、`σ_post`=后态、`wp`=最弱前置条件、`R_f`=callee `requires`、`E_f`=callee `ensures`、`B_f`=callee body、`P_T`=constraint 类型 T 的谓词、`p`=property。

**VC 作用域（合格函数）**：所有四类 VC 仅对**合格函数**（§12.3 闸门层：`public` 或 `private` **且** `pure`）生成。非合格（非 pure）函数**不生成任何 VC**、不进 `results[]`、不计入 `summary.total`（§7）——其内的 `T(expr)` 构造点安全网仍是运行期 `ConstraintViolation`（§15.4 / §34 / §11）。此「合格函数内」作用域使 constraint-invariant VC 的「每个构造点」与 §12.3 pure 闸门不冲突（承 RFC 0003 §9.2 skip 先例：非 pure fn 的推导属性入 `.ailmeta` 但标 `[skipped]` 不执行；Tier 3 同此——非 pure fn 不生成 VC）。

**function-contract VC 的精确语义**：它**不是**直接证 `∀x·requires⇒ensures`（那是 spec），而是**证 body 是该 spec 的实现**——即「在 `requires` 成立的预态下，body 的最弱前置条件蕴含 `ensures`」。`old(e)`（§20 line 990 调用前快照）在 VC 里绑定为 `σ_pre` 的分量。这正确捕获了「实现符合规范」。

**constraint-invariant VC 的精确语义**：运行期 `T(expr)` 构造算符（§15.4 line 547「`let a: Age = Age(n)`」、§92 #2 line 3385）是运行期断言谓词，违约 panic `ConstraintViolation`（§34 line 1764、§92 #9 line 3392）。Tier 3 的 constraint-invariant VC = **静态证明这个运行期断言永不违约**——在每个 `T(expr)` 构造点证明 `P_T(expr_val)`。注意：§15.4 line 546 已对**字面量**做编译期折叠检查（`let a: Age = 30` 编译期判、`let a: Age = -5` 编译错）；Tier 3 覆盖**运行期值**（变量/表达式）的构造点证明，补全 §15.4「完整 refinement 证明不在 v0.2（§83 #3）」的缺口。**`ConstraintViolation` 仍是运行期 panic**——Tier 3 只静态证明它不触发，**不改变其运行期行为**（§34 line 1764 不变）。

**property VC 的精确语义（两源 + 归并）**：RFC 0003 的 `properties[]` 有两源（RFC 0003 §5 line 124），Tier 3 对应两种 property VC：

- **derived property**（`source:"ensures"`）：复用本函数的 `contracts.requires` 作域约束、`holds`=该 `ensures`。其 VC 与该函数的 **function-contract VC 的 `ensures` 分量是同一义务**——故**归并入 function-contract VC、不单独 emit**（dedup，避免 §7 `summary.total` 双计同一义务）。
- **handwritten property**（`source:"test"`）：`requires(inputs)` **≡ true**（全域，匹配 generator 类型域——RFC 0003 §7 手写属性无 requires 守卫、runner 从 generator 抽值），`holds`=该 test 块断言。其 VC 是**独立的** per-function for-all，单独 emit（如 `reverse(reverse(x)) == x` 对所有 int `x`）。**多归属去重（按 test 块 span）**：一条多归属手写属性（RFC 0003 §5 line 131 归属规则——test 块 body 直接调用的每个 public fn 各得一条 source:"test" 条目，如 `test { assert(f(x)==g(x)) }` 同时归属 f 与 g）的 VC，其逻辑身份 = **该 test 块**（`holds` 与 `requires≡true` 均不随归属函数改变），故**按 test 块 §69.1 span 去重**（每条 source:"test" 条目携带其 test 块 span，RFC 0003 §5）：同一 test 块（多归属到多个 public fn，span 相同、holds 相同）只 emit **一条** VC、`summary.total` 计 1 次（与 derived 的「按逻辑义务去重」同口径，避免多归属夸大计数）；不同 test 块即便 string_lit 同名（span 不同、holds 可能不同）各 emit 一条（**避免同名碰撞丢义务**——§27 line 1502 `test := "test" string_lit block` 不强制 string_lit 模块内唯一，故用 span 而非名作键）。其 result 行 `subject.function` 取归属函数之一、`vc_summary` 标注多归属函数集。

**为何 handwritten 的 `requires≡true`**：手写属性意在 generator 全域（RFC 0003 §5 归属规则把 source:"test" 条目结构化进所归属 public fn 的 `properties[]`，但属性本身无 requires 字段）。若误把手写属性的 `requires` 绑到归属函数的 `contracts.requires`（function-contract VC 也用同函数 requires，易混淆），VC 被不当削弱——只证 requires 子域，违例子域外的失败被静默放过（under-verification 误报 `proved`）。`requires≡true` 消除此风险。property VC 是 RFC 0003 line 24「逻辑 for-all」的可操作化；level≥2 时手写属性 body 中调用的其他函数的 `ensures` 可作假设（更强），但 property VC 的基线属 level 1。

### 4.2 VC IR（中间表示）

VC 用**自定义 VC IR**（类 WhyML/Boogie/SPARK VC 语言），**非直接**生成 SMT-LIB。管线：source expr → **VC IR** → **SMT-LIB v2**（per-backend 翻译）→ solver。

VC IR 元素：**命名断言**（hypothesis，如 `h1: amount > 0`）、**作用域假设**（call-site 传播的 callee `ensures`，§5）、**goal**（要证的目标，如 function-contract 的 `ensures`）——每个带 **span**（§69.1）+ **溯源**（指向 `contracts.ensures[i]` / property / body 行 / `T(expr)` 构造点）。

**命名变量的反向绑定（model→结构化反例的机械保证）**：VC IR 的每个量化/自由变量携带**源码反向绑定**——参数名（顶层入参）+ 字段路径（struct，如 `account.balance`）+ 数组下标（如 `xs[2]`）。这使 solver 返回 `sat`+model 后，验证器能**机械**地把 model 变量值映射回 §7 `counterexample.inputs`（按参数名/字段路径/数组下标为 key），无需猜测。此反向绑定是「结构化反例、非 SMT model 原文」（§7 / 研究 §9.2 line 163）的实现机制，被声明为 VC IR 元素的必备属性（非实现细节）。struct/数组反例的 `inputs` 表示 = **嵌套对象 + 数组**（如 `{ "account": { "balance": 30 }, "xs": [1, 2, -3] }`），非扁平点号字符串——使任意嵌套类型的反例可精确表达。

**理论片段标注**：每个 VC IR 节点标注其涉及的理论片段（`linear-arithmetic` / `nonlinear-arithmetic` / `quantifier` / `uninterpreted-function` / `datatypes` / `array` 等）。此标注是 §6.2 把 solver `unknown` 映射为结构化 `reason category` 的输入（静态标注说明 VC 涉及哪个理论），与 solver 自身报告的 `:reason-unknown`（§6.2 映射表）共同确定 reason。

**为何 VC IR 而非直 SMT-LIB**：研究 §9.2 line 163 + §6① line 80「杜绝 Z3 opaque unknown」——直接喂 SMT-LIB 时 solver 返回 `unknown` 无法定位是哪个理论片段、哪个表达式。VC IR 给每个假设/义务命名 + span + 理论片段标注，使 solver 的 `unsat`/`sat`/`unknown` 能**映射回结构化结果**（§7：proved / counterexample / unknown-with-reason）。VC IR 的溯源使 counterexample 能精确指到失败的 `contracts.ensures[i]`（RFC 0002 §3 expr_entry）或 property，喂 self-repair。

---

## 5. 跨函数契约传播引擎（DAFNYCOMP gap-closer，level≥2）

### 5.1 Hoare 逻辑 + 过程摘要

传播引擎采用标准 **Hoare 逻辑 + 过程摘要（procedure summaries）**。验证 caller `g` 的 function-contract VC 时，对其 body 中每个调用点 `f(args)`：

- **Obligation（义务）**：证 callee 的 `requires[args]` 在调用点成立（caller 须建立 callee 前置条件）→ 产生 **call-site-precondition VC**（§4.1 第四类）。
- **Assumption（假设）**：调用后，callee 的 `ensures[args]`（`old()` 快照点绑定为**调用前**状态，§20 line 990）成为 caller VC 中的**可用假设**——callee 的后置条件机械传播为 caller 的一等信息。

**Soundness 前提（callee VC 须证毕）**：上述 Assumption Hoare-sound 的前提是 callee 自身的 function-contract VC 已证毕（即 callee body 在 `requires` 下确能建立 `ensures`）。若 callee 的 function-contract VC **未达 proved**——`unknown` / `timeout` / `unsupported-theory` / `solver-error` / `failed`——则其 `ensures` **不可**作 caller 的可靠假设。**降级规则**：callee VC 非 proved 时，依赖该 callee `ensures` 的 caller VC 判 **`unknown`**、`unknown_reason` = `unverified-callee:<f>`（标注哪个 callee 未证毕），而非 `proved`。这杜绝「基于未证/假前提判 caller proved」的不健全性。callee `failed`（有反例证其 body 不满足自身 `ensures`）更不可作假设——caller VC 同样降级 `unknown:unverified-callee:<f>`。模块内自底向上验证（叶函数先证）可最大化可达 proved 比例。**递归 SCC 例外**：上述 callee-须证毕 前提与降级规则**仅适用于当前递归 SCC 之外的 callee**；递归 SCC 内部调用边（直接/相互递归）的 callee ensures 作**归纳假设**（rule of consequence），豁免 callee-须证毕 前提——SCC 作为整体被验证，内部边假设自身契约是标准部分正确性手法（详见 §5.3）。

这正是 RFC 0002 §6（line 125/128）「备料不传播」推迟的传播逻辑：`contracts.ensures[]` 数据已由 RFC 0002 结构化进 `.ailmeta`/`.ailmeta.verify`，Tier 3 消费它做机械传播。

**为何这是 DAFNYCOMP gap-closer**：研究 §4 #4（line 53）+ §1（line 16）——LLM 验证单函数 60-90% 但系统 <7%，根因是组合性。被调函数的 `ensures` 成为一等、可机械传播，使 caller 不必「重新发现」callee 的保证——这是把单函数证明能力组合成系统能力、压窄 3.69% gap 的核心机制。

### 5.2 loop 处理（零新关键字 + 诚实声明）

**冻结事实**：v0.2.1 **无 `loop invariant`**（grep 零命中；`invariant` 仅在 §15.4 prose 指 refinement，非循环不变式）。`loop`/`while`/`for` 是硬关键字（§9 line 381），但**无不变式注解语法**。

**v0.3 决断**：
- **loop-free 函数**：function-contract VC 直接处理（body 是直线/分支代码）。
- **有界循环**（如 `for x in list` 遍历有限集合）：有限展开（bounded unrolling），把循环体展开到其有界上界。
- **unbounded 循环**（`while cond { … }` 无静态界）：function-contract VC 报 **`unsupported-theory`**（结构化降级，§6.2 / §7，非 opaque unknown）。

**「全覆盖」= Tier 3 架构全覆盖，非「所有函数可验证」**（诚实声明）：含 unbounded loop 的函数 v0.3 一律 `unsupported-theory`；传播引擎的 v0.3 实际可达域 = **loop-free + 有界循环**函数。这是 loop invariant 标注缺失下的诚实边界——不在 v0.3 引入新文法来谎称覆盖。

**v0.4+ 路线**：两路推进 unbounded loop 验证——(i) 谓词抽象 / 抽象解释**自动推断** loop invariant（无用户标注）；(ii) 若需用户标注，引入**上下文关键字** `invariant`（同 §9 line 393 `where`/`constraint` 上下文关键字先例，**非硬关键字，不入 56**）——这是新 RFC，不在本 RFC。loop invariant 标注引入新文法表面（违「零新文法」），故 v0.3 用「loop-free / 有界展开 + 否则 `unsupported-theory`」完全规避。

### 5.3 递归 / mutual recursion（归纳假设豁免）

- **递归 SCC 的归纳假设**：递归函数（直接自递归或相互递归 f↔g）按**强连通分量（SCC）**处理。在 SCC **内部**调用边上，callee 的契约（`requires`/`ensures`）作**归纳假设**（induction hypothesis）——**豁免** §5.1 的 callee-须证毕 前提与降级规则。这是标准部分正确性的 rule of consequence：证 SCC 内函数时，假设该 SCC 所有函数自身的契约为前提（归纳步骤），而非要求每个 callee 已独立 proved。若 SCC 内部某函数 body 确违反自身契约，其 VC 直接 `failed`（不依赖归纳假设）。
- **§5.1 降级规则的适用边界**：callee-须证毕 前提 + `unverified-callee:<f>` 降级 + 自底向上「叶函数先证」排序**仅适用于 SCC 之外的 callee**（非递归依赖）。SCC 无内部叶节点，故作为整体验证（内部用归纳假设、外部用传播规则）。递归深度不展开（传播引擎处理为图边，非调用树展开）。
- **部分正确性（partial correctness）**：v0.3 Tier 3 证明**部分正确性**——「若函数终止，则 `requires⇒ensures` 成立」，**假设终止**、不证终止。这与 unbounded loop 的处理**一致**（非终止循环函数 `unsupported-theory`，非 bogus 证）。**「假设终止」与「callee VC 证毕」是两件事**——部分正确性假设运行期终止（不证），递归边的归纳假设成立性由 SCC 整体证明保证（上一条），二者不可混淆。
- **终止性**（递归 well-founded 证明）**不在 v0.3**（§12 #1，跨停止规则——超多项式）。故递归函数的证明是部分正确性（假设终止），非全正确性。

---

## 6. SMT 后端抽象

### 6.1 solver-agnostic via SMT-LIB v2

后端抽象到 **SMT-LIB v2** 标准 interchange。任何说 SMT-LIB 的 solver（Z3 / CVC5 / Alt-Ergo）均可，默认 z3，`--solver` 可配。VC IR → SMT-LIB 是**唯一后端边界**——换 solver 只换后端翻译、不动 VC IR。用户可按理论片段选最强 backend 缓解 SMT 漂移（研究 §4 #2 line 51：不同 solver 对不同片段强弱不同）。

### 6.2 六态结构化降级（永不 panic、永不阻塞编译）

`ail verify` 是 spawn 外部 solver 子进程（z3/cvc5/alt-ergo）的 side-car（§3.1）。solver 子进程的所有可观测结果——包括**基础设施失败**——映射到**六态结构化分类**（§7 的 status），**永不 panic、永不阻塞 `ail build`**（分离公理 §2 #1）：

| solver 结果 | Tier 3 status | 附加信息 |
|---|---|---|
| `unsat` | `proved` | — |
| `sat`（+model） | `failed-with-counterexample` | 从 model 提**结构化反例**（按参数名/字段路径/数组下标映射，§4.2 反向绑定，非 SMT model 原文）|
| `unknown` | `unknown` | **reason category**（见下表 `:reason-unknown`→category 映射 + §5.1 传播侧 `unverified-callee`）|
| 超墙钟预算（`--timeout`） | `timeout` | `elapsed_ms`（`elapsed_ms ≥ --timeout` 判 timeout）|
| VC IR 无法编码理论（v0.3：`unbounded-loop` / `heap` / `string`） | `unsupported-theory` | which：`unbounded-loop`/`heap`/`string`（**不含 nonlinear**——见下）|
| solver 基础设施失败（崩溃/启动失败/输出解析失败/model 提取失败/OOM） | `solver-error` | `error_kind`：`solver-crash`/`spawn-failed`/`parse-failure`/`model-extraction-failure`/`oom`（§9.3 / §11）|

**`:reason-unknown` → reason category 映射**（solver 经 SMT-LIB `:reason-unknown` 属性报告理由——自由串，典型值 `memout`/`incomplete`/`timeout`/`(incomplete (theory arithmetic))`/`other`）：

| solver `:reason-unknown` | Tier 3 reason category | 备注 |
|---|---|---|
| 含 `quantifier`/`quantif` | `quantifier` | 量词推理 |
| 含 `(incomplete (theory arithmetic))`/非线性片段 | `nonlinear` | 非线性算术（可编码、仅不完备，故归 `unknown` 非 `unsupported-theory`）|
| 含 `timeout`/`memout`（但未超 `--timeout` 墙钟） | `timeout-adjacent` | solver 自报资源边界（`memout`/内部 `timeout`），未达墙钟 `--timeout`；另：`elapsed_ms/--timeout ≥ 0.9`（近墙钟）亦判 `timeout-adjacent` |
| 其他/无法分类 | `unclassified` | fallback——结构化但无更细类别 |

**一类传播引擎侧 category**（非 solver 报告，由传播引擎赋值）：`unverified-callee`（值形 `unverified-callee:<f>`，§5.1）——caller VC 因所依赖的 callee（SCC 之外）function-contract VC 未达 `proved` 而降级 `unknown` 时填此 reason。VC IR 静态标注（§4.2，含 `linear-arithmetic`/`nonlinear-arithmetic`/`quantifier`/`uninterpreted-function`/`datatypes`/`array`）是上表把 solver `unknown` 映射为 `nonlinear` 等 category 的输入；v0.3 这些片段分三类：(a) 完整可编码且完备（linear/quantifier/uninterpreted/datatypes/array）→ solver 决定、unknown 走上表映射；(b) 可编码但不完备（nonlinear）→ `nonlinear` category；(c) 真无法编码（unbounded-loop/heap/string）→ 预判 `unsupported-theory` status（§6.2 表，不进 solver）。故**无独立「VC 侧静态 reason category」**——VC IR 标注是 (b) 的检测输入与 (a)(c) 的路由依据。reason category 共**两类来源**：solver-derived（`quantifier`/`nonlinear`/`timeout-adjacent`/`unclassified`）+ 传播侧（`unverified-callee:<f>`），§7.1 `unknown_reason` 值集 = 此两类之并，封闭可枚举。

**`timeout` status 与 `unknown[timeout-adjacent]` 的边界**：`elapsed_ms ≥ --timeout`（墙钟超预算）→ `timeout` status；`elapsed_ms < --timeout` 但 solver 自报 `timeout`/`memout` → `unknown` + `timeout-adjacent`。两者不混淆。

**非线性算术裁定**：非线性算术（变量相乘）在 SMT-LIB 是**可编码**标准理论（QF_NIA/QF_NRA，VC IR `expr_dag` 可表达、§6.1 SMT-LIB v2 接受），仅**不完备**——故走 `unknown` + reason `nonlinear`，**不**入 `unsupported-theory`（后者只保留 v0.3 真无法编码的 `unbounded-loop`/`heap`/`string`）。

研究 §9.2 line 163 + §6① line 80：`unknown` 必须有**结构化理由**，不是 Z3 opaque。VC IR 的 span + 理论片段标注（§4.2）+ 上表 `:reason-unknown` 映射共同使 solver 路径的 reason category 可靠提取（含 `unclassified` fallback，绝不退回 opaque）；传播路径的 `unverified-callee:<f>`（§5.1）由传播引擎直接赋值。两类来源合起来使 `unknown_reason` 值集封闭可枚举。这是「AI-era 加法 ①」（研究 §6 line 80）的核心可操作化。

---

## 7. 结构化验证结果协议

### 7.1 结果 schema（`ail verify --json`，独立 `version`，非 `.ailmeta`）

```json
{
  "version": "0.1",
  "tool": "ail-verify", "solver": "z3-4.13", "level": 2,
  "summary": {
    "proved": 41, "failed": 3, "unknown": 1, "timeout": 0, "unsupported": 1, "solver-error": 1, "total": 47,
    "fixture_coverage": { "functions": 15, "with_fixture": 12, "without_fixture": 3 }
  },
  "results": [
    {
      "kind": "function-contract",
      "status": "failed-with-counterexample",
      "subject": { "function": "deduct", "module": "math" },
      "span": { "file": "math.ail", "line_start": 41, "col_start": 1, "line_end": 44, "col_end": 1 },
      "vc_summary": "∀balance,amount· amount>0 ∧ balance≥amount ⇒ result==balance-amount",
      "solver": "z3-4.13", "elapsed_ms": 340,
      "counterexample": {
        "inputs": { "balance": 100, "amount": 30 },
        "reason": "requires 成立但 body 返回 40，未建立 ensures（期望 balance-amount=70，math.ail line 88）"
      },
      "unknown_reason": null, "unsupported_theory": null, "error_kind": null
    },
    {
      "kind": "constraint-invariant",
      "status": "proved",
      "subject": { "type": "Age", "module": "age" },
      "span": { "file": "age.ail", "line_start": 71, "col_start": 18, "line_end": 71, "col_end": 22 },
      "vc_summary": "∀n· 0 ≤ n ∧ n ≤ 150  (at Age(n) construction)",
      "solver": "z3-4.13", "elapsed_ms": 12,
      "counterexample": null, "unknown_reason": null, "unsupported_theory": null, "error_kind": null
    },
    {
      "kind": "property",
      "status": "failed-with-counterexample",
      "subject": { "function": "reverse", "module": "list", "property": "reverse_idempotent" },
      "span": { "file": "list.ail", "line_start": 58, "col_start": 5, "line_end": 58, "col_end": 40 },
      "vc_summary": "∀xs· reverse(reverse(xs)) == xs   (handwritten, source:test)",
      "solver": "z3-4.13", "elapsed_ms": 210,
      "counterexample": {
        "inputs": { "xs": [3, 1, 2] },
        "reason": "reverse(reverse([3,1,2])) == [2,3,1] ≠ [3,1,2]（reverse 实现非对合，list.ail line 30）"
      },
      "unknown_reason": null, "unsupported_theory": null, "error_kind": null
    },
    {
      "kind": "function-contract",
      "status": "unknown",
      "subject": { "function": "interp", "module": "math" },
      "span": { "file": "math.ail", "line_start": 30, "col_start": 1, "line_end": 60, "col_end": 1 },
      "vc_summary": "∀x,y· requires(x,y) ⇒ ensures(x,y)",
      "solver": "z3-4.13", "elapsed_ms": 4200,
      "unknown_reason": "nonlinear",
      "counterexample": null, "unsupported_theory": null, "error_kind": null
    },
    {
      "kind": "call-site-precondition",
      "status": "failed-with-counterexample",
      "subject": { "caller": "transfer", "callee": "debit", "module": "bank" },
      "span": { "file": "bank.ail", "line_start": 95, "col_start": 9, "line_end": 95, "col_end": 24 },
      "vc_summary": "at transfer→debit call site: debit.requires[acct,amt] holds",
      "solver": "z3-4.13", "elapsed_ms": 88,
      "counterexample": {
        "inputs": { "amt": -5 },
        "reason": "transfer 在 amt=-5 下未建立 debit.requires（amt>0），调用点违反 callee 前置（bank.ail line 95）"
      },
      "unknown_reason": null, "unsupported_theory": null, "error_kind": null
    },
    {
      "kind": "function-contract",
      "status": "unsupported-theory",
      "subject": { "function": "retry_until", "module": "net" },
      "span": { "file": "net.ail", "line_start": 12, "col_start": 1, "line_end": 20, "col_end": 1 },
      "vc_summary": "(unbounded while loop)",
      "solver": null, "elapsed_ms": 0,
      "unsupported_theory": "unbounded-loop",
      "counterexample": null, "unknown_reason": null, "error_kind": null
    },
    {
      "kind": "function-contract",
      "status": "solver-error",
      "subject": { "function": "normalize", "module": "vec" },
      "span": { "file": "vec.ail", "line_start": 14, "col_start": 1, "line_end": 22, "col_end": 1 },
      "vc_summary": "∀v· v != zero ⇒ result==scale(v, 1/len(v))",
      "solver": null, "elapsed_ms": 0,
      "error_kind": "spawn-failed",
      "counterexample": null, "unknown_reason": null, "unsupported_theory": null
    }
  ]
}
```

**字段语义**：
- `kind`：`function-contract` | `constraint-invariant` | `property` | `call-site-precondition`（§4.1 四类）。
- `status`：`proved` | `failed-with-counterexample` | `unknown` | `timeout` | `unsupported-theory` | `solver-error`（§6.2 六态）。
- `subject`（四形，按 kind）：`{function, module}`（function-contract）/ `{type, module}`（constraint-invariant）/ `{function, module, property}`（property，复用其归属 fn 标识 + property 名，承 RFC 0003 §5 归属规则）/ `{caller, callee, module}`（call-site-precondition，标识失败的 caller→callee 调用边）。
- `span`（§69.1 / §24，按 kind 四类映射）：function-contract → 契约区 span；constraint-invariant → `T(expr)` 构造点 span；property → property 块 span；**call-site-precondition → 调用点 span**（取自 §8.2 `calls[].span`，§5 传播引擎用之）。
- `vc_summary`：人/AI 可读的 for-all 语句（喂 self-repair）。
- `counterexample.inputs`：**结构化、非 SMT model 原文**（§9.2 line 163 / §4.2 反向绑定）。按 kind：function-contract/call-site-precondition → 按参数名为 key（标量）；constraint-invariant → 按构造实参名；property → 按 generator draw 名或形参名。**struct/数组反例用嵌套对象 + 数组**（如 `{ "xs": [3,1,2] }` 或 `{ "account": { "balance": 30 } }`），非扁平点号字符串。仅 `failed-with-counterexample` 时非 null。
- `unknown_reason` / `unsupported_theory` / `error_kind`：仅对应 status 时非 null（§6.2）；其余恒 null。
- `spec_smells`（可选）：spec-smell 标记数组（§10），值 ∈ `vacuous-ensures`/`vacuous-requires`/`vacuity-unknown`/`vacuous-property`/`no-fixture`；无 smell 时 absent 或 `[]`。**放置（附于其相关 spec 所在的 VC 行）**：`vacuous-ensures`/`vacuous-requires`/`vacuity-unknown`（源于 ensures/requires 查询）附于该函数的 **function-contract VC 行**（requires/ensures 所在）；`vacuous-property`/`vacuity-unknown`（源于 holds 查询）附于该 **property VC 行**（holds 所在——**per-property 非 per-function**，故不附 function-contract 行，以免误读为契约本身空虚）；`no-fixture` 附于标识该函数的任一 VC 行。若相关 VC 行不存在（如该函数无 function-contract VC、仅有 property/call-site-precondition VC，smell 为 `no-fixture` 等），则附于 subject 标识该函数的任一 result 行。status 可仍为 `proved`（空虚真），`spec_smells` 标记它不被信任为「真验证」（§10.1）。示例：`ensures:["true"]` 的 function-contract VC 行可带 `"spec_smells": ["vacuous-ensures"]`；`assert(1==1)` 的 property VC 行可带 `"spec_smells": ["vacuous-property"]`。
- `summary`（聚合）：`proved`/`failed`/`unknown`/`timeout`/`unsupported`/`solver-error` 为各 status 的 VC 计数——**缩写映射**：`failed` = status==`failed-with-counterexample` 的条目数、`unsupported` = status==`unsupported-theory` 的条目数、其余四键与 status 同名同计；`total` = 六键之和 = `results[]` 总条目数。聚合不变量：`results[]` 为完整集时各键 == 该 status 条目数；为展示子集时各键 ≥ 子集中该 status 条目数（上例 7 条为 47 条之子集：shown failed=3 ≤ summary.failed=3、shown proved=1 ≤ summary.proved=41）。
- `summary.fixture_coverage`（可选）：`functions` = 被验证的 **public 函数数**（分母；private=fixture-N/A 豁免、不计，§10.2，故 `functions` ≤ `total`——典型 `functions` < `total`，因多 VC/函数 + private 函数 VC 计入 `total` 但不计 `functions`；等号在「每 public 函数恰 1 条 VC 且无私有函数」时成立）；`with_fixture`/`without_fixture` = 其中有/无夹具子集（`with_fixture + without_fixture = functions`）。

这是「AI-era 加法 ①」（研究 §6 line 80）的产物 schema——机器可解析、喂 self-repair 闭环，非 opaque pass/fail。`deduct`/`Age`/`reverse`/`transfer→debit` 均为纯、loop-free、无堆突变的函数（落在 v0.3 实际可达域，§5.2 / §12.3），与 §6.2 / §12.2 能力边界自洽。

---

## 8. `.ailmeta` 扩展与 `.ailmeta.verify`

### 8.1 主 `.ailmeta` compact `verification` 字段（0.3.0 第 4 rider）

主 `.ailmeta` 的 `function` 条目新增**可选** `verification` 字段——**共享 `schema_version` 0.2.0 → 0.3.0 bump 的第 4 rider**（RFC 0001 `examples[]` + RFC 0002 `contracts` + RFC 0003 `properties[]` + **本 RFC `verification`**），**不另声明 bump**（§24 line 1332 schema_version 独立 semver；§88 #4/#5 line 3318-3319 可选字段先例）：

```json
"verification": { "status": "proved", "level": 2, "solver": "z3-4.13" }
```

- `status` ∈ `proved`（该函数全部 VC 证毕）| `partial`（**非全部 proved 且无 failed VC**——覆盖「部分证毕 + 其余 unknown/timeout/unsupported/solver-error」与「0 proved、全 unknown/timeout/unsupported/solver-error」两情形）| `failed`（有任意 VC `failed-with-counterexample`）。**`unverified` 不是可 emit 的枚举值**（见下 absent 规则）。
- `level`（int）：证毕所用最高验证等级（§9.2）。
- `solver`（string）：所用 solver + 版本。

**compact status 聚合映射**（§7 per-VC status → compact）：函数有任意 VC `failed-with-counterexample` → `failed`；否则全部 `proved` → `proved`；否则（有 `unknown`/`timeout`/`unsupported-theory`/`solver-error` 但无 `failed`，**含 0-proved-all-unknown**）→ `partial`。三分故该函数 compact status 总可判定，无未定义间隙。

**存在规则**：`verification` 字段 **absent-by-default**——`ailc` **从不** emit 它（ailc 不验证）；仅 `ail verify --write-meta`（opt-in）把它写回主 `.ailmeta`，且**仅为已验证函数**（有 §7 result 行者）写——不为未验证函数写 `status:unverified`。**absent ≡ unverified**：`unverified` 是**字段缺省的逻辑值**而非可 emit 枚举值；一个函数的 `verification` 字段 absent ⇔ 该函数未跑过验证。消费者见无 `verification` 字段即判 unverified，无需区分「未验证」与「字段未写」（二者等价）。**这是第 4 rider 中唯一非 ailc 产生的字段**（前三 rider examples/contracts/properties 均由 ailc 从源码 emit；verification 由 `ail verify` 产生）——明文标注此不对称。**staleness 语义**：`ailc` 重新生成 `.ailmeta` 时**丢弃** `verification`（源码已变 → 旧证明失效，须重跑 `ail verify`）——这是**正确**行为，非缺陷。

**为何 compact 字段在主 `.ailmeta`**：主 `.ailmeta` 是 AI 读「程序语义」的源（§24 line 1361「`.ailmeta` 是 AILang 存在的全部理由」）。一个「契约已机器证明」的 compact marker 给 AI **置信度信号**——区分「契约已声明」（contracts 在）vs「契约已机器证明」（verification 在），无需读 `.ailmeta.verify` 详情。反例/solver 输出等**验证过程状态**不入主 `.ailmeta`（保紧凑、高语义信噪比），只进结构化结果（§7）。

**写入机制**（firm choice + 备选在 §12 #6）：`ail verify --write-meta` amend 主 `.ailmeta`、为每个验证过的 function 填 `verification`。备选 = `ailc --emit verify-bundle` 时合并一份 verify 输出文件——留未决。

### 8.2 `.ailmeta.verify` 伴随工件（独立 schema_version）

`.ailmeta.verify` 是 `ailc --emit verify-bundle` 产出的**独立文件**（非 §24 `.ailmeta` 本体，不受 §88 #2 line 3316 schema 锁定约束），有**独立** `schema_version`（如 `"0.1"`）：

```json
{
  "schema_version": "0.1",
  "ailmeta_schema_version": "0.3.0",
  "module": "math",
  "functions": [
    {
      "identity": { "name": "deduct", "description": "从余额扣除金额。" },
      "visibility": "public",
      "is_pure": true, "is_async": false,
      "input": [ { "name": "balance", "type": "i64" }, { "name": "amount", "type": "i64" } ],
      "output": "i64",
      "contracts": { "requires": [ … ], "ensures": [ … ] },
      "body_ir": { "expr_dag": [ … ], "cfg": [ … ], "spans": [ … ] },
      "property_assertions": [ { "property": "nonneg_balance", "span": { … }, "holds_ir": { "expr_dag": [ … ], "spans": [ … ] } } ],
      "calls": [ { "target": "sub", "span": { … } } ]
    }
  ],
  "types": [ { "identity": { … }, "constraint": [ "value >= 0", "value <= 150" ] } ]
}
```

- `schema_version`：本伴随工件独立版本（与主 `.ailmeta` 解耦）。
- `ailmeta_schema_version`：所伴随主 `.ailmeta` 的版本（交叉引用）。
- `functions[]`：**含 public + private** 全函数（兑现 RFC 0002 §10#6，private 契约进 verify-bundle 而非主 `.ailmeta`）。每函数带 `visibility`/`is_pure`/`is_async`/`input[]`（含类型，function-contract VC 派生 SMT sort 所需）/`output`（返回类型，`ensures` 返回值绑定所需）/`contracts`/`body_ir`/`property_assertions`/`calls`。`input[]`/`output` 携带 §24 function 条目的**类型子集**——`input[]` 条目 = `{name, type}`、`output` = 返回 type（串）；**省略** §24 的 `mode`/`meaning`（Tier 3 仅验 pure 函数，参数 `mode`（copy/borrow）对 VC 编码无影响、仅需 type 派生 SMT sort；`meaning` 是自然语言注释、无证明用途）。private 函数（主 `.ailmeta` 无其条目）的签名类型**由此工件提供**，使 verify-bundle 对 private 函数亦「自包含」（§3.2）。**不携带** `examples[]`/`properties[]` **元数据**（source/name/span/generators，public-only、读主 `.ailmeta` 即可）；但携带 `property_assertions`（手写属性的断言 IR，见下），使 Mode A 对 handwritten property VC 亦自包含（§3.2 / §4.1）。
- **无 `constraint_params` 字段**——constraint 谓词已在 `types[].constraint`（§24 line 1350），形参约束经 `input[].type`（若该 type 为 constraint 类型）派生，无需冗余字段。
- `body_ir`：受限 body IR（设计契约，§3.2；具体序列化待 ailc 落地）。
- `property_assertions[]`（可选，仅该函数有归属手写属性时）：每个手写属性（RFC 0003 source:"test"，经 RFC 0003 §5 归属规则挂本函数）的**断言 IR**（`holds`——test 块断言体的表达式 DAG + span + 调用边）+ 其 test 块 §69.1 `span`（= 交叉引用键，见下）。用 test 块 §69.1 `span` 交叉引用主 `.ailmeta` `properties[]` **同 span** 条目——RFC 0003 §5 line 116/125 每条 `source:"test"` 属性携带其 test 块 `span`；string_lit 名**不唯一**（§27 line 1502 不强制、RFC 0003 §5 line 126 仅「必填」），故**以 span 为键**（多归属同 test 块的跨函数条目 span 相同→交叉引用一致；不同 test 块即便 string_lit 同名 span 不同→不混），避免同名碰撞使交叉引用歧义。这是 handwritten property VC（§4.1）的 `holds` 来源 + 多归属去重键——test 块是顶级 §27 item、非函数体、不在主 `.ailmeta` properties[] 元数据中，故断言 IR 须由 verify-bundle 提供；缺此（无 verify-bundle 或未携带）则 Mode A 无法生成该函数的 handwritten property VC（须 Mode B 源码重解析兜底）。
- `calls[]`：调用图边（callee 标识 + span），供传播引擎（§5）。
- `types[]`：带 constraint 谓词的类型（§24 line 1350 形）。

**向后兼容**：主 `.ailmeta` 的 `verification` 字段是可选字段，旧 reader 忽略之即可（§88 #4/#5 先例）；`.ailmeta.verify` 是新文件，不影响既有 `.ailmeta` 消费者。

---

## 9. 工具链

### 9.1 CLI 表面

见 §3.1 表。`ail verify` 是独立子命令；`ailc --emit verify-bundle` 是 `ailc` 的 opt-in emit flag（非默认——研究 §3 line 42「初期不烤进 `ailc`」指 solver/VC/传播逻辑不烤进，emit 更多元数据是「备料」、不引入 solver 依赖、不让编译时间不确定）。

### 9.2 四级 `--level`（零关键字，镜像 SPARK 分层）

研究 §6③ line 82 SPARK 分层：默认内存安全 → opt-in 无运行时错误 → opt-in 关键模块功能正确。`--level` flag 表达四级：

| level | 名称 | 行为 | SPARK 类比 |
|---|---|---|---|
| `0` | off | `ail verify` no-op（验证未启用）：不生成任何 VC、不调用 solver、不 amend `.ailmeta`（`--write-meta` 写 0 个 `verification`）、`--json` 输出 `results:[]` + `summary.total=0`、退出码 0 | — |
| `1` | per-function | 孤立证每个**闸门合格**函数的 function-contract VC + constraint-invariant VC + **property VC**（无跨函数）。property VC：**derived**（`source:"ensures"`）归并入该函数 function-contract VC 的 `ensures` 分量（同一义务，**dedup** 不双计 `summary.total`）、**handwritten**（`source:"test"`）独立 emit、多归属按 test 块 §69.1 span 去重（§4.1 两源精确语义） | per-function |
| `2` | inter-procedural | level 1 + **跨函数传播引擎**（callee `ensures`→caller 假设；caller 证 callee `requires`，§5） | 无运行时错误（跨函数）|
| `3` | whole-module | level 2 + constraint invariant 跨全路径维持 + 模块级断言 | 关键模块功能正确 |

**默认 level = 1**（`0` 表示显式关闭）。零关键字——`--level` 是 CLI flag。

### 9.3 退出码

- `ail verify` 有任意 VC `failed`（`failed-with-counterexample`）→ **非零退出码**。
- `--strict`：`unknown`/`timeout`/`solver-error` 也算失败 → 非零。
- **fixture-coverage gate**（§10.2）在 `--require-fixture` 或 `--strict` 下对某 public 函数触发 `no-fixture` → **非零退出码**（独立于 VC status——缺有效夹具的 hard fail，不因无 failed VC 而放过；level≥1 下成立，level 0 无函数被验证故 `no-fixture` 不触发）。
- 全部 `proved`/`unknown`/`timeout`/`unsupported-theory`/`solver-error`（无 `failed`）且非 `--strict` 且**无 `no-fixture` hard fail**（即未设 `--require-fixture` 或无 public 函数缺有效夹具）→ **零退出码**。
- **solver 基础设施失败**（solver 二进制缺失/崩溃/OOM/输出解析失败，§6.2 `solver-error`）默认 warning、不阻塞退出码（同 `unknown`，因 VC 未被证伪、只是未判定）；`--strict` 下计非零。`solver-error` 经 `--json` 的 `error_kind` 结构化（喂 self-repair：如 `spawn-failed` 提示「未安装 z3」、`oom` 提示「调大内存或缩小 `--timeout`」）。
- **关键**：这是 `ail verify` 的退出码，**不是** `ail build` 的（分离公理 §2 #1）——`ail build` 永不调用验证器、永不因验证结果非零。

---

## 10. 反 spec-hacking 集成

研究 §4 #3 line 52：RLVR 模型 `ensures true` + `assume false` 骗 solver，58%→过滤后 31% honest——**验证不能单独被信任**，必配测试夹具。两层机制：

### 10.1 trivial-spec 检测（验证器内置，always-on）

检测 vacuous spec 并标 `spec-smell`（附 warning，**不被信任为「真验证」**）：
- `ensures: ["true"]` 或规范化后等价 `true` → `spec-smell: vacuous-ensures`。
- `requires: ["false"]` 或不可满足 → `spec-smell: vacuous-requires`（函数空虚真——任何调用都违反前置，故 `requires⇒ensures` 平凡成立，但函数无实际用途）。
- 这正是 RLVR 作弊模式（`ensures true` + `assume false`）。
- **vacuity 查询自身的 `unknown`**：三类 vacuity 判定本身是**有界 SMT 查询**、均可能返回 `unknown`（§6.2 同类边界）——「ensures 等价 true」（= `¬ensures` 是否 unsat）、「requires 不可满足」（= `∃σ·requires(σ)` 是否 sat）、「holds 等价 true」（= `¬holds` 是否 unsat，vacuous-property 检测用，见下条）。vacuity 查询 `unknown`（三类任一）→ `spec-smell: vacuity-unknown`（保守 warning：无法判定 spec 是否空虚，**不**强行标 `vacuous-ensures`/`vacuous-requires`/`vacuous-property` 亦**不**放过）——填补反 spec-hacking 检测自身可被「无法判定」绕过的缝隙（三类查询全覆盖）。
- **handwritten property 的平凡断言**（source:"test"，§4.1）：手写属性的 `holds`（test 块断言）规范化后等价 `true`（如 `assert(1==1)`）→ `spec-smell: vacuous-property`——与 `vacuous-ensures` 对称：其 VC `∀inputs·true` 平凡 `proved` 且其 presence 满足 §10.2 fixture gate（若无此 smell），正是 property 版的 RLVR trivial-spec 漏网。

检测到 trivial/vacuity 的 VC：status 仍可能是 `proved`（空虚真），但附 `spec-smell` warning（§7 results 可扩展 `spec_smells: [...]`，值如 `vacuous-ensures`/`vacuous-requires`/`vacuity-unknown`/`vacuous-property`/`no-fixture`，§7.1）——**不被信任为「真验证」**。

### 10.2 fixture-coverage gate（`--require-fixture` opt-in，默认 warn-on-zero）

**仅作用于 public 函数**。一个被 SMT 验证的 **public** 函数，若**同时缺** Tier 1a doctest（`examples[]`，RFC 0001）**且缺有效** Tier 1b PBT 夹具（`properties[]` 中无任何**有效夹具**——vacuous 属性不算，精确定义见下「fixture 判定」）→ `spec-smell: no-fixture`。
- **private 函数 = `fixture-N/A`**，**永不**判 `no-fixture`、不计入指标分母——其 `examples[]`/`properties[]` 经 RFC 0001 §5（非 public 不入 `.ailmeta`）/ RFC 0003 §9.1（emission 闸门=public）public-gated，验证器可读工件（主 `.ailmeta` + `.ailmeta.verify`，§3.2）中无其 fixture 数据（`.ailmeta.verify` 亦不携带 examples[]/properties[]，§8.2），无从判定故显式豁免。这杜绝「任何含 private helper 的模块在 `--strict`/`--require-fixture` 下系统误报」。
- 默认 **warn**（研究 §4 #3 line 52：验证不能单独被信任，warn 是最低提醒）。
- `--require-fixture` / `--strict` 升级为 **hard fail**（仅对 public 函数触发）。

**fixture 判定 = 纯存在性（presence，存在量化）**：public 函数在主 `.ailmeta` 有非空 `examples[]`，**或**其 `properties[]` 中**至少一条为有效夹具**即判「有夹具」——一条属性为**有效夹具**当且仅当其**非 vacuous**：手写属性（`source:"test"`）无 §10.1 `vacuous-property` 标记、**且**推导属性（`source:"ensures"`）其 ref 指向的 `ensures` 无 §10.1 `vacuous-ensures` 标记（`ensures:["true"]` 推导出的属性 VC = `∀inputs·true`，与 `assert(1==1)` 同为平凡夹具、不算有效——正是 §10 line 424 点名的 RLVR `ensure true` 作弊向量）**且**该函数无 §10.1 `vacuous-requires` 标记（`requires:["false"]` 不可满足，使 derived property VC = `∀inputs·false⇒ensures ≡ ∀inputs·true` 平凡真——RLVR `assume false` 向量，与 `ensure true` 对称的另一半；RFC 0003 §5 derived property 按 `ensures` **存在**即生成、不看 `requires` 可满足性，故 `requires:["false"]` 仍会 emit 一条 derived property，须由 fixture gate 此款堵住）。**手写属性（`source:"test"`）的 `requires` ≡ true 恒可满足、永不触发 `vacuous-requires`**，故此款仅作用于 derived 属性。**存在量化**：单个 vacuous 属性**不**污染同 `properties[]` 中其他有效夹具源（如函数同时有一条 derived 有效属性 + 一条 vacuous 手写属性 → 仍判「有夹具」）。**不**要求 property 曾被 `ail test` 执行（执行态不在 `.ailmeta` 可读字段；且 RFC 0003 §9.2 对非 pure/async fn 的推导 property 标 `[skipped]` 不执行，故 `source:"ensures"` ≠ executed）。presence 判据与验证器可读数据（主 `.ailmeta`）一致、可机械实现。fixture-coverage 指标**仅进 §7 summary**（`with_fixture`/`without_fixture`/`functions`，分母 `functions` = 被验证的 public 函数数，§7.1）；**不**进主 `.ailmeta` compact `verification`（§8.1 schema 锁定为 `{status, level, solver}`，保紧凑，§8.1 line 设计原则）。

### 10.3 三层 Tier 角色锁定

研究 §6②/③ + RFC 0003 line 24：Tier 1a = exists 实例、Tier 1b = 经验 for-all、Tier 3 = 逻辑 for-all。Tier 3 是**最上层**但**不替代**下两层——fixture-coverage gate 强制这个分层不被绕过（光有机器证明、无经验/实例夹具 = 红旗）。

### 10.4 与 Tier 2 的边界（正交）

Tier 3（全 SMT、opt-in side-car、本 RFC）与**未来 Tier 2**（窄 decidable refinement，研究 §3 line 41——若建则编译器集成、仅 decidable 算术片段、可能 default-on）**正交**、互补不重叠：Tier 2 保核心类型系统 decidable（停止规则要求），Tier 3 跨 SMT 停止线作 opt-in 上层。本 RFC **不依赖** Tier 2 是否启动——Tier 3 可独立落地（消费 `.ailmeta`，与 Tier 2 无数据依赖）。

---

## 11. 诊断与容错

- **独立诊断通道**（非 panic、非编译失败）：验证结果**不走** §73 typecheck / §74 borrow（那些是编译期检查、影响 `ail build`）。验证结果是 `ail verify` 自身的诊断流——复用 **§69.3 Diagnostic 结构**（`{severity, code, message, span, notes}`，line 2772-2782）+ **§69.1 Span**（line 2759），但走 `ail verify` 的 stdout/JSON。
- **severity 语义**：`failed-with-counterexample` → error；`unknown`/`timeout`/`solver-error` → warning（除非 `--strict`，则 error，§9.3）；`proved` → 不报或 info；`unsupported-theory` → info/warning（明示能力边界，非失败）。
- **§34 panic 通道隔离**：§34（line 1753-1764）panic 符号表**无证明失败符号**——Tier 3 验证失败**不触发** `ConstraintViolation` 或任何 panic；它只是结构化诊断 + 退出码（§9.3）。`ConstraintViolation` 仍是运行期断言违约 panic（§34 line 1764 不变），Tier 3 只静态证明它不触发。
- **`unsupported-theory` 优雅降级**：unbounded loop / heap / string 等理论不支持的函数 → 结构化 `unsupported-theory`（§6.2 / §7），明示能力边界、不 panic、不阻塞编译、可被 self-repair 理解（「此函数超出 v0.3 验证能力，需 v0.4 loop invariant」）。
- **span 溯源**：每个验证结果带 §69.1 span，精确到源码字符位置（function-contract → 契约区、constraint-invariant → `T(expr)` 构造点、property → property 块、call-site-precondition → 调用点 span §7.1/§8.2），便于 AI 闭环修复「契约与实现不符」。

---

## 12. 限制与未决（首版 v0.3）

### 12.1 IN（v0.3）

- `ail verify` 独立 CLI 子命令（§3.1）。
- `ailc --emit verify-bundle` → `.ailmeta.verify` 伴随工件（§3.2 / §8.2，Mode A；Mode B 兜底）。
- 四类 VC 生成（§4.1；property VC level 1、call-site-precondition level≥2）+ 自定义 VC IR（§4.2）。
- SMT-LIB v2 solver-agnostic 后端（§6.1）+ **六态**结构化降级（§6.2）。
- 跨函数传播引擎 level≥2（§5）：callee `ensures`→caller 假设 + caller 证 callee `requires`（含 callee VC 未证毕时降级 `unknown:unverified-callee:<f>`，§5.1 Soundness 前提）。
- 反 spec-hacking：trivial-spec 检测 + fixture-coverage gate（§10）。
- 四级 `--level`（§9.2）。
- 主 `.ailmeta` compact `verification` 字段（§8.1，0.3.0 第 4 rider）+ `.ailmeta.verify` 独立版本（§8.2）。
- loop-free / 有界循环验证（unbounded loop → `unsupported-theory`，§5.2）。

### 12.2 OUT（v0.4+）

- **trait/interface 方法契约传播**（RFC 0002 §10#7 line 173）：`sigs[]` 表示模型 v0.2.1 §24 未定义 → Tier 3 无法读 trait 方法 postcondition → 跨 trait 边界传播待 v0.4。
- **终止性检查**（well-founded 递归证明）：跨停止规则（超多项式）。v0.4+。
- **ghost 类型 / ghost code**（仅规范变量）：需新文法。v0.4+。
- **refinement reflection**（类型携带逻辑信息进证明）：触类型系统（§86 #9 line 3283 nominal 不变性）。v0.4+。
- **完整非线性算术支持**：SMT 不完备 → 留 `unknown` + reason `nonlinear` 结构化（§6.2；非线性可编码、仅不完备，故走 `unknown` **非** `unsupported-theory`）。v0.4+ 随 backend 改进提升 proved 比例。
- **loop invariant 标注**（上下文关键字 `invariant`，§5.2）+ 自动 invariant 推断：v0.4 新 RFC。
- **heap/aliasing 推理**（separation logic）：v0.4+。
- **effectful/async fn 功能正确验证**：需 effect-injection 模型（冻结 spec 静默，RFC 0003 §9#4 line 245 同步推迟）。v0.4+。

### 12.3 闸门层 / 数据层 / 执行层三分（mirror RFC 0002 §7#2 line 140 / RFC 0003 §9）

| 层 | Tier 3 含义 | 实现 |
|---|---|---|
| **闸门层**（eligibility）| 哪些函数可验证。v0.3：`public` **或** `private`（verify-bundle 含两者）+ `pure` 函数（执行层要求 pure，承 RFC 0003 §9.2）| `ail verify` 内部判据 |
| **数据层**（`.ailmeta` + `.ailmeta.verify`）| ailc emit。主 `.ailmeta` public-gated 不变（RFC 0002 §7#2）；`.ailmeta.verify` 伴随 = 全 contracts（含 private）+ 签名类型（input[]/output）+ 调用图 + 受限 body IR | `ailc --emit verify-bundle` |
| **执行层**（`ail verify`）| VC 生成 + SMT 求解 + 结果 emit。opt-in，永不影响 `ail build`（分离公理 §2 #1）| `ail verify` |

**注**：闸门层含 `private` 函数（与 RFC 0002 §7#2 主 `.ailmeta` 的 public 闸门不同）——因 verify-bundle 是验证专用富视图、不受公开面信噪比约束（§3.2）。但执行层仍要求 `pure`（确定性 + 非纯 fn 的 effect/IO 超出 SMT 编码域，承 RFC 0003 §9.2）。

### 12.4 未决问题（待 review）

1. **终止性检查**（v0.4）：递归 well-founded 证明如何进 Tier 3？（跨停止规则，延后）。
2. **trait 契约传播**（v0.4）：与 RFC 0002 §10#7 的 `sigs[]` 表示模型联动。
3. **loop invariant 标注文法**（v0.4）：上下文关键字 `invariant`（同 `where`/`constraint` 先例）vs 纯自动推断（无标注）—— v0.4 新 RFC 决断。
4. **VC IR 具体序列化格式**：本 RFC 锁信息需求契约（§3.2），具体 JSON/text/binary 待 ailc 内部 IR 定型。
5. **默认 `--level` = 1**（**已定**，§9.2：per-function 是最低有意义的验证；`0` 为显式关闭，其与 `--write-meta`/`--json`/退出码组合的 no-op 行为已在 §9.2 level 0 行 + §9.3 定义，无未定义组合——§9.3 含 fixture-gate `no-fixture` hard-fail 规则，覆盖 `--require-fixture`/`--strict` × 缺有效夹具组合）。
6. **`verification` 写入主 `.ailmeta` 的机制**：`ail verify --write-meta` amend（推荐）vs `ailc --emit verify-bundle` 时合并 verify 输出文件。
7. **默认 solver**：z3（推荐，最广可用）vs 抽象不 mandate。
8. **`--timeout` 默认值**：5000ms/VC（推荐）vs 可配全局预算。
9. **body IR 携带粒度**：完整表达式 DAG（推荐，VC 生成需要）vs 仅控制流 + 调用边（不足，function-contract VC 需表达式）。

### 12.5 handoff 兑现表

| handoff 来源 | 兑现节 |
|---|---|
| RFC 0002 §1 line 22（1c 是 Tier 3 共同地基）| §1（契约 = VC 输入源）、§4.1 |
| RFC 0002 §6 line 125/128（传播引擎 = Tier 3，备料不传播）| §5（传播引擎）|
| RFC 0002 §7#2 line 140（闸门/数据/执行三分）| §12.3 |
| RFC 0002 §10#6 line 172（非 public 函数契约 × Tier 3）| §3.2 / §8.2（`.ailmeta.verify` 含 private 契约）/ §12.3（闸门含 private）|
| RFC 0002 §10#7 line 173（trait 契约 × Tier 3 → v0.4）| §12.2 OUT |
| RFC 0003 line 22（Tier 3 = for-all 逻辑证明，消费 contracts/properties）| §1（for-all 两层）、§4.1（property VC）|
| RFC 0003 line 24（Tier 3 证 ∀x·requires⇒ensures）| §4.1（function-contract VC）、§1 |
| RFC 0003 §9#4 line 245（自动 SMT discharge = Tier 3）| §6（SMT 后端）|
| 研究 §3 line 42（Tier 3 定义：永不默认 / 消费 .ailmeta / 初期不烤进 ailc）| §2 #1（分离公理）、§3（架构）|
| 研究 §4#3 line 52（反 spec-hacking 58%→31%）| §10 |
| 研究 §4#4 line 53（跨函数传播堵 DAFNYCOMP gap）| §5 |
| 研究 §6① line 80（结构化验证器协议）| §7 |
| 研究 §6③ line 82（SPARK 分层）| §9.2（四级 level）|
| 研究 §9.2 line 163（杜绝 Z3 opaque）| §6.2（六态降级）、§4.2（VC IR）|

---

## 13. 不触动 v0.2.1（合规声明）

1. **不改 §1–§94 任何条款**——本 RFC 只读引用 §9(373-393) / §15.4(535-548) / §20(974-994) / §24(1330-1361) / §27(1461-1554) / §34(1753-1764) / §69.1(2759) / §69.3(2772) / §82#3(3204) / §83#3(3216) / §86#9(3283) / §88#2(3316) / §88#4/#5(3318-3319) / §89#3(3334) / §92#2(3385) / §92#4(3387) / §92#9(3392)。不修改 `docs/AILANG.md`（同 RFC 0001 §12 / RFC 0002 §11 / RFC 0003 §13 定位）。
2. **不增关键字（F6=56 不变）**——`verify`/`proof`/`solver`/`refine`/`invariant`/`ghost`/`level`/`emit` 均**非关键字**（§9 line 373-393 的 56 不含其一，grep 确认）。`ail verify` 是子命令 ident（如 `build`/`test`/`doc`）；`--level`/`--solver`/`--timeout`/`--json`/`--strict`/`--require-fixture`/`--write-meta`/`--emit` 为 CLI flag；`verification` 是 `.ailmeta` 字段名；`verify-bundle`/`.ailmeta.verify` 是文件/工件名。**零新词法，56 不变**。
3. **不增文法产生式**——本 RFC **不动 §27 任何产生式**。全部表面 = CLI + `.ailmeta`/`.ailmeta.verify` 字段。源码内无新注解（loop invariant 等显式推迟 v0.4，§5.2 / §12.2）。**不触动 §27**。
4. **不增 §34 panic 符号**——验证失败**不是 panic**（§11）——它是结构化诊断 + 退出码，留在 panic 通道之外。`ConstraintViolation` 仍是 §15.4 运行期断言违约 panic（§34 line 1764 不变），Tier 3 只静态证明它不触发、不改变其运行期行为。
5. **不引入子类型（§86 #9 line 3283 nominal 不变性）**——Tier 3 验证是**外部证明**，非类型系统。constraint 的 `T(expr)` 是**检查构造**模型（§92 #2 line 3385），不是 refinement 子类型。验证器证「这个构造的谓词成立」，不引入 `Age <: int` 子类型关系。
6. **不改 110 项决议**——§82 #3（line 3204）+ §83 #3（line 3216）**正是**把 SMT 留 v0.3+ 的决议——本 RFC 填补的正是这条移交。§82 #3 把它归为**工程实现层开放问题（非语言决议）**，独立外部验证器形态最忠实。
7. **不违停止规则（研究 §2 line 20-28）**——Tier 3 **正因为**跨过 SMT 停止线（启发式 solver / 超多项式 / 标注:代码比），才留在 opt-in 默认路径之外（分离公理 §2 #1）。per-VC `--timeout` 预算 + 四级 `--level` 分级 + `unsupported-theory` 结构化降级共同保证可预测。
8. **schema 协调**：主 `.ailmeta` `verification` 字段 = 共享 0.2.0→0.3.0 bump **第 4 rider**（RFC 0001 `examples[]` + RFC 0002 `contracts` + RFC 0003 `properties[]` + 本 RFC `verification`），向后兼容（可选字段，§88 #4/#5 line 3318-3319 先例）。`.ailmeta.verify` 伴随工件**独立** `schema_version`（独立文件，非 §24 本体，不受 §88 #2 line 3316 锁定）。
- SMT side-car 是 **v0.3+ 特性**；本 RFC 为**提案**，经 review + 批准后进 v0.3 规范（届时 AILANG.md §24 schema 增 `verification` 字段描述、`.ailmeta.verify` 工件规格、`schema_version` 共享 bump 0.2.0→0.3.0 第 4 rider）。
- 本 RFC 不修改 `docs/AILANG.md`（同 RFC 0001 §12 / RFC 0002 §11 / RFC 0003 §13 定位）；不修改 RFC 0001/0002/0003 与研究存档。
