# PyTorch Basics - 30+ Questions and Answers

## 1. What is PyTorch and what are its main advantages?

**Answer:** PyTorch is an open-source deep learning framework developed by Facebook's AI Research lab. Its main advantages include:
- Dynamic computation graph (define-by-run) that allows for more intuitive and flexible model development
- Pythonic interface that integrates seamlessly with the Python ecosystem
- Strong GPU acceleration support for faster computations
- Excellent debugging capabilities through standard Python tools
- Comprehensive support for tensor computations and deep neural networks

## 2. What is a tensor in PyTorch?

**Answer:** A tensor in PyTorch is a multi-dimensional matrix containing elements of a single data type. It's the fundamental building block for all operations in PyTorch, similar to NumPy arrays but with GPU support. Tensors can be 0-dimensional (scalar), 1-dimensional (vector), 2-dimensional (matrix), or higher-dimensional.

## 3. How do you create tensors in PyTorch?

**Answer:** Tensors can be created in several ways:
- `torch.tensor([1, 2, 3])` - from Python lists
- `torch.zeros(3, 4)` - tensor of zeros
- `torch.ones(3, 4)` - tensor of ones
- `torch.randn(3, 4)` - tensor with random values from normal distribution
- `torch.arange(0, 10)` - tensor with evenly spaced values
- `torch.linspace(0, 10, 5)` - tensor with values linearly spaced between start and end

## 4. How do you check the shape, data type, and device of a tensor?

**Answer:**
```python
import torch

x = torch.randn(3, 4)
print(x.shape)    # torch.Size([3, 4])
print(x.dtype)    # torch.float32
print(x.device)   # cpu or cuda:0
```

## 5. What is the difference between torch.Tensor() and torch.tensor()?

**Answer:** `torch.Tensor()` creates a tensor with default dtype (float32) and initializes with random values, while `torch.tensor()` creates a tensor from data and infers the dtype. `torch.tensor()` is generally preferred because it's more explicit about the data type.

## 6. How do you move tensors between CPU and GPU?

**Answer:**
```python
# Move to GPU
x = x.cuda() or x.to('cuda')
# Move to CPU
x = x.cpu() or x.to('cpu')
# Move to specific GPU
x = x.to('cuda:1')
```

## 7. What is the autograd package in PyTorch?

**Answer:** Autograd is PyTorch's automatic differentiation engine that computes gradients for tensor operations. It tracks operations on tensors that have `requires_grad=True` and builds a computational graph to compute gradients using the chain rule during backpropagation.

## 8. How do you enable gradient computation for a tensor?

**Answer:** You can enable gradient computation by setting `requires_grad=True` when creating the tensor or using the `requires_grad_()` method:
```python
x = torch.randn(3, 4, requires_grad=True)
# or
x = torch.randn(3, 4)
x.requires_grad_(True)
```

## 9. How do you compute gradients using backward()?

**Answer:** After performing operations on tensors with `requires_grad=True`, call `loss.backward()` on the final scalar output to compute gradients for all tensors in the computational graph with `requires_grad=True`. The gradients are stored in the `.grad` attribute of each tensor.

## 10. What is the difference between torch.Tensor and torch.tensor?

**Answer:** `torch.Tensor` is an alias for `torch.FloatTensor` and creates tensors with default float32 type, while `torch.tensor` creates a new tensor from data and infers the appropriate data type. `torch.tensor` is the recommended method.

## 11. What is the detach() method used for?

**Answer:** The `detach()` method creates a new tensor that doesn't require gradients and is detached from the computational graph. This is useful when you want to prevent gradient computation during certain operations or when converting tensors to NumPy arrays.

## 12. How do you convert a PyTorch tensor to a NumPy array?

**Answer:**
```python
import torch
import numpy as np

x = torch.randn(3, 4)
# Convert to numpy (tensor must be on CPU and not require gradients)
numpy_array = x.detach().numpy()

# If tensor requires gradients, detach first
y = torch.randn(3, 4, requires_grad=True)
numpy_array = y.detach().numpy()
```

## 13. What is the difference between in-place and out-of-place operations?

**Answer:** In-place operations modify the original tensor and are denoted by a trailing underscore (e.g., `add_()`, `mul_()`), while out-of-place operations return a new tensor (e.g., `add()`, `mul()`). In-place operations save memory but prevent gradient computation in some cases.

## 14. How do you perform basic mathematical operations on tensors?

**Answer:**
```python
a = torch.randn(3, 4)
b = torch.randn(3, 4)

# Addition
c = a + b or torch.add(a, b)
# Subtraction
c = a - b or torch.sub(a, b)
# Multiplication
c = a * b or torch.mul(a, b)
# Division
c = a / b or torch.div(a, b)
# In-place operations
a.add_(b)  # a is modified
```

## 15. What is torch.no_grad() context manager?

**Answer:** `torch.no_grad()` is a context manager that disables gradient computation for all operations within its block. This is useful for inference, validation, or when you want to perform tensor operations without tracking gradients, saving memory and computation time.

## 16. How do you reshape tensors in PyTorch?

**Answer:**
```python
x = torch.randn(2, 6)

# Using view() - creates a new view of the same data
y = x.view(3, 4)
z = x.view(-1)  # Flatten to 1D (-1 infers the size)

# Using reshape() - can copy data if necessary
w = x.reshape(3, 4)

# Using squeeze() and unsqueeze()
a = torch.randn(1, 3, 1, 4)
b = a.squeeze()  # Remove dimensions of size 1
c = a.unsqueeze(1)  # Add dimension at index 1
```

## 17. What is the difference between view() and reshape()?

**Answer:** `view()` can only be used when the tensor is contiguous in memory and creates a new view of the same data without copying. `reshape()` can work with non-contiguous tensors and may create a copy if needed. Use `view()` when you're sure the tensor is contiguous for better performance.

