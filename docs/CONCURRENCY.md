# AILang Concurrency Model v0.4

> **状态：草案 / 供 review**
> 范围：Task、spawn、async/await、Channel、select、Actor、CPU 并行、服务器模型。
> 目标：**比 Go 更安全、比 Rust async 更简单、AI 能自动生成正确并发代码。**
> 关键字 `task`/`spawn`/`channel`/`select` 在 SPEC §13 已预留；本文展开语义。并发安全类型 `Shared`/`Mutex` 见 [MEMORY.md](./MEMORY.md) §12。

> **版本说明**：v0.2 语法 / v0.3 内存 / v0.4 并发 / v0.5 AI 层为**领域里程碑**（每领域一部专档，共同描述同一门语言）。`v0.4` 是并发专档的里程碑标签，**非决断落定版本**——本文所载并发决断由 SPEC §24（v0.2）及 SPEC §20/§21（v0.2.1）落定。

---

## 1. 哲学

| 语言 | 模型 | 问题 |
|------|------|------|
| C/C++ | `pthread_create` 裸线程 | 死锁、数据竞争、生命周期危险 |
| Java | 线程池 | 重量级、资源管理复杂 |
| Go | goroutine + channel | 简单，但**共享内存安全靠程序员** |
| Rust | `async` + `Future` + `Pin` + `Send/Sync` | 安全，但**学习成本高** |
| **AILang** | **Task-Based Concurrency** | 简单 + 安全 + AI 可分析 |

> AILang 既不照搬 goroutine，也不照搬 Rust Future 的复杂层，而是设计统一模型。

---

## 2. 核心单位：Task（非 Thread）

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

---

## 3. Task 定义与 spawn

```ail
task download() {
    fetch()
}

let h: TaskHandle = spawn download()    // 返回 TaskHandle（见 SPEC §24）
let h2 = spawn {                        // 匿名块；默认 scoped（作用域末隐式 join）
    await fetch_async()
}
spawn detached compute()                // 显式分离：仍返回 TaskHandle，cancel 可传播（非 fire-and-forget，§24 #4）
```

`spawn` 时编译器检查（见 SPEC §24）：
- 参数是否 `Send`（可跨线程移动，auto-trait 推导）
- 生命周期是否安全（被 spawn 的数据不能是 `borrow` 的非逃逸借用——`borrow` 不逃逸且禁跨 `await`，权威见 SPEC §10.3 / DESIGN §9.1 / MEMORY §8；本文 §4 释其与「无 Pin」的关系）

**结构化并发**：spawn 默认 **scoped**——task 附属 spawn 所在作用域，作用域末**隐式 join**（父等待子）；显式分离用 `spawn detached`。`cancel h`（§12（本文））向 `h` 子树传播。

---

## 4. async / await

网络 IO 等「等待型」操作用 `async fn`：

```ail
async fn request() -> Response { ... }      // T = Response

let response = await request()              // await request() 得 Response
```

- **返回语义**：`async fn f() -> T` 的 `T` **即 `await` 产出的值**；调用 `f()`（不 await）得 `Future<T>`（`std.async` lang item），`await f()` 得 `T`。`await` 为**一元前缀**：`await f()` / `await fut.field`。（详见 SPEC §24。）
- **无 `Pin`**（相对 Rust async 的关键简化）：`borrow`/`borrow_mut` **不逃逸、禁跨 `await` 点**（权威：SPEC §10.3 / DESIGN §9.1 / MEMORY §8）→ async 状态机**永不自引用** → 无需 Rust 的 `Pin`。此「禁跨 `await`」是并发各节反复引用的拱顶规则（§3、§5 借用不逃逸均其推论）。
- **阻塞禁忌**：`async fn` 内**禁止同步阻塞 I/O**（`io.blocking` effect，如 `fs.read` 同步版）——会占死 M:N worker；阻塞须经 `spawn_blocking { }` 或 async 变体（`fs.read_async`）。

---

## 5. Channel 通信

