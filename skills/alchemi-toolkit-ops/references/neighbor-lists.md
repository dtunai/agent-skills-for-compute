---
title: Neighbor Lists
impact: CRITICAL
tags: neighbor-list, cell-list, spatial-decomposition, pbc, dual-cutoff, torch-compile, rebuild
---

# Skill: Neighbor Lists

GPU-accelerated neighbor list construction for atomistic simulations and GNNs.

## Quick Pattern

```python
from nvalchemiops.neighborlist import neighbor_list

result = neighbor_list(
    positions,        # [N, 3] atomic coordinates
    cutoff=5.0,       # interaction cutoff distance
    cell=cell,        # [1, 3, 3] lattice vectors (rows)
    pbc=pbc,          # [1, 3] periodic boundary conditions
)
# Auto-selects: naive (<5000 atoms) or cell_list (>=5000)
# Auto-prepends batch_ if batch_idx or batch_ptr provided
```

## When to Use

- Building neighbor lists for molecular simulations
- Constructing edge indices for molecular GNNs
- Periodic boundary conditions with PBC shift vectors
- Dual-cutoff neighbor lists for multi-range potentials
- Molecular dynamics with skin distance and rebuild detection

## Step-by-Step Instructions

### 1. Auto-Dispatch Neighbor List

The `neighbor_list()` function auto-selects the best algorithm:

```python
import torch
from nvalchemiops.neighborlist import neighbor_list

positions = torch.randn(10000, 3, device="cuda")
cell = torch.eye(3, device="cuda").unsqueeze(0) * 30.0
pbc = torch.tensor([[True, True, True]], device="cuda")

result = neighbor_list(positions, cutoff=5.0, cell=cell, pbc=pbc)

# Dense output (default):
neighbor_matrix = result.neighbor_matrix        # [N, max_neighbors], int32
num_neighbors = result.num_neighbors             # [N], int32
shifts = result.neighbor_matrix_shifts           # [N, max_neighbors, 3], int32

# COO sparse output:
result = neighbor_list(
    positions, cutoff=5.0, cell=cell, pbc=pbc,
    return_neighbor_list=True,
)
neighbor_coo = result.neighbor_list     # [2, num_pairs]
neighbor_ptr = result.neighbor_ptr      # CSR pointer
shifts_coo = result.neighbor_list_shifts  # [num_pairs, 3]
```

Auto-dispatch logic:
1. `cutoff2` provided → dual cutoff method
2. `total_atoms >= 5000` → `"cell_list"` (O(N))
3. Otherwise → `"naive"` (O(N^2))
4. `batch_idx` or `batch_ptr` provided → prepends `"batch_"`

### 2. Individual Algorithm Functions

```python
from nvalchemiops.neighborlist import (
    naive_neighbor_list,         # O(N^2), torch.compile for non-PBC
    cell_list,                   # O(N), spatial decomposition
    batch_naive_neighbor_list,   # Batched O(N^2)
    batch_cell_list,             # Batched O(N)
    naive_neighbor_list_dual_cutoff,       # Two cutoffs, one pass
    batch_naive_neighbor_list_dual_cutoff, # Batched dual cutoff
)

# Cell list: separate build + query for reuse
from nvalchemiops.neighborlist import build_cell_list, query_cell_list

cell_data = build_cell_list(positions, cell, pbc, cutoff)  # expensive
result = query_cell_list(cell_data, positions, cutoff)      # cheap (reusable)
```

### 3. Key Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `positions` | `Tensor [N, 3]` | Atomic coordinates |
| `cutoff` | `float` | Interaction cutoff distance |
| `cutoff2` | `float` | Second cutoff (dual-cutoff mode) |
| `cell` | `Tensor [1,3,3]` or `[B,3,3]` | Unit cell lattice vectors (rows) |
| `pbc` | `Tensor [1,3]` or `[B,3]`, bool | Periodic boundary per axis |
| `batch_idx` | `Tensor [N]`, int32 | Atom-to-system mapping |
| `batch_ptr` | `Tensor` | Batch pointer array |
| `method` | `str` | Force specific algorithm |
| `return_neighbor_list` | `bool` | COO sparse instead of dense |
| `max_neighbors` | `int` | Max neighbors per atom (auto-estimated if omitted) |
| `atomic_density` | `float` (0.5) | Atoms/volume for estimation |
| `safety_factor` | `float` (5.0) | Multiplier for neighbor estimates |
| `half_fill` | `bool` | Store only i<j pairs (avoid double-counting) |
| `fill_value` | `int` | Padding value in matrix (typically `num_atoms`) |

### 4. Output Format Conversion

