# Ray Train and Ray Serve Reference

Sources:
- [Ray Train Documentation](https://docs.ray.io/en/latest/train/train.html)
- [Ray Serve Documentation](https://docs.ray.io/en/latest/serve/index.html)
- [Ray Train API](https://docs.ray.io/en/latest/train/api.html)
- [Ray Serve API](https://docs.ray.io/en/latest/serve/api/index.html)

## Ray Train

### Overview

Ray Train makes distributed model training simple. It abstracts away the complexity of setting up distributed training across popular frameworks like PyTorch and TensorFlow.

### Installation

```bash
# Install Ray Train
pip install -U "ray[train]"

# With specific frameworks
pip install -U "ray[train]" torch
pip install -U "ray[train]" tensorflow
```

## PyTorch Distributed Training

### Basic Setup

```python
from ray.train.torch import TorchTrainer
from ray.train import ScalingConfig
import ray.train

def train_func(config):
    import torch
    import torch.nn as nn

    # Create model
    model = nn.Linear(10, 1)

    # Prepare for distributed training
    model = ray.train.torch.prepare_model(model)

    # Create data loader
    train_dataset = get_dataset()
    train_loader = torch.utils.data.DataLoader(train_dataset, batch_size=32)
    train_loader = ray.train.torch.prepare_data_loader(train_loader)

    # Optimizer
    optimizer = torch.optim.Adam(model.parameters(), lr=config["lr"])

    # Training loop
    for epoch in range(config["num_epochs"]):
        for batch in train_loader:
            loss = train_step(model, batch, optimizer)

        # Report metrics
        ray.train.report({"loss": loss, "epoch": epoch})

# Configure training
trainer = TorchTrainer(
    train_func,
    train_loop_config={"num_epochs": 10, "lr": 0.001},
    scaling_config=ScalingConfig(
        num_workers=4,
        use_gpu=True,
        resources_per_worker={"CPU": 2, "GPU": 1}
    )
)

# Run training
result = trainer.fit()
print(result.metrics)
```

### Advanced PyTorch

```python
def train_func(config):
    import torch
    import torch.nn as nn
    from torch.nn.parallel import DistributedDataParallel

    # Get distributed context
    rank = ray.train.get_context().get_world_rank()
    local_rank = ray.train.get_context().get_local_rank()
    world_size = ray.train.get_context().get_world_size()

    # Model
    model = create_model()
    model = model.to(f"cuda:{local_rank}")
    model = DistributedDataParallel(model)

    # Or use prepare_model (simpler)
    model = ray.train.torch.prepare_model(model)

    # Data
    train_dataset = ray.train.get_dataset_shard("train")

    # Training loop
    for epoch in range(config["num_epochs"]):
        for batch in train_dataset.iter_torch_batches(batch_size=32):
            loss = train_step(model, batch)

        # Checkpoint
        checkpoint = ray.train.Checkpoint.from_dict({
            "model_state": model.state_dict(),
            "epoch": epoch
        })
        ray.train.report({"loss": loss}, checkpoint=checkpoint)

trainer = TorchTrainer(
    train_func,
    train_loop_config={"num_epochs": 20},
    scaling_config=ScalingConfig(num_workers=8, use_gpu=True),
    datasets={"train": train_dataset}
)

result = trainer.fit()
```

## TensorFlow Distributed Training

### Basic TensorFlow

```python
from ray.train.tensorflow import TensorflowTrainer
from ray.train import ScalingConfig
import ray.train

def train_func(config):
    import tensorflow as tf

    # Multi-worker strategy
    strategy = tf.distribute.MultiWorkerMirroredStrategy()

    with strategy.scope():
        model = tf.keras.Sequential([
            tf.keras.layers.Dense(128, activation='relu'),
            tf.keras.layers.Dense(10, activation='softmax')
        ])

        model.compile(
            optimizer=tf.keras.optimizers.Adam(config["lr"]),
            loss='sparse_categorical_crossentropy',
            metrics=['accuracy']
        )

    # Get dataset
    train_dataset = ray.train.get_dataset_shard("train")
    tf_dataset = train_dataset.to_tf(
        feature_columns=["features"],
        label_column="label",
        batch_size=32
    )

    # Custom callback for reporting
    class ReportCallback(tf.keras.callbacks.Callback):
        def on_epoch_end(self, epoch, logs=None):
            ray.train.report(logs)

    # Train
    model.fit(
        tf_dataset,
        epochs=config["num_epochs"],
        callbacks=[ReportCallback()]
    )

trainer = TensorflowTrainer(
    train_func,
    train_loop_config={"num_epochs": 10, "lr": 0.001},
    scaling_config=ScalingConfig(num_workers=4, use_gpu=True),
    datasets={"train": train_dataset}
)

result = trainer.fit()
```

## Checkpointing

### Saving Checkpoints

```python
def train_func(config):
    model = create_model()

    for epoch in range(config["num_epochs"]):
        loss = train_epoch(model)

        # Create checkpoint
        checkpoint = ray.train.Checkpoint.from_dict({
            "model_state": model.state_dict(),
            "epoch": epoch,
            "config": config
        })

        # Report with checkpoint
        ray.train.report(
            {"loss": loss, "epoch": epoch},
            checkpoint=checkpoint
        )

# Configure checkpointing
from ray.train import CheckpointConfig

trainer = TorchTrainer(
    train_func,
    checkpoint_config=CheckpointConfig(
        num_to_keep=3,  # Keep 3 best checkpoints
        checkpoint_score_attribute="loss",
        checkpoint_score_order="min"
    ),
    scaling_config=ScalingConfig(num_workers=4, use_gpu=True)
)

result = trainer.fit()
```

### Loading Checkpoints

```python
# Get best checkpoint
checkpoint = result.checkpoint

# Load model
with checkpoint.as_directory() as checkpoint_dir:
    import torch
    model = create_model()
    checkpoint_dict = torch.load(f"{checkpoint_dir}/checkpoint.pt")
    model.load_state_dict(checkpoint_dict["model_state"])
```

### Resuming Training

```python
# Resume from checkpoint
trainer = TorchTrainer(
    train_func,
    resume_from_checkpoint=checkpoint,
    scaling_config=ScalingConfig(num_workers=4)
)

result = trainer.fit()
```

## Hyperparameter Tuning with Train

```python
from ray import tune
from ray.tune import Tuner

def train_func(config):
    # Training code
    for epoch in range(10):
        loss = train_epoch(config["lr"], config["batch_size"])
        ray.train.report({"loss": loss})

# Define parameter space
param_space = {
    "train_loop_config": {
        "lr": tune.loguniform(1e-4, 1e-1),
        "batch_size": tune.choice([16, 32, 64])
    }
}

trainer = TorchTrainer(
    train_func,
    scaling_config=ScalingConfig(num_workers=2, use_gpu=True)
)

tuner = Tuner(
    trainer,
    param_space=param_space,
    tune_config=tune.TuneConfig(num_samples=10)
)

results = tuner.fit()
best_result = results.get_best_result()
```

---

## Ray Serve

### Overview

Ray Serve is a scalable model serving library for building online inference APIs. According to the documentation, it is "framework-agnostic" and enables deployment of deep learning models alongside business logic.

### Installation

```bash
# Install Ray Serve
pip install -U "ray[serve]"
```

## Basic Deployment

### Simple Deployment

```python
from ray import serve
import ray

ray.init()
serve.start()

@serve.deployment
class Predictor:
    def __init__(self, model_path="model.pkl"):
        import pickle
        with open(model_path, "rb") as f:
            self.model = pickle.load(f)

    async def __call__(self, request):
        data = await request.json()
        prediction = self.model.predict([data["features"]])
        return {"prediction": prediction[0]}

# Deploy
serve.run(Predictor.bind(), route_prefix="/predict")

# Test
import requests
response = requests.post(
    "http://localhost:8000/predict",
    json={"features": [1, 2, 3, 4, 5]}
)
print(response.json())
```

### Deployment Configuration

```python
@serve.deployment(
    num_replicas=4,
    ray_actor_options={"num_cpus": 2, "num_gpus": 1}
)
class GPUModel:
    def __init__(self):
        import torch
        self.model = torch.load("model.pt").cuda()

    async def __call__(self, request):
        data = await request.json()
        result = self.model(data["input"])
        return {"result": result.tolist()}

serve.run(GPUModel.bind())
```

## FastAPI Integration

### FastAPI Deployment

```python
from ray import serve
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Input(BaseModel):
    text: str

class Output(BaseModel):
    prediction: str

@serve.deployment(num_replicas=2)
@serve.ingress(app)
class TextClassifier:
    def __init__(self):
        self.model = load_model()

    @app.post("/classify", response_model=Output)
    async def classify(self, input: Input):
        prediction = self.model.predict(input.text)
        return Output(prediction=prediction)

serve.run(TextClassifier.bind())
```

### Multiple Endpoints

```python
from fastapi import FastAPI
from ray import serve

app = FastAPI()

@serve.deployment
@serve.ingress(app)
class MultiEndpoint:
    def __init__(self):
        self.model_a = load_model_a()
        self.model_b = load_model_b()

    @app.post("/model-a")
    async def predict_a(self, data: dict):
        return {"result": self.model_a.predict(data["input"])}

    @app.post("/model-b")
    async def predict_b(self, data: dict):
        return {"result": self.model_b.predict(data["input"])}

    @app.get("/health")
    async def health(self):
        return {"status": "healthy"}

serve.run(MultiEndpoint.bind())
```

## Model Composition

### Sequential Models

```python
@serve.deployment
class Preprocessor:
    def preprocess(self, data):
        return normalize(data)

@serve.deployment
class Model:
    def __init__(self, preprocessor):
        self.preprocessor = preprocessor
        self.model = load_model()

    async def __call__(self, request):
        data = await request.json()
        processed = await self.preprocessor.preprocess.remote(data)
        prediction = self.model.predict(processed)
        return {"prediction": prediction}

preprocessor = Preprocessor.bind()
model = Model.bind(preprocessor)

serve.run(model, route_prefix="/predict")
```

### Ensemble Models

```python
@serve.deployment
class ModelA:
    def predict(self, data):
        return self.model.predict(data)

@serve.deployment
class ModelB:
    def predict(self, data):
        return self.model.predict(data)

@serve.deployment
class Ensemble:
    def __init__(self, model_a, model_b):
        self.model_a = model_a
        self.model_b = model_b

    async def __call__(self, request):
        data = await request.json()

        # Run models in parallel
        pred_a = await self.model_a.predict.remote(data)
        pred_b = await self.model_b.predict.remote(data)

        # Combine predictions
        result = (pred_a + pred_b) / 2
        return {"prediction": result}

model_a = ModelA.bind()
model_b = ModelB.bind()
ensemble = Ensemble.bind(model_a, model_b)

serve.run(ensemble)
```

## Scaling and Autoscaling

### Manual Scaling

```python
@serve.deployment(
    num_replicas=8,  # 8 replicas
    max_ongoing_requests=10  # Max 10 concurrent requests per replica
)
class ScalableModel:
    async def __call__(self, request):
        return process(request)

serve.run(ScalableModel.bind())
```

### Autoscaling

```python
@serve.deployment(
    autoscaling_config={
        "min_replicas": 1,
        "max_replicas": 10,
        "target_ongoing_requests": 5
    }
)
class AutoscalingModel:
    async def __call__(self, request):
        return process(request)

serve.run(AutoscalingModel.bind())
```

## Batching

### Request Batching

```python
@serve.deployment
class BatchedModel:
    def __init__(self):
        self.model = load_model()

    @serve.batch(max_batch_size=32, batch_wait_timeout_s=0.1)
    async def predict_batch(self, inputs):
        # inputs is a list of requests
        predictions = self.model.predict_batch(inputs)
        return predictions

    async def __call__(self, request):
        data = await request.json()
        prediction = await self.predict_batch(data["input"])
        return {"prediction": prediction}

serve.run(BatchedModel.bind())
```

## Deployment Management

### List Deployments

```python
from ray import serve

# Get all deployments
deployments = serve.list_deployments()
for name, deployment in deployments.items():
    print(f"{name}: {deployment}")
```

### Update Deployment

```python
# Update configuration
@serve.deployment(num_replicas=8)
class UpdatedModel:
    async def __call__(self, request):
        return new_logic(request)

serve.run(UpdatedModel.bind(), name="my-model")
```

### Delete Deployment

```python
serve.delete("deployment-name")
```

## Production Patterns

### Health Checks

```python
from fastapi import FastAPI
from ray import serve

app = FastAPI()

@serve.deployment
@serve.ingress(app)
class Production:
    def __init__(self):
        self.model = load_model()
        self.ready = True

    @app.get("/health")
    async def health(self):
        return {"status": "healthy" if self.ready else "unhealthy"}

    @app.post("/predict")
    async def predict(self, data: dict):
        if not self.ready:
            return {"error": "Service not ready"}
        return {"prediction": self.model.predict(data["input"])}

serve.run(Production.bind())
```

### Logging and Monitoring

```python
import logging
from ray import serve

logger = logging.getLogger("ray.serve")

@serve.deployment
class MonitoredModel:
    def __init__(self):
        self.model = load_model()
        self.request_count = 0

    async def __call__(self, request):
        self.request_count += 1
        logger.info(f"Request {self.request_count}")

        data = await request.json()
        prediction = self.model.predict(data["input"])

        logger.info(f"Prediction: {prediction}")
        return {"prediction": prediction}

serve.run(MonitoredModel.bind())
```

## Best Practices

### Ray Train

1. **Use prepare_model and prepare_data_loader**: Simplifies distributed setup
2. **Report metrics regularly**: Enable monitoring and checkpointing
3. **Checkpoint frequently**: Recover from failures
4. **Use Ray Data for loading**: Efficient distributed data loading
5. **Specify resources**: Ensure proper GPU allocation
6. **Start with small scale**: Test locally before scaling
7. **Profile training**: Identify bottlenecks
8. **Use Tune for HPO**: Automate hyperparameter search

### Ray Serve

1. **Use FastAPI integration**: Better request handling
2. **Enable batching**: Improve throughput
3. **Implement health checks**: Monitor service status
4. **Use autoscaling**: Handle variable load
5. **Compose models**: Build complex pipelines
6. **Log requests**: Debug and monitor
7. **Set resource limits**: Prevent OOM errors
8. **Test locally first**: Validate before deployment

## Common Pitfalls

### Ray Train

- **Not using prepare_model**: Manual distributed setup is complex
- **Forgetting ray.train.report()**: No metrics or checkpoints
- **Not specifying num_workers**: Defaults to 1 worker
- **Calling .compute() in training**: Blocks distributed execution
- **Not using Ray Data**: Inefficient data loading
- **Over-allocating resources**: Wastes cluster capacity

### Ray Serve

- **Blocking in deployment**: Use async methods
- **Not batching requests**: Poor GPU utilization
- **Too many replicas**: Memory issues
- **Not handling errors**: Service crashes
- **Ignoring autoscaling**: Manual scaling overhead
- **Large model in memory**: Use lazy loading
- **Not testing under load**: Production issues
