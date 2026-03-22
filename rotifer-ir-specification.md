# Rotifer IR Specification

# Rotifer Intermediate Representation Specification: Cross-Binding Gene Portability Standard

**Author:** Rotifer Foundation
**Status:** Draft (Pre-1.0) — This specification is in draft stage; interfaces may undergo breaking changes
**Version:** 0.2.0
**Prerequisite:** [Rotifer Protocol Specification](./rotifer-protocol-specification.md)

> **Stability Warning:** The cross-binding interop protocol (§10) has been partially validated by two binding implementations (LocalBinding + Web3MockBinding) in v0.6.5. Host function classification (§6) and Capability Negotiation (§10.5) may change based on real integration feedback from non-mock bindings. IR 1.0 will be declared after at least two *production* binding environments pass cross-binding interop tests and one round of external developer RFC feedback is completed.

---

## Table of Contents

- [Abstract](#abstract)
- [1. Introduction](#1-introduction)
- [2. Design Goals](#2-design-goals)
- [3. Technical Decision](#3-technical-decision-wasm--rotifer-constraint-layer)
- [4. Type System](#4-type-system)
- [5. Instruction Set Constraints](#5-instruction-set-constraints)
- [6. Host Functions](#6-host-functions)
- [7. Security Model](#7-security-model)
- [8. Serialization Format](#8-serialization-format)
- [9. Compilation Pipeline](#9-compilation-pipeline)
- [10. Cross-Binding Interop](#10-cross-binding-interop)
  - [10.5 Capability Negotiation Protocol](#105-capability-negotiation-protocol)
- [11. Versioning](#11-versioning)
- [Glossary](#glossary)
- [References](#references)

---

## Abstract

Rotifer IR (Rotifer Intermediate Representation) is the cross-binding gene portability layer of the Rotifer Protocol. It enables genes born in one binding environment (e.g., Web3) to be compiled and executed in an entirely different environment (e.g., Cloud, Edge, TEE) — without manual rewriting.

Rotifer IR is to genes what LLVM IR is to programs, or JVM bytecode is to Java classes: an abstract, verifiable, cross-platform compilable logic representation layer. However, unlike general-purpose compiler IRs, Rotifer IR is **Gene-Aware** — its type system natively understands gene input/output patterns, resource constraints, and security boundaries.

This document defines the type system, instruction set constraints, host functions, security model, serialization format, compilation pipeline, and cross-binding interoperability protocol of Rotifer IR.

---

## 1. Introduction

### 1.1 Motivation

One of the core visions of the Rotifer Protocol is to build a **cross-environment gene circulation network**. An excellent gene should not be trapped in the environment where it was born — it should be able to run in any environment that implements a Rotifer binding.

However, vast execution model differences exist between different binding environments:

| Binding Environment | Execution Model | Native Language | Constraint Model |
|---------|---------|---------|---------|
| Web3 (EVM) | Stack-based VM, deterministic, Gas metering | Solidity / Vyper | Gas limit |
| Cloud (K8s) | Containerized process, non-deterministic, no intrinsic metering | Python / Go / Rust | CPU/Mem quota |
| Edge (IoT) | Bare metal / RTOS, extremely resource-constrained | C / Rust / MicroPython | Hardware resource ceiling |
| TEE | Secure enclave, restricted system calls | C / Rust | Enclave memory limit |

Directly transferring source code between these environments is infeasible — EVM bytecode cannot run on ARM bare metal, and Python scripts cannot execute on EVM. An **intermediate layer** is needed to bridge these differences.

### 1.2 Positioning of Rotifer IR

```
[High-level Source Code]            [High-level Source Code]
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
  │          Rotifer IR (defined by this doc)    │
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
   EVM Bytecode  WASM Module  ARM/x86 Binary
```

### 1.3 Comparison with LLVM IR

| Dimension | LLVM IR | Rotifer IR |
|------|---------|-----------|
| Goal | General-purpose program compilation optimization | Gene cross-binding portability |
| Security Model | No intrinsic security constraints | Security by design (no direct memory, no system calls) |
| Type System | Low-level (integers, floats, pointers) | High-level + Gene-Aware (Schema, Context, Result) |
| Resource Metering | None | Built-in deterministic resource metering |
| Serialization | Bitcode (compact but not human-readable) | Based on WASM binary format + custom sections |
| Verification | Optional (opt-in) | Mandatory (must pass verifier before execution) |

---

## 2. Design Goals

The design of Rotifer IR must satisfy the following five goals, listed in priority order:

### 2.1 Security — Highest Priority

IR security does not rely on runtime checks; instead, unsafe operations are excluded at the design level.

**Security Guarantees:**
- No direct memory address manipulation (no pointer arithmetic, no arbitrary memory read/write)
- No system calls (all external interactions must go through host functions)
- No unbounded loops (all loops must have deterministic resource ceilings)
- No floating-point ambiguity (all floating-point operations use IEEE 754 strict mode, or are replaced with fixed-point arithmetic)

### 2.2 Verifiability

The IR must support **static verification** after compilation and before execution — security checks can be completed at the IR level without knowledge of the target environment.

**Verifiable Properties:**
- Type correctness (types of all operations are fully determined at compile time)
- Resource upper bounds (maximum resource consumption can be computed during static analysis)
- Termination (guaranteed gene execution completes in finite steps)
- Host function compliance (only whitelisted host functions are called)

### 2.3 Portability

Compile once to IR, execute in all binding environments that support the IR.

**Portability Constraints:**
- IR does not depend on any specific operating system, CPU architecture, or virtual machine
- IR semantics remain consistent across all bindings (deterministic execution)
- IR serialization format is platform-independent

### 2.4 Compactness

The serialized IR should be as small as possible, suitable for P2P network propagation.

**Compactness Targets:**
- Simple genes (e.g., parameter encoders): IR size < 4 KB
- Complex genes (e.g., transaction builders): IR size < 64 KB
- Support incremental transfer (transmit only diffs from a known version)

### 2.5 Determinism

The same IR + same input must produce the same output across all bindings.

**Determinism Boundaries:**
- Pure computational operations: strictly deterministic
- Host function calls: defined by the host function's semantic contract (may be non-deterministic, e.g., network requests)
- Time-related operations: direct system clock access is prohibited; time information is injected only through Context

---

## 3. Technical Decision: WASM + Rotifer Constraint Layer

### 3.1 Rationale

Among the candidate paths evaluated in the Core Specification, this specification selects the **hybrid approach: WASM + Rotifer Constraint Layer**.

**Reasons for choosing WASM as the base layer:**

1. **Mature language ecosystem:** 40+ languages including Rust, C/C++, Go, AssemblyScript, Python (via Pyodide) can compile to WASM
2. **Mature sandbox runtimes:** Wasmer, Wasmtime, wasm3, WAMR provide ready-to-use secure sandboxes
3. **Broad platform support:** Browsers, servers, and embedded devices all have WASM runtimes
4. **Existing metering solutions:** Instruction-level Gas metering for WASM has mature implementations (e.g., Wasmer metering middleware)
5. **Industry trend:** Cosmos (CosmWasm), NEAR, Polkadot, Fastly, Cloudflare Workers all chose WASM

**Why not pure WASM:**

Standard WASM lacks the following capabilities required by the Rotifer Protocol:
- **Gene-Aware type system:** WASM only has `i32/i64/f32/f64` and does not understand `GeneInput/GeneOutput/Context`
- **Security constraint declarations:** WASM has no standard way to declare "this module does not access the filesystem"
- **Resource upper bound proofs:** WASM does not guarantee termination
- **Metadata association:** WASM has no standard way to embed gene Phenotype information

### 3.2 Hybrid Architecture

Rotifer IR = Standard WASM Module + Rotifer Custom Sections

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

**Advantages:**
- Any standard WASM toolchain can process Rotifer IR (custom sections are ignored but do not impede parsing)
- Rotifer-specific verification and analysis tools only need to read the custom sections
- Custom section formats can evolve independently in the future without affecting the WASM core

---

## 4. Type System

### 4.1 Base Types

Rotifer IR's type system builds upon WASM value types, but adds Gene-Aware high-level types:

**WASM Low-Level Types (used directly):**

| WASM Type | Usage |
|----------|------|
| `i32` | Integers, booleans, enums |
| `i64` | Long integers, timestamps |
| `f64` | Floating-point numbers (strict IEEE 754 mode) |

**Rotifer High-Level Types (declared in custom sections, mapped to WASM linear memory layout):**

```
type GeneInput {
    schema: Schema          // Type definition of input data (JSON Schema subset)
    data: Bytes             // Serialized input data
}

type GeneOutput {
    schema: Schema          // Type definition of output data
    data: Bytes             // Serialized output data
    metadata: OutputMeta    // Execution metadata (duration, resource consumption, etc.)
}

type Context {
    agentId: Hash           // Identity of the executing Agent
    timestamp: u64          // Logical timestamp injected by the host
    environment: EnvMap     // Key-value pairs provided by the binding environment
    resourceBudget: u64     // Remaining resource budget
}

type Result {
    success: Boolean
    output: GeneOutput?
    error: ErrorInfo?
    resourceUsed: u64       // Actual resource units consumed
}
```

### 4.2 Schema System

Gene input/output Schemas use a **deterministic subset of JSON Schema** — the following non-deterministic features are removed:

| JSON Schema Feature | Treatment in Rotifer Schema |
|-----------------|----------------------|
| `$ref` (external references) | Prohibited — all types must be defined inline |
| `additionalProperties: true` | Prohibited — all properties must be explicitly declared |
| `oneOf` / `anyOf` | Allowed, but requires exhaustive enumeration of all possible match paths |
| `pattern` (regular expressions) | Restricted to the RE2 safe subset (guarantees linear-time matching) |
| Default values | Allowed — ensures deterministic behavior in default cases |

### 4.3 Type Mapping

Mapping rules between high-level types and WASM linear memory:

```
GeneInput / GeneOutput / Context → Serialized as MessagePack byte stream
                                    → Written to WASM linear memory
                                    → Passed via (offset: i32, length: i32) pairs

String   → UTF-8 encoded byte sequence + length prefix
Boolean  → i32 (0 = false, 1 = true)
Hash     → 32-byte fixed-length array
Schema   → MessagePack-serialized JSON Schema subset
```

---

## 5. Instruction Set Constraints

### 5.1 WASM Instruction Whitelist

Rotifer IR adopts a **whitelist strategy** — only WASM instructions explicitly included in the whitelist are permitted in IR.

**Allowed instruction categories:**

| Category | Example Instructions | Description |
|------|---------|------|
| Integer arithmetic | `i32.add`, `i64.mul`, `i32.eq` | All deterministic integer operations |
| Floating-point arithmetic | `f64.add`, `f64.mul`, `f64.sqrt` | Strict IEEE 754 mode only |
| Control flow | `block`, `loop`, `if`, `br`, `br_if`, `return` | All structured control flow |
| Local variables | `local.get`, `local.set`, `local.tee` | Stack variable operations |
| Linear memory | `i32.load`, `i32.store`, `memory.size`, `memory.grow` | Constrained memory operations (see §5.2) |
| Function calls | `call`, `call_indirect` | Module-internal + whitelisted host functions only |

**Prohibited operations (not on the whitelist):**

| Prohibited Item | Reason |
|--------|------|
| WASM SIMD instructions | Cross-platform behavioral differences |
| WASM threads and atomic operations | Non-deterministic |
| WASM exception handling proposal | Specification not yet stable |
| Any host-side FFI (non-whitelisted host functions) | Security boundary breach |

### 5.2 Memory Constraints

Linear memory access in Rotifer IR is subject to the following constraints:

```
constraints MemoryConstraints {
    maxInitialPages: u32     // Maximum initial memory pages (64 KB per page)
    maxGrowPages: u32        // Maximum growable pages
    totalMemoryLimit: u32    // Absolute memory ceiling (bytes)
    noOutOfBoundsAccess: true // Out-of-bounds access triggers a trap, not undefined behavior
}
```

**Default constraint values (can be overridden by bindings):**

| Parameter | Default Value | Description |
|------|--------|------|
| `maxInitialPages` | 16 (1 MB) | Sufficient for simple genes |
| `maxGrowPages` | 64 (4 MB) | Upper limit for complex genes |
| `totalMemoryLimit` | 5,242,880 (5 MB) | Includes initial + growth |

### 5.3 Loop Constraints and Termination

All `loop` instructions must satisfy at least one of the following termination conditions:

1. **Statically bounded:** The number of loop iterations is determinable at compile time (e.g., `for i in 0..N` where N is a constant)
2. **Fuel-bounded:** The loop body contains resource metering checkpoints (configured by the `rotifer.metering` section); execution is forcibly interrupted when the budget is exceeded
3. **Convergence-bounded:** The verifier can prove that the loop condition converges in finite steps (via abstract interpretation or symbolic execution)

IR that does not satisfy any of the above conditions will be rejected by the verifier.

### 5.4 Cross-Binding Resource Metering Semantics

Different binding environments may use different resource metering units. The IR specification defines the following interoperability rules:

**Metering Unit Declaration:**

Each binding declares its native metering unit:

| Metering Unit | Binding Examples | Semantics |
|---------------|-----------------|-----------|
| Fuel | Cloud, Local, Edge | Instruction-level deterministic metering (wasmtime fuel) |
| Gas | Web3 (EVM-compatible) | Transaction cost metering; `Gas = Fuel × gas_price` |
| WallClock | TEE | Real-time execution duration in milliseconds |

**Cross-Binding Comparison Rules:**

When performing Capability Negotiation (§10.5) across bindings with different metering units:

1. Each binding exposes a `resource_ceiling` in a **normalized unit** (equivalent fuel count)
2. The conversion formula is defined by the receiving binding, not by the IR layer
3. The IR module's `rotifer.constraints.fuel.maxFuel` is compared against the receiving binding's normalized ceiling
4. If `maxFuel > resource_ceiling`, the negotiation returns `Incompatible` with reason `ResourceBudgetExceeded`

**Result Reporting:**

When the same IR executes in different bindings, the `resourceUsed` field in `GeneResult` is reported in the binding's native unit. Cross-binding resource comparison requires the caller to apply the receiving binding's conversion formula.

---

## 6. Host Functions

### 6.1 Design Principles

Host functions are the **sole channel** through which IR interacts with the outside world (runtime environment, other genes, Agent state). All host functions share the following properties:

- **Explicit declaration:** IR modules must declare all host functions they use in the import section
- **Permission annotation:** Each host function call is annotated with the required permission level
- **Metering participation:** Resource consumption of host functions is included in overall metering

### 6.2 Standard Host Function List

The following host functions must be provided by all compliant bindings:

**Infrastructure Functions:**

```
module "rotifer" {

    // ── Logging ──
    // Record execution logs (does not affect computation results; for debugging and auditing only)
    function log(level: i32, msgPtr: i32, msgLen: i32) -> void
    // level: 0=DEBUG, 1=INFO, 2=WARN, 3=ERROR

    // ── Context Reading ──
    // Read a specified field from the execution context
    function readContext(keyPtr: i32, keyLen: i32, outPtr: i32, outBufLen: i32) -> i32
    // Returns the number of bytes actually written; returns -1 if outBufLen is insufficient

    // ── Resource Query ──
    // Query the remaining resource budget
    function remainingBudget() -> i64

    // ── Time ──
    // Get the logical timestamp injected by the host (not the system clock)
    function logicalTimestamp() -> i64
}
```

**Inter-Gene Communication Functions:**

```
module "rotifer.gene" {

    // Call another gene (synchronous)
    // The callee gene must exist in the current Agent's genome
    function call(
        geneIdPtr: i32, geneIdLen: i32,     // Target gene ID
        inputPtr: i32, inputLen: i32,        // Input data
        outPtr: i32, outBufLen: i32          // Output buffer
    ) -> i32
    // Returns: >0 = output byte count, 0 = target gene has no output, <0 = error code

    // Query whether a gene is available
    function exists(geneIdPtr: i32, geneIdLen: i32) -> i32
    // Returns: 1 = exists and callable, 0 = does not exist or not callable

    // Query current call depth (for nesting depth checks in controller genes)
    function callDepth() -> i32
    // Returns: current nesting level of inter-gene calls (0 at top level)
}
```

**Controller Gene Call Constraints:**

`rotifer.gene.call()` supports recursive calls (gene A calls gene B, gene B calls gene C), which is the foundation of the controller gene pattern. Recursive calls are subject to the following constraints:

| Constraint | Default Value | Description |
|------|--------|------|
| `maxCallDepth` | 8 | Maximum nesting depth of inter-gene calls. When exceeded, `call()` returns `ERROR_MAX_DEPTH_EXCEEDED` (-100) |
| `maxControllerDepth` | 3 | Maximum nesting depth for a controller gene calling another controller gene. Identified at runtime via `phenotype().domain` prefix `orchestrate.*` / `plan.*` |
| `callFuelMultiplier` | 1.5x | Fuel consumption multiplier for each nesting level. Depth 1 costs 1.5x the base cost, depth 2 costs 2.25x (= 1.5^2), and so on. This naturally discourages excessively deep nesting through exponentially growing costs |

**Controller Gene Audit Log:** When a gene's `rotifer.gene.call()` invocation count exceeds `controllerAuditThreshold` (default: 5), the runtime automatically tags that gene as exhibiting "controller behavior" and writes the target ID, input digest, and return status of each `call()` to an audit log. This does not affect execution but ensures that the decision-making process of dynamic orchestration is traceable.

**Cryptographic Functions:**

```
module "rotifer.crypto" {

    // SHA-256 hash
    function sha256(dataPtr: i32, dataLen: i32, outPtr: i32) -> void
    // outPtr points to a 32-byte output buffer

    // Verify an Ed25519 signature
    function ed25519Verify(
        msgPtr: i32, msgLen: i32,
        sigPtr: i32,                         // 64-byte signature
        pubKeyPtr: i32                       // 32-byte public key
    ) -> i32
    // Returns: 1 = verification passed, 0 = verification failed
}
```

### 6.3 Binding Extension Host Functions

Each binding may define additional host functions, but must adhere to the following rules:

1. Extension functions must use the `rotifer.ext.<binding_name>` WASM module namespace (e.g., `rotifer.ext.web3`). The `<binding_name>` identifies the binding type, not the individual function.
2. Function names within the module are plain identifiers (e.g., `getBlockNumber`, not `web3.getBlockNumber`). The full WASM import path is `(import "rotifer.ext.web3" "getBlockNumber" ...)`.
3. IR that uses extension functions must annotate the dependent binding type when propagated. This annotation is stored in the `rotifer.ext` custom section (§8.1).
4. Each extension function must be annotated as `required` or `optional` in the `rotifer.ext` section. See §10.4 and §10.5 for how this annotation affects cross-binding Capability Negotiation.
5. Other bindings encountering unknown extension functions should handle them according to the strategy defined in §10.4, rather than crashing.

**Web3 Binding Extension Example:**

```
module "rotifer.ext.web3" {
    function ethCall(toPtr: i32, dataPtr: i32, dataLen: i32, outPtr: i32, outBufLen: i32) -> i32
    function getBlockNumber() -> i64          // optional — informational, gene can function without it
    function getChainId() -> i32              // optional — informational
}
```

**Naming Convention Summary:**

```
WASM import:  (import "rotifer.ext.web3" "getBlockNumber" (func ...))
                       ^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^
                       module namespace   function name
```

---

## 7. Security Model

### 7.1 Security Layers

Rotifer IR's security model is divided into three progressive layers:

```
┌────────────────────────────────────────────┐
│  Layer 3: Runtime Security                 │  ← Resource metering + exception capture
│  ┌────────────────────────────────────────┐ │
│  │  Layer 2: Verification-Time Security   │ │  ← Static analysis + proof checking
│  │  ┌────────────────────────────────────┐│ │
│  │  │  Layer 1: Security by Design       ││ │  ← Instruction whitelist + type system
│  │  └────────────────────────────────────┘│ │
│  └────────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

### 7.2 Layer 1: Security by Design

**The exclusion of unsafe operations is structural, not policy-based:**

| Unsafe Operation | Exclusion Method |
|-----------|---------|
| Arbitrary memory access | No pointer type; linear memory is bounded and out-of-bounds triggers a trap |
| System calls | No system call instructions; all external interactions go through host functions |
| Infinite loops | Fuel metering forced interruption + static termination checks |
| Type confusion | WASM's stack-based type checking + serialization validation of Rotifer high-level types |
| Code injection | IR does not support dynamic code generation or `eval` |
| Information leakage | Host function permission annotations + no direct I/O |

### 7.3 Layer 2: Verification-Time Security

IR modules must pass static analysis by the **Rotifer IR Verifier** before execution:

**Verification Checklist:**

```
verifier RotiferIRVerifier {

    // Structural verification
    check validWasmModule()              // Valid WASM binary format
    check hasRequiredCustomSections()    // Contains all required rotifer.* custom sections
    check noProhibitedInstructions()     // No instructions outside the whitelist

    // Type verification
    check inputSchemaValid()             // Input Schema conforms to the deterministic subset
    check outputSchemaValid()            // Output Schema conforms to the deterministic subset
    check typeConsistency()              // Types of all operations are determined at compile time

    // Resource verification
    check memoryWithinLimits()           // Memory declarations are within constraint bounds
    check terminationGuarantee()         // All loops satisfy termination conditions
    check resourceBoundComputable()      // Maximum resource consumption is statically estimable

    // Security verification
    check hostFunctionsWhitelisted()     // Only whitelisted host functions are called
    check noFloatingPointAmbiguity()     // No cross-platform floating-point ambiguity
    check dependenciesResolvable()       // Declared gene dependencies are resolvable
}
```

**Verification Results:**

| Result | Meaning | Subsequent Action |
|------|------|---------|
| `PASS` | Passed all checks | May proceed to L2 calibration |
| `WARN` | Passed required checks; some advisory checks failed | May proceed to L2, but with warnings annotated |
| `FAIL` | At least one required check failed | Rejected; does not proceed to L2 |

### 7.4 Layer 3: Runtime Security

Even after passing static verification, the following runtime safety mechanisms remain in effect:

- **Fuel Metering:** Each WASM instruction consumes a predefined number of fuel units. Execution is forcibly interrupted when fuel is exhausted.
- **Memory Out-of-Bounds Trap:** Out-of-bounds linear memory access triggers a trap rather than returning garbage data.
- **Host Function Timeout:** Host function calls have independent timeouts to prevent external services from blocking.
- **Output Size Limit:** The serialized size of `GeneOutput` must not exceed a preconfigured ceiling.

### 7.5 Relationship with L2 Calibration

The Rotifer IR Verifier is a **prerequisite step** for L2 Calibration:

```
Candidate Gene (IR) → [IR Verifier: Static Safety Checks]
                        ↓ (PASS)
               → [L2 Stage 1: Static Analysis (Extended)]
                        ↓
               → [L2 Stage 2: Sandbox Simulation + F(g)/V(g) Computation]
                        ↓
               → [L2 Stage 3: Controlled Field Deployment]
```

The IR Verifier does not replace L2 Calibration — it addresses the baseline question of "can this be executed safely"; L2 Calibration addresses the quality question of "does this execute well".

---

## 8. Serialization Format

### 8.1 Module Binary Format

The binary format of a Rotifer IR module is a **strict superset** of the standard WASM binary format — any standard WASM parser can parse a Rotifer IR module (custom sections are ignored but do not impede parsing).

**Rotifer Custom Sections:**

| Section Name | Required | Content | Format |
|--------|------|------|------|
| `rotifer.version` | Yes | IR specification version number | `major.minor.patch` (UTF-8) |
| `rotifer.phenotype` | Yes | Gene's Phenotype declaration | MessagePack serialized |
| `rotifer.constraints` | Yes | Resource constraints and security declarations | MessagePack serialized |
| `rotifer.metering` | Yes | Fuel metering configuration | MessagePack serialized |
| `rotifer.dependencies` | No | Declared dependent gene IDs | `[Hash]` array |
| `rotifer.ext` | No | Binding extension function declarations used | MessagePack serialized |
| `rotifer.source` | No | Optional: original source code reference | UTF-8 string |

### 8.2 Phenotype Section Format

```
rotifer.phenotype = MessagePack({
    domain: String,             // "swap.encode"
    inputSchema: Schema,        // JSON Schema deterministic subset
    outputSchema: Schema,
    version: String,            // SemVer "1.2.0"
    author: Bytes,              // AgentIdentity public key (32 bytes)
    createdAt: u64,             // Unix timestamp
    description: String?,       // Optional: human-readable description
    tags: [String]?             // Optional: classification tags
})
```

### 8.3 Constraints Section Format

```
rotifer.constraints = MessagePack({
    memory: {
        maxInitialPages: u32,
        maxGrowPages: u32,
        totalMemoryLimit: u32
    },
    fuel: {
        maxFuel: u64,           // Maximum fuel units
        fuelPerInstruction: {   // Fuel cost per instruction category
            arithmetic: u32,    // Arithmetic operations
            memory: u32,        // Memory operations
            control: u32,       // Control flow
            hostCall: u32       // Host function calls (base cost)
        }
    },
    output: {
        maxOutputSize: u32      // Maximum output size in bytes
    },
    hostFunctions: [String]     // Whitelist of host functions used
})
```

### 8.4 Content-Addressable Hash

The `irHash` field in the gene Phenotype is computed as follows:

```
irHash = SHA-256(
    byte content of rotifer.version section
    || byte content of rotifer.phenotype section
    || byte content of rotifer.constraints section
    || byte content of WASM Code Section
)
```

**Note:** `irHash` does not include `rotifer.source` (optional section) or `rotifer.ext` (binding-specific). This ensures that when the same gene is compiled by different frontends in different bindings, the `irHash` is identical as long as the core logic is the same.

---

## 9. Compilation Pipeline

### 9.1 Overall Architecture

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

### 9.2 Frontend SDK Interface

Each source language requires a compilation frontend:

```
interface IRFrontend {

    // Compile source code to Rotifer IR
    function compile(
        source: SourceCode,
        phenotype: Phenotype,
        constraints: Constraints?
    ) -> CompileResult

    // Compilation result
    struct CompileResult {
        ir: RotiferIR?          // Returns the IR module on success
        errors: [CompileError]  // List of compilation errors
        warnings: [CompileWarn] // List of compilation warnings
    }

    // Report the source languages supported by this frontend
    function supportedLanguages() -> [String]
}
```

**Planned Official Frontends:**

| Frontend | Source Language | Priority | Status |
|------|--------|--------|------|
| `rotifer-frontend-rust` | Rust | P0 | Planned |
| `rotifer-frontend-typescript` | TypeScript / JavaScript | P0 | Planned |
| `rotifer-frontend-solidity` | Solidity (EVM → IR transpilation) | P1 | Under research |
| `rotifer-frontend-python` | Python (via RustPython/Pyodide) | P2 | Under research |

### 9.3 Backend SDK Interface

Each target environment requires a compilation backend:

```
interface IRBackend {

    // Compile IR to target environment executable code
    function compile(
        ir: RotiferIR,
        targetConfig: TargetConfig
    ) -> CompileResult

    // Compilation result
    struct CompileResult {
        code: TargetCode?       // Target code
        errors: [CompileError]
        warnings: [CompileWarn]
    }

    // Report the target environment of this backend
    function targetEnvironment() -> String
}
```

**Planned Official Backends:**

| Backend | Target Environment | Output Format | Priority |
|------|---------|---------|--------|
| `rotifer-backend-wasm` | General WASM runtimes | Standard WASM module (custom sections stripped) | P0 |
| `rotifer-backend-evm` | EVM-compatible chains | EVM bytecode | P1 |
| `rotifer-backend-native` | x86_64 / ARM64 | Native shared library | P2 |

### 9.4 End-to-End Example

A gene written in Rust, compiled to IR, and executed across three bindings:

```
[Rust Source: estimate_gas.rs]
    │
    ▼ (rotifer-frontend-rust)
[Rotifer IR: estimate_gas.rir]
    │
    ├──▶ (rotifer-backend-wasm) → WASM Module → Wasmer Sandbox Execution [Cloud Binding]
    │
    ├──▶ (rotifer-backend-evm) → EVM Bytecode → On-chain Execution [Web3 Binding]
    │
    └──▶ (rotifer-backend-native) → ARM64 Shared Library → Embedded Device Execution [Edge Binding]
```

---

## 10. Cross-Binding Interop

### 10.1 IR Propagation Flow

When a gene propagates between Agents in different bindings via L3:

```
Agent A (Web3 Binding)              Agent B (Cloud Binding)
┌─────────────────┐                ┌─────────────────┐
│ Gene: foo       │                │ Needs foo's     │
│ Has IR version  │                │ capability      │
└────────┬────────┘                └────────┬────────┘
         │                                  │
         │  [1] Broadcast gene metadata     │          ┐
         │  (with irHash + phenotype)       │          │ Discovery
         │ ─────────────────────────────────▶│          │ Protocol
         │                                  │          │ (L3-dependent)
         │  [2] Request to pull IR          │          │
         │◀───────────────────────────────── │          ┘
         │                                  │
         │  [3] Send IR module              │          ┐
         │ ─────────────────────────────────▶│          │
         │                                  │  [4] IR Verifier validation     │ Core Interop
         │                                  │  [4b] Capability Negotiation    │ Protocol
         │                                  │  [5] Compile for target binding │ (independently
         │                                  │          ┘  verifiable)
         │                                  │
         │                                  │  [6] L2 Calibration             ┐ Admission
         │                                  │  [7] Admission to main sequence ┘ (L2/L4-dependent)
         │                                  │
```

**Core Interop Protocol vs Full Propagation Flow:**

The seven steps above divide into three independent layers:

| Layer | Steps | Dependencies | Verifiable Without |
|-------|-------|-------------|-------------------|
| Discovery Protocol | [1]-[2] | L3 P2P network | Cannot verify without P2P stack |
| **Core Interop Protocol** | **[3]-[5]** | **IR + Binding only** | **Independently verifiable with two bindings** |
| Admission Flow | [6]-[7] | L2 Calibration + Arena | Cannot verify without L2/L4 |

The **Core Interop Protocol** (steps 3-5) is the minimal verifiable unit for cross-binding interop testing. IR 0.2.0 validation (v0.6.5) has verified this layer with LocalBinding and Web3MockBinding.

### 10.2 Version Negotiation

When the propagating party and the receiving party support different versions of the IR specification:

```
Propagating party IR version: X.Y.Z
Receiving party supported version: A.B.C

Rules:
- If X == A (same major version): Direct transfer; minor version differences handled by receiving party's Verifier
- If X != A (different major version):
  - Receiving party checks if an X→A transpiler is available
  - Available: Transpile then verify
  - Not available: Reject transfer, return ERROR_IR_VERSION_INCOMPATIBLE
```

### 10.3 Fallback Handling for Non-IR Genes

Not all genes have an IR version (`irHash` is an optional field in the Phenotype). Cross-binding propagation rules for non-IR genes:

| Scenario | Handling |
|------|---------|
| Target Agent uses the same binding as the source Agent | Propagate source code directly; IR not required |
| Target Agent uses a different binding; gene has no IR | Propagation fails; returns `ERROR_NO_IR_AVAILABLE` |
| Target Agent uses a different binding; gene has IR | Propagate the IR version |

### 10.4 Handling of Binding Extension Functions

When IR containing binding extension host functions (`rotifer.ext.*`) is propagated cross-binding:

1. The receiving party's Verifier checks the extension functions declared in the `rotifer.ext` section
2. If the receiving binding provides all declared extension functions: normal execution
3. If the receiving binding is missing some extension functions:
   - Check whether the missing functions are annotated as `required` or `optional` in the `rotifer.ext` custom section
   - Optional function missing: see strategy selection below
   - Required function missing: IR cannot execute in this binding; Capability Negotiation returns `Incompatible`

**Extension Function Handling Strategies:**

Bindings may implement one of two strategies for handling missing optional extension functions:

| Strategy | Behavior | Trade-off |
|----------|----------|-----------|
| **Compile-time Interception** (recommended) | Capability Negotiation rejects or warns *before* execution. The gene is never instantiated in a binding that lacks required functions. | Higher safety: no runtime surprises. Lower flexibility: genes cannot gracefully degrade at runtime. |
| **Runtime Degradation** | Register stub host functions that return `ERROR_UNSUPPORTED` (-200). The gene can be instantiated and will receive error codes when calling missing functions, allowing it to handle degradation in its own logic. | Higher flexibility: gene can adapt. Lower safety: gene must handle errors correctly or risk unexpected behavior. |

**Recommendation:** Use Compile-time Interception as the default strategy. Runtime Degradation may be offered as an opt-in compatibility mode for bindings that wish to maximize gene portability at the cost of requiring genes to handle `ERROR_UNSUPPORTED` gracefully.

### 10.5 Capability Negotiation Protocol

Capability Negotiation is the formal sub-protocol that determines whether a given IR module can execute in a target binding. It is invoked as step [4b] of the Core Interop Protocol (§10.1).

**Input:**

```
IrRequirements {
    ir_version: String              // IR spec version (e.g., "0.2.0")
    required_host_functions: [String]   // Functions the gene MUST have
    optional_host_functions: [String]   // Functions the gene CAN use but doesn't require
    min_memory: u64                 // Minimum memory in bytes
    min_resource_budget: u64        // Minimum fuel/gas budget
}

BindingCapabilities {
    binding_id: String              // e.g., "local", "web3-mock", "web3-ethereum"
    host_functions: [String]        // All host functions this binding provides
    resource_ceiling: ResourceCeiling {
        max_memory: u64             // Maximum memory in bytes
        max_fuel: u64               // Maximum fuel units (normalized)
        max_timeout_ms: u64         // Maximum execution time
    }
    filesystem_access: bool
    network_access: bool
}
```

**Output:**

```
NegotiationResult =
    | Compatible                    // All requirements satisfied
    | PartiallyCompatible {         // Required functions satisfied; some optional missing
        missing_optional: [String]
      }
    | Incompatible {                // At least one hard requirement violated
        reasons: [IncompatibilityReason]
      }

IncompatibilityReason =
    | MissingRequiredFunction { name: String }
    | MemoryExceeded { required: u64, available: u64 }
    | ResourceBudgetExceeded { required: u64, available: u64 }
    | IrVersionIncompatible { required: String, supported: String }
```

**Algorithm:**

```
function negotiate(requirements: IrRequirements, capabilities: BindingCapabilities) -> NegotiationResult:
    reasons = []

    // Step 1: Check required host functions
    for fn in requirements.required_host_functions:
        if fn not in capabilities.host_functions:
            reasons.push(MissingRequiredFunction { name: fn })

    // Step 2: Check optional host functions, record missing
    missing_optional = []
    for fn in requirements.optional_host_functions:
        if fn not in capabilities.host_functions:
            missing_optional.push(fn)

    // Step 3: Check memory ceiling
    if requirements.min_memory > capabilities.resource_ceiling.max_memory:
        reasons.push(MemoryExceeded { ... })

    // Step 4: Check resource budget
    if requirements.min_resource_budget > capabilities.resource_ceiling.max_fuel:
        reasons.push(ResourceBudgetExceeded { ... })

    // Step 5: Determine result
    if reasons is not empty:
        return Incompatible { reasons }
    if missing_optional is not empty:
        return PartiallyCompatible { missing_optional }
    return Compatible
```

**Usage in the Core Interop Protocol:**

| Negotiation Result | Action |
|-------------------|--------|
| `Compatible` | Proceed to compile and execute |
| `PartiallyCompatible` | Proceed with warnings; gene should not call missing functions (Compile-time Interception strategy) or will receive `ERROR_UNSUPPORTED` (Runtime Degradation strategy) |
| `Incompatible` | Reject transfer; return reasons to the propagating party |

---

## 11. Versioning

### 11.1 Version Number Format

The Rotifer IR Specification uses Semantic Versioning `MAJOR.MINOR.PATCH`:

| Version Component | Change Rules | Compatibility Impact |
|---------|---------|-----------|
| `MAJOR` | Incompatible instruction set changes, type system restructuring | IR across different major versions is not guaranteed compatible |
| `MINOR` | Backward-compatible additions (new host functions, new custom sections) | Older IR can run in newer environments |
| `PATCH` | Bug fixes, documentation errata | Fully compatible |

### 11.2 Version Embedding

Each IR module embeds the IR specification version used at compilation time in the `rotifer.version` custom section.

### 11.2.1 Current Specification Version

This document defines IR Specification version **0.2.0**. Per §11.1 rules, major version 0 indicates an unstable API.

**0.2.0 Changes from 0.1.0:**
- §5.4: Added cross-binding resource metering semantics
- §6.3: Clarified extension function naming convention and required/optional annotations
- §10.1: Distinguished Core Interop Protocol (steps 3-5) from full propagation flow
- §10.4: Defined compile-time interception vs runtime degradation strategies
- §10.5: Promoted Capability Negotiation to formal sub-protocol with defined types and algorithm
- Updated Table of Contents to include §10.5

Prerequisites for IR 1.0 release:
1. At least two *production* (non-mock) binding environments pass cross-binding interop tests
2. Host function classification (§6 required/optional) validated through real integration
3. At least one round of external developer RFC feedback completed

### 11.3 Deprecation Policy

| Phase | Duration | Behavior |
|------|---------|------|
| Active | — | Fully supported |
| Deprecation notice | ≥ 6 months | Verifier returns `WARN`; migration recommended |
| Deprecated | After notice period ends | Verifier returns `FAIL`; execution refused |

### 11.4 Migration Tools

When the IR major version is upgraded, an official `rotifer-ir-migrate` tool is provided:

```
rotifer-ir-migrate --from 0.x --to 1.x input.rir -o output.rir
```

The tool handles automatic transpilation of instruction set changes and reports any parts that cannot be automatically migrated.

---

## Glossary

| English Term | Chinese (中文) | Definition |
|------|------|------|
| Rotifer IR | 轮虫中间表示 | The cross-binding gene intermediate representation format of the Rotifer Protocol |
| WASM (WebAssembly) | — | The base instruction set source for Rotifer IR |
| Custom Section | 自定义段 | An extensible section in the WASM binary format used to embed Rotifer metadata |
| Host Function | 宿主函数 | The sole interface through which IR modules invoke external functionality |
| Fuel | 燃料 | A deterministic resource metering unit; each instruction consumes a predefined amount |
| IR Frontend | IR 前端 | The compiler component that compiles a source language into Rotifer IR |
| IR Backend | IR 后端 | The compiler component that compiles Rotifer IR into target environment code |
| IR Verifier | IR 验证器 | An analysis tool that performs static safety checks on IR modules |
| MessagePack | — | A compact binary serialization format used in custom sections |
| Content-Addressable Hash | 内容寻址哈希 | A unique identifier computed based on module content (irHash) |
| Fuel Metering | 燃料计量 | A runtime instruction-level resource consumption tracking mechanism |
| Termination | 终止性 | The property guaranteeing that a program completes execution in finite steps |
| Deterministic Subset | 确定性子集 | A safe subset of JSON Schema with non-deterministic features removed |

---

## Future Work

The following areas of the Core Specification have potential implications for the IR layer. Design directions are noted here for future version implementation:

### RemoteGene Compilation Support

`AlgebraExpr` has introduced a new `RemoteGene(agentId, domain, input, constraints)` node. The IR layer needs to extend the following capabilities:

- **Network call node:** Introduce a `RemoteCall` node type in the DataFlowGraph, compiling RemoteGene into instruction sequences that include serialization, network transport, and deserialization
- **Timeout and retry semantics:** RemoteCall nodes must carry `timeout` and `retryPolicy` attributes, enforced by the runtime binding
- **Fuel metering extension:** Fuel consumption for remote calls should distinguish between local computation and network waiting, to prevent remote-end latency from exhausting local fuel

### Formal Verification Integration

Formal proofs (`FormalProof`) can be deeply integrated with the IR layer:

- **IR-level verifier interface:** Provide APIs for extracting control flow graphs (CFG) and data flow graphs (DFG) directly from IR modules for use by external formal verification tools
- **Verification annotation embedding:** Allow pre-condition/post-condition annotations to be embedded in IR custom sections as verification auxiliary information
- **Verified optimizations:** IR modules that have undergone formal verification may skip certain runtime safety checks, improving execution efficiency

### Protocol Adapter Support

Cross-protocol adapters may require IR layer support:

- **Adapter compilation target:** Compile the translation logic of ProtocolAdapters into IR modules, enabling adapters themselves to participate in evolution as genes
- **External call security sandbox:** Subject external protocol calls to the same sandbox and fuel restrictions as local gene execution

---

## References

1. Rotifer Foundation. (2026). *Rotifer Protocol Specification*. [Prerequisite document]
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
