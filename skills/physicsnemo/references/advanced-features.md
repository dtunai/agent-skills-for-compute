---
title: Advanced Features
impact: HIGH
tags: active-learning, physics-informer, profiling, logging, metrics, domain-parallel, shard-tensor, curator
---

# Skill: Advanced Features

PhysicsNeMo capabilities beyond basic training: PhysicsInformer gradient methods, active learning, domain parallelism, logging, profiling, metrics, and data curation.

## Quick Pattern

```python
from physicsnemo.utils.physics_informer import PhysicsInformer
from physicsnemo.sym.eq.pdes.navier_stokes import NavierStokes

ns = NavierStokes(nu=0.01, rho=1.0, dim=2, time=False)
informer = PhysicsInformer(
    required_outputs=["continuity", "momentum_x", "momentum_y"],
    equations=ns, grad_method="meshless_fd", device="cuda",
)
residuals = informer.forward({"coordinates": coords, "u": u, "v": v, "p": p})
physics_loss = sum(r.pow(2).mean() for r in residuals.values())
```

## When to Use

- Adding physics loss to data-driven models without full PINNs
- Running active learning loops with external solvers
- Domain decomposition for high-resolution data exceeding single GPU
- Tracking experiments with MLflow or Weights & Biases
- Profiling training bottlenecks
- Evaluating weather/climate model quality

## Step-by-Step Instructions

### 1. PhysicsInformer — 5 Gradient Methods

PhysicsInformer computes PDE residuals from model outputs using different differentiation strategies:

```python
from physicsnemo.utils.physics_informer import PhysicsInformer
from physicsnemo.sym.eq.pdes.navier_stokes import NavierStokes

ns = NavierStokes(nu=0.01, rho=1.0, dim=2, time=False)

# Method 1: Automatic Differentiation (for MLPs, DeepONets)
informer = PhysicsInformer(
    required_outputs=["continuity", "momentum_x", "momentum_y"],
    equations=ns, grad_method="autodiff", device="cuda",
)

# Method 2: Meshless Finite Difference (for point clouds)
informer = PhysicsInformer(
    required_outputs=["continuity", "momentum_x"],
    equations=ns, grad_method="meshless_fd", device="cuda",
)
# Requires stencil points in input dict, fd_dx typically 0.001

# Method 3: Finite Difference (for structured grids)
# Second-order central differences, requires uniform spacing

# Method 4: Spectral Derivatives (for periodic boundaries)
# FFT-based, known artifacts at non-periodic boundaries

# Method 5: Least Squares (for unstructured meshes)
# Requires connectivity via compute_connectivity_tensor

# Usage pattern
residuals = informer.forward({
    "coordinates": coords,   # [N, 2] or [N, 3]
    "u": model_output[:, 0:1],
    "v": model_output[:, 1:2],
    "p": model_output[:, 2:3],
})
# Check required inputs
print(informer.required_inputs)
```

Physics-informed data-driven training:

```python
out = model(node_features, edge_features, graph)
loss_data = torch.nn.functional.l1_loss(out, true_out)
loss_physics = (1 / out.shape[0]) * torch.sum(torch.sum(out, dim=1)).abs()
loss = loss_data + 0.001 * loss_physics
```

### 2. Domain Parallelism with ShardTensor

Split high-resolution data across GPUs when a single sample exceeds GPU memory:

```python
from physicsnemo.distributed import DistributedManager, scatter_tensor
from torch.distributed.tensor import distribute_module
from torch.distributed.tensor.placement_types import Shard, Replicate

dm = DistributedManager()
mesh = dm.initialize_mesh(mesh_shape=(-1,), mesh_dim_names=("domain_parallel",))

# Scatter spatial dimension across GPUs
sharded = scatter_tensor(tensor, 0, mesh, (Shard(2),), requires_grad=True)

# Distribute model (auto-handles halo communication)
dist_model = distribute_module(conv_model, mesh)

# Forward/backward — no manual communication needed
output = dist_model(sharded)
output.mean().backward()
```

Combining domain parallelism + FSDP for 2D parallelism:

```python
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP

mesh = dm.initialize_mesh(
    mesh_shape=(ddp_size, domain_size),
    mesh_dim_names=["ddp", "domain"]
)

# Stage 1: Domain decomposition
model = distribute_module(model, device_mesh=mesh["domain"],
                          partition_fn=partition_model)
# Stage 2: FSDP wrapping
model = FSDP(model, device_mesh=mesh["ddp"], use_orig_params=False)
```

Supported operations: Conv1D/2D/3D, Neighborhood Attention 2D.

### 3. Active Learning Framework

Protocol-based system for iterative model improvement (v25.11+):

```python
from physicsnemo.active_learning.driver import Driver

driver = Driver(
    driver_config=driver_config,
    strategies_config=strategies_config,
    training_config=training_config,
    model=model,
)

# Full loop: train → query → label → evaluate → repeat
driver.run()

# Or single iteration
driver.active_learning_step()

# Resume from checkpoint
driver.load_checkpoint()
```

