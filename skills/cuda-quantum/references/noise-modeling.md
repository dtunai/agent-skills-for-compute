---
title: Noise Modeling
impact: MEDIUM
tags: noise, depolarization, bit-flip, phase-flip, amplitude-damping, kraus, density-matrix
---

# Skill: Noise Modeling

Simulate quantum circuits with realistic noise using CUDA-Q noise models, channels, and Kraus operators.

## Quick Pattern

```python
import cudaq

cudaq.set_target("density-matrix-cpu")

@cudaq.kernel
def kernel():
    q = cudaq.qvector(2)
    x(q)

noise = cudaq.NoiseModel()
noise.add_channel("x", [0], cudaq.DepolarizationChannel(0.1))
noise.add_channel("x", [1], cudaq.BitFlipChannel(0.1))

result = cudaq.sample(kernel, noise_model=noise, shots_count=1000)
result.dump()
```

## When to Use

- Simulating realistic quantum hardware noise
- Benchmarking algorithm noise resilience
- Developing quantum error mitigation strategies
- Testing error correction codes
- Comparing ideal vs noisy circuit behavior

## Step-by-Step Instructions

### 1. Set Density Matrix Backend

```python
import cudaq
import numpy as np

# Required for noise simulation
cudaq.set_target("density-matrix-cpu")
```

### 2. Built-in Noise Channels

```python
error_prob = 0.1

# Standard channels
depol = cudaq.DepolarizationChannel(error_prob)
bit_flip = cudaq.BitFlipChannel(error_prob)
phase_flip = cudaq.PhaseFlipChannel(error_prob)
amp_damp = cudaq.AmplitudeDampingChannel(error_prob)
```

### 3. Apply Noise to Specific Gates and Qubits

```python
@cudaq.kernel
def kernel():
    q = cudaq.qvector(2)
    x(q)

noise = cudaq.NoiseModel()
noise.add_channel("x", [0], cudaq.DepolarizationChannel(0.1))  # X gate on qubit 0
noise.add_channel("x", [1], cudaq.BitFlipChannel(0.05))        # X gate on qubit 1
noise.add_channel("h", [0], cudaq.PhaseFlipChannel(0.02))      # H gate on qubit 0

result = cudaq.sample(kernel, noise_model=noise, shots_count=1000)
```

### 4. Custom Kraus Operators

```python
import numpy as np

error_prob = 0.1

# Custom bit-flip via Kraus operators
kraus_0 = np.sqrt(1 - error_prob) * np.array(
    [[1.0, 0.0], [0.0, 1.0]], dtype=np.complex128)
kraus_1 = np.sqrt(error_prob) * np.array(
    [[0.0, 1.0], [1.0, 0.0]], dtype=np.complex128)

custom_channel = cudaq.KrausChannel([kraus_0, kraus_1])

noise = cudaq.NoiseModel()
noise.add_channel("x", [0], custom_channel)
```

### 5. Noisy Expectation Values

```python
from cudaq import spin

hamiltonian = spin.z(0)

@cudaq.kernel
def kernel():
    q = cudaq.qvector(2)
    x(q)

noise = cudaq.NoiseModel()
noise.add_channel("x", [0], cudaq.DepolarizationChannel(0.1))

noisy_result = cudaq.observe(kernel, hamiltonian, noise_model=noise)
print("Noisy <Z>:", noisy_result.expectation())
```

### 6. Explicit Noise Injection in Kernels

```python
@cudaq.kernel
def noisy_circuit():
    q = cudaq.qvector(3)

    # Inject noise at specific points
    cudaq.apply_noise(cudaq.DepolarizationChannel, 0.1, q[0])

    h(q[1])
    x.ctrl(q[1], q[2])

    cudaq.apply_noise(cudaq.Pauli1, 0.1, 0.1, 0.1, q[2])

noise = cudaq.NoiseModel()
counts = cudaq.sample(noisy_circuit, noise_model=noise)
```

### 7. Trajectory-Based Noisy Simulation

For GPU-accelerated noisy simulation using state vector trajectory method:

```python
# GPU trajectory simulation (faster for many qubits)
cudaq.set_target("nvidia", option="fp64")

noise = cudaq.NoiseModel()
noise.add_channel("x", [0], cudaq.DepolarizationChannel(0.1))

# Trajectory simulation on GPU
result = cudaq.sample(kernel, noise_model=noise, shots_count=10000)
```

### 8. Stim Backend (Clifford Circuits)

For fast noisy Clifford circuit simulation:

```python
cudaq.set_target("stim")

# Stim is extremely fast for Clifford-only circuits with Pauli noise
noise = cudaq.NoiseModel()
noise.add_channel("h", [0], cudaq.DepolarizationChannel(0.01))

result = cudaq.sample(kernel, noise_model=noise, shots_count=100000)
```

## Noise Channel Reference

| Channel | Parameters | Effect |
|---------|-----------|--------|
| `DepolarizationChannel(p)` | Error probability p | Random Pauli error (X, Y, or Z) |
| `BitFlipChannel(p)` | Flip probability p | X error with probability p |
| `PhaseFlipChannel(p)` | Flip probability p | Z error with probability p |
| `AmplitudeDampingChannel(p)` | Decay probability p | Energy relaxation (T1 decay) |
| `KrausChannel([K0, K1, ...])` | Kraus matrices | Custom noise channel |
| `Pauli1(px, py, pz)` | X/Y/Z probabilities | Per-axis Pauli noise |

## Noisy Simulator Backends

| Target | Type | GPU | Best For |
|--------|------|-----|----------|
| `density-matrix-cpu` | Density matrix | No | Small circuits, full noise |
| `nvidia` (trajectory) | State vector trajectories | Yes | Larger circuits, GPU speed |
| `stim` | Stabilizer | No | Clifford circuits, QEC, millions of shots |

## Common Pitfalls

- **Wrong backend**: Full density matrix noise requires `density-matrix-cpu`. GPU targets use trajectory simulation.
- **Kraus completeness**: Custom Kraus operators must satisfy sum(Ki^dag Ki) = I.
- **Gate name strings**: Use lowercase gate names in `add_channel("x", ...)`, not class names.
- **Performance**: Density matrix simulation is O(4^n) memory vs O(2^n) for state vector. Use fewer qubits.
- **Stim limitations**: Only supports Clifford gates + Pauli noise. Non-Clifford gates will error.

## Related Skills

- [kernels-and-gates.md](./kernels-and-gates.md) - Build circuits to apply noise to
- [sampling-and-observe.md](./sampling-and-observe.md) - Execute noisy circuits
- [hardware-backends.md](./hardware-backends.md) - Real hardware has inherent noise
