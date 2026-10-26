# 29. Production Best Practices

## 问题与解答

### 1. 深度学习模型生产部署的主要挑战有哪些？
深度学习模型生产部署面临的主要挑战包括：
1. **性能要求**：低延迟、高吞吐量、资源限制
2. **可扩展性**：处理流量波动、自动扩缩容
3. **可靠性**：容错机制、故障恢复、监控告警
4. **安全性**：数据保护、访问控制、模型保护
5. **可维护性**：版本管理、更新部署、回滚机制
6. **成本控制**：计算资源优化、存储优化
7. **合规性**：数据隐私、审计日志、监管要求

### 2. 生产环境中的模型版本管理策略？
```python
import os
import json
from datetime import datetime
from typing import Dict, Any

class ModelVersionManager:
    def __init__(self, model_registry_path: str):
        self.registry_path = model_registry_path
        self.versions_file = os.path.join(model_registry_path, 'versions.json')
        self._ensure_registry_exists()
    
    def _ensure_registry_exists(self):
        """确保模型注册表存在"""
        os.makedirs(self.registry_path, exist_ok=True)
        if not os.path.exists(self.versions_file):
            with open(self.versions_file, 'w') as f:
                json.dump({'versions': {}}, f)
    
    def register_model(self, version: str, model_path: str, 
                      metadata: Dict[str, Any]) -> bool:
        """注册新模型版本"""
        try:
            # 验证模型文件存在
            if not os.path.exists(model_path):
                raise FileNotFoundError(f"模型文件不存在: {model_path}")
            
            # 读取现有版本信息
            with open(self.versions_file, 'r') as f:
                registry = json.load(f)
            
            # 添加新版本
            registry['versions'][version] = {
                'path': model_path,
                'created_at': datetime.now().isoformat(),
                'metadata': metadata,
                'status': 'registered'
            }
            
            # 保存更新
            with open(self.versions_file, 'w') as f:
                json.dump(registry, f, indent=2)
            
            print(f"模型版本 {version} 注册成功")
            return True
            
        except Exception as e:
            print(f"模型注册失败: {e}")
            return False
    
    def promote_model(self, version: str, stage: str) -> bool:
        """提升模型到指定阶段（staging, production等）"""
        try:
            with open(self.versions_file, 'r') as f:
                registry = json.load(f)
            
            if version not in registry['versions']:
                raise ValueError(f"版本 {version} 不存在")
            
            # 更新阶段信息
            if 'stages' not in registry['versions'][version]:
                registry['versions'][version]['stages'] = []
            
            registry['versions'][version]['stages'].append({
                'stage': stage,
                'promoted_at': datetime.now().isoformat()
            })
            
            registry['versions'][version]['status'] = stage
            
            # 保存更新
            with open(self.versions_file, 'w') as f:
                json.dump(registry, f, indent=2)
            
            print(f"模型版本 {version} 已提升到 {stage} 阶段")
            return True
            
        except Exception as e:
            print(f"模型提升失败: {e}")
            return False
    
    def get_model_by_stage(self, stage: str) -> Dict[str, Any]:
        """获取指定阶段的模型"""
        try:
            with open(self.versions_file, 'r') as f:
                registry = json.load(f)
            
            for version, info in registry['versions'].items():
                if info.get('status') == stage:
                    return {'version': version, **info}
            
            return None
        except Exception as e:
            print(f"获取模型失败: {e}")
            return None
    
    def list_versions(self) -> Dict[str, Any]:
        """列出所有版本"""
        try:
            with open(self.versions_file, 'r') as f:
                return json.load(f)
        except Exception as e:
            print(f"列出版本失败: {e}")
            return {}

# 使用示例
version_manager = ModelVersionManager('./model_registry')
metadata = {
    'accuracy': 0.95,
    'dataset': 'imagenet',
    'training_time': '2h',
    'framework': 'PyTorch 1.9.0'
}

# 注册模型
version_manager.register_model('v1.0.0', './models/model_v1.pt', metadata)

# 提升到生产环境
version_manager.promote_model('v1.0.0', 'production')

# 获取生产模型
prod_model = version_manager.get_model_by_stage('production')
```

### 3. 如何实现模型的健康检查和监控？
```python
import time
import psutil
import logging
from typing import Dict, Any
import torch

class ModelHealthMonitor:
    def __init__(self, model, logger=None):
        self.model = model
        self.logger = logger or logging.getLogger(__name__)
        self.metrics = {
            'inference_count': 0,
            'error_count': 0,
            'avg_latency': 0,
            'total_latency': 0
        }
        self.start_time = time.time()
    
    def health_check(self) -> Dict[str, Any]:
        """执行健康检查"""
        try:
            # 检查模型是否可以加载
            device = next(self.model.parameters()).device
            dummy_input = torch.randn(1, 3, 224, 224).to(device)
            
            start_time = time.time()
            with torch.no_grad():
                _ = self.model(dummy_input)
            inference_time = (time.time() - start_time) * 1000  # 转换为毫秒
            
            health_status = {
                'status': 'healthy',
                'model_loadable': True,
                'inference_time_ms': inference_time,
                'timestamp': datetime.now().isoformat()
            }
            
            self.logger.info(f"健康检查通过: 推理时间 {inference_time:.2f}ms")
            
        except Exception as e:
            health_status = {
                'status': 'unhealthy',
                'model_loadable': False,
                'error': str(e),
                'timestamp': datetime.now().isoformat()
            }
            
            self.logger.error(f"健康检查失败: {e}")
        
        return health_status
    
    def system_metrics(self) -> Dict[str, float]:
        """获取系统指标"""
        try:
            cpu_percent = psutil.cpu_percent(interval=1)
            memory_percent = psutil.virtual_memory().percent
            
            # GPU指标（如果可用）
            gpu_metrics = {}
            if torch.cuda.is_available():
                gpu_metrics = {
                    'gpu_memory_allocated_gb': torch.cuda.memory_allocated() / 1024**3,
                    'gpu_memory_reserved_gb': torch.cuda.memory_reserved() / 1024**3,
                    'gpu_utilization_percent': self._get_gpu_utilization()
                }
            
            return {
                'cpu_percent': cpu_percent,
                'memory_percent': memory_percent,
                'timestamp': datetime.now().isoformat(),
                **gpu_metrics
            }
        except Exception as e:
            self.logger.error(f"获取系统指标失败: {e}")
            return {}
    
    def _get_gpu_utilization(self) -> float:
        """获取GPU利用率（简化实现）"""
        try:
            # 这里可以使用nvidia-ml-py等库获取真实GPU利用率
            # 简化实现：基于内存使用率估算
            if torch.cuda.is_available():
                allocated = torch.cuda.memory_allocated()
                reserved = torch.cuda.memory_reserved()
                if reserved > 0:
                    return min(100.0, (allocated / reserved) * 100)
            return 0.0
        except:
            return 0.0
    
    def record_inference(self, latency_ms: float, success: bool = True):
        """记录推理指标"""
        self.metrics['inference_count'] += 1
        self.metrics['total_latency'] += latency_ms
        
        if not success:
            self.metrics['error_count'] += 1
        
        # 计算平均延迟
        self.metrics['avg_latency'] = (
            self.metrics['total_latency'] / self.metrics['inference_count']
        )
    
    def get_metrics(self) -> Dict[str, Any]:
        """获取性能指标"""
        uptime = time.time() - self.start_time
        
        return {
            **self.metrics,
            'uptime_seconds': uptime,
            'success_rate': (
                (self.metrics['inference_count'] - self.metrics['error_count']) / 
                max(self.metrics['inference_count'], 1)
            ),
            'requests_per_second': self.metrics['inference_count'] / max(uptime, 1)
        }

# Flask健康检查端点示例
from flask import Flask, jsonify

app = Flask(__name__)
health_monitor = ModelHealthMonitor(model)

@app.route('/health')
def health_check():
    """健康检查端点"""
    health_status = health_monitor.health_check()
    return jsonify(health_status), 200 if health_status['status'] == 'healthy' else 500

@app.route('/metrics')
def metrics():
    """指标端点"""
    system_metrics = health_monitor.system_metrics()
    model_metrics = health_monitor.get_metrics()
    
    return jsonify({
        'system': system_metrics,
        'model': model_metrics
    })

@app.route('/ready')
def readiness_check():
    """就绪检查端点"""
    try:
        # 检查依赖服务
        # 检查数据库连接
        # 检查模型是否加载完成
        return jsonify({'status': 'ready'}), 200
    except Exception as e:
        return jsonify({'status': 'not ready', 'error': str(e)}), 503
```

### 4. 如何实现模型的自动扩缩容？
```python
import threading
import time
from typing import List, Dict, Any

class AutoScaler:
    def __init__(self, 
                 model_service,
                 min_replicas: int = 1,
                 max_replicas: int = 10,
                 target_cpu_utilization: float = 70.0,
                 scale_up_threshold: float = 80.0,
                 scale_down_threshold: float = 30.0,
                 scale_interval: int = 60):
        self.model_service = model_service
        self.min_replicas = min_replicas
        self.max_replicas = max_replicas
        self.target_cpu_utilization = target_cpu_utilization
        self.scale_up_threshold = scale_up_threshold
        self.scale_down_threshold = scale_down_threshold
        self.scale_interval = scale_interval
        
        self.current_replicas = min_replicas
        self.monitoring = True
        self.monitor_thread = None
    
    def start_monitoring(self):
        """启动监控线程"""
        self.monitoring = True
        self.monitor_thread = threading.Thread(target=self._monitor_and_scale, daemon=True)
        self.monitor_thread.start()
        print("自动扩缩容监控已启动")
    
    def stop_monitoring(self):
        """停止监控"""
        self.monitoring = False
        if self.monitor_thread:
            self.monitor_thread.join()
        print("自动扩缩容监控已停止")
    
    def _monitor_and_scale(self):
        """监控和扩缩容主循环"""
        while self.monitoring:
            try:
                # 获取系统指标
                metrics = self._get_system_metrics()
                cpu_utilization = metrics.get('cpu_percent', 0)
                
                print(f"当前CPU利用率: {cpu_utilization:.1f}%")
                
                # 根据CPU利用率决定是否扩缩容
                if cpu_utilization > self.scale_up_threshold:
                    self._scale_up()
                elif cpu_utilization < self.scale_down_threshold:
                    self._scale_down()
                
                time.sleep(self.scale_interval)
                
            except Exception as e:
                print(f"监控循环出错: {e}")
                time.sleep(self.scale_interval)
    
    def _get_system_metrics(self) -> Dict[str, float]:
        """获取系统指标"""
        try:
            cpu_percent = psutil.cpu_percent(interval=1)
            memory_percent = psutil.virtual_memory().percent
            
            return {
                'cpu_percent': cpu_percent,
                'memory_percent': memory_percent
            }
        except Exception as e:
            print(f"获取系统指标失败: {e}")
            return {'cpu_percent': 0, 'memory_percent': 0}
    
    def _scale_up(self):
        """扩容"""
        if self.current_replicas < self.max_replicas:
            new_replicas = min(self.current_replicas + 1, self.max_replicas)
            if self._update_replicas(new_replicas):
                self.current_replicas = new_replicas
                print(f"扩容到 {self.current_replicas} 个副本")
    
    def _scale_down(self):
        """缩容"""
        if self.current_replicas > self.min_replicas:
            new_replicas = max(self.current_replicas - 1, self.min_replicas)
            if self._update_replicas(new_replicas):
                self.current_replicas = new_replicas
                print(f"缩容到 {self.current_replicas} 个副本")
    
    def _update_replicas(self, new_replicas: int) -> bool:
        """更新副本数"""
        try:
            # 这里应该调用实际的容器编排API（如Kubernetes）
            # 示例：更新Docker容器数量或K8s副本数
            print(f"更新副本数从 {self.current_replicas} 到 {new_replicas}")
            
            # 模拟更新过程
            time.sleep(2)
            
            return True
        except Exception as e:
            print(f"更新副本数失败: {e}")
            return False
    
    def get_status(self) -> Dict[str, Any]:
        """获取自动扩缩容状态"""
        return {
            'current_replicas': self.current_replicas,
            'min_replicas': self.min_replicas,
            'max_replicas': self.max_replicas,
            'target_cpu_utilization': self.target_cpu_utilization,
            'scale_up_threshold': self.scale_up_threshold,
            'scale_down_threshold': self.scale_down_threshold
        }

# Kubernetes扩缩容示例
class KubernetesAutoScaler(AutoScaler):
    def __init__(self, deployment_name: str, namespace: str, **kwargs):
        super().__init__(**kwargs)
        self.deployment_name = deployment_name
        self.namespace = namespace
        self.k8s_client = self._get_k8s_client()
    
    def _get_k8s_client(self):
        """获取Kubernetes客户端"""
        try:
            from kubernetes import client, config
            config.load_incluster_config()  # 在集群内运行时使用
            # config.load_kube_config()  # 本地测试时使用
            return client.AppsV1Api()
        except Exception as e:
            print(f"Kubernetes客户端初始化失败: {e}")
            return None
    
    def _update_replicas(self, new_replicas: int) -> bool:
        """使用Kubernetes API更新副本数"""
        if not self.k8s_client:
            return False
        
        try:
            from kubernetes.client.rest import ApiException
            
            # 更新Deployment
            body = {
                'spec': {
                    'replicas': new_replicas
                }
            }
            
            self.k8s_client.patch_namespaced_deployment(
                name=self.deployment_name,
                namespace=self.namespace,
                body=body
            )
            
            return True
        except ApiException as e:
            print(f"Kubernetes API调用失败: {e}")
            return False
        except Exception as e:
            print(f"更新副本数失败: {e}")
            return False
```

