# NVIDIA NIM Deployment Reference

Sources:
- [NVIDIA NIM for LLMs](https://docs.nvidia.com/nim/large-language-models/latest/)
- [Deploy NVIDIA NIM](https://docs.nvidia.com/nemo/microservices/latest/run-inference/tutorials/deploy-nim.html)

## Overview

NVIDIA NIM (NVIDIA Inference Microservices) provides optimized containers for deploying LLMs with peak inference performance and OpenAI-compatible APIs.

## Prerequisites

### Hardware

**CPU:**
- x86 processor
- 8+ cores recommended

**GPU:**
- NVIDIA GPU with CUDA compute capability > 7.0
- Compute capability 8.0+ for bfloat16

**Memory:**
- System RAM: 5-10 GB for OS + (2GB per billion parameters)
- GPU VRAM: Model-dependent
  - Nano 30B: ~60GB (1x A100 80GB or H100)
  - Super 49B: ~90GB (2x A100 80GB or 1x H100)
  - Ultra 253B: ~500GB (4x A100 80GB or 2x H100)

### Software

**Required:**
- Linux OS (Ubuntu 22.04+ recommended)
- NVIDIA Driver 580+
- Docker 23.0.1+
- CUDA 13.0

**Optional:**
- NGC CLI for registry management
- Kubernetes for orchestration

### Access

**NGC API Key:**
1. Create NVIDIA Developer account
2. Visit https://build.nvidia.com or NGC
3. Generate API key
4. Store securely

## Deployment Options

### Option 1: API Catalog (Recommended)

```bash
# Get NGC API key
export NGC_API_KEY="your_api_key_here"

# Authenticate Docker
echo "$NGC_API_KEY" | docker login nvcr.io \
  --username '$oauthtoken' \
  --password-stdin

# Deploy Nemotron 3 Nano
docker run -it --rm \
  --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0

# Wait for startup
# INFO: Application startup complete
# INFO: Uvicorn running on http://0.0.0.0:8000
```

### Option 2: NGC Registry

```bash
# List available NIMs
ngc registry image list "nim/*"

# Pull specific version
docker pull nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0

# Run with explicit tag
docker run -it --rm \
  --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
```

### Option 3: Multi-LLM Container

```bash
# For custom models or unsupported formats
docker run -it --rm \
  --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -e NIM_MODEL_NAME=custom-nemotron \
  -v /path/to/model:/model \
  -p 8000:8000 \
  nvcr.io/nim/meta/llama-3-multi-llm:1.0.0

# Supports: HuggingFace, GGUF, TensorRT-LLM engines
```

## Configuration

### Environment Variables

```bash
# Required
NGC_API_KEY           # NGC authentication

# Optional
NIM_MODEL_NAME        # Override model name
NIM_CACHE_PATH        # Model cache directory
NIM_MAX_BATCH_SIZE    # Maximum batch size
NIM_MAX_SEQ_LEN       # Maximum sequence length
CUDA_VISIBLE_DEVICES  # GPU selection
```

### Docker Options

```bash
docker run -it --rm \
  --gpus '"device=0,1"' \          # Select specific GPUs
  --shm-size=8g \                  # Shared memory
  -e NGC_API_KEY=$NGC_API_KEY \
  -e NIM_MAX_BATCH_SIZE=32 \
  -v /data/cache:/opt/nim/.cache \ # Persistent cache
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
```

## Health Checks

### Readiness Probe

```bash
# Check if server is ready
curl http://localhost:8000/v1/health/ready

# Response when ready:
# {"status": "ready"}
```

### Liveness Probe

```bash
# Check if server is alive
curl http://localhost:8000/v1/health/live

# Response:
# {"status": "alive"}
```

### Model List

```bash
# List available models
curl http://localhost:8000/v1/models

# Response:
# {
#   "object": "list",
#   "data": [
#     {
#       "id": "nvidia/nemotron-3-nano-30b-a3b",
#       "object": "model",
#       "created": 1234567890,
#       "owned_by": "nvidia"
#     }
#   ]
# }
```

## API Endpoints

### Chat Completions

```bash
# Basic chat
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/nemotron-3-nano-30b-a3b",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Explain quantum computing."}
    ],
    "temperature": 0.7,
    "max_tokens": 500
  }'

# Streaming
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/nemotron-3-nano-30b-a3b",
    "messages": [{"role": "user", "content": "Count to 10"}],
    "stream": true
  }'
```

### Completions

```bash
# Base model completion
curl -X POST http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/nemotron-3-nano-30b-a3b",
    "prompt": "Once upon a time",
    "max_tokens": 100,
    "temperature": 0.8
  }'
```

## Python Client

### OpenAI Library

```python
from openai import OpenAI

# Connect to NIM
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-used"  # NIM doesn't require API key
)

# Chat completion
response = client.chat.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is machine learning?"}
    ],
    temperature=0.7,
    max_tokens=500
)

print(response.choices[0].message.content)

# Streaming
stream = client.chat.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    messages=[{"role": "user", "content": "Write a poem"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### Async Client

```python
from openai import AsyncOpenAI
import asyncio

async def main():
    client = AsyncOpenAI(
        base_url="http://localhost:8000/v1",
        api_key="not-used"
    )
    
    response = await client.chat.completions.create(
        model="nvidia/nemotron-3-nano-30b-a3b",
        messages=[{"role": "user", "content": "Hello!"}]
    )
    
    print(response.choices[0].message.content)

asyncio.run(main())
```

## Scaling Deployment

### Multi-GPU

```bash
# Use all GPUs
docker run -it --rm \
  --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0

# Specific GPUs
docker run -it --rm \
  --gpus '"device=0,1,2,3"' \
  -e NGC_API_KEY=$NGC_API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nemotron-nim
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nemotron
  template:
    metadata:
      labels:
        app: nemotron
    spec:
      containers:
      - name: nim
        image: nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
        env:
        - name: NGC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ngc-secret
              key: api-key
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: 1
---
apiVersion: v1
kind: Service
metadata:
  name: nemotron-service
spec:
  selector:
    app: nemotron
  ports:
  - port: 8000
    targetPort: 8000
  type: LoadBalancer
```

### Docker Compose

```yaml
version: '3.8'

services:
  nemotron:
    image: nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
    environment:
      - NGC_API_KEY=${NGC_API_KEY}
    ports:
      - "8000:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

## Performance Optimization

### Batch Size Tuning

```bash
# Increase batch size for throughput
docker run -it --rm \
  --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -e NIM_MAX_BATCH_SIZE=64 \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
```

### Caching

```bash
# Persistent model cache
docker run -it --rm \
  --gpus all \
  -e NGC_API_KEY=$NGC_API_KEY \
  -v /data/nim-cache:/opt/nim/.cache \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
```

### Memory Management

```bash
# Increase shared memory
docker run -it --rm \
  --gpus all \
  --shm-size=16g \
  -e NGC_API_KEY=$NGC_API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/nemotron-3-nano-30b-a3b:1.0.0
```

## Monitoring

### Logs

```bash
# View container logs
docker logs -f <container_id>

# Startup messages indicate readiness:
# INFO: Started server process
# INFO: Application startup complete
# INFO: Uvicorn running on http://0.0.0.0:8000
```

### Metrics

```bash
# Get model info
curl http://localhost:8000/v1/models

# Health status
curl http://localhost:8000/v1/health/ready
curl http://localhost:8000/v1/health/live
```

## Troubleshooting

### Container Won't Start

```bash
# Check GPU availability
nvidia-smi

# Verify Docker has GPU access
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi

# Check NGC authentication
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
```

### Out of Memory

```bash
# Reduce batch size
-e NIM_MAX_BATCH_SIZE=16

# Increase shared memory
--shm-size=32g

# Use fewer GPUs
--gpus '"device=0"'
```

### Slow Inference

```bash
# Check GPU utilization
nvidia-smi

# Increase batch size
-e NIM_MAX_BATCH_SIZE=64

# Use more GPUs
--gpus all
```

## Best Practices

1. **Use persistent cache**: Mount volume for model cache
2. **Health checks**: Implement readiness probes
3. **Resource limits**: Set appropriate GPU allocation
4. **Monitoring**: Track inference latency and throughput
5. **Scaling**: Use Kubernetes for multi-replica deployment
6. **Security**: Rotate NGC API keys regularly
7. **Updates**: Pull latest container versions
8. **Batch size**: Tune for your workload
