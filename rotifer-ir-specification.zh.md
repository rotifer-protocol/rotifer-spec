# Rotifer IR Specification

# 轮虫中间表示规范：跨绑定基因可移植性标准

**作者：** Rotifer Foundation
**状态：** Draft (Pre-1.0) — 本规范处于草案阶段，接口可能发生 Breaking Change
**当前版本：** 0.2.0
**前置文档：** [Rotifer Protocol Specification](./rotifer-protocol-specification.md)

> ⚠️ **稳定性警告：** 跨绑定互操作协议（§10）已在 v0.6.5 版本中通过两个绑定实现（LocalBinding + Web3MockBinding）的部分验证。宿主函数分类（§6）和能力协商协议（§10.5）可能因非 Mock 绑定的实际集成反馈而变更。IR 1.0 将在至少两个*生产级*绑定环境通过跨绑定互操作测试，且完成一轮外部开发者 RFC 反馈后正式发布。

---

## 目录 | Table of Contents

- [摘要 (Abstract)](#abstract--摘要)
- [1. 引言 (Introduction)](#1-introduction--引言)
- [2. 设计目标 (Design Goals)](#2-design-goals--设计目标)
- [3. 技术决策 (Technical Decision)](#3-technical-decision--技术决策wasm--rotifer-约束层)
- [4. 类型系统 (Type System)](#4-type-system--类型系统)
- [5. 指令集约束 (Instruction Set Constraints)](#5-instruction-set-constraints--指令集约束)
- [6. 宿主函数 (Host Functions)](#6-host-functions--宿主函数)
- [7. 安全模型 (Security Model)](#7-security-model--安全模型)
- [8. 序列化格式 (Serialization Format)](#8-serialization-format--序列化格式)
- [9. 编译管线 (Compilation Pipeline)](#9-compilation-pipeline--编译管线)
- [10. 跨绑定互操作协议 (Cross-Binding Interop)](#10-cross-binding-interop--跨绑定互操作协议)
  - [10.5 能力协商协议 (Capability Negotiation Protocol)](#105-能力协商协议-capability-negotiation-protocol)
- [11. 版本管理 (Versioning)](#11-versioning--版本管理)
- [术语表 (Glossary)](#glossary--术语表)
- [参考文献 (References)](#references--参考文献)

---

## Abstract | 摘要

Rotifer IR（轮虫中间表示）是轮虫协议的跨绑定基因可移植性层。它使得在一个绑定环境中（如 Web3）诞生的基因，能够被编译后在完全不同的环境中（如 Cloud、Edge、TEE）执行——而无需人工重写。

Rotifer IR 之于基因，如同 LLVM IR 之于程序、JVM bytecode 之于 Java 类：一个抽象的、可验证的、可跨平台编译的逻辑表示层。但与通用编译器 IR 不同，Rotifer IR 是 **基因感知 (Gene-Aware)** 的——它的类型系统原生理解基因的输入/输出模式、资源约束和安全边界。

本文档定义了 Rotifer IR 的类型系统、指令集约束、宿主函数、安全模型、序列化格式、编译管线与跨绑定互操作协议。

---

## 1. Introduction | 引言

### 1.1 动机

轮虫协议的核心愿景之一是构建 **跨环境的基因流通网络**。一个优秀的基因不应被困在它诞生的环境中——它应当能够在任何实现了轮虫绑定的环境中运行。

然而，不同绑定环境之间存在巨大的执行模型差异：

| 绑定环境 | 执行模型 | 原生语言 | 约束模型 |
|---------|---------|---------|---------|
| Web3 (EVM) | 栈式虚拟机，确定性，Gas 计量 | Solidity / Vyper | Gas 上限 |
| Cloud (K8s) | 容器化进程，非确定性，无内在计量 | Python / Go / Rust | CPU/Mem 配额 |
| Edge (IoT) | 裸机 / RTOS，极度资源受限 | C / Rust / MicroPython | 硬件资源上限 |
| TEE | 安全飞地，受限系统调用 | C / Rust | 飞地内存上限 |

直接在这些环境之间传递源代码是不可行的——EVM 字节码无法在 ARM 裸机上运行，Python 脚本无法在 EVM 上执行。需要一个 **中间层** 来桥接这些差异。

### 1.2 Rotifer IR 的定位

```
[高级源代码]                    [高级源代码]
  Solidity / Python / Rust         Go / C / MicroPython
        │                                │
        ▼                                ▼
  ┌─────────────┐                 ┌─────────────┐
  │  IR Frontend │                 │  IR Frontend │
  │  (per-lang)  │                 │  (per-lang)  │
  └──────┬──────┘                 └──────┬──────┘
         │                                │
         ▼                                ▼
  ┌─────────────────────────────────────────────┐
  │            Rotifer IR (本文档定义)             │
  │  ┌─────────────────────────────────────────┐ │
  │  │  WASM Core  +  Rotifer Constraint Layer │ │
  │  └─────────────────────────────────────────┘ │
  └──────────────────┬──────────────────────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │IR Backend│ │IR Backend│ │IR Backend│
   │  (EVM)   │ │  (WASM)  │ │ (Native) │
   └──────────┘ └──────────┘ └──────────┘
         │           │           │
         ▼           ▼           ▼
      EVM字节码   WASM模块    ARM/x86二进制
```

### 1.3 与 LLVM IR 的异同

| 维度 | LLVM IR | Rotifer IR |
|------|---------|-----------|
| 目标 | 通用程序编译优化 | 基因跨绑定可移植性 |
| 安全模型 | 无内在安全约束 | 安全性由设计保证（无直接内存、无系统调用） |
| 类型系统 | 低级（整数、浮点、指针） | 高级 + 基因感知（Schema、Context、Result） |
| 资源计量 | 无 | 内置确定性资源计量 |
| 序列化 | bitcode（紧凑但不可读） | 基于 WASM 二进制格式 + 自定义段 |
| 验证 | 可选（opt-in） | 强制（必须通过验证器才能执行） |

---

## 2. Design Goals | 设计目标

Rotifer IR 的设计必须满足以下五个目标，按优先级排序：

### 2.1 安全性 (Security) — 最高优先级

IR 的安全性不依赖于运行时检查，而是在设计层面排除不安全操作。

**安全性保证：**
- 无直接内存地址操作（无指针算术、无任意内存读写）
- 无系统调用（所有外部交互必须通过宿主函数）
- 无无界循环（所有循环必须有确定性的资源上限）
- 无浮点歧义（所有浮点运算使用 IEEE 754 strict 模式，或替换为定点数）

### 2.2 可验证性 (Verifiability)

IR 必须支持在编译后、执行前的 **静态验证**——安全检查可以在 IR 层完成，无需了解目标环境。

**可验证属性：**
- 类型正确性（所有操作的类型在编译时完全确定）
- 资源上界（可在静态分析阶段计算最大资源消耗）
- 终止性（保证基因执行在有限步内完成）
- 宿主函数合规性（仅调用白名单内的宿主函数）

### 2.3 可移植性 (Portability)

一次编译为 IR，可在所有支持 IR 的绑定环境中执行。

**可移植性约束：**
- IR 不依赖任何特定的操作系统、CPU 架构或虚拟机
- IR 的语义在所有绑定中保持一致（确定性执行）
- IR 的序列化格式是平台无关的

### 2.4 紧凑性 (Compactness)

IR 的序列化体积应尽可能小，适合 P2P 网络传播。

**紧凑性目标：**
- 简单基因（如参数编码器）的 IR 体积 < 4 KB
- 复杂基因（如交易构建器）的 IR 体积 < 64 KB
- 支持增量传输（仅传输与已知版本的差异）

### 2.5 确定性 (Determinism)

相同的 IR + 相同的输入在所有绑定中必须产生相同的输出。

**确定性边界：**
- 纯计算操作：严格确定性
- 宿主函数调用：由宿主函数的语义合约定义（可能非确定性，如网络请求）
- 时间相关操作：禁止直接访问系统时钟；时间信息仅通过 Context 注入

---

## 3. Technical Decision | 技术决策：WASM + Rotifer 约束层

### 3.1 选择依据

在核心规范评估的候选路径中，本规范选择 **混合方案：WASM + Rotifer 约束层**。

**选择 WASM 作为基础层的理由：**

1. **成熟的语言生态：** Rust、C/C++、Go、AssemblyScript、Python (via Pyodide) 等 40+ 种语言可编译为 WASM
2. **成熟的沙箱运行时：** Wasmer、Wasmtime、wasm3、WAMR 等提供即用的安全沙箱
3. **广泛的平台支持：** 浏览器、服务器、嵌入式设备均有 WASM 运行时
4. **已有计量方案：** WASM 的指令级 Gas 计量已有成熟实现（如 Wasmer metering middleware）
5. **行业趋势：** Cosmos (CosmWasm)、NEAR、Polkadot、Fastly、Cloudflare Workers 均选择 WASM

**为什么不是纯 WASM：**

标准 WASM 缺乏以下轮虫协议所需的能力：
- **基因感知类型系统：** WASM 只有 `i32/i64/f32/f64`，不理解 `GeneInput/GeneOutput/Context`
- **安全约束声明：** WASM 没有标准方式声明"此模块不访问文件系统"
- **资源上界证明：** WASM 不保证终止性
- **元数据关联：** WASM 没有标准方式嵌入基因的 Phenotype 信息

### 3.2 混合架构

Rotifer IR = 标准 WASM 模块 + Rotifer 自定义段 (Custom Sections)

```
┌─────────────────────────────────────────────┐
│              Rotifer IR Module               │
│                                             │
│  ┌─────────────────────────────────────────┐│
│  │           WASM Core Sections            ││
│  │  ┌──────────┐  ┌──────────┐            ││
│  │  │  Type    │  │  Code    │  ...       ││
│  │  │  Section │  │  Section │            ││
│  │  └──────────┘  └──────────┘            ││
│  └─────────────────────────────────────────┘│
│                                             │
│  ┌─────────────────────────────────────────┐│
│  │     Rotifer Custom Sections             ││
│  │  ┌──────────────┐  ┌─────────────────┐ ││
│  │  │  rotifer.    │  │  rotifer.       │ ││
│  │  │  phenotype   │  │  constraints    │ ││
│  │  └──────────────┘  └─────────────────┘ ││
│  │  ┌──────────────┐  ┌─────────────────┐ ││
│  │  │  rotifer.    │  │  rotifer.       │ ││
│  │  │  metering    │  │  dependencies   │ ││
│  │  └──────────────┘  └─────────────────┘ ││
│  └─────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

**优势：**
- 任何标准 WASM 工具链都可以处理 Rotifer IR（自定义段被忽略但不阻碍解析）
- Rotifer 特有的验证和分析工具只需读取自定义段
- 未来自定义段的格式可以独立演化，不影响 WASM 核心

---

## 4. Type System | 类型系统

### 4.1 基础类型

Rotifer IR 的类型系统建立在 WASM 值类型之上，但增加了基因感知的高级类型：

**WASM 底层类型（直接使用）：**

| WASM 类型 | 用途 |
|----------|------|
| `i32` | 整数、布尔值、枚举 |
| `i64` | 长整数、时间戳 |
| `f64` | 浮点数（strict IEEE 754 模式） |

**Rotifer 高级类型（在自定义段中声明，映射到 WASM 线性内存布局）：**

```
type GeneInput {
    schema: Schema          // 输入数据的类型定义（JSON Schema 子集）
    data: Bytes             // 序列化后的输入数据
}

type GeneOutput {
    schema: Schema          // 输出数据的类型定义
    data: Bytes             // 序列化后的输出数据
    metadata: OutputMeta    // 执行元数据（耗时、资源消耗等）
}

type Context {
    agentId: Hash           // 执行者 Agent 的身份标识
    timestamp: u64          // 由宿主注入的逻辑时间戳
    environment: EnvMap     // 绑定环境提供的键值对
    resourceBudget: u64     // 剩余资源预算
}

type Result {
    success: Boolean
    output: GeneOutput?
    error: ErrorInfo?
    resourceUsed: u64       // 实际消耗的资源单位
}
```

### 4.2 Schema 系统

基因的输入/输出 Schema 使用 **JSON Schema 的确定性子集**——去除了以下非确定性特性：

| JSON Schema 特性 | Rotifer Schema 中的处理 |
|-----------------|----------------------|
| `$ref` (外部引用) | 禁止——所有类型必须内联定义 |
| `additionalProperties: true` | 禁止——所有属性必须显式声明 |
| `oneOf` / `anyOf` | 允许，但要求穷举所有可能的匹配路径 |
| `pattern` (正则表达式) | 限制为 RE2 安全子集（保证线性时间匹配） |
| 默认值 | 允许——确保缺省情况下的确定性行为 |

### 4.3 类型映射

高级类型与 WASM 线性内存之间的映射规则：

```
GeneInput / GeneOutput / Context → 序列化为 MessagePack 字节流
                                    → 写入 WASM 线性内存
                                    → 通过 (offset: i32, length: i32) 对传递

String   → UTF-8 编码字节序列 + 长度前缀
Boolean  → i32 (0 = false, 1 = true)
Hash     → 32 字节固定长度数组
Schema   → MessagePack 序列化的 JSON Schema 子集
```

---

## 5. Instruction Set Constraints | 指令集约束

### 5.1 WASM 指令白名单

Rotifer IR 采用 **白名单策略**——只有显式列入白名单的 WASM 指令才允许在 IR 中出现。

**允许的指令类别：**

| 类别 | 示例指令 | 说明 |
|------|---------|------|
| 整数运算 | `i32.add`, `i64.mul`, `i32.eq` | 所有确定性整数运算 |
| 浮点运算 | `f64.add`, `f64.mul`, `f64.sqrt` | 仅 strict IEEE 754 模式 |
| 控制流 | `block`, `loop`, `if`, `br`, `br_if`, `return` | 所有结构化控制流 |
| 局部变量 | `local.get`, `local.set`, `local.tee` | 栈上变量操作 |
| 线性内存 | `i32.load`, `i32.store`, `memory.size`, `memory.grow` | 受约束的内存操作（见 §5.2） |
| 函数调用 | `call`, `call_indirect` | 仅限模块内部 + 白名单宿主函数 |

**禁止的操作（不在白名单中）：**

| 禁止项 | 理由 |
|--------|------|
| WASM SIMD 指令 | 跨平台行为差异 |
| WASM 线程与原子操作 | 非确定性 |
| WASM 异常处理提案 | 规范尚未稳定 |
| 任何宿主侧 FFI（非白名单宿主函数） | 安全边界突破 |

### 5.2 内存约束

Rotifer IR 的线性内存访问受以下约束：

```
constraints MemoryConstraints {
    maxInitialPages: u32     // 初始内存页数上限（每页 64 KB）
    maxGrowPages: u32        // 最大可增长页数
    totalMemoryLimit: u32    // 绝对内存上限（字节）
    noOutOfBoundsAccess: true // 越界访问触发陷阱（trap），不产生未定义行为
}
```

**默认约束值（可由绑定覆盖）：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `maxInitialPages` | 16 (1 MB) | 简单基因足够 |
| `maxGrowPages` | 64 (4 MB) | 复杂基因的上限 |
| `totalMemoryLimit` | 5,242,880 (5 MB) | 包含初始 + 增长 |

### 5.3 循环约束与终止性

所有 `loop` 指令必须满足以下至少一个终止性条件：

1. **静态有界：** 循环迭代次数在编译时可确定（如 `for i in 0..N` 其中 N 为常量）
2. **燃料有界：** 循环体内包含资源计量检查点（由 `rotifer.metering` 段配置），超出预算时强制中断
3. **收敛有界：** 验证器可证明循环条件在有限步内收敛（通过抽象解释或符号执行）

不满足上述任何条件的 IR 将被验证器拒绝。

### 5.4 跨绑定资源计量语义

不同绑定环境可能使用不同的资源计量单位。IR 规范定义以下互操作规则：

**计量单位声明：**

每个绑定声明其原生计量单位：

| 计量单位 | 绑定示例 | 语义 |
|---------|---------|------|
| Fuel | Cloud, Local, Edge | 指令级确定性计量（wasmtime fuel） |
| Gas | Web3 (EVM 兼容) | 交易成本计量；`Gas = Fuel × gas_price` |
| WallClock | TEE | 实际执行时长（毫秒） |

**跨绑定比较规则：**

在不同计量单位的绑定之间进行能力协商（§10.5）时：

1. 每个绑定以**标准化单位**（等效 Fuel 数量）暴露其 `resource_ceiling`
2. 转换公式由接收方绑定定义，而非 IR 层
3. IR 模块的 `rotifer.constraints.fuel.maxFuel` 与接收方绑定的标准化上限进行比较
4. 若 `maxFuel > resource_ceiling`，协商返回 `Incompatible`，原因为 `ResourceBudgetExceeded`

**结果报告：**

同一 IR 在不同绑定中执行时，`GeneResult` 中的 `resourceUsed` 字段以绑定的原生单位报告。跨绑定资源比较需要调用方应用接收方绑定的转换公式。

---

## 6. Host Functions | 宿主函数

### 6.1 设计原则

宿主函数是 IR 与外部世界（运行时环境、其他基因、Agent 状态）交互的 **唯一通道**。所有宿主函数具有以下共同属性：

- **显式声明：** IR 模块必须在导入段中声明其使用的所有宿主函数
- **权限标注：** 每个宿主函数调用标注所需权限级别
- **计量参与：** 宿主函数的资源消耗纳入整体计量

### 6.2 标准宿主函数列表

以下宿主函数由所有合规绑定必须提供：

**基础设施函数：**

```
module "rotifer" {

    // ── 日志 ──
    // 记录执行日志（不影响计算结果，仅用于调试和审计）
    function log(level: i32, msgPtr: i32, msgLen: i32) -> void
    // level: 0=DEBUG, 1=INFO, 2=WARN, 3=ERROR

    // ── 上下文读取 ──
    // 读取执行上下文中的指定字段
    function readContext(keyPtr: i32, keyLen: i32, outPtr: i32, outBufLen: i32) -> i32
    // 返回实际写入的字节数；若 outBufLen 不足则返回 -1

    // ── 资源查询 ──
    // 查询剩余资源预算
    function remainingBudget() -> i64

    // ── 时间 ──
    // 获取宿主注入的逻辑时间戳（非系统时钟）
    function logicalTimestamp() -> i64
}
```

**基因间通信函数：**

```
module "rotifer.gene" {

    // 调用另一个基因（同步）
    // 被调用基因必须在当前 Agent 的基因组中存在
    function call(
        geneIdPtr: i32, geneIdLen: i32,     // 目标基因 ID
        inputPtr: i32, inputLen: i32,        // 输入数据
        outPtr: i32, outBufLen: i32          // 输出缓冲区
    ) -> i32
    // 返回: >0 = 输出字节数, 0 = 目标基因无输出, <0 = 错误码

    // 查询基因是否可用
    function exists(geneIdPtr: i32, geneIdLen: i32) -> i32
    // 返回: 1 = 存在且可调用, 0 = 不存在或不可调用

    // 查询当前调用深度（用于控制器基因的嵌套深度检查）
    function callDepth() -> i32
    // 返回: 当前基因间调用的嵌套层数（顶层为 0）
}
```

**控制器基因的调用约束 (Controller Gene Call Constraints)：**

`rotifer.gene.call()` 支持递归调用（基因 A 调用基因 B，基因 B 再调用基因 C），这是控制器基因模式的基础。递归调用受以下约束：

| 约束 | 默认值 | 说明 |
|------|--------|------|
| `maxCallDepth` | 8 | 基因间调用的最大嵌套深度。超过时 `call()` 返回 `ERROR_MAX_DEPTH_EXCEEDED` (-100) |
| `maxControllerDepth` | 3 | 控制器基因调用另一个控制器基因的最大嵌套深度。由运行时通过 `phenotype().domain` 前缀 `orchestrate.*` / `plan.*` 识别 |
| `callFuelMultiplier` | 1.5x | 每一层嵌套调用的燃料消耗乘数。深度 1 消耗 1.5x 基础成本，深度 2 消耗 2.25x（= 1.5^2），以此类推。这通过指数增长的成本自然地抑制过深嵌套 |

**控制器基因的审计日志：** 当一个基因的 `rotifer.gene.call()` 调用次数超过 `controllerAuditThreshold`（默认 5 次）时，运行时自动将该基因标记为"控制器行为"，并将每次 `call()` 的目标 ID、输入摘要和返回状态写入审计日志。此行为不影响执行，但确保动态编排的决策过程可追溯。

**密码学函数：**

```
module "rotifer.crypto" {

    // SHA-256 哈希
    function sha256(dataPtr: i32, dataLen: i32, outPtr: i32) -> void
    // outPtr 指向 32 字节输出缓冲区

    // 验证 Ed25519 签名
    function ed25519Verify(
        msgPtr: i32, msgLen: i32,
        sigPtr: i32,                         // 64 字节签名
        pubKeyPtr: i32                       // 32 字节公钥
    ) -> i32
    // 返回: 1 = 验证通过, 0 = 验证失败
}
```

### 6.3 绑定扩展宿主函数

各绑定可以定义额外的宿主函数，但必须遵循以下规则：

1. 扩展函数必须使用 `rotifer.ext.<binding_name>` WASM 模块命名空间（如 `rotifer.ext.web3`）。`<binding_name>` 标识绑定类型，而非单个函数。
2. 模块内的函数名是纯标识符（如 `getBlockNumber`，而非 `web3.getBlockNumber`）。完整的 WASM 导入路径为 `(import "rotifer.ext.web3" "getBlockNumber" ...)`。
3. 使用了扩展函数的 IR 在传播时必须标注依赖的绑定类型。此标注存储在 `rotifer.ext` 自定义段中（§8.1）。
4. 每个扩展函数必须在 `rotifer.ext` 段中标注为 `required`（必需）或 `optional`（可选）。详见 §10.4 和 §10.5 中此标注对跨绑定能力协商的影响。
5. 其他绑定遇到未知扩展函数时应按 §10.4 定义的策略处理，而非崩溃。

**Web3 绑定扩展示例：**

```
module "rotifer.ext.web3" {
    function ethCall(toPtr: i32, dataPtr: i32, dataLen: i32, outPtr: i32, outBufLen: i32) -> i32
    function getBlockNumber() -> i64          // optional — 信息性函数，基因可在缺失时正常运行
    function getChainId() -> i32              // optional — 信息性函数
}
```

**命名约定总结：**

```
WASM import:  (import "rotifer.ext.web3" "getBlockNumber" (func ...))
                       ^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^
                       模块命名空间         函数名
```

---

## 7. Security Model | 安全模型

### 7.1 安全层次

Rotifer IR 的安全模型分为三个层次，层层递进：

```
┌────────────────────────────────────────────┐
│  Layer 3: 运行时安全 (Runtime Security)      │  ← 资源计量 + 异常捕获
│  ┌────────────────────────────────────────┐ │
│  │  Layer 2: 验证时安全 (Verification)     │ │  ← 静态分析 + 证明检查
│  │  ┌────────────────────────────────────┐│ │
│  │  │  Layer 1: 设计安全 (By Design)     ││ │  ← 指令白名单 + 类型系统
│  │  └────────────────────────────────────┘│ │
│  └────────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

### 7.2 Layer 1: 设计安全

**不安全操作的排除是结构性的，不是策略性的：**

| 不安全操作 | 排除方式 |
|-----------|---------|
| 任意内存访问 | 无指针类型；线性内存有界且越界触发陷阱 |
| 系统调用 | 无系统调用指令；所有外部交互通过宿主函数 |
| 无限循环 | 燃料计量强制中断 + 静态终止性检查 |
| 类型混淆 | WASM 的栈式类型检查 + Rotifer 高级类型的序列化校验 |
| 代码注入 | IR 不支持动态代码生成或 `eval` |
| 信息泄漏 | 宿主函数权限标注 + 无直接 I/O |

### 7.3 Layer 2: 验证时安全

IR 模块在执行前必须通过 **Rotifer IR Verifier** 的静态分析：

**验证检查清单：**

```
verifier RotiferIRVerifier {

    // 结构验证
    check validWasmModule()              // 合法的 WASM 二进制格式
    check hasRequiredCustomSections()    // 包含所有必需的 rotifer.* 自定义段
    check noProhibitedInstructions()     // 不包含白名单之外的指令

    // 类型验证
    check inputSchemaValid()             // 输入 Schema 符合确定性子集
    check outputSchemaValid()            // 输出 Schema 符合确定性子集
    check typeConsistency()              // 所有操作的类型在编译时确定

    // 资源验证
    check memoryWithinLimits()           // 内存声明在约束范围内
    check terminationGuarantee()         // 所有循环满足终止性条件
    check resourceBoundComputable()      // 最大资源消耗可静态估算

    // 安全验证
    check hostFunctionsWhitelisted()     // 仅调用白名单宿主函数
    check noFloatingPointAmbiguity()     // 浮点运算无跨平台歧义
    check dependenciesResolvable()       // 声明的基因依赖可解析
}
```

**验证结果：**

| 结果 | 含义 | 后续动作 |
|------|------|---------|
| `PASS` | 通过所有检查 | 可进入 L2 校准流程 |
| `WARN` | 通过必需检查，部分建议检查未通过 | 可进入 L2，但标注警告 |
| `FAIL` | 至少一项必需检查未通过 | 拒绝，不进入 L2 |

### 7.4 Layer 3: 运行时安全

即使通过了静态验证，运行时仍需以下安全机制：

- **燃料计量 (Fuel Metering)：** 每条 WASM 指令消耗预定义的燃料单位。燃料耗尽时执行强制中断。
- **内存越界陷阱：** 线性内存访问越界触发 trap，而非返回垃圾数据。
- **宿主函数超时：** 宿主函数调用设置独立超时，防止外部服务阻塞。
- **输出大小限制：** `GeneOutput` 的序列化大小不超过预配置上限。

### 7.5 与 L2 校准的关系

Rotifer IR Verifier 是 L2 校准的 **前置步骤**：

```
候选基因 (IR) → [IR Verifier: 静态安全检查]
                        ↓ (PASS)
               → [L2 Stage 1: 静态分析（扩展）]
                        ↓
               → [L2 Stage 2: 沙箱模拟 + F(g)/V(g) 计算]
                        ↓
               → [L2 Stage 3: 受控实战]
```

IR Verifier 不替代 L2 校准——它处理的是"能不能安全执行"的底线问题；L2 校准处理的是"执行效果好不好"的质量问题。

---

## 8. Serialization Format | 序列化格式

### 8.1 模块二进制格式

Rotifer IR 模块的二进制格式是标准 WASM 二进制格式的 **严格超集**——任何标准 WASM 解析器都可以解析 Rotifer IR 模块（自定义段被忽略但不阻碍解析）。

**Rotifer 自定义段 (Custom Sections)：**

| 段名称 | 必需 | 内容 | 格式 |
|--------|------|------|------|
| `rotifer.version` | 是 | IR 规范版本号 | `major.minor.patch` (UTF-8) |
| `rotifer.phenotype` | 是 | 基因的 Phenotype 声明 | MessagePack 序列化 |
| `rotifer.constraints` | 是 | 资源约束与安全声明 | MessagePack 序列化 |
| `rotifer.metering` | 是 | 燃料计量配置 | MessagePack 序列化 |
| `rotifer.dependencies` | 否 | 声明依赖的其他基因 ID | `[Hash]` 数组 |
| `rotifer.ext` | 否 | 使用的绑定扩展函数声明 | MessagePack 序列化 |
| `rotifer.source` | 否 | 可选：原始源代码引用 | UTF-8 字符串 |

### 8.2 Phenotype 段格式

```
rotifer.phenotype = MessagePack({
    domain: String,             // "swap.encode"
    inputSchema: Schema,        // JSON Schema 确定性子集
    outputSchema: Schema,
    version: String,            // SemVer "1.2.0"
    author: Bytes,              // AgentIdentity 公钥（32 字节）
    createdAt: u64,             // Unix 时间戳
    description: String?,       // 可选：人类可读描述
    tags: [String]?             // 可选：分类标签
})
```

### 8.3 Constraints 段格式

```
rotifer.constraints = MessagePack({
    memory: {
        maxInitialPages: u32,
        maxGrowPages: u32,
        totalMemoryLimit: u32
    },
    fuel: {
        maxFuel: u64,           // 最大燃料单位
        fuelPerInstruction: {   // 每类指令的燃料成本
            arithmetic: u32,    // 算术运算
            memory: u32,        // 内存操作
            control: u32,       // 控制流
            hostCall: u32       // 宿主函数调用（基础成本）
        }
    },
    output: {
        maxOutputSize: u32      // 输出最大字节数
    },
    hostFunctions: [String]     // 使用的宿主函数白名单
})
```

### 8.4 内容寻址哈希

基因 Phenotype 中的 `irHash` 字段计算方式：

```
irHash = SHA-256(
    rotifer.version 段的字节内容
    || rotifer.phenotype 段的字节内容
    || rotifer.constraints 段的字节内容
    || WASM Code Section 的字节内容
)
```

**注意：** `irHash` 不包含 `rotifer.source`（可选段）和 `rotifer.ext`（绑定相关）。这确保了同一基因在不同绑定中由不同前端编译时，只要核心逻辑相同，`irHash` 就相同。

---

## 9. Compilation Pipeline | 编译管线

### 9.1 总体架构

```
┌─────────────────────────────────────────────────────┐
│                  Rotifer IR Pipeline                  │
│                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐   │
│  │ Frontend │ →  │ Verifier │ →  │   Backend    │   │
│  │  SDK     │    │          │    │    SDK       │   │
│  └──────────┘    └──────────┘    └──────────────┘   │
│       ↑                                ↓            │
│  Source Code                      Target Code       │
│  (any language)                   (any binding)     │
└─────────────────────────────────────────────────────┘
```

### 9.2 Frontend SDK 接口

每种源语言需实现一个编译前端：

```
interface IRFrontend {

    // 将源代码编译为 Rotifer IR
    function compile(
        source: SourceCode,
        phenotype: Phenotype,
        constraints: Constraints?
    ) -> CompileResult

    // 编译结果
    struct CompileResult {
        ir: RotiferIR?          // 成功时返回 IR 模块
        errors: [CompileError]  // 编译错误列表
        warnings: [CompileWarn] // 编译警告列表
    }

    // 报告此前端支持的源语言
    function supportedLanguages() -> [String]
}
```

**已规划的官方前端：**

| 前端 | 源语言 | 优先级 | 状态 |
|------|--------|--------|------|
| `rotifer-frontend-rust` | Rust | P0 | 规划中 |
| `rotifer-frontend-typescript` | TypeScript / JavaScript | P0 | 规划中 |
| `rotifer-frontend-solidity` | Solidity (EVM → IR 转译) | P1 | 研究中 |
| `rotifer-frontend-python` | Python (via RustPython/Pyodide) | P2 | 研究中 |

### 9.3 Backend SDK 接口

每种目标环境需实现一个编译后端：

```
interface IRBackend {

    // 将 IR 编译为目标环境可执行代码
    function compile(
        ir: RotiferIR,
        targetConfig: TargetConfig
    ) -> CompileResult

    // 编译结果
    struct CompileResult {
        code: TargetCode?       // 目标代码
        errors: [CompileError]
        warnings: [CompileWarn]
    }

    // 报告此后端的目标环境
    function targetEnvironment() -> String
}
```

**已规划的官方后端：**

| 后端 | 目标环境 | 输出格式 | 优先级 |
|------|---------|---------|--------|
| `rotifer-backend-wasm` | 通用 WASM 运行时 | 标准 WASM 模块（剥离自定义段） | P0 |
| `rotifer-backend-evm` | EVM 兼容链 | EVM 字节码 | P1 |
| `rotifer-backend-native` | x86_64 / ARM64 | 本地共享库 | P2 |

### 9.4 端到端示例

一个 Rust 编写的基因，编译为 IR，在三个绑定中执行：

```
[Rust 源码: estimate_gas.rs]
    │
    ▼ (rotifer-frontend-rust)
[Rotifer IR: estimate_gas.rir]
    │
    ├──▶ (rotifer-backend-wasm) → WASM 模块 → Wasmer 沙箱执行 [Cloud Binding]
    │
    ├──▶ (rotifer-backend-evm) → EVM 字节码 → 链上执行 [Web3 Binding]
    │
    └──▶ (rotifer-backend-native) → ARM64 共享库 → 嵌入式设备执行 [Edge Binding]
```

---

## 10. Cross-Binding Interop | 跨绑定互操作协议

### 10.1 IR 传播流程

当基因通过 L3 在不同绑定的 Agent 之间传播时：

```
Agent A (Web3 Binding)              Agent B (Cloud Binding)
┌─────────────────┐                ┌─────────────────┐
│ Gene: foo       │                │ 需要 foo 的能力   │
│ 有 IR 版本       │                │                  │
└────────┬────────┘                └────────┬────────┘
         │                                  │
         │  [1] 广播 Gene 元数据              │          ┐
         │  (含 irHash + phenotype)          │          │ 发现协议
         │ ─────────────────────────────────▶│          │ (依赖 L3)
         │                                  │          │
         │  [2] 请求拉取 IR                   │          │
         │◀───────────────────────────────── │          ┘
         │                                  │
         │  [3] 发送 IR 模块                  │          ┐
         │ ─────────────────────────────────▶│          │
         │                                  │  [4] IR Verifier 验证      │ 核心互操作
         │                                  │  [4b] 能力协商 (§10.5)     │ 协议
         │                                  │  [5] 编译为目标绑定代码     │ (可独立验证)
         │                                  │          ┘
         │                                  │
         │                                  │  [6] L2 校准              ┐ 准入流程
         │                                  │  [7] 准入主序列           ┘ (依赖 L2/L4)
         │                                  │
```

**核心互操作协议 vs 完整传播流程：**

上述七个步骤分为三个独立层次：

| 层次 | 步骤 | 依赖 | 可独立验证性 |
|------|------|------|------------|
| 发现协议 | [1]-[2] | L3 P2P 网络 | 无 P2P 栈则无法验证 |
| **核心互操作协议** | **[3]-[5]** | **仅 IR + Binding** | **可用两个绑定独立验证** |
| 准入流程 | [6]-[7] | L2 校准 + Arena | 无 L2/L4 则无法验证 |

**核心互操作协议**（步骤 3-5）是跨绑定互操作测试的最小可验证单元。IR 0.2.0 验证（v0.6.5）已使用 LocalBinding 和 Web3MockBinding 验证了此层。

### 10.2 版本协商

当传播方和接收方支持不同版本的 IR 规范时：

```
传播方 IR 版本: X.Y.Z
接收方支持版本: A.B.C

规则:
- 若 X == A（主版本相同）：直接传输，次版本差异由接收方 Verifier 处理
- 若 X != A（主版本不同）：
  - 接收方检查是否有 X→A 的转译器
  - 有：转译后验证
  - 无：拒绝传输，返回 ERROR_IR_VERSION_INCOMPATIBLE
```

### 10.3 无 IR 基因的降级处理

并非所有基因都有 IR 版本（`irHash` 是 Phenotype 中的可选字段）。无 IR 基因的跨绑定传播规则：

| 场景 | 处理方式 |
|------|---------|
| 目标 Agent 与源 Agent 使用相同绑定 | 直接传播源代码，无需 IR |
| 目标 Agent 使用不同绑定，基因无 IR | 传播失败，返回 `ERROR_NO_IR_AVAILABLE` |
| 目标 Agent 使用不同绑定，基因有 IR | 传播 IR 版本 |

### 10.4 绑定扩展函数的处理

包含绑定扩展宿主函数（`rotifer.ext.*`）的 IR 在跨绑定传播时：

1. 接收方的 Verifier 检查 `rotifer.ext` 自定义段中声明的扩展函数
2. 若接收方绑定提供了所有声明的扩展函数：正常执行
3. 若接收方绑定缺少部分扩展函数：
   - 检查缺少的函数在 `rotifer.ext` 段中标注为 `required` 还是 `optional`
   - 可选函数缺失：见下方策略选择
   - 必需函数缺失：IR 无法在此绑定执行，能力协商返回 `Incompatible`

**扩展函数处理策略：**

绑定可实现以下两种策略之一来处理缺失的可选扩展函数：

| 策略 | 行为 | 权衡 |
|------|------|------|
| **编译期拦截**（推荐） | 能力协商在执行*前*拒绝或警告。基因不会在缺少必需函数的绑定中被实例化。 | 安全性更高：无运行时意外。灵活性较低：基因无法在运行时优雅降级。 |
| **运行时降级** | 注册 stub 宿主函数，返回 `ERROR_UNSUPPORTED`（-200）。基因可被实例化，调用缺失函数时收到错误码，允许基因在自身逻辑中处理降级。 | 灵活性更高：基因可自适应。安全性较低：基因必须正确处理错误，否则可能出现意外行为。 |

**建议：** 使用编译期拦截作为默认策略。运行时降级可作为 opt-in 的兼容模式提供，适用于希望以基因正确处理 `ERROR_UNSUPPORTED` 为代价来最大化基因可移植性的绑定。

### 10.5 能力协商协议 (Capability Negotiation Protocol)

能力协商是确定给定 IR 模块能否在目标绑定中执行的正式子协议。它在核心互操作协议（§10.1）的步骤 [4b] 中被调用。

**输入：**

```
IrRequirements {
    ir_version: String              // IR 规范版本（如 "0.2.0"）
    required_host_functions: [String]   // 基因必须拥有的函数
    optional_host_functions: [String]   // 基因可使用但不强制要求的函数
    min_memory: u64                 // 最小内存需求（字节）
    min_resource_budget: u64        // 最小燃料/Gas 预算
}

BindingCapabilities {
    binding_id: String              // 如 "local"、"web3-mock"、"web3-ethereum"
    host_functions: [String]        // 此绑定提供的所有宿主函数
    resource_ceiling: ResourceCeiling {
        max_memory: u64             // 最大内存（字节）
        max_fuel: u64               // 最大燃料单位（标准化）
        max_timeout_ms: u64         // 最大执行时间
    }
    filesystem_access: bool
    network_access: bool
}
```

**输出：**

```
NegotiationResult =
    | Compatible                    // 所有需求均满足
    | PartiallyCompatible {         // 必需函数满足；部分可选函数缺失
        missing_optional: [String]
      }
    | Incompatible {                // 至少一个硬性需求不满足
        reasons: [IncompatibilityReason]
      }

IncompatibilityReason =
    | MissingRequiredFunction { name: String }
    | MemoryExceeded { required: u64, available: u64 }
    | ResourceBudgetExceeded { required: u64, available: u64 }
    | IrVersionIncompatible { required: String, supported: String }
```

**算法：**

```
function negotiate(requirements: IrRequirements, capabilities: BindingCapabilities) -> NegotiationResult:
    reasons = []

    // 步骤 1：检查必需宿主函数
    for fn in requirements.required_host_functions:
        if fn not in capabilities.host_functions:
            reasons.push(MissingRequiredFunction { name: fn })

    // 步骤 2：检查可选宿主函数，记录缺失
    missing_optional = []
    for fn in requirements.optional_host_functions:
        if fn not in capabilities.host_functions:
            missing_optional.push(fn)

    // 步骤 3：检查内存上限
    if requirements.min_memory > capabilities.resource_ceiling.max_memory:
        reasons.push(MemoryExceeded { ... })

    // 步骤 4：检查资源预算
    if requirements.min_resource_budget > capabilities.resource_ceiling.max_fuel:
        reasons.push(ResourceBudgetExceeded { ... })

    // 步骤 5：确定结果
    if reasons is not empty:
        return Incompatible { reasons }
    if missing_optional is not empty:
        return PartiallyCompatible { missing_optional }
    return Compatible
```

**在核心互操作协议中的使用：**

| 协商结果 | 动作 |
|---------|------|
| `Compatible` | 继续编译和执行 |
| `PartiallyCompatible` | 附带警告继续；基因不应调用缺失函数（编译期拦截策略），或将收到 `ERROR_UNSUPPORTED`（运行时降级策略） |
| `Incompatible` | 拒绝传输；向传播方返回原因 |

---

## 11. Versioning | 版本管理

### 11.1 版本号格式

Rotifer IR 规范使用语义化版本号 `MAJOR.MINOR.PATCH`：

| 版本组件 | 变更规则 | 兼容性影响 |
|---------|---------|-----------|
| `MAJOR` | 不兼容的指令集变更、类型系统重构 | 不同主版本的 IR 不保证兼容 |
| `MINOR` | 向后兼容的新增（新宿主函数、新自定义段） | 旧 IR 可在新环境中运行 |
| `PATCH` | Bug 修复、文档勘误 | 完全兼容 |

### 11.2 版本嵌入

每个 IR 模块在 `rotifer.version` 自定义段中嵌入编译时使用的 IR 规范版本。

### 11.2.1 当前规范版本

本文档定义的 IR 规范版本为 **0.2.0**。根据 §11.1 的规则，主版本号 0 表示 API 尚未稳定。

**0.2.0 相比 0.1.0 的变更：**
- §5.4：新增跨绑定资源计量语义
- §6.3：明确扩展函数命名约定和 required/optional 标注
- §10.1：区分核心互操作协议（步骤 3-5）和完整传播流程
- §10.4：定义编译期拦截与运行时降级两种策略
- §10.5：将能力协商提升为正式子协议，定义类型和算法
- 目录新增 §10.5 条目

IR 1.0 的发布条件：
1. 至少有两个*生产级*（非 Mock）绑定环境通过跨绑定互操作测试
2. 宿主函数分类（§6 必需/可选）经过实际集成验证
3. 完成至少一轮外部开发者 RFC 反馈周期

### 11.3 废弃策略

| 阶段 | 持续时间 | 行为 |
|------|---------|------|
| 活跃 | — | 完全支持 |
| 废弃预告 | ≥ 6 个月 | Verifier 返回 `WARN`，建议迁移 |
| 废弃 | 预告结束后 | Verifier 返回 `FAIL`，拒绝执行 |

### 11.4 迁移工具

当 IR 主版本升级时，官方提供 `rotifer-ir-migrate` 工具：

```
rotifer-ir-migrate --from 0.x --to 1.x input.rir -o output.rir
```

工具负责自动转译指令集变更，并报告无法自动迁移的部分。

---

## Glossary | 术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| Rotifer IR | Intermediate Representation | 轮虫协议的跨绑定基因中间表示格式 |
| WASM | WebAssembly | Rotifer IR 的基础指令集来源 |
| 自定义段 | Custom Section | WASM 二进制格式中的可扩展段，用于嵌入 Rotifer 元数据 |
| 宿主函数 | Host Function | IR 模块调用外部功能的唯一接口 |
| 燃料 | Fuel | 确定性资源计量单位，每条指令消耗预定义数量 |
| IR 前端 | IR Frontend | 将源语言编译为 Rotifer IR 的编译器组件 |
| IR 后端 | IR Backend | 将 Rotifer IR 编译为目标环境代码的编译器组件 |
| IR 验证器 | IR Verifier | 对 IR 模块执行静态安全检查的分析工具 |
| MessagePack | — | 自定义段中使用的紧凑二进制序列化格式 |
| 内容寻址哈希 | Content-Addressable Hash | 基于模块内容计算的唯一标识符（irHash） |
| 燃料计量 | Fuel Metering | 运行时指令级资源消耗跟踪机制 |
| 终止性 | Termination | 保证程序在有限步内完成执行的属性 |
| 确定性子集 | Deterministic Subset | JSON Schema 中去除非确定性特性后的安全子集 |

---

## Future Work | 未来工作

核心规范以下领域对 IR 层有潜在影响，在此标注设计方向以供后续版本实现：

### RemoteGene 编译支持

`AlgebraExpr` 新增了 `RemoteGene(agentId, domain, input, constraints)` 节点。IR 层需扩展以下能力：

- **网络调用节点：** 在 DataFlowGraph 中引入 `RemoteCall` 节点类型，编译 RemoteGene 为包含序列化、网络传输和反序列化的指令序列
- **超时与重试语义：** RemoteCall 节点需携带 `timeout` 和 `retryPolicy` 属性，由运行时绑定执行
- **燃料计量扩展：** 远程调用的燃料消耗应区分本地计算和网络等待，避免因远端延迟耗尽本地燃料

### 形式化验证集成

形式化证明（`FormalProof`）可与 IR 层深度集成：

- **IR 级验证器接口：** 提供从 IR 模块直接提取控制流图（CFG）和数据流图（DFG）的 API，供外部形式化验证工具使用
- **验证标注嵌入：** 允许在 IR 自定义段中嵌入前置条件/后置条件标注，作为验证辅助信息
- **经过验证的优化：** 经形式化验证的 IR 模块可跳过部分运行时安全检查，提升执行效率

### 协议适配器支持

跨协议适配器可能需要 IR 层支持：

- **适配器编译目标：** 将 ProtocolAdapter 的翻译逻辑编译为 IR 模块，使适配器本身可作为基因参与演化
- **外部调用安全沙箱：** 对外部协议调用施加与本地基因执行相同的沙箱和燃料限制

---

## References | 参考文献

1. Rotifer Foundation. (2026). *Rotifer Protocol Specification*. [前置文档]
2. Haas, A., et al. (2017). Bringing the web up to speed with WebAssembly. *ACM SIGPLAN PLDI*.
3. Lattner, C., & Adve, V. (2004). LLVM: A compilation framework for lifelong program analysis & transformation. *IEEE/ACM CGO*.
4. WebAssembly Community Group. (2024). *WebAssembly Specification 2.0*. https://webassembly.github.io/spec/
5. Clark, L. (2019). Standardizing WASI: A system interface to run WebAssembly outside the web. *Mozilla Hacks*.
6. Furukawa, Y. (2023). MessagePack Specification. https://msgpack.org/
7. Cloudflare. (2024). *How Workers works: The V8 Isolates architecture*.
8. CosmWasm Team. (2023). *CosmWasm Documentation: Smart Contracts in WebAssembly*.
9. Wasmer Team. (2025). *Wasmer Runtime Documentation: Metering Middleware*.
10. Wright, A., et al. (2022). *JSON Schema: A Media Type for Describing JSON Documents*. IETF Draft.
11. IEEE. (2019). *IEEE 754-2019: Standard for Floating-Point Arithmetic*.
12. Cox, R. (2007). Regular Expression Matching Can Be Simple And Fast. https://swtch.com/~rsc/regexp/regexp1.html (RE2 safe subset rationale)

---

**© 2026 Rotifer Foundation. This specification is released under CC BY-SA 4.0.**
