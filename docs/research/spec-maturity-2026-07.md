# AILang 规范工程成熟度审计报告

**日期**：2026-07-27
**审计主题**：AILang 文档（docs/AILANG.md §1–§94 + 附录 A/B/C + docs/rfc/0001–0004）作为规范工程产物，距离支撑真实编译器实现的成熟标准有多远
**材料基础**：8 主领域 + 4 补漏领域，共 68 条 confirmed 发现（confirmed 数组原样计数，含 3 条 strength 类发现；其中 3 个主题——字面量词法、UB 三分法、并发内存模型——在多个维度交叉列举）；本轮 0 refuted、0 uncertain（说明审计论据扎实，但亦说明规范在多数承重点均无反驳空间，本身即是成熟度信号）
**对照基线**：commit 2f63e18「四个极致」深度评审总判决 5.25/10

---

## 1. TL;DR

**文档不成熟，不能支撑独立团队据此实现编译器。**

整体成熟度经 12 维加权计算为 **5.2 / 10**（算法见 §2）。注：此分数与上轮「四个极致」5.25/10 采用不同的维度划分与聚合方法（本轮 12 维加权平均 vs 上轮 4 维简单平均），数值接近但不构成互证——两套框架度量的是不同切面，不应将数值 proximity 解读为相互印证。判据有三：(1) 规范治理三件套——conformance 条款 / RFC 2119 规范性词汇 / normative-informative 分区——全缺（§44 仅为语法约定，全文 grep `MUST|SHALL|RFC 2119` 零命中）；(2) 形式层只有 EBNF 骨架、无内核（§27 把 if/for/while/loop 列为裸终结符，无 if_stmt 产生式，连规范自身的 §16 控制流示例都无法推导）；(3) 字面量词法、数值转换算子、并发内存模型、常量求值域、名字解析算法、借用检查器形式规则六大承重子系统为零规范。

**约当 Rust 2010–2012 年（pre-alpha：骨架已具、内核未形）**。各维度成熟度高度不均：错误模型语义层、泛型概念骨架、形式化层 EBNF 覆盖面等约当成熟标准 50–70%；而常量求值域（对照 Rust Reference Constant evaluation 章约 5% 完成度，系全规范最薄弱单点）、ABI/FFI（2.0/5）、名字解析（2.0/5）等维度则远低于此。加权后的 5.2/10 反映了这种极度不均衡——少数亮点（数值语义、panic 跨 FFI 契约）被大面积空白拉低。

局部亮点可肯定：整数溢出/除零/NaN 永不 UB 的精确定性（§15.1/§90 #1-#3）、panic 跨 FFI→abort 非 UB 的早期锁定（§17/§77/§90 #4；精度高于 Rust 同期 C-unwind 历史轨迹——定义时点更早、UB 灰区更小——但表达力更窄，仅 abort-on-FFI 一种边界策略，无 Rust extern "C-unwind" 的 opt-in 跨边界继续展开；「优于 Rust 同期」严格指定义时点更早、UB 灰区更小，非表达能力更丰富）、56 关键字精确计数、12 轮深审决议可追溯、交叉引用零悬空。但这些被系统性缺位淹没，无法把规范整体抬升到可独立实现的高度。

---

## 2. 成熟度评分表

| 领域 | 分数(/5) | 对标基准 | 一句话现状 |
|---|---|---|---|
| 形式化层·文法/词法/类型规则 | **3.0** | Go spec EBNF + Rust Reference 词法 + JLS Ch.4/15/18 | EBNF 骨架完整但无控制流体产生式、无字面量词法、无 typing rule；唯一亮点是 56 关键字与三处显式消歧 |
| 动态执行语义 | **2.8** | JLS §15.7 + C++ [intro.execution] + Rust temp scope | 数值语义闭合（§90），但求值序/UB 三分法/并发内存模型/临时值 drop 全空 |
| 并发与内存模型 | **2.5** | Go memory model + C++11 [atomics] + Erlang OTP | API 形态较好（Send/Sync/Channel/Actor），但 happens-before / 原子序零规范，Send/Sync "安全"是工程启发式非语义定理 |
| ABI/FFI/数据布局 | **2.0** | Itanium C++ ABI + Rust Type Layout + Swift ABI Stability | 全规范最薄弱层：无 FFI-safe 白名单、无调用约定、无 size_of/offset_of，§77 lowering 表被错位当 ABI 契约 |
| 基础类型语义 | **2.5** | Rust casts + JLS §5.1 + Zig 四分 + Swift String | 算术结果语义达 Zig/Rust 档，但 cast 算子/字面量词法/string Unicode 三大地基近乎全空 |
| 工具链与包管理 | **2.5** | cargo (lockfile/profile/sigstore) + go modules + npm | 口号层；lockfile/profile/签名/解析算法零匹配，§42 把"安全"等同静态扫描 |
| 错误模型与诊断规范 | **3.5** | rustc --error-format=json + E0xxx + tokio JoinError | 语义层扎实（Result/panic/FFI→abort），诊断层整体缺位且已被 RFC 0003 踩中 |
| 规范治理与方法论（本轮最大增量） | **2.5** | HTML §2 Conformance + ISO Directives + Rust RFCs + Swift Evolution | 治理脚手架近乎全缺；交叉引用零悬空 + 决议追溯是唯一亮点 |
| *补漏：名字解析* | *2.0* | Rust Name resolution + JLS §6.5 + Go scope | 编译器第一个语义 pass 却是最大盲区，§73 仅一行职责 |
| *补漏：借用检查器形式规则* | *2.5* | rustc-dev-guide NLL + Stacked/Tree Borrows + SPARK | 散文陈述借用互斥，无 lattice/fixpoint/可判定性声明 |
| *补漏：常量求值域* | *2.0* | Rust Constant evaluation + C++ [expr.const] | 五处使用"编译期常量"零处定义，const fn / CTFE 整体缺失 |
| *补漏：泛型/Trait 单态化* | *3.0* | rustc_solve + Rust associated types + Swift protocol | 概念骨架（trait/interface 二分、orphan）在，但 impl 搜索算法/关联类型/overlap 全空 |

