# Nautilus MVP 降维策略 - 如何降低开发难度

**整理**: 明 (OpenClaw)
**日期**: 2026-02-16
**核心目标**: 最小化开发复杂度，快速验证核心假设

---

## 🎯 核心原则

> **MVP = 最小可行产品，不是最小功能产品**
> 
> 去掉所有"以后可能会有用"的功能，专注于验证核心假设

---

## 1. 架构简化

### 当前问题：7层架构太重

```
当前方案: 用户 → NMACS → 任务引擎 → Agent → PoW → 区块链 → MEME → 记忆
```

### 简化方案：3层架构

```
MVP方案: 用户 → Agent(简化) → PoW(本地) → 记忆(本地)
```

### 对比

| 模块 | 当前 | MVP | 简化方式 |
|------|------|-----|----------|
| NMACS通信 | Socket.IO + Redis | Webhook轮询 | 直接HTTP调用 |
| 任务队列 | Celery | 同步执行 | 去掉队列 |
| 区块链 | Polygon实时 | 本地模拟 | Mock + 本地存储 |
| MEME经济 | 智能合约 | 数据库记录 | 积分系统 |
| 记忆系统 | EverMemOS | SQLite | 轻量存储 |
| 多Agent | CrewAI | 单Agent | 硬编码路由 |

---

## 2. MVP 功能清单

### 核心功能（必须）

| # | 功能 | 描述 | 预估工时 |
|---|------|------|----------|
| 1 | 任务提交 | 用户输入任务 | 2h |
| 2 | 任务执行 | Agent执行任务 | 4h |
| 3 | PoW生成 | 本地生成证明 | 2h |
| 4 | 结果存储 | SQLite存储结果 | 2h |
| 5 | 任务列表 | 查看历史任务 | 2h |

### 扩展功能（验证后加）

| # | 功能 | 描述 | 预估工时 |
|---|------|------|----------|
| 6 | 多Agent路由 | 本地/云端分发 | 8h |
| 7 | 记忆系统 | 上下文记忆 | 8h |
| 8 | 区块链上链 | 模拟→真实 | 16h |
| 9 | MEME积分 | 经济系统 | 16h |
| 10 | Telegram集成 | Bot交互 | 8h |

### MVP = 功能1-5，总计 ~12小时

---

## 3. 技术栈降级

### 当前 vs MVP

| 层级 | 当前方案 | MVP方案 | 替代方案 |
|------|----------|---------|----------|
| **Web框架** | FastAPI | Flask | 更简单 |
| **数据库** | PostgreSQL | SQLite | 单文件 |
| **缓存/队列** | Redis | 内存dict | 不需要 |
| **记忆系统** | EverMemOS | JSON文件 | 最简化 |
| **区块链** | Polygon | Mock类 | 本地模拟 |
| **智能合约** | Solidity | 不需要 | 积分记账 |
| **多Agent** | CrewAI | 硬编码if/else | 单Agent |

### MVP 技术栈

```python
# requirements.txt
flask==3.0.0          # Web框架
sqlite3               # 内置，不需要装
requests==2.31.0      # HTTP调用
python-dotenv==1.0.0  # 环境变量
```

**对比**：
- 当前：需要 10+ 依赖，4个Docker服务
- MVP：只需 4个依赖，无Docker

---

## 4. 开发流程优化

### 阶段1：快速验证（1-2周）

```python
# main.py - 单文件MVP
from flask import Flask, request, jsonify
import sqlite3
import hashlib
import time

app = Flask(__name__)

# 初始化数据库
def init_db():
    conn = sqlite3.connect('nautilus.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE tasks (
        id TEXT PRIMARY KEY,
        description TEXT,
        status TEXT,
        result TEXT,
        pow_hash TEXT,
        created_at INTEGER
    )''')
    conn.commit()
    conn.close()

# 1. 提交任务
@app.route('/tasks', methods=['POST'])
def create_task():
    data = request.json
    task_id = hashlib.md5(f"{time.time()}".encode()).hexdigest()
    
    # 2. 执行任务（简化版）
    result = execute_task(data['description'])
    
    # 3. 生成PoW（本地）
    pow_hash = hashlib.sha256(f"{task_id}{result}".encode()).hexdigest()
    
    # 4. 存储
    conn = sqlite3.connect('nautilus.db')
    c = conn.cursor()
    c.execute('INSERT INTO tasks VALUES (?,?,?,?,?,?)',
        (task_id, data['description'], 'completed', result, pow_hash, int(time.time())))
    conn.commit()
    conn.close()
    
    return jsonify({'task_id': task_id, 'result': result, 'pow_hash': pow_hash})

# 简化任务执行
def execute_task(description):
    # MVP: 简单 echo + 固定逻辑
    return f"Executed: {description}"

# 5. 查询任务
@app.route('/tasks/<task_id>')
def get_task(task_id):
    conn = sqlite3.connect('nautilus.db')
    c = conn.cursor()
    c.execute('SELECT * FROM tasks WHERE id = ?', (task_id,))
    row = c.fetchone()
    conn.close()
    if row:
        return jsonify({
            'id': row[0], 'description': row[1], 
            'status': row[2], 'result': row[3], 
            'pow_hash': row[4], 'created_at': row[5]
        })
    return jsonify({'error': 'Not found'}), 404

if __name__ == '__main__':
    init_db()
    app.run(port=5000)
```

**运行**：
```bash
pip install flask
python main.py
```

**测试**：
```bash
curl -X POST http://localhost:5000/tasks -H "Content-Type: application/json" -d '{"description":"Hello"}'
curl http://localhost:5000/tasks/<task_id>
```

---

### 阶段2：增加复杂度（3-4周）