## 18. How do you index and slice tensors in PyTorch?

**Answer:**
```python
x = torch.randn(4, 5, 6)

# Basic indexing
y = x[0]        # First slice along first dimension
z = x[0, 1]     # First slice of first, second slice of second
w = x[0, 1, 2]  # Element at [0, 1, 2]

# Slicing
a = x[:, 0:2]    # All first dim, first two of second dim
b = x[1:3, :, :] # Second and third of first dim
c = x[::2]       # Every other element along first dim
```

## 19. What is broadcasting in PyTorch?

**Answer:** Broadcasting is PyTorch's ability to perform operations between tensors of different shapes by automatically expanding the smaller tensor to match the larger one. Two tensors are broadcastable if for each dimension, either the dimensions are equal, one of them is 1, or one of them doesn't exist.

## 20. How do you concatenate tensors in PyTorch?

**Answer:**
```python
a = torch.randn(2, 3)
b = torch.randn(2, 3)

# Along dimension 0 (rows)
c = torch.cat([a, b], dim=0)  # Shape: (4, 3)
# Along dimension 1 (columns)
d = torch.cat([a, b], dim=1)  # Shape: (2, 6)

# Stack creates a new dimension
e = torch.stack([a, b], dim=0)  # Shape: (2, 2, 3)
```

## 21. What is torch.nn.Parameter?

**Answer:** `torch.nn.Parameter` is a tensor that is automatically registered as a parameter when assigned to an `nn.Module`. It always has `requires_grad=True` by default and is included in the module's `parameters()` iterator, making it part of the model's learnable parameters.

## 22. How do you create random tensors with different distributions?

**Answer:**
```python
# Normal distribution
x = torch.randn(3, 4)  # Mean=0, Std=1
y = torch.normal(mean=0.0, std=1.0, size=(3, 4))

# Uniform distribution
z = torch.rand(3, 4)  # [0, 1)
w = torch.uniform_(torch.zeros(3, 4), 0, 1)  # In-place uniform

# Integer random
a = torch.randint(0, 10, (3, 4))  # Integers [0, 10)
```

## 23. What is torch.manual_seed() used for?

**Answer:** `torch.manual_seed(seed)` sets the seed for PyTorch's random number generator for reproducible results. Setting the same seed before operations will produce the same random values across runs, which is crucial for reproducible experiments.

## 24. How do you check if CUDA is available and use it?

**Answer:**
```python
import torch

# Check CUDA availability
if torch.cuda.is_available():
    device = torch.device('cuda')
    print(f'CUDA available. GPU count: {torch.cuda.device_count()}')
    print(f'Current GPU: {torch.cuda.current_device()}')
    print(f'GPU name: {torch.cuda.get_device_name(0)}')
else:
    device = torch.device('cpu')
    print('CUDA not available, using CPU')

# Move tensor to device
x = torch.randn(3, 4).to(device)
```

## 25. What are the memory considerations when working with tensors?

**Answer:** Important considerations include:
- Tensors on GPU are stored in GPU memory, which is limited compared to CPU RAM
- Use `torch.no_grad()` during inference to reduce memory usage
- Delete large tensors explicitly with `del tensor` when no longer needed
- Use `torch.cuda.empty_cache()` to clear GPU cache when needed
- Be careful with in-place operations that might affect gradient computation

## 26. How do you perform operations with different data types?

**Answer:**
```python
a = torch.tensor([1, 2, 3], dtype=torch.float32)
b = torch.tensor([4, 5, 6], dtype=torch.float64)

# Convert to same type before operation
result = a.to(torch.float64) + b  # or b.to(torch.float32) + a
# Or specify output type
result = torch.add(a, b, out=torch.empty_like(a, dtype=torch.float64))
```

## 27. What is tensor contiguity and why does it matter?

**Answer:** A contiguous tensor has elements stored in a continuous block of memory following the row-major order. Operations like `transpose()` or `permute()` can make tensors non-contiguous. Contiguous tensors are necessary for operations like `view()`. Use `tensor.contiguous()` to ensure contiguity.

## 28. How do you handle tensor operations with different shapes?

**Answer:** PyTorch supports broadcasting rules similar to NumPy. Tensors are compatible for operations if dimensions are equal or one of them is 1. For incompatible shapes, you might need to reshape, expand, or use operations like `unsqueeze()` to make them compatible.

## 29. What is torch.clamp() and how is it used?

**Answer:** `torch.clamp()` restricts tensor values to a specified range. It can be used for gradient clipping or limiting values:
```python
x = torch.tensor([-2.0, -1.0, 0.0, 1.0, 2.0])
clamped = torch.clamp(x, min=-1.0, max=1.0)  # [-1.0, -1.0, 0.0, 1.0, 1.0]
```

## 30. How do you perform element-wise operations on tensors?

**Answer:**
```python
x = torch.tensor([1.0, 2.0, 3.0])
y = torch.tensor([4.0, 5.0, 6.0])

# Common element-wise operations
add = torch.add(x, y)          # or x + y
mul = torch.mul(x, y)          # or x * y
pow = torch.pow(x, 2)          # or x ** 2
exp = torch.exp(x)
log = torch.log(x)
sqrt = torch.sqrt(x)
abs = torch.abs(x)
```

## 31. What is torch.where() and how is it used?

**Answer:** `torch.where()` selects elements from one of two tensors based on a condition:
```python
x = torch.tensor([1, 2, 3])
y = torch.tensor([4, 5, 6])
cond = torch.tensor([True, False, True])
result = torch.where(cond, x, y)  # [1, 5, 3]
# It can also be used to find indices where condition is true
indices = torch.where(x > 2)  # Returns indices where x > 2
```