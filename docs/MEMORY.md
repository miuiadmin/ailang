# AILang Memory Model v0.3

> **状态：草案 / 供 review**
> 范围：内存区域、Ownership、Borrow、生命周期推导、自动释放、Unsafe、FFI、内存布局。
> 这是「AILang 能否达到 Rust 级安全与性能」的关键层。
> 语法基础见 [SPEC.md](./SPEC.md)（所有权概述 §10）；编译器实现见 [DESIGN.md](./DESIGN.md) §9；并发安全类型见 [CONCURRENCY.md](./CONCURRENCY.md)。

> **版本说明**：AILang 设计文档按**领域里程碑**标记——v0.2 语法 / v0.3 内存 / v0.4 并发 / v0.5 AI 层。它们共同描述同一门语言，互为补充而非替代。

---

## 1. 核心理念

| 范式 | 管理者 | 问题 |
|------|--------|------|
| C/C++ | 程序员（`malloc`/`free`/pointer） | 内存泄漏、野指针、double free |
| Java/Python | GC | GC 暂停、内存不可控、性能波动 |
| Rust | Ownership（编译期） | 生命周期语法复杂、学习成本高 |
| **AILang** | **Ownership + Automatic Lifetime Analysis** | **隐藏复杂生命周期语法** |

> AILang 的取舍：保留 Rust 的编译期内存安全与**无追踪 GC**（详见 §6），但把生命周期分析**完全交给编译器内部**，不暴露 `'a`。

---

## 2. 内存区域

```
Memory
├── Stack      函数帧、小型固定数据
├── Heap       复杂动态对象
├── Static     全局/编译期常量
└── Resource   文件/套接字/连接等需显式生命周期的外部资源
```

---

## 3. Stack 规则

小型、固定大小、Copy 语义的数据自动入栈：

```ail
let age: int = 20      // 栈上，main 帧
```

函数返回时栈帧自动释放——零成本、无跟踪。

## 4. Heap 规则

复杂/动态对象入堆，栈上持有指针：

```ail
let user = User { name: "Tom" }
// 栈：user (指针)  →  堆：User 对象
```

堆对象由**所有者**管理；所有者离开作用域时释放（见 §9）。

---

## 5. Ownership 核心规则

### Rule 1：每个资源只有一个 Owner
```ail
let file = File.open("a.txt")
// Owner = main 作用域
```

### Rule 2：默认 Move
```ail
process(file)          // 等价 process(move file)
print(file)            // ❌ AIL2001 Use after move
                       //    Resource: file  /  Moved to: process()
```

> 错误码方案：`AIL2xxx` = 所有权/内存类（如 `AIL2001 Use after move`）。统一错误码体系（`AIL2xxx` 编号空间）待后续设计阶段细化。

---

## 6. 值语义三分类（Copy / COW / Move）

| 类别 | 类型 | 行为 |
|------|------|------|
| **位复制 Copy** | `int` `uint` `float` `bool` `byte` + 全 Copy/COW 字段 struct | 赋值即复制，原值仍可用 |
| **COW 值类型** | `string` `List` `Map` `Set` | 赋值=共享缓冲（**原子引用计数**），修改=克隆；值语义，行为同 Copy；**`Send`**（SPEC §22 #2） |
| **Move** | `File` `Socket` `Connection` 等资源型 | 赋值即转移，原值不可用；使用须显式 `borrow`/`borrow_mut`（不可 `copy`） |

```ail
let a = 10
let b = a             // Copy：a、b 都可用
let s2 = s1           // COW：s1、s2 共享缓冲均可用，改 s2 才克隆
```

显式控制：`move` / `copy`（Copy 与 COW 类型）/ `borrow` / `borrow_mut`。

> v0.2.1 锁定 COW 值语义（类 Swift），消除 `&str`/`String` 双类型，**无追踪 GC**（COW/RC 引用计数仍存；SPEC §22 #2、SPEC §20 #7）。COW 字段计为 Copy-compatible（SPEC §22 #7）。

---

## 7. Borrow（不逃逸 → 无 `'a`）

AILang 不使用 Rust 的 `&` `&mut` `'a`，改用更自然的前缀：

```ail
fn print_user(user: borrow User) -> void { ... }       // 只读借用
fn update(user: borrow_mut User) { ... }               // 可写借用（独占）
```

- **`borrow x`**：只读，可同时多个；存在期间原值冻结。
- **`borrow_mut x`**：独占，同时只能一个，且不能与任何只读借用并存。
- **`borrow_mut` 与 COW**（SPEC §22 #3）：对 COW 类型，`borrow_mut` 要求**唯一所有权（rc == 1）**——共享值须先 `copy`/克隆；其下突变**原地**进行（独占 → 无需克隆），即 opt-out COW。
- **不逃逸**：禁止存入 struct 字段、禁止返回、禁止跨 `await` 存活 → 生命周期恒等于当前语句 → **无需 `'a` 标注**。

详见 SPEC §10.3、DESIGN §9.1（所有权状态机）。

---

## 8. 生命周期（隐藏，编译器推导）

用户**永不**写生命周期参数：

```ail
// Rust：fn get<'a>(...) -> &'a T
// AILang：
fn length(text: string) -> int {
    return text.length()       // 编译器内部建 Lifetime Graph，判定 text 生命周期 = 函数作用域
}
```

编译器内部的 Lifetime Analyzer 自动推导；推导失败时报清晰错误（而非要求用户加 `'a`）。

---

## 9. 自动资源释放（确定性析构）

资源在所有者离开作用域时**确定性地**释放（类 Rust `Drop`）：

```ail
fn read() {
    let file = File.open("a.txt")
    read(file)
}   // ← 编译器在此自动插入 file.close()
```

无需手动 `close`、无延迟。这是确定性析构安全管理的根因。

