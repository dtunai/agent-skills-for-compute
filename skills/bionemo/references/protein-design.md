---
title: Protein Design
impact: HIGH
tags: rfdiffusion, proteinmpnn, inverse-folding, binder, scaffold, generative
---

# Skill: Protein Design

Generate novel protein structures and sequences using RFdiffusion and ProteinMPNN.

## Quick Pattern

```bash
# Generate protein structure with RFdiffusion
curl -X POST http://localhost:8000/biology/ipd/rfdiffusion/generate \
  -H "Content-Type: application/json" \
  -d '{"contigs": "100", "diffusion_steps": 50}'

# Design sequence for structure with ProteinMPNN
curl -X POST http://localhost:8000/biology/ipd/proteinmpnn/predict \
  -H "Content-Type: application/json" \
  -d '{"input_pdb": "<PDB_CONTENT>", "num_seq_per_target": 8}'
```

## When to Use

- Designing novel protein structures de novo
- Generating protein binders for a target protein
- Inverse folding: designing sequences that fold into a given structure
- Scaffold design around functional motifs
- Exploring protein sequence space with controlled diversity

## Step-by-Step Instructions

### 1. RFdiffusion: Generate Protein Structures

RFdiffusion uses diffusion models to generate novel protein backbones.

**De novo protein generation:**
```bash
curl -X POST http://localhost:8000/biology/ipd/rfdiffusion/generate \
  -H "Content-Type: application/json" \
  -d '{
    "contigs": "100",
    "diffusion_steps": 50
  }'
```

**Binder design for a target protein:**
```bash
curl -X POST http://localhost:8000/biology/ipd/rfdiffusion/generate \
  -H "Content-Type: application/json" \
  -d '{
    "input_pdb": "<TARGET_PDB_CONTENT>",
    "contigs": "A10-100/0 50-150",
    "hotspot_res": ["A50", "A51", "A52"],
    "diffusion_steps": 50
  }'
```

**Contigs syntax:**
| Pattern | Meaning |
|---------|---------|
| `100` | Generate 100-residue protein |
| `50-150` | Generate protein with 50-150 residues |
| `A10-100/0 50-150` | Keep residues 10-100 of chain A, generate 50-150 new residues |
| `A1-50/0 B1-50` | Keep chain A residues 1-50, generate chain B |

### 2. RFdiffusion API Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `input_pdb` | string | No | Target protein PDB for binder design |
| `contigs` | string | **Yes** | Defines what to generate (see syntax above) |
| `hotspot_res` | array | No | Target residues to contact (e.g., `["A50", "A51"]`) |
| `diffusion_steps` | int | No | Denoising steps (default: 50, more = higher quality) |
| `random_seed` | int | No | For reproducibility |

**Response:**
```json
{
  "output_pdb": "ATOM  1  N   ...",
  "elapsed_ms": 12345
}
```

### 3. ProteinMPNN: Design Sequences

ProteinMPNN performs inverse folding — given a backbone structure, it designs amino acid sequences that will fold into that structure.

```bash
curl -X POST http://localhost:8000/biology/ipd/proteinmpnn/predict \
  -H "Content-Type: application/json" \
  -d '{
    "input_pdb": "<PDB_FROM_RFDIFFUSION>",
    "num_seq_per_target": 8,
    "sampling_temp": [0.1],
    "use_soluble_model": true
  }'
```

### 4. ProteinMPNN API Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `input_pdb` | string | — | Input backbone structure |
| `num_seq_per_target` | int | 1 | Number of sequences to generate |
| `sampling_temp` | array | — | Temperature (0.1-0.3 recommended, lower = less diverse) |
| `use_soluble_model` | bool | false | Optimized for soluble proteins |
| `ca_only` | bool | false | Use C-alpha only model |
| `fixed_positions_jsonl` | string | — | Lock specific residues |
| `omit_AAs` | array | — | Exclude amino acids |
| `tied_positions_jsonl` | string | — | Force identical residues at positions |

**Response:**
```json
{
  "mfasta": ">design_1\nMKTLIF...\n>design_2\nMKALIF...",
  "scores": [-1.23, -1.45],
  "probs": [...]
}
```

### 5. Python Client for Protein Design

```python
import requests

class ProteinDesigner:
    def __init__(self, rfdiffusion_url="http://localhost:8001",
                 proteinmpnn_url="http://localhost:8002"):
        self.rfdiffusion = rfdiffusion_url
        self.proteinmpnn = proteinmpnn_url

    def generate_structure(self, contigs, target_pdb=None,
                          hotspots=None, steps=50):
        payload = {"contigs": contigs, "diffusion_steps": steps}
        if target_pdb:
            payload["input_pdb"] = target_pdb
        if hotspots:
            payload["hotspot_res"] = hotspots
        resp = requests.post(
            f"{self.rfdiffusion}/biology/ipd/rfdiffusion/generate",
            json=payload)
        resp.raise_for_status()
        return resp.json()["output_pdb"]

    def design_sequences(self, pdb_structure, n_sequences=8, temp=0.1):
        payload = {
            "input_pdb": pdb_structure,
            "num_seq_per_target": n_sequences,
            "sampling_temp": [temp],
            "use_soluble_model": True,
        }
        resp = requests.post(
            f"{self.proteinmpnn}/biology/ipd/proteinmpnn/predict",
            json=payload)
        resp.raise_for_status()
        return resp.json()

    def design_pipeline(self, contigs, n_structures=5, n_seqs=8):
        """Generate structures then design sequences for each."""
        results = []
        for i in range(n_structures):
            structure = self.generate_structure(contigs)
            sequences = self.design_sequences(structure, n_seqs)
            results.append({
                "structure": structure,
                "sequences": sequences["mfasta"],
                "scores": sequences["scores"],
            })
        return results

# Usage
designer = ProteinDesigner()
designs = designer.design_pipeline("100", n_structures=3, n_seqs=4)
```

### 6. Fixed Position Design

Design sequences while keeping specific residues fixed:

```bash
curl -X POST http://localhost:8000/biology/ipd/proteinmpnn/predict \
  -H "Content-Type: application/json" \
  -d '{
    "input_pdb": "<PDB_CONTENT>",
    "num_seq_per_target": 8,
    "fixed_positions_jsonl": "{\"A\": [1, 2, 3, 45, 46, 47]}",
    "sampling_temp": [0.2]
  }'
```

## Design Quality Metrics

| Metric | Good Range | Description |
|--------|-----------|-------------|
| ProteinMPNN score | < -1.5 | Lower = more confident design |
| pLDDT (AlphaFold2) | > 80 | Structure prediction confidence |
| pAE (AlphaFold2) | < 5 | Interface quality for binders |
| RMSD to target | < 2.0 A | Structural similarity |

## Common Pitfalls

- **Contigs syntax errors**: Validate contigs string format. Spaces separate chains, `/0` means no gap.
- **Temperature too high**: `sampling_temp > 0.3` produces highly diverse but less reliable sequences.
- **Not validating designs**: Always run AlphaFold2 on designed sequences to check if they fold correctly.
- **Missing hotspot residues**: For binder design, hotspots dramatically improve binding site targeting.
- **PDB format issues**: Ensure PDB content is properly formatted with standard chain IDs.

## Related Skills

- [structure-prediction.md](./structure-prediction.md) - Validate designs with AlphaFold2
- [nim-microservices.md](./nim-microservices.md) - Deploy design services
- [drug-discovery.md](./drug-discovery.md) - Full binder design pipeline
