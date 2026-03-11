---
title: CUDA-QX Extension Libraries
impact: HIGH
tags: cudaqx, solvers, qec, vqe, adapt-vqe, qaoa, gqe, error-correction, decoder, molecular
---

# Skill: CUDA-QX Extension Libraries

CUDA-QX provides high-level quantum-classical algorithms (cudaq-solvers) and quantum error correction workflows (cudaq-qec) built on CUDA-Q.

## Quick Pattern

```python
import cudaq
import cudaq_solvers as solvers

# Molecular Hamiltonian → VQE in 5 lines
geometry = [('H', (0., 0., 0.)), ('H', (0., 0., .7474))]
molecule = solvers.create_molecule(geometry, 'sto-3g', 0, 0, casci=True)

energy, params, data = solvers.vqe(
    lambda thetas: ansatz(thetas[0]),
    molecule.hamiltonian,
    initial_parameters=[0.0],
    verbose=True
)
```

## When to Use

- Molecular ground state energy calculations (VQE, ADAPT-VQE)
- Combinatorial optimization (QAOA for Max-Cut, Max-Clique)
- Quantum error correction research and experiments
- GPU-accelerated syndrome decoding
- Real-time decoding on quantum hardware
- Molecular Hamiltonian construction from geometry

## Installation

```bash
# Individual libraries
pip install cudaq-solvers
pip install cudaq-qec

# Both together
pip install cudaq-qec cudaq-solvers

# Optional extras
pip install cudaq-qec[tensor-network-decoder]
pip install cudaq-solvers[gqe]

# libgfortran required for solvers
apt-get install gfortran

# Docker (all-in-one)
docker pull ghcr.io/nvidia/cudaqx
docker run --gpus all -it ghcr.io/nvidia/cudaqx
```

## Step-by-Step Instructions

### 1. Molecular Hamiltonian Generation

```python
import cudaq_solvers as solvers

geometry = [('H', (0., 0., 0.)), ('H', (0., 0., .7474))]
molecule = solvers.create_molecule(geometry, 'sto-3g', 0, 0, casci=True)

# Access properties
print(molecule.hamiltonian)       # SpinOperator
print(molecule.n_electrons)       # Number of electrons
print(molecule.n_orbitals)        # Number of orbitals
print(molecule.hpq)               # One-electron integrals (2D array)
print(molecule.hpqrs)             # Two-electron integrals (4D array)
print(molecule.energies)          # Classical computation results
```

Advanced molecular options:

```python
# N2 with active space
molecule = solvers.create_molecule(
    [('N', (0., 0., 0.56)), ('N', (0., 0., -0.56))],
    'sto-3g', 0, 0,
    nele_cas=2, norb_cas=3, verbose=True
)

# MP2 natural orbitals
molecule = solvers.create_molecule(geometry, 'sto-3g', 0, 0,
    nele_cas=2, norb_cas=3, MP2=True, integrals_natorb=True)

# CASSCF orbitals
molecule = solvers.create_molecule(geometry, 'sto-3g', 0, 0,
    nele_cas=2, norb_cas=3, casscf=True, integrals_casscf=True)

# Bravyi-Kitaev instead of Jordan-Wigner
molecule = solvers.create_molecule(geometry, 'sto-3g', 0, 0,
    fermion_to_spin='bravyi_kitaev')
```

### 2. VQE with UCCSD Ansatz

```python
import cudaq
import cudaq_solvers as solvers
from scipy.optimize import minimize

geometry = [('H', (0., 0., 0.)), ('H', (0., 0., .7474))]
molecule = solvers.create_molecule(geometry, 'sto-3g', 0, 0, casci=True)

numQubits = molecule.n_orbitals * 2
numElectrons = molecule.n_electrons
spin = 0

@cudaq.kernel
def ansatz(thetas: list[float]):
    q = cudaq.qvector(numQubits)
    for i in range(numElectrons):
        x(q[i])
    solvers.stateprep.uccsd(q, thetas, numElectrons, spin)

num_params = solvers.stateprep.get_num_uccsd_parameters(numElectrons, numQubits)
initial = [-.2] * num_params

energy, params, data = solvers.vqe(
    ansatz, molecule.hamiltonian, initial,
    optimizer=minimize, method='L-BFGS-B', jac='3-point',
    tol=1e-4, options={'disp': True}
)
print(f'Ground state energy: {energy}')
```

### 3. ADAPT-VQE

```python
import cudaq
import cudaq_solvers as solvers

geometry = [('H', (0., 0., 0.)), ('H', (0., 0., 0.7474))]
molecule = solvers.create_molecule(geometry, 'sto-3g', 0, 0, casci=True)

# Get operator pool
operators = solvers.get_operator_pool(
    "spin_complement_gsd", num_orbitals=molecule.n_orbitals
)

numElectrons = molecule.n_electrons

@cudaq.kernel
def initial_state(q: cudaq.qview):
    for i in range(numElectrons):
        x(q[i])

energy, parameters, selected_ops = solvers.adapt_vqe(
    initial_state, molecule.hamiltonian, operators,
    grad_norm_tolerance=1e-3, verbose=True
)
print(f"ADAPT-VQE energy: {energy}")
```

