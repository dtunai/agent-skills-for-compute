# Slurm GPU and GRES Allocation Reference

Source: [Slurm GRES Documentation](https://slurm.schedmd.com/gres.html)

## Overview

Generic Resources (GRES) in Slurm manage specialized hardware like GPUs. **Jobs must explicitly request GPUs**—no automatic allocation occurs.

## GPU Request Methods

### Command-Line Options

```bash
# Total GPUs for entire job
sbatch --gpus=4 script.sh
srun --gpus=2 command

# GPUs per node
sbatch --gpus-per-node=2 script.sh

# GPUs per socket (requires socket count)
sbatch --gpus-per-socket=1 --sockets-per-node=2 script.sh

# GPUs per task (requires task count)
sbatch --gpus-per-task=1 -n 4 script.sh

# GRES format (most flexible)
sbatch --gres=gpu:2 script.sh
sbatch --gres=gpu:tesla:2 script.sh
sbatch --gres=gpu:a100:4 script.sh
```

### GRES Format Syntax

```
--gres=<name>[:<type>][:<count>]
```

**Examples:**
```bash
--gres=gpu              # 1 GPU (any type)
--gres=gpu:2            # 2 GPUs (any type)
--gres=gpu:tesla        # 1 Tesla GPU
--gres=gpu:tesla:2      # 2 Tesla GPUs
--gres=gpu:a100:4       # 4 A100 GPUs
```

### Multiple GRES Types

```bash
# GPUs and MPS
sbatch --gres=gpu:2,mps:100 script.sh

# Multiple resource types
sbatch --gres=gpu:1,bandwidth:1000 script.sh
```

## CUDA_VISIBLE_DEVICES

Slurm automatically sets `CUDA_VISIBLE_DEVICES` for each job step to control which GPUs are available.

### Automatic Configuration

```bash
#!/bin/bash
#SBATCH --gpus=2

# Slurm sets CUDA_VISIBLE_DEVICES automatically
echo "GPUs: $CUDA_VISIBLE_DEVICES"
# Output example: 0,1

nvidia-smi
python train.py  # Sees only allocated GPUs
```

### Device Ordering

For alignment with CUDA numbering, set:

```bash
export CUDA_DEVICE_ORDER=PCI_BUS_ID
```

**Why:** NVIDIA NVML numbers GPUs by PCI bus ID. This ensures consistency between Slurm allocation and CUDA visibility.

## GPU Types and Selection

### Listing Available GPU Types

```bash
# Show GPU resources
sinfo --Format=NodeList,Gres:30,GresUsed:30

# Example output:
# NODELIST    GRES                          GRES_USED
# node01      gpu:tesla:4                   gpu:tesla:2
# node02      gpu:a100:8                    gpu:a100:0
```

### Requesting Specific Types

```bash
# Tesla GPUs only
sbatch --gres=gpu:tesla:2 script.sh

# A100 GPUs only
sbatch --gres=gpu:a100:4 script.sh

# V100 GPUs
sbatch --gres=gpu:v100:2 script.sh

# Multiple acceptable types
sbatch --gres=gpu:a100:1 --gres=gpu:v100:1 script.sh
```

## GPU Binding

Control how GPUs are assigned to tasks.

### Binding Strategies

```bash
# Closest GPU to task (NUMA-aware)
sbatch --gpus-per-task=1 --gpu-bind=closest script.sh

# Single GPU binding
sbatch --gpus-per-task=1 --gpu-bind=single:1 script.sh

# Map GPUs to tasks
sbatch --gpus-per-task=2 --gpu-bind=map_gpu:0,1,2,3 script.sh

# Mask GPUs
sbatch --gpu-bind=mask_gpu:0x3 script.sh  # GPUs 0,1
```

**Binding Types:**
- `closest`: Allocate GPU closest to task (NUMA)
- `single:N`: Allocate N GPUs to each task
- `map_gpu`: Explicit GPU-to-task mapping
- `mask_gpu`: Bitmask for GPU selection

## MPS (Multi-Process Service)

Enables GPU sharing among multiple jobs using percentage-based allocation.

### MPS Requests

```bash
# Request 50% of GPU via MPS
sbatch --gres=mps:50 script.sh

# 100 MPS units (percentage)
sbatch --gres=mps:100 script.sh
```

### MPS Environment Variables

Jobs receive:
- `CUDA_VISIBLE_DEVICES`: Set to 0 for single GPU
- `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE`: Job's resource percentage
- `CUDA_DEVICE_ORDER`: Device ordering

**Important Constraint:** Same GPU cannot be allocated as both GPU GRES and MPS GRES simultaneously.

### MPS vs Full GPU

| Allocation | Use Case | Pros | Cons |
|------------|----------|------|------|
| Full GPU (`--gres=gpu`) | Large workloads | Full resources | Potential underutilization |
| MPS (`--gres=mps`) | Small workloads | Share GPU, higher utilization | Limited parallelism |

## GPU Frequency Control

```bash
# Request specific GPU frequency
sbatch --gpu-freq=high script.sh
sbatch --gpu-freq=medium script.sh
sbatch --gpu-freq=low script.sh

# Explicit frequency in kHz
sbatch --gpu-freq=1200000 script.sh

# Memory frequency
sbatch --gpu-freq=memory_high script.sh
```

**Options:**
- `low`, `medium`, `high`, `highm1`
- Specific frequency in kHz
- `memory_low`, `memory_medium`, `memory_high`

## Job Step GPU Allocation

Job steps can request different GRES than parent job.

### Pattern

```bash
#!/bin/bash
#SBATCH --gpus=4

# Step 1: Use 2 GPUs
srun --gpus=2 python train_model1.py &

# Step 2: Use 1 GPU
srun --gpus=1 python train_model2.py &

# Step 3: Use remaining GPU
srun --gpus=1 python train_model3.py &

wait
```

**Default:** Job step inherits all GPUs from parent job unless explicitly specified.

## GPU Memory Allocation

```bash
# Memory per GPU
sbatch --mem-per-gpu=16G --gpus=2 script.sh

# Total memory (all GPUs)
sbatch --mem=32G --gpus=2 script.sh
```

## Configuration Files

### slurm.conf

Defines available GRES types:

```
GresTypes=gpu,mps,bandwidth
NodeName=node[01-04] Gres=gpu:tesla:4,mps:400
NodeName=node[05-08] Gres=gpu:a100:8,mps:800
```

### gres.conf

Details GPU-specific configuration:

```
# AutoDetect with NVML
AutoDetect=nvml
Name=gpu Type=a100 File=/dev/nvidia0 Cores=0-15
Name=gpu Type=a100 File=/dev/nvidia1 Cores=16-31

# MPS configuration
Name=mps Count=100
```

**AutoDetect Options:**
- `nvml`: NVIDIA Management Library
- `nvidia`: NVIDIA driver
- `rsmi`: AMD ROCm SMI
- `nrt`: Intel NRT
- `oneapi`: Intel OneAPI

### File Parameters

**Critical:** File parameters in `gres.conf` must be in increasing numeric order.

```
# Correct
Name=gpu File=/dev/nvidia0
Name=gpu File=/dev/nvidia1
Name=gpu File=/dev/nvidia2

# Incorrect
Name=gpu File=/dev/nvidia2
Name=gpu File=/dev/nvidia0
Name=gpu File=/dev/nvidia1
```

## GPU Accounting

### Tracking GPU Usage

With `AccountingStorageTRES=gres/gpu`:

```bash
# View GPU usage
sacct -j JOBID --format=JobID,AllocGRES,TRESUsageInAve

# GPU memory usage
sacct -j JOBID --format=JobID,TRESUsageInAve | grep gres/gpumem

# GPU utilization
sacct -j JOBID --format=JobID,TRESUsageInAve | grep gres/gpuutil
```

**Metrics Available:**
- `gres/gpu`: Number of GPUs allocated
- `gres/gpumem`: GPU memory used
- `gres/gpuutil`: GPU utilization percentage

**Support:**
- NVIDIA GPUs: `AutoDetect=nvml` (memory and utilization)
- AMD GPUs: `AutoDetect=rsmi` (memory and utilization)
- MIG devices: Memory only (no utilization)

## GPU Constraints

### Node Selection with GPU Features

```bash
# Nodes with Tesla GPUs
sbatch -C "gpu&tesla" script.sh

# Nodes with A100 or V100
sbatch -C "gpu&(a100|v100)" script.sh

# Specific GPU count
sbatch -C "gpu*4" script.sh  # Nodes with 4+ GPUs
```

## Best Practices

1. **Always request GPUs explicitly**: No default allocation
2. **Use --gpus-per-task for scalability**: Easier to scale task count
3. **Set CUDA_DEVICE_ORDER=PCI_BUS_ID**: Consistent numbering
4. **Check CUDA_VISIBLE_DEVICES**: Verify allocation
5. **Use MPS for small workloads**: Improve utilization
6. **Request specific types when needed**: Ensure hardware compatibility
7. **Monitor GPU utilization**: `nvidia-smi` or `sacct`
8. **Use job arrays for multi-GPU sweeps**: Efficient resource use
9. **Bind GPUs to tasks**: Improve NUMA performance
10. **Check available types**: `sinfo` before submission

## Common Pitfalls

- **Forgetting --gpus flag**: No GPU allocated despite CUDA code
- **Assuming CUDA_VISIBLE_DEVICES is set**: Only set when GPUs requested
- **Requesting more GPUs than available**: Job never schedules
- **Not checking GPU type availability**: Wrong hardware
- **Using MPS with incompatible code**: Some apps don't support MPS
- **Incorrect device file order in gres.conf**: GPU mapping errors
- **Not monitoring GPU usage**: Wasting resources
- **Over-requesting GPU memory**: Delays scheduling
- **Mixing GPU and MPS on same device**: Configuration error
- **Not setting PCI_BUS_ID**: Mismatched device IDs

## Example Workflows

### Single GPU Job

```bash
#!/bin/bash
#SBATCH --gpus=1
#SBATCH --mem-per-gpu=16G
#SBATCH --cpus-per-gpu=4

export CUDA_DEVICE_ORDER=PCI_BUS_ID

nvidia-smi
python train.py
```

### Multi-GPU Data Parallel

```bash
#!/bin/bash
#SBATCH --gpus-per-node=4
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=32

# 8 total GPUs across 2 nodes
python -m torch.distributed.launch --nproc_per_node=4 train_ddp.py
```

### GPU Array Job

```bash
#!/bin/bash
#SBATCH --array=0-99%10
#SBATCH --gpus-per-task=1
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-gpu=8G

# Each task gets 1 GPU
python experiment.py --task_id $SLURM_ARRAY_TASK_ID
```

### MPS Sharing Example

```bash
#!/bin/bash
#SBATCH --gres=mps:25
#SBATCH --mem=4G

# Uses 25% of GPU
python small_workload.py
```
