---
name: alchemi-toolkit-ops
description: Provides NVIDIA ALCHEMI Toolkit-Ops patterns for GPU-accelerated atomistic simulation kernels. Applies to tasks involving neighbor list construction, electrostatic interactions (Ewald, PME, Coulomb, multipole), DFT-D3 dispersion corrections, molecular dynamics, GNN-based molecular property prediction, or high-throughput computational chemistry on NVIDIA GPUs.
license: MIT
metadata:
  author: Agent Cluster
  tags: alchemi, molecular-dynamics, neighbor-list, electrostatics, ewald, pme, dft-d3, dispersion, nvidia, warp, atomistic
---

# ALCHEMI Toolkit-Ops

## Overview

GPU-accelerated library of low-level, high-performance kernels for atomistic simulations, computational chemistry, and graph neural networks. Built on NVIDIA Warp, outputs native PyTorch tensors with full `torch.compile` compatibility. Covers neighbor list construction, long-range electrostatics (Ewald/PME/multipole), and DFT-D3 dispersion corrections — all with native batch processing for heterogeneous molecular systems.

## Quick Pattern

**Incorrect — CPU-based neighbor search without batching:**

```python
from scipy.spatial import KDTree
tree = KDTree(positions)
pairs = tree.query_pairs(cutoff)
```

**Correct — GPU-accelerated neighbor list with auto-dispatch:**

```python
import torch
from nvalchemiops.neighborlist import neighbor_list

positions = torch.randn(10000, 3, device="cuda")
cell = torch.eye(3, device="cuda").unsqueeze(0) * 30.0
pbc = torch.tensor([[True, True, True]], device="cuda")

result = neighbor_list(
    positions, cutoff=5.0, cell=cell, pbc=pbc,
)
# result.neighbor_matrix: [N, max_neighbors], int32
# result.num_neighbors: [N], int32
# result.neighbor_matrix_shifts: [N, max_neighbors, 3], int32
```

## Quick Command

```bash
# Install from PyPI
pip install nvalchemi-toolkit-ops

# Install from source
pip install git+https://github.com/NVIDIA/nvalchemi-toolkit-ops.git

# Verify installation
python -c "import nvalchemiops; print(nvalchemiops.__version__)"

# Docker
docker run --gpus all nvidia/cuda:13.0.0-runtime-ubuntu24.04
# Then: pip install nvalchemi-toolkit-ops
```

## Quick Reference

### Core Modules

| Module | Import | Purpose |
|--------|--------|---------|
| Neighbor List | `nvalchemiops.neighborlist` | GPU-accelerated neighbor search |
| Ewald Summation | `nvalchemiops.interactions.electrostatics` | Long-range electrostatics O(N^2) |
| PME | `nvalchemiops.interactions.electrostatics` | Particle Mesh Ewald O(N log N) |
| Coulomb | `nvalchemiops.interactions.electrostatics.coulomb` | Direct Coulomb interactions |
| Multipole | `nvalchemiops.interactions.electrostatics` | Dipole/quadrupole Ewald/PME |
| DFT-D3 | `nvalchemiops.interactions.dispersion.dftd3` | Dispersion corrections (BJ damping) |
| Spherical Harmonics | `nvalchemiops.math.spherical_harmonics` | Real spherical harmonics |
| GTO | `nvalchemiops.math.gto` | Gaussian Type Orbital functions |

### Neighbor List Algorithms

| Method | Complexity | Auto-Selected When |
|--------|-----------|-------------------|
| `"naive"` | O(N^2) | Single system, <5000 atoms |
| `"cell_list"` | O(N) | Single system, >=5000 atoms |
| `"batch_naive"` | O(N^2)/system | Batched small systems |
| `"batch_cell_list"` | O(N)/system | Batched large systems |
| `"naive_dual_cutoff"` | O(N^2) | `cutoff2` provided |
| `"batch_naive_dual_cutoff"` | O(N^2) | Batched + `cutoff2` |

### Electrostatics Methods

