# Qiskit Fractional Gates Reference

Sources:
- [Fractional Gates Guide](https://quantum.cloud.ibm.com/docs/en/guides/fractional-gates)
- [Fractional Gates Blog](https://www.ibm.com/quantum/blog/fractional-gates)
- [Migrate from Pulse to Fractional Gates](https://quantum.cloud.ibm.com/docs/en/guides/pulse-migration)

## Overview

Fractional gates enable **direct execution of arbitrary-angle rotations** on IBM Quantum Heron processors without decomposition into basis gates. This reduces circuit depth and error rates.

## Supported Gates

### RX(θ) - Single-Qubit Rotation

**Range:** Any angle θ

**Benefit:** Up to 2× reduction in duration and error compared to decomposed rotations.

**Standard Decomposition (Without Fractional Gates):**
```
RX(θ) → SX - RZ(θ) - SX†
```
(3 gates)

**With Fractional Gates:**
```
RX(θ) → RX(θ)
```
(1 gate, direct pulse)

### RZZ(θ) - Two-Qubit Entangling Rotation

**Range:** 0 < θ ≤ π/2

**Benefit:** Direct execution avoids multi-gate decomposition, reducing entangling gate overhead.

**Standard Decomposition (Without Fractional Gates):**
```
RZZ(θ) → CX - RZ(θ) - CX
```
(Multiple CNOTs required)

**With Fractional Gates:**
```
RZZ(θ) → RZZ(θ)
```
(1 gate, direct pulse)

## Enabling Fractional Gates

### Backend Configuration

```python
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()

# Load backend WITH fractional gates
backend = service.backend('ibm_torino', use_fractional_gates=True)

# Check if fractional gates are enabled
print(backend.target.operation_names)  # Should include 'rx', 'rzz'
```

### Supported Backends

As of 2026, fractional gates are available on:
- **ibm_torino** (133 qubits, Heron r2)
- **ibm_kyiv** (127 qubits, Heron r2)
- Other Heron r2 processors

Check backend documentation for current availability.

## Transpilation with Fractional Gates

### Automatic Transpilation

```python
from qiskit import QuantumCircuit
from qiskit.transpiler import generate_preset_pass_manager

# Create circuit with arbitrary rotations
qc = QuantumCircuit(2)
qc.rx(0.7, 0)
qc.ry(1.2, 1)
qc.rzz(0.5, 0, 1)

# Transpile with fractional gates enabled
backend = service.backend('ibm_torino', use_fractional_gates=True)
pm = generate_preset_pass_manager(backend=backend, optimization_level=3)

isa_circuit = pm.run(qc)

# Result: RX(0.7) and RZZ(0.5) preserved as fractional gates
```

### RZZ Angle Constraint: FoldRzzAngle Pass

**Constraint:** All RZZ angles must satisfy 0 < θ ≤ π/2.

**Automatic Correction:** Qiskit applies `FoldRzzAngle` pass when transpiling to fractional-gate backends.

```python
from qiskit.transpiler.passes import FoldRzzAngle

# This pass is automatically applied
# Transforms RZZ(π) → RZZ(π/2) + phase corrections
```

**Manual Application:**
```python
from qiskit.transpiler import PassManager
from qiskit.transpiler.passes import FoldRzzAngle

pm = PassManager([FoldRzzAngle()])
corrected_circuit = pm.run(qc)
```

**Example Transformation:**
```python
# Original circuit
qc.rzz(0.8 * np.pi, 0, 1)  # θ = 0.8π > π/2

# After FoldRzzAngle
# Decomposes to: RZZ(0.2π) + phase corrections
# Since RZZ(0.8π) = RZZ(π - 0.2π) with basis change
```

## Performance Benefits

### Circuit Depth Reduction

**Example: Variational Algorithm**

```python
# Standard basis (without fractional gates)
# RX(θ) → SX + RZ(θ) + SX†
# Depth: 3 gates per rotation

# With fractional gates
# RX(θ) → RX(θ)
# Depth: 1 gate per rotation

# For 100 rotations:
# Standard: 300 gates
# Fractional: 100 gates
# Reduction: 66%
```

### Error Rate Improvement

**Single-qubit rotations:** Up to **2× error reduction**

**Two-qubit entangling:** Reduces decomposition overhead, lowering cumulative error.

### Gate Duration

**RX(θ)**: ~60 ns (vs ~120 ns for SX + RZ + SX†)

**RZZ(θ)**: Direct pulse execution eliminates multiple CNOT decomposition.

## Error Rates

### Current Status

"The error value reported in the Target of a backend with fractional gates enabled is just a copy of the non-fractional gate's counterpart."

**Implications:**
- No formal calibration data yet available
- Assumed comparable to basis gate equivalents
- Use with understanding that error characterization is ongoing

### Benchmarking

```python
# Compare circuit fidelity with/without fractional gates
backend_frac = service.backend('ibm_torino', use_fractional_gates=True)
backend_std = service.backend('ibm_torino', use_fractional_gates=False)

# Run same circuit on both
# Compare fidelity via quantum volume or mirror benchmarks
```

## Parametrized Circuits

### Runtime Parameter Binding

**CRITICAL:** Parametrized RZZ gates assume runtime values fall within 0 < θ ≤ π/2.

```python
from qiskit.circuit import Parameter

theta = Parameter('θ')
qc = QuantumCircuit(2)
qc.rzz(theta, 0, 1)

# Transpile (assumes theta will be in valid range)
backend = service.backend('ibm_torino', use_fractional_gates=True)
pm = generate_preset_pass_manager(backend=backend)
isa_circuit = pm.run(qc)

# Bind parameters
# VALID
bound = isa_circuit.assign_parameters({theta: 0.3})

# INVALID: Will fail at runtime!
bound = isa_circuit.assign_parameters({theta: 2.0})  # > π/2
```

### Workaround for Out-of-Range Parameters

```python
import numpy as np

def fold_rzz_angle(theta):
    """Manually fold RZZ angle to valid range."""
    if theta > np.pi / 2:
        # Apply identity: RZZ(θ) = -RZZ(π - θ) + phase
        # This is simplified; actual folding involves basis changes
        return np.pi - theta
    return theta

# Apply folding before binding
safe_theta = fold_rzz_angle(2.0)
bound = isa_circuit.assign_parameters({theta: safe_theta})
```

## Limitations

### Unsupported Features

Fractional gates **cannot** be used with:

1. **Dynamic Circuits**
   - `if_test` conditions incompatible with fractional gates

2. **Pauli Twirling**
   - Gate-level twirling not supported
   - Measurement twirling with TREX is supported

3. **Probabilistic Error Cancellation (PEC)**
   - Requires gate decomposition

4. **Zero-Noise Extrapolation (ZNE) with Probabilistic Amplification**
   - Unitary folding method not compatible
   - Other ZNE methods may work

### Error Mitigation Compatibility

| Technique | Compatible |
|-----------|-----------|
| Readout mitigation | ✅ Yes |
| Dynamical decoupling | ✅ Yes |
| Measurement twirling (TREX) | ✅ Yes |
| Gate twirling | ❌ No |
| ZNE (noise scaling) | ⚠️ Partial |
| PEC | ❌ No |

## Use Cases

### 1. Variational Quantum Eigensolver (VQE)

Fractional gates excel in VQE due to many parametrized rotations.

```python
from qiskit_algorithms import VQE
from qiskit.circuit.library import RealAmplitudes

# Ansatz with many RY rotations
ansatz = RealAmplitudes(num_qubits=4, reps=3)

# Enable fractional gates for transpilation
backend = service.backend('ibm_torino', use_fractional_gates=True)
pm = generate_preset_pass_manager(backend=backend, optimization_level=3)

isa_ansatz = pm.run(ansatz)

# Reduced depth → better VQE performance
```

### 2. Hamiltonian Time Evolution

Direct RZZ execution simplifies Trotter decomposition.

```python
from qiskit.circuit.library import PauliEvolutionGate
from qiskit.quantum_info import SparsePauliOp

H = SparsePauliOp.from_list([('ZZ', 1.0), ('XX', 0.5)])

evolution = PauliEvolutionGate(H, time=0.1, synthesis='lie_trotter')

# Transpile with fractional gates
# RZZ gates in Trotter steps execute directly
```

### 3. Quantum Machine Learning

Feature maps with many rotations benefit from reduced depth.

```python
from qiskit.circuit.library import ZZFeatureMap

feature_map = ZZFeatureMap(feature_dimension=4, reps=2)

# Fractional gates reduce encoding circuit depth
```

## Migration from Pulse

Fractional gates replace custom pulse sequences for arbitrary rotations.

### Before (Pulse-Level)

```python
from qiskit import pulse

# Define custom pulse for RX(θ)
with pulse.build(backend) as rx_schedule:
    pulse.play(pulse.Gaussian(...), pulse.DriveChannel(0))
    # Complex pulse shaping

qc.append(rx_schedule, [0])
```

### After (Fractional Gates)

```python
# Simple gate-level interface
qc.rx(theta, 0)

# Transpiler handles pulse generation
```

**Benefits:**
- Simpler API
- Automatic calibration
- Cross-backend portability

## Experimental Status

**Current Classification:** Experimental feature

**Future Changes:**
- `use_fractional_gates` flag behavior may change
- Calibration and error characterization ongoing
- Feature set expansion planned

## Best Practices

1. **Enable for rotation-heavy circuits**: VQE, QAOA, Hamiltonian simulation
2. **Check angle constraints**: Ensure RZZ θ ∈ (0, π/2] for parametrized circuits
3. **Benchmark against standard basis**: Verify performance gains for your application
4. **Avoid with dynamic circuits**: Not supported
5. **Use with compatible error mitigation**: DD, readout mitigation, TREX
6. **Monitor backend compatibility**: Check documentation for supported backends
7. **Prefer optimization_level=3**: Maximizes fractional gate usage

## Common Pitfalls

- **RZZ angle violations**: Parametrized circuits with θ > π/2 fail at runtime
- **Combining with unsupported features**: Dynamic circuits, gate twirling cause errors
- **Assuming calibrated error rates**: Formal error data not yet published
- **Using on non-Heron backends**: Only specific backends support fractional gates
- **Forgetting to set flag**: Must use `use_fractional_gates=True` when loading backend

## Example: Complete VQE Workflow

```python
from qiskit import QuantumCircuit
from qiskit.circuit.library import TwoLocal
from qiskit_algorithms import VQE
from qiskit_algorithms.optimizers import SLSQP
from qiskit_ibm_runtime import EstimatorV2, QiskitRuntimeService
from qiskit.quantum_info import SparsePauliOp

# Initialize service
service = QiskitRuntimeService()
backend = service.backend('ibm_torino', use_fractional_gates=True)

# Define Hamiltonian
H = SparsePauliOp.from_list([("ZZ", 1.0), ("XX", -0.5)])

# Create ansatz
ansatz = TwoLocal(num_qubits=2, rotation_blocks=['rx', 'ry'],
                  entanglement_blocks='rzz', reps=3)

# Transpile
pm = generate_preset_pass_manager(backend=backend, optimization_level=3)
isa_ansatz = pm.run(ansatz)
isa_H = H.apply_layout(isa_ansatz.layout)

# Run VQE
estimator = EstimatorV2(mode=backend)
optimizer = SLSQP(maxiter=100)

vqe = VQE(estimator, isa_ansatz, optimizer)
result = vqe.compute_minimum_eigenvalue(isa_H)

print(f"Ground state energy: {result.eigenvalue}")
print(f"Circuit depth: {isa_ansatz.depth()}")  # Reduced with fractional gates
```
