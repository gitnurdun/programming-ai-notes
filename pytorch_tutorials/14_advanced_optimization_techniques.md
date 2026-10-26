# Advanced Optimization Techniques in PyTorch - 30+ Questions and Answers

## 1. What are advanced optimization techniques beyond standard SGD and Adam?

**Answer:** Advanced optimization techniques in PyTorch go beyond basic optimizers like SGD and Adam to include methods that improve convergence, stability, and performance:

1. **Adaptive Learning Rate Methods:**
   - **AdamW:** Adam with decoupled weight decay
   - **RAdam:** Adam with rectified update directions
   - **Lookahead:** Maintains fast and slow weights
   - **NovoGrad:** Gradient normalization-based optimizer

2. **Second-Order Methods:**
   - **LBFGS:** Limited-memory Broyden–Fletcher–Goldfarb–Shanno
   - **K-FAC:** Kronecker-Factored Approximate Curvature
   - **Shampoo:** Second-order optimizer with preconditioning

3. **Gradient-Based Methods:**
   - **Gradient Clipping:** Prevents exploding gradients
   - **Gradient Accumulation:** Simulates larger batch sizes
   - **Mixed Precision Training:** Uses FP16 for efficiency

4. **Specialized Techniques:**
   - **Cyclical Learning Rates:** Varies learning rate cyclically
   - **Cosine Annealing:** Schedules learning rate with cosine function
   - **One-Cycle Policy:** Single cycle of learning rate changes

These advanced techniques help with training deep networks more efficiently and effectively.

## 2. How do you implement and use the AdamW optimizer?

**Answer:**
```python
import torch
import torch.optim as optim
import torch.nn as nn

class AdamWExample:
    def __init__(self, model, lr=0.001, weight_decay=0.01):
        # AdamW with decoupled weight decay
        self.optimizer = optim.AdamW(
            model.parameters(),
            lr=lr,
            weight_decay=weight_decay,  # Decoupled from gradient updates
            betas=(0.9, 0.999),
            eps=1e-8
        )
        self.model = model
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step(self, inputs, targets):
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        self.optimizer.step()
        
        return loss.item()

# Example usage
class SimpleModel(nn.Module):
    def __init__(self):
        super(SimpleModel, self).__init__()
        self.fc1 = nn.Linear(784, 256)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(256, 10)
    
    def forward(self, x):
        x = self.fc1(x)
        x = self.relu(x)
        x = self.fc2(x)
        return x

# Compare Adam vs AdamW
model_adam = SimpleModel()
model_adamw = SimpleModel()

# Adam optimizer (weight decay coupled with gradient)
optimizer_adam = optim.Adam(model_adam.parameters(), lr=0.001, weight_decay=0.01)

# AdamW optimizer (weight decay decoupled)
optimizer_adamw = optim.AdamW(model_adamw.parameters(), lr=0.001, weight_decay=0.01)

print("Adam optimizer:")
print(f"  Weight decay is applied to gradients: True")
print(f"  Parameters: lr=0.001, weight_decay=0.01")

print("\nAdamW optimizer:")
print(f"  Weight decay is decoupled from gradients: True")
print(f"  Parameters: lr=0.001, weight_decay=0.01")

# Manual implementation of AdamW
class ManualAdamW(optim.Optimizer):
    def __init__(self, params, lr=1e-3, betas=(0.9, 0.999), eps=1e-8, weight_decay=1e-2):
        defaults = dict(lr=lr, betas=betas, eps=eps, weight_decay=weight_decay)
        super(ManualAdamW, self).__init__(params, defaults)
    
    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()
        
        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue
                
                grad = p.grad.data
                if grad.is_sparse:
                    raise RuntimeError('AdamW does not support sparse gradients')
                
                state = self.state[p]
                
                # State initialization
                if len(state) == 0:
                    state['step'] = 0
                    state['exp_avg'] = torch.zeros_like(p.data)
                    state['exp_avg_sq'] = torch.zeros_like(p.data)
                
                exp_avg, exp_avg_sq = state['exp_avg'], state['exp_avg_sq']
                beta1, beta2 = group['betas']
                
                state['step'] += 1
                
                # Decay the first and second moment running average coefficient
                exp_avg.mul_(beta1).add_(grad, alpha=1 - beta1)
                exp_avg_sq.mul_(beta2).addcmul_(grad, grad, value=1 - beta2)
                
                denom = exp_avg_sq.sqrt().add_(group['eps'])
                
                bias_correction1 = 1 - beta1 ** state['step']
                bias_correction2 = 1 - beta2 ** state['step']
                
                step_size = group['lr'] * (bias_correction2.sqrt() / bias_correction1)
                
                # Apply weight decay AFTER the gradient update
                p.data.addcdiv_(exp_avg, denom, value=-step_size)
                p.data.add_(p.data, alpha=-group['weight_decay'] * group['lr'])
        
        return loss

# Example of when AdamW is preferred
print("\nWhen to use AdamW:")
print("1. When training with high weight decay (e.g., BERT, ResNet)")
print("2. For better generalization with decoupled weight decay")
print("3. When you want cleaner regularization")
print("4. In transformer models and large-scale training")
```

## 3. How do you implement and use the Lookahead optimizer?

**Answer:**
```python
import torch
import torch.optim as optim
import torch.nn as nn

class Lookahead(optim.Optimizer):
    """
    Lookahead optimizer: Maintains two sets of weights - fast and slow.
    Fast weights are updated frequently, slow weights are updated less often
    by taking steps towards the fast weights.
    """
    def __init__(self, optimizer, k=5, alpha=0.5):
        self.optimizer = optimizer
        self.k = k  # Number of fast steps before slow update
        self.alpha = alpha  # Interpolation factor
        self.defaults = {'k': k, 'alpha': alpha}
        
        # Track step count
        self.step_counter = 0
        
        # Store slow weights
        self.slow_weights = {}
        for group in optimizer.param_groups:
            for p in group['params']:
                self.slow_weights[id(p)] = p.data.clone().detach()
    
    def step(self, closure=None):
        # Fast optimizer step
        loss = self.optimizer.step(closure)
        
        self.step_counter += 1
        
        # Update slow weights every k steps
        if self.step_counter % self.k == 0:
            for group in self.optimizer.param_groups:
                for p in group['params']:
                    # Update slow weight: slow = alpha * fast + (1 - alpha) * slow
                    slow_weight = self.slow_weights[id(p)]
                    slow_weight.mul_(1 - self.alpha).add_(p.data, alpha=self.alpha)
                    
                    # Update fast weight to be: alpha * fast + (1 - alpha) * slow
                    p.data.mul_(self.alpha).add_(slow_weight, alpha=1 - self.alpha)
        
        return loss
    
    def state_dict(self):
        """Return optimizer state"""
        fast_state = self.optimizer.state_dict()
        slow_state = {'slow_weights': self.slow_weights, 'step_counter': self.step_counter}
        return {'fast_state': fast_state, 'slow_state': slow_state}
    
    def load_state_dict(self, state_dict):
        """Load optimizer state"""
        self.optimizer.load_state_dict(state_dict['fast_state'])
        self.slow_weights = state_dict['slow_state']['slow_weights']
        self.step_counter = state_dict['slow_state']['step_counter']

# Example usage
class LookaheadExample:
    def __init__(self, model, base_optimizer='adam', k=5, alpha=0.5):
        if base_optimizer == 'adam':
            base_opt = optim.Adam(model.parameters(), lr=0.001)
        elif base_optimizer == 'sgd':
            base_opt = optim.SGD(model.parameters(), lr=0.01, momentum=0.9)
        else:
            base_opt = optim.Adam(model.parameters(), lr=0.001)
        
        self.lookahead = Lookahead(base_opt, k=k, alpha=alpha)
        self.model = model
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step(self, inputs, targets):
        self.lookahead.zero_grad()
        
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        self.lookahead.step()
        
        return loss.item()

# Comparison with regular optimizer
def compare_lookahead():
    model_regular = SimpleModel()
    model_lookahead = SimpleModel()
    
    # Regular optimizer
    optimizer_regular = optim.Adam(model_regular.parameters(), lr=0.001)
    
    # Lookahead optimizer
    base_opt = optim.Adam(model_lookahead.parameters(), lr=0.001)
    optimizer_lookahead = Lookahead(base_opt, k=5, alpha=0.5)
    
    print("Lookahead Optimizer:")
    print("  - Maintains fast and slow weights")
    print("  - Updates slow weights every k steps")
    print("  - Combines fast convergence with stability")
    print("  - k=5, alpha=0.5 (typical values)")
    print("  - Often improves convergence speed and final performance")
    print("  - Works well with adaptive optimizers like Adam")
    
    return optimizer_regular, optimizer_lookahead

# Example of when to use Lookahead
print("When to use Lookahead:")
print("1. When training is unstable or has high variance")
print("2. With adaptive optimizers like Adam for better generalization")
print("3. When you want to improve convergence without changing learning rate schedule")
print("4. In large-scale training where stability is important")
print("5. With cyclical learning rates for better exploration")
```

## 4. How do you implement gradient clipping and why is it important?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GradientClippingExample:
    def __init__(self, model, clip_value=1.0, clip_norm=1.0):
        self.model = model
        self.clip_value = clip_value  # For value clipping
        self.clip_norm = clip_norm    # For norm clipping
        
        self.optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step_with_value_clipping(self, inputs, targets):
        """Clip gradients by value"""
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        # Clip gradients by value
        torch.nn.utils.clip_grad_value_(self.model.parameters(), clip_value=self.clip_value)
        
        self.optimizer.step()
        
        return loss.item()
    
    def train_step_with_norm_clipping(self, inputs, targets):
        """Clip gradients by norm"""
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        # Clip gradients by norm
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=self.clip_norm)
        
        self.optimizer.step()
        
        return loss.item()
    
    def train_step_with_custom_clipping(self, inputs, targets):
        """Custom gradient clipping"""
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        # Custom clipping for specific parameters
        for name, param in self.model.named_parameters():
            if param.grad is not None:
                # Apply different clipping for different types of layers
                if 'weight' in name and len(param.shape) > 1:  # Only for weight matrices
                    # Clip by norm
                    param_norm = param.grad.data.norm(2)
                    if param_norm > self.clip_norm:
                        param.grad.data.mul_(self.clip_norm / (param_norm + 1e-6))
        
        self.optimizer.step()
        
        return loss.item()

