# 26. ONNX and Model Export

## 问题与解答

### 1. 什么是ONNX（Open Neural Network Exchange）？
ONNX（Open Neural Network Exchange）是一种开放的神经网络交换格式，允许AI模型在不同的框架之间进行转换和部署，如PyTorch、TensorFlow、Caffe2等。

### 2. ONNX的主要优势是什么？
1. 跨框架兼容性
2. 模型优化
3. 硬件加速支持
4. 降低部署复杂性
5. 提高推理性能
6. 与推理引擎集成

### 3. 如何将PyTorch模型导出为ONNX格式？
```python
import torch
import torch.onnx

def export_to_onnx(model, example_input, onnx_path):
    model.eval()  # 设置为评估模式
    
    torch.onnx.export(
        model,                          # 要导出的模型
        example_input,                  # 模型的示例输入
        onnx_path,                      # 输出文件路径
        export_params=True,             # 导出模型参数
        opset_version=11,               # ONNX操作集版本
        do_constant_folding=True,       # 执行常量折叠优化
        input_names=['input'],          # 输入名称
        output_names=['output'],        # 输出名称
        dynamic_axes={
            'input': {0: 'batch_size'},  # 动态轴定义
            'output': {0: 'batch_size'}
        }
    )

# 使用示例
model = MyModel()
example_input = torch.randn(1, 3, 224, 224)
export_to_onnx(model, example_input, 'model.onnx')
```

### 4. ONNX操作集版本（opset_version）的作用是什么？
ONNX操作集版本定义了模型中可以使用的一组操作符。不同版本的操作集包含不同的功能和优化。选择合适的版本很重要：
- 较高版本：支持更多功能，但兼容性可能较差
- 较低版本：兼容性好，但功能有限

### 5. 如何处理动态轴（dynamic_axes）？
动态轴允许模型在某些维度上接受可变大小的输入：
```python
# 为批大小设置动态轴
dynamic_axes = {
    'input': {0: 'batch_size'},        # 批维度是动态的
    'output': {0: 'batch_size'}
}

# 为序列长度设置动态轴（如RNN/LSTM）
dynamic_axes = {
    'input': {0: 'batch_size', 1: 'sequence_length'},
    'output': {0: 'batch_size', 1: 'sequence_length'}
}

# 为图像尺寸设置动态轴
dynamic_axes = {
    'input': {2: 'height', 3: 'width'},
    'output': {2: 'height', 3: 'width'}
}
```

### 6. 如何验证导出的ONNX模型？
```python
import onnx

def validate_onnx_model(onnx_path):
    # 加载ONNX模型
    model = onnx.load(onnx_path)
    
    # 验证模型结构
    onnx.checker.check_model(model)
    print("ONNX模型验证通过")
    
    # 打印模型信息
    print(f"模型输入: {model.graph.input}")
    print(f"模型输出: {model.graph.output}")
    print(f"模型节点数: {len(model.graph.node)}")

# 使用示例
validate_onnx_model('model.onnx')
```

### 7. 如何使用ONNX Runtime进行推理？
```python
import onnxruntime as ort
import numpy as np

def run_onnx_inference(onnx_path, input_data):
    # 创建推理会话
    session = ort.InferenceSession(onnx_path)
    
    # 获取输入名称
    input_name = session.get_inputs()[0].name
    
    # 运行推理
    outputs = session.run(None, {input_name: input_data})
    
    return outputs[0]  # 返回第一个输出

# 使用示例
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
result = run_onnx_inference('model.onnx', input_data)
```

### 8. 如何在不同硬件上优化ONNX模型？
```python
import onnxruntime as ort

def create_optimized_session(onnx_path, use_gpu=True):
    # 配置会话选项
    sess_options = ort.SessionOptions()
    sess_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
    
    # 配置执行提供者
    if use_gpu and 'CUDAExecutionProvider' in ort.get_available_providers():
        providers = ['CUDAExecutionProvider', 'CPUExecutionProvider']
    else:
        providers = ['CPUExecutionProvider']
    
    # 创建优化的推理会话
    session = ort.InferenceSession(
        onnx_path,
        sess_options=sess_options,
        providers=providers
    )
    
    return session

# 使用示例
session = create_optimized_session('model.onnx', use_gpu=True)
```

