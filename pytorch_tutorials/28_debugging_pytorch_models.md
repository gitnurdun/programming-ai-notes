# 28. Debugging PyTorch Models

## 问题与解答

### 1. PyTorch模型调试的主要挑战有哪些？
PyTorch模型调试的主要挑战包括：
1. 动态图的不确定性
2. GPU内存管理和调试困难
3. 梯度消失或爆炸
4. 数值不稳定性和NaN/Inf值
5. 数据预处理错误
6. 模型架构设计问题
7. 超参数设置不当
8. 并行训练中的同步问题

### 2. 如何使用PyTorch内置的调试工具？
```python
import torch
import torch.nn as nn
import torch.autograd as autograd

def debug_autograd_graph():
    """
    调试自动梯度图
    """
    # 启用异常检测
    autograd.set_detect_anomaly(True)
    
    # 检查梯度
    x = torch.randn(2, 3, requires_grad=True)
    y = x * 2
    z = y.mean()
    
    # 反向传播
    z.backward()
    
    print(f"x的梯度: {x.grad}")
    
    # 关闭异常检测
    autograd.set_detect_anomaly(False)

def check_tensor_properties(tensor, name="tensor"):
    """
    检查张量属性
    """
    print(f"{name}形状: {tensor.shape}")
    print(f"{name}数据类型: {tensor.dtype}")
    print(f"{name}设备: {tensor.device}")
    print(f"{name}是否需要梯度: {tensor.requires_grad}")
    print(f"{name}是否有梯度: {tensor.grad is not None}")
    if tensor.grad is not None:
        print(f"{name}梯度形状: {tensor.grad.shape}")
        print(f"{name}梯度范数: {tensor.grad.norm().item()}")
    
    # 检查NaN和Inf值
    has_nan = torch.isnan(tensor).any().item()
    has_inf = torch.isinf(tensor).any().item()
    print(f"{name}包含NaN: {has_nan}")
    print(f"{name}包含Inf: {has_inf}")

# 使用示例
x = torch.randn(3, 4, requires_grad=True)
check_tensor_properties(x, "输入张量")
```

### 3. 如何调试梯度问题？
```python
def debug_gradients(model, inputs, targets, criterion):
    """
    调试模型梯度
    """
    model.train()
    
    # 前向传播
    outputs = model(inputs)
    loss = criterion(outputs, targets)
    
    # 检查损失
    print(f"损失值: {loss.item()}")
    print(f"损失是否包含NaN: {torch.isnan(loss).item()}")
    print(f"损失是否包含Inf: {torch.isinf(loss).item()}")
    
    # 反向传播
    loss.backward()
    
    # 检查参数梯度
    total_norm = 0
    param_count = 0
    
    for name, param in model.named_parameters():
        if param.grad is not None:
            grad_norm = param.grad.norm().item()
            total_norm += grad_norm ** 2
            param_count += 1
            
            print(f"{name}梯度范数: {grad_norm:.6f}")
            
            # 检查梯度中的异常值
            if torch.isnan(param.grad).any():
                print(f"警告: {name}梯度包含NaN值")
            if torch.isinf(param.grad).any():
                print(f"警告: {name}梯度包含Inf值")
            
            # 检查梯度是否为零
            if grad_norm < 1e-8:
                print(f"警告: {name}梯度接近零 ({grad_norm:.2e})")
        else:
            print(f"{name}无梯度")
    
    total_norm = total_norm ** 0.5
    print(f"总梯度范数: {total_norm:.6f}")
    
    return total_norm

def gradient_clipping(model, max_norm=1.0):
    """
    梯度裁剪
    """
    total_norm = torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm)
    print(f"梯度裁剪后的范数: {total_norm:.6f}")
    return total_norm

# 使用示例
model = nn.Linear(10, 1)
inputs = torch.randn(32, 10)
targets = torch.randn(32, 1)
criterion = nn.MSELoss()

debug_gradients(model, inputs, targets, criterion)
```

### 4. 如何调试模型中的NaN和Inf值？
```python
def detect_nan_inf(model, inputs):
    """
    检测模型中的NaN和Inf值
    """
    model.eval()
    
    def check_module(module, module_name):
        # 检查模块参数
        for name, param in module.named_parameters():
            full_name = f"{module_name}.{name}"
            if torch.isnan(param).any():
                print(f"警告: {full_name} 包含NaN参数")
            if torch.isinf(param).any():
                print(f"警告: {full_name} 包含Inf参数")
    
    # 注册前向钩子检查输出
    def forward_hook(module, input, output, module_name):
        if isinstance(output, torch.Tensor):
            if torch.isnan(output).any():
                print(f"警告: {module_name} 输出包含NaN值")
            if torch.isinf(output).any():
                print(f"警告: {module_name} 输出包含Inf值")
        elif isinstance(output, (tuple, list)):
            for i, out in enumerate(output):
                if isinstance(out, torch.Tensor):
                    if torch.isnan(out).any():
                        print(f"警告: {module_name} 输出[{i}] 包含NaN值")
                    if torch.isinf(out).any():
                        print(f"警告: {module_name} 输出[{i}] 包含Inf值")
    
    # 为所有模块注册钩子
    hooks = []
    for name, module in model.named_modules():
        check_module(module, name)
        hook = module.register_forward_hook(
            lambda module, input, output, name=name: forward_hook(module, input, output, name)
        )
        hooks.append(hook)
    
    # 执行前向传播
    try:
        with torch.no_grad():
            output = model(inputs)
    finally:
        # 移除钩子
        for hook in hooks:
            hook.remove()
    
    return output

def replace_nan_inf(tensor, replacement=0.0):
    """
    替换张量中的NaN和Inf值
    """
    tensor = torch.where(torch.isnan(tensor), torch.full_like(tensor, replacement), tensor)
    tensor = torch.where(torch.isinf(tensor), torch.full_like(tensor, replacement), tensor)
    return tensor

# 使用示例
model = nn.Sequential(
    nn.Linear(10, 20),
    nn.ReLU(),
    nn.Linear(20, 1)
)

inputs = torch.randn(5, 10)
# 故意引入一些问题
inputs[0, 0] = float('nan')

detect_nan_inf(model, inputs)
```

### 5. 如何使用PyTorch Profiler进行调试？
```python
import torch.profiler

def profile_model_debug(model, inputs, num_steps=10):
    """
    使用Profiler调试模型性能和问题
    """
    model.eval()
    
    with torch.profiler.profile(
        activities=[
            torch.profiler.ProfilerActivity.CPU,
            torch.profiler.ProfilerActivity.CUDA
        ],
        schedule=torch.profiler.schedule(wait=1, warmup=1, active=3),
        on_trace_ready=torch.profiler.tensorboard_trace_handler('./debug_logs'),
        record_shapes=True,
        profile_memory=True,
        with_stack=True,
        with_flops=True
    ) as prof:
        for step in range(num_steps):
            with torch.no_grad():
                output = model(inputs)
                prof.step()
            
            # 检查输出
            if torch.isnan(output).any() or torch.isinf(output).any():
                print(f"步骤 {step}: 输出包含NaN或Inf值")
    
    # 打印详细统计
    print("性能统计 (按CPU时间排序):")
    print(prof.key_averages().table(sort_by="cpu_time_total", row_limit=10))
    
    if torch.cuda.is_available():
        print("\nGPU统计 (按CUDA时间排序):")
        print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=10))
    
    return prof

# 使用示例
# profiler = profile_model_debug(model, inputs)
```

### 6. 如何调试数据加载问题？
```python
def debug_dataloader(dataloader, num_batches=5):
    """
    调试数据加载器
    """
    print(f"数据加载器信息:")
    print(f"  批大小: {dataloader.batch_size}")
    print(f"  数据集大小: {len(dataloader.dataset)}")
    print(f"  批数量: {len(dataloader)}")
    print(f"  工作进程数: {dataloader.num_workers}")
    
    for batch_idx, batch in enumerate(dataloader):
        if batch_idx >= num_batches:
            break
            
        print(f"\n批次 {batch_idx}:")
        
        if isinstance(batch, (list, tuple)):
            for i, item in enumerate(batch):
                if isinstance(item, torch.Tensor):
                    print(f"  项目 {i}:")
                    print(f"    形状: {item.shape}")
                    print(f"    数据类型: {item.dtype}")
                    print(f"    设备: {item.device}")
                    print(f"    范围: [{item.min().item():.4f}, {item.max().item():.4f}]")
                    print(f"    包含NaN: {torch.isnan(item).any().item()}")
                    print(f"    包含Inf: {torch.isinf(item).any().item()}")
                else:
                    print(f"  项目 {i}: {type(item)}")
        elif isinstance(batch, torch.Tensor):
            print(f"  形状: {batch.shape}")
            print(f"  数据类型: {batch.dtype}")
            print(f"  设备: {batch.device}")
            print(f"  范围: [{batch.min().item():.4f}, {batch.max().item():.4f}]")
            print(f"  包含NaN: {torch.isnan(batch).any().item()}")
            print(f"  包含Inf: {torch.isinf(batch).any().item()}")
        
        # 检查批次大小一致性
        if isinstance(batch, (list, tuple)) and len(batch) > 0:
            if isinstance(batch[0], torch.Tensor):
                expected_batch_size = dataloader.batch_size
                actual_batch_size = batch[0].shape[0]
                if actual_batch_size != expected_batch_size:
                    print(f"  警告: 批大小不一致，期望 {expected_batch_size}，实际 {actual_batch_size}")

def create_debug_dataset(dataset, max_samples=100):
    """
    创建调试用的小数据集
    """
    class DebugDataset(torch.utils.data.Dataset):
        def __init__(self, original_dataset, max_samples):
            self.original_dataset = original_dataset
            self.max_samples = min(max_samples, len(original_dataset))
        
        def __len__(self):
            return self.max_samples
        
        def __getitem__(self, idx):
            if idx >= self.max_samples:
                raise IndexError("索引超出调试数据集范围")
            return self.original_dataset[idx]
    
    return DebugDataset(dataset, max_samples)

# 使用示例
# debug_dataloader(train_loader)
```

