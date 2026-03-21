# OpenClaw 日志系统详解

> 了解 OpenClaw 的三套日志体系，掌握诊断标志、Cache Trace、OpenTelemetry 导出等高级功能。

---

## 一、日志架构总览

OpenClaw 有 **三套日志体系**：

| 体系 | 用途 | 格式 | 输出位置 |
|------|------|------|----------|
| **文件日志** | 持久化存储，故障排查 | JSONL | `/tmp/openclaw/openclaw-YYYY-MM-DD.log` |
| **控制台日志** | 实时查看，开发调试 | 文本/JSON | 终端 / Control UI |
| **诊断日志** | 性能监控，指标导出 | 结构化事件 | 本地文件 + OTLP 导出 |

---

## 二、文件日志

### 2.1 默认位置

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log   # 主日志（按日期滚动）
/tmp/openclaw/gateway.log                # Gateway 进程 stdout
/tmp/openclaw/error.log                  # Gateway 进程 stderr
```

### 2.2 日志格式（JSONL）

每行一个 JSON 对象，包含完整的元数据：

```json
{
  "0": "消息内容",
  "_meta": {
    "runtime": "node",
    "runtimeVersion": "22.12.0",
    "hostname": "unknown",
    "name": "openclaw",
    "date": "2026-03-16T13:38:03.296Z",
    "logLevelId": 2,
    "logLevelName": "DEBUG",
    "path": {
      "fileName": "subsystem-xxx.js",
      "fileNameWithLine": "subsystem-xxx.js:1020",
      "fileLine": "1020"
    }
  },
  "time": "2026-03-16T21:38:03.296+08:00"
}
```

### 2.3 日志级别

| 级别 | ID | 用途 |
|------|-----|------|
| TRACE | 0 | 最详细，追踪所有 |
| DEBUG | 2 | 调试信息 |
| INFO | 3 | 常规信息 |
| WARN | 4 | 警告 |
| ERROR | 5 | 错误 |
| FATAL | 6 | 致命错误 |

### 2.4 配置日志级别

**在 openclaw.json 中配置**：

```json
{
  "logging": {
    "level": "trace"
  }
}
```

**环境变量**：

```bash
export OPENCLAW_LOG_LEVEL=debug
```

---

## 三、诊断日志（Diagnostics）

### 3.1 调试标志（Flags）

启用特定子系统的详细日志，无需全局提高日志级别。

**在 openclaw.json 中配置**：

```json
{
  "diagnostics": {
    "enabled": true,
    "flags": ["telegram.http", "feishu.*", "gateway.*"]
  }
}
```

**环境变量临时启用**：

```bash
export OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload
```

**支持通配符**：

| 模式 | 匹配范围 |
|------|---------|
| `telegram.http` | 精确匹配 |
| `telegram.*` | 所有 Telegram 相关 |
| `*` | 启用所有标志 |

### 3.2 Cache Trace 诊断

专门用于追踪 **Prompt Cache** 行为，包含完整的请求内容。

**配置**：

```json
{
  "diagnostics": {
    "enabled": true,
    "cacheTrace": {
      "enabled": true,
      "filePath": "~/.openclaw/logs/cache-trace.jsonl",
      "includeMessages": true,
      "includePrompt": true,
      "includeSystem": true
    }
  }
}
```

**环境变量**：

```bash
export OPENCLAW_CACHE_TRACE=1
export OPENCLAW_CACHE_TRACE_FILE=/path/to/file.jsonl
```

**记录阶段**：

| 阶段 | 说明 |
|------|------|
| `session:loaded` | 会话加载时的消息状态 |
| `stream:context` | 发送给模型前的完整 context |
| `session:sanitized` | 清理后的消息 |
| `session:limited` | 限制后的消息 |

### 3.3 Anthropic Payload Log

**只对 Anthropic 模型有效**：

```bash
export OPENCLAW_ANTHROPIC_PAYLOAD_LOG=1
export OPENCLAW_ANTHROPIC_PAYLOAD_LOG_FILE=/path/to/file.jsonl
```

**输出位置**：`~/.openclaw/logs/anthropic-payload.jsonl`

**记录内容**：

| 阶段 | 说明 |
|------|------|
| `stage: "request"` | 发送给模型的完整 payload |
| `stage: "usage"` | Token 使用量 |

---

## 四、诊断事件目录

| 事件 | 说明 | 关键属性 |
|------|------|----------|
| `model.usage` | 模型调用 | tokens, cost, duration, provider, model |
| `webhook.received` | Webhook 入口 | channel, updateType |
| `webhook.processed` | Webhook 处理完成 | channel, durationMs |
| `webhook.error` | Webhook 错误 | channel, error |
| `message.queued` | 消息入队 | channel, source, queueDepth |
| `message.processed` | 消息处理完成 | channel, outcome, durationMs |
| `queue.lane.enqueue` | 命令入队 | lane, queueSize |
| `queue.lane.dequeue` | 命令出队 | lane, queueSize, waitMs |
| `session.state` | 会话状态变更 | state, reason |
| `session.stuck` | 会话卡住警告 | state, ageMs, queueDepth |
| `run.attempt` | 运行重试 | attempt |
| `diagnostic.heartbeat` | 心跳聚合统计 | queued |

---

## 五、OpenTelemetry 导出

### 5.1 配置示例

```json
{
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://otel-collector:4318",
      "protocol": "http/protobuf",
      "serviceName": "openclaw-gateway",
      "traces": true,
      "metrics": true,
      "logs": true,
      "sampleRate": 0.2,
      "flushIntervalMs": 60000
    }
  }
}
```

### 5.2 导出的 Metrics

**模型使用**：

| Metric | 类型 | 说明 |
|--------|------|------|
| `openclaw.tokens` | counter | Token 计数 |
| `openclaw.cost.usd` | counter | 成本 |
| `openclaw.run.duration_ms` | histogram | 运行时长 |
| `openclaw.context.tokens` | histogram | 上下文大小 |

**消息流**：

| Metric | 类型 | 说明 |
|--------|------|------|
| `openclaw.webhook.received` | counter | Webhook 接收数 |
| `openclaw.webhook.error` | counter | Webhook 错误数 |
| `openclaw.webhook.duration_ms` | histogram | Webhook 处理时长 |
| `openclaw.message.queued` | counter | 消息入队数 |
| `openclaw.message.processed` | counter | 消息处理数 |
| `openclaw.message.duration_ms` | histogram | 消息处理时长 |

**队列和会话**：

| Metric | 类型 | 说明 |
|--------|------|------|
| `openclaw.queue.lane.enqueue` | counter | 命令入队数 |
| `openclaw.queue.lane.dequeue` | counter | 命令出队数 |
| `openclaw.queue.depth` | histogram | 队列深度 |
| `openclaw.queue.wait_ms` | histogram | 队列等待时间 |
| `openclaw.session.state` | counter | 会话状态变更 |
| `openclaw.session.stuck` | counter | 会话卡住次数 |

---

## 六、日志查看命令

### 6.1 基本命令

```bash
# 实时查看日志
openclaw logs --follow