### 9. 如何处理PyTorch模型中的控制流（if-else、循环）？
```python
import torch
import torch.nn as nn

class ModelWithControlFlow(nn.Module):
    def __init__(self):
        super().__init__()
        self.linear1 = nn.Linear(10, 5)
        self.linear2 = nn.Linear(5, 1)
    
    def forward(self, x, threshold=0.5):
        x = torch.relu(self.linear1(x))
        
        # 条件逻辑 - ONNX支持torch.where
        x = torch.where(x > threshold, x, torch.zeros_like(x))
        
        x = self.linear2(x)
        return x

# 导出包含控制流的模型
model = ModelWithControlFlow()
example_input = (torch.randn(1, 10), 0.5)  # 元组作为输入

torch.onnx.export(
    model,
    example_input,
    'model_with_control_flow.onnx',
    opset_version=11,
    input_names=['input', 'threshold'],
    output_names=['output'],
    dynamic_axes={'input': {0: 'batch_size'}}
)
```

### 10. 如何处理自定义操作？
```python
# 方法1: 使用torch.jit.script
@torch.jit.script
def custom_operation(x: torch.Tensor) -> torch.Tensor:
    return torch.clamp(x, min=0.0, max=1.0)

class ModelWithCustomOp(nn.Module):
    def forward(self, x):
        x = custom_operation(x)
        return x

# 使用torch.jit.script然后导出
model = ModelWithCustomOp()
model_scripted = torch.jit.script(model)
example_input = torch.randn(1, 10)
torch.onnx.export(model_scripted, example_input, 'custom_op_model.onnx')

# 方法2: 重新实现为ONNX支持的操作
def clamp_operation(x, min_val, max_val):
    # 使用ONNX支持的操作实现clamp
    x = torch.max(x, torch.tensor(min_val))
    x = torch.min(x, torch.tensor(max_val))
    return x
```

### 11. 如何优化ONNX模型大小？
```python
from onnx import optimizer

def optimize_onnx_model(onnx_path, output_path):
    # 加载模型
    model = onnx.load(onnx_path)
    
    # 应用优化
    passes = [
        'eliminate_deadend',      # 消除死端
        'eliminate_identity',     # 消除恒等操作
        'eliminate_nop_dropout',  # 消除无操作dropout
        'eliminate_nop_pad',      # 消除无操作padding
        'eliminate_nop_transpose',# 消除无操作transpose
        'fuse_add_bias_into_conv',# 将bias融合到conv中
        'fuse_bn_into_conv',      # 将batchnorm融合到conv中
        'fuse_pad_into_conv'      # 将pad融合到conv中
    ]
    
    optimized_model = optimizer.optimize(model, passes)
    
    # 保存优化后的模型
    onnx.save(optimized_model, output_path)
    
    return optimized_model

# 使用示例
optimized_model = optimize_onnx_model('model.onnx', 'optimized_model.onnx')
```

### 12. 如何处理包含TorchScript的模型？
```python
import torch
import torch.nn as nn

class TorchScriptModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv = nn.Conv2d(3, 64, 3)
        self.relu = nn.ReLU()
    
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.conv(x)
        x = self.relu(x)
        return x

# 方法1: 先转换为TorchScript再导出为ONNX
model = TorchScriptModel()
model.eval()

# 使用trace
traced_model = torch.jit.trace(model, torch.randn(1, 3, 224, 224))
traced_model.save('traced_model.pt')

# 导出为ONNX
torch.onnx.export(
    traced_model,
    torch.randn(1, 3, 224, 224),
    'traced_model.onnx',
    opset_version=11
)

# 方法2: 直接从脚本化模型导出
scripted_model = torch.jit.script(model)
torch.onnx.export(
    scripted_model,
    torch.randn(1, 3, 224, 224),
    'scripted_model.onnx',
    opset_version=11
)
```

### 13. 如何处理循环和RNN层？
```python
class LSTMModel(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers, num_classes):
        super(LSTMModel, self).__init__()
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
        self.fc = nn.Linear(hidden_size, num_classes)
    
    def forward(self, x):
        # LSTM输出: (output, (hidden, cell))
        lstm_out, (hidden, cell) = self.lstm(x)
        
        # 使用最后一个时间步的输出
        out = self.fc(lstm_out[:, -1, :])
        return out

# 导出LSTM模型
model = LSTMModel(10, 20, 2, 5)
example_input = torch.randn(32, 10, 10)  # (batch, seq, feature)

torch.onnx.export(
    model,
    example_input,
    'lstm_model.onnx',
    opset_version=11,
    input_names=['input'],
    output_names=['output'],
    dynamic_axes={
        'input': {0: 'batch_size', 1: 'sequence_length'},
        'output': {0: 'batch_size'}
    }
)
```

