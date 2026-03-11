# Triton Advanced Tutorials Reference

Sources:
- [Fused Attention](https://triton-lang.org/main/getting-started/tutorials/06-fused-attention.html)
- [Low-Memory Dropout](https://triton-lang.org/main/getting-started/tutorials/04-low-memory-dropout.html)
- [Layer Normalization](https://triton-lang.org/main/getting-started/tutorials/05-layer-norm.html)
- [Group GEMM](https://triton-lang.org/main/getting-started/tutorials/08-grouped-gemm.html)
- [Persistent Matmul](https://triton-lang.org/main/getting-started/tutorials/09-persistent-matmul.html)
- [Block-Scaled Matmul](https://triton-lang.org/main/getting-started/tutorials/10-block-scaled-matmul.html)

## Fused Attention (Flash Attention v2)

### Overview
Implementation of Flash Attention v2 by Tri Dao - memory-efficient attention through block-based processing.

### Core Algorithm

**Staged Computation:**
- Stage 1: Off-band computation for causal attention
- Stage 2: On-band (diagonal) blocks
- Stage 3: Full computation for non-causal attention

### Implementation

```python
@triton.jit
def _attn_fwd(
    Q, K, V, sm_scale,
    L, M,  # Outputs: log-sum-exp, max values
    Out,
    stride_qz, stride_qh, stride_qm, stride_qk,
    stride_kz, stride_kh, stride_kn, stride_kk,
    stride_vz, stride_vh, stride_vn, stride_vk,
    stride_oz, stride_oh, stride_om, stride_ok,
    Z, H, N_CTX,
    BLOCK_M: tl.constexpr,
    BLOCK_N: tl.constexpr,
    BLOCK_DMODEL: tl.constexpr,
    STAGE: tl.constexpr,
):
    # Program IDs
    start_m = tl.program_id(0)
    off_hz = tl.program_id(1)
    
    # Initialize pointers
    offs_m = start_m * BLOCK_M + tl.arange(0, BLOCK_M)
    offs_n = tl.arange(0, BLOCK_N)
    offs_d = tl.arange(0, BLOCK_DMODEL)
    
    # Load Q block
    q_ptrs = Q + off_hz * stride_qh + offs_m[:, None] * stride_qm + offs_d[None, :] * stride_qk
    q = tl.load(q_ptrs)
    
    # Initialize accumulators
    m_i = tl.zeros([BLOCK_M], dtype=tl.float32) - float("inf")
    l_i = tl.zeros([BLOCK_M], dtype=tl.float32)
    acc = tl.zeros([BLOCK_M, BLOCK_DMODEL], dtype=tl.float32)
    
    # Loop over K, V
    for start_n in range(0, N_CTX, BLOCK_N):
        # Load K, V blocks
        k_ptrs = K + off_hz * stride_kh + (start_n + offs_n[None, :]) * stride_kn + offs_d[:, None] * stride_kk
        v_ptrs = V + off_hz * stride_vh + (start_n + offs_n[:, None]) * stride_vn + offs_d[None, :] * stride_vk
        
        k = tl.load(k_ptrs)
        v = tl.load(v_ptrs)
        
        # Compute attention scores: QK^T
        qk = tl.dot(q, k) * sm_scale
        
        # Causal masking
        if STAGE == 1:  # Causal
            qk = tl.where(offs_m[:, None] >= (start_n + offs_n[None, :]), qk, float("-inf"))
        
        # Softmax computation (online)
        m_ij = tl.max(qk, 1)
        qk = qk - m_ij[:, None]
        p = tl.exp(qk)
        l_ij = tl.sum(p, 1)
        
        # Update max and running sum
        m_i_new = tl.maximum(m_i, m_ij)
        alpha = tl.exp(m_i - m_i_new)
        beta = tl.exp(m_ij - m_i_new)
        
        l_i_new = alpha * l_i + beta * l_ij
        
        # Scale previous accumulator
        p_scale = beta / l_i_new
        p = p * p_scale[:, None]
        acc_scale = l_i / l_i_new * alpha
        acc = acc * acc_scale[:, None]
        
        # Update accumulator
        acc += tl.dot(p.to(v.dtype), v)
        
        # Update statistics
        l_i = l_i_new
        m_i = m_i_new
    
    # Write output
    out_ptrs = Out + off_hz * stride_oh + offs_m[:, None] * stride_om + offs_d[None, :] * stride_ok
    tl.store(out_ptrs, acc)
    
    # Store statistics
    l_ptrs = L + off_hz * N_CTX + offs_m
    m_ptrs = M + off_hz * N_CTX + offs_m
    tl.store(l_ptrs, l_i)
    tl.store(m_ptrs, m_i)
```

### Performance Results

**FP16 Forward (Non-Causal):**
- 1K context: 111 TFLOPS
- 4K context: 145 TFLOPS
- 16K context: 166 TFLOPS

**FP8 Forward:**
- Generally 5-15% lower than FP16
- Better memory efficiency

**Backward Pass:**
- ~60% slower than forward
- Still faster than unfused implementations

### Memory Optimization

1. **Block-wise processing**: O(N) memory vs O(N²) for standard attention
2. **SRAM reuse**: Load K, V blocks once per Q block
3. **Online softmax**: Avoid materializing full attention matrix
4. **Causal masking**: Skip computation for masked positions

## Low-Memory Dropout

### Overview
Memory-efficient dropout using single seed instead of storing full mask tensors.

### Philox PRNG

```python
@triton.jit
def seeded_dropout(
    x_ptr, output_ptr,
    n_elements,
    p,  # Dropout probability
    seed,
    BLOCK_SIZE: tl.constexpr,
):
    # Program ID
    pid = tl.program_id(0)
    
    # Compute offsets
    block_start = pid * BLOCK_SIZE
    offsets = block_start + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements
    
    # Load input
    x = tl.load(x_ptr + offsets, mask=mask)
    
    # Generate random values using Philox algorithm
    random = tl.rand(seed, offsets)
    
    # Create keep mask
    keep_mask = random > p
    
    # Apply dropout with scaling
    output = tl.where(keep_mask, x / (1.0 - p), 0.0)
    
    # Store result
    tl.store(output_ptr + offsets, output, mask=mask)
```

### Benefits

**Memory Savings:**
- Traditional: Store mask tensor (N * sizeof(bool))
- Seeded: Store single int32 seed
- Example: 1M elements: 1MB → 4 bytes

**Checkpointing:**
- Only save seed instead of full RNG state
- Deterministic reproducibility with same seed

**Extensions:**

```python
# Per-row seeds
@triton.jit
def row_seeded_dropout(x_ptr, output_ptr, M, N, p, seed_base, BLOCK_SIZE: tl.constexpr):
    row = tl.program_id(0)
    
    # Unique seed per row
    row_seed = seed_base + row
    
    cols = tl.arange(0, BLOCK_SIZE)
    random = tl.rand(row_seed, cols)
    # ... rest of kernel
```

## Layer Normalization

### Forward Pass

```python
@triton.jit
def _layer_norm_fwd_fused(
    X, Y, W, B, Mean, Rstd,
    stride, N,
    eps,
    BLOCK_SIZE: tl.constexpr,
):
    # Row index
    row = tl.program_id(0)
    
    # Load row
    cols = tl.arange(0, BLOCK_SIZE)
    mask = cols < N
    x = tl.load(X + row * stride + cols, mask=mask, other=0.0)
    
    # Compute mean
    mean = tl.sum(x, axis=0) / N
    
    # Compute variance
    x_centered = tl.where(mask, x - mean, 0.0)
    variance = tl.sum(x_centered * x_centered, axis=0) / N
    rstd = 1 / tl.sqrt(variance + eps)
    
    # Normalize
    x_hat = x_centered * rstd
    
    # Load weights and bias
    w = tl.load(W + cols, mask=mask)
    b = tl.load(B + cols, mask=mask)
    
    # Apply affine transformation
    y = x_hat * w + b
    
    # Store output
    tl.store(Y + row * stride + cols, y, mask=mask)
    
    # Store statistics for backward pass
    tl.store(Mean + row, mean)
    tl.store(Rstd + row, rstd)
```

### Backward Pass

```python
@triton.jit
def _layer_norm_bwd_dx_fused(
    DX, DY, DW, DB,
    X, W, Mean, Rstd,
    Lock, stride, N,
    GROUP_SIZE_M: tl.constexpr,
    BLOCK_SIZE_N: tl.constexpr,
):
    row = tl.program_id(0)
    
    # Load inputs
    cols = tl.arange(0, BLOCK_SIZE_N)
    mask = cols < N
    
    x = tl.load(X + row * stride + cols, mask=mask, other=0.0)
    dy = tl.load(DY + row * stride + cols, mask=mask, other=0.0)
    w = tl.load(W + cols, mask=mask)
    
    mean = tl.load(Mean + row)
    rstd = tl.load(Rstd + row)
    
    # Compute gradient
    x_hat = (x - mean) * rstd
    wdy = w * dy
    
    # Partial gradients
    c1 = tl.sum(x_hat * wdy, axis=0) / N
    c2 = tl.sum(wdy, axis=0) / N
    
    # Input gradient
    dx = (wdy - (x_hat * c1 + c2)) * rstd
    
    # Store
    tl.store(DX + row * stride + cols, dx, mask=mask)
    
    # Accumulate weight/bias gradients (with locking)
    # ... lock management code ...
```

### Performance

**Benchmark (M=4096, float16):**
- N=8192: 987 GB/s (Triton) vs 561 GB/s (PyTorch) backward
- Triton achieves 75% improvement over PyTorch

## Grouped GEMM

### Overview
Batched matrix multiplication with variable sizes - each GEMM can have different M, N, K.

### Dynamic Scheduling

```python
@triton.jit
def grouped_matmul_kernel(
    A_ptr, B_ptr, C_ptr,
    group_m, group_n, group_k,  # Shape arrays
    lda, ldb, ldc,  # Stride arrays
    BLOCK_SIZE_M: tl.constexpr,
    BLOCK_SIZE_N: tl.constexpr,
    BLOCK_SIZE_K: tl.constexpr,
    NUM_SMS: tl.constexpr,
):
    # Get CTA ID
    tile_id = tl.program_id(0)
    
    # Determine which GEMM this tile belongs to
    group_id = 0
    tiles_per_group = []
    
    # Accumulate tile counts
    for g in range(num_groups):
        m = tl.load(group_m + g)
        n = tl.load(group_n + g)
        tiles_in_group = (m // BLOCK_SIZE_M) * (n // BLOCK_SIZE_N)
        
        if tile_id < tiles_per_group[g]:
            group_id = g
            break
        tile_id -= tiles_in_group
    
    # Load problem size for this group
    M = tl.load(group_m + group_id)
    N = tl.load(group_n + group_id)
    K = tl.load(group_k + group_id)
    
    # Standard matmul kernel logic
    # ... (similar to regular matmul)
```

### Work Distribution

**Horizontal Scaling:**
```python
# Each CTA processes tiles with stride NUM_SMS
for tile_offset in range(cta_id, total_tiles, NUM_SMS):
    process_tile(tile_offset)
```

### Use Cases

- Variable-length sequence batching
- Expert models (MoE) with different expert sizes
- Dynamic neural architecture search
- Multi-resolution image processing

## Persistent Matmul

### Persistent Kernel Pattern

```python
@triton.jit
def matmul_kernel_persistent(
    a_ptr, b_ptr, c_ptr,
    M, N, K,
    stride_am, stride_ak,
    stride_bk, stride_bn,
    stride_cm, stride_cn,
    BLOCK_SIZE_M: tl.constexpr,
    BLOCK_SIZE_N: tl.constexpr,
    BLOCK_SIZE_K: tl.constexpr,
    NUM_SMS: tl.constexpr,
):
    # Starting program ID
    start_pid = tl.program_id(0)
    
    num_pid_m = tl.cdiv(M, BLOCK_SIZE_M)
    num_pid_n = tl.cdiv(N, BLOCK_SIZE_N)
    num_tiles = num_pid_m * num_pid_n
    
    # Each program processes multiple tiles
    for tile_id in tl.range(start_pid, num_tiles, NUM_SMS):
        # Convert 1D tile_id to 2D (m, n)
        pid_m = tile_id // num_pid_n
        pid_n = tile_id % num_pid_n
        
        # Standard matmul block computation
        offs_am = (pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)) % M
        offs_bn = (pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)) % N
        # ... rest of matmul logic
```

### Benefits

1. **Reduced launch overhead**: Launch NUM_SMS blocks instead of num_tiles
2. **Better hardware utilization**: Keep SMs busy
3. **Load balancing**: Work-stealing across tiles
4. **Lower latency**: Avoid kernel re-launch overhead

### TMA (Tensor Memory Accelerator)

```python
# Create TMA descriptor
desc_a = tl.make_tensor_descriptor(
    a_ptr,
    shape=[M, K],
    strides=[stride_am, stride_ak],
    block_shape=[BLOCK_SIZE_M, BLOCK_SIZE_K],
)

# Load using TMA
@triton.jit
def tma_matmul_kernel(...):
    # Asynchronous TMA load
    a = tl.load_tensor_descriptor(desc_a, [pid_m, 0])
    # Computation overlaps with load
```

### Warp Specialization (Blackwell GPUs)

```python
# Separate warps for different operations
@triton.jit
def warp_specialized_kernel(...):
    warp_id = tl.program_id(axis=2)
    
    if warp_id == 0:
        # Prologue: Load data
        load_data()
    elif warp_id < 7:
        # Compute: Matrix multiply
        compute_matmul()
    else:
        # Epilogue: Store results
        store_results()
```

## Block-Scaled Matrix Multiplication

### FP8/FP4 Quantization

```python
@triton.jit
def block_scaled_matmul(
    a_ptr, scale_a_ptr,
    b_ptr, scale_b_ptr,
    c_ptr,
    M, N, K,
    BLOCK_SIZE_M: tl.constexpr,
    BLOCK_SIZE_N: tl.constexpr,
    BLOCK_SIZE_K: tl.constexpr,
    VEC_SIZE: tl.constexpr,  # Elements per scale factor
):
    # Standard block indices
    pid_m = tl.program_id(0)
    pid_n = tl.program_id(1)
    
    # Load quantized blocks
    offs_am = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
    offs_k = tl.arange(0, BLOCK_SIZE_K)
    
    a_ptrs = a_ptr + offs_am[:, None] * K + offs_k[None, :]
    a_block = tl.load(a_ptrs)  # FP8/FP4 data
    
    # Load scale factors (special layout)
    scale_idx = pid_m * (K // VEC_SIZE) + (offs_k // VEC_SIZE)
    scale_a = tl.load(scale_a_ptr + scale_idx)
    
    # Dequantize: a_float = a_quant * scale_a
    a_scaled = a_block.to(tl.float32) * scale_a[:, None]
    
    # Similar for B
    # ... load and scale B ...
    
    # Matrix multiply on dequantized values
    accumulator = tl.dot(a_scaled, b_scaled)
    
    # Store FP16/FP32 result
    c = accumulator.to(tl.float16)
    tl.store(c_ptrs, c)
```

### Scale Factor Layout Optimization

**Standard layout** (slow):
```
scales: (M, K // VEC_SIZE)
```

**Optimized 5D layout** (fast):
```
scales: (M // 32 // 4, K // VEC_SIZE // 4, 32, 4, 4)
```

Ensures contiguous access of 128 rows of scale factors along M axis during tensor core ops.

### Supported Formats

- **nvfp4**: NVIDIA 4-bit floating point
- **mxfp4**: OCP Microscaling 4-bit FP
- **mxfp8**: OCP Microscaling 8-bit FP
- **Mixed**: FP8 * FP8 → FP16/FP32

### Hardware Support

- NVIDIA: Compute capability 10-11 (Hopper, Blackwell)
- AMD: CDNA4 matrix cores

### Performance

Enables 2-4x memory bandwidth reduction with minimal accuracy loss for appropriate workloads.
