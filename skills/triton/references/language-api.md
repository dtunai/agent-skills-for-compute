# Triton Language API Reference

Sources:
- [Language API](https://triton-lang.org/main/python-api/triton.language.html)

## Program Intrinsics

### Program ID

```python
# Get program ID for specified axis
pid = tl.program_id(axis=0)  # axis in [0, 1, 2]

# Number of programs along axis
num_programs = tl.num_programs(axis=0)
```

## Data Creation

### Range

```python
# Create range [0, 1, ..., end-1]
offsets = tl.arange(0, BLOCK_SIZE)

# With start and end
offsets = tl.arange(start, end)
```

### Zeros and Full

```python
# Zero tensor
zeros = tl.zeros((M, N), dtype=tl.float32)

# Full tensor with value
ones = tl.full((M, N), value=1.0, dtype=tl.float32)
```

## Memory Operations

### Load

```python
# Basic load
data = tl.load(ptr + offsets)

# Masked load with default value
data = tl.load(ptr + offsets, mask=mask, other=0.0)

# Cache eviction policy
data = tl.load(ptr + offsets, cache_modifier=".ca")  # Cache all levels
data = tl.load(ptr + offsets, cache_modifier=".cg")  # Cache global
```

### Store

```python
# Basic store
tl.store(ptr + offsets, values)

# Masked store
tl.store(ptr + offsets, values, mask=mask)
```

### Atomic Operations

```python
# Atomic add
old = tl.atomic_add(ptr + offsets, values, mask=mask)

# Atomic max
old = tl.atomic_max(ptr + offsets, values)

# Atomic min
old = tl.atomic_min(ptr + offsets, values)

# Atomic xchg (exchange)
old = tl.atomic_xchg(ptr + offsets, values)

# Atomic compare-and-swap
old = tl.atomic_cas(ptr + offset, cmp, val)
```

## Math Operations

### Element-wise Functions

```python
# Exponential and logarithm
y = tl.exp(x)
y = tl.exp2(x)
y = tl.log(x)
y = tl.log2(x)

# Trigonometric
y = tl.sin(x)
y = tl.cos(x)

# Power and roots
y = tl.sqrt(x)
y = tl.rsqrt(x)  # 1/sqrt(x)

# Absolute value and sign
y = tl.abs(x)
y = tl.sign(x)

# Rounding
y = tl.floor(x)
y = tl.ceil(x)

# Clamping
y = tl.clamp(x, min_val, max_val)
```

### Activation Functions

```python
# ReLU
y = tl.maximum(x, 0)

# Sigmoid
y = 1.0 / (1.0 + tl.exp(-x))

# Tanh
y = tl.tanh(x)
```

### Reductions

```python
# Sum
total = tl.sum(x, axis=0)

# Max and min
max_val = tl.max(x, axis=0)
min_val = tl.min(x, axis=0)

# Argmax and argmin
idx = tl.argmax(x, axis=0)
idx = tl.argmin(x, axis=0)
```

### Linear Algebra

```python
# Matrix multiply (uses tensor cores)
C = tl.dot(A, B)

# With accumulator
C = tl.dot(A, B, acc=C_init)

# Transpose
B = tl.trans(A)
```

## Conditional Operations

### Where

```python
# Ternary conditional
result = tl.where(condition, true_val, false_val)

# Example: clamp
clamped = tl.where(x < min_val, min_val, 
                   tl.where(x > max_val, max_val, x))
```

## Broadcasting and Reshaping

### Broadcast

```python
# Expand dimensions for broadcasting
# Shape: (M,) -> (M, 1)
x_col = x[:, None]

# Shape: (N,) -> (1, N)
y_row = y[None, :]

# Element-wise outer operation
result = x_col + y_row  # Shape: (M, N)
```

### View and Reshape

```python
# Reshape tensor
reshaped = tl.reshape(x, (new_M, new_N))

# Split
chunks = tl.split(x, axis=0)
```

## Data Types

### Type Definitions

```python
# Floating point
tl.float8_e4m3fn
tl.float8_e5m2
tl.float16
tl.bfloat16
tl.float32
tl.float64

# Integer
tl.int1
tl.int8
tl.int16
tl.int32
tl.int64

# Unsigned integer
tl.uint8
tl.uint16
tl.uint32
tl.uint64

# Pointer
tl.pointer_type(tl.float32)
```

### Type Conversion

```python
# Cast to different type
y = x.to(tl.float32)

# Bitcast (reinterpret bits)
y = x.to(tl.int32, bitcast=True)
```

## Control Flow

### Static Assert

```python
# Compile-time assertion
tl.static_assert(BLOCK_SIZE % 16 == 0, "BLOCK_SIZE must be multiple of 16")
```

### Device Assert

```python
# Runtime assertion
tl.device_assert(x > 0, "x must be positive")
```

### Multiple Of

```python
# Hint to compiler that value is multiple of N
# Enables optimizations
offsets = tl.multiple_of(offsets, 16)
```

### Max Constancy

```python
# Hint that value stays constant across programs
# Enables optimizations
n_elements = tl.max_constancy(n_elements)
```

## Debugging

### Print

```python
# Device-side print (use sparingly)
tl.device_print("value:", x)

# Print with formatting
tl.device_print("pid =", pid, "value =", x)
```

## Advanced Features

### Inline Assembly

```python
# PTX assembly
result = tl.inline_asm_elementwise(
    "mad.lo.u32 $0, $1, $2, $3;",
    "=r,r,r,r",
    [a, b, c],
    dtype=tl.int32,
    is_pure=True
)
```

### Random Number Generation

```python
# Generate random numbers
# Note: Use with caution, not all seeds/distributions supported
rand_vals = tl.rand(seed, offsets)
```

## Utility Functions

### Division Ceiling

```python
# Ceiling division helper (in triton module, not tl)
import triton
grid_size = triton.cdiv(n_elements, BLOCK_SIZE)
```

### Next Power of 2

```python
# Round up to next power of 2
import triton
BLOCK_SIZE = triton.next_power_of_2(n_cols)
```

## Common Patterns

### Masked Load-Compute-Store

```python
# Standard pattern
offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
mask = offsets < n_elements

# Load
x = tl.load(x_ptr + offsets, mask=mask, other=0.0)

# Compute
y = tl.exp(x)

# Store
tl.store(y_ptr + offsets, y, mask=mask)
```

### 2D Indexing

```python
# Row and column indices
row_idx = tl.program_id(0)
col_offsets = tl.arange(0, BLOCK_SIZE)

# Pointer arithmetic
ptrs = base_ptr + row_idx * stride + col_offsets
data = tl.load(ptrs, mask=col_offsets < n_cols)
```

### Matrix Block Loading

```python
# Load MxN block
rows = pid_m * BLOCK_M + tl.arange(0, BLOCK_M)
cols = pid_n * BLOCK_N + tl.arange(0, BLOCK_N)

# 2D pointer array
ptrs = base_ptr + rows[:, None] * stride + cols[None, :]
mask = (rows[:, None] < M) & (cols[None, :] < N)

block = tl.load(ptrs, mask=mask)
```

### Accumulation Pattern

```python
# Initialize accumulator
acc = tl.zeros((BLOCK_M, BLOCK_N), dtype=tl.float32)

# Accumulate in loop
for k in range(0, K, BLOCK_K):
    a = tl.load(...)
    b = tl.load(...)
    acc += tl.dot(a, b)

# Convert and store
result = acc.to(tl.float16)
tl.store(result_ptr, result)
```
