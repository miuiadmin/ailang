# AILang Compiler Design v0.2

> **状态：草案 / 供 review（对齐 v0.2.1 语法）**
> 范围：编译器架构与各模块**实现**设计。语言规范见 [SPEC.md](./SPEC.md)；概览见 [WHITEPAPER.md](./WHITEPAPER.md)。
> 目标读者：准备动手写编译器的人。每个模块按 **职责 / 关键设计 / 数据结构 / 陷阱** 书写。

---

## 0. 设计哲学（与实现的对应）

AILang 的卖点是 **AI-native**：编译产物里有一份机器可读的、富语义的 IR（`*.ailmeta`），AI 直接消费。

这条主张对编译器架构有一个硬性推论：

> Parser 只能看到源码字面，看不到类型/效果/所有权。
> AI 要读的那些语义信息，是**分析**出来的，不是**解析**出来的。

因此 AST 必须分层：

```
raw AST   ← Parser 产出，只有结构 + doc 注释 + 显式注解
   │  语义分析（类型 / 约束 / 效果 / 所有权 / 契约）
   ▼
AILang IR (HIR)   ← 富语义 IR。.ailmeta 从这里生成。AILang 存在的理由。
```

**HIR（AILang IR）就是「AI-native AST」的正式名字。** 把它和 raw AST 分开，是本架构最关键的决定，决定每个模块的边界。

---

## 1. 总体架构

```
.ail 源代码
   │  Lexer              词法分析
   ▼
Token 流 (+ trivia: doc 注释)
   │  Parser             语法分析
   ▼
raw AST  +  诊断
   │  ┌────────────┬─────────────────┬──────────────┐
   │  ▼            ▼                 ▼              │
   │  Type      Ownership         AI Analyzer       │
   │  Checker   / Borrow          (知识图谱)        │
   │            Checker                             │
   │  └────────────┴─────────────────┴──────────────┘
   ▼
AILang IR (HIR)  +  诊断
   ├──► AI Metadata      ──►  program.ailmeta (JSON)
   ▼
Optimization
   ▼
LLVM IR
   ▼
Native Machine Code  (.exe / ELF / Mach-O)
```

设计原则：

- **每个阶段独立可测**。阶段间用纯数据结构传递（AST、HIR），无共享可变状态。
- **多错误上报**。任何阶段都不在第一个错误处停摆（rustc 风格）。
- **Span 贯穿全程**。每个 AST/HIR 节点带 `Span`，用于诊断与 `.ailmeta` 源码定位。
- **诊断统一类型**。所有阶段产出同一种 `Diagnostic`，由 `ailc` 收集打印。

---

## 2. 实现语言与工具

- **宿主语言：Rust**。理由：内存安全、性能高、适合写编译器、LLVM 生态成熟。
- **词法：`logos`**（高性能声明式 tokenizer）。
- **语法：手写递归下降 + Pratt 表达式解析**（不用 parser generator——v0.2 的 doc 注释附着、契约块、可选分号等需要完全控制与最佳错误恢复）。
- **LLVM 绑定：`inkwell`**（安全高层封装；pin 一个 LLVM 版本）。
- **元数据序列化：`serde` + `serde_json`**（`.ailmeta` JSON）。

---

## 3. 项目结构

```
ailang/                       # 仓库根 / Cargo workspace
├── Cargo.toml                # [workspace]
├── docs/                     # WHITEPAPER / SPEC / PHILOSOPHY / DESIGN / STDLIB / MEMORY / CONCURRENCY / EXAMPLES
├── compiler/                 # 编译器（分阶段 crate）
│   ├── ail-ast/              # AST + HIR 类型、Span、Symbol、Diagnostic（公共地基）
│   ├── ail-lexer/            # logos tokenizer
│   ├── ail-parser/           # 手写递归下降 + Pratt
│   ├── ail-checker/          # 类型检查器（含语义/约束类型）
│   ├── ail-ownership/        # 所有权 + Borrow Checker
│   ├── ail-analyzer/         # AI Analyzer（效果/契约 → 知识图谱）
│   ├── ail-ir/               # AILang IR (HIR) 构造
│   ├── ail-codegen/          # codegen 编排（IR → LLVM）
│   └── ailc/                 # 编译器二进制（rustc 式）
├── std/                      # 标准库（.ail 源 + 原生 runtime）
├── package/                  # ail 包管理器（cargo 式）
├── cli/                      # ail CLI（new/build/run/test/publish）
└── tests/
    └── golden/*.ail          # 黄金测试：源码 → AST/HIR/ailmeta 期望输出
```

