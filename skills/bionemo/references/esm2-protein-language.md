---
title: ESM-2 Protein Language Model
impact: CRITICAL
tags: esm2, protein, embedding, fine-tuning, inference, language-model, sequence
---

# Skill: ESM-2 Protein Language Model

Train, fine-tune, and run inference with ESM-2 protein language models in BioNeMo.

## Quick Pattern

```python
from bionemo.esm2.model.finetune.finetune_regressor import (
    ESM2FineTuneSeqConfig, ESM2FineTuneSeqModel
)
from bionemo.core.data.load import load

# Load pretrained ESM-2 650M
pretrain_ckpt = load("esm2/650m:2.0")
config = ESM2FineTuneSeqConfig(initial_ckpt_path=str(pretrain_ckpt))
```

## When to Use

- Generating protein sequence embeddings for downstream tasks
- Fine-tuning ESM-2 for property prediction (stability, fitness, binding)
- Predicting protein function from sequence
- Zero-shot protein design and mutation analysis
- Clustering proteins by learned representations

## Step-by-Step Instructions

### 1. Download Pretrained Checkpoint

```python
from bionemo.core.data.load import load

# Available models
ckpt_650m = load("esm2/650m:2.0")   # 650M parameters
ckpt_3b   = load("esm2/3b:2.0")     # 3B parameters (larger, more accurate)
```

### 2. Fine-Tune for Sequence Regression

Predict a continuous property (e.g., stability score) from protein sequence:

```python
from pathlib import Path
from bionemo.esm2.model.finetune.finetune_regressor import (
    ESM2FineTuneSeqConfig,
    InMemorySingleValueDataset,
)
from bionemo.esm2.model.finetune.datamodule import ESM2FineTuneDataModule
from bionemo.core.data.load import load
from bionemo.llm.train import train_model

# Prepare data: list of (sequence, value) tuples
data = [
    ("TLILGWSDKLGSLLNQLAIANESLGGGTIAVMAERDKEDMELDIGKMEFDFKGTSVI", 0.57),
    ("LYSGDHSTQGARFLRDLAENTGRAEYELLSLF", 0.32),
    ("GRFNVWLGGNESKIRQVLKAVKEIGVSPTLFAVYEKN", 0.38),
    ("DELTALGGLLHDIGKPVQRAGLYSGDHSTQGARFLRDLAENTGRAEYELLSLF", 0.55),
    ("KLGSLLNQLAIANESLGGGTIAVMAERDKEDMELDIGKMEFDFKGTSVI", 0.49),
    ("LFGAIGNAISAIHGQSAVEELVDAFVGGARISSAFPYSGDTYYLPKP", 0.46),
    ("LGGLLHDIGKPVQRAGLYSGDHSTQGARFLRDLAENTGRAEYELLSLF", 0.32),
    ("ISAIHGQSAVEELVDAFVGGARISSAFPYSGDTYYLPKP", 0.39),
]

# Create dataset and data module
dataset = InMemorySingleValueDataset(data)
data_module = ESM2FineTuneDataModule(
    train_dataset=dataset,
    valid_dataset=dataset,
    micro_batch_size=4,
    global_batch_size=8,
)

# Configure fine-tuning
pretrain_ckpt = load("esm2/650m:2.0")
config = ESM2FineTuneSeqConfig(
    initial_ckpt_path=str(pretrain_ckpt),
    encoder_frozen=True,      # Freeze ESM-2 encoder
    ft_dropout=0.25,          # Dropout for regression head
)

# Train
checkpoint, metrics, trainer = train_model(
    experiment_name="stability_prediction",
    experiment_dir=Path("./results"),
    config=config,
    data_module=data_module,
    n_steps_train=50,
)
```

### 3. Custom Task Head

Define a custom MLP head for specific prediction tasks:

```python
import torch
from megatron.core.transformer.module import MegatronModule
from megatron.core.transformer.transformer_config import TransformerConfig

class MegatronMLPHead(MegatronModule):
    def __init__(self, config: TransformerConfig):
        super().__init__(config)
        layer_sizes = [config.hidden_size, 256, 1]
        self.linear_layers = torch.nn.ModuleList(
            [torch.nn.Linear(i, o) for i, o in zip(layer_sizes[:-1], layer_sizes[1:])]
        )
        self.act = torch.nn.ReLU()
        self.dropout = torch.nn.Dropout(p=config.ft_dropout)

    def forward(self, hidden_states: torch.Tensor):
        # Take CLS token embedding
        x = hidden_states[:, 0, :]  # [B, hidden_size]
        for i, layer in enumerate(self.linear_layers[:-1]):
            x = self.dropout(self.act(layer(x)))
        return self.linear_layers[-1](x)  # [B, 1]
```

### 4. Custom Loss Function

