# 25. Model Deployment

## 问题与解答

### 1. 什么是模型部署（Model Deployment）？
模型部署是将训练好的机器学习模型集成到生产环境中，使其能够为实际应用提供预测服务的过程。这包括模型优化、服务化、监控和维护等环节。

### 2. 模型部署的主要挑战有哪些？
1. 性能优化（延迟、吞吐量）
2. 可扩展性
3. 可靠性和容错性
4. 安全性
5. 监控和日志
6. 版本管理
7. A/B测试
8. 成本控制

### 3. 模型部署的常见方式有哪些？
1. Web服务（REST API、GraphQL）
2. 批处理服务
3. 边缘部署（移动设备、IoT）
4. 云部署（AWS、GCP、Azure）
5. 容器化部署（Docker、Kubernetes）
6. Serverless部署

### 4. 如何将PyTorch模型转换为生产就绪格式？
```python
import torch
import torch.jit as jit

# 方法1：使用TorchScript
def convert_to_torchscript(model, example_input):
    model.eval()
    
    # 跟踪模型
    traced_model = jit.trace(model, example_input)
    
    # 或者使用脚本化
    scripted_model = jit.script(model)
    
    # 保存模型
    traced_model.save("model_traced.pt")
    scripted_model.save("model_scripted.pt")
    
    return traced_model, scripted_model

# 方法2：使用ONNX
def convert_to_onnx(model, example_input, onnx_path):
    model.eval()
    
    torch.onnx.export(
        model,
        example_input,
        onnx_path,
        export_params=True,
        opset_version=11,
        do_constant_folding=True,
        input_names=['input'],
        output_names=['output'],
        dynamic_axes={
            'input': {0: 'batch_size'},
            'output': {0: 'batch_size'}
        }
    )
```

### 5. 什么是TorchScript？
TorchScript是PyTorch的中间表示（IR），可以将PyTorch模型序列化为可以在没有Python依赖的环境中运行的格式。它支持两种模式：跟踪（Tracing）和脚本化（Scripting）。

### 6. 如何创建REST API服务来部署PyTorch模型？
```python
from flask import Flask, request, jsonify
import torch
from torchvision import transforms
from PIL import Image
import io

app = Flask(__name__)

# 加载模型
model = torch.jit.load('model_traced.pt')
model.eval()

# 图像预处理
transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

@app.route('/predict', methods=['POST'])
def predict():
    if 'image' not in request.files:
        return jsonify({'error': 'No image provided'}), 400
    
    # 读取图像
    image_file = request.files['image']
    image = Image.open(io.BytesIO(image_file.read()))
    
    # 预处理
    input_tensor = transform(image).unsqueeze(0)
    
    # 预测
    with torch.no_grad():
        output = model(input_tensor)
        prediction = torch.argmax(output, dim=1).item()
        confidence = torch.softmax(output, dim=1).max().item()
    
    return jsonify({
        'prediction': prediction,
        'confidence': confidence
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 7. 如何使用FastAPI部署PyTorch模型？
```python
from fastapi import FastAPI, File, UploadFile
from fastapi.responses import JSONResponse
import torch
from torchvision import transforms
from PIL import Image
import io
import uvicorn

app = FastAPI()

# 加载模型
model = torch.jit.load('model_traced.pt')
model.eval()

# 图像预处理
transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

@app.post("/predict")
async def predict(image: UploadFile = File(...)):
    try:
        # 读取图像
        contents = await image.read()
        img = Image.open(io.BytesIO(contents))
        
        # 预处理
        input_tensor = transform(img).unsqueeze(0)
        
        # 预测
        with torch.no_grad():
            output = model(input_tensor)
            prediction = torch.argmax(output, dim=1).item()
            confidence = torch.softmax(output, dim=1).max().item()
        
        return JSONResponse({
            'prediction': prediction,
            'confidence': confidence
        })
    except Exception as e:
        return JSONResponse({'error': str(e)}, status_code=500)

