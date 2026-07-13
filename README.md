# AILang

[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](./LICENSE)
[![Status: Pre-Alpha](https://img.shields.io/badge/status-Pre--Alpha-orange.svg)](#status)
[![Spec: v0.2.1](https://img.shields.io/badge/spec-v0.2.1-frozen-informational.svg)](./docs/AILANG.md)
[![Discussions](https://img.shields.io/github/discussions/miuiadmin/ailang)](https://github.com/miuiadmin/ailang/discussions)

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

The complete design lives in a single document — [`docs/AILANG.md`](./docs/AILANG.md) (v0.2.1, frozen). It's organized in six parts:

- [Part 0 · Motivation & Philosophy](./docs/AILANG.md#part-0)
- [Part I · Language Specification](./docs/AILANG.md#part-i) — the authoritative spec (incl. ownership & concurrency deep-dives)
- [Part II · Standard Library & Packages](./docs/AILANG.md#part-ii)
- [Part III · Tutorial (runnable examples)](./docs/AILANG.md#part-iii)
- [Part IV · Compiler Implementation Design](./docs/AILANG.md#part-iv)
- [Part V · Decision Records](./docs/AILANG.md#part-v)

> First time here? Start with [Part 0](./docs/AILANG.md#part-0), then read [Part I](./docs/AILANG.md#part-i).

---

## License & contact

**Apache License 2.0** — see [LICENSE](./LICENSE).

- Repository: https://github.com/miuiadmin/ailang
- Discussions: https://github.com/miuiadmin/ailang/discussions

---

> AILang believes code should express intent, not just instructions — and that performance, safety, and AI-legibility need not be traded against each other.
