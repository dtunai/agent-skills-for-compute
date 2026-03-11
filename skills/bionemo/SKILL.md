---
name: bionemo
description: Provides NVIDIA BioNeMo patterns for computational biology and AI-driven drug discovery. Applies to tasks involving protein language models (ESM-2), structure prediction (AlphaFold2), protein design (RFdiffusion, ProteinMPNN), molecular generation, NIM microservice deployment, or multi-model biology pipelines.
license: MIT
metadata:
  author: Agent Cluster
  tags: bionemo, protein, esm2, alphafold2, drug-discovery, nim, nvidia, biology
---

# BioNeMo

## Overview

NVIDIA's software ecosystem for building, training, fine-tuning, and deploying AI models for life sciences. BioNeMo Framework provides optimized biomolecular foundation models (ESM-2, AlphaFold2, ProteinMPNN, RFdiffusion) with distributed training support, while BioNeMo NIMs offer production-ready inference microservices with REST API endpoints for scalable deployment.

## Quick Pattern

**Incorrect — manual protein embedding without framework:**

```python
# Raw transformer, no optimization, no pretrained weights
model = TransformerEncoder(...)
embeddings = model(tokenize(sequence))
```

**Correct — BioNeMo ESM-2 fine-tuning with pretrained checkpoint:**

```python
from bionemo.esm2.model.finetune.finetune_regressor import ESM2FineTuneSeqConfig
from bionemo.core.data.load import load

pretrain_ckpt = load("esm2/650m:2.0")
config = ESM2FineTuneSeqConfig(initial_ckpt_path=str(pretrain_ckpt))

checkpoint, metrics, trainer = train_model(
    experiment_name="finetune_regressor",
    experiment_dir=Path(results_dir),
    config=config,
    data_module=data_module,
    n_steps_train=50,
)
```

## Quick Command

```bash
# Pull BioNeMo container
docker pull nvcr.io/nvidia/bionemo/bionemo-framework:latest

# Run container with GPU
docker run --gpus all -it --rm \
  -v ${PWD}:/workspace \
  nvcr.io/nvidia/bionemo/bionemo-framework:latest bash

# Download ESM-2 checkpoint
python -c "from bionemo.core.data.load import load; load('esm2/650m:2.0')"

# Run ESM-2 inference
infer_esm2 --checkpoint-path /path/to/ckpt \
  --data-path data.csv \
  --results-path results/

# NIM health check
curl http://localhost:8000/v1/health/ready

# RFdiffusion via NIM
curl -X POST http://localhost:8000/biology/ipd/rfdiffusion/generate \
  -H "Content-Type: application/json" \
  -d '{"contigs": "A10-100/0 50-150"}'
```

## Quick Reference

### Supported Models

| Model | Type | Task |
|-------|------|------|
| ESM-2 | Protein language model | Sequence embedding, property prediction |
| AlphaFold2 | Structure prediction | 3D protein folding |
| AlphaFold2-Multimer | Complex prediction | Multi-chain structure |
| ProteinMPNN | Inverse folding | Sequence design from structure |
| RFdiffusion | Generative diffusion | De novo protein structure generation |
| Evo2 | Genomic foundation model | DNA/RNA generation, variant prediction, 1B/7B/40B |
| Geneformer | Single-cell transformer | Cell embeddings, type classification, GRN, 10M/106M |
| AMPLIFY | Protein language model | ESM-2 variant, 120M/350M, modified layers |
| DiffDock | Docking model | Protein-ligand binding |
| MolMIM | Molecular model | Small molecule generation |

### NIM Microservice Endpoints

| Service | Endpoint | Method |
|---------|----------|--------|
| AlphaFold2 | `/protein-structure/alphafold2/predict-structure-from-sequence` | POST |
| AlphaFold2-Multimer | `/protein-structure/alphafold2/multimer/predict-structure-from-sequences` | POST |
| RFdiffusion | `/biology/ipd/rfdiffusion/generate` | POST |
| ProteinMPNN | `/biology/ipd/proteinmpnn/predict` | POST |
| Health check | `/v1/health/ready` | GET |

### Protein Binder Design Pipeline

```
Target PDB → AlphaFold2 (structure) → RFdiffusion (binder scaffold)
  → ProteinMPNN (sequence design) → AlphaFold2-Multimer (validation)
```

### Key Packages

| Package | Purpose |
|---------|---------|
| `bionemo-esm2` | ESM-2 model, training, fine-tuning, inference |
| `bionemo-core` | Shared utilities, data loading, checkpoint management |
| `bionemo-geneformer` | Single-cell transcriptomics |
| `bionemo-amplify` | AMPLIFY protein language model (ESM-2 variant) |
| `bionemo-evo2` | Evo2 genomic foundation model |
| `bionemo-moco` | Continuous/discrete generative modeling |
| `bionemo-llm` | Shared LLM utilities, BioBERT, training |
| `bionemo-scdl` | Single Cell Data Loader format |