分 crate 是**用 Cargo 强制阶段边界**：`ail-parser` 无法依赖 `ail-codegen`，循环依赖在编译期被挡掉。crate 统一 `ail-*` 前缀；编译器二进制 `ailc`；包/构建工具 `ail`（见 §14）。

依赖方向（只允许向下）：

```
ailc ──► parser ──► lexer ──► ast
     └► checker ──► ast
     └► ownership ──► ast
     └► analyzer ──► ast
     └► ir ──► ast
     └► codegen ──► ir
```

---

## 4. 公共基础（ail-ast）

### 4.1 Span
```rust
#[derive(Clone, Copy, Debug)]
pub struct Span { pub start: u32, pub end: u32, pub line: u32, pub col: u32 }
```
每个 AST/HIR 节点带 span。`ailc` 持源码 + 行表，把 span 渲染为 `main.ail:12:5` 并截取上下文。

### 4.2 Symbol（字符串驻留）
```rust
pub struct Symbol(pub u32);   // 进 interner 后的句柄
```
标识符、字符串字面量、doc 文本全部驻留；同名串只存一份，比较退化为 u32 比较。AST/HIR 要序列化进 `.ailmeta`，用 Symbol 比到处 `String` 省内存、省比较。

### 4.3 Diagnostic
```rust
pub struct Diagnostic {
    pub severity: Severity,        // Error | Warning | Note
    pub code: Option<&'static str>,
    pub message: String,
    pub span: Span,
    pub notes: Vec<(String, Option<Span>)>,
}
```
所有阶段输出 `Vec<Diagnostic>`。**绝不在阶段内部 panic / `eprintln!`**——一切走 Diagnostic，由 `ailc` 去重、排序、彩色打印。

---

## 5. 模块一：Lexer（ail-lexer）

### 职责
源码字节流 → Token 流。不做语义判断。

### 关键设计
- **`logos` 声明式 tokenizer**，零依赖、高性能。
- **大括号语法，非缩进敏感**（v0.2 锁定）。缩进仅为可读性，不影响语义。
- **doc 注释是一等公民**：`/// ...` 不丢弃，作为 trivia 保留，供 Parser 附着到下一个声明——这是 `meaning` 的来源之一。
- **可选分号（Go 风格）**：`;` 是合法 token，但**不强制**。换行作为语句分隔（见 Parser §6 的换行规则）。lexer 需保留换行位置信息供 parser 判断语句边界。
- **隐式行连接**：`() [] {}` 内换行忽略。
- **错误恢复**：词法错误（未闭合字符串、非法字符）产 `Error` token + 诊断，继续推进。

### 数据结构
```rust
pub enum TokenKind {
    IntLit(i64), FloatLit(f64), StrLit(Symbol), BoolLit(bool),
    Ident(Symbol),
    // 关键字（56，权威列表见 SPEC §2；v0.2.1 收敛）
    KwPackage, KwImport, KwFrom, KwAs,
    KwLet, KwVar, KwConst,
    KwFn, KwReturn, KwAsync, KwAwait, KwSelf,
    KwType, KwStruct, KwEnum, KwInterface, KwTrait, KwImplements,
    KwIf, KwElse, KwMatch, KwFor, KwWhile, KwLoop, KwBreak, KwContinue,
    KwError, KwTry, KwCatch, KwThrow,
    KwTask, KwSpawn, KwChannel, KwSelect,
    KwActor, KwMessage, KwParallel, KwServer, KwRoute, KwCancel,
    KwMeaning, KwEffects, KwPure, KwRequires, KwEnsures, KwAgent, KwTool,
    KwMove, KwBorrow, KwBorrowMut, KwCopy, KwUnsafe, KwExtern,
    KwPublic, KwPrivate,
    KwTest,
    // 标点与运算符
    Arrow, Colon, Semicolon, Comma, Dot,
    Eq, EqEq, BangEq, Lt, Gt, Le, Ge,
    Plus, Minus, Star, Slash, Percent, Bang,
    LParen, RParen, LBrace, RBrace, LBracket, RBracket,
    Newline,   // 供 parser 的可选分号规则使用
    Eof,
}
pub struct Token { pub kind: TokenKind, pub span: Span }
```

### 陷阱
- 可选分号的语言，lexer 必须**保留换行 token**（或至少换行位置），parser 才能判断「语句是否结束」。Go 在 lexer 自动插入分号；AILang 采用「parser 以换行为分隔、`;` 可选」更灵活，但 parser 需处理续行例外。
- 嵌套块注释 `/* /* */ */`：v0.2 支持（深度计数器）。
- Unicode 标识符：v0.2 仅 ASCII。

---

## 6. 模块二：Parser（ail-parser）

### 职责
Token 流 → raw AST。只做语法，不做语义。

