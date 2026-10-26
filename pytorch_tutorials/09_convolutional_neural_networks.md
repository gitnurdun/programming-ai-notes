# Convolutional Neural Networks (CNNs) in PyTorch - 30+ Questions and Answers

## 1. What is a Convolutional Neural Network (CNN) and how does it work?

**Answer:** A Convolutional Neural Network (CNN) is a deep learning architecture specifically designed for processing grid-like data such as images. CNNs use convolutional layers that apply learnable filters (kernels) to input data to detect local patterns like edges, textures, and shapes. The key components include:
- **Convolutional Layers:** Apply filters to extract features
- **Pooling Layers:** Reduce spatial dimensions and computational load
- **Activation Functions:** Introduce non-linearity (typically ReLU)
- **Fully Connected Layers:** Make final predictions
CNNs work by learning hierarchical feature representations, starting with simple patterns in early layers and complex patterns in deeper layers.

## 2. How do you implement a basic CNN in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BasicCNN(nn.Module):
    def __init__(self, num_classes=10):
        super(BasicCNN, self).__init__()
        # Convolutional layers
        self.conv1 = nn.Conv2d(in_channels=3, out_channels=32, kernel_size=3, padding=1)
        self.conv2 = nn.Conv2d(in_channels=32, out_channels=64, kernel_size=3, padding=1)
        self.conv3 = nn.Conv2d(in_channels=64, out_channels=128, kernel_size=3, padding=1)
        
        # Pooling layer
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)
        
        # Fully connected layers
        self.fc1 = nn.Linear(128 * 4 * 4, 512)  # Assuming input is 32x32
        self.fc2 = nn.Linear(512, num_classes)
        
        # Activation function
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x):
        # First convolutional block
        x = self.pool(self.relu(self.conv1(x)))  # Output: (batch, 32, 16, 16)
        
        # Second convolutional block
        x = self.pool(self.relu(self.conv2(x)))  # Output: (batch, 64, 8, 8)
        
        # Third convolutional block
        x = self.pool(self.relu(self.conv3(x)))  # Output: (batch, 128, 4, 4)
        
        # Flatten for fully connected layers
        x = x.view(-1, 128 * 4 * 4)  # Flatten to (batch, 128*4*4)
        
        # Fully connected layers
        x = self.relu(self.fc1(x))
        x = self.dropout(x)
        x = self.fc2(x)
        
        return x

# Usage
model = BasicCNN(num_classes=10)
print(model)

# Test with sample input
sample_input = torch.randn(1, 3, 32, 32)  # Batch of 1, 3 channels, 32x32 image
output = model(sample_input)
print(f"Output shape: {output.shape}")  # Should be (1, 10)
```

## 3. What are the different types of convolution operations in PyTorch?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# 1. Standard 2D Convolution
conv2d = nn.Conv2d(in_channels=3, out_channels=64, kernel_size=3, padding=1)

# 2. 1D Convolution (for sequences/text)
conv1d = nn.Conv1d(in_channels=128, out_channels=256, kernel_size=3, padding=1)

# 3. 3D Convolution (for volumetric data)
conv3d = nn.Conv3d(in_channels=1, out_channels=32, kernel_size=3, padding=1)

# 4. Depthwise Convolution (separable convolution)
class DepthwiseConv2d(nn.Module):
    def __init__(self, in_channels, out_channels, kernel_size, padding=0):
        super(DepthwiseConv2d, self).__init__()
        # Depthwise convolution
        self.depthwise = nn.Conv2d(in_channels, in_channels, kernel_size, 
                                  padding=padding, groups=in_channels)
        # Pointwise convolution
        self.pointwise = nn.Conv2d(in_channels, out_channels, 1)
    
    def forward(self, x):
        x = self.depthwise(x)
        x = self.pointwise(x)
        return x

# 5. Transposed Convolution (for upsampling)
conv_transpose = nn.ConvTranspose2d(in_channels=64, out_channels=32, 
                                   kernel_size=3, stride=2, padding=1, output_padding=1)

# 6. Dilated Convolution (atrous convolution)
dilated_conv = nn.Conv2d(in_channels=64, out_channels=128, kernel_size=3, 
                        padding=2, dilation=2)

# 7. Grouped Convolution
grouped_conv = nn.Conv2d(in_channels=64, out_channels=128, kernel_size=3, 
                        padding=1, groups=4)

# Example usage
x = torch.randn(1, 3, 32, 32)

# Standard convolution
out1 = conv2d(x)
print(f"Standard Conv2D output shape: {out1.shape}")

# Depthwise separable convolution
depthwise_conv = DepthwiseConv2d(3, 64, 3, padding=1)
out2 = depthwise_conv(x)
print(f"Depthwise Conv output shape: {out2.shape}")

# Transposed convolution
out3 = conv_transpose(out1)
print(f"Transposed Conv output shape: {out3.shape}")
```

## 4. What is the difference between padding='valid' and padding='same' in convolutions?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# In PyTorch, padding is specified as an integer or tuple
# 'valid' padding means no padding (padding=0)
conv_valid = nn.Conv2d(3, 64, kernel_size=3, padding=0)

# 'same' padding means padding to maintain same spatial dimensions
# For odd kernel sizes, padding = (kernel_size - 1) // 2
conv_same = nn.Conv2d(3, 64, kernel_size=3, padding=1)  # Same as padding='same' for kernel_size=3

# Manual calculation for 'same' padding
def calculate_same_padding(kernel_size, stride=1):
    """Calculate padding for 'same' convolution"""
    return (kernel_size - 1) // 2

# Example with different kernel sizes
kernel_sizes = [1, 3, 5, 7]
for k in kernel_sizes:
    padding = calculate_same_padding(k)
    print(f"Kernel size {k}: padding = {padding}")

# Functional approach for 'same' padding
def conv2d_same_padding(x, weight, bias=None, stride=1, dilation=1):
    """Conv2d with same padding using functional API"""
    kernel_size = weight.shape[2]
    padding = calculate_same_padding(kernel_size)
    return F.conv2d(x, weight, bias, stride, padding, dilation)

# Test with sample data
x = torch.randn(1, 3, 32, 32)
conv_layer = nn.Conv2d(3, 64, kernel_size=3)

# Valid padding (no padding)
out_valid = F.conv2d(x, conv_layer.weight, conv_layer.bias, padding=0)
print(f"Valid padding output shape: {out_valid.shape}")  # (1, 64, 30, 30)

# Same padding
out_same = F.conv2d(x, conv_layer.weight, conv_layer.bias, padding=1)
print(f"Same padding output shape: {out_same.shape}")   # (1, 64, 32, 32)
```

## 5. How do you implement different pooling operations in CNNs?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# 1. Max Pooling
max_pool = nn.MaxPool2d(kernel_size=2, stride=2)
max_pool_func = F.max_pool2d

# 2. Average Pooling
avg_pool = nn.AvgPool2d(kernel_size=2, stride=2)
avg_pool_func = F.avg_pool2d

# 3. Adaptive Pooling (output size is fixed regardless of input size)
adaptive_max_pool = nn.AdaptiveMaxPool2d((4, 4))  # Output will be (batch, channels, 4, 4)
adaptive_avg_pool = nn.AdaptiveAvgPool2d((1, 1))  # Global average pooling

# 4. Fractional Max Pooling
frac_max_pool = nn.FractionalMaxPool2d(kernel_size=2, output_ratio=(0.5, 0.5))

# 5. LPPool (Lp norm pooling)
lp_pool = nn.LPPool2d(norm_type=2, kernel_size=2, stride=2)

# Example usage
x = torch.randn(2, 64, 16, 16)  # Batch=2, Channels=64, Height=16, Width=16

# Different pooling operations
max_out = max_pool(x)
avg_out = avg_pool(x)
adaptive_max_out = adaptive_max_pool(x)
adaptive_avg_out = adaptive_avg_pool(x)  # Global average pooling

print(f"Input shape: {x.shape}")
print(f"MaxPool2D output: {max_out.shape}")
print(f"AvgPool2D output: {avg_out.shape}")
print(f"AdaptiveMaxPool2D output: {adaptive_max_out.shape}")
print(f"AdaptiveAvgPool2D output: {adaptive_avg_out.shape}")

# Global Average Pooling for fully connected layer replacement
class CNNWithGlobalAvgPool(nn.Module):
    def __init__(self, num_classes=10):
        super(CNNWithGlobalAvgPool, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.conv3 = nn.Conv2d(128, 256, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.relu = nn.ReLU()
        
        # Global average pooling instead of flattening
        self.global_avg_pool = nn.AdaptiveAvgPool2d((1, 1))
        self.fc = nn.Linear(256, num_classes)
    
    def forward(self, x):
        x = self.pool(self.relu(self.conv1(x)))
        x = self.pool(self.relu(self.conv2(x)))
        x = self.pool(self.relu(self.conv3(x)))
        
        # Global average pooling
        x = self.global_avg_pool(x)  # (batch, 256, 1, 1)
        x = x.view(x.size(0), -1)    # (batch, 256)
        x = self.fc(x)
        
        return x
```

## 6. How do you implement batch normalization in CNNs?

