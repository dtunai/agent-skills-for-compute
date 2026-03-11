---
name: cuda-quantum
description: Provides CUDA-Q hybrid quantum-classical programming patterns for GPU-accelerated quantum simulation, circuit construction, variational algorithms, hardware backend configuration, dynamics simulation, and CUDA-QX extension libraries (solvers for molecular chemistry, QEC for error correction). Applies to tasks involving quantum kernels, qubit operations, cuQuantum simulation, VQE/QAOA optimization, multi-GPU quantum workflows, noise modeling, dynamics/time-evolution, molecular Hamiltonians, or quantum error correction.
license: MIT
metadata:
  author: Agent Cluster
  tags: cuda-q, quantum-computing, cuquantum, vqe, qaoa, gpu-simulation, nvidia, cudaqx, qec, molecular, dynamics
---

# CUDA-Q (CUDA Quantum)

## Overview

Programming model and toolchain for hybrid quantum-classical computing on NVIDIA GPUs. CUDA-Q provides Python and C++ APIs for building quantum circuits, running GPU-accelerated simulations, executing variational algorithms, and targeting real quantum hardware — all through a unified kernel-based programming model.

## Quick Pattern

**Incorrect — raw gate operations without kernel:**

```python
# No kernel decorator, no GPU acceleration
from qiskit import QuantumCircuit
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
```

**Correct — CUDA-Q kernel with GPU simulation:**

```python
import cudaq

@cudaq.kernel
def bell_pair():
    qubits = cudaq.qvector(2)
    h(qubits[0])
    x.ctrl(qubits[0], qubits[1])
    mz(qubits)

result = cudaq.sample(bell_pair, shots_count=1000)
print(result)
```

## Quick Command

```bash
# Install CUDA-Q
pip install cuda-quantum

# Install CUDA-QX extensions
pip install cudaq-solvers cudaq-qec

# Run with GPU backend
python my_circuit.py --target nvidia

# Run with multi-GPU
mpiexec -np 4 python my_circuit.py --target nvidia --target-option mgpu

# Draw circuit
python -c "import cudaq; print(cudaq.draw(my_kernel, *args))"
```

## Quick Reference

### Core API

| Function | Purpose |
|----------|---------|
| `@cudaq.kernel` | Decorate function as quantum kernel (JIT compiled) |
| `cudaq.qvector(N)` | Allocate N qubits |
| `cudaq.qubit()` | Allocate single qubit |
| `cudaq.sample(kernel, *args)` | Sample measurement outcomes |
| `cudaq.run(kernel, *args)` | Execute kernel with return values |
| `cudaq.observe(kernel, hamiltonian, *args)` | Compute expectation value |
| `cudaq.evolve(hamiltonian, dims, schedule, state)` | Dynamics time evolution |
| `cudaq.vqe(kernel, hamiltonian, optimizer)` | Run VQE optimization |
| `cudaq.get_state(kernel, *args)` | Get full statevector |
| `cudaq.draw(kernel, *args)` | ASCII circuit visualization |
| `cudaq.translate(kernel, format)` | Translate to OpenQASM |
| `cudaq.set_target(name)` | Select simulation/hardware backend |

### Quantum Gates

| Gate | Syntax | Description |
|------|--------|-------------|
| Hadamard | `h(qubit)` | Superposition |
| Pauli-X | `x(qubit)` | Bit flip |
| Pauli-Y | `y(qubit)` | Y rotation |
| Pauli-Z | `z(qubit)` | Phase flip |
| CNOT | `x.ctrl(control, target)` | Controlled-NOT |
| RX | `rx(angle, qubit)` | X-axis rotation |
| RY | `ry(angle, qubit)` | Y-axis rotation |
| RZ | `rz(angle, qubit)` | Z-axis rotation |
| T | `t(qubit)` | T gate |
| S | `s(qubit)` | S gate |
| SWAP | `swap(q0, q1)` | Swap qubits |
| Measure | `mz(qubit)` | Z-basis measurement |
| Adjoint | `t.adj(qubit)` | Inverse gate |

### Simulation Backends

| Target | Type | GPU | Description |
|--------|------|-----|-------------|
| `qpp-cpu` | State vector | No | CPU-only Q++ simulator |
| `nvidia` | State vector | Yes | cuStateVec GPU-accelerated |
| `nvidia-fp64` | State vector | Yes | Double precision GPU |
| `nvidia-mgpu` | State vector | Multi | Distributed across GPUs via MPI |
| `nvidia-mqpu` | Multi-QPU | Multi | Parallel QPU simulation |
| `tensornet` | Tensor network | Yes | Exact tensor contraction |
| `tensornet-mps` | MPS | Yes | Matrix product state approximation |
| `density-matrix-cpu` | Density matrix | No | Noisy simulation support |
| `dynamics` | Time evolution | Yes | Hamiltonian dynamics (Schrödinger/Lindblad) |
| `stim` | Stabilizer | No | Fast Clifford circuits, QEC, Pauli noise |
| `fermioniq` | Tensor network | Cloud | Third-party GPU tensor network emulator |
| `orca-photonics` | Photonic | No | Bosonic/photonic qudit simulation |

### Hardware Backends

