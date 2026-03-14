# OpenClaw 关键配置与优化记录

> 这篇文章记录了 OpenClaw 的关键配置和优化，作为"恢复指南"，方便重装后快速恢复。

---

## 一、核心准则（最重要）

这是龙虾的"价值观"，决定了它的行为模式。

**位置**：`MEMORY.md` → 用户画像 → 核心准则

```
1. 绝对不臆造 — 不确定就先核实，所有输出基于可验证事实
2. 穷尽方法解决问题 — 不轻易说"做不到"，有限制就提供替代方案
3. 我能做的事情就不让用户做 — 凡是我能完成的任务，绝不推给用户
4. 犯过的错绝不再犯 — 从错误中学习，案例见下方
5. 永久遵循 — 以上准则贯穿所有交互
```

**犯过的错**：
- 成本/数据类陈述，必须拆分每个组件逐一核实，不能以偏概全

---

## 二、会话配置

### 2.1 防止频繁重置

**问题**：默认每天凌晨 4 点重置会话，第二天就失忆。

**配置**：

```json
"session": {
  "reset": {
    "mode": "idle",
    "idleMinutes": 20160
  }
}
```

**效果**：14 天不活跃才重置。

### 2.2 上下文溢出防护

**问题**：聊多了会崩，崩后 `/new` 恢复但记录全丢。

**配置**：

```json
"compaction": {
  "mode": "safeguard",
  "reserveTokens": 60000,
  "keepRecentTokens": 80000,
  "memoryFlush": {
    "enabled": true,
    "softThresholdTokens": 15000
  }
},
"contextPruning": {
  "mode": "cache-ttl",
  "ttl": "1h",
  "keepLastAssistants": 3,
  "softTrimRatio": 0.3,
  "hardClearRatio": 0.5
}
```

**效果**：自动压缩和剪枝，保留核心数据。

### 2.3 模型降级链

**问题**：主模型挂了或欠费，整个系统不可用。

**配置**：

```json
"model": {
  "primary": "tencentcodingplan/glm-5",
  "fallbacks": [
    "tencentcodingplan/hunyuan-2.0-thinking",
    "deepseek/deepseek-reasoner"
  ]
}
```

**效果**：主模型挂了自动切换备用。

---

## 三、联网能力

### 3.1 Tavily 搜索

**位置**：`skills/tavily-search/`

**配置**：环境变量 `TAVILY_API_KEY`

**触发词**：搜索、查一下、联网

---

## 四、深度研究能力

### 4.1 GPT-Researcher

**位置**：`/root/gpt-researcher/`

**配置文件**：`/root/gpt-researcher/.env`

```bash
# 腾讯云 GLM-5
OPENAI_API_KEY=你的腾讯云API密钥
OPENAI_BASE_URL=https://api.lkeap.cloud.tencent.com/coding/v3

# Tavily 搜索
TAVILY_API_KEY=你的Tavily密钥

# LLM 配置
FAST_LLM=openai:glm-5
SMART_LLM=openai:glm-5
STRATEGIC_LLM=openai:glm-5

# Embedding - 本地模型
EMBEDDING=huggingface:sentence-transformers/all-MiniLM-L6-v2

# 语言
LANGUAGE=chinese
```

**触发词**：研究、调研、探究、分析、报告

### 4.2 后处理规则（重要）

**位置**：`skills/gpt-researcher/SKILL.md`

调研完成后自动执行：

```
1. 翻译：英文内容翻译成中文
2. 格式优化：混乱内容结构化
3. 自动转 PDF：pandoc + xelatex
4. 主动发送：不用询问，直接发送给用户
```

**PDF 转换命令**：

```bash
pandoc 报告.md -o 报告.pdf \
  --toc --number-sections \
  --pdf-engine=xelatex \
  -V CJKmainfont="Noto Sans CJK SC" \
  -V geometry:margin=1in
```

---

## 五、长期记忆系统

### 5.1 memU 架构

```
OpenClaw 主会话
    │
    ├─▶ MEMORY.md（核心记忆，手动维护）
    │
    └─▶ sessions/*.jsonl（对话记录）
            │
            ▼ memUbot 守护进程（每 10 秒同步）
        长期记忆系统
            │
            ├─ 腾讯云 GLM-5：提取记忆项
            │
            ├─ 本地 bge-small-zh-v1.5：向量化（~0.1秒）
            │
            └─ PostgreSQL + pgvector：存储
```

### 5.2 memUbot 守护进程

**服务**：`systemd memubot.service`

**检查间隔**：10 秒

**命令**：

```bash
# 启动
systemctl start memubot

# 状态
systemctl status memubot

# 日志
journalctl -u memubot -f
```

### 5.3 新会话自动加载记忆

**位置**：`AGENTS.md`

```markdown
## Every Session

4. **查询最近 10 条记忆**：执行 `python3 /root/.openclaw/workspace/skills/memu/local_memory.py recent 10`
```

**效果**：新会话自动了解用户偏好和历史。

### 5.4 memU 命令

```bash
# 查询最近 N 条记忆
python local_memory.py recent [N]

# 高重要性记忆
python local_memory.py important [N]

# 最近 N 天记忆
python local_memory.py since [days]

# 语义搜索
python local_memory.py retrieve "关键词"
```

---

## 六、写作流水线

### 6.1 三种场景

| 场景 | 触发词 | 流程 |
|------|--------|------|
| 新建文章 | 写文章、新建文章、帮我写一篇 | 调研 → 写作 → 润色 |
| 优化文章 | 优化文章、润色文章、帮我改 | 诊断 → 补充 → 重构 → 润色 |
| 快速观点文 | 快速写、写个短文、观点文 | 构思 → 写作 → 润色 |

### 6.2 输出位置

- 初稿：`tmp/drafts/`
- 最终稿：用户指定或 `docs/`
- 调研报告：`reports/`

---

## 七、已安装的核心技能

| 技能 | 位置 | 用途 |
|-----|------|------|
| gpt-researcher | skills/gpt-researcher | 深度研究 |
| tavily-search | skills/tavily-search | 联网搜索 |
| pdf-convert | skills/pdf-convert | 转 PDF |
| writing-pipeline | skills/writing-pipeline | 写作流水线 |
| github | skills/github | GitHub 操作 |
| summarize | skills/summarize | 摘要 |
| weather | skills/weather | 天气 |

---

## 八、API Keys 汇总

| 服务 | 用途 | 获取方式 |
|-----|------|---------|
| 腾讯云 | GLM-5 主力模型 | 腾讯云控制台 |
| DeepSeek | 备用模型 | DeepSeek 官网 |
| Tavily | 联网搜索 | Tavily 官网 |

---

## 九、工作习惯

1. **文件要整理好** — 临时文件放 `tmp/`，报告放 `reports/`，用完及时清理
2. **重要的事情写文件** — "脑记"不靠谱，文件才持久
3. **有记录就必须先查** — 时间、状态、数据相关，开口前先查记录

---

## 十、恢复清单

重装后按顺序执行：

```
1. 安装 OpenClaw
2. 配置 openclaw.json（会话、上下文、模型降级）
3. 安装 Tavily（联网能力）
4. 安装 GPT-Researcher（深度研究）
5. 配置 memU（长期记忆）
6. 启动 memUbot 守护进程
7. 复制 skills/ 目录（技能体系）
8. 复制 MEMORY.md + AGENTS.md（核心准则 + 记忆）
9. 复制 USER.md + SOUL.md（用户画像 + 身份）
```

---

*最后更新：2026-03-14*

*本文是 OpenClaw 的"恢复指南"，记录关键配置和优化，方便重装后快速恢复。*
