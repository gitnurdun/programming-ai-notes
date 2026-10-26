# Building Models with PyTorch - 30+ Questions and Answers

## 1. What is nn.Module in PyTorch?

**Answer:** `nn.Module` is the base class for all neural network modules in PyTorch. It provides functionality for:
- Registering parameters and submodules
- Moving parameters between CPU/GPU
- Saving and loading model states
- Implementing forward pass with `forward()` method
All neural network components inherit from this class.

## 2. How do you create a custom neural network by subclassing nn.Module?

**Answer:**
```python
import torch
import torch.nn as nn

class CustomNet(nn.Module):
    def __init__(self):
        super(CustomNet, self).__init__()
        self.fc1 = nn.Linear(784, 128)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(128, 10)
        
    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = self.fc2(x)
        return x
```

## 3. What is the difference between nn.Sequential and custom nn.Module?

**Answer:** `nn.Sequential` is a container module that applies modules in sequence, good for simple architectures. Custom `nn.Module` provides more flexibility for complex architectures with branching, skip connections, or conditional operations. Sequential is easier for linear architectures, while custom modules allow for more complex logic.

## 4. How do you define and use parameters in a custom nn.Module?

**Answer:** Parameters are automatically registered when you assign an `nn.Parameter` or `nn.Module` subclass to a module attribute. For example:
```python
self.weight = nn.Parameter(torch.randn(10, 5))  # Registered as parameter
self.layer = nn.Linear(10, 5)  # Automatically registers its parameters
self.register_parameter('custom_param', nn.Parameter(torch.zeros(10)))  # Explicit registration
```

## 5. How do you access model parameters and buffers?

**Answer:**
```python
model = CustomNet()

# Access parameters
for param in model.parameters():
    print(param.shape)

# Access named parameters
for name, param in model.named_parameters():
    print(f"{name}: {param.shape}")

# Access buffers (like running_mean in BatchNorm)
for buffer in model.buffers():
    print(buffer)

# Access named buffers
for name, buffer in model.named_buffers():
    print(f"{name}: {buffer}")
```

## 6. What is the purpose of the forward() method?

**Answer:** The `forward()` method defines the forward pass of the neural network - how input data flows through the layers to produce output. When you call `model(input_tensor)`, PyTorch internally calls the `forward()` method. This method should implement the computational graph of your network.

## 7. How do you implement different behavior during training and evaluation?

**Answer:** Use `model.train()` and `model.eval()` to set the model's mode:
```python
# During training
model.train()  # Enables dropout, batch norm updates
output = model(input)

# During evaluation
model.eval()   # Disables dropout, uses running stats in batch norm
with torch.no_grad():
    output = model(input)
```

## 8. What are the common layer types in PyTorch?

**Answer:**
- `nn.Linear`: Fully connected (dense) layer
- `nn.Conv2d`: 2D convolutional layer
- `nn.Conv1d`, `nn.Conv3d`: 1D and 3D convolutions
- `nn.LSTM`, `nn.GRU`: Recurrent layers
- `nn.BatchNorm1d/2d/3d`: Batch normalization
- `nn.Dropout`: Dropout layer
- `nn.MaxPool2d`: Max pooling layer

## 9. How do you implement a custom layer with learnable parameters?

**Answer:**
```python
class CustomLinear(nn.Module):
    def __init__(self, in_features, out_features):
        super(CustomLinear, self).__init__()
        self.weight = nn.Parameter(torch.randn(out_features, in_features))
        self.bias = nn.Parameter(torch.zeros(out_features))
        
    def forward(self, input):
        return torch.mm(input, self.weight.t()) + self.bias
```

## 10. What is the difference between nn.Parameter and regular tensors?

**Answer:** `nn.Parameter` is a tensor that is automatically registered as a learnable parameter when assigned to an `nn.Module`. Regular tensors are not registered automatically. Parameters are included in `model.parameters()` and are updated during training. `nn.Parameter` always has `requires_grad=True` by default.

## 11. How do you implement a residual connection in PyTorch?

**Answer:**
```python
class ResidualBlock(nn.Module):
    def __init__(self, dim):
        super(ResidualBlock, self).__init__()
        self.linear1 = nn.Linear(dim, dim)
        self.linear2 = nn.Linear(dim, dim)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        residual = x
        out = self.relu(self.linear1(x))
        out = self.linear2(out)
        out += residual  # Residual connection
        out = self.relu(out)
        return out
```

## 12. How do you handle variable input sizes in neural networks?

**Answer:** Use adaptive pooling layers like `nn.AdaptiveAvgPool2d` or `nn.AdaptiveMaxPool2d` that output fixed-size features regardless of input size. For fully connected layers after convolutions, calculate the required size dynamically or use global pooling:
```python
x = nn.AdaptiveAvgPool2d((1, 1))(x)  # Always outputs 1x1
x = x.view(x.size(0), -1)  # Flatten for linear layer
```

