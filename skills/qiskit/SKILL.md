---
name: qiskit
description: Provides Qiskit quantum computing framework for building quantum circuits, executing on IBM Quantum hardware and simulators, using runtime primitives (Sampler, Estimator), transpiler optimization, error mitigation, quantum information science, and ecosystem tools including Qiskit Metal for quantum hardware design. Applies to tasks involving quantum circuit construction, quantum algorithms, variational methods (VQE, QAOA), quantum chemistry, quantum machine learning, hardware execution, device characterization, or superconducting qubit design.
license: MIT
metadata:
  author: Agent Cluster
  tags: qiskit, quantum-computing, ibm-quantum, quantum-circuits, primitives, sampler, estimator, transpiler, vqe, qaoa, quantum-algorithms, qiskit-metal, quantum-hardware-design, superconducting-qubits
---

# Qiskit

## Overview

Open-source SDK for quantum computing developed by IBM Quantum. Qiskit provides a modular framework for building quantum circuits, optimizing them through transpilation, executing on real quantum hardware and simulators via runtime primitives, and developing quantum algorithms across research domains including chemistry, machine learning, and optimization.

## Quick Pattern

**Incorrect — mixing frameworks or missing primitives:**

```python
# No provider context, no runtime primitives
from qiskit import QuantumCircuit
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
qc.measure_all()
# Missing backend and primitive execution
```

**Correct — Qiskit Runtime with primitives:**

```python
from qiskit import QuantumCircuit
from qiskit_ibm_runtime import QiskitRuntimeService, SamplerV2 as Sampler
from qiskit.transpiler import generate_preset_pass_manager

# Initialize runtime service
service = QiskitRuntimeService()
backend = service.least_busy(operational=True, simulator=False)

# Build circuit
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
qc.measure_all()

# Transpile to ISA
pm = generate_preset_pass_manager(optimization_level=1, backend=backend)
isa_circuit = pm.run(qc)

# Execute with Sampler
sampler = Sampler(mode=backend)
job = sampler.run([isa_circuit])
result = job.result()
bitstrings = result[0].data.meas.get_bitstrings()
print(bitstrings)
```

## Quick Command

```bash
# Install Qiskit
pip install qiskit

# Install IBM Runtime for hardware access
pip install qiskit-ibm-runtime

# Install full ecosystem (visualization, optimization, etc.)
pip install qiskit[all]

# Install Qiskit Metal for quantum hardware design
# (source installation required for v0.5+)
git clone https://github.com/qiskit-community/qiskit-metal.git
cd qiskit-metal
pip install -e .

# Run circuit with local simulator
python my_circuit.py

# Visualize circuit
python -c "from qiskit import QuantumCircuit; qc = QuantumCircuit(2); qc.h(0); qc.cx(0,1); print(qc.draw())"
```

## Quick Reference

### Core API

| Function | Purpose |
|----------|---------|
| `QuantumCircuit(n)` | Create quantum circuit with n qubits |
| `QuantumRegister(n)` | Allocate n-qubit quantum register |
| `ClassicalRegister(n)` | Allocate n-bit classical register |
| `circuit.measure_all()` | Add measurements to all qubits |
| `circuit.compose(other)` | Compose with another circuit |
| `circuit.decompose()` | Decompose into basis gates |
| `circuit.draw(output='mpl')` | Visualize circuit (matplotlib, text, latex) |
| `transpile(circuit, backend)` | Optimize circuit for target backend |
| `generate_preset_pass_manager()` | Create transpilation pipeline |
| `Operator(circuit)` | Convert circuit to operator matrix |
| `Statevector(circuit)` | Compute statevector from circuit |

### Quantum Gates

| Gate | Syntax | Description |
|------|--------|-------------|
| Hadamard | `qc.h(qubit)` | Superposition |
| Pauli-X | `qc.x(qubit)` | Bit flip (NOT) |
| Pauli-Y | `qc.y(qubit)` | Y rotation |
| Pauli-Z | `qc.z(qubit)` | Phase flip |
| CNOT | `qc.cx(control, target)` | Controlled-NOT |
| CZ | `qc.cz(control, target)` | Controlled-Z |
| SWAP | `qc.swap(q0, q1)` | Swap qubits |
| Toffoli | `qc.ccx(c1, c2, target)` | Controlled-controlled-NOT |
| RX | `qc.rx(theta, qubit)` | X-axis rotation |
| RY | `qc.ry(theta, qubit)` | Y-axis rotation |
| RZ | `qc.rz(phi, qubit)` | Z-axis rotation |
| Phase | `qc.p(phi, qubit)` | Phase gate |
| T | `qc.t(qubit)` | T gate (π/4) |
| S | `qc.s(qubit)` | S gate (π/2) |
| Measure | `qc.measure(qubit, cbit)` | Z-basis measurement |
| Barrier | `qc.barrier()` | Prevent optimization across |
| Reset | `qc.reset(qubit)` | Reset to |0⟩ |

