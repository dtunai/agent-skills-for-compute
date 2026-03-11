---
title: Math & Utilities
impact: MEDIUM
tags: spherical-harmonics, gto, batch-processing, types, warp
---

# Skill: Math & Utilities

Spherical harmonics, Gaussian Type Orbitals, batch processing patterns, and type conversion utilities.

## Quick Pattern

```python
from nvalchemiops.math.gto import eval_gto_density_pytorch

# Evaluate GTO density from atomic positions
density = eval_gto_density_pytorch(positions, sigma=1.0, L_max=2)
# Returns [N, num_components] for L=0,1,2
```

## When to Use

- Computing real spherical harmonics for angular momentum / multipole expansions
- Evaluating Gaussian Type Orbital basis functions
- Batching heterogeneous molecular systems for single-kernel GPU processing
- Converting between PyTorch and Warp dtypes

## Step-by-Step Instructions

### 1. Gaussian Type Orbitals (GTO)

```python
from nvalchemiops.math.gto import (
    eval_gto_density_pytorch,   # GTO density from positions
    eval_gto_fourier_pytorch,   # GTO Fourier transform from k-vectors
)

# Real-space GTO density
# L_max: 0 (1 component), 1 (4 components), 2 (9 components)
density = eval_gto_density_pytorch(
    positions,     # [N, 3]
    sigma=1.0,     # Gaussian width
    L_max=2,       # max angular momentum
)
# Returns [N, 9] for L_max=2

# Fourier-space GTO
real_part, imag_part = eval_gto_fourier_pytorch(
    k_vectors,     # [K, 3]
    sigma=1.0,
    L_max=2,
)
# Each: [K, 9] for L_max=2
```

### 2. Spherical Harmonics

```python
from nvalchemiops.math.spherical_harmonics import real_spherical_harmonics

# Compute real spherical harmonics Y_l^m(theta, phi)
# Used in multipole expansions and angular basis functions
```

### 3. Batch Processing Pattern

All ALCHEMI components support heterogeneous batching — processing multiple molecular systems of different sizes in a single kernel launch:

```python
import torch

# Concatenate positions from multiple systems
positions = torch.cat([pos_sys0, pos_sys1, pos_sys2])
charges = torch.cat([q_sys0, q_sys1, q_sys2])

# Create batch_idx: maps each atom to its system index
batch_idx = torch.cat([
    torch.zeros(len(pos_sys0), dtype=torch.int32),
    torch.ones(len(pos_sys1), dtype=torch.int32),
    torch.full((len(pos_sys2),), 2, dtype=torch.int32),
]).to("cuda")

# Stack per-system tensors
cells = torch.stack([cell0, cell1, cell2])     # [B, 3, 3]
pbc = torch.tensor([[True, True, True]] * 3, device="cuda")  # [B, 3]

# All functions accept batch_idx for batched processing
from nvalchemiops.neighborlist import neighbor_list
result = neighbor_list(positions, cutoff=5.0, cell=cells, pbc=pbc, batch_idx=batch_idx)

from nvalchemiops.interactions.electrostatics import ewald_summation
energies, forces = ewald_summation(
    positions, charges, cells,
    neighbor_list=result.neighbor_list, neighbor_shifts=result.neighbor_list_shifts,
    batch_idx=batch_idx, accuracy=5e-4,
)
# energies: [3] — one per system
```

Recover per-system quantities from per-atom outputs:

```python
# Per-system energy from per-atom contributions
energy_per_system = torch.zeros(num_systems, device="cuda")
energy_per_system.scatter_add_(0, batch_idx.long(), per_atom_energies)
```

### 4. Type Conversions

```python
from nvalchemiops.types import (
    get_wp_dtype,       # PyTorch → Warp scalar dtype
    get_wp_vec_dtype,   # PyTorch → Warp vector dtype
    get_wp_mat_dtype,   # PyTorch → Warp matrix dtype
)

wp_float = get_wp_dtype(torch.float32)      # warp.float32
wp_vec3 = get_wp_vec_dtype(torch.float64)   # warp.vec3d
wp_mat33 = get_wp_mat_dtype(torch.float32)  # warp.mat33f
```

### 5. Common Tensor Specifications

| Tensor | Shape | Dtype | Description |
|--------|-------|-------|-------------|
| `positions` | `(N, 3)` | float32/64 | Atomic coordinates |
| `charges` | `(N,)` | float64 | Partial charges |
| `numbers` | `(N,)` | int32 | Atomic numbers |
| `cell` | `(1,3,3)` or `(B,3,3)` | float32/64 | Lattice vectors (rows) |
| `pbc` | `(1,3)` or `(B,3)` | bool | Periodic boundary per axis |
| `batch_idx` | `(N,)` | int32 | Atom-to-system mapping |
| `batch_ptr` | `(B+1,)` | int32 | CSR-style batch pointer |
| `multipoles` | `(N, 9)` | float64 | Monopole + dipole + quadrupole |

### 6. Installation and Docker

```bash
# PyPI (simplest)
pip install nvalchemi-toolkit-ops

# From source
pip install git+https://github.com/NVIDIA/nvalchemi-toolkit-ops.git

# With uv
uv venv --seed --python 3.12
uv pip install nvalchemi-toolkit-ops

# Docker
FROM nvidia/cuda:13.0.0-runtime-ubuntu24.04
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
RUN uv venv --seed --python 3.12 /opt/venv
ENV VIRTUAL_ENV=/opt/venv PATH="/opt/venv/bin:$PATH"
RUN uv pip install nvalchemi-toolkit-ops
# Run with: docker run --gpus all ...
```

Verify: `python -c "import nvalchemiops; print(nvalchemiops.__version__)"`

### 7. Performance Notes

- All kernels use NVIDIA Warp for GPU acceleration
- `torch.compile` supported for neighbor lists and dispersion (pre-allocate outputs)
- Cell list algorithms: O(N) scaling, consistent throughput as system size grows
- PME: O(N log N), preferred for >5000 atoms
- Naive algorithms: O(N^2), only for small systems
- Benchmarked on H100 80GB HBM3 with FCC lattice, float32

## Common Pitfalls

- **batch_idx dtype**: Must be `int32`, not `int64`.
- **PME mesh dimensions**: Must be identical across all systems in a batch.
- **Alpha per-system**: For batched Ewald/PME, alpha can vary per system but mesh dims cannot.
- **Cell volume**: Must be positive (`det(cell) > 0`). Check lattice vector ordering.
- **CPU fallback**: All ops run on CPU via Warp (x86, ARM, Apple Silicon) but much slower than GPU.

## Related Skills

- [neighbor-lists.md](./neighbor-lists.md) - Neighbor list construction
- [electrostatics.md](./electrostatics.md) - Electrostatic interactions
- [dispersion-corrections.md](./dispersion-corrections.md) - DFT-D3 corrections
