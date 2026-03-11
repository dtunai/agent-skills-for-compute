# Triton Programming Guide Reference

Sources:
- [Programming Guide](https://triton-lang.org/main/programming-guide/chapter-1/introduction.html)

## Design Philosophy

### Block-Based Programming Model

**Traditional CUDA:**
```
Thread-centric programming
- Each thread processes one element
- Explicit thread coordination (syncthreads)
- Manual shared memory management
- Manual memory coalescing
```

**Triton:**
```
Block-centric programming
- Each program instance processes a block of elements
- Automatic thread coordination
- Automatic shared memory optimization
- Automatic memory coalescing
```

### SPMD Model

Single Program, Multiple Data:
- Write one program that operates on a block
- Compiler instantiates multiple copies (grid)
- Each instance identified by program_id
- Automatic parallelization across GPU

## Core Concepts

### Program Instances

```python
@triton.jit
def kernel(ptr, n, BLOCK_SIZE: tl.constexpr):
    # Get unique program ID
    pid = tl.program_id(axis=0)
    
    # Each program processes BLOCK_SIZE elements
    block_start = pid * BLOCK_SIZE
    offsets = block_start + tl.arange(0, BLOCK_SIZE)
```

### Grid Launch

```python
# Calculate grid dimensions
n_elements = 10000
BLOCK_SIZE = 256
grid = (triton.cdiv(n_elements, BLOCK_SIZE),)

# Launch kernel with grid
kernel[grid](ptr, n_elements, BLOCK_SIZE=BLOCK_SIZE)
```

### Constexpr Parameters

```python
# Compile-time constants for optimization
def kernel(ptr, BLOCK_SIZE: tl.constexpr):
    # BLOCK_SIZE known at compile time
    # Enables loop unrolling, better code generation
    offsets = tl.arange(0, BLOCK_SIZE)
```

## Memory Model

### Pointer Arithmetic

```python
# Base pointer + offsets
base_ptr = input_ptr
offsets = tl.arange(0, BLOCK_SIZE)
element_ptrs = base_ptr + offsets

# Load from calculated addresses
data = tl.load(element_ptrs)
```

### Masked Operations

```python
# Guard against out-of-bounds access
mask = offsets < n_elements

# Masked load (use 'other' for invalid addresses)
data = tl.load(ptrs, mask=mask, other=0.0)

# Masked store (only valid addresses written)
tl.store(ptrs, data, mask=mask)
```

### Strided Access

```python
# 2D tensor access
row_idx = tl.program_id(0)
col_offsets = tl.arange(0, BLOCK_SIZE)

# Row stride for row-major layout
row_start = input_ptr + row_idx * stride
element_ptrs = row_start + col_offsets

data = tl.load(element_ptrs, mask=col_offsets < n_cols)
```

## Automatic Optimizations

### Memory Coalescing

Compiler automatically:
- Detects contiguous access patterns
- Generates coalesced memory transactions
- Minimizes memory bandwidth usage

### Shared Memory

Compiler automatically:
- Allocates shared memory for block data
- Inserts synchronization barriers
- Optimizes bank conflicts

### Tensor Core Scheduling

When using `tl.dot`:
- Detects matrix multiply patterns
- Schedules tensor core operations
- Handles data layout conversions
- Maximizes tensor core utilization

### Vectorization

Compiler automatically:
- Generates vector instructions (SIMD)
- Optimizes element-wise operations
- Handles alignment requirements

## Multi-Dimensional Grids

### 2D Grid

```python
@triton.jit
def matmul_kernel(a_ptr, b_ptr, c_ptr, M, N, K, ...):
    # 2D program ID
    pid_m = tl.program_id(0)  # Row blocks
    pid_n = tl.program_id(1)  # Column blocks
    
    # Process block (pid_m, pid_n)
    ...

# Launch 2D grid
grid = (triton.cdiv(M, BLOCK_M), triton.cdiv(N, BLOCK_N))
matmul_kernel[grid](...)
```

### Dynamic Grid

```python
# Grid calculated at runtime
def grid_fn(meta):
    # Access compile-time constants
    BLOCK_SIZE = meta['BLOCK_SIZE']
    return (triton.cdiv(n_elements, BLOCK_SIZE),)

kernel[grid_fn](ptr, n_elements, BLOCK_SIZE=256)
```

## Execution Model

### Warp Execution

- 32 threads per warp (NVIDIA GPUs)
- Warps execute in lockstep
- Compiler manages warp-level parallelism
- `num_warps` controls number of warps per block

### Pipelining

- `num_stages` controls pipeline depth
- Overlaps memory loads with computation
- Hides memory latency
- Requires sufficient registers

### Occupancy

```python
# Control occupancy with num_warps
@triton.jit
def kernel(...):
    pass

# Low occupancy, more registers per thread
kernel[grid](ptr, num_warps=1)

# High occupancy, fewer registers per thread
kernel[grid](ptr, num_warps=8)
```

## Best Practices

### Block Size Selection

1. **Power of 2**: Enables vectorization
2. **Multiple of 32**: Aligns with warp size
3. **Memory constraints**: Balance block size vs shared memory
4. **Compute intensity**: Larger blocks for compute-bound kernels

### Memory Access Patterns

1. **Coalesced access**: Contiguous addresses
2. **Aligned access**: Align to cache lines
3. **Minimize atomics**: Use reductions instead
4. **Avoid bank conflicts**: Careful shared memory indexing

### Optimization Strategy

1. **Start simple**: Get correctness first
2. **Profile**: Identify bottlenecks
3. **Tune block size**: Use auto-tuning
4. **Adjust num_warps/stages**: Balance occupancy vs resources
5. **Validate**: Compare against reference implementation

### Debugging

```python
# Device assertions
tl.device_assert(condition, "Error message")

# Static assertions (compile-time)
tl.static_assert(BLOCK_SIZE % 16 == 0)

# Print (use sparingly)
tl.device_print("value:", x)
```

## Comparison with CUDA

| Aspect | CUDA | Triton |
|--------|------|--------|
| Programming unit | Thread | Block |
| Synchronization | Explicit (__syncthreads) | Automatic |
| Shared memory | Manual allocation | Automatic |
| Memory coalescing | Manual optimization | Automatic |
| Tensor cores | Manual WMMA API | Automatic (tl.dot) |
| Vectorization | Manual (float4, etc) | Automatic |
| Code complexity | High | Low |
| Performance | Excellent (if optimized) | Excellent (automatic) |

## Common Patterns

### Reduction Pattern

```python
@triton.jit
def reduce_kernel(input_ptr, output_ptr, n_elements, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(0)
    
    # Load block
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements
    data = tl.load(input_ptr + offsets, mask=mask, other=0.0)
    
    # Reduce within block
    result = tl.sum(data)
    
    # Store block result
    tl.store(output_ptr + pid, result)
```

### Stencil Pattern

```python
@triton.jit
def stencil_kernel(input_ptr, output_ptr, n, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(0)
    
    # Load with halo
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    
    left = tl.load(input_ptr + offsets - 1, mask=offsets > 0)
    center = tl.load(input_ptr + offsets)
    right = tl.load(input_ptr + offsets + 1, mask=offsets < n - 1)
    
    # Compute stencil
    result = 0.25 * left + 0.5 * center + 0.25 * right
    
    tl.store(output_ptr + offsets, result, mask=offsets < n)
```

### Transpose Pattern

```python
@triton.jit
def transpose_kernel(input_ptr, output_ptr, M, N, BLOCK_SIZE: tl.constexpr):
    pid_m = tl.program_id(0)
    pid_n = tl.program_id(1)
    
    # Load input block (row-major)
    offs_m = pid_m * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    offs_n = pid_n * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    
    input_ptrs = input_ptr + offs_m[:, None] * N + offs_n[None, :]
    block = tl.load(input_ptrs)
    
    # Store transposed (column-major)
    output_ptrs = output_ptr + offs_n[:, None] * M + offs_m[None, :]
    tl.store(output_ptrs, tl.trans(block))
```