### Runtime Primitives

| Primitive | Purpose | Input PUB |
|-----------|---------|-----------|
| `SamplerV2` | Sample measurement outcomes | `(circuit, param_values)` |
| `EstimatorV2` | Compute expectation values | `(circuit, observable, param_values)` |
| `BackendSamplerV2` | Provider-agnostic sampler | `(circuit, param_values)` |
| `BackendEstimatorV2` | Provider-agnostic estimator | `(circuit, observable, param_values)` |

**Primitive Execution Pattern:**
```python
# Sampler
sampler = SamplerV2(mode=backend)
job = sampler.run([(isa_circuit, param_values)])
result = job.result()
bitstrings = result[0].data.meas.get_bitstrings()

# Estimator
estimator = EstimatorV2(mode=backend)
job = estimator.run([(isa_circuit, isa_observable, param_values)])
result = job.result()
expectation_values = result[0].data.evs
```

### Execution Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `batch` | Fire-and-forget execution | Single job submission |
| `session` | Persistent connection | Multiple related jobs |
| `backend` | Direct backend object | Local/simulator execution |

### Transpiler

| Component | Purpose |
|-----------|---------|
| `generate_preset_pass_manager(level, backend)` | Create optimization pipeline (level 0-3) |
| `pm.run(circuit)` | Execute transpilation passes |
| `circuit.layout` | Get qubit mapping layout |
| `observable.apply_layout(layout)` | Map observable to ISA qubits |
| `PassManager(passes)` | Custom transpilation pipeline |
| `transpile(circuit, optimization_level=n)` | One-step transpile (convenience) |

### Quantum Information

| Class | Purpose |
|-------|---------|
| `Statevector(data)` | Statevector representation |
| `DensityMatrix(data)` | Density matrix representation |
| `Operator(data)` | Unitary/matrix operator |
| `SparsePauliOp(...)` | Sparse Pauli operator |
| `PauliList(...)` | List of Pauli strings |
| `Clifford(circuit)` | Clifford tableau |
| `random_statevector(dims)` | Generate random state |
| `partial_trace(state, qubits)` | Trace out qubits |
| `entropy(state)` | Von Neumann entropy |
| `entanglement_of_formation(state)` | Entanglement measure |

## Core Patterns

### 1. Local Simulation (Statevector)

```python
from qiskit import QuantumCircuit
from qiskit.primitives import StatevectorSampler, StatevectorEstimator
from qiskit.quantum_info import SparsePauliOp

# Build circuit
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
qc.measure_all()

# Sample with StatevectorSampler
sampler = StatevectorSampler()
job = sampler.run([qc])
result = job.result()
counts = result[0].data.meas.get_counts()

# Estimate expectation value
observable = SparsePauliOp("ZZ")
estimator = StatevectorEstimator()
job = estimator.run([(qc, observable)])
result = job.result()
expectation = result[0].data.evs
```

### 2. IBM Quantum Hardware Execution

```python
from qiskit_ibm_runtime import QiskitRuntimeService, SamplerV2
from qiskit.transpiler import generate_preset_pass_manager

# Initialize service (first time: save credentials)
# QiskitRuntimeService.save_account(channel="ibm_quantum", token="YOUR_TOKEN")
service = QiskitRuntimeService()

# Select backend
backend = service.least_busy(operational=True, simulator=False, min_num_qubits=127)

# Transpile to ISA
pm = generate_preset_pass_manager(optimization_level=3, backend=backend)
isa_circuit = pm.run(qc)

# Execute
sampler = SamplerV2(mode=backend)
job = sampler.run([isa_circuit], shots=1000)
result = job.result()
```

### 3. Parametrized Circuits (Variational Algorithms)

```python
from qiskit.circuit import Parameter
import numpy as np

# Create parametrized circuit
theta = Parameter('θ')
phi = Parameter('φ')

qc = QuantumCircuit(2)
qc.ry(theta, 0)
qc.ry(phi, 1)
qc.cx(0, 1)

# Bind parameters
param_values = [np.pi/4, np.pi/2]
bound_circuit = qc.assign_parameters({theta: param_values[0], phi: param_values[1]})

# Or use with primitives (pass param_values directly)
# job = sampler.run([(isa_circuit, param_values)])
```

