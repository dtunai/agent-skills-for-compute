# Slurm Job Submission Reference

Source: [Slurm sbatch Documentation](https://slurm.schedmd.com/sbatch.html)

## sbatch - Submit Batch Scripts

### Purpose

Submit batch scripts for later execution. Scripts typically contain one or more `srun` commands to launch parallel tasks.

### Basic Syntax

```bash
sbatch [OPTIONS] script_file
```

### Resource Allocation Options

#### CPU and Task Configuration

```bash
# Number of tasks (processes)
sbatch -n 16 script.sh
sbatch --ntasks=16 script.sh

# CPUs per task (threads)
sbatch -c 4 script.sh
sbatch --cpus-per-task=4 script.sh

# Number of nodes
sbatch -N 2 script.sh
sbatch --nodes=2-4 script.sh  # Min 2, max 4

# Tasks per node
sbatch --ntasks-per-node=8 script.sh

# Tasks per socket/core
sbatch --ntasks-per-socket=4 script.sh
sbatch --ntasks-per-core=1 script.sh
```

#### Memory Configuration

```bash
# Memory per node (MB)
sbatch --mem=16G script.sh
sbatch --mem=16384 script.sh  # Same, in MB

# Memory per CPU
sbatch --mem-per-cpu=4G script.sh

# Memory per GPU
sbatch --mem-per-gpu=8G script.sh
```

**Important:** If neither --mem nor --mem-per-cpu specified, uses partition default or system limit.

#### Time Limits

```bash
# Time limit formats
sbatch -t 01:30:00 script.sh     # hh:mm:ss
sbatch -t 90 script.sh           # minutes
sbatch -t 1-12:00:00 script.sh   # days-hh:mm:ss

# Minimum time (for preemption)
sbatch --time-min=00:30:00 --time=02:00:00 script.sh
```

### GPU and GRES Options

```bash
# Total GPUs
sbatch -G 4 script.sh
sbatch --gpus=4 script.sh

# GPUs per node
sbatch --gpus-per-node=2 script.sh

# GPUs per socket
sbatch --gpus-per-socket=1 script.sh

# GPUs per task
sbatch --gpus-per-task=1 script.sh

# GPU type specification
sbatch --gres=gpu:tesla:2 script.sh
sbatch --gres=gpu:a100:4 script.sh

# Multiple GRES
sbatch --gres=gpu:2,mps:100 script.sh
```

### Node Selection and Constraints

#### Feature Constraints

```bash
# Single feature
sbatch -C avx2 script.sh

# Multiple features (AND)
sbatch -C "avx2&ib" script.sh

# Alternative features (OR)
sbatch -C "gpu|bigmem" script.sh

# Feature counts
sbatch -C "[rack1*2&rack2*2]" script.sh

# Preferred (not required)
sbatch --prefer="avx512,nvme" script.sh
```

#### Explicit Node Selection

```bash
# Specific nodes
sbatch -w node[01-04] script.sh
sbatch --nodelist=node01,node05 script.sh

# Exclude nodes
sbatch -x node02 script.sh
sbatch --exclude=node[10-20] script.sh
```

### Job Naming and I/O

#### Job Identification

```bash
# Job name
sbatch -J my_job script.sh
sbatch --job-name=experiment_42 script.sh

# Account billing
sbatch -A project_name script.sh
sbatch --account=gpu_users script.sh
```

#### Output Files

```bash
# Standard output
sbatch -o output_%j.txt script.sh
sbatch --output=logs/job_%A_%a.out script.sh

# Standard error
sbatch -e error_%j.txt script.sh
sbatch --error=logs/job_%A_%a.err script.sh

# Combined output and error
sbatch -o combined_%j.log script.sh

# Append vs truncate
sbatch --open-mode=append -o log.txt script.sh
sbatch --open-mode=truncate -o log.txt script.sh
```

**Format Specifiers:**
- `%j`: Job ID
- `%A`: Array job ID
- `%a`: Array task ID
- `%N`: Node name
- `%u`: Username
- `%x`: Job name

### Working Directory

```bash
# Change directory before execution
sbatch -D /scratch/$USER/project script.sh
sbatch --chdir=/data/experiments script.sh
```

**Default:** Submission directory

### Partition and QOS

```bash
# Partition selection
sbatch -p gpu script.sh
sbatch --partition=debug script.sh

# Quality of Service
sbatch -q high script.sh
sbatch --qos=priority script.sh

# Priority adjustment
sbatch --priority=1000 script.sh
sbatch --nice=100 script.sh  # Lower priority
```

### Job Arrays

```bash
# Basic array
sbatch --array=0-99 script.sh

# With step
sbatch --array=0-99:10 script.sh  # 0,10,20,...,90

# Specific indices
sbatch --array=1,3,5,7,9 script.sh

# Concurrency limit
sbatch --array=0-999%50 script.sh  # Max 50 concurrent
```

**Inside script:**
```bash
echo "Task ID: $SLURM_ARRAY_TASK_ID"
echo "Job ID: $SLURM_ARRAY_JOB_ID"
```

### Job Dependencies

```bash
# After successful completion
sbatch -d afterok:12345 script.sh

# After any completion (success or fail)
sbatch -d afterany:12345 script.sh

# After corresponding array task
sbatch -d aftercorr:12345 script.sh

# Multiple dependencies
sbatch -d afterok:12345:12346 script.sh

# Singleton (one job with this name)
sbatch -d singleton script.sh
```

**Dependency Types:**
- `after:jobid`: After job starts
- `afterok:jobid`: After successful completion
- `afternotok:jobid`: After failed completion
- `afterany:jobid`: After completion (any state)
- `aftercorr:jobid`: After corresponding array task
- `singleton`: Only one job with name at a time

### Environment Variables

```bash
# Export all environment
sbatch --export=ALL script.sh

# Export none
sbatch --export=NONE script.sh

# Export specific variables
sbatch --export=VAR1,VAR2=value script.sh

# Export from file
sbatch --export-file=env.txt script.sh

# Get user login environment
sbatch --get-user-env script.sh
```

### Resource Distribution

```bash
# Distribution methods
sbatch -m block script.sh        # Block distribution
sbatch -m cyclic script.sh       # Cyclic (round-robin)
sbatch -m plane=4 script.sh      # Plane with size 4

# CPU binding
sbatch --cpu-bind=cores script.sh
sbatch --cpu-bind=sockets script.sh
sbatch --cpu-bind=threads script.sh

# Memory binding (NUMA)
sbatch --mem-bind=local script.sh
sbatch --mem-bind=prefer script.sh
```

### Exclusive and Sharing

```bash
# Exclusive node access
sbatch --exclusive script.sh

# Exclusive user access
sbatch --exclusive=user script.sh

# Allow over-subscription
sbatch --oversubscribe script.sh

# Over-commit (allocate as if 1 task/node)
sbatch -O script.sh
```

### Advanced Options

#### Signal Handling

```bash
# Send signal before time limit
sbatch --signal=B:USR1@60 script.sh  # 60 seconds before

# Kill on invalid dependency
sbatch --kill-on-invalid-dep=yes script.sh
```

#### Job Control

```bash
# Submit in held state
sbatch -H script.sh

# Requeue on failure
sbatch --requeue script.sh
sbatch --no-requeue script.sh

# No kill on node failure
sbatch --no-kill script.sh
```

#### Accounting and Profiling

```bash
# Accounting frequency
sbatch --acctg-freq=30 script.sh  # Every 30 seconds

# Enable profiling
sbatch --profile=All script.sh
sbatch --profile=Energy,Task script.sh
```

#### Burst Buffers

```bash
# Inline burst buffer spec
sbatch --bb="create size=10GB" script.sh

# Burst buffer from file
sbatch --bbf=bb_spec.txt script.sh
```

## srun - Interactive Execution

### Purpose

Submit jobs for immediate execution or initiate job steps within existing allocations.

### Basic Syntax

```bash
# Outside allocation: submit job
srun [OPTIONS] command

# Inside allocation: job step
srun [OPTIONS] command
```

### Common Usage Patterns

```bash
# Interactive shell
srun --pty bash

# Interactive with GPU
srun --gpus=1 --mem=16G --pty bash

# Run command on allocated resources
srun hostname
srun python train.py

# Parallel execution
srun -n 16 ./mpi_app

# Within sbatch script
#!/bin/bash
#SBATCH -n 8
srun ./parallel_task
```

### Key Differences from sbatch

- **Immediate execution**: Waits for resources, then runs
- **Interactive**: Can use --pty for interactive shell
- **Job steps**: Creates steps within batch jobs
- **No script**: Executes command directly

## salloc - Resource Allocation

### Purpose

Allocate resources and spawn a shell. Used for interactive work sessions.

### Basic Syntax

```bash
salloc [OPTIONS] [command]
```

### Usage Patterns

```bash
# Allocate and get shell
salloc -N 2 -n 8

# Inside allocation:
srun hostname
srun my_program
exit  # Release resources

# One-liner with command
salloc -N 1 srun hostname
```

### Common Workflow

```bash
# Allocate resources
salloc --nodes=2 --ntasks=8 --gpus=2 --time=02:00:00

# Now in interactive session
srun python preprocess.py
srun --gpus=1 python train.py
srun python evaluate.py

exit  # Done, release resources
```

## Best Practices

1. **Use sbatch for production**: Batch jobs for reproducibility
2. **Use srun for interactive development**: Quick testing
3. **Use salloc for multi-step interactive**: Complex workflows
4. **Always specify time limits**: Prevents runaway jobs
5. **Request resources accurately**: Improves scheduling
6. **Use job arrays for parameter sweeps**: Efficient bulk submission
7. **Set meaningful job names**: Easier tracking
8. **Redirect output to logs directory**: Organized file management
9. **Use dependencies for pipelines**: Automate workflows
10. **Check available partitions**: `sinfo` before submission

## Common Pitfalls

- **Forgetting time limit**: May default or fail
- **Over-requesting resources**: Delays scheduling
- **Not checking queue**: `squeue` before long waits
- **Hardcoding paths**: Use $SLURM_SUBMIT_DIR
- **Not using array variables**: Duplicating work in arrays
- **Missing #SBATCH directives**: Options ignored
- **Wrong shebang**: Must be `#!/bin/bash` or similar
- **Not testing with small jobs**: Validate before large runs
- **Ignoring exit codes**: Check `sacct` for failures
- **Mixing allocation methods**: Confusing resource accounting
