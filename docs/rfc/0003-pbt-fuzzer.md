# RFC 0003 · PBT + 覆盖率引导 Fuzzer —— 属性化测试与元数据化

| | |
|---|---|
| **状态** | 草案（Draft v5）—— 待 review（收敛轨迹 v1=0H/11M/9L[首轮大 RFC：20 确认聚类 12 缺陷]→v2 全修。v1→v2 修复：①§7 推导 emission 误写 `public pure fn`（数据层应为 `public fn`，与 §9.1/§9.2 闸门层/执行层分离一致）；②`source:"requires"`/`"constraint"` 作为独立 `properties[]` 源却无检查语义——`requires`/`constraint` 改为**生成侧元数据**（harness 直读 `contracts.requires[]`/type `constraint` 数组、不入 `properties[]`），与 §1 line 24「`ensures` 是单一属性源」+ §1 line 26「requires→守卫/constraint→不变式」一致；③随②消解 `source:"constraint"` ref 跨条目歧义；④`source:"test"` 手写属性→fn **归属规则**补定义（body 直接调用的 public fn）；⑤手写 body purity 范围「被测 fn」单数→**body 直接调用的所有 fn**（修 extern-frame 安全落空）；⑥§4.2 purity 理由补双重正当性（**确定性是承重**、extern 安全为推论）；⑦§7 constraint 生成删「等价地 `T(expr)` 构造」（no `catch_unwind` 下构造 panic 被误判为属性失败）；⑧§7 自动执行加**参数可生成性**限制（泛型/struct/enum 推迟）；⑨§9.1「非 public 仍可执行」按手写/推导区分；⑩生成器守卫加重试上限（防 hang）；⑪`panic_symbol` 对 assert 失败定保留串 `AssertionFailure`；⑫shrinking 重写为**值最小化**（非 seed；SplitMix64 非单调）+ builtin shrink 候选；⑬多 draw 反例 `input` 结构约定补定义）。**v2→v3 修复**（v2 验证 11 确认聚类 7 缺陷，详 §4.2 #4 / §5 / §6 / §7 / §9.1 / §9.4 / §12 #14）：⑭§9.1「手写属性不入 `.ailmeta`」措辞矛盾（④⑤⑨ 交织遗留）——改为「无独立 `.ailmeta` fn 条目；`source:"test"` 按 §5 归属规则落入所调用 public fn 的 `properties[]`」，与 §5/§9.1 数据层句对齐；⑮§4.2 #4 shrink int 候选 `{x∓1,MIN,MAX}` 非良基→`MIN↔MAX` 震荡 / debug `MAX+1` overflow 非终止——改良基度量（int `|x|`）收缩、**仅递归严格更小候选**、`MIN`/`MAX`/`±inf`/`NaN` 作一次性探针（不递归、wrapping 生成不 panic）；⑯§4.2 #4 补全 `gen_float`/`gen_range` shrink 候选（IEEE 754 特殊值 + 有界 clamp，消「为每个 builtin generator 定义」过强声称的遗漏）；⑰有界生成器（`gen_range`/`gen_int(min,max)`）shrink 未 clamp 到界——候选 clamp 到 `[lo,hi]`/`[min,max]`、`gen_one_of` 仅换更早索引 + visited-set 防回访震荡；⑱§7/§9.4「constraint 包装类型」可生成与 struct/enum 不可生成碰撞（§15.4/§92 #4 允许 constraint 施于 struct）——限定「基础类型可生成」、constraint 施于 struct/enum 按 skip 处理；⑲§5 归属规则「直接调用」未界定运算符解糖/方法调用——归属仅含显式 fn/方法调用、**排除运算符解糖 trait 方法**（purity 侧含、由 §19 闭包保）；⑳§6 推导属性（`source:"ensures"`）反例 `input` 结构未定义——补按 fn 形参名为 key）。**v3→v4 修复**（v3 验证 7 确认聚类 4 缺陷，皆在 §4.2 #4 shrink 段 + §12 #14）：㉑§4.2 `gen_float` 的 ±inf/NaN 误置于递归候选花括号内（与 §12 #14/header ⑮「四者皆一次性探针」三处矛盾）+ float 良基度量未定义——移 ±inf/NaN 入一次性探针 `{±inf,NaN,MIN,MAX}`、度量枚举补「float 按 |x|」、并明确「所有候选（含探针）均重放测试一次、度量筛选只决定是否递归」（消「丢弃 vs 重放」语义歧义）；㉒`gen_list` 逐元素递归 shrink 由度量暗示但候选枚举未列 + §4.2 与 §12 #14 list 度量不一致（前者「长度+递归每元素」、后者「长度」）——枚举「删元素 i + 对保留元素 i 应用子生成器候选」、两处统一为「长度+递归每元素」（string 仅长度）；㉓`gen_one_of` shrink「换更早索引」未界定值如何产生 + 排除生成器内 shrink（单元素 `gen_one_of([g_0])` 零候选）——补「生成器内 shrink（应用 g_i 自身候选）+ 更早索引探针（g_j 最小规范值，visited-set）」；㉔推导属性 shrink 重放未界定 requires/constraint 违例候选处理（违例致 fn 入口 requires 断言 panic→误记失败→伪反例）——定「域外候选空虚真、跳过不重放不计失败不递归」+ constraint 包装类型复用基础类型候选）。**v4→v5 修复**（v4 验证：5 维度、2 原始 finding 聚 1 处、皆被对抗验证否决为非缺陷——判「已由『最小规范值』原则蕴含、且 gen_one_of 非唯一（gen_list→[] 等复合同等隐式可推断）」；仍显式化以消实现者读歧义）：㉕§4.2 #4 `gen_one_of` 一次性探针「g_j 首候选」对嵌套 gen_one_of 未显式递归定义（internal-consistency + completeness 两独立维度同提）——显式化为「首候选 = g_j 能产生的最小规范值、与运行时值 v 无关」、枚举全部 9 个 builtin 的首候选、复合生成器（gen_one_of）递归取其首子生成器 g_0 的首候选（生成器描述符树有限故良基终止）；与 gen_list→[]/gen_string→\"\" 等复合一并列出、不单列 gen_one_of，§12 #14 同步）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94、56 关键字、110 决议不变）|
| **日期** | 2026-07-15 |
| **承接** | [`docs/research/ai-friendliness-2026-07.md`](../research/ai-friendliness-2026-07.md) §3 Tier 1b（「PBT + 覆盖率引导 fuzzer 进 stdlib」，风险 Low / 杠杆「极高」）+ [RFC 0001](./0001-doctest.md) §11#5（doctest 作 PBT 种子）+ [RFC 0002](./0002-contracts.md) §10#3（契约→PBT 推导接口）+ §2.3 line 39（`properties[]` 已被命名为共享 schema bump 第 3 rider）|

---

## 1. 动机

