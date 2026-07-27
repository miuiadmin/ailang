# RFC 0007 · P0-2 确定性条款批—— 求值序 / Drop 扩展 / Map 迭代+Hash / 数值转换 cast / 字符串最小语义包 / build profile 配置

| | |
|---|---|
| **状态** | 草案（Draft v1）—— 待 review（尚未跑对抗式 workflow；目标收敛 0H/0M/0L，对齐 RFC 0001 v6 / 0002 v8 / 0003 v5 / 0004 v5 / 0005 v1 / 0006 v1）|
| **目标版本** | **v0.3+**（**不触动 v0.2.1 冻结规范**：§1–§94 语义决断、56 关键字、110 决议均不变；`as` 转换算子复用既有 56 关键字之一——`as` 已在 §9 模块类别，零新关键字）|
| **日期** | 2026-07-27 |
| **分级** | **P0-2**（综合判断 [`docs/research/synthesis-2026-07.md`](../research/synthesis-2026-07.md) §6 第三优先级；synthesis §4 交叉印证 #1–#5 + #12，全部 high 严重度，两轮独立双盲命中）|
| **承接** | [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §4 交叉印证矩阵（#1 求值序 / #2 Drop / #3 数值转换 / #4 字符串 / #5 Map 迭代+Hash / #12 build profile）+ §6 P0-2；[`deep-review-2026-07.md`](../research/deep-review-2026-07.md) 可预期维度 6.25 缺口；[RFC 0005](./0005-spec-governance.md) §6（行为分类，求值序/Map 迭代当前为 unspecified）+ §7（build profile 规范语义，本 RFC 补工具链配置项）；[RFC 0006](./0006-formalization-name-resolution.md) §5（字面量词法，char_lit 已定义）|

---

## 1. 动机

synthesis §4 的**交叉印证矩阵**是全场可信度最高的必修项——下列 5 条 gap 被两套完全独立的评审框架分别命中（双盲 = 既非某一框架偏见，也非偶发），全部 high 严重度，且全部归在「可预期 / 运行结果可预期」维度：

- **#1 求值顺序未定义**——§11 只有运算符优先级，无操作数求值序；
- **#2 locals / 临时值 Drop 顺序**——§90 #5 只锁字段，locals / 临时值未定；
- **#3 数值转换算子缺失**——「禁隐式转换」却无 cast 算子（printf 示例自触 UB）；
- **#4 字符串 / Unicode 整层空白**——length 单位、char、迭代、Eq 全缺；
- **#5 Map/Set 迭代序 + Hash trait 缺**——Map 底层 HashMap 却无迭代方法、无 Hash trait。

外加 **#12 build profile 配置项**（RFC 0005 §7 留给工具链层）。

**为什么是「确定性条款批」**。这些 gap 的共同性质是**让运行结果可预期**——同一程序在不同实现 / 不同运行下产生相同结果。它们多为**纸面决断**（每条一两段），不阻塞编译器地基（P0-1 已解），但直接决定「可预期」维度能否从 6.25 提升。synthesis §3 把它们归为 **A 类未定义地基**（便宜、是编译器前置）。

**与 RFC 0005 的交叉**：**求值序**当前在 RFC 0005 §6 登记为「未指定行为」（RFC 0005 §1.5.4 就地登记），**Map/Set 迭代序则未被 RFC 0005 提及**；本 RFC 将前者**收紧为定义行为**（左到右，登记关闭），并为后者**新增一条显式未指定登记**（诚实登记，不假装有序），二者都更新 RFC 0005 §6 的状态（§10）。

---

## 2. 设计目标

1. **不触动冻结语义决断**——§1–§94 语义、56 关键字、110 决议不变。`as` 转换算子复用既有 56 关键字（§9 模块类别），零新关键字。
2. **确定性优先**——本批主题是「让运行结果可预期」，凡能锁定的求值 / Drop / 转换语义**必须**锁定为定义行为；不能免费确定的（Map 迭代序）**必须**显式登记为未指定（诚实，不假装有序）。
3. **安全代码永不 UB**（RFC 0005 §6）——数值转换**不得**引入 UB：float→int saturating（NaN→0、越界 clamp），而非 C 的截断 UB。
4. **与既有自洽**——求值序与 §11 优先级、Drop 与 §90 #5 字段序、cast 与 §15.1 类型 / §90 数值语义、字符串与 §36 / §86 #8 Eq 解糖完全对齐（§11 自洽核查表）。
5. **最小语义包**——字符串只补 length 单位 + char + 迭代 + Eq（让 `for ch in s` / `s == s` 可用），完整 Unicode（grapheme、正规化、case folding）留后续。

---

## 3. 现状与缺口诊断

| 子项 | 现状（v0.2.1） | 缺口 | 决断（本 RFC） |
|---|---|---|---|
| 求值序 | §11 仅优先级 / 结合性；`&&`/`\|\|` 短路内在规定左先，余皆未定 | 操作数求值次序未规定 | **§4 左到右逐产生式** |
| Drop 扩展 | §90 #5 锁**字段**声明序；locals / 临时值未定 | locals 逆声明序 + 临时值构造逆序 | **§5 逆序 drop** |
| Map/Set | §35 底层 HashMap（无序），无 iter 方法、无 Hash trait | 迭代方法 + Hash trait + 迭代序 | **§6 Hash trait + 未指定序登记** |
| 数值转换 | §11「转换必须显式」无算子；`as` 在 56（import 用） | cast 算子 + float→int 语义 | **§7 `as` 算子，float→int saturating** |
| 字符串 | §36 length 返回 int 单位未定、无 char、无迭代、Eq 未接地 | 最小语义包 | **§8 length=字节 + char + 迭代 + Eq** |
| build profile | RFC 0005 §7 规范语义已定，配置项留给工具链 | ail.toml [profile] schema | **§9 配置项 schema** |

---

## 4. 求值顺序 = 左到右（逐产生式）

落地形态：§11 新增「求值顺序」小节 + §16 控制流补注。

**决断**：除 `&&` / `||` 短路（右操作数条件性）外，**所有表达式的操作数按左到右求值**，逐产生式规定：

| 产生式 | 求值序 |
|---|---|
| 二元 `a op b`（`+ - * / % == != < > <= >=`） | a 先，b 后 |
| 赋值 `x = e`（含位置 `a[i] = e` / `o.f = e`） | 先求值左值位置 `x` 的子表达式（接收者 / 索引 / 字段路径，左到右，如 `a[g()]` 中 `a`→`g()`），再求右值 e，最后写回**已解析的位置**（位置子表达式不重复求值）——与复合赋值一致（位置先于右值） |
| 复合赋值 `x op= e`（`+=` `-=`） | 先求值左值位置 `x` 的子表达式（左到右）→ 读当前位置值 → 求 e → 运算 → 写回已解析位置 |
| 调用 `f(a, b, c)` | f 先，再 a → b → c |
| 成员 `a.b` | a 先 |
| 下标 `a[i]` | a 先，i 后 |
| 结构体构造 `T { f1: e1, f2: e2 }` | 各字段值表达式按**构造器书写序**（左到右，e1 → e2）求值——该序独立于 struct 类型内的字段**声明序**（构造器允许乱序写 `T { y: e2, x: e1 }`，求值序 = 书写序）。字段 **Drop 序**仍为类型声明序（§90 #5），与求值序是两件事 |
| 列表 / Map 字面量 `[a, b]` / `[k1: v1, k2: v2]` | 元素左到右 |
| `send` `h ! m` | h 先，m 后 |
| `&&` / `\|\|` | 左先；右**条件性**求值（短路） |

> **作用域**：本表覆盖**表达式产生式**（§27 `expr` 链的操作数）。控制流**语句**（`if`/`for`/`while`/`loop`/`match`/`return`/`throw`）的执行序属语句语义、见 §16（条件 / 迭代源 / scrutinee 先于体求值为常识序，非表达式求值序范畴）；`match` 的 scrutinee 作为**表达式**遵循本表（左到右），arm 选择与体执行属 §16。

**设计说明**：

- 左到右对齐 Java / C# / Go / Python / Rust 主流取向，最大化 AI 可预测性（AILang 核心卖点）。
- **位置子表达式副作用可观测**：赋值 / 复合赋值的左值位置子表达式（如 `a[g()] += f()` 中的 `g()`、`f()`）对程序副作用**完全可观测**，故「先位置、后右值」的序**必须**锁定（非「以求完备」的可有可无）。**别名论证**仅说明存储位置 `x` 与右值 `e` 不会经同一可变位置交互（borrow 不逃逸 §18.5、`borrow_mut` 独占），故无需为数据冒险额外规定，但**不**消除位置子表达式自身的副作用求值序。
- **赋值表达式产 `void`（unit）**：`x = e` 作为表达式求值为 `void`（对齐 Rust，区别于 C 的「yield 被赋值」）。故赋值链 `a = b = c` 因内层 `(b = c)` 为 `void`、与外层赋值的右值类型不匹配 → **编译错误**（语法可解析、类型检查拒绝）；文法 `assign` 的右递归虽允许链式解析，但类型规则排除之。赋值作为语句（`x = e;`）是惯用法。
- 此决断把 RFC 0005 §6「求值顺序未指定」**收紧为定义行为**（§10 交叉更新）。

---

## 5. Drop 顺序扩展（locals 逆声明序 + 临时值构造逆序）

落地形态：§18.7 扩展 + §90 #5 补注（不修改字段序决断，仅扩展到 locals / 临时值）。

**决断**：在 §90 #5「字段 Drop 顺序 = 声明序」之上，扩展：

1. **局部变量（locals）**：作用域内的局部变量按**逆声明序** drop（后声明的先释放——栈序，LIFO）。
2. **临时值**：表达式求值产生的临时值（如 `f(g(), h())` 的中间结果）按**构造逆序** drop。
3. **调用实参临时值**：实参按左到右求值（§4）后，在函数返回后**逆序** drop。
4. **混合序**：同一作用域内，locals 与临时值统一按**析构时点逆序** drop（后构造 / 后声明的先释放）；临时值的「析构时点 / 作用域」由 #6 的 lifetime 规则确定（语句级 vs 块级）。
5. **panic 展开**：同序 drop（§89 #3 已有，保持）。
6. **临时值作用域（lifetime）**：临时值在包围它的**完整表达式**（full expression——语句末 `;` 或最外层表达式边界）结束时 drop（对齐 Rust Reference / C++ temporary lifetime）。**例外**：若临时值被 `let` 初始化器 **move 进** local（直接成为该 local 的值），则**不构成临时值**（归入该 local 的逆声明序 #1）。**裸表达式语句**（§27 `stmt := … | expr`，如 `foo();` 丢弃返回值、`a + b;`）产生的非实参临时值，在**该语句末** drop（语句级作用域，不进入块级 locals 的混合序）。此规则给 #2 / #4 的「析构时点」提供确定作用域，使混合序可一致应用。
7. **构造期 panic 的字段 Drop 归类**：struct 字面量 `T { f1: e1, … }` 构造途中某字段初始化器 panic 时，**已求值的字段值**按 §5 #2 **临时值逆构造序** drop（即逆书写求值序——§4 求值序为书写序，故 Drop 序 = 逆书写序）；struct **完整构造后**字段 Drop 序才转用 §90 #5 ① 声明序。「move 进 struct 字段」与「move 进 local」（#6 例外）同理：仅在 struct 完整构造后方成为字段（§90 #5 接管），构造中途的字段值仍按临时值处理。此条款消除「书写序 ≠ 声明序」时两种读法（逆构造序 vs 声明序）产出不同可观测 Drop 序的歧义（对齐 Rust Reference：struct literal 构造期已初始化字段按逆初始化序 drop）。

**设计说明**：

- locals 逆声明序 = 栈展开的自然序（后分配的栈帧先释放），与 Rust / C++ RAII 一致。
- 字段声明序（§90 #5，自上而下）与 locals 逆声明序（LIFO）**不矛盾**——前者是 struct 内部字段的逻辑释放序，后者是栈上 locals 的物理释放序，二者作用域不同。
- 此扩展闭合「Drop 顺序」gap 的剩余部分（字段已锁，locals / 临时值补齐）。

---

## 6. Map/Set 迭代序 + Hash trait

落地形态：§35 扩展（Hash trait + 迭代方法 + 迭代序登记）。

### 6.1 Hash trait（新增 std.core trait）

```
trait Hash {
    fn hash(borrow self, state: borrow_mut Hasher) -> void      // 写入哈希状态
}
```

- **基础类型实现**：`int`/`uint`/`bool`/`float`/`byte`/`char` 直接实现 `Hash`（值经 `Hasher` 写入）；`string` 实现 `Hash`（UTF-8 字节序列）。**`float` 的 `Hash` 须与其 `Eq` 一致**（守住 `a == b ⟹ hash(a) == hash(b)` 不变量）：IEEE 754 下 `+0.0 == -0.0` 为真（§90 #2），故 `float` 的 `Hash` **必须**把 `+0.0` 与 `-0.0` 归一为同一哈希（按数值而非位模式）；`NaN ≠ NaN`（§90 #2），其相等前提为假、对 `Hash` 不施加一致性约束（实现可按位模式或固定值，因 `Eq` 已排除其相等匹配，查找 NaN 键永不命中）。
- **语义类型**：透明别名（`kind: alias`）继承 base 的 `Hash`（透明别名不改变类型身份，按 §15.3 与 base 同一）；名义 semantic（`kind: semantic`，§15.3）的 `Hash` **不自动继承**——`Hash` 是**运算 trait**（带方法 `fn hash(...)`），属 §15.3 / §92 #7 锁定的「marker vs 运算」二分中的**运算类**，与 `Eq` / `Ord` 同类，**须显式实现**（v0.3 impl 语法落地后），保持 newtype 语义安全与 `Hash`/`Eq` 对称（二者皆须显式实现，避免「`Eq` 显式而 `Hash` 隐式继承」导致二者可能不一致 → 破坏 `a == b ⟹ hash(a) == hash(b)` 不变量）。唯一自动经 base 的例外是 `Display`（§15.3 已锁定的封闭例外集，非运算 trait）。
- **Map/Set 的 `K` / `T` 约束**：`Map<K, V>` 要求 `where { Hash<K>, Eq<K> }`；`Set<T>` 要求 `where { Hash<T>, Eq<T> }`（插入 / 查找需哈希 + 相等）。
- `Hasher` 为 std.core 类型（默认 SipHash-1-3，对齐 Rust HashMap 默认；**seed 运行期每进程生成**——对齐 Rust `RandomState`，二进制可复现（seed 不烘焙进编译产物，故与 RFC 0009 可复现构建无冲突）；seed 随机化以抗 HashDoS）。

### 6.2 迭代方法

| 类型 | 迭代方法 | 产出 |
|---|---|---|
| `List<T>` | `iter()` / `for x in list` | `T` |
| `Map<K, V>` | `iter()` / `for (k, v) in map` | `(K, V)` 元组 |
| `Set<T>` | `iter()` / `for x in set` | `T` |

- for-in 经 `Iterator` trait（std.core lang item，§18.8 已暗示）。
- Map 迭代产出 `(K, V)` 元组；解构 `for (k, v) in map`。

### 6.3 迭代序决断（核心）

**Map/Set 默认迭代序 = 未指定**（unspecified behavior，进 RFC 0005 §6 分类）：

- 底层 HashMap / HashSet 的迭代序由哈希分布 + seed 决定，**实现间、运行间均可能不同**；
- 规范**显式声明**此为未指定（不假装有序），需确定序的场景**必须**显式排序：

```ail
// 示意：物化 + 排序 API 随 std.collections 落地
let pairs = map.iter().collect()         // List<(K, V)>，序未指定（collect 随 std.collections）
let sorted = sort(pairs)                 // 排序 API 随 std.collections（冻结规范中 fn sort<T> 仅作 §15.10 泛型语法示例，非已声明 std 函数）
```

**trade-off**：强制 Map 有序（如改 LinkedHashMap / BTreeMap 底层）牺牲性能（违背「极致性能」目标）；默认未指定 + 显式排序逃生通道（物化 `iter()` 为 `List` 后排序；排序 API、元组 `Ord` 派生、按-key 排序便利方法、`collect` / `to_list` 物化方法均留 `std.collections`），兼顾性能与可控性。**有序变体**（`SortedMap<K,V>`，BTree 底层，迭代=K 升序）作为可选标准库类型，留开放问题 #3（是否引入）。

> 此决断为 RFC 0005 §6 **未提及**的 Map/Set 迭代序**新增一条未指定行为登记**（诚实登记，非假装有序），与本 RFC §10 表一致；求值序的既有未指定登记（RFC 0005 §1.5.4 就地登记）则由 §4 收紧关闭。

---

## 7. 数值转换 cast 算子（`as`，float→int saturating）

落地形态：§11 新增 `as` 转换表达式 + §15.1 补转换语义。

**决断**：复用既有 56 关键字 `as`（§9 模块类别，原用于 `import x as y`）作**类型转换表达式算子**：

```
cast_expr := unary ("as" type)*         // x as int / x as float32 as f64（链式）；操作数为 unary（非 postfix），使 as 位于 unary 之上、mul 之下
```

- `as` 在**表达式位置**为转换算子，在 **import 语句位置**为别名（parser 按位置消歧，零冲突）。
- 优先级：**高于乘除算术**（`*` / `/` / `%`）、**低于一元前缀**（`*` 解引用 / `!` / `try`）——产生式 `cast_expr := unary ("as" type)*` 把操作数定为 `unary`（§27 文法链 `mul := unary (...)` / `unary := prefix_op unary | postfix`），故 `as` 嵌于 `unary` 与 `mul` 之间、绑定**松于**一元前缀（一元先作用、cast 后作用）、**紧于**乘除。对齐 Rust `as`（`x as f32 * 2.0` = `(x as f32) * 2.0`、`*p as int` = `(*p) as int`）。落地时在 §27 文法链插入 `mul := cast_expr (...)` / `cast_expr := unary ("as" type)* | unary`。

**转换语义**（安全代码永不 UB，RFC 0005 §6）：

| 转换 | 语义 |
|---|---|
| 整数扩宽（`int8` → `int16` → `int32` → `int64`） | 值不变，零成本（符号扩展） |
| 整数收窄（`int64` → `int8`） | **截断低位**（保留低 N 位）——定义行为，非 UB |
| 有符号 ↔ 无符号（同宽度） | 位模式重解释（`int as uint` = 同 bit pattern 重读） |
| 符号 + 宽度同时变化（如 `int8 as uint16`、`uint32 as int8`） | **先按源符号扩展/截断到目标宽度，再按目标符号位重解释**（定义行为）。例：`int8` 值 `-1`（位 `0xFF`）`as uint16` → 符号扩展为 `0xFFFF` → 重解释为 `65535`（对齐 Rust `as`）；`uint8` 值 `255` `as int16` → 零扩展为 `0x00FF` → `255` |
| `float` → 整数 | **saturating**：NaN → 0；`x < INT_MIN` → `INT_MIN`；`x > INT_MAX` → `INT_MAX`；否则**向零截断**（trunc）。对齐 Rust `as` + WebAssembly `trunc_sat`，**永不 UB** |
| 整数 → `float` | 精确（小整数）或 IEEE 754 round-to-nearest（大整数失精度，定义行为） |
| `float32` ↔ `float64` | 精确（f32→f64）或 round-to-nearest（f64→f32） |
| `bool` ↔ 数值 | **禁止**（编译错误）——避免 C 的隐式 bool 真值陷阱。`if` 为语句（§27）、不产出值，故 `(if b 1 else 0)` **非合法 AILang**；需数值化时用 std.core 方法 `b.to_int()`（`true→1` / `false→0`），或在函数体内用 `match` |
| `char` ↔ 整数（任意宽度/符号） | `char as <整数类型>` = Unicode 标量值（零扩展到目标宽度；目标过窄则按整数收窄规则截断，定义行为）；`<整数类型> as char` 取值须为合法标量值（U+0000–U+10FFFF，非 surrogate），否则 panic `InvalidCodePoint`（签名类型负值必失败） |

> **合法转换集为封闭集合**：`as` **仅**支持上表所列转换（基础数值类型之间、与 `char` / `bool` 之间）；所有其他 `as` 转换（含 `char` ↔ `float`、enum 判别值、`raw_pointer<T>` ↔ `raw_pointer<U>` / ↔ 整数、struct / `Optional` ↔ 数值等）均为**编译错误**。`raw_pointer` 相关 cast 属 unsafe 域（§25.1 / §90 #4），不入本表——闭合「安全代码永不 UB」的 soundness 论证（表外无未定义转换）。

**设计说明**：

- 「禁隐式转换、转换必须显式」（§11 现状）现在有了**显式算子** `as`——闭合 synthesis #3「禁隐式却无 cast」缺口，修正「printf 示例自触 UB」（深度评审 §可预期）。
- float→int saturating 是**安全代码永不 UB** 立场（RFC 0005 §6）的必然——C 的截断 UB 在 AILang 不可接受。
- `as` 是**有损 / 可失败**转换（收窄截断、float→int clamp）——区别于透明别名（零成本同型）与 constraint 构造 `T(expr)`（§15.4，语义不变零成本转型）。

---

## 8. 字符串最小语义包（length 单位 + char + 迭代 + Eq）

落地形态：§36 扩展 + 引入 `char` 类型（std.core，非关键字）。

**决断**：

> **`string` 维持有效 UTF-8 不变量**（构造即校验）：`string` 值恒为合法 UTF-8 序列——构造期（字面量 / `string` 构造 / `byte`→`string` 转换）校验，非法字节输入为构造期错误或 panic `InvalidUtf8`。故下文 `iter` / `char_count` / `Eq` 等 UTF-8 解码型方法**正常路径永不遇无效字节**；§8.3 / 开放问题 #4 的 `InvalidUtf8` panic 退化为该不变量的**防御性违约 panic**（非正常运行路径）。

1. **`length()` = UTF-8 字节长度**（O(1)，直接取缓冲长度）——对齐 Rust `String::len()` / Go `len(string)`，性能优先。
   - 另提供 `char_count()` = Unicode 码点数（O(n) 遍历解码），供「字符数」语义需求。
2. **引入 `char` 类型**（标准库类型，非关键字，类 `byte`）：Unicode 标量值（U+0000–U+10FFFF，4 字节 / `uint32` 存储，对齐 Rust `char`）；字面量 `'a'`（RFC 0006 §5 `char_lit` 已定义）。`char` 为**位复制 Copy**（同 `byte` / `int` / `uint` / `float` / `bool`，§18.4 三分类追加——定宽 4 字节、无所有权语义、按值拷贝）。
3. **迭代**：`for ch in s` / `string.iter() -> Iterator<char>` 按 UTF-8 解码逐码点产出 `char`；因 `string` 维持有效 UTF-8 不变量（构造即校验，见上方引言），正常路径**永不遇无效字节**；一旦该不变量被违约（仅理论上 / 未来 unsafe 字节重解释路径），迭代立即 **panic `InvalidUtf8`**（防御性违约 panic，**不做** U+FFFD 替换、**不跳过**无效字节——与开篇引言、开放问题 #4 收敛态一致）。
4. **Eq 接地**：`string` 实现 `Eq`（`string == string` 经 `Eq.eq`，§86 #8 运算符解糖）——比较为 **UTF-8 字节序**（等价于码点序，因 UTF-8 编码保序）。

**扩展后的 §36 方法表**（在现表基础上增补）：

| 方法 | 签名 | 新增 |
|---|---|---|
| `length` | `(borrow self) -> int`（字节长度，O(1)） | 语义明确 |
| `char_count` | `(borrow self) -> int`（码点数，O(n)） | ✅ 新增 |
| `iter` | `(borrow self) -> Iterator<char>` | ✅ 新增 |
| `contains` / `starts_with` / `upper` / `lower` / `is_empty` / `append` | （现状不变） | — |

**设计说明**：

- length=字节（O(1)）+ char_count=码点（O(n)）分离，让调用方按需选择（性能 vs 语义），避免「length 到底是字节还是字符」的歧义（synthesis #4 的核心困惑）。
- `char` 为 Unicode 标量值（非字素 grapheme）——字素簇（`\r\n`、组合字符）留后续；最小包只到码点级。
- Eq 字节序 = 码点序（UTF-8 保序性定理），故 `==` 语义无歧义。

---

## 9. build profile 配置项（ail.toml，RFC 0005 §7 的工具链层延续）

落地形态：§30.1 `ail.toml` 补 `[profile]` 表。

RFC 0005 §7 已定义 profile 的**规范语义**（debug 检测全开 / release 回绕）；本节补**配置项 schema**：

```toml
[profile.debug]
opt-level = 0              # 优化级别（0=无优化，开发期）
overflow-checks = true     # 整数溢出 → panic（§90 #1 debug）

[profile.release]
opt-level = 3              # 优化级别（3=最高）
overflow-checks = false    # 整数算术二补数回绕（§90 #1 release 规范语义）
```

- 两配置项：`opt-level`（0–3，优化级别）、`overflow-checks`（bool，**整数溢出**检测）。
- **契约 / 约束 / `assert` / 边界检查不进 profile**：均为**无条件运行期检查**，不受 profile 调节——约束违约 panic `ConstraintViolation`（§92 #2 / §15.4）、`assert` / `unwrap` panic（§89 #3）、越界 panic `IndexOutOfBounds`（§17）、整数除零 panic `DivideByZero`（§90 #3）。§17 panic 表中**仅整数溢出**标注 `(debug)`（§90 #1，唯一 profile-gated 检查），其余均无 profile 限定。故**本表无 `debug-assertions` 配置项**（其无可控对象）；profile 仅调节 `opt-level` 与整数溢出检测——此与 RFC 0005 §7 release 行对齐（契约断言 **MUST** 启用、不可关闭）。
- debug 默认值固定（检测全开）；release 的 `overflow-checks` 固定 false（切到 §90 #1 回绕规范语义）。
- **安全性不变量**（RFC 0005 §7）：无论配置，安全代码不产生 UB；`overflow-checks=false` 仅切到 §90 #1 锁定的 release 回绕规范语义，非 UB。

---

## 10. 与 RFC 0005 的交叉更新

本 RFC 落地后，RFC 0005 §6 的行为分类状态变化（合并时同步更新 RFC 0005 / AILANG.md §1.5.4 / 附录 B）：

| 条目 | RFC 0005 现状 | 本 RFC 后 |
|---|---|---|
| 表达式求值顺序 | unspecified（RFC 0005 §1.5.4 就地登记） | **defined behavior**（左到右，§4）——登记关闭 |
| Map/Set 迭代序 | 未提及 | **显式 unspecified**（§6.3，诚实登记）——新增登记 |
| float→int 转换 | 未提及（禁隐式转换、无 cast 算子） | **defined behavior**（saturating，§7） |

> 即：本 RFC 把 1 个 unspecified 收紧为 defined（求值序），把 1 个未提及显式登记为 unspecified（Map 迭代），新增 1 个 defined 操作（float→int saturating cast，从定义起即不引入 UB）——净提升「可预期」维度。（printf 式 FFI UB 风险由 RFC 0009 封送修正、非本 RFC saturating 语义关闭——FFI 调用本身仍在 unsafe 域。）

---

## 11. 不变量与自洽核查

| 本 RFC 条款 | 对齐的既有规范 | 自洽性 |
|---|---|---|
| §4 左到右 + `&&`/`\|\|` 短路 | §11 运算符表（短路已注） | ✅ |
| §4 赋值左值位置子表达式先求值 + 产 `void` | 与复合赋值一致（位置先于右值）；链 `a=b=c` 类型拒绝 | ✅ 自洽（不重复求值位置） |
| §4 结构体构造求值序 = 构造器书写序（独立于字段声明序） | §90 #5 字段 Drop 声明序（求值序 ≠ Drop 序，作用域不同） | ✅ 一致（不矛盾） |
| §5 locals 逆声明序 | §90 #5 字段声明序（作用域不同，不矛盾） | ✅ |
| §5 panic 同序 drop | §89 #3 panic 展开 | ✅ |
| §6.1 名义 semantic 的 Hash 不自动继承 | §15.3 / §92 #7 marker-vs-operational 二分（Hash=运算 trait，同 Eq） | ✅ 对齐（消除与 §15.3 矛盾 + Hash/Eq 对称） |
| §6 Hash<K> + Eq<K> | §35 contains（已用 Eq<T>）/ §86 #8 Eq 解糖 | ✅ |
| §6 Map 迭代产出 (K,V) | §15.9 元组 / §27 map_lit | ✅ |
| §7 `as` 复用关键字 | §9 模块类别含 `as`（56 之一） | ✅ 零新关键字 |
| §7 float→int saturating | §90 #1 永不 UB + RFC 0005 §6 | ✅ |
| §7 `as` 位置消歧 | §12 import `as`（语句位）/ 表达式位 | ✅ 无冲突 |
| §8 length 字节 | §36 length 返回 int + RFC 0006 §5 char_lit | ✅ |
| §8 char 位复制 Copy | §18.4 三分类（追加 char，同 byte） | ✅ |
| §8 string Eq 字节序 | §86 #8 == 解糖 Eq.eq | ✅ |
| §9 overflow-checks | §90 #1 + RFC 0005 §7 | ✅ |
| §9 契约/约束/assert/边界检查不进 profile（无 debug-assertions 项） | §92 #2 / §15.4 / §89 #3 / §17（ConstraintViolation·IndexOutOfBounds·DivideByZero 均无条件） | ✅ 与 RFC 0005 §7 一致 |
| 零新关键字 | §9（56）；`as` 已含、`char` 为类型词非关键字 | ✅ 已核查 |

---

## 12. 落地映射（合并进 AILANG.md 的位置）

| 本 RFC 内容 | 落地位置 | 性质 |
|---|---|---|
| §4 求值序 | §11 新增「求值顺序」小节 + §16 补注 | 规范性 |
| §5 Drop 扩展 | §18.7 扩展 + §90 #5 补注 | 规范性 |
| §6 Hash trait + 迭代 + 序 | §35 扩展 + §34 std.core 加 Hash trait | 规范性 |
| §7 `as` 转换算子 | §11 表 + §15.1 转换语义 + **§17 panic 目录**（登记 `InvalidCodePoint`） | 规范性 |
| §8 字符串最小包 + char | §36 扩展 + §15.1 加 char 类型 + **§18.4 位复制 Copy 枚举追加 `char`**（同 `int`/`uint`/`float`/`bool`/`byte`；§74.1 Copy 摘要同步）+ **§17 panic 目录**（登记 `InvalidUtf8`） | 规范性 |
| §9 build profile 配置 | §30.1 ail.toml [profile] | 规范性（工具链） |
| §10 交叉更新 | RFC 0005 §6 + 附录 B #7 关闭 | 同步 |

---

## 13. 开放问题

1. **`as` 优先级定位**——已定为**高于乘除算术、低于一元前缀**，产生式 `cast_expr := unary ("as" type)*`（操作数 `unary`，嵌于 §27 `mul`/`unary` 之间，§7）；精确表内级号待 §11 运算符表落地时编入。
2. **复合赋值别名语义**——§4 `x += e` 的读-求-写序在安全代码中无歧义（borrow 不逃逸），但若未来引入 `unsafe` 别名，需复查。
3. **有序 Map 变体**——§6.3 默认未指定；是否引入 `SortedMap<K,V>`（BTree，迭代=K 升序，需 Ord）作为标准库类型，留 review（性能 vs 确定性 trade-off）。
4. **无效 UTF-8 迭代行为**——§8 声明 `string` 维持有效 UTF-8 不变量（构造即校验），故 `for ch in s` / `char_count()` 正常路径永不遇无效字节；`InvalidUtf8` panic 仅作该不变量的**防御性违约 panic**（已收敛为 panic，非两选一选项）。若未来引入非校验构造路径（如 unsafe 字节缓冲直接重解释），再议其解码失败语义。
5. **char 与整数互转**——§7 `uint as char` 非法标量 panic；是否提供 `char::from_uint_safe() -> Optional<char>` 非 panic 路径，留 std API 设计。
6. **Hash seed：运行期确定性 vs HashDoS 抗性**——§6.1 seed 运行期每进程生成（对齐 Rust `RandomState`，二进制可复现、与构建可复现性**无冲突**）；开放的是**运行期确定性** trade-off：是否为测试 / 回放提供 fixed-seed 运行期模式（同程序跨运行同序）。归属 `std.collections` / 后续 RFC（不再指向已收尾的 P0-4——后者未承接 Hasher seed）。

---

## 14. 收敛轨迹

**Draft v1（本文）**——尚未跑对抗式 workflow。计划按 RFC 0001–0006 既有流程：多维度 workflow 审查（建议维度：求值序逐产生式完备性 / Drop 序与 §90 #5 一致性 / Hash trait 与 Eq 协调 / `as` 语义安全无 UB / 字符串 length·char·Eq 自洽 / build profile 与 §90 对齐 / 与 RFC 0005 交叉更新正确性 / 性能 trade-off 论证）+ 对抗式 verify，目标收敛 **0H / 0M / 0L**。

> 本批是「确定性」主题，验证重心在**语义决断的安全性与完备性**（尤其 float→int 永不 UB、求值序无遗漏 case），比 0006 的形式化更偏语义正确性。

---

*本 RFC 由综合判断 [`synthesis-2026-07.md`](../research/synthesis-2026-07.md) §4 交叉印证矩阵（5 条 high 双盲命中）+ §6 P0-2 驱动。落地后将闭合「可预期」维度的核心缺口（求值序 / Drop / 转换 / 字符串 / Map 迭代），把 deep-review 可预期 6.25 分向「确定性达成」推进，并修正 printf 式 FFI UB 风险。*