**加权总成熟度算法**：8 主领域各权重 1.0，4 补漏领域各权重 0.5（补漏为补充视角非主轴）。

- 主领域加权和 = 3.0+2.8+2.5+2.0+2.5+2.5+3.5+2.5 = **21.3**
- 补漏加权和 = 0.5×(2.0+2.5+2.0+3.0) = **4.75**
- 总权重 = 8 + 2 = 10
- 0–5 尺度均值 = 26.05 / 10 = **2.605**
- 归一到 0–10 = **5.21 ≈ 5.2 / 10**

注：本评分（12 维加权）与上轮「四个极致」5.25/10（4 维简单平均：性能 4.75 / AI 友好 6.0 / 可观测 4.0 / 可预期 6.25）采用完全不同的维度划分与聚合方法。两套框架度量的是规范的不同切面（本轮聚焦"能否支撑独立实现"的规范工程成熟度，上轮聚焦"四个极致"的目标达成度），数值接近属巧合，不构成互证；亦无法从两套不同聚合中把差值归因于某一维度。

---

## 3. 核心产出：成熟规范必备章节类型 × AILang 现状对照表

图例：✓ 具备 ｜ ◐ 部分 ｜ ✗ 缺失。补齐成本：**[纸]**=纸面条款可直接补；**[决]**=需在备选中做设计决断；**[RFC]**=需经 RFC 流程（因其影响面或需外部研究）。

| # | 成熟规范必备特征 | 状态 | 证据 § | 补齐成本 |
|---|---|---|---|---|
| 1 | 完整 EBNF（含 if/for/while/loop 体产生式） | ◐ | §27 行 1515 裸终结符、行 1516 `parallel ...` 截断；§87#9 parallel for 表达式化无入口 | [决]（parallel for 表达式化） |
| 2 | 字面量词法（string/int/float/char + 进制 + 转义） | ✗ | §8.6 行 358–360 仅 ident 正则 + 一句"整数无后缀" | [纸] |
| 3 | 形式化 typing rules / 推断算法 / 元性质声明 | ✗ | §15.2 行 510–511 两行散文；全文 Γ/⊢/HM/bidirectional/soundness/decidable 零命中 | [决]+[RFC] |
| 4 | 求值顺序规则（实参/操作数/字面量元素） | ✗ | §27 args/list_lit/struct_init 产生式无序注；全文"求值顺序"零命中 | [纸]（选 left-to-right） |
| 5 | UB / impl-defined / unspecified 三分法 + conformance | ✗ | §90 仅"永不 UB"立场；全文 conformance/MUST/SHALL 零命中 | [RFC] |
| 6 | 并发内存模型（happens-before / 原子序 / synchronizes-with） | ✗ | §18.6 仅 Send/Sync 组合；happens-before 全文零命中 | [RFC] |
| 7 | 用户级原子类型 + 内存序 API | ✗ | §33 std.sync 仅 Mutex/Shared/lock；atomic/Ordering 全文零命中 | [RFC] |
| 8 | ABI / FFI-safe 类型白名单 / 调用约定 / 平台映射 | ✗ | §25.3 仅两示例；§77 lowering 表非 ABI 契约；extern ident 不透明 | [决]+[RFC] |
| 9 | 数据布局控制（packed/align/transparent + size_of/offset_of + 字节序） | ◐ | §25.4 行 1419 仅 layout(C)；size_of/align_of/offset_of/endian 全文零命中 | [决] |
| 10 | 数值转换算子（cast/as / Zig 四分） | ✗ | §27 无 cast 产生式；§11 禁隐式却无显式手段；T(expr) 锁定约束构造 | [决] |
| 11 | 字符串/Unicode 语义（length 单位/char/迭代/规范化/大小写） | ✗ | §36 行 1829 仅"UTF-8"+方法表；char/grapheme/NFC 全文零命中 | [纸]+[决] |
| 12 | 常量求值域 / const fn / CTFE 边界 | ✗ | §13/§15.9/§77/§92#2 五处用"编译期常量"零处定义；const fn/comptime 关键字无 | [RFC] |
| 13 | 名字解析算法（scope 层级 + 点分路径 + .member 查找序） | ✗ | §73 行 2982 一行职责；§12.2 唯一遮蔽规则；"由名字解析消歧"四处黑箱 | [RFC] |
| 14 | 借用检查器形式规则（Borrow Graph / lattice / fixpoint） | ✗ | §18.5/§74.2 两句散文；region/NLL/reborrow/可判定性零命中 | [RFC] |
| 15 | impl 搜索算法 + coherence/overlap + 关联类型投影 | ✗ | §73 TypeDb 无 implements 索引；bound 单层无 super-trait；关联类型机制整体缺失 | [RFC] |
| 16 | 结构化诊断协议（JSON / fix-it / applicability / expected-found） | ✗ | §69.3 五字段仅"彩色打印"；severity 仅 Error/Warning/Note 行内注释 | [RFC] |
| 17 | 错误码注册表 + lint/告警控制 + deprecation | ✗ | §18.2 AIL2xxx TBD、仅 AIL2001 孤例；#[allow]/#[deny]/deprecated 全文零命中 | [RFC] |
| 18 | 工具链（lockfile + build profile + registry 治理 + 可复现构建） | ✗ | §29/§30/§42 口号层；lockfile/profile/opt-level/integrity 全文零命中 | [RFC] |
| — | 规范治理（conformance / normative-informative / RFC lifecycle / glossary / 版本契约 / deprecation） | ✗ | 全文 conformance=0、RFC 2119=0、glossary=0；四 RFC 全 Draft 无转正流程 | [RFC] |

