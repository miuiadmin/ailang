# AILang Language Specification v0.2.1（Consolidated）

> **状态：v0.2 语法冻结 + v0.2.1 收敛（自洽）**
> 本文件是 AILang 的**权威语言参考**：词法、关键字、文法、类型系统、所有权、错误模型、并发、AI 声明、Unsafe/FFI、形式文法。
> 概览见 [WHITEPAPER.md](./WHITEPAPER.md)；动因见 [PHILOSOPHY.md](./PHILOSOPHY.md)；编译器实现见 [DESIGN.md](./DESIGN.md)；
> 标准库见 [STDLIB.md](./STDLIB.md)；内存模型见 [MEMORY.md](./MEMORY.md)；并发模型见 [CONCURRENCY.md](./CONCURRENCY.md)。

### 相对 v0.1 的主要变化
- ✅ `{}` 代码块，**不使用强制缩进**
- ✅ **可选分号**（Go 风格）
- ✅ 扩展名 `.ail`，元数据 `.ailmeta`
- ✅ `meaning` / `constraint` 独立声明
- ✅ `effects` / `requires` / `ensures` 契约块置于签名与函数体之间
- ✅ 泛型 trait 约束用 `where { Bound<T> }`（编译期单态化）；`requires { }` 装运行期前置断言（§7.10 / §23 #2）
- ✅ `interface`（服务能力）与 `trait`（类型能力）区分，新增 `implements`