### 7. 如何调试模型训练过程？
```python
class TrainingDebugger:
    def __init__(self, model, optimizer, criterion):
        self.model = model
        self.optimizer = optimizer
        self.criterion = criterion
        self.step_count = 0
        self.loss_history = []
        self.grad_norm_history = []
    
    def debug_step(self, inputs, targets):
        """
        调试单步训练
        """
        self.step_count += 1
        print(f"\n=== 训练步骤 {self.step_count} ===")
        
        # 检查输入
        self._check_inputs(inputs, targets)
        
        # 前向传播
        outputs = self.model(inputs)
        print(f"输出形状: {outputs.shape}")
        print(f"输出范围: [{outputs.min().item():.6f}, {outputs.max().item():.6f}]")
        
        # 检查输出中的异常值
        if torch.isnan(outputs).any():
            print("警告: 输出包含NaN值")
        if torch.isinf(outputs).any():
            print("警告: 输出包含Inf值")
        
        # 计算损失
        loss = self.criterion(outputs, targets)
        print(f"损失: {loss.item():.6f}")
        
        if torch.isnan(loss) or torch.isinf(loss):
            print("错误: 损失为NaN或Inf")
            return False
        
        self.loss_history.append(loss.item())
        
        # 反向传播
        self.optimizer.zero_grad()
        loss.backward()
        
        # 检查梯度
        grad_norm = self._check_gradients()
        self.grad_norm_history.append(grad_norm)
        
        if grad_norm > 1000:
            print("警告: 梯度范数过大，可能存在梯度爆炸")
        
        # 优化步骤
        self.optimizer.step()
        
        return True
    
    def _check_inputs(self, inputs, targets):
        """
        检查输入数据
        """
        print(f"输入形状: {inputs.shape}")
        print(f"目标形状: {targets.shape}")
        print(f"输入范围: [{inputs.min().item():.4f}, {inputs.max().item():.4f}]")
        print(f"目标范围: [{targets.min().item():.4f}, {targets.max().item():.4f}]")
        
        if torch.isnan(inputs).any() or torch.isnan(targets).any():
            print("警告: 输入数据包含NaN值")
        if torch.isinf(inputs).any() or torch.isinf(targets).any():
            print("警告: 输入数据包含Inf值")
    
    def _check_gradients(self):
        """
        检查模型梯度
        """
        total_norm = 0
        param_count = 0
        
        for name, param in self.model.named_parameters():
            if param.grad is not None:
                grad_norm = param.grad.norm().item()
                total_norm += grad_norm ** 2
                param_count += 1
                
                if torch.isnan(param.grad).any():
                    print(f"警告: {name} 梯度包含NaN")
                if torch.isinf(param.grad).any():
                    print(f"警告: {name} 梯度包含Inf")
                if grad_norm > 100:
                    print(f"警告: {name} 梯度范数过大 ({grad_norm:.2f})")
            else:
                print(f"警告: {name} 无梯度")
        
        total_norm = total_norm ** 0.5
        print(f"总梯度范数: {total_norm:.6f}")
        
        return total_norm
    
    def get_training_stats(self):
        """
        获取训练统计信息
        """
        if not self.loss_history:
            return "无训练数据"
        
        avg_loss = sum(self.loss_history) / len(self.loss_history)
        avg_grad_norm = sum(self.grad_norm_history) / len(self.grad_norm_history) if self.grad_norm_history else 0
        
        return {
            'steps': self.step_count,
            'avg_loss': avg_loss,
            'avg_grad_norm': avg_grad_norm,
            'loss_history': self.loss_history[-10:],  # 最近10个损失值
            'grad_norm_history': self.grad_norm_history[-10:]  # 最近10个梯度范数
        }

# 使用示例
# debugger = TrainingDebugger(model, optimizer, criterion)
# success = debugger.debug_step(inputs, targets)
# stats = debugger.get_training_stats()
```

### 8. 如何调试模型推理过程？
```python
class InferenceDebugger:
    def __init__(self, model):
        self.model = model
        self.hooks = []
    
    def debug_inference(self, inputs, expected_output_range=None):
        """
        调试模型推理
        """
        print("=== 推理调试 ===")
        
        # 检查输入
        self._check_inputs(inputs)
        
        # 注册调试钩子
        self._register_hooks()
        
        try:
            # 执行推理
            self.model.eval()
            with torch.no_grad():
                outputs = self.model(inputs)
            
            # 检查输出
            self._check_outputs(outputs, expected_output_range)
            
            return outputs
        finally:
            # 清理钩子
            self._remove_hooks()
    
    def _check_inputs(self, inputs):
        """
        检查输入数据
        """
        print(f"输入形状: {inputs.shape}")
        print(f"输入数据类型: {inputs.dtype}")
        print(f"输入设备: {inputs.device}")
        print(f"输入范围: [{inputs.min().item():.6f}, {inputs.max().item():.6f}]")
        
        has_nan = torch.isnan(inputs).any().item()
        has_inf = torch.isinf(inputs).any().item()
        print(f"输入包含NaN: {has_nan}")
        print(f"输入包含Inf: {has_inf}")
        
        if has_nan or has_inf:
            print("错误: 输入数据存在问题")
    
    def _check_outputs(self, outputs, expected_range=None):
        """
        检查输出结果
        """
        print(f"输出形状: {outputs.shape}")
        print(f"输出数据类型: {outputs.dtype}")
        print(f"输出范围: [{outputs.min().item():.6f}, {outputs.max().item():.6f}]")
        
        has_nan = torch.isnan(outputs).any().item()
        has_inf = torch.isinf(outputs).any().item()
        print(f"输出包含NaN: {has_nan}")
        print(f"输出包含Inf: {has_inf}")
        
        if expected_range:
            min_expected, max_expected = expected_range
            if outputs.min().item() < min_expected or outputs.max().item() > max_expected:
                print(f"警告: 输出超出预期范围 [{min_expected}, {max_expected}]")
    
    def _register_hooks(self):
        """
        注册调试钩子
        """
        def forward_hook(module, input, output, module_name):
            print(f"\n模块: {module_name}")
            if isinstance(input, tuple) and len(input) > 0:
                inp = input[0]
                if isinstance(inp, torch.Tensor):
                    print(f"  输入形状: {inp.shape}")
                    print(f"  输入范围: [{inp.min().item():.6f}, {inp.max().item():.6f}]")
                    if torch.isnan(inp).any():
                        print("  警告: 输入包含NaN")
                    if torch.isinf(inp).any():
                        print("  警告: 输入包含Inf")
            
            if isinstance(output, torch.Tensor):
                print(f"  输出形状: {output.shape}")
                print(f"  输出范围: [{output.min().item():.6f}, {output.max().item():.6f}]")
                if torch.isnan(output).any():
                    print("  警告: 输出包含NaN")
                if torch.isinf(output).any():
                    print("  警告: 输出包含Inf")
        
        # 为所有模块注册钩子
        for name, module in self.model.named_modules():
            if len(name) > 0:  # 跳过根模块
                hook = module.register_forward_hook(
                    lambda module, input, output, name=name: forward_hook(module, input, output, name)
                )
                self.hooks.append(hook)
    
    def _remove_hooks(self):
        """
        移除所有钩子
        """
        for hook in self.hooks:
            hook.remove()
        self.hooks.clear()

# 使用示例
# debugger = InferenceDebugger(model)
# outputs = debugger.debug_inference(inputs, expected_output_range=(-1, 1))
```

### 9. 如何调试分布式训练问题？
```python
import torch.distributed as dist

def debug_distributed_setup():
    """
    调试分布式训练设置
    """
    if not dist.is_available():
        print("分布式训练不可用")
        return False
    
    if not dist.is_initialized():
        print("分布式训练未初始化")
        return False
    
    print("分布式训练信息:")
    print(f"  后端: {dist.get_backend()}")
    print(f"  世界大小: {dist.get_world_size()}")
    print(f"  当前排名: {dist.get_rank()}")
    print(f"  主节点: {dist.get_backend()}")
    
    return True

def debug_distributed_gradients(model):
    """
    调试分布式训练中的梯度
    """
    if not dist.is_initialized():
        return
    
    rank = dist.get_rank()
    world_size = dist.get_world_size()
    
    print(f"进程 {rank}/{world_size} 梯度检查:")
    
    for name, param in model.named_parameters():
        if param.grad is not None:
            # 收集所有进程的梯度
            grad_copy = param.grad.clone()
            dist.all_reduce(grad_copy, op=dist.ReduceOp.SUM)
            avg_grad = grad_copy / world_size
            
            local_norm = param.grad.norm().item()
            global_norm = avg_grad.norm().item()
            
            print(f"  {name}:")
            print(f"    本地梯度范数: {local_norm:.6f}")
            print(f"    全局梯度范数: {global_norm:.6f}")
            
            if abs(local_norm - global_norm) > 1e-6:
                print(f"    警告: 梯度不同步")
        else:
            print(f"  {name}: 无梯度")

def sync_debug_info(info_dict, rank):
    """
    同步调试信息
    """
    if dist.is_initialized():
        # 将信息广播给所有进程
        for key, value in info_dict.items():
            if isinstance(value, (int, float)):
                tensor = torch.tensor([value], dtype=torch.float32)
                dist.broadcast(tensor, src=0)
                info_dict[key] = tensor.item()
    
    return info_dict
```

### 10. 如何调试GPU内存问题？
```python
def debug_gpu_memory():
    """
    调试GPU内存使用
    """
    if not torch.cuda.is_available():
        print("CUDA不可用")
        return
    
    print("GPU内存信息:")
    for i in range(torch.cuda.device_count()):
        device_name = torch.cuda.get_device_name(i)
        memory_allocated = torch.cuda.memory_allocated(i) / 1024**3
        memory_reserved = torch.cuda.memory_reserved(i) / 1024**3
        memory_max_allocated = torch.cuda.max_memory_allocated(i) / 1024**3
        
        print(f"  GPU {i} ({device_name}):")
        print(f"    已分配内存: {memory_allocated:.2f} GB")
        print(f"    已保留内存: {memory_reserved:.2f} GB")
        print(f"    最大分配内存: {memory_max_allocated:.2f} GB")
        
        # 重置最大内存统计
        torch.cuda.reset_peak_memory_stats(i)

def clear_gpu_cache():
    """
    清理GPU缓存
    """
    if torch.cuda.is_available():
        torch.cuda.empty_cache()
        print("GPU缓存已清理")

def monitor_memory_during_training(model, inputs):
    """
    监控训练过程中的内存使用
    """
    if not torch.cuda.is_available():
        return
    
    device = next(model.parameters()).device
    
    # 训练前内存
    before_mem = torch.cuda.memory_allocated(device) / 1024**2
    print(f"训练前内存: {before_mem:.2f} MB")
    
    # 前向传播
    outputs = model(inputs)
    forward_mem = torch.cuda.memory_allocated(device) / 1024**2
    print(f"前向传播后内存: {forward_mem:.2f} MB (+{forward_mem - before_mem:.2f} MB)")
    
    # 反向传播
    loss = outputs.sum()
    loss.backward()
    backward_mem = torch.cuda.memory_allocated(device) / 1024**2
    print(f"反向传播后内存: {backward_mem:.2f} MB (+{backward_mem - forward_mem:.2f} MB)")
    
    # 清理梯度
    model.zero_grad()
    zero_grad_mem = torch.cuda.memory_allocated(device) / 1024**2
    print(f"清理梯度后内存: {zero_grad_mem:.2f} MB (+{zero_grad_mem - backward_mem:.2f} MB)")

# 使用示例
# debug_gpu_memory()
# monitor_memory_during_training(model, inputs)
```