**统计**：✓ 具备 0 项 ｜ ◐ 部分 2 项 ｜ ✗ 缺失 16 项（其中 13 项需 RFC 级工作）。这是本报告的核心结论载体：**18 项成熟规范承重特征，AILang 仅 2 项部分具备、0 项完全具备**。

---

## 4. 按领域展开

### 4.1 形式化层·文法/词法/类型规则（3.0）

**现状**：§27 提供一份看似完整的 EBNF，但无法推导规范自身的 §16 控制流示例（if/for/while/loop 在 stmt 行 1515 是裸终结符、无体产生式），parallel 分支以字面 `...` 截断（行 1516）。词法层 §8.6 仅 ident 正则，所有字面量终结符（string_lit/int_lit/float_lit/char）从未被任何产生式定义。类型系统 §15 是纯散文（语义层），无 typing judgment、无推断算法族命名、未声明 soundness/decidability/principal type。唯一高风险真歧义——小于号 vs 泛型实参——恰好未给消歧规则（§27 call 行 1529 vs §11 level-3 比较算符），而规范对 `!` send、try 块、Variant(args) 三处自造歧义都明确写了消歧。

**对标**：Go spec IfStmt/ForStmt 完整产生式；Rust Reference Lexical structure 全产生式 + `::<T>` turbofish；JLS Ch.4/15/18 typing rules + 可判定推断算法。

**已确认差距**（4 条 high + 1 条 medium）：控制流体产生式缺（high）、字面量词法缺（high）、typing/subtyping 规则缺（high）、`<` 消歧规则缺（high）、string length 单位与整数宽度转换未定义（medium）。

**建议**：§27 补全 if_stmt/for_stmt/while_stmt/loop_stmt 产生式（含 parallel for 表达式入口）；§8.6 扩为完整字面量词法章（参照 Go spec Source code representation）；§27 增一条 `<` 消歧规则（推荐 `::<T>` turbofish）；§15.2 命名推断算法族并声明 decidability。

### 4.2 动态执行语义（2.8）

**现状**：数值语义（§90 #1-#3）做到定义行为永不 UB，是规范层少见的闭合度。但求值顺序全文无一处明文（短路 && / || 由语义内在规定除外）；UB 术语被使用 4 次却从未定义；并发内存模型完全缺失；locals 与临时值 Drop 顺序仅字段有定义（§90 #5），"临时值"一词全文零出现。

**对标**：JLS §15.7 明文左到右并逐条列举；C++ [intro.execution] 序点与三分法；Rust Temporary scopes / Drop scopes。

**已确认差距**（4 条 high + 1 条 medium）：求值顺序未定义（high）、UB 三分法未建立（high）、并发内存模型缺失（high）、locals/临时值 Drop 序未定义（high）、panic 表达式中途展开序未定义（medium）。

**建议**：§11 或 §27 增"求值顺序=左到右"逐产生式条款；§17 或附录增 UB/impl-defined/unspecified 定义段（这是治理项，见 §4.8）；§18.7 扩展 Drop 顺序到 locals（逆声明序）与临时值（构造逆序）。

### 4.3 并发与内存模型（2.5）

**现状**：API 形态较好（Send/Sync 组合、scoped spawn、MPMC Channel、CSP+Actor、select），§87 V 对 executor 做了 10 轮收敛。但地基——并发内存模型——几乎完全空白：happens-before 全文零命中、"数据竞争"仅 4 处口号无形式定义、§18 标题"内存模型"只讲 ownership/stack/heap/COW/drop。连带 Mutex 持锁 panic 语义、隐式 join 子 Task panic 父侧行为、死信、Channel 背压与保序、调度确定性均未定义。

