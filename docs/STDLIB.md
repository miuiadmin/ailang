# AILang Standard Library & Package Ecosystem v0.2

> **状态：草案 / 供 review（对齐 v0.2.1 语法）**
> 范围：标准库 API 规范 + `ail` 包管理器 + 包生态（含 `.ailmeta` 作为包产物）。
> 语言规范见 [SPEC.md](./SPEC.md)；编译器实现见 [DESIGN.md](./DESIGN.md)；动因见 [PHILOSOPHY.md](./PHILOSOPHY.md)。
> 定位：**这一层决定 AILang 能否从「语言」变成「可用平台」。**

---

## 设计目标

对标：

| 语言 | 包管理 |
|------|--------|
| Rust | cargo |
| Python | pip |
| Go | go modules |
| **AILang** | **`ail`** |

`ail` = **AILang Integrated Launcher**。在 cargo/go 的能力之上，AILang 包**额外携带机器可读的语义元数据**——这是生态层的 AI 原生增量。

---

# 第一部分：包管理器 `ail`

## 1. 命令集

| 命令 | 作用 |
|------|------|
| `ail new <project>` | 生成项目骨架 |
| `ail build` | 编译当前包 |
| `ail run` | 编译并运行 |
| `ail test` | 运行内置测试 |
| `ail add <pkg>` | 增加依赖（改写 `ail.toml`） |
| `ail remove <pkg>` | 删除依赖 |
| `ail update` | 更新依赖 |
| `ail publish` | 发布到包仓库（含安全扫描，见 §15） |

## 2. 项目结构

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

## 3. 包清单 `ail.toml`

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

**包名 ↔ 模块名（SPEC §28 #10）**：`[package].name` = **根包名 = 导入根前缀**——`name = "web_server"` 则内部声明 `package web_server`，依赖方 `import web_server.<module>`；依赖 `http = "1.0"` 安装后以 `import http.<module>` 引用（依赖包名作根）；版本遵循 semver；包名**禁与 `std` 冲突**。

---

## 4. AILang 包 ≠ 普通语言的包

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

## 5. `.ailmeta` 作为包的强制产物

每个库**必须**随包发布 `xxx.ailmeta`（权威 schema 见 WHITEPAPER §3）。例（`http` 库，节选——省略 `span`）：

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

> 包版本不在 `.ailmeta` 顶层（在 `ail.toml`，§3）；`.ailmeta` 顶层仅 `schema_version`（schema 自身 semver，与语言/包版本解耦，WHITEPAPER §3）。

**AI 使用库的方式**：不读源码，直接 `load metadata → understand API → generate code`。这是 AILang 生态相对其他语言的核心差异——库对 AI 是「即插即用」的。

---

# 第二部分：标准库 `std`

## 6. 模块总览

```
std/
├── core          # 默认加载：Optional / Result / error / print
├── collections   # List / Map / Set
├── string        # UTF-8 字符串
├── math          # 数值边界运算符族（SPEC §27）+ 基础数学（v0.3+ 细化）
├── io            # v0.2 占位：流式 IO（stdin/stdout 流，与 print 互补）
├── fs            # 文件系统（§10）
├── net           # 网络（§6.1）：NetworkError
├── http          # HTTP 客户端/服务端（§11）：Request / Response / URL
├── crypto        # v0.2 占位：hash / hmac / ...
├── time          # v0.2 占位：now / duration / ...
├── async         # 异步运行时（§7：Future / TaskHandle / ...）
├── database      # 统一数据库接口（§12）：Row / decode<T>
├── ai            # AI 原生（§13）：SearchResult / AIError
└── testing       # 内置测试（§14）
```

### 6.1 占位 / 底层模块（SPEC §25 #8/#9/#10；SPEC §27 数值语义）

`std.io` / `std.crypto` / `std.time` 为 **v0.2 占位**——仅占名空间，API v0.3+ 细化（`std.io` 为流式 IO，与 `std.core.print` 互补）。`std.math` 承载**数值语义运算符族**（SPEC §27 #1/#2/#3，非占位）：

