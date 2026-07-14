# RFC 0001 · Doctest —— 可执行文档示例

| | |
|---|---|
| **状态** | 草案（Draft v6）—— 待 review（v1→v2：65→21；v2→v3：10→8+1；v3→v4：5→4 含 1 规范冲突；v4→v5：5→2 含 1M/1L——unsafe/extern effect 归类据 §19 修正；v5→v6：2→2（0M/2L）——§7.1 `unsafe fn` 措辞据 §27/§25.1 修正、should_panic×skip 交集按方向 a 闭合，均已处理；终轮验证 0H/0M/0L） |
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94、56 关键字、110 决议不变）|
| **日期** | 2026-07-14 |
| **承接** | [`docs/research/ai-friendliness-2026-07.md`](../research/ai-friendliness-2026-07.md) §9.1（Tier 1a，最低风险第一落地项）|

---

## 1. 动机

研究存档 §9.1 的结论：**doctest 是「AI 友好性最大化」Tier 1a——最低风险、最高性价比的第一落地项**。一石三鸟：

- **示例即测试**：文档里的可运行示例自动成为回归测试 → **减少手写单测**。
- **AI 读示例即知用法**：signature + 一个可运行示例比任何注释都准；「输入→输出」具象实例正中「AI 零猜测」核心诉求。
- **文档不漂移**：doctest 失败 = 文档过时 → 编译/测试期强制文档与代码一致。

它和契约（Tier 1c，for-all 规范）、PBT（Tier 1b，泛化属性）形成互补三层层级：

| 层 | 表达力 | 形式 | AILang 载体 |
|---|---|---|---|
| **doctest**（本 RFC） | exists（具体实例） | `///` 内可运行块 | DocComment + `.ailmeta` `examples[]` |
| 契约（§20） | for-all（前置/后置条件） | `requires`/`ensures` | 契约块 + `.ailmeta` |
| PBT（Tier 1b） | for-all（量化属性） | 属性 + generator | `std.testing` 扩展（后续 RFC） |

---

## 2. 设计目标

1. **零新关键字**（保 F6 的 56 不变）—— 复用 `///` trivia；`should_panic` 等是 fenced block 的 **info-string 标记词，非语言关键字**。
2. **复用 §71 附着机制**—— 连续 `///` 拼成 DocComment 附着下一声明，doctest 是其子结构。
3. **与 §17/§41 `test`/`assert` 一致**—— 同一套断言（`assert` 为 std 函数，**调用形** `assert(cond)` / `assert(cond, msg)`）、同一测试入口。
4. **example 进 `.ailmeta`**—— AI 直接读到典型输入/输出用法。
5. **`ail test` 统一收集**—— 不引入新命令。

---

## 3. 语法

在 `///` 文档注释内写带 `ail` info string 的 fenced code block，块内用 `assert` **函数调用**：

```ail
/// 返回两数之和。
///
/// 基本加法
/// ```ail
/// assert(add(2, 3) == 5)
/// assert(add(-1, 1) == 0)
/// assert(add(0, 0) == 0)
/// ```
pure fn add(a: int, b: int) -> int {
    a + b
}
```

**fenced block 词法：**
- 起始 ` ``` `，info string 首个 token 必须为 `ail`（语言标识）；其后可跟空格分隔的**标记词**（首版仅 `should_panic`）。形如 ` ```ail ` 或 ` ```ail should_panic `。
- 非 `ail` 起头的 fenced block 视为**纯展示**——不解析为 doctest、不执行、不入 `.ailmeta`。

**assert 调用形（合规关键）：**
- 块内断言必须用 **函数调用语法** `assert(cond)` 或 `assert(cond, msg)`，与 §17（`assert(cond: bool, msg?: string)`）/ §27（调用产生式 `postfix "(" args? ")"`）一致。
- **不得**用宏式 `assert <expr>`（如 `assert add(2,3) == 5`）——那需要 §27 新增 assert 语句产生式，违「不触动 §27」。详见 §12 关于 §41 预存笔误的说明。
- 预期 panic 的块用 `should_panic` 标记：块体执行若 panic → 通过；正常返回 → 失败（演示 constraint 违约等错误路径，见 §7.4）。

**附着规则：**
- 复用 §71：连续 `///`（不被空行打断）→ DocComment 附着到紧邻的**下一个声明**，fenced block 是 DocComment 的子结构（**零新附着规则**）。
- 一个声明可有**多个** fenced block（多个示例）。

