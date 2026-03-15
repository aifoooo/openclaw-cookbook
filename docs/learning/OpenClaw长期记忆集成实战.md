# OpenClaw 长期记忆集成实战

配置 memU 记忆系统，让 OpenClaw 拥有真正的长期记忆能力。

---

## 一、为什么需要长期记忆？

### 1.1 OpenClaw 默认记忆机制

| 机制 | 持久性 | 容量 |
|-----|-------|------|
| 上下文 | 会话级 | 受模型限制 |
| MEMORY.md | 永久 | 需手动维护 |
| 会话 JSONL | 永久 | 只存储不检索 |

### 1.2 memU 的优势

| 特性 | 说明 |
|-----|------|
| **自动记忆** | 从对话中自动提取重要信息 |
| **语义搜索** | 按意义搜索，不只是关键词 |
| **重要性排序** | 自动识别重要记忆 |
| **过期清理** | 自动清理过期记忆 |

---

## 二、memU 架构

### 2.1 组件

```
memU/
├── local_memory.py    # 本地记忆操作
├── memubot.py         # 守护进程
├── SKILL.md           # Skill 定义
└── config.json        # 配置文件
```

### 2.2 数据流

```
对话发生
    ↓
memUbot 监听
    ↓
提取记忆项（LLM）
    ↓
生成 Embedding（本地模型）
    ↓
存入 PostgreSQL
    ↓
下次对话时检索
```

---

## 三、环境准备

### 3.1 安装依赖

```bash
# Python 依赖
pip install psycopg2-binary sentence-transformers pgvector

# PostgreSQL + pgvector
# Ubuntu/Debian
sudo apt install postgresql postgresql-contrib
sudo apt install postgresql-16-pgvector  # 或对应版本
```

### 3.2 创建数据库

```sql
-- 连接 PostgreSQL
sudo -u postgres psql

-- 创建数据库
CREATE DATABASE memu;

-- 启用 pgvector
\c memu
CREATE EXTENSION IF NOT EXISTS vector;

-- 创建记忆表
CREATE TABLE memories (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(512),
    importance FLOAT DEFAULT 0.5,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    metadata JSONB
);

-- 创建向量索引
CREATE INDEX ON memories USING ivfflat (embedding vector_cosine_ops);
```

### 3.3 下载 Embedding 模型

```bash
# 模型会自动下载到 ~/.cache/memvid/text-models/
# 首次运行时会自动下载 bge-small-zh-v1.5
```

---

## 四、配置 memU

### 4.1 创建配置文件

```json
// ~/.openclaw/workspace/skills/memu/config.json
{
  "database": {
    "host": "localhost",
    "port": 5432,
    "database": "memu",
    "user": "postgres",
    "password": "your-password"
  },
  "embedding": {
    "model": "BAAI/bge-small-zh-v1.5",
    "dimension": 512
  },
  "llm": {
    "provider": "tencentcloud",
    "model": "glm-5",
    "api_key": "your-api-key"
  },
  "memory": {
    "default_importance": 0.5,
    "expiry_days": 30
  }
}
```

### 4.2 配置 OpenClaw

```json
// ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace"
    }
  }
}
```

---

## 五、安装 memU Skill

### 5.1 克隆或复制

```bash
# 如果从仓库安装
git clone https://github.com/your-repo/memu.git ~/.openclaw/workspace/skills/memu

# 或直接复制文件
mkdir -p ~/.openclaw/workspace/skills/memu
# 复制 local_memory.py, memubot.py, SKILL.md
```

### 5.2 重载 Skill

```bash
openclaw skills reload
```

---

## 六、启动 memUbot 守护进程

### 6.1 手动启动

```bash
# 前台运行
python3 ~/.openclaw/workspace/skills/memu/memubot.py

# 后台运行
nohup python3 ~/.openclaw/workspace/skills/memu/memubot.py > /tmp/memubot.log 2>&1 &
```

### 6.2 配置为系统服务

```bash
# 创建服务文件
sudo tee /etc/systemd/system/memubot.service << EOF
[Unit]
Description=memU Memory Bot
After=network.target postgresql.service

[Service]
Type=simple
User=root
WorkingDirectory=/root/.openclaw/workspace/skills/memu
ExecStart=/usr/bin/python3 /root/.openclaw/workspace/skills/memu/memubot.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# 启动服务
sudo systemctl daemon-reload
sudo systemctl start memubot
sudo systemctl enable memubot
```

### 6.3 检查状态