```ail
// std.math —— 数值边界运算（SPEC §27 #1/#2/#3）
// 整数算术溢出：release 二补数回绕（规范）/ debug panic(ArithmeticOverflow)；以下为显式控制
wrapping_add(a: int, b: int) -> int             // 总是回绕（同名 sub/mul/neg）
checked_add(a: int, b: int) -> Optional<int>     // 溢出 → None（同名 sub/mul）
saturating_add(a: int, b: int) -> int            // 饱和到类型边界（同名 sub/mul）
overflowing_add(a: int, b: int) -> (int, bool)   // (结果, 是否溢出)
checked_div(a: int, b: int) -> Optional<int>     // 安全除法：b==0 → None（裸 a / b 零除 → panic DivideByZero）

// 浮点 IEEE 754 特殊值检测（float 方法，borrow self；运算永不 panic；SPEC §27 #2）
is_nan(borrow self) -> bool        // self.is_nan()：是否 NaN（NaN == NaN 为 false，勿用 == 判 NaN）
is_inf(borrow self) -> bool        // self.is_inf()：是否 ±inf
is_finite(borrow self) -> bool     // self.is_finite()：是否有限（非 NaN、非 inf）
```

`std.net` 声明底层网络错误：

```ail
// std.net
error NetworkError {            // 网络错误枚举（SPEC §25 #10）
    Timeout
    ConnectionFailed
}
```

`std.http`（§11）的网络错误复用 `std.net.NetworkError`。

---

## 7. `std.core`（默认加载）

无需 `import`。包含：

### Optional\<T>
```ail
fn find_user(id: UserId) -> Optional<User> { ... }
// Some(user) 或 None；无 null
```

**`Optional<T>` 方法签名表**（SPEC §25 #7；接收者模式 SPEC §22 #1）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `unwrap` | `(self) -> T` | 取值；`None` 则 **panic**（栈展开+Drop，SPEC §26 #3；消费 self） |
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
| `unwrap` | `(self) -> T` | 取 `Ok` 值；`Err` 则 **panic**（SPEC §26 #3） |
| `unwrap_or` | `(self, default: T) -> T` | 取值或默认 |
| `is_ok` | `(borrow self) -> bool` | 是否 `Ok` |
| `is_err` | `(borrow self) -> bool` | 是否 `Err` |

### error —— 统一错误系统
命名错误枚举经 `error` 声明（规范示例见 `std.net.NetworkError`，§6.1；`std.fs.FileError` §10、`std.ai.AIError` §13 同形）。

### `print` —— stdio 默认内建
`print(x: Display) -> void` 输出到 stdout，默认加载、无需 `import`（类 Go `println`）；`.ailmeta` 标注 effect `io.write`（SPEC §21 #9、SPEC §25 #6；须 `Display` trait SPEC §23 #8）。

### `Error` —— 通用错误（动态边界）
`std.core.Error` 为 opt-in 通用错误类型，用于动态边界（DB 驱动 / FFI / 泛型 Task）；惯用代码用命名 `error` 枚举保 `.ailmeta` 精度，边界处显式 `Error → 命名错误` 映射（SPEC §9、SPEC §21 #6）。

### `panic` / `assert` —— 不可恢复错误（SPEC §26 #3）
`std.core.panic(msg: string) -> never` 与 `assert(cond: bool, msg?: string) -> void`（`std.testing` 亦暴露 `assert`）；`unwrap`/`assert` 失败 = panic。**panic = 确定性栈展开 + Drop/release**（与确定性释放一致），展开至最近的 **Task 边界或 `extern` 帧**——Task 内 panic 终止当前 Task（→ TaskHandle 错误态 / actor 死信），**不传染进程**；触达 `extern` 帧（AILang 被 C 调入后 panic）→ **转进程 abort**（确定性终止，**非 UB**；FFI 边界为「不传染进程」的唯一例外，SPEC §27 #4）。v0.2 **无 `catch_unwind`**（仅 Task 边界可观测）。`panic` 为 std 函数，非关键字（56 不变）；完整 panic 语义见 SPEC §9。

已知 panic 触发原因 → 符号 → 指针：

| 原因 | panic 符号 | 指针 |
|------|-----------|------|
| `unwrap` / `assert` 失败 | （直接 panic，无独立符号） | SPEC §26 #3 |
| 整数溢出（debug 构建） | `ArithmeticOverflow` | SPEC §27 #1 |
| 整数除零 / 取模零 | `DivideByZero` | SPEC §27 #3 |
| 越界访问（下标越界） | `IndexOutOfBounds` | SPEC §9 |
| 约束违约（`T(expr)` 谓词失败） | `ConstraintViolation` | SPEC §7.4 / SPEC §29 #9 |