**对标**：Go memory model 8 条 hb 规则；C++11 [intro.races]+[atomics.order] data race 形式定义 + 6 种 memory_order；Rust + Erlang OTP link/monitor/down。

**已确认差距**（4 条 high + 2 条 medium）：并发内存模型/happens-before 全缺（high）、用户级原子类型/内存序 API 全缺（high）、Mutex 持锁 panic 语义 + try_lock/可重入/公平性未定（high）、并发失败可观测性空白带（隐式 join 子 panic 父侧行为/TaskHandle 错误态 API/死信/向已死 actor 投递，high）、Channel 顺序保证与背压语义（medium，其中 FIFO 保序与 OOM 不属已定义结局成立；try_send 可由 select 组合故子点降级）、调度确定性与 parallel/reduce 结果可观测性（medium）。

**建议**：这是规范到编译器落地的最大承重缺口。至少为 Channel send/receive、lock/unlock、spawn-join、actor 投递定义最小 happens-before（可委派 LLVM memory model 但须显式声明）；§18.6 增 Mutex poison/try_lock/可重入决断；§17/§34 定义 TaskHandle 错误态 API（已被 RFC 0003 依赖）。

### 4.4 ABI/FFI/数据布局（2.0，全规范最薄弱层）

**现状**：3507 行无任何独立「ABI/布局」章节。§25.3 仅 extern c/rust 两示例（rust 为空壳），§25.4 仅 layout(C) struct 一例，§77 类型映射表实为 LLVM lowering 被错位当 ABI 契约。无 FFI-safe 白名单、无调用约定（extern ABI ident 完全不透明）、无变参语法、无 size_of/align_of/offset_of、无字节序/目标三元组参数化。规范自己的 §25.3 printf 教学示例（`extern c { fn printf(text: string) }`）在其自身类型模型下即触发 C17 UB：变参未规约 + §77 string=`{i64 len,i8* ptr}` 非 `const char*` + 无 NUL 终结 + 无封送章节。

**对标**：Itanium C++ ABI；Rust Reference Type Layout + 封闭 extern ABI 集合（C/system/stdcall/fastcall/vectorcall/sysv64/win64/aapcs/C-unwind）；Swift ABI Stability + Value Witness Tables。

**已确认差距**（3 条 high + 2 条 medium + 1 条 strength）：FFI-safe 白名单缺（high）、调用约定全未规范（high）、变参未规约且 printf 示例自触 UB（high）、string ABI 自相矛盾（medium）、布局控制仅 layout(C) 单旋钮（medium）、**panic 跨 FFI→abort 非 UB + longjmp 跨入禁（strength；精度高于 Rust 同期 C-unwind 历史轨迹——定义时点更早、UB 灰区更小——但表达力更窄：仅 abort-on-FFI 一种边界策略，无 Rust extern "C-unwind" 的 opt-in 跨边界继续展开 ABI；「优于 Rust 同期」严格指定义时点与 UB 灰区，非表达能力更丰富）**。

**建议**：§25 扩为独立 ABI 章节，定义 FFI-safe 原始类型集 + 调用约定封闭集合 + 平台默认映射；修正 printf 示例（引入 CString/CStr 等价物或标注"示意非规约"）；§25.4 增 packed/align(N)/transparent + size_of/align_of/offset_of 内征。

### 4.5 基础类型语义（2.5）

**现状**：算术结果语义（§90）达 Zig/Rust 档。但三大地基近乎全空：(1) 禁隐式转换（§7/§11）却无 cast 算子（§27 无 cast 产生式、`as` 仅 import 别名、T(expr) 锁定约束构造），八种整数宽度无互通手段；(2) 形式文法引用 string_lit/literal 却无产生式；(3) string 的 length 单位/char 类型/迭代/规范化/大小写映射全部未定义且无 char 类型。

**对标**：Rust Reference Type cast expressions + str 不变式 + char=Unicode scalar；JLS §5.1 转换表 + §3.10 字面量；Zig @intCast/@floatToInt/@truncate 四分；Swift String 字形簇语义。

**已确认差距**（3 条 high + 2 条 medium + 2 条 strength）：字符串/Unicode 语义整层缺位（high）、数值显式转换机制完全缺失（high）、字面量词法未定义（high）、溢出定性 build profile 未定义（medium，注意：wrapping_* 非冗余而是 debug 下抑制 panic 的唯一出口，论断原"自相矛盾"系误读已修正）、UTF-8 有效性不变式 + 索引/切片语义未定（medium）、**整数溢出/除零/NaN 永不 UB（strength）**、**算术结果定性为定义行为（strength，但配套求值序/移位/浮点精度仍缺）**。

**建议**：§27 增 cast 产生式并定义 float→int saturating/窄化截断语义；§8.6 扩字面量词法；§36 增 char 类型 + length 单位决断（推荐字节语义对齐 Rust str::len）+ 大小写映射 Unicode 版本。

### 4.6 工具链与包管理（2.5）

