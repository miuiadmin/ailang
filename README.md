# AILang

> **An AI-native programming language** · Artificial Intelligence Native Language
>
> A systems programming language designed for the AI era.

> 🌐 Languages: **English** | [简体中文](./README.zh-CN.md)

```
Status: Design phase (Pre-Alpha)     Spec: v0.2.1 (frozen)     Reference compiler: not yet implemented
```

---

## What is AILang

**AILang is a systems programming language designed from the ground up for AI to understand, generate, and maintain.**

It is not a scripting language. AILang compiles to native binaries via LLVM — no VM, no interpreter, no hard GC dependency — targeting Rust-level safety and performance. Its distinctive idea: **let programs actively express their intent** (what a type *means*, what side effects a function has, how it can fail, who owns a resource), and have the compiler turn that intent into machine-readable metadata (`.ailmeta`) that AI agents, IDEs, and automation tools can consume directly.

> AILang = Python's simplicity + C/Go's brace structure + Rust's safety model + TypeScript's type expressiveness + **AI-native semantics**.

---

## Why AILang

Software development is entering a new phase: **AI is becoming a primary producer of code.**

Today's languages are designed around the machine and the human programmer — friendly to humans, friendly to compilers, but never optimized for letting an AI *precisely understand* a program. So when AI writes code, it leans heavily on **guessing**: What does this parameter mean? Are negative values allowed? Does this have side effects? Can it fail? Does it take ownership? **Guessing is where errors come from.**

AILang's thesis:

> Good code should not hide information — it should actively express intent. A type should say not just *what* something is, but *what it means*.

---

## Core features

- **🤖 AI-native type system** — Semantic Types (`type UserId = int` + `meaning`), Constraint Types (compile-time + runtime), an Effect System, and errors derived from return types.
- **📄 `.ailmeta` metadata** — semantic facts *verified by the compiler* (not comments). AI can trust them the way it trusts a type signature.
- **🛡️ Rust-level safety** — Ownership / Borrow / static typing, **with no lifetime annotations** (non-escaping borrows → no `'a`).
- **⚡ Rust-level performance** — LLVM backend, native binaries, zero-cost abstraction.
- **🐍 Python-level simplicity + C-style braces** — braces `{}` + optional semicolons; simplicity from fewer symbols, not enforced indentation.
- **🔀 Analyzable concurrency** — Task / Channel / Actor: safer than Go, simpler than Rust async.
- **🧩 AI as a first-class citizen** — `agent` / `tool` are language-level declarations that auto-generate OpenAI Tool Schema / JSON Schema.

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

