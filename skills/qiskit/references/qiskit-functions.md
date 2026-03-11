# Qiskit Functions and Circuit Functions Reference

Sources:
- [Introduction to Qiskit Functions](https://quantum.cloud.ibm.com/docs/en/guides/functions)
- [Qiskit Functions Catalog Announcement](https://www.ibm.com/quantum/blog/qiskit-functions-catalog)
- [Qiskit Functions 2026 Updates](https://www.ibm.com/quantum/blog/functions-2026)
- [Qiskit Serverless](https://github.com/Qiskit/qiskit-serverless)

## Overview

**Qiskit Functions** streamline quantum algorithm development by abstracting complex workflow components including transpilation, error suppression, error mitigation, and classical-quantum orchestration.

## Two Function Categories

### Circuit Functions

**Purpose:** Simplified interface for running quantum circuits on IBM Quantum hardware.

**Key Features:**
- Automatic transpilation to ISA
- Built-in error suppression
- Built-in error mitigation
- Optimized for algorithm developers

**Input:** Abstract PUB objects (circuits + observables + parameters)

**Output:** Mitigated expectation values or measurement outcomes

**Target Users:** Quantum researchers developing algorithms without managing hardware optimization details.

### Application Functions

**Purpose:** Domain-specific quantum solutions with classical I/O.

**Key Features:**
- High-level domain abstractions
- Classical input/output
- End-to-end workflows
- Optimized for non-quantum experts

**Input:** Classical problem specifications (molecules, optimization graphs, financial portfolios)

**Output:** Classical results (energies, solutions, predictions)

**Target Users:** Domain experts integrating quantum capabilities into classical workflows.

## Qiskit Functions Catalog

### Available Functions (2026)

| Function | Provider | Category | Use Case |
|----------|----------|----------|----------|
| **Tensor-Network Error Mitigation** | Algorithmiq | Circuit Function | Low-weight observables |
| **QESEM** | Qedma | Circuit Function | Parameterized gates, high-weight observables |
| **Performance Management** | Q-CTRL | Circuit Function | AI-driven error suppression |
| **HI-VQE Chemistry** | Qunova | Application Function | Molecular electronic structure |
| **Quantum Portfolio Optimizer** | Global Data Quantum | Application Function | Financial optimization |
| **Iskay Quantum Optimizer** | Kipu | Application Function | Scheduling and routing |
| **Circuit Function** | IBM | Circuit Function | General-purpose circuit execution |

### Function Capabilities

**Circuit Functions:**
- Advanced error mitigation (beyond standard Runtime options)
- Specialized optimization for specific circuit types
- Hardware-aware compilation
- Noise-adaptive execution strategies

**Application Functions:**
- Molecular Hamiltonian construction (chemistry)
- QUBO formulation (optimization)
- Portfolio constraint handling (finance)
- Domain-specific post-processing

## Installation and Setup

### Install Qiskit Functions Client

```bash
pip install qiskit-ibm-catalog
```

### Authentication

```python
from qiskit_ibm_catalog import QiskitFunctionsCatalog

# Initialize catalog
catalog = QiskitFunctionsCatalog()

# Authenticate (uses IBM Quantum Platform credentials)
catalog.save_account(token="YOUR_IBM_QUANTUM_TOKEN")

# Or use environment variable
# export QISKIT_IBM_TOKEN="YOUR_TOKEN"
```

### Browse Catalog

```python
# List available functions
functions = catalog.list()

for func in functions:
    print(f"{func.name}: {func.description}")
    print(f"  Provider: {func.provider}")
    print(f"  Category: {func.category}")
```

## Using Circuit Functions

### Basic Pattern

```python
from qiskit import QuantumCircuit
from qiskit.quantum_info import SparsePauliOp
from qiskit_ibm_catalog import QiskitFunctionsCatalog

# Load circuit function
catalog = QiskitFunctionsCatalog()
circuit_function = catalog.load("circuit-function")

# Create circuit and observable
qc = QuantumCircuit(4)
qc.h(0)
qc.cx(0, 1)
qc.cx(1, 2)
qc.cx(2, 3)

observable = SparsePauliOp.from_list([("ZZZZ", 1.0)])

# Run function
job = circuit_function.run(
    circuit=qc,
    observable=observable,
    backend="ibm_torino",
    shots=10000
)

# Get results
result = job.result()
expectation_value = result.evs[0]
print(f"Expectation value: {expectation_value}")
```

### Advanced Error Mitigation (Algorithmiq)

```python
# Load Algorithmiq tensor-network mitigation
tn_function = catalog.load("algorithmiq-tem")

# Configure mitigation
job = tn_function.run(
    circuit=qc,
    observable=observable,
    backend="ibm_torino",
    options={
        "max_bond_dimension": 64,  # Tensor network truncation
        "noise_model": "device",    # Use device noise characterization
        "mitigation_level": "heavy"
    }
)

result = job.result()
```

### AI-Driven Optimization (Q-CTRL)

```python
# Load Q-CTRL performance management
qctrl_function = catalog.load("qctrl-performance")

# AI-optimized execution
job = qctrl_function.run(
    circuit=qc,
    observable=observable,
    backend="ibm_torino",
    options={
        "optimization_strategy": "ai",
        "target_fidelity": 0.95
    }
)

result = job.result()
```

## Using Application Functions

### Molecular Chemistry (Qunova HI-VQE)

```python
# Load HI-VQE chemistry function
hi_vqe = catalog.load("qunova-hi-vqe")

# Classical input: molecular structure
molecule = {
    "atoms": ["H", "H"],
    "coordinates": [[0.0, 0.0, 0.0], [0.0, 0.0, 0.74]],
    "charge": 0,
    "multiplicity": 1
}

# Run chemistry calculation
job = hi_vqe.run(
    molecule=molecule,
    basis="sto-3g",
    backend="ibm_kyiv"
)

result = job.result()
print(f"Ground state energy: {result.energy} Ha")
print(f"Orbital energies: {result.orbital_energies}")
```

### Portfolio Optimization (Global Data Quantum)

```python
# Load portfolio optimizer
portfolio_opt = catalog.load("gdq-portfolio-optimizer")

# Classical input: financial data
portfolio = {
    "assets": ["AAPL", "GOOGL", "MSFT", "TSLA"],
    "returns": [0.12, 0.15, 0.10, 0.20],
    "covariance_matrix": [[...], [...], [...], [...]],
    "budget": 1000000,
    "constraints": {
        "max_weight": 0.4,
        "min_weight": 0.1
    }
}

# Run optimization
job = portfolio_opt.run(
    portfolio=portfolio,
    risk_tolerance=0.5,
    backend="ibm_torino"
)

result = job.result()
print(f"Optimal allocation: {result.weights}")
print(f"Expected return: {result.expected_return}")
print(f"Portfolio risk: {result.risk}")
```

### Combinatorial Optimization (Kipu Iskay)

```python
# Load Iskay optimizer
iskay = catalog.load("kipu-iskay")

# QUBO problem (vehicle routing)
qubo = {
    "Q": qubo_matrix,  # Quadratic coefficients
    "linear": linear_terms,
    "offset": constant_term
}

# Run optimization
job = iskay.run(
    problem=qubo,
    num_reads=1000,
    backend="ibm_torino",
    options={
        "algorithm": "qaoa",
        "reps": 3
    }
)

result = job.result()
print(f"Best solution: {result.best_solution}")
print(f"Energy: {result.energy}")
```

## Job Management

### Submit Job

```python
job = function.run(...)

# Job metadata
print(f"Job ID: {job.job_id()}")
print(f"Status: {job.status()}")
```

### Monitor Progress

```python
# Wait for completion
result = job.result()  # Blocks until done

# Non-blocking status check
status = job.status()
print(status)  # QUEUED, RUNNING, DONE, ERROR
```

### Job Status Stages

| Status | Description |
|--------|-------------|
| **QUEUED** | Waiting for resources |
| **INITIALIZING** | Setting up execution environment |
| **OPTIMIZING** | Transpiling and optimizing circuits |
| **RUNNING** | Executing on quantum hardware |
| **POST_PROCESSING** | Applying error mitigation |
| **DONE** | Completed successfully |
| **ERROR** | Failed with error message |

### Retrieve Results

```python
# Block until completion
result = job.result()

# Check if completed first
if job.status() == "DONE":
    result = job.result()
```

## Qiskit Serverless

### Overview

**Qiskit Serverless** is the infrastructure powering Qiskit Functions.

**Key Features:**
- Hybrid quantum-classical orchestration
- Automatic resource management
- Distributed execution
- Function-as-a-Service (FaaS) model

### Creating Custom Functions

```python
from qiskit_serverless import QiskitFunction, get_arguments, save_result

# Define function
def my_function():
    # Get input arguments
    args = get_arguments()
    circuit = args.get("circuit")
    backend = args.get("backend")

    # Hybrid quantum-classical workflow
    # ... transpile, execute, post-process ...

    # Save result
    save_result({"expectation_value": 0.5})

# Register function
my_serverless_function = QiskitFunction(
    title="my-custom-function",
    entrypoint=my_function,
    dependencies=["qiskit>=1.0", "numpy"]
)

# Upload to catalog (requires authorization)
catalog.upload(my_serverless_function)
```

### Distributed Execution

```python
from qiskit_serverless import distribute_task

@distribute_task
def parallel_vqe_iteration(params):
    # Each iteration runs on separate worker
    result = run_vqe_step(params)
    return result

# Map across parameter grid
results = [parallel_vqe_iteration(p) for p in param_grid]
```

## Function Configuration

### Backend Selection

```python
# Specify backend
job = function.run(
    circuit=qc,
    observable=obs,
    backend="ibm_kyiv"  # or "ibm_torino", "ibm_osaka"
)

# Let function choose optimal backend
job = function.run(
    circuit=qc,
    observable=obs,
    backend="auto"
)
```

### Error Mitigation Options

```python
job = function.run(
    circuit=qc,
    observable=obs,
    backend="ibm_torino",
    options={
        "resilience_level": 2,  # Standard Runtime mitigation
        "optimization_level": 3,
        "dynamical_decoupling": {
            "enable": True,
            "sequence_type": "XY4"
        }
    }
)
```

### Shot Budget

```python
job = function.run(
    circuit=qc,
    observable=obs,
    backend="ibm_torino",
    shots=20000  # Total shot budget
)
```

## Pricing and Credits

### Cost Model

Functions consume IBM Quantum credits based on:
- Circuit depth
- Qubit count
- Shot count
- Error mitigation level
- Backend choice

### Check Cost Before Execution

```python
# Estimate cost (if supported by function)
estimate = function.estimate_cost(
    circuit=qc,
    observable=obs,
    backend="ibm_torino",
    shots=10000
)

print(f"Estimated credits: {estimate.credits}")
```

## Best Practices

1. **Use circuit functions for algorithm development**: Focus on circuits, not hardware
2. **Use application functions for domain problems**: Let experts handle quantum details
3. **Start with small test jobs**: Verify functionality before large runs
4. **Monitor job status**: Check for errors early
5. **Choose appropriate backends**: Match qubit count and connectivity to problem
6. **Set realistic shot budgets**: More shots = higher cost but better statistics
7. **Leverage advanced mitigation**: Functions offer beyond-standard Runtime capabilities
8. **Review function documentation**: Each function has specific input requirements
9. **Cache results**: Avoid re-running identical jobs
10. **Use async execution**: Submit multiple jobs concurrently

## Common Pitfalls

- **Wrong input format**: Application functions expect domain-specific structures
- **Insufficient shots**: Low shot counts yield noisy results
- **Exceeding qubit limits**: Ensure circuit fits on selected backend
- **Ignoring job status**: Check for errors before calling `result()`
- **Over-optimizing inputs**: Let functions handle transpilation and optimization
- **Mixing function types**: Circuit functions need PUBs; application functions need domain inputs
- **Not handling exceptions**: Jobs can fail; wrap in try-except
- **Forgetting to authenticate**: Save credentials before using catalog

## Example: Complete Workflow

```python
from qiskit import QuantumCircuit
from qiskit.circuit.library import TwoLocal
from qiskit.quantum_info import SparsePauliOp
from qiskit_ibm_catalog import QiskitFunctionsCatalog
import numpy as np

# Initialize catalog
catalog = QiskitFunctionsCatalog()
circuit_function = catalog.load("circuit-function")

# Define problem (VQE for H2)
H = SparsePauliOp.from_list([
    ("II", -1.052),
    ("IZ", 0.398),
    ("ZI", -0.398),
    ("ZZ", -0.011),
    ("XX", 0.181)
])

# Create ansatz
ansatz = TwoLocal(num_qubits=2, rotation_blocks=['ry', 'rz'],
                  entanglement_blocks='cx', reps=3)

# VQE loop
def vqe_objective(params):
    bound_circuit = ansatz.assign_parameters(params)
    job = circuit_function.run(
        circuit=bound_circuit,
        observable=H,
        backend="ibm_kyiv",
        shots=10000
    )
    result = job.result()
    return result.evs[0]

# Classical optimization
from scipy.optimize import minimize
initial_params = np.random.rand(ansatz.num_parameters)
result = minimize(vqe_objective, initial_params, method='COBYLA')

print(f"Ground state energy: {result.fun}")
print(f"Optimal parameters: {result.x}")
```

## Future Developments

- **Expanded catalog**: More functions from IBM and partners
- **Custom function marketplace**: Upload and share functions
- **Enhanced serverless features**: Better distributed execution
- **Real-time collaboration**: Multi-user function development
- **Integration with Qiskit Patterns**: Unified workflow framework

## Resources

- **Catalog**: [quantum.cloud.ibm.com/catalog](https://quantum.cloud.ibm.com/catalog)
- **Documentation**: [quantum.cloud.ibm.com/docs/en/guides/functions](https://quantum.cloud.ibm.com/docs/en/guides/functions)
- **GitHub**: [github.com/Qiskit/qiskit-serverless](https://github.com/Qiskit/qiskit-serverless)
- **Tutorials**: Function-specific guides in catalog