AILang 推荐**不共享内存，用 Channel 通信**（CSP，类 Go）：

```ail
let ch = Channel<int>()              // 无界 MPMC（见 SPEC §24）
let bounded = Channel<int>(8)        // 有界，容量 8；cap=0 = rendezvous 握手

spawn {
    ch.send(10)
}
ch.close()                           // 关闭：此后 receive()/try_receive() 耗尽即返回 None

let value = ch.receive()             // Optional<int>；阻塞至有值（Some）或关闭耗尽（None）
let maybe = ch.try_receive()         // Optional<int>；None = 空或关闭耗尽
```

- **模型**：**MPMC**（多生产者多消费者，匹配多 Task 同 `receive`）；无界 `Channel<T>()` / 有界 `Channel<T>(cap)`；`close()` 关闭；`receive() -> Optional<T>`（阻塞至有值得 `Some(v)`，关闭耗尽得 `None`）、`try_receive() -> Optional<T>`（非阻塞；`None` = 空或关闭耗尽）；select 含关闭检测分支（§7（本文））。（详见 SPEC §24。）
- **关闭后的 `receive()`**：`receive() -> Optional<T>` 在**关闭且耗尽**时返回 `None`（与 `try_receive()` 同型，**不 panic、不引入新 panic 符号**）——调用方可经 `match` / select 分支判 `None` 优雅退出。`receive()` 与 `try_receive()` 返回类型相同（`Optional<T>`），唯一差别在「open 且空」：前者阻塞等待，后者立即 `None`。
- `Channel` 为运行时支持的**共享 `Send` 句柄**（跨 spawn 克隆，底层队列）；`spawn { }` 块体为**控制构造**，可捕获外层不可变 / `Send` 变量（**不含 `borrow`**——借用不逃逸，详见 §4（本文）/ SPEC §10.3 / DESIGN §9.1 / MEMORY §8；非通用闭包，v0.2 无一等闭包）——见 SPEC §13、SPEC §21 #7。

---

## 6. Channel 类型安全

Channel 强类型：`Channel<User>` 只能 `send(User)`，`send(int)` 报 `Channel Type Error: Expected User, Found int`。

---

## 7. select（多路等待）

```ail
select {
    ch1.receive() { v ->
        handle1(v)              // v: Optional<T>（Some=值 / None=关闭）；v -> body（同 SPEC §8 match 约定）
    }
    ch.send(x) { ok ->          // send 分支
        send_done(ok)
    }
    timeout(100ms) { give_up() }    // 超时分支
    default { fallback() }          // 无就绪即走（非阻塞）
}
```

- **分支种类**：`receive` / `send` / `await` + 可选 `default { }`（非阻塞）+ 可选 `timeout(d) { }`（`default`/`timeout` 为 select 内上下文关键字，不入 56）。（详见 SPEC §24。）
- **公平性**：多分支就绪**随机公平择一**（类 Go）；**关闭** channel 的 receive 分支立即可就绪并交付 `None`（配合 §5（本文）`close`）。
- 接收值绑定复用 match 的 `{ 绑定 -> body }` 约定（SPEC §8）。

---

## 8. 默认禁止跨 Task 共享可变

```ail
var count: int = 0
spawn task1()
spawn task2()           // ❌ 编译器拒绝：count 跨 Task 共享可变 → 数据竞争
```

强制走 Channel 或 `Mutex`/`Shared`。跨 Task 移动 / 共享的安全性由 `Send`/`Sync` auto-trait 静态判定（SPEC §24 #6、STDLIB §7）。

---

## 9. 安全共享

```ail
let config: Shared<Config> = ...     // 多 Task 只读共享
let data = Mutex<Data>(...)
lock(data) {                         // 临界区，自动解锁
    update()
}
```

`Shared<T>` / `Mutex<T>` / `lock` 见 MEMORY §12。

---

## 10. Actor 模型（AI 友好）

Actor = 拥有**独立状态 + 消息入口 + 生命周期**的并发实体：