研究存档 §3 把 **Tier 1b = 「PBT + 覆盖率引导 fuzzer 进 stdlib」** 定为 **Low 风险 / 杠杆「极高」**，两条量化锚定：「**1 PBT ≈ 50 测试**」（OOPSLA 2025，DOI 10.1145/3764068）与「**LLM 现在就能生成属性**」「**fuzzing 是不依赖 LLM 的确定性层**」。它补全「意图表达」元数据的第五层，承接 RFC 0001 §1 / RFC 0002 §1 的层级表：

| 层 | 表达力 | 形式 | AILang 载体 | `.ailmeta` 槽位 |
|---|---|---|---|---|
| **doctest**（RFC 0001） | exists（具体实例） | `///` 内可运行块 | DocComment | `examples[]` |
| **constraint**（§15.4） | 类型级谓词（refinement） | `constraint T { expr* }` | 独立声明 | `constraint` 字段（line 1350） |
| **契约**（RFC 0002） | for-all（运行时前置/后置断言） | `requires` / `ensures` 块（§20） | 契约块 | `contracts` |
| **PBT / fuzz（本 RFC）** | **for-all（经验量化属性）** | **`test` 块 + `std.testing` generator** | **`test` + `std.testing`** | **`properties`（本 RFC 新增）** |
| **SMT side-car**（Tier 3） | for-all（逻辑证明） | 外部验证器消费 `contracts`/`properties` | — | （Tier 3，不在本 RFC） |

**关键概念**：`ensures` 是**单一属性源**（推导属性的唯一来源），它允许两个互补消费者——(a) Tier 3 SMT **逻辑地**消费它（for-all 证明），(b) 本 RFC **经验地**消费它（for-all 检查：在 N 个生成输入上跑 `ensures`）。§7 的推导接口正是逻辑 for-all 与经验 for-all 之间的桥。故 for-all 有两层：**契约 = 逻辑 for-all**（Tier 3 证明 `∀x·requires(x)⇒ensures(x)`）、**PBT = 经验 for-all**（本 RFC 在满足 `requires` 的生成 `x` 上检查 `ensures`）。`requires`/`constraint` 不是属性源——它们是**生成侧守卫/不变式**，塑形输入生成（§7），不进 `properties[]`。

本 RFC 同时兑现两个显式 handoff：**RFC 0001 §11#5**（doctest 示例作 PBT generator 种子）+ **RFC 0002 §10#3**（契约→PBT 属性机械推导接口：`ensures`→属性 / `requires`→生成器守卫 / `constraint`→类型生成器不变式）。并落地研究 §4#3（测试夹具是**架构硬要求**，防 spec-hacking；RLVR 骗 solver 实证 58%→过滤后 31% honest）与 §6① / §9.2（结构化、机器可解析的反例喂 self-repair，非 Z3 式 opaque「unknown」）。

> **Tier 顺序引用 §3**（非 §7）。研究存档 §7「待选推进路径」误标 tier（line ~93 称「Tier 1a = PBT+fuzzer」「Tier 1b = 契约」），与 §3 权威表（1a=doctest / 1b=PBT+fuzzer / 1c=契约）冲突——本 RFC 一律引 §3。

---

## 2. 设计目标

1. **零新关键字**（保 F6 的 56 不变）—— `property`/`prop`/`forall`/`fuzz`/`random` 在 v0.2.1 **全部缺失且不可加**。属性表面**复用已有 `test` 硬关键字**（§9 line 387「测试」类别、§27 line 1502 `test := "test" string_lit block`）+ `std.testing`/`std.random` **模块路径 ident**（§91，非关键字）。`std`/`testing`/`draw`/`gen_int` 均为 ident，`--fuzz`/`--prop`/`--seed` 为 CLI flag——**零新词法，56 不变**。
2. **零新文法产生式**—— 不引入尾随块组合子（`for_all(gen){…}`）、不新增 `stmt` 产生式。全部手写属性形式由 §27 **既有**产生式组合：`test string_lit block`（line 1502）、`let`（line 1511）、`postfix`（line 1528：`primary` `.`ident `.`ident `(`args?`)`）、`call`（line 1529-1530）、`assert` 调用形（§17）。**不触动 §27**。
3. **与 RFC 0001/0002 共享 schema bump**—— `properties[]` 是 `schema_version` 0.2.0 → 0.3.0 的**第 3 rider**（RFC 0002 §2.3 line 39 已命名「将来 1b `properties[]`」）；**不另声明 bump**，避免碎片化。先例：§88 #4/#5（line 3318-3319）以**可选字段**加 `declarations[]` 与 per-entry `span`，未破坏冻结状态。
4. **确定性可复现**—— 属性运行的**唯一**不确定性是**记录在案的 seed**。seeded PRNG 为 stdlib 构造（无新关键字），**不依赖** `std.time`/`std.crypto`（两者皆 v0.2 占位，§33 line 1684-1685 / §33.1 line 1693-1695）。默认 seed = 固定常量（零熵原语，开箱即复现）。
5. **`ail test` 单一收集入口**—— `--prop` / `--fuzz` / `--seed` 是 `ail test` 的**模式/flag**，非新命令（承 RFC 0001 §6 单入口先例）。

---

## 3. 语法（双通道，零新关键字 / 零新产生式）

### 通道 (i)：手写属性 = 复用 `test` 块

**决定性冻结事实**：**v0.2.1 无一等闭包**（§21 line 1038「控制构造块捕获（v0.2 无一等闭包）」、line 1033「v0.2 无闭包依赖」）。因此 QuickCheck/Haskell 式组合子 `for_all(gen, |x| { … })` 在 v0.2.1 **不可能**——它需要闭包字面量文法（缺失）或新增尾随块 `stmt` 产生式（违「不触动 §27」）。

**无闭包的干净形态**：**一个属性 = 一个 `test` 块，其 body 恰是一次 trial；runner 循环 N 次，每次 trial 在独立 Task 里、从 per-trial seeded RNG 抽值**。「谓词」字面就是 `test` 块的 body（pure fn 调用 + `assert`）——不存在「谓词作为值」需要传递。

```ail
test "reverse is involutive" {
    let x = std.testing.draw(std.testing.gen_int())
    assert(reverse(reverse(x)) == x)
}

test "addition is commutative" {
    let a = std.testing.draw(std.testing.gen_int())
    let b = std.testing.draw(std.testing.gen_int())
    assert(add(a, b) == add(b, a))
}
```

**文法溯源**（每个 token 均映射到既有产生式，零新增）：
- `test "…" { … }` → §27 line 1502。
- `let x = …` → §27 line 1511（`stmt := "let" ident (":" type)? "=" expr`）。
- `std.testing.draw(…)` / `std.testing.gen_int()` → §27 `postfix`（line 1528）：`primary`（ident `std`，line 1535）`.`ident（`testing`，line 1532）`.`ident（`draw`）`(`args?`)`（call，line 1530）。`std`/`testing`/`draw`/`gen_int` 为 §91 模块路径 ident，**非关键字**。
- `assert(…)` → §17 调用形 `assert(cond)` / `assert(cond, msg)`（与 RFC 0001 §3 强制的调用约定一致；§41 line 1992 的宏形 `assert result.is_ok()` 是预存冻结 spec 笔误，RFC 0001 §12 line 223 已标注，本 RFC 不沿用）。