### 4. QAOA for Max-Clique

```python
import cudaq
import cudaq_solvers as solvers
import networkx as nx
import numpy as np

G = nx.Graph()
weights = [0.6686, 0.6686, 0.6686, 0.1453, 0.1453, 0.1453]
edges = [[0,1],[0,2],[0,4],[0,5],[1,2],[1,3],[1,5],[2,3],[2,4],[3,4],[3,5],[4,5]]
for i, w in enumerate(weights):
    G.add_node(i, weight=w)
G.add_edges_from(edges)

H = solvers.get_clique_hamiltonian(G, penalty=6.0)
num_layers = 3
param_count = solvers.get_num_qaoa_parameters(
    H, num_layers, full_parameterization=True, counterdiabatic=True
)
init_params = np.random.uniform(-np.pi/8, np.pi/8, param_count)

opt_value, opt_params, opt_config = solvers.qaoa(
    H, num_layers, init_params,
    full_parameterization=True, counterdiabatic=True
)
print(f'Optimal energy: {opt_value}')
print(f'Best configuration: {opt_config.most_probable()}')
```

### 5. QEC — Code-Capacity Experiment

```python
import cudaq
import cudaq_qec as qec

cudaq.set_target("stim")

code = qec.get_code("steane")
decoder = qec.get_decoder("single_error_lut", code.get_parity())

noise = cudaq.NoiseModel()
noise.add_all_qubit_channel("x", cudaq.Depolarization2(0.01), 1)

syndromes, measurements = qec.sample_memory_circuit(
    code, op=qec.operation.prep0, numShots=1000, numRounds=10, noise=noise
)

logical_errors = 0
for shot in range(1000):
    result = decoder.decode(syndromes[shot].tolist())
    if result.converged:
        errors = [i for i, p in enumerate(result.result) if p > 0.5]
```

### 6. QEC — Circuit-Level with DEM

```python
import cudaq
import cudaq_qec as qec
import numpy as np

cudaq.set_target("stim")

code = qec.get_code("repetition", distance=3)
noise = cudaq.NoiseModel()
noise.add_all_qubit_channel("x", cudaq.Depolarization2(0.01), 1)

# Generate Detector Error Model
dem = qec.z_dem_from_memory_circuit(code, qec.operation.prep0, 6, noise)

decoder = qec.get_decoder("single_error_lut", dem.detector_error_matrix)
syndromes, data = qec.sample_memory_circuit(code, qec.operation.prep0, 10000, 6, noise)

syndromes = syndromes.reshape((10000, -1))
logical_obs = code.get_observables_z()
obs_flips = dem.observables_flips_matrix

for i in range(10000):
    result = decoder.decode(syndromes[i, :])
    prediction = np.array(result.result, dtype=np.uint8)
    predicted_flip = obs_flips @ prediction % 2
    measured = logical_obs @ data[i, :] % 2
    corrected = predicted_flip ^ measured
```

### 7. QEC — GPU-Accelerated QLDPC Decoder

```python
import cudaq_qec as qec
import numpy as np

H = np.array([[1,0,0,1,0,1,1],
              [0,1,0,1,1,0,1],
              [0,0,1,0,1,1,1]], dtype=np.uint8)

# Sum-Product BP (default)
decoder = qec.get_decoder("nv-qldpc-decoder", H,
    bp_method=0, max_iterations=30)

# Min-Sum BP
decoder = qec.get_decoder("nv-qldpc-decoder", H,
    bp_method=1, max_iterations=30, scale_factor=1.0)

# Memory BP
decoder = qec.get_decoder("nv-qldpc-decoder", H,
    bp_method=2, max_iterations=30, gamma0=0.5)

# Disordered Memory BP
decoder = qec.get_decoder("nv-qldpc-decoder", H,
    bp_method=3, max_iterations=30, gamma_dist=[0.1, 0.5], bp_seed=42)

# With OSD post-processing
decoder = qec.get_decoder("nv-qldpc-decoder", H,
    use_osd=True, osd_order=10, max_iterations=50)

syndrome = np.array([1.0, 0.0, 1.0], dtype=np.float32)
result = decoder.decode(syndrome)
print(f"Converged: {result.converged}, Correction: {result.result}")
```

### 8. QEC — Real-Time Decoding

