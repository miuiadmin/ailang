# AILang 语言设计文档

> **AILang：面向 AI 时代设计的高性能系统级编程语言。**
>
> **一句话定位**：让人类与 AI 共同创造可靠、高性能的软件系统。
>
> 本文档是 AILang 的**单一权威设计文档**——整合设计哲学、语言规范、标准库、教程、编译器实现设计与全部决议记录。规范为 v0.2.1（冻结）。
>
> **文档结构**：Part 0 阐明动机与哲学；Part I 是权威语言规范（含内存/并发深解）；Part II 是标准库与包生态；Part III 是可运行教程；Part IV 是编译器实现设计（受众=编译器编写者）；Part V 是 v0.2.1 收敛的决议记录（可逆）；附录收版本演进、工程遗留与结语。
>
> **历史**：本文由原 8 份分立文档（WHITEPAPER / SPEC / PHILOSOPHY / EXAMPLES / DESIGN / STDLIB / MEMORY / CONCURRENCY）于 v0.2.1 合并而成，消除跨文档引用与重复表述，建立单一章节命名空间（§1–§94 + 附录 A–C）。原分立文档可在 git 历史（commit `ce6143f` 及之前）查阅。

---

## 目录

- **Part 0 · 前言与动机**（§1 摘要 / §2 为什么是 AILang / §3 四大核心原则 / §4 与现有语言的关系 / §5 语言导览 / §6 名称与宣言）
- **Part I · 语言规范**（§7 语法层原则 / §8 词法 / §9 关键字 / §10 基础语法 / §11 运算符 / §12 模块 / §13 变量 / §14 函数 / §15 类型系统 / §16 控制流 / §17 错误模型 / §18 所有权与内存模型 / §19 效果系统 / §20 契约 / §21 并发模型 / §22 可见性 / §23 AI 原生声明 / §24 .ailmeta 规范 / §25 Unsafe 与 FFI / §26 完整示例 / §27 形式文法）
- **Part II · 标准库与包生态**（§28–§43：设计目标 / 命令集 / 包结构 / 包模型 / .ailmeta 包产物 / 模块总览 / std.core / collections / string / fs / http / database / ai / testing / 发布安全 / 生态优势）
- **Part III · 教程：可运行示例**（§44 约定 / §45–§63 示例 1–19 / §64 特性覆盖索引）
- **Part IV · 编译器实现设计**（§65–§82：哲学 / 总体架构 / 工具 / 项目结构 / ail-ast / Lexer / Parser / AST / Type Checker / Ownership Checker / AI Analyzer / IR / Codegen / Metadata / 工具链 / MVP 路线 / 设计决议 / 开放问题）
- **Part V · 决议记录**（§83–§94：记录 I–XII，110 项可逆决断）
- **附录**（A 版本演进与收敛 / B 工程遗留 / C 结语）

---


<a id="part-0"></a>
# Part 0 · 前言与动机

## 1. 摘要

软件开发正在进入一个新阶段：AI 不再只是辅助工具，而正在成为代码的主要生产者之一。然而，现有编程语言几乎全部围绕「机器」与「人类程序员」两个对象设计——它们对人友好、对编译器友好，却没有为「让 AI 精确理解程序」这一目标做过优化。结果是 AI 在生成代码时大量依赖猜测：猜测参数含义、猜测副作用、猜测错误可能、猜测资源归属。猜测是错误的来源。

**AILang 是第一代面向 AI 理解的系统级编程语言。** 它在源码层面强制程序主动表达**意图**——通过语义类型（Semantic Type，可带 constraint）、效果系统（Effect System）、显式所有权（Ownership）与结构化错误模型——并由编译器将这些语义编译为机器可读的元数据（`.ailmeta`），供 AI Agent、IDE 与自动化工具直接消费。同时，AILang 通过 LLVM 后端与所有权分析达到接近 Rust 的原生性能，是一个真正的系统级语言，而非脚本语言。

AILang 的目标是：**让人类与 AI 共同创造可靠、高性能的软件系统。**

---

## 2. 为什么是 AILang

### 2.1 AI 成为代码生产者

过去几十年，编程语言主要围绕两个对象设计：**机器**与**人类程序员**。C 追求接近硬件；Java 追求工程生产力；Python 追求开发效率；Rust 追求安全和性能；Go 追求简单和工程规模。

但是 AI 编程时代出现了新的参与者：

> AI 不再只是辅助工具，而正在成为代码的主要生产者之一。

未来大量代码可能由 AI Agent、自动代码生成系统、智能开发工具完成。这带来一个新问题：

> 如何让 AI **精确理解**程序，而不是通过猜测生成代码？

**现有语言对 AI 不够友好**

```ail
fn add(a: int, b: int) -> int {
    return a + b
}
```

编译器完全理解，人类大致理解，但 AI 缺少关键信息：`a`、`b` 的现实含义？是否允许负数？是否有副作用？是否可能失败？是否取得输入的所有权？这些信息在人脑中靠命名约定与经验补全，但对 AI 而言是搜索空间里的未知，只能靠猜。

### 2.2 AILang 的核心目标

AILang 不只是让人写代码。它的目标是：

> 让人类和 AI 都能够准确理解、生成、维护高可靠软件。

传统语言主要解决：

```
人类 → 编写 → 机器执行
```

AILang 解决：

```
人类
  ↓
AILang 代码
  ↓
AI 理解 / 生成 / 维护
  ↓
编译器验证
  ↓
机器执行
```

### 2.3 AILang 的主张

> 优秀的代码不应该隐藏信息，而应该主动表达意图。

---

## 3. 四大核心设计原则

### 3.1 Rust 级安全

目标：内存安全、无空指针、无数据竞争、编译期检查、无 GC 强依赖。

采用：Ownership、Borrow Analysis、Static Type System。

**但是：不复制 Rust 的复杂语法。** 例如生命周期由编译器推导，不暴露给程序员：

```rust
// Rust：程序员必须标注生命周期
fn process<'a>(data: &'a str) -> &'a str
```

```ail
// AILang：编译器自动推导，无需标注
fn process(data: string) -> string {
    ...
}
```

### 3.2 Rust 级性能

AILang 是编译型语言，而非脚本语言：

```
源码 → AST → Type Checker / Ownership Analyzer / AI Analyzer → HIR（AILang IR）→ .ailmeta → LLVM IR → Machine Code
```

输出原生二进制（`.exe` / ELF / Mach-O），无虚拟机、无解释器、原生执行。

### 3.3 Python 级简洁 + C 系大括号结构

表达简洁接近 Python，但**代码结构用 C/Go/Rust 风格的 `{ }` 大括号**，不采用 Python 的强制缩进语义。

理由：大括号让 AST 更稳定、复制代码不破坏结构、IDE 与 AI 更易解析、生成错误率更低——这更适合系统级语言。简洁性来自少符号与可选分号，而非缩进。

```python
# Python（缩进式）
def add(a, b):
    return a + b
```

```ail
// AILang（大括号 + 可选分号 + 类型表达）
fn add(a: int, b: int) -> int {
    return a + b
}
```

### 3.4 AI First 类型系统（AILang 最大区别）

传统语言：`type = 数据格式`。
AILang：让类型可携带 **含义 + 约束 + 行为**（语义类型 / Semantic Type；§15.3 / §92 #1）；裸 `type X = Y` 仍可为透明别名。

例如，定义一个语义类型（Semantic Type；§15.3）：

```ail
type UserId = int
meaning UserId:
    "unique identifier of a user"
```

AI 看到的不只是 `int`，而是：

```
UserId
├── 基础类型: int
├── 含义: 用户唯一标识
└── 不可用于金额计算
```

> Rust 让程序员告诉编译器安全规则；AILang 让编译器自动推导安全规则，让 AI 更容易生成正确代码。

---

## 4. 与现有语言的关系

AILang 不试图替代所有语言，而是吸收各所长处，补上 AI 原生语义这一缺失维度：

> Rust：让**程序员**告诉编译器安全规则。
>
> AILang：让**编译器**自动推导安全规则，让 AI 更容易生成正确代码。

典型体现：生命周期不再由人标注（见 §18.5），所有权与借用规则由编译器内部的分析器推导。

吸收关系：

| 语言 | AILang 吸收 | AILang 的不同 |
|------|-------------|---------------|
| **Rust** | 性能、内存安全、所有权 | 无生命周期标注；borrow 不逃逸；产出 AI 元数据 |
| **TypeScript** | 类型表达、接口 | 编译型系统语言；名义语义类型 + 约束类型 |
| **Python** | 简洁表达 | 静态类型、零运行时开销 |
| **Go** | 大括号工程结构、少而明确 | 显式效果与所有权 |
| **AILang 独有** | AI 原生语义（Semantic Type / `.ailmeta`） | — |

一句话：

> AILang = Python 的简洁 + C/Go/Rust 风格的大括号结构 + Rust 的安全模型 + TypeScript 的类型表达 + AI 原生语义。

能力对比（Go / Rust / AILang）：

| 能力 | Go | Rust | AILang |
|------|----|------|--------|
| 轻量任务 | ★★★★★ | ★★★ | ★★★★★ |
| 内存安全 | ★★ | ★★★★★ | ★★★★★ |
| Ownership / Borrow Checker | — | ✅ | ✅ |
| 生命周期语法 | — | 复杂（显式 `'a`） | **自动推导（隐藏）** |
| 追踪式 GC | ✅ | 无 | 无 |
| 性能 | ★★★ | ★★★★★ | ★★★★★（目标） |
| 学习成本 / 语法难度 | 低 | 高 | 低 |
| AI 理解度 | ★★★ | ★★★ | ★★★★★ |
| 消息模型 | ★★★★★ | ★★★ | ★★★★★ |
| 自动分析 | ★★ | ★★★ | ★★★★★ |

---

## 5. 语言导览（at a glance）

### 5.1 完整程序 + 双产物

下面是一个完整的 AILang 程序（`.ail`，大括号 + 可选分号），集中展示语义类型、结构体、错误枚举、效果标注与 `Result`：

```ail
package app

type UserId = int
meaning UserId:
    "unique identifier of a user"

struct User {
    id: UserId
    name: string
}

error UserError {
    NotFound
    PermissionDenied
}

fn get_user(id: UserId) -> Result<User, UserError>
effects {
    database.read
}
{
    // 简化示例；完整签名（含 db: borrow Database 参数）与实现见 §63
    ...
}

fn main() -> void {
    let user = get_user(1)
    print(user)
}
```

**编译它，会同时产出两样东西：**

**(a) 原生可执行文件** `app`（经 LLVM）。

**(b) 机器可读的语义元数据** `app.ailmeta`，这就是 AI 直接消费的东西：

```json
{
  "schema_version": "0.2.0",
  "language": "AILang",
  "package": "app",
  "functions": [
    {
      "identity": { "name": "get_user", "description": null },
      "span": { "file": "src/main.ail", "line_start": 10, "col_start": 1, "line_end": 14, "col_end": 1 },
      "input": [
        { "name": "id", "type": "UserId", "meaning": "unique identifier of a user", "mode": "copy" }
      ],
      "output": { "type": "Result<User, UserError>", "meaning": null },
      "effects": ["database.read"],
      "errors": ["NotFound", "PermissionDenied"],
      "ownership": { "output": "new" },
      "is_pure": false,
      "is_async": false
    }
  ],
  "types": [ ... ],
  "errors": [ ... ],
  "declarations": [ ... ]
}
```

（`.ailmeta` 字段规范见 §24；完整示例见 §63。）

传统编译器只产出二进制。AILang 的编译器在编译同时产出结构化的语义元数据 `*.ailmeta`，提供：身份与意图（`identity`）、输入/输出语义（含 `meaning` 与所有权 `mode`）、行为边界（`effects`）、失败可能（`errors`）、所有权签名、源码定位。

**关键：`.ailmeta` 的内容不是注释或约定，而是编译器强制验证后的事实。** 若一个函数声称 `pure`，编译器已校验它确实无副作用；若声称 `effects { database.read }`，编译器已校验它不会写库。AI 可以信任 `.ailmeta`，如同信任类型签名。

**这一份 `.ailmeta` 是 AILang 存在的全部理由。** AI 不需要「阅读源码并猜测」——它读到的是一份结构化、可验证、由编译器背书的程序语义说明书。

### 5.2 核心语言速览

- **程序结构**：`package <路径>` 开头（点分模块路径，如 `package app` / `package std.http`），可跟 `import` / `from ... import` / `import ... as`（**全路径**，如 `import std.http`；§91）。
- **变量**：`let`（不可变）/ `var`（可变）/ `const`（编译期常量）。
- **函数**：`fn name(params) -> Type { }`；`pure` / `async` 为前缀修饰；`where`（泛型 bound，编译期）与 `effects` / `requires` / `ensures`（契约块）置于签名与函数体之间。
- **控制流**：`if` / `else` / `match` / `for` / `while` / `loop` / `break` / `continue`，统一 `{ }`。
- **关键字**：共 56 个，十一类（完整表见 §9）。

---

## 6. 名称 · 宣言 · 一句话定位

**名称**

- **AILang**
- 全称：**A**rtificial **I**ntelligence Native **Lang**uage
- 定位：面向 AI 原生时代设计的高性能系统级编程语言

**一句话定位**

> AILang 是第一代为人工智能理解、生成和运行而设计的系统级编程语言。

**AILang 宣言**

> **AILang 是为 AI 原生时代设计的系统级编程语言。**
>
> 它相信代码应该表达意图，而不仅是指令；
> 类型应该表达意义，而不仅是数据；
> 程序行为应该透明，而不是隐藏；
> 性能、安全与 AI 理解能力不应该互相牺牲。
>
> AILang 的目标，是成为人类与人工智能之间的新一代软件表达语言。

<a id="part-i"></a>
# Part I · 语言规范

## 7. 语法层设计原则

> Python 的简洁表达 + Rust 的安全模型 + TypeScript 的类型表达 + C/Go 的代码结构 + AI 原生语义。

- **默认安全**：编译期确定类型，禁止隐式数值/类型转换 / 隐式空值 / 运行时类型错误（唯一例外：字符串拼接中的 `Display` 字符串化，见 §11）。
- **默认不可变**。
- **类型不仅表示「是什么」，还表示「意义」**。
- **类型在公共边界强制声明**（函数参数/返回必须显式；局部可推断）。
- **AI 信息进入语言结构**（meaning / effects / requires / ensures / agent / tool）。

---

## 8. 词法约定

### 8.1 文件与扩展名
源文件 `.ail`（UTF-8）；元数据产物 `.ailmeta`。

### 8.2 代码块
统一 `{ }`，不使用强制缩进。

### 8.3 分号（可选）
Go 风格：换行分隔语句，分号可省。`let x = 1` 与 `let x = 1;` 等价。

### 8.4 隐式行连接
`() [] {}` 内换行忽略。

### 8.5 注释
`//` 行注释；`/* */` 块注释（可嵌套）；`///` 文档注释——附着到下一个声明，进 `.ailmeta`：**type 上为 description**（`meaning` 声明权威、冲突 `meaning` 胜），**function / param / field 为其 meaning / 描述来源**（§92 #8 双轨优先级）。

### 8.6 标识符与字面量
- 标识符：ASCII `[A-Za-z_][A-Za-z0-9_]*`（v0.2）。
- 整数字面量无类型后缀，按上下文推断/强制。

### 8.7 命名规范（官方风格）
| 实体 | 风格 | 示例 |
|------|------|------|
| 变量、函数 | camelCase | `userName`、`get_user` |
| 类型（struct/enum/trait/interface/actor） | PascalCase | `UserAccount`、`HttpClient` |
| 常量（`const`） | UPPER_CASE | `MAX_SIZE`、`VERSION` |

> 风格不强制（编译器不报错），但 `ail` 提供 `ail fmt` 检查。

---

## 9. 关键字（共 56）

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

> 类型 `Optional` `Result` `List` `Map` `Set` `Array` `Tuple` `Channel` `Shared` `Mutex` `raw_pointer`、字面量/类型词 `true`/`false`/`void`/`never` 不计入关键字（属标准库/保留字面量·类型词）。精确计数见 §83 #5。

基础 56 个（§9 表）。`true` / `false`（布尔字面量）、`void`（unit 类型）、`never`（bottom 类型，§89 #4）为**保留字面量/类型词，不计入 56**（§83 #5 已锁定）。`borrow_mut` 计为单关键字。`channel` 计入 56 为**保留关键字**：v0.2 通道经 `Channel<T>` 类型构造（§21.4），关键字预留以备未来声明式语法，当前无独立产生式（§27）。

**上下文关键字**（仅特定语法位置保留、词法层产出 `Ident`、不入 56）：`where`（泛型 bound 子句，§15.10）、`constraint`（§15.4 声明引导词；与硬关键字 `meaning` 分类不同，为已知非对称）、`on`/`reply`/`detached`（§21 / §87）、`default`/`timeout`（select）、`reduce`（§87 #9）。另：`lock`/`yield`/`spawn_blocking` 为 `std.sync`/`std.async` 内建函数/控制构造（**非关键字**，§21 / §87 #7）。

---

## 10. 基础语法规则

1. 代码块用 `{ }`（§8.2）
2. 分号可选（§8.3）
3. 类型写法：`name: type`（小写，`string` 非 `String`）
4. 函数返回后置：`fn xxx() -> Type { }`
5. 默认不可变：`let`；可变 `var`；编译期常量 `const`
6. 类型在公共边界强制声明：函数参数与返回必须显式；局部可推断

## 11. 运算符与优先级（低 → 高）

| 级 | 运算符 | 说明 |
|----|--------|------|
| 0 | `=` `+=` `-=` | 赋值（语句级，右结合）|
| 1 | `||` | 逻辑或 |
| 2 | `&&` | 逻辑与 |
| 3 | `==` `!=` `<` `>` `<=` `>=` | 比较 |
| 4 | `+` `-` | 加减 |
| 5 | `*` `/` `%` | 乘除模 |
| 6 | `-`（负）`!` `borrow` `borrow_mut` `move` `copy` `await` `try` | 一元前缀（§87 #3 / §89 #1） |
| 7 | `()` `[]` `.` | 后缀：调用 / 下标 / 字段 |
| 8 | 字面量 / 标识符 / `( expr )` / 结构体构造 | 基本 |

> 逻辑运算符为符号 `&&` `||` `!`，与 C/Go/Rust 一致（v0.2.1 锁定）。
>
> **运算符重载 = trait 解糖**（§86 #8）：运算符解糖为 std.core 运算符 trait 方法调用（`a + b` → `Add.add(a, b)`、`a == b` → `Eq.eq(a, b)`、`a < b` → `Ord.lt(a, b)`）。泛型 bound 复用（`where { Ord<T> }`，§15.10）。仅 `&&`/`||`/`!` 为内置短路，不可重载。

**字符串拼接（v0.2.1 锁定）**：`+` 重载用于字符串——
- `string + string` → 拼接，产生新 `string`（`string` 为 COW：非 bitwise Copy，拼接必分配新缓冲；§18.4、§85 #7）。
- `string + X` / `X + string`，其中 `X` 实现 `Display` → `X` 经 `Display` 字符串化后拼接（覆盖基础类型、以其为 base 的语义类型、显式实现 `Display` 的类型）。
- 这是 AILang **唯一**允许的隐式转换（无损显示），其余转换必须显式。

```ail
let s = "id: " + userId        // UserId(int) 经 Display 字符串化
let t = "a" + "b"              // "ab"
```

---

## 12. 模块系统（§91 #3–#8）

### 12.1 package 声明 + Go 式包模型
`package <dotted-path>` 声明当前文件的**完整模块路径**：根包单段（`package app`），嵌套点分（`package std.http` / `package app.utils`）。**同一 `package` 声明可跨多文件共享命名空间**（Go 式——包内文件互不需 import）；**文件路径须与声明的模块路径一致**（编译期校验：`src/http/request.ail` 须声明 `package *.http.request`）。根包名 = `ail.toml.name`（§91 #10）。

```ail
package app                       // 根包
import std.http                   // 标准库：全路径，std. 前缀必需（§91 #4）
import std.database
from std.database import Client   // 仅可导入 std.database 的 public item（§91 #6）
import std.json as j              // 别名（§91 #5）
```

> **`std.core` 自动加载**（免 import，§84 #9）；其余 `std.*` 显式 import。第三方库 `import <pkg>.<module>`（依赖包名作根前缀，§91 #10）。

### 12.2 导入规则（§91 #5/#6）
- **全路径 + 禁通配**：导入路径必含根包；**禁 `import *`**（保 `.ailmeta`/AI 可追溯）。
- **冲突**：两导入同名 = **编译错误**（须 `as` 消歧）；**本地定义优先于导入**（遮蔽 + warn）。
- **`from M import name`**：仅可导入 M 的 **public 顶级 item**；导入 private = 编译错误。

### 12.3 再导出 / 循环导入（§91 #7/#8）
- **v0.2 不支持再导出**（无 `public import` / `pub use`）：`import a` 仅得 `a` 自身 public item；要 `b` 的 item 须显式 `import b`。
- **循环导入**：类型级（签名互引）OK；值级（fn 体跨模块调用）OK（惰性绑定）；**顶层初始化循环 = 编译错误**（`const` / 全局求值成环）。须两遍分析（先签名，再体）。

可见性见 §22（默认 `private`）。

---

## 13. 变量

```ail
let age: int = 18              // 不可变
var count: int = 0             // 可变
const VERSION: string = "1.0"  // 编译期常量
```

---

## 14. 函数

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

> `effects` 取值遵循统一 `<domain>.<verb>` 词表（权威枚举见 §19 效果系统；§88 #6）。

---

## 15. 类型系统

> AILang 的类型 = 数据结构 + 语义 + 约束 + 行为。

### 15.1 基础类型
`int`（默认 i64）`uint`（默认 u64）`float`（默认 f64）`bool` `string` `byte` `void`。显式宽度：`int8..int64`、`uint8..uint64`、`float32`/`float64`（§90 #2）。

> **数值语义（§90 #1/#2/#3）**：整数算术溢出 = **定义行为**（永不 UB）——release **二补数回绕**（规范），debug **溢出 → panic `ArithmeticOverflow`**（开发期检测层）；整数**除零 / 取模零 → panic `DivideByZero`**（确定性）；浮点遵循 **IEEE 754**（`x / 0.0`（`x≠0`）→ `±inf`、`0.0 / 0.0` → `NaN`、NaN 经运算传播、`NaN == NaN` 为 `false`，**浮点运算永不 panic**）。显式运算符族（`wrapping_*`/`checked_*`/`saturating_*`）/ 安全除法 `checked_div` 见 `std.math`（§33.1）；浮点 NaN 检测（`is_nan`/`is_inf`/`is_finite`）为 **float 方法** `x.is_nan()`（非自由函数，§94 #4）。

> **`never`（⊥，bottom）**：空类型，内建 lang item（§89 #4）——无值，表示「不返回」（`throw`/`return`/`loop` 表达式类型为 `never`）。统一规则 `never | T = T`：穷尽检查中 `throw`/`return` 的 match arm 不占变体。`never` ≠ `void`（`void`=unit 有值；`never` 无值）。可作返回类型 `fn fail() -> never`。`never` 为保留类型词，不计 56（同 `void`/`true`/`false`，§9）。

### 15.2 类型推断
局部可推断；**函数参数与返回必须显式**（公共边界）。泛型 `T` 在参数中可由实参推断；**return-only 泛型**（`fn empty<T>() -> List<T>`）须调用点显式 `empty<int>()`（§86 #7）。

