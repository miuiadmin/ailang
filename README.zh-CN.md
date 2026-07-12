# AILang

> **人工智能原生编程语言** · Artificial Intelligence Native Language
>
> 为 AI 时代设计的系统级编程语言。

> 🌐 语言：[English](./README.md) | **简体中文**

```
状态：设计阶段（Pre-Alpha）     规范：v0.2.1（冻结）     参考编译器：尚未实现
```

---

## 这是什么

**AILang 是第一代面向「AI 理解、生成、维护」而设计的系统级编程语言。**

它不是脚本语言——编译为原生二进制（经 LLVM），无虚拟机、无解释器、无 GC 强依赖，目标是接近 Rust 的安全与性能。它的独特之处在于：**让程序主动表达意图**（类型含义、副作用、错误可能、资源归属），并由编译器将这些语义产出为机器可读的元数据（`.ailmeta`），供 AI Agent、IDE 与自动化工具直接消费。

> AILang = Python 的简洁 + C/Go 的大括号结构 + Rust 的安全模型 + TypeScript 的类型表达 + **AI 原生语义**。

---

## 为什么需要 AILang

软件开发正在进入新阶段：**AI 正在成为代码的主要生产者之一。**

现有语言围绕「机器」与「人类程序员」设计——对人友好、对编译器友好，却没有为「让 AI 精确理解程序」优化。结果 AI 生成代码时大量依赖**猜测**：参数含义？是否允许负数？有无副作用？会不会失败？是否取得所有权？**猜测是错误的来源。**

AILang 的主张：

> 优秀的代码不应隐藏信息，而应主动表达意图。类型不仅表示「是什么」，还表示「意义」。

---

## 核心特性

- **🤖 AI 原生类型系统** — Semantic Type（`type UserId = int` + `meaning`）、Constraint Type（编译期+运行期约束）、Effect System、错误从返回类型派生。
- **📄 `.ailmeta` 元数据** — 编译器**验证后**的语义事实（非注释），AI 可像信任类型签名一样信任它。
- **🛡️ Rust 级安全** — Ownership / Borrow / 静态类型，**无生命周期标注**（borrow 不逃逸 → 无 `'a`）。
- **⚡ Rust 级性能** — LLVM 后端，原生二进制，零成本抽象。
- **🐍 Python 级简洁 + C 系大括号** — 大括号 `{}` + 可选分号；简洁来自少符号，而非强制缩进。
- **🔀 可分析的并发** — Task / Channel / Actor，比 Go 更安全、比 Rust async 更简单。
- **🧩 AI 一等公民** — `agent` / `tool` 为语言级声明，自动生成 OpenAI Tool Schema / JSON Schema。

---

## 一眼看清

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
    return database.find(id)
}

