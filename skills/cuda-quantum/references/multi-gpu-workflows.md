---
title: Multi-GPU Workflows
impact: HIGH
tags: multi-gpu, mgpu, mqpu, distributed, mpi, parameter-batching, remote-mqpu
---

# Skill: Multi-GPU Workflows

Scale quantum simulations across multiple GPUs and nodes using CUDA-Q's multi-GPU and multi-QPU capabilities.

## Quick Command

```bash
# Single GPU (default nvidia target)
python my_circuit.py --target nvidia

# Multi-GPU state vector (requires MPI)
mpiexec -np 4 python my_circuit.py --target nvidia --target-option mgpu

# Multi-QPU Hamiltonian batching
python my_vqe.py --target nvidia --target-option mqpu
```

## When to Use

- Simulating circuits with >30 qubits (state vector exceeds single GPU memory)
- Batching expectation value computation across GPUs
- Running parameter sweeps in parallel
- Distributing Hamiltonian term evaluation
- Scaling to multi-node HPC clusters

## Step-by-Step Instructions

### 1. CPU to GPU Transition

```python
import cudaq

@cudaq.kernel
def ghz_state(n: int):
    qubits = cudaq.qvector(n)
    h(qubits[0])
    for i in range(1, n):
        x.ctrl(qubits[0], qubits[i])
    mz(qubits)

# CPU execution
cudaq.set_target("qpp-cpu")
cpu_result = cudaq.sample(ghz_state, 2, shots_count=1000)

# GPU execution (much faster for large circuits)
if cudaq.num_available_gpus() > 0:
    cudaq.set_target("nvidia")
    gpu_result = cudaq.sample(ghz_state, 25, shots_count=1000)
```

### 2. Multi-GPU State Vector (mgpu)

For circuits with >30 qubits where the state vector exceeds single GPU memory:

```bash
# Launch with MPI — each rank uses one GPU
mpiexec -np 4 python my_circuit.py --target nvidia --target-option mgpu
```

Or set in code:

```python
cudaq.set_target('nvidia', option='mgpu')
```

**Key details:**
- State vector distributed across GPU memories via MPI
- Up to 6 gates fused together (vs 4 for single-GPU)
- Requires MPI installation and multi-GPU system

### 3. Multi-QPU Hamiltonian Batching (mqpu)

Distribute Hamiltonian term evaluation across virtual QPUs:

```python
import cudaq
from cudaq import spin

cudaq.set_target("nvidia", option="mqpu")

target = cudaq.get_target()
print(f"Number of QPUs: {target.num_qpus()}")

qubit_count = 15
term_count = 100000

@cudaq.kernel
def kernel(n: int):
    qubits = cudaq.qvector(n)
    h(qubits[0])
    for i in range(1, n):
        x.ctrl(qubits[0], qubits[i])

hamiltonian = cudaq.SpinOperator.random(qubit_count, term_count)

# Single GPU baseline
result = cudaq.observe(kernel, hamiltonian, qubit_count)

# Multi-GPU on single node (thread-based distribution)
result = cudaq.observe(kernel, hamiltonian, qubit_count,
                       execution=cudaq.parallel.thread)

# Multi-node multi-GPU (MPI-based distribution)
cudaq.mpi.initialize()
result = cudaq.observe(kernel, hamiltonian, qubit_count,
                       execution=cudaq.parallel.mpi)
cudaq.mpi.finalize()
```

### 4. Parameter Sweep Batching

Distribute parameter evaluations across GPUs:

```python
import cudaq
from cudaq import spin
import numpy as np

cudaq.set_target("nvidia", option="mqpu")

@cudaq.kernel
def kernel(params: list[float]):
    qubits = cudaq.qvector(5)
    for i in range(5):
        rx(params[i], qubits[i])

h = spin.z(0)
sample_count = 10000
parameters = np.random.default_rng(13).uniform(0, 1, (sample_count, 5))

# Split parameters across GPUs
num_gpus = cudaq.num_available_gpus()
param_splits = np.split(parameters, num_gpus)

# Async observe across QPUs
asyncresults = []
for gpu_id, params_chunk in enumerate(param_splits):
    for j in range(params_chunk.shape[0]):
        asyncresults.append(
            cudaq.observe_async(kernel, h, params_chunk[j, :], qpu_id=gpu_id))

results = [res.get() for res in asyncresults]
```

### 5. Remote Multi-QPU (remote-mqpu)

Distribute across remote simulation servers:

