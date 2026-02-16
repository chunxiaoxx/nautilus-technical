# Nautilus 模块详细设计 - Part 3

**整理**: 明 (OpenClaw)
**日期**: 2026-02-16

---

## 13. 错误处理系统

### 13.1 异常类定义

```python
# exceptions.py
from typing import Optional, Any

class NautilusException(Exception):
    """基础异常类"""
    def __init__(self, message: str, code: str = "UNKNOWN_ERROR"):
        self.message = message
        self.code = code
        super().__init__(self.message)

class TaskException(NautilusException):
    """任务相关异常"""
    pass

class TaskNotFoundError(TaskException):
    def __init__(self, task_id: str):
        super().__init__(f"Task not found: {task_id}", "TASK_NOT_FOUND")
        self.task_id = task_id

class TaskAlreadyExistsError(TaskException):
    def __init__(self, task_id: str):
        super().__init__(f"Task already exists: {task_id}", "TASK_EXISTS")
        self.task_id = task_id

class TaskExecutionError(TaskException):
    def __init__(self, task_id: str, reason: str):
        super().__init__(f"Task execution failed: {reason}", "TASK_EXECUTION_ERROR")
        self.task_id = task_id

class AgentException(NautilusException):
    """Agent相关异常"""
    pass

class AgentNotAvailableError(AgentException):
    def __init__(self, agent_id: str):
        super().__init__(f"Agent not available: {agent_id}", "AGENT_NOT_AVAILABLE")
        self.agent_id = agent_id

class AgentTimeoutError(AgentException):
    def __init__(self, agent_id: str, timeout: int):
        super().__init__(f"Agent timeout: {agent_id}", "AGENT_TIMEOUT")
        self.agent_id = agent_id
        self.timeout = timeout

class PoWException(NautilusException):
    """PoW相关异常"""
    pass

class PoWVerificationError(PoWException):
    def __init__(self, reason: str):
        super().__init__(f"PoW verification failed: {reason}", "POW_VERIFICATION_ERROR")

class BlockchainException(NautilusException):
    """区块链相关异常"""
    pass

class BlockchainConnectionError(BlockchainException):
    def __init__(self, network: str):
        super().__init__(f"Cannot connect to blockchain: {network}", "BLOCKCHAIN_CONNECTION_ERROR")
        self.network = network
```

### 13.2 全局错误处理

```python
# error_handlers.py
from flask import Flask, jsonify
from exceptions import NautilusException

def register_error_handlers(app: Flask):
    """注册全局错误处理器"""
    
    @app.errorhandler(NautilusException)
    def handle_nautilus_exception(e: NautilusException):
        return jsonify({
            'success': False,
            'error': {
                'code': e.code,
                'message': e.message
            }
        }), 400
    
    @app.errorhandler(404)
    def handle_404(e):
        return jsonify({
            'success': False,
            'error': {'code': 'NOT_FOUND', 'message': str(e)}
        }), 404
    
    @app.errorhandler(500)
    def handle_500(e):
        return jsonify({
            'success': False,
            'error': {'code': 'INTERNAL_ERROR', 'message': 'Internal server error'}
        }), 500
    
    @app.errorhandler(Exception)
    def handle_generic_exception(e: Exception):
        # 生产环境隐藏详细信息
        import current_app
        if current_app.config.get('DEBUG'):
            return jsonify({
                'success': False,
                'error': {
                    'code': 'UNHANDLED_ERROR',
                    'message': str(e)
                }
            }), 500
        else:
            return jsonify({
                'success': False,
                'error': {
                    'code': 'UNHANDLED_ERROR',
                    'message': 'An unexpected error occurred'
                }
            }), 500
```

### 13.3 重试机制

```python
# retry.py
import time
import functools
from typing import Callable, Type, Tuple
from exceptions import AgentTimeoutError, BlockchainConnectionError

def retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: Tuple[Type[Exception], ...] = (Exception,)
):
    """重试装饰器"""
    def decorator(func: Callable):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            attempt = 1
            current_delay = delay
            
            while attempt <= max_attempts:
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        raise
                    
                    print(f"Attempt {attempt}/{max_attempts} failed: {e}. Retrying in {current_delay}s...")
                    time.sleep(current_delay)
                    current_delay *= backoff
                    attempt += 1
            
            return func(*args, **kwargs)
        return wrapper
    return decorator

# 使用示例
class AgentClient:
    @retry(max_attempts=3, delay=1.0, exceptions=(AgentTimeoutError,))
    def execute_task(self, task: str) -> dict:
        # 可能超时的操作
        pass
    
    @retry(max_attempts=5, delay=2.0, backoff=1.5, 
           exceptions=(BlockchainConnectionError,))
    def submit_transaction(self, data: dict) -> str:
        # 可能失败的操作
        pass
```

