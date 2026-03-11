---
name: mpi
description: Provides MPI (Message Passing Interface) for parallel and distributed computing. Covers point-to-point communication, collective operations, process management, Open MPI, MPICH, and mpi4py Python bindings for HPC applications.
license: MIT
metadata:
  author: Agent Cluster
  tags: mpi, parallel-computing, distributed-computing, hpc, message-passing, open-mpi, mpich, mpi4py, slurm-integration
---

# MPI Agent Skill

Agent-optimized skill for MPI (Message Passing Interface) parallel computing.

## Quick Reference

### mpi4py - Python Bindings

```python
from mpi4py import MPI

# Initialize
comm = MPI.COMM_WORLD
rank = comm.Get_rank()
size = comm.Get_size()

# Point-to-point (Python objects)
if rank == 0:
    data = {'key': 'value', 'number': 42}
    comm.send(data, dest=1, tag=11)
elif rank == 1:
    data = comm.recv(source=0, tag=11)

# NumPy arrays (fast)
import numpy as np
if rank == 0:
    data = np.arange(100, dtype='float64')
    comm.Send(data, dest=1, tag=13)
elif rank == 1:
    data = np.empty(100, dtype='float64')
    comm.Recv(data, source=0, tag=13)

# Collective operations
data = np.arange(10, dtype='float64') * rank
total = np.empty(10, dtype='float64')
comm.Allreduce(data, total, op=MPI.SUM)

# Broadcast
if rank == 0:
    data = {'a': 1, 'b': 2}
else:
    data = None
data = comm.bcast(data, root=0)
```

### Launching MPI Programs

```bash
# Basic execution
mpirun -n 4 python script.py
mpiexec -np 8 ./mpi_program

# Hostfile
mpirun -np 16 --hostfile hosts.txt python script.py

# Process mapping
mpirun -np 8 --map-by node python script.py  # 1 per node
mpirun -np 8 --map-by core python script.py  # 1 per core
mpirun -np 8 --bind-to core python script.py  # Bind to cores

# SLURM integration
srun -n 16 python mpi_script.py
```

## Common Patterns

### Point-to-Point Communication

```python
from mpi4py import MPI
import numpy as np

comm = MPI.COMM_WORLD
rank = comm.Get_rank()
size = comm.Get_size()

# Send/Recv (blocking)
if rank == 0:
    data = np.random.randn(100)
    comm.Send(data, dest=1, tag=0)
elif rank == 1:
    data = np.empty(100)
    comm.Recv(data, source=0, tag=0)

# Isend/Irecv (non-blocking)
if rank == 0:
    data = np.random.randn(100)
    req = comm.Isend(data, dest=1, tag=0)
    req.wait()
elif rank == 1:
    data = np.empty(100)
    req = comm.Irecv(data, source=0, tag=0)
    req.wait()

# Sendrecv (exchange)
if rank == 0:
    send_data = np.ones(10)
    recv_data = np.empty(10)
    comm.Sendrecv(send_data, dest=1, recvbuf=recv_data, source=1)
```

### Collective Operations

```python
# Broadcast
if rank == 0:
    data = np.arange(100, dtype='float64')
else:
    data = np.empty(100, dtype='float64')
comm.Bcast(data, root=0)

# Scatter
if rank == 0:
    data = np.arange(100, dtype='float64')
else:
    data = None
local_data = np.empty(100 // size, dtype='float64')
comm.Scatter(data, local_data, root=0)

# Gather
local_data = np.arange(10, dtype='float64') * rank
if rank == 0:
    global_data = np.empty(10 * size, dtype='float64')
else:
    global_data = None
comm.Gather(local_data, global_data, root=0)

# Allgather
local_data = np.arange(10, dtype='float64') * rank
global_data = np.empty(10 * size, dtype='float64')
comm.Allgather(local_data, global_data)

# Reduce
local_sum = np.array([rank], dtype='int')
global_sum = np.empty(1, dtype='int')
comm.Reduce(local_sum, global_sum, op=MPI.SUM, root=0)

# Allreduce
local_data = np.ones(10) * rank
global_data = np.empty(10)
comm.Allreduce(local_data, global_data, op=MPI.SUM)
```

### Reduction Operations

```python
# Available operations
MPI.SUM      # Sum
MPI.PROD     # Product
MPI.MAX      # Maximum
MPI.MIN      # Minimum
MPI.LAND     # Logical AND
MPI.LOR      # Logical OR
MPI.BAND     # Bitwise AND
MPI.BOR      # Bitwise OR
MPI.MAXLOC   # Max and location
MPI.MINLOC   # Min and location

# Custom reduction
def custom_op(x, y, datatype):
    return x * 2 + y

op = MPI.Op.Create(custom_op, commute=True)
result = comm.reduce(data, op=op, root=0)
op.Free()
```

## Process Topology

### Cartesian Topology

```python
# Create 2D grid
dims = MPI.Compute_dims(size, [0, 0])  # Auto-determine dims
cart_comm = comm.Create_cart(dims, periods=[False, False])

# Get coordinates
coords = cart_comm.Get_coords(rank)

# Get neighbors
left, right = cart_comm.Shift(0, 1)
up, down = cart_comm.Shift(1, 1)

# Exchange with neighbors
send_data = np.ones(10) * rank
recv_data = np.empty(10)
cart_comm.Sendrecv(send_data, dest=right, recvbuf=recv_data, source=left)
```

### Communicator Splitting

```python
# Split by color
color = rank // 2
new_comm = comm.Split(color, key=rank)

# Split by type
new_comm = comm.Split_type(MPI.COMM_TYPE_SHARED)
```

## Parallel I/O

### MPI-IO

