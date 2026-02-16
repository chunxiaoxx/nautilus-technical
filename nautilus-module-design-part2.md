# Nautilus 模块详细设计 - Part 2

**整理**: 明 (OpenClaw)
**日期**: 2026-02-16

---

## 7. 记忆系统设计

### 7.1 简单记忆存储

```python
# memory/simple_memory.py
import json
import os
from datetime import datetime
from typing import List, Optional, Dict
from pathlib import Path

class SimpleMemory:
    """简化版记忆系统 - 使用JSON文件存储"""
    
    def __init__(self, storage_dir: str = "./memory"):
        self.storage_dir = Path(storage_dir)
        self.storage_dir.mkdir(exist_ok=True)
        
        # 记忆文件
        self.episodic_file = self.storage_dir / "episodic.json"   # 事件记忆
        self.semantic_file = self.storage_dir / "semantic.json"   # 语义记忆
        self.preference_file = self.storage_dir / "preference.json" # 偏好记忆
        
        # 初始化文件
        for f in [self.episodic_file, self.semantic_file, self.preference_file]:
            if not f.exists():
                f.write_text("[]")
    
    # ========== 记忆存储 ==========
    
    def store_episodic(self, user_id: str, content: str, 
                       context: Optional[Dict] = None) -> str:
        """存储事件记忆"""
        memory = {
            'id': f"ep_{datetime.now().timestamp()}",
            'user_id': user_id,
            'content': content,
            'context': context or {},
            'timestamp': datetime.now().isoformat()
        }
        
        # 读取现有记忆
        memories = self._read_json(self.episodic_file)
        memories.append(memory)
        
        # 写回（限制1000条）
        self._write_json(self.episodic_file, memories[-1000:])
        
        return memory['id']
    
    def store_semantic(self, user_id: str, fact: str, 
                       category: str = "general") -> str:
        """存储语义记忆（事实）"""
        memory = {
            'id': f"sm_{datetime.now().timestamp()}",
            'user_id': user_id,
            'fact': fact,
            'category': category,
            'timestamp': datetime.now().isoformat()
        }
        
        memories = self._read_json(self.semantic_file)
        
        # 更新或添加
        updated = False
        for i, m in enumerate(memories):
            if m['user_id'] == user_id and m['category'] == category:
                memories[i] = memory
                updated = True
                break
        
        if not updated:
            memories.append(memory)
        
        self._write_json(self.semantic_file, memories)
        
        return memory['id']
    
    def store_preference(self, user_id: str, preference: Dict) -> str:
        """存储偏好记忆"""
        memory = {
            'id': f"pf_{datetime.now().timestamp()}",
            'user_id': user_id,
            'preferences': preference,
            'timestamp': datetime.now().isoformat()
        }
        
        memories = self._read_json(self.preference_file)
        
        # 更新用户偏好
        updated = False
        for i, m in enumerate(memories):
            if m['user_id'] == user_id:
                # 合并偏好
                merged = {**m.get('preferences', {}), **preference}
                memories[i]['preferences'] = merged
                updated = True
                break
        
        if not updated:
            memories.append(memory)
        
        self._write_json(self.preference_file, memories)
        
        return memory['id']
    
    # ========== 记忆检索 ==========
    
    def retrieve_context(self, user_id: str, 
                        query: str = "", limit: int = 5) -> Dict:
        """检索上下文记忆"""
        # 1. 获取最近的Episodic记忆
        episodic = self._read_json(self.episodic_file)
        user_episodic = [m for m in episodic if m['user_id'] == user_id]
        recent_episodic = user_episodic[-limit:]
        
        # 2. 获取Semantic记忆
        semantic = self._read_json(self.semantic_file)
        user_semantic = [m for m in semantic if m['user_id'] == user_id]
        
        # 3. 获取Preference记忆
        preference = self._read_json(self.preference_file)
        user_pref = next((m for m in preference if m['user_id'] == user_id), None)
        
        return {
            'recent_events': [m['content'] for m in recent_episodic],
            'facts': [m['fact'] for m in user_semantic],
            'preferences': user_pref.get('preferences', {}) if user_pref else {},
            'count': len(recent_episodic)
        }
    
    def search_episodic(self, user_id: str, keyword: str, 
                        limit: int = 10) -> List[Dict]:
        """搜索事件记忆"""
        episodic = self._read_json(self.episodic_file)
        results = [
            m for m in episodic 
            if m['user_id'] == user_id and keyword.lower() in m['content'].lower()
        ]
        return results[-limit:]
    
    # ========== 工具方法 ==========
    
    def _read_json(self, filepath: Path) -> List:
        """读取JSON文件"""
        try:
            return json.loads(filepath.read_text())
        except:
            return []
    
    def _write_json(self, filepath: Path, data: List):
        """写入JSON文件"""
        filepath.write_text(json.dumps(data, ensure_ascii=False, indent=2))
    
    def clear(self, user_id: str = None):
        """清理记忆"""
        if user_id:
            # 清理特定用户的记忆
            for f in [self.episodic_file, self.semantic_file, self.preference_file]:
                memories = [m for m in self._read_json(f) if m['user_id'] != user_id]
                self._write_json(f, memories)
        else:
            # 清理所有
            for f in [self.episodic_file, self.semantic_file, self.preference_file]:
                self._write_json(f, [])
```