**现状**：§29/§79 仅命令名表（且两表不一致：§29 列 8 命令、§79 列 5 命令），§30.1 ail.toml 仅 [package]+[dependencies]，§42 发布安全仅 4 步静态扫描流程图。cargo/go modules/npm 三大基石——lockfile、build profile、签名验证与 registry 治理、依赖解析算法、可复现构建、增量编译——要么完全缺失要么以单词带过。规范在 §17/§15.1/§77/§90 反复使用"debug 构建""release"却通篇无 build profile 定义。

**对标**：cargo Cargo.lock + [profile.dev/release] + sigstore + MVS；go modules go.sum + 校验和 DB；npm package-lock.json + provenance。

**已确认差距**（3 条 high + 2 条 medium + 1 条 low）：无 lockfile（high）、无 build profile（high，与数值语义悬空直接相关）、§42 发布安全缺签名/出处/registry 治理（high）、§29 vs §79 命令表不一致 + 孤儿命令 ail install/ail fmt（medium，RFC 引入的 ail doc/ail verify 等亦未回写）、无依赖解析算法与冲突策略（medium）、无增量编译/缓存/watch/可复现构建（low）。

**建议**：§29/§79 统一为单一权威命令表并回写 RFC 引入命令；§30.1 增 [profile.dev/release]（opt-level/debug-assertions/overflow-checks）；新增 lockfile 格式（字段+checksum+是否随包发布）；§42 扩签名/provenance/registry 治理或明文推迟并标注 gap。

### 4.7 错误模型与诊断规范（3.5，语义层强、诊断层弱）

**现状**：语义层相对扎实——Result/panic 边界、try/throw 合法性、panic 栈展开与 Task 边界、FFI→abort、确定性 Drop 顺序自洽。但诊断/可观测层整体缺位：§69.3 仅 5 字段且输出方式仅"彩色打印"，无 JSON 协议/fix-it/applicability/expected-found；错误码 §18.2 自承"AIL2xxx 待后续细化"且仅 AIL2001 孤例；TaskHandle 错误态类型/API/死信语义全未定义却被 RFC 0003 依赖；panic 报文格式/退出码/backtrace/hook 全未规定。

**对标**：rustc --error-format=json 约 25 字段 + Applicability 枚举 + E0xxx 注册表；TS fixId+suggestion；Elm 可执行建议；Rust tokio JoinError。

**已确认差距**（4 条 high + 2 条 medium）：无结构化诊断协议（high，AI-first 语言修复闭环承重点）、错误码注册表整体缺位（high）、TaskHandle 错误态/死信全未定义已被 RFC 0003 踩中（high）、panic 符号分类学不完整（unwrap/assert 主导模式无符号，high）、panic 报文/退出码/backtrace/hook 全缺（medium）、Severity 枚举未定义 + 无 lint 控制 + 无 deprecation（medium）。

**建议**：§69.3 扩为 JSON 诊断协议（severity 扩 Help/Hint/Fatal + fix-it + applicability + expected/found）；§18.2 落地 AIL2xxx 编号空间分类规则与稳定性承诺；§17/§34 定义 TaskHandle 错误态 API 与死信；§34 panic 符号表补 unwrap/assert 符号。

### 4.8 规范治理与方法论（2.5，本轮最大增量·上轮完全未碰）

**现状**：成熟规范赖以立足的脚手架近乎全缺。全文无 conformance 条款（何为 conforming 实现）、无 RFC 2119 规范性词汇（MUST/SHALL/MAY 英文零使用，中文"必须"13 处+"不得"1 处均为普通行文）、无 normative/informative 分区（仅 Part 级粗粒度中文标注）、无术语表（glossary）、无语言级版本/稳定性契约（semver 仅施于 schema 与包、主动排除语言本体，"可逆"承诺 18 处却无操作性定义）、无 RFC 生命周期流程（四 RFC 全 Draft 待 review，仓库无 shepherd/FCP/采纳标准/废弃流程文档）。

**对标**：HTML Living Standard §2 Conformance + RFC 2119 + Glossary；ISO/IEC Directives Part 2；Rust RFCs + FCP + shepherd；Swift Evolution；Python PEP 1 + PEP 387。

**已确认差距**（3 条 high + 3 条 medium）：无 conformance 条款（high）、无 normative/informative + RFC 2119 分类（high）、无 RFC 治理流程 + 无 deprecation 策略（high）、无 UB/impl-defined/unspecified 三分法（medium，与 §4.2 的 UB 三分法系同一 gap 的交叉列举：在动态执行语义维度评 high，在规范治理方法论维度评 medium）、无术语表（medium）、语言级无版本/稳定性契约 + "可逆"未定义（medium）。

**建议**：这是本报告认为优先级最高的维度——治理基础设施缺位使其他所有补齐工作都缺乏"规范性约束力"的载体。建议新增 §1.5「Conformance 与规范性词汇」+ 附录 D「Glossary」+ 独立 RFC 流程文档（lifecycle/shepherd/FCP/采纳标准/deprecation 周期）。

### 4.9 补漏：名字解析（2.0）