# Gradient clipping utilities
def compute_gradient_norm(parameters):
    """Compute the gradient norm across all parameters"""
    total_norm = 0
    param_count = 0
    
    for p in parameters:
        if p.grad is not None:
            param_norm = p.grad.data.norm(2)
            total_norm += param_norm.item() ** 2
            param_count += 1
    
    total_norm = total_norm ** (1. / 2)
    return total_norm, param_count

def adaptive_gradient_clipping(model, clip_factor=0.01, eps=1e-3):
    """Adaptive gradient clipping based on parameter norms"""
    for name, param in model.named_parameters():
        if param.grad is not None:
            # Compute parameter norm
            param_norm = param.data.norm(2).clamp(min=eps)
            # Compute gradient norm
            grad_norm = param.grad.data.norm(2)
            # Adaptive clip value
            max_norm = clip_factor * param_norm
            clip_coef = max_norm / (grad_norm + eps)
            # Apply clipping if gradient is too large
            if clip_coef < 1:
                param.grad.data.mul_(clip_coef)

# Example usage
class RNNWithGradientClipping(nn.Module):
    def __init__(self, vocab_size, embed_size, hidden_size, num_layers):
        super(RNNWithGradientClipping, self).__init__()
        self.embedding = nn.Embedding(vocab_size, embed_size)
        self.lstm = nn.LSTM(embed_size, hidden_size, num_layers, batch_first=True)
        self.fc = nn.Linear(hidden_size, vocab_size)
        self.dropout = nn.Dropout(0.5)
        
        self.hidden_size = hidden_size
        self.num_layers = num_layers
    
    def forward(self, x, hidden=None):
        embedded = self.dropout(self.embedding(x))
        
        if hidden is None:
            lstm_out, hidden = self.lstm(embedded)
        else:
            lstm_out, hidden = self.lstm(embedded, hidden)
        
        # Use last time step for prediction
        output = self.fc(lstm_out[:, -1, :])
        return output, hidden

def train_with_gradient_clipping():
    """Example showing different gradient clipping techniques"""
    
    print("Gradient Clipping Techniques:")
    print("1. Value Clipping: Clip gradients to [-value, value]")
    print("2. Norm Clipping: Clip gradients by L2 norm")
    print("3. Adaptive Clipping: Scale based on parameter norms")
    
    # Sample data
    inputs = torch.randint(0, 1000, (32, 50))  # (batch_size, seq_len)
    targets = torch.randint(0, 1000, (32,))    # (batch_size,)
    
    # Model with potential gradient issues
    model = RNNWithGradientClipping(vocab_size=1000, embed_size=128, hidden_size=256, num_layers=2)
    
    # Different clipping approaches
    clipping_example = GradientClippingExample(model, clip_value=0.5, clip_norm=1.0)
    
    # Test different clipping methods
    loss1 = clipping_example.train_step_with_value_clipping(inputs, targets)
    loss2 = clipping_example.train_step_with_norm_clipping(inputs, targets)
    loss3 = clipping_example.train_step_with_custom_clipping(inputs, targets)
    
    print(f"\nSample losses with different clipping:")
    print(f"  Value clipping: {loss1:.4f}")
    print(f"  Norm clipping: {loss2:.4f}")
    print(f"  Custom clipping: {loss3:.4f}")
    
    # Compute gradient norms
    grad_norm, param_count = compute_gradient_norm(model.parameters())
    print(f"\nGradient norm: {grad_norm:.4f} (across {param_count} parameters)")

# When to use gradient clipping
print("When to use gradient clipping:")
print("1. Training RNNs/LSTMs to prevent exploding gradients")
print("2. In deep networks with many layers")
print("3. When using high learning rates")
print("4. In reinforcement learning with high variance gradients")
print("5. When training with large batch sizes")
print("6. In GAN training for more stable training")
```

## 5. How do you implement gradient accumulation for large batch training?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

class GradientAccumulation:
    def __init__(self, model, optimizer, accumulation_steps=4):
        self.model = model
        self.optimizer = optimizer
        self.accumulation_steps = accumulation_steps
        self.step_counter = 0
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step(self, inputs, targets):
        # Normalize loss by accumulation steps
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets) / self.accumulation_steps
        
        # Backward pass
        loss.backward()
        
        self.step_counter += 1
        
        # Update weights after accumulating gradients
        if self.step_counter % self.accumulation_steps == 0:
            # Optional: gradient clipping
            torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
            
            self.optimizer.step()
            self.optimizer.zero_grad()
        
        return loss.item() * self.accumulation_steps  # Return actual loss
    
    def train_epoch(self, dataloader, device='cpu'):
        self.model.train()
        total_loss = 0
        num_batches = 0
        
        # Ensure optimizer is zeroed at the start
        self.optimizer.zero_grad()
        
        for batch_idx, (data, target) in enumerate(dataloader):
            data, target = data.to(device), target.to(device)
            
            loss = self.train_step(data, target)
            total_loss += loss
            num_batches += 1
            
            if batch_idx % 10 == 0:
                print(f'Batch {batch_idx}, Accumulated Loss: {loss:.6f}')
        
        # Handle remaining gradients if total batches not divisible by accumulation steps
        if self.step_counter % self.accumulation_steps != 0:
            self.optimizer.step()
            self.optimizer.zero_grad()
        
        return total_loss / num_batches

# Alternative implementation with context manager
class GradientAccumulator:
    def __init__(self, optimizer, accumulation_steps):
        self.optimizer = optimizer
        self.accumulation_steps = accumulation_steps
        self.step_counter = 0
    
    def __enter__(self):
        self.optimizer.zero_grad()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.step_counter % self.accumulation_steps != 0:
            self.optimizer.step()
            self.optimizer.zero_grad()
    
    def step(self, loss):
        # Normalize loss
        normalized_loss = loss / self.accumulation_steps
        normalized_loss.backward()
        
        self.step_counter += 1
        
        if self.step_counter % self.accumulation_steps == 0:
            # Optional gradient clipping
            torch.nn.utils.clip_grad_norm_(self.optimizer.param_groups[0]['params'], 1.0)
            self.optimizer.step()
            self.optimizer.zero_grad()

# Example with custom gradient accumulation optimizer
class AccumulationOptimizer:
    def __init__(self, base_optimizer, accumulation_steps=1):
        self.base_optimizer = base_optimizer
        self.accumulation_steps = accumulation_steps
        self.accumulation_counter = 0
    
    def zero_grad(self):
        if self.accumulation_counter == 0:
            self.base_optimizer.zero_grad()
    
    def step(self):
        self.accumulation_counter += 1
        if self.accumulation_counter == self.accumulation_steps:
            # Update parameters
            self.base_optimizer.step()
            self.accumulation_counter = 0
            # Zero gradients for next accumulation cycle
            self.base_optimizer.zero_grad()
    
    def __getattr__(self, name):
        # Delegate attribute access to base optimizer
        return getattr(self.base_optimizer, name)

# Practical example
def create_large_batch_simulation():
    """Simulate training with gradient accumulation to mimic large batch size"""
    
    # Create sample data - smaller batches to simulate memory constraint
    X = torch.randn(1280, 784)  # 1280 samples
    y = torch.randint(0, 10, (1280,))  # 10 classes
    
    # Create small batches (as if limited by memory)
    dataset = TensorDataset(X, y)
    dataloader = DataLoader(dataset, batch_size=16, shuffle=True)  # Small batch size
    
    # Model
    model = nn.Sequential(
        nn.Linear(784, 512),
        nn.ReLU(),
        nn.Dropout(0.2),
        nn.Linear(512, 256),
        nn.ReLU(),
        nn.Linear(256, 10)
    )
    
    # Optimizer
    optimizer = optim.Adam(model.parameters(), lr=0.001)
    
    # Gradient accumulation (accumulates 4 small batches to simulate batch size of 64)
    grad_accum = GradientAccumulation(model, optimizer, accumulation_steps=4)
    
    print("Gradient Accumulation Example:")
    print(f"  Actual batch size: 16")
    print(f"  Accumulation steps: 4")
    print(f"  Effective batch size: {16 * 4}")
    
    # Train for one epoch
    avg_loss = grad_accum.train_epoch(dataloader)
    print(f"  Average loss: {avg_loss:.4f}")
    
    return model

# Advanced gradient accumulation with memory monitoring
class MemoryAwareAccumulator:
    def __init__(self, model, optimizer, accumulation_steps=4, max_memory_usage=0.8):
        self.model = model
        self.optimizer = optimizer
        self.accumulation_steps = accumulation_steps
        self.max_memory_usage = max_memory_usage
        self.step_counter = 0
        self.criterion = nn.CrossEntropyLoss()
    
    def get_gpu_memory_usage(self):
        """Get GPU memory usage if available"""
        if torch.cuda.is_available():
            return torch.cuda.memory_allocated() / torch.cuda.get_device_properties(0).total_memory
        return 0.0
    
    def train_step_adaptive(self, inputs, targets):
        """Adaptive gradient accumulation based on memory usage"""
        current_memory = self.get_gpu_memory_usage()
        
        # If memory usage is high, reduce effective batch size by increasing accumulation steps
        if current_memory > self.max_memory_usage:
            # In a real implementation, you'd reduce the effective batch size
            # Here we just log the memory pressure
            print(f"High memory usage: {current_memory:.2%}, consider reducing batch size")
        
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets) / self.accumulation_steps
        
        loss.backward()
        self.step_counter += 1
        
        if self.step_counter % self.accumulation_steps == 0:
            torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
            self.optimizer.step()
            self.optimizer.zero_grad()
        
        return loss.item() * self.accumulation_steps

# When to use gradient accumulation
print("When to use gradient accumulation:")
print("1. When desired batch size doesn't fit in memory")
print("2. For better gradient estimates with larger effective batch sizes")
print("3. In distributed training to simulate larger batch sizes")
print("4. When training with limited GPU memory")
print("5. To improve model convergence with stable gradients")
print("6. In fine-tuning large models with limited resources")
```

## 6. How do you implement mixed precision training for optimization?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.cuda.amp import autocast, GradScaler