# JSON 格式
openclaw logs --json

# 限制行数
openclaw logs --limit 500

# 本地时间显示
openclaw logs --local-time
```

### 6.2 渠道日志

```bash
# 查看特定渠道日志
openclaw channels logs --channel telegram
openclaw channels logs --channel feishu
```

### 6.3 直接查看文件

```bash
# 实时查看
tail -f /tmp/openclaw/openclaw-$(date +%F).log

# 搜索错误
rg "error" /tmp/openclaw/openclaw-*.log

# 查看特定时间段
rg "2026-03-16T21:" /tmp/openclaw/openclaw-2026-03-16.log
```

### 6.4 查看 Cache Trace

```bash
# 查看全部
cat ~/.openclaw/logs/cache-trace.jsonl | jq .

# 查看特定阶段
grep "stream:context" ~/.openclaw/logs/cache-trace.jsonl | jq .

# 统计 Token 使用
grep "usage" ~/.openclaw/logs/cache-trace.jsonl | jq '.usage'
```

---

## 七、日志轮转配置

### 7.1 logrotate 配置

**文件**：`/etc/logrotate.d/openclaw`

```
/tmp/openclaw/openclaw-*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 0644 root root
}
```

**效果**：

- 每天轮转一次
- 保留最近 7 天
- 自动压缩旧日志

### 7.2 Cache Trace 清理

Cache Trace 文件不在 logrotate 管理范围内，需定期清理：

```bash
# 清理 7 天前的文件
find ~/.openclaw/logs -name "cache-trace*.jsonl" -mtime +7 -delete

# 或者设置 cron 任务
(crontab -l 2>/dev/null | grep -v "cache-trace"; \
 echo "0 0 * * * find ~/.openclaw/logs -name 'cache-trace*.jsonl' -mtime +7 -delete") | crontab -