**Drop 语义精确化（SPEC §27 #5）**：字段 **Drop 顺序 = 声明顺序**（自上而下，确定性，与 Rust 一致）；panic 展开时 Drop 同序执行（SPEC §26 #3）。安全代码**结构上无循环引用**（单一所有者 + COW rc 为缓冲共享、非对象图共享）→ **v0.2 不引入 `Weak`/`Rc`/`Arc`**；需共享对象用 Actor / `Mutex<T>` / `Shared<T>`（§12）。递归类型须 `Box<T>`（SPEC §7.6）。

---

## 10. Resource trait

所有需显式释放的资源实现：

```ail
trait Resource {
    fn release(self)        // 消费 self（SPEC §22 #8）
}
```

`File` / `Socket` / `DatabaseConnection` 等实现之；编译器在 owner 作用域末插入 `release`（消费 owner，SPEC §22 #8）。

---

## 11. 数据竞争安全

跨线程访问由编译器静态检查（类 Rust `Send + Sync`；AILang 自有 auto-trait，SPEC §24 #6、STDLIB §7）：

```ail
spawn worker(data)      // 编译器检查 data 是否可跨线程移动
```

线程 A 写 `data`、线程 B 读 `data` 且无保护 → **拒绝编译**。

---

## 12. 并发安全类型

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

完整并发语义见 [CONCURRENCY.md](./CONCURRENCY.md)。

---

## 13. Unsafe 模式（逃生舱）

AILang 默认安全；系统级开发需要底层能力时：

```ail
unsafe {
    pointer_operation()    // 进入 Unsafe Zone，编译器降低限制
}
```

`unsafe` 块内的违规**由程序员负责**，且在 `ail publish` 时被审计（STDLIB §15）。

**`unsafe { }` 内必需操作的精确清单（SPEC §27 #4）**：① 裸指针（`raw_pointer<T>`）解引用 / 读 / 写；② 调用 `extern`（FFI）函数；③ 调用其他标 `unsafe` 的函数；④（预留）全局可变——v0.2 **不引入 `static mut`**（全局可变须走 `Mutex<T>`/`Shared<T>`，§12/CONCURRENCY §9），亦**无 union**（用 `enum` tagged union + `layout(C)` struct 对接 FFI）。清单外操作（约束断言、越界检查、整数溢出）**非 unsafe**——违约 = panic（确定性、安全）。

---

## 14. 指针设计

默认**无裸指针**。仅在 `unsafe` 内：

```ail
unsafe {
    let p: raw_pointer<int> = ...
}
```

---

## 15. FFI（外部函数接口）

调用 C：
```ail
extern c {
    fn printf(text: string)
}
```

**panic 跨 FFI 边界（SPEC §27 #4）**：panic 展开至 Task 边界或 `extern` 帧；触达 `extern` 帧（AILang 被 C 调入后 panic）→ **转进程 abort**（确定性终止，**非 UB**；C 无 Drop、不可跨语言栈展开）。正常 Task 内 panic 止于 Task 边界、不传染进程（FFI 边界为唯一例外）。C 的 `longjmp` 跨 FFI 进 AILang 帧 = UB（禁）。

调用 Rust（未来）：
```ail
extern rust {
    ...
}
```

---

## 16. 内存布局控制

系统/协议开发需 C 兼容布局：

```ail
layout(C) struct Packet {
    id: int
    data: Array<byte, 256>
}
```

`layout(C)` 强制 C 兼容的成员排列与对齐（用于 FFI、二进制协议）。

---

## 17. 零成本抽象

高级语法不牺牲性能：
```ail
for item in list { ... }
// 编译优化为指针循环，无迭代器对象开销
```

`Semantic Type`（`type UserId = int` + `meaning UserId: "用户标识"`——nominal 语义新类型，区别于透明别名 `type Alias = int`）、`Optional<T>`、`Result<T,E>` 等均零成本展开（DESIGN §12 类型映射）。

---

## 18. AILang vs Rust

| 能力 | Rust | AILang |
|------|------|--------|
| Ownership | ✅ | ✅ |
| Borrow Checker | ✅ | ✅ |
| 生命周期语法 | 复杂（显式 `'a`） | **自动推导（隐藏）** |
| GC（追踪式） | 无 | 无 |
| 性能 | ★★★★★ | ★★★★★（目标） |
| AI 理解度 | ★★★ | ★★★★★ |
| 语法难度 | 高 | 低 |

---

## 19. 模型总览

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

---

## 已对齐 / 遗留记录

1. ✅ **关键字/语法**：`unsafe`/`extern` 已入 SPEC §2；`layout(C)` 入 SPEC §18 文法（struct 属性前缀）；`raw_pointer<T>` 为标准库类型；独立 `impl T for X`：v0.2 不支持（用内联 `implements`，SPEC §7.5）；v0.3 计划引入（SPEC §20 #2）。
2. ✅ **`lock`**：`std.sync` 内建控制构造（非关键字，SPEC §13）；`Shared<T>`/`Mutex<T>` 归 `std.sync`。

**遗留与已决记录**：
- ~~Drop 语义细节~~（→ **已决**，见 §9 / SPEC §27 #5）。
- 自动 `release` 与 `Resource` trait 的精确契约（与 trait 默认实现的交互）。
- ~~`unsafe` 边界精确清单（裸指针解引用 / FFI / 共享可变）~~（→ **已决**：精确清单锁定 + panic 跨 FFI → abort + 无 `static mut` / 无 union；SPEC §27 #4）。
- ~~`string` 内存模型与是否引入最小 GC~~（→ **已决**：v0.2.1 锁定 COW 值语义，不引入最小 GC（无追踪 GC 详见 §6）；SPEC §22 #2、SPEC §20 #7）。