### 15.3 Semantic Type（名义新类型）与 Type Alias（透明别名）

`type X = Y` 按 **§92 #1** 分两类：

- **无 `meaning` / `constraint` → 透明别名（`kind: alias`）**：编译期展开，`X` 与 `Y` 同型、可互转（如 `type URL = string`、`type Row = ...`，§92 #10）。
- **附 `meaning` 或 `constraint` → 名义 semantic（`kind: semantic`）**：与 base 不同型，底层共享 → 零成本（constraint 附着见 §15.4）：
```ail
type UserId = int
meaning UserId:
    "unique identifier of a user"
```
整数字面量可强制到整型语义类型（`let u: UserId = 100`，§84 #1）；变量间不隐式转换（nominal 不变，§86 #9）。

> **运算 / 标记继承（§92 #7，marker vs operational 二分）**：semantic 新类型对 base 的 trait 继承按两类拆分——
> - **标记 trait（`Copy` / `Send` / `Sync`）自动继承 base**：`type UserId = int`（附 meaning）保持 `Copy + Send + Sync`（与 base 同的位级与线程安全类别）。
> - **运算 trait（`Add` / `Sub` / `Mul` / `Eq` / `Ord` / ...）不自动继承**（类 Rust newtype），须显式实现；`==` 须显式实现 `Eq`。
> - **唯一例外：`Display`**（字符串化自动经 base，§11）+ 字面量强制（`let u: UserId = 100`，§15.3 / §84 #1）。
>
> **实现机制（v0.2.1 gap）**：v0.2.1 仅有内联 `struct X implements Trait { }`（§15.5），尚无给语义新类型（`type UserId = int`）实现运算 trait 的语法——运算 trait 继承为 **v0.3 计划**（独立 `impl Trait for Type { }` 块 + orphan 规则下放开外部类型，§83 #2 / §92 #7）。
>
> **meaning 传播（§92 #3）**：值为带 meaning 的 semantic → 在 `.ailmeta` 各处（input / output / field）携带此 meaning；函数 output.meaning = 返回类型 meaning（函数级不可另写，描述用 `///`→identity.description）；bare 基础类型 / alias = null。

### 15.4 Constraint Type（semantic + 约束；§92 #6）

Constraint Type = **Semantic Type + `constraint`**（特化，非独立 kind；`.ailmeta` 仍 `kind: semantic` + `constraint` 字段）：
```ail
type Age = int
constraint Age:
{
    value >= 0
    value <= 150
}
```
- **求值分级**：字面量编译期检查（`let a: Age = 30` 折叠后判定；`let a: Age = -5` → 编译错）；运行期构造插断言。完整 refinement 证明不在 v0.2（§83 #3）。
- **构造算符 `T(expr)`（§92 #2）**：运行期值经 `let a: Age = Age(n)` 构造——运行期断言谓词，违约 panic `ConstraintViolation`（§92 #9）；字面量走编译期分支。nominal 不变（§86 #9）使 `let a: Age = n`（n 为 int 变量）类型错，**须 `Age(n)`**。
- **谓词语言（§92 #4）**：`value` 魔法只读绑定 + 纯布尔表达式（可 `value.field` / `value.method()`、跨字段须约束 struct 引用字段名、可调 `pure fn`）；禁 `old()` / 副作用 / IO（`old()` 属 `ensures`）；谓词须 pure。

### 15.5 Struct（含 trait 实现 + 方法）
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

**接收者模式**（§85 #1，复用形参关键字）：`borrow self`（只读，**默认**，可省）/ `borrow_mut self`（可变，原地改）/ `self`（消费）。可变方法（如集合 `insert`/`append`）须声明 `borrow_mut self`；`release` 须声明 `self`（§18.7）。**trait 默认方法**（§86 #6）：trait 方法带 `{ body }` = 默认实现，struct 同名方法覆写，`implements Trait { }` 可留空继承默认。栈布局、无对象头、无追踪 GC。**trait 实现内联于 struct 定义**（v0.2.1 锁定，§83 #2）。

### 15.6 Enum
```ail
enum Status {              // 无 payload variant（标签）
    Active
    Disabled
    Deleted
}

enum Expr {                // 带 payload variant（代数和，类 Rust）
    Num(int)
    Add(Box<Expr>, Box<Expr>)      // 递归须 Box<T> 堆间接（§86 #4）
    Leaf
}
```
内存：tag + data（类 Rust）。variant 可携带 payload；`match` 解构并绑定（见 §16）。**递归类型须 `Box<T>` 间接**（owned 堆，Move/非 Copy；§86 #4），否则无限大小。

### 15.7 Optional\<T>（禁止 null）
```ail
fn find_user(id: UserId) -> Optional<User> { ... }   // Some / None
```

### 15.8 Result\<T, E>（错误安全核心）
```ail
fn read_file(path: string) -> Result<string, FileError> { ... }
```

> **lang item**（§86 #5）：`Optional`/`Result`（及 `Ok`/`Err`/`Some`/`None`、`List`/`Map`/`string` 等）为 **`#[lang]`** 标注的 std.core 类型——既是库类型，又被编译器特判（errors 派生 §17、`try/catch` 解糖 §83 #4、特权 `match` arm）。

### 15.9 复合类型
`List<T>` `Array<T, const N>` `Map<K, V>` `Set<T>` `Tuple`。均为标准库类型，非关键字。`Array<T, const N: int>` 的 `N` 为 **const 泛型**（编译期整数，固定内联布局，§86 #3、§25.4）。

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

### 15.10 泛型（约束用 `where { }`）
```ail
fn sort<T>(list: List<T>) -> List<T>
where {                                     // trait bound（编译期，单态化；§86 #2）
    Ord<T>
}
{
    ...
}
```
泛型构造器**不变**（`List<UserId>` ≠ `List<int>`，§86 #9）。运算符 bound 复用 std 运算符 trait（§11、§86 #8）。

### 15.11 Interface 与 Trait
- **`interface` = 服务能力**（外部 API 抽象，**dyn 派发**）。方法**必须携带 `effects`**（§85 #4）；**方法不得泛型**（dyn vtable 无法单态化，§86 #1）：
  ```ail
  interface Database {
      fn query(borrow self, sql: string) -> Result<List<Row>, Error>   // 非泛型；Row = 动态行
      effects { database.read }
      fn exec(borrow self, sql: string) -> Result<void, Error>          // 写操作（UPDATE/INSERT/DELETE）
      effects { database.write }
  }
  // typed 解码由单态化自由函数完成：fn decode<T>(rows: List<Row>) -> Result<List<T>, Error>
  ```
- **`trait` = 类型能力**（可带默认实现，作泛型约束）：
  ```ail
  trait Serialize {
      fn serialize(borrow self) -> string      // 接收者模式同 §15.5
  }
  ```
- 类型用 **`implements`**（内联）获得 trait 能力（§15.5）。

> **interface 值的所有权**（§85 #6）：interface 类型值（`Database`/`Model` 等）为**不可 Copy 的 Move 句柄**（vtable + 指针），经 `borrow` 借用、`Shared<T>`/连接池共享。
>
> **trait vs interface 类型层二分**（§86 #10）：`trait` 作泛型 bound → **编译期单态化**；`interface` → **运行期 dyn 派发**。此二分即 interface 禁泛型方法（#1）的根因。
>
> v0.2.1：`implements` 是唯一的 trait 实现语法；无独立 `impl T for X` 块（§83 #2）。`interface` 与 `trait` 是否都需要见 §83 #1。
>
> **orphan 规则（§91 #9）**：`Trait` 或 `Type` 之一须定义于当前 package——禁止在 `app` 里为 `std.http.Request` impl 自定义 trait（除非 trait 或类型之一属本 package）；`public` 类型 impl 本 package `public` trait 可跨模块可见（类 Rust coherence）。

---

## 16. 控制流

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
    Expr.Add: { l, r -> print(l); print(r) }
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

## 17. 错误模型

核心 `Result<T, E>`；`error` 定义错误枚举。

```ail
error UserError {
    NotFound
    Forbidden
}

fn login() -> Result<Token, UserError> { ... }
```

**`errors[]` 单一真源（§89 #10）**：函数 `errors` ≡ 签名 `E`（`error` 枚举）的变体集，进 `.ailmeta`；`throw`/`try` **永不越界、永不增补**。非 `Result` 返回 → `errors:[]`；`E` 为 `std.core.Error`（动态边界 opt-in 通用型，§84 #6，非命名 `error` 枚举）→ `errors[]:["Error"]`（类型名占位，变体不展开）。

**变体值构造（§89 #2）**：`Ok(x)` / `Err` 变体构造**复用 call 文法**（`Variant(args)`），由名字解析消歧（变体 vs 函数）；裸变体 = `ident`。**成功返回须显式 `Ok(x)`，无 auto-wrap**（呼应 §11 禁隐式转换）；`Result<void, E>` 成功构造 = `return Ok(void)`（无 `()` 字面）。

**`try`/`catch`/`throw` = `Result` + `match` 的语法糖**（精确解糖 §83 #4；§89 #1/#5/#6）：

```ail
fn load() -> Result<Data, UserError> {
    let raw = try read()                 // try expr：前缀一元（同 await）；unwrap-or-propagate，要求同错误类型
    let v = match parse(raw) {           // 跨类型转换须 match + throw（无隐式 From）
        Ok: { d -> d }
        ParseError.Bad: { throw UserError.Forbidden }   // 命名 → 命名跨型转换（match + throw，§89 #5）
    }
    return Ok(v)                         // 显式 Ok，无 auto-wrap
}
```

- `throw V`：**语句级**控制流，类型 `never`（§15.1、§89 #4），可与任意类型统一（match arm/分支），不可作子表达式（§85 #10）。**合法性（§89 #6）**：仅当所在函数/处理子返回 `Result<T,E>` 且 `V ∈ E`；`void`/`task`/actor `on`/server handler 体**禁 throw**（错误经各自通道，§21、§89 #8）。
- `try expr`：**前缀一元**（§27 unary，与 `await` 一致）；`expr: Result<T,E1>` 在外层 `Result<U,E2>` 且 `E1≠E2` 时**类型错误**——**无隐式 `From`**（§11），跨类型转换唯一手段 = `match { 错误分支 → throw 新变体 }`（边界 `Error → 命名错误`，§84 #6）。
- `try { body } catch { arms }`：局部捕获（§27 `try_catch`）；未匹配且无 `_` → 重新传播。
- **`error` 变体可带 payload**（§89 #7，与 enum 同形 `Variant(field: T)`）：`throw NotFound(404)` / match arm 绑定同 §16；`.ailmeta` `errors[]` 每项可选 `payload:[{name,type}]`（§24）。惯法推荐裸变体。
- **`pure` 与 throw（§89 #9）**：`pure fn` 可返回 `Result` 并 `throw`/`try`——throw 是控制流（早返 Err），**非 effect、不改入参**；`.ailmeta` `is_pure` 仅看 effect 集与入参突变，与 errors 无关（§19）。

**不可恢复错误：panic（§89 #3）**：`unwrap`/`assert` 失败或显式 `std.core.panic(msg)` 触发 **panic = 确定性栈展开 + Drop/release**（与确定性释放一致），展开至最近的 **Task 边界或 `extern` 帧**——Task 内 panic 终止当前 Task（→ TaskHandle 错误态 / actor 死信），不传染进程；**触达 `extern` 帧 → 转进程 abort**（FFI 边界为「不传染进程」唯一例外，§90 #4）；v0.2 无 `catch_unwind`（仅 Task 边界可观测）。已知 panic 原因：`unwrap`/`assert`、整数溢出（debug，`ArithmeticOverflow`，§90 #1）、整数除零 / 取模零（`DivideByZero`，§90 #3）、越界访问（`IndexOutOfBounds`）、约束违约（`ConstraintViolation`，§15.4 / §92 #9）。`panic` 为 std 函数，非关键字。

> **`std.core.Error`**：动态边界（DB 驱动 / FFI / 泛型 Task）的 opt-in 通用错误类型；惯用代码用命名 `error` 枚举以保 `.ailmeta` 精度，边界处显式 `Error → 命名错误` 映射（§84 #6）。

---

## 18. 所有权 · 借用 · 内存模型

### 18.1 核心理念 + 内存区域 + 栈堆规则

#### 核心理念

| 范式 | 管理者 | 问题 |
|------|--------|------|
| C/C++ | 程序员（`malloc`/`free`/pointer） | 内存泄漏、野指针、double free |
| Java/Python | GC | GC 暂停、内存不可控、性能波动 |
| Rust | Ownership（编译期） | 生命周期语法复杂、学习成本高 |
| **AILang** | **Ownership + Automatic Lifetime Analysis** | **隐藏复杂生命周期语法** |

> AILang 的取舍：保留 Rust 的编译期内存安全与**无追踪 GC**（详见 §18.4），但把生命周期分析**完全交给编译器内部**，不暴露 `'a`。

#### 内存区域

```
Memory
├── Stack      函数帧、小型固定数据
├── Heap       复杂动态对象
├── Static     全局/编译期常量
└── Resource   文件/套接字/连接等需显式生命周期的外部资源
```

#### Stack 规则

小型、固定大小、Copy 语义的数据自动入栈：

```ail
let age: int = 20      // 栈上，main 帧
```

函数返回时栈帧自动释放——零成本、无跟踪。

#### Heap 规则

复杂/动态对象入堆，栈上持有指针：

```ail
let user = User { name: "Tom" }
// 栈：user (指针)  →  堆：User 对象
```

堆对象由**所有者**管理；所有者离开作用域时释放（见 §18.7）。

### 18.2 Ownership 核心规则

#### Rule 1：每个资源只有一个 Owner
```ail
let file = File.open("a.txt")
// Owner = main 作用域
```

#### Rule 2：默认 Move
```ail
process(file)          // 等价 process(move file)
print(file)            // ❌ AIL2001 Use after move
                       //    Resource: file  /  Moved to: process()
```

> 错误码方案：`AIL2xxx` = 所有权/内存类（如 `AIL2001 Use after move`）。统一错误码体系（`AIL2xxx` 编号空间）待后续设计阶段细化。

### 18.3 默认 move
```ail
let file = File.open("a.txt")
process(file)       // move
print(file)         // ❌ AIL2001 Use after move
```
显式：`borrow`（只读）/ `borrow_mut`（可写独占）/ `move` / `copy`（Copy 与 COW 类型；在 COW 上强制克隆——对齐 §18.4 / §85 #3）。

> 调用点显式标注**可选**（§84 #13）：省略时按**被调形参模式**推断（形参 `borrow T` → 实参自动借用、不 move）；显式形式用于澄清或消歧。

### 18.4 值语义三分类（v0.2.1 锁定）

| 类别 | 类型 | 行为 |
|------|------|------|
| **位复制 Copy** | `int` `uint` `float` `bool` `byte` + 全 Copy/COW 字段 struct | 赋值即位复制，原值仍可用 |
| **COW 值类型** | `string` `List` `Map` `Set` | 值语义：赋值=共享缓冲（**原子引用计数**），修改=克隆（写时复制）；**为 `Send`**（§85 #2）；行为同 Copy，无 `&str`/`String` 双类型 |
| **Move（独占资源）** | `File` `Socket` `Connection` 等 | 默认 move，使用须显式 `borrow`/`borrow_mut`（Move-only 不可 `copy`） |

> `string`/集合采用 **COW 值语义**（类 Swift 数据模型）：Python 式廉价传递 + 值语义安全，**无追踪 GC**（COW/RC 引用计数仍存，确定性释放；§85 #2、§83 #7）。资源型 move-only。COW 字段计为 Copy-compatible → 含 COW 字段的全 Copy-like struct 为 Copy（§85 #7）。
>
> **`Array<T, N>` 与 `Tuple` 值语义（§87 #6）**：值类型聚合——`Copy` 当且仅当所有元素 / 字段为 `Copy`（含 COW 计 Copy-compatible），否则整聚合 `Move`（任一 Move/`Resource` 元素 → 整聚合 Move）。

```ail
let a = 10
let b = a             // Copy：a、b 都可用
let s1 = "hello"      // string 为 COW 值类型
let s2 = s1           // COW：s1、s2 共享缓冲均可用，改 s2 才克隆
```

显式控制：`move` / `copy`（Copy 与 COW 类型）/ `borrow` / `borrow_mut`。

### 18.5 生命周期（推导，不暴露 `'a`）+ borrow 不逃逸

AILang 不使用 Rust 的 `&` `&mut` `'a`，改用更自然的前缀：

```ail
fn print_user(user: borrow User) -> void { ... }       // 只读借用
fn update(user: borrow_mut User) { ... }               // 可写借用（独占）
```

- **`borrow x`**：只读，可同时多个；存在期间原值冻结。
- **`borrow_mut x`**：独占，同时只能一个，且不能与任何只读借用并存。
- **`borrow_mut` 与 COW**（§85 #3）：对 COW 类型，`borrow_mut` 要求**唯一所有权（rc == 1）**——共享值须先 `copy`/克隆；其下突变**原地**进行（独占 → 无需克隆），即 opt-out COW。
- **不逃逸**：禁止存入 struct 字段、禁止返回、禁止跨 `await` 存活 → 生命周期恒等于当前语句 → **无需 `'a` 标注**。

用户**永不**写生命周期参数：

```ail
// Rust：fn get<'a>(...) -> &'a T
// AILang：
fn length(text: string) -> int {
    return text.length()       // 编译器内部建 Lifetime Graph，判定 text 生命周期 = 函数作用域
}
```

编译器内部的 Lifetime Analyzer 自动推导；推导失败时报清晰错误（而非要求用户加 `'a`）。详见 §74.1（所有权状态机）。

### 18.6 线程安全 + 数据竞争 + 并发安全类型

跨线程移动/共享自动分析（`Send`/`Sync` 为编译器自动派生的 marker trait，§87 #6）；COW 类型 `Send`（原子 rc，§85 #2）。**组合规则（§87 #6）**：聚合类型 `Send` 当且仅当所有字段 `Send`；`Sync` 当且仅当所有字段 `Sync`；所有 `Copy` → `Send + Sync`；Move 资源（`File`/`Socket`）→ `Send`、非 `Sync`；`Mutex<T>` → `Send + Sync`（`T: Send`）；`Shared<T>` → `Send + Sync`（`T: Sync`）；`TaskHandle`/`ActorHandle`/`Channel<T>` → `Send`（`T: Send`）。`spawn` / `Channel.send` / actor 投递边界静态查 `Send`。

跨线程访问由编译器静态检查（类 Rust `Send + Sync`；AILang 自有 auto-trait，§87 #6、§34）：

```ail
spawn worker(data)      // 编译器检查 data 是否可跨线程移动
```

线程 A 写 `data`、线程 B 读 `data` 且无保护 → **拒绝编译**。

| 类型 | 语义 |
|------|------|
| `Shared<T>` | 多线程**只读**共享 |
| `Mutex<T>` | 多线程**可变**共享，访问须加锁 |

```ail
let config: Shared<Config> = ...
let data = Mutex<Data>(...)
lock(data) {           // 临界区
    update()
}
```

完整并发语义见 §21。

### 18.7 资源确定性释放 + Resource trait

所有者离开作用域时自动 `release`（消费 owner，类 Rust `Drop`，§85 #8）。

```ail
fn read() {
    let file = File.open("a.txt")
    read(file)
}   // ← 编译器在此自动插入 file.release()
```

无需手动 `close`、无延迟。这是确定性析构安全管理的根因。

> **Drop 语义精确化（§90 #5）**：① 字段 **Drop 顺序 = 声明顺序**（自上而下，确定性，与 Rust 一致）；② 安全代码**结构上无循环引用**（单一所有者 + COW rc 为缓冲共享、非对象图共享）→ **v0.2 不引入 `Weak`/`Rc`/`Arc`**，需共享对象用 Actor / `Mutex<T>` / `Shared<T>`（§21）；③ panic 展开时 Drop 同序执行（§89 #3）；④ 自定义释放经 `Resource.release(self)`（消费 owner）。递归类型须 `Box<T>`（§15.6）。

所有需显式释放的资源实现：

```ail
trait Resource {
    fn release(self)        // 消费 self（§85 #8）
}
```

`File` / `Socket` / `DatabaseConnection` 等实现之；编译器在 owner 作用域末插入 `release`（消费 owner，§85 #8）。

### 18.8 零成本抽象 + 模型总览

高级语法不牺牲性能：
```ail
for item in list { ... }
// 编译优化为指针循环，无迭代器对象开销
```

`Semantic Type`（`type UserId = int` + `meaning UserId: "用户标识"`——nominal 语义新类型，区别于透明别名 `type Alias = int`）、`Optional<T>`、`Result<T,E>` 等均零成本展开（§77 类型映射）。

```
AILang Code
    │
    ▼
Ownership Analyzer
    │
    ├──► Stack      （小型固定数据，自动）
    ├──► Heap       （复杂对象，所有者管理）
    ├──► Static     （全局/编译期常量）
    └──► Resource   （文件/连接，确定性释放）
    │
    ▼
LLVM Native Code
```

## 19. 效果系统

AI 生成代码最大的风险不是「不会写」，而是「不知道代码会产生什么影响」。AILang 让每个函数的能力边界可被静态分析：

```ail
fn save_user(user: User) -> Result<void, Error>
effects {
    database.write
    network
}
{ ... }
```
点分粒度（`database.read` `filesystem.write` `network` `alloc`）。**推断为主，标注为约束**（推断集 ⊆ 标注集）。`pure` = **无任何效果（含 `alloc`）且不修改入参**（§85 #9）。`.ailmeta` 输出推断集；`alloc` 为**隐含效果**——堆分配普遍存在、无信息量，故 `alloc` 仅参与 `pure` 判定（堆分配 → 非 pure），**不进 effects 标注与 `.ailmeta` 数组**（数组只记 I/O·状态·并发等行为效果）。

> **`is_pure` 与 errors 无关（§89 #9）**：`pure fn` 可返回 `Result` 并 `throw`/`try`——throw 是控制流（早返 Err），非 effect、不改入参；`is_pure` 仅看 effect 集 + 入参突变，与 `errors[]` 正交。

**Effect 词表（权威，`<domain>.<verb>`，§88 #6）**：

| effect | 语义 |
|--------|------|
| `database.read` / `database.write` | 数据库读 / 写 |
| `network` | 网络（别名 `network.read` / `network.write` 细化读写） |
| `io.read` / `io.write` | 标准流 / 控制台读写（`print` → `io.write`） |
| `io.blocking` | 同步阻塞（`fs.read` 同步版；`async fn` 禁，§87 #7） |
| `filesystem.read` / `filesystem.write` | 文件系统读 / 写 |
| `alloc` | 堆分配 |
| `extern` | FFI（`extern` 块默认上界，§85 #4） |
| `unsafe` | `unsafe` 块（逃生舱，§25） |

`pure` = **空 effect 集** + 不改入参。新效果须遵循 `<domain>.<verb>` 命名；跨文档统一用本表枚举值。

> **健全边界**（§85 #4）：① **interface 方法必须声明 `effects`**（§15.11），dyn 调用效果取声明上界；② **`extern`（FFI）默认 `effects { extern }`**（保守上界，可收窄）；③ 泛型方法效果随 trait bound 传播。如此「推断集 ⊆ 标注集」对 dyn/FFI/泛型可强制。

---

## 20. 契约（前置 / 后置条件）

