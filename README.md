<div align="center">
<table role="presentation" border="0" cellspacing="0" cellpadding="0">
<tr>

<td border="0" colspan="2" width="700" valign="middle" align="center">
  <img src="assets/img/QVM.png" width="700" />
</td>
</tr>
<tr>
<td width="190" border="0" valign="middle" align="center">

<br><br>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/banner/banner.dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="assets/banner/banner.light.svg">
  <img alt="QVM banner" src="assets/banner/banner.light.svg" width="500">
</picture>

<br><br>

</td>

<td colspan="1" border="0" valign="middle" align="center">
<br>
<strong>Quantum Virtual Machine</strong><br>
<em>A hardened, deterministic quantum execution environment.</em>

<br><br>

<img src="https://img.shields.io/badge/build-passing-brightgreen">
<img src="https://img.shields.io/badge/license-EINIX-blue">
<img src="https://img.shields.io/badge/rust-1.75%2B-orange">
<img src="https://img.shields.io/badge/security-hardened-red">

<br><br>

<strong>
<a href="docs/">Documentation</a> ·
<a href="ARCHITECTURE.md">Architecture</a> ·
<a href="SECURE.md">Security Model</a> ·
<a href="ROADMAP.md">Roadmap</a>
</strong>

</td>
</table>
</div>

---

<p align="center">
<em>If you wouldn't run unverified bytecode in production,<br />you shouldn't run unverified quantum circuits either.</em>
</p>

---

## What is QVM?

QVM is a **security-first Quantum Virtual Machine** that treats quantum programs as **untrusted input**.

Think `qemu -sandbox` meets quantum computing.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   Untrusted         Mandatory           Deterministic        Tamper     │
│    Input    ──────► Verification ──────► Execution ──────► Evident     │
│                                                             Transcript  │
│   (QIR)             (Policy)             (Seeded PRNG)     (SHA-256)   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

Unlike typical quantum simulators that focus on expressiveness and physics accuracy, QVM focuses on:

| Feature          | Traditional Simulators | QVM                           |
| ---------------- | ---------------------- | ----------------------------- |
| Input validation | ⚠️ Optional          | ✅**Mandatory**         |
| Reproducibility  | ❌ Random each run     | ✅**Deterministic**     |
| Audit trail      | ❌ None                | ✅**Hash-chained**      |
| Resource limits  | ❌ Crash on OOM        | ✅**Pre-flight checks** |
| Execution policy | ❌ Run anything        | ✅**Gate whitelist**    |

---

## Quick Start

```bash
# Build QVM
cargo build --release

# Create a Bell state program
./target/release/qvm sample bell

# Verify it passes policy
./target/release/qvm verify bell.qir

# Execute with deterministic seed
./target/release/qvm run bell.qir --seed 42

# Run again - identical results guaranteed
./target/release/qvm run bell.qir --seed 42
```

<details>
<summary><b>Sample Output</b></summary>

```
$ ./qvm run bell.qir --seed 42

QVM Execution Results
═════════════════════

Program: bell_state
Shots: 1

Seed: 42
  Result: 11 (3)
  Transcript hash: 8a7d3f...
```

</details>

---

## Why QVM Exists

Most quantum toolchains optimize for:

- ✨ Circuit expressiveness
- 🔧 Hardware mapping
- 📚 Educational demos

They largely ignore:

- 🔒 Verification
- 🔄 Reproducibility
- 🛡️ Adversarial inputs
- 📜 Audit trails
- 🚫 Resource limits

**QVM fills this gap.**

### Built For

| Who                                | Why                                                    |
| ---------------------------------- | ------------------------------------------------------ |
| 🔐**Security Researchers**   | Analyze quantum algorithms in a controlled environment |
| 🔑**Cryptography Engineers** | Test post-quantum migration with reproducible results  |
| ⚙️**CI/CD Pipelines**      | Sandboxed quantum execution with resource limits       |
| 📊**Formal Methods**         | Deterministic semantics for verification               |
| 🎓**Education**              | Teach quantum computing without lying about guarantees |

---

## Core Guarantees

### 1. No Execution Without Verification

```rust
// Every program is verified against policy before execution
let result = runtime.execute(&program)?;  // Verification is implicit
                                          // Fail closed. Always.
```

### 2. Deterministic Execution

```bash
# Same seed = same results. Always.
$ qvm run circuit.qir --seed 12345
Result: 101 (5)

$ qvm run circuit.qir --seed 12345
Result: 101 (5)  # Identical. Guaranteed.
```

### 3. Tamper-Evident Transcripts

Every measurement is logged with SHA-256 hash chaining:

```json
{
  "event_type": "Measurement",
  "qubit": 0,
  "result": true,
  "hash": "a7f3d8...",
  "prev_hash": "c91e4b..."
}
```

### 4. Resource Limits

```rust
let config = VerificationConfig::builder()
    .max_qubits(16)           // Hard cap
    .max_instructions(10000)   // No infinite loops
    .max_depth(1000)           // Circuit depth limit
    .gate_whitelist(GateWhitelist::clifford_only())  // No T gates
    .build();
```