### 4. Observable Construction (Hamiltonians)

```python
from qiskit.quantum_info import SparsePauliOp

# Define Hamiltonian
H = SparsePauliOp.from_list([
    ("II", 0.5),
    ("ZZ", 0.25),
    ("XX", -0.5),
    ("YY", -0.5)
])

# Or from sparse list
H = SparsePauliOp.from_sparse_list([
    ("Z", [0, 1], 0.25),
    ("X", [0, 1], -0.5)
], num_qubits=2)

# Apply layout after transpilation
isa_observable = H.apply_layout(isa_circuit.layout)
```

### 5. VQE (Variational Quantum Eigensolver)

```python
from qiskit.circuit.library import TwoLocal
from qiskit_algorithms import VQE
from qiskit_algorithms.optimizers import SLSQP
from qiskit.primitives import Estimator

# Define ansatz
ansatz = TwoLocal(num_qubits=2, rotation_blocks='ry', entanglement_blocks='cx')

# Define Hamiltonian
hamiltonian = SparsePauliOp.from_list([("ZZ", 1.0), ("XX", -0.5)])

# Run VQE
estimator = Estimator()
optimizer = SLSQP(maxiter=100)
vqe = VQE(estimator, ansatz, optimizer)
result = vqe.compute_minimum_eigenvalue(hamiltonian)

print(f"Eigenvalue: {result.eigenvalue}")
print(f"Optimal parameters: {result.optimal_parameters}")
```

### 6. QAOA (Quantum Approximate Optimization Algorithm)

```python
from qiskit.circuit.library import qaoa_ansatz
from qiskit_algorithms import QAOA
from qiskit_algorithms.optimizers import COBYLA

# Define cost Hamiltonian
cost_hamiltonian = SparsePauliOp.from_list([("ZIII", 1), ("IZII", 1), ("IIZI", 1)])

# Create QAOA circuit
p = 2  # number of layers
circuit = qaoa_ansatz(cost_hamiltonian, reps=p)

# Run QAOA
qaoa = QAOA(sampler=Sampler(), optimizer=COBYLA(), reps=p)
result = qaoa.compute_minimum_eigenvalue(cost_hamiltonian)
```

### 7. Custom Transpilation Pipeline

```python
from qiskit.transpiler import PassManager
from qiskit.transpiler.passes import (
    Optimize1qGatesDecomposition,
    CXCancellation,
    CommutativeCancellation
)

# Build custom pass manager
pm = PassManager([
    Optimize1qGatesDecomposition(),
    CXCancellation(),
    CommutativeCancellation()
])

# Run passes
optimized_circuit = pm.run(qc)
```

### 8. Error Mitigation

```python
from qiskit_ibm_runtime import EstimatorV2 as Estimator
from qiskit_ibm_runtime import Options

# Configure error mitigation
options = Options()
options.resilience_level = 1  # 0=none, 1=light, 2=heavy
options.optimization_level = 3

# Execute with mitigation
estimator = Estimator(mode=backend, options=options)
job = estimator.run([(isa_circuit, isa_observable)])
result = job.result()
```

## Qiskit Metal (Quantum Hardware Design)

### Overview

Open-source quantum EDA platform for designing superconducting quantum devices. Provides Python API + GUI for automating chip design, integrating with Ansys/GDS tools, and performing electromagnetic analysis.

### Installation

```bash
git clone https://github.com/qiskit-community/qiskit-metal.git
cd qiskit-metal
pip install -e .
```

### Quick Start

```python
from qiskit_metal import designs, MetalGUI
from qiskit_metal.qlibrary.qubits.transmon_pocket import TransmonPocket
from qiskit_metal.qlibrary.tlines.meandered import RouteMeander

# Create design
design = designs.DesignPlanar()
gui = MetalGUI(design)

# Add transmon qubit
qubit_opts = dict(
    pos_x='0mm', pos_y='0mm',
    connection_pads=dict(
        a=dict(loc_W=+1, loc_H=+1),
        b=dict(loc_W=-1, loc_H=-1)
    )
)
q1 = TransmonPocket(design, 'Q1', options=qubit_opts)

# Add routing
route_opts = dict(
    pin_inputs=dict(start_pin=dict(component='Q1', pin='a'),
                   end_pin=dict(component='Q1', pin='b')),
    total_length='6mm'
)
route = RouteMeander(design, 'route1', options=route_opts)

# Rebuild and visualize
gui.rebuild()
gui.autoscale()
```

