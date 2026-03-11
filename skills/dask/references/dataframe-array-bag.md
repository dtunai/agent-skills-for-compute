# Dask Core Collections Reference

Sources:
- [Dask DataFrame Documentation](https://docs.dask.org/en/stable/dataframe.html)
- [Dask Array Documentation](https://docs.dask.org/en/stable/array.html)
- [Dask Bag Documentation](https://docs.dask.org/en/stable/bag.html)
- [Dask Delayed Documentation](https://docs.dask.org/en/stable/delayed.html)

## Dask DataFrame

### Overview

Dask DataFrame enables parallel processing of large tabular datasets by coordinating multiple pandas DataFrames. As stated in the official documentation: **"Dask DataFrames are a collection of many pandas DataFrames"** with identical API and execution patterns.

### Key Characteristics

- **Scalability**: Functions on datasets ranging from 100 GiB on single machines to 100 TiB across distributed clusters
- **Architecture**: Data is partitioned row-wise along the index, with constituent pandas objects potentially stored on disk or remote machines
- **Familiarity**: The API mirrors pandas, enabling straightforward transitions between libraries

### When to Use Dask DataFrames

Use Dask DataFrame when:
- Datasets exceed available memory
- Computation speed becomes problematic despite optimization attempts
- You need to scale pandas workflows to larger data

### When to Avoid Dask DataFrames

Stick with pandas if:
- Datasets are small (fit in RAM)
- Computations complete in subsecond timeframes
- Simpler acceleration methods exist (built-in pandas functions vs. `.apply()` or loops)

### Creating DataFrames

```python
import dask.dataframe as dd

# Read from files
df = dd.read_csv('data/*.csv')
df = dd.read_parquet('data/*.parquet')
df = dd.read_json('data/*.json', lines=True)
df = dd.read_hdf('data.h5', '/key')
df = dd.read_sql_table('table_name', connection_uri, npartitions=10)

# From pandas
import pandas as pd
pdf = pd.DataFrame({'x': [1, 2, 3], 'y': [4, 5, 6]})
df = dd.from_pandas(pdf, npartitions=2)

# From delayed objects
import dask
dfs = [dask.delayed(pd.read_csv)(f) for f in filenames]
df = dd.from_delayed(dfs)
```

### Common Operations

```python
# Selection
df['column']
df[['col1', 'col2']]
df[df.value > 10]

# Aggregations
df.sum()
df.mean()
df.std()
df.groupby('key').value.mean()
df.groupby(['key1', 'key2']).agg({'value': ['sum', 'mean']})

# Transformations
df['new_col'] = df.col1 + df.col2
df = df.drop(columns=['unwanted'])
df = df.fillna(0)
df = df.dropna()

# Merges and joins
result = dd.merge(df1, df2, on='key')
result = df1.join(df2, on='key')

# Sorting (expensive)
df = df.set_index('timestamp')  # Set index first
df = df.sort_values('column')  # Triggers shuffle
```

### Computation Model

```python
# Lazy evaluation - builds task graph
result = df.groupby('key').value.mean()

# Execute with .compute()
final = result.compute()

# Persist in distributed memory
df = df.persist()

# Write to disk
df.to_parquet('output/', compression='snappy')
df.to_csv('output/*.csv')
```

### Partitioning

```python
# Check current partitions
df.npartitions

# Repartition
df = df.repartition(npartitions=100)
df = df.repartition(partition_size='100MB')

# Set index (creates sorted partitions)
df = df.set_index('timestamp')
df = df.set_index('user_id', npartitions=50)

# Reset index
df = df.reset_index()
```

### Indexing

```python
# Set index for efficient queries
df = df.set_index('timestamp')

# Loc-based selection (fast with sorted index)
subset = df.loc['2026-01-01':'2026-01-31']

# Iloc not supported
# df.iloc[100:200]  # Not supported

# Boolean indexing
df[df.value > threshold]
```

### Apply and Map

```python
# Map partitions (apply function to each partition)
def process_partition(pdf):
    # pdf is pandas DataFrame
    return pdf.groupby('key').sum()

result = df.map_partitions(process_partition)

# Apply (row-wise, slow)
df['new'] = df.apply(lambda row: row.x + row.y, axis=1, meta=('new', 'f8'))

# Map (column-wise, fast)
df['doubled'] = df.value.map(lambda x: x * 2, meta=('doubled', 'f8'))
```

### Best Practices

1. **Use built-in operations**: Faster than apply
2. **Set index for queries**: Enables efficient filtering
3. **Avoid shuffles**: Sorts and set_index trigger expensive shuffles
4. **Use Parquet**: Better performance than CSV
5. **Specify dtypes**: Prevents schema inference overhead
6. **Persist reused data**: Use `.persist()` for intermediate results
7. **Avoid large results**: Filter/aggregate before `.compute()`

## Dask Array

### Overview

Dask Array implements a subset of NumPy's interface using blocked algorithms. As stated in the documentation: **"Dask Array implements a subset of the NumPy ndarray interface using blocked algorithms, cutting up the large array into many small arrays."**

Dask coordinates many NumPy arrays (or compatible "duck arrays" like CuPy or Sparse arrays) arranged in a grid.

### Creating Arrays

```python
import dask.array as da
import numpy as np

# Random arrays
x = da.random.random((10000, 10000), chunks=(1000, 1000))
x = da.random.normal(0, 1, size=(5000, 5000), chunks=(500, 500))

# From NumPy
arr = np.ones((10000, 10000))
x = da.from_array(arr, chunks=(1000, 1000))

# From delayed
import dask
arrays = [dask.delayed(np.load)(f) for f in filenames]
x = da.from_delayed(arrays[0], shape=(1000, 1000), dtype=float)
x = da.stack([da.from_delayed(a, shape=(1000, 1000), dtype=float)
              for a in arrays])

# From Zarr
x = da.from_zarr('data.zarr')

# Ones, zeros, full
x = da.ones((5000, 5000), chunks=(500, 500))
x = da.zeros((5000, 5000), chunks=(500, 500))
x = da.full((5000, 5000), 42, chunks=(500, 500))
```

### Supported Operations

```python
# Arithmetic
y = x + 10
y = x * 2
y = x ** 2
y = da.exp(x)
y = da.log(x)

# Reductions
x.sum()
x.mean()
x.std()
x.sum(axis=0)  # Along axis
x.max(axis=1)

# Linear algebra
y = x @ x.T  # Matrix multiplication
y = x.dot(x.T)
u, s, v = da.linalg.svd(x)
q, r = da.linalg.qr(x)

# Slicing
y = x[100:200, 300:400]
y = x[::2, ::2]  # Every other element

# Transpose and reshape
y = x.T
y = x.transpose()
y = x.reshape((5000, 20000))

# Stacking
y = da.stack([x1, x2, x3])
y = da.concatenate([x1, x2], axis=0)
```

### Chunking

```python
# Create with specific chunks
x = da.random.random((10000, 10000), chunks=(1000, 1000))

# Rechunk
x = x.rechunk((2000, 500))

# Auto-rechunk
x = x.rechunk('auto')

# Check chunks
x.chunks  # ((1000, 1000, ...), (1000, 1000, ...))
x.numblocks  # (10, 10)

# Chunk size guidelines:
# - 100MB - 1GB per chunk optimal
# - Match access patterns (row-major vs column-major)
```

### Notable Limitations

The documentation explicitly identifies what Dask Array does **not** support:

- Most of `np.linalg` functions remain unimplemented
- Operations with unknown shapes have restricted capabilities
- Parallel-unfriendly operations like full sorting
- Inefficient operations such as `tolist()` or iteration loops
- Many lesser-used NumPy functions

### FFT and Signal Processing

```python
# FFT
y = da.fft.fft(x)
y = da.fft.fft2(x)
y = da.fft.rfft(x)

# Inverse FFT
x_reconstructed = da.fft.ifft(y)
```

### Random Number Generation

```python
# Create RandomState
rs = da.random.RandomState(42)

# Generate random data
x = rs.normal(0, 1, size=(10000, 10000), chunks=(1000, 1000))
x = rs.uniform(0, 1, size=(5000, 5000), chunks=(500, 500))
x = rs.binomial(10, 0.5, size=(10000,), chunks=(1000,))
```

### Storage

```python
# To Zarr
da.to_zarr(x, 'output.zarr')

# To HDF5
da.to_hdf5('output.h5', '/dataset', x)

# To NumPy (compute)
result = x.compute()
```

### Best Practices

1. **Choose chunk size wisely**: 100MB-1GB per chunk
2. **Rechunk for access patterns**: Match computation requirements
3. **Use built-in operations**: Avoid element-wise apply
4. **Leverage GPU arrays**: CuPy for GPU acceleration
5. **Store intermediate results**: Use Zarr for large arrays
6. **Profile before distributing**: Ensure NumPy optimization first

## Dask Bag

### Overview

Dask Bag implements operations on unstructured or semi-structured data using Python sequences. Best for processing log files, JSON records, or any data that doesn't fit DataFrame or Array models.

### Creating Bags

```python
import dask.bag as db

# From text files
b = db.read_text('logs/*.txt')
b = db.read_text('data/*.json').map(json.loads)

# From sequences
b = db.from_sequence([1, 2, 3, 4, 5], npartitions=2)

# From delayed
import dask
objs = [dask.delayed(load_data)(f) for f in filenames]
b = db.from_delayed(objs)
```

### Common Operations

```python
# Map
b = b.map(lambda x: x.upper())
b = b.map(str.split)

# Filter
b = b.filter(lambda x: len(x) > 5)

# Flatten
b = b.flatten()  # [[1,2], [3,4]] -> [1,2,3,4]

# Fold/reduce
total = b.fold(lambda acc, x: acc + x, initial=0).compute()
total = b.sum().compute()

# Frequencies
counts = b.frequencies().compute()

# Top k
top = b.topk(10, key=lambda x: x[1]).compute()

# GroupBy
grouped = b.groupby(lambda x: x['category'])

# Pluck
records = [{'name': 'Alice', 'age': 30}, {'name': 'Bob', 'age': 25}]
b = db.from_sequence(records, npartitions=2)
names = b.pluck('name').compute()  # ['Alice', 'Bob']
```

### Conversion

```python
# To DataFrame
b = db.read_text('data/*.json').map(json.loads)
df = b.to_dataframe()

# To delayed
delayed_objs = b.to_delayed()
```

### Best Practices

1. **Use for semi-structured data**: Logs, JSON, variable-length records
2. **Convert to DataFrame when possible**: Better performance for tabular data
3. **Avoid large individual items**: Bag assumes small items
4. **Use built-in methods**: map, filter, fold instead of custom logic

## Dask Delayed

### Overview

Dask Delayed enables custom parallelism by wrapping Python functions to create lazy task graphs. Use when DataFrame/Array/Bag don't fit your use case.

### Basic Usage

```python
from dask import delayed

# Wrap functions
@delayed
def load_data(filename):
    return pd.read_csv(filename)

@delayed
def clean_data(df):
    return df.dropna()

@delayed
def analyze_data(df):
    return df.groupby('key').sum()

# Build task graph
data = load_data('input.csv')
cleaned = clean_data(data)
result = analyze_data(cleaned)

# Execute
final = result.compute()
```

### Combining Delayed Objects

```python
# Process multiple files
files = ['data1.csv', 'data2.csv', 'data3.csv']
loaded = [delayed(pd.read_csv)(f) for f in files]
cleaned = [delayed(lambda df: df.dropna())(df) for df in loaded]

# Combine results
combined = delayed(pd.concat)(cleaned)
result = combined.compute()
```

### Converting to Collections

```python
# To DataFrame
dfs = [delayed(pd.read_csv)(f) for f in files]
df = dd.from_delayed(dfs)

# To Array
arrays = [delayed(np.load)(f) for f in files]
x = da.from_delayed(arrays[0], shape=(1000, 1000), dtype=float)

# To Bag
objs = [delayed(load_object)(f) for f in files]
b = db.from_delayed(objs)
```

### Best Practices

1. **Use for custom workflows**: When collections don't fit
2. **Minimize delayed objects**: Large graphs have overhead
3. **Convert to collections**: Better performance when possible
4. **Visualize task graph**: Use `.visualize()` for debugging
5. **Avoid nesting**: Don't call delayed functions from within delayed functions

### Task Graph Visualization

```python
# Visualize task graph
result.visualize(filename='task_graph.png')

# Requires graphviz:
# pip install graphviz
```

## Performance Comparison

| Collection | Best For | Parallel Primitive | Memory Model |
|------------|----------|-------------------|--------------|
| DataFrame | Tabular data | Partition-wise ops | Out-of-core |
| Array | Numeric arrays | Blocked algorithms | Out-of-core |
| Bag | Semi-structured | Map-reduce | In-memory items |
| Delayed | Custom workflows | Task graphs | User-defined |

## Common Pitfalls

- **Not chunking appropriately**: Too small = overhead, too large = memory issues
- **Using apply on DataFrames**: Slow, use built-in operations
- **Returning large results**: Use `.to_parquet()` instead of `.compute()`
- **Ignoring partitions**: Repartition for better parallelism
- **Not persisting**: Recomputation when reusing intermediate results
- **Mixing collections unnecessarily**: Stick to one when possible
- **Creating huge task graphs**: Use collections instead of many delayed objects
- **Not setting meta**: DataFrame operations need meta parameter
- **Assuming sorted data**: Set index explicitly for sorted operations
- **Ignoring chunk alignment**: Misaligned chunks cause shuffles