```ail
fn withdraw(db: borrow Database, account: borrow_mut Account, amount: int) -> Result<void, Error>
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
v0.2：运行期断言。自动证明不在范围（§83 #3）。`account` 取 `borrow_mut`（§85 #5）使 `ensures` 对 caller 可观测；`old(e)` 为**调用前快照**；函数先突变内存 `account.balance` 再 `database.write` 持久化，内存后态与持久化一致。

> **`requires`/`ensures` 仅装运行期断言**；泛型 trait bound 用 `where { }`（§86 #2）。

---

## 21. 并发模型

> 完整模型见 §21.1–§21.13。关键字：`task` `spawn` `async` `await` `channel` `select` `actor` `message` `parallel` `server` `route` `cancel`。

```ail
task download() { fetch() }
let h = spawn download()              // h: TaskHandle（§87 #4）；编译器检查 Send + 生命周期

let ch = Channel<int>()
ch.send(10)
let v = ch.receive()                    // Optional<int>（阻塞至有值 Some / 关闭耗尽 None，§87 #2）

select {                                      // 多路等待（类 Go select，§87 #10）
    ch1.receive() { v -> handle1(v) }         // v: Optional<T>（Some=值 / None=关闭）；绑定名 -> body（同 match §16）
    ch2.receive() { handle2() }               // 无需值则省略绑定
    timeout(100ms) { give_up() }              // 超时分支（上下文关键字）
}

actor UserService {                   // 独立状态 + 消息入口，AI 友好
    state:
        users: Map<UserId, User>
    message AddUser { user: User }
    on AddUser { msg ->               // 消息处理（§87 #5），self.state 单 Task 串行
        self.state.users.insert(msg.user.id, msg.user)
    }
}

parallel for item in data {           // CPU 并行，自动分片/合并（§87 #9：表达式，返回 List<R>）
    process(item)
}
let squares: List<int> = parallel for x in data { x * x }   // body 须返回 R
let sum = parallel reduce((a, b) -> a + b, data) { x -> x } // op 须结合律

cancel h                            // h: TaskHandle；结构化取消，向子 Task 传播（§87 #4/#8）
```

- **默认禁止跨 Task 共享可变**（`var` 跨 spawn → 拒绝）。
- 安全共享：`Shared<T>`（只读）/ `Mutex<T>` + `lock(data) { }`（§18.6）；`lock` 为 `std.sync` 内建控制构造（**非关键字**），块语法规避 v0.2 无闭包依赖，待闭包落地后可改方法式 `m.with_lock(...)`。
- **`spawn` 作用域语义（§87 #4）**：三形式 `spawn task(args)`（具名 Task）/ `spawn actor Name(args)`（Actor，返 `ActorHandle`，§87 #5）/ `spawn { ... }`（匿名块）；具名/匿名返 `TaskHandle`。**默认 scoped**——task 附属 spawn 所在作用域，作用域末**隐式 join**（父等待子完成）；**显式分离用 `spawn detached`**（`detached` 为上下文关键字，不入 56）。**非** fire-and-forget（仍返回 `TaskHandle`、`cancel` 可达；§87 #4）。
- **`task` 与 `async fn` 区分**（§87 #1）：`task` = void body + CSP 通道（不可 async），经 Channel 投递消息 / `cancel` 通信、**不强制 `await`**；`async fn f() -> T` 的 `T` 即 `await` 产出值（调用得 `Future<T>` lang item，`await f()` 得 `T`）。**「须返回 Result」仅约束用户代码显式 `await` 的 async fn**；server route 等 runtime 派发 handler 豁免（返回声明类型即可）。**无 `Pin`**：borrow 禁跨 await（§18.5 / §74.1）→ 状态机不自引用。
- **并发错误传递**（§89 #8 统一表）：`throw` 仅合法于返回 `Result<T,E>` 的函数体（§17、§89 #6）——`async fn -> Result<T,E>` 可 `throw`/`try`（错误随 `await` 传）；`task`（void 体）/ actor `on` handler / server handler **禁 throw**，分别经 Channel 投递错误消息或 `cancel`、`reply(Err...)`、runtime 派发传递。完整表见 §21.9。
- **Channel**（§87 #2）：**MPMC**；无界 `Channel<T>()` / 有界 `Channel<T>(cap)`（`cap=0` rendezvous）；`ch.close()`；`receive() -> Optional<T>`（阻塞至有值得 `Some(v)`；关闭且耗尽得 `None`）、`try_receive() -> Optional<T>`（非阻塞；`None` = 空或关闭耗尽）；两者仅「open 且空」时不同（前者阻塞、后者立即 `None`）；select 的 receive 分支在关闭时立即就绪并交付 `None`（§87 #10）。
- **控制构造块捕获**（v0.2 无一等闭包）：`spawn { }` / `parallel for` / `lock(m) { }` / `server route` 的块体可捕获外层**不可变 / `Send`** 变量（**不含 `borrow`**——借用不逃逸，§18.5）；`Channel` 为共享 `Send` 句柄，跨 spawn 克隆（§84 #7、§87 #6）。
- **`await` 为一元前缀**（§87 #3）：`await f()` / `await fut.field`。
- **Actor**（§87 #5）：`actor { state; message; on Msg { msg -> body } }`——`on` 为消息处理子句；`spawn actor Name(...)` 返回 `ActorHandle`；`h.send(Msg{...})` / `h ! Msg{...}` 投递；状态经 `self.state` 串行访问（无需锁）。**`reply`**（HIGH#5）：`on` 子句内的**上下文关键字**——`reply(expr)` 为该 handler 的值/错误出口（`reply` 值确定 message 的响应类型，可为 `Result<T, E>`）；actor `on` handler **禁 `throw`**，错误经 `reply(Err...)` 传递（§89 #8）。
- **select 扩展**（§87 #10）：支持 `receive`/`send`/`await` 分支 + `default { }`（非阻塞）+ `timeout(d) { }`；多分支就绪**随机公平**择一；关闭 channel 的 receive 立即可就绪（交付 `None`）。
- **parallel for / reduce**（§87 #9）：`parallel for x in xs { f(x) }` 为**表达式**（返回 `List<R>`，body 须返回 `R`）；`parallel reduce(op, xs) { ... }` 归约（`op` 须结合律）；body 须 `pure` 或仅 `io.read`、无共享可变。
- v0.2：定义语义；executor / codegen 后置（§77）；并发运行时完整语义见 **§87 决议记录 V**。

### 21.1 哲学 + Task 单位

| 语言 | 模型 | 问题 |
|------|------|------|
| C/C++ | `pthread_create` 裸线程 | 死锁、数据竞争、生命周期危险 |
| Java | 线程池 | 重量级、资源管理复杂 |
| Go | goroutine + channel | 简单，但**共享内存安全靠程序员** |
| Rust | `async` + `Future` + `Pin` + `Send/Sync` | 安全，但**学习成本高** |
| **AILang** | **Task-Based Concurrency** | 简单 + 安全 + AI 可分析 |

> AILang 既不照搬 goroutine，也不照搬 Rust Future 的复杂层，而是设计统一模型。

```
Task
  │
  ▼
Runtime Scheduler        （AILang 自带，M:N 调度）
  │
  ▼
OS Thread
```

Task 是**可调度、有生命周期、编译器可追踪**的并发单位；程序员不直接操作线程。

### 21.2 Task 定义与 spawn

```ail
task download() {
    fetch()
}

let h: TaskHandle = spawn download()    // 返回 TaskHandle（见 §87）
let h2 = spawn {                        // 匿名块；默认 scoped（作用域末隐式 join）
    await fetch_async()
}
spawn detached compute()                // 显式分离：仍返回 TaskHandle，cancel 可传播（非 fire-and-forget，§87 #4）
```

`spawn` 时编译器检查（见 §87）：
- 参数是否 `Send`（可跨线程移动，auto-trait 推导）
- 生命周期是否安全（被 spawn 的数据不能是 `borrow` 的非逃逸借用——`borrow` 不逃逸且禁跨 `await`，权威见 §18.5 / §74.1；本文 §21.3 释其与「无 Pin」的关系）

**结构化并发**：spawn 默认 **scoped**——task 附属 spawn 所在作用域，作用域末**隐式 join**（父等待子）；显式分离用 `spawn detached`。`cancel h`（§21.8）向 `h` 子树传播。

### 21.3 async / await

网络 IO 等「等待型」操作用 `async fn`：

```ail
async fn request() -> Response { ... }      // T = Response

let response = await request()              // await request() 得 Response
```

- **返回语义**：`async fn f() -> T` 的 `T` **即 `await` 产出的值**；调用 `f()`（不 await）得 `Future<T>`（`std.async` lang item），`await f()` 得 `T`。`await` 为**一元前缀**：`await f()` / `await fut.field`。（详见 §87。）
- **无 `Pin`**（相对 Rust async 的关键简化）：`borrow`/`borrow_mut` **不逃逸、禁跨 `await` 点**（权威：§18.5 / §74.1）→ async 状态机**永不自引用** → 无需 Rust 的 `Pin`。此「禁跨 `await`」是并发各节反复引用的拱顶规则（§21.2、§21.4 借用不逃逸均其推论）。
- **阻塞禁忌**：`async fn` 内**禁止同步阻塞 I/O**（`io.blocking` effect，如 `fs.read` 同步版）——会占死 M:N worker；阻塞须经 `spawn_blocking { }` 或 async 变体（`fs.read_async`）。

### 21.4 Channel 通信 + 类型安全

AILang 推荐**不共享内存，用 Channel 通信**（CSP，类 Go）：

```ail
let ch = Channel<int>()              // 无界 MPMC（见 §87）
let bounded = Channel<int>(8)        // 有界，容量 8；cap=0 = rendezvous 握手

spawn {
    ch.send(10)
}
ch.close()                           // 关闭：此后 receive()/try_receive() 耗尽即返回 None

let value = ch.receive()             // Optional<int>；阻塞至有值（Some）或关闭耗尽（None）
let maybe = ch.try_receive()         // Optional<int>；None = 空或关闭耗尽
```

- **模型**：**MPMC**（多生产者多消费者，匹配多 Task 同 `receive`）；无界 `Channel<T>()` / 有界 `Channel<T>(cap)`；`close()` 关闭；`receive() -> Optional<T>`（阻塞至有值得 `Some(v)`，关闭耗尽得 `None`）、`try_receive() -> Optional<T>`（非阻塞；`None` = 空或关闭耗尽）；select 含关闭检测分支（§21.5）。（详见 §87。）
- **关闭后的 `receive()`**：`receive() -> Optional<T>` 在**关闭且耗尽**时返回 `None`（与 `try_receive()` 同型，**不 panic、不引入新 panic 符号**）——调用方可经 `match` / select 分支判 `None` 优雅退出。`receive()` 与 `try_receive()` 返回类型相同（`Optional<T>`），唯一差别在「open 且空」：前者阻塞等待，后者立即 `None`。
- `Channel` 为运行时支持的**共享 `Send` 句柄**（跨 spawn 克隆，底层队列）；`spawn { }` 块体为**控制构造**，可捕获外层不可变 / `Send` 变量（**不含 `borrow`**——借用不逃逸，详见 §21.3 / §18.5 / §74.1；非通用闭包，v0.2 无一等闭包）——见 §21、§84 #7。

Channel 强类型：`Channel<User>` 只能 `send(User)`，`send(int)` 报 `Channel Type Error: Expected User, Found int`。

### 21.5 select（多路等待）

```ail
select {
    ch1.receive() { v ->
        handle1(v)              // v: Optional<T>（Some=值 / None=关闭）；v -> body（同 §16 match 约定）
    }
    ch.send(x) { ok ->          // send 分支
        send_done(ok)
    }
    timeout(100ms) { give_up() }    // 超时分支
    default { fallback() }          // 无就绪即走（非阻塞）
}
```

- **分支种类**：`receive` / `send` / `await` + 可选 `default { }`（非阻塞）+ 可选 `timeout(d) { }`（`default`/`timeout` 为 select 内上下文关键字，不入 56）。（详见 §87。）
- **公平性**：多分支就绪**随机公平择一**（类 Go）；**关闭** channel 的 receive 分支立即可就绪并交付 `None`（配合 §21.4 `close`）。
- 接收值绑定复用 match 的 `{ 绑定 -> body }` 约定（§16）。

### 21.6 共享可变 + 安全共享

```ail
var count: int = 0
spawn task1()
spawn task2()           // ❌ 编译器拒绝：count 跨 Task 共享可变 → 数据竞争
```

强制走 Channel 或 `Mutex`/`Shared`。跨 Task 移动 / 共享的安全性由 `Send`/`Sync` auto-trait 静态判定（§87 #6、§34）。

```ail
let config: Shared<Config> = ...     // 多 Task 只读共享
let data = Mutex<Data>(...)
lock(data) {                         // 临界区，自动解锁
    update()
}
```

`Shared<T>` / `Mutex<T>` / `lock` 见 §18.6。

### 21.7 Actor 模型（AI 友好）

Actor = 拥有**独立状态 + 消息入口 + 生命周期**的并发实体：

```ail
actor UserService {
    state:
        users: Map<UserId, User>

    message AddUser { user: User }
    message GetCount

    on AddUser { msg ->                             // 消息处理子句 on（见 §87）
        self.state.users.insert(msg.user.id, msg.user)
    }
    on GetCount { _ ->                              // reply(expr) 返回该消息的响应值
        reply(self.state.users.length())
    }
}

let h: ActorHandle = spawn actor UserService()      // 启动（返回 ActorHandle）
h.send(AddUser { user: u })                         // 投递（或 h ! AddUser { ... }）
```

特点：状态完全封装，仅通过消息交互，无共享可变。`on` 子句定义每条消息的处理；`self.state` 单 Task 串行访问（无需锁）。

- **`reply`（on-handler 上下文关键字）**：`reply` 是 `on` handler 内的**上下文关键字**（与 `on` 同，不入 56 关键字）；`reply(expr)` 退出当前消息处理并返回响应值 `expr`，其类型即**确定（infer）该 `message` 的响应类型**（response type；可为 `Result<T,E>`；无 `reply` 的 `on` handler 响应类型为 `void`）。故 **`on` handler 禁 `throw`**（actor 无 `Result` 返回签名）——错误出口即 `reply(Err...)`，或经死信传递（§21.9、§89 #8）。产生式：`reply` 无独立 EBNF 产生式——为上下文关键字，经 §27 `primary` 的 ident 路径派生；其 return 式控制流语义见本节（§21.7）。
- **AI 极易理解 Actor 的行为边界**（状态 + 可接收消息清单 + 每条消息的行为）。

### 21.8 Task 生命周期 + 取消

```
Created → Running → Waiting → Completed
                       │
                       └──► Cancelled
```

上图运行时生命周期为 executor 调度态，**不入** `.ailmeta`；task 的 `.ailmeta` `declarations[]` 条目（节选——省略 `span`，schema `task` 附加 `sig` 字段，§88 #4 / §24）：
```json
{ "kind": "task", "name": "download", "sig": "task download()" }
```

```ail
cancel h                      // h: TaskHandle
if h.cancelled() { ... }      // 查询取消状态
```

- **协作式**：`cancel` **仅在 `await` 点生效**——CPU-bound task（无 await）不可取消，须调 `task.yield()` 让出以检查 `cancelled()`（`yield` 为 `std.async` 方法，非关键字）。
- **取消安全**：取消 = 在挂起点**丢弃状态机** → **确定性释放所有 locals**（§18.7）→ 资源正确 close；`lock(m){ }` 块边界（含取消展开）**自动释放锁**。结合 §21.3「无 Pin」，状态机丢弃即安全。
- **传播**：`cancel h` 自动向 `h` 的子 Task 传播（结构化并发；详见 §87）：

```
parent task ──cancel──► child task ──cancel──► ...
```

### 21.9 并发错误处理

**不隐藏错误**（§87 #1 / §89 #8 统一表）：

| 上下文 | `throw`/`try` | 错误出口 |
|--------|---------------|----------|
| `async fn -> Result<T,E>`（用户 `await`） | ✓ | 错误随 `await` 传调用方 |
| `task`（void 体） | ✗ 禁 | 经 Channel 投递错误消息或 `cancel` |
| actor `on` handler | ✗ 禁 | `reply(Err...)`（响应类型由 `reply` 表达式确定，可为 `Result<T,E>`，见 §21.7）或死信 |
| server route handler | ✗ 禁（豁免） | runtime 唯一 awaiter，返回声明类型即可 |

```ail
let result = await risky_async()         // Result<T, Error>：用户 await → 须 Result
// task：错误经通道 —— ch.send(Err) 或 cancel h
// actor：on GetCount { _ -> reply(Ok(self.state.count)) }   // reply 返回响应值（可为 Result）
// server handler 豁免：async fn chat(req: Request) -> Response（runtime 派发）
```

`throw` 仅合法于返回 `Result<T,E>` 的函数体（§17、§89 #6）；`void`/`task`/handler 体禁 throw。无「静默失败」。

### 21.10 CPU 并行（parallel for / reduce）

计算密集型：

```ail
let squares: List<int> = parallel for x in data {     // 表达式：返回 List<R>
    x * x                                              // body 须返回 R
}
let sum = parallel reduce((a, b) -> a + b, data) { x -> x }   // 归约：op 须结合律
```

- **表达式化**：`parallel for x in xs { f(x) }` 返回 `List<R>`（body 须返回 `R`）；`parallel reduce(op, xs) { x -> f(x) }` 归约（`op` 结合律，`reduce` 为 `parallel` 后上下文关键字）。（详见 §87。）
- 编译器自动**分片 → 调度线程 → 合并结果**；body 须 `pure` 或仅 `io.read`、**无共享可变**（§87 #9）（§21.6）。

### 21.11 AI 自动并发建议

编译器分析：纯函数 + 无共享状态 → 提示：

```
ℹ This loop can run in parallel
  (process is pure, no shared mutable state)
```

未来可自动并行化（依赖效果系统判定 `pure`）。

### 21.12 服务器模型（原生构造）

AILang 原生适合 Web/数据库/AI Agent/分布式：

```ail
server app {
    route "/chat" {
        async fn chat(request: Request) -> Response {
            return agent.run(request)
        }
    }
}
```

`server` / `route` 为语言级构造，编译器生成路由 + 并发调度 + 元数据。handler 块为控制构造（§84 #7/#8），可捕获外层 `Send` 句柄（如连接池 `db`）。interface 值为 **Move 句柄**（§85 #6）：handler 捕获 owned 句柄，每次调用经形参模式推断 `borrow`（§84 #13），借用仅存于同步处理帧内、不跨 `await`（§18.5 / §74.1）。**handler 豁免 Result 规则**（§87 #1）：runtime 是唯一 awaiter，`async fn chat(req) -> Response` 返回声明类型即可（非用户 `await`）。

### 21.13 并发安全总结

| 层次 | 手段 |
|------|------|
| **默认** | 共享不可变 + 消息通信 + 编译期检查 |
| **高级** | `Mutex` / `Shared` / `Actor` |
| **逃生** | `unsafe`（见 §25.1） |

## 22. 可见性（§91 #2）

```ail
public fn api() { ... }              // 跨 package 可见
private fn internal() { ... }        // 仅本 package（默认）

public struct User {
    public id: UserId                // 字段须显式 public 才跨 package 可见
    cache: string                    // 默认 private
}
```

`public` / `private` 修饰**所有顶级 item + struct/enum 字段 + 方法**；**默认 `private`**（含字段、方法）。trait/interface 方法**不单独修饰**（可见性随类型，类 Rust）。跨 package 仅可访问 `public`；`from M import name` 仅可导入 M 的 public item（§91 #6）。

---

## 23. AI 原生声明

> AILang 核心差异化。详见 §40。

### 23.1 agent（Agent 为语言一级公民）
```ail
agent ResearchAgent {
    goal: "research AI news"
    tools: [ web.search, database.query ]
}
```
编译为元数据（`kind/goal/tools`），供 AI Agent 运行时消费。

### 23.2 tool（自动生成下游 Schema）
```ail
tool Search {
    fn query(text: string) -> List<SearchResult>
}
```
`tool` 自动产出 OpenAI Tool Schema / JSON Schema / API 文档。

> `agent` / `tool` 是新的顶级声明（v0.2.1 入关键字）。与 `struct`/`interface`/`actor` 的精确边界见 §83 #6。

---

## 24. `.ailmeta` 正式字段规范（v0.2，权威；§88 #2）

**顶层**（`schema_version` / `language` / `package` required；`functions` / `types` / `errors` / `declarations` 按需）。`schema_version` 为**独立 semver**，与语言版本解耦。

**`function` 条目**：

| 字段 | 类型 | req | 说明 |
|------|------|-----|------|
| `identity` | `{name, description?}` | ✓ | 函数身份（description 来自 `///`） |
| `span` | `{file,line_start,col_start,line_end,col_end}` | ✓ | 源码定位（§69.1） |
| `input[]` | 见下 | | 入参 |
| `output` | `{type, meaning?}` | | 返回类型 + 含义 |
| `effects[]` | `string[]` | | effect 词表（§19、§88 #6） |
| `errors[]` | `string[]` \| `{name, payload?}[]` | | **单一真源**：≡ 签名 `E`（error 枚举）变体集；裸变体为 `string`，带 payload 变体为 `{name, payload:[{name,type}]}`。非 `Result` 返回 → `[]`；`E` 为 `std.core.Error`（opt-in 通用型，非命名枚举）→ `["Error"]`（类型名占位，变体不展开，§17） |
| `ownership` | `{output}` | | 见枚举（**不含** `inputs`——mode 已在 input[]） |
| `is_pure` | `bool` | ✓ | 无任何 effect（含 `alloc`）+ 不改入参；与 `errors[]` 正交 |
| `is_async` | `bool` | ✓ | `async fn` |

**`input` 条目**：`{name, type, meaning?, mode}`。

**`type` 条目**：`{name, kind, span, …}`；`kind` 决定附加字段——`semantic`(+base/meaning/constraint?)、`struct`(+fields[])、`enum`(+variants[])、`interface`(+sigs[])、`trait`(+sigs[])、`error`(+variants[])、`alias`(+base)。`type X = Y` 无 `meaning`/`constraint` → `alias`（透明同型）；附 `meaning` 或 `constraint` → `semantic`（名义；`constraint` 为可选字段，谓词如 `["value >= 0", "value <= 150"]`）。input / output / field 的 meaning 按名义类型附着传递；bare 基础类型 / alias = null。（类型层语义详见 §15、§92。）

**`error` 条目**：`{name, variants[], span}`；`variants[]` 项可为裸名或 `{name, payload:[{name,type}]}`（带数据变体，§89 #7）。

**`declaration` 条目**（顶级声明统一容器，§88 #4）：`{kind, name, span, …}`——`kind` ∈ {`agent`(+goal/tools)、`tool`(+fns，派生 OpenAI Schema)、`actor`(+state/messages/on-handler/ActorHandle)、`server`(+routes)、`task`(+sig)}。

**枚举汇总**：
- `mode`（input）：`move` / `borrow` / `borrow_mut` / `copy`（§88 #2 / #3）。
- `ownership.output`：`new`（新建）/ `move`（转移入参）。（无 `borrowed`——§18.5 禁 borrow 返回，函数输出恒为新建或转移入参；§88 #2）
- `kind`（type）：`semantic` / `struct` / `enum` / `interface` / `trait` / `error` / `alias`。

**这一份 `.ailmeta` 是 AILang 存在的全部理由。** AI 不需要「阅读源码并猜测」——它读到的是一份结构化、可验证、由编译器背书的程序语义说明书。

---

