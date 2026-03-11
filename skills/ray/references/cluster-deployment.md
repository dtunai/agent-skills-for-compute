# Ray Cluster Deployment and Management Reference

Sources:
- [Ray Cluster Documentation](https://docs.ray.io/en/latest/cluster/getting-started.html)
- [Ray on Kubernetes](https://docs.ray.io/en/latest/cluster/kubernetes/index.html)
- [Ray on Cloud](https://docs.ray.io/en/latest/cluster/vms/index.html)
- [Ray Job Submission](https://docs.ray.io/en/latest/cluster/running-applications/job-submission/index.html)

## Overview

Ray supports native cluster deployment across cloud providers (AWS, GCP, Azure), Kubernetes, and on-premise infrastructure. According to the documentation, clusters can "maintain fixed sizes or dynamically scale based on workload demands through the cluster autoscaler feature."

**Important Constraint**: Multi-node Ray clusters require Linux environments. Windows and macOS deployments are unsupported.

## Local Development

### Single Machine

```python
import ray

# Auto-detect resources
ray.init()

# Manual configuration
ray.init(
    num_cpus=8,
    num_gpus=2,
    memory=16 * 1024**3,  # 16GB
    object_store_memory=4 * 1024**3  # 4GB
)

# Custom resources
ray.init(resources={"special_hardware": 4})
```

### Local Cluster (Manual)

```bash
# Start head node
ray start --head --port=6379

# On other machines, start workers
ray start --address='head-node-ip:6379'

# Connect from Python
import ray
ray.init(address='head-node-ip:6379')
```

## Cloud Deployment

### Installation

```bash
# Install Ray with cloud support
pip install -U "ray[default]"

# AWS dependencies
pip install boto3

# GCP dependencies
pip install google-api-python-client google-cloud-storage

# Azure dependencies
pip install azure-cli azure-core
```

### AWS Deployment

#### Cluster Configuration (cluster.yaml)

```yaml
cluster_name: ray-aws-cluster

provider:
    type: aws
    region: us-west-2
    availability_zone: us-west-2a

auth:
    ssh_user: ubuntu

head_node_type: head_node

available_node_types:
    head_node:
        node_config:
            InstanceType: m5.2xlarge
            ImageId: ami-0a2363a9cff180a64  # Ubuntu 20.04
            BlockDeviceMappings:
                - DeviceName: /dev/sda1
                  Ebs:
                      VolumeSize: 100
        min_workers: 0
        max_workers: 0

    worker_node:
        node_config:
            InstanceType: m5.xlarge
            ImageId: ami-0a2363a9cff180a64
        min_workers: 0
        max_workers: 10

# Setup commands
setup_commands:
    - pip install -U ray[default] torch
    - pip install -U transformers datasets

# Files to sync
file_mounts:
    "/home/ubuntu/data": "s3://bucket/data"

# Initialization
head_setup_commands:
    - pip install -U ray[default]

worker_setup_commands:
    - pip install -U ray[default]
```

#### Launch Cluster

```bash
# Create cluster
ray up cluster.yaml

# SSH to head node
ray attach cluster.yaml

# Execute command
ray exec cluster.yaml 'python script.py'

# Submit job
ray submit cluster.yaml script.py --args

# Get cluster status
ray status

# Tear down cluster
ray down cluster.yaml
```

### GCP Deployment

```yaml
cluster_name: ray-gcp-cluster

provider:
    type: gcp
    region: us-central1
    availability_zone: us-central1-a
    project_id: my-project-id

auth:
    ssh_user: ubuntu

head_node_type: head_node

available_node_types:
    head_node:
        node_config:
            machineType: n1-standard-4
            disks:
              - boot: true
                autoDelete: true
                initializeParams:
                    diskSizeGb: 100
                    sourceImage: projects/ubuntu-os-cloud/global/images/family/ubuntu-2004-lts
        min_workers: 0
        max_workers: 0

    worker_node:
        node_config:
            machineType: n1-standard-2
            disks:
              - boot: true
                autoDelete: true
                initializeParams:
                    diskSizeGb: 50
        min_workers: 0
        max_workers: 10
        resources: {}

setup_commands:
    - pip install -U ray[default]
```

### Azure Deployment

```yaml
cluster_name: ray-azure-cluster

provider:
    type: azure
    location: westus2
    resource_group: ray-cluster-rg
    subscription_id: your-subscription-id

auth:
    ssh_user: ubuntu

head_node_type: head_node

available_node_types:
    head_node:
        node_config:
            azure_arm_parameters:
                vmSize: Standard_D4s_v3
                imagePublisher: Canonical
                imageOffer: UbuntuServer
                imageSku: 18.04-LTS
                imageVersion: latest
        min_workers: 0
        max_workers: 0

    worker_node:
        node_config:
            azure_arm_parameters:
                vmSize: Standard_D2s_v3
        min_workers: 0
        max_workers: 10

setup_commands:
    - pip install -U ray[default]
```

## Kubernetes Deployment

### Installation

```bash
# Install KubeRay operator
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update

# Install operator
helm install kuberay-operator kuberay/kuberay-operator --version 1.0.0
```

### Ray Cluster Specification

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  name: ray-cluster
spec:
  rayVersion: '2.53.0'

  headGroupSpec:
    rayStartParams:
      dashboard-host: '0.0.0.0'
      num-cpus: '0'  # Don't schedule tasks on head
    replicas: 1
    template:
      spec:
        containers:
        - name: ray-head
          image: rayproject/ray:2.53.0
          ports:
          - containerPort: 6379  # Redis
          - containerPort: 8265  # Dashboard
          - containerPort: 10001  # Client
          resources:
            limits:
              cpu: "4"
              memory: "16Gi"
            requests:
              cpu: "2"
              memory: "8Gi"

  workerGroupSpecs:
  - replicas: 3
    minReplicas: 1
    maxReplicas: 10
    groupName: worker-group
    rayStartParams: {}
    template:
      spec:
        containers:
        - name: ray-worker
          image: rayproject/ray:2.53.0
          resources:
            limits:
              cpu: "4"
              memory: "16Gi"
              nvidia.com/gpu: "1"
            requests:
              cpu: "2"
              memory: "8Gi"
              nvidia.com/gpu: "1"
```

### Deploy Cluster

```bash
# Apply configuration
kubectl apply -f ray-cluster.yaml

# Check status
kubectl get rayclusters

# Get head node service
kubectl get svc ray-cluster-head-svc

# Port forward to dashboard
kubectl port-forward svc/ray-cluster-head-svc 8265:8265

# Connect from Python
import ray
ray.init(address="ray://localhost:10001")
```

### GPU Cluster on Kubernetes

```yaml
workerGroupSpecs:
- replicas: 4
  minReplicas: 1
  maxReplicas: 8
  groupName: gpu-workers
  rayStartParams:
    num-gpus: "1"
  template:
    spec:
      containers:
      - name: ray-worker
        image: rayproject/ray-ml:2.53.0-gpu
        resources:
          limits:
            nvidia.com/gpu: "1"
            cpu: "8"
            memory: "32Gi"
          requests:
            nvidia.com/gpu: "1"
            cpu: "4"
            memory: "16Gi"
```

## Autoscaling

### Cloud Autoscaling

```yaml
# In cluster.yaml
available_node_types:
    worker_node:
        min_workers: 2   # Minimum workers
        max_workers: 20  # Maximum workers
        node_config:
            InstanceType: m5.xlarge

# Autoscaler settings
autoscaling_mode: default
idle_timeout_minutes: 5  # Terminate idle workers after 5 min
```

### Kubernetes Autoscaling

```yaml
spec:
  workerGroupSpecs:
  - minReplicas: 2
    maxReplicas: 20
    replicas: 5  # Initial replicas
    scaleStrategy:
      workersToDelete: []
```

### Programmatic Scaling

```python
from ray.autoscaler.sdk import request_resources

# Request specific resources
request_resources(num_cpus=100, num_gpus=10)

# Ray automatically scales cluster to meet demand
```

## Job Submission

### Ray Jobs API

```bash
# Submit job to cluster
ray job submit --address=http://localhost:8265 \
    --working-dir="./my_project" \
    -- python train.py --epochs=10

# Check job status
ray job status <job-id>

# Get job logs
ray job logs <job-id>

# Stop job
ray job stop <job-id>

# List all jobs
ray job list
```

### Python SDK

```python
from ray.job_submission import JobSubmissionClient

client = JobSubmissionClient("http://localhost:8265")

# Submit job
job_id = client.submit_job(
    entrypoint="python train.py",
    runtime_env={
        "working_dir": "./my_project",
        "pip": ["torch", "transformers"]
    }
)

# Wait for completion
status = client.get_job_status(job_id)
while status not in ["SUCCEEDED", "FAILED"]:
    time.sleep(5)
    status = client.get_job_status(job_id)

# Get logs
logs = client.get_job_logs(job_id)
print(logs)
```

## Runtime Environments

### Conda Environment

```python
import ray

ray.init(
    runtime_env={
        "conda": {
            "dependencies": [
                "python=3.9",
                "pytorch",
                "transformers"
            ]
        }
    }
)
```

### Pip Dependencies

```python
ray.init(
    runtime_env={
        "pip": [
            "torch==2.0.0",
            "transformers==4.30.0",
            "datasets"
        ]
    }
)
```

### Working Directory

```python
# Sync local directory to cluster
ray.init(
    runtime_env={
        "working_dir": "./my_project"
    }
)
```

### Container Image

```python
# Use custom Docker image
ray.init(
    runtime_env={
        "container": {
            "image": "rayproject/ray-ml:2.53.0-gpu",
            "worker_path": "/home/ray/anaconda3/bin/python"
        }
    }
)
```

## Monitoring and Observability

### Ray Dashboard

```python
# Dashboard runs on port 8265 by default
# Access at http://head-node:8265

# Configure dashboard
ray.init(dashboard_host="0.0.0.0", dashboard_port=8265)
```

### Prometheus Metrics

```yaml
# Enable Prometheus metrics
head_start_ray_commands:
  - ray start --head --metrics-export-port=8080

# Scrape metrics at http://head-node:8080
```

### Logging

```python
import logging
import ray

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

ray.init()

@ray.remote
def task():
    logger.info("Task executing")
    return "done"
```

## Resource Management

### Placement Groups

```python
from ray.util.placement_group import placement_group

# Create placement group
pg = placement_group([
    {"CPU": 4, "GPU": 1},
    {"CPU": 4, "GPU": 1},
    {"CPU": 4, "GPU": 1}
], strategy="STRICT_PACK")  # All on same node

ray.get(pg.ready())

# Use in tasks
@ray.remote(num_cpus=4, num_gpus=1)
def gpu_task():
    return work()

futures = [
    gpu_task.options(
        placement_group=pg,
        placement_group_bundle_index=i
    ).remote()
    for i in range(3)
]

results = ray.get(futures)
```

### Custom Resources

```python
# Define custom resources on nodes
ray start --head --resources='{"special_hardware": 4}'

# Use in tasks
@ray.remote(resources={"special_hardware": 1})
def special_task():
    return use_special_hardware()
```

## Multi-Cluster Federation

### Connecting Multiple Clusters

```python
# Connect to multiple clusters
import ray

# Cluster 1
ray.init(address="ray://cluster1:10001", namespace="cluster1")

# Cluster 2 (separate connection)
ray.init(address="ray://cluster2:10001", namespace="cluster2")
```

## Security

### TLS/SSL

```yaml
# Configure TLS in cluster.yaml
head_node:
    ray_start_commands:
        - ray start --head --tls-server-cert=/path/to/cert.pem --tls-server-key=/path/to/key.pem
```

### Authentication

```bash
# Set Ray password
export RAY_PASSWORD=mysecretpassword

# Connect with password
ray.init(address="ray://cluster:10001", _redis_password="mysecretpassword")
```

## Fault Tolerance

### Task Retries

```python
@ray.remote(max_retries=3, retry_exceptions=[ConnectionError])
def fault_tolerant_task():
    return unstable_operation()
```

### Actor Reconstruction

```python
@ray.remote(max_restarts=5)
class FaultTolerantActor:
    def method(self):
        return work()
```

### Checkpointing

```python
# Regular checkpointing
for i in range(1000):
    result = train_step()

    if i % 100 == 0:
        checkpoint = create_checkpoint(model)
        ray.put(checkpoint)  # Store in object store
```

## Performance Tuning

### Object Store Configuration

```bash
# Increase object store memory
ray start --head --object-store-memory=50000000000  # 50GB
```

### Network Configuration

```yaml
# Configure Ray for high-bandwidth networks
cluster_synced_files: []
file_mounts_sync_continuously: false
```

### Locality Scheduling

```python
# Ensure tasks run where data is located
large_data = ray.put(data)

@ray.remote
def process(data_ref):
    data = ray.get(data_ref)
    return transform(data)

# Task scheduled on node with data
result = process.remote(large_data)
```

## Best Practices

1. **Use cloud cluster launcher**: Simplifies AWS/GCP/Azure deployment
2. **Enable autoscaling**: Efficient resource usage
3. **Submit jobs via Ray Jobs**: Better than SSH
4. **Use runtime environments**: Isolate dependencies
5. **Monitor with dashboard**: Track resource usage
6. **Use placement groups**: Co-locate related tasks
7. **Configure object store**: Prevent memory issues
8. **Enable fault tolerance**: Automatic recovery
9. **Use TLS in production**: Secure communication
10. **Clean up resources**: Terminate unused clusters

## Common Pitfalls

- **Not using Ray Jobs**: Manual job management is error-prone
- **Undersized object store**: Out of memory errors
- **Not enabling autoscaling**: Wasted resources
- **Ignoring security**: Exposed clusters
- **Not monitoring**: Miss performance issues
- **Over-provisioning**: High costs
- **Not using runtime envs**: Dependency conflicts
- **Blocking head node**: Schedule tasks on workers
- **Not using placement groups**: Poor data locality
- **Forgetting to tear down**: Accumulating cloud costs

## Troubleshooting

### Connection Issues

```bash
# Check Ray status
ray status

# Check firewall rules
# Allow ports: 6379 (Redis), 8265 (Dashboard), 10001 (Client)

# Test connection
telnet head-node 6379
```

### Worker Not Joining

```bash
# Check worker logs
cat /tmp/ray/session_latest/logs/raylet.out

# Verify Ray version match
ray --version  # On head and workers

# Check network connectivity
ping head-node
```

### Out of Memory

```bash
# Increase object store
ray start --head --object-store-memory=100000000000

# Enable spilling to disk
ray start --head --object-spilling-enabled
```

### Slow Performance

```python
# Profile with Ray dashboard
# Check:
# - Task execution time
# - Data transfer time
# - Object store pressure
# - Resource utilization
```
