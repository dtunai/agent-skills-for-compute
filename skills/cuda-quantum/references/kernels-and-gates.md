---
title: Kernels and Gates
impact: CRITICAL
tags: kernel, qubit, qvector, gates, parameterized, controlled, adjoint
---

# Skill: Kernels and Gates

Create CUDA-Q quantum kernels with qubit allocation, gate operations, parameterization, and kernel composition.

## Quick Pattern

**Incorrect:**

```python
# No @cudaq.kernel decorator — won't compile to quantum IR
def my_circuit():
    q = allocate_qubit()
    apply_h(q)
```

**Correct:**

```python
import cudaq

@cudaq.kernel
def my_circuit():
    q = cudaq.qvector(2)
    h(q[0])
    x.ctrl(q[0], q[1])
    mz(q)
```

## When to Use

- Building any quantum circuit in CUDA-Q
- Defining parameterized ansatze for variational algorithms
- Creating reusable quantum subroutines
- Initializing qubits in specific states

## Step-by-Step Instructions

### 1. Basic Kernel with Qubit Allocation

```python
import cudaq

@cudaq.kernel
def kernel():
    single = cudaq.qubit()      # Single qubit
    register = cudaq.qvector(3) # 3-qubit register
    big_reg = cudaq.qvector(5)  # 5-qubit register
```

### 2. Parameterized Kernels

```python
@cudaq.kernel
def kernel(N: int):
    register = cudaq.qvector(N)  # Runtime-sized register

@cudaq.kernel
def ansatz(thetas: list[float]):
    qubits = cudaq.qvector(2)
    rx(thetas[0], qubits[0])
    ry(thetas[1], qubits[1])

ansatz([0.024, 0.543])
```

### 3. Gate Operations on Registers

```python
@cudaq.kernel
def kernel():
    register = cudaq.qvector(10)
    h(register)           # Apply H to ALL qubits
    h(register[0])        # Apply H to first qubit
    h(register[-1])       # Apply H to last qubit
```

### 4. Controlled Operations

```python
@cudaq.kernel
def kernel():
    reg = cudaq.qvector(10)

    # Single-control CNOT
    x.ctrl(reg[0], reg[1])

    # Multi-controlled Toffoli
    x.ctrl([reg[0], reg[1]], reg[2])
```

### 5. Controlled Kernel Calls

```python
@cudaq.kernel
def x_kernel(qubit: cudaq.qubit):
    x(qubit)

@cudaq.kernel
def kernel():
    control_vector = cudaq.qvector(2)
    target = cudaq.qubit()
    x(control_vector)
    x(target)
    x(control_vector[1])
    cudaq.control(x_kernel, control_vector, target)

results = cudaq.sample(kernel)
```

### 6. Adjoint (Inverse) Operations

```python
@cudaq.kernel
def kernel():
    register = cudaq.qvector(10)
    t.adj(register[0])    # Inverse T gate
    s.adj(register[1])    # Inverse S gate
```

### 7. Kernel Composition

```python
@cudaq.kernel
def sub_circuit(q0: cudaq.qubit, q1: cudaq.qubit):
    x.ctrl(q0, q1)

@cudaq.kernel
def main_circuit():
    reg = cudaq.qvector(10)
    for i in range(5):
        sub_circuit(reg[i], reg[i + 1])
```

### 8. State Initialization

```python
import numpy as np

# From complex vector
c = [0.70710678 + 0j, 0., 0., 0.70710678]

@cudaq.kernel
def from_vector():
    q = cudaq.qvector(c)

# From amplitudes helper
c = cudaq.amplitudes([0.70710678 + 0j, 0., 0., 0.70710678])

@cudaq.kernel
def from_amplitudes():
    q = cudaq.qvector(c)

# From another kernel's state
state = cudaq.get_state(from_vector)

@cudaq.kernel
def from_state(state: cudaq.State):
    q = cudaq.qvector(state)

from_state(state)
```