class MixedPrecisionTrainer:
    def __init__(self, model, optimizer, loss_fn=None):
        self.model = model
        self.optimizer = optimizer
        self.loss_fn = loss_fn or nn.CrossEntropyLoss()
        
        # Initialize gradient scaler for mixed precision
        self.scaler = GradScaler()
        
        # Check if CUDA is available
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.device), targets.to(self.device)
        
        self.optimizer.zero_grad()
        
        # Use autocast for mixed precision
        with autocast():
            outputs = self.model(inputs)
            loss = self.loss_fn(outputs, targets)
        
        # Scale loss and backpropagate
        self.scaler.scale(loss).backward()
        
        # Unscaled gradients can be 'clipped' before the optimizer step
        self.scaler.unscale_(self.optimizer)
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        # Update parameters
        self.scaler.step(self.optimizer)
        
        # Update scaler for next iteration
        self.scaler.update()
        
        return loss.item()
    
    def evaluate(self, dataloader):
        self.model.eval()
        total_loss = 0
        correct = 0
        total = 0
        
        with torch.no_grad():
            for inputs, targets in dataloader:
                inputs, targets = inputs.to(self.device), targets.to(self.device)
                
                with autocast():
                    outputs = self.model(inputs)
                    loss = self.loss_fn(outputs, targets)
                
                total_loss += loss.item()
                _, predicted = torch.max(outputs.data, 1)
                total += targets.size(0)
                correct += (predicted == targets).sum().item()
        
        accuracy = 100 * correct / total
        avg_loss = total_loss / len(dataloader)
        
        return avg_loss, accuracy

# Custom mixed precision module
class MixedPrecisionModel(nn.Module):
    def __init__(self, base_model):
        super(MixedPrecisionModel, self).__init__()
        self.base_model = base_model
        
        # Quantize/convert specific layers if needed
        self.quant = torch.quantization.QuantStub()
        self.dequant = torch.quantization.DeQuantStub()
    
    def forward(self, x):
        x = self.quant(x)
        x = self.base_model(x)
        x = self.dequant(x)
        return x

# Training loop with mixed precision
def train_with_mixed_precision():
    # Define model, data, and optimizer
    model = nn.Sequential(
        nn.Linear(784, 512),
        nn.ReLU(),
        nn.Linear(512, 256),
        nn.ReLU(),
        nn.Linear(256, 10)
    )
    
    optimizer = optim.Adam(model.parameters(), lr=0.001)
    trainer = MixedPrecisionTrainer(model, optimizer)
    
    # Sample data
    inputs = torch.randn(32, 784)
    targets = torch.randint(0, 10, (32,))
    
    # Training step with mixed precision
    loss = trainer.train_step(inputs, targets)
    print(f"Mixed precision training loss: {loss:.4f}")
    
    return trainer

# Advanced mixed precision with custom configurations
class AdvancedMixedPrecisionTrainer:
    def __init__(self, model, optimizer, 
                 loss_scale_factor=2.0, 
                 init_scale=2.**16,
                 backoff_factor=0.5,
                 growth_interval=2000):
        self.model = model
        self.optimizer = optimizer
        
        # Advanced scaler configuration
        self.scaler = GradScaler(
            init_scale=init_scale,
            growth_factor=loss_scale_factor,
            backoff_factor=backoff_factor,
            growth_interval=growth_interval
        )
        
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        self.criterion = nn.CrossEntropyLoss()
    
    def train_batch(self, inputs, targets):
        inputs, targets = inputs.to(self.device), targets.to(self.device)
        
        self.optimizer.zero_grad()
        
        with autocast():
            outputs = self.model(inputs)
            loss = self.criterion(outputs, targets)
        
        # Backward pass with gradient scaling
        self.scaler.scale(loss).backward()
        
        # Gradient clipping in scaled space
        self.scaler.unscale_(self.optimizer)
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        # Optimizer step with scaling
        self.scaler.step(self.optimizer)
        self.scaler.update()
        
        return loss.item()
    
    def get_scale_info(self):
        """Get information about current scaling"""
        return {
            'current_scale': self.scaler.get_scale(),
            'growth_tracker': self.scaler._get_growth_tracker()
        }

# Mixed precision with different data types
class MixedPrecisionWithTypes:
    def __init__(self, model, optimizer, use_bfloat16=False):
        self.model = model
        self.optimizer = optimizer
        self.use_bfloat16 = use_bfloat16
        
        # Check if bfloat16 is supported
        if use_bfloat16 and torch.cuda.is_bf16_supported():
            print("Using bfloat16 mixed precision")
            self.scaler = GradScaler(enabled=True)
            self.cast_dtype = torch.bfloat16
        else:
            print("Using float16 mixed precision")
            self.scaler = GradScaler(enabled=True)
            self.cast_dtype = torch.float16
        
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model.to(self.device)
        self.criterion = nn.CrossEntropyLoss()
    
    def forward_with_casting(self, x):
        if self.use_bfloat16 and torch.cuda.is_bf16_supported():
            with autocast(dtype=torch.bfloat16):
                return self.model(x)
        else:
            with autocast():
                return self.model(x)

# Performance comparison functions
def compare_precision_performance():
    """Compare training performance between FP32 and mixed precision"""
    
    print("Mixed Precision Training Benefits:")
    print("1. 2x memory efficiency (FP16 vs FP32)")
    print("2. Faster training on modern GPUs")
    print("3. Larger effective batch sizes possible")
    print("4. Maintained model accuracy")
    print("5. Better performance on tensor cores")
    
    # Model comparison
    model_fp32 = nn.Linear(1000, 1000)
    model_fp16 = nn.Linear(1000, 1000)
    
    # Memory comparison
    fp32_memory = sum(p.numel() * p.element_size() for p in model_fp32.parameters())
    fp16_memory = sum(p.numel() * 2 for p in model_fp16.parameters())  # FP16 = 2 bytes
    
    print(f"\nMemory Usage Comparison:")
    print(f"  FP32 parameters: {fp32_memory / 1024**2:.2f} MB")
    print(f"  FP16 parameters: {fp16_memory / 1024**2:.2f} MB")
    print(f"  Memory savings: {(fp32_memory - fp16_memory) / fp32_memory * 100:.1f}%")

# When to use mixed precision
print("When to use mixed precision training:")
print("1. Training large models that don't fit in memory")
print("2. Using modern GPUs with tensor cores (V100, A100, RTX series)")
print("3. When you want faster training times")
print("4. For inference optimization")
print("5. When working with large batch sizes")
print("6. In research settings where memory efficiency is crucial")
```

## 7. How do you implement learning rate scheduling techniques?

**Answer:**
```python
import torch
import torch.optim as optim
import torch.nn as nn
import math
import matplotlib.pyplot as plt

class LearningRateScheduler:
    """Comprehensive learning rate scheduling techniques"""
    
    def __init__(self, optimizer):
        self.optimizer = optimizer
        self.base_lr = optimizer.param_groups[0]['lr']
    
    def step(self):
        """Override this in subclasses"""
        pass
    
    def get_lr(self):
        """Get current learning rate"""
        return self.optimizer.param_groups[0]['lr']

# 1. Cosine Annealing
class CosineAnnealingScheduler:
    def __init__(self, optimizer, T_max, eta_min=0, last_epoch=-1):
        self.optimizer = optimizer
        self.T_max = T_max
        self.eta_min = eta_min
        self.last_epoch = last_epoch
        self.base_lr = optimizer.param_groups[0]['lr']
    
    def step(self):
        self.last_epoch += 1
        lr = self.eta_min + (self.base_lr - self.eta_min) * \
             (1 + math.cos(math.pi * self.last_epoch / self.T_max)) / 2
        
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# 2. Step LR
class StepLRScheduler:
    def __init__(self, optimizer, step_size, gamma=0.1):
        self.optimizer = optimizer
        self.step_size = step_size
        self.gamma = gamma
        self.last_epoch = 0
        self.base_lr = optimizer.param_groups[0]['lr']
    
    def step(self):
        self.last_epoch += 1
        if self.last_epoch % self.step_size == 0:
            for param_group in self.optimizer.param_groups:
                param_group['lr'] *= self.gamma
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# 3. Exponential LR
class ExponentialLRScheduler:
    def __init__(self, optimizer, gamma):
        self.optimizer = optimizer
        self.gamma = gamma
        self.last_epoch = 0
        self.base_lr = optimizer.param_groups[0]['lr']
    
    def step(self):
        self.last_epoch += 1
        for param_group in self.optimizer.param_groups:
            param_group['lr'] *= self.gamma
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# 4. MultiStep LR
class MultiStepLRScheduler:
    def __init__(self, optimizer, milestones, gamma=0.1):
        self.optimizer = optimizer
        self.milestones = milestones  # List of epoch numbers
        self.gamma = gamma
        self.last_epoch = 0
        self.base_lr = optimizer.param_groups[0]['lr']
    
    def step(self):
        self.last_epoch += 1
        if self.last_epoch in self.milestones:
            for param_group in self.optimizer.param_groups:
                param_group['lr'] *= self.gamma
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# 5. Cyclical Learning Rates
class CyclicalLRScheduler:
    def __init__(self, optimizer, base_lr, max_lr, step_size_up=2000, mode='triangular'):
        self.optimizer = optimizer
        self.base_lr = base_lr
        self.max_lr = max_lr
        self.step_size_up = step_size_up
        self.mode = mode
        self.step_counter = 0
    
    def step(self):
        self.step_counter += 1
        cycle = math.floor(1 + self.step_counter / (2 * self.step_size_up))
        x = abs(self.step_counter / self.step_size_up - 2 * cycle + 1)
        
        if self.mode == 'triangular':
            lr = self.base_lr + (self.max_lr - self.base_lr) * max(0, (1 - x))
        elif self.mode == 'triangular2':
            lr = self.base_lr + (self.max_lr - self.base_lr) * max(0, (1 - x)) / float(2 ** (cycle - 1))
        
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# 6. One Cycle Policy
class OneCycleLRScheduler:
    def __init__(self, optimizer, max_lr, total_steps, pct_start=0.3, anneal_strategy='cos'):
        self.optimizer = optimizer
        self.max_lr = max_lr
        self.total_steps = total_steps
        self.pct_start = pct_start
        self.anneal_strategy = anneal_strategy
        
        self.step_idx = 0
        self.base_lr = optimizer.param_groups[0]['lr']
        
        # Calculate step counts
        self.up_step = int(total_steps * pct_start)
        self.down_step = total_steps - self.up_step
    
    def step(self):
        if self.step_idx < self.up_step:
            # Linear increase
            pct = self.step_idx / self.up_step
            lr = self.base_lr + (self.max_lr - self.base_lr) * pct
        else:
            # Annealing phase
            pct = (self.step_idx - self.up_step) / self.down_step
            if self.anneal_strategy == 'cos':
                lr = self.base_lr + (self.max_lr - self.base_lr) * (1 + math.cos(math.pi * pct)) / 2
            else:  # linear
                lr = self.max_lr - (self.max_lr - self.base_lr) * pct
        
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
        
        self.step_idx += 1
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# 7. Cosine Annealing with Warm Restarts
class CosineAnnealingWarmRestarts:
    def __init__(self, optimizer, T_0, T_mult=1, eta_min=0):
        self.optimizer = optimizer
        self.T_0 = T_0
        self.T_mult = T_mult
        self.eta_min = eta_min
        self.base_lr = optimizer.param_groups[0]['lr']
        
        self.T_i = T_0
        self.last_epoch = 0
        self.T_cur = 0
    
    def step(self):
        self.T_cur += 1
        if self.T_cur >= self.T_i:
            self.T_cur = 0
            self.T_i *= self.T_mult
        
        lr = self.eta_min + (self.base_lr - self.eta_min) * \
             (1 + math.cos(math.pi * self.T_cur / self.T_i)) / 2
        
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# 8. ReduceLROnPlateau (monitoring metric)
class ReduceLROnPlateauScheduler:
    def __init__(self, optimizer, mode='min', factor=0.1, patience=10, min_lr=0):
        self.optimizer = optimizer
        self.mode = mode
        self.factor = factor
        self.patience = patience
        self.min_lr = min_lr if isinstance(min_lr, list) else [min_lr] * len(optimizer.param_groups)
        
        self.best = None
        self.num_bad_epochs = 0
        self.mode_worse = float('inf') if mode == 'min' else -float('inf')
    
    def step(self, current_metric):
        if self.best is None or self._is_better(current_metric, self.best):
            self.best = current_metric
            self.num_bad_epochs = 0
        else:
            self.num_bad_epochs += 1
        
        if self.num_bad_epochs >= self.patience:
            self._reduce_lr()
            self.num_bad_epochs = 0
    
    def _is_better(self, current, best):
        if self.mode == 'min':
            return current < best
        else:
            return current > best
    
    def _reduce_lr(self):
        for i, param_group in enumerate(self.optimizer.param_groups):
            old_lr = float(param_group['lr'])
            new_lr = max(old_lr * self.factor, self.min_lr[i] if i < len(self.min_lr) else self.min_lr[0])
            param_group['lr'] = new_lr
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# Example usage and comparison
def demonstrate_schedulers():
    # Create a simple model and optimizer
    model = nn.Linear(10, 1)
    optimizer = optim.SGD(model.parameters(), lr=0.1)
    
    # Initialize different schedulers
    schedulers = {
        'Cosine Annealing': CosineAnnealingScheduler(optimizer, T_max=100, eta_min=0.01),
        'Step LR': StepLRScheduler(optimizer, step_size=30, gamma=0.5),
        'Cyclical': CyclicalLRScheduler(optimizer, base_lr=0.01, max_lr=0.1, step_size_up=50),
        'One Cycle': OneCycleLRScheduler(optimizer, max_lr=0.1, total_steps=100, pct_start=0.3),
        'Cosine Warm Restarts': CosineAnnealingWarmRestarts(optimizer, T_0=30, T_mult=2)
    }
    
    # Track learning rates over time
    steps = 100
    lr_history = {name: [] for name in schedulers.keys()}
    
    for step in range(steps):
        for name, scheduler in schedulers.items():
            scheduler.step()
            lr_history[name].append(scheduler.get_lr())
    
    print("Learning Rate Scheduling Techniques:")
    print("1. Cosine Annealing: Smooth cosine decay")
    print("2. Step LR: Drop LR by factor at fixed intervals")
    print("3. Cyclical LR: Oscillate between min/max LR")
    print("4. One Cycle: Single cycle with annealing")
    print("5. Cosine Annealing with Warm Restarts: Periodic restarts")
    
    return lr_history