## 25. Unsafe 与 FFI

> 详见 §25.1-§25.4。AILang 默认安全；以下为系统级逃生舱。

### 25.1 Unsafe 块 + 精确清单
```ail
unsafe {
    let p: raw_pointer<int> = ...     // 裸指针仅在 unsafe 内
}
```

`unsafe` 块内的违规**由程序员负责**，且在 `ail publish` 时被审计（§42）。

**`unsafe { }` 内必需操作的精确清单（§90 #4）**：① 裸指针（`raw_pointer<T>`）解引用 / 读 / 写；② 调用 `extern`（FFI）函数；③ 调用其他标 `unsafe` 的函数；④（预留）全局可变——v0.2 **不引入 `static mut`**（全局可变须走 `Mutex<T>`/`Shared<T>`，与「禁跨 Task 共享可变」一致，§21），亦**无 union**（用 `enum` tagged union + `layout(C)` struct 对接 FFI）。清单外操作（约束断言、越界检查）**非 unsafe**——违约 = panic（确定性、安全）。整数溢出亦**非 unsafe**（debug=panic / release=回绕，§15.1）。

### 25.2 指针设计

默认**无裸指针**。仅在 `unsafe` 内：

```ail
unsafe {
    let p: raw_pointer<int> = ...
}
```

### 25.3 FFI

调用 C：
```ail
extern c {
    fn printf(text: string)
}
```

调用 Rust（未来）：
```ail
extern rust {
    ...
}
```

> panic 跨 FFI 边界详见 §17。

### 25.4 内存布局

系统/协议开发需 C 兼容布局：

```ail
layout(C) struct Packet {             // C 兼容布局（FFI / 二进制协议）
    id: int
    data: Array<byte, 256>
}
```

`layout(C)` 强制 C 兼容的成员排列与对齐（用于 FFI、二进制协议）。

---

## 26. 完整示例

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
    // 简化示例；完整签名（含 db: borrow Database 参数）与实现见 §63
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

## 27. 形式文法（EBNF，v0.2.1）

```
module      := "package" module_path import* item*       // 包名=完整点分路径（§91 #1/#3）
import      := "import" module_path ("as" ident)?        // 全路径（§91 #4）；禁 import *
            |  "from" module_path "import" ident ("as" ident)?   // 仅可导入 public item（§91 #6）
module_path := ident ("." ident)*                        // 如 std.http / app.utils（§91 #1）
visibility  := "public" | "private"                      // §91 #1/#2
item        := visibility? ( fn | struct | enum | interface | trait
                           | error | type_alias | meaning | constraint
                           | agent | tool | actor | task | server | test | extern )

fn          := ("pure")? ("async")? "fn" ident generic? "(" params? ")"
              "->" type where_clause? contract* block
fn_sig      := ("pure")? ("async")? "fn" ident generic? "(" params? ")" ("->" type)?
              where_clause? contract*          // 签名（无 body）；interface/trait 用
where_clause := "where" "{" bound* "}"         // 编译期 trait bound（§86 #2）；where 为上下文关键字（同 lock 先例，不入 56，详见 §9）
bound       := ident "<" type ">"              // 如 Ord<T>、Eq<T>（§86 #8 运算符 trait）
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
type_alias  := "type" ident "=" type              // §92 #1：无 meaning/constraint=alias；附 meaning=semantic；附 constraint=semantic+constraint
meaning     := "meaning" ident ":" string_lit_or_block   // §92 #5：目标 ident 须引用同模块已声明 type；块形多行
constraint  := "constraint" ident ":" "{" expr* "}"      // §92 #5：目标 ident 须引用已声明 type；value 绑定 §92 #4
agent       := "agent" ident "{" ("goal" ":" string_lit)? ("tools" ":" "[" expr_list "]")? "}"
tool        := "tool" ident "{" fn* "}"
actor       := "actor" ident "{" ("state" ":" field+)? ("message" ident ("{" field* "}")? | on_clause)* "}"   // §87 #5
on_clause   := "on" ident arm_body           // 消息处理：on AddUser { msg -> ... }（绑定复用 match §16）
task        := "task" ident "(" params? ")" block
server      := "server" ident "{" route* "}"
route       := "route" string_lit "{" fn* "}"
test        := "test" string_lit block
extern      := "extern" ident "{" extern_fn* "}"       // 如 extern c { ... }
extern_fn   := "fn" ident "(" params? ")" ("->" type)?

type        := ident ( "<" type_arg ("," type_arg)* ">" )?
type_arg    := type | "const" ident ":" ident                // const 泛型实参（§86 #3），如 Array<int, 3>
generic     := "<" ident ("," ident)* ">"                    // 泛型形参列表（声明侧）；编译期 bound 经 where_clause（§86 #2），不用内联 `T: Trait`
block       := "{" stmt* "}"
stmt        := "let" ident (":" type)? "=" expr
            |  "var" ident (":" type)? "=" expr
            |  "return" expr?
            |  "throw" expr                         // 语句级（§85 #10），类型 never（⊥，§89 #4）
            |  "if" | "for" | "while" | "loop" | match | select          // match/select 引独立产生式（形见下，同 try_catch）；select 语句 §87 #10
            |  "spawn" ("detached")? "actor"? ( postfix | block ) | "cancel" expr | "parallel" ("for"|"reduce") ...   // spawn task/block→TaskHandle、spawn actor Name()→ActorHandle（§87 #5）/ cancel h（§87 #4/#9）；postfix 含被调者＋调用后缀（call 仅后缀，故不可单用）
            |  try_catch                        // "try { } catch { }" 局部捕获（§89 #1，形见下 try_catch）；"try {" → 捕获形，"try" 后非 "{" → unary 传播算符（形见下 unary / §89 #1），按下一 token 消歧
            |  "unsafe" block | expr

expr        := assign | send        // | send：actor 投递糖 `h ! Msg`（§87 #5），语句形二元 infix（右操作数贪婪至 expr）
assign      := or ( ("="|"+="|"-=") assign )?
or          := and ( "||" and )*
and         := eq ( "&&" eq )*
eq          := add ( ("=="|"!="|"<"|">"|"<="|">=") add )*
add         := mul ( ("+"|"-") mul )*
mul         := unary ( ("*"|"/"|"%") unary )*
unary       := ("-"|"!"|"borrow"|"borrow_mut"|"move"|"copy"|"await"|"try") unary | postfix   // 一元前缀：await/try（§87 #3、§89 #1）；try f() = unwrap-or-propagate
postfix     := primary ( call | index | member )*
call        := "<" type_arg ("," type_arg)* ">" "(" args? ")"  // 可选类型实参（§86 #7），如 decode<User>(row)
            |  "(" args? ")"
index       := "[" expr "]"
member      := "." ident
send        := postfix "!" expr        // actor 投递糖：h ! Msg ≡ h.send(Msg)（§87 #5）；"!" 二元 infix（投递），与 unary 前缀逻辑非按位置消歧
args        := expr ("," expr)*
primary     := literal | ident | "(" expr ")" | struct_init | list_lit | map_lit   // Variant(args) 复用 call 文法、由名字解析消歧（§89 #2）；约束类型构造 T(expr) 亦复用 call 文法（名字解析消歧「类型名 vs 函数 vs 变体」，§92 #2）；裸变体 = ident
list_lit    := "[" "]" | "[" expr ("," expr)* "]"
map_lit     := "[" ":" "]" | "[" expr ":" expr ("," expr ":" expr)* "]"
variant     := ident ("(" type_list ")")?            // 裸变体（ident）或带 payload（§15.6）；enum/error 共用
effect      := ident ("." ident)*                    // 点分效果名 §19 词表 <domain>.<verb>，如 database.write / io.blocking
type_list   := type ("," type)*                      // implements 子句 / variant payload 类型组
string_lit_or_block := string_lit | "{" string_lit+ "}"   // meaning 单行串或多行块（§92 #5）
struct_init := ident "{" (ident ":" expr ("," ident ":" expr)*)? "}"   // 结构字面量 TypeName { f: v, ... }
expr_list   := expr ("," expr)*                      // agent tools 列表等

match       := "match" expr "{" arm* "}"
try_catch   := "try" block "catch" "{" arm* "}"    // 局部捕获（§89 #1）；"try" 后跟 "{" → 捕获形，否则为 unary 传播算符
arm         := pattern ":" arm_body
arm_body    := "{" bind ("," bind)* "->" stmt+ "}" | block     // { 绑定 -> stmt+ } 单括号内联（§16 match / §21.7 on handler）；无绑定形 = block
pattern     := (ident ".")? ident | "_"
bind        := ident
select      := "select" "{" (sel_arm | sel_special)* "}"      // §87 #10
sel_arm     := ("await")? postfix "(" args? ")" arm_body        // receive()/send(v)/await f() 分支（§87 #10）
sel_special := "default" arm_body | "timeout" "(" expr ")" arm_body   // 非阻塞 / 超时（上下文关键字）
```

> 文法为骨架；`async/await`、泛型 `where`、`!` 投递糖（send，§87 #5）、`spawn actor` 等细节见对应小节与各领域文档。

---

<a id="part-ii"></a>
# Part II · 标准库与包生态

## 28. 设计目标

对标：

| 语言 | 包管理 |
|------|--------|
| Rust | cargo |
| Python | pip |
| Go | go modules |
| **AILang** | **`ail`** |

`ail` = **AILang Integrated Launcher**。在 cargo/go 的能力之上，AILang 包**额外携带机器可读的语义元数据**——这是生态层的 AI 原生增量。

---

## 29. 命令集

| 命令 | 作用 |
|------|------|
| `ail new <project>` | 生成项目骨架 |
| `ail build` | 编译当前包 |
| `ail run` | 编译并运行 |
| `ail test` | 运行内置测试 |
| `ail add <pkg>` | 增加依赖（改写 `ail.toml`） |
| `ail remove <pkg>` | 删除依赖 |
| `ail update` | 更新依赖 |
| `ail publish` | 发布到包仓库（含安全扫描，见 §42） |

## 30. 项目结构 + 包清单 ail.toml

`ail new hello` 生成：

```
hello/
├── ail.toml              # 包清单
├── src/
│   └── main.ail          # 入口
├── tests/                # 测试
├── assets/               # 资源
└── build/                # 构建产物（原生二进制 + .ailmeta）
```

库项目额外有 `src/lib.ail`。

### 30.1 包清单 `ail.toml`

```toml
[package]
name = "web_server"
version = "0.1.0"
language = "ailang"

[dependencies]
http = "1.0"
database = "2.1"
ai = "0.5"
```

`ail add http` 自动写入 `[dependencies]`；`ail remove http` 删除；`ail update` 拉取符合版本约束的最新版。

**包名 ↔ 模块名（§91 #10）**：`[package].name` = **根包名 = 导入根前缀**——`name = "web_server"` 则内部声明 `package web_server`，依赖方 `import web_server.<module>`；依赖 `http = "1.0"` 安装后以 `import http.<module>` 引用（依赖包名作根）；版本遵循 semver；包名**禁与 `std` 冲突**。

---

## 31. AILang 包 ≠ 普通语言的包

普通语言：**包 = 代码**。
AILang：**包 = 代码 + 类型信息 + AI Metadata + 安全描述**。

一个发布的 AILang 包：

```
http/
├── src/              # 源码
├── ail.toml          # 清单
├── http.ailmeta      # 机器可读语义（强制产物）
└── docs/
```

## 32. `.ailmeta` 作为包的强制产物

每个库**必须**随包发布 `xxx.ailmeta`（权威 schema 见 §24）。例（`http` 库，节选——省略 `span`）：

```json
{
  "schema_version": "0.2.0",
  "language": "AILang",
  "package": "http",
  "functions": [
    {
      "identity": { "name": "get", "description": "send HTTP GET request" },
      "input": [ { "name": "url", "type": "URL", "meaning": "uniform resource locator", "mode": "borrow" } ],
      "output": { "type": "Result<Response, NetworkError>", "meaning": null },
      "effects": [ "network" ],
      "errors": [ "Timeout", "ConnectionFailed" ],
      "ownership": { "output": "new" },
      "is_pure": false,
      "is_async": true
    }
  ]
}
```

> 包版本不在 `.ailmeta` 顶层（在 `ail.toml`，§30）；`.ailmeta` 顶层仅 `schema_version`（schema 自身 semver，与语言/包版本解耦，§24）。

**AI 使用库的方式**：不读源码，直接 `load metadata → understand API → generate code`。这是 AILang 生态相对其他语言的核心差异——库对 AI 是「即插即用」的。

---

## 33. 模块总览

```
std/
├── core          # 默认加载：Optional / Result / error / print
├── collections   # List / Map / Set
├── string        # UTF-8 字符串
├── math          # 数值边界运算符族（§90）+ 基础数学（v0.3+ 细化）
├── io            # v0.2 占位：流式 IO（stdin/stdout 流，与 print 互补）
├── fs            # 文件系统（§37）
├── net           # 网络（§33.1）：NetworkError
├── http          # HTTP 客户端/服务端（§38）：Request / Response / URL
├── crypto        # v0.2 占位：hash / hmac / ...
├── time          # v0.2 占位：now / duration / ...
├── async         # 异步运行时（§34：Future / TaskHandle / ...）
├── sync          # 同步原语（Mutex<T> / Shared<T> / lock；§18.6 / §21.6）
├── database      # 统一数据库接口（§39）：Row / decode<T>
├── ai            # AI 原生（§40）：SearchResult / AIError
└── testing       # 内置测试（§41）
```

### 33.1 占位 / 底层模块（§88 #8/#9/#10；§90 数值语义）

`std.io` / `std.crypto` / `std.time` 为 **v0.2 占位**——仅占名空间，API v0.3+ 细化（`std.io` 为流式 IO，与 `std.core.print` 互补）。`std.math` 承载**整数数值语义运算符族**（§90 #1/#3，非占位）：边界运算 `wrapping_add` / `checked_add` / `saturating_add` / `overflowing_add` / `checked_div`（同名 sub/mul/neg），完整定义详见 §15.1。浮点 IEEE 754 特殊值检测 `is_nan` / `is_inf` / `is_finite` 为 **float 方法**（`x.is_nan()`，borrow self），非 `std.math` 自由函数（§94 #4），运算永不 panic。

`std.net` 声明底层网络错误：

```ail
// std.net
error NetworkError {            // 网络错误枚举（§88 #10）
    Timeout
    ConnectionFailed
}
```

`std.http`（§38）的网络错误复用 `std.net.NetworkError`。

---

## 34. `std.core`（默认加载）

无需 `import`。包含：

### Optional\<T>
```ail
fn find_user(id: UserId) -> Optional<User> { ... }
// Some(user) 或 None；无 null
```

**`Optional<T>` 方法签名表**（§88 #7；接收者模式 §85 #1）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `unwrap` | `(self) -> T` | 取值；`None` 则 **panic**（栈展开+Drop，§89 #3；消费 self） |
| `unwrap_or` | `(self, default: T) -> T` | 取值或默认 |
| `is_some` | `(borrow self) -> bool` | 是否 `Some` |
| `is_none` | `(borrow self) -> bool` | 是否 `None` |

### Result\<T, E>
```ail
fn read_config() -> Result<Config, ConfigError> { ... }
```

**`Result<T, E>` 方法签名表**：

| 方法 | 签名 | 说明 |
|------|------|------|
| `unwrap` | `(self) -> T` | 取 `Ok` 值；`Err` 则 **panic**（§89 #3） |
| `unwrap_or` | `(self, default: T) -> T` | 取值或默认 |
| `is_ok` | `(borrow self) -> bool` | 是否 `Ok` |
| `is_err` | `(borrow self) -> bool` | 是否 `Err` |

### error —— 统一错误系统
命名错误枚举经 `error` 声明（规范示例见 `std.net.NetworkError`，§33.1；`std.fs.FileError` §37、`std.ai.AIError` §40 同形）。

### `print` —— stdio 默认内建
`print(x: Display) -> void` 输出到 stdout，默认加载、无需 `import`（类 Go `println`）；`.ailmeta` 标注 effect `io.write`（§84 #9、§88 #6；须 `Display` trait §86 #8）。

### `Error` —— 通用错误（动态边界）
`std.core.Error` 为 opt-in 通用错误类型，用于动态边界（DB 驱动 / FFI / 泛型 Task）；惯用代码用命名 `error` 枚举保 `.ailmeta` 精度，边界处显式 `Error → 命名错误` 映射（§17、§84 #6）。

### `panic` / `assert` —— 不可恢复错误（§89 #3）
`std.core.panic(msg: string) -> never` 与 `assert(cond: bool, msg?: string) -> void`（`std.testing` 亦暴露 `assert`）；`unwrap`/`assert` 失败 = panic。完整 panic 语义（确定性栈展开 + Drop/release、展开至 Task 边界或 `extern` 帧、不传染进程、FFI 边界转 abort、v0.2 无 `catch_unwind`）见 §17。`panic`/`assert` 为 std 函数，非关键字（56 不变）。

已知 panic 触发原因 → 符号 → 指针：

| 原因 | panic 符号 | 指针 |
|------|-----------|------|
| `unwrap` / `assert` 失败 | （直接 panic，无独立符号） | §89 #3 |
| 整数溢出（debug 构建） | `ArithmeticOverflow` | §90 #1 |
| 整数除零 / 取模零 | `DivideByZero` | §90 #3 |
| 越界访问（下标越界） | `IndexOutOfBounds` | §17 |
| 约束违约（`T(expr)` 谓词失败） | `ConstraintViolation` | §15.4 / §92 #9 |

### 运算符 trait（§86 #8）
运算符解糖为 std.core trait 方法调用；可作泛型 bound（`where { Ord<T> }`）。核心集：

| trait | 方法 | 运算符 |
|-------|------|--------|
| `Add` / `Sub` / `Mul` | `add` / `sub` / `mul` | `+` / `-` / `*` |
| `Eq` | `eq` | `==` `!=` |
| `Ord` | `lt` | `<` `>` `<=` `>=` |
| `Display` | `display` | 字符串化（`+` 拼接，§11） |

`&&` / `||` / `!` 为内置短路，不可重载。

### `Send` / `Sync` —— 并发安全 marker trait（§87 #6）
`Send`（可跨线程 move）/ `Sync`（可跨线程共享只读）为编译器**自动派生**的 marker trait（`#[lang]`，§86 #5），无需手写 `impl`。Send/Sync 组合规则（Copy 基础 / COW / Move 资源 / `Mutex<T>` / `Shared<T>` / `TaskHandle` 等类型的派生条件）详见 §18.6。

`spawn` / `Channel.send` / actor 投递边界静态查 `Send`（§21、§21.3/§21.8、§73/§74）。

### 并发运行时类型（`std.async` lang item）
`Future<T>`（`async fn` 状态机产物，**无 `Pin`**）、`TaskHandle`（`spawn` 返回，结构化作用域 join / `cancel` / `cancelled`）、`ActorHandle`（`spawn actor` 返回，`send`/`!` 投递）、`spawn_blocking { }`（独立线程池，不占 M:N worker）均为 `std.async` lang item（§86 #5）；完整语义见 §87 决议记录 V、§21、§77。

---

## 35. `std.collections`

设计原则：**不像 Rust 那样碎片化**（一个 `List` 而非 `Vec`/`slice`/`&[T]` 多套）。

| 类型 | 用途 | 底层 |
|------|------|------|
| `List<T>` | 动态数组 | Rust `Vec` |
| `Map<K, V>` | 哈希表 | Rust `HashMap` |
| `Set<T>` | 集合 | Rust `HashSet` |

```ail
let users: List<User> = []
let by_id: Map<UserId, User> = [:]
let ids: Set<UserId> = []
```

> `List` / `Map` / `Set` 为 **COW 值类型**（§18.4、§85 #2/#7）：赋值=共享缓冲（**共享句柄，非位复制** → Copy-compatible，计入 `is_copy`；**原子 rc** → `Send`），修改=克隆；行为同 Copy，**无追踪 GC**。可变方法（`insert`/`append`/`remove`）声明 `borrow_mut self`。

**方法签名表**（§88 #7；接收者模式同 §34；下标 `xs[i]` / `m[k]` 返回值——borrow 不逃逸，故按值返回，要求 `T` 为 Copy/COW）：

| 类型 | 方法 | 签名 |
|------|------|------|
| `List<T>` | `length` | `(borrow self) -> int` |
| | `is_empty` | `(borrow self) -> bool` |
| | `append` | `(borrow_mut self, item: T) -> void` |
| | `remove` | `(borrow_mut self, index: int) -> void` |
| | `contains` | `(borrow self, item: T) -> bool`（`where { Eq<T> }`） |
| `Map<K, V>` | `length` | `(borrow self) -> int` |
| | `insert` | `(borrow_mut self, key: K, value: V) -> Optional<V>`（返回旧值） |
| | `get` | `(borrow self, key: K) -> Optional<V>` |
| | `remove` | `(borrow_mut self, key: K) -> Optional<V>` |
| | `contains` | `(borrow self, key: K) -> bool` |
| `Set<T>` | `length` | `(borrow self) -> int` |
| | `insert` | `(borrow_mut self, item: T) -> bool`（新建返回 true） |
| | `remove` | `(borrow_mut self, item: T) -> bool` |
| | `contains` | `(borrow self, item: T) -> bool` |

---

## 36. `std.string`

统一类型 `string`，UTF-8（COW 值类型，**共享句柄** → Copy-compatible；§18.4、§85 #7）。

```ail
let name = "AI"
name.length()         // 2
name.contains("A")    // true
name.upper()          // "AI"
```

**`string` 方法签名表**（§88 #7；接收者模式同 §34；`+` 拼接见运算符 trait §86 #8）：

| 方法 | 签名 |
|------|------|
| `length` | `(borrow self) -> int` |
| `is_empty` | `(borrow self) -> bool` |
| `contains` | `(borrow self, sub: string) -> bool` |
| `starts_with` | `(borrow self, prefix: string) -> bool` |
| `upper` | `(borrow self) -> string` |
| `lower` | `(borrow self) -> string` |
| `append` | `(borrow_mut self, other: string) -> void` |

---

## 37. `std.fs`（文件系统）

```ail
import std.fs

fn read() -> Result<string, FileError> {
    let text = fs.read("a.txt")
    return text
}
```

编译器自动从 `fs.read` 推断效果 `filesystem.read` + `io.blocking`（同步阻塞，§87 #7），写入 `.ailmeta`：
```json
{ "effects": [ "filesystem.read", "io.blocking" ] }
```

**`FileError` 声明于 `std.fs`**（§88 #10；与 `File` 同模块）：
```ail
// std.fs
error FileError {
    NotFound
    PermissionDenied
}
```

> `fs.read` 为**同步阻塞**（effect 含 `io.blocking`，§87 #7）：可在 `task`/同步 `fn` 内用，但**禁止在 `async fn` 内直接调用**（占死 M:N worker）。async 上下文用 `fs.read_async`（返回 `Future<Result<...>>`），或经 `spawn_blocking { fs.read(...) }`。标准库 IO 双轨（sync / async 变体）。

---

## 38. `std.http`（HTTP 客户端/服务端）

**`Request` / `Response` / `URL` 声明于 `std.http`**（§88 #8）：
```ail
// std.http
type URL = string                  // 语义类型（名义，§92 #1）
meaning URL:
    "uniform resource locator"
struct Request { ... }             // HTTP 请求（method / headers / body / param<T>）
struct Response { ... }            // HTTP 响应（ok / not_found / forbidden / ...）
```