**DocComment 文本切分（summary vs description）：**
- 紧邻 fenced block **上方**的连续 `///` 文本行 = 该 block 的 `summary`。
- 其余 `///` 文本行（不紧邻 block）= 声明的 `description`（进 §8.5 / §92#8 描述槽）。
- 即：紧跟 block 上方的文字归示例，其余归声明描述。

**首版可执行附着范围（白名单）：**
- 首版 doctest **仅 `function` 声明可执行**。
- 其余 `item`（§27 中 `function` 以外的全部 item kind —— 含 struct/enum/type_alias/interface/trait/error/meaning/constraint/agent/tool/actor/server/task/test/extern）及字段/参数上的 `ail` fenced block：首版视为**纯文档**——`ail doc` 展示、`ail test` 不执行、不入 `.ailmeta` `examples[]`。
- 后续版本（v0.4+）按类扩展执行模型，见 §7.2 / §11。

---

## 4. 语义

- **作用域**：doctest 在被附着 function 的可见作用域内执行——可直接调用该函数，**无需 import**。首版仅 function 可执行（类型/非函数声明 doctest 为纯文档，见 §3 白名单与 §7.2）。
- **执行**：`ail test` 收集所有 doctest + `test` 块，统一执行（见 §6）。
- **失败**：`assert(cond)` 返回 false → panic（§17 / §89#3 确定性栈展开），诊断报告 **doc 源码定位**（指向 `///` fenced block 的 span）。`should_panic` 块正常返回（未 panic）→ 同样判失败。
- **隔离**：每个 doctest 独立作用域，互不污染（块内 `let` 不泄漏）。
- **块内允许**：`let`、`match`、调用其他函数——doctest 是完整代码块，非单行。

---

## 5. `.ailmeta` 扩展（schema semver bump：v0.2 → v0.3）

`function` 条目新增**可选** `examples[]` 槽位：

```json
{
  "identity": { "name": "add", "description": "返回两数之和。" },
  "input": [
    { "name": "a", "type": "int", "mode": "copy" },
    { "name": "b", "type": "int", "mode": "copy" }
  ],
  "output": { "type": "int" },
  "is_pure": true,
  "examples": [
    {
      "summary": "基本加法",
      "code": "assert(add(2, 3) == 5)\nassert(add(-1, 1) == 0)\nassert(add(0, 0) == 0)"
    }
  ]
}
```
> 上例省略 `span` 字段以省篇幅；规范要求每个 example 条目含 `span`（见下）。

**`example` 条目**：`{summary?, code, span, expect_panic?}`。
- `summary`（可选）：fenced block 上方紧邻的 `///` 文本行（切分规则见 §3）。
- `code`（必填）：fenced block 内容（原始字符串）。
- `span`（必填）：`///` fenced block 的源码定位。
- `expect_panic`（可选，bool）：`should_panic` 标记块为 true。按 §7.4 方向 a，`should_panic` 免疫静默 skip、总尝试执行——故该示例要么已被 `ail test` 执行并验证 panic 触发，要么因缺执行环境（fixture / async harness）而**判失败**（非零退码），**不会**出现「`expect_panic: true` 却从未执行且零退码放行」的虚假声称。

**填充闸门（`public` 优先，保高信噪比）：**
- **`public` 函数**：`examples[]` 进 `.ailmeta`（对外 API 表面，AI 消费目标）。
- **非 `public` 函数**：doctest 仍可写、`ail test` 仍执行，但**不入 `.ailmeta`**（内部实现细节）。
- **该闸门作用于 item 级**：以被测 **function 声明本身**是否 `public` 为准；方法/字段的可见性组合（如 `public` struct 的 `private` 方法）不改变 item 级闸门，细则留 §11。

---

## 6. 工具链

| 命令 | 行为 |
|---|---|
| `ail test` | 收集 `test` 块 **+ doctest**，统一执行（默认含 doctest） |
| `ail test --no-doc` | 跳过 doctest，只跑 `test` 块 |
| `ail doc --test` | 只跑 doctest（`ail test --doc` 别名） |
| `ail doc` | 生成文档，doctest 作为「示例」区块展示（不执行） |

**退出码**：任何 doctest 编译错 / assert 失败 → 非零退出（CI 可用）；完整三类语义见 §8。

---

## 7. 限制（首版 v0.3.0）

为控制首版复杂度，设以下限制（后续版本可放宽）：

