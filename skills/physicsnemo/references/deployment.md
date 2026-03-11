---
title: Deployment
impact: MEDIUM
tags: checkpoint, inference, export, mdlus, eval, production
---

# Skill: Deployment

Save, load, and run PhysicsNeMo models for inference and production deployment.

## Quick Pattern

```python
import torch
import physicsnemo

# Save trained model
model.save("trained_model.mdlus")

# Load and run inference
model = physicsnemo.Module.from_checkpoint("trained_model.mdlus").to("cuda")
model.eval()
with torch.inference_mode():
    output = model(input_tensor)
```

## When to Use

- Saving model checkpoints during/after training
- Loading trained models for inference
- Exporting models for production deployment
- Running real-time physics predictions

## Step-by-Step Instructions

### 1. Save Checkpoint

```python
from physicsnemo.models.fno.fno import FNO

model = FNO(
    in_channels=1, out_channels=1,
    decoder_layers=1, decoder_layer_size=32,
    dimension=2, latent_channels=32,
    num_fno_layers=4, num_fno_modes=12, padding=5,
).to("cuda")

# After training...
model.save("fno_darcy.mdlus")
```

### 2. Load and Run Inference

```python
import torch
import physicsnemo

# Load model (architecture auto-detected from checkpoint)
model = physicsnemo.Module.from_checkpoint("fno_darcy.mdlus").to("cuda")
model.eval()

# Run inference
with torch.inference_mode():
    input_tensor = torch.ones(8, 1, 256, 256).to("cuda")
    output = model(input_tensor)
    print(output.shape)  # [8, 1, 256, 256]
```

### 3. Periodic Checkpointing During Training

```python
for epoch in range(num_epochs):
    for batch in dataloader:
        loss = training_step(batch)

    # Save every N epochs
    if (epoch + 1) % save_every == 0:
        model.save(f"checkpoint_epoch{epoch+1}.mdlus")
        print(f"Saved checkpoint at epoch {epoch+1}")

# Save final model
model.save("final_model.mdlus")
```

### 4. Inference Pipeline

```python
import torch
import numpy as np
import physicsnemo

class PhysicsPredictor:
    def __init__(self, checkpoint_path, device="cuda"):
        self.model = physicsnemo.Module.from_checkpoint(checkpoint_path).to(device)
        self.model.eval()
        self.device = device
        self.normaliser = None

    def set_normaliser(self, input_stats, output_stats):
        self.normaliser = {
            "input": input_stats,   # (mean, std)
            "output": output_stats,
        }

    @torch.inference_mode()
    def predict(self, input_data):
        if isinstance(input_data, np.ndarray):
            input_data = torch.tensor(input_data, dtype=torch.float32)

        x = input_data.to(self.device)

        # Normalize input
        if self.normaliser:
            mean, std = self.normaliser["input"]
            x = (x - mean) / std

        # Predict
        output = self.model(x)

        # Denormalize output
        if self.normaliser:
            mean, std = self.normaliser["output"]
            output = output * std + mean

        return output.cpu().numpy()

# Usage
predictor = PhysicsPredictor("fno_darcy.mdlus")
predictor.set_normaliser(
    input_stats=(1.25, 0.75),
    output_stats=(4.52e-2, 2.79e-2),
)
result = predictor.predict(input_array)
```

### 5. Batch Inference

```python
@torch.inference_mode()
def batch_inference(model, data_loader, device="cuda"):
    model.eval()
    all_predictions = []

    for batch in data_loader:
        inputs = batch["input"].to(device)
        predictions = model(inputs)
        all_predictions.append(predictions.cpu())

    return torch.cat(all_predictions, dim=0)
```

### 6. ONNX Export (PhysicsNeMo Deploy API)

```python
from physicsnemo.deploy.onnx import export_to_onnx_stream, run_onnx_inference

# Export to ONNX byte stream (preferred)
model.eval()
sample_input = torch.randn(1, 1, 256, 256).to("cuda")
onnx_stream = export_to_onnx_stream(model, sample_input)

# Run ONNX Runtime inference
output = run_onnx_inference(onnx_stream, sample_input)
```

Manual export with dynamic axes:

```python
torch.onnx.export(
    model, sample_input, "fno_model.onnx",
    opset_version=17,
    input_names=["input"], output_names=["output"],
    dynamic_axes={"input": {0: "batch"}, "output": {0: "batch"}},
)
```

### 7. Launch Utils Checkpointing

```python
from physicsnemo.launch.utils import load_checkpoint, save_checkpoint

# Save with optimizer and scheduler state
save_checkpoint("./checkpoints", models=model,
                optimizer=optimizer, scheduler=scheduler, epoch=epoch)

# Load (returns last saved epoch)
loaded_epoch = load_checkpoint("./checkpoints", models=model,
                               optimizer=optimizer, scheduler=scheduler, device="cuda")
```

## Checkpoint Format

PhysicsNeMo uses `.mdlus` files which contain:
- Model weights
- Architecture metadata (auto-reconstructs model)
- Training configuration
- Optimizer state (optional)

### 8. DoMINO NIM Inference (PhysicsNeMo-CFD)

Pre-trained AI surrogate for external aerodynamics via NVIDIA NIM:

```python
from physicsnemo.cfd.inference.domino_nim import call_domino_nim

# Run inference on STL geometry (e.g., DrivAerML car dataset)
output_dict = call_domino_nim(
    stl_path="./drivaer_202.stl",
    inference_api_url="http://localhost:8000/v1/infer",
    data={
        "stream_velocity": "38.89",
        "stencil_size": "1",
        "point_cloud_size": "500000",
    },
    verbose=True,
)
# Returns surface pressure, velocity fields on point cloud
```

Install PhysicsNeMo-CFD: see [PhysicsNeMo-CFD Installation Guide](https://docs.nvidia.com/physicsnemo/physicsnemo-cfd/).

### 9. Earth2Studio Weather Inference

AI weather prediction with pre-trained models (DLWP, FourCastNet, GraphCast):

```python
from earth2studio.models.px import DLWP
from earth2studio.data import GFS
from earth2studio.io import NetCDF4Backend
from earth2studio.run import deterministic as run

model = DLWP.load_model(DLWP.load_default_package())
ds = GFS()
io = NetCDF4Backend("output.nc")
run(["2024-01-01"], 10, model, ds, io)  # 10-day forecast
```

Install Earth2Studio: see [Earth2Studio docs](https://nvidia.github.io/earth2studio/).

### 10. NGC Pre-trained Checkpoints

Selected models available on [NGC Catalog](https://catalog.ngc.nvidia.com/):

```bash
# Download pre-trained checkpoints
ngc registry resource download-version nvidia/physicsnemo/<model>:latest
```

## Common Pitfalls

- **Forgetting `model.eval()`**: Without eval mode, BatchNorm/Dropout behave differently.
- **Missing `torch.inference_mode()`**: Disables gradient computation for 2x+ memory savings.
- **Not denormalizing outputs**: If training used normalized data, predictions need inverse transform.
- **Loading on wrong device**: Use `.to("cuda")` after `from_checkpoint()` for GPU inference.
- **ONNX operator support**: Some PhysicsNeMo layers may not export to ONNX. Test before deploying.
- **DoMINO NIM URL**: Default inference endpoint is `http://localhost:8000/v1/infer`. Ensure NIM container is running.

## Related Skills

- [training-recipes.md](./training-recipes.md) - Train model before deploying
- [models-and-architectures.md](./models-and-architectures.md) - Model architecture details
- [distributed-training.md](./distributed-training.md) - Save from distributed training
- [data-pipelines.md](./data-pipelines.md) - Prepare input data