## 13. What is the purpose of register_buffer()?

**Answer:** `register_buffer()` registers a tensor as a buffer in the module. Buffers are not parameters but are part of the module's state (included in `state_dict`). They're useful for tensors like running statistics in BatchNorm that should be saved/loaded with the model but are not updated by the optimizer.

## 14. How do you implement a model with conditional layers?

**Answer:**
```python
class ConditionalNet(nn.Module):
    def __init__(self, use_skip_connection=True):
        super(ConditionalNet, self).__init__()
        self.layer1 = nn.Linear(10, 20)
        self.layer2 = nn.Linear(20, 30)
        self.skip = nn.Linear(10, 30) if use_skip_connection else None
        self.relu = nn.ReLU()
        
    def forward(self, x):
        out = self.relu(self.layer1(x))
        out = self.layer2(out)
        
        if self.skip is not None:
            out += self.skip(x)
        return out
```

## 15. How do you create parameter-free layers like activation functions?

**Answer:** Parameter-free layers like activation functions are created by instantiating them as attributes in your module:
```python
class Net(nn.Module):
    def __init__(self):
        super(Net, self).__init__()
        self.linear = nn.Linear(10, 5)
        self.relu = nn.ReLU()  # No parameters
        self.sigmoid = nn.Sigmoid()  # No parameters
        
    def forward(self, x):
        return self.sigmoid(self.relu(self.linear(x)))
```

## 16. What is torch.nn.functional and how is it different from nn.Module?

**Answer:** `torch.nn.functional` (often imported as `F`) contains functional versions of neural network operations that are stateless and don't have parameters. Unlike `nn.Module`, these functions are called directly and don't maintain state. Use `F` for operations without learnable parameters (like ReLU) or when you need more flexibility in forward pass.

## 17. How do you implement a model with shared weights?

**Answer:**
```python
class SharedWeightNet(nn.Module):
    def __init__(self):
        super(SharedWeightNet, self).__init__()
        self.shared_layer = nn.Linear(10, 5)  # Single layer instance
        self.output1 = nn.Linear(5, 2)
        self.output2 = nn.Linear(5, 2)
        
    def forward(self, x1, x2):
        # Both inputs go through the same shared layer
        shared_out1 = self.shared_layer(x1)
        shared_out2 = self.shared_layer(x2)  # Same weights
        return self.output1(shared_out1), self.output2(shared_out2)
```

## 18. How do you implement multi-output models?

**Answer:**
```python
class MultiOutputNet(nn.Module):
    def __init__(self):
        super(MultiOutputNet, self).__init__()
        self.shared = nn.Sequential(
            nn.Linear(784, 128),
            nn.ReLU(),
            nn.Linear(128, 64),
            nn.ReLU()
        )
        self.classifier = nn.Linear(64, 10)  # For classification
        self.regressor = nn.Linear(64, 1)    # For regression
        
    def forward(self, x):
        features = self.shared(x)
        classification = self.classifier(features)
        regression = self.regressor(features)
        return classification, regression
```

## 19. What is the difference between state_dict() and named_parameters()?

**Answer:** `state_dict()` returns an ordered dictionary containing all parameters and persistent buffers of the module, including those of child modules. `named_parameters()` returns only the parameters with their names. `state_dict()` is used for saving/loading models, while `named_parameters()` is often used for inspecting or customizing parameter updates.

## 20. How do you freeze specific layers in a model?

**Answer:**
```python
# Freeze all parameters
for param in model.parameters():
    param.requires_grad = False

# Or freeze specific layer
for param in model.layer1.parameters():
    param.requires_grad = False

# Or freeze based on name
for name, param in model.named_parameters():
    if 'layer1' in name:
        param.requires_grad = False

# Check if parameters require grad
for name, param in model.named_parameters():
    print(f"{name}: {param.requires_grad}")
```

## 21. How do you implement a model with different learning rates for different layers?

**Answer:**
```python
optimizer = torch.optim.Adam([
    {'params': model.layer1.parameters(), 'lr': 1e-4},  # Lower LR for early layers
    {'params': model.layer2.parameters(), 'lr': 1e-3},  # Higher LR for later layers
    {'params': model.classifier.parameters(), 'lr': 1e-2}  # Highest LR for classifier
])
```

## 22. What are hooks in PyTorch and how do you use them?

**Answer:** Hooks allow you to register functions that are executed at specific points during forward/backward passes. Forward hooks are called after forward(), backward hooks are called during backward():
```python
def hook_fn(module, input, output):
    print(f"Module: {module.__class__.__name__}")
    print(f"Input shape: {input[0].shape}")
    print(f"Output shape: {output.shape}")

# Register hook
handle = model.layer1.register_forward_hook(hook_fn)
output = model(input)
handle.remove()  # Remove when done
```

## 23. How do you implement gradient clipping in PyTorch?