### 关键设计
- **手写递归下降 + Pratt 表达式解析**。
- **可选分号的换行规则**：换行默认结束语句；例外——行尾是二元运算符、逗号、未闭合的 `(` `{` `[`、`->` 等——则续行。`;` 显式结束语句。这是 v0.2 最易出错的语法点。
- **doc 注释附着**：解析声明时回看紧邻的连续 `///` trivia（不被空行打断）拼成 `DocComment`，附着到声明；参数级 doc 同理附着到 `Param`。
- **契约块位置**（v0.2 锁定）：`fn name(params) -> RetType` 之后、函数体 `{ }` 之前，可出现 `effects { }` / `requires { }` / `ensures { }` 契约块；`pure` 是 `fn` 前缀修饰。
- **错误恢复**：遇错合成 `Error` 节点，跳到同步点（`}` `;` 或下一个顶层声明起始关键字）继续。一次编译报多个语法错。
- **保留所有 span**。

### 文法（EBNF 草图，v0.2 大括号；权威见 SPEC §18）
```
module      := "package" module_path import* item*       // 包名=完整点分路径（SPEC §28 #1/#3）
import      := "import" module_path ("as" ident)?        // 全路径（SPEC §28 #4）；禁 import *
            |  "from" module_path "import" ident ("as" ident)?   // 仅可导入 public item（SPEC §28 #6）
module_path := ident ("." ident)*                        // 如 std.http / app.utils（SPEC §28 #1）
visibility  := "public" | "private"                      // SPEC §28 #1/#2（修饰 item + 字段 + 方法）
item        := visibility? ( fn | struct | enum | interface | trait
                           | error | type_alias | meaning | constraint
                           | agent | tool | actor | task | server | test | extern )
fn          := ("pure")? ("async")? "fn" ident generic? "(" params? ")"
              "->" type where_clause? contract* block
fn_sig      := ("pure")? ("async")? "fn" ident generic? "(" params? ")" ("->" type)?
              where_clause? contract*            // 签名（无 body）；interface/trait 用
where_clause := "where" "{" bound* "}"           // 编译期 trait bound（SPEC §23 #2）
bound       := ident "<" type ">"                // 如 Ord<T>、Eq<T>（SPEC §23 #8 运算符 trait）
contract    := "effects" "{" effect* "}"
            |  "requires" "{" expr* "}"          // 运行期前置断言
            |  "ensures" "{" expr* "}"           // 运行期后置断言
params      := param ("," param)*
param       := doc? ident ":" type
layout      := "layout" "(" ident ")"            // 如 layout(C)；对接 FFI
struct      := layout? "struct" ident generic? ("implements" type_list)? "{" (field | fn)* "}"  // 内联方法（SPEC §7.5）
field       := doc? ident ":" type
enum        := "enum" ident "{" variant* "}"
interface   := "interface" ident generic? "{" fn_sig* "}"   // 方法禁泛型（SPEC §23 #1）
trait       := "trait" ident generic? "{" (fn_sig | fn)* "}"   // 可带默认方法（SPEC §23 #6）
error       := "error" ident "{" variant* "}"
type_alias  := "type" ident "=" type             // SPEC §29 #1：无 meaning=alias(透明)；附 meaning=semantic(名义)
meaning     := "meaning" ident ":" string_lit_or_block   // SPEC §29 #5：目标 ident 须引用同模块已声明 type；块形多行
constraint  := "constraint" ident ":" "{" expr* "}"      // SPEC §29 #5：目标 ident 须引用已声明 type；value 绑定 SPEC §29 #4
agent       := "agent" ident "{" ("goal" ":" string_lit)? ("tools" ":" "[" expr_list "]")? "}"
tool        := "tool" ident "{" fn* "}"
actor       := "actor" ident "{" ("state" ":" field+)? ("message" ident ("{" field* "}")? | on_clause)* "}"  // SPEC §24 #5
on_clause   := "on" ident arm_body               // on AddUser { msg -> ... }（绑定复用 match，SPEC §8）
task        := "task" ident "(" params? ")" block
server      := "server" ident "{" route* "}"
route       := "route" string_lit "{" fn* "}"
test        := "test" string_lit block
extern      := "extern" ident "{" extern_fn* "}"   // 如 extern c { ... }
extern_fn   := "fn" ident "(" params? ")" ("->" type)?
type        := ident ( "<" type_arg ("," type_arg)* ">" )?
type_arg    := type | "const" ident ":" ident     // const 泛型实参（SPEC §23 #3），如 Array<int, 3>
block       := "{" stmt* "}"
stmt        := "let" ident (":" type)? "=" expr
            | "var" ident (":" type)? "=" expr
            | "return" expr?
            | "throw" expr                        // 语句级，类型 never（SPEC §22 #10、SPEC §26 #4）
            | "if" | "for" | "while" | "loop" | "match"
            | "spawn" ("detached")? "actor"? ( postfix | block ) | "cancel" expr | "parallel" ...  // SPEC §24 #4/#5/#9（postfix 含被调者＋后缀，镜像 SPEC §18）
            | "unsafe" block | expr
expr        := pratt（见下）
```
> 仅为骨架；`select`、`try_catch`、`match` arm 等完整产生式见 SPEC §18。

