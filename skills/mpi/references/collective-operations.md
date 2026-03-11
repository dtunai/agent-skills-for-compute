# MPI Collective Operations Reference

Sources:
- [mpi4py Tutorial](https://mpi4py.readthedocs.io/en/stable/tutorial.html)
- [mpi4py API Reference](https://mpi4py.readthedocs.io/en/stable/reference/mpi4py.MPI.Comm.html)
- [MPI Standard 4.1](https://www.mpi-forum.org/docs/mpi-4.1/mpi41-report.pdf)
- [Open MPI Documentation](https://docs.open-mpi.org/)

## Overview

Collective operations involve communication among all processes in a communicator. According to the documentation, collective operations are generally more efficient than manually implementing the same functionality with point-to-point operations.

**Key Properties**:
- All processes in communicator must participate
- Operations are blocking (return when operation complete)
- More efficient than equivalent point-to-point loops
- Some operations require data on root process only

## Synchronization

### Barrier

```python
from mpi4py import MPI
import time

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

print(f"Rank {rank} before barrier at {time.time()}")

# All processes wait here
comm.Barrier()

print(f"Rank {rank} after barrier at {time.time()}")
```

**Use Cases**:
- Synchronize before timing
- Ensure all processes reach checkpoint
- Coordinate phases of computation

## Broadcast

### Basic Broadcast

```python
# Broadcast Python object
if rank == 0:
    data = {'key': 'value', 'number': 42, 'array': [1, 2, 3]}
else:
    data = None

# Root sends to all
data = comm.bcast(data, root=0)
print(f"Rank {rank} has: {data}")
```

### NumPy Broadcast

```python
import numpy as np

# Root has data
if rank == 0:
    data = np.arange(100, dtype='float64')
else:
    data = np.empty(100, dtype='float64')

# Broadcast array
comm.Bcast(data, root=0)
print(f"Rank {rank}: {data[:5]}")
```

**Efficiency**: O(log(n)) instead of O(n) for point-to-point loop

## Scatter

### Equal-Sized Chunks

```python
import numpy as np

size = comm.Get_size()
rank = comm.Get_rank()

# Root has all data
if rank == 0:
    data = np.arange(size * 10, dtype='float64')
    print(f"Root scattering: {data}")
else:
    data = None

# Each rank gets equal chunk
recvbuf = np.empty(10, dtype='float64')
comm.Scatter(data, recvbuf, root=0)

print(f"Rank {rank} received: {recvbuf}")
```

### Variable-Sized Chunks (Scatterv)

```python
# Different size chunks
if rank == 0:
    # Varying sizes
    sendcounts = [10, 20, 30, 15]
    displacements = [0, 10, 30, 60]
    sendbuf = np.arange(sum(sendcounts), dtype='float64')
else:
    sendcounts = None
    displacements = None
    sendbuf = None

# Receive appropriate size
recvcount = [10, 20, 30, 15][rank]
recvbuf = np.empty(recvcount, dtype='float64')

comm.Scatterv([sendbuf, sendcounts, displacements, MPI.DOUBLE], recvbuf, root=0)
```

## Gather

### Equal-Sized Gather

```python
# Each rank has local data
sendbuf = np.ones(10, dtype='float64') * rank

# Root gathers all
if rank == 0:
    recvbuf = np.empty(size * 10, dtype='float64')
else:
    recvbuf = None

comm.Gather(sendbuf, recvbuf, root=0)

if rank == 0:
    print(f"Root gathered: {recvbuf[:20]}")
```

### Variable-Sized Gather (Gatherv)

```python
# Each rank has different amount of data
sendcount = (rank + 1) * 10
sendbuf = np.ones(sendcount, dtype='float64') * rank

if rank == 0:
    recvcounts = [(i + 1) * 10 for i in range(size)]
    displacements = [sum(recvcounts[:i]) for i in range(size)]
    recvbuf = np.empty(sum(recvcounts), dtype='float64')
else:
    recvcounts = None
    displacements = None
    recvbuf = None

comm.Gatherv(sendbuf, [recvbuf, recvcounts, displacements, MPI.DOUBLE], root=0)
```

## Allgather

### Gather to All Processes

```python
# Like Gather, but all processes get result
sendbuf = np.ones(10, dtype='float64') * rank
recvbuf = np.empty(size * 10, dtype='float64')

comm.Allgather(sendbuf, recvbuf)

# All ranks have complete data
print(f"Rank {rank} has all data: {recvbuf[:20]}")
```

### Allgatherv

```python
# Variable-sized allgather
sendcount = (rank + 1) * 10
sendbuf = np.ones(sendcount, dtype='float64') * rank

recvcounts = [(i + 1) * 10 for i in range(size)]
displacements = [sum(recvcounts[:i]) for i in range(size)]
recvbuf = np.empty(sum(recvcounts), dtype='float64')

comm.Allgatherv(sendbuf, [recvbuf, recvcounts, displacements, MPI.DOUBLE])
```

## Reduce

### Basic Reduce Operations

```python
# Local value
local_sum = np.array([rank], dtype='int')

# Reduce to root
if rank == 0:
    global_sum = np.empty(1, dtype='int')
else:
    global_sum = None

comm.Reduce(local_sum, global_sum, op=MPI.SUM, root=0)

if rank == 0:
    print(f"Sum of all ranks: {global_sum[0]}")
```

### Available Operations

```python
# Reduction operations
MPI.SUM      # Sum elements
MPI.PROD     # Product of elements
MPI.MAX      # Maximum element
MPI.MIN      # Minimum element
MPI.LAND     # Logical AND
MPI.LOR      # Logical OR
MPI.LXOR     # Logical XOR
MPI.BAND     # Bitwise AND
MPI.BOR      # Bitwise OR
MPI.BXOR     # Bitwise XOR
MPI.MAXLOC   # Maximum and its location
MPI.MINLOC   # Minimum and its location
```

### Vector Reduce

```python
# Reduce arrays
local_data = np.random.randn(100) * rank

if rank == 0:
    global_max = np.empty(100, dtype='float64')
else:
    global_max = None

comm.Reduce(local_data, global_max, op=MPI.MAX, root=0)

if rank == 0:
    print(f"Maximum values: {global_max[:5]}")
```

## Allreduce

### Reduce to All

```python
# All processes get result
local_data = np.ones(100, dtype='float64') * rank
global_sum = np.empty(100, dtype='float64')

comm.Allreduce(local_data, global_sum, op=MPI.SUM)

# All ranks have sum
print(f"Rank {rank} sum: {global_sum[:5]}")
```

### In-Place Allreduce

```python
# Reduce in-place (data overwritten with result)
data = np.ones(100, dtype='float64') * rank

comm.Allreduce(MPI.IN_PLACE, data, op=MPI.SUM)

print(f"Rank {rank} result: {data[:5]}")
```

## Reduce_scatter

### Reduce and Scatter

```python
# Reduce then scatter result
sendbuf = np.arange(size * 10, dtype='float64') * rank
recvbuf = np.empty(10, dtype='float64')
recvcounts = [10] * size

comm.Reduce_scatter(sendbuf, recvbuf, recvcounts, op=MPI.SUM)

print(f"Rank {rank} portion: {recvbuf[:5]}")
```

### Variable Counts

```python
# Different receive counts
sendcount = size * (rank + 1) * 10
sendbuf = np.ones(sendcount, dtype='float64') * rank

recvcounts = [(i + 1) * 10 for i in range(size)]
recvcount = recvcounts[rank]
recvbuf = np.empty(recvcount, dtype='float64')

comm.Reduce_scatter_block(sendbuf, recvbuf, op=MPI.SUM)
```

## Scan (Prefix Operations)

### Inclusive Scan

```python
# Each rank gets sum of all lower ranks + itself
sendbuf = np.array([rank + 1], dtype='int')
recvbuf = np.empty(1, dtype='int')

comm.Scan(sendbuf, recvbuf, op=MPI.SUM)

# Rank 0: 1, Rank 1: 3, Rank 2: 6, Rank 3: 10
print(f"Rank {rank} prefix sum: {recvbuf[0]}")
```

### Exclusive Scan (Exscan)

```python
# Excludes current rank's contribution
sendbuf = np.array([rank + 1], dtype='int')
recvbuf = np.empty(1, dtype='int')

comm.Exscan(sendbuf, recvbuf, op=MPI.SUM)

if rank > 0:
    # Rank 1: 1, Rank 2: 3, Rank 3: 6
    print(f"Rank {rank} exclusive sum: {recvbuf[0]}")
```

## Alltoall

### Complete Exchange

```python
# Each rank sends different data to each rank
sendbuf = np.arange(size * 10, dtype='float64') + rank * 100
recvbuf = np.empty(size * 10, dtype='float64')

comm.Alltoall(sendbuf, recvbuf)

# Rank i receives from all ranks their i-th chunk
print(f"Rank {rank} received: {recvbuf[:15]}")
```

### Alltoallv

```python
# Variable-sized complete exchange
# Each rank sends different amounts to each rank
sendcounts = [(i + 1) * 10 for i in range(size)]
sdispls = [sum(sendcounts[:i]) for i in range(size)]
sendbuf = np.arange(sum(sendcounts), dtype='float64') * rank

recvcounts = [(rank + 1) * 10] * size
rdispls = [i * (rank + 1) * 10 for i in range(size)]
recvbuf = np.empty(sum(recvcounts), dtype='float64')

comm.Alltoallv(
    [sendbuf, sendcounts, sdispls, MPI.DOUBLE],
    [recvbuf, recvcounts, rdispls, MPI.DOUBLE]
)
```

## Custom Reduction Operations

### Defining Custom Op

```python
# Define custom reduction function
def custom_sum(x, y, datatype):
    # Custom logic
    return x * 2 + y

# Create MPI operation
custom_op = MPI.Op.Create(custom_sum, commute=True)

# Use in reduction
local_data = np.array([rank], dtype='int')
result = comm.reduce(local_data, op=custom_op, root=0)

# Free operation
custom_op.Free()
```

### Lambda Operations

```python
# Simple lambda (for Python objects only)
data = rank
result = comm.reduce(data, op=lambda x, y: max(x, y), root=0)
```

## Non-Blocking Collectives

### Immediate Collectives

```python
# Non-blocking broadcast
if rank == 0:
    data = np.arange(100, dtype='float64')
else:
    data = np.empty(100, dtype='float64')

req = comm.Ibcast(data, root=0)

# Do other work
local_computation()

# Wait for completion
req.wait()
```

### Available Non-Blocking Collectives

```python
# All collective operations have non-blocking versions
req_bcast = comm.Ibcast(data, root=0)
req_scatter = comm.Iscatter(sendbuf, recvbuf, root=0)
req_gather = comm.Igather(sendbuf, recvbuf, root=0)
req_allgather = comm.Iallgather(sendbuf, recvbuf)
req_reduce = comm.Ireduce(sendbuf, recvbuf, op=MPI.SUM, root=0)
req_allreduce = comm.Iallreduce(sendbuf, recvbuf, op=MPI.SUM)
req_alltoall = comm.Ialltoall(sendbuf, recvbuf)

# Wait for all
MPI.Request.Waitall([req_bcast, req_scatter, req_gather, req_allgather,
                     req_reduce, req_allreduce, req_alltoall])
```

## Performance Patterns

### Hierarchical Communication

```python
# Split into node-local and inter-node communication
shared_comm = comm.Split_type(MPI.COMM_TYPE_SHARED)
shared_rank = shared_comm.Get_rank()
shared_size = shared_comm.Get_size()

# Reduce within node
local_sum = np.array([rank], dtype='int')
if shared_rank == 0:
    node_sum = np.empty(1, dtype='int')
else:
    node_sum = None

shared_comm.Reduce(local_sum, node_sum, op=MPI.SUM, root=0)

# Node leaders reduce across nodes
if shared_rank == 0:
    # Create inter-node communicator
    color = 0
else:
    color = MPI.UNDEFINED

leader_comm = comm.Split(color, rank)

if color == 0:
    global_sum = np.empty(1, dtype='int')
    leader_comm.Reduce(node_sum, global_sum, op=MPI.SUM, root=0)

    if rank == 0:
        print(f"Global sum: {global_sum[0]}")
```

### Overlapping Collectives

```python
# Overlap multiple non-blocking collectives
data1 = np.ones(100, dtype='float64') * rank
data2 = np.ones(50, dtype='float64') * rank
result1 = np.empty(100, dtype='float64')
result2 = np.empty(50, dtype='float64')

req1 = comm.Iallreduce(data1, result1, op=MPI.SUM)
req2 = comm.Iallreduce(data2, result2, op=MPI.MAX)

# Do independent computation
local_work = expensive_computation()

# Wait for both
MPI.Request.Waitall([req1, req2])
```

## Common Collective Patterns

### Global Statistics

```python
# Compute global min, max, mean
local_data = np.random.randn(1000)

# Min/Max
global_min = np.empty(1, dtype='float64')
global_max = np.empty(1, dtype='float64')
local_min = np.array([local_data.min()])
local_max = np.array([local_data.max()])

comm.Allreduce(local_min, global_min, op=MPI.MIN)
comm.Allreduce(local_max, global_max, op=MPI.MAX)

# Mean
local_sum = np.array([local_data.sum()])
local_count = np.array([len(local_data)])
global_sum = np.empty(1, dtype='float64')
global_count = np.empty(1, dtype='int')

comm.Allreduce(local_sum, global_sum, op=MPI.SUM)
comm.Allreduce(local_count, global_count, op=MPI.SUM)

global_mean = global_sum[0] / global_count[0]

if rank == 0:
    print(f"Global: min={global_min[0]}, max={global_max[0]}, mean={global_mean}")
```

### Matrix Distribution

```python
# Distribute matrix rows
if rank == 0:
    matrix = np.random.randn(100, 50)
else:
    matrix = None

rows_per_rank = 100 // size
local_rows = np.empty((rows_per_rank, 50), dtype='float64')

comm.Scatter(matrix, local_rows, root=0)

# Process local rows
local_result = process_rows(local_rows)

# Gather results
if rank == 0:
    results = np.empty((100, 50), dtype='float64')
else:
    results = None

comm.Gather(local_result, results, root=0)
```

### Ring Sum

```python
# Each rank contributes to running sum
partial_sum = np.array([rank], dtype='int')

for step in range(1, size):
    # Send to right, receive from left
    right = (rank + step) % size
    left = (rank - step + size) % size

    recvbuf = np.empty(1, dtype='int')
    comm.Sendrecv(partial_sum, dest=right, recvbuf=recvbuf, source=left)

    # Add to partial sum
    partial_sum += recvbuf

# Result: sum of all ranks
print(f"Rank {rank} final sum: {partial_sum[0]}")
```

## Debugging Collectives

### Ensuring All Participate

```python
# BAD: Not all ranks participate
if rank % 2 == 0:
    comm.Barrier()  # Only even ranks - DEADLOCK!

# GOOD: All ranks participate
comm.Barrier()
```

### Matching Buffer Sizes

```python
# BAD: Mismatched buffer sizes
if rank == 0:
    data = np.arange(100, dtype='float64')  # Size 100
else:
    data = np.empty(50, dtype='float64')  # Size 50 - ERROR!

comm.Bcast(data, root=0)

# GOOD: Same size on all ranks
if rank == 0:
    data = np.arange(100, dtype='float64')
else:
    data = np.empty(100, dtype='float64')

comm.Bcast(data, root=0)
```

### Logging Collective Operations

```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

logger.debug(f"Rank {rank} entering Bcast")
comm.Bcast(data, root=0)
logger.debug(f"Rank {rank} completed Bcast")
```

## Best Practices

1. **Use collectives over point-to-point**: More efficient, optimized
2. **All ranks must participate**: Collective requires all processes
3. **Match buffer sizes**: Same size across all ranks
4. **Use non-blocking for overlap**: Overlap communication and computation
5. **In-place operations**: Save memory when possible
6. **Hierarchical for large clusters**: Node-local then inter-node
7. **Profile collective operations**: Identify bottlenecks
8. **Use appropriate operation**: Allreduce vs Reduce+Bcast
9. **Consider topology**: Some collectives benefit from specific layouts
10. **Test at small scale**: Verify correctness before scaling

## Common Pitfalls

- **Not all ranks participating**: Leads to deadlock
- **Buffer size mismatch**: Undefined behavior or errors
- **Wrong root rank**: Only root needs valid buffer for some operations
- **Incorrect operation**: Using SUM when PROD needed
- **Forgetting to allocate**: Receive buffers not allocated
- **Type mismatch**: Different datatypes across ranks
- **Using blocking collectives in nested calls**: Potential deadlock
- **Not checking return values**: Silent failures
- **Assuming order**: Results may not preserve order in some operations
- **Over-synchronizing**: Excessive barriers slow down program
