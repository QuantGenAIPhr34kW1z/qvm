# Quantum Algorithms

## The Circuit Model

Quantum computation proceeds by:
1. Initialize qubits to |0⟩^⊗n
2. Apply sequence of unitary gates
3. Measure some or all qubits

QVM faithfully implements this model with deterministic randomness.

## Quantum Teleportation

Transfer quantum state using shared entanglement + classical communication.

```python
import qvm

# Create Bell pair between Alice (qubit 1) and Bob (qubit 2)
# Teleport qubit 0's state to Bob
builder = qvm.ProgramBuilder()
builder.name("teleport")
builder.qubits(3)
builder.classical_bits(2)

# Prepare state to teleport (example: |+⟩)
builder.h(0)

# Create Bell pair
builder.h(1)
builder.cnot(1, 2)

# Bell measurement on qubits 0,1
builder.cnot(0, 1)
builder.h(0)
builder.measure(0, 0)
builder.measure(1, 1)

# Conditional corrections on qubit 2 based on measurements
# (In real teleportation, sent classically to Bob)

program = builder.build()
```

## Deutsch-Jozsa Algorithm

Determines if f:{0,1}^n → {0,1} is constant or balanced in ONE query.

Classical: O(2^(n-1) + 1) queries worst case
Quantum: O(1) queries

```python
# f(x) = x (balanced function)
builder = qvm.ProgramBuilder()
builder.name("deutsch")
builder.qubits(2)
builder.classical_bits(1)

# Prepare |01⟩
builder.x(1)

# Apply H to both
builder.h(0)
builder.h(1)

# Oracle: CNOT implements f(x) = x
builder.cnot(0, 1)

# Apply H to first qubit
builder.h(0)
builder.measure(0, 0)

program = builder.build()
# Measures 1 → balanced, 0 → constant
```

## Grover's Search

Search unstructured database of N items in O(√N) queries.

```python
import math

def grover_circuit(n_qubits, marked_state):
    builder = qvm.ProgramBuilder()
    builder.name("grover")
    builder.qubits(n_qubits)
    builder.classical_bits(n_qubits)

    # Initial superposition
    for q in range(n_qubits):
        builder.h(q)

    # Optimal iterations: ~π√N/4
    iterations = int(math.pi * math.sqrt(2**n_qubits) / 4)

    for _ in range(iterations):
        # Oracle: mark the solution (phase flip)
        # Implementation depends on marked_state

        # Diffusion operator
        for q in range(n_qubits):
            builder.h(q)
        for q in range(n_qubits):
            builder.x(q)
        # Multi-controlled Z
        builder.h(n_qubits - 1)
        # builder.mcx(list(range(n_qubits-1)), n_qubits-1)
        builder.h(n_qubits - 1)
        for q in range(n_qubits):
            builder.x(q)
        for q in range(n_qubits):
            builder.h(q)

    for q in range(n_qubits):
        builder.measure(q, q)

    return builder.build()
```

## Quantum Fourier Transform

Basis transformation from computational to Fourier basis.

```
QFT|j⟩ = (1/√N) Σₖ e^(2πijk/N) |k⟩
```

```python
def qft_circuit(n_qubits):
    builder = qvm.ProgramBuilder()
    builder.name("qft")
    builder.qubits(n_qubits)

    for j in range(n_qubits):
        builder.h(j)
        for k in range(j+1, n_qubits):
            # Controlled rotation by 2π/2^(k-j+1)
            angle = math.pi / (2 ** (k - j))
            builder.crz(angle, k, j)

    # Swap to reverse bit order
    for j in range(n_qubits // 2):
        builder.swap(j, n_qubits - j - 1)

    return builder.build()
```

QFT is used in:
- Shor's factoring algorithm
- Phase estimation
- Quantum simulation

## Variational Quantum Eigensolver (VQE)

Hybrid classical-quantum algorithm for finding ground state energy.

```python
def vqe_ansatz(n_qubits, params):
    """Parametrized circuit ansatz"""
    builder = qvm.ProgramBuilder()
    builder.name("vqe_ansatz")
    builder.qubits(n_qubits)

    # Layer of single-qubit rotations
    for i, (theta, phi) in enumerate(zip(params[::2], params[1::2])):
        builder.ry(theta, i % n_qubits)
        builder.rz(phi, i % n_qubits)

    # Entangling layer
    for i in range(n_qubits - 1):
        builder.cnot(i, i + 1)

    return builder.build()
```

## Algorithm Complexity Classes

| Algorithm | Problem | Speedup |
|-----------|---------|---------|
| Grover | Unstructured search | Quadratic |
| Shor | Factoring | Exponential |
| QFT | Fourier transform | Exponential |
| HHL | Linear systems | Exponential* |
| VQE | Ground state | Problem-dependent |

*With caveats on input/output

## Clifford Circuits

Circuits using only {H, S, CNOT} gates:
- Efficiently simulable classically (Gottesman-Knill)
- Useful for error correction
- QVM's stabilizer backend handles these in O(n²) space

```python
result = qvm.Simulator.is_clifford_only(program)
if result:
    # Use efficient stabilizer simulation
    pass
```

## T-Count Analysis

T gates are expensive in fault-tolerant quantum computing.

QVM tracks T-count during verification:
```python
config = qvm.VerificationConfig.default()
config.max_t_count = 100  # Limit T gates

result = qvm.verify(program, config)
print(f"T-count: {result.t_count}")
```
