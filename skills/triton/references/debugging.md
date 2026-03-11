# Triton Debugging Reference

Sources:
- [Debugging Guide](https://triton-lang.org/main/programming-guide/chapter-3/debugging.html)
- [device_print API](https://triton-lang.org/main/python-api/generated/triton.language.device_print.html)
- [device_assert API](https://triton-lang.org/main/python-api/generated/triton.language.device_assert.html)
- [static_print API](https://triton-lang.org/main/python-api/generated/triton.language.static_print.html)
- [static_assert API](https://triton-lang.org/main/python-api/generated/triton.language.static_assert.html)

## Built-in Debugging Operations

### Compile-Time Debugging

**static_print:**
```python
@triton.jit
def kernel(..., BLOCK_SIZE: tl.constexpr):
    # Print at compile time
    tl.static_print("BLOCK_SIZE =", BLOCK_SIZE)
    # Useful for verifying constexpr values
```

**static_assert:**
```python
@triton.jit
def kernel(..., BLOCK_SIZE: tl.constexpr):
    # Compile-time assertion
    tl.static_assert(BLOCK_SIZE % 16 == 0, "BLOCK_SIZE must be multiple of 16")
    tl.static_assert(BLOCK_SIZE <= 1024, "BLOCK_SIZE too large")
```

### Runtime Debugging

**device_print:**
```python
@triton.jit
def kernel(x_ptr, ...):
    pid = tl.program_id(0)
    
    # Print scalar values
    tl.device_print("pid =", pid)
    
    # Print tensor values
    offsets = tl.arange(0, BLOCK_SIZE)
    x = tl.load(x_ptr + offsets)
    tl.device_print("x[0:4] =", x)
    
    # Multiple values
    tl.device_print("pid =", pid, "max(x) =", tl.max(x))
```

**device_assert:**
```python
# Requires TRITON_DEBUG=1 environment variable
@triton.jit
def kernel(x_ptr, n_elements, ...):
    offsets = tl.arange(0, BLOCK_SIZE)
    
    # Runtime assertion
    tl.device_assert(offsets < n_elements, "Out of bounds access")
    
    x = tl.load(x_ptr + offsets)
    tl.device_assert(x >= 0, "Negative values not allowed")
```

**Enable device_assert:**
```bash
export TRITON_DEBUG=1
python script.py
```

## Interpreter Mode

### Overview
Bypass compilation and simulate kernels on CPU using NumPy equivalents.

### Enable Interpreter

```bash
# Set environment variable
export TRITON_INTERPRET=1
python script.py
```

### Use Case 1: Print Inspection

```python
@triton.jit
def debug_kernel(x_ptr, output_ptr, n, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(0)
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    
    x = tl.load(x_ptr + offsets, mask=offsets < n)
    
    # Print entire tensor
    print(x)
    
    # Print specific element
    print(f"x[0] = {x.handle.data[0]}")
    
    result = tl.exp(x)
    print(f"exp(x[0]) = {result.handle.data[0]}")
    
    tl.store(output_ptr + offsets, result, mask=offsets < n)
```

### Use Case 2: PDB Debugging

```bash
# Launch with debugger
TRITON_INTERPRET=1 python -m pdb script.py
```

**In script:**
```python
import pdb

@triton.jit
def kernel(...):
    pid = tl.program_id(0)
    
    # Set breakpoint
    pdb.set_trace()
    
    # Step through kernel execution
    offsets = tl.arange(0, BLOCK_SIZE)
    x = tl.load(x_ptr + offsets)
    # ... rest of kernel
```

**PDB commands:**
```
(Pdb) n          # Next line
(Pdb) s          # Step into
(Pdb) c          # Continue
(Pdb) p x        # Print variable
(Pdb) l          # List code
(Pdb) b 10       # Set breakpoint at line 10
```

### Use Case 3: External Debugger

```bash
# Use any debugger
TRITON_INTERPRET=1 gdb -ex run --args python script.py
```

### Interpreter Limitations

**1. No bfloat16 support:**
```python
# Convert to float32 for interpreter mode
import os
if os.getenv('TRITON_INTERPRET') == '1':
    dtype = torch.float32
else:
    dtype = torch.bfloat16
```

**2. Cannot handle indirect memory access:**
```python
# This may fail in interpreter mode
indices = tl.load(idx_ptr + offsets)
data = tl.load(data_ptr + indices)  # Indirect access
```

## Third-Party Tools

### NVIDIA: compute-sanitizer

Detects memory errors, data races, and synchronization issues.

```bash
# Memory check
compute-sanitizer --tool memcheck python script.py

# Race detection
compute-sanitizer --tool racecheck python script.py

# Initcheck (uninitialized memory)
compute-sanitizer --tool initcheck python script.py

# Synccheck (synchronization)
compute-sanitizer --tool synccheck python script.py
```

**Example output:**
```
========= COMPUTE-SANITIZER
========= Invalid __global__ write of size 4 bytes
=========     at 0x000001a0 in kernel
=========     by thread (0,0,0) in block (1,0,0)
=========     Address 0x7f8b40000000 is out of bounds
```

### AMD: AddressSanitizer

LLVM-based sanitizer for ROCm.

```bash
# Enable ASAN for ROCm
export ASAN_OPTIONS=detect_leaks=0
python script.py
```

### triton-viz

Visualize memory access patterns.

```bash
# Install
pip install triton-viz

# Use in code
import triton_viz

@triton_viz.trace
@triton.jit
def kernel(...):
    # Kernel code
    pass

# Launch visualization
kernel[grid](...)
triton_viz.show()
```

## Environment Variables

### TRITON_INTERPRET
```bash
# Enable interpreter mode
export TRITON_INTERPRET=1
```

### TRITON_DEBUG
```bash
# Enable device_assert
export TRITON_DEBUG=1
```

### TRITON_PRINT_AUTOTUNING
```bash
# Print autotuning information
export TRITON_PRINT_AUTOTUNING=1
```

### TRITON_CACHE_DIR
```bash
# Set cache directory for compiled kernels
export TRITON_CACHE_DIR=/path/to/cache
```

### TRITON_DUMP_DIR
```bash
# Dump intermediate representations
export TRITON_DUMP_DIR=/path/to/dumps
```

## Debugging Workflow

### Step 1: Isolate the Issue

```python
# Create minimal reproducer
import torch
import triton
import triton.language as tl

@triton.jit
def minimal_kernel(x_ptr, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(0)
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    x = tl.load(x_ptr + offsets)
    # Minimal failing case

# Test
x = torch.randn(1024, device='cuda')
grid = (1,)
minimal_kernel[grid](x, BLOCK_SIZE=256)
```

### Step 2: Add Assertions

```python
@triton.jit
def kernel(x_ptr, n, BLOCK_SIZE: tl.constexpr):
    # Compile-time checks
    tl.static_assert(BLOCK_SIZE > 0, "Invalid BLOCK_SIZE")
    
    pid = tl.program_id(0)
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    
    # Runtime checks (requires TRITON_DEBUG=1)
    tl.device_assert(offsets < n, "Out of bounds")
    
    x = tl.load(x_ptr + offsets, mask=offsets < n)
    tl.device_assert(tl.all(x != float('nan')), "NaN detected")
```

### Step 3: Inspect Values

```python
@triton.jit
def kernel(...):
    x = tl.load(x_ptr + offsets)
    
    # Print intermediate values
    tl.device_print("min(x) =", tl.min(x))
    tl.device_print("max(x) =", tl.max(x))
    tl.device_print("mean(x) =", tl.sum(x) / BLOCK_SIZE)
    
    result = compute(x)
    tl.device_print("result[0] =", result)
```

### Step 4: Use Interpreter

```bash
# Run in interpreter mode for detailed inspection
TRITON_INTERPRET=1 python script.py
```

### Step 5: Check Memory

```bash
# Use compute-sanitizer
compute-sanitizer --tool memcheck python script.py
```

## Common Issues

### Out-of-Bounds Access

**Problem:**
```python
# Bug: No masking
offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
x = tl.load(x_ptr + offsets)  # May access out of bounds
```

**Solution:**
```python
# Always use masking
offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
mask = offsets < n_elements
x = tl.load(x_ptr + offsets, mask=mask, other=0.0)
```

### Race Conditions

**Problem:**
```python
# Bug: Unsynchronized shared memory access
shared_data = tl.zeros([BLOCK_SIZE], dtype=tl.float32)
# ... multiple writes from different programs ...
result = tl.load(shared_data)  # Race!
```

**Solution:**
```python
# Use atomics or proper synchronization
tl.atomic_add(shared_ptr + offset, value)
```

### NaN/Inf Values

**Debug:**
```python
@triton.jit
def kernel(...):
    x = tl.load(x_ptr + offsets)
    
    # Check for invalid values
    has_nan = tl.any(x != x)  # NaN != NaN
    has_inf = tl.any(tl.abs(x) == float('inf'))
    
    tl.device_assert(not has_nan, "NaN detected")
    tl.device_assert(not has_inf, "Inf detected")
```

### Type Mismatches

**Debug:**
```python
@triton.jit
def kernel(...):
    x = tl.load(x_ptr + offsets)  # float16
    y = tl.load(y_ptr + offsets)  # float32
    
    # Print types at compile time
    tl.static_print("x dtype =", x.dtype)
    tl.static_print("y dtype =", y.dtype)
    
    # Explicit conversion
    result = x.to(tl.float32) + y
```

## Best Practices

1. **Start with static_assert**: Catch issues at compile time
2. **Use interpreter mode early**: Debug logic on CPU
3. **Add device_assert liberally**: Catch runtime errors
4. **Print intermediate values**: Understand data flow
5. **Use compute-sanitizer**: Find memory bugs
6. **Create minimal reproducers**: Isolate issues
7. **Check bounds carefully**: Always mask loads/stores
8. **Validate inputs**: Assert preconditions
