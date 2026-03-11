---
title: Electrostatics
impact: CRITICAL
tags: ewald, pme, coulomb, multipole, long-range, periodic, forces, autograd
---

# Skill: Electrostatics

GPU-accelerated electrostatic interactions: Ewald summation, PME, direct Coulomb, and multipole expansions.

## Quick Pattern

```python
from nvalchemiops.interactions.electrostatics import ewald_summation

energies, forces = ewald_summation(
    positions=positions,   # [N, 3], float64
    charges=charges,       # [N], float64
    cell=cell,             # [1, 3, 3], float64
    neighbor_list=nl_coo,  # [2, num_pairs]
    neighbor_shifts=shifts,
    accuracy=5e-4,         # auto-estimates alpha, cutoffs
)
```

## When to Use

- Computing long-range electrostatic energy and forces in periodic systems
- Ewald summation for small systems (<5000 atoms)
- PME for large periodic systems (O(N log N))
- Direct Coulomb for non-periodic or real-space-only calculations
- Multipole electrostatics (dipoles, quadrupoles) for ML force fields
- Autograd for charge gradients (ML charge prediction) or stress tensors

## Step-by-Step Instructions

### 1. Ewald Summation

Complete real + reciprocal + self-energy + background corrections:

```python
from nvalchemiops.interactions.electrostatics import ewald_summation

energies, forces = ewald_summation(
    positions=positions,       # [N, 3], float64
    charges=charges,           # [N], float64
    cell=cell,                 # [1, 3, 3] or [B, 3, 3], float64
    neighbor_list=nl_coo,      # [2, num_pairs] or neighbor_matrix
    neighbor_shifts=shifts,    # matching shift format
    accuracy=5e-4,             # auto-estimates alpha, cutoffs
    # OR explicit parameters:
    # alpha=0.3,
    # k_cutoff=8.0,
    batch_idx=batch_idx,       # for batched systems
    compute_forces=True,
)
```

Component functions for fine-grained control:

```python
from nvalchemiops.interactions.electrostatics import (
    ewald_real_space,         # erfc-damped Coulomb
    ewald_reciprocal_space,   # structure factor + self/background corrections
)

# Real space: returns energies, forces, charge gradients
e_real, f_real, dE_dq_real = ewald_real_space(
    positions, charges, cell, nl_coo, shifts, alpha,
)

# Reciprocal space
e_recip, f_recip = ewald_reciprocal_space(
    positions, charges, cell, k_vectors, alpha,
)
```

### 2. Particle Mesh Ewald (PME)

O(N log N) for large periodic systems:

```python
from nvalchemiops.interactions.electrostatics import particle_mesh_ewald

energies, forces = particle_mesh_ewald(
    positions=positions,
    charges=charges,
    cell=cell,
    neighbor_list=nl_coo,
    neighbor_shifts=shifts,
    accuracy=5e-4,              # auto-estimates everything
    # OR explicit:
    # alpha=0.3,
    # mesh_dimensions=(32, 32, 32),  # FFT grid size
    # mesh_spacing=0.5,              # alternative to mesh_dimensions
    # spline_order=4,                # B-spline order (1-6, 4 recommended)
)
```

PME algorithm: B-spline charge spreading → FFT → Green's function convolution → IFFT → force interpolation.

| B-Spline Order | Name | Notes |
|----------------|------|-------|
| 1 | Nearest-grid-point (NGP) | Lowest accuracy |
| 2 | Cloud-in-cell (CIC) | |
| 3 | Triangular-shaped cloud (TSC) | |
| 4 | Cubic B-spline | **Recommended** |
| 5-6 | Higher orders | Smoother but wider support |

### 3. Direct Coulomb

For non-periodic or real-space-only calculations:

```python
from nvalchemiops.interactions.electrostatics.coulomb import (
    coulomb_energy_forces,
    coulomb_energy,
    coulomb_forces,
)

energies, forces = coulomb_energy_forces(
    positions=positions, charges=charges, cell=cell,
    cutoff=10.0, alpha=0.0,  # alpha=0 means no Ewald splitting
    neighbor_list=nl_coo, neighbor_shifts=shifts,
)
```

### 4. Multipole Electrostatics

Supports up to quadrupoles (L_max=2, 9 coefficients per atom):