```

---

## 八、systemd 服务配置

**文件**：`/etc/systemd/system/openclaw-gateway.service`

```ini
[Unit]
Description=OpenClaw Gateway Service
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
Environment="PATH=/root/.nvm/versions/node/v22.12.0/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
# 诊断环境变量
Environment="OPENCLAW_ANTHROPIC_PAYLOAD_LOG=1"
Environment="OPENCLAW_CACHE_TRACE=1"
Environment="OPENCLAW_CACHE_TRACE_MESSAGES=1"
ExecStart=/root/.nvm/versions/node/v22.12.0/bin/node /path/to/openclaw.mjs gateway start
Restart=always
RestartSec=5
StandardOutput=append:/tmp/openclaw/gateway.log
StandardError=append:/tmp/openclaw/error.log

[Install]
WantedBy=multi-user.target
```

---

## 九、注意事项

### 9.1 Anthropic Payload Log 的限制

- **只对 Anthropic 模型有效**
- 使用 GLM-5、DeepSeek 等模型不会生成此日志
- 如需记录非 Anthropic 模型的请求，需使用 Cache Trace

### 9.2 Cache Trace 包含的内容

| 内容 | 是否包含 |
|------|---------|
| 完整的 messages | ✅ |
| 完整的 system prompt | ✅ |
| 模型调用参数 | ✅ |
| 响应内容 | ⚠️ 不完整，主要记录 usage |

### 9.3 日志量增长

| 日志类型 | 每次调用增量 | 建议清理周期 |
|---------|-------------|-------------|
| 主日志 | 1-5 KB | 自动轮转 |
| Cache Trace | 10-50 KB | 7 天 |
| Anthropic Payload | 5-20 KB | 7 天 |

---

## 十、最佳实践

### 10.1 生产环境配置

```json
{
  "logging": {
    "level": "info"
  },
  "diagnostics": {
    "enabled": true,
    "cacheTrace": {
      "enabled": false
    }
  }
}
```

### 10.2 调试环境配置

```json
{
  "logging": {
    "level": "debug"
  },
  "diagnostics": {
    "enabled": true,
    "flags": ["*"],
    "cacheTrace": {
      "enabled": true,
      "includeMessages": true,
      "includePrompt": true,
      "includeSystem": true
    }
  }
}
```

### 10.3 性能分析配置

```json
{
  "logging": {
    "level": "info"
  },
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://otel-collector:4318",
      "sampleRate": 0.1
    }
  }
}
```

---

## 十一、日志文件字段详解

本节详细说明 OpenClaw 各日志文件的字段结构和用途。

### 11.1 cache-trace.jsonl

**位置**：`~/.openclaw/logs/cache-trace.jsonl`

**用途**：记录 LLM 调用的缓存信息，用于上下文压缩和缓存命中优化。

**文件格式**：每行一个 JSON 对象（JSONL 格式）

**核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `runId` | string | 本次 LLM 调用的唯一 ID |
| `sessionId` | string | 会话 ID |
| `sessionKey` | string | 会话键（格式：`agent:{agent}:{channel}:{chatType}:{chatId}`） |
| `provider` | string | 模型提供商（如 `tencentcodingplan`） |
| `modelId` | string | 模型 ID（如 `glm-5`） |
| `modelApi` | string | API 类型（如 `openai-completions`） |
| `ts` | string | 时间戳（ISO 8601 格式） |
| `seq` | number | 会话中的调用序号 |
| `stage` | string | 调用阶段（如 `stream:context`） |
| `messageCount` | number | 本次调用包含的消息数量 |
| `messageRoles` | array | 消息角色列表（如 `["user", "assistant"]`） |
| `messageFingerprints` | array | 消息指纹（用于缓存匹配） |
| `messagesDigest` | string | 消息内容的摘要哈希 |
| `messages` | array | 实际消息内容（仅在调试模式） |

**实用查询**：

```bash
# 查看最近的 LLM 调用
tail -10 ~/.openclaw/logs/cache-trace.jsonl | jq .

# 统计某个会话的调用次数
cat ~/.openclaw/logs/cache-trace.jsonl | jq -r '.sessionKey' | sort | uniq -c

# 查看某个模型的调用统计
cat ~/.openclaw/logs/cache-trace.jsonl | jq -r '.modelId' | sort | uniq -c
```

---

### 11.2 sessions.json

**位置**：`~/.openclaw/agents/{agent-name}/sessions/sessions.json`

**用途**：索引所有会话的元数据，快速查找会话信息。

**核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionId` | string | 会话唯一 ID |
| `updatedAt` | number | 最后更新时间（毫秒时间戳） |
| `chatType` | string | 聊天类型（`direct` 私聊，`group` 群聊） |
| `deliveryContext` | object | 投递上下文（渠道、接收者、账号） |
| `lastChannel` | string | 最后使用的渠道 |
| `lastAccountId` | string | 最后使用的账号 ID |
| `origin` | object | 来源信息（渠道、表面、聊天类型等） |
| `sessionFile` | string | 会话消息文件路径 |
| `compactionCount` | number | 上下文压缩次数 |
| `modelProvider` | string | 使用的模型提供商 |
| `model` | string | 使用的模型 ID |
| `contextTokens` | number | 上下文窗口大小 |
| `inputTokens` | number | 累计输入 token 数 |
| `outputTokens` | number | 累计输出 token 数 |
| `totalTokens` | number | 累计总 token 数 |