### 运算符 trait（SPEC §23 #8）
运算符解糖为 std.core trait 方法调用；可作泛型 bound（`where { Ord<T> }`）。核心集：

| trait | 方法 | 运算符 |
|-------|------|--------|
| `Add` / `Sub` / `Mul` | `add` / `sub` / `mul` | `+` / `-` / `*` |
| `Eq` | `eq` | `==` `!=` |
| `Ord` | `lt` | `<` `>` `<=` `>=` |
| `Display` | `display` | 字符串化（`+` 拼接，SPEC §3b） |

`&&` / `||` / `!` 为内置短路，不可重载。

### `Send` / `Sync` —— 并发安全 marker trait（SPEC §24 #6）
`Send`（可跨线程 move）/ `Sync`（可跨线程共享只读）为编译器**自动派生**的 marker trait（`#[lang]`，SPEC §23 #5），无需手写 `impl`。组合规则：

| 类型 | Send | Sync | 条件 |
|------|------|------|------|
| Copy 基础（`int`/`bool`/...） | ✓ | ✓ | — |
| COW（`string`/`List`/`Map`/`Set`，原子 rc） | ✓ | — | Send ✓（原子 rc，SPEC §22 #2）；Sync ✗ |
| Move 资源（`File`/`Socket`） | ✓ | ✗ | 独占转移 |
| `Mutex<T>` | ✓ | ✓ | `T: Send` |
| `Shared<T>` | ✓ | ✓ | `T: Sync` |
| `TaskHandle` / `ActorHandle` / `Channel<T>` | ✓ | — | `T: Send` |

`spawn` / `Channel.send` / actor 投递边界静态查 `Send`（SPEC §13、CONCURRENCY §3/§8、DESIGN §8/§9）。

### 并发运行时类型（`std.async` lang item）
`Future<T>`（`async fn` 状态机产物，**无 `Pin`**）、`TaskHandle`（`spawn` 返回，结构化作用域 join / `cancel` / `cancelled`）、`ActorHandle`（`spawn actor` 返回，`send`/`!` 投递）、`spawn_blocking { }`（独立线程池，不占 M:N worker）均为 `std.async` lang item（SPEC §23 #5）；完整语义见 SPEC §24 决议记录 V、CONCURRENCY、DESIGN §12。

---

## 8. `std.collections`

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

> `List` / `Map` / `Set` 为 **COW 值类型**（SPEC §10.2、SPEC §22 #2/#7）：赋值=共享缓冲（**共享句柄，非位复制** → Copy-compatible，计入 `is_copy`；**原子 rc** → `Send`），修改=克隆；行为同 Copy，**无追踪 GC**。可变方法（`insert`/`append`/`remove`）声明 `borrow_mut self`。

**方法签名表**（SPEC §25 #7；接收者模式同 §7；下标 `xs[i]` / `m[k]` 返回值——borrow 不逃逸，故按值返回，要求 `T` 为 Copy/COW）：

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

## 9. `std.string`

统一类型 `string`，UTF-8（COW 值类型，**共享句柄** → Copy-compatible；SPEC §10.2、SPEC §22 #7）。

```ail
let name = "AI"
name.length()         // 2
name.contains("A")    // true
name.upper()          // "AI"
```

**`string` 方法签名表**（SPEC §25 #7；接收者模式同 §7；`+` 拼接见运算符 trait SPEC §23 #8）：

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

## 10. `std.fs`（文件系统）

```ail
import std.fs

fn read() -> Result<string, FileError> {
    let text = fs.read("a.txt")
    return text
}
```

编译器自动从 `fs.read` 推断效果 `filesystem.read` + `io.blocking`（同步阻塞，SPEC §24 #7），写入 `.ailmeta`：
```json
{ "effects": [ "filesystem.read", "io.blocking" ] }
```

**`FileError` 声明于 `std.fs`**（SPEC §25 #10；与 `File` 同模块）：
```ail
// std.fs
error FileError {
    NotFound
    PermissionDenied
}
```

> `fs.read` 为**同步阻塞**（effect 含 `io.blocking`，SPEC §24 #7）：可在 `task`/同步 `fn` 内用，但**禁止在 `async fn` 内直接调用**（占死 M:N worker）。async 上下文用 `fs.read_async`（返回 `Future<Result<...>>`），或经 `spawn_blocking { fs.read(...) }`。标准库 IO 双轨（sync / async 变体）。

