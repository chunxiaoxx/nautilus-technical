# Nautilus 技术选型建议汇总

**整理**: 明 (OpenClaw)
**日期**: 2026-02-16
**目的**: 为 Claude Code 提供技术选型决策参考

---

## 1. 记忆系统部署

### 硬件配置
- **当前设备**: 6核12线程 + 32GB RAM
- **EverMemOS 需求**: ~18GB RAM (Milvus 8GB + ES 4GB + MongoDB 4GB + Redis 2GB)

**结论**: ✅ 配置足够，本地 Docker Compose 一键启动

### 部署命令
```bash
git clone https://github.com/EverMind-AI/EverMemOS.git
cd EverMemOS
docker compose up -d
uv sync
uv run python src/run.py
```

---

## 2. CrewAI 扩展策略

### 核心原则
> **扩展 CrewAI，不重复造轮子**

### 实现方式
```python
from crewai import Agent

class NautilusCoordinator(Agent):
    """继承 CrewAI Agent，扩展 PoW + 路由能力"""
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.pow_generator = PoWGenerator()
        self.router = DynamicRouter()
    
    # 在任务执行前后嵌入 PoW 和路由逻辑
```

### 需要扩展的功能
- [ ] 任务复杂度自动评估 → 路由决策
- [ ] PoW 生成与验证嵌入
- [ ] MEME 币激励触发

---

## 3. 区块链选择

### 推荐网络

| 网络 | Gas 费 | 速度 | 阶段 |
|------|--------|------|------|
| **Polygon Amoy** | ~$0.001 | 秒级 | 测试网 |
| **Polygon** | $0.01-0.1 | 秒级 | 主网 |
| Arbitrum | $0.1-0.5 | 秒级 | 备选 |
| Base | $0.01-0.1 | 秒级 | 备选 |
| Ethereum | $5-50 | 12s | ❌ 不推荐 |

**建议**: MVP 用 Polygon Amoy → 验证后切换 Polygon 主网

### Gas 费用分摊
```python
GAS_FEE = gas_price * gas_used

if task_value > threshold:
    user_pays = GAS_FEE * 0.3   # 大任务：项目方补贴
    project_pays = GAS_FEE * 0.7
else:
    user_pays = GAS_FEE          # 小任务：用户全付
```

---

## 4. OpenClaw + OpenManus 集成

### 集成方式
> 注：两者均为开源项目，API 稳定性可自行控制

```python
# 封装为 CrewAI Tools
from crewai.tools import BaseTool

class OpenClawTool(BaseTool):
    name = "openclaw_executor"
    description = "本地机器执行简单任务"
    
    def _run(self, task: str):
        return openclaw.execute(task)

class OpenManusTool(BaseTool):
    name = "openmanus_executor"
    description = "云端执行复杂任务"
    
    def _run(self, task: str):
        return openmanus.execute(task)
```

### 待确认
- [ ] OpenClaw 官方 SDK 获取方式
- [ ] OpenManus 官方 SDK 获取方式
- [ ] API 速率限制和认证方式

---

## 5. PoW 防作弊机制

### 推荐方案：质押 + 惩罚 + 哈希锁定

#### 5.1 哈希锁定（基础）
```python
import hashlib
import secrets

def generate_pow(task_result: dict) -> ProofOfWork:
    nonce = secrets.token_hex(16)  # 防彩虹表攻击
    data = f"{task_result['id']}:{task_result['hash']}:{timestamp}:{nonce}"
    pow_hash = hashlib.sha256(data.encode()).hexdigest()
    
    return ProofOfWork(
        task_id=task_result['id'],
        result_hash=task_result['hash'],
        timestamp=timestamp,
        nonce=nonce,
        pow_hash=pow_hash
    )
```

#### 5.2 质押博弈机制（推荐）
```python
class PoWGamification:
    """质押 + 惩罚机制"""
    
    STAKE_AMOUNT = 10  # MEME 币质押
    
    def submit_pow(self, agent_id: str, pow: ProofOfWork):
        # 1. 质押 MEME
        self.lock_stake(agent_id, self.STAKE_AMOUNT)
        
        # 2. 验证
        if self.verify(pow):
            # 3a. 通过：返还 + 奖励
            self.refund_stake(agent_id)
            self.mint_reward(agent_id, pow.workload * REWARD_RATE)
        else:
            # 3b. 失败：惩罚
            self.slash(agent_id, self.STAKE_AMOUNT * 0.5)
```

#### 5.3 可选：第三方验证
- 引入验证者节点（类似 PoS）
- 随机抽查 n% 的 PoW 结果
- 验证者获得抽查手续费

---

## 6. 安全架构

### 6.1 私钥管理方案

| 阶段 | 方案 | 安全性 |
|------|------|--------|
| 开发 | 加密配置文件 + 环境变量 | 中 |
| MVP | 热钱包 + 主密钥加密 | 中 |
| 生产 | MPC / 多签钱包 | 高 |

```python
# 开发阶段示例
# ~/.nautilus/secrets.yaml (权限 600)
wallet:
  private_key: "encrypted:xxx"  # 用主密钥加密
  main_key_id: "aws/kms/key-id"
```

### 6.2 敏感数据加密

```python
from cryptography.fernet import Fernet

class SecureStorage:
    def __init__(self):
        self.key = self._load_master_key()
        self.cipher = Fernet(self.key)
    
    def encrypt(self, data: str) -> str:
        return self.cipher.encrypt(data.encode()).decode()
    
    def decrypt(self, encrypted: str) -> str:
        return self.cipher.decrypt(encrypted.encode()).decode()
    
    def _load_master_key(self) -> bytes:
        # 生产环境: AWS KMS / HashiCorp Vault
        return os.environ['NAUTILUS_MASTER_KEY'].encode()
```

### 6.3 数据分类

| 数据类型 | 加密方式 | 存储位置 |
|----------|----------|----------|
| 私钥 | AES-256-GCM + KMS | 加密存储 |
| API Keys | AES-256 | Vault / KMS |
| 用户数据 | AES-256 | PostgreSQL |
| 记忆数据 | 可选 | EverMemOS |

---

## 待确认清单

### P0（必须确认）
- [ ] OpenClaw SDK 获取方式
- [ ] OpenManus SDK 获取方式
- [ ] Polygon 合约Gas估算

### P1（尽快确认）
- [ ] 验证者节点机制设计
- [ ] KMS/Vault 选型
- [ ] MEME 币经济模型参数

---

**整理人**: 明
**如有问题，请随时询问**
