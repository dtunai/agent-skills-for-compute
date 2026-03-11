---
name: slurm
description: Provides Slurm Workload Manager commands for HPC cluster job scheduling, GPU allocation with GRES, job arrays, MPI integration, resource management, QOS configuration, and accounting. Applies to tasks involving batch job submission (sbatch), interactive jobs (srun, salloc), job monitoring (squeue, sacct), GPU/CUDA workloads, parallel computing, cluster resource allocation, or HPC workflow management.
license: MIT
metadata:
  author: Agent Cluster
  tags: slurm, hpc, job-scheduler, gpu-allocation, gres, mpi, batch-jobs, sacct, squeue, sbatch, srun, parallel-computing, cluster-management
---

# Slurm Workload Manager

## Overview

Open-source cluster management and job scheduling system for Linux clusters. Slurm allocates compute resources, provides a framework for executing parallel jobs, and manages resource contention through job queues.

## Quick Pattern

**Incorrect — missing resource specifications:**

```bash
# No resource specification, will fail
sbatch my_script.sh

# Implicit assumptions about GPUs
srun python train.py
```

**Correct — explicit resource requests:**

```bash
# Batch job with resources
sbatch --nodes=2 --ntasks-per-node=4 --gpus=2 --time=02:00:00 my_script.sh

# Interactive with GPU
srun --gpus=1 --mem=16G --pty bash

# Job array with concurrency limit
sbatch --array=0-99%10 --gpus-per-task=1 --mem=8G array_script.sh
```

## Quick Command

```bash
# Job submission
sbatch script.sh                    # Submit batch job
srun command                        # Interactive execution
salloc -N 2 bash                    # Allocate resources, spawn shell

# Job monitoring
squeue                              # View job queue
squeue -u $USER                     # Your jobs only
sinfo                               # Partition/node status
sacct -j JOBID                      # Job accounting info
sstat -j JOBID                      # Running job stats

# Job control
scancel JOBID                       # Cancel job
scancel -u $USER                    # Cancel all your jobs
scontrol hold JOBID                 # Hold job
scontrol release JOBID              # Release held job
```

## Quick Reference

### Core Commands

| Command | Purpose |
|---------|---------|
| `sbatch` | Submit batch script for later execution |
| `srun` | Submit job for immediate execution or initiate job step |
| `salloc` | Allocate resources and spawn shell |
| `squeue` | View job queue status |
| `scancel` | Cancel jobs or job steps |
| `sinfo` | Display partition and node info |
| `scontrol` | View/modify Slurm state (admin tool) |
| `sacct` | Job accounting information from database |
| `sstat` | Resource utilization for running jobs |

### Resource Allocation

| Option | Purpose |
|--------|---------|
| `-N, --nodes` | Number of nodes |
| `-n, --ntasks` | Number of tasks |
| `-c, --cpus-per-task` | CPUs per task |
| `--mem` | Memory per node (MB) |
| `--mem-per-cpu` | Memory per CPU |
| `-t, --time` | Time limit (hh:mm:ss) |
| `-p, --partition` | Partition name |
| `-G, --gpus` | Total GPUs required |
| `--gpus-per-node` | GPUs per node |
| `--gres` | Generic resources (e.g., gpu:2) |

### GPU Allocation

| Option | Purpose |
|--------|---------|
| `--gpus=N` | Total GPUs for job |
| `--gpus-per-node=N` | GPUs per node |
| `--gpus-per-task=N` | GPUs per task |
| `--gres=gpu:type:N` | GPUs with specific type |
| `--mem-per-gpu=SIZE` | Memory per GPU |
| `--gpu-bind` | GPU binding strategy |

### Job Arrays

| Syntax | Meaning |
|--------|---------|
| `--array=0-15` | Tasks 0 through 15 |
| `--array=0,5,10` | Specific task IDs |
| `--array=0-15:4` | Step of 4 (0,4,8,12) |
| `--array=0-99%10` | Max 10 concurrent tasks |

**Environment Variables:**
- `$SLURM_ARRAY_JOB_ID`: Array job ID
- `$SLURM_ARRAY_TASK_ID`: Current task ID
- `$SLURM_ARRAY_TASK_COUNT`: Total tasks