### 5. 如何实现模型的安全性保护？
```python
import hashlib
import hmac
import jwt
import secrets
from functools import wraps
from typing import Dict, Any
import torch

class ModelSecurityManager:
    def __init__(self, secret_key: str = None):
        self.secret_key = secret_key or secrets.token_hex(32)
        self.api_keys = set()
        self.blocked_ips = set()
        self.rate_limits = {}  # IP -> {count, reset_time}
    
    def generate_api_key(self) -> str:
        """生成API密钥"""
        api_key = secrets.token_urlsafe(32)
        self.api_keys.add(api_key)
        return api_key
    
    def verify_api_key(self, api_key: str) -> bool:
        """验证API密钥"""
        return api_key in self.api_keys
    
    def revoke_api_key(self, api_key: str) -> bool:
        """撤销API密钥"""
        if api_key in self.api_keys:
            self.api_keys.remove(api_key)
            return True
        return False
    
    def hash_model_weights(self, model) -> str:
        """计算模型权重哈希值"""
        weights_hash = hashlib.sha256()
        
        for param in model.parameters():
            # 将参数展平并更新哈希
            param_flat = param.detach().cpu().numpy().tobytes()
            weights_hash.update(param_flat)
        
        return weights_hash.hexdigest()
    
    def verify_model_integrity(self, model, expected_hash: str) -> bool:
        """验证模型完整性"""
        current_hash = self.hash_model_weights(model)
        return hmac.compare_digest(current_hash, expected_hash)
    
    def sign_predictions(self, predictions, model_version: str) -> Dict[str, Any]:
        """对预测结果进行签名"""
        payload = {
            'predictions': predictions.tolist() if hasattr(predictions, 'tolist') else predictions,
            'model_version': model_version,
            'timestamp': time.time(),
            'nonce': secrets.token_hex(8)
        }
        
        signature = jwt.encode(payload, self.secret_key, algorithm='HS256')
        return {
            'data': payload,
            'signature': signature
        }
    
    def verify_predictions(self, signed_data: Dict[str, Any]) -> bool:
        """验证预测结果签名"""
        try:
            jwt.decode(signed_data['signature'], self.secret_key, algorithms=['HS256'])
            return True
        except jwt.InvalidTokenError:
            return False
    
    def is_ip_blocked(self, ip_address: str) -> bool:
        """检查IP是否被阻止"""
        return ip_address in self.blocked_ips
    
    def block_ip(self, ip_address: str):
        """阻止IP地址"""
        self.blocked_ips.add(ip_address)
    
    def unblock_ip(self, ip_address: str):
        """解除IP阻止"""
        self.blocked_ips.discard(ip_address)
    
    def check_rate_limit(self, ip_address: str, max_requests: int = 100, 
                        window_seconds: int = 3600) -> bool:
        """检查速率限制"""
        current_time = time.time()
        
        if ip_address not in self.rate_limits:
            self.rate_limits[ip_address] = {
                'count': 1,
                'reset_time': current_time + window_seconds
            }
            return True
        
        ip_info = self.rate_limits[ip_address]
        
        # 重置窗口
        if current_time > ip_info['reset_time']:
            ip_info['count'] = 1
            ip_info['reset_time'] = current_time + window_seconds
            return True
        
        # 检查限制
        if ip_info['count'] >= max_requests:
            return False
        
        # 增加计数
        ip_info['count'] += 1
        return True

# Flask安全装饰器
def require_api_key(security_manager: ModelSecurityManager):
    """API密钥验证装饰器"""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            api_key = request.headers.get('X-API-Key')
            if not api_key or not security_manager.verify_api_key(api_key):
                return jsonify({'error': 'Invalid or missing API key'}), 401
            return f(*args, **kwargs)
        return decorated_function
    return decorator

def rate_limit(security_manager: ModelSecurityManager, max_requests: int = 100):
    """速率限制装饰器"""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            client_ip = request.remote_addr
            
            if security_manager.is_ip_blocked(client_ip):
                return jsonify({'error': 'IP blocked'}), 403
            
            if not security_manager.check_rate_limit(client_ip, max_requests):
                return jsonify({'error': 'Rate limit exceeded'}), 429
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

def protect_model_weights(model_path: str, security_manager: ModelSecurityManager):
    """保护模型权重文件"""
    try:
        # 加载模型
        model = torch.load(model_path, map_location='cpu')
        
        # 计算并存储哈希值
        model_hash = security_manager.hash_model_weights(model)
        
        # 保存带签名的模型
        signed_model_path = model_path + '.signed'
        torch.save({
            'model_state_dict': model.state_dict(),
            'model_hash': model_hash,
            'signature': security_manager.sign_predictions(model_hash, 'model_integrity')
        }, signed_model_path)
        
        return signed_model_path, model_hash
    except Exception as e:
        print(f"模型保护失败: {e}")
        return None, None

# 使用示例
security_manager = ModelSecurityManager()

# 生成API密钥
api_key = security_manager.generate_api_key()
print(f"生成的API密钥: {api_key}")

# 保护模型
signed_model_path, model_hash = protect_model_weights('model.pt', security_manager)
```

### 6. 如何实现模型的配置管理？
```python
import os
import yaml
import json
from typing import Dict, Any, Optional
from dataclasses import dataclass, asdict
from pathlib import Path

@dataclass
class ModelConfig:
    """模型配置数据类"""
    # 模型配置
    model_path: str
    input_shape: list
    output_shape: list
    framework: str = "PyTorch"
    
    # 推理配置
    batch_size: int = 1
    num_workers: int = 4
    device: str = "cuda"
    precision: str = "fp32"
    
    # 性能配置
    max_concurrent_requests: int = 100
    timeout_seconds: int = 30
    cache_enabled: bool = True
    cache_size: int = 1000
    
    # 监控配置
    metrics_enabled: bool = True
    log_level: str = "INFO"
    log_file: str = "model_service.log"
    
    # 安全配置
    api_key_required: bool = True
    rate_limit_enabled: bool = True
    max_requests_per_minute: int = 1000
    
    # 版本配置
    model_version: str = "1.0.0"
    environment: str = "production"

class ConfigManager:
    def __init__(self, config_path: str = None):
        self.config_path = config_path or self._find_config_file()
        self.config = self._load_config()
        self._validate_config()
    
    def _find_config_file(self) -> str:
        """查找配置文件"""
        possible_paths = [
            'config.yaml',
            'config.yml',
            'model_config.yaml',
            'conf/config.yaml',
            '/etc/model-service/config.yaml'
        ]
        
        for path in possible_paths:
            if os.path.exists(path):
                return path
        
        # 如果没有找到，创建默认配置
        return self._create_default_config()
    
    def _create_default_config(self) -> str:
        """创建默认配置文件"""
        default_config = ModelConfig(
            model_path="./models/model.pt",
            input_shape=[1, 3, 224, 224],
            output_shape=[1, 1000]
        )
        
        config_path = 'config.yaml'
        self._save_config(asdict(default_config), config_path)
        return config_path
    
    def _load_config(self) -> ModelConfig:
        """加载配置"""
        try:
            if self.config_path.endswith('.yaml') or self.config_path.endswith('.yml'):
                with open(self.config_path, 'r') as f:
                    config_dict = yaml.safe_load(f)
            else:
                with open(self.config_path, 'r') as f:
                    config_dict = json.load(f)
            
            # 环境变量覆盖
            config_dict = self._apply_env_overrides(config_dict)
            
            return ModelConfig(**config_dict)
        except Exception as e:
            print(f"加载配置失败，使用默认配置: {e}")
            return ModelConfig(
                model_path="./models/model.pt",
                input_shape=[1, 3, 224, 224],
                output_shape=[1, 1000]
            )
    
    def _apply_env_overrides(self, config_dict: Dict[str, Any]) -> Dict[str, Any]:
        """应用环境变量覆盖"""
        env_prefix = 'MODEL_'
        
        for key, value in os.environ.items():
            if key.startswith(env_prefix):
                config_key = key[len(env_prefix):].lower()
                if config_key in config_dict:
                    # 尝试转换类型
                    if isinstance(config_dict[config_key], bool):
                        config_dict[config_key] = value.lower() in ('true', '1', 'yes')
                    elif isinstance(config_dict[config_key], int):
                        config_dict[config_key] = int(value)
                    elif isinstance(config_dict[config_key], float):
                        config_dict[config_key] = float(value)
                    else:
                        config_dict[config_key] = value
        
        return config_dict
    
    def _validate_config(self):
        """验证配置"""
        if not os.path.exists(self.config.model_path):
            raise FileNotFoundError(f"模型文件不存在: {self.config.model_path}")
        
        if self.config.batch_size <= 0:
            raise ValueError("批大小必须大于0")
        
        if self.config.timeout_seconds <= 0:
            raise ValueError("超时时间必须大于0")
    
    def _save_config(self, config_dict: Dict[str, Any], path: str):
        """保存配置"""
        if path.endswith('.yaml') or path.endswith('.yml'):
            with open(path, 'w') as f:
                yaml.dump(config_dict, f, default_flow_style=False)
        else:
            with open(path, 'w') as f:
                json.dump(config_dict, f, indent=2)
    
    def get_config(self) -> ModelConfig:
        """获取配置"""
        return self.config
    
    def update_config(self, updates: Dict[str, Any]):
        """更新配置"""
        config_dict = asdict(self.config)
        config_dict.update(updates)
        
        try:
            new_config = ModelConfig(**config_dict)
            self.config = new_config
            self._save_config(config_dict, self.config_path)
            self._validate_config()
        except Exception as e:
            raise ValueError(f"配置更新失败: {e}")
    
    def reload_config(self):
        """重新加载配置"""
        self.config = self._load_config()
        print("配置已重新加载")

# 配置文件示例 (config.yaml)
"""
model_path: "./models/resnet50.pt"
input_shape: [1, 3, 224, 224]
output_shape: [1, 1000]
framework: "PyTorch"

batch_size: 32
num_workers: 4
device: "cuda"
precision: "fp16"

max_concurrent_requests: 200
timeout_seconds: 10
cache_enabled: true
cache_size: 5000

metrics_enabled: true
log_level: "INFO"
log_file: "/var/log/model_service.log"

api_key_required: true
rate_limit_enabled: true
max_requests_per_minute: 2000

model_version: "2.1.0"
environment: "production"
"""

# 使用示例
config_manager = ConfigManager('config.yaml')
config = config_manager.get_config()

print(f"模型路径: {config.model_path}")
print(f"输入形状: {config.input_shape}")
print(f"批大小: {config.batch_size}")
```

