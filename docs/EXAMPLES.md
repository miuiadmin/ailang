# AILang Examples v0.2.1

> **状态：可运行样例集（对齐 SPEC v0.2.1）**
> 范围：用完整程序演示 AILang 全部特性，从 Hello World 到 AI Agent。
> 权威规范见 [SPEC.md](./SPEC.md)；语义细节分散于 [MEMORY.md](./MEMORY.md) / [CONCURRENCY.md](./CONCURRENCY.md) / [STDLIB.md](./STDLIB.md)。

### 约定
- 大括号 `{}` + **可选分号**（下列示例省略分号）。
- 逻辑运算符 `&&` `||` `!`；trait 内联 `implements`；方法显式接收者模式（`borrow`/`borrow_mut`/`self`，省略时默认 `borrow self`；SPEC §22 #1）。
- 枚举构造 `Enum.Variant`；`match` arm 用 bare variant（scrutinee 类型已知时）。
- 字符串拼接用 `+`。
- 标 **ⓘ codegen 后置** 的特性：v0.2.1 已定义语义，运行时/LLVM codegen 列为后续（DESIGN §12）。

---

## 1. Hello World

```ail
package main

fn main() -> void {
    print("Hello AILang")
}
```

---

## 2. 变量与基础类型

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

## 3. 函数与运算符

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

> **整数算术语义（SPEC §27 #1/#2/#3）**：`a + b` 溢出——release **二补数回绕**（规范）、debug **panic `ArithmeticOverflow`**；整数除零 / 取模零 → **panic `DivideByZero`**；浮点遵循 **IEEE 754**（`x/0.0`→`±inf`、`0.0/0.0`→`NaN`，**永不 panic**）。显式控制用 `math.checked_add` / `math.wrapping_add` / `math.checked_div` / `math.is_nan`（STDLIB §6.1）。

---

## 4. Semantic Type（可带 constraint）

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
    // celebrate(n, age)      // ❌ n 是 int 变量，不能隐式转 UserId（名义不同）；字面量 celebrate(99, age) 则合法（字面量强制，SPEC §7.3 / SPEC §21 #1）
    // let bad: Age = -5      // ❌ 编译期约束违约 → 编译错（SPEC §7.4 / §29 #2）：Age 须 0..150；运行期经 Age(n) 构造违约才 panic ConstraintViolation（SPEC §29 #9）
    //
    // 运行期构造（SPEC §29 #2）：字面量之外，运行期 int 须经构造算符 T(expr) 变约束类型
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

> **marker-trait 继承（SPEC §29 #7）**：语义新类型（如 `UserId`）自动继承基底 `int` 的 `Copy`/`Send`/`Sync`——故 §19 `.ailmeta` 中 `id: UserId` 的 `mode: copy` 成立，`Map<UserId, User>` 可跨 `borrow_mut` 传递。运算 trait（`Add`/`Sub`/`Eq`/`Ord`/…）不自动继承，需显式 impl。

---

## 5. Struct + Trait（内联 implements + 方法）

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

## 6. Enum + Match（穷尽）

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

## 7. Result + Error 模型

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

**`try expr` 传播算符（前缀一元，SPEC §26 #1）+ 显式 `Ok` 构造（SPEC §26 #2）**：
```ail
fn load_both() -> Result<string, FileError> {
    let a = try read_config("a.toml")    // try：前缀一元，unwrap-or-propagate（同错误类型；SPEC §26 #5）
    let b = try read_config("b.toml")    // 任一 FileError 变体 → 自动传播（∈ FileError）
    return Ok(a + b)                     // 显式 Ok，无 auto-wrap（SPEC §26 #2/#10）
}
```

> `try/catch` 是 `match` 的语法糖；`try expr`（前缀）= unwrap-or-propagate，`try { } catch { }` = 局部捕获（SPEC §9 / SPEC §20 #4 / SPEC §26）。

---

## 8. Optional（禁止 null）

```ail
fn find_user(id: UserId) -> Optional<User> {
    if id == 1 {
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

## 9. 泛型 + where

```ail
fn first<T>(list: List<T>) -> Optional<T> {
    if list.length() > 0 {
        return Some(list[0])
    }
    return None
}

