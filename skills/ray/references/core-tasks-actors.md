# Ray Core: Tasks, Actors, and Objects Reference

Sources:
- [Ray Core Walkthrough](https://docs.ray.io/en/latest/ray-core/walkthrough.html)
- [Ray Core Key Concepts](https://docs.ray.io/en/latest/ray-core/key-concepts.html)
- [Tasks User Guide](https://docs.ray.io/en/latest/ray-core/tasks.html)
- [Actors User Guide](https://docs.ray.io/en/latest/ray-core/actors.html)
- [Objects User Guide](https://docs.ray.io/en/latest/ray-core/objects.html)

## Overview

Ray Core is a distributed computing framework built on three fundamental primitives: **tasks**, **actors**, and **objects**. As stated in the official documentation: "Tasks are the simplest way to parallelize your Python functions across a Ray cluster."

## Installation and Initialization

### Installation

```bash
# Basic installation
pip install -U ray

# With dashboard
pip install -U "ray[default]"

# Nightly build
pip install -U "ray[default]" --pre
```

### Initialization

```python
import ray

# Local mode (auto-detect resources)
ray.init()

# Specify resources
ray.init(num_cpus=8, num_gpus=2, memory=10*1024*1024*1024)

# Connect to cluster
ray.init(address="ray://cluster-address:10001")

# Check initialization
ray.is_initialized()  # True

# Shutdown
ray.shutdown()
```

## Tasks (Remote Functions)

### Basic Task Definition

```python
import ray

@ray.remote
def square(x):
    return x ** 2

# Call task
future = square.remote(4)

# Get result
result = ray.get(future)  # 16
```

### Task Features

**Stateless**: Tasks do not maintain state between calls

**Parallelizable**: Multiple tasks execute concurrently

**Asynchronous**: Tasks return immediately with ObjectRef (future)

### Multiple Returns

```python
@ray.remote(num_returns=3)
def multi_return():
    return 1, 2, 3

a, b, c = multi_return.remote()
values = ray.get([a, b, c])  # [1, 2, 3]
```

### Task Dependencies

```python
@ray.remote
def fetch_data(url):
    return download(url)

@ray.remote
def process_data(data):
    return transform(data)

@ray.remote
def save_result(data):
    write_to_disk(data)
    return "saved"

# Build dependency graph
data = fetch_data.remote("http://example.com")
processed = process_data.remote(data)
status = save_result.remote(processed)

# Execute graph
ray.get(status)
```

### Passing ObjectRefs

```python
@ray.remote
def add(a, b):
    return a + b

# Pass ObjectRefs directly
a = add.remote(1, 2)
b = add.remote(3, 4)
c = add.remote(a, b)  # Automatically waits for a and b

result = ray.get(c)  # 10
```

### Task Options

```python
# Specify resources
@ray.remote(num_cpus=2, num_gpus=1, memory=1024*1024*1024)
def gpu_task(data):
    return process_on_gpu(data)

# Override at runtime
future = gpu_task.options(num_gpus=2).remote(data)

# Retry failed tasks
@ray.remote(max_retries=3, retry_exceptions=True)
def flaky_task():
    return unstable_operation()
```

### Cancellation

```python
import time

@ray.remote
def long_task():
    time.sleep(100)
    return "done"

future = long_task.remote()

# Cancel task
ray.cancel(future)

# Check if cancelled
try:
    ray.get(future)
except ray.exceptions.TaskCancelledError:
    print("Task was cancelled")
```

## Actors (Remote Classes)

### Basic Actor Definition

```python
@ray.remote
class Counter:
    def __init__(self, initial_value=0):
        self.value = initial_value

    def increment(self):
        self.value += 1
        return self.value

    def get_value(self):
        return self.value

# Create actor
counter = Counter.remote(initial_value=10)

# Call methods
future = counter.increment.remote()
result = ray.get(future)  # 11

value = ray.get(counter.get_value.remote())  # 11
```

### Actor Features

**Stateful**: Actors maintain state between method calls

**Serial Execution**: Methods execute one at a time in order received

**Dedicated Process**: Each actor runs in its own Python process

### Actor Handles

```python
@ray.remote
class DataStore:
    def __init__(self):
        self.data = {}

    def put(self, key, value):
        self.data[key] = value

    def get(self, key):
        return self.data.get(key)

# Create actor
store = DataStore.remote()

# Pass handle to tasks
@ray.remote
def worker(store_handle, key, value):
    store_handle.put.remote(key, value)

# Multiple tasks use same actor
futures = [worker.remote(store, f"key_{i}", i) for i in range(10)]
ray.get(futures)
```

### Actor Lifetime

```python
# Create actor
actor = MyActor.remote()

# Check if alive
ray.get(actor.ping.remote())

# Kill actor
ray.kill(actor)

# Detached actors (survive driver exit)
@ray.remote
class DetachedActor:
    pass

actor = DetachedActor.options(
    name="my_actor",
    lifetime="detached"
).remote()

# Retrieve detached actor
actor = ray.get_actor("my_actor")
```

### Actor Pools

```python
from ray.util.actor_pool import ActorPool

@ray.remote
class Worker:
    def process(self, item):
        return expensive_operation(item)

# Create pool
workers = [Worker.remote() for _ in range(10)]
pool = ActorPool(workers)

# Submit work
for item in dataset:
    pool.submit(lambda w, i: w.process.remote(i), item)

# Get results
results = []
while pool.has_next():
    results.append(pool.get_next())
```

### Async Actors

```python
@ray.remote
class AsyncActor:
    async def async_method(self):
        await async_operation()
        return "done"

    def sync_method(self):
        return "sync"

actor = AsyncActor.remote()

# Concurrent async calls
futures = [actor.async_method.remote() for _ in range(10)]
results = ray.get(futures)
```

### Actor Concurrency

```python
# Allow concurrent method execution
@ray.remote(max_concurrency=5)
class ConcurrentActor:
    async def concurrent_method(self, x):
        await async_work(x)
        return x * 2

actor = ConcurrentActor.remote()

# Up to 5 methods execute concurrently
futures = [actor.concurrent_method.remote(i) for i in range(20)]
results = ray.get(futures)
```

## Objects and Object Store

### Creating Objects

```python
# Implicit creation (task return)
@ray.remote
def create_object():
    return [1, 2, 3, 4, 5]

obj_ref = create_object.remote()

# Explicit creation
large_data = load_large_dataset()
obj_ref = ray.put(large_data)
```

### Retrieving Objects

```python
# Get single object
result = ray.get(obj_ref)

# Get multiple objects
results = ray.get([obj_ref1, obj_ref2, obj_ref3])

# Timeout
try:
    result = ray.get(obj_ref, timeout=10)
except ray.exceptions.GetTimeoutError:
    print("Timeout waiting for result")
```

### Object Pinning

```python
# Object stays in memory until dereferenced
obj_ref = ray.put(large_array)

# Pass to multiple tasks
futures = [process.remote(obj_ref) for _ in range(100)]

# Object not copied, passed by reference
results = ray.get(futures)

# Object freed when no references remain
del obj_ref
```

### Object Spilling

Objects automatically spill to disk when object store is full:

```python
# Configure spilling
ray.init(
    object_store_memory=10 * 1024 * 1024 * 1024,  # 10GB
    _system_config={
        "object_spilling_config": json.dumps({
            "type": "filesystem",
            "params": {
                "directory_path": "/tmp/ray_spill"
            }
        })
    }
)
```

### Object Metadata

```python
# Check object location
obj_ref = ray.put([1, 2, 3])
metadata = ray.experimental.get_object_locations(obj_ref)

# Object size
size = ray.experimental.get_object_size(obj_ref)
```

## Resource Management

### Specifying Resources

```python
# Task resources
@ray.remote(num_cpus=2, num_gpus=1, memory=512*1024*1024)
def resource_task():
    return work()

# Actor resources
@ray.remote(num_cpus=4, num_gpus=2)
class ResourceActor:
    def method(self):
        return computation()

# Custom resources
ray.init(resources={"special_hardware": 4})

@ray.remote(resources={"special_hardware": 1})
def special_task():
    return use_special_hardware()
```

### Fractional Resources

```python
# Use 0.5 CPUs
@ray.remote(num_cpus=0.5)
def light_task():
    return quick_work()

# Schedule 2 tasks per CPU
futures = [light_task.remote() for _ in range(16)]  # 8 CPUs total
```

### Placement Groups

```python
from ray.util.placement_group import placement_group, remove_placement_group

# Create placement group
pg = placement_group([
    {"CPU": 2, "GPU": 1},  # Bundle 0
    {"CPU": 2, "GPU": 1},  # Bundle 1
    {"CPU": 4}             # Bundle 2
], strategy="STRICT_PACK")  # Co-locate on same node

# Wait for creation
ray.get(pg.ready())

# Use placement group
@ray.remote(num_cpus=2, num_gpus=1)
def task():
    return work()

# Schedule to specific bundle
futures = [
    task.options(placement_group=pg, placement_group_bundle_index=0).remote(),
    task.options(placement_group=pg, placement_group_bundle_index=1).remote()
]

# Cleanup
remove_placement_group(pg)
```

**Placement Strategies:**
- `STRICT_PACK`: All bundles on same node
- `PACK`: Best-effort packing
- `STRICT_SPREAD`: Each bundle on different node
- `SPREAD`: Best-effort spreading

## Advanced Features

### Ray Workflows

```python
from ray import workflow

@workflow.step
def process_step(data):
    return transform(data)

@workflow.step
def save_step(data):
    write_to_database(data)
    return "saved"

# Create workflow
data = load_data()
processed = process_step.bind(data)
result = save_step.bind(processed)

# Execute (durable, fault-tolerant)
output = workflow.run(result, workflow_id="my_workflow")
```

### Ray Queues

```python
from ray.util.queue import Queue

queue = Queue(maxsize=100)

# Producer
@ray.remote
def producer(queue):
    for i in range(10):
        queue.put(i)

# Consumer
@ray.remote
def consumer(queue):
    results = []
    for _ in range(10):
        results.append(queue.get())
    return results

# Run
p = producer.remote(queue)
c = consumer.remote(queue)
results = ray.get(c)
```

### Dynamic Remote Functions

```python
def create_remote_function(multiplier):
    def dynamic_func(x):
        return x * multiplier

    return ray.remote(dynamic_func)

# Create different versions
double = create_remote_function(2)
triple = create_remote_function(3)

# Use
result1 = ray.get(double.remote(5))  # 10
result2 = ray.get(triple.remote(5))  # 15
```

## Error Handling

### Task Exceptions

```python
@ray.remote
def failing_task():
    raise ValueError("Something went wrong")

future = failing_task.remote()

try:
    result = ray.get(future)
except ray.exceptions.RayTaskError as e:
    print(f"Task failed: {e}")
```

### Retry on Failure

```python
@ray.remote(max_retries=3, retry_exceptions=[ConnectionError])
def unstable_task():
    if random.random() < 0.5:
        raise ConnectionError("Network error")
    return "success"

result = ray.get(unstable_task.remote())
```

### Actor Failures

```python
@ray.remote(max_restarts=3)
class FaultTolerantActor:
    def __init__(self):
        self.state = 0

    def increment(self):
        self.state += 1
        return self.state

actor = FaultTolerantActor.remote()

# Actor auto-restarts on failure (state is lost)
```

## Performance Optimization

### Avoid Ray Overhead for Small Tasks

```python
# Bad: Too many small tasks
futures = [tiny_task.remote(i) for i in range(1000000)]

# Good: Batch into larger tasks
def batch_task(items):
    return [tiny_operation(i) for i in items]

batches = [list(range(i, i+1000)) for i in range(0, 1000000, 1000)]
futures = [batch_task.remote(batch) for batch in batches]
```

### Reduce Object Transfers

```python
# Bad: Large object passed repeatedly
large_data = load_large_dataset()
futures = [process.remote(large_data) for _ in range(100)]

# Good: Put in object store once
obj_ref = ray.put(large_data)
futures = [process.remote(obj_ref) for _ in range(100)]
```

### Use Actors for Stateful Computation

```python
# Bad: Reload model for each task
@ray.remote
def predict(data):
    model = load_model()  # Expensive
    return model.predict(data)

# Good: Load once in actor
@ray.remote
class Predictor:
    def __init__(self):
        self.model = load_model()  # Once

    def predict(self, data):
        return self.model.predict(data)

predictor = Predictor.remote()
futures = [predictor.predict.remote(d) for d in dataset]
```

## Best Practices

1. **Use tasks for stateless operations**: Prefer tasks when no state needed
2. **Use actors for stateful services**: Database connections, models, caches
3. **Pass ObjectRefs, not values**: Avoid copying large objects
4. **Batch small operations**: Reduce scheduling overhead
5. **Specify resources**: Help scheduler make better decisions
6. **Use placement groups**: Co-locate related computations
7. **Monitor object store**: Prevent memory issues
8. **Handle failures**: Use retries and fault tolerance
9. **Profile with dashboard**: Identify bottlenecks
10. **Start local, scale to cluster**: Code runs unchanged

## Common Pitfalls

- **Creating too many small tasks**: Overhead dominates computation
- **Not using ray.put() for large objects**: Unnecessary copying
- **Blocking in actor methods**: Serializes all operations
- **Not specifying resources**: Poor scheduling decisions
- **Ignoring object store pressure**: Out-of-memory errors
- **Calling ray.get() in loops**: Blocks parallelism
- **Creating actors in tasks**: Leads to nested actors
- **Not handling exceptions**: Silent failures
- **Forgetting ray.init()**: Tasks run locally, slowly
- **Over-provisioning resources**: Wastes cluster capacity
