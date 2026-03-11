---
title: Models and Architectures
impact: CRITICAL
tags: fno, afno, graphcast, meshgraphnet, unet, mlp, custom-model, module
---

# Skill: Models and Architectures

Select and configure PhysicsNeMo model architectures for physics-informed ML tasks.

## Quick Reference

| Model | Best For | Scale |
|-------|----------|-------|
| FNO | PDE solving, spectral methods | Small to medium |
| AFNO | Global weather (FourCastNet) | Large-scale |
| GraphCast | Medium-range weather | Global |
| MeshGraphNet | Mesh-based simulation | Unstructured grids |
| X-MeshGraphNet | Large-scale mesh (100M+ cells) | Multi-scale distributed |
| Hybrid MeshGraphNet | Complex boundary conditions | Heterogeneous graphs |
| SFNO | Weather on lat-lon grids | Spatial model parallelism |
| DPOT | PDE foundation model | Spectral Fourier attention |
| DeepONet | Operator learning (branch+trunk) | Structured + unstructured |
| DoMINO | Aerodynamics, point-cloud CFD | Multi-scale, distributed |
| Transolver | General PDE solving | Transformer + TE fp8 |
| StormCast | Km-scale weather emulation | Diffusion, multi-GPU |
| 3D UNet | Voxel-based simulation | Structured 3D grids |
| FigConvUNet | External aerodynamics | Point cloud, 3D mesh |
| Pix2Pix | Image-to-image physics | 2D fields |
| DLWP | Weather prediction | Global |
| SRResNet | Super-resolution | Downscaling |
| FullyConnected | General regression | Small |

## When to Use

- Choosing a neural operator for a physics problem
- Converting a custom PyTorch model to PhysicsNeMo
- Setting up model metadata for optimization
- Configuring model hyperparameters

## Step-by-Step Instructions

### 1. Fourier Neural Operator (FNO)

Best for problems with spectral structure (Darcy flow, Navier-Stokes):

```python
from physicsnemo.models.fno.fno import FNO

model = FNO(
    in_channels=1,         # Input field channels
    out_channels=1,        # Output field channels
    decoder_layers=1,      # MLP decoder depth
    decoder_layer_size=32, # MLP decoder width
    dimension=2,           # 2D or 3D
    latent_channels=32,    # Latent space width
    num_fno_layers=4,      # Number of Fourier layers
    num_fno_modes=12,      # Fourier modes to keep
    padding=5,             # Periodic padding
).to("cuda")
```

### 2. Adaptive Fourier Neural Operator (AFNO)

Used in FourCastNet for global weather prediction at 0.25deg:

```python
from physicsnemo.models.afno.afno import AFNO

model = AFNO(
    inp_shape=(720, 1440),  # H x W grid
    in_channels=20,          # Input variables (temperature, pressure, etc.)
    out_channels=20,         # Output variables
    patch_size=(8, 8),       # Vision transformer patches
    embed_dim=768,           # Embedding dimension
    depth=12,                # Transformer depth
    num_blocks=8,            # AFNO blocks per layer
).to("cuda")
```

### 3. FullyConnected MLP

Simple regression tasks:

```python
from physicsnemo.models.mlp.fully_connected import FullyConnected

model = FullyConnected(
    in_features=32,
    out_features=64,
    num_layers=4,
    layer_size=256,
).to("cuda")

# Verify
input_tensor = torch.randn(128, 32).to("cuda")
output = model(input_tensor)
print(output.shape)  # [128, 64]
```

### 4. Convert PyTorch Model to PhysicsNeMo

Wrapping gives you checkpoint support, CUDA graphs, and AMP:

