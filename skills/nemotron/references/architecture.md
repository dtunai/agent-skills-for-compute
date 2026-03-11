# Nemotron Architecture Reference

Sources:
- [Inside NVIDIA Nemotron 3](https://developer.nvidia.com/blog/inside-nvidia-nemotron-3-techniques-tools-and-data-that-make-it-efficient-and-accurate)
- [Nemotron 3 Research](https://research.nvidia.com/labs/nemotron/Nemotron-3/)

## Hybrid Mamba-Transformer MoE Architecture

### Core Design

Nemotron 3 combines three architectural components:

```
Layer Pattern:
[Mamba-2, MoE] pairs + selective self-attention

Components:
1. Mamba layers: Efficient sequence modeling
2. Transformer attention: Precision reasoning
3. MoE routing: Scalable compute efficiency
```

### Architectural Benefits

**Mamba Layers:**
- Efficient tracking of long-range dependencies
- Linear complexity for sequence length
- Minimal memory consumption
- Ideal for 1M token context windows

**Transformer Attention:**
- Complex logical relationships
- Code manipulation
- Mathematical reasoning
- Precision tasks

**Mixture of Experts (MoE):**
- Amplifies effective parameters
- Reduces computational overhead
- Only activates relevant experts per token
- 31.6B total params, 3.2B active (Nano)

### Layer Configuration

**Nemotron 3 Nano:**
```
Total Layers: 64
Layer Pattern: Alternating Mamba-2/MoE + selective attention
- Mamba-2 groups: Sequence modeling
- MoE pairs: Conditional computation
- Self-attention (selective): Precision reasoning

Parameters:
- Total: 31.6B
- Active per token: 3.2B
- With embeddings: 3.6B active
```

## Advanced Features (Super/Ultra)

### Latent MoE

**Concept:**
Projects tokens into reduced latent dimensions before expert routing.

**Benefits:**
- 4x more experts at identical inference cost
- Decreased communication overhead
- Better expert specialization
- Reduced memory footprint

**Implementation:**
```
Token (d) → Latent Projection (d/4) → Expert Routing → 
Expert Computation → Output Projection (d)
```

### Multi-Token Prediction (MTP)

**Mechanism:**
Predicts multiple successive tokens simultaneously during training.

**Advantages:**
- ~2.4% accuracy improvement during pretraining
- Enables speculative decoding at inference
- Better long-range dependency learning
- Faster generation with speculation

**Architecture:**
```
Base Model → Multiple Prediction Heads
                ├─ Token t+1
                ├─ Token t+2
                ├─ Token t+3
                └─ Token t+4
```

### NVFP4 Training

**NVIDIA 4-bit Floating Point Format:**
- Custom numerical format for training
- Optimizes cost-accuracy trade-off
- 25 trillion token pretraining
- Reduced memory and compute

**Precision:**
```
Standard: FP16, BF16
NVFP4: 4-bit custom format
Inference: FP8, BF16 (deployment)
```

## Context Window Engineering

### 1M Token Context

**Native Support:**
- No chunking required
- Deep multi-document reasoning
- Long-running agent memory
- Enterprise retrieval without fragmentation

**Memory Optimization:**
- Mamba layers: Linear scaling
- Selective attention: Sparse computation
- Efficient key-value caching
- Gradient checkpointing

**Use Cases:**
- Full document analysis (reports, contracts)
- Multi-document reasoning
- Long conversation history
- Code repository analysis

## Training Methodology

### Multi-Environment Reinforcement Learning (NeMo Gym)

**Trajectory-Based RL:**
```
Environment 1 (Code Execution)
    ├─ Generate action (code)
    ├─ Execute in sandbox
    ├─ Verify result
    └─ Compute reward

Environment 2 (Tool Use)
    ├─ Select tool
    ├─ Generate parameters
    ├─ Execute tool call
    └─ Validate outcome

Environment 3 (Multi-Step Planning)
    ├─ Generate plan steps
    ├─ Execute sequence
    ├─ Check criteria satisfaction
    └─ Reward based on completion
```

**Advantages:**
- Sequential action optimization
- Verification-based rewards
- Reduced reasoning drift
- Multi-step workflow capability

**Open Components:**
- RL datasets
- Training environments
- Evaluation harnesses
- Domain customization tools

### Pretraining Data

**Nemotron-CC Corpus:**
- ~10 trillion tokens
- Curated web data
- Code repositories
- Mathematical content
- Scientific literature
- Multilingual coverage

**Data Quality:**
- Filtered for quality
- Deduplicated
- Domain-balanced
- Safety-filtered
- Openly available

### Synthetic Data Generation

**Nemotron-4 340B Pipeline:**
- 98% synthetic SFT data
- 98% synthetic preference data

**Generation Types:**
1. Open Q&A: Topic → Subtopic → Questions
2. Writing: Prompts across material types
3. Math: School-level problem generation
4. Coding: Python-focused challenges

**Diversity:**
- Multiple domains
- Varying difficulty levels
- Different task formats
- Balanced distribution

## Inference Optimizations

### Throughput Enhancements

**Nemotron 3 Nano:**
- 4x faster than Nemotron 2 Nano
- 3.3x faster than Qwen3-30B-A3B
- 2.2x faster than GPT-OSS-20B

**Optimization Techniques:**
- Sparse activation (MoE)
- Efficient attention (Mamba)
- Kernel fusion
- Operator optimization

### Memory Efficiency

**KV Cache Management:**
- Mamba: No KV cache needed
- Transformer: Selective caching
- MoE: Only active expert weights
- Context parallelism for long sequences

**Quantization:**
- FP8 inference
- INT8 post-training quantization
- Dynamic quantization
- Minimal accuracy loss

## Model Scaling

### Nano (3.2B Active / 31.6B Total)

```
Hardware: 1x A100 80GB or H100
Memory: ~60GB VRAM
Throughput: Highest (4x Nemotron 2 Nano)
Use Case: Cost-efficient agentic tasks
```

### Super (~49B)

```
Hardware: 2x A100 80GB or 1x H100
Memory: ~90GB VRAM
Features: Latent MoE, MTP
Use Case: Multi-agent reasoning, research
```

### Ultra (~253B)

```
Hardware: 4x A100 80GB or 2x H100
Memory: ~500GB VRAM
Features: Latent MoE, MTP, NVFP4
Use Case: Enterprise critical applications
```

## Benchmarks

### Intelligence Index

**Artificial Analysis Intelligence Index v3.0:**
- Nemotron 3 Nano: Score 52
- Leading among similarly-sized models
- Balance of accuracy and throughput

### Domain Benchmarks

**Coding:**
- HumanEval
- MBPP
- Code generation accuracy

**Math:**
- GSM8K
- MATH dataset
- Multi-step reasoning

**Reasoning:**
- MMLU
- ARC Challenge
- Complex problem solving

**Long Context:**
- LongBench
- Multi-document QA
- Context retention

## Architecture Innovations Summary

1. **Hybrid Design**: Mamba + Transformer + MoE
2. **Latent MoE**: 4x experts, same cost (Super/Ultra)
3. **Multi-Token Prediction**: +2.4% accuracy
4. **NVFP4 Training**: Custom 4-bit format
5. **1M Context**: Native long-range support
6. **Multi-Environment RL**: Trajectory optimization
7. **Sparse Activation**: Only 10% params active (Nano)
8. **Open Source**: Weights, data, recipes available

## Design Philosophy

**Efficiency + Accuracy:**
- Maximize throughput without sacrificing quality
- Sparse computation for cost reduction
- Dense computation for critical reasoning

**Flexibility:**
- Adaptive routing (MoE)
- Context-aware attention
- Multi-modal capabilities (VL variant)

**Scalability:**
- From 3B to 253B parameters
- Consistent architecture across sizes
- Efficient distributed training

**Openness:**
- Open weights
- Open training data
- Open techniques
- Reproducible results
