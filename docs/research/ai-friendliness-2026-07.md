# AI 友好性最大化研究 · 存档

- **日期**：2026-07-14
- **状态**：✅ 研究完成 / 📦 结论存档 / ⏸ 待推进
- **适用范围**：v0.3+ 形式化层前瞻设计 —— **不触动 v0.2.1 冻结规范**（§1–§94、110 项决议不变）
- **来源**：4 个深度调研 agent（共 ~100 次 WebSearch）+ 状态审计，覆盖 10 个方向

> 本文是「以后捡起来」的备忘录。5 分钟读完即可重新进入状态。

---

## 1. 一句话结论

**AI 友好到极致 = Rust/SPARK 线，不是 Dafny 线。**

最强的**自动验证**（decidable + syntactic），不是最强的**类型系统**（dependent + SMT）。判据由一个决定性数字定义：**DAFNYCOMP 3.69%**——LLM 能验证单函数（60–90%），但验证不了多函数系统（<7%）。AILang 的全部设计价值，就是把这个 gap 尽量压窄。

---

## 2. 停止规则（黄金）

新增任何「保证特性」时，**三条任一满足即停**：

1. 验证时间变超多项式
2. 依赖启发式 solver（SMT 漂移）
3. 标注 : 代码 > 1:1

**Rust 正好停在这条线（所以上位），Dafny 越线（所以 niche 14 年）。** 这是字段跑了 40 年、几十次实验的共识。

---

## 3. 重新排序的 Tier 计划

（相对初稿，PBT 提前、SMT 降级、dependent 收窄）

| Tier | 内容 | 风险 | 杠杆 | 备注 |
|---|---|---|---|---|
| **1a** | **doctest / 可执行文档示例**（`///` + `--test`） | **Low** | 高 | 示例即测试 + AI 读示例即知用法；零新语法（`///` 已有 §70/§71）。**最便宜的第一落地项**（§9.1） |
| **1b** | **PBT + 覆盖率引导 fuzzer 进 stdlib** | **Low** | **极高** | 1 PBT ≈ 50 测试；LLM 现在就能生成属性；fuzzing 是不依赖 LLM 的确定性层 |
| **1c** | **契约（requires/ensures/invariant/panic/alloc）进 `.ailmeta`** | Low-Med | 高 | 一切的基础；当前 `.ailmeta` 缺这些字段，是关键缺口 |
| **2** | **窄而 decidable 的 refinement 层**（nullness/bounds/capacity） | Med | 高 | **仅 decidable 片段，不做任意算术**；Flux 式 ~0% 标注推断 |
| **3** | **SMT 验证器作可选 side-car**（SPARK Gold 式 opt-in） | Med | 中 | **永不作默认 typechecker**；外部验证器消费 `.ailmeta`；初期不烤进 `ailc` |

**当前 v0.2.1 的 `.ailmeta` 已有**：type/effect/error/ownership 的机器化语义——这是 Tier 1b 的天然地基。

---

## 4. 实质修正（相对研究初稿的 6 处调整）

1. **➖ 排除全 dependent types** —— Idris/Agda/ATS/F*/Spec# 全 niche 或被砍。原 Tier 2.4「refinement 模拟 dependent」明确收窄为**仅 decidable 片段**。
2. **⬇️ SMT 仅可选层** —— SMT 启发式漂移（「昨天能证、今天不能、代码没动」）是 Dafny 专业用户**头号投诉**（Mugnier/Jhala OOPSLA 2025）。对要调验证器上千次的 AI repair 循环是**致命的**。核心类型系统必须 syntactic + decidable。
3. **➕ 每个验证器必配功能测试夹具** —— RLVR 训练的模型会写 `ensures true` + `assume false` **骗过 solver**（实证 58% → 过滤后仅 31% honest）。测试夹具是防 spec-hacking 的**架构硬要求**，非 nice-to-have。
4. **🎯 跨函数契约机械传播** —— 让 callee 的 postcondition 成一等公民、可机器传递（refinement / capability types），而非每处调用点重新断言。这是堵 **DAFNYCOMP 3.69%** 组合性 gap 的核心设计指令。
5. **⚠️ 纯函数默认目前对 AI 是负面的** —— GPT-5：Haskell 42% vs Java 61%（721 任务）。→ **不走 Haskell 式纯默认**；用 Roc 式显式 `!` 效果标记 + `const fn` 式可选纯化。
6. **⚠️ Ownership 对 LLM 首轮不利，但反馈循环中和** —— RustEvo²：Sonnet 4.5 21%、GPT-5 Codex 28%（首轮）；但 borrow checker 即 oracle，反馈循环收敛任意模型（Hecking-Harbusch 2025）。→ AILang 已有 ownership（被证据支持），**必配 Rust-doctor 级诊断**。

---

## 5. 关键证据数字