```python
from mpi4py import MPI
import numpy as np

comm = MPI.COMM_WORLD
rank = comm.Get_rank()
size = comm.Get_size()

# Write parallel file
amode = MPI.MODE_WRONLY | MPI.MODE_CREATE
fh = MPI.File.Open(comm, "output.dat", amode)

# Each rank writes its chunk
offset = rank * 100 * 8  # 100 doubles per rank
data = np.arange(100, dtype='float64') * rank
fh.Write_at(offset, data)
fh.Close()

# Read parallel file
amode = MPI.MODE_RDONLY
fh = MPI.File.Open(comm, "output.dat", amode)
data = np.empty(100, dtype='float64')
fh.Read_at(offset, data)
fh.Close()
```

### Collective I/O

```python
# Collective write
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Define file view
offset = rank * 100 * 8
fh.Set_view(offset, MPI.DOUBLE)

# Collective write
data = np.arange(100, dtype='float64') * rank
fh.Write_all(data)
fh.Close()
```

## Performance Patterns

### Non-Blocking Communication

```python
# Overlap communication and computation
requests = []

# Post receives
for src in range(size):
    if src != rank:
        buf = np.empty(100, dtype='float64')
        req = comm.Irecv(buf, source=src, tag=0)
        requests.append((req, buf))

# Post sends
send_data = np.ones(100, dtype='float64') * rank
for dest in range(size):
    if dest != rank:
        req = comm.Isend(send_data, dest=dest, tag=0)
        requests.append((req, None))

# Do computation while communication happens
local_result = expensive_computation()

# Wait for communication
for req, buf in requests:
    req.wait()
```

### Buffering

```python
# Buffered send (immediate return)
data = np.random.randn(1000)
comm.Bsend(data, dest=1, tag=0)

# Attach buffer for Bsend
buffer_size = 10000 * MPI.DOUBLE.size + MPI.BSEND_OVERHEAD
buffer = bytearray(buffer_size)
MPI.Attach_buffer(buffer)

# Use Bsend...

# Detach buffer
MPI.Detach_buffer()
```

## GPU-Aware MPI

### CUDA Integration

```python
# With cupy
import cupy as cp
from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

# GPU array
gpu_data = cp.arange(100, dtype='float64') * rank

# Direct GPU-to-GPU transfer (if MPI is CUDA-aware)
if rank == 0:
    comm.Send(gpu_data, dest=1, tag=0)
elif rank == 1:
    recv_data = cp.empty(100, dtype='float64')
    comm.Recv(recv_data, source=0, tag=0)

# Collective with GPU arrays
result = cp.empty(100, dtype='float64')
comm.Allreduce(gpu_data, result, op=MPI.SUM)
```

## Error Handling

```python
# Check MPI errors
try:
    comm.send(data, dest=invalid_rank)
except MPI.Exception as e:
    print(f"MPI Error: {e}")

# Error handler
def my_error_handler(comm, error_code):
    print(f"MPI Error on rank {comm.Get_rank()}: {error_code}")

comm.Set_errhandler(MPI.ERRORS_RETURN)
```

## Timing and Profiling

```python
# MPI timer (wall clock)
start = MPI.Wtime()

# Work...
result = computation()

end = MPI.Wtime()
elapsed = end - start

# Global timing
local_time = elapsed
max_time = comm.reduce(local_time, op=MPI.MAX, root=0)

if rank == 0:
    print(f"Max time: {max_time} seconds")
```

## Integration Patterns

### NumPy Integration

```python
# Automatic datatype detection
data = np.array([1, 2, 3], dtype=np.int32)
comm.Send(data, dest=1)  # MPI.INT automatically detected

# Manual datatype
comm.Send([data, MPI.INT], dest=1)
```

### SLURM Integration

```bash
#!/bin/bash
#SBATCH -N 4
#SBATCH --ntasks-per-node=8
#SBATCH --time=01:00:00

# mpirun auto-detects SLURM
mpirun python mpi_script.py

# Or use srun directly
srun python mpi_script.py
```

### Hybrid MPI + OpenMP

```python
from mpi4py import MPI
import os

# Set OpenMP threads
os.environ['OMP_NUM_THREADS'] = '4'

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

# MPI between nodes, OpenMP within nodes
# Use MPI.COMM_TYPE_SHARED for node-local communication
```

## Debugging

```bash
# Verbose output
mpirun -n 4 --verbose python script.py

# Display map
mpirun -n 4 --display-map python script.py

# Attach debugger
mpirun -n 4 --debug python script.py

# Valgrind
mpirun -n 2 valgrind --leak-check=full python script.py
```

## References

Detailed documentation in `references/`:

- **mpi-standard.md**: MPI standard, concepts, and specifications
- **point-to-point.md**: Send/Recv operations, tags, buffering
- **collective-operations.md**: Broadcast, scatter, gather, reduce
- **parallel-io.md**: MPI-IO, collective I/O, file views
- **execution-deployment.md**: mpirun, hostfiles, SLURM, process mapping

## Official Documentation

- [MPI Forum Standard](https://www.mpi-forum.org/docs/)
- [Open MPI Documentation](https://docs.open-mpi.org/)
- [MPICH Documentation](https://www.mpich.org/documentation/)
- [mpi4py Documentation](https://mpi4py.readthedocs.io/)

## Best Practices

1. **Use collective operations**: More efficient than point-to-point loops
2. **Non-blocking communication**: Overlap communication and computation
3. **Match send/recv counts**: Avoid deadlocks
4. **Use MPI-IO**: Parallel file access
5. **Profile communication**: Identify bottlenecks
6. **Uppercase for NumPy**: Send/Recv for arrays
7. **Check buffer sizes**: Prevent overflows
8. **Use appropriate communicators**: Split for subsystems
9. **GPU-aware MPI**: Direct GPU transfers when available
10. **Test at small scale**: Debug before large runs
