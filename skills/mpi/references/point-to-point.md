# MPI Point-to-Point Communication Reference

Sources:
- [mpi4py Tutorial](https://mpi4py.readthedocs.io/en/stable/tutorial.html)
- [mpi4py API Reference](https://mpi4py.readthedocs.io/en/stable/reference/mpi4py.MPI.Comm.html)
- [MPI Standard 4.1](https://www.mpi-forum.org/docs/mpi-4.1/mpi41-report.pdf)
- [Open MPI Documentation](https://docs.open-mpi.org/)

## Overview

Point-to-point communication involves message passing between two specific processes: a sender and a receiver. As stated in the mpi4py documentation, these operations come in two styles: **lowercase methods for Python objects** (using pickle serialization) and **uppercase methods for buffer-like objects** (fast, direct transfer).

## Basic Send and Receive

### Python Objects (Lowercase)

```python
from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

# Send Python object
if rank == 0:
    data = {'key': 'value', 'number': 42, 'list': [1, 2, 3]}
    comm.send(data, dest=1, tag=11)
    print(f"Rank {rank} sent: {data}")

# Receive Python object
if rank == 1:
    data = comm.recv(source=0, tag=11)
    print(f"Rank {rank} received: {data}")
```

**Features**:
- Flexible: Any picklable Python object
- Automatic serialization
- Slower than buffer-based communication
- Returns received object directly

### NumPy Arrays (Uppercase)

```python
import numpy as np
from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

# Send NumPy array
if rank == 0:
    data = np.arange(100, dtype='float64')
    comm.Send(data, dest=1, tag=13)
    print(f"Rank {rank} sent array of shape {data.shape}")

# Receive NumPy array
if rank == 1:
    data = np.empty(100, dtype='float64')
    comm.Recv(data, source=0, tag=13)
    print(f"Rank {rank} received: {data[:5]}...")
```

**Features**:
- Fast: Near C-speed
- Direct memory access
- Buffer must be pre-allocated
- Automatic MPI datatype detection

## Non-Blocking Communication

### Isend/Irecv (Immediate)

```python
# Non-blocking send
if rank == 0:
    data = np.random.randn(1000)
    req = comm.Isend(data, dest=1, tag=0)

    # Do other work
    local_computation()

    # Wait for completion
    req.wait()

# Non-blocking receive
if rank == 1:
    data = np.empty(1000)
    req = comm.Irecv(data, source=0, tag=0)

    # Do other work
    local_computation()

    # Wait for data
    req.wait()
    print(f"Received: {data[:5]}...")
```

### Request Objects

```python
# Test if complete (non-blocking)
req = comm.Isend(data, dest=1)
if req.test():
    print("Send completed")
else:
    print("Send still in progress")

# Wait with timeout is not standard, but test in loop
import time
while not req.test():
    time.sleep(0.001)
    # Do other work

# Cancel request
req.cancel()
```

### Multiple Requests

```python
# Send to multiple destinations
requests = []
data = np.ones(100, dtype='float64') * rank

for dest in range(size):
    if dest != rank:
        req = comm.Isend(data, dest=dest, tag=0)
        requests.append(req)

# Wait for all
MPI.Request.Waitall(requests)

# Wait for any
index = MPI.Request.Waitany(requests)
print(f"Request {index} completed first")

# Test all
flag, indices = MPI.Request.Testall(requests)
if flag:
    print("All requests complete")
```

## Send Modes

### Standard Send

```python
# May buffer or block until receiver ready
comm.send(data, dest=1, tag=0)  # Python object
comm.Send(data, dest=1, tag=0)  # NumPy array
```

### Buffered Send

```python
# Always buffers, returns immediately
# Requires buffer attachment

# Attach buffer
buffer_size = 10000 * MPI.DOUBLE.size + MPI.BSEND_OVERHEAD
buffer = bytearray(buffer_size)
MPI.Attach_buffer(buffer)

# Buffered send
data = np.random.randn(1000)
comm.Bsend(data, dest=1, tag=0)

# Detach buffer
MPI.Detach_buffer()
```

### Synchronous Send

```python
# Blocks until matching receive starts
# Guarantees synchronization
data = np.ones(100, dtype='float64')
comm.Ssend(data, dest=1, tag=0)
```

### Ready Send

```python
# Assumes receiver already waiting
# Undefined behavior if receiver not ready
# Can be faster if receiver is guaranteed ready

data = np.ones(100, dtype='float64')
comm.Rsend(data, dest=1, tag=0)
```

## Combined Send-Receive

### Sendrecv

```python
# Exchange data between two processes
# Avoids deadlock
sendbuf = np.ones(100, dtype='float64') * rank
recvbuf = np.empty(100, dtype='float64')

comm.Sendrecv(
    sendbuf, dest=(rank + 1) % size,
    recvbuf=recvbuf, source=(rank - 1) % size,
    sendtag=0, recvtag=0
)

print(f"Rank {rank} received from {(rank - 1) % size}: {recvbuf[:5]}")
```

### Sendrecv_replace

```python
# Send and receive into same buffer
# Temporary copy made automatically
data = np.ones(100, dtype='float64') * rank

comm.Sendrecv_replace(
    data,
    dest=(rank + 1) % size,
    source=(rank - 1) % size
)

print(f"Rank {rank} now has: {data[:5]}")
```

## Message Probing

### Probe (Blocking)

```python
# Check for incoming message
if rank == 1:
    status = MPI.Status()
    comm.probe(source=MPI.ANY_SOURCE, tag=MPI.ANY_TAG, status=status)

    # Get message info
    source = status.Get_source()
    tag = status.Get_tag()
    count = status.Get_count(MPI.DOUBLE)

    # Receive message
    data = np.empty(count, dtype='float64')
    comm.Recv(data, source=source, tag=tag)
```

### Iprobe (Non-Blocking)

```python
# Non-blocking probe
status = MPI.Status()
flag = comm.iprobe(source=MPI.ANY_SOURCE, tag=MPI.ANY_TAG, status=status)

if flag:
    # Message available
    source = status.Get_source()
    count = status.Get_count(MPI.DOUBLE)
    data = np.empty(count, dtype='float64')
    comm.Recv(data, source=source)
else:
    # No message, do other work
    pass
```

## Status Objects

### Getting Message Information

```python
status = MPI.Status()

# Receive with status
if rank == 1:
    data = np.empty(100, dtype='float64')
    comm.Recv(data, source=MPI.ANY_SOURCE, tag=MPI.ANY_TAG, status=status)

    # Extract info
    source = status.Get_source()
    tag = status.Get_tag()
    error = status.Get_error()
    count = status.Get_count(MPI.DOUBLE)

    print(f"Received from rank {source}, tag {tag}, count {count}")
```

### Status with Non-Blocking

```python
# Non-blocking with status
req = comm.Irecv(data, source=0, tag=0)
status = MPI.Status()
req.wait(status)

source = status.Get_source()
count = status.Get_count(MPI.DOUBLE)
```

## Tags and Wildcards

### Explicit Tags

```python
# Different messages with different tags
if rank == 0:
    data1 = np.ones(10)
    data2 = np.ones(20) * 2
    comm.Send(data1, dest=1, tag=1)
    comm.Send(data2, dest=1, tag=2)

if rank == 1:
    # Receive in specific order
    data2 = np.empty(20)
    comm.Recv(data2, source=0, tag=2)  # Receive tag 2 first

    data1 = np.empty(10)
    comm.Recv(data1, source=0, tag=1)
```

### Wildcards

```python
# Receive from any source
data = comm.recv(source=MPI.ANY_SOURCE)

# Receive any tag
data = comm.recv(source=0, tag=MPI.ANY_TAG)

# Receive from any source with any tag
status = MPI.Status()
data = comm.recv(source=MPI.ANY_SOURCE, tag=MPI.ANY_TAG, status=status)

# Check who sent it
actual_source = status.Get_source()
actual_tag = status.Get_tag()
```

## Persistent Communication

### Creating Persistent Requests

```python
# Initialize persistent send
sendbuf = np.ones(100, dtype='float64')
send_req = comm.Send_init(sendbuf, dest=1, tag=0)

# Initialize persistent receive
recvbuf = np.empty(100, dtype='float64')
recv_req = comm.Recv_init(recvbuf, source=0, tag=0)
```

### Using Persistent Requests

```python
# Reuse multiple times
for iteration in range(1000):
    # Start communication
    send_req.Start()
    recv_req.Start()

    # Wait for completion
    send_req.Wait()
    recv_req.Wait()

    # Process data
    process(recvbuf)

# Free when done
send_req.Free()
recv_req.Free()
```

## Buffering and Memory

### User-Provided Buffers

```python
# Pre-allocate receive buffer
data = np.empty(1000, dtype='float64')

# Receive into buffer
comm.Recv(data, source=0, tag=0)

# Buffer is filled with received data
```

### Buffer Attachment for Bsend

```python
# Calculate buffer size
num_messages = 10
message_size = 1000
buffer_size = num_messages * (message_size * MPI.DOUBLE.size + MPI.BSEND_OVERHEAD)

# Attach buffer
buffer = bytearray(buffer_size)
MPI.Attach_buffer(buffer)

# Use buffered sends
for i in range(num_messages):
    data = np.random.randn(message_size)
    comm.Bsend(data, dest=1, tag=i)

# Detach buffer
MPI.Detach_buffer()
```

## Error Handling

### Checking for Errors

```python
try:
    comm.send(data, dest=invalid_rank)
except MPI.Exception as e:
    print(f"MPI Error: {e}")
```

### Status Error Codes

```python
status = MPI.Status()
comm.Recv(data, source=0, tag=0, status=status)

if status.Get_error() != MPI.SUCCESS:
    print(f"Error in receive: {status.Get_error()}")
```

## Performance Patterns

### Overlapping Communication and Computation

```python
# Start non-blocking send
req_send = comm.Isend(sendbuf, dest=1, tag=0)

# Do computation while send progresses
result = expensive_computation()

# Complete send
req_send.wait()

# Send results
comm.send(result, dest=1, tag=1)
```

### Pipeline Pattern

```python
# Pipeline: each rank receives, processes, sends to next
if rank > 0:
    # Receive from previous rank
    data = np.empty(100, dtype='float64')
    comm.Recv(data, source=rank-1, tag=0)
else:
    # First rank generates data
    data = np.random.randn(100)

# Process data
processed = process(data)

if rank < size - 1:
    # Send to next rank
    comm.Send(processed, dest=rank+1, tag=0)
else:
    # Last rank outputs result
    print(f"Final result: {processed[:5]}")
```

### Ring Communication

```python
# Send to right neighbor, receive from left
right = (rank + 1) % size
left = (rank - 1 + size) % size

sendbuf = np.ones(100, dtype='float64') * rank
recvbuf = np.empty(100, dtype='float64')

# Non-blocking to avoid deadlock
req_send = comm.Isend(sendbuf, dest=right, tag=0)
req_recv = comm.Irecv(recvbuf, source=left, tag=0)

# Wait for both
MPI.Request.Waitall([req_send, req_recv])

print(f"Rank {rank} received from {left}: {recvbuf[:5]}")
```

## Advanced Patterns

### Master-Worker

```python
MASTER = 0

if rank == MASTER:
    # Master distributes work
    for worker in range(1, size):
        task = generate_task(worker)
        comm.send(task, dest=worker, tag=1)

    # Collect results
    results = []
    for worker in range(1, size):
        result = comm.recv(source=worker, tag=2)
        results.append(result)

else:
    # Worker receives task
    task = comm.recv(source=MASTER, tag=1)

    # Process task
    result = process(task)

    # Send result back
    comm.send(result, dest=MASTER, tag=2)
```

### Dynamic Load Balancing

```python
if rank == 0:
    # Master with task queue
    tasks = list(range(100))
    results = []

    # Send initial tasks
    for worker in range(1, min(size, len(tasks)+1)):
        if tasks:
            comm.send(tasks.pop(0), dest=worker, tag=1)

    # Receive results and send new tasks
    while len(results) < 100:
        status = MPI.Status()
        result = comm.recv(source=MPI.ANY_SOURCE, tag=2, status=status)
        results.append(result)

        worker = status.Get_source()
        if tasks:
            comm.send(tasks.pop(0), dest=worker, tag=1)
        else:
            comm.send(None, dest=worker, tag=1)  # Shutdown signal

else:
    # Worker
    while True:
        task = comm.recv(source=0, tag=1)
        if task is None:
            break

        result = process(task)
        comm.send(result, dest=0, tag=2)
```

## Debugging Point-to-Point

### Deadlock Detection

```python
# BAD: Potential deadlock
if rank == 0:
    comm.send(data, dest=1)
    comm.recv(source=1)
if rank == 1:
    comm.send(data, dest=0)
    comm.recv(source=0)

# GOOD: Use Sendrecv
if rank == 0:
    comm.Sendrecv(sendbuf, dest=1, recvbuf=recvbuf, source=1)
if rank == 1:
    comm.Sendrecv(sendbuf, dest=0, recvbuf=recvbuf, source=0)

# GOOD: Non-blocking
if rank == 0:
    req_send = comm.isend(data, dest=1)
    req_recv = comm.irecv(source=1)
    req_send.wait()
    req_recv.wait()
```

### Message Tracing

```python
# Add logging
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

if rank == 0:
    logger.debug(f"Rank {rank} sending to {1}")
    comm.send(data, dest=1)
    logger.debug(f"Rank {rank} sent successfully")
```

## Best Practices

1. **Use non-blocking when possible**: Overlap communication and computation
2. **Match send/recv counts**: Avoid buffer overflows
3. **Use Sendrecv for exchanges**: Avoids deadlocks
4. **Uppercase for NumPy**: Much faster than lowercase
5. **Pre-allocate receive buffers**: Required for uppercase methods
6. **Use tags meaningfully**: Different message types
7. **Probe before receiving**: When message size unknown
8. **Free persistent requests**: Prevent memory leaks
9. **Test at small scale**: Debug before large runs
10. **Profile communication**: Identify bottlenecks

## Common Pitfalls

- **Deadlock from circular dependencies**: Use non-blocking or Sendrecv
- **Buffer size mismatch**: Receiver buffer too small
- **Tag mismatch**: Send and receive tags don't match
- **Source mismatch**: Receiving from wrong rank
- **Type mismatch**: Different datatypes in send/recv
- **Forgetting to wait**: Non-blocking without wait
- **Using blocking in both directions**: Classic deadlock
- **Not checking request status**: Assuming completion
- **Attaching buffer for wrong send mode**: Bsend needs buffer
- **Ready send without ready receiver**: Undefined behavior