if __name__ == '__main__':
    uvicorn.run(app, host='0.0.0.0', port=8000)
```

### 8. 如何在Docker中部署PyTorch模型？
```dockerfile
# Dockerfile
FROM pytorch/pytorch:1.9.0-cuda11.1-cudnn8-runtime

WORKDIR /app

# 复制依赖文件
COPY requirements.txt .
RUN pip install -r requirements.txt

# 复制应用代码
COPY . .

# 暴露端口
EXPOSE 8000

# 启动服务
CMD ["python", "app.py"]
```

```bash
# requirements.txt
fastapi==0.68.0
uvicorn==0.15.0
torchvision==0.10.0
pillow==8.3.1

# 构建和运行
docker build -t pytorch-model .
docker run -p 8000:8000 pytorch-model
```

### 9. 如何实现模型的批处理预测？
```python
class BatchPredictor:
    def __init__(self, model, batch_size=32, max_wait_time=0.1):
        self.model = model
        self.batch_size = batch_size
        self.max_wait_time = max_wait_time
        self.request_queue = []
        self.lock = threading.Lock()
        
    def add_request(self, input_data, callback):
        with self.lock:
            self.request_queue.append((input_data, callback))
            
            # 如果队列已满，立即处理
            if len(self.request_queue) >= self.batch_size:
                self._process_batch()
    
    def _process_batch(self):
        if not self.request_queue:
            return
            
        # 获取一批请求
        batch_requests = self.request_queue[:self.batch_size]
        self.request_queue = self.request_queue[self.batch_size:]
        
        # 组合输入数据
        batch_inputs = torch.stack([req[0] for req in batch_requests])
        
        # 批量预测
        with torch.no_grad():
            outputs = self.model(batch_inputs)
        
        # 分发结果
        for i, (input_data, callback) in enumerate(batch_requests):
            callback(outputs[i])

# 使用示例
predictor = BatchPredictor(model, batch_size=16)

def handle_prediction_result(result):
    # 处理预测结果
    pass

# 添加请求
predictor.add_request(input_tensor, handle_prediction_result)
```

### 10. 如何实现模型版本管理？
```python
class ModelVersionManager:
    def __init__(self, model_dir):
        self.model_dir = model_dir
        self.current_version = None
        self.models = {}
        
    def load_model(self, version):
        model_path = os.path.join(self.model_dir, f"model_v{version}.pt")
        if os.path.exists(model_path):
            model = torch.jit.load(model_path)
            self.models[version] = model
            return model
        else:
            raise ValueError(f"Model version {version} not found")
    
    def switch_version(self, version):
        if version in self.models:
            self.current_version = version
            return self.models[version]
        else:
            # 加载模型
            model = self.load_model(version)
            self.current_version = version
            return model
    
    def get_current_model(self):
        if self.current_version:
            return self.models[self.current_version]
        else:
            raise ValueError("No model loaded")

# 使用示例
version_manager = ModelVersionManager("./models")
model_v1 = version_manager.switch_version("1.0")
model_v2 = version_manager.switch_version("2.0")
```

### 11. 如何实现A/B测试？
```python
import random

class ABTester:
    def __init__(self, models, weights=None):
        self.models = models  # 字典：{'A': model_a, 'B': model_b}
        self.weights = weights or {k: 1/len(models) for k in models}
        self.results = {k: {'requests': 0, 'correct': 0} for k in models}
        
    def predict(self, input_data, ground_truth=None):
        # 根据权重选择模型
        model_key = random.choices(
            list(self.models.keys()), 
            weights=list(self.weights.values())
        )[0]
        
        # 记录请求
        self.results[model_key]['requests'] += 1
        
        # 预测
        with torch.no_grad():
            output = self.models[model_key](input_data)
            prediction = torch.argmax(output, dim=1)
        
        # 如果有真实标签，记录准确性
        if ground_truth is not None:
            correct = (prediction == ground_truth).sum().item()
            self.results[model_key]['correct'] += correct
        
        return prediction, model_key
    
    def get_results(self):
        return {
            model: {
                'requests': stats['requests'],
                'accuracy': stats['correct'] / max(stats['requests'], 1)
            }
            for model, stats in self.results.items()
        }