### 14. 如何处理模型量化？
```python
from onnxruntime.quantization import quantize_dynamic, QuantType

def quantize_onnx_model(onnx_path, quantized_path):
    # 动态量化（权重从float32转为int8）
    quantized_model = quantize_dynamic(
        model_input=onnx_path,
        model_output=quantized_path,
        weight_type=QuantType.QUInt8  # 或 QuantType.QInt8
    )
    
    return quantized_model

# 使用示例
quantized_model = quantize_onnx_model('model.onnx', 'quantized_model.onnx')

# 静态量化需要校准数据
from onnxruntime.quantization import quantize_static, CalibrationDataReader

class MyCalibrationDataReader(CalibrationDataReader):
    def __init__(self, calibration_files):
        self.data = iter(calibration_files)
    
    def get_next(self):
        try:
            return next(self.data)
        except StopIteration:
            return None

# 静态量化
def quantize_static_example(onnx_path, quantized_path, calibration_data_reader):
    quantized_model = quantize_static(
        model_input=onnx_path,
        model_output=quantized_path,
        calibration_data_reader=calibration_data_reader,
        quant_format=QuantFormat.QDQ,
        activation_type=QuantType.QUInt8,
        weight_type=QuantType.QUInt8
    )
```

### 15. 如何验证ONNX模型的数值一致性？
```python
def verify_onnx_export(pytorch_model, onnx_path, test_input):
    pytorch_model.eval()
    
    # PyTorch推理
    with torch.no_grad():
        pytorch_output = pytorch_model(test_input)
    
    # ONNX Runtime推理
    session = ort.InferenceSession(onnx_path)
    input_name = session.get_inputs()[0].name
    onnx_output = session.run(None, {input_name: test_input.numpy()})[0]
    
    # 比较输出
    import numpy as np
    diff = np.abs(pytorch_output.numpy() - onnx_output)
    max_diff = np.max(diff)
    
    print(f"最大差异: {max_diff}")
    print(f"平均差异: {np.mean(diff)}")
    
    if max_diff < 1e-5:
        print("模型导出正确")
    else:
        print("模型导出可能存在问题")
    
    return max_diff < 1e-5

# 使用示例
test_input = torch.randn(1, 3, 224, 224)
is_correct = verify_onnx_export(model, 'model.onnx', test_input)
```

### 16. 如何处理PyTorch中的自定义层？
```python
class CustomLayer(nn.Module):
    def __init__(self, in_features, out_features):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(out_features, in_features))
        self.bias = nn.Parameter(torch.randn(out_features))
    
    def forward(self, x):
        # 自定义矩阵乘法
        return torch.addmm(self.bias, x, self.weight.t())

# 将自定义层转换为ONNX支持的操作
class ONNXCompatibleCustomLayer(nn.Module):
    def __init__(self, in_features, out_features):
        super().__init__()
        self.linear = nn.Linear(in_features, out_features)
    
    def forward(self, x):
        return self.linear(x)

# 或者使用torch.autograd.Function来定义导出友好的操作
class CustomFunction(torch.autograd.Function):
    @staticmethod
    def forward(ctx, input, weight, bias):
        return torch.addmm(bias, input, weight.t())
    
    @staticmethod
    def backward(ctx, grad_output):
        # 实现反向传播
        pass

class ModelWithCustomFunction(nn.Module):
    def __init__(self, in_features, out_features):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(out_features, in_features))
        self.bias = nn.Parameter(torch.randn(out_features))
    
    def forward(self, x):
        return CustomFunction.apply(x, self.weight, self.bias)
```

### 17. 如何处理包含多个输出的模型？
```python
class MultiOutputModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.shared = nn.Linear(100, 50)
        self.classifier = nn.Linear(50, 10)
        self.regressor = nn.Linear(50, 1)
    
    def forward(self, x):
        shared_features = torch.relu(self.shared(x))
        classification_output = self.classifier(shared_features)
        regression_output = self.regressor(shared_features)
        return classification_output, regression_output

# 导出多输出模型
model = MultiOutputModel()
example_input = torch.randn(1, 100)

torch.onnx.export(
    model,
    example_input,
    'multi_output_model.onnx',
    opset_version=11,
    input_names=['input'],
    output_names=['classification_output', 'regression_output'],
    dynamic_axes={'input': {0: 'batch_size'}}
)
```