### 11. 如何调试模型收敛问题？
```python
class ConvergenceDebugger:
    def __init__(self, model, optimizer, criterion):
        self.model = model
        self.optimizer = optimizer
        self.criterion = criterion
        self.loss_history = []
        self.param_history = {}
        self.grad_history = {}
    
    def track_convergence(self, inputs, targets, num_steps=100):
        """
        跟踪模型收敛情况
        """
        print("开始收敛调试...")
        
        for step in range(num_steps):
            # 前向传播
            outputs = self.model(inputs)
            loss = self.criterion(outputs, targets)
            
            # 记录损失
            self.loss_history.append(loss.item())
            
            # 反向传播
            self.optimizer.zero_grad()
            loss.backward()
            
            # 记录梯度和参数
            self._record_parameters(step)
            self._record_gradients(step)
            
            # 优化步骤
            self.optimizer.step()
            
            # 每10步打印一次进度
            if step % 10 == 0:
                print(f"步骤 {step}: 损失 = {loss.item():.6f}")
        
        # 分析收敛性
        self._analyze_convergence()
    
    def _record_parameters(self, step):
        """
        记录参数变化
        """
        for name, param in self.model.named_parameters():
            if name not in self.param_history:
                self.param_history[name] = []
            self.param_history[name].append(param.detach().clone())
    
    def _record_gradients(self, step):
        """
        记录梯度变化
        """
        for name, param in self.model.named_parameters():
            if param.grad is not None:
                if name not in self.grad_history:
                    self.grad_history[name] = []
                self.grad_history[name].append(param.grad.detach().clone())
    
    def _analyze_convergence(self):
        """
        分析收敛性
        """
        print("\n收敛性分析:")
        
        # 损失分析
        if len(self.loss_history) > 10:
            recent_loss = self.loss_history[-10:]
            avg_recent_loss = sum(recent_loss) / len(recent_loss)
            print(f"最近10步平均损失: {avg_recent_loss:.6f}")
            
            if abs(recent_loss[-1] - recent_loss[0]) < 1e-6:
                print("警告: 损失几乎没有变化，可能已经收敛或卡住")
        
        # 参数变化分析
        for name, param_history in self.param_history.items():
            if len(param_history) > 1:
                param_change = torch.norm(param_history[-1] - param_history[0]).item()
                print(f"{name} 总变化量: {param_change:.6f}")
                
                if param_change < 1e-6:
                    print(f"  警告: {name} 参数几乎没有变化")
        
        # 梯度分析
        for name, grad_history in self.grad_history.items():
            if grad_history:
                avg_grad_norm = sum([grad.norm().item() for grad in grad_history]) / len(grad_history)
                print(f"{name} 平均梯度范数: {avg_grad_norm:.6f}")
                
                if avg_grad_norm < 1e-6:
                    print(f"  警告: {name} 梯度接近零")
                elif avg_grad_norm > 100:
                    print(f"  警告: {name} 梯度过大")

# 使用示例
# debugger = ConvergenceDebugger(model, optimizer, criterion)
# debugger.track_convergence(inputs, targets, num_steps=50)
```

### 12. 如何调试自定义操作和层？
```python
class CustomLayerDebugger:
    def __init__(self):
        self.forward_calls = 0
        self.backward_calls = 0
    
    @staticmethod
    def debug_custom_function(func, *args, **kwargs):
        """
        调试自定义函数
        """
        print(f"调试自定义函数: {func.__name__}")
        
        # 检查输入
        for i, arg in enumerate(args):
            if isinstance(arg, torch.Tensor):
                print(f"  参数 {i}:")
                print(f"    形状: {arg.shape}")
                print(f"    数据类型: {arg.dtype}")
                print(f"    范围: [{arg.min().item():.6f}, {arg.max().item():.6f}]")
                print(f"    包含NaN: {torch.isnan(arg).any().item()}")
                print(f"    包含Inf: {torch.isinf(arg).any().item()}")
        
        # 执行函数
        try:
            result = func(*args, **kwargs)
            print(f"  执行成功")
            
            if isinstance(result, torch.Tensor):
                print(f"  结果形状: {result.shape}")
                print(f"  结果范围: [{result.min().item():.6f}, {result.max().item():.6f}]")
                print(f"  结果包含NaN: {torch.isnan(result).any().item()}")
                print(f"  结果包含Inf: {torch.isinf(result).any().item()}")
            
            return result
        except Exception as e:
            print(f"  执行失败: {e}")
            raise

# 自定义函数调试示例
def custom_activation(x):
    """
    自定义激活函数
    """
    return torch.where(x > 0, x, x * 0.1)

# 调试自定义函数
# result = CustomLayerDebugger.debug_custom_function(custom_activation, torch.randn(10, 5))

class DebuggableCustomFunction(torch.autograd.Function):
    """
    可调试的自定义自动梯度函数
    """
    @staticmethod
    def forward(ctx, input, weight):
        print(f"前向传播: 输入形状 {input.shape}, 权重形状 {weight.shape}")
        
        # 检查输入
        if torch.isnan(input).any() or torch.isnan(weight).any():
            print("警告: 前向传播输入包含NaN")
        if torch.isinf(input).any() or torch.isinf(weight).any():
            print("警告: 前向传播输入包含Inf")
        
        output = torch.mm(input, weight.t())
        ctx.save_for_backward(input, weight)
        
        print(f"前向传播输出形状: {output.shape}")
        return output
    
    @staticmethod
    def backward(ctx, grad_output):
        print(f"反向传播: 梯度输出形状 {grad_output.shape}")
        
        input, weight = ctx.saved_tensors
        grad_input = grad_weight = None
        
        if ctx.needs_input_grad[0]:
            grad_input = torch.mm(grad_output, weight)
            print(f"输入梯度形状: {grad_input.shape}")
            
            if torch.isnan(grad_input).any():
                print("警告: 输入梯度包含NaN")
            if torch.isinf(grad_input).any():
                print("警告: 输入梯度包含Inf")
        
        if ctx.needs_input_grad[1]:
            grad_weight = torch.mm(grad_output.t(), input)
            print(f"权重梯度形状: {grad_weight.shape}")
            
            if torch.isnan(grad_weight).any():
                print("警告: 权重梯度包含NaN")
            if torch.isinf(grad_weight).any():
                print("警告: 权重梯度包含Inf")
        
        return grad_input, grad_weight
```

### 13. 如何调试模型保存和加载问题？
```python
def debug_model_save_load(model, save_path):
    """
    调试模型保存和加载
    """
    print("调试模型保存和加载...")
    
    # 保存前检查
    print("保存前模型检查:")
    param_count = sum(p.numel() for p in model.parameters())
    print(f"  参数总数: {param_count}")
    
    has_nan = any(torch.isnan(p).any() for p in model.parameters())
    has_inf = any(torch.isinf(p).any() for p in model.parameters())
    print(f"  包含NaN参数: {has_nan}")
    print(f"  包含Inf参数: {has_inf}")
    
    # 保存模型
    try:
        torch.save(model.state_dict(), save_path)
        print(f"模型已保存到: {save_path}")
    except Exception as e:
        print(f"保存失败: {e}")
        return False
    
    # 检查文件
    import os
    if os.path.exists(save_path):
        file_size = os.path.getsize(save_path) / (1024 * 1024)  # MB
        print(f"文件大小: {file_size:.2f} MB")
    else:
        print("文件未创建")
        return False
    
    # 加载模型
    try:
        loaded_state_dict = torch.load(save_path, map_location='cpu')
        print("模型加载成功")
    except Exception as e:
        print(f"加载失败: {e}")
        return False
    
    # 比较参数
    print("参数比较:")
    model_state_dict = model.state_dict()
    
    if set(model_state_dict.keys()) != set(loaded_state_dict.keys()):
        print("警告: 参数键不匹配")
        return False
    
    for key in model_state_dict.keys():
        original_param = model_state_dict[key]
        loaded_param = loaded_state_dict[key]
        
        if original_param.shape != loaded_param.shape:
            print(f"警告: {key} 形状不匹配")
            return False
        
        # 数值比较
        diff = torch.abs(original_param - loaded_param).max().item()
        if diff > 1e-6:
            print(f"警告: {key} 数值差异: {diff}")
            return False
    
    print("模型保存和加载验证通过")
    return True

def debug_checkpoint_loading(model, checkpoint_path):
    """
    调试检查点加载
    """
    print(f"调试检查点加载: {checkpoint_path}")
    
    try:
        checkpoint = torch.load(checkpoint_path, map_location='cpu')
        print("检查点加载成功")
        
        # 检查检查点内容
        print("检查点内容:")
        for key in checkpoint.keys():
            print(f"  {key}: {type(checkpoint[key])}")
        
        # 加载模型状态
        if 'model_state_dict' in checkpoint:
            model.load_state_dict(checkpoint['model_state_dict'])
            print("模型状态加载成功")
        else:
            print("警告: 检查点中未找到model_state_dict")
        
        # 加载优化器状态
        if 'optimizer_state_dict' in checkpoint:
            print("优化器状态存在")
        else:
            print("警告: 检查点中未找到optimizer_state_dict")
        
        return checkpoint
    except Exception as e:
        print(f"检查点加载失败: {e}")
        return None
```

### 14. 如何调试模型的数值稳定性？
```python
def debug_numerical_stability(model, inputs):
    """
    调试模型数值稳定性
    """
    print("数值稳定性调试...")
    
    # 设置为评估模式
    model.eval()
    
    # 检查输入
    print("输入分析:")
    print(f"  形状: {inputs.shape}")
    print(f"  范围: [{inputs.min().item():.6f}, {inputs.max().item():.6f}]")
    print(f"  均值: {inputs.mean().item():.6f}")
    print(f"  标准差: {inputs.std().item():.6f}")
    
    # 前向传播
    with torch.no_grad():
        outputs = model(inputs)
    
    print("输出分析:")
    print(f"  形状: {outputs.shape}")
    print(f"  范围: [{outputs.min().item():.6f}, {outputs.max().item():.6f}]")
    print(f"  均值: {outputs.mean().item():.6f}")
    print(f"  标准差: {outputs.std().item():.6f}")
    
    # 检查极端值
    extreme_threshold = 1e6
    if outputs.max().item() > extreme_threshold or outputs.min().item() < -extreme_threshold:
        print(f"警告: 输出包含极端值 (阈值: {extreme_threshold})")
    
    # 检查梯度流
    print("梯度流分析:")
    model.train()
    outputs = model(inputs)
    loss = outputs.sum()
    
    loss.backward()
    
    for name, param in model.named_parameters():
        if param.grad is not None:
            grad_norm = param.grad.norm().item()
            print(f"  {name} 梯度范数: {grad_norm:.6f}")
            
            if grad_norm > 1000:
                print(f"    警告: 梯度范数过大")
            elif grad_norm < 1e-8:
                print(f"    警告: 梯度范数过小")

def implement_numerical_safety(model):
    """
    实施数值安全性措施
    """
    # 梯度裁剪
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
    
    # 使用数值稳定的损失函数
    class StableCrossEntropyLoss(nn.Module):
        def __init__(self, eps=1e-8):
            super().__init__()
            self.eps = eps
        
        def forward(self, input, target):
            # 数值稳定的softmax
            input_stable = input - input.max(dim=1, keepdim=True)[0]
            exp_input = torch.exp(input_stable)
            softmax_output = exp_input / (exp_input.sum(dim=1, keepdim=True) + self.eps)
            
            # 交叉熵
            log_softmax = torch.log(softmax_output + self.eps)
            return -torch.sum(target * log_softmax) / input.size(0)
    
    # 使用混合精度训练
    try:
        from torch.cuda.amp import autocast, GradScaler
        scaler = GradScaler()
        return scaler
    except ImportError:
        print("混合精度训练不可用")
        return None
```

