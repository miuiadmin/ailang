# AILang 设计哲学（Design Philosophy）v0.2

> 本文档解释**为什么创造 AILang**——一门面向 AI 原生时代的系统级编程语言。
> 语法与类型见 [SPEC.md](./SPEC.md)；编译器实现见 [DESIGN.md](./DESIGN.md)。

---

## 名称

- **AILang**
- 全称：**A**rtificial **I**ntelligence Native **Lang**uage
- 定位：面向 AI 原生时代设计的高性能系统级编程语言

---

## 1. 背景：AI 成为代码生产者

过去几十年，编程语言主要围绕两个对象设计：**机器**与**人类程序员**。C 追求接近硬件；Java 追求工程生产力；Python 追求开发效率；Rust 追求安全和性能；Go 追求简单和工程规模。

但是 AI 编程时代出现了新的参与者：

> AI 不再只是辅助工具，而正在成为代码的主要生产者之一。

未来大量代码可能由 AI Agent、自动代码生成系统、智能开发工具完成。这带来一个新问题：

> 如何让 AI **精确理解**程序，而不是通过猜测生成代码？

---

## 2. AILang 的核心目标

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

---

## 3. 四大核心原则

### 原则一：Rust 级安全

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

### 原则二：Rust 级性能

AILang 是编译型语言，而非脚本语言：

```
源码 → AST → Type Checker / Ownership Analyzer / AI Analyzer → HIR（AILang IR）→ .ailmeta → LLVM IR → Machine Code
```

输出原生二进制（`.exe` / ELF / Mach-O），无虚拟机、无解释器、原生执行。

### 原则三：Python 级简洁 + C 系大括号结构

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

### 原则四：AI First 类型系统（AILang 最大区别）

传统语言：`type = 数据格式`。
AILang：让类型可携带 **含义 + 约束 + 行为**（语义类型 / Semantic Type；SPEC §7.3 / SPEC §29 #1）；裸 `type X = Y` 仍可为透明别名。

例如，定义一个语义类型（Semantic Type；SPEC §7.3）：

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

---

## 4. AILang 与 Rust 的最大区别

> Rust：让**程序员**告诉编译器安全规则。
>
> AILang：让**编译器**自动推导安全规则，让 AI 更容易生成正确代码。

典型体现：生命周期不再由人标注（见 SPEC §10.3），所有权与借用规则由编译器内部的分析器推导。

---

## 5. 与其他语言的关系

AILang 不试图替代所有语言，而是吸收各所长处，补上 AI 原生语义这一缺失维度：

| 语言 | AILang 吸收 |
|------|-------------|
| **Rust** | 安全、性能 |
| **TypeScript** | 类型表达 |
| **Python** | 简洁表达 |
| **Go** | 大括号工程结构、工程效率 |
| **AILang 独有** | AI 原生语义（Semantic Type / `.ailmeta`） |

一句话：

> AILang = Python 的简洁 + C/Go/Rust 风格的大括号结构 + Rust 的安全模型 + TypeScript 的类型表达 + AI 原生语义。

---

## 6. 一句话定位

> AILang 是第一代为人工智能理解、生成和运行而设计的系统级编程语言。

---

## AILang 宣言

> **AILang 是为 AI 原生时代设计的系统级编程语言。**
>
> 它相信代码应该表达意图，而不仅是指令；
> 类型应该表达意义，而不仅是数据；
> 程序行为应该透明，而不是隐藏；
> 性能、安全与 AI 理解能力不应该互相牺牲。
>
> AILang 的目标，是成为人类与人工智能之间的新一代软件表达语言。