| Method | Complexity | Best For |
|--------|-----------|----------|
| Ewald Summation | O(N^(3/2)) | Systems <5000 atoms |
| PME | O(N log N) | Large periodic systems |
| Direct Coulomb | O(N * pairs) | Non-periodic or real-space only |
| Multipole Ewald/PME | Extended | Dipoles and quadrupoles |

### Output Formats

| Format | Tensors | Best For |
|--------|---------|----------|
| Neighbor Matrix (dense) | `neighbor_matrix [N, max_nb]`, `num_neighbors [N]` | torch.compile, uniform density |
| Neighbor List (COO sparse) | `neighbor_list [2, pairs]`, `neighbor_ptr` | GNNs, variable density |

### Hardware Requirements

| Platform | Support | Notes |
|----------|---------|-------|
| NVIDIA GPU (CC 8.0+) | Full GPU | A100, H100 recommended |
| CUDA Toolkit 12+ | Required for GPU | Driver 570.xx+ |
| CPU (x86/ARM) | Via Warp | Development/testing |
| Apple Silicon | CPU only | macOS support |

## When to Apply

Reference these guidelines when:
- Building neighbor lists for molecular simulations or GNNs
- Computing long-range electrostatic interactions (periodic systems)
- Adding DFT-D3(BJ) dispersion corrections to DFT workflows
- Batching heterogeneous molecular systems for GPU throughput
- Integrating atomistic kernels with `torch.compile`
- Building molecular dynamics simulations with rebuild detection
- Computing multipole electrostatics for ML force fields
- Accelerating computational chemistry pipelines on NVIDIA GPUs

## Priority-Ordered Guidelines

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Neighbor Lists | CRITICAL | `neighbor-*` |
| 2 | Electrostatics | CRITICAL | `electrostatics-*` |
| 3 | Dispersion Corrections | HIGH | `dispersion-*` |
| 4 | Math & Utilities | MEDIUM | `math-*` |

## References

Full documentation with code examples in [references/](references/):

| File | Impact | Description |
|------|--------|-------------|
| [neighbor-lists.md][neighbor-lists] | CRITICAL | Auto-dispatch, cell list, rebuild detection, pre-allocation, torch.compile |
| [electrostatics.md][electrostatics] | CRITICAL | Ewald, PME, Coulomb, multipole, parameter estimation, autograd |
| [dispersion-corrections.md][dispersion] | HIGH | DFT-D3(BJ), D3Parameters, functional params, multi-pass kernels |
| [math-and-utilities.md][math-utils] | MEDIUM | Spherical harmonics, GTO, batch processing, type converters |

## Problem -> Skill Mapping

| Problem | Start With |
|---------|------------|
| Build neighbor list for molecules | [neighbor-lists.md][neighbor-lists] |
| Compute electrostatic energy/forces | [electrostatics.md][electrostatics] |
| Add dispersion corrections to DFT | [dispersion-corrections.md][dispersion] |
| Batch multiple molecular systems | [math-and-utilities.md][math-utils] |
| Optimize with torch.compile | [neighbor-lists.md][neighbor-lists] |
| Molecular dynamics with skin distance | [neighbor-lists.md][neighbor-lists] |
| Compute PME for large periodic system | [electrostatics.md][electrostatics] |
| Multipole electrostatics for ML | [electrostatics.md][electrostatics] |
| Compute coordination numbers | [dispersion-corrections.md][dispersion] |
| GNN edge construction from atoms | [neighbor-lists.md][neighbor-lists] |
| Stress tensor from cell gradients | [electrostatics.md][electrostatics] |
| Choose Ewald vs PME | [electrostatics.md][electrostatics] |

[neighbor-lists]: references/neighbor-lists.md
[electrostatics]: references/electrostatics.md
[dispersion]: references/dispersion-corrections.md
[math-utils]: references/math-and-utilities.md

## Attribution

Based on [NVIDIA ALCHEMI Toolkit-Ops](https://nvidia.github.io/nvalchemi-toolkit-ops/userguide/index.html) documentation and [nvalchemi-toolkit-ops](https://github.com/NVIDIA/nvalchemi-toolkit-ops) repository.