### 7. 如何实现模型的错误处理和恢复？
```python
import traceback
import logging
from typing import Optional, Dict, Any
from functools import wraps
import time

class ModelError(Exception):
    """模型相关错误基类"""
    pass

class ModelLoadError(ModelError):
    """模型加载错误"""
    pass

class ModelInferenceError(ModelError):
    """模型推理错误"""
    pass

class ModelTimeoutError(ModelError):
    """模型超时错误"""
    pass

class ModelRecoveryManager:
    def __init__(self, max_retries: int = 3, retry_delay: float = 1.0):
        self.max_retries = max_retries
        self.retry_delay = retry_delay
        self.failure_count = 0
        self.last_failure_time = None
        self.fallback_model = None
        self.logger = logging.getLogger(__name__)
    
    def set_fallback_model(self, model):
        """设置备用模型"""
        self.fallback_model = model
        self.logger.info("备用模型已设置")
    
    def handle_model_error(self, error: Exception, context: str = "") -> Optional[Any]:
        """处理模型错误"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        error_info = {
            'error_type': type(error).__name__,
            'error_message': str(error),
            'context': context,
            'failure_count': self.failure_count,
            'timestamp': self.last_failure_time
        }
        
        self.logger.error(f"模型错误 [{context}]: {error}", 
                         extra={'error_info': error_info})
        
        # 记录详细堆栈跟踪
        self.logger.debug(f"堆栈跟踪: {traceback.format_exc()}")
        
        # 尝试使用备用模型
        if self.fallback_model:
            self.logger.info("尝试使用备用模型")
            return self.fallback_model
        
        return None
    
    def should_circuit_break(self, failure_threshold: int = 5, 
                           reset_timeout: int = 300) -> bool:
        """检查是否应该断路"""
        if self.failure_count < failure_threshold:
            return False
        
        if self.last_failure_time is None:
            return False
        
        # 检查是否过了重置时间
        return (time.time() - self.last_failure_time) < reset_timeout
    
    def reset_failure_count(self):
        """重置失败计数"""
        self.failure_count = 0
        self.last_failure_time = None
        self.logger.info("失败计数已重置")

def retry_on_failure(max_retries: int = 3, delay: float = 1.0, 
                    exceptions=(Exception,)):
    """重试装饰器"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries + 1):
                try:
                    result = func(*args, **kwargs)
                    if attempt > 0:
                        logging.info(f"函数 {func.__name__} 在第 {attempt + 1} 次尝试后成功")
                    return result
                except exceptions as e:
                    last_exception = e
                    if attempt < max_retries:
                        logging.warning(f"函数 {func.__name__} 第 {attempt + 1} 次尝试失败: {e}")
                        time.sleep(delay * (2 ** attempt))  # 指数退避
                    else:
                        logging.error(f"函数 {func.__name__} 所有 {max_retries + 1} 次尝试都失败了")
                        raise last_exception
            
            raise last_exception
        return wrapper
    return decorator

def circuit_breaker(failure_threshold: int = 5, timeout: int = 60):
    """断路器装饰器"""
    def decorator(func):
        failure_count = 0
        last_failure_time = None
        state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            nonlocal failure_count, last_failure_time, state
            
            # 检查断路器状态
            if state == 'OPEN':
                if time.time() - last_failure_time > timeout:
                    state = 'HALF_OPEN'
                    logging.info("断路器进入半开状态")
                else:
                    raise ModelError("断路器已打开，服务不可用")
            
            try:
                result = func(*args, **kwargs)
                
                # 成功后重置状态
                if state in ['OPEN', 'HALF_OPEN']:
                    failure_count = 0
                    state = 'CLOSED'
                    logging.info("断路器已关闭")
                
                return result
                
            except Exception as e:
                failure_count += 1
                last_failure_time = time.time()
                
                if failure_count >= failure_threshold:
                    state = 'OPEN'
                    logging.error(f"断路器已打开，失败次数: {failure_count}")
                
                raise e
        
        return wrapper
    return decorator

class GracefulShutdown:
    """优雅关闭管理器"""
    def __init__(self):
        self.shutdown_handlers = []
        self.is_shutting_down = False
        self.setup_signal_handlers()
    
    def setup_signal_handlers(self):
        """设置信号处理器"""
        import signal
        
        def signal_handler(signum, frame):
            self.shutdown()
        
        signal.signal(signal.SIGTERM, signal_handler)
        signal.signal(signal.SIGINT, signal_handler)
    
    def register_shutdown_handler(self, handler):
        """注册关闭处理器"""
        self.shutdown_handlers.append(handler)
    
    def shutdown(self):
        """执行优雅关闭"""
        if self.is_shutting_down:
            return
        
        self.is_shutting_down = True
        logging.info("开始优雅关闭...")
        
        # 执行所有关闭处理器
        for handler in self.shutdown_handlers:
            try:
                handler()
            except Exception as e:
                logging.error(f"关闭处理器执行失败: {e}")
        
        logging.info("优雅关闭完成")

# 使用示例
recovery_manager = ModelRecoveryManager(max_retries=3, retry_delay=1.0)
graceful_shutdown = GracefulShutdown()

@retry_on_failure(max_retries=3, delay=1.0)
@circuit_breaker(failure_threshold=5, timeout=60)
def predict_with_error_handling(model, inputs, timeout: int = 30):
    """带错误处理的预测函数"""
    try:
        import signal
        
        def timeout_handler(signum, frame):
            raise ModelTimeoutError(f"预测超时 ({timeout}秒)")
        
        # 设置超时
        signal.signal(signal.SIGALRM, timeout_handler)
        signal.alarm(timeout)
        
        try:
            with torch.no_grad():
                result = model(inputs)
            return result
        finally:
            signal.alarm(0)  # 取消超时
            
    except Exception as e:
        # 处理错误并尝试恢复
        fallback_result = recovery_manager.handle_model_error(e, "模型预测")
        if fallback_result is not None:
            return fallback_result
        else:
            raise

# 注册关闭处理器
def cleanup_handler():
    """清理资源"""
    logging.info("清理模型资源...")
    # 清理GPU内存等

graceful_shutdown.register_shutdown_handler(cleanup_handler)
```

### 8. 如何实现模型的负载均衡？
```python
import random
import time
from typing import List, Dict, Any
from abc import ABC, abstractmethod
import threading

class LoadBalancerStrategy(ABC):
    """负载均衡策略抽象基类"""
    
    @abstractmethod
    def select_model(self, models: List[Any]) -> Any:
        """选择模型"""
        pass

class RoundRobinStrategy(LoadBalancerStrategy):
    """轮询策略"""
    
    def __init__(self):
        self.current_index = 0
        self.lock = threading.Lock()
    
    def select_model(self, models: List[Any]) -> Any:
        with self.lock:
            model = models[self.current_index]
            self.current_index = (self.current_index + 1) % len(models)
            return model

class LeastConnectionsStrategy(LoadBalancerStrategy):
    """最少连接策略"""
    
    def __init__(self):
        self.connection_counts = {}
        self.lock = threading.Lock()
    
    def select_model(self, models: List[Any]) -> Any:
        with self.lock:
            # 初始化连接计数
            for model in models:
                if id(model) not in self.connection_counts:
                    self.connection_counts[id(model)] = 0
            
            # 选择连接数最少的模型
            min_connections = min(self.connection_counts[id(model)] for model in models)
            for model in models:
                if self.connection_counts[id(model)] == min_connections:
                    self.connection_counts[id(model)] += 1
                    return model

class WeightedRoundRobinStrategy(LoadBalancerStrategy):
    """加权轮询策略"""
    
    def __init__(self, weights: List[int]):
        self.weights = weights
        self.current_weights = weights.copy()
        self.current_index = 0
        self.lock = threading.Lock()
    
    def select_model(self, models: List[Any]) -> Any:
        if len(models) != len(self.weights):
            raise ValueError("模型数量和权重数量不匹配")
        
        with self.lock:
            # 找到权重最大的模型
            max_weight = max(self.current_weights)
            selected_index = self.current_weights.index(max_weight)
            
            # 更新权重
            self.current_weights[selected_index] -= sum(self.weights)
            for i in range(len(self.current_weights)):
                self.current_weights[i] += self.weights[i]
            
            return models[selected_index]

class ModelLoadBalancer:
    def __init__(self, models: List[Any], strategy: LoadBalancerStrategy):
        self.models = models
        self.strategy = strategy
        self.model_stats = {id(model): {'requests': 0, 'errors': 0, 'avg_latency': 0} 
                           for model in models}
        self.lock = threading.Lock()
    
    def predict(self, inputs, timeout: int = 30) -> Any:
        """负载均衡预测"""
        start_time = time.time()
        
        try:
            # 选择模型
            selected_model = self.strategy.select_model(self.models)
            model_id = id(selected_model)
            
            # 更新统计
            with self.lock:
                self.model_stats[model_id]['requests'] += 1
            
            # 执行预测
            with torch.no_grad():
                result = selected_model(inputs)
            
            # 记录延迟
            latency = (time.time() - start_time) * 1000  # 转换为毫秒
            with self.lock:
                stats = self.model_stats[model_id]
                stats['avg_latency'] = (stats['avg_latency'] * (stats['requests'] - 1) + latency) / stats['requests']
            
            return result
            
        except Exception as e:
            # 记录错误
            if 'selected_model' in locals():
                with self.lock:
                    self.model_stats[model_id]['errors'] += 1
            
            raise ModelInferenceError(f"负载均衡预测失败: {e}")
    
    def get_model_stats(self) -> Dict[str, Any]:
        """获取模型统计信息"""
        with self.lock:
            return self.model_stats.copy()
    
    def get_load_distribution(self) -> Dict[str, float]:
        """获取负载分布"""
        stats = self.get_model_stats()
        total_requests = sum(stat['requests'] for stat in stats.values())
        
        if total_requests == 0:
            return {str(model_id): 0 for model_id in stats.keys()}
        
        return {str(model_id): stat['requests'] / total_requests 
                for model_id, stat in stats.items()}

# 异步负载均衡器
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AsyncModelLoadBalancer:
    def __init__(self, models: List[Any], strategy: LoadBalancerStrategy, 
                 max_workers: int = 4):
        self.models = models
        self.strategy = strategy
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
        self.model_stats = {id(model): {'requests': 0, 'errors': 0} 
                           for model in models}
        self.lock = threading.Lock()
    
    async def predict_async(self, inputs, timeout: int = 30) -> Any:
        """异步负载均衡预测"""
        loop = asyncio.get_event_loop()
        
        try:
            # 在线程池中执行预测
            result = await asyncio.wait_for(
                loop.run_in_executor(self.executor, self._sync_predict, inputs),
                timeout=timeout
            )
            return result
        except asyncio.TimeoutError:
            raise ModelTimeoutError(f"异步预测超时 ({timeout}秒)")
    
    def _sync_predict(self, inputs):
        """同步预测（在线程池中执行）"""
        selected_model = self.strategy.select_model(self.models)
        model_id = id(selected_model)
        
        # 更新统计
        with self.lock:
            self.model_stats[model_id]['requests'] += 1
        
        try:
            with torch.no_grad():
                return selected_model(inputs)
        except Exception as e:
            with self.lock:
                self.model_stats[model_id]['errors'] += 1
            raise
    
    def shutdown(self):
        """关闭线程池"""
        self.executor.shutdown(wait=True)

# 使用示例
# 创建多个模型实例
models = [model1, model2, model3]

# 使用轮询策略
round_robin_strategy = RoundRobinStrategy()
load_balancer = ModelLoadBalancer(models, round_robin_strategy)

# 使用加权轮询策略
weights = [3, 2, 1]  # 第一个模型权重最高
weighted_strategy = WeightedRoundRobinStrategy(weights)
weighted_load_balancer = ModelLoadBalancer(models, weighted_strategy)

# 异步负载均衡器
async_load_balancer = AsyncModelLoadBalancer(models, round_robin_strategy)

# 执行预测
try:
    result = load_balancer.predict(inputs)
    print(f"预测结果形状: {result.shape}")
    
    # 查看负载分布
    distribution = load_balancer.get_load_distribution()
    print(f"负载分布: {distribution}")
    
except Exception as e:
    print(f"预测失败: {e}")
```

