# Aura Lang: Formal Verification & SMT-Driven Compilation Spec
**Document Reference:** PL-AURA-VERIFY-v1.0  
**Module Target:** `aura-verify` / `aura-core`  
**Core Prover Integration:** Z3 SMT Solver Core v4.12+  
**Memory Model:** Non-GC Strict Linear Type Ownership Matrix  

---

## Abstract
This document provides the foundational engineering specification for the top-down mathematical verification layer of the Aura Programming Language. Traditional compiler front-ends rely on syntax checking and structural type inference to establish basic memory safety and symbol alignment. While effective at blocking primitive type mismatches, these static checkers are completely blind to multi-path semantic safety, concurrent deadlocks, linear resource leaks, and temporal hardware invariant violations.

`aura-verify` re-engineers the compilation pipeline by treating source code not merely as a tree of executable instructions, but as a system of first-order logical constraints. By translating the verified Abstract Syntax Tree (AST) directly into symbolic logic profiles (`SmtProfile`) evaluated by an embedded **Z3 SMT Solver**, Aura mathematically ensures semantic validity, memory safety, and thread invariance prior to native assembly code generation via LLVM. This specification details the formal translation mechanics, the constraint generation matrix, and the static enforcement of linear type assets.

---

## 1. The Verified Compilation Pipeline Geometry

The compilation chain shifts safety checks entirely out of the execution domain into compile-time logic validation. The pipeline processes source text sequentially, establishing structural, semantic, and mathematical invariants at each boundary:

```
[Raw Aura Source (`.aura`)]
            │
            ▼
   [aura-lex & aura-parse]  ──► Enforces Lexical Invariants & Generates Token Streams
            │
            ▼
     [Verified AST]         ──► Constructs `Program`, `Stmt`, `ExprKind` Representations
            │
            ▼
     [aura-verify Layer]    ──► Maps Control Flow Graphs (CFG) to Logical Formula Arrays
            │
            ├──► SmtProfile Injections ──► Populates Z3 Solver Contexts Statically
            │
            ▼
    [Z3 SMT Solver Core]
           /           /    (Satisfiable) (Unsatisfiable / Logical Flaw Evident)
        /              ▼         ▼
[LLVM Intermediate IR] [Compilation Abort & Diagnostic Mapping to LSP]
```

### 1.1 AST Node Classifications
The compiler captures grammar semantics in highly structured data definitions (`aura-ast`). The verification subsystem directly processes the following primitive node variations:
* **`Program`:** The global scope definition vector containing arrays of top-level imports, module definitions, and operational strands (`StrandDef`).
* **`Stmt`:** Structural control directives, including immutable bindings (`Let`), linear mutations, loop boundaries, and conditional branch forks.
* **`ExprKind`:** Pure mathematical evaluations, array index operations, object instantiations, capability invocations, and linear value moves (`CallArg`).

---

## 2. AST-to-SMT Translation Framework (`aura-verify`)

### 2.1 The Symbolic Prover Logic Architecture
The core translation layer maps the semantic flow of an Aura program directly to a set of mathematical asset formulas. Each variable binding is treated as an abstract mathematical symbol within a bounded execution window. 

The compiler module `src/verify.rs` initializes a unique `Prover` instance that allocates a isolated verification context for every parsed function block:

```rust
// Architectural Representation of the Verification Core
pub struct Prover<'ctx> {
    pub context: &'ctx z3::Context,
    pub solver: z3::Solver<'ctx>,
    pub binding_matrix: HashMap<String, TypedBinding<'ctx>>,
    pub invariant_assertions: Vec<z3::Bool<'ctx>>,
}
```

When checking a function or a parallel strand execution matrix (`StrandDef`), the `Prover` extracts variables and builds an equivalent logical representation called an `SmtProfile`. Every execution pathway through the function's Control Flow Graph (CFG) must satisfy the generated logic formulas for the compilation to proceed.

### 2.2 Numerical Invariant Generation and Array Bounds Protection
Aura completely eliminates dynamic runtime array-bounds tracking and integer overflow panic routines by enforcing numeric bounds checking during compile-time verification. 

Consider a tracking expression calculating an index offset:

$$	ext{let } y = x + 4; \quad 	ext{let } 	ext{val} = 	ext{array}[y];$$

During compilation, `aura-verify/src/verify.rs` translates this statement sequence into a system of bounded assertions injected straight into the Z3 context:

```
;; Z3 Symbolic Invariant Representation for Array Bounds Validation
(declare-const x Int)
(declare-const y Int)
(declare-const array_len Int)

;; Enforce structural type constraints from the AST
(assert (>= x 0))
(assert (< x 65536))

;; Map the assignment equation
(assert (= y (+ x 4)))

;; The Verification Condition (VC): Is y guaranteed to be within valid array bounds?
;; To prove this holds globally, the prover attempts to satisfy the negation of the safety target
(assert (not (and (>= y 0) (< y array_len))))
```

