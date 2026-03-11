# Qiskit Runtime Primitives Reference

Source: [IBM Quantum Documentation - Get Started with Primitives](https://quantum.cloud.ibm.com/docs/en/guides/get-started-with-primitives)

## Overview

Qiskit Runtime provides two primary primitives for quantum computation:
- **Estimator**: Calculates expectation values of quantum observables
- **Sampler**: Generates measurement bitstrings from quantum circuits

Both primitives include built-in error mitigation for IBM Quantum hardware execution.

## Primitive Unified Blocs (PUBs)

Both primitives accept input as **Primitive Unified Bloc (PUB)** tuples:

**Estimator PUB:**
```python
(circuit, observable, parameter_values)
```

**Sampler PUB:**
```python
(circuit, parameter_values)
```

## Estimator Workflow

### 1. Initialize Service
```python
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()
backend = service.least_busy(operational=True, simulator=False, min_num_qubits=127)
```

### 2. Create Circuit and Observable
```python
from qiskit.circuit.library import qaoa_ansatz
from qiskit.quantum_info import SparsePauliOp

observable = SparsePauliOp.from_sparse_list([
    ("Z", [0, 1], 0.25),
    ("X", [0, 1], -0.5)
], num_qubits=2)

circuit = qaoa_ansatz(observable, reps=2)
param_values = [0.1, 0.2, 0.3, 0.4]
```

### 3. Transpile to ISA
**CRITICAL**: Circuits must be transformed to instruction set architecture (ISA) format.

```python
from qiskit.transpiler import generate_preset_pass_manager

pm = generate_preset_pass_manager(optimization_level=1, backend=backend)
isa_circuit = pm.run(circuit)
isa_observable = observable.apply_layout(isa_circuit.layout)
```

### 4. Execute with EstimatorV2
```python
from qiskit_ibm_runtime import EstimatorV2 as Estimator

estimator = Estimator(mode=backend)
job = estimator.run([(isa_circuit, isa_observable, param_values)])
result = job.result()
expectation_values = result[0].data.evs
```

## Sampler Workflow

### 1-2. Initialize and Create Circuit
```python
from qiskit.circuit.library import efficient_su2
import numpy as np

circuit = efficient_su2(127, entanglement="linear")
circuit.measure_all()
param_values = np.random.rand(circuit.num_parameters)
```

### 3-4. Transpile and Execute
```python
from qiskit_ibm_runtime import SamplerV2 as Sampler

pm = generate_preset_pass_manager(optimization_level=1, backend=backend)
isa_circuit = pm.run(circuit)

sampler = Sampler(mode=backend)
job = sampler.run([(isa_circuit, param_values)])
result = job.result()
bitstrings = result[0].data.meas.get_bitstrings()
```

## Execution Modes

The `mode` parameter accepts three types:

| Mode | Type | Use Case |
|------|------|----------|
| `batch` | str | Fire-and-forget execution for single jobs |
| `session` | str | Persistent connection for multiple related jobs |
| `backend` | Backend object | Direct backend execution (local/simulator) |

### Session Example
```python
from qiskit_ibm_runtime import Session

with Session(backend=backend) as session:
    sampler = Sampler(mode=session)
    job1 = sampler.run([circuit1])
    job2 = sampler.run([circuit2])
```

## Backend Primitives

For provider-agnostic execution (works with any Qiskit provider):

```python
from qiskit.primitives import BackendEstimatorV2, BackendSamplerV2

estimator = BackendEstimatorV2(backend)
sampler = BackendSamplerV2(backend)
```

**Key differences:**
- Run locally without server-side error mitigation
- Require backends supporting the `memory` option
- Compatible with non-IBM providers

## Local Simulation

For statevector simulation (no noise):

```python
from qiskit.primitives import StatevectorSampler, StatevectorEstimator

sampler = StatevectorSampler()
estimator = StatevectorEstimator()

# Execute
job = sampler.run([circuit])
result = job.result()
```

## Error Mitigation

Configure error mitigation via runtime options:

```python
from qiskit_ibm_runtime import Options

options = Options()
options.resilience_level = 1  # 0=none, 1=light, 2=heavy
options.optimization_level = 3

estimator = Estimator(mode=backend, options=options)
```

**Resilience Levels:**
- **0**: No error mitigation
- **1**: Light mitigation (measurement error, gate twirling)
- **2**: Heavy mitigation (ZNE, PEC)

## Advanced Features

### Fractional Gates
Enable experimental fractional gate support:

```python
backend = service.backend("ibm_torino", use_fractional_gates=True)
```

### Vectorized Parameter Sweeps
Submit multiple parameter sets in single PUB:

```python
param_values = [[0.1, 0.2], [0.3, 0.4], [0.5, 0.6]]
job = estimator.run([(isa_circuit, isa_observable, param_values)])
```

### Observable Batching
Compute multiple observables in single job:

```python
observables = [obs1, obs2, obs3]
pubs = [(isa_circuit, obs, params) for obs in observables]
job = estimator.run(pubs)
```

## Version Requirements

- `qiskit~=2.3.0`
- `qiskit-ibm-runtime~=0.43.1`

## API Versions

- **V2 Primitives** (current): SamplerV2, EstimatorV2 with vectorized inputs
- **V1 Primitives** (deprecated): Legacy interface, scheduled for removal

Always use V2 primitives for new code.
