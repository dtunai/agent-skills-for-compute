# MPI Parallel I/O Reference

Sources:
- [mpi4py I/O](https://mpi4py.readthedocs.io/en/stable/tutorial.html#mpi-io)
- [MPI Standard 4.1](https://www.mpi-forum.org/docs/mpi-4.1/mpi41-report.pdf)
- [Open MPI Documentation](https://docs.open-mpi.org/)

## Overview

MPI-IO provides parallel file I/O capabilities, allowing multiple processes to access a single file concurrently. This avoids serialization bottlenecks and enables high-performance parallel applications.

**Key Benefits**:
- Multiple processes read/write simultaneously
- Atomicity guarantees
- Collective I/O optimization
- Portable across file systems

## File Access Modes

### Opening Files

```python
from mpi4py import MPI
import numpy as np

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

# Access modes
amode = MPI.MODE_WRONLY | MPI.MODE_CREATE  # Write, create if needed
amode = MPI.MODE_RDONLY                     # Read only
amode = MPI.MODE_RDWR                       # Read and write
amode = MPI.MODE_APPEND                     # Append mode
amode = MPI.MODE_EXCL                       # Exclusive create
amode = MPI.MODE_DELETE_ON_CLOSE            # Delete when closed
amode = MPI.MODE_SEQUENTIAL                 # Sequential access
amode = MPI.MODE_UNIQUE_OPEN                # Single open per process

# Open file
fh = MPI.File.Open(comm, "output.dat", amode)

# Close file
fh.Close()
```

### File Info

```python
# Set file info (hints)
info = MPI.Info.Create()
info.Set('striping_factor', '4')
info.Set('striping_unit', '1048576')  # 1MB

fh = MPI.File.Open(comm, "output.dat", amode, info=info)

info.Free()
```

## Individual File Pointers

### Explicit Offsets

```python
# Write at specific offset
amode = MPI.MODE_WRONLY | MPI.MODE_CREATE
fh = MPI.File.Open(comm, "output.dat", amode)

# Each rank writes to different location
offset = rank * 100 * 8  # 100 doubles per rank
data = np.arange(100, dtype='float64') * rank

fh.Write_at(offset, data)
fh.Close()
```

### Read with Explicit Offsets

```python
# Read from specific offset
amode = MPI.MODE_RDONLY
fh = MPI.File.Open(comm, "output.dat", amode)

offset = rank * 100 * 8
data = np.empty(100, dtype='float64')

fh.Read_at(offset, data)
fh.Close()

print(f"Rank {rank} read: {data[:5]}")
```

### Individual File Pointer

```python
# Each process has own file pointer
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Seek to position
fh.Seek(rank * 100 * 8, MPI.SEEK_SET)

# Write at current position
data = np.arange(100, dtype='float64') * rank
fh.Write(data)

fh.Close()
```

## Shared File Pointer

### All Processes Share Pointer

```python
# Shared pointer moves atomically
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Processes take turns (serialized)
data = np.ones(50, dtype='float64') * rank
fh.Write_shared(data)

# Each process writes after previous
fh.Close()
```

### Ordered Writes

```python
# Write in rank order
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

data = np.ones(100, dtype='float64') * rank
fh.Write_ordered(data)

fh.Close()
```

## File Views

### Setting Views

```python
# Define what each process sees
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Each rank's displacement
disp = rank * 100 * 8  # Byte offset
etype = MPI.DOUBLE     # Elementary type
filetype = MPI.DOUBLE  # File type

fh.Set_view(disp, etype, filetype, datarep='native')

# Write from local offset 0 (maps to global offset disp)
data = np.arange(100, dtype='float64') * rank
fh.Write_at(0, data)

fh.Close()
```

### Complex File Views

```python
# Interleaved access pattern
# Rank 0: positions 0, 2, 4, 6...
# Rank 1: positions 1, 3, 5, 7...

# Create strided datatype
count = 100
blocklength = 1
stride = 2

filetype = MPI.DOUBLE.Create_vector(count, blocklength, stride)
filetype.Commit()

# Set view
disp = rank * MPI.DOUBLE.Get_size()
fh.Set_view(disp, MPI.DOUBLE, filetype, datarep='native')

data = np.arange(100, dtype='float64') * rank
fh.Write_at(0, data)

filetype.Free()
fh.Close()
```

## Collective I/O

### Collective Write

```python
# All processes participate
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Set view
offset = rank * 100 * 8
fh.Set_view(offset, MPI.DOUBLE)

# Collective write (optimized)
data = np.arange(100, dtype='float64') * rank
fh.Write_all(data)

fh.Close()
```

### Collective Read

```python
# All processes read together
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_RDONLY)

offset = rank * 100 * 8
fh.Set_view(offset, MPI.DOUBLE)

data = np.empty(100, dtype='float64')
fh.Read_all(data)

fh.Close()
```

### Split Collective I/O

```python
# Some processes participate in collective
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

if rank < comm.Get_size() // 2:
    # First half writes
    data = np.arange(100, dtype='float64') * rank
    fh.Write_all(data)
else:
    # Second half doesn't write but must participate
    data = np.empty(0, dtype='float64')
    fh.Write_all(data)

fh.Close()
```

## Non-Blocking I/O

### Immediate Operations

```python
# Non-blocking write
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

data = np.arange(100, dtype='float64') * rank
offset = rank * 100 * 8

req = fh.Iwrite_at(offset, data)

# Do other work
computation()

# Wait for I/O completion
req.wait()

fh.Close()
```

### Non-Blocking Read

```python
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_RDONLY)

data = np.empty(100, dtype='float64')
offset = rank * 100 * 8

req = fh.Iread_at(offset, data)

# Do other work
computation()

# Complete read
req.wait()
print(f"Rank {rank}: {data[:5]}")

fh.Close()
```

## Atomic Operations

### Atomic Mode

```python
# Enable atomic operations
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Set atomic mode
fh.Set_atomicity(True)

# Writes are atomic
data = np.ones(100, dtype='float64') * rank
fh.Write_at(rank * 100 * 8, data)

# Check atomic mode
is_atomic = fh.Get_atomicity()

fh.Close()
```

## File Manipulation

### File Size

```python
# Get file size
fh = MPI.File.Open(comm, "data.dat", MPI.MODE_RDONLY)
size = fh.Get_size()

if rank == 0:
    print(f"File size: {size} bytes")

fh.Close()
```

### Preallocate

```python
# Preallocate file space
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Allocate 1GB
size = 1024 * 1024 * 1024
fh.Preallocate(size)

# Write data...

fh.Close()
```

### Sync

```python
# Flush data to disk
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

data = np.arange(100, dtype='float64') * rank
fh.Write_at(rank * 100 * 8, data)

# Force write to disk
fh.Sync()

fh.Close()
```

### Delete File

```python
# Delete file
MPI.File.Delete("old_file.dat", MPI.INFO_NULL)
```

## Consistency Semantics

### Consistency Guarantees

```python
# Sequential consistency
fh = MPI.File.Open(comm, "data.dat", MPI.MODE_RDWR)

# Write
data = np.ones(100, dtype='float64') * rank
fh.Write_at(rank * 100 * 8, data)

# Sync to ensure visibility
fh.Sync()

# Barrier to coordinate
comm.Barrier()

# Read (sees previous writes)
read_data = np.empty(100, dtype='float64')
fh.Read_at(0, read_data)  # Reads rank 0's data

fh.Close()
```

## Error Handling

### Checking Errors

```python
try:
    fh = MPI.File.Open(comm, "nonexistent.dat", MPI.MODE_RDONLY)
except MPI.Exception as e:
    print(f"Error opening file: {e}")

# Check after operations
fh = MPI.File.Open(comm, "data.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

try:
    data = np.arange(100, dtype='float64')
    fh.Write_at(rank * 100 * 8, data)
except MPI.Exception as e:
    print(f"Write error: {e}")

fh.Close()
```

## Performance Optimization

### Collective Buffering

```python
# Enable collective buffering
info = MPI.Info.Create()
info.Set('collective_buffering', 'true')
info.Set('cb_buffer_size', '16777216')  # 16MB
info.Set('cb_nodes', '4')  # Number of aggregators

fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE, info=info)

# Collective I/O benefits from buffering
data = np.arange(100, dtype='float64') * rank
fh.Write_all(data)

fh.Close()
info.Free()
```

### Data Sieving

```python
# Enable data sieving for non-contiguous access
info = MPI.Info.Create()
info.Set('romio_ds_write', 'enable')
info.Set('romio_ds_read', 'enable')

fh = MPI.File.Open(comm, "data.dat", MPI.MODE_RDWR, info=info)

# Operations here...

fh.Close()
info.Free()
```

### Striping

```python
# Lustre filesystem hints
info = MPI.Info.Create()
info.Set('striping_factor', '8')   # 8 OSTs
info.Set('striping_unit', '4194304')  # 4MB stripe size

fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE, info=info)

# Large parallel writes benefit from striping

fh.Close()
info.Free()
```

## Common Patterns

### Parallel HDF5 Style

```python
# Each rank writes to own dataset
fh = MPI.File.Open(comm, "output.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Header (rank 0 only)
if rank == 0:
    header = np.array([comm.Get_size(), 100], dtype='int32')
    fh.Write_at(0, header)

comm.Barrier()

# Data offset after header
header_size = 2 * 4  # 2 int32s
offset = header_size + rank * 100 * 8

data = np.arange(100, dtype='float64') * rank
fh.Write_at(offset, data)

fh.Close()
```

### Checkpoint/Restart

```python
def checkpoint(iteration, state):
    """Save checkpoint"""
    filename = f"checkpoint_{iteration:06d}.dat"
    fh = MPI.File.Open(comm, filename, MPI.MODE_WRONLY | MPI.MODE_CREATE)

    offset = rank * len(state) * 8
    fh.Write_at(offset, state)
    fh.Close()

def restart(iteration):
    """Load checkpoint"""
    filename = f"checkpoint_{iteration:06d}.dat"
    fh = MPI.File.Open(comm, filename, MPI.MODE_RDONLY)

    state = np.empty(100, dtype='float64')
    offset = rank * len(state) * 8
    fh.Read_at(offset, state)
    fh.Close()

    return state

# Save every N iterations
for i in range(1000):
    # Compute
    state = compute_state()

    if i % 100 == 0:
        checkpoint(i, state)
```

### Log File

```python
# Append to shared log file
fh = MPI.File.Open(comm, "simulation.log",
                   MPI.MODE_WRONLY | MPI.MODE_CREATE | MPI.MODE_APPEND)

# Each rank writes log entry (serialized)
message = f"Rank {rank}: iteration complete\n"
fh.Write_shared(message.encode('utf-8'))

fh.Close()
```

### Large Array Distribution

```python
# Write large distributed array
size = comm.Get_size()
rank = comm.Get_rank()

local_size = 1000000  # Elements per rank
local_data = np.random.randn(local_size)

fh = MPI.File.Open(comm, "big_array.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Collective write
offset = rank * local_size * 8
fh.Set_view(offset, MPI.DOUBLE)
fh.Write_all(local_data)

fh.Close()
```

## Integration with NumPy

### Direct Array I/O

```python
# NumPy arrays work directly
data = np.random.randn(1000, 500)

fh = MPI.File.Open(comm, "matrix.dat", MPI.MODE_WRONLY | MPI.MODE_CREATE)

# Write 2D array
offset = rank * data.nbytes
fh.Write_at(offset, data)

fh.Close()

# Read back
fh = MPI.File.Open(comm, "matrix.dat", MPI.MODE_RDONLY)

read_data = np.empty((1000, 500), dtype='float64')
fh.Read_at(offset, read_data)

fh.Close()
```

## Best Practices

1. **Use collective I/O**: Much faster than independent I/O
2. **Set appropriate hints**: Optimize for filesystem
3. **Preallocate files**: Avoid fragmentation
4. **Use file views**: Simplify access patterns
5. **Avoid small I/O**: Batch operations when possible
6. **Sync sparingly**: Expensive operation
7. **Close files**: Free resources
8. **Test I/O performance**: Profile before production
9. **Use parallel file systems**: Lustre, GPFS for best performance
10. **Handle errors**: Check return values

## Common Pitfalls

- **Not using collective I/O**: Missing performance benefits
- **Small scattered I/O**: Very slow on parallel filesystems
- **Not setting hints**: Default settings may be suboptimal
- **Overlapping writes**: Undefined behavior without atomic mode
- **Not syncing**: Data may not be visible to other processes
- **Memory alignment**: Some filesystems require aligned buffers
- **Forgetting to close**: Resource leaks
- **Inconsistent file views**: Different processes see different data
- **Not handling errors**: Silent failures
- **Using POSIX I/O**: Serial bottleneck in parallel code
