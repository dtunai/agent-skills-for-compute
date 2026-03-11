---
name: physicsnemo
description: Provides NVIDIA PhysicsNeMo patterns for building, training, and deploying physics-informed machine learning models at scale. Applies to tasks involving neural operators (FNO, AFNO), weather/climate modeling (FourCastNet, CorrDiff, GraphCast), scientific simulation surrogates, physics-constrained training, or distributed GPU training for scientific AI.
license: MIT
metadata:
  author: Agent Cluster
  tags: physicsnemo, physics-ml, neural-operator, fno, weather, simulation, nvidia, pinn
---

# PhysicsNeMo

## Overview

Open-source Python framework for building, training, and fine-tuning physics-informed AI models on NVIDIA GPUs. PhysicsNeMo provides optimized model architectures (FNO, AFNO, GNNs, diffusion models), scalable distributed training, and physics-constrained learning — enabling real-time AI surrogates for scientific simulation across weather, CFD, molecular dynamics, and engineering domains.

## Quick Pattern

**Incorrect — manual PyTorch training without physics optimization:**

```python
model = MyModel().cuda()
for batch in dataloader:
    pred = model(batch)
    loss = F.mse_loss(pred, target)
    loss.backward()
```

**Correct — PhysicsNeMo with optimized training and CUDA graphs:**

```python
import physicsnemo
from physicsnemo.datapipes.benchmarks.darcy import Darcy2D
from physicsnemo.metrics.general.mse import mse
from physicsnemo.models.fno.fno import FNO

model = FNO(
    in_channels=1, out_channels=1,
    dimension=2, latent_channels=32,
    num_fno_layers=4, num_fno_modes=12,
    padding=5,
).to("cuda")

optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
dataloader = Darcy2D(resolution=256, batch_size=64)

for batch in dataloader:
    pred = model(batch["permeability"])
    loss = mse(pred, batch["darcy"])
    loss.backward()
    optimizer.step()
```

## Quick Command

```bash
# Install PhysicsNeMo
pip install nvidia-physicsnemo

# Install with all optional dependencies
pip install nvidia-physicsnemo[all]

# Install PhysicsNeMo-Sym (physics-informed constraints)
pip install Cython
pip install nvidia-physicsnemo-sym --no-build-isolation

# Run with Docker container
docker pull nvcr.io/nvidia/physicsnemo/physicsnemo:latest
docker run --shm-size=1g --ulimit memlock=-1 --ulimit stack=67108864 \
  --runtime nvidia -v ${PWD}:/workspace \
  -it --rm nvcr.io/nvidia/physicsnemo/physicsnemo:latest bash

# Distributed training
torchrun --nproc_per_node=4 train.py

# Clone training recipes
git clone https://github.com/NVIDIA/physicsnemo.git
```

## Quick Reference

### Built-in Model Architectures

| Model | Module | Domain |
|-------|--------|--------|
| FNO | `physicsnemo.models.fno.fno.FNO` | General PDE solving |
| AFNO | `physicsnemo.models.afno.afno.AFNO` | Weather forecasting (FourCastNet) |
| GraphCast | `physicsnemo.models.graphcast` | Global weather prediction |
| MeshGraphNet | `physicsnemo.models.meshgraphnet` | Mesh-based simulation |
| X-MeshGraphNet | `physicsnemo.models.meshgraphnet` | Large-scale mesh (100M+ cells) |
| Hybrid MeshGraphNet | `physicsnemo.models.meshgraphnet` | Complex boundary conditions |
| SFNO | `physicsnemo.models.sfno` | Weather on lat-lon grids |
| DeepONet | `physicsnemo.models.deeponet` | Operator learning (branch+trunk) |
| Pix2Pix | `physicsnemo.models.pix2pix` | Image-to-image physics |
| DLWP | `physicsnemo.models.dlwp` | Weather prediction |
| DoMINO | `physicsnemo.models.domino` | Aerodynamics, point-cloud CFD |
| Transolver | `physicsnemo.models.transolver` | General PDE, fp8 on Hopper |
| DPOT | `physicsnemo.models.dpot` | Pre-trained operator transformer |
| SRResNet | `physicsnemo.models.srrn` | Super-resolution |
| RNN | `physicsnemo.models.rnn` | Transient physics |
| FigConvNet/UNet | `physicsnemo.models.figconvnet` | External aerodynamics |
| 3D UNet | `physicsnemo.models.unet` | Voxel-based 3D simulation |
| FullyConnected | `physicsnemo.models.mlp.fully_connected` | General MLP |
| Diffusion UNets | `physicsnemo.models.diffusion_unets` | Downscaling, FWI, topology opt |

