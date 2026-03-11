---
name: nemotron
description: "NVIDIA Nemotron — open LLM family with hybrid Mamba-Transformer MoE architecture, 1M context, NIM deployment, synthetic data generation"
license: MIT
metadata:
  author: Agent Cluster
  tags: [nemotron, nvidia, llm, moe, mamba, transformer, nim, nemo, synthetic-data, agentic-ai]
---

# NVIDIA Nemotron Skill

Family of open LLMs with hybrid Mamba-Transformer MoE architecture, 1M-token context windows, optimized for agentic AI with NIM deployment and synthetic data generation.

**Official Sources:**
- [NVIDIA Nemotron](https://developer.nvidia.com/nemotron)
- [Nemotron 3 Research](https://research.nvidia.com/labs/nemotron/Nemotron-3/)
- [NeMo Framework Docs](https://docs.nvidia.com/nemo-framework/user-guide/25.09/llms/nemotron.html)
- [NIM for LLMs](https://docs.nvidia.com/nim/large-language-models/latest/)
- [Synthetic Data Generation](https://docs.nvidia.com/nemo-framework/user-guide/24.12/datacuration/syntheticdata.html)

## Model Family

### Reasoning Models

```python
# Nano 30B A3B - Cost-efficient agentic tasks
# - 3.2B active params (31.6B total)
# - 1M token context
# - 4x faster than Nemotron 2 Nano

# Super 49B - Multi-agent reasoning
# - ~49B params
# - High accuracy, efficient deep research

# Ultra 253B - Enterprise workflows
# - ~253B params
# - Maximum accuracy for complex scenarios
```

### Specialized Models

```python
# Vision Language 12B
# - Document intelligence
# - Video understanding

# RAG Models
# - Extraction, embedding, reranking

# Safety Models
# - Jailbreak detection
# - Content safety (multilingual)

# Speech Models
# - ASR, TTS, neural MT
```

## Quick Start with NIM

```bash
# Get NGC API key from https://build.nvidia.com

# Authenticate Docker
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin

# Deploy Nemotron Nano
docker run -it --rm --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0

# Wait for startup
# INFO: Application startup complete
# INFO: Uvicorn running on http://0.0.0.0:8000
```

## API Usage

```python
from openai import OpenAI

# Connect to NIM
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-used"
)

# Chat completion
response = client.chat.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain quantum computing."}
    ],
    temperature=0.7,
    max_tokens=500
)

print(response.choices[0].message.content)

# Streaming
for chunk in client.chat.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    messages=[{"role": "user", "content": "Count to 10"}],
    stream=True
):
    print(chunk.choices[0].delta.content or "", end="")
```

## NeMo Framework Training

```python
from nemo.collections import llm
from nemo import lightning as nl
import nemo_run as run

# Pretraining recipe
pretrain = llm.nemotron3_8b.pretrain_recipe(
    name="nemotron3_8b_pretraining",
    dir="/workspace/checkpoints",
    num_nodes=2,
    num_gpus_per_node=8
)

# Custom data (replace MockDataModule)
pretrain.data = YourDataModule()

# Execute locally
executor = run.LocalExecutor()
run.run(pretrain, executor=executor)

# Or run directly
run.run(pretrain, direct=True)
```

## Synthetic Data Generation

```python
from nemo_curator.synthetic import NemotronGenerator, OpenAIClient

# Connect to API
client = OpenAIClient(
    base_url="https://integrate.api.nvidia.com/v1",
    api_key="your_api_key",
    model_name="nvidia/nemotron-3-nano-30b-a3b"
)

generator = NemotronGenerator(client)

# Generate open Q&A pipeline
qa_data = generator.run_open_qa_pipeline(
    n_macro_topics=5,
    n_subtopics=10,
    n_openlines=3,
    output_file="qa_dataset.jsonl"
)

# Generate math problems
math_data = generator.run_math_pipeline(
    n_macro_topics=3,
    n_subtopics=5,
    school_level="university",
    output_file="math_dataset.jsonl"
)

# Generate Python coding problems
code_data = generator.run_python_pipeline(
    n_macro_topics=4,
    n_subtopics=6,
    output_file="code_dataset.jsonl"
)
```

## Available Model Sizes

```python
# Nemotron 3 (NeMo 2.0)
llm.nemotron3_4b.pretrain_recipe()
llm.nemotron3_8b.pretrain_recipe()

# Nemotron 4 (NeMo 2.0)
llm.nemotron4_15b.pretrain_recipe()      # 16k context
llm.nemotron4_15b_16k.pretrain_recipe()
llm.nemotron4_15b_64k.pretrain_recipe()
llm.nemotron4_22b.pretrain_recipe()      # 16k context
llm.nemotron4_22b_16k.pretrain_recipe()
llm.nemotron4_22b_64k.pretrain_recipe()
llm.nemotron4_340b.pretrain_recipe()
```

## Deployment Options

### NIM Microservices

```bash
# Pre-optimized from API Catalog
docker run -it --rm --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0

# Custom model with multi-LLM NIM
docker run -it --rm --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -e NIM_MODEL_NAME=custom-model \
  -v /path/to/model:/model \
  -p 8000:8000 \
  nvcr.io/nim/meta/llama-3-multi-llm:1.0.0
```

### Open Frameworks

```python
# vLLM
from vllm import LLM, SamplingParams

llm = LLM(model="nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16")
outputs = llm.generate(["Explain AI"], SamplingParams(temperature=0.7))

# Hugging Face Transformers
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained(
    "nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16",
    torch_dtype=torch.bfloat16,
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained("nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16")

# Ollama
# ollama pull nemotron-3-nano
# ollama run nemotron-3-nano
```

## Architecture Highlights

### Hybrid Mamba-Transformer MoE

```
Layer Pattern:
[Mamba-2, MoE] pairs + selective self-attention

Benefits:
- Mamba: Efficient long-sequence modeling (1M tokens)
- Transformer: Precision reasoning (code, math)
- MoE: Scalable compute (31.6B total, 3.2B active)
```

### Advanced Features (Super/Ultra)

```python
# Latent MoE
# - 4x more experts at same cost
# - Reduced communication overhead

# Multi-Token Prediction (MTP)
# - Predicts multiple tokens simultaneously
# - ~2.4% accuracy improvement
# - Speculative decoding speedups

# NVFP4 Training
# - 4-bit floating-point format
# - Cost-accuracy optimization
# - 25 trillion token pretraining
```

## Synthetic Data Pipelines

### Open Q&A

```python
# Generate topics → subtopics → questions → revisions
data = generator.run_open_qa_pipeline(
    n_macro_topics=10,
    n_subtopics=20,
    n_openlines=5,
    output_file="openqa.jsonl"
)
```

### Writing Tasks

```python
# Generate and revise writing prompts
data = generator.run_writing_pipeline(
    n_macro_topics=5,
    n_subtopics=10,
    output_file="writing.jsonl"
)
```

### Math Problems

```python
# School-level math generation
data = generator.run_math_pipeline(
    n_macro_topics=4,
    n_subtopics=8,
    school_level="high school",  # beginner, middle school, high school, university
    output_file="math.jsonl"
)
```

### Coding Problems

```python
# Python-focused problem generation
data = generator.run_python_pipeline(
    n_macro_topics=6,
    n_subtopics=12,
    output_file="python.jsonl"
)
```

## Dialogue Generation

```python
# Multi-turn conversations
dialogue = generator.generate_dialogue(
    persona="expert programmer",
    n_user_turns=3,
    topic="async Python"
)

# Two-turn prompts
two_turn = generator.generate_two_turn_prompt(
    topic="machine learning",
    n_samples=100
)
```

## Async Generation

```python
from nemo_curator.synthetic import AsyncNemotronGenerator

# Concurrent requests with rate limiting
async_gen = AsyncNemotronGenerator(
    client,
    max_concurrent_requests=10
)

data = async_gen.run_open_qa_pipeline(
    n_macro_topics=100,
    n_subtopics=200,
    output_file="large_dataset.jsonl"
)
```

## Integration with NeMo Curator

```python
from nemo_curator import DocumentDataset
from nemo_curator.synthetic import NemotronGenerator

# Generate synthetic data
generator = NemotronGenerator(client)
df = generator.run_open_qa_pipeline(n_macro_topics=5)

# Convert to DocumentDataset
dataset = DocumentDataset.from_pandas(df)

# Apply filtering
from nemo_curator.filters import WordCountFilter
filtered = dataset.filter(WordCountFilter(min_words=10))

# Deduplicate
from nemo_curator.modules import ExactDuplicates
deduped = ExactDuplicates().deduplicate(filtered)

# Export
deduped.to_pandas().to_json("curated_data.jsonl", orient="records", lines=True)
```

## Hardware Requirements

```
Nano 30B A3B:
- GPU: 1x A100 80GB or H100
- RAM: 64GB+
- VRAM: ~60GB

Super 49B:
- GPU: 2x A100 80GB or 1x H100
- RAM: 128GB+

Ultra 253B:
- GPU: 4x A100 80GB or 2x H100
- RAM: 256GB+
```

## Best Practices

1. **NIM Deployment**: Use pre-built containers for production
2. **Synthetic Data**: Start with small batches, scale up
3. **Training**: Replace MockDataModule with real data
4. **Context Window**: Leverage 1M tokens for long documents
5. **API Compatibility**: OpenAI-compatible endpoints
6. **Async Generation**: Use for large-scale data creation
7. **Model Selection**: Nano for cost, Ultra for accuracy

## Common Patterns

```python
# Deploy + Generate + Train pipeline
# 1. Deploy NIM for inference
# 2. Generate synthetic data
# 3. Train custom model with NeMo

# Agentic AI workflow
# - Use Nemotron for reasoning
# - 1M context for long-running memory
# - Tool calling via function API

# Document intelligence
# - VL 12B for multimodal understanding
# - RAG models for extraction/embedding
# - 1M context for full documents
```

## References

- **[Model Family](references/model-family.md)** - Nano, Super, Ultra, VL, RAG, Safety, Speech models
- **[NIM Deployment](references/nim-deployment.md)** - Containers, API, inference, scaling
- **[Synthetic Data](references/synthetic-data.md)** - NemotronGenerator, pipelines, customization
- **[NeMo Training](references/nemo-training.md)** - Pretraining recipes, configurations, distributed training
- **[Architecture](references/architecture.md)** - Hybrid Mamba-Transformer MoE, training techniques
- **[API Reference](references/api-reference.md)** - Endpoints, parameters, examples
