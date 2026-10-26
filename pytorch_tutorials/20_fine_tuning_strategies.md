# 20. Fine-tuning Strategies

## 问题与解答

### 1. 什么是微调（Fine-tuning）？
微调是在预训练模型的基础上，使用特定任务的数据对模型进行进一步训练的过程。它允许模型适应新任务的特征和要求。

### 2. 微调与从头训练的主要区别是什么？
1. 起始点：微调从预训练权重开始，从头训练从随机权重开始
2. 训练时间：微调通常需要更少的训练时间
3. 数据需求：微调可以在较小的数据集上工作
4. 收敛速度：微调通常收敛更快

### 3. 微调的基本步骤是什么？
1. 选择合适的预训练模型
2. 根据新任务调整模型架构（如修改分类器层）
3. 冻结部分或全部预训练层
4. 使用新数据训练模型
5. 逐步解冻更多层进行进一步微调

### 4. 如何在PyTorch中实现基本微调？
```python
import torch
import torch.nn as nn
import torchvision.models as models

# 加载预训练模型
model = models.resnet50(pretrained=True)

# 修改分类器层以适应新任务
num_classes = 10  # 新任务的类别数
model.fc = nn.Linear(model.fc.in_features, num_classes)

# 冻结所有预训练层
for param in model.parameters():
    param.requires_grad = False

# 只训练新添加的分类器层
for param in model.fc.parameters():
    param.requires_grad = True

# 定义优化器（只优化需要训练的参数）
optimizer = torch.optim.Adam(filter(lambda p: p.requires_grad, model.parameters()), lr=0.001)
```

### 5. 什么是分层微调（Layer-wise Fine-tuning）？
分层微调是逐步解冻和训练模型层的策略，通常从最后几层开始，逐渐向前解冻更多层。

### 6. 如何实现分层微调？
```python
def layer_wise_finetuning(model, dataloaders, num_epochs_per_stage=5):
    # 阶段1：只训练分类器层
    print("Stage 1: Training classifier layer only")
    for param in model.parameters():
        param.requires_grad = False
    for param in model.fc.parameters():  # 假设fc是分类器层
        param.requires_grad = True
    
    optimizer = torch.optim.Adam(filter(lambda p: p.requires_grad, model.parameters()), lr=1e-3)
    train_model(model, dataloaders, optimizer, num_epochs_per_stage)
    
    # 阶段2：训练最后几层
    print("Stage 2: Training last layers")
    for param in model.layer4.parameters():  # ResNet的最后一层组
        param.requires_grad = True
    
    optimizer = torch.optim.Adam(filter(lambda p: p.requires_grad, model.parameters()), lr=1e-4)
    train_model(model, dataloaders, optimizer, num_epochs_per_stage)
    
    # 阶段3：训练所有层（使用更小的学习率）
    print("Stage 3: Training all layers")
    for param in model.parameters():
        param.requires_grad = True
    
    optimizer = torch.optim.Adam(model.parameters(), lr=1e-5)
    train_model(model, dataloaders, optimizer, num_epochs_per_stage)
```

### 7. 如何为不同层设置不同的学习率？
```python
# 为不同层组设置不同的学习率
optimizer = torch.optim.SGD([
    {'params': model.features.parameters(), 'lr': 1e-5},      # 特征提取层
    {'params': model.layer4.parameters(), 'lr': 1e-4},        # 倒数第二层组
    {'params': model.fc.parameters(), 'lr': 1e-3}             # 分类器层
], momentum=0.9)
```

### 8. 什么是差分学习率（Discriminative Learning Rates）？
差分学习率是一种微调策略，为模型的不同层设置不同的学习率，通常前面的层使用较小的学习率，后面的层使用较大的学习率。

### 9. 如何实现差分学习率？
```python
def get_discriminative_lr_params(model, lr_mult=0.1):
    """
    为模型的不同层设置差分学习率
    """
    layer_groups = []
    
    # 获取模型的所有层
    layers = list(model.children())
    
    # 为每组层设置递增的学习率
    for i, layer in enumerate(layers):
        lr = 1e-5 * (lr_mult ** (len(layers) - i - 1))
        layer_groups.append({'params': layer.parameters(), 'lr': lr})
    
    return layer_groups

# 使用差分学习率
param_groups = get_discriminative_lr_params(model)
optimizer = torch.optim.Adam(param_groups)
```

### 10. 什么是学习率预热（Learning Rate Warmup）？
学习率预热是在训练开始时使用较小的学习率，然后逐渐增加到目标学习率，有助于稳定训练过程。

