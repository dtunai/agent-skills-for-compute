---
title: Distributed Training
impact: HIGH
tags: distributed, ddp, multi-gpu, multi-node, torchrun, mpi
---

# Skill: Distributed Training

Scale PhysicsNeMo training across multiple GPUs and nodes using DistributedManager and DDP.

## Quick Command

```bash
# Single node, 4 GPUs
torchrun --nproc_per_node=4 train.py

# Multi-node (2 nodes, 4 GPUs each)
torchrun --nnodes=2 --nproc_per_node=4 \
  --rdzv_backend=c10d --rdzv_endpoint=head_node:29500 \
  train.py
```

## When to Use

- Training is too slow on a single GPU
- Model or data doesn't fit in single GPU memory
- Training on HPC clusters with multiple nodes
- Need to reduce training time for large-scale physics models

## Step-by-Step Instructions

### 1. Basic Distributed Setup

```python
import torch
from torch.nn.parallel import DistributedDataParallel
from physicsnemo.distributed import DistributedManager
from physicsnemo.models.fno.fno import FNO
from physicsnemo.datapipes.benchmarks.darcy import Darcy2D
from physicsnemo.metrics.general.mse import mse
from physicsnemo.utils import StaticCaptureTraining

def main():
    # Initialize distributed (auto-detects environment)
    DistributedManager.initialize()
    dist = DistributedManager()

    # Create model on correct device
    model = FNO(
        in_channels=1, out_channels=1,
        decoder_layers=1, decoder_layer_size=32,
        dimension=2, latent_channels=32,
        num_fno_layers=4, num_fno_modes=12, padding=5,
    ).to(dist.device)

    # Wrap with DDP
    if dist.distributed:
        ddps = torch.cuda.Stream()
        with torch.cuda.stream(ddps):
            model = DistributedDataParallel(
                model,
                device_ids=[dist.local_rank],
                output_device=dist.device,
                broadcast_buffers=dist.broadcast_buffers,
                find_unused_parameters=dist.find_unused_parameters,
            )
        torch.cuda.current_stream().wait_stream(ddps)

    # Setup optimizer and data
    optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
    scheduler = torch.optim.lr_scheduler.LambdaLR(
        optimizer, lr_lambda=lambda step: 0.85**step)

    normaliser = {
        "permeability": (1.25, 0.75),
        "darcy": (4.52e-2, 2.79e-2),
    }
    dataloader = Darcy2D(
        resolution=256, batch_size=64,
        nr_permeability_freq=5, normaliser=normaliser)

    # Optimized training step
    @StaticCaptureTraining(model=model, optim=optimizer, cuda_graph_warmup=11)
    def training_step(invar, outvar):
        predvar = model(invar)
        loss = mse(predvar, outvar)
        return loss

    # Training loop
    dataloader = iter(dataloader)
    for i in range(20):
        batch = next(dataloader)
        loss = training_step(batch["permeability"], batch["darcy"])
        scheduler.step()

if __name__ == "__main__":
    main()
```

### 2. DistributedManager Properties

```python
dist = DistributedManager()

dist.rank          # Global rank
dist.local_rank    # Rank within node
dist.world_size    # Total processes
dist.device        # Assigned device (cuda:N)
dist.distributed   # Whether DDP is active
dist.broadcast_buffers
dist.find_unused_parameters
```

### 3. Data Parallelism with DistributedSampler

```python
from torch.utils.data import DataLoader, DistributedSampler

dataset = MyPhysicsDataset(...)
sampler = DistributedSampler(dataset, num_replicas=dist.world_size,
                              rank=dist.rank, shuffle=True)
dataloader = DataLoader(dataset, batch_size=32, sampler=sampler,
                        num_workers=4, pin_memory=True)

# Each epoch, set epoch for shuffling
for epoch in range(num_epochs):
    sampler.set_epoch(epoch)
    for batch in dataloader:
        loss = training_step(batch)
```

### 4. Launch Configurations

**Single node, multiple GPUs:**
```bash
torchrun --nproc_per_node=4 train.py
```

**Multi-node via Slurm:**
```bash
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=4
#SBATCH --gpus-per-node=4

srun torchrun \
  --nnodes=$SLURM_NNODES \
  --nproc_per_node=4 \
  --rdzv_backend=c10d \
  --rdzv_endpoint=$(hostname):29500 \
  train.py
```

**Docker with multiple GPUs:**
```bash
docker run --gpus all --shm-size=1g \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --runtime nvidia -v ${PWD}:/workspace \
  nvcr.io/nvidia/physicsnemo/physicsnemo:latest \
  torchrun --nproc_per_node=4 /workspace/train.py
```

## Scaling Recommendations

| GPUs | Batch Size Scaling | Learning Rate Scaling |
|------|-------------------|----------------------|
| 1 | Base (B) | Base (lr) |
| 4 | 4 * B | 4 * lr (linear) or 2 * lr (sqrt) |
| 8 | 8 * B | 8 * lr (linear) or 2.8 * lr (sqrt) |
| 16+ | 16 * B | Use warmup + linear scaling |

## Common Pitfalls

- **Missing `DistributedManager.initialize()`**: Must be called before any distributed ops.
- **Wrong batch size**: With DDP, effective batch = per_gpu_batch * world_size. Scale LR accordingly.
- **Logging from all ranks**: Only log from rank 0: `if dist.rank == 0: print(...)`.
- **Not setting DistributedSampler epoch**: Without `set_epoch()`, data isn't reshuffled across epochs.
- **NCCL timeout on slow nodes**: Set `NCCL_TIMEOUT` environment variable for large clusters.

## Related Skills

- [training-recipes.md](./training-recipes.md) - Single-GPU training patterns
- [models-and-architectures.md](./models-and-architectures.md) - Model configuration
- [data-pipelines.md](./data-pipelines.md) - Data loading for distributed
