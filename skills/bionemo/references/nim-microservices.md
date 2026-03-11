---
title: NIM Microservices
impact: HIGH
tags: nim, api, deployment, docker, microservice, rest, production
---

# Skill: NIM Microservices

Deploy and use BioNeMo NIM (NVIDIA Inference Microservices) for production biology AI.

## Quick Command

```bash
# Check health
curl http://localhost:8000/v1/health/ready

# Deploy AlphaFold2 NIM
docker run --gpus all -p 8000:8000 \
  -e NGC_API_KEY=$NGC_API_KEY \
  nvcr.io/nim/nvidia/alphafold2:latest
```

## When to Use

- Deploying biology AI models as REST APIs
- Building production protein analysis pipelines
- Scaling inference across multiple requests
- Integrating AI predictions into lab workflows
- Running multi-model pipelines (predict → design → validate)

## Step-by-Step Instructions

### 1. Available BioNeMo NIMs

| NIM | Container | Endpoint |
|-----|-----------|----------|
| AlphaFold2 | `nvcr.io/nim/nvidia/alphafold2:latest` | `/protein-structure/alphafold2/predict-structure-from-sequence` |
| AlphaFold2-Multimer | `nvcr.io/nim/nvidia/alphafold2-multimer:latest` | `/protein-structure/alphafold2/multimer/predict-structure-from-sequences` |
| RFdiffusion | `nvcr.io/nim/nvidia/rfdiffusion:latest` | `/biology/ipd/rfdiffusion/generate` |
| ProteinMPNN | `nvcr.io/nim/nvidia/proteinmpnn:latest` | `/biology/ipd/proteinmpnn/predict` |
| ESM-2 | `nvcr.io/nim/nvidia/esm2:latest` | `/biology/nvidia/esm2/embedding` |
| DiffDock | `nvcr.io/nim/nvidia/diffdock:latest` | `/molecular-docking/diffdock/generate` |
| Evo2 | `nvcr.io/nim/nvidia/evo2:latest` | `/biology/nvidia/evo2/generate` |

### 2. Deploy a NIM Container

```bash
# Set NGC API key
export NGC_API_KEY=your_ngc_api_key

# Deploy AlphaFold2 on port 8000
docker run -d --name alphafold2 \
  --gpus all \
  -p 8000:8000 \
  -e NGC_API_KEY=$NGC_API_KEY \
  -v /data/alphafold_db:/database \
  nvcr.io/nim/nvidia/alphafold2:latest

# Deploy ProteinMPNN on port 8001
docker run -d --name proteinmpnn \
  --gpus '"device=1"' \
  -p 8001:8000 \
  -e NGC_API_KEY=$NGC_API_KEY \
  nvcr.io/nim/nvidia/proteinmpnn:latest

# Deploy RFdiffusion on port 8002
docker run -d --name rfdiffusion \
  --gpus '"device=2"' \
  -p 8002:8000 \
  -e NGC_API_KEY=$NGC_API_KEY \
  nvcr.io/nim/nvidia/rfdiffusion:latest
```

### 3. Health Check and Readiness

```bash
# Health check (all NIMs)
curl http://localhost:8000/v1/health/ready
# {"status": "ready"}

# Wait for readiness in script
while ! curl -s http://localhost:8000/v1/health/ready | grep -q ready; do
  echo "Waiting for NIM..."
  sleep 5
done
echo "NIM is ready"
```

### 4. Python NIM Client