### Output/Error Files

| Format | Meaning |
|--------|---------|
| `%j` | Job ID |
| `%A` | Array job ID |
| `%a` | Array task ID |
| `%N` | Node name |
| `%u` | Username |

## Core Patterns

### 1. Basic Batch Job

```bash
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --output=output_%j.txt
#SBATCH --error=error_%j.txt
#SBATCH --nodes=1
#SBATCH --ntasks=4
#SBATCH --cpus-per-task=2
#SBATCH --mem=16G
#SBATCH --time=01:00:00
#SBATCH --partition=compute

# Your commands
srun hostname
srun my_application
```

### 2. GPU Job

```bash
#!/bin/bash
#SBATCH --job-name=gpu_job
#SBATCH --gpus=2
#SBATCH --cpus-per-gpu=4
#SBATCH --mem-per-gpu=16G
#SBATCH --time=04:00:00

# CUDA environment set automatically
nvidia-smi
python train_model.py
```

### 3. Job Array

```bash
#!/bin/bash
#SBATCH --job-name=array_job
#SBATCH --output=logs/job_%A_%a.out
#SBATCH --array=0-99%20
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G

# Use $SLURM_ARRAY_TASK_ID
python process_data.py --index $SLURM_ARRAY_TASK_ID
```

### 4. MPI Job

```bash
#!/bin/bash
#SBATCH --job-name=mpi_job
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --time=02:00:00

# Method 1: Direct launch (PMI2/PMIx)
srun ./my_mpi_app

# Method 2: mpirun with Slurm
mpirun -n 32 ./my_mpi_app
```

### 5. Multi-Step Job

```bash
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --ntasks=8
#SBATCH --gpus=4

# Step 1: Preprocessing
srun --ntasks=4 --gpus=0 preprocess.sh

# Step 2: Training (use GPUs)
srun --ntasks=4 --gpus=4 python train.py

# Step 3: Evaluation
srun --ntasks=2 --gpus=2 python eval.py
```

### 6. Interactive Job

```bash
# Allocate resources and get shell
salloc --nodes=1 --ntasks=4 --gpus=1 --time=02:00:00

# Inside allocation:
srun python interactive_work.py
exit  # Release resources

# Or one-liner
srun --pty --gpus=1 --mem=16G bash
```

### 7. Job Dependencies

```bash
# Submit first job
JOBID1=$(sbatch --parsable job1.sh)

# Submit dependent job (runs after job1 succeeds)
sbatch --dependency=afterok:$JOBID1 job2.sh

# Multiple dependencies
sbatch --dependency=afterok:$JOBID1:$JOBID2 job3.sh

# Array dependency (corresponding tasks)
sbatch --dependency=aftercorr:$JOBID1 --array=0-9 job2.sh
```

### 8. Heterogeneous Jobs

```bash
# Different resource requirements per component
sbatch -n4 --cpus-per-task=2 : -n8 --cpus-per-task=1 hetero_script.sh
```

## GRES and GPU Patterns

### Request Specific GPU Type

```bash
sbatch --gres=gpu:tesla:2 script.sh
sbatch --gres=gpu:a100:4 script.sh
```

### GPU Binding

```bash
# Closest GPU to task
sbatch --gpus-per-task=1 --gpu-bind=closest script.sh

# Single GPU binding
sbatch --gpus-per-task=1 --gpu-bind=single:1 script.sh
```

### MPS (Multi-Process Service)

```bash
# Request MPS percentage instead of full GPU
sbatch --gres=mps:50 script.sh  # 50% of GPU
```

### CUDA_VISIBLE_DEVICES

Automatically set by Slurm for GPU jobs.

```bash
#!/bin/bash
#SBATCH --gpus=2

echo "GPUs available: $CUDA_VISIBLE_DEVICES"
# Example output: 0,1
```

## Monitoring and Accounting

### Queue Status

```bash
# All jobs
squeue

# Your jobs
squeue -u $USER

# Specific job
squeue -j JOBID

# Detailed format
squeue -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"

# Array job tasks
squeue --array -j JOBID
```

