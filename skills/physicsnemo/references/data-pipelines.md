---
title: Data Pipelines
impact: HIGH
tags: datapipe, darcy, normalization, scientific-data, hdf5, netcdf
---

# Skill: Data Pipelines

Load and preprocess scientific simulation data for PhysicsNeMo training.

## Quick Pattern

```python
from physicsnemo.datapipes.benchmarks.darcy import Darcy2D

normaliser = {
    "permeability": (1.25, 0.75),  # (mean, std)
    "darcy": (4.52e-2, 2.79e-2),
}

dataloader = Darcy2D(
    resolution=256, batch_size=64,
    nr_permeability_freq=5, normaliser=normaliser
)

batch = next(iter(dataloader))
print(batch.keys())  # dict_keys(['permeability', 'darcy'])
```

## When to Use

- Loading benchmark physics datasets (Darcy, Navier-Stokes)
- Setting up data normalization for scientific data
- Creating custom datapipes for domain-specific data
- Working with HDF5, NetCDF, or NumPy data formats

## Step-by-Step Instructions

### 1. Built-in Benchmark Datapipes

```python
from physicsnemo.datapipes.benchmarks.darcy import Darcy2D

# Darcy flow benchmark
normaliser = {
    "permeability": (1.25, 0.75),
    "darcy": (4.52e-2, 2.79e-2),
}

dataloader = Darcy2D(
    resolution=256,
    batch_size=64,
    nr_permeability_freq=5,
    normaliser=normaliser,
)

batch = next(iter(dataloader))
# batch["permeability"]: [B, 1, 256, 256] - input field
# batch["darcy"]:        [B, 1, 256, 256] - target solution
```

### 2. Custom Data from HDF5

```python
import h5py
import torch
from torch.utils.data import Dataset, DataLoader

class HDF5PhysicsDataset(Dataset):
    def __init__(self, filepath, input_key, output_key, normaliser=None):
        self.file = h5py.File(filepath, 'r')
        self.inputs = self.file[input_key]
        self.outputs = self.file[output_key]
        self.normaliser = normaliser or {}

    def __len__(self):
        return self.inputs.shape[0]

    def __getitem__(self, idx):
        x = torch.tensor(self.inputs[idx], dtype=torch.float32)
        y = torch.tensor(self.outputs[idx], dtype=torch.float32)

        # Normalize
        if "input" in self.normaliser:
            mean, std = self.normaliser["input"]
            x = (x - mean) / std
        if "output" in self.normaliser:
            mean, std = self.normaliser["output"]
            y = (y - mean) / std

        return {"input": x.unsqueeze(0), "output": y.unsqueeze(0)}

dataset = HDF5PhysicsDataset(
    "simulation_data.h5",
    input_key="velocity_field",
    output_key="pressure_field",
    normaliser={"input": (0.0, 1.0), "output": (0.0, 0.01)},
)

dataloader = DataLoader(dataset, batch_size=32, shuffle=True, num_workers=4)
```

### 3. Custom Data from NumPy

```python
import numpy as np
import torch
from torch.utils.data import TensorDataset, DataLoader

# Load simulation data
inputs = np.load("inputs.npy")   # [N, H, W]
outputs = np.load("outputs.npy") # [N, H, W]

# Normalize
in_mean, in_std = inputs.mean(), inputs.std()
out_mean, out_std = outputs.mean(), outputs.std()
inputs = (inputs - in_mean) / in_std
outputs = (outputs - out_mean) / out_std

# Convert to tensors with channel dim
inputs_t = torch.tensor(inputs, dtype=torch.float32).unsqueeze(1)   # [N, 1, H, W]
outputs_t = torch.tensor(outputs, dtype=torch.float32).unsqueeze(1)  # [N, 1, H, W]

dataset = TensorDataset(inputs_t, outputs_t)
dataloader = DataLoader(dataset, batch_size=32, shuffle=True)
```

### 4. Normalization Strategies

| Strategy | When to Use | Pattern |
|----------|-------------|---------|
| Per-variable (mean, std) | Default for most fields | `(x - mean) / std` |
| Min-max [0, 1] | Bounded variables | `(x - min) / (max - min)` |
| Log transform | Highly skewed data | `log(x + epsilon)` then normalize |
| Per-sample | Varying magnitudes | Normalize each sample independently |