### 表达式优先级（Pratt，低→高；与 SPEC §3b 一致）
```
 0  =  +=  -=                    赋值（语句级，右结合）
 1  ||
 2  &&
 3  == != < > <= >=
 4  + -
 5  * / %
 6  unary: - ! borrow borrow_mut move copy await try
 7  postfix: call()  index[]  field.
 8  primary: literal | ident | "(" expr ")" | struct_init     // Variant(args) 复用 call 文法（SPEC §26 #2）
```
`borrow` / `borrow_mut` / `move` / `copy` / `await` / `try` 作为一元前缀运算符（`await`/`try` 紧贴 postfix：`await f()` / `try f()`，SPEC §24 #3、SPEC §26 #1）。`try expr` = unwrap-or-propagate（类 Rust `?`，但无 `From` 隐式转换——`E1≠E2` 类型错，SPEC §26 #5）；`try { } catch { }` 为局部捕获（SPEC §18 `try_catch`、SPEC §26 #1）。`throw` 为**语句级控制流构造**（非运算符），类型 `never`（⊥，SPEC §26 #4），见 SPEC §9、SPEC §26 #6。

### 陷阱
- **可选分号 + return 表达式**：`return a + b` 换行后结束；但 `return a\n + b` 是否续行需明确规则——v0.2 规定「二元运算符在行尾 → 续行」。
- 契约块与函数体都用 `{ }`，解析时 `effects/requires/ensures` 关键字先行即可区分。
- `match` arm 语法（v0.2.1 锁定）：`Pattern: { 绑定 -> body }`（见 SPEC §8；SPEC §20 #9 已锁定）。

---

## 7. 模块三：AST 设计（raw AST ↔ AILang IR）

AST 同时服务**编译器、AI、IDE**——但 raw AST 与 HIR 是两套独立类型（见 §0）。

### raw AST（节选）
```rust
pub struct FnDecl {
    pub span: Span,
    pub doc: Option<DocComment>,
    pub visibility: Visibility,       // Public | Private
    pub is_pure: bool,
    pub is_async: bool,
    pub name: Ident,
    pub generics: Vec<GenericParam>,
    pub params: Vec<Param>,
    pub ret_ty: TypeRef,
    pub contracts: Vec<Contract>,     // effects / requires / ensures
    pub body: Option<Block>,
}
pub enum Contract {
    Effects(Vec<EffectPath>),
    Requires(Vec<Expr>),
    Ensures(Vec<Expr>),
}
pub struct Param { pub span: Span, pub doc: Option<DocComment>,
                   pub name: Ident, pub ty: TypeRef, pub mode: ParamMode }
pub enum ParamMode { Move, Borrow, BorrowMut, Copy }   // 4 值（mode 权威，SPEC §25 #3 / SPEC §10.1）
```

### HIR / AILang IR（AI 读取的富语义 IR，节选）
```rust
pub struct HirFn {
    pub id: DefId, pub span: Span,
    pub identity: Identity,            // { name, description(from doc) }
    pub params: Vec<HirParam>,         // 解析后类型 + meaning + mode（mode 权威，SPEC §25 #3）
    pub output: HirOutput,
    pub effects: ResolvedEffects,      // 总是完整推断集
    pub errors: Vec<HirErrorRef>,      // 从 Result<T,E> 派生
    pub ownership: OwnershipSig,       // { output: New|Move }（无 inputs——mode 在 params；无 Borrowed——SPEC §10.3 禁 borrow 返回，SPEC §25 #2/#3）
    pub body: HirBlock,
}
```

### 关键设计决策
- **`type X = Y` 按 SPEC §29 #1 三态分类**（对齐 SPEC §7.3）：
  - **裸 `type X = Y`（无 `meaning`/`constraint`）→ 透明别名（`kind: alias`）**：编译期展开，`X` 与 `Y` 同型可互转（如 `type URL = string`）。零成本。
  - **附 `meaning` → 名义 semantic（`kind: semantic`）**：与 base 不同型，底层共享 → 零成本（`type UserId = int` + `meaning UserId: "..."`）。`UserId` ≠ `int`。
  - **再附 `constraint` → semantic + constraint（仍 `kind: semantic`，`constraint` 为可选字段）**。
  - 即 **`meaning`/`constraint` 的存在触发 nominal**。
