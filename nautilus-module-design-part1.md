# Nautilus 模块详细设计 - Part 1

**整理**: 明 (OpenClaw)
**日期**: 2026-02-16

---

## 1. 任务系统设计

### 1.1 任务数据模型

```python
# models/task.py
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional, List
import uuid

class TaskStatus(Enum):
    PENDING = "pending"           # 等待执行
    ROUTING = "routing"           # 路由决策中
    EXECUTING = "executing"      # 执行中
    COMPLETED = "completed"       # 已完成
    FAILED = "failed"             # 执行失败
    CANCELLED = "cancelled"       # 已取消

class TaskPriority(Enum):
    LOW = 1
    NORMAL = 2
    HIGH = 3
    URGENT = 4

@dataclass
class Task:
    """任务模型"""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    description: str = ""
    status: TaskStatus = TaskStatus.PENDING
    priority: TaskPriority = TaskPriority.NORMAL
    
    # 执行信息
    assigned_agent: Optional[str] = None      # 分配给哪个Agent
    result: Optional[str] = None              # 执行结果
    error: Optional[str] = None               # 错误信息
    
    # PoW信息
    pow_hash: Optional[str] = None
    workload: float = 0.0
    
    # 时间戳
    created_at: datetime = field(default_factory=datetime.now)
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    
    # 依赖
    depends_on: List[str] = field(default_factory=list)  # 依赖的任务ID
    
    def duration(self) -> float:
        """执行耗时（秒）"""
        if self.started_at and self.completed_at:
            return (self.completed_at - self.started_at).total_seconds()
        return 0.0
    
    def to_dict(self) -> dict:
        return {
            'id': self.id,
            'description': self.description,
            'status': self.status.value,
            'priority': self.priority.value,
            'assigned_agent': self.assigned_agent,
            'result': self.result,
            'error': self.error,
            'pow_hash': self.pow_hash,
            'workload': self.workload,
            'created_at': self.created_at.isoformat(),
            'started_at': self.started_at.isoformat() if self.started_at else None,
            'completed_at': self.completed_at.isoformat() if self.completed_at else None,
            'depends_on': self.depends_on,
            'duration': self.duration()
        }
```

### 1.2 任务存储（SQLite）

```python
# storage/task_storage.py
import sqlite3
from typing import List, Optional
from models.task import Task, TaskStatus, TaskPriority

class TaskStorage:
    """任务存储层"""
    
    def __init__(self, db_path: str = "nautilus.db"):
        self.db_path = db_path
        self._init_db()
    
    def _init_db(self):
        """初始化数据库表"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''
                CREATE TABLE IF NOT EXISTS tasks (
                    id TEXT PRIMARY KEY,
                    description TEXT NOT NULL,
                    status TEXT NOT NULL,
                    priority INTEGER DEFAULT 2,
                    assigned_agent TEXT,
                    result TEXT,
                    error TEXT,
                    pow_hash TEXT,
                    workload REAL DEFAULT 0,
                    created_at TEXT NOT NULL,
                    started_at TEXT,
                    completed_at TEXT,
                    depends_on TEXT
                )
            ''')
            conn.execute('''
                CREATE INDEX IF NOT EXISTS idx_status ON tasks(status)
            ''')
            conn.execute('''
                CREATE INDEX IF NOT EXISTS idx_created ON tasks(created_at)
            ''')
    
    def create(self, task: Task) -> Task:
        """创建任务"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''
                INSERT INTO tasks (id, description, status, priority, 
                                  assigned_agent, result, error,
                                  pow_hash, workload, created_at, 
                                  started_at, completed_at, depends_on)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                task.id, task.description, task.status.value,
                task.priority.value, task.assigned_agent, task.result,
                task.error, task.pow_hash, task.workload,
                task.created_at.isoformat(),
                task.started_at.isoformat() if task.started_at else None,
                task.completed_at.isoformat() if task.completed_at else None,
                ','.join(task.depends_on)
            ))
        return task
    
    def get(self, task_id: str) -> Optional[Task]:
        """获取任务"""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                'SELECT * FROM tasks WHERE id = ?', (task_id,)
            )
            row = cursor.fetchone()
            if row:
                return self._row_to_task(row)
        return None
    
    def update(self, task: Task) -> Task:
        """更新任务"""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute('''
                UPDATE tasks SET
                    status = ?, assigned_agent = ?, result = ?,
                    error = ?, pow_hash = ?, workload = ?,
                    started_at = ?, completed_at = ?
                WHERE id = ?
            ''', (
                task.status.value, task.assigned_agent, task.result,
                task.error, task.pow_hash, task.workload,
                task.started_at.isoformat() if task.started_at else None,
                task.completed_at.isoformat() if task.completed_at else None,
                task.id
            ))
        return task
    
    def list_by_status(self, status: TaskStatus, limit: int = 100) -> List[Task]:
        """按状态查询任务"""
        with sqlite3.connect(self.db_path) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute(
                'SELECT * FROM tasks WHERE status = ? ORDER BY created_at DESC LIMIT ?',
                (status.value, limit)
            )
            return [self._row_to_task(row) for row in cursor.fetchall()]
    
    def _row_to_task(self, row) -> Task:
        """行转Task对象"""
        return Task(
            id=row['id'],
            description=row['description'],
            status=TaskStatus(row['status']),
            priority=TaskPriority(row['priority']),
            assigned_agent=row['assigned_agent'],
            result=row['result'],
            error=row['error'],
            pow_hash=row['pow_hash'],
            workload=row['workload'],
            depends_on=row['depends_on'].split(',') if row['depends_on'] else []
        )
```