### 11. 如何实现学习率预热？
```python
class WarmupScheduler:
    def __init__(self, optimizer, warmup_steps, initial_lr=1e-6):
        self.optimizer = optimizer
        self.warmup_steps = warmup_steps
        self.initial_lr = initial_lr
        self.step_count = 0
        
    def step(self, target_lr):
        self.step_count += 1
        if self.step_count <= self.warmup_steps:
            # 线性预热
            lr = self.initial_lr + (target_lr - self.initial_lr) * (self.step_count / self.warmup_steps)
        else:
            lr = target_lr
            
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr

# 使用示例
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
scheduler = WarmupScheduler(optimizer, warmup_steps=1000, initial_lr=1e-6)

for epoch in range(num_epochs):
    for batch in dataloader:
        # 训练步骤
        loss = train_step(model, batch)
        
        # 更新学习率
        scheduler.step(target_lr=1e-3)
```

### 12. 什么是循环学习率（Cyclical Learning Rates）？
循环学习率是在训练过程中周期性地改变学习率，从较低值到较高值再回到较低值，有助于跳出局部最优。

### 13. 如何实现循环学习率？
```python
class CyclicalLR:
    def __init__(self, optimizer, base_lr=1e-5, max_lr=1e-3, step_size=1000):
        self.optimizer = optimizer
        self.base_lr = base_lr
        self.max_lr = max_lr
        self.step_size = step_size
        self.step_count = 0
        
    def step(self):
        self.step_count += 1
        cycle = np.floor(1 + self.step_count / (2 * self.step_size))
        x = np.abs(self.step_count / self.step_size - 2 * cycle + 1)
        lr = self.base_lr + (self.max_lr - self.base_lr) * np.maximum(0, (1 - x))
        
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr

# 使用示例
optimizer = torch.optim.Adam(model.parameters())
scheduler = CyclicalLR(optimizer, base_lr=1e-5, max_lr=1e-3, step_size=1000)
```

### 14. 什么是渐进式调整大小（Progressive Resizing）？
渐进式调整大小是一种训练策略，从较小的图像尺寸开始训练，然后逐渐增加到目标尺寸，可以加快训练速度并提高最终性能。

### 15. 如何实现渐进式调整大小？
```python
class ProgressiveResize:
    def __init__(self, initial_size=128, target_size=224, steps=4):
        self.sizes = np.linspace(initial_size, target_size, steps).astype(int)
        self.current_step = 0
        
    def get_current_size(self):
        return self.sizes[min(self.current_step, len(self.sizes) - 1)]
    
    def step(self):
        if self.current_step < len(self.sizes) - 1:
            self.current_step += 1

# 在数据加载器中使用
def get_dataloader(image_size):
    transform = transforms.Compose([
        transforms.Resize((image_size, image_size)),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
    ])
    return DataLoader(dataset, batch_size=32, transform=transform)

# 训练循环
progressive_resize = ProgressiveResize(initial_size=128, target_size=224, steps=4)

for epoch in range(num_epochs):
    image_size = progressive_resize.get_current_size()
    dataloader = get_dataloader(image_size)
    
    train_epoch(model, dataloader)
    
    # 每隔几个epoch调整一次尺寸
    if epoch % 10 == 9:
        progressive_resize.step()
```

### 16. 什么是标签平滑（Label Smoothing）？
标签平滑是一种正则化技术，通过将硬标签（0或1）替换为软标签来减少模型过拟合的风险。

### 17. 如何实现标签平滑？
```python
class LabelSmoothingLoss(nn.Module):
    def __init__(self, smoothing=0.1, num_classes=1000):
        super(LabelSmoothingLoss, self).__init__()
        self.smoothing = smoothing
        self.num_classes = num_classes
        
    def forward(self, pred, target):
        # 创建平滑的标签
        true_dist = torch.zeros_like(pred)
        true_dist.fill_(self.smoothing / (self.num_classes - 1))
        true_dist.scatter_(1, target.unsqueeze(1), 1.0 - self.smoothing)
        
        # 计算交叉熵损失
        return torch.mean(torch.sum(-true_dist * F.log_softmax(pred, dim=1), dim=1))

# 使用标签平滑损失
criterion = LabelSmoothingLoss(smoothing=0.1, num_classes=10)
```

### 18. 什么是混合精度训练（Mixed Precision Training）？
混合精度训练使用16位浮点数（FP16）进行前向和反向传播，同时保持32位主副本用于参数更新，可以减少内存使用并加快训练速度。

### 19. 如何在PyTorch中实现混合精度训练？
```python
from torch.cuda.amp import autocast, GradScaler

# 创建梯度缩放器
scaler = GradScaler()

def train_step_mixed_precision(model, data, target, optimizer, criterion):
    optimizer.zero_grad()
    
    # 使用自动混合精度
    with autocast():
        output = model(data)
        loss = criterion(output, target)
    
    # 缩放损失并反向传播
    scaler.scale(loss).backward()
    
    # 更新参数
    scaler.step(optimizer)
    scaler.update()
    
    return loss
```