```python
# v2: 加入Agent路由

AGENTS = {
    'local': {'type': 'openclaw', 'endpoint': 'http://localhost:8080'},
    'cloud': {'type': 'openmanus', 'endpoint': 'http://cloud:8080'}
}

def route_task(description):
    # 简单规则路由
    if len(description) < 100:
        return AGENTS['local']
    else:
        return AGENTS['cloud']

def execute_task(description):
    agent = route_task(description)
    # 调用实际Agent
    response = requests.post(f"{agent['endpoint']}/execute", 
                           json={'task': description})
    return response.json()['result']
```

---

### 阶段3：加入区块链模拟（5-6周）

```python
# v3: 本地区块链模拟

class LocalChain:
    def __init__(self):
        self.blocks = []
        self.pending_pows = []
    
    def add_pow(self, pow.pending_pows.append(pow_data)
_data):
        self    
    def mine(self):
        # 简化：直接添加区块
        block = {
            'index': len(self.blocks),
            'pow': self.pending_pows.copy(),
            'timestamp': time.time()
        }
        self.blocks.append(block)
        self.pending_pows = []
        return block

# 使用
chain = LocalChain()
chain.add_pow(pow_hash)
chain.mine()
```

---

## 5. 开发优先级

### 每周计划

| 周 | 目标 | 交付物 | 复杂度 |
|----|------|--------|--------|
| **1** | 跑通基础流程 | 单文件Web服务 | ⭐ |
| **2** | 任务执行 | Agent调用 | ⭐⭐ |
| **3** | PoW本地化 | 哈希生成 | ⭐⭐ |
| **4** | 数据持久化 | SQLite存储 | ⭐⭐ |
| **5** | Agent路由 | 本地/云端分发 | ⭐⭐⭐ |
| **6** | 记忆系统 | 上下文记忆 | ⭐⭐⭐⭐ |
| **7-8** | 区块链模拟 | 本地链 | ⭐⭐⭐ |
| **9-10** | 真实区块链 | Polygon集成 | ⭐⭐⭐⭐⭐ |
| **11-12** | MEME经济 | 积分系统 | ⭐⭐⭐⭐ |
| **13-14** | Telegram Bot | 用户交互 | ⭐⭐⭐ |

---

## 6. 降维技巧

### 1. 用Mock代替真实服务

```python
# 区块链Mock
class MockBlockchain:
    def submit(self, data):
        return {'tx_hash': f"mock_{hash(data)}", 'confirmed': True}

# 替换
blockchain = MockBlockchain()  # 开发时
# blockchain = PolygonBlockchain()  # 生产时
```

### 2. 用单文件代替多模块

```
当前: nautilus/{nmacs,engine,agents,pow,economy,blockchain,memory,db}/__init__.py
MVP: main.py
```

### 3. 用硬编码代替配置

```python
# 配置
config = {'api_key': os.environ['API_KEY']}  # 复杂

# 硬编码
API_KEY = "test_key_123"  # MVP
```

### 4. 用同步代替异步

```python
# 异步（复杂）
async def execute():
    result = await agent.execute(task)

# 同步（简单）
def execute():
    result = requests.post(url, json={'task': task})
```

### 5. 用SQLite代替分布式数据库

```python
# PostgreSQL
conn = psycopg2.connect(host='localhost', database='nautilus')

# SQLite
conn = sqlite3.connect('nautilus.db')  # 单文件，无需安装
```

---

## 7. 验证标准

### MVP完成标志

| 指标 | 目标 | 验证方式 |
|------|------|----------|
| 功能可用性 | 任务能提交、执行、查询 | 手动测试 |
| 响应时间 | < 2秒 | 计时测试 |
| 成功率 | > 90% | 100次测试 |
| 代码行数 | < 500行 | wc -l |
| 依赖数量 | < 10个 | pip list |

### 核心假设验证

| 假设 | 验证方式 |
|------|----------|
| 用户愿意提交任务 | 10+ 测试用户 |
| Agent能执行任务 | 50+ 任务执行 |
| PoW能生成 | 100% 成功率 |
| 记忆能存储 | 10+ 上下文任务 |

---

## 8. 风险控制

### 开发风险

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 依赖安装失败 | 中 | 高 | Docker镜像 |
| 第三方API不稳定 | 高 | 中 | Mock替代 |
| 复杂度失控 | 高 | 高 | 每周重构 |
| 目标偏离 | 中 | 高 | 每日站会 |

### 技术风险

| 风险 | 缓解方案 |
|------|----------|
| CrewAI太重 | 用简单if/else |
| EverMemOS部署难 | 用JSON文件 |
| 区块链Gas高 | 先用Mock |
| Telegram Bot复杂 | 用Web界面 |

---

## 9. 推荐开发顺序

```
第1步: Flask + SQLite 骨架
    ↓
第2步: 任务提交/执行/查询 API
    ↓
第3步: 本地执行器（echo/固定逻辑）
    ↓
第4步: PoW 生成和存储
    ↓
第5步: Web界面（可选）
    ↓
第6步: 真实Agent集成
    ↓
第7步: 区块链模拟 → 真实
    ↓
第8步: MEME积分
    ↓
第9步: Telegram Bot
```

---

## 10. 总结

### MVP = 3个文件

```
nautilus-mvp/
├── main.py          # Web服务 (~100行)
├── database.py      # SQLite操作 (~50行)
├── agents.py        # Agent调用 (~50行)
└── requirements.txt # 依赖 (~5行)
```

### 总计

- **代码量**: ~200行
- **依赖**: Flask, requests (2个)
- **部署**: `python main.py`
- **验证时间**: 2周

---

**下一步**: 你想让我先写哪个模块的MVP代码？

1. 基础Web服务（Flask + SQLite）
2. 任务执行 + PoW
3. 或者是其他？