### Job Accounting

```bash
# Job summary
sacct -j JOBID

# Detailed format
sacct -j JOBID --format=JobID,JobName,Partition,Account,AllocCPUS,State,ExitCode

# Time range
sacct -S 2026-01-01 -E 2026-01-31

# GPU usage
sacct -j JOBID --format=JobID,AllocGRES,TRESUsageInAve
```

### Running Job Stats

```bash
# Resource utilization
sstat -j JOBID

# Specific fields
sstat -j JOBID --format=JobID,AveCPU,AveRSS,AveVMSize
```

### Node Information

```bash
# Partition status
sinfo

# Available nodes
sinfo -N -l

# GPU nodes
sinfo --Format=NodeList,Gres:30,GresUsed:30

# Specific partition
sinfo -p gpu
```

## Job Control

### Cancel Jobs

```bash
scancel JOBID                    # Single job
scancel JOBID_[1-5]              # Array tasks 1-5
scancel JOBID_[1,3,5]            # Specific array tasks
scancel -u $USER                 # All your jobs
scancel -n job_name              # By name
scancel -p partition_name        # All in partition
```

### Modify Jobs

```bash
# Hold/release
scontrol hold JOBID
scontrol release JOBID

# Update time limit
scontrol update JobId=JOBID TimeLimit=05:00:00

# Update QOS
scontrol update JobId=JOBID QOS=high

# Show job details
scontrol show job JOBID
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `SLURM_JOB_ID` | Job ID |
| `SLURM_JOB_NAME` | Job name |
| `SLURM_SUBMIT_DIR` | Submission directory |
| `SLURM_JOB_NODELIST` | Nodes allocated |
| `SLURM_NTASKS` | Number of tasks |
| `SLURM_CPUS_PER_TASK` | CPUs per task |
| `SLURM_LOCALID` | Task rank on node |
| `SLURM_PROCID` | Global task rank |
| `CUDA_VISIBLE_DEVICES` | GPU devices (auto-set) |

## Best Practices

1. **Always specify time limits**: Prevents jobs running indefinitely
2. **Use job arrays for parameter sweeps**: More efficient than individual jobs
3. **Limit concurrent array tasks**: Use `%` to avoid overwhelming scheduler
4. **Request only needed resources**: Improves scheduling and fairness
5. **Use dependencies for workflows**: Automate multi-stage pipelines
6. **Check CUDA_VISIBLE_DEVICES**: Verify GPU allocation
7. **Monitor job efficiency**: Use sacct to optimize resource requests
8. **Use specific partitions**: Target appropriate hardware
9. **Set memory limits explicitly**: Avoid node oversubscription
10. **Consolidate into job steps**: Better than many small jobs

## Common Pitfalls

- **Not specifying time limit**: May use partition default or fail
- **Requesting more GPUs than available**: Job never schedules
- **Forgetting --gpus flag**: No GPUs allocated despite CUDA code
- **Array without concurrency limit**: Overwhelming the scheduler
- **Not using $SLURM_ARRAY_TASK_ID**: All array tasks do same work
- **Hardcoding node names**: Reduces portability
- **Excessive memory requests**: Delays scheduling
- **Not checking job output**: Missing errors until later
- **Using srun in sbatch without resources**: Inherits full job allocation
- **Mixing salloc and sbatch**: Confusing resource accounting

## Version Notes

- **Slurm 23.x+**: Improved GPU scheduling, MPS support
- **Slurm 22.x**: Enhanced job arrays (up to 4M tasks)
- **Slurm 21.x**: Heterogeneous job support
- Check `sinfo --version` for cluster version

## References

Official Slurm documentation at [slurm.schedmd.com](https://slurm.schedmd.com/):
- Quick Start: [quickstart.html](https://slurm.schedmd.com/quickstart.html)
- sbatch: [sbatch.html](https://slurm.schedmd.com/sbatch.html)
- GRES/GPU: [gres.html](https://slurm.schedmd.com/gres.html)
- Job Arrays: [job_array.html](https://slurm.schedmd.com/job_array.html)
- QOS: [qos.html](https://slurm.schedmd.com/qos.html)