```python
import torch.nn as nn
from dataclasses import dataclass
from physicsnemo.models.meta import ModelMetaData
from physicsnemo.models.module import Module

# Step 1: Define metadata
@dataclass
class UNetMeta(ModelMetaData):
    name: str = "UNet"
    jit: bool = False
    cuda_graphs: bool = True
    amp_cpu: bool = True
    amp_gpu: bool = True

# Step 2: Extend Module instead of nn.Module
class UNet(Module):
    def __init__(self, in_channels=1, out_channels=1):
        super().__init__(meta=UNetMeta())

        self.enc1 = nn.Sequential(
            nn.Conv2d(in_channels, 64, 3, padding=1),
            nn.ReLU(inplace=True), nn.MaxPool2d(2))
        self.enc2 = nn.Sequential(
            nn.Conv2d(64, 128, 3, padding=1),
            nn.ReLU(inplace=True), nn.MaxPool2d(2))
        self.dec1 = nn.Sequential(
            nn.ConvTranspose2d(128, 64, 2, stride=2),
            nn.Conv2d(64, 64, 3, padding=1), nn.ReLU(inplace=True))
        self.dec2 = nn.Sequential(
            nn.ConvTranspose2d(64, 32, 2, stride=2),
            nn.Conv2d(32, 32, 3, padding=1), nn.ReLU(inplace=True))
        self.final = nn.Conv2d(32, out_channels, kernel_size=1)

    def forward(self, x):
        x = self.enc1(x)
        x = self.enc2(x)
        x = self.dec1(x)
        x = self.dec2(x)
        return self.final(x)
```

### 5. PhysicsNeMo-Sym Model Wrapper

For physics-informed models using the Sym Key API:

```python
from typing import Dict
import torch
from physicsnemo.sym.key import Key
from physicsnemo.sym.models.arch import Arch

class SymUNet(Arch):
    def __init__(
        self,
        input_keys=[Key("a")],
        output_keys=[Key("b")],
        in_channels=1, out_channels=1,
    ):
        super().__init__(input_keys=input_keys, output_keys=output_keys)
        self.model = UNet(in_channels, out_channels)

    def forward(self, dict_tensor: Dict[str, torch.Tensor]):
        x = self.concat_input(dict_tensor, self.input_key_dict,
                              detach_dict=None, dim=1)
        out = self.model(x)
        return self.split_output(out, self.output_key_dict, dim=1)
```

### 6. Checkpoint Save and Load

PhysicsNeMo Module provides built-in checkpointing:

```python
# Save
model = FNO(in_channels=1, out_channels=1, dimension=2,
            latent_channels=32, num_fno_layers=4,
            num_fno_modes=12, padding=5).to("cuda")
model.save("fno_checkpoint.mdlus")

# Load (architecture auto-detected)
model_loaded = physicsnemo.Module.from_checkpoint("fno_checkpoint.mdlus").to("cuda")
model_loaded.eval()
```

### 7. DoMINO (Decomposable Multi-scale Iterative Neural Operator)

For aerodynamics and point-cloud CFD (v25.03+):

```python
from physicsnemo.models.domino import DoMINO
# Local + multi-scale architecture for point clouds
# Point-cloud to point-cloud spatial projection via Warp kernels
# 10x faster training recipe (v25.06), NIM deployment (v25.08)
# Fine-tuning support for custom geometries
```

### 8. Transolver

Transformer-based PDE solver with PhysicsAttention:

```python
from physicsnemo.models.transolver import Transolver
# PhysicsAttention on surface mesh data + on-the-fly SDF values
# Transformer Engine: 25% speedup, fp8 training on Hopper+
# Scaling: Multi-GPU (DDP), Domain Parallel
```

### 9. X-MeshGraphNet

Multi-scale extension of MeshGraphNet for massive meshes (v25.11+):

```python
from physicsnemo.models.meshgraphnet import MeshGraphNet
# Partitions large graphs with halo regions for distributed processing
# Custom graphs from tessellated geometry via point clouds + kNN
# Builds multi-scale graphs for 100M+ cell meshes
# Used for automotive crash dynamics, surface force prediction
```

### 10. Hybrid MeshGraphNet

Heterogeneous graph variant for complex boundary conditions:

```python
from physicsnemo.models.meshgraphnet import MeshGraphNet
# Uses heterogeneous graph with multiple edge types
# Handles varying geometries and boundary conditions
# Example: deforming plate simulation
```

### 11. SFNO (Spherical Fourier Neural Operator)

For weather forecasting on structured lat-lon grids:

```python
from physicsnemo.models.sfno import SFNO
# Spatial model parallelism: splits model + data across GPUs
# AMP, activation checkpointing for GPU memory
# N-D tensor on structured grids
```

### 12. DPOT (Operator Transformer)

Pre-trained PDE foundation model:

```python
from physicsnemo.models.dpot import DPOT
# Spectral Fourier attention for PDE solving
# Designed as a foundation model for multiple PDE families
```

