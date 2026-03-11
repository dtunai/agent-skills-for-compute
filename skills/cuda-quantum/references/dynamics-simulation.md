---
title: Dynamics Simulation
impact: HIGH
tags: dynamics, evolve, hamiltonian, time-evolution, open-quantum, lindblad, super-operator, integrator, batch
---

# Skill: Dynamics Simulation

Simulate quantum system dynamics (time evolution) using `cudaq.evolve` with the `dynamics` backend target.

## Quick Pattern

```python
import cudaq
import cupy as cp
import numpy as np
from cudaq import spin, ScalarOperator, Schedule

cudaq.set_target("dynamics")

# Define Hamiltonian
hamiltonian = 0.5 * 6.5 * spin.z(0) + 4.0 * ScalarOperator(lambda t: np.cos(0.5 * t)) * spin.x(0)

# Initial state (ground state density matrix)
rho0 = cudaq.State.from_data(cp.array([[1.0, 0.0], [0.0, 0.0]], dtype=cp.complex128))

# Schedule and evolve
steps = np.linspace(0, 1.0, 100)
schedule = Schedule(steps, ["t"])

result = cudaq.evolve(hamiltonian, {0: 2}, schedule, rho0,
                      observables=[spin.x(0), spin.y(0), spin.z(0)],
                      collapse_operators=[],
                      store_intermediate_results=cudaq.IntermediateResultSave.ALL)
```

## When to Use

- Simulating time evolution of quantum systems (Schrödinger/Lindblad equations)
- Modeling superconducting qubits, spin systems, trapped ions, cavity QED
- Simulating open quantum systems with decoherence (collapse operators)
- Time-dependent Hamiltonians with control pulses
- Batch simulations for parameter sweeps or process tomography
- Multi-GPU distributed dynamics

## Step-by-Step Instructions

### 1. Setup and Basics

```python
import cudaq
import cupy as cp
import numpy as np
from cudaq import spin, ScalarOperator, Schedule

cudaq.set_target("dynamics")

# System: single qubit (dimension 2)
dimensions = {0: 2}

# Initial state as density matrix
rho0 = cudaq.State.from_data(cp.array([[1.0, 0.0], [0.0, 0.0]], dtype=cp.complex128))

# Time-independent Hamiltonian
hamiltonian = 2 * np.pi * 0.1 * spin.x(0)

# Time schedule
steps = np.linspace(0, 10, 101)
schedule = Schedule(steps, ["t"])

# Evolve
result = cudaq.evolve(hamiltonian, dimensions, schedule, rho0,
                      observables=[spin.z(0)],
                      store_intermediate_results=cudaq.IntermediateResultSave.ALL)
```

### 2. Time-Dependent Hamiltonians

Use `ScalarOperator` for time-dependent coefficients:

```python
omega_z = 6.5
omega_x = 4.0
omega_d = 0.5

# H = (ω_z/2) σ_z + ω_x cos(ω_d t) σ_x
hamiltonian = 0.5 * omega_z * spin.z(0) \
    + omega_x * ScalarOperator(lambda t: np.cos(omega_d * t)) * spin.x(0)
```

### 3. Bosonic Operators (Jaynes-Cummings Model)

```python
from cudaq import operators

omega_c = 5.0   # Cavity frequency
omega_a = 4.5   # Atom frequency
Omega = 0.25    # Coupling

# Cavity (index 1) + atom (index 0)
hamiltonian = omega_c * operators.create(1) * operators.annihilate(1) \
    + (omega_a / 2) * spin.z(0) \
    + (Omega / 2) * (operators.annihilate(1) * spin.plus(0) \
                    + operators.create(1) * spin.minus(0))

# Multi-level Fock space: cavity with 10 levels, atom with 2
dimensions = {0: 2, 1: 10}
```

### 4. Builtin Dynamics Operators

| Operator | Description |
|----------|-------------|
| `operators.identity(i)` | Identity |
| `operators.zero(i)` | Zero/null operator |
| `operators.annihilate(i)` | Bosonic annihilation (a) |
| `operators.create(i)` | Bosonic creation (a†) |
| `operators.number(i)` | Number operator (a†a) |
| `operators.parity(i)` | Parity (exp(iπ a†a)) |
| `operators.displace(i)` | Displacement D(α) |
| `operators.squeeze(i)` | Squeezing S(z) |
| `operators.position(i)` | Position (a† + a)/2 |
| `operators.momentum(i)` | Momentum i(a† - a)/2 |
| `spin.x(i)` | Pauli σ_x |
| `spin.y(i)` | Pauli σ_y |
| `spin.z(i)` | Pauli σ_z |
| `spin.plus(i)` | Raising σ_+ |
| `spin.minus(i)` | Lowering σ_- |

### 5. Open Quantum Systems (Lindblad)

Add collapse operators for decoherence:

```python
gamma = 0.1  # Decay rate

# Collapse operator: sqrt(gamma) * sigma_minus
collapse_ops = [np.sqrt(gamma) * spin.minus(0)]

result = cudaq.evolve(hamiltonian, dimensions, schedule, rho0,
                      observables=[spin.z(0)],
                      collapse_operators=collapse_ops,
                      store_intermediate_results=cudaq.IntermediateResultSave.ALL)
```

### 6. Custom Operators

Define custom matrix operators with `operators.define`:

```python
from cudaq import operators, NumericType
import numpy as np
import scipy

def displacement_matrix(dimension: int, displacement: NumericType):
    displacement = complex(displacement)
    term1 = displacement * operators.create(0).to_matrix({0: dimension})
    term2 = np.conjugate(displacement) * operators.annihilate(0).to_matrix({0: dimension})
    return scipy.linalg.expm(term1 - term2)

# Register: acts on 1 degree of freedom, any dimension
operators.define("displace", [0], displacement_matrix)

# Instantiate
disp_op = operators.instantiate("displace", [0])
```