```python
import cudaq
import cudaq_qec as qec

cudaq.set_target("stim")

code = qec.get_code("steane")
noise = cudaq.NoiseModel()
noise.add_all_qubit_channel("x", cudaq.Depolarization2(0.01), 1)
dem = qec.z_dem_from_memory_circuit(code, qec.operation.prep0, 3, noise)

# Configure decoder
config = qec.decoder_config()
config.id = 0
config.type = "nv-qldpc-decoder"
config.block_size = dem.detector_error_matrix.shape[1]
config.H_sparse = qec.pcm_to_sparse_vec(dem.detector_error_matrix)
config.O_sparse = qec.pcm_to_sparse_vec(dem.observables_flips_matrix)

# In-kernel decoding functions
@cudaq.kernel
def decode_and_correct(logical: qec.patch, decoder_id: int):
    syndromes = measure_stabilizers(logical)
    qec.enqueue_syndromes(decoder_id, syndromes, 0)
    corrections = qec.get_corrections(decoder_id, 1, False)
    if corrections[0]:
        x(logical.data)
```

## API Quick Reference

### Solvers Algorithms

| Algorithm | Function | Use Case |
|-----------|----------|----------|
| VQE | `solvers.vqe(kernel, H, params)` | Ground state energy |
| ADAPT-VQE | `solvers.adapt_vqe(init, H, pool)` | Adaptive ansatz construction |
| QAOA | `solvers.qaoa(H, layers, params)` | Combinatorial optimization |
| GQE | `solvers.gqe(cost, pool, ...)` | AI-guided circuit generation |

### Operator Pools

| Pool | ID | Description |
|------|----|-------------|
| Spin-complement GSD | `"spin_complement_gsd"` | Generalized excitations with spin symmetry |
| UCCSD | `"uccsd"` | Standard occupied-to-virtual excitations |
| UCCGSD | `"uccgsd"` | Unrestricted all singles and doubles |
| QAOA | `"qaoa"` | Single-qubit + two-qubit interaction terms |

### State Preparation

| Function | Purpose |
|----------|---------|
| `solvers.stateprep.uccsd(q, params, n_electrons, spin)` | UCCSD ansatz |
| `solvers.stateprep.uccgsd(q, params, words, coeffs)` | UCCGSD ansatz |
| `solvers.stateprep.get_num_uccsd_parameters(n_e, n_q)` | Parameter count |
| `solvers.stateprep.single_excitation(q, param, i, j)` | Single excitation |
| `solvers.stateprep.double_excitation(q, param, i, j, k, l)` | Double excitation |

### Hamiltonian Utilities

| Function | Purpose |
|----------|---------|
| `solvers.create_molecule(geometry, basis, spin, charge)` | Molecular Hamiltonian |
| `solvers.jordan_wigner(hpq, hpqrs)` | Fermion-to-spin mapping |
| `solvers.bravyi_kitaev(hpq, hpqrs)` | Fermion-to-spin mapping |
| `solvers.get_maxcut_hamiltonian(graph)` | MaxCut problem |
| `solvers.get_clique_hamiltonian(graph, penalty)` | Max-Clique problem |

### QEC Codes

| Code | ID | Parameters |
|------|----|------------|
| Steane | `"steane"` | `[[7,1,3]]` code |
| Repetition | `"repetition"` | `distance=N` |
| Surface Code | `"surface_code"` | `distance=N` |

### QEC Decoders

| Decoder | ID | GPU | Real-Time |
|---------|----|-----|-----------|
| NVIDIA QLDPC | `"nv-qldpc-decoder"` | Yes | Yes |
| Tensor Network | `"tensor_network_decoder"` | Yes | No |
| TensorRT AI | `"trt_decoder"` | Yes | No |
| Single-Error LUT | `"single_error_lut"` | No | Yes |
| Multi-Error LUT | `"multi_error_lut"` | No | Yes |
| Sliding Window | `"sliding_window"` | No | No |

### QLDPC BP Methods

| Method | `bp_method` | Key Params |
|--------|:-----------:|------------|
| Sum-Product | 0 | `max_iterations` |
| Min-Sum | 1 | `scale_factor` |
| Memory BP | 2 | `gamma0` |
| Disordered Memory | 3 | `gamma_dist`, `bp_seed` |

## Common Pitfalls

- **Missing libgfortran**: `apt-get install gfortran` required for solvers.
- **Wrong basis set**: Use `'sto-3g'` for quick tests; larger bases need more qubits.
- **ADAPT-VQE pool size**: `spin_complement_gsd` pool grows quickly with orbitals. Start small.
- **QEC target**: Must `cudaq.set_target("stim")` for QEC memory circuit sampling.
- **Decoder mismatch**: Parity check matrix dimensions must match code's stabilizer count.
- **GQE requires PyTorch**: Install with `pip install cudaq-solvers[gqe]`.

## Related Skills

- [variational-algorithms.md](./variational-algorithms.md) - Core VQE/QAOA with cudaq.vqe
- [noise-modeling.md](./noise-modeling.md) - Noise channels for QEC experiments
- [kernels-and-gates.md](./kernels-and-gates.md) - Kernel construction
- [hardware-backends.md](./hardware-backends.md) - Real-time decoding on Quantinuum
