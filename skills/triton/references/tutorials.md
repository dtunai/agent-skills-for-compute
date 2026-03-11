# Triton Tutorials Reference

Sources:
- [Tutorials](https://triton-lang.org/main/getting-started/tutorials/index.html)

## Vector Addition Tutorial

### Objective
Perform element-wise addition of two vectors: `z[i] = x[i] + y[i]`

### Kernel Implementation

```python
import torch
import triton
import triton.language as tl

@triton.jit
def add_kernel(
    x_ptr,  # Pointer to first input vector
    y_ptr,  # Pointer to second input vector
    output_ptr,  # Pointer to output vector
    n_elements,  # Size of vectors
    BLOCK_SIZE: tl.constexpr,  # Number of elements each program should process
):
    # Identify which program (block) this is
    pid = tl.program_id(axis=0)
    
    # This program will process BLOCK_SIZE elements
    # starting at block_start
    block_start = pid * BLOCK_SIZE
    
    # Create range of offsets [0, 1, ..., BLOCK_SIZE-1]
    offsets = block_start + tl.arange(0, BLOCK_SIZE)
    
    # Create mask to guard memory operations
    # Some programs might need to process fewer than BLOCK_SIZE elements
    mask = offsets < n_elements
    
    # Load x and y from DRAM
    # Mask prevents out-of-bounds access
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    
    # Compute
    output = x + y
    
    # Write back to DRAM
    tl.store(output_ptr + offsets, output, mask=mask)
```

### Host Function

```python
def add(x: torch.Tensor, y: torch.Tensor):
    # Allocate output
    output = torch.empty_like(x)
    
    # Check tensors are on GPU
    assert x.is_cuda and y.is_cuda and output.is_cuda
    
    # Get number of elements
    n_elements = output.numel()
    
    # Define grid
    # BLOCK_SIZE is a tunable parameter
    # grid size = ceil(n_elements / BLOCK_SIZE)
    grid = lambda meta: (triton.cdiv(n_elements, meta['BLOCK_SIZE']),)
    
    # Launch kernel
    add_kernel[grid](x, y, output, n_elements, BLOCK_SIZE=1024)
    
    return output
```

### Usage

```python
# Create random vectors
size = 98432
x = torch.rand(size, device='cuda')
y = torch.rand(size, device='cuda')

# Run Triton kernel
output_triton = add(x, y)

# Verify correctness
output_torch = x + y
print(f'Max error: {torch.max(torch.abs(output_triton - output_torch))}')
```

## Fused Softmax Tutorial

### Objective
Compute softmax: `softmax(x)[i] = exp(x[i]) / sum(exp(x[j]))`

### Kernel Implementation

```python
@triton.jit
def softmax_kernel(
    output_ptr,
    input_ptr,
    input_row_stride,
    output_row_stride,
    n_cols,
    BLOCK_SIZE: tl.constexpr,
):
    # Row index - each program processes one row
    row_idx = tl.program_id(0)
    
    # Compute pointers to start of this row
    row_start_ptr = input_ptr + row_idx * input_row_stride
    
    # Column offsets
    col_offsets = tl.arange(0, BLOCK_SIZE)
    input_ptrs = row_start_ptr + col_offsets
    
    # Load row into SRAM
    mask = col_offsets < n_cols
    row = tl.load(input_ptrs, mask=mask, other=-float('inf'))
    
    # Subtract maximum for numerical stability
    row_minus_max = row - tl.max(row, axis=0)
    
    # Compute numerator
    numerator = tl.exp(row_minus_max)
    
    # Compute denominator
    denominator = tl.sum(numerator, axis=0)
    
    # Final result
    softmax_output = numerator / denominator
    
    # Write back to DRAM
    output_row_start_ptr = output_ptr + row_idx * output_row_stride
    output_ptrs = output_row_start_ptr + col_offsets
    tl.store(output_ptrs, softmax_output, mask=mask)
```

### Host Function

```python
def softmax(x):
    n_rows, n_cols = x.shape
    
    # Block size must be power of 2 and >= n_cols
    BLOCK_SIZE = triton.next_power_of_2(n_cols)
    
    # Allocate output
    y = torch.empty_like(x)
    
    # Launch one program per row
    num_programs = n_rows
    
    softmax_kernel[(num_programs,)](
        y,
        x,
        x.stride(0),
        y.stride(0),
        n_cols,
        BLOCK_SIZE=BLOCK_SIZE,
    )
    
    return y
```

### Benchmarking

```python
@triton.testing.perf_report(
    triton.testing.Benchmark(
        x_names=['N'],  # Argument names to vary
        x_vals=[128 * i for i in range(2, 100)],  # Different values for N
        line_arg='provider',  # Argument name whose value corresponds to different lines
        line_vals=['triton', 'torch'],  # Label for each line
        line_names=['Triton', 'PyTorch'],  # Human-readable name for each line
        styles=[('blue', '-'), ('green', '-')],
        ylabel='GB/s',  # Y-axis label
        plot_name='softmax-performance',
        args={'M': 4096},  # Fixed arguments
    )
)
def benchmark(M, N, provider):
    x = torch.randn(M, N, device='cuda', dtype=torch.float32)
    quantiles = [0.5, 0.2, 0.8]
    
    if provider == 'torch':
        ms, min_ms, max_ms = triton.testing.do_bench(lambda: torch.softmax(x, axis=-1), quantiles=quantiles)
    if provider == 'triton':
        ms, min_ms, max_ms = triton.testing.do_bench(lambda: softmax(x), quantiles=quantiles)
    
    gbps = lambda ms: 2 * x.numel() * x.element_size() * 1e-9 / (ms * 1e-3)
    return gbps(ms), gbps(max_ms), gbps(min_ms)

# Run benchmark
benchmark.run(print_data=True, show_plots=True)
```

## Matrix Multiplication Tutorial

### Kernel Implementation

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
    
    # Reorder programs for better L2 cache locality
    num_pid_in_group = GROUP_SIZE_M * num_pid_n
    group_id = pid // num_pid_in_group
    first_pid_m = group_id * GROUP_SIZE_M
    group_size_m = min(num_pid_m - first_pid_m, GROUP_SIZE_M)
    
    pid_m = first_pid_m + (pid % group_size_m)
    pid_n = (pid % num_pid_in_group) // group_size_m
    
    # Create pointers for A and B blocks
    offs_am = (pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)) % M
    offs_bn = (pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)) % N
    offs_k = tl.arange(0, BLOCK_SIZE_K)
    
    a_ptrs = a_ptr + (offs_am[:, None] * stride_am + offs_k[None, :] * stride_ak)
    b_ptrs = b_ptr + (offs_k[:, None] * stride_bk + offs_bn[None, :] * stride_bn)
    
    # Initialize accumulator
    accumulator = tl.zeros((BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32)
    
    # Loop over K dimension in blocks
    for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
        # Load current blocks of A and B
        a = tl.load(a_ptrs, mask=offs_k[None, :] < K - k * BLOCK_SIZE_K, other=0.0)
        b = tl.load(b_ptrs, mask=offs_k[:, None] < K - k * BLOCK_SIZE_K, other=0.0)
        
        # Matrix multiply using tensor cores
        accumulator += tl.dot(a, b)
        
        # Advance pointers to next K block
        a_ptrs += BLOCK_SIZE_K * stride_ak
        b_ptrs += BLOCK_SIZE_K * stride_bk
    
    # Convert accumulator to output type
    c = accumulator.to(tl.float16)
    
    # Write output
    offs_cm = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
    offs_cn = pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)
    c_ptrs = c_ptr + stride_cm * offs_cm[:, None] + stride_cn * offs_cn[None, :]
    c_mask = (offs_cm[:, None] < M) & (offs_cn[None, :] < N)
    tl.store(c_ptrs, c, mask=c_mask)