### Core Utilities

| Component | Import | Purpose |
|-----------|--------|---------|
| Distributed | `physicsnemo.distributed.DistributedManager` | Multi-GPU orchestration |
| ShardTensor | `physicsnemo.distributed.scatter_tensor` | Domain parallelism |
| CUDA Graphs | `physicsnemo.utils.StaticCaptureTraining` | Training optimization |
| PhysicsInformer | `physicsnemo.utils.physics_informer.PhysicsInformer` | PDE residual computation |
| Metrics | `physicsnemo.metrics.general.mse` | Loss computation |
| Datapipes | `physicsnemo.datapipes.benchmarks.*` | Scientific data loading |
| Curator | `physicsnemo_curator.etl.*` | GPU-accelerated data curation |
| Module | `physicsnemo.models.module.Module` | Base model with checkpoint support |
| Warp Neighbors | `physicsnemo.utils.neighbors` | GPU ball query for point clouds |
| Logging | `physicsnemo.launch.logging.LaunchLogger` | MLflow / W&B integration |
| Active Learning | `physicsnemo.active_learning.driver.Driver` | Iterative model improvement |

### Hardware Requirements

| GPU Family | Examples | Notes |
|-----------|----------|-------|
| Blackwell | B100, RTX PRO 6000 | Latest, full support |
| Hopper | H100 | Recommended, fp8 via TE |
| Ada Lovelace | RTX 40xx | Supported |
| Ampere | A100, A30, A6000 | Recommended for training |
| Turing | T4 | Inference focused |
| ARM64 | — | Supported |

OS: Ubuntu 24.04 (WSL supported). Python >= 3.10. Docker: `nvcr.io/nvidia/physicsnemo/physicsnemo:<tag>`

### Domain-Specific Packages

| Package | Purpose | Install |
|---------|---------|---------|
| PhysicsNeMo-CFD | CFD inference (DoMINO NIM) | `physicsnemo.cfd` |
| PhysicsNeMo Curator | Data curation ETL | `physicsnemo-curator` |
| Earth2Studio | Weather/climate inference | `earth2studio` |

### Example Applications

| Application | Domain | Key Models |
|-------------|--------|------------|
| FourCastNet (AFNO) | Global weather | AFNO at 0.25deg resolution |
| CorrDiff | Km-scale downscaling | Diffusion + multi-diffusion |
| StormCast | Convection-allowing weather | Diffusion UNet, EDM sampler |
| GraphCast | Medium-range weather | Distributed GNN |
| DoMINO Aerodynamics | External CFD | Multi-scale neural operator |
| XAeroNet | External aerodynamics | Surface + volume prediction |
| Crash Dynamics | Structural mechanics | X-MeshGraphNet (v25.11) |
| Datacenter Thermal | Digital twin | 3D UNet, physics-informed FD |
| FWI Geophysics | Subsurface inversion | Diffusion + DPS |
| TopoDiff | Design optimization | Diffusion + manufacturability |
| Darcy Flow | CFD benchmark | FNO, Transolver, DeepONet |
| Blood Flow 1D | Healthcare | MeshGraphNet reduced-order |
| Helmholtz | Wave equations | PhysicsNeMo-Sym PINN |

## When to Apply

Reference these guidelines when:
- Training neural operators (FNO, AFNO, DoMINO) for PDE solving
- Building weather/climate AI models (GraphCast, CorrDiff, StormCast)
- Creating physics-informed neural networks (PINNs)
- Scaling scientific AI training to multi-GPU clusters
- Developing AI surrogates for CFD, structural mechanics, or engineering simulation
- Converting PyTorch models to PhysicsNeMo for optimization
- Deploying physics-AI models via NIM or Earth2Studio
- Curating engineering datasets with GPU-accelerated ETL pipelines
- Integrating external kernels (cuML, Warp) with torch.compile
- Migrating GNN code from DGL to PyTorch Geometric