### 15. 如何调试模型的过拟合和欠拟合？
```python
class OverfittingUnderfittingDebugger:
    def __init__(self, model):
        self.model = model
        self.train_losses = []
        self.val_losses = []
        self.train_accuracies = []
        self.val_accuracies = []
    
    def debug_fitting(self, train_loader, val_loader, num_epochs=10):
        """
        调试过拟合和欠拟合
        """
        print("调试模型拟合情况...")
        
        for epoch in range(num_epochs):
            # 训练阶段
            train_loss, train_acc = self._evaluate_model(train_loader, train=True)
            self.train_losses.append(train_loss)
            self.train_accuracies.append(train_acc)
            
            # 验证阶段
            val_loss, val_acc = self._evaluate_model(val_loader, train=False)
            self.val_losses.append(val_loss)
            self.val_accuracies.append(val_acc)
            
            print(f"Epoch {epoch+1}:")
            print(f"  训练损失: {train_loss:.4f}, 准确率: {train_acc:.4f}")
            print(f"  验证损失: {val_loss:.4f}, 准确率: {val_acc:.4f}")
            
            # 检查过拟合迹象
            if epoch > 0:
                if val_loss > self.val_losses[-2] and train_loss < self.train_losses[-2]:
                    print("  警告: 可能出现过拟合")
        
        # 分析拟合情况
        self._analyze_fitting()
    
    def _evaluate_model(self, dataloader, train=False):
        """
        评估模型性能
        """
        if train:
            self.model.train()
            total_loss = 0
            correct = 0
            total = 0
            
            for inputs, targets in dataloader:
                outputs = self.model(inputs)
                loss = F.cross_entropy(outputs, targets)
                
                total_loss += loss.item()
                _, predicted = outputs.max(1)
                total += targets.size(0)
                correct += predicted.eq(targets).sum().item()
            
            avg_loss = total_loss / len(dataloader)
            accuracy = correct / total
            return avg_loss, accuracy
        else:
            self.model.eval()
            total_loss = 0
            correct = 0
            total = 0
            
            with torch.no_grad():
                for inputs, targets in dataloader:
                    outputs = self.model(inputs)
                    loss = F.cross_entropy(outputs, targets)
                    
                    total_loss += loss.item()
                    _, predicted = outputs.max(1)
                    total += targets.size(0)
                    correct += predicted.eq(targets).sum().item()
            
            avg_loss = total_loss / len(dataloader)
            accuracy = correct / total
            return avg_loss, accuracy
    
    def _analyze_fitting(self):
        """
        分析拟合情况
        """
        print("\n拟合分析:")
        
        # 欠拟合检查
        final_train_acc = self.train_accuracies[-1]
        if final_train_acc < 0.8:  # 阈值可根据任务调整
            print("警告: 训练准确率较低，可能存在欠拟合")
        
        # 过拟合检查
        if len(self.train_losses) > 5:
            recent_train_improvement = self.train_losses[-5] - self.train_losses[-1]
            recent_val_improvement = self.val_losses[-5] - self.val_losses[-1]
            
            if recent_train_improvement > 0 and recent_val_improvement < 0:
                print("警告: 验证损失增加而训练损失减少，可能存在过拟合")
        
        # 输出最终性能
        print(f"最终训练准确率: {self.train_accuracies[-1]:.4f}")
        print(f"最终验证准确率: {self.val_accuracies[-1]:.4f}")
```

### 16. 如何调试模型的批处理问题？
```python
def debug_batch_processing(model, dataloader):
    """
    调试批处理问题
    """
    print("调试批处理...")
    
    model.eval()
    
    for batch_idx, (inputs, targets) in enumerate(dataloader):
        print(f"\n批次 {batch_idx}:")
        print(f"  输入形状: {inputs.shape}")
        print(f"  目标形状: {targets.shape}")
        
        # 检查批次大小一致性
        expected_batch_size = dataloader.batch_size
        actual_batch_size = inputs.shape[0]
        if actual_batch_size != expected_batch_size:
            print(f"  警告: 批大小不一致，期望 {expected_batch_size}，实际 {actual_batch_size}")
        
        # 检查数据质量
        if torch.isnan(inputs).any():
            print("  警告: 输入包含NaN值")
        if torch.isinf(inputs).any():
            print("  警告: 输入包含Inf值")
        
        # 执行推理
        try:
            with torch.no_grad():
                outputs = model(inputs)
            
            print(f"  输出形状: {outputs.shape}")
            
            if torch.isnan(outputs).any():
                print("  警告: 输出包含NaN值")
            if torch.isinf(outputs).any():
                print("  警告: 输出包含Inf值")
                
        except Exception as e:
            print(f"  推理失败: {e}")
            continue
        
        # 只调试前几个批次
        if batch_idx >= 2:
            break

def debug_data_collation(dataset, batch_size=4):
    """
    调试数据整理
    """
    print("调试数据整理...")
    
    # 创建简单的数据加载器
    dataloader = torch.utils.data.DataLoader(
        dataset,
        batch_size=batch_size,
        shuffle=False,
        collate_fn=lambda x: x  # 不使用默认整理函数
    )
    
    try:
        batch = next(iter(dataloader))
        print(f"原始批次类型: {type(batch)}")
        print(f"批次长度: {len(batch)}")
        
        # 检查批次内容
        for i, item in enumerate(batch):
            print(f"  项目 {i}: {type(item)}")
            if isinstance(item, (tuple, list)):
                for j, subitem in enumerate(item):
                    if isinstance(subitem, torch.Tensor):
                        print(f"    子项目 {j}: 形状 {subitem.shape}")
                    else:
                        print(f"    子项目 {j}: {type(subitem)}")
            elif isinstance(item, torch.Tensor):
                print(f"    形状: {item.shape}")
        
        return batch
    except Exception as e:
        print(f"数据整理失败: {e}")
        return None
```

### 17. 如何调试模型的随机性和可重复性？
```python
def debug_reproducibility(seed=42):
    """
    调试模型的可重复性
    """
    print(f"设置随机种子: {seed}")
    
    # 设置所有随机种子
    torch.manual_seed(seed)
    torch.cuda.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    np.random.seed(seed)
    random.seed(seed)
    
    # 设置确定性算法
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False
    
    print("随机种子设置完成")

def compare_model_runs(model_class, inputs, num_runs=3, seed=42):
    """
    比较多次运行的结果
    """
    print("比较模型多次运行结果...")
    
    results = []
    
    for run in range(num_runs):
        print(f"运行 {run + 1}:")
        
        # 设置种子
        debug_reproducibility(seed + run)
        
        # 创建新模型
        model = model_class()
        
        # 执行推理
        with torch.no_grad():
            output = model(inputs)
        
        results.append(output.clone())
        print(f"  输出范围: [{output.min().item():.6f}, {output.max().item():.6f}]")
    
    # 比较结果
    print("\n结果比较:")
    for i in range(1, num_runs):
        diff = torch.abs(results[0] - results[i]).max().item()
        print(f"运行1与运行{i+1}的最大差异: {diff:.6f}")
        
        if diff > 1e-6:
            print(f"  警告: 运行结果不一致")
        else:
            print(f"  运行结果一致")

def debug_random_sampling(dataloader, num_batches=5):
    """
    调试随机采样
    """
    print("调试随机采样...")
    
    # 设置不同的种子
    seeds = [42, 123, 456]
    
    for seed in seeds:
        print(f"\n使用种子 {seed}:")
        torch.manual_seed(seed)
        
        # 重新创建数据加载器（因为采样器依赖于种子）
        new_dataloader = torch.utils.data.DataLoader(
            dataloader.dataset,
            batch_size=dataloader.batch_size,
            shuffle=True
        )
        
        # 显示前几个批次
        for batch_idx, (inputs, targets) in enumerate(new_dataloader):
            if batch_idx >= 2:
                break
            print(f"  批次 {batch_idx}: 第一个样本索引 {targets[0].item()}")
```

### 18. 如何调试模型的设备分配问题？
```python
def debug_device_allocation(model, inputs):
    """
    调试设备分配
    """
    print("调试设备分配...")
    
    # 检查模型设备
    model_devices = set()
    for param in model.parameters():
        model_devices.add(param.device)
    
    print(f"模型设备: {model_devices}")
    
    # 检查输入设备
    if isinstance(inputs, torch.Tensor):
        input_device = inputs.device
        print(f"输入设备: {input_device}")
    elif isinstance(inputs, (list, tuple)):
        input_devices = set()
        for item in inputs:
            if isinstance(item, torch.Tensor):
                input_devices.add(item.device)
        print(f"输入设备: {input_devices}")
    
    # 检查兼容性
    if len(model_devices) > 1:
        print("警告: 模型分布在多个设备上")
    
    # 尝试设备迁移
    target_device = 'cuda' if torch.cuda.is_available() else 'cpu'
    print(f"目标设备: {target_device}")
    
    try:
        model.to(target_device)
        print("模型设备迁移成功")
        
        if isinstance(inputs, torch.Tensor):
            inputs = inputs.to(target_device)
        elif isinstance(inputs, (list, tuple)):
            inputs = [item.to(target_device) if isinstance(item, torch.Tensor) else item for item in inputs]
        
        print("输入设备迁移成功")
        
        # 执行推理
        with torch.no_grad():
            outputs = model(inputs)
        print("推理成功执行")
        
    except Exception as e:
        print(f"设备迁移或推理失败: {e}")

def debug_multi_gpu_setup():
    """
    调试多GPU设置
    """
    if not torch.cuda.is_available():
        print("CUDA不可用")
        return
    
    print("多GPU调试信息:")
    print(f"GPU数量: {torch.cuda.device_count()}")
    
    for i in range(torch.cuda.device_count()):
        device_name = torch.cuda.get_device_name(i)
        memory = torch.cuda.get_device_properties(i).total_memory / 1024**3
        print(f"  GPU {i}: {device_name} ({memory:.1f} GB)")
    
    # 检查当前设备
    current_device = torch.cuda.current_device()
    print(f"当前设备: {current_device}")
    
    # 检查可用设备
    for i in range(torch.cuda.device_count()):
        try:
            x = torch.randn(10, 10).cuda(i)
            print(f"  GPU {i}: 可用")
        except Exception as e:
            print(f"  GPU {i}: 不可用 ({e})")
```