- **运算继承边界**（SPEC §29 #7；marker/operational 二分）：semantic 新型**继承 base 的 marker trait**（`Copy`/`Send`/`Sync` 随 base——`type UserId = int` 即 Copy+Send+Sync）；但**不自动继承 operational trait**（`Add`/`Sub`/`Mul`/`Eq`/`Ord`/... 须显式 `impl`，类 Rust newtype）。**唯一例外：`Display`**（自动经 base）+ 整型字面量强制（`let u: UserId = 100`）。
- **`meaning` / `constraint` 是独立声明**（SPEC §29 #5），语义分析阶段关联到对应 `type`（目标 `ident` 须引用同模块已声明 `type`）；结构化 `meaning` 权威、`///` 补充、冲突 `meaning` 胜（SPEC §20 #10、SPEC §29 #8 已锁定）。

---

## 8. 模块四：Type Checker（ail-checker）

### 职责
名字解析 + 类型推导/检查 + 语义类型 + 约束类型 + 泛型 + 从 `Result` 派生 errors。

### 关键设计
- **TypeDb**：`DefId → TypeDef`（Primitive / Struct{is_copy} / Alias{base} / Semantic{base, meaning?, constraint?} / Enum / Interface / Trait / Builtin(Result, Optional, List, Map, ...)）。`Alias` = 裸 `type X = Y`（透明，SPEC §29 #1）；`Semantic` = 附 `meaning`（+ 可选 `constraint`）。`Builtin(...)` 即 **lang item** 集合（`#[lang]`，编译器特判：errors 派生 / `try/catch` 解糖 / 特权 match；SPEC §23 #5）。
- **类型推断**：局部可推断；**函数参数与返回必须显式**（公共边界，AI 需明确 API）。泛型 `T` 在参数中可推断；return-only 泛型须调用点显式 `<T>`（SPEC §23 #7）。
- **语义类型检查**：`UserId` 与 `ProductId` 底层都是 `int`，但名义不同 → `get_user(product_id)` 报 `Type mismatch: Expected UserId, Found ProductId`。字面量可强制到整型语义类型；变量间不隐式转换。泛型构造器**不变**（`List<UserId>` ≠ `List<int>`，SPEC §23 #9）。
- **约束类型检查**：字面量编译期检查（`let age: Age = -10` → `Constraint violation: Age must be 0..150`）；运行期经构造算符 `T(expr)`（SPEC §29 #2）插断言，违约 panic `ConstraintViolation`（SPEC §29 #9）。
- **`is_copy` 自动判定**：所有字段为 Copy 或 COW（COW 计为 Copy-compatible，SPEC §22 #7）→ struct 为 Copy；含任一 Move/`Resource` 字段 → Move。
- **trait vs interface 派发二分**（SPEC §23 #10）：`trait` 作泛型 bound（`where { }`，SPEC §23 #2）→ **编译期单态化**；`interface` → **运行期 dyn 派发**（vtable）→ **方法禁止泛型**（SPEC §23 #1）。
- **`Send`/`Sync` auto-trait 推导**（SPEC §24 #6）：编译器按字段组合自动派生 `Send`/`Sync`（marker lang item，SPEC §23 #5）——Copy→Send+Sync、COW（原子 rc）→ Send、Move 资源→Send 非 Sync、`Mutex<T>`/`Shared<T>` 按规则；`spawn`/`Channel.send`/actor 投递边界静态查 `Send`（类 Rust，无需手写 `impl`）。
- **errors 派生**：返回 `Result<T,E>` 且 `E` 为 `error` 枚举 → variant 进入 `HirFn.errors`，进 `.ailmeta`。

### 陷阱
- 整数字面量强制规则要精确（可强制到整型 primitive 及以其为 base 的语义类型；显式 `int` 变量不隐式转 `UserId`）。
- 递归 struct / 循环 type alias 检测。

---

## 9. 模块五：Ownership + Borrow Checker（ail-ownership）

### 职责
类型检查通过后，对每个函数体做所有权数据流分析。

### 关键设计
> **Single Owner Model**：任何资源同时只有一个 owner。borrow / borrow_mut **不逃逸**（禁存 struct 字段、禁返回、禁跨 await）——这是无生命周期标注的根因。

