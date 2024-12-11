# QVM Architecture

> *Security and correctness dominate. Performance is important but secondary.*

---

## 1. Design Principles

QVM is built around five non-negotiable principles:

| Principle | Implementation |
|-----------|----------------|
| **Determinism** | Seeded PRNG, stable instruction ordering, reproducible results |
| **Verifiability** | Mandatory pre-execution checks, no bypass mechanism |
| **Minimal TCB** | Small core, explicit invariants, no dynamic loading |
| **Explicit Policy** | Gate whitelists, resource limits, topology constraints |
| **Auditability** | Hash-chained transcripts, tamper-evident logs |

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            UNTRUSTED ZONE                                   │
│                                                                             │
│    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                    │
│    │   QASM      │    │   Python    │    │    C        │                    │
│    │  Frontend   │    │   Bindings  │    │   Compiler  │                    │
│    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                    │
│           │                  │                  │                           │
│           └──────────────────┼──────────────────┘                           │
│                              │                                              │
│                              ▼                                              │
│                    ┌─────────────────┐                                      │
│                    │   QIR Binary    │  ← Untrusted bytecode                │
│                    │   (*.qir)       │                                      │
│                    └────────┬────────┘                                      │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            TRUST BOUNDARY                                   │
│                                                                             │
│    ┌────────────────────────────────────────────────────────────────────┐   │
│    │                         QIR DECODER                                │   │
│    │                                                                    │   │
│    │  • Magic byte validation (QIR\0)                                   │   │
│    │  • Version compatibility check                                     │   │
│    │  • Structural parsing with bounds checks                           │   │
│    │  • Fail-fast on malformed input                                    │   │
│    └──────────────────────────────┬─────────────────────────────────────┘   │
│                                   │                                         │
│                                   ▼                                         │
│    ┌────────────────────────────────────────────────────────────────────┐   │
│    │                         VERIFIER                                   │   │
│    │                                                                    │   │
│    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │   │
│    │  │   Qubit     │  │ Instruction │  │    Gate     │                 │   │
│    │  │   Limits    │  │   Limits    │  │  Whitelist  │                 │   │
│    │  └─────────────┘  └─────────────┘  └─────────────┘                 │   │
│    │                                                                    │   │
│    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │   │
│    │  │   Depth     │  │  Topology   │  │  Structural │                 │   │
│    │  │   Limits    │  │ Constraints │  │  Validation │                 │   │
│    │  └─────────────┘  └─────────────┘  └─────────────┘                 │   │
│    │                                                                    │   │
│    │                    ⚠️  NO BYPASS POSSIBLE ⚠️                       │   │
│    └──────────────────────────────┬─────────────────────────────────────┘   │
│                                   │                                         │
│                                   ▼                                         │
│    ┌────────────────────────────────────────────────────────────────────┐   │
│    │                         QVM CORE                                   │   │
│    │                                                                    │   │
│    │  ┌─────────────────────────────────────────────────────────────┐   │   │
│    │  │                   Instruction Dispatcher                    │   │   │
│    │  │                                                             │   │   │
│    │  │  • Gate application (delegates to backend)                  │   │   │
│    │  │  • Measurement with seeded PRNG                             │   │   │
│    │  │  • Conditional execution                                    │   │   │
│    │  │  • Reset operations                                         │   │   │
│    │  └─────────────────────────────────────────────────────────────┘   │   │
│    │                                                                    │   │
│    │  ┌─────────────────────────────────────────────────────────────┐   │   │
│    │  │                   Transcript Generator                      │   │   │
│    │  │                                                             │   │   │
│    │  │  • SHA-256 hash chaining                                    │   │   │
│    │  │  • Measurement event logging                                │   │   │
│    │  │  • Execution metadata binding                               │   │   │
│    │  └─────────────────────────────────────────────────────────────┘   │   │
│    └──────────────────────────────┬─────────────────────────────────────┘   │
│                                   │                                         │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            BACKENDS                                         │
│                                                                             │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐        │
│  │   State Vector    │  │    Stabilizer     │  │  Tensor Network   │        │
│  │                   │  │                   │  │                   │        │
│  │  • Full state     │  │  • Clifford-only  │  │  • MPS-based      │        │
│  │  • Deterministic  │  │  • O(n²) tableau  │  │  • Bond dimension │        │
│  │  • Reference impl │  │  • Efficient      │  │    control        │        │
│  │  • ✅ IMPLEMENTED  │  │  • ✅ IMPLEMENTED │  │  • ✅ IMPLEMENTED  │      │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Quantum IR (QIR)