### 9. 如何实现模型的缓存机制？
```python
import time
import hashlib
from typing import Dict, Any, Optional
from collections import OrderedDict
import threading

class LRUCache:
    """LRU缓存实现"""
    
    def __init__(self, capacity: int = 1000):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = threading.RLock()
        self.hits = 0
        self.misses = 0
    
    def get(self, key: str) -> Optional[Any]:
        """获取缓存项"""
        with self.lock:
            if key in self.cache:
                # 移动到末尾（最近使用）
                self.cache.move_to_end(key)
                self.hits += 1
                return self.cache[key]
            else:
                self.misses += 1
                return None
    
    def put(self, key: str, value: Any, ttl: int = 3600) -> None:
        """放入缓存项"""
        with self.lock:
            # 如果键已存在，先删除
            if key in self.cache:
                del self.cache[key]
            
            # 如果缓存已满，删除最老的项
            if len(self.cache) >= self.capacity:
                self.cache.popitem(last=False)
            
            # 添加新项
            self.cache[key] = {
                'value': value,
                'timestamp': time.time(),
                'ttl': ttl
            }
    
    def cleanup_expired(self) -> int:
        """清理过期项"""
        with self.lock:
            current_time = time.time()
            expired_keys = []
            
            for key, item in self.cache.items():
                if current_time - item['timestamp'] > item['ttl']:
                    expired_keys.append(key)
            
            for key in expired_keys:
                del self.cache[key]
            
            return len(expired_keys)
    
    def get_stats(self) -> Dict[str, Any]:
        """获取缓存统计"""
        with self.lock:
            total_requests = self.hits + self.misses
            hit_rate = self.hits / max(total_requests, 1)
            
            return {
                'hits': self.hits,
                'misses': self.misses,
                'hit_rate': hit_rate,
                'size': len(self.cache),
                'capacity': self.capacity
            }
    
    def clear(self) -> None:
        """清空缓存"""
        with self.lock:
            self.cache.clear()
            self.hits = 0
            self.misses = 0

class ModelCacheManager:
    """模型缓存管理器"""
    
    def __init__(self, cache_capacity: int = 10000, default_ttl: int = 3600):
        self.cache = LRUCache(capacity=cache_capacity)
        self.default_ttl = default_ttl
        self.cache_enabled = True
        self.logger = logging.getLogger(__name__)
    
    def _hash_inputs(self, inputs) -> str:
        """为输入生成哈希值"""
        if isinstance(inputs, torch.Tensor):
            # 使用张量内容生成哈希
            input_bytes = inputs.detach().cpu().numpy().tobytes()
        elif isinstance(inputs, (list, tuple)):
            # 处理输入列表
            input_bytes = str([self._tensor_to_hash(item) for item in inputs]).encode()
        else:
            input_bytes = str(inputs).encode()
        
        return hashlib.md5(input_bytes).hexdigest()
    
    def _tensor_to_hash(self, tensor) -> str:
        """将张量转换为哈希友好的表示"""
        if isinstance(tensor, torch.Tensor):
            return hashlib.md5(tensor.detach().cpu().numpy().tobytes()).hexdigest()
        return str(tensor)
    
    def get_cached_result(self, inputs, ttl: Optional[int] = None) -> Optional[Any]:
        """获取缓存结果"""
        if not self.cache_enabled:
            return None
        
        try:
            key = self._hash_inputs(inputs)
            cached_item = self.cache.get(key)
            
            if cached_item is not None:
                # 检查TTL
                if ttl is None:
                    ttl = self.default_ttl
                
                if time.time() - cached_item['timestamp'] <= ttl:
                    self.logger.debug(f"缓存命中: {key}")
                    return cached_item['value']
                else:
                    # 缓存项过期
                    self.logger.debug(f"缓存项过期: {key}")
            
            return None
        except Exception as e:
            self.logger.warning(f"缓存获取失败: {e}")
            return None
    
    def cache_result(self, inputs, result, ttl: Optional[int] = None) -> None:
        """缓存结果"""
        if not self.cache_enabled:
            return
        
        try:
            key = self._hash_inputs(inputs)
            if ttl is None:
                ttl = self.default_ttl
            
            self.cache.put(key, result, ttl)
            self.logger.debug(f"结果已缓存: {key}")
        except Exception as e:
            self.logger.warning(f"结果缓存失败: {e}")
    
    def enable_cache(self, enabled: bool = True) -> None:
        """启用/禁用缓存"""
        self.cache_enabled = enabled
        if not enabled:
            self.cache.clear()
        self.logger.info(f"缓存已{'启用' if enabled else '禁用'}")
    
    def get_cache_stats(self) -> Dict[str, Any]:
        """获取缓存统计"""
        return self.cache.get_stats()
    
    def cleanup_cache(self) -> int:
        """清理过期缓存"""
        return self.cache.cleanup_expired()
    
    def clear_cache(self) -> None:
        """清空缓存"""
        self.cache.clear()
        self.logger.info("缓存已清空")

# 带缓存的模型包装器
class CachedModel:
    """带缓存的模型包装器"""
    
    def __init__(self, model, cache_manager: ModelCacheManager):
        self.model = model
        self.cache_manager = cache_manager
        self.logger = logging.getLogger(__name__)
    
    def predict(self, inputs, use_cache: bool = True, ttl: Optional[int] = None):
        """带缓存的预测"""
        if use_cache:
            # 尝试从缓存获取结果
            cached_result = self.cache_manager.get_cached_result(inputs, ttl)
            if cached_result is not None:
                return cached_result
        
        # 执行实际预测
        try:
            with torch.no_grad():
                result = self.model(inputs)
            
            # 缓存结果
            if use_cache:
                self.cache_manager.cache_result(inputs, result, ttl)
            
            return result
            
        except Exception as e:
            self.logger.error(f"模型预测失败: {e}")
            raise ModelInferenceError(f"预测失败: {e}")
    
    def batch_predict(self, input_list: List[Any], use_cache: bool = True, 
                     ttl: Optional[int] = None) -> List[Any]:
        """批量预测"""
        results = []
        
        for inputs in input_list:
            result = self.predict(inputs, use_cache, ttl)
            results.append(result)
        
        return results

# 分布式缓存示例（使用Redis）
class RedisModelCache:
    """Redis分布式缓存"""
    
    def __init__(self, redis_host: str = 'localhost', redis_port: int = 6379,
                 default_ttl: int = 3600):
        try:
            import redis
            self.redis_client = redis.Redis(host=redis_host, port=redis_port, decode_responses=False)
            self.default_ttl = default_ttl
            self.logger = logging.getLogger(__name__)
        except ImportError:
            self.logger.error("Redis客户端未安装")
            self.redis_client = None
    
    def _hash_inputs(self, inputs) -> str:
        """为输入生成哈希值"""
        if isinstance(inputs, torch.Tensor):
            input_bytes = inputs.detach().cpu().numpy().tobytes()
        else:
            input_bytes = str(inputs).encode()
        
        return hashlib.sha256(input_bytes).hexdigest()
    
    def get_cached_result(self, inputs) -> Optional[Any]:
        """从Redis获取缓存结果"""
        if not self.redis_client:
            return None
        
        try:
            key = self._hash_inputs(inputs)
            cached_data = self.redis_client.get(key)
            
            if cached_data:
                # 反序列化结果
                import pickle
                result = pickle.loads(cached_data)
                self.logger.debug(f"Redis缓存命中: {key}")
                return result
        except Exception as e:
            self.logger.warning(f"Redis缓存获取失败: {e}")
        
        return None
    
    def cache_result(self, inputs, result, ttl: Optional[int] = None) -> None:
        """将结果缓存到Redis"""
        if not self.redis_client:
            return
        
        try:
            key = self._hash_inputs(inputs)
            if ttl is None:
                ttl = self.default_ttl
            
            # 序列化结果
            import pickle
            serialized_result = pickle.dumps(result)
            
            self.redis_client.setex(key, ttl, serialized_result)
            self.logger.debug(f"结果已缓存到Redis: {key}")
        except Exception as e:
            self.logger.warning(f"Redis结果缓存失败: {e}")

# 使用示例
# 创建缓存管理器
cache_manager = ModelCacheManager(cache_capacity=5000, default_ttl=1800)  # 30分钟TTL

# 创建带缓存的模型
cached_model = CachedModel(model, cache_manager)

# 执行预测（自动使用缓存）
result1 = cached_model.predict(inputs)
result2 = cached_model.predict(inputs)  # 这次会从缓存获取

# 查看缓存统计
stats = cache_manager.get_cache_stats()
print(f"缓存统计: {stats}")

# 清理过期缓存
expired_count = cache_manager.cleanup_cache()
print(f"清理了 {expired_count} 个过期项")
```