# Custom learning rate functions
class CustomLRScheduler:
    def __init__(self, optimizer, lr_lambda):
        self.optimizer = optimizer
        self.lr_lambda = lr_lambda
        self.last_epoch = 0
    
    def step(self):
        lr = self.lr_lambda(self.last_epoch)
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
        self.last_epoch += 1
    
    def get_lr(self):
        return self.optimizer.param_groups[0]['lr']

# Example custom schedulers
def linear_warmup_scheduler(optimizer, warmup_steps, total_steps):
    """Linear warmup followed by linear decay"""
    def lr_lambda(current_step):
        if current_step < warmup_steps:
            return float(current_step) / float(max(1, warmup_steps))
        return max(
            0.0, 
            float(total_steps - current_step) / float(max(1, total_steps - warmup_steps))
        )
    return CustomLRScheduler(optimizer, lr_lambda)

def polynomial_decay_scheduler(optimizer, total_steps, power=1.0, end_lr=0.0):
    """Polynomial decay scheduler"""
    def lr_lambda(current_step):
        return (1.0 - float(current_step) / float(total_steps)) ** power
    return CustomLRScheduler(optimizer, lr_lambda)

# Practical example with training loop
def training_with_scheduling():
    """Example showing LR scheduling in training"""
    
    model = nn.Sequential(
        nn.Linear(784, 256),
        nn.ReLU(),
        nn.Linear(256, 10)
    )
    
    optimizer = optim.Adam(model.parameters(), lr=0.001)
    
    # Use PyTorch's built-in scheduler
    scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=50)
    
    print("Training with learning rate scheduling:")
    print(f"Initial LR: {optimizer.param_groups[0]['lr']}")
    
    # Simulate training steps
    for epoch in range(10):
        # Training code here...
        scheduler.step()
        current_lr = optimizer.param_groups[0]['lr']
        print(f"Epoch {epoch + 1}: LR = {current_lr:.6f}")

# When to use different scheduling techniques
print("When to use different LR scheduling techniques:")
print("\nCosine Annealing:")
print("- For smooth LR decay")
print("- When you want to avoid sharp LR drops")
print("- Works well with Adam optimizer")

print("\nStep LR:")
print("- Simple and effective")
print("- Good for SGD with momentum")
print("- When you know good drop points")

print("\nCyclical LR:")
print("- When optimal LR is unknown")
print("- Can escape local minima")
print("- Works well with small datasets")

print("\nOne Cycle Policy:")
print("- Modern approach for fast training")
print("- Combines benefits of high LR exploration and low LR exploitation")
print("- Popular for training on ImageNet and similar tasks")

print("\nReduceLROnPlateau:")
print("- When you want adaptive LR reduction")
print("- Based on actual training progress")
print("- Good when performance plateaus")
```

## 8. How do you implement advanced gradient-based optimizers?

**Answer:**
```python
import torch
import torch.optim as optim
import torch.nn as nn
import math

# 1. RAdam (Rectified Adam)
class RAdam(optim.Optimizer):
    def __init__(self, params, lr=1e-3, betas=(0.9, 0.999), eps=1e-8, weight_decay=0):
        defaults = dict(lr=lr, betas=betas, eps=eps, weight_decay=weight_decay)
        super(RAdam, self).__init__(params, defaults)

    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()

        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue

                grad = p.grad.data
                if grad.is_sparse:
                    raise RuntimeError('RAdam does not support sparse gradients')

                state = self.state[p]

                # State initialization
                if len(state) == 0:
                    state['step'] = 0
                    state['exp_avg'] = torch.zeros_like(p.data)
                    state['exp_avg_sq'] = torch.zeros_like(p.data)

                exp_avg, exp_avg_sq = state['exp_avg'], state['exp_avg_sq']
                beta1, beta2 = group['betas']

                state['step'] += 1

                # Decay the first and second moment running average coefficient
                exp_avg.mul_(beta1).add_(grad, alpha=1 - beta1)
                exp_avg_sq.mul_(beta2).addcmul_(grad, grad, value=1 - beta2)

                # Compute bias correction
                bias_correction1 = 1 - beta1 ** state['step']
                bias_correction2 = 1 - beta2 ** state['step']

                # Compute length of approximated SMA
                N_sma_max = 2 / (1 - beta2) - 1
                N_sma = N_sma_max - 2 * state['step'] * bias_correction2 / N_sma_max

                # Update parameters
                if N_sma >= 5:
                    # Variance rectification
                    rect_step_size = math.sqrt(
                        (N_sma - 4) / (N_sma_max - 4) *
                        (N_sma - 2) / N_sma *
                        N_sma_max / (N_sma_max - 2)
                    ) / bias_correction1

                    denom = exp_avg_sq.sqrt().add_(group['eps'])
                    p.data.addcdiv_(exp_avg, denom, value=-group['lr'] * rect_step_size)
                else:
                    # Un-rectified update
                    p.data.add_(exp_avg, alpha=-group['lr'] / bias_correction1)

                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['lr'] * group['weight_decay'])

        return loss

# 2. NovoGrad
class NovoGrad(optim.Optimizer):
    def __init__(self, params, lr=1e-3, betas=(0.95, 0.98), eps=1e-8, weight_decay=0):
        defaults = dict(lr=lr, betas=betas, eps=eps, weight_decay=weight_decay)
        super(NovoGrad, self).__init__(params, defaults)

    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()

        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue

                grad = p.grad.data
                if grad.is_sparse:
                    raise RuntimeError('NovoGrad does not support sparse gradients')

                state = self.state[p]

                # State initialization
                if len(state) == 0:
                    state['step'] = 0
                    state['v'] = torch.zeros_like(p.data)  # Second moment
                    state['m'] = torch.zeros_like(p.data)  # First moment

                v, m = state['v'], state['m']
                beta1, beta2 = group['betas']

                state['step'] += 1

                # Normalize gradient
                grad_norm = torch.norm(grad)
                
                # Update second moment
                v.mul_(beta2).addcmul_(grad, grad, value=1 - beta2)
                
                # Compute momentum
                if grad_norm != 0:
                    grad = grad / (grad_norm + group['eps'])
                
                # Update first moment
                m.mul_(beta1).add_(grad, alpha=1 - beta1)

                # Update parameters
                update = m / (torch.sqrt(v) + group['eps'])
                p.data.add_(update, alpha=-group['lr'])

                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['lr'] * group['weight_decay'])

        return loss

