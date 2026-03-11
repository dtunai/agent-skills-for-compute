---
title: Data Preparation
impact: MEDIUM
tags: data, tokenization, fasta, pdb, sequences, preprocessing, dataset
---

# Skill: Data Preparation

Prepare biological data for BioNeMo model training, fine-tuning, and inference.

## Quick Pattern

```python
from bionemo.esm2.data.tokenizer import get_tokenizer
from bionemo.esm2.model.finetune.finetune_regressor import InMemorySingleValueDataset

# Tokenize protein sequences
tokenizer = get_tokenizer()
tokens = tokenizer.text_to_tokens("MVLSPADKTNVKAAWGKVGA")

# Create dataset from (sequence, label) pairs
data = [("MVLSPADKTNVKA...", 0.85), ("GRFNVWLGGNES...", 0.42)]
dataset = InMemorySingleValueDataset(data)
```

## When to Use

- Preparing protein sequences for ESM-2 training/inference
- Processing PDB structure files for structure prediction
- Building datasets for fine-tuning BioNeMo models
- Converting between biological data formats
- Cleaning and filtering sequence/structure data

## Step-by-Step Instructions

### 1. Protein Sequence Tokenization

```python
from bionemo.esm2.data import tokenizer

# Get ESM-2 tokenizer
tok = tokenizer.get_tokenizer()

# Tokenize sequence
sequence = "MVLSPADKTNVKAAWGKVGAHAGEYGAEALERMFLSFPTTKTYFPHFDLSH"
tokens = tok.text_to_tokens(sequence)
ids = tok.tokens_to_ids(tokens)

print(f"Sequence length: {len(sequence)}")
print(f"Token count: {len(tokens)}")
print(f"Token IDs: {ids[:10]}...")
```

### 2. In-Memory Dataset for Fine-Tuning

```python
import numpy as np
from bionemo.esm2.model.finetune.finetune_regressor import InMemorySingleValueDataset
from bionemo.esm2.data import tokenizer

# (sequence, value) pairs
data = [
    ("TLILGWSDKLGSLLNQLAIANESLGGGTIAVMAERDKEDMELDIGKMEFDFKGTSVI", 0.57),
    ("LYSGDHSTQGARFLRDLAENTGRAEYELLSLF", 0.32),
    ("GRFNVWLGGNESKIRQVLKAVKEIGVSPTLFAVYEKN", 0.38),
    ("DELTALGGLLHDIGKPVQRAGLYSGDHSTQGARFLRDLAENTGRAEYELLSLF", 0.55),
    ("KLGSLLNQLAIANESLGGGTIAVMAERDKEDMELDIGKMEFDFKGTSVI", 0.49),
]

dataset = InMemorySingleValueDataset(
    data=data,
    tokenizer=tokenizer.get_tokenizer(),
    seed=42,
)

# Access a sample
sample = dataset[0]
print(sample.keys())
```

### 3. CSV/TSV to Dataset

```python
import csv
from pathlib import Path

def load_sequences_from_csv(filepath, seq_col="sequence", label_col="value"):
    """Load protein sequence data from CSV."""
    data = []
    with open(filepath) as f:
        reader = csv.DictReader(f)
        for row in reader:
            seq = row[seq_col].strip().upper()
            value = float(row[label_col])
            data.append((seq, value))
    return data

# Example CSV format:
# sequence,value
# MVLSPADKTNVKA...,0.85
# GRFNVWLGGNES...,0.42

data = load_sequences_from_csv("training_data.csv")
dataset = InMemorySingleValueDataset(data)
```

### 4. FASTA File Processing

```python
def parse_fasta(filepath):
    """Parse FASTA file into list of (header, sequence) tuples."""
    sequences = []
    current_header = None
    current_seq = []

    with open(filepath) as f:
        for line in f:
            line = line.strip()
            if line.startswith(">"):
                if current_header:
                    sequences.append((current_header, "".join(current_seq)))
                current_header = line[1:]
                current_seq = []
            else:
                current_seq.append(line)

    if current_header:
        sequences.append((current_header, "".join(current_seq)))

    return sequences

# Usage
sequences = parse_fasta("proteins.fasta")
print(f"Loaded {len(sequences)} sequences")
```

### 5. PDB File Handling

```python
def read_pdb(filepath):
    """Read PDB file and return as string."""
    with open(filepath) as f:
        return f.read()

def extract_sequence_from_pdb(pdb_content):
    """Extract amino acid sequence from PDB ATOM records."""
    three_to_one = {
        'ALA': 'A', 'CYS': 'C', 'ASP': 'D', 'GLU': 'E', 'PHE': 'F',
        'GLY': 'G', 'HIS': 'H', 'ILE': 'I', 'LYS': 'K', 'LEU': 'L',
        'MET': 'M', 'ASN': 'N', 'PRO': 'P', 'GLN': 'Q', 'ARG': 'R',
        'SER': 'S', 'THR': 'T', 'VAL': 'V', 'TRP': 'W', 'TYR': 'Y',
    }
    residues = {}
    for line in pdb_content.split('\n'):
        if line.startswith('ATOM') and line[12:16].strip() == 'CA':
            chain = line[21]
            resnum = int(line[22:26])
            resname = line[17:20].strip()
            residues[(chain, resnum)] = three_to_one.get(resname, 'X')

    # Group by chain
    chains = {}
    for (chain, resnum), aa in sorted(residues.items()):
        chains.setdefault(chain, []).append(aa)

    return {chain: "".join(aas) for chain, aas in chains.items()}

# Usage
pdb = read_pdb("target.pdb")
sequences = extract_sequence_from_pdb(pdb)
print(sequences)  # {'A': 'MVLSPAD...', 'B': 'GRFNVWL...'}
```

### 6. Data Splitting

