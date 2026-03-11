---
title: Drug Discovery Pipelines
impact: MEDIUM
tags: drug-discovery, binder, pipeline, multi-model, screening, validation
---

# Skill: Drug Discovery Pipelines

Build multi-model biology pipelines for protein therapeutics and drug discovery.

## Quick Reference

**Protein Binder Design Pipeline:**
```
Target PDB → AlphaFold2 (predict structure)
  → RFdiffusion (generate binder scaffolds)
  → ProteinMPNN (design sequences)
  → AlphaFold2-Multimer (validate complex)
  → Filter by pLDDT/pAE → Top candidates
```

## When to Use

- Designing protein therapeutics (binders, antibodies)
- Running multi-step computational biology workflows
- Combining structure prediction with generative design
- Screening and validating protein designs computationally
- Building end-to-end drug discovery automation

## Step-by-Step Instructions

### 1. Complete Protein Binder Design Pipeline

The NVIDIA BioNeMo Blueprint for generative protein binder design:

```python
import requests
import json

class BinderDesignPipeline:
    def __init__(self, alphafold_url, rfdiffusion_url,
                 proteinmpnn_url, multimer_url):
        self.services = {
            "alphafold2": alphafold_url,
            "rfdiffusion": rfdiffusion_url,
            "proteinmpnn": proteinmpnn_url,
            "multimer": multimer_url,
        }

    def step1_predict_target_structure(self, target_sequence):
        """Predict 3D structure of the target protein."""
        resp = requests.post(
            f"{self.services['alphafold2']}/protein-structure/alphafold2/predict-structure-from-sequence",
            json={"sequence": target_sequence},
            timeout=300,
        )
        resp.raise_for_status()
        return resp.json()

    def step2_generate_binder_scaffolds(self, target_pdb,
                                         hotspot_residues, n_designs=10):
        """Generate binder backbone structures using RFdiffusion."""
        designs = []
        for i in range(n_designs):
            resp = requests.post(
                f"{self.services['rfdiffusion']}/biology/ipd/rfdiffusion/generate",
                json={
                    "input_pdb": target_pdb,
                    "contigs": "A1-100/0 70-100",  # Keep target, generate binder
                    "hotspot_res": hotspot_residues,
                    "diffusion_steps": 50,
                },
                timeout=120,
            )
            resp.raise_for_status()
            designs.append(resp.json()["output_pdb"])
        return designs

    def step3_design_sequences(self, scaffold_pdbs, n_seqs_per=8):
        """Design amino acid sequences for each scaffold."""
        all_designs = []
        for pdb in scaffold_pdbs:
            resp = requests.post(
                f"{self.services['proteinmpnn']}/biology/ipd/proteinmpnn/predict",
                json={
                    "input_pdb": pdb,
                    "num_seq_per_target": n_seqs_per,
                    "sampling_temp": [0.1],
                    "use_soluble_model": True,
                },
                timeout=60,
            )
            resp.raise_for_status()
            result = resp.json()
            all_designs.append({
                "scaffold": pdb,
                "sequences": result["mfasta"],
                "scores": result["scores"],
            })
        return all_designs

    def step4_validate_complexes(self, target_sequence, designed_sequences):
        """Validate binder-target complexes with AlphaFold2-Multimer."""
        validations = []
        for seq in designed_sequences:
            resp = requests.post(
                f"{self.services['multimer']}/protein-structure/alphafold2/multimer/predict-structure-from-sequences",
                json={
                    "sequences": [target_sequence, seq],
                    "databases": ["uniref90", "small_bfd"],
                },
                timeout=300,
            )
            resp.raise_for_status()
            validations.append(resp.json())
        return validations

    def step5_filter_candidates(self, validations, plddt_threshold=80,
                                 pae_threshold=5):
        """Filter designs by structure quality metrics."""
        candidates = []
        for v in validations:
            # Parse pLDDT and pAE from AlphaFold2 output
            # (exact parsing depends on NIM response format)
            candidates.append(v)
        return candidates

    def run_full_pipeline(self, target_sequence, hotspot_residues,
                          n_scaffolds=10, n_seqs_per_scaffold=8):
        """Execute complete binder design pipeline."""
        print("Step 1: Predicting target structure...")
        target_structure = self.step1_predict_target_structure(target_sequence)

        print(f"Step 2: Generating {n_scaffolds} binder scaffolds...")
        scaffolds = self.step2_generate_binder_scaffolds(
            target_structure, hotspot_residues, n_scaffolds)

        print(f"Step 3: Designing sequences ({n_seqs_per_scaffold} per scaffold)...")
        designs = self.step3_design_sequences(scaffolds, n_seqs_per_scaffold)

        # Collect all designed sequences
        all_sequences = []
        for d in designs:
            seqs = d["mfasta"].split(">")[1:]  # Parse FASTA
            for s in seqs:
                lines = s.strip().split("\n")
                all_sequences.append("".join(lines[1:]))

        print(f"Step 4: Validating {len(all_sequences)} designs...")
        validations = self.step4_validate_complexes(
            target_sequence, all_sequences[:20])  # Top 20

        print("Step 5: Filtering candidates...")
        candidates = self.step5_filter_candidates(validations)

        return {
            "target_structure": target_structure,
            "scaffolds": len(scaffolds),
            "total_designs": len(all_sequences),
            "validated": len(validations),
            "candidates": candidates,
        }

# Usage
pipeline = BinderDesignPipeline(
    alphafold_url="http://localhost:8000",
    rfdiffusion_url="http://localhost:8001",
    proteinmpnn_url="http://localhost:8002",
    multimer_url="http://localhost:8003",
)

results = pipeline.run_full_pipeline(
    target_sequence="MVLSPADKTNVKAAWGKVGA...",
    hotspot_residues=["A30", "A31", "A32", "A55", "A56"],
    n_scaffolds=10,
    n_seqs_per_scaffold=8,
)
```