### 9.1 所有权状态机
对每个变量按语句推进状态：
```
Uninitialized → Alive → Moved
                 ↓
              Borrowed / Frozen（被借用期间）
```
- `process(file)` → move → `print(file)` 报 **Use after move**。
- 值语义三分类：位复制 Copy（`int/uint/float/bool/byte` + 全 Copy struct）、COW 值类型（`string/List/Map/Set`，赋值共享·修改克隆）、Move（`File/Socket` 等资源，需显式 `borrow`/`borrow_mut`，不可 `copy`）。

### 9.2 Borrow Graph（无显式生命周期）
内部建立 Borrow Graph，记录每次借用的 owner / 借用方 / 生命周期（恒等于当前语句，由 Lifetime Analyzer 推导）。
- **只读借用** `borrow x`：允许同时多个；存在期间原值冻结（不可 move/mut）。
- **可写借用** `borrow_mut x`：独占——同时只能一个 mutable borrow，且不能与任何只读借用并存。

### 9.3 线程安全
自动分析跨线程移动/共享（类 Rust `Send + Sync`）：`spawn process(data)` 时检查 `data` 是否可跨线程移动，不可则报错。

### 陷阱
- **v0.2 引入 `borrow_mut`**（推翻 v0.1「仅不可变借用」），但同样不逃逸 → 仍无需生命周期。
- **Drop 顺序 + 无循环引用（SPEC §27 #5）**：字段按**声明顺序**释放（确定性）；安全代码单一所有者 + COW 缓冲共享 → 结构上无环，v0.2 无 `Weak`/`Rc`/`Arc`；codegen 按声明序插 `release`/drop，panic 展开同序（SPEC §26 #3）。
- **Phase 1-2 缺 borrow checker 的内存策略**（见 §17 #1）：在 borrow checker 落地前（Phase 3），需一个临时内存模型（引用计数 / 作用域释放），否则无法安全运行。

---

## 10. 模块六：AI Analyzer（ail-analyzer）

### 职责（AILang 特色）
读取 `meaning` / `effects` / `requires` / `ensures`，结合类型/所有权分析结果，生成**程序知识图谱（Program Knowledge Graph）**，作为 `.ailmeta` 的数据源。

### 关键设计
- 推断每个函数的完整 `effects`（callee 并集 + 自身操作，含 `alloc`）；校验用户标注（推断集 ⊆ 标注集）。
- **dyn/FFI/泛型效果上界**（SPEC §22 #4）：interface 调用取 interface 声明 effects；`extern` 默认 `effects { extern }`；泛型方法随 bound。
- **`pure`**（SPEC §22 #9）：无任何效果（含 `alloc`）+ 不改入参；触发 COW 克隆/堆分配 → 非 pure。
- 从 `Result<T,E>` 提取 `errors`。
- 汇总每个类型/字段的 `meaning`、每个函数的 `identity`/契约/所有权签名。
- 调用图不动点（递归函数的效果传播）。

### 输出示例
源码：
```ail
fn login(cred: UserCredential) -> Result<Token, Error>
effects { database.read }
{ ... }
```
Analyzer 产出：
```
login
  INPUT:  UserCredential
  OUTPUT: Result<Token>
  FAIL:   Error
  EFFECT: database.read
  MEMORY: owned
```

---

## 11. 模块七：AILang IR（ail-ir）

### 为什么需要自己的 IR？
**LLVM 不懂 AI 语义。** 直接把 AST 降到 LLVM IR 会丢失 meaning/effects/契约等 AI 信息。因此设中间层：

```
AILang AST → AILang IR (HIR) → LLVM IR
```

AILang IR 是 `.ailmeta` 的唯一数据源，也是优化、多后端的承载层。未来可加 cranelift 等后端而不动前端。

### AILang IR 形态
保留函数签名 + 完整语义标注 + 类型化函数体，但比 raw AST 更规整（已解析、已去糖：`try/catch` → Result match、`borrow` → 借用标记）。

---

## 12. 模块八：LLVM Codegen（ail-codegen）

### 职责
AILang IR → LLVM IR → 优化 → 目标文件 → 链接原生二进制。