---

## Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              Frontends                      │
                    │         QASM Parser · Python Bindings       │
                    └─────────────────┬───────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────────────┐
                    │              QIR (Bytecode)                 │
                    │         Canonical · Validated · Bounded     │
                    └─────────────────┬───────────────────────────┘
                                      │
                    ┌─────────────────▼───────────────────────────┐
                    │              Verifier                       │
                    │    Gate Whitelist · Limits · Topology       │
                    │                                             │
                    │         ⚠️  MANDATORY - NO BYPASS ⚠️         │
                    └─────────────────┬───────────────────────────┘
                                      │
                    ┌─────────────────▼───────────────────────────┐
                    │              QVM Core                       │
                    │    Dispatch · Measurement · Transcripts     │
                    └─────────────────┬───────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  State Vector   │         │   Stabilizer    │         │  Tensor Network │
│      (✅)       │         │      (✅)       │         │      (✅)       │
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

### Crate Structure

```
qvm/
├── crates/
│   ├── qvm-ir/                   # Quantum IR definition & binary codec
│   ├── qvm-verify/               # Static verifier & policy enforcement
│   ├── qvm-backend-sv/           # State-vector simulator (deterministic)
│   ├── qvm-backend-stabilizer/   # Stabilizer simulator (Clifford circuits)
│   ├── qvm-backend-mps/          # MPS tensor network simulator
│   ├── qvm-core/                 # Runtime orchestration & transcripts
│   ├── qvm-cli/                  # Command-line interface
│   ├── qvm-qasm/                 # OpenQASM 2.0 parser
│   ├── qvm-python/               # Python bindings (PyO3)
│   └── qvm-noise/                # Noise channel models
└── docs/
    ├── ARCHITECTURE.md   # Detailed architecture
    ├── SECURE.md         # Security model & threat analysis
    └── ROADMAP.md        # Development phases
```

---

## Security Model

### Threat Model

QVM assumes:

- ⚠️ QIR input **may be malicious**
- ⚠️ Execution environment **may be shared**
- ⚠️ Users **may attempt DoS**
- ⚠️ Reproducibility failures **are security bugs**

QVM does **NOT** assume:

- ❌ Trusted frontends
- ❌ Trusted users
- ❌ Trusted program authors

### Security Philosophy

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│           Fail closed.  Reject early.  Never guess.              │
│                                                                  │
│                    Prefer refusal over ambiguity.                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Defense in Depth

| Layer                  | Control                                          |
| ---------------------- | ------------------------------------------------ |
| **Parsing**      | Strict binary format, magic bytes, version check |
| **Verification** | Qubit limits, instruction caps, gate whitelist   |
| **Execution**    | Bounded state vector, seeded PRNG                |
| **Output**       | Hash-chained transcripts, zeroization on drop    |

---

## CLI Reference

```bash
# Verify a program
qvm verify program.qir

# Execute with seed
qvm run program.qir --seed 42

# Multiple shots
qvm run program.qir --shots 100 --seed 0

# Export transcript
qvm run program.qir --transcript audit.json

# Verify transcript integrity
qvm transcript audit.json --verify

# Create sample programs
qvm sample bell
qvm sample ghz
qvm sample grover
```

---

## Programmatic Usage

```rust
use qvm_core::{Runtime, RuntimeConfig, Program, Instruction};

// Build a Bell state circuit
let program = Program::builder()
    .name("bell")
    .qubits(2)
    .classical_bits(2)
    .instruction(Instruction::h(0))
    .instruction(Instruction::cnot(0, 1))
    .instruction(Instruction::measure(0, 0))
    .instruction(Instruction::measure(1, 1))
    .build()?;

// Execute with deterministic seed
let config = RuntimeConfig::new(42);
let mut runtime = Runtime::new(config);
let result = runtime.execute(&program)?;

// Results are reproducible
assert_eq!(result.classical_bits[0], result.classical_bits[1]); // Bell correlation
println!("Transcript hash: {}", result.transcript.final_hash());
```

---

## Benchmarks

| Circuit    | Qubits | Depth | Time   |
| ---------- | ------ | ----- | ------ |
| Bell state | 2      | 2     | <1μs  |
| GHZ-10     | 10     | 10    | ~10μs |
| Random-16  | 16     | 100   | ~1ms   |
| Random-20  | 20     | 100   | ~20ms  |
| Random-24  | 24     | 100   | ~300ms |

*State-vector backend, single-threaded, M1 Mac*

---

## Building from Source

```bash
# Clone
git clone https://github.com/QuantGenAIPhr34kW1z/qvm-core qvm
cd qvm

# Build
cargo build --release

# Test
cargo test

# Install
cargo install --path crates/qvm-cli
```

### Requirements

- Rust 1.75+
- No external dependencies (pure Rust)

---

## License

© EINIX SA - All rights reserved.

---

<div align="center">

**QVM is not a demo. It's systems software.**

*Built with paranoia. Tested with malice.*

```
Remember: In quantum computing, if you can't reproduce it,
          you can't trust it. And if you can't verify it,
          you shouldn't run it.
```

</div>
