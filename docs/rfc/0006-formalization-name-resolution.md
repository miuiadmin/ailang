# RFC 0006 · P0-1 形式化层 + 名字解析—— 控制流体产生式 / 字面量词法 / 类型推断 / `<` 消歧 / 名字解析算法

| | |
|---|---|
| **状态** | 草案（Draft v1）—— 待 review（已跑多轮对抗式 workflow pass-2–pass-14，post-pass-14 全部 findings 已修正、待 pass-15 复验；目标收敛 0H/0M/0L，对齐 RFC 0001 v6 / 0002 v8 / 0003 v5 / 0004 v5 / 0005 v1）|
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

**`primary` 产生式更新**（并入 parallel 表达式，§87 #9）：

```
primary := literal | ident | "(" expr ")" | struct_init | list_lit | map_lit | parallel_for | parallel_reduce
```

`parallel_for` / `parallel_reduce` 以关键字 `parallel` 起头，与 `ident` / `literal` 分支 first-set 不交，无 LL(1) 冲突；此更新使 §21.10 / §58 的 `let squares = parallel for x in data { x * x }` 可从 `expr` 链派生（语句位置经 `stmt := … | expr` 分支自动可达）。**产生式图闭合**：`parallel_for` / `parallel_reduce` 经 `primary → postfix → expr → stmt` 可从 start symbol 推导，不再悬空。

**设计说明**：

- `if_stmt` 的 `expr` 条件**必须**为 `bool`（§15），无 truthy 隐式转换（对齐 AILang 无隐式转换原则，唯一例外是 `string + Display` §11）。
- **头表达式 no-struct-literal 上下文**：`if` / `while` / `for` / `parallel for` 头部的条件 / 迭代源 `expr`（及 `match` scrutinee）为 **no-struct-literal 上下文**——当 `primary` 在这些位置解析到 `ident` 后直接跟 `{` 时，`{` **一律视为控制流 block 的起点**，**不**走 `struct_init` 分支（消除 `if flag { … }` 的 LL(1) 歧义：`flag` 归约为 bare ident 条件、`{ … }` 为 then-block）。需在头表达式位置构造 struct 须括号包裹 `(Foo { x: 1 })`（对齐 Rust no-struct-literal 模式）。此规则闭合 §27 `primary` 含 `struct_init` 与控制流体共用 `expr` 的句法歧义。
- `for` 的迭代源 `expr` 须为可迭代类型（`List`/`Array`/`Set`/`Map`/`Iterator` trait）；`Map` 迭代序见 [RFC 0007](./0007-determinism-batch.md) §6.3（显式登记为 unspecified，诚实不假装有序；需确定序须显式排序——物化为 `List` 后排序（std 排序 API 随 `std.collections` 落地；冻结规范中 `fn sort<T>(List<T>)` 仅作泛型语法示例出现于 §15.10，**非**已声明的 std 函数））。
- `loop_stmt` 是**语句**（非 value 表达式），体类型为 `never`（§15.1：`loop` 表达式类型为 `never`/⊥）；`break` / `return` 是控制流出口（**不携带值**），`loop` 不产出值。这与冻结 §15.1 完全一致、**非语义演进**——`loop` 未加入 §27 `expr` 产生式，故 `let x = loop { … }` 不可达（loop 仅语句位置合法）。若未来要让 `loop` 产出值（`break expr` + loop 类型 = break 值类型），须作为对 §15.1 的显式演进另立 RFC（走 RFC 0005 §9 解冻通道）并同时把 `loop` 加入 `expr` 产生式——本 RFC 不引入此项（见 §11 #1）。
- `parallel_for` / `parallel_reduce` 按 §87 #9 为**表达式**（返回 `List<R>` / 归约值），本 RFC 将其**并入 §27 `primary`**（与 `struct_init` / `list_lit` / `map_lit` 同级），故 `let squares = parallel for x in data { x * x }`（§21.10 / §58 冻结示例）可从 `expr` 链派生；语句位置经 §27 `stmt := … | expr` 分支自动可达（故 `stmt` 产生式不再单列 parallel）。`parallel_reduce` 的 `op` 实参为 lambda 字面量（如 `(a, b) -> a + b`），其形式依赖一等闭包——§21 冻结声明 v0.2 无一等闭包，故 §21.10 / §58 的 `parallel reduce(…)` 示例为**前向示例**，op 字面量的完整定义随闭包 RFC 落地（见 §11 #2）；本 RFC 只锁 `parallel_reduce` 产生式骨架（body 已用 `arm_body` 兼容冻结 body 形 `{ x -> … }` 与纯 block）。
- `break`/`continue` 仅在 `loop` / `for` / `while` 体内合法，嵌套函数内非法（编译期校验）。
- **`in` 为 `for` 迭代分隔符**（`for_stmt := "for" pattern "in" expr block`）：`in` 在 §9 的 56 关键字表与 §9 节内「上下文关键字」段落（行 393）中**均未登记**——本 RFC 确认其为**上下文关键字**（contextual keyword，仅 `for` 头部合法、其余位置作普通 `ident`），落地时登记进 §9 该上下文关键字段落（行 393，与既有上下文关键字同列），**不**计入 §9 的 56 保留关键字（零新关键字）。