1. **执行判据（effect 谓词，精确化）**：**本判据适用于非 async function**（async 函数一律见 §7.3）。`ail test` 执行满足以下**任一**的非 async function doctest：
   - `is_pure == true`；**或**
   - `effects == []`（`effects` 数组为空，即无显式 IO/状态效果）。
   - **关键**：`alloc` 是隐含效果、**不进 `effects` 数组**（§19 / EFF-1 决议）。故「返回堆类型但无 IO」的 `public` 函数（如 `fn make() -> Box<T>`）doctest **仍可执行**，不会被静默 skip。
   - 不满足者（含 `database.read` / `filesystem.write` / `network` / `unsafe` / `extern` 等任一显式效果）：doctest 仍可写；若 `public` 则入 `.ailmeta`（文档价值）；`ail test` 标记 **`[skipped: needs fixture]`** 不执行。按 §19，**仅 `alloc` 不进 `effects` 数组**（隐含、无信息量）；`unsafe`（§25）、`extern`（FFI，§85#4）均进数组，故**含 `unsafe` 块的函数**（§25.1：AILang 无 `unsafe fn` 声明形，`unsafe` 仅作块语句）/ 含 FFI 函数一律命中本分支 skip，无需特判。（**例外**：`should_panic` 块免疫本节静默 skip，见 §7.4 方向 a。）
   - **闸门一致性**：上述所有「入 `.ailmeta`」均以 §5 的 `public` 闸门为前提（即 `public` 且 skip：入 meta + skip 执行；非 `public`：不入 meta）。

2. **非函数声明 doctest（边界声明）**：首版仅 `function` 可执行（见 §3 白名单）。`function` 以外全部 item kind（§27，含 `extern` 块——独立 item kind、非 function；其内 `extern_fn` 为 FFI 声明无 body）上的 fenced block 为纯文档（展示、不执行、不入 meta）。按类扩展执行模型留 v0.4+（见 §11）。

3. **async 函数 doctest**：**一律 skip（覆盖 §7.1 执行判据）**。首版可写；`public` 者入 `.ailmeta`；但 `ail test` 标记 **`[skipped: async harness TBD]`** **不执行**。隐式 async harness（块体包成 async 块 + `spawn` 取值）、块内 `await` 合法性、与 §18.5「禁跨 await 借用」的衔接，留 v0.4。（**例外**：`should_panic` 块免疫本节静默 skip，见 §7.4 方向 a；但首版 async harness TBD，故 async 函数上的 `should_panic` 块因缺 harness **判失败**而非静默 skip。）

4. **错误路径示例（`should_panic`）**：可用 ` ```ail should_panic ` 标记块演示预期 panic（constraint 违约 §15.4 / §92#9、`unwrap` 失败、显式 `panic(msg)`）。块体须触发 panic 才通过；普通块不得依赖 panic 作通过条件。
   - **`should_panic` 免疫静默 skip（方向 a 决断）**：`should_panic` 块**不受** §7.1（effect-ful）/§7.3（async）的静默 skip 约束——**总尝试执行**以验证 panic 是否触发。`should_panic` 是强承诺（声明「此示例预期 panic」），不能挂在 effect-ful/async 函数上被悄悄放行、却让 `.ailmeta` 的 `expect_panic: true` 声称一个从未验证的期望（§5）。
   - **环境不满足 → 明确失败（非静默 skip）**：若执行环境不满足（effect-ful 函数缺 fixture / async 函数缺 harness，首版 async harness TBD 见 §7.3），`ail test` 报**明确错误**（**非零退码**，§8），而非静默放行。可自洽执行的 panic（**针对 §7.1 非 async 的 effect-ful 函数**：含 `unsafe` 块的函数上的 constraint 违约 / `unwrap` 失败 / 显式 `panic(msg)` / 溢出 / 越界——不依赖外部 IO 即可触发）按「块体 panic → 通过；正常返回 → 失败」正常验证；**async 函数的 should_panic 一律见 §7.3（缺 harness 判失败，不在此列）**。

5. **泛型函数**：doctest 须指定具体类型实参（`parse<int>("42")`，非裸 `parse`）。

---

## 8. 诊断与容错

doctest 不是「绕过编译器的脚本」——它过**与正式代码相同的静态管线**：

- **静态检查**：doctest body 经完整 **§70/§71（lex+parse）→ §73（typecheck）→ §74（borrow）→ §19（effect）** 管线。语法/类型/所有权/effect 错误均照常报。
- **诊断 span**：精确到 fenced block **内**的字符位置，并附 `///` 源码定位（指向 doc 源），便于 AI 闭环修复。
- **三类退出语义**：

  | 情形 | `ail test` 行为 | 退码 |
  |---|---|---|
  | doctest **编译错**（语法/类型/借用） | 报编译错、不执行该块 | 非零 |
  | **运行时失败**（assert 失败 / `unwrap` / 溢出 / 越界等任意 panic） / `should_panic` 块未 panic / `should_panic` 块缺 fixture 或 async harness | panic 或报缺失错误、报 doc 源 span | 非零 |
  | **skip**（effect-ful / async，**不含 `should_panic` 块**——见 §7.4 方向 a） | 标记 skip 标签、不计失败 | 零 |