**判别（runner 侧静态扫描，非文法）**：runner 把一个 `test` 块当作**属性**（循环 N trial、每 trial 独立 Task）**当且仅当其 body 含至少一处 `std.testing.draw(…)` 调用**。无 `draw` 的 `test` 块仍按原样跑一次（§41 既定语义不变）。概念依据：「属性 = 参数化于生成输入的测试」。runner 对「形似属性」（提到 `gen_*`）却无 `draw` 的块发 hint。

**生成器 = 描述符值**（无用户函数组合）：`gen_int()` / `gen_int(min, max)` / `gen_bool()` / `gen_float()` / `gen_string()` / `gen_byte()` / `gen_range(lo, hi)` / `gen_list(elem_gen, max_len)` / `gen_optional(gen)` / `gen_one_of([gen, …])`。`std.testing.draw(gen)` 是**唯一** effectful 点（从 per-trial seeded RNG 物化一个值）。**v0.3.0 仅 builtin 原语 + 结构组合子**；用户自定义 `map`/`filter` 组合子延后 v0.4（需一等函数值/闭包，§21；见 §12 #2）。

### 通道 (ii)：契约推导属性 = 零源码语法

镜像 RFC 0002 模式：编译器从 `ensures` **推导**属性元数据（`requires`/`constraint` 作**生成侧守卫/不变式**、非属性源，§7），emit 进 `.ailmeta properties[]`（§5）。**零源码语法新增**——用户只写契约（v0.2.1 已有，§20），编译器把 `ensures` 镜像为属性。此通道使 PBT 自动化：带 `ensures` 的 `public pure fn`（且参数类型 v0.3.0 可生成，§7）在 `ail test --prop` 下**免费**获得一个被检查的属性。推导协议见 §7。

---

## 4. 语义

### 4.1 seeded RNG 与 purity 边界

**RNG 设计**：seeded PRNG 类型 housed 在 `std.testing`（§41；独立 `std.random` 模块留 v0.4，见 §12 #1）：`std.testing.Rng`、`Rng::from_seed(seed: int) -> Rng`、`rng.next_u64() -> int`（effectful——变更内部状态）。采用**可分裂**确定性生成（SplitMix64 式）：`seed_i = split(base_seed, trial_index)` 是 `(base_seed, i)` 的**纯函数**，故每个 trial 的 seed 独立可复现、无需重放先前抽值。**注**：SplitMix64 输出对 seed 非单调——故 shrinking 作用于**生成值**而非 seed（§4.2 #4）。

**purity 边界（承重）**：生成器状态变更是 effect。解决：**seeded RNG 完全活在 test/harness 作用域，永不进 §19 effect 推断的生产 `fn` 体**——因为它只出现在 `test` 块（测试代码，不受生产 `fn` 的 effect 闸门约束；`test` 是顶级 item、非 `fn`、无 `is_pure`/`effects` 标注）。purity 闸门约束的是**property body 直接调用的所有 fn**（property body 里调用的函数），**不是** harness：

- **生成器抽值（effectful）**：限于 `std.testing.draw` / runner，在 `test` 块作用域。
- **property body（pure fn 调用 + `assert`）**：body **直接调用的所有 fn** 须 `is_pure: true`（§19 line 951 传递闭包保被调者之被调者亦 pure、故校验直接调用集即充分；§85 #9 line 3264；§9.2）。

如此「生成器 effectful、property body pure」被**钉在 property body 的调用集**上干净满足，且**不向 §19 effect 词表新增任何 effect**（无 `random` effect；§88 #6 line 3320 词表不动）。

**默认 seed 策略（确定性默认）**：因 `std.time`/`std.crypto` 为 v0.2 占位（§33.1 line 1693-1695）、零熵原语，**默认 base seed = 固定常量**（开箱即完全确定性、完全可复现）。可经 `--seed <N>` CLI flag 与 `AIL_TEST_SEED` 环境变量覆盖。**失败 seed-stamp**：任一属性失败，runner 打印并写一份回归 fixture，含 `{base_seed, failing_trial_index, derived_seed, minimized_input, panic_symbol}`——以 `--seed <base_seed>`（及记录的 trial index）重跑可确定性复现原始失败。

### 4.2 每 trial 一 Task 的失败捕获（无 `catch_unwind`）

v0.2.1 **无 `catch_unwind`**（§89 #3 line 3334；§17 line 736）。唯一可观测 unwind 边界是 **Task**（§17 line 736「展开至最近的 Task 边界或 `extern` 帧……Task 内 panic 终止当前 Task → TaskHandle 错误态」）。故**每个 PBT/fuzz trial 跑在独立 Task**：

1. runner 算 base_seed（默认常量或 `--seed`）。
2. 对 trial `i ∈ 0..N`：`derived_seed = split(base_seed, i)`；`spawn` 一个 Task（§21.8 line 1034 scoped spawn、§87 #4）——其 body 设 per-Task trial seed（`std.testing.__set_trial_seed(derived_seed)`）后跑 property body 一次；runner join TaskHandle。
3. TaskHandle 错误态 ⇒ 该 trial 失败 ⇒ runner 记 `(base_seed, i, derived_seed)`，跳出转 shrink。
4. **Shrink（值最小化，非 seed，良基可终止）**：在独立 Task 里对失败输入的**生成值**做最小化。按**良基度量**单调收缩：int/float 按 `|x|`、list 按「长度（主）+ 递归每元素（次）」、string 按长度、optional `Some→None`、bool `true→false`。每个 builtin generator 的候选分两类——**递归候选**（度量严格小于当前失败值者）与**一次性探针**（边界值）；**所有候选（含探针）均重放测试一次**（推导属性的 `requires`/`constraint` 违例候选例外、见末段域外规则），度量筛选**只决定是否递归**（不决定是否测试），保证有限终止。候选枚举：`gen_int`→递归候选 `{0, x/2（向 0 整除）, 向 0 邻居（x>0 取 x−1 / x<0 取 x+1）}`、一次性探针 `{MIN, MAX}`（wrapping 算术直接生成、不递归、永不 panic）；`gen_byte`→同 `gen_int` clamp 到 `[0,255]`；`gen_float`→递归候选 `{0.0, x/2.0, 向 0 邻居}`、一次性探针 `{±inf, NaN, MIN, MAX}`（IEEE 754 特殊值，仅测试不递归、不经度量筛选）；`gen_range(lo,hi)`/`gen_int(min,max)`→`gen_int` 递归候选 clamp 到界（clamp 后落在界内且严格更小者为递归候选、clamp 到端点者归探针）、一次性探针 `{lo, hi}`；`gen_bool`→递归候选 `{false}`；`gen_list`→递归候选 `{[]（空）, 删除元素 i（∀i）, 对保留元素 i 应用其子生成器 shrink 候选（∀i）}`（删元素减长度、缩元素减元素度量，皆严格更小）；`gen_optional`→递归候选 `{None}`；`gen_string`→递归候选 `{""（空）, 截短}`；`gen_one_of`→递归候选 = 所选生成器 g_i **自身的 shrink 候选**应用于值 v（生成器内最小化，覆盖单元素 `gen_one_of([g_0])` 情形）、一次性探针 = 每个更早索引 g_j（j<i）的最小规范值（g_j 首候选 = g_j **能产生的最小规范值、与运行时值 v 无关**：gen_int→0、gen_bool→false、gen_byte→0、gen_float→0.0、gen_string→\"\"、gen_list→[]、gen_optional→None、gen_range/gen_int(min,max)→clamp(0,lo,hi)、**gen_one_of→其首子生成器 g_0 的首候选（递归；生成器描述符树有限故良基终止）**），visited-set 防回访。逐候选重放 trial，仅递归「严格更小且仍失败」的递归候选，至无候选失败即最小。**推导属性（`source:"ensures"`）域外候选**：shrink 候选可能违例 `requires`/`constraint`（如 `requires{x>10}` 失败值 100 的候选 0、constraint `PositiveInt{value>0}` 失败值 50 的候选 0）；此类候选视为**域外、属性空虚真**——**跳过（不重放调 fn、不计失败、不递归）**，避免违例输入致 fn 入口 `requires` 断言 panic（§7 / RFC 0002：requires 违约仍于入口 panic）被误记为属性失败、产出伪反例；constraint 包装类型复用其基础类型的 builtin shrink 候选，违例 constraint 的候选按上述域外规则跳过。SplitMix64 输出对 seed **非单调**（§4.1），故**最小化作用于值、非 seed**——`derived_seed` 仅供**定位/复现失败 trial**（重放 `derived_seed=split(base,i)` 确定性重现原始失败输入）。最小化输入 + `(base_seed, i, derived_seed)` 进回归 fixture。