### v0.2.1 收敛说明（本次新增）
1. **逻辑运算符采用符号** `&&` `||` `!`；`and`/`or`/`not` 不作关键字。
2. **trait 实现统一为内联** `struct X implements T1, T2 { ...方法... }`；v0.2 不设独立 `impl T for X { }` 块。代价：v0.2 不支持为外部（非本模块定义）类型实现 trait（决议见 §20 #2）。
3. **关键字扩充至 56**：新增 内存(`unsafe` `extern`)、并发(`actor` `message` `parallel` `server` `route` `cancel`)、AI(`agent` `tool`)、测试(`test`)、接收者(`self`)。
4. **类型（非关键字）**：`Shared` `Mutex` `Channel` `Optional` `Result` `List` `Map` `Set` `Array` `Tuple` `raw_pointer` 为标准库类型；`layout(C)` 为属性标注。
5. 新增小节：命名规范(§1.7)、运算符与优先级(§3b)、AI 原生声明(§15)、Unsafe 与 FFI(§16)、并发(§13 升级)、形式文法(§18)。
6. **match 绑定语法锁定**：`Pattern: { 绑定 -> body }`（§8）；enum variant 支持带 payload（§7.6）。
7. **字符串 `+` 拼接锁定**：`string + string`，及 `string + Display` 自动字符串化（§3b，唯一隐式转换例外）。
8. **集合字面量锁定**：`[a, b, c]`（List/Array/Set，标注定类型，默认 List）+ `[k: v]` / `[:]`（Map）；统一 `[]`，不用 `{}`（§7.9）。
9. **`lock` 定性**：`std.sync` 内建控制构造（**非关键字**）；块语法 `lock(m) { }` 规避 v0.2 无闭包依赖（§13）。
10. **`spawn` 三形式锁定**：`spawn task(args)`（具名 Task）/ `spawn actor Name(args)`（Actor，返 `ActorHandle`，§24 #5）/ `spawn { block }`（匿名块）均合法（§13、§18 文法）。
11. **关键字计数校正**：精确 56 个（§2、§19）；原「约 55」为误计。
12. **§20 开放问题逐条决断**：本轮将 §20 全部 10 项作可逆决断——interface/trait 边界(#1)、外部类型 impl 分期(#2)、契约证明分级(#3)、try/catch 解糖(#4)、字面量不计入关键字(#5)、顶级声明边界(#6)、**string/集合 COW 值语义(#7)**、async 运行时(#8)、match arm 形式锁定(#9)、meaning 双轨(#10)；详见 §20 决议记录。

---

## 0. 语法层设计原则

> Python 的简洁表达 + Rust 的安全模型 + TypeScript 的类型表达 + C/Go 的代码结构 + AI 原生语义。

- **默认安全**：编译期确定类型，禁止隐式数值/类型转换 / 隐式空值 / 运行时类型错误（唯一例外：字符串拼接中的 `Display` 字符串化，见 §3b）。
- **默认不可变**。
- **类型不仅表示「是什么」，还表示「意义」**。
- **类型在公共边界强制声明**（函数参数/返回必须显式；局部可推断）。
- **AI 信息进入语言结构**（meaning / effects / requires / ensures / agent / tool）。

---

## 1. 词法约定

### 1.1 文件与扩展名
源文件 `.ail`（UTF-8）；元数据产物 `.ailmeta`。

### 1.2 代码块
统一 `{ }`，不使用强制缩进。

### 1.3 分号（可选）
Go 风格：换行分隔语句，分号可省。`let x = 1` 与 `let x = 1;` 等价。

### 1.4 隐式行连接
`() [] {}` 内换行忽略。

### 1.5 注释
`//` 行注释；`/* */` 块注释（可嵌套）；`///` 文档注释——附着到下一个声明，进 `.ailmeta`：**type 上为 description**（`meaning` 声明权威、冲突 `meaning` 胜），**function / param / field 为其 meaning / 描述来源**（§29 #8 双轨优先级）。

### 1.6 标识符与字面量
- 标识符：ASCII `[A-Za-z_][A-Za-z0-9_]*`（v0.2）。
- 整数字面量无类型后缀，按上下文推断/强制。

### 1.7 命名规范（官方风格）
| 实体 | 风格 | 示例 |
|------|------|------|
| 变量、函数 | camelCase | `userName`、`get_user` |
| 类型（struct/enum/trait/interface/actor） | PascalCase | `UserAccount`、`HttpClient` |
| 常量（`const`） | UPPER_CASE | `MAX_SIZE`、`VERSION` |

> 风格不强制（编译器不报错），但 `ail` 提供 `ail fmt` 检查。

---

## 2. 关键字（共 56）

| 类别 | 关键字 |
|------|--------|
| 模块 | `package` `import` `from` `as` |
| 变量 | `let` `var` `const` |
| 函数 | `fn` `return` `async` `await` `self` |
| 类型 | `type` `struct` `enum` `interface` `trait` `implements` |
| 控制流 | `if` `else` `match` `for` `while` `loop` `break` `continue` |
| 错误 | `error` `try` `catch` `throw` |
| 并发 | `task` `spawn` `channel` `select` `actor` `message` `parallel` `server` `route` `cancel` |
| AI 语义 | `meaning` `effects` `pure` `requires` `ensures` `agent` `tool` |
| 内存 | `move` `borrow` `borrow_mut` `copy` `unsafe` `extern` |
| 可见性 | `public` `private` |
| 测试 | `test` |

> 类型 `Optional` `Result` `List` `Map` `Set` `Array` `Tuple` `Channel` `Shared` `Mutex` `raw_pointer`、字面量/类型词 `true`/`false`/`void`/`never` 不计入关键字（属标准库/保留字面量·类型词）。精确计数见 §20 #5。

---

## 3. 基础语法规则

1. 代码块用 `{ }`（§1.2）
2. 分号可选（§1.3）
3. 类型写法：`name: type`（小写，`string` 非 `String`）
4. 函数返回后置：`fn xxx() -> Type { }`
5. 默认不可变：`let`；可变 `var`；编译期常量 `const`
6. 类型在公共边界强制声明：函数参数与返回必须显式；局部可推断

## 3b. 运算符与优先级（低 → 高）

| 级 | 运算符 | 说明 |
|----|--------|------|
| 0 | `=` `+=` `-=` | 赋值（语句级，右结合）|
| 1 | `||` | 逻辑或 |
| 2 | `&&` | 逻辑与 |
| 3 | `==` `!=` `<` `>` `<=` `>=` | 比较 |
| 4 | `+` `-` | 加减 |
| 5 | `*` `/` `%` | 乘除模 |
| 6 | `-`（负）`!` `borrow` `borrow_mut` `move` `copy` `await` `try` | 一元前缀（§24 #3 / §26 #1） |
| 7 | `()` `[]` `.` | 后缀：调用 / 下标 / 字段 |
| 8 | 字面量 / 标识符 / `( expr )` / 结构体构造 | 基本 |

> 逻辑运算符为符号 `&&` `||` `!`，与 C/Go/Rust 一致（v0.2.1 锁定）。
>
> **运算符重载 = trait 解糖**（§23 #8）：运算符解糖为 std.core 运算符 trait 方法调用（`a + b` → `Add.add(a, b)`、`a == b` → `Eq.eq(a, b)`、`a < b` → `Ord.lt(a, b)`）。泛型 bound 复用（`where { Ord<T> }`，§7.10）。仅 `&&`/`||`/`!` 为内置短路，不可重载。

**字符串拼接（v0.2.1 锁定）**：`+` 重载用于字符串——
- `string + string` → 拼接，产生新 `string`（`string` 为 COW：非 bitwise Copy，拼接必分配新缓冲；§10.2、§22 #7）。
- `string + X` / `X + string`，其中 `X` 实现 `Display` → `X` 经 `Display` 字符串化后拼接（覆盖基础类型、以其为 base 的语义类型、显式实现 `Display` 的类型）。
- 这是 AILang **唯一**允许的隐式转换（无损显示），其余转换必须显式。

```ail
let s = "id: " + userId        // UserId(int) 经 Display 字符串化
let t = "a" + "b"              // "ab"
```

---

## 4. 模块系统（§28 #3–#8）

### 4.1 package 声明 + Go 式包模型
`package <dotted-path>` 声明当前文件的**完整模块路径**：根包单段（`package app`），嵌套点分（`package std.http` / `package app.utils`）。**同一 `package` 声明可跨多文件共享命名空间**（Go 式——包内文件互不需 import）；**文件路径须与声明的模块路径一致**（编译期校验：`src/http/request.ail` 须声明 `package *.http.request`）。根包名 = `ail.toml.name`（§28 #10）。

```ail
package app                       // 根包
import std.http                   // 标准库：全路径，std. 前缀必需（§28 #4）
import std.database
from std.database import Client   // 仅可导入 std.database 的 public item（§28 #6）
import std.json as j              // 别名（§28 #5）
```

> **`std.core` 自动加载**（免 import，§21 #9）；其余 `std.*` 显式 import。第三方库 `import <pkg>.<module>`（依赖包名作根前缀，§28 #10）。

### 4.2 导入规则（§28 #5/#6）
- **全路径 + 禁通配**：导入路径必含根包；**禁 `import *`**（保 `.ailmeta`/AI 可追溯）。
- **冲突**：两导入同名 = **编译错误**（须 `as` 消歧）；**本地定义优先于导入**（遮蔽 + warn）。
- **`from M import name`**：仅可导入 M 的 **public 顶级 item**；导入 private = 编译错误。

### 4.3 再导出 / 循环导入（§28 #7/#8）
- **v0.2 不支持再导出**（无 `public import` / `pub use`）：`import a` 仅得 `a` 自身 public item；要 `b` 的 item 须显式 `import b`。
- **循环导入**：类型级（签名互引）OK；值级（fn 体跨模块调用）OK（惰性绑定）；**顶层初始化循环 = 编译错误**（`const` / 全局求值成环）。须两遍分析（先签名，再体）。

可见性见 §14（默认 `private`）。

---

## 5. 变量

```ail
let age: int = 18              // 不可变
var count: int = 0             // 可变
const VERSION: string = "1.0"  // 编译期常量
```

---

## 6. 函数

```ail
fn add(a: int, b: int) -> int {
    return a + b
}
```

契约块（`effects` / `requires` / `ensures`）置于签名与函数体之间；`pure` 为前缀：

```ail
pure fn add(a: int, b: int) -> int {
    return a + b
}

async fn fetch() -> Data {
    ...
}
let data = await fetch()
```

> `effects` 取值遵循统一 `<domain>.<verb>` 词表（权威枚举见 §11 效果系统；§25 #6）。

---

## 7. 类型系统

> AILang 的类型 = 数据结构 + 语义 + 约束 + 行为。

### 7.1 基础类型
`int`（默认 i64）`uint`（默认 u64）`float`（默认 f64）`bool` `string` `byte` `void`。显式宽度：`int8..int64`、`uint8..uint64`、`float32`/`float64`（§27 #2）。

> **数值语义（§27 #1/#2/#3）**：整数算术溢出 = **定义行为**（永不 UB）——release **二补数回绕**（规范），debug **溢出 → panic `ArithmeticOverflow`**（开发期检测层）；整数**除零 / 取模零 → panic `DivideByZero`**（确定性）；浮点遵循 **IEEE 754**（`x / 0.0`（`x≠0`）→ `±inf`、`0.0 / 0.0` → `NaN`、NaN 经运算传播、`NaN == NaN` 为 `false`，**浮点运算永不 panic**）。显式运算符族（`wrapping_*`/`checked_*`/`saturating_*`）/ 安全除法 `checked_div` / NaN 检测（`is_nan`/`is_inf`/`is_finite`）见 `std.math`（STDLIB §6.1）。

> **`never`（⊥，bottom）**：空类型，内建 lang item（§26 #4）——无值，表示「不返回」（`throw`/`return`/`loop` 表达式类型为 `never`）。统一规则 `never | T = T`：穷尽检查中 `throw`/`return` 的 match arm 不占变体。`never` ≠ `void`（`void`=unit 有值；`never` 无值）。可作返回类型 `fn fail() -> never`。`never` 为保留类型词，不计 56（同 `void`/`true`/`false`，§19）。

### 7.2 类型推断
局部可推断；**函数参数与返回必须显式**（公共边界）。泛型 `T` 在参数中可由实参推断；**return-only 泛型**（`fn empty<T>() -> List<T>`）须调用点显式 `empty<int>()`（§23 #7）。

### 7.3 Semantic Type（名义新类型）与 Type Alias（透明别名）

`type X = Y` 按 **§29 #1** 分两类：

- **无 `meaning` / `constraint` → 透明别名（`kind: alias`）**：编译期展开，`X` 与 `Y` 同型、可互转（如 `type URL = string`、`type Row = ...`，§29 #10）。
- **附 `meaning` → 名义 semantic（`kind: semantic`）**：与 base 不同型，底层共享 → 零成本：
```ail
type UserId = int
meaning UserId:
    "unique identifier of a user"
```
整数字面量可强制到整型语义类型（`let u: UserId = 100`，§21 #1）；变量间不隐式转换（nominal 不变，§23 #9）。

> **运算 / 标记继承（§29 #7，marker vs operational 二分）**：semantic 新类型对 base 的 trait 继承按两类拆分——
> - **标记 trait（`Copy` / `Send` / `Sync`）自动继承 base**：`type UserId = int`（附 meaning）保持 `Copy + Send + Sync`（与 base 同的位级与线程安全类别）。
> - **运算 trait（`Add` / `Sub` / `Mul` / `Eq` / `Ord` / ...）不自动继承**（类 Rust newtype），须显式实现；`==` 须显式实现 `Eq`。
> - **唯一例外：`Display`**（字符串化自动经 base，§3b）+ 字面量强制（`let u: UserId = 100`，§7.3 / §21 #1）。
>
> **实现机制（v0.2.1 gap）**：v0.2.1 仅有内联 `struct X implements Trait { }`（§7.5），尚无给语义新类型（`type UserId = int`）实现运算 trait 的语法——运算 trait 继承为 **v0.3 计划**（独立 `impl Trait for Type { }` 块 + orphan 规则下放开外部类型，§20 #2 / §29 #7）。
>
> **meaning 传播（§29 #3）**：值为带 meaning 的 semantic → 在 `.ailmeta` 各处（input / output / field）携带此 meaning；函数 output.meaning = 返回类型 meaning（函数级不可另写，描述用 `///`→identity.description）；bare 基础类型 / alias = null。

### 7.4 Constraint Type（semantic + 约束；§29 #6）

Constraint Type = **Semantic Type + `constraint`**（特化，非独立 kind；`.ailmeta` 仍 `kind: semantic` + `constraint` 字段）：
```ail
type Age = int
constraint Age:
{
    value >= 0
    value <= 150
}
```
- **求值分级**：字面量编译期检查（`let a: Age = 30` 折叠后判定；`let a: Age = -5` → 编译错）；运行期构造插断言。完整 refinement 证明不在 v0.2（§20 #3）。
- **构造算符 `T(expr)`（§29 #2）**：运行期值经 `let a: Age = Age(n)` 构造——运行期断言谓词，违约 panic `ConstraintViolation`（§29 #9）；字面量走编译期分支。nominal 不变（§23 #9）使 `let a: Age = n`（n 为 int 变量）类型错，**须 `Age(n)`**。
- **谓词语言（§29 #4）**：`value` 魔法只读绑定 + 纯布尔表达式（可 `value.field` / `value.method()`、跨字段须约束 struct 引用字段名、可调 `pure fn`）；禁 `old()` / 副作用 / IO（`old()` 属 `ensures`）；谓词须 pure。

### 7.5 Struct（含 trait 实现 + 方法）
```ail
trait Display {
    fn display(borrow self) -> string          // 只读接收者
}

struct User implements Display {
    id: UserId
    username: string

    fn display(borrow self) -> string {        // 满足 Display（默认 = borrow self，可省）
        return self.username
    }

    fn rename(borrow_mut self, name: string) { // 可变接收者：原地改 self
        self.username = name
    }
}
```

**接收者模式**（§22 #1，复用形参关键字）：`borrow self`（只读，**默认**，可省）/ `borrow_mut self`（可变，原地改）/ `self`（消费）。可变方法（如集合 `insert`/`append`）须声明 `borrow_mut self`；`release` 须声明 `self`（§10.5）。**trait 默认方法**（§23 #6）：trait 方法带 `{ body }` = 默认实现，struct 同名方法覆写，`implements Trait { }` 可留空继承默认。栈布局、无对象头、无追踪 GC。**trait 实现内联于 struct 定义**（v0.2.1 锁定，§20 #2）。

### 7.6 Enum
```ail
enum Status {              // 无 payload variant（标签）
    Active
    Disabled
    Deleted
}

enum Expr {                // 带 payload variant（代数和，类 Rust）
    Num(int)
    Add(Box<Expr>, Box<Expr>)      // 递归须 Box<T> 堆间接（§23 #4）
    Leaf
}
```
内存：tag + data（类 Rust）。variant 可携带 payload；`match` 解构并绑定（见 §8）。**递归类型须 `Box<T>` 间接**（owned 堆，Move/非 Copy；§23 #4），否则无限大小。

### 7.7 Optional\<T>（禁止 null）
```ail
fn find_user(id: UserId) -> Optional<User> { ... }   // Some / None
```

### 7.8 Result\<T, E>（错误安全核心）
```ail
fn read_file(path: string) -> Result<string, FileError> { ... }
```

> **lang item**（§23 #5）：`Optional`/`Result`（及 `Ok`/`Err`/`Some`/`None`、`List`/`Map`/`string` 等）为 **`#[lang]`** 标注的 std.core 类型——既是库类型，又被编译器特判（errors 派生 §9、`try/catch` 解糖 §20 #4、特权 `match` arm）。

### 7.9 复合类型
`List<T>` `Array<T, const N>` `Map<K, V>` `Set<T>` `Tuple`。均为标准库类型，非关键字。`Array<T, const N: int>` 的 `N` 为 **const 泛型**（编译期整数，固定内联布局，§23 #3、§16.3）。

**字面量（v0.2.1 锁定）**——统一用方括号 `[]`，**不用 `{}`**（避免与块 `{}`、struct 构造 `User { }` 冲突）：
- 元素集合（`List` / `Array` / `Set`）：`[a, b, c]`；类型由上下文标注决定，无标注默认 `List<T>`；空 `[]`。
- `Map`：`[k: v, k2: v2]`（冒号为键值分隔）；空 `[:]`。
- 判别：方括号内首对含 `:` → Map；否则元素集合。

```ail
let users: List<User> = [u1, u2]
let nums: Array<int, 3> = [1, 2, 3]
let ids: Set<UserId> = [1, 2, 3]
let cache: Map<string, User> = ["a": ua, "b": ub]
let empty_map: Map<int, int> = [:]
let tuple: (int, string) = (1, "x")       // Tuple 用圆括号
```

### 7.10 泛型（约束用 `where { }`）
```ail
fn sort<T>(list: List<T>) -> List<T>
where {                                     // trait bound（编译期，单态化；§23 #2）
    Ord<T>
}
{
    ...
}
```
泛型构造器**不变**（`List<UserId>` ≠ `List<int>`，§23 #9）。运算符 bound 复用 std 运算符 trait（§3b、§23 #8）。

### 7.11 Interface 与 Trait
- **`interface` = 服务能力**（外部 API 抽象，**dyn 派发**）。方法**必须携带 `effects`**（§22 #4）；**方法不得泛型**（dyn vtable 无法单态化，§23 #1）：
  ```ail
  interface Database {
      fn query(borrow self, sql: string) -> Result<List<Row>, Error>   // 非泛型；Row = 动态行
      effects { database.read }
  }
  // typed 解码由单态化自由函数完成：fn decode<T>(rows: List<Row>) -> Result<List<T>, Error>
  ```
- **`trait` = 类型能力**（可带默认实现，作泛型约束）：
  ```ail
  trait Serialize {
      fn serialize(borrow self) -> string      // 接收者模式同 §7.5
  }
  ```
- 类型用 **`implements`**（内联）获得 trait 能力（§7.5）。

> **interface 值的所有权**（§22 #6）：interface 类型值（`Database`/`Model` 等）为**不可 Copy 的 Move 句柄**（vtable + 指针），经 `borrow` 借用、`Shared<T>`/连接池共享。
>
> **trait vs interface 类型层二分**（§23 #10）：`trait` 作泛型 bound → **编译期单态化**；`interface` → **运行期 dyn 派发**。此二分即 interface 禁泛型方法（#1）的根因。
>
> v0.2.1：`implements` 是唯一的 trait 实现语法；无独立 `impl T for X` 块（§20 #2）。`interface` 与 `trait` 是否都需要见 §20 #1。
>
> **orphan 规则（§28 #9）**：`Trait` 或 `Type` 之一须定义于当前 package——禁止在 `app` 里为 `std.http.Request` impl 自定义 trait（除非 trait 或类型之一属本 package）；`public` 类型 impl 本 package `public` trait 可跨模块可见（类 Rust coherence）。

---

## 8. 控制流

```ail
if age >= 18 { allow() } else { deny() }

match status {                  // 无 payload：直接块体
    Active: { run() }
    Disabled: { stop() }
}                               // 必须穷尽所有 variant

match find_user(1) {            // 带 payload：用  绑定名 ->  取关联值
    Some: { u -> print(u.username) }
    None: { print("none") }
}

match expr {                    // 多 payload：逗号分隔绑定
    Expr.Num: { n -> print(n) }
    Expr.Add: { l, r -> print(l, r) }
    Expr.Leaf: { print("leaf") }
}

match opt {                     // _: 通配兜底
    Some: { x -> use(x) }
    _: { print("else") }
}

for user in users { print(user) }
while running { update() }
loop {
    work()
    if done { break }
}
```

**match arm（v0.2.1 锁定）**：每个 arm 形如 `Pattern: { 绑定 -> body }`（带 payload）或 `Pattern: { body }`（无 payload / 忽略数据）。
- `Pattern`：variant 名（裸，如 `Active`/`Some`/`Ok`；多 enum 消歧用 `Enum.Variant`）。
- 带 payload 的 variant（`Some`/`Ok`/声明带数据的 enum variant）用 `绑定名 -> ` 取关联值；多值 `a, b -> `；忽略数据则省略。
- `Result<T, E>`：`Ok` 绑定 T，错误分支为 `E` 的各 variant（无 `Err` 包装）。
- `_:` 通配兜底；编译期强制穷尽。

---

## 9. 错误模型

核心 `Result<T, E>`；`error` 定义错误枚举。

```ail
error UserError {
    NotFound
    Forbidden
}

fn login() -> Result<Token, UserError> { ... }
```

**`errors[]` 单一真源（§26 #10）**：函数 `errors` ≡ 签名 `E`（`error` 枚举）的变体集，进 `.ailmeta`；`throw`/`try` **永不越界、永不增补**。非 `Result` 返回 → `errors:[]`；`E` 为 `std.core.Error`（动态边界 opt-in 通用型，§21 #6，非命名 `error` 枚举）→ `errors[]:["Error"]`（类型名占位，变体不展开）。

**变体值构造（§26 #2）**：`Ok(x)` / `Err` 变体构造**复用 call 文法**（`Variant(args)`），由名字解析消歧（变体 vs 函数）；裸变体 = `ident`。**成功返回须显式 `Ok(x)`，无 auto-wrap**（呼应 §3b 禁隐式转换）；`Result<void, E>` 成功构造 = `return Ok(void)`（无 `()` 字面）。

**`try`/`catch`/`throw` = `Result` + `match` 的语法糖**（精确解糖 §20 #4；§26 #1/#5/#6）：

```ail
fn load() -> Result<Data, UserError> {
    let raw = try read()                 // try expr：前缀一元（同 await）；unwrap-or-propagate，要求同错误类型
    let v = match parse(raw) {           // 跨类型转换须 match + throw（无隐式 From）
        Ok: { d -> d }
        ParseError.Bad: { throw UserError.Forbidden }   // Error → 命名错误（§21 #6）
    }
    return Ok(v)                         // 显式 Ok，无 auto-wrap
}
```

- `throw V`：**语句级**控制流，类型 `never`（§7.1、§26 #4），可与任意类型统一（match arm/分支），不可作子表达式（§22 #10）。**合法性（§26 #6）**：仅当所在函数/处理子返回 `Result<T,E>` 且 `V ∈ E`；`void`/`task`/actor `on`/server handler 体**禁 throw**（错误经各自通道，§13、§26 #8）。
- `try expr`：**前缀一元**（§18 unary，与 `await` 一致）；`expr: Result<T,E1>` 在外层 `Result<U,E2>` 且 `E1≠E2` 时**类型错误**——**无隐式 `From`**（§3b），跨类型转换唯一手段 = `match { 错误分支 → throw 新变体 }`（边界 `Error → 命名错误`，§21 #6）。
- `try { body } catch { arms }`：局部捕获（§18 `try_catch`）；未匹配且无 `_` → 重新传播。
- **`error` 变体可带 payload**（§26 #7，与 enum 同形 `Variant(field: T)`）：`throw NotFound(404)` / match arm 绑定同 §8；`.ailmeta` `errors[]` 每项可选 `payload:[{name,type}]`（WHITEPAPER §3）。惯法推荐裸变体。
- **`pure` 与 throw（§26 #9）**：`pure fn` 可返回 `Result` 并 `throw`/`try`——throw 是控制流（早返 Err），**非 effect、不改入参**；`.ailmeta` `is_pure` 仅看 effect 集与入参突变，与 errors 无关（§11）。

**不可恢复错误：panic（§26 #3）**：`unwrap`/`assert` 失败或显式 `std.core.panic(msg)` 触发 **panic = 确定性栈展开 + Drop/release**（与确定性释放一致），展开至最近的 **Task 边界或 `extern` 帧**——Task 内 panic 终止当前 Task（→ TaskHandle 错误态 / actor 死信），不传染进程；**触达 `extern` 帧 → 转进程 abort**（FFI 边界为「不传染进程」唯一例外，§27 #4）；v0.2 无 `catch_unwind`（仅 Task 边界可观测）。已知 panic 原因：`unwrap`/`assert`、整数溢出（debug，`ArithmeticOverflow`，§27 #1）、整数除零 / 取模零（`DivideByZero`，§27 #3）、越界访问（`IndexOutOfBounds`）、约束违约（`ConstraintViolation`，§7.4 / §29 #9）。`panic` 为 std 函数，非关键字。

> **`std.core.Error`**：动态边界（DB 驱动 / FFI / 泛型 Task）的 opt-in 通用错误类型；惯用代码用命名 `error` 枚举以保 `.ailmeta` 精度，边界处显式 `Error → 命名错误` 映射（§21 #6）。

---

## 10. 所有权与借用（Single Owner Model）

> 完整内存模型见 [MEMORY.md](./MEMORY.md)。

### 10.1 默认 move
```ail
let file = File.open("a.txt")
process(file)       // move
print(file)         // ❌ AIL2001 Use after move
```
显式：`borrow`（只读）/ `borrow_mut`（可写独占）/ `move` / `copy`（Copy 与 COW 类型；在 COW 上强制克隆——对齐 MEMORY §6 / SPEC §22 #3）。

> 调用点显式标注**可选**（§21 #13）：省略时按**被调形参模式**推断（形参 `borrow T` → 实参自动借用、不 move）；显式形式用于澄清或消歧。

### 10.2 值语义三分类（v0.2.1 锁定）

| 类别 | 类型 | 行为 |
|------|------|------|
| **位复制 Copy** | `int` `uint` `float` `bool` `byte` + 全 Copy/COW 字段 struct | 赋值即位复制，原值仍可用 |
| **COW 值类型** | `string` `List` `Map` `Set` | 值语义：赋值=共享缓冲（**原子引用计数**），修改=克隆（写时复制）；**为 `Send`**（§22 #2）；行为同 Copy，无 `&str`/`String` 双类型 |
| **Move（独占资源）** | `File` `Socket` `Connection` 等 | 默认 move，使用须显式 `borrow`/`borrow_mut`（Move-only 不可 `copy`） |

> `string`/集合采用 **COW 值语义**（类 Swift 数据模型）：Python 式廉价传递 + 值语义安全，**无追踪 GC**（COW/RC 引用计数仍存，确定性释放；§22 #2、§20 #7）。资源型 move-only。COW 字段计为 Copy-compatible → 含 COW 字段的全 Copy-like struct 为 Copy（§22 #7）。
>
> **`Array<T, N>` 与 `Tuple` 值语义（§24 #6）**：值类型聚合——`Copy` 当且仅当所有元素 / 字段为 `Copy`（含 COW 计 Copy-compatible），否则整聚合 `Move`（任一 Move/`Resource` 元素 → 整聚合 Move）。

### 10.3 生命周期：编译器推导，不暴露 `'a`
`borrow`/`borrow_mut` **不逃逸**（禁存字段、禁返回、禁跨 `await`）→ 生命周期恒等于当前语句 → 无 `'a`。详见 MEMORY §7。

> **`borrow_mut` 与 COW**（§22 #3）：`borrow_mut` 要求 COW 值**唯一所有权（rc == 1）**——共享（rc>1）值须先 `copy`/克隆；`borrow_mut` 下突变**原地**进行（独占 → 无需克隆），即 opt-out COW。

### 10.4 线程安全
跨线程移动/共享自动分析（`Send`/`Sync` 为编译器自动派生的 marker trait，§24 #6）；COW 类型 `Send`（原子 rc，§22 #2）。**组合规则（§24 #6）**：聚合类型 `Send` 当且仅当所有字段 `Send`；`Sync` 当且仅当所有字段 `Sync`；所有 `Copy` → `Send + Sync`；Move 资源（`File`/`Socket`）→ `Send`、非 `Sync`；`Mutex<T>` → `Send + Sync`（`T: Send`）；`Shared<T>` → `Send + Sync`（`T: Sync`）；`TaskHandle`/`ActorHandle`/`Channel<T>` → `Send`（`T: Send`）。`spawn` / `Channel.send` / actor 投递边界静态查 `Send`。详见 MEMORY §11、CONCURRENCY。

### 10.5 资源确定性释放
所有者离开作用域时自动 `release`（消费 owner，类 Rust `Drop`，§22 #8、MEMORY §9-10）。

> **Drop 语义精确化（§27 #5）**：① 字段 **Drop 顺序 = 声明顺序**（自上而下，确定性，与 Rust 一致）；② 安全代码**结构上无循环引用**（单一所有者 + COW rc 为缓冲共享、非对象图共享）→ **v0.2 不引入 `Weak`/`Rc`/`Arc`**，需共享对象用 Actor / `Mutex<T>` / `Shared<T>`（§13）；③ panic 展开时 Drop 同序执行（§26 #3）；④ 自定义释放经 `Resource.release(self)`（消费 owner）。

---

## 11. 效果系统

```ail
fn save_user(user: User) -> Result<void, Error>
effects {
    database.write
    network
}
{ ... }
```
点分粒度（`database.read` `filesystem.write` `network` `alloc`）。**推断为主，标注为约束**（推断集 ⊆ 标注集）。`pure` = **无任何效果（含 `alloc`）且不修改入参**（§22 #9）。`.ailmeta` 输出推断后的完整集。

> **`is_pure` 与 errors 无关（§26 #9）**：`pure fn` 可返回 `Result` 并 `throw`/`try`——throw 是控制流（早返 Err），非 effect、不改入参；`is_pure` 仅看 effect 集 + 入参突变，与 `errors[]` 正交。

**Effect 词表（权威，`<domain>.<verb>`，§25 #6）**：

| effect | 语义 |
|--------|------|
| `database.read` / `database.write` | 数据库读 / 写 |
| `network` | 网络（别名 `network.read` / `network.write` 细化读写） |
| `io.read` / `io.write` | 标准流 / 控制台读写（`print` → `io.write`） |
| `io.blocking` | 同步阻塞（`fs.read` 同步版；`async fn` 禁，§24 #7） |
| `filesystem.read` / `filesystem.write` | 文件系统读 / 写 |
| `alloc` | 堆分配 |
| `extern` | FFI（`extern` 块默认上界，§22 #4） |
| `unsafe` | `unsafe` 块（逃生舱，§16） |

`pure` = **空 effect 集** + 不改入参。新效果须遵循 `<domain>.<verb>` 命名；跨文档统一用本表枚举值。

> **健全边界**（§22 #4）：① **interface 方法必须声明 `effects`**（§7.11），dyn 调用效果取声明上界；② **`extern`（FFI）默认 `effects { extern }`**（保守上界，可收窄）；③ 泛型方法效果随 trait bound 传播。如此「推断集 ⊆ 标注集」对 dyn/FFI/泛型可强制。

---

## 12. 契约（前置 / 后置条件）

```ail
fn withdraw(account: borrow_mut Account, amount: int) -> Result<void, Error>
requires {
    amount > 0
    account.balance >= amount
}
ensures {
    account.balance == old(account.balance) - amount
}
effects {
    database.write
}
{ ... }
```
v0.2：运行期断言。自动证明不在范围（§20 #3）。`account` 取 `borrow_mut`（§22 #5）使 `ensures` 对 caller 可观测；`old(e)` 为**调用前快照**；函数先突变内存 `account.balance` 再 `database.write` 持久化，内存后态与持久化一致。

> **`requires`/`ensures` 仅装运行期断言**；泛型 trait bound 用 `where { }`（§23 #2）。

---

## 13. 并发（Task-Based）

> 完整模型见 [CONCURRENCY.md](./CONCURRENCY.md)。关键字：`task` `spawn` `async` `await` `channel` `select` `actor` `message` `parallel` `server` `route` `cancel`。

```ail
task download() { fetch() }
let h = spawn download()              // h: TaskHandle（§24 #4）；编译器检查 Send + 生命周期

let ch = Channel<int>()
ch.send(10)
let v = ch.receive()                    // Optional<int>（阻塞至有值 Some / 关闭耗尽 None，§24 #2）

select {                                      // 多路等待（类 Go select，§24 #10）
    ch1.receive() { v -> handle1(v) }         // v: Optional<T>（Some=值 / None=关闭）；绑定名 -> body（同 match §8）
    ch2.receive() { handle2() }               // 无需值则省略绑定
    timeout(100ms) { give_up() }              // 超时分支（上下文关键字）
}

actor UserService {                   // 独立状态 + 消息入口，AI 友好
    state:
        users: Map<UserId, User>
    message AddUser { user: User }
    on AddUser { msg ->               // 消息处理（§24 #5），self.state 单 Task 串行
        self.state.users.insert(msg.user.id, msg.user)
    }
}

parallel for item in data {           // CPU 并行，自动分片/合并（§24 #9：表达式，返回 List<R>）
    process(item)
}
let squares: List<int> = parallel for x in data { x * x }   // body 须返回 R
let sum = parallel reduce((a, b) -> a + b, data) { x -> x } // op 须结合律

cancel h                            // h: TaskHandle；结构化取消，向子 Task 传播（§24 #4/#8）
```

- **默认禁止跨 Task 共享可变**（`var` 跨 spawn → 拒绝）。
- 安全共享：`Shared<T>`（只读）/ `Mutex<T>` + `lock(data) { }`（MEMORY §12）；`lock` 为 `std.sync` 内建控制构造（**非关键字**），块语法规避 v0.2 无闭包依赖，待闭包落地后可改方法式 `m.with_lock(...)`。
- **`spawn` 作用域语义（§24 #4）**：三形式 `spawn task(args)`（具名 Task）/ `spawn actor Name(args)`（Actor，返 `ActorHandle`，§24 #5）/ `spawn { ... }`（匿名块）；具名/匿名返 `TaskHandle`。**默认 scoped**——task 附属 spawn 所在作用域，作用域末**隐式 join**（父等待子完成）；**显式分离用 `spawn detached`**（`detached` 为上下文关键字，不入 56）。**非** fire-and-forget（仍返回 `TaskHandle`、`cancel` 可达；§24 #4）。
- **`task` 与 `async fn` 区分**（§24 #1）：`task` = void body + CSP 通道（不可 async），经 Channel 投递消息 / `cancel` 通信、**不强制 `await`**；`async fn f() -> T` 的 `T` 即 `await` 产出值（调用得 `Future<T>` lang item，`await f()` 得 `T`）。**「须返回 Result」仅约束用户代码显式 `await` 的 async fn**；server route 等 runtime 派发 handler 豁免（返回声明类型即可）。**无 `Pin`**：borrow 禁跨 await（SPEC §10.3 / DESIGN §9.1 / MEMORY §7）→ 状态机不自引用。
- **并发错误传递**（§26 #8 统一表）：`throw` 仅合法于返回 `Result<T,E>` 的函数体（§9、§26 #6）——`async fn -> Result<T,E>` 可 `throw`/`try`（错误随 `await` 传）；`task`（void 体）/ actor `on` handler / server handler **禁 throw**，分别经 Channel 投递错误消息或 `cancel`、`reply(Err...)`、runtime 派发传递。完整表见 CONCURRENCY §13。
- **Channel**（§24 #2）：**MPMC**；无界 `Channel<T>()` / 有界 `Channel<T>(cap)`（`cap=0` rendezvous）；`ch.close()`；`receive() -> Optional<T>`（阻塞至有值得 `Some(v)`；关闭且耗尽得 `None`）、`try_receive() -> Optional<T>`（非阻塞；`None` = 空或关闭耗尽）；两者仅「open 且空」时不同（前者阻塞、后者立即 `None`）；select 的 receive 分支在关闭时立即就绪并交付 `None`（§24 #10）。
- **控制构造块捕获**（v0.2 无一等闭包）：`spawn { }` / `parallel for` / `lock(m) { }` / `server route` 的块体可捕获外层**不可变 / `Send`** 变量（**不含 `borrow`**——借用不逃逸，§10.3）；`Channel` 为共享 `Send` 句柄，跨 spawn 克隆（§21 #7、§24 #6）。
- **`await` 为一元前缀**（§24 #3）：`await f()` / `await fut.field`。
- **Actor**（§24 #5）：`actor { state; message; on Msg { msg -> body } }`——`on` 为消息处理子句；`spawn actor Name(...)` 返回 `ActorHandle`；`h.send(Msg{...})` / `h ! Msg{...}` 投递；状态经 `self.state` 串行访问（无需锁）。**`reply`**（HIGH#5）：`on` 子句内的**上下文关键字**——`reply(expr)` 为该 handler 的值/错误出口（`reply` 值确定 message 的响应类型，可为 `Result<T, E>`）；actor `on` handler **禁 `throw`**，错误经 `reply(Err...)` 传递（§26 #8）。
- **select 扩展**（§24 #10）：支持 `receive`/`send`/`await` 分支 + `default { }`（非阻塞）+ `timeout(d) { }`；多分支就绪**随机公平**择一；关闭 channel 的 receive 立即可就绪（交付 `None`）。
- **parallel for / reduce**（§24 #9）：`parallel for x in xs { f(x) }` 为**表达式**（返回 `List<R>`，body 须返回 `R`）；`parallel reduce(op, xs) { ... }` 归约（`op` 须结合律）；body 须 `pure` 或仅 `io.read`、无共享可变。
- v0.2：定义语义；executor / codegen 后置（DESIGN §12）；并发运行时完整语义见 **§24 决议记录 V**。

---

## 14. 可见性（§28 #2）

```ail
public fn api() { ... }              // 跨 package 可见
private fn internal() { ... }        // 仅本 package（默认）

public struct User {
    public id: UserId                // 字段须显式 public 才跨 package 可见
    cache: string                    // 默认 private
}
```

`public` / `private` 修饰**所有顶级 item + struct/enum 字段 + 方法**；**默认 `private`**（含字段、方法）。trait/interface 方法**不单独修饰**（可见性随类型，类 Rust）。跨 package 仅可访问 `public`；`from M import name` 仅可导入 M 的 public item（§28 #6）。

---

## 15. AI 原生声明

> AILang 核心差异化。详见 [STDLIB.md](./STDLIB.md) §13。

### 15.1 agent（Agent 为语言一级公民）
```ail
agent ResearchAgent {
    goal: "research AI news"
    tools: [ web.search, database.query ]
}
```
编译为元数据（`type/goal/tools`），供 AI Agent 运行时消费。

### 15.2 tool（自动生成下游 Schema）
```ail
tool Search {
    fn query(text: string) -> List<SearchResult>
}
```
`tool` 自动产出 OpenAI Tool Schema / JSON Schema / API 文档。

> `agent` / `tool` 是新的顶级声明（v0.2.1 入关键字）。与 `struct`/`interface`/`actor` 的精确边界见 §20 #6。

---

## 16. Unsafe 与 FFI

> 详见 [MEMORY.md](./MEMORY.md) §13-16。AILang 默认安全；以下为系统级逃生舱。

**`unsafe { }` 内必需操作的精确清单（§27 #4）**：① 裸指针（`raw_pointer<T>`）解引用 / 读 / 写；② 调用 `extern`（FFI）函数；③ 调用其他标 `unsafe` 的函数；④（预留）全局可变——v0.2 **不引入 `static mut`**（全局可变须走 `Mutex<T>`/`Shared<T>`，与「禁跨 Task 共享可变」一致，§13），亦**无 union**（用 `enum` tagged union + `layout(C)` struct 对接 FFI）。清单外操作（约束断言、越界检查、整数溢出）**非 unsafe**——违约 = panic（确定性、安全）。

### 16.1 Unsafe 块
```ail
unsafe {
    let p: raw_pointer<int> = ...     // 裸指针仅在 unsafe 内
}
```

### 16.2 FFI
```ail
extern c {
    fn printf(text: string)
}
```

> **panic 跨 FFI 边界（§27 #4）**：panic 展开至 Task 边界或 `extern` 帧；触达 `extern` 帧（AILang 被 C 调入后 panic）→ **转进程 abort**（确定性终止，**非 UB**；C 无 Drop、不可跨语言栈展开）。正常 Task 内 panic 止于 Task 边界、不传染进程（FFI 边界为唯一例外）。C 的 `longjmp` 跨 FFI 进 AILang 帧 = UB（禁）。

### 16.3 内存布局
```ail
layout(C) struct Packet {             // C 兼容布局（FFI / 二进制协议）
    id: int
    data: Array<byte, 256>
}
```

`unsafe` 块在 `ail publish` 时被审计（STDLIB §15）。

---

## 17. 完整示例

```ail
package main

type UserId = int
meaning UserId:
    "unique identifier of a user"

struct User {
    id: UserId
    name: string
}

error UserError {
    NotFound
}

fn get_user(id: UserId) -> Result<User, UserError>
effects {
    database.read
}
{
    // 简化示例；完整签名（含 db: borrow Database 参数）与实现见 EXAMPLES §19
    ...
}

fn main() -> void {
    let userId: UserId = 100
    let user = get_user(userId)
    print(user)
}
```

编译产出：原生二进制 + `program.ailmeta`。AI 直接读到：输入 `UserId` / 输出 `User` / 失败 `NotFound` / 效果 `database.read`——**无需猜测**。

---

## 18. 形式文法（EBNF，v0.2.1）

```
module      := "package" module_path import* item*       // 包名=完整点分路径（§28 #1/#3）
import      := "import" module_path ("as" ident)?        // 全路径（§28 #4）；禁 import *
            |  "from" module_path "import" ident ("as" ident)?   // 仅可导入 public item（§28 #6）
module_path := ident ("." ident)*                        // 如 std.http / app.utils（§28 #1）
visibility  := "public" | "private"                      // §28 #1/#2
item        := visibility? ( fn | struct | enum | interface | trait
                           | error | type_alias | meaning | constraint
                           | agent | tool | actor | task | server | test | extern )

fn          := ("pure")? ("async")? "fn" ident generic? "(" params? ")"
              "->" type where_clause? contract* block
fn_sig      := ("pure")? ("async")? "fn" ident generic? "(" params? ")" ("->" type)?
              where_clause? contract*          // 签名（无 body）；interface/trait 用
where_clause := "where" "{" bound* "}"         // 编译期 trait bound（§23 #2）；where 为上下文关键字（同 lock 先例，不入 56，详见 §19）
bound       := ident "<" type ">"              // 如 Ord<T>、Eq<T>（§23 #8 运算符 trait）
contract    := "effects" "{" effect* "}"
            |  "requires" "{" expr* "}"        // 运行期前置断言
            |  "ensures" "{" expr* "}"         // 运行期后置断言
params      := param ("," param)*
param       := doc? ident ":" type
layout      := "layout" "(" ident ")"                  // 如 layout(C)
struct      := layout? "struct" ident generic? ("implements" type_list)? "{" (field | fn)* "}"
field       := doc? ident ":" type
enum        := "enum" ident "{" variant* "}"
interface   := "interface" ident generic? "{" fn_sig* "}"
trait       := "trait" ident generic? "{" (fn_sig | fn)* "}"
error       := "error" ident "{" variant* "}"
type_alias  := "type" ident "=" type              // §29 #1：无 meaning=alias(透明)；附 meaning=semantic(名义)
meaning     := "meaning" ident ":" string_lit_or_block   // §29 #5：目标 ident 须引用同模块已声明 type；块形多行
constraint  := "constraint" ident ":" "{" expr* "}"      // §29 #5：目标 ident 须引用已声明 type；value 绑定 §29 #4
agent       := "agent" ident "{" ("goal" ":" string_lit)? ("tools" ":" "[" expr_list "]")? "}"
tool        := "tool" ident "{" fn* "}"
actor       := "actor" ident "{" ("state" ":" field+)? ("message" ident ("{" field* "}")? | on_clause)* "}"   // §24 #5
on_clause   := "on" ident arm_body           // 消息处理：on AddUser { msg -> ... }（绑定复用 match §8）
task        := "task" ident "(" params? ")" block
server      := "server" ident "{" route* "}"
route       := "route" string_lit "{" fn* "}"
test        := "test" string_lit block
extern      := "extern" ident "{" extern_fn* "}"       // 如 extern c { ... }
extern_fn   := "fn" ident "(" params? ")" ("->" type)?

type        := ident ( "<" type_arg ("," type_arg)* ">" )?
type_arg    := type | "const" ident ":" ident                // const 泛型实参（§23 #3），如 Array<int, 3>
block       := "{" stmt* "}"
stmt        := "let" ident (":" type)? "=" expr
            |  "var" ident (":" type)? "=" expr
            |  "return" expr?
            |  "throw" expr                         // 语句级（§22 #10），类型 never（⊥，§26 #4）
            |  "if" | "for" | "while" | "loop" | "match" | "select"          // select 语句（§24 #10；完整形见下 select 产生式）
            |  "spawn" ("detached")? "actor"? ( postfix | block ) | "cancel" expr | "parallel" ("for"|"reduce") ...   // spawn task/block→TaskHandle、spawn actor Name()→ActorHandle（§24 #5）/ cancel h（§24 #4/#9）；postfix 含被调者＋调用后缀（call 仅后缀，故不可单用）
            |  try_catch                        // "try { } catch { }" 局部捕获（§26 #1，形见下 try_catch）；"try {" → 捕获形，"try" 后非 "{" → unary 传播算符（762），按下一 token 消歧
            |  "unsafe" block | expr

expr        := assign | send        // | send：actor 投递糖 `h ! Msg`（§24 #5），语句形二元 infix（右操作数贪婪至 expr）
assign      := or ( ("="|"+="|"-=") assign )?
or          := and ( "||" and )*
and         := eq ( "&&" eq )*
eq          := add ( ("=="|"!="|"<"|">"|"<="|">=") add )*
add         := mul ( ("+"|"-") mul )*
mul         := unary ( ("*"|"/"|"%") unary )*
unary       := ("-"|"!"|"borrow"|"borrow_mut"|"move"|"copy"|"await"|"try") unary | postfix   // 一元前缀：await/try（§24 #3、§26 #1）；try f() = unwrap-or-propagate
postfix     := primary ( call | index | field )*
call        := "<" type_arg ("," type_arg)* ">" "(" args? ")"  // 可选类型实参（§23 #7），如 decode<User>(row)
            |  "(" args? ")"
index       := "[" expr "]"
field       := "." ident
send        := postfix "!" expr        // actor 投递糖：h ! Msg ≡ h.send(Msg)（§24 #5）；"!" 二元 infix（投递），与 unary 前缀逻辑非按位置消歧
args        := expr ("," expr)*
primary     := literal | ident | "(" expr ")" | struct_init | list_lit | map_lit   // Variant(args) 复用 call 文法、由名字解析消歧（§26 #2）；约束类型构造 T(expr) 亦复用 call 文法（名字解析消歧「类型名 vs 函数 vs 变体」，§29 #2）；裸变体 = ident
list_lit    := "[" "]" | "[" expr ("," expr)* "]"
map_lit     := "[" ":" "]" | "[" expr ":" expr ("," expr ":" expr)* "]"

match       := "match" expr "{" arm* "}"
try_catch   := "try" block "catch" "{" arm* "}"    // 局部捕获（§26 #1）；"try" 后跟 "{" → 捕获形，否则为 unary 传播算符
arm         := pattern ":" arm_body
arm_body    := "{" bind ("," bind)* "->" block | block
pattern     := (ident ".")? ident | "_"
bind        := ident
select      := "select" "{" (sel_arm | sel_special)* "}"      // §24 #10
sel_arm     := ("await")? postfix "(" args? ")" arm_body        // receive()/send(v)/await f() 分支（§24 #10）
sel_special := "default" arm_body | "timeout" "(" expr ")" arm_body   // 非阻塞 / 超时（上下文关键字）
```

> 文法为骨架；`async/await`、泛型 `where`、`!` 投递糖（send，§24 #5）、`spawn actor` 等细节见对应小节与各领域文档。

---

## 19. 关键字精确集合与计数

基础 56 个（§2 表）。`true` / `false`（布尔字面量）、`void`（unit 类型）、`never`（bottom 类型，§26 #4）为**保留字面量/类型词，不计入 56**（§20 #5 已锁定）。`borrow_mut` 计为单关键字。

**上下文关键字**（仅特定语法位置保留、词法层产出 `Ident`、不入 56）：`where`（泛型 bound 子句，§7.10）、`constraint`（§7.4 声明引导词；与硬关键字 `meaning` 分类不同，为已知非对称）、`on`/`reply`/`detached`（§13 / §24）、`default`/`timeout`（select）、`reduce`（§24 #9）。另：`lock`/`yield`/`spawn_blocking` 为 `std.sync`/`std.async` 内建函数/控制构造（**非关键字**，§13 / §24 #7）。

---

## 20. 决议记录（v0.2.1 收敛）

> 下列原「开放问题」已于 v0.2.1 逐条决断；均**可逆**。本节为**紧凑索引**——权威表述已迁入正文（§7–§18），每项给一行决断摘要 + 正文指针。

1. **`interface`（dyn 派发、不可作泛型约束）与 `trait`（可作泛型约束、单态化）二分保留**——根因为 dyn vs 单态化（取代早期「服务 vs 行为」措辞，§23 #10）。=> 体现在 §7.11。
2. **外部类型实现 trait = v0.3 计划**：v0.2.1 仅内联 `implements` + orphan 规则（§28 #9）；v0.3 引入独立 `impl Trait for Type { }` 块。=> 体现在 §7.5 / §7.11。
3. **Constraint / 契约证明分级**：v0.2 字面量编译期检查 + 构造运行期断言 + `ensures` `old()` 快照；自动 SMT 证明留 v0.3+。=> 体现在 §7.4 / §12。
4. **`try/catch/throw` 解糖为 `Result` + `match`**（`try expr` 前缀一元、同错误类型、`throw` 语句级 never、显式 `Ok`、无 `finally`）。=> 体现在 §9 / §18。
5. **关键字计数 = 56**；`true`/`false`/`void`/`never` 为保留字面量/类型词，不计入 56。=> 体现在 §2 / §19。
6. **顶级声明边界锁定**：struct / actor / interface / trait / agent / tool / server / route 各自独立构造。=> 体现在 §15 / §18。
7. **`string` / 集合 = COW 值语义**（写时复制、原子 rc、行为同 Copy；资源型 move-only；无追踪 GC）。=> 体现在 §10.2。
8. **async 运行时 = M:N work-stealing，内建 `std.async`**（Channel = **MPMC**、`Future<T>` + 无 Pin、spawn 返回 TaskHandle + 结构化作用域、actor `on` handler、`spawn_blocking`）；v0.2 仅语义，codegen 后置。=> 体现在 §13。
9. **`match` arm 形式锁定**：`Pattern: { 绑定 -> body }` / `Pattern: { body }` / `Pattern: { a, b -> body }` / `_: { }`。=> 体现在 §8。
10. **`meaning` / `constraint` 独立声明 + `///` 双轨**（结构化权威、`///` 补充、按位置优先级填充）。=> 体现在 §1.5 / §7.3。

> 至此 v0.2.1 无未决开放问题；全部决议可逆。深审（EXAMPLES ↔ SPEC 语义级）14 项见 §21。

---

## 21. 决议记录 II：深审语义收敛（v0.2.1）

> §20 收敛后对 EXAMPLES ↔ SPEC 做了一轮**语义级深审**，发现 14 项语义矛盾 / 未定义点。下列已于 v0.2.1 内决断并就地传播；均**可逆**。本节为**紧凑索引**——权威表述在正文，每项一行摘要 + 指针。

1. `[示例]` **字面量强制到整型语义类型，作用于所有强制位**（绑定 + 调用实参，非仅 `let`）。=> 体现在 §7.3。
2. `[文法]` **payload-less `message` 合法**（`message Ident` 与 `message Ident { fields }`，镜像 enum variant）。=> 体现在 §18。
3. `[示例]` **§17 `get_user` 类型自洽**（边界抽取单值 + `Error → 命名错误` 转换；`query<T>` 后改非泛型 + `decode<T>` 边界解码，§23 #1）。=> 体现在 §17 / §7.11。
4. `[示例]` **`id: UserId = int` 为 Copy**（元数据标 `copy`）；interface dyn 句柄为 `borrow`。=> 体现在 §10.2 / §17。
5. `[语义]` **`task`（void body + CSP 通道，经 channel/cancel 通信、不强制 await）与 `async fn`（`Future<T>` + `await`）区分**；「须返回 Result」仅约束用户显式 await 的 async fn（取代早期 fire-and-forget 措辞，§24 #4）。=> 体现在 §13。
6. `[定义]` **`std.core.Error` 为动态边界 opt-in 通用错误**；惯用代码用命名 `error` 枚举；边界处 `Error → 命名错误` 显式映射。=> 体现在 §9。
7. `[定义]` **Channel 为共享 `Send` 句柄**；`spawn { }` / `parallel for` / `lock(m) { }` / `server route` 块体可捕获外层不可变 / `Send` 变量（非通用闭包，v0.2 无一等闭包）。=> 体现在 §13。
8. `[定义]` **server/route handler 捕获外层 `Send` 句柄**（控制构造，#7 之推论）。=> 体现在 §13。
9. `[定义]` **`print` 为 `std.core` 默认内建，无需 import**（类 Go `println`）。=> 体现在 §4.1 / §15。
10. `[示例]` **§16 裸 `Result` 改具型 `List<SearchResult>`**。=> 体现在 §16。
11. `[示例]` **`parallel for` 许可标准 = body 须 `pure` 或仅 `io.read`、且无共享可变**（效果门由 §24 #9 锁定）。=> 体现在 §14。
12. `[文法]` **`agent` 文法 `tools` 子句 `?`（至多一次）**。=> 体现在 §18。
13. `[语义]` **调用点 `borrow`/`borrow_mut`/`move`/`copy` 可选**（省略时按被调形参模式推断）。=> 体现在 §10.1。
14. `[语义]` **`Request.param<T>(name: string) -> T`**（HTTP 边界解析 + 强制）；`let id: UserId = req.param("id")` 合法。=> 体现在 §17。

> 至此深审 14 项全部决断并就地传播；v0.2.1 语义层自洽。深审 III（所有权 + 效果系统内在矛盾）10 项见 §22。

---

## 22. 决议记录 III：所有权 + 效果系统深审收敛（v0.2.1）

> §21 收敛后换维度做了一轮**所有权/效果模型内在自洽性**深审，发现 10 项定义空白 / 语义矛盾。下列已决断并就地传播；均**可逆**。本节为**紧凑索引**——权威表述在正文（#1 接收者模式为 #3/#5/#8/#9 及「无共享可变」判定的共同前置，单一权威处见 §7.5）。

1. `[定义]` **方法接收者模式（拱顶石）**：复用形参关键字 `borrow self`（默认）/ `borrow_mut self` / `self`（消费）；省略 = `borrow self`。=> 体现在 §7.5。
2. `[定义]` **COW 原子引用计数 → `Send`**；「无 GC」修订为「无追踪 GC；COW/RC 引用计数仍存，确定性释放」。=> 体现在 §10.2 / §10.4。
3. `[语义]` **`borrow_mut` 要求 COW 唯一所有权（rc == 1）**，突变原地；`borrow_mut` = opt-out COW 克隆。=> 体现在 §10.3。
4. `[语义]` **效果健全边界**：interface 方法必带 `effects`（dyn 取声明上界）/ `extern` 默认 `effects { extern }`（可收窄）/ 泛型随 trait bound 传播 → 「推断集 ⊆ 标注集」可强制。=> 体现在 §11。
5. `[示例/语义]` **`withdraw` 契约自洽**：`account: borrow_mut Account`（使 `ensures` 对 caller 可观测），`old()` 为调用前快照。=> 体现在 §12。
6. `[定义]` **interface 值为不可 Copy 的 Move 句柄**（vtable + 指针），经 `borrow` / `Shared<T>` / 连接池共享。=> 体现在 §7.11。
7. `[定义]` **`is_copy` 含 COW 字段**：COW 字段计 Copy-compatible → 全 Copy/COW 字段 struct 为 Copy，含任一 Move/Resource 字段则 Move。=> 体现在 §10.2。
8. `[定义/示例]` **`Resource.release(self)` 消费签名**（move 接收者），编译器在 owner 作用域末插入。=> 体现在 §10.5。
9. `[语义]` **`pure` 禁分配 + `alloc` 效果**：`pure` = 空效果集（含 `alloc`）且不改入参。=> 体现在 §11。
10. `[文法/语义]` **`throw` 语句级 + bottom（`never`）类型**，不可作子表达式（与 `return` 同）。=> 体现在 §9。

> 至此深审 III 的 10 项全部决断；v0.2.1 所有权 + 效果系统内在自洽。深审 IV（类型系统 + 泛型单态化）10 项见 §23。

---

## 23. 决议记录 IV：类型系统 + 泛型深审收敛（v0.2.1）

> §22 收敛后再换维度做了一轮**类型系统 / 泛型单态化内在自洽性**深审（聚焦「dyn 派发 vs 单态化」断层），发现 10 项定义空白 / 矛盾。下列已决断并就地传播；均**可逆**。本节为**紧凑索引**（#10 trait→单态化 / interface→dyn 二分是 #1 的根因）。

1. `[语义]` **interface 方法禁泛型**（dyn vtable 无法为每个 `T` 预留表项）；`Database.query` 非泛型 + `decode<T>` 边界解码。=> 体现在 §7.11。
2. `[文法/语义]` **`where { Bound<T> }`（编译期 trait bound）与 `requires { }`（运行期断言）拆分**；`ensures` 后置不变。=> 体现在 §7.10 / §12 / §18。
3. `[文法/示例]` **const 泛型 `Array<T, const N: int>`**（`layout(C)` 固定布局的必需；codegen 内联 `T × N`）。=> 体现在 §7.9 / §16.3 / §18。
4. `[定义]` **递归类型须显式 `Box<T>` 间接**（owned 堆、Move、非 Copy、确定性释放）。=> 体现在 §7.6。
5. `[定义]` **lang item（`#[lang]`）**：`Result`/`Optional`/`List`/`Map`/`string` 等被编译器特判的 std.core 类型。=> 体现在 §7.8。
6. `[定义/语义]` **trait 默认方法 + 覆写**：trait 方法带 body = 默认实现；`implements Trait { }` 可留空继承默认。=> 体现在 §7.5。
7. `[文法/语义]` **调用点显式泛型实参 `foo<T>(args)` + return-only 泛型须显式标注**。=> 体现在 §7.2 / §18。
8. `[定义]` **运算符重载 = std.core 运算符 trait 解糖**（`a + b` → `Add.add(a,b)` 等；泛型 bound 复用 `where { Ord<T> }`）。=> 体现在 §3b / §7.10。
9. `[定义]` **泛型不变性（invariance）**：`List<UserId>` 与 `List<int>` 无子类型关系（nominal）。=> 体现在 §7.3 / §7.10。
10. `[定义]` **trait bound → 编译期单态化；interface → 运行期 dyn 派发**（#1 的根因）。=> 体现在 §7.11。

> 至此深审 IV 的 10 项全部决断；v0.2.1 类型系统 + 泛型内在自洽。深审 V（并发 executor / async 状态机）10 项见 §24。

---

## 24. 决议记录 V：并发 executor / async 状态机深审收敛（v0.2）

> §23 收敛后再换维度做了一轮**并发运行时语义 / async 状态机**深审（聚焦 executor、Future、取消、Channel、Actor 行为），发现 10 项矛盾 / 定义空白。下列已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 async 返回 + 无 Pin 是相对 Rust async 的核心简化，#2 修正 Channel 模型，#5 补 Actor 行为）。
>
> **关键字计数**：本轮新增并发控制形式（`on` / `detached` / select `default`/`timeout` / `parallel reduce` / `spawn_blocking` / `yield`）均为**上下文关键字或 `std.async`/`std.sync` 函数**，不计入 §20 #5 的 56 全局关键字（同 `lock` 先例）。

1. `[语义/矛盾]` **`async fn f() -> T` 的 `T` 即 `await` 值**（`Future<T>` lang item）；「须返回 Result」仅约束用户显式 `await` 的 async fn，runtime 派发 handler 豁免；**状态机无 Pin**（borrow 禁跨 await，规则在 SPEC §10.3 / DESIGN §9.1 / MEMORY §7）。`task` 不可 async（void body + CSP 通道，经 channel/cancel 通信）。=> 体现在 §13。
2. `[定义/矛盾]` **Channel 改 MPMC**（取代 §20 #8 早期 MPSC）+ 无界/有界（`cap=0` rendezvous）+ `close` + `receive`/`try_receive`（均 `-> Optional<T>`，关闭耗尽 `None`）+ select 关闭检测。=> 体现在 §13。
3. `[文法]` **`await` 为一元前缀**（Pratt level 6，`await f()` / `await fut.field`；§18 `unary`）。=> 体现在 §3b / §18。
4. `[定义]` **`spawn` 返回 `TaskHandle` + `cancel` 操作数为 TaskHandle 表达式 + 结构化并发作用域**（默认 scoped、作用域末隐式 join；`spawn detached` 显式分离——**非** fire-and-forget；`cancel` 向子树传播）。=> 体现在 §13。
5. `[定义]` **Actor `on` 子句 + `ActorHandle`**（`on Msg { msg -> body }`，`spawn actor Name(...)` 返回 `ActorHandle`，`h.send`/`h !` 投递，`self.state` 串行访问）。=> 体现在 §13。
6. `[定义]` **`Send`/`Sync` 为自动派生 marker trait（lang item）+ 组合规则**：`Send` 当且仅当所有字段 `Send`、`Sync` 当且仅当所有字段 `Sync`；Copy → Send+Sync、COW → Send、Move 资源 → Send 非 Sync、`Mutex<T>`/`Shared<T>` → Send+Sync。=> 体现在 §10.4。
7. `[语义]` **阻塞调用与 executor**：`async fn` 内禁 `io.blocking`，阻塞经 `spawn_blocking { }`（独立 OS 线程池）或 async 变体；新增 effect `io.blocking`。=> 体现在 §13 / §11。
8. `[语义]` **取消语义**：协作式（仅 `await` 点生效，CPU-bound 须 `task.yield()`），丢弃状态机 → 确定性释放 locals，`lock(m){ }` 边界自动释放锁，`cancel` 向子树传播。=> 体现在 §13。
9. `[语义]` **`parallel for` 表达式化（返回 `List<R>`）+ `parallel reduce(op, xs)`**（`op` 须结合律；body 须 `pure` 或仅 `io.read`、无共享可变）。=> 体现在 §13 / §14。
10. `[语义]` **select 扩展**（`receive`/`send`/`await` 分支 + `default { }` + `timeout(d) { }`；多分支就绪随机公平择一；关闭 channel 的 receive 立即可就绪，交付 `None`）。=> 体现在 §13 / §18。

> 至此深审 V 的 10 项全部决断；v0.2 并发 executor / async 状态机语义自洽。深审 VI（std API 一致性 / `.ailmeta` schema 完整性）10 项见 §25。

---

## 25. 决议记录 VI：std API 一致性 / `.ailmeta` schema 深审收敛（v0.2）

> §24 收敛后再换维度做了一轮**标准库 API 一致性 + AI 元数据 schema 完整性**深审——`.ailmeta` 是 AILang 存在的全部理由（WHITEPAPER §3），却四处样例用了 ≥3 种 schema 形、且无正式字段规范。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 schema 统一 + #2 正式 schema 规范为 keystone；多数权威表述落在 WHITEPAPER/STDLIB，#6 effect 词表落 SPEC §11）。

1. `[一致性/矛盾]` **`.ailmeta` schema 四处样例统一为 WHITEPAPER §3 权威富形**（STDLIB §5 / EXAMPLES §19 / EXAMPLES §11 对齐）。=> 体现在 WHITEPAPER §3。
2. `[定义]` **正式 `.ailmeta` schema 规范**：字段表 + 枚举（`mode` / `ownership.output` / `kind`）+ `schema_version` 独立 semver。=> 体现在 WHITEPAPER §3。
3. `[一致性]` **`input[].mode` 权威；`ownership` 删 `inputs[]` 仅留 `output`**（消除双重编码）。=> 体现在 WHITEPAPER §3。
4. `[完整性]` **`.ailmeta` 增 `declarations[]`**（agent/tool/actor/server/task 顶级声明容器；`tool` 派生 OpenAI Tool Schema）。=> 体现在 WHITEPAPER §3。
5. `[完整性]` **`.ailmeta` 每条目增 `span` 源码定位**（`span{file,line_start,col_start,...}`，与编译器 Span 一致）。=> 体现在 WHITEPAPER §3。
6. `[定义]` **effect 词表规约 `<domain>.<verb>`**（`database.read|write` / `network` / `io.*` / `filesystem.*` / `alloc` / `extern` / `unsafe`；`pure` = 空集）。=> 体现在 SPEC §11。
7. `[完整性]` **std 方法签名表（含接收者模式）**：为 `List`/`Map`/`Set`/`string`/`Optional`/`Result` 出签名表。=> 体现在 STDLIB §7/§8/§9。
8. `[一致性]` **样例引用类型归属**：`Request`/`Response`/`URL`→std.http、`Row`→std.database、`SearchResult`→std.ai。=> 体现在 STDLIB §11/§12/§13。
9. `[完整性]` **空白 std 模块占位**（math/io/net/crypto/time）；`print` 归 std.core、`std.io` 为流式 IO。=> 体现在 STDLIB §6。
10. `[一致性]` **`print(x: Display) -> void` 签名 + `decode<T>` 归 std.database + 错误枚举按模块编目**。=> 体现在 STDLIB §7/§10/§11/§13。

> 至此深审 VI 的 10 项全部决断；v0.2 标准库 API 一致性 + `.ailmeta` schema 完整性收敛。深审 VII（错误处理模型精确语义）10 项见 §26。

## 26. 决议记录 VII：错误处理模型深审收敛（v0.2）

> §25 收敛后再换维度做了一轮**错误处理模型精确语义**深审——`try/catch/throw` 解糖虽于 §20 #4 锁定骨架，但文法悬空、值构造未定、panic 模型缺失、错误转换/并发传递未落到精确语义。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 try/catch 文法 + #3 panic 模型为 keystone）。

1. `[文法/完整性]` **`try expr` 为前缀一元（与 `await` 一致，§18 `unary`）+ `try { } catch { }` 捕获形式**（新增 `try_catch` 产生式，按下一 token 消歧）；拒后缀 `expr try` 与 sigil `?`；56 不变。=> 体现在 §9 / §18。
2. `[文法/语义]` **`Result`/变体值构造复用 call 文法**（名字解析消歧），裸变体 = ident；成功返回须显式 `Ok(x)`、无 auto-wrap。=> 体现在 §9 / §18。
3. `[定义]` **panic = 确定性栈展开 + Drop/release，展开至 Task 边界**（不传染进程）；v0.2 无 `catch_unwind`；`unwrap`/`assert` 失败 = panic，`std.core.panic` 为 std 函数（非关键字）。=> 体现在 §9。
4. `[定义]` **`⊥` = `never`（空类型，内建 lang item）**：可作返回类型，`never | T = T`；`never ≠ void`；为保留类型词，不计 56。=> 体现在 §7.1 / §9 / §19。
5. `[语义]` **错误转换无 `From`**：`try expr` 要求同错误类型（`E1≠E2` 类型错）；转换唯一手段 = `match { 错误分支 → throw 新变体 }`。=> 体现在 §9。
6. `[语义]` **`throw` 合法性**：仅合法于返回 `Result<T,E>` 且 `V∈E` 的体；`void`/`task`/actor `on`/server handler 禁 throw（错误经各自通道）。=> 体现在 §9。
7. `[定义]` **`error` 变体可带 payload**（与 enum 同形 `Variant(field: T)`）；`.ailmeta` `errors[]` 每项可选 `payload`；惯法推荐裸变体。=> 体现在 §9。
8. `[一致性]` **并发错误传递统一表**：`async fn -> Result` 可 throw/try；`task` 禁 throw（经 channel/cancel）；actor `on` 禁 throw（经 `reply(Err...)`，reply 值可 Result）；server handler ✗ 禁 throw（豁免于须返回 Result，runtime 派发）。=> 体现在 §13。
9. `[语义]` **`pure fn` 可 throw/try**（throw 非 effect、不改入参）；`is_pure` 仅看 effect 集与入参突变，与 `errors[]` 正交。=> 体现在 §9 / §11。
10. `[完整性]` **`Result<void,E>` 成功构造 = `return Ok(void)`（显式，无 auto-wrap）；`errors[]` 单一真源**（≡ 签名 `E` 变体，throw/try 永不越界；非 Result 返回 → `errors:[]`）。=> 体现在 §9。

> 至此深审 VII 的 10 项全部决断；v0.2 错误处理模型精确语义收敛。深审 VIII（数值语义 + 逃生舱边界）5 项见 §27。

---

## 27. 决议记录 VIII：数值语义 + 逃生舱边界深审收敛（v0.2）

> §26 收敛后做了一轮**全面性能/安全复核**，暴露 5 个未被前 7 轮覆盖的真实语义缺口：整数溢出、浮点 NaN/特殊值、除零、`unsafe` 边界精确清单、Drop 语义细节（外加 panic 跨 FFI，并入 #4）。下列 5 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 整数溢出为 keystone）。

1. `[定义]` **整数算术溢出 = 定义行为，永不 UB**：release 二补数回绕（规范），debug 溢出 → panic `ArithmeticOverflow`（开发期检测层）；`std.math` 提供 `wrapping_*`/`checked_*`/`saturating_*`/`overflowing_*` 运算符族。=> 体现在 §7.1 / §9。
2. `[定义]` **浮点遵循 IEEE 754**：`x/0.0`→`±inf`、`0.0/0.0`→`NaN`、NaN 经运算传播、`NaN==NaN` 为 `false`，**浮点运算永不 panic**；`std.math` 提供 `is_nan`/`is_inf`/`is_finite`（float 方法 `x.is_nan()`）。=> 体现在 §7.1。
3. `[语义]` **除零语义**：整数除零/取模零 → panic `DivideByZero`（确定性）；浮点除零遵循 IEEE（不 panic）；`std.math.checked_div` 安全除法。=> 体现在 §7.1 / §9。
4. `[定义]` **`unsafe` 边界精确清单**（裸指针解引用 / 调 `extern` fn / 调 `unsafe` fn；不引入 `static mut`、无 union；清单外操作非 unsafe）+ **panic 跨 FFI 边界 → 进程 abort**（非 UB；C 的 `longjmp` 跨 FFI = UB 禁）。=> 体现在 §16 / §9。
5. `[定义]` **Drop 语义精确化**：字段 Drop 顺序 = 声明顺序；安全代码结构上无循环引用 → 不引入 `Weak`/`Rc`/`Arc`（共享用 Actor/`Mutex<T>`/`Shared<T>`）；panic 展开时同序 Drop；自定义释放经 `Resource.release(self)`。=> 体现在 §10.5。

> 至此深审 VIII 的 5 项全部决断；v0.2 数值语义 + 逃生舱边界收敛。深审 IX（模块系统 / 可见性 / 导入 / 包语义）10 项见 §28。

---

## 28. 决议记录 IX：模块系统 / 可见性 / 导入 / 包语义深审收敛（v0.2）

> 前 8 轮覆盖语法/类型/所有权/泛型/并发/std+schema/错误/数值+逃生舱，唯独**模块系统**——6 个关键字（`package`/`import`/`from`/`as`/`public`/`private`）出现在几乎每个样例、却无任何专门规范；且 §18 引用 `module_path`/`visibility` 两非终结符无产生式，SPEC §4 与 EXAMPLES 在 `std.` 前缀上真实矛盾。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 文法 + #3 Go 式包模型为 keystone，阻塞 Phase 1）。

1. `[文法]` **补 `module_path := ident ("." ident)*` 与 `visibility := "public" | "private"` 产生式**；`module := "package" module_path import* item*`（包名升级为点分路径）。=> 体现在 §18。
2. `[定义]` **可见性粒度 + 默认 private**：`public`/`private` 修饰所有顶级 item + struct/enum 字段 + 方法；trait/interface 方法不单独修饰（随类型）；跨模块仅可访问 `public`。=> 体现在 §14。
3. `[定义]` **`package <dotted-path>` Go 式包模型**：同一 package 声明可跨多文件共享命名空间；文件路径须与声明模块路径一致（编译期校验）；根包名 = `ail.toml.name`。=> 体现在 §4.1。
4. `[一致性]` **导入路径必含根包**（标准库根 `std` → `import std.http`，`std.` 必需）；**`std.core` 自动加载（免 import，§21 #9）**，其余 `std.*` 显式 import；第三方库 `import <pkg>.<module>`。=> 体现在 §4.1 / §4.2。
5. `[语义]` **导入冲突 = 编译错误（须 `as` 消歧）/ 本地定义优先遮蔽（+ warn）/ `import X as Y` 别名 / 禁 `import *` 通配**。=> 体现在 §4.2。
6. `[定义]` **`from M import name` 仅可导入 M 的 public 顶级 item**（导入 private = 编译错误）。=> 体现在 §4.2。
7. `[语义]` **v0.2 不支持再导出**（无 `public import` / `pub use`）；要 `b` 的 item 须显式 `import b`。=> 体现在 §4.3。
8. `[语义]` **循环导入**：类型级 / 值级 OK（惰性绑定）；顶层初始化循环（`const`/全局求值成环）= 编译错误；须两遍分析。=> 体现在 §4.3。
9. `[定义]` **orphan 规则**：`Trait` 或 `Type` 之一须定义于当前 package（类 Rust coherence）；`public` 类型 impl 本 package `public` trait 可跨模块可见。=> 体现在 §7.5 / §7.11。
10. `[完整性]` **`ail.toml.name` = 根包名 = 导入根前缀**；依赖 `dep = "1.2.3"` 安装后 `import dep.<module>`；版本遵循 semver；包名禁与 `std` 冲突。=> 体现在 §4.1。

> 至此深审 IX 的 10 项全部决断；v0.2 模块系统收敛。深审 X（语义类型 / 约束类型 / `meaning` 传播 / 约束求值）10 项见 §29。

---

## 29. 决议记录 X：语义类型 / 约束类型 / meaning 传播 / 约束求值深审收敛（v0.2）

> 前 9 轮覆盖语法/所有权/泛型/并发/std/错误/数值/模块，唯独 **AILang 立身之本——「AI First 类型系统」**（语义类型 / 约束类型 / `meaning` 传播 / 约束求值）作为连贯子系统从未专门深审。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 type X=Y 分类 + #2 约束构造算符为 keystone）。

1. `[定义/文法]` **`type X = Y` 三义性裁定**：无 `meaning`/`constraint` → 透明别名（`kind: alias`）；附 `meaning` → 名义 semantic（`kind: semantic`）；附 `constraint` → semantic + constraint。即 meaning/constraint 的存在触发 nominal。=> 体现在 §7.3。
2. `[语义]` **约束类型构造算符 `T(expr)`**（复用 call 文法，名字解析消歧）：运行期断言谓词，违约 panic `ConstraintViolation`（非 `Result`，不入 `.ailmeta errors`）；字面量走编译期折叠检查。=> 体现在 §7.4。
3. `[语义/完整性]` **`meaning` 按名义类型附着传播**：带 meaning 的 semantic 值在 `.ailmeta` 各处携带；函数 output.meaning = 返回类型 meaning（函数级不可另写）；bare 基础类型 / alias = null。=> 体现在 §7.3。
4. `[定义]` **约束谓词语言**：`value` 魔法只读绑定 + 纯布尔表达式（可 `value.field` / `value.method()` / 调 `pure fn`）；禁 `old()` / 副作用 / IO；谓词须 pure。=> 体现在 §7.4。
5. `[文法]` **`meaning`/`constraint` 文法**（目标 ident 须引用同模块已声明 `type`；`meaning` 块形支持多行）。=> 体现在 §18。
6. `[一致性]` **Constraint Type = Semantic Type + `constraint`**（特化，非独立 kind；`.ailmeta` 仍 `kind: semantic` + constraint 字段）。=> 体现在 §7.4。
7. `[语义]` **semantic 运算继承边界（marker vs operational 二分）**：**标记 trait（`Copy`/`Send`/`Sync`）自动继承 base**；**运算 trait（`Add`/`Eq`/`Ord`/...）不自动继承**（v0.2.1 无给语义新类型实现运算 trait 的语法，为 v0.3 计划）；例外：`Display` 自动经 base + 字面量强制。=> 体现在 §7.3。
8. `[一致性]` **`meaning` 双轨优先级统一**：type meaning = `meaning` 声明权威（type 上 `///` → description，冲突 `meaning` 胜）；function / param / field meaning = 仅 `///`；字段 meaning 优先字段类型 meaning。=> 体现在 §1.5 / §7.3。
9. `[完整性]` **约束违约 panic 名 `ConstraintViolation`**（与 `ArithmeticOverflow` / `DivideByZero` / `IndexOutOfBounds` 同族；构造算符违约即此；56 关键字不变）。=> 体现在 §9 / §7.4。
10. `[一致性/完整性]` **`type X = Y` 复杂/不透明体 kind**：无 meaning = alias（base 可不透明，记规范化类型串）；有 meaning = semantic；`...` 仅示例简写非真实语法。=> 体现在 §7.3。

> 至此深审 X 的 10 项全部决断；v0.2 语义/约束类型收敛。全部决议可逆，记录于此供复盘。跨文档一致性终审收敛（五轮 workflow 迭代）见 §30（四审）、§31（五审）。

---

## 30. 决议记录 XI：跨文档一致性终审收敛（v0.2.1）

> §20–§29（记录 I–X）逐子系统深审收敛后，又跑四轮**全文档集一致性 workflow**（一审→四审，每轮 12 维度审查 + 对抗式验证），逐轮递减暴露跨切面残留（一审多 → 二审 13 → 三审 8 → 四审 4 unique / 0 HIGH）。下列 4 项为四审终验后落定的跨文档决断与机械修复；均**可逆**。本节为**紧凑索引**（#1 §18 零悬空 + #2 ownership 删死值为决断，#3/#4 为机械一致性修复）。

1. `[文法/决断]` **§18 零悬空产生式**：`send`（actor 投递糖 `h ! Msg`）接入 expr 链（`expr := assign | send`，§24 #5）；`try_catch`（§26 #1）/ `select`（§24 #10）接入 stmt（与 `match` 同层）。决断：§18 所有产生式须从起符可达——`async`/`await`/`where`/`spawn actor` 既已最小接线，`send`/`try_catch`/`select` 不再悬空例外。代价：`!` 二元 infix（投递）vs unary 前缀（逻辑非）、`"try {"`（捕获）vs `"try" expr`（unary 传播）按下一 token 消歧。=> 体现在 §18。
2. `[schema/决断]` **`ownership.output` 删 `borrowed` 死值** → `new | move`：§10.3 禁 borrow 返回（核心不变量，5 处复述）→ `borrowed`（借用返回）不可达，与 schema 自相矛盾；删之以消解 §25 #2（深审 VI）遗留四轮的 schema-vs-不变量矛盾。代价：schema 枚举变更（无函数依赖该值）。=> 体现在 WHITEPAPER §3 / DESIGN §7 / SPEC §25 #2。
3. `[一致性]` **DESIGN §6 spawn 镜像同步 `"actor"?`**（注释 `SPEC §24 #4/#5/#9`）：M2「spawn 三形式」更新 SPEC §18 后，DESIGN §6 近乎逐字镜像须同步。=> 体现在 DESIGN §6。
4. `[一致性]` **DESIGN §6 try/catch 措辞「表达式级捕获」→「局部捕获」**：对齐权威术语（SPEC §9 / §18 `try_catch` / §26 #1 + EXAMPLES 一致用「局部捕获」）。=> 体现在 DESIGN §6。

> 至此四轮一致性终审收敛；v0.2.1 全文档集自洽——§18 零悬空产生式、ownership schema 零死值（五审续见 §31）。全部决议可逆，记录于此供复盘。

---

## 31. 决议记录 XII：跨文档一致性五审收敛（v0.2.1）

> 记录 XI（§30）四审终验后，又跑第五轮**全文档集一致性 workflow**（一审→五审，12 维度审查 + 对抗式验证）。收敛轨迹：一审多 → 二审 13 → 三审 8 → 四审 4 → **五审 6**（0 HIGH）。五审回升（4→6）因重读 §18 邻域 + 审 §30 新内容，暴露 2 处四审自引入回归 + 4 处预存缺陷——前四轮只查产生式**可达性**、漏查**引用正确性**。下列 6 项：#3 为决断，余为缺陷清理（#1/#2 修四审回归，#4/#5/#6 对齐权威）；均**可逆**。本节为**紧凑索引**。

1. `[回归/一致性]` **WHITEPAPER §3 裸 `§10.3` → `SPEC §10.3`**：四审 C1「ownership.output 删 borrowed」新增「无 borrowed——§10.3 禁 borrow 返回」注解时漏带 SPEC 前缀（同句 SPEC §25 #2 已带，段内不对称；全集另 9 处 §10.3 跨文档引用均带前缀）。=> 体现在 WHITEPAPER §3。
2. `[回归/一致性]` **§30 #2「体现在」指针矫正**：四审 sink §30 记录 XI 时，#2 落点误植 `DESIGN §6`（实为 `DESIGN §7`——ownership.output 的 OwnershipSig HIR 结构在 §7 模块三 AST，非 §6 Parser）＋ `§25 #2` 漏 SPEC 前缀（F4 跨文档分布）。一行两错。=> 体现在 §30 #2。
3. `[文法/决断]` **§18 spawn 引用矫正** `call` → `postfix`：spawn 产生式 `( call | block )` 的 `call`（§18 定义为纯调用后缀 `(<…>) | (args)`，不含被调者）无法归约 `spawn download()` / `spawn actor Name()`（须 `primary + call = postfix`），与 §13/§1/CONCURRENCY「均合法（§18）」矛盾——前四轮查可达性漏此引用错。改 `( postfix | block )`（与 `send := postfix "!" expr` 一致），DESIGN §6 镜像同步。代价：`postfix` 允许 `spawn x.y()`（方法引用 spawn，合理）、裸值（语义分析拒，骨架可忍）——合理过生成；`call` 仍被 `postfix` 引用，零悬空不破。=> 体现在 §18 / DESIGN §6。
4. `[一致性]` **EXAMPLES §3 `math.is_nan` → `x.is_nan()` 方法形**：浮点 IEEE 754 特殊值检测（is_nan/is_inf/is_finite）为 float 方法（`is_nan(borrow self)`，STDLIB §6.1），非 std.math 模块级自由函数；原与同列真自由函数 `math.checked_add` 混用模块前缀。整数（checked_add 等）与浮点（is_nan 方法）拆分表述。=> 体现在 EXAMPLES §3 / STDLIB §6.1。
5. `[一致性]` **WHITEPAPER §3 type 分类两分支 → 三分支**：原「无 meaning→alias / 附 meaning→semantic」漏 constraint 触发分支，致 constraint-only `Age`（无 meaning）按字面应判 alias，与 EXAMPLES §4 实际发射 `kind:semantic` 矛盾。补全对齐 SPEC §29 #1：无 meaning/constraint→alias；附 meaning 或 constraint→semantic。=> 体现在 WHITEPAPER §3 / SPEC §29 #1。
6. `[一致性]` **EXAMPLES §12 注释「无 GC」→「无追踪 GC」**：确定性释放代码注释残留裸「无 GC」，与 §22 #2 决断「『无 GC』修订为『无追踪 GC；COW/RC 引用计数仍存，确定性释放』」不一致（全集正式处均已改）。=> 体现在 EXAMPLES §12 / SPEC §22 #2。

> 至此五轮一致性终审收敛；v0.2.1 全文档集自洽——§18 零悬空产生式 + spawn 引用正确、ownership schema 零死值、F4 引用前缀清洁、类型三分支对齐。全部决议可逆，记录于此供复盘。
