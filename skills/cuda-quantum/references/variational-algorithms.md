---
title: Variational Algorithms
impact: HIGH
tags: vqe, qaoa, optimization, hamiltonian, ansatz, cobyla, gradient
---

# Skill: Variational Algorithms

Implement VQE, QAOA, and custom variational quantum-classical optimization loops using CUDA-Q.

## Quick Pattern

```python
import cudaq
from cudaq import spin

@cudaq.kernel
def ansatz(theta: float):
    q = cudaq.qvector(2)
    x(q[0])
    ry(theta, q[1])
    x.ctrl(q[1], q[0])

H = 5.907 - 2.1433 * spin.x(0) * spin.x(1) \
    - 2.1433 * spin.y(0) * spin.y(1) \
    + 0.21829 * spin.z(0) - 6.125 * spin.z(1)

energy, params, data = cudaq.vqe(ansatz, H, cudaq.optimizers.COBYLA(), parameter_count=1)
print(f"Ground state energy: {energy}")
```

## When to Use

- Finding ground state energies of molecular Hamiltonians
- Solving combinatorial optimization problems (Max-Cut, TSP)
- Implementing quantum machine learning circuits
- Benchmarking quantum algorithms with classical optimization

## Step-by-Step Instructions

### 1. VQE with Built-in Optimizer

```python
import cudaq
from cudaq import spin

# Molecular hydrogen Hamiltonian
hamiltonian = 5.907 - 2.1433 * spin.x(0) * spin.x(1) \
    - 2.1433 * spin.y(0) * spin.y(1) \
    + 0.21829 * spin.z(0) - 6.125 * spin.z(1)

@cudaq.kernel
def ansatz(theta: float):
    q = cudaq.qvector(2)
    x(q[0])
    ry(theta, q[1])
    x.ctrl(q[1], q[0])

# Run VQE
optimizer = cudaq.optimizers.COBYLA()
optimizer.max_iterations = 50
energy, params, data = cudaq.vqe(ansatz, hamiltonian, optimizer, parameter_count=1)
print(f"VQE Energy: {energy:.6f}")
print(f"Optimal params: {params}")
```

### 2. Manual VQE Loop (Full Control)

```python
import cudaq
from cudaq import spin
import numpy as np

hamiltonian = 5.907 - 2.1433 * spin.x(0) * spin.x(1) \
    - 2.1433 * spin.y(0) * spin.y(1) \
    + 0.21829 * spin.z(0) - 6.125 * spin.z(1)

@cudaq.kernel
def ansatz(thetas: list[float]):
    q = cudaq.qvector(2)
    x(q[0])
    ry(thetas[0], q[1])
    x.ctrl(q[1], q[0])

def cost_function(params):
    return cudaq.observe(ansatz, hamiltonian, params).expectation()

# SciPy optimizer
from scipy.optimize import minimize

initial_params = [0.0]
result = minimize(cost_function, initial_params, method='COBYLA',
                  options={'maxiter': 100})
print(f"Energy: {result.fun:.6f}, Params: {result.x}")
```

### 3. QAOA for Max-Cut

```python
import cudaq
from cudaq import spin
import numpy as np

# Max-Cut Hamiltonian for a graph
# Graph edges: (0,1), (1,2), (2,3), (3,0)
qubit_count = 4

hamiltonian = 0.5 * (spin.z(0) * spin.z(1) + spin.z(1) * spin.z(2) \
    + spin.z(2) * spin.z(3) + spin.z(3) * spin.z(0))

@cudaq.kernel
def qaoa_kernel(thetas: list[float], layers: int, qubit_count: int):
    q = cudaq.qvector(qubit_count)
    # Initial superposition
    h(q)
    for layer in range(layers):
        gamma = thetas[2 * layer]
        beta = thetas[2 * layer + 1]
        # Problem unitary
        for i in range(qubit_count):
            j = (i + 1) % qubit_count
            x.ctrl(q[i], q[j])
            rz(gamma, q[j])
            x.ctrl(q[i], q[j])
        # Mixer unitary
        for i in range(qubit_count):
            rx(beta, q[i])

# Optimize
layers = 2
initial_params = np.random.uniform(0, 2 * np.pi, 2 * layers).tolist()

def qaoa_cost(params):
    return cudaq.observe(qaoa_kernel, hamiltonian, params, layers, qubit_count).expectation()

from scipy.optimize import minimize
result = minimize(qaoa_cost, initial_params, method='COBYLA', options={'maxiter': 200})
print(f"QAOA energy: {result.fun:.6f}")
```

### 4. Multi-Parameter Ansatz

