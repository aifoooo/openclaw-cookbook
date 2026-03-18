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

## 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [日志配置参考](https://docs.openclaw.ai/logging)
- [Diagnostics 配置](https://docs.openclaw.ai/diagnostics)

---

*本文基于 OpenClaw 2026.3.x 版本编写*