```ail
actor UserService {
    state:
        users: Map<UserId, User>

    message AddUser { user: User }
    message GetCount

    on AddUser { msg ->                             // 消息处理子句 on（见 SPEC §24）
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

- **`reply`（on-handler 上下文关键字）**：`reply` 是 `on` handler 内的**上下文关键字**（与 `on` 同，不入 56 关键字）；`reply(expr)` 退出当前消息处理并返回响应值 `expr`，其类型即**确定（infer）该 `message` 的响应类型**（response type；可为 `Result<T,E>`；无 `reply` 的 `on` handler 响应类型为 `void`）。故 **`on` handler 禁 `throw`**（actor 无 `Result` 返回签名）——错误出口即 `reply(Err...)`，或经死信传递（§13（本文）、SPEC §26 #8）。产生式（`on_clause` body 内的 `reply`）归属 SPEC §24 #5 / SPEC §18。
- **AI 极易理解 Actor 的行为边界**（状态 + 可接收消息清单 + 每条消息的行为）。

---

## 11. Task 生命周期

```
Created → Running → Waiting → Completed
                       │
                       └──► Cancelled
```

生命周期进 `.ailmeta` `declarations[]`（节选——省略 `span`，SPEC §25 #4）：
```json
{ "kind": "task", "name": "download", "states": ["Running", "Waiting", "Completed"] }
```

---

## 12. 取消（cancel）

```ail
cancel h                      // h: TaskHandle
if h.cancelled() { ... }      // 查询取消状态
```

- **协作式**：`cancel` **仅在 `await` 点生效**——CPU-bound task（无 await）不可取消，须调 `task.yield()` 让出以检查 `cancelled()`（`yield` 为 `std.async` 方法，非关键字）。
- **取消安全**：取消 = 在挂起点**丢弃状态机** → **确定性释放所有 locals**（MEMORY §9）→ 资源正确 close；`lock(m){ }` 块边界（含取消展开）**自动释放锁**。结合 §4（本文）「无 Pin」，状态机丢弃即安全。
- **传播**：`cancel h` 自动向 `h` 的子 Task 传播（结构化并发；详见 SPEC §24）：

```
parent task ──cancel──► child task ──cancel──► ...
```

---

## 13. 并发中的错误处理

**不隐藏错误**（SPEC §24 #1 / SPEC §26 #8 统一表）：

| 上下文 | `throw`/`try` | 错误出口 |
|--------|---------------|----------|
| `async fn -> Result<T,E>`（用户 `await`） | ✓ | 错误随 `await` 传调用方 |
| `task`（void 体） | ✗ 禁 | 经 Channel 投递错误消息或 `cancel` |
| actor `on` handler | ✗ 禁 | `reply(Err...)`（响应类型由 `reply` 表达式确定，可为 `Result<T,E>`，见 §10（本文））或死信 |
| server route handler | ✗ 禁（豁免） | runtime 唯一 awaiter，返回声明类型即可 |

```ail
let result = await risky_async()         // Result<T, Error>：用户 await → 须 Result
// task：错误经通道 —— ch.send(Err) 或 cancel h
// actor：on GetCount { _ -> reply(Ok(self.state.count)) }   // reply 返回响应值（可为 Result）
// server handler 豁免：async fn chat(req: Request) -> Response（runtime 派发）
```

`throw` 仅合法于返回 `Result<T,E>` 的函数体（SPEC §9、SPEC §26 #6）；`void`/`task`/handler 体禁 throw。无「静默失败」。

---

## 14. CPU 并行（parallel for / reduce）

计算密集型：

```ail
let squares: List<int> = parallel for x in data {     // 表达式：返回 List<R>
    x * x                                              // body 须返回 R
}
let sum = parallel reduce((a, b) -> a + b, data) { x -> x }   // 归约：op 须结合律
```

- **表达式化**：`parallel for x in xs { f(x) }` 返回 `List<R>`（body 须返回 `R`）；`parallel reduce(op, xs) { x -> f(x) }` 归约（`op` 结合律，`reduce` 为 `parallel` 后上下文关键字）。（详见 SPEC §24。）
- 编译器自动**分片 → 调度线程 → 合并结果**；body 须 `pure` 或仅 `io.read`（`print` 允许但输出交错无序保证）、**无共享可变**（§8（本文））。

---

## 15. AI 自动并发建议

编译器分析：纯函数 + 无共享状态 → 提示：

```
ℹ This loop can run in parallel
  (process is pure, no shared mutable state)
