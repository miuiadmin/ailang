# RFC 0009 · P0-4 FFI/ABI + 供应链最小闭环—— FFI-safe 白名单 / 调用约定 / 封送+修正 printf / 布局内征 / lockfile / .ailmeta 信任链 / §42 provenance

| | |
|---|---|
| **状态** | 草案（Draft v1）—— 待 review（已跑对抗式 workflow pass-1/pass-2/pass-3/pass-4/pass-5/pass-6，pass-6 报 5 confirmed [0H/2M/3L] 并已全部修正、待 pass-7 复验；目标收敛 0H/0M/0L，对齐 RFC 0001 v6 / 0002 v8 / 0003 v5 / 0004 v5 / 0005 v1 / 0006 v1 / 0007 v1 / 0008 v1）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结语义决断**：§1–§94 语义、56 关键字、110 决议均不变。**两处显式扩展**：① §27 `layout` 产生式体扩展（`layout(C)` → 可组合 `layout(C + packed + align(N))`，类比 RFC 0006 对 `stmt`/`call` 的产生式体扩展、属 Authorized RFC 演进通道 RFC 0005 §9）；② `extern` 的 `ident` 收紧为封闭 ABI 集合（名字解析校验、文法不变）。`CStr`/`CString` 为 std.core 类型（落 §34，与 `panic`/`assert` 同模块、默认加载免 import）、`size_of`/`align_of`/`offset_of` 为编译期内征（同 `panic`/`assert` 先例）、`ail.lock` 为工具链文件——均非关键字、零新关键字）|
| **日期** | 2026-07-27 |
| **分级** | **P0-4**（综合判断 [`docs/research/synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 第五/末优先级；§4 交叉印证 #6「FFI 封送/ABI 未规范，printf 示例自触 UB」+ #11「供应链 lockfile/签名/可复现构建缺失」；[`deep-review-2026-07.md`](../research/deep-review-2026-07.md) §6.1 包供应链 1.5 全规范最薄弱环 + §5.4 可观测维度 FFI UB 风险；[`spec-maturity-2026-07.md`](../research/spec-maturity-2026-07.md) §4.4 ABI/FFI 2.0 全规范最薄弱层 + §4.6 工具链 2.5）|
| **承接** | synthesis §6 P0-4（§25 扩独立 ABI 章节 + lockfile + §42 签名/provenance + .ailmeta registry 重建为权威）+ §4.4/§4.6 建议；deep-review §6.1 三发现（无 lockfile 构建不可复现 / .ailmeta 信任链未定义 / 补链地基就位）+ §可预期 printf UB；spec-maturity §4.4（FFI-safe 白名单 + 调用约定封闭集 + 修正 printf + packed/align/transparent + size_of/align_of/offset_of）+ §4.6（lockfile 格式 + 解析算法 + 可复现构建）；[RFC 0005](./0005-spec-governance.md) §6 panic 跨 FFI→abort（strength，重申）+ §9 RFC lifecycle 演进通道；[RFC 0008](./0008-diagnostics-taskhandle.md) §5 AILxxxx（FFI 违规可接入错误码）|

---

## 1. 动机

两轮深度评审把 **ABI/FFI 评为 2.0**（全规范最薄弱层）、**包供应链评为 1.5**（全规范最薄弱环）——两项垫底，且后者被规范自身的信任话术放大。三个旗舰级缺陷：

- **printf 示例自触 UB**——§25.3 / §61 的教学示例 `extern c { fn printf(text: string) }` 在 AILang **自身的**类型模型下即 C17 UB：`string` 按 §77 是胖指针 `{i64 len, i8* ptr}`（**无 NUL 终结**），直传给 `const char*` 字节错位；且 `printf` 是变参，§27 无 `...` 语法、按不匹配原型调用在 C17 6.5.2.2 下即 UB。**规范用自己的示例演示了自己的 ABI 不成立**——这是全场最刺眼的「未设计」。
- **无 lockfile，构建不可复现**——§30.1 依赖仅 semver 区间，`ail update` 明文「拉取符合约束的最新版」。同一源码两次构建可得**不同依赖闭包**。而语言层钉死了溢出 / Drop 序 / `.ailmeta` 字段序（§90/§78），测试层连 PBT seed 都钉死为固定常量（RFC 0003）——**语言与测试消灭了一切不确定性，依赖层却放任整个代码闭包漂移，同文档自相矛盾**。
- **`.ailmeta` 信任链未定义**——§5.1/§78「AI 可信任它，如同信任类型签名」+ §32/§43「不读源码直接 load metadata」构成信任教义。但类型签名在消费方每次编译被重查，**随包 `.ailmeta` 是无人复核的 JSON**——手改 `effects`/`is_pure` 即可让 AI 跳过防御性处理。RFC 0004 §10 已建 spec 作者作弊的威胁模型，唯独**供应链对手从未被纳入**。

**两者为何同批（P0-4）**。FFI/ABI 与供应链是**同一主题「与外部世界交互的信任与确定性」的两面**：FFI/ABI 管**跨语言边界的类型/调用正确性**；供应链管**跨组织边界的代码完整性/可复现性**。合在一起把 AILang 与 C 生态、与包注册表的交互从「口号层」抬到「最小可工程闭环」，并修复规范自带的 UB 示例。这是 P0 路线**最后一项**——完成后 deep-review §8 的 P0 五项齐备，方建议启动编译器。

**本 RFC 的性质**：① ABI/FFI 部分为**语言层扩展**（FFI-safe 白名单 + 封闭调用约定 + 封送 + 布局控制 + 内征）——含一处 §27 `layout` 产生式体扩展（显式标注、类比 RFC 0006）；② 供应链部分为**工具链层**（lockfile + 解析算法 + registry 重建 + provenance）——零语言改动。`extern` 的 `ident` 收紧为封闭集合是名字解析层的校验（文法不变）。不修改 §17 panic 模型、§25.1 unsafe 精确清单、§90 #4 panic 跨 FFI→abort 任一冻结决议——仅**扩展填补**「ABI 未规范」「供应链空白」两个被前轮反复引用却悬空的缝隙。

---

## 2. 设计目标

1. **不触动冻结语义决断**——§1–§94 语义、56 关键字、110 决议不变。panic 跨 FFI→abort（§90 #4，**strength**）保持并重申。
2. **最小自由度 / 封闭集合**——调用约定、FFI-safe 类型、布局标识符均为**封闭可枚举集合**（非任意 ident），最大化 AI 可预测性（AILang 核心卖点）。
3. **修正规范自带 UB**——printf 示例**必须**改为规范合规形（禁变参 + `CStr` 封送），消除「规范用自身示例演示自身 ABI 不成立」的刺眼缺陷。
4. **可复现构建优先**——lockfile + 解析算法 + registry 重建 `.ailmeta` 闭合「语言层确定性 vs 依赖层漂移」矛盾；签名等强密码学依赖（std.crypto 占位）**诚实推迟**，不假装已实现。
5. **与既有自洽**——FFI-safe 白名单与 §77 类型映射 / §18.6 Send-Sync、调用约定与 §25.3 `extern c/rust`、封送与 §17 panic 跨 FFI、布局与 §25.4 `layout(C)`、lockfile 与 §30.1 ail.toml / §42 发布审计完全对齐（§13 自洽核查表）。

---

## 3. 现状与缺口诊断

| 子项 | 现状（v0.2.1） | 缺口 | 决断（本 RFC） |
|---|---|---|---|
| FFI-safe 类型 | §25.3 任意类型可入 `extern` 签名（`string`/`printf`） | FFI-safe 白名单 + 非白名单类型拒绝 | **§4 白名单** |
| 调用约定 | `extern ident`，ident 不透明（§27）；仅 `c`/`rust` 两例 | 封闭 ABI 集合 + 平台默认映射 | **§5 封闭集合** |
| 封送 + string ABI | §77 string={len,ptr} 无 NUL、无封送章、printf 自触 UB | CStr 封送 API + 修正 printf | **§6 封送 + CStr** |
| 变参 | §27 无 `...` 语法；printf 变参按 UB | 变参决断（禁）+ 替代 | **§7 变参禁** |
| 布局控制 | §25.4 仅 `layout(C)` 单旋钮 | packed/align(N)/transparent + size_of/align_of/offset_of 内征 | **§7 布局控制 + 内征** |
| panic 跨 FFI | §17/§90 #4 已锁 abort 非 UB（**strength**） | （无缺口，重申） | **§8 重申** |
| lockfile | §30.1 仅 semver 区间、`ail update` 拉最新 | lockfile 格式 + 解析算法 + 可复现构建 | **§9 ail.lock** |
| .ailmeta 信任链 | §32/§78「如信任类型签名」但随包 JSON 无人复核 | registry 重建 .ailmeta 为权威 | **§10 信任链** |
| §42 发布安全 | §42 仅 4 步静态扫描 | provenance + registry 治理（签名推迟） | **§11 provenance** |

---

# Part A · FFI / ABI（扩展 §25 为独立 ABI 层）

## 4. FFI-safe 类型白名单

落地形态：§25 新增「FFI-safe 类型」小节 + §27 `extern_fn` 参数/返回类型校验规则。

**决断**：`extern` 函数的参数与返回类型**必须**为 FFI-safe 类型；非 FFI-safe 类型在 `extern` 签名中 = **编译错误**（接入 [RFC 0008](./0008-diagnostics-taskhandle.md) §5 错误码 **`AIL8xxx`** FFI/ABI 段，如 `AIL8001 ffi-unsafe-type`——非 `AIL6xxx`，6xxx 保持纯并发）。

**FFI-safe 类型（封闭集合）**：

| 类别 | 类型 | 说明 |
|---|---|---|
| 原始整数 | `int`（≡`int64`，§15.1/§77）、`uint`（≡`uint64`，§15.1）、`int8/16/32/64`、`uint8/16/32/64`、`bool`、`byte` | 定宽、C 兼容（默认宽度原语按 §15.1/§77 映射的定宽身份入白名单） |
| 原始浮点 | `float`（≡`float64`，§15.1/§77）、`float32`、`float64` | IEEE 754、C 兼容 |
| 裸指针 | `raw_pointer<T>`（`T` 须 FFI-safe） | 仅 `unsafe` 内解引用（§25.1）；`Copy` |
| C 布局 struct | `layout(C)` / `layout(C + packed)` / `layout(C + align(N))` struct（全部字段 FFI-safe） | 按值或 `borrow` 跨界；裸 `layout(packed)` / `layout(align(N))`（无 `C`）**非** FFI-safe（布局非 C ABI 兼容、不入白名单 → `AIL8001`） |
| 固定布局数组 | `Array<T, const N>`（`T` 须 FFI-safe） | **内联 `[T × N]` 固定布局**（§77 line 3095；非堆 COW、非 `List/Map/Set`）；FFI-safe **限定作 `layout(C)` struct 字段**（§25.4 `layout(C) struct Packet { data: Array<byte,256> }` 既有用法）——**裸 `extern` 参数 / 返回的 `Array<T,N>` 封送（C 数组 decay vs 按值聚合 ABI）v0.3 未规约**，留开放问题 #10 |
| 透明包装 | 含 `transparent` spec 的**合法** layout（`layout(transparent)` / `layout(C + transparent)` / `layout(transparent + align(N))` / `layout(C + transparent + align(N))`；`packed + transparent` 已互斥非法，§7.2）的 struct（单字段，字段须 FFI-safe） | FFI-safe 透明包装的判定基于 transparent **语义**（继承单字段 ABI）而非精确语法形——任何含 `transparent` spec 的**合法** layout 且单字段 FFI-safe 者，均归此类、按值跨界继承字段 ABI（§7.2 `C + transparent` 冗余合法、transparent 主导）；newtype FFI 包装（如 `CStr`） |
| C 串 | `CStr`（`layout(transparent)` over `raw_pointer<byte>`，§6.2） | NUL 终结 C 串，按值封送为 `const char*`（**仅 `CStr`**——`CString` 为 2-field owning 内部表示、非 FFI-safe，见下非白名单集） |
| `void` | 仅返回 | 无值返回 |

> **`layout(C) enum` 不入白名单**：§27 enum 产生式为 `enum := "enum" ident "{" variant* "}"`（**无 `layout?` 前缀**，仅 `struct` 接 `layout`），§77 enum = `tag + data`（tagged union，布局不稳）；§25.1 line 1378 明定 FFI 变体对接用「`enum` tagged union + `layout(C)` struct」——即 C 侧变体用 **`layout(C) struct`**（非 `layout(C) enum`）。故白名单不含「C 布局 enum」行（原 Draft 误列）；若未来需 C 式整数 repr enum，须先扩展 §27 enum 产生式（Authorized RFC 通道），留开放问题 #8。
>
> **C 函数指针（回调）推迟**：白名单不含 `extern fn pointer`——v0.2.1 无独立函数指针类型语法（§27 `fn` 仅声明、无 first-class fn-pointer 类型）。回调 C 需先引入函数指针类型，留开放问题 #9。

**非 FFI-safe（`extern` 签名中拒绝，触发 `AIL8001`）**：`string`（胖指针 `{i64 len, i8* ptr}`、无 NUL，§77）、**`CString`**（2-field owning `{buf: raw_pointer<byte>, len: uint64}`、非 `layout(transparent)` 单字段——内部表示、drop 释放分配，§6.2 明定不经 `extern`）、`List`/`Map`/`Set`（堆 COW、布局不稳）、`Optional<T>`/`Result<T,E>`（tagged、布局不稳）、`enum`（tagged union、布局不稳，§77；FFI 变体用 `layout(C) struct` 而非 enum，见上）、semantic 名义类型（§15.3：**若其 base 为 FFI-safe，则按 §77「codegen 同 base 布局」继承 base ABI、视为 FFI-safe**；base 非 FFI-safe 则随 base 拒绝）、泛型 struct（除非单态化为 `layout(C)`/`layout(transparent)` 且全部字段 FFI-safe）、trait 对象（`dyn`）、`TaskHandle`/`Channel`/`ActorHandle`（运行时句柄）。

> 注：`Array<T, const N>` 之前 Draft 误归「堆 COW」非 FFI-safe——实为 §77 内联固定布局（与 `List`/`Map`/`Set` 的堆 COW 不同），且 §25.4 既有 `layout(C) struct Packet { data: Array<byte,256> }` 已将其作 FFI-safe 使用，故修正归入白名单（递归约束 `T` 须 FFI-safe）。
>
> 白名单直接修复 printf UB 的根因之一：`string` **非** FFI-safe，**禁止**入 `extern` 签名——须封送为 `CStr`（§6）。

---

## 5. 调用约定封闭集合

落地形态：§25.3 扩展 + §27 `extern` 的 `ident` 收紧（名字解析校验、文法不变）。

**决断**：`extern <abi>` 的 `<abi>` 为**封闭可枚举集合**（非任意 ident），名字解析阶段校验 `abi ∈ 集合`，越界 = 编译错误：

| abi | 语义 |
|---|---|
| `c` | C 调用约定（平台默认 C ABI） |
| `system` | **平台默认 ABI（按目标三元组）**：x86_64 Linux/macOS/BSD = sysv64、x86_64 Windows = win64、aarch64 Linux/macOS = AAPCS64、Windows aarch64 = AAPCS64——跨平台可移植默认（具体映射由 ailc 按目标三元组决定；目标三元组枚举留开放问题 #11） |
| `stdcall` / `fastcall` / `vectorcall` | Windows 特定约定 |
| `aapcs` | ARM ABI |
| `c_unwind` | opt-in 跨边界继续展开（**v0.3 不入有效集合**——名字解析校验时视为越界 ident，触发 `AIL8002 abi-mismatch`；当前 panic 跨 FFI→abort §8；`c_unwind` opt-in 留 §15 开放问题 #2） |

> **v0.3 有效 ABI 集合** = `{ c, system, stdcall, fastcall, vectorcall, aapcs }`（封闭、可枚举）；`c_unwind`（§15 #2）与 `rust`（§15 #3）为**显式排除**项（非有效集合成员、名字解析即拒）。`extern ident` 的 `ident` 越此集合 = `AIL8002 abi-mismatch` 编译错误。
>
> **`system` 与平台 ABI 的语义重合**：`system` 在 aarch64 目标上 ≡ `aapcs`（同为 AAPCS64）、在 x86_64 Linux/macOS 上 ≡ `c`（同为 sysv64）——此重合是**预期行为、非冗余**：`system` 表达「跨平台可移植默认」意图（不钉死 ABI、随目标三元组），具体 ABI 名（`aapcs`/`stdcall`/...）表达「钉死某一 ABI」意图。二者在不同语境下各有用途，名字解析均接受（均属有效集合）。
>
> **平台—ABI 一致性校验**（闭合「有效集成员但目标不兼容」中间空白、消除静默误编 UB 入口）：平台特定 ABI 仅在兼容目标上合法——`stdcall` / `fastcall` **仅 x86（32 位）Windows** 目标合法（x86_64 Windows 用 win64、aarch64 Windows 用 AAPCS64，二者皆无 stdcall/fastcall；非 Windows 目标亦不合法）、`vectorcall` **仅 x86 或 x86_64 Windows** 目标合法（aarch64 Windows / 非 Windows 目标不合法）、`aapcs` 仅 ARM（aarch32/aarch64）目标；在**不兼容目标**上使用（如 `extern stdcall` on x86_64 Linux、`extern aapcs` on x86_64、`extern vectorcall` on aarch64 Linux、`extern vectorcall` on aarch64 Windows、`extern stdcall` on x86_64 Windows）= 编译错误。复用 `AIL8002 abi-mismatch`（语义扩展为「ABI 名非法**或**不适用于当前目标」——前者 ident ∉ 封闭集合、后者 ident ∈ 集合但 target 不兼容，二者均属「ABI 不可用」、复用同一码保错误码空间紧凑）。`c` / `system` 在**所有目标**上合法（由 ailc 按目标三元组解析具体 ABI，§15 #11）。此校验在名字解析**之后**的独立 pass 执行——名字解析仅判 in-set（§5 既有规则），target 兼容性须结合目标三元组、故为后续 pass。闭合：一个忠实遵循规范的编译器既不可将 `stdcall`-on-Linux 静默映射到错误 ABI（→ FFI 调用点 ABI 不匹配 = UB），亦不可产生无规范错误码的 codegen 失败——两条发散路径均被规则消除。

- §27 文法 `extern := "extern" ident "{" extern_fn* "}"` **不变**——`ident` 经名字解析校验为封闭集合成员（同 RFC 0006 §8 名字解析的类别校验思路）。
- §25.3 的 `extern rust { ... }` 空壳：**移除或推迟**——v0.3 仅规范 C ABI（`c`/`system` 及平台特定）；Rust ABI（非稳定、内部实现细节）不入封闭集合，留开放问题 #3。

---

## 6. 封送 + string ABI + CStr（修正 printf）

落地形态：§25 新增「封送」小节 + §77 string ABI 注 + §34 std.core 加 `CStr`/`CString`（FFI 核心封送类型、与 `panic`/`assert` 内征同模块、默认加载免 import；旧 Draft 「std.ffi」勘误为 std.core——冻结 §33 模块总览无 std.ffi 模块）。

### 6.1 string ABI（明确，闭合 §77 歧义）

`string` 的 ABI = 胖指针 `{ i64 len, i8* ptr }`（UTF-8 字节序列，**无 NUL 终结**；`len` 为 `i64`、`ptr` 为 `i8*`——与 §77 line 3093 一致）。此表示**非** C `const char*`——直传 C = 字节错位 + 越界读（UB）。故 `string` 非 FFI-safe（§4），跨 C 边界**必须**经 `CStr` 封送。

### 6.2 CStr / CString（std.core）

```ail
// CStr = layout(transparent) 单字段透传：按值跨界继承 raw_pointer<byte> 的 ABI = byte* = const char*
// （§7.2 transparent。UB 归因勘误：按值 layout(C) struct{byte*} 在主流 ABI（sysv64/win64/AAPCS64）下归 INTEGER 类、
// 单寄存器传指针值本身 ≡ 按值 byte*——无双重间接、亦非 UB。真正的 UB 源于 borrow：`borrow CStr`（无论 layout(C)
// 还是 transparent）会传指向 CStr 的指针 = struct{byte*}*，C 把该指针位模式当 const char* 读 → 双重间接/越界 UB。
// 选 transparent 的真实因果：使 CStr 落入 §6.4 对 transparent 类型的笼统 borrow 禁令（AIL8004），令 `borrow CStr`
// 在类型层不可表达——从根上闭合 borrow-UB 路径；同时 transparent 按值 = 字段 ABI = const char*。）
layout(transparent) struct CStr {
    ptr: raw_pointer<byte>          // 指向 NUL 终结的字节序列（Copy：裸指针为 Copy）
}                                   // FFI-safe（transparent over FFI-safe raw_pointer<byte>，§4）；按值跨界 = const char*

// string ↔ CStr 封送——std.core 内建方法【签名】（§34 std.core API）：
//   string.to_cstr(borrow self) -> CString     // 分配 + 拷贝 + 加 NUL（含内嵌 NUL 则失败 → panic 或返回 Optional，见开放问题 #1）
//
// ⚠️ 文法依赖声明（消除「§6.2 用独立 `impl` 块 = 引入新文法」与 §13「零新产生式」+ §27（无 impl 产生式）的矛盾）：
//   `string` 是 #[lang] 内建类型（§86 #5 / §77 {i64 len, i8* ptr}、非 struct）——无法用冻结合规的内联
//   `struct string { ...; fn to_cstr ... }` 形式附加方法。`string.to_cstr` 的方法定义语法依赖 §83 #2 既列的
//   v0.3【内建类型 / 外部类型方法附加（独立 `impl` 块）】特性——该特性尚无 owner RFC、其产生式 / 关键字未由任何 RFC
//   规范化（已 grep RFC 0005–0008，均未涉及）。本 RFC 仅规范【方法签名 + 语义】（to_cstr 的接收者模式、返回类型、
//   NUL 处理语义），不规范其定义文法；落地前须先指定内建类型方法附加的文法通道（§83 #2 owner RFC）。
//   故本 RFC 不出现字面 `impl string { ... }` 块（避免预设未规范的文法）——`string.to_cstr` 以签名声明呈现。

// CString 为 struct——as_cstr 用冻结合规的内联方法（§27 struct 内联 fn、§27 line 1486、无需独立 impl 块）：
struct CString {
    buf: raw_pointer<byte>,         // 拥有版（drop 释放分配）；非 FFI-safe（内部表示，不经 extern）
    len: uint64,
    fn as_cstr(borrow self) -> CStr // 借用：返回 CStr（Copy，按值；指向 CString 的 NUL 终结缓冲）——内联方法、冻结合规（§27 line 1486）
}
```

- **`CStr` = `layout(transparent)` 单字段**（非 `layout(C)`）——这是消除 printf UB 根因的关键：`layout(transparent)` 使 CStr **按值**跨界时**封送为其单字段的 ABI**（`raw_pointer<byte>` = `byte*` = `const char*`，单指针）；若用 `layout(C)`，则 `borrow CStr` 会把 `struct{byte*}*`（指向 struct 的指针）传给 C，C 把该指针的位模式当 ASCII 串读 → 双重间接 + 越界读 = UB。
- **封送规则**：`extern` 参数 `s: CStr`（按值）封送为 `const char*`（CStr 的 transparent 字段）；**禁止** `s: borrow CStr`（会产生指向 CStr 的指针 = 双重间接）。
- `to_cstr` 总分配拷贝（AILang string 无 NUL，须加）；`as_cstr` 借用 `CString` 的 NUL 终结缓冲，返回 `CStr`（Copy，按值）。

### 6.3 修正 printf 示例（旗舰修复）

```ail
// v0.2.1 教学示例（自触 UB，本 RFC 修正）——禁用：
// extern c { fn printf(text: string) }     // ❌ string 非 FFI-safe（§4）+ 变参未规约（§7 禁）

// v0.3+ 修正：固定元数 C 调用 + CStr 封送（CStr 按值跨界 = const char*）
extern c {
    fn puts(s: CStr) -> int32               // 固定元数；CStr = layout(transparent) 按值封送为 const char*（§6.2）
}

fn main() -> void {
    unsafe {
        let owned = "hello".to_cstr()        // string → CString（加 NUL）
        puts(owned.as_cstr())                // as_cstr 返回 CStr（Copy，按值）→ puts 跨界 = const char*
    }
}
```

> 落地时 §25.3 / §61 的 printf 示例**替换**为上述 `puts` 形（对齐 §2 设计目标 3「printf **必须**（MUST）改为规范合规形」）——规范不再用自身示例演示自身 ABI 不成立。原 printf 形（`extern c { fn printf(text: string) }`：`string` 非 FFI-safe + 变参禁 §7.1）**移除**，不留「保留 printf 仅标注示意」的退路（该退路与 §2 MUST 目标矛盾）。

### 6.4 `borrow` / `borrow_mut` 跨 FFI 的一般规则

`extern` 参数的 `borrow`（只读借用）/ `borrow_mut`（可变借用）传递模式跨 FFI 的合法性**仅对 C 布局 struct（`layout(C)` / `layout(C + packed)` / `layout(C + align(N))`）成立**——`borrow T` / `borrow_mut T` 跨 FFI 封送为「指向 T 的指针」（C `T*`），前提是 T 为 C 布局 struct（C 兼容排列、指针可被 C 侧解引用；`packed` / `align(N)` 仅调整字段间填充与对齐、不破坏「C 聚合可借用」前提）。`borrow`（只读）允许 C 侧读不写；`borrow_mut`（可变）允许 C 侧经 `T*` 写入调用方缓冲（变异跨 FFI 的合法用例，如 `read` / `gethostname` / `snprintf` / `sqlite3_bind_text` 的输出指针参数）——借检查器在调用点保证调用方侧独占（`borrow_mut` 的排他性），变异由 `unsafe` 块（§25.1，`extern` 调用本身已在 `unsafe` 内）授权负责。**其他 FFI-safe 类型一律按值跨界，`borrow` / `borrow_mut` 参数 = `AIL8004 ffi-marshalling` 编译错误**：

| 类型 | `borrow` / `borrow_mut` 跨 FFI 判定 | 理由 |
|---|---|---|
| C 布局 struct（`layout(C)` / `layout(C + packed)` / `layout(C + align(N))`） | **合法**（按值 / `borrow` / `borrow_mut`） | C 兼容聚合，`borrow` / `borrow_mut` = `T*`（C 侧可读 / 读写；借检查器保证 `borrow_mut` 调用方侧独占） |
| 原始整数 / 浮点 / `bool` / `byte` | **拒绝**（`AIL8004`） | C 无「引用整数」标准 ABI；欲传指针须显式 `raw_pointer<T>` |
| `raw_pointer<T>` | **拒绝**（`AIL8004`） | 本身即指针，`borrow raw_pointer<T>` = 指向指针的指针（双重间接）；欲传 `T**` 用 `raw_pointer<raw_pointer<T>>` |
| `Array<T,N>` | **拒绝**（`AIL8004`） | 裸 `borrow Array` 封送未规约（开放问题 #10）；仅作 `layout(C)` struct 字段时随 struct 借用 |
| `layout(transparent) struct`（含 `CStr`） | **拒绝**（`AIL8004`） | `borrow` transparent = 指向单字段 struct 的指针：对**指针型字段** transparent newtype（如 `CStr` over `raw_pointer<byte>`）即双重间接、C 误读指针位模式 → UB（§6.2 CStr）；对非指针型字段 transparent newtype 为指针-当-值误用。笼统禁 `borrow` 为**保守安全**（一律按值、transparent 按值 = 字段 ABI） |

**判定准则**：`borrow` / `borrow_mut` 跨 FFI **当且仅当** T 为 C 布局 struct（`layout(C)` / `layout(C + packed)` / `layout(C + align(N))`，C 聚合的可借用性）；其余 FFI-safe 类型按值跨界（`borrow_mut` 同 `borrow` 规则、仅语义为可变——§27 `mode := "borrow" | "borrow_mut" | "move" | "copy"`，`extern_fn` 复用标准 `params` 产生式故 `borrow_mut` 为合法语法、本规则为其跨 FFI 合法性）。此准则闭合「transparent 指针型 newtype 的 `borrow` / `borrow_mut` 双重间接 UB」一类漏洞（`CStr` 为典型例，§6.2）。

---

## 7. 变参决断 + 布局控制 + 内征

### 7.1 变参：v0.3 禁用

**决断**：v0.3 **禁止** C 变参（`...`）——§27 不引入 `...` 语法；`extern` fn 须固定元数。理由：① 变参与 AILang「无隐式转换 / 类型显式」哲学冲突（变参=类型擦除）；② printf 式变参是 UB 高发区（类型不匹配）；③ 替代路径：固定元数包装 / `va_list` 等价物推迟 v0.4。需变参的 C 函数（`printf`/`fprintf`）由 AILang 侧写**固定元数包装**对接（或标注推迟）。

### 7.2 布局控制（扩展 §25.4 + §27 `layout` 产生式体）

**决断**：`layout` 标识符扩展为可组合集合（**§27 `layout` 产生式体扩展——显式标注、唯一文法触及**）：

```ebnf
layout_spec := "C" | "packed" | "transparent" | "align" "(" int_lit ")"
layout      := "layout" "(" layout_spec ( "+" layout_spec )* ")"     // 可组合：layout(C + packed)
```

| 标识 | 语义 |
|---|---|
| `C` | C 兼容排列与对齐（既有 §25.4） |
| `packed` | 去除填充（紧凑布局，二进制协议 / 对齐敏感 FFI） |
| `transparent` | 单字段透传（继承字段 ABI——newtype FFI 包装，如 `layout(transparent) struct CInt(int32)`） |
| `align(N)` | 强制 N 字节对齐（SIMD / DMA / 共享内存） |

可组合：`layout(C + packed)`、`layout(C + align(16))`。

**组合合法性**（校验失败 = `AIL8003 layout-mismatch` 编译错误）：

| 规则 | 约束 | 违反 |
|---|---|---|
| `transparent` | 目标 struct **恰好 1 个字段**（0 或 ≥2 字段非法） | `layout(transparent) struct Empty {}` / `struct Pair(A,B)` → `AIL8003` |
| `align(N)` | `N` 为正整数且为 **2 的幂**（`1, 2, 4, 8, 16, …`，≤ 平台最大对齐）；**且** `N` ≥ 目标 struct 字段自然对齐之最大值（`align(N)` 只可**提升**对齐、不可**降低**自然对齐；欲降低须配 `packed`） | `align(3)` / `align(0)` → `AIL8003`；`layout(align(1)) struct { x: int64 }` → `AIL8003`（int64 自然对齐 8 > 1，对齐 Rust E0558「lower than natural alignment」） |
| `packed` + `align(N)` | `packed` 移除填充（字段间无 padding、struct 对齐 = 1）；与 `align(N)`（N>1）**互斥**（packed 强制对齐 1，align 强制 N>1，二者矛盾） | `layout(packed + align(16))` → `AIL8003` |
| `packed` + `transparent` | `transparent` 要求继承单字段**自然 ABI**（含字段对齐），`packed` 强制对齐 = 1——二者矛盾（packed 抹去 transparent 字段的对齐前提），**互斥**（对齐 Rust `#[repr(packed, transparent)]` 拒收） | `layout(packed + transparent) struct W(byte*)` → `AIL8003` |
| `C` + `transparent` | 合法但 `C` 冗余（transparent 单字段已 C 兼容）——允许、告警级提示 | — |
| `C` + `packed` | 合法（C 排列 + 去填充，二进制协议常用） | — |
| `layout` 适用目标 | 仅 `struct`（§27 `struct := layout? "struct" ...`）；**enum / type_alias / interface / trait 不接 `layout`**——文法已强制（仅 `struct` 产生式带 `layout?` 前缀，§27）；`layout` 前置于 `enum`/`type_alias`/`interface`/`trait` 为**语法错误**（RFC 0008 §5 AIL1xxx 词法/语法段），**不进入 AIL8003 语义校验** | —（语法层拒绝，无 AIL8003 触发场景） |
| 重复标识 | 同一 `layout(...)` 内同一 spec **不得重复**（spec 身份 = 基关键字——`align(N1)` 与 `align(N2)` 视为同一 spec `align`、不论 N 是否相等；故双 align 不论参数同异均构成冲突） | `layout(C + C)` → `AIL8003`；`layout(align(16) + align(32))` / `layout(align(8) + align(8))` → `AIL8003`（conflicting-align，对齐 Rust E0558） |

> `transparent` 的单字段约束是 FFI-sound 的前提（§6.2 `CStr` 依赖之封送为单指针）；多字段 transparent 会破坏「继承字段 ABI」的良定义性。

> 此为 §27 `layout` 产生式**体扩展**（原 `layout "(" ident ")"` → `layout_spec` 可组合形），类比 RFC 0006 对 `stmt`/`call` 的产生式体扩展——属 Authorized RFC 演进通道（RFC 0005 §9），显式标注为 Draft 决断、待 review。

### 7.3 布局内征（编译期内征，非关键字）

std.core / 编译期内征（同 `panic`/`assert` 先例，非关键字、经类型实参调用（**调用式随 RFC 0006 §7 终审决断 contingent**：方案 B = turbofish `size_of::<T>()`、方案 A = `size_of<T>()`——截至本 RFC Draft，§7 推荐方案 B 但尚未收敛、待 review + 对抗验证终审），RFC 0006 §7）：

| 内征 | 签名 | 说明 |
|---|---|---|
| `size_of<T>` | `() -> uint64` | `T` 的字节大小（编译期常量） |
| `align_of<T>` | `() -> uint64` | `T` 的对齐 |
| `offset_of<T>` | `(field: Field) -> uint64` | 字段偏移（FFI / 协议字段定位） |

> 返回类型为 `uint64`（规范定义的定宽无符号整数，§15.1；**非 `uintptr`**——`uintptr` 非本规范类型）。对标 Rust `core::mem::{size_of, align_of, offset_of}`（`usize`）/ C `sizeof`/`_Alignof`/`offsetof`（`size_t`）。FFI / 二进制协议 / 布局敏感代码的必需工具。

---

## 8. panic 跨 FFI 边界（既有 strength，重申）

**无缺口，重申既有决断**（§17 / §89 #3 / §90 #4）：

- panic 展开至 `extern` 帧 → **转进程 abort**（**非 UB**）——C 无 Drop、不可跨语言栈展开，abort 是唯一 sound 边界策略。
- C 的 `longjmp` 跨 FFI 进 AILang 帧 = **UB 禁止**。
- 两轮评审均肯定此为 **strength**（定义时点早于 Rust 同期 C-unwind 历史轨迹、UB 灰区更小——严格指时点与灰区，非表达力；表达力更窄：仅 abort-on-FFI，无 `c_unwind` opt-in，留 §15 开放问题 #2）。

> 本 RFC 不修改此决断；`c_unwind`（opt-in 跨边界继续展开）为**未来可选扩展**，v0.3 不入有效 ABI 集合（§5）——保持 abort 策略的简单 sound。

---

# Part B · 供应链最小闭环（工具链层）

## 9. lockfile（ail.lock）+ 解析算法 + 可复现构建

落地形态：§30 新增 `ail.lock` + §29 CLI 补 `ail lock`/校验。

### 9.1 ail.lock 格式

`ail.lock` 记录**已解析的依赖闭包**（每个依赖：精确版本 + 完整性校验和 + 来源），**签入 VCS**（类 Cargo.lock）：

```toml
# ail.lock —— 自动生成、签入 VCS、可复现构建的依据
version = 1                          # lockfile schema 版本

[[package]]
name = "http"
version = "1.2.3"                    # 精确解析版本（非区间）
source = "registry+https://registry.ailang.dev"
integrity = "sha256-2c26b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e7ae"

[[package]]
name = "database"
version = "2.1.0"
source = "registry+https://registry.ailang.dev"
integrity = "sha256-..."
```

- 字段：`name` / `version`（精确）/ `source`（registry URL / git / path）/ `integrity`（`sha256-<hex>`，对标 npm/go.sum）。
- **`integrity` 哈希对象**（显式定义，消除「哈希什么」的歧义）：`sha256` over **registry 下发的包源码归档字节流**（`.ailsrc.tar` 或等价——registry 从源码重建 `.ailmeta` 所用的同一归档，§10）——**非** `.ailmeta` 文本（后者由源码确定性派生；「校验源码归档即连带锁定 `.ailmeta`」**仅当消费方自源码重建 `.ailmeta` 时成立**——§32 消费方不读源码直接 load metadata、不重建，故被入侵 registry 仍可下发与源码归档不一致的伪造 `.ailmeta` 而不被消费方检出，见 §10/§11 #4 残留缺口）。`integrity = "sha256-" + hex(sha256(archive_bytes))`。
- **校验**：`ail build` / `ail run` 前比对依赖实际下载归档的 sha256 与 `integrity`，不匹配 → **错误**（拒绝构建）。
- **随包发布**：应用（二进制）与库（源）均签入 `ail.lock`；库的 `ail.lock` 标记 `lock-as-updated`（消费方可覆写，类 cargo）。

### 9.2 解析算法 + 可复现构建

- **MVS（最小版本选择）**：从 `ail.toml` 的 semver 约束，选满足全部约束的**最低**版本（对标 cargo），写入 `ail.lock`。
- `ail update`：重解析、更新 `ail.lock` 到符合约束的最新（**显式**操作，非每次构建隐式漂移）。
- **可复现构建**：`ail.lock` 在 → 同源码 + 同 lock → **字节一致的依赖闭包** → 闭合 deep-review §6.1「语言层钉死确定性、依赖层放任漂移」矛盾。
- **`.ailmeta` 数组字段规范化义务**（闭合「set/Map 派生数组序不稳」漏洞、按数组性质分类）：`.ailmeta` 中**由编译器内部 set/Map 派生**的数组字段（如 `effects[]`，§24 schema_version 0.2.0——其元素集合由 effect 分析得出、迭代序受 RFC 0007 §6 Hash seed 影响）序列化时**必须**为**规范化有序**——按字段自然序（字符串字典序 / 数值序）**预先排序后**再写出，**不得**按内存中 set/Map 的迭代序（后者由 RFC 0007 §6 显式登记为**未指定**、且 Hash seed 随机化 → 直接序列化会破坏字节级复现）。**声明序数组**（`input[]` / `fields[]` / `variants[]` / `errors[]` / `sigs[]`，§24）**必须保留源码声明序**、**不得**排序——其序承载 positional 与 ABI 语义（形参位置即 positional 身份、`layout(C) struct` 字段序 = 内存布局序 = C ABI（§25.4 line 1413）、enum/error `variants[]` 判别式序承载语义），且本就以有序 AST 承载、非 Map 派生、不存在迭代序漂移。**顶层数组**（`functions[]` / `types[]` / `errors[]` / `declarations[]`，§24 line 1332 顶层 schema）**同归声明序类、必须保留源码声明序**——顶层条目以有序 AST 承载、不得按字母序或 Map 迭代序序列化（强制 ailc 以有序结构存储顶层符号表、消除 HashMap 符号表实现导致的跨构建/跨实现 `.ailmeta` 字节漂移）；顶层 `errors[]`（全模块 error 枚举索引、§24 line 1332）同归声明序（与函数级 `errors[]` 同理、保留源码序）。**`errors[]` 为签名派生**（§24 line 1343「单一真源：≡ 签名 E（error 枚举）变体集」、§17 line 713）——与类型层 `variants[]`（§24 line 1352 error 条目）**同源同序**（同一 E 的声明序），故同归声明序类、**不得**字母序规范化（字母序化 `errors[]` 会与同文件 `variants[]` 形成同源双序；「须按 Hash seed 排序」前提对签名派生数组不成立——Hash seed 仅影响 effect 分析的 set 迭代序、不影响签名 E 的声明序）。**schema 版本关系**：数组规范化有序义务**适用于全部数组字段、无论其所属 schema 版本**；当前冻结 schema_version 0.2.0（§24）的数组字段如上归类，将来 RFC 0001 `examples[]` / RFC 0003 `properties[]` / RFC 0002 `contracts` rider 字段（schema_version 0.3.0，RFC 0005 §12 / 附录 D.2）落地后**同此义务**——rider 字段一经引入即按其性质归类（set 派生者规范化有序、声明序者保留）。registry 重建件（§10）的 schema_version 随 RFC 0001–0004 rider 落地而从 0.2.0 升至 0.3.0。（注：旧 Draft 把 `examples[]` / `properties[]` 标注为「§24」是悬空交叉引用——二者不在冻结 §24 0.2.0 schema 内，属未合并的 0.3.0 rider；本节勘误为按 schema 版本分别归属。）
- **确定性链结论口径收紧**（修正「构建产物字节级确定性」过强声称）：`.ailmeta` 顶层字段序固定（RFC 0005 §3 #3；§78 仅资料性）+ 数组字段规范化有序 → **`.ailmeta` 字节级确定性**；依赖闭包固定（§9.1 `integrity` 哈希锁定源码归档）→ **字节一致的依赖闭包**（**非**二进制产物整体——原生二进制复现另依赖 ailc/codegen lowering 确定性，见 §15 #6 未闭合、留 review）。三项前因中前两项只约束 `.ailmeta`、第三项只锁定依赖闭包、均**不触及**原生二进制产物（LLVM codegen / 链接 / 内嵌构建路径与时间戳）；§5.1「双产物」明定构建产物 = 原生二进制 + `.ailmeta`，本 RFC 闭合的为 `.ailmeta` 一侧 + 依赖闭包，二进制 lowering 一侧诚实地不在本 RFC 闭合范围。（此条确立 `.ailmeta` **数组字段规范化有序**义务的规范性来源——为 §9.2 新立义务、闭合 RFC 0007 §6 矛盾；**顶层字段序固定**义务的来源为 RFC 0005 §3 #3（§3 #3 原文「固定字段排序」仅指顶层字段序）；§78 仅为 Part IV 资料性实现层描述、不构成本义务权威，见 §10。）

> deep-review §6.1 明言「缺的是连接，不是构件」——§78 确定性输出 + §31 强制随包源码已就位，加 `ail.lock` 记哈希**显著降低**元数据伪造与下发篡改两类攻击面（**非完全闭合**——残留缺口**至少包括**：首次 registry 拉取的 TOFU 信任、作者签名 / 账号接管、以及 **registry 自身被入侵**（§10 把 `.ailmeta` 权威集中于 registry、§32 消费方不重建、§9.1 `integrity` 哈希覆盖**源码归档**而非 `.ailmeta` 文本，故被入侵 registry 可下发伪造 `.ailmeta` 而不被消费方检出），见 §11 #4 / #5 签名推迟）。

---

## 10. `.ailmeta` 信任链（registry 重建为权威）

落地形态：§32 / §42 扩展 + registry 治理规则。

**决断**：发布包的 `.ailmeta` **不以作者提交件为权威**——registry（包仓库）从**随包源码**（§31 强制）重新编译生成 `.ailmeta`，**以重建件为权威**（docs.rs 范式）。此为 RFC 2119 **必须（MUST）** 级义务：registry **必须**以重建件为消费方下载的 `.ailmeta` 真源，**不得**直接转发作者提交的 `.ailmeta`（若作者提交件与重建件分歧，**必须**以重建件为准、**必须拒绝发布**该版本——「告警但放行」不满足 MUST 级信任链义务，分歧本身即篡改信号）。

```
作者发布 → registry 收源码 + ail.toml
        → registry 编译 → 重新生成 .ailmeta（确定性输出：顶层字段序固定（RFC 0005 §3 #3；§78 仅资料性）+ 数组字段规范化有序 §9.2）
        → 重建件为权威（消费方下载的 .ailmeta = registry 重建件，非作者提交）
```

> **`.ailmeta` 确定性的规范性来源**（消除「§78 作权威」的分区冲突）：`.ailmeta` 字段集与 schema 的规范性来源为 **§24**（`.ailmeta` schema）。确定性输出义务**分两层、各承其源**：① **顶层字段序固定**义务的权威来源为 **RFC 0005 §3 #3** Conformance（「生成符合 §24 schema 的 `.ailmeta`…输出确定性（固定字段排序）」）——该义务**非本 RFC 首创**；② **数组字段规范化有序**义务为本 RFC **§9.2 新增**（RFC 0005 §3 #3 仅及顶层字段序、未及数组内部元素序）——补全「Hash 随机化下 set/Map 迭代序不可复现」的最后一公里。两层叠加 → 字节级复现。**§78 仅为 Part IV 资料性的实现层排序描述、不构成任一层义务的规范性权威**（与 RFC 0005 §5 normative/informative 分区一致）。
- **闭合「手改 effects/is_pure 让 AI 跳过防御」**：作者无法伪造 `.ailmeta`——任何字段篡改在 registry 重建时被覆盖。
- **完整性**：`ail.lock` `integrity` 哈希对象为 **registry 下发的源码归档**字节流（§9.1 定义）——**非**重建 `.ailmeta` 文本；消费方校验对象为该源码归档，仅防**源码归档下发篡改**（**不**覆盖伪造 `.ailmeta`——见下条 residual gap：§32 消费方不重建，被入侵 registry 仍可下发与源码归档不一致的伪造 `.ailmeta`）。
- **配合 lockfile**：deep-review §6.1 末两类攻击面（元数据伪造 / 下发篡改）由「registry 重建 + lockfile 哈希」**显著降低**（**非完全闭合**——残留缺口见 §11 #4 / #5：首次 TOFU 信任、作者签名 / 账号接管、**以及 registry 自身被入侵**。本 §10 把 `.ailmeta` 权威集中于 registry，§9 `integrity` 哈希覆盖**源码归档**而非 `.ailmeta` 文本、§32 消费方不重建——故**被入侵 registry 可下发伪造 `.ailmeta` 而不被消费方检出**，密码学签名为其唯一长期闭合手段、推迟至 §11 #4/#5）。
- **作者签名 / 账号接管**：仍为独立缺口（非本 RFC 闭合），留 §11 签名推迟。

> 此决断使 §5.1/§78「AI 可信任 `.ailmeta` 如同类型签名」从**教义**升级为**可验证信任链**：信任源于 registry 确定性重建（可复算，义务由 RFC 0005 §3 #3 + 本 RFC §9.2 承载），而非作者自报。

---

## 11. §42 发布安全扩展（provenance + 治理，签名诚实推迟）

落地形态：§42 扩展（现 4 步静态扫描 → 加 provenance + 治理）。

**决断**：`ail publish` / registry 扩展：

1. **既有 4 步静态扫描保留**（与 §42 line 2005–2011 逐一对应：**包扫描（package scan）/ 类型安全（type safety）/ unsafe 审计（unsafe usage）/ 依赖审计（dependency audit）**——非「effect 策略 / 错误完整性」，后者为旧 Draft 误述）。
2. **provenance（出处记录）**：构建出处记录（SBOM-lite）由 **registry 在重建 `.ailmeta` 时生成**（§10 registry 重建为权威——作者 `ail publish` 时 §42 仅 4 步静态扫描、§31 为源码分发，**无目标构建可测「构建环境」**，故 provenance 的「构建环境」字段真源为 registry 重建环境、非作者侧）：依赖闭包（来自 `ail.lock`）+ 构建环境（**registry 重建所用编译器版本 / 目标三元组**，§10）+ 校验和（§9.1 源码归档哈希），**随包发布、可审计**。作者侧 `ail publish` **不生成** provenance（无构建环境可测），仅提交源码 + `ail.lock`；provenance 由 registry 重建时产出、与重建 `.ailmeta` 一同下发。
3. **registry 治理**：registry 重建 `.ailmeta`（§10）+ 拒绝「重建件与声明 `.ailmeta` 分歧」的包（篡改检测）+ 强制 `ail.lock` 完整性校验。
4. **签名（诚实推迟）**：v0.3 **不引入**密码学签名——std.crypto 仍为占位（§33.1），无成熟签名原语可用。provenance 记录**无签名**（仅校验和 + 闭包），防伪造/篡改靠 §10 registry 重建 + §9 lockfile 哈希。**残留缺口诚实登记**：§10 把 `.ailmeta` 权威集中于 registry、§9 `integrity` 哈希覆盖**源码归档**而非 `.ailmeta` 文本、§32 消费方不重建——故**被入侵的 registry 可下发伪造 `.ailmeta` 而不被消费方检出**（首次 TOFU 信任 + 作者签名 / 账号接管同理），此为签名推迟的直接代价。**正式签名（sigstore 式）推迟至 std.crypto 成熟**（开放问题 #5），明文登记为 gap，不假装已实现。

> 对标 cargo（Cargo.lock + sigstore）/ go modules（go.sum + 校验和 DB）/ npm（package-lock + provenance）。v0.3 达到「校验和 + 重建权威」最小闭环；签名为下一台阶。

---

## 12. 与 RFC 0005 / 0007 / 0008 的交叉更新

| 既有引用 | 现状 | 本 RFC 后 |
|---|---|---|
| RFC 0005 §6 panic 跨 FFI→abort | UB 分类（strength） | 重申不变（§8）；`c_unwind` opt-in 留未来（§15 #2） |
| RFC 0005 §3 #3 Conformance `.ailmeta` 确定性 | 确定性义务的规范性权威来源 | 本 RFC §9.2 承载该义务（顶层序 + 数组规范化有序 → `.ailmeta` 字节级复现 + 声明序数组保留源码序）；二进制 lowering **不在**本 RFC 闭合（见 §15 #6）；§78 仅资料性实现描述 |
| RFC 0005 §9 RFC lifecycle 演进通道 | Authorized RFC 可改冻结 | 本 RFC §27 `layout` 产生式体扩展经此通道（显式标注） |
| RFC 0007 §6 Hash/Map/Set 迭代序 | 显式登记为**未指定**（seed 随机化） | 本 RFC §9.2 因此要求 `.ailmeta` 中**由 set/Map 派生的数组**（`effects[]`）**预先规范化排序**后序列化（不得按 set/Map 迭代序）；**声明序数组**（`input[]` / `fields[]` / `variants[]` / `errors[]` / `sigs[]`）保留源码序、不受影响（`errors[]` 签名派生 ≡ E variants、§24 line 1343、不受 Hash seed 影响）——闭合 Hash 随机化 vs `.ailmeta` 字节级复现矛盾。⚠️ **回引同步**：RFC 0007 §6.1（line 123）+ §13 OQ#6（line 307）回引本 RFC §9.2 时用笼统「`.ailmeta` 数组字段必须预先规范化排序」措辞——该措辞**先于** §9.2 的收窄（set/Map 派生预排序 + 声明序保留），权威规则以本 RFC §9.2 为准；RFC 0007 两处回引的笼统措辞待其下一轮验证（pass-20）同步收紧（本 RFC 不越界改 RFC 0007 正文、pass-19 验证进行中） |
| RFC 0007 §7 cast `as` | float→int saturating | `char`/FFI 类型转换经 `as` 不变；FFI-safe 校验独立 |
| RFC 0008 §5 AILxxxx 错误码 | FFI 违规无码 | FFI-safe 违规 / ABI 不匹配 / 布局违规 / 封送违规接入 **`AIL8xxx`**（FFI/ABI/布局/codegen 段，非 6xxx）；种子 `AIL8001 ffi-unsafe-type` / `AIL8002 abi-mismatch`（语义含「越封闭集合」与「目标平台不兼容」两类，§5）/ `AIL8003 layout-mismatch` / `AIL8004 ffi-marshalling`（封送违规：`borrow` 跨 FFI 用于非 C 布局 struct，含 transparent 指针型 newtype 如 `CStr`，§6.4） |
| RFC 0008 §4 诊断协议 | 编译期诊断 JSON | FFI-safe 违规诊断经同一协议输出（带 AIL8xxx code + `category:"ffi"`） |
| §17/§77 string ABI | `{i64 len, i8* ptr}` 无 NUL（歧义源） | §6.1 明确（i64/i8* 对齐 §77）+ CStr `layout(transparent)` 封送闭合 printf UB |
| §42 4 步扫描 | 包扫描 / 类型安全 / unsafe 审计 / 依赖审计 | §11 provenance + 治理**追加**（4 步不变，§11 #1 校正旧 Draft 误述） |
| §30.1 ail.toml / §42 | 口号层 | §9 lockfile + §11 provenance 补齐工具链 |

---

## 13. 不变量与自洽核查

| 本 RFC 条款 | 对齐的既有规范 | 自洽性 |
|---|---|---|
| §4 FFI-safe 白名单 | §77 类型映射 / §25.3 extern | ✅ 收紧（拒非白名单） |
| §4 `string` 非 FFI-safe | §77 string=`{i64 len, i8* ptr}` 无 NUL | ✅ 一致（printf UB 根因） |
| §4 `Array<T,const N>` FFI-safe | §77 line 3095 内联 `[T×N]` 固定布局 + §25.4 `layout(C) struct Packet { data: Array<byte,256> }` 既有用法 | ✅ 修正旧 Draft 误归「堆 COW」（Array 内联、非 List/Map/Set） |
| §4 不含 `layout(C) enum` | §27 `enum := "enum" ident "{" variant* "}"`（无 `layout?` 前缀）/ §25.1 用 `layout(C) struct` 对接 FFI 变体 | ✅ enum tagged union 非白名单；C 式 enum repr 留 §15 #8 |
| §4 semantic 继承 base FFI-safety | §77 line **3098** semantic「codegen 同 base 布局」（3099 为 `Optional<T>`） | ✅ 消除旧「除非 transparent」不可达子句 |
| §4 不含 fn pointer | §27 `fn` 仅声明、无 first-class fn-pointer 类型 | ✅ 回调推迟 §15 #9 |
| §4 默认宽度原语 `int`/`uint`/`float` FFI-safe | §15.1 `int`(默认 i64)/`uint`(默认 u64)/`float`(默认 f64) + §77 line 3089/3091 `int`→`i64`、`float`→`f64` 独立行 + §25.4 line 1413–1416 `layout(C) struct Packet { id: int; data: Array<byte,256> }` 既有用法 | ✅ 默认宽度原语按 §15.1/§77 映射的定宽身份入白名单（`int`≡`int64`、`uint`≡`uint64`、`float`≡`float64`）；`Packet{id:int}` 自洽、不再被白名单封闭集拒 |
| §4↔§7.2 组合布局 FFI-safe 分类 | §4 白名单「C 布局 struct」含 `layout(C)` / `layout(C + packed)` / `layout(C + align(N))`；§6.4 borrow 同步归入「C 布局 struct」享合法性；裸 `layout(packed)` / `layout(align(N))`（无 `C`）非 FFI-safe；transparent 组合（含冗余 C / 附加 align(N)）FFI-safe 身份由 transparent **语义**判定（§4「透明包装」行） | ✅ §4/§6.4 与 §7.2 新组合布局分类一致（避免新特性 FFI-safe 身份与 borrow 资格悬空） |
| §5 封闭 ABI 集合（v0.3 有效集 6 项）+ 平台—ABI 一致性校验 | §25.3 `extern c/rust`（rust 推迟 §15 #3）/ c_unwind 推迟 §15 #2 | ✅ 收紧 ident 越界 AIL8002 + 平台特定 ABI 用于不兼容目标复用 AIL8002（§5 末段，闭合「in-set 但目标不兼容」UB 入口） |
| §5 `extern ident` 文法不变 | §27 `extern := "extern" ident { }` | ✅ 名字解析校验 |
| §6 CStr = `layout(transparent)` | §7.2 transparent 单字段透传 / §4 白名单 | ✅ 按值封送 = `const char*`；transparent 触发 §6.4 笼统 borrow 禁令（AIL8004）从根闭合 borrow 双重间接 UB（F11 勘误：UB 源于 borrow 非 by-value，§6.2 注释已修正） |
| §4 `CString` 非 FFI-safe（白名单仅 `CStr`） | §6.2 `CString { buf, len }` 2-field owning（非 transparent 单字段） | ✅ 白名单 C 串行仅列 `CStr`；`CString` 入非 FFI-safe 拒绝集（消解「CStr/CString 同为 transparent FFI-safe」三方矛盾：§4 白名单 vs §6.2 2-field vs §7.2 单字段规则） |
| §6.2 封送方法（`string.to_cstr` / `CString.as_cstr`）文法合规 | §27 struct 内联 fn（`as_cstr`、§27 line 1486）+ §83 #2 v0.3 内建类型方法附加（`string.to_cstr`、unowned） | ✅ 本 RFC 仅规范方法【签名+语义】、**不出现字面 `impl` 块**——零新产生式成立（§27 无 impl 产生式）；`CString.as_cstr` 冻结合规内联（CString 为 struct）、`string.to_cstr`（string 为 #[lang] 内建非 struct、无法内联）签名层声明、定义文法依赖 §83 #2 v0.3 独立 impl 块特性（unowned、落地前须先指定 owner RFC） |
| §6.1 string ABI `{i64 len, i8* ptr}` | §77 line 3093 | ✅ 明确化（修正旧 Draft 误用 `uintptr`——非规范类型） |
| §7.2 layout 产生式体扩展 + 组合合法性 | §27 `layout "(" ident ")"`（显式演进） | ⚠️ 文法体扩展（标注）+ §7.2 组合合法性表 8 行：**5 条触发 AIL8003**（transparent 单字段 / align(N) 2 的幂+下界 / packed+align(N) 互斥 / packed+transparent 互斥 / 重复标识）+ **2 条合法**（C+packed / C+transparent 冗余告警）+ **1 条 AIL1xxx 语法**（layout 仅 struct、§27 文法强制、pass-4 F8 改归 AIL1xxx 非 AIL8003） |
| §7.3 size_of/align_of/offset_of 返 `uint64` | 经类型实参调用（**调用式 contingent on RFC 0006 §7 A/B 终审**：方案 B = `size_of::<T>()`、方案 A = `size_of<T>()`）/ 同 panic 内征先例 | ✅ 零新关键字（修正旧 `uintptr`→`uint64`） |
| §8 panic 跨 FFI→abort | §17 / §89 #3 / §90 #4 | ✅ 重申不变 |
| §9.1 ail.lock `integrity` 哈希对象 = 源码归档字节 | §10 registry 从源码重建 / §31 强制随包源码 | ✅ 显式定义（消除「哈希什么」歧义） |
| §9.2 `.ailmeta` 字节级确定性（数组规范化有序 + 声明序数组保留） | RFC 0005 §3 #3（确定性义务权威）/ RFC 0007 §6（Hash 迭代序未指定）/ §25.4 `layout(C)` 字段序 = ABI | ✅ MUST 仅覆盖 set/Map 派生数组（`effects[]`）；声明序数组（`input[]`/`fields[]`/`variants[]`/`errors[]`/`sigs[]` + **顶层** `functions[]`/`types[]`/`errors[]`/`declarations[]`、§24 line 1332 顶层 schema）保留源码序（`errors[]` 签名派生 ≡ E variants、§24 line 1343、与类型层 `variants[]` 同源同序、不入 set/Map 派生类；顶层数组以有序 AST 承载、强制 ailc 有序存储顶层符号表消除 HashMap 漂移）；闭合 Hash 随机化 vs `.ailmeta` 字节级复现矛盾（§78 仅资料性）；结论口径收紧为 `.ailmeta` 字节级 + 依赖闭包字节一致（二进制 lowering 见 §15 #6 未闭合） |
| §9.2 schema 版本归属（0.2.0 vs 0.3.0 rider） | §24 schema_version 0.2.0 / RFC 0005 §12「schema_version 无关」handoff（0.2.0→0.3.0 升级由 rider 承担）/ 附录 D.2 语言与类型术语（schema_version 当前 0.2.0、RFC 0001–0004 预留 0.3.0） | ✅ `effects[]` 归 §24（0.2.0、set/Map 派生）；`errors[]` 签名派生、归声明序类（同 §9.2）；`examples[]`/`properties[]`/`contracts` 为 RFC 0001–0003 rider（0.3.0）；规范化义务跨版本适用 |
| §9 ail.lock + MVS | §30.1 ail.toml semver / §42 依赖审计 | ✅ 工具链层补齐 |
| §10 registry 重建 .ailmeta（MUST 级） | §31 强制随包源码 / RFC 0005 §3 #3 确定性 | ✅ 闭合信任链（§24 schema 为规范性来源，§78 资料性） |
| §11 §42 4 步校正 + provenance（签名推迟） | §42 line 2005–2011 / §33.1 std.crypto 占位 | ✅ 4 步与冻结一致 + 诚实登记 gap |
| FFI 违规接入 AIL8xxx | RFC 0008 §5 AIL8xxx 段 + DiagnosticCategory `Ffi` | ✅ 6xxx 保持纯并发 |
| 零新关键字 | §9（56）；`CStr`/`CString` std.core 类型（§34）、`size_of` 等内征、layout 标识 `packed`/`transparent`/`align` 为标识符非关键字 | ✅ 已核查 |
| 零新产生式类别 | §27 `layout` 体扩展（非新类别）/ `extern` 文法不变 | ✅ 已核查 |

---

## 14. 落地映射（合并进 AILANG.md 的位置）

| 本 RFC 内容 | 落地位置 | 性质 |
|---|---|---|
| §4 FFI-safe 白名单 | §25 新增「FFI-safe 类型」+ §27 extern_fn 校验 | 规范性 |
| §5 封闭 ABI 集合 | §25.3 扩展（rust 推迟） | 规范性 |
| §6 封送 + CStr + string ABI | §25 新增「封送」+ §77 注 + §34 std.core 新增 `CStr`/`CString`（FFI 核心封送类型，与 `panic`/`assert` 内征同模块、默认加载免 import；**非新模块**——冻结 §33 模块总览 15 模块不变、无 std.ffi；旧 Draft 「§38 std.ffi」勘误：§38 冻结为 std.http） | 规范性 |
| §6.3 修正 printf | §25.3 / §61 示例替换 | 规范性（示例） |
| §7.1 变参禁 | §27 / §25.3 注 | 规范性 |
| §7.2 布局控制 | §25.4 扩展 + §27 `layout` 产生式体（显式演进） | 规范性（含文法） |
| §7.3 内征 | §34 std.core / §77 类型表 | 规范性（std API） |
| §8 panic 跨 FFI | §17 / §90 #4（重申，无改） | — |
| §9 ail.lock | §30 新增 + §29 CLI `ail lock` | 规范性（工具链） |
| §10 registry 重建 | §32 / §42 扩展 | 规范性（信任链） |
| §11 provenance | §42 扩展（签名 gap 登记） | 规范性 + gap 登记 |

---

## 15. 开放问题

1. **`to_cstr` 内嵌 NUL 行为**——§6.2 string 含内嵌 NUL 时 `to_cstr`（C 串 NUL 截断）会丢数据。panic（严格）或返回 `Optional<CString>`（宽容）？推荐 panic（安全代码不静默吞错，对齐 RFC 0007 §8 字符串严格立场），留 review。
2. **`c_unwind` opt-in 跨边界展开**——§5/§8 当前 abort-on-FFI；`c_unwind`（对标 Rust extern "C-unwind"）允许 panic 跨 FFI 继续展开而非 abort。v0.3 不引入（保简单 sound），留未来 ABI 扩展。
3. **`extern rust` 去留**——§5 移除 `extern rust` 空壳（Rust ABI 非稳定）；或保留为「未来」标注。推荐移除（封闭集合不含非稳定 ABI），留 review。
4. **`offset_of<T>(field)` 的 field 表达**——§7.3 `offset_of` 的 `field` 参数如何表达（字符串名 / 类型化字段句柄）？Rust 用 `std::mem::offset_of!(Type, field)` 宏；AILang 无宏系统，倾向类型化 `Field` 句柄或字符串名，留 std API 设计。
5. **签名推迟的里程碑**——§11 签名（sigstore 式）推迟至 std.crypto 成熟；触发条件 = std.crypto 提供稳定签名原语（ed25519 等）。具体版本（v0.4? v1.0?）留治理。
6. **lockfile 与可复现构建的编译器确定性**——§9 可复现构建依赖 ailc 输出确定性；但 ailc 自身版本变化可改变 lowering。是否锁定 ailc 版本进 lockfile / provenance？推荐进 provenance（构建环境记录），留 review。
7. **私有 registry / git 依赖**——§9 `source` 支持 registry URL；git 依赖 / path 依赖 / 私有 registry 的解析与校验细节留工具链实现。
8. **`layout(C) enum`（C 式整数 repr enum）**——§4 白名单不含（§27 enum 无 `layout?` 前缀、§77 enum = tag+data tagged union）。若未来需对接 C enum / int 常量，须先扩展 §27 enum 产生式（Authorized RFC 通道）定义 `layout(C)` enum 的 C 整数 repr（fieldless 或显式判别式类型）。v0.3 用 `layout(C) struct` + 显式 tag 字段作替代，留 review。
9. **C 函数指针（回调）**——§4 白名单不含 `extern fn pointer`（v0.2.1 无 first-class 函数指针类型语法）。回调 C 需引入函数指针类型（如 `extern "c" fn(byte*) -> void` 作类型），其与 `extern` fn 声明、闭包捕获的边界留 std.core（§34）后续设计。
10. **`Array<T,N>` 作裸 `extern` 参数 / 返回的 ABI**——§4 限定 `Array` 仅作 `layout(C) struct` 字段时 FFI-safe；作**裸** `extern` 参数或返回值（如 `extern "c" fn(Array<byte,4>) -> void`）的封送 ABI（C 数组按值？退化为指针？）v0.3 **未规约**，一律触发 `AIL8001 ffi-unsafe-type` 拒绝。若未来需裸传定长数组跨 FFI，须先定 ABI 惯例（对标 C「数组参数退化为指针」语义），留 review。
11. **目标三元组与平台默认 ABI 的映射**——§5 `system` ABI 定义为「按目标三元组确定的平台默认 ABI」（aarch64 Linux/macOS = AAPCS64 等），但 ailc 的**目标三元组命名空间**（`<arch>-<vendor>-<os>-<env>` 格式、支持的目标列表、三元组→默认 ABI 映射表）v0.3 **未规约**，留工具链 / 后端 ABI 设计。此映射关乎可复现构建（§11 provenance 记录目标三元组）与 FFI ABI 选择的确定性。

---

## 16. 收敛轨迹

**Draft v1 → pass-1（已完成）→ 修正后待 pass-2**。pass-1 对抗式 workflow（与 RFC 0008 并行）报 **2H / 10M / 9L = 21 条**（RFC 0009 部分），两条 H 已修正：① **Array FFI-safe 误归类**——§77 内联 `[T×N]` 非「堆 COW」，修正归入白名单（递归约束 T FFI-safe），对齐 §25.4 既有 `layout(C) struct Packet { data: Array<byte,256> }`；② **CStr 封送 UB**——`layout(C) struct CStr` 传 `borrow CStr` = 双重间接 → UB，改为 `layout(transparent) struct CStr` 按值封送为 `const char*`（puts 示例同步去 `borrow`）。其余 M/L 修正：移除悬空 `layout(C) enum` 白名单行 + fn pointer 推迟（§15 #8/#9）、semantic 继承 base FFI-safety（§77 codegen 同 base）、`uintptr`→`i64`/`uint64`（非规范类型）、§42 4 步校正（与 §42 line 2005–2011 一致）、c_unwind 引用 §7→§15 #2 + v0.3 有效 ABI 集、layout_spec 8 条组合合法性（含 packed+transparent 互斥）+ AIL8003、ail.lock integrity 哈希对象定义、`.ailmeta` 字节级确定性（数组规范化有序，闭合 RFC 0007 Hash 随机化矛盾）、§10 registry MUST 级 + §78→§24/RFC0005§3#3 权威重定向、TOFU/签名残留缺口措辞收紧、FFI 违规接入 AIL8xxx（非 6xxx）。

**pass-2 修正**（pass-1 对抗 workflow 复验报 11 条 confirmed，本批逐一修正）：① **`string` C 串封送三方矛盾**——白名单 C 串行仅列 `CStr`、`CString` 入非 FFI-safe 拒绝集（2-field owning 非 transparent 单字段），消解「CStr/CString 同为 transparent FFI-safe」与 §6.2/§7.2 的矛盾；② **裸 `Array` 跨 FFI**——§4 限定 `Array` 仅作 `layout(C) struct` 字段时 FFI-safe、裸 `extern` 参数/返回 v0.3 未规约（新增开放问题 #10）；③ **`system` ABI 定义**——改述为「按目标三元组确定的平台默认 ABI」（aarch64 Linux/macOS = AAPCS64）+ system≡aapcs 重叠说明（新增开放问题 #11 目标三元组命名空间）；④ **printf UB 退路**——移除「保留 printf 仅标注示意」退路（与 §2 MUST 目标矛盾）；⑤ **`borrow` 跨 FFI 一般规则**——新增 §6.4：`borrow` 跨 FFI 当且仅当 T 为 `layout(C) struct`，其余（含 transparent 指针型 newtype 如 `CStr`）→ `AIL8004`，闭合 transparent 双重间接 UB 一类；⑥ **packed + transparent 互斥**——§7.2 组合合法性增至 8 条（加 packed+transparent 互斥，对齐 Rust `#[repr(packed, transparent)]` 拒收）；⑦ **§10 registry 信任链**——分歧「拒绝发布或告警」收紧为 MUST 级「必须拒绝发布」、确定性义务权威分两层各承其源（顶层序 ← RFC 0005 §3 #3 / 数组规范化有序 ← 本 RFC §9.2）、**registry 自身被入侵**显式登记为残留缺口（§9.2/§10/§11 #4 三处呼应，签名为唯一长期闭合手段、推迟 #4/#5）；⑧ 自洽核查 §13 行号 3099→3098、加 `CString` 非 FFI-safe 行、组合合法性 7→8 条；⑨ §9.2 数组字段规范化义务来源与顶层字段序固定义务分离标注。待 pass-3 复验收敛 + regression。

**pass-3 修正**（pass-2 对抗 workflow 复验报 11 条 confirmed [0H/7M/4L]，本批逐一修正；F14/F15 同 §9.2 一处合并修正、故实为 10 处编辑闭合 11 条）：

- **F7（M）平台—ABI 一致性校验**——§5 增 target-ABI 校验段：平台特定 ABI（`stdcall`/`fastcall`/`vectorcall` 仅 Windows、`aapcs` 仅 ARM）用于不兼容目标 = 编译错误、复用 `AIL8002`（语义扩展为「非法或不适用于当前目标」）；`c`/`system` 在所有目标合法；校验在名字解析之后独立 pass。闭合「in-set 但目标不兼容」静默误编 UB 入口。
- **F8（M）双 align 冲突**——§7.2 规则 8 增「同 `layout(...)` 内 `align` 至多一个」（spec 身份 = 基关键字、不论 N 是否相等）→ `AIL8003`；`layout(align(16)+align(32))` / `layout(align(8)+align(8))` 均拒。
- **F9（M）align(N) 下界**——§7.2 规则 2 增「N ≥ 目标 struct 字段自然对齐之最大值」（`align` 只可提升、不可降低；欲降低须配 `packed`）+ `layout(align(1)) struct { x: int64 }` → `AIL8003`（对齐 Rust E0558）。
- **F10（M）组合布局 FFI-safe 分类**——§4 白名单「C 布局 struct」扩为 `layout(C)` / `layout(C+packed)` / `layout(C+align(N))`（裸 `packed`/`align(N)` 无 C 非 FFI-safe）；§6.4 borrow 规则与判定准则同步把组合 C 布局纳入；§13 加 §4↔§7.2 分类行。
- **F11（L）CStr UB 归因勘误**——§6.2 注释修正：按值 `layout(C) struct{byte*}` ≡ `byte*`（主流 ABI 归 INTEGER 类、单寄存器传指针值）**无双重间接**、非 UB；UB 源于 `borrow`（传 `struct{byte*}*`、C 误读指针位模式）。选 `transparent` 真实因果 = 触发 §6.4 笼统 borrow 禁令从根闭合 borrow-UB 路径。§6.4 transparent 行「双重间接」措辞加限定（指针型字段 transparent newtype 成立、笼统禁 borrow 为保守安全）。
- **F12（M）数组规范化 MUST 收窄**——§9.2 MUST 仅覆盖 set/Map 派生数组（`effects[]`/`errors[]`）；声明序数组（`input[]`/`fields[]`/`variants[]`/`sigs[]`）保留源码序（承载 positional + ABI 语义、§25.4 `layout(C)` 字段序 = C ABI）；§13 加行。
- **F13（M）确定性链结论口径收紧**——§9.2 末句改「`.ailmeta` 字节级确定性 + 字节一致的依赖闭包（非二进制整体）」+ 交叉指向 §15 #6（ailc lowering 未闭合）。§12 RFC 0005 §3#3 行同步收紧。
- **F14/F15（L/L）schema 版本归属勘误**（同 §9.2 一处合并修正）——勘误 `examples[]`/`properties[]`/`contracts` 非 §24 字段（属 RFC 0001–0003 0.3.0 rider，RFC 0005 §13）；`effects[]`/`errors[]` 归 §24 0.2.0；补 schema_version 关系说明（规范化义务跨版本适用、registry 重建 schema 随 rider 落地 0.2.0→0.3.0）。§13 加 schema 版本归属行。
- **F16（L）§12 AIL8004 补全**——FFI 违规行加第 4 类「封送违规 → `AIL8004 ffi-marshalling`」（§6.4 borrow 跨 FFI 用于非 C 布局 struct、含 transparent 指针型 newtype 如 `CStr`）；AIL8002 语义补充「目标平台不兼容」一类。
- **F17（M）§14 落地映射 §38 std.ffi 勘误**——§38 冻结为 `std.http`（§33 模块总览 15 模块无 std.ffi），`CStr`/`CString` 改落 §34 `std.core`（与 `panic`/`assert` 同模块、默认加载免 import）；RFC 全文 7 处 `std.ffi` 引用统一勘误为 `std.core`（状态行目标版本、§6 落地形态、§6.2 header、§6.2 封送注释、§13 零新关键字行、§15 #9）；line 380「§33 std.core / §34 类型表」勘误为「§34 std.core / §77 类型表」。

**pass-4 修正**（pass-3 对抗 workflow 复验报 9 条 confirmed [0H/5M/4L]，本批逐一修正；finding ID F5–F13 为 pass-4 workflow 编号、与 pass-3 F7–F17 为独立两批）：

- **F5（M）ffi-safe-whitelist-completeness**——§4 白名单「原始整数」行补 `int`（≡`int64`）/`uint`（≡`uint64`）、「原始浮点」行补 `float`（≡`float64`），按 §15.1 line 504 / §77 line 3089(`int`→i64)、3091(`float`→f64) 映射的定宽身份入封闭集（消除「白名单封闭 → `extern c { fn f(n: int) -> int }` 被 AIL8001 误拒、与冻结 §25.4 line 1413–1416 `Packet{id:int}` 自相矛盾」）；§13 加「§4 默认宽度原语 FFI-safe」自洽行（含 `Packet{id:int}` 合规）。
- **F6（M）abi-closed-set-sufficiency**——§5 平台—ABI 一致性校验收紧至架构粒度（对齐 `aapcs`=ARM-only 先例）：`stdcall`/`fastcall` **仅 x86（32 位）Windows** 合法（x86_64 Windows=win64 / aarch64 Windows=AAPCS64 / 非 Windows → AIL8002）、`vectorcall` **仅 x86 或 x86_64 Windows** 合法（aarch64 Windows / 非 Windows → AIL8002）；不兼容例增至 5 个（补 `vectorcall` on aarch64 Windows、`stdcall` on x86_64 Windows），闭合 aarch64 Windows / x86_64 Windows 上「Windows 目标但无该 ABI」的 codegen 失败空白。
- **F7（M）layout-grammar-extension**——§4 「透明包装」行改语义判定（非精确语法匹配）：任何含 `transparent` spec 的**合法** layout（`layout(transparent)` / `layout(C + transparent)` / `layout(transparent + align(N))` / `layout(C + transparent + align(N))`；`packed + transparent` 已互斥非法）且单字段 FFI-safe 者，均归 FFI-safe 透明包装、按值跨界继承字段 ABI；§13 §4↔§7.2 行同步加「transparent 组合 FFI-safe 身份由 transparent 语义判定」注，消解 `extern layout(C + transparent) struct W(int32)` 触发 AIL8001 false-positive 的悬空。
- **F8（L）layout-grammar-extension**——§7.2 规则 7 改 NOTE、移除 `→ AIL8003` 归因：文法已强制（仅 `struct` 产生式带 `layout?` 前缀，§27 line 1486），`layout` 前置于 `enum`/`type_alias`/`interface`/`trait` 为**语法错误**（RFC 0008 §5 AIL1xxx 词法/语法段），不进入 AIL8003 语义校验（原 `layout(C) enum → AIL8003` 为不可达路径、错误段归属错）。
- **F9（M）ail-lock-reproducibility-tension**——§9.2 把函数级 `errors[]` 从 set/Map 派生类移至声明序类（与类型层 `variants[]` 同源同序、保留 E 声明序、**不得**排序）：纠正「`errors[]` 由 set/Map 派生、须字母序规范化」错误前提（§24 line 1343「errors[] 单一真源 ≡ 签名 E 变体集」、§17 line 713——签名派生、Hash seed 不影响）；set/Map 派生（须预排序）类仅留 `effects[]`；§9.2 加「`errors[]` 签名派生、同源同序于 `variants[]`、字母序化会形成同源双序」说明；§12 RFC 0007 §6 行 + §13 §9.2 行 + §13 schema 版本归属行三处同步移 `errors[]` 出 set/Map 类。
- **F10（M）ailmeta-registry-authority**——§10 line 308「完整性」项对齐 §9.1 精确语义：`integrity` 哈希对象 = registry 下发的**源码归档**字节流（**非**重建 `.ailmeta`）、仅防源码归档下发篡改（不覆盖伪造 `.ailmeta`）；§9.1 末括注加条件限定——「校验源码归档即连带锁定 `.ailmeta`」**仅当消费方自源码重建时成立**、§32 消费方不重建故被入侵 registry 仍可下发伪造 `.ailmeta`（指向 §10/§11 #4 残留缺口）；与 §9.2 line 290 / §10 line 309 / §11 #4 line 325 gap-disclosure 措辞一致。
- **F11（L）ailmeta-registry-authority**——§10 flowchart line 302「顶层字段序固定 §78」改「顶层字段序固定（RFC 0005 §3 #3；§78 仅资料性）」，与 line 306 权威重定向注 + §13 自洽行一致（顶层字段序义务权威 = RFC 0005 §3 #3 Conformance，§78 为 Part IV 资料性）。
- **F12（L）cross-rfc-update-0009**——§9.2 + §13 schema 版本归属行的「RFC 0005 §13」勘误为「RFC 0005 §12 / 附录 D.2」（§13 开放问题无 schema_version；真源 §12 line 217 handoff「0.2.0→0.3.0 升级由 rider 承担」+ 附录 D.2 line 152「schema_version 当前 0.2.0、RFC 0001–0004 预留 0.3.0」）；§13 schema 行同时补 line 217/152 锚点。
- **F13（L）cross-rfc-update-0009**——§7.3 line 226 + §13 §7.3 自洽行加 contingent 标记：内征调用式随 RFC 0006 §7 A/B 终审决断 contingent（方案 B = turbofish `size_of::<T>()`、方案 A = `size_of<T>()`）——RFC 0006 §7 截至 Draft 推荐方案 B 但尚未收敛（待 review + 对抗验证终审），§7.3 不再硬编码 turbofish 为既决事实。

**pass-5 修正**（pass-4 对抗 workflow 复验报 6 条 confirmed [0H/2M/4L]，本批逐一修正；finding ID F1–F6 为 pass-5 workflow 编号）：

- **F1（M）ffi-marshalling-completeness（`borrow_mut` 跨 FFI）**——§6.4 标题与全文扩为「`borrow` / `borrow_mut` 跨 FFI 的一般规则」：`borrow_mut T`（可变借用）跨 FFI 复用与 `borrow T` 相同规则——**当且仅当 T 为 C 布局 struct（`layout(C)` / `layout(C + packed)` / `layout(C + align(N))`）**，封送为 C `T*`（允许 C 侧经指针写入调用方缓冲、变异跨 FFI 合法用例如 `read` / `gethostname` / `snprintf` 的输出参数）；借检查器在调用点保证调用方侧独占（`borrow_mut` 排他性）、变异由 `unsafe` 块授权。其余 FFI-safe 类型 `borrow_mut` 同 `borrow` 一律 → `AIL8004`。§6.4 标题 / 开头段 / 表头+表首行 / 判定准则四处同步（消除「§27 `mode := "borrow" | "borrow_mut" | "move" | "copy"` 合法但 §6.4 仅判 `borrow`、`borrow_mut` 跨 FFI 合法性悬空」缝隙）。
- **F2（M）ailmeta-byte-determinism（顶层数组声明序）**——§9.2 声明序数组清单扩为含**顶层数组** `functions[]` / `types[]` / `errors[]` / `declarations[]`（§24 line 1332 顶层 schema）——顶层条目以有序 AST 承载、必须保留源码声明序、不得按字母序或 Map 迭代序序列化（强制 ailc 以有序结构存储顶层符号表、消除 HashMap 符号表实现导致的跨构建/跨实现 `.ailmeta` 字节漂移）；§13 §9.2 自洽行同步补顶层数组清单。
- **F3/F4（L/L）ailmeta-byte-determinism（§9.2 顶层字段序归属 §78→RFC 0005 §3 #3，两 dim 同一 §9.2 line 288 首子句）**——§9.2 确定性链结论口径首子句「`.ailmeta` 顶层字段序固定（§78）」勘误为「（RFC 0005 §3 #3；§78 仅资料性）」——与 line 288 末括注「§78 仅为 Part IV 资料性…不构成本义务权威」、§10 line 302 flowchart + line 306 权威重定向注、§13 §9.2 自洽行四处一致（顶层字段序义务权威 = RFC 0005 §3 #3 Conformance、非 §78），消解首子句单独引 §78 作权威的内部矛盾。
- **F5（L）cross-rfc-update-0009（line 锚点→章节锚，去脆弱行号）**——§13 schema 版本归属行的「RFC 0005 §12 handoff / 附录 D.2」交叉引用：pass-4 F12 曾硬编码行号（§12 **line 215** / 附录 D.2 **line 150**）、pass-5 F5 勘误为（**line 217** / **line 152**）——但 pass-6 核查两轮行锚**均为悬空**（RFC 0005 line 215 = 分隔线 `---` 非「正交」、line 217 = §12 节标题非 handoff 正文、line 150/152 = 空行非 D.1/D.2；真正 schema_version handoff 句在 §12 正文、D.2 schema_version 在 D.2 正文，且 RFC 0005 尚在 pass-19 验证中行号会再漂）。故 §13 改用**章节锚**（§12「schema_version 无关」handoff / 附录 D.2 语言与类型术语）、**去除全部行号**——消除脆弱行锚、保交叉引用稳定（行号勘误→章节锚的范式修正）。
- **F6（L）ffi-grammar-frozen-compliance（§6.2 独立 impl 块超冻结合法）**——§6.2 改写为冻结合规形：`impl CString { fn as_cstr... }` → 内联进 struct 定义（`struct CString { buf, len, fn as_cstr(borrow self) -> CStr }`、§27 line 1486 struct 内联 fn、冻结合规）；`impl string { fn to_cstr... }`（`string` 为 #[lang] 内建非 struct、无法内联）改为 std.core 内建方法【签名】声明 + 显式「文法依赖声明」注——`string.to_cstr` 的定义语法依赖 §83 #2 既列 v0.3【内建类型方法附加（独立 impl 块）】特性、该特性尚无 owner RFC、本 RFC 仅规范【签名+语义】不出现字面 `impl` 块、落地前须先指定文法通道。§13 加 §6.2 文法合规自洽行（零新产生式成立 + §83 #2 依赖声明）——消解「§6.2 用独立 `impl` 块」与 §13「零新产生式」+ §27（无 impl 产生式）的矛盾。

**pass-6 报 0H / 2M / 3L = 5 条 confirmed**（default-refute workflow，refuted 2、共 raw 7），已逐条修正、待 pass-7 复验：

- **F1（L，layout-grammar-extension）**：§13 自洽核查「**8 条** AIL8003 校验规则」措辞**在 pass-4 F8 后过期**——pass-4 F8 把规则 7（layout 仅 struct）从 AIL8003 改归 AIL1xxx 语法段，故 §7.2 组合合法性表 8 行中**仅 5 条触发 AIL8003**（transparent 单字段 / align(N) 2 的幂+下界 / packed+align(N) 互斥 / packed+transparent 互斥 / 重复标识）、2 条合法（C+packed / C+transparent 冗余告警）、1 条 AIL1xxx 语法（layout 仅 struct）。修正：§13 该行改述为「8 行 = 5 条 AIL8003 + 2 合法 + 1 AIL1xxx 语法」精确分类，闭合自洽核查计数过期。
- **F2（L，ailmeta-registry-authority）** + **F5（L，cross-rfc-update-0009）**（同根——§13/§16 schema 版本归属行的 RFC 0005 行锚悬空）：pass-4 F12 硬编码「§12 line 215 / 附录 D.2 line 150」、pass-5 F5 勘误为「line 217 / line 152」——pass-6 核查**两轮行锚均悬空**（line 215 = 分隔线 `---` 非「正交」、line 217 = §12 节标题非 handoff 正文 [真正 handoff 句在 §12 正文]、line 150/152 = 空行非 D.1/D.2 [真正 D.2 schema_version 在 D.2 正文]；且 RFC 0005 尚在 pass-19 验证中、行号会再漂）。修正：§13 schema 版本归属行 + §16 pass-5 F5 记录改用**章节锚**（§12「schema_version 无关」handoff / 附录 D.2 语言与类型术语）、**去除全部行号**——消除脆弱行锚、保交叉引用稳定（同一 §9.2 line 288 末括注 + §10 line 302/306 flowchart 仍引 §78 资料性正确、未受影响）。
- **F3（M，provenance-sbom-defer-honesty）**：§11 #2 provenance「构建环境（编译器版本 / 目标三元组）」字段在 `ail publish` 时**无构建可测量**——§42 = 4 步静态扫描（无目标构建）、§31 = 源码分发，作者侧无「构建环境」可填、字段真源未定义（与 §42/§31 矛盾）。修正：§11 #2 provenance 改述为由 **registry 在重建 `.ailmeta` 时生成**（§10 registry 重建为权威）——「构建环境」字段真源 = registry 重建所用编译器版本 / 目标三元组；作者侧 `ail publish` 不生成 provenance（仅提交源码 + `ail.lock`），provenance 由 registry 重建时产出、与重建 `.ailmeta` 一同下发。与 §10 registry 重建权威 architecturally 一致。
- **F4（M，cross-rfc-update-0009）**：RFC 0007 §6.1（line 123）+ §13 OQ#6（line 307）回引本 RFC §9.2 时用笼统「`.ailmeta` 数组字段必须预先规范化排序」措辞——**先于** §9.2 的收窄（set/Map 派生 `effects[]` 预排序 + 声明序数组 `input[]`/`fields[]`/`variants[]`/`errors[]`/`sigs[]` 保留源码序），笼统措辞与本 RFC 收窄后规则不符。修正：本 RFC §12「RFC 0007 §6」交叉更新行加「⚠️ 回引同步」注——权威规则以本 RFC §9.2 为准、RFC 0007 两处回引笼统措辞待其下一轮验证（pass-20）同步收紧（本 RFC 不越界改 RFC 0007 正文、pass-19 验证进行中）。

> 本批是「与外部世界交互的信任与确定性」主题，验证重心在**FFI 安全性（白名单是否真闭合、printf UB 是否真消除）与供应链 soundness（lockfile 是否真复现、信任链是否真闭合）**，比 0006 形式化、0007 确定性、0008 可观测更偏「跨边界正确性 + 攻击面闭合」。

---

*本 RFC 由综合判断 [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 P0-4（末项）+ [`deep-review-2026-07.md`](../research/deep-review-2026-07.md) §6.1（供应链 1.5）+ §4.4（ABI/FFI 2.0）驱动。落地后将闭合「FFI 封送未规范、printf 自触 UB」「无 lockfile 构建不可复现」「.ailmeta 信任链未定义」三个被两轮评审判为最薄弱的缝隙，把 ABI/FFI 与供应链从「口号层」抬到「最小可工程闭环」，并修复规范自带的 UB 示例。**P0 五项（治理/形式化/确定性/诊断/FFI+供应链）至此齐备**——deep-review §8 明言 P0 完成前方不建议启动编译器。*