### 19. 如何调试模型的学习率问题？
```python
class LearningRateDebugger:
    def __init__(self, model, optimizer):
        self.model = model
        self.optimizer = optimizer
        self.lr_history = []
        self.loss_history = []
        self.grad_norm_history = []
    
    def debug_learning_rate_scheduling(self, dataloader, scheduler=None, num_steps=50):
        """
        调试学习率调度
        """
        print("调试学习率调度...")
        
        for step in range(num_steps):
            # 获取当前学习率
            current_lr = self._get_current_lr()
            self.lr_history.append(current_lr)
            
            print(f"步骤 {step}: 学习率 = {current_lr:.6f}")
            
            # 执行一个训练步骤
            loss, grad_norm = self._train_step(dataloader)
            self.loss_history.append(loss)
            self.grad_norm_history.append(grad_norm)
            
            # 更新学习率
            if scheduler:
                scheduler.step()
                new_lr = self._get_current_lr()
                if new_lr != current_lr:
                    print(f"  学习率更新为: {new_lr:.6f}")
            
            # 检查异常情况
            if loss > 10 or torch.isnan(torch.tensor(loss)):
                print(f"  警告: 损失异常 ({loss})")
            if grad_norm > 100:
                print(f"  警告: 梯度范数过大 ({grad_norm})")
    
    def _get_current_lr(self):
        """
        获取当前学习率
        """
        lrs = []
        for param_group in self.optimizer.param_groups:
            lrs.append(param_group['lr'])
        return sum(lrs) / len(lrs) if lrs else 0
    
    def _train_step(self, dataloader):
        """
        执行单步训练
        """
        self.model.train()
        
        # 获取一个批次
        try:
            batch = next(iter(dataloader))
            inputs, targets = batch
        except Exception as e:
            print(f"数据加载失败: {e}")
            return float('inf'), 0
        
        # 前向传播
        outputs = self.model(inputs)
        loss = F.cross_entropy(outputs, targets)
        
        # 反向传播
        self.optimizer.zero_grad()
        loss.backward()
        
        # 检查梯度
        grad_norm = 0
        for param in self.model.parameters():
            if param.grad is not None:
                grad_norm += param.grad.norm().item() ** 2
        grad_norm = grad_norm ** 0.5
        
        # 优化步骤
        self.optimizer.step()
        
        return loss.item(), grad_norm
    
    def plot_learning_dynamics(self):
        """
        绘制学习动态（需要matplotlib）
        """
        try:
            import matplotlib.pyplot as plt
            
            fig, (ax1, ax2, ax3) = plt.subplots(3, 1, figsize=(10, 12))
            
            # 学习率
            ax1.plot(self.lr_history)
            ax1.set_title('学习率变化')
            ax1.set_xlabel('步骤')
            ax1.set_ylabel('学习率')
            
            # 损失
            ax2.plot(self.loss_history)
            ax2.set_title('损失变化')
            ax2.set_xlabel('步骤')
            ax2.set_ylabel('损失')
            
            # 梯度范数
            ax3.plot(self.grad_norm_history)
            ax3.set_title('梯度范数变化')
            ax3.set_xlabel('步骤')
            ax3.set_ylabel('梯度范数')
            
            plt.tight_layout()
            plt.show()
            
        except ImportError:
            print("matplotlib不可用，无法绘制图表")

def debug_lr_find(model, dataloader, start_lr=1e-7, end_lr=10, num_steps=100):
    """
    调试学习率查找
    """
    print("执行学习率查找...")
    
    # 保存原始状态
    original_state = model.state_dict()
    original_optimizer_state = None
    
    # 创建优化器
    optimizer = torch.optim.SGD(model.parameters(), lr=start_lr)
    
    # 学习率调度器
    lr_mult = (end_lr / start_lr) ** (1.0 / num_steps)
    
    losses = []
    lrs = []
    
    model.train()
    
    for step in range(num_steps):
        # 更新学习率
        current_lr = start_lr * (lr_mult ** step)
        for param_group in optimizer.param_groups:
            param_group['lr'] = current_lr
        
        lrs.append(current_lr)
        
        # 执行训练步骤
        try:
            batch = next(iter(dataloader))
            inputs, targets = batch
            
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = F.cross_entropy(outputs, targets)
            loss.backward()
            optimizer.step()
            
            losses.append(loss.item())
            
            print(f"步骤 {step}: LR={current_lr:.2e}, Loss={loss.item():.4f}")
            
        except Exception as e:
            print(f"步骤 {step} 失败: {e}")
            break
    
    # 恢复原始状态
    model.load_state_dict(original_state)
    
    # 分析结果
    if losses:
        min_loss_idx = np.argmin(losses)
        suggested_lr = lrs[min_loss_idx] / 10  # 建议使用最低损失点前一个数量级的学习率
        print(f"建议学习率: {suggested_lr:.2e}")
    
    return lrs, losses
```

### 20. 如何调试模型的超参数？
```python
class HyperparameterDebugger:
    def __init__(self, model_class, train_loader, val_loader):
        self.model_class = model_class
        self.train_loader = train_loader
        self.val_loader = val_loader
    
    def debug_batch_size(self, batch_sizes=[16, 32, 64, 128]):
        """
        调试批大小
        """
        print("调试批大小...")
        results = {}
        
        for batch_size in batch_sizes:
            print(f"\n测试批大小: {batch_size}")
            
            # 创建新的数据加载器
            train_loader = torch.utils.data.DataLoader(
                self.train_loader.dataset,
                batch_size=batch_size,
                shuffle=True
            )
            
            # 创建新模型
            model = self.model_class()
            optimizer = torch.optim.Adam(model.parameters())
            criterion = nn.CrossEntropyLoss()
            
            # 快速训练几步
            model.train()
            train_losses = []
            
            for step, (inputs, targets) in enumerate(train_loader):
                if step >= 5:  # 只训练5步
                    break
                
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                
                optimizer.zero_grad()
                loss.backward()
                optimizer.step()
                
                train_losses.append(loss.item())
            
            avg_loss = sum(train_losses) / len(train_losses) if train_losses else float('inf')
            results[batch_size] = avg_loss
            
            print(f"  平均训练损失: {avg_loss:.4f}")
        
        best_batch_size = min(results, key=results.get)
        print(f"\n推荐批大小: {best_batch_size} (损失: {results[best_batch_size]:.4f})")
        
        return results
    
    def debug_weight_initialization(self, init_methods=['xavier', 'kaiming', 'normal']):
        """
        调试权重初始化
        """
        print("调试权重初始化...")
        results = {}
        
        def init_weights_xavier(m):
            if isinstance(m, (nn.Linear, nn.Conv2d)):
                nn.init.xavier_uniform_(m.weight)
                if m.bias is not None:
                    nn.init.zeros_(m.bias)
        
        def init_weights_kaiming(m):
            if isinstance(m, (nn.Linear, nn.Conv2d)):
                nn.init.kaiming_uniform_(m.weight, nonlinearity='relu')
                if m.bias is not None:
                    nn.init.zeros_(m.bias)
        
        def init_weights_normal(m):
            if isinstance(m, (nn.Linear, nn.Conv2d)):
                nn.init.normal_(m.weight, 0, 0.1)
                if m.bias is not None:
                    nn.init.zeros_(m.bias)
        
        init_functions = {
            'xavier': init_weights_xavier,
            'kaiming': init_weights_kaiming,
            'normal': init_weights_normal
        }
        
        for method in init_methods:
            print(f"\n测试初始化方法: {method}")
            
            # 创建新模型
            model = self.model_class()
            
            # 应用初始化
            if method in init_functions:
                model.apply(init_functions[method])
            
            # 快速验证
            model.eval()
            with torch.no_grad():
                sample_input = torch.randn(1, *next(iter(self.train_loader))[0].shape[1:])
                try:
                    output = model(sample_input)
                    output_range = (output.min().item(), output.max().item())
                    results[method] = output_range
                    print(f"  输出范围: [{output_range[0]:.4f}, {output_range[1]:.4f}]")
                except Exception as e:
                    print(f"  初始化失败: {e}")
                    results[method] = None
        
        return results

def debug_model_architecture(model):
    """
    调试模型架构
    """
    print("调试模型架构...")
    
    total_params = sum(p.numel() for p in model.parameters())
    trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
    
    print(f"总参数数: {total_params:,}")
    print(f"可训练参数数: {trainable_params:,}")
    
    # 详细参数信息
    print("\n各层参数详情:")
    for name, param in model.named_parameters():
        print(f"  {name}: {param.shape} ({param.numel():,} 参数)")
    
    # 检查参数初始化
    print("\n参数初始化检查:")
    for name, param in model.named_parameters():
        mean = param.mean().item()
        std = param.std().item()
        print(f"  {name}: 均值={mean:.6f}, 标准差={std:.6f}")
        
        if abs(mean) > 1.0 or std > 10.0:
            print(f"    警告: 参数范围可能不适当")
    
    # 模型复杂度分析
    print(f"\n模型复杂度:")
    print(f"  参数数量级: 10^{int(np.log10(total_params))}")
    
    if total_params > 1e8:
        print("  警告: 模型可能过于复杂")
    elif total_params < 1e4:
        print("  警告: 模型可能过于简单")

# 使用示例
# hp_debugger = HyperparameterDebugger(ModelClass, train_loader, val_loader)
# batch_size_results = hp_debugger.debug_batch_size()
# init_results = hp_debugger.debug_weight_initialization()
```

