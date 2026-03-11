# Dask GPU Acceleration Reference

Sources:
- [Dask-CUDA Documentation](https://docs.rapids.ai/api/dask-cuda/stable/)
- [cuDF Documentation](https://docs.rapids.ai/api/cudf/stable/)
- [RAPIDS Documentation](https://docs.rapids.ai)
- [Dask GPU Best Practices](https://docs.dask.org/en/stable/gpu.html)

## Overview

Dask supports GPU acceleration through integration with NVIDIA RAPIDS libraries:

- **cuDF**: GPU-accelerated DataFrame (pandas-like API)
- **CuPy**: GPU-accelerated arrays (NumPy-like API)
- **cuML**: GPU-accelerated machine learning
- **Dask-CUDA**: GPU cluster management

## Installation

```bash
# Install RAPIDS (includes cuDF, CuPy, cuML, Dask-CUDA)
conda create -n rapids-env -c rapidsai -c conda-forge -c nvidia \
    rapids=24.10 python=3.10 cuda-version=12.0

conda activate rapids-env

# Or individual packages
conda install -c rapidsai -c conda-forge -c nvidia \
    cudf dask-cudf cupy dask-cuda
```

### Requirements

- NVIDIA GPU (Pascal architecture or newer)
- CUDA Toolkit 11.2+ or 12.0+
- Linux operating system (recommended)

## Dask-cuDF (GPU DataFrames)

### Basic Usage

```python
import dask_cudf

# Read data on GPU
df = dask_cudf.read_parquet('data/*.parquet')

# Operations use GPU
result = df.groupby('key').value.mean().compute()

# Result is cuDF DataFrame (GPU)
type(result)  # cudf.DataFrame
```

### Creating GPU DataFrames

```python
import dask_cudf
import cudf

# From pandas
import pandas as pd
pdf = pd.DataFrame({'x': [1, 2, 3], 'y': [4, 5, 6]})
gdf = dask_cudf.from_cudf(cudf.from_pandas(pdf), npartitions=2)

# From parquet
gdf = dask_cudf.read_parquet('data/*.parquet')

# From CSV
gdf = dask_cudf.read_csv('data/*.csv')

# From Dask DataFrame
import dask.dataframe as dd
df = dd.read_parquet('data.parquet')
gdf = df.map_partitions(cudf.from_pandas)
```

### Operations

```python
# All pandas-like operations on GPU
gdf['new_col'] = gdf.x + gdf.y
filtered = gdf[gdf.value > 10]
grouped = gdf.groupby('category').agg({'value': ['sum', 'mean']})

# Compute returns cuDF DataFrame
result = grouped.compute()

# Convert to pandas
pdf = result.to_pandas()
```

### Performance Tips

```python
# Use GPU-native formats
gdf = dask_cudf.read_parquet('data.parquet')  # Fast
gdf = dask_cudf.read_csv('data.csv')  # Slower, parsing on GPU

# Avoid transfers between CPU/GPU
# Bad:
df = dd.read_parquet('data.parquet')  # CPU
df = df.map_partitions(cudf.from_pandas)  # CPU → GPU transfer

# Good:
gdf = dask_cudf.read_parquet('data.parquet')  # GPU directly
```

## CuPy Arrays

### Basic Usage

```python
import dask.array as da
import cupy as cp

# Create CuPy array
gpu_array = cp.random.random((10000, 10000), dtype='float32')

# Wrap in Dask Array
x = da.from_array(gpu_array, chunks=(1000, 1000))

# Operations stay on GPU
y = x @ x.T
result = y.mean().compute()

# Result is CuPy array (GPU)
type(result)  # cupy.ndarray
```

### Random Number Generation

```python
import dask.array as da
import cupy as cp

# CuPy random state
rs = cp.random.RandomState(42)

# Create random GPU array
x = da.from_array(
    rs.random((10000, 10000), dtype='float32'),
    chunks=(1000, 1000)
)
```

### Matrix Operations

```python
# Matrix multiplication
y = x @ x.T

# SVD (on GPU)
u, s, v = da.linalg.svd(x)

# Element-wise operations
y = da.exp(x)
z = da.log(x + 1)
```

### FFT on GPU

```python
# FFT using CuPy backend
y = da.fft.fft(x)
y = da.fft.fft2(x)
```

## LocalCUDACluster

### Basic Setup

```python
from dask_cuda import LocalCUDACluster
from dask.distributed import Client

# Auto-detect GPUs
cluster = LocalCUDACluster()
client = Client(cluster)

# Check cluster
print(client)
# Shows workers, each with dedicated GPU
```

### Configuration

```python
cluster = LocalCUDACluster(
    n_workers=4,              # 4 GPUs
    threads_per_worker=1,     # 1 thread per GPU
    device_memory_limit='16GB',  # GPU memory limit
    memory_limit='32GB',      # CPU memory limit
    local_directory='/tmp',   # Spill directory
    dashboard_address=':8787'
)

client = Client(cluster)
```

### Multi-GPU Strategy

```python
# One worker per GPU (recommended)
cluster = LocalCUDACluster(
    CUDA_VISIBLE_DEVICES='0,1,2,3',  # Specify GPUs
    n_workers=4
)

# Each worker gets dedicated GPU
# - Worker 0: GPU 0
# - Worker 1: GPU 1
# - Worker 2: GPU 2
# - Worker 3: GPU 3
```

### Environment Variables

```python
import os

# Set before creating cluster
os.environ['CUDA_VISIBLE_DEVICES'] = '0,1,2,3'
os.environ['CUDA_DEVICE_ORDER'] = 'PCI_BUS_ID'

cluster = LocalCUDACluster()
```

## GPU Memory Management

### Device Memory Limit

```python
# Limit GPU memory per worker
cluster = LocalCUDACluster(device_memory_limit='16GB')

# Auto (80% of available GPU memory)
cluster = LocalCUDACluster(device_memory_limit='auto')

# Or set via environment
os.environ['DASK_CUDA__DEVICE_MEMORY_LIMIT'] = '16GB'
```

### Spilling

```python
# Enable GPU → CPU spilling
cluster = LocalCUDACluster(
    device_memory_limit='16GB',
    memory_limit='64GB',  # CPU spill buffer
    local_directory='/scratch'  # Disk spill location
)
```

### Manual Memory Management

```python
import cupy as cp
import cudf

# Explicitly free GPU memory
del gpu_dataframe
cp.get_default_memory_pool().free_all_blocks()

# Clear Dask cache
client.cancel(future)
client.restart()  # Nuclear option
```

## UCX Communication

UCX (Unified Communication X) enables high-speed GPU-to-GPU communication.

### Enable UCX

```python
from dask_cuda import LocalCUDACluster

cluster = LocalCUDACluster(
    protocol='ucx',           # Enable UCX
    enable_tcp_over_ucx=True, # Fallback to TCP
    enable_nvlink=True,       # Use NVLink if available
    enable_infiniband=True    # Use InfiniBand if available
)
```

### Requirements

- UCX library installed
- CUDA-aware UCX build
- NVLink or InfiniBand for best performance

```bash
# Install UCX
conda install -c conda-forge ucx-proc=*=gpu ucx ucx-py
```

## Multi-GPU Workflows

### Data Parallel Processing

```python
from dask_cuda import LocalCUDACluster
from dask.distributed import Client
import dask_cudf

# Setup cluster
cluster = LocalCUDACluster()
client = Client(cluster)

# Load data on all GPUs
gdf = dask_cudf.read_parquet('data/*.parquet')

# Parallel operations across GPUs
result = gdf.groupby('key').value.agg(['sum', 'mean', 'std']).compute()
```

### Model Parallel Training

```python
import cupy as cp
from dask_cuda import LocalCUDACluster
from dask.distributed import Client

cluster = LocalCUDACluster(n_workers=4)
client = Client(cluster)

# Distribute model layers across GPUs
def train_layer(layer_data, gpu_id):
    cp.cuda.Device(gpu_id).use()
    # Training code here
    return trained_weights

futures = [client.submit(train_layer, data, i) for i in range(4)]
results = client.gather(futures)
```

## cuML Integration

### GPU Machine Learning

```python
from dask_cuda import LocalCUDACluster
from dask.distributed import Client
import dask_cudf
from cuml.dask.ensemble import RandomForestClassifier

# Setup
cluster = LocalCUDACluster()
client = Client(cluster)

# Load data
train = dask_cudf.read_parquet('train.parquet')
X = train[['feature1', 'feature2', 'feature3']]
y = train['target']

# Train on GPUs
rf = RandomForestClassifier(n_estimators=100)
rf.fit(X, y)

# Predict
test = dask_cudf.read_parquet('test.parquet')
predictions = rf.predict(test[['feature1', 'feature2', 'feature3']]).compute()
```

### XGBoost GPU

```python
import xgboost as xgb
import dask_cudf
from dask_cuda import LocalCUDACluster
from dask.distributed import Client

cluster = LocalCUDACluster()
client = Client(cluster)

# Load data on GPU
df = dask_cudf.read_parquet('data.parquet')
X = df[features]
y = df['target']

# Train with GPU
dtrain = xgb.dask.DaskDeviceQuantileDMatrix(client, X, y)
params = {
    'tree_method': 'gpu_hist',  # GPU training
    'max_depth': 5,
    'objective': 'reg:squarederror'
}
model = xgb.dask.train(client, params, dtrain, num_boost_round=100)
```

## Monitoring GPU Usage

### Dashboard

```python
client = Client(cluster)
print(client.dashboard_link)  # View at http://localhost:8787/status

# Dashboard shows:
# - GPU utilization per worker
# - GPU memory usage
# - Task assignment to GPUs
```

### Programmatic Monitoring

```python
import cupy as cp

# GPU memory info
mempool = cp.get_default_memory_pool()
print(f"Used: {mempool.used_bytes() / 1e9:.2f} GB")
print(f"Total: {mempool.total_bytes() / 1e9:.2f} GB")

# CUDA device properties
device = cp.cuda.Device()
print(f"Name: {device.attributes['Name']}")
print(f"Compute Capability: {device.compute_capability}")
```

### nvidia-smi Integration

```bash
# Monitor GPUs while running
watch -n 1 nvidia-smi

# Or within Python
import subprocess
result = subprocess.run(['nvidia-smi'], capture_output=True, text=True)
print(result.stdout)
```

## Best Practices

### 1. Data Transfer Minimization

```python
# Bad: Multiple CPU ↔ GPU transfers
df = dd.read_parquet('data.parquet')  # CPU
gdf = df.map_partitions(cudf.from_pandas)  # CPU → GPU
result = gdf.compute()  # GPU → CPU

# Good: Stay on GPU
gdf = dask_cudf.read_parquet('data.parquet')  # GPU
result = gdf.compute()  # GPU
pdf = result.to_pandas()  # GPU → CPU only at end
```

### 2. Chunk Size

```python
# GPU memory is limited
# Use smaller chunks than CPU

# CPU: 100MB-1GB chunks
df = dd.read_parquet('data.parquet', blocksize='500MB')

# GPU: 50MB-500MB chunks
gdf = dask_cudf.read_parquet('data.parquet', blocksize='200MB')
```

### 3. Memory Limits

```python
# Set conservative limits
cluster = LocalCUDACluster(
    device_memory_limit=0.7,  # 70% of GPU memory
    memory_limit='auto'        # Generous CPU for spilling
)
```

### 4. UCX for Multi-GPU

```python
# Enable UCX for multi-node GPU clusters
cluster = LocalCUDACluster(
    protocol='ucx',
    enable_tcp_over_ucx=True,
    enable_nvlink=True
)
```

### 5. One Worker Per GPU

```python
# Each GPU gets dedicated worker
cluster = LocalCUDACluster(
    n_workers=4,  # 4 GPUs
    threads_per_worker=1  # No thread contention
)
```

## Common Pitfalls

- **Not setting CUDA_VISIBLE_DEVICES**: Workers share GPUs
- **Too large chunks**: GPU out-of-memory errors
- **Unnecessary CPU ↔ GPU transfers**: Performance bottleneck
- **Multiple threads per GPU**: Context switching overhead
- **Not enabling UCX**: Slow multi-GPU communication
- **Ignoring memory limits**: Worker crashes from OOM
- **Not monitoring dashboard**: Miss GPU utilization issues
- **Mixed CPU/GPU operations**: Forces transfers
- **Not using GPU-native formats**: Parquet better than CSV
- **Forgetting to free memory**: GPU memory leaks

## Troubleshooting

### Out of Memory

```python
# Reduce chunk size
gdf = dask_cudf.read_parquet('data.parquet', blocksize='100MB')

# Lower memory limit
cluster = LocalCUDACluster(device_memory_limit='12GB')

# Enable spilling
cluster = LocalCUDACluster(
    device_memory_limit='12GB',
    memory_limit='64GB'
)
```

### Slow Performance

```python
# Check if operations are GPU-accelerated
# Look for cudf/cupy in task names in dashboard

# Verify GPU usage
nvidia-smi

# Profile with CUDA profiler
import cupy as cp
with cp.prof.time_range('my_operation'):
    result = gdf.groupby('key').sum().compute()
```

### Worker Crashes

```bash
# Check CUDA errors in worker logs
dask worker scheduler:8786 --log-level DEBUG

# Verify CUDA installation
python -c "import cupy; print(cupy.cuda.runtime.getDeviceCount())"
```

### UCX Issues

```bash
# Verify UCX installation
python -c "import ucp; print(ucp.get_ucx_version())"

# Disable UCX if problematic
cluster = LocalCUDACluster(protocol='tcp')
```

## Performance Comparison

| Operation | CPU (Dask) | GPU (Dask-CUDA) | Speedup |
|-----------|------------|-----------------|---------|
| DataFrame groupby | 10s | 0.5s | 20x |
| Matrix multiply (10k×10k) | 30s | 0.3s | 100x |
| XGBoost training | 120s | 8s | 15x |
| Data loading (Parquet) | 5s | 1s | 5x |

*Approximate speedups, varies by hardware and workload*

## Example: End-to-End GPU Workflow

```python
from dask_cuda import LocalCUDACluster
from dask.distributed import Client
import dask_cudf
from cuml.dask.ensemble import RandomForestRegressor

# Setup GPU cluster
cluster = LocalCUDACluster(
    n_workers=4,
    threads_per_worker=1,
    device_memory_limit='16GB',
    protocol='ucx',
    enable_nvlink=True
)
client = Client(cluster)

# Load data on GPU
train = dask_cudf.read_parquet('train/*.parquet')
test = dask_cudf.read_parquet('test/*.parquet')

# Feature engineering on GPU
train['feature_ratio'] = train.feature1 / (train.feature2 + 1e-6)
train = train.fillna(0)

# Train model on GPU
X = train[['feature1', 'feature2', 'feature_ratio']]
y = train['target']

rf = RandomForestRegressor(n_estimators=100, max_depth=10)
rf.fit(X, y)

# Predict on GPU
test['feature_ratio'] = test.feature1 / (test.feature2 + 1e-6)
predictions = rf.predict(test[['feature1', 'feature2', 'feature_ratio']])

# Save results (GPU → disk)
output = test[['id']].assign(prediction=predictions)
output.to_parquet('predictions/')

# Cleanup
client.close()
cluster.close()
```