| 数字 | 含义 | 来源 |
|---|---|---|
| **DAFNYCOMP 3.69%** | 组合性验证 gap；AILang 要解决的核心问题 | arXiv:2509.23061 |
| **1 PBT ≈ 50 测试** | PBT 变异杀伤力 | OOPSLA 2025，DOI 10.1145/3764068 |
| 单函数 Dafny 86%→29% | 给 spec（仅写证明） vs 全写（NL→code+spec+proof） | arXiv:2503.14183 |
| Verus 15% | Rust 基质、AI 全写——最接近 AILang 的数 | 同上 |
| Haskell 42% vs Java 61% | 纯函数式目前对 AI 是 negative | "Perish or Flourish" 2025 |
| RustEvo² ownership 21–28% | LLM 首轮在 ownership 上崩；反馈循环中和 | Strand tech report 2025-12 |
| AWS 授权引擎 1B/s、零事故 | SMT 验证替代测试的**最强工业证据** | ICSE 2025 Distinguished |
| spec-hacking 58%→31% | RLVR 模型骗 solver，需测试夹具兜底 | arXiv:2605.30914 |
| seL4 ~10× 成本 | 全功能正确性验证的代价上限 | sel4.systems |
| Dafny ~4.8 LoA/LoC | 标注负担 | Furia/Meyer |

---

## 6. 最终稳定论点

> 核心：**affine/ownership + ADT + traits + capability 效果 + 窄而 decidable 的 refinement 层。**
> 再加五个 AI-era 加法：
> ① **结构化验证器协议**（喂 self-repair，不是 Z3 的 opaque "unknown"）
> ② **验证器必配测试夹具**（防 spec-hacking）
> ③ **SPARK 式分层保证**（默认内存安全 → opt-in 无运行时错误 → opt-in 关键模块功能正确）
> ④ **结构化诊断一等公民**（机器可解析错误 + 修复建议，AI 闭环命脉，见 §9.2）
> ⑤ **`meaning` 统一自然语言层**（类型/函数/错误/效果全配语义进 `.ailmeta`，AILang 独有差异化，见 §9.3）

> **⚠️ 一个必须显式的张力**：Kleppmann 2025-12 预测「AI 会让形式验证主流化」，**部分反对**本结论「别把 SMT 作默认 typechecker」。调和——Kleppmann 对一半：AI 确实降低 *writing* tax（Mode-1 Dafny 86% 证此）；但硬数据反对其外推：DAFNYCOMP **3.69%**（组合性崩）、spec-hacking **58%→31%**（验证器可骗）、Mugnier/Jhala（SMT 漂移不可维护）。故本结论取**窄解**：AI 让 *opt-in* 验证层更可用（Tier 3 side-car 受益），但不改变「核心类型系统必须 decidable」的底线。**重评触发条件**：LLM 组合性验证突破 ~50%，或 spec-hacking 出现结构性解法。

---

## 7. 待选的推进路径（未决）

1. 写成 `docs/` RFC（v0.3 形式化层提案）
2. 从 Tier 1a 起步（PBT + fuzzer 进 stdlib）
3. 从 Tier 1b 起步（契约进 `.ailmeta`）
4. 深入辩论某个具体点（如 side-car 验证器架构 / 跨函数契约机械传播）

---

## 8. 关键参考来源

> 按主题分组。带反引号的是可直接检索的标识符（arXiv ID / DOI）。

**LLM 验证能力（核心判据）**
- Shefer et al. 2025 (JetBrains) —— `arXiv:2503.14183`（单函数 Dafny 86%→29%、Verus 15%）
- DAFNYCOMP —— `arXiv:2509.23061`（组合性 3.69%）
- Tan 2026 RLVR spec-hacking —— `arXiv:2605.30914`（58%→31% honest）
- Verina —— ICML 2025（proof 生成是瓶颈）

**LLM 在 ownership / 纯函数上的表现**
- Strand-Rust-Coder tech report 2025-12（HuggingFace blog）—— ownership 微调
- RustEvo² —— `arXiv:2503.16922`（首轮 21–28%）
- Hecking-Harbusch 2025 —— `arXiv:2512.02567`（反馈循环中和难度）
- "Perish or Flourish" 2025 —— `arXiv:2601.02060`（Haskell 42% vs Java 61%）

**PBT / 测试**
- OOPSLA 2025 PBT 实测 —— `DOI 10.1145/3764068`（1 PBT ≈ 50 测试）
- Agentic PBT —— NeurIPS 2025（LLM 生成属性，56% 有效）

**工业证据 / 代价**
- AWS 授权引擎 Dafny —— ICSE 2025 Distinguished Paper（1B/s、零事故）
- Mugnier/Jhala —— OOPSLA 2025 `DOI 10.1145/3763181`（Dafny 用户研究 + SMT 漂移 #1 投诉）
- seL4 —— sel4.systems（~10× 成本）
- CompCert —— compcert.org（~6 人年）
- SPARK Adoption Guidance（AdaCore/Thales）—— 唯一成功上量的验证语言

