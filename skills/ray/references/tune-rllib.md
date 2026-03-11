# Ray Tune and RLlib Reference

Sources:
- [Ray Tune Documentation](https://docs.ray.io/en/latest/tune/index.html)
- [Ray Tune Getting Started](https://docs.ray.io/en/latest/tune/getting-started.html)
- [Ray Tune Search Algorithms](https://docs.ray.io/en/latest/tune/api/suggestion.html)
- [Ray RLlib Documentation](https://docs.ray.io/en/latest/rllib/index.html)

## Ray Tune

### Overview

Ray Tune is a Python library for experiment execution and hyperparameter tuning at any scale. According to the documentation, it integrates with "PyTorch, XGBoost, TensorFlow and Keras" and supports advanced algorithms including "Population Based Training (PBT) and HyperBand/ASHA."

### Installation

```bash
# Install Ray Tune
pip install -U "ray[tune]"

# With search libraries
pip install -U "ray[tune]" optuna bayesian-optimization hyperopt
```

## Basic Hyperparameter Tuning

### Simple Example

```python
from ray import tune
from ray.tune import Tuner

def objective(config):
    score = config["a"] ** 2 + config["b"]
    return {"score": score}

# Define search space
param_space = {
    "a": tune.uniform(0, 10),
    "b": tune.choice([1, 2, 3])
}

# Create tuner
tuner = Tuner(
    objective,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        num_samples=20,  # 20 trials
        metric="score",
        mode="min"
    )
)

# Run tuning
results = tuner.fit()

# Get best result
best_result = results.get_best_result()
print(f"Best config: {best_result.config}")
print(f"Best score: {best_result.metrics}")
```

### Training Function

```python
from ray import tune

def train_model(config):
    # Setup
    model = create_model(config["hidden_units"])
    optimizer = get_optimizer(config["lr"])

    # Training loop
    for epoch in range(10):
        loss = train_epoch(model, optimizer, config["batch_size"])
        accuracy = evaluate(model)

        # Report metrics
        tune.report(loss=loss, accuracy=accuracy)

# Search space
param_space = {
    "lr": tune.loguniform(1e-4, 1e-1),
    "batch_size": tune.choice([16, 32, 64, 128]),
    "hidden_units": tune.randint(32, 512)
}

tuner = Tuner(
    train_model,
    param_space=param_space,
    tune_config=tune.TuneConfig(num_samples=100)
)

results = tuner.fit()
```

## Search Spaces

### Continuous Distributions

```python
from ray import tune

param_space = {
    # Uniform distribution
    "lr": tune.uniform(0.001, 0.1),

    # Log-uniform distribution
    "lr": tune.loguniform(1e-5, 1e-1),

    # Normal distribution
    "noise": tune.randn(0, 0.1),  # mean, std

    # Log-normal distribution
    "multiplier": tune.lognormal(1, 0.5)
}
```

### Discrete Distributions

```python
param_space = {
    # Choice from list
    "activation": tune.choice(["relu", "tanh", "sigmoid"]),

    # Random integer
    "num_layers": tune.randint(1, 10),  # [1, 10)

    # Quantized uniform
    "dropout": tune.quniform(0.0, 0.5, 0.05),  # 0.0, 0.05, 0.1, ..., 0.5

    # Quantized log-uniform
    "lr": tune.qloguniform(1e-5, 1e-1, 5e-5)
}
```

### Grid Search

```python
param_space = {
    "lr": tune.grid_search([0.001, 0.01, 0.1]),
    "batch_size": tune.grid_search([16, 32, 64])
}
# Total: 3 * 3 = 9 trials
```

## Search Algorithms

### Random Search (Default)

```python
tuner = Tuner(
    objective,
    param_space=param_space,
    tune_config=tune.TuneConfig(num_samples=100)
)
```

### Optuna

```python
from ray.tune.search.optuna import OptunaSearch

search_alg = OptunaSearch(
    metric="accuracy",
    mode="max"
)

tuner = Tuner(
    objective,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        search_alg=search_alg,
        num_samples=100
    )
)
```

### Bayesian Optimization

```python
from ray.tune.search.bayesopt import BayesOptSearch

search_alg = BayesOptSearch(
    metric="loss",
    mode="min"
)

tuner = Tuner(
    objective,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        search_alg=search_alg,
        num_samples=50
    )
)
```

### Hyperopt

```python
from ray.tune.search.hyperopt import HyperOptSearch

search_alg = HyperOptSearch(
    metric="score",
    mode="max"
)

tuner = Tuner(
    objective,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        search_alg=search_alg,
        num_samples=100
    )
)
```

## Schedulers

### ASHA (Async Successive Halving)

```python
from ray.tune.schedulers import ASHAScheduler

scheduler = ASHAScheduler(
    time_attr="training_iteration",
    metric="loss",
    mode="min",
    max_t=100,  # Max iterations
    grace_period=10,  # Min iterations before stopping
    reduction_factor=3
)

tuner = Tuner(
    train_model,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        scheduler=scheduler,
        num_samples=100
    )
)
```

### Population Based Training (PBT)

```python
from ray.tune.schedulers import PopulationBasedTraining

scheduler = PopulationBasedTraining(
    time_attr="training_iteration",
    metric="accuracy",
    mode="max",
    perturbation_interval=5,
    hyperparam_mutations={
        "lr": tune.loguniform(1e-4, 1e-1),
        "momentum": [0.8, 0.9, 0.95, 0.99]
    }
)

tuner = Tuner(
    train_model,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        scheduler=scheduler,
        num_samples=10
    )
)
```

### Hyperband

```python
from ray.tune.schedulers import HyperBandScheduler

scheduler = HyperBandScheduler(
    time_attr="training_iteration",
    metric="loss",
    mode="min",
    max_t=100
)

tuner = Tuner(
    train_model,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        scheduler=scheduler,
        num_samples=100
    )
)
```

## Checkpointing in Tune

### Saving Checkpoints

```python
from ray.air import session
from ray.air.checkpoint import Checkpoint

def train_model(config):
    model = create_model(config)

    for epoch in range(100):
        loss = train_epoch(model)

        # Create checkpoint
        checkpoint = Checkpoint.from_dict({
            "model_state": model.state_dict(),
            "epoch": epoch
        })

        # Report with checkpoint
        session.report(
            {"loss": loss},
            checkpoint=checkpoint
        )

tuner = Tuner(
    train_model,
    param_space=param_space,
    run_config=RunConfig(
        checkpoint_config=CheckpointConfig(
            num_to_keep=3,  # Keep 3 best
            checkpoint_score_attribute="loss",
            checkpoint_score_order="min"
        )
    )
)
```

### Loading Checkpoints

```python
# Get best result
best_result = results.get_best_result()

# Load checkpoint
checkpoint = best_result.checkpoint
with checkpoint.as_directory() as checkpoint_dir:
    model = load_model_from_checkpoint(checkpoint_dir)
```

## Analysis and Results

### Result Grid

```python
results = tuner.fit()

# Get all results
df = results.get_dataframe()
print(df[["config/lr", "config/batch_size", "loss", "accuracy"]])

# Get best result
best_result = results.get_best_result(metric="accuracy", mode="max")
print(f"Best config: {best_result.config}")
print(f"Best metrics: {best_result.metrics}")

# Filter results
filtered = results.filter(lambda result: result.metrics["accuracy"] > 0.9)
```

### TensorBoard Integration

```python
from ray.tune.logger import TBXLoggerCallback

tuner = Tuner(
    train_model,
    param_space=param_space,
    run_config=RunConfig(
        callbacks=[TBXLoggerCallback()]
    )
)

results = tuner.fit()
# View in TensorBoard: tensorboard --logdir ~/ray_results
```

## Distributed Tuning

### Resource Allocation

```python
# Allocate resources per trial
tuner = Tuner(
    train_model,
    param_space=param_space,
    tune_config=tune.TuneConfig(
        num_samples=100
    ),
    run_config=RunConfig(
        resources_per_trial={"cpu": 2, "gpu": 1}
    )
)

# If cluster has 8 GPUs, runs 8 trials concurrently
results = tuner.fit()
```

### Multi-Node Trials

```python
from ray.train import ScalingConfig
from ray.train.torch import TorchTrainer

def train_func(config):
    # Distributed training code
    pass

# Each trial uses 4 workers
trainer = TorchTrainer(
    train_func,
    scaling_config=ScalingConfig(
        num_workers=4,
        use_gpu=True
    )
)

tuner = Tuner(
    trainer,
    param_space={"train_loop_config": {"lr": tune.loguniform(1e-4, 1e-1)}},
    tune_config=tune.TuneConfig(num_samples=10)
)

results = tuner.fit()
```

---

## Ray RLlib

### Overview

Ray RLlib is a scalable reinforcement learning library that provides implementations of popular RL algorithms with support for distributed training.

### Installation

```bash
# Install RLlib
pip install -U "ray[rllib]"

# With specific environments
pip install -U "ray[rllib]" gymnasium torch
```

## Basic RL Training

### Simple Example

```python
from ray.rllib.algorithms.ppo import PPOConfig

# Configure algorithm
config = (
    PPOConfig()
    .environment("CartPole-v1")
    .framework("torch")
    .training(
        lr=0.0001,
        train_batch_size=4000,
        sgd_minibatch_size=128,
        num_sgd_iter=30
    )
    .resources(num_gpus=1)
    .rollouts(num_rollout_workers=4)
)

# Build algorithm
algo = config.build()

# Train
for i in range(100):
    result = algo.train()
    print(f"Iteration {i}: reward={result['episode_reward_mean']}")

# Save checkpoint
checkpoint = algo.save()
print(f"Checkpoint saved at {checkpoint}")
```

### Supported Algorithms

```python
# PPO (Proximal Policy Optimization)
from ray.rllib.algorithms.ppo import PPOConfig
algo = PPOConfig().build(env="CartPole-v1")

# DQN (Deep Q-Network)
from ray.rllib.algorithms.dqn import DQNConfig
algo = DQNConfig().build(env="CartPole-v1")

# SAC (Soft Actor-Critic)
from ray.rllib.algorithms.sac import SACConfig
algo = SACConfig().build(env="Pendulum-v1")

# A3C (Asynchronous Advantage Actor-Critic)
from ray.rllib.algorithms.a3c import A3CConfig
algo = A3CConfig().build(env="CartPole-v1")

# IMPALA
from ray.rllib.algorithms.impala import ImpalaConfig
algo = ImpalaConfig().build(env="CartPole-v1")
```

## Custom Environments

### Gymnasium Environment

```python
import gymnasium as gym
from ray.rllib.algorithms.ppo import PPOConfig

class CustomEnv(gym.Env):
    def __init__(self, config):
        self.action_space = gym.spaces.Discrete(2)
        self.observation_space = gym.spaces.Box(0, 1, shape=(4,))

    def reset(self, *, seed=None, options=None):
        return self.observation_space.sample(), {}

    def step(self, action):
        obs = self.observation_space.sample()
        reward = 1.0
        terminated = False
        truncated = False
        return obs, reward, terminated, truncated, {}

# Register environment
from ray.tune.registry import register_env

register_env("custom_env", lambda config: CustomEnv(config))

# Train
config = PPOConfig().environment("custom_env")
algo = config.build()
```

## Distributed Training

### Multi-Worker Rollouts

```python
config = (
    PPOConfig()
    .environment("CartPole-v1")
    .rollouts(
        num_rollout_workers=16,  # 16 parallel workers
        num_envs_per_worker=4   # 4 envs per worker
    )
)

algo = config.build()
```

### Multi-GPU Training

```python
config = (
    PPOConfig()
    .environment("Atari-v1")
    .resources(
        num_gpus=4,  # Use 4 GPUs for training
        num_cpus_per_worker=2
    )
)

algo = config.build()
```

## Hyperparameter Tuning for RL

### Tune RLlib Algorithms

```python
from ray import tune
from ray.rllib.algorithms.ppo import PPOConfig

# Define config function
def get_config(config):
    return (
        PPOConfig()
        .environment("CartPole-v1")
        .training(
            lr=config["lr"],
            train_batch_size=config["train_batch_size"]
        )
    )

# Parameter space
param_space = {
    "lr": tune.loguniform(1e-5, 1e-3),
    "train_batch_size": tune.choice([2000, 4000, 8000])
}

# Run tuning
tuner = tune.Tuner(
    "PPO",
    param_space={
        "env": "CartPole-v1",
        "lr": tune.loguniform(1e-5, 1e-3),
        "train_batch_size": tune.choice([2000, 4000, 8000])
    },
    run_config=RunConfig(stop={"training_iteration": 100})
)

results = tuner.fit()
```

### Population Based Training with RLlib

```python
from ray.tune.schedulers import PopulationBasedTraining

pbt = PopulationBasedTraining(
    time_attr="training_iteration",
    metric="episode_reward_mean",
    mode="max",
    perturbation_interval=10,
    hyperparam_mutations={
        "lr": tune.loguniform(1e-5, 1e-3),
        "train_batch_size": [2000, 4000, 8000]
    }
)

tuner = tune.Tuner(
    "PPO",
    param_space={
        "env": "CartPole-v1",
        "lr": 0.001,
        "train_batch_size": 4000
    },
    tune_config=tune.TuneConfig(
        scheduler=pbt,
        num_samples=8
    )
)

results = tuner.fit()
```

## Evaluation and Inference

### Evaluation

```python
# Evaluate trained algorithm
result = algo.evaluate()
print(f"Evaluation reward: {result['episode_reward_mean']}")
```

### Inference

```python
import gymnasium as gym

# Load environment
env = gym.make("CartPole-v1")

# Inference loop
obs, info = env.reset()
done = False
total_reward = 0

while not done:
    action = algo.compute_single_action(obs)
    obs, reward, terminated, truncated, info = env.step(action)
    done = terminated or truncated
    total_reward += reward

print(f"Total reward: {total_reward}")
```

## Checkpointing and Restoration

### Save and Load

```python
# Save
checkpoint = algo.save()
print(f"Saved at: {checkpoint}")

# Load
from ray.rllib.algorithms.algorithm import Algorithm

algo = Algorithm.from_checkpoint(checkpoint)

# Continue training
for i in range(10):
    result = algo.train()
```

## Best Practices

### Ray Tune

1. **Use appropriate search algorithms**: Optuna/BOHB for small budgets
2. **Enable early stopping**: ASHA for faster convergence
3. **Checkpoint frequently**: Resume failed experiments
4. **Use PBT for dynamic HPO**: Adjust hyperparameters during training
5. **Monitor with TensorBoard**: Track all trials
6. **Specify resources**: Control parallelism
7. **Use grid search sparingly**: Exponential growth with parameters
8. **Report metrics regularly**: Enable schedulers to work

### Ray RLlib

1. **Start with PPO**: Robust general-purpose algorithm
2. **Use multiple workers**: Parallelize rollouts
3. **Tune hyperparameters**: Default configs may not be optimal
4. **Monitor training**: Watch reward trends
5. **Use evaluation**: Separate evaluation from training
6. **Checkpoint regularly**: Save progress
7. **Use appropriate framework**: PyTorch vs TensorFlow
8. **Scale gradually**: Test locally before distributing

## Common Pitfalls

### Ray Tune

- **Too many trials**: Wastes resources
- **Not using schedulers**: No early stopping benefit
- **Ignoring resource specs**: Trials compete for resources
- **Not checkpointing**: Lost progress on failure
- **Wrong metric/mode**: Minimizing when should maximize
- **Blocking in training**: Prevents parallelism
- **Not logging enough**: Hard to debug

### Ray RLlib

- **Wrong algorithm**: DQN for continuous actions
- **Too few workers**: Underutilizes cluster
- **Insufficient training**: Early stopping
- **Not tuning hyperparameters**: Poor performance
- **Ignoring framework**: PyTorch vs TensorFlow differences
- **Not evaluating**: Training metrics misleading
- **Large batch sizes**: Memory issues