# 使用示例
ab_tester = ABTester({'A': model_a, 'B': model_b})
prediction, model_used = ab_tester.predict(input_data, ground_truth)
```

### 12. 如何监控模型性能？
```python
import time
import logging
from collections import deque

class ModelMonitor:
    def __init__(self, window_size=1000):
        self.window_size = window_size
        self.latencies = deque(maxlen=window_size)
        self.predictions = deque(maxlen=window_size)
        self.errors = deque(maxlen=window_size)
        self.start_time = time.time()
        
    def record_prediction(self, latency, prediction=None, error=None):
        self.latencies.append(latency)
        if prediction is not None:
            self.predictions.append(prediction)
        if error is not None:
            self.errors.append(error)
    
    def get_metrics(self):
        if not self.latencies:
            return {}
            
        return {
            'avg_latency': sum(self.latencies) / len(self.latencies),
            'p95_latency': sorted(self.latencies)[int(0.95 * len(self.latencies))],
            'p99_latency': sorted(self.latencies)[int(0.99 * len(self.latencies))],
            'error_rate': len(self.errors) / max(len(self.latencies), 1),
            'throughput': len(self.latencies) / max(time.time() - self.start_time, 1)
        }
    
    def log_metrics(self):
        metrics = self.get_metrics()
        logging.info(f"Model Performance Metrics: {metrics}")

# 在预测服务中使用
monitor = ModelMonitor()

@app.post("/predict")
async def predict(image: UploadFile = File(...)):
    start_time = time.time()
    
    try:
        # 预测逻辑...
        result = model(input_tensor)
        latency = time.time() - start_time
        
        # 记录性能指标
        monitor.record_prediction(latency, result)
        
        return JSONResponse({'result': result.tolist()})
    except Exception as e:
        latency = time.time() - start_time
        monitor.record_prediction(latency, error=str(e))
        return JSONResponse({'error': str(e)}, status_code=500)
```

### 13. 如何实现模型的热更新？
```python
import threading
import time

class HotSwappableModel:
    def __init__(self, model_path):
        self.model_path = model_path
        self.current_model = self._load_model()
        self.lock = threading.RLock()
        self.last_modified = os.path.getmtime(model_path)
        
        # 启动监控线程
        self.monitor_thread = threading.Thread(target=self._monitor_model, daemon=True)
        self.monitor_thread.start()
    
    def _load_model(self):
        return torch.jit.load(self.model_path)
    
    def _monitor_model(self):
        while True:
            try:
                current_modified = os.path.getmtime(self.model_path)
                if current_modified > self.last_modified:
                    print("检测到模型更新，重新加载...")
                    with self.lock:
                        self.current_model = self._load_model()
                        self.last_modified = current_modified
                time.sleep(5)  # 每5秒检查一次
            except Exception as e:
                print(f"监控模型时出错: {e}")
                time.sleep(10)
    
    def predict(self, input_data):
        with self.lock:
            with torch.no_grad():
                return self.current_model(input_data)

# 使用示例
hot_model = HotSwappableModel("model.pt")
```

### 14. 如何优化模型推理性能？
```python
# 1. 使用TensorRT（如果可用）
def optimize_with_tensorrt(model, input_shape):
    try:
        import torch_tensorrt
        compiled_model = torch_tensorrt.compile(
            model,
            inputs=[torch_tensorrt.Input(input_shape)],
            enabled_precisions={torch.float, torch.half}
        )
        return compiled_model
    except ImportError:
        print("TensorRT not available, using original model")
        return model

