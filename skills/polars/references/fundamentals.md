# Polars Fundamentals Reference

Sources:
- [Polars Documentation](https://docs.pola.rs/)
- [API Reference](https://docs.pola.rs/api/python/stable/)

## Overview

Polars is a blazingly fast DataFrame library implemented in Rust with Python bindings. It leverages Apache Arrow's columnar memory format and provides both eager and lazy evaluation modes.

**Key Features**:
- Lightning-fast performance through Rust + Arrow
- Lazy evaluation with query optimization
- Expressive API with method chaining
- GPU acceleration via RAPIDS cuDF
- Zero-copy data sharing with Arrow ecosystem
- Multi-threading by default

## Core Data Structures

### DataFrame (Eager Evaluation)

DataFrames provide immediate execution of operations.

```python
import polars as pl

# Create DataFrame from dict
df = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie", "Diana"],
    "age": [25, 30, 35, 28],
    "city": ["NYC", "SF", "LA", "NYC"],
    "salary": [70000, 85000, 90000, 75000]
})

# From records
df = pl.DataFrame([
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 30}
])

# From Arrow table
import pyarrow as pa
arrow_table = pa.table({"a": [1, 2, 3], "b": [4, 5, 6]})
df = pl.from_arrow(arrow_table)

# From NumPy
import numpy as np
arr = np.array([[1, 2], [3, 4]])
df = pl.DataFrame(arr, schema=["col1", "col2"])

# From Pandas
import pandas as pd
pdf = pd.DataFrame({"a": [1, 2, 3]})
df = pl.from_pandas(pdf)
```

**DataFrame Properties:**

```python
# Shape
print(df.shape)  # (rows, cols)
print(df.height)  # Number of rows
print(df.width)   # Number of columns

# Columns
print(df.columns)  # List of column names
print(df.dtypes)   # List of data types
print(df.schema)   # Dict of {column: dtype}

# Preview
print(df.head())    # First 5 rows
print(df.tail())    # Last 5 rows
print(df.sample(3)) # Random 3 rows
print(df.glimpse()) # Transposed preview
```

### LazyFrame (Deferred Execution)

LazyFrames build query plans without executing until `.collect()`.

```python
# Create LazyFrame from dict
lf = pl.LazyFrame({
    "a": [1, 2, 3],
    "b": [4, 5, 6]
})

# From existing DataFrame
lf = df.lazy()

# From file (most common)
lf = pl.scan_csv("data.csv")
lf = pl.scan_parquet("data.parquet")
lf = pl.scan_ndjson("data.ndjson")

# Build query
query = (
    lf
    .filter(pl.col("value") > 100)
    .select(["id", "name", "value"])
    .group_by("name")
    .agg(pl.col("value").sum())
)

# Inspect plan
print(query.explain(optimized=False))  # Non-optimized
print(query.explain(optimized=True))   # Optimized

# Execute
result = query.collect()  # Returns DataFrame

# Streaming (for large datasets)
result = query.collect(streaming=True)

# GPU execution
result = query.collect(engine="gpu")
```

**LazyFrame Benefits:**
- Query optimization (predicate/projection pushdown)
- Memory efficiency
- Parallel execution planning
- Works with datasets larger than RAM (streaming)

### Series (1D Array)

Series are single columns with a name and dtype.

```python
# Create Series
s = pl.Series("values", [1, 2, 3, 4, 5])

# From list
s = pl.Series([10, 20, 30])

# From NumPy
s = pl.Series(np.array([1.5, 2.5, 3.5]))

# Properties
print(s.name)   # Series name
print(s.dtype)  # Data type
print(s.len())  # Length

# Access values
print(s[0])         # First element
print(s[-1])        # Last element
print(s[1:3])       # Slice
print(s.to_list())  # Convert to list

# Operations
s.sum()
s.mean()
s.std()
s.min()
s.max()
s.median()
```

## Data Types

### Numeric Types

```python
# Integers
pl.Int8     # -128 to 127
pl.Int16    # -32768 to 32767
pl.Int32    # -2^31 to 2^31-1
pl.Int64    # -2^63 to 2^63-1

pl.UInt8    # 0 to 255
pl.UInt16   # 0 to 65535
pl.UInt32   # 0 to 2^32-1
pl.UInt64   # 0 to 2^64-1

# Floats
pl.Float32  # 32-bit floating point
pl.Float64  # 64-bit floating point

# Example
df = pl.DataFrame({
    "int_col": pl.Series([1, 2, 3], dtype=pl.Int32),
    "float_col": pl.Series([1.5, 2.5, 3.5], dtype=pl.Float64)
})
```

### String and Binary

```python
# String (UTF-8)
pl.String

# Binary (byte sequences)
pl.Binary

# Example
df = pl.DataFrame({
    "text": pl.Series(["hello", "world"], dtype=pl.String),
    "data": pl.Series([b"\\x00\\x01", b"\\xff"], dtype=pl.Binary)
})
```

### Temporal Types

```python
# Date (calendar date)
pl.Date

# Datetime (timestamp)
pl.Datetime         # Microsecond precision
pl.Datetime("ms")   # Millisecond
pl.Datetime("ns")   # Nanosecond
pl.Datetime("us", time_zone="UTC")  # With timezone

# Time (time of day)
pl.Time

# Duration (time delta)
pl.Duration         # Microsecond
pl.Duration("ms")   # Millisecond

# Example
from datetime import date, datetime, time, timedelta

df = pl.DataFrame({
    "date": [date(2024, 1, 1), date(2024, 1, 2)],
    "timestamp": [datetime(2024, 1, 1, 12, 0), datetime(2024, 1, 2, 13, 30)],
    "time": [time(12, 0), time(13, 30)],
    "delta": [timedelta(days=1), timedelta(hours=2)]
})

print(df.dtypes)
# [Date, Datetime(time_unit='us', time_zone=None), Time, Duration(time_unit='us')]
```

### Nested Types

```python
# List (variable-length sequences)
pl.List(pl.Int64)
pl.List(pl.String)

df = pl.DataFrame({
    "lists": [[1, 2, 3], [4, 5], [6, 7, 8, 9]]
})

# Struct (named fields)
pl.Struct({
    "field1": pl.Int64,
    "field2": pl.String
})

df = pl.DataFrame({
    "structs": [
        {"a": 1, "b": "x"},
        {"a": 2, "b": "y"}
    ]
})

# Array (fixed-length sequences)
pl.Array(pl.Float64, width=3)

df = pl.DataFrame({
    "arrays": [[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]
})
```

### Categorical and Enum

```python
# Categorical (string with limited cardinality)
pl.Categorical

df = pl.DataFrame({
    "category": pl.Series(["A", "B", "A", "C", "B"], dtype=pl.Categorical)
})

# Memory efficient for repeated strings
# Internally uses integer encoding

# Enum (fixed set of values)
pl.Enum(["red", "green", "blue"])

df = pl.DataFrame({
    "color": pl.Series(["red", "blue", "red"], dtype=pl.Enum(["red", "green", "blue"]))
})
```

### Boolean

```python
pl.Boolean

df = pl.DataFrame({
    "flag": [True, False, True, False]
})

# Useful for filtering
mask = df["value"] > 50  # Returns boolean Series
filtered = df.filter(mask)
```

### Null Type

```python
pl.Null

# Represents null values
df = pl.DataFrame({
    "values": [1, None, 3, None, 5]
})

# Check for nulls
df.filter(pl.col("values").is_null())
df.filter(pl.col("values").is_not_null())

# Fill nulls
df.with_columns(pl.col("values").fill_null(0))
```

## Type Casting

```python
# Cast during creation
df = pl.DataFrame({
    "a": pl.Series([1, 2, 3], dtype=pl.Int8)
})

# Cast existing column
df = df.with_columns(pl.col("a").cast(pl.Float64))

# Cast during read
df = pl.read_csv("data.csv", dtypes={"id": pl.Int64, "value": pl.Float32})

# Strict casting (errors on failure)
df.with_columns(pl.col("value").cast(pl.Int64, strict=True))

# Non-strict (nulls on failure)
df.with_columns(pl.col("value").cast(pl.Int64, strict=False))
```

## Null Handling

```python
# Check for nulls
df.null_count()  # Nulls per column

# Filter
df.filter(pl.col("value").is_null())
df.filter(pl.col("value").is_not_null())

# Fill
df.with_columns(pl.col("value").fill_null(0))
df.with_columns(pl.col("value").fill_null(strategy="forward"))  # Forward fill
df.with_columns(pl.col("value").fill_null(strategy="backward")) # Backward fill
df.with_columns(pl.col("value").fill_null(strategy="mean"))     # Mean imputation

# Drop
df.drop_nulls()  # Drop rows with any null
df.drop_nulls(subset=["col1", "col2"])  # Drop if specific columns null
```

## Schema Operations

```python
# View schema
print(df.schema)  # OrderedDict of {col: dtype}
print(df.dtypes)  # List of dtypes

# Rename columns
df = df.rename({"old_name": "new_name"})

# Select columns
df = df.select(["col1", "col2"])

# Drop columns
df = df.drop(["col3", "col4"])

# Reorder columns
df = df.select(["col2", "col1", "col3"])

# Change schema
new_schema = {"a": pl.Int32, "b": pl.Float64}
df = df.cast(new_schema)
```

## Indexing and Selection

```python
# Row indexing
df[0]      # First row (returns dict)
df[0:5]    # First 5 rows (returns DataFrame)
df[-1]     # Last row

# Column selection
df["name"]                 # Series
df[["name", "age"]]        # DataFrame
df.select("name")          # DataFrame
df.select(["name", "age"]) # DataFrame

# Boolean indexing
mask = df["age"] > 30
df.filter(mask)

# Conditional selection
df.filter(
    (pl.col("age") > 30) & (pl.col("city") == "NYC")
)
```

## Iteration

```python
# Iterate rows (as dicts)
for row in df.iter_rows(named=True):
    print(row)  # {'name': 'Alice', 'age': 25, ...}

# Iterate rows (as tuples)
for row in df.iter_rows(named=False):
    print(row)  # ('Alice', 25, ...)

# Iterate columns
for col_name in df.columns:
    series = df[col_name]
    print(f"{col_name}: {series.mean()}")

# Iterate slices (memory efficient)
for batch in df.iter_slices(chunk_size=1000):
    process(batch)  # batch is DataFrame
```

## Conversion

```python
# To Pandas
pdf = df.to_pandas()

# To Arrow
arrow_table = df.to_arrow()

# To NumPy
arr = df.to_numpy()

# To dict
d = df.to_dict()            # {col: [values]}
d = df.to_dict(as_series=False)  # {col: [values]}

# To records
records = df.to_dicts()  # [{'col': val, ...}, ...]

# To CSV string
csv_str = df.write_csv()

# To JSON string
json_str = df.write_json()
```

## Memory and Performance

```python
# Memory usage
df.estimated_size()         # Bytes
df.estimated_size("mb")     # Megabytes

# Clone
df_copy = df.clone()

# Lazy conversion (zero-cost)
lf = df.lazy()

# Rechunk (consolidate chunks)
df = df.rechunk()
```

## Best Practices

1. **Use Lazy API**: Enable query optimization for large datasets
2. **Appropriate Types**: Use smallest types that fit data (Int32 vs Int64)
3. **Categorical for Strings**: Use Categorical for low-cardinality strings
4. **Avoid Iteration**: Use vectorized operations instead
5. **Batch Operations**: Process in chunks for very large datasets
6. **Schema on Read**: Specify dtypes when reading files
7. **Null Strategy**: Decide upfront: drop, fill, or keep nulls
8. **Preview Before Collect**: Use `.head()` on LazyFrame to test queries

## Common Patterns

### Data Loading

```python
# Single file
df = pl.read_parquet("data.parquet")

# Multiple files
df = pl.read_parquet("data/*.parquet")

# Lazy (for large files)
lf = pl.scan_parquet("large.parquet")
result = lf.filter(pl.col("date") == "2024-01-01").collect()
```

### Data Inspection

```python
# Quick overview
print(df.head())
print(df.describe())
print(df.glimpse())

# Check dtypes
print(df.schema)

# Null counts
print(df.null_count())

# Unique values
print(df["category"].n_unique())
```

### Data Cleaning

```python
# Drop nulls
df = df.drop_nulls()

# Fill nulls
df = df.with_columns(pl.col("value").fill_null(0))

# Remove duplicates
df = df.unique()
df = df.unique(subset=["id"])

# Type conversion
df = df.with_columns(pl.col("id").cast(pl.Int64))
```

### Combining DataFrames

```python
# Vertical concat (stack rows)
combined = pl.concat([df1, df2], how="vertical")

# Horizontal concat (side by side)
combined = pl.concat([df1, df2], how="horizontal")

# Diagonal concat (align by column names)
combined = pl.concat([df1, df2], how="diagonal")
```