- **`ail doc` vs `ail test`**：`ail doc` 默认只解析展示 doctest、不执行、不报编译错（除非配 `--test`）；`ail test` 执行上表完整语义。

---

## 9. 与现有机制的衔接（自洽性）

| 现有机制 | 衔接方式 |
|---|---|
| **`///` trivia（§70/§71）** | fenced block 是 `DocComment` 的子结构，**零新附着规则**——解析时回看连续 `///`，识别其中的 fenced block |
| **`meaning`（§15.3）** | `meaning` = 语义描述（自然语言，"是什么"）；doctest = 行为示例（可执行，"怎么用"）。**互补不冲突**，两者都进 `.ailmeta` 不同槽位 |
| **§41 `test` / `assert`** | doctest 复用 `assert` **函数调用**语义（`assert(cond)`，§17/§27）、`ail test` 统一入口——**无第二套断言体系** |
| **契约 `requires`/`ensures`（§20）** | 契约 = for-all 规范；doctest = exists 示例。doctest 的 assert **不得与契约矛盾**（验证器职责，Tier 3 side-car） |
| **`.ailmeta` schema_version** | 加 `examples[]` = schema 自身 semver bump（v0.2 → v0.3），与语言版本解耦（§24） |

---

## 10. 备选方案（及为何不选）

- **A. 表达式求值风格**（Python doctest `>>> expr` → 比对输出）：简洁，但对 `void`/`Result`/无返回值函数不直观，输出比对脆弱（依赖格式）。**不选**——`assert` 更明确、AI 更易生成与解析。
- **B. 独立 `example` 声明**（`example "..." for fn add { assert ... }`）：显式，但增加语法表面 + 新关键字风险（违 F6）+ 文档与代码分离倾向。**不选**——`///` 内嵌更轻、文档紧贴代码。
- **C. 独立 doctest 文件**（`*.doctest.ail`）：彻底分离，但**违反「文档不漂移」核心目标**（分离 = 漂移）。**不选**。

---

## 11. 未决问题（待 review）

1. doctest 块内能否 `import` 辅助模块？（倾向：能，但仅 std + 同包）
2. 非函数声明执行模型（v0.4）：struct/enum/type/trait 各自的「构造 + 断言」模式；agent/tool 是否仅入 meta；actor/server/task 是否走 fixture。
3. async harness 设计（v0.4）：隐式 async 块 + `spawn` + §18.5 衔接。
4. `should_panic` 之外还需哪些标记？（如 `compile_fail`——演示「这段不该编译」）
5. 与未来 PBT（Tier 1b）集成：doctest 示例作 PBT generator 种子？（留 PBT RFC）
6. `ail doc` 生成的 HTML 里 doctest 渲染样式（折叠？高亮？）—— 工具链细节，非语言层。
7. item 级 `public` 闸门下，方法/字段可见性组合的细节语义。

---

## 12. 不触动 v0.2.1（合规声明）

- **不改 §1–§94 任何条款**。
- **不增关键字**（56 不变；doctest 零新关键字，纯 `///` + fenced block + info-string 标记词）。
- **不改 110 项决议**。
- doctest 是 **v0.3+ 特性**；本 RFC 为**提案**，经 review + 批准后进 v0.3 规范（届时 AILANG.md 新增章节，schema_version bump）。
- 本 RFC 本身不修改 `docs/AILANG.md`。

**⚠️ 关于 §41 的预存笔误（不在本 RFC 修）**：审计发现冻结规范 **§41（测试示例）** 用了**宏式** assert（如 `assert result.is_ok()`——assert 后直接跟表达式，非调用形 `assert(result.is_ok())`），与 §17（assert 是函数）/ §27（无 assert 语句产生式）不一致。这是 **v0.2.1 冻结规范自身的预存缺陷**，**不在 RFC 0001 范围内修复**（冻结规范需另案决议）。RFC 0001 全部示例一律用调用形，以保合规。
