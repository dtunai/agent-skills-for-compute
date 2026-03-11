---
name: polars
description: "Polars DataFrame library with GPU acceleration via RAPIDS cuDF — lazy evaluation, expressions API, and high-performance data transformations"
license: MIT
metadata:
  author: Agent Cluster
  tags: [polars, dataframe, gpu, rapids, cudf, lazy-evaluation, expressions]
---

# Polars GPU Skill

Fast DataFrame library with GPU acceleration via RAPIDS cuDF engine. Lazy evaluation, expressions API, and optimized query execution.

**Official Sources:**
- [Polars Documentation](https://docs.pola.rs/)
- [GPU Support Guide](https://docs.pola.rs/user-guide/gpu-support/)
- [cuDF Polars Engine](https://docs.rapids.ai/api/cudf/stable/cudf_polars/)

## Installation

```bash
# CPU-only Polars
pip install polars

# With GPU support
pip install polars[gpu]

# CUDA 11 (RAPIDS v25.06 and earlier)
pip install polars cudf-polars-cu11
```

**GPU Requirements:**
- NVIDIA Volta™ or newer (compute capability 7.0+)
- CUDA 12 (CUDA 11 support ends with RAPIDS v25.06)
- Linux or WSL2

## Quick Start

```python
import polars as pl

# DataFrame (eager)
df = pl.DataFrame({"name": ["Alice", "Bob"], "age": [25, 30]})
df.filter(pl.col("age") > 28)

# LazyFrame (deferred)
lf = pl.LazyFrame({"a": [1, 2, 3], "b": [4, 5, 6]})
result = lf.filter(pl.col("a") > 1).collect()      # CPU
result = lf.filter(pl.col("a") > 1).collect(engine="gpu")  # GPU
```

## GPU Execution

```python
# CPU vs GPU
df = lf.collect()                    # CPU
df = lf.collect(engine="gpu")        # GPU

# Config: device, raise_on_fail, verbose
df = lf.collect(engine=pl.GPUEngine(device=1, raise_on_fail=True))

# Peak performance: export POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=0
```

## Expressions

```python
# Column selection
pl.col("name")                        # Single
pl.col("a", "b", "c")                 # Multiple
pl.col("^value_.*$")                  # Regex
pl.col(pl.Int64)                      # By dtype

# Arithmetic & comparisons
pl.col("a") + 10
pl.col("age") > 30
pl.col("value").is_between(10, 20)

# Aggregations
pl.col("value").sum()
pl.col("value").mean()
df.group_by("category").agg([
    pl.col("value").sum().alias("total"),
    pl.col("id").count().alias("count")
])
```

# String, temporal, list operations
pl.col("text").str.to_lowercase()
pl.col("text").str.contains("pattern")
pl.col("date").dt.year()
pl.col("timestamp").dt.truncate("1h")
pl.col("items").list.sum()
pl.col("items").list.explode()
```

## I/O Operations

```python
# CSV, Parquet, JSON
df = pl.read_csv("data.csv")
lf = pl.scan_parquet("data.parquet")  # Lazy (recommended)
df = pl.read_ndjson("data.ndjson")

# Write
df.write_csv("output.csv")
df.write_parquet("output.parquet", compression="snappy")
df.write_ndjson("output.ndjson")

# Multiple files
df = pl.scan_parquet("data/*.parquet").collect()
```

## Lazy API and Query Optimization

```python
# Build query
lf = pl.scan_parquet("data.parquet")
query = (
    lf
    .filter(pl.col("status") == "active")  # Predicate pushdown
    .select(["id", "name", "value"])       # Projection pushdown
    .group_by("name")
    .agg(pl.col("value").sum())
)

print(query.explain(optimized=True))  # View optimized plan
result = query.collect()              # Execute
```

## Data Transformations

```python
# Filter, select, add columns
df.filter((pl.col("age") > 30) & (pl.col("city") == "NYC"))
df.select(["name", "age"])
df.with_columns((pl.col("price") * pl.col("quantity")).alias("total"))

# Joins
df1.join(df2, on="id", how="left")
df1.join(df2, on=["id", "date"], how="inner")

# Group by and aggregate
df.group_by(["category", "region"]).agg([
    pl.col("value").sum().alias("total"),
    pl.col("id").count().alias("count")
])

# Window functions
df.with_columns(pl.col("value").sum().over("category").alias("total"))
df.with_columns(pl.col("value").rolling_mean(window_size=3).alias("ma"))

# Pivot/melt
df.pivot(values="value", index="date", columns="category")
df.melt(id_vars=["id"], value_vars=["a", "b", "c"])
```

## GPU-Specific Considerations

**Supported:** LazyFrame, CSV/Parquet/NDJSON, numeric/string/datetime ops, aggregations, joins, window functions

**Not Supported:** Eager API, streaming, Date/Categorical/Enum types, UDFs

**Best Performance:** Grouped aggregations, complex joins, compute-heavy ops on large datasets (up to ~50-100 GiB)

**Memory:** Set `POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=0` for peak performance

## SQL Interface

```python
result = pl.sql("SELECT a * 2 FROM self WHERE a > 1", self=lf).collect()
result = pl.sql("SELECT category, SUM(value) FROM data GROUP BY category",
                data=pl.scan_parquet("data.parquet")).collect(engine="gpu")
```

## Type System

**Types:** Int8/16/32/64, UInt8/16/32/64, Float32/64, String, Binary, Date, Datetime, Duration, Time, List, Struct, Array, Boolean, Categorical, Enum

```python
# Cast
df.with_columns(pl.col("value").cast(pl.Float64))
pl.read_csv("data.csv", dtypes={"id": pl.Int64})
```

## Performance Tips

1. Use Lazy API (`scan_*` not `read_*`)
2. GPU for compute-heavy ops (aggregations, joins)
3. Set `POLARS_GPU_ENABLE_CUDA_MANAGED_MEMORY=0`
4. Select columns early, filter early
5. Use appropriate types

## Common Patterns

```python
# GPU ETL pipeline
lf = pl.scan_parquet("raw/*.parquet")
result = (
    lf
    .filter(pl.col("status") == "active")
    .with_columns((pl.col("price") * pl.col("quantity")).alias("total"))
    .group_by(["hour", "category"])
    .agg(pl.col("total").sum().alias("revenue"))
).collect(engine="gpu")
result.write_parquet("output.parquet")
```

## References

- **[Fundamentals](references/fundamentals.md)** - DataFrame, LazyFrame, Series, core concepts
- **[Lazy Query Optimization](references/lazy-query-optimization.md)** - Query plans, optimizations, predicate/projection pushdown
- **[GPU Engine](references/gpu-engine.md)** - cuDF integration, configuration, performance tuning
- **[I/O Operations](references/io-operations.md)** - CSV, Parquet, JSON, cloud storage, multi-file reads
- **[Expressions API](references/expressions-api.md)** - Expression syntax, namespaces, aggregations, window functions
