# OpenClaw 日志与故障排查

OpenClaw 提供完善的诊断工具链，帮助快速定位问题。

---

## 一、诊断工具链

### 1.1 "前60秒"快速排查

遇到问题时，按顺序执行：

```bash
# 1. 快速健全性检查
openclaw status

# 2. 详细诊断报告
openclaw status --all

# 3. 网关可达性检查
openclaw gateway probe

# 4. 网关运行状态
openclaw gateway status

# 5. 系统诊断
openclaw doctor

# 6. 渠道状态检查
openclaw channels status --probe

# 7. 实时日志
openclaw logs --follow
```

### 1.2 命令详解

| 命令 | 功能 | 预期输出 |
|-----|------|---------|
| `openclaw status` | 快速检查 | 显示已配置渠道 |
| `openclaw gateway probe` | 网关可达性 | `Reachable: yes` |
| `openclaw gateway status` | 运行状态 | `Runtime: running` |
| `openclaw doctor` | 系统诊断 | 无阻塞错误 |

---

## 二、openclaw doctor

### 2.1 功能

检查以下内容：
- Node.js 版本（需 v22+）
- 配置文件完整性
- 模型认证状态
- 网关状态
- 安全配置

### 2.2 使用方式

```bash
# 基本诊断
openclaw doctor

# 自动修复
openclaw doctor --fix

# 强制修复（无需确认）
openclaw doctor --yes
```

### 2.3 常见问题修复

| 问题 | 原因 | 修复方式 |
|-----|------|---------|
| Node.js 版本过低 | 版本 < 22 | 升级 Node.js |
| 配置文件错误 | JSON 语法错误 | `--fix` 自动修复 |
| 权限问题 | 文件所有权错误 | `--fix` 自动修复 |
| API Key 无效 | 密钥过期/错误 | 更新配置 |

---

## 三、日志系统

### 3.1 查看日志

```bash
# 实时日志
openclaw logs --follow

# 最近 N 行
openclaw logs --tail 100

# 按级别过滤
openclaw logs --level error
openclaw logs --level warn

# JSON 格式输出
openclaw logs --json
```

### 3.2 日志级别

| 级别 | 说明 |
|-----|------|
| `error` | 错误，需要关注 |
| `warn` | 警告，可能有问题 |
| `info` | 信息，正常运行 |
| `debug` | 调试信息 |
| `trace` | 详细追踪 |

### 3.3 日志位置

```
~/.openclaw/logs/
├── cache-trace.jsonl           # LLM 调用缓存追踪（重要）
├── commands.log                # 命令执行记录
└── config-audit.jsonl          # 配置变更审计

~/.openclaw/agents/{agent-name}/sessions/
├── sessions.json               # 会话元数据索引
└── {session-id}.jsonl          # 实际对话消息文件
```

### 3.4 日志文件详解

#### 3.4.1 cache-trace.jsonl

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

**实用场景**：
- 排查 LLM 调用次数异常
- 分析缓存命中率
- 追踪特定会话的调用历史

```bash
# 查看最近的 LLM 调用
tail -10 ~/.openclaw/logs/cache-trace.jsonl | jq .

# 统计某个会话的调用次数
cat ~/.openclaw/logs/cache-trace.jsonl | jq -r '.sessionKey' | sort | uniq -c

# 查看某个模型的调用统计
cat ~/.openclaw/logs/cache-trace.jsonl | jq -r '.modelId' | sort | uniq -c
```

#### 3.4.2 sessions.json

**用途**：索引所有会话的元数据，快速查找会话信息。

**位置**：`~/.openclaw/agents/{agent-name}/sessions/sessions.json`

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

**实用场景**：
- 查找特定用户的会话
- 统计 token 使用量
- 追踪会话活跃度

```bash
# 查看所有会话的 token 使用统计
cat ~/.openclaw/agents/*/sessions/sessions.json | jq -r '.[] | "\(.origin.label): \(.totalTokens) tokens"'

# 查找某个会话的文件路径
cat ~/.openclaw/agents/mime-qq/sessions/sessions.json | jq -r '.[] | select(.origin.label | contains("特定标识")) | .sessionFile'
```

#### 3.4.3 会话消息文件 (*.jsonl)

**用途**：存储实际对话消息内容，包括用户输入、AI 回复、工具调用等。

**位置**：`~/.openclaw/agents/{agent-name}/sessions/{session-id}.jsonl`

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

**实用场景**：
- 导出对话历史
- 分析对话模式
- 恢复误删的对话

```bash
# 查看最近 10 条消息
tail -10 /path/to/session.jsonl | jq .

# 统计消息类型分布
cat /path/to/session.jsonl | jq -r '.message.role' | sort | uniq -c

# 提取所有用户消息
cat /path/to/session.jsonl | jq 'select(.message.role == "user") | .message.content[].text'
```