If the solver discovers a satisfying model for the *negated* condition, it implies that an invalid access path exists. The compiler captures this model as a physical `Counterexample`, maps the logical breakdown directly back into user-space diagnostic metadata (`DiagnosticMetadata`), and triggers an immediate compilation termination.

---

## 3. Linear Type Ownership Verification System (`linear_types.rs`)

### 3.1 The Ownership Invariant Matrix
To deliver execution limits on edge systems without utilizing reference counting schemes or Garbage Collection loops, Aura evaluates resource Lifetimes statically. The module `aura-verify/src/linear_types.rs` monitors variable states across code branches using a formal single-use asset matrix:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum Ownership {
    Owned,      // Resource is active, localized, and ready for deployment
    Borrowed,   // Read-only reference access is active; allocation block is pinned
    Moved,      // Resource asset has moved into a downstream frame; local symbol closed
}
```

The compiler walks the CFG sequentially, updating an internal tracking ledger (`HashMap<String, Ownership>`) to enforce resource state invariants:

```
  ┌──────────────────────────────────────────────────────────┐
  │                   [ OWNED STATE ]                        │
  └────────────────────────────┬─────────────────────────────┘
                               │
                               │  Value Passed as Linearly Moved Argument
                               ▼
  ┌──────────────────────────────────────────────────────────┐
  │                   [ MOVED STATE ]                        │
  ├──────────────────────────────────────────────────────────┤
  │ Structural Restrictions Enforced:                         │
  │  - Subsequent Reads Flag Compilation Failure             │
  │  - Double-Drop Paths Forbidden Mathematically            │
  └──────────────────────────────────────────────────────────┘
```

### 3.2 Mathematical Enforcement of Asset Conservation
Linear values (such as physical I/O descriptors, raw tensor allocation slots, or high-bandwidth network ports) are treated as physical assets. The verification engine converts ownership transitions into logic verification formulas:

1.  **Instantiation Invariant:** When a linear asset is initialized, the tracking compiler asserts its resource allocation footprint inside the current execution sequence:
    $$	ext{Asset}_{	ext{State}}(x) = 	ext{Owned}$$
2.  **Move Invariance:** When passed to an asynchronous execution strand or another function structure, the local context passes ownership completely, marking the target variable dead:
    $$	ext{Asset}_{	ext{State}}(x) \longrightarrow 	ext{Moved}$$
3.  **Path Convergence and Fork Invariance:** When branches intersect (e.g., if-else matrices), the verification framework verifies both paths match perfectly. If variable $x$ is moved inside the true branch but remains active inside the false branch, the states diverge:
    $$	ext{Branch}_{	ext{True}}(x) 
eq 	ext{Branch}_{	ext{False}}(x)$$
    This state discrepancy violates conservation rules. The compiler blocks compilation and provides clear diagnostics to prevent runtime state corruption.

---

## 4. SMT Solver Validation & LSP Diagnostics Integration

### 4.1 Proof Generation and Counterexample Extraction
When the solver identifies an invalid execution sequence, it extracts structural assignments from the machine context to identify the exact cause of the compilation failure:

```rust
// Counterexample Model Evaluation Pipeline
pub fn evaluate_counterexample(&self, error_node: &Expr) -> VerifyError {
    if let Some(model) = self.solver.get_model() {
        let mut trace_ledger = Vec::new();
        for (symbol_name, binding) in &self.binding_matrix {
            if let Some(evaluated_value) = model.eval(&binding.z3_variable, true) {
                trace_ledger.push(format!("Variable {} resolved to: {}", symbol_name, evaluated_value));
            }
        }
        VerifyError::InvalidInvariant {
            node_id: error_node.id,
            metadata: DiagnosticMetadata::new(trace_ledger),
        }
    } else {
        VerifyError::InternalSolverFault
    }
}
```

### 4.2 Language Server Protocol (LSP) Integration Matrix
Rather than outputting obscure compilation summaries, the verification engine links errors directly to the editor workspace (`aura-lsp`). Counterexamples generated by Z3 are sent to the compiler telemetry module, displaying edge-case debugging data inline within the IDE:

```
[Aura Core Compiler Verification Fault]
File: /src/inference/pipeline.aura
Line: 142, Column: 28

[Diagnostic Code: E-0412] - CRITICAL TENSOR DIMENSION MISMATCH DETECTED
Mathematical Verification proved that the downstream Tensor dimension inputs will fail.

Logical Counterexample Path discovered by Z3 Solver Engine:
  - Initial Sensor Stream Vector: [Batch: 1, Channels: 3, Height: 256, Width: 256]
  - Execution Transform Layer 3: Reductions calculated an unexpected target dimension
  - Target Matrix Input Parameter expects Width: 128, but received Width: 64

Downstream execution pipeline would trigger an uncoordinated hardware fault.
Compilation terminated safely. Zero byte code generated.
```

By connecting formal proofs with real-time feedback loops, the compiler turns complex logical analysis into actionable assistance, enabling developers to build safe, low-latency code for critical infrastructure.
