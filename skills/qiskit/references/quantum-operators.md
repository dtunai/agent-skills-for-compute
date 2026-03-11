# Qiskit Quantum Operators and States Reference

Sources:
- [Operators Module Overview](https://quantum.cloud.ibm.com/docs/en/guides/operators-overview)
- [SparsePauliOp API](https://docs.quantum.ibm.com/api/qiskit/qiskit.quantum_info.SparsePauliOp)
- [Operator API](https://quantum.cloud.ibm.com/docs/en/api/qiskit/qiskit.quantum_info.Operator)

## Overview

Qiskit's `qiskit.quantum_info` module provides classes for representing quantum states, operators, and channels. These are essential for quantum algorithm development, simulation, and analysis.

## Quantum States

### Statevector

Represents a pure quantum state as a complex vector.

**Creation:**
```python
from qiskit.quantum_info import Statevector
import numpy as np

# From vector
psi = Statevector([1/np.sqrt(2), 1/np.sqrt(2)])  # |+⟩

# From label
psi = Statevector.from_label('01')  # |01⟩

# From circuit
from qiskit import QuantumCircuit
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
psi = Statevector(qc)  # Bell state
```

**Operations:**
```python
# Inner product
phi = Statevector.from_label('00')
overlap = psi.inner(phi)

# Expectation value
from qiskit.quantum_info import Pauli
obs = Pauli('ZZ')
exp_val = psi.expectation_value(obs)

# Evolve under operator
U = Operator(qc)
psi_evolved = psi.evolve(U)

# Measure
outcome, new_state = psi.measure([0])  # Measure qubit 0
```

**Properties:**
```python
psi.dim         # Hilbert space dimension
psi.num_qubits  # Number of qubits
psi.is_valid()  # Check normalization
psi.purity()    # Purity (1 for pure states)
psi.data        # Raw numpy array
```

### DensityMatrix

Represents mixed or pure states as density matrices.

**Creation:**
```python
from qiskit.quantum_info import DensityMatrix

# From statevector
rho = DensityMatrix(psi)

# From matrix
rho = DensityMatrix(np.array([[0.5, 0], [0, 0.5]]))  # Maximally mixed

# From circuit
rho = DensityMatrix(qc)
```

**Operations:**
```python
# Expectation value
exp_val = rho.expectation_value(obs)

# Evolve
rho_evolved = rho.evolve(U)

# Partial trace (trace out qubits)
rho_reduced = partial_trace(rho, [1])  # Trace out qubit 1

# Purity
print(rho.purity())  # 1 for pure, < 1 for mixed
```

**Entropy:**
```python
from qiskit.quantum_info import entropy

S = entropy(rho)  # Von Neumann entropy
```

## Quantum Operators

### Operator

General unitary or non-unitary operators as matrices.

**Creation:**
```python
from qiskit.quantum_info import Operator

# From circuit
U = Operator(qc)

# From matrix
U = Operator(np.array([[1, 0], [0, -1]]))  # Z gate

# From gate
from qiskit.circuit.library import HGate
U = Operator(HGate())
```

**Operations:**
```python
# Composition
U1 = Operator(HGate())
U2 = Operator(XGate())
U_combined = U1.compose(U2)  # U2 then U1

# Tensor product
U_tensor = U1.tensor(U2)

# Adjoint
U_dag = U.adjoint()

# Power
U_squared = U.power(2)

# Equivalence check
U1.equiv(U2)  # True if equal up to global phase
```

**Properties:**
```python
U.dim           # Operator dimension
U.num_qubits    # Number of qubits
U.is_unitary()  # Check unitarity
U.to_matrix()   # Convert to numpy array
```

### Pauli

Single Pauli string representation.

**Creation:**
```python
from qiskit.quantum_info import Pauli

# From string
P = Pauli('XYZ')  # X ⊗ Y ⊗ Z

# From Z and X booleans
P = Pauli((z_bits, x_bits))  # (Z phase, X phase)

# With phase
P = Pauli('iXY')   # i × (X ⊗ Y)
P = Pauli('-ZZ')   # -1 × (Z ⊗ Z)
```

**Operations:**
```python
# Composition
P1 = Pauli('XY')
P2 = Pauli('YZ')
P3 = P1.compose(P2)  # Pauli product with phase

# Tensor product
P_tensor = P1.tensor(P2)

# Conjugation
P1.commutes(P2)  # Check commutation
P1.anticommutes(P2)  # Check anti-commutation

# Evolve Pauli under Clifford
from qiskit.quantum_info import Clifford
cliff = Clifford(qc)
P_evolved = P.evolve(cliff)
```

**Properties:**
```python
P.x         # X component (boolean array)
P.z         # Z component (boolean array)
P.phase     # Phase factor (0, 1, 2, 3 for 1, i, -1, -i)
P.to_label()  # Convert to string
P.to_matrix()  # Convert to matrix
```

### SparsePauliOp

**Most Important**: Linear combination of Pauli strings.

**Creation:**
```python
from qiskit.quantum_info import SparsePauliOp

# From list
H = SparsePauliOp.from_list([
    ('II', -1.052),
    ('IZ', 0.398),
    ('ZI', -0.398),
    ('ZZ', -0.011),
    ('XX', 0.181)
])

# From sparse list (specify qubit indices)
H = SparsePauliOp.from_sparse_list([
    ('Z', [0], 1.0),
    ('Z', [1], -0.5),
    ('ZZ', [0, 1], 0.25)
], num_qubits=3)

# Empty operator
zero_op = SparsePauliOp('I', coeffs=[0])

# Identity
I = SparsePauliOp('II')
```

**Operations:**
```python
# Addition
H_total = H1 + H2

# Multiplication by scalar
H_scaled = 2.5 * H

# Composition (operator product)
H_composed = H1 @ H2

# Tensor product
H_tensor = H1.tensor(H2)

# Adjoint
H_dag = H.adjoint()

# Simplify (combine like terms)
H_simplified = H.simplify()

# Expectation value
exp_val = psi.expectation_value(H)
```

**Properties:**
```python
H.num_qubits       # Number of qubits
H.size             # Number of Pauli terms
H.paulis           # PauliList of Pauli strings
H.coeffs           # Coefficient array
H.to_matrix()      # Convert to dense matrix (careful: exponential size!)
H.is_hermitian()   # Check if Hermitian
```

**Layout Application (for ISA circuits):**
```python
# After transpilation
isa_circuit = pm.run(qc)

# Map observable to physical qubits
isa_H = H.apply_layout(isa_circuit.layout)
```

**Chained Operations:**
```python
H = SparsePauliOp.from_list([('XX', 1.0), ('YY', 1.0), ('ZZ', 0.5)])
H_processed = H.simplify().group_commuting()
```

## PauliList

Collection of Pauli operators.

**Creation:**
```python
from qiskit.quantum_info import PauliList

# From strings
paulis = PauliList(['XX', 'YY', 'ZZ'])

# From Pauli objects
paulis = PauliList([Pauli('XX'), Pauli('YY')])
```

**Operations:**
```python
# Indexing
P = paulis[0]  # First Pauli

# Iteration
for p in paulis:
    print(p.to_label())

# Concatenation
combined = paulis1 + paulis2

# Sorting
sorted_paulis = paulis.sort()

# Commutation check
commutes_matrix = paulis.commutes_with_all(paulis)
```

## Advanced Operators

### Clifford

Stabilizer state and Clifford operations.

**Creation:**
```python
from qiskit.quantum_info import Clifford

# From circuit
cliff = Clifford(qc)

# From stabilizer tableau
cliff = Clifford.from_stabilizer_tableau(stab)
```

**Operations:**
```python
# Compose
cliff_combined = cliff1.compose(cliff2)

# Tensor
cliff_tensor = cliff1.tensor(cliff2)

# Adjoint
cliff_dag = cliff.adjoint()

# Evolve Pauli
P_evolved = P.evolve(cliff)
```

### Chi Matrix and PTM

Process representations for quantum channels.

```python
from qiskit.quantum_info import Chi, PTM

# Chi matrix representation
chi = Chi(quantum_channel)

# Pauli Transfer Matrix
ptm = PTM(quantum_channel)
```

## Quantum Channels

### Kraus Operators

**Creation:**
```python
from qiskit.quantum_info import Kraus

# Define Kraus operators for amplitude damping
K0 = np.array([[1, 0], [0, np.sqrt(1-gamma)]])
K1 = np.array([[0, np.sqrt(gamma)], [0, 0]])

channel = Kraus([K0, K1])
```

### Depolarizing Channel

```python
from qiskit.quantum_info import depolarizing_error

# Single-qubit depolarizing
channel = depolarizing_error(p, num_qubits=1)
```

### Pauli Channel

```python
from qiskit.quantum_info import pauli_error

# X error with probability p
channel = pauli_error([('X', p), ('I', 1-p)])
```

## Random States and Operators

```python
from qiskit.quantum_info import random_statevector, random_unitary, random_hermitian

# Random pure state
psi = random_statevector(dims=4)  # 2-qubit state

# Random unitary
U = random_unitary(dims=4)

# Random Hermitian operator
H = random_hermitian(dims=4)

# Random density matrix
rho = random_density_matrix(dims=4)
```

## Entanglement Measures

### Concurrence

```python
from qiskit.quantum_info import concurrence

# For 2-qubit states
C = concurrence(psi)  # 0 (separable) to 1 (maximally entangled)
```

### Entanglement of Formation

```python
from qiskit.quantum_info import entanglement_of_formation

E = entanglement_of_formation(psi)
```

### Mutual Information

```python
from qiskit.quantum_info import mutual_information

I = mutual_information(rho, [0, 1, 2])  # Partition indices
```

## Fidelity and Distance Measures

### State Fidelity

```python
from qiskit.quantum_info import state_fidelity

F = state_fidelity(psi1, psi2)  # 0 (orthogonal) to 1 (identical)
```

### Process Fidelity

```python
from qiskit.quantum_info import process_fidelity

F_proc = process_fidelity(channel1, channel2)
```

### Trace Distance

```python
from qiskit.quantum_info import trace_distance

D = trace_distance(rho1, rho2)  # 0 (identical) to 1 (orthogonal)
```

## Partial Operations

### Partial Trace

```python
from qiskit.quantum_info import partial_trace

# Trace out qubits 1 and 2, keep qubit 0
rho_reduced = partial_trace(rho, [1, 2])
```

### Partial Transpose

```python
from qiskit.quantum_info import partial_transpose

# Transpose subsystem 1
rho_pt = partial_transpose(rho, [1])
```

## Visualization

### Plot State

```python
from qiskit.visualization import plot_state_qsphere, plot_bloch_multivector

# Qsphere
plot_state_qsphere(psi)

# Bloch vector
plot_bloch_multivector(psi)

# State city
from qiskit.visualization import plot_state_city
plot_state_city(rho)

# Hinton plot
from qiskit.visualization import plot_state_hinton
plot_state_hinton(rho)
```

## Best Practices

1. **Use SparsePauliOp for Hamiltonians**: Avoids exponential memory growth
2. **Simplify after operations**: Call `.simplify()` to combine like terms
3. **Apply layout to observables**: After transpilation, map observables to ISA
4. **Check Hermiticity**: Observables must be Hermitian for physical expectation values
5. **Prefer Statevector for pure states**: DensityMatrix is overkill for pure states
6. **Use random states for testing**: Benchmark algorithms with random inputs
7. **Verify operator properties**: Check unitarity, Hermiticity before use
8. **Partial trace carefully**: Ensure correct subsystem indices

## Common Pitfalls

- **Forgetting to apply layout**: Observables must match ISA circuit qubit mapping
- **Dense matrix conversion**: `to_matrix()` on large SparsePauliOp causes memory issues
- **Not simplifying**: Accumulated SparsePauliOp terms bloat without `.simplify()`
- **Phase errors in Pauli**: Incorrect phase handling leads to wrong results
- **Dimension mismatch**: Ensure operators match state dimensions
- **Unnormalized states**: Statevector must be normalized (checked with `.is_valid()`)
- **Non-Hermitian observables**: Expectation values require Hermitian operators
