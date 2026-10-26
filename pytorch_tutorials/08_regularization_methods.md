# Regularization Methods in PyTorch - 30+ Questions and Answers

## 1. What is regularization and why is it important in deep learning?

**Answer:** Regularization is a technique used to prevent overfitting by adding constraints or penalties to the model during training. It helps the model generalize better to unseen data by reducing its complexity and preventing it from memorizing the training data. Regularization is crucial because it improves model performance on validation and test sets, making the model more robust and reliable.

## 2. What are the main types of regularization techniques?

**Answer:** The main regularization techniques include:
- **L1/L2 Regularization (Weight Decay):** Adds penalty terms to the loss function based on parameter magnitudes
- **Dropout:** Randomly sets a fraction of neurons to zero during training
- **Batch Normalization:** Normalizes layer inputs to stabilize training
- **Data Augmentation:** Increases dataset diversity through transformations
- **Early Stopping:** Stops training when validation performance degrades
- **Label Smoothing:** Softens hard labels to reduce model overconfidence

## 3. How do you implement L1 and L2 regularization in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim

# L2 Regularization (Weight Decay) - built into optimizers
optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=0.01)

# Manual L2 regularization
def l2_regularization(model, lambda_reg=0.01):
    l2_reg = torch.tensor(0., device=next(model.parameters()).device)
    for param in model.parameters():
        l2_reg += torch.norm(param, 2)
    return lambda_reg * l2_reg

# Manual L1 regularization
def l1_regularization(model, lambda_reg=0.01):
    l1_reg = torch.tensor(0., device=next(model.parameters()).device)
    for param in model.parameters():
        l1_reg += torch.norm(param, 1)
    return lambda_reg * l1_reg

# Training with manual regularization
def train_with_regularization(model, dataloader, optimizer, criterion, lambda_l1=0.0, lambda_l2=0.01):
    model.train()
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        # Add regularization terms
        if lambda_l1 > 0:
            loss += l1_regularization(model, lambda_l1)
        if lambda_l2 > 0:
            loss += l2_regularization(model, lambda_l2)
        
        loss.backward()
        optimizer.step()
```

## 4. How do you implement dropout in PyTorch?

**Answer:**
```python
import torch.nn as nn

# Using nn.Dropout layers
class NetworkWithDropout(nn.Module):
    def __init__(self, dropout_rate=0.5):
        super(NetworkWithDropout, self).__init__()
        self.fc1 = nn.Linear(784, 256)
        self.dropout1 = nn.Dropout(dropout_rate)
        self.fc2 = nn.Linear(256, 128)
        self.dropout2 = nn.Dropout(dropout_rate)
        self.fc3 = nn.Linear(128, 10)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = self.dropout1(x)  # Dropout during training
        x = self.relu(self.fc2(x))
        x = self.dropout2(x)  # Dropout during training
        x = self.fc3(x)
        return x

# Using F.dropout functionally
import torch.nn.functional as F

class FunctionalDropoutNetwork(nn.Module):
    def __init__(self, dropout_rate=0.5):
        super(FunctionalDropoutNetwork, self).__init__()
        self.fc1 = nn.Linear(784, 256)
        self.fc2 = nn.Linear(256, 128)
        self.fc3 = nn.Linear(128, 10)
        self.dropout_rate = dropout_rate
        
    def forward(self, x):
        x = F.relu(self.fc1(x))
        x = F.dropout(x, self.dropout_rate, training=self.training)
        x = F.relu(self.fc2(x))
        x = F.dropout(x, self.dropout_rate, training=self.training)
        x = self.fc3(x)
        return x

# Training mode vs Evaluation mode
model = NetworkWithDropout()
model.train()  # Enable dropout
output_train = model(input_data)

model.eval()   # Disable dropout
with torch.no_grad():
    output_eval = model(input_data)
```

## 5. What is the difference between training and evaluation modes for dropout?

**Answer:** 
- **Training Mode (`model.train()`):** Dropout is active - randomly sets a fraction of neurons to zero during forward pass, helping prevent overfitting
- **Evaluation Mode (`model.eval()`):** Dropout is inactive - all neurons are active with their original values scaled by the keep probability to maintain expected output magnitude
This distinction is crucial because dropout should only be applied during training to provide regularization, not during inference.

## 6. How do you implement batch normalization in PyTorch?

**Answer:**
```python
import torch.nn as nn

# For fully connected layers
class NetworkWithBatchNorm(nn.Module):
    def __init__(self):
        super(NetworkWithBatchNorm, self).__init__()
        self.fc1 = nn.Linear(784, 256)
        self.bn1 = nn.BatchNorm1d(256)
        self.fc2 = nn.Linear(256, 128)
        self.bn2 = nn.BatchNorm1d(128)
        self.fc3 = nn.Linear(128, 10)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        x = self.fc1(x)
        x = self.bn1(x)  # Batch normalization
        x = self.relu(x)
        x = self.fc2(x)
        x = self.bn2(x)  # Batch normalization
        x = self.relu(x)
        x = self.fc3(x)
        return x