```python
import requests
import time

class BioNeMoNIM:
    def __init__(self, services: dict):
        """
        services: {"alphafold2": "http://localhost:8000",
                    "proteinmpnn": "http://localhost:8001", ...}
        """
        self.services = services

    def _post(self, service, endpoint, payload):
        url = f"{self.services[service]}{endpoint}"
        response = requests.post(url, json=payload, timeout=300)
        response.raise_for_status()
        return response.json()

    def health_check(self, service):
        url = f"{self.services[service]}/v1/health/ready"
        try:
            resp = requests.get(url, timeout=5)
            return resp.json().get("status") == "ready"
        except:
            return False

    def wait_for_ready(self, service, timeout=300):
        start = time.time()
        while time.time() - start < timeout:
            if self.health_check(service):
                return True
            time.sleep(5)
        raise TimeoutError(f"{service} not ready after {timeout}s")

    def predict_structure(self, sequence):
        return self._post("alphafold2",
            "/protein-structure/alphafold2/predict-structure-from-sequence",
            {"sequence": sequence})

    def predict_multimer(self, sequences, databases=None):
        payload = {"sequences": sequences}
        if databases:
            payload["databases"] = databases
        return self._post("alphafold2-multimer",
            "/protein-structure/alphafold2/multimer/predict-structure-from-sequences",
            payload)

    def generate_structure(self, contigs, target_pdb=None, hotspots=None):
        payload = {"contigs": contigs}
        if target_pdb:
            payload["input_pdb"] = target_pdb
        if hotspots:
            payload["hotspot_res"] = hotspots
        return self._post("rfdiffusion",
            "/biology/ipd/rfdiffusion/generate", payload)

    def design_sequences(self, pdb, n_seqs=8, temp=0.1):
        return self._post("proteinmpnn",
            "/biology/ipd/proteinmpnn/predict",
            {"input_pdb": pdb, "num_seq_per_target": n_seqs,
             "sampling_temp": [temp]})

# Usage
nim = BioNeMoNIM({
    "alphafold2": "http://localhost:8000",
    "rfdiffusion": "http://localhost:8001",
    "proteinmpnn": "http://localhost:8002",
})

# Predict structure
structure = nim.predict_structure("MVLSPADKTNVKA...")
```

### 5. Docker Compose for Multi-NIM

```yaml
version: '3.8'
services:
  alphafold2:
    image: nvcr.io/nim/nvidia/alphafold2:latest
    ports: ["8000:8000"]
    environment:
      NGC_API_KEY: ${NGC_API_KEY}
    volumes:
      - alphafold_db:/database
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
              count: 1

  proteinmpnn:
    image: nvcr.io/nim/nvidia/proteinmpnn:latest
    ports: ["8001:8000"]
    environment:
      NGC_API_KEY: ${NGC_API_KEY}
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
              count: 1

  rfdiffusion:
    image: nvcr.io/nim/nvidia/rfdiffusion:latest
    ports: ["8002:8000"]
    environment:
      NGC_API_KEY: ${NGC_API_KEY}
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]
              count: 1

volumes:
  alphafold_db:
```

```bash
NGC_API_KEY=your_key docker compose up -d
```

### 6. Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alphafold2-nim
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alphafold2
  template:
    metadata:
      labels:
        app: alphafold2
    spec:
      containers:
      - name: alphafold2
        image: nvcr.io/nim/nvidia/alphafold2:latest
        ports:
        - containerPort: 8000
        env:
        - name: NGC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ngc-secret
              key: api-key
        resources:
          limits:
            nvidia.com/gpu: 1
```

## GPU Requirements

| NIM | Minimum GPU | Recommended |
|-----|------------|-------------|
| AlphaFold2 | A100 40GB | A100 80GB |
| ProteinMPNN | T4 16GB | A100 |
| RFdiffusion | A100 40GB | A100 80GB |
| ESM-2 (650M) | A10G 24GB | A100 |
| ESM-2 (3B) | A100 80GB | H100 |
| DiffDock | A100 40GB | A100 |

## Common Pitfalls

- **Missing NGC_API_KEY**: Required to pull NIM containers from NGC registry.
- **Insufficient GPU memory**: Check GPU requirements table above. OOM will crash the container.
- **Database not mounted**: AlphaFold2 needs MSA databases. Mount a volume or let it download (slow).
- **Port conflicts**: Each NIM needs its own port. Map to different host ports.
- **Timeout on first request**: First inference may be slow due to model loading. Set client timeout > 60s.

## Related Skills

- [structure-prediction.md](./structure-prediction.md) - AlphaFold2 usage details
- [protein-design.md](./protein-design.md) - RFdiffusion/ProteinMPNN usage
- [drug-discovery.md](./drug-discovery.md) - Multi-NIM pipelines
