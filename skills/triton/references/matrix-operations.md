# Triton Matrix Operations Reference

Sources:
- [Matrix Multiplication Tutorial](https://triton-lang.org/main/getting-started/tutorials/03-matrix-multiplication.html)

## Matrix Multiplication Basics

### Operation
Compute `C = A @ B` where:
- A: (M, K)
- B: (K, N)
- C: (M, N)

### Algorithm
```
for m in range(M):
    for n in range(N):
        for k in range(K):
            C[m, n] += A[m, k] * B[k, n]
```

## Tiling Strategy

### Block Decomposition

```
Split matrices into blocks:
- A: (M, K) → (M/BLOCK_M, K/BLOCK_K) blocks of size (BLOCK_M, BLOCK_K)
- B: (K, N) → (K/BLOCK_K, N/BLOCK_N) blocks of size (BLOCK_K, BLOCK_N)
- C: (M, N) → (M/BLOCK_M, N/BLOCK_N) blocks of size (BLOCK_M, BLOCK_N)

Each program computes one C block:
  C[m:m+BLOCK_M, n:n+BLOCK_N] = sum over k of 
    A[m:m+BLOCK_M, k:k+BLOCK_K] @ B[k:k+BLOCK_K, n:n+BLOCK_N]
```

### Implementation

```python
@triton.jit
def matmul_kernel(
    a_ptr, b_ptr, c_ptr,
    M, N, K,
    stride_am, stride_ak,  # A strides
    stride_bk, stride_bn,  # B strides
    stride_cm, stride_cn,  # C strides
    BLOCK_SIZE_M: tl.constexpr,
    BLOCK_SIZE_N: tl.constexpr,
    BLOCK_SIZE_K: tl.constexpr,
):
    # Program ID
    pid = tl.program_id(axis=0)
    
    # Number of blocks in M and N dimensions
    num_pid_m = tl.cdiv(M, BLOCK_SIZE_M)
    num_pid_n = tl.cdiv(N, BLOCK_SIZE_N)
    
    # Convert 1D program ID to 2D (block_m, block_n)
    pid_m = pid // num_pid_n
    pid_n = pid % num_pid_n
    
    # Create block offsets
    offs_am = (pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)) % M
    offs_bn = (pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)) % N
    offs_k = tl.arange(0, BLOCK_SIZE_K)
    
    # Initialize pointers for A and B
    a_ptrs = a_ptr + (offs_am[:, None] * stride_am + offs_k[None, :] * stride_ak)
    b_ptrs = b_ptr + (offs_k[:, None] * stride_bk + offs_bn[None, :] * stride_bn)
    
    # Accumulator for C block
    accumulator = tl.zeros((BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32)
    
    # Loop over K dimension
    for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
        # Load A and B blocks
        a = tl.load(a_ptrs, mask=offs_k[None, :] < K - k * BLOCK_SIZE_K, other=0.0)
        b = tl.load(b_ptrs, mask=offs_k[:, None] < K - k * BLOCK_SIZE_K, other=0.0)
        
        # Matrix multiply using tensor cores
        accumulator += tl.dot(a, b)
        
        # Advance K pointers
        a_ptrs += BLOCK_SIZE_K * stride_ak
        b_ptrs += BLOCK_SIZE_K * stride_bk
    
    # Convert to output dtype
    c = accumulator.to(tl.float16)
    
    # Write C block
    offs_cm = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
    offs_cn = pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)
    c_ptrs = c_ptr + stride_cm * offs_cm[:, None] + stride_cn * offs_cn[None, :]
    c_mask = (offs_cm[:, None] < M) & (offs_cn[None, :] < N)
    tl.store(c_ptrs, c, mask=c_mask)
```

## L2 Cache Optimization

### Super-Grouping

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
    GROUP_SIZE_M: tl.constexpr,  # New parameter
):
    pid = tl.program_id(axis=0)
    num_pid_m = tl.cdiv(M, BLOCK_SIZE_M)
    num_pid_n = tl.cdiv(N, BLOCK_SIZE_N)
    
    # Group programs for L2 cache locality
    num_pid_in_group = GROUP_SIZE_M * num_pid_n
    group_id = pid // num_pid_in_group
    first_pid_m = group_id * GROUP_SIZE_M
    group_size_m = min(num_pid_m - first_pid_m, GROUP_SIZE_M)
    
    # Compute position within group
    pid_m = first_pid_m + (pid % group_size_m)
    pid_n = (pid % num_pid_in_group) // group_size_m
    
    # Rest of kernel unchanged
    ...