---

## 14. 日志系统

### 14.1 日志配置

```python
# logging_config.py
import logging
import sys
from logging.handlers import RotatingFileHandler
from pathlib import Path

def setup_logging(
    log_level: str = "INFO",
    log_file: str = "logs/nautilus.log",
    max_bytes: int = 10 * 1024 * 1024,  # 10MB
    backup_count: int = 5
):
    """配置日志系统"""
    
    # 创建logs目录
    Path(log_file).parent.mkdir(parents=True, exist_ok=True)
    
    # 根日志配置
    logging.basicConfig(
        level=getattr(logging, log_level.upper()),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            # 控制台输出
            logging.StreamHandler(sys.stdout),
            # 文件输出（轮转）
            RotatingFileHandler(
                log_file,
                maxBytes=max_bytes,
                backupCount=backup_count,
                encoding='utf-8'
            )
        ]
    )
    
    # 设置第三方库日志级别
    logging.getLogger('werkzeug').setLevel(logging.WARNING)
    logging.getLogger('urllib3').setLevel(logging.WARNING)
    
    return logging.getLogger('nautilus')

# 模块级日志
logger = logging.getLogger('nautilus')
```

### 14.2 业务日志

```python
# business_logger.py
import logging
from datetime import datetime
from typing import Optional, Dict, Any

class BusinessLogger:
    """业务日志记录器"""
    
    def __init__(self):
        self.logger = logging.getLogger('nautilus.business')
    
    def log_task_created(self, task_id: str, description: str, user_id: str):
        self.logger.info(
            f"Task created | task_id={task_id} | user_id={user_id} | "
            f"description={description[:50]}"
        )
    
    def log_task_executed(self, task_id: str, agent_id: str, 
                         duration: float, success: bool):
        self.logger.info(
            f"Task executed | task_id={task_id} | agent_id={agent_id} | "
            f"duration={duration:.2f}s | success={success}"
        )
    
    def log_task_failed(self, task_id: str, agent_id: str, error: str):
        self.logger.error(
            f"Task failed | task_id={task_id} | agent_id={agent_id} | "
            f"error={error}"
        )
    
    def log_pow_generated(self, task_id: str, pow_hash: str, workload: float):
        self.logger.info(
            f"PoW generated | task_id={task_id} | pow_hash={pow_hash[:16]}... | "
            f"workload={workload:.2f}"
        )
    
    def log_reward_awarded(self, agent_id: str, amount: float, balance: float):
        self.logger.info(
            f"Reward awarded | agent_id={agent_id} | amount={amount} | "
            f"new_balance={balance}"
        )
    
    def log_blockchain_submitted(self, task_id: str, tx_hash: str, 
                                  network: str, gas_fee: float):
        self.logger.info(
            f"Blockchain submitted | task_id={task_id} | tx_hash={tx_hash[:16]}... | "
            f"network={network} | gas_fee={gas_fee}"
        )

# 全局实例
business_logger = BusinessLogger()
```

---

## 15. 监控指标

### 15.1 指标收集