```python
import numpy as np

def split_data(data, train_ratio=0.8, val_ratio=0.1, seed=42):
    """Split data into train/val/test sets."""
    rng = np.random.default_rng(seed)
    indices = rng.permutation(len(data))

    n_train = int(len(data) * train_ratio)
    n_val = int(len(data) * val_ratio)

    train_idx = indices[:n_train]
    val_idx = indices[n_train:n_train + n_val]
    test_idx = indices[n_train + n_val:]

    return (
        [data[i] for i in train_idx],
        [data[i] for i in val_idx],
        [data[i] for i in test_idx],
    )

train, val, test = split_data(data)
train_dataset = InMemorySingleValueDataset(train)
val_dataset = InMemorySingleValueDataset(val)
```

### 7. Sequence Filtering and Cleaning

```python
STANDARD_AAS = set("ACDEFGHIKLMNPQRSTVWY")

def clean_sequence(seq):
    """Remove non-standard amino acids and whitespace."""
    seq = seq.strip().upper()
    seq = "".join(c for c in seq if c in STANDARD_AAS)
    return seq

def filter_sequences(data, min_length=10, max_length=1024):
    """Filter sequences by length."""
    filtered = []
    for seq, label in data:
        seq = clean_sequence(seq)
        if min_length <= len(seq) <= max_length:
            filtered.append((seq, label))
    return filtered

# Clean and filter
clean_data = filter_sequences(data, min_length=20, max_length=512)
print(f"Filtered: {len(data)} -> {len(clean_data)} sequences")
```

### 8. CELLxGENE Census for Geneformer

Download single-cell data for Geneformer pre-training/fine-tuning:

```python
import cellxgene_census

# Download from CELLxGENE Census
census = cellxgene_census.open_soma(census_version="2023-12-15")
# Query Homo sapiens, primary, non-diseased cells
```

Convert h5ad to SCDL format (required for Geneformer):

```bash
# Convert AnnData h5ad to SCDL format
convert_h5ad_to_scdl --input data.h5ad --output-dir ./scdl_output
```

CELLxGENE Census (v2023-12-15) statistics:
- 23.87M cells (23.64M train / 0.13M val / 0.11M test)
- Homo sapiens, non-diseased, primary data only
- Known biases: nervous system tissue (~40%), 10x assays dominant, European ethnicity
- Stratified train/holdout split by dataset_id

### 9. UniProt Dataset for ESM-2

ESM-2 pre-training uses UniProt 2024_03 release:

```python
from bionemo.core.data.load import load
sanity_data = load("esm2/testdata_esm2_pretrain:2.0")  # ~150MB test slice
```

- 65.7M UniRef50 clusters → 328,360 validation sequences (0.5% of clusters)
- MMSeqs removes training sequences similar to validation
- Training: 65.2M UniRef50 clusters → 187.4M UniRef90 sequences
- Batches: sample one UniRef50 cluster → pick random UniRef90 sequence

Full dataset ~80GB, sanity slice ~150MB (10,000 UniRef50 clusters).

### 10. Evo2 FASTA Preprocessing

Convert raw FASTA files to Megatron format for Evo2:

```yaml
# preprocess_config.yaml
- datapaths: ["/path/to/genome.fa"]
  output_dir: "/path/to/output"
  output_prefix: my_genome
  train_split: 0.9
  valid_split: 0.05
  test_split: 0.05
  embed_reverse_complement: true
  indexed_dataset_dtype: "uint8"
  tokenizer_type: "Byte-Level"
  transcribe: "back_transcribe"
```

```bash
preprocess_evo2 --config preprocess_config.yaml
```

## Data Format Requirements

| Model | Input Format | Key Requirements |
|-------|-------------|-----------------|
| ESM-2 | Protein sequences (FASTA/parquet) | Standard amino acids, < 1024 residues |
| Geneformer | Single-cell (SCDL from h5ad) | Gene expression counts, 25K gene tokens |
| Evo2 | DNA/RNA (FASTA→Megatron) | Byte-level tokenizer, preprocessed |
| AlphaFold2 | Protein sequence + MSA databases | Single sequence or multi-chain |
| ProteinMPNN | PDB structure file | Valid backbone coordinates |
| RFdiffusion | PDB structure + contigs | Chain IDs, residue numbering |
| DiffDock | PDB (protein) + SDF (ligand) | Clean structures, no solvent |

## Common Pitfalls

- **Non-standard amino acids**: Remove U (selenocysteine), O (pyrrolysine), B, Z, X, J before tokenization.
- **Sequence length limits**: ESM-2 has position embedding limits (~1024). Truncate or split long proteins.
- **Missing chain IDs in PDB**: PDB files for NIM endpoints need proper chain identifiers.
- **Label normalization**: For regression, normalize labels to [0,1] or standardize for better training.
- **Duplicate sequences**: Remove exact duplicates. Use CD-HIT for sequence identity-based deduplication.
- **Wrong tokenizer**: Always use `bionemo.esm2.data.tokenizer.get_tokenizer()`, not generic tokenizers.
- **SCDL format required**: Geneformer cannot read h5ad directly. Must convert with `convert_h5ad_to_scdl`.
- **Evo2 preprocessing mandatory**: Raw FASTA must pass through `preprocess_evo2` before training/fine-tuning.

## Related Skills

- [esm2-protein-language.md](./esm2-protein-language.md) - Use prepared data for ESM-2
- [geneformer-single-cell.md](./geneformer-single-cell.md) - Single-cell data for Geneformer
- [evo2-genomic-model.md](./evo2-genomic-model.md) - Genomic data for Evo2
- [structure-prediction.md](./structure-prediction.md) - PDB data for structure prediction
- [protein-design.md](./protein-design.md) - Data for design pipelines
