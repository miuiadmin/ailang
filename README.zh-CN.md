# AILang

> **人工智能原生系统级编程语言** —— 为「让 AI 理解、生成、维护」而设计。

🌐 [English](./README.md) | **简体中文**

`Pre-Alpha · 设计阶段`　`规范 v0.2.1（冻结）`　`参考编译器：尚未实现`

---

## 这是什么

AILang 是为 AI 时代设计的系统级编程语言，经 LLVM 编译为原生二进制——无虚拟机、无 GC 强依赖，目标是 Rust 级的安全与性能。

核心理念：**代码应主动表达意图。** 类型携带*含义*，函数声明*副作用*，错误来自*签名*，所有权*显式可读*——编译器把这些语义产出为机器可读的元数据（`.ailmeta`），AI 可像信任类型签名一样信任它。

> AILang = Python 的简洁 + C/Go 大括号 + Rust 的安全 + TypeScript 的类型 + **AI 原生语义**。

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
```

编译它，同时产出原生二进制**和** `app.ailmeta`：

```json
{
  "name": "get_user",
  "input":  [{ "name": "id", "type": "UserId", "meaning": "unique identifier of a user" }],
  "output": "Result<User, UserError>",
  "effects": ["database.read"],
  "errors": ["NotFound", "PermissionDenied"]
}
```

AI 读取这份**结构化、由编译器验证**的描述，而不是从源码里猜。**这是 AILang 存在的全部理由。**

---

## 核心特性

- **🤖 AI 原生类型** —— 语义类型（`type UserId = int` + `meaning`）、约束类型、效果系统、错误从返回类型派生。
- **📄 `.ailmeta`** —— 编译器**验证后**的语义事实（非注释），AI 可像信任类型一样信任它。
- **🛡️ Rust 级安全，无生命周期** —— Ownership / borrow，生命周期自动推导；borrow 不逃逸，所以没有 `'a`。
- **⚡ Rust 级性能** —— LLVM 后端，原生二进制，零成本抽象。
- **🐍 Python 级简洁** —— 大括号 `{}` + 可选分号。
- **🧩 AI 一等公民** —— `agent` / `tool` 声明自动生成 OpenAI Tool Schema。

---

## 状态

当前处于设计阶段。规范已冻结（**v0.2.1**），但**尚无参考编译器**——你还不能 `ail build`。欢迎参与规范 review 与设计讨论；编译器工作（Phase 1）即将启动。

---

## 文档

所有设计文档位于 [`docs/`](./docs)：

📜 [白皮书](./docs/WHITEPAPER.md) · 💡 [哲学](./docs/PHILOSOPHY.md) · 📐 [规范](./docs/SPEC.md) · 🧪 [示例](./docs/EXAMPLES.md) · ⚙️ [架构](./docs/DESIGN.md) · 📦 [标准库](./docs/STDLIB.md) · 🧠 [内存模型](./docs/MEMORY.md) · 🔀 [并发模型](./docs/CONCURRENCY.md)

> 第一次了解？从[白皮书](./docs/WHITEPAPER.md)开始。

---

## 许可与联系

**Apache License 2.0** —— 见 [LICENSE](./LICENSE)。

- 仓库：https://github.com/miuiadmin/ailang
- 讨论：https://github.com/miuiadmin/ailang/discussions

---

> AILang 相信代码应表达意图，而非仅是指令——而性能、安全与 AI 可理解性，本不必互相牺牲。