```

**Benefits:**
- Programs in same group share L2 cache
- Reduces redundant memory loads
- Improves overall bandwidth utilization

## Tensor Core Usage

### Automatic Scheduling

```python
# tl.dot automatically uses tensor cores when available
C = tl.dot(A, B)

# Compiler handles:
# - Data layout conversion (row-major to tensor core layout)
# - Instruction scheduling
# - Register allocation
```

### Tensor Core Tile Sizes

**NVIDIA Ampere (A100):**
- FP16/BF16: 16x16x16
- TF32: 16x16x8
- FP64: 8x8x4

**NVIDIA Hopper (H100):**
- FP16/BF16: 16x16x16
- FP8: 16x16x32
- TF32: 16x16x8

### Optimal Block Sizes

```python
# Choose BLOCK_SIZE as multiple of tensor core tile
BLOCK_SIZE_M = 128  # Multiple of 16
BLOCK_SIZE_N = 256  # Multiple of 16
BLOCK_SIZE_K = 64   # Multiple of 16
```

## Mixed Precision

### FP32 Accumulation

```python
# Load in FP16/BF16
a = tl.load(a_ptrs).to(tl.float16)
b = tl.load(b_ptrs).to(tl.float16)

# Accumulate in FP32 for accuracy
accumulator = tl.zeros((BLOCK_M, BLOCK_N), dtype=tl.float32)
accumulator += tl.dot(a, b)  # tl.dot promotes to FP32

# Store in FP16
c = accumulator.to(tl.float16)
tl.store(c_ptrs, c)
```

### FP8 Support (H100)

```python
# Load as FP8
a = tl.load(a_ptrs, dtype=tl.float8_e4m3fn)
b = tl.load(b_ptrs, dtype=tl.float8_e4m3fn)

# Tensor cores handle FP8 → FP32 accumulation
accumulator += tl.dot(a, b)
```

## Batched Matrix Multiply

### 3D Tensors

```python
@triton.jit
def batched_matmul_kernel(
    a_ptr, b_ptr, c_ptr,
    batch_size, M, N, K,
    stride_ab, stride_am, stride_ak,
    stride_bb, stride_bk, stride_bn,
    stride_cb, stride_cm, stride_cn,
    BLOCK_SIZE_M: tl.constexpr,
    BLOCK_SIZE_N: tl.constexpr,
    BLOCK_SIZE_K: tl.constexpr,
):
    # 2D program ID
    pid_batch = tl.program_id(0)
    pid_mn = tl.program_id(1)
    
    # Compute (m, n) from flat pid_mn
    num_pid_m = tl.cdiv(M, BLOCK_SIZE_M)
    num_pid_n = tl.cdiv(N, BLOCK_SIZE_N)
    pid_m = pid_mn // num_pid_n
    pid_n = pid_mn % num_pid_n
    
    # Offset pointers to current batch
    a_ptr += pid_batch * stride_ab
    b_ptr += pid_batch * stride_bb
    c_ptr += pid_batch * stride_cb
    
    # Standard matmul kernel
    ...

# Launch with 2D grid
grid = (batch_size, triton.cdiv(M, BLOCK_M) * triton.cdiv(N, BLOCK_N))
batched_matmul_kernel[grid](...)
```

## Strided Operations

### Transpose Multiply

```python
# Compute C = A.T @ B

