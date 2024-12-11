# QVM: Quantum Virtual Machine

## Overview

QVM is a security-first, deterministic Quantum Virtual Machine designed to execute verified quantum bytecode with reproducibility, auditability, and explicit policy controls.

## Core Design Principles

### 1. Deterministic Execution

Every execution with the same seed produces identical results:
- Seeded ChaCha20 PRNG for all random operations
- No implicit randomness from external sources
- Reproducible measurement outcomes across runs

```python
import qvm

program = qvm.Program.bell()
result1 = qvm.run(program, seed=42)
result2 = qvm.run(program, seed=42)
assert result1.bits == result2.bits  # Always true
```

### 2. Mandatory Verification

Programs must pass policy checks before execution:
- Qubit count limits (memory bounds)
- Instruction count limits (CPU bounds)
- Gate whitelist enforcement
- Structural validation

```python
import qvm

config = qvm.VerificationConfig.default()
config.max_qubits = 10
config.max_instructions = 1000

result = qvm.verify(program, config)
if not result.valid:
    print(f"Rejected: {result.errors}")
```

### 3. Hash-Chained Transcripts

All execution events are logged with tamper-evident hash chains:
- SHA-256 hash chaining for integrity
- Measurement outcomes recorded with proofs
- Verifiable execution history

### 4. Fail-Closed Security

Any error results in rejection, not degradation:
- Invalid programs are rejected, not "fixed"
- Policy violations halt execution
- State vectors are zeroized post-execution

## Architecture

```
Input (QIR binary / QASM / Python builder)
    ↓
QIR Decoder (magic bytes, version check, parsing)
    ↓
Verifier (mandatory - limits, whitelist, structure)
    ↓
Runtime (dispatch, PRNG seeding, transcript init)
    ↓
Backend (state-vector / stabilizer / MPS simulation)
    ↓
Output (results + hash-chained transcript)
```

## Available Backends

| Backend | Use Case | Complexity | Gates Supported |
|---------|----------|------------|-----------------|
| State Vector | Full simulation | O(2^n) space | All |
| Stabilizer | Clifford circuits | O(n^2) space | Clifford only |
| MPS | Low-entanglement | O(n * chi^2) | All |

## Key Guarantees

1. **No execution without verification** - Programs must pass policy checks
2. **Reproducible execution** - Same seed = identical results
3. **Tamper-evident logging** - SHA-256 hash chain on all transcripts
4. **Memory cleanup** - State vectors zeroized on drop
5. **Explicit randomness** - All randomness from seeded PRNG
