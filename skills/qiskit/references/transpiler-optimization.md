# Qiskit Transpiler and Optimization Reference

Sources:
- [IBM Quantum Documentation - Transpiler](https://docs.quantum.ibm.com/api/qiskit/transpiler)
- [Introduction to Transpilation](https://qiskit.org/documentation/tutorials/circuits_advanced/04_transpiler_passes_and_passmanager.html)

## Overview

The Qiskit transpiler transforms abstract quantum circuits into optimized, hardware-executable ISA (Instruction Set Architecture) circuits through a multi-stage pipeline.

## Transpiler Stages

### Six Core Stages

1. **Init**: Unroll custom instructions, convert to 1- and 2-qubit gates
2. **Layout**: Map virtual qubits to physical qubits
3. **Routing**: Insert SWAP gates for non-adjacent qubit operations
4. **Translation**: Convert gates to backend basis set
5. **Optimization**: Reduce gate count and circuit depth
6. **Scheduling**: Assign gate timing for pulse-level execution

### Stage Flow

```
Circuit → Init → Layout → Routing → Translation → Optimization → Scheduling → ISA Circuit
```

## Preset Pass Managers

### Quick Usage

```python
from qiskit.transpiler import generate_preset_pass_manager

pm = generate_preset_pass_manager(
    optimization_level=3,
    backend=backend,
    seed_transpiler=42
)

isa_circuit = pm.run(circuit)
```

### Optimization Levels

| Level | Passes | Speed | Quality | Use Case |
|-------|--------|-------|---------|----------|
| **0** | Minimal (layout, basis translation) | Fastest | Lowest | Debugging, simulation |
| **1** | Light optimization | Fast | Medium | Quick hardware tests |
| **2** | Medium optimization | Moderate | Good | General-purpose |
| **3** | Aggressive optimization | Slow | Best | Production, benchmarking |

## Layout Passes

### Purpose

Map virtual circuit qubits to physical backend qubits, considering:
- Qubit connectivity (coupling map)
- Gate error rates
- Readout fidelities

### Key Layout Passes

**TrivialLayout:**
```python
from qiskit.transpiler.passes import TrivialLayout

# Maps qubit i → physical qubit i
# Fast but ignores hardware topology
```

**DenseLayout:**
```python
from qiskit.transpiler.passes import DenseLayout

# Uses highest-connectivity qubits
# Minimizes SWAP overhead
```

**SabreLayout:**
```python
from qiskit.transpiler.passes import SabreLayout

# Heuristic layout+routing optimization
# Default for level 3
# Best for complex circuits
```

**NoiseAdaptiveLayout:**
```python
from qiskit.transpiler.passes import NoiseAdaptiveLayout

# Selects qubits by error rates
# Requires backend properties
# Best for noisy hardware
```

### Layout Inspection

```python
# After transpilation
print(isa_circuit.layout)

# Maps virtual → physical
for virtual, physical in isa_circuit.layout.get_virtual_bits().items():
    print(f"Virtual {virtual} → Physical {physical}")
```

## Routing Passes

### Purpose

Insert SWAP gates to enable operations on non-adjacent qubits according to coupling map.

### Key Routing Passes

**SabreSwap:**
```python
from qiskit.transpiler.passes import SabreSwap

# Heuristic SWAP insertion
# Default for levels 2-3
# Greedy lookahead optimization
```

**StochasticSwap:**
```python
from qiskit.transpiler.passes import StochasticSwap

# Randomized SWAP trials
# Slower but explores more solutions
```

**BasicSwap:**
```python
from qiskit.transpiler.passes import BasicSwap

# Simple SWAP insertion
# Fastest, least optimal
```

## Optimization Passes

### Single-Qubit Optimization

**Optimize1qGates:**
```python
from qiskit.transpiler.passes import Optimize1qGates

# Combines chains of single-qubit gates
# Reduces u1, u2, u3 to minimal form
# Critical for reducing gate count
```

**Optimize1qGatesDecomposition:**
```python
from qiskit.transpiler.passes import Optimize1qGatesDecomposition

# Optimizes to specific basis (e.g., RZ, SX)
# Hardware-aware decomposition
```

### Two-Qubit Optimization

**CXCancellation:**
```python
from qiskit.transpiler.passes import CXCancellation

# Cancels adjacent CNOT pairs
# Exploits CNOT² = I
```

**CommutativeCancellation:**
```python
from qiskit.transpiler.passes import CommutativeCancellation

# Cancels commuting self-inverse gates
# Works for CNOT, SWAP, Pauli gates
```

**InverseCancellation:**
```python
from qiskit.transpiler.passes import InverseCancellation

# Removes gate + gate.inverse() pairs
# Example: RZ(θ) + RZ(-θ) → I
```

### Circuit Consolidation

**ConsolidateBlocks:**
```python
from qiskit.transpiler.passes import ConsolidateBlocks

# Combines gate sequences into unitaries
# Enables further optimization
```

**Collect2qBlocks:**
```python
from qiskit.transpiler.passes import Collect2qBlocks

# Groups two-qubit gate blocks
# Prepares for consolidated synthesis
```

**UnitarySynthesis:**
```python
from qiskit.transpiler.passes import UnitarySynthesis

# Decomposes arbitrary unitaries
# Optimizes to basis gates
```

## Synthesis Passes

### Basis Translation

**BasisTranslator:**
```python
from qiskit.transpiler.passes import BasisTranslator
from qiskit.circuit.equivalence_library import SessionEquivalenceLibrary

# Symbolic translation to target basis
# Uses equivalence library
basis_translator = BasisTranslator(
    SessionEquivalenceLibrary,
    target_basis=['rx', 'ry', 'rz', 'cx']
)
```

**UnrollCustomDefinitions:**
```python
from qiskit.transpiler.passes import UnrollCustomDefinitions

# Expands custom gate definitions
# Converts to standard gates
```

### Advanced Synthesis

**SolovayKitaev:**
```python
from qiskit.transpiler.passes import SolovayKitaev

# Approximates arbitrary single-qubit gates
# Uses discrete gate set (e.g., Clifford+T)
# Critical for fault-tolerant compilation
```

**KAKDecomposition:**
```python
from qiskit.transpiler.passes import KAKDecomposition

# Optimal two-qubit gate decomposition
# Minimal CNOT count
```

## Scheduling Passes

### Time-Aware Scheduling

**ALAPSchedule:**
```python
from qiskit.transpiler.passes import ALAPSchedule

# As-Late-As-Possible scheduling
# Pushes gates toward end
# Minimizes idle time before measurement
```

**ASAPSchedule:**
```python
from qiskit.transpiler.passes import ASAPSchedule

# As-Soon-As-Possible scheduling
# Executes gates immediately when ready
```

**DynamicalDecoupling:**
```python
from qiskit.transpiler.passes import DynamicalDecoupling
from qiskit.circuit.library import XGate

# Inserts DD sequences on idle qubits
# Suppresses decoherence
dd_sequence = [XGate(), XGate()]
dd_pass = DynamicalDecoupling(
    durations=backend.instruction_durations,
    dd_sequence=dd_sequence
)
```

## Custom Pass Managers

### Building Custom Pipeline

```python
from qiskit.transpiler import PassManager
from qiskit.transpiler.passes import (
    TrivialLayout,
    SabreSwap,
    Optimize1qGates,
    CXCancellation,
    UnitarySynthesis
)

# Create custom pipeline
pm = PassManager([
    TrivialLayout(coupling_map),
    SabreSwap(coupling_map),
    Optimize1qGates(),
    CXCancellation(),
    UnitarySynthesis(basis_gates=['rx', 'ry', 'rz', 'cx'])
])

optimized_circuit = pm.run(circuit)
```

### Conditional Passes

```python
from qiskit.transpiler import ConditionalController

# Run pass only if condition met
conditional_optimization = ConditionalController(
    [Optimize1qGates()],
    condition=lambda property_set: property_set['depth'] > 10
)
```

### Iterative Passes

```python
from qiskit.transpiler import FlowController

# Repeat passes until convergence
flow = FlowController.controller_factory(
    [Optimize1qGates(), CXCancellation()],
    options={'max_iteration': 10}
)
```

## Analysis Passes

### Circuit Metrics

**Depth:**
```python
from qiskit.transpiler.passes import Depth

depth_pass = Depth()
depth_pass(circuit)
circuit_depth = depth_pass.property_set['depth']
```

**Size:**
```python
from qiskit.transpiler.passes import Size

size_pass = Size()
size_pass(circuit)
gate_count = size_pass.property_set['size']
```

**CountOps:**
```python
from qiskit.transpiler.passes import CountOps

count_pass = CountOps()
count_pass(circuit)
ops_dict = count_pass.property_set['count_ops']
```

## Target-Aware Transpilation

### Using Target Object

```python
from qiskit.transpiler import Target

# Target encodes backend capabilities
# Replaces basis_gates + coupling_map
isa_circuit = pm.run(circuit)

# Access via backend
target = backend.target
pm = generate_preset_pass_manager(target=target, optimization_level=3)
```

### Hardware Properties

```python
# Error-aware optimization
properties = backend.properties()
gate_errors = properties.gate_error('cx', [0, 1])
readout_error = properties.readout_error(0)
```

## Clifford+T Compilation

### For Fault-Tolerant QPUs

```python
from qiskit.transpiler.passes import SolovayKitaev

# Compile to discrete gate set
sk = SolovayKitaev(recursion_degree=3)
clifford_t_circuit = sk(circuit)

# Basis: {H, S, T, CNOT}
```

## Pauli-Based Compilation

### New in Qiskit 2.3

```python
from qiskit.transpiler.passes import SynthesizePauliEvolution

# Compile Pauli rotations efficiently
# Supports PauliProductMeasurement
pauli_pass = SynthesizePauliEvolution()
```

## Best Practices

1. **Always transpile before hardware execution** (ISA requirement)
2. **Use optimization_level=3 for production** (best fidelity)
3. **Set seed_transpiler for reproducibility**
4. **Inspect layout** to understand qubit mapping
5. **Compare circuit metrics** before/after optimization
6. **Use target-aware transpilation** when available
7. **Enable dynamical decoupling** for long idle times
8. **Cache transpiled circuits** to avoid redundant computation
9. **Profile transpilation time** for large circuits
10. **Verify basis gates** match backend capabilities

## Common Pitfalls

- **Forgetting to transpile**: Runtime requires ISA circuits
- **Over-optimizing**: Level 3 can be slow for large circuits
- **Ignoring coupling map**: Invalid layouts cause routing overhead
- **Mismatched basis gates**: Translation failures
- **Not applying observable layout**: Estimator requires mapped observables
- **Excessive SWAP gates**: Poor layout choice increases depth
- **Skipping analysis passes**: Unaware of circuit metrics