### 7.2 记忆中间件

```python
# memory/middleware.py
from functools import wraps
from flask import request, g
from memory.simple_memory import SimpleMemory

memory = SimpleMemory()

def with_memory(f):
    """记忆中间件 - 自动注入记忆"""
    @wraps(f)
    def decorated(*args, **kwargs):
        # 获取user_id (从header或参数)
        user_id = request.headers.get('X-User-ID') or \
                  request.args.get('user_id') or \
                  'default'
        
        g.user_id = user_id
        g.memory = memory
        g.context = memory.retrieve_context(user_id)
        
        return f(*args, **kwargs)
    return decorated

# 使用示例
@app.route('/task/execute', methods=['POST'])
@with_memory
def execute_task():
    user_id = g.user_id
    context = g.context
    
    # 在任务执行时使用上下文
    task = request.json['description']
    
    # 注入上下文到任务
    if context['preferences']:
        # 根据偏好调整任务
        pass
    
    # 存储执行结果到记忆
    g.memory.store_episodic(
        user_id=user_id,
        content=f"Executed task: {task}",
        context={'task_id': task_id, 'result': result}
    )
```

---

## 8. 配置管理

### 8.1 配置类

```python
# config.py
import os
from dataclasses import dataclass, field
from typing import Optional
from pathlib import Path

@dataclass
class DatabaseConfig:
    """数据库配置"""
    path: str = "nautilus.db"
    
@dataclass
class AgentConfig:
    """Agent配置"""
    local_endpoint: str = "http://localhost:8080"
    cloud_endpoint: str = "http://cloud:8080"
    timeout: int = 30
    
@dataclass
class PoWConfig:
    """PoW配置"""
    difficulty: int = 1
    base_reward: float = 10.0
    workload_multiplier: float = 0.1
    
@dataclass
class BlockchainConfig:
    """区块链配置（后期使用）"""
    network: str = "polygon"  # polygon, arbitrum, etc.
    rpc_url: str = ""
    contract_address: str = ""
    use_mock: bool = True  # MVP阶段用Mock
    
@dataclass
class MemoryConfig:
    """记忆配置"""
    storage_dir: str = "./memory"
    max_episodic: int = 1000
    
@dataclass
class Config:
    """主配置"""
    debug: bool = True
    host: str = "0.0.0.0"
    port: int = 5000
    
    database: DatabaseConfig = field(default_factory=DatabaseConfig)
    agent: AgentConfig = field(default_factory=AgentConfig)
    pow: PoWConfig = field(default_factory=PoWConfig)
    blockchain: BlockchainConfig = field(default_factory=BlockchainConfig)
    memory: MemoryConfig = field(default_factory=MemoryConfig)
    
    @classmethod
    def from_env(cls) -> 'Config':
        """从环境变量加载配置"""
        return cls(
            debug=os.getenv('DEBUG', 'true').lower() == 'true',
            host=os.getenv('HOST', '0.0.0.0'),
            port=int(os.getenv('PORT', 5000)),
            database=DatabaseConfig(
                path=os.getenv('DB_PATH', 'nautilus.db')
            ),
            agent=AgentConfig(
                local_endpoint=os.getenv('LOCAL_AGENT', 'http://localhost:8080'),
                cloud_endpoint=os.getenv('CLOUD_AGENT', 'http://cloud:8080')
            ),
            blockchain=BlockchainConfig(
                network=os.getenv('BLOCKCHAIN_NETWORK', 'polygon'),
                rpc_url=os.getenv('RPC_URL', ''),
                use_mock=os.getenv('USE_MOCK', 'true').lower() == 'true'
            )
        )
    
    @classmethod
    def load(cls, path: str = "config.yaml") -> 'Config':
        """从YAML加载配置"""
        # 简化版：先用环境变量
        return cls.from_env()

# 全局配置
config = Config.load()
```