```ail
import std.http

async fn request() -> Response {
    let result = await http.get("https://example.com")
    return result
}
```

`.ailmeta` 告诉 AI：`async` / `network` 效果 / `Response` 输出（网络错误复用 `std.net.NetworkError`，§33.1）。

---

## 39. `std.database`（统一数据库接口）

**`Row` 声明于 `std.database`**（动态行，§86 #1、§88 #8）；`decode<T>` 同模块 lang item（§88 #10）：
```ail
// std.database
type Row = ...                     // 别名（动态行，列名 → 值；§92 #10）
fn decode<T>(rows: List<Row>) -> Result<List<T>, Error>   // typed 解码（单态化，§86 #1）
```

```ail
interface Database {
    fn query(borrow self, sql: string) -> Result<List<Row>, Error>   // 非泛型（§86 #1）
    effects { database.read }                // interface 须带 effects（§85 #4）
    fn exec(borrow self, sql: string) -> Result<void, Error>          // 写操作（UPDATE/INSERT/DELETE）
    effects { database.write }
}
```

驱动支持：PostgreSQL / MySQL / SQLite / Redis。

> `interface` = 服务能力抽象（**dyn 派发**，§15.11）；interface 值为 **Move 句柄**，经 `borrow` 借用（§85 #6）；方法非泛型（§86 #1）。

---

## 40. `std.ai`（AILang 核心竞争力）

这是 AILang 区别于所有现有语言的标准库模块。

**`SearchResult` / `AIError` 声明于 `std.ai`**（§88 #8/#10）：
```ail
// std.ai
struct SearchResult { ... }        // 搜索结果（title / url / snippet）
error AIError {                    // AI 调用错误
    RateLimited
    ModelUnavailable
    BadResponse
}
```

### 40.1 Model 接口
```ail
interface Model {
    async fn generate(borrow self, prompt: string) -> Result<Response, AIError>
    effects { network }
}
```

### 40.2 Agent —— AILang 原生对象
```ail
agent ResearchAgent {
    goal: "research AI news"
    tools: [ web.search, database.query ]
}
```

编译器将 `agent` 编译为 `.ailmeta` `declarations[]` 条目（节选——省略 `span`，§88 #4）：
```json
{
  "kind": "agent",
  "name": "ResearchAgent",
  "goal": "research AI news",
  "tools": [ "web.search", "database.query" ]
}
```

### 40.3 Tool 系统
```ail
tool Search {
    fn query(text: string) -> List<SearchResult>
}
```

`tool` 自动生成多种下游产物：
- **OpenAI Tool Schema**
- **JSON Schema**
- **API 文档**

> `agent` 与 `tool` 是 v0.2.1 入 §9 关键字、§27 文法的顶级声明（详见 §23）。

---

## 41. `std.testing`（内置测试）

```ail
test "login success" {
    let result = login()
    assert result.ok
}
```

`ail test` 收集所有 `test` 块并执行。

---

## 42. 包发布安全（`ail publish`）

发布前自动扫描：

```
package scan        # 包本身扫描
   ↓
type safety         # 类型安全
   ↓
unsafe usage        # unsafe 审计
   ↓
dependency audit    # 依赖审计（类 cargo audit）
```

未通过则拒绝发布。

---

## 43. AILang 生态的优势

传统语言生态：
```
代码 → 编译器 → 程序
```

AILang 生态：
```
代码 + 类型 + 语义 + AI Metadata
        ↓
     Compiler
        ↓
      Binary
        ↓
   AI 可理解的软件生态
```

库不仅被人调用，也被 AI **直接理解并调用**——无需读源码、无需猜语义。这是 AILang 从「语言」升级为「平台」的关键。

<a id="part-iii"></a>
# Part III · 教程：可运行示例

## 44. 约定
- 大括号 `{}` + **可选分号**（下列示例省略分号）。
- 逻辑运算符 `&&` `||` `!`；trait 内联 `implements`；方法显式接收者模式（`borrow`/`borrow_mut`/`self`，省略时默认 `borrow self`；§85 #1）。
- 枚举构造 `Enum.Variant`；`match` arm 用 bare variant（scrutinee 类型已知时）。
- 字符串拼接用 `+`。
- 标 **ⓘ codegen 后置** 的特性：v0.2.1 已定义语义，运行时/LLVM codegen 列为后续（§77）。

---

## 45. Hello World

```ail
package main

fn main() -> void {
    print("Hello AILang")
}
```

---

## 46. 变量与基础类型

```ail
fn main() -> void {
    let age: int = 20              // 不可变
    var count: int = 0             // 可变
    count = count + 1
    const VERSION: string = "0.2.1"

    let pi: float = 3.14
    let ok: bool = true
    let name = "AI"                // 推断为 string

    print(name)
}
```

---

## 47. 函数与运算符

```ail
pure fn add(a: int, b: int) -> int {
    return a + b
}

pure fn can_vote(age: int) -> bool {
    return age >= 18 && age <= 150     // && 优先级低于比较
}

fn main() -> void {
    let x = add(2, 3)                  // 5
    let y = 1 + 2 * 3                  // 7（* 高于 +）
    let flag = !can_vote(10)           // true（一元 !）
    print(x)
}
```

> **整数算术语义（§90 #1/#2/#3）**：`a + b` 溢出——release **二补数回绕**（规范）、debug **panic `ArithmeticOverflow`**；整数除零 / 取模零 → **panic `DivideByZero`**；浮点遵循 **IEEE 754**（`x/0.0`→`±inf`、`0.0/0.0`→`NaN`，**永不 panic**）。显式控制用 `math.checked_add` / `math.wrapping_add` / `math.checked_div`（整数），浮点 NaN 检测用方法 `x.is_nan()`（§33.1）。

---

## 48. Semantic Type（可带 constraint）

```ail
package app

type StatusCode = int              // 透明别名（kind:alias，默认）：无 meaning/constraint，与 int 双向可互换

type UserId = int
meaning UserId:
    "unique identifier of a user"

type Age = int
constraint Age:
{
    value >= 0
    value <= 150
}

fn celebrate(id: UserId, age: Age) -> void {
    print(id)
    print(age)
}

fn main() -> void {
    let id: UserId = 100
    let age: Age = 30
    celebrate(id, age)

    // let n: int = 99
    // celebrate(n, age)      // ❌ n 是 int 变量，不能隐式转 UserId（名义不同）；字面量 celebrate(99, age) 则合法（字面量强制，§15.3 / §84 #1）
    // let bad: Age = -5      // ❌ 编译期约束违约 → 编译错（§15.4 / §92 #2）：Age 须 0..150；运行期经 Age(n) 构造违约才 panic ConstraintViolation（§92 #9）
    //
    // 运行期构造（§92 #2）：字面量之外，运行期 int 须经构造算符 T(expr) 变约束类型
    // let n = compute_age()
    // let a: Age = Age(n)    // 运行期断言 0..150；违约 panic ConstraintViolation（nominal 不变使 let a: Age = n 类型错）
}
```

**AI 从 `.ailmeta` 看到**（`types[]` 条目，节选——省略 `span`）：
```json
{ "name": "StatusCode", "kind": "alias", "base": "int" }
{ "name": "UserId", "kind": "semantic", "base": "int", "meaning": "unique identifier of a user" }
{ "name": "Age", "kind": "semantic", "base": "int", "constraint": ["value >= 0", "value <= 150"] }
```

> **marker-trait 继承（§92 #7）**：语义新类型（如 `UserId`）自动继承基底 `int` 的 `Copy`/`Send`/`Sync`——故 §63 `.ailmeta` 中 `id: UserId` 的 `mode: copy` 成立，`Map<UserId, User>` 可跨 `borrow_mut` 传递。运算 trait（`Add`/`Sub`/`Eq`/`Ord`/…）不自动继承，需显式 impl。

---

## 49. Struct + Trait（内联 implements + 方法）

```ail
trait Display {
    fn display(borrow self) -> string
}

struct User implements Display {
    id: UserId
    username: string

    fn display(borrow self) -> string {
        return "User(" + self.username + ")"
    }
}

fn main() -> void {
    let u = User { id: 1, username: "Tom" }
    print(u.display())                 // User(Tom)
    print(u.username)                  // Tom
}
```

---

## 50. Enum + Match（穷尽）

```ail
enum Status {
    Active
    Disabled
    Deleted
}

fn handle(status: Status) -> void {
    match status {
        Active: { start() }
        Disabled: { stop() }
        Deleted: { archive() }        // 漏掉任一 variant → 编译期报错
    }
}
```

---

## 51. Result + Error 模型

```ail
error FileError {
    NotFound
    PermissionDenied
}

fn read_config(path: string) -> Result<string, FileError> {
    return fs.read(path)              // fs.read 已返回 Result<string, FileError>，同类型直接返回（无 Ok 包装）
}

fn main() -> void {
    match read_config("app.toml") {
        Ok: { text -> print(text) }    // Ok 绑定成功值
        FileError.NotFound: { print("missing") }
        FileError.PermissionDenied: { print("forbidden") }
    }
}
```

**`try expr` 传播算符（前缀一元，§89 #1）+ 显式 `Ok` 构造（§89 #2）**：
```ail
fn load_both() -> Result<string, FileError> {
    let a = try read_config("a.toml")    // try：前缀一元，unwrap-or-propagate（同错误类型；§89 #5）
    let b = try read_config("b.toml")    // 任一 FileError 变体 → 自动传播（∈ FileError）
    return Ok(a + b)                     // 显式 Ok，无 auto-wrap（§89 #2/#10）
}
```

> `try/catch` 是 `match` 的语法糖；`try expr`（前缀）= unwrap-or-propagate，`try { } catch { }` = 局部捕获（§17 / §83 #4 / §89）。

---

## 52. Optional（禁止 null）

```ail
fn find_user(id: UserId) -> Optional<User> {
    if id == 1 {   // UserId 的 == 需 Eq（运算 trait 不自动继承，v0.3 显式 impl，§15.3）；示例假定已支持
        return Some(User { id: 1, username: "Tom" })
    }
    return None
}

fn main() -> void {
    match find_user(1) {
        Some: { u -> print(u.username) }
        None: { print("no user") }
    }
}
```

---

## 53. 泛型 + where

```ail
fn first<T>(list: List<T>) -> Optional<T> {
    if list.length() > 0 {
        return Some(list[0])
    }
    return None
}

fn sort<T>(list: List<T>) -> List<T>
where {                                     // trait bound（编译期，单态化；§86 #2）
    Ord<T>
}
{
    // ... 排序实现
    return list
}
```

---

## 54. Interface vs Trait

```ail
interface Database {                          // 服务能力（dyn 派发；方法非泛型 §86 #1）
    fn query(borrow self, sql: string) -> Result<List<Row>, Error>   // Row = 动态行
    effects { database.read }                // interface 须带 effects（§85 #4）
    fn exec(borrow self, sql: string) -> Result<void, Error>          // 写操作（UPDATE/INSERT/DELETE）
    effects { database.write }
}
// typed 解码：fn decode<T>(rows: List<Row>) -> Result<List<T>, Error>（单态化）

trait Serialize {                             // 类型能力（可作泛型约束）
    fn serialize(borrow self) -> string
}

struct User implements Serialize {
    id: UserId
    username: string
    fn serialize(borrow self) -> string {
        return "{ id: " + self.id + ", name: " + self.username + " }"
    }
}
```

---

## 55. 效果系统 + 契约

```ail
fn withdraw(db: borrow Database, account: borrow_mut Account, amount: int) -> Result<void, Error>
requires {                                   // 前置条件
    amount > 0
    account.balance >= amount
}
ensures {                                    // 后置条件（borrow_mut → caller 可观测）
    account.balance == old(account.balance) - amount
}
effects {                                    // 行为边界
    database.write
}
{
    account.balance = account.balance - amount   // 先突变内存（§85 #5）
    return db.exec("UPDATE ...")                  // 再持久化（Database.exec，§39）
}

pure fn square(x: int) -> int {              // pure = 无效果（含 alloc）+ 不改入参（§85 #9）
    return x * x
}
```

**`.ailmeta` 输出真实、完整的效果集**（节选——省略 `span`；完整规范见 §24）：
```json
{
  "identity": { "name": "withdraw", "description": null },
  "input": [
    { "name": "db", "type": "Database", "mode": "borrow" },
    { "name": "account", "type": "Account", "mode": "borrow_mut" },
    { "name": "amount", "type": "int", "mode": "copy" }
  ],
  "output": { "type": "Result<void, Error>", "meaning": null },
  "effects": ["database.write"],
  "errors": ["Error"],
  "ownership": { "output": "new" },
  "is_pure": false,
  "is_async": false
}
```

---

## 56. Ownership（move / borrow / 借用 / 确定性释放）

```ail
fn print_lines(file: borrow File) -> void {  // 只读借用，不取所有权
    print(file.read())
}

fn process_file() -> void {
    let file = File.open("a.txt")            // owner = process_file 作用域
    print_lines(borrow file)                  // 借用：调用后 file 仍可用
    print(file.size())
}                                             // ← file 在此自动 release（确定性，无追踪 GC）

fn bad() -> void {
    let f = File.open("b.txt")
    consume(f)                                // move
    // print(f)                               // ❌ AIL2001 Use after move
}

fn update(map: borrow_mut Map<UserId, User>) {   // 可写借用（独占）
    map.insert(1, User { id: 1, username: "x" })  // Map.insert: borrow_mut self
}
```

---

## 57. 并发：Task / Channel / select　ⓘ codegen 后置

```ail
task producer(ch: Channel<int>) {
    ch.send(1)
    ch.send(2)
}

fn main() -> void {
    let ch = Channel<int>()
    spawn producer(ch)

    select {
        ch.receive() { v ->                    // v: Optional<int>（Some=值，None=关闭耗尽）
            match v {
                Some: { x -> print(x) }
                None: { }
            }
        }
    }
}
```

- `spawn` 编译器检查参数 `Send` + 生命周期安全。
- 默认禁止跨 Task 共享可变；要共享用 `Mutex<T>` / `Shared<T>`（§18.6）。

---

## 58. 并发：parallel for（CPU 并行）　ⓘ codegen 后置

```ail
pure fn heavy(x: int) -> int {
    return x * x
}

fn main() -> void {
    let data: List<int> = [1, 2, 3, 4, 5]
    let squares: List<int> = parallel for item in data {   // 表达式：返回 List<R>（§87 #9）
        heavy(item)                                        // body 须返回 R
    }
    let sum = parallel reduce((a, b) -> a + b, squares) { x -> x }  // 归约（op 结合律）
    print(sum)
}
```

> 编译器见 body **纯 + 无共享可变**（`heavy` 为 `pure fn`；§87 #9：body 须 `pure` 或仅 `io.read`、无共享可变）→ 允许并行（§21.6、§21.10）。

---

## 59. Actor（独立状态 + 消息）　ⓘ codegen 后置

```ail
actor Counter {
    state:
        count: int

    message Increment
    message Get

    on Increment { _ -> self.state.count = self.state.count + 1 }   // §87 #5
    on Get { _ -> reply(self.state.count) }
}

let h = spawn actor Counter()       // ActorHandle（§87 #5）
```

> AI 极易理解：状态 = { count }，可接收消息 = { Increment, Get }，每条消息的行为由 on 定义

---

## 60. AI 原生：Agent + Tool

```ail
tool WebSearch {
    /// search the public web
    fn query(text: string) -> List<SearchResult>
}

tool DbQuery {
    fn query(sql: string) -> Result<List<Row>, Error>
}

agent ResearchAgent {
    goal: "research AI news and summarize"
    tools: [ WebSearch.query, DbQuery.query ]
}
```

**`tool` 自动生成 OpenAI Tool Schema（下游派生产物，非 `.ailmeta` 权威形）**（§40.3、§88 #4）：
```json
{
  "tool": "WebSearch.query",
  "description": "search the public web",
  "input": [{ "name": "text", "type": "string" }],
  "output": "List<SearchResult>"
}
```

**`agent` 编译为 `.ailmeta` `declarations[]` 条目**（节选——省略 `span`，§88 #4；完整规范见 §24）：
```json
{
  "kind": "agent",
  "name": "ResearchAgent",
  "goal": "research AI news and summarize",
  "tools": ["WebSearch.query", "DbQuery.query"]
}
```

---

## 61. Unsafe + FFI + 布局

```ail
extern c {                                    // FFI：调用 C
    fn printf(text: string)
}

layout(C) struct Packet {                     // C 兼容布局（二进制协议/FFI）
    id: int
    data: Array<byte, 256>
}

fn low_level() -> void {
    unsafe {                                  // 逃生舱：裸指针仅在此
        let p: raw_pointer<int> = get_ptr()
        printf("raw work")
    }
}
```

> `unsafe` 块在 `ail publish` 时被审计（§42）。

---

## 62. 服务器（原生构造）　ⓘ codegen 后置

```ail
server app {
    route "/health" {
        fn health() -> Response {
            return Response.ok("alive")
        }
    }
    route "/chat" {
        async fn chat(request: Request) -> Response {
            return agent.run(ResearchAgent, request.body)
        }
    }
}
```

---

## 63. 综合示例：迷你用户服务

```ail
package userservice

import std.http
import std.database

type UserId = int
meaning UserId:
    "unique identifier of a user"

struct User implements Serialize {
    id: UserId
    name: string
    fn serialize(borrow self) -> string {
        return "{ id: " + self.id + ", name: " + self.name + " }"
    }
}

error UserError {
    NotFound
    PermissionDenied
}

fn get_user(db: borrow Database, id: UserId) -> Result<User, UserError>   // interface = Move 句柄，按值借（§85 #6）
effects {
    database.read
}
{
    let rows = db.query("SELECT * FROM users WHERE id = " + id)   // Result<List<Row>, Error>（interface 非泛型，§86 #1）
    let list = match rows {
        Ok: { list -> list }
        _: { throw UserError.NotFound }        // 边界：Error → 命名错误（§84 #3/#6）
    }
    let users = decode<User>(list)             // List<Row> → Result<List<User>, Error>，单态化（§86 #1）
    match users {
        Ok: { list ->
            if list.length() == 0 {
                throw UserError.NotFound
            }
            return Ok(list[0])
        }
        _: { throw UserError.NotFound }
    }
}

fn handle_request(db: borrow Database, req: Request) -> Response
effects {
    database.read
    network
}
{
    let id: UserId = req.param("id")        // param<UserId>：HTTP 边界解析 + 强制（§84 #14）
    match get_user(db, id) {
        Ok: { user -> return Response.ok(user.serialize()) }
        UserError.NotFound: { return Response.not_found("no user") }
        UserError.PermissionDenied: { return Response.forbidden() }
    }
}

fn main() -> void {
    let db = database.connect("postgres://localhost/app")
    server app {
        route "/users/:id" {
            async fn users(req: Request) -> Response {
                return handle_request(db, req)
            }
        }
    }
}
```

**`userservice.ailmeta`（节选——省略 `span`）**：
```json
{
  "schema_version": "0.2.0",
  "language": "AILang",
  "package": "userservice",
  "functions": [
    {
      "identity": { "name": "get_user", "description": null },
      "input": [
        { "name": "db", "type": "Database", "mode": "borrow" },
        { "name": "id", "type": "UserId", "meaning": "unique identifier of a user", "mode": "copy" }
      ],
      "output": { "type": "Result<User, UserError>", "meaning": null },
      "effects": ["database.read"],
      "errors": ["NotFound", "PermissionDenied"],
      "ownership": { "output": "new" },
      "is_pure": false,
      "is_async": false
    }
  ]
}
```

AI 无需读源码，从 `.ailmeta` 即知：`get_user` 读库、可能 `NotFound`/`PermissionDenied`、返回新 `User`——**零猜测**。

---

## 64. 特性覆盖索引

| 特性 | 示例 |
|------|------|
| package / fn / main | §45 |
| let / var / const / 推断 | §46 |
| pure / 运算符 / 优先级 | §47 |
| Semantic Type / Constraint Type | §48 |
| struct / trait / implements / 方法 / self | §49 |
| enum / match 穷尽 | §50 |
| Result / error / 错误派生 | §51 |
| Optional / Some / None | §52 |
| 泛型 / where | §53 |
| interface / trait 边界 | §54 |
| effects / requires / ensures / pure | §55 |
| move / borrow / borrow_mut / 确定性释放 | §56 |
| task / spawn / Channel / select | §57 |
| parallel for | §58 |
| actor / message | §59 |
| agent / tool / Schema 生成 | §60 |
| unsafe / extern / layout(C) / raw_pointer | §61 |
| server / route | §62 |
| 综合集成 | §63 |

<a id="part-iv"></a>
# Part IV · 编译器实现设计

## 65. 设计哲学（与实现的对应）

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

## 66. 总体架构

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

## 67. 实现语言与工具

- **宿主语言：Rust**。理由：内存安全、性能高、适合写编译器、LLVM 生态成熟。
- **词法：`logos`**（高性能声明式 tokenizer）。
- **语法：手写递归下降 + Pratt 表达式解析**（不用 parser generator——v0.2 的 doc 注释附着、契约块、可选分号等需要完全控制与最佳错误恢复）。
- **LLVM 绑定：`inkwell`**（安全高层封装；pin 一个 LLVM 版本）。
- **元数据序列化：`serde` + `serde_json`**（`.ailmeta` JSON）。

---

## 68. 项目结构

```
ailang/                       # 仓库根 / Cargo workspace
├── Cargo.toml                # [workspace]
├── docs/                     # AILANG.md（单一权威文档；原 8 文档已合并）
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

分 crate 是**用 Cargo 强制阶段边界**：`ail-parser` 无法依赖 `ail-codegen`，循环依赖在编译期被挡掉。crate 统一 `ail-*` 前缀；编译器二进制 `ailc`；包/构建工具 `ail`（见 §79）。

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

## 69. 公共基础（ail-ast）

### 69.1 Span
```rust
#[derive(Clone, Copy, Debug)]
pub struct Span { pub start: u32, pub end: u32, pub line: u32, pub col: u32 }
```
每个 AST/HIR 节点带 span。`ailc` 持源码 + 行表，把 span 渲染为 `main.ail:12:5` 并截取上下文。

### 69.2 Symbol（字符串驻留）
```rust
pub struct Symbol(pub u32);   // 进 interner 后的句柄
```
标识符、字符串字面量、doc 文本全部驻留；同名串只存一份，比较退化为 u32 比较。AST/HIR 要序列化进 `.ailmeta`，用 Symbol 比到处 `String` 省内存、省比较。

### 69.3 Diagnostic
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

## 70. 模块一：Lexer（ail-lexer）

### 职责
源码字节流 → Token 流。不做语义判断。

### 关键设计
- **`logos` 声明式 tokenizer**，零依赖、高性能。
- **大括号语法，非缩进敏感**（v0.2 锁定）。缩进仅为可读性，不影响语义。
- **doc 注释是一等公民**：`/// ...` 不丢弃，作为 trivia 保留，供 Parser 附着到下一个声明——这是 `meaning` 的来源之一。
- **可选分号（Go 风格）**：`;` 是合法 token，但**不强制**。换行作为语句分隔（见 §71 的换行规则）。lexer 需保留换行位置信息供 parser 判断语句边界。
- **隐式行连接**：`() [] {}` 内换行忽略。
- **错误恢复**：词法错误（未闭合字符串、非法字符）产 `Error` token + 诊断，继续推进。