### 9. Custom Gate Operations

```python
import numpy as np

cudaq.register_operation("custom_x", np.array([0, 1, 1, 0]))

@cudaq.kernel
def kernel():
    qubits = cudaq.qvector(2)
    h(qubits[0])
    custom_x(qubits[0])
    custom_x.ctrl(qubits[0], qubits[1])
```

### 10. Visualize and Translate Circuits

```python
# ASCII circuit drawing
print(cudaq.draw(my_kernel, *args))

# Translate to OpenQASM
qasm = cudaq.translate(my_kernel, format="openqasm2")

# Estimate resources (gate counts, depth)
resources = cudaq.estimate_resources(my_kernel, *args)
print(resources)
```

### 11. Mid-Circuit Measurement and Conditional Logic

```python
@cudaq.kernel
def teleport():
    q = cudaq.qvector(3)
    # Prepare Bell pair
    h(q[1])
    x.ctrl(q[1], q[2])
    # Encode
    x.ctrl(q[0], q[1])
    h(q[0])
    # Measure and conditionally correct
    b0 = mz(q[0])
    b1 = mz(q[1])
    if b1:
        x(q[2])
    if b0:
        z(q[2])
```

### 12. Negated Control Polarity

Apply transformation when control qubit is |0> instead of |1>:

```python
@cudaq.kernel
def kernel():
    c, q = cudaq.qubit(), cudaq.qubit()
    h(c)
    # Apply X to q when c is |0> (negated control)
    x.ctrl(~c, q)
```

## Complete Gate Reference

| Gate | Syntax | Description |
|------|--------|-------------|
| `h` | `h(q)` | Hadamard |
| `x` | `x(q)` | Pauli-X (NOT) |
| `y` | `y(q)` | Pauli-Y |
| `z` | `z(q)` | Pauli-Z |
| `s` | `s(q)` | Phase (S) |
| `t` | `t(q)` | T gate |
| `rx` | `rx(θ, q)` | X rotation |
| `ry` | `ry(θ, q)` | Y rotation |
| `rz` | `rz(λ, q)` | Z rotation |
| `r1` | `r1(λ, q)` | Phase rotation |
| `u3` | `u3(θ, φ, λ, q)` | Universal 3-parameter gate |
| `swap` | `swap(q0, q1)` | SWAP |
| `mz` | `mz(q)` | Z-basis measurement |
| `mx` | `mx(q)` | X-basis measurement |
| `my` | `my(q)` | Y-basis measurement |
| `reset` | `reset(q)` | Reset qubit to \|0> |

### Controlled variants: `gate.ctrl(control, target)` or `gate.ctrl([c0, c1], target)`
### Adjoint variants: `gate.adj(qubit)`
### Negated control: `gate.ctrl(~control, target)` (apply when control is \|0>)

### Photonic Operations (orca-photonics target only)

| Gate | Syntax | Description |
|------|--------|-------------|
| `create` | `create(qumode)` | Increment photon number |
| `annihilate` | `annihilate(qumode)` | Decrement photon number |
| `phase_shift` | `phase_shift(qumode, φ)` | Phase shift on qumode |
| `beam_splitter` | `beam_splitter(q0, q1, θ)` | Beam splitter on two qumodes |

## Common Pitfalls

- **Missing `@cudaq.kernel`**: Without the decorator, the function runs as normal Python without quantum compilation.
- **Using Python types inside kernels**: Kernels are JIT-compiled — use `cudaq.qubit`, `cudaq.qvector`, and typed parameters.
- **Index out of range**: `cudaq.qvector(N)` indices are `0` to `N-1`. Use `q[-1]` for the last qubit.
- **Forgetting measurement**: Without `mz()`, `cudaq.sample()` returns empty results.

## Related Skills

- [sampling-and-observe.md](./sampling-and-observe.md) - Execute kernels and get results
- [variational-algorithms.md](./variational-algorithms.md) - Use parameterized kernels in VQE/QAOA