---

## 2. PoW 系统设计

### 2.1 PoW 数据模型

```python
# models/pow.py
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
import hashlib
import secrets

@dataclass
class ProofOfWork:
    """工作量证明"""
    task_id: str
    agent_id: str
    result_hash: str           # 任务结果的哈希
    timestamp: int              # 时间戳
    nonce: str = field(default_factory=lambda: secrets.token_hex(16))
    
    # 计算出的字段
    pow_hash: str = ""
    workload: float = 0.0
    
    def __post_init__(self):
        self._calculate()
    
    def _calculate(self):
        """计算PoW哈希和工作量"""
        # 构建PoW数据
        pow_data = f"{self.task_id}:{self.agent_id}:{self.result_hash}:{self.timestamp}:{self.nonce}"
        
        # 计算哈希
        self.pow_hash = hashlib.sha256(pow_data.encode()).hexdigest()
        
        # 计算工作量 = 时长 × 复杂度因子
        # 简化：基于结果哈希的某些特征
        hash_int = int(self.pow_hash[:8], 16)
        self.workload = hash_int / (16**8) * 1000  # 0-1000
    
    @staticmethod
    def verify(pow_data: dict) -> bool:
        """验证PoW"""
        pow = ProofOfWork(
            task_id=pow_data['task_id'],
            agent_id=pow_data['agent_id'],
            result_hash=pow_data['result_hash'],
            timestamp=pow_data['timestamp'],
            nonce=pow_data['nonce']
        )
        return pow.pow_hash == pow_data['pow_hash']
    
    def to_dict(self) -> dict:
        return {
            'task_id': self.task_id,
            'agent_id': self.agent_id,
            'result_hash': self.result_hash,
            'timestamp': self.timestamp,
            'nonce': self.nonce,
            'pow_hash': self.pow_hash,
            'workload': self.workload
        }
```

### 2.2 PoW 生成器