**Answer:**
```python
import torch
import torch.nn as nn

class CNNWithBatchNorm(nn.Module):
    def __init__(self, num_classes=10):
        super(CNNWithBatchNorm, self).__init__()
        
        # Convolutional layers with batch normalization
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.bn1 = nn.BatchNorm2d(64)
        
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.bn2 = nn.BatchNorm2d(128)
        
        self.conv3 = nn.Conv2d(128, 256, 3, padding=1)
        self.bn3 = nn.BatchNorm2d(256)
        
        # Pooling
        self.pool = nn.MaxPool2d(2, 2)
        self.relu = nn.ReLU()
        
        # Fully connected layers
        self.fc1 = nn.Linear(256 * 4 * 4, 512)
        self.bn_fc1 = nn.BatchNorm1d(512)
        self.fc2 = nn.Linear(512, num_classes)
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x):
        # First conv block
        x = self.conv1(x)
        x = self.bn1(x)  # Batch norm before activation
        x = self.relu(x)
        x = self.pool(x)
        
        # Second conv block
        x = self.conv2(x)
        x = self.bn2(x)
        x = self.relu(x)
        x = self.pool(x)
        
        # Third conv block
        x = self.conv3(x)
        x = self.bn3(x)
        x = self.relu(x)
        x = self.pool(x)
        
        # Flatten and fully connected
        x = x.view(-1, 256 * 4 * 4)
        x = self.fc1(x)
        x = self.bn_fc1(x)
        x = self.relu(x)
        x = self.dropout(x)
        x = self.fc2(x)
        
        return x

# Alternative: Using nn.Sequential with batch normalization
class SequentialCNNWithBatchNorm(nn.Module):
    def __init__(self, num_classes=10):
        super(SequentialCNNWithBatchNorm, self).__init__()
        
        self.features = nn.Sequential(
            # First conv block
            nn.Conv2d(3, 64, 3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2, 2),
            
            # Second conv block
            nn.Conv2d(64, 128, 3, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.MaxPool2d(2, 2),
            
            # Third conv block
            nn.Conv2d(128, 256, 3, padding=1),
            nn.BatchNorm2d(256),
            nn.ReLU(),
            nn.MaxPool2d(2, 2),
        )
        
        self.classifier = nn.Sequential(
            nn.AdaptiveAvgPool2d((1, 1)),
            nn.Flatten(),
            nn.Linear(256, 512),
            nn.BatchNorm1d(512),
            nn.ReLU(),
            nn.Dropout(0.5),
            nn.Linear(512, num_classes)
        )
    
    def forward(self, x):
        x = self.features(x)
        x = self.classifier(x)
        return x

# Important: Remember to set train/eval modes
model = CNNWithBatchNorm()
model.train()  # Uses batch statistics
# or
model.eval()   # Uses running statistics
```

## 7. How do you implement residual connections in CNNs?

**Answer:**
```python
import torch
import torch.nn as nn

class ResidualBlock(nn.Module):
    def __init__(self, in_channels, out_channels, stride=1):
        super(ResidualBlock, self).__init__()
        
        # Main path
        self.conv1 = nn.Conv2d(in_channels, out_channels, 3, stride, padding=1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        
        # Shortcut connection
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 1, stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )
    
    def forward(self, x):
        # Main path
        residual = x
        out = self.conv1(x)
        out = self.bn1(out)
        out = self.relu(out)
        out = self.conv2(out)
        out = self.bn2(out)
        
        # Shortcut connection
        residual = self.shortcut(x)
        
        # Element-wise addition
        out += residual
        out = self.relu(out)
        
        return out

class ResNet(nn.Module):
    def __init__(self, num_classes=10, num_blocks=[2, 2, 2, 2]):
        super(ResNet, self).__init__()
        
        self.in_channels = 64
        
        # Initial convolution
        self.conv1 = nn.Conv2d(3, 64, 7, stride=2, padding=3, bias=False)
        self.bn1 = nn.BatchNorm2d(64)
        self.relu = nn.ReLU(inplace=True)
        self.maxpool = nn.MaxPool2d(3, stride=2, padding=1)
        
        # Residual blocks
        self.layer1 = self._make_layer(64, num_blocks[0], stride=1)
        self.layer2 = self._make_layer(128, num_blocks[1], stride=2)
        self.layer3 = self._make_layer(256, num_blocks[2], stride=2)
        self.layer4 = self._make_layer(512, num_blocks[3], stride=2)
        
        # Final layers
        self.avgpool = nn.AdaptiveAvgPool2d((1, 1))
        self.fc = nn.Linear(512, num_classes)
    
    def _make_layer(self, out_channels, num_blocks, stride):
        strides = [stride] + [1] * (num_blocks - 1)
        layers = []
        for stride in strides:
            layers.append(ResidualBlock(self.in_channels, out_channels, stride))
            self.in_channels = out_channels
        return nn.Sequential(*layers)
    
    def forward(self, x):
        # Initial layers
        x = self.conv1(x)
        x = self.bn1(x)
        x = self.relu(x)
        x = self.maxpool(x)
        
        # Residual blocks
        x = self.layer1(x)
        x = self.layer2(x)
        x = self.layer3(x)
        x = self.layer4(x)
        
        # Final layers
        x = self.avgpool(x)
        x = torch.flatten(x, 1)
        x = self.fc(x)
        
        return x

# Simplified residual block for smaller networks
class SimpleResidualBlock(nn.Module):
    def __init__(self, channels):
        super(SimpleResidualBlock, self).__init__()
        self.conv1 = nn.Conv2d(channels, channels, 3, padding=1)
        self.bn1 = nn.BatchNorm2d(channels)
        self.conv2 = nn.Conv2d(channels, channels, 3, padding=1)
        self.bn2 = nn.BatchNorm2d(channels)
        self.relu = nn.ReLU(inplace=True)
    
    def forward(self, x):
        residual = x
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += residual  # Residual connection
        out = self.relu(out)
        return out

# Usage
resnet = ResNet(num_classes=10)
sample_input = torch.randn(1, 3, 224, 224)
output = resnet(sample_input)
print(f"ResNet output shape: {output.shape}")
```

## 8. How do you implement different activation functions in CNNs?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class CNNWithDifferentActivations(nn.Module):
    def __init__(self, num_classes=10):
        super(CNNWithDifferentActivations, self).__init__()
        
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.conv3 = nn.Conv2d(128, 256, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        
        # Different activation functions
        self.relu = nn.ReLU()
        self.leaky_relu = nn.LeakyReLU(0.01)
        self.elu = nn.ELU()
        self.selu = nn.SELU()
        self.gelu = nn.GELU()
        self.swish = nn.SiLU()  # Swish activation
        
        self.fc1 = nn.Linear(256 * 4 * 4, 512)
        self.fc2 = nn.Linear(512, num_classes)
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x):
        # Using different activations in different layers
        x = self.pool(self.relu(self.conv1(x)))      # ReLU
        x = self.pool(self.leaky_relu(self.conv2(x))) # LeakyReLU
        x = self.pool(self.elu(self.conv3(x)))       # ELU
        
        x = x.view(-1, 256 * 4 * 4)
        x = self.swish(self.fc1(x))                  # Swish/SiLU
        x = self.dropout(x)
        x = self.fc2(x)
        
        return x

# Custom activation functions
class Swish(nn.Module):
    def forward(self, x):
        return x * torch.sigmoid(x)

class Mish(nn.Module):
    def forward(self, x):
        return x * torch.tanh(F.softplus(x))