```python
# metrics.py
import time
from functools import wraps
from typing import Callable, Dict, Any
import threading

class MetricsCollector:
    """指标收集器"""
    
    def __init__(self):
        self._lock = threading.Lock()
        self._counters: Dict[str, int] = {}
        self._gauges: Dict[str, float] = {}
        self._histograms: Dict[str, list] = {}
    
    def increment(self, name: str, value: int = 1):
        """计数器+1"""
        with self._lock:
            self._counters[name] = self._counters.get(name, 0) + value
    
    def gauge(self, name: str, value: float):
        """设置仪表值"""
        with self._lock:
            self._gauges[name] = value
    
    def histogram(self, name: str, value: float):
        """直方图记录"""
        with self._lock:
            if name not in self._histograms:
                self._histograms[name] = []
            self._histograms[name].append(value)
            # 保留最近1000条
            self._histograms[name] = self._histograms[name][-1000:]
    
    def get_all(self) -> dict:
        """获取所有指标"""
        with self._lock:
            # 计算直方图统计
            histogram_stats = {}
            for name, values in self._histograms.items():
                if values:
                    histogram_stats[name] = {
                        'count': len(values),
                        'sum': sum(values),
                        'avg': sum(values) / len(values),
                        'min': min(values),
                        'max': max(values),
                        'p50': sorted(values)[len(values) // 2],
                        'p95': sorted(values)[int(len(values) * 0.95)],
                        'p99': sorted(values)[int(len(values) * 0.99)]
                    }
            
            return {
                'counters': self._counters.copy(),
                'gauges': self._gauges.copy(),
                'histograms': histogram_stats
            }

# 全局实例
metrics = MetricsCollector()

# 指标装饰器
def track_time(name: str):
    """追踪执行时间"""
    def decorator(func: Callable):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            try:
                result = func(*args, **kwargs)
                return result
            finally:
                duration = time.time() - start
                metrics.histogram(f"{name}_duration", duration)
                metrics.increment(f"{name}_total")
        return wrapper
    return decorator

def track_counter(name: str):
    """追踪计数器"""
    def decorator(func: Callable):
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                result = func(*args, **kwargs)
                metrics.increment(f"{name}_success")
                return result
            except Exception:
                metrics.increment(f"{name}_error")
                raise
        return wrapper
    return decorator
```

### 15.2 指标API

```python
# metrics_api.py
from flask import jsonify
from metrics import metrics

@app.route('/metrics', methods=['GET'])
def get_metrics():
    """获取监控指标"""
    return jsonify(metrics.get_all())

@app.route('/metrics/counters', methods=['GET'])
def get_counters():
    """获取计数器"""
    return jsonify(metrics.get_all()['counters'])

@app.route('/metrics/gauges', methods=['GET'])
def get_gauges():
    """获取仪表值"""
    return jsonify(metrics.get_all()['gauges'])

# 使用示例
@track_time('task_execute')
@track_counter('task_execute')
def execute_task(task_id: str):
    # 任务执行逻辑
    metrics.increment('tasks_created')
    metrics.gauge('active_tasks', get_active_count())
    pass
```

---

## 16. CI/CD 流水线

### 16.1 GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
    
    - name: Cache dependencies
      uses: actions/cache@v4
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov
    
    - name: Run linter
      run: |
        pip install flake8
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
    
    - name: Run tests
      run: |
        pytest tests/ --cov=. --cov-report=xml --cov-report=term
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml

  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Build Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: false
        tags: nautilus:${{ github.sha }}
        cache-from: type=registry,ref=nautilus:latest
        cache-to: type=inline
```

### 16.2 部署流水线

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to server
      uses: appleboy/ssh-action@v1
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /home/ubuntu/nautilus
          git pull
          docker-compose down
          docker-compose build
          docker-compose up -d
          docker system prune -f
```

---

## 17. OpenClaw/OpenManus 集成

### 17.1 OpenClaw 客户端

```python
# agents/openclaw_client.py
import requests
from typing import Optional, Dict, Any
from dataclasses import dataclass
from exceptions import AgentNotAvailableError, AgentTimeoutError

@dataclass
class OpenClawConfig:
    endpoint: str = "http://localhost:8080"
    timeout: int = 30
    api_key: Optional[str] = None

class OpenClawClient:
    """OpenClaw 客户端"""
    
    def __init__(self, config: OpenClawConfig = None):
        self.config = config or OpenClawConfig()
        self.session = requests.Session()
        
        if self.config.api_key:
            self.session.headers['Authorization'] = f"Bearer {self.config.api_key}"
    
    def health_check(self) -> bool:
        """检查服务健康状态"""
        try:
            resp = self.session.get(
                f"{self.config.endpoint}/health",
                timeout=5
            )
            return resp.status_code == 200
        except:
            return False
    
    def execute_task(self, task: str, context: Optional[Dict] = None) -> Dict[str, Any]:
        """执行任务"""
        payload = {
            'task': task,
            'context': context or {}
        }
        
        try:
            resp = self.session.post(
                f"{self.config.endpoint}/execute",
                json=payload,
                timeout=self.config.timeout
            )
            resp.raise_for_status()
            return resp.json()
        except requests.exceptions.Timeout as e:
            raise AgentTimeoutError('openclaw', self.config.timeout) from e
        except requests.exceptions.RequestException as e:
            raise AgentNotAvailableError('openclaw') from e
    
    def get_capabilities(self) -> Dict[str, Any]:
        """获取Agent能力"""
        resp = self.session.get(
            f"{self.config.endpoint}/capabilities",
            timeout=10
        )
        resp.raise_for_status()
        return resp.json()
    
    def list_tools(self) -> list:
        """列出可用工具"""
        resp = self.session.get(
            f"{self.config.endpoint}/tools",
            timeout=10
        )
        resp.raise_for_status()
        return resp.json()
```