# 3. QHAdam (Quasi-Hyperbolic Adam)
class QHAdam(optim.Optimizer):
    def __init__(self, params, lr=1e-3, betas=(0.9, 0.999), nus=(0.7, 1.0), weight_decay=0):
        defaults = dict(lr=lr, betas=betas, nus=nus, weight_decay=weight_decay)
        super(QHAdam, self).__init__(params, defaults)

    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()

        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue

                grad = p.grad.data
                state = self.state[p]

                if len(state) == 0:
                    state['step'] = 0
                    state['exp_avg'] = torch.zeros_like(p.data)
                    state['exp_avg_sq'] = torch.zeros_like(p.data)

                exp_avg, exp_avg_sq = state['exp_avg'], state['exp_avg_sq']
                beta1, beta2 = group['betas']
                nu1, nu2 = group['nus']

                state['step'] += 1

                # Update biased first moment estimate
                exp_avg.mul_(beta1).add_(grad, alpha=1 - beta1)

                # Update biased second raw moment estimate
                exp_avg_sq.mul_(beta2).addcmul_(grad, grad, value=1 - beta2)

                # Compute QH update
                bias_correction1 = 1 - beta1 ** state['step']
                bias_correction2 = 1 - beta2 ** state['step']

                # QHAdam update rule
                numerator = (1 - nu1) * grad / bias_correction1 + nu1 * exp_avg / bias_correction1
                denominator = (1 - nu2) * (grad ** 2) / bias_correction2 + nu2 * exp_avg_sq / bias_correction2

                update = numerator / (torch.sqrt(denominator) + group['eps'])
                p.data.add_(update, alpha=-group['lr'])

                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['weight_decay'])

        return loss

# 4. AccSGD (Accelerated SGD)
class AccSGD(optim.Optimizer):
    def __init__(self, params, lr=1e-3, kappa=1000.0, xi=10.0, small_const=0.7, weight_decay=0):
        defaults = dict(lr=lr, kappa=kappa, xi=xi, small_const=small_const, weight_decay=weight_decay)
        super(AccSGD, self).__init__(params, defaults)

    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()

        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue

                grad = p.grad.data
                state = self.state[p]

                if len(state) == 0:
                    state['step'] = 0
                    state['momentum_buffer'] = torch.zeros_like(p.data)
                    state['gradient_buffer'] = torch.zeros_like(p.data)

                momentum_buffer = state['momentum_buffer']
                gradient_buffer = state['gradient_buffer']
                
                # Update step
                alpha = 1.0 - (math.sqrt(group['kappa']) * (group['small_const'] ** (state['step'] + 1))) / (
                    math.sqrt(group['kappa']) + (group['small_const'] ** (state['step'] + 1))
                )
                beta = 1.0 - (math.sqrt(group['kappa']) + (group['small_const'] ** (state['step'] + 1))) / (
                    (group['xi'] * math.sqrt(group['kappa'])) + (group['small_const'] ** (state['step'] + 1))
                )
                
                # Update momentum
                momentum_buffer.mul_(alpha).add_(grad, alpha=1 - alpha)
                
                # Update parameter
                p.data.add_(momentum_buffer, alpha=-group['lr'])
                p.data.add_(gradient_buffer, alpha=-beta * group['lr'])

                # Update gradient buffer
                gradient_buffer.mul_(alpha).add_(grad, alpha=1 - alpha)

                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['lr'] * group['weight_decay'])

                state['step'] += 1

        return loss

# 5. Yogi optimizer
class Yogi(optim.Optimizer):
    def __init__(self, params, lr=1e-2, betas=(0.9, 0.999), eps=1e-3, weight_decay=0):
        defaults = dict(lr=lr, betas=betas, eps=eps, weight_decay=weight_decay)
        super(Yogi, self).__init__(params, defaults)

    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()

        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue

                grad = p.grad.data
                state = self.state[p]

                if len(state) == 0:
                    state['step'] = 0
                    state['exp_avg'] = torch.zeros_like(p.data)
                    state['exp_avg_sq'] = torch.zeros_like(p.data)

                exp_avg, exp_avg_sq = state['exp_avg'], state['exp_avg_sq']
                beta1, beta2 = group['betas']

                state['step'] += 1

                # Update first moment
                exp_avg.mul_(beta1).add_(grad, alpha=1 - beta1)

                # Update second moment (Yogi modification)
                grad_squared = grad.mul(grad)
                exp_avg_sq.addcmul_(torch.sign(exp_avg_sq - grad_squared), grad_squared, value=-(1 - beta2))

                # Bias correction
                bias_correction1 = 1 - beta1 ** state['step']
                bias_correction2 = 1 - beta2 ** state['step']

                # Update parameters
                denom = exp_avg_sq.sqrt().add_(group['eps'])
                step_size = group['lr'] * math.sqrt(bias_correction2) / bias_correction1
                p.data.addcdiv_(exp_avg, denom, value=-step_size)

                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['lr'] * group['weight_decay'])

        return loss

# Example usage and comparison
def compare_advanced_optimizers():
    """Compare different advanced optimizers"""
    
    # Create a simple model
    model = nn.Sequential(
        nn.Linear(100, 50),
        nn.ReLU(),
        nn.Linear(50, 10)
    )
    
    # Different optimizers
    optimizers = {
        'RAdam': RAdam(model.parameters(), lr=0.001),
        'NovoGrad': NovoGrad(model.parameters(), lr=0.01),
        'QHAdam': QHAdam(model.parameters(), lr=0.001),
        'AccSGD': AccSGD(model.parameters(), lr=0.01),
        'Yogi': Yogi(model.parameters(), lr=0.01)
    }
    
    print("Advanced Gradient-Based Optimizers:")
    print("\nRAdam (Rectified Adam):")
    print("- Rectifies the variance of the adaptive learning rate")
    print("- More stable than Adam, especially with high initial learning rates")
    
    print("\nNovoGrad:")
    print("- Normalizes gradient by its norm before applying updates")
    print("- Good for large-batch training and models with high gradient variance")
    
    print("\nQHAdam (Quasi-Hyperbolic Adam):")
    print("- Combines momentum and adaptive learning rates with quasi-hyperbolic averaging")
    print("- Can provide better generalization than Adam")
    
    print("\nAccSGD (Accelerated SGD):")
    print("- Accelerated SGD with momentum and gradient tracking")
    print("- Good for non-convex optimization")
    
    print("\nYogi:")
    print("- Modification of Adam that uses sign information for second moment updates")
    print("- Better at handling sparse gradients than Adam")
    
    return optimizers

# When to use each optimizer
print("When to use advanced optimizers:")
print("\nRAdam:")
print("- When you want the benefits of Adam but with more stable early training")
print("- For transformer models and large-scale training")
print("- As a drop-in replacement for Adam when you experience instability")

print("\nNovoGrad:")
print("- When training with large batch sizes")
print("- For models with high gradient variance (e.g., RNNs, LSTMs)")
print("- When you need better normalization of gradient updates")

print("\nQHAdam:")
print("- When you want to combine benefits of different update rules")
print("- For improved generalization in certain tasks")
print("- As an alternative to Adam when seeking better convergence")

print("\nAccSGD:")
print("- For non-convex optimization where acceleration helps")
print("- When you need the benefits of momentum but with SGD-style updates")
print("- For training with high learning rates")

print("\nYogi:")
print("- For sparse gradient problems")
print("- When Adam shows poor performance due to second moment issues")
print("- As a more robust alternative to Adam")
```

## 9. How do you implement second-order optimization methods?

**Answer:**
```python
import torch
import torch.optim as optim
import torch.nn as nn
import numpy as np
from scipy.optimize import minimize
import math

# 1. LBFGS (L-BFGS) optimizer
class LBGFSExample:
    def __init__(self, model):
        self.model = model
        # LBFGS is good for small to medium datasets
        self.optimizer = optim.LBFGS(model.parameters(), lr=1.0, max_iter=20)
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step(self, inputs, targets):
        def closure():
            self.optimizer.zero_grad()
            outputs = self.model(inputs)
            loss = self.criterion(outputs, targets)
            loss.backward()
            return loss
        
        # LBFGS requires closure function
        loss = self.optimizer.step(closure)
        return loss.item()

# 2. K-FAC (Kronecker-Factored Approximate Curvature)
class KFACOptimizer(optim.Optimizer):
    """
    Simplified K-FAC implementation for demonstration
    Original K-FAC is more complex and requires special layer handling
    """
    def __init__(self, model, lr=0.001, damping=0.001):
        defaults = dict(lr=lr, damping=damping)
        super(KFACOptimizer, self).__init__(model.parameters(), defaults)
        
        self.model = model
        self.damping = damping
        
        # Store layer information for K-FAC
        self.layer_info = {}
        self.setup_kfac()
    
    def setup_kfac(self):
        """Setup K-FAC for linear and convolutional layers"""
        for name, module in self.model.named_modules():
            if isinstance(module, (nn.Linear, nn.Conv2d)):
                # Initialize Fisher information matrix approximations
                self.layer_info[name] = {
                    'A': None,  # Activation covariance
                    'G': None,  # Gradient covariance
                    'step': 0
                }
    
    def step(self, closure=None):
        """K-FAC step implementation (simplified)"""
        loss = None
        if closure is not None:
            loss = closure()
        
        # This is a simplified version - real K-FAC is much more complex
        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue
                
                # K-FAC would compute Kronecker product approximations here
                # For now, just do a basic Newton-like update
                grad = p.grad.data
                
                # In real K-FAC, we'd compute Fisher information matrix
                # and apply its inverse (approximately)
                effective_lr = group['lr'] / (torch.norm(grad) + group['damping'])
                p.data.add_(grad, alpha=-effective_lr)
        
        return loss

# 3. Shampoo optimizer (Second-order method with preconditioning)
class Shampoo(optim.Optimizer):
    """
    Shampoo: Second Order Method Using Matrix Sqrt Preconditioning
    """
    def __init__(self, params, lr=0.001, momentum=0.9, weight_decay=0, 
                 epsilon=1e-8, update_freq=1):
        defaults = dict(lr=lr, momentum=momentum, weight_decay=weight_decay,
                       epsilon=epsilon, update_freq=update_freq)
        super(Shampoo, self).__init__(params, defaults)
    
    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()
        
        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue
                
                grad = p.grad.data
                state = self.state[p]
                
                # State initialization
                if len(state) == 0:
                    state['step'] = 0
                    if grad.dim() > 1:  # Only for matrices/vectors
                        # Create preconditioning matrices for each dimension
                        shapes = list(grad.shape)
                        state['preconditioners'] = []
                        for dim in range(grad.dim()):
                            size = shapes[dim]
                            state['preconditioners'].append(
                                torch.eye(size, device=grad.device) * group['epsilon']
                            )
                
                state['step'] += 1
                
                if grad.dim() > 1 and len(state['preconditioners']) > 0:
                    # Simplified Shampoo update
                    # In practice, this would involve computing Kronecker products
                    # and taking matrix square roots
                    
                    # Update preconditioning matrices (simplified)
                    if state['step'] % group['update_freq'] == 0:
                        for i, P in enumerate(state['preconditioners']):
                            # Compute gradient moment for this dimension
                            grad_slice = grad
                            for j in range(grad.dim()):
                                if j != i:
                                    grad_slice = grad_slice.sum(dim=j, keepdim=True)
                            grad_slice = grad_slice.squeeze()
                            
                            # Update preconditioner (simplified)
                            grad_outer = torch.outer(grad_slice.flatten(), grad_slice.flatten())
                            state['preconditioners'][i].add_(grad_outer, alpha=0.01)
                    
                    # Apply preconditioning (simplified)
                    update = grad.clone()
                    for i, P in enumerate(state['preconditioners']):
                        # This is a placeholder - real Shampoo involves matrix sqrt
                        # and more complex tensor operations
                        U, S, V = torch.svd(P)
                        sqrt_P = U @ torch.diag(torch.sqrt(S + group['epsilon'])) @ V.t()
                        # In practice: apply sqrt_P inverse to gradient
                        update = update  # Placeholder
                    
                    # Apply update
                    p.data.add_(update, alpha=-group['lr'])
                else:
                    # For 1D parameters, use regular SGD update
                    p.data.add_(grad, alpha=-group['lr'])
                
                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['lr'] * group['weight_decay'])
        
        return loss