### 7. Super-Operator Representation

For arbitrary state evolution equations beyond Lindblad:

```python
from cudaq import SuperOperator, RungeKuttaIntegrator

hamiltonian = 2.0 * np.pi * 0.1 * spin.x(0)

# Schrödinger equation as super-operator: d|ψ>/dt = -iH|ψ>
se_super_op = SuperOperator()
se_super_op += SuperOperator.left_multiply(-1j * hamiltonian)

result = cudaq.evolve(se_super_op, {0: 2}, schedule,
                      cudaq.dynamics.InitialState.ZERO,
                      observables=[spin.z(0)],
                      store_intermediate_results=True,
                      integrator=RungeKuttaIntegrator())
```

### 8. Numerical Integrators

| Integrator (Python) | Description |
|---------------------|-------------|
| `RungeKuttaIntegrator` | 4th-order Runge-Kutta (default) |
| `ScipyZvodeIntegrator` | Complex-valued ODE solver (SciPy) |
| `CUDATorchDiffEqDopri5Integrator` | RK5 Dormand-Prince (torchdiffeq) |
| `CUDATorchDiffEqAdaptiveHeunIntegrator` | RK2 Adaptive Heun (torchdiffeq) |
| `CUDATorchDiffEqBosh3Integrator` | RK3 Bogacki-Shampine (torchdiffeq) |
| `CUDATorchDiffEqDopri8Integrator` | RK8 Dormand-Prince (torchdiffeq) |
| `CUDATorchDiffEqEulerIntegrator` | Euler method (torchdiffeq) |
| `CUDATorchDiffEqRK4Integrator` | RK4 with 3/8 rule (torchdiffeq) |

Torch-based integrators require `pip install torchdiffeq` with CUDA-enabled PyTorch.

### 9. Batch Simulation

Batch multiple initial states or Hamiltonians for parallel execution:

```python
# Multiple initial states with same Hamiltonian
sic_states = [psi_1, psi_2, psi_3, psi_4]

results = cudaq.evolve(hamiltonian, dimensions, schedule, sic_states,
                       observables=[spin.x(0), spin.y(0), spin.z(0)],
                       store_intermediate_results=cudaq.IntermediateResultSave.EXPECTATION_VALUE,
                       integrator=RungeKuttaIntegrator())

# Multiple Hamiltonians (e.g., frequency sweep)
hamiltonians = [
    0.5 * omega_z * spin.z(0) + omega_x * ScalarOperator(lambda t, w=w: np.cos(w * t)) * spin.x(0)
    for w in omega_drive
]
initial_states = [psi0] * len(hamiltonians)

results = cudaq.evolve(hamiltonians, dimensions, schedule, initial_states,
                       observables=[spin.x(0), spin.y(0), spin.z(0)],
                       store_intermediate_results=cudaq.IntermediateResultSave.EXPECTATION_VALUE,
                       max_batch_size=2)  # Control memory usage
```

**Batching rules**: Hamiltonians must have same structure (same number of product terms, same degrees of freedom). Different coefficients/callbacks/operators are OK. Non-batchable Hamiltonians run sequentially with a warning.

### 10. Multi-GPU Dynamics

```python
cudaq.mpi.initialize()
cudaq.set_target("dynamics")

result = cudaq.evolve(hamiltonian, dimensions, schedule,
                      cudaq.dynamics.InitialState.ZERO,
                      integrator=RungeKuttaIntegrator())

cudaq.mpi.finalize()
```

```bash
mpiexec -np <N> python dynamics_program.py
```

Number of MPI processes must be a power of 2, one GPU per process.

### 11. Retrieving Results

```python
# Final state
final_state = result.state()

# Expectation values at each time step
get_result = lambda idx, res: [exp_vals[idx].expectation() for exp_vals in res.expectation_values()]

exp_x = get_result(0, result)  # <σ_x> over time
exp_y = get_result(1, result)  # <σ_y> over time
exp_z = get_result(2, result)  # <σ_z> over time
```

## IntermediateResultSave Options

| Option | Saves |
|--------|-------|
| `NONE` | Only final state (default) |
| `EXPECTATION_VALUE` | Expectation values at each step |
| `ALL` | States + expectation values (memory-intensive) |

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CUDAQ_DYNAMICS_MIN_MULTIDIAGONAL_DIMENSION` | 4 | Min dimension for sparse format |
| `CUDAQ_DYNAMICS_MAX_DIAGONAL_COUNT_FOR_MULTIDIAGONAL` | 1 | Max diagonals for sparse storage |

## Common Pitfalls

- **Missing `cudaq.set_target("dynamics")`**: The dynamics backend must be explicitly set.
- **Wrong dimension map**: Each degree of freedom needs its dimension specified (e.g., `{0: 2}` for a qubit).
- **Memory with `store_intermediate_results=ALL`**: Storing all intermediate states is memory-intensive for large systems. Use `EXPECTATION_VALUE` if you only need observables.
- **Torch integrators without CUDA**: Torch-based integrators require CUDA-enabled PyTorch. Verify with `python3 -c "import torch; print(torch.version.cuda)"`.
- **Non-power-of-2 MPI ranks**: Multi-GPU dynamics requires power-of-2 process count.
- **Batch structure mismatch**: Hamiltonians with different numbers of terms cannot be batched.

## Related Skills

- [kernels-and-gates.md](./kernels-and-gates.md) - Circuit-based quantum programming
- [sampling-and-observe.md](./sampling-and-observe.md) - Circuit execution and measurement
- [multi-gpu-workflows.md](./multi-gpu-workflows.md) - Multi-GPU circuit simulation