#### 3.4.4 commands.log

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

#### 3.4.5 config-audit.jsonl

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

**实用场景**：
- 追踪谁在什么时候改了配置
- 排查配置错误的时间点
- 安全审计

```bash
# 查看最近的配置变更
tail -20 ~/.openclaw/logs/config-audit.jsonl | jq .

# 查找特定时间段的配置变更
cat ~/.openclaw/logs/config-audit.jsonl | jq 'select(.ts >= "2026-03-01" and .ts < "2026-04-01")'
```

### 3.5 日志分析技巧

```bash
# 搜索错误
openclaw logs | grep -i error

# 搜索特定渠道
openclaw logs | grep -i telegram

# 统计错误数量
openclaw logs --level error | wc -l
```

---

## 四、常见故障排查

### 4.1 网关连接失败：Pairing Required

**症状**：
```
Error: Gateway connect failed: Pairing required
```

**原因**：
- 新安装，设备未授权
- 更新后设备身份过期
- 缓存问题

**修复步骤**：

```bash
# 1. 列出待授权设备
openclaw devices list

# 2. 授权设备
openclaw devices approve <REQUEST_ID>

# 3. 验证连接
openclaw gateway status
```

### 4.2 网关超时

**症状**：
```
Error: Gateway timeout
```

**原因**：
- Gateway 未启动
- 端口被占用
- 防火墙阻止

**修复步骤**：

```bash
# 1. 检查 Gateway 状态
openclaw gateway status

# 2. 检查端口
netstat -tlnp | grep 18789

# 3. 检查防火墙
openclaw doctor

# 4. 重启 Gateway
openclaw gateway restart
```

### 4.3 模型调用失败

**症状**：
```
Error: Model request failed
```

**原因**：
- API Key 无效
- 模型不可用
- 网络问题

**修复步骤**：

```bash
# 1. 检查模型配置
openclaw config get model

# 2. 检查 API Key
openclaw doctor

# 3. 测试模型连接
openclaw models test
```

### 4.4 渠道连接失败

**症状**：
```
Channel telegram: disconnected
```

**原因**：
- Token 错误
- 网络问题
- 渠道服务问题

**修复步骤**：

```bash
# 1. 检查渠道状态
openclaw channels status --probe

# 2. 重新登录
openclaw channels logout telegram
openclaw channels login telegram

# 3. 查看渠道日志
openclaw channels logs --channel telegram
```

---

## 五、高级调试

### 5.1 深度诊断

```bash
# 完整诊断报告
openclaw status --all

# 深度检查
openclaw status --deep
```

### 5.2 安全审计

```bash
# 基本审计
openclaw security audit

# 深度审计
openclaw security audit deep --fix
```

### 5.3 会话诊断

```bash
# 查看活跃会话
openclaw sessions list

# 查看会话详情
openclaw sessions show <session-id>
```

---

## 六、故障排查流程图

```
问题发生
    ↓
openclaw status
    ↓
Gateway 正常？ ─否→ openclaw gateway start
    ↓是
openclaw doctor
    ↓
有错误？ ─是→ openclaw doctor --fix
    ↓否
openclaw logs --follow
    ↓
找到错误？ ─否→ openclaw status --deep
    ↓是
针对性修复
```

---

## 七、预防性维护

### 7.1 定期检查

```bash
# 每周运行
openclaw doctor

# 每月运行
openclaw security audit deep
```

### 7.2 日志监控

```bash
# 设置日志告警
openclaw logs --follow | grep -i error | mail -s "OpenClaw Error" admin@example.com
```

### 7.3 配置备份

```bash
# 备份配置
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak

# 版本控制
cd ~/.openclaw && git add . && git commit -m "backup"
```

---

## 八、获取帮助

### 8.1 生成诊断报告

```bash
# 生成完整报告
openclaw status --all > diagnostic-report.txt

# 分享给社区
# 将报告发送到 Discord 或 GitHub Issue
```

### 8.2 社区资源

| 资源 | 链接 |
|-----|------|
| Discord | https://discord.com/invite/clawd |
| GitHub Issues | https://github.com/openclaw/openclaw/issues |
| 官方文档 | https://docs.openclaw.ai/help/troubleshooting |

---

## 九、常见错误速查表

| 错误 | 原因 | 解决方案 |
|-----|------|---------|
| `Pairing required` | 设备未授权 | `openclaw devices approve` |
| `Gateway timeout` | Gateway 未启动 | `openclaw gateway start` |
| `Config validation failed` | 配置错误 | `openclaw doctor --fix` |
| `Model auth failed` | API Key 错误 | 检查模型配置 |
| `Channel disconnected` | 渠道连接断开 | 重新登录渠道 |
| `Context limit exceeded` | 上下文超限 | 调整 compaction 配置 |

---

*本文基于 OpenClaw 官方文档和社区经验整理。*