### 数据结构
```rust
pub enum TokenKind {
    IntLit(i64), FloatLit(f64), StrLit(Symbol), BoolLit(bool),
    Ident(Symbol),
    // 关键字（56，权威列表见 §9；v0.2.1 收敛）
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

## 71. 模块二：Parser（ail-parser）

### 职责
Token 流 → raw AST。只做语法，不做语义。

### 关键设计
- **手写递归下降 + Pratt 表达式解析**。
- **可选分号的换行规则**：换行默认结束语句；例外——行尾是二元运算符、逗号、未闭合的 `(` `{` `[`、`->` 等——则续行。`;` 显式结束语句。这是 v0.2 最易出错的语法点。
- **doc 注释附着**：解析声明时回看紧邻的连续 `///` trivia（不被空行打断）拼成 `DocComment`，附着到声明；参数级 doc 同理附着到 `Param`。
- **契约块位置**（v0.2 锁定）：`fn name(params) -> RetType` 之后、函数体 `{ }` 之前，可出现 `effects { }` / `requires { }` / `ensures { }` 契约块；`pure` 是 `fn` 前缀修饰。
- **错误恢复**：遇错合成 `Error` 节点，跳到同步点（`}` `;` 或下一个顶层声明起始关键字）继续。一次编译报多个语法错。
- **保留所有 span**。

### 文法（EBNF 草图，v0.2 大括号；权威见 §27）
```
module      := "package" module_path import* item*       // 包名=完整点分路径（§91 #1/#3）
import      := "import" module_path ("as" ident)?        // 全路径（§91 #4）；禁 import *
            |  "from" module_path "import" ident ("as" ident)?   // 仅可导入 public item（§91 #6）
module_path := ident ("." ident)*                        // 如 std.http / app.utils（§91 #1）
visibility  := "public" | "private"                      // §91 #1/#2（修饰 item + 字段 + 方法）
item        := visibility? ( fn | struct | enum | interface | trait
                           | error | type_alias | meaning | constraint
                           | agent | tool | actor | task | server | test | extern )
fn          := ("pure")? ("async")? "fn" ident generic? "(" params? ")"
              "->" type where_clause? contract* block
fn_sig      := ("pure")? ("async")? "fn" ident generic? "(" params? ")" ("->" type)?
              where_clause? contract*            // 签名（无 body）；interface/trait 用
where_clause := "where" "{" bound* "}"           // 编译期 trait bound（§86 #2）
bound       := ident "<" type ">"                // 如 Ord<T>、Eq<T>（§86 #8 运算符 trait）
contract    := "effects" "{" effect* "}"
            |  "requires" "{" expr* "}"          // 运行期前置断言
            |  "ensures" "{" expr* "}"           // 运行期后置断言
params      := param ("," param)*
param       := doc? ident ":" type
layout      := "layout" "(" ident ")"            // 如 layout(C)；对接 FFI
struct      := layout? "struct" ident generic? ("implements" type_list)? "{" (field | fn)* "}"  // 内联方法（§15.5）
field       := doc? ident ":" type
enum        := "enum" ident "{" variant* "}"
interface   := "interface" ident generic? "{" fn_sig* "}"   // 方法禁泛型（§86 #1）
trait       := "trait" ident generic? "{" (fn_sig | fn)* "}"   // 可带默认方法（§86 #6）
error       := "error" ident "{" variant* "}"
type_alias  := "type" ident "=" type             // §92 #1：无 meaning/constraint=alias；附 meaning=semantic；附 constraint=semantic+constraint
meaning     := "meaning" ident ":" string_lit_or_block   // §92 #5：目标 ident 须引用同模块已声明 type；块形多行
constraint  := "constraint" ident ":" "{" expr* "}"      // §92 #5：目标 ident 须引用已声明 type；value 绑定 §92 #4
agent       := "agent" ident "{" ("goal" ":" string_lit)? ("tools" ":" "[" expr_list "]")? "}"
tool        := "tool" ident "{" fn* "}"
actor       := "actor" ident "{" ("state" ":" field+)? ("message" ident ("{" field* "}")? | on_clause)* "}"  // §87 #5
on_clause   := "on" ident arm_body               // on AddUser { msg -> ... }（绑定复用 match，§16）
task        := "task" ident "(" params? ")" block
server      := "server" ident "{" route* "}"
route       := "route" string_lit "{" fn* "}"
test        := "test" string_lit block
extern      := "extern" ident "{" extern_fn* "}"   // 如 extern c { ... }
extern_fn   := "fn" ident "(" params? ")" ("->" type)?
type        := ident ( "<" type_arg ("," type_arg)* ">" )?
type_arg    := type | "const" ident ":" ident     // const 泛型实参（§86 #3），如 Array<int, 3>
block       := "{" stmt* "}"
stmt        := "let" ident (":" type)? "=" expr
            | "var" ident (":" type)? "=" expr
            | "return" expr?
            | "throw" expr                        // 语句级，类型 never（§85 #10、§89 #4）
            | "if" | "for" | "while" | "loop" | "match"
            | "spawn" ("detached")? "actor"? ( postfix | block ) | "cancel" expr | "parallel" ...  // §87 #4/#5/#9（postfix 含被调者＋后缀，镜像 §27）
            | "unsafe" block | expr
expr        := pratt（见下）
```
> 仅为骨架；`select`、`try_catch`、`match` arm 等完整产生式见 §27。

### 表达式优先级（Pratt，低→高；与 §11 一致）
```
 0  =  +=  -=                    赋值（语句级，右结合）
 1  ||
 2  &&
 3  == != < > <= >=
 4  + -
 5  * / %
 6  unary: - ! borrow borrow_mut move copy await try
 7  postfix: call()  index[]  field.
 8  primary: literal | ident | "(" expr ")" | struct_init     // Variant(args) 复用 call 文法（§89 #2）
```
`borrow` / `borrow_mut` / `move` / `copy` / `await` / `try` 作为一元前缀运算符（`await`/`try` 紧贴 postfix：`await f()` / `try f()`，§87 #3、§89 #1）。`try expr` = unwrap-or-propagate（类 Rust `?`，但无 `From` 隐式转换——`E1≠E2` 类型错，§89 #5）；`try { } catch { }` 为局部捕获（§27 `try_catch`、§89 #1）。`throw` 为**语句级控制流构造**（非运算符），类型 `never`（⊥，§89 #4），见 §17、§89 #6。

### 陷阱
- **可选分号 + return 表达式**：`return a + b` 换行后结束；但 `return a\n + b` 是否续行需明确规则——v0.2 规定「二元运算符在行尾 → 续行」。
- 契约块与函数体都用 `{ }`，解析时 `effects/requires/ensures` 关键字先行即可区分。
- `match` arm 语法（v0.2.1 锁定）：`Pattern: { 绑定 -> body }`（见 §16；§83 #9 已锁定）。

---

## 72. 模块三：AST 设计（raw AST ↔ AILang IR）

AST 同时服务**编译器、AI、IDE**——但 raw AST 与 HIR 是两套独立类型（见 §65）。

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
pub enum ParamMode { Move, Borrow, BorrowMut, Copy }   // 4 值（mode 权威，§88 #3 / §18.3）
```

### HIR / AILang IR（AI 读取的富语义 IR，节选）
```rust
pub struct HirFn {
    pub id: DefId, pub span: Span,
    pub identity: Identity,            // { name, description(from doc) }
    pub params: Vec<HirParam>,         // 解析后类型 + meaning + mode（mode 权威，§88 #3）
    pub output: HirOutput,
    pub effects: ResolvedEffects,      // 总是完整推断集
    pub errors: Vec<HirErrorRef>,      // 从 Result<T,E> 派生
    pub ownership: OwnershipSig,       // { output: New|Move }（无 inputs——mode 在 params；无 Borrowed——§18.5 禁 borrow 返回，§88 #2/#3）
    pub body: HirBlock,
}
```

### 关键设计决策
- **`type X = Y` 按 §92 #1 三态分类**（对齐 §15.3）：
  - **裸 `type X = Y`（无 `meaning`/`constraint`）→ 透明别名（`kind: alias`）**：编译期展开，`X` 与 `Y` 同型可互转（如 `type URL = string`）。零成本。
  - **附 `meaning` → 名义 semantic（`kind: semantic`）**：与 base 不同型，底层共享 → 零成本（`type UserId = int` + `meaning UserId: "..."`）。`UserId` ≠ `int`。
  - **再附 `constraint` → semantic + constraint（仍 `kind: semantic`，`constraint` 为可选字段）**。
  - 即 **`meaning`/`constraint` 的存在触发 nominal**。
- **运算继承边界**（§92 #7；marker/operational 二分）：semantic 新型**继承 base 的 marker trait**（`Copy`/`Send`/`Sync` 随 base——`type UserId = int` 即 Copy+Send+Sync）；但**不自动继承 operational trait**（`Add`/`Sub`/`Mul`/`Eq`/`Ord`/... 须显式 `impl`，类 Rust newtype）。**唯一例外：`Display`**（自动经 base）+ 整型字面量强制（`let u: UserId = 100`）。
- **`meaning` / `constraint` 是独立声明**（§92 #5），语义分析阶段关联到对应 `type`（目标 `ident` 须引用同模块已声明 `type`）；结构化 `meaning` 权威、`///` 补充、冲突 `meaning` 胜（§83 #10、§92 #8 已锁定）。

---

## 73. 模块四：Type Checker（ail-checker）

### 职责
名字解析 + 类型推导/检查 + 语义类型 + 约束类型 + 泛型 + 从 `Result` 派生 errors。

### 关键设计
- **TypeDb**：`DefId → TypeDef`（Primitive / Struct{is_copy} / Alias{base} / Semantic{base, meaning?, constraint?} / Enum / Interface / Trait / Builtin(Result, Optional, List, Map, ...)）。`Alias` = 裸 `type X = Y`（透明，§92 #1）；`Semantic` = 附 `meaning` 或 `constraint`（二者存在即触发 nominal，§92 #1 三分支；`constraint` 为可选字段）。`Builtin(...)` 即 **lang item** 集合（`#[lang]`，编译器特判：errors 派生 / `try/catch` 解糖 / 特权 match；§86 #5）。
- **类型推断**：局部可推断；**函数参数与返回必须显式**（公共边界，AI 需明确 API）。泛型 `T` 在参数中可推断；return-only 泛型须调用点显式 `<T>`（§86 #7）。
- **语义类型检查**：`UserId` 与 `ProductId` 底层都是 `int`，但名义不同 → `get_user(product_id)` 报 `Type mismatch: Expected UserId, Found ProductId`。字面量可强制到整型语义类型；变量间不隐式转换。泛型构造器**不变**（`List<UserId>` ≠ `List<int>`，§86 #9）。
- **约束类型检查**：字面量编译期检查（`let age: Age = -10` → `Constraint violation: Age must be 0..150`）；运行期经构造算符 `T(expr)`（§92 #2）插断言，违约 panic `ConstraintViolation`（§92 #9）。
- **`is_copy` 自动判定**：所有字段为 Copy 或 COW（COW 计为 Copy-compatible，§85 #7）→ struct 为 Copy；含任一 Move/`Resource` 字段 → Move。
- **trait vs interface 派发二分**（§86 #10）：`trait` 作泛型 bound（`where { }`，§86 #2）→ **编译期单态化**；`interface` → **运行期 dyn 派发**（vtable）→ **方法禁止泛型**（§86 #1）。
- **`Send`/`Sync` auto-trait 推导**（§87 #6）：编译器按字段组合自动派生 `Send`/`Sync`（marker lang item，§86 #5）——Copy→Send+Sync、COW（原子 rc）→ Send、Move 资源→Send 非 Sync、`Mutex<T>`/`Shared<T>` 按规则；`spawn`/`Channel.send`/actor 投递边界静态查 `Send`（类 Rust，无需手写 `impl`）。
- **errors 派生**：返回 `Result<T,E>` 且 `E` 为 `error` 枚举 → variant 进入 `HirFn.errors`，进 `.ailmeta`。

### 陷阱
- 整数字面量强制规则要精确（可强制到整型 primitive 及以其为 base 的语义类型；显式 `int` 变量不隐式转 `UserId`）。
- 递归 struct / 循环 type alias 检测。

---

## 74. 模块五：Ownership + Borrow Checker（ail-ownership）

### 职责
类型检查通过后，对每个函数体做所有权数据流分析。

### 关键设计
> **Single Owner Model**：任何资源同时只有一个 owner。borrow / borrow_mut **不逃逸**（禁存 struct 字段、禁返回、禁跨 await）——这是无生命周期标注的根因。

### 74.1 所有权状态机
对每个变量按语句推进状态：
```
Uninitialized → Alive → Moved
                 ↓
              Borrowed / Frozen（被借用期间）
```
- `process(file)` → move → `print(file)` 报 **Use after move**。
- 值语义三分类：位复制 Copy（`int/uint/float/bool/byte` + 全 Copy struct）、COW 值类型（`string/List/Map/Set`，赋值共享·修改克隆）、Move（`File/Socket` 等资源，需显式 `borrow`/`borrow_mut`，不可 `copy`）。

### 74.2 Borrow Graph（无显式生命周期）
内部建立 Borrow Graph，记录每次借用的 owner / 借用方 / 生命周期（恒等于当前语句，由 Lifetime Analyzer 推导）。
- **只读借用** `borrow x`：允许同时多个；存在期间原值冻结（不可 move/mut）。
- **可写借用** `borrow_mut x`：独占——同时只能一个 mutable borrow，且不能与任何只读借用并存。

### 74.3 线程安全
自动分析跨线程移动/共享（类 Rust `Send + Sync`）：`spawn process(data)` 时检查 `data` 是否可跨线程移动，不可则报错。

### 陷阱
- **v0.2 引入 `borrow_mut`**（推翻 v0.1「仅不可变借用」），但同样不逃逸 → 仍无需生命周期。
- **Drop 顺序 + 无循环引用（§90 #5）**：字段按**声明顺序**释放（确定性）；安全代码单一所有者 + COW 缓冲共享 → 结构上无环，v0.2 无 `Weak`/`Rc`/`Arc`；codegen 按声明序插 `release`/drop，panic 展开同序（§89 #3）。
- **borrow checker 增量交付**（见 §82 #1）：**无临时内存模型**——Phase 1 即交付**所有权子集**（move 语义 + use-after-move 检测 + 确定性释放 §18.7 + 函数内 borrow/borrow_mut aliasing 检查 §74.2，覆盖非泛型 struct/fn），Phase 3 扩展为**完整 borrow checker**（泛型所有权规则 + Lifetime Analyzer 跨函数推导，依赖 Phase 2 类型系统）。每阶段交付 §18 冻结语义的真子集、永久正确、零迁移。

---

## 75. 模块六：AI Analyzer（ail-analyzer）

### 职责（AILang 特色）
读取 `meaning` / `effects` / `requires` / `ensures`，结合类型/所有权分析结果，生成**程序知识图谱（Program Knowledge Graph）**，作为 `.ailmeta` 的数据源。

### 关键设计
- 推断每个函数的完整 `effects`（callee 并集 + 自身操作）；校验用户标注（推断集 ⊆ 标注集）。`alloc` 推断仅用于 `pure` 判定，**不进 `.ailmeta` effects 数组**（隐含，§19）。
- **dyn/FFI/泛型效果上界**（§85 #4）：interface 调用取 interface 声明 effects；`extern` 默认 `effects { extern }`；泛型方法随 bound。
- **`pure`**（§85 #9）：无任何效果（含 `alloc`）+ 不改入参；触发 COW 克隆/堆分配 → 非 pure。
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

## 76. 模块七：AILang IR（ail-ir）

### 为什么需要自己的 IR？
**LLVM 不懂 AI 语义。** 直接把 AST 降到 LLVM IR 会丢失 meaning/effects/契约等 AI 信息。因此设中间层：

```
AILang AST → AILang IR (HIR) → LLVM IR
```

AILang IR 是 `.ailmeta` 的唯一数据源，也是优化、多后端的承载层。未来可加 cranelift 等后端而不动前端。

### AILang IR 形态
保留函数签名 + 完整语义标注 + 类型化函数体，但比 raw AST 更规整（已解析、已去糖：`try/catch` → Result match、`borrow` → 借用标记）。

---

## 77. 模块八：LLVM Codegen（ail-codegen）

### 职责
AILang IR → LLVM IR → 优化 → 目标文件 → 链接原生二进制。

### 类型映射
| AILang | LLVM |
|--------|------|
| `int` | `i64` |
| `int32` / `byte` | `i32` / `i8` |
| `float` | `f64` |
| `bool` | `i8`（存储）/ `i1`（寄存器）|
| `string` | `{i64 len, i8* ptr}`（COW：共享缓冲 + **原子引用计数** → `Send`，修改时克隆；§85 #2） |
| `struct` | LLVM struct（无对象头）|
| `Array<T, const N>` | 内联 `[T × N]`（固定布局，const 泛型 N；§86 #3）|
| `Box<T>` | 堆指针 `T*`（owned，Move/非 Copy，确定性释放；§86 #4）|
| `type X = Y`（裸，无 meaning/constraint） | 透明别名：展开为 base（零成本，§92 #1）|
| `type X = Y` + `meaning`/`constraint`（semantic） | 名义新型：与 base 不同型、底层共享（零成本，§92 #1）；codegen 同 base 布局 |
| `Optional<T>` | `{i1 has, T}` |
| `Result<T,E>` | `{i1 is_err, T, E}` tagged union |
| `enum` | tag + data |
| `Future<T>` | 状态机 struct（`async fn` 编译产物）；**无 `Pin`**——borrow 禁跨 await（§18.5 / §74.1）→ 不自引用（§87 #1） |
| `Channel<T>` | MPMC 队列句柄（有界 ring / 无界链）；`close` 标志 + `receive`/`try_receive`（§87 #2） |
| `TaskHandle` / `ActorHandle` | 调度器句柄（`Send`）；`cancel`/`cancelled`/`join`（§87 #4/#5/#8） |

### 所有权 / 效果 / 契约的 codegen
- **move / borrow / borrow_mut / copy**：move 无运行时代码；borrow 编译为指针（`alloca`+ptr，因不逃逸，生命周期 = 栈帧）；copy 按值复制。
- **effects / 契约 / pure**：纯编译期检查，不产生运行时代码（除非契约断言）；基本擦除。
- **泛型**：单态化（每个具体 `Result<X,Y>` 生成独立 LLVM 类型）。
- **数值边界（§90 #1/#2/#3）**：整数算术——release 直接用 LLVM `add`/`sub`/`mul`，**不**置 `nsw`/`nuw`（二补数回绕为规范行为）；debug 构建在每条算术后插溢出检测 → 命中 `panic(ArithmeticOverflow)`；整数 `sdiv`/`udiv`/`srem`/`urem` 前插除数==0 检测 → `panic(DivideByZero)`；浮点直接用 LLVM 浮点指令（IEEE 754，无检测、无 panic）。编译期常量除零 / 溢出由类型检查期报错。

### 流水线
```
AILang IR → 构建 LLVM Module → 优化 passes（内联 / 死代码删除 / 常量折叠 / SIMD）→ emit .o → 链接（lld / clang）→ native binary
```

### 陷阱
- `string` 需最小 runtime（len+ptr；**无追踪 GC**——COW/RC 引用计数 + 确定性释放，§18.4 / §85 #2）。
- **panic 展开与 FFI 边界（§89 #3、§90 #4）**：panic 经 LLVM landing pad / personality function 栈展开 + Drop；展开至 **Task 边界**（→ Task 错误态）或 **`extern` 帧**（→ **转进程 abort**：C 无 Drop、不可跨语言栈展开，**非 UB**；C 的 `longjmp` 跨 FFI 进 AILang 帧 = UB 禁）；landing pad / abort-on-FFI 须由最小 runtime 提供。
- **`async/task/spawn/channel/select` 的 codegen 需状态机 + executor——**v0.2 仅标记，不做 codegen**（调用报「not yet supported」）。语义见 §87 决议记录 V：`async fn` → `Future<T>` 状态机（**无 `Pin`**，borrow 禁跨 await，见 §18.5 / §74.1）、`Channel` = MPMC + close、`spawn` 返回 `TaskHandle`（结构化作用域 join / `detached`）、`actor` = Task + `on` 邮箱循环、`cancel` = 协作式（await 点丢弃状态机 → 确定性释放）、阻塞经 `spawn_blocking`（独立线程池，不占 M:N worker，§87 #7）。
- 错误传播：v0.2 不引入 `?`，强制 `match`，codegen 直白。

---

## 78. AI Metadata 生成

AILang IR → `program.ailmeta`（JSON，`serde`）。确定性输出（字段排序固定）、带 `schema_version`、带源码定位、完整语义。Schema 见 §24。

**`.ailmeta` 的内容是编译器验证后的事实，不是注释**：声称 `pure` 即已校验无副作用；声称 `effects { database.read }` 即已校验不写库。AI 可信任它，如同信任类型签名。

---

## 79. 工具链与命令（ail / ailc）

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

## 80. MVP 开发路线（Phase 1–4）

| Phase | 时间 | 产出 | 验证 |
|-------|------|------|------|
| 设计 | — | 哲学 + 规范(v0.2.1) + 架构 + 白皮书 + 内存/并发模型 | ✅ 完成 |
| **1** | 1–3 月 | Lexer、Parser、AST、基础类型、函数、struct、**最小 LLVM 输出** | `fn main() { print("hello") }` 编译为原生二进制并运行 |
| **2** | 3–6 月 | `Result`、`enum`、泛型、`interface`、**AI Metadata（`.ailmeta`）** | 端到端：源码 → 可信 AI 元数据 |
| **3** | 6–12 月 | **Ownership + Borrow Checker**、并发、标准库核心 | 所有权错误用例全部报出 |
| **4** | 1 年+ | 包管理、IDE 插件、Debugger、AI Agent 生态 | 生产级 |

> **关键 checkpoint**：Phase 1 证明全工具链打通（原生 hello world）；Phase 2 交付 `.ailmeta`，验证 AI-native 主张；Phase 3 补齐 Rust 级安全。

> **关键洞察**：`.ailmeta` 由 HIR 直接生成，**不依赖 LLVM codegen**——即便在 Phase 1 最小 codegen 阶段，也能独立验证「源码 → 可信 AI 元数据」这一 AILang 核心主张。Phase 2 的 `.ailmeta` 闸门即据此设计。

> **borrow checker 增量交付**（原「临时内存模型」开放缺口，**2026-07-13 已定**）：**无临时补丁**——Phase 1 即交付**所有权子集**（move 语义 + use-after-move 检测 §74.1 + 确定性释放 §18.7 + 函数内 borrow aliasing §74.2，非泛型），Phase 3 扩展为**完整 borrow checker**（泛型所有权规则 + Lifetime Analyzer 跨函数推导 §18.5，依赖 Phase 2 泛型/trait 系统）。排除 RC（move 退化违 §18.3）/ unsafe（违编译期安全卖点）。每阶段语义为 §18 真子集、永久正确（见 §82 #1）。

---

## 81. 关键设计决议汇总（v0.2）

| # | 决议 | 选择 | 代价 |
|---|------|------|------|
| ① | 代码块 | **大括号 `{}`**（非缩进） | 放弃 Python 缩进语义 |
| ② | 分号 | **可选**（Go 风格，换行分隔） | parser 需换行/续行规则 |
| ③ | 扩展名/元数据 | **`.ail` / `.ailmeta`** | — |
| ④ | `type` 语义 | **默认透明别名；附 meaning 或 constraint -> 名义 semantic（§92 #1）** | 需字面量强制规则 |
| ⑤ | 所有权简化 | **borrow/borrow_mut 不逃逸** → 无 `'a` | struct 字段不能存引用 |
| ⑥ | 可变借用 | **v0.2 支持 `borrow_mut`**（独占） | 借用检查器更复杂 |
| ⑦ | 效果策略 | **推断为主，标注为约束** | 调用图不动点 |
| ⑧ | 泛型约束 | **`where { Bound<T> }`**（编译期 trait bound）+ `requires { }`（运行期断言）（§86 #2） | 不复用 Rust `<T: Trait>` 语法 |
| ⑨ | interface/trait | **interface=服务能力（dyn 派发，方法禁泛型），trait=类型能力（bound→单态化），`implements`**（§86 #1/#10） | dyn 与泛型方法不可兼得 |
| ⑩ | LLVM 引入 | **Phase 1 即最小 codegen**（全链路打通） | 早期 codegen 受限 |
| ⑪ | IR 分层 | **AILang IR (HIR) 独立于 LLVM IR** | 多一层 lowering |

---

## 82. 开放问题（工程实现层）

> **语言设计层面的开放问题已全部在 §83 决断**（见决议记录）。本节仅列**编译器实现层**的遗留工程决断——非语言语义，而是「如何实现」的工程取舍。

### 已决（语言层 → §83）

| 本节原 # | 主题 | 决议 |
|---|---|---|
| #2 | `try/catch/throw` 解糖 | §83 #4、§89 |
| #3 | `interface` / `trait` 边界 | §83 #1、§86 #1/#10 |
| #6 | `match` arm 形式 | §83 #9 |
| #7 | `meaning` / `constraint` 独立声明 | §83 #10、§92 #1/#5/#8/#9 |
| #8 | `string` / 集合内存模型（COW） | §83 #7 |
| #10 | 关键字精确集合（56） | §83 #5 |

### 遗留（工程实现层）

1. **Phase 1 所有权子集（已定，2026-07-13）**：borrow checker **增量交付**，无临时模型——Phase 1 交付**所有权子集**（move 语义 + use-after-move 检测 + 确定性释放 §18.7 + 函数内 borrow/borrow_mut aliasing §74.2，非泛型），Phase 3 扩展**完整 borrow checker**（泛型所有权 + Lifetime Analyzer，依赖 Phase 2 类型系统）。排除 RC（move 退化违 §18.3）/ unsafe（违安全卖点）。每阶段语义为 §18 真子集、零迁移。=> 体现在 §80 / §74。
2. **可选分号的换行/续行规则边界用例**：二元运算符行尾、`return` 表达式跨行等——需 Parser 实现时以测试集敲定。
3. **Constraint / 契约证明的实现深度**：语言层已分级锁定（§83 #3，v0.2 仅断言）；实现层待定 v0.3+ 是否集成 SMT。
4. **`async/task/spawn/channel/select` 的 codegen 与运行时**：模型已指定（§83 #8，M:N work-stealing）；状态机 + executor 实现后置 Phase 3–4。

<a id="part-v"></a>
# Part V · 决议记录

## 83. 决议记录 I（v0.2.1 收敛）

> 下列原「开放问题」已于 v0.2.1 逐条决断；均**可逆**。本节为**紧凑索引**——权威表述已迁入正文（§15–§27），每项给一行决断摘要 + 正文指针。

1. **`interface`（dyn 派发、不可作泛型约束）与 `trait`（可作泛型约束、单态化）二分保留**——根因为 dyn vs 单态化（取代早期「服务 vs 行为」措辞，§86 #10）。=> 体现在 §15.11。
2. **外部类型实现 trait = v0.3 计划**：v0.2.1 仅内联 `implements` + orphan 规则（§91 #9）；v0.3 引入独立 `impl Trait for Type { }` 块。=> 体现在 §15.5 / §15.11。
3. **Constraint / 契约证明分级**：v0.2 字面量编译期检查 + 构造运行期断言 + `ensures` `old()` 快照；自动 SMT 证明留 v0.3+。=> 体现在 §15.4 / §20。
4. **`try/catch/throw` 解糖为 `Result` + `match`**（`try expr` 前缀一元、同错误类型、`throw` 语句级 never、显式 `Ok`、无 `finally`）。=> 体现在 §17 / §27。
5. **关键字计数 = 56**；`true`/`false`/`void`/`never` 为保留字面量/类型词，不计入 56。=> 体现在 §9。
6. **顶级声明边界锁定**：struct / actor / interface / trait / agent / tool / server / route 各自独立构造。=> 体现在 §23 / §27。
7. **`string` / 集合 = COW 值语义**（写时复制、原子 rc、行为同 Copy；资源型 move-only；无追踪 GC）。=> 体现在 §18.4。
8. **async 运行时 = M:N work-stealing，内建 `std.async`**（Channel = **MPMC**、`Future<T>` + 无 Pin、spawn 返回 TaskHandle + 结构化作用域、actor `on` handler、`spawn_blocking`）；v0.2 仅语义，codegen 后置。=> 体现在 §21。
9. **`match` arm 形式锁定**：`Pattern: { 绑定 -> body }` / `Pattern: { body }` / `Pattern: { a, b -> body }` / `_: { }`。=> 体现在 §16。
10. **`meaning` / `constraint` 独立声明 + `///` 双轨**（结构化权威、`///` 补充、按位置优先级填充）。=> 体现在 §8.5 / §15.3。

> 至此 v0.2.1 无未决开放问题；全部决议可逆。深审（EXAMPLES ↔ SPEC 语义级）14 项见 §84。

---

## 84. 决议记录 II：深审语义收敛（v0.2.1）

> §83 收敛后对 EXAMPLES ↔ SPEC 做了一轮**语义级深审**，发现 14 项语义矛盾 / 未定义点。下列已于 v0.2.1 内决断并就地传播；均**可逆**。本节为**紧凑索引**——权威表述在正文，每项一行摘要 + 指针。

1. `[示例]` **字面量强制到整型语义类型，作用于所有强制位**（绑定 + 调用实参，非仅 `let`）。=> 体现在 §15.3。
2. `[文法]` **payload-less `message` 合法**（`message Ident` 与 `message Ident { fields }`，镜像 enum variant）。=> 体现在 §27。
3. `[示例]` **§26 `get_user` 类型自洽**（边界抽取单值 + `Error → 命名错误` 转换；`query<T>` 后改非泛型 + `decode<T>` 边界解码，§86 #1）。=> 体现在 §26 / §15.11。
4. `[示例]` **`id: UserId = int` 为 Copy**（元数据标 `copy`）；interface dyn 句柄为 `borrow`。=> 体现在 §18.4 / §26。
5. `[语义]` **`task`（void body + CSP 通道，经 channel/cancel 通信、不强制 await）与 `async fn`（`Future<T>` + `await`）区分**；「须返回 Result」仅约束用户显式 await 的 async fn（取代早期 fire-and-forget 措辞，§87 #4）。=> 体现在 §21。
6. `[定义]` **`std.core.Error` 为动态边界 opt-in 通用错误**；惯用代码用命名 `error` 枚举；边界处 `Error → 命名错误` 显式映射。=> 体现在 §17。
7. `[定义]` **Channel 为共享 `Send` 句柄**；`spawn { }` / `parallel for` / `lock(m) { }` / `server route` 块体可捕获外层不可变 / `Send` 变量（非通用闭包，v0.2 无一等闭包）。=> 体现在 §21。
8. `[定义]` **server/route handler 捕获外层 `Send` 句柄**（控制构造，#7 之推论）。=> 体现在 §21。
9. `[定义]` **`print` 为 `std.core` 默认内建，无需 import**（类 Go `println`）。=> 体现在 §12.1 / §34。
10. `[示例]` **§40 裸 `Result` 改具型 `List<SearchResult>`**（`SearchResult` 声明于 `std.ai`）。=> 体现在 §40。
11. `[示例]` **`parallel for` 许可标准 = body 须 `pure` 或仅 `io.read`、且无共享可变**（效果门由 §87 #9 锁定）。=> 体现在 §21.10。
12. `[文法]` **`agent` 文法 `tools` 子句 `?`（至多一次）**。=> 体现在 §27。
13. `[语义]` **调用点 `borrow`/`borrow_mut`/`move`/`copy` 可选**（省略时按被调形参模式推断）。=> 体现在 §18.3。
14. `[语义]` **`Request.param<T>(name: string) -> T`**（HTTP 边界解析 + 强制）；`let id: UserId = req.param("id")` 合法。=> 体现在 §63。

> 至此深审 14 项全部决断并就地传播；v0.2.1 语义层自洽。深审 III（所有权 + 效果系统内在矛盾）10 项见 §85。

---

## 85. 决议记录 III：所有权 + 效果系统深审收敛（v0.2.1）

> §84 收敛后换维度做了一轮**所有权/效果模型内在自洽性**深审，发现 10 项定义空白 / 语义矛盾。下列已决断并就地传播；均**可逆**。本节为**紧凑索引**——权威表述在正文（#1 接收者模式为 #3/#5/#8/#9 及「无共享可变」判定的共同前置，单一权威处见 §15.5）。

1. `[定义]` **方法接收者模式（拱顶石）**：复用形参关键字 `borrow self`（默认）/ `borrow_mut self` / `self`（消费）；省略 = `borrow self`。=> 体现在 §15.5。
2. `[定义]` **COW 原子引用计数 → `Send`**；「无 GC」修订为「无追踪 GC；COW/RC 引用计数仍存，确定性释放」。=> 体现在 §18.4 / §18.6。
3. `[语义]` **`borrow_mut` 要求 COW 唯一所有权（rc == 1）**，突变原地；`borrow_mut` = opt-out COW 克隆。=> 体现在 §18.5。
4. `[语义]` **效果健全边界**：interface 方法必带 `effects`（dyn 取声明上界）/ `extern` 默认 `effects { extern }`（可收窄）/ 泛型随 trait bound 传播 → 「推断集 ⊆ 标注集」可强制。=> 体现在 §19。
5. `[示例/语义]` **`withdraw` 契约自洽**：`account: borrow_mut Account`（使 `ensures` 对 caller 可观测），`old()` 为调用前快照。=> 体现在 §20。
6. `[定义]` **interface 值为不可 Copy 的 Move 句柄**（vtable + 指针），经 `borrow` / `Shared<T>` / 连接池共享。=> 体现在 §15.11。
7. `[定义]` **`is_copy` 含 COW 字段**：COW 字段计 Copy-compatible → 全 Copy/COW 字段 struct 为 Copy，含任一 Move/Resource 字段则 Move。=> 体现在 §18.4。
8. `[定义/示例]` **`Resource.release(self)` 消费签名**（move 接收者），编译器在 owner 作用域末插入。=> 体现在 §18.7。
9. `[语义]` **`pure` 禁分配 + `alloc` 效果**：`pure` = 空效果集（含 `alloc`）且不改入参。=> 体现在 §19。
10. `[文法/语义]` **`throw` 语句级 + bottom（`never`）类型**，不可作子表达式（与 `return` 同）。=> 体现在 §17。

> 至此深审 III 的 10 项全部决断；v0.2.1 所有权 + 效果系统内在自洽。深审 IV（类型系统 + 泛型单态化）10 项见 §86。

---

## 86. 决议记录 IV：类型系统 + 泛型深审收敛（v0.2.1）

> §85 收敛后再换维度做了一轮**类型系统 / 泛型单态化内在自洽性**深审（聚焦「dyn 派发 vs 单态化」断层），发现 10 项定义空白 / 矛盾。下列已决断并就地传播；均**可逆**。本节为**紧凑索引**（#10 trait→单态化 / interface→dyn 二分是 #1 的根因）。

1. `[语义]` **interface 方法禁泛型**（dyn vtable 无法为每个 `T` 预留表项）；`Database.query` 非泛型 + `decode<T>` 边界解码。=> 体现在 §15.11。
2. `[文法/语义]` **`where { Bound<T> }`（编译期 trait bound）与 `requires { }`（运行期断言）拆分**；`ensures` 后置不变。=> 体现在 §15.10 / §20 / §27。
3. `[文法/示例]` **const 泛型 `Array<T, const N: int>`**（`layout(C)` 固定布局的必需；codegen 内联 `T × N`）。=> 体现在 §15.9 / §25.4 / §27。
4. `[定义]` **递归类型须显式 `Box<T>` 间接**（owned 堆、Move、非 Copy、确定性释放）。=> 体现在 §15.6。
5. `[定义]` **lang item（`#[lang]`）**：`Result`/`Optional`/`List`/`Map`/`string` 等被编译器特判的 std.core 类型。=> 体现在 §15.8。
6. `[定义/语义]` **trait 默认方法 + 覆写**：trait 方法带 body = 默认实现；`implements Trait { }` 可留空继承默认。=> 体现在 §15.5。
7. `[文法/语义]` **调用点显式泛型实参 `foo<T>(args)` + return-only 泛型须显式标注**。=> 体现在 §15.2 / §27。
8. `[定义]` **运算符重载 = std.core 运算符 trait 解糖**（`a + b` → `Add.add(a,b)` 等；泛型 bound 复用 `where { Ord<T> }`）。=> 体现在 §11 / §15.10。
9. `[定义]` **泛型不变性（invariance）**：`List<UserId>` 与 `List<int>` 无子类型关系（nominal）。=> 体现在 §15.3 / §15.10。
10. `[定义]` **trait bound → 编译期单态化；interface → 运行期 dyn 派发**（#1 的根因）。=> 体现在 §15.11。

> 至此深审 IV 的 10 项全部决断；v0.2.1 类型系统 + 泛型内在自洽。深审 V（并发 executor / async 状态机）10 项见 §87。

---

## 87. 决议记录 V：并发 executor / async 状态机深审收敛（v0.2）

> §86 收敛后再换维度做了一轮**并发运行时语义 / async 状态机**深审（聚焦 executor、Future、取消、Channel、Actor 行为），发现 10 项矛盾 / 定义空白。下列已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 async 返回 + 无 Pin 是相对 Rust async 的核心简化，#2 修正 Channel 模型，#5 补 Actor 行为）。
>
> **关键字计数**：本轮新增并发控制形式（`on` / `detached` / select `default`/`timeout` / `parallel reduce` / `spawn_blocking` / `yield`）均为**上下文关键字或 `std.async`/`std.sync` 函数**，不计入 §83 #5 的 56 全局关键字（同 `lock` 先例）。

1. `[语义/矛盾]` **`async fn f() -> T` 的 `T` 即 `await` 值**（`Future<T>` lang item）；「须返回 Result」仅约束用户显式 `await` 的 async fn，runtime 派发 handler 豁免；**状态机无 Pin**（borrow 禁跨 await，规则在 §18.5 / §74.1）。`task` 不可 async（void body + CSP 通道，经 channel/cancel 通信）。=> 体现在 §21。
2. `[定义/矛盾]` **Channel 改 MPMC**（取代 §83 #8 早期 MPSC）+ 无界/有界（`cap=0` rendezvous）+ `close` + `receive`/`try_receive`（均 `-> Optional<T>`，关闭耗尽 `None`）+ select 关闭检测。=> 体现在 §21。
3. `[文法]` **`await` 为一元前缀**（Pratt level 6，`await f()` / `await fut.field`；§27 `unary`）。=> 体现在 §11 / §27。
4. `[定义]` **`spawn` 返回 `TaskHandle` + `cancel` 操作数为 TaskHandle 表达式 + 结构化并发作用域**（默认 scoped、作用域末隐式 join；`spawn detached` 显式分离——**非** fire-and-forget；`cancel` 向子树传播）。=> 体现在 §21。
5. `[定义]` **Actor `on` 子句 + `ActorHandle`**（`on Msg { msg -> body }`，`spawn actor Name(...)` 返回 `ActorHandle`，`h.send`/`h !` 投递，`self.state` 串行访问）。=> 体现在 §21。
6. `[定义]` **`Send`/`Sync` 为自动派生 marker trait（lang item）+ 组合规则**：`Send` 当且仅当所有字段 `Send`、`Sync` 当且仅当所有字段 `Sync`；Copy → Send+Sync、COW → Send、Move 资源 → Send 非 Sync、`Mutex<T>`/`Shared<T>` → Send+Sync。=> 体现在 §18.6。
7. `[语义]` **阻塞调用与 executor**：`async fn` 内禁 `io.blocking`，阻塞经 `spawn_blocking { }`（独立 OS 线程池）或 async 变体；新增 effect `io.blocking`。=> 体现在 §21 / §19。
8. `[语义]` **取消语义**：协作式（仅 `await` 点生效，CPU-bound 须 `task.yield()`），丢弃状态机 → 确定性释放 locals，`lock(m){ }` 边界自动释放锁，`cancel` 向子树传播。=> 体现在 §21。
9. `[语义]` **`parallel for` 表达式化（返回 `List<R>`）+ `parallel reduce(op, xs)`**（`op` 须结合律；body 须 `pure` 或仅 `io.read`、无共享可变）。=> 体现在 §21.10。
10. `[语义]` **select 扩展**（`receive`/`send`/`await` 分支 + `default { }` + `timeout(d) { }`；多分支就绪随机公平择一；关闭 channel 的 receive 立即可就绪，交付 `None`）。=> 体现在 §21 / §27。

> 至此深审 V 的 10 项全部决断；v0.2 并发 executor / async 状态机语义自洽。深审 VI（std API 一致性 / `.ailmeta` schema 完整性）10 项见 §88。

---

## 88. 决议记录 VI：std API 一致性 / `.ailmeta` schema 深审收敛（v0.2）

> §87 收敛后再换维度做了一轮**标准库 API 一致性 + AI 元数据 schema 完整性**深审——`.ailmeta` 是 AILang 存在的全部理由（§24），却四处样例用了 ≥3 种 schema 形、且无正式字段规范。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 schema 统一 + #2 正式 schema 规范为 keystone；多数权威表述落 §24 / §33–§40，#6 effect 词表落 §19）。

1. `[一致性/矛盾]` **`.ailmeta` schema 四处样例统一为 §24 权威富形**（§32 / §63 / §55 对齐）。=> 体现在 §24。
2. `[定义]` **正式 `.ailmeta` schema 规范**：字段表 + 枚举（`mode` / `ownership.output` / `kind`）+ `schema_version` 独立 semver。=> 体现在 §24。
3. `[一致性]` **`input[].mode` 权威；`ownership` 删 `inputs[]` 仅留 `output`**（消除双重编码）。=> 体现在 §24。
4. `[完整性]` **`.ailmeta` 增 `declarations[]`**（agent/tool/actor/server/task 顶级声明容器；`tool` 派生 OpenAI Tool Schema）。=> 体现在 §24。
5. `[完整性]` **`.ailmeta` 每条目增 `span` 源码定位**（`span{file,line_start,col_start,...}`，与编译器 Span 一致）。=> 体现在 §24。
6. `[定义]` **effect 词表规约 `<domain>.<verb>`**（`database.read|write` / `network` / `io.*` / `filesystem.*` / `alloc` / `extern` / `unsafe`；`pure` = 空集）。=> 体现在 §19。
7. `[完整性]` **std 方法签名表（含接收者模式）**：为 `List`/`Map`/`Set`/`string`/`Optional`/`Result` 出签名表。=> 体现在 §34/§35/§36。
8. `[一致性]` **样例引用类型归属**：`Request`/`Response`/`URL`→std.http、`Row`→std.database、`SearchResult`→std.ai。=> 体现在 §38/§39/§40。
9. `[完整性]` **空白 std 模块占位**（math/io/net/crypto/time）；`print` 归 std.core、`std.io` 为流式 IO。=> 体现在 §33。
10. `[一致性]` **`print(x: Display) -> void` 签名 + `decode<T>` 归 std.database + 错误枚举按模块编目**。=> 体现在 §34/§37/§38/§39/§40。

> 至此深审 VI 的 10 项全部决断；v0.2 标准库 API 一致性 + `.ailmeta` schema 完整性收敛。深审 VII（错误处理模型精确语义）10 项见 §89。

## 89. 决议记录 VII：错误处理模型深审收敛（v0.2）

> §88 收敛后再换维度做了一轮**错误处理模型精确语义**深审——`try/catch/throw` 解糖虽于 §83 #4 锁定骨架，但文法悬空、值构造未定、panic 模型缺失、错误转换/并发传递未落到精确语义。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 try/catch 文法 + #3 panic 模型为 keystone）。

1. `[文法/完整性]` **`try expr` 为前缀一元（与 `await` 一致，§27 `unary`）+ `try { } catch { }` 捕获形式**（新增 `try_catch` 产生式，按下一 token 消歧）；拒后缀 `expr try` 与 sigil `?`；56 不变。=> 体现在 §17 / §27。
2. `[文法/语义]` **`Result`/变体值构造复用 call 文法**（名字解析消歧），裸变体 = ident；成功返回须显式 `Ok(x)`、无 auto-wrap。=> 体现在 §17 / §27。
3. `[定义]` **panic = 确定性栈展开 + Drop/release，展开至 Task 边界**（不传染进程）；v0.2 无 `catch_unwind`；`unwrap`/`assert` 失败 = panic，`std.core.panic` 为 std 函数（非关键字）。=> 体现在 §17。
4. `[定义]` **`⊥` = `never`（空类型，内建 lang item）**：可作返回类型，`never | T = T`；`never ≠ void`；为保留类型词，不计 56。=> 体现在 §15.1 / §17 / §9。
5. `[语义]` **错误转换无 `From`**：`try expr` 要求同错误类型（`E1≠E2` 类型错）；转换唯一手段 = `match { 错误分支 → throw 新变体 }`。=> 体现在 §17。
6. `[语义]` **`throw` 合法性**：仅合法于返回 `Result<T,E>` 且 `V∈E` 的体；`void`/`task`/actor `on`/server handler 禁 throw（错误经各自通道）。=> 体现在 §17。
7. `[定义]` **`error` 变体可带 payload**（与 enum 同形 `Variant(field: T)`）；`.ailmeta` `errors[]` 每项可选 `payload`；惯法推荐裸变体。=> 体现在 §17。
8. `[一致性]` **并发错误传递统一表**：`async fn -> Result` 可 throw/try；`task` 禁 throw（经 channel/cancel）；actor `on` 禁 throw（经 `reply(Err...)`，reply 值可 Result）；server handler ✗ 禁 throw（豁免于须返回 Result，runtime 派发）。=> 体现在 §21。
9. `[语义]` **`pure fn` 可 throw/try**（throw 非 effect、不改入参）；`is_pure` 仅看 effect 集与入参突变，与 `errors[]` 正交。=> 体现在 §17 / §19。
10. `[完整性]` **`Result<void,E>` 成功构造 = `return Ok(void)`（显式，无 auto-wrap）；`errors[]` 单一真源**（≡ 签名 `E` 变体，throw/try 永不越界；非 Result 返回 → `errors:[]`）。=> 体现在 §17。

> 至此深审 VII 的 10 项全部决断；v0.2 错误处理模型精确语义收敛。深审 VIII（数值语义 + 逃生舱边界）5 项见 §90。

---

## 90. 决议记录 VIII：数值语义 + 逃生舱边界深审收敛（v0.2）

> §89 收敛后做了一轮**全面性能/安全复核**，暴露 5 个未被前 7 轮覆盖的真实语义缺口：整数溢出、浮点 NaN/特殊值、除零、`unsafe` 边界精确清单、Drop 语义细节（外加 panic 跨 FFI，并入 #4）。下列 5 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 整数溢出为 keystone）。

1. `[定义]` **整数算术溢出 = 定义行为，永不 UB**：release 二补数回绕（规范），debug 溢出 → panic `ArithmeticOverflow`（开发期检测层）；`std.math` 提供 `wrapping_*`/`checked_*`/`saturating_*`/`overflowing_*` 运算符族。=> 体现在 §15.1 / §17。
2. `[定义]` **浮点遵循 IEEE 754**：`x/0.0`→`±inf`、`0.0/0.0`→`NaN`、NaN 经运算传播、`NaN==NaN` 为 `false`，**浮点运算永不 panic**；浮点 NaN 检测为 **float 方法** `x.is_nan()`/`x.is_inf()`/`x.is_finite()`（非 `std.math` 自由函数，§94 #4）。=> 体现在 §15.1。
3. `[语义]` **除零语义**：整数除零/取模零 → panic `DivideByZero`（确定性）；浮点除零遵循 IEEE（不 panic）；`std.math.checked_div` 安全除法。=> 体现在 §15.1 / §17。
4. `[定义]` **`unsafe` 边界精确清单**（裸指针解引用 / 调 `extern` fn / 调 `unsafe` fn；不引入 `static mut`、无 union；清单外操作非 unsafe）+ **panic 跨 FFI 边界 → 进程 abort**（非 UB；C 的 `longjmp` 跨 FFI = UB 禁）。=> 体现在 §25 / §17。
5. `[定义]` **Drop 语义精确化**：字段 Drop 顺序 = 声明顺序；安全代码结构上无循环引用 → 不引入 `Weak`/`Rc`/`Arc`（共享用 Actor/`Mutex<T>`/`Shared<T>`）；panic 展开时同序 Drop；自定义释放经 `Resource.release(self)`。=> 体现在 §18.7。

> 至此深审 VIII 的 5 项全部决断；v0.2 数值语义 + 逃生舱边界收敛。深审 IX（模块系统 / 可见性 / 导入 / 包语义）10 项见 §91。

---

## 91. 决议记录 IX：模块系统 / 可见性 / 导入 / 包语义深审收敛（v0.2）

> 前 8 轮覆盖语法/类型/所有权/泛型/并发/std+schema/错误/数值+逃生舱，唯独**模块系统**——6 个关键字（`package`/`import`/`from`/`as`/`public`/`private`）出现在几乎每个样例、却无任何专门规范；且 §27 引用 `module_path`/`visibility` 两非终结符无产生式，§12 与 EXAMPLES 在 `std.` 前缀上真实矛盾。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 文法 + #3 Go 式包模型为 keystone，阻塞 Phase 1）。

1. `[文法]` **补 `module_path := ident ("." ident)*` 与 `visibility := "public" | "private"` 产生式**；`module := "package" module_path import* item*`（包名升级为点分路径）。=> 体现在 §27。
2. `[定义]` **可见性粒度 + 默认 private**：`public`/`private` 修饰所有顶级 item + struct/enum 字段 + 方法；trait/interface 方法不单独修饰（随类型）；跨模块仅可访问 `public`。=> 体现在 §22。
3. `[定义]` **`package <dotted-path>` Go 式包模型**：同一 package 声明可跨多文件共享命名空间；文件路径须与声明模块路径一致（编译期校验）；根包名 = `ail.toml.name`。=> 体现在 §12.1。
4. `[一致性]` **导入路径必含根包**（标准库根 `std` → `import std.http`，`std.` 必需）；**`std.core` 自动加载（免 import，§84 #9）**，其余 `std.*` 显式 import；第三方库 `import <pkg>.<module>`。=> 体现在 §12.1 / §12.2。
5. `[语义]` **导入冲突 = 编译错误（须 `as` 消歧）/ 本地定义优先遮蔽（+ warn）/ `import X as Y` 别名 / 禁 `import *` 通配**。=> 体现在 §12.2。
6. `[定义]` **`from M import name` 仅可导入 M 的 public 顶级 item**（导入 private = 编译错误）。=> 体现在 §12.2。
7. `[语义]` **v0.2 不支持再导出**（无 `public import` / `pub use`）；要 `b` 的 item 须显式 `import b`。=> 体现在 §12.3。
8. `[语义]` **循环导入**：类型级 / 值级 OK（惰性绑定）；顶层初始化循环（`const`/全局求值成环）= 编译错误；须两遍分析。=> 体现在 §12.3。
9. `[定义]` **orphan 规则**：`Trait` 或 `Type` 之一须定义于当前 package（类 Rust coherence）；`public` 类型 impl 本 package `public` trait 可跨模块可见。=> 体现在 §15.5 / §15.11。
10. `[完整性]` **`ail.toml.name` = 根包名 = 导入根前缀**；依赖 `dep = "1.2.3"` 安装后 `import dep.<module>`；版本遵循 semver；包名禁与 `std` 冲突。=> 体现在 §12.1。

> 至此深审 IX 的 10 项全部决断；v0.2 模块系统收敛。深审 X（语义类型 / 约束类型 / `meaning` 传播 / 约束求值）10 项见 §92。

---

## 92. 决议记录 X：语义类型 / 约束类型 / meaning 传播 / 约束求值深审收敛（v0.2）

> 前 9 轮覆盖语法/所有权/泛型/并发/std/错误/数值/模块，唯独 **AILang 立身之本——「AI First 类型系统」**（语义类型 / 约束类型 / `meaning` 传播 / 约束求值）作为连贯子系统从未专门深审。下列 10 项已决断并就地传播；均**可逆**。本节为**紧凑索引**（#1 type X=Y 分类 + #2 约束构造算符为 keystone）。

1. `[定义/文法]` **`type X = Y` 三义性裁定**：无 `meaning`/`constraint` → 透明别名（`kind: alias`）；附 `meaning` → 名义 semantic（`kind: semantic`）；附 `constraint` → semantic + constraint。即 meaning/constraint 的存在触发 nominal。=> 体现在 §15.3。
2. `[语义]` **约束类型构造算符 `T(expr)`**（复用 call 文法，名字解析消歧）：运行期断言谓词，违约 panic `ConstraintViolation`（非 `Result`，不入 `.ailmeta errors`）；字面量走编译期折叠检查。=> 体现在 §15.4。
3. `[语义/完整性]` **`meaning` 按名义类型附着传播**：带 meaning 的 semantic 值在 `.ailmeta` 各处携带；函数 output.meaning = 返回类型 meaning（函数级不可另写）；bare 基础类型 / alias = null。=> 体现在 §15.3。
4. `[定义]` **约束谓词语言**：`value` 魔法只读绑定 + 纯布尔表达式（可 `value.field` / `value.method()` / 调 `pure fn`）；禁 `old()` / 副作用 / IO；谓词须 pure。=> 体现在 §15.4。
5. `[文法]` **`meaning`/`constraint` 文法**（目标 ident 须引用同模块已声明 `type`；`meaning` 块形支持多行）。=> 体现在 §27。
6. `[一致性]` **Constraint Type = Semantic Type + `constraint`**（特化，非独立 kind；`.ailmeta` 仍 `kind: semantic` + constraint 字段）。=> 体现在 §15.4。
7. `[语义]` **semantic 运算继承边界（marker vs operational 二分）**：**标记 trait（`Copy`/`Send`/`Sync`）自动继承 base**；**运算 trait（`Add`/`Eq`/`Ord`/...）不自动继承**（v0.2.1 无给语义新类型实现运算 trait 的语法，为 v0.3 计划）；例外：`Display` 自动经 base + 字面量强制。=> 体现在 §15.3。
8. `[一致性]` **`meaning` 双轨优先级统一**：type meaning = `meaning` 声明权威（type 上 `///` → description，冲突 `meaning` 胜）；function / param / field meaning = 仅 `///`；字段 meaning 优先字段类型 meaning。=> 体现在 §8.5 / §15.3。
9. `[完整性]` **约束违约 panic 名 `ConstraintViolation`**（与 `ArithmeticOverflow` / `DivideByZero` / `IndexOutOfBounds` 同族；构造算符违约即此；56 关键字不变）。=> 体现在 §17 / §15.4。
10. `[一致性/完整性]` **`type X = Y` 复杂/不透明体 kind**：无 meaning = alias（base 可不透明，记规范化类型串）；有 meaning = semantic；`...` 仅示例简写非真实语法。=> 体现在 §15.3。

> 至此深审 X 的 10 项全部决断；v0.2 语义/约束类型收敛。全部决议可逆，记录于此供复盘。跨文档一致性终审收敛（五轮 workflow 迭代）见 §93（四审）、§94（五审）。

---

## 93. 决议记录 XI：跨文档一致性终审收敛（v0.2.1）

> §83–§92（记录 I–X）逐子系统深审收敛后，又跑四轮**全文档集一致性 workflow**（一审→四审，每轮 12 维度审查 + 对抗式验证），逐轮递减暴露跨切面残留（一审多 → 二审 13 → 三审 8 → 四审 4 unique / 0 HIGH）。下列 4 项为四审终验后落定的跨文档决断与机械修复；均**可逆**。本节为**紧凑索引**（#1 §27 零悬空 + #2 ownership 删死值为决断，#3/#4 为机械一致性修复）。

1. `[文法/决断]` **§27 零悬空产生式**：`send`（actor 投递糖 `h ! Msg`）接入 expr 链（`expr := assign | send`，§87 #5）；`try_catch`（§89 #1）/ `select`（§87 #10）接入 stmt（与 `match` 同层）。决断：§27 所有产生式须从起符可达——`async`/`await`/`where`/`spawn actor` 既已最小接线，`send`/`try_catch`/`select` 不再悬空例外。代价：`!` 二元 infix（投递）vs unary 前缀（逻辑非）、`"try {"`（捕获）vs `"try" expr`（unary 传播）按下一 token 消歧。=> 体现在 §27。
2. `[schema/决断]` **`ownership.output` 删 `borrowed` 死值** → `new | move`：§18.5 禁 borrow 返回（核心不变量，5 处复述）→ `borrowed`（借用返回）不可达，与 schema 自相矛盾；删之以消解 §88 #2（深审 VI）遗留四轮的 schema-vs-不变量矛盾。代价：schema 枚举变更（无函数依赖该值）。=> 体现在 §24 / §72 / §88 #2。
3. `[一致性]` **§71 spawn 镜像同步 `"actor"?`**（注释 `§87 #4/#5/#9`）：M2「spawn 三形式」更新 §27 后，§71 近乎逐字镜像须同步。=> 体现在 §71。
4. `[一致性]` **§71 try/catch 措辞「表达式级捕获」→「局部捕获」**：对齐权威术语（§17 / §27 `try_catch` / §89 #1 + EXAMPLES 一致用「局部捕获」）。=> 体现在 §71。

> 至此四轮一致性终审收敛；v0.2.1 全文档集自洽——§27 零悬空产生式、ownership schema 零死值（五审续见 §94）。全部决议可逆，记录于此供复盘。

---

## 94. 决议记录 XII：跨文档一致性五审收敛（v0.2.1）

> 记录 XI（§93）四审终验后，又跑第五轮**全文档集一致性 workflow**（一审→五审，12 维度审查 + 对抗式验证）。收敛轨迹：一审多 → 二审 13 → 三审 8 → 四审 4 → **五审 6**（0 HIGH）。五审回升（4→6）因重读 §27 邻域 + 审 §93 新内容，暴露 2 处四审自引入回归 + 4 处预存缺陷——前四轮只查产生式**可达性**、漏查**引用正确性**。下列 6 项五审决断（#1–#6，#3 为决断，余为缺陷清理：#1/#2 修四审回归，#4/#5/#6 对齐权威）；均**可逆**。本节为**紧凑索引**。**#7 为发布后 12 维全审 R3 追加决断（EFF-1），非五审项**。

1. `[回归/一致性]` **§24 裸 `§18.5` → 补文档前缀**：四审 C1「ownership.output 删 borrowed」新增「无 borrowed——§18.5 禁 borrow 返回」注解时漏带跨文档前缀（同句 §88 #2 已带，段内不对称）。合并建单一文档后已统一去全部跨文档前缀，§24 现为裸 `§18.5`，本条保留 v0.2.1 决断原貌。=> 体现在 §24。
2. `[回归/一致性]` **§93 #2「体现在」指针矫正**：四审 sink §93 记录 XI 时，#2 落点误植 `§71`（实为 `§72`——ownership.output 的 OwnershipSig HIR 结构在 §72 模块三 AST，非 §71 Parser）＋ `§88 #2` 漏 SPEC 前缀（F4 跨文档分布）。一行两错。=> 体现在 §93 #2。
3. `[文法/决断]` **§27 spawn 引用矫正** `call` → `postfix`：spawn 产生式 `( call | block )` 的 `call`（§27 定义为纯调用后缀 `(<…>) | (args)`，不含被调者）无法归约 `spawn download()` / `spawn actor Name()`（须 `primary + call = postfix`），与 §21/§8/CONCURRENCY「均合法（§27）」矛盾——前四轮查可达性漏此引用错。改 `( postfix | block )`（与 `send := postfix "!" expr` 一致），§71 镜像同步。代价：`postfix` 允许 `spawn x.y()`（方法引用 spawn，合理）、裸值（语义分析拒，骨架可忍）——合理过生成；`call` 仍被 `postfix` 引用，零悬空不破。=> 体现在 §27 / §71。
4. `[一致性]` **§47 `math.is_nan` → `x.is_nan()` 方法形**：浮点 IEEE 754 特殊值检测（is_nan/is_inf/is_finite）为 float 方法（`is_nan(borrow self)`，§33.1），非 std.math 模块级自由函数；原与同列真自由函数 `math.checked_add` 混用模块前缀。整数（checked_add 等）与浮点（is_nan 方法）拆分表述。=> 体现在 §47 / §33.1。
5. `[一致性]` **§24 type 分类两分支 → 三分支**：原「无 meaning→alias / 附 meaning→semantic」漏 constraint 触发分支，致 constraint-only `Age`（无 meaning）按字面应判 alias，与 §48 实际发射 `kind:semantic` 矛盾。补全对齐 §92 #1：无 meaning/constraint→alias；附 meaning 或 constraint→semantic。=> 体现在 §24 / §92 #1。
6. `[一致性]` **§56 注释「无 GC」→「无追踪 GC」**：确定性释放代码注释残留裸「无 GC」，与 §85 #2 决断「『无 GC』修订为『无追踪 GC；COW/RC 引用计数仍存，确定性释放』」不一致（全集正式处均已改）。=> 体现在 §56 / §85 #2。
7. `[效果/决断]` **`alloc` 为隐含效果**：发布后 12 维全审 R3 发现 §19/§75 原规定「效果推断含 `alloc`」与「`.ailmeta` 输出完整集」相悖——堆分配普遍存在、无信息量，列入数组反降信噪比。决断：`alloc` 推断**仅用于 `pure` 判定**（堆分配 → 非 pure），**不进 effects 标注与 `.ailmeta` 数组**（数组只记 I/O·状态·并发等行为效果）。代价：`pure` 仍禁堆分配（语义不变），仅 effects 数组形态调整（剔除无信息量项）。=> 体现在 §19 / §75。

<a id="appendix"></a>
# 附录

## A. 版本演进与 v0.2.1 收敛说明

### 相对 v0.1 的主要变化
- ✅ `{}` 代码块，**不使用强制缩进**
- ✅ **可选分号**（Go 风格）
- ✅ 扩展名 `.ail`，元数据 `.ailmeta`
- ✅ `meaning` / `constraint` 独立声明
- ✅ `effects` / `requires` / `ensures` 契约块置于签名与函数体之间
- ✅ 泛型 trait 约束用 `where { Bound<T> }`（编译期单态化）；`requires { }` 装运行期前置断言（§15.10 / §86 #2）
- ✅ `interface`（服务能力）与 `trait`（类型能力）区分，新增 `implements`

### v0.2.1 收敛说明（本次新增）
1. **逻辑运算符采用符号** `&&` `||` `!`；`and`/`or`/`not` 不作关键字。
2. **trait 实现统一为内联** `struct X implements T1, T2 { ...方法... }`；v0.2 不设独立 `impl T for X { }` 块。代价：v0.2 不支持为外部（非本模块定义）类型实现 trait（决议见 §83 #2）。
3. **关键字扩充至 56**：新增 内存(`unsafe` `extern`)、并发(`actor` `message` `parallel` `server` `route` `cancel`)、AI(`agent` `tool`)、测试(`test`)、接收者(`self`)。
4. **类型（非关键字）**：`Shared` `Mutex` `Channel` `Optional` `Result` `List` `Map` `Set` `Array` `Tuple` `raw_pointer` 为标准库类型；`layout(C)` 为属性标注。
5. 新增小节：命名规范(§8.7)、运算符与优先级(§11)、AI 原生声明(§23)、Unsafe 与 FFI(§25)、并发(§21 升级)、形式文法(§27)。
6. **match 绑定语法锁定**：`Pattern: { 绑定 -> body }`（§16）；enum variant 支持带 payload（§15.6）。
7. **字符串 `+` 拼接锁定**：`string + string`，及 `string + Display` 自动字符串化（§11，唯一隐式转换例外）。
8. **集合字面量锁定**：`[a, b, c]`（List/Array/Set，标注定类型，默认 List）+ `[k: v]` / `[:]`（Map）；统一 `[]`，不用 `{}`（§15.9）。
9. **`lock` 定性**：`std.sync` 内建控制构造（**非关键字**）；块语法 `lock(m) { }` 规避 v0.2 无闭包依赖（§21）。
10. **`spawn` 三形式锁定**：`spawn task(args)`（具名 Task）/ `spawn actor Name(args)`（Actor，返 `ActorHandle`，§87 #5）/ `spawn { block }`（匿名块）均合法（§21、§27 文法）。
11. **关键字计数校正**：精确 56 个（§9）；原「约 55」为误计。
12. **§83 开放问题逐条决断**：本轮将 §83 全部 10 项作可逆决断——interface/trait 边界(#1)、外部类型 impl 分期(#2)、契约证明分级(#3)、try/catch 解糖(#4)、字面量不计入关键字(#5)、顶级声明边界(#6)、**string/集合 COW 值语义(#7)**、async 运行时(#8)、match arm 形式锁定(#9)、meaning 双轨(#10)；详见 §83 决议记录。

---

## B. 已对齐 / 遗留（工程实现层）

> 语言设计层面的开放问题已全部在 §83 决断（见决议记录）。本节仅列**编译器实现层**的遗留工程决断——非语言语义，而是「如何实现」的工程取舍。

### 已对齐（v0.2.1 SPEC 收敛后复核）

v0.2.1 SPEC 收敛后复核各模块先前的「待对齐」，结论：

1. ✅ **`agent` / `tool` 顶级声明**：已入 §9 关键字、§27 文法、§23 语义。
2. ✅ **`std.ai` 的 `agent { goal, tools }` / `tool { fn }`**：语法冻结（§23）。
3. ✅ **新关键字已入 §9**：`actor`/`message`/`parallel`/`server`/`route`/`cancel`（与 `task`/`spawn`/`channel`/`select`/`async`/`await`）；`unsafe`/`extern` 已入 §9。
4. 🔶 **`test` 是关键字**（§9）；`assert` 为 `std.testing` 内建（非关键字）。
5. ✅ **`unsafe { }` 块**：已入 §25 / §25.1。
6. ✅ **关键字/语法**：`layout(C)` 入 §27 文法（struct 属性前缀）；`raw_pointer<T>` 为标准库类型；独立 `impl T for X`：v0.2 不支持（用内联 `implements`，§15.5）；v0.3 计划引入（§83 #2）。
7. ✅ **`lock` 定性**：`std.sync` 内建控制构造（**非关键字**，§21）；块语法 `lock(m) { }` 规避 v0.2 无闭包；`Shared<T>`/`Mutex<T>` 归 `std.sync`。
8. ✅ **`spawn` 三形式锁定**：`spawn task(args)` / `spawn actor Name(args)`（§87 #5）/ `spawn { ... }` 块均合法（§21、§27）。
9. ✅ **路线图**：已统一为 Phase 1–4（§80）。

### 已决（语言层 → §83 决议记录）

| 主题 | 决议 |
|---|---|
| `try/catch/throw` 解糖 | §83 #4、§89 |
| `interface` / `trait` 边界 | §83 #1、§86 #1/#10 |
| `match` arm 形式 | §83 #9 |
| `meaning` / `constraint` 独立声明 | §83 #10、§92 #1/#5/#8/#9 |
| `string` / 集合内存模型（COW） | §83 #7（v0.2.1 锁定 COW 值语义，不引入最小 GC，无追踪 GC 详见 §18.4；§85 #2） |
| 关键字精确集合（56） | §83 #5 |
| Drop 语义 | §18.7 / §90 #5 |
| `unsafe` 边界精确清单（裸指针解引用 / FFI / 共享可变） | §90 #4（精确清单锁定 + panic 跨 FFI → abort + 无 `static mut` / 无 union） |
| Actor 与 `interface`/`trait`/`struct`/`agent` 的精确边界 | §83 #6（顶级声明边界锁定：actor 为独立构造，非 struct 变体） |
| `server`/`route` 语言级 vs 库级（与 `std.http` 关系） | §83 #6（语言级构造，解糖为 `std.http` + Task 调度） |
| Channel 有界/无界、close、多生产者多消费者语义 | §87 #2（MPMC + 容量 + close 状态机） |
| `parallel for` 归约语义（表达式化 + `parallel reduce`） | §87 #9 |
| `async` 运行时（executor）与 codegen | §83 #8 / §87（M:N work-stealing 内建于 `std.async`；v0.2 仅语义，codegen 见 §77） |

### 遗留（工程实现层）

1. **Phase 1 所有权子集（已定，2026-07-13）**：borrow checker **增量交付**，无临时模型——Phase 1 交付**所有权子集**（move 语义 + use-after-move 检测 + 确定性释放 §18.7 + 函数内 borrow/borrow_mut aliasing §74.2，非泛型），Phase 3 扩展**完整 borrow checker**（泛型所有权 + Lifetime Analyzer，依赖 Phase 2 类型系统）。排除 RC（move 退化违 §18.3）/ unsafe（违安全卖点）。每阶段语义为 §18 真子集、零迁移。=> 体现在 §80 / §74。
2. **可选分号的换行/续行规则边界用例**：二元运算符行尾、`return` 表达式跨行等——需 Parser 实现时以测试集敲定。
3. **Constraint / 契约证明的实现深度**：语言层已分级锁定（§83 #3，v0.2 仅断言）；实现层待定 v0.3+ 是否集成 SMT。
4. **`async/task/spawn/channel/select` 的 codegen 与运行时**：模型已指定（§83 #8，M:N work-stealing）；状态机 + executor 实现后置 Phase 3–4。
5. **`parallel for` 跨迭代 `Result<T,E>` 的自动合并策略**：部分失败如何归约（短路、收集 `List<E>`、或一律改显式 `parallel reduce`）——§87 #9 已锁表达式化与归约运算，但 Result 合并语义未定。
6. **自动 `release` 与 `Resource` trait 的精确契约**（与 trait 默认实现的交互）。

---

## C. 结语（Conclusion）

> 当 AI 写软件成为主流时，现有语言是否仍然适合？

AILang 的回答：**未来的语言必须不仅让人理解，也必须让 AI 精确理解。**

语义类型（可带约束）让概念显式、边界可验，效果系统让行为透明，所有权让资源归属清晰，AI 元数据让这一切可被机器直接消费——而这一切都不以牺牲原生性能为代价。AILang 希望成为人类与人工智能之间的新一代软件表达语言。
