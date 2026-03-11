# Qiskit Dynamic Circuits and Classical Control Flow Reference

Sources:
- [Classical Feedforward and Control Flow](https://quantum.cloud.ibm.com/docs/en/guides/classical-feedforward-and-control-flow)
- [Dynamic Circuits Considerations](https://docs.quantum.ibm.com/guides/dynamic-circuits-considerations)
- [Utility-Scale Dynamic Circuits Blog](https://www.ibm.com/quantum/blog/utility-scale-dynamic-circuits)

## Overview

Dynamic circuits enable **mid-circuit measurements** followed by **classical logic operations** that determine subsequent quantum operations. This capability, termed "classical feedforward," allows quantum programs to adapt execution based on measurement outcomes.

## Core Mechanism: if_test

### Basic Syntax

```python
from qiskit import QuantumCircuit

qc = QuantumCircuit(2, 2)

# Measure qubit and conditionally operate
qc.h(0)
qc.measure(0, 0)

# If classical bit 0 == 1, apply X gate
with qc.if_test((0, 1)):
    qc.x(1)
```

### If-Else Blocks

```python
qc = QuantumCircuit(2, 2)
qc.h(0)
qc.measure(0, 0)

# Conditional branches
with qc.if_test((0, 1)) as else_:
    qc.h(1)  # Execute if bit == 1
with else_:
    qc.x(1)  # Execute if bit == 0
```

### Multi-Bit Conditions

```python
from qiskit import QuantumCircuit, ClassicalRegister

qc = QuantumCircuit(3, 3)
cr = ClassicalRegister(3, 'meas')

qc.h(range(3))
qc.measure(range(3), cr)

# Condition on full register value
with qc.if_test((cr, 0b101)):  # If register == 5
    qc.x(0)
```

### Testing Individual Bits from Large Registers

```python
# Valid: test single bit from large register (bypasses 32-bit limit)
cr = ClassicalRegister(64, 'big_register')
qc = QuantumCircuit(3, cr)

qc.measure(0, cr[0])

# Test single bit (works regardless of register size)
with qc.if_test((cr[5], 1)):
    qc.x(1)
```

## Classical Expressions

The `qiskit.circuit.classical.expr` module enables runtime operations on classical values.

### Parity Computation (XOR)

```python
from qiskit.circuit.classical import expr

qc = QuantumCircuit(4, 4)
qc.h(range(3))
qc.measure(range(3), range(3))

# Compute parity of first 3 bits
parity = expr.lift(0, 0)  # Initialize with classical bit 0
parity = expr.bit_xor(parity, expr.lift(0, 1))
parity = expr.bit_xor(parity, expr.lift(0, 2))

# Use parity in condition
with qc.if_test(parity):
    qc.x(3)
```

### Supported Operations

| Operation | Function |
|-----------|----------|
| XOR | `expr.bit_xor(a, b)` |
| AND | `expr.bit_and(a, b)` |
| OR | `expr.bit_or(a, b)` |
| NOT | `expr.bit_not(a)` |
| Equality | `expr.equal(a, b)` |
| Less than | `expr.less(a, b)` |

## Applications of Dynamic Circuits

### 1. Qubit Reset (Conditional Feedback)

```python
def reset_qubit(qc, qubit, cbit):
    """Deterministic reset to |0⟩ via measurement and feedback."""
    qc.measure(qubit, cbit)
    with qc.if_test((cbit, 1)):
        qc.x(qubit)  # Flip to |0⟩ if measured |1⟩

qc = QuantumCircuit(1, 1)
qc.h(0)  # Arbitrary state
reset_qubit(qc, 0, 0)
```

### 2. GHZ State Preparation (Shallow Circuit)

```python
def create_ghz_dynamic(n):
    """GHZ state with O(log n) depth using mid-circuit measurements."""
    qc = QuantumCircuit(n, n)

    # Create Bell pair
    qc.h(0)
    qc.cx(0, 1)

    # Expand via measurement and feedforward
    for i in range(2, n):
        qc.h(i)
        qc.measure(i, i)
        with qc.if_test((i, 1)):
            qc.x(0)  # Correct phase
        qc.cx(0, i)

    return qc
```

### 3. Repeat-Until-Success (RUS)

```python
def repeat_until_success(qc, target_qubit, ancilla, cbit):
    """Probabilistic gate with post-selection."""
    qc.h(ancilla)
    qc.cx(ancilla, target_qubit)
    qc.measure(ancilla, cbit)

    # Retry if measurement failed (cbit == 1)
    with qc.if_test((cbit, 1)):
        qc.reset(ancilla)
        repeat_until_success(qc, target_qubit, ancilla, cbit)
```

### 4. Quantum Teleportation

```python
def teleportation():
    qc = QuantumCircuit(3, 2)

    # Bell pair (qubits 1, 2)
    qc.h(1)
    qc.cx(1, 2)

    # Prepare state to teleport (qubit 0)
    qc.h(0)

    # Bell measurement
    qc.cx(0, 1)
    qc.h(0)
    qc.measure([0, 1], [0, 1])

    # Corrections based on measurement
    with qc.if_test((1, 1)):
        qc.x(2)
    with qc.if_test((0, 1)):
        qc.z(2)

    return qc
```

## Hardware Limitations

### Broadcast Limits

**Definition:** Broadcasting occurs when measurement data transfers from measurement logic to control logic for use in `if_test`.

**Current Limits:**
- ~300 broadcasts at 60 bits per broadcast
- OR ~2400 broadcasts using single bits

**Optimization:**
- Reusing the same classical register in multiple `if_test` counts as ONE broadcast
- Testing individual bits is more efficient than full registers

```python
# Inefficient: 3 broadcasts
cr1 = ClassicalRegister(3)
cr2 = ClassicalRegister(3)
cr3 = ClassicalRegister(3)

with qc.if_test((cr1, 0)):
    ...
with qc.if_test((cr2, 0)):
    ...
with qc.if_test((cr3, 0)):
    ...

# Efficient: 1 broadcast (reuse register)
cr = ClassicalRegister(3)

with qc.if_test((cr, 0)):
    ...
with qc.if_test((cr, 1)):
    ...
with qc.if_test((cr, 2)):
    ...
```

### Operand Size Constraint

Classical operands must not exceed **32 bits**.

```python
# Invalid: register > 32 bits
cr = ClassicalRegister(64)
with qc.if_test((cr, 15)):  # ERROR
    qc.x(0)

# Valid workaround: test individual bits
with qc.if_test((cr[5], 1)):  # OK
    qc.x(0)
```

### Unsupported Constructs

**Not Supported on Hardware:**
- Nested conditionals (if within if)
- Reset or measurement inside `if_test` blocks
- Arithmetic operations (+, -, *, /)
- Loop constructs (`for`, `while`)
- `switch` statements

```python
# Invalid: nested if
with qc.if_test((cr, 0)):
    with qc.if_test((cr, 1)):  # NOT SUPPORTED
        qc.x(0)

# Invalid: measurement inside if
with qc.if_test((cr, 0)):
    qc.measure(0, 0)  # NOT SUPPORTED
```

### Latency

**Mid-circuit measurement latency:** 400-700 ns

This adds to circuit duration. For time-critical applications, minimize mid-circuit measurements.

## MidCircuitMeasure Instruction

Use optimized mid-circuit measurement when backend supports it.

```python
from qiskit.circuit import MidCircuitMeasure

# Faster than standard measure instruction
qc.append(MidCircuitMeasure(), [qubit], [cbit])
```

## Primitives and Dynamic Circuits

### Sampler

**Supported:** Dynamic circuits work with SamplerV2.

```python
from qiskit_ibm_runtime import SamplerV2

sampler = SamplerV2(mode=backend)
job = sampler.run([dynamic_circuit])
result = job.result()
```

### Estimator

**Not Supported:** EstimatorV2 does NOT support dynamic circuits.

**Workaround:** Use Sampler with manual measurement basis changes.

```python
# Instead of Estimator with dynamic circuit
# Use Sampler and compute expectation values manually
from qiskit.quantum_info import SparsePauliOp

observable = SparsePauliOp("ZZ")

# Rotate to measurement basis
qc_meas = dynamic_circuit.copy()
# Apply basis rotation for observable
# ...
sampler = SamplerV2(mode=backend)
job = sampler.run([qc_meas])
# Compute expectation from counts
```

## Debugging Dynamic Circuits

### Visualize Timing

```python
# Draw with scheduling info
from qiskit.visualization import timeline_drawer

timeline_drawer(qc, show_idle=True)
```

### Check Broadcast Count

Manually count `if_test` statements with unique classical registers.

### Verify Operand Sizes

```python
for reg in qc.cregs:
    assert reg.size <= 32, f"Register {reg.name} exceeds 32 bits"
```

## Best Practices

1. **Minimize broadcasts**: Reuse classical registers across `if_test` blocks
2. **Use single-bit conditions** when possible (more efficient)
3. **Avoid nested conditionals**: Not supported on hardware
4. **Profile latency**: Mid-circuit measurements add 400-700 ns
5. **Use MidCircuitMeasure**: Optimized instruction for supported backends
6. **Test with Sampler**: Estimator doesn't support dynamic circuits
7. **Check backend compatibility**: Verify `backend.target.supports('if_test')`
8. **Keep operands ≤ 32 bits**: Test individual bits from large registers

## Common Pitfalls

- **Using Estimator**: Dynamic circuits require Sampler
- **Exceeding broadcast limit**: Unique registers count as separate broadcasts
- **Operands > 32 bits**: Test individual bits instead
- **Measuring inside if_test**: Not allowed on hardware
- **Nested conditionals**: Hardware doesn't support nested `if_test`
- **Assuming zero latency**: Mid-circuit measurements have 400-700 ns overhead
- **Forgetting else block**: Use `as else_:` syntax for if-else

## Advanced: Executor Primitive (Future)

The upcoming **Executor** primitive will natively support dynamic circuits for both sampling and expectation value computation.

```python
# Future API (not yet available)
from qiskit_ibm_runtime import Executor

executor = Executor(mode=backend)
job = executor.run(dynamic_circuit)
result = job.result()
```

## Examples Library

### Syndrome Measurement (QEC)

```python
def syndrome_measurement(qc, data_qubits, ancilla, syndrome_bit):
    """Measure stabilizer syndrome without destroying data qubits."""
    # Entangle ancilla with data qubits
    qc.h(ancilla)
    for data in data_qubits:
        qc.cx(ancilla, data)

    # Measure syndrome
    qc.measure(ancilla, syndrome_bit)

    # Reset ancilla for reuse
    with qc.if_test((syndrome_bit, 1)):
        qc.x(ancilla)
```

### Long-Range Entanglement

```python
def long_range_cx(qc, control, target, ancilla, cbit):
    """Implement CNOT across distant qubits using mid-circuit measurement."""
    qc.cx(control, ancilla)
    qc.measure(ancilla, cbit)

    with qc.if_test((cbit, 1)):
        qc.x(target)

    qc.reset(ancilla)
```

## Status and Roadmap

- **Current (2026)**: `if_test` supported on IBM Quantum utility-scale devices
- **Future**: Full control flow (while, for, switch), nested conditionals, arithmetic
- **Research**: Classical compute co-processors, real-time feedback optimization
