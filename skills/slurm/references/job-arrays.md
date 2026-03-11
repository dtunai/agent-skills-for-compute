# Slurm Job Arrays Reference

Source: [Slurm Job Array Documentation](https://slurm.schedmd.com/job_array.html)

## Overview

Job arrays enable efficient submission of similar jobs in bulk. Arrays with millions of tasks can be submitted in milliseconds, with all jobs sharing initial options.

## Basic Syntax

```bash
sbatch --array=<specification> [options] script.sh
```

## Array Specifications

### Range Syntax

```bash
# Tasks 0 through 15
sbatch --array=0-15 script.sh

# Tasks 1 through 100
sbatch --array=1-100 script.sh

# Single task
sbatch --array=5 script.sh
```

### Specific Values

```bash
# Explicit task IDs
sbatch --array=1,3,5,7,9 script.sh

# Mixed ranges and values
sbatch --array=0-10,20,30-40 script.sh
```

### Step Function

```bash
# Every 4th index: 0,4,8,12
sbatch --array=0-15:4 script.sh

# Even numbers: 0,2,4,6,8,10
sbatch --array=0-10:2 script.sh

# Every 10th: 0,10,20,...,100
sbatch --array=0-100:10 script.sh
```

**Syntax:** `start-end:step`

### Concurrency Limit

```bash
# Max 10 concurrent tasks
sbatch --array=0-99%10 script.sh

# Max 5 concurrent from 1000 tasks
sbatch --array=0-999%5 script.sh

# No limit (all tasks can run simultaneously)
sbatch --array=0-15 script.sh
```

**Syntax:** `<indices>%<max_running>`

**Purpose:** Prevents overwhelming the scheduler and cluster resources.

## Environment Variables

Five environment variables are automatically set for each array task:

| Variable | Description | Example |
|----------|-------------|---------|
| `SLURM_ARRAY_JOB_ID` | Job ID of entire array | 12345 |
| `SLURM_ARRAY_TASK_ID` | Current task's index | 7 |
| `SLURM_ARRAY_TASK_COUNT` | Total number of tasks | 100 |
| `SLURM_ARRAY_TASK_MAX` | Highest index value | 99 |
| `SLURM_ARRAY_TASK_MIN` | Lowest index value | 0 |

### Usage in Scripts

```bash
#!/bin/bash
#SBATCH --array=0-99

# Use task ID to process different data
INPUT_FILE="data_${SLURM_ARRAY_TASK_ID}.txt"
OUTPUT_FILE="result_${SLURM_ARRAY_TASK_ID}.txt"

python process.py --input $INPUT_FILE --output $OUTPUT_FILE
```

## Output File Naming

### Format Specifiers

| Specifier | Replacement | Example |
|-----------|-------------|---------|
| `%A` | SLURM_ARRAY_JOB_ID | 12345 |
| `%a` | SLURM_ARRAY_TASK_ID | 7 |
| `%j` | Job ID (array job + task) | 12345_7 |
| `%N` | Node name | node01 |
| `%u` | Username | alice |

### Examples

```bash
# Default format (if not specified)
# slurm-%A_%a.out

# Custom output files
#SBATCH --output=logs/job_%A_task_%a.out
#SBATCH --error=logs/job_%A_task_%a.err

# Single directory per array
#SBATCH --output=experiment_42/task_%a.log

# Organized by task ID ranges
#SBATCH --output=results/batch_%A/task_%a.txt
```

## Job Identification

Array jobs are identified by:
- **Array Job ID**: `SLURM_ARRAY_JOB_ID` (e.g., 12345)
- **Task ID**: Appended with underscore (e.g., 12345_7)

### Command Examples

```bash
# View all tasks
squeue -j 12345

# View specific task
squeue -j 12345_7

# Cancel entire array
scancel 12345

# Cancel specific tasks
scancel 12345_7
scancel 12345_[1-5]
```

## Job Array Commands

### squeue - View Array Status

```bash
# Default view (compressed)
squeue -j 12345

# Show individual tasks
squeue --array -j 12345

# Filter by task state
squeue -j 12345 -t RUNNING
squeue -j 12345 -t PENDING
```

### scancel - Cancel Tasks

```bash
# Cancel entire array
scancel 12345

# Cancel specific task
scancel 12345_7

# Cancel range of tasks
scancel 12345_[10-20]

# Cancel specific tasks
scancel 12345_[1,3,5,7]

# Cancel by state
scancel -j 12345 -t PENDING
```

### scontrol - Modify Arrays

```bash
# Update entire array
scontrol update JobId=12345 TimeLimit=10:00:00

# Update specific task
scontrol update JobId=12345_5 QOS=high

# Hold array
scontrol hold 12345

# Release array
scontrol release 12345

# Show array details
scontrol show job 12345
```

## Job Dependencies

### Dependency Types

```bash
# After entire array completes successfully
sbatch --dependency=afterok:12345 next_job.sh

# After any completion (success or failure)
sbatch --dependency=afterany:12345 next_job.sh

# After corresponding task completes
sbatch --dependency=aftercorr:12345 --array=0-99 next_array.sh
```

### aftercorr - Corresponding Tasks

**Most Important for Arrays:**

```bash
# First array
JOB1=$(sbatch --parsable --array=0-9 preprocess.sh)

# Second array waits for corresponding tasks
# Task 0 waits for task 0, task 1 waits for task 1, etc.
sbatch --dependency=aftercorr:$JOB1 --array=0-9 analyze.sh
```

**Use Case:** Multi-stage pipelines where each task in stage 2 depends on corresponding task in stage 1.

## Configuration Limits

### MaxArraySize

Maximum array size configured in `slurm.conf`:

```
MaxArraySize=1001  # Default
MaxArraySize=4000001  # Maximum supported
```

**Query current limit:**
```bash
scontrol show config | grep MaxArraySize
```

## Common Patterns

### 1. Parameter Sweep

```bash
#!/bin/bash
#SBATCH --array=0-99
#SBATCH --output=logs/param_%a.out

# Map task ID to parameters
LEARNING_RATES=(0.001 0.01 0.1)
BATCH_SIZES=(16 32 64 128)

# Calculate parameter combination
LR_IDX=$((SLURM_ARRAY_TASK_ID / 4))
BS_IDX=$((SLURM_ARRAY_TASK_ID % 4))

LR=${LEARNING_RATES[$LR_IDX]}
BS=${BATCH_SIZES[$BS_IDX]}

python train.py --lr $LR --batch_size $BS
```

### 2. File Processing

```bash
#!/bin/bash
#SBATCH --array=0-999

# List of files
FILES=(data/*.txt)

# Get file for this task
FILE=${FILES[$SLURM_ARRAY_TASK_ID]}

python process.py --input $FILE
```

### 3. Multi-Stage Pipeline

```bash
# Stage 1: Preprocessing
JOB1=$(sbatch --parsable --array=0-99%20 preprocess.sh)

# Stage 2: Training (waits for corresponding preprocess task)
JOB2=$(sbatch --parsable --dependency=aftercorr:$JOB1 --array=0-99%10 train.sh)

# Stage 3: Evaluation (waits for all training)
JOB3=$(sbatch --dependency=afterok:$JOB2 evaluate.sh)
```

### 4. GPU Array Job

```bash
#!/bin/bash
#SBATCH --array=0-19%5
#SBATCH --gpus-per-task=1
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-gpu=16G

# Each task gets dedicated GPU
python experiment.py --experiment_id $SLURM_ARRAY_TASK_ID
```

### 5. Chunked Processing

```bash
#!/bin/bash
#SBATCH --array=0-9

# Process chunk of large dataset
START=$((SLURM_ARRAY_TASK_ID * 1000))
END=$((START + 999))

python process_chunk.py --start $START --end $END
```

## Advanced Features

### Heterogeneous Array Tasks

```bash
# Different resources per task (not directly supported)
# Workaround: conditional resource request

#!/bin/bash
#SBATCH --array=0-9

if [ $SLURM_ARRAY_TASK_ID -lt 5 ]; then
    # First 5 tasks: CPU-only
    srun --cpus-per-task=4 ./cpu_task.sh
else
    # Last 5 tasks: GPU
    srun --gpus=1 ./gpu_task.sh
fi
```

### Dynamic Task Addition

Array tasks cannot be added after submission. Workaround:

```bash
# Submit new array with different indices
JOB1=$(sbatch --parsable --array=0-99 script.sh)
JOB2=$(sbatch --parsable --array=100-199 script.sh)

# Chain dependencies
sbatch --dependency=afterok:$JOB1:$JOB2 final.sh
```

## Performance Considerations

### Submission Speed

Job arrays submit much faster than individual jobs:

```bash
# Slow: 1000 individual submissions
for i in {0..999}; do
    sbatch script.sh
done

# Fast: Single array submission
sbatch --array=0-999 script.sh
```

**Speed Improvement:** Milliseconds vs. seconds/minutes.

### Scheduler Load

Concurrency limits reduce scheduler load:

```bash
# Bad: All 1000 tasks pending simultaneously
sbatch --array=0-999 script.sh

# Good: Max 50 running at once
sbatch --array=0-999%50 script.sh
```

### Resource Utilization

```bash
# Monitor array job efficiency
sacct -j 12345 --format=JobID,State,ExitCode,CPUTime,TotalCPU

# Check GPU utilization across array
sacct -j 12345 --format=JobID,TRESUsageInAve | grep gres/gpuutil
```

## Best Practices

1. **Use concurrency limits**: Prevent overwhelming scheduler
2. **Set meaningful output paths**: Easy result tracking
3. **Check MaxArraySize**: Know cluster limits
4. **Use aftercorr for pipelines**: Efficient task dependencies
5. **Monitor resource usage**: Optimize requests
6. **Test with small array first**: Validate before large submission
7. **Use task ID for randomness seed**: Reproducible experiments
8. **Organize output files**: Create subdirectories
9. **Handle task failures**: Check exit codes
10. **Document index mapping**: Comment parameter calculations

## Common Pitfalls

- **Not using task ID**: All tasks do identical work
- **No concurrency limit**: Overloading scheduler
- **Hardcoded file paths**: Tasks overwrite each other's output
- **Exceeding MaxArraySize**: Job rejected
- **Not checking exit codes**: Missing failed tasks
- **Over-requesting resources**: Unnecessary queuing
- **Forgetting %A_%a in output**: Files overwritten
- **Not testing single task**: Bugs multiply across array
- **Assuming sequential execution**: Tasks run in arbitrary order
- **Not cleaning up failed tasks**: Cluttered output directories

## Troubleshooting

### View Failed Tasks

```bash
# Show failed array tasks
sacct -j 12345 -X --format=JobID,State,ExitCode | grep FAILED
```

### Rerun Specific Tasks

```bash
# Cancel failed tasks
scancel -j 12345 -t FAILED

# Resubmit only failed tasks
sbatch --array=5,7,12,23 script.sh  # Specific failed IDs
```

### Debug Single Task

```bash
# Run one task interactively
export SLURM_ARRAY_TASK_ID=5
bash script.sh
```
