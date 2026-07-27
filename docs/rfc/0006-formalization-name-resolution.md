# RFC 0006 · P0-1 形式化层 + 名字解析—— 控制流体产生式 / 字面量词法 / 类型推断 / `<` 消歧 / 名字解析算法

| | |
|---|---|
| **状态** | 草案（Draft v1）—— 待 review（尚未跑对抗式 workflow；目标收敛 0H/0M/0L，对齐 RFC 0001 v6 / 0002 v8 / 0003 v5 / 0004 v5 / 0005 v1）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94 语义决断、56 关键字、110 决议均不变；**唯一例外**见 §7 —— `<` 消歧若采纳方案 B 将修改 §86 #7，作为本 RFC 显式决断演进，需 review 重点审查）|
| **日期** | 2026-07-27 |
| **分级** | **P0-1**（综合判断 [`docs/research/synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 第二优先级；编译器前三环 lexer/parser/typechecker 的前置依赖）|
| **承接** | [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 P0-1（5 子项）+ §5（形式化层内核：§27 控制流体产生式缺、typing rules 缺、`<`消歧缺、名字解析是最大盲区）；[`spec-maturity-2026-07.md`](../research/spec-maturity-2026-07.md)（最薄弱=常量求值域 ~5%、名字解析 2.0、§27 文法推不出自身示例）；[RFC 0005](./0005-spec-governance.md)（P0-0 治理地基，本 RFC 的规范性约束力载体）|

---

## 1. 动机

spec-maturity 第二轮把 AILang 的形式化层评为**最薄弱单点之一**：§27 形式文法「推不出自身教程示例」（控制流语句只有裸终结符）、§8.6 字面量词法近乎空白、§15.2 类型推断无算法、名字解析是「编译器第一个语义 pass 却是最大盲区」（§73 仅一行）。这些不是设计缺陷，而是**形式化层未展开**——骨架已在（§27 有 `stmt`/`expr` 骨架、§8.6 有节标题、§15.2 有两句原则），但产生式体、词法规则、算法、decidability 声明全部缺席。

**为什么必须排 P0-1（仅次于 P0-0 治理地基）**。编译器四个核心 pass 全卡在这一层：

- **Lexer** 卡在 §8.6（无字面量词法，无法 token 化 string/int/float/char）；
- **Parser** 卡在 §27（无 if/for/while/loop/parallel 完整产生式，无法 parse 控制流；`<` 无消歧，无法 parse 泛型调用）；
- **Name Resolver**（编译器第一个语义 pass）卡在 §12（无名字解析算法，无法解析 `a.b.c` / `T(x)` 类别）；
- **Type Checker** 卡在 §15.2（无推断算法 + decidability 声明，无法证明类型检查终止）。

不补这一层，实现者会自行发明这些产生式 / 规则 / 算法并固化进编译器，未来补规范即破坏性变更——与 synthesis §3「A 类未定义地基」的紧迫性判断一致。

**本 RFC 的性质**：4/5 子项是**展开既有骨架**（补产生式体 / 词法规则 / 算法），**零新关键字、零新产生式类别**；唯一需要**决断**的是 §7 `<` 消歧（方案 A 保决断加规则 vs 方案 B 改语法 `::<T>`），作为本 RFC 的核心开放决断点。

---

## 2. 设计目标

1. **不触动冻结语义决断**——§1–§94 的语义、56 关键字、§83–§94 的 110 决议不变。§7 方案 B 为唯一例外（显式标注）。
2. **展开而非新增**——§4/§5/§6/§8 是补全既有骨架（stmt 体、§8.6 内容、§15.2 算法、§12 算法），不引入新关键字、新产生式类别、新类型系统特性。
3. **与既有自洽**——补全的产生式 / 规则**必须**与 §16（控制流语义）、§87 #9（parallel for 表达式）、§86 #7（`<T>` 类型实参）、§89 #2/§92 #2（名字消歧）、§15.10（泛型）完全对齐（§9 自洽核查表）。
4. **decidability 显式声明**——§6 类型推断**必须**声明可判定性 + 复杂度量级，使「类型检查终止」成为可证明的规范承诺，而非工程假设。
5. **AI 友好**——补全后的文法 / 词法 / 算法**应**可被 AI 直接消费（无歧义、可预测、`.ailmeta` 可结构化），对齐 AILang 核心卖点。

---

## 3. 现状与缺口诊断

| 子项 | 现状（v0.2.1） | 缺口 | 阻塞的 pass |
|---|---|---|---|
| §27 控制流 | `stmt` 列了 `if\|for\|while\|loop` 为**裸终结符**、`parallel (...)` 为 `...` 省略 | 完整产生式体（条件 / 迭代子句 / else / body） | Parser |
| §8.6 字面量 | 仅「标识符 ASCII 规则 + 整数无后缀」两行 | string/int/float/char 词法（进制 / 转义 / 分隔符 / 优先级） | Lexer |
| §15.2 推断 | 仅「局部可推断 + 签名显式 + return-only 泛型显式」两句 | 推断算法族 + 流程 + decidability 声明 | Type Checker |
| §27 `<` 消歧 | `call := "<" type_arg* ">" "(" args? ")"` 与 `eq` 的 `"<"` 比较共存，**无消歧规则** | `f < g` 是比较还是泛型调用的判定规则 | Parser |
| §12 名字解析 | 有 import 规则（全路径 / 禁通配 / 冲突→as / 本地优先），但**无解析算法** | 作用域层级 + 点分路径 + `.member` 查找序 + 类型/值命名空间 | Name Resolver |

---

## 4. §27 控制流体产生式补全

落地形态：展开 §27 的 `stmt` 产生式，将裸终结符替换为完整产生式引用。

**补全的产生式**（EBNF，沿用 §27 记法）：

```
if_stmt        := "if" expr block ("else" (if_stmt | block))?        // 条件 expr 须 bool（§15）；else-if 链
for_stmt       := "for" pattern "in" expr block                       // pattern 复用 §27 pattern（含 _）；expr 须可迭代（List/Array/Set/Map/Iterator）
while_stmt     := "while" expr block                                  // 条件 expr 须 bool
loop_stmt      := "loop" block                                        // 无条件循环（语句）；体类型 never（§15.1）；break/return 为控制流出口
break_stmt     := "break"                                             // 跳出 loop/for（无值；loop 非 value 表达式，对齐 §15.1）
continue_stmt  := "continue"                                          // 跳至下一次迭代
parallel_for   := "parallel" "for" pattern "in" expr block            // **primary 级表达式**（并入 §27 primary），返回 List<R>（§87 #9）；body 须返回 R、pure 或仅 io.read
parallel_reduce:= "parallel" "reduce" "(" expr "," expr ")" arm_body  // **primary 级表达式**；body 用 arm_body（§27，兼容 { x -> f(x) } 与纯 block）；op 形式见设计说明
```

**`stmt` 产生式更新**（替换原裸终结符行）：

```
stmt := "let" ident (":" type)? "=" expr
     |  "var" ident (":" type)? "=" expr
     |  "return" expr?
     |  "throw" expr
     |  if_stmt | for_stmt | while_stmt | loop_stmt
     |  break_stmt | continue_stmt
     |  match | select
     |  "spawn" ("detached")? "actor"? ( postfix | block ) | "cancel" expr
     |  try_catch
     |  "unsafe" block | expr
```

**设计说明**：

- `if_stmt` 的 `expr` 条件**必须**为 `bool`（§15），无 truthy 隐式转换（对齐 AILang 无隐式转换原则，唯一例外是 `string + Display` §11）。
- `for` 的迭代源 `expr` 须为可迭代类型（`List`/`Array`/`Set`/`Map`/`Iterator` trait）；`Map` 迭代序见 §6/RFC 后续 P0-2（当前未指定，RFC 0005 §13 已登记）。
- `loop_stmt` 是**语句**（非 value 表达式），体类型为 `never`（§15.1：`loop` 表达式类型为 `never`/⊥）；`break` / `return` 是控制流出口（**不携带值**），`loop` 不产出值。这与冻结 §15.1 完全一致、**非语义演进**——`loop` 未加入 §27 `expr` 产生式，故 `let x = loop { … }` 不可达（loop 仅语句位置合法）。若未来要让 `loop` 产出值（`break expr` + loop 类型 = break 值类型），须作为对 §15.1 的显式演进另立 RFC（走 RFC 0005 §9 解冻通道）并同时把 `loop` 加入 `expr` 产生式——本 RFC 不引入此项（见 §11 #1）。
- `parallel_for` / `parallel_reduce` 按 §87 #9 为**表达式**（返回 `List<R>` / 归约值），本 RFC 将其**并入 §27 `primary`**（与 `struct_init` / `list_lit` / `map_lit` 同级），故 `let squares = parallel for x in data { x * x }`（§21.10 / §58 冻结示例）可从 `expr` 链派生；语句位置经 §27 `stmt := … | expr` 分支自动可达（故 `stmt` 产生式不再单列 parallel）。`parallel_reduce` 的 `op` 实参为 lambda 字面量（如 `(a, b) -> a + b`），其形式依赖一等闭包——§21 冻结声明 v0.2 无一等闭包，故 §21.10 / §58 的 `parallel reduce(…)` 示例为**前向示例**，op 字面量的完整定义随闭包 RFC 落地（见 §11 #2）；本 RFC 只锁 `parallel_reduce` 产生式骨架（body 已用 `arm_body` 兼容冻结 body 形 `{ x -> … }` 与纯 block）。
- `break`/`continue` 仅在 loop/for 体内合法，嵌套函数内非法（编译期校验）。

> `break`/`continue` 已在 §9 控制流类别（56 关键字之一，§83 #5 锁定），本 RFC 仅补其**产生式体**（原 §27 `stmt` 骨架未列其形），**不新增关键字**。其语义在 §16（统一 `{ }` 控制流）已隐含使用、§298/§381 已列。

---

## 5. §8.6 字面量词法补全

落地形态：扩充 §8.6「标识符与字面量」为完整词法规则。

**补全的词法产生式**：

```
// 整数（无类型后缀，沿用现状；类型由上下文推断 §15.2）
int_lit   := decimal | hex_lit | oct_lit | bin_lit
decimal   := [1-9] ("_"? [0-9])* | "0"                      // 十进制整数部分，允许 _ 分隔；纯 "0" 合法（禁前导零，避免 C 的 010 陷阱）
hex_lit   := "0x" [0-9a-fA-F] ("_"? [0-9a-fA-F])*           // 0xFF 0xDE_AD
oct_lit   := "0o" [0-7] ("_"? [0-7])*                       // 0o777
bin_lit   := "0b" [01] ("_"? [01])*                         // 0b1010
digits    := [0-9] ("_"? [0-9])*                            // 允许前导零与 _ 的数字序列（用于浮点小数部分/指数；与 decimal 禁前导零区分）

// 浮点（无类型后缀；默认 f64，§15.1）
float_lit := decimal "." digits exp?                        // 1.0  1.5  3.14e0  3.141_592（小数与指数均允许 _）
           | decimal exp                                     // 1e10  2E-3  1e05（指数允许前导零）
exp       := ("e" | "E") ("+" | "-")? digits                // 指数用 digits（允许前导零，如 e05；不复用 decimal）

// 字符
char_lit  := "'" ( char_escape | [^'\\ \n] ) "'"             // 'a'  '\n'  '\u{1F600}'；单引号内单字符或转义

// 字符串
string_lit:= '"' ( string_escape | [^"\\] )* '"'             // "hello\nworld"；双引号包裹
string_escape := "\\" ( "n" | "t" | "r" | "\\" | "'" | '"' | "0"   // 常规转义 \n \t \\ \' \" \0
                       | "u{" hex+ "}" )                      // Unicode 标量值 \u{1F600}（1–6 位 hex）
char_escape := string_escape                                // 字符转义 = 字符串转义（共用；char_lit 引用此别名，闭合悬空非终结符）

literal   := int_lit | float_lit | char_lit | string_lit | "true" | "false"
```

**设计说明**：

- **无类型后缀**（继承现状）：`int`/`float` 字面量无 `i32`/`u64`/`f32` 后缀，类型由上下文推断（§15.2）或默认（`int`→i64、`float`→f64，§15.1/§90 #2）。这符合 AILang 简化设计，但要求推断算法处理字面量类型确定（§6）。
- **数字分隔符 `_`**：允许 `1_000_000` 提升可读性（Rust/Java/Python 通行）；`_` 不可在首尾或 `0x` 紧后。
- **进制前缀**：`0x`/`0o`/`0b`（对齐 Rust/Go）；纯 `0` 开头**不**作八进制（避免 C 的 `010` 陷阱），八进制必须 `0o`。
- **转义**：字符串/字符共用 `string_escape`；`\u{H+}` 为 Unicode 标量值（U+0000–U+10FFFF， surrogate 禁止——编译期校验）。原始字符串（`r"..."`）**推迟 v0.4**（开放问题 #2）。
- **词法优先级**：`float_lit` 优先于 `int_lit`（含 `.` 或 `e` 者为 float）；`string_lit`/`char_lit` 由起始引号区分；`true`/`false` 为 bool 字面量（关键字，§9）。
- **UTF-8 源码**：`.ail` 为 UTF-8（§8.1）；非字面量上下文的非 ASCII 字符仅允许在注释与字符串/字符内容内（标识符 ASCII，§8.6 现状）。

> 此补全使 §27 使用的 `string_lit` / `literal` 终结符在词法层有了定义，闭合「文法引用词法未定义」的缺口。

---

## 6. §15.2 类型推断算法

落地形态：扩充 §15.2「类型推断」为算法描述 + decidability 声明。

**算法选型：双向类型检查（bidirectional typechecking）+ 局部约束求解**。

AILang 的既定设计（§15.2 现状）——「函数参数与返回**必须显式**、局部可推断、泛型由实参推断」——天然契合 **bidirectional** 模型（Pierce & Turner, 「Local Type Inference」），而非完整 Hindley–Milner：

- **`synth`（推断模式）**：从表达式推断类型。用于 `let x = expr`（局部变量）、函数体表达式、泛型实参推断。
- **`check`（检查模式）**：给定期望类型，检查表达式是否符合。用于函数参数（签名提供 annotation）、`return` 表达式（对照返回类型）、`let x: T = expr`（带标注）。

**核心规则**：

1. **签名边界全显式**——函数参数类型、返回类型、`where { }` 泛型 bound（§15.10）在签名处**必须**显式；这是 `check` 模式的锚点，使推断**局部**于函数体，不跨函数传播（跨函数契约传播属 RFC 0004 Tier 3，非本层）。
2. **局部 `let` 推断**——`let x = e`：`synth(e)` 得 `T`，`x: T`；`let x: T = e`：`check(e, T)`。局部变量**不泛化**（无 let-polymorphism）——简化推断、对齐「签名显式」边界。
3. **泛型实参推断**——`f(args)` 调用：由 `args` 的 `synth` 类型 + `f` 签名，生成约束 `T_i = synth(arg_i)`，求解泛型参数 `T_i`。**return-only 泛型**（`fn empty<T>() -> List<T>`，§86 #7）无法从实参推断，**必须**调用点显式 `empty<int>()`（turbofish 或 `<T>`，见 §7）。
4. **字面量类型确定**——无后缀字面量（§5）的 `synth`：`int_lit` → 默认 `int`（i64），但若处 `check(T)` 上下文且 `T` 为整数类型（`uint`/`int8`..），则采用 `T`（双向消歧）；`float_lit` → 默认 `float`（f64），同理可被 `check(f32)` 收窄；`char_lit` → `char`（Unicode 标量值类型，由 [RFC 0007](./0007-determinism-batch.md) §8 引入 §15.1——本 RFC 定义 `char_lit` 的**词法**、其**类型归属** `char` 跨 RFC 由 0007 §8 闭合，两 RFC 同批落地故自洽）；`string_lit` → `string`；`true`/`false` → `bool`。
5. **约束传播**——`constraint T { ... }`（§15.4）的构造时断言在 `check` 完成后运行期求值（非推断的一部分，§92 #8）。

**Decidability 声明（规范承诺）**：

> **类型推断可判定**：给定全显式的函数签名，函数体内的**类型推断（synth/check 等价合一求解）**在 **O(n·α(n))** 时间内终止（n = 函数体 AST 节点数，α ≈ 近线性，来自 union-find 约束求解），**且总是产生唯一类型或报告类型错误**。**范围限定**：本承诺仅覆盖等价合一求解（局部约束图的并查集求解）；**不含** trait bound 解析（`where { Ord<T> }` 在全局 impl 表中查找，§15.10 / §86 #8 / §86 #10）——后者单独可判定（必终止、唯一结果或类型错误），但复杂度随全局 impl 表大小与单态化实例如数增长，不归 union-find、不在 O(n·α(n)) 内。可判定的依据：① 签名显式消除跨函数传播；② 局部 `let` 不泛化（无 HM 的 let-poly、无 principle type 通集问题）；③ 类型项有限 + 每次调用取 fresh 类型变量 + 名义类型按 head 相等（§15.3），Robinson 合一在有限项上必终止（`Box<T>` §15.6 仅保内存布局有限，**非**合一终止条件）；④ 等价约束经 union-find 求解，有限约束集必终止（不要求 DAG）。**前置良型条件**：decidability 承诺仅覆盖**类型良型**的程序——透明别名环（`type A = B; type B = A`）、递归类型未 `Box<T>` 间接等须由前置良型 pass 拒绝（对齐 §73 实现层陷阱），不在本终止保证范围内。

> 此声明把「类型检查终止」从工程假设提升为**规范承诺**，对齐 SPARK/Rust 的 decidability 取向（synthesis §5 第二轮「borrow checker 无 soundness 声明」的同层问题，本 RFC 先解决类型推断层）。

---

## 7. §27 `<` 消歧规则（核心开放决断）

**冲突**：§27 同时定义
- `call := "<" type_arg ("," type_arg)* ">" "(" args? ")"`（类型实参，如 `decode<User>(row)`，§86 #7）
- `eq := add ( ("<"|">"|"<="|">="|"=="|"!=") add )*`（比较运算符）

导致 `f < g` 歧义：是比较 `f` 与 `g`，还是泛型调用 `f<g>(...)`？当前**无消歧规则**。

**两方案**：

### 方案 A：lookahead 消歧（保留 `<T>` 语法，不动 §86 #7）

`call` 的 `<` 后缀仅当后续 token 序列匹配 `type_arg ("," type_arg)* ">" "(" ` 时才采纳为类型实参；否则 `<` 回退为比较运算符。

```
call := type_args "(" args? ")"        // 纯后缀；嵌入 §27 postfix := primary (call|index|member)*
      | "(" args? ")"                  // 仅当 lookahead 匹配 type_args ">" "(" 时采纳上一行（类型实参），否则 < 回退为比较
type_args := "<" type_arg ("," type_arg)* ">"    // lookahead：须后跟 "("
```

- **优点**：不修改 §86 #7，`decode<User>(row)` 语法不变；零迁移成本。
- **缺点**：保留理论歧义边界——`f < g > (h)` 这类「比较链后跟括号」会被误解析为泛型调用（Rust pre-2018 的已知陷阱）；需 parser 维护一个试探回溯或优先级规则。对 AI 不够「可预测」。

### 方案 B：turbofish `::<T>`（彻底消歧，修改 §86 #7）

类型实参**必须**写作 `::<...>`（Rust现行 turbofish），`<` **永远**是比较运算符。

```
call := "::" type_args "(" args? ")" | "(" args? ")"     // 纯后缀；嵌入 §27 postfix := primary (call|index|member)*
```
迁移：`decode<User>(row)` → `decode::<User>(row)`（§86 #7 示例更新）。

- **优点**：彻底无歧义——`<` 永远是比较，`::<` 永远是类型实参；parser 无回溯；对 AI 高度可预测（对齐 AILang 核心）；与 Rust 生态经验证一致。
- **缺点**：**修改 §86 #7 冻结决议**（调用点 `<T>` → `::<T>`），属决断演进；现有**调用点** `<T>` 示例需迁移——分布于 §15.2（`empty<int>()`）、§18（`Mutex<Data>(...)`）、§21（`Channel<int>()`）、§63 / §84 #14（`req.param<UserId>("id")`）、§27 文法注释（`decode<User>(row)`）、§86 #7 等（非穷尽）。**注意**：§15.10 的 `fn sort<T>` 属**泛型声明**（`generic` 产生式，非调用点 `call`），**不**需迁移（声明位在任何语言都不用 turbofish）。本 RFC 为 Draft（未实现），迁移成本仅在文档层。

### 推荐

**方案 B（turbofish）**——一劳永逸消除歧义、最大化 AI 可预测性（AILang 核心卖点）、与 Rust 生态一致。代价是修改 §86 #7，但作为本 RFC 的**显式决断演进**（§1 表格已标注唯一例外），走 Accepted RFC 授权通道（RFC 0005 §9 规范解冻通道）合规。

若 review 判定迁移成本不可接受，回退方案 A（保决断、加 lookahead 规则），但需在 §9 自洽核查中登记其歧义边界为已知限制。

**最终采纳（A/B）留作本 RFC 的核心决断**，需 review + 对抗验证后定（见 §11 #3）。

---

## 8. §12 名字解析算法

落地形态：§12 新增 **§12.4 名字解析算法**。

**作用域层级**（查找序：内 → 外，首个匹配胜出）：

1. **块作用域**（`{ }` 内 `let`/`var`/`pattern` 绑定，§27 block）；
2. **函数作用域**（参数 + 函数内局部）；
3. **模块作用域**（当前 `package` 的顶级 item + 同包其他文件的 item——Go 式共享命名空间，§12.1）；
4. **导入作用域**（`import`/`from import` 的 public item + `as` 别名，§12.2）；
5. **`std.core`**（自动加载，§84 #9）。

**命名空间**：**类型命名空间**（`type`/`struct`/`enum`/`interface`/`trait`/`error`/`actor`/`agent`）与**值命名空间**（`fn`/`let`/`const`/`field`）**分离**——`type X` 与 `fn X` 不冲突（同名合法，按上下文分派）。`module`/`package` 名在两者均可引用（路径前缀）。

**简单名 `ident` 解析**：

- 按作用域层级 1→5 顺序查找；**首个匹配**胜出（shadowing：内层遮蔽外层）；
- **本地定义优先于导入**（§12.2：遮蔽 + warn）；
- 导入冲突（两导入同名）= 编译错误（须 `as`，§12.2）；
- 查无 = 编译错误 `UnresolvedName`。

**点分路径 `a.b.c` 解析**：

1. `a` 按简单名解析（值或类型或模块）；
2. `.b` 按 `a` 的类别查找 member：
   - `a` 为**值**：`.b` = 字段访问（struct field）或方法（trait method，无括号时为方法引用）；
   - `a` 为**类型**：`.b` = enum variant（`Color.Red`）或关联 item（v0.2 无关联函数，推迟）；
   - `a` 为**模块**：`.b` = 模块的子 item（`std.http.Client`）；
3. `.c` 递归同上；
4. 查无 = `UnresolvedMember`。

**`call` 形 `Name(args)` 类别消歧**（填补 §89 #2/§92 #2 引用的「名字解析消歧」）：

`Name` 解析结果决定 `Name(args)` 的语义：
- **函数** → 函数调用；
- **类型名** → 若 `args` 为 `ident: expr` 形 → `struct_init`（§27 struct_init）；若 `args` 为单 `expr` → 约束类型构造 `T(expr)`（§15.4/§92 #2，零成本转型）；
- **enum variant**（含路径 `Color.Red(args)`）→ variant 构造；
- **类型名 + 泛型** → 泛型实例化（`List<int>`，type 产生式 §27）；
- 歧义（同名既函数又类型）→ 按命名空间分离无冲突；同命名空间歧义 = 编译错误。

**shadowing 细节**：内层 `let` 遮蔽外层 `let`/参数合法（warn 可配）；遮蔽导入合法 + warn（§12.2）；**类型名不可被值遮蔽**（命名空间分离）。

> 此算法闭合 §27 多处「由名字解析消歧」的前向引用，使 Name Resolver pass 有了可实现的规范定义。

---

## 9. 不变量与自洽核查

| 本 RFC 条款 | 对齐的既有规范 | 自洽性 |
|---|---|---|
| §4 if 条件须 bool | §15 无隐式转换（仅 string+Display 例外 §11） | ✅ |
| §4 loop 体 never（语句，非 value 表达式，break 无值） | §15.1 `loop` 表达式类型 = `never`（⊥） | ✅ 一致（loop 未入 expr 产生式，非演进） |
| §4 parallel_for/reduce 并入 §27 primary（expr 可达） | §87 #9 parallel 表达式化 + §21.10/§58 示例 | ✅ 已并入 primary，`let x = parallel for…` 可派生 |
| §5 无后缀字面量 + 默认类型 | §8.6 现状 + §15.1 int=i64/float=f64 + §90 #2 | ✅ |
| §5 char_lit → char 类型 | RFC 0007 §8 引入 char 类型（跨 RFC，同批落地） | ✅ 词法在本 RFC、类型在 0007 §8 |
| §6 签名显式 + 局部推断 | §15.2 现状两句 + §15.10 泛型 where | ✅（展开） |
| §6 return-only 泛型显式 | §86 #7 | ✅ |
| §7 方案 B 修改 `<T>` | §86 #7（**显式演进**，唯一例外） | ⚠️ 需决断 |
| §8 类型/值命名空间分离 | §89 #2/§92 #2 名字消歧引用 | ✅ 填补 |
| §8 struct_init vs T(expr) | §27 struct_init + §15.4 constraint 构造 | ✅ |
| 零新关键字 | §9（56 表）；`break`/`continue` 已在控制流类别（§83 #5 锁定） | ✅ 已核查含 |

---

## 10. 落地映射（合并进 AILANG.md 的位置）

| 本 RFC 内容 | 落地位置 | 性质 |
|---|---|---|
| §4 控制流体产生式 | §27 `stmt` 产生式展开（替换裸终结符） | 规范性（Part I 文法） |
| §5 字面量词法 | §8.6 扩充 | 规范性（Part I 词法） |
| §6 推断算法 + decidability | §15.2 扩充 | 规范性（Part I 类型系统） |
| §7 `<` 消歧（方案 A/B） | §27 `call` 产生式 + §86 #7 示例（若 B） | 规范性（若 B 则含决断演进） |
| §8 名字解析算法 | §12 新增 §12.4 | 规范性（Part I 模块） |

---

## 11. 开放问题

1. **`loop` 作 value 表达式（推迟）**——本 RFC 保持 `loop` 为语句、体类型 `never`、`break` 无值（对齐冻结 §15.1，零语义演进，loop 未入 §27 `expr` 产生式）。若未来要让 `loop` 产出值（`break expr` + loop 表达式类型 = break 值类型，多 break 点同型合流），须作为对 §15.1 的显式演进另立 RFC（走 RFC 0005 §9 解冻通道）并同时把 `loop` 加入 `expr` 产生式——本 RFC 不引入此项。
2. **`parallel_reduce` 的 `op` lambda 字面量**——§4 `parallel_reduce` 的 `op` 实参为 lambda（如 `(a, b) -> a + b`），其形式依赖一等闭包；§21 冻结声明 v0.2 无一等闭包，故 §21.10 / §58 的 `parallel reduce(…)` 示例为前向示例。本 RFC 只锁产生式骨架（body 用 `arm_body`）；op 字面量的完整定义随闭包 RFC 落地。
3. **原始字符串 / 字节串**——§5 仅定义基础 `string_lit`；`r"..."` 原始字符串、`b"..."` 字节串推迟 v0.4（与 raw_pointer/字节处理一并）。
4. **`<` 消歧方案 A/B 最终决断**——§7 推荐 B，但需 review + 对抗验证评估迁移成本与歧义边界后定。
5. **推断算法的 const fn / CTFE 边界**——§6 仅覆盖运行期类型推断；常量求值域（const fn / CTFE）是 spec-maturity 最薄弱单点（~5%），但其形式化超出 P0-1 范围，留独立 RFC（P1+）。
6. **类型推断与 borrow checker 的交互**——§6 的推断独立于所有权（§18/§74）；二者在 codegen 前的 pass 顺序（infer → borrow）待编译器实现层定（§65+）。

---

## 12. 收敛轨迹

**Draft v1（本文）**——尚未跑对抗式 workflow。计划按 RFC 0001–0005 既有流程：多维度 workflow 审查（建议维度：EBNF 产生式自洽 / 词法优先级无冲突 / 推断算法 decidability 证明严谨性 / `<` 消歧两方案 tradeoff 完备性 / 名字解析与 §89 §92 §12.2 对齐 / 与 §16 §87 §86 既有语义无矛盾 / 关键字地位 / AI 可消费性）+ 对抗式 verify，目标收敛 **0H / 0M / 0L**。

> 本 RFC 是形式化层（lexer/parser/typechecker/name-resolver 四 pass 前置），验证重心在**文法/算法的自洽与完备**，比 0001–0004 的实现可行性更偏理论，但比 0005（纯元规则）更需算法正确性论证。

---

*本 RFC 由综合判断 [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 P0-1 驱动。落地后将闭合「§27 推不出自身示例」「名字解析是最大盲区」「类型推断无 decidability」三个形式化层关键缺口，使 AILang 规范从「骨架已具、内核未形」推进到「编译器四 pass 可依据形式化定义开工」。*