# 4. Natural Gradient Descent (simplified)
class NaturalGradient(optim.Optimizer):
    """
    Simplified Natural Gradient implementation
    """
    def __init__(self, params, lr=0.01, weight_decay=0):
        defaults = dict(lr=lr, weight_decay=weight_decay)
        super(NaturalGradient, self).__init__(params, defaults)
        
        # Fisher Information Matrix approximations
        self.fisher_approx = {}
    
    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()
        
        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue
                
                grad = p.grad.data
                state = self.state[p]
                
                if len(state) == 0:
                    state['step'] = 0
                    # Initialize Fisher approximation
                    state['fisher'] = torch.eye(grad.numel(), device=grad.device) * 0.01
                
                state['step'] += 1
                
                # In real natural gradient, we'd use actual Fisher matrix
                # This is a simplified version using gradient covariance
                grad_flat = grad.view(-1)
                fisher_update = torch.ger(grad_flat, grad_flat)
                
                # Update Fisher approximation
                state['fisher'].mul_(0.99).add_(fisher_update, alpha=0.01)
                
                # Apply natural gradient: F^{-1} * gradient
                try:
                    fisher_inv = torch.inverse(state['fisher'])
                    nat_grad = (fisher_inv @ grad_flat).view_as(grad)
                except:
                    # Fallback to regular gradient if inverse fails
                    nat_grad = grad
                
                # Apply update
                p.data.add_(nat_grad, alpha=-group['lr'])
                
                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['lr'] * group['weight_decay'])
        
        return loss

# 5. Hessian-Free Optimization (conceptual)
class HessianFreeOpt(optim.Optimizer):
    """
    Conceptual Hessian-Free optimizer
    Real implementation would use conjugate gradient to solve Hessian system
    """
    def __init__(self, params, lr=0.1, cg_max_iter=10, weight_decay=0):
        defaults = dict(lr=lr, cg_max_iter=cg_max_iter, weight_decay=weight_decay)
        super(HessianFreeOpt, self).__init__(params, defaults)
        
        # Hessian-Free requires computing Hessian-vector products
        # This would typically be done using Pearlmutter's trick
        self.cg_max_iter = cg_max_iter
    
    def step(self, closure=None):
        loss = None
        if closure is not None:
            loss = closure()
        
        for group in self.param_groups:
            for p in group['params']:
                if p.grad is None:
                    continue
                
                grad = p.grad.data
                
                # In real Hessian-Free, we'd solve H * s = -g
                # using conjugate gradient method to get search direction s
                # Here we just make a conceptual update
                
                # Placeholder Hessian-vector product computation
                # In practice, this would use Pearlmutter's "trick":
                # H*v = d²L/dθ² * v ≈ (g(θ + εv) - g(θ - εv)) / (2ε)
                
                # For demonstration, assume we have computed search direction
                search_direction = -grad  # Placeholder
                
                # Apply update
                p.data.add_(search_direction, alpha=group['lr'])
                
                # Weight decay
                if group['weight_decay'] != 0:
                    p.data.add_(p.data, alpha=-group['lr'] * group['weight_decay'])
        
        return loss

# Example usage
def demonstrate_second_order_methods():
    """Demonstrate second-order optimization methods"""
    
    model = nn.Sequential(
        nn.Linear(784, 256),
        nn.ReLU(),
        nn.Linear(256, 10)
    )
    
    # Sample data
    inputs = torch.randn(32, 784)
    targets = torch.randint(0, 10, (32,))
    
    print("Second-Order Optimization Methods:")
    print("\n1. LBFGS (Limited-memory BFGS):")
    print("   - Quasi-Newton method")
    print("   - Good for small-medium datasets")
    print("   - Uses Hessian approximation")
    
    # LBFGS example
    lbfgs_model = nn.Sequential(
        nn.Linear(784, 64),
        nn.ReLU(),
        nn.Linear(64, 10)
    )
    lbfgs_opt = optim.LBFGS(lbfgs_model.parameters())
    
    def lbfgs_closure():
        lbfgs_opt.zero_grad()
        output = lbfgs_model(inputs)
        loss = nn.CrossEntropyLoss()(output, targets)
        loss.backward()
        return loss
    
    # LBFGS needs closure
    lbfgs_loss = lbfgs_opt.step(lbfgs_closure)
    print(f"   LBFGS loss: {lbfgs_loss:.4f}")
    
    print("\n2. K-FAC (Kronecker-Factored Approximate Curvature):")
    print("   - Second-order method for neural networks")
    print("   - Approximates Fisher matrix with Kronecker products")
    print("   - Efficient for fully connected and convolutional layers")
    
    kfac_opt = KFACOptimizer(model)
    try:
        kfac_loss = kfac_opt.step()
        print(f"   K-FAC conceptual step completed")
    except:
        print("   K-FAC requires more complex implementation")
    
    print("\n3. Shampoo:")
    print("   - Second-order method with matrix square root preconditioning")
    print("   - Good for matrices and high-dimensional parameters")
    print("   - Uses eigendecomposition for preconditioning")
    
    shampoo_opt = Shampoo(model.parameters())
    shampoo_loss = shampoo_opt.step()
    print(f"   Shampoo conceptual step completed")
    
    print("\n4. Natural Gradient:")
    print("   - Uses Fisher Information Matrix")
    print("   - Invariant to parameterization")
    print("   - Good for probabilistic models")
    
    nat_grad_opt = NaturalGradient(model.parameters())
    nat_grad_loss = nat_grad_opt.step()
    print(f"   Natural Gradient conceptual step completed")
    
    print("\n5. Hessian-Free:")
    print("   - Solves Newton system without forming Hessian")
    print("   - Uses conjugate gradient to solve H*s = -g")
    print("   - Good for large neural networks")
    
    hess_free_opt = HessianFreeOpt(model.parameters())
    hess_free_loss = hess_free_opt.step()
    print(f"   Hessian-Free conceptual step completed")

# When to use second-order methods
print("When to use second-order optimization methods:")
print("\nLBFGS:")
print("- Small to medium datasets")
print("- When you need very accurate solutions")
print("- For fine-tuning or final optimization steps")
print("- When first-order methods are too slow")

print("\nK-FAC:")
print("- Training neural networks with proper implementation")
print("- When you want second-order benefits with first-order cost")
print("- For convolutional and fully connected layers")
print("- In distributed training scenarios")

print("\nShampoo:")
print("- Training with matrix/tensor parameters")
print("- When you want better conditioning than Adam")
print("- For transformers and large models")
print("- When memory allows for full matrix operations")

print("\nNatural Gradient:")
print("- Probabilistic models and variational inference")
print("- When parameterization invariance is important")
print("- In reinforcement learning with policy gradients")
print("- For models with probabilistic outputs")

