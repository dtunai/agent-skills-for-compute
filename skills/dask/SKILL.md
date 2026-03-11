---
name: dask
description: Provides Dask parallel computing framework for scaling Python workloads across cores and clusters. Covers DataFrames, Arrays, Bags, Delayed, distributed scheduler, GPU acceleration with cuDF/CuPy, and HPC deployment with Dask-Jobqueue.
license: MIT
metadata:
  author: Agent Cluster
  tags: dask, parallel-computing, distributed-computing, dataframes, gpu-acceleration, hpc, python, pandas, numpy, cudf, rapids
---

# Dask Agent Skill

Agent-optimized skill for Dask parallel computing framework.

## Quick Reference

### Core Collections

```python
import dask.dataframe as dd
import dask.array as da
import dask.bag as db
from dask import delayed

# DataFrame - parallel pandas
df = dd.read_parquet('data/*.parquet')
result = df.groupby('key').value.mean().compute()

# Array - parallel NumPy
x = da.from_zarr('data.zarr')
y = (x + x.T).mean(axis=0).compute()

# Bag - parallel lists
b = db.read_text('logs/*.txt')
counts = b.map(str.split).flatten().frequencies().compute()

# Delayed - custom parallelism
@delayed
def process(x):
    return x * 2

results = [process(i) for i in range(10)]
total = delayed(sum)(results).compute()
```

### Distributed Scheduler

```python
from dask.distributed import Client, LocalCluster

# Local cluster (automatic)
client = Client()

# Local cluster (manual configuration)
cluster = LocalCluster(n_workers=4, threads_per_worker=2, memory_limit='4GB')
client = Client(cluster)

# Remote cluster
client = Client('scheduler-address:8786')

# Persist data in distributed memory
df = df.persist()

# Monitor progress
client.dashboard_link  # http://localhost:8787/status
```

### GPU Acceleration

```python
import dask_cudf
import cupy as cp

# GPU DataFrame (cuDF)
df = dask_cudf.read_parquet('data/*.parquet')
result = df.groupby('key').value.mean().compute()

# GPU Array (CuPy)
x = da.from_array(cp.random.random((10000, 10000), dtype='float32'), chunks=(1000, 1000))
y = x @ x.T
result = y.compute()

# GPU cluster
from dask_cuda import LocalCUDACluster
cluster = LocalCUDACluster()
client = Client(cluster)
```

### HPC Deployment

```python
from dask_jobqueue import SLURMCluster

# SLURM cluster
cluster = SLURMCluster(
    cores=24,
    processes=4,
    memory="100GB",
    walltime="02:00:00",
    queue="gpu",
    job_extra_directives=["--gres=gpu:2"]
)

cluster.scale(jobs=10)  # Submit 10 jobs
client = Client(cluster)
```

## Common Patterns

### DataFrame Operations

```python
import dask.dataframe as dd

# Read multiple files
df = dd.read_csv('data/*.csv')
df = dd.read_parquet('data/*.parquet')
df = dd.read_json('data/*.json')

# Compute lazily, execute with .compute()
result = df.groupby('category').price.mean()  # Lazy
result.compute()  # Execute

# Persist in memory for reuse
df = df.persist()

# Write results
df.to_parquet('output/', compression='snappy')
df.to_csv('output/*.csv')

# Set index for efficient queries
df = df.set_index('timestamp')
subset = df.loc['2026-01-01':'2026-01-31'].compute()
```

### Array Operations

```python
import dask.array as da
import numpy as np

# Create arrays
x = da.random.random((10000, 10000), chunks=(1000, 1000))
x = da.from_array(np.ones((5000, 5000)), chunks=(500, 500))
x = da.from_zarr('data.zarr')

# NumPy-like operations
y = x.mean(axis=0)
z = da.linalg.svd(x)
w = x @ x.T

# Rechunk for different access patterns
x = x.rechunk((2000, 500))

# Compute
result = y.compute()

# Store
da.to_zarr(x, 'output.zarr')
```

### Custom Parallelism with Delayed

```python
from dask import delayed
import pandas as pd

# Wrap functions
@delayed
def load_data(filename):
    return pd.read_csv(filename)

@delayed
def clean_data(df):
    return df.dropna()

@delayed
def process_data(df):
    return df.groupby('key').sum()

# Build task graph
files = ['data1.csv', 'data2.csv', 'data3.csv']
loaded = [load_data(f) for f in files]
cleaned = [clean_data(df) for df in loaded]
processed = [process_data(df) for df in cleaned]

# Combine and compute
result = delayed(pd.concat)(processed).compute()
```

### Distributed Futures

```python
from dask.distributed import Client

client = Client()

# Submit tasks
def square(x):
    return x ** 2

futures = client.map(square, range(1000))

# Gather results
results = client.gather(futures)

# Submit single task
future = client.submit(sum, range(1000))
result = future.result()

# Submit with dependencies
a = client.submit(sum, [1, 2, 3])
b = client.submit(lambda x: x * 2, a)
b.result()
```

## Resource Management

### Cluster Configuration

```python
from dask.distributed import Client, LocalCluster

# Configure workers
cluster = LocalCluster(
    n_workers=8,
    threads_per_worker=2,
    processes=True,
    memory_limit='8GB',
    dashboard_address=':8787'
)

client = Client(cluster)

# Adaptive scaling
cluster.adapt(minimum=2, maximum=10)

# Scale manually
cluster.scale(5)
```

### GPU Clusters