### 类型映射
| AILang | LLVM |
|--------|------|
| `int` | `i64` |
| `int32` / `byte` | `i32` / `i8` |
| `float` | `f64` |
| `bool` | `i8`（存储）/ `i1`（寄存器）|
| `string` | `{i64 len, i8* ptr}`（COW：共享缓冲 + **原子引用计数** → `Send`，修改时克隆；SPEC §22 #2） |
| `struct` | LLVM struct（无对象头）|
| `Array<T, const N>` | 内联 `[T × N]`（固定布局，const 泛型 N；SPEC §23 #3）|
| `Box<T>` | 堆指针 `T*`（owned，Move/非 Copy，确定性释放；SPEC §23 #4）|
| `type X = Y`（裸，无 meaning） | 透明别名：展开为 base（零成本，SPEC §29 #1）|
| `type X = Y` + `meaning`（semantic） | 名义新型：与 base 不同型、底层共享（零成本，SPEC §29 #1）；codegen 同 base 布局 |
| `Optional<T>` | `{i1 has, T}` |
| `Result<T,E>` | `{i1 is_err, T, E}` tagged union |
| `enum` | tag + data |
| `Future<T>` | 状态机 struct（`async fn` 编译产物）；**无 `Pin`**——borrow 禁跨 await（SPEC §10.3 / DESIGN §9.1）→ 不自引用（SPEC §24 #1） |
| `Channel<T>` | MPMC 队列句柄（有界 ring / 无界链）；`close` 标志 + `receive`/`try_receive`（SPEC §24 #2） |
| `TaskHandle` / `ActorHandle` | 调度器句柄（`Send`）；`cancel`/`cancelled`/`join`（SPEC §24 #4/#5/#8） |

### 所有权 / 效果 / 契约的 codegen
- **move / borrow / borrow_mut / copy**：move 无运行时代码；borrow 编译为指针（`alloca`+ptr，因不逃逸，生命周期 = 栈帧）；copy 按值复制。
- **effects / 契约 / pure**：纯编译期检查，不产生运行时代码（除非契约断言）；基本擦除。
- **泛型**：单态化（每个具体 `Result<X,Y>` 生成独立 LLVM 类型）。
- **数值边界（SPEC §27 #1/#2/#3）**：整数算术——release 直接用 LLVM `add`/`sub`/`mul`，**不**置 `nsw`/`nuw`（二补数回绕为规范行为）；debug 构建在每条算术后插溢出检测 → 命中 `panic(ArithmeticOverflow)`；整数 `sdiv`/`udiv`/`srem`/`urem` 前插除数==0 检测 → `panic(DivideByZero)`；浮点直接用 LLVM 浮点指令（IEEE 754，无检测、无 panic）。编译期常量除零 / 溢出由类型检查期报错。

### 流水线
```
AILang IR → 构建 LLVM Module → 优化 passes（内联 / 死代码删除 / 常量折叠 / SIMD）→ emit .o → 链接（lld / clang）→ native binary
```

### 陷阱
- `string` 需最小 runtime（len+ptr；GC 推后）。
- **panic 展开与 FFI 边界（SPEC §26 #3、SPEC §27 #4）**：panic 经 LLVM landing pad / personality function 栈展开 + Drop；展开至 **Task 边界**（→ Task 错误态）或 **`extern` 帧**（→ **转进程 abort**：C 无 Drop、不可跨语言栈展开，**非 UB**；C 的 `longjmp` 跨 FFI 进 AILang 帧 = UB 禁）；landing pad / abort-on-FFI 须由最小 runtime 提供。
- **`async/task/spawn/channel/select` 的 codegen 需状态机 + executor——**v0.2 仅标记，不做 codegen**（调用报「not yet supported」）。语义见 SPEC §24 决议记录 V：`async fn` → `Future<T>` 状态机（**无 `Pin`**，borrow 禁跨 await，见 SPEC §10.3 / DESIGN §9.1 / MEMORY §7）、`Channel` = MPMC + close、`spawn` 返回 `TaskHandle`（结构化作用域 join / `detached`）、`actor` = Task + `on` 邮箱循环、`cancel` = 协作式（await 点丢弃状态机 → 确定性释放）、阻塞经 `spawn_blocking`（独立线程池，不占 M:N worker，SPEC §24 #7）。
- 错误传播：v0.2 不引入 `?`，强制 `match`，codegen 直白。

---

## 13. AI Metadata 生成

AILang IR → `program.ailmeta`（JSON，`serde`）。确定性输出（字段排序固定）、带 `schema_version`、带源码定位、完整语义。Schema 见 WHITEPAPER §3。

**`.ailmeta` 的内容是编译器验证后的事实，不是注释**：声称 `pure` 即已校验无副作用；声称 `effects { database.read }` 即已校验不写库。AI 可信任它，如同信任类型签名。

---

## 14. 工具链与命令（ail / ailc）

`ailc` = 编译器二进制（rustc 式，单文件编译）。`ail` = 包管理器/构建工具（cargo 式）。

| 命令 | 作用 |
|------|------|
| `ail new <project>` | 生成 `project/`（`main.ail` + `ail.toml`） |
| `ail build` | 编译当前包 |
| `ail run` | 编译并运行 |
| `ail test` | 运行内置测试 |
| `ail publish` | 发布到包仓库 |

