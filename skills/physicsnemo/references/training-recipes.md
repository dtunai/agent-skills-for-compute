---
title: Training Recipes
impact: CRITICAL
tags: training, fno, cuda-graphs, optimizer, loss, static-capture, darcy
---

# Skill: Training Recipes

Build optimized training pipelines with PhysicsNeMo models, CUDA graphs, and scientific data.

## Quick Pattern

**Incorrect — vanilla PyTorch without PhysicsNeMo optimization:**

```python
model = FNO(...).cuda()
for batch in dataloader:
    pred = model(batch["input"])
    loss = F.mse_loss(pred, batch["target"])
    loss.backward()
    optimizer.step()
```

**Correct — PhysicsNeMo with CUDA graph capture:**

```python
from physicsnemo.utils import StaticCaptureTraining

@StaticCaptureTraining(model=model, optim=optimizer, cuda_graph_warmup=11)
def training_step(invar, outvar):
    predvar = model(invar)
    loss = mse(predvar, outvar)
    return loss
```

## When to Use

- Training any PhysicsNeMo model (FNO, AFNO, etc.)
- Setting up optimized training with CUDA graphs
- Configuring loss functions for physics problems
- Running benchmark problems (Darcy flow, etc.)

## Step-by-Step Instructions

### 1. Basic FNO Training (Darcy Flow)

```python
import torch
import physicsnemo
from physicsnemo.datapipes.benchmarks.darcy import Darcy2D
from physicsnemo.metrics.general.mse import mse
from physicsnemo.models.fno.fno import FNO

normaliser = {
    "permeability": (1.25, 0.75),
    "darcy": (4.52e-2, 2.79e-2),
}

dataloader = Darcy2D(
    resolution=256, batch_size=64,
    nr_permeability_freq=5, normaliser=normaliser
)

model = FNO(
    in_channels=1, out_channels=1,
    decoder_layers=1, decoder_layer_size=32,
    dimension=2, latent_channels=32,
    num_fno_layers=4, num_fno_modes=12, padding=5,
).to("cuda")

optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
scheduler = torch.optim.lr_scheduler.LambdaLR(
    optimizer, lr_lambda=lambda step: 0.85**step
)

dataloader = iter(dataloader)
for i in range(20):
    batch = next(dataloader)
    truth = batch["darcy"]
    pred = model(batch["permeability"])
    loss = mse(pred, truth)
    loss.backward()
    optimizer.step()
    scheduler.step()
    print(f"Iteration: {i}. Loss: {loss.detach().cpu().numpy()}")
```

### 2. Optimized Training with CUDA Graphs

CUDA graphs eliminate CPU-GPU synchronization overhead:

```python
from physicsnemo.utils import StaticCaptureTraining

model = FNO(
    in_channels=1, out_channels=1,
    decoder_layers=1, decoder_layer_size=32,
    dimension=2, latent_channels=32,
    num_fno_layers=4, num_fno_modes=12, padding=5,
).to("cuda")

optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
scheduler = torch.optim.lr_scheduler.LambdaLR(
    optimizer, lr_lambda=lambda step: 0.85**step
)

@StaticCaptureTraining(
    model=model,
    optim=optimizer,
    cuda_graph_warmup=11,  # Warmup iterations before capture
)
def training_step(invar, outvar):
    predvar = model(invar)
    loss = mse(predvar, outvar)
    return loss

dataloader = iter(Darcy2D(resolution=256, batch_size=8,
                          nr_permeability_freq=5, normaliser=normaliser))
for i in range(20):
    batch = next(dataloader)
    loss = training_step(batch["permeability"], batch["darcy"])
    scheduler.step()
```

### 3. Training from Example Recipes

PhysicsNeMo examples are cloned separately (not in pip wheel):

```bash
git clone https://github.com/NVIDIA/physicsnemo.git
cd physicsnemo/examples/cfd/darcy_fno/
pip install -r requirements.txt
python train_fno_darcy.py
```

### 4. Common Training Patterns

**Optimizer setup:**
```python
optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
scheduler = torch.optim.lr_scheduler.LambdaLR(
    optimizer, lr_lambda=lambda step: 0.85**step)

# Or cosine annealing
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer, T_max=1000, eta_min=1e-6)
```

**Loss functions:**
```python
from physicsnemo.metrics.general.mse import mse

# MSE loss (most common)
loss = mse(pred, truth)

# Weighted MSE for multi-output
loss = mse(pred_pressure, truth_pressure) + 0.1 * mse(pred_velocity, truth_velocity)
```

**Gradient clipping:**
```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

## Training Configuration Checklist

| Parameter | Typical Value | Notes |
|-----------|--------------|-------|
| Learning rate | 1e-3 to 1e-2 | Start high, decay |
| Batch size | 8-64 | Limited by GPU memory |
| CUDA graph warmup | 11 | First iterations without capture |
| LR schedule | Exponential or cosine | 0.85^step is common |
| Gradient clipping | 1.0 | Prevents explosion |
| Mixed precision | AMP | Enabled in ModelMetaData |

## Common Pitfalls

- **CUDA graph with dynamic shapes**: `StaticCaptureTraining` requires fixed tensor shapes. Use fixed batch sizes.
- **Forgetting normalization**: Scientific data ranges vary wildly. Always normalize inputs/outputs.
- **Wrong dimension parameter**: `FNO(dimension=2)` for 2D problems, `dimension=3` for 3D.
- **Memory overflow**: Reduce batch size or resolution first. Use gradient accumulation if needed.

## Related Skills

- [models-and-architectures.md](./models-and-architectures.md) - Choose the right model
- [distributed-training.md](./distributed-training.md) - Scale to multiple GPUs
- [data-pipelines.md](./data-pipelines.md) - Load scientific data