Custom query strategy:

```python
class UncertaintyQuery:
    __protocol_name__ = "UncertaintyQuery"

    def __init__(self, max_samples):
        self.max_samples = max_samples
        self.driver = None

    def attach(self, driver):
        self.driver = driver

    @property
    def is_attached(self):
        return self.driver is not None

    def sample(self, query_queue, *args, **kwargs):
        # Select most uncertain samples from unlabeled pool
        pool = self.driver.unlabeled_pool
        # Compute uncertainty, enqueue top-k to query_queue
        pass
```

External solver integration for labeling:

```python
class CFDSolverLabeler:
    __is_external_process__ = True
    __provides_fields__ = {"pressure", "velocity"}

    def __init__(self, solver_path, timeout=3600):
        self.solver_path = solver_path
        self.timeout = timeout

    def label(self, queue_to_label, serialize_queue):
        while not queue_to_label.empty():
            sample = queue_to_label.get()
            result = subprocess.run(
                [self.solver_path, "--input", str(sample)],
                timeout=self.timeout, capture_output=True, check=True
            )
            serialize_queue.put(labeled_result)
```

### 4. Logging with MLflow and W&B

```python
from physicsnemo.launch.logging import LaunchLogger, PythonLogger
from physicsnemo.launch.logging.mlflow import initialize_mlflow
from physicsnemo.launch.logging.wandb import initialize_wandb

# Console logging
logger = PythonLogger("main")

# MLflow
initialize_mlflow(experiment_name="darcy_fno", run_name="run_001", mode="offline")
LaunchLogger.initialize(use_mlflow=True)

# Weights & Biases
initialize_wandb(project="physics_ml", name="fno_run", entity="team", mode="offline")
LaunchLogger.initialize(use_wandb=True)

# Usage in training loop
for epoch in range(num_epochs):
    with LaunchLogger("train", epoch=epoch) as log:
        for batch in dataloader:
            loss = training_step(batch)
            log.log_minibatch({"Loss": loss.item()})
        log.log_epoch({"LR": scheduler.get_last_lr()[0]})
```

### 5. Checkpointing with Launch Utils

```python
from physicsnemo.launch.utils import load_checkpoint, save_checkpoint

# Save
save_checkpoint("./checkpoints", models=model,
                optimizer=optimizer, scheduler=scheduler, epoch=epoch)

# Load (returns last epoch)
loaded_epoch = load_checkpoint("./checkpoints", models=model,
                               optimizer=optimizer, scheduler=scheduler, device="cuda")
```

### 6. Profiling

```python
from physicsnemo.utils.profiling import Profiler, profile, annotate

p = Profiler()
p.enable("line_profiler")   # Or "torch" or external nsys
p.initialize()

@profile
def forward(self, x):
    with annotate(domain="forward", color="blue"):
        return self.net(x)

# Output: physicsnemo_profiling_outputs/
```

Three backends: `line_profiler`, PyTorch Profiler (perfetto trace), NVIDIA Nsight Systems.

### 7. ONNX Stream Export

```python
from physicsnemo.deploy.onnx import export_to_onnx_stream, run_onnx_inference

# Export to ONNX byte stream (not file)
onnx_stream = export_to_onnx_stream(model, sample_input)

# Run inference via ONNX Runtime
output = run_onnx_inference(onnx_stream, input_tensor)
```

### 8. Metrics

```python
from physicsnemo.metrics.general.mse import mse, rmse
from physicsnemo.metrics.general.histogram import Histogram
from physicsnemo.metrics.general.crps import CRPS
from physicsnemo.metrics.general.wasserstein import WassersteinDistance

# Weather-specific
from physicsnemo.metrics.climate.acc import ACC
from physicsnemo.metrics.climate.efi import EFI

# Online/streaming metrics
from physicsnemo.metrics.general.reduction import WeightedMean, WeightedVariance

# CRPS (3 methods: kernel, sort, histogram)
crps = CRPS(method="sort")
score = crps(predictions, observations)

# Anomaly Correlation Coefficient (latitude-weighted)
acc = ACC(lat_weights)
score = acc(forecast, analysis, climatology)
```

### 9. CombinedOptimizer

```python
from physicsnemo.optim import CombinedOptimizer

opt = CombinedOptimizer(
    [optimizer_backbone, optimizer_head],
    torch_compile_kwargs={"mode": "reduce-overhead"}
)
```

### 10. Performance Tips

| Technique | Speedup | When to Use |
|-----------|---------|-------------|
| `StaticCaptureTraining` | 1.5-2x | Fixed-shape training |
| `torch.compile` | 1.3-1.5x | End-to-end compilation |
| PyTorch Geometric (over DGL) | 1.5-2x | GNN with >200k nodes |
| Transformer Engine + fp8 | ~25% | Large transformers on Hopper+ |
| RMM shared memory pool | >20% | RAPIDS + PyTorch workflows |
| Warp ball query | varies | Radius-based point search |
| ShardTensor domain parallel | ~6x | High-res data (>2048x2048) |