### 10. 如何实现模型的A/B测试？
```python
import random
import time
from typing import Dict, List, Any, Callable
from dataclasses import dataclass
from collections import defaultdict
import threading

@dataclass
class ABTestConfig:
    """A/B测试配置"""
    test_name: str
    variants: Dict[str, Dict[str, Any]]  # 变体配置
    traffic_split: Dict[str, float]      # 流量分配比例
    metrics: List[str]                   # 要收集的指标
    duration_hours: int = 24            # 测试持续时间

class ABTestVariant:
    """A/B测试变体"""
    
    def __init__(self, name: str, model, config: Dict[str, Any]):
        self.name = name
        self.model = model
        self.config = config
        self.metrics = defaultdict(list)
        self.lock = threading.Lock()
    
    def predict(self, inputs) -> Any:
        """执行预测并收集指标"""
        start_time = time.time()
        
        try:
            with torch.no_grad():
                result = self.model(inputs)
            
            latency = (time.time() - start_time) * 1000  # 转换为毫秒
            
            # 收集指标
            with self.lock:
                self.metrics['latency'].append(latency)
                self.metrics['requests'].append(1)
            
            return result
            
        except Exception as e:
            with self.lock:
                self.metrics['errors'].append(1)
            raise

class ABTestManager:
    """A/B测试管理器"""
    
    def __init__(self, config: ABTestConfig):
        self.config = config
        self.variants = {}
        self.active = True
        self.start_time = time.time()
        self.logger = logging.getLogger(__name__)
        self.lock = threading.Lock()
    
    def add_variant(self, name: str, model, config: Dict[str, Any] = None) -> None:
        """添加测试变体"""
        if config is None:
            config = {}
        
        variant = ABTestVariant(name, model, config)
        self.variants[name] = variant
        self.logger.info(f"添加A/B测试变体: {name}")
    
    def select_variant(self) -> ABTestVariant:
        """根据流量分配选择变体"""
        if not self.active:
            raise RuntimeError("A/B测试未激活")
        
        # 根据流量分配比例选择变体
        rand_value = random.random()
        cumulative_ratio = 0
        
        for variant_name, ratio in self.config.traffic_split.items():
            cumulative_ratio += ratio
            if rand_value <= cumulative_ratio:
                return self.variants[variant_name]
        
        # 默认返回第一个变体
        return list(self.variants.values())[0]
    
    def predict(self, inputs) -> tuple:
        """执行A/B测试预测"""
        variant = self.select_variant()
        
        try:
            result = variant.predict(inputs)
            return result, variant.name
        except Exception as e:
            self.logger.error(f"变体 {variant.name} 预测失败: {e}")
            raise
    
    def get_results(self) -> Dict[str, Dict[str, float]]:
        """获取测试结果"""
        results = {}
        
        for name, variant in self.variants.items():
            with variant.lock:
                metrics = {}
                
                # 计算平均延迟
                if variant.metrics['latency']:
                    metrics['avg_latency_ms'] = sum(variant.metrics['latency']) / len(variant.metrics['latency'])
                    metrics['p95_latency_ms'] = self._percentile(variant.metrics['latency'], 95)
                    metrics['p99_latency_ms'] = self._percentile(variant.metrics['latency'], 99)
                
                # 计算请求量
                metrics['total_requests'] = sum(variant.metrics['requests'])
                
                # 计算错误率
                total_errors = sum(variant.metrics['errors']) if 'errors' in variant.metrics else 0
                metrics['error_rate'] = total_errors / max(metrics['total_requests'], 1)
                
                # 计算吞吐量
                uptime_hours = (time.time() - self.start_time) / 3600
                metrics['requests_per_hour'] = metrics['total_requests'] / max(uptime_hours, 1)
                
                results[name] = metrics
        
        return results
    
    def _percentile(self, data: List[float], percentile: float) -> float:
        """计算百分位数"""
        if not data:
            return 0
        sorted_data = sorted(data)
        index = int(len(sorted_data) * percentile / 100)
        return sorted_data[min(index, len(sorted_data) - 1)]
    
    def is_test_complete(self) -> bool:
        """检查测试是否完成"""
        elapsed_hours = (time.time() - self.start_time) / 3600
        return elapsed_hours >= self.config.duration_hours
    
    def stop_test(self) -> None:
        """停止测试"""
        self.active = False
        self.logger.info(f"A/B测试 {self.config.test_name} 已停止")
    
    def get_winner(self) -> str:
        """获取获胜变体（基于主要指标）"""
        results = self.get_results()
        
        if not results:
            return "unknown"
        
        # 基于最低平均延迟选择获胜者
        winner = min(results.items(), key=lambda x: x[1].get('avg_latency_ms', float('inf')))
        return winner[0]

# 多层A/B测试管理器
class MultiLayerABTestManager:
    """多层A/B测试管理器"""
    
    def __init__(self):
        self.tests = {}
        self.logger = logging.getLogger(__name__)
    
    def add_test(self, test_name: str, ab_test: ABTestManager) -> None:
        """添加A/B测试"""
        self.tests[test_name] = ab_test
        self.logger.info(f"添加A/B测试: {test_name}")
    
    def predict(self, inputs, test_layers: List[str] = None) -> Dict[str, Any]:
        """在多个测试层上执行预测"""
        if test_layers is None:
            test_layers = list(self.tests.keys())
        
        results = {}
        
        for test_name in test_layers:
            if test_name in self.tests:
                try:
                    result, variant_name = self.tests[test_name].predict(inputs)
                    results[test_name] = {
                        'result': result,
                        'variant': variant_name
                    }
                except Exception as e:
                    self.logger.error(f"测试 {test_name} 执行失败: {e}")
                    results[test_name] = {'error': str(e)}
        
        return results
    
    def get_all_results(self) -> Dict[str, Dict[str, Dict[str, float]]]:
        """获取所有测试结果"""
        return {name: test.get_results() for name, test in self.tests.items()}

# 动态配置A/B测试
class DynamicABTestManager:
    """动态配置A/B测试管理器"""
    
    def __init__(self, config_file: str = None):
        self.config_file = config_file
        self.tests = {}
        self.config_watcher = None
        
        if config_file:
            self.load_config_from_file()
            self.start_config_watching()
    
    def load_config_from_file(self) -> None:
        """从文件加载配置"""
        try:
            with open(self.config_file, 'r') as f:
                if self.config_file.endswith('.yaml'):
                    import yaml
                    config_data = yaml.safe_load(f)
                else:
                    import json
                    config_data = json.load(f)
            
            self.update_tests_from_config(config_data)
            
        except Exception as e:
            logging.error(f"加载配置文件失败: {e}")
    
    def update_tests_from_config(self, config_data: Dict[str, Any]) -> None:
        """根据配置更新测试"""
        for test_name, test_config in config_data.get('ab_tests', {}).items():
            ab_config = ABTestConfig(
                test_name=test_name,
                variants=test_config.get('variants', {}),
                traffic_split=test_config.get('traffic_split', {}),
                metrics=test_config.get('metrics', ['latency', 'accuracy']),
                duration_hours=test_config.get('duration_hours', 24)
            )
            
            if test_name not in self.tests:
                self.tests[test_name] = ABTestManager(ab_config)
                logging.info(f"创建A/B测试: {test_name}")
            else:
                # 更新现有测试配置
                self.tests[test_name].config = ab_config
                logging.info(f"更新A/B测试配置: {test_name}")
    
    def start_config_watching(self) -> None:
        """开始监控配置文件变化"""
        import threading
        import os
        
        def watch_config():
            if not self.config_file:
                return
            
            last_modified = os.path.getmtime(self.config_file)
            
            while True:
                try:
                    current_modified = os.path.getmtime(self.config_file)
                    if current_modified > last_modified:
                        logging.info("检测到配置文件变化，重新加载...")
                        self.load_config_from_file()
                        last_modified = current_modified
                    time.sleep(5)
                except Exception as e:
                    logging.error(f"配置监控失败: {e}")
                    time.sleep(30)
        
        self.config_watcher = threading.Thread(target=watch_config, daemon=True)
        self.config_watcher.start()

# 配置文件示例 (ab_tests.yaml)
"""
ab_tests:
  model_architecture_test:
    variants:
      resnet50:
        model_path: "./models/resnet50.pt"
        batch_size: 32
      efficientnet:
        model_path: "./models/efficientnet.pt"
        batch_size: 64
    traffic_split:
      resnet50: 0.5
      efficientnet: 0.5
    metrics:
      - latency
      - accuracy
      - memory_usage
    duration_hours: 48

  preprocessing_test:
    variants:
      standard:
        preprocessing: "standard"
      enhanced:
        preprocessing: "enhanced"
    traffic_split:
      standard: 0.7
      enhanced: 0.3
    metrics:
      - latency
      - accuracy
    duration_hours: 24
"""

# 使用示例
# 创建A/B测试配置
config = ABTestConfig(
    test_name="model_performance_test",
    variants={
        "baseline": {"model_type": "resnet50"},
        "optimized": {"model_type": "efficientnet"}
    },
    traffic_split={"baseline": 0.5, "optimized": 0.5},
    metrics=["latency", "accuracy"],
    duration_hours=24
)

# 创建A/B测试管理器
ab_test_manager = ABTestManager(config)

# 添加模型变体
ab_test_manager.add_variant("baseline", baseline_model)
ab_test_manager.add_variant("optimized", optimized_model)

# 执行A/B测试预测
try:
    result, variant_used = ab_test_manager.predict(inputs)
    print(f"使用变体: {variant_used}")
    print(f"预测结果形状: {result.shape}")
    
    # 定期检查结果
    if ab_test_manager.is_test_complete():
        results = ab_test_manager.get_results()
        winner = ab_test_manager.get_winner()
        print(f"测试完成，获胜变体: {winner}")
        print(f"详细结果: {results}")
        
        ab_test_manager.stop_test()
        
except Exception as e:
    print(f"A/B测试执行失败: {e}")
```

