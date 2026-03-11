---
title: Dispersion Corrections
impact: HIGH
tags: dft-d3, dispersion, bj-damping, coordination-number, c6, forces, virial
---

# Skill: Dispersion Corrections

GPU-accelerated DFT-D3(BJ) dispersion energy, forces, and coordination numbers.

## Quick Pattern

```python
from nvalchemiops.interactions.dispersion.dftd3 import dftd3

energy, forces, coord_num = dftd3(
    positions=positions,          # [N, 3] in Bohr
    numbers=numbers,              # [N] atomic numbers, int32
    neighbor_matrix=nm,           # dense neighbor matrix
    neighbor_matrix_shifts=shifts,
    cell=cell,                    # [B, 3, 3] lattice vectors
    a1=0.3981, a2=4.4211, s8=0.7875,  # PBE parameters
)
```

## When to Use

- Adding DFT-D3 dispersion corrections to DFT calculations
- Computing environment-dependent C6 coefficients via coordination numbers
- Computing dispersion forces and virial tensors for MD
- Batched dispersion corrections for high-throughput screening

## Step-by-Step Instructions

### 1. Core DFT-D3 Function

```python
from nvalchemiops.interactions.dispersion.dftd3 import dftd3, D3Parameters

energy, forces, coord_num = dftd3(
    positions=positions,           # [N, 3] in Bohr
    numbers=numbers,               # [N] atomic numbers, int32
    # Neighbor data (dense OR sparse):
    neighbor_matrix=nm,            # [N, max_nb], int32
    neighbor_matrix_shifts=shifts, # [N, max_nb, 3], int32
    # OR sparse:
    # neighbor_list=nl_coo,        # [2, pairs]
    # neighbor_ptr=ptr,            # CSR pointer
    # unit_shifts=unit_shifts,     # [pairs, 3]
    cell=cell,                     # [B, 3, 3] lattice vectors
    batch_idx=batch_idx,           # atom-to-system mapping
    # BJ damping parameters:
    a1=0.3981,                     # dimensionless
    a2=4.4211,                     # Bohr
    s8=0.7875,                     # C8 scaling factor
    # Optional:
    d3_params=None,                # D3Parameters or dict (auto-loaded if None)
    compute_virial=False,          # virial tensor for pressure
    s5_smoothing_on=None,          # smooth cutoff start (Bohr)
    s5_smoothing_off=None,         # smooth cutoff end (Bohr)
)
# Returns:
# energy: [num_systems], float32 — total dispersion energy (Hartree)
# forces: [N, 3], float32 — atomic forces (Hartree/Bohr)
# coord_num: [N], float32 — coordination numbers
```

With virial tensor:

```python
energy, forces, coord_num, virial = dftd3(
    ..., compute_virial=True,
)
# virial: [num_systems, 3, 3], float32 (Hartree)
```

### 2. D3Parameters Dataclass

```python
from nvalchemiops.interactions.dispersion.dftd3 import D3Parameters

d3_params = D3Parameters(
    rcov=covalent_radii,     # [max_Z+1] covalent radii in Bohr
    r4r2=r4r2_values,        # [max_Z+1] <r^4>/<r^2> expectation values
    c6ab=c6_reference,       # [max_Z+1, max_Z+1, 5, 5] C6 reference grid
    cn_ref=coord_num_ref,    # [max_Z+1, max_Z+1, 5, 5] coord num reference
)
d3_params.to(device, dtype)

# Pass as dict also works:
d3_params = {"rcov": rcov, "r4r2": r4r2, "c6ab": c6ab, "cn_ref": cn_ref}

# If d3_params=None, default values are loaded automatically
```

### 3. Functional-Specific BJ Parameters

| Functional | a1 | a2 (Bohr) | s8 |
|-----------|-----|-----------|-----|
| PBE | 0.3981 | 4.4211 | 0.7875 |
| PBE0 | 0.4145 | 4.8593 | 1.2177 |
| B3LYP | 0.3981 | 4.4211 | 1.9889 |

### 4. Unit Conversions

| Quantity | ALCHEMI Unit | Convert To |
|----------|-------------|------------|
| Positions (input) | Bohr | Angstrom × 1.8897259886 |
| Energy (output) | Hartree | eV × 27.211386245981 |
| Forces (output) | Hartree/Bohr | eV/A × 51.422 |
| a2 parameter | Bohr | — |
| Cutoffs | Bohr | — |

### 5. Multi-Pass Kernel Architecture

Four GPU passes internally:

1. **Pass 0** (PBC only): Convert integer unit cell shifts to Cartesian coordinates
2. **Pass 1**: Compute coordination numbers for all atoms
3. **Pass 2**: Interpolate C6, compute damped energies, direct forces, accumulate dE/dCN
4. **Pass 3**: Chain-rule force contributions from coordination number gradients

Dispatches to `_dftd3_nm_op` (neighbor matrix) or `_dftd3_nl_op` (neighbor list + pointer).

### 6. torch.compile Support

```python
compiled_dftd3 = torch.compile(dftd3)

energy, forces, coord_num = compiled_dftd3(
    positions=positions, numbers=numbers,
    neighbor_list=nl, neighbor_ptr=ptr,
    a1=0.3981, a2=4.4211, s8=0.7875,
    d3_params=d3_params,
    num_systems=num_systems,  # pass explicitly to prevent CUDA graph break
)
```

### 7. Smooth Cutoff (S5 Switching)

```python
# C^2 continuous switching function for smooth energy/force cutoffs
energy, forces, coord_num = dftd3(
    ...,
    s5_smoothing_on=8.0,    # start smoothing at 8 Bohr
    s5_smoothing_off=10.0,  # fully zero at 10 Bohr
)
```

### 8. Batched Dispersion

```python
# Multiple molecules in one kernel launch
positions = torch.cat([pos_mol1, pos_mol2])
numbers = torch.cat([Z_mol1, Z_mol2])
batch_idx = torch.cat([
    torch.zeros(len(pos_mol1), dtype=torch.int32),
    torch.ones(len(pos_mol2), dtype=torch.int32),
]).to("cuda")
cells = torch.stack([cell1, cell2])

energy, forces, coord_num = dftd3(
    positions, numbers,
    neighbor_matrix=nm, neighbor_matrix_shifts=shifts,
    cell=cells, batch_idx=batch_idx,
    a1=0.3981, a2=4.4211, s8=0.7875,
)
# energy: [2] — per-system dispersion energies
```

## Numerical Stability

- Log-sum-exp trick for Gaussian-weighted C6 interpolation prevents overflow
- Threshold skipping: contributions below `exp(arg) < 1e-12` are skipped
- S5 switching function provides C^2 continuity for smooth cutoffs

## Common Pitfalls

- **Units**: Positions must be in Bohr, not Angstroms. Output energy is in Hartree.
- **Atomic numbers**: Must be `int32` tensor of actual atomic numbers (1=H, 6=C, etc.).
- **Neighbor list cutoff**: Must be large enough for D3 interaction range (typically 15-20 Bohr).
- **Dense vs sparse**: Both neighbor matrix and COO formats supported; both produce identical results.
- **num_systems for torch.compile**: Pass explicitly to avoid CUDA graph breaks.

## Related Skills

- [neighbor-lists.md](./neighbor-lists.md) - Build neighbor list for D3 cutoff
- [electrostatics.md](./electrostatics.md) - Combine with electrostatic interactions
- [math-and-utilities.md](./math-and-utilities.md) - Batch processing patterns