print("\nHessian-Free:")
print("- Large neural networks where Hessian is too expensive")
print("- When you need Newton-like convergence")
print("- For difficult optimization landscapes")
print("- With conjugate gradient acceleration")
```

## 10. How do you implement distributed optimization techniques?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim
import torch.distributed as dist
import torch.multiprocessing as mp
from torch.nn.parallel import DistributedDataParallel as DDP
import os

# 1. Basic Distributed Data Parallel Setup
class DistributedOptimizer:
    def __init__(self, model, optimizer, rank, world_size):
        self.rank = rank
        self.world_size = world_size
        
        # Setup distributed training
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        # Create DDP model
        self.model = DDP(
            model, 
            device_ids=[rank],
            find_unused_parameters=True
        )
        
        self.optimizer = optimizer
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        self.optimizer.zero_grad()
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        # DDP handles gradient synchronization automatically
        self.optimizer.step()
        
        return loss.item()

# 2. Gradient Compression for Communication Efficiency
class GradientCompressionOptimizer:
    def __init__(self, model, optimizer, rank, world_size, compression_ratio=0.1):
        self.rank = rank
        self.world_size = world_size
        self.compression_ratio = compression_ratio
        
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        self.model = DDP(model, device_ids=[rank])
        self.optimizer = optimizer
        self.criterion = nn.CrossEntropyLoss()
    
    def compress_gradients(self):
        """Top-k gradient compression"""
        for param_group in self.optimizer.param_groups:
            for param in param_group['params']:
                if param.grad is not None:
                    grad_flat = param.grad.data.view(-1)
                    
                    # Get top-k indices (k = compression_ratio * total_params)
                    k = max(1, int(len(grad_flat) * self.compression_ratio))
                    _, indices = torch.topk(torch.abs(grad_flat), k)
                    
                    # Zero out non-top-k gradients
                    mask = torch.zeros_like(grad_flat)
                    mask[indices] = 1.0
                    param.grad.data.mul_(mask.view_as(param.grad))
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        self.optimizer.zero_grad()
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        # Apply gradient compression before optimizer step
        self.compress_gradients()
        
        self.optimizer.step()
        
        return loss.item()

# 3. Gradient Quantization
class QuantizedDistributedOptimizer:
    def __init__(self, model, optimizer, rank, world_size, bits=8):
        self.rank = rank
        self.world_size = world_size
        self.bits = bits
        self.quantization_levels = 2 ** bits
        
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        self.model = DDP(model, device_ids=[rank])
        self.optimizer = optimizer
        self.criterion = nn.CrossEntropyLoss()
    
    def quantize_gradients(self):
        """Quantize gradients to reduce communication"""
        for param_group in self.optimizer.param_groups:
            for param in param_group['params']:
                if param.grad is not None:
                    # Get gradient range
                    grad_min, grad_max = param.grad.data.min(), param.grad.data.max()
                    
                    # Quantize to discrete levels
                    scale = (grad_max - grad_min) / (self.quantization_levels - 1)
                    quantized = torch.round((param.grad.data - grad_min) / scale)
                    dequantized = quantized * scale + grad_min
                    
                    param.grad.data = dequantized
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        self.optimizer.zero_grad()
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        # Quantize gradients before synchronization
        self.quantize_gradients()
        
        self.optimizer.step()
        
        return loss.item()

# 4. Local SGD (Federated-style)
class LocalSGDOptimizer:
    def __init__(self, model, optimizer, rank, world_size, local_steps=10):
        self.rank = rank
        self.world_size = world_size
        self.local_steps = local_steps
        self.step_count = 0
        
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        self.model = DDP(model, device_ids=[rank], broadcast_buffers=False)
        self.optimizer = optimizer
        self.criterion = nn.CrossEntropyLoss()
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        self.optimizer.zero_grad()
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        self.optimizer.step()
        
        # After local_steps, synchronize with other workers
        self.step_count += 1
        if self.step_count % self.local_steps == 0:
            self.synchronize_weights()
        
        return loss.item()
    
    def synchronize_weights(self):
        """Synchronize model weights across all processes"""
        for param in self.model.parameters():
            dist.all_reduce(param.data, op=dist.ReduceOp.SUM)
            param.data /= self.world_size

# 5. Asynchronous SGD
class AsyncSGDOptimizer:
    def __init__(self, model, optimizer, rank, world_size, sync_frequency=5):
        self.rank = rank
        self.world_size = world_size
        self.sync_frequency = sync_frequency
        
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        self.model = model  # Not using DDP for async
        self.optimizer = optimizer
        self.criterion = nn.CrossEntropyLoss()
        
        # Shared parameters storage (in practice, this would use shared memory)
        self.shared_params = {name: param.data.clone() for name, param in model.named_parameters()}
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        self.optimizer.zero_grad()
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        
        self.optimizer.step()
        
        # Occasionally synchronize with shared parameters
        if torch.rand(1).item() < 1.0 / self.sync_frequency:
            self.async_update_shared_params()
        
        return loss.item()
    
    def async_update_shared_params(self):
        """Update shared parameters asynchronously"""
        # This is a conceptual implementation
        # Real async SGD would use parameter servers or shared memory
        pass

# 6. Distributed Lookahead
class DistributedLookahead:
    def __init__(self, model, base_optimizer, rank, world_size, k=5, alpha=0.5):
        self.rank = rank
        self.world_size = world_size
        self.k = k
        self.alpha = alpha
        self.step_counter = 0
        
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        self.model = DDP(model, device_ids=[rank])
        self.base_optimizer = base_optimizer
        self.criterion = nn.CrossEntropyLoss()
        
        # Initialize slow weights
        self.slow_weights = {}
        for name, param in self.model.named_parameters():
            self.slow_weights[name] = param.data.clone().detach()
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        self.base_optimizer.zero_grad()
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets)
        loss.backward()
        self.base_optimizer.step()
        
        self.step_counter += 1
        
        # Update slow weights every k steps
        if self.step_counter % self.k == 0:
            self.synchronize_slow_weights()
        
        return loss.item()
    
    def synchronize_slow_weights(self):
        """Synchronize slow weights across processes"""
        for name, param in self.model.named_parameters():
            # Update slow weight: slow = alpha * fast + (1 - alpha) * slow
            slow_weight = self.slow_weights[name]
            slow_weight.mul_(1 - self.alpha).add_(param.data, alpha=self.alpha)
            
            # Average slow weights across all processes
            dist.all_reduce(slow_weight, op=dist.ReduceOp.SUM)
            slow_weight /= self.world_size
            
            # Update fast weight to be: alpha * fast + (1 - alpha) * slow
            param.data.mul_(self.alpha).add_(slow_weight, alpha=1 - self.alpha)

# 7. Gradient Accumulation in Distributed Setting
class DistributedGradAccum:
    def __init__(self, model, optimizer, rank, world_size, accumulation_steps=4):
        self.rank = rank
        self.world_size = world_size
        self.accumulation_steps = accumulation_steps
        self.step_counter = 0
        
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        self.model = DDP(model, device_ids=[rank])
        self.optimizer = optimizer
        self.criterion = nn.CrossEntropyLoss()
        
        # Zero gradients initially
        self.optimizer.zero_grad()
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        # Normalize loss by accumulation steps
        outputs = self.model(inputs)
        loss = self.criterion(outputs, targets) / self.accumulation_steps
        loss.backward()
        
        self.step_counter += 1
        
        # Update weights after accumulating gradients
        if self.step_counter % self.accumulation_steps == 0:
            # All-reduce gradients in DDP
            self.optimizer.step()
            self.optimizer.zero_grad()
        
        return loss.item() * self.accumulation_steps

# 8. Distributed Mixed Precision
class DistributedMixedPrecision:
    def __init__(self, model, optimizer, rank, world_size):
        self.rank = rank
        self.world_size = world_size
        
        torch.cuda.set_device(rank)
        model = model.to(rank)
        
        self.model = DDP(model, device_ids=[rank])
        self.optimizer = optimizer
        self.criterion = nn.CrossEntropyLoss()
        
        # Initialize gradient scaler for mixed precision
        from torch.cuda.amp import GradScaler
        self.scaler = GradScaler()
    
    def train_step(self, inputs, targets):
        inputs, targets = inputs.to(self.rank), targets.to(self.rank)
        
        self.optimizer.zero_grad()
        
        with torch.cuda.amp.autocast():
            outputs = self.model(inputs)
            loss = self.criterion(outputs, targets)
        
        # Scale loss and backpropagate
        self.scaler.scale(loss).backward()
        
        # Unscaled gradients can be clipped before optimizer step
        self.scaler.unscale_(self.optimizer)
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        # Update parameters
        self.scaler.step(self.optimizer)
        self.scaler.update()
        
        return loss.item()

# Example setup function
def setup_distributed(rank, world_size):
    """Initialize the distributed environment"""
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '12355'
    
    # Initialize the process group
    dist.init_process_group("nccl", rank=rank, world_size=world_size)

def cleanup_distributed():
    """Clean up the distributed environment"""
    dist.destroy_process_group()

# Example usage
def demonstrate_distributed_optimization():
    """Demonstrate distributed optimization techniques"""
    
    print("Distributed Optimization Techniques:")
    print("\n1. Distributed Data Parallel (DDP):")
    print("   - Most common distributed training method")
    print("   - Each process gets a batch of data")
    print("   - Gradients synchronized automatically")
    
    print("\n2. Gradient Compression:")
    print("   - Reduces communication overhead")
    print("   - Top-k or random sparsification")
    print("   - Trade-off between communication and accuracy")
    
    print("\n3. Gradient Quantization:")
    print("   - Reduces precision of gradients")
    print("   - Further reduces communication cost")
    print("   - Can maintain training accuracy")
    
    print("\n4. Local SGD:")
    print("   - Local updates between synchronizations")
    print("   - Reduces communication frequency")
    print("   - Good for federated learning")
    
    print("\n5. Asynchronous SGD:")
    print("   - No synchronization barriers")
    print("   - Faster but potentially less stable")
    print("   - Requires careful implementation")
    
    print("\n6. Distributed Lookahead:")
    print("   - Combines fast and slow weights")
    print("   - Maintains stability in distributed setting")
    print("   - Reduces communication of fast weights")
    
    print("\n7. Gradient Accumulation:")
    print("   - Simulates larger batch sizes")
    print("   - Useful when per-GPU memory is limited")
    print("   - Maintains effective batch size")
    
    print("\n8. Distributed Mixed Precision:")
    print("   - Combines DDP with mixed precision")
    print("   - Reduces memory and increases speed")
    print("   - Essential for large model training")

# When to use distributed optimization
print("When to use distributed optimization:")
print("\nDistributed Data Parallel:")
print("- When you have multiple GPUs on one machine or multiple machines")
print("- For scaling training of large models")
print("- When you want to reduce training time significantly")
print("- For models that fit in single GPU memory")

print("\nGradient Compression:")
print("- When network bandwidth is limited")
print("- For reducing communication overhead")
print("- In distributed environments with slow networks")
print("- When communication cost dominates computation cost")

print("\nLocal SGD:")
print("- When communication is expensive")
print("- For federated learning scenarios")
print("- When you can tolerate some staleness")
print("- For reducing synchronization frequency")

print("\nAsynchronous SGD:")
print("- When you need maximum throughput")
print("- For research on asynchronous optimization")
print("- When strict synchronization is not required")
print("- For challenging the consistency requirements")
```

## 11. How do you implement optimization for different neural network architectures?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim
import math

# 1. Optimization for RNNs/LSTMs
class RNNOptimization:
    def __init__(self, model, lr=0.001):
        self.model = model
        
        # RNNs benefit from specific optimization techniques
        # Use optimizers that handle gradient issues well
        self.optimizer = optim.Adam(model.parameters(), lr=lr)
        
        # RNN-specific optimization techniques
        self.gradient_clipping = True
        self.use_layer_norm = True
    
    def train_step(self, inputs, targets):
        self.optimizer.zero_grad()
        
        outputs, _ = self.model(inputs)
        loss = nn.CrossEntropyLoss()(outputs, targets)
        loss.backward()
        
        # Gradient clipping is crucial for RNNs
        if self.gradient_clipping:
            torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        self.optimizer.step()
        
        return loss.item()

# 2. Optimization for CNNs
class CNNOptimization:
    def __init__(self, model, lr=0.001):
        self.model = model
        
        # CNNs often benefit from momentum-based optimizers
        self.optimizer = optim.SGD(
            model.parameters(), 
            lr=lr, 
            momentum=0.9, 
            weight_decay=1e-4
        )
        
        # Learning rate scheduling is important
        self.scheduler = optim.lr_scheduler.CosineAnnealingLR(
            self.optimizer, T_max=200
        )
    
    def train_step(self, inputs, targets):
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = nn.CrossEntropyLoss()(outputs, targets)
        loss.backward()
        
        self.optimizer.step()
        
        return loss.item()