QIR is the canonical bytecode format for QVM.

### Design Goals

- **Explicit**: No implicit allocation or hidden behavior
- **Deterministic**: Well-defined instruction semantics
- **Bounded**: All resources declared upfront
- **Compact**: Efficient binary encoding

### Binary Format

```
┌────────────────────────────────────────────────────────────────┐
│ Offset  │ Size    │ Field                                      │
├─────────┼─────────┼────────────────────────────────────────────┤
│ 0x00    │ 4       │ Magic: "QIR\0"                             │
│ 0x04    │ 1       │ Version                                    │
│ 0x05    │ 1       │ Flags (reserved)                           │
│ 0x06    │ 2       │ Qubit count (u16 LE)                       │
│ 0x08    │ 2       │ Classical bit count (u16 LE)               │
│ 0x0A    │ 4       │ Instruction count (u32 LE)                 │
│ 0x0E    │ 4       │ Metadata length (u32 LE)                   │
│ 0x12    │ var     │ Metadata (JSON)                            │
│ var     │ 4       │ Policy length (u32 LE)                     │
│ var     │ var     │ Policy (JSON)                              │
│ var     │ var     │ Instructions (binary)                      │
└────────────────────────────────────────────────────────────────┘
```

### Instruction Encoding

| Opcode | Instruction | Format |
|--------|-------------|--------|
| 0x01 | Gate | `[opcode][gate_id][target:u16]` |
| 0x02 | ParameterizedGate | `[opcode][gate_type][params][target:u16]` |
| 0x03 | ControlledGate | `[opcode][gate_id][control:u16][target:u16]` |
| 0x05 | DoublyControlledGate | `[opcode][gate_id][c1:u16][c2:u16][target:u16]` |
| 0x06 | Swap | `[opcode][q1:u16][q2:u16]` |
| 0x10 | Measure | `[opcode][qubit:u16][classical:u16]` |
| 0x11 | Reset | `[opcode][qubit:u16]` |
| 0x20 | Barrier | `[opcode][count:u16][qubits...]` |
| 0x30 | Conditional | `[opcode][cbit:u16][expected:u8][nested_inst]` |

---

## 4. Verifier

The verifier is the mandatory gateway. **No program executes without verification.**

### Verification Stages

```
Input Program
     │
     ▼
┌─────────────────┐
│ Resource Limits │ ← Qubit count, instruction count, depth
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Gate Whitelist │ ← Only allowed gates pass
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Structural    │ ← Index bounds, duplicate qubits, etc.
│   Validation    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Policy Checks   │ ← Mid-circuit measurement, conditionals, etc.
└────────┬────────┘
         │
         ▼
    PASS or FAIL
```

### Configuration Options

```rust
VerificationConfig {
    max_qubits: 32,              // Hard cap on qubit count
    max_instructions: 100_000,   // Instruction limit
    max_depth: 10_000,           // Circuit depth limit
    gate_whitelist: AllowAll,    // Or Only([...]) or Except([...])
    allow_mid_circuit_measurement: true,
    allow_reset: true,
    allow_conditional: true,
    require_all_measured: false,
    max_two_qubit_gates: None,   // Optional limit
    max_three_qubit_gates: None, // Optional limit
}
```

---

## 5. Execution Core

The core dispatches instructions and generates transcripts.

### Responsibilities