**现状**：编译器第一个语义 pass 却是最大盲区。§73 仅一行职责，§12.2 仅一条遮蔽规则（本地优先于导入），§27 给三条点分产生式却无逐步解析算法。无作用域层级定义、无点分限定路径解析、无 .member 查找次序（字段/方法/trait/interface/子模块）、无 prelude 名注入层规范、无两遍分析边界。规范四处声明"由名字解析消歧"（§27/§89#2/§92#2）却从不展开，构成"声明依赖黑箱"。

**对标**：Rust Reference Name resolution / Scopes / Preludes；JLS §6.5 Meaning of Name（6 种 NameContext）；Go spec universe/package/file/block 五层。

**已确认差距**（3 条 high + 2 条 medium）：无作用域层级（high）、无点分路径解析算法（high）、无 .member 查找次序与 call 文法消歧（high）、prelude 无名字解析层陈述（medium）、两遍分析签名/体边界未定义（medium）。

**建议**：§12 增 §12.4 Preludes + 作用域层级枚举 + 点分路径逐步解析算法 + .member 查找序。这是单点影响最大的缺口（每个 ident、每个 `std.http.Client`、每个 `a.b()` 都受影响）。

### 4.10 补漏：借用检查器形式规则（2.5）

**现状**：高层语义有真材实料（值语义三分类 §18.4、"不逃逸"简化 §18.5、Phase 增量交付 §82#1）。但形式规则整片空白：借用互斥只用两句散文（§18.5/§74.2），无 typing judgment、无 Borrow Graph 合法性判定算法（无 lattice/transfer function/fixpoint）、无 region/NLL/reborrow（全文零命中）、无可判定性/soundness 声明（"永久正确"是零迁移工程承诺非形式 soundness）、reborrow/不相交字段借用/嵌套借用语义未定义。

**对标**：rustc-dev-guide Borrow checking / MIR dataflow（lattice+fixpoint）；NLL O(n) 算法；Stacked/Tree Borrows；SPARK 所有权 soundness 证明。

**已确认差距**（3 条 high + 2 条 medium）：借用区域与别名约束未形式化（high）、Borrow Graph 合法性判定算法未给出（high）、跨函数借用推导推迟 + 无可判定性/soundness 声明（high）、所有权错误分类学仅 AIL2001（medium）、reborrow/不相交字段借用/嵌套借用未定义（medium）。

**建议**：§74 增 Borrow Graph 形式结构 + 合法性谓词 + 数据流分析传递函数；§18.5 增 reborrow/disjoint field borrow 决断；声明 borrow 分析的可判定性（不逃逸前提下本应 trivial 却仍无承诺）。

### 4.11 补漏：常量求值与编译期求值边界（2.0）

**现状**：最严重的隐蔽断层。§13/§15.9/§77/§92#2/§77 五处（实为六处+）反复使用"编译期常量""编译期整数""常量折叠"却从未定义。无常量表达式文法（§27 无 const_expr，const 声明甚至不在 stmt 产生式中）、无允许操作集、未区分"强制检查"（§15.4 字面量编译错）与"尽力优化"（§77 LLVM 常量折叠 pass）。§15.4 允许约束谓词调 pure fn → 要求编译期求值器能调 pure fn，但无 const fn 关键字、pure 仅保证无副作用不保证终止。const 泛型实参文法锁死为裸 literal（`type_arg := type | literal`），命名 const 与常量表达式均不可用。

**对标**：Rust Reference Constant evaluation（constant expression + const context + promotability + CTFE）；C++ [expr.const] core constant expression；Zig comptime。

**已确认差距**（3 条 high + 1 条 medium + 1 条 inconsistency high）：常量表达式/编译期求值域定义完全缺失（high）、无 const fn/comptime 却要求编译期求值含 pure fn 调用（high）、const 泛型实参求值/统一规则缺失 + 文法锁死裸 literal（high）、前端 CTFE 与后端 LLVM 常量折叠未分离（inconsistency，high→修正后 medium，§77 行 3110 已隐含锚定前端但未声明 as-if）、非裸字面量常量初始化边界悬空（medium）。

**建议**：§15 新增「Constant evaluation」子章（const 表达式文法 + 允许操作集 + promotability）；§9 引入 const fn 关键字 + step limit；§27 扩 type_arg 接受 const 路径与常量表达式；§77 显式声明 LLVM pass 为 as-if。

### 4.12 补漏：泛型/Trait 单态化与 impl 求解（3.0）

**现状**：概念骨架存在（trait→单态化/interface→dyn 二分 §86#10、泛型不变性 §86#9、orphan 一句 §91#9、where bound 文法）。但 impl 求解核心算法几乎全空白：无 impl 搜索/选择算法、无 overlap/冲突检测、bound 文法仅单层无 super-trait、关联类型机制整体缺失（bound 无 `Iterator<Item=T>`、类型语法无 `T::Item`），致运算符 trait 的 rhs/返回类型在规范层不可形式化求解（§34 运算符 trait 表仅方法名无签名，与同节 Optional/Result 表对照证明非偶然省略）。

**对标**：Rust Reference Trait Resolution + rustc_solve 四阶段（候选装配/朴素匹配/确认/投影）+ associated types；JLS §15.12 三阶段方法派发；Swift protocol associatedtype。

