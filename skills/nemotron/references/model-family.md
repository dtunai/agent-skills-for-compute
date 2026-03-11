# NVIDIA Nemotron Model Family Reference

Sources:
- [NVIDIA Nemotron](https://developer.nvidia.com/nemotron)
- [Nemotron 3 Research](https://research.nvidia.com/labs/nemotron/Nemotron-3/)

## Overview

NVIDIA Nemotron is "a family of open models with open weights, training data, and recipes" designed for building specialized AI agents with reasoning capabilities.

## Nemotron 3 Family

### Reasoning Models

**Nano 30B A3B**
- **Parameters**: 3.2B active (3.6B with embeddings), 31.6B total
- **Architecture**: Hybrid Mamba-Transformer MoE
- **Context Window**: Up to 1M tokens
- **Performance**: 4x faster throughput than Nemotron 2 Nano
- **Use Case**: Cost-efficient targeted agentic tasks
- **Release**: December 15, 2025
- **Availability**: Open source (Hugging Face, Ollama, build.nvidia.com)

**Super 49B**
- **Parameters**: ~49B total
- **Architecture**: Hybrid Mamba-Transformer MoE with Latent MoE
- **Features**: Multi-Token Prediction, NVFP4 training
- **Use Case**: High accuracy multi-agent reasoning, efficient deep research
- **Availability**: Expected H1 2026

**Ultra 253B**
- **Parameters**: ~253B total
- **Architecture**: Hybrid Mamba-Transformer MoE with Latent MoE
- **Features**: Multi-Token Prediction, NVFP4 training
- **Use Case**: Enterprise workflows requiring maximum accuracy
- **Availability**: Expected H1 2026

### Performance Comparison

```
Nemotron 3 Nano:
- 3.3x higher throughput than Qwen3-30B-A3B
- 2.2x higher throughput than GPT-OSS-20B
- Score 52 on Artificial Analysis Intelligence Index v3.0
- Better accuracy than Nemotron 2 Nano with <50% active params
```

## Specialized Models

### Vision Language (VL) 12B

**Capabilities:**
- Document intelligence
- Video understanding
- Multi-modal reasoning
- Best-in-class accuracy for visual tasks

**Use Cases:**
- PDF/document analysis
- Video content understanding
- Visual Q&A
- Multi-modal agentic applications

### RAG Models

**Components:**
- **Extraction**: Extract structured data from documents
- **Embedding**: Dense vector representations for retrieval
- **Reranking**: Improve retrieval precision

**Features:**
- Industry-leading accuracy
- Optimized for document intelligence
- Seamless integration with RAG pipelines

**Use Cases:**
- Enterprise search
- Document Q&A
- Knowledge base retrieval
- Semantic search

### Safety Models

**Llama 3.1 Nemotron Safety Guard 8B**

**Capabilities:**
- Advanced jailbreak detection
- Multilingual content safety
- Real-time moderation
- Policy compliance

**Features:**
- Low latency
- High accuracy
- Customizable safety policies
- Integration with NIM

**Use Cases:**
- Content moderation
- User input filtering
- Output validation
- Compliance enforcement

### Speech Models

**Capabilities:**
- Automatic Speech Recognition (ASR)
- Text-to-Speech (TTS)
- Neural Machine Translation (NMT)

**Features:**
- High throughput
- Low latency
- Multilingual support
- Optimized for agentic applications

**Use Cases:**
- Voice assistants
- Real-time translation
- Speech-to-text
- Conversational AI

## Model Availability

### Open Source Releases

**Nemotron 3 Nano:**
- Weights: FP8 and BF16 variants
- Training Data: Nemotron-CC pretraining datasets (~10T tokens)
- Code: NVIDIA Nemotron Developer Repository (GitHub)
- License: NVIDIA Open Model License

**Access Points:**
- Hugging Face: `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-BF16`
- Ollama: `ollama pull nemotron-3-nano`
- NVIDIA Build: https://build.nvidia.com
- NGC Catalog: https://catalog.ngc.nvidia.com

### Deployment Options

**NVIDIA NIM Microservices:**
- Pre-optimized containers
- GPU-accelerated inference
- OpenAI-compatible API
- Scalable deployment

**Inference Providers:**
- Baseten
- DeepInfra
- Fireworks AI
- Together AI
- OpenRouter

**Open Frameworks:**
- vLLM
- SGLang
- Ollama
- llama.cpp
- Hugging Face Transformers
- TensorRT-LLM

## Model Selection Guide

### By Use Case

**Cost-Efficient Agentic Tasks:**
- Nemotron 3 Nano 30B A3B
- Best throughput-to-cost ratio
- 1M token context
- Suitable for: chatbots, simple reasoning, tool calling

**Multi-Agent Reasoning:**
- Nemotron 3 Super 49B
- High accuracy, efficient deep research
- Collaborative agent workflows
- Suitable for: research assistants, complex problem solving

**Enterprise Critical Applications:**
- Nemotron 3 Ultra 253B
- Maximum accuracy
- Complex scenario handling
- Suitable for: financial analysis, legal reasoning, mission-critical

**Visual Understanding:**
- Nemotron VL 12B
- Document + video intelligence
- Suitable for: document analysis, visual Q&A, multimodal agents

**Retrieval Augmented Generation:**
- Nemotron RAG models
- Extraction, embedding, reranking
- Suitable for: enterprise search, knowledge bases

**Content Safety:**
- Llama 3.1 Nemotron Safety Guard 8B
- Jailbreak detection, content moderation
- Suitable for: user-facing applications, compliance

### By Hardware

**1x A100 80GB or H100:**
- Nemotron 3 Nano 30B A3B
- Nemotron VL 12B
- Safety Guard 8B

**2x A100 80GB or 1x H100:**
- Nemotron 3 Super 49B

**4x A100 80GB or 2x H100:**
- Nemotron 3 Ultra 253B

### By Throughput Requirements

**Highest Throughput:**
- Nemotron 3 Nano (4x faster than Nemotron 2 Nano)
- Best for high-volume workloads

**Balanced Throughput/Accuracy:**
- Nemotron 3 Super
- Efficient for multi-agent systems

**Maximum Accuracy:**
- Nemotron 3 Ultra
- Optimized for correctness over speed

## Training Data

### Nemotron-CC Pretraining Datasets

**Size:** ~10 trillion tokens

**Composition:**
- Curated web data
- Code repositories
- Mathematical content
- Scientific literature
- Multilingual text

**Quality:**
- Filtered and deduplicated
- Domain-balanced
- Safety-filtered
- Open source availability

### Synthetic Data Generation

**Nemotron-4 340B Pipeline:**
- 98% synthetic data for SFT
- UltraChat-based generation
- Open Q&A, writing, math, coding
- Preference data generation

## Licensing

**NVIDIA Open Model License:**
- Open weights
- Open training data
- Open recipes
- Commercial use permitted
- Research use permitted

**Access:**
- No API key required for downloads
- NGC account for NIM deployment
- Community support via GitHub

## Benchmarks

### Nemotron 3 Nano

**Artificial Analysis Intelligence Index v3.0:**
- Score: 52 (leading among similar-sized models)

**Efficiency:**
- 4x throughput vs Nemotron 2 Nano
- <50% active parameters
- Better accuracy than predecessor

**Long Context:**
- Native 1M token support
- Superior multi-document reasoning
- Reduced fragmentation in RAG

### Comparison to Alternatives

**vs Qwen3-30B-A3B:**
- 3.3x higher inference throughput

**vs GPT-OSS-20B:**
- 2.2x higher inference throughput

**vs Nemotron 2 Nano:**
- 4x faster throughput
- Better accuracy with fewer active params

## Developer Resources

**GitHub Repository:**
- Training recipes
- Usage cookbooks
- End-to-end examples
- Dataset preparation scripts

**Documentation:**
- NeMo Framework User Guide
- NIM Deployment Guides
- Synthetic Data Generation Tutorials

**Community:**
- NVIDIA Developer Forums
- GitHub Issues
- Technical Blog Posts
- Research Papers