```

### Host Function with Auto-tuning

```python
def matmul(a, b):
    # Check constraints
    assert a.shape[1] == b.shape[0], "Incompatible dimensions"
    assert a.is_contiguous() and b.is_contiguous()
    
    M, K = a.shape
    K, N = b.shape
    
    # Allocate output
    c = torch.empty((M, N), device=a.device, dtype=a.dtype)
    
    # 2D launch grid
    grid = lambda META: (
        triton.cdiv(M, META['BLOCK_SIZE_M']) * triton.cdiv(N, META['BLOCK_SIZE_N']),
    )
    
    matmul_kernel[grid](
        a, b, c,
        M, N, K,
        a.stride(0), a.stride(1),
        b.stride(0), b.stride(1),
        c.stride(0), c.stride(1),
    )
    
    return c
```

### Testing

```python
# Create random matrices
a = torch.randn((512, 512), device='cuda', dtype=torch.float16)
b = torch.randn((512, 512), device='cuda', dtype=torch.float16)

# Triton result
triton_output = matmul(a, b)

# PyTorch result
torch_output = torch.matmul(a, b)

# Check correctness
print(f"Max difference: {torch.max(torch.abs(triton_output - torch_output))}")

# Relative error
rtol = 0
if torch.allclose(triton_output, torch_output, atol=1e-2, rtol=rtol):
    print("✅ Triton and PyTorch match")
else:
    print("❌ Triton and PyTorch differ")
```

## Layer Normalization Tutorial

```python
@triton.jit
def layer_norm_kernel(
    output_ptr,
    input_ptr,
    weight_ptr,
    bias_ptr,
    input_row_stride,
    output_row_stride,
    n_cols,
    eps,
    BLOCK_SIZE: tl.constexpr,
):
    row_idx = tl.program_id(0)
    
    # Load row
    row_start_ptr = input_ptr + row_idx * input_row_stride
    col_offsets = tl.arange(0, BLOCK_SIZE)
    input_ptrs = row_start_ptr + col_offsets
    mask = col_offsets < n_cols
    row = tl.load(input_ptrs, mask=mask, other=0.0)
    
    # Compute mean
    row_mean = tl.sum(row, axis=0) / n_cols
    
    # Compute variance
    row_var = tl.sum((row - row_mean) * (row - row_mean), axis=0) / n_cols
    
    # Normalize
    row_normalized = (row - row_mean) / tl.sqrt(row_var + eps)
    
    # Apply affine transformation
    weight = tl.load(weight_ptr + col_offsets, mask=mask)
    bias = tl.load(bias_ptr + col_offsets, mask=mask)
    output = row_normalized * weight + bias
    
    # Store result
    output_row_start_ptr = output_ptr + row_idx * output_row_stride
    output_ptrs = output_row_start_ptr + col_offsets
    tl.store(output_ptrs, output, mask=mask)
```
