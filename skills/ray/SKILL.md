---
name: ray
description: Provides Ray distributed computing framework for scaling Python applications. Covers Ray Core (tasks, actors, objects), Ray Data (data processing), Ray Train (distributed training), Ray Serve (model serving), Ray Tune (hyperparameter tuning), and Ray RLlib (reinforcement learning).
license: MIT
metadata:
  author: Agent Cluster
  tags: ray, distributed-computing, machine-learning, data-processing, model-serving, hyperparameter-tuning, reinforcement-learning, pytorch, tensorflow, gpu-acceleration
---

# Ray Agent Skill

Agent-optimized skill for Ray distributed computing framework.

## Quick Reference

### Ray Core - Tasks and Actors

```python
import ray
ray.init()

# Task (remote function)
@ray.remote
def square(x):
    return x ** 2

future = square.remote(4)
result = ray.get(future)  # 16

# Parallel tasks
futures = [square.remote(i) for i in range(100)]
results = ray.get(futures)

# Actor (remote class)
@ray.remote
class Counter:
    def __init__(self):
        self.count = 0
    def increment(self):
        self.count += 1
        return self.count

counter = Counter.remote()
result = ray.get(counter.increment.remote())
```

### Ray Data - Distributed Data Processing

```python
import ray

# Read data
ds = ray.data.read_parquet("s3://bucket/data/*.parquet")
ds = ray.data.read_csv("data/*.csv")
ds = ray.data.read_json("data/*.json")

# Transform data
def preprocess(batch):
    batch["new_col"] = batch["col1"] * 2
    return batch

ds = ds.map_batches(preprocess, batch_format="pandas")

# Filter
ds = ds.filter(lambda row: row["value"] > 10)

# Write results
ds.write_parquet("output/")
ds.write_csv("output/")

# Batch inference
def predict(batch):
    # Model inference
    return {"predictions": model.predict(batch["features"])}

predictions = ds.map_batches(predict, batch_size=32)
```

### Ray Train - Distributed Training

```python
import ray
from ray import train
from ray.train import ScalingConfig
from ray.train.torch import TorchTrainer

def train_func(config):
    # Your training code
    model = create_model()
    train_dataset = train.get_dataset_shard("train")

    for epoch in range(config["num_epochs"]):
        # Training loop
        loss = train_epoch(model, train_dataset)
        train.report({"loss": loss})

# Configure distributed training
trainer = TorchTrainer(
    train_func,
    train_loop_config={"num_epochs": 10, "lr": 0.001},
    scaling_config=ScalingConfig(
        num_workers=4,
        use_gpu=True,
        resources_per_worker={"CPU": 2, "GPU": 1}
    ),
    datasets={"train": train_dataset}
)

result = trainer.fit()
```

### Ray Serve - Model Serving

```python
from ray import serve
import ray

ray.init()
serve.start()

# Simple deployment
@serve.deployment
class Predictor:
    def __init__(self, model_path):
        self.model = load_model(model_path)

    async def __call__(self, request):
        data = await request.json()
        prediction = self.model.predict(data["input"])
        return {"prediction": prediction}

# Deploy
serve.run(Predictor.bind("model.pkl"), route_prefix="/predict")

# FastAPI integration
from fastapi import FastAPI

app = FastAPI()

@serve.deployment
@serve.ingress(app)
class FastAPIDeployment:
    @app.post("/predict")
    async def predict(self, data: dict):
        return {"result": model.predict(data["input"])}

serve.run(FastAPIDeployment.bind())
```

### Ray Tune - Hyperparameter Tuning

```python
from ray import tune
from ray.tune import Tuner
from ray.tune.search.optuna import OptunaSearch

def objective(config):
    # Training function
    for epoch in range(10):
        loss = train_step(config["lr"], config["batch_size"])
        tune.report(loss=loss)

# Define search space
param_space = {
    "lr": tune.loguniform(1e-4, 1e-1),
    "batch_size": tune.choice([16, 32, 64, 128]),
    "hidden_units": tune.randint(32, 512)
}

# Run tuning
tuner = Tuner(
    objective,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        num_samples=100,
        search_alg=OptunaSearch()
    )
)

results = tuner.fit()
best_config = results.get_best_result().config
```

## Common Patterns

### Distributed Data Processing

```python
import ray

# Load and process large dataset
ds = ray.data.read_parquet("s3://bucket/data/*.parquet")

# Parallel preprocessing
ds = ds.map_batches(
    lambda batch: preprocess(batch),
    batch_format="pandas",
    num_cpus=1
)

# Repartition for better parallelism
ds = ds.repartition(100)

# Aggregate
stats = ds.aggregate(
    count=ray.data.aggregate.Count(),
    mean=ray.data.aggregate.Mean("value"),
    std=ray.data.aggregate.Std("value")
)

# Write output
ds.write_parquet("output/", num_rows_per_file=10000)
```

### Task Dependencies and Actor Pools

```python
# Task graph
@ray.remote
def fetch(src):
    return load_data(src)

@ray.remote
def process(data):
    return transform(data)

data = [fetch.remote(s) for s in sources]
results = ray.get([process.remote(d) for d in data])

# Actor pool
@ray.remote
class Worker:
    def process(self, data):
        return computation(data)

pool = [Worker.remote() for _ in range(10)]
futures = [pool[i % len(pool)].process.remote(d) for i, d in enumerate(dataset)]
results = ray.get(futures)
```

### Multi-GPU Training