### 11. 如何实现模型的监控和告警？
```python
import time
import logging
from typing import Dict, List, Any, Callable
from dataclasses import dataclass
from collections import defaultdict, deque
import threading
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

@dataclass
class AlertRule:
    """告警规则"""
    name: str
    metric: str
    threshold: float
    operator: str  # 'gt', 'lt', 'eq'
    duration_seconds: int = 300  # 持续时间
    severity: str = "warning"  # 'info', 'warning', 'critical'
    enabled: bool = True

class MetricsCollector:
    """指标收集器"""
    
    def __init__(self, window_size: int = 1000):
        self.window_size = window_size
        self.metrics = defaultdict(lambda: deque(maxlen=window_size))
        self.lock = threading.Lock()
        self.logger = logging.getLogger(__name__)
    
    def record_metric(self, name: str, value: float, tags: Dict[str, str] = None) -> None:
        """记录指标"""
        if tags is None:
            tags = {}
        
        metric_data = {
            'value': value,
            'timestamp': time.time(),
            'tags': tags
        }
        
        with self.lock:
            self.metrics[name].append(metric_data)
    
    def get_metric_stats(self, name: str, window_seconds: int = None) -> Dict[str, float]:
        """获取指标统计"""
        with self.lock:
            if name not in self.metrics:
                return {}
            
            metrics = self.metrics[name]
            if not metrics:
                return {}
            
            # 过滤时间窗口内的数据
            if window_seconds:
                cutoff_time = time.time() - window_seconds
                filtered_metrics = [m for m in metrics if m['timestamp'] >= cutoff_time]
            else:
                filtered_metrics = list(metrics)
            
            if not filtered_metrics:
                return {}
            
            values = [m['value'] for m in filtered_metrics]
            
            return {
                'count': len(values),
                'avg': sum(values) / len(values),
                'min': min(values),
                'max': max(values),
                'latest': values[-1] if values else None
            }
    
    def get_recent_metrics(self, name: str, limit: int = 100) -> List[Dict[str, Any]]:
        """获取最近的指标数据"""
        with self.lock:
            if name not in self.metrics:
                return []
            return list(self.metrics[name])[-limit:]

class AlertManager:
    """告警管理器"""
    
    def __init__(self, metrics_collector: MetricsCollector):
        self.metrics_collector = metrics_collector
        self.alert_rules = []
        self.active_alerts = {}
        self.notification_channels = []
        self.logger = logging.getLogger(__name__)
        self.lock = threading.Lock()
    
    def add_alert_rule(self, rule: AlertRule) -> None:
        """添加告警规则"""
        self.alert_rules.append(rule)
        self.logger.info(f"添加告警规则: {rule.name}")
    
    def add_notification_channel(self, channel: Callable[[str, str, str], None]) -> None:
        """添加通知渠道"""
        self.notification_channels.append(channel)
    
    def check_alerts(self) -> None:
        """检查告警"""
        current_time = time.time()
        
        for rule in self.alert_rules:
            if not rule.enabled:
                continue
            
            try:
                stats = self.metrics_collector.get_metric_stats(
                    rule.metric, rule.duration_seconds
                )
                
                if not stats:
                    continue
                
                current_value = stats.get('latest')
                if current_value is None:
                    continue
                
                # 检查告警条件
                alert_triggered = False
                if rule.operator == 'gt' and current_value > rule.threshold:
                    alert_triggered = True
                elif rule.operator == 'lt' and current_value < rule.threshold:
                    alert_triggered = True
                elif rule.operator == 'eq' and current_value == rule.threshold:
                    alert_triggered = True
                
                # 处理告警状态
                with self.lock:
                    alert_key = f"{rule.name}_{rule.metric}"
                    
                    if alert_triggered:
                        if alert_key not in self.active_alerts:
                            # 新告警
                            self.active_alerts[alert_key] = {
                                'rule': rule,
                                'first_triggered': current_time,
                                'last_notified': 0
                            }
                            self._send_alert(rule, current_value, "触发")
                    else:
                        if alert_key in self.active_alerts:
                            # 告警恢复
                            del self.active_alerts[alert_key]
                            self._send_alert(rule, current_value, "恢复")
                            
            except Exception as e:
                self.logger.error(f"检查告警规则 {rule.name} 时出错: {e}")
    
    def _send_alert(self, rule: AlertRule, current_value: float, status: str) -> None:
        """发送告警"""
        message = (
            f"告警{status}: {rule.name}\n"
            f"指标: {rule.metric}\n"
            f"当前值: {current_value}\n"
            f"阈值: {rule.threshold}\n"
            f"严重级别: {rule.severity}\n"
            f"时间: {time.strftime('%Y-%m-%d %H:%M:%S')}"
        )
        
        subject = f"[{rule.severity.upper()}] 模型服务告警 - {rule.name} {status}"
        
        for channel in self.notification_channels:
            try:
                channel(subject, message, rule.severity)
            except Exception as e:
                self.logger.error(f"发送告警通知失败: {e}")
    
    def get_active_alerts(self) -> Dict[str, Any]:
        """获取活跃告警"""
        with self.lock:
            return self.active_alerts.copy()

class ModelMonitor:
    """模型监控器"""
    
    def __init__(self, model, metrics_collector: MetricsCollector):
        self.model = model
        self.metrics_collector = metrics_collector
        self.logger = logging.getLogger(__name__)
        self.prediction_count = 0
        self.error_count = 0
        self.total_latency = 0
    
    def predict_with_monitoring(self, inputs, timeout: int = 30) -> Any:
        """带监控的预测"""
        start_time = time.time()
        self.prediction_count += 1
        
        try:
            # 执行预测
            with torch.no_grad():
                result = self.model(inputs)
            
            # 计算延迟
            latency = (time.time() - start_time) * 1000  # 转换为毫秒
            self.total_latency += latency
            
            # 记录指标
            self.metrics_collector.record_metric('prediction_latency_ms', latency)
            self.metrics_collector.record_metric('prediction_count', 1)
            
            # 记录吞吐量
            throughput = 1000 / max(latency, 1)  # 每秒请求数
            self.metrics_collector.record_metric('throughput_rps', throughput)
            
            return result
            
        except Exception as e:
            self.error_count += 1
            self.metrics_collector.record_metric('prediction_errors', 1)
            self.logger.error(f"预测失败: {e}")
            raise
    
    def get_performance_stats(self, window_seconds: int = 3600) -> Dict[str, float]:
        """获取性能统计"""
        stats = self.metrics_collector.get_metric_stats('prediction_latency_ms', window_seconds)
        
        total_predictions = self.prediction_count
        error_rate = self.error_count / max(total_predictions, 1)
        
        avg_latency = self.total_latency / max(total_predictions, 1) if total_predictions > 0 else 0
        
        return {
            **stats,
            'avg_latency_overall_ms': avg_latency,
            'total_predictions': total_predictions,
            'error_rate': error_rate,
            'success_rate': 1 - error_rate
        }

# 通知渠道实现
def email_notification(subject: str, message: str, severity: str, 
                      smtp_config: Dict[str, str] = None) -> None:
    """邮件通知"""
    if smtp_config is None:
        return  # 需要配置SMTP信息
    
    try:
        msg = MIMEMultipart()
        msg['From'] = smtp_config.get('from_email')
        msg['To'] = smtp_config.get('to_email')
        msg['Subject'] = subject
        
        msg.attach(MIMEText(message, 'plain'))
        
        server = smtplib.SMTP(smtp_config.get('smtp_server'), smtp_config.get('smtp_port'))
        server.starttls()
        server.login(smtp_config.get('username'), smtp_config.get('password'))
        server.send_message(msg)
        server.quit()
        
    except Exception as e:
        logging.error(f"邮件通知发送失败: {e}")

def slack_notification(subject: str, message: str, severity: str,
                      webhook_url: str = None) -> None:
    """Slack通知"""
    if not webhook_url:
        return
    
    try:
        import requests
        
        payload = {
            'text': f"*{subject}*\n{message}",
            'attachments': [{
                'color': 'danger' if severity == 'critical' else 'warning' if severity == 'warning' else 'good',
                'fields': [
                    {'title': 'Severity', 'value': severity, 'short': True},
                    {'title': 'Time', 'value': time.strftime('%Y-%m-%d %H:%M:%S'), 'short': True}
                ]
            }]
        }
        
        requests.post(webhook_url, json=payload)
        
    except Exception as e:
        logging.error(f"Slack通知发送失败: {e}")

# 监控仪表板
class MonitoringDashboard:
    """监控仪表板"""
    
    def __init__(self, metrics_collector: MetricsCollector, model_monitor: ModelMonitor):
        self.metrics_collector = metrics_collector
        self.model_monitor = model_monitor
        self.logger = logging.getLogger(__name__)
    
    def get_dashboard_data(self) -> Dict[str, Any]:
        """获取仪表板数据"""
        # 获取最近的性能指标
        recent_latency = self.metrics_collector.get_recent_metrics('prediction_latency_ms', 100)
        recent_throughput = self.metrics_collector.get_recent_metrics('throughput_rps', 100)
        
        # 获取性能统计
        perf_stats = self.model_monitor.get_performance_stats(3600)  # 1小时窗口
        
        # 获取系统指标
        system_metrics = self._get_system_metrics()
        
        return {
            'performance': perf_stats,
            'recent_latency': recent_latency,
            'recent_throughput': recent_throughput,
            'system': system_metrics,
            'timestamp': time.time()
        }
    
    def _get_system_metrics(self) -> Dict[str, float]:
        """获取系统指标"""
        try:
            import psutil
            
            cpu_percent = psutil.cpu_percent(interval=1)
            memory_percent = psutil.virtual_memory().percent
            
            # GPU指标（如果可用）
            gpu_metrics = {}
            if torch.cuda.is_available():
                gpu_metrics = {
                    'gpu_memory_allocated_gb': torch.cuda.memory_allocated() / 1024**3,
                    'gpu_memory_reserved_gb': torch.cuda.memory_reserved() / 1024**3
                }
            
            return {
                'cpu_percent': cpu_percent,
                'memory_percent': memory_percent,
                **gpu_metrics
            }
        except Exception as e:
            self.logger.error(f"获取系统指标失败: {e}")
            return {}

# 使用示例
# 创建指标收集器和监控器
metrics_collector = MetricsCollector(window_size=10000)
model_monitor = ModelMonitor(model, metrics_collector)

# 创建告警管理器
alert_manager = AlertManager(metrics_collector)

# 添加告警规则
alert_manager.add_alert_rule(AlertRule(
    name="高延迟告警",
    metric="prediction_latency_ms",
    threshold=1000,  # 1秒
    operator="gt",
    duration_seconds=300,  # 5分钟
    severity="critical"
))

alert_manager.add_alert_rule(AlertRule(
    name="高错误率告警",
    metric="prediction_errors",
    threshold=0.05,  # 5%错误率
    operator="gt",
    duration_seconds=600,  # 10分钟
    severity="warning"
))

# 添加通知渠道
smtp_config = {
    'smtp_server': 'smtp.gmail.com',
    'smtp_port': 587,
    'username': 'your-email@gmail.com',
    'password': 'your-password',
    'from_email': 'your-email@gmail.com',
    'to_email': 'alert-recipient@gmail.com'
}

# alert_manager.add_notification_channel(
#     lambda subject, message, severity: email_notification(subject, message, severity, smtp_config)
# )

# 执行带监控的预测
try:
    result = model_monitor.predict_with_monitoring(inputs)
    print(f"预测完成，结果形状: {result.shape}")
    
    # 定期检查告警
    alert_manager.check_alerts()
    
    # 获取性能统计
    perf_stats = model_monitor.get_performance_stats()
    print(f"性能统计: {perf_stats}")
    
except Exception as e:
    print(f"预测失败: {e}")
```

