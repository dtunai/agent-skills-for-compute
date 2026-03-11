---
name: cutile
description: "NVIDIA CuTile Python — tile-based GPU programming DSL. Write kernels using tiles instead of threads, with automatic tensor core and TMA optimization. Blackwell GPUs (compute 10.x/12.x)."
license: MIT
metadata:
  author: Agent Cluster
  tags: [cutile, cuda, gpu, nvidia, tile, tensor-cores, tma, blackwell, kernels, python]
---

# NVIDIA CuTile Python

Tile-based GPU programming DSL in Python. Write kernels that operate on tiles (multidimensional array chunks) instead of individual threads. The compiler automatically leverages tensor cores, TMA, and hardware features. Portable across NVIDIA Blackwell architectures.

**Official Sources:**
- [Documentation](https://docs.nvidia.com/cuda/cutile-python/)
- [GitHub](https://github.com/NVIDIA/cutile-python)
- [TileGym (tutorials/benchmarks)](https://github.com/NVIDIA/TileGym)
- [Blog: Simplify GPU Programming](https://developer.nvidia.com/blog/simplify-gpu-programming-with-nvidia-cuda-tile-in-python/)
- [Blog: Tuning Flash Attention](https://developer.nvidia.com/blog/tuning-flash-attention-for-peak-performance-in-nvidia-cuda-tile/)

## Requirements

- GPU: Compute capability 10.x or 12.x (Blackwell)
- NVIDIA Driver: r580+
- CUDA Toolkit: 13.1+
- Python: 3.10-3.13

## Quick Start

```bash
pip install cuda-tile
pip install cupy-cuda13x  # or torch for PyTorch integration
```

```python
import cuda.tile as ct
import cupy

TILE_SIZE = 16

@ct.kernel
def vector_add(a, b, result):
    pid = ct.bid(0)
    a_tile = ct.load(a, index=(pid,), shape=(TILE_SIZE,))
    b_tile = ct.load(b, index=(pid,), shape=(TILE_SIZE,))
    ct.store(result, index=(pid,), tile=a_tile + b_tile)

# Host code
n = 1024
a = cupy.random.rand(n).astype(cupy.float32)
b = cupy.random.rand(n).astype(cupy.float32)
c = cupy.empty(n, dtype=cupy.float32)
grid = (ct.cdiv(n, TILE_SIZE), 1, 1)
ct.launch(cupy.cuda.get_current_stream(), grid, vector_add, (a, b, c))
```

## Core Concepts

### Kernels and Launch

```python
@ct.kernel
def my_kernel(a, b, c, TILE: ct.Constant[int]):
    pid = ct.bid(0)  # block index along axis 0
    # ... tile operations ...

# Launch: (stream, grid, kernel, args)
ct.launch(stream, (num_blocks_x, num_blocks_y, num_blocks_z), my_kernel, (a, b, c, 128))
```

- `@ct.kernel` marks GPU entry points (cannot call directly)
- `ct.bid(axis)` returns block index (0, 1, or 2) as int32
- `ct.num_blocks(axis)` returns grid size along axis
- `ct.Constant[int]` marks compile-time constant parameters (generates distinct kernel per value)
- Grid is a 3-tuple `(x, y, z)`

### Arrays vs Tiles

| | Arrays | Tiles |
|---|---|---|
| **Location** | Global GPU memory | Kernel-local (no defined storage) |
| **Mutability** | Mutable (via store) | Immutable |
| **Creation** | Host only (PyTorch/CuPy) | Kernel only (load, factory) |
| **Shape** | Any | Powers of 2, compile-time constant |
| **Operations** | load/store only | Full compute (arithmetic, matmul, reduce...) |

### Dtypes

`bool_`, `uint8/16/32/64`, `int8/16/32/64`, `float16/32/64`, `bfloat16`, `tfloat32`, `float8_e4m3fn`, `float8_e5m2`

### Standard Pattern: load -> compute -> store

```python
@ct.kernel
def elementwise(x, y, out, N: ct.Constant[int]):
    pid = ct.bid(0)
    x_tile = ct.load(x, index=(pid,), shape=(N,))
    y_tile = ct.load(y, index=(pid,), shape=(N,))
    result = x_tile * y_tile + ct.ones((N,), ct.float32)
    ct.store(out, index=(pid,), tile=result)
```

## Load and Store

```python
# Structured load: partitions array into tile grid, loads tile at index
tile = ct.load(array, index=(pid_row, pid_col), shape=(TM, TN))

# Store: writes tile back (out-of-bounds ignored)
ct.store(array, index=(pid_row, pid_col), tile=result)

# Gather/scatter: indirect indexed access
vals = ct.gather(array, indices, mask=mask, padding_value=0)
ct.scatter(array, indices, values, mask=mask)
```

**Parameters:**
- `order`: `'C'` (default), `'F'` (reversed axes), or tuple of axis indices
- `padding_mode`: `ZERO`, `NEG_ZERO`, `NAN`, `POS_INF`, `NEG_INF`, `UNDETERMINED` (default)
- `latency`: int 1-10 (1=low DRAM traffic, 10=high) — performance hint
- `allow_tma`: bool (default True) — enable Tensor Memory Accelerator

## Factory Functions

```python
ct.zeros((M, N), dtype=ct.float32)
ct.ones((M, N), dtype=ct.float16)
ct.full((M, N), fill_value=3.14, dtype=ct.float32)
ct.arange(N, dtype=ct.int32)  # [0, 1, ..., N-1]
```

## Matrix Operations

```python
# Standard matmul
c = ct.matmul(a, b)  # or a @ b

# Fused multiply-accumulate (preserves acc dtype)
acc = ct.zeros((TM, TN), dtype=ct.float32)
acc = ct.mma(a_tile, b_tile, acc)  # (a @ b) + acc
```

**MMA input/accumulator types:**

| Input | Accumulator |
|-------|------------|
| f16 | f16 or f32 |
| bf16 | f32 |
| f32 | f32 |
| tf32 | f32 |
| f8e4m3fn/f8e5m2 | f16 or f32 |
| i8/u8 | i32 |

**Tiled matmul with K-loop:**
```python
@ct.kernel
def matmul(A, B, C, TM: ct.Constant[int], TN: ct.Constant[int], TK: ct.Constant[int]):
    bid_m, bid_n = ct.bid(0), ct.bid(1)
    acc = ct.zeros((TM, TN), dtype=ct.float32)
    for k in range(ct.num_tiles(A, axis=1, shape=(TM, TK))):
        a = ct.load(A, (bid_m, k), (TM, TK))
        b = ct.load(B, (k, bid_n), (TK, TN))
        acc = ct.mma(a, b, acc)
    ct.store(C, (bid_m, bid_n), tile=acc.astype(ct.float16))
```

## Reductions and Scans

```python
# Reductions along axis
ct.sum(tile, axis=0)
ct.max(tile, axis=1)
ct.min(tile, axis=0)
ct.prod(tile, axis=1)
ct.argmax(tile, axis=0)
ct.argmin(tile, axis=1)

# Custom reduction
ct.reduce(tile, axis=0, func=lambda a, b: ct.maximum(a, b), identity=float('-inf'))

# Prefix scans
ct.cumsum(tile, axis=0)
ct.cumprod(tile, axis=0, reverse=True)
ct.scan(tile, axis=0, func=lambda a, b: a + b, identity=0)
```

## Math Functions

```python
ct.exp(x)      ct.exp2(x)     ct.log(x)      ct.log2(x)
ct.sqrt(x)     ct.rsqrt(x)    ct.pow(x, n)
ct.sin(x)      ct.cos(x)      ct.tan(x)
ct.sinh(x)     ct.cosh(x)     ct.tanh(x)
ct.floor(x)    ct.ceil(x)     ct.abs(x)
ct.isnan(x)    ct.negative(x)
ct.minimum(a, b)  ct.maximum(a, b)
ct.cdiv(a, b)  # ceiling division
```

## Shape Operations

```python
ct.reshape(tile, new_shape)
ct.permute(tile, axes)
ct.transpose(tile, axis0, axis1)
ct.expand_dims(tile, axis)
ct.broadcast_to(tile, shape)
ct.cat(tile_a, tile_b, axis)  # concatenate (doubles one dimension)
tile.extract(index, shape)     # extract sub-tile
```

## Selection and Masking

```python
ct.where(condition, x, y)  # element-wise conditional

# Boundary masking
mask = ct.arange(TILE, dtype=ct.int32) < actual_size
masked = ct.where(mask, values, ct.zeros((TILE,), ct.float32))
```

## Atomic Operations

```python
# All return pre-update value
ct.atomic_add(array, indices, update)
ct.atomic_max(array, indices, update)
ct.atomic_min(array, indices, update)
ct.atomic_and(array, indices, update)
ct.atomic_or(array, indices, update)
ct.atomic_xor(array, indices, update)
ct.atomic_cas(array, indices, expected, desired)  # compare-and-swap
ct.atomic_xchg(array, indices, update)             # exchange
```

Parameters: `memory_order` (default `ACQ_REL`), `memory_scope` (default `DEVICE`), `check_bounds`

## Kernel Configuration

```python
@ct.kernel(num_ctas=8, occupancy=4, opt_level=3)
def tuned_kernel(x): ...

# Architecture-specific tuning
from cuda.tile import ByTarget

@ct.kernel(num_ctas=ByTarget(sm_100=8, sm_120=4, default=2))
def arch_aware_kernel(x): ...
```

- `num_ctas`: Power of 2 in [1, 16] — CTAs in Cooperative Group Array
- `occupancy`: [1, 32] — active CTAs per SM
- `opt_level`: [0, 3] — compiler optimization level

## Performance Patterns

**Persistent kernel (loop over tiles):**
```python
@ct.kernel
def persistent(x, out, TILE: ct.Constant[int]):
    bid = ct.bid(0)
    stride = ct.num_blocks(0)
    num = ct.num_tiles(x, axis=0, shape=(TILE,))
    while bid < num:
        tile = ct.load(x, (bid,), (TILE,))
        ct.store(out, (bid,), tile=ct.exp(tile))
        bid += stride
```

**FP32 accumulation with lower-precision I/O:**
```python
acc = ct.zeros((TM, TN), dtype=ct.float32)
# ... accumulate in fp32 ...
ct.store(out, idx, tile=acc.astype(ct.float16))
```

**TF32 for tensor cores on FP32 data:**
```python
a_tf32 = ct.astype(a_tile, ct.tfloat32)  # enables tensor core path
```

**Fast math flags:**
```python
ct.cumsum(tile, axis=0, rounding_mode=ct.RoundingMode.APPROX, flush_to_zero=True)
```

## Memory Model

- `ct.MemoryOrder`: `RELAXED`, `ACQUIRE`, `RELEASE`, `ACQ_REL`
- `ct.MemoryScope`: `BLOCK`, `DEVICE`, `SYS`

**Atomic lock pattern:**
```python
while ct.atomic_cas(locks, (idx,), 0, 1, memory_order=ct.ACQUIRE) != 0:
    pass
# critical section
ct.atomic_xchg(locks, (idx,), 0, memory_order=ct.RELEASE)
```

## Integration

**PyTorch:** Pass tensors directly as kernel args. Use `torch.cuda.current_stream()`.
**CuPy:** Pass arrays directly. Use `cupy.cuda.get_current_stream()`.

## Metaprogramming

```python
ct.static_assert(TILE % 2 == 0, "TILE must be even")
ct.static_eval(TILE * 2)  # compile-time evaluation
for i in ct.static_iter(range(4)):  # compile-time unrolling
    ...
```

## Debugging

```python
ct.printf("value: %f\n", tile.item())
ct.print(tile)
ct.assert_(tile > 0)  # validates all elements
```

**Environment variables:**
- `CUDA_TILE_ENABLE_CRASH_DUMP=1` — generate IR for bug reports
- `CUDA_TILE_LOGS=CUTILEIR` — output IR to stderr
- `CUDA_TILE_CACHE_DIR` — bytecode cache (default `~/.cache/cutile-python`)

## Comparison with Triton

| | CuTile | Triton |
|---|---|---|
| **Hardware** | Blackwell only (10.x/12.x) | Ampere, Hopper, Blackwell+ |
| **Vendor** | NVIDIA native | OpenAI (open source) |
| **TMA** | Automatic | Manual |
| **Tensor cores** | Automatic via `mma` | Via `tl.dot` |
| **Architecture tuning** | `ByTarget`, `num_ctas` | `num_warps`, `num_stages` |
| **Tile IR** | Native | Optional backend |
