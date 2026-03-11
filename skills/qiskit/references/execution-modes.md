# Qiskit Execution Modes Reference

Sources:
- [Introduction to Execution Modes](https://docs.quantum.ibm.com/guides/execution-modes)
- [Choose the Right Execution Mode](https://docs.quantum.ibm.com/guides/choose-execution-mode)
- [Run Jobs in Session](https://docs.quantum.ibm.com/guides/run-jobs-session)
- [Run Jobs in Batch](https://docs.quantum.ibm.com/guides/run-jobs-batch)

## Overview

Qiskit Runtime offers three execution modes for running quantum workloads on IBM Quantum hardware: **Job**, **Batch**, and **Session**. Each mode optimizes for different use cases.

## Execution Modes Summary

| Mode | Use Case | Queue Priority | Cost | Dedicated Access |
|------|----------|----------------|------|------------------|
| **Job** | Single primitive request | Standard | Standard | No |
| **Batch** | Multiple independent jobs | Lower | Discounted | No |
| **Session** | Iterative workloads | Higher | Premium | Yes |

## Job Mode

### Definition

A single primitive request (Sampler or Estimator) submitted without a context manager.

### When to Use

- Single circuit execution
- One-off experiments
- Testing and debugging
- Non-iterative workloads

### Usage Pattern

```python
from qiskit_ibm_runtime import QiskitRuntimeService, SamplerV2

service = QiskitRuntimeService()
backend = service.least_busy(operational=True, simulator=False)

# Job mode: direct primitive instantiation
sampler = SamplerV2(mode=backend)
job = sampler.run([isa_circuit])
result = job.result()
```

### Characteristics

- **Queue behavior**: Standard FIFO queue
- **Execution**: Single job, immediate execution when resources available
- **Cost**: Standard per-job pricing
- **Isolation**: No dedicated access, shares QPU with other users

## Batch Mode

### Definition

A multi-job manager for running independent jobs efficiently without blocking the queue.

### When to Use

- Multiple independent circuits
- Parameter sweeps (non-iterative)
- Parallel experiments
- High-throughput screening

### Usage Pattern

```python
from qiskit_ibm_runtime import Batch, SamplerV2

service = QiskitRuntimeService()
backend = service.backend('ibm_torino')

# Batch context manager
with Batch(backend=backend) as batch:
    sampler = SamplerV2(mode=batch)

    # Submit multiple jobs
    job1 = sampler.run([circuit1])
    job2 = sampler.run([circuit2])
    job3 = sampler.run([circuit3])

# Wait outside batch
result1 = job1.result()
result2 = job2.result()
result3 = job3.result()
```

### Characteristics

- **Queue behavior**: Lower priority, waits for idle time
- **Execution**: Jobs run when QPU is available
- **Cost**: Discounted pricing (lower than job mode)
- **Parallelism**: Jobs submitted concurrently, executed when resources allow
- **No blocking**: Other users' jobs can interleave

### Best Practices

**Do:**
- Submit all jobs within the batch context
- Use for independent, non-conditional jobs
- Leverage for large parameter sweeps

**Don't:**
- Use for iterative algorithms (use session instead)
- Mix batch with other modes for same backend
- Expect FIFO execution order

## Session Mode

### Definition

A dedicated window providing exclusive QPU access for iterative multi-job workloads.

### When to Use

- Variational algorithms (VQE, QAOA)
- Iterative optimization loops
- Adaptive circuits
- Experiments requiring predictable timing

### Usage Pattern

```python
from qiskit_ibm_runtime import Session, EstimatorV2
from scipy.optimize import minimize

service = QiskitRuntimeService()
backend = service.backend('ibm_kyiv')

# Session context manager
with Session(backend=backend) as session:
    estimator = EstimatorV2(mode=session)

    # VQE optimization loop
    def cost_function(params):
        bound_circuit = ansatz.assign_parameters(params)
        job = estimator.run([(bound_circuit, observable)])
        result = job.result()
        return result[0].data.evs

    # Iterative optimization within session
    result = minimize(cost_function, initial_params, method='COBYLA')

# Session closes automatically
```

### Characteristics

- **Queue behavior**: Higher priority, reserved time slot
- **Execution**: Dedicated access during session window
- **Cost**: Premium pricing (higher than job/batch)
- **Exclusivity**: No other jobs run on QPU during session
- **TTL (Time to Live)**: Session expires after inactivity (default: 300s)

### Session Configuration

```python
# Configure session TTL
session = Session(backend=backend, max_time='1h')

# Check session status
print(session.status())  # 'Pending', 'In progress', 'Closed'

# Close session manually
session.close()
```

### Interactive Time Limit (ITTL)

**Maximum interactive time**: Limit on total active QPU time.

```python
# Session closes when ITTL exhausted
# Current limits: ~2 hours for Premium Plan
```

## Choosing the Right Mode

### Decision Tree

```
Single job?
  └─> YES → Job mode

Multiple jobs?
  └─> YES → Independent jobs?
             └─> YES → Batch mode
             └─> NO (iterative) → Session mode
```

### Use Case Examples

**Job Mode:**
- Algorithm prototyping
- Single circuit benchmark
- Testing transpilation

**Batch Mode:**
- Noise characterization across parameter sweep
- Randomized benchmarking (many independent circuits)
- Tomography experiments
- Grid search over hyperparameters

**Session Mode:**
- VQE energy minimization
- QAOA optimization
- Quantum-classical hybrid loops
- Adaptive error mitigation
- Experiments requiring consistent QPU calibration

## Advanced Usage

### Mixing Primitives in Session

```python
with Session(backend=backend) as session:
    sampler = SamplerV2(mode=session)
    estimator = EstimatorV2(mode=session)

    # Use both primitives in same session
    job1 = sampler.run([circuit1])
    job2 = estimator.run([(circuit2, obs)])
```

### Nested Context Managers

**Not Supported:**
```python
# Invalid: cannot nest sessions/batches
with Session(backend=backend1):
    with Batch(backend=backend2):  # ERROR
        ...
```

**Valid: Sequential**
```python
# Run session, then batch
with Session(backend=backend):
    ...

with Batch(backend=backend):
    ...
```

## Job Status and Monitoring

### Check Job Status

```python
job = sampler.run([circuit])

# Job status
print(job.status())  # 'QUEUED', 'RUNNING', 'DONE', 'ERROR'

# Wait for completion
result = job.result()  # Blocks until done

# Non-blocking check
if job.status() == 'DONE':
    result = job.result()
```

### Session Status

```python
# Within session
print(session.status())
# 'Pending': Waiting for resources
# 'In progress': Running jobs
# 'Closed': Session ended
```

## Cost Optimization

### Pricing Comparison (Relative)

| Mode | Cost per Job | Total Cost (10 jobs) |
|------|--------------|----------------------|
| Job | 1.0× | 10.0× |
| Batch | 0.6× | 6.0× |
| Session | 1.5× | 15.0× |

**Note**: Actual pricing depends on circuit depth, qubit count, and backend.

### When to Use Each for Cost

**Batch for lowest total cost:**
- Independent jobs
- No time constraints
- Can tolerate variable execution timing

**Session for predictable cost:**
- Iterative workflows where consistency matters
- Time-sensitive experiments
- Premium plans with included session credits

**Job for simplicity:**
- Single experiments
- Low volume
- Prototyping

## Error Handling

### Job Failures

```python
try:
    result = job.result()
except Exception as e:
    print(f"Job failed: {e}")
    # Retry or handle error
```

### Session Timeout

```python
try:
    with Session(backend=backend, max_time='30m') as session:
        # ... long computation ...
except Exception as e:
    print(f"Session timed out: {e}")
```

## Best Practices

1. **Use job mode for single experiments**: Simplest, no overhead
2. **Use batch for independent parameter sweeps**: Cost-effective
3. **Use session for VQE/QAOA**: Dedicated access ensures consistency
4. **Set appropriate session TTL**: Balance between timeout and cost
5. **Close sessions explicitly**: Don't waste reserved time
6. **Monitor session status**: Avoid premature closure
7. **Submit batch jobs together**: All jobs within context for efficiency
8. **Don't mix modes for same backend**: Can cause resource conflicts
9. **Profile execution time**: Estimate session length before reserving
10. **Handle errors gracefully**: Retry logic for failed jobs

## Common Pitfalls

- **Using session for independent jobs**: Wastes premium cost
- **Forgetting to close session**: Continues charging
- **Submitting batch jobs sequentially outside context**: Loses batch benefits
- **Exceeding session ITTL**: Session terminates mid-experiment
- **Not checking job status**: Attempting to retrieve results too early
- **Mixing execution modes**: Resource contention
- **Underestimating session time**: Premature timeout
- **Overestimating batch priority**: Batch jobs wait for idle time

## Execution Mode FAQs

### Can I change mode after job submission?
No. Mode is set when primitive is instantiated.

### Can I use multiple backends in one session/batch?
No. Each session/batch is tied to a single backend.

### What happens if session times out?
Running job completes, but subsequent jobs fail. Re-open session to continue.

### How long does batch wait?
Indefinite. Jobs execute when QPU is idle, no guaranteed time.

### Can I extend a session?
No. Re-create session if more time needed.

### Do sessions guarantee no calibration drift?
Not guaranteed, but significantly reduces drift compared to job mode.

## Example: Complete VQE with Session

```python
from qiskit import QuantumCircuit
from qiskit.circuit.library import TwoLocal
from qiskit.quantum_info import SparsePauliOp
from qiskit_ibm_runtime import Session, EstimatorV2, QiskitRuntimeService
from qiskit.transpiler import generate_preset_pass_manager
from scipy.optimize import minimize
import numpy as np

# Setup
service = QiskitRuntimeService()
backend = service.backend('ibm_torino')

# Define problem
H = SparsePauliOp.from_list([('ZZ', 1.0), ('XX', -0.5)])
ansatz = TwoLocal(num_qubits=2, rotation_blocks=['ry', 'rz'],
                  entanglement_blocks='cx', reps=3)

# Transpile
pm = generate_preset_pass_manager(backend=backend, optimization_level=3)
isa_ansatz = pm.run(ansatz)
isa_H = H.apply_layout(isa_ansatz.layout)

# VQE in session
with Session(backend=backend, max_time='30m') as session:
    estimator = EstimatorV2(mode=session)

    def cost_fn(params):
        bound = isa_ansatz.assign_parameters(params)
        job = estimator.run([(bound, isa_H)])
        result = job.result()
        return result[0].data.evs

    initial = np.random.rand(isa_ansatz.num_parameters)
    result = minimize(cost_fn, initial, method='COBYLA', options={'maxiter': 100})

print(f"Ground state energy: {result.fun}")
print(f"Session used: {session.usage()}")  # Time and cost summary
```