```python
from nvalchemiops.neighborlist import get_neighbor_list_from_neighbor_matrix

# Dense → COO sparse
neighbor_coo, neighbor_ptr, shifts_coo = get_neighbor_list_from_neighbor_matrix(
    neighbor_matrix, num_neighbors, neighbor_matrix_shifts,
    fill_value=num_atoms,
)
# neighbor_coo: [2, num_pairs] — source/target atom indices
# Use as edge_index for PyG: Data(edge_index=neighbor_coo)
```

### 5. Estimation Utilities

```python
from nvalchemiops.neighborlist import (
    estimate_max_neighbors,
    estimate_cell_list_sizes,
    allocate_cell_list,
)

# Estimate max neighbors (rounded to next power of 2)
max_nb = estimate_max_neighbors(cutoff=5.0, atomic_density=0.3, safety_factor=1.0)
# Formula: 2^ceil(log2(S * rho * (4/3)*pi*r^3))

# Estimate cell list allocation
max_cells, search_radius = estimate_cell_list_sizes(cell, pbc, cutoff, max_nbins=1000)

# Pre-allocate all cell list structures
cell_data = allocate_cell_list(positions, cell, pbc, cutoff, device="cuda")
```

### 6. Rebuild Detection (Molecular Dynamics)

```python
from nvalchemiops.neighborlist import (
    cell_list_needs_rebuild,       # torch.compile compatible
    neighbor_list_needs_rebuild,   # skin distance check
)

# Skin distance pattern: build with effective_cutoff, query with actual
cutoff = 5.0
skin_distance = 1.0
effective_cutoff = cutoff + skin_distance

# Build initial neighbor list with effective cutoff
result = neighbor_list(positions, cutoff=effective_cutoff, cell=cell, pbc=pbc)

# During MD: check if rebuild needed
needs_rebuild = cell_list_needs_rebuild(
    positions, atom_to_cell_mapping, cells_per_dimension, cell, pbc,
)
# OR: check if any atom moved beyond skin_distance / 2
needs_rebuild = neighbor_list_needs_rebuild(
    positions, old_positions, skin_distance,
)
```

### 7. Pre-Allocation for torch.compile

```python
# Pre-allocate output tensors (required for torch.compile)
num_atoms = positions.shape[0]
max_nb = estimate_max_neighbors(cutoff=5.0)

neighbor_matrix = torch.full(
    (num_atoms, max_nb), num_atoms, dtype=torch.int32, device="cuda"
)
neighbor_matrix_shifts = torch.zeros(
    (num_atoms, max_nb, 3), dtype=torch.int32, device="cuda"
)
num_neighbors = torch.zeros(num_atoms, dtype=torch.int32, device="cuda")

# Pass pre-allocated tensors
result = neighbor_list(
    positions, cutoff=5.0, cell=cell, pbc=pbc,
    neighbor_matrix=neighbor_matrix,
    neighbor_matrix_shifts=neighbor_matrix_shifts,
    num_neighbors=num_neighbors,
    fill_value=num_atoms,
)

# Compile the entire workflow
compiled_nl = torch.compile(neighbor_list)
result = compiled_nl(positions, cutoff=5.0, cell=cell, pbc=pbc)
```

### 8. Batched Systems

```python
# Process multiple molecular systems in one kernel launch
positions = torch.cat([pos_mol1, pos_mol2, pos_mol3])
batch_idx = torch.cat([
    torch.zeros(len(pos_mol1), dtype=torch.int32),
    torch.ones(len(pos_mol2), dtype=torch.int32),
    torch.full((len(pos_mol3),), 2, dtype=torch.int32),
]).to("cuda")

cells = torch.stack([cell1, cell2, cell3])  # [B, 3, 3]
pbc = torch.tensor([[True, True, True]] * 3, device="cuda")

result = neighbor_list(
    positions, cutoff=5.0, cell=cells, pbc=pbc, batch_idx=batch_idx,
)
# Auto-dispatches to batch_cell_list or batch_naive
```

## Common Pitfalls

- **Silent neighbor truncation**: If `max_neighbors` is too small, neighbors beyond the limit are silently dropped. Monitor `num_neighbors.max()` against allocation.
- **Cell vectors as rows**: `cell` tensor expects lattice vectors as rows, not columns.
- **Sparse waste vs dense truncation**: Dense format wastes memory for sparse systems; sparse format has conversion overhead. Choose based on system uniformity.
- **return_neighbor_list overhead**: Setting `return_neighbor_list=True` converts from dense internally, adding overhead.
- **PBC shift vectors**: Always use `neighbor_matrix_shifts` / `neighbor_list_shifts` for minimum image convention distances.

## Related Skills

- [electrostatics.md](./electrostatics.md) - Use neighbor list for Ewald/PME
- [dispersion-corrections.md](./dispersion-corrections.md) - Use neighbor list for DFT-D3
- [math-and-utilities.md](./math-and-utilities.md) - Batch processing patterns