fn sort<T>(list: List<T>) -> List<T>
where {                                     // trait bound（编译期，单态化；SPEC §23 #2）
    Ord<T>
}
{
    // ... 排序实现
    return list
}
```

---

## 10. Interface vs Trait

```ail
interface Database {                          // 服务能力（dyn 派发；方法非泛型 SPEC §23 #1）
    fn query(borrow self, sql: string) -> Result<List<Row>, Error>   // Row = 动态行
    effects { database.read }                // interface 须带 effects（SPEC §22 #4）
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

## 11. 效果系统 + 契约

```ail
fn withdraw(account: borrow_mut Account, amount: int) -> Result<void, Error>
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
    account.balance = account.balance - amount   // 先突变内存（SPEC §22 #5）
    return database.exec("UPDATE ...")            // 再持久化
}

pure fn square(x: int) -> int {              // pure = 无效果（含 alloc）+ 不改入参（SPEC §22 #9）
    return x * x
}
```

**`.ailmeta` 输出真实、完整的效果集**（节选——省略 `span`）：
```json
{
  "identity": { "name": "withdraw", "description": null },
  "input": [
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

## 12. Ownership（move / borrow / 借用 / 确定性释放）

```ail
fn print_lines(file: borrow File) -> void {  // 只读借用，不取所有权
    print(file.read())
}

fn process_file() -> void {
    let file = File.open("a.txt")            // owner = process_file 作用域
    print_lines(borrow file)                  // 借用：调用后 file 仍可用
    print(file.size())
}                                             // ← file 在此自动 release（确定性，无 GC）

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

## 13. 并发：Task / Channel / select　ⓘ codegen 后置

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
- 默认禁止跨 Task 共享可变；要共享用 `Mutex<T>` / `Shared<T>`（MEMORY §12）。

---

## 14. 并发：parallel for（CPU 并行）　ⓘ codegen 后置

```ail
pure fn heavy(x: int) -> int {
    return x * x
}

fn main() -> void {
    let data: List<int> = [1, 2, 3, 4, 5]
    let squares: List<int> = parallel for item in data {   // 表达式：返回 List<R>（SPEC §24 #9）
        heavy(item)                                        // body 须返回 R
    }
    let sum = parallel reduce((a, b) -> a + b, squares) { x -> x }  // 归约（op 结合律）
    print(sum)
}
```

> 编译器见 body **无共享可变** → 允许并行（`heavy` 纯；CONCURRENCY §8、CONCURRENCY §14）。

---

## 15. Actor（独立状态 + 消息）　ⓘ codegen 后置

```ail
actor Counter {
    state:
        count: int

    message Increment
    message Get

    on Increment { _ -> self.state.count = self.state.count + 1 }   // SPEC §24 #5
    on Get { _ -> reply(self.state.count) }
}

let h = spawn actor Counter()       // ActorHandle（SPEC §24 #5）
```

> AI 极易理解：状态 = { count }，可接收消息 = { Increment, Get }，每条消息的行为由 on 定义

---

## 16. AI 原生：Agent + Tool

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

**`tool` 自动生成 OpenAI Tool Schema（下游派生产物，非 `.ailmeta` 权威形）**（STDLIB §13.3、SPEC §25 #4）：
```json
{
  "tool": "WebSearch.query",
  "description": "search the public web",
  "input": [{ "name": "text", "type": "string" }],
  "output": "List<SearchResult>"
}
```

**`agent` 编译为 `.ailmeta` `declarations[]` 条目**（节选——省略 `span`，SPEC §25 #4）：
```json
{
  "kind": "agent",
  "name": "ResearchAgent",
  "goal": "research AI news and summarize",
  "tools": ["WebSearch.query", "DbQuery.query"]
}
```

---

## 17. Unsafe + FFI + 布局

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

> `unsafe` 块在 `ail publish` 时被审计（STDLIB §15）。

---

## 18. 服务器（原生构造）　ⓘ codegen 后置

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

## 19. 综合示例：迷你用户服务

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

fn get_user(db: borrow Database, id: UserId) -> Result<User, UserError>   // interface = Move 句柄，按值借（SPEC §22 #6）
effects {
    database.read
}
{
    let rows = db.query("SELECT * FROM users WHERE id = " + id)   // Result<List<Row>, Error>（interface 非泛型，SPEC §23 #1）
    let list = match rows {
        Ok: { list -> list }
        _: { throw UserError.NotFound }        // 边界：Error → 命名错误（SPEC §21 #3/#6）
    }
    let users = decode<User>(list)             // List<Row> → Result<List<User>, Error>，单态化（SPEC §23 #1）
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
    let id: UserId = req.param("id")        // param<UserId>：HTTP 边界解析 + 强制（SPEC §21 #14）
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

## 特性覆盖索引

| 特性 | 示例 |
|------|------|
| package / fn / main | §1 |
| let / var / const / 推断 | §2 |
| pure / 运算符 / 优先级 | §3 |
| Semantic Type / Constraint Type | §4 |
| struct / trait / implements / 方法 / self | §5 |
| enum / match 穷尽 | §6 |
| Result / error / 错误派生 | §7 |
| Optional / Some / None | §8 |
| 泛型 / where | §9 |
| interface / trait 边界 | §10 |
| effects / requires / ensures / pure | §11 |
| move / borrow / borrow_mut / 确定性释放 | §12 |
| task / spawn / Channel / select | §13 |
| parallel for | §14 |
| actor / message | §15 |
| agent / tool / Schema 生成 | §16 |
| unsafe / extern / layout(C) / raw_pointer | §17 |
| server / route | §18 |
| 综合集成 | §19 |