### 18. 如何在ONNX Runtime中使用GPU？
```python
def create_gpu_session(onnx_path):
    # 检查CUDA支持
    available_providers = ort.get_available_providers()
    print(f"可用执行提供者: {available_providers}")
    
    if 'CUDAExecutionProvider' not in available_providers:
        print("CUDA执行提供者不可用，使用CPU")
        return ort.InferenceSession(onnx_path)
    
    # 配置CUDA执行提供者
    session_options = ort.SessionOptions()
    
    # 添加CUDA执行提供者
    cuda_provider_options = {
        'device_id': 0,  # GPU设备ID
        'arena_extend_strategy': 'kNextPowerOfTwo',
        'gpu_mem_limit': 2 * 1024 * 1024 * 1024,  # 2GB
        'cudnn_conv_algo_search': 'EXHAUSTIVE'
    }
    
    session = ort.InferenceSession(
        onnx_path,
        sess_options=session_options,
        providers=[('CUDAExecutionProvider', cuda_provider_options), 'CPUExecutionProvider']
    )
    
    return session

# 使用示例
session = create_gpu_session('model.onnx')
```

### 19. 如何处理大型模型？
```python
def export_large_model(model, example_input, onnx_path, use_external_data=True):
    """
    导出大型ONNX模型，使用外部数据
    """
    model.eval()
    
    if use_external_data:
        # 对于大型模型，使用外部数据格式
        torch.onnx.export(
            model,
            example_input,
            onnx_path,
            export_params=True,
            opset_version=11,
            do_constant_folding=True,
            input_names=['input'],
            output_names=['output'],
            dynamic_axes={'input': {0: 'batch_size'}},
            # 使用外部数据存储大张量
            external_data=True,
            export_single_file=False
        )
    else:
        # 标准导出
        torch.onnx.export(
            model,
            example_input,
            onnx_path,
            export_params=True,
            opset_version=11,
            do_constant_folding=True,
            input_names=['input'],
            output_names=['output'],
            dynamic_axes={'input': {0: 'batch_size'}}
        )
```

### 20. 如何实现ONNX模型的版本控制？
```python
import onnx
from datetime import datetime

def add_metadata_to_onnx(onnx_path, model_info):
    """
    向ONNX模型添加元数据
    """
    model = onnx.load(onnx_path)
    
    # 添加元数据
    metadata_props = {
        'model_name': model_info.get('name', 'unknown'),
        'model_version': model_info.get('version', '1.0'),
        'export_date': datetime.now().isoformat(),
        'framework': 'PyTorch',
        'onnx_version': str(onnx.__version__),
        'description': model_info.get('description', ''),
        'author': model_info.get('author', ''),
        'license': model_info.get('license', 'MIT')
    }
    
    # 清除现有元数据
    del model.metadata_props[:]
    
    # 添加新的元数据
    for key, value in metadata_props.items():
        meta = model.metadata_props.add()
        meta.key = key
        meta.value = str(value)
    
    # 保存更新的模型
    onnx.save(model, onnx_path)
    
    return model

# 使用示例
model_info = {
    'name': 'MyImageClassifier',
    'version': '1.2.0',
    'description': 'Image classification model exported from PyTorch',
    'author': 'AI Team'
}

model = add_metadata_to_onnx('model.onnx', model_info)
```

### 21. 如何实现ONNX模型的性能分析？
```python
import onnxruntime as ort
import time

def profile_onnx_model(onnx_path, input_shape, num_runs=100):
    """
    分析ONNX模型性能
    """
    session = ort.InferenceSession(onnx_path)
    input_name = session.get_inputs()[0].name
    
    # 创建测试数据
    test_input = np.random.randn(*input_shape).astype(np.float32)
    
    # 预热
    for _ in range(10):
        session.run(None, {input_name: test_input})
    
    # 性能测试
    latencies = []
    for _ in range(num_runs):
        start_time = time.time()
        session.run(None, {input_name: test_input})
        end_time = time.time()
        latencies.append((end_time - start_time) * 1000)  # 转换为毫秒
    
    # 计算统计数据
    import numpy as np
    avg_latency = np.mean(latencies)
    p95_latency = np.percentile(latencies, 95)
    p99_latency = np.percentile(latencies, 99)
    throughput = (num_runs / sum(latencies)) * 1000  # 每秒推理数
    
    print(f"平均延迟: {avg_latency:.3f} ms")
    print(f"P95延迟: {p95_latency:.3f} ms")
    print(f"P99延迟: {p99_latency:.3f} ms")
    print(f"吞吐量: {throughput:.2f} 推理/秒")
    
    return {
        'avg_latency': avg_latency,
        'p95_latency': p95_latency,
        'p99_latency': p99_latency,
        'throughput': throughput
    }

# 使用示例
profile_results = profile_onnx_model('model.onnx', (1, 3, 224, 224))
```