# 3. Optimization for Transformers
class TransformerOptimization:
    def __init__(self, model, d_model=512, warmup_steps=4000, lr=0.0001):
        self.model = model
        
        # Transformer-specific optimizer (AdamW)
        self.optimizer = optim.AdamW(
            model.parameters(),
            lr=lr,
            betas=(0.9, 0.98),
            eps=1e-9
        )
        
        # Transformer uses a specific learning rate schedule
        self.step_num = 0
        self.d_model = d_model
        self.warmup_steps = warmup_steps
    
    def get_lr(self):
        """Transformer learning rate schedule"""
        self.step_num += 1
        arg1 = 1 / math.sqrt(self.d_model)
        arg2 = self.step_num * (self.warmup_steps ** -1.5)
        return arg1 * min(arg1, arg2)
    
    def train_step(self, inputs, targets):
        # Update learning rate according to schedule
        new_lr = self.get_lr()
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = new_lr
        
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = nn.CrossEntropyLoss()(outputs, targets)
        loss.backward()
        
        # Gradient clipping for transformers
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        self.optimizer.step()
        
        return loss.item()

# 4. Optimization for GANs
class GANOptimization:
    def __init__(self, generator, discriminator, lr=0.0002):
        self.generator = generator
        self.discriminator = discriminator
        
        # GANs use different optimizers for G and D
        self.optimizer_G = optim.Adam(generator.parameters(), lr=lr, betas=(0.5, 0.999))
        self.optimizer_D = optim.Adam(discriminator.parameters(), lr=lr, betas=(0.5, 0.999))
        
        self.criterion = nn.BCELoss()
    
    def train_step(self, real_data):
        batch_size = real_data.size(0)
        real_labels = torch.ones(batch_size, 1, device=real_data.device)
        fake_labels = torch.zeros(batch_size, 1, device=real_data.device)
        
        # Train Discriminator
        self.optimizer_D.zero_grad()
        
        # Real data
        real_output = self.discriminator(real_data)
        d_real_loss = self.criterion(real_output, real_labels)
        
        # Fake data
        noise = torch.randn(batch_size, 100, device=real_data.device)
        fake_data = self.generator(noise)
        fake_output = self.discriminator(fake_data.detach())
        d_fake_loss = self.criterion(fake_output, fake_labels)
        
        d_loss = d_real_loss + d_fake_loss
        d_loss.backward()
        self.optimizer_D.step()
        
        # Train Generator
        self.optimizer_G.zero_grad()
        
        fake_output = self.discriminator(fake_data)
        g_loss = self.criterion(fake_output, real_labels)
        
        g_loss.backward()
        self.optimizer_G.step()
        
        return d_loss.item(), g_loss.item()

# 5. Optimization for Autoencoders
class AutoencoderOptimization:
    def __init__(self, model, lr=0.001, use_adversarial=False):
        self.model = model
        self.use_adversarial = use_adversarial
        
        # Standard optimizer for autoencoders
        self.optimizer = optim.Adam(model.parameters(), lr=lr, weight_decay=1e-5)
        
        if use_adversarial:
            # For adversarial autoencoders
            self.discriminator = nn.Linear(model.latent_dim, 1)
            self.optimizer_D = optim.Adam(self.discriminator.parameters(), lr=lr)
    
    def train_step(self, inputs):
        self.optimizer.zero_grad()
        
        reconstructed, latent = self.model(inputs)
        
        # Reconstruction loss
        recons_loss = nn.MSELoss()(reconstructed, inputs)
        
        if self.use_adversarial:
            # Adversarial loss for VAE/GAN-based AE
            adv_loss = self.adversarial_loss(latent)
            total_loss = recons_loss + 0.1 * adv_loss
        else:
            total_loss = recons_loss
        
        total_loss.backward()
        self.optimizer.step()
        
        return total_loss.item()
    
    def adversarial_loss(self, latent):
        # Implementation for adversarial component
        pass

# 6. Optimization for ResNets
class ResNetOptimization:
    def __init__(self, model, lr=0.1):
        self.model = model
        
        # ResNets benefit from specific techniques
        self.optimizer = optim.SGD(
            model.parameters(),
            lr=lr,
            momentum=0.9,
            weight_decay=1e-4,
            nesterov=True
        )
        
        # Learning rate schedule for ResNets
        self.scheduler = optim.lr_scheduler.MultiStepLR(
            self.optimizer,
            milestones=[30, 60, 90],
            gamma=0.1
        )
    
    def train_step(self, inputs, targets):
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = nn.CrossEntropyLoss()(outputs, targets)
        loss.backward()
        
        # ResNets can use gradient clipping
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=10.0)
        
        self.optimizer.step()
        
        return loss.item()

# 7. Optimization for DenseNets
class DenseNetOptimization:
    def __init__(self, model, lr=0.1):
        self.model = model
        
        # DenseNets can use standard SGD with momentum
        self.optimizer = optim.SGD(
            model.parameters(),
            lr=lr,
            momentum=0.9,
            weight_decay=1e-4
        )
        
        # DenseNet-specific considerations
        self.memory_efficient = True  # Use checkpointing if needed
    
    def train_step(self, inputs, targets):
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = nn.CrossEntropyLoss()(outputs, targets)
        loss.backward()
        
        # DenseNets may need gradient clipping due to skip connections
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        self.optimizer.step()
        
        return loss.item()

# 8. Optimization for Vision Transformers
class ViTOptimization:
    def __init__(self, model, image_size=224, patch_size=16, lr=0.003):
        self.model = model
        self.num_patches = (image_size // patch_size) ** 2
        
        # ViT-specific optimizer
        self.optimizer = optim.AdamW(
            model.parameters(),
            lr=lr,
            weight_decay=0.05
        )
        
        # Layer-wise learning rate decay
        self.apply_layer_decay(model, lr, 0.75)
    
    def apply_layer_decay(self, model, base_lr, decay_factor):
        """Apply layer-wise learning rate decay"""
        param_groups = []
        
        # Different learning rates for different layers
        for i, (name, param) in enumerate(model.named_parameters()):
            if 'blocks' in name:
                # Get block number
                block_num = int(name.split('.')[1]) if name.startswith('blocks.') else 0
                layer_lr = base_lr * (decay_factor ** (len(list(model.blocks)) - block_num))
            else:
                layer_lr = base_lr
            
            param_groups.append({
                'params': param,
                'lr': layer_lr
            })
        
        # Reinitialize optimizer with new parameter groups
        self.optimizer = optim.AdamW(
            param_groups,
            weight_decay=0.05
        )
    
    def train_step(self, inputs, targets):
        self.optimizer.zero_grad()
        
        outputs = self.model(inputs)
        loss = nn.CrossEntropyLoss()(outputs, targets)
        loss.backward()
        
        # ViT benefits from gradient clipping
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        self.optimizer.step()
        
        return loss.item()

# 9. Optimization for Recurrent Networks with Attention
class AttentionRNNOptimization:
    def __init__(self, model, lr=0.001):
        self.model = model
        
        # Attention RNNs benefit from adaptive optimizers
        self.optimizer = optim.Adam(
            model.parameters(),
            lr=lr,
            weight_decay=0.0
        )
    
    def train_step(self, inputs, targets):
        self.optimizer.zero_grad()
        
        outputs, attention_weights = self.model(inputs)
        loss = nn.CrossEntropyLoss()(outputs, targets)
        loss.backward()
        
        # Gradient clipping is important
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=1.0)
        
        self.optimizer.step()
        
        return loss.item(), attention_weights

# 10. Optimization for Normalizing Flows
class FlowOptimization:
    def __init__(self, model, lr=0.001):
        self.model = model
        
        # Flows often use Adam with specific hyperparameters
        self.optimizer = optim.Adam(
            model.parameters(),
            lr=lr,
            betas=(0.0, 0.9)
        )
    
    def train_step(self, inputs):
        self.optimizer.zero_grad()
        
        z, log_det = self.model(inputs)
        
        # Standard normal prior
        log_prob = -0.5 * torch.sum(z ** 2, dim=1) - 0.5 * math.log(2 * math.pi) * z.shape[1]
        loss = -torch.mean(log_prob + log_det)
        
        loss.backward()
        
        # Flows may need gradient clipping
        torch.nn.utils.clip_grad_norm_(self.model.parameters(), max_norm=100.0)
        
        self.optimizer.step()
        
        return loss.item()

# Architecture-specific optimization strategies
def architecture_specific_optimization():
    """Guidelines for different architectures"""
    
    print("Architecture-Specific Optimization Strategies:")
    
    print("\nRNNs/LSTMs:")
    print("- Use gradient clipping (max_norm=1.0)")
    print("- Consider adaptive optimizers (Adam, RMSprop)")
    print("- Use layer normalization")
    print("- Consider gradient clipping techniques")
    
    print("\nCNNs:")
    print("- Use momentum-based optimizers (SGD with momentum)")
    print("- Apply weight decay for regularization")
    print("- Use learning rate schedules (step, cosine)")
    print("- Batch normalization is crucial")
    
    print("\nTransformers:")
    print("- Use AdamW optimizer with specific parameters")
    print("- Apply warmup learning rate schedule")
    print("- Use gradient clipping")
    print("- Consider layer-wise learning rate decay")
    
    print("\nGANs:")
    print("- Use different optimizers for generator and discriminator")
    print("- Balance G and D training with different frequencies")
    print("- Apply label smoothing")
    print("- Consider spectral normalization")
    
    print("\nAutoencoders:")
    print("- Use reconstruction loss (MSE, BCE, L1)")
    print("- Add regularization (sparsity, denoising)")
    print("- Consider adversarial loss for better reconstructions")
    print("- Apply appropriate activation functions")
    
    print("\nResNets:")
    print("- Use SGD with momentum and Nesterov acceleration")
    print("- Apply weight decay")
    print("- Use learning rate schedules with milestones")
    print("- Consider batch normalization")
    
    print("\nDenseNets:")
    print("- Similar to ResNets but with different connectivity")
    print("- Use dropout in dense blocks")
    print("- Apply batch normalization before convolutions")
    print("- Consider memory-efficient implementations")
    
    print("\nVision Transformers:")
    print("- Use AdamW with weight decay")
    print("- Apply layer-wise learning rate decay")
    print("- Use gradient clipping")
    print("- Consider data augmentation strategies")
    
    print("\nAttention Models:")
    print("- Use scaled dot-product attention")
    print("- Apply dropout for regularization")
    print("- Use learned position embeddings")
    print("- Consider multi-head attention")

# Example usage
def demonstrate_architecture_optimization():
    """Demonstrate optimization for different architectures"""
    
    print("Demonstrating architecture-specific optimization...")

architecture_specific_optimization()