```python
# Compute normalization from training data
normaliser = {}
for key in ['temperature', 'pressure', 'velocity_x', 'velocity_y']:
    data = training_data[key]
    normaliser[key] = (data.mean().item(), data.std().item())
```

### 5. Weather Data Loading (ERA5, HRRR)

For weather/climate applications:

```bash
# Download CorrDiff-Mini dataset from NGC
ngc registry resource download-version nvidia/physicsnemo/corrdiff-mini-dataset:latest
```

Weather data typically uses:
- **Format**: NetCDF4, Zarr, or HDF5
- **Grid**: Regular lat/lon (e.g., 0.25deg = 721x1440)
- **Variables**: Temperature, pressure, wind, humidity at multiple levels
- **Time**: Sequential snapshots for autoregressive training

### 6. PhysicsNeMo Curator — ETL Pipeline

GPU-accelerated data curation for engineering datasets (`physicsnemo-curator`):

```bash
# Install from source (or use PhysicsNeMo Docker container)
git clone https://github.com/NVIDIA/physicsnemo-curator.git
cd physicsnemo_curator && pip install -e "."
```

Core components:

| Component | Import | Purpose |
|-----------|--------|---------|
| DataSource | `physicsnemo_curator.etl.data_sources.DataSource` | Read/write data files |
| DataTransformation | `physicsnemo_curator.etl.data_transformations.DataTransformation` | Transform data formats |
| DatasetValidator | `physicsnemo_curator.etl.dataset_validator` | Validate structure/content |
| ParallelProcessor | `physicsnemo_curator.etl.parallel_processor` | Orchestrate parallel processing |
| ProcessingConfig | `physicsnemo_curator.etl.processing_config.ProcessingConfig` | Pipeline configuration |

Example: HDF5 → Zarr conversion pipeline:

```python
from physicsnemo_curator.etl.data_sources import DataSource
from physicsnemo_curator.etl.data_transformations import DataTransformation
from physicsnemo_curator.etl.processing_config import ProcessingConfig

class H5DataSource(DataSource):
    def get_file_list(self): return sorted(Path(self.input_dir).glob("*.h5"))
    def read_file(self, filename):
        with h5py.File(f"{filename}.h5", 'r') as f:
            return {'temperature': np.array(f['fields/temperature']),
                    'velocity': np.array(f['fields/velocity'])}

class DataTypeTransform(DataTransformation):
    def transform(self, data):
        return {k: {'data': v.astype(np.float32), 'compressor': Blosc(),
                     'dtype': np.float32} for k, v in data.items()}
```

YAML pipeline config:

```yaml
etl:
  processing:
    num_processes: 2
  source:
    _target_: h5_data_source.H5DataSource
  transformations:
    data_type:
      _target_: data_transformations.DataTypeTransformation
  sink:
    _target_: zarr_data_source.ZarrDataSource
    output_dir: output_zarr
```

Run with: `physicsnemo-curator-etl --config-dir . --config-name pipeline_config`

Supported formats: **HDF5, VTK, STL, Zarr, NetCDF4**

### 7. ERA5 Data Download and Conversion

For weather/climate model training:

```bash
# Download ERA5 data via Climate Data Store (CDS) API
# See examples/weather/era5_data_downloader for tools
pip install cdsapi

# Convert to ML-ready format (NetCDF4, Zarr, HDF5)
# Grid: Regular lat/lon (0.25deg = 721x1440)
# Variables: temperature, pressure, wind, humidity at multiple levels
```

## Common Pitfalls

- **Missing normalization**: Raw physics data spans orders of magnitude. Always normalize.
- **Wrong tensor shape**: PhysicsNeMo expects `[B, C, H, W]` (channel-first). Transpose if needed.
- **Data on CPU during training**: Use `.to("cuda")` or `pin_memory=True` + `num_workers>0`.
- **Inconsistent dtypes**: Use `float32` unless model requires `float64`.
- **Not splitting train/val**: Always hold out validation data. Physics problems can overfit.
- **Curator PYTHONPATH**: Add directory containing custom DataSource/DataTransformation modules to `PYTHONPATH` before running `physicsnemo-curator-etl`.

## Related Skills

- [training-recipes.md](./training-recipes.md) - Use loaded data in training
- [models-and-architectures.md](./models-and-architectures.md) - Match in_channels to data
- [distributed-training.md](./distributed-training.md) - DistributedSampler for multi-GPU
- [advanced-features.md](./advanced-features.md) - Metrics, profiling