```python
import cudaq

cudaq.set_target("remote-mqpu",
                 backend="tensornet-mps",
                 auto_launch="2")  # Launch 2 server instances

qpu_count = cudaq.get_target().num_qpus()

@cudaq.kernel
def kernel(controls_count: int):
    controls = cudaq.qvector(controls_count)
    targets = cudaq.qvector(40)
    h(controls)
    for t in range(40):
        x.ctrl(controls, targets[t])
    mz(controls)
    mz(targets)

# Async sampling across remote QPUs
futures = []
for i in range(qpu_count):
    futures.append(cudaq.sample_async(kernel, i + 1, qpu_id=i))

for f in futures:
    print(f.get())
```

### 6. Gate Fusion Optimization

Control how many gates are fused together for performance:

```bash
# Single-GPU fusion (nvidia target)
export CUDAQ_FUSION_MAX_QUBITS=5

# Multi-GPU fusion (nvidia mgpu target)
export CUDAQ_MGPU_FUSE=6
```

**Default gate fusion sizes by GPU:**

| GPU | FP32 | FP64 |
|-----|------|------|
| A100 (CC 8.0) | 4 | 5 |
| H100/H200 (CC 9.0) | 5 | 6 |
| GB200/B200 (CC 10.0) | 5 | 4 |
| B300 (CC 10.3) | 5 | 1 |

**Key environment variables:**

| Variable | Target | Description |
|----------|--------|-------------|
| `CUDAQ_FUSION_MAX_QUBITS` | nvidia | Max qubits fused (single-GPU) |
| `CUDAQ_FUSION_DIAGONAL_GATE_MAX_QUBITS` | nvidia | Diagonal gate fusion (-1=auto) |
| `CUDAQ_MGPU_FUSE` | nvidia mgpu | Max qubits fused (multi-GPU) |
| `CUDAQ_MGPU_NQUBITS_THRESH` | nvidia mgpu | Qubit threshold for distribution (default: 25) |
| `CUDAQ_MAX_CPU_MEMORY_GB` | nvidia | CPU memory for state-vector migration |
| `CUDAQ_MAX_GPU_MEMORY_GB` | nvidia | GPU memory limit for state-vector |
| `CUDAQ_MPS_MAX_BOND` | tensornet-mps | Max singular values (default: 64) |
| `CUDAQ_MPS_ABS_CUTOFF` | tensornet-mps | Absolute SVD cutoff (default: 1e-5) |
| `CUDAQ_TENSORNET_NUM_HYPER_SAMPLES` | tensornet | Contraction path samples (default: 8) |

Higher fusion levels reduce kernel launches but use more memory. Tune based on circuit structure.

### 7. Fermioniq Tensor Network Backend

Third-party GPU tensor network simulator:

```python
cudaq.set_target("fermioniq")
# Requires Fermioniq credentials and API access
```

## Backend Selection Guide

| Scenario | Target | Option |
|----------|--------|--------|
| Quick prototyping (no GPU) | `qpp-cpu` | — |
| Single GPU simulation | `nvidia` | — |
| Double precision | `nvidia-fp64` | — |
| >30 qubits, multi-GPU | `nvidia` | `mgpu` |
| Hamiltonian term parallelism | `nvidia` | `mqpu` |
| Tensor network (many qubits) | `tensornet` | — |
| MPS approximation | `tensornet-mps` | — |
| Fermioniq tensor network | `fermioniq` | — |
| Noisy simulation (CPU) | `density-matrix-cpu` | — |
| Noisy simulation (GPU) | `nvidia` | fp64 + noise_model |
| Clifford/QEC fast noisy | `stim` | — |
| Dynamics/time-evolution | `dynamics` | — |
| Remote distributed | `remote-mqpu` | backend, auto_launch |

## Common Pitfalls

- **MPI not initialized**: `nvidia-mgpu` requires `mpiexec`. Running without MPI silently falls back to single GPU.
- **GPU memory exceeded**: Use `tensornet` or `mgpu` for large qubit counts instead of crashing.
- **QPU ID out of range**: Check `cudaq.get_target().num_qpus()` before assigning `qpu_id`.
- **Forgetting `cudaq.mpi.finalize()`**: Always call finalize after MPI-based execution.

## Related Skills

- [sampling-and-observe.md](./sampling-and-observe.md) - Single-GPU execution patterns
- [variational-algorithms.md](./variational-algorithms.md) - Distribute VQE/QAOA
- [hardware-backends.md](./hardware-backends.md) - Target real quantum hardware