```python
from nvalchemiops.interactions.electrostatics import ewald_multipole_summation

# Multipole coefficient ordering:
# [0]: monopole (charge)
# [1]: dipole_y  [2]: dipole_z   [3]: dipole_x
# [4]: quad_xy   [5]: quad_yz    [6]: quad_3z2-r2
# [7]: quad_xz   [8]: quad_x2-y2

multipoles = torch.zeros((num_atoms, 9), dtype=torch.float64, device="cuda")
multipoles[:, 0] = charges
multipoles[:, 1:4] = dipoles       # [N, 3]
multipoles[:, 4:9] = quadrupoles   # [N, 5]

energies = ewald_multipole_summation(
    positions=positions, multipoles=multipoles, cell=cell,
    neighbor_list=nl, neighbor_shifts=shifts, accuracy=1e-6,
)

# Reciprocal-space only with response tensor (for ML training)
from nvalchemiops.interactions.electrostatics import ewald_multipole_reciprocal_space

energies, response = ewald_multipole_reciprocal_space(
    positions, multipoles, cell, k_vectors, alpha,
    compute_response=True,
)

# PME variant for large systems
from nvalchemiops.interactions.electrostatics import pme_multipole_summation
```

### 5. Parameter Estimation

```python
from nvalchemiops.interactions.electrostatics import (
    estimate_ewald_parameters,
    estimate_pme_parameters,
    estimate_pme_mesh_dimensions,
)

# Ewald: uses Kolafa-Perram formula
params = estimate_ewald_parameters(
    positions, cell, batch_idx=None, accuracy=1e-6,
)
# Returns EwaldParameters(alpha, real_space_cutoff, reciprocal_space_cutoff)

# PME: combines Ewald + mesh sizing
params = estimate_pme_parameters(
    positions, cell, batch_idx=None, accuracy=1e-6,
)
# Returns PMEParameters(alpha, mesh_dimensions, mesh_spacing, real_space_cutoff)

# Mesh dimensions from accuracy
mesh_dims = estimate_pme_mesh_dimensions(cell, accuracy=1e-6)
```

### 6. K-Vector Generation

```python
from nvalchemiops.interactions.electrostatics.k_vectors import (
    generate_k_vectors_ewald_summation,  # half-space, ~2x speedup
    generate_k_vectors_pme,              # regular grid for FFT
)

k_vectors = generate_k_vectors_ewald_summation(cell, k_cutoff=8.0)
k_vectors_pme = generate_k_vectors_pme(mesh_dimensions, cell)
```

### 7. Autograd Support

```python
# Forces via autograd (matches explicit forces)
positions.requires_grad_(True)
energies, explicit_forces = ewald_summation(positions, charges, cell, ...)
energies.sum().backward()
autograd_forces = -positions.grad  # matches explicit_forces

# Charge gradients (for ML charge prediction)
charges.requires_grad_(True)
energies = ewald_summation(positions, charges, cell, ..., compute_forces=False)
energies.sum().backward()
dE_dq = charges.grad

# Stress tensor via cell gradients
cell.requires_grad_(True)
energies, forces = ewald_summation(positions, charges, cell, ...)
energies.sum().backward()
volume = torch.abs(torch.linalg.det(cell)) + 1e-7
stress = cell.grad / volume
```

### 8. Unit Systems

Functions assume Coulomb constant k_e = 1 (atomic units). Scale for other systems:

| System | Positions | Energy | k_e |
|--------|-----------|--------|-----|
| Atomic units | Bohr | Hartree | 1 |
| eV-Angstrom | Angstrom | eV | 14.3996 |
| LAMMPS "real" | Angstrom | kcal/mol | 332.0637 |

### 9. B-Spline Utilities

```python
from nvalchemiops.interactions.electrostatics import (
    spline_spread,           # atoms → mesh grid
    spline_gather,           # mesh → atoms (scalars)
    spline_gather_vec3,      # mesh → atoms (3D vectors)
    spline_gather_gradient,  # force contributions via B-spline gradient
)
```

## Common Pitfalls

- **Float64 required**: Electrostatic calculations need `float64` for numerical stability. `float32` leads to energy drift.
- **Cell as rows**: Lattice vectors must be rows of the `cell` tensor, not columns.
- **PME batch constraint**: Mesh dimensions must be identical across all systems in a batch.
- **Energy non-convergence**: Verify cell definition and positive volume (`det(cell) > 0`).
- **Force discontinuities**: Ensure real-space cutoff aligns with erfc damping and neighbor list cutoff.
- **NaN/Inf**: Check for overlapping atoms, positive cell volume, finite charges.
- **Ewald vs PME**: Use Ewald for <5000 atoms, PME for larger systems.

## Related Skills

- [neighbor-lists.md](./neighbor-lists.md) - Build neighbor list for real-space cutoff
- [dispersion-corrections.md](./dispersion-corrections.md) - Combine with dispersion
- [math-and-utilities.md](./math-and-utilities.md) - GTO, spherical harmonics