```python
import torch
from typing import Dict, Tuple, Sequence, Union
from bionemo.llm.model.loss import BERTMLMLossWithReduction

class RegressorLossReduction(BERTMLMLossWithReduction):
    def forward(
        self,
        batch: Dict[str, torch.Tensor],
        forward_out: Dict[str, torch.Tensor],
    ) -> Tuple[torch.Tensor, Dict]:
        targets = batch["labels"]
        regression_output = forward_out
        loss = torch.nn.functional.mse_loss(regression_output, targets)
        return loss, {"avg": loss}

    def reduce(self, losses_reduced_per_micro_batch):
        losses = torch.stack([l["avg"] for l in losses_reduced_per_micro_batch])
        return losses.mean()
```

### 5. Run Inference

```bash
# CLI inference
infer_esm2 --checkpoint-path /path/to/checkpoint \
  --data-path sequences.csv \
  --results-path ./results/ \
  --config-class ESM2FineTuneSeqConfig

# Include hidden states for embeddings
infer_esm2 --checkpoint-path /path/to/checkpoint \
  --data-path sequences.csv \
  --results-path ./results/ \
  --include-hiddens
```

```python
# Load results
import torch
results = torch.load("./results/predictions.pt")
# results['regression_output']: [N, 1]
# results['hidden_states']: [N, seq_len, hidden_dim] (if --include-hiddens)
```

### 6. LoRA Fine-Tuning (Parameter-Efficient)

```bash
infer_esm2 --checkpoint-path /path/to/lora_checkpoint \
  --data-path sequences.csv \
  --results-path ./results/ \
  --config-class ESM2FineTuneSeqConfig
```

LoRA performance vs full fine-tuning:

| Metric | Full Fine-Tune | LoRA |
|--------|---------------|------|
| GPU memory | Baseline | 2.5-4x reduction |
| Throughput (tokens/s) | Baseline | 25-80% increase |

### 7. DDP Inference (Multi-GPU)

```bash
torchrun --nproc_per_node=4 \
  -m bionemo.esm2.scripts.infer_esm2 \
  --checkpoint-path /path/to/checkpoint \
  --data-path large_dataset.csv \
  --results-path ./results/
```

### 8. Pre-Training from Scratch

NVIDIA pre-trained checkpoints (trained on UniProt 2024_03):

```python
ckpt_nv_650m = load("esm2/nv_650m:2.1")  # NVIDIA-trained 650M
ckpt_nv_3b   = load("esm2/nv_3b:2.1")    # NVIDIA-trained 3B
```

Validation perplexity (NVIDIA validation set, 500K steps):

| Model | Perplexity |
|-------|-----------|
| 8M | 10.26 |
| 650M | 7.14 |
| 3B | 6.42 |

Pre-training CLI (example: 650M on 64 H100s):

```bash
train_esm2 \
  --train-cluster-path /data/train_clusters.parquet \
  --train-database-path /data/train.db \
  --valid-cluster-path /data/valid_clusters.parquet \
  --valid-database-path /data/validation.db \
  --num-steps 500000 \
  --micro-batch-size 32 \
  --num-nodes 8 --num-gpus 8 \
  --val-check-interval 10000 \
  --num-layers 33 --hidden-size 1280 \
  --num-attention-heads 20 --ffn-hidden-size 5120 \
  --result-dir /results/esm2_pretrain_650m
```

### 9. Pydantic Config Workflow

```bash
# Generate validated config
bionemo-esm2-recipe \
  --recipe esm2_650m_recipe \
  --train-cluster-path /data/train_clusters.parquet \
  --train-database-path /data/train.db \
  --valid-cluster-path /data/valid_clusters.parquet \
  --valid-database-path /data/validation.db \
  --result-dir ./results --dest my_config.yaml

# Train from config
bionemo-esm2-train \
  --data-config-cls bionemo.esm2.run.config_models.ESM2DataConfig \
  --model-config-cls bionemo.esm2.run.config_models.ExposedESM2PretrainConfig \
  --config my_config.yaml
```

Available recipes: `esm2_8m_recipe`, `esm2_650m_recipe`, `esm2_3b_recipe`.

## ESM-2 Model Sizes

| Model | Parameters | Hidden Size | Layers | Heads |
|-------|-----------|-------------|--------|-------|
| ESM-2 8M | 8M | 320 | 6 | 20 |
| ESM-2 35M | 35M | 480 | 12 | 20 |
| ESM-2 150M | 150M | 640 | 30 | 20 |
| ESM-2 650M | 650M | 1280 | 33 | 20 |
| ESM-2 3B | 3B | 2560 | 36 | 40 |

## Common Pitfalls

- **Container required**: BioNeMo dependencies are complex. Use the official Docker container.
- **encoder_frozen=True**: Start with frozen encoder for small datasets. Unfreeze for large data.
- **Tokenizer mismatch**: Use `bionemo.esm2.tokenizer.get_tokenizer()`, not generic tokenizers.
- **Sequence length limits**: ESM-2 has position embedding limits. Truncate long sequences.
- **GPU memory**: ESM-2 3B requires multi-GPU. Use 650M for single-GPU fine-tuning.

## Related Skills

- [structure-prediction.md](./structure-prediction.md) - Predict 3D structure from ESM-2 embeddings
- [protein-design.md](./protein-design.md) - Use ESM-2 for protein design validation
- [data-preparation.md](./data-preparation.md) - Prepare sequence data for ESM-2
