# Single-Qubit Quantum Computing

## Qubit Representation

A qubit is a two-level quantum system described by a complex vector:

```
|ψ⟩ = α|0⟩ + β|1⟩
```

Where α and β are complex amplitudes satisfying |α|² + |β|² = 1.

In QVM, we represent this as a vector: `[α, β]`

## The Bloch Sphere

Any pure single-qubit state can be visualized on the Bloch sphere:

```
|ψ⟩ = cos(θ/2)|0⟩ + e^(iφ)sin(θ/2)|1⟩
```

Where θ ∈ [0, π] and φ ∈ [0, 2π).

Key states:
- |0⟩ = [1, 0] - North pole
- |1⟩ = [0, 1] - South pole
- |+⟩ = [1, 1]/√2 - +X direction
- |-⟩ = [1, -1]/√2 - -X direction
- |i⟩ = [1, i]/√2 - +Y direction
- |-i⟩ = [1, -i]/√2 - -Y direction

## Single-Qubit Gates

### Pauli Gates

**X (NOT)**: Bit flip
```
X = [0, 1]   X|0⟩ = |1⟩
    [1, 0]   X|1⟩ = |0⟩
```

**Y**: Bit flip + phase flip
```
Y = [0, -i]   Y|0⟩ = i|1⟩
    [i,  0]   Y|1⟩ = -i|0⟩
```

**Z**: Phase flip
```
Z = [1,  0]   Z|0⟩ = |0⟩
    [0, -1]   Z|1⟩ = -|1⟩
```

### Hadamard Gate

Creates superposition:
```
H = [1,  1]/√2    H|0⟩ = |+⟩
    [1, -1]/√2    H|1⟩ = |-⟩
```

### Phase Gates

**S (√Z)**: π/2 phase
```
S = [1, 0]    S|0⟩ = |0⟩
    [0, i]    S|1⟩ = i|1⟩
```

**T (√S)**: π/4 phase
```
T = [1,    0  ]
    [0, e^(iπ/4)]
```

### Rotation Gates

**Rx(θ)**: Rotation around X-axis
```
Rx(θ) = [cos(θ/2),   -i·sin(θ/2)]
        [-i·sin(θ/2),  cos(θ/2) ]
```

**Ry(θ)**: Rotation around Y-axis
```
Ry(θ) = [cos(θ/2), -sin(θ/2)]
        [sin(θ/2),  cos(θ/2)]
```

**Rz(θ)**: Rotation around Z-axis
```
Rz(θ) = [e^(-iθ/2),    0    ]
        [   0,      e^(iθ/2)]
```

### Universal Gate U3

Any single-qubit operation:
```
U3(θ, φ, λ) = [       cos(θ/2),       -e^(iλ)·sin(θ/2)]
              [e^(iφ)·sin(θ/2), e^(i(φ+λ))·cos(θ/2)]
```

## Measurement

Measurement in the computational basis collapses the state:

For |ψ⟩ = α|0⟩ + β|1⟩:
- Probability of 0: |α|²
- Probability of 1: |β|²

After measuring 0: state becomes |0⟩
After measuring 1: state becomes |1⟩

In QVM, measurement uses seeded PRNG for determinism:

```python
import qvm

program = qvm.Program.builder()
program.qubits(1)
program.classical_bits(1)
program.h(0)  # Create superposition
program.measure(0, 0)
result = program.build()

# Same seed = same result
r1 = qvm.run(result, seed=42)
r2 = qvm.run(result, seed=42)
assert r1.bits == r2.bits
```

## Gate Identities

Useful relationships:
- HXH = Z (X and Z are conjugates via H)
- HYH = -Y
- HZH = X
- SS = Z (S is √Z)
- TT = S (T is √S)
- XX = YY = ZZ = I (Paulis are involutions)
- HH = I (H is self-inverse)
