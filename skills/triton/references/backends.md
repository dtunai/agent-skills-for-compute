# Triton Compiler Backends Reference

Sources:
- [GitHub Repository](https://github.com/triton-lang/triton)
- [Installation Guide](https://triton-lang.org/main/getting-started/installation.html)

## Supported Backends

Triton supports multiple GPU backends through a plugin architecture, allowing compilation to different vendor targets while maintaining a consistent frontend programming model.

### NVIDIA CUDA Backend

**Requirements:**
- CUDA 10.0 or higher
- Compute capability 7.0+ (Volta, Turing, Ampere, Hopper, Blackwell)

**Target format:**
```
cuda:<arch>:<warp-size>
```

**Examples:**
```python
# Volta (V100)
target = 'cuda:70:32'

# Turing (T4)
target = 'cuda:75:32'

# Ampere (A100)
target = 'cuda:80:32'

# Hopper (H100)
target = 'cuda:90:32'

# Blackwell
target = 'cuda:100:32'
```

**Architecture codes:**
- `70`: Volta (V100)
- `75`: Turing (T4, RTX 20xx)
- `80`: Ampere (A100, RTX 30xx)
- `86`: Ampere (RTX 30xx mobile)
- `89`: Ada Lovelace (RTX 40xx)
- `90`: Hopper (H100)
- `100`: Blackwell

### AMD HIP Backend

**Requirements:**
- ROCm 5.0 or higher
- CDNA or RDNA GPU architectures

**Target format:**
```
hip:<arch>:<warp-size>
```

**Examples:**
```python
# MI100
target = 'hip:gfx908:64'

# MI200 series
target = 'hip:gfx90a:64'

# MI300 series
target = 'hip:gfx942:64'

# MI400 series (CDNA4)
target = 'hip:gfx950:64'
```

**Architecture codes:**
- `gfx908`: CDNA1 (MI100)
- `gfx90a`: CDNA2 (MI210, MI250)
- `gfx940`: CDNA3 (MI300A)
- `gfx942`: CDNA3 (MI300X)
- `gfx950`: CDNA4 (MI400)

**Note:** AMD GPUs use warp size of 64 (vs NVIDIA's 32)

### Experimental CPU Backend

**Repository:** [triton-cpu](https://github.com/triton-lang/triton-cpu)

**Purpose:**
- Development and testing without GPU
- Prototyping kernels
- Educational use

**Limitations:**
- Performance not optimized
- May not support all features
- Not recommended for production

## Compilation Pipeline

### Stage 1: Triton-IR

**Input:** Python AST from @triton.jit function

**Process:**
- Parse decorated function
- Convert to Triton Intermediate Representation
- Machine-independent representation

**Output:** Triton-IR (MLIR dialect)

### Stage 2: Triton-GPU IR

**Input:** Triton-IR

**Process:**
- Apply optimizations
- Add GPU-specific information
- Block-level data-flow analysis
- Memory coalescing analysis

**Output:** Triton-TTGIR (MLIR dialect)

### Stage 3: LLVM-IR

**Input:** Triton-TTGIR

**Process:**
- Lower to LLVM intermediate representation
- Target-independent optimizations
- LLVM optimization passes

**Output:** LLVM-IR

### Stage 4: Backend Code Generation

**NVIDIA Path:**
```
LLVM-IR → PTX → CUBIN
```

**AMD Path:**
```
LLVM-IR → AMDGCN → HSACO
```

**PTX (NVIDIA):**
- Parallel Thread Execution
- Virtual ISA for NVIDIA GPUs
- JIT compiled to SASS by driver

**AMDGCN (AMD):**
- AMD GPU Code Object
- Native ISA for AMD GPUs

## Compilation Control

### Environment Variables

**TRITON_CACHE_DIR:**
```bash
# Set cache directory for compiled kernels
export TRITON_CACHE_DIR=/path/to/cache

# Default: ~/.triton/cache
```

**TRITON_DUMP_DIR:**
```bash
# Dump intermediate representations
export TRITON_DUMP_DIR=/tmp/triton_dump
```

**TRITON_PRINT_AUTOTUNING:**
```bash
# Print autotuning information
export TRITON_PRINT_AUTOTUNING=1
```

**TRITON_INTERPRET:**
```bash
# Use interpreter instead of compilation
export TRITON_INTERPRET=1
```

### Programmatic Compilation

```python
from triton.compiler import compile

# Compile kernel ahead-of-time
compiled = compile(
    fn=kernel,
    signature={
        'x_ptr': '*fp32',
        'y_ptr': '*fp32',
        'n': 'i32',
        'BLOCK_SIZE': 256,
    },
    device_type='cuda',
    device_capability=(8, 0),  # A100
)

# Save compiled kernel
import pickle
with open('kernel.pkl', 'wb') as f:
    pickle.dump(compiled, f)

# Load and use
with open('kernel.pkl', 'rb') as f:
    compiled = pickle.load(f)
```

## Installation

### Binary Installation

```bash
# Install from PyPI
pip install triton

# Supported Python versions: 3.10, 3.11, 3.12, 3.13, 3.14
```

### Build from Source

```bash
# Clone repository
git clone https://github.com/triton-lang/triton.git
cd triton

# Install build dependencies
pip install -r python/requirements.txt

# Build and install
pip install -e .
```

**Notes:**
- Setup script auto-downloads LLVM if not present
- Build requires C++17 compiler
- Full test suite requires GPU

### Custom LLVM

```bash
# Build with custom LLVM
export LLVM_BUILD_DIR=/path/to/llvm/build
pip install -e .
```

### GPU-Free Testing

```bash
# Run tests without GPU
make test-nogpu
```

## Backend-Specific Features

### NVIDIA-Specific

**Tensor Cores:**
- Automatic use via `tl.dot`
- FP16/BF16: 16x16x16 tiles (Ampere)
- FP8: 16x16x32 tiles (Hopper)
- TF32: 16x16x8 tiles

**Tensor Memory Accelerator (TMA):**
- Hopper (H100) and newer
- Hardware-accelerated data movement
- Async copy between global and shared memory

**Warp Specialization:**
- Blackwell GPUs
- Separate warps for load/compute/store
- Improved pipeline efficiency

### AMD-Specific

**Matrix Cores:**
- Similar to NVIDIA tensor cores
- CDNA2+: MFMA instructions
- FP16/BF16: 16x16x16 tiles
- FP32: 16x16x4 tiles

**Microscaling (CDNA4):**
- Block-scaled FP8/FP4
- 8-bit scales for 32-element groups
- Hardware-accelerated scaling

## Cross-Platform Portability

### Write Once, Run Anywhere

```python
@triton.jit
def portable_kernel(x_ptr, y_ptr, n, BLOCK_SIZE: tl.constexpr):
    # Same code runs on NVIDIA, AMD, or CPU
    pid = tl.program_id(0)
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n
    
    x = tl.load(x_ptr + offsets, mask=mask)
    y = x * 2
    tl.store(y_ptr + offsets, y, mask=mask)

# Auto-detects backend
x = torch.randn(1024, device='cuda')  # or 'hip'
y = torch.empty_like(x)

grid = (triton.cdiv(x.numel(), 256),)
portable_kernel[grid](x, y, x.numel(), BLOCK_SIZE=256)
```

### Platform Detection

```python
import torch

if torch.cuda.is_available():
    if torch.version.hip:
        backend = 'hip'
        device = 'cuda'  # PyTorch uses 'cuda' for both
    else:
        backend = 'cuda'
        device = 'cuda'
else:
    backend = 'cpu'
    device = 'cpu'

print(f"Using {backend} backend on {device}")
```

## Backend Limitations

### CUDA Backend
- Requires NVIDIA GPU
- PTX assembly may vary by driver version
- Compute capability must match GPU

### HIP Backend
- ROCm stack required
- Some features may lag CUDA backend
- Warp size difference (64 vs 32)

### CPU Backend
- Experimental status
- Limited feature support
- Poor performance compared to GPU

## Performance Considerations

### Backend-Specific Tuning

**NVIDIA:**
```python
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_SIZE': 128}, num_warps=4),  # 128 threads
        triton.Config({'BLOCK_SIZE': 256}, num_warps=8),  # 256 threads
    ],
    key=['n'],
)
@triton.jit
def nvidia_kernel(...):
    pass
```

**AMD:**
```python
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_SIZE': 256}, num_warps=4),  # 256 threads
        triton.Config({'BLOCK_SIZE': 512}, num_warps=8),  # 512 threads
    ],
    key=['n'],
)
@triton.jit
def amd_kernel(...):
    pass
```

Note: AMD's warp size of 64 affects optimal configuration.

## Compatibility Matrix

| Feature | NVIDIA CUDA | AMD HIP | CPU |
|---------|-------------|---------|-----|
| Basic kernels | ✅ | ✅ | ✅ |
| Tensor cores/MFMA | ✅ | ✅ | ❌ |
| FP8 support | ✅ (H100+) | ✅ (MI300+) | ❌ |
| TMA | ✅ (H100+) | ❌ | ❌ |
| Warp specialization | ✅ (Blackwell) | ❌ | ❌ |
| Persistent kernels | ✅ | ✅ | ❌ |
| Auto-tuning | ✅ | ✅ | ✅ |