**为何 property 执行限 pure fn（双重正当性）**：(1) **确定性（承重）**——seeded-RNG 模型与 shrinking（#4）预设「seed → 生成输入 → pass/fail」映射可复现；仅当被测 fn 无 IO/状态/墙钟效应（即 pure）时，同 seed 重放才忠实现原始输入与结果、shrink 重放才健全。非 pure fn 使重放非确定、shrink 失真。(2) **extern-frame 安全（pure 的推论）**——pure ⟹ effect 集空（§19/§85 #9）⟹ trial 调用路径无 `extern`/`unsafe`/IO ⟹ trial panic 恒展开至 Task 边界（TaskHandle 错误态，§90 #4 line 3354 / §17 line 736）、**永不进程 abort**。(2) 单独**不足以**论证 purity：一个带 `database.read` 效应但内存 mock 实现的 fn 无 extern 帧、panic 仍安全，却非 pure、重放非确定——故**确定性是承重理由、extern 安全为其推论**。`test` 块作用域本身只调 `std.testing.draw`/`assert`（std、非 extern）。purity 校验作用于 **property body 直接调用的所有 fn**（§9.2 / §10）。

panic 符号报告取自 §34 表（line 1758-1764）：`ArithmeticOverflow` / `DivideByZero` / `IndexOutOfBounds` / `ConstraintViolation`；**`unwrap`/`assert` 失败无 §34 符号、却是 PBT 主导失败模式**，反例中以**保留串 `AssertionFailure`** 标记（§9.6 / §6）。counterexample 输出会点名终止该 trial Task 的 panic 符号。Drop 顺序 = 声明顺序（§90 #5 line 3355），利于复现。

---

## 5. `.ailmeta` schema 扩展（核心）

`function` 条目新增**可选** `properties` 字段——**`properties[]` 是共享 `schema_version` 0.2.0 → 0.3.0 bump 的第 3 rider**（RFC 0001 `examples[]` + RFC 0002 `contracts` + 本 RFC `properties[]`），**不另声明 bump**（§24 line 1332 schema_version 为独立 semver；RFC 0002 §2.3 line 39 已命名；§88 #4/#5 line 3318-3319 可选字段先例）。

```json
"properties": [
  { "source": "test", "name": "reverse is involutive",
    "span": { "file": "list.ail", "line_start": 12, "col_start": 5, "line_end": 14, "col_end": 1 },
    "generators": ["std.testing.gen_int()"] },
  { "source": "ensures", "ref": 0,
    "span": { "file": "list.ail", "line_start": 8, "col_start": 13, "line_end": 8, "col_end": 57 } }
]
```

**`properties` 条目**：`{source, span, …}`（**ref 引用、DRY，不复制 expr**）。
- `source`（必填，string）：`"test"`（手写属性）｜`"ensures"`（推导属性）。**仅此两种**——`requires`/`constraint` 是**生成侧元数据**、**非独立属性源**（§7：harness 直接读 `contracts.requires[]` / type `constraint` 数组塑形输入，不进 `properties[]`；与 §1 line 24「`ensures` 是单一属性源」一致）。
- `span`（必填，§24 line 1339 `.ailmeta` span 形 `{file,line_start,col_start,line_end,col_end}`，概念锚 §69.1）：`source:"test"` → 该 `test` 块 span；`source:"ensures"` → 源 `ensures` 表达式 span（同 RFC 0002 `contracts.ensures[].span` 逻辑）。
- `source:"test"`：`name`（`string_lit`，必填）+ `generators`（可选 `string[]`，生成器描述符源码串；空则省略该键——同 RFC 0002 §3 line 84「子键仅非空时写」）。
- `source:"ensures"`：`ref`（必填，int）= 索引进**本条目**的 `contracts.ensures[]`（RFC 0002）。**不复制 `expr`/`olds`**——消费者解引用 `contracts.ensures[ref]`；其 `olds` 已在 `contracts.ensures[ref].olds`（RFC 0002 §3），零重复编码。

**为何 ref 而非复制**：RFC 0002 已存契约表达式为原始串（`contracts.ensures[].expr`）。本 RFC **不重存**——推导属性是**引用**进 `contracts`（DRY，避免同一表达式两份漂移）。手写属性携带 test 标签 + 生成器摘要（body 即 `test` 块本身、经 `span` 指向源码，同 RFC 0001 `examples[].code` 存原始块的先例）。`properties[]` 因此紧凑、单一真源。

**`source:"test"` 归属规则**（手写属性是顶级 `test` item，§27 line 1502，不附属于任何 fn）：一条手写属性进 `properties[]` 的归属 = **其 body 中每个被直接调用的 `public` fn 各得一条 `source:"test"` 条目**（编译器解析 body、收集直接调用；间接/传递调用不计——其纯性另由 §19 传递闭包保）。**「直接调用」精确界定（归属侧 vs purity 侧不同）**：(归属) **归属目标仅含显式自由函数调用 + 方法调用 `receiver.f()`**，**排除运算符解糖产生的 trait 方法调用**（`==`→`Eq.eq`、`+`→`Add.add`、`<`→`Ord.lt` 等，§11 line 422 / §34 line 1766 运算符 trait 表 §86 #8；仅 `&&`/`||`/`!` 为内置短路、非调用）——它们是运算表面、非被测 fn，否则连本 RFC 示例 `assert(reverse(reverse(x))==x)` 中的 `==` 也会被误归属到 `Eq.eq` 产生无意义条目；(purity) **purity 校验侧（§4.2 / §9.2 / §10）的「直接调用集」含**运算符解糖与方法调用——其纯性由 §19 传递闭包保，故校验显式直接调用集即充分。body 调用 0 个 public fn（仅测私有 fn 或内置短路运算）⇒ 该属性**仅执行**（runner 仍扫描顶级 draw 块、跑 N trial）、**不入任何 fn 的 `properties[]`**。例：`test "reverse is involutive" { … assert(reverse(reverse(x))==x) }` → 归 `reverse`（`==` 解糖不计入归属）；`test "addition is commutative" { … assert(add(a,b)==add(b,a)) }` → 归 `add`。

