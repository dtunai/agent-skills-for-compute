# MPI Execution and Deployment Reference

Sources:
- [Open MPI Launching Apps](https://docs.open-mpi.org/en/v5.0.x/launching-apps/index.html)
- [MPICH User Guide](https://www.mpich.org/documentation/guides/)
- [Open MPI mpirun Man Page](https://docs.open-mpi.org/en/v5.0.x/man-openmpi/man1/mpirun.1.html)

## Overview

According to Open MPI documentation, **mpirun(1)** and **mpiexec(1)** are "exactly identical" and function as symbolic links to the common backend launcher. MPI programs can be launched in scheduled environments (Slurm, PBS, LSF) or non-scheduled environments (SSH, local).

## Basic Execution

### Simple Launch

```bash
# Basic execution
mpirun -n 4 python script.py
mpiexec -np 8 ./mpi_program

# Equivalent
mpirun -np 4 python script.py
mpiexec -n 4 python script.py
```

### Number of Processes

```bash
# Specify process count
mpirun -n 16 python script.py
mpirun --np 16 python script.py

# Use all available cores
mpirun -n $(nproc) python script.py
mpirun --use-hwthread-cpus python script.py
```

## Process Mapping

### Mapping by Resource

```bash
# Map by node (one process per node)
mpirun -np 8 --map-by node python script.py

# Map by socket (one per CPU socket)
mpirun -np 8 --map-by socket python script.py

# Map by core (one per CPU core)
mpirun -np 16 --map-by core python script.py

# Map by hardware thread
mpirun -np 32 --map-by hwthread python script.py
```

### Process Binding

```bash
# Bind to core
mpirun -np 8 --bind-to core python script.py

# Bind to socket
mpirun -np 4 --bind-to socket python script.py

# Bind to NUMA node
mpirun -np 2 --bind-to numa python script.py

# No binding
mpirun -np 8 --bind-to none python script.py
```

### Detailed Mapping

```bash
# Rank distribution
# --map-by <object>:PE=<n>
# PE = processing elements per object

# 4 cores per rank
mpirun -np 4 --map-by core:PE=4 python script.py

# 2 ranks per node, 8 cores each
mpirun -np 8 --map-by node:PE=8 python script.py
```

## Host Files

### Creating Host File

```bash
# hostfile.txt
node01 slots=4
node02 slots=8
node03 slots=4
node04 slots=8
```

### Using Host Files

```bash
# Use hostfile
mpirun -np 16 --hostfile hosts.txt python script.py
mpirun -np 16 -machinefile hosts.txt python script.py

# Explicit host list
mpirun -np 8 --host node01,node02,node03,node04 python script.py
```

### Slots and Oversubscribe

```bash
# Allow oversubscription
mpirun -np 32 --hostfile hosts.txt --oversubscribe python script.py

# Limit per-node processes
mpirun -np 16 --hostfile hosts.txt --map-by ppr:2:node python script.py
```

## SLURM Integration

### Basic SLURM Execution

```bash
#!/bin/bash
#SBATCH -N 4
#SBATCH --ntasks-per-node=8
#SBATCH --time=01:00:00
#SBATCH --job-name=mpi_job

# mpirun auto-detects SLURM
mpirun python mpi_script.py

# Or use srun directly
srun python mpi_script.py
```

### srun with MPI

```bash
# srun is SLURM-aware mpirun
srun -n 32 python script.py

# With specific options
srun -N 4 --ntasks-per-node=8 python script.py

# GPU allocation
srun -n 4 --gpus-per-task=1 python gpu_mpi.py
```

### PMI/PMIx

```bash
# Specify PMI version
mpirun --mca pml ob1 --mca btl ^openib python script.py

# Use PMIx
export SLURM_MPI_TYPE=pmix
srun python script.py
```

## PBS/Torque Integration

### PBS Script

```bash
#!/bin/bash
#PBS -l nodes=4:ppn=8
#PBS -l walltime=01:00:00
#PBS -N mpi_job

cd $PBS_O_WORKDIR

# Read node list from PBS
mpirun -np 32 -machinefile $PBS_NODEFILE python script.py
```

## Environment Variables

### MPI Environment

```bash
# Set MPI implementation variables
export OMPI_MCA_btl=tcp,self  # Open MPI: Use TCP
export MPICH_ASYNC_PROGRESS=1  # MPICH: Async progress

# Process binding
export OMP_NUM_THREADS=1  # Disable OpenMP threading
export MKL_NUM_THREADS=1  # Disable MKL threading
```

### Passing Environment

```bash
# Export all environment
mpirun -x PATH -x LD_LIBRARY_PATH python script.py

# Export specific variables
mpirun -x CUSTOM_VAR=value python script.py

# From file
mpirun --envfile env.txt python script.py
```

## Working Directory

### Setting Directory

```bash
# Run in specific directory
mpirun -np 4 --wdir /scratch/project python script.py

# Different per rank
mpirun -np 4 --wdir /scratch/rank_\${OMPI_COMM_WORLD_RANK} python script.py
```

## Output Control

### Standard Output

```bash
# Prefix output with rank
mpirun -np 4 --tag-output python script.py

# Timestamp output
mpirun -np 4 --timestamp-output python script.py

# Redirect stdout/stderr
mpirun -np 4 python script.py > output.log 2> error.log

# Per-rank output files
mpirun -np 4 --output-filename out python script.py
# Creates: out.1-0, out.1-1, out.1-2, out.1-3
```

## Display and Debugging

### Display Information

```bash
# Show process map
mpirun -np 4 --display-map python script.py

# Show bindings
mpirun -np 4 --report-bindings python script.py

# Verbose output
mpirun -np 4 --verbose python script.py
mpirun -np 4 -v python script.py

# Display MCA parameters
mpirun --display-mca-params python script.py
```

### Debugging

```bash
# Attach debugger
mpirun -np 4 --debug python script.py

# Wait for debugger attach
mpirun -np 4 --debug-daemons python script.py

# Run under valgrind
mpirun -np 2 valgrind --leak-check=full python script.py

# Run under gdb
mpirun -np 1 xterm -e gdb python script.py
```

## Network Configuration

### Network Selection

```bash
# Specify network (Open MPI)
mpirun -np 4 --mca btl tcp,self python script.py
mpirun -np 4 --mca btl ^openib python script.py  # Exclude InfiniBand

# InfiniBand
mpirun -np 4 --mca btl openib,self python script.py

# Shared memory
mpirun -np 4 --mca btl sm,self python script.py
```

### TCP Configuration

```bash
# Specify interface
mpirun -np 4 --mca btl_tcp_if_include eth0 python script.py

# Exclude interface
mpirun -np 4 --mca btl_tcp_if_exclude lo python script.py

# Port range
mpirun -np 4 --mca oob_tcp_dynamic_ports 10000-10100 python script.py
```

## Hybrid MPI + OpenMP

### Hybrid Execution

```bash
# 4 MPI ranks, 8 OpenMP threads each
export OMP_NUM_THREADS=8
mpirun -np 4 --bind-to socket --map-by socket python script.py

# SLURM hybrid
#!/bin/bash
#SBATCH -N 2
#SBATCH --ntasks-per-node=4
#SBATCH --cpus-per-task=8

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
srun python hybrid_script.py
```

### Process/Thread Affinity

```bash
# Bind MPI ranks to sockets, OpenMP threads to cores
export OMP_PROC_BIND=close
export OMP_PLACES=cores

mpirun -np 4 --bind-to socket --map-by socket:PE=8 python script.py
```

## GPU-Aware MPI

### GPU Allocation

```bash
# SLURM with GPUs
#!/bin/bash
#SBATCH -N 4
#SBATCH --ntasks-per-node=4
#SBATCH --gpus-per-task=1

# Each MPI rank gets 1 GPU
srun python gpu_mpi.py
```

### CUDA_VISIBLE_DEVICES

```bash
# Manually set GPUs per rank
mpirun -np 4 bash -c 'export CUDA_VISIBLE_DEVICES=$OMPI_COMM_WORLD_LOCAL_RANK; python gpu_mpi.py'

# Or in Python script
import os
from mpi4py import MPI

comm = MPI.COMM_WORLD
rank = comm.Get_rank()

# Set GPU based on local rank
local_rank = rank % 4  # Assuming 4 GPUs per node
os.environ['CUDA_VISIBLE_DEVICES'] = str(local_rank)
```

## Fault Tolerance

### Checkpoint/Restart

```bash
# DMTCP checkpointing
dmtcp_launch mpirun -np 4 python script.py

# Restart from checkpoint
dmtcp_restart ckpt_*.dmtcp
```

### Process Monitoring

```bash
# Enable fault tolerance (Open MPI)
mpirun -np 4 --mca orte_enable_recovery 1 python script.py

# Timeout
mpirun -np 4 --timeout 3600 python script.py  # 1 hour timeout
```

## Performance Tuning

### Eager Limits

```bash
# Increase eager limit (Open MPI)
mpirun -np 4 --mca btl_tcp_eager_limit 65536 python script.py

# RDMA threshold
mpirun -np 4 --mca btl_openib_eager_limit 32768 python script.py
```

### Progress Threads

```bash
# Enable async progress (MPICH)
export MPICH_ASYNC_PROGRESS=1

# Open MPI progress threads
mpirun -np 4 --mca opal_progress_threads 1 python script.py
```

## Troubleshooting

### Connection Issues

```bash
# Test connectivity
mpirun -np 4 hostname

# Verbose network info
mpirun -np 4 --mca btl_base_verbose 30 python script.py

# Check paths
which mpirun
mpirun --version
```

### Common Errors

```bash
# "ssh: Could not resolve hostname"
# Fix: Check hostfile for typos

# "error while loading shared libraries"
# Fix: Set LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/path/to/mpi/lib:$LD_LIBRARY_PATH

# "btl_tcp_endpoint.c error"
# Fix: Use different network
mpirun --mca btl ^tcp python script.py
```

### Logging

```bash
# Enable logging
export MPICH_DEBUG=1  # MPICH
export OMPI_MCA_mpi_show_mca_params=1  # Open MPI

# Save to file
mpirun -np 4 python script.py 2> debug.log
```

## Container Execution

### Docker

```bash
# Run MPI in Docker
docker run --rm -v $(pwd):/work -w /work \
    mpi-image mpirun -np 4 python script.py
```

### Singularity

```bash
# MPI with Singularity
mpirun -np 4 singularity exec container.sif python script.py

# Host MPI + container
srun -n 4 singularity exec --bind /mnt container.sif python script.py
```

## Cloud Deployment

### AWS ParallelCluster

```bash
# Submit to Slurm on AWS
sbatch --nodes=4 --ntasks-per-node=8 mpi_job.sh
```

### Configuration File

```bash
# ~/.openmpi/mca-params.conf
btl = tcp,self
btl_tcp_if_include = eth0
mpi_show_mca_params = all
```

## Best Practices

1. **Use scheduler integration**: srun for SLURM, automatic detection
2. **Bind processes**: Improve performance with --bind-to
3. **Use appropriate network**: InfiniBand for HPC, TCP for cloud
4. **Set thread counts**: Avoid oversubscription
5. **Profile first**: Test at small scale
6. **Use hostfiles**: Explicit control over placement
7. **Monitor resources**: Check CPU/GPU utilization
8. **Handle failures**: Implement checkpointing
9. **Test connectivity**: Verify network before large runs
10. **Document configuration**: Save mpirun flags

## Common Pitfalls

- **Oversubscription**: More ranks than cores
- **Wrong network**: Using TCP when IB available
- **No process binding**: Cache thrashing
- **Environment issues**: Missing libraries
- **Heterogeneous nodes**: Different CPU counts
- **Firewall blocking**: MPI ports not open
- **Wrong MPI**: Multiple MPI implementations
- **Not using scheduler**: SSH overhead
- **Incorrect paths**: Different paths on nodes
- **Resource limits**: ulimit too low

## Platform-Specific Notes

### Open MPI

```bash
# Check version
ompi_info --version

# Show MCA parameters
ompi_info --param all all

# Tuned collectives
mpirun --mca coll_tuned_use_dynamic_rules 1 python script.py
```

### MPICH

```bash
# Check version
mpiexec --version

# Hydra process manager
mpiexec.hydra -np 4 python script.py

# Environment variable passing
mpiexec -genvall python script.py
```

### Intel MPI

```bash
# Check version
mpirun -V

# Fabric selection
mpirun -np 4 -genv I_MPI_FABRICS shm:ofi python script.py

# Debug info
mpirun -np 4 -genv I_MPI_DEBUG=5 python script.py
```