# 2. 使用量化
def quantize_model_for_inference(model):
    model.eval()
    model_quantized = torch.quantization.quantize_dynamic(
        model, {torch.nn.Linear, torch.nn.Conv2d}, dtype=torch.qint8
    )
    return model_quantized

# 3. 使用混合精度
def enable_mixed_precision(model):
    model.eval()
    # 在推理时使用半精度
    return model.half()

# 4. 优化数据加载
def create_optimized_dataloader(dataset, batch_size):
    return torch.utils.data.DataLoader(
        dataset,
        batch_size=batch_size,
        num_workers=0,  # 减少工作进程开销
        pin_memory=True,
        persistent_workers=False
    )
```

### 15. 如何处理模型部署中的错误和异常？
```python
import traceback
from functools import wraps

def handle_model_errors(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except torch.cuda.OutOfMemoryError:
            logging.error("CUDA Out of Memory")
            return {"error": "Out of memory"}, 500
        except Exception as e:
            logging.error(f"Model error: {str(e)}")
            logging.error(traceback.format_exc())
            return {"error": "Internal server error"}, 500
    return wrapper

@app.post("/predict")
@handle_model_errors
def predict():
    # 预测逻辑
    pass
```

### 16. 如何实现模型的健康检查？
```python
@app.get("/health")
def health_check():
    try:
        # 检查模型是否可以正常加载和推理
        dummy_input = torch.randn(1, 3, 224, 224)
        with torch.no_grad():
            model(dummy_input)
        return {"status": "healthy", "model": "loaded"}
    except Exception as e:
        logging.error(f"Health check failed: {e}")
        return {"status": "unhealthy", "error": str(e)}, 500

@app.get("/ready")
def readiness_check():
    # 检查模型是否准备好处理请求
    if model is not None:
        return {"status": "ready"}
    else:
        return {"status": "not ready"}, 503
```

### 17. 如何实现模型的缓存机制？
```python
from functools import lru_cache
import hashlib

class ModelCache:
    def __init__(self, maxsize=128):
        self.cache = {}
        self.maxsize = maxsize
        self.hits = 0
        self.misses = 0
    
    def _hash_input(self, input_tensor):
        # 创建输入的哈希值
        return hashlib.md5(input_tensor.numpy().tobytes()).hexdigest()
    
    def predict_with_cache(self, model, input_tensor):
        input_hash = self._hash_input(input_tensor)
        
        if input_hash in self.cache:
            self.hits += 1
            return self.cache[input_hash]
        else:
            self.misses += 1
            # 缓存未命中，执行预测
            with torch.no_grad():
                result = model(input_tensor)
            
            # 更新缓存
            if len(self.cache) >= self.maxsize:
                # 移除最旧的条目
                oldest_key = next(iter(self.cache))
                del self.cache[oldest_key]
            
            self.cache[input_hash] = result
            
            return result
    
    def get_cache_stats(self):
        total = self.hits + self.misses
        hit_rate = self.hits / max(total, 1)
        return {
            'hits': self.hits,
            'misses': self.misses,
            'hit_rate': hit_rate,
            'cache_size': len(self.cache)
        }

# 使用示例
cache = ModelCache(maxsize=1000)
result = cache.predict_with_cache(model, input_tensor)
```

### 18. 如何实现模型的负载均衡？
```python
import random
from threading import Lock

class LoadBalancer:
    def __init__(self, models):
        self.models = models  # 模型列表
        self.lock = Lock()
        self.request_counts = [0] * len(models)
    
    def get_model_round_robin(self):
        with self.lock:
            # 轮询选择
            min_index = self.request_counts.index(min(self.request_counts))
            self.request_counts[min_index] += 1
            return self.models[min_index], min_index
    
    def get_model_random(self):
        # 随机选择
        index = random.randint(0, len(self.models) - 1)
        return self.models[index], index
    
    def get_model_least_loaded(self):
        # 选择负载最少的模型
        with self.lock:
            min_index = self.request_counts.index(min(self.request_counts))
            self.request_counts[min_index] += 1
            return self.models[min_index], min_index

# 使用示例
load_balancer = LoadBalancer([model1, model2, model3])
selected_model, model_index = load_balancer.get_model_least_loaded()
```

### 19. 如何实现模型的异步预测？
```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AsyncModelPredictor:
    def __init__(self, model, max_workers=4):
        self.model = model
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
    
    async def predict_async(self, input_data):
        loop = asyncio.get_event_loop()
        # 在线程池中执行同步预测
        result = await loop.run_in_executor(
            self.executor, 
            self._sync_predict, 
            input_data
        )
        return result
    
    def _sync_predict(self, input_data):
        with torch.no_grad():
            return self.model(input_data).tolist()
    
    def shutdown(self):
        self.executor.shutdown(wait=True)

# 使用示例
predictor = AsyncModelPredictor(model)

@app.post("/predict")
async def predict(image: UploadFile = File(...)):
    # 异步处理图像和预测
    result = await predictor.predict_async(input_tensor)
    return JSONResponse({'result': result})
```

### 20. 如何实现模型的自动扩缩容？
```python
import psutil
import threading

class AutoScaler:
    def __init__(self, model_service, min_replicas=1, max_replicas=10, 
                 cpu_threshold=80, memory_threshold=80):
        self.model_service = model_service
        self.min_replicas = min_replicas
        self.max_replicas = max_replicas
        self.cpu_threshold = cpu_threshold
        self.memory_threshold = memory_threshold
        self.current_replicas = min_replicas
        
        # 启动监控线程
        self.monitor_thread = threading.Thread(target=self._monitor_resources, daemon=True)
        self.monitor_thread.start()
    
    def _monitor_resources(self):
        while True:
            try:
                # 获取系统资源使用情况
                cpu_percent = psutil.cpu_percent(interval=1)
                memory_percent = psutil.virtual_memory().percent
                
                # 根据资源使用情况调整副本数
                if cpu_percent > self.cpu_threshold or memory_percent > self.memory_threshold:
                    self._scale_up()
                elif cpu_percent < self.cpu_threshold * 0.5 and memory_percent < self.memory_threshold * 0.5:
                    self._scale_down()
                
                time.sleep(30)  # 每30秒检查一次
            except Exception as e:
                print(f"自动扩缩容监控出错: {e}")
                time.sleep(60)
    
    def _scale_up(self):
        if self.current_replicas < self.max_replicas:
            self.current_replicas += 1
            print(f"扩缩容: 增加到 {self.current_replicas} 个副本")
            # 实际的扩缩容逻辑
    
    def _scale_down(self):
        if self.current_replicas > self.min_replicas:
            self.current_replicas -= 1
            print(f"扩缩容: 减少到 {self.current_replicas} 个副本")
            # 实际的扩缩容逻辑
```

### 21. 如何实现模型的配置管理？
```python
import yaml
import json

class ModelConfig:
    def __init__(self, config_path):
        self.config_path = config_path
        self.config = self._load_config()
    
    def _load_config(self):
        with open(self.config_path, 'r') as f:
            if self.config_path.endswith('.yaml') or self.config_path.endswith('.yml'):
                return yaml.safe_load(f)
            else:
                return json.load(f)
    
    def get(self, key, default=None):
        keys = key.split('.')
        value = self.config
        try:
            for k in keys:
                value = value[k]
            return value
        except (KeyError, TypeError):
            return default
    
    def reload(self):
        self.config = self._load_config()

# 配置文件示例 (config.yaml)
"""
model:
  path: "models/model_v1.pt"
  batch_size: 32
  num_workers: 4

server:
  host: "0.0.0.0"
  port: 8000
  debug: false

monitoring:
  enabled: true
  interval: 60
"""

# 使用示例
config = ModelConfig("config.yaml")
model_path = config.get("model.path")
batch_size = config.get("model.batch_size", 32)
```

### 22. 如何实现模型的日志和追踪？
```python
import logging
import uuid
from datetime import datetime

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('model_service.log'),
        logging.StreamHandler()
    ]
)