**存在规则**：`properties` 当且仅当函数有**至少一条手写属性（含 `draw` 的 `test` 块、按归属规则落本 fn）或至少一条 `ensures` 块**时出现——同 RFC 0001 `examples[]` / RFC 0002 `contracts` 的 **absent-if-none** 先例（保 `.ailmeta` 紧凑）。**注**：仅有 `requires` 或仅有 `constraint`（无 `ensures`、无手写属性）的函数**不产生** `properties[]`——二者是生成侧守卫、非可检查属性（§7）。`public` 闸门见 §9。

---

## 6. 工具链

`ail test` 仍是单一收集入口（RFC 0001 §6）。新增模式/flag：

| 命令 | 行为 |
|---|---|
| `ail test` | 收集 `test` 块（含 property 块）+ doctest，统一执行；property 块按 N-trial/Task 模型跑 |
| `ail test --prop` | 仅跑 property 块（手写 `draw` 块 + 推导属性） |
| `ail test --fuzz` | 覆盖率引导 fuzz 模式（见下） |
| `ail test --seed <N>` | 覆盖默认 base seed（可复现） |
| `ail test --runs <N>` | 覆盖默认 trial 数（property 模式） |
| `ail test --no-doc` / `--no-prop` | 跳过 doctest / property（RFC 0001 §6 风格） |

### 覆盖率引导 fuzzer（`ail test --fuzz`）—— 工具链/stdlib 特性，非语言特性

- **插桩**：编译器在 `--fuzz`/`--coverage` 构建下 emit 边覆盖计数（SanitizerCoverage 式）。被测 fn 编入边计数器；harness 读取之。
- **可插拔 draw 后端**：fuzz 模式下 `std.testing.draw(gen)` 从 **fuzzer 控制的变异缓冲**（libFuzzer 式）读，而非 seeded RNG。**同一 property body 即 fuzz target**——每个手写属性自动成为 fuzz 目标。
- **种子语料**：(a) doctest examples（RFC 0001 §11#5 handoff——从 `examples[].code` 解析字面输入）、(b) 推导 `ensures`/`requires`/`constraint` 边界值（定义域边界）、(c) `tests/corpus/<property>/` 语料目录、(d) 既往回归 fixture（最小化反例）。
- **变异策略**：bit/byte flip、边界整数（0/1/-1/MIN/MAX）、算术微调、长度边界（0/1/MAX_LEN）、从 doctest 字符串字面量提取的字典 token。
- **预算**：`--fuzz-iterations N`（默认如 100 000）与/或 `--fuzz-time T`（墙钟），先到先停。
- **确定性默认 seed**：fuzz 模式仍 stamp 一个 base seed（默认常量）使运行可复现；`--seed` 覆盖。
- **结构化反例输出**（研究 §6① / §9.2——机器可解析、喂 self-repair）：失败时 emit JSON 记录：
  ```json
  { "property": "reverse is involutive", "mode": "fuzz",
    "base_seed": 12648430, "trial_index": 4821, "derived_seed": 9821,
    "input": { "x": -2147483648 }, "minimized_input": { "x": 0 },
    "panic_symbol": "ArithmeticOverflow", "coverage_edges_hit": 17,
    "span": { "file": "list.ail", "line_start": 12, "col_start": 5, "line_end": 14, "col_end": 1 } }
  ```
  **`input`/`minimized_input` 结构约定**（多 draw，研究 §6① 核心）：按 **trial 内 draw 的执行顺序**为槽位——`let`-绑定 draw 用其 `let` 名作 key（如 `{"a":…,"b":…}`），内联 draw（无 `let` 名）用保留序号 key `_draw_0`/`_draw_1`；控制流条件 draw 只记**实际执行**的 draw。多变量示例（commutativity 反例）：
  ```json
  { "property": "addition is commutative", "mode": "prop",
    "base_seed": 12648430, "trial_index": 137,
    "input": { "a": 2147483647, "b": 1 }, "minimized_input": { "a": 1, "b": 1 },
    "panic_symbol": "ArithmeticOverflow",
    "span": { "file": "math.ail", "line_start": 20, "col_start": 5, "line_end": 23, "col_end": 1 } }
  ```
  **推导属性（`source:"ensures"`）反例 `input`/`minimized_input` 结构**：推导属性无源码 `draw`/`let`（harness 按 fn 形参类型生成、调 fn、查 `ensures`，§7），故其反例 `input` 按被测 fn 的**形参名**为 key（如 `fn add(a: int, b: int)` → `{"a": …, "b": …}`），与手写约定并列、同进回归 fixture（研究 §6① 结构化反例因此覆盖推导属性这一自动化主通道）。
  `panic_symbol` 取自 §34 表（line 1758-1764）；**`assert`/`unwrap` 失败**（PBT 主导失败模式、无 §34 符号）以**保留串 `AssertionFailure`** 标记（非新 §34 符号、仅反例约定，§9.6）；`span` 取自 §69.1 / §24。此即研究 §6① 要求的可操作、非 opaque 反例（对照「Z3 opaque unknown」）。

---

## 7. 契约 / doctest → PBT 机械推导（备料）

**定位：备料 + 经验执行，非逻辑证明。** 镜像 RFC 0002 §6「备料，不传播」分裂：本 RFC 让契约数据**现在即可经验检查**；SMT **逻辑证明**引擎属 Tier 3（研究 §3，opt-in、永不作默认 typechecker）。for-all 分裂：契约 = 逻辑 for-all（Tier 3 证明）、PBT = 经验 for-all（本 RFC 检查）、`ensures` 是共享桥（同一 `expr` 串，RFC 0002 `contracts.ensures[].expr`，喂两个消费者）。

### 推导规则