```python
# pow/generator.py
import time
from typing import Optional
from models.pow import ProofOfWork

class PoWGenerator:
    """PoW生成器"""
    
    def __init__(self, difficulty: int = 1):
        """
        Args:
            difficulty: 难度系数 (1-10)
        """
        self.difficulty = difficulty
    
    def generate(self, task_id: str, agent_id: str, result: str) -> ProofOfWork:
        """生成PoW"""
        result_hash = hashlib.sha256(result.encode()).hexdigest()
        
        pow = ProofOfWork(
            task_id=task_id,
            agent_id=agent_id,
            result_hash=result_hash,
            timestamp=int(time.time())
        )
        
        return pow
    
    def generate_with_stake(self, task_id: str, agent_id: str, 
                            result: str, stake_amount: float) -> dict:
        """生成带质押的PoW"""
        pow = self.generate(task_id, agent_id, result)
        
        return {
            'pow': pow.to_dict(),
            'stake': {
                'amount': stake_amount,
                'locked': True,
                'reason': 'task_execution'
            }
        }
    
    def verify(self, pow_data: dict) -> bool:
        """验证PoW"""
        return ProofOfWork.verify(pow_data)
```

---

## 3. 任务路由设计

### 3.1 简单路由规则

```python
# routing/simple_router.py
from typing import Literal
from dataclasses import dataclass

@dataclass
class RoutingDecision:
    agent_type: Literal['local', 'cloud']
    reason: str
    confidence: float  # 0-1

class SimpleRouter:
    """简单路由器 - 基于规则的路由"""
    
    # 路由阈值
    COMPLEXITY_THRESHOLD = 50
    LENGTH_THRESHOLD = 200
    
    def __init__(self):
        pass
    
    def decide(self, task_description: str, 
               available_agents: dict) -> RoutingDecision:
        """决定任务路由"""
        
        # 计算简单复杂度
        complexity = self._estimate_complexity(task_description)
        length = len(task_description)
        
        # 规则1: 任务长度过短 → 本地
        if length < 50:
            return RoutingDecision(
                agent_type='local',
                reason=f"任务简单 (长度={length})",
                confidence=0.9
            )
        
        # 规则2: 任务太长 → 云端
        if length > self.LENGTH_THRESHOLD:
            return RoutingDecision(
                agent_type='cloud',
                reason=f"任务复杂 (长度={length})",
                confidence=0.8
            )
        
        # 规则3: 包含特定关键词 → 云端
        cloud_keywords = ['分析', '报告', '调研', '开发', '编写', '创建']
        if any(kw in task_description for kw in cloud_keywords):
            return RoutingDecision(
                agent_type='cloud',
                reason="包含复杂任务关键词",
                confidence=0.7
            )
        
        # 默认: 本地
        return RoutingDecision(
            agent_type='local',
            reason="默认路由",
            confidence=0.6
        )
    
    def _estimate_complexity(self, description: str) -> float:
        """估算任务复杂度 (0-100)"""
        score = 0
        
        # 长度分数
        score += min(len(description) / 5, 30)
        
        # 关键词分数
        complex_words = ['分析', '比较', '设计', '开发', '创建', '实现']
        simple_words = ['查询', '获取', '告诉', '简单']
        
        for word in complex_words:
            if word in description:
                score += 10
        
        for word in simple_words:
            if word in description:
                score -= 5
        
        return max(0, min(100, score))
```

### 3.2 Agent接口

```python
# agents/base.py
from abc import ABC, abstractmethod
from typing import Optional
from dataclasses import dataclass

@dataclass
class ExecutionResult:
    success: bool
    result: Optional[str] = None
    error: Optional[str] = None
    duration: float = 0.0

class BaseAgent(ABC):
    """Agent基类"""
    
    def __init__(self, agent_id: str, name: str, endpoint: str):
        self.agent_id = agent_id
        self.name = name
        self.endpoint = endpoint
    
    @abstractmethod
    def execute(self, task: str) -> ExecutionResult:
        """执行任务"""
        pass
    
    @abstractmethod
    def health_check(self) -> bool:
        """健康检查"""
        pass

class LocalAgent(BaseAgent):
    """本地Agent"""
    
    def __init__(self):
        super().__init__(
            agent_id='local-001',
            name='OpenClaw Local',
            endpoint='http://localhost:8080'
        )
    
    def execute(self, task: str) -> ExecutionResult:
        """执行简单任务"""
        import time
        start = time.time()
        
        try:
            # MVP: 简单echo
            result = f"[Local] Processed: {task}"
            return ExecutionResult(
                success=True,
                result=result,
                duration=time.time() - start
            )
        except Exception as e:
            return ExecutionResult(
                success=False,
                error=str(e),
                duration=time.time() - start
            )
    
    def health_check(self) -> bool:
        return True

class CloudAgent(BaseAgent):
    """云端Agent"""
    
    def __init__(self):
        super().__init__(
            agent_id='cloud-001',
            name='OpenManus Cloud',
            endpoint='http://cloud:8080'
        )
    
    def execute(self, task: str) -> ExecutionResult:
        """执行复杂任务"""
        import time
        start = time.time()
        
        try:
            # MVP: 模拟云端处理
            result = f"[Cloud] Analyzed: {task}\n[Cloud] Generated report with insights."
            return ExecutionResult(
                success=True,
                result=result,
                duration=time.time() - start
            )
        except Exception as e:
            return ExecutionResult(
                success=False,
                error=str(e),
                duration=time.time() - start
            )
    
    def health_check(self) -> bool:
        return True
```