class RequestTracer:
    def __init__(self):
        self.logger = logging.getLogger(__name__)
    
    def trace_request(self, func):
        def wrapper(*args, **kwargs):
            request_id = str(uuid.uuid4())
            start_time = datetime.now()
            
            self.logger.info(f"Request {request_id} started")
            
            try:
                result = func(*args, **kwargs)
                end_time = datetime.now()
                duration = (end_time - start_time).total_seconds()
                
                self.logger.info(f"Request {request_id} completed in {duration:.3f}s")
                return result
            except Exception as e:
                end_time = datetime.now()
                duration = (end_time - start_time).total_seconds()
                
                self.logger.error(f"Request {request_id} failed after {duration:.3f}s: {str(e)}")
                raise
        return wrapper

# 使用示例
tracer = RequestTracer()

@app.post("/predict")
@tracer.trace_request
def predict():
    # 预测逻辑
    pass
```

### 23. 如何实现模型的安全性？
```python
from functools import wraps
import secrets

def require_api_key(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')
        if not api_key or not verify_api_key(api_key):
            return jsonify({'error': 'Unauthorized'}), 401
        return func(*args, **kwargs)
    return wrapper

def verify_api_key(api_key):
    # 验证API密钥（从数据库或配置文件中获取）
    valid_keys = ["your-secret-api-key"]
    return api_key in valid_keys

def rate_limit(max_requests=100, window=3600):
    def decorator(func):
        request_counts = {}
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            client_ip = request.remote_addr
            current_time = time.time()
            
            if client_ip not in request_counts:
                request_counts[client_ip] = []
            
            # 清理过期的请求记录
            request_counts[client_ip] = [
                req_time for req_time in request_counts[client_ip]
                if current_time - req_time < window
            ]
            
            # 检查是否超过限制
            if len(request_counts[client_ip]) >= max_requests:
                return jsonify({'error': 'Rate limit exceeded'}), 429
            
            # 记录当前请求
            request_counts[client_ip].append(current_time)
            
            return func(*args, **kwargs)
        return wrapper
    return decorator

# 使用示例
@app.post("/predict")
@require_api_key
@rate_limit(max_requests=100, window=3600)
def predict():
    # 预测逻辑
    pass
```

### 24. 如何实现模型的容器化部署？
```dockerfile
# Dockerfile
FROM nvidia/cuda:11.1-cudnn8-runtime-ubuntu20.04

# 安装Python和依赖
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY requirements.txt .
RUN pip3 install -r requirements.txt

# 复制模型和服务代码
COPY models/ models/
COPY app.py .
COPY config.yaml .

# 暴露端口
EXPOSE 8000

# 启动服务
CMD ["python3", "app.py"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  model-service:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./models:/app/models
    environment:
      - CUDA_VISIBLE_DEVICES=0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### 25. 如何实现模型的Kubernetes部署？
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-model-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pytorch-model
  template:
    metadata:
      labels:
        app: pytorch-model
    spec:
      containers:
      - name: model-service
        image: your-registry/pytorch-model-service:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
            nvidia.com/gpu: 1
          limits:
            memory: "4Gi"
            cpu: "2"
            nvidia.com/gpu: 1
        env:
        - name: MODEL_PATH
          value: "/app/models/model.pt"
        volumeMounts:
        - name: model-storage
          mountPath: /app/models
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: model-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: pytorch-model-service
spec:
  selector:
    app: pytorch-model
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
```

### 26. 如何实现模型的自动测试？
```python
import unittest
import torch
from app import ModelService

class TestModelDeployment(unittest.TestCase):
    def setUp(self):
        self.model_service = ModelService()
        self.test_input = torch.randn(1, 3, 224, 224)
    
    def test_model_loading(self):
        """测试模型加载"""
        self.assertIsNotNone(self.model_service.model)
    
    def test_model_prediction(self):
        """测试模型预测"""
        result = self.model_service.predict(self.test_input)
        self.assertIsInstance(result, torch.Tensor)
        self.assertEqual(result.shape[0], 1)
    
    def test_api_endpoint(self):
        """测试API端点"""
        with self.model_service.app.test_client() as client:
            # 模拟文件上传
            response = client.post('/predict', 
                                 data={'image': (io.BytesIO(b"fake image data"), 'test.jpg')})
            self.assertEqual(response.status_code, 200)
    
    def test_health_check(self):
        """测试健康检查"""
        with self.model_service.app.test_client() as client:
            response = client.get('/health')
            self.assertEqual(response.status_code, 200)
            self.assertIn('status', response.get_json())

if __name__ == '__main__':
    unittest.main()
```

### 27. 如何实现模型的性能基准测试？
```python
import time
import numpy as np

def benchmark_model(model, input_shape, num_iterations=1000, warmup_iterations=100):
    """基准测试模型性能"""
    # 创建测试输入
    test_input = torch.randn(input_shape)
    
    # 预热
    for _ in range(warmup_iterations):
        with torch.no_grad():
            _ = model(test_input)
    
    # 基准测试
    latencies = []
    for _ in range(num_iterations):
        start_time = time.time()
        with torch.no_grad():
            _ = model(test_input)
        end_time = time.time()
        latencies.append((end_time - start_time) * 1000)  # 转换为毫秒
    
    # 计算统计信息
    avg_latency = np.mean(latencies)
    p50_latency = np.percentile(latencies, 50)
    p95_latency = np.percentile(latencies, 95)
    p99_latency = np.percentile(latencies, 99)
    throughput = 1000 / avg_latency  # 每秒处理的样本数
    
    print(f"平均延迟: {avg_latency:.2f} ms")
    print(f"P50延迟: {p50_latency:.2f} ms")
    print(f"P95延迟: {p95_latency:.2f} ms")
    print(f"P99延迟: {p99_latency:.2f} ms")
    print(f"吞吐量: {throughput:.2f} 样本/秒")
    
    return {
        'avg_latency': avg_latency,
        'p50_latency': p50_latency,
        'p95_latency': p95_latency,
        'p99_latency': p99_latency,
        'throughput': throughput
    }

# 使用示例
# benchmark_results = benchmark_model(model, (1, 3, 224, 224))
```

### 28. 如何实现模型的CI/CD流水线？
```yaml
# .github/workflows/deploy.yml
name: Model Deployment CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.8
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
    
    - name: Run unit tests
      run: |
        python -m unittest discover tests/
    
    - name: Run model tests
      run: |
        python tests/test_model.py

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Build Docker image
      run: |
        docker build -t ${{ secrets.DOCKER_USERNAME }}/pytorch-model:${{ github.sha }} .
    
    - name: Push to Docker Hub
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push ${{ secrets.DOCKER_USERNAME }}/pytorch-model:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/pytorch-model-service model-service=${{ secrets.DOCKER_USERNAME }}/pytorch-model:${{ github.sha }}
```

### 29. 模型部署中的常见问题有哪些？
1. 性能不一致（训练环境vs生产环境）
2. 内存泄漏
3. 版本兼容性问题
4. 依赖冲突
5. 资源不足
6. 网络延迟
7. 数据预处理不一致
8. 错误处理不当
9. 监控不足
10. 安全漏洞

### 30. 模型部署的最佳实践有哪些？
1. 使用容器化部署（Docker）
2. 实现健康检查和监控
3. 使用负载均衡
4. 实现版本管理和回滚机制
5. 进行充分的测试（单元测试、集成测试、性能测试）
6. 实现日志记录和追踪
7. 使用配置管理
8. 实现安全措施（认证、授权、限流）
9. 优化模型性能（量化、剪枝等）
10. 建立CI/CD流水线
11. 实现自动扩缩容
12. 定期更新和维护