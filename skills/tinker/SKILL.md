---
name: tinker
description: "Tinker by Thinking Machines Lab — low-level training API for LLM fine-tuning. LoRA training, RL environments, supervised learning, DPO, and custom loss functions with full algorithmic control over remote GPU clusters."
license: MIT
metadata:
  author: Agent Cluster
  tags: [tinker, thinking-machines, llm, fine-tuning, lora, rl, training, dpo, grpo]
---

# Tinker (Thinking Machines Lab)

Low-level training API that abstracts distributed LLM fine-tuning without hiding the knobs. Write a simple Python script on a CPU machine; Tinker executes the GPU-heavy computation on remote clusters.

**Official Sources:**
- [Tinker Documentation](https://tinker-docs.thinkingmachines.ai/)
- [Tinker SDK GitHub](https://github.com/thinking-machines-lab/tinker)
- [Tinker Cookbook GitHub](https://github.com/thinking-machines-lab/tinker-cookbook)
- [Blog: Announcing Tinker](https://thinkingmachines.ai/blog/announcing-tinker/)

## Philosophy

| You control | Tinker handles |
|-------------|----------------|
| Data, RL environments, loss functions | Distributed GPU training at scale |
| Training loop logic, hyperparameters | Hardware failures, scheduling, reliability |
| Evaluations, custom algorithms | Efficient execution up to 235B-param models |

**Not a black box** — full algorithmic control. Changing models requires changing a single string.

## Quick Start

```bash
# 1. Sign up: https://auth.thinkingmachines.ai/sign-up
# 2. Get API key: https://tinker-console.thinkingmachines.ai
export TINKER_API_KEY=<your-key>

# 3. Install
pip install tinker

# 4. Install cookbook (for recipes and helpers)
git clone https://github.com/thinking-machines-lab/tinker-cookbook
cd tinker-cookbook && pip install -e .
```

## Core API (4 Primitives)

```python
import tinker

# Create a training client
service_client = tinker.ServiceClient()
training_client = service_client.create_lora_training_client(
    base_model="Qwen/Qwen3-8B",
    rank=32,
)

# 1. FORWARD_BACKWARD — compute gradients
training_client.forward_backward(data, loss_fn="cross_entropy")

# 2. OPTIM_STEP — update weights
training_client.optim_step(tinker.AdamParams(learning_rate=1e-4))

# 3. SAVE + SAMPLE — checkpoint and generate
sampling_client = training_client.save_weights_and_get_sampling_client("step-100")
responses = sampling_client.sample(
    prompt=[tinker.TextChunk(text="Hello")],
    num_samples=4,
    sampling_params=tinker.SamplingParams(temperature=0.7, max_tokens=256),
)

# 4. SAVE_STATE — full checkpoint (weights + optimizer)
training_client.save_state("checkpoint-100")
```

## Data Types

```python
import tinker
import numpy as np

# Datum = model_input + loss_fn_inputs
datum = tinker.Datum(
    model_input=tinker.ModelInput(
        chunks=[tinker.TextChunk(text="What is 2+2?")],
    ),
    loss_fn_inputs={
        "target_tokens": tinker.TensorData.from_numpy(
            np.array([42, 43, 44]),
            name="target_tokens",
        ),
    },
)

# Batch is just a list of Datums
batch = [datum1, datum2, datum3]
training_client.forward_backward(batch, loss_fn="cross_entropy")
```

## Loss Functions

| Loss Function | Use Case |
|---------------|----------|
| `cross_entropy` | Supervised fine-tuning |
| `importance_sampling` | REINFORCE with IS correction |
| `ppo` | Proximal Policy Optimization |
| `cispo` | Clipped Importance Sampling Policy Optimization |
| `dro` | Direct Reward Optimization |
| `forward_backward_custom` | Arbitrary differentiable loss |

## Supervised Learning

```python
from tinker_cookbook.supervised.train import train as sl_train
from tinker_cookbook.renderers import llama3

# Minimal SFT
config = SLConfig(
    base_model="meta-llama/Llama-3.1-8B-Instruct",
    renderer=llama3.Renderer(),
    dataset_builder=my_dataset_builder,
    learning_rate=1e-4,
    rank=32,
    num_epochs=3,
)
sl_train(config)
```

## RL Training

### Environment Interface

```python
from tinker_cookbook.rl.types import Env, StepResult, EnvGroupBuilder

class MathEnv(Env):
    def __init__(self, problem: str, answer: str):
        self.problem = problem
        self.answer = answer

    def initial_observation(self) -> list[dict]:
        """Return the initial messages for this rollout."""
        return [
            {"role": "system", "content": "Solve the math problem."},
            {"role": "user", "content": self.problem},
        ]

    def step(self, action: str) -> StepResult:
        """Score the model's response. Called once per turn."""
        correct = self.answer.strip() in action
        return StepResult(
            reward=1.0 if correct else 0.0,
            done=True,
            next_observation=None,
        )
```

### Environment Groups (for GRPO)

```python
class MathEnvGroup(EnvGroupBuilder):
    def __init__(self, problem: str, answer: str, group_size: int = 8):
        self.problem = problem
        self.answer = answer
        self.group_size = group_size

    def build(self) -> list[Env]:
        """Create a group of identical envs for advantage estimation."""
        return [MathEnv(self.problem, self.answer) for _ in range(self.group_size)]

    def compute_group_rewards(self, trajectories: list) -> list[float]:
        """Optional: override for group-level reward shaping."""
        rewards = [t.reward for t in trajectories]
        mean = sum(rewards) / len(rewards)
        return [r - mean for r in rewards]  # baseline subtraction
```

### RL Training Loop

```python
from tinker_cookbook.rl.train import train as rl_train

config = RLConfig(
    base_model="Qwen/Qwen3-8B",
    renderer=qwen3.Renderer(),
    dataset_builder=my_rl_dataset_builder,
    learning_rate=3e-5,
    rank=32,
    group_size=8,
    max_turns=1,
    kl_coeff=0.01,
)
rl_train(config)
```

## DPO Training

```python
from tinker_cookbook.preference.train_dpo import train as dpo_train

config = DPOConfig(
    base_model="meta-llama/Llama-3.1-8B-Instruct",
    renderer=llama3.Renderer(),
    dataset_builder=preference_dataset_builder,
    beta=0.1,
    learning_rate=5e-5,
)
dpo_train(config)
```

## Renderers

Bridge between chat messages and token sequences. Must match the model:

| Renderer | Models |
|----------|--------|
| `llama3` | Llama 3.x family |
| `qwen3` | Qwen 3.x family |
| `deepseek_v3` | DeepSeek V3 |
| `kimi_k2` | Kimi K2 |
| `role_colon` | Generic fallback |

```python
from tinker_cookbook.renderers import qwen3
renderer = qwen3.Renderer()
```

## Async Pattern (Performance)

```python
# Submit both operations before awaiting — pipeline the GPU work
fb_future = training_client.forward_backward_async(batch, loss_fn="cross_entropy")
optim_future = training_client.optim_step_async(tinker.AdamParams(lr=1e-4))

# Now await
fb_result = await fb_future
optim_result = await optim_future
```

## Recipes (Worked Examples)

| Recipe | Description |
|--------|-------------|
| `sl_basic` / `sl_loop` | Minimal supervised fine-tuning |
| `rl_basic` / `rl_loop` | Minimal RL training |
| `math_rl` | Math reasoning via RL (GSM8K) |
| `code_rl` | Code generation with sandbox execution |
| `search_tool` | Train LLMs to use retrieval tools |
| `preference/dpo` | DPO training |
| `preference/rlhf` | Full 3-stage RLHF pipeline |
| `multiplayer_rl` | Multi-agent RL (twenty questions, text arena) |
| `distillation` | On/off-policy distillation |
| `prompt_distillation` | Internalize long system prompts |
| `harbor_rl` | Terminal/tool-use RL |
| `rubric` | Rubric-based evaluation training |
| `vlm_classifier` | Vision-language model classification |

## Evaluation

```python
from tinker_cookbook.eval import Evaluator

evaluator = Evaluator(
    sampling_client=sampling_client,
    renderer=renderer,
    eval_tasks=["gsm8k", "mmlu"],
)
results = evaluator.run(num_samples=100)
```

## Conventions and Pitfalls

1. **Tensor naming**: subscript suffixes `_P` (problems), `_G` (groups), `_T` (tokens), `_D` (datums)
2. **LoRA learning rate**: use `hyperparam_utils.get_lr(model_name)` — LoRA needs ~10x higher LR than full fine-tuning
3. **Renderer must match model**: wrong renderer = garbage tokenization
4. **Env objects are single-use**: no `reset()` method — create new instances per rollout
5. **Always create new sampling client after `save_weights`**: old clients point to stale weights
6. **Use `chz.Blueprint`** for CLI-constructable configs that serialize for sweeps
7. **Supported models**: up to 235B parameters — check docs for current model list

## CLI

```bash
# List active runs
tinker run list

# Get checkpoint info
tinker checkpoint info <checkpoint-id>

# Resume from checkpoint
training_client = service_client.resume_training_client(checkpoint_id="...")
```