| 契约源 | 推导为 | 机械规则 |
|---|---|---|
| `ensures { E }`（§20 line 982） | **property（唯一可检查属性源）** | harness 生成满足 `requires`、尊重 `constraint` 的输入、调用前快照 `old(e)`（§20 line 990、§85 #5 line 3260）、调 `f`、求值 `E`、`assert(E)`。`old()` 实参式读自 RFC 0002 `contracts.ensures[].olds[]`。→ emit 进 `properties[]`（`source:"ensures"`，§5）。 |
| `requires { R }`（§20 line 978） | **生成器守卫（非独立属性）** | 生成输入后预检：若 `R` 假，**拒绝并重新生成**（只测良构调用——绝不生成违反 `requires` 的调用，否则入口 panic 报伪失败）。**重试有界**：连续拒绝达上限（默认 1000）⇒ 该属性 skip + 诊断「`requires` 过严、生成器命中率低」（§9.6）。`requires` **不进 `properties[]`**——harness 直接读 `contracts.requires[]`（RFC 0002）。 |
| `constraint T { P }`（§15.4 line 540） | **类型生成器不变式（非独立属性）** | 生成 constraint 类型 T 的值时**预检谓词 P**（绑定 `value`，§15.4 line 548 / §92 #4）：只接受满足 P 的基础值，否则重新生成（同 `requires` 守卫、同有界重试）。**禁用 `T(expr)` 构造路径生成**——其违约 panic `ConstraintViolation`（§15.4 line 547 / §92 #2），在 trial Task 内 panic 会被误判为属性失败、且无 `catch_unwind` 不可恢复重试（§89 #3）。`constraint` **不进 `properties[]`**——harness 直接读 type 条目 `constraint` 数组（§24 line 1350）。 |
| doctest `examples[]`（RFC 0001） | fuzz 种子语料 + PBT 回归 fixture | RFC 0001 §11#5 handoff：每个 example 的字面输入 seed fuzz 语料、钉已知良构用例。 |

**只有 `ensures` 产生可检查属性（进 `properties[]`）**；`requires`/`constraint` 是**生成侧元数据**，由 harness 直接从 `.ailmeta` 的 `contracts.requires[]` / type `constraint` 数组读取、塑形输入生成，**不作为独立 `properties[]` 条目**（与 §1 line 24「`ensures` 是单一属性源」、§1 line 26「requires→守卫 / constraint→不变式」一致）。故仅有 `requires` 或仅有 `constraint`（无 `ensures`、无手写属性）的函数**不产生推导属性**。

### 自动 vs 手动 —— 决断：皆自动（数据 + 执行），手动 API 延后

- **自动（数据层）**：编译器为每个带 `ensures` 块的 `public fn` emit 推导属性进 `properties[]`（通道 ii，§5）。用户零操作。此即**备料**——数据就位。（emission 闸门 = `public`，§9.1；与执行层 pure **分离**，承 RFC 0002 §7#2 闸门层/数据层区分——非 pure 的 `public fn` 的推导属性仍入 `.ailmeta`、执行 skip。）
- **自动（执行层）**：`ail test` / `ail test --prop` 自动跑所有有推导属性的 `public pure fn` 的推导属性——**且所有参数类型须 v0.3.0 可生成**（§3 builtin 原语 + list/optional/one_of + constraint 包装类型**且其基础类型 v0.3.0 可生成**）。**constraint 可施于任意已命名 type**（§15.4 / §92 #4：constraint 谓词绑定 `value`、可用 `value.field`，故可施于 struct 等字段型 type）——故「constraint 包装类型」为可生成**当且仅当其基础类型本身可生成**：constraint 施于 struct/record/enum 时按基础类型不可生成处理（推导属性入 `.ailmeta` 但 skip 执行，§9.4 OUT）。其余不可生成参数（泛型 `T`/`where`-bound、`borrow_mut` 接收者——纯 fn 不修改参数故后者多属 moot）的推导属性**入 `.ailmeta`（数据层）但标 `[skipped: param type not generable in v0.3.0]` 不执行**（§9.4 OUT）。用户零操作。
- **手动（可选）**：`std.testing.from_contract("fn_name")` 显式以自定义生成器/seed 调一个推导属性——**延后 v0.4**（v0.3.0 仅 auto，保表面最小）。
- **引擎止于何处**：**数据**（推导 `properties[]`）+ **经验执行**现在即有；**逻辑传播/证明** = Tier 3 side-car（研究 §3、§4 #4）。镜像 RFC 0002 §6 line 123「备料，不传播」。

### 反 spec-hacking（研究 §4#3：RLVR 58%→31% honest）

空属性（`ensures true`、`requires true`）是 spec-hacking 向量。防御，具体：

1. **doctest 夹具锚定（架构硬要求）**：每个有推导属性的 `public pure fn` **应**有 doctest examples（RFC 0001）作具体夹具。runner 报 **fixture-coverage 指标**（有 ≥1 doctest 的推导属性占比）。低 fixture 覆盖 ⇒ warning。此即研究 §4#3 架构要求。
2. **`requires` 作守卫使定义域宽**：空 `requires` ⇒ 生成器覆盖整个输入域 ⇒ 更易撞真实边。
3. **覆盖率引导 fuzz 报边覆盖**：空 `ensures true` 仍执行 fn；fuzz `coverage_edges_hit` 指标暴露欠执行 fn（属性在、仅 3 条边命中 = 红旗）。
4. **软 lint**：编译器对 `ensures true` / 字面 `true` 的 requires **warn**（低置信属性），契合 RFC 0002「描述性字段不静默吞垃圾」的精神。
5. **结构化反例喂 self-repair**（研究 §6①）：失败为机器可解析 JSON（§6），LLM 修复环拿到可操作信号，非 opaque pass/fail。

净效果：（具体夹具 + 宽域生成 + 覆盖率指标 + 结构化反例）使 `ensures true` 低价值且可见——直击 58%→31% spec-hacking 发现。

---

## 8. 与现有机制的衔接

| 现有机制 | 衔接方式 |
|---|---|
| `test` 块 + `assert`（§27 line 1502、§17、§41 line 1987） | property 复用 `test` 块作 body、`assert(cond)` 调用形（与 RFC 0001 §3 一致）；无第二套断言/测试体系。 |
| `std.testing`（§41） | 扩展为 generator + seeded-RNG 宿主（§4.1）；`assert` 亦暴露于此（§34 line 1754）。 |
| 契约 `requires`/`ensures`（§20） | `ensures`=**唯一属性源**（进 `properties[]`）、`requires`=生成器守卫、`constraint`=类型生成器不变式（§7；三者中**仅 `ensures` 进 `properties[]`**）。与 RFC 0002 `contracts` **共享 expr**（`properties[]` 以 `ref` 引用 `ensures`）。 |
| `pure` / effect 系统（§19、§85 #9） | 仅 property body **直接调用的所有 fn** `is_pure: true` 可被 property 执行（确定性 + extern-frame 安全，§4.2）；RNG 效果限 test 域、**不入 §19 词表**。 |
| panic 模型（§17 line 736、§89 #3、§90 #4） | 每 trial 一 Task；panic → TaskHandle 错误态（不触 extern → 不 abort）；panic 符号报自 §34 表（assert 失败用保留串 `AssertionFailure`）。 |
| Task / spawn（§21.8 line 1034、§87 #4） | 每 trial `spawn` 一 Task、join 观测 TaskHandle；scoped 结构化作用域。 |
| `schema_version` bump（§24、RFC 0001/0002） | `properties[]` 共享 0.2.0→0.3.0 bump（第 3 rider）。 |
| `examples[]`（RFC 0001） | doctest 作 fuzz 种子语料 + 回归 fixture（RFC 0001 §11#5 handoff）。 |
| `.ailmeta` 槽位（RFC 0002 §5） | 扩为五槽位：examples / contracts / constraint / meaning / **properties**。 |
| 静态管线（§70/§71 → §73 → §74 → §19） | property body 经完整管线（与 RFC 0001 §8 line 169 一致）。 |