### 21. 如何调试模型的输入预处理？
```python
def debug_input_preprocessing(dataset, transform=None, num_samples=10):
    """
    调试输入预处理
    """
    print("调试输入预处理...")
    
    # 选择样本
    indices = np.random.choice(len(dataset), min(num_samples, len(dataset)), replace=False)
    
    for i, idx in enumerate(indices):
        print(f"\n样本 {i+1} (索引 {idx}):")
        
        # 获取原始数据
        try:
            if isinstance(dataset, torch.utils.data.Subset):
                original_idx = dataset.indices[idx]
                raw_data = dataset.dataset[original_idx]
            else:
                raw_data = dataset[idx]
            
            print(f"  原始数据类型: {type(raw_data)}")
            
            if isinstance(raw_data, (tuple, list)) and len(raw_data) >= 2:
                raw_input, raw_target = raw_data[0], raw_data[1]
                print(f"  原始输入类型: {type(raw_input)}")
                print(f"  原始目标类型: {type(raw_target)}")
                
                if hasattr(raw_input, 'shape'):
                    print(f"  原始输入形状: {raw_input.shape}")
                if hasattr(raw_target, 'shape'):
                    print(f"  原始目标形状: {raw_target.shape}")
            else:
                raw_input = raw_data
                print(f"  原始输入类型: {type(raw_input)}")
                if hasattr(raw_input, 'shape'):
                    print(f"  原始输入形状: {raw_input.shape}")
            
        except Exception as e:
            print(f"  获取原始数据失败: {e}")
            continue
        
        # 应用变换
        if transform:
            try:
                transformed_data = transform(raw_input)
                print(f"  变换后类型: {type(transformed_data)}")
                
                if isinstance(transformed_data, torch.Tensor):
                    print(f"  变换后形状: {transformed_data.shape}")
                    print(f"  变换后范围: [{transformed_data.min().item():.4f}, {transformed_data.max().item():.4f}]")
                    print(f"  变换后均值: {transformed_data.mean().item():.4f}")
                    print(f"  变换后标准差: {transformed_data.std().item():.4f}")
                    
                    if torch.isnan(transformed_data).any():
                        print("  警告: 变换后包含NaN值")
                    if torch.isinf(transformed_data).any():
                        print("  警告: 变换后包含Inf值")
                else:
                    print(f"  变换后数据: {transformed_data}")
                    
            except Exception as e:
                print(f"  变换失败: {e}")

def debug_normalization_stats(dataloader, num_batches=10):
    """
    调试归一化统计
    """
    print("调试归一化统计...")
    
    # 计算均值和标准差
    mean_sum = 0
    std_sum = 0
    total_samples = 0
    
    for batch_idx, (inputs, _) in enumerate(dataloader):
        if batch_idx >= num_batches:
            break
            
        if isinstance(inputs, torch.Tensor):
            # 假设输入是图像 (B, C, H, W)
            batch_mean = inputs.mean(dim=(0, 2, 3))  # 每个通道的均值
            batch_std = inputs.std(dim=(0, 2, 3))    # 每个通道的标准差
            
            mean_sum += batch_mean * inputs.size(0)
            std_sum += batch_std * inputs.size(0)
            total_samples += inputs.size(0)
    
    if total_samples > 0:
        overall_mean = mean_sum / total_samples
        overall_std = std_sum / total_samples
        
        print(f"计算的均值: {overall_mean}")
        print(f"计算的标准差: {overall_std}")
        
        # 与常用归一化值比较
        imagenet_mean = torch.tensor([0.485, 0.456, 0.406])
        imagenet_std = torch.tensor([0.229, 0.224, 0.225])
        
        print(f"\n与ImageNet归一化比较:")
        print(f"  均值差异: {torch.abs(overall_mean - imagenet_mean)}")
        print(f"  标准差差异: {torch.abs(overall_std - imagenet_std)}")

# 使用示例
# debug_input_preprocessing(train_dataset, transform=train_transform)
# debug_normalization_stats(train_loader)
```

### 22. 如何调试模型的输出后处理？
```python
def debug_output_postprocessing(model, inputs, postprocess_fn=None):
    """
    调试输出后处理
    """
    print("调试输出后处理...")
    
    model.eval()
    
    with torch.no_grad():
        # 原始输出
        raw_outputs = model(inputs)
        print(f"原始输出形状: {raw_outputs.shape}")
        print(f"原始输出范围: [{raw_outputs.min().item():.6f}, {raw_outputs.max().item():.6f}]")
        
        # 检查原始输出
        if torch.isnan(raw_outputs).any():
            print("警告: 原始输出包含NaN值")
        if torch.isinf(raw_outputs).any():
            print("警告: 原始输出包含Inf值")
        
        # 应用后处理
        if postprocess_fn:
            try:
                processed_outputs = postprocess_fn(raw_outputs)
                print(f"后处理输出形状: {processed_outputs.shape}")
                
                if isinstance(processed_outputs, torch.Tensor):
                    print(f"后处理输出范围: [{processed_outputs.min().item():.6f}, {processed_outputs.max().item():.6f}]")
                    
                    if torch.isnan(processed_outputs).any():
                        print("警告: 后处理输出包含NaN值")
                    if torch.isinf(processed_outputs).any():
                        print("警告: 后处理输出包含Inf值")
                else:
                    print(f"后处理输出类型: {type(processed_outputs)}")
                    print(f"后处理输出: {processed_outputs}")
                    
            except Exception as e:
                print(f"后处理失败: {e}")
                return raw_outputs
        
        return processed_outputs if postprocess_fn else raw_outputs

def debug_probability_outputs(outputs, temperature=1.0):
    """
    调试概率输出
    """
    print("调试概率输出...")
    
    # 应用温度缩放
    scaled_outputs = outputs / temperature
    
    # Softmax
    probabilities = F.softmax(scaled_outputs, dim=-1)
    
    print(f"缩放后输出范围: [{scaled_outputs.min().item():.6f}, {scaled_outputs.max().item():.6f}]")
    print(f"概率输出范围: [{probabilities.min().item():.6f}, {probabilities.max().item():.6f}]")
    print(f"概率输出和: {probabilities.sum(dim=-1)}")
    
    # 检查概率分布
    max_probs, max_indices = torch.max(probabilities, dim=-1)
    print(f"最大概率: {max_probs}")
    print(f"预测类别: {max_indices}")
    
    # 熵计算
    entropy = -torch.sum(probabilities * torch.log(probabilities + 1e-8), dim=-1)
    print(f"输出熵: {entropy}")
    
    return probabilities

def debug_thresholding(outputs, thresholds=None):
    """
    调试阈值处理
    """
    print("调试阈值处理...")
    
    if thresholds is None:
        # 自适应阈值
        thresholds = torch.mean(outputs, dim=0) + torch.std(outputs, dim=0)
        print(f"自适应阈值: {thresholds}")
    
    # 应用阈值
    if isinstance(thresholds, (int, float)):
        binary_outputs = (outputs > thresholds).float()
    else:
        # 多类别阈值
        thresholds = thresholds.to(outputs.device)
        binary_outputs = (outputs > thresholds).float()
    
    print(f"二值化输出范围: [{binary_outputs.min().item():.6f}, {binary_outputs.max().item():.6f}]")
    print(f"正样本比例: {binary_outputs.mean().item():.4f}")
    
    return binary_outputs
```

### 23. 如何调试模型的评估指标？
```python
def debug_evaluation_metrics(predictions, targets, metrics=['accuracy', 'precision', 'recall']):
    """
    调试评估指标
    """
    print("调试评估指标...")
    
    # 确保输入是正确的格式
    if isinstance(predictions, torch.Tensor):
        predictions = predictions.detach().cpu()
    if isinstance(targets, torch.Tensor):
        targets = targets.detach().cpu()
    
    print(f"预测形状: {predictions.shape}")
    print(f"目标形状: {targets.shape}")
    
    # 检查数据一致性
    if predictions.shape[0] != targets.shape[0]:
        print("警告: 预测和目标样本数不匹配")
        return
    
    # 分类任务
    if len(predictions.shape) > 1 and predictions.shape[1] > 1:
        # 多分类
        pred_classes = torch.argmax(predictions, dim=1)
    else:
        # 二分类
        pred_classes = (predictions > 0.5).float().squeeze()
        targets = targets.float().squeeze()
    
    print(f"预测类别形状: {pred_classes.shape}")
    print(f"目标类别形状: {targets.shape}")
    
    # 计算指标
    for metric in metrics:
        try:
            if metric == 'accuracy':
                correct = (pred_classes == targets).float().sum()
                accuracy = correct / len(targets)
                print(f"准确率: {accuracy.item():.4f}")
            
            elif metric == 'precision':
                if len(torch.unique(targets)) == 2:  # 二分类
                    tp = ((pred_classes == 1) & (targets == 1)).float().sum()
                    fp = ((pred_classes == 1) & (targets == 0)).float().sum()
                    precision = tp / (tp + fp + 1e-8)
                    print(f"精确率: {precision.item():.4f}")
                else:
                    print("精确率计算需要二分类任务")
            
            elif metric == 'recall':
                if len(torch.unique(targets)) == 2:  # 二分类
                    tp = ((pred_classes == 1) & (targets == 1)).float().sum()
                    fn = ((pred_classes == 0) & (targets == 1)).float().sum()
                    recall = tp / (tp + fn + 1e-8)
                    print(f"召回率: {recall.item():.4f}")
                else:
                    print("召回率计算需要二分类任务")
            
        except Exception as e:
            print(f"计算 {metric} 时出错: {e}")

def debug_confusion_matrix(predictions, targets, num_classes=None):
    """
    调试混淆矩阵
    """
    print("调试混淆矩阵...")
    
    if isinstance(predictions, torch.Tensor):
        predictions = predictions.detach().cpu()
    if isinstance(targets, torch.Tensor):
        targets = targets.detach().cpu()
    
    # 获取预测类别
    if len(predictions.shape) > 1:
        pred_classes = torch.argmax(predictions, dim=1)
    else:
        pred_classes = (predictions > 0.5).float()
    
    # 确定类别数
    if num_classes is None:
        num_classes = max(pred_classes.max(), targets.max()).int().item() + 1
    
    # 计算混淆矩阵
    confusion_matrix = torch.zeros(num_classes, num_classes, dtype=torch.long)
    
    for t, p in zip(targets, pred_classes):
        confusion_matrix[t.long(), p.long()] += 1
    
    print("混淆矩阵:")
    print(confusion_matrix)
    
    # 分析每类性能
    for i in range(num_classes):
        tp = confusion_matrix[i, i].item()
        fp = confusion_matrix[:, i].sum().item() - tp
        fn = confusion_matrix[i, :].sum().item() - tp
        tn = confusion_matrix.sum().item() - tp - fp - fn
        
        precision = tp / (tp + fp + 1e-8)
        recall = tp / (tp + fn + 1e-8)
        f1 = 2 * precision * recall / (precision + recall + 1e-8)
        
        print(f"类别 {i}:")
        print(f"  精确率: {precision:.4f}")
        print(f"  召回率: {recall:.4f}")
        print(f"  F1分数: {f1:.4f}")
    
    return confusion_matrix

def debug_roc_curve(predictions, targets):
    """
    调试ROC曲线（二分类）
    """
    try:
        from sklearn.metrics import roc_curve, auc
        import matplotlib.pyplot as plt
    except ImportError:
        print("sklearn或matplotlib不可用")
        return
    
    print("调试ROC曲线...")
    
    if isinstance(predictions, torch.Tensor):
        predictions = predictions.detach().cpu().numpy()
    if isinstance(targets, torch.Tensor):
        targets = targets.detach().cpu().numpy()
    
    # 确保是概率值
    if len(predictions.shape) > 1:
        if predictions.shape[1] == 2:
            probabilities = predictions[:, 1]  # 正类概率
        else:
            probabilities = np.max(predictions, axis=1)
    else:
        probabilities = predictions
    
    # 计算ROC曲线
    fpr, tpr, thresholds = roc_curve(targets, probabilities)
    roc_auc = auc(fpr, tpr)
    
    print(f"AUC: {roc_auc:.4f}")
    
    # 绘制ROC曲线
    try:
        plt.figure()
        plt.plot(fpr, tpr, color='darkorange', lw=2, label=f'ROC curve (AUC = {roc_auc:.2f})')
        plt.plot([0, 1], [0, 1], color='navy', lw=2, linestyle='--')
        plt.xlim([0.0, 1.0])
        plt.ylim([0.0, 1.05])
        plt.xlabel('假正率')
        plt.ylabel('真正率')
        plt.title('ROC曲线')
        plt.legend(loc="lower right")
        plt.show()
    except:
        print("无法绘制ROC曲线")
    
    return fpr, tpr, roc_auc
```

