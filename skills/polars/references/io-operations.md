# Polars I/O Operations Reference

Sources:
- [Polars I/O Documentation](https://docs.pola.rs/user-guide/io/)
- [API Reference](https://docs.pola.rs/api/python/stable/)

## Overview

Polars supports comprehensive I/O operations across multiple file formats, databases, and cloud storage. Operations come in two modes: **eager** (`read_*`) and **lazy** (`scan_*`).

**Eager vs Lazy:**
- `read_*`: Loads data immediately into memory
- `scan_*`: Creates lazy query plan, enables optimizations (preferred)

## CSV Files

### Reading CSV

```python
import polars as pl

# Eager read
df = pl.read_csv("data.csv")

# Lazy scan (recommended)
lf = pl.scan_csv("data.csv")
df = lf.collect()

# GPU execution
df = pl.scan_csv("data.csv").collect(engine="gpu")
```

### CSV Options

```python
# Custom delimiter
df = pl.read_csv("data.tsv", separator="\\t")

# Skip rows
df = pl.read_csv("data.csv", skip_rows=2)

# No header
df = pl.read_csv("data.csv", has_header=False)

# Custom column names
df = pl.read_csv(
    "data.csv",
    has_header=False,
    new_columns=["col1", "col2", "col3"]
)

# Select columns
df = pl.read_csv("data.csv", columns=["name", "age"])

# Specify dtypes
df = pl.read_csv(
    "data.csv",
    dtypes={"id": pl.Int64, "value": pl.Float32}
)

# Schema override
df = pl.read_csv(
    "data.csv",
    schema={"id": pl.Int64, "name": pl.String}
)

# Null values
df = pl.read_csv("data.csv", null_values=["NA", "NULL", ""])

# Comment lines
df = pl.read_csv("data.csv", comment_char="#")

# Quote character
df = pl.read_csv("data.csv", quote_char='"')

# Encoding
df = pl.read_csv("data.csv", encoding="utf8-lossy")

# Limit rows
df = pl.read_csv("data.csv", n_rows=1000)

# Batch size
df = pl.read_csv("data.csv", batch_size=10000)
```

### Writing CSV

```python
# Write DataFrame
df.write_csv("output.csv")

# Custom separator
df.write_csv("output.tsv", separator="\\t")

# Include header
df.write_csv("output.csv", include_header=True)

# No header
df.write_csv("output.csv", include_header=False)

# Quote style
df.write_csv("output.csv", quote_style="necessary")  # necessary, always, non_numeric

# Null representation
df.write_csv("output.csv", null_value="NULL")

# Float precision
df.write_csv("output.csv", float_precision=4)
```

### Multiple CSV Files

```python
# Read multiple files (eager)
df = pl.read_csv("data/*.csv")

# Read multiple files (lazy)
lf = pl.scan_csv("data/*.csv")

# Specific pattern
df = pl.read_csv("logs/2024-*.csv")

# With glob
import glob
files = glob.glob("data/**/*.csv", recursive=True)
lf = pl.scan_csv(files)
```

## Parquet Files

### Reading Parquet

```python
# Eager read
df = pl.read_parquet("data.parquet")

# Lazy scan (recommended)
lf = pl.scan_parquet("data.parquet")
df = lf.collect()

# GPU execution
df = pl.scan_parquet("data.parquet").collect(engine="gpu")
```

### Parquet Options

```python
# Select columns
df = pl.read_parquet("data.parquet", columns=["id", "name"])

# Row count
df = pl.read_parquet("data.parquet", n_rows=1000)

# Parallel reading
df = pl.read_parquet("data.parquet", parallel="auto")

# Memory map
df = pl.read_parquet("data.parquet", use_pyarrow=True, memory_map=True)

# Rechunk
df = pl.read_parquet("data.parquet", rechunk=True)
```

### Writing Parquet

```python
# Write DataFrame
df.write_parquet("output.parquet")

# Compression
df.write_parquet("output.parquet", compression="snappy")  # snappy, gzip, lz4, zstd

# Compression level
df.write_parquet("output.parquet", compression="zstd", compression_level=9)

# Statistics
df.write_parquet("output.parquet", statistics=True)

# Row group size
df.write_parquet("output.parquet", row_group_size=100000)
```

### Parquet Partitioning

```python
# Write partitioned dataset
df.write_parquet(
    "output",
    partition_by=["year", "month"]
)
# Creates: output/year=2024/month=01/data.parquet

# Read partitioned dataset
lf = pl.scan_parquet("output/**/*.parquet")
```

### Multiple Parquet Files

```python
# Read multiple files
df = pl.read_parquet("data/*.parquet")
lf = pl.scan_parquet("data/*.parquet")

# Specific pattern
df = pl.read_parquet("warehouse/year=2024/month=*/day=*/*.parquet")

# Hive partitioning
lf = pl.scan_parquet("data/year=*/month=*/*.parquet", hive_partitioning=True)
# Automatically adds 'year' and 'month' columns
```

## JSON Files

### Reading JSON

```python
# Read JSON array
df = pl.read_json("data.json")

# Read NDJSON (newline-delimited)
df = pl.read_ndjson("data.ndjson")

# Lazy scan NDJSON
lf = pl.scan_ndjson("data.ndjson")

# Schema inference
df = pl.read_ndjson("data.ndjson", infer_schema_length=1000)
```

### Writing JSON

```python
# Write JSON array
df.write_json("output.json")

# Write NDJSON (recommended)
df.write_ndjson("output.ndjson")

# Pretty print
df.write_json("output.json", pretty=True)

# Row-oriented
df.write_json("output.json", row_oriented=True)
```

### JSON Options

```python
# Read with schema
df = pl.read_ndjson(
    "data.ndjson",
    schema={"id": pl.Int64, "value": pl.Float64}
)

# Batch size
df = pl.read_ndjson("data.ndjson", batch_size=10000)
```

## Excel Files

### Reading Excel

```python
# Read Excel sheet
df = pl.read_excel("data.xlsx")

# Specific sheet
df = pl.read_excel("data.xlsx", sheet_name="Sheet2")

# Sheet by index
df = pl.read_excel("data.xlsx", sheet_id=2)

# Select columns
df = pl.read_excel("data.xlsx", columns=["A", "B", "C"])

# Skip rows
df = pl.read_excel("data.xlsx", read_options={"skip_rows": 2})
```

### Writing Excel

```python
# Write to Excel
df.write_excel("output.xlsx")

# Multiple sheets
with pl.ExcelWriter("output.xlsx") as writer:
    df1.write_excel(writer, worksheet="Sales")
    df2.write_excel(writer, worksheet="Inventory")
```

## Databases

### Reading from SQL

```python
import polars as pl
from sqlalchemy import create_engine

# Create connection
engine = create_engine("postgresql://user:pass@localhost:5432/db")

# Read SQL query
df = pl.read_database("SELECT * FROM users WHERE age > 30", engine)

# Read table
df = pl.read_database("users", engine)

# Using connection string
df = pl.read_database(
    "SELECT * FROM sales",
    "postgresql://user:pass@localhost/db"
)
```

### Writing to SQL

```python
# Write to database
df.write_database(
    table_name="new_table",
    connection=engine
)

# If exists strategy
df.write_database(
    table_name="users",
    connection=engine,
    if_table_exists="replace"  # fail, replace, append
)
```

### Database Engines

```python
# PostgreSQL
engine = create_engine("postgresql://user:pass@host:5432/db")

# MySQL
engine = create_engine("mysql+pymysql://user:pass@host:3306/db")

# SQLite
engine = create_engine("sqlite:///database.db")

# DuckDB (recommended for analytics)
import duckdb
conn = duckdb.connect("database.duckdb")
df = pl.read_database("SELECT * FROM table", conn)
```

## Cloud Storage

### AWS S3

```python
# Read from S3 (Parquet)
df = pl.read_parquet("s3://bucket/path/data.parquet")

# With credentials
import os
os.environ["AWS_ACCESS_KEY_ID"] = "your_key"
os.environ["AWS_SECRET_ACCESS_KEY"] = "your_secret"
os.environ["AWS_REGION"] = "us-west-2"

lf = pl.scan_parquet("s3://bucket/data/*.parquet")

# Write to S3
df.write_parquet("s3://bucket/output/data.parquet")
```

### Google Cloud Storage

```python
# Read from GCS
df = pl.read_parquet("gs://bucket/data.parquet")

# Set credentials
os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "/path/to/credentials.json"

lf = pl.scan_parquet("gs://bucket/data/*.parquet")
```

### Azure Blob Storage

```python
# Read from Azure
df = pl.read_parquet("az://container/data.parquet")

# Set credentials
os.environ["AZURE_STORAGE_ACCOUNT_NAME"] = "account"
os.environ["AZURE_STORAGE_ACCOUNT_KEY"] = "key"
```

### S3-Compatible Storage

```python
# MinIO, Wasabi, etc.
import pyarrow.fs as fs

s3 = fs.S3FileSystem(
    endpoint_override="https://s3.example.com",
    access_key="key",
    secret_key="secret"
)

df = pl.read_parquet("s3://bucket/data.parquet", storage_options={"filesystem": s3})
```

## Delta Lake

### Reading Delta Tables

```python
from deltalake import DeltaTable

# Read Delta table
dt = DeltaTable("path/to/delta/table")
df = pl.from_arrow(dt.to_pyarrow_table())

# With version
dt = DeltaTable("path/to/delta/table", version=5)
df = pl.from_arrow(dt.to_pyarrow_table())

# With timestamp
from datetime import datetime
dt = DeltaTable("path/to/delta/table", version=datetime(2024, 1, 1))
df = pl.from_arrow(dt.to_pyarrow_table())
```

### Writing Delta Tables

```python
# Write Delta table
df.to_arrow().to_delta("path/to/delta/table")

# Append mode
df.to_arrow().to_delta("path/to/delta/table", mode="append")
```

## Apache Arrow

### Reading Arrow

```python
import pyarrow as pa

# From Arrow table
arrow_table = pa.table({"a": [1, 2, 3], "b": [4, 5, 6]})
df = pl.from_arrow(arrow_table)

# From Arrow IPC/Feather
df = pl.read_ipc("data.arrow")
lf = pl.scan_ipc("data.arrow")

# From Arrow stream
with pa.ipc.open_stream("data.arrows") as reader:
    df = pl.from_arrow(reader.read_all())
```

### Writing Arrow

```python
# To Arrow table
arrow_table = df.to_arrow()

# Write IPC/Feather
df.write_ipc("output.arrow")

# Compressed
df.write_ipc("output.arrow", compression="zstd")
```

## Hugging Face Datasets

```python
from datasets import load_dataset

# Load HuggingFace dataset
hf_dataset = load_dataset("squad", split="train")

# Convert to Polars
df = pl.from_arrow(hf_dataset.data.table)

# Or directly
df = pl.DataFrame(hf_dataset)
```

## Streaming I/O

### Chunked Reading

```python
# Read in batches
reader = pl.read_csv_batched("large.csv", batch_size=100000)

for batch in reader:
    # Process batch (batch is DataFrame)
    result = batch.filter(pl.col("value") > 100)
    result.write_parquet("output_batch.parquet", append=True)
```

### Streaming Writes

```python
# Append to existing file
df1.write_parquet("output.parquet")
df2.write_parquet("output.parquet", append=True)
df3.write_parquet("output.parquet", append=True)
```

## Common Patterns

### ETL Pipeline

```python
# Extract (lazy scan)
lf = pl.scan_parquet("raw/sales_*.parquet")

# Transform
transformed = (
    lf
    .filter(pl.col("status") == "completed")
    .with_columns([
        (pl.col("price") * pl.col("quantity")).alias("total"),
        pl.col("timestamp").dt.truncate("1d").alias("date")
    ])
    .group_by(["date", "region"])
    .agg([
        pl.col("total").sum().alias("revenue"),
        pl.col("order_id").n_unique().alias("orders")
    ])
)

# Load
transformed.collect().write_parquet("output/daily_sales.parquet")
```

### Incremental Processing

```python
from datetime import date, timedelta

# Process data by date
start_date = date(2024, 1, 1)
end_date = date(2024, 12, 31)

current = start_date
while current <= end_date:
    date_str = current.strftime("%Y-%m-%d")

    # Read date partition
    lf = pl.scan_parquet(f"raw/date={date_str}/*.parquet")

    # Process
    result = lf.group_by("category").agg(
        pl.col("value").sum()
    ).collect()

    # Write result
    result.write_parquet(f"output/date={date_str}/aggregated.parquet")

    current += timedelta(days=1)
```

### Multi-Format Pipeline

```python
# Read CSV
lf_csv = pl.scan_csv("source.csv")

# Join with Parquet
lf_parquet = pl.scan_parquet("reference.parquet")

# Join
joined = lf_csv.join(lf_parquet, on="id", how="left")

# Write as JSON
joined.collect().write_ndjson("output.ndjson")
```

### Cloud to Local

```python
# Download from S3, process, write locally
lf = pl.scan_parquet("s3://bucket/data/*.parquet")

result = (
    lf
    .filter(pl.col("region") == "us-west-2")
    .group_by("category")
    .agg(pl.col("value").sum())
    .collect()
)

result.write_parquet("local_output.parquet")
```

## Performance Tips

1. **Use Lazy API**: `scan_*` enables predicate/projection pushdown
2. **Parquet Over CSV**: Columnar format, compressed, type-aware
3. **Select Columns Early**: Read only needed columns
4. **Filter During Read**: Pushdown filters to file scan
5. **Appropriate Compression**: `snappy` for speed, `zstd` for size
6. **Batch Processing**: Use batched readers for large files
7. **Rechunk After Read**: Consolidate memory chunks
8. **Use Arrow Format**: Zero-copy interchange
9. **Cloud Credentials**: Set env vars for authenticated access
10. **Parallel Reading**: Parquet supports parallel column reads

## Common Pitfalls

- **Using read_* for large files**: Misses optimizations, use scan_*
- **Reading all columns**: Slow, select only needed columns
- **CSV for large data**: Use Parquet instead
- **Not specifying dtypes**: Inference can be slow or wrong
- **Collecting too early**: Breaks lazy optimization chain
- **Single large file**: Partition data for parallel processing
- **Uncompressed files**: Waste space and I/O bandwidth
- **Missing cloud credentials**: Set env vars before reading
- **Eager Excel reads**: Can be slow for large sheets
- **Not using rechunk**: Fragmented memory chunks hurt performance