```python
from dask_cuda import LocalCUDACluster
from dask.distributed import Client

# Auto-detect GPUs
cluster = LocalCUDACluster()
client = Client(cluster)

# Manual GPU configuration
cluster = LocalCUDACluster(
    n_workers=4,  # 4 GPUs
    threads_per_worker=1,
    memory_limit='16GB',
    device_memory_limit='16GB'
)
```

### SLURM Integration

```python
from dask_jobqueue import SLURMCluster

cluster = SLURMCluster(
    cores=24,              # CPUs per job
    processes=4,           # Dask workers per job
    memory="96GB",         # Memory per job
    walltime="04:00:00",   # Job time limit
    queue="gpu",           # Partition
    job_extra_directives=[
        "--gres=gpu:2",
        "--account=myproject"
    ]
)

# Submit worker jobs
cluster.scale(jobs=20)  # 20 SLURM jobs

# Or adaptive
cluster.adapt(minimum_jobs=5, maximum_jobs=50)

client = Client(cluster)
```

## Performance Optimization

### Chunking Strategy

```python
# DataFrame partitions
df = dd.read_parquet('data.parquet', npartitions=100)
df = df.repartition(npartitions=50)

# Array chunks
x = da.from_array(data, chunks=(1000, 1000))  # 1000x1000 chunks
x = x.rechunk((2000, 500))  # Rechunk for different access

# Optimal chunk size: 100MB-1GB per chunk
```

### Persist vs Compute

```python
# Compute: Execute and return to client
result = df.sum().compute()

# Persist: Execute and keep in distributed memory
df = df.persist()  # Reuse without recomputation

# Explicit task graph
from dask import delayed, compute
x = delayed(load_data)('file1.csv')
y = delayed(load_data)('file2.csv')
z = delayed(merge)(x, y)
result = compute(z)[0]  # Execute entire graph
```

### Dashboard Monitoring

```python
client = Client()
print(client.dashboard_link)  # http://localhost:8787/status

# Monitor:
# - Task stream
# - Progress bars
# - Memory usage
# - Worker status
# - Task graph visualization
```

## Machine Learning Integration

### Dask-ML

```python
import dask_ml.cluster as dask_cluster
from dask_ml.linear_model import LogisticRegression

# Scalable k-means
kmeans = dask_cluster.KMeans(n_clusters=10)
kmeans.fit(X)

# Incremental learning
lr = LogisticRegression()
lr.fit(X_train, y_train)
```

### XGBoost

```python
import xgboost as xgb
import dask.dataframe as dd

df = dd.read_parquet('data.parquet')
X = df[['feature1', 'feature2', 'feature3']]
y = df['target']

dtrain = xgb.dask.DaskDMatrix(client, X, y)
params = {'max_depth': 5, 'objective': 'reg:squarederror'}
model = xgb.dask.train(client, params, dtrain, num_boost_round=100)
```

### GPU Machine Learning

```python
import cuml.dask as cuml_dask
from cuml.dask.ensemble import RandomForestClassifier

# GPU random forest
rf = RandomForestClassifier(n_estimators=100)
rf.fit(X_train, y_train)
predictions = rf.predict(X_test).compute()
```

## Common Issues

### Memory Management

```python
# Avoid large results returned to client
result = df.compute()  # Bad if result is huge

# Instead, write to disk
df.to_parquet('output/')  # Good

# Or aggregate before computing
result = df.groupby('key').mean().compute()  # Good if grouped result is small
```

### Scheduler Selection

```python
# Threaded (default for arrays)
import dask
dask.config.set(scheduler='threads')

# Processes (default for dataframes)
dask.config.set(scheduler='processes')

# Distributed (for clusters)
from dask.distributed import Client
client = Client()  # Uses distributed scheduler
```

### Task Graph Optimization

```python
# Avoid large task graphs
for i in range(10000):  # Bad: 10000 tasks
    result = delayed(process)(i)

# Use collections instead
data = da.from_delayed([delayed(process)(i) for i in range(10000)],
                       shape=(10000,), dtype=float)  # Good
```

## References

Detailed documentation in `references/`:

- **dataframe-array-bag.md**: Core collections API and patterns
- **distributed-scheduler.md**: Cluster setup, client usage, task scheduling
- **gpu-acceleration.md**: cuDF, CuPy, LocalCUDACluster, best practices
- **deployment.md**: Local, cloud, HPC deployment options

## Official Documentation

- [Dask Documentation](https://docs.dask.org)
- [Distributed Scheduler](https://distributed.dask.org)
- [Dask-Jobqueue (HPC)](https://jobqueue.dask.org)
- [Dask-CUDA](https://docs.rapids.ai/api/dask-cuda/stable/)
- [Dask-ML](https://ml.dask.org)

## Best Practices

1. **Use appropriate collections**: DataFrame for tabular, Array for numeric, Bag for semi-structured
2. **Chunk wisely**: 100MB-1GB chunks for optimal performance
3. **Persist intermediate results**: Use `.persist()` for reused computations
4. **Monitor with dashboard**: Track task progress and resource usage
5. **Write results to disk**: Avoid returning large datasets to client
6. **Use GPU acceleration**: cuDF/CuPy for GPU-compatible operations
7. **Deploy on HPC**: Use Dask-Jobqueue for SLURM/PBS clusters
8. **Adaptive scaling**: Let cluster adjust to workload demands
9. **Optimize task graphs**: Minimize overhead with collection-based parallelism
10. **Profile before scaling**: Ensure single-machine optimization first