### 24. 如何调试模型的集成和投票？
```python
def debug_ensemble_predictions(models, inputs, voting='hard'):
    """
    调试模型集成预测
    """
    print(f"调试模型集成预测 ({voting} voting)...")
    
    predictions = []
    
    # 获取每个模型的预测
    for i, model in enumerate(models):
        model.eval()
        with torch.no_grad():
            pred = model(inputs)
            predictions.append(pred)
            print(f"模型 {i+1} 输出形状: {pred.shape}")
            print(f"模型 {i+1} 输出范围: [{pred.min().item():.6f}, {pred.max().item():.6f}]")
    
    # 集成预测
    predictions = torch.stack(predictions)
    print(f"堆叠后形状: {predictions.shape}")
    
    if voting == 'hard':
        # 硬投票（多数投票）
        if len(predictions.shape) > 2:
            # 多分类
            pred_classes = torch.argmax(predictions, dim=2)
        else:
            # 二分类
            pred_classes = (predictions > 0.5).float()
        
        print(f"预测类别形状: {pred_classes.shape}")
        
        # 投票
        ensemble_pred = torch.mode(pred_classes, dim=0)[0]  # 众数
        print(f"集成预测形状: {ensemble_pred.shape}")
        
    elif voting == 'soft':
        # 软投票（平均概率）
        ensemble_pred = torch.mean(predictions, dim=0)
        print(f"集成预测形状: {ensemble_pred.shape}")
        print(f"集成预测范围: [{ensemble_pred.min().item():.6f}, {ensemble_pred.max().item():.6f}]")
    
    # 分析模型一致性
    std_predictions = torch.std(predictions, dim=0)
    print(f"预测标准差范围: [{std_predictions.min().item():.6f}, {std_predictions.max().item():.6f}]")
    print(f"平均预测标准差: {std_predictions.mean().item():.6f}")
    
    if std_predictions.mean().item() > 0.5:
        print("警告: 模型预测差异较大")
    
    return ensemble_pred

def debug_weighted_ensemble(models, weights, inputs):
    """
    调试加权集成
    """
    print("调试加权集成...")
    
    if len(models) != len(weights):
        print("错误: 模型数和权重数不匹配")
        return
    
    print(f"权重: {weights}")
    
    weighted_predictions = []
    
    for i, (model, weight) in enumerate(zip(models, weights)):
        model.eval()
        with torch.no_grad():
            pred = model(inputs)
            weighted_pred = pred * weight
            weighted_predictions.append(weighted_pred)
            print(f"模型 {i+1} 加权预测范围: [{weighted_pred.min().item():.6f}, {weighted_pred.max().item():.6f}]")
    
    # 加权平均
    ensemble_pred = torch.sum(torch.stack(weighted_predictions), dim=0)
    print(f"加权集成预测范围: [{ensemble_pred.min().item():.6f}, {ensemble_pred.max().item():.6f}]")
    
    return ensemble_pred
```

### 25. 如何调试模型的数据增强？
```python
def debug_data_augmentation(dataset, transform, num_samples=5):
    """
    调试数据增强
    """
    print("调试数据增强...")
    
    # 选择样本
    indices = np.random.choice(len(dataset), min(num_samples, len(dataset)), replace=False)
    
    for i, idx in enumerate(indices):
        print(f"\n样本 {i+1}:")
        
        # 获取原始数据
        try:
            original_data = dataset[idx]
            
            if isinstance(original_data, (tuple, list)):
                original_input = original_data[0]
            else:
                original_input = original_data
            
            print(f"原始输入形状: {original_input.shape}")
            print(f"原始输入范围: [{original_input.min().item():.4f}, {original_input.max().item():.4f}]")
            
        except Exception as e:
            print(f"获取原始数据失败: {e}")
            continue
        
        # 应用增强
        try:
            for aug_step in range(3):  # 应用多次增强查看变化
                augmented_data = transform(original_input)
                print(f"  增强 {aug_step+1} 形状: {augmented_data.shape}")
                print(f"  增强 {aug_step+1} 范围: [{augmented_data.min().item():.4f}, {augmented_data.max().item():.4f}]")
                
                # 检查增强效果
                if torch.equal(original_input, augmented_data):
                    print(f"  警告: 增强 {aug_step+1} 未改变数据")
                
        except Exception as e:
            print(f"  增强失败: {e}")

def debug_augmentation_consistency(dataset, transform, num_tests=10):
    """
    调试增强一致性（相同输入应该产生不同输出）
    """
    print("调试增强一致性...")
    
    # 选择一个样本进行多次增强
    idx = np.random.randint(0, len(dataset))
    original_data = dataset[idx]
    
    if isinstance(original_data, (tuple, list)):
        original_input = original_data[0]
    else:
        original_input = original_data
    
    print(f"测试样本索引: {idx}")
    print(f"原始输入形状: {original_input.shape}")
    
    augmented_results = []
    
    for i in range(num_tests):
        try:
            augmented = transform(original_input)
            augmented_results.append(augmented)
            print(f"增强 {i+1} 范围: [{augmented.min().item():.4f}, {augmented.max().item():.4f}]")
        except Exception as e:
            print(f"增强 {i+1} 失败: {e}")
    
    # 检查结果多样性
    if len(augmented_results) > 1:
        # 计算成对差异
        differences = []
        for i in range(len(augmented_results)):
            for j in range(i+1, len(augmented_results)):
                diff = torch.abs(augmented_results[i] - augmented_results[j]).mean().item()
                differences.append(diff)
        
        avg_diff = sum(differences) / len(differences) if differences else 0
        print(f"平均成对差异: {avg_diff:.6f}")
        
        if avg_diff < 1e-6:
            print("警告: 增强结果过于相似，可能没有实际增强效果")

# 使用示例
# debug_data_augmentation(train_dataset, train_transform)
# debug_augmentation_consistency(train_dataset, train_transform)
```

### 26. 如何调试模型的早停和检查点？
```python
class EarlyStoppingDebugger:
    def __init__(self, patience=7, min_delta=0):
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_loss = None
        self.early_stop = False
        self.loss_history = []
    
    def debug_step(self, val_loss, epoch):
        """
        调试早停步骤
        """
        print(f"早停调试 - Epoch {epoch}:")
        print(f"  验证损失: {val_loss:.6f}")
        
        self.loss_history.append(val_loss)
        
        if self.best_loss is None:
            self.best_loss = val_loss
            print(f"  设置最佳损失: {val_loss:.6f}")
        elif val_loss < self.best_loss - self.min_delta:
            print(f"  改进! 旧最佳: {self.best_loss:.6f}, 新最佳: {val_loss:.6f}")
            self.best_loss = val_loss
            self.counter = 0
        else:
            self.counter += 1
            print(f"  无改进次数: {self.counter}/{self.patience}")
            
            if self.counter >= self.patience:
                self.early_stop = True
                print("  触发早停!")
        
        return self.early_stop

def debug_checkpoint_saving(model, optimizer, epoch, loss, checkpoint_dir='./checkpoints'):
    """
    调试检查点保存
    """
    print("调试检查点保存...")
    
    # 创建检查点目录
    os.makedirs(checkpoint_dir, exist_ok=True)
    
    checkpoint = {
        'epoch': epoch,
        'model_state_dict': model.state_dict(),
        'optimizer_state_dict': optimizer.state_dict(),
        'loss': loss,
        'timestamp': time.time()
    }
    
    checkpoint_path = os.path.join(checkpoint_dir, f'checkpoint_epoch_{epoch}.pth')
    
    try:
        torch.save(checkpoint, checkpoint_path)
        print(f"检查点已保存到: {checkpoint_path}")
        
        # 验证保存
        loaded_checkpoint = torch.load(checkpoint_path)
        print(f"检查点加载验证成功")
        print(f"保存的epoch: {loaded_checkpoint['epoch']}")
        print(f"保存的损失: {loaded_checkpoint['loss']:.6f}")
        
        return checkpoint_path
    except Exception as e:
        print(f"检查点保存失败: {e}")
        return None

def debug_best_model_tracking(model, current_loss, best_loss, save_path):
    """
    调试最佳模型跟踪
    """
    print("调试最佳模型跟踪...")
    print(f"当前损失: {current_loss:.6f}")
    print(f"最佳损失: {best_loss:.6f}")
    
    if current_loss < best_loss:
        print("发现更好的模型!")
        try:
            torch.save(model.state_dict(), save_path)
            print(f"最佳模型已保存到: {save_path}")
            return current_loss
        except Exception as e:
            print(f"保存最佳模型失败: {e}")
            return best_loss
    else:
        print("当前模型不是最佳的")
        return best_loss
```

