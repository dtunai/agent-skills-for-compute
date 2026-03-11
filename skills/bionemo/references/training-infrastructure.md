---
title: Training Infrastructure
impact: HIGH
tags: training, nemo2, megatron, distributed, pydantic, transformer-engine, recipes, gpu, docker
---

# Skill: Training Infrastructure

Set up and run BioNeMo model training at scale with NeMo 2, Megatron, and TransformerEngine.

## Quick Pattern

```bash
# Download checkpoint and data
ESM2_CKPT=$(download_bionemo_data esm2/650m:2.0 --source ngc)
TEST_DATA=$(download_bionemo_data esm2/testdata_esm2_pretrain:2.0 --source ngc)

# Train ESM-2
train_esm2 --train-cluster-path ${TEST_DATA}/train_clusters.parquet \
  --train-database-path ${TEST_DATA}/train.db \
  --result-dir ./results --num-gpus 1 --num-steps 100
```

## When to Use

- Pre-training or fine-tuning BioNeMo models at scale
- Setting up distributed training across multi-GPU/multi-node
- Configuring training recipes with Pydantic validation
- Accelerating training with TransformerEngine and FP8

## Step-by-Step Instructions

### 1. Hardware Prerequisites

GPU Support Matrix:

| GPU | Compute Capability | Support |
|-----|-------------------|---------|
| H100 | 9.0 | Full |
| L4 | 8.9 | Full |
| L40 | 8.9 | Full |
| A100 | 8.0 | Full |
| A40 | 8.6 | Full |
| RTX 6000 | 8.9 | Full |
| RTX A6000 | 8.6 | Full |

bfloat16 requires Ampere (CC >=8.0). Driver minimum: 560.

Software: Docker with GPU support, NVIDIA Container Toolkit.

### 2. Container Access

```bash
# Pull container
docker pull nvcr.io/nvidia/clara/bionemo-framework:nightly

# Run with GPU
docker run --rm -it --gpus all \
  nvcr.io/nvidia/clara/bionemo-framework:nightly /bin/bash
```

NGC API key required. Setup: NGC > User > Setup > Generate API Key.

Cloud: Supported on AWS, GCP, Azure, OCI via NVIDIA GPU-Optimized VMI.
Also available on brev.dev (takes ~10 min to deploy).

### 3. Data Download Utility

```bash
# CLI
download_bionemo_data esm2/650m:2.0 --source ngc

# Python
from bionemo.core.data.load import load
ckpt_path = load("esm2/650m:2.0")
```

Available resources:

| Resource | Description |
|----------|-------------|
| `esm2/650m:2.0` | ESM-2 650M checkpoint |
| `esm2/3b:2.0` | ESM-2 3B checkpoint |
| `esm2/8m:2.0` | ESM-2 8M checkpoint |
| `esm2/nv_650m:2.1` | NVIDIA pre-trained 650M |
| `esm2/nv_3b:2.1` | NVIDIA pre-trained 3B |
| `geneformer/10M_240530:2.0` | Geneformer 10M |
| `geneformer/106M_240530:2.0` | Geneformer 106M |
| `evo2/1b-8k:1.0` | Evo2 1B (8K context) |
| `evo2/7b-8k:1.0` | Evo2 7B (8K context) |
| `evo2/7b-1m:1.0` | Evo2 7B (1M context) |
| `esm2/testdata_esm2_pretrain:2.0` | ESM-2 test data |
| `single_cell/testdata-20241203` | Single-cell test data |

### 4. Pydantic Configuration System

Two entrypoints per model: argparse CLI + Pydantic config.

**Step 1: Generate recipe config**

```bash
bionemo-esm2-recipe \
  --recipe esm2_650m_recipe \
  --train-cluster-path /data/train_clusters.parquet \
  --train-database-path /data/train.db \
  --valid-cluster-path /data/valid_clusters.parquet \
  --valid-database-path /data/validation.db \
  --result-dir ./results \
  --dest my_config.yaml
```

Recipes available: `esm2_8m_recipe`, `esm2_650m_recipe`, `esm2_3b_recipe`.

**Step 2: Edit config YAML** (adjust devices, precision, W&B, etc.)

**Step 3: Train from config**

```bash
bionemo-esm2-train \
  --data-config-cls bionemo.esm2.run.config_models.ESM2DataConfig \
  --model-config-cls bionemo.esm2.run.config_models.ExposedESM2PretrainConfig \
  --config my_config.yaml
```

Custom DataConfig and ModelConfig classes can be user-defined.

### 5. TransformerEngine Recipes

Training recipes with TransformerEngine acceleration:

| Recipe | Description |
|--------|-------------|
| ESM-2 Accelerate TE | HuggingFace Trainer + TransformerEngine |
| ESM-2 Native TE | Native PyTorch loop + TransformerEngine |
| ESM-2 PEFT TE | LoRA fine-tuning with TransformerEngine |
| Geneformer Native TE MFSDP FP8 | Megatron-FSDP + FP8 |
| ViT | Vision Transformer with Megatron-FSDP + TE |
| CodonFM | Codon frequency model |

### 6. ESM-2 Pre-Training at Scale

| Model | GPUs | GPU Type | Batch/GPU | Steps |
|-------|------|----------|-----------|-------|
| 8M | 32 | A100 | 64 | 500K |
| 650M | 64 | H100 | 32 | 500K |
| 3B | 128 | H100 | 16 | 500K |

LoRA: 2.5-4x memory reduction, 25-80% throughput increase.
MFU: 59.2% on A100 (650M), 60.6% at 256-device (3B).
Device scaling: 96.85% of theoretical linear throughput at 256 A100s.

### 7. NeMo2 and Megatron Backend

BioNeMo 2 is built on NeMo 2 framework with Megatron-LM for:

- Tensor parallelism, pipeline parallelism
- Mixed precision (bf16, fp8)
- Distributed data loading
- Megatron dataset format for efficient training

### 8. AMPLIFY Model

ESM-2 variant with modified layer structure. 120M and 350M sizes.

```python
from bionemo.llm.lightning import biobert_lightning_module
from bionemo.amplify.config import AMPLIFYConfig
import nemo.lightning.io as io

module = biobert_lightning_module(config=AMPLIFYConfig())
io.import_ckpt(module, "hf://chandar-lab/AMPLIFY_120M", "/tmp/nemo_checkpoint")
```

| Model | GPUs | Batch/GPU | Step Time |
|-------|------|-----------|-----------|
| 120M | 16x H100 | 256 | 0.461s |
| 350M | 32x H100 | 128 | 0.525s |

## Common Pitfalls

- **No W&B auto-config**: Must manually edit YAML for Weights & Biases.
- **Container required**: BioNeMo dependencies are complex. Always use Docker.
- **download_bionemo_data caches**: Second call returns cached path instantly.
- **Custom configs**: Both `data-config-cls` and `model-config-cls` support user-defined classes.
- **Argparse vs Pydantic**: Both produce identical results. Pydantic adds YAML validation.

## Related Skills

- [esm2-protein-language.md](./esm2-protein-language.md) - ESM-2 fine-tuning
- [geneformer-single-cell.md](./geneformer-single-cell.md) - Geneformer training
- [evo2-genomic-model.md](./evo2-genomic-model.md) - Evo2 training
- [data-preparation.md](./data-preparation.md) - Dataset preparation
