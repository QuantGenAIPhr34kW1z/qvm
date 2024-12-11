# Multi-Qubit Quantum Computing

## Tensor Products

Multi-qubit states are formed by tensor products:

```
|ψ⟩ ⊗ |φ⟩ = |ψφ⟩
```

For n qubits, the state space has dimension 2^n.

### Example: Two Qubits

```
|00⟩ = |0⟩ ⊗ |0⟩ = [1, 0, 0, 0]ᵀ
|01⟩ = |0⟩ ⊗ |1⟩ = [0, 1, 0, 0]ᵀ
|10⟩ = |1⟩ ⊗ |0⟩ = [0, 0, 1, 0]ᵀ
|11⟩ = |1⟩ ⊗ |1⟩ = [0, 0, 0, 1]ᵀ
```

QVM uses little-endian ordering: qubit 0 is LSB.

## Two-Qubit Gates

### CNOT (Controlled-X)

Flips target if control is |1⟩:
```
CNOT = [1, 0, 0, 0]    CNOT|00⟩ = |00⟩
       [0, 1, 0, 0]    CNOT|01⟩ = |01⟩
       [0, 0, 0, 1]    CNOT|10⟩ = |11⟩
       [0, 0, 1, 0]    CNOT|11⟩ = |10⟩
```

### CZ (Controlled-Z)

Applies Z if control is |1⟩:
```
CZ = [1, 0, 0,  0]    CZ|00⟩ = |00⟩
     [0, 1, 0,  0]    CZ|01⟩ = |01⟩
     [0, 0, 1,  0]    CZ|10⟩ = |10⟩
     [0, 0, 0, -1]    CZ|11⟩ = -|11⟩
```

### SWAP

Exchanges two qubits:
```
SWAP|01⟩ = |10⟩
SWAP|10⟩ = |01⟩
SWAP = CNOT(1,2) · CNOT(2,1) · CNOT(1,2)
```

## Entanglement

Entangled states cannot be written as tensor products.

### Bell States

The four maximally entangled two-qubit states:

```
|Φ+⟩ = (|00⟩ + |11⟩)/√2  ← Bell state
|Φ-⟩ = (|00⟩ - |11⟩)/√2
|Ψ+⟩ = (|01⟩ + |10⟩)/√2
|Ψ-⟩ = (|01⟩ - |10⟩)/√2
```

Creating |Φ+⟩:
```python
import qvm

builder = qvm.ProgramBuilder()
builder.name("bell")
builder.qubits(2)
builder.classical_bits(2)
builder.h(0)           # |0⟩ → (|0⟩+|1⟩)/√2
builder.cnot(0, 1)     # → (|00⟩+|11⟩)/√2
builder.measure(0, 0)
builder.measure(1, 1)
program = builder.build()

result = qvm.run(program, seed=42)
# Measurements are always correlated: 00 or 11
```

### GHZ State

n-qubit maximally entangled state:
```
|GHZ⟩ = (|00...0⟩ + |11...1⟩)/√2
```

```python
program = qvm.Program.ghz(5)  # 5-qubit GHZ
result = qvm.run(program, seed=42)
# All qubits measured same value
```

## Three-Qubit Gates

### Toffoli (CCX)

Controlled-Controlled-NOT:
```
CCX|110⟩ = |111⟩
CCX|111⟩ = |110⟩
```

Only flips target when both controls are |1⟩.

The Toffoli gate is universal for classical computation and, combined with Hadamard, universal for quantum computation.

### Fredkin (CSWAP)

Controlled-SWAP: swaps two qubits if control is |1⟩.

## Multi-Controlled Gates

QVM supports arbitrary C^n gates:

```python
builder.mcx([0, 1, 2], 3)  # X on qubit 3 when qubits 0,1,2 are all |1⟩
```

Decomposed internally using ancilla-free gray code methods.

## Entanglement Detection

A pure state |ψ⟩ of qubits A and B is separable iff:
```
|ψ⟩ = |α⟩_A ⊗ |β⟩_B
```

For mixed states, use the partial trace and check if ρ_A is pure.

### Schmidt Decomposition

Any bipartite state can be written:
```
|ψ⟩_AB = Σᵢ λᵢ |aᵢ⟩_A ⊗ |bᵢ⟩_B
```

Where λᵢ ≥ 0 are Schmidt coefficients. The state is entangled iff more than one λᵢ > 0.

## Quantum Parallelism

Applying gates to superpositions:
```
H⊗H|00⟩ = (|0⟩+|1⟩)(|0⟩+|1⟩)/2 = (|00⟩+|01⟩+|10⟩+|11⟩)/2
```

This encodes 2^n classical states in n qubits, enabling quantum algorithms.