### 27. 如何调试模型的梯度流和反向传播？
```python
def debug_gradient_flow(model, inputs, targets, criterion):
    """
    调试梯度流
    """
    print("调试梯度流...")
    
    model.train()
    
    # 前向传播
    outputs = model(inputs)
    loss = criterion(outputs, targets)
    
    print(f"损失: {loss.item():.6f}")
    
    # 反向传播
    loss.backward()
    
    # 分析梯度
    print("\n梯度分析:")
    total_norm = 0
    layer_count = 0
    
    for name, param in model.named_parameters():
        if param.grad is not None:
            grad_norm = param.grad.norm().item()
            total_norm += grad_norm ** 2
            layer_count += 1
            
            print(f"{name}:")
            print(f"  参数形状: {param.shape}")
            print(f"  梯度形状: {param.grad.shape}")
            print(f"  梯度范数: {grad_norm:.6f}")
            
            # 检查梯度问题
            if torch.isnan(param.grad).any():
                print(f"  警告: 梯度包含NaN值")
            if torch.isinf(param.grad).any():
                print(f"  警告: 梯度包含Inf值")
            if grad_norm < 1e-8:
                print(f"  警告: 梯度接近零")
            elif grad_norm > 100:
                print(f"  警告: 梯度范数过大")
        else:
            print(f"{name}: 无梯度")
    
    total_norm = total_norm ** 0.5
    print(f"\n总梯度范数: {total_norm:.6f}")
    
    # 梯度可视化（如果可能）
    try:
        import matplotlib.pyplot as plt
        
        grad_norms = []
        layer_names = []
        
        for name, param in model.named_parameters():
            if param.grad is not None:
                grad_norms.append(param.grad.norm().item())
                layer_names.append(name)
        
        if grad_norms:
            plt.figure(figsize=(10, 6))
            plt.bar(range(len(grad_norms)), grad_norms)
            plt.xlabel('层')
            plt.ylabel('梯度范数')
            plt.title('各层梯度范数')
            plt.xticks(range(len(layer_names)), layer_names, rotation=45, ha='right')
            plt.tight_layout()
            plt.show()
    except ImportError:
        print("matplotlib不可用，跳过梯度可视化")

def debug_backward_hooks(model):
    """
    使用钩子调试反向传播
    """
    print("使用钩子调试反向传播...")
    
    backward_hooks = []
    
    def backward_hook(module, grad_input, grad_output, module_name):
        print(f"\n{module_name} 反向传播钩子:")
        print(f"  梯度输入数量: {len(grad_input)}")
        print(f"  梯度输出数量: {len(grad_output)}")
        
        for i, grad in enumerate(grad_input):
            if grad is not None:
                print(f"  梯度输入 {i} 形状: {grad.shape}")
                print(f"  梯度输入 {i} 范数: {grad.norm().item():.6f}")
        
        for i, grad in enumerate(grad_output):
            if grad is not None:
                print(f"  梯度输出 {i} 形状: {grad.shape}")
                print(f"  梯度输出 {i} 范数: {grad.norm().item():.6f}")
    
    # 为所有模块注册反向钩子
    for name, module in model.named_modules():
        if len(name) > 0:  # 跳过根模块
            hook = module.register_backward_hook(
                lambda module, grad_input, grad_output, name=name: 
                backward_hook(module, grad_input, grad_output, name)
            )
            backward_hooks.append(hook)
    
    return backward_hooks

def debug_gradient_clipping(model, max_norm=1.0):
    """
    调试梯度裁剪
    """
    print("调试梯度裁剪...")
    
    # 计算裁剪前的梯度范数
    total_norm_before = 0
    for param in model.parameters():
        if param.grad is not None:
            total_norm_before += param.grad.norm().item() ** 2
    total_norm_before = total_norm_before ** 0.5
    
    print(f"裁剪前总梯度范数: {total_norm_before:.6f}")
    
    # 应用梯度裁剪
    total_norm_after = torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm)
    
    print(f"裁剪后总梯度范数: {total_norm_after:.6f}")
    print(f"裁剪阈值: {max_norm}")
    
    if total_norm_before > max_norm:
        print("梯度已被裁剪")
    else:
        print("梯度未被裁剪")
    
    return total_norm_after
```

### 28. 如何调试模型的损失函数？
```python
def debug_loss_function(loss_fn, predictions, targets):
    """
    调试损失函数
    """
    print("调试损失函数...")
    
    # 检查输入
    print(f"预测形状: {predictions.shape}")
    print(f"目标形状: {targets.shape}")
    
    if isinstance(predictions, torch.Tensor):
        print(f"预测范围: [{predictions.min().item():.6f}, {predictions.max().item():.6f}]")
        print(f"预测包含NaN: {torch.isnan(predictions).any().item()}")
        print(f"预测包含Inf: {torch.isinf(predictions).any().item()}")
    
    if isinstance(targets, torch.Tensor):
        print(f"目标范围: [{targets.min().item():.6f}, {targets.max().item():.6f}]")
        print(f"目标包含NaN: {torch.isnan(targets).any().item()}")
        print(f"目标包含Inf: {torch.isinf(targets).any().item()}")
    
    # 计算损失
    try:
        loss = loss_fn(predictions, targets)
        print(f"损失值: {loss.item():.6f}")
        
        if torch.isnan(loss):
            print("警告: 损失为NaN")
        if torch.isinf(loss):
            print("警告: 损失为Inf")
        
        return loss
    except Exception as e:
        print(f"损失计算失败: {e}")
        return None

def debug_custom_loss_function():
    """
    调试自定义损失函数
    """
    class DebuggableLoss(nn.Module):
        def __init__(self, eps=1e-8):
            super().__init__()
            self.eps = eps
        
        def forward(self, predictions, targets):
            print("自定义损失函数前向传播:")
            print(f"  预测形状: {predictions.shape}")
            print(f"  目标形状: {targets.shape}")
            
            # 检查输入
            if torch.isnan(predictions).any():
                print("  警告: 预测包含NaN")
            if torch.isinf(predictions).any():
                print("  警告: 预测包含Inf")
            
            # 计算损失（示例：均方误差）
            mse = torch.mean((predictions - targets) ** 2)
            print(f"  MSE: {mse.item():.6f}")
            
            # 添加正则化项（示例）
            l2_reg = torch.sum(predictions ** 2) * 0.01
            print(f"  L2正则化: {l2_reg.item():.6f}")
            
            total_loss = mse + l2_reg
            print(f"  总损失: {total_loss.item():.6f}")
            
            return total_loss
    
    return DebuggableLoss()

def compare_loss_functions(predictions, targets, loss_functions):
    """
    比较不同损失函数
    """
    print("比较损失函数...")
    
    results = {}
    
    for name, loss_fn in loss_functions.items():
        print(f"\n测试 {name}:")
        try:
            loss = loss_fn(predictions, targets)
            results[name] = loss.item()
            print(f"  损失值: {loss.item():.6f}")
        except Exception as e:
            print(f"  计算失败: {e}")
            results[name] = float('inf')
    
    # 找到最佳损失函数
    best_loss_fn = min(results, key=results.get)
    print(f"\n最佳损失函数: {best_loss_fn} (损失: {results[best_loss_fn]:.6f})")
    
    return results
```

### 29. 如何调试模型的优化器？
```python
def debug_optimizer(optimizer, model, inputs, targets, criterion, steps=5):
    """
    调试优化器
    """
    print("调试优化器...")
    
    # 显示优化器信息
    print(f"优化器类型: {type(optimizer).__name__}")
    print("优化器参数组:")
    for i, param_group in enumerate(optimizer.param_groups):
        print(f"  组 {i}:")
        for key, value in param_group.items():
            if key != 'params':
                print(f"    {key}: {value}")
    
    # 训练几步观察行为
    model.train()
    loss_history = []
    lr_history = []
    
    for step in range(steps):
        # 获取当前学习率
        current_lr = [param_group['lr'] for param_group in optimizer.param_groups]
        lr_history.append(current_lr[0] if current_lr else 0)
        print(f"\n步骤 {step}:")
        print(f"  学习率: {current_lr}")
        
        # 前向传播
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss_history.append(loss.item())
        print(f"  损失: {loss.item():.6f}")
        
        # 反向传播
        optimizer.zero_grad()
        loss.backward()
        
        # 检查梯度
        grad_norm = 0
        for param in model.parameters():
            if param.grad is not None:
                grad_norm += param.grad.norm().item() ** 2
        grad_norm = grad_norm ** 0.5
        print(f"  梯度范数: {grad_norm:.6f}")
        
        # 优化步骤
        optimizer.step()
        
        # 检查参数变化
        if step > 0:
            param_change = 0
            for param in model.parameters():
                if hasattr(param, 'prev_data'):
                    param_change += torch.norm(param.data - param.prev_data).item() ** 2
                param.prev_data = param.data.clone()
            param_change = param_change ** 0.5
            print(f"  参数变化: {param_change:.6f}")
    
    # 分析优化过程
    print(f"\n优化过程分析:")
    print(f"  初始损失: {loss_history[0]:.6f}")
    print(f"  最终损失: {loss_history[-1]:.6f}")
    print(f"  损失减少: {loss_history[0] - loss_history[-1]:.6f}")
    
    if loss_history[-1] > loss_history[0]:
        print("  警告: 损失增加了，可能学习率过高")
    
    return loss_history, lr_history

def debug_learning_rate_schedules(scheduler, optimizer, steps=20):
    """
    调试学习率调度
    """
    print("调试学习率调度...")
    
    lr_history = []
    
    for step in range(steps):
        current_lr = [param_group['lr'] for param_group in optimizer.param_groups]
        lr_history.append(current_lr[0] if current_lr else 0)
        print(f"步骤 {step}: 学习率 = {current_lr[0] if current_lr else 0:.6f}")
        
        # 更新学习率
        scheduler.step()
    
    # 分析学习率变化
    print(f"\n学习率变化分析:")
    print(f"  初始学习率: {lr_history[0]:.6f}")
    print(f"  最终学习率: {lr_history[-1]:.6f}")
    print(f"  最大学习率: {max(lr_history):.6f}")
    print(f"  最小学习率: {min(lr_history):.6f}")
    
    # 可视化（如果可能）
    try:
        import matplotlib.pyplot as plt
        plt.figure(figsize=(10, 6))
        plt.plot(lr_history)
        plt.xlabel('步骤')
        plt.ylabel('学习率')
        plt.title('学习率调度')
        plt.grid(True)
        plt.show()
    except ImportError:
        print("matplotlib不可用，跳过可视化")
    
    return lr_history

def debug_optimizer_state(optimizer):
    """
    调试优化器状态
    """
    print("调试优化器状态...")
    
    print("优化器状态字典:")
    state_dict = optimizer.state_dict()
    
    for key, value in state_dict.items():
        if key == 'state':
            print(f"  {key}:")
            for param_id, param_state in value.items():
                print(f"    参数 {param_id}:")
                for state_key, state_value in param_state.items():
                    if isinstance(state_value, torch.Tensor):
                        print(f"      {state_key}: 形状 {state_value.shape}, 范围 [{state_value.min().item():.6f}, {state_value.max().item():.6f}]")
                    else:
                        print(f"      {state_key}: {state_value}")
        else:
            print(f"  {key}: {value}")
```

### 30. 模型调试的最佳实践有哪些？
1. **系统化调试**：从输入到输出逐步检查每个组件
2. **使用调试工具**：充分利用PyTorch内置的调试工具和第三方工具
3. **记录详细日志**：记录关键信息以便回溯和分析
4. **小规模测试**：先在小数据集上验证模型正确性
5. **可视化辅助**：使用图表和可视化工具帮助理解模型行为
6. **版本控制**：对代码和模型进行版本控制
7. **自动化测试**：建立自动化测试流程
8. **文档记录**：详细记录调试过程和发现的问题
9. **团队协作**：与团队成员分享调试经验和解决方案
10. **持续改进**：根据调试经验不断优化调试流程和工具