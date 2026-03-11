# Slurm Monitoring and Accounting Reference

Sources:
- [Slurm Quick Start Guide](https://slurm.schedmd.com/quickstart.html)
- [Slurm QOS Documentation](https://slurm.schedmd.com/qos.html)
- [Slurm sacctmgr Documentation](https://slurm.schedmd.com/sacctmgr.html)

## squeue - View Job Queue

### Purpose

Reports the state of jobs or job steps with filtering and sorting options.

### Basic Usage

```bash
# All jobs
squeue

# Your jobs only
squeue -u $USER

# Specific job
squeue -j JOBID

# Specific partition
squeue -p gpu

# By state
squeue -t RUNNING
squeue -t PENDING
```

### State Codes

| Code | State | Meaning |
|------|-------|---------|
| PD | PENDING | Waiting for resources |
| R | RUNNING | Currently executing |
| S | SUSPENDED | Execution suspended |
| CG | COMPLETING | Finishing up |
| CD | COMPLETED | Successfully completed |
| CA | CANCELLED | Cancelled by user/admin |
| F | FAILED | Failed with non-zero exit |
| TO | TIMEOUT | Exceeded time limit |
| NF | NODE_FAIL | Terminated due to node failure |

### Custom Format

```bash
# Default format
squeue -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"

# Detailed format
squeue -o "%.10i %.9P %.20j %.8u %.8T %.10M %.10l %.6D %R"

# GPU info
squeue -o "%.10i %.9P %.8j %.8u %.2t %b %R"
```

**Format Specifiers:**
- `%i`: Job ID
- `%P`: Partition
- `%j`: Job name
- `%u`: Username
- `%t`: Job state (compact)
- `%T`: Job state (full)
- `%M`: Time used
- `%l`: Time limit
- `%D`: Number of nodes
- `%R`: Reason/nodelist
- `%b`: GRES (GPUs)

### Array Jobs

```bash
# Compressed view (default)
squeue -j 12345

# Show individual tasks
squeue --array -j 12345

# Specific array task
squeue -j 12345_5
```

### Job Start Time

```bash
# Show start time estimates
squeue --start -u $USER
```

## sacct - Job Accounting

### Purpose

Display job accounting information from Slurm database.

### Basic Usage

```bash
# Recent jobs
sacct

# Specific job
sacct -j JOBID

# Time range
sacct -S 2026-01-01 -E 2026-01-31

# Your jobs only
sacct -u $USER

# Specific partition
sacct -p gpu
```

### Time Formats

```bash
# Start time
sacct -S YYYY-MM-DD
sacct -S YYYY-MM-DDTHH:MM:SS

# End time
sacct -E 2026-02-15

# Relative times
sacct -S now-1week
sacct -S now-1month
```

### Custom Fields

```bash
# Basic fields
sacct -j JOBID --format=JobID,JobName,Partition,State,ExitCode

# Detailed performance
sacct -j JOBID --format=JobID,Elapsed,CPUTime,TotalCPU,MaxRSS,MaxVMSize

# GPU usage
sacct -j JOBID --format=JobID,AllocGRES,TRESUsageInAve

# Full resource usage
sacct -j JOBID --format=JobID,State,Elapsed,CPUTime,TotalCPU,MaxRSS,MaxVMSize,AllocGRES
```

**Common Fields:**
- `JobID`: Job identifier
- `JobName`: Job name
- `Partition`: Partition used
- `Account`: Account charged
- `State`: Final job state
- `ExitCode`: Exit code
- `Elapsed`: Wall clock time
- `CPUTime`: CPU time allocated
- `TotalCPU`: CPU time used
- `MaxRSS`: Maximum RSS (memory)
- `MaxVMSize`: Maximum virtual memory
- `AllocCPUS`: CPUs allocated
- `AllocGRES`: GRES allocated (e.g., GPUs)
- `TRESUsageInAve`: Average TRES usage

### TRES (Trackable Resources)

```bash
# GPU memory and utilization
sacct -j JOBID --format=JobID,TRESUsageInAve

# Filter for specific TRES
sacct -j JOBID --format=JobID,TRESUsageInAve | grep gres/gpu
sacct -j JOBID --format=JobID,TRESUsageInAve | grep gres/gpumem
sacct -j JOBID --format=JobID,TRESUsageInAve | grep gres/gpuutil
```

### Job Steps

```bash
# Include job steps
sacct -j JOBID --format=JobID,JobName,State,Elapsed

# Output includes:
# 12345       job_name    COMPLETED  01:23:45
# 12345.0     step1       COMPLETED  00:30:00
# 12345.1     step2       COMPLETED  00:53:45
```

## sstat - Running Job Statistics

### Purpose

Report resource utilization for currently running jobs.

### Basic Usage

```bash
# All steps of running job
sstat -j JOBID

# Specific fields
sstat -j JOBID --format=JobID,AveCPU,AveRSS,AveVMSize

# All available fields
sstat -j JOBID -a
```

### Common Fields

```bash
sstat -j JOBID --format=JobID,AveCPU,MinCPU,MaxCPU,AveRSS,MaxRSS,AveVMSize,MaxVMSize
```

**Fields:**
- `AveCPU`: Average CPU time
- `MinCPU`: Minimum CPU time
- `MaxCPU`: Maximum CPU time
- `AveRSS`: Average RSS
- `MaxRSS`: Maximum RSS
- `AveVMSize`: Average VM size
- `MaxVMSize`: Maximum VM size

## sinfo - Partition and Node Info

### Purpose

Display partition and node status.

### Basic Usage

```bash
# All partitions
sinfo

# Specific partition
sinfo -p gpu

# Node-centric view
sinfo -N

# Long format
sinfo -l
```

### Custom Format

```bash
# Partition summary
sinfo --format="%P %.5a %.10l %.6D %.6t %N"

# GPU nodes
sinfo --Format=NodeList,Gres:30,GresUsed:30,StateLong

# Detailed node info
sinfo -N --format="%N %.6D %.9P %.11T %.4c %.8z %.6m %.8d %.6w %.8f %G"
```

**Format Specifiers:**
- `%P`: Partition name
- `%a`: Availability
- `%l`: Time limit
- `%D`: Number of nodes
- `%t`: State
- `%N`: Node list
- `%c`: CPUs
- `%m`: Memory
- `%G`: GRES (GPUs)
- `%f`: Features

### Node States

| State | Meaning |
|-------|---------|
| idle | Available for jobs |
| alloc | Fully allocated |
| mix | Partially allocated |
| down | Unavailable (down) |
| drain | Not accepting new jobs |

## scontrol - Administrative Tool

### Purpose

View and modify Slurm state (many commands require root).

### View Information

```bash
# Job details
scontrol show job JOBID

# Node details
scontrol show node nodename

# Partition details
scontrol show partition partition_name

# Reservation details
scontrol show reservation reservation_name

# Configuration
scontrol show config
```

### Modify Jobs (User Commands)

```bash
# Hold job
scontrol hold JOBID

# Release job
scontrol release JOBID

# Requeue job
scontrol requeue JOBID

# Update time limit
scontrol update JobId=JOBID TimeLimit=10:00:00

# Update QOS
scontrol update JobId=JOBID QOS=high

# Update dependency
scontrol update JobId=JOBID Dependency=afterok:12345
```

## QOS (Quality of Service)

### Purpose

QOS affects scheduling priority, preemption, and resource limits.

### View QOS

```bash
# List all QOS
sacctmgr list qos

# Detailed QOS info
sacctmgr list qos format=Name,Priority,MaxWall,MaxTRES
```

### Request QOS

```bash
# Submit with QOS
sbatch --qos=high script.sh
sbatch --qos=gpu script.sh
```

### QOS Limits

Common QOS parameters:
- `Priority`: Scheduling priority
- `MaxWall`: Maximum wall clock time
- `MaxTRES`: Maximum trackable resources (CPUs, GPUs, memory)
- `MaxJobsPerUser`: Maximum concurrent jobs
- `MaxSubmitJobsPerUser`: Maximum pending jobs
- `Preempt`: Can preempt other QOS jobs
- `PreemptMode`: How preemption occurs

## Partitions

### View Partitions

```bash
# List all partitions
scontrol show partition

# Specific partition
scontrol show partition gpu
```

### Partition Properties

- **AllowGroups**: Users/groups allowed
- **MaxTime**: Maximum time limit
- **DefaultTime**: Default time if not specified
- **MaxNodes**: Maximum nodes per job
- **State**: UP, DOWN, DRAIN, INACTIVE

## Accounting Reports

### User Summary

```bash
# User's total usage
sacct -u $USER -S 2026-01-01 --format=User,Account,JobID,Elapsed,State
```

### Account Summary

```bash
# Account usage
sacct -A project_name -S 2026-01-01 --format=Account,User,JobID,AllocCPUS,Elapsed
```

### Failed Jobs

```bash
# Show failed jobs
sacct -u $USER -S 2026-01-01 -s FAILED --format=JobID,JobName,State,ExitCode

# Non-zero exit codes
sacct -u $USER -S 2026-01-01 --format=JobID,JobName,ExitCode | grep -v "0:0"
```

### Resource Efficiency

```bash
# CPU efficiency
sacct -j JOBID --format=JobID,CPUTime,TotalCPU

# Calculate efficiency
# Efficiency = TotalCPU / CPUTime

# Memory usage
sacct -j JOBID --format=JobID,MaxRSS,ReqMem
```

## Monitoring Scripts

### Check Queue Status

```bash
#!/bin/bash
# queue_status.sh

echo "Your Running Jobs:"
squeue -u $USER -t RUNNING

echo -e "\nYour Pending Jobs:"
squeue -u $USER -t PENDING

echo -e "\nPartition Status:"
sinfo --format="%P %.5a %.10l %.6D %.6t"
```

### Job Completion Notification

```bash
#!/bin/bash
#SBATCH --job-name=notify_job
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=your@email.com

# Your commands
./my_application
```

### Resource Usage Report

```bash
#!/bin/bash
# usage_report.sh

JOBID=$1

echo "Job Summary for $JOBID:"
sacct -j $JOBID --format=JobID,JobName,Partition,State,ExitCode,Elapsed

echo -e "\nResource Usage:"
sacct -j $JOBID --format=JobID,AllocCPUS,TotalCPU,MaxRSS,MaxVMSize,AllocGRES

echo -e "\nGPU Usage:"
sacct -j $JOBID --format=JobID,TRESUsageInAve | grep gres/gpu
```

## Best Practices

1. **Monitor job status regularly**: `squeue -u $USER`
2. **Check resource efficiency**: Optimize future requests
3. **Use sacct for completed jobs**: Not squeue
4. **Set up email notifications**: Catch failures early
5. **Profile resource usage**: Right-size allocations
6. **Check partition limits**: Before submission
7. **Use QOS appropriately**: Match priority to urgency
8. **Save job IDs**: For later analysis
9. **Monitor GPU utilization**: Avoid waste
10. **Review failed jobs**: Fix errors promptly

## Common Pitfalls

- **Using squeue for old jobs**: Use sacct instead
- **Not checking exit codes**: Missing failures
- **Ignoring resource efficiency**: Wasting allocations
- **Not setting time estimates**: Default may be too short
- **Forgetting to specify account**: Billing issues
- **Not monitoring pending time**: Could indicate issues
- **Assuming immediate start**: Check `squeue --start`
- **Not cleaning up old output**: Cluttered directories
- **Ignoring email notifications**: Missing failures
- **Not documenting job IDs**: Hard to track experiments
