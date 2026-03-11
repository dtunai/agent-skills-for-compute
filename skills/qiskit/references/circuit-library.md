# Qiskit Circuit Library Reference

Sources:
- [Qiskit Circuit Library API](https://quantum.cloud.ibm.com/docs/en/api/qiskit/circuit_library)
- [Guide to the Qiskit Circuit Library](https://medium.com/qiskit/a-guide-to-the-qiskit-circuit-library-36ee0f189956)

## Overview

The Qiskit circuit library (`qiskit.circuit.library`) provides pre-built quantum circuits for common algorithms, ansätze, and computational primitives.

## Categories

1. **Standard gates and instructions**
2. **Generalized gates**
3. **Boolean logic circuits**
4. **Basis state circuits**
5. **Arithmetic circuits**
6. **Particular quantum circuits**
7. **N-local circuits**
8. **Data encoding circuits**

## N-Local Circuits (Ansätze)

### EfficientSU2

Hardware-efficient ansatz for variational algorithms.

**Structure:**
- Alternating layers of single-qubit SU(2) rotations and entangling gates
- Minimal gate count for expressiveness

**Usage:**
```python
from qiskit.circuit.library import EfficientSU2

ansatz = EfficientSU2(
    num_qubits=4,
    su2_gates=['ry', 'rz'],  # single-qubit rotations
    entanglement='linear',   # connectivity pattern
    reps=3                   # number of repetitions
)

print(f"Parameters: {ansatz.num_parameters}")
```

**Entanglement Patterns:**
- `'linear'`: Chain connectivity (0-1, 1-2, 2-3)
- `'full'`: All-to-all connectivity
- `'circular'`: Ring connectivity (includes n-1 to 0)
- `'pairwise'`: Adjacent pairs only
- `'sca'`: Shifted-circular alternating

**Deprecation Note:** As of Qiskit 2.1, use `qiskit.circuit.library.efficient_su2()` function instead of class.

### RealAmplitudes

Real-valued wavefunction ansatz (no complex amplitudes).

**Structure:**
- Only RY rotations (real-valued gates)
- CX entanglement
- Results in real amplitudes only

**Usage:**
```python
from qiskit.circuit.library import RealAmplitudes

ansatz = RealAmplitudes(
    num_qubits=3,
    entanglement='full',
    reps=2,
    insert_barriers=True  # visual separation
)
```

**Applications:**
- Quantum chemistry (real molecular Hamiltonians)
- Classification circuits (QSVM)
- Optimization problems

**Deprecation Note:** As of Qiskit 2.1, use `qiskit.circuit.library.real_amplitudes()` function.

### TwoLocal

Generalized two-local circuit template.

**Structure:**
Customizable rotation and entanglement blocks.

**Usage:**
```python
from qiskit.circuit.library import TwoLocal

ansatz = TwoLocal(
    num_qubits=4,
    rotation_blocks=['ry', 'rz'],
    entanglement_blocks='cx',
    entanglement='linear',
    reps=3,
    skip_final_rotation_layer=False
)
```

**Custom Blocks:**
```python
from qiskit import QuantumCircuit

# Custom rotation block
def custom_rotation(num_qubits):
    qc = QuantumCircuit(num_qubits)
    for i in range(num_qubits):
        qc.rx(Parameter(f'θ_{i}'), i)
        qc.ry(Parameter(f'φ_{i}'), i)
    return qc

ansatz = TwoLocal(
    num_qubits=3,
    rotation_blocks=custom_rotation,
    entanglement_blocks='cz',
    reps=2
)
```

### ExcitationPreserving

Particle-number-conserving ansatz for fermionic systems.

**Structure:**
Preserves total qubit excitation number (Hamming weight).

**Usage:**
```python
from qiskit.circuit.library import ExcitationPreserving

ansatz = ExcitationPreserving(
    num_qubits=4,
    mode='fsim',  # fSim gate (iSWAP + CZ)
    entanglement='linear',
    reps=2
)
```

**Modes:**
- `'iswap'`: iSWAP entanglement
- `'fsim'`: fSim gate (more expressive)

**Applications:**
- Quantum chemistry (particle conservation)
- Lattice models
- VQE for molecular systems

### PauliTwoDesign

2-design based on Pauli rotations.

**Structure:**
Random Pauli rotations forming a 2-design (uniform sampling of Haar measure).

**Usage:**
```python
from qiskit.circuit.library import PauliTwoDesign

circuit = PauliTwoDesign(
    num_qubits=4,
    reps=3,
    seed=42  # reproducibility
)
```

**Applications:**
- Randomized benchmarking
- State tomography
- Quantum machine learning

## Boolean Logic Circuits

### AND, OR, XOR

Classical logic gates implemented with quantum circuits.

**Usage:**
```python
from qiskit.circuit.library import AND, OR, XOR

# AND gate (2 inputs, 1 output)
and_gate = AND(num_variable_qubits=2)

# XOR gate
xor_gate = XOR(num_variable_qubits=3)
```

### Inner Product

Computes inner product (parity) of two bit strings.

**Usage:**
```python
from qiskit.circuit.library import InnerProduct

inner_product = InnerProduct(num_qubits=4)
```

## Arithmetic Circuits

### Adder Circuits

**IntegerComparator:**
```python
from qiskit.circuit.library import IntegerComparator

comparator = IntegerComparator(num_state_qubits=3, value=5)
# Compares quantum register to classical value
```

**WeightedAdder:**
```python
from qiskit.circuit.library import WeightedAdder

adder = WeightedAdder(num_state_qubits=3, weights=[1, 2, 4])
```

### Quadratic Form

Computes quadratic function: x^T A x + b^T x + c

**Usage:**
```python
from qiskit.circuit.library import QuadraticForm
import numpy as np

A = np.array([[1, 0.5], [0.5, 1]])
b = np.array([1, -1])
c = 0.5

quad_form = QuadraticForm(
    num_result_qubits=4,
    quadratic=A,
    linear=b,
    offset=c
)
```

## Particular Quantum Circuits

### Quantum Fourier Transform (QFT)

**Usage:**
```python
from qiskit.circuit.library import QFT

qft = QFT(num_qubits=4, approximation_degree=0, do_swaps=True)

# Inverse QFT
iqft = qft.inverse()
```

**Approximation:**
Set `approximation_degree=k` to drop rotations smaller than π/2^k for shallower circuits.

### Phase Estimation

**Usage:**
```python
from qiskit.circuit.library import PhaseEstimation
from qiskit import QuantumCircuit
from qiskit.quantum_info import Operator

# Define unitary
U = QuantumCircuit(1)
U.t(0)  # T gate: e^(iπ/4)

pe = PhaseEstimation(
    num_evaluation_qubits=4,
    unitary=Operator(U)
)
```

### Grover Operator

**Usage:**
```python
from qiskit.circuit.library import GroverOperator
from qiskit import QuantumCircuit

# Define oracle (marks |11⟩)
oracle = QuantumCircuit(2)
oracle.cz(0, 1)

grover_op = GroverOperator(oracle, reflection_qubits=[0, 1])
```

### GHZ State

**Usage:**
```python
from qiskit.circuit.library import GHZ

ghz = GHZ(num_qubits=5)
```

### W State

**Usage:**
```python
from qiskit.circuit.library import WState

w_state = WState(num_qubits=4)
```

## Data Encoding Circuits

### ZFeatureMap

Encodes classical data using Z rotations.

**Usage:**
```python
from qiskit.circuit.library import ZFeatureMap

feature_map = ZFeatureMap(
    feature_dimension=4,
    reps=2,
    data_map_func=lambda x: x  # identity mapping
)

# Bind data
from qiskit.circuit import ParameterVector
data = [0.1, 0.2, 0.3, 0.4]
bound_circuit = feature_map.assign_parameters(data)
```

**Encoding:**
```
U_Φ(x) = exp(i Σ φ_i(x) Z_i)
```

### ZZFeatureMap

Encodes with ZZ interactions (entangling).

**Usage:**
```python
from qiskit.circuit.library import ZZFeatureMap

feature_map = ZZFeatureMap(
    feature_dimension=3,
    reps=2,
    entanglement='linear'  # or 'full', 'circular'
)
```

**Encoding:**
```
U_Φ(x) = exp(i Σ φ_i(x) Z_i) exp(i Σ φ_{ij}(x_i, x_j) Z_i Z_j)
```

**Applications:**
- Quantum kernel methods
- QSVM (Quantum Support Vector Machine)
- Feature embedding

### PauliFeatureMap

General Pauli-based encoding.

**Usage:**
```python
from qiskit.circuit.library import PauliFeatureMap

feature_map = PauliFeatureMap(
    feature_dimension=4,
    reps=2,
    paulis=['Z', 'ZZ', 'ZZZ']  # Pauli strings to use
)
```

## Hamiltonian Evolution

### PauliEvolutionGate

Time evolution under Pauli Hamiltonian.

**Usage:**
```python
from qiskit.circuit.library import PauliEvolutionGate
from qiskit.quantum_info import SparsePauliOp

# Define Hamiltonian
H = SparsePauliOp.from_list([
    ('XX', 1.0),
    ('YY', 1.0),
    ('ZZ', 0.5)
])

# Evolution gate
evolution = PauliEvolutionGate(H, time=0.1)

# Add to circuit
qc.append(evolution, range(2))
```

**Synthesis Methods:**
- `'lie_trotter'`: First-order Trotter
- `'suzuki'`: Higher-order Suzuki-Trotter
- `'qdrift'`: Randomized Trotter (stochastic)

## IQP Circuits

Instantaneous Quantum Polynomial circuits (classically hard to simulate).

**Usage:**
```python
from qiskit.circuit.library import IQP

iqp = IQP(interactions=[[0, 1], [1, 2], [2, 3]])
```

## Hidden Linear Function

BQP-complete problem circuit.

**Usage:**
```python
from qiskit.circuit.library import HiddenLinearFunction

hlf = HiddenLinearFunction(adjacency_matrix)
```

## Variational Forms (Deprecated)

Legacy ansätze from Qiskit Aqua (use N-local circuits instead):
- `RY`, `RYRZ`, `SwapRZ`: Replaced by `RealAmplitudes`, `EfficientSU2`

## Circuit Composition

Combine library circuits:

```python
from qiskit import QuantumCircuit
from qiskit.circuit.library import QFT, GroverOperator

qc = QuantumCircuit(4)
qc.h(range(4))  # Hadamard layer
qc.append(QFT(4), range(4))  # QFT subcircuit
qc.measure_all()
```

## Parametrization

Access and bind parameters:

```python
# View parameters
print(ansatz.parameters)

# Bind specific values
bound = ansatz.assign_parameters({param: value})

# Bind ordered list
bound = ansatz.assign_parameters([0.1, 0.2, 0.3, ...])

# Partial binding
partial = ansatz.assign_parameters({ansatz.parameters[0]: 0.5})
```

## Best Practices

1. **Use function forms** (Qiskit 2.1+) instead of deprecated classes
2. **Match ansatz to problem**: RealAmplitudes for chemistry, EfficientSU2 for general VQE
3. **Start with low reps**: Increase depth gradually to avoid barren plateaus
4. **Profile parameter count**: More parameters ≠ better performance
5. **Leverage entanglement patterns**: Match to hardware topology
6. **Use insert_barriers=True** for visualization clarity
7. **Check circuit depth** before hardware execution
8. **Transpile library circuits** — they're abstract, not ISA-ready

## Common Pitfalls

- **Over-parameterization**: Too many parameters cause optimization difficulty
- **Wrong entanglement**: Mismatch with backend coupling map increases SWAP overhead
- **Ignoring depth**: Deep circuits suffer more noise
- **Mixing ansätze**: Stick to one ansatz family per VQE run
- **Not checking num_parameters**: Verify before optimizer initialization
- **Forgetting to transpile**: Library circuits require transpilation
