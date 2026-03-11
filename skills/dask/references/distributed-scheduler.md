# Dask Distributed Scheduler Reference

Sources:
- [Dask.distributed Documentation](https://distributed.dask.org)
- [Distributed Quickstart](https://distributed.dask.org/en/stable/quickstart.html)
- [Client API](https://distributed.dask.org/en/stable/client.html)
- [Scheduling](https://distributed.dask.org/en/stable/scheduling-policies.html)

## Overview

The Dask distributed scheduler coordinates task execution across multiple workers. It provides:

- **Low latency**: ~1ms task overhead
- **Data locality**: Moves computation to data
- **Resilience**: Handles worker failures
- **Live feedback**: Real-time dashboard
- **Flexible deployment**: Local, cloud, HPC

## Architecture

### Components

1. **Scheduler**: Centralized coordinator that assigns tasks to workers
2. **Workers**: Execute tasks and store data
3. **Client**: User interface for submitting work

```
Client → Scheduler → Workers
          ↓            ↓
       Dashboard   Computations
```

## Installation

```bash
pip install dask distributed --upgrade

# Or with conda
conda install dask distributed -c conda-forge
```

## Local Cluster (Simple Method)

```python
from dask.distributed import Client

# Automatic local cluster
client = Client()

# This creates:
# - 1 scheduler
# - N workers (N = number of CPU cores)
# - 1 thread per worker by default
```

### Manual Local Cluster

```python
from dask.distributed import Client, LocalCluster

cluster = LocalCluster(
    n_workers=4,
    threads_per_worker=2,
    processes=True,
    memory_limit='8GB',
    dashboard_address=':8787'
)

client = Client(cluster)
```

### Configuration Options

```python
cluster = LocalCluster(
    n_workers=8,              # Number of worker processes
    threads_per_worker=2,     # Threads per worker
    processes=True,           # Use processes (vs threads)
    memory_limit='auto',      # Memory per worker
    dashboard_address=':8787',# Dashboard port
    silence_logs=False,       # Show logs
    host='localhost',         # Scheduler host
    scheduler_port=0,         # Auto-assign port
    worker_dashboard_address=None  # Worker dashboards
)
```

## Distributed Cluster (Multi-Machine)

### Start Scheduler

```bash
# On scheduler machine
dask scheduler
# Scheduler at: tcp://192.168.1.100:8786
# Dashboard at: http://192.168.1.100:8787/status
```

### Start Workers

```bash
# On worker machines
dask worker tcp://192.168.1.100:8786

# With configuration
dask worker tcp://192.168.1.100:8786 \
    --nthreads 4 \
    --nworkers 2 \
    --memory-limit 16GB \
    --local-directory /scratch
```

### Connect Client

```python
from dask.distributed import Client

client = Client('tcp://192.168.1.100:8786')
```

## Client Operations

### Basic Usage

```python
from dask.distributed import Client

client = Client()

# View cluster info
print(client)

# Dashboard link
print(client.dashboard_link)

# Cluster information
client.ncores()  # Cores per worker
client.nthreads()  # Total threads
client.who_has()  # Data location
client.has_what()  # Worker contents
```

### Submit and Map

```python
# Submit single task
def square(x):
    return x ** 2

future = client.submit(square, 10)
result = future.result()  # Block until complete

# Map over many inputs
futures = client.map(square, range(1000))
results = client.gather(futures)

# Submit with dependencies
a = client.submit(sum, [1, 2, 3])
b = client.submit(lambda x: x * 2, a)
b.result()  # 12
```

### Futures

```python
# Submit returns Future objects
future = client.submit(square, 10)

# Check status
future.status  # 'pending', 'running', 'finished', 'error'
future.done()  # True if complete

# Get result (blocks)
result = future.result()

# Get result (non-blocking)
if future.done():
    result = future.result(timeout=0)

# Cancel
future.cancel()

# Exception handling
try:
    result = future.result()
except Exception as e:
    print(f"Task failed: {e}")
```

### Gather

```python
# Gather multiple futures
futures = client.map(square, range(100))
results = client.gather(futures)

# Gather specific futures
subset = client.gather([futures[0], futures[10], futures[50]])

# Non-blocking gather
results = client.gather(futures, asynchronous=True)
```

### Scatter

```python
# Send data to cluster
data = list(range(1000))
scattered = client.scatter(data)

# Use scattered data in computations
futures = client.map(square, scattered)

# Scatter with replication
scattered = client.scatter(data, broadcast=True)
```

### Persist

```python
import dask.array as da

# Persist Dask collection in distributed memory
x = da.random.random((10000, 10000), chunks=(1000, 1000))
x = x.persist()  # Execute and keep in memory

# Check memory usage
client.who_has(x)
```

### Restart and Shutdown

```python
# Restart cluster (clear memory)
client.restart()

# Shutdown cluster
client.close()
cluster.close()

# Context manager
with Client() as client:
    # Work here
    result = client.submit(task, data).result()
# Automatically closed
```

## Adaptive Scaling

```python
from dask.distributed import Client, LocalCluster

cluster = LocalCluster()
client = Client(cluster)

# Enable adaptive scaling
cluster.adapt(
    minimum=2,      # Minimum workers
    maximum=10,     # Maximum workers
    interval='1s'   # Check interval
)

# Scale based on load
# - Add workers when tasks pending
# - Remove workers when idle
```

## Task Scheduling

### Scheduling Policies

The distributed scheduler uses several policies:

1. **Data locality**: Prefer workers that already have required data
2. **Load balancing**: Distribute work evenly
3. **Memory management**: Avoid workers with memory pressure
4. **Task priority**: Respect task dependencies and priorities

### Priority

```python
# High priority tasks
future = client.submit(important_task, data, priority=10)

# Low priority tasks
future = client.submit(background_task, data, priority=-10)
```

### Resources

```python
# Launch workers with resources
dask worker tcp://scheduler:8786 --resources "GPU=2"

# Submit tasks requiring resources
future = client.submit(gpu_task, data, resources={'GPU': 1})
```

## Dashboard

Access dashboard at `http://localhost:8787/status`

### Dashboard Views

1. **Status**: Overall cluster state
2. **Workers**: Worker CPU, memory, tasks
3. **Tasks**: Task stream and progress
4. **System**: System metrics (CPU, memory, network)
5. **Profile**: Task profiling data
6. **Graph**: Task dependency graph

### Monitoring

```python
# Get dashboard link
client.dashboard_link

# Get performance report
with performance_report(filename="report.html"):
    result = client.submit(task, data).result()
```

## Performance Optimization

### Worker Configuration

```python
cluster = LocalCluster(
    n_workers=8,
    threads_per_worker=1,  # More workers, fewer threads
    processes=True,        # Processes > threads for CPU-bound
    memory_limit='8GB'     # Prevent memory issues
)
```

### Memory Management

```python
# Explicitly delete data
del future
client.cancel(future)

# Rebalance data across workers
client.rebalance()

# Replicate frequently-used data
client.replicate(dataset, n=3)
```

### Task Batching

```python
# Bad: Many small tasks
futures = [client.submit(small_task, i) for i in range(100000)]

# Good: Batch into larger tasks
def batch_task(items):
    return [small_task(i) for i in items]

batches = [range(i, i+1000) for i in range(0, 100000, 1000)]
futures = client.map(batch_task, batches)
```

### Data Locality

```python
# Scatter data to specific workers
worker_addresses = list(client.scheduler_info()['workers'].keys())
scattered = client.scatter(data, workers=worker_addresses[0])

# Submit tasks to same worker
future = client.submit(process, scattered, workers=worker_addresses[0])
```

## Work Stealing

Dask automatically balances load by "stealing" tasks from busy workers:

```python
# Enable/disable work stealing (enabled by default)
client.run_on_scheduler(lambda dask_scheduler:
    dask_scheduler.extensions['stealing'].stop())

# Re-enable
client.run_on_scheduler(lambda dask_scheduler:
    dask_scheduler.extensions['stealing'].start())
```

## Debugging

### Logging

```python
# Get worker logs
logs = client.get_worker_logs()
for worker, log in logs.items():
    print(f"Worker {worker}:")
    print(log)

# Get scheduler logs
scheduler_logs = client.get_scheduler_logs()
```

### Profiling

```python
# Profile task execution
from dask.distributed import performance_report

with performance_report(filename="profile.html"):
    result = computation.compute()

# View profile.html in browser
```

### Task Errors

```python
# Get exception from failed task
future = client.submit(failing_task, data)
try:
    result = future.result()
except Exception as e:
    print(f"Task failed: {e}")

# Get traceback
future.traceback()

# Retry failed task
future = client.retry(future)
```

## Security

### TLS/SSL Encryption

```bash
# Generate self-signed certificates
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Start scheduler with TLS
dask scheduler --tls-ca-file cert.pem --tls-cert cert.pem --tls-key key.pem

# Start worker with TLS
dask worker tls://scheduler:8786 --tls-ca-file cert.pem --tls-cert cert.pem --tls-key key.pem
```

### Authentication

```python
from dask.distributed import Client, Security

security = Security(
    tls_ca_file='ca.pem',
    tls_client_cert='client-cert.pem',
    tls_client_key='client-key.pem',
    require_encryption=True
)

client = Client('tls://scheduler:8786', security=security)
```

## Advanced Features

### Actors

```python
# Create stateful actor
class Counter:
    def __init__(self):
        self.count = 0

    def increment(self):
        self.count += 1
        return self.count

# Create actor on worker
actor = client.submit(Counter, actor=True).result()

# Call actor methods
future = actor.increment()
result = future.result()  # 1
```

### Queues

```python
from dask.distributed import Queue

queue = Queue()

# Producer
def producer():
    for i in range(10):
        queue.put(i)

# Consumer
def consumer():
    results = []
    for _ in range(10):
        item = queue.get()
        results.append(item ** 2)
    return results

client.submit(producer)
result = client.submit(consumer).result()
```

### Variables

```python
from dask.distributed import Variable

var = Variable('my-variable')

# Set value
var.set(42)

# Get value
value = var.get()

# Delete
var.delete()
```

## Integration with Collections

### DataFrames

```python
import dask.dataframe as dd

client = Client()

df = dd.read_parquet('data/*.parquet')
result = df.groupby('key').value.mean().compute()
```

### Arrays

```python
import dask.array as da

client = Client()

x = da.random.random((10000, 10000), chunks=(1000, 1000))
result = x.mean().compute()
```

### Delayed

```python
from dask import delayed

client = Client()

@delayed
def process(x):
    return x ** 2

results = [process(i) for i in range(100)]
total = delayed(sum)(results).compute()
```

## Best Practices

1. **Use LocalCluster for development**: Easy setup and debugging
2. **Monitor dashboard**: Track performance and bottlenecks
3. **Batch small tasks**: Reduce scheduling overhead
4. **Persist intermediate results**: Avoid recomputation
5. **Use scatter for large data**: Send data once, use many times
6. **Configure worker memory**: Prevent out-of-memory errors
7. **Profile before scaling**: Optimize single-machine code first
8. **Use processes for CPU-bound**: Threads for I/O-bound
9. **Enable adaptive scaling**: Efficient resource usage
10. **Clean up futures**: Delete unused futures to free memory

## Common Pitfalls

- **Not closing client**: Memory leaks from unclosed clients
- **Too many small tasks**: Overhead dominates computation
- **Not persisting**: Recomputing expensive intermediate results
- **Returning large results**: Use `.to_parquet()` instead of `.compute()`
- **Not monitoring dashboard**: Missing performance bottlenecks
- **Incorrect worker count**: Too many workers for available memory
- **Ignoring data locality**: Unnecessary data transfer
- **Not batching**: Scheduler overhead from fine-grained tasks
- **Blocking main thread**: Use async operations when possible
- **Not handling errors**: Tasks fail silently without error checks

## Troubleshooting

### Workers Disappearing

```python
# Increase memory limit
cluster = LocalCluster(memory_limit='16GB')

# Disable memory spilling
cluster = LocalCluster(memory_limit='auto', memory_spill_fraction=False)
```

### Slow Performance

```python
# Check task stream in dashboard
# Look for:
# - Long serialization times (reduce data size)
# - Data transfer (improve locality)
# - Straggler tasks (load imbalance)
```

### Out of Memory

```python
# Reduce chunk size
df = dd.read_parquet('data.parquet', blocksize='50MB')

# Use disk spilling
cluster = LocalCluster(
    memory_limit='8GB',
    memory_spill_fraction=0.8,
    memory_target_fraction=0.6,
    memory_pause_fraction=0.9
)
```

### Connection Issues

```bash
# Check scheduler is running
netstat -an | grep 8786

# Check firewall rules
# Allow ports 8786 (scheduler) and 8787 (dashboard)

# Test connection
telnet scheduler-address 8786
```