---

## 4. 积分/奖励系统（简化版MEME）

```python
# economy/simple_rewards.py
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, Optional

@dataclass
class AgentBalance:
    agent_id: str
    balance: float = 0.0          # 积分余额
    total_earned: float = 0.0     # 总获得
    total_spent: float = 0.0     # 总消耗
    task_count: int = 0          # 完成任务数

class SimpleRewardSystem:
    """简化版奖励系统"""
    
    # 奖励配置
    BASE_REWARD = 10.0           # 基础奖励
    WORKLOAD_MULTIPLIER = 0.1     # 工作量乘数
    QUALITY_BONUS = 1.5          # 质量bonus
    
    def __init__(self):
        self.balances: Dict[str, AgentBalance] = {}
    
    def calculate_reward(self, workload: float, quality_score: float = 1.0) -> float:
        """计算奖励"""
        reward = self.BASE_REWARD + (workload * self.WORKLOAD_MULTIPLIER)
        reward *= quality_score
        return round(reward, 2)
    
    def award(self, agent_id: str, workload: float, 
              quality_score: float = 1.0) -> float:
        """发放奖励"""
        # 初始化余额
        if agent_id not in self.balances:
            self.balances[agent_id] = AgentBalance(agent_id=agent_id)
        
        # 计算奖励
        reward = self.calculate_reward(workload, quality_score)
        
        # 更新余额
        balance = self.balances[agent_id]
        balance.balance += reward
        balance.total_earned += reward
        balance.task_count += 1
        
        return reward
    
    def get_balance(self, agent_id: str) -> Optional[AgentBalance]:
        """查询余额"""
        return self.balances.get(agent_id)
    
    def get_leaderboard(self, limit: int = 10) -> list:
        """排行榜"""
        sorted_agents = sorted(
            self.balances.values(),
            key=lambda x: x.balance,
            reverse=True
        )
        return [
            {
                'rank': i+1,
                'agent_id': a.agent_id,
                'balance': a.balance,
                'task_count': a.task_count
            }
            for i, a in enumerate(sorted_agents[:limit])
        ]
```

---

## 5. API 接口设计