# Using functional API for activations
class FunctionalActivationCNN(nn.Module):
    def __init__(self, num_classes=10):
        super(FunctionalActivationCNN, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.fc = nn.Linear(128 * 8 * 8, num_classes)
        self.pool = nn.MaxPool2d(2, 2)
    
    def forward(self, x):
        x = self.pool(F.relu(self.conv1(x)))
        x = self.pool(F.gelu(self.conv2(x)))  # Using functional GELU
        x = x.view(-1, 128 * 8 * 8)
        x = self.fc(x)
        return x

# Comparison of activation functions
def compare_activations():
    x = torch.linspace(-5, 5, 100)
    
    activations = {
        'ReLU': F.relu(x),
        'LeakyReLU': F.leaky_relu(x, 0.01),
        'ELU': F.elu(x),
        'SELU': F.selu(x),
        'GELU': F.gelu(x),
        'Swish': x * torch.sigmoid(x),
        'Mish': x * torch.tanh(F.softplus(x))
    }
    
    return activations

# Usage
model = CNNWithDifferentActivations()
activations = compare_activations()
print("Available activations implemented")
```

## 9. How do you implement data augmentation for CNNs?

**Answer:**
```python
import torch
import torchvision.transforms as transforms
import albumentations as A
from albumentations.pytorch import ToTensorV2

# Basic torchvision transforms
train_transform_basic = transforms.Compose([
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.RandomRotation(degrees=15),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2, hue=0.1),
    transforms.RandomResizedCrop(224, scale=(0.8, 1.0)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

val_transform_basic = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

# Advanced albumentations transforms
train_transform_advanced = A.Compose([
    A.RandomRotate90(),
    A.Flip(),
    A.Transpose(),
    A.OneOf([
        A.IAAAdditiveGaussianNoise(),
        A.GaussNoise(),
    ], p=0.2),
    A.OneOf([
        A.MotionBlur(p=0.2),
        A.MedianBlur(blur_limit=3, p=0.1),
        A.Blur(blur_limit=3, p=0.1),
    ], p=0.2),
    A.ShiftScaleRotate(shift_limit=0.0625, scale_limit=0.2, rotate_limit=15, p=0.8),
    A.OneOf([
        A.OpticalDistortion(p=0.3),
        A.GridDistortion(p=0.1),
        A.IAAPiecewiseAffine(p=0.3),
    ], p=0.2),
    A.OneOf([
        A.CLAHE(clip_limit=2),
        A.IAASharpen(),
        A.IAAEmboss(),
        A.RandomBrightnessContrast(),
    ], p=0.3),
    A.HueSaturationValue(p=0.3),
    A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ToTensorV2()
])

# Custom data augmentation class
class CustomAugmentation:
    def __init__(self, p=0.5):
        self.p = p
    
    def __call__(self, image):
        if torch.rand(1) < self.p:
            # Random horizontal flip
            if torch.rand(1) < 0.5:
                image = torch.flip(image, [-1])  # Flip along width dimension
            
            # Random brightness adjustment
            if torch.rand(1) < 0.5:
                brightness_factor = 0.8 + 0.4 * torch.rand(1)
                image = image * brightness_factor
            
            # Random contrast adjustment
            if torch.rand(1) < 0.5:
                contrast_factor = 0.8 + 0.4 * torch.rand(1)
                mean = torch.mean(image)
                image = (image - mean) * contrast_factor + mean
        
        return image

# Mixup augmentation
def mixup_data(x, y, alpha=1.0):
    """Compute the mixup data"""
    if alpha > 0:
        lam = torch.distributions.Beta(alpha, alpha).sample()
    else:
        lam = 1
    
    batch_size = x.size()[0]
    index = torch.randperm(batch_size)
    
    mixed_x = lam * x + (1 - lam) * x[index, :]
    y_a, y_b = y, y[index]
    return mixed_x, y_a, y_b, lam

def mixup_criterion(criterion, pred, y_a, y_b, lam):
    return lam * criterion(pred, y_a) + (1 - lam) * criterion(pred, y_b)

# Cutout augmentation
class Cutout:
    def __init__(self, n_holes=1, length=16):
        self.n_holes = n_holes
        self.length = length
    
    def __call__(self, img):
        h = img.size(1)
        w = img.size(2)
        
        mask = torch.ones((h, w), dtype=torch.float32)
        
        for n in range(self.n_holes):
            y = torch.randint(h, (1,)).item()
            x = torch.randint(w, (1,)).item()
            
            y1 = max(0, y - self.length // 2)
            y2 = min(h, y + self.length // 2)
            x1 = max(0, x - self.length // 2)
            x2 = min(w, x + self.length // 2)
            
            mask[y1:y2, x1:x2] = 0
        
        mask = mask.expand_as(img)
        img = img * mask
        
        return img

# Usage example
train_transform_with_cutout = transforms.Compose([
    transforms.RandomHorizontalFlip(),
    transforms.ToTensor(),
    Cutout(n_holes=1, length=16)
])
```

## 10. How do you implement transfer learning with CNNs?

**Answer:**
```python
import torch
import torch.nn as nn
import torchvision.models as models

# 1. Load pretrained model and freeze all layers
def create_transfer_learning_model_pretrained(num_classes=10, freeze_backbone=True):
    # Load pretrained ResNet18
    model = models.resnet18(pretrained=True)
    
    if freeze_backbone:
        # Freeze all parameters
        for param in model.parameters():
            param.requires_grad = False
    
    # Replace the final classifier layer
    model.fc = nn.Linear(model.fc.in_features, num_classes)
    
    # Only the new layer will have requires_grad=True
    return model

# 2. Fine-tuning with differential learning rates
def create_transfer_learning_model_finetune(num_classes=10):
    model = models.resnet18(pretrained=True)
    
    # Freeze early layers
    for param in model.parameters():
        param.requires_grad = False
    
    # Unfreeze last layer block for fine-tuning
    for param in model.layer4.parameters():
        param.requires_grad = True
    
    # Replace classifier
    model.fc = nn.Linear(model.fc.in_features, num_classes)
    
    return model

# 3. Setup optimizer with differential learning rates
def setup_differential_optimizer(model, base_lr=1e-4):
    # Parameters that require gradients
    params_to_update = []
    for param in model.parameters():
        if param.requires_grad:
            params_to_update.append(param)
    
    # Different learning rates for different parts
    param_groups = [
        {'params': model.fc.parameters(), 'lr': base_lr},  # New classifier
        {'params': model.layer4.parameters(), 'lr': base_lr * 0.1}  # Fine-tune layer4
    ]
    
    optimizer = torch.optim.Adam(param_groups, weight_decay=1e-4)
    return optimizer

# 4. Feature extraction approach
class FeatureExtractorCNN(nn.Module):
    def __init__(self, num_classes=10):
        super(FeatureExtractorCNN, self).__init__()
        
        # Load pretrained model as feature extractor
        backbone = models.resnet18(pretrained=True)
        
        # Remove the final classifier
        self.features = nn.Sequential(*list(backbone.children())[:-1])  # Remove fc layer
        
        # Freeze backbone
        for param in self.features.parameters():
            param.requires_grad = False
        
        # New classifier
        self.classifier = nn.Linear(backbone.fc.in_features, num_classes)
    
    def forward(self, x):
        # Extract features
        x = self.features(x)  # (batch, 512, 1, 1)
        x = torch.flatten(x, 1)  # (batch, 512)
        x = self.classifier(x)   # (batch, num_classes)
        return x

# 5. Progressive unfreezing
class ProgressiveTransferLearning(nn.Module):
    def __init__(self, num_classes=10):
        super(ProgressiveTransferLearning, self).__init__()
        self.backbone = models.resnet18(pretrained=True)
        self.num_classes = num_classes
        self.setup_layers()
    
    def setup_layers(self):
        # Initially freeze all layers
        for param in self.backbone.parameters():
            param.requires_grad = False
        
        # Replace classifier
        self.backbone.fc = nn.Linear(self.backbone.fc.in_features, self.num_classes)
    
    def unfreeze_layers(self, epoch):
        """Progressively unfreeze layers"""
        if epoch >= 10:
            # Unfreeze layer4
            for param in self.backbone.layer4.parameters():
                param.requires_grad = True
        
        if epoch >= 20:
            # Unfreeze layer3
            for param in self.backbone.layer3.parameters():
                param.requires_grad = True
        
        if epoch >= 30:
            # Unfreeze layer2
            for param in self.backbone.layer2.parameters():
                param.requires_grad = True

# 6. Custom head for transfer learning
class CustomHeadTransferLearning(nn.Module):
    def __init__(self, num_classes=10):
        super(CustomHeadTransferLearning, self).__init__()
        
        # Backbone
        backbone = models.resnet18(pretrained=True)
        self.features = nn.Sequential(*list(backbone.children())[:-1])
        
        # Freeze backbone
        for param in self.features.parameters():
            param.requires_grad = False
        
        # Custom classifier head
        self.classifier = nn.Sequential(
            nn.Dropout(0.5),
            nn.Linear(backbone.fc.in_features, 512),
            nn.ReLU(),
            nn.BatchNorm1d(512),
            nn.Dropout(0.3),
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.BatchNorm1d(256),
            nn.Linear(256, num_classes)
        )
    
    def forward(self, x):
        x = self.features(x)
        x = torch.flatten(x, 1)
        x = self.classifier(x)
        return x

# Usage examples
def train_transfer_learning_model():
    # Create model
    model = create_transfer_learning_model_pretrained(num_classes=10, freeze_backbone=True)
    
    # Setup optimizer
    if any(param.requires_grad for param in model.parameters()):
        optimizer = setup_differential_optimizer(model)
    else:
        # Only train the new classifier layer
        optimizer = torch.optim.Adam(model.fc.parameters(), lr=1e-3)
    
    # Training code would go here
    return model, optimizer

# Example with custom head
model = CustomHeadTransferLearning(num_classes=10)
sample_input = torch.randn(1, 3, 224, 224)
output = model(sample_input)
print(f"Custom head model output shape: {output.shape}")
```

## 11. How do you implement different CNN architectures?

**Answer:**
```python
import torch
import torch.nn as nn

# 1. LeNet-5 Architecture
class LeNet5(nn.Module):
    def __init__(self, num_classes=10):
        super(LeNet5, self).__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 6, 5),      # (28, 28) -> (24, 24)
            nn.Tanh(),
            nn.AvgPool2d(2, 2),      # (24, 24) -> (12, 12)
            nn.Conv2d(6, 16, 5),     # (12, 12) -> (8, 8)
            nn.Tanh(),
            nn.AvgPool2d(2, 2),      # (8, 8) -> (4, 4)
            nn.Conv2d(16, 120, 4),   # (4, 4) -> (1, 1)
            nn.Tanh()
        )
        self.classifier = nn.Sequential(
            nn.Linear(120, 84),
            nn.Tanh(),
            nn.Linear(84, num_classes)
        )
    
    def forward(self, x):
        x = self.features(x)
        x = torch.flatten(x, 1)
        x = self.classifier(x)
        return x

# 2. AlexNet Architecture
class AlexNet(nn.Module):
    def __init__(self, num_classes=1000):
        super(AlexNet, self).__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 96, 11, stride=4, padding=2),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(3, stride=2),
            
            nn.Conv2d(96, 256, 5, padding=2),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(3, stride=2),
            
            nn.Conv2d(256, 384, 3, padding=1),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(384, 384, 3, padding=1),
            nn.ReLU(inplace=True),
            
            nn.Conv2d(384, 256, 3, padding=1),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(3, stride=2),
        )
        self.avgpool = nn.AdaptiveAvgPool2d((6, 6))
        self.classifier = nn.Sequential(
            nn.Dropout(),
            nn.Linear(256 * 6 * 6, 4096),
            nn.ReLU(inplace=True),
            nn.Dropout(),
            nn.Linear(4096, 4096),
            nn.ReLU(inplace=True),
            nn.Linear(4096, num_classes),
        )
    
    def forward(self, x):
        x = self.features(x)
        x = self.avgpool(x)
        x = torch.flatten(x, 1)
        x = self.classifier(x)
        return x

# 3. VGG Architecture (simplified)
class VGG(nn.Module):
    def __init__(self, num_classes=1000, vgg_type='VGG16'):
        super(VGG, self).__init__()
        
        cfg = {
            'VGG11': [64, 'M', 128, 'M', 256, 256, 'M', 512, 512, 'M', 512, 512, 'M'],
            'VGG13': [64, 64, 'M', 128, 128, 'M', 256, 256, 'M', 512, 512, 'M', 512, 512, 'M'],
            'VGG16': [64, 64, 'M', 128, 128, 'M', 256, 256, 256, 'M', 512, 512, 512, 'M', 512, 512, 512, 'M'],
            'VGG19': [64, 64, 'M', 128, 128, 'M', 256, 256, 256, 256, 'M', 512, 512, 512, 512, 'M', 512, 512, 512, 512, 'M'],
        }
        
        self.features = self._make_layers(cfg[vgg_type])
        self.classifier = nn.Sequential(
            nn.Linear(512 * 7 * 7, 4096),
            nn.ReLU(True),
            nn.Dropout(),
            nn.Linear(4096, 4096),
            nn.ReLU(True),
            nn.Dropout(),
            nn.Linear(4096, num_classes),
        )
    
    def _make_layers(self, cfg):
        layers = []
        in_channels = 3
        for x in cfg:
            if x == 'M':
                layers += [nn.MaxPool2d(kernel_size=2, stride=2)]
            else:
                layers += [nn.Conv2d(in_channels, x, kernel_size=3, padding=1),
                          nn.BatchNorm2d(x),
                          nn.ReLU(inplace=True)]
                in_channels = x
        layers += [nn.AdaptiveAvgPool2d((7, 7))]
        return nn.Sequential(*layers)
    
    def forward(self, x):
        x = self.features(x)
        x = torch.flatten(x, 1)
        x = self.classifier(x)
        return x

# 4. Simple CNN for CIFAR-10
class SimpleCNN(nn.Module):
    def __init__(self, num_classes=10):
        super(SimpleCNN, self).__init__()
        self.conv1 = nn.Conv2d(3, 32, 3, padding=1)
        self.conv2 = nn.Conv2d(32, 64, 3, padding=1)
        self.conv3 = nn.Conv2d(64, 128, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(0.25)
        
        # Calculate the size after convolutions and pooling
        self.fc1 = nn.Linear(128 * 4 * 4, 512)
        self.fc2 = nn.Linear(512, num_classes)
        
    def forward(self, x):
        x = self.pool(self.relu(self.conv1(x)))  # 32x32 -> 16x16
        x = self.pool(self.relu(self.conv2(x)))  # 16x16 -> 8x8
        x = self.pool(self.relu(self.conv3(x)))  # 8x8 -> 4x4
        x = self.dropout(x)
        x = x.view(-1, 128 * 4 * 4)
        x = self.relu(self.fc1(x))
        x = self.dropout(x)
        x = self.fc2(x)
        return x

# 5. Depthwise Separable CNN (MobileNet-style)
class DepthwiseSeparableConv(nn.Module):
    def __init__(self, in_channels, out_channels, stride=1):
        super(DepthwiseSeparableConv, self).__init__()
        self.depthwise = nn.Conv2d(in_channels, in_channels, 3, stride, 1, groups=in_channels, bias=False)
        self.bn1 = nn.BatchNorm2d(in_channels)
        self.pointwise = nn.Conv2d(in_channels, out_channels, 1, 1, 0, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)
    
    def forward(self, x):
        x = self.depthwise(x)
        x = self.bn1(x)
        x = self.relu(x)
        x = self.pointwise(x)
        x = self.bn2(x)
        x = self.relu(x)
        return x

class MobileNet(nn.Module):
    def __init__(self, num_classes=1000):
        super(MobileNet, self).__init__()
        
        def conv_bn(inp, oup, stride):
            return nn.Sequential(
                nn.Conv2d(inp, oup, 3, stride, 1, bias=False),
                nn.BatchNorm2d(oup),
                nn.ReLU(inplace=True)
            )
        
        self.model = nn.Sequential(
            conv_bn(3, 32, 2),
            DepthwiseSeparableConv(32, 64, 1),
            DepthwiseSeparableConv(64, 128, 2),
            DepthwiseSeparableConv(128, 128, 1),
            DepthwiseSeparableConv(128, 256, 2),
            DepthwiseSeparableConv(256, 256, 1),
            DepthwiseSeparableConv(256, 512, 2),
            DepthwiseSeparableConv(512, 512, 1),
            DepthwiseSeparableConv(512, 512, 1),
            DepthwiseSeparableConv(512, 512, 1),
            DepthwiseSeparableConv(512, 512, 1),
            DepthwiseSeparableConv(512, 512, 1),
            DepthwiseSeparableConv(512, 1024, 2),
            DepthwiseSeparableConv(1024, 1024, 1),
            nn.AdaptiveAvgPool2d(1)
        )
        self.fc = nn.Linear(1024, num_classes)
    
    def forward(self, x):
        x = self.model(x)
        x = x.view(-1, 1024)
        x = self.fc(x)
        return x

# Test different architectures
def test_architectures():
    input_tensor = torch.randn(1, 3, 224, 224)
    
    # Test different models
    models = {
        'LeNet5': LeNet5(num_classes=10),
        'AlexNet': AlexNet(num_classes=1000),
        'SimpleCNN': SimpleCNN(num_classes=10),
        'MobileNet': MobileNet(num_classes=1000)
    }
    
    for name, model in models.items():
        if name == 'LeNet5':
            # LeNet5 expects 1 channel input
            input_tensor_gray = torch.randn(1, 1, 28, 28)
            output = model(input_tensor_gray)
        else:
            output = model(input_tensor)
        print(f"{name} output shape: {output.shape}")

# Run tests
test_architectures()
```

## 12. How do you implement attention mechanisms in CNNs?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# 1. Squeeze-and-Excitation (SE) Block
class SEBlock(nn.Module):
    def __init__(self, channel, reduction=16):
        super(SEBlock, self).__init__()
        self.avg_pool = nn.AdaptiveAvgPool2d(1)
        self.fc = nn.Sequential(
            nn.Linear(channel, channel // reduction, bias=False),
            nn.ReLU(inplace=True),
            nn.Linear(channel // reduction, channel, bias=False),
            nn.Sigmoid()
        )
    
    def forward(self, x):
        b, c, _, _ = x.size()
        y = self.avg_pool(x).view(b, c)
        y = self.fc(y).view(b, c, 1, 1)
        return x * y.expand_as(x)

# 2. Convolutional Block Attention Module (CBAM)
class ChannelAttention(nn.Module):
    def __init__(self, in_planes, ratio=16):
        super(ChannelAttention, self).__init__()
        self.avg_pool = nn.AdaptiveAvgPool2d(1)
        self.max_pool = nn.AdaptiveMaxPool2d(1)
        
        self.fc1 = nn.Conv2d(in_planes, in_planes // ratio, 1, bias=False)
        self.relu1 = nn.ReLU()
        self.fc2 = nn.Conv2d(in_planes // ratio, in_planes, 1, bias=False)
        self.sigmoid = nn.Sigmoid()
    
    def forward(self, x):
        avg_out = self.fc2(self.relu1(self.fc1(self.avg_pool(x))))
        max_out = self.fc2(self.relu1(self.fc1(self.max_pool(x))))
        out = avg_out + max_out
        return self.sigmoid(out)

class SpatialAttention(nn.Module):
    def __init__(self, kernel_size=7):
        super(SpatialAttention, self).__init__()
        self.conv1 = nn.Conv2d(2, 1, kernel_size, padding=kernel_size//2, bias=False)
        self.sigmoid = nn.Sigmoid()
    
    def forward(self, x):
        avg_out = torch.mean(x, dim=1, keepdim=True)
        max_out, _ = torch.max(x, dim=1, keepdim=True)
        x = torch.cat([avg_out, max_out], dim=1)
        x = self.conv1(x)
        return self.sigmoid(x)

class CBAM(nn.Module):
    def __init__(self, in_planes, ratio=16, kernel_size=7):
        super(CBAM, self).__init__()
        self.channel_attention = ChannelAttention(in_planes, ratio)
        self.spatial_attention = SpatialAttention(kernel_size)
    
    def forward(self, x):
        x = x * self.channel_attention(x)
        x = x * self.spatial_attention(x)
        return x

# 3. Self-Attention for CNNs
class SelfAttention(nn.Module):
    def __init__(self, in_channels):
        super(SelfAttention, self).__init__()
        self.query_conv = nn.Conv2d(in_channels, in_channels // 8, 1)
        self.key_conv = nn.Conv2d(in_channels, in_channels // 8, 1)
        self.value_conv = nn.Conv2d(in_channels, in_channels, 1)
        self.gamma = nn.Parameter(torch.zeros(1))
        
        self.softmax = nn.Softmax(dim=-1)
    
    def forward(self, x):
        batch_size, C, width, height = x.size()
        
        proj_query = self.query_conv(x).view(batch_size, -1, width * height).permute(0, 2, 1)
        proj_key = self.key_conv(x).view(batch_size, -1, width * height)
        energy = torch.bmm(proj_query, proj_key)
        attention = self.softmax(energy)
        proj_value = self.value_conv(x).view(batch_size, -1, width * height)
        
        out = torch.bmm(proj_value, attention.permute(0, 2, 1))
        out = out.view(batch_size, C, width, height)
        
        out = self.gamma * out + x
        return out

# 4. Non-Local Attention Block
class NonLocalBlock(nn.Module):
    def __init__(self, in_channels):
        super(NonLocalBlock, self).__init__()
        self.in_channels = in_channels
        self.inter_channels = in_channels // 2
        
        self.g = nn.Conv2d(in_channels, self.inter_channels, 1)
        self.theta = nn.Conv2d(in_channels, self.inter_channels, 1)
        self.phi = nn.Conv2d(in_channels, self.inter_channels, 1)
        
        self.W = nn.Sequential(
            nn.Conv2d(self.inter_channels, in_channels, 1),
            nn.BatchNorm2d(in_channels)
        )
        
    def forward(self, x):
        batch_size = x.size(0)
        
        g_x = self.g(x).view(batch_size, self.inter_channels, -1)
        g_x = g_x.permute(0, 2, 1)
        
        theta_x = self.theta(x).view(batch_size, self.inter_channels, -1)
        theta_x = theta_x.permute(0, 2, 1)
        
        phi_x = self.phi(x).view(batch_size, self.inter_channels, -1)
        
        f = torch.matmul(theta_x, phi_x)
        f_div_C = F.softmax(f, dim=-1)
        
        y = torch.matmul(f_div_C, g_x)
        y = y.permute(0, 2, 1).contiguous()
        y = y.view(batch_size, self.inter_channels, *x.size()[2:])
        W_y = self.W(y)
        z = W_y + x
        
        return z

# 5. CNN with Attention Blocks
class AttentionCNN(nn.Module):
    def __init__(self, num_classes=10, attention_type='se'):
        super(AttentionCNN, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.conv3 = nn.Conv2d(128, 256, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.relu = nn.ReLU()
        
        # Add attention blocks
        if attention_type == 'se':
            self.attention1 = SEBlock(64)
            self.attention2 = SEBlock(128)
            self.attention3 = SEBlock(256)
        elif attention_type == 'cbam':
            self.attention1 = CBAM(64)
            self.attention2 = CBAM(128)
            self.attention3 = CBAM(256)
        elif attention_type == 'self':
            self.attention1 = SelfAttention(64)
            self.attention2 = SelfAttention(128)
            self.attention3 = SelfAttention(256)
        
        self.global_avg_pool = nn.AdaptiveAvgPool2d((1, 1))
        self.fc = nn.Linear(256, num_classes)
    
    def forward(self, x):
        x = self.pool(self.relu(self.conv1(x)))
        x = self.attention1(x)
        
        x = self.pool(self.relu(self.conv2(x)))
        x = self.attention2(x)
        
        x = self.pool(self.relu(self.conv3(x)))
        x = self.attention3(x)
        
        x = self.global_avg_pool(x)
        x = x.view(x.size(0), -1)
        x = self.fc(x)
        
        return x

# 6. Lightweight Attention Module
class LightweightAttention(nn.Module):
    def __init__(self, in_channels, out_channels):
        super(LightweightAttention, self).__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, 1)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, padding=1)
        self.conv3 = nn.Conv2d(out_channels, 1, 1)
        self.sigmoid = nn.Sigmoid()
    
    def forward(self, x):
        attention = self.conv1(x)
        attention = F.relu(self.conv2(attention))
        attention = self.sigmoid(self.conv3(attention))
        return x * attention

# Usage examples
def test_attention_modules():
    x = torch.randn(2, 64, 32, 32)
    
    # Test different attention modules
    se_block = SEBlock(64)
    cbam_block = CBAM(64)
    self_attention = SelfAttention(64)
    nonlocal_block = NonLocalBlock(64)
    
    se_output = se_block(x)
    cbam_output = cbam_block(x)
    self_output = self_attention(x)
    nonlocal_output = nonlocal_block(x)
    
    print(f"SE Block output shape: {se_output.shape}")
    print(f"CBAM output shape: {cbam_output.shape}")
    print(f"Self-Attention output shape: {self_output.shape}")
    print(f"Non-Local output shape: {nonlocal_output.shape}")

# Test attention modules
test_attention_modules()
```

## 13. How do you implement CNN visualization techniques?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import matplotlib.pyplot as plt
import numpy as np

# 1. Feature Map Visualization
def visualize_feature_maps(model, input_tensor, layer_name=None, max_channels=16):
    """Visualize feature maps from CNN layers"""
    model.eval()
    feature_maps = {}
    
    # Register hooks to capture feature maps
    def hook_fn(module, input, output):
        feature_maps['features'] = output.detach()
    
    # Register hook on specific layer or all conv layers
    hooks = []
    if layer_name:
        target_layer = dict(model.named_modules())[layer_name]
        hook = target_layer.register_forward_hook(hook_fn)
        hooks.append(hook)
    else:
        # Hook all conv layers
        for name, module in model.named_modules():
            if isinstance(module, nn.Conv2d):
                hook = module.register_forward_hook(hook_fn)
                hooks.append(hook)
    
    # Forward pass
    with torch.no_grad():
        output = model(input_tensor)
    
    # Remove hooks
    for hook in hooks:
        hook.remove()
    
    # Visualize feature maps
    if 'features' in feature_maps:
        features = feature_maps['features']
        batch_size, channels, height, width = features.shape
        
        # Limit number of channels to visualize
        channels_to_show = min(channels, max_channels)
        
        fig, axes = plt.subplots(4, 4, figsize=(12, 12))
        axes = axes.ravel()
        
        for i in range(channels_to_show):
            ax = axes[i]
            feature_map = features[0, i].cpu().numpy()
            ax.imshow(feature_map, cmap='viridis')
            ax.set_title(f'Channel {i}')
            ax.axis('off')
        
        plt.tight_layout()
        plt.show()
    
    return feature_maps

# 2. Grad-CAM Implementation
class GradCAM:
    def __init__(self, model, target_layer):
        self.model = model.eval()
        self.target_layer = target_layer
        self.gradients = None
        self.activations = None
        self._register_hooks()
    
    def _register_hooks(self):
        def forward_hook(module, input, output):
            self.activations = output
        
        def backward_hook(module, grad_input, grad_output):
            self.gradients = grad_output[0]
        
        self.target_layer.register_forward_hook(forward_hook)
        self.target_layer.register_backward_hook(backward_hook)
    
    def generate_cam(self, input_tensor, target_class=None):
        self.model.zero_grad()
        
        # Forward pass
        output = self.model(input_tensor)
        
        if target_class is None:
            target_class = output.argmax(dim=1).item()
        
        # Backward pass
        self.model.zero_grad()
        one_hot = torch.zeros_like(output)
        one_hot[0][target_class] = 1
        output.backward(gradient=one_hot, retain_graph=True)
        
        # Generate CAM
        weights = torch.mean(self.gradients, dim=(2, 3), keepdim=True)
        cam = torch.sum(weights * self.activations, dim=1, keepdim=True)
        cam = F.relu(cam)
        cam = F.interpolate(cam, size=input_tensor.shape[2:], mode='bilinear', align_corners=False)
        
        return cam

# 3. Class Activation Mapping (CAM)
class CAMVisualizer:
    def __init__(self, model):
        self.model = model
        self.feature_extractor = nn.Sequential(*list(model.children())[:-1])
        self.classifier = list(model.children())[-1]
    
    def generate_cam(self, input_tensor, target_class=None):
        # Get features
        features = self.feature_extractor(input_tensor)
        features = features.view(features.size(0), -1)
        
        # Get weights for target class
        if target_class is None:
            output = self.classifier(features)
            target_class = output.argmax(dim=1).item()
        
        # Get weights from classifier
        weights = list(self.classifier.parameters())[0][target_class]
        
        # Reshape features for CAM
        features_map = features.view(1, -1, 1, 1)
        cam = torch.sum(weights.view(1, -1, 1, 1) * features_map, dim=1, keepdim=True)
        cam = F.relu(cam)
        cam = F.interpolate(cam, size=input_tensor.shape[2:], mode='bilinear', align_corners=False)
        
        return cam

# 4. Filter Visualization
def visualize_filters(model, layer_name, filter_index=0, input_size=(1, 3, 224, 224)):
    """Visualize what a specific filter responds to"""
    model.eval()
    
    # Get target layer
    target_layer = dict(model.named_modules())[layer_name]
    
    # Create random input
    input_tensor = torch.randn(input_size, requires_grad=True)
    optimizer = torch.optim.Adam([input_tensor], lr=0.1)
    
    for i in range(100):
        optimizer.zero_grad()
        
        # Forward pass
        output = model(input_tensor)
        
        # Get activation of specific filter
        layer_output = None
        def hook_fn(module, input, output):
            nonlocal layer_output
            layer_output = output
        
        hook = target_layer.register_forward_hook(hook_fn)
        model(input_tensor)
        hook.remove()
        
        # Maximize activation of specific filter
        loss = -layer_output[0, filter_index].mean()
        loss.backward()
        optimizer.step()
    
    return input_tensor.detach()

# 5. Saliency Map Visualization
def generate_saliency_map(model, input_tensor, target_class=None):
    """Generate saliency map showing important pixels"""
    model.eval()
    input_tensor.requires_grad_()
    
    # Forward pass
    output = model(input_tensor)
    
    if target_class is None:
        target_class = output.argmax(dim=1).item()
    
    # Backward pass
    model.zero_grad()
    output[0, target_class].backward()
    
    # Generate saliency map
    saliency = torch.abs(input_tensor.grad)
    saliency = torch.max(saliency, dim=1, keepdim=True)[0]
    
    return saliency

# 6. Complete Visualization Pipeline
class CNNVisualizer:
    def __init__(self, model):
        self.model = model
    
    def visualize_all(self, input_tensor, target_class=None):
        """Comprehensive visualization of CNN"""
        fig, axes = plt.subplots(2, 2, figsize=(15, 15))
        
        # Original image
        original = input_tensor[0].permute(1, 2, 0).cpu().numpy()
        axes[0, 0].imshow(np.clip(original, 0, 1))
        axes[0, 0].set_title('Original Image')
        axes[0, 0].axis('off')
        
        # Saliency map
        saliency = generate_saliency_map(self.model, input_tensor, target_class)
        saliency_map = saliency[0, 0].cpu().numpy()
        axes[0, 1].imshow(saliency_map, cmap='hot')
        axes[0, 1].set_title('Saliency Map')
        axes[0, 1].axis('off')
        
        # Grad-CAM
        # Assuming the last conv layer is named appropriately
        for name, module in self.model.named_modules():
            if isinstance(module, nn.Conv2d):
                target_layer = module
        
        grad_cam = GradCAM(self.model, target_layer)
        cam = grad_cam.generate_cam(input_tensor, target_class)
        cam_map = cam[0, 0].cpu().numpy()
        
        axes[1, 0].imshow(np.clip(original, 0, 1))
        axes[1, 0].imshow(cam_map, cmap='jet', alpha=0.5)
        axes[1, 0].set_title('Grad-CAM')
        axes[1, 0].axis('off')
        
        # Overlay saliency on image
        axes[1, 1].imshow(np.clip(original, 0, 1))
        axes[1, 1].imshow(saliency_map, cmap='jet', alpha=0.5)
        axes[1, 1].set_title('Saliency Overlay')
        axes[1, 1].axis('off')
        
        plt.tight_layout()
        plt.show()

# 7. Utility functions for visualization
def denormalize(tensor, mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]):
    """Denormalize tensor for visualization"""
    for t, m, s in zip(tensor, mean, std):
        t.mul_(s).add_(m)
    return tensor

def show_cam_on_image(img, mask, use_rgb=True):
    """Overlay CAM on image"""
    heatmap = cv2.applyColorMap(np.uint8(255 * mask), cv2.COLORMAP_JET)
    if use_rgb:
        heatmap = cv2.cvtColor(heatmap, cv2.COLOR_BGR2RGB)
    cam = heatmap + np.float32(img)
    cam = cam / np.max(cam)
    return np.uint8(255 * cam)

# Usage example
def demonstrate_cnn_visualization():
    # Create a simple CNN model
    model = SimpleCNN(num_classes=10)
    model.eval()
    
    # Create sample input
    input_tensor = torch.randn(1, 3, 32, 32)
    
    # Initialize visualizer
    visualizer = CNNVisualizer(model)
    
    # This would show various visualizations
    # visualizer.visualize_all(input_tensor)
    
    print("CNN visualization tools ready for use")

# Run demonstration
demonstrate_cnn_visualization()
```

## 14. How do you implement CNN model optimization techniques?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.optim.lr_scheduler import CosineAnnealingLR, ReduceLROnPlateau
import torch.nn.utils.prune as prune

# 1. Model Pruning
class PrunedCNN(nn.Module):
    def __init__(self, num_classes=10, pruning_ratio=0.2):
        super(PrunedCNN, self).__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.conv3 = nn.Conv2d(128, 256, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(0.5)
        self.fc = nn.Linear(256 * 4 * 4, num_classes)
        
        # Store pruning ratio
        self.pruning_ratio = pruning_ratio
    
    def forward(self, x):
        x = self.pool(self.relu(self.conv1(x)))
        x = self.pool(self.relu(self.conv2(x)))
        x = self.pool(self.relu(self.conv3(x)))
        x = x.view(-1, 256 * 4 * 4)
        x = self.dropout(x)
        x = self.fc(x)
        return x
    
    def prune_model(self):
        """Apply pruning to convolutional layers"""
        parameters_to_prune = [
            (self.conv1, 'weight'),
            (self.conv2, 'weight'),
            (self.conv3, 'weight'),
        ]
        
        prune.global_unstructured(
            parameters_to_prune,
            pruning_method=prune.L1Unstructured,
            amount=self.pruning_ratio,
        )

# 2. Model Quantization
class QuantizedCNN(nn.Module):
    def __init__(self, num_classes=10):
        super(QuantizedCNN, self).__init__()
        self.quant = torch.quantization.QuantStub()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 128, 3, padding=1)
        self.conv3 = nn.Conv2d(128, 256, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(0.5)
        self.fc = nn.Linear(256 * 4 * 4, num_classes)
        self.dequant = torch.quantization.DeQuantStub()
    
    def forward(self, x):
        x = self.quant(x)
        x = self.pool(self.relu(self.conv1(x)))
        x = self.pool(self.relu(self.conv2(x)))
        x = self.pool(self.relu(self.conv3(x)))
        x = x.view(-1, 256 * 4 * 4)
        x = self.dropout(x)
        x = self.fc(x)
        x = self.dequant(x)
        return x
    
    def fuse_model(self):
        """Fuse convolution, batch norm, and ReLU layers"""
        torch.quantization.fuse_modules(self, [
            ['conv1', 'relu'],
            ['conv2', 'relu'],
            ['conv3', 'relu']
        ], inplace=True)

# 3. Knowledge Distillation
class DistilledCNN(nn.Module):
    def __init__(self, student_model, teacher_model, temperature=3.0, alpha=0.7):
        super(DistilledCNN, self).__init__()
        self.student = student_model
        self.teacher = teacher_model
        self.temperature = temperature
        self.alpha = alpha
        
        # Freeze teacher model
        for param in self.teacher.parameters():
            param.requires_grad = False
    
    def forward(self, x):
        student_output = self.student(x)
        with torch.no_grad():
            teacher_output = self.teacher(x)
        return student_output, teacher_output
    
    def distillation_loss(self, student_output, teacher_output, labels):
        """Compute distillation loss"""
        # Soft targets from teacher
        soft_targets = F.softmax(teacher_output / self.temperature, dim=1)
        soft_predictions = F.log_softmax(student_output / self.temperature, dim=1)
        
        # Hard targets (true labels)
        hard_loss = F.cross_entropy(student_output, labels)
        
        # Soft loss (knowledge distillation)
        soft_loss = F.kl_div(soft_predictions, soft_targets, reduction='batchmean') * (self.temperature ** 2)
        
        # Combined loss
        return self.alpha * soft_loss + (1 - self.alpha) * hard_loss

# 4. Mixed Precision Training
def train_mixed_precision(model, dataloader, optimizer, criterion, device):
    """Train model with mixed precision"""
    model.train()
    scaler = torch.cuda.amp.GradScaler()
    
    for inputs, targets in dataloader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        optimizer.zero_grad()
        
        # Forward pass with automatic mixed precision
        with torch.cuda.amp.autocast():
            outputs = model(inputs)
            loss = criterion(outputs, targets)
        
        # Backward pass
        scaler.scale(loss).backward()
        scaler.step(optimizer)
        scaler.update()

# 5. Gradient Accumulation
def train_with_gradient_accumulation(model, dataloader, optimizer, criterion, 
                                   accumulation_steps=4, device='cpu'):
    """Train with gradient accumulation for larger effective batch sizes"""
    model.train()
    
    for i, (inputs, targets) in enumerate(dataloader):
        inputs, targets = inputs.to(device), targets.to(device)
        
        # Normalize loss by accumulation steps
        outputs = model(inputs)
        loss = criterion(outputs, targets) / accumulation_steps
        
        # Backward pass
        loss.backward()
        
        # Update weights after accumulating gradients
        if (i + 1) % accumulation_steps == 0:
            optimizer.step()
            optimizer.zero_grad()

# 6. Advanced Optimizer Setup
def setup_advanced_optimizer(model, optimizer_type='adamw'):
    """Setup advanced optimizers for CNN training"""
    if optimizer_type == 'adamw':
        optimizer = optim.AdamW(model.parameters(), lr=0.001, weight_decay=0.01)
    elif optimizer_type == 'sgd':
        optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9, weight_decay=0.0001)
    elif optimizer_type == 'rmsprop':
        optimizer = optim.RMSprop(model.parameters(), lr=0.001, alpha=0.99, weight_decay=0.0001)
    else:
        optimizer = optim.Adam(model.parameters(), lr=0.001)
    
    return optimizer

# 7. Learning Rate Scheduling
def setup_lr_scheduler(optimizer, scheduler_type='cosine', **kwargs):
    """Setup learning rate scheduler"""
    if scheduler_type == 'cosine':
        scheduler = CosineAnnealingLR(optimizer, T_max=kwargs.get('T_max', 100))
    elif scheduler_type == 'plateau':
        scheduler = ReduceLROnPlateau(optimizer, mode='min', factor=0.5, patience=10)
    elif scheduler_type == 'step':
        scheduler = optim.lr_scheduler.StepLR(optimizer, step_size=30, gamma=0.1)
    else:
        scheduler = None
    
    return scheduler

# 8. Model Checkpointing with Best Validation
class ModelCheckpoint:
    def __init__(self, filepath, monitor='val_loss', mode='min', save_best_only=True):
        self.filepath = filepath
        self.monitor = monitor
        self.mode = mode
        self.save_best_only = save_best_only
        self.best_value = float('inf') if mode == 'min' else float('-inf')
    
    def __call__(self, model, current_value):
        if self.save_best_only:
            if (self.mode == 'min' and current_value < self.best_value) or \
               (self.mode == 'max' and current_value > self.best_value):
                self.best_value = current_value
                torch.save(model.state_dict(), self.filepath)
                print(f"Model saved to {self.filepath}")
        else:
            torch.save(model.state_dict(), self.filepath)

# 9. Early Stopping
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

# 10. Complete Training Pipeline with Optimizations
def train_optimized_cnn(model, train_loader, val_loader, device, epochs=100):
    """Complete training pipeline with optimizations"""
    
    # Setup optimizer and scheduler
    optimizer = setup_advanced_optimizer(model, 'adamw')
    scheduler = setup_lr_scheduler(optimizer, 'cosine', T_max=epochs)
    
    # Setup loss function
    criterion = nn.CrossEntropyLoss()
    
    # Setup callbacks
    checkpoint = ModelCheckpoint('best_model.pth', monitor='val_loss', mode='min')
    early_stopping = EarlyStopping(patience=15, min_delta=0.001)
    
    # Training loop
    for epoch in range(epochs):
        # Training phase
        model.train()
        train_loss = 0.0
        
        for inputs, targets in train_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            loss.backward()
            
            # Gradient clipping
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            
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
        
        # Calculate metrics
        avg_train_loss = train_loss / len(train_loader)
        avg_val_loss = val_loss / len(val_loader)
        accuracy = 100 * correct / total
        
        # Update scheduler
        if isinstance(scheduler, ReduceLROnPlateau):
            scheduler.step(avg_val_loss)
        else:
            scheduler.step()
        
        # Callbacks
        checkpoint(model, avg_val_loss)
        early_stopping(avg_val_loss, model)
        
        # Logging
        print(f'Epoch {epoch+1}/{epochs}:')
        print(f'  Train Loss: {avg_train_loss:.4f}')
        print(f'  Val Loss: {avg_val_loss:.4f}')
        print(f'  Val Acc: {accuracy:.2f}%')
        
        # Early stopping check
        if early_stopping.early_stop:
            print("Early stopping triggered")
            break
    
    return model

# Usage example
def demonstrate_optimization_techniques():
    # Create model
    model = SimpleCNN(num_classes=10)
    
    # Apply pruning
    pruned_model = PrunedCNN(num_classes=10, pruning_ratio=0.3)
    pruned_model.prune_model()
    
    # Setup quantization (requires calibration)
    quantized_model = QuantizedCNN(num_classes=10)
    quantized_model.fuse_model()
    quantized_model.qconfig = torch.quantization.get_default_qconfig('fbgemm')
    torch.quantization.prepare(quantized_model, inplace=True)
    # Calibration would happen here
    torch.quantization.convert(quantized_model, inplace=True)
    
    print("Optimization techniques demonstrated:")
    print("- Pruning applied")
    print("- Quantization setup")
    print("- Knowledge distillation framework")
    print("- Mixed precision training ready")
    print("- Gradient accumulation ready")
    print("- Advanced optimizers available")
    print("- Learning rate scheduling available")
    print("- Model checkpointing ready")
    print("- Early stopping ready")

# Run demonstration
demonstrate_optimization_techniques()
```

## 15. How do you implement CNN model evaluation and metrics?

**Answer:**
```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from sklearn.metrics import accuracy_score, precision_recall_fscore_support, confusion_matrix
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np

# 1. Comprehensive Model Evaluator
class CNNEvaluator:
    def __init__(self, model, device='cpu'):
        self.model = model
        self.device = device
        self.model.to(device)
        self.model.eval()
    
    def evaluate(self, dataloader, criterion=None):
        """Comprehensive evaluation of model"""
        all_predictions = []
        all_targets = []
        all_probabilities = []
        total_loss = 0.0
        
        with torch.no_grad():
            for inputs, targets in dataloader:
                inputs, targets = inputs.to(self.device), targets.to(self.device)
                
                outputs = self.model(inputs)
                probabilities = F.softmax(outputs, dim=1)
                
                if criterion:
                    loss = criterion(outputs, targets)
                    total_loss += loss.item()
                
                _, predicted = torch.max(outputs.data, 1)
                
                all_predictions.extend(predicted.cpu().numpy())
                all_targets.extend(targets.cpu().numpy())
                all_probabilities.extend(probabilities.cpu().numpy())
        
        # Calculate metrics
        accuracy = accuracy_score(all_targets, all_predictions)
        precision, recall, f1, _ = precision_recall_fscore_support(
            all_targets, all_predictions, average='weighted'
        )
        
        avg_loss = total_loss / len(dataloader) if criterion else 0.0
        
        return {
            'accuracy': accuracy,
            'precision': precision,
            'recall': recall,
            'f1_score': f1,
            'loss': avg_loss,
            'predictions': all_predictions,
            'targets': all_targets,
            'probabilities': all_probabilities
        }

# 2. Per-Class Metrics
def calculate_per_class_metrics(targets, predictions, class_names=None):
    """Calculate per-class metrics"""
    unique_classes = np.unique(np.concatenate([targets, predictions]))
    
    if class_names is None:
        class_names = [f"Class_{i}" for i in unique_classes]
    
    metrics = {}
    for i, class_id in enumerate(unique_classes):
        # Create binary masks
        class_mask = targets == class_id
        predicted_class_mask = predictions == class_id
        
        # True/False Positives/Negatives
        tp = np.sum((targets == class_id) & (predictions == class_id))
        fp = np.sum((targets != class_id) & (predictions == class_id))
        fn = np.sum((targets == class_id) & (predictions != class_id))
        tn = np.sum((targets != class_id) & (predictions != class_id))
        
        # Calculate metrics
        precision = tp / (tp + fp) if (tp + fp) > 0 else 0
        recall = tp / (tp + fn) if (tp + fn) > 0 else 0
        f1 = 2 * (precision * recall) / (precision + recall) if (precision + recall) > 0 else 0
        specificity = tn / (tn + fp) if (tn + fp) > 0 else 0
        accuracy = (tp + tn) / (tp + tn + fp + fn) if (tp + tn + fp + fn) > 0 else 0
        
        metrics[class_names[i]] = {
            'precision': precision,
            'recall': recall,
            'f1_score': f1,
            'specificity': specificity,
            'accuracy': accuracy,
            'support': int(np.sum(class_mask))
        }
    
    return metrics

# 3. Confusion Matrix Visualization
def plot_confusion_matrix(targets, predictions, class_names=None, normalize=False):
    """Plot confusion matrix"""
    cm = confusion_matrix(targets, predictions)
    
    if normalize:
        cm = cm.astype('float') / cm.sum(axis=1)[:, np.newaxis]
        fmt = '.2f'
    else:
        fmt = 'd'
    
    plt.figure(figsize=(10, 8))
    sns.heatmap(cm, annot=True, fmt=fmt, cmap='Blues',
                xticklabels=class_names, yticklabels=class_names)
    plt.title('Confusion Matrix')
    plt.xlabel('Predicted Label')
    plt.ylabel('True Label')
    plt.show()
    
    return cm

# 4. ROC Curve and AUC
def calculate_roc_auc(probabilities, targets, num_classes):
    """Calculate ROC AUC for multi-class classification"""
    from sklearn.metrics import roc_curve, auc
    from sklearn.preprocessing import label_binarize
    
    # Binarize the output
    y_true = label_binarize(targets, classes=range(num_classes))
    
    if num_classes == 2:
        fpr, tpr, _ = roc_curve(y_true, probabilities[:, 1])
        roc_auc = auc(fpr, tpr)
        return {'fpr': fpr, 'tpr': tpr, 'auc': roc_auc}
    else:
        # Multi-class ROC AUC
        fpr = dict()
        tpr = dict()
        roc_auc = dict()
        
        for i in range(num_classes):
            fpr[i], tpr[i], _ = roc_curve(y_true[:, i], probabilities[:, i])
            roc_auc[i] = auc(fpr[i], tpr[i])
        
        # Compute micro-average ROC curve and ROC area
        fpr["micro"], tpr["micro"], _ = roc_curve(y_true.ravel(), probabilities.ravel())
        roc_auc["micro"] = auc(fpr["micro"], tpr["micro"])
        
        return {'fpr': fpr, 'tpr': tpr, 'auc': roc_auc}

def plot_roc_curves(roc_data, class_names=None):
    """Plot ROC curves"""
    plt.figure(figsize=(10, 8))
    
    if isinstance(roc_data['auc'], dict):
        # Multi-class
        for i in roc_data['auc']:
            if i != "micro":
                plt.plot(roc_data['fpr'][i], roc_data['tpr'][i],
                        label=f'Class {i} (AUC = {roc_data["auc"][i]:.2f})')
        
        plt.plot(roc_data['fpr']["micro"], roc_data['tpr']["micro"],
                label=f'Micro-average (AUC = {roc_data["auc"]["micro"]:.2f})',
                linestyle='--')
    else:
        # Binary classification
        plt.plot(roc_data['fpr'], roc_data['tpr'],
                label=f'ROC curve (AUC = {roc_data["auc"]:.2f})')
    
    plt.plot([0, 1], [0, 1], 'k--', label='Random')
    plt.xlim([0.0, 1.0])
    plt.ylim([0.0, 1.05])
    plt.xlabel('False Positive Rate')
    plt.ylabel('True Positive Rate')
    plt.title('Receiver Operating Characteristic')
    plt.legend(loc="lower right")
    plt.show()

# 5. Precision-Recall Curve
def calculate_precision_recall(probabilities, targets, num_classes):
    """Calculate precision-recall curves"""
    from sklearn.metrics import precision_recall_curve, average_precision_score
    from sklearn.preprocessing import label_binarize
    
    y_true = label_binarize(targets, classes=range(num_classes))
    
    if num_classes == 2:
        precision, recall, _ = precision_recall_curve(y_true, probabilities[:, 1])
        avg_precision = average_precision_score(y_true, probabilities[:, 1])
        return {'precision': precision, 'recall': recall, 'avg_precision': avg_precision}
    else:
        precision = dict()
        recall = dict()
        avg_precision = dict()
        
        for i in range(num_classes):
            precision[i], recall[i], _ = precision_recall_curve(y_true[:, i], probabilities[:, i])
            avg_precision[i] = average_precision_score(y_true[:, i], probabilities[:, i])
        
        return {'precision': precision, 'recall': recall, 'avg_precision': avg_precision}

def plot_precision_recall_curves(pr_data, class_names=None):
    """Plot precision-recall curves"""
    plt.figure(figsize=(10, 8))
    
    if isinstance(pr_data['avg_precision'], dict):
        # Multi-class
        for i in pr_data['avg_precision']:
            plt.plot(pr_data['recall'][i], pr_data['precision'][i],
                    label=f'Class {i} (AP = {pr_data["avg_precision"][i]:.2f})')
    else:
        # Binary classification
        plt.plot(pr_data['recall'], pr_data['precision'],
                label=f'PR curve (AP = {pr_data["avg_precision"]:.2f})')
    
    plt.xlabel('Recall')
    plt.ylabel('Precision')
    plt.title('Precision-Recall Curve')
    plt.legend(loc="lower left")
    plt.grid(True)
    plt.show()

# 6. Model Calibration Analysis
def evaluate_calibration(probabilities, targets, n_bins=10):
    """Evaluate model calibration"""
    if probabilities.shape[1] == 2:
        confidences = probabilities[:, 1]
    else:
        confidences = np.max(probabilities, axis=1)
    
    # Calculate accuracy in each bin
    bin_boundaries = np.linspace(0, 1, n_bins + 1)
    bin_lowers = bin_boundaries[:-1]
    bin_uppers = bin_boundaries[1:]
    
    accuracies = []
    confidences_in_bins = []
    prop_samples = []
    
    for bin_lower, bin_upper in zip(bin_lowers, bin_uppers):
        # Find samples in this bin
        in_bin = (confidences > bin_lower) & (confidences <= bin_upper)
        prop_in_bin = np.mean(in_bin)
        
        if prop_in_bin > 0:
            accuracy_in_bin = np.mean(targets[in_bin] == np.argmax(probabilities[in_bin], axis=1))
            avg_confidence_in_bin = np.mean(confidences[in_bin])
        else:
            accuracy_in_bin = 0
            avg_confidence_in_bin = 0
        
        accuracies.append(accuracy_in_bin)
        confidences_in_bins.append(avg_confidence_in_bin)
        prop_samples.append(prop_in_bin)
    
    # Calculate Expected Calibration Error (ECE)
    ece = np.sum(np.array(prop_samples) * 
                np.abs(np.array(accuracies) - np.array(confidences_in_bins)))
    
    return {
        'accuracies': accuracies,
        'confidences': confidences_in_bins,
        'proportions': prop_samples,
        'ece': ece
    }

def plot_reliability_diagram(calibration_data):
    """Plot reliability diagram"""
    plt.figure(figsize=(8, 8))
    plt.plot([0, 1], [0, 1], 'k--', label='Perfectly calibrated')
    plt.plot(calibration_data['confidences'], calibration_data['accuracies'], 'o-', 
             label=f'Model (ECE={calibration_data["ece"]:.3f})')
    
    # Add histogram
    plt.bar(np.linspace(0, 1, len(calibration_data['proportions'])),
            calibration_data['proportions'], width=0.1, alpha=0.3, 
            label='Fraction of samples')
    
    plt.xlabel('Confidence')
    plt.ylabel('Accuracy')
    plt.title('Reliability Diagram')
    plt.legend()
    plt.grid(True)
    plt.show()

# 7. Feature Importance Analysis
def analyze_feature_importance(model, input_tensor, target_class=None, method='gradient'):
    """Analyze feature importance using different methods"""
    model.eval()
    input_tensor.requires_grad_()
    
    if method == 'gradient':
        # Gradient-based importance
        output = model(input_tensor)
        if target_class is None:
            target_class = output.argmax(dim=1).item()
        
        model.zero_grad()
        output[0, target_class].backward()
        
        # Calculate importance as gradient magnitude
        importance = torch.abs(input_tensor.grad)
        return importance
    
    elif method == 'integrated_gradients':
        # Integrated gradients
        baseline = torch.zeros_like(input_tensor)
        alphas = torch.linspace(0, 1, 50).to(input_tensor.device)
        
        gradients = []
        for alpha in alphas:
            interpolated = baseline + alpha * (input_tensor - baseline)
            interpolated.requires_grad_()
            
            output = model(interpolated)
            if target_class is None:
                target_class = output.argmax(dim=1).item()
            
            model.zero_grad()
            output[0, target_class].backward()
            gradients.append(interpolated.grad)
        
        gradients = torch.stack(gradients)
        integrated_grads = (input_tensor - baseline) * torch.mean(gradients, dim=0)
        return torch.abs(integrated_grads)
    
    elif method == 'occlusion':
        # Occlusion sensitivity
        original_output = model(input_tensor)
        if target_class is None:
            target_class = original_output.argmax(dim=1).item()
        
        importance_map = torch.zeros_like(input_tensor)
        patch_size = 8
        
        for i in range(0, input_tensor.shape[2], patch_size):
            for j in range(0, input_tensor.shape[3], patch_size):
                # Create occluded input
                occluded_input = input_tensor.clone()
                occluded_input[:, :, i:i+patch_size, j:j+patch_size] = 0
                
                # Calculate change in output
                occluded_output = model(occluded_input)
                change = original_output[0, target_class] - occluded_output[0, target_class]
                importance_map[:, :, i:i+patch_size, j:j+patch_size] = change
        
        return importance_map

# 8. Model Performance Report
class ModelPerformanceReport:
    def __init__(self, evaluator_results, class_names=None):
        self.results = evaluator_results
        self.class_names = class_names
    
    def generate_report(self):
        """Generate comprehensive performance report"""
        report = []
        report.append("=== MODEL PERFORMANCE REPORT ===\n")
        
        # Overall metrics
        report.append("OVERALL METRICS:")
        report.append(f"  Accuracy: {self.results['accuracy']:.4f}")
        report.append(f"  Precision: {self.results['precision']:.4f}")
        report.append(f"  Recall: {self.results['recall']:.4f}")
        report.append(f"  F1-Score: {self.results['f1_score']:.4f}")
        if 'loss' in self.results:
            report.append(f"  Loss: {self.results['loss']:.4f}")
        report.append("")
        
        # Per-class metrics
        if 'targets' in self.results and 'predictions' in self.results:
            per_class_metrics = calculate_per_class_metrics(
                self.results['targets'], 
                self.results['predictions'], 
                self.class_names
            )
            report.append("PER-CLASS METRICS:")
            for class_name, metrics in per_class_metrics.items():
                report.append(f"  {class_name}:")
                report.append(f"    Precision: {metrics['precision']:.4f}")
                report.append(f"    Recall: {metrics['recall']:.4f}")
                report.append(f"    F1-Score: {metrics['f1_score']:.4f}")
                report.append(f"    Support: {metrics['support']}")
            report.append("")
        
        # Additional analysis
        if 'probabilities' in self.results:
            report.append("PROBABILITY ANALYSIS:")
            probs = np.array(self.results['probabilities'])
            max_probs = np.max(probs, axis=1)
            report.append(f"  Mean Max Probability: {np.mean(max_probs):.4f}")
            report.append(f"  Std Max Probability: {np.std(max_probs):.4f}")
            report.append(f"  Min Max Probability: {np.min(max_probs):.4f}")
            report.append(f"  Max Max Probability: {np.max(max_probs):.4f}")
        
        return "\n".join(report)
    
    def plot_metrics(self):
        """Plot various metrics"""
        fig, axes = plt.subplots(2, 2, figsize=(15, 12))
        
        # Confusion Matrix
        if 'targets' in self.results and 'predictions' in self.results:
            plot_confusion_matrix(
                self.results['targets'], 
                self.results['predictions'], 
                self.class_names,
                ax=axes[0, 0]
            )
            axes[0, 0].set_title('Confusion Matrix')
        
        # ROC Curve
        if 'probabilities' in self.results and 'targets' in self.results:
            num_classes = len(np.unique(self.results['targets']))
            roc_data = calculate_roc_auc(
                np.array(self.results['probabilities']), 
                np.array(self.results['targets']), 
                num_classes
            )
            plot_roc_curves(roc_data, self.class_names)
            axes[0, 1].set_title('ROC Curves')
        
        # Precision-Recall Curve
        if 'probabilities' in self.results and 'targets' in self.results:
            num_classes = len(np.unique(self.results['targets']))
            pr_data = calculate_precision_recall(
                np.array(self.results['probabilities']), 
                np.array(self.results['targets']), 
                num_classes
            )
            plot_precision_recall_curves(pr_data, self.class_names)
            axes[1, 0].set_title('Precision-Recall Curves')
        
        # Calibration
        if 'probabilities' in self.results and 'targets' in self.results:
            calibration_data = evaluate_calibration(
                np.array(self.results['probabilities']), 
                np.array(self.results['targets'])
            )
            plot_reliability_diagram(calibration_data)
            axes[1, 1].set_title('Reliability Diagram')
        
        plt.tight_layout()
        plt.show()

# 9. Cross-Validation for CNNs
def cross_validate_cnn(model_class, dataset, k_folds=5, batch_size=32, epochs=10):
    """Perform k-fold cross-validation"""
    from torch.utils.data import Subset, DataLoader
    from sklearn.model_selection import KFold
    
    kfold = KFold(n_splits=k_folds, shuffle=True, random_state=42)
    fold_results = []
    
    for fold, (train_idx, val_idx) in enumerate(kfold.split(dataset)):
        print(f"Fold {fold + 1}/{k_folds}")
        
        # Create data loaders
        train_subset = Subset(dataset, train_idx)
        val_subset = Subset(dataset, val_idx)
        
        train_loader = DataLoader(train_subset, batch_size=batch_size, shuffle=True)
        val_loader = DataLoader(val_subset, batch_size=batch_size, shuffle=False)
        
        # Create model
        model = model_class()
        device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        model.to(device)
        
        # Train model (simplified training)
        optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
        criterion = nn.CrossEntropyLoss()
        
        for epoch in range(epochs):
            model.train()
            for inputs, targets in train_loader:
                inputs, targets = inputs.to(device), targets.to(device)
                optimizer.zero_grad()
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                loss.backward()
                optimizer.step()
        
        # Evaluate
        evaluator = CNNEvaluator(model, device)
        results = evaluator.evaluate(val_loader, criterion)
        fold_results.append(results)
        
        print(f"Fold {fold + 1} Accuracy: {results['accuracy']:.4f}")
    
    # Calculate average metrics
    avg_metrics = {}
    for key in fold_results[0].keys():
        if key not in ['predictions', 'targets', 'probabilities']:
            avg_metrics[key] = np.mean([fold[key] for fold in fold_results])
    
    print(f"\nCross-Validation Results:")
    for key, value in avg_metrics.items():
        std_value = np.std([fold[key] for fold in fold_results])
        print(f"{key}: {value:.4f} ± {std_value:.4f}")
    
    return fold_results, avg_metrics

# 10. Usage Example
def demonstrate_evaluation():
    """Demonstrate CNN evaluation techniques"""
    # Create sample data
    model = SimpleCNN(num_classes=10)
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    
    # Create sample inputs and targets
    sample_inputs = torch.randn(100, 3, 32, 32)
    sample_targets = torch.randint(0, 10, (100,))
    
    # Mock dataloader
    from torch.utils.data import TensorDataset, DataLoader
    dataset = TensorDataset(sample_inputs, sample_targets)
    dataloader = DataLoader(dataset, batch_size=32, shuffle=False)
    
    # Evaluate model
    evaluator = CNNEvaluator(model, device)
    results = evaluator.evaluate(dataloader, nn.CrossEntropyLoss())
    
    # Generate report
    report_generator = ModelPerformanceReport(results, class_names=[f"Class_{i}" for i in range(10)])
    report = report_generator.generate_report()
    print(report)
    
    print("Evaluation techniques demonstrated:")
    print("- Comprehensive model evaluation")
    print("- Per-class metrics calculation")
    print("- Confusion matrix visualization")
    print("- ROC and Precision-Recall curves")
    print("- Model calibration analysis")
    print("- Feature importance analysis")
    print("- Performance reporting")
    print("- Cross-validation framework")

# Run demonstration
demonstrate_evaluation()