### 22. 如何处理模型转换错误？
```python
def safe_onnx_export(model, example_input, onnx_path, **kwargs):
    """
    安全的ONNX导出，包含错误处理
    """
    try:
        # 设置模型为评估模式
        model.eval()
        
        # 检查输入形状
        with torch.no_grad():
            example_output = model(example_input)
            print(f"模型输入形状: {example_input.shape}")
            print(f"模型输出形状: {example_output.shape}")
        
        # 设置默认参数
        export_kwargs = {
            'export_params': True,
            'opset_version': 11,
            'do_constant_folding': True,
            'input_names': ['input'],
            'output_names': ['output'],
            'dynamic_axes': {'input': {0: 'batch_size'}}
        }
        export_kwargs.update(kwargs)
        
        # 导出模型
        torch.onnx.export(model, example_input, onnx_path, **export_kwargs)
        
        # 验证导出的模型
        onnx_model = onnx.load(onnx_path)
        onnx.checker.check_model(onnx_model)
        
        print(f"模型成功导出到: {onnx_path}")
        return True
        
    except torch.jit.Error as e:
        print(f"JIT错误: {e}")
        print("尝试使用torch.jit.trace而不是torch.jit.script")
        return False
    except Exception as e:
        print(f"导出失败: {e}")
        
        # 尝试不同的opset版本
        for opset in [9, 10, 11, 12, 13]:
            try:
                print(f"尝试opset版本 {opset}")
                export_kwargs = {
                    'export_params': True,
                    'opset_version': opset,
                    'do_constant_folding': True,
                    'input_names': ['input'],
                    'output_names': ['output']
                }
                export_kwargs.update({k: v for k, v in kwargs.items() if k != 'opset_version'})
                
                torch.onnx.export(model, example_input, onnx_path, **export_kwargs)
                print(f"成功使用opset {opset}导出")
                return True
            except:
                continue
        
        return False
```

### 23. 如何实现模型的增量更新？
```python
def update_onnx_model(original_onnx_path, new_pytorch_model, updated_onnx_path):
    """
    增量更新ONNX模型
    """
    # 加载原始ONNX模型
    original_model = onnx.load(original_onnx_path)
    
    # 导出新模型
    example_input = torch.randn(1, 3, 224, 224)  # 根据实际输入调整
    new_pytorch_model.eval()
    
    torch.onnx.export(
        new_pytorch_model,
        example_input,
        updated_onnx_path,
        opset_version=original_model.opset_import[0].version,  # 保持相同opset版本
        export_params=True,
        do_constant_folding=True,
        input_names=[inp.name for inp in original_model.graph.input],
        output_names=[out.name for out in original_model.graph.output]
    )
    
    # 添加版本信息
    model = onnx.load(updated_onnx_path)
    for prop in original_model.metadata_props:
        if prop.key == 'model_version':
            # 更新版本号
            prop.value = str(float(prop.value) + 0.1)
    
    onnx.save(model, updated_onnx_path)
    return model
```

### 24. 如何处理模型的输入/输出验证？
```python
def validate_io_spec(onnx_path, expected_input_shape, expected_output_shape):
    """
    验证ONNX模型的输入/输出规格
    """
    model = onnx.load(onnx_path)
    
    # 检查输入
    inputs = model.graph.input
    if len(inputs) != 1:
        print(f"警告: 模型有 {len(inputs)} 个输入，期望1个")
    
    for input_tensor in inputs:
        input_shape = [dim.dim_value if dim.dim_value > 0 else -1 
                      for dim in input_tensor.type.tensor_type.shape.dim]
        print(f"输入形状: {input_shape}")
        
        # 比较期望输入形状
        if expected_input_shape != input_shape:
            print(f"输入形状不匹配: 期望 {expected_input_shape}, 得到 {input_shape}")
    
    # 检查输出
    outputs = model.graph.output
    if len(outputs) != 1:
        print(f"警告: 模型有 {len(outputs)} 个输出，期望1个")
    
    for output_tensor in outputs:
        output_shape = [dim.dim_value if dim.dim_value > 0 else -1 
                       for dim in output_tensor.type.tensor_type.shape.dim]
        print(f"输出形状: {output_shape}")
        
        # 比较期望输出形状
        if expected_output_shape != output_shape:
            print(f"输出形状不匹配: 期望 {expected_output_shape}, 得到 {output_shape}")
    
    return True
```