### 12. 如何实现模型的日志记录和追踪？
```python
import logging
import json
import time
import uuid
from typing import Dict, Any, Optional
from functools import wraps
from contextlib import contextmanager
import traceback

class RequestTracer:
    """请求追踪器"""
    
    def __init__(self, logger_name: str = "model_service"):
        self.logger = logging.getLogger(logger_name)
        self.setup_logger()
    
    def setup_logger(self):
        """设置日志记录器"""
        # 创建格式化器
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        
        # 文件处理器
        file_handler = logging.FileHandler('model_service.log')
        file_handler.setFormatter(formatter)
        self.logger.addHandler(file_handler)
        
        # 控制台处理器
        console_handler = logging.StreamHandler()
        console_handler.setFormatter(formatter)
        self.logger.addHandler(console_handler)
        
        self.logger.setLevel(logging.INFO)
    
    @contextmanager
    def trace_request(self, request_id: str = None, operation: str = "predict"):
        """请求追踪上下文管理器"""
        if request_id is None:
            request_id = str(uuid.uuid4())
        
        start_time = time.time()
        
        # 记录请求开始
        self.logger.info(f"Request {request_id} - {operation} started")
        
        try:
            yield request_id
        except Exception as e:
            # 记录异常
            self.logger.error(
                f"Request {request_id} - {operation} failed: {str(e)}",
                extra={'exception': traceback.format_exc()}
            )
            raise
        finally:
            # 记录请求结束
            duration = (time.time() - start_time) * 1000  # 转换为毫秒
            self.logger.info(f"Request {request_id} - {operation} completed in {duration:.2f}ms")

def trace_model_operations(tracer: RequestTracer):
    """模型操作追踪装饰器"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            operation_name = func.__name__
            
            with tracer.trace_request(operation=operation_name) as request_id:
                # 添加请求ID到关键字参数中（如果需要）
                kwargs['_request_id'] = request_id
                
                # 执行函数
                result = func(*args, **kwargs)
                
                # 记录结果信息
                if hasattr(result, 'shape'):
                    tracer.logger.info(
                        f"Request {request_id} - {operation_name} result shape: {result.shape}"
                    )
                
                return result
        return wrapper
    return decorator

class StructuredLogger:
    """结构化日志记录器"""
    
    def __init__(self, logger_name: str = "structured_logger"):
        self.logger = logging.getLogger(logger_name)
        self.setup_structured_logging()
    
    def setup_structured_logging(self):
        """设置结构化日志"""
        class JSONFormatter(logging.Formatter):
            def format(self, record):
                log_entry = {
                    'timestamp': self.formatTime(record),
                    'level': record.levelname,
                    'logger': record.name,
                    'message': record.getMessage(),
                    'module': record.module,
                    'function': record.funcName,
                    'line': record.lineno
                }
                
                # 添加额外字段
                if hasattr(record, 'request_id'):
                    log_entry['request_id'] = record.request_id
                if hasattr(record, 'model_version'):
                    log_entry['model_version'] = record.model_version
                if hasattr(record, 'latency_ms'):
                    log_entry['latency_ms'] = record.latency_ms
                if hasattr(record, 'exception'):
                    log_entry['exception'] = record.exception
                
                return json.dumps(log_entry)
        
        handler = logging.StreamHandler()
        handler.setFormatter(JSONFormatter())
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)

class ModelAuditLogger:
    """模型审计日志记录器"""
    
    def __init__(self, audit_file: str = "model_audit.log"):
        self.audit_file = audit_file
        self.logger = logging.getLogger("model_audit")
        self.setup_audit_logging()
    
    def setup_audit_logging(self):
        """设置审计日志"""
        handler = logging.FileHandler(self.audit_file)
        formatter = logging.Formatter(
            '%(asctime)s - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def log_model_access(self, user: str, model_version: str, 
                        action: str, request_id: str = None):
        """记录模型访问"""
        audit_entry = {
            'event_type': 'model_access',
            'user': user,
            'model_version': model_version,
            'action': action,
            'request_id': request_id,
            'timestamp': time.time()
        }
        
        self.logger.info(json.dumps(audit_entry))
    
    def log_model_deployment(self, model_version: str, deployed_by: str,
                           deployment_info: Dict[str, Any]):
        """记录模型部署"""
        audit_entry = {
            'event_type': 'model_deployment',
            'model_version': model_version,
            'deployed_by': deployed_by,
            'deployment_info': deployment_info,
            'timestamp': time.time()
        }
        
        self.logger.info(json.dumps(audit_entry))
    
    def log_model_update(self, model_version: str, updated_by: str,
                        update_info: Dict[str, Any]):
        """记录模型更新"""
        audit_entry = {
            'event_type': 'model_update',
            'model_version': model_version,
            'updated_by': updated_by,
            'update_info': update_info,
            'timestamp': time.time()
        }
        
        self.logger.info(json.dumps(audit_entry))

class PerformanceTracer:
    """性能追踪器"""
    
    def __init__(self):
        self.traces = []
        self.logger = logging.getLogger("performance_tracer")
    
    @contextmanager
    def trace_operation(self, operation_name: str, request_id: str = None):
        """追踪操作性能"""
        start_time = time.time()
        start_memory = self._get_memory_usage()
        
        self.logger.info(f"Operation {operation_name} started")
        
        try:
            yield
        finally:
            end_time = time.time()
            end_memory = self._get_memory_usage()
            
            duration = (end_time - start_time) * 1000  # 转换为毫秒
            memory_diff = end_memory - start_memory
            
            trace_info = {
                'operation': operation_name,
                'request_id': request_id,
                'duration_ms': duration,
                'memory_diff_mb': memory_diff,
                'start_time': start_time,
                'end_time': end_time
            }
            
            self.traces.append(trace_info)
            self.logger.info(
                f"Operation {operation_name} completed in {duration:.2f}ms, "
                f"memory change: {memory_diff:.2f}MB"
            )
    
    def _get_memory_usage(self) -> float:
        """获取内存使用量（MB）"""
        try:
            import psutil
            process = psutil.Process()
            return process.memory_info().rss / 1024 / 1024
        except:
            return 0.0
    
    def get_performance_report(self) -> Dict[str, Any]:
        """获取性能报告"""
        if not self.traces:
            return {}
        
        durations = [trace['duration_ms'] for trace in self.traces]
        memory_changes = [trace['memory_diff_mb'] for trace in self.traces]
        
        return {
            'total_operations': len(self.traces),
            'avg_duration_ms': sum(durations) / len(durations),
            'max_duration_ms': max(durations),
            'min_duration_ms': min(durations),
            'avg_memory_change_mb': sum(memory_changes) / len(memory_changes),
            'operations_by_type': self._group_traces_by_operation()
        }
    
    def _group_traces_by_operation(self) -> Dict[str, Dict[str, float]]:
        """按操作类型分组追踪信息"""
        grouped = {}
        for trace in self.traces:
            op_name = trace['operation']
            if op_name not in grouped:
                grouped[op_name] = {
                    'count': 0,
                    'total_duration_ms': 0,
                    'avg_duration_ms': 0
                }
            
            grouped[op_name]['count'] += 1
            grouped[op_name]['total_duration_ms'] += trace['duration_ms']
        
        # 计算平均值
        for op_name, stats in grouped.items():
            stats['avg_duration_ms'] = stats['total_duration_ms'] / stats['count']
        
        return grouped

# 集成所有日志功能的主类
class ComprehensiveModelLogger:
    """综合模型日志记录器"""
    
    def __init__(self, service_name: str = "model_service"):
        self.service_name = service_name
        self.request_tracer = RequestTracer(f"{service_name}_tracer")
        self.structured_logger = StructuredLogger(f"{service_name}_structured")
        self.audit_logger = ModelAuditLogger(f"{service_name}_audit.log")
        self.performance_tracer = PerformanceTracer()
        
        # 设置全局日志记录器
        self.logger = logging.getLogger(service_name)
    
    def log_prediction(self, request_id: str, model_version: str, 
                      inputs_info: Dict[str, Any], result_info: Dict[str, Any],
                      latency_ms: float):
        """记录预测日志"""
        log_data = {
            'event_type': 'prediction',
            'request_id': request_id,
            'model_version': model_version,
            'inputs': inputs_info,
            'result': result_info,
            'latency_ms': latency_ms
        }
        
        self.structured_logger.logger.info(
            f"Prediction completed for request {request_id}",
            extra=log_data
        )
    
    def log_error(self, request_id: str, error_type: str, error_message: str,
                 stack_trace: str = None):
        """记录错误日志"""
        log_data = {
            'event_type': 'error',
            'request_id': request_id,
            'error_type': error_type,
            'error_message': error_message,
            'stack_trace': stack_trace
        }
        
        self.structured_logger.logger.error(
            f"Error in request {request_id}: {error_message}",
            extra=log_data
        )

# 使用示例
# 创建综合日志记录器
comprehensive_logger = ComprehensiveModelLogger("image_classifier")

# 使用追踪装饰器
@trace_model_operations(comprehensive_logger.request_tracer)
def predict_with_tracing(model, inputs, model_version: str = "1.0.0"):
    """带追踪的预测函数"""
    start_time = time.time()
    
    # 记录输入信息
    inputs_info = {
        'shape': list(inputs.shape) if hasattr(inputs, 'shape') else str(type(inputs)),
        'dtype': str(inputs.dtype) if hasattr(inputs, 'dtype') else 'unknown'
    }
    
    try:
        # 执行预测
        with comprehensive_logger.performance_tracer.trace_operation("model_inference"):
            with torch.no_grad():
                result = model(inputs)
        
        # 计算延迟
        latency_ms = (time.time() - start_time) * 1000
        
        # 记录结果信息
        result_info = {
            'shape': list(result.shape) if hasattr(result, 'shape') else str(type(result)),
            'dtype': str(result.dtype) if hasattr(result, 'dtype') else 'unknown'
        }
        
        # 记录预测日志
        request_id = kwargs.get('_request_id', str(uuid.uuid4()))
        comprehensive_logger.log_prediction(
            request_id, model_version, inputs_info, result_info, latency_ms
        )
        
        return result
        
    except Exception as e:
        # 记录错误
        request_id = kwargs.get('_request_id', str(uuid.uuid4()))
        comprehensive_logger.log_error(
            request_id, type(e).__name__, str(e), traceback.format_exc()
        )
        raise

# 执行带完整日志记录的预测
try:
    result = predict_with_tracing(model, inputs, model_version="2.1.0")
    print(f"预测完成，结果形状: {result.shape}")
    
    # 获取性能报告
    perf_report = comprehensive_logger.performance_tracer.get_performance_report()
    print(f"性能报告: {perf_report}")
    
except Exception as e:
    print(f"预测失败: {e}")
```

