---
name: autoresearch-setup
description: "Scaffold an autoresearch-style autonomous experiment loop for any domain. Generates program.md, fixed evaluation harness, mutable experiment file, and results logging — inspired by Karpathy's autoresearch philosophy."
license: MIT
metadata:
  author: Agent Cluster
  tags: [autoresearch, autonomous, experiments, research, optimization, agent-loop]
---

# Autoresearch Setup

Generate a complete autonomous research loop for any domain. Based on the [autoresearch](https://github.com/karpathy/autoresearch) philosophy: an AI agent modifies code, runs experiments on a fixed budget, keeps or discards based on a single metric, and repeats indefinitely.

**When to use:** User wants to set up autonomous experimentation for ML training, kernel optimization, prompt engineering, hyperparameter search, or any domain where iterative improvement can be measured by a single metric.

## Core Philosophy

1. **Single mutable file** — the agent only edits ONE file. Everything else is fixed.
2. **Fixed time budget** — every experiment runs for the same wall-clock time, making results directly comparable.
3. **Single metric** — one number decides keep/discard. Lower or higher, pick one direction.
4. **program.md as research org code** — the human writes strategy in Markdown, not Python. The agent interprets and executes.
5. **Git-backed experiments** — every change is a commit. Keep = advance branch. Discard = reset.
6. **Results in TSV** — plain text, human-readable, machine-parseable.
7. **Never stop** — the agent runs autonomously until manually interrupted.
8. **Simplicity criterion** — if equal performance, simpler code wins. Ugly complexity for tiny gains is not worth it.

## Setup Flow

When the user asks to set up autoresearch for a given context, follow these steps:

### Step 1: Understand the Domain

Ask or infer:
- **What is being optimized?** (model architecture, kernel, prompt, config, algorithm...)
- **What is the metric?** (loss, accuracy, throughput, latency, score...)
- **Metric direction?** (lower is better / higher is better)
- **Time budget per experiment?** (default: 5 minutes)
- **What hardware/environment?** (GPU, CPU, cloud, local...)
- **What are the fixed constraints?** (data, evaluation, dependencies)

### Step 2: Generate Project Structure

Create this structure in the target directory:

```
<project>/
├── program.md        — Agent instructions (human-edited strategy)
├── prepare.py        — Fixed: data prep, evaluation, constants (DO NOT MODIFY)
├── experiment.py     — Mutable: the file the agent edits (ONLY THIS FILE)
├── results.tsv       — Experiment log (git-ignored)
├── pyproject.toml    — Dependencies (locked)
└── .gitignore        — Ignores results.tsv, run.log, __pycache__
```

### Step 3: Generate `program.md`

This is the core of autoresearch. It must contain:

```markdown
# <Project Name> — Autoresearch Program

## Setup

1. **Agree on a run tag** (e.g. `mar11`). Branch: `autoresearch/<tag>`.
2. **Create the branch**: `git checkout -b autoresearch/<tag>`
3. **Read all files** for full context.
4. **Verify prerequisites** (data, dependencies, hardware).
5. **Initialize results.tsv** with header row.
6. **Run baseline** — first run is always unmodified to establish baseline.

## Rules

**You CAN:** Modify `experiment.py` — architecture, hyperparameters, algorithm, approach. Everything is fair game.

**You CANNOT:**
- Modify `prepare.py` (fixed evaluation, data, constants)
- Add new dependencies
- Change the metric or time budget

**Goal:** Optimize <METRIC> (<lower/higher> is better).

**Time budget:** <N> minutes per experiment (wall clock).

**Simplicity criterion:** All else equal, simpler is better. Tiny gains with ugly complexity → discard. Equal performance with less code → keep.

## Output Format

The experiment script prints a summary:
```
---
<metric_name>: <value>
wall_seconds:  <value>
<other_stats>: <value>
```

Extract metric: `grep "^<metric_name>:" run.log`

## Results Logging

TSV format (tab-separated):
```
commit	<metric_name>	status	description
<hash>	<value>	keep	baseline
```

Statuses: `keep`, `discard`, `crash`

## Experiment Loop

LOOP FOREVER:

1. Review current state (git log, results.tsv, code)
2. Form a hypothesis — what change might improve the metric?
3. Edit `experiment.py` with ONE focused change
4. `git commit -am "description of change"`
5. Run: `uv run experiment.py > run.log 2>&1`
6. Extract metric: `grep "^<metric_name>:" run.log`
7. If metric improved → KEEP (advance branch)
8. If metric worse or equal → DISCARD (`git reset --hard HEAD~1`)
9. If crashed → check `tail -50 run.log`, fix if trivial, else discard
10. Log to results.tsv (do NOT commit results.tsv)
11. GOTO 1

**NEVER STOP.** Do not ask if you should continue. The human may be asleep. Run until manually interrupted.

**Timeout:** If a run exceeds 2x the time budget, kill it and treat as failure.

**When stuck:** Re-read the code, try combining near-misses, try radical changes, search for papers referenced in code.
```

### Step 4: Generate `prepare.py`

The fixed harness. Must contain:
- **Constants**: time budget, metric name, evaluation parameters
- **Data loading**: fixed dataset or data generation
- **Evaluation function**: the single metric computation
- **Utilities**: timing, logging, any shared infrastructure

Template pattern:
```python
"""Fixed evaluation harness. DO NOT MODIFY."""
import time

# === CONSTANTS ===
TIME_BUDGET = 300  # seconds (5 minutes)
METRIC_NAME = "<metric>"
METRIC_DIRECTION = "<lower/higher>"  # lower or higher is better

# === DATA ===
def load_data():
    """Load/generate fixed evaluation data."""
    ...

# === EVALUATION ===
def evaluate(model_or_result):
    """Compute the single metric. Returns float."""
    ...

# === TIMING ===
class Timer:
    def __init__(self, budget=TIME_BUDGET):
        self.budget = budget
        self.start = None
    def begin(self):
        self.start = time.time()
    def elapsed(self):
        return time.time() - self.start
    def expired(self):
        return self.elapsed() >= self.budget
```

### Step 5: Generate `experiment.py`

The mutable file. Must:
- Import from `prepare.py`
- Contain the full experiment logic
- Use the timer to respect time budget
- Print the summary format at the end
- Be self-contained (no external state beyond prepare.py)

### Step 6: Generate Supporting Files

**`.gitignore`:**
```
results.tsv
run.log
__pycache__/
*.pyc
.venv/
```

**`pyproject.toml`:** Lock all dependencies. Agent cannot add new ones.

**`results.tsv`:** Header row only:
```
commit	<metric_name>	status	description
```

## Domain Templates

### ML Training (Default)
- Metric: `val_loss` or `val_bpb` (lower is better)
- Mutable: model architecture, optimizer, hyperparameters, training loop
- Fixed: data loading, tokenizer, evaluation, time budget

### Kernel Optimization
- Metric: `throughput_tflops` (higher is better)
- Mutable: kernel implementation (Triton/CUDA)
- Fixed: benchmark harness, correctness checks, reference implementation

### Prompt Engineering
- Metric: `accuracy` or `score` (higher is better)
- Mutable: system prompt, few-shot examples, chain-of-thought structure
- Fixed: evaluation dataset, scoring function, API calls

### Algorithm Optimization
- Metric: `runtime_ms` (lower is better) or `quality_score` (higher is better)
- Mutable: algorithm implementation
- Fixed: test cases, correctness verification, benchmarking harness

### Configuration Tuning
- Metric: domain-specific (latency, throughput, error rate...)
- Mutable: configuration file or parameter set
- Fixed: deployment script, evaluation harness, workload generator

## Post-Setup Checklist

After generating all files, verify:

- [ ] `program.md` has clear rules, metric, time budget, and loop instructions
- [ ] `prepare.py` has evaluation function, constants, and data loading
- [ ] `experiment.py` runs successfully as baseline
- [ ] `results.tsv` has header row
- [ ] `.gitignore` excludes results.tsv and run.log
- [ ] Git repo initialized with initial commit
- [ ] Agent can run the full loop: edit → commit → run → evaluate → keep/discard

## Running

Tell the agent:
```
Read program.md and start the experiment loop. Establish baseline first.
```

Or for Claude Code / Codex:
```
Have a look at program.md and let's kick off a new experiment! Do the setup first.
```

The agent will run autonomously. ~12 experiments/hour at 5-min budget. ~100 overnight.
