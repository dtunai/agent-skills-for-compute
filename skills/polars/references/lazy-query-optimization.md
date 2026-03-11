# Polars Lazy Query Optimization Reference

Sources:
- [Lazy API User Guide](https://docs.pola.rs/user-guide/lazy/)
- [Query Plan Documentation](https://docs.pola.rs/user-guide/lazy/query-plan/)

## Overview

Polars' lazy evaluation mode builds a query plan without executing operations. The optimizer then transforms this plan for maximum performance before execution.

**Key Benefits**:
- Predicate pushdown (filter early)
- Projection pushdown (read only needed columns)
- Parallel execution planning
- Memory efficiency
- Works with larger-than-RAM datasets (streaming)

## Creating LazyFrames

```python
import polars as pl

# From file (recommended)
lf = pl.scan_csv("data.csv")
lf = pl.scan_parquet("data.parquet")
lf = pl.scan_ndjson("data.ndjson")

# From existing DataFrame
df = pl.DataFrame({"a": [1, 2, 3], "b": [4, 5, 6]})
lf = df.lazy()

# From dict
lf = pl.LazyFrame({
    "name": ["Alice", "Bob"],
    "age": [25, 30]
})

# From multiple files
lf = pl.scan_parquet("data/*.parquet")
lf = pl.scan_csv("logs/2024-*.csv")
```

## Query Plans

### Viewing Query Plans

```python
lf = pl.scan_csv("data.csv")

query = (
    lf
    .filter(pl.col("status") == "active")
    .select(["id", "name", "value"])
    .group_by("name")
    .agg(pl.col("value").sum())
)

# Non-optimized plan (as written)
print(query.explain(optimized=False))

# Optimized plan (after transformations)
print(query.explain(optimized=True))

# Graphical visualization (requires Graphviz)
query.show_graph(optimized=False)
query.show_graph(optimized=True)
```

### Reading Query Plans

Query plans are displayed bottom-to-top, with each box representing an operation:

```
 AGGREGATE
    σ [col("status") == "active"]  # σ = filter (selection)
    π */3 [id, name, value]        # π = projection
    CSV SCAN data.csv              # Data source
```

**Symbols**:
- `σ` (sigma) = Filter/selection
- `π` (pi) = Projection/column selection
- `*/N` = All columns or N columns
- Numbers indicate columns remaining after each step

## Optimization Techniques

### Predicate Pushdown

Moves filters as close to the data source as possible, reducing data read from disk.

**Without Optimization:**

```python
# Reads ALL data, then filters
df = pl.read_csv("large.csv")
result = df.filter(pl.col("date") == "2024-01-01")
```

Query plan (inefficient):
```
σ [col("date") == "2024-01-01"]
  CSV SCAN large.csv [ALL COLUMNS]
```

**With Optimization:**

```python
# Filters DURING read
lf = pl.scan_csv("large.csv")
result = lf.filter(pl.col("date") == "2024-01-01").collect()
```

Optimized plan:
```
CSV SCAN large.csv
  FILTER [col("date") == "2024-01-01"]  # Applied during scan
```

**Example:**

```python
# Original query
query = (
    pl.scan_parquet("sales.parquet")
    .select(["date", "product", "revenue"])
    .filter(pl.col("date") >= "2024-01-01")
    .filter(pl.col("revenue") > 1000)
)

# Optimized execution pushes both filters to Parquet reader
# Only reads rows matching filters
result = query.collect()
```

### Projection Pushdown

Reads only necessary columns from the data source.

**Without Optimization:**

```python
# Reads ALL columns
df = pl.read_parquet("wide_table.parquet")
result = df.select(["id", "name"])
```

**With Optimization:**

```python
# Reads ONLY id and name columns
lf = pl.scan_parquet("wide_table.parquet")
result = lf.select(["id", "name"]).collect()
```

Optimized plan:
```
PARQUET SCAN wide_table.parquet
  PROJECT */2 [id, name]  # Only these columns read from file
```

**Example:**

```python
# Table has 100 columns, we need 3
query = (
    pl.scan_parquet("events.parquet")
    .select(["timestamp", "user_id", "event_type"])
    .filter(pl.col("event_type") == "purchase")
)

# Only reads 3 columns from Parquet, not all 100
result = query.collect()
```

### Slice Pushdown

Applies LIMIT operations at the source.

```python
# Only reads first 100 rows from file
query = pl.scan_csv("data.csv").head(100)
result = query.collect()

# Plan shows:
# CSV SCAN data.csv [LIMIT: 100]
```

### Common Subexpression Elimination

Avoids recomputing the same expression multiple times.

```python
# Without optimization: computes col("a") + col("b") twice
query = (
    pl.scan_parquet("data.parquet")
    .with_columns([
        (pl.col("a") + pl.col("b")).alias("sum"),
        (pl.col("a") + pl.col("b") * 2).alias("weighted")
    ])
)

# Optimizer recognizes shared computation
# Only computes col("a") + col("b") once
```

### Aggregate Pushdown

Moves aggregations closer to the data source when possible.

```python
# Count pushed to Parquet metadata (very fast)
count = pl.scan_parquet("data.parquet").select(pl.count()).collect()

# Aggregation on filtered data optimized
result = (
    pl.scan_parquet("data.parquet")
    .filter(pl.col("status") == "active")
    .select(pl.col("value").sum())
    .collect()
)
```

## Execution Modes

### Standard Collection

```python
# Execute query and return DataFrame
df = lf.collect()

# With GPU
df = lf.collect(engine="gpu")

# Multiple queries in parallel
df1, df2 = pl.collect_all([query1, query2])
```

### Streaming Execution

For datasets larger than RAM.

```python
# Streaming mode (processes in chunks)
df = lf.collect(streaming=True)

# Combines with other optimizations
result = (
    pl.scan_parquet("huge.parquet")
    .filter(pl.col("date") >= "2024-01-01")
    .group_by("category")
    .agg(pl.col("value").sum())
    .collect(streaming=True)
)
```

**When to use streaming:**
- Dataset > RAM
- Simple aggregations
- Pipeline-friendly operations

**Limitations:**
- Not all operations support streaming
- May be slower than in-memory for small data
- Some optimizations unavailable

### Lazy Collect with Partitions

```python
# Collect into multiple DataFrames (parallel processing)
partitions = lf.collect(
    streaming=True,
    slice_pushdown=True
)
```

## Building Efficient Queries

### Chain Operations Efficiently

```python
# Good: filters early, selects needed columns
query = (
    pl.scan_parquet("large.parquet")
    .filter(pl.col("date") == "2024-01-01")  # Early filter
    .select(["id", "value"])                  # Only needed columns
    .group_by("id")
    .agg(pl.col("value").sum())
)

# Less efficient: filters late
query = (
    pl.scan_parquet("large.parquet")
    .group_by("id")
    .agg(pl.col("value").sum())
    .filter(pl.col("value_sum") > 1000)  # Late filter
)
```

### Combine Filters

```python
# Good: single combined filter
lf.filter(
    (pl.col("age") > 30) & (pl.col("city") == "NYC")
)

# Less efficient: separate filters
lf.filter(pl.col("age") > 30).filter(pl.col("city") == "NYC")
```

### Avoid Collecting Early

```python
# Bad: collects mid-pipeline
lf1 = pl.scan_parquet("data1.parquet")
df1 = lf1.collect()  # Forces execution
lf2 = df1.lazy().join(pl.scan_parquet("data2.parquet"), on="id")

# Good: keep lazy until the end
lf1 = pl.scan_parquet("data1.parquet")
lf2 = lf1.join(pl.scan_parquet("data2.parquet"), on="id")
result = lf2.collect()  # Single optimized execution
```

### Use Scan for File Reads

```python
# Prefer scan_* over read_*
lf = pl.scan_parquet("data.parquet")  # Enables pushdown
df = pl.read_parquet("data.parquet")  # Reads immediately

# scan_* enables:
# - Predicate pushdown
# - Projection pushdown
# - Slice pushdown
# - Statistics from file metadata
```

## Query Debugging

### Explain Plans

```python
# View optimized plan
print(query.explain())

# Include type information
print(query.explain(type_coercion=True))

# Streaming plan
print(query.explain(streaming=True))

# Compare before/after
print("=== Non-Optimized ===")
print(query.explain(optimized=False))
print("\\n=== Optimized ===")
print(query.explain(optimized=True))
```

### Profiling

```python
# Profile execution
df = lf.collect(profile=True)

# Returns (DataFrame, profiling_info)
# profiling_info contains timing details
```

### Common Issues

**Issue: No Predicate Pushdown**

```python
# Cause: using Python expressions
lf.filter(lambda x: x["value"] > 100)  # Can't optimize

# Fix: use Polars expressions
lf.filter(pl.col("value") > 100)  # Optimized
```

**Issue: Collecting Too Early**

```python
# Cause: collect() in middle of pipeline
df = lf1.collect()
result = df.lazy().join(lf2, on="id").collect()

# Fix: stay lazy
result = lf1.join(lf2, on="id").collect()
```

**Issue: Reading Unneeded Columns**

```python
# Cause: selecting after reading
df = pl.read_parquet("wide.parquet")
result = df.select(["a", "b"])

# Fix: use scan with select
result = pl.scan_parquet("wide.parquet").select(["a", "b"]).collect()
```

## Advanced Patterns

### Incremental Processing

```python
# Process data in date partitions
dates = ["2024-01-01", "2024-01-02", "2024-01-03"]

results = []
for date in dates:
    lf = pl.scan_parquet(f"data/{date}/*.parquet")
    result = lf.filter(pl.col("status") == "active").collect()
    results.append(result)

combined = pl.concat(results)
```

### Parallel Queries

```python
# Execute multiple independent queries in parallel
query1 = pl.scan_parquet("sales.parquet").group_by("region").agg(pl.col("revenue").sum())
query2 = pl.scan_parquet("inventory.parquet").group_by("warehouse").agg(pl.col("units").sum())

# Parallel execution
df1, df2 = pl.collect_all([query1, query2])
```

### Caching Intermediate Results

```python
# Cache expensive intermediate computation
lf = pl.scan_parquet("large.parquet")
expensive = lf.filter(complex_condition).cache()

# Reuse cached result
result1 = expensive.select(["a", "b"]).collect()
result2 = expensive.select(["c", "d"]).collect()
```

### Window Function Optimization

```python
# Efficient window operations
result = (
    pl.scan_parquet("data.parquet")
    .with_columns([
        pl.col("value").mean().over("category").alias("category_mean"),
        pl.col("value").rank().over("category").alias("rank")
    ])
    .collect()
)
```

## Performance Benchmarking

```python
import time

# Eager execution
start = time.time()
df = pl.read_parquet("data.parquet")
result = df.filter(pl.col("value") > 100)
eager_time = time.time() - start

# Lazy execution
start = time.time()
lf = pl.scan_parquet("data.parquet")
result = lf.filter(pl.col("value") > 100).collect()
lazy_time = time.time() - start

print(f"Eager: {eager_time:.2f}s")
print(f"Lazy: {lazy_time:.2f}s")
print(f"Speedup: {eager_time / lazy_time:.2f}x")
```

## Best Practices

1. **Default to Lazy**: Use `scan_*` instead of `read_*`
2. **Filter Early**: Place filters before other operations
3. **Select Early**: Choose needed columns as soon as possible
4. **Avoid Collecting Early**: Stay lazy until final result
5. **Use Expressions**: Polars expressions over Python lambdas
6. **Combine Filters**: Single combined filter vs multiple filters
7. **Inspect Plans**: Use `explain()` to verify optimizations
8. **Streaming for Large Data**: Use `collect(streaming=True)` when data > RAM
9. **Profile**: Use `profile=True` to identify bottlenecks
10. **GPU for Heavy Compute**: Use `collect(engine="gpu")` for aggregations/joins

## Common Pitfalls

- **Using `read_*` for large files**: Misses optimization opportunities
- **Collecting mid-pipeline**: Breaks optimization chain
- **Python UDFs in lazy context**: Can't be optimized
- **Complex expressions without caching**: Recomputed unnecessarily
- **Not checking explain()**: Missing obvious inefficiencies
- **Assuming all operations stream**: Some operations require materialization
- **Over-using streaming**: May be slower for small data
- **Ignoring file formats**: Parquet enables more optimizations than CSV
