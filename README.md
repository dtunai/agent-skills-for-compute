# Agent Skills for Compute

Agent-optimized skills for the full LLM lifecycle — pre-training, post-training (RL/DPO/RLHF), inference, and autonomous research — plus GPU/TPU/QPU kernel programming, simulation, and scientific computing.

Structured, actionable instructions for AI coding assistants (Claude Code, Cursor, Codex, Gemini CLI, OpenCode).

## Available Skills

| Skill | Domain | Key Technologies |
|-------|--------|-----------------|
| **[cuda-quantum](skills/cuda-quantum/SKILL.md)** | Quantum Computing | CUDA-Q kernels, cuQuantum simulation, VQE/QAOA, multi-GPU QPU |
| **[qiskit](skills/qiskit/SKILL.md)** | Quantum Computing | IBM Quantum, runtime primitives, transpiler, VQE/QAOA, Qiskit Metal |
| **[slurm](skills/slurm/SKILL.md)** | HPC Job Scheduling | sbatch, srun, GPU/GRES allocation, job arrays, MPI, cluster management |
| **[dask](skills/dask/SKILL.md)** | Parallel Computing | DataFrame, Array, Bag, distributed scheduler, GPU (cuDF/CuPy), HPC deployment |
| **[ray](skills/ray/SKILL.md)** | Distributed AI Compute | Tasks, actors, Ray Data, Ray Train, Ray Serve, Ray Tune, RLlib |
| **[mpi](skills/mpi/SKILL.md)** | Message Passing | Point-to-point, collectives, MPI-IO, Open MPI, MPICH, mpi4py |
| **[polars](skills/polars/SKILL.md)** | GPU DataFrames | Lazy evaluation, expressions API, RAPIDS cuDF GPU engine, query optimization |
| **[rdkit](skills/rdkit/SKILL.md)** | Cheminformatics | Molecular I/O, descriptors, fingerprints, substructure search, reactions, 3D conformers |
| **[materials-project](skills/materials-project/SKILL.md)** | Materials Science | MPRester API, crystal structures, DFT properties, pymatgen, phase diagrams |
| **[nemotron](skills/nemotron/SKILL.md)** | LLM/Generative AI | Hybrid Mamba-Transformer MoE, 1M context, NIM, synthetic data, NeMo training |
| **[omniverse-simready](skills/omniverse-simready/SKILL.md)** | 3D Simulation Assets | OpenUSD, physics, semantic labeling, Isaac Sim, digital twins, robotics |
| **[triton](skills/triton/SKILL.md)** | GPU Programming | Block-based kernels, automatic optimization, tensor cores, @triton.jit, PyTorch integration |
| **[flashinfer](skills/flashinfer/SKILL.md)** | LLM Inference | Attention kernels, paged KV-cache, FP8/FP4 quantization, vLLM/SGLang integration |
| **[physicsnemo](skills/physicsnemo/SKILL.md)** | Physics-Informed ML | FNO, FourCastNet, CorrDiff, GraphCast, distributed training |
| **[bionemo](skills/bionemo/SKILL.md)** | Computational Biology | ESM-2, AlphaFold2, ProteinMPNN, RFdiffusion, NIM microservices |
| **[alchemi-toolkit-ops](skills/alchemi-toolkit-ops/SKILL.md)** | MLOps & Infrastructure | GPU cluster management, Docker, Kubernetes, CI/CD |
| **[autoresearch-setup](skills/autoresearch-setup/SKILL.md)** | Autonomous Research | Experiment loop scaffolding, program.md, single-metric optimization |
| **[prime-verifiers](skills/prime-verifiers/SKILL.md)** | LLM Post-Training | RL environments, rubrics, reward functions, GEPA, PrimeRL |
| **[tinker](skills/tinker/SKILL.md)** | LLM Fine-Tuning | LoRA training API, RL/SL/DPO, remote GPU clusters |
| **[biosimspace](skills/biosimspace/SKILL.md)** | Biomolecular Simulation | Engine-agnostic MD, FEP, metadynamics (AMBER/GROMACS/OpenMM) |