---

## 11. `std.http`（HTTP 客户端/服务端）

**`Request` / `Response` / `URL` 声明于 `std.http`**（SPEC §25 #8）：
```ail
// std.http
type URL = string                  // 语义类型（名义，SPEC §29 #1）
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

`.ailmeta` 告诉 AI：`async` / `network` 效果 / `Response` 输出（网络错误复用 `std.net.NetworkError`，§6.1）。

---

## 12. `std.database`（统一数据库接口）

**`Row` 声明于 `std.database`**（动态行，SPEC §23 #1、SPEC §25 #8）；`decode<T>` 同模块 lang item（SPEC §25 #10）：
```ail
// std.database
type Row = ...                     // 别名（动态行，列名 → 值；SPEC §29 #10）
fn decode<T>(rows: List<Row>) -> Result<List<T>, Error>   // typed 解码（单态化，SPEC §23 #1）
```

```ail
interface Database {
    fn query(borrow self, sql: string) -> Result<List<Row>, Error>   // 非泛型（SPEC §23 #1）
    effects { database.read }                // interface 须带 effects（SPEC §22 #4）
}
```

驱动支持：PostgreSQL / MySQL / SQLite / Redis。

> `interface` = 服务能力抽象（**dyn 派发**，SPEC §7.11）；interface 值为 **Move 句柄**，经 `borrow` 借用（SPEC §22 #6）；方法非泛型（SPEC §23 #1）。

---

## 13. `std.ai`（AILang 核心竞争力）

这是 AILang 区别于所有现有语言的标准库模块。

**`SearchResult` / `AIError` 声明于 `std.ai`**（SPEC §25 #8/#10）：
```ail
// std.ai
struct SearchResult { ... }        // 搜索结果（title / url / snippet）
error AIError {                    // AI 调用错误
    RateLimited
    ModelUnavailable
    BadResponse
}
```

### 13.1 Model 接口
```ail
interface Model {
    async fn generate(borrow self, prompt: string) -> Result<Response, AIError>
    effects { network }
}
```

### 13.2 Agent —— AILang 原生对象
```ail
agent ResearchAgent {
    goal: "research AI news"
    tools: [ web.search, database.query ]
}
```

编译器将 `agent` 编译为 `.ailmeta` `declarations[]` 条目（节选——省略 `span`，SPEC §25 #4）：
```json
{
  "kind": "agent",
  "name": "ResearchAgent",
  "goal": "research AI news",
  "tools": [ "web.search", "database.query" ]
}
```

### 13.3 Tool 系统
```ail
tool Search {
    fn query(text: string) -> List<SearchResult>
}
```

`tool` 自动生成多种下游产物：
- **OpenAI Tool Schema**
- **JSON Schema**
- **API 文档**

> `agent` 与 `tool` 是 v0.2.1 入 SPEC §2 关键字、SPEC §18 文法的顶级声明（详见 SPEC §15）。

---

## 14. `std.testing`（内置测试）

```ail
test "login success" {
    let result = login()
    assert result.ok
}
```

`ail test` 收集所有 `test` 块并执行。

---

## 15. 包发布安全（`ail publish`）

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

# 第三部分：生态主张

## 16. AILang 生态的优势

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

---

## 已对齐 / 遗留（v0.2.1 收敛后复核）

v0.2.1 SPEC 收敛后复核本文件先前的「待对齐」，结论：

1. ✅ **`agent` / `tool` 顶级声明**：已入 SPEC §2 关键字、SPEC §18 文法、SPEC §15 语义。
2. ✅ **`std.ai` 的 `agent { goal, tools }` / `tool { fn }`**：语法冻结（SPEC §15）。
3. 🔶 **`test` 是关键字**（SPEC §2）；`assert` 为 `std.testing` 内建（非关键字）。
4. ✅ **`unsafe { }` 块**：已入 SPEC §16 / MEMORY §13。
5. ✅ **路线图**：WHITEPAPER §12 与 DESIGN §15 已统一为 Phase 1–4。

**遗留开放**（见 SPEC §20）：`agent`/`tool`/`actor` 与 `struct`/`interface` 的精确边界（SPEC §20 #6）；`server`/`route` 语言级 vs 库级（SPEC §20 #6）。