`ail.toml`：包元数据（名称、版本、依赖、`ail install <pkg>` 的来源）。

---

## 15. MVP 开发路线（Phase 1–4）

| Phase | 时间 | 产出 | 验证 |
|-------|------|------|------|
| **1** | 1–3 月 | Lexer、Parser、AST、基础类型、函数、struct、**最小 LLVM 输出** | `fn main() { print("hello") }` 编译为原生二进制并运行 |
| **2** | 3–6 月 | `Result`、`enum`、泛型、`interface`、**AI Metadata（`.ailmeta`）** | 端到端：源码 → 可信 AI 元数据 |
| **3** | 6–12 月 | **Ownership + Borrow Checker**、并发、标准库核心 | 所有权错误用例全部报出 |
| **4** | 1 年+ | 包管理、IDE 插件、Debugger、AI Agent 生态 | 生产级 |

> **关键 checkpoint**：Phase 1 证明全工具链打通（原生 hello world）；Phase 2 交付 `.ailmeta`，验证 AI-native 主张；Phase 3 补齐 Rust 级安全。

> **开放缺口**：Phase 1–2（borrow checker 尚未落地）的临时内存模型待定——候选：引用计数（类 Swift）/ 作用域释放 / 受限 unsafe。这是必须在 Phase 1 启动前敲定的决策（见 §17）。

---

## 16. 关键设计决议汇总（v0.2）

| # | 决议 | 选择 | 代价 |
|---|------|------|------|
| ① | 代码块 | **大括号 `{}`**（非缩进） | 放弃 Python 缩进语义 |
| ② | 分号 | **可选**（Go 风格，换行分隔） | parser 需换行/续行规则 |
| ③ | 扩展名/元数据 | **`.ail` / `.ailmeta`** | — |
| ④ | `type` 语义 | **默认透明别名；附 meaning -> 名义 semantic（SPEC §29 #1）** | 需字面量强制规则 |
| ⑤ | 所有权简化 | **borrow/borrow_mut 不逃逸** → 无 `'a` | struct 字段不能存引用 |
| ⑥ | 可变借用 | **v0.2 支持 `borrow_mut`**（独占） | 借用检查器更复杂 |
| ⑦ | 效果策略 | **推断为主，标注为约束** | 调用图不动点 |
| ⑧ | 泛型约束 | **`where { Bound<T> }`**（编译期 trait bound）+ `requires { }`（运行期断言）（SPEC §23 #2） | 不复用 Rust `<T: Trait>` 语法 |
| ⑨ | interface/trait | **interface=服务能力（dyn 派发，方法禁泛型），trait=类型能力（bound→单态化），`implements`**（SPEC §23 #1/#10） | dyn 与泛型方法不可兼得 |
| ⑩ | LLVM 引入 | **Phase 1 即最小 codegen**（全链路打通） | 早期 codegen 受限 |
| ⑪ | IR 分层 | **AILang IR (HIR) 独立于 LLVM IR** | 多一层 lowering |

---

## 17. 开放问题（工程实现层）

> **语言设计层面的开放问题已全部在 SPEC §20 决断**（见决议记录）。本节仅列**编译器实现层**的遗留工程决断——非语言语义，而是「如何实现」的工程取舍。

### 已决（语言层 → SPEC §20）

| 本节原 # | 主题 | 决议 |
|---|---|---|
| #2 | `try/catch/throw` 解糖 | SPEC §20 #4、SPEC §26 |
| #3 | `interface` / `trait` 边界 | SPEC §20 #1、SPEC §23 #1/#10 |
| #6 | `match` arm 形式 | SPEC §20 #9 |
| #7 | `meaning` / `constraint` 独立声明 | SPEC §20 #10、SPEC §29 #1/#5/#8/#9 |
| #8 | `string` / 集合内存模型（COW） | SPEC §20 #7 |
| #10 | 关键字精确集合（56） | SPEC §20 #5 |

### 遗留（工程实现层）

1. **Phase 1–2 临时内存模型**：borrow checker 落地前用 RC / 作用域释放 / 受限 unsafe？（**阻塞 Phase 1**；见 §15 开放缺口）。
2. **可选分号的换行/续行规则边界用例**：二元运算符行尾、`return` 表达式跨行等——需 Parser 实现时以测试集敲定。
3. **Constraint / 契约证明的实现深度**：语言层已分级锁定（SPEC §20 #3，v0.2 仅断言）；实现层待定 v0.3+ 是否集成 SMT。
4. **`async/task/spawn/channel/select` 的 codegen 与运行时**：模型已指定（SPEC §20 #8，M:N work-stealing）；状态机 + executor 实现后置 Phase 3–4。
