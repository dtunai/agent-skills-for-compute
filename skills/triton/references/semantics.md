# Triton Semantics Reference

Sources:
- [Triton Semantics](https://triton-lang.org/main/python-api/triton-semantics.html)

## NumPy Compatibility

Triton "mostly follows the semantics of NumPy with minor exceptions." The language implements array computing features aligned with NumPy conventions, with documented deviations.

## Type Promotion Rules

Triton automatically converts tensors to a common data type through a hierarchical process:

### Promotion Hierarchy

**1. Kind Priority (Highest)**
Promotes to higher-kind dtype:
```python
int32 + bfloat16 → bfloat16  # float > int
int32 + float16 → float16    # float > int
```

**2. Width Priority**
Within same kind, promotes to wider dtype:
```python
float32 + float16 → float32  # 32-bit > 16-bit
int64 + int32 → int64        # 64-bit > 32-bit
```

**3. Float16 Preference**
Mixed float16/bfloat16 promote to float16:
```python
float16 + bfloat16 → float16
```

**4. Unsigned Preference**
Mixed signedness promotes to unsigned:
```python
int32 + uint32 → uint32
int16 + uint16 → uint16
```

### Scalar Type Promotion

For operations with scalar values (numeric literals, constexpr variables):

**Rule 1: Lower-kind scalars don't trigger promotion**
```python
# Scalar is int, tensor is float
x = tl.zeros([10], dtype=tl.float16)
y = x + 1  # int scalar doesn't promote float16 tensor
# Result: float16
```

**Rule 2: Higher-kind scalars promote both operands**
```python
# Scalar is float, tensor is int
x = tl.zeros([10], dtype=tl.int32)
y = x + 1.5  # float scalar promotes int32 tensor
# Result: promoted to lowest fitting dtype for both
```

### Examples

```python
import triton.language as tl

# Example 1: Kind priority
@triton.jit
def example1():
    a = tl.zeros([10], dtype=tl.int32)
    b = tl.zeros([10], dtype=tl.float16)
    c = a + b  # dtype: float16 (float > int)

# Example 2: Width priority
@triton.jit
def example2():
    a = tl.zeros([10], dtype=tl.float32)
    b = tl.zeros([10], dtype=tl.float16)
    c = a + b  # dtype: float32 (32-bit > 16-bit)

# Example 3: Float16 preference
@triton.jit
def example3():
    a = tl.zeros([10], dtype=tl.float16)
    b = tl.zeros([10], dtype=tl.bfloat16)
    c = a + b  # dtype: float16

# Example 4: Unsigned preference
@triton.jit
def example4():
    a = tl.zeros([10], dtype=tl.int32)
    b = tl.zeros([10], dtype=tl.uint32)
    c = a + b  # dtype: uint32

# Example 5: Scalar doesn't promote
@triton.jit
def example5():
    a = tl.zeros([10], dtype=tl.float16)
    b = a + 5  # int scalar, float16 tensor
    # dtype: float16 (int < float, so no promotion)
```

## Broadcasting Rules

Triton follows NumPy broadcasting semantics for element-wise operations between tensors of different shapes.

### Broadcasting Algorithm

**Step 1: Pad shorter shape on left with ones**
```python
Shape A: (3, 4)
Shape B: (5, 3, 4)

After padding:
Shape A: (1, 3, 4)
Shape B: (5, 3, 4)
```

**Step 2: Check dimension compatibility**
Dimensions are compatible if:
- They are equal, OR
- One of them is 1

**Step 3: Output shape**
For each dimension, output size is the maximum of the two:
```python
(1, 3, 4) + (5, 3, 4) → (5, 3, 4)
```

### Broadcasting Examples

```python
@triton.jit
def broadcast_examples():
    # Example 1: Vector + Scalar
    a = tl.arange(0, 10)        # Shape: (10,)
    b = 5                        # Shape: ()
    c = a + b                    # Shape: (10,)
    
    # Example 2: Row vector + Column vector
    row = tl.arange(0, 4)[None, :]      # Shape: (1, 4)
    col = tl.arange(0, 3)[:, None]      # Shape: (3, 1)
    matrix = row + col                   # Shape: (3, 4)
    
    # Example 3: Matrix + Vector
    matrix = tl.zeros([3, 4], dtype=tl.float32)  # Shape: (3, 4)
    vector = tl.ones([4], dtype=tl.float32)      # Shape: (4,)
    # Broadcast vector to (1, 4), then to (3, 4)
    result = matrix + vector                      # Shape: (3, 4)
    
    # Example 4: Outer product via broadcasting
    x = tl.arange(0, 5)          # Shape: (5,)
    y = tl.arange(0, 3)          # Shape: (3,)
    outer = x[:, None] * y[None, :]  # Shape: (5, 3)
```

### Common Broadcasting Patterns

**Pattern 1: Add bias to rows**
```python
# Add different value to each column
matrix = tl.zeros([M, N], dtype=tl.float32)  # (M, N)
bias = tl.arange(0, N)                        # (N,)
result = matrix + bias                         # Broadcast to (M, N)
```

**Pattern 2: Scale each row**
```python
# Multiply each row by different scalar
matrix = tl.zeros([M, N], dtype=tl.float32)  # (M, N)
scales = tl.arange(0, M)                      # (M,)
result = matrix * scales[:, None]             # Broadcast to (M, N)
```

**Pattern 3: Pairwise operations**
```python
# Compute all pairwise differences
x = tl.arange(0, 5)                  # (5,)
diffs = x[:, None] - x[None, :]      # (5, 5)
# diffs[i, j] = x[i] - x[j]
```

## Key NumPy Differences

### Integer Division Semantics

**CRITICAL DIFFERENCE:** Triton implements C-style integer division, not Python-style.

**Python (NumPy) semantics:**
- `//` rounds toward minus infinity
```python
# Python
7 // 3 = 2
-7 // 3 = -3  # Rounds toward -∞
```

**C (Triton) semantics:**
- `//` rounds toward zero
```python
# Triton
7 // 3 = 2
-7 // 3 = -2  # Rounds toward 0
```

**Modulus operator:**
Triton's `%` follows corresponding C semantics:
```python
# Python
-7 % 3 = 2    # Always non-negative

# Triton (C semantics)
-7 % 3 = -1   # Sign matches dividend
```

**Exception:** Scalar-only computations use Python semantics:
```python
@triton.jit
def division_example():
    # Constexpr (compile-time) - Python semantics
    x: tl.constexpr = -7
    y: tl.constexpr = 3
    result = x // y  # = -3 (Python semantics)
    
    # Tensor operations - C semantics
    a = tl.full([10], -7, dtype=tl.int32)
    b = tl.full([10], 3, dtype=tl.int32)
    c = a // b  # = -2 (C semantics)
```

### Workaround for Python-style Division

```python
@triton.jit
def python_style_div(a, b):
    # Implement Python // using C division
    q = a // b  # C division
    r = a % b   # C modulus
    
    # Adjust if signs differ and there's a remainder
    different_signs = (a < 0) != (b < 0)
    has_remainder = r != 0
    
    result = tl.where(different_signs & has_remainder, q - 1, q)
    return result
```

## Data Type Reference

### Available Types

```python
# Floating point
tl.float8_e4m3fn   # 8-bit float (4 exp, 3 mantissa)
tl.float8_e5m2     # 8-bit float (5 exp, 2 mantissa)
tl.float16         # IEEE 16-bit float
tl.bfloat16        # Brain float 16-bit
tl.float32         # IEEE 32-bit float
tl.float64         # IEEE 64-bit float

# Integer
tl.int1            # Boolean
tl.int8            # 8-bit signed
tl.int16           # 16-bit signed
tl.int32           # 32-bit signed
tl.int64           # 64-bit signed

# Unsigned integer
tl.uint8           # 8-bit unsigned
tl.uint16          # 16-bit unsigned
tl.uint32          # 32-bit unsigned
tl.uint64          # 64-bit unsigned
```

### Type Hierarchy (for promotion)

```
Unsigned Int < Signed Int < Float8 < Float16/BFloat16 < Float32 < Float64
     ^              ^            ^              ^             ^          ^
   uint32        int32      float8_*        float16      float32    float64
   uint16        int16                      bfloat16
   uint8         int8
```

## Indexing and Slicing

### Basic Indexing

```python
@triton.jit
def indexing_example():
    # Create tensor
    x = tl.arange(0, 10)  # [0, 1, 2, ..., 9]
    
    # Single element (returns scalar)
    elem = x[5]  # 5
    
    # Slice (returns tensor)
    sub = x[2:7]  # [2, 3, 4, 5, 6]
```

### Multi-dimensional Indexing

```python
@triton.jit
def multidim_indexing():
    # 2D tensor
    matrix = tl.arange(0, 12).reshape(3, 4)
    # [[0, 1, 2, 3],
    #  [4, 5, 6, 7],
    #  [8, 9, 10, 11]]
    
    # Single element
    elem = matrix[1, 2]  # 6
    
    # Row slice
    row = matrix[1, :]   # [4, 5, 6, 7]
    
    # Column slice
    col = matrix[:, 2]   # [2, 6, 10]
    
    # Submatrix
    sub = matrix[0:2, 1:3]  # [[1, 2], [5, 6]]
```

### Advanced Indexing

```python
@triton.jit
def advanced_indexing():
    # Boolean masking
    x = tl.arange(0, 10)
    mask = x > 5
    # Note: Triton doesn't support x[mask] directly
    # Use tl.where instead
    filtered = tl.where(mask, x, 0)
    
    # Conditional selection
    y = tl.where(x % 2 == 0, x * 2, x * 3)
```

## Best Practices

1. **Explicit type conversion**: Don't rely on automatic promotion
   ```python
   # Bad: Relies on promotion
   result = int_tensor + float_tensor
   
   # Good: Explicit conversion
   result = int_tensor.to(tl.float32) + float_tensor
   ```

2. **Be aware of integer division**: Use C semantics or implement Python-style
   ```python
   # Be explicit about division semantics
   # C-style (Triton default)
   c_div = a // b
   
   # Python-style (implement manually if needed)
   py_div = python_style_div(a, b)
   ```

3. **Use broadcasting carefully**: Ensure shapes are compatible
   ```python
   # Check shapes before broadcasting
   tl.static_assert(M == tensor.shape[0], "Shape mismatch")
   ```

4. **Prefer explicit reshaping**: Make broadcasting intentions clear
   ```python
   # Bad: Implicit broadcasting
   result = matrix + vector
   
   # Good: Explicit reshaping
   result = matrix + vector[None, :]
   ```
