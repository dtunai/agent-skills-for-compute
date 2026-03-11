---
name: prime-verifiers
description: "Prime Intellect verifiers & PrimeRL — build RL environments, rubrics, and reward functions for LLM post-training. Covers environment classes, tool integration, GEPA prompt optimization, and async distributed RL training."
license: MIT
metadata:
  author: Agent Cluster
  tags: [prime-intellect, verifiers, rl, reinforcement-learning, llm, post-training, environments, rewards, grpo]
---

# Prime Intellect Verifiers & PrimeRL

Build RL environments, rubrics, and reward functions for LLM post-training using Prime Intellect's verifiers library and PrimeRL distributed training framework.

**Official Sources:**
- [Verifiers Documentation](https://docs.primeintellect.ai/verifiers/source)
- [Verifiers GitHub](https://github.com/PrimeIntellect-ai/verifiers)
- [PrimeRL GitHub](https://github.com/PrimeIntellect-ai/prime-rl)
- [Prime CLI/SDK](https://github.com/PrimeIntellect-ai/prime)
- [Environments Hub](https://docs.primeintellect.ai/environments)

## Architecture Overview

Three integrated components form the stack:

| Component | Purpose | Key API |
|-----------|---------|---------|
| **verifiers** | Environment definitions, rubrics, evaluation | `vf.Environment`, `vf.Rubric`, `vf.ToolEnv` |
| **prime-rl** | Async distributed RL training (1000+ GPUs) | Orchestrator, Trainer, Inference |
| **prime CLI** | Workspace setup, sandboxes, publishing | `prime lab setup`, `prime env init` |

## Quick Start

```bash
# Install verifiers
pip install verifiers

# Or with all extras
pip install "verifiers[all]"

# Set up a lab workspace
prime lab setup

# Create a new environment
prime env init my-env
cd environments/my-env
```

## Core Concepts

### Environments

Everything is an **Environment**. An environment encapsulates: dataset + interaction harness + reward rubric.

**Environment class hierarchy:**

| Class | Use Case |
|-------|----------|
| `SingleTurnEnv` | One prompt, one response, score it |
| `MultiTurnEnv` | Multi-step agent interactions |
| `ToolEnv` | OpenAI-compatible tool calling |
| `MCPEnv` | Tools via MCP servers |
| `StatefulToolEnv` | Per-rollout state (hidden args, dynamic tools) |
| `SandboxEnv` | Containerized bash shell execution |
| `PythonEnv` | Persistent Python REPL in sandbox |
| `EnvGroup` | Combine multiple environments for multi-task training |

### Environment Contract

Every environment must expose a `load_environment()` function:

```python
import verifiers as vf

def load_environment(
    split: str = "train",
    seed: int | None = None,
    system_prompt: str | None = None,
    **kwargs,
) -> vf.Environment:
    dataset = ...  # HF Dataset with prompt/answer/info columns
    rubric = vf.Rubric(
        funcs=[reward_fn],
        weights=[1.0],
        parser=vf.XMLParser(fields=["answer"]),
    )
    return vf.SingleTurnEnv(
        dataset=dataset,
        rubric=rubric,
        system_prompt=system_prompt or "You are a helpful assistant.",
    )
```

### Rubrics and Reward Functions

Rubrics define how completions are scored. Reward functions use **argument injection by name**:

```python
def exact_match(completion: str, answer: str, parser: vf.XMLParser) -> float:
    """Score 1.0 if parsed answer matches expected."""
    parsed = parser(completion)
    return 1.0 if parsed.get("answer", "").strip() == answer.strip() else 0.0

def length_penalty(completion: str) -> float:
    """Penalize overly long responses."""
    return max(0.0, 1.0 - len(completion) / 5000)

rubric = vf.Rubric(
    funcs=[exact_match, length_penalty],
    weights=[0.8, 0.2],
    parser=vf.XMLParser(fields=["answer"]),
)
```

**Available injectable arguments:**
- `completion` — model output text
- `answer` — ground truth from dataset
- `info` — metadata dict from dataset
- `parser` — the rubric's parser instance
- `judge` — LLM judge client (for `JudgeRubric`)
- `state` — rollout state dict (for stateful envs)
- `messages` — full conversation history

**Built-in rubric types:**
- `Rubric` — weighted combination of reward functions
- `JudgeRubric` — LLM-as-judge scoring
- `MathRubric` — symbolic math verification via `math-verify`
- `RubricGroup` — combines multiple rubrics in parallel

### Parsers

Extract structured fields from model output:

```python
# XML-style parsing: <answer>42</answer>
parser = vf.XMLParser(fields=["answer", "reasoning"])

# Think/answer parsing: <think>...</think>\n<answer>...</answer>
parser = vf.ThinkParser(fields=["answer"])

# Optional thinking: works with or without <think> tags
parser = vf.MaybeThinkParser(fields=["answer"])
```

### Tool Environments

```python
import verifiers as vf

async def calculator(expression: str) -> str:
    """Evaluate a mathematical expression."""
    try:
        result = eval(expression)
        return str(result)
    except Exception as e:
        return f"Error: {e}"

def load_environment(**kwargs) -> vf.Environment:
    dataset = ...
    rubric = vf.Rubric(funcs=[reward_fn], parser=parser)
    return vf.ToolEnv(
        dataset=dataset,
        tools=[calculator],
        rubric=rubric,
        max_turns=10,
    )
```

### Stateful Tool Environments

For tools that need per-rollout hidden state:

```python
class MyEnv(vf.StatefulToolEnv):
    def setup_state(self, prompt, answer, info):
        """Initialize per-rollout state."""
        return {"secret": info["hidden_value"], "attempts": 0}

    def update_tool_args(self, state):
        """Inject state into tool calls as hidden args."""
        return {"check_answer": {"_secret": state["secret"]}}
```

### Rollout Lifecycle

For custom multi-turn environments:

```python
class MyMultiTurnEnv(vf.MultiTurnEnv):
    def env_response(self, messages, state):
        """Generate environment response after each model turn."""
        last = messages[-1]["content"]
        if "DONE" in last:
            raise vf.stop()  # End the rollout
        return f"Feedback: {self.evaluate(last)}"

    @vf.cleanup
    def cleanup(self, messages, state):
        """Run after rollout ends (success or error)."""
        return messages  # Can modify final messages

    @vf.teardown
    def teardown(self, messages, state):
        """Run after everything, for resource cleanup."""
        pass
```

## CLI Workflow

```bash
# Set up workspace (downloads AGENTS.md, skills, templates)
prime lab setup

# Create new environment from template
prime env init math-solver

# Install environment locally
prime env install math-solver

# Run evaluation
prime eval run math-solver --endpoint openai://gpt-4o --num-samples 100

# Optimize system prompt with GEPA
prime gepa run math-solver --rounds 5

# Publish to Environments Hub
prime env push
```

## Evaluation

```bash
# Local eval with any OpenAI-compatible endpoint
prime eval run my-env \
    --endpoint openai://gpt-4o \
    --num-samples 200 \
    --split test

# With custom system prompt
prime eval run my-env \
    --endpoint vllm://localhost:8000 \
    --system-prompt "Think step by step."
```

## GEPA (Genetic-Pareto Prompt Optimization)

Iteratively optimize system prompts without gradient training:

```bash
prime gepa run my-env \
    --endpoint openai://gpt-4o \
    --rounds 10 \
    --population 8

# Uses a teacher LLM to reflect on eval results
# and propose improved system prompts
```

## PrimeRL Training

Three disaggregated components for distributed RL:

```
Orchestrator (CPU) ──→ Trainer (GPU, FSDP2) ──→ Inference (vLLM)
     │                       │                        │
     ├─ Schedules rollouts   ├─ GRPO/PPO/RLOO        ├─ OpenAI-compat API
     ├─ Assembles batches    ├─ Custom model impls    ├─ Hot weight reload
     └─ Hosts env servers    └─ LoRA support          └─ update_weights()
```

**Supported RL objectives:** GRPO, GSPO, OPO, RLOO, CISPO, PPO

**Training config pattern:**
```yaml
model: Qwen/Qwen3-8B
objective: grpo
group_size: 8
max_turns: 10
environment: my-env
```

## Environment Publishing

Package structure for publishable environments:

```
my-env/
├── my_env/
│   ├── __init__.py
│   └── env.py          # load_environment() here
├── pyproject.toml       # [tool.verifiers.eval] config
└── README.md
```

**pyproject.toml:**
```toml
[project]
name = "my-env"
dependencies = ["verifiers>=0.1.10"]

[tool.verifiers.eval]
default_num_samples = 100
default_endpoint = "openai://gpt-4o"

[tool.hatch.build]
packages = ["my_env"]
```

## Best Practices

1. **Always use `prime env init`** to scaffold environments — never create manually
2. **Validate with `prime eval run`** before publishing
3. **Use `vf.ensure_keys()`** to validate required API keys at environment load time
4. **Choose the right env class**: `ToolEnv` for stateless tools, `StatefulToolEnv` for per-rollout state, `SandboxEnv` for code execution
5. **Keep reward functions pure** — no side effects, deterministic when possible
6. **Use group reward functions** for GRPO-style advantage centering (accept `completions` plural, return lists)
7. **Test locally first**: `prime eval run my-env --endpoint openai://gpt-4o --num-samples 10`
