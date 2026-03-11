---
name: triton
description: "Triton — Python-like language for writing high-performance GPU kernels with automatic optimization, block-based programming, tensor cores"
license: MIT
metadata:
  author: Agent Cluster
  tags: [triton, gpu, cuda, kernels, compiler, tensor-cores, performance, openai, neural-networks]
---

# Triton Language Skill

Python-like language and compiler for writing high-performance GPU kernels with automatic optimization, eliminating the need for manual CUDA programming.

**Official Sources:**
- [Triton Documentation](https://triton-lang.org/)
- [Programming Guide](https://triton-lang.org/main/programming-guide/chapter-1/introduction.html)
- [Tutorials](https://triton-lang.org/main/getting-started/tutorials/index.html)
- [API Reference](https://triton-lang.org/main/python-api/triton.language.html)
- [GitHub](https://github.com/triton-lang/triton)

## What is Triton?

**Definition:**
> "Open-source Python-like programming language which enables researchers with no CUDA experience to write highly efficient GPU code."

**Key Innovation:**
- **Block-based programming**: Work with blocks of data, not individual threads
- **Automatic optimization**: Compiler handles memory coalescing, shared memory, tensor cores
- **Python-like syntax**: Familiar to ML researchers
- **Performance**: Matches hand-optimized CUDA and cuBLAS

## Quick Start

### Installation

```bash
# Install from PyPI
pip install triton

# Or from source
git clone https://github.com/triton-lang/triton.git
cd triton/python
pip install -e .
```

### Vector Addition Example

```python
import torch
import triton
import triton.language as tl

@triton.jit
def add_kernel(
    x_ptr,  # Pointer to first input
    y_ptr,  # Pointer to second input
    output_ptr,  # Pointer to output
    n_elements,  # Size of vectors
    BLOCK_SIZE: tl.constexpr,  # Elements per block
):
    # Identify which program instance this is
    pid = tl.program_id(axis=0)

    # Compute this program's data range
    block_start = pid * BLOCK_SIZE
    offsets = block_start + tl.arange(0, BLOCK_SIZE)

    # Create mask to guard memory operations
    mask = offsets < n_elements

    # Load data
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)

    # Compute
    output = x + y

    # Store result
    tl.store(output_ptr + offsets, output, mask=mask)

# Launch kernel
def add(x: torch.Tensor, y: torch.Tensor):
    output = torch.empty_like(x)
    n_elements = output.numel()

    # Grid: number of parallel programs
    grid = lambda meta: (triton.cdiv(n_elements, meta['BLOCK_SIZE']),)

    # Launch with grid[...](args)
    add_kernel[grid](x, y, output, n_elements, BLOCK_SIZE=1024)

    return output

# Usage
x = torch.randn(98432, device='cuda')
y = torch.randn(98432, device='cuda')
result = add(x, y)
```

## Matrix Multiplication

```python
@triton.jit
def matmul_kernel(
    a_ptr, b_ptr, c_ptr,
    M, N, K,
    stride_am, stride_ak,
    stride_bk, stride_bn,
    stride_cm, stride_cn,
    BLOCK_SIZE_M: tl.constexpr,
    BLOCK_SIZE_N: tl.constexpr,
    BLOCK_SIZE_K: tl.constexpr,
    GROUP_SIZE_M: tl.constexpr,
):
    # Program ID
    pid = tl.program_id(axis=0)
    num_pid_m = tl.cdiv(M, BLOCK_SIZE_M)
    num_pid_n = tl.cdiv(N, BLOCK_SIZE_N)

    # Reorder programs for L2 cache locality
    num_pid_in_group = GROUP_SIZE_M * num_pid_n
    group_id = pid // num_pid_in_group
    first_pid_m = group_id * GROUP_SIZE_M
    group_size_m = min(num_pid_m - first_pid_m, GROUP_SIZE_M)
    pid_m = first_pid_m + (pid % group_size_m)
    pid_n = (pid % num_pid_in_group) // group_size_m

    # Compute block offsets
    offs_am = (pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)) % M
    offs_bn = (pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)) % N
    offs_k = tl.arange(0, BLOCK_SIZE_K)

    # Initialize pointers
    a_ptrs = a_ptr + (offs_am[:, None] * stride_am + offs_k[None, :] * stride_ak)
    b_ptrs = b_ptr + (offs_k[:, None] * stride_bk + offs_bn[None, :] * stride_bn)

    # Accumulator
    accumulator = tl.zeros((BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32)

    # Loop over K dimension
    for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
        # Load blocks
        a = tl.load(a_ptrs, mask=offs_k[None, :] < K - k * BLOCK_SIZE_K, other=0.0)
        b = tl.load(b_ptrs, mask=offs_k[:, None] < K - k * BLOCK_SIZE_K, other=0.0)

        # Matrix multiply
        accumulator += tl.dot(a, b)

        # Advance pointers
        a_ptrs += BLOCK_SIZE_K * stride_ak
        b_ptrs += BLOCK_SIZE_K * stride_bk

    # Convert and store
    c = accumulator.to(tl.float16)

    offs_cm = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
    offs_cn = pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)
    c_ptrs = c_ptr + stride_cm * offs_cm[:, None] + stride_cn * offs_cn[None, :]
    c_mask = (offs_cm[:, None] < M) & (offs_cn[None, :] < N)
    tl.store(c_ptrs, c, mask=c_mask)
```

## Fused Softmax

```python
@triton.jit
def softmax_kernel(
    output_ptr, input_ptr,
    input_row_stride,
    output_row_stride,
    n_cols,
    BLOCK_SIZE: tl.constexpr,
):
    # Row index
    row_idx = tl.program_id(0)

    # Row start pointer
    row_start_ptr = input_ptr + row_idx * input_row_stride

    # Column offsets
    col_offsets = tl.arange(0, BLOCK_SIZE)
    input_ptrs = row_start_ptr + col_offsets

    # Load row
    mask = col_offsets < n_cols
    row = tl.load(input_ptrs, mask=mask, other=-float('inf'))

    # Subtract max for numerical stability
    row_minus_max = row - tl.max(row, axis=0)

    # Compute exponentials
    numerator = tl.exp(row_minus_max)

    # Normalize
    denominator = tl.sum(numerator, axis=0)
    softmax_output = numerator / denominator

    # Store result
    output_row_start_ptr = output_ptr + row_idx * output_row_stride
    output_ptrs = output_row_start_ptr + col_offsets
    tl.store(output_ptrs, softmax_output, mask=mask)
```

## Core Language Features

### Memory Operations

```python
# Load from memory
x = tl.load(ptr + offsets, mask=mask, other=0.0)

# Store to memory
tl.store(ptr + offsets, values, mask=mask)

# Atomic operations
tl.atomic_add(ptr + offsets, values)
tl.atomic_max(ptr + offsets, values)
tl.atomic_cas(ptr + offset, cmp, val)
```

### Math Operations

```python
# Element-wise
y = tl.exp(x)
y = tl.sqrt(x)
y = tl.sin(x)
y = tl.log(x)

# Reductions
max_val = tl.max(x, axis=0)
sum_val = tl.sum(x, axis=0)
min_val = tl.min(x, axis=0)

# Linear algebra
C = tl.dot(A, B)  # Matrix multiply (uses tensor cores)
```

### Control Flow

```python
# Static assertions (compile-time)
tl.static_assert(BLOCK_SIZE % 16 == 0)

# Runtime assertions
tl.device_assert(x > 0, "x must be positive")

# Conditionals
result = tl.where(condition, true_val, false_val)
```

## Auto-Tuning

```python
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_SIZE_M': 128, 'BLOCK_SIZE_N': 256, 'BLOCK_SIZE_K': 64, 'GROUP_SIZE_M': 8}, num_stages=3, num_warps=8),
        triton.Config({'BLOCK_SIZE_M': 64, 'BLOCK_SIZE_N': 256, 'BLOCK_SIZE_K': 32, 'GROUP_SIZE_M': 8}, num_stages=4, num_warps=4),
        triton.Config({'BLOCK_SIZE_M': 128, 'BLOCK_SIZE_N': 128, 'BLOCK_SIZE_K': 32, 'GROUP_SIZE_M': 4}, num_stages=4, num_warps=4),
        # ... more configs
    ],
    key=['M', 'N', 'K'],
)
@triton.jit
def matmul_kernel(...):
    # Kernel implementation
    pass
```

## Programming Model

### Block-Based Paradigm

**CUDA:** Scalar Program, Blocked Threads
```
Each thread processes one element
Programmer manages thread coordination
```

**Triton:** Blocked Program, Scalar Threads
```
Each program processes a block of elements
Compiler manages thread coordination
```

### Automatic Optimizations

Triton compiler automatically:
- **Memory coalescing**: Efficient memory access patterns
- **Shared memory management**: Allocation and synchronization
- **Tensor core scheduling**: Automatic use of specialized hardware
- **Thread swizzling**: Optimize warp execution
- **Vectorization**: SIMD instruction generation
- **Asynchronous copy**: Overlapped memory transfers

## Data Types

```python
# Floating point
tl.float16
tl.float32
tl.bfloat16
tl.float8_e5m2  # FP8

# Integer
tl.int8
tl.int32
tl.int64

# Pointer types
tl.pointer_type(tl.float32)
```

## Performance Tips

### Memory Access

1. **Coalescing**: Access contiguous memory
2. **Masking**: Use masks for irregular bounds
3. **Block size**: Power-of-two for efficiency
4. **Alignment**: Align data to cache lines

### Computation

1. **Tensor cores**: Use `tl.dot` for matrix ops
2. **FP32 accumulation**: Accumulate in higher precision
3. **Fused operations**: Combine ops in one kernel
4. **Register pressure**: Balance block size vs registers

### Tuning

1. **Auto-tune**: Let compiler find best config
2. **num_warps**: Adjust occupancy
3. **num_stages**: Pipeline depth
4. **BLOCK_SIZE**: Trade-off memory vs parallelism

## Common Patterns

### Element-wise Operations

```python
@triton.jit
def elementwise_kernel(x_ptr, y_ptr, out_ptr, n, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(0)
    offs = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offs < n

    x = tl.load(x_ptr + offs, mask=mask)
    y = tl.load(y_ptr + offs, mask=mask)

    result = x * y + 1.0  # Fused multiply-add-constant

    tl.store(out_ptr + offs, result, mask=mask)
```

### Reductions

```python
@triton.jit
def reduce_kernel(x_ptr, out_ptr, M, N, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(0)

    # Process row
    row_start = x_ptr + pid * N
    cols = tl.arange(0, BLOCK_SIZE)
    mask = cols < N

    # Load and reduce
    x = tl.load(row_start + cols, mask=mask, other=0.0)
    result = tl.sum(x)

    # Store
    tl.store(out_ptr + pid, result)
```

## Integration with PyTorch

```python
import torch
import triton

# PyTorch tensors as input
x = torch.randn(1024, device='cuda')
y = torch.randn(1024, device='cuda')

# Launch Triton kernel
output = torch.empty_like(x)
grid = (triton.cdiv(1024, BLOCK_SIZE),)
my_kernel[grid](x, y, output, 1024, BLOCK_SIZE=256)

# Use in nn.Module
class CustomOp(torch.nn.Module):
    def forward(self, x):
        output = torch.empty_like(x)
        # Launch Triton kernel
        return output
```

## Best Practices

1. **Start simple**: Begin with element-wise ops
2. **Profile first**: Identify bottlenecks
3. **Use auto-tune**: Let compiler optimize
4. **Validate correctness**: Test against PyTorch
5. **Benchmark properly**: Warm-up, multiple runs
6. **Fuse operations**: Reduce memory traffic
7. **Power-of-two sizes**: Optimize for hardware

## References

- **[Programming Guide](references/programming-guide.md)** - Design philosophy, block-based model, optimization
- **[Tutorials](references/tutorials.md)** - Vector add, softmax, matmul examples
- **[Advanced Tutorials](references/advanced-tutorials.md)** - Flash Attention, dropout, layer norm, grouped GEMM, persistent kernels, FP8
- **[Language API](references/language-api.md)** - Operations, data types, memory, math
- **[Semantics](references/semantics.md)** - NumPy compatibility, type promotion, broadcasting, integer division
- **[Performance Tuning](references/performance-tuning.md)** - Auto-tuning, optimization strategies
- **[Matrix Operations](references/matrix-operations.md)** - Matmul, tensor cores, tiling
- **[Integration](references/integration.md)** - PyTorch, deployment, profiling
- **[Debugging](references/debugging.md)** - TRITON_INTERPRET, device_print, device_assert, compute-sanitizer
- **[Backends](references/backends.md)** - CUDA, HIP, CPU, compilation pipeline, installation
