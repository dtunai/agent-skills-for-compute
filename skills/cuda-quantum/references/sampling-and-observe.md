---
title: Sampling and Observe
impact: CRITICAL
tags: sample, observe, expectation, statevector, async, run, measurement
---

# Skill: Sampling and Observe

Execute quantum kernels to get measurement results, compute expectation values, and access statevectors.

## Quick Pattern

**Sampling (measurement counts):**

```python
import cudaq

@cudaq.kernel
def ghz(n: int):
    qubits = cudaq.qvector(n)
    h(qubits[0])
    for i in range(1, n):
        x.ctrl(qubits[0], qubits[i])
    mz(qubits)

result = cudaq.sample(ghz, 3, shots_count=1000)
print(result)  # {'000': ~500, '111': ~500}
```

**Observe (expectation values):**

```python
from cudaq import spin

hamiltonian = spin.z(0) + spin.y(1) + spin.x(0) * spin.z(0)

@cudaq.kernel
def kernel(n: int):
    qubits = cudaq.qvector(n)
    h(qubits[0])
    for i in range(1, n):
        x.ctrl(qubits[0], qubits[i])

result = cudaq.observe(kernel, hamiltonian, 2)
print('<H> =', result.expectation())
```

## When to Use

- Getting measurement statistics from quantum circuits
- Computing Hamiltonian expectation values
- Running parameter sweeps over circuits
- Accessing full quantum statevector
- Asynchronous batch execution

## Step-by-Step Instructions

### 1. Basic Sampling

```python
cudaq.set_target("qpp-cpu")

@cudaq.kernel
def kernel(n: int):
    qvector = cudaq.qvector(n)
    h(qvector[0])
    for i in range(1, n):
        x.ctrl(qvector[0], qvector[i])
    mz(qvector)

print(cudaq.draw(kernel, 2))
result = cudaq.sample(kernel, 2, shots_count=1000)
print(result)
```

### 2. Asynchronous Sampling

```python
result_async = cudaq.sample_async(kernel, 2, shots_count=1000)
# Do other work...
result = result_async.get()
print(result)
```

### 3. Run with Return Values

```python
@cudaq.kernel
def ghz_count(num_qubits: int) -> int:
    qubits = cudaq.qvector(num_qubits)
    h(qubits[0])
    for i in range(1, num_qubits):
        x.ctrl(qubits[0], qubits[i])
    res = 0
    for i in range(num_qubits):
        if mz(qubits[i]):
            res += 1
    return res

results = cudaq.run(ghz_count, 3, shots_count=20)
print(f"Results: {results}")
```

### 4. Run with Custom Dataclass

```python
from dataclasses import dataclass

@dataclass(slots=True)
class MeasurementResult:
    first_qubit: bool
    last_qubit: bool
    total_ones: int

@cudaq.kernel
def bell_data() -> MeasurementResult:
    qubits = cudaq.qvector(2)
    h(qubits[0])
    x.ctrl(qubits[0], qubits[1])
    first = mz(qubits[0])
    last = mz(qubits[1])
    total = 0
    if first: total = 1
    if last: total = total + 1
    return MeasurementResult(first, last, total)

results = cudaq.run(bell_data, shots_count=10)
for i, res in enumerate(results):
    print(f"Shot {i}: {res.first_qubit}, {res.last_qubit}")
```

### 5. Observe with Spin Hamiltonian

```python
from cudaq import spin

# Build Hamiltonian: H = 5.907 - 2.1433 XX - 2.1433 YY + 0.21829 Z0 - 6.125 Z1
hamiltonian = 5.907 - 2.1433 * spin.x(0) * spin.x(1) \
    - 2.1433 * spin.y(0) * spin.y(1) \
    + 0.21829 * spin.z(0) - 6.125 * spin.z(1)

@cudaq.kernel
def ansatz(angle: float):
    qvector = cudaq.qvector(2)
    x(qvector[0])
    ry(angle, qvector[1])
    x.ctrl(qvector[1], qvector[0])

exp_val = cudaq.observe(ansatz, hamiltonian, 0.59).expectation()
print("Expectation value:", exp_val)
```

### 6. Async Observe across QPUs

```python
cudaq.set_target("nvidia", option="mqpu")

result_async = cudaq.observe_async(kernel, hamiltonian, 2, qpu_id=0)
# Do other work...
print(result_async.get().expectation())
```

### 7. Get Statevector

```python
import numpy as np

@cudaq.kernel
def bell_state():
    q = cudaq.qvector(2)
    h(q[0])
    x.ctrl(q[0], q[1])

state = cudaq.get_state(bell_state)
print(np.array(state))  # [0.707, 0, 0, 0.707]

# Async version
state_future = cudaq.get_state_async(bell_state)
state = state_future.get()
```

### 8. Async Run for Parameter Sweeps

```python
@cudaq.kernel
def parameterized(angle: float) -> int:
    q = cudaq.qubit()
    rx(angle, q)
    return int(mz(q))

futures = []
angles = [0.0, 0.2, 0.4, 0.6, 0.8, 1.0, 1.2, 1.4]
for angle in angles:
    futures.append(cudaq.run_async(parameterized, angle, shots_count=10))

for i, future in enumerate(futures):
    results = future.get()
    ones = sum(results)
    print(f"Angle {angles[i]:.1f}: {ones}/10 ones")
```

### 9. Explicit Measurements Option

Control how measurements appear in `sample_result`:

```python
@cudaq.kernel
def kernel():
    q = cudaq.qubit()
    h(q)
    reg1 = mz(q)    # Named register
    reset(q)
    x(q)

# Default: __global__ register re-measures at end
cudaq.sample(kernel).dump()
# { __global__: {1:1000}, reg1: {0:506, 1:494} }

# Explicit: global = concatenated measurements in order
cudaq.sample(kernel, explicit_measurements=True).dump()
# { 0:479, 1:521 }
```

### 10. Sample Options with Noise

```python
result = cudaq.sample(kernel, noise_model=noise, shots_count=1000)

# Or via observe
result = cudaq.observe(kernel, hamiltonian, noise_model=noise, shots_count=1000)
```

## Spin Operator Construction

```python
from cudaq import spin

# Pauli operators on specific qubits
z0 = spin.z(0)
x1 = spin.x(1)
y2 = spin.y(2)
i0 = spin.i(0)  # Identity

# Combine with arithmetic
H = 2.0 * spin.z(0) + 0.5 * spin.x(0) * spin.x(1)

# Random Hamiltonian
H_random = cudaq.SpinOperator.random(qubit_count=5, term_count=100)

# From Pauli word string
H = cudaq.SpinOperator.from_word("XXYZ")
```

## Common Pitfalls

- **No `mz()` in kernel for `cudaq.sample`**: Sample requires measurement gates. Observe does not.
- **Wrong parameter types**: Kernel parameters must match decorated type hints exactly.
- **Blocking on async**: Call `.get()` only when you need the result — calling too early defeats the purpose of async.
- **Forgetting `shots_count`**: Default shot count may be low. Set explicitly for statistical significance.

## Related Skills

- [kernels-and-gates.md](./kernels-and-gates.md) - Build the kernels to execute
- [variational-algorithms.md](./variational-algorithms.md) - Use observe in optimization loops
- [multi-gpu-workflows.md](./multi-gpu-workflows.md) - Distribute execution across GPUs