### 13. 如何实现模型的容器化部署？
```python
# Dockerfile
"""
FROM pytorch/pytorch:1.9.0-cuda11.1-cudnn8-runtime

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY requirements.txt .

# 安装Python依赖
RUN pip install --no-cache-dir -r requirements.txt

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    libglib2.0-0 \
    libsm6 \
    libxext6 \
    libxrender-dev \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# 复制应用代码
COPY . .

# 创建模型目录
RUN mkdir -p models

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 启动命令
CMD ["python", "app.py"]
"""

# requirements.txt
"""
fastapi==0.68.0
uvicorn==0.15.0
torch==1.9.0
torchvision==0.10.0
numpy==1.21.2
pydantic==1.8.2
python-multipart==0.0.5
redis==3.5.3
psutil==5.8.0
"""

# app.py - 主应用文件
import os
import torch
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import logging

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# 创建FastAPI应用
app = FastAPI(title="PyTorch Model Service", version="1.0.0")

# 全局模型变量
model = None
device = None

class PredictionRequest(BaseModel):
    """预测请求模型"""
    data: list
    model_version: str = "latest"

class PredictionResponse(BaseModel):
    """预测响应模型"""
    prediction: list
    confidence: float
    model_version: str
    latency_ms: float

def load_model():
    """加载模型"""
    global model, device
    
    try:
        # 检查模型文件
        model_path = os.getenv("MODEL_PATH", "./models/model.pt")
        if not os.path.exists(model_path):
            raise FileNotFoundError(f"模型文件不存在: {model_path}")
        
        # 设置设备
        device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        logger.info(f"使用设备: {device}")
        
        # 加载模型
        model = torch.load(model_path, map_location=device)
        model.eval()
        
        logger.info(f"模型加载成功: {model_path}")
        return True
        
    except Exception as e:
        logger.error(f"模型加载失败: {e}")
        return False

@app.on_event("startup")
async def startup_event():
    """应用启动事件"""
    logger.info("应用启动中...")
    
    if not load_model():
        raise RuntimeError("模型加载失败，应用无法启动")

@app.get("/health")
async def health_check():
    """健康检查端点"""
    try:
        # 检查模型是否可以推理
        if model is None:
            return {"status": "unhealthy", "error": "模型未加载"}
        
        # 执行简单推理测试
        dummy_input = torch.randn(1, 3, 224, 224).to(device)
        with torch.no_grad():
            _ = model(dummy_input)
        
        return {"status": "healthy", "device": str(device)}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

@app.get("/ready")
async def readiness_check():
    """就绪检查端点"""
    if model is not None:
        return {"status": "ready"}
    else:
        return {"status": "not ready"}, 503

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    """预测端点"""
    import time
    start_time = time.time()
    
    try:
        # 验证输入
        if not request.data:
            raise HTTPException(status_code=400, detail="输入数据为空")
        
        # 转换输入数据
        input_tensor = torch.tensor(request.data, dtype=torch.float32).to(device)
        
        # 执行预测
        with torch.no_grad():
            output = model(input_tensor)
        
        # 处理输出
        prediction = output.argmax(dim=1).tolist()
        confidence = torch.softmax(output, dim=1).max().item()
        
        # 计算延迟
        latency_ms = (time.time() - start_time) * 1000
        
        return PredictionResponse(
            prediction=prediction,
            confidence=confidence,
            model_version=request.model_version,
            latency_ms=latency_ms
        )
        
    except Exception as e:
        logger.error(f"预测失败: {e}")
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "app:app",
        host=os.getenv("HOST", "0.0.0.0"),
        port=int(os.getenv("PORT", 8000)),
        workers=int(os.getenv("WORKERS", 1)),
        log_level=os.getenv("LOG_LEVEL", "info")
    )

# docker-compose.yml
"""
version: '3.8'

services:
  model-service:
    build: .
    ports:
      - "8000:8000"
    environment:
      - MODEL_PATH=/app/models/model.pt
      - HOST=0.0.0.0
      - PORT=8000
      - WORKERS=2
      - LOG_LEVEL=info
    volumes:
      - ./models:/app/models
      - ./logs:/app/logs
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  redis_data:
"""

# Kubernetes部署文件 (deployment.yaml)
"""
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-model-service
  labels:
    app: pytorch-model-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pytorch-model-service
  template:
    metadata:
      labels:
        app: pytorch-model-service
    spec:
      containers:
      - name: model-service
        image: your-registry/pytorch-model-service:latest
        ports:
        - containerPort: 8000
        env:
        - name: MODEL_PATH
          value: "/app/models/model.pt"
        - name: WORKERS
          value: "2"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        volumeMounts:
        - name: model-storage
          mountPath: /app/models
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
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
    app: pytorch-model-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: pytorch-model-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: pytorch-model-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
"""

# 构建和部署脚本 (build.sh)
"""
#!/bin/bash

# 设置变量
IMAGE_NAME="pytorch-model-service"
IMAGE_TAG="latest"
REGISTRY="your-registry.com"

# 构建Docker镜像
echo "构建Docker镜像..."
docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .

# 推送到注册表
echo "推送到注册表..."
docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

echo "构建和推送完成"
"""

# 运行脚本 (run.sh)
"""
#!/bin/bash

# 设置环境变量
export MODEL_PATH="./models/model.pt"
export HOST="0.0.0.0"
export PORT="8000"
export WORKERS="2"

# 运行容器
docker run -d \
  --name pytorch-model-service \
  -p 8000:8000 \
  -v $(pwd)/models:/app/models \
  -v $(pwd)/logs:/app/logs \
  -e MODEL_PATH=$MODEL_PATH \
  -e HOST=$HOST \
  -e PORT=$PORT \
  -e WORKERS=$WORKERS \
  pytorch-model-service:latest

echo "服务已启动，访问 http://localhost:8000 查看"
"""
```

### 14. 如何实现模型的持续集成和部署？
```python
# .github/workflows/ci-cd.yml
"""
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
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
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov
    
    - name: Run unit tests
      run: |
        pytest tests/unit/ -v --cov=src --cov-report=xml
    
    - name: Run integration tests
      run: |
        pytest tests/integration/ -v
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v1
      with:
        file: ./coverage.xml

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1
    
    - name: Login to DockerHub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Extract metadata (tags, labels) for Docker
      id: meta
      uses: docker/metadata-action@v3
      with:
        images: your-username/pytorch-model-service
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=sha
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Deploy to staging
      run: |
        echo "Deploying to staging environment..."
        # 这里添加部署到staging环境的命令
        # 例如使用kubectl部署到K8s集群
        # kubectl set image deployment/pytorch-model-service model-service=your-username/pytorch-model-service:${{ github.sha }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    steps:
    - name: Deploy to production
      run: |
        echo "Deploying to production environment..."
        # 这里添加部署到production环境的命令
        # 需要手动批准或使用特定的部署策略
"""

# Makefile for development
"""
.PHONY: help install test build run clean

help:
	@echo "Available targets:"
	@echo "  install    - Install dependencies"
	@echo "  test       - Run tests"
	@echo "  build      - Build Docker image"
	@echo "  run        - Run service locally"
	@echo "  clean      - Clean build artifacts"

install:
	pip install -r requirements.txt

test:
	pytest tests/ -v

build:
	docker build -t pytorch-model-service .

run:
	docker run -p 8000:8000 pytorch-model-service

clean:
	rm -rf build/
	rm -rf dist/
	find . -type f -name "*.pyc" -delete
	find . -type d -name "__pycache__" -delete
"""

# setup.py for package distribution
"""
from setuptools import setup, find_packages

setup(
    name="pytorch-model-service",
    version="1.0.0",
    packages=find_packages(),
    install_requires=[
        "torch>=1.9.0",
        "torchvision>=0.10.0",
        "fastapi>=0.68.0",
        "uvicorn>=0.15.0",
        "pydantic>=1.8.2",
        "redis>=3.5.3",
        "psutil>=5.8.0",
    ],
    extras_require={
        "dev": [
            "pytest>=6.2.4",
            "pytest-cov>=2.12.1",
            "black>=21.7b0",
            "flake8>=3.9.2",
        ],
        "test": [
            "pytest>=6.2.4",
            "pytest-mock>=3.6.1",
            "httpx>=0.18.2",
        ]
    },
    entry_points={
        "console_scripts": [
            "model-service=src.app:main",
        ],
    },
    python_requires=">=3.7",
    author="Your Name",
    author_email="your.email@example.com",
    description="PyTorch model serving service",
    long_description=open("README.md").read(),
    long_description_content_type="text/markdown",
    url="https://github.com/your-username/pytorch-model-service",
    classifiers=[
        "Development Status :: 4 - Beta",
        "Intended Audience :: Developers",
        "License :: OSI Approved :: MIT License",
        "Programming Language :: Python :: 3",
        "Programming Language :: Python :: 3.7",
        "Programming Language :: Python :: 3.8",
        "Programming Language :: Python :: 3.9",
    ],
)
"""

# pyproject.toml for modern Python packaging
"""
[build-system]
requires = ["setuptools>=45", "wheel", "setuptools_scm[toml]>=6.2"]

[project]
name = "pytorch-model-service"
dynamic = ["version"]
description = "PyTorch model serving service"
readme = "README.md"
authors = [{name = "Your Name", email = "your.email@example.com"}]
license = {text = "MIT"}
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.7",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
]
requires-python = ">=3.7"
dependencies = [
    "torch>=1.9.0",
    "torchvision>=0.10.0",
    "fastapi>=0.68.0",
    "uvicorn>=0.15.0",
    "pydantic>=1.8.2",
    "redis>=3.5.3",
    "psutil>=5.8.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=6.2.4",
    "pytest-cov>=2.12.1",
    "black>=21.7b0",
    "flake8>=3.9.2",
]
test = [
    "pytest>=6.2.4",
    "pytest-mock>=3.6.1",
    "httpx>=0.18.2",
]

[project.scripts]
model-service = "src.app:main"

[tool.setuptools_scm]
write_to = "src/_version.py"

[tool.black]
line-length = 88
target-version = ['py37', 'py38', 'py39']
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | build
  | dist
)/
'''

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --cov=src --cov-report=term-missing"
"""

# pre-commit配置 (.pre-commit-config.yaml)
"""
repos:
  - repo: https://github.com/psf/black
    rev: 21.7b0
    hooks:
      - id: black
        language_version: python3

  - repo: https://github.com/pycqa/flake8
    rev: 3.9.2
    hooks:
      - id: flake8

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v0.910
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.0.1
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
"""

# 部署脚本 (deploy.sh)
"""
#!/bin/bash

set -e

# 配置变量
NAMESPACE="model-service"
DEPLOYMENT_NAME="pytorch-model-service"
IMAGE_NAME="your-registry/pytorch-model-service"
IMAGE_TAG=${1:-latest}

echo "开始部署 ${IMAGE_NAME}:${IMAGE_TAG}"

# 应用Kubernetes配置
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml

# 更新部署镜像
kubectl set image deployment/${DEPLOYMENT_NAME} model-service=${IMAGE_NAME}:${IMAGE_TAG} -n ${NAMESPACE}

# 等待部署完成
kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}

echo "部署完成"
"""

# 回滚脚本 (rollback.sh)
"""
#!/bin/bash

echo "开始回滚到上一个版本"

# 回滚到上一个版本
kubectl rollout undo deployment/pytorch-model-service -n model-service

# 等待回滚完成
kubectl rollout status deployment/pytorch-model-service -n model-service

echo "回滚完成"
"""

# 监控和告警配置 (monitoring/prometheus-rules.yaml)
"""
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: model-service-rules
  namespace: monitoring
spec:
  groups:
  - name: model-service.rules
    rules:
    - alert: HighLatency
      expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High latency detected"
        description: "95th percentile latency is above 1 second"

    - alert: HighErrorRate
      expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is above 5%"

    - alert: LowAvailability
      expr: up{job="model-service"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Service is down"
        description: "Model service is not responding"
"""
```

### 15. 生产环境最佳实践有哪些？
1. **模型版本控制**：使用Git LFS管理大模型文件，实施蓝绿部署
2. **配置管理**：使用环境变量和配置文件分离配置，支持动态配置更新
3. **监控告警**：实施全面的指标监控，设置合理的告警阈值
4. **日志管理**：结构化日志记录，集中日志收集和分析
5. **安全防护**：API密钥验证，速率限制，输入验证和清理
6. **性能优化**：模型量化，批处理优化，缓存策略
7. **容错机制**：断路器模式，重试机制，优雅降级
8. **自动扩缩容**：基于CPU/内存使用率的水平扩缩容
9. **健康检查**：实现就绪和存活探针
10. **备份恢复**：定期备份模型和配置，制定恢复计划