| Component | Function |
|-----------|----------|
| **Dispatcher** | Routes instructions to backend |
| **PRNG** | ChaCha20 seeded for determinism |
| **Transcript** | SHA-256 hash-chained event log |
| **Cleanup** | Zeroization on completion |

### Execution Flow

```rust
fn execute(program: &Program, seed: u64) -> ExecutionResult {
    // 1. Initialize
    let mut state = StateVector::new(program.qubit_count());
    let mut classical = vec![false; program.classical_bit_count()];
    let mut rng = ChaCha20Rng::seed_from_u64(seed);
    let mut transcript = Transcript::new();

    transcript.start_execution(program.name(), seed, ...);

    // 2. Execute instructions
    for (idx, inst) in program.instructions().enumerate() {
        match inst {
            Gate { gate, target } => {
                state.apply_gate(gate, target);
            }
            Measure { qubit, classical } => {
                let random = rng.gen();
                let result = state.measure(qubit, random);
                classical[classical] = result;
                transcript.record_measurement(idx, qubit, classical, result);
            }
            // ...
        }
    }

    // 3. Finalize
    transcript.end_execution(...);
    state.zeroize();  // Security: clear state

    ExecutionResult { classical, transcript, ... }
}
```

---

## 6. State Vector Backend

Reference implementation using full state-vector simulation.

### State Representation

For n qubits, the state is a vector of 2^n complex amplitudes:

```
|ψ⟩ = Σᵢ αᵢ|i⟩

where i is the basis state index in little-endian bit order:
  |00⟩ = index 0
  |01⟩ = index 1  (qubit 0 = 1)
  |10⟩ = index 2  (qubit 1 = 1)
  |11⟩ = index 3
```

### Gate Application

Single-qubit gate on qubit q:

```rust
fn apply_gate(&mut self, matrix: [[Complex; 2]; 2], target: usize) {
    let target_mask = 1 << target;

    for i in 0..self.dimension() {
        if i & target_mask != 0 { continue; }  // Process pairs once

        let i0 = i;
        let i1 = i | target_mask;

        let a0 = self.amplitudes[i0];
        let a1 = self.amplitudes[i1];

        self.amplitudes[i0] = matrix[0][0] * a0 + matrix[0][1] * a1;
        self.amplitudes[i1] = matrix[1][0] * a0 + matrix[1][1] * a1;
    }
}
```

### Measurement

Measurement collapses the state:

```rust
fn measure(&mut self, qubit: usize, random: f64) -> bool {
    // Calculate P(0)
    let prob_0 = self.amplitudes.iter()
        .enumerate()
        .filter(|(i, _)| i & (1 << qubit) == 0)
        .map(|(_, a)| a.norm_sqr())
        .sum();

    let result = random >= prob_0;  // Deterministic given random

    // Collapse and renormalize
    let norm = if result { 1.0 - prob_0 } else { prob_0 };
    for (i, amp) in self.amplitudes.iter_mut().enumerate() {
        if ((i >> qubit) & 1 == 1) != result {
            *amp = Complex::ZERO;
        } else {
            *amp = amp.scale(1.0 / norm.sqrt());
        }
    }

    result
}
```

---

## 7. Transcript System

Hash-chained event log for audit trail.

### Chain Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                        TRANSCRIPT                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Entry 0: ExecutionStart                                         │
│  ├── program: "bell"                                             │
│  ├── seed: 42                                                    │
│  ├── prev_hash: 0000...0000 (genesis)                            │
│  └── hash: SHA256(prev_hash || content)                          │
│           │                                                      │
│           ▼                                                      │
│  Entry 1: Measurement                                            │
│  ├── qubit: 0, result: true                                      │
│  ├── prev_hash: [Entry 0 hash]                                   │
│  └── hash: SHA256(prev_hash || content)                          │
│           │                                                      │
│           ▼                                                      │
│  Entry 2: Measurement                                            │
│  ├── qubit: 1, result: true                                      │
│  ├── prev_hash: [Entry 1 hash]                                   │
│  └── hash: SHA256(prev_hash || content)                          │
│           │                                                      │
│           ▼                                                      │
│  Entry 3: ExecutionEnd                                           │
│  ├── measurements: 2                                             │
│  ├── results: [true, true]                                       │
│  ├── prev_hash: [Entry 2 hash]                                   │
│  └── hash: SHA256(prev_hash || content)  ← FINAL HASH            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Tamper Detection