```bash
# 查看服务状态
sudo systemctl status memubot

# 查看日志
sudo journalctl -u memubot -f
```

---

## 七、使用 memU

### 7.1 查询记忆

```bash
# 最近 10 条记忆
python3 ~/.openclaw/workspace/skills/memu/local_memory.py recent 10

# 高重要性记忆
python3 ~/.openclaw/workspace/skills/memu/local_memory.py important 5

# 最近 7 天记忆
python3 ~/.openclaw/workspace/skills/memu/local_memory.py since 7

# 语义搜索
python3 ~/.openclaw/workspace/skills/memu/local_memory.py retrieve "用户偏好"
```

### 7.2 与 OpenClaw 集成

在 `AGENTS.md` 中添加：

```markdown
## Every Session

Before doing anything else:

1. Read `SOUL.md`
2. Read `USER.md`
3. Read `memory/YYYY-MM-DD.md`
4. **查询最近 10 条记忆**：
   python3 /root/.openclaw/workspace/skills/memu/local_memory.py recent 10
5. If in MAIN SESSION: Also read `MEMORY.md`
```

---

## 八、记忆类型

### 8.1 自动记忆类型

| 类型 | 说明 | 示例 |
|-----|------|------|
| preference | 用户偏好 | 用户偏好文件名大写开头 |
| goal | 目标 | 用户想深入学习 OpenClaw |
| fact | 事实 | 用户有 GitHub 仓库 aifoooo/openclaw-cookbook |
| decision | 决策 | 用户决定使用腾讯云作为主力模型 |

### 8.2 记忆重要性

| 重要性 | 说明 |
|-------|------|
| 0.9+ | 核心偏好、关键决策 |
| 0.7-0.9 | 重要信息 |
| 0.5-0.7 | 一般信息 |
| < 0.5 | 琐碎信息 |

---

## 九、维护记忆

### 9.1 清理过期记忆

```bash
# 清理 30 天前的记忆
python3 ~/.openclaw/workspace/skills/memu/local_memory.py cleanup 30
```

### 9.2 手动添加记忆

```python
from local_memory import add_memory

add_memory(
    content="用户偏好使用腾讯云 GLM-5 模型",
    memory_type="preference",
    importance=0.8
)
```

### 9.3 查看记忆统计

```bash
# 列出所有记忆
python3 ~/.openclaw/workspace/skills/memu/local_memory.py list
```

---

## 十、性能优化

### 10.1 Embedding 缓存

模型会缓存到 `~/.cache/memvid/text-models/`，无需重复下载。

### 10.2 数据库索引

```sql
-- 优化查询
CREATE INDEX idx_memories_created ON memories(created_at);
CREATE INDEX idx_memories_importance ON memories(importance);
```

### 10.3 批量插入

memUbot 默认每 10 秒批量同步一次，减少数据库压力。

---

## 十一、故障排查

### 11.1 数据库连接失败

**症状**：
```
Error: could not connect to server
```

**解决**：
```bash
# 检查 PostgreSQL 状态
sudo systemctl status postgresql

# 检查连接
psql -h localhost -U postgres -d memu
```

### 11.2 Embedding 模型加载失败

**症状**：
```
Error: Failed to load model
```

**解决**：
```bash
# 检查模型缓存
ls ~/.cache/memvid/text-models/

# 手动下载
python3 -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('BAAI/bge-small-zh-v1.5')"
```

### 11.3 memUbot 不记录记忆

**可能原因**：
- 守护进程未运行
- LLM API 配置错误
- 数据库连接问题

**排查**：
```bash
# 检查服务状态
sudo systemctl status memubot

# 查看日志
sudo journalctl -u memubot -n 100
```

---

## 十二、最佳实践

### 12.1 定期维护

```bash
# 每周清理过期记忆
# 添加到 crontab
0 0 * * 0 python3 /root/.openclaw/workspace/skills/memu/local_memory.py cleanup 30
```

### 12.2 监控记忆质量

```bash
# 查看最近记忆
python3 local_memory.py recent 20

# 检查是否有重复或错误记忆
```

### 12.3 备份数据库

```bash
# 备份
pg_dump memu > memu_backup.sql

# 恢复
psql memu < memu_backup.sql
```

---

## 十三、效果验证

配置完成后，OpenClaw 会：

1. **自动记录**：对话中的重要信息自动存入记忆
2. **智能检索**：新会话自动加载相关记忆
3. **长期记忆**：跨会话记住用户偏好和重要决策

---

*本文基于 memU 实战经验整理。*
