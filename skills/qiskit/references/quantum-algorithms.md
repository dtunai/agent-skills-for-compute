# Qiskit Quantum Algorithms Reference

Sources:
- [Qiskit Algorithms Documentation](https://qiskit-community.github.io/qiskit-algorithms/)
- [IBM Quantum Learning - Variational Quantum Algorithms](https://quantum.cloud.ibm.com/learning/en/courses/utility-scale-quantum-computing/variational-quantum-algorithms)

## Overview

Qiskit provides domain-independent quantum algorithms through the `qiskit-algorithms` package, including variational algorithms (VQE, QAOA), search algorithms (Grover), factorization (Shor), and quantum phase estimation.

## Variational Quantum Eigensolver (VQE)

### Purpose

VQE finds the ground state energy of a quantum Hamiltonian using a hybrid quantum-classical optimization approach. Essential for NISQ devices due to resilience to certain noise types.

### Key Components

1. **Ansatz**: Parametrized quantum circuit representing trial wavefunction
2. **Hamiltonian**: Observable operator (typically `SparsePauliOp`)
3. **Optimizer**: Classical optimization routine (COBYLA, SLSQP, SPSA)
4. **Estimator**: Primitive for computing expectation values

### Implementation Pattern

```python
from qiskit_algorithms import VQE
from qiskit_algorithms.optimizers import SLSQP
from qiskit.circuit.library import TwoLocal
from qiskit.quantum_info import SparsePauliOp
from qiskit.primitives import Estimator

# Define Hamiltonian (e.g., H2 molecule)
H = SparsePauliOp.from_list([
    ("II", -1.052373245772859),
    ("IZ", 0.39793742484318045),
    ("ZI", -0.39793742484318045),
    ("ZZ", -0.01128010425623538),
    ("XX", 0.18093119978423156)
])

# Create ansatz
ansatz = TwoLocal(
    num_qubits=2,
    rotation_blocks=['ry', 'rz'],
    entanglement_blocks='cx',
    entanglement='linear',
    reps=2
)

# Initialize VQE
estimator = Estimator()
optimizer = SLSQP(maxiter=100)
vqe = VQE(estimator, ansatz, optimizer)

# Run optimization
result = vqe.compute_minimum_eigenvalue(H)

print(f"Ground state energy: {result.eigenvalue}")
print(f"Optimal parameters: {result.optimal_parameters}")
print(f"Optimizer evaluations: {result.cost_function_evals}")
```

### Ansatz Selection

**Circuit Library Options:**

| Ansatz | Description | Use Case |
|--------|-------------|----------|
| `TwoLocal` | Layered rotation + entanglement | General-purpose VQE |
| `RealAmplitudes` | RY rotations + CX | Chemistry, real-valued states |
| `EfficientSU2` | RY, RZ rotations + CX | Highly expressive, low depth |
| `ExcitationPreserving` | Particle-conserving gates | Fermionic systems |

### Optimizer Selection

| Optimizer | Type | Best For |
|-----------|------|----------|
| `COBYLA` | Gradient-free | Noisy evaluation, constrained |
| `SLSQP` | Gradient-based | Smooth landscapes |
| `SPSA` | Gradient approximation | Noisy hardware, fast convergence |
| `L_BFGS_B` | Quasi-Newton | Large parameter spaces |

## Quantum Approximate Optimization Algorithm (QAOA)

### Purpose

Hybrid quantum-classical algorithm for combinatorial optimization problems (MaxCut, TSP, graph coloring). Alternates between cost and mixer Hamiltonians.

### Structure

**Cost Hamiltonian (Problem):**
Encodes optimization objective as quantum operator.

**Mixer Hamiltonian:**
Explores solution space (typically X gates).

**Layers (p):**
Repetitions of cost + mixer operations (higher p = better approximation).

### Implementation Pattern

```python
from qiskit_algorithms import QAOA
from qiskit_algorithms.optimizers import COBYLA
from qiskit.quantum_info import SparsePauliOp
from qiskit.primitives import Sampler

# Define MaxCut problem on 3-node graph
# Edges: (0,1), (1,2), (0,2)
cost_hamiltonian = SparsePauliOp.from_list([
    ("ZZI", 0.5),
    ("IZZ", 0.5),
    ("ZIZ", 0.5)
])

# Create QAOA
sampler = Sampler()
optimizer = COBYLA()
qaoa = QAOA(sampler=sampler, optimizer=optimizer, reps=2)

# Run optimization
result = qaoa.compute_minimum_eigenvalue(cost_hamiltonian)

print(f"Optimal value: {result.eigenvalue}")
print(f"Optimal bitstring: {result.best_measurement}")
```

### QAOA Variants

**Warm-Start QAOA:**
Initialize with classical solution for better convergence.

**Multi-Angle QAOA:**
Independent parameters per layer for flexibility.

**Recursive QAOA:**
Iteratively eliminate variables to reduce problem size.

## Grover's Search Algorithm

### Purpose

Quadratically faster unstructured search: O(√N) vs. classical O(N).

### Components

1. **Oracle**: Marks target states with phase flip
2. **Diffusion Operator**: Amplifies marked states
3. **Iterations**: ~π/4 × √N applications

### Implementation Pattern

```python
from qiskit.circuit.library import GroverOperator
from qiskit_algorithms import AmplificationProblem, Grover
from qiskit.primitives import Sampler

# Define oracle (marking state |11⟩)
from qiskit import QuantumCircuit
oracle = QuantumCircuit(2)
oracle.cz(0, 1)

# Create Grover instance
problem = AmplificationProblem(oracle, is_good_state=lambda x: x == '11')
grover = Grover(sampler=Sampler())
result = grover.amplify(problem)

print(f"Result: {result.top_measurement}")
```

## Shor's Factorization Algorithm

### Purpose

Exponentially faster integer factorization using Quantum Fourier Transform (QFT).

### Key Steps

1. **Classical preprocessing**: Choose random a < N
2. **Period finding**: Use QFT to find period r of f(x) = a^x mod N
3. **Classical postprocessing**: Extract factors from period

### QFT Pattern

```python
from qiskit.circuit.library import QFT

# 4-qubit QFT
qft = QFT(num_qubits=4, approximation_degree=0, do_swaps=True)

# Inverse QFT (for period finding)
iqft = qft.inverse()
```

## Quantum Phase Estimation (QPE)

### Purpose

Estimate eigenvalues of unitary operators. Foundation for Shor's algorithm and quantum chemistry.

### Pattern

```python
from qiskit_algorithms import PhaseEstimation
from qiskit import QuantumCircuit
from qiskit.quantum_info import Operator

# Define unitary (e.g., T gate: e^(iπ/4))
unitary = QuantumCircuit(1)
unitary.t(0)
U = Operator(unitary)

# Eigenstate preparation
state_prep = QuantumCircuit(1)
state_prep.x(0)

# Phase estimation
pe = PhaseEstimation(num_evaluation_qubits=4, unitary=U)
result = pe.estimate(state_preparation=state_prep)
```

## Amplitude Estimation

### Purpose

Quantum speedup for Monte Carlo estimation. Quadratic advantage in sample complexity.

### Application

```python
from qiskit_algorithms import AmplitudeEstimation
from qiskit.primitives import Sampler

# Define amplitude function
from qiskit.circuit.library import LinearAmplitudeFunction
a_factory = LinearAmplitudeFunction(
    num_state_qubits=3,
    slope=0.25,
    offset=0.5
)

# Run amplitude estimation
ae = AmplitudeEstimation(
    num_eval_qubits=3,
    sampler=Sampler()
)
result = ae.estimate(a_factory)

print(f"Estimated amplitude: {result.estimation}")
print(f"Confidence interval: {result.confidence_interval}")
```

## HHL (Harrow-Hassidim-Lloyd) Algorithm

### Purpose

Exponential speedup for solving linear systems Ax = b.

### Requirements

- Sparse, well-conditioned matrix A
- Efficient oracle for A
- Access to quantum RAM (theoretical)

### Limitations

**Readout bottleneck**: Extracting solution requires measurements proportional to solution size, limiting practical advantage.

## Variational Quantum Linear Solver (VQLS)

### Purpose

Hybrid alternative to HHL for NISQ devices. Solves Ax = b variationally.

### Pattern

```python
from qiskit_algorithms.linear_solvers import VQLS
from qiskit.circuit.library import RealAmplitudes

# Define matrix A and vector b
A = [[1, 0.2], [0.2, 1]]
b = [1, -2.1]

# Create ansatz
ansatz = RealAmplitudes(num_qubits=1, reps=3)

# Solve
vqls = VQLS(ansatz, optimizer=SLSQP())
result = vqls.solve(A, b)
```

## Circuit Library

Qiskit provides pre-built circuits for common algorithms:

| Circuit | Purpose |
|---------|---------|
| `QFT` | Quantum Fourier Transform |
| `GroverOperator` | Grover diffusion + oracle |
| `PhaseEstimation` | Eigenvalue estimation |
| `HiddenLinearFunction` | BQP-complete problem |
| `IQP` | Instantaneous Quantum Polynomial |
| `EfficientSU2` | Hardware-efficient ansatz |
| `PauliEvolutionGate` | Hamiltonian time evolution |

## Algorithm Selection Guide

| Problem Type | Algorithm | Quantum Advantage |
|--------------|-----------|-------------------|
| Ground state energy | VQE | NISQ-compatible, polynomial speedup |
| Combinatorial optimization | QAOA | Approximate, problem-dependent |
| Unstructured search | Grover | Quadratic speedup |
| Integer factorization | Shor | Exponential speedup (BQP vs NP) |
| Linear systems | HHL/VQLS | Exponential (with caveats) |
| Monte Carlo sampling | Amplitude Estimation | Quadratic speedup |

## NISQ vs. Fault-Tolerant

**NISQ Algorithms (Current Hardware):**
- VQE, QAOA, variational algorithms
- Short circuit depth
- Resilient to moderate noise
- Hybrid quantum-classical

**Fault-Tolerant (Future Hardware):**
- Shor, Grover, HHL
- Deep circuits requiring error correction
- Exponential speedups
- Requires logical qubits

## Best Practices

1. **Start with statevector simulation** before hardware
2. **Use optimizer callbacks** to monitor convergence
3. **Set initial parameters** near expected optimum (warm-start)
4. **Increase ansatz depth gradually** to avoid barren plateaus
5. **Choose gradient-free optimizers** for noisy hardware
6. **Benchmark against classical** to verify quantum advantage
7. **Use circuit transpilation** before algorithm execution
8. **Monitor shot noise** in sampler-based algorithms