### 17.2 OpenManus 客户端

```python
# agents/openmanus_client.py
import requests
from typing import Optional, Dict, Any, List
from dataclasses import dataclass
from exceptions import AgentNotAvailableError, AgentTimeoutError

@dataclass
class OpenManusConfig:
    endpoint: str = "http://cloud:8080"
    timeout: int = 120  # 复杂任务需要更长时间
    api_key: Optional[str] = None

class OpenManusClient:
    """OpenManus 客户端"""
    
    def __init__(self, config: OpenManusConfig = None):
        self.config = config or OpenManusConfig()
        self.session = requests.Session()
        
        if self.config.api_key:
            self.session.headers['Authorization'] = f"Bearer {self.config.api_key}"
    
    def health_check(self) -> bool:
        """检查服务健康状态"""
        try:
            resp = self.session.get(
                f"{self.config.endpoint}/health",
                timeout=5
            )
            return resp.status_code == 200
        except:
            return False
    
    def execute_task(self, task: str, 
                    context: Optional[Dict] = None,
                    parameters: Optional[Dict] = None) -> Dict[str, Any]:
        """执行复杂任务"""
        payload = {
            'task': task,
            'context': context or {},
            'parameters': parameters or {}
        }
        
        try:
            resp = self.session.post(
                f"{self.config.endpoint}/execute",
                json=payload,
                timeout=self.config.timeout
            )
            resp.raise_for_status()
            return resp.json()
        except requests.exceptions.Timeout as e:
            raise AgentTimeoutError('openmanus', self.config.timeout) from e
        except requests.exceptions.RequestException as e:
            raise AgentNotAvailableError('openmanus') from e
    
    def execute_batch(self, tasks: List[str]) -> List[Dict[str, Any]]:
        """批量执行任务"""
        payload = {'tasks': tasks}
        
        resp = self.session.post(
            f"{self.config.endpoint}/batch",
            json=payload,
            timeout=self.config.timeout * len(tasks)
        )
        resp.raise_for_status()
        return resp.json()['results']
    
    def get_status(self, job_id: str) -> Dict[str, Any]:
        """获取任务状态"""
        resp = self.session.get(
            f"{self.config.endpoint}/status/{job_id}",
            timeout=10
        )
        resp.raise_for_status()
        return resp.json()
```

### 17.3 Agent工厂

