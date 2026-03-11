---
title: Geneformer Single-Cell Transcriptomics
impact: HIGH
tags: geneformer, single-cell, scrna, transcriptomics, cell-type, gene-expression, embedding
---

# Skill: Geneformer Single-Cell Transcriptomics

Geneformer is a BERT-style foundation model for single-cell RNA-seq data. Learns co-expression patterns within cells from the CELLxGENE Census. Generates dense cell embeddings used for cell type classification, gene regulation networks, and perturbation prediction.

## Quick Pattern

```python
from bionemo.core.data.load import load

# Load pretrained Geneformer 10M
ckpt_10m = load("geneformer/10M_240530:2.0")

# Load pretrained Geneformer 106M
ckpt_106m = load("geneformer/106M_240530:2.0")
```

## When to Use

- Generating cell-level embeddings from single-cell RNA-seq data
- Cell type classification from gene expression profiles
- Inferring gene co-expression and regulatory networks (GRN)
- Perturbation prediction from PERTURB-seq data
- Clustering and annotating cells by learned representations

## Step-by-Step Instructions

### 1. Model Versions

| Model | Parameters | Hidden | Heads | Layers | FFN | Embedding Dim |
|-------|-----------|--------|-------|--------|-----|---------------|
| geneformer-10M-240530 | 10.3M | 256 | 4 | 6 | 512 | 256 |
| geneformer-106M-240530 | 106M | 768 | 12 | 12 | 3072 | 768 |

Both use: 25,429 ensemble ID gene tokens, ReLU activation, bf16 mixed precision, 2% hidden dropout, 10% attention dropout.

```python
from bionemo.core.data.load import load
ckpt_10m = load("geneformer/10M_240530:2.0")
ckpt_106m = load("geneformer/106M_240530:2.0")
```

### 2. Pre-Training

```bash
download_bionemo_data single_cell/testdata-20241203 --source ngc

train_geneformer \
  --data-dir ${TEST_DATA_DIR}/cellxgene_2023-12-15_small_processed_scdl \
  --result-dir ./results \
  --restore-from-checkpoint-path ${GENEFORMER_10M_CKPT} \
  --experiment-name geneformer_pretrain \
  --num-gpus 1 --num-nodes 1 \
  --val-check-interval 10 \
  --num-dataset-workers 0 \
  --num-steps 55 \
  --seq-length 128 \
  --limit-val-batches 2 \
  --micro-batch-size 2
```

### 3. Fine-Tuning

```bash
train_geneformer \
  --data-dir ${TEST_DATA_DIR}/cellxgene_2023-12-15_small_processed_scdl \
  --result-dir ./results \
  --experiment-name finetune_experiment \
  --training-model-config-class FineTuneSeqLenBioBertConfig \
  --restore-from-checkpoint-path results/pretrain/checkpoints/... \
  --num-gpus 1 --num-nodes 1 \
  --num-steps 55 --seq-length 128 \
  --micro-batch-size 2
```

### 4. Pydantic Config Workflow

```bash
# Generate recipe config
bionemo-geneformer-recipe \
  --recipe 10m-pretrain \
  --dest my_config.json \
  --data-path ${DATA_DIR}/cellxgene_processed_scdl \
  --result-dir ./results

# Train from config
bionemo-geneformer-train \
  --data-config-cls bionemo.geneformer.run.config_models.GeneformerPretrainingDataConfig \
  --model-config-cls bionemo.geneformer.run.config_models.ExposedGeneformerPretrainConfig \
  --config my_config.yaml
```

Available recipes: `10m-pretrain`, `106m-pretrain`, plus fine-tuning example.

### 5. Inference and Embedding Extraction

```bash
infer_geneformer \
  --checkpoint-path /path/to/checkpoint \
  --data-dir ${DATA_DIR}/cellxgene_processed_scdl \
  --results-path ./results/ \
  --include-hiddens
```

Use `infer_geneformer` script for extracting cell embeddings from trained/fine-tuned models.

### 6. Downstream Tasks

- **Cell type classification**: Use embeddings + classifier (e.g., random forest, fine-tuned head)
- **Gene co-expression networks (GRN)**: Gene-level embeddings for regulation networks
- **Perturbation prediction**: PERTURB-seq data analysis

### 7. Training Data

CELLxGENE Census version 2023-12-15:
- 23.87M cells total (23.64M train, 0.13M val, 0.11M test)
- Homo sapiens, non-diseased, primary data only
- Biases: nervous system tissue (~40%), 10x assays dominant, European ethnicity
- Stratified by dataset_id for generalizability

## Performance

| Model | MLM Token Loss | Training |
|-------|---------------|----------|
| Baseline Geneformer | 3.206 | HuggingFace |
| geneformer-10M | 3.18 | 64 A100s, 4 days |
| geneformer-106M | 2.89 | 128 A100s, 8 hours |

106M achieves >50 TFLOPS/GPU. Set `num_dataset_workers=8`, `micro_batch_size=16` (106M) or 120 (10M).

## Common Pitfalls

- **SCDL format**: Data must be in SCDL (Single Cell Data Loader) format. Convert h5ad with `convert_h5ad_to_scdl`.
- **Sequence length**: Default 128 tokens (genes). Increase for richer cells.
- **W&B not auto-configured**: Must manually edit config YAML for Weights & Biases.
- **Dataset workers**: Set `num_dataset_workers` > 0 for optimal throughput.
- **Fine-tune config class**: Must specify `--training-model-config-class FineTuneSeqLenBioBertConfig`.

## Related Skills

- [data-preparation.md](./data-preparation.md) - Single-cell data preprocessing
- [esm2-protein-language.md](./esm2-protein-language.md) - Complementary protein model
