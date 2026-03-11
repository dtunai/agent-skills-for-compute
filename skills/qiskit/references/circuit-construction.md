# Qiskit Circuit Construction Reference

Sources:
- [IBM Quantum Documentation - Construct Circuits](https://quantum.cloud.ibm.com/docs/en/guides/construct-circuits)
- [Qiskit API - QuantumCircuit](https://quantum.cloud.ibm.com/docs/en/api/qiskit/qiskit.circuit.QuantumCircuit)

## Basic Circuit Creation

### Simple Initialization

```python
from qiskit import QuantumCircuit

# Method 1: Specify qubit and classical bit counts
qc = QuantumCircuit(3, 3)  # 3 qubits, 3 classical bits

# Method 2: Qubits only (no classical bits)
qc = QuantumCircuit(5)

# Method 3: Named registers
from qiskit import QuantumRegister, ClassicalRegister

qr = QuantumRegister(4, 'q')
cr = ClassicalRegister(4, 'c')
qc = QuantumCircuit(qr, cr)
```

### Multiple Registers

```python
# Separate control and target qubits
control = QuantumRegister(2, 'control')
target = QuantumRegister(3, 'target')
meas = ClassicalRegister(5, 'meas')

qc = QuantumCircuit(control, target, meas)
```

## Adding Gates

### Single-Qubit Gates

```python
# Pauli gates
qc.x(0)      # Bit flip (NOT)
qc.y(1)      # Y rotation
qc.z(2)      # Phase flip

# Hadamard
qc.h(0)      # Superposition

# Rotation gates
qc.rx(0.5, 0)      # X-axis rotation
qc.ry(1.2, 1)      # Y-axis rotation
qc.rz(0.3, 2)      # Z-axis rotation

# Phase gates
qc.p(0.7, 0)       # Phase gate
qc.s(1)            # S gate (π/2 phase)
qc.t(2)            # T gate (π/4 phase)

# Inverse gates
qc.sdg(1)          # S† (inverse S)
qc.tdg(2)          # T† (inverse T)

# Universal single-qubit gate
qc.u(theta, phi, lam, 0)  # Generic U gate
```

### Multi-Qubit Gates

```python
# Two-qubit gates
qc.cx(0, 1)        # CNOT (control, target)
qc.cy(0, 1)        # Controlled-Y
qc.cz(0, 1)        # Controlled-Z
qc.swap(0, 1)      # SWAP qubits

# Controlled rotations
qc.crx(0.5, 0, 1)  # Controlled-RX
qc.cry(0.3, 0, 1)  # Controlled-RY
qc.crz(0.7, 0, 1)  # Controlled-RZ

# Three-qubit gates
qc.ccx(0, 1, 2)    # Toffoli (CCNOT)
qc.cswap(0, 1, 2)  # Fredkin (CSWAP)

# Multi-controlled gates
qc.mcx([0, 1, 2], 3)           # Multi-controlled-X
qc.mcp(0.5, [0, 1, 2], 3)      # Multi-controlled-phase
```

## Measurements

### Basic Measurement

```python
# Measure single qubit
qc.measure(0, 0)  # qubit 0 → classical bit 0

# Measure all qubits
qc.measure_all()

# Measure specific qubits to specific bits
qc.measure([0, 1, 2], [0, 1, 2])
```

### Conditional Measurement

```python
# Mid-circuit measurement for dynamic circuits
qc.h(0)
qc.measure(0, 0)

# Conditional operations based on measurement
with qc.if_test((0, 1)):  # if classical bit 0 == 1
    qc.x(1)
```

## Circuit Composition

### Combining Circuits

```python
# Create subcircuits
qc1 = QuantumCircuit(2)
qc1.h(0)
qc1.cx(0, 1)

qc2 = QuantumCircuit(2)
qc2.x(0)
qc2.z(1)

# Compose (in-place)
qc1.compose(qc2, inplace=True)

# Compose (return new circuit)
qc3 = qc1.compose(qc2)

# Compose on specific qubits
qc_main = QuantumCircuit(4)
qc_main.compose(qc1, qubits=[1, 2], inplace=True)
```

### Tensor Product

```python
# Combine circuits side-by-side
qc1 = QuantumCircuit(2)
qc2 = QuantumCircuit(3)

qc_combined = qc1.tensor(qc2)  # 5 qubits total
```

### Append vs Compose

```python
# append: Add instruction/gate
qc.append(gate, qubits)

# compose: Merge full circuits
qc.compose(other_circuit, qubits, inplace=True)
```

## Parametrized Circuits

### Using Parameters

```python
from qiskit.circuit import Parameter

# Define parameters
theta = Parameter('θ')
phi = Parameter('φ')

# Create parametrized circuit
qc = QuantumCircuit(2)
qc.ry(theta, 0)
qc.rz(phi, 1)
qc.cx(0, 1)

# Bind parameters
bound_circuit = qc.assign_parameters({theta: 0.5, phi: 1.2})

# Or with ordered list
bound_circuit = qc.assign_parameters([0.5, 1.2])
```

### Parameter Vectors

```python
from qiskit.circuit import ParameterVector

# Create parameter vector
params = ParameterVector('θ', 10)

qc = QuantumCircuit(5)
for i in range(5):
    qc.ry(params[i], i)

# Bind all parameters
values = [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
bound = qc.assign_parameters({params: values})
```

### Partial Binding

```python
# Bind subset of parameters
partial = qc.assign_parameters({theta: 0.5})  # phi remains free
```

## Barriers and Delays

### Barriers

Prevent transpiler optimizations across boundary.

```python
qc.h(0)
qc.cx(0, 1)
qc.barrier()  # No optimization across this line
qc.h(1)
qc.measure_all()
```

### Delays

Explicit idle time (for pulse-level control).

```python
from qiskit.circuit import Delay

# Add delay in dt units (backend time unit)
qc.delay(100, 0)  # 100 dt on qubit 0

# Delay in physical units
qc.delay(duration=500, unit='ns', qubits=0)  # 500 nanoseconds
```

## Reset Operations

```python
# Reset qubit to |0⟩
qc.reset(0)

# Reset multiple qubits
qc.reset([0, 1, 2])
```

## Custom Gates

### From Unitary Matrix

```python
from qiskit.quantum_info import Operator
import numpy as np

# Define custom unitary
U = np.array([
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 0, 1],
    [0, 0, 1, 0]
]) / np.sqrt(2)

# Create gate
from qiskit.extensions import UnitaryGate
custom_gate = UnitaryGate(U, label='MY_GATE')

# Add to circuit
qc.append(custom_gate, [0, 1])
```

### From Subcircuit

```python
# Define subcircuit
def bell_pair():
    qc = QuantumCircuit(2, name='Bell')
    qc.h(0)
    qc.cx(0, 1)
    return qc

# Convert to gate
bell_gate = bell_pair().to_gate()

# Use in larger circuit
qc_main = QuantumCircuit(4)
qc_main.append(bell_gate, [0, 1])
qc_main.append(bell_gate, [2, 3])
```

### Controlled Gates

```python
# Make any gate controlled
gate = QuantumCircuit(1)
gate.h(0)

controlled_gate = gate.to_gate().control(num_ctrl_qubits=2)
qc.append(controlled_gate, [0, 1, 2])  # 2 controls, 1 target
```

## State Initialization

### Initialize to Statevector

```python
from qiskit.quantum_info import Statevector
import numpy as np

# Define state
state = Statevector([1/np.sqrt(2), 1/np.sqrt(2)])  # |+⟩

# Initialize circuit
qc = QuantumCircuit(1)
qc.initialize(state, 0)
```

### Prepare Basis States

```python
# Prepare |10⟩ (MSB first: qubit 1 = |1⟩, qubit 0 = |0⟩)
qc = QuantumCircuit(2)
qc.x(1)

# Or use initialize
from qiskit.quantum_info import Statevector
state = Statevector.from_label('10')
qc.initialize(state, [0, 1])
```

## Circuit Visualization

### Text Drawer

```python
print(qc.draw())

# ASCII art with barriers
print(qc.draw(output='text', fold=-1))
```

### Matplotlib

```python
qc.draw(output='mpl')  # Returns matplotlib Figure

# Customization
qc.draw(output='mpl', style='iqp', fold=20, scale=0.8)
```

### LaTeX

```python
qc.draw(output='latex')
qc.draw(output='latex_source')  # Get LaTeX code
```

## Circuit Properties

```python
# Number of qubits
print(qc.num_qubits)

# Number of classical bits
print(qc.num_clbits)

# Circuit depth
print(qc.depth())

# Gate count
print(qc.size())

# Count operations
print(qc.count_ops())

# Qubit/clbit mapping
print(qc.qubits)
print(qc.clbits)

# Parameters
print(qc.parameters)

# Check if circuit is parametrized
print(len(qc.parameters) > 0)
```

## Decomposition

```python
# Decompose to basis gates
decomposed = qc.decompose()

# Recursive decomposition
fully_decomposed = qc.decompose(reps=3)

# Decompose specific gates
qc.decompose(['h', 'cx'])
```

## Circuit Reversal

```python
# Reverse circuit (time-reversed)
reversed_qc = qc.reverse_ops()

# Inverse (dagger)
inverse_qc = qc.inverse()
```

## Copy and Deepcopy

```python
# Shallow copy (references same objects)
qc_copy = qc.copy()

# Deep copy (independent)
import copy
qc_deep = copy.deepcopy(qc)
```

## Exporting Circuits

### OpenQASM

```python
# Export to QASM 2.0
qasm_str = qc.qasm()

# Export to QASM 3.0
from qiskit import qasm3
qasm3_str = qasm3.dumps(qc)

# Save to file
with open('circuit.qasm', 'w') as f:
    f.write(qasm_str)
```

### QPY (Qiskit binary format)

```python
from qiskit import qpy

# Save
with open('circuit.qpy', 'wb') as f:
    qpy.dump(qc, f)

# Load
with open('circuit.qpy', 'rb') as f:
    circuits = qpy.load(f)
```

## Advanced Features

### Add Calibration

Add custom pulse-level gate definition.

```python
from qiskit.pulse import Schedule

# Define pulse schedule
sched = Schedule(...)

# Add calibration
qc.add_calibration('h', [0], sched)
```

### Circuit Metadata

```python
# Add metadata
qc.metadata = {
    'experiment': 'VQE_H2',
    'date': '2026-02-15',
    'author': 'Agent Cluster'
}

# Access metadata
print(qc.metadata)
```

### Global Phase

```python
# Set global phase
qc.global_phase = np.pi / 4

# Access global phase
print(qc.global_phase)
```

## Best Practices

1. **Use barriers sparingly**: Only when optimization must be prevented
2. **Name registers descriptively**: Aids debugging and visualization
3. **Parametrize for flexibility**: Enables parameter sweeps without circuit rebuilding
4. **Compose, don't reconstruct**: Reuse subcircuits via composition
5. **Measure at the end**: Avoid mid-circuit measurement unless using dynamic circuits
6. **Check depth before execution**: Deep circuits accumulate errors
7. **Decompose incrementally**: Verify gates after each decomposition step
8. **Document with metadata**: Track experiments and provenance

## Common Pitfalls

- **Measuring before final operations**: Measurement collapses state
- **Forgetting measure_all()**: Sampler requires measurements
- **Qubit indexing confusion**: Remember 0-indexing and LSB/MSB conventions
- **Over-decomposing**: Excessive decomposition increases gate count
- **Mixing register types**: Ensure quantum operations only on quantum registers
- **Barrier overuse**: Prevents beneficial optimizations
- **Not copying circuits**: Modifying circuits in-place affects references
