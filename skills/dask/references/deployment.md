# Dask Deployment Reference

Sources:
- [Dask Deployment Documentation](https://docs.dask.org/en/stable/deploying.html)
- [Dask-Jobqueue Documentation](https://jobqueue.dask.org)
- [Dask Cloud Provider](https://cloudprovider.dask.org)
- [Coiled Documentation](https://docs.coiled.io)

## Deployment Options

Dask supports multiple deployment patterns:

1. **Local**: Single machine (development, small data)
2. **Manual**: SSH to machines, start scheduler/workers
3. **HPC**: Job queue systems (SLURM, PBS, SGE, LSF)
4. **Cloud**: AWS, GCP, Azure (auto-scaling)
5. **Kubernetes**: Container orchestration

## Local Deployment

### Single Machine

```python
from dask.distributed import Client

# Automatic local cluster
client = Client()

# Manual configuration
from dask.distributed import LocalCluster
cluster = LocalCluster(
    n_workers=4,
    threads_per_worker=2,
    processes=True,
    memory_limit='8GB'
)
client = Client(cluster)
```

### When to Use

- Development and testing
- Datasets that fit on one machine
- Prototyping before scaling
- Small to medium workloads (< 1TB)

## Manual Deployment

### Multi-Machine Setup

```bash
# On scheduler machine
dask scheduler
# Scheduler at: tcp://192.168.1.100:8786
# Dashboard at: http://192.168.1.100:8787/status

# On worker machines
dask worker tcp://192.168.1.100:8786
dask worker tcp://192.168.1.100:8786  # Multiple workers per machine
```

### Worker Configuration

```bash
dask worker tcp://scheduler:8786 \
    --nthreads 4 \
    --nworkers 2 \
    --memory-limit 16GB \
    --local-directory /scratch \
    --dashboard-address :8788
```

### Client Connection

```python
from dask.distributed import Client

client = Client('tcp://192.168.1.100:8786')
```

### When to Use

- Fixed cluster of known machines
- On-premise infrastructure
- Custom network configurations
- Long-running clusters

## HPC Deployment (Dask-Jobqueue)

### Overview

Dask-Jobqueue enables deployment on high-performance computing clusters using job scheduling systems.

**Supported Schedulers:**
- SLURM
- PBS/Torque
- MOAB
- SGE (Sun Grid Engine)
- LSF (IBM Spectrum)
- HTCondor

### Installation

```bash
pip install dask-jobqueue

# Or with conda
conda install -c conda-forge dask-jobqueue
```

### SLURM Cluster

```python
from dask_jobqueue import SLURMCluster
from dask.distributed import Client

cluster = SLURMCluster(
    cores=24,              # CPUs per job
    processes=4,           # Dask workers per job
    memory="96GB",         # Memory per job
    walltime="04:00:00",   # Job time limit
    queue="gpu",           # Partition
    account="myproject",   # Account for billing
    job_extra_directives=[
        "--gres=gpu:2",
        "--constraint=v100"
    ]
)

# Submit worker jobs
cluster.scale(jobs=10)  # 10 SLURM jobs

# Or adaptive scaling
cluster.adapt(minimum_jobs=2, maximum_jobs=50)

client = Client(cluster)
```

### PBS Cluster

```python
from dask_jobqueue import PBSCluster

cluster = PBSCluster(
    cores=24,
    processes=4,
    memory="96GB",
    walltime="04:00:00",
    queue="gpu",
    resource_spec="select=1:ncpus=24:mem=96GB:ngpus=2"
)

cluster.scale(jobs=10)
client = Client(cluster)
```

### SGE Cluster

```python
from dask_jobqueue import SGECluster

cluster = SGECluster(
    cores=16,
    processes=4,
    memory="64GB",
    walltime="02:00:00",
    queue="all.q",
    resource_spec="h_vmem=64G"
)

cluster.scale(jobs=20)
client = Client(cluster)
```

### LSF Cluster

```python
from dask_jobqueue import LSFCluster

cluster = LSFCluster(
    cores=24,
    processes=4,
    memory="96GB",
    walltime="04:00",
    queue="normal",
    project="myproject"
)

cluster.scale(jobs=10)
client = Client(cluster)
```

### Job Script Templates

Dask-Jobqueue generates job scripts. View template:

```python
print(cluster.job_script())
```

Example SLURM script generated:

```bash
#!/bin/bash

#SBATCH -J dask-worker
#SBATCH -n 24
#SBATCH --cpus-per-task=1
#SBATCH --mem=96G
#SBATCH -t 04:00:00
#SBATCH -p gpu
#SBATCH --gres=gpu:2

/path/to/python -m distributed.cli.dask_worker tcp://scheduler:8786 \
    --nthreads 6 --nworkers 4 --memory-limit 24GB
```

### Deployment Patterns

#### Dynamic Clusters

Scheduler runs locally, workers submitted to queue:

```python
# Client submits worker jobs dynamically
cluster = SLURMCluster(...)
cluster.scale(jobs=10)
client = Client(cluster)

# Workload runs
result = computation.compute()

# Cleanup
client.close()
cluster.close()
```

**Advantages:**
- Autoscaling capabilities
- Backfill unused cluster capacity
- Flexible scaling during workload
- Earlier job starts

**Use When:**
- Cluster allows many small jobs
- Workload varies over time
- Interactive development

#### Batch Runners

Entire workload (client + scheduler + workers) submitted as single job:

```bash
#!/bin/bash
#SBATCH -N 10
#SBATCH -t 04:00:00

# Start scheduler
dask scheduler &
SCHEDULER_PID=$!
sleep 5

# Start workers
for i in {0..9}; do
    srun --nodes=1 --ntasks=1 --exclusive \
        dask worker tcp://$(hostname):8786 &
done

# Run workload
python my_script.py

# Cleanup
kill $SCHEDULER_PID
```

**Advantages:**
- Faster inter-worker communication
- Guaranteed worker availability
- Aligns with HPC practices
- Greater reliability

**Use When:**
- Large allocation preferred
- Network locality important
- Predictable workload

### Adaptive Scaling

```python
cluster = SLURMCluster(...)

# Enable adaptive scaling
cluster.adapt(
    minimum=2,      # Keep at least 2 workers
    maximum=100,    # Scale up to 100 workers
    interval='30s',  # Check every 30 seconds
    wait_count=3    # Wait 3 checks before scaling down
)

client = Client(cluster)
```

### Configuration Files

Create `~/.config/dask/jobqueue.yaml`:

```yaml
jobqueue:
  slurm:
    cores: 24
    memory: 96GB
    processes: 4
    walltime: '04:00:00'
    queue: gpu
    account: myproject
    job-extra-directives:
      - '--gres=gpu:2'
```

Use in code:

```python
from dask_jobqueue import SLURMCluster

cluster = SLURMCluster()  # Uses config file
```

## Cloud Deployment

### Coiled (Managed Service)

```bash
pip install coiled
```

```python
import coiled
from dask.distributed import Client

# Create cluster on AWS/GCP/Azure
cluster = coiled.Cluster(
    name="my-cluster",
    n_workers=10,
    software="coiled/default",  # Software environment
    worker_cpu=4,
    worker_memory="16 GiB",
    scheduler_cpu=2,
    scheduler_memory="8 GiB"
)

client = Client(cluster)

# Workload runs
result = computation.compute()

# Cleanup
cluster.close()
```

### Cloud Provider (AWS/GCP/Azure)

```bash
pip install dask-cloudprovider
```

#### AWS (EC2)

```python
from dask_cloudprovider.aws import EC2Cluster

cluster = EC2Cluster(
    n_workers=10,
    region='us-east-1',
    instance_type='m5.2xlarge',
    ami='ami-12345678',  # Dask-compatible AMI
    bootstrap=True,
    security_group='sg-12345678'
)

client = Client(cluster)
```

#### AWS (Fargate)

```python
from dask_cloudprovider.aws import FargateCluster

cluster = FargateCluster(
    n_workers=10,
    image='daskdev/dask:latest',
    fargate_use_private_ip=False,
    vpc='vpc-12345678',
    cluster_arn='arn:aws:ecs:us-east-1:...'
)

client = Client(cluster)
```

#### GCP

```python
from dask_cloudprovider.gcp import GCPCluster

cluster = GCPCluster(
    n_workers=10,
    projectid='my-project',
    zone='us-central1-a',
    machine_type='n1-standard-4',
    docker_image='daskdev/dask:latest'
)

client = Client(cluster)
```

#### Azure

```python
from dask_cloudprovider.azure import AzureVMCluster

cluster = AzureVMCluster(
    n_workers=10,
    resource_group='my-resource-group',
    vnet='my-vnet',
    security_group='my-sg',
    vm_size='Standard_D4s_v3',
    docker_image='daskdev/dask:latest'
)

client = Client(cluster)
```

### Adaptive Cloud Scaling

```python
cluster = EC2Cluster(...)

# Auto-scale based on load
cluster.adapt(
    minimum=2,
    maximum=50,
    interval='30s'
)
```

## Kubernetes Deployment

### Dask Kubernetes Operator

```bash
# Install operator
kubectl apply -f https://raw.githubusercontent.com/dask/dask-kubernetes/main/dask_kubernetes/operator/deployment.yaml
```

### Create Cluster

```python
from dask_kubernetes.operator import KubeCluster

cluster = KubeCluster(
    name='my-dask-cluster',
    namespace='default',
    image='ghcr.io/dask/dask:latest',
    n_workers=10,
    resources={
        'requests': {'cpu': '2', 'memory': '8Gi'},
        'limits': {'cpu': '4', 'memory': '16Gi'}
    }
)

client = Client(cluster)
```

### Helm Chart

```bash
# Add Dask helm repo
helm repo add dask https://helm.dask.org
helm repo update

# Install Dask
helm install my-dask dask/dask \
    --set worker.replicas=10 \
    --set worker.resources.requests.memory=8Gi \
    --set worker.resources.requests.cpu=2
```

### Adaptive Kubernetes

```python
cluster = KubeCluster(...)

cluster.adapt(
    minimum=2,
    maximum=100
)
```

## Deployment Best Practices

### 1. Resource Sizing

```python
# CPU-bound: More workers, fewer threads
cluster = LocalCluster(
    n_workers=8,
    threads_per_worker=1,
    processes=True
)

# I/O-bound: Fewer workers, more threads
cluster = LocalCluster(
    n_workers=2,
    threads_per_worker=4,
    processes=True
)
```

### 2. Memory Management

```python
# Set conservative memory limits
cluster = SLURMCluster(
    memory="96GB",
    processes=4  # 24GB per worker
)

# Enable spilling
cluster = LocalCluster(
    memory_limit='8GB',
    memory_spill_fraction=0.8,
    memory_target_fraction=0.6
)
```

### 3. Network Configuration

```python
# Specify network interfaces
cluster = LocalCluster(
    host='192.168.1.100',  # External interface
    dashboard_address=':8787'
)

# Use TLS for security
from dask.distributed import Security

security = Security(
    tls_ca_file='ca.pem',
    tls_cert='cert.pem',
    tls_key='key.pem'
)

client = Client('tls://scheduler:8786', security=security)
```

### 4. Monitoring

```python
# Always use dashboard
print(client.dashboard_link)

# Log to file
import logging
logging.basicConfig(
    filename='dask.log',
    level=logging.INFO
)
```

### 5. Cleanup

```python
# Always close resources
try:
    result = computation.compute()
finally:
    client.close()
    cluster.close()

# Or use context manager
with Client(cluster) as client:
    result = computation.compute()
```

## Environment Management

### Conda Environments

```bash
# Create environment
conda create -n dask-env python=3.10 dask distributed

# Export environment
conda env export > environment.yml

# Replicate on workers
conda env create -f environment.yml
```

### Docker Containers

```dockerfile
FROM python:3.10-slim

RUN pip install dask[complete] distributed

WORKDIR /app
COPY . .

CMD ["dask", "worker", "tcp://scheduler:8786"]
```

### Software Environments (Coiled)

```python
import coiled

# Create software environment
coiled.create_software_environment(
    name="my-env",
    conda={
        "channels": ["conda-forge"],
        "dependencies": [
            "python=3.10",
            "dask",
            "pandas",
            "scikit-learn"
        ]
    }
)

# Use in cluster
cluster = coiled.Cluster(
    software="my-env",
    n_workers=10
)
```

## Troubleshooting

### Connection Issues

```python
# Test connection
from dask.distributed import Client

try:
    client = Client('tcp://scheduler:8786', timeout='10s')
    print("Connected!")
except Exception as e:
    print(f"Connection failed: {e}")
```

### Worker Failures

```bash
# Check worker logs (SLURM)
cat slurm-*.out

# Check scheduler logs
cat dask-scheduler.log

# Increase worker timeout
cluster = SLURMCluster(
    death_timeout='300s'  # 5 minutes
)
```

### Performance Issues

```python
# Profile deployment
from dask.distributed import performance_report

with performance_report(filename="deployment_profile.html"):
    result = computation.compute()
```

### Resource Limits

```bash
# Check HPC limits
scontrol show config | grep MaxArraySize
scontrol show partition

# Adjust requests
cluster = SLURMCluster(
    cores=24,     # Within partition limit
    memory="96GB",  # Within partition limit
    walltime="04:00:00"  # Within QOS limit
)
```

## Comparison Matrix

| Deployment | Setup Effort | Auto-Scaling | Cost | Best For |
|------------|-------------|--------------|------|----------|
| Local | Low | No | Free | Development |
| Manual | Medium | No | Low | Fixed clusters |
| HPC (Jobqueue) | Medium | Yes | Low | Academic/Research |
| Cloud (Coiled) | Low | Yes | Medium | Production |
| Cloud (DIY) | High | Yes | Low | Cost optimization |
| Kubernetes | High | Yes | Variable | Container workloads |

## Example Workflows

### HPC Batch Job

```python
# submit_job.py
from dask_jobqueue import SLURMCluster
from dask.distributed import Client
import dask.dataframe as dd

cluster = SLURMCluster(
    cores=24,
    processes=4,
    memory="96GB",
    walltime="04:00:00",
    queue="gpu",
    job_extra_directives=["--gres=gpu:2"]
)

cluster.scale(jobs=20)
client = Client(cluster)

# Wait for workers
client.wait_for_workers(20)

# Computation
df = dd.read_parquet('data/*.parquet')
result = df.groupby('key').mean().compute()
result.to_parquet('output/')

client.close()
cluster.close()
```

Submit:

```bash
sbatch --wrap="python submit_job.py"
```

### Cloud Auto-Scaling

```python
import coiled
from dask.distributed import Client
import dask.dataframe as dd

# Create auto-scaling cluster
cluster = coiled.Cluster(
    n_workers=[2, 50],  # Auto-scale 2-50 workers
    software="coiled/default",
    worker_cpu=4,
    worker_memory="16 GiB"
)

client = Client(cluster)

# Large computation
df = dd.read_parquet('s3://bucket/data/*.parquet')
result = df.groupby('category').agg({'value': ['sum', 'mean']})

# Cluster scales based on demand
result.to_parquet('s3://bucket/output/')

cluster.close()
```