**Answer:**
```python
# Method 1: Clip gradients by norm
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

# Method 2: Clip gradients by value
torch.nn.utils.clip_grad_value_(model.parameters(), clip_value=0.5)

# In training loop:
optimizer.zero_grad()
loss.backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
```

## 24. How do you implement a model with custom initialization?

**Answer:**
```python
def init_weights(m):
    if isinstance(m, nn.Linear):
        torch.nn.init.xavier_uniform_(m.weight)
        m.bias.data.fill_(0.01)
    elif isinstance(m, nn.Conv2d):
        torch.nn.init.kaiming_normal_(m.weight, mode='fan_out', nonlinearity='relu')

model = CustomNet()
model.apply(init_weights)  # Apply initialization to all layers
```

## 25. What is the difference between torch.cat and torch.stack in model building?

**Answer:** `torch.cat` concatenates tensors along an existing dimension, preserving the number of dimensions. `torch.stack` creates a new dimension by concatenating tensors along it. For example, if you have two tensors of shape (3, 4), `cat` along dim=0 gives (6, 4), while `stack` along dim=0 gives (2, 3, 4).

## 26. How do you implement attention mechanisms in PyTorch?

**Answer:**
```python
class SimpleAttention(nn.Module):
    def __init__(self, hidden_size):
        super(SimpleAttention, self).__init__()
        self.hidden_size = hidden_size
        self.attn = nn.Linear(hidden_size, 1)
        
    def forward(self, x):
        # x shape: (batch_size, seq_len, hidden_size)
        attn_weights = torch.softmax(self.attn(x), dim=1)
        # Apply attention weights
        context = torch.sum(attn_weights * x, dim=1)
        return context, attn_weights
```

## 27. How do you handle tensor shapes and dimensions in complex models?

**Answer:** Always track tensor shapes by printing them during development:
```python
def forward(self, x):
    print(f"Input shape: {x.shape}")  # Debug prints
    x = self.conv1(x)
    print(f"After conv1: {x.shape}")
    x = self.pool(x)
    print(f"After pool: {x.shape}")
    x = x.view(x.size(0), -1)  # Flatten
    print(f"After flatten: {x.shape}")
    x = self.fc(x)
    print(f"Output shape: {x.shape}")
    return x
```

## 28. How do you implement a model with multiple inputs?

**Answer:**
```python
class MultiInputNet(nn.Module):
    def __init__(self):
        super(MultiInputNet, self).__init__()
        self.text_branch = nn.Sequential(
            nn.Linear(100, 64),
            nn.ReLU(),
            nn.Linear(64, 32)
        )
        self.image_branch = nn.Sequential(
            nn.Linear(784, 64),
            nn.ReLU(),
            nn.Linear(64, 32)
        )
        self.classifier = nn.Linear(64, 10)  # Combined features
        
    def forward(self, text_input, image_input):
        text_features = self.text_branch(text_input)
        image_features = self.image_branch(image_input)
        combined = torch.cat([text_features, image_features], dim=1)
        return self.classifier(combined)
```

## 29. What is the purpose of torch.no_grad() in model evaluation?

**Answer:** `torch.no_grad()` disables gradient computation, reducing memory usage and speeding up execution during inference. It should be used during evaluation, testing, or when you don't need gradients:
```python
model.eval()
with torch.no_grad():
    output = model(input)
    loss = criterion(output, target)
```

## 30. How do you implement a complex model architecture with skip connections?

**Answer:**
```python
class DenseBlock(nn.Module):
    def __init__(self, num_layers, in_channels, growth_rate):
        super(DenseBlock, self).__init__()
        self.layers = nn.ModuleList()
        channels = in_channels
        
        for i in range(num_layers):
            layer = nn.Sequential(
                nn.BatchNorm2d(channels),
                nn.ReLU(inplace=True),
                nn.Conv2d(channels, growth_rate, kernel_size=3, padding=1)
            )
            self.layers.append(layer)
            channels += growth_rate
            
    def forward(self, x):
        features = [x]
        for layer in self.layers:
            # Concatenate with all previous features
            inputs = torch.cat(features, dim=1)
            output = layer(inputs)
            features.append(output)
        return torch.cat(features, dim=1)
```

## 31. How do you implement a model that can work with both training and inference modes?

**Answer:**
```python
class DualModeNet(nn.Module):
    def __init__(self):
        super(DualModeNet, self).__init__()
        self.feature_extractor = nn.Sequential(
            nn.Linear(100, 64),
            nn.ReLU(),
            nn.Dropout(0.5)  # Only active during training
        )
        self.classifier = nn.Linear(64, 10)
        self.return_features = False  # Inference flag
        
    def forward(self, x):
        features = self.feature_extractor(x)
        
        if self.return_features:
            return features  # Return features for inference
        else:
            return self.classifier(features)  # Return predictions during training
    
    def set_return_features(self, value):
        self.return_features = value
```