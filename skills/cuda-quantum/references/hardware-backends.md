---
title: Hardware Backends
impact: MEDIUM
tags: ionq, quantinuum, iqm, oqc, orca, braket, hardware, qpu, cloud
---

# Skill: Hardware Backends

Configure CUDA-Q to target real quantum hardware from cloud providers.

## Quick Command

```bash
# Set target via command line
python my_circuit.py --target ionq

# Set target via environment
export CUDAQ_DEFAULT_SIMULATOR=ionq
python my_circuit.py
```

## When to Use

- Running circuits on real quantum processors
- Benchmarking algorithms on actual hardware
- Comparing simulator results with hardware execution
- Accessing cloud-based quantum computing services

## Step-by-Step Instructions

### 1. IonQ (Trapped Ion)

```python
import cudaq

cudaq.set_target("ionq",
                 machine="simulator")  # or "qpu.aria-1", "qpu.forte-1"

# Set API key
# export IONQ_API_KEY=your_key

@cudaq.kernel
def bell():
    q = cudaq.qvector(2)
    h(q[0])
    x.ctrl(q[0], q[1])
    mz(q)

result = cudaq.sample(bell, shots_count=1000)
print(result)
```

**IonQ machines:**
| Machine | Type | Qubits |
|---------|------|--------|
| `simulator` | Cloud simulator | Up to 29 |
| `qpu.aria-1` | Trapped ion QPU | 25 |
| `qpu.forte-1` | Trapped ion QPU | 36 |

### 2. Quantinuum (Trapped Ion)

```python
cudaq.set_target("quantinuum",
                 machine="H1-1SC")  # Syntax checker (free)

# export QUANTINUUM_API_KEY=your_key

@cudaq.kernel
def ghz():
    q = cudaq.qvector(3)
    h(q[0])
    for i in range(1, 3):
        x.ctrl(q[0], q[i])
    mz(q)

result = cudaq.sample(ghz)
```

**Quantinuum machines:**
| Machine | Type | Notes |
|---------|------|-------|
| `H1-1SC` | Syntax checker | Free, validates circuit |
| `H1-1E` | Emulator | Noisy simulation |
| `H1-1` | QPU | Production hardware |
| `H2-1SC` | Syntax checker | H2 series |
| `H2-1E` | Emulator | H2 emulator |
| `H2-1` | QPU | H2 hardware |

### 3. IQM (Superconducting)

```python
cudaq.set_target("iqm",
                 url="https://your-iqm-server.com/cocos",
                 machine="Adonis")

# export IQM_TOKEN=your_token

result = cudaq.sample(kernel, shots_count=1000)
```

### 4. OQC (Superconducting)

```python
cudaq.set_target("oqc",
                 url="https://cloud.oqc.app",
                 machine="Lucy")

# export OQC_AUTH_TOKEN=your_token
```

### 5. ORCA (Photonic)

```python
cudaq.set_target("orca",
                 url="http://your-orca-server:port/")
```

### 6. Infleqtion (Neutral Atom)

```python
cudaq.set_target("infleqtion")

# export INFLEQTION_API_KEY=your_key
# Supports Sqale neutral atom QPU
```

### 7. Pasqal (Neutral Atom)

```python
cudaq.set_target("pasqal",
                 machine="fresnel1")

# export PASQAL_AUTH_TOKEN=your_token
```

### 8. QuEra Computing (Neutral Atom)

```python
cudaq.set_target("quera",
                 machine="Aquila")

# export QUERA_API_KEY=your_key
```

### 9. Quantum Machines (Quantum Control)

```python
cudaq.set_target("qm",
                 url="https://your-qm-server.com")

# Supports OPX+ control hardware
```

### 10. Quantum Circuits, Inc. (QCI)

```python
cudaq.set_target("qci")

# export QCI_API_KEY=your_key
```

### 11. Anyon Technologies (Superconducting)

```python
cudaq.set_target("anyon",
                 machine="yukon")

# export ANYON_API_KEY=your_key
```

### 12. Amazon Braket (Multi-Vendor Cloud)

```python
cudaq.set_target("braket",
                 machine="arn:aws:braket:::device/quantum-simulator/amazon/sv1")

# Requires AWS credentials configured
# aws configure

@cudaq.kernel
def kernel():
    q = cudaq.qvector(2)
    h(q[0])
    x.ctrl(q[0], q[1])
    mz(q)

result = cudaq.sample(kernel)
```

**Braket devices:**
| ARN Suffix | Provider | Type |
|------------|----------|------|
| `amazon/sv1` | Amazon | State vector simulator |
| `amazon/tn1` | Amazon | Tensor network simulator |
| `amazon/dm1` | Amazon | Density matrix simulator |
| `ionq/Aria-1` | IonQ | Trapped ion |
| `rigetti/Ankaa-3` | Rigetti | Superconducting |

## Authentication Patterns

All hardware backends require API credentials via environment variables:

```bash
# IonQ
export IONQ_API_KEY=your_key

# Quantinuum
export QUANTINUUM_API_KEY=your_key

# IQM
export IQM_TOKEN=your_token

# OQC
export OQC_AUTH_TOKEN=your_token

# AWS Braket
aws configure  # Uses standard AWS credentials
```

## Simulator vs Hardware Decision

| Factor | Simulator | Hardware |
|--------|-----------|----------|
| Speed | Instant | Minutes to hours (queue) |
| Cost | Free (local) | Per-shot pricing |
| Qubits | Limited by memory | Limited by device |
| Noise | Configurable | Real noise profile |
| Use case | Development, debugging | Validation, benchmarking |

## Common Pitfalls

- **Missing API key**: Set environment variables before `cudaq.set_target()`.
- **Gate set mismatch**: Hardware backends support limited gate sets. CUDA-Q auto-transpiles, but complex circuits may decompose poorly.
- **Queue times**: Hardware jobs are queued. Use `sample_async` and check results later.
- **Shot limits**: Some providers cap maximum shots per job.
- **Connectivity constraints**: Real devices have limited qubit connectivity. Transpiler adds SWAP gates automatically.

## Related Skills

- [kernels-and-gates.md](./kernels-and-gates.md) - Build hardware-compatible circuits
- [noise-modeling.md](./noise-modeling.md) - Simulate hardware noise locally first
- [multi-gpu-workflows.md](./multi-gpu-workflows.md) - GPU simulation alternatives