RMM shared memory pool setup:

```python
import rmm
from rmm.allocators.torch import rmm_torch_allocator
rmm.reinitialize(pool_allocator=True)
torch.cuda.memory.change_current_allocator(rmm_torch_allocator)
```

### 11. torch.compile + External Kernels

Integrate external GPU libraries (cuML, NVIDIA Warp) with `torch.compile` without graph breaks:

```python
import torch

# Step 1: Register external op as custom operator
@torch.library.custom_op("mylib::knn_search", mutates_args=())
def knn_search(points: torch.Tensor, queries: torch.Tensor, k: int) -> torch.Tensor:
    import cuml
    nn_model = cuml.neighbors.NearestNeighbors(n_neighbors=k)
    nn_model.fit(points)
    distances, indices = nn_model.kneighbors(queries)
    return torch.as_tensor(indices, device=points.device)

# Step 2: Register fake (shape propagation for compilation)
@knn_search.register_fake
def knn_search_fake(points, queries, k):
    return torch.empty(queries.shape[0], k, dtype=torch.long, device=queries.device)

# Step 3: Register autograd (if needed for backward pass)
def knn_search_setup(ctx, points, queries, k):
    return knn_search(points, queries, k)

def knn_search_backward(ctx, grad_output):
    return None, None, None  # kNN is non-differentiable

torch.library.register_autograd("mylib::knn_search", knn_search_setup, knn_search_backward)

# Now torch.compile works seamlessly with the external kernel
model = torch.compile(MyModel())
```

Performance bonus — shared RMM memory pool (PyTorch + RAPIDS):

```python
import rmm
from rmm.allocators.torch import rmm_torch_allocator
rmm.reinitialize(pool_allocator=True)
torch.cuda.memory.change_current_allocator(rmm_torch_allocator)
# Eliminates memory allocation overhead between PyTorch and cuML
```

### 12. GPU-Optimized Layers

**Warp Accelerated Ball Query** (`physicsnemo.utils.neighbors`):
- Radius-based point selection for DoMINO stencils
- GPU-accelerated via NVIDIA Warp HashGrid
- 10x+ faster than brute-force PyTorch for point-cloud spatial projection

**TransformerEngine LayerNorm**:
- Drop-in replacement for `torch.nn.LayerNorm`
- ~1/3 reduction in MeshGraphNet training time (up to 200k nodes, 1.2M edges)
- Automatic fallback when TransformerEngine is not installed

**PyG backend** (replaces DGL, ~2x faster):
- Almost halves training iteration latency vs DGL for MeshGraphNet

### 13. DGL → PyG Migration

DGL is deprecated (starting v25.06), PyG is the recommended GNN backend:

```python
# Backend auto-detected by input graph type:
# - dgl.DGLGraph → DGL backend (deprecated)
# - torch_geometric.data.Data → PyG backend (recommended, 1.5-2x faster)

# PyG graph construction
import torch
from torch_geometric.data import Data

edge_index = torch.stack([torch.tensor(src), torch.tensor(dst)], dim=0)
graph = Data(x=node_features, edge_index=edge_index)

# DGL → PyG API mapping
# dgl.graph((src, dst)) → Data(edge_index=torch.stack([src, dst]))
# dgl.save_graphs()     → torch.save()  (incompatible formats)
# dgl.load_graphs()     → torch.load()
# dgl.to_bidirected()   → torch_geometric.utils.to_undirected()
# dgl.add_self_loop()   → torch_geometric.utils.add_self_loops()
```

Existing DGL checkpoints are compatible with PyG backend if input data matches.

## Common Pitfalls

- **PhysicsInformer grad_method mismatch**: Use `"autodiff"` for MLPs, `"meshless_fd"` for point clouds, `"spectral"` only with periodic BCs.
- **ShardTensor unsupported ops**: Only Conv1D/2D/3D and NeighborhoodAttention2D are supported.
- **Active learning `__is_external_process__`**: Label strategies with external solvers must set this flag.
- **ONNX export in Docker v25.11**: Known issue with CUDA 13.X in container — use host export.
- **Multiple CUDA graphs**: Can cause invalid memory access if used in same program.
- **torch.compile graph breaks**: External ops without `@torch.library.custom_op` cause graph breaks. Always register external kernels.
- **DGL data format**: `dgl.save_graphs()` output cannot be read by `torch.load()`. Convert data before migrating to PyG.
- **RMM pool**: Must be initialized before any CUDA allocation to avoid fragmentation.

## Related Skills

- [training-recipes.md](./training-recipes.md) - Basic training patterns
- [distributed-training.md](./distributed-training.md) - DDP setup
- [physics-constraints.md](./physics-constraints.md) - Full PINN workflows
- [deployment.md](./deployment.md) - Model export and inference
- [data-pipelines.md](./data-pipelines.md) - Data curation with Curator