### 8.2 环境变量示例

```bash
# .env 文件
DEBUG=true
HOST=0.0.0.0
PORT=5000

# 数据库
DB_PATH=nautilus.db

# Agent
LOCAL_AGENT=http://localhost:8080
CLOUD_AGENT=http://cloud:8080

# 区块链
BLOCKCHAIN_NETWORK=polygon
RPC_URL=https://polygon-rpc.com
USE_MOCK=true

# 奖励
BASE_REWARD=10
WORKLOAD_MULTIPLIER=0.1
```

---

## 9. 测试用例

### 9.1 单元测试

```python
# tests/test_task.py
import pytest
import sys
sys.path.insert(0, '.')

from models.task import Task, TaskStatus, TaskPriority
from storage.task_storage import TaskStorage
import os
import tempfile

@pytest.fixture
def temp_db():
    """临时数据库"""
    fd, path = tempfile.mkstemp(suffix='.db')
    os.close(fd)
    yield path
    os.unlink(path)

@pytest.fixture
def storage(temp_db):
    return TaskStorage(temp_db)

def test_create_task(storage):
    """测试创建任务"""
    task = Task(description="测试任务")
    result = storage.create(task)
    
    assert result.id is not None
    assert result.description == "测试任务"
    assert result.status == TaskStatus.PENDING

def test_get_task(storage):
    """测试获取任务"""
    task = Task(description="测试任务")
    storage.create(task)
    
    retrieved = storage.get(task.id)
    assert retrieved is not None
    assert retrieved.id == task.id
    assert retrieved.description == "测试任务"

def test_update_task(storage):
    """测试更新任务"""
    task = Task(description="测试任务")
    storage.create(task)
    
    task.status = TaskStatus.COMPLETED
    task.result = "执行结果"
    storage.update(task)
    
    updated = storage.get(task.id)
    assert updated.status == TaskStatus.COMPLETED
    assert updated.result == "执行结果"

def test_list_by_status(storage):
    """测试按状态查询"""
    # 创建多个任务
    for i in range(5):
        task = Task(description=f"任务{i}")
        storage.create(task)
    
    pending = storage.list_by_status(TaskStatus.PENDING)
    assert len(pending) == 5
```

### 9.2 集成测试

```python
# tests/test_integration.py
import pytest
from models.task import Task, TaskStatus
from storage.task_storage import TaskStorage
from pow.generator import PoWGenerator
from routing.simple_router import SimpleRouter
from economy.simple_rewards import SimpleRewardSystem
import tempfile
import os

@pytest.fixture
def setup():
    """设置测试环境"""
    fd, db_path = tempfile.mkstemp(suffix='.db')
    os.close(fd)
    
    storage = TaskStorage(db_path)
    pow_gen = PoWGenerator()
    router = SimpleRouter()
    rewards = SimpleRewardSystem()
    
    yield {
        'storage': storage,
        'pow': pow_gen,
        'router': router,
        'rewards': rewards
    }
    
    os.unlink(db_path)

def test_full_task_flow(setup):
    """完整任务流程测试"""
    storage = setup['storage']
    pow_gen = setup['pow']
    router = setup['router']
    rewards = setup['rewards']
    
    # 1. 创建任务
    task = Task(description="简单任务测试")
    storage.create(task)
    assert task.status == TaskStatus.PENDING
    
    # 2. 路由决策
    decision = router.decide(task.description, {})
    assert decision.agent_type == 'local'
    
    # 3. 模拟执行
    result = "执行结果"
    task.result = result
    task.status = TaskStatus.COMPLETED
    
    # 4. 生成PoW
    pow = pow_gen.generate(task.id, 'local-001', result)
    assert pow.pow_hash is not None
    assert pow.workload > 0
    
    # 5. 发放奖励
    reward = rewards.award('local-001', pow.workload)
    assert reward > 0
    
    # 6. 验证余额
    balance = rewards.get_balance('local-001')
    assert balance.balance == reward
```

### 9.3 API测试