fn main() -> void {
    let user = get_user(1)
    print(user)
}
```

编译它，同时产出两样东西：**原生可执行文件**，以及机器可读的 `app.ailmeta`：

```json
{
  "name": "get_user",
  "input": [{ "name": "id", "type": "UserId", "meaning": "unique identifier of a user" }],
  "output": "Result<User, UserError>",
  "effects": ["database.read"],
  "errors": ["NotFound", "PermissionDenied"]
}
```

AI 无需读源码、无需猜测——读到的是一份**结构化、可验证、由编译器背书**的程序语义说明书。**这是 AILang 存在的全部理由。**

---

## 状态（诚实说明）

AILang **当前处于设计阶段**。语言规范已冻结（v0.2.1），但**尚无参考编译器**——你还不能 `ail build` 一个程序。

| 项 | 状态 |
|----|------|
| 设计文档（8 份） | ✅ 完成 |
| 语言规范 v0.2.1 | ✅ 冻结 |
| 标准库 / 包管理设计 | ✅ 完成 |
| 参考编译器（`ailc`） | ⬜ 未开始（Phase 1） |
| 可运行 hello world | ⬜ 未开始 |

欢迎参与**规范 review** 与**设计讨论**；编译器实现即将启动。

---

## 文档

所有设计文档位于 [`docs/`](./docs)。按阅读路径索引：

| 文档 | 内容 | 适合谁 |
|------|------|--------|
| 📜 [WHITEPAPER.md](./docs/WHITEPAPER.md) | **概览白皮书**——为何存在、长什么样、如何实现 | 第一次了解的人 |
| 💡 [PHILOSOPHY.md](./docs/PHILOSOPHY.md) | **设计哲学**——四大原则、与 Rust/Go/TS 的关系 | 想理解「为什么」 |
| 📐 [SPEC.md](./docs/SPEC.md) | **语言规范 v0.2.1（权威）**——词法/关键字/文法/类型/所有权/并发/形式文法 | 写代码 / 实现编译器 |
| 🧪 [EXAMPLES.md](./docs/EXAMPLES.md) | **19 个可运行样例**——从 Hello World 到 AI Agent | 边看边学 |
| ⚙️ [DESIGN.md](./docs/DESIGN.md) | **编译器架构 + 实现**——分层 AST、Borrow Checker、LLVM、MVP 路线 | 实现编译器 |
| 📦 [STDLIB.md](./docs/STDLIB.md) | **标准库 + `ail` 包管理**——14 模块、`.ailmeta` 包产物、安全发布 | 写库 / 生态 |
| 🧠 [MEMORY.md](./docs/MEMORY.md) | **内存模型 v0.3**——Stack/Heap、Ownership、确定性释放、Unsafe、FFI | 系统级深入 |
| 🔀 [CONCURRENCY.md](./docs/CONCURRENCY.md) | **并发模型 v0.4**——Task/Channel/Actor/parallel/server | 高并发场景 |

### 推荐阅读路径

- **5 分钟了解** → WHITEPAPER
- **想写 AILang** → SPEC → EXAMPLES
- **想实现编译器** → SPEC → DESIGN → EXAMPLES
- **想写库 / 生态** → SPEC → STDLIB → EXAMPLES
- **关心安全 / 性能** → SPEC §10 → MEMORY → CONCURRENCY

> 文档按**领域里程碑**版本标记：v0.2 语法 / v0.3 内存 / v0.4 并发 / v0.5（规划中）AI 层。它们共同描述同一门语言。

---

## 路线图

| Phase | 时间 | 产出 | 验证标准 |
|-------|------|------|----------|
| **1** | 1–3 月 | Lexer / Parser / AST / 基础类型 / 函数 / struct / 最小 codegen | `fn main() { print("hello") }` 编译为原生二进制并运行 |
| **2** | 3–6 月 | `Result` / `enum` / 泛型 / `interface` / **`.ailmeta`** | 端到端：源码 → 可信 AI 元数据 |
| **3** | 6–12 月 | **Ownership + Borrow Checker** / 并发 / 标准库核心 | 所有权错误用例全部报出 |
| **4** | 1 年+ | 包管理 / IDE 插件 / Debugger / AI Agent 生态 | 生产级 |

详见 [DESIGN.md §15](./docs/DESIGN.md)。

---

## 仓库结构

```
ailang/
├── README.md              # 英文入口（English）
├── README.zh-CN.md        # 本文件（简体中文）
├── docs/                  # 8 份设计文档
├── compiler/              # ⬜ 参考编译器（规划：ail-lexer/parser/ast/checker/.../ailc）
├── std/                   # ⬜ 标准库（规划）
└── package/               # ⬜ ail 包管理器（规划）
```

---

## 贡献

当前最需要的是**规范 review 与设计反馈**：发现歧义、边界用例、与目标冲突之处。v0.2.1 开放问题已全部决断（见 SPEC §20 决议记录），欢迎针对任一决议提出反馈。

编译器实现（Phase 1）即将启动，届时欢迎模块级贡献。

---

## 许可

AILang 采用 **Apache License 2.0** —— 见 [LICENSE](./LICENSE)。

Apache-2.0 为宽松协议（允许商用、修改、再分发），并含显式专利授权 —— 与 Rust / Swift / TypeScript / Kotlin 同。

---

## 联系

- 仓库：（待公开）
- 设计讨论：（待建立）

---

> **AILang 是为 AI 时代设计的系统级编程语言。**
> 它相信代码应表达意图而非仅是指令，类型应表达意义而非仅是数据，程序行为应透明而非隐藏——而性能、安全与 AI 理解能力不应互相牺牲。