## When to Apply

Reference these guidelines when:
- Working with protein language models (ESM-2 embeddings, fine-tuning)
- Predicting protein 3D structures (AlphaFold2)
- Designing novel proteins or binders (RFdiffusion, ProteinMPNN)
- Building drug discovery pipelines
- Deploying biology AI models as microservices (NIM)
- Processing biological sequence data
- Analyzing genomic sequences (DNA/RNA variant effects, generation) with Evo2
- Processing single-cell RNA-seq data (cell type classification, GRN) with Geneformer
- Setting up distributed training infrastructure (NeMo2, Megatron, TransformerEngine)
- Running multi-model biology workflows

## Priority-Ordered Guidelines

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | ESM-2 Protein Language | CRITICAL | `esm2-*` |
| 2 | Structure Prediction | CRITICAL | `structure-*` |
| 3 | Protein Design | HIGH | `protein-*` |
| 4 | NIM Microservices | HIGH | `nim-*` |
| 5 | Evo2 Genomic Model | HIGH | `evo2-*` |
| 6 | Geneformer Single-Cell | HIGH | `geneformer-*` |
| 7 | Training Infrastructure | HIGH | `training-*` |
| 8 | Drug Discovery | MEDIUM | `drug-*` |
| 9 | Data Preparation | MEDIUM | `data-*` |

## References

Full documentation with code examples in [references/](references/):

| File | Impact | Description |
|------|--------|-------------|
| [esm2-protein-language.md][esm2-protein-language] | CRITICAL | ESM-2 training, fine-tuning, inference, embeddings |
| [structure-prediction.md][structure-prediction] | CRITICAL | AlphaFold2 folding, MSA, multimer prediction |
| [protein-design.md][protein-design] | HIGH | RFdiffusion generation, ProteinMPNN inverse folding |
| [nim-microservices.md][nim-microservices] | HIGH | NIM API endpoints, deployment, health checks |
| [drug-discovery.md][drug-discovery] | MEDIUM | Multi-model pipelines, binder design blueprint |
| [evo2-genomic-model.md][evo2-genomic-model] | HIGH | Evo2 DNA/RNA model, variant prediction, generation |
| [geneformer-single-cell.md][geneformer-single-cell] | HIGH | Geneformer single-cell embeddings, cell type classification |
| [training-infrastructure.md][training-infrastructure] | HIGH | Hardware, containers, distributed training, recipes |
| [data-preparation.md][data-preparation] | MEDIUM | Biological data preprocessing, tokenization, datasets |

## Problem -> Skill Mapping

| Problem | Start With |
|---------|------------|
| Get protein embeddings | [esm2-protein-language.md][esm2-protein-language] |
| Predict protein structure | [structure-prediction.md][structure-prediction] |
| Design new proteins | [protein-design.md][protein-design] |
| Deploy model as API | [nim-microservices.md][nim-microservices] |
| Build drug discovery pipeline | [drug-discovery.md][drug-discovery] |
| Prepare biological data | [data-preparation.md][data-preparation] |
| Fine-tune ESM-2 | [esm2-protein-language.md][esm2-protein-language] |
| Validate protein binder | [drug-discovery.md][drug-discovery] |
| Predict variant effects on DNA/RNA | [evo2-genomic-model.md][evo2-genomic-model] |
| Generate DNA/RNA sequences | [evo2-genomic-model.md][evo2-genomic-model] |
| Classify cell types from scRNA-seq | [geneformer-single-cell.md][geneformer-single-cell] |
| Build gene regulatory networks | [geneformer-single-cell.md][geneformer-single-cell] |
| Set up distributed training | [training-infrastructure.md][training-infrastructure] |
| Pre-train ESM-2 from scratch | [esm2-protein-language.md][esm2-protein-language] -> [training-infrastructure.md][training-infrastructure] |
| Convert h5ad to SCDL | [data-preparation.md][data-preparation] |
| Run AlphaFold2 inference | [structure-prediction.md][structure-prediction] -> [nim-microservices.md][nim-microservices] |

[esm2-protein-language]: references/esm2-protein-language.md
[structure-prediction]: references/structure-prediction.md
[protein-design]: references/protein-design.md
[nim-microservices]: references/nim-microservices.md
[drug-discovery]: references/drug-discovery.md
[evo2-genomic-model]: references/evo2-genomic-model.md
[geneformer-single-cell]: references/geneformer-single-cell.md
[training-infrastructure]: references/training-infrastructure.md
[data-preparation]: references/data-preparation.md

## Attribution

Based on [NVIDIA BioNeMo Framework](https://nvidia.github.io/bionemo-framework/) documentation, [bionemo-framework](https://github.com/NVIDIA/bionemo-framework) repository, and [generative-protein-binder-design](https://github.com/NVIDIA-BioNeMo-blueprints/generative-protein-binder-design) blueprint.
