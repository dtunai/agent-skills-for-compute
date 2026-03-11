# Triton Integration Reference

Sources:
- [Triton Documentation](https://triton-lang.org/)

## PyTorch Integration

### Basic Usage

```python
import torch
import triton
import triton.language as tl

# PyTorch tensors as input
x = torch.randn(1024, device='cuda', dtype=torch.float32)
y = torch.randn(1024, device='cuda', dtype=torch.float32)

# Call Triton kernel
@triton.jit
def add_kernel(x_ptr, y_ptr, output_ptr, n, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(0)
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n
    x_vals = tl.load(x_ptr + offsets, mask=mask)
    y_vals = tl.load(y_ptr + offsets, mask=mask)
    tl.store(output_ptr + offsets, x_vals + y_vals, mask=mask)

# Launch kernel
output = torch.empty_like(x)
grid = (triton.cdiv(x.numel(), 256),)
add_kernel[grid](x, y, output, x.numel(), BLOCK_SIZE=256)
```

### nn.Module Integration

```python
import torch.nn as nn

class TritonLinear(nn.Module):
    def __init__(self, in_features, out_features):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(out_features, in_features))
        self.bias = nn.Parameter(torch.randn(out_features))
    
    def forward(self, x):
        # x: (batch, in_features)
        # weight: (out_features, in_features)
        
        batch_size = x.shape[0]
        output = torch.empty(batch_size, self.weight.shape[0], 
                           device=x.device, dtype=x.dtype)
        
        # Launch Triton matmul
        grid = lambda META: (
            triton.cdiv(batch_size, META['BLOCK_M']) * 
            triton.cdiv(self.weight.shape[0], META['BLOCK_N']),
        )
        
        triton_matmul[grid](
            x, self.weight.t(), output,
            batch_size, self.weight.shape[0], self.weight.shape[1],
            # ... strides ...
        )
        
        # Add bias
        output += self.bias
        return output
```

### Autograd Integration

```python
class TritonFunction(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x, weight):
        # Save for backward
        ctx.save_for_backward(x, weight)
        
        # Forward pass with Triton
        output = torch.empty(x.shape[0], weight.shape[0], 
                           device=x.device, dtype=x.dtype)
        triton_matmul(x, weight, output)
        return output
    
    @staticmethod
    def backward(ctx, grad_output):
        x, weight = ctx.saved_tensors
        
        # Gradient w.r.t. x: grad_output @ weight
        grad_x = torch.empty_like(x)
        triton_matmul(grad_output, weight, grad_x)
        
        # Gradient w.r.t. weight: grad_output.T @ x
        grad_weight = torch.empty_like(weight)
        triton_matmul(grad_output.t(), x, grad_weight)
        
        return grad_x, grad_weight

# Usage
triton_op = TritonFunction.apply
output = triton_op(x, weight)
output.backward(grad_output)
```

## Tensor Compatibility

### Data Types

```python
# Triton supports PyTorch dtypes
torch_to_triton = {
    torch.float32: tl.float32,
    torch.float16: tl.float16,
    torch.bfloat16: tl.bfloat16,
    torch.int32: tl.int32,
    torch.int64: tl.int64,
}

# Access PyTorch tensor in Triton
def triton_kernel_wrapper(tensor):
    assert tensor.is_cuda, "Tensor must be on GPU"
    assert tensor.is_contiguous(), "Tensor must be contiguous"
    
    # Pass tensor directly to kernel
    kernel[grid](tensor, ...)
```

### Stride Handling

```python
def handle_strides(tensor):
    # Get strides for multidimensional access
    stride_0 = tensor.stride(0)
    stride_1 = tensor.stride(1)
    
    # Pass to kernel
    kernel[grid](
        tensor,
        stride_0,
        stride_1,
        ...
    )

# In kernel
@triton.jit
def kernel(ptr, stride_0, stride_1, ...):
    # 2D indexing with strides
    idx = row * stride_0 + col * stride_1
    value = tl.load(ptr + idx)
```

### Non-Contiguous Tensors

```python
def ensure_contiguous(tensor):
    # Make contiguous if needed
    if not tensor.is_contiguous():
        tensor = tensor.contiguous()
    
    return tensor

# Or handle in kernel with strides
def handle_noncontiguous(tensor):
    # Pass actual strides to kernel
    kernel[grid](tensor, *tensor.stride(), ...)
```

## Deployment

### JIT Compilation

```python
# Kernels are JIT compiled on first call
@triton.jit
def kernel(...):
    pass

# First call: compilation happens (slow)
kernel[grid](...)

# Subsequent calls: use cached compilation (fast)
kernel[grid](...)
```

### AOT Compilation

```python
# Ahead-of-time compilation
from triton.compiler import compile

# Compile kernel
compiled = compile(kernel, signature={...}, device_capability=...)

# Save compiled kernel
torch.save(compiled, 'kernel.pt')

# Load and use
compiled = torch.load('kernel.pt')
```

### Multi-GPU

```python
# Launch on specific GPU
device_id = 0
with torch.cuda.device(device_id):
    tensor = torch.randn(1024, device='cuda')
    kernel[grid](tensor, ...)

# Multi-GPU data parallel
def multi_gpu_launch(tensors):
    outputs = []
    for i, tensor in enumerate(tensors):
        with torch.cuda.device(i):
            output = torch.empty_like(tensor)
            kernel[grid](tensor, output, ...)
            outputs.append(output)
    return outputs
```

## Profiling

### PyTorch Profiler

```python
from torch.profiler import profile, ProfilerActivity

with profile(activities=[ProfilerActivity.CUDA]) as prof:
    kernel[grid](x, y, output, ...)

print(prof.key_averages().table(sort_by="cuda_time_total"))
```

### Nsight Systems

```bash
# Profile Triton kernel with Nsight Systems
nsys profile -o profile.qdrep python script.py

# View in Nsight Systems GUI
nsight-sys profile.qdrep
```

### Nsight Compute

```bash
# Detailed kernel profiling
ncu --set full -o profile python script.py

# Metrics
ncu --metrics sm__throughput.avg.pct_of_peak_sustained_elapsed python script.py
```

### Triton Profiler

```python
# Built-in profiling
@triton.testing.perf_report(...)
def benchmark(...):
    triton.testing.do_bench(lambda: kernel[grid](...))
```

## Testing

### Correctness Validation

```python
def test_kernel():
    # Generate test data
    x = torch.randn(1024, device='cuda')
    y = torch.randn(1024, device='cuda')
    
    # Triton result
    triton_output = torch.empty_like(x)
    add_kernel[grid](x, y, triton_output, x.numel(), BLOCK_SIZE=256)
    
    # Reference result
    torch_output = x + y
    
    # Compare
    assert torch.allclose(triton_output, torch_output, atol=1e-5)
    print("✅ Test passed")

test_kernel()
```

### Property-Based Testing

```python
import hypothesis
from hypothesis import given
import hypothesis.strategies as st

@given(
    size=st.integers(min_value=1, max_value=10000),
    dtype=st.sampled_from([torch.float32, torch.float16])
)
def test_add_kernel(size, dtype):
    x = torch.randn(size, device='cuda', dtype=dtype)
    y = torch.randn(size, device='cuda', dtype=dtype)
    
    output = torch.empty_like(x)
    grid = (triton.cdiv(size, 256),)
    add_kernel[grid](x, y, output, size, BLOCK_SIZE=256)
    
    expected = x + y
    assert torch.allclose(output, expected, atol=1e-3)
```

## Best Practices

### Error Handling

```python
def safe_kernel_launch(tensor, ...):
    # Validate inputs
    assert tensor.is_cuda, "Tensor must be on GPU"
    assert tensor.dtype == torch.float32, "Only float32 supported"
    assert tensor.is_contiguous(), "Tensor must be contiguous"
    
    try:
        kernel[grid](tensor, ...)
    except Exception as e:
        print(f"Kernel launch failed: {e}")
        raise
```

### Memory Management

```python
# Pre-allocate output tensors
def efficient_kernel_wrapper(input_tensor):
    # Allocate output once
    output = torch.empty_like(input_tensor)
    
    # Reuse in loop
    for i in range(1000):
        kernel[grid](input_tensor, output, ...)
    
    return output
```

### Device Synchronization

```python
# Explicit synchronization when needed
def benchmark_with_sync():
    torch.cuda.synchronize()  # Ensure previous ops finished
    
    start = torch.cuda.Event(enable_timing=True)
    end = torch.cuda.Event(enable_timing=True)
    
    start.record()
    kernel[grid](...)
    end.record()
    
    torch.cuda.synchronize()  # Wait for kernel
    
    elapsed_ms = start.elapsed_time(end)
    return elapsed_ms
```

## Production Deployment

### Package Structure

```
my_triton_ops/
  __init__.py
  kernels/
    matmul.py
    attention.py
  tests/
    test_matmul.py
    test_attention.py
  setup.py
```

### Installation

```python
# setup.py
from setuptools import setup

setup(
    name='my-triton-ops',
    version='0.1.0',
    packages=['my_triton_ops'],
    install_requires=[
        'torch>=2.0.0',
        'triton>=2.0.0',
    ],
)
```

### Usage

```python
# In production code
from my_triton_ops.kernels import triton_matmul

# Use as drop-in replacement
output = triton_matmul(a, b)
```
