# Qiskit Error Mitigation and Suppression Reference

Sources:
- [IBM Quantum Documentation - Configure Error Mitigation](https://docs.quantum.ibm.com/guides/configure-error-mitigation)
- [Error Mitigation and Suppression Techniques](https://docs.quantum.ibm.com/guides/error-mitigation-and-suppression-techniques)

## Overview

Quantum computers suffer from noise-induced errors. Qiskit Runtime provides error **suppression** (reducing error occurrence) and error **mitigation** (correcting errors post-measurement) techniques.

## Error Suppression vs. Mitigation

**Suppression (Pre-execution):**
- Dynamical decoupling
- Gate/measurement twirling
- Noise-aware compilation

**Mitigation (Post-execution):**
- Readout error mitigation
- Zero Noise Extrapolation (ZNE)
- Probabilistic Error Cancellation (PEC)

## Resilience Levels

### Quick Configuration

```python
from qiskit_ibm_runtime import EstimatorV2 as Estimator
from qiskit_ibm_runtime import Options

options = Options()
options.resilience_level = 2  # 0, 1, or 2

estimator = Estimator(mode=backend, options=options)
```

### Level Breakdown

| Level | Techniques | Runtime | Accuracy | Use Case |
|-------|------------|---------|----------|----------|
| **0** | None | Fastest | Raw | Debugging, noiseless simulation |
| **1** | Twirling + readout mitigation | Fast | Good | General-purpose NISQ |
| **2** | Level 1 + ZNE | Slow | Best | High-fidelity requirements |

## Dynamical Decoupling (DD)

### Purpose

Insert gate sequences on idle qubits to suppress dephasing and relaxation errors during wait times.

### Mechanism

Idle qubits accumulate phase errors. DD applies periodic π-pulses (X gates) that average out environmental noise.

**Effective for:**
- Circuits with gaps (barriers, long multi-qubit operations)
- High coherence time / gate time ratio

**Ineffective for:**
- Dense circuits (no idle time)
- Fast gates relative to decoherence

### Configuration

```python
from qiskit_ibm_runtime import Options

options = Options()
options.dynamical_decoupling.enable = True
options.dynamical_decoupling.sequence_type = 'XY4'  # or 'XX'

estimator = Estimator(mode=backend, options=options)
```

### DD Sequences

| Sequence | Gates | Robustness | Use Case |
|----------|-------|------------|----------|
| **XX** | X-X | Basic | Simple dephasing |
| **XY4** | X-Y-X-Y | Advanced | Dephasing + relaxation |

### Manual DD Insertion

```python
from qiskit.transpiler.passes import DynamicalDecoupling
from qiskit.circuit.library import XGate, YGate
from qiskit.transpiler import PassManager

# XY4 sequence
dd_sequence = [XGate(), YGate(), XGate(), YGate()]

dd_pass = DynamicalDecoupling(
    durations=backend.instruction_durations,
    dd_sequence=dd_sequence,
    qubits=[0, 1, 2]  # apply to specific qubits
)

pm = PassManager([dd_pass])
dd_circuit = pm.run(isa_circuit)
```

## Gate and Measurement Twirling

### Purpose

Randomize coherent errors into stochastic noise, making them easier to mitigate.

### Gate Twirling

Insert random Pauli gates before/after operations that cancel out but randomize error channels.

**Example:**
```
Original: CNOT
Twirled:  X-CNOT-X (with 50% probability)
```

**Benefit:** Converts coherent errors → depolarizing noise (easier to handle).

### Measurement Twirling

Randomly flip qubits before measurement and adjust results post-processing.

### Configuration

```python
options = Options()
options.twirling.enable_gates = True
options.twirling.enable_measure = True
options.twirling.num_randomizations = 300  # shots per randomization

estimator = Estimator(mode=backend, options=options)
```

## Readout Error Mitigation

### Purpose

Correct measurement errors (bitflip probabilities in readout).

### Mechanism

1. **Characterization**: Benchmark readout by preparing |0⟩ and |1⟩, measuring error rates
2. **Inversion**: Construct assignment matrix and invert to correct counts

**Assignment Matrix Example (1 qubit):**
```
       Measured
Prepared | 0    1
-----------------
    0    | 0.98 0.02
    1    | 0.03 0.97
```

### Automatic Mitigation

Enabled by default in resilience levels 1-2.

```python
# Automatically applied
options.resilience_level = 1  # includes readout mitigation
```

### Manual Mitigation

```python
from qiskit_ibm_runtime import SamplerV2 as Sampler

options = Options()
options.resilience.measure_mitigation = True

sampler = Sampler(mode=backend, options=options)
```

## Zero Noise Extrapolation (ZNE)

### Purpose

Estimate ideal (zero-noise) expectation value by artificially amplifying noise and extrapolating to zero-noise limit.

### Mechanism

1. **Noise Amplification**: Run circuit with increased noise (e.g., 1×, 3×, 5× noise)
   - Methods: Gate stretching, pulse stretching, unitary folding
2. **Extrapolation**: Fit noise curve and extrapolate to zero noise

**Noise Amplification Example (Unitary Folding):**
```
Original: U
1× noise: U
3× noise: U U† U U† U
5× noise: U U† U U† U U† U U† U
```

### Configuration

```python
options = Options()
options.resilience_level = 2  # enables ZNE

# Fine-tune ZNE
options.resilience.zne_mitigation = True
options.resilience.zne.noise_factors = [1, 3, 5]  # amplification levels
options.resilience.zne.extrapolator = 'exponential'  # or 'linear', 'polynomial'

estimator = Estimator(mode=backend, options=options)
```

### ZNE Extrapolators

| Type | Fit | Use Case |
|------|-----|----------|
| **Linear** | f(λ) = a + bλ | Short noise range |
| **Exponential** | f(λ) = a + b exp(cλ) | General-purpose |
| **Polynomial** | f(λ) = a + bλ + cλ² | Complex noise models |

### ZNE Effectiveness

**Best for:**
- Estimator primitive (expectation values)
- Circuits with moderate noise
- Smooth error scaling

**Less effective for:**
- Sampler primitive (discrete outcomes)
- Highly noisy circuits (extrapolation fails)
- Non-linear noise models

## Probabilistic Error Cancellation (PEC)

### Purpose

Exactly cancel noise by sampling from quasi-probability distribution over noisy operations.

### Mechanism

Represent ideal gate as linear combination of noisy gates:
```
U_ideal = Σ α_i U_noisy_i
```

Sample operations with weights α_i (some negative → quasi-probability).

### Properties

- **Unbiased**: Converges to exact expectation value
- **High overhead**: Requires many shots (scales exponentially with circuit depth)
- **Requires noise model**: Need accurate characterization

### Status in Qiskit Runtime

PEC is experimental and not widely exposed in Runtime options. Primarily available through research tools.

## Noise-Aware Compilation

### Purpose

Optimize circuit considering backend error rates.

### Techniques

**Layout Selection:**
Choose qubits with lowest error rates.

```python
from qiskit.transpiler.passes import NoiseAdaptiveLayout

# Automatically selects best qubits
pm = generate_preset_pass_manager(
    optimization_level=3,
    backend=backend  # uses error rates from backend.properties()
)
```

**Routing Optimization:**
Minimize SWAPs on high-error qubit pairs.

**Gate Decomposition:**
Choose decompositions minimizing total error.

## Combining Techniques

### Recommended Stack

```python
from qiskit_ibm_runtime import EstimatorV2, Options

options = Options()

# Suppression
options.dynamical_decoupling.enable = True
options.dynamical_decoupling.sequence_type = 'XY4'

options.twirling.enable_gates = True
options.twirling.enable_measure = True

# Mitigation
options.resilience_level = 2  # ZNE + readout mitigation

# Optimization
options.optimization_level = 3

estimator = EstimatorV2(mode=backend, options=options)
```

### Trade-offs

| Technique | Overhead | Accuracy Gain | Best For |
|-----------|----------|---------------|----------|
| **DD** | None | Low-Medium | Idle-heavy circuits |
| **Twirling** | 2-5× shots | Medium | General NISQ |
| **Readout Mitigation** | Minimal | Medium | All circuits |
| **ZNE** | 3-10× shots | High | Expectation values |
| **PEC** | Exponential | Exact (unbiased) | Small, critical circuits |

## Advanced Options

### Layer Noise Learning

Automatically learn and mitigate layer-specific noise.

```python
options.resilience.layer_noise_learning.enable = True
options.resilience.layer_noise_learning.shots_per_randomization = 128
```

### Measure Noise Learning

Characterize and mitigate measurement crosstalk.

```python
options.resilience.measure_noise_learning.enable = True
```

## Monitoring Mitigation

### Job Metadata

```python
job = estimator.run(pubs)
result = job.result()

# Check mitigation applied
metadata = result.metadata
print(f"Mitigation used: {metadata['resilience']}")
print(f"ZNE noise factors: {metadata['zne_noise_factors']}")
```

## Best Practices

1. **Start with resilience_level=1** for general use
2. **Enable DD for circuits with barriers** or long wait times
3. **Use ZNE (level 2) for high-accuracy** expectation values
4. **Twirling is cheap** — enable by default
5. **Readout mitigation is essential** for all hardware jobs
6. **Noise-aware compilation** (level 3) is free performance
7. **Monitor shot overhead** — mitigation increases runtime
8. **Benchmark against simulation** to verify mitigation effectiveness
9. **Consider circuit depth** — mitigation degrades for very deep circuits
10. **Don't over-mitigate** — more is not always better (shot noise dominates)

## Limitations

- **No silver bullet**: Mitigation reduces but doesn't eliminate errors
- **Overhead scales**: Deep circuits require exponentially more shots
- **Assumes noise models**: Techniques fail if noise behaves unexpectedly
- **Hardware-dependent**: Effectiveness varies by backend
- **Shot noise floor**: Mitigation can't beat fundamental shot noise limit

## Error Correction (Future)

Current Qiskit focuses on **error mitigation** (classical post-processing). Future releases will integrate **quantum error correction** (QEC) with:
- Logical qubits
- Syndrome measurement
- Surface codes, stabilizer codes

Experimental QEC available via `cudaq-qec` (CUDA-QX ecosystem).