---

## 9. 限制（首版 v0.3.0）

为控制首版复杂度，设以下限制（后续版本可放宽）。

1. **输出范围 = item 级 `public` 闸门（闸门层）**：`properties[]` 进 `.ailmeta` **仅由 item 级 `public` 决定**——仅 `public` 函数的 `properties[]` 进 `.ailmeta`。**手写 `test`-block 属性**是顶级 item、本身无独立 `.ailmeta` fn 条目，`ail test` **仍执行**（runner 扫描顶级 draw 块、与可见性无关）；其 `source:"test"` 条目按 §5 **归属规则**落入 body 直接调用的各 `public` fn 的 `properties[]`（调用 0 个 `public` fn 时仅执行、不入任何 fn 的 `properties[]`）；**推导属性**仅对 `public fn` 生成（§7），故非 public fn 的 `ensures` 不被推导为属性、不被 PBT 执行——「仍可执行」仅适用手写通道。闸门对 `properties[]`、`examples[]`、`contracts` **一致放行/阻断**——杜绝**闸门层**非对称（承 RFC 0002 §7#2 闸门层/数据层区分）。**数据层**：`properties[]` 当且仅当函数有手写或 `ensures` 推导属性（§5 存在规则），与 `examples[]`/`contracts` 各自 absent-if-none 独立——故一个有属性无 doctest 的 `public` 函数可 `properties[]` 在 `examples[]` 缺（数据层非对称合法）。§91 #2 line 3366 默认 `private`。
2. **执行范围 = `is_pure: true` 且非 async（执行层）**：property **执行**（`ail test`）额外要求 **property body 直接调用的所有 fn** `is_pure: true` 且非 async（§19 line 951 传递闭包保其传递被调者亦 pure、故校验直接调用集即充分；§4.1）。这与 RFC 0001 §7.1 doctest 执行判据（effect 谓词）**同构**——emission 由 `public`、execution 由 effect/pure（两层独立，承 RFC 0002 §7#2）。非 pure 或 async 的 `public` fn：推导 `properties[]` **仍入 `.ailmeta`**（数据/文档价值），但 `ail test` 标 `[skipped: fn not pure]` / `[skipped: async harness TBD]` 不执行。
3. **IN（v0.3.0）**：pure fn；手写 `test`-block 属性（`std.testing.draw`）；`std.testing` seeded-RNG（SplitMix64 式 `split`）；**值最小化 shrinking**（对生成值、非 seed；builtin shrink 候选，§4.2）+ seed 可复现失败 trial；`ail test --fuzz` 覆盖率引导；推导 `properties[]` 入 `.ailmeta`；结构化 JSON 反例。
4. **OUT（v0.4+）**：
   - **effectful/async fn property 测**：需 effect-injection/mock 缝（冻结 spec 静默）→ 延后。
   - **trait/interface 方法属性**（`sigs[]`）：与 RFC 0002 §7#1 OUT 联动——`sigs[]` 表示模型 v0.2.1 §24 未定义 → 延后。
   - **不可生成参数类型的推导属性执行**（泛型 `T`/`where`-bound、struct/record/enum、及 constraint 施于不可生成基础类型者）：v0.3.0 generator 仅覆盖原语 + 结构组合子 + constraint 包装类型（**基础类型可生成**者；constraint 生成是 §7 harness 内部逻辑、非 §3 用户面 generator）；这些参数形状的自动 generator 派生延后 v0.4——其推导属性**入 `.ailmeta`（数据层）但 skip 执行**（§7 自动执行层）。
   - **用户自定义 generator 组合子**（`map`/`filter`）：需一等函数值/闭包（缺失，§21 line 1038）→ 延后。
   - **穷举枚举**（有界域 for-all）：延后。
   - **自动 SMT discharge**：Tier 3 side-car（研究 §3）。
5. **`should_panic` 类比**：预期 panic 的 property body（如测 `ConstraintViolation` 路径）——marker 延后 v0.4；v0.3.0 property body 断言成功。
6. **panic 符号 / 反例标记**：不引入新 §34 panic 符号（property 内的契约/约束违约复用现有 §34 符号；`ContractViolation` 与契约 requires 违约的区分是 RFC 0002 §7#5 / §10#1 的 open item，非本 RFC）。**`assert`/`unwrap` 失败**（无 §34 符号、却是 PBT 主导失败模式）在反例 JSON 中以**保留串 `AssertionFailure`** 标记——此为**反例输出约定**、**非新增 §34 panic 符号**（§4.2 / §6）。**生成器守卫重试上限**：`requires`/`constraint` 守卫连续拒绝达上限（默认 1000）⇒ 该属性 skip + 诊断（§7，防过严谓词致 hang）。

---

## 10. 诊断与容错

- **静态管线**：property body 经完整 **§70/§71（lex+parse）→ §73（typecheck）→ §74（borrow）→ §19（effect）** 管线（与 RFC 0001 §8 line 169、RFC 0002 §8 line 150 同一管线）。类型/借用/effect 错误照常报。
- **纯度校验**：property body **任一直接调用 fn** 非 pure（`is_pure == false` 或 `effects != []`）⇒ property 标 skip + 诊断（列出违规 fn 名；执行层信息，非失败；承 RFC 0001 §7.1 skip 语义）。§19 传递闭包保被调者之被调者亦 pure，故校验直接调用集即充分。
- **生成器校验**：`draw(gen)` 的 `gen` 类型与所绑定 `let` 类型须一致（§73）；不符报类型错。
- **失败诊断**：property 失败 ⇒ 结构化反例（§6 JSON）+ 源 span（§69.1 / §24）+ panic 符号（§34；`assert` 失败用 `AssertionFailure`）。**非 Z3 opaque「unknown」**（研究 §9.2）。
- **三类退出语义**（承 RFC 0001 §8）：编译错（非零）/ 运行时 panic（非零，报 seed+input+span）/ skip（零退码）。

---

## 11. 备选方案（及为何不选）

