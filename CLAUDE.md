# Agent Skills for Compute

This repository contains agent-optimized skills for NVIDIA GPU-accelerated computing frameworks. Skills provide structured, actionable instructions for AI coding assistants working with quantum computing, physics-informed ML, and computational biology.

## Skills

| Skill | Description |
|-------|-------------|
| [cuda-quantum](skills/cuda-quantum/SKILL.md) | Hybrid quantum-classical programming with CUDA-Q |
| [qiskit](skills/qiskit/SKILL.md) | IBM Quantum platform with runtime primitives and hardware design |
| [physicsnemo](skills/physicsnemo/SKILL.md) | Physics-informed machine learning with PhysicsNeMo |
| [bionemo](skills/bionemo/SKILL.md) | Computational biology and drug discovery with BioNeMo |
| [alchemi-toolkit-ops](skills/alchemi-toolkit-ops/SKILL.md) | GPU cluster operations, Slurm, Docker, Kubernetes |
| [autoresearch-setup](skills/autoresearch-setup/SKILL.md) | Scaffold autonomous experiment loops (Karpathy's autoresearch philosophy) |
| [prime-verifiers](skills/prime-verifiers/SKILL.md) | Prime Intellect verifiers & PrimeRL for LLM post-training |
| [tinker](skills/tinker/SKILL.md) | Thinking Machines Lab training API for LLM fine-tuning |
| [biosimspace](skills/biosimspace/SKILL.md) | Engine-agnostic biomolecular simulation (AMBER, GROMACS, OpenMM) |

## Adding Skills

Follow the [agentskills.io spec](https://agentskills.io/specification). Each skill needs:

- A directory under `skills/<skill-name>/`
- A `SKILL.md` with required frontmatter (name, description)
- Reference files in `references/` for deep-dive content
- Under 500 lines per SKILL.md

Run `/validate-skills` to check compliance.