## Installation

### Claude Code (Marketplace)

```bash
claude install-skill agent-skills-for-compute
```

### Cursor

Add as a remote rule pointing to this repository, or clone locally:

```bash
git clone https://github.com/dtunai/agent-skills-for-compute.git .cursor/skills/compute
```

### Manual

Clone this repository and point your AI assistant to the `skills/` directory.

## Structure

```
skills/
  cuda-quantum/
    SKILL.md                     # Quick reference + skill overview
    references/                  # Detailed documentation
  qiskit/
    SKILL.md                     # Qiskit quantum computing
    references/
      runtime-primitives.md      # Sampler, Estimator, PUBs, ISA circuits
      qiskit-metal.md            # Quantum hardware design, EPR analysis
      quantum-algorithms.md      # VQE, QAOA, Grover, Shor, QPE
      transpiler-optimization.md # Passes, basis gates, routing
      error-mitigation.md        # ZNE, readout mitigation, DD
      ecosystem-updates.md       # SDK 2.3, ecosystem packages
  slurm/
    SKILL.md                     # HPC job scheduling
    references/
      job-submission.md          # sbatch, srun, salloc, resource allocation
      gpu-gres-allocation.md     # GPU/GRES allocation, CUDA_VISIBLE_DEVICES
      job-arrays.md              # Array jobs, concurrency, dependencies
      monitoring-accounting.md   # squeue, sacct, sstat, sinfo, QOS
  dask/
    SKILL.md                     # Parallel computing framework
    references/
      dataframe-array-bag.md     # Core collections (DataFrame, Array, Bag, Delayed)
      distributed-scheduler.md   # Cluster setup, client, task scheduling
      gpu-acceleration.md        # cuDF, CuPy, LocalCUDACluster, RAPIDS
      deployment.md              # Local, cloud, HPC (Jobqueue), Kubernetes
  ray/
    SKILL.md                     # Distributed AI compute engine
    references/
      core-tasks-actors.md       # Ray Core primitives, tasks, actors, objects
      data-processing.md         # Ray Data transformations, batch operations
      train-serve.md             # Ray Train distributed training, Ray Serve model serving
      tune-rllib.md              # Ray Tune hyperparameter tuning, Ray RLlib RL
      cluster-deployment.md      # Cluster setup, cloud, Kubernetes, autoscaling
  mpi/
    SKILL.md                     # Message passing interface
    references/
      mpi-standard.md            # MPI standard, concepts, data types, communicators
      point-to-point.md          # Send/Recv operations, tags, non-blocking
      collective-operations.md   # Broadcast, scatter, gather, reduce, allreduce
      parallel-io.md             # MPI-IO, collective I/O, file views
      execution-deployment.md    # mpirun, hostfiles, SLURM integration, process mapping
  polars/
    SKILL.md                     # Polars DataFrames with GPU acceleration
    references/
      fundamentals.md            # DataFrame, LazyFrame, Series, data types
      lazy-query-optimization.md # Query plans, predicate/projection pushdown
      gpu-engine.md              # cuDF integration, performance tuning
      io-operations.md           # CSV, Parquet, JSON, cloud storage
      expressions-api.md         # Expression syntax, aggregations, window functions
  rdkit/
    SKILL.md                     # RDKit cheminformatics toolkit
    references/
      molecular-io.md            # SMILES, SDF, Mol files, suppliers, writers
      descriptors-fingerprints.md # Molecular properties, fingerprints, similarity
      substructure-reactions.md  # Pattern matching, SMARTS, chemical reactions
      3d-conformers.md           # Coordinate generation, conformer optimization
      stereochemistry.md         # Chirality, CIP rules, stereo groups, atropisomers
      sanitization-aromaticity.md # Molecular sanitization, aromaticity models
      scaffold-analysis.md       # Murcko scaffolds, generic scaffolds, splits
      advanced-features.md       # Drawing, fragmentation, serialization, MCS
  materials-project/
    SKILL.md                     # Materials Project API client
    references/
      api-basics.md              # Installation, authentication, MPRester config
      querying-data.md           # Search methods, filtering, pagination
      structures-thermo.md       # Structure retrieval, phase diagrams, entries
      electronic-phonon.md       # Band structures, DOS, phonons
      pymatgen-integration.md    # Structure analysis, symmetry, comparisons
      advanced-usage.md          # Routes, batch operations, optimization
  nemotron/
    SKILL.md                     # NVIDIA Nemotron LLM family
    references/
      model-family.md            # Nano, Super, Ultra, VL, RAG, Safety models
      nim-deployment.md          # NIM containers, API, inference, scaling
      synthetic-data.md          # NemotronGenerator, pipelines, customization
      nemo-training.md           # Pretraining recipes, distributed training
      architecture.md            # Hybrid Mamba-Transformer MoE, techniques
      api-reference.md           # Endpoints, parameters, examples
  omniverse-simready/
    SKILL.md                     # NVIDIA Omniverse SimReady assets
    references/
      specification.md           # SimReady standard, requirements, scope
      asset-creation.md          # Modeling, UVs, materials, workflow
      physics-setup.md           # USDPhysics, colliders, mass, materials
      semantic-labeling.md       # WikiData, taxonomy, metadata
      usd-structure.md           # OpenUSD, composition, schemas
      isaac-sim-integration.md   # Loading assets, synthetic data, robotics
  triton/
    SKILL.md                     # Triton GPU programming language
    references/
      programming-guide.md       # Block-based model, SPMD, optimizations
      tutorials.md               # Vector add, softmax, matmul examples
      advanced-tutorials.md      # Flash Attention, dropout, layer norm, grouped GEMM, persistent kernels, FP8
      language-api.md            # tl.load/store/dot, math ops, reductions
      semantics.md               # NumPy compatibility, type promotion, broadcasting, integer division
      performance-tuning.md      # Auto-tuning, num_warps, num_stages
      matrix-operations.md       # Matmul, tiling, tensor cores, batched ops
      integration.md             # PyTorch, autograd, deployment, profiling
      debugging.md               # TRITON_INTERPRET, device_print, device_assert, compute-sanitizer
      backends.md                # CUDA, HIP, CPU, compilation pipeline, installation
  flashinfer/
    SKILL.md                     # FlashInfer LLM inference kernels
    references/
      attention-kernels.md       # Prefill, decode, append, KV-cache formats (paged/ragged/padded)
      advanced-techniques.md     # Cascade inference, MLA, XQA, POD-Attention, JIT compilation
      gemm-moe.md                # FP8/FP4 GEMM, MoE fusion, quantization, segment GEMM
      sampling.md                # Top-k/p/min-p, speculative decoding, sorting-free algorithms
      integration.md             # vLLM, SGLang, MLC-Engine, PyTorch, deployment
      performance.md             # Benchmarks, roofline analysis, optimization strategies
  physicsnemo/
    SKILL.md
    references/                  # Physics-informed ML patterns
  bionemo/
    SKILL.md
    references/                  # Computational biology workflows
  alchemi-toolkit-ops/
    SKILL.md
    references/                  # GPU infrastructure and MLOps
  autoresearch-setup/
    SKILL.md                     # Autonomous experiment loop scaffolding
  prime-verifiers/
    SKILL.md                     # Prime Intellect verifiers & PrimeRL
    references/
  tinker/
    SKILL.md                     # Thinking Machines Lab training API
    references/
  biosimspace/
    SKILL.md                     # Engine-agnostic biomolecular simulation
    references/
```

## Contributing

Follow the [agentskills.io specification](https://agentskills.io/specification). Each skill needs:

1. A directory under `skills/<skill-name>/`
2. A `SKILL.md` with YAML frontmatter (name, description)
3. Reference files in `references/` for detailed content
4. Keep SKILL.md under 500 lines

Run `/validate-skills` to check compliance.

## License

MIT