- **A. 新关键字 `property`/`forall`/`fuzz`**：显式可读，但**违 F6（56 不变）**且违约束 #1（唯一自由槽是 `test`）。**不选** —— 复用 `test` 块 + `std.testing`。
- **B. QuickCheck 式 `for_all(gen, |x| …)` 组合子**：表达力强，但**需闭包字面量或尾随块产生式** —— v0.2.1 无一等闭包（§21 line 1038），新产生式违「不触动 §27」。**不选** —— 单 trial `test` 块 + runner 循环（§3）。
- **C. 独立 `prop` 声明 / 独立 `.prop.ail` 文件**：分离清晰，但增语法表面 + 新关键字风险 + 文档漂移。**不选** —— `test` 块内嵌（同 RFC 0001 §10 B/C 论证）。
- **D. 仅 derived（无手写）**：最简，但放弃 `ensures` 难表达的元属性（如 `reverse ∘ reverse == id` 这类无自然后置条件的代数性质）。**不选** —— 双通道（§3）。
- **E. 仅手写（无 derived）**：漏掉自动属性，降低「1 PBT≈50」杠杆、不兑现 RFC 0002 §10#3 handoff。**不选** —— 双通道。
- **F. 基于 `std.time`/硬件熵的真随机种子**：违确定性 + `std.time` 为占位（§33.1 line 1693）。**不选** —— 固定默认 seed + `--seed` 覆盖。
- **G. `properties[]` 自包含（复制 expr 而非 ref）**：自包含，但与 `contracts` 重复、易漂移、bloat `.ailmeta`。**不选** —— ref 引用（§5）。
- **H. 向 §19 词表加 `random` effect**：使 `draw` 经隐式 state effect 让调用者非 pure。**不选** —— RNG 效果限 test 域、不入词表（§4.1）；`draw` compiler-gate 到 test 块（§12 #4）。

---

## 12. 未决问题（待 review）

1. **RNG 宿主**：`std.testing`（v0.3.0 推荐，免触 §33 模块树）vs 独立 `std.random` 模块（v0.4，非测消费者出现时）。
2. **用户自定义 generator 组合子**（`map`/`filter`）：v0.3.0 仅 builtin（推荐，闭包缺失）vs 依赖 fn-name-as-value（§73 gap）。
3. **手写 property 形式**：单 trial `test` 块 + runner 循环（推荐，1:1 映射 per-trial Task）vs `for x in std.testing.samples{}`（同 Task 多 trial、失首轮隔离）。
4. **`std.testing.draw` 作用域**：compiler-gate 到 test 块（推荐，免造隐式 effect）vs 经隐式 state effect 使调用者非 pure（备选 H）。
5. **`properties[]` derived 条目**：ref 引用 `contracts.ensures[]`（推荐，DRY；**v2 定**：仅 `ensures` 产生推导属性，`requires`/`constraint` 为生成侧元数据、不进 `properties[]`）vs 自包含复制。
6. **property 判别**：`draw` 调用检测（推荐）vs 显式 marker（如 `test "[prop] …"` label 约定）。
7. **per-trial seed 派生算法**：SplitMix64（推荐，O(1) 可分裂、纯函数 `split(seed,i)`；**输出非单调故 shrink 取值非 seed**，§4.2）vs PCG vs 域特定。
8. **`std.testing.from_contract` 手动 API**：v0.3.0 仅 auto（推荐）vs 同时 ship 手动调用。
9. **effectful/async fn property 测缝（v0.4）**：mock/effect-injection 模型待冻结 spec 明确。
10. **trait/interface 方法属性（v0.4）**：与 RFC 0002 §7#1/§10#7 的 `sigs[]` 表示模型联动。
11. **fixture-coverage 阈值**：多低触发 warning？（默认 warn-on-zero 推荐）。
12. **fuzz 语料目录约定**：`tests/corpus/<property>/`（推荐）vs `ail.toml` 配置。
13. **手写属性→fn 归属规则（v2 决断）**：body **直接调用的 public fn**（§5）；备选曾为 `///` trivia 附着 / 模块级 `properties` 容器——前者无位置约束（`test` 是顶级 item），后者破 per-function 模型，均不取。
14. **shrinking 机制（v2→v5 决断）**：**值最小化**——按良基度量（int/float `|x|`、list 长度+递归每元素、string 长度、optional `Some→None`、bool `true→false`）分**递归候选**（度量严格更小者，仅这些递归）与**一次性探针**（`MIN`/`MAX`/`±inf`/`NaN`/有界端点 `lo`/`hi`，仅重放测试不递归、wrapping 生成不 panic）；**所有候选均测试一次、度量筛选只决定递归**；有界生成器 clamp 到界；`gen_list` 删元素 + 逐元素 shrink、`gen_one_of` 生成器内 shrink + 更早索引探针（**探针首候选 = g_j 最小规范值、与运行时值 v 无关、复合生成器递归取其 g_0 首候选**；visited-set）；**推导属性域外候选（违例 requires/constraint）空虚真、跳过不计失败不递归**（避伪反例）；候选补全 `gen_float`/`gen_range`。保有限终止（§4.2 #4）。seed 仅复现失败 trial（**非**最小化途径，SplitMix64 非单调）。
15. **生成器守卫重试上限（v2 决断）**：连续拒绝默认 1000 ⇒ skip + 诊断「`requires`/`constraint` 过严」（§7 / §9.6，防 unsatisfiable 谓词致 hang；类比 fuzz `--fuzz-iterations` 预算）。
16. **`assert`/`unwrap` 失败 panic 标记（v2 决断）**：反例保留串 `AssertionFailure`（反例输出约定、非新 §34 符号，§9.6；与 RFC 0002 §7#5/§10#1 的 `ContractViolation` 符号 open item 并列、各自独立）。

---

## 13. 不触动 v0.2.1（合规声明）

- **不改 §1–§94 任何条款**（§19 effect 词表、§20 契约、§24 schema、§27 文法、§33 模块树、§34 panic 符号、§41 test runner —— 均只读引用）。
- **不增关键字**（56 不变）：`property`/`prop`/`forall`/`fuzz`/`random`/`draw`/`gen_*` 均非关键字——`test` 是 v0.2.1 **已有**硬关键字（§9 line 387，已计入 56）；`std`/`testing`/`draw`/`gen_int` 为 §91 模块路径 ident；`--fuzz`/`--prop`/`--seed`/`--runs` 为 CLI flag。
- **不增文法产生式**：全部手写 property 形式由 §27 既有产生式组合（line 1502 `test`、1511 `let`、1528 `postfix`、1530 `call`、1532 `member`、1535 `primary`）。
- **不改 110 项决议**：尤其不动 §88 #6 effect 词表（**不增 `random` effect**——RNG 效果限 test 域、不入词表）、不动 §89 #3 panic 模型（复用 Task 边界，不引 `catch_unwind`）、不动 §90 #4（复用 panic-unwind-to-Task 语义）。**不增 §34 panic 符号**——`AssertionFailure` 是反例 JSON 输出约定串、非 panic 符号。
- PBT/fuzz 是 **v0.3+ 特性**；本 RFC 为**提案**，经 review + 批准后进 v0.3 规范（届时 §24 schema 增 `properties[]` 字段描述、`std.testing` 扩展 generator/RNG API、`schema_version` 共享 bump 0.2.0→0.3.0）。
- 本 RFC 不修改 `docs/AILANG.md`（同 RFC 0001 §12 / RFC 0002 §11 定位）。
- **schema 协调**：RFC 0001 `examples[]` + RFC 0002 `contracts` + 本 RFC `properties[]` 共享**同一次** 0.2.0→0.3.0 bump（RFC 0002 §2.3 line 39 已命名 `properties[]`）—— 向后兼容（皆可选字段）。
