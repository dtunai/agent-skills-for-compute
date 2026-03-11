# Agent Skills for Compute

Markdown-only repository of agent-optimized skills for GPU-TPU-QPU-accelerated computing. No build step required.

## Structure

```
skills/
  cuda-quantum/          # CUDA-Q quantum computing (NVIDIA)
    SKILL.md             # Main skill file
    references/          # Deep-dive reference docs
  qiskit/                # Qiskit quantum computing (IBM)
    SKILL.md
    references/
  physicsnemo/           # PhysicsNeMo physics-informed ML
    SKILL.md
    references/
  bionemo/               # BioNeMo computational biology
    SKILL.md
    references/
  alchemi-toolkit-ops/   # GPU infrastructure and MLOps
    SKILL.md
    references/
```

## Skill Format

Each skill follows a hybrid format:
- **Quick Pattern**: Correct/incorrect code snippets
- **Quick Command**: Shell commands for common tasks
- **Quick Config**: Configuration snippets
- **Quick Reference**: Summary tables
- **Deep Dive**: Full context in reference files

## Adding New Skills

See [agentskills.io specification](https://agentskills.io/specification) and run `/validate-skills`.
