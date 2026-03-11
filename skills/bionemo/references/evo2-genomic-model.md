---
title: Evo2 Genomic Foundation Model
impact: HIGH
tags: evo2, genomics, dna, rna, striped-hyena, variant-prediction, generation
---

# Skill: Evo2 Genomic Foundation Model

Predict and generate DNA/RNA sequences with Evo2, a 40B-parameter genomic foundation model from Arc Institute built on the StripedHyena architecture, trained on ~9 trillion nucleotides from OpenGenome, GTDB, NCBI eukaryotic genomes, metagenomes, and organelles.

## Quick Pattern

```python
from bionemo.core.data.load import load

# Load pretrained Evo2 7B (8K context)
ckpt_7b = load("evo2/7b-8k:1.0")

# Run inference on a DNA sequence
# python -m bionemo.evo2.run.infer \
#   --checkpoint-path $ckpt_7b \
#   --prompt "ATGCGATCGATCG" \
#   --max-length 1000
```

## When to Use

- Predicting functional impact of single nucleotide variants (zero-shot)
- Generating novel DNA/RNA sequences
- Scoring genomic sequences by likelihood
- Fine-tuning on custom genomic datasets (e.g., specific organism, regulatory elements)
- Long-range genomic context modeling (up to 1M nucleotides with 7b-1m/40b-1m)

## Step-by-Step Instructions

### 1. Download Pretrained Checkpoints

```python
from bionemo.core.data.load import load

ckpt_1b = load("evo2/1b-8k:1.0")           # 1B params, 8K context
ckpt_1b_bf16 = load("evo2/1b-8k-bf16:1.0")  # 1B, bf16-compatible
ckpt_7b = load("evo2/7b-8k:1.0")            # 7B params, 8K context
ckpt_7b_1m = load("evo2/7b-1m:1.0")         # 7B, 1M context
```

**Hardware compatibility:**

| Model | Blackwell FP8 | Blackwell BF16 | Hopper FP8 | Hopper BF16 | Ampere |
|-------|---------------|----------------|------------|-------------|--------|
| 1b-8k | Yes | No | Yes | No | No |
| 1b-8k-bf16 | Yes | Yes | Yes | Yes | Yes |
| 7b-8k | Yes | Yes | Yes | Yes | Yes |
| 7b-1m | Yes | Yes | Yes | Yes | Yes |
| 40b-1m-fp8-bf16 | Yes | Yes | Yes | Yes | Yes |

The original 1b-8k checkpoint has low accuracy on bf16 and Ampere GPUs. Use the 1b-8k-bf16 variant for any non-FP8 hardware.

### 2. Preprocess FASTA Data

Evo2 uses Megatron-style datasets. Convert raw FASTA files to the preprocessed format:

```yaml
# preprocess_config.yaml
- datapaths: ["/path/to/genome.fa"]
  output_dir: "/path/to/output"
  output_prefix: my_genome
  train_split: 0.9
  valid_split: 0.05
  test_split: 0.05
  overwrite: True
  embed_reverse_complement: true
  indexed_dataset_dtype: "uint8"
  tokenizer_type: "Byte-Level"
  transcribe: "back_transcribe"
  force_uppercase: false
```

```bash
preprocess_evo2 --config preprocess_config.yaml
```

### 3. Fine-Tune on Custom Data

```bash
# Fine-tune 1B model on single GPU
python -m bionemo.evo2.run.train \
  --initial-ckpt-path $(download_bionemo_data evo2/1b-8k:1.0) \
  --data-config /path/to/preprocessed \
  --micro-batch-size 4 \
  --devices 1 \
  --num-steps 1000
```

For multi-GPU training, use tensor parallelism to split the model across devices:

```bash
python -m bionemo.evo2.run.train \
  --initial-ckpt-path $(download_bionemo_data evo2/7b-8k:1.0) \
  --data-config /path/to/preprocessed \
  --micro-batch-size 2 \
  --devices 4 \
  --tensor-parallel-size 4 \
  --num-steps 5000
```

### 4. Run Inference and Generate Sequences

```bash
# Generate DNA sequences from a prompt
python -m bionemo.evo2.run.infer \
  --checkpoint-path $(download_bionemo_data evo2/7b-8k:1.0) \
  --prompt "ATGCGATCGATCG" \
  --max-length 1000
```

### 5. Zero-Shot Variant Effect Prediction

Score reference vs variant sequences to predict functional impact of single nucleotide variants (SNVs):

1. Compute log-likelihood scores for the reference sequence
2. Compute log-likelihood scores for the variant sequence
3. Compare scores to determine predicted variant effect

**Benchmark (BRCA1 Findlay et al., 3,893 SNVs):**

| Model | AUROC |
|-------|-------|
| Evo2 1B | 0.76 |
| Evo2 7B | 0.87 |

### 6. Convert HuggingFace Checkpoints to NeMo2

Original Savanna-format models from HuggingFace can be converted for use with BioNeMo:

- `arcinstitute/savanna_evo2_1b_base`
- `arcinstitute/savanna_evo2_7b_base`
- `arcinstitute/savanna_evo2_7b`
- `arcinstitute/savanna_evo2_40b_base`
- `arcinstitute/savanna_evo2_40b`

## Performance

| Model | Context | Relative Throughput |
|-------|---------|---------------------|
| 7B | 8K | Baseline |
| 7B | 1M | ~9.7x slower |
| 40B | 8K | ~4.9x slower than 7B |

## Common Pitfalls

- **FP8 sensitivity**: The 1b-8k checkpoint requires FP8 hardware for accurate scoring. Use 1b-8k-bf16 for Ampere or bf16-only GPUs.
- **Preprocessing required**: Raw FASTA files must be converted to Megatron format with `preprocess_evo2` before training or fine-tuning.
- **Memory for 40B**: The 40B model requires multi-GPU setups with tensor parallelism.
- **Context length trade-off**: 8K context is the default. The 1M context models (7b-1m, 40b-1m) enable long-range modeling but are significantly slower.
- **Container required**: BioNeMo dependencies are complex. Use the official Docker container.

## Related Skills

- [data-preparation.md](./data-preparation.md) - Biological data preprocessing
- [esm2-protein-language.md](./esm2-protein-language.md) - Protein language model (complementary to genomic model)