```python
# agents/factory.py
from typing import Dict
from agents.base import BaseAgent, LocalAgent, CloudAgent
from agents.openclaw_client import OpenClawClient, OpenClawConfig
from agents.openmanus_client import OpenManusClient, OpenManusConfig
from config import config

class AgentFactory:
    """Agent工厂"""
    
    _instances: Dict[str, BaseAgent] = {}
    
    @classmethod
    def get_local_agent(cls) -> LocalAgent:
        """获取本地Agent"""
        if 'local' not in cls._instances:
            # 尝试使用真实OpenClaw客户端
            try:
                client = OpenClawClient(OpenClawConfig(
                    endpoint=config.agent.local_endpoint
                ))
                if client.health_check():
                    cls._instances['local'] = OpenClawAgent(client)
                    return cls._instances['local']
            except:
                pass
            
            # 回退到模拟Agent
            cls._instances['local'] = LocalAgent()
        
        return cls._instances['local']
    
    @classmethod
    def get_cloud_agent(cls) -> CloudAgent:
        """获取云端Agent"""
        if 'cloud' not in cls._instances:
            # 尝试使用真实OpenManus客户端
            try:
                client = OpenManusClient(OpenManusConfig(
                    endpoint=config.agent.cloud_endpoint
                ))
                if client.health_check():
                    cls._instances['cloud'] = OpenManusAgent(client)
                    return cls._instances['cloud']
            except:
                pass
            
            # 回退到模拟Agent
            cls._instances['cloud'] = CloudAgent()
        
        return cls._instances['cloud']
    
    @classmethod
    def get_all_agents(cls) -> Dict[str, BaseAgent]:
        """获取所有Agent"""
        return {
            'local': cls.get_local_agent(),
            'cloud': cls.get_cloud_agent()
        }


class OpenClawAgent(BaseAgent):
    """OpenClaw包装Agent"""
    
    def __init__(self, client: OpenClawClient):
        super().__init__(
            agent_id='openclaw-001',
            name='OpenClaw',
            endpoint=client.config.endpoint
        )
        self.client = client
    
    def execute(self, task: str):
        result = self.client.execute_task(task)
        return ExecutionResult(
            success=result.get('success', True),
            result=result.get('result', ''),
            duration=result.get('duration', 0)
        )
    
    def health_check(self) -> bool:
        return self.client.health_check()


class OpenManusAgent(BaseAgent):
    """OpenManus包装Agent"""
    
    def __init__(self, client: OpenManusClient):
        super().__init__(
            agent_id='openmanus-001',
            name='OpenManus',
            endpoint=client.config.endpoint
        )
        self.client = client
    
    def execute(self, task: str):
        result = self.client.execute_task(task)
        return ExecutionResult(
            success=result.get('success', True),
            result=result.get('result', ''),
            duration=result.get('duration', 0)
        )
    
    def health_check(self) -> bool:
        return self.client.health_check()
```

---

## 18. Telegram Bot 集成

### 18.1 Bot基础

```python
# telegram_bot.py
from flask import Flask, request
import requests
from typing import Optional
import json

class TelegramBot:
    """Telegram Bot"""
    
    def __init__(self, token: str):
        self.token = token
        self.api_url = f"https://api.telegram.org/bot{token}"
        self.flask_app = Flask(__name__)
        self._register_handlers()
    
    def _register_handlers(self):
        """注册消息处理"""
        
        @self.flask_app.route(f"/webhook/{self.token}", methods=['POST'])
        def webhook():
            data = request.get_json()
            self._handle_update(data)
            return 'OK'
    
    def _handle_update(self, update: dict):
        """处理更新"""
        if 'message' in update:
            message = update['message']
            chat_id = message['chat']['id']
            text = message.get('text', '')
            
            # 处理命令
            if text.startswith('/start'):
                self.send_message(chat_id, "欢迎使用 Nautilus！")
            elif text.startswith('/help'):
                self.send_message(chat_id, self._get_help())
            elif text.startswith('/status'):
                self.send_message(chat_id, self._get_status())
            else:
                # 转发给主服务处理
                self._process_user_message(chat_id, text)
    
    def send_message(self, chat_id: int, text: str, 
                    parse_mode: str = 'Markdown'):
        """发送消息"""
        url = f"{self.api_url}/sendMessage"
        data = {
            'chat_id': chat_id,
            'text': text,
            'parse_mode': parse_mode
        }
        requests.post(url, json=data)
    
    def send_inline_keyboard(self, chat_id: int, text: str, 
                           buttons: list):
        """发送带按钮的消息"""
        url = f"{self.api_url}/sendMessage"
        keyboard = [[{'text': btn['text'], 'callback_data': btn['data']} 
                   for btn in row] for row in buttons]
        data = {
            'chat_id': chat_id,
            'text': text,
            'reply_markup': {'inline_keyboard': keyboard}
        }
        requests.post(url, json=data)
    
    def _get_help(self) -> str:
        return """
*可用命令*:
/start - 开始
/help - 帮助
/status - 状态
/task - 创建任务
/list - 任务列表
        """
    
    def _get_status(self) -> str:
        return "*系统状态*: 运行中 ✅"
    
    def _process_user_message(self, chat_id: int, text: str):
        """处理用户消息"""
        # TODO: 调用主服务API
        self.send_message(chat_id, f"收到: {text}")
    
    def run(self, host: str = '0.0.0.0', port: int = 5001):
        """运行Bot"""
        self.flask_app.run(host=host, port=port)
    
    def set_webhook(self, url: str):
        """设置Webhook"""
        url = f"{self.api_url}/setWebhook"
        data = {'url': url}
        requests.post(url, json=data)
```