> `break`/`continue` 已在 §9 控制流类别（56 关键字之一，§83 #5 锁定），本 RFC 仅补其**产生式体**（原 §27 `stmt` 骨架未列其形），**不新增关键字**。其语义在 §16（统一 `{ }` 控制流）已隐含使用、§7 / §9（行 298 / 381）已列。

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
char_lit  := "'" ( char_escape | [^'\\\r\n] ) "'"            // 'a'  ' '  '\n'  '\u{1F600}'；单引号内单字符（含空格 ' '）或转义；裸 \r / \n 禁止（须转义）

// 字符串
string_lit:= '"' ( string_escape | [^"\\\r\n] )* '"'          // "hello\nworld"；双引号包裹，原始换行（\n / \r）禁止（须转义，与 char_lit 一致）
string_escape := "\\" ( "n" | "t" | "r" | "\\" | "'" | '"' | "0"   // 常规转义 \n \t \\ \' \" \0
                       | "u{" hex+ "}" )                      // Unicode 标量值 \u{1F600}（1–6 位 hex）
char_escape := string_escape                                // 字符转义 = 字符串转义（共用；char_lit 引用此别名，闭合悬空非终结符）
hex        := [0-9a-fA-F]                                   // 单个十六进制数字：\u{hex+} 的元素与 hex_lit 的数字共用单一来源

literal   := int_lit | float_lit | char_lit | string_lit | "true" | "false"
```

**设计说明**：

- **无类型后缀**（继承现状）：`int`/`float` 字面量无 `i32`/`u64`/`f32` 后缀，类型由上下文推断（§15.2）或默认（`int`→i64、`float`→f64，§15.1/§90 #2）。这符合 AILang 简化设计，但要求推断算法处理字面量类型确定（§6）。
- **数字分隔符 `_`**：允许 `1_000_000` 提升可读性（Rust/Java/Python 通行）；`_` 不可在首尾或 `0x` 紧后。
- **进制前缀**：`0x`/`0o`/`0b`（对齐 Rust/Go）；纯 `0` 开头**不**作八进制（避免 C 的 `010` 陷阱），八进制必须 `0o`。
- **进制前缀承诺 / 前导零（词法错误模型）**：词法分析一旦识别 `0x`/`0o`/`0b` 前缀即**承诺**该进制——其后无至少一位有效数字（裸 `0x`/`0o`/`0b`）或出现越界数字（`0b2`、`0o9`、`0xG`）报告**词法错误**（AIL1xxx，错误码见 [RFC 0008](./0008-diagnostics-taskhandle.md) §5），**不**回退为 `decimal "0"` + 标识符拆分（对齐所引 Rust/Go 在前缀处承诺进制并对无效数字报词法错）。同理 `0` 紧跟 `("_"? [0-9])`（前导零十进制，如 `0123`、`0_5`、`0_123`）为**词法错误**（呼应「禁前导零」——覆盖 `_` 分隔边界：否则 `0_5` 会被 maximal munch 拆分为 `int 0` + `ident _5`，因 §8.6 标识符 `[A-Za-z_][A-Za-z0-9_]*` 允许 `_` 起首）。无此前缀承诺，maximal munch 会将 `0b2` 静默拆分为 `int 0` + `ident b2`、`0123` 拆分为 `int 0` + `int 123`，延后到 parser 报令人困惑的「相邻表达式」错。
- **尾部 `_` 承诺（对称的词法错误模型）**：与上述前导侧对称，数字字面量以 `_` **收尾**后紧跟分隔符 / 运算符 / EOF（如 `123_`、`0xFF_`、`1.0_`、`1_000_`）为**词法错误**（AIL1xxx）。承诺依据：词法在识别数字字面量时已承诺其内 `_` 为分隔符（「`_` 不可在首尾」§5 数字分隔符规则），故不得遗留裸尾 `_`；**不**回退为「字面量 + `ident _`」拆分——否则 `123_` 会被 maximal munch 拆为 `int 123` + `ident _`（`_` 单独亦匹配 §8.6 标识符 `[A-Za-z_][A-Za-z0-9_]*` 的零续位）、`0xFF_` 拆为 `int 0xFF` + `ident _`、`1.0_` 拆为 `float 1.0` + `ident _`，延后报令人困惑的「相邻表达式」错。此条款把「`_` 不可在首尾」规则的**尾部**半边落地为与「前导零」同形的词法错误模型（前导侧由前导零承诺覆盖、尾部侧由本条覆盖）。
- **转义**：字符串/字符共用 `string_escape`；`\u{H+}` 为 Unicode 标量值（U+0000–U+10FFFF， surrogate 禁止——编译期校验）。原始字符串（`r"..."`）**推迟 v0.4**（开放问题 #2）。
- **裸 `\r` 禁止 / 行终止符归一化**：`char_lit` / `string_lit` 的字符类排除集含 `\r`（`[^'\\\r\n]` / `[^"\\\r\n]`），与排除 `\n` 对称——字面量内的回车/换行**必须**经转义序列（`\r` / `\n`）书写，不可裸含原始 CR/LF（避免「源码里看不见的 `\r`」造成可观测差异，守「可预期」）。源码行终止符（LF `\n` / CRLF `\r\n` / 纯 CR）由词法层在逐行扫描时**归一化为 LF**（对齐 §8.1 UTF-8 源码 + Rust/Go 的 CRLF 归一惯例），故 CRLF 源文件中**跨行的**字面量仍词法错误（原始换行不得入字面量），而**字面量内部**显式书写的 `\r\n` 两字符转义序列经 `string_escape` 合法表达 CRLF 内容（归一化仅作用于源码行边界、不改动字面量内转义序列的语义）。
- **词法优先级（maximal munch）**：词法采用**最长匹配**——当且仅当 `float_lit` 产生式（`decimal "." digits exp?` 或 `decimal exp`）**完整匹配**时采为 float，否则回退 `int_lit`。即 `.` 后**须紧跟 digits**、`e`/`E` 后**须紧跟可选符号 + digits** 方为 float：`1.5` / `1e10` → float；`1.method` → `int 1` + `.method`（成员访问，**非** float——`.` 后 `m` 非数字）；`1ello` → `int 1` + `ident ello`（`e` 后 `l` 非数字）。`string_lit`/`char_lit` 由起始引号区分；`true`/`false` 为 bool **保留字面量**（**非** §9 的 56 关键字之一，§9 / §83 #5 锁定——作 `literal` 终结符的 bool 分支词法化，不计入关键字表）。
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
4. **字面量类型确定**——无后缀字面量（§5）的 `synth`：`int_lit` → 默认 `int`（i64），但若处 `check(T)` 上下文且 `T` 为整数类型（`uint`/`int8`..），则采用 `T`（双向消歧）；**`T` 为整型 base 的 semantic / constraint 类型时**（如 `UserId`、`type Age = int + meaning`，§15.3 / §15.4），`int_lit` 经**字面量强制**（§15.3 / §84 #1）跨 nominal 归属 `T`——此为 coercion（单向字面量→semantic，作用于**全部 check 位**：`let` 绑定 + 调用实参，非仅 `let`；变量间仍禁隐式转换，§15.3 nominal 不变 §86 #9），与上方原始整型宽度收窄为不同机制；`float_lit` → 默认 `float`（f64），可被 `check(f32)` 收窄（宽度收窄，与整型同机制）；**不**扩展字面量强制到 float-base semantic 类型——冻结 §15.3 / §84 #1 / §92 #7 的字面量强制授权**仅以整数为例**，本 RFC 不擅自对称扩展到浮点（float-base semantic 的字面量构造留后续 RFC）；`char_lit` → `char`（Unicode 标量值类型，由 [RFC 0007](./0007-determinism-batch.md) §8 引入 §15.1——本 RFC 定义 `char_lit` 的**词法**、其**类型归属** `char` 跨 RFC 由 0007 §8 闭合，两 RFC 同批落地故自洽）；`string_lit` → `string`；`true`/`false` → `bool`。
5. **约束构造断言**——`constraint T { ... }`（§15.4）的构造时断言在 `check` 完成后于**运行期**求值（非推断的一部分）：构造 `T(expr)` 触发断言、违约 panic `ConstraintViolation`（§92 #2 / §15.4，**无条件**运行期检查、不受 profile 调节）；类型层只校验 `expr` 与 `T` 的 base 类型相容，**不**证断言成立。
6. **block / 控制流类型规则**——函数体 `block := { stmt* }` 的类型：尾表达式 → 其 `synth/check` 类型；无尾表达式（以 `;` 收尾）→ `void`；尾为**函数级控制流出口**（`throw` / `return`）→ `never`（§15.1，`never | T = T` 使 `fn f() -> T { throw E }` / `fn f() -> T { return x }` 良型）。`if` / `for` / `while` / `loop` / `match` / `select` / `try_catch`（§27 控制流语句全集，line 1517）作**语句**不产生值；作为 block 尾时，**当且仅当其全部控制流路径均 diverge**（如 `if c { throw E } else { return }` 两分支均出口）方使 block 类型为 `never`；否则（存在 diverge 与非 diverge 混合，或全非 diverge）block 按「无尾表达式 → `void`」定型。**`loop` 的 diverge 判定（break 可达性）**：`loop` 体**无可达 `break`**（如 `loop {}`、`loop { work() }`）→ 恒 diverge → block 尾 `never`（`fn f() -> T { loop {} }` 良型）；`loop` 体**含可达 `break`**（如冻结 §16 惯用法 `loop { work(); if done { break } }`，`break` 跳出 loop 且**不携带值** §4）→ 可经 `break` 终止、退出后 block 无尾表达式 → block 尾 `void`（故 `fn f() -> int { loop { if done { break } } }` 为**类型错**：`void` ≠ `int`——**修正**「无条件 `loop` → never」会接受缺返回值函数的 soundness 缺口；对齐 Rust `loop { break; }` 类型为 `()` 而非 `!`、仅 `loop {}` 方为 `!`）。可达 `break` 判定为**保守语法分析**（loop 体内出现**归属本 `loop`** 的 `break`——即未被内层 `fn` / 闭包 / `for` / `while` 屏蔽：`break` 按最近外层循环归属，内层 `for`/`while` 体内的 `break` 归属该内层循环、**不计入**外层 `loop` 的可达 `break`——即视为可达），可判定、与 `if` 的两 arm 出口判定同形（for/while 不做发散判定，见下）。**`for` / `while` 的 diverge 判定（保守 = 一律 `void`）**：与 `loop` 不同，`for` / `while` 作 block 尾时**一律按 `void` 定型**——**不**尝试证明其条件恒真 / 迭代源非空致必发散（条件常量折叠、迭代源可达性属数据流分析、非保守语法分析范畴、留实现自由度但**不**影响 block 尾定型）。故 `fn f() -> T { while true {} }` / `fn f() -> int { for x in xs { return g(x) } }` 这类 `for`/`while` block 尾函数为**类型错**（block 尾 `void` ≠ 返回类型）；欲使 block 尾为 `never`（发散函数良型）须改写为 `loop {}`（无 break）或经 `throw` / `return` 出口。此保守裁断消除「全部路径 diverge」语义表述对 for/while 条件分析的悬空前向引用——三类控制流 block 尾定型各循其道（均保守语法分析、可判定）：**`if` / `match` / `select` 经 arm 出口判定**——`if` 两 arm、`match` N arm、`select` N sel_arm，当且仅当**全部 arm 体均 diverge**（`throw`/`return`/函数级出口）方使 block 尾 `never`（如 `if c { throw E } else { return }`、`match x { A: { throw E } _: { return } }`——arm 体用冻结 §16 line 692 / §27 line 1547 锁定的 `Pattern: { body }` 形，非胖箭头 `=>`；对齐冻结 §15.1 line 508 `never | T = T`——`throw`/`return` 的 match arm「不占变体」即 arm diverge、使全 arm diverge 的 match block 尾为 `never`）；**`loop` 经 break 可达性判定**；**`for`/`while` 一律 `void`**；**`try_catch` 经 try 体 + catch arm 联合出口判定**——try 体无可达正常完成出口**且**全部 catch arm 均 diverge → block 尾 `never`，否则（try 体可正常完成、或任一 catch arm 非 diverge）→ `void`（§27 line 1517 try_catch 为控制流语句、可作 block 尾）。**`break` / `continue`** 为**循环内部控制流转移**（仅合法于 `loop`/`for`/`while` 体内，§4；作用域是内层循环、**非**函数级返回出口）——不单列为函数 block 尾的 `never` 源。故 `{ if c {} else {} }`（两 arm 均空、非 diverge）→ `void`（**非** `never`），`fn f() -> int { if c {} else {} }` 为类型错（`void` ≠ `int`，不会被 `never | T = T` 误判为良型）。此规则补齐 §4 将 `loop` 降为语句（移出 `expr`）后切断的「diverging 尾 → never」路径——冻结 §15.1「`loop` 表达式类型为 `never`」为**表达式形态**（tail 位恒发散）的表征，§4 既已将 `loop` 移出 `expr` 降为语句，其 block 尾定型须按 break 可达性重判、**非**机械搬运表达式形态的 never——同时守住 `never | T = T` 不被非 diverge 控制流误触发。

**Decidability 声明（规范承诺）**：

> **类型推断可判定**：给定全显式的函数签名，函数体内的**类型推断（synth/check 等价合一求解）**在 **O(n·α(n))** 时间内终止（n = 函数体 AST 节点数，α ≈ 近线性，来自 union-find 约束求解），**且总是产生唯一类型或报告类型错误**。**范围限定**：本承诺的 O(n·α(n)) 仅覆盖**等价合一求解**（局部约束图的并查集求解）；**trait 解析**（`where { Ord<T> }` 的 bound 解析、运算符 / 方法解糖 `a + b → Add.add(a, b)` §86 #8 在全局 impl 表中查找 impl）**另行可判定**——AILang trait 系统**无关联类型、无高阶 bound**、orphan coherence（§91 #9）保证任一 `impl Trait<T>` 在有限 impl 表上**唯一匹配或类型错误**、名义类型按 head 相等不做递归 unfolding，故每次 trait 解析在**有限 impl 表上按 head 匹配、必终止**（无 Prolog 式回溯发散），复杂度随全局 impl 表大小与单态化实例如数增长、不归 union-find、不在 O(n·α(n)) 内，但**仍为规范承诺的可判定子步**。故「类型检查整体终止」= 等价合一 O(n·α(n)) 终止 ∩ trait 解析有限表终止 ∩ **auto-trait 派生（Send/Sync）终止**（见下段），三者**均**规范承诺（缺一即不构成「整体终止」）。可判定的依据：① 签名显式消除跨函数传播；② 局部 `let` 不泛化（无 HM 的 let-poly、无 principle type 通集问题）；③ 类型项有限 + 每次调用取 fresh 类型变量 + 名义类型按 head 相等（§15.3），Robinson 合一在有限项上必终止（`Box<T>` §15.6 仅保内存布局有限，**非**合一终止条件）；合一规则内置 **occurs-check**（禁止 `T = f(... T ...)` 的无限类型，如 `T = List<T>` 直接报告类型错误、不构造循环项——对齐 HM 系合一的终止保证）；④ 等价约束经 union-find 求解，有限约束集必终止（不要求 DAG）。**前置良型条件**：decidability 承诺仅覆盖**类型良型**的程序——透明别名环（`type A = B; type B = A`）、递归类型未 `Box<T>` 间接等须由前置良型 pass 拒绝（对齐 §73 实现层陷阱），不在本终止保证范围内。 **auto-trait 派生（Send / Sync，§73 / §87 #6）的可判定性**：编译器自动派生 `Send` / `Sync`（marker auto-trait）是类型检查的**第三子步**，不在上述等价合一 nor trait 解析两项内。其终止性**不**依赖冻结 §90 #5 的「无循环引用」——后者约束**值层**引用（不引入 `Rc`/`Arc`/`Weak`、单一所有者），而**类型层**经 `Box<T>` 可构成互递归字段环（`struct A { b: Box<B> }; struct B { a: Box<A> }`——派生 `A: Send` 需 `B: Send` 需 `A: Send`，朴素字段递归不终止）。故 auto-trait 派生采用 **worklist + 已访问集（cycle detection）**算法：每个 (类型, trait) 求解目标至多入队一次，遇环按「假设当前目标成立」做 coinductive 假设收敛（对齐 Rust auto-trait 语义），状态空间 = 有限类型集 × {`Send`, `Sync`}，有限故**必终止**。**开放问题**：冻结 §87 #6 的 `Send` 组合清单未显式给出 `Box<T>: Send ⟺ T: Send` 规则（见 §11 #8）；auto-trait 派生的终止性**不依赖**该规则的具体形式（worklist + cycle detection 在有限状态空间上终止，无论该规则由字段递归自然推出或需显式声明）。

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
      | "(" args? ")"                  // 仅当 lookahead 匹配 type_args "(" 时采纳上一行（类型实参），否则 < 回退为比较
type_args := "<" type_arg ("," type_arg)* ">"    // lookahead：须后跟 "("
```

- **优点**：不修改 §86 #7，`decode<User>(row)` 语法不变；零迁移成本。
- **缺点**：保留理论歧义边界——`f < g > (h)` 这类「比较链后跟括号」会被误解析为泛型调用（Rust pre-2018 的已知陷阱）；需 parser 维护一个试探回溯或优先级规则。对 AI 不够「可预测」。

### 方案 B：turbofish `::<T>`（彻底消歧，修改 §86 #7）

类型实参**必须**写作 `::<...>`（Rust现行 turbofish），`<` **永远**是比较运算符。

```
call := "::" type_args "(" args? ")" | "(" args? ")"     // 纯后缀；嵌入 §27 postfix := primary (call|index|member)*
```

> **词法前置（`::` token——采纳方案 B 的前置条件）**：上述产生式中的 `"::"` 为**单一词法 token**（TokenKind `ColonColon`），经 **maximal munch** 贪吃连续 `:`——源码 `::` 一律切为单个 token、不拆为两个 `:` token（对齐 Rust 的 `::` 词法惯例；maximal munch 规则：`::` 优先于单 `:` 匹配，使既有 `:` token——类型标注分隔 / struct 字段初始化等——与 `::` 不冲突）。`type_args` **复用方案 A 的产生式**（`type_args := "<" type_arg ("," type_arg)* ">"`，见上），故 turbofish 形 = `::` 紧跟 `<…>`；因 `::` 已先行消歧，其后的 `<` 无歧义地为类型实参开口（方案 B 下 `<` 在 turbofish `::` 外恒为比较运算符）。此为采纳方案 B 的**词法前置条件**——落地时须先在 §8.6 词法层登记 `::` token，方可解析 turbofish。

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

**命名空间**：**类型命名空间**（`type`/`struct`/`enum`/`interface`/`trait`/`error`/`actor`/`agent`）与**值命名空间**（`fn`/`let`/`const`/`field`/`tool`/`task`/`server`）**分离**——`type X` 与 `fn X` 不冲突（同名合法，按上下文分派）；`tool` 为可被引用 / 派发的运行期实体——**兼具「值命名空间成员」与「路径前缀容器」双重身份**（同 `module`/`package`）：作为值，它是 spawn / 派发的运行期对象；作为路径前缀，其体 `{ fn* }`（§27 `tool := "tool" ident "{" fn* "}"`）内含 `fn`，可经点分路径 `WebSearch.query` 作**符号引用**解析（见下「点分路径」分支），使冻结 **§60** 的 `agent { tools: [ WebSearch.query, DbQuery.query ] }`（§60 line 2443 `tool WebSearch` 声明 + line 2454 `WebSearch.query` 引用，与 `toolName.fnName` 点分路径规则一致；§27 `agent` 的 `tools` 字段为 `[expr_list]`、`WebSearch.query` 经 `postfix := primary member` 解析为 `ident.member`）可达。**注**：冻结 §23.1 line 1313 / §40.2 line 1957 的 `tools: [ web.search, database.query ]` 形**与本 RFC 规则不匹配**——其声明为 `tool Search`（§23.2 line 1320 / §40 line 1973）、无 `web` tool 在作用域，`web.search` 既不匹配声明的 `Search`、也不被 `toolName.fnName` 规则解析；该 `web.search` 形为冻结规范自身的过时示例残留、本 RFC 点分路径规则**不覆盖**（规则对齐 §60 规范形 `tool WebSearch` / `WebSearch.query`）。**`task` / `server` 仅值命名空间成员、非路径前缀容器**（文法事实）：`task` 体为单个 `block := { stmt* }`（§27 `task := "task" ident "(" params? ")" block`），而 `fn` 是顶级 `item` 不出现在 `stmt` 产生式内，故 task **无内含 `fn` 成员**——`taskName.fnName` 无文法依据；`server` 体为 `{ route* }`（§27 `server := "server" ident "{" route* "}"`、`route := "route" string_lit "{" fn* "}"`），fn 嵌于 route 内且 route 名为 `string_lit`（HTTP 路径，如 `"GET /users"`）而非 `ident`，故 `serverName.fnName`（ident 点分）文法不可达——server 的 fn 经 runtime route 派发（§21.9 行 1229、§27 文法行 1500-1501）、不经名字解析的点分路径。`task` / `server` 作为整体值（spawn / 派发对象）参与简单名解析，其内部不经点分路径暴露 `fn`。`extern` 的 `ident` 为 ABI 标签（如 `extern c`），**不入命名空间**（仅作 ABI 名、名字解析阶段校验其 ∈ 封闭调用约定集合，见 [RFC 0009](./0009-ffi-abi-supply-chain.md) §5）。**`extern` 块内的 `extern_fn`（§27 `extern_fn := "fn" ident "(" params? ")" ("->" type)?`，嵌于 `extern := "extern" ident "{" extern_fn* "}"` 块内、非顶级 `item`）在名字解析阶段**hoist 至当前模块作用域的值命名空间**（layer 3，同顶级 `fn`），可经简单名直接调用——冻结 §25.3 / §61 的 `printf("raw work")`（行 2495）、RFC 0009 §6.3 的 `puts(...)` 裸名调用均据此解析；**不**经 `abi.fn_name` 点分路径访问（`extern` 块名非命名空间实体、**非路径前缀容器**，与 `tool` 的路径前缀模型不同——`extern_fn` 嵌套于块内仅文法事实，名字解析时提升为模块级值）。`extern_fn` 的调用本身须置于 `unsafe` 块内（§25.1 ② / §90 #4，封送见 RFC 0009）。`module`/`package` 名在两者均可引用（路径前缀）。**enum variant 的命名空间归属**：variant 属其 enum 类型在类型命名空间下的**子作用域**（`Color.Red` = 类型 `Color` 的成员 `Red`）；裸 variant 名（`Red`）仅在 enum 自身作用域内直接可解析，**跨作用域引用须用点分路径 `Color.Red`**（§27 `pattern := (ident ".")? ident`，零语法增量）。**注意**：enum variant 是类型的成员、非顶级 item，不在 §12.2 / §91 #6 允许的导入集中（`from M import name` 仅可导入 M 的 public 顶级 item）——故无 `from Color import Red` 形态，跨作用域统一走点分路径。**例外（std.core lang-item variant）**：`#[lang]` 标注的 std.core enum（`Result` / `Optional`）的 variant（`Ok` / `Some` / `None`，§86 #5 / §15.8）随 std.core 自动加载（layer 5）**将其裸名一并注入当前作用域**（prelude 式），故 `Ok(x)` / `Some(x)` / `None` 可跨作用域裸用（§89 #2 显式 `Ok(x)` 无 auto-wrap、§34 裸形示范）。**`Err` 不在此列**——AILang 的 `Result<T, E>` 错误侧为 `E` 的各命名 variant、**无 `Err` 包装**（§16「错误分支为 `E` 的各 variant（无 `Err` 包装）」、§17「错误侧 = `E` 的命名变体（无 `Err` 包装）」、§89 #2 决议；§15.8 lang-item variant 列举亦仅 `Ok`/`Some`/`None`，不含 `Err`）——错误值经 `throw V∈E`（§89 #6，如 `throw NotFound(404)`，`V` 经下述类型导向解析为 `E` 的 variant）或点分路径 `E.V`（如 `UserError.NotFound`）表达，**不存在裸 `Err(e)` 构造形态**。其余（用户自定义或显式 import 的）enum 的跨作用域 variant 引用仍须点分路径 `Enum.V`，**或**经下述**类型导向解析**直接裸用。variant 构造 `Color.Red(args)` 与 variant 类型引用按上文 `Name(args)` 消歧。

**actor message 的命名空间归属与解析**：`message` 声明嵌于 `actor` 体内（§27 `actor := "actor" ident "{" (ident ":" field+)? ("message" ident ("{" field* "}")? | on_clause)* "}"`——`message` **非**顶级 `item`、不在 §27 `item` 产生式，故不入 layer 3 模块顶级 item 集、亦不在 §12.2 / §91 #6 导入集）。message 属其 actor 类型在类型命名空间下的**子作用域**——形态对称于 enum variant（variant 是 enum 类型的成员、message 是 actor 类型的成员）；**载荷 message** `message AddUser { user: User }`（§21.7）对应带字段 variant 形、**无载荷 message** `message GetCount` 镜像无字段 variant（§84 #2，裸名即构造）。**解析路径 = 类型导向（第五机制，以 send 接收者恢复的 actor 类型为锚——与 ①②③ variant 解析以 scrutinee / `throw E` / 标注类型为锚同构）**：`h ! Msg{...}` / `h.send(Msg{...})` / `h ! GetCount` / `h.send(GetCount)` 中，send 算子（§27 `send := postfix "!" expr`）/ `.send(...)` 调用的**接收者 `h`** 的 actor 类型 `A` 使消息名 `Msg` / `GetCount` 解析为 `A.Msg`。**`Name{...}` 花括号形与裸名**：`Msg{...}` 复用下文 `Name{...}` 花括号分支（struct / message 共用花括号构造形，见下「类别消歧」）；`GetCount`（无载荷）为裸名构造（镜像无字段 variant）。**`ActorHandle` 为裸类型**（冻结 §21.7 `let h: ActorHandle = spawn actor UserService()` 行 1185、§87 #4/#5、§21.2——全库**无 `ActorHandle<A>` 参数化**）：actor 类型**不在源码语法层标注于 handle**，而由类型系统**内部跟踪**（spawn 点 `spawn actor Name()` 的 `Name` 流入 handle 的隐式类型）——此为冻结 `h ! Msg{...}` 示例可类型化的**隐含前提**（否则 send 无法核对消息 ∈ actor 的消息集），本 RFC 把这一隐含机制**形式化为**「接收者 actor 类型经 spawn 点 provenance 恢复」的解析规则、**不**改动冻结 `ActorHandle` 裸语法（不引入 `ActorHandle<A>`、不触动 §87 #4/#5）。**多 actor 同名消息无歧义**：send 接收者 `h` 的恢复 actor 类型 `A` 已唯一确定（一个 handle 绑定一个 actor 类型），消息名按 `A.Msg` 解析即无歧义。**脱离 send 上下文的 message 构造**（消息名无接收者锚，如 `let m = Msg{...}` 后再 `h.send(m)`）= actor 类型不可经 send 接收者恢复 → 裸名 `Msg` = 编译错误 `UnresolvedName`；冻结示例均经 send 算子直接投递（§21.7 `h.send(AddUser{...})` / §87 #5 `h ! Msg{...}`），脱离上下文构造非冻结惯法、留 open question #7。

**类型导向（type-directed）variant 解析**（填补冻结 §44「`match` arm 用 bare variant（scrutinee 类型已知时）」+ §89 #7「惯法推荐裸变体」+ §89 #6「`throw V` 仅当 `V ∈ E`」所依赖、而作用域 / 点分路径 / prelude 三机制均未覆盖的解析路径）：当 enum / error 类型可由上下文**唯一确定**时，该 enum 的 variant 裸名可直接解析（跨词法作用域合法，是与「作用域查找」「点分路径」「std.core prelude」**并列的第四机制**）：

- ① **`match` scrutinee** 的静态类型为某 `enum` / `error` `E`（`match status { Active: … }`，`status: Status` → `Active` 解析为 `Status.Active`）；
- ② **`throw V`** 出现于返回 `Result<_, E>` 的函数体 / 处理子、且 `V` 是 `E` 的 variant（§89 #6，如 `throw NotFound(404)` → `NotFound` 解析为当前 `E` 的 `NotFound` variant）；
- ③ **显式标注的 variant 构造位**（`let x: Color = Red`，期望类型 `Color` 使 `Red` 解析为 `Color.Red`）。

多 enum 共存致上下文无法唯一确定（如两 enum 同有 `Active`）时，仍须点分路径 `Enum.V` 消歧（`AmbiguousVariant`）。此机制与 std.core prelude **正交**：prelude 处理**无类型上下文线索**的裸 `Ok` / `Some`（任意位置可裸用），类型导向处理**有上下文线索**的用户定义 variant——故无需为 std.core 单独 carve-out 即可统一解释 `Ok(x)` / `Active` / `NotFound(404)` 三类裸 variant（prelude carve-out 仍保留作 Ok/Some 的无条件可用保证，二者不冲突）。

**简单名 `ident` 解析**：

- 按作用域层级 1→5 顺序查找；**首个匹配**胜出（shadowing：内层遮蔽外层）；
- **本地定义优先于导入**（§12.2：遮蔽 + warn）；
- 导入冲突（两导入同名）= 编译错误（须 `as`，§12.2）；
- 查无 = 编译错误 `UnresolvedName`。

**点分路径 `a.b.c` 解析**：

1. `a` 按简单名解析（值或类型或模块）；
2. `.b` 按 `a` 的类别查找 member：
   - `a` 为**值**：`.b`（无括号）= 字段访问（仅当 `b` 为 struct field）；**字段与方法同名时 `.b`（无括号）优先解析为字段**。若 `b` **仅为方法**（无同名字段），则 `a.b`（无括号）= **编译错误 `UnresolvedMember`**（v0.2 无一等闭包 §21、无函数类型 §15.1/§15.9，不存在「方法引用」值；方法调用须显式 `a.b()`，对齐 §15.5 struct 内字段与 fn 方法共存）；
   - `a` 为**类型**：`.b` = enum variant（`Color.Red`）或关联 item（v0.2 无关联函数，推迟）；
   - `a` 为**模块 / package / `tool`**（路径前缀容器，见上「命名空间」）：`.b` = 容器的子 item——模块 / package 为子 item（`import std.http` 后经**末段** `http` 访问子 item，如 `http.Client`——Go 式导入仅注入末段、不注入根 `std`，§38 行 1894 / §12.1 / §91 #3）；`tool` 为其体 `{ fn* }` 内含的 `fn`（`WebSearch.query` → 解析为 `fn` 的**符号引用** DefId / 名字串，**非字段值、非一等闭包值**），仅在 `agent tools:[…]` 等**符号引用上下文**合法；普通 expr 位引用 tool 内 `fn` 仍须 `a.b()` 调用（裸 `fn` 符号非 v0.2 一等值，§21）。**`task` / `server` 不属路径前缀容器**（见上「命名空间」：task 体为 `block` 无 `fn` 成员、server 的 `fn` 经 `string_lit` route 间接包含非 ident 可达）——`taskName.b` / `serverName.b` 不走此分支（task / server 作整体值参与简单名解析，其内部不经点分路径暴露 `fn`）；
3. `.c` 递归同上；
4. 查无 = `UnresolvedMember`。

**`Name(args)` 与 `Name{...}` 的类别消歧**（填补 §89 #2/§92 #2 引用的「名字解析消歧」）——**括号形 `Name(args)` 与花括号形 `Name{...}` 是不同产生式**（§27：`struct_init := ident "{" (ident ":" expr ("," ident ":" expr)*)? "}"` 为**花括号形**、属 `primary`；`call` 为**括号形**后缀）。消歧按**文法形态 + `Name` 解析结果**双重决定：

- **`Name(args)`（括号形，`call` 后缀）**：
  - `Name` 为**函数** → 函数调用 `f(args)`；
  - `Name` 为**约束类型名**（`constraint T { ... }`，§15.4）→ 约束类型构造 `T(expr)`（§15.4 / §92 #2，零成本语义转型 + 运行期断言；单实参，违约 panic `ConstraintViolation`）。**非约束类型名**（透明别名 `type URL = string`、纯 semantic `type UserId = int`、enum 类型整体）出现在 `Name(args)` 位置 = **编译错误**（§15.4 的 `T(expr)` 仅授权约束类型；冻结 §15.3「变量间不隐式转换」无别名 / semantic 的 call 形构造算符）——运行期值（非字面量）转型到纯 semantic / 透明别名类型在 v0.3+ 由独立 RFC 定义——RFC 0007 §7 的封闭 `as` 集**仅含基础数值类型 + char/bool**、不含 semantic / 别名（`int as UserId` 为编译错误），故「运行期 `int` 变量 → 纯 semantic（无 constraint）」当前无定义路径（字面量→semantic 经 §6 #4 字面量强制、constraint 类型经 `T(expr)`，纯 semantic 仅字面量可达）。**括号形不用于 struct 构造**——struct 构造恒为花括号形（见下）。
  - `Name` 为 **std.core lang-item（`Builtin`）类型**（§86 #5 / §73 TypeDb `Builtin(Result, Optional, List, Map, …)`）——按**是否具 call 形位置构造器**二分：**具位置构造器的 Builtin**（`Mutex<T>(v)` §18.6 行 881/1157、`Channel<T>(cap)` §21.4 行 1004/1108/1109——冻结规范有真实 `Mutex<Data>(...)` / `Channel<int>(8)` 构造形）→ **内建位置构造器** `Name<T>(args)`（编译器特判：**非** `struct_init` 花括号形、**非** 用户 `fn` 调用、**非** `constraint T(expr)`；语义随各类型定义，如 `Channel<T>(cap)` 创建 MPMC 通道、`Mutex<T>(v)` 包装值，填补 §18/§21/§87 普遍用法；§7 turbofish 迁移清单已含此类调用点）；**无 call 形构造器的 Builtin**（`Future<T>` = `async fn` 调用返回值 §21.3/§87 #1、`TaskHandle` / `ActorHandle` = `spawn` / `spawn actor` 返回值 §21.2/§87 #4/#5、`List<T>` 经 `list_lit` `[a,b]`、`Map<K,V>` 经 `map_lit` `[k:v]` §27/§15.9、`Result` / `Optional` 经 variant 构造 `Ok(x)` / `Some(x)` 而非类型名 call、`Shared<T>` 多线程只读共享 §18.6 其构造形冻结规范未示——`let c: Shared<Config> = …` 行 880/1156 均为省略号、**无 `Shared<T>(args)` call 形**、待 `std.sync` API 定义）出现在 `Name(args)` 位 = **编译错误**（该类型无 call 形构造算子；冻结规范全库无 `Future<T>(…)` / `TaskHandle(…)` / `List<T>(…)` / `Map<K,V>(…)` / `Shared<T>(…)` 构造形）。
  - `Name` 为 **enum variant**（含路径 `Color.Red(args)`）→ 带载荷 variant 构造。
- **`Name { f1: e1, ... }`（花括号形，`struct_init`）**：`Name` 为 **struct 类型名** → struct 构造（§27 `struct_init`）；**`Name` 为 actor message**（经 send 接收者恢复的 actor 类型解析，见上「actor message」段；载荷 message）→ 消息构造（形态复用花括号形）。求值序见 [RFC 0007](./0007-determinism-batch.md) §4（构造器书写序）。
- **`Name<T>`（`<` 类型实参）**：`Name` 为类型名 → 泛型实例化（`List<int>`）——此为**类型表达式**、属 `type` 产生式而非 `call`（参见 §7 `<` 消歧）。
- 歧义（同名既函数又类型）→ 声明层按类型 / 值命名空间分离**无冲突**（`fn X` 与 `type X` 可并存）；但**引用层 `Name(args)`** 形态下 `Name` 同时命中值域（函数）与类型域（约束类型）时，**tie-break：值命名空间优先**（`Name(args)` 解析为函数调用）。若调用方意图为约束构造，`Name::<_>(args)` **不能**作为逃生口——turbofish（§7 方案 B）的 `<…>` 为**类型实参**，仅附加于**泛型可调用实体**，而约束类型（`constraint T`）按 §27 / §15.4 / §92 #2 **非泛型**（无 `generic` 形参、构造算符 `T(expr)` 取值实参）；且即便 §27 `type_arg := type | literal` 文法层**接受** `_`（§8.6 标识符 `[A-Za-z_][A-Za-z0-9_]*` 首字符含 `_`、零续位合法 → 词法产出 `Ident("_")` → §27 `type := ident` → `type_arg`，**非** parser 阶段拒绝 AIL1xxx），**name resolution 阶段**亦无名为 `_` 的类型 → `UnresolvedName`（AIL4xxx；`_` 仅为 pattern 通配、非类型位占位符）；且 turbofish 为**值域 call 后缀**、constraint 类型非泛型，故 `Name::<_>(args)` 解析为函数调用而非约束构造。结论：`fn X` + `constraint X` 同名在 `Name(args)` 形态下**任何方案（A/B）均无语法逃生口**，须**改名消歧**（如 `fn make_x` / `constraint X`）；此为病态罕见场景，不值得为它引入新语法。同命名空间内真歧义 = 编译错误 `AmbiguousCall`。

**shadowing 细节**：内层 `let` 遮蔽外层 `let`/参数合法（warn 可配）；遮蔽导入合法 + warn（§12.2）；**类型名不可被值遮蔽**（命名空间分离）。

> 此算法闭合 §27 多处「由名字解析消歧」的前向引用，使 Name Resolver pass 有了可实现的规范定义。

---

## 9. 不变量与自洽核查

| 本 RFC 条款 | 对齐的既有规范 | 自洽性 |
|---|---|---|
| §4 if 条件须 bool | §15 无隐式转换（仅 string+Display 例外 §11） | ✅ |
| §4 loop 为语句（非 value 表达式，break 无值）——block 尾定型见 §6 rule 6（break 可达性） | §15.1 `loop` 表达式类型 = `never`（⊥）为**表达式形态**表征 + §4 loop 移出 `expr` 降为语句 | ✅ 一致（loop 未入 expr 产生式，非演进；block 尾 never / void 由 §6 rule 6 判定，非机械搬运 §15.1） |
| §4 parallel_for/reduce 并入 §27 primary（expr 可达） | §87 #9 parallel 表达式化 + §21.10/§58 示例 | ✅ 已并入 primary（§4 `primary` 体已给出），`let x = parallel for…` 可派生 |
| §5 无后缀字面量 + 默认类型 | §8.6 现状 + §15.1 int=i64/float=f64 + §90 #2 | ✅ |
| §5 char_lit → char 类型 | RFC 0007 §8 引入 char 类型（跨 RFC，同批落地） | ✅ 词法在本 RFC、类型在 0007 §8 |
| §6 签名显式 + 局部推断 | §15.2 现状两句 + §15.10 泛型 where | ✅（展开） |
| §6 return-only 泛型显式 | §86 #7 | ✅ |
| §6 rule 6 loop block 尾定型 = break 可达性（无可达 `break` → `never`；含可达 `break` → `void`） | §15.1 `loop` 表达式类型 = `never`（表达式形态表征）+ §16 loop-break 惯用法 `loop { work(); if done { break } }` + §4 loop 移出 `expr` 降为语句 | ✅ soundness 闭合（修正「无条件 `loop` → never」会接受缺返回值函数 `fn f() -> int { loop { if done { break } } }` 的缺口；对齐 Rust `loop { break; }` 类型 = `()` 而 `loop {}` = `!`；可达 break 判定为保守语法分析、可判定） |
| §7 方案 B 修改 `<T>` + `::` 词法 token 登记 | §86 #7（**显式演进**，唯一例外）+ §8.6 词法层 `::` maximal munch | ⚠️ 需决断（含 `::` token 词法前置条件） |
| §8 类型/值命名空间分离 | §89 #2/§92 #2 名字消歧引用 | ✅ 填补 |
| §8 `Name{...}`(struct_init 花括号) vs `Name(args)`(constraint/variant 括号) | §27 `struct_init := ident "{" ... "}"`（花括号形·primary）+ §15.4/§92 #2 `T(expr)`（括号形·call） | ✅ 形态分离（不混 call） |
| §8 类型导向 variant 解析（match scrutinee / `throw V∈E` / 标注位） | §44「match arm bare variant（scrutinee 已知）」+ §89 #7 裸变体惯法 + §89 #6 `throw V∈E` | ✅ 填补（第四解析机制，与作用域 / 点分 / prelude 并列、正交） |
| §8 `tool` 作路径前缀容器（`WebSearch.query` 符号引用）；`task`/`server` 仅值成员、非路径前缀容器 | §27 `tool := "tool" ident "{" fn* "}"`（体含 fn ✓）/ `task := "task" ident "(" params? ")" block`（体为 stmt* 无 fn）/ `server := "server" ident "{" route* "}"`（fn 经 `string_lit` route 间接包含）+ **§60** `agent { tools: [WebSearch.query] }`（仅 tool 走点分路径） | ✅ 收窄至 tool（文法事实：仅 tool 体 `{ fn* }`）；task/server 的 `.b` 不走路径前缀分支；**§23.1/§40.2 的 `web.search` 形为冻结残留、规则不覆盖**（见 §8 注） |
| §8 Builtin lang-item 位置构造器（`Mutex<T>(args)` / `Channel<T>(cap)`；Future/TaskHandle/List/Map 无 call 形构造 → 编译错误） | §86 #5 / §73 TypeDb `Builtin(...)` + §18.6 `Mutex<Data>(...)` / §21.4 `Channel<int>(8)` 构造用法 | ✅ 填补 + 二分（具构造器 Mutex/Channel vs 无构造器 Future/TaskHandle/List/Map，编译器特判，非 struct_init / fn / constraint） |
| §8 actor message 解析（send 接收者恢复 actor 类型 → 消息名解析；`Name{...}`/裸名；ActorHandle 裸、类型系统内部跟踪 spawn provenance） | §27 `actor`/`message`/`send` 产生式 + §21.7 `h.send(AddUser{...})`/§87 #5 `h ! Msg{...}` + §84 #2 无载荷 message 镜像 variant + §21.7 裸 `ActorHandle`（行 1185） | ✅ 填补（第五解析机制，以 send 接收者为锚、与 variant 类型导向同构；ActorHandle 裸 = actor 类型经 spawn provenance 内部恢复、**非**源码参数化、不触动冻结语义；脱离 send 上下文构造见 open question #7） |
| §8 extern_fn hoist 至模块作用域值命名空间（layer 3，裸名调用；不经 `abi.fn` 点分路径） | §27 `extern := "extern" ident "{" extern_fn* "}"` + `extern_fn := "fn" ident ...`（嵌套、非顶级 item）+ §25.3/§61 `printf("raw work")` 裸名（行 2495）+ RFC 0009 §6.3 `puts(...)` | ✅ 填补（5 层算法遗漏 case；extern 块名非命名空间实体 = 非路径前缀容器，与 `tool` 模型区分；extern_fn 调用须 `unsafe` 块 §25.1②） |
| §8 turbofish 非约束构造逃生口（**已撤回**） | §27 `type_arg := type \| literal` 文法层接受 `_`（§8.6 标识符含 `_` 首字符、零续位合法）但 **name resolution 拒绝**（无名为 `_` 的类型 → `UnresolvedName` AIL4xxx，**非** parser AIL1xxx）+ constraint 非泛型（§27/§15.4/§92 #2）+ turbofish 为值域 call 后缀 | ✅ 诚实登记「fn+constraint 同名须改名消歧」（A/B 均无逃生口；拒绝发生在 name resolution/typecheck 阶段，非 parser 文法层） |
| 零新关键字 | §9（56 表）；`break`/`continue` 已在控制流类别（§83 #5 锁定）；`in` 登记为上下文关键字（不入 56，落地于 §9 行 393 段落） | ✅ 已核查含 |

---

## 10. 落地映射（合并进 AILANG.md 的位置）

| 本 RFC 内容 | 落地位置 | 性质 |
|---|---|---|
| §4 控制流体产生式（if/for/while/loop/break/continue） | §27 `stmt` 产生式展开（替换裸终结符） | 规范性（Part I 文法） |
| §4 `in` 上下文关键字登记 | §9 上下文关键字段落（行 393） | 规范性（Part I 词法） |
| §4 `parallel_for` / `parallel_reduce` | §27 `primary` 产生式（表达式位置，§87 #9；`let x = parallel for …` 经 expr 链派生） | 规范性（Part I 文法） |
| §5 字面量词法 | §8.6 扩充 | 规范性（Part I 词法） |
| §6 推断算法 + decidability | §15.2 扩充 | 规范性（Part I 类型系统） |
| §7 `<` 消歧（方案 A/B） | §27 `call` 产生式 + §86 #7 示例（若 B）+ §8.6 `::` token 登记（方案 B 词法前置） | 规范性（若 B 则含决断演进） |
| §8 名字解析算法 | §12 新增 §12.4 | 规范性（Part I 模块） |

---

## 11. 开放问题

1. **`loop` 作 value 表达式（推迟）**——本 RFC 保持 `loop` 为语句、体类型 `never`、`break` 无值（对齐冻结 §15.1，零语义演进，loop 未入 §27 `expr` 产生式）。若未来要让 `loop` 产出值（`break expr` + loop 表达式类型 = break 值类型，多 break 点同型合流），须作为对 §15.1 的显式演进另立 RFC（走 RFC 0005 §9 解冻通道）并同时把 `loop` 加入 `expr` 产生式——本 RFC 不引入此项。
2. **`parallel_reduce` 的 `op` lambda 字面量**——§4 `parallel_reduce` 的 `op` 实参为 lambda（如 `(a, b) -> a + b`），其形式依赖一等闭包；§21 冻结声明 v0.2 无一等闭包，故 §21.10 / §58 的 `parallel reduce(…)` 示例为前向示例。本 RFC 只锁产生式骨架（body 用 `arm_body`）；op 字面量的完整定义随闭包 RFC 落地。
3. **原始字符串 / 字节串**——§5 仅定义基础 `string_lit`；`r"..."` 原始字符串、`b"..."` 字节串推迟 v0.4（与 raw_pointer/字节处理一并）。
4. **`<` 消歧方案 A/B 最终决断**——§7 推荐 B，但需 review + 对抗验证评估迁移成本与歧义边界后定。
5. **推断算法的 const fn / CTFE 边界**——§6 仅覆盖运行期类型推断；常量求值域（const fn / CTFE）是 spec-maturity 最薄弱单点（~5%），但其形式化超出 P0-1 范围，留独立 RFC（P1+）。
6. **类型推断与 borrow checker 的交互**——§6 的推断独立于所有权（§18/§74）；二者在 codegen 前的 pass 顺序（infer → borrow）待编译器实现层定（§65+）。
7. **actor message 脱离 send 上下文的构造 + `ActorHandle` 类型化机制**——§8 把消息名解析锚定在 send 接收者（`h`）经 spawn 点 provenance 恢复的 actor 类型上，覆盖冻结惯法（`h.send(Msg{...})` / `h ! Msg{...}`，§21.7 / §87 #5）。但两处仍欠冻结定义：① **脱离 send 算子直接构造 message**（`let m = Msg{...}; h.send(m)`，消息名无接收者锚）当前无解析路径 = 编译错误 `UnresolvedName`（冻结无此惯法）；② 冻结 `ActorHandle` 为**裸类型**（无 `ActorHandle<A>` 参数化，§21.7 行 1185），actor 类型由类型系统内部跟踪——是否在未来 RFC 把 `ActorHandle` 显式参数化为 `ActorHandle<A>`（使 actor 类型在源码层可见、支持脱离 send 的消息构造推断与跨函数传递 handle 时的消息集核对），留后续 RFC（若引入须触动 §87 #4/#5、走 RFC 0005 §9 解冻通道）。本 RFC 在该决断前**不**引入 `ActorHandle<A>`、不触动冻结语义。
8. **auto-trait（Send / Sync）派生规则与 `Box<T>` 的显式规则**——§6 decidability 声明把 auto-trait 派生列为类型检查第三子步（worklist + cycle detection，有限状态空间上终止）。冻结 §87 #6 的 `Send` 组合清单（line 864）未显式给出 `Box<T>: Send ⟺ T: Send`（及 `Sync`）规则——本 RFC 不擅自补 §87 #6（冻结、不可改）；该规则的精确措辞（字段递归自然推出 vs 显式规则、`Box<T>` 在 unsafe 内部可变性下的 `Sync` 边界）留 §87 #6 的后续解冻决议（走 RFC 0005 §9 通道）。auto-trait 派生的**终止性**已在 §6 闭合（不依赖此规则形式）；此处仅登记**规则形式**为开放。

---

## 12. 收敛轨迹

**收敛轨迹**：已跑多轮对抗式 workflow（pass-2 → pass-14，与 RFC 0005/0007 同批 pipeline，每轮修正后 FRESH 重跑、无 `resumeFromRunId`）。历轮累计修正：pass-2 报 1H/5M/5L（11 条，含 §3 Conformance 重定向后的回归扫描、`<` 消歧方案 B 迁移清单、EBNF 产生式自洽、decidability 论证）；pass-6/pass-10/pass-11 多轮 M/L（类型导向 variant 解析、`Err` 无包装、actor message 解析等）；**pass-14 报 2L 并已全部修正**：① §6 rule 6 block 尾定型示例 match arm 误用胖箭头 `=>` → 改冻结 §16/§27 锁定的 `Pattern: { body }` 形；② §8/§9 `WebSearch.query` 引用由「§23 / §60」收窄至**仅 §60**（§23.1/§40.2 的 `web.search` 形与自身 `tool Search` 声明不一致、为本 RFC 点分路径规则不覆盖的冻结残留示例）。post-pass-14 待 pass-15 复验收敛。审查维度：EBNF 产生式自洽 / 词法优先级无冲突 / 推断算法 decidability 证明严谨性 / `<` 消歧两方案 tradeoff 完备性 / 名字解析与 §89 §92 §12.2 对齐 / 与 §16 §87 §86 既有语义无矛盾 / 关键字地位 / AI 可消费性。

> 本 RFC 是形式化层（lexer/parser/typechecker/name-resolver 四 pass 前置），验证重心在**文法/算法的自洽与完备**，比 0001–0004 的实现可行性更偏理论，但比 0005（纯元规则）更需算法正确性论证。

---

*本 RFC 由综合判断 [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 P0-1 驱动。落地后将闭合「§27 推不出自身示例」「名字解析是最大盲区」「类型推断无 decidability」三个形式化层关键缺口，使 AILang 规范从「骨架已具、内核未形」推进到「编译器四 pass 可依据形式化定义开工」。*