**已确认差距**（2 条 high + 2 条 medium + 1 条 low）：impl 搜索/选择算法完全未定义（high）、关联类型概念与投影规则完全缺失致运算符 trait 返回类型不可形式化（high）、overlap/coherence 仅有 orphan 一句 + diamond 歧义无消解（medium）、单态化终止条件与代码共享策略未定义（medium）、trait bound 不满足诊断分类学缺失（low）。

**建议**：§73 增 impl 搜索算法；§15.11 增关联类型语法（`type Output;` + bound `T: Add<Rhs>` + projection `T::Output`）；§86#6 增 diamond 歧义消解规则（C3 线性化或明文禁止）；引入 UFCS 限定语法作为手动消歧出口。

---

## 5. 上轮已发现 vs 本轮新发现

### 5.1 上轮「四个极致」已点出、本轮重申并升级为规范工程视角（5 类）

这些 gap 在 docs/research/deep-review-2026-07.md 已有记录，本轮从"性能/可观测"视角转为"规范能否支撑独立实现"视角重审并确认：

1. **0 用户级原子类型与内存序**（上轮 §56）→ 本轮升级为"并发内存模型整片空白"（§4.3，6 条 confirmed）
2. **Mutex 持锁 panic 三方案未择一**（上轮 §5.3.7）→ 本轮确认并扩展为 Mutex 完整 API 完备性缺口
3. **并发失败可观测性空白带**（上轮 §160，自评"中"）→ 本轮确认 TaskHandle 错误态/死信全未定义且 RFC 0003 已前向依赖（升级为 high）
4. **Channel 默认无界把程序引向无定义失败模式**（上轮 §161）→ 本轮确认 OOM 不属任何已定义结局 + FIFO 保序未定义
5. **包供应链与构建可复现性**（上轮 §179-185，评 1.5/10）→ 本轮确认 lockfile/profile/签名/解析算法零匹配，与语言层钉死确定性构成同文档自相矛盾

### 5.2 本轮首次发现（关键新增）

下列领域是上轮「四个极致」**完全未碰**的，本轮为最大增量：

1. **规范治理整维度**（§4.8，6 条 confirmed）—— conformance / RFC 2119 / normative-informative / glossary / RFC lifecycle / 版本契约全部零命中。这是"规范工程成熟度约当 Rust 2012 年前"判决的关键成因，上轮未触及。
2. **形式化层·typing rules / 字面量词法 / 控制流体产生式 / `<` 消歧**（§4.1，5 条）—— 上轮未做词法/类型规则级审计。
3. **名字解析整维度**（§4.9，5 条）—— 上轮 8 审计域无一直面语言级名字解析。
4. **借用检查器形式规则整维度**（§4.10，5 条）—— 上轮虽提及 borrow checker 但未审形式规则层。
5. **常量求值域整维度**（§4.11，5 条）—— 上轮与四 RFC 从未触及，是规范工程最大的空白盲区。
6. **泛型/Trait 单态化与 impl 求解**（§4.12，5 条）—— 含关联类型机制整体缺失这一比"未定义"更严重的发现。
7. **ABI/FFI/数据布局的 printf 自触 UB**（§4.4）—— 上轮未做 FFI 类型兼容性核查。

---

## 6. 经核查不成立的担忧

**本轮 adversarial 验证后 0 条进入 refuted、0 条留在 uncertain**——即所有 68 条 confirmed 发现（含 3 条 strength）在对抗验证后均成立（其中多条附带"措辞精化/引文订正/严重度微调"修正，但核心结论无一被推翻）。注：68 条系 confirmed 数组原样计数，含 3 个主题（字面量词法、UB 三分法、并发内存模型）在多个维度的交叉列举；若按唯一 gap 主题去重，独立 gap 数少于 68。

代表性的是**若干"自相矛盾"指控在验证中被收窄为"严谨度不一致"或"文档化规则缺失"**，反映了审计的诚实校准：

- 「溢出定性自相矛盾：wrapping_* 族冗余」→ 经核对 §90 #1 与 §15.1 的 debug panic 语义，wrapping_* 恰是 debug 下抑制 panic 的唯一出口，**非冗余、非矛盾**；真正的 gap 收窄为"build profile 未定义"（confirmed 带修正）。
- 「§27 自标骨架与 §71 标权威互相矛盾」→ 核对后 async/await/where/send/spawn 产生式全在 §27 块内，注释"细节见对应小节"指语义/类型规则非文法外推；真问题收窄为"骨架措辞 + 一处省略号 + literal 未定义"（severity 由 medium 下修为 low）。
- 「TaskHandle 仅有 cancelled() 无错误态 API」→ 核对 §34 行 3104 已列 join 操作，论断漏列；但错误态类型/API 仍全未定义，核心 gap 反而因 RFC 0003 前向依赖而更严重。

这类"修正后判决不变"的模式本身说明：规范的局部自洽性比"矛盾"更接近"不完整"，但**不完整的广度与深度足以使其无法支撑独立实现**。

---

## 7. 总结论与 P0 必做清单

### 7.1 把文档补到"可支撑独立实现"的最小必做清单（P0，按优先级排序）

