---
title: Structure Prediction
impact: CRITICAL
tags: alphafold2, folding, structure, msa, multimer, pdb, 3d
---

# Skill: Structure Prediction

Predict 3D protein structures using AlphaFold2 and AlphaFold2-Multimer via BioNeMo NIM.

## Quick Pattern

```bash
# Predict single protein structure
curl -X POST http://localhost:8000/protein-structure/alphafold2/predict-structure-from-sequence \
  -H "Content-Type: application/json" \
  -d '{"sequence": "MVLSPADKTNVKAAWGKVGAHAGEYGAEALERMFLSFPTTKTYFPHFDLSH"}'

# Predict multi-chain complex
curl -X POST http://localhost:8000/protein-structure/alphafold2/multimer/predict-structure-from-sequences \
  -H "Content-Type: application/json" \
  -d '{"sequences": ["MNVIDIAIAMAI", "IAMNVIDIAAI"], "databases": ["uniref90", "mgnify", "small_bfd"]}'
```

## When to Use

- Predicting 3D structure of a protein from its amino acid sequence
- Modeling multi-chain protein complexes
- Validating designed protein structures
- Understanding protein-protein interactions
- Generating structural features for downstream ML tasks

## Step-by-Step Instructions

### 1. Deploy AlphaFold2 NIM

```bash
# Pull and run AlphaFold2 NIM container
docker run --gpus all -p 8000:8000 \
  -e NGC_API_KEY=$NGC_API_KEY \
  nvcr.io/nim/nvidia/alphafold2:latest

# Wait for readiness
curl http://localhost:8000/v1/health/ready
# {"status": "ready"}
```

### 2. Single Sequence Structure Prediction

```bash
curl -X POST \
  http://localhost:8000/protein-structure/alphafold2/predict-structure-from-sequence \
  -H "Content-Type: application/json" \
  -d '{
    "sequence": "MVLSPADKTNVKAAWGKVGAHAGEYGAEALERMFLSFPTTKTYFPHFDLSH"
  }'
```

**Response**: PDB-formatted 3D structure with confidence scores (pLDDT).

### 3. Multi-Chain Complex Prediction (Multimer)

```bash
curl -X POST \
  http://localhost:8000/protein-structure/alphafold2/multimer/predict-structure-from-sequences \
  -H "Content-Type: application/json" \
  -d '{
    "sequences": ["PROTEIN_A_SEQUENCE", "PROTEIN_B_SEQUENCE"],
    "databases": ["uniref90", "mgnify", "small_bfd"]
  }'
```

### 4. Python Client for AlphaFold2 NIM

```python
import requests
import json

class AlphaFold2Client:
    def __init__(self, base_url="http://localhost:8000"):
        self.base_url = base_url

    def predict_structure(self, sequence: str) -> str:
        """Predict 3D structure for a single protein sequence."""
        response = requests.post(
            f"{self.base_url}/protein-structure/alphafold2/predict-structure-from-sequence",
            json={"sequence": sequence},
        )
        response.raise_for_status()
        return response.json()

    def predict_multimer(self, sequences: list[str],
                         databases: list[str] = None) -> str:
        """Predict multi-chain complex structure."""
        payload = {"sequences": sequences}
        if databases:
            payload["databases"] = databases
        response = requests.post(
            f"{self.base_url}/protein-structure/alphafold2/multimer/predict-structure-from-sequences",
            json=payload,
        )
        response.raise_for_status()
        return response.json()

    def health_check(self) -> bool:
        response = requests.get(f"{self.base_url}/v1/health/ready")
        return response.json().get("status") == "ready"

# Usage
client = AlphaFold2Client()
pdb_result = client.predict_structure("MVLSPADKTNVKAAWGKVGA...")
```

### 5. MSA (Multiple Sequence Alignment)

AlphaFold2 uses MSA for evolutionary information:

```bash
# MMseqs2 GPU-accelerated MSA search
# Typically handled internally by the NIM container
# Databases: uniref90, mgnify, small_bfd, pdb70, uniclust30
```

**Available databases:**
| Database | Description | Size |
|----------|-------------|------|
| `uniref90` | UniRef90 clusters | ~100GB |
| `mgnify` | Metagenomics sequences | ~64GB |
| `small_bfd` | BFD subset | ~17GB |
| `pdb70` | PDB structures at 70% identity | ~56GB |
| `uniclust30` | UniClust at 30% identity | ~86GB |

### 6. Understanding AlphaFold2 Output

```python
# AlphaFold2 produces:
# - PDB file with predicted 3D coordinates
# - pLDDT scores (per-residue confidence, 0-100)
# - PAE (Predicted Aligned Error) for domain analysis

# pLDDT interpretation:
# > 90: Very high confidence
# 70-90: Confident
# 50-70: Low confidence (often loops or disordered)
# < 50: Very low confidence (likely disordered)
```

### 7. Batch Structure Prediction

```python
import concurrent.futures

sequences = [
    "SEQUENCE_1...",
    "SEQUENCE_2...",
    "SEQUENCE_3...",
]

client = AlphaFold2Client()

# Parallel prediction
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(client.predict_structure, seq): seq
               for seq in sequences}
    for future in concurrent.futures.as_completed(futures):
        seq = futures[future]
        try:
            result = future.result()
            print(f"Predicted structure for {seq[:20]}...")
        except Exception as e:
            print(f"Failed: {e}")
```

## Performance Notes

| Configuration | Speed | Notes |
|--------------|-------|-------|
| AlphaFold2 NIM (A100) | ~30-60s per protein | GPU-accelerated MSA + inference |
| AlphaFold2 NIM (H100) | ~15-30s per protein | Faster with H100 |
| CPU baseline | 10-30 minutes | Not recommended |

## Common Pitfalls

- **NGC API key required**: Set `NGC_API_KEY` environment variable before launching NIM.
- **Database download time**: First launch downloads MSA databases (~200GB+). Use pre-cached volumes.
- **Sequence length limits**: Very long sequences (>2000 residues) may OOM. Split into domains.
- **Non-standard amino acids**: Remove non-standard residues before prediction.
- **Multimer stoichiometry**: Provide each chain separately. Repeated chains should be listed multiple times.

## Related Skills

- [protein-design.md](./protein-design.md) - Design proteins, then predict their structure
- [nim-microservices.md](./nim-microservices.md) - NIM deployment patterns
- [drug-discovery.md](./drug-discovery.md) - Structure prediction in drug pipelines