```python
from ray.train.torch import TorchTrainer
from ray.train import ScalingConfig

def train_func(config):
    import torch.distributed as dist

    # Automatic distributed setup
    model = create_model()
    model = ray.train.torch.prepare_model(model)

    train_loader = ray.train.torch.prepare_data_loader(train_loader)

    for epoch in range(config["epochs"]):
        for batch in train_loader:
            loss = train_step(model, batch)

        ray.train.report({"loss": loss})

trainer = TorchTrainer(
    train_func,
    train_loop_config={"epochs": 10},
    scaling_config=ScalingConfig(
        num_workers=4,
        use_gpu=True,
        resources_per_worker={"GPU": 1}
    )
)

result = trainer.fit()
```

### Batch Inference Pipeline

```python
import ray

# Load data
ds = ray.data.read_parquet("data/*.parquet")

# Load model in actors for efficient batching
class ModelInference:
    def __init__(self):
        self.model = load_model()

    def __call__(self, batch):
        predictions = self.model.predict(batch["features"])
        return {"predictions": predictions}

# Run batch inference
predictions = ds.map_batches(
    ModelInference,
    batch_size=32,
    num_gpus=1,
    concurrency=4
)

predictions.write_parquet("predictions/")
```

## Cluster Management

### Local Cluster

```python
import ray

# Auto-detect resources
ray.init()

# Manual configuration
ray.init(num_cpus=8, num_gpus=2)

# Connect to existing cluster
ray.init(address="ray://cluster-address:10001")
```

### Cloud Deployment

```bash
# AWS/GCP/Azure cluster
pip install -U "ray[default]"
ray up cluster.yaml  # See references/cluster-deployment.md for config
```

### Kubernetes Deployment

```yaml
# KubeRay cluster - see references/cluster-deployment.md for full spec
apiVersion: ray.io/v1
kind: RayCluster
spec:
  rayVersion: '2.53.0'
  headGroupSpec:
    replicas: 1
  workerGroupSpecs:
  - replicas: 3
    resources:
      nvidia.com/gpu: "1"
```

## Resource Management

### Specifying Resources

```python
# Task resources
@ray.remote(num_cpus=2, num_gpus=1, memory=1024*1024*1024)
def gpu_task(data):
    return process_on_gpu(data)

# Actor resources
@ray.remote(num_cpus=4, num_gpus=2)
class GPUActor:
    def process(self, data):
        return gpu_computation(data)

# Custom resources
ray.init(resources={"custom_resource": 4})

@ray.remote(resources={"custom_resource": 1})
def custom_task():
    return work()
```

### Object Store

```python
# Put objects explicitly
large_data = load_large_dataset()
obj_ref = ray.put(large_data)

# Pass to tasks
@ray.remote
def process(data_ref):
    data = ray.get(data_ref)
    return transform(data)

result = process.remote(obj_ref)
```

## Performance Optimization

### Batching

```python
# Ray Data batching
ds = ray.data.read_parquet("data.parquet")
ds = ds.map_batches(
    process_function,
    batch_size=1000,  # Process 1000 rows at a time
    num_cpus=2
)
```

### Pipelining

```python
# Pipeline stages
stage1 = ray.data.read_parquet("input.parquet")
stage2 = stage1.map_batches(preprocess)
stage3 = stage2.map_batches(feature_engineering)

# Execution is pipelined automatically
stage3.write_parquet("output/")
```

### Placement Groups

```python
from ray.util.placement_group import placement_group

# Co-locate tasks
pg = placement_group([
    {"CPU": 2, "GPU": 1},
    {"CPU": 2, "GPU": 1},
    {"CPU": 2, "GPU": 1}
], strategy="STRICT_PACK")

@ray.remote(num_cpus=2, num_gpus=1)
def task():
    return work()

# Submit to placement group
futures = [task.options(placement_group=pg).remote() for _ in range(3)]
```

## Monitoring and Debugging

### Ray Dashboard

```python
import ray

# Dashboard available at http://localhost:8265
ray.init()
```

### Logging

```python
import ray
import logging

logger = logging.getLogger(__name__)

@ray.remote
def task_with_logging():
    logger.info("Task started")
    result = work()
    logger.info(f"Task completed: {result}")
    return result
```

### Profiling

```python
# Enable profiling
ray.init(runtime_env={"env_vars": {"RAY_PROFILING": "1"}})

# View timeline in dashboard
```

## References

Detailed documentation in `references/`:

- **core-tasks-actors.md**: Ray Core primitives, tasks, actors, objects
- **data-processing.md**: Ray Data API, transformations, batch operations
- **train-serve.md**: Ray Train distributed training, Ray Serve model serving
- **tune-rllib.md**: Ray Tune hyperparameter tuning, Ray RLlib reinforcement learning
- **cluster-deployment.md**: Cluster setup, cloud deployment, Kubernetes, resource management

## Official Documentation

- [Ray Documentation](https://docs.ray.io)
- [Ray Core](https://docs.ray.io/en/latest/ray-core/walkthrough.html)
- [Ray Data](https://docs.ray.io/en/latest/data/data.html)
- [Ray Train](https://docs.ray.io/en/latest/train/train.html)
- [Ray Serve](https://docs.ray.io/en/latest/serve/index.html)
- [Ray Tune](https://docs.ray.io/en/latest/tune/index.html)
- [Ray RLlib](https://docs.ray.io/en/latest/rllib/index.html)

## Best Practices

1. **Use tasks for stateless operations**: Prefer tasks over actors when state not needed
2. **Batch operations**: Use Ray Data for efficient batch processing
3. **Resource specification**: Always specify CPU/GPU requirements
4. **Object store management**: Use ray.put() for large objects shared across tasks
5. **Monitor dashboard**: Track resource usage and task execution
6. **Placement groups**: Co-locate related tasks for better performance
7. **Fault tolerance**: Ray automatically retries failed tasks
8. **Scaling**: Start local, scale to cluster without code changes
9. **Data locality**: Process data where it lives to minimize transfers
10. **Profile first**: Use Ray dashboard to identify bottlenecks