```python
@cudaq.kernel
def hardware_efficient_ansatz(thetas: list[float], n_qubits: int, depth: int):
    q = cudaq.qvector(n_qubits)
    param_idx = 0
    for d in range(depth):
        # Single-qubit rotation layer
        for i in range(n_qubits):
            ry(thetas[param_idx], q[i])
            param_idx += 1
            rz(thetas[param_idx], q[i])
            param_idx += 1
        # Entangling layer
        for i in range(n_qubits - 1):
            x.ctrl(q[i], q[i + 1])
```

### 5. Available Optimizers

| Optimizer | Type | Usage |
|-----------|------|-------|
| `cudaq.optimizers.COBYLA()` | Gradient-free | General purpose |
| `cudaq.optimizers.NelderMead()` | Gradient-free | Simplex method |
| `cudaq.optimizers.LBFGS()` | Gradient-based | Large parameter spaces |
| `cudaq.optimizers.GradientDescent()` | Gradient-based | Simple gradient descent |
| `cudaq.optimizers.Adam()` | Gradient-based | Adaptive learning rate |
| SciPy `minimize()` | External | Full control over optimization |

### 6. Gradient Computation

```python
# Parameter-shift gradient
@cudaq.kernel
def ansatz(theta: float):
    q = cudaq.qubit()
    rx(theta, q)

hamiltonian = spin.z(0)

# CUDA-Q computes gradients via parameter shift rule
gradient = cudaq.gradients.CentralDifference()
optimizer = cudaq.optimizers.LBFGS()

energy, params, data = cudaq.vqe(ansatz, hamiltonian, optimizer,
                                  gradient=gradient, parameter_count=1)
```

## Hamiltonian Construction Patterns

```python
from cudaq import spin

# Molecular: explicit terms
H_mol = 5.907 - 2.1433 * spin.x(0) * spin.x(1) + 0.21829 * spin.z(0)

# Ising model
H_ising = sum(spin.z(i) * spin.z(i+1) for i in range(n-1)) \
        + h_field * sum(spin.x(i) for i in range(n))

# Random for benchmarking
H_random = cudaq.SpinOperator.random(qubit_count=10, term_count=1000)
```

## Common Pitfalls

- **Too few optimization iterations**: VQE often needs 100+ iterations for convergence. Set `max_iterations` appropriately.
- **Bad initial parameters**: Random initialization near 0 often works. Avoid symmetry-trapped initializations.
- **Wrong parameter_count**: Must match the number of float parameters the kernel expects.
- **Using gradient optimizers without gradient support**: COBYLA/NelderMead are safer defaults.

### 7. UCCSD Built-in Ansatz

CUDA-Q provides a built-in UCCSD (Unitary Coupled Cluster Singles and Doubles) ansatz:

```python
@cudaq.kernel
def uccsd_kernel(qubit_num: int, electron_num: int, thetas: list[float]):
    qubits = cudaq.qvector(qubit_num)
    # Hartree-Fock reference state
    for i in range(electron_num):
        x(qubits[i])
    # Built-in UCCSD ansatz
    cudaq.kernels.uccsd(qubits, thetas, electron_num, qubit_num)

# Get parameter count
param_count = cudaq.kernels.uccsd_num_parameters(electron_count, qubit_count)
```

### 8. Active Space Approximation

Reduce qubit and parameter counts by restricting to active orbitals:

```python
ncore = 3              # Frozen core orbitals
nele_cas, norb_cas = (4, 3)  # 4 electrons in 3 active orbitals

electron_count = nele_cas
qubit_count = 2 * norb_cas  # 6 qubits instead of 14

param_count = cudaq.kernels.uccsd_num_parameters(electron_count, qubit_count)
# 8 parameters instead of 140
```

### 9. Trotter-Suzuki Time Evolution

For Hamiltonian simulation via circuit-based Trotterization:

```python
@cudaq.kernel
def trotter_step(state: cudaq.State, dt: float,
                 coefficients: list[complex], words: list[cudaq.pauli_word]):
    qubits = cudaq.qvector(state)
    for i in range(len(coefficients)):
        exp_pauli(coefficients[i].real * dt, qubits, words[i])
```

`exp_pauli(theta, qubits, pauli_word)` applies e^(iθP) for Pauli word P.

## Related Skills

- [kernels-and-gates.md](./kernels-and-gates.md) - Build ansatz circuits
- [sampling-and-observe.md](./sampling-and-observe.md) - Execute and measure
- [multi-gpu-workflows.md](./multi-gpu-workflows.md) - Distribute parameter sweeps
- [dynamics-simulation.md](./dynamics-simulation.md) - Continuous-time Hamiltonian dynamics