### 20. 什么是早停（Early Stopping）？
早停是一种防止过拟合的技术，当验证集性能停止改善时停止训练。

### 21. 如何实现早停？
```python
class EarlyStopping:
    def __init__(self, patience=7, min_delta=0):
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_loss = None
        self.early_stop = False
        
    def __call__(self, val_loss):
        if self.best_loss is None:
            self.best_loss = val_loss
        elif val_loss < self.best_loss - self.min_delta:
            self.best_loss = val_loss
            self.counter = 0
        else:
            self.counter += 1
            if self.counter >= self.patience:
                self.early_stop = True

# 使用示例
early_stopping = EarlyStopping(patience=10, min_delta=0.001)

for epoch in range(num_epochs):
    train_loss = train_epoch(model, train_loader)
    val_loss = validate_epoch(model, val_loader)
    
    early_stopping(val_loss)
    if early_stopping.early_stop:
        print("Early stopping")
        break
```

### 22. 什么是模型集成（Model Ensembling）？
模型集成是将多个模型的预测结果组合起来以提高整体性能的技术。

### 23. 如何实现模型集成？
```python
class ModelEnsemble(nn.Module):
    def __init__(self, models):
        super(ModelEnsemble, self).__init__()
        self.models = nn.ModuleList(models)
        
    def forward(self, x):
        outputs = []
        for model in self.models:
            outputs.append(F.softmax(model(x), dim=1))
        
        # 平均预测结果
        ensemble_output = torch.stack(outputs).mean(dim=0)
        return ensemble_output

# 创建集成模型
models = [model1, model2, model3]
ensemble = ModelEnsemble(models)
```

### 24. 什么是TTA（Test Time Augmentation）？
TTA是在测试时对输入数据进行多种变换，对每个变换后的版本进行预测，然后平均结果以提高预测准确性。

### 25. 如何实现TTA？
```python
def tta_predict(model, image, transforms_list):
    predictions = []
    
    for transform in transforms_list:
        # 应用变换
        transformed_image = transform(image)
        transformed_image = transformed_image.unsqueeze(0).to(device)
        
        # 预测
        with torch.no_grad():
            output = model(transformed_image)
            predictions.append(F.softmax(output, dim=1))
    
    # 平均预测结果
    avg_prediction = torch.stack(predictions).mean(dim=0)
    return avg_prediction

# 使用示例
transforms_list = [
    transforms.Compose([transforms.ToTensor()]),
    transforms.Compose([transforms.RandomHorizontalFlip(p=1.0), transforms.ToTensor()]),
    transforms.Compose([transforms.RandomRotation(10), transforms.ToTensor()]),
]

prediction = tta_predict(model, image, transforms_list)
```

### 26. 什么是知识蒸馏微调？
知识蒸馏微调是使用大模型（教师模型）指导小模型（学生模型）训练的过程，以保持性能的同时减少模型大小。

### 27. 如何实现知识蒸馏微调？
```python
def distillation_loss(student_outputs, teacher_outputs, true_labels, 
                     temperature=3.0, alpha=0.7):
    # 软标签损失
    soft_loss = nn.KLDivLoss(reduction='batchmean')(
        F.log_softmax(student_outputs / temperature, dim=1),
        F.softmax(teacher_outputs / temperature, dim=1)
    ) * (temperature ** 2)
    
    # 硬标签损失
    hard_loss = F.cross_entropy(student_outputs, true_labels)
    
    return alpha * soft_loss + (1 - alpha) * hard_loss

# 训练循环
for data, target in dataloader:
    student_outputs = student_model(data)
    with torch.no_grad():
        teacher_outputs = teacher_model(data)
    
    loss = distillation_loss(student_outputs, teacher_outputs, target)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

### 28. 如何选择合适的微调策略？
1. 考虑数据集大小：小数据集适合冻结更多层
2. 考虑任务相似性：相似任务可以解冻更多层
3. 考虑计算资源：资源有限时使用更保守的策略
4. 实验验证：通过验证集性能选择最佳策略

### 29. 微调过程中的常见问题有哪些？
1. 过拟合：特别是在小数据集上
2. 梯度消失：深层网络中可能出现
3. 学习率选择不当：过大或过小都会影响性能
4. 层冻结策略不当：可能导致性能不佳

### 30. 如何调试微调过程？
1. 监控训练和验证损失曲线
2. 可视化梯度流和激活值
3. 分析不同层的学习进度
4. 使用学习率查找器找到合适的学习率
5. 检查模型权重的变化情况