Any modification to any entry breaks the chain:

```rust
fn verify(&self) -> bool {
    let mut prev_hash = TranscriptHash::ZERO;

    for entry in &self.entries {
        if entry.prev_hash != prev_hash { return false; }
        if !entry.verify() { return false; }  // Recompute hash
        prev_hash = entry.hash;
    }

    true
}
```

---

## 8. Architectural Invariants

These are **non-negotiable**:

| Invariant | Enforcement |
|-----------|-------------|
| No dynamic code loading | No `eval`, no JIT, no plugins at runtime |
| No implicit randomness | All random from seeded ChaCha20 |
| No backend leakage | Core doesn't expose backend internals |
| No unverified execution | Verification cannot be bypassed |
| Deterministic results | Same seed = same output, always |

---

## 9. Alternative Backends

### Stabilizer Backend (Implemented)

For Clifford circuits only (qvm-backend-stabilizer):

- O(n²) space using Aaronson-Gottesman tableau
- Supports: H, S, S†, X, Y, Z, CNOT, CZ gates
- Efficient measurement (deterministic or random)
- Automatic detection of Clifford-only programs

```rust
pub struct Tableau {
    n: usize,
    x: Vec<Vec<bool>>,  // X Pauli components
    z: Vec<Vec<bool>>,  // Z Pauli components
    r: Vec<u8>,         // Phase bits (mod 4)
}
```

### Tensor Network Backend (Implemented)

Matrix Product State simulation (qvm-backend-mps):

- MPS for low-entanglement circuits
- Configurable maximum bond dimension
- SVD-based truncation with error tracking
- Non-adjacent gates via SWAP network

```rust
pub struct MatrixProductState {
    tensors: Vec<Tensor>,       // One tensor per qubit
    max_bond_dim: usize,        // Truncation limit
    truncation_threshold: f64,  // SVD cutoff
}
```

### Symbolic Backend (Research)

For formal verification (future):

- Symbolic amplitudes
- Equivalence checking
- Proof generation

---

## 10. Crate Dependencies

```
qvm-cli ──────────────────────────────────────────────┐
   │                                                  │
   └──► qvm-core                                      │
           │                                          │
           ├──► qvm-ir                                │
           │                                          │
           ├──► qvm-verify ──────────► qvm-ir         │
           │                                          │
           ├──► qvm-backend-sv ──────► qvm-ir         │
           │                                          │
           ├──► qvm-backend-stabilizer ─► qvm-ir      │
           │                                          │
           └──► qvm-backend-mps ─────► qvm-ir         │
                                                      │
qvm-python ──► qvm-core, qvm-ir, qvm-verify, qvm-qasm │
                                                      │
qvm-qasm ────► qvm-ir                                 │
                                                      │
qvm-noise ───► (standalone, num-complex)              │
```

All dependencies flow downward. No cycles. Minimal coupling.

### Crate Summary

| Crate | Purpose |
|-------|---------|
| `qvm-ir` | Quantum IR definition, binary codec |
| `qvm-verify` | Static verification, policy enforcement |
| `qvm-backend-sv` | State-vector simulator (reference) |
| `qvm-backend-stabilizer` | Clifford circuit simulator |
| `qvm-backend-mps` | Tensor network simulator |
| `qvm-core` | Runtime orchestration, transcripts |
| `qvm-cli` | Command-line interface |
| `qvm-qasm` | OpenQASM 2.0 parser |
| `qvm-python` | Python bindings (PyO3) |
| `qvm-noise` | Noise channel models |