### 18.2 Flask集成

```python
# telegram_flask.py
from flask import Flask, request, jsonify
import os
from telegram_bot import TelegramBot

app = Flask(__name__)

# 初始化Bot
TELEGRAM_TOKEN = os.getenv('TELEGRAM_TOKEN')
if TELEGRAM_TOKEN:
    bot = TelegramBot(TELEGRAM_TOKEN)
else:
    bot = None

@app.route('/telegram/webhook', methods=['POST'])
def telegram_webhook():
    """Telegram Webhook"""
    if not bot:
        return jsonify({'error': 'Bot not configured'}), 500
    
    data = request.get_json()
    bot._handle_update(data)
    return 'OK'

@app.route('/telegram/send', methods=['POST'])
def send_to_telegram():
    """主动发送消息到Telegram"""
    data = request.json
    chat_id = data.get('chat_id')
    message = data.get('message')
    
    if not bot:
        return jsonify({'error': 'Bot not configured'}), 500
    
    bot.send_message(chat_id, message)
    return jsonify({'success': True})
```

---

## 19. 安全加固

### 19.1 API认证

```python
# auth.py
import hashlib
import hmac
import time
from functools import wraps
from flask import request, jsonify, g
import os

def generate_api_key(user_id: str, secret: str) -> str:
    """生成API Key"""
    timestamp = str(int(time.time()))
    message = f"{user_id}:{timestamp}"
    signature = hmac.new(
        secret.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()
    return f"{user_id}:{timestamp}:{signature}"

def verify_api_key(api_key: str, secret: str) -> bool:
    """验证API Key"""
    try:
        user_id, timestamp, signature = api_key.split(':')
        
        # 检查时间戳（24小时内有效）
        if abs(int(time.time()) - int(timestamp)) > 86400:
            return False
        
        # 验证签名
        message = f"{user_id}:{timestamp}"
        expected = hmac.new(
            secret.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return hmac.compare_digest(signature, expected)
    except:
        return False

def require_auth(f):
    """认证装饰器"""
    @wraps(f)
    def decorated(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')
        if not api_key:
            return jsonify({'error': 'Missing API key'}), 401
        
        secret = os.getenv('API_SECRET', 'default_secret')
        if not verify_api_key(api_key, secret):
            return jsonify({'error': 'Invalid API key'}), 401
        
        # 提取user_id
        g.user_id = api_key.split(':')[0]
        
        return f(*args, **kwargs)
    return decorated

# 使用
@app.route('/api/tasks', methods=['POST'])
@require_auth
def create_task():
    # g.user_id 包含认证的用户ID
    pass
```

### 19.2 输入验证

```python
# validation.py
from typing import Optional
from dataclasses import dataclass
import re

@dataclass
class ValidationResult:
    valid: bool
    error: Optional[str] = None

class InputValidator:
    """输入验证器"""
    
    @staticmethod
    def validate_task_description(description: str) -> ValidationResult:
        """验证任务描述"""
        if not description:
            return ValidationResult(False, "Description cannot be empty")
        
        if len(description) > 10000:
            return ValidationResult(False, "Description too long (max 10000 chars)")
        
        return ValidationResult(True)
    
    @staticmethod
    def validate_user_id(user_id: str) -> ValidationResult:
        """验证用户ID"""
        if not user_id:
            return ValidationResult(False, "User ID cannot be empty")
        
        if not re.match(r'^[a-zA-Z0-9_-]{1,64}$', user_id):
            return ValidationResult(False, "Invalid user ID format")
        
        return ValidationResult(True)
    
    @staticmethod
    def validate_agent_id(agent_id: str) -> ValidationResult:
        """验证Agent ID"""
        if not re.match(r'^[a-zA-Z0-9_-]{1,64}$', agent_id):
            return ValidationResult(False, "Invalid agent ID format")
        
        return ValidationResult(True)
```

---

## 20. 性能优化