### 13. DeepONet

Operator learning with branch + trunk architecture:

```python
from physicsnemo.models.deeponet import DeepONet
# FNO branch net + FullyConnected trunk net
# Physics-informed via automatic differentiation
# Supports both structured tensors and unstructured sampling
```

### 14. Diffusion Models

Generative models for downscaling, super-resolution, and inverse problems:

```python
from physicsnemo.models.diffusion_unets import SongUNet, DhariwalUNet

# Preconditioners: VPPrecond, VEPrecond, EDMPrecond, iDDPMPrecond
# Samplers: EDM, DDPM, Diffusion Posterior Sampling (DPS)
# Multi-diffusion for >2000x2000 pixel domains
# Modular Diffusion Transformer backbone (v25.11+):
#   custom attention, tokenization, de-tokenization backends
```

Key diffusion applications:

| Application | Architecture | Special Features |
|-------------|-------------|-----------------|
| StormCast | Diffusion UNet | EDM sampler, bf16 AMP, Apex GroupNorm |
| CorrDiff | Diffusion UNet | Multi-diffusion, patch-wise grad accumulation |
| FWI | Diffusion UNet + Global Filter | Physics-informed DPS for geophysics |
| TopoDiff | Diffusion UNet + encoder | DDPM + DPS for topology optimization |

### 15. Voxel-Based Models (3D UNet, FigConvUNet)

```python
# 3D UNet — structured 3D grids (datacenter thermal digital twin)
# Physics-informed via finite differences, Multi-GPU (DDP)

# FigConvUNet — point cloud from 3D CAD or simulation mesh
# External aerodynamics, AMP (fp16/bf16), Multi-GPU (DDP)
from physicsnemo.models.figconvnet import FigConvNet
```

### 16. RNN Models

For transient physics and time-series:

```python
from physicsnemo.models.rnn import One2ManyRNN, Seq2SeqRNN
# RNN + UNet for 4D data (time + 3D volume fields)
# Gray-Scott reaction-diffusion, transient Navier-Stokes
```

## Model Selection Guide

| Problem Type | Recommended Model | Why |
|-------------|-------------------|-----|
| Regular grid PDE | FNO | Spectral efficiency |
| Global weather | AFNO, GraphCast, SFNO | Scale and accuracy |
| Unstructured mesh (<1M) | MeshGraphNet | Handles irregular geometry |
| Massive mesh (100M+) | X-MeshGraphNet | Multi-scale distributed GNN |
| Complex boundaries | Hybrid MeshGraphNet | Heterogeneous edge types |
| Aerodynamics | DoMINO, FigConvUNet, XAeroNet | Point-cloud / surface modeling |
| General PDE | Transolver, DPOT | Transformer-based flexibility |
| Operator learning | DeepONet, FNO | Branch+trunk / spectral |
| Stochastic downscaling | CorrDiff (Diffusion UNet) | Generative, multi-scale |
| Km-scale weather | StormCast (Diffusion UNet) | Convection-allowing model emulation |
| Geophysics inversion | FWI Diffusion + DPS | Physics-informed posterior sampling |
| Topology optimization | TopoDiff | Manufacturability-aware generation |
| Voxel-based CFD | 3D UNet | Structured 3D grids |
| Transient dynamics | RNN + UNet, Temporal Attn GNN | Time-series prediction |
| Super-resolution | SRResNet, Pix2Pix | Image-to-image |
| Simple regression | FullyConnected | Lightweight |
| Custom architecture | Module wrapper | Full PhysicsNeMo benefits |

## Common Pitfalls

- **Wrong `dimension` for FNO**: Use `dimension=2` for 2D grids, `dimension=3` for 3D volumes.
- **Forgetting `meta` in Module subclass**: Custom models need `ModelMetaData` for optimization features.
- **Checkpoint format**: PhysicsNeMo uses `.mdlus` format. Standard `.pt` files need manual loading.
- **Input shape mismatch**: Check `in_channels` matches your data — scientific data often has many variables per grid point.

## Related Skills

- [training-recipes.md](./training-recipes.md) - Train chosen model
- [distributed-training.md](./distributed-training.md) - Scale training
- [physics-constraints.md](./physics-constraints.md) - Add physics to model