```

未来可自动并行化（依赖效果系统判定 `pure`）。

---

## 16. 服务器模型（原生构造）

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

`server` / `route` 为语言级构造，编译器生成路由 + 并发调度 + 元数据。handler 块为控制构造（SPEC §21 #7/#8），可捕获外层 `Send` 句柄（如连接池 `db`）。interface 值为 **Move 句柄**（SPEC §22 #6）：handler 捕获 owned 句柄，每次调用经形参模式推断 `borrow`（SPEC §21 #13），借用仅存于同步处理帧内、不跨 `await`（SPEC §10.3 / DESIGN §9.1 / MEMORY §8）。**handler 豁免 Result 规则**（SPEC §24 #1）：runtime 是唯一 awaiter，`async fn chat(req) -> Response` 返回声明类型即可（非用户 `await`）。

---

## 17. 并发安全总结

| 层次 | 手段 |
|------|------|
| **默认** | 共享不可变 + 消息通信 + 编译期检查 |
| **高级** | `Mutex` / `Shared` / `Actor` |
| **逃生** | `unsafe`（见 MEMORY §13） |

---

## 18. 对比

| 能力 | Go | Rust | AILang |
|------|----|------|--------|
| 轻量任务 | ★★★★★ | ★★★ | ★★★★★ |
| 内存安全 | ★★ | ★★★★★ | ★★★★★ |
| 学习成本 | 低 | 高 | 低 |
| AI 理解度 | ★★★ | ★★★ | ★★★★★ |
| 消息模型 | ★★★★★ | ★★★ | ★★★★★ |
| 自动分析 | ★★ | ★★★ | ★★★★★ |

---

## 已对齐 / 遗留（SPEC 决议已落地）

1. ✅ **新关键字已入 SPEC §2**：`actor`/`message`/`parallel`/`server`/`route`/`cancel`（与 `task`/`spawn`/`channel`/`select`/`async`/`await`）。
2. ✅ **`lock` 定性**：`std.sync` 内建控制构造（**非关键字**）；块语法 `lock(m) { }` 规避 v0.2 无闭包（SPEC §13）。
3. ✅ **`spawn` 双形式锁定**：`spawn task(args)` 与 `spawn { ... }` 均合法（SPEC §13、SPEC §18）。

**已决**（SPEC §20 收敛于「无未决开放问题」）：
- Actor 与 `interface`/`trait`/`struct`/`agent` 的精确边界 → SPEC §20 #6（顶级声明边界锁定：actor 为独立构造，非 struct 变体）。
- `server`/`route` 语言级 vs 库级（与 `std.http` 关系）→ SPEC §20 #6（语言级构造，解糖为 `std.http` + Task 调度）。
- Channel 有界/无界、close、多生产者多消费者语义 → SPEC §24 #2（MPMC + 容量 + close 状态机）。
- `parallel for` 归约语义（表达式化 + `parallel reduce`）→ SPEC §24 #9。
- `async` 运行时（executor）与 codegen → SPEC §20 #8 / SPEC §24（M:N work-stealing 内建于 `std.async`；v0.2 仅语义，codegen 见 DESIGN §12）。

**遗留开放**（1 项）：
- `parallel for` 跨迭代 `Result<T,E>` 的自动合并策略（部分失败如何归约：短路、收集 `List<E>`、或一律改显式 `parallel reduce`）——SPEC §24 #9 已锁表达式化与归约运算，但 Result 合并语义未定。
