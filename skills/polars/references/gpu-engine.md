# Polars GPU Engine Reference

Sources:
- [Polars GPU Support](https://docs.pola.rs/user-guide/gpu-support/)
- [cuDF Polars Engine](https://docs.rapids.ai/api/cudf/stable/cudf_polars/)
- [RAPIDS Polars GPU Engine](https://rapids.ai/polars-gpu-engine/)

## Overview

The Polars GPU engine is an in-memory, GPU-accelerated execution engine powered by RAPIDS cuDF. It transparently accelerates compute-heavy operations while maintaining CPU fallback for unsupported features.

**Key Features**:
- Up to 13x speedup on compute-heavy queries (NVIDIA H100 benchmarks)
- Transparent CPU fallback for unsupported operations
- Single GPU implementation
- Open Beta status (actively developed)
- 99.2% Polars test suite compatibility with fallback
- 88.8% without fallback

## System Requirements

### Hardware

```bash
# Required
NVIDIA Volta™ or newer (compute capability 7.0+)
# Examples: V100, T4, A100, H100, RTX 2000+, GTX 1660+

# Check your GPU
nvidia-smi
```

### Software

```bash
# CUDA version
CUDA 12 (required)
# CUDA 11 supported until RAPIDS v25.06

# Operating system
Linux (native)
WSL2 (Windows Subsystem for Linux)
# macOS not supported (no NVIDIA GPU)
```

## Installation

### Standard Installation

```bash
# Install with GPU support
pip install polars[gpu]

# Verify installation
python -c "import polars as pl; print(pl.__version__)"
python -c "import cudf_polars; print('GPU engine available')"
```

### CUDA 11 Installation

```bash
# For RAPIDS v25.06 and earlier
pip install polars cudf-polars-cu11
```

### Docker

```dockerfile
# Use RAPIDS container
FROM rapidsai/base:25.02-cuda12.0-py3.11

# Install Polars
RUN pip install polars[gpu]

# Verify
RUN python -c "import polars as pl; import cudf_polars"
```

### Conda

```bash
# Create environment with RAPIDS
conda create -n polars-gpu -c rapidsai -c conda-forge \
    cudf polars python=3.11 cuda-version=12.0

conda activate polars-gpu
```

## Basic Usage

### CPU vs GPU Execution

```python
import polars as pl

# Create lazy query
lf = pl.scan_parquet("data.parquet").filter(
    pl.col("value") > 100
).group_by("category").agg(
    pl.col("value").sum()
)

# CPU execution
df_cpu = lf.collect()

# GPU execution
df_gpu = lf.collect(engine="gpu")

# Results are identical
assert df_cpu.equals(df_gpu)
```

### GPU Engine Object

```python
# Default GPU (device 0)
df = lf.collect(engine="gpu")

# Explicit GPU Engine
gpu_engine = pl.GPUEngine()
df = lf.collect(engine=gpu_engine)

# Specific GPU device
gpu_engine = pl.GPUEngine(device=1)
df = lf.collect(engine=gpu_engine)

# Raise on failure (no CPU fallback)
gpu_engine = pl.GPUEngine(raise_on_fail=True)
df = lf.collect(engine=gpu_engine)  # Raises if GPU can't execute
```

## Configuration

### Environment Variables

```bash
# Peak performance (recommended for production)
export POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=0

# Managed memory (safer, default)
export POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=1

# Maximum GPU memory usage (e.g., 16GB)
export CUDF_SPILL=on
export CUDF_SPILL_DEVICE_LIMIT=16GB
```

**Managed Memory:**
- **Enabled (1)**: Safer, automatic CPU-GPU memory movement, protects from OOM
- **Disabled (0)**: Faster, requires careful memory management

### Python Configuration

```python
import cudf

# Check current memory resource
print(cudf.get_option("memory_resource"))

# Set memory pool
from rmm.allocators.cupy import rmm_cupy_allocator
import cupy as cp

cp.cuda.set_allocator(rmm_cupy_allocator)
```

### Debugging Configuration

```python
import polars.config as cfg

# Enable verbose warnings
cfg.set_verbose(True)

# Query with warnings
lf = pl.scan_parquet("data.parquet")
result = lf.collect(engine="gpu")
# Outputs PerformanceWarning if CPU fallback occurs

# Raise on failure
engine = pl.GPUEngine(raise_on_fail=True)
result = lf.collect(engine=engine)
# Raises exception instead of fallback
```

## Supported Operations

### Data Types

✓ **Supported:**
- Numeric: Int8, Int16, Int32, Int64, UInt8, UInt16, UInt32, UInt64, Float32, Float64
- String: String (UTF-8)
- Boolean: Boolean
- Null: Null
- Datetime: Datetime (with limitations)
- List: List (nested operations limited)
- Struct: Struct (nested operations limited)

✗ **Not Supported:**
- Date
- Time
- Categorical
- Enum
- Array (fixed-size)
- Binary
- Duration

### Expressions

✓ **Supported:**
- Arithmetic: +, -, *, /, %, //
- Comparisons: ==, !=, <, >, <=, >=
- Logical: &, |, ~, is_null, is_not_null
- String: contains, starts_with, ends_with, to_lowercase, to_uppercase
- Aggregations: sum, mean, min, max, count, first, last, n_unique
- Window functions: over, rolling_*
- Conditional: when, then, otherwise

✗ **Not Supported:**
- User-defined functions (UDFs)
- Some advanced string operations
- Time series resampling
- Regex-based operations (limited)

### I/O Operations

✓ **Supported:**
- CSV: read and write
- Parquet: read and write
- NDJSON: read and write

✗ **Not Supported:**
- Streaming execution with GPU
- Excel
- Database connectors
- Cloud storage direct read (must download first)

### DataFrame Operations

✓ **Supported:**
- Filter
- Select
- With_columns
- Group_by + agg
- Join (inner, left, outer)
- Sort
- Slice
- Explode
- Pivot (limited)
- Melt

✗ **Not Supported:**
- Eager DataFrame API (use LazyFrame)
- Streaming
- Some complex window operations

## Performance Characteristics

### Best Performance Gains

**Grouped Aggregations:**

```python
# CPU: ~2.5s, GPU: ~0.2s (12x faster)
result = (
    pl.scan_parquet("sales.parquet")  # 100M rows
    .group_by(["region", "product"])
    .agg([
        pl.col("revenue").sum(),
        pl.col("quantity").mean(),
        pl.col("transaction_id").count()
    ])
    .collect(engine="gpu")
)
```

**Complex Joins:**

```python
# CPU: ~5s, GPU: ~0.5s (10x faster)
result = (
    pl.scan_parquet("orders.parquet")  # 50M rows
    .join(
        pl.scan_parquet("customers.parquet"),  # 10M rows
        on="customer_id",
        how="left"
    )
    .group_by("customer_segment")
    .agg(pl.col("order_value").sum())
    .collect(engine="gpu")
)
```

**Rolling Aggregations:**

```python
# CPU: ~3s, GPU: ~0.3s (10x faster)
result = (
    pl.scan_parquet("timeseries.parquet")
    .with_columns(
        pl.col("value").rolling_mean(window_size=100).alias("ma_100")
    )
    .collect(engine="gpu")
)
```

### Similar CPU/GPU Performance

**I/O-Bound Operations:**

```python
# Bottleneck is disk I/O, not compute
result = pl.scan_parquet("data.parquet").collect(engine="gpu")
# GPU provides minimal benefit
```

**Simple Filters:**

```python
# Lightweight operation, GPU overhead > benefit
result = lf.filter(pl.col("id") > 1000).collect(engine="gpu")
```

**Small Datasets:**

```python
# < 1M rows: GPU initialization overhead dominates
lf = pl.scan_csv("small.csv")  # 10K rows
result = lf.collect(engine="gpu")  # Slower than CPU
```

### Memory Limits

**Dataset Size Guidelines:**

```python
# Rule of thumb: dataset < 60-70% of GPU memory
# 80GB GPU → ~50-100 GiB dataset max

# Check GPU memory
import cudf
import cupy as cp

free, total = cp.cuda.Device(0).mem_info
print(f"Free: {free / 1e9:.1f}GB, Total: {total / 1e9:.1f}GB")

# Monitor during execution
result = lf.collect(engine="gpu")
```

**Memory Spilling:**

```bash
# Enable spilling to host memory
export CUDF_SPILL=on
export CUDF_SPILL_DEVICE_LIMIT=16GB  # Spill when >16GB used
```

## Fallback Behavior

### Automatic Fallback

```python
# Unsupported operations fall back to CPU automatically
lf = pl.scan_parquet("data.parquet")

result = (
    lf
    .filter(pl.col("value") > 100)  # GPU
    .with_columns(
        pl.col("date").dt.year()  # CPU fallback (Date type)
    )
    .group_by("year")  # GPU
    .agg(pl.col("value").sum())  # GPU
    .collect(engine="gpu")
)

# Result is correct, some ops ran on CPU
```

### Detecting Fallback

```python
import polars.config as cfg
import warnings

# Enable warnings
cfg.set_verbose(True)
warnings.simplefilter("always", PerformanceWarning)

# Run query
result = lf.collect(engine="gpu")

# Will show warnings like:
# PerformanceWarning: Query execution with GPU not possible, falling back to CPU
```

### Preventing Fallback

```python
# Raise exception on fallback
engine = pl.GPUEngine(raise_on_fail=True)

try:
    result = lf.collect(engine=engine)
except Exception as e:
    print(f"GPU execution failed: {e}")
    # Handle error or use CPU
    result = lf.collect()
```

## Optimization Tips

### 1. Disable Managed Memory

```bash
# Set before running Python
export POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=0
```

```python
# Verify setting
import os
print(os.getenv("POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY"))
# Should print '0'

# Run query
result = lf.collect(engine="gpu")
```

**Benchmark impact (cuDF Polars benchmarks):**
- Managed memory enabled: 6.2s
- Managed memory disabled: 2.8s (2.2x faster)

### 2. Use Appropriate Types

```python
# Avoid unsupported types
lf = pl.scan_csv("data.csv", dtypes={
    "date_col": pl.Datetime,  # ✓ Supported
    # "date_col": pl.Date,    # ✗ Not supported
    "category": pl.String,     # ✓ Supported (fallback to CPU for categorical)
})

result = lf.collect(engine="gpu")
```

### 3. Batch Large Datasets

```python
# Process in date partitions
dates = pl.date_range(
    start="2024-01-01",
    end="2024-12-31",
    interval="1d"
)

results = []
for date in dates:
    lf = pl.scan_parquet(f"data/{date}/*.parquet")
    result = lf.group_by("category").agg(
        pl.col("value").sum()
    ).collect(engine="gpu")
    results.append(result)

combined = pl.concat(results)
```

### 4. Profile Before GPU

```python
import time

# Profile CPU execution
start = time.time()
cpu_result = lf.collect()
cpu_time = time.time() - start

# Profile GPU execution
start = time.time()
gpu_result = lf.collect(engine="gpu")
gpu_time = time.time() - start

print(f"CPU: {cpu_time:.2f}s")
print(f"GPU: {gpu_time:.2f}s")
print(f"Speedup: {cpu_time / gpu_time:.2f}x")

# Only use GPU if speedup > 1.5x (to account for overhead)
```

### 5. Optimize Query Plan

```python
# Ensure predicate/projection pushdown
lf = pl.scan_parquet("data.parquet")

query = (
    lf
    .filter(pl.col("date") >= "2024-01-01")  # Filter early
    .select(["id", "value", "category"])      # Project early
    .group_by("category")
    .agg(pl.col("value").sum())
)

# Check optimized plan
print(query.explain(optimized=True))

# Execute on GPU
result = query.collect(engine="gpu")
```

## Multi-GPU Support

Currently, Polars GPU engine targets single GPU execution.

**Workaround for Multiple GPUs:**

```python
import os
from concurrent.futures import ProcessPoolExecutor

def process_partition(device_id, partition_path):
    # Set GPU device
    os.environ["CUDA_VISIBLE_DEVICES"] = str(device_id)

    import polars as pl

    lf = pl.scan_parquet(partition_path)
    result = lf.group_by("category").agg(
        pl.col("value").sum()
    ).collect(engine="gpu")

    return result

# Partition data across GPUs
partitions = [
    "data/part_0.parquet",
    "data/part_1.parquet",
    "data/part_2.parquet",
    "data/part_3.parquet"
]

with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(
        process_partition,
        [0, 1, 2, 3],  # GPU device IDs
        partitions
    ))

# Combine results
combined = pl.concat(results)
```

## Troubleshooting

### Out of Memory Errors

```python
# Enable spilling
import os
os.environ["CUDF_SPILL"] = "on"
os.environ["CUDF_SPILL_DEVICE_LIMIT"] = "16GB"

# Or reduce dataset size
lf = pl.scan_parquet("data.parquet")
result = lf.filter(pl.col("date") == "2024-01-01").collect(engine="gpu")
```

### Slow GPU Performance

```bash
# Check managed memory setting
echo $POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY
# Should be '0' for best performance

# Disable if needed
export POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=0
```

### GPU Not Detected

```python
# Verify CUDA installation
import cupy as cp
print(cp.cuda.runtime.getDeviceCount())  # Should be > 0

# Verify cudf-polars
import cudf_polars
print("GPU engine available")

# Check CUDA_VISIBLE_DEVICES
import os
print(os.getenv("CUDA_VISIBLE_DEVICES"))
```

### Unsupported Operation

```python
# Enable verbose warnings to identify issue
import polars.config as cfg
cfg.set_verbose(True)

result = lf.collect(engine="gpu")
# Check warnings for unsupported operations

# Or raise on fallback
engine = pl.GPUEngine(raise_on_fail=True)
try:
    result = lf.collect(engine=engine)
except Exception as e:
    print(f"Unsupported operation: {e}")
```

## Benchmarks

### Polars Decision Support (PDS) Benchmark

NVIDIA H100 PCIe vs Intel Xeon (official benchmarks):

```
Query   CPU Time   GPU Time   Speedup
Q1      2.45s      0.19s      12.9x
Q3      3.12s      0.31s      10.1x
Q5      5.87s      0.45s      13.0x
Q6      1.89s      0.21s      9.0x
Q9      4.23s      0.38s      11.1x

Average speedup: 11.2x
```

**Best speedups:**
- Grouped aggregations: 10-13x
- Complex joins: 8-12x
- Rolling operations: 9-11x

**Minimal speedups:**
- Simple filters: 1-2x
- I/O-heavy: 1.1-1.5x
- Small datasets (< 1M rows): 0.8-1.2x (GPU overhead)

## Best Practices

1. **Profile First**: Measure CPU vs GPU performance on your data
2. **Disable Managed Memory**: Set `POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=0`
3. **Use LazyFrame**: GPU engine requires lazy evaluation
4. **Appropriate Types**: Avoid unsupported types (Date, Categorical, Enum)
5. **Batch Large Data**: Process in chunks if > 60% GPU memory
6. **Filter Early**: Reduce data before GPU execution
7. **Monitor Fallback**: Enable verbose warnings to detect CPU fallback
8. **GPU for Heavy Compute**: Use GPU for aggregations, joins, rolling ops
9. **CPU for Simple Ops**: Don't use GPU for trivial filters or small data
10. **Test on Subset**: Validate query on small subset before full dataset

## Common Pitfalls

- **Using eager API**: GPU engine requires LazyFrame
- **Managed memory enabled**: Significantly slower, disable for production
- **Unsupported types**: Date, Categorical, Enum cause CPU fallback
- **Too-small datasets**: GPU overhead > benefit for < 1M rows
- **Memory overload**: Dataset > 70% GPU memory causes OOM
- **Not checking fallback**: Silently falls back to CPU, losing performance
- **Assuming all ops accelerate**: I/O-bound ops see minimal benefit
- **UDFs in pipeline**: User-defined functions not supported on GPU