| Provider | Target | Type |
|----------|--------|------|
| IonQ | `ionq` | Trapped ion |
| Quantinuum | `quantinuum` | Trapped ion |
| IQM | `iqm` | Superconducting |
| OQC | `oqc` | Superconducting |
| ORCA | `orca` | Photonic |
| Infleqtion | `infleqtion` | Neutral atom |
| Pasqal | `pasqal` | Neutral atom |
| QuEra | `quera` | Neutral atom |
| Anyon | `anyon` | Superconducting |
| QCI | `qci` | Cloud |
| Amazon Braket | `braket` | Cloud multi-vendor |

## When to Apply

Reference these guidelines when:
- Building quantum circuits for simulation or hardware execution
- Implementing variational algorithms (VQE, QAOA, quantum ML)
- Scaling quantum simulations to multi-GPU or multi-node
- Modeling quantum noise and error channels
- Targeting real quantum hardware from cloud providers
- Benchmarking quantum algorithms with GPU acceleration
- Constructing hybrid quantum-classical optimization loops
- Generating molecular Hamiltonians for quantum chemistry
- Quantum error correction research and syndrome decoding
- Simulating Hamiltonian dynamics and time evolution

## Priority-Ordered Guidelines

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Kernel Construction | CRITICAL | `kernels-*` |
| 2 | Circuit Execution | CRITICAL | `sampling-*` |
| 3 | Variational Algorithms | HIGH | `variational-*` |
| 4 | Multi-GPU Scaling | HIGH | `multi-gpu-*` |
| 5 | CUDA-QX Extensions | HIGH | `cudaqx-*` |
| 6 | Dynamics Simulation | HIGH | `dynamics-*` |
| 7 | Noise Modeling | MEDIUM | `noise-*` |
| 8 | Hardware Backends | MEDIUM | `hardware-*` |

## References

Full documentation with code examples in [references/](references/):

| File | Impact | Description |
|------|--------|-------------|
| [kernels-and-gates.md][kernels-and-gates] | CRITICAL | Kernel creation, qubit allocation, gate operations, parameterization |
| [sampling-and-observe.md][sampling-and-observe] | CRITICAL | cudaq.sample, cudaq.observe, async execution, statevector access |
| [variational-algorithms.md][variational-algorithms] | HIGH | VQE, QAOA, optimization loops, Hamiltonian construction |
| [multi-gpu-workflows.md][multi-gpu-workflows] | HIGH | Multi-GPU simulation, mqpu batching, distributed execution |
| [cudaqx-extensions.md][cudaqx-extensions] | HIGH | Molecular Hamiltonians, ADAPT-VQE, QAOA, QEC codes, decoders |
| [dynamics-simulation.md][dynamics-simulation] | HIGH | cudaq.evolve, time-dependent Hamiltonians, Lindblad, bosonic operators |
| [noise-modeling.md][noise-modeling] | MEDIUM | Noise channels, density matrix simulation, Kraus operators, Stim |
| [hardware-backends.md][hardware-backends] | MEDIUM | IonQ, Quantinuum, IQM, OQC, ORCA, Infleqtion, Pasqal, Braket |

## Problem -> Skill Mapping

| Problem | Start With |
|---------|------------|
| Build a quantum circuit | [kernels-and-gates.md][kernels-and-gates] |
| Run circuit and get results | [sampling-and-observe.md][sampling-and-observe] |
| Implement VQE/QAOA | [variational-algorithms.md][variational-algorithms] |
| Scale to multiple GPUs | [multi-gpu-workflows.md][multi-gpu-workflows] |
| Simulate with noise | [noise-modeling.md][noise-modeling] |
| Run on real quantum hardware | [hardware-backends.md][hardware-backends] |
| Compute expectation values | [sampling-and-observe.md][sampling-and-observe] |
| Optimize circuit parameters | [variational-algorithms.md][variational-algorithms] |
| Batch parameter sweeps | [multi-gpu-workflows.md][multi-gpu-workflows] |
| Molecular chemistry (H₂, N₂) | [cudaqx-extensions.md][cudaqx-extensions] |
| Quantum error correction | [cudaqx-extensions.md][cudaqx-extensions] |
| Syndrome decoding | [cudaqx-extensions.md][cudaqx-extensions] |
| Hamiltonian time evolution | [dynamics-simulation.md][dynamics-simulation] |
| Open quantum systems (Lindblad) | [dynamics-simulation.md][dynamics-simulation] |
| Bosonic/photonic simulation | [dynamics-simulation.md][dynamics-simulation] |

[kernels-and-gates]: references/kernels-and-gates.md
[sampling-and-observe]: references/sampling-and-observe.md
[variational-algorithms]: references/variational-algorithms.md
[multi-gpu-workflows]: references/multi-gpu-workflows.md
[noise-modeling]: references/noise-modeling.md
[cudaqx-extensions]: references/cudaqx-extensions.md
[dynamics-simulation]: references/dynamics-simulation.md
[hardware-backends]: references/hardware-backends.md

## Attribution

Based on [NVIDIA CUDA-Q](https://nvidia.github.io/cuda-quantum/latest/index.html), [CUDA-QX](https://nvidia.github.io/cudaqx/) documentation and [cuda-quantum](https://github.com/NVIDIA/cuda-quantum) repository.
