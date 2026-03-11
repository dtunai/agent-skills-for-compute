# Triton Performance Tuning Reference

Sources:
- [Performance Tuning Guide](https://triton-lang.org/main/getting-started/tutorials/03-matrix-multiplication.html)

## Auto-Tuning

### Basic Auto-Tuning

```python
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_SIZE_M': 128, 'BLOCK_SIZE_N': 256, 'BLOCK_SIZE_K': 64}, num_stages=3, num_warps=8),
        triton.Config({'BLOCK_SIZE_M': 64, 'BLOCK_SIZE_N': 256, 'BLOCK_SIZE_K': 32}, num_stages=4, num_warps=4),
        triton.Config({'BLOCK_SIZE_M': 128, 'BLOCK_SIZE_N': 128, 'BLOCK_SIZE_K': 32}, num_stages=4, num_warps=4),
        triton.Config({'BLOCK_SIZE_M': 128, 'BLOCK_SIZE_N': 64, 'BLOCK_SIZE_K': 32}, num_stages=4, num_warps=4),
        triton.Config({'BLOCK_SIZE_M': 64, 'BLOCK_SIZE_N': 128, 'BLOCK_SIZE_K': 32}, num_stages=4, num_warps=4),
        triton.Config({'BLOCK_SIZE_M': 128, 'BLOCK_SIZE_N': 32, 'BLOCK_SIZE_K': 32}, num_stages=4, num_warps=4),
        triton.Config({'BLOCK_SIZE_M': 64, 'BLOCK_SIZE_N': 32, 'BLOCK_SIZE_K': 32}, num_stages=5, num_warps=2),
        triton.Config({'BLOCK_SIZE_M': 32, 'BLOCK_SIZE_N': 64, 'BLOCK_SIZE_K': 32}, num_stages=5, num_warps=2),
    ],
    key=['M', 'N', 'K'],
)
@triton.jit
def matmul_kernel(...):
    pass
```

### Configuration Parameters

**Block Sizes:**
- `BLOCK_SIZE_M`, `BLOCK_SIZE_N`, `BLOCK_SIZE_K`: Tile dimensions
- Larger blocks: More computation per memory load
- Smaller blocks: Better cache utilization

**num_warps:**
- Number of warps per thread block
- More warps: Higher occupancy
- Fewer warps: More registers per thread
- Typical values: 1, 2, 4, 8

**num_stages:**
- Pipeline depth for async memory operations
- More stages: Better latency hiding
- Fewer stages: Lower register pressure
- Typical values: 2, 3, 4, 5

### Key Selection

```python
# Auto-tune based on input sizes
@triton.autotune(
    configs=[...],
    key=['M', 'N', 'K'],  # Tune separately for each (M, N, K) combination
)
```

### Heuristics

```python
@triton.heuristics({
    'EVEN_K': lambda args: args['K'] % args['BLOCK_SIZE_K'] == 0,
})
@triton.jit
def kernel(ptr, M, N, K, BLOCK_SIZE_K: tl.constexpr, EVEN_K: tl.constexpr):
    # Use EVEN_K to skip masking when K is multiple of BLOCK_SIZE_K
    if EVEN_K:
        data = tl.load(ptr)  # No mask needed
    else:
        data = tl.load(ptr, mask=...)
```

## Memory Optimization

### Coalescing

**Good (coalesced):**
```python
# Contiguous access
offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
data = tl.load(ptr + offsets)  # Sequential memory access
```

**Bad (uncoalesced):**
```python
# Strided access
offsets = pid + tl.arange(0, BLOCK_SIZE) * stride
data = tl.load(ptr + offsets)  # Non-contiguous access
```

### Alignment

```python
# Align block sizes to cache lines (typically 128 bytes)
BLOCK_SIZE = 128  # For float32: 128 * 4 = 512 bytes

# Hint to compiler about alignment
offsets = tl.multiple_of(offsets, 16)
```

### Cache Hints

```python
# Cache all levels (default)
data = tl.load(ptr, cache_modifier=".ca")

# Cache global only (L2)
data = tl.load(ptr, cache_modifier=".cg")

# Streaming (bypass L1)
data = tl.load(ptr, cache_modifier=".cs")
```

### Shared Memory

Compiler manages automatically, but you can influence:

```python
# Larger blocks use more shared memory
# Balance block size vs shared memory capacity
# GPU has limited shared memory per SM (e.g., 48KB, 96KB)
```

## Compute Optimization

### Tensor Core Utilization

```python
# Use tl.dot for matrix multiply (automatically uses tensor cores)
C = tl.dot(A, B)

# Ensure shapes are multiples of tensor core tile sizes
# Ampere (A100): 16x16x16 for FP16/BF16
# Hopper (H100): 16x16x16 for FP16/BF16, 16x16x32 for FP8
```

### FP32 Accumulation

```python
# Accumulate in FP32 for accuracy
accumulator = tl.zeros((BLOCK_M, BLOCK_N), dtype=tl.float32)

for k in range(...):
    a = tl.load(...).to(tl.float16)  # Load as FP16
    b = tl.load(...).to(tl.float16)
    accumulator += tl.dot(a, b)  # Accumulate in FP32

# Convert to output type
result = accumulator.to(tl.float16)
```

### Fused Operations

```python
# Fuse operations to reduce memory traffic
# Bad: Multiple kernel launches
x = input + bias  # Kernel 1
y = relu(x)       # Kernel 2
z = x * scale     # Kernel 3

# Good: Single fused kernel
@triton.jit
def fused_kernel(...):
    x = tl.load(input_ptr) + bias
    y = tl.maximum(x, 0)
    z = y * scale
    tl.store(output_ptr, z)
```

## Occupancy Tuning

### Warp Count

```python
# Low occupancy: More resources per thread
kernel[grid](ptr, num_warps=1)  # 32 threads
# Pros: More registers, shared memory
# Cons: Less parallelism

# High occupancy: More threads
kernel[grid](ptr, num_warps=8)  # 256 threads
# Pros: Better latency hiding
# Cons: Resource contention
```

### Register Pressure

```python
# Reduce register usage:
# 1. Smaller block sizes
# 2. Fewer num_stages
# 3. Recompute instead of store

# Example: Recompute offsets
for i in range(loops):
    offsets = compute_offsets()  # Recompute each iteration
    data = tl.load(ptr + offsets)
```

## Profiling

### Basic Benchmarking

```python
import triton.testing

# Benchmark function
def benchmark_kernel():
    kernel[grid](...)

# Measure time
ms = triton.testing.do_bench(benchmark_kernel)
print(f"Time: {ms:.3f} ms")
```

### Percentile Benchmarking

```python
# Get median, min, max
quantiles = [0.5, 0.2, 0.8]
median_ms, min_ms, max_ms = triton.testing.do_bench(
    benchmark_kernel, 
    quantiles=quantiles
)

print(f"Median: {median_ms:.3f} ms")
print(f"Min: {min_ms:.3f} ms")
print(f"Max: {max_ms:.3f} ms")
```

### Throughput Calculation

```python
# Calculate bandwidth
def compute_bandwidth(ms, num_bytes):
    return num_bytes * 1e-9 / (ms * 1e-3)  # GB/s

# Example: Vector add
num_bytes = 3 * n_elements * 4  # 3 arrays * 4 bytes (float32)
bandwidth = compute_bandwidth(ms, num_bytes)
print(f"Bandwidth: {bandwidth:.2f} GB/s")
```

### Comparative Benchmarking

```python
@triton.testing.perf_report(
    triton.testing.Benchmark(
        x_names=['size'],
        x_vals=[2**i for i in range(12, 28, 1)],
        line_arg='provider',
        line_vals=['triton', 'torch', 'cublas'],
        line_names=['Triton', 'PyTorch', 'cuBLAS'],
        ylabel='TFLOPS',
        plot_name='performance',
        args={}
    )
)
def benchmark(size, provider):
    # Setup
    a = torch.randn((size, size), device='cuda', dtype=torch.float16)
    b = torch.randn((size, size), device='cuda', dtype=torch.float16)
    
    # Benchmark
    if provider == 'cublas':
        ms = triton.testing.do_bench(lambda: torch.matmul(a, b))
    elif provider == 'triton':
        ms = triton.testing.do_bench(lambda: triton_matmul(a, b))
    elif provider == 'torch':
        ms = triton.testing.do_bench(lambda: a @ b)
    
    # Calculate TFLOPS
    flops = 2 * size**3  # Matrix multiply FLOPs
    tflops = flops * 1e-12 / (ms * 1e-3)
    
    return tflops
```

## Best Practices

### Block Size Selection

1. **Start with powers of 2**: 64, 128, 256
2. **Multiple of warp size (32)**: Ensures full warp utilization
3. **Consider GPU limits**: Max threads per block (1024)
4. **Balance**: Larger blocks → more computation but more resources

### Memory-Bound Kernels

1. **Maximize bandwidth**: Coalesced access
2. **Minimize traffic**: Fuse operations
3. **Use cache hints**: Streaming for one-time-use data
4. **Async loads**: Increase num_stages

### Compute-Bound Kernels

1. **Use tensor cores**: `tl.dot` for matrix ops
2. **High occupancy**: More warps
3. **FP16/BF16**: Half precision when possible
4. **Minimize synchronization**: Reduce dependencies

### General Guidelines

1. **Profile first**: Identify bottleneck
2. **Auto-tune**: Let compiler find best config
3. **Validate**: Check correctness vs reference
4. **Iterate**: Tune one parameter at a time

## Performance Tips

### DO

- ✅ Use power-of-2 block sizes
- ✅ Coalesce memory accesses
- ✅ Use `tl.dot` for matrix multiply
- ✅ Accumulate in FP32
- ✅ Fuse operations
- ✅ Auto-tune configurations
- ✅ Profile and benchmark

### DON'T

- ❌ Use non-contiguous memory access patterns
- ❌ Launch too many small kernels
- ❌ Ignore alignment
- ❌ Use excessive atomics
- ❌ Over-allocate shared memory
- ❌ Skip validation

## Common Patterns

### Memory-Bound Optimization

```python
# Element-wise: Focus on bandwidth
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_SIZE': 1024}, num_warps=4),
        triton.Config({'BLOCK_SIZE': 2048}, num_warps=8),
    ],
    key=['n_elements'],
)
@triton.jit
def elementwise_kernel(x_ptr, y_ptr, n_elements, BLOCK_SIZE: tl.constexpr):
    # Maximize bandwidth utilization
    pass
```

### Compute-Bound Optimization

```python
# Matrix multiply: Focus on FLOPs
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_M': 128, 'BLOCK_N': 128, 'BLOCK_K': 32}, num_warps=8),
        triton.Config({'BLOCK_M': 256, 'BLOCK_N': 64, 'BLOCK_K': 32}, num_warps=8),
    ],
    key=['M', 'N', 'K'],
)
@triton.jit
def matmul_kernel(...):
    # Maximize compute throughput
    accumulator = tl.dot(a, b)  # Use tensor cores
```