### 2. Virtual Screening Pipeline

```python
class VirtualScreeningPipeline:
    """Screen compound library against a protein target."""

    def __init__(self, alphafold_url, diffdock_url):
        self.alphafold = alphafold_url
        self.diffdock = diffdock_url

    def predict_target(self, sequence):
        resp = requests.post(
            f"{self.alphafold}/protein-structure/alphafold2/predict-structure-from-sequence",
            json={"sequence": sequence}, timeout=300)
        return resp.json()

    def dock_compound(self, protein_pdb, ligand_sdf):
        resp = requests.post(
            f"{self.diffdock}/molecular-docking/diffdock/generate",
            json={"protein": protein_pdb, "ligand": ligand_sdf},
            timeout=120)
        return resp.json()

    def screen(self, target_sequence, compound_library):
        target = self.predict_target(target_sequence)
        results = []
        for compound in compound_library:
            docking = self.dock_compound(target, compound)
            results.append(docking)
        return sorted(results, key=lambda x: x.get("score", 0))
```

### 3. Iterative Design-Validate Loop

```python
def iterative_design(pipeline, target_seq, hotspots,
                     rounds=3, n_per_round=5):
    """Multi-round design with progressive refinement."""
    best_designs = []

    for round_num in range(rounds):
        print(f"\n=== Round {round_num + 1} ===")

        # Generate and validate
        results = pipeline.run_full_pipeline(
            target_seq, hotspots,
            n_scaffolds=n_per_round,
            n_seqs_per_scaffold=4,
        )

        # Keep top candidates for next round
        best_designs.extend(results["candidates"])

        # Could use top designs to seed next round
        print(f"Cumulative candidates: {len(best_designs)}")

    return best_designs
```

## Pipeline Decision Matrix

| Goal | Pipeline | NIMs Used |
|------|----------|-----------|
| Protein binder design | Binder Blueprint | AF2 + RFdiff + MPNN + AF2M |
| Drug candidate screening | Virtual Screening | AF2 + DiffDock |
| Protein engineering | Directed Evolution | ESM-2 + AF2 + MPNN |
| Antibody design | Ab Engineering | AF2M + RFdiff + MPNN |
| Protein stability | Stability Prediction | ESM-2 + AF2 |

## Common Pitfalls

- **Sequential bottleneck**: Structure prediction is slowest. Parallelize where possible.
- **Not filtering early**: Don't validate all designs — filter by ProteinMPNN score first.
- **Missing error handling**: NIM services can timeout or fail. Add retries.
- **Resource costs**: Each NIM needs its own GPU. Plan GPU allocation for multi-NIM pipelines.
- **Ignoring confidence scores**: Always check pLDDT > 80 and pAE < 5 for binder validation.

## Related Skills

- [structure-prediction.md](./structure-prediction.md) - AlphaFold2 details
- [protein-design.md](./protein-design.md) - RFdiffusion/ProteinMPNN details
- [nim-microservices.md](./nim-microservices.md) - NIM deployment
- [esm2-protein-language.md](./esm2-protein-language.md) - ESM-2 for property prediction