```python
# tests/test_api.py
import pytest
import json
import sys
sys.path.insert(0, '.')

from main import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_health(client):
    """健康检查"""
    response = client.get('/health')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['status'] == 'ok'

def test_create_task(client):
    """创建任务"""
    response = client.post('/api/tasks',
        data=json.dumps({'description': '测试任务'}),
        content_type='application/json'
    )
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['success'] is True
    assert 'task' in data

def test_get_task(client):
    """获取任务"""
    # 先创建
    create_resp = client.post('/api/tasks',
        data=json.dumps({'description': '测试'}),
        content_type='application/json'
    )
    task_id = json.loads(create_resp.data)['task']['id']
    
    # 再获取
    response = client.get(f'/api/tasks/{task_id}')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['task']['id'] == task_id

def test_execute_task(client):
    """执行任务"""
    # 创建任务
    create_resp = client.post('/api/tasks',
        data=json.dumps({'description': '执行这个任务'}),
        content_type='application/json'
    )
    task_id = json.loads(create_resp.data)['task']['id']
    
    # 执行
    response = client.post(f'/api/tasks/{task_id}/execute')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['success'] is True
    assert data['task']['status'] == 'completed'
    assert data['task']['pow_hash'] is not None
```

---

## 10. 部署脚本

### 10.1 本地运行

```bash
# run_local.sh
#!/bin/bash

# 设置环境变量
export DEBUG=true
export PORT=5000
export DB_PATH=nautilus.db
export USE_MOCK=true

# 安装依赖
pip install -r requirements.txt

# 运行
python main.py
```

### 10.2 Docker部署

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 创建数据目录
RUN mkdir -p /app/data /app/memory

# 暴露端口
EXPOSE 5000

# 运行
CMD ["python", "main.py"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  nautilus:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - ./data:/app/data
      - ./memory:/app/memory
    environment:
      - DEBUG=true
      - PORT=5000
      - DB_PATH=/app/data/nautilus.db
      - USE_MOCK=true
    restart: unless-stopped
```

```bash
# run_docker.sh
#!/bin/bash

# 构建镜像
docker build -t nautilus:mvp .

# 运行
docker run -d \
  --name nautilus \
  -p 5000:5000 \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/memory:/app/memory \
  -e DEBUG=true \
  nautilus:mvp
```

### 10.3 systemd服务（Linux服务器）

```ini
# /etc/systemd/system/nautilus.service
[Unit]
Description=Nautilus MVP Service
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/nautilus
Environment="DEBUG=false"
Environment="PORT=5000"
Environment="DB_PATH=/home/ubuntu/nautilus/data/nautilus.db"
ExecStart=/usr/bin/python3 /home/ubuntu/nautilus/main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# 部署命令
sudo cp nautilus.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable nautilus
sudo systemctl start nautilus
sudo systemctl status nautilus
```

---

## 11. 快速启动指南

### 11.1 一键启动

```bash
# setup.sh
#!/bin/bash

echo "🚀 安装 Nautilus MVP..."

# 1. 创建目录
mkdir -p nautilus-mvp
cd nautilus-mvp

# 2. 创建虚拟环境
python3 -m venv venv
source venv/bin/activate

# 3. 安装依赖
cat > requirements.txt << 'EOF'
flask==3.0.0
requests==2.31.0
python-dotenv==1.0.0
pytest==7.4.0
EOF
pip install -r requirements.txt

# 4. 下载代码
# (这里应该是从Git clone，这里简化)

# 5. 启动
echo "✅ 安装完成！启动服务..."
export DEBUG=true
python main.py &
echo "🚀 服务已启动: http://localhost:5000"
echo "📚 API文档: http://localhost:5000/health"
```

### 11.2 验证命令

```bash
# 1. 健康检查
curl http://localhost:5000/health

# 2. 创建任务
curl -X POST http://localhost:5000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"description":"Hello World"}'

# 3. 执行任务
curl -X POST http://localhost:5000/api/tasks/<TASK_ID>/execute

# 4. 查看结果
curl http://localhost:5000/api/tasks/<TASK_ID>

# 5. 排行榜
curl http://localhost:5000/api/leaderboard
```

---

## 12. 下一步

### Part 3 预告
- 完整错误处理
- 日志系统
- 监控指标
- CI/CD流水线

---

**代码总行数统计**:
- Part 1: ~400行
- Part 2: ~400行
- **总计MVP**: ~800行