fn main() -> void {
    let user = get_user(1)
    print(user)
}
```

Compiling this produces two things: a **native executable**, and a machine-readable `app.ailmeta`:

```json
{
  "name": "get_user",
  "input": [{ "name": "id", "type": "UserId", "meaning": "unique identifier of a user" }],
  "output": "Result<User, UserError>",
  "effects": ["database.read"],
  "errors": ["NotFound", "PermissionDenied"]
}
```

An AI doesn't need to read source code or guess — it reads a **structured, verifiable, compiler-backed** description of program semantics. **This is the entire reason AILang exists.**

---

## Status (honestly)

AILang is **currently in the design phase**. The language specification is frozen (v0.2.1), but there is **no reference compiler yet** — you cannot `ail build` a program today.

| Area | Status |
|------|--------|
| Design documents (8) | ✅ Done |
| Language spec v0.2.1 | ✅ Frozen |
| Standard library / package manager design | ✅ Done |
| Reference compiler (`ailc`) | ⬜ Not started (Phase 1) |
| Runnable hello world | ⬜ Not started |

We welcome **spec review** and **design discussion**; compiler implementation will begin soon.

> 📝 Note: the design documents under `docs/` are currently written in **Simplified Chinese**; English translations are planned. This README is the English entry point — the linked documents remain in Chinese for now.

---

## Documentation

All design documents live in [`docs/`](./docs). Indexed by reading path:

| Document | Contents | For |
|----------|----------|-----|
| 📜 [WHITEPAPER.md](./docs/WHITEPAPER.md) | **Overview whitepaper** — why it exists, what it looks like, how it's built | First-time readers |
| 💡 [PHILOSOPHY.md](./docs/PHILOSOPHY.md) | **Design philosophy** — four principles, relation to Rust/Go/TS | Understanding "why" |
| 📐 [SPEC.md](./docs/SPEC.md) | **Language spec v0.2.1 (authoritative)** — lexicon/keywords/grammar/types/ownership/concurrency/formal grammar | Writing code / building the compiler |
| 🧪 [EXAMPLES.md](./docs/EXAMPLES.md) | **19 runnable examples** — from Hello World to AI Agents | Learning by reading |
| ⚙️ [DESIGN.md](./docs/DESIGN.md) | **Compiler architecture & implementation** — layered AST, borrow checker, LLVM, MVP roadmap | Building the compiler |
| 📦 [STDLIB.md](./docs/STDLIB.md) | **Standard library + `ail` package manager** — 14 modules, `.ailmeta` package artifact, secure publishing | Writing libraries / ecosystem |
| 🧠 [MEMORY.md](./docs/MEMORY.md) | **Memory model v0.3** — Stack/Heap, ownership, deterministic release, unsafe, FFI | Systems-level depth |
| 🔀 [CONCURRENCY.md](./docs/CONCURRENCY.md) | **Concurrency model v0.4** — Task/Channel/Actor/parallel/server | High-concurrency use cases |

### Recommended reading paths

- **5-minute tour** → WHITEPAPER
- **I want to write AILang** → SPEC → EXAMPLES
- **I want to build the compiler** → SPEC → DESIGN → EXAMPLES
- **I want to write libraries** → SPEC → STDLIB → EXAMPLES
- **I care about safety / performance** → SPEC §10 → MEMORY → CONCURRENCY

> Documents are versioned by **domain milestone**: v0.2 syntax / v0.3 memory / v0.4 concurrency / v0.5 (planned) AI layer. Together they describe one language.

---

## Roadmap

| Phase | Timeframe | Deliverables | Exit criterion |
|-------|-----------|--------------|----------------|
| **1** | 1–3 months | Lexer / Parser / AST / basic types / functions / structs / minimal codegen | `fn main() { print("hello") }` compiles to a native binary and runs |
| **2** | 3–6 months | `Result` / `enum` / generics / `interface` / **`.ailmeta`** | End-to-end: source → trustworthy AI metadata |
| **3** | 6–12 months | **Ownership + Borrow Checker** / concurrency / core standard library | All ownership-error test cases caught |
| **4** | 1 year+ | Package manager / IDE plugin / debugger / AI Agent ecosystem | Production-ready |

See [DESIGN.md §15](./docs/DESIGN.md) for details.

---

## Repository structure

```
ailang/
├── README.md              # this file (English)
├── README.zh-CN.md        # Chinese README (简体中文)
├── docs/                  # 8 design documents (currently in Simplified Chinese)
├── compiler/              # ⬜ reference compiler (planned: ail-lexer/parser/ast/checker/.../ailc)
├── std/                   # ⬜ standard library (planned)
└── package/               # ⬜ ail package manager (planned)
```

---

## Contributing

What's most needed right now is **spec review and design feedback**: ambiguities, edge cases, and conflicts with the stated goals. The v0.2.1 open questions have all been decided (see the SPEC §20 decision record); please raise feedback against any of those decisions.

Compiler implementation (Phase 1) will start soon; module-level contributions will be welcome then.

---

## License

AILang is licensed under the **Apache License 2.0** — see [LICENSE](./LICENSE).

Apache-2.0 is a permissive license (commercial use, modification, and distribution allowed) that includes an explicit patent grant — the same choice as Rust, Swift, TypeScript, and Kotlin.

---

## Contact

- Repository: (to be made public)
- Design discussion: (to be set up)

---

> **AILang is a systems programming language designed for the AI era.**
> It holds that code should express intent, not just instructions; that types should express meaning, not just data; that program behavior should be transparent, not hidden — and that performance, safety, and AI-legibility should not be traded against each other.