## Priority-Ordered Guidelines

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Training Recipes | CRITICAL | `training-*` |
| 2 | Model Architectures | CRITICAL | `models-*` |
| 3 | Distributed Training | HIGH | `distributed-*` |
| 4 | Data Pipelines | HIGH | `data-*` |
| 5 | Advanced Features | HIGH | `advanced-*` |
| 6 | Deployment | MEDIUM | `deployment-*` |
| 7 | Physics Constraints | MEDIUM | `physics-*` |

## References

Full documentation with code examples in [references/](references/):

| File | Impact | Description |
|------|--------|-------------|
| [training-recipes.md][training-recipes] | CRITICAL | FNO training, CUDA graphs, optimizer setup, loss functions |
| [models-and-architectures.md][models-and-architectures] | CRITICAL | 18+ architectures: GNNs, transformers, neural operators, diffusion, voxel |
| [distributed-training.md][distributed-training] | HIGH | DistributedManager, DDP, multi-node, torchrun patterns |
| [data-pipelines.md][data-pipelines] | HIGH | Datapipes, normalization, Curator ETL, ERA5 download |
| [advanced-features.md][advanced-features] | HIGH | PhysicsInformer, torch.compile, Warp/TE layers, DGL→PyG, active learning |
| [deployment.md][deployment] | MEDIUM | Checkpointing, DoMINO NIM, Earth2Studio, ONNX export |
| [physics-constraints.md][physics-constraints] | MEDIUM | PhysicsNeMo-Sym, PINNs, custom physics losses, Key API |

## Problem -> Skill Mapping

| Problem | Start With |
|---------|------------|
| Train a neural operator | [training-recipes.md][training-recipes] |
| Choose a model architecture | [models-and-architectures.md][models-and-architectures] |
| Scale to multiple GPUs | [distributed-training.md][distributed-training] |
| Load scientific data | [data-pipelines.md][data-pipelines] |
| Deploy trained model | [deployment.md][deployment] |
| Add physics constraints | [physics-constraints.md][physics-constraints] |
| Build weather AI model | [models-and-architectures.md][models-and-architectures] -> [training-recipes.md][training-recipes] |
| Convert PyTorch model | [models-and-architectures.md][models-and-architectures] |
| Optimize training speed | [training-recipes.md][training-recipes] -> [distributed-training.md][distributed-training] |
| Add physics loss to data model | [advanced-features.md][advanced-features] |
| Active learning loop | [advanced-features.md][advanced-features] |
| Domain decomposition | [advanced-features.md][advanced-features] |
| Experiment tracking | [advanced-features.md][advanced-features] |
| Evaluate weather model | [advanced-features.md][advanced-features] |
| Curate engineering datasets | [data-pipelines.md][data-pipelines] |
| Run DoMINO NIM inference | [deployment.md][deployment] |
| Run weather forecast with AI | [deployment.md][deployment] |
| Integrate cuML/Warp with torch.compile | [advanced-features.md][advanced-features] |
| Migrate DGL to PyG | [advanced-features.md][advanced-features] |
| Simulate 100M+ cell mesh | [models-and-architectures.md][models-and-architectures] -> [distributed-training.md][distributed-training] |
| Topology optimization | [models-and-architectures.md][models-and-architectures] |
| Full waveform inversion | [models-and-architectures.md][models-and-architectures] |

[training-recipes]: references/training-recipes.md
[models-and-architectures]: references/models-and-architectures.md
[distributed-training]: references/distributed-training.md
[data-pipelines]: references/data-pipelines.md
[deployment]: references/deployment.md
[advanced-features]: references/advanced-features.md
[physics-constraints]: references/physics-constraints.md

## Attribution

Based on [NVIDIA PhysicsNeMo](https://docs.nvidia.com/physicsnemo/latest/overview.html) documentation and [physicsnemo](https://github.com/NVIDIA/physicsnemo) repository.
