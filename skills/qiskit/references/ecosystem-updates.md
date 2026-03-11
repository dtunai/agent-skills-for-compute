# Qiskit Ecosystem and Recent Updates

## Qiskit SDK 2.3 (January 2026)

Source: [IBM Quantum Latest Updates](https://quantum.cloud.ibm.com/docs/en/guides/latest-updates)

### Major Features

**1. Expanded C API**
- Significant expansion for high-performance computing (HPC) users
- Boost performance for quantum workflows at scale
- Integration with HPC cluster environments

**2. PauliProductMeasurement Instruction**
- New instruction enabling joint projective measurement across multiple qubits
- Single operation for multi-qubit measurements
- Unlocks compilation to Pauli-based computation
- Critical for early fault-tolerant quantum computing (EFTQC)

**3. Enhanced Transpilation**
- Improved Clifford+T transpilation for EFTQC targets
- Early support for Pauli-based computation compilation
- More efficient transpilation pipelines for future QPUs
- Support for fault-tolerant architecture development

### Performance Improvements

- Faster circuit optimization for large-scale circuits
- More flexible tools for building transpilation workflows
- Reduced overhead for batch processing

## Qiskit Ecosystem Overview

Source: [Qiskit GitHub Organization](https://github.com/qiskit)

The Qiskit ecosystem comprises open-source projects interfacing with Qiskit core SDK, developed by IBM Quantum and the wider community.

### Core Packages

| Package | Purpose | GitHub |
|---------|---------|--------|
| **qiskit** | Core SDK (circuits, operators, primitives, transpiler) | [Qiskit/qiskit](https://github.com/Qiskit/qiskit) |
| **qiskit-ibm-runtime** | IBM Quantum hardware access, primitives | [Qiskit/qiskit-ibm-runtime](https://github.com/Qiskit/qiskit-ibm-runtime) |
| **qiskit-aer** | High-performance local simulators | [Qiskit/qiskit-aer](https://github.com/Qiskit/qiskit-aer) |

### Algorithm Libraries

| Package | Purpose |
|---------|---------|
| **qiskit-algorithms** | VQE, QAOA, Grover, Shor, amplitude estimation |
| **qiskit-optimization** | QUBO, Ising, constraint optimization |
| **qiskit-machine-learning** | Quantum kernels, neural networks, classifiers |
| **qiskit-nature** | Quantum chemistry, molecular simulation |
| **qiskit-finance** | Portfolio optimization, option pricing |

### Experimental & Tools

| Package | Purpose |
|---------|---------|
| **qiskit-experiments** | Device characterization, calibration experiments |
| **qiskit-dynamics** | Pulse-level simulation, time evolution |
| **qiskit-metal** | Quantum hardware design automation |
| **qiskit-terra** | (Deprecated - merged into qiskit core) |

### Provider Integrations

| Provider | Package | Backend Access |
|----------|---------|----------------|
| **IBM Quantum** | qiskit-ibm-runtime | IBM quantum processors |
| **IonQ** | qiskit-ionq | Trapped ion systems |
| **Rigetti** | qiskit-rigetti | Superconducting QPUs |
| **AWS Braket** | qiskit-braket-provider | AWS Braket backends |
| **Azure Quantum** | azure-quantum | Microsoft quantum stack |
| **Quantinuum** | qiskit-quantinuum-provider | QCCD ion trap processors |
| **AQT** | qiskit-aqt-provider | Alpine Quantum Technologies |

## Qiskit Functions (2026)

Source: [IBM Quantum Functions Blog](https://www.ibm.com/quantum/blog/functions-2026)

### Overview

Pre-built, cloud-native quantum workflows in the **Qiskit Functions Catalog** accelerating research and development.

### Available Functions

**Quantum Error Handling:**
- Error mitigation strategies
- Noise characterization
- ZNE (Zero-Noise Extrapolation)

**Partial Differential Equations:**
- Quantum PDE solvers
- Variational quantum algorithms for PDEs

**Chemistry Simulation:**
- Molecular ground state finding
- Electronic structure calculations
- VQE for quantum chemistry

**Optimization:**
- QAOA for combinatorial optimization
- Quantum-enhanced optimization routines

**Machine Learning:**
- Quantum kernels for classification
- Quantum neural networks

### Access Pattern

```python
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()

# List available functions
functions = service.functions()

# Execute function
result = service.run_function(
    function="circuit-function",
    inputs={"circuits": [qc], "backend": backend_name}
)
```

## Qiskit 1.0 Release Summary (February 2024)

Source: [Qiskit 1.0 Blog](https://www.ibm.com/quantum/blog/qiskit-1-0-release-summary)

### Major Changes

**1. Primitives V2**
- Introduced SamplerV2 and EstimatorV2
- Vectorized inputs for batch processing
- Replaced execute() function (now deprecated)

**2. ISA Circuits**
- Requirement for transpilation to Instruction Set Architecture (ISA)
- Explicit circuit layout management
- Observable layout mapping via `apply_layout()`

**3. Unified Qiskit Package**
- Merged qiskit-terra into core qiskit package
- Simplified installation: `pip install qiskit`

**4. Stable API**
- Commitment to semantic versioning
- Backwards compatibility guarantees
- Migration guides for legacy code

## Hardware Provider Ecosystem

### IBM Quantum Platform

- **127-qubit Eagle processors**
- **433-qubit Osprey processors**
- **1000+ qubit Condor (upcoming)**
- Quantum Compute services via Qiskit Runtime
- Cloud access through quantum.cloud.ibm.com

### Alternative Hardware

**IonQ:**
- Trapped ion quantum computers
- 29-qubit Aria system
- High gate fidelities

**Rigetti:**
- Superconducting QPUs
- Aspen-M series (80+ qubits)
- Quantum Cloud Services

**AWS Braket:**
- Multi-provider access (IonQ, Rigetti, Oxford Quantum Circuits)
- Managed Jupyter notebooks
- Hybrid job execution

**Quantinuum:**
- QCCD ion trap processors
- H-series systems
- Industry-leading gate fidelities

**Azure Quantum:**
- Provider-agnostic platform
- Integration with Quantinuum, IonQ, Rigetti
- Resource estimation tools

## Community Resources

### Official Channels

- **Documentation**: [quantum.cloud.ibm.com/docs](https://quantum.cloud.ibm.com/docs)
- **GitHub**: [github.com/Qiskit](https://github.com/Qiskit)
- **Slack**: Qiskit workspace
- **Stack Exchange**: Quantum Computing Stack Exchange

### Learning Resources

- **Qiskit Tutorials**: [github.com/Qiskit/qiskit-tutorials](https://github.com/Qiskit/qiskit-tutorials)
- **Qiskit Textbook**: Learn quantum computing with Python
- **IBM Quantum Learning**: quantum.cloud.ibm.com/learning
- **YouTube**: IBM Quantum channel

### Events

- **Qiskit Global Summer School**: Annual quantum computing bootcamp
- **IBM Quantum Summit**: Annual conference showcasing advances
- **Qiskit Hackathons**: Community coding events worldwide

## Version Compatibility

| Qiskit Version | Python Support | Key Features |
|----------------|----------------|--------------|
| **2.3** (Jan 2026) | 3.9-3.12 | C API, PauliProductMeasurement, EFTQC |
| **2.0** (2025) | 3.9-3.11 | Enhanced transpiler, improved primitives |
| **1.0** (Feb 2024) | 3.8-3.11 | SamplerV2, EstimatorV2, ISA circuits |
| **0.x** (legacy) | 3.7-3.10 | Deprecated execute(), pre-ISA |

## Migration Notes

### From Qiskit 0.x to 1.0+

**Replace execute() with primitives:**
```python
# Old (0.x)
from qiskit import execute
result = execute(circuit, backend, shots=1024).result()

# New (1.0+)
from qiskit.primitives import Sampler
sampler = Sampler()
result = sampler.run([circuit], shots=1024).result()
```

**Add ISA transpilation:**
```python
# Old (0.x) - backend handled transpilation
job = backend.run(circuit)

# New (1.0+) - explicit ISA transpilation required
from qiskit.transpiler import generate_preset_pass_manager
pm = generate_preset_pass_manager(backend=backend)
isa_circuit = pm.run(circuit)
job = sampler.run([isa_circuit])
```

**Map observables to layout:**
```python
# New (1.0+) - required for EstimatorV2
isa_observable = observable.apply_layout(isa_circuit.layout)
```

## Roadmap

### Near-term (2026)

- Further EFTQC compilation optimizations
- Enhanced error mitigation strategies
- Expanded Qiskit Functions catalog
- Improved multi-QPU orchestration

### Long-term

- Fault-tolerant quantum computing support
- Logical qubit primitives
- Error correction code integration
- Quantum-classical hybrid runtime

## License

All core Qiskit packages: **Apache License 2.0**
