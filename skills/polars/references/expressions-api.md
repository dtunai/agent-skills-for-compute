# Polars Expressions API Reference

Sources:
- [Expressions User Guide](https://docs.pola.rs/user-guide/expressions/)
- [API Reference](https://docs.pola.rs/api/python/stable/reference/expressions/)

## Overview

Polars expressions are the primary way to specify computations. They are composable, type-aware, and optimizable by the query planner.

**Key Principles**:
- Method chaining for readability
- Type-safe operations
- Lazy evaluation compatible
- Parallelized automatically
- GPU-acceleratable

## Column Selection

### Basic Selection

```python
import polars as pl

# Single column
pl.col("name")

# Multiple columns
pl.col("a", "b", "c")

# All columns
pl.all()
pl.col("*")

# Exclude columns
pl.exclude("id", "timestamp")
```

### Pattern Matching

```python
# Regex pattern
pl.col("^value_.*$")  # Columns starting with "value_"
pl.col(".*_total$")   # Columns ending with "_total"

# Glob pattern
pl.col("col*")  # col1, col2, col3, etc.

# By dtype
pl.col(pl.Int64)    # All Int64 columns
pl.col(pl.String)   # All String columns
pl.col(pl.Float64, pl.Float32)  # All float columns
```

### Context-Aware Selection

```python
df = pl.DataFrame({
    "id": [1, 2, 3],
    "name": ["a", "b", "c"],
    "value1": [10, 20, 30],
    "value2": [40, 50, 60]
})

# Select with regex
df.select(pl.col("^value.*$"))
# Returns: value1, value2

# Exclude pattern
df.select(pl.exclude("^value.*$"))
# Returns: id, name

# By type
df.select(pl.col(pl.Int64))
# Returns: id, value1, value2
```

## Arithmetic Operations

### Basic Arithmetic

```python
# Addition
pl.col("a") + 10
pl.col("a") + pl.col("b")

# Subtraction
pl.col("a") - 5
pl.col("price") - pl.col("discount")

# Multiplication
pl.col("a") * 2
pl.col("quantity") * pl.col("price")

# Division
pl.col("a") / 3
pl.col("total") / pl.col("count")

# Floor division
pl.col("a") // 2

# Modulo
pl.col("a") % 10

# Power
pl.col("a") ** 2
pl.col("a").pow(3)

# Square root
pl.col("a").sqrt()

# Absolute value
pl.col("a").abs()
```

### Example

```python
df = pl.DataFrame({
    "price": [100, 200, 150],
    "quantity": [2, 1, 3],
    "discount": [10, 20, 5]
})

result = df.select([
    pl.col("price"),
    pl.col("quantity"),
    (pl.col("price") * pl.col("quantity")).alias("subtotal"),
    ((pl.col("price") - pl.col("discount")) * pl.col("quantity")).alias("total")
])
```

## Comparison Operations

```python
# Equal
pl.col("status") == "active"
pl.col("value").eq(100)

# Not equal
pl.col("status") != "inactive"
pl.col("value").ne(0)

# Greater than
pl.col("age") > 30
pl.col("value").gt(100)

# Less than
pl.col("age") < 65
pl.col("value").lt(1000)

# Greater than or equal
pl.col("age") >= 18
pl.col("value").ge(0)

# Less than or equal
pl.col("age") <= 100
pl.col("value").le(200)

# Between
pl.col("value").is_between(10, 100)

# In set
pl.col("status").is_in(["active", "pending"])

# Not in set
pl.col("status").is_in(["inactive", "deleted"]).not_()
```

## Logical Operations

```python
# AND
(pl.col("age") > 30) & (pl.col("city") == "NYC")

# OR
(pl.col("status") == "active") | (pl.col("status") == "pending")

# NOT
pl.col("flag").not_()
~pl.col("flag")

# XOR
pl.col("a").xor(pl.col("b"))

# Complex conditions
(
    (pl.col("age") > 30) &
    (pl.col("salary") > 50000) &
    (pl.col("city").is_in(["NYC", "SF", "LA"]))
)
```

## Null Handling

```python
# Check for nulls
pl.col("value").is_null()
pl.col("value").is_not_null()

# Count nulls
pl.col("value").null_count()

# Fill nulls
pl.col("value").fill_null(0)
pl.col("value").fill_null(strategy="forward")
pl.col("value").fill_null(strategy="backward")
pl.col("value").fill_null(strategy="mean")
pl.col("value").fill_null(strategy="min")
pl.col("value").fill_null(strategy="max")

# Drop nulls
pl.col("value").drop_nulls()

# Replace
pl.col("value").fill_nan(0)  # For float NaN
```

## Aggregations

### Basic Aggregations

```python
# Sum
pl.col("value").sum()

# Mean
pl.col("value").mean()

# Median
pl.col("value").median()

# Standard deviation
pl.col("value").std()
pl.col("value").var()  # Variance

# Min/Max
pl.col("value").min()
pl.col("value").max()

# Count
pl.col("value").count()
pl.col("*").count()  # Count rows

# Unique count
pl.col("category").n_unique()

# First/Last
pl.col("value").first()
pl.col("value").last()

# Quantile
pl.col("value").quantile(0.5)  # Median
pl.col("value").quantile(0.95)
```

### Grouped Aggregations

```python
df = pl.DataFrame({
    "category": ["A", "B", "A", "B", "A"],
    "value": [10, 20, 30, 40, 50]
})

# Single aggregation
result = df.group_by("category").agg(
    pl.col("value").sum()
)

# Multiple aggregations
result = df.group_by("category").agg([
    pl.col("value").sum().alias("total"),
    pl.col("value").mean().alias("average"),
    pl.col("value").count().alias("count"),
    pl.col("value").min().alias("min_value"),
    pl.col("value").max().alias("max_value")
])
```

### Statistical Functions

```python
# Correlation
pl.corr(pl.col("a"), pl.col("b"))

# Covariance
pl.cov(pl.col("a"), pl.col("b"))

# Cumulative sum
pl.col("value").cum_sum()

# Cumulative product
pl.col("value").cum_prod()

# Cumulative min/max
pl.col("value").cum_min()
pl.col("value").cum_max()

# Rank
pl.col("value").rank()
pl.col("value").rank(method="dense")

# Percentile rank
pl.col("value").rank(method="average") / pl.count()
```

## String Operations

### String Namespace

```python
# Case conversion
pl.col("text").str.to_lowercase()
pl.col("text").str.to_uppercase()
pl.col("text").str.to_titlecase()

# Contains pattern
pl.col("text").str.contains("pattern")
pl.col("text").str.contains("^start")  # Regex

# Starts/ends with
pl.col("text").str.starts_with("prefix")
pl.col("text").str.ends_with("suffix")

# Replace
pl.col("text").str.replace("old", "new")
pl.col("text").str.replace_all("old", "new")

# Strip whitespace
pl.col("text").str.strip_chars()
pl.col("text").str.strip_chars_start()
pl.col("text").str.strip_chars_end()

# Slice
pl.col("text").str.slice(0, 5)  # First 5 chars
pl.col("text").str.slice(-3)    # Last 3 chars

# Length
pl.col("text").str.len_chars()
pl.col("text").str.len_bytes()

# Split
pl.col("text").str.split(",")
pl.col("text").str.split_exact(",", 2)

# Concatenate
pl.col("first_name") + " " + pl.col("last_name")
pl.concat_str([pl.col("first"), pl.col("last")], separator=" ")

# Padding
pl.col("text").str.pad_start(10, "0")
pl.col("text").str.pad_end(10, " ")

# Extract
pl.col("text").str.extract("(\\d+)", 1)  # Extract first group
pl.col("text").str.extract_all("\\d+")   # All matches

# Count matches
pl.col("text").str.count_matches("\\d")
```

### Example

```python
df = pl.DataFrame({
    "email": [
        "alice@example.com",
        "bob@test.org",
        "charlie@example.com"
    ]
})

result = df.select([
    pl.col("email"),
    pl.col("email").str.split("@").list.first().alias("username"),
    pl.col("email").str.split("@").list.last().alias("domain"),
    pl.col("email").str.to_lowercase().alias("normalized")
])
```

## Temporal Operations

### Datetime Namespace

```python
# Extract components
pl.col("date").dt.year()
pl.col("date").dt.month()
pl.col("date").dt.day()
pl.col("timestamp").dt.hour()
pl.col("timestamp").dt.minute()
pl.col("timestamp").dt.second()
pl.col("timestamp").dt.millisecond()
pl.col("timestamp").dt.microsecond()
pl.col("timestamp").dt.nanosecond()

# Weekday/week
pl.col("date").dt.weekday()  # 0=Monday
pl.col("date").dt.week()
pl.col("date").dt.day_of_year()

# Quarter
pl.col("date").dt.quarter()

# Truncate
pl.col("timestamp").dt.truncate("1h")
pl.col("timestamp").dt.truncate("1d")
pl.col("timestamp").dt.truncate("1mo")

# Round
pl.col("timestamp").dt.round("15m")

# Add duration
pl.col("date") + pl.duration(days=7)
pl.col("timestamp") + pl.duration(hours=2, minutes=30)

# Difference
pl.col("end_date") - pl.col("start_date")

# Epoch time
pl.col("timestamp").dt.epoch(time_unit="s")  # Seconds since epoch
pl.col("timestamp").dt.epoch(time_unit="ms") # Milliseconds

# Timezone conversion
pl.col("timestamp").dt.replace_time_zone("UTC")
pl.col("timestamp").dt.convert_time_zone("America/New_York")

# Strftime
pl.col("date").dt.strftime("%Y-%m-%d")
pl.col("timestamp").dt.strftime("%Y-%m-%d %H:%M:%S")

# Parse string to datetime
pl.col("date_str").str.strptime(pl.Datetime, format="%Y-%m-%d")
```

### Example

```python
from datetime import datetime

df = pl.DataFrame({
    "timestamp": [
        datetime(2024, 1, 15, 10, 30),
        datetime(2024, 2, 20, 14, 45),
        datetime(2024, 3, 25, 9, 15)
    ]
})

result = df.select([
    pl.col("timestamp"),
    pl.col("timestamp").dt.year().alias("year"),
    pl.col("timestamp").dt.month().alias("month"),
    pl.col("timestamp").dt.day().alias("day"),
    pl.col("timestamp").dt.hour().alias("hour"),
    pl.col("timestamp").dt.truncate("1d").alias("date"),
    pl.col("timestamp").dt.strftime("%Y-%m-%d").alias("formatted")
])
```

## List Operations

### List Namespace

```python
# Get element
pl.col("items").list.get(0)    # First element
pl.col("items").list.get(-1)   # Last element
pl.col("items").list.first()
pl.col("items").list.last()

# Slice
pl.col("items").list.slice(0, 3)  # First 3 elements

# Length
pl.col("items").list.len()

# Contains
pl.col("items").list.contains(10)

# Aggregations on lists
pl.col("items").list.sum()
pl.col("items").list.mean()
pl.col("items").list.min()
pl.col("items").list.max()

# Sort
pl.col("items").list.sort()
pl.col("items").list.sort(descending=True)

# Unique
pl.col("items").list.unique()

# Explode (flatten)
pl.col("items").list.explode()

# Join to string
pl.col("items").list.join(", ")

# Reverse
pl.col("items").list.reverse()

# Concat lists
pl.col("list1").list.concat(pl.col("list2"))
```

### Example

```python
df = pl.DataFrame({
    "id": [1, 2, 3],
    "values": [[1, 2, 3], [4, 5], [6, 7, 8, 9]]
})

result = df.select([
    pl.col("id"),
    pl.col("values"),
    pl.col("values").list.len().alias("count"),
    pl.col("values").list.sum().alias("total"),
    pl.col("values").list.mean().alias("average"),
    pl.col("values").list.first().alias("first"),
    pl.col("values").list.last().alias("last")
])

# Explode to rows
exploded = df.explode("values")
```

## Struct Operations

### Struct Namespace

```python
# Access field
pl.col("person").struct.field("name")
pl.col("person").struct.field("age")

# Rename field
pl.col("person").struct.rename_fields(["full_name", "years"])

# JSON encode
pl.col("person").struct.json_encode()
```

### Example

```python
df = pl.DataFrame({
    "id": [1, 2, 3],
    "person": [
        {"name": "Alice", "age": 30},
        {"name": "Bob", "age": 25},
        {"name": "Charlie", "age": 35}
    ]
})

result = df.select([
    pl.col("id"),
    pl.col("person").struct.field("name").alias("name"),
    pl.col("person").struct.field("age").alias("age")
])

# Or unnest
result = df.unnest("person")
```

## Window Functions

### Over (Partitioned Windows)

```python
df = pl.DataFrame({
    "category": ["A", "A", "B", "B", "A"],
    "value": [10, 20, 30, 40, 50]
})

# Window aggregation
result = df.with_columns([
    pl.col("value").sum().over("category").alias("category_total"),
    pl.col("value").mean().over("category").alias("category_avg"),
    pl.col("value").count().over("category").alias("category_count")
])

# Rank within group
result = df.with_columns(
    pl.col("value").rank().over("category").alias("rank_in_category")
)

# Cumulative within group
result = df.with_columns(
    pl.col("value").cum_sum().over("category").alias("running_total")
)
```

### Rolling Windows

```python
# Rolling mean
pl.col("value").rolling_mean(window_size=3)

# Rolling sum
pl.col("value").rolling_sum(window_size=5)

# Rolling min/max
pl.col("value").rolling_min(window_size=10)
pl.col("value").rolling_max(window_size=10)

# Rolling std
pl.col("value").rolling_std(window_size=20)

# Rolling quantile
pl.col("value").rolling_quantile(quantile=0.5, window_size=100)

# With center
pl.col("value").rolling_mean(window_size=5, center=True)

# With min periods
pl.col("value").rolling_mean(window_size=5, min_periods=3)
```

### Example

```python
import polars as pl

df = pl.DataFrame({
    "time": range(10),
    "value": [10, 15, 13, 17, 20, 18, 22, 25, 23, 27]
})

result = df.with_columns([
    pl.col("value").rolling_mean(window_size=3).alias("ma_3"),
    pl.col("value").rolling_sum(window_size=3).alias("sum_3"),
    pl.col("value").rolling_max(window_size=3).alias("max_3")
])
```

## Conditional Expressions

### When-Then-Otherwise

```python
# Simple condition
pl.when(pl.col("age") >= 18).then("adult").otherwise("minor")

# Multiple conditions
(
    pl.when(pl.col("score") >= 90)
    .then("A")
    .when(pl.col("score") >= 80)
    .then("B")
    .when(pl.col("score") >= 70)
    .then("C")
    .otherwise("F")
)

# With expressions
(
    pl.when(pl.col("value") > 100)
    .then(pl.col("value") * 0.9)  # 10% discount
    .otherwise(pl.col("value"))
)
```

### Example

```python
df = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "age": [25, 17, 30]
})

result = df.with_columns(
    pl.when(pl.col("age") >= 18)
    .then(pl.lit("adult"))
    .otherwise(pl.lit("minor"))
    .alias("status")
)
```

## Type Casting

```python
# Cast to type
pl.col("value").cast(pl.Float64)
pl.col("id").cast(pl.Int32)
pl.col("flag").cast(pl.Boolean)

# Strict casting (error on failure)
pl.col("value").cast(pl.Int64, strict=True)

# Non-strict (null on failure)
pl.col("value").cast(pl.Int64, strict=False)

# Parse datetime
pl.col("date_str").str.strptime(pl.Date, format="%Y-%m-%d")
pl.col("timestamp_str").str.strptime(pl.Datetime, format="%Y-%m-%d %H:%M:%S")
```

## Utility Functions

```python
# Literal value
pl.lit(42)
pl.lit("constant")
pl.lit([1, 2, 3])

# Alias
pl.col("old_name").alias("new_name")

# Exclude
pl.exclude("col1", "col2")

# All columns
pl.all()

# Column count
pl.count()

# Coalesce (first non-null)
pl.coalesce([pl.col("a"), pl.col("b"), pl.lit(0)])

# Concat columns
pl.concat_list([pl.col("a"), pl.col("b")])
pl.concat_str([pl.col("first"), pl.col("last")], separator=" ")

# Format
pl.format("Hello, {}!", pl.col("name"))

# Clip
pl.col("value").clip(0, 100)  # Constrain to range

# Round
pl.col("value").round(2)
pl.col("value").floor()
pl.col("value").ceil()
```

## Best Practices

1. **Use Expressions**: Prefer `pl.col()` over Python lambdas
2. **Chain Methods**: Build complex expressions via chaining
3. **Alias Results**: Name computed columns clearly
4. **Type Safety**: Cast explicitly when needed
5. **Null Handling**: Decide strategy early (fill, drop, keep)
6. **Pattern Matching**: Use regex for flexible column selection
7. **Window Functions**: Avoid manual loops for grouped operations
8. **Combine Conditions**: Use `&`, `|` for compound logic
9. **Test on Subset**: Validate expressions on small data first
10. **Read Explain Plan**: Verify expression optimization

## Common Pitfalls

- **Using Python functions**: Can't be optimized, use Polars expressions
- **Not aliasing**: Confusing auto-generated column names
- **Inefficient conditionals**: Use `when-then` instead of applying lambdas
- **Type mismatches**: Cast before arithmetic operations
- **Forgetting over()**: Window functions need `.over()` for grouping
- **Complex regex**: Simple patterns perform better
- **Not handling nulls**: Null propagation can surprise
- **Eager evaluation in lazy**: Breaking optimization with `.collect()` early
- **Ignoring list operations**: Manually exploding instead of using list namespace
- **Literal comparisons**: Use `pl.lit()` for constants in expressions
