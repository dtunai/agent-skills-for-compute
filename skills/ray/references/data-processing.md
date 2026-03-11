# Ray Data Processing Reference

Sources:
- [Ray Data Documentation](https://docs.ray.io/en/latest/data/data.html)
- [Ray Data Dataset API](https://docs.ray.io/en/latest/data/api/dataset.html)
- [Ray Data Transformations](https://docs.ray.io/en/latest/data/transforming-data.html)
- [Ray Data Loading](https://docs.ray.io/en/latest/data/loading-data.html)

## Overview

Ray Data is a distributed data processing library designed specifically for AI workloads. According to the official documentation, it "provides flexible and performant APIs for common operations such as batch inference, data preprocessing, and data loading for ML training."

## Installation

```bash
# Install Ray Data
pip install -U "ray[data]"

# With specific backends
pip install -U "ray[data,aws]"  # AWS S3
pip install -U "ray[data,gcp]"  # Google Cloud Storage
```

## Creating Datasets

### From Files

```python
import ray

# Parquet (recommended)
ds = ray.data.read_parquet("s3://bucket/data/*.parquet")
ds = ray.data.read_parquet("data/", parallelism=200)

# CSV
ds = ray.data.read_csv("data/*.csv")
ds = ray.data.read_csv("data.csv", schema={"col1": int, "col2": str})

# JSON
ds = ray.data.read_json("data/*.json")
ds = ray.data.read_json("data.json", lines=True)

# Images
ds = ray.data.read_images("images/")

# Binary files
ds = ray.data.read_binary_files("files/*.bin")

# Text files
ds = ray.data.read_text("logs/*.log")
```

### From Python Data

```python
# From list
ds = ray.data.from_items([{"x": 1, "y": 2}, {"x": 3, "y": 4}])
ds = ray.data.from_items([1, 2, 3, 4, 5])

# From pandas
import pandas as pd
df = pd.DataFrame({"x": [1, 2, 3], "y": [4, 5, 6]})
ds = ray.data.from_pandas(df)

# From NumPy
import numpy as np
arr = np.array([[1, 2], [3, 4]])
ds = ray.data.from_numpy(arr)

# From range
ds = ray.data.range(1000000)
ds = ray.data.range_table(1000000, num_columns=5)
```

### From Dask/Spark

```python
# From Dask
import dask.dataframe as dd
dask_df = dd.read_parquet("data.parquet")
ds = ray.data.from_dask(dask_df)

# From Spark
spark_df = spark.read.parquet("data.parquet")
ds = ray.data.from_spark(spark_df)
```

## Transformations

### Map Operations

```python
# Map rows
def square(row):
    row["x_squared"] = row["x"] ** 2
    return row

ds = ray.data.range(100)
ds = ds.map(square)

# Map batches (faster)
def batch_square(batch):
    import pandas as pd
    batch["x_squared"] = batch["x"] ** 2
    return batch

ds = ds.map_batches(batch_square, batch_format="pandas")

# Map batches with NumPy
def numpy_transform(batch):
    import numpy as np
    batch["normalized"] = (batch["value"] - np.mean(batch["value"])) / np.std(batch["value"])
    return batch

ds = ds.map_batches(numpy_transform, batch_format="numpy")
```

### Filter

```python
# Filter rows
ds = ray.data.range(1000)
ds = ds.filter(lambda row: row["id"] % 2 == 0)

# Filter batches
def filter_batch(batch):
    return batch[batch["value"] > 10]

ds = ds.map_batches(filter_batch, batch_format="pandas")
```

### Flat Map

```python
# Expand rows
def expand(row):
    return [{"value": row["x"]}, {"value": row["x"] * 2}]

ds = ray.data.from_items([{"x": 1}, {"x": 2}])
ds = ds.flat_map(expand)  # 4 rows
```

### Select Columns

```python
# Select specific columns
ds = ray.data.read_parquet("data.parquet")
ds = ds.select_columns(["feature1", "feature2", "label"])

# Drop columns
ds = ds.drop_columns(["unnecessary_col"])

# Add column
ds = ds.add_column("new_col", lambda row: row["x"] + row["y"])
```

## Aggregations

### Basic Aggregations

```python
ds = ray.data.range(1000)

# Count
count = ds.count()

# Sum
total = ds.sum(on="id")

# Mean
avg = ds.mean(on="id")

# Min/Max
minimum = ds.min(on="id")
maximum = ds.max(on="id")

# Std
std_dev = ds.std(on="id")
```

### GroupBy

```python
ds = ray.data.read_parquet("data.parquet")

# Group and aggregate
grouped = ds.groupby("category").count()
grouped = ds.groupby("category").mean("value")
grouped = ds.groupby("category").sum("amount")

# Multiple aggregations
grouped = ds.groupby("category").aggregate(
    count=("id", "count"),
    total=("amount", "sum"),
    avg=("amount", "mean")
)
```

### Custom Aggregations

```python
from ray.data.aggregate import AggregateFn

class MedianAggregation(AggregateFn):
    def __init__(self, column):
        self.column = column

    def accumulate_row(self, accumulator, row):
        accumulator.append(row[self.column])
        return accumulator

    def merge(self, acc1, acc2):
        return acc1 + acc2

    def finalize(self, accumulator):
        import numpy as np
        return np.median(accumulator)

median = ds.aggregate(MedianAggregation("value"))
```

## Shuffling and Sorting

### Repartition

```python
# Change number of blocks
ds = ray.data.range(10000)
ds = ds.repartition(100)  # 100 blocks

# By size
ds = ds.repartition(shuffle=False)  # Even distribution
```

### Sort

```python
# Sort by column
ds = ds.sort("timestamp")

# Descending
ds = ds.sort("value", descending=True)

# Multiple columns
ds = ds.sort([("category", "ascending"), ("value", "descending")])
```

### Random Shuffle

```python
# Shuffle data
ds = ds.random_shuffle()

# With seed
ds = ds.random_shuffle(seed=42)
```

## Batch Operations

### Batch Size

```python
# Configure batch size
ds = ray.data.read_parquet("data.parquet")
ds = ds.map_batches(
    process_function,
    batch_size=1000,  # 1000 rows per batch
    batch_format="pandas"
)

# Dynamic batching
ds = ds.map_batches(
    process_function,
    batch_size=None,  # Use default
    batch_format="pandas"
)
```

### Batch Formats

```python
# Pandas DataFrame
def pandas_batch(batch):
    # batch is pandas.DataFrame
    return batch[batch["value"] > 0]

ds = ds.map_batches(pandas_batch, batch_format="pandas")

# NumPy array
def numpy_batch(batch):
    # batch is dict of numpy arrays
    import numpy as np
    return {"result": np.sqrt(batch["value"])}

ds = ds.map_batches(numpy_batch, batch_format="numpy")

# PyArrow Table
def arrow_batch(batch):
    # batch is pyarrow.Table
    import pyarrow.compute as pc
    return batch.append_column("doubled", pc.multiply(batch["value"], 2))

ds = ds.map_batches(arrow_batch, batch_format="pyarrow")
```

## Resource Specification

### CPU Resources

```python
# Specify CPUs per task
ds = ds.map_batches(
    cpu_intensive_function,
    num_cpus=2,  # 2 CPUs per task
    batch_size=100
)
```

### GPU Resources

```python
# GPU batch processing
class GPUPreprocessor:
    def __init__(self):
        import torch
        self.device = torch.device("cuda")

    def __call__(self, batch):
        import torch
        tensor = torch.tensor(batch["data"]).to(self.device)
        result = process_on_gpu(tensor)
        return {"processed": result.cpu().numpy()}

ds = ds.map_batches(
    GPUPreprocessor,
    num_gpus=1,  # 1 GPU per task
    batch_size=32
)
```

### Concurrency

```python
# Control parallelism
ds = ds.map_batches(
    process_function,
    concurrency=10,  # Max 10 concurrent tasks
    batch_size=100
)
```

## Reading and Writing

### Read Options

```python
# Parallelism
ds = ray.data.read_parquet(
    "s3://bucket/data/*.parquet",
    parallelism=200  # 200 read tasks
)

# Partitioning
ds = ray.data.read_parquet(
    "s3://bucket/data/",
    partition_filter=lambda d: d["year"] == "2026"
)

# Schema override
ds = ray.data.read_csv(
    "data.csv",
    schema={"col1": int, "col2": str, "col3": float}
)
```

### Write Operations

```python
# Write Parquet
ds.write_parquet("output/", num_rows_per_file=10000)

# Write CSV
ds.write_csv("output/")

# Write JSON
ds.write_json("output/", lines=True)

# Write to cloud
ds.write_parquet("s3://bucket/output/", compression="snappy")
```

## Iteration and Consumption

### Iterate Batches

```python
# Iterate over batches
for batch in ds.iter_batches(batch_size=100, batch_format="pandas"):
    process_batch(batch)

# With prefetch
for batch in ds.iter_batches(batch_size=100, prefetch_batches=2):
    # Next batch prefetched while processing current
    process_batch(batch)
```

### Take Samples

```python
# Take first N rows
sample = ds.take(10)

# Take all rows (fits in memory)
all_rows = ds.take_all()

# Take batch
batch = ds.take_batch(100, batch_format="pandas")
```

### Show

```python
# Display sample
ds.show(5)  # First 5 rows

# Limit columns
ds.show(5, columns=["col1", "col2"])
```

## ML Integration

### PyTorch Integration

```python
import torch
from ray.data import Dataset

ds = ray.data.read_parquet("train.parquet")

# Convert to PyTorch DataLoader
torch_ds = ds.to_torch(
    label_column="label",
    feature_columns=["feature1", "feature2"],
    batch_size=32
)

# Use in training
for batch in torch_ds:
    features, labels = batch
    # Training step
```

### TensorFlow Integration

```python
import tensorflow as tf

ds = ray.data.read_parquet("train.parquet")

# Convert to TensorFlow Dataset
tf_ds = ds.to_tf(
    label_column="label",
    feature_columns=["feature1", "feature2"],
    batch_size=32
)

# Use in training
model.fit(tf_ds, epochs=10)
```

### Batch Inference

```python
class ModelInference:
    def __init__(self):
        self.model = load_model()

    def __call__(self, batch):
        import pandas as pd
        predictions = self.model.predict(batch["features"])
        return pd.DataFrame({"predictions": predictions})

ds = ray.data.read_parquet("test.parquet")
predictions = ds.map_batches(
    ModelInference,
    batch_size=32,
    num_gpus=1,
    concurrency=4
)

predictions.write_parquet("predictions/")
```

## Performance Optimization

### Execution Settings

```python
import ray

# Configure execution
ctx = ray.data.DataContext.get_current()

# Streaming execution
ctx.execution_options.preserve_order = False
ctx.execution_options.verbose_progress = True

# Resource limits
ctx.execution_options.resource_limits.object_store_memory = 10 * 1024**3  # 10GB
```

### Pipelining

```python
# Automatic pipelining
ds = ray.data.read_parquet("input.parquet")
ds = ds.map_batches(preprocess)
ds = ds.map_batches(feature_engineering)
ds = ds.map_batches(model_inference, num_gpus=1)

# Execution pipelined automatically
ds.write_parquet("output/")
```

### Caching

```python
# Cache dataset in memory
ds = ray.data.read_parquet("data.parquet")
ds = ds.map_batches(expensive_preprocessing)

# Materialize and cache
ds = ds.materialize()

# Reuse multiple times
result1 = ds.filter(lambda x: x["category"] == "A").count()
result2 = ds.filter(lambda x: x["category"] == "B").count()
```

## Statistics and Debugging

### Dataset Statistics

```python
# Get statistics
stats = ds.stats()

# Row count
count = ds.count()

# Schema
schema = ds.schema()

# Columns
columns = ds.columns()

# Size estimate
size_bytes = ds.size_bytes()
```

### Debugging

```python
# Show execution plan
ds.show_plan()

# Limit rows for testing
ds_sample = ds.limit(1000)

# Inspect schema
print(ds.schema())

# Take sample
sample = ds.take(5)
for row in sample:
    print(row)
```

## Common Patterns

### ETL Pipeline

```python
# Extract
ds = ray.data.read_parquet("s3://bucket/raw/*.parquet")

# Transform
ds = ds.map_batches(clean_data, batch_format="pandas")
ds = ds.filter(lambda row: row["valid"])
ds = ds.map_batches(feature_engineering, batch_format="pandas")

# Load
ds.write_parquet("s3://bucket/processed/", num_rows_per_file=50000)
```

### Multi-Modal Processing

```python
# Load images and labels
images = ray.data.read_images("images/")
labels = ray.data.read_csv("labels.csv")

# Zip datasets
ds = images.zip(labels)

# Process
def preprocess_image(batch):
    import numpy as np
    processed = np.array([resize_and_normalize(img) for img in batch["image"]])
    return {"image": processed, "label": batch["label"]}

ds = ds.map_batches(preprocess_image, batch_format="pandas")
```

### Data Validation

```python
def validate_batch(batch):
    import pandas as pd

    # Check for nulls
    if batch.isnull().any().any():
        raise ValueError("Null values found")

    # Check ranges
    if (batch["value"] < 0).any():
        raise ValueError("Negative values found")

    return batch

ds = ds.map_batches(validate_batch, batch_format="pandas")
```

## Best Practices

1. **Use Parquet format**: Better compression and performance than CSV
2. **Specify batch_size**: Control memory usage and parallelism
3. **Use map_batches over map**: More efficient for most operations
4. **Materialize expensive operations**: Cache intermediate results
5. **Specify resources**: Help scheduler allocate correctly
6. **Use appropriate batch format**: Pandas for tabular, NumPy for numeric
7. **Pipeline operations**: Combine transformations for efficiency
8. **Monitor object store**: Prevent memory issues
9. **Use concurrency wisely**: Balance parallelism and resources
10. **Profile with stats()**: Identify bottlenecks

## Common Pitfalls

- **Reading too many small files**: Use fewer larger files
- **Not specifying batch_size**: Memory issues or poor performance
- **Using map for batch operations**: Slower than map_batches
- **Calling take_all() on large datasets**: Out of memory
- **Not materializing reused datasets**: Recomputation overhead
- **Ignoring schema**: Type errors in transformations
- **Over-parallelizing**: Too many small tasks
- **Not using resources**: Poor GPU utilization
- **Blocking iteration**: Use async iteration when possible
- **Forgetting to write results**: Data lost after processing