### Core Components

| Component | Description |
|-----------|-------------|
| `TransmonPocket` | Pocket-style transmon qubit |
| `TransmonCross` | Cross-shaped transmon qubit |
| `TransmonInterdigitated` | Interdigitated capacitor transmon |
| `RouteMeander` | Meandered transmission line |
| `RouteStraight` | Straight CPW transmission line |
| `ResonatorLumped` | Lumped element resonator |
| `CapNInterdigital` | Interdigital capacitor |

### Renderers & Analysis

```python
from qiskit_metal.analyses.quantization import EPRanalysis
from qiskit_metal.renderers.renderer_gds import QGDSRenderer

# GDS Export
gds = QGDSRenderer(design)
gds.options['fabricate'] = True
gds.export_to_gds('my_chip.gds')

# EPR Analysis (requires Ansys)
epr = EPRanalysis(design, "hfss")
epr.sim.setup.max_passes = 15
epr.sim.run()
epr.get_frequencies()
epr.get_coupling_matrix()
```

## Ecosystem Packages

| Package | Purpose |
|---------|---------|
| `qiskit-aer` | High-performance local simulators |
| `qiskit-ibm-runtime` | IBM Quantum hardware access |
| `qiskit-algorithms` | VQE, QAOA, Grover, Shor algorithms |
| `qiskit-optimization` | Optimization problem solvers |
| `qiskit-machine-learning` | Quantum ML classifiers/regressors |
| `qiskit-nature` | Quantum chemistry/physics |
| `qiskit-experiments` | Device characterization |
| `qiskit-metal` | Quantum hardware design |

## Hardware Providers

| Provider | Package |
|----------|---------|
| IBM Quantum | `qiskit-ibm-runtime` |
| IonQ, Rigetti, AWS Braket, Azure Quantum, Quantinuum | Provider-specific packages available |

## Best Practices

1. **Always transpile circuits to ISA format** before hardware execution
2. **Use runtime primitives** (SamplerV2, EstimatorV2) instead of deprecated `execute()`
3. **Apply observable layout** after transpilation: `observable.apply_layout(circuit.layout)`
4. **Set optimization_level=3** for hardware to minimize gate count
5. **Use sessions** for multiple related jobs to reduce queue time
6. **Enable error mitigation** for noisy hardware via `options.resilience_level`
7. **Vectorize parameter sweeps** in primitives for batch efficiency
8. **Prefer SparsePauliOp** over dense matrices for Hamiltonians
9. **Use barrier()** to prevent unwanted transpiler optimizations
10. **Save runtime credentials** once: `QiskitRuntimeService.save_account(token=...)`

## Common Pitfalls

- **Missing ISA transpilation**: Runtime primitives require ISA circuits
- **Ignoring circuit layout**: Observables must be mapped with `apply_layout()`
- **Using deprecated execute()**: Replaced by primitives in Qiskit 1.0+
- **Not binding parameters**: Parametrized circuits need values via PUB or `assign_parameters()`
- **Mixing simulator/hardware**: Check `backend.configuration()` before execution
- **Exceeding qubit limits**: Verify `backend.configuration().n_qubits` before submission
- **Forgetting measurement**: Sampler requires `measure_all()` or explicit measurements
- **Dense Hamiltonians**: Use `SparsePauliOp` to avoid memory/performance issues

## Version Notes

- **Qiskit 1.0+ (Feb 2024)**: Introduced SamplerV2/EstimatorV2, deprecated execute()
- **Qiskit SDK 2.3 (Jan 2026)**: Expanded C API, PauliProductMeasurement, Clifford+T transpilation
- **Qiskit Metal 0.5**: Breaking changes from pre-0.5; source install required
- **IBM Runtime 0.43+**: Required for latest primitive features

## References

- Official docs: [quantum.cloud.ibm.com/docs](https://quantum.cloud.ibm.com/docs)
- Qiskit GitHub: [github.com/Qiskit/qiskit](https://github.com/Qiskit/qiskit)
- Qiskit Metal: [qiskit-community.github.io/qiskit-metal](https://qiskit-community.github.io/qiskit-metal)
- Primitives guide: [quantum.cloud.ibm.com/docs/en/guides/get-started-with-primitives](https://quantum.cloud.ibm.com/docs/en/guides/get-started-with-primitives)
- PyPI: [pypi.org/project/qiskit](https://pypi.org/project/qiskit)
