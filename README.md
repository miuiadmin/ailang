# AILang

> **An AI-native systems programming language** — built for AI to understand, generate, and maintain.

🌐 **English** | [简体中文](./README.zh-CN.md)

`Pre-Alpha · design phase`　`Spec v0.2.1 (frozen)`　`Reference compiler: not yet`

---

## What it is

AILang is a systems programming language for the AI era. It compiles to native binaries via LLVM — no VM, no hard GC — targeting Rust-level safety and performance.

Its core idea: **code should actively express intent.** Types carry *meaning*, functions declare their *effects*, errors come from the *signature*, ownership is *explicit* — and the compiler turns all of it into machine-readable metadata (`.ailmeta`) that AI agents can trust like a type signature.

> AILang = Python's simplicity + C/Go braces + Rust's safety + TypeScript's types + **AI-native semantics**.

---

## At a glance

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

Compiling this produces a native binary **and** an `app.ailmeta`:

```json
{
  "name": "get_user",
  "input":  [{ "name": "id", "type": "UserId", "meaning": "unique identifier of a user" }],
  "output": "Result<User, UserError>",
  "effects": ["database.read"],
  "errors": ["NotFound", "PermissionDenied"]
}
```

An AI reads this **structured, compiler-verified** description instead of guessing from source. **That is the whole reason AILang exists.**

---

## Key features

- **🤖 AI-native types** — semantic types (`type UserId = int` + `meaning`), constraint types, an effect system, errors derived from return types.
- **📄 `.ailmeta`** — semantic facts *verified by the compiler* (not comments). AI trusts them like a type.
- **🛡️ Rust-level safety, no lifetimes** — ownership & borrow with inferred lifetimes; borrows don't escape, so there's no `'a`.
- **⚡ Rust-level performance** — LLVM backend, native binaries, zero-cost abstraction.
- **🐍 Python-level simplicity** — braces `{}` + optional semicolons.
- **🧩 AI as a first-class citizen** — `agent` / `tool` declarations auto-generate OpenAI Tool Schema.

---

## Status

In design phase. The spec is frozen (**v0.2.1**), but there is **no reference compiler yet** — you can't `ail build` today. Spec review and design feedback are very welcome; compiler work (Phase 1) starts soon.

---

## Documentation

All design docs are in [`docs/`](./docs):

📜 [WHITEPAPER](./docs/WHITEPAPER.md) · 💡 [PHILOSOPHY](./docs/PHILOSOPHY.md) · 📐 [SPEC](./docs/SPEC.md) · 🧪 [EXAMPLES](./docs/EXAMPLES.md) · ⚙️ [DESIGN](./docs/DESIGN.md) · 📦 [STDLIB](./docs/STDLIB.md) · 🧠 [MEMORY](./docs/MEMORY.md) · 🔀 [CONCURRENCY](./docs/CONCURRENCY.md)

> First time here? Start with the [WHITEPAPER](./docs/WHITEPAPER.md).

---

## License & contact

**Apache License 2.0** — see [LICENSE](./LICENSE).

- Repository: https://github.com/miuiadmin/ailang
- Discussions: https://github.com/miuiadmin/ailang/discussions

---

> AILang believes code should express intent, not just instructions — and that performance, safety, and AI-legibility need not be traded against each other.
