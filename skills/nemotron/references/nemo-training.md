# NeMo Framework Training Reference

Sources:
- [Nemotron — NVIDIA NeMo Framework](https://docs.nvidia.com/nemo-framework/user-guide/25.09/llms/nemotron.html)
- [NeMo 2.0 Documentation](https://docs.nvidia.com/nemo-framework/user-guide/latest/)

## Overview

NeMo Framework provides pretraining recipes for Nemotron models using NeMo 2.0 and NeMo-Run, supporting distributed training across multiple GPUs and nodes.

## Available Models

### Nemotron 3 (NeMo 2.0)

```python
from nemo.collections import llm

# 4B parameter model
llm.nemotron3_4b.pretrain_recipe()

# 8B parameter model
llm.nemotron3_8b.pretrain_recipe()
```

### Nemotron 4 (NeMo 2.0)

```python
# 15B parameter models
llm.nemotron4_15b.pretrain_recipe()       # Base 16k context
llm.nemotron4_15b_16k.pretrain_recipe()   # Explicit 16k
llm.nemotron4_15b_64k.pretrain_recipe()   # Extended 64k

# 22B parameter models
llm.nemotron4_22b.pretrain_recipe()       # Base 16k context
llm.nemotron4_22b_16k.pretrain_recipe()   # Explicit 16k
llm.nemotron4_22b_64k.pretrain_recipe()   # Extended 64k

# 340B parameter model
llm.nemotron4_340b.pretrain_recipe()
```

## Basic Training Recipe

```python
from nemo.collections import llm
from nemo import lightning as nl
import nemo_run as run

# Create pretraining recipe
pretrain = llm.nemotron3_8b.pretrain_recipe(
    name="nemotron3_8b_pretraining",
    dir="/workspace/checkpoints",
    num_nodes=2,
    num_gpus_per_node=8
)

# Replace MockDataModule with real data
from your_data import CustomDataModule
pretrain.data = CustomDataModule(
    seq_length=2048,
    global_batch_size=256,
    micro_batch_size=2
)

# Execute training
executor = run.LocalExecutor()
run.run(pretrain, executor=executor)
```

## Recipe Configuration

### Using run.Partial

```python
import nemo_run as run
from nemo.collections import llm

# Recipes use run.Partial for lazy configuration
pretrain = llm.nemotron3_8b.pretrain_recipe()

# Inspect configuration
print(pretrain.model)
print(pretrain.trainer)
print(pretrain.data)
print(pretrain.optim)

# Modify configuration
pretrain.trainer.max_steps = 100000
pretrain.optim.lr = 3e-4
```

### Recipe Parameters

```python
pretrain = llm.nemotron3_8b.pretrain_recipe(
    name="my_nemotron_training",           # Experiment name
    dir="/path/to/checkpoints",            # Checkpoint directory
    num_nodes=4,                           # Number of nodes
    num_gpus_per_node=8,                   # GPUs per node
    performance_mode=True,                 # Optimize for speed
    tensor_parallelism=2,                  # TP degree
    pipeline_parallelism=1,                # PP degree
    context_parallelism=1,                 # CP degree
)
```

## Data Configuration

### Custom Data Module

```python
from nemo.collections.nlp.data.language_modeling.megatron_gpt_sft_dataset import GPTSFTDataset
from torch.utils.data import DataLoader

class CustomDataModule:
    def __init__(self, seq_length=2048, global_batch_size=256, micro_batch_size=2):
        self.seq_length = seq_length
        self.global_batch_size = global_batch_size
        self.micro_batch_size = micro_batch_size
    
    def setup(self, stage=None):
        # Load your training data
        self.train_dataset = GPTSFTDataset(
            file_path="/data/train.jsonl",
            tokenizer=self.tokenizer,
            max_seq_length=self.seq_length
        )
    
    def train_dataloader(self):
        return DataLoader(
            self.train_dataset,
            batch_size=self.micro_batch_size,
            num_workers=8,
            pin_memory=True
        )

# Use in recipe
pretrain.data = CustomDataModule()
```

### Data Format

```jsonl
{"text": "This is a training example for pretraining."}
{"text": "Another example with longer text..."}
{"text": "Third example demonstrating the format."}
```

## Distributed Training

### Tensor Parallelism

```python
# Split model across GPUs within a node
pretrain = llm.nemotron4_22b.pretrain_recipe(
    num_nodes=1,
    num_gpus_per_node=8,
    tensor_parallelism=8  # Model split across 8 GPUs
)
```

### Pipeline Parallelism

```python
# Split model layers across GPUs
pretrain = llm.nemotron4_340b.pretrain_recipe(
    num_nodes=8,
    num_gpus_per_node=8,
    tensor_parallelism=8,
    pipeline_parallelism=8  # Pipeline stages
)
```

### Data Parallelism

```python
# Automatic data parallelism
# DP degree = total_gpus / (TP * PP * CP)
pretrain = llm.nemotron4_22b.pretrain_recipe(
    num_nodes=4,
    num_gpus_per_node=8,  # 32 total GPUs
    tensor_parallelism=2,   # TP=2
    pipeline_parallelism=1  # PP=1
    # DP = 32 / (2 * 1 * 1) = 16
)
```

### Context Parallelism

```python
# For long sequences
pretrain = llm.nemotron4_15b_64k.pretrain_recipe(
    num_nodes=2,
    num_gpus_per_node=8,
    context_parallelism=4  # Split long context
)
```

## Execution Options

### Local Execution

```python
# Local executor
executor = run.LocalExecutor()
run.run(pretrain, executor=executor)
```

### Direct Execution

```python
# Run directly in current process
run.run(pretrain, direct=True)
```

### SLURM Execution

```python
# SLURM cluster
executor = run.SlurmExecutor(
    account="your_account",
    partition="gpu",
    nodes=4,
    ntasks_per_node=8,
    gpus_per_node=8,
    mem="480GB",
    time="48:00:00"
)

run.run(pretrain, executor=executor)
```

## Optimization

### Learning Rate Schedule

```python
from nemo.lightning.pytorch.optim import CosineAnnealingScheduler

pretrain.optim.lr = 3e-4
pretrain.optim.sched = CosineAnnealingScheduler(
    warmup_steps=2000,
    constant_steps=10000,
    min_lr=3e-5
)
```

### Mixed Precision

```python
# Automatic mixed precision (default)
pretrain.trainer.precision = "bf16-mixed"

# Options: "32", "16-mixed", "bf16-mixed"
```

### Gradient Accumulation

```python
# Increase effective batch size
pretrain.trainer.accumulate_grad_batches = 4
```

### Activation Checkpointing

```python
# Reduce memory usage
pretrain.model.activations_checkpoint_granularity = "selective"
# Options: "full", "selective", None
```

## Checkpointing

### Save Checkpoints

```python
from nemo.lightning.pytorch.callbacks import ModelCheckpoint

pretrain.trainer.callbacks = [
    ModelCheckpoint(
        dirpath="/workspace/checkpoints",
        filename="nemotron-{epoch:02d}-{step}",
        every_n_train_steps=1000,
        save_top_k=5,
        monitor="val_loss"
    )
]
```

### Resume Training

```python
# Resume from checkpoint
pretrain = llm.nemotron3_8b.pretrain_recipe(
    dir="/workspace/checkpoints",
    resume_if_exists=True  # Auto-resume from latest
)

# Or specify checkpoint
pretrain.trainer.ckpt_path = "/workspace/checkpoints/nemotron-epoch=01-step=5000.ckpt"
```

## Monitoring

### Logging

```python
from nemo.lightning.pytorch.callbacks import NeMoLogger

pretrain.log = NeMoLogger(
    name="nemotron_training",
    dir="/workspace/logs",
    log_dir="/workspace/tensorboard",
    use_stdout=True
)
```

### TensorBoard

```bash
# Launch TensorBoard
tensorboard --logdir=/workspace/tensorboard --port=6006
```

### Weights & Biases

```python
from pytorch_lightning.loggers import WandbLogger

pretrain.trainer.logger = WandbLogger(
    project="nemotron-training",
    name="nemotron3-8b-run1"
)
```

## Fine-Tuning

### Supervised Fine-Tuning (SFT)

```python
from nemo.collections import llm

# Load pretrained model
model = llm.nemotron3_8b.model()

# SFT recipe
sft = llm.nemotron3_8b.finetune_recipe(
    name="nemotron3_8b_sft",
    dir="/workspace/sft_checkpoints",
    num_nodes=1,
    num_gpus_per_node=8,
    peft_scheme="lora"  # LoRA, None for full fine-tuning
)

# Custom SFT data
sft.data = CustomSFTDataModule()

# Execute
run.run(sft, executor=run.LocalExecutor())
```

### LoRA Configuration

```python
from nemo.collections.nlp.modules.common.megatron.adapters.parallel_adapters import ParallelLinearAdapter

sft.peft = ParallelLinearAdapter(
    in_features=4096,
    out_features=4096,
    rank=16,
    alpha=32,
    dropout=0.1
)
```

## Evaluation

### Validation During Training

```python
pretrain.data.validation_ds = ValidationDataset()
pretrain.trainer.val_check_interval = 1000  # Validate every 1000 steps
pretrain.trainer.limit_val_batches = 100    # Number of validation batches
```

### Benchmarking

```python
from nemo.collections.nlp.models.language_modeling.megatron_gpt_model import MegatronGPTModel

# Load checkpoint
model = MegatronGPTModel.restore_from("/workspace/checkpoints/nemotron.nemo")

# Run evaluation
from nemo.collections.nlp.modules.common.megatron.megatron_utils import (
    compute_model_parallel_rank,
    get_ltor_masks_and_position_ids
)

# Evaluate on benchmark
results = model.generate(
    inputs=["Prompt 1", "Prompt 2"],
    length_params={
        "max_length": 512,
        "min_length": 1
    }
)
```

## Best Practices

### Training

1. **Start small**: Test on single GPU before scaling
2. **Data quality**: Use high-quality, diverse data
3. **Checkpointing**: Save frequently
4. **Monitoring**: Track loss, learning rate, throughput
5. **Validation**: Regular validation during training

### Scaling

1. **Profile first**: Identify bottlenecks
2. **Optimize parallelism**: Balance TP, PP, DP
3. **Batch size**: Maximize GPU utilization
4. **Mixed precision**: Use bf16 for memory/speed
5. **Gradient accumulation**: Increase effective batch size

### Resources

1. **Memory**: 2GB per billion parameters (approx)
2. **Storage**: Plan for checkpoints (model size × 3)
3. **Network**: High-bandwidth for multi-node
4. **GPUs**: A100/H100 recommended

## Common Patterns

### Full Pretraining Pipeline

```python
# 1. Prepare data
data = CustomDataModule(
    train_path="/data/train.jsonl",
    val_path="/data/val.jsonl",
    seq_length=2048,
    global_batch_size=512
)

# 2. Configure training
pretrain = llm.nemotron3_8b.pretrain_recipe(
    name="nemotron3_8b_full",
    dir="/workspace/checkpoints",
    num_nodes=4,
    num_gpus_per_node=8,
    tensor_parallelism=2
)

pretrain.data = data
pretrain.trainer.max_steps = 500000
pretrain.optim.lr = 3e-4

# 3. Execute
executor = run.SlurmExecutor(
    account="ml_team",
    partition="gpu",
    nodes=4,
    gpus_per_node=8,
    time="72:00:00"
)

run.run(pretrain, executor=executor)
```