@triton.jit
def matmul_transpose_kernel(
    a_ptr, b_ptr, c_ptr,
    M, N, K,
    # A is stored as (K, M) but accessed as (M, K).T
    stride_ak, stride_am,  # Swapped for transpose
    stride_bk, stride_bn,
    stride_cm, stride_cn,
    ...
):
    # Same as regular matmul, just with swapped strides
    ...
```

### Non-Contiguous Tensors

```python
# Handle arbitrary strides
def matmul(a, b):
    assert a.shape[1] == b.shape[0]
    M, K = a.shape
    K, N = b.shape
    
    c = torch.empty((M, N), device=a.device, dtype=a.dtype)
    
    # Use actual tensor strides
    grid = lambda META: (
        triton.cdiv(M, META['BLOCK_SIZE_M']) * triton.cdiv(N, META['BLOCK_SIZE_N']),
    )
    
    matmul_kernel[grid](
        a, b, c,
        M, N, K,
        a.stride(0), a.stride(1),  # May be non-standard
        b.stride(0), b.stride(1),
        c.stride(0), c.stride(1),
    )
    
    return c
```

## Specialized Operations

### Matrix-Vector Multiply

```python
@triton.jit
def matvec_kernel(
    a_ptr, x_ptr, y_ptr,
    M, N,
    stride_am, stride_an,
    BLOCK_SIZE: tl.constexpr,
):
    # One program per row
    row = tl.program_id(0)
    
    # Load row of A
    cols = tl.arange(0, BLOCK_SIZE)
    a_ptrs = a_ptr + row * stride_am + cols * stride_an
    
    # Accumulate dot product
    acc = 0.0
    for col_start in range(0, N, BLOCK_SIZE):
        mask = cols < N - col_start
        a = tl.load(a_ptrs, mask=mask, other=0.0)
        x = tl.load(x_ptr + cols, mask=mask, other=0.0)
        
        acc += tl.sum(a * x)
        
        a_ptrs += BLOCK_SIZE * stride_an
        cols += BLOCK_SIZE
    
    # Store result
    tl.store(y_ptr + row, acc)
```

### Outer Product

```python
@triton.jit
def outer_product_kernel(
    x_ptr, y_ptr, out_ptr,
    M, N,
    BLOCK_M: tl.constexpr,
    BLOCK_N: tl.constexpr,
):
    pid_m = tl.program_id(0)
    pid_n = tl.program_id(1)
    
    # Load x and y blocks
    offs_m = pid_m * BLOCK_M + tl.arange(0, BLOCK_M)
    offs_n = pid_n * BLOCK_N + tl.arange(0, BLOCK_N)
    
    x = tl.load(x_ptr + offs_m, mask=offs_m < M)
    y = tl.load(y_ptr + offs_n, mask=offs_n < N)
    
    # Compute outer product: x[:, None] * y[None, :]
    result = x[:, None] * y[None, :]
    
    # Store
    out_ptrs = out_ptr + offs_m[:, None] * N + offs_n[None, :]
    mask = (offs_m[:, None] < M) & (offs_n[None, :] < N)
    tl.store(out_ptrs, result, mask=mask)
```

## Performance Characteristics

### Roofline Model

```
Performance limited by:
1. Memory bandwidth (memory-bound)
2. Compute throughput (compute-bound)

Matmul arithmetic intensity:
  FLOPs = 2*M*N*K
  Bytes = (M*K + K*N + M*N) * sizeof(dtype)
  Intensity = FLOPs / Bytes

For large matrices, matmul is compute-bound
Goal: Maximize FLOPS → use tensor cores
```

### Expected Performance

**A100 (40GB):**
- Peak FP16 tensor core: 312 TFLOPS
- Typical matmul: 250-280 TFLOPS (80-90% peak)

**H100 (80GB):**
- Peak FP16 tensor core: 989 TFLOPS
- Peak FP8 tensor core: 1979 TFLOPS
- Typical matmul: 800-900 TFLOPS FP16 (80-90% peak)