```python
# api/routes.py
from flask import Flask, request, jsonify
from datetime import datetime
import time

from models.task import Task, TaskStatus, TaskPriority
from storage.task_storage import TaskStorage
from pow.generator import PoWGenerator
from routing.simple_router import SimpleRouter
from agents.base import LocalAgent, CloudAgent
from economy.simple_rewards import SimpleRewardSystem

app = Flask(__name__)

# 初始化组件
storage = TaskStorage()
pow_generator = PoWGenerator()
router = SimpleRouter()
local_agent = LocalAgent()
cloud_agent = CloudAgent()
rewards = SimpleRewardSystem()

# 路由映射
AGENT_MAP = {
    'local': local_agent,
    'cloud': cloud_agent
}

# ========== 任务API ==========

@app.route('/api/tasks', methods=['POST'])
def create_task():
    """创建任务"""
    data = request.json
    
    task = Task(
        description=data['description'],
        priority=TaskPriority(data.get('priority', 2))
    )
    
    storage.create(task)
    
    return jsonify({
        'success': True,
        'task': task.to_dict()
    })

@app.route('/api/tasks/<task_id>', methods=['GET'])
def get_task(task_id):
    """获取任务"""
    task = storage.get(task_id)
    if task:
        return jsonify({'success': True, 'task': task.to_dict()})
    return jsonify({'success': False, 'error': 'Task not found'}), 404

@app.route('/api/tasks/<task_id>/execute', methods=['POST'])
def execute_task(task_id):
    """执行任务"""
    task = storage.get(task_id)
    if not task:
        return jsonify({'success': False, 'error': 'Task not found'}), 404
    
    # 1. 路由决策
    decision = router.decide(task.description, AGENT_MAP)
    task.assigned_agent = decision.agent_type
    
    # 2. 选择Agent执行
    agent = AGENT_MAP[decision.agent_type]
    task.status = TaskStatus.EXECUTING
    task.started_at = datetime.now()
    storage.update(task)
    
    # 3. 执行
    result = agent.execute(task.description)
    
    # 4. 处理结果
    task.completed_at = datetime.now()
    if result.success:
        task.status = TaskStatus.COMPLETED
        task.result = result.result
        
        # 5. 生成PoW
        pow = pow_generator.generate(task.id, agent.agent_id, result.result)
        task.pow_hash = pow.pow_hash
        task.workload = pow.workload
        
        # 6. 发放奖励
        reward = rewards.award(agent.agent_id, pow.workload)
    else:
        task.status = TaskStatus.FAILED
        task.error = result.error
    
    storage.update(task)
    
    return jsonify({
        'success': True,
        'task': task.to_dict(),
        'reward': reward if result.success else 0
    })

@app.route('/api/tasks', methods=['GET'])
def list_tasks():
    """任务列表"""
    status = request.args.get('status')
    limit = int(request.args.get('limit', 100))
    
    if status:
        tasks = storage.list_by_status(TaskStatus(status), limit)
    else:
        # 返回所有
        with storage._init_db() or True:
            pass  # TODO
    
    return jsonify({
        'success': True,
        'tasks': [t.to_dict() for t in tasks]
    })

# ========== 积分API ==========

@app.route('/api/agents/<agent_id>/balance', methods=['GET'])
def get_balance(agent_id):
    """查询积分"""
    balance = rewards.get_balance(agent_id)
    if balance:
        return jsonify({
            'success': True,
            'balance': {
                'agent_id': balance.agent_id,
                'balance': balance.balance,
                'total_earned': balance.total_earned,
                'task_count': balance.task_count
            }
        })
    return jsonify({'success': False, 'error': 'Agent not found'}), 404

@app.route('/api/leaderboard', methods=['GET'])
def get_leaderboard():
    """排行榜"""
    return jsonify({
        'success': True,
        'leaderboard': rewards.get_leaderboard()
    })

# ========== 健康检查 ==========

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'ok', 'timestamp': int(time.time())})

if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

---

## 6. 完整文件结构（MVP）

```
nautilus-mvp/
├── main.py                 # 入口 + API (100行)
├── models/
│   ├── __init__.py
│   ├── task.py            # 任务模型 (50行)
│   └── pow.py             # PoW模型 (30行)
├── storage/
│   ├── __init__.py
│   └── task_storage.py    # SQLite存储 (60行)
├── agents/
│   ├── __init__.py
│   └── base.py            # Agent基类 (40行)
├── routing/
│   ├── __init__.py
│   └── simple_router.py   # 简单路由 (40行)
├── pow/
│   ├── __init__.py
│   └── generator.py       # PoW生成 (30行)
├── economy/
│   ├── __init__.py
│   └── simple_rewards.py # 积分系统 (40行)
├── requirements.txt
└── nautilus.db           # SQLite数据库（自动创建）

# 总计: ~400行代码
```

---

**下期预告 Part 2**:
- 记忆系统设计
- 配置管理
- 测试用例
- 部署脚本