# For convolutional layers
class CNNWithBatchNorm(nn.Module):
    def __init__(self):
        super(CNNWithBatchNorm, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.bn1 = nn.BatchNorm2d(64)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.bn2 = nn.BatchNorm2d(128)
        self.pool = nn.MaxPool2d(2, 2)
        self.fc = nn.Linear(128 * 8 * 8, 10)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        x = self.pool(self.relu(self.bn1(self.conv1(x))))
        x = self.pool(self.relu(self.bn2(self.conv2(x))))
        x = x.view(-1, 128 * 8 * 8)
        x = self.fc(x)
        return x

# Important: BatchNorm behaves differently in train vs eval modes
model = NetworkWithBatchNorm()
model.train()  # Uses batch statistics
output_train = model(input_batch)

model.eval()   # Uses running statistics
with torch.no_grad():
    output_eval = model(input_batch)
```

## 7. What are the differences between BatchNorm1d, BatchNorm2d, and BatchNorm3d?

**Answer:**
- **BatchNorm1d:** For 2D or 3D inputs (batch_size, features) or (batch_size, features, sequence_length) - used with fully connected layers or 1D convolutions
- **BatchNorm2d:** For 4D inputs (batch_size, channels, height, width) - used with 2D convolutions
- **BatchNorm3d:** For 5D inputs (batch_size, channels, depth, height, width) - used with 3D convolutions
The key difference is the dimension along which normalization is applied - each variant normalizes across the batch dimension for each channel/feature.

## 8. How do you implement different dropout rates for different layers?

**Answer:**
```python
class VariableDropoutNetwork(nn.Module):
    def __init__(self):
        super(VariableDropoutNetwork, self).__init__()
        self.fc1 = nn.Linear(784, 512)
        self.dropout1 = nn.Dropout(0.2)  # Lower dropout for early layers
        self.fc2 = nn.Linear(512, 256)
        self.dropout2 = nn.Dropout(0.5)  # Higher dropout for middle layers
        self.fc3 = nn.Linear(256, 128)
        self.dropout3 = nn.Dropout(0.3)  # Moderate dropout for later layers
        self.fc4 = nn.Linear(128, 10)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = self.dropout1(x)
        x = self.relu(self.fc2(x))
        x = self.dropout2(x)
        x = self.relu(self.fc3(x))
        x = self.dropout3(x)
        x = self.fc4(x)
        return x

# Alternative using functional dropout with different rates
class FunctionalVariableDropout(nn.Module):
    def __init__(self):
        super(FunctionalVariableDropout, self).__init__()
        self.fc1 = nn.Linear(784, 512)
        self.fc2 = nn.Linear(512, 256)
        self.fc3 = nn.Linear(256, 128)
        self.fc4 = nn.Linear(128, 10)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = F.dropout(x, 0.2, training=self.training)
        x = self.relu(self.fc2(x))
        x = F.dropout(x, 0.5, training=self.training)
        x = self.relu(self.fc3(x))
        x = F.dropout(x, 0.3, training=self.training)
        x = self.fc4(x)
        return x
```

## 9. How do you implement advanced dropout techniques like DropConnect and SpatialDropout?

**Answer:**
```python
class DropConnect(nn.Module):
    """DropConnect - randomly set weights to zero"""
    def __init__(self, layer, drop_rate=0.5):
        super(DropConnect, self).__init__()
        self.layer = layer
        self.drop_rate = drop_rate
        
    def forward(self, x):
        if self.training:
            # Create dropout mask for weights
            weight_mask = torch.bernoulli(torch.full_like(self.layer.weight, 1 - self.drop_rate))
            masked_weight = self.layer.weight * weight_mask
            return F.linear(x, masked_weight, self.layer.bias)
        else:
            # Scale weights during inference
            return self.layer(x) * (1 - self.drop_rate)

class SpatialDropout(nn.Module):
    """Spatial Dropout - drop entire feature maps"""
    def __init__(self, drop_rate=0.5):
        super(SpatialDropout, self).__init__()
        self.drop_rate = drop_rate
        
    def forward(self, x):
        if self.training:
            # For 4D input (batch, channels, height, width)
            # Create dropout mask for entire channels
            mask = torch.bernoulli(torch.full((x.size(0), x.size(1), 1, 1), 1 - self.drop_rate))
            mask = mask.to(x.device)
            return x * mask / (1 - self.drop_rate)
        else:
            return x

# Usage
class AdvancedDropoutNetwork(nn.Module):
    def __init__(self):
        super(AdvancedDropoutNetwork, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.spatial_dropout = SpatialDropout(0.2)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.fc1 = nn.Linear(128 * 8 * 8, 256)
        self.drop_connect = DropConnect(nn.Linear(256, 10), 0.5)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        x = self.pool(self.relu(self.conv1(x)))
        x = self.spatial_dropout(x)
        x = self.pool(self.relu(self.conv2(x)))
        x = x.view(-1, 128 * 8 * 8)
        x = self.relu(self.fc1(x))
        x = self.drop_connect(x)
        return x
```

## 10. How do you implement different normalization techniques?

**Answer:**
```python
class NormalizationComparison(nn.Module):
    def __init__(self, input_size=784, hidden_size=256, output_size=10):
        super(NormalizationComparison, self).__init__()
        self.fc1 = nn.Linear(input_size, hidden_size)
        
        # Batch Normalization
        self.bn = nn.BatchNorm1d(hidden_size)
        
        # Layer Normalization
        self.ln = nn.LayerNorm(hidden_size)
        
        # Instance Normalization (less common for fully connected)
        self.in_ = nn.InstanceNorm1d(hidden_size, affine=True)
        
        self.fc2 = nn.Linear(hidden_size, output_size)
        self.relu = nn.ReLU()
        
    def forward(self, x, norm_type='batch'):
        x = self.fc1(x)
        
        if norm_type == 'batch':
            x = self.bn(x)
        elif norm_type == 'layer':
            x = self.ln(x)
        elif norm_type == 'instance':
            # InstanceNorm1d expects (N, C, L) format
            x = x.unsqueeze(2)  # Add sequence dimension
            x = self.in_(x)
            x = x.squeeze(2)    # Remove sequence dimension
        
        x = self.relu(x)
        x = self.fc2(x)
        return x

# Group Normalization (useful for computer vision)
class GroupNormNetwork(nn.Module):
    def __init__(self, num_groups=32):
        super(GroupNormNetwork, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.gn1 = nn.GroupNorm(num_groups, 64)  # GroupNorm
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.gn2 = nn.GroupNorm(min(num_groups, 128), 128)
        self.pool = nn.MaxPool2d(2, 2)
        self.fc = nn.Linear(128 * 8 * 8, 10)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        x = self.pool(self.relu(self.gn1(self.conv1(x))))
        x = self.pool(self.relu(self.gn2(self.conv2(x))))
        x = x.view(-1, 128 * 8 * 8)
        x = self.fc(x)
        return x
```

## 11. How do you implement early stopping as a regularization technique?

**Answer:**
```python
class EarlyStopping:
    def __init__(self, patience=7, min_delta=0, restore_best_weights=True):
        self.patience = patience
        self.min_delta = min_delta
        self.restore_best_weights = restore_best_weights
        self.best_loss = None
        self.best_weights = None
        self.counter = 0
        self.early_stop = False
        
    def __call__(self, val_loss, model=None):
        if self.best_loss is None:
            self.best_loss = val_loss
            if model is not None:
                self.best_weights = model.state_dict().copy()
        elif val_loss > self.best_loss - self.min_delta:
            self.counter += 1
            if self.counter >= self.patience:
                self.early_stop = True
                if self.restore_best_weights and model is not None and self.best_weights is not None:
                    model.load_state_dict(self.best_weights)
        else:
            self.best_loss = val_loss
            self.counter = 0
            if model is not None:
                self.best_weights = model.state_dict().copy()

# Usage in training loop
def train_with_early_stopping(model, train_loader, val_loader, optimizer, criterion, 
                             patience=10, max_epochs=100):
    early_stopping = EarlyStopping(patience=patience)
    
    for epoch in range(max_epochs):
        # Training phase
        model.train()
        train_loss = 0.0
        for inputs, targets in train_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            loss.backward()
            optimizer.step()
            train_loss += loss.item()
        
        # Validation phase
        model.eval()
        val_loss = 0.0
        with torch.no_grad():
            for inputs, targets in val_loader:
                inputs, targets = inputs.to(device), targets.to(device)
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                val_loss += loss.item()
        
        avg_val_loss = val_loss / len(val_loader)
        print(f'Epoch {epoch+1}, Train Loss: {train_loss/len(train_loader):.4f}, Val Loss: {avg_val_loss:.4f}')
        
        # Check early stopping
        early_stopping(avg_val_loss, model)
        if early_stopping.early_stop:
            print("Early stopping triggered")
            break
    
    return model
```

## 12. How do you implement data augmentation as regularization?

**Answer:**
```python
import torchvision.transforms as transforms

# For image data
train_transform = transforms.Compose([
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.RandomRotation(degrees=15),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2, hue=0.1),
    transforms.RandomResizedCrop(224, scale=(0.8, 1.0)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# For evaluation (no augmentation)
val_transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# Custom data augmentation with albumentations (more advanced)
import albumentations as A
from albumentations.pytorch import ToTensorV2

def get_advanced_transforms():
    train_transform = A.Compose([
        A.HorizontalFlip(p=0.5),
        A.ShiftScaleRotate(shift_limit=0.0625, scale_limit=0.2, rotate_limit=15, p=0.9),
        A.RandomBrightnessContrast(brightness_limit=0.2, contrast_limit=0.2, p=0.9),
        A.HueSaturationValue(hue_shift_limit=20, sat_shift_limit=30, val_shift_limit=20, p=0.9),
        A.GaussNoise(var_limit=(10.0, 50.0), p=0.5),
        A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
        ToTensorV2()
    ])
    
    val_transform = A.Compose([
        A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
        ToTensorV2()
    ])
    
    return train_transform, val_transform

# Mixup regularization
def mixup_data(x, y, alpha=1.0):
    """Compute the mixup data. Return mixed inputs, pairs of targets, and lambda"""
    if alpha > 0:
        lam = np.random.beta(alpha, alpha)
    else:
        lam = 1
    
    batch_size = x.size()[0]
    index = torch.randperm(batch_size).to(x.device)
    
    mixed_x = lam * x + (1 - lam) * x[index, :]
    y_a, y_b = y, y[index]
    return mixed_x, y_a, y_b, lam

def mixup_criterion(criterion, pred, y_a, y_b, lam):
    return lam * criterion(pred, y_a) + (1 - lam) * criterion(pred, y_b)

# Usage in training loop
def train_with_mixup(model, dataloader, optimizer, criterion, alpha=1.0):
    model.train()
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        # Apply mixup
        inputs, targets_a, targets_b, lam = mixup_data(inputs, targets, alpha)
        
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = mixup_criterion(criterion, outputs, targets_a, targets_b, lam)
        loss.backward()
        optimizer.step()
```

## 13. How do you implement label smoothing as regularization?

**Answer:**
```python
import torch.nn.functional as F

class LabelSmoothingLoss(nn.Module):
    def __init__(self, smoothing=0.1, num_classes=10):
        super(LabelSmoothingLoss, self).__init__()
        self.smoothing = smoothing
        self.num_classes = num_classes
        self.confidence = 1.0 - smoothing
        
    def forward(self, predictions, targets):
        log_probs = F.log_softmax(predictions, dim=-1)
        
        # Create smoothed targets
        with torch.no_grad():
            true_dist = torch.zeros_like(log_probs)
            true_dist.fill_(self.smoothing / (self.num_classes - 1))
            true_dist.scatter_(1, targets.unsqueeze(1), self.confidence)
        
        # Compute loss
        loss = torch.mean(torch.sum(-true_dist * log_probs, dim=-1))
        return loss

# Alternative implementation
def label_smoothing_cross_entropy(predictions, targets, smoothing=0.1):
    """Label smoothing cross entropy loss"""
    confidence = 1.0 - smoothing
    log_probs = F.log_softmax(predictions, dim=-1)
    num_classes = predictions.size(-1)
    
    # Create smoothed targets
    with torch.no_grad():
        true_dist = torch.zeros_like(log_probs)
        true_dist.fill_(smoothing / (num_classes - 1))
        true_dist.scatter_(1, targets.unsqueeze(1), confidence)
    
    return torch.mean(torch.sum(-true_dist * log_probs, dim=-1))

# Usage
criterion = LabelSmoothingLoss(smoothing=0.1, num_classes=10)
# or
# loss = label_smoothing_cross_entropy(outputs, targets, smoothing=0.1)
```

## 14. How do you implement weight initialization techniques for regularization?

**Answer:**
```python
def xavier_init(model):
    """Xavier/Glorot initialization"""
    for name, param in model.named_parameters():
        if 'weight' in name and param.dim() > 1:
            nn.init.xavier_uniform_(param)
        elif 'bias' in name:
            nn.init.zeros_(param)

def he_init(model):
    """He initialization for ReLU networks"""
    for name, param in model.named_parameters():
        if 'weight' in name and param.dim() > 1:
            nn.init.kaiming_normal_(param, mode='fan_out', nonlinearity='relu')
        elif 'bias' in name:
            nn.init.zeros_(param)

def orthogonal_init(model):
    """Orthogonal initialization"""
    for name, param in model.named_parameters():
        if 'weight' in name and param.dim() > 1:
            nn.init.orthogonal_(param)
        elif 'bias' in name:
            nn.init.zeros_(param)

# Custom initialization with different strategies per layer
def custom_init(model, init_config=None):
    """Custom initialization with different strategies per layer type"""
    if init_config is None:
        init_config = {
            'conv': 'he',
            'linear': 'xavier',
            'bn': 'normal',
            'default': 'xavier'
        }
    
    for name, param in model.named_parameters():
        if 'conv' in name and param.dim() > 1:
            if init_config['conv'] == 'he':
                nn.init.kaiming_normal_(param, mode='fan_out', nonlinearity='relu')
            elif init_config['conv'] == 'xavier':
                nn.init.xavier_normal_(param)
        elif ('linear' in name or 'classifier' in name) and param.dim() > 1:
            if init_config['linear'] == 'xavier':
                nn.init.xavier_normal_(param)
            elif init_config['linear'] == 'he':
                nn.init.kaiming_normal_(param, mode='fan_out', nonlinearity='relu')
        elif 'bias' in name:
            nn.init.zeros_(param)
        else:
            # Default initialization
            if init_config['default'] == 'xavier':
                nn.init.xavier_normal_(param)

# Usage
model = MyModel()
custom_init(model, {'conv': 'he', 'linear': 'xavier', 'default': 'orthogonal'})
```

## 15. How do you implement spectral normalization for regularization?

**Answer:**
```python
class SpectralNorm(nn.Module):
    """Spectral normalization for weight matrices"""
    def __init__(self, module, name='weight', power_iterations=1):
        super(SpectralNorm, self).__init__()
        self.module = module
        self.name = name
        self.power_iterations = power_iterations
        self.register_buffer('u', torch.randn(1, getattr(module, name).size(0)))
        
    def forward(self, *args):
        self._update_u_v()
        return self.module.forward(*args)
    
    def _update_u_v(self):
        w = getattr(self.module, self.name)
        u = self.u
        
        with torch.no_grad():
            for _ in range(self.power_iterations):
                v = F.normalize(torch.mv(w.data.t(), u[0]), dim=0, eps=1e-12)
                u = F.normalize(torch.mv(w.data, v), dim=0, eps=1e-12)
            
            sigma = torch.dot(u[0], torch.mv(w.data, v))
            self.u.copy_(u)
            setattr(self.module, self.name, w / sigma.expand_as(w))

# Using PyTorch's built-in spectral norm
def apply_spectral_norm(model):
    """Apply spectral normalization to all linear and convolutional layers"""
    for name, module in model.named_modules():
        if isinstance(module, (nn.Linear, nn.Conv2d)):
            spectral_norm_module = nn.utils.spectral_norm(module)
            # Replace the module in the model
            parent_module = model
            name_parts = name.split('.')
            for part in name_parts[:-1]:
                parent_module = getattr(parent_module, part)
            setattr(parent_module, name_parts[-1], spectral_norm_module)
    
    return model

# Usage
model = MyModel()
model = apply_spectral_norm(model)
```

## 16. How do you implement virtual adversarial training (VAT) as regularization?

**Answer:**
```python
class VATLoss(nn.Module):
    """Virtual Adversarial Training Loss"""
    def __init__(self, xi=1e-6, eps=1.0, ip=1):
        super(VATLoss, self).__init__()
        self.xi = xi
        self.eps = eps
        self.ip = ip
    
    def forward(self, model, x):
        with torch.no_grad():
            pred = F.softmax(model(x), dim=1)
        
        # Prepare random unit tensor
        d = torch.randn_like(x, device=x.device)
        d = F.normalize(d.view(d.size(0), -1), dim=1, p=2).view(d.size())
        d.requires_grad_()
        
        # Calculate VAT loss
        for _ in range(self.ip):
            d.requires_grad_()
            pred_hat = model(x + self.xi * d)
            logp_hat = F.log_softmax(pred_hat, dim=1)
            adv_distance = F.kl_div(logp_hat, pred, reduction='batchmean')
            adv_distance.backward()
            d = F.normalize(d.grad, dim=-1, p=2).detach()
        
        r_adv = d * self.eps
        
        # Calculate final VAT loss
        pred_hat = model(x + r_adv)
        logp_hat = F.log_softmax(pred_hat, dim=1)
        lds = F.kl_div(logp_hat, pred, reduction='batchmean')
        
        return lds

# Usage in training loop
def train_with_vat(model, dataloader, optimizer, criterion, vat_weight=1.0):
    vat_loss = VATLoss()
    model.train()
    
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        optimizer.zero_grad()
        outputs = model(inputs)
        
        # Standard classification loss
        classification_loss = criterion(outputs, targets)
        
        # VAT regularization loss
        vat_reg_loss = vat_loss(model, inputs)
        
        # Combined loss
        total_loss = classification_loss + vat_weight * vat_reg_loss
        
        total_loss.backward()
        optimizer.step()
```

## 17. How do you implement cutout and random erasing as regularization?

**Answer:**
```python
class Cutout:
    """Randomly mask out one or more patches from an image"""
    def __init__(self, n_holes=1, length=16):
        self.n_holes = n_holes
        self.length = length
    
    def __call__(self, img):
        h = img.size(1)
        w = img.size(2)
        
        mask = np.ones((h, w), np.float32)
        
        for n in range(self.n_holes):
            y = np.random.randint(h)
            x = np.random.randint(w)
            
            y1 = np.clip(y - self.length // 2, 0, h)
            y2 = np.clip(y + self.length // 2, 0, h)
            x1 = np.clip(x - self.length // 2, 0, w)
            x2 = np.clip(x + self.length // 2, 0, w)
            
            mask[y1: y2, x1: x2] = 0.
        
        mask = torch.from_numpy(mask)
        mask = mask.expand_as(img)
        img = img * mask
        
        return img

# Using torchvision's RandomErasing (PyTorch 1.5+)
from torchvision.transforms import RandomErasing

# Add to transforms
train_transform = transforms.Compose([
    transforms.RandomHorizontalFlip(),
    transforms.ToTensor(),
    RandomErasing(p=0.5, scale=(0.02, 0.33), ratio=(0.3, 3.3), value=0, inplace=False)
])

# Custom implementation
class RandomErasingCustom:
    def __init__(self, probability=0.5, sl=0.02, sh=0.4, r1=0.3, mean=[0.4914, 0.4822, 0.4465]):
        self.probability = probability
        self.mean = mean
        self.sl = sl
        self.sh = sh
        self.r1 = r1
    
    def __call__(self, img):
        if random.uniform(0, 1) > self.probability:
            return img
        
        for attempt in range(100):
            area = img.size()[1] * img.size()[2]
            
            target_area = random.uniform(self.sl, self.sh) * area
            aspect_ratio = random.uniform(self.r1, 1/self.r1)
            
            h = int(round(math.sqrt(target_area * aspect_ratio)))
            w = int(round(math.sqrt(target_area / aspect_ratio)))
            
            if w < img.size()[2] and h < img.size()[1]:
                x1 = random.randint(0, img.size()[1] - h)
                y1 = random.randint(0, img.size()[2] - w)
                
                if img.size()[0] == 3:
                    img[0, x1:x1+h, y1:y1+w] = self.mean[0]
                    img[1, x1:x1+h, y1:y1+w] = self.mean[1]
                    img[2, x1:x1+h, y1:y1+w] = self.mean[2]
                else:
                    img[0, x1:x1+h, y1:y1+w] = self.mean[0]
                return img
        
        return img
```

## 18. How do you implement noise injection as regularization?

**Answer:**
```python
class NoiseInjection(nn.Module):
    """Add noise to inputs or hidden layers"""
    def __init__(self, noise_factor=0.1):
        super(NoiseInjection, self).__init__()
        self.noise_factor = noise_factor
    
    def forward(self, x):
        if self.training:
            noise = torch.randn_like(x) * self.noise_factor
            return x + noise
        return x

class GaussianNoiseLayer(nn.Module):
    """Add Gaussian noise to inputs"""
    def __init__(self, std=0.1):
        super(GaussianNoiseLayer, self).__init__()
        self.std = std
        self.noise = torch.tensor(0.0)
        
    def forward(self, x):
        if self.training:
            sampled_noise = self.noise.repeat(*x.size()).normal_(mean=0, std=self.std)
            x = x + sampled_noise.to(x.device)
        return x

# Add noise to inputs
class NoisyInputNetwork(nn.Module):
    def __init__(self, noise_std=0.1):
        super(NoisyInputNetwork, self).__init__()
        self.noise_layer = GaussianNoiseLayer(std=noise_std)
        self.fc1 = nn.Linear(784, 256)
        self.fc2 = nn.Linear(256, 128)
        self.fc3 = nn.Linear(128, 10)
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(0.5)
        
    def forward(self, x):
        x = self.noise_layer(x)  # Add noise to inputs
        x = self.relu(self.fc1(x))
        x = self.dropout(x)
        x = self.relu(self.fc2(x))
        x = self.dropout(x)
        x = self.fc3(x)
        return x

# Add noise to gradients (gradient noise)
def add_gradient_noise(model, noise_factor=0.01):
    """Add noise to gradients"""
    for param in model.parameters():
        if param.grad is not None:
            noise = torch.randn_like(param.grad) * noise_factor
            param.grad.add_(noise)

# Usage in training loop
def train_with_gradient_noise(model, dataloader, optimizer, criterion, noise_factor=0.01):
    model.train()
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()
        
        # Add gradient noise
        add_gradient_noise(model, noise_factor)
        
        optimizer.step()
```

## 19. How do you implement adversarial training as regularization?

**Answer:**
```python
def generate_fgsm_adversarial_example(model, x, y, epsilon=0.01):
    """Generate FGSM adversarial example"""
    x_adv = x.clone().detach().requires_grad_(True)
    
    output = model(x_adv)
    loss = F.cross_entropy(output, y)
    loss.backward()
    
    # Fast Gradient Sign Method
    x_adv = x_adv + epsilon * x_adv.grad.sign()
    
    # Clamp to valid range [0,1]
    x_adv = torch.clamp(x_adv, 0, 1)
    
    return x_adv.detach()

def train_adversarial_fgsm(model, dataloader, optimizer, criterion, epsilon=0.01):
    """Train with FGSM adversarial examples"""
    model.train()
    total_loss = 0.0
    
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        # Generate adversarial examples
        inputs_adv = generate_fgsm_adversarial_example(model, inputs, targets, epsilon)
        
        optimizer.zero_grad()
        
        # Train on both clean and adversarial examples
        outputs_clean = model(inputs)
        outputs_adv = model(inputs_adv)
        
        loss_clean = criterion(outputs_clean, targets)
        loss_adv = criterion(outputs_adv, targets)
        
        # Combined loss
        loss = (loss_clean + loss_adv) / 2
        
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
    
    return total_loss / len(dataloader)

# PGD (Projected Gradient Descent) adversarial training
def generate_pgd_adversarial_example(model, x, y, epsilon=0.01, alpha=0.001, num_iter=10):
    """Generate PGD adversarial example"""
    x_adv = x.clone().detach() + torch.zeros_like(x).uniform_(-epsilon, epsilon)
    x_adv = torch.clamp(x_adv, 0, 1)
    
    for _ in range(num_iter):
        x_adv.requires_grad_(True)
        output = model(x_adv)
        loss = F.cross_entropy(output, y)
        loss.backward()
        
        # Update adversarial example
        x_adv = x_adv + alpha * x_adv.grad.sign()
        
        # Project back to epsilon ball
        eta = torch.clamp(x_adv - x, -epsilon, epsilon)
        x_adv = torch.clamp(x + eta, 0, 1).detach()
    
    return x_adv

def train_adversarial_pgd(model, dataloader, optimizer, criterion, 
                         epsilon=0.01, alpha=0.001, num_iter=10):
    """Train with PGD adversarial examples"""
    model.train()
    total_loss = 0.0
    
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        # Generate PGD adversarial examples
        inputs_adv = generate_pgd_adversarial_example(model, inputs, targets, 
                                                     epsilon, alpha, num_iter)
        
        optimizer.zero_grad()
        
        # Train on both clean and adversarial examples
        outputs_clean = model(inputs)
        outputs_adv = model(inputs_adv)
        
        loss_clean = criterion(outputs_clean, targets)
        loss_adv = criterion(outputs_adv, targets)
        
        # Combined loss
        loss = (loss_clean + loss_adv) / 2
        
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
    
    return total_loss / len(dataloader)
```

## 20. How do you implement different regularization combinations?

**Answer:**
```python
class ComprehensiveRegularizationNetwork(nn.Module):
    """Network with multiple regularization techniques combined"""
    def __init__(self, input_size=784, hidden_sizes=[256, 128], output_size=10, 
                 dropout_rates=[0.3, 0.5], batch_norm=True):
        super(ComprehensiveRegularizationNetwork, self).__init__()
        
        layers = []
        prev_size = input_size
        
        for i, hidden_size in enumerate(hidden_sizes):
            # Linear layer
            layers.append(nn.Linear(prev_size, hidden_size))
            
            # Batch normalization (optional)
            if batch_norm:
                layers.append(nn.BatchNorm1d(hidden_size))
            
            # Activation
            layers.append(nn.ReLU())
            
            # Dropout
            layers.append(nn.Dropout(dropout_rates[min(i, len(dropout_rates)-1)]))
            
            prev_size = hidden_size
        
        # Output layer (no regularization)
        layers.append(nn.Linear(prev_size, output_size))
        
        self.network = nn.Sequential(*layers)
    
    def forward(self, x):
        return self.network(x)

# Training with multiple regularization techniques
def train_with_comprehensive_regularization(model, train_loader, val_loader, 
                                          optimizer, criterion, scheduler=None,
                                          early_stopping_patience=10, 
                                          label_smoothing=0.1,
                                          l2_weight_decay=0.01,
                                          gradient_clip_norm=1.0,
                                          mixup_alpha=0.2):
    
    # Setup early stopping
    early_stopping = EarlyStopping(patience=early_stopping_patience)
    
    # Setup label smoothing
    if label_smoothing > 0:
        criterion = LabelSmoothingLoss(smoothing=label_smoothing, 
                                     num_classes=criterion.num_classes)
    
    for epoch in range(100):  # Max epochs
        # Training phase
        model.train()
        train_loss = 0.0
        
        for inputs, targets in train_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            # Mixup augmentation
            if mixup_alpha > 0:
                inputs, targets_a, targets_b, lam = mixup_data(inputs, targets, mixup_alpha)
            
            optimizer.zero_grad()
            outputs = model(inputs)
            
            # Loss calculation
            if mixup_alpha > 0:
                loss = mixup_criterion(criterion, outputs, targets_a, targets_b, lam)
            else:
                loss = criterion(outputs, targets)
            
            loss.backward()
            
            # Gradient clipping
            if gradient_clip_norm > 0:
                torch.nn.utils.clip_grad_norm_(model.parameters(), gradient_clip_norm)
            
            optimizer.step()
            train_loss += loss.item()
        
        # Validation phase
        model.eval()
        val_loss = 0.0
        correct = 0
        total = 0
        
        with torch.no_grad():
            for inputs, targets in val_loader:
                inputs, targets = inputs.to(device), targets.to(device)
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                val_loss += loss.item()
                
                _, predicted = torch.max(outputs.data, 1)
                total += targets.size(0)
                correct += (predicted == targets).sum().item()
        
        avg_val_loss = val_loss / len(val_loader)
        accuracy = 100 * correct / total
        
        print(f'Epoch {epoch+1}: Train Loss: {train_loss/len(train_loader):.4f}, '
              f'Val Loss: {avg_val_loss:.4f}, Accuracy: {accuracy:.2f}%')
        
        # Learning rate scheduling
        if scheduler is not None:
            if isinstance(scheduler, torch.optim.lr_scheduler.ReduceLROnPlateau):
                scheduler.step(avg_val_loss)
            else:
                scheduler.step()
        
        # Early stopping
        early_stopping(avg_val_loss, model)
        if early_stopping.early_stop:
            print("Early stopping triggered")
            break
    
    return model
```

## 21. How do you implement regularization scheduling during training?

**Answer:**
```python
class RegularizationScheduler:
    """Schedule regularization hyperparameters during training"""
    def __init__(self, initial_values, schedule_type='linear'):
        self.initial_values = initial_values
        self.schedule_type = schedule_type
        self.current_values = initial_values.copy()
        self.step_count = 0
    
    def step(self, epoch=None):
        self.step_count += 1
        if epoch is None:
            epoch = self.step_count
            
        # Update regularization values based on schedule
        for key, initial_value in self.initial_values.items():
            if self.schedule_type == 'linear':
                # Linear decay
                self.current_values[key] = max(0.0, initial_value * (1.0 - epoch / 100))
            elif self.schedule_type == 'exponential':
                # Exponential decay
                self.current_values[key] = initial_value * (0.95 ** epoch)
            elif self.schedule_type == 'step':
                # Step decay
                if epoch > 50:
                    self.current_values[key] = initial_value * 0.1
                elif epoch > 25:
                    self.current_values[key] = initial_value * 0.5
    
    def get_values(self):
        return self.current_values.copy()

# Usage in training loop
def train_with_scheduled_regularization(model, dataloader, optimizer, criterion):
    # Initialize regularization scheduler
    reg_scheduler = RegularizationScheduler({
        'dropout_rate': 0.5,
        'weight_decay': 0.01,
        'label_smoothing': 0.1
    }, schedule_type='linear')
    
    for epoch in range(100):
        # Update regularization values
        reg_scheduler.step(epoch)
        reg_values = reg_scheduler.get_values()
        
        # Update dropout rates (requires model modification)
        update_model_dropout_rates(model, reg_values['dropout_rate'])
        
        # Update optimizer with new weight decay
        for param_group in optimizer.param_groups:
            param_group['weight_decay'] = reg_values['weight_decay']
        
        # Update criterion with new label smoothing
        criterion = LabelSmoothingLoss(smoothing=reg_values['label_smoothing'])
        
        # Training code...
        print(f"Epoch {epoch}: Dropout={reg_values['dropout_rate']:.3f}, "
              f"Weight Decay={reg_values['weight_decay']:.4f}, "
              f"Label Smoothing={reg_values['label_smoothing']:.3f}")

def update_model_dropout_rates(model, dropout_rate):
    """Update dropout rates in model"""
    for module in model.modules():
        if isinstance(module, nn.Dropout):
            module.p = dropout_rate
```

## 22. How do you implement group regularization (group lasso)?

**Answer:**
```python
class GroupLassoRegularization:
    """Group Lasso regularization for structured sparsity"""
    def __init__(self, groups, lambda_reg=0.01):
        self.groups = groups  # List of parameter groups
        self.lambda_reg = lambda_reg
    
    def compute_regularization(self, model):
        """Compute group lasso penalty"""
        reg_loss = 0.0
        
        for group_indices in self.groups:
            # Compute L2 norm for each group
            group_norm = 0.0
            for idx in group_indices:
                param = list(model.parameters())[idx]
                group_norm += torch.sum(param ** 2)
            group_norm = torch.sqrt(group_norm)
            reg_loss += group_norm
        
        return self.lambda_reg * reg_loss

# Example: Group regularization for CNN channels
class GroupRegularizedCNN(nn.Module):
    def __init__(self):
        super(GroupRegularizedCNN, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.fc = nn.Linear(128 * 8 * 8, 10)
        
        # Define groups (e.g., each output channel of conv1 as a group)
        self.define_groups()
    
    def define_groups(self):
        """Define parameter groups for regularization"""
        self.groups = []
        # Each output channel of conv1 as a group
        conv1_params_per_channel = self.conv1.weight.shape[1] * self.conv1.weight.shape[2] * self.conv1.weight.shape[3]
        for i in range(self.conv1.out_channels):
            start_idx = i * conv1_params_per_channel
            end_idx = (i + 1) * conv1_params_per_channel
            self.groups.append(list(range(start_idx, end_idx)))
    
    def forward(self, x):
        x = self.pool(F.relu(self.conv1(x)))
        x = self.pool(F.relu(self.conv2(x)))
        x = x.view(-1, 128 * 8 * 8)
        x = self.fc(x)
        return x

# Usage
def train_with_group_regularization(model, dataloader, optimizer, criterion, lambda_group=0.01):
    model.train()
    group_reg = GroupLassoRegularization(model.groups, lambda_group)
    
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        # Add group regularization
        reg_loss = group_reg.compute_regularization(model)
        loss += reg_loss
        
        loss.backward()
        optimizer.step()
```

## 23. How do you implement elastic net regularization?

**Answer:**
```python
class ElasticNetRegularization:
    """Elastic Net regularization (combination of L1 and L2)"""
    def __init__(self, alpha=0.5, lambda_reg=0.01):
        self.alpha = alpha  # L1 ratio (0 = L2 only, 1 = L1 only)
        self.lambda_reg = lambda_reg
    
    def compute_regularization(self, model):
        """Compute elastic net penalty"""
        l1_reg = torch.tensor(0., device=next(model.parameters()).device)
        l2_reg = torch.tensor(0., device=next(model.parameters()).device)
        
        for param in model.parameters():
            l1_reg += torch.sum(torch.abs(param))
            l2_reg += torch.sum(param ** 2)
        
        # Elastic net: alpha * L1 + (1 - alpha) * L2
        elastic_net_reg = self.alpha * l1_reg + (1 - self.alpha) * l2_reg
        return self.lambda_reg * elastic_net_reg

# Alternative implementation with separate L1 and L2 weights
def elastic_net_regularization(model, l1_weight=0.005, l2_weight=0.005):
    """Elastic net regularization with separate L1 and L2 weights"""
    l1_reg = torch.tensor(0., device=next(model.parameters()).device)
    l2_reg = torch.tensor(0., device=next(model.parameters()).device)
    
    for param in model.parameters():
        l1_reg += torch.sum(torch.abs(param))
        l2_reg += torch.sum(param ** 2)
    
    return l1_weight * l1_reg + l2_weight * l2_reg

# Usage
def train_with_elastic_net(model, dataloader, optimizer, criterion, 
                          l1_weight=0.005, l2_weight=0.005):
    model.train()
    
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        # Add elastic net regularization
        reg_loss = elastic_net_regularization(model, l1_weight, l2_weight)
        loss += reg_loss
        
        loss.backward()
        optimizer.step()
```

## 24. How do you implement pathwise regularization?

**Answer:**
```python
class PathwiseRegularization:
    """Pathwise regularization for neural networks"""
    def __init__(self, lambda_reg=0.01):
        self.lambda_reg = lambda_reg
    
    def compute_regularization(self, model):
        """Compute pathwise regularization penalty"""
        reg_loss = 0.0
        
        # Get all parameters
        params = list(model.parameters())
        
        # Regularize paths between consecutive layers
        for i in range(len(params) - 1):
            if params[i].dim() == 2 and params[i+1].dim() == 2:  # Linear layers
                # Compute path regularization between consecutive weight matrices
                path_reg = torch.sum(torch.abs(params[i]) * torch.sum(torch.abs(params[i+1]), dim=0, keepdim=True))
                reg_loss += path_reg
        
        return self.lambda_reg * reg_loss

# Implementation using activation paths
class PathRegularizedNetwork(nn.Module):
    def __init__(self, input_size=784, hidden_sizes=[256, 128], output_size=10):
        super(PathRegularizedNetwork, self).__init__()
        
        self.layers = nn.ModuleList()
        prev_size = input_size
        
        for hidden_size in hidden_sizes:
            self.layers.append(nn.Linear(prev_size, hidden_size))
            prev_size = hidden_size
        
        self.output_layer = nn.Linear(prev_size, output_size)
        self.relu = nn.ReLU()
    
    def forward(self, x, return_activations=False):
        activations = [x] if return_activations else None
        
        for layer in self.layers:
            x = layer(x)
            x = self.relu(x)
            if return_activations:
                activations.append(x)
        
        x = self.output_layer(x)
        
        if return_activations:
            activations.append(x)
            return x, activations
        return x

def pathwise_regularization_loss(activations, lambda_reg=0.01):
    """Compute pathwise regularization based on activation paths"""
    reg_loss = 0.0
    
    for i in range(len(activations) - 1):
        # Regularize the path between consecutive activations
        act1 = activations[i]
        act2 = activations[i + 1]
        
        # Compute path regularization
        path_reg = torch.sum(torch.abs(act1)) * torch.sum(torch.abs(act2))
        reg_loss += path_reg
    
    return lambda_reg * reg_loss

# Usage
def train_with_path_regularization(model, dataloader, optimizer, criterion, lambda_reg=0.01):
    model.train()
    
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        optimizer.zero_grad()
        outputs, activations = model(inputs, return_activations=True)
        loss = criterion(outputs, targets)
        
        # Add pathwise regularization
        path_reg_loss = pathwise_regularization_loss(activations, lambda_reg)
        loss += path_reg_loss
        
        loss.backward()
        optimizer.step()
```

## 25. How do you implement dropout uncertainty estimation?

**Answer:**
```python
class MonteCarloDropout:
    """Use dropout at inference time for uncertainty estimation"""
    def __init__(self, model, dropout_rate=0.5):
        self.model = model
        self.dropout_rate = dropout_rate
        self.enable_dropout()
    
    def enable_dropout(self):
        """Enable dropout during inference"""
        for module in self.model.modules():
            if isinstance(module, nn.Dropout):
                module.train()  # Enable dropout in eval mode
    
    def predict(self, x, n_samples=100):
        """Get multiple predictions with different dropout masks"""
        self.model.eval()  # Set to eval mode but with dropout enabled
        predictions = []
        
        with torch.no_grad():
            for _ in range(n_samples):
                pred = self.model(x)
                predictions.append(pred)
        
        predictions = torch.stack(predictions)
        return predictions
    
    def predict_with_uncertainty(self, x, n_samples=100):
        """Get predictions with uncertainty estimates"""
        predictions = self.predict(x, n_samples)
        
        # Mean prediction
        mean_pred = torch.mean(predictions, dim=0)
        
        # Uncertainty (variance across samples)
        uncertainty = torch.var(predictions, dim=0)
        
        # Confidence (inverse of uncertainty)
        confidence = 1.0 / (uncertainty + 1e-8)
        
        return mean_pred, uncertainty, confidence

# Bayesian neural network with dropout
class BayesianNetwork(nn.Module):
    def __init__(self, input_size=784, hidden_sizes=[256, 128], output_size=10):
        super(BayesianNetwork, self).__init__()
        
        layers = []
        prev_size = input_size
        
        for hidden_size in hidden_sizes:
            layers.append(nn.Linear(prev_size, hidden_size))
            layers.append(nn.ReLU())
            layers.append(nn.Dropout(0.5))  # Always keep dropout
            prev_size = hidden_size
        
        layers.append(nn.Linear(prev_size, output_size))
        self.network = nn.Sequential(*layers)
    
    def forward(self, x):
        return self.network(x)
    
    def predict_proba(self, x, n_samples=100):
        """Predict with uncertainty using Monte Carlo dropout"""
        self.train()  # Enable dropout
        predictions = []
        
        with torch.no_grad():
            for _ in range(n_samples):
                logits = self.forward(x)
                probs = F.softmax(logits, dim=-1)
                predictions.append(probs)
        
        predictions = torch.stack(predictions)
        mean_probs = torch.mean(predictions, dim=0)
        uncertainty = torch.var(predictions, dim=0)
        
        return mean_probs, uncertainty

# Usage
model = BayesianNetwork()
mc_dropout = MonteCarloDropout(model)

# During inference
with torch.no_grad():
    mean_pred, uncertainty, confidence = mc_dropout.predict_with_uncertainty(test_input, n_samples=50)
    print(f"Mean prediction shape: {mean_pred.shape}")
    print(f"Uncertainty shape: {uncertainty.shape}")
    print(f"Confidence shape: {confidence.shape}")
```

## 26. How do you implement structured dropout?

**Answer:**
```python
class StructuredDropout(nn.Module):
    """Dropout that drops structured patterns (e.g., entire channels, filters)"""
    def __init__(self, drop_type='channel', drop_rate=0.5):
        super(StructuredDropout, self).__init__()
        self.drop_type = drop_type
        self.drop_rate = drop_rate
    
    def forward(self, x):
        if not self.training:
            return x
        
        if self.drop_type == 'channel':
            return self._channel_dropout(x)
        elif self.drop_type == 'filter':
            return self._filter_dropout(x)
        elif self.drop_type == 'block':
            return self._block_dropout(x)
        else:
            return F.dropout(x, self.drop_rate, self.training)
    
    def _channel_dropout(self, x):
        """Drop entire channels (for convolutional layers)"""
        if x.dim() != 4:  # (batch, channels, height, width)
            return F.dropout(x, self.drop_rate, self.training)
        
        # Create dropout mask for channels
        batch_size, num_channels = x.size(0), x.size(1)
        channel_mask = torch.bernoulli(torch.full((batch_size, num_channels, 1, 1), 
                                                 1 - self.drop_rate, device=x.device))
        return x * channel_mask / (1 - self.drop_rate)
    
    def _filter_dropout(self, x):
        """Drop entire filters (for linear layers viewed as 1x1 conv)"""
        if x.dim() != 2:  # (batch, features)
            return F.dropout(x, self.drop_rate, self.training)
        
        # Reshape as if it's a 1x1 conv: (batch, features, 1, 1)
        x = x.unsqueeze(2).unsqueeze(3)
        x = self._channel_dropout(x)
        return x.squeeze(3).squeeze(2)
    
    def _block_dropout(self, x):
        """Drop spatial blocks"""
        if x.dim() != 4:
            return F.dropout(x, self.drop_rate, self.training)
        
        # Simple block dropout (drop 2x2 blocks)
        batch_size, channels, height, width = x.size()
        block_size = 2
        
        # Create block mask
        block_h, block_w = height // block_size, width // block_size
        block_mask = torch.bernoulli(torch.full((batch_size, channels, block_h, block_w), 
                                               1 - self.drop_rate, device=x.device))
        
        # Upsample mask to original size
        mask = block_mask.repeat(1, 1, block_size, block_size)
        if mask.size(2) != height or mask.size(3) != width:
            mask = F.pad(mask, (0, width - mask.size(3), 0, height - mask.size(2)))
        
        return x * mask / (1 - self.drop_rate)

# Network with structured dropout
class StructuredDropoutNetwork(nn.Module):
    def __init__(self):
        super(StructuredDropoutNetwork, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.channel_dropout1 = StructuredDropout('channel', 0.2)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.channel_dropout2 = StructuredDropout('channel', 0.3)
        self.pool = nn.MaxPool2d(2, 2)
        self.fc1 = nn.Linear(128 * 8 * 8, 256)
        self.filter_dropout = StructuredDropout('filter', 0.5)
        self.fc2 = nn.Linear(256, 10)
        self.relu = nn.ReLU()
    
    def forward(self, x):
        x = self.pool(self.relu(self.conv1(x)))
        x = self.channel_dropout1(x)
        x = self.pool(self.relu(self.conv2(x)))
        x = self.channel_dropout2(x)
        x = x.view(-1, 128 * 8 * 8)
        x = self.relu(self.fc1(x))
        x = self.filter_dropout(x)
        x = self.fc2(x)
        return x
```

## 27. How do you implement regularization for specific architectures?

**Answer:**
```python
# For RNNs/LSTMs
class RegularizedRNN(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers, output_size, dropout_rate=0.5):
        super(RegularizedRNN, self).__init__()
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, 
                           dropout=dropout_rate if num_layers > 1 else 0,
                           batch_first=True)
        self.dropout = nn.Dropout(dropout_rate)
        self.fc = nn.Linear(hidden_size, output_size)
    
    def forward(self, x):
        # LSTM forward pass
        lstm_out, (hidden, cell) = self.lstm(x)
        
        # Apply dropout to output
        lstm_out = self.dropout(lstm_out)
        
        # Use last time step for classification
        output = self.fc(lstm_out[:, -1, :])
        return output

# For Transformers
class RegularizedTransformer(nn.Module):
    def __init__(self, d_model=512, nhead=8, num_layers=6, dropout_rate=0.1):
        super(RegularizedTransformer, self).__init__()
        
        # Embedding with dropout
        self.embedding_dropout = nn.Dropout(dropout_rate)
        
        # Transformer layers with attention dropout
        encoder_layer = nn.TransformerEncoderLayer(
            d_model=d_model, 
            nhead=nhead,
            dropout=dropout_rate,
            batch_first=True
        )
        self.transformer = nn.TransformerEncoder(encoder_layer, num_layers)
        
        # Output dropout
        self.output_dropout = nn.Dropout(dropout_rate)
        self.fc = nn.Linear(d_model, 10)
    
    def forward(self, x, src_mask=None):
        # Embedding with dropout
        x = self.embedding_dropout(x)
        
        # Transformer forward pass
        x = self.transformer(x, src_mask)
        
        # Apply dropout to output
        x = self.output_dropout(x)
        
        # Use first token for classification
        x = self.fc(x[:, 0, :])
        return x

# For GANs
class RegularizedGANDiscriminator(nn.Module):
    def __init__(self, input_size=784, hidden_sizes=[256, 128], dropout_rate=0.3):
        super(RegularizedGANDiscriminator, self).__init__()
        
        layers = []
        prev_size = input_size
        
        for hidden_size in hidden_sizes:
            layers.append(nn.Linear(prev_size, hidden_size))
            layers.append(nn.LeakyReLU(0.2))
            layers.append(nn.Dropout(dropout_rate))  # Dropout for GAN regularization
            layers.append(nn.BatchNorm1d(hidden_size))  # Batch norm for GAN stability
            prev_size = hidden_size
        
        layers.append(nn.Linear(prev_size, 1))
        layers.append(nn.Sigmoid())
        
        self.network = nn.Sequential(*layers)
    
    def forward(self, x):
        return self.network(x)

# Spectral normalization for GANs
class SpectralNormDiscriminator(nn.Module):
    def __init__(self, input_size=784, hidden_sizes=[256, 128]):
        super(SpectralNormDiscriminator, self).__init__()
        
        layers = []
        prev_size = input_size
        
        for i, hidden_size in enumerate(hidden_sizes):
            # Apply spectral normalization to linear layers
            linear = nn.Linear(prev_size, hidden_size)
            layers.append(nn.utils.spectral_norm(linear))
            layers.append(nn.LeakyReLU(0.2))
            prev_size = hidden_size
        
        # Output layer with spectral norm
        output_linear = nn.Linear(prev_size, 1)
        layers.append(nn.utils.spectral_norm(output_linear))
        layers.append(nn.Sigmoid())
        
        self.network = nn.Sequential(*layers)
    
    def forward(self, x):
        return self.network(x)
```

## 28. How do you implement adaptive regularization?

**Answer:**
```python
class AdaptiveRegularization:
    """Adaptive regularization that adjusts based on training dynamics"""
    def __init__(self, initial_lambda=0.01, adaptation_rate=0.1):
        self.initial_lambda = initial_lambda
        self.current_lambda = initial_lambda
        self.adaptation_rate = adaptation_rate
        self.loss_history = []
        self.grad_norm_history = []
    
    def compute_gradient_norm(self, model):
        """Compute the norm of gradients"""
        total_norm = 0
        for param in model.parameters():
            if param.grad is not None:
                param_norm = param.grad.data.norm(2)
                total_norm += param_norm.item() ** 2
        total_norm = total_norm ** (1. / 2)
        return total_norm
    
    def update_lambda(self, model, current_loss):
        """Adaptively update regularization strength"""
        self.loss_history.append(current_loss)
        grad_norm = self.compute_gradient_norm(model)
        self.grad_norm_history.append(grad_norm)
        
        if len(self.loss_history) < 10:
            return  # Need more history for meaningful adaptation
        
        # Compute recent trends
        recent_losses = self.loss_history[-10:]
        recent_grads = self.grad_norm_history[-10:]
        
        loss_trend = (recent_losses[-1] - recent_losses[0]) / len(recent_losses)
        grad_trend = (recent_grads[-1] - recent_grads[0]) / len(recent_grads)
        
        # Adapt regularization based on training dynamics
        if loss_trend > 0:  # Loss is increasing (overfitting)
            self.current_lambda *= (1 + self.adaptation_rate)
        elif abs(loss_trend) < 0.001 and grad_trend < 0.01:  # Plateau
            self.current_lambda *= (1 - self.adaptation_rate * 0.5)
        else:  # Normal training
            pass  # Keep current lambda
        
        # Keep lambda within reasonable bounds
        self.current_lambda = max(0.001, min(self.current_lambda, 0.1))
    
    def get_regularization_loss(self, model):
        """Compute regularization loss with current lambda"""
        l2_reg = torch.tensor(0., device=next(model.parameters()).device)
        for param in model.parameters():
            l2_reg += torch.norm(param, 2)
        return self.current_lambda * l2_reg

# Usage in training loop
def train_with_adaptive_regularization(model, dataloader, optimizer, criterion):
    adaptive_reg = AdaptiveRegularization(initial_lambda=0.01, adaptation_rate=0.1)
    
    for epoch in range(100):
        model.train()
        epoch_loss = 0.0
        
        for inputs, targets in dataloader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            
            # Add adaptive regularization
            reg_loss = adaptive_reg.get_regularization_loss(model)
            total_loss = loss + reg_loss
            
            total_loss.backward()
            optimizer.step()
            
            epoch_loss += total_loss.item()
        
        avg_loss = epoch_loss / len(dataloader)
        
        # Update regularization strength
        adaptive_reg.update_lambda(model, avg_loss)
        
        print(f"Epoch {epoch+1}: Loss = {avg_loss:.4f}, "
              f"Lambda = {adaptive_reg.current_lambda:.6f}")
```

## 29. How do you implement regularization for transfer learning?

**Answer:**
```python
class TransferLearningRegularization:
    """Regularization techniques specifically for transfer learning"""
    def __init__(self, model, pretrained_layers, fine_tune_layers):
        self.model = model
        self.pretrained_layers = pretrained_layers
        self.fine_tune_layers = fine_tune_layers
    
    def setup_differential_learning_rates(self, base_lr=1e-4):
        """Setup different learning rates for pretrained vs new layers"""
        param_groups = []
        
        # Lower learning rate for pretrained layers
        pretrained_params = []
        for name, param in self.model.named_parameters():
            if any(layer_name in name for layer_name in self.pretrained_layers):
                pretrained_params.append(param)
        
        if pretrained_params:
            param_groups.append({
                'params': pretrained_params,
                'lr': base_lr * 0.1  # 10x lower learning rate
            })
        
        # Higher learning rate for new/fine-tune layers
        finetune_params = []
        for name, param in self.model.named_parameters():
            if any(layer_name in name for layer_name in self.fine_tune_layers):
                finetune_params.append(param)
        
        if finetune_params:
            param_groups.append({
                'params': finetune_params,
                'lr': base_lr
            })
        
        return param_groups
    
    def setup_layer_freezing(self, freeze_pretrained=True):
        """Freeze pretrained layers during initial training"""
        for name, param in self.model.named_parameters():
            if any(layer_name in name for layer_name in self.pretrained_layers):
                param.requires_grad = not freeze_pretrained
    
    def setup_progressive_unfreezing(self, epoch, unfreeze_schedule=None):
        """Progressively unfreeze layers during training"""
        if unfreeze_schedule is None:
            unfreeze_schedule = {10: ['layer4'], 20: ['layer3'], 30: ['layer2']}
        
        layers_to_unfreeze = unfreeze_schedule.get(epoch, [])
        
        for name, param in self.model.named_parameters():
            if any(layer_name in name for layer_name in layers_to_unfreeze):
                param.requires_grad = True
                print(f"Unfrozen layer: {name}")

# Example: Transfer learning with ResNet
class TransferLearningModel(nn.Module):
    def __init__(self, num_classes=10, pretrained=True):
        super(TransferLearningModel, self).__init__()
        # Load pretrained ResNet
        self.backbone = models.resnet18(pretrained=pretrained)
        
        # Replace classifier
        self.backbone.fc = nn.Linear(self.backbone.fc.in_features, num_classes)
        
        # Identify layer names for regularization
        self.pretrained_layers = [name for name, _ in self.backbone.named_parameters() 
                                 if 'fc' not in name]
        self.fine_tune_layers = ['fc.weight', 'fc.bias']
    
    def forward(self, x):
        return self.backbone(x)

# Usage
def train_transfer_learning_model(model, train_loader, val_loader, 
                                 num_epochs=50, base_lr=1e-4):
    # Setup transfer learning regularization
    transfer_reg = TransferLearningRegularization(
        model, 
        model.pretrained_layers, 
        model.fine_tune_layers
    )
    
    # Freeze pretrained layers initially
    transfer_reg.setup_layer_freezing(freeze_pretrained=True)
    
    # Setup differential learning rates
    param_groups = transfer_reg.setup_differential_learning_rates(base_lr)
    optimizer = optim.Adam(param_groups, weight_decay=0.01)
    
    # Setup schedulers
    scheduler = optim.lr_scheduler.StepLR(optimizer, step_size=20, gamma=0.1)
    
    criterion = nn.CrossEntropyLoss()
    
    for epoch in range(num_epochs):
        # Progressive unfreezing
        if epoch == 10:
            transfer_reg.setup_layer_freezing(freeze_pretrained=False)
            print("Unfreezing all layers")
        
        # Training phase
        model.train()
        train_loss = 0.0
        
        for inputs, targets in train_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            
            # Add L2 regularization only to fine-tune layers
            l2_reg = torch.tensor(0., device=loss.device)
            for name, param in model.named_parameters():
                if 'fc' in name:  # Only regularize new layers
                    l2_reg += torch.norm(param, 2)
            loss += 0.01 * l2_reg
            
            loss.backward()
            optimizer.step()
            train_loss += loss.item()
        
        # Validation phase
        model.eval()
        val_loss = 0.0
        correct = 0
        total = 0
        
        with torch.no_grad():
            for inputs, targets in val_loader:
                inputs, targets = inputs.to(device), targets.to(device)
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                val_loss += loss.item()
                
                _, predicted = torch.max(outputs, 1)
                total += targets.size(0)
                correct += (predicted == targets).sum().item()
        
        scheduler.step()
        
        print(f'Epoch {epoch+1}/{num_epochs}:')
        print(f'  Train Loss: {train_loss/len(train_loader):.4f}')
        print(f'  Val Loss: {val_loss/len(val_loader):.4f}')
        print(f'  Val Acc: {100.*correct/total:.2f}%')
```

## 30. How do you implement multi-task regularization?

**Answer:**
```python
class MultiTaskRegularization:
    """Regularization for multi-task learning"""
    def __init__(self, num_tasks=2, reg_type='l2'):
        self.num_tasks = num_tasks
        self.reg_type = reg_type
    
    def compute_task_similarity_regularization(self, task_representations):
        """Regularize based on similarity between task representations"""
        if len(task_representations) < 2:
            return torch.tensor(0.0, device=task_representations[0].device)
        
        # Compute pairwise similarities
        similarities = []
        for i in range(len(task_representations)):
            for j in range(i + 1, len(task_representations)):
                # Cosine similarity
                cos_sim = F.cosine_similarity(
                    task_representations[i], 
                    task_representations[j], 
                    dim=1
                )
                similarities.append(torch.mean(cos_sim))
        
        # Encourage diversity (minimize similarity)
        if self.reg_type == 'diversity':
            return -torch.mean(torch.stack(similarities))
        # Encourage similarity (maximize similarity)
        elif self.reg_type == 'similarity':
            return torch.mean(torch.stack(similarities))
        else:
            return torch.tensor(0.0, device=task_representations[0].device)
    
    def compute_shared_representation_regularization(self, shared_features, task_features):
        """Regularize the relationship between shared and task-specific features"""
        # Encourage task features to be complementary to shared features
        reg_loss = 0.0
        for task_feat in task_features:
            # Orthogonality regularization
            shared_norm = torch.norm(shared_features, dim=1, keepdim=True)
            task_norm = torch.norm(task_feat, dim=1, keepdim=True)
            dot_product = torch.sum(shared_features * task_feat, dim=1, keepdim=True)
            cosine_sim = dot_product / (shared_norm * task_norm + 1e-8)
            reg_loss += torch.mean(cosine_sim ** 2)
        
        return reg_loss

class MultiTaskNetwork(nn.Module):
    def __init__(self, input_size=784, shared_hidden=[256, 128], 
                 task_hidden=[64], num_tasks=2, output_sizes=[10, 5]):
        super(MultiTaskNetwork, self).__init__()
        
        self.num_tasks = num_tasks
        
        # Shared layers
        shared_layers = []
        prev_size = input_size
        for hidden_size in shared_hidden:
            shared_layers.append(nn.Linear(prev_size, hidden_size))
            shared_layers.append(nn.ReLU())
            prev_size = hidden_size
        self.shared_layers = nn.Sequential(*shared_layers)
        
        # Task-specific layers
        self.task_layers = nn.ModuleList()
        self.task_output_layers = nn.ModuleList()
        
        for i in range(num_tasks):
            # Task-specific layers
            task_layers = []
            for hidden_size in task_hidden:
                task_layers.append(nn.Linear(prev_size, hidden_size))
                task_layers.append(nn.ReLU())
            self.task_layers.append(nn.Sequential(*task_layers))
            
            # Output layer
            output_layer = nn.Linear(task_hidden[-1], output_sizes[i])
            self.task_output_layers.append(output_layer)
    
    def forward(self, x):
        # Shared representation
        shared_features = self.shared_layers(x)
        
        # Task-specific representations and outputs
        task_features = []
        task_outputs = []
        
        for i in range(self.num_tasks):
            task_feat = self.task_layers[i](shared_features)
            task_features.append(task_feat)
            task_output = self.task_output_layers[i](task_feat)
            task_outputs.append(task_output)
        
        return task_outputs, shared_features, task_features

# Training with multi-task regularization
def train_multitask_model(model, dataloader, optimizer, criterions, 
                         task_weights=[1.0, 1.0], reg_weight=0.01):
    multitask_reg = MultiTaskRegularization(num_tasks=2, reg_type='diversity')
    
    model.train()
    total_loss = 0.0
    
    for batch in dataloader:
        inputs = batch['inputs'].to(device)
        targets_task1 = batch['targets_task1'].to(device)
        targets_task2 = batch['targets_task2'].to(device)
        
        optimizer.zero_grad()
        
        # Forward pass
        task_outputs, shared_features, task_features = model(inputs)
        
        # Compute task losses
        loss_task1 = criterions[0](task_outputs[0], targets_task1)
        loss_task2 = criterions[1](task_outputs[1], targets_task2)
        
        # Weighted combination
        total_task_loss = task_weights[0] * loss_task1 + task_weights[1] * loss_task2
        
        # Add regularization
        reg_loss = multitask_reg.compute_task_similarity_regularization(task_features)
        reg_loss += multitask_reg.compute_shared_representation_regularization(
            shared_features, task_features
        )
        
        # Final loss
        final_loss = total_task_loss + reg_weight * reg_loss
        
        final_loss.backward()
        optimizer.step()
        
        total_loss += final_loss.item()
    
    return total_loss / len(dataloader)
```

## 31. How do you evaluate the effectiveness of different regularization techniques?

**Answer:**
```python
class RegularizationEvaluator:
    """Evaluate the effectiveness of different regularization techniques"""
    def __init__(self, model, train_loader, val_loader, test_loader):
        self.model = model
        self.train_loader = train_loader
        self.val_loader = val_loader
        self.test_loader = test_loader
        self.device = next(model.parameters()).device
    
    def evaluate_overfitting_metrics(self, model, train_loader, val_loader, epochs=50):
        """Evaluate overfitting behavior"""
        train_losses = []
        val_losses = []
        train_accuracies = []
        val_accuracies = []
        
        # Setup optimizer and criterion
        optimizer = optim.Adam(model.parameters(), lr=0.001)
        criterion = nn.CrossEntropyLoss()
        
        for epoch in range(epochs):
            # Training
            model.train()
            train_loss = 0.0
            train_correct = 0
            train_total = 0
            
            for inputs, targets in train_loader:
                inputs, targets = inputs.to(self.device), targets.to(self.device)
                
                optimizer.zero_grad()
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                loss.backward()
                optimizer.step()
                
                train_loss += loss.item()
                _, predicted = torch.max(outputs.data, 1)
                train_total += targets.size(0)
                train_correct += (predicted == targets).sum().item()
            
            # Validation
            model.eval()
            val_loss = 0.0
            val_correct = 0
            val_total = 0
            
            with torch.no_grad():
                for inputs, targets in val_loader:
                    inputs, targets = inputs.to(self.device), targets.to(self.device)
                    outputs = model(inputs)
                    loss = criterion(outputs, targets)
                    val_loss += loss.item()
                    _, predicted = torch.max(outputs.data, 1)
                    val_total += targets.size(0)
                    val_correct += (predicted == targets).sum().item()
            
            train_losses.append(train_loss / len(train_loader))
            val_losses.append(val_loss / len(val_loader))
            train_accuracies.append(100 * train_correct / train_total)
            val_accuracies.append(100 * val_correct / val_total)
        
        # Calculate overfitting metrics
        overfitting_gap = max(val_accuracies) - val_accuracies[-1]  # Performance drop
        final_gap = train_accuracies[-1] - val_accuracies[-1]  # Final train-val gap
        
        return {
            'train_losses': train_losses,
            'val_losses': val_losses,
            'train_accuracies': train_accuracies,
            'val_accuracies': val_accuracies,
            'overfitting_gap': overfitting_gap,
            'final_gap': final_gap
        }
    
    def compare_regularization_techniques(self, regularization_configs):
        """Compare different regularization techniques"""
        results = {}
        
        for reg_name, reg_config in regularization_configs.items():
            print(f"Evaluating {reg_name}...")
            
            # Create fresh model
            model = self.model.__class__()  # Assuming same architecture
            model = model.to(self.device)
            
            # Apply regularization configuration
            self.apply_regularization(model, reg_config)
            
            # Evaluate overfitting behavior
            metrics = self.evaluate_overfitting_metrics(
                model, self.train_loader, self.val_loader, epochs=30
            )
            
            results[reg_name] = metrics
            
            # Test set performance
            test_accuracy = self.evaluate_test_accuracy(model, self.test_loader)
            results[reg_name]['test_accuracy'] = test_accuracy
            
            print(f"{reg_name}: Test Accuracy = {test_accuracy:.2f}%, "
                  f"Overfitting Gap = {metrics['overfitting_gap']:.2f}%, "
                  f"Final Gap = {metrics['final_gap']:.2f}%")
        
        return results
    
    def apply_regularization(self, model, reg_config):
        """Apply regularization configuration to model"""
        # This would implement the specific regularization based on config
        # For example, setting dropout rates, weight decay, etc.
        pass
    
    def evaluate_test_accuracy(self, model, test_loader):
        """Evaluate model on test set"""
        model.eval()
        correct = 0
        total = 0
        
        with torch.no_grad():
            for inputs, targets in test_loader:
                inputs, targets = inputs.to(self.device), targets.to(self.device)
                outputs = model(inputs)
                _, predicted = torch.max(outputs.data, 1)
                total += targets.size(0)
                correct += (predicted == targets).sum().item()
        
        return 100 * correct / total
    
    def plot_regularization_comparison(self, results):
        """Plot comparison of regularization techniques"""
        import matplotlib.pyplot as plt
        
        fig, axes = plt.subplots(2, 2, figsize=(15, 10))
        
        for reg_name, metrics in results.items():
            epochs = range(len(metrics['train_losses']))
            
            # Training vs Validation Loss
            axes[0, 0].plot(epochs, metrics['train_losses'], label=f"{reg_name} (Train)")
            axes[0, 0].plot(epochs, metrics['val_losses'], label=f"{reg_name} (Val)")
            axes[0, 0].set_title('Training vs Validation Loss')
            axes[0, 0].set_xlabel('Epoch')
            axes[0, 0].set_ylabel('Loss')
            axes[0, 0].legend()
            
            # Training vs Validation Accuracy
            axes[0, 1].plot(epochs, metrics['train_accuracies'], label=f"{reg_name} (Train)")
            axes[0, 1].plot(epochs, metrics['val_accuracies'], label=f"{reg_name} (Val)")
            axes[0, 1].set_title('Training vs Validation Accuracy')
            axes[0, 1].set_xlabel('Epoch')
            axes[0, 1].set_ylabel('Accuracy (%)')
            axes[0, 1].legend()
        
        # Overfitting comparison
        reg_names = list(results.keys())
        overfitting_gaps = [results[name]['overfitting_gap'] for name in reg_names]
        final_gaps = [results[name]['final_gap'] for name in reg_names]
        test_accuracies = [results[name]['test_accuracy'] for name in reg_names]
        
        axes[1, 0].bar(reg_names, overfitting_gaps)
        axes[1, 0].set_title('Overfitting Gap (Max Val Acc - Final Val Acc)')
        axes[1, 0].set_ylabel('Accuracy Drop (%)')
        axes[1, 0].tick_params(axis='x', rotation=45)
        
        axes[1, 1].bar(reg_names, final_gaps)
        axes[1, 1].set_title('Final Train-Val Gap')
        axes[1, 1].set_ylabel('Accuracy Gap (%)')
        axes[1, 1].tick_params(axis='x', rotation=45)
        
        plt.tight_layout()
        plt.show()
        
        # Print summary
        print("\nRegularization Comparison Summary:")
        print("-" * 50)
        for reg_name in reg_names:
            print(f"{reg_name:20} | Test Acc: {test_accuracies[reg_names.index(reg_name)]:6.2f}% | "
                  f"Overfit: {overfitting_gaps[reg_names.index(reg_name)]:6.2f}% | "
                  f"Gap: {final_gaps[reg_names.index(reg_name)]:6.2f}%")

# Usage example
def run_regularization_evaluation():
    # Define regularization configurations to compare
    reg_configs = {
        'No Regularization': {},
        'L2 Regularization': {'weight_decay': 0.01},
        'Dropout': {'dropout_rate': 0.5},
        'BatchNorm': {'batch_norm': True},
        'Combined': {'weight_decay': 0.01, 'dropout_rate': 0.5, 'batch_norm': True}
    }
    
    # Initialize evaluator
    evaluator = RegularizationEvaluator(model, train_loader, val_loader, test_loader)
    
    # Compare regularization techniques
    results = evaluator.compare_regularization_techniques(reg_configs)
    
    # Plot and analyze results
    evaluator.plot_regularization_comparison(results)
    
    return results
```