**P0-1｜规范治理地基（先于一切）**—— 新增 §1.5 Conformance + RFC 2119 规范性词汇 + normative/informative 分区 + 附录 D Glossary + 独立 RFC lifecycle 文档。
*理由*：没有 conformance 与规范性词汇，所有后续补齐都无法定义"约束力"——补了 typing rule 也无从说明它是 MUST 还是 SHOULD。这是 §4.8 全维度缺位且为其他补齐的前提。

**P0-2｜形式化层三件套**—— §27 补全控制流体产生式 + §8.6 补全字面量词法 + §15.2 命名推断算法族并声明 decidability + §27 增 `<` 消歧规则。
*理由*：词法器/解析器/类型检查器是编译器前三环，目前三环全卡（§4.1，4 条 high）。无此则独立团队无法开工。

**P0-3｜名字解析算法**—— §12 增作用域层级 + 点分路径逐步解析 + .member 查找序 + §12.4 Preludes。
*理由*：单点影响最大的缺口（每个 ident、每个 `a.b()`、每个 import 都受影响，§4.9），两实现必然分叉。

**P0-4｜求值顺序 + Drop 顺序 + UB 三分法**—— §11/§27 增"求值顺序=左到右"逐产生式 + §18.7 扩 Drop 顺序到 locals/临时值 + §17 增 UB/impl-defined/unspecified 定义段。
*理由*：动态执行语义的最低可裁判框架（§4.2，4 条 high），双实现一致性的前提。多为纸面条款，成本低。

**P0-5｜诊断协议 + 错误码 + TaskHandle 错误态 API**—— §69.3 扩 JSON 协议 + §18.2 落地 AIL2xxx + §17/§34 定义 TaskHandle 错误态（RFC 0003 已依赖）。
*理由*：AI-first 语言修复闭环的承重点（§4.7），且 TaskHandle 缺口已被下游 RFC 踩中形成悬空前向引用。

**P0-6｜数值转换算子 + 字符串 Unicode 语义**—— §27 增 cast 产生式（定义 float→int saturating/窄化）+ §36 增 char 类型 + length 单位决断。
*理由*：禁隐式转换却无 cast 是承重矛盾（§4.5）；string 是最高频类型却 Unicode 语义全空。

**P1（重要但可在 P0 后启动）**：并发内存模型（§4.3）、借用检查器形式规则（§4.10）、常量求值域（§4.11）、impl 求解/关联类型（§4.12）、ABI/FFI 章节（§4.4）、lockfile/build profile（§4.6）。

### 7.2 「先补文档 vs 先动编译器」取舍

**明确建议：先补文档，P0-1 至 P0-6 完成前不启动编译器实现。**

理由：
1. **P0-1（治理）与 P0-2（形式化层）缺位时启动编译器，实现者只能自行发明规范未给的决断**（如 `<` 消歧策略、求值顺序、字面量词法），这些"实现决断"会固化进编译器并被用户依赖，使未来补规范时面临 deprecation 成本——与规范自身"零迁移"承诺冲突。
2. **名字解析（P0-3）与借用检查形式规则（P1）是编译器基础设施**，规范缺位时两实现必然分叉，且分叉点恰在最高频路径（每个 ident、每个借用）。
3. **诊断协议（P0-5）是 AI-first 语言存在理由的承重墙**，先动编译器再补诊断会导致诊断结构碎片化、难以回头统一。
4. **本轮 0 refuted / 0 uncertain 表明所识别的 gap 本身真实存在**（非误报），审计论据扎实。但这仅确认了"缺口存在"，并不等价于"填补缺口无风险"——设计 conformance 条款、typing rules、并发内存模型、关联类型投影或借用检查算法的过程中，可能暴露设计张力、与冻结 v0.2.1 决议的不一致、或需修订既有章节（如 §77 行 3110 const-eval 锚定、§86 决议）。换言之，gap 判决稳健降低了"补了才发现是误报"的概率，但不消除"补的过程中发现需修订既有决断"的风险。
5. P0 清单多为**纸面条款或单一设计决断**（§3 表中 [纸] 与 [决] 类），无需等待外部研究；只有并发内存模型/常量求值域/关联类型等 P1 项可能需 RFC 级研究。

**反证**：若先动编译器，按规范现状实现者将在词法器（字面量词法缺）、解析器（控制流体产生式缺 + `<` 消歧缺）、类型检查器（typing rules 缺 + impl 求解算法缺 + 名字解析缺）、借用检查器（形式规则缺）四个核心 pass 全部卡住或自行发明——发明的结果无法保证双实现一致，违背规范作为"可独立实现契约"的根本目的。

**一句话总结**：AILang 规范当前是"一份有亮点的设计文档"，而非"一份可支撑实现的规范"；距离后者，至少需要完成上述 P0-1 至 P0-6 六项补齐，其中规范治理地基（P0-1）是其余一切的前提。

---

*报告完。全部论据取自 68 条 confirmed 发现（confirmed 数组原样计数，含 3 条 strength 类发现与 3 个跨维度交叉列举主题）；0 条 refuted、0 条 uncertain。§ 引用均指向 docs/AILANG.md（commit 2f63e18 冻结 v0.2.1）或 docs/rfc/0001-0004。*