**实用查询**：

```bash
# 查看所有会话的 token 使用统计
cat ~/.openclaw/agents/*/sessions/sessions.json | jq -r '.[] | "\(.origin.label): \(.totalTokens) tokens"'

# 查找某个会话的文件路径
cat ~/.openclaw/agents/mime-qq/sessions/sessions.json | jq -r '.[] | select(.origin.label | contains("特定标识")) | .sessionFile'

# 查看活跃会话（最近更新的前 10 个）
cat ~/.openclaw/agents/*/sessions/sessions.json | jq -r '.[] | "\(.updatedAt) \(.sessionKey)"' | sort -rn | head -10
```

---

### 11.3 会话消息文件 (*.jsonl)

**位置**：`~/.openclaw/agents/{agent-name}/sessions/{session-id}.jsonl`

**用途**：存储实际对话消息内容，包括用户输入、AI 回复、工具调用等。

**每条消息格式**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 消息类型（`message`） |
| `id` | string | 消息唯一 ID |
| `parentId` | string | 父消息 ID（用于消息树结构） |
| `timestamp` | string | 时间戳（ISO 8601 格式） |
| `message` | object | 消息内容对象 |
| `message.role` | string | 角色（`user`、`assistant`、`toolResult`） |
| `message.content` | array | 内容数组（可包含文本、思考、工具调用等） |
| `message.api` | string | 使用的 API 类型 |
| `message.provider` | string | 使用的提供商 |
| `message.model` | string | 使用的模型 |
| `message.usage` | object | token 使用统计 |
| `message.stopReason` | string | 停止原因（`stop`、`toolUse` 等） |

**内容类型**：
- `text`：普通文本
- `thinking`：模型思考过程（reasoning）
- `toolCall`：工具调用请求
- `toolResult`：工具调用结果

> ⚠️ **注意**：`toolResult` 类型的消息是工具调用结果，不应作为独立消息显示。

**实用查询**：

```bash
# 查看最近 10 条消息
tail -10 /path/to/session.jsonl | jq .

# 统计消息类型分布
cat /path/to/session.jsonl | jq -r '.message.role' | sort | uniq -c

# 提取所有用户消息
cat /path/to/session.jsonl | jq 'select(.message.role == "user") | .message.content[].text'

# 统计消息数量（排除 toolResult）
cat /path/to/session.jsonl | jq -c '. | select(.message.role != "toolResult")' | wc -l
```

---

### 11.4 commands.log

**位置**：`~/.openclaw/logs/commands.log`

**用途**：记录 OpenClaw 命令执行历史。

**格式**：每行一个 JSON 对象

**核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `timestamp` | string | 时间戳 |
| `action` | string | 动作类型（`new` 新会话） |
| `sessionKey` | string | 会话键 |
| `senderId` | string | 发送者 ID |
| `source` | string | 来源渠道 |

---

### 11.5 config-audit.jsonl

**位置**：`~/.openclaw/logs/config-audit.jsonl`

**用途**：审计配置文件变更，用于安全审计和问题排查。

**格式**：每行一个 JSON 对象

**核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `ts` | string | 时间戳 |
| `source` | string | 变更来源（`config-io`） |
| `event` | string | 事件类型（`config.write`） |
| `configPath` | string | 配置文件路径 |
| `pid` | number | 进程 ID |
| `argv` | array | 执行的命令参数 |
| `previousHash` | string | 变更前的文件哈希 |
| `nextHash` | string | 变更后的文件哈希 |
| `result` | string | 操作结果（`rename`） |

**实用查询**：

```bash
# 查看最近的配置变更
tail -20 ~/.openclaw/logs/config-audit.jsonl | jq .

# 查找特定时间段的配置变更
cat ~/.openclaw/logs/config-audit.jsonl | jq 'select(.ts >= "2026-03-01" and .ts < "2026-04-01")'

# 查看谁改了配置
cat ~/.openclaw/logs/config-audit.jsonl | jq -r '.argv | join(" ")'
```

---

## 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [日志配置参考](https://docs.openclaw.ai/logging)
- [Diagnostics 配置](https://docs.openclaw.ai/diagnostics)

---

*本文基于 OpenClaw 2026.3.x 版本编写*