**类型系统 / 验证技术**
- Flux —— Rust refinement types（~0% 标注推断）
- RefinedRust —— PLDI 2024（ownership 形式化验证）
- Creusot / Kani / Verus —— Rust 验证生态对照

**趋势预测**
- Kleppmann 2025-12「AI 会让形式验证主流化」（via Simon Willison 报道）—— 见 §6 张力声明

---

## 9. 补充方向（覆盖性审查后补入，2026-07-14）

> 以下 3 个方向在首轮 10 方向研究中零覆盖，经覆盖性审查补入。基于已有证据 + AILang 特性判断，**未起 agent 实证**。

### 9.1 doctest / 可执行文档示例

**定位**：文档注释里的可运行代码示例，工具链执行为测试（Rust `rustdoc --test` / Python doctest / Swift·Go example）。AILang 已有 `///` trivia（§70/§71，R16 确认）——doctest 是其**零新语法**延伸。

**对三目标**：
- **减单测**：示例即测试，文档示例自动成回归测试。
- **AI 可预测**：**最强机制**——signature + 一个可运行示例比任何注释都准；「输入→输出」具象实例正中核心诉求。
- **可预测性**：锁定具体输入行为（契约管 for-all，doctest 管 exists）。

**利弊**：利——零额外语法、文档不漂移（doctest 失败=文档过时）、AI 训练数据「signature+示例」海量、落地极简。弊——只覆盖写出的例子（非 PBT 自动生成）、可能脆弱（依赖输出格式）、需 `--test` 工具链。

**建议**：Tier 1a（最便宜第一落地项）。与 PBT（1b）互补：doctest 给具体示例、PBT 给泛化属性。`.ailmeta` 加 `example` 槽位（canonical input→output），AI 直接读到典型用法。

### 9.2 诊断 / 错误消息质量（独立维度）

**定位**：编译器/验证器输出的准确性、定位精度、可解析性、可操作性（修复建议）。

**为什么独立**：首轮被提 4 次但散落。对 AI 友好性，**诊断质量 ≈ 类型系统一半价值**——AI self-repair 闭环全靠解析错误（AutoVerus/Clover 依赖 verifier 反馈；Rust-doctor 存在因 Rust 错误是 LLM 痛点；「self-repair mandatory」的输入就是诊断）。

**AILang 应做**：
1. **结构化诊断协议**——错误是机器可解析结构（错误码 + span + 相关符号 + 建议 diff），非纯字符串。
2. **对齐 `.ailmeta`**——编译失败指回语义事实（「声明 database.read 但调 io.blocking，违 §21.3」）。
3. **suggested fix**——带「did you mean」，AI 可直接 apply。
4. **杜绝 Z3 opaque "unknown"**——SMT 不可解析失败对 AI 致命（又一 SMT 降级理由）。

**利弊**：利巨大（AI 闭环命脉）；弊是持续工程投入，但 Day 1 设计比事后补（Rust 的痛）便宜得多。

**建议**：升格为 AI-era 加法第 ④ 条，横向贯穿所有 Tier。

### 9.3 `meaning` 关键字最大化

**定位**：AILang **独有**——给类型/字段附自然语言语义进 `.ailmeta`。其他研究语言（Dafny/F*/Idris）都没有。`meaning` 填「形式化契约太贵 / 注释太随意」之间的中间层：**机器验证的结构 + 人类/AI 可读的自然语言**。

**对三目标**：
- **AI 可预测**：核心——`type UserId = int` + `meaning: "unique identifier"` 比裸 `int` 多语义层，是「AI 零猜测」的字面实现。
- **减单测**：间接——语义明确 = 更少误解 = 更少测试澄清。
- **可预测性**：`meaning` 是「意图」的机器化表达。

**最大化方向**：
1. **统一自然语言层**——`meaning` 附类型/函数/错误变体/效果/字段，全进 `.ailmeta`。
2. **与弱契约配套**——`meaning UserId: "positive"` 配约束 `int where x > 0`（自然语言提示 + 机器验证）。
3. **AI retrieval 索引**——`.ailmeta` 的 `meaning` 是 AI 检索代码的语义索引。
4. **一致性 lint**——检查 `meaning` 与行为明显矛盾（弱提示，非强验证）。

**利弊**：利——独有差异化、零成本扩展、直击核心卖点。**关键限制**——`meaning` 是自然语言，**编译器只能验证其存在与结构，不能验证其正确性**（与 refinement 的根本区别，不能假装替代验证）。

**建议**：升格为 AI-era 加法第 ⑤ 条，横向贯穿——每个 Tier 的契约槽位都配 `meaning` 自然语言伴随。
