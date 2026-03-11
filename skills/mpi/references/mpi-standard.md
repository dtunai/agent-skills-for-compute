# MPI Standard and Concepts Reference

Sources:
- [MPI Forum Official Documentation](https://www.mpi-forum.org/docs/)
- [MPI Standard 4.1](https://www.mpi-forum.org/docs/mpi-4.1/mpi41-report.pdf)
- [Open MPI Documentation](https://docs.open-mpi.org/)
- [MPICH Overview](https://www.mpich.org/about/overview/)

## Overview

The Message Passing Interface (MPI) is a standardized and portable message-passing system designed to function on parallel computing architectures. The MPI standard defines the syntax and semantics of library routines for writing portable message-passing programs.

**Current Standard**: MPI 4.1 (with MPI 5.0 approved June 2025)

## MPI Implementations

### Open MPI

Open MPI is an open source implementation developed and maintained by a consortium of academic, research, and industry partners. According to the documentation, Open MPI v5.0 series is the latest and is recommended for all users.

**Key Features**:
- Full MPI-3.1 and MPI-4.0 compliance
- Support for multiple network fabrics
- Advanced process management
- Extensive runtime options

### MPICH

MPICH is a high-performance and widely portable implementation of MPI (MPI-1, MPI-2, MPI-3, and MPI-4). As stated in the documentation, MPICH is "distributed under a BSD-like license" and received the 2024 ACM Software System Award.

**Key Features**:
- High performance
- Wide portability
- Support for heterogeneous systems
- Device abstraction layer

## Core Concepts

### Communicators

Communicators define groups of processes that can communicate with each other.

```python
from mpi4py import MPI

# World communicator (all processes)
comm = MPI.COMM_WORLD

# Get rank and size
rank = comm.Get_rank()  # Process ID (0 to size-1)
size = comm.Get_size()  # Total number of processes

# Self communicator (only this process)
self_comm = MPI.COMM_SELF
```

### Ranks

Each process in a communicator has a unique rank (integer ID starting from 0).

```python
if rank == 0:
    print("I am the root process")
elif rank == 1:
    print("I am process 1")
else:
    print(f"I am process {rank}")
```

### Tags

Tags are integers used to identify messages in point-to-point communication.

```python
# Send with tag
comm.send(data, dest=1, tag=42)

# Receive with matching tag
data = comm.recv(source=0, tag=42)

# MPI.ANY_TAG accepts any tag
data = comm.recv(source=0, tag=MPI.ANY_TAG)
```

## Data Types

### Built-in MPI Datatypes

MPI defines standard datatypes that map to language types:

| MPI Datatype | C Type | Python/NumPy |
|--------------|--------|--------------|
| MPI.CHAR | char | np.int8 |
| MPI.SHORT | short | np.int16 |
| MPI.INT | int | np.int32 |
| MPI.LONG | long | np.int64 |
| MPI.FLOAT | float | np.float32 |
| MPI.DOUBLE | double | np.float64 |
| MPI.BYTE | - | np.byte |

### Automatic Type Detection

mpi4py automatically detects NumPy array datatypes:

```python
import numpy as np

# Automatic detection
int_array = np.array([1, 2, 3], dtype=np.int32)
comm.Send(int_array, dest=1)  # MPI.INT detected

float_array = np.array([1.0, 2.0], dtype=np.float64)
comm.Send(float_array, dest=1)  # MPI.DOUBLE detected
```

### Derived Datatypes

Create custom datatypes for complex structures:

```python
# Contiguous type
oldtype = MPI.INT
count = 5
newtype = oldtype.Create_contiguous(count)
newtype.Commit()

# Vector type (strided)
blocklength = 2
stride = 3
count = 4
newtype = oldtype.Create_vector(count, blocklength, stride)
newtype.Commit()

# Free when done
newtype.Free()
```

## Communication Modes

### Standard Send (MPI_Send)

```python
# May buffer or block until matching receive
comm.send(data, dest=1, tag=0)
```

### Buffered Send (MPI_Bsend)

```python
# Always buffers, returns immediately
comm.bsend(data, dest=1, tag=0)
```

### Synchronous Send (MPI_Ssend)

```python
# Blocks until matching receive starts
comm.ssend(data, dest=1, tag=0)
```

### Ready Send (MPI_Rsend)

```python
# Assumes receive already posted (undefined behavior otherwise)
comm.rsend(data, dest=1, tag=0)
```

## Blocking vs Non-Blocking

### Blocking Communication

```python
# Sender blocks until data sent
if rank == 0:
    comm.send(data, dest=1)

# Receiver blocks until data received
if rank == 1:
    data = comm.recv(source=0)
```

### Non-Blocking Communication

```python
# Returns immediately with request object
if rank == 0:
    req = comm.isend(data, dest=1)
    # Can do other work
    req.wait()  # Block until complete

if rank == 1:
    req = comm.irecv(source=0)
    # Can do other work
    data = req.wait()  # Get data
```

## Synchronization

### Barrier

```python
# All processes wait until all reach barrier
comm.Barrier()
print(f"Process {rank} passed barrier")
```

### Wait Operations

```python
# Wait for single request
request = comm.isend(data, dest=1)
request.wait()

# Test if complete (non-blocking)
if request.test():
    print("Communication complete")

# Wait for any of multiple requests
requests = [comm.isend(data, dest=i) for i in range(1, size)]
index = MPI.Request.Waitany(requests)

# Wait for all requests
MPI.Request.Waitall(requests)
```

## Process Groups

### Creating Groups

```python
# Get world group
world_group = comm.Get_group()

# Create subset
ranks = [0, 2, 4]
new_group = world_group.Incl(ranks)

# Create communicator from group
new_comm = comm.Create(new_group)
```

### Group Operations

```python
# Union
group3 = MPI.Group.Union(group1, group2)

# Intersection
group3 = MPI.Group.Intersect(group1, group2)

# Difference
group3 = MPI.Group.Difference(group1, group2)

# Free group
group.Free()
```

## Communicator Management

### Splitting Communicators

```python
# Split by color
color = rank % 2  # Even/odd ranks
key = rank  # Order within new communicator
new_comm = comm.Split(color, key)

# Split by type (shared memory)
shared_comm = comm.Split_type(MPI.COMM_TYPE_SHARED)
```

### Duplicating Communicators

```python
# Create independent copy
dup_comm = comm.Dup()

# Changes to dup_comm don't affect comm
dup_comm.Set_name("Duplicate")

# Free when done
dup_comm.Free()
```

## Virtual Topologies

### Cartesian Topology

```python
# Create 2D Cartesian grid
ndims = 2
dims = [4, 3]  # 4x3 grid
periods = [False, True]  # Periodic in Y dimension
reorder = True

cart_comm = comm.Create_cart(dims, periods, reorder)

# Get coordinates
coords = cart_comm.Get_coords(rank)

# Get rank from coordinates
target_rank = cart_comm.Get_cart_rank([1, 2])

# Shift (get neighbors)
source, dest = cart_comm.Shift(0, 1)  # Shift in X by 1
```

### Graph Topology

```python
# Create graph topology
index = [2, 3, 5]  # Cumulative edge counts
edges = [1, 2, 0, 2, 0, 1]  # Edge destinations
reorder = False

graph_comm = comm.Create_graph(index, edges, reorder)

# Get neighbors
nneighbors, neighbors = graph_comm.Get_neighbors(rank)
```

## MPI Environment

### Initialization and Finalization

```python
from mpi4py import MPI

# Automatically initialized when importing
comm = MPI.COMM_WORLD

# Manual init (if needed)
if not MPI.Is_initialized():
    MPI.Init()

# Finalization (automatic at exit)
# MPI.Finalize()  # Rarely needed
```

### Environment Queries

```python
# MPI version
version, subversion = MPI.Get_version()
print(f"MPI {version}.{subversion}")

# Library version
library_version = MPI.Get_library_version()

# Processor name
processor_name = MPI.Get_processor_name()

# Thread support
provided = MPI.Query_thread()
# MPI.THREAD_SINGLE, THREAD_FUNNELED, THREAD_SERIALIZED, THREAD_MULTIPLE
```

## Error Handling

### Error Classes

```python
# MPI error classes
MPI.SUCCESS
MPI.ERR_BUFFER
MPI.ERR_COUNT
MPI.ERR_TYPE
MPI.ERR_TAG
MPI.ERR_COMM
MPI.ERR_RANK
MPI.ERR_REQUEST
MPI.ERR_ROOT
MPI.ERR_GROUP
MPI.ERR_OP
MPI.ERR_TOPOLOGY
```

### Error Handlers

```python
# Default: abort on error
comm.Set_errhandler(MPI.ERRORS_ARE_FATAL)

# Return error codes
comm.Set_errhandler(MPI.ERRORS_RETURN)

# Custom error handler
def custom_handler(comm, error_code):
    error_string = MPI.Get_error_string(error_code)
    print(f"MPI Error: {error_string}")

# Note: Custom handlers in mpi4py have limitations
```

## Process Management

### Spawning Processes

```python
# Spawn new MPI processes
intercomm = MPI.COMM_SELF.Spawn(
    'python',
    args=['worker.py'],
    maxprocs=4
)

# Parent-child communication
if intercomm != MPI.COMM_NULL:
    intercomm.send(data, dest=0)
```

### Dynamic Process Management

```python
# Connect/Accept pattern
if rank == 0:
    # Server accepts connection
    port_name = MPI.Open_port()
    print(f"Port: {port_name}")
    newcomm = comm.Accept(port_name)
else:
    # Client connects
    newcomm = comm.Connect(port_name)

# Disconnect
newcomm.Disconnect()
MPI.Close_port(port_name)
```

## Attributes and Caching

### Communicator Attributes

```python
# Set attribute
comm.Set_attr(MPI.TAG_UB, 100000)

# Get attribute
tag_ub = comm.Get_attr(MPI.TAG_UB)

# Predefined attributes
MPI.TAG_UB       # Upper bound for tags
MPI.HOST         # Host rank
MPI.IO           # I/O rank
MPI.WTIME_IS_GLOBAL  # Synchronized time
```

## One-Sided Communication (RMA)

### Window Creation

```python
import numpy as np

# Allocate window
size = 100 if rank == 0 else 0
itemsize = MPI.DOUBLE.Get_size()
win = MPI.Win.Allocate(size * itemsize, itemsize, comm=comm)

# Create window from existing memory
data = np.arange(100, dtype='float64')
win = MPI.Win.Create(data, comm=comm)
```

### RMA Operations

```python
# Put data
win.Lock(0)
data = np.ones(10, dtype='float64') * rank
win.Put(data, target_rank=0, target=slice(rank*10, (rank+1)*10))
win.Unlock(0)

# Get data
win.Lock(0)
data = np.empty(10, dtype='float64')
win.Get(data, target_rank=0, target=slice(0, 10))
win.Unlock(0)

# Accumulate
win.Lock(0)
data = np.ones(10, dtype='float64')
win.Accumulate(data, target_rank=0, target=slice(0, 10), op=MPI.SUM)
win.Unlock(0)

# Free window
win.Free()
```

## Thread Safety

### Thread Levels

```python
# Query required thread support
required = MPI.THREAD_MULTIPLE
provided = MPI.Init_thread(required)

if provided < required:
    print("Warning: Insufficient thread support")

# Thread levels (increasing support)
MPI.THREAD_SINGLE      # Only one thread
MPI.THREAD_FUNNELED    # Multithreaded, only main thread calls MPI
MPI.THREAD_SERIALIZED  # Multithreaded, serialized MPI calls
MPI.THREAD_MULTIPLE    # Multithreaded, any thread can call MPI
```

## Performance Considerations

### Buffering

```python
# Attach buffer for buffered sends
buffer_size = 10000 * MPI.DOUBLE.size + MPI.BSEND_OVERHEAD
buffer = bytearray(buffer_size)
MPI.Attach_buffer(buffer)

# Use Bsend...

# Detach when done
MPI.Detach_buffer()
```

### Persistent Communication

```python
# Create persistent send
send_buf = np.ones(100, dtype='float64')
send_req = comm.Send_init(send_buf, dest=1, tag=0)

# Create persistent receive
recv_buf = np.empty(100, dtype='float64')
recv_req = comm.Recv_init(recv_buf, source=0, tag=0)

# Start operations multiple times
for i in range(100):
    send_req.Start()
    recv_req.Start()
    send_req.Wait()
    recv_req.Wait()

# Free requests
send_req.Free()
recv_req.Free()
```

## Best Practices

1. **Use collective operations**: More efficient than loops of point-to-point
2. **Non-blocking when possible**: Overlap communication and computation
3. **Match message counts**: Avoid deadlocks and hangs
4. **Use appropriate communicators**: Split for logical groups
5. **Free resources**: Datatypes, groups, communicators, windows
6. **Check return values**: When using ERRORS_RETURN
7. **Profile communication**: Identify bottlenecks
8. **Use derived types**: Avoid multiple messages for structured data
9. **Barrier sparingly**: Global synchronization is expensive
10. **Test at small scale**: Debug before large-scale runs

## Common Pitfalls

- **Deadlock**: Circular dependencies in blocking communication
- **Buffer overflow**: Sending more data than receiver expects
- **Tag mismatch**: Send and receive tags don't match
- **Type mismatch**: Incompatible datatypes
- **Rank out of bounds**: Invalid source/dest ranks
- **Not freeing resources**: Memory leaks from datatypes, communicators
- **Race conditions**: Non-deterministic message ordering
- **Assuming message order**: Messages with same tag may arrive out of order
- **Forgetting Commit()**: Derived datatypes not committed
- **MPI after Finalize()**: Calling MPI functions after finalization