### 25. ONNX Runtime中的内存管理策略有哪些？
```python
def configure_memory_options():
    """
    配置ONNX Runtime的内存选项
    """
    session_options = ort.SessionOptions()
    
    # 内存优化选项
    session_options.enable_mem_pattern = False  # 禁用内存模式优化
    session_options.enable_mem_reuse = True     # 启用内存重用
    
    # 会话配置
    session_options.session_log_verbosity_level = 0
    session_options.session_logid = "ONNXModel"
    
    # 禁用某些优化以减少内存使用
    session_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_EXTENDED
    
    # 设置内存分配器选项
    session_options.add_session_config_entry('session.intra_op.allow_spinning', '1')
    
    return session_options

def create_memory_efficient_session(onnx_path):
    """
    创建内存高效的会话
    """
    session_options = configure_memory_options()
    
    session = ort.InferenceSession(
        onnx_path,
        sess_options=session_options,
        providers=['CPUExecutionProvider']
    )
    
    return session
```

### 26. 如何实现模型的A/B测试？
```python
class ONNXModelA/BTester:
    def __init__(self, model_paths):
        self.sessions = {}
        for name, path in model_paths.items():
            self.sessions[name] = ort.InferenceSession(path)
    
    def predict_and_compare(self, input_data):
        """
        在多个模型上运行推理并比较结果
        """
        results = {}
        
        for model_name, session in self.sessions.items():
            input_name = session.get_inputs()[0].name
            start_time = time.time()
            output = session.run(None, {input_name: input_data})[0]
            end_time = time.time()
            
            results[model_name] = {
                'output': output,
                'latency': (end_time - start_time) * 1000,
                'shape': output.shape
            }
        
        # 比较结果
        model_names = list(results.keys())
        if len(model_names) >= 2:
            output1 = results[model_names[0]]['output']
            output2 = results[model_names[1]]['output']
            diff = np.abs(output1 - output2).mean()
            print(f"模型间平均差异: {diff}")
        
        return results

# 使用示例
tester = ModelABTester({
    'v1': 'model_v1.onnx',
    'v2': 'model_v2.onnx'
})
results = tester.predict_and_compare(test_input)
```

### 27. 如何处理模型的批处理推理？
```python
def batch_inference_with_onnx(session, input_data_list, batch_size=32):
    """
    使用ONNX Runtime进行批处理推理
    """
    input_name = session.get_inputs()[0].name
    results = []
    
    for i in range(0, len(input_data_list), batch_size):
        batch = input_data_list[i:i + batch_size]
        
        # 堆叠批次数据
        batch_tensor = np.stack(batch)
        
        # 运行推理
        batch_result = session.run(None, {input_name: batch_tensor})[0]
        
        # 分解批次结果
        for result in batch_result:
            results.append(result)
    
    return results

# 使用示例
session = ort.InferenceSession('model.onnx')
input_data_list = [np.random.randn(1, 3, 224, 224).astype(np.float32) for _ in range(100)]
results = batch_inference_with_onnx(session, input_data_list, batch_size=16)
```

### 28. ONNX模型的常见限制有哪些？
1. 某些PyTorch操作不支持
2. 动态形状支持有限
3. 自定义操作需要特殊处理
4. 某些控制流可能不被支持
5. 版本兼容性问题

### 29. 如何调试ONNX模型转换问题？
1. 检查PyTorch版本兼容性
2. 使用较低的opset版本
3. 简化模型结构
4. 使用TorchScript预处理
5. 检查模型中的自定义操作

### 30. ONNX部署的最佳实践有哪些？
1. 使用适当的opset版本
2. 进行充分的数值验证
3. 优化模型大小和性能
4. 添加模型元数据
5. 实现错误处理和日志记录
6. 进行性能基准测试
7. 使用适当的内存管理
8. 实现模型版本控制
9. 考虑硬件加速选项
10. 建立监控和告警机制