### 20.1 连接池

```python
# connection_pool.py
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
from typing import Optional

def create_session(
    max_retries: int = 3,
    backoff_factor: float = 0.5,
    pool_connections: int = 10,
    pool_maxsize: int = 20
) -> requests.Session:
    """创建带连接池的Session"""
    
    session = requests.Session()
    
    # 配置重试
    retry = Retry(
        total=max_retries,
        read=max_retries,
        connect=max_retries,
        backoff_factor=backoff_factor,
        status_forcelist=[500, 502, 503, 504]
    )
    
    # 配置适配器
    adapter = HTTPAdapter(
        max_retries=retry,
        pool_connections=pool_connections,
        pool_maxsize=pool_maxsize
    )
    
    # 挂载适配器
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    
    return session

# 全局Session
http_session = create_session()
```

### 20.2 缓存

```python
# cache.py
import time
from functools import wraps
from typing import Callable, Any, Optional

class SimpleCache:
    """简单内存缓存"""
    
    def __init__(self, default_ttl: int = 300):
        self._cache = {}
        self._ttl = default_ttl
    
    def get(self, key: str) -> Optional[Any]:
        """获取缓存"""
        if key in self._cache:
            value, expire_at = self._cache[key]
            if time.time() < expire_at:
                return value
            else:
                del self._cache[key]
        return None
    
    def set(self, key: str, value: Any, ttl: Optional[int] = None):
        """设置缓存"""
        expire_at = time.time() + (ttl or self._ttl)
        self._cache[key] = (value, expire_at)
    
    def delete(self, key: str):
        """删除缓存"""
        if key in self._cache:
            del self._cache[key]
    
    def clear(self):
        """清空缓存"""
        self._cache.clear()
    
    def cached(self, ttl: Optional[int] = None):
        """缓存装饰器"""
        def decorator(func: Callable):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # 生成缓存key
                key = f"{func.__name__}:{str(args)}:{str(kwargs)}"
                
                # 尝试获取缓存
                result = self.get(key)
                if result is not None:
                    return result
                
                # 执行函数
                result = func(*args, **kwargs)
                
                # 设置缓存
                self.set(key, result, ttl)
                
                return result
            return wrapper
        return decorator

# 全局缓存
cache = SimpleCache(default_ttl=300)

# 使用示例
@cache.cached(ttl=60)
def get_agent_capabilities(agent_id: str):
    """获取Agent能力（缓存60秒）"""
    pass
```

---

## 21. 完整项目结构

```
nautilus/
├── main.py                    # 入口
├── config.py                  # 配置
├── exceptions.py              # 异常定义
├── logging_config.py          # 日志配置
├── requirements.txt           # 依赖
├── models/
│   ├── __init__.py
│   ├── task.py               # 任务模型
│   └── pow.py                # PoW模型
├── storage/
│   ├── __init__.py
│   └── task_storage.py       # SQLite存储
├── agents/
│   ├── __init__.py
│   ├── base.py               # Agent基类
│   ├── factory.py            # Agent工厂
│   ├── openclaw_client.py    # OpenClaw客户端
│   └── openmanus_client.py   # OpenManus客户端
├── routing/
│   ├── __init__.py
│   └── simple_router.py      # 路由
├── pow/
│   ├── __init__.py
│   └── generator.py          # PoW生成
├── economy/
│   ├── __init__.py
│   └── simple_rewards.py     # 奖励
├── memory/
│   ├── __init__.py
│   └── simple_memory.py      # 记忆
├── telegram/
│   ├── __init__.py
│   └── bot.py                # Telegram Bot
├── api/
│   ├── __init__.py
│   ├── routes.py             # API路由
│   ├── error_handlers.py    # 错误处理
│   └── auth.py               # 认证
├── monitoring/
│   ├── __init__.py
│   ├── metrics.py            # 指标收集
│   └── business_logger.py    # 业务日志
├── tests/
│   ├── __init__.py
│   ├── test_task.py
│   ├── test_pow.py
│   ├── test_integration.py
│   └── test_api.py
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
└── docs/
    └── README.md

# 总代码量: ~2000行
```

---

**Part 3 完成！**

**累计代码量**:
- Part 1: ~400行
- Part 2: ~400行
- Part 3: ~600行
- **总计**: ~1400行
