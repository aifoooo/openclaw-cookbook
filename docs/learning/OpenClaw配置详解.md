# openclaw.json 完整配置详解

OpenClaw 的核心配置文件，控制智能体的行为、内存管理、模型选择等。

---

## 一、配置文件结构

### 1.1 文件位置

```
~/.openclaw/openclaw.json
```

支持 JSON5 格式（允许注释、尾逗号）。

### 1.2 目录结构

```
~/.openclaw/
├── openclaw.json          # 主配置文件
├── sessions/              # 会话数据
├── agents/                # 智能体工作区
├── credentials/           # 凭证存储
└── skills/                # 技能目录
```

---

## 二、核心配置项

### 2.1 会话配置（session）

控制会话生命周期和重置策略。

```json
{
  "session": {
    "reset": {
      "mode": "idle",
      "idleMinutes": 20160
    }
  }
}
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `mode` | 重置模式：`idle`（空闲重置）、`schedule`（定时重置） | `idle` |
| `idleMinutes` | 空闲多少分钟后重置 | `20160`（14天） |

### 2.2 模型配置（model）

控制模型选择和降级链。

```json
{
  "model": {
    "primary": "tencentcodingplan/glm-5",
    "fallbacks": [
      "tencentcodingplan/hunyuan-2.0-thinking",
      "deepseek/deepseek-reasoner"
    ]
  }
}
```

| 参数 | 说明 |
|------|------|
| `primary` | 主模型 |
| `fallbacks` | 降级链，主模型失败时依次尝试 |

### 2.3 工具权限配置（tools）

控制智能体能执行哪些操作。

```json
{
  "tools": {
    "allow": {
      "exec": ["npm", "git", "ls", "cat"],
      "web_search": true,
      "read": true,
      "write": true
    }
  }
}
```

| 参数 | 说明 |
|------|------|
| `allow.exec` | 允许执行的命令列表 |
| `allow.web_search` | 是否允许联网搜索 |
| `allow.read/write` | 是否允许文件操作 |

---

## 三、内存管理配置（重要）

### 3.1 核心原则

**如果未写入文件，即不存在。**

智能体的长期记忆完全依赖磁盘文件，对话历史在压缩后会丢失。

### 3.2 上下文压缩（compaction）

当对话历史填满上下文窗口时触发。

```json
{
  "compaction": {
    "mode": "safeguard",
    "reserveTokens": 60000,
    "keepRecentTokens": 80000,
    "memoryFlush": {
      "enabled": true,
      "softThresholdTokens": 15000,
      "systemPrompt": "Session nearing compaction. Store durable memories now.",
      "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
    }
  }
}
```

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `mode` | 压缩模式 | `safeguard` |
| `reserveTokens` | 保留的 token 数量 | `60000` |
| `keepRecentTokens` | 保留最近多少 token | `80000` |
| `memoryFlush.enabled` | 启用预压缩内存刷新 | `true` |
| `memoryFlush.softThresholdTokens` | 触发内存刷新的阈值 | `15000` |

### 3.3 上下文修剪（contextPruning）

在内存中修剪旧的工具调用结果。

```json
{
  "contextPruning": {
    "mode": "cache-ttl",
    "ttl": "1h",
    "keepLastAssistants": 3,
    "softTrimRatio": 0.3,
    "hardClearRatio": 0.5
  }
}
```

| 参数 | 说明 |
|------|------|
| `mode` | 修剪模式：`cache-ttl`（缓存 TTL） |
| `ttl` | 缓存有效期 |
| `keepLastAssistants` | 保留最近几次助手回复 |
| `softTrimRatio` | 软修剪比例 |
| `hardClearRatio` | 硬清除比例 |

---

## 四、压缩生命周期

### 4.1 好路径（维护性压缩）

```
上下文接近阈值
    ↓
触发预压缩内存刷新
    ↓
智能体静默保存重要信息到磁盘
    ↓
执行压缩（生成摘要）
    ↓
智能体继续工作（摘要 + 最新消息 + 磁盘文件）
```

### 4.2 坏路径（溢出恢复）

```
上下文过大导致 API 拒绝
    ↓
进入损害控制模式
    ↓
强制压缩所有内容
    ↓
无内存刷新，关键信息丢失
```

**目标**：通过配置预压缩内存刷新，始终走"好路径"。

---

## 五、配置编辑方式

### 5.1 交互式向导

```bash
openclaw onboard
openclaw configure
```

### 5.2 CLI 命令

```bash
# 查看配置
openclaw config get session.reset.mode

# 设置配置
openclaw config set session.reset.idleMinutes 20160

# 删除配置
openclaw config unset session.reset.schedule
```

### 5.3 控制 UI

访问 `http://127.0.0.1:18789` 的 Config 标签页。

### 5.4 直接编辑

直接修改 `openclaw.json`，网关自动监听并应用更改。

---

## 六、热加载模式

```json
{
  "hotReload": "hybrid"
}
```

| 模式 | 说明 |
|------|------|
| `hybrid` | 安全更改即时生效，关键更改自动重启（默认） |
| `hot` | 仅应用安全更改，需重启时记录警告 |
| `restart` | 任何更改均重启网关 |
| `off` | 关闭热加载 |

---

## 七、完整配置示例

```json
{
  "session": {
    "reset": {
      "mode": "idle",
      "idleMinutes": 20160
    }
  },
  "model": {
    "primary": "tencentcodingplan/glm-5",
    "fallbacks": [
      "tencentcodingplan/hunyuan-2.0-thinking",
      "deepseek/deepseek-reasoner"
    ]
  },
  "tools": {
    "allow": {
      "exec": ["npm", "git", "ls", "cat", "mkdir", "rm"],
      "web_search": true,
      "read": true,
      "write": true,
      "edit": true
    }
  },
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
  },
  "hotReload": "hybrid"
}
```

---

## 八、关键建议

### 8.1 三大关键建议

1. **规则文件化**：将持久性规则写入 `MEMORY.md` 和 `AGENTS.md`，而非对话中临时指令
2. **启用内存刷新**：配置预压缩内存刷新，防止关键信息丢失
3. **强制检索**：在 `AGENTS.md` 中设定规则，要求行动前必须搜索记忆

### 8.2 压缩时会丢失的内容

- 对话中嵌入的指令
- 用户偏好与纠正
- 压缩前的图片
- 工具调用结果及其上下文
- 原始指令的细微差别

### 8.3 压缩时会保留的内容

- 工作区文件（`SOUL.md`, `AGENTS.md`, `USER.md`, `MEMORY.md`, `TOOLS.md`）
- 每日记忆日志（`memory/YYYY-MM-DD.md`）
- 压缩前写入磁盘的内容
- 最近约 20,000 tokens 的消息

---

*本文基于 OpenClaw 官方文档和社区最佳实践整理。*
