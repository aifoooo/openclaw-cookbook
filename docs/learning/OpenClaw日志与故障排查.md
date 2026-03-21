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
├── gateway.log           # 网关日志
├── agent.log             # 代理日志
├── channel.log           # 渠道日志
├── cache-trace.jsonl     # LLM 调用追踪日志
└── commands.log          # 命令执行记录
```

### 3.4 日志分析技巧

```bash
# 搜索错误
openclaw logs | grep -i error

# 搜索特定渠道
openclaw logs | grep -i telegram

# 统计错误数量
openclaw logs --level error | wc -l
```

---

## 四、日志文件详解

### 4.1 cache-trace.jsonl - LLM 调用追踪

**位置**：`~/.openclaw/logs/cache-trace.jsonl`

**用途**：记录每次 LLM 调用的完整上下文，用于缓存优化和调试。

**每行格式**：JSON 对象

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `runId` | string | 本次调用的唯一标识 |
| `sessionId` | string | 会话 ID |
| `sessionKey` | string | 会话完整标识（格式：`agent:<agent>:<channel>:<chatType>:<chatId>`） |
| `provider` | string | 模型提供商（如 `tencentcodingplan`） |
| `modelId` | string | 模型 ID（如 `glm-5`） |
| `modelApi` | string | API 类型（如 `openai-completions`） |
| `workspaceDir` | string | 工作目录 |
| `ts` | string | ISO 时间戳 |
| `seq` | number | 序列号（同一会话中的调用次数） |
| `stage` | string | 调用阶段（如 `stream:context`） |
| `messageCount` | number | 发送给模型的消息数量 |
| `messageRoles` | array | 消息角色列表（如 `["user", "assistant", "toolResult"]`） |
| `messageFingerprints` | array | 每条消息的指纹（用于缓存） |
| `messagesDigest` | string | 消息整体的摘要（用于缓存） |
| `messages` | array | 完整消息内容（仅当 `stage: "stream:context"` 时） |
| `model` | object | 模型详细配置 |
| `options` | object | 调用选项（含 API Key、模型配置等） |

#### 示例记录

```json
{
  "runId": "11971fec-03c7-4fc6-9ad0-971bae078c76",
  "sessionId": "7add3c20-e176-460b-9eac-a1a7dc1d808b",
  "sessionKey": "agent:mime-qq:qqbot:direct:e82d1c349eecb745d2a4fa8085552bba",
  "provider": "tencentcodingplan",
  "modelId": "glm-5",
  "modelApi": "openai-completions",
  "workspaceDir": "/root/ws-mime-qq",
  "ts": "2026-03-21T01:36:04.832Z",
  "seq": 12,
  "stage": "stream:context",
  "messageCount": 3,
  "messageRoles": ["user", "assistant", "toolResult"],
  "messagesDigest": "6ce2ad78469348382b36737c9f07e58b...",
  "messageFingerprints": ["c225aff8...", "d4d9cdc8...", "c855ce55..."]
}
```

#### 实用查询

```bash
# 查看最近的 LLM 调用
tail -5 ~/.openclaw/logs/cache-trace.jsonl | jq .

# 统计每个模型的调用次数
cat ~/.openclaw/logs/cache-trace.jsonl | jq -r '.modelId' | sort | uniq -c

# 查看特定会话的调用
grep "sessionId\":\"<session-id>" ~/.openclaw/logs/cache-trace.jsonl | jq .

# 计算平均消息长度
cat ~/.openclaw/logs/cache-trace.jsonl | jq '.messageCount' | awk '{sum+=$1; count++} END {print "avg:", sum/count}'
```

---

### 4.2 sessions.json - 会话元数据

**位置**：`~/.openclaw/agents/<agent>/sessions/sessions.json`

**用途**：存储所有会话的元数据和状态。

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionId` | string | 会话唯一 ID |
| `sessionKey` | string | 完整会话标识（作为 JSON key） |
| `updatedAt` | number | 最后更新时间戳（毫秒） |
| `systemSent` | boolean | 系统消息是否已发送 |
| `abortedLastRun` | boolean | 上次运行是否被中止 |
| `chatType` | string | 聊天类型（`direct` 私聊，`group` 群聊） |
| `deliveryContext` | object | 消息投递上下文 |
| `lastChannel` | string | 最后使用的渠道（如 `qqbot`） |
| `lastTo` | string | 最后发送目标 |
| `lastAccountId` | string | 最后使用的账号 ID |
| `origin` | object | 会话来源信息 |
| `sessionFile` | string | 会话消息文件路径 |
| `compactionCount` | number | 压缩次数 |
| `skillsSnapshot` | object | 技能快照 |
| `modelProvider` | string | 当前使用的模型提供商 |
| `model` | string | 当前使用的模型 |
| `contextTokens` | number | 模型上下文窗口大小 |
| `inputTokens` | number | 输入 Token 数 |
| `outputTokens` | number | 输出 Token 数 |
| `totalTokens` | number | 总 Token 数 |

#### 示例结构

```json
{
  "agent:mime-qq:qqbot:direct:e82d1c349eecb745d2a4fa8085552bba": {
    "sessionId": "7add3c20-e176-460b-9eac-a1a7dc1d808b",
    "updatedAt": 1774056862596,
    "chatType": "direct",
    "deliveryContext": {
      "channel": "qqbot",
      "to": "qqbot:c2c:E82D1C349EECB745D2A4FA8085552BBA",
      "accountId": "mime"
    },
    "lastChannel": "qqbot",
    "sessionFile": "/root/.openclaw/agents/mime-qq/sessions/7add3c20-e176-460b-9eac-a1a7dc1d808b.jsonl",
    "compactionCount": 2,
    "modelProvider": "tencentcodingplan",
    "model": "glm-5",
    "inputTokens": 73422,
    "outputTokens": 123,
    "totalTokens": 73422
  }
}
```

#### 实用查询

```bash
# 查看所有会话的 Token 使用情况
cat ~/.openclaw/agents/*/sessions/sessions.json | jq '.[].totalTokens' | awk '{sum+=$1} END {print "Total tokens:", sum}'

# 查看活跃会话（最近更新的前 10 个）
cat ~/.openclaw/agents/*/sessions/sessions.json | jq -r '.[] | "\(.updatedAt) \(.sessionKey)"' | sort -rn | head -10

# 统计会话数量
cat ~/.openclaw/agents/*/sessions/sessions.json | jq 'keys | length'
```

---

### 4.3 Session 文件 - 消息记录

**位置**：`~/.openclaw/agents/<agent>/sessions/<session-id>.jsonl`  
**位置**：`~/.openclaw/sessions/<date>.jsonl`（全局会话文件）

**用途**：存储会话中的所有消息，每行一个 JSON 对象。

#### 消息字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 消息唯一 ID |
| `timestamp` | number | 消息时间戳（毫秒） |
| `role` | string | 角色：`user`、`assistant`、`toolResult` |
| `content` | string/array | 消息内容（文本或内容块数组） |

#### 消息类型说明

| 类型 | 说明 |
|------|------|
| `user` | 用户消息 |
| `assistant` | 助手回复 |
| `toolResult` | 工具调用结果（不应作为独立消息显示） |

#### 内容块格式

```json
{
  "type": "text",
  "text": "消息文本"
}
```

```json
{
  "type": "toolCall",
  "id": "call_xxx",
  "name": "read",
  "arguments": {...}
}
```

```json
{
  "type": "thinking",
  "thinking": "思考内容"
}
```

#### 示例消息

**用户消息**：
```json
{
  "id": "1773996114409-zkc8dbyu4",
  "timestamp": 1773996114409,
  "role": "user",
  "content": "继续"
}
```

**助手消息**：
```json
{
  "timestamp": 1773996116329,
  "role": "assistant",
  "content": [
    {
      "type": "thinking",
      "thinking": "用户说继续..."
    },
    {
      "type": "text",
      "text": "🦐 好的，继续..."
    },
    {
      "type": "toolCall",
      "id": "callf81175e54b134fc6919b303c",
      "name": "read",
      "arguments": {"path": "/root/..."}
    }
  ]
}
```

#### 实用查询

```bash
# 统计消息数量（排除 toolResult）
cat ~/.openclaw/sessions/2026-03-20.jsonl | jq -c '. | select(.role != "toolResult")' | wc -l

# 查看最新的 10 条消息
tail -10 ~/.openclaw/sessions/$(date +%Y-%m-%d).jsonl | jq .

# 提取所有用户消息
cat ~/.openclaw/sessions/*.jsonl | jq -c 'select(.role == "user")' > user_messages.jsonl

# 搜索包含关键词的消息
cat ~/.openclaw/sessions/*.jsonl | jq -c 'select(.content | contains("错误"))'
```

---

### 4.4 日志轮转与清理

#### 自动清理

OpenClaw 使用 systemd-tmpfiles 自动清理旧日志：

```bash
# 查看清理配置
cat /etc/tmpfiles.d/openclaw.conf

# 手动触发清理
systemd-tmpfiles --clean
```

#### cache-trace.jsonl 清理

```bash
# 查看文件大小
du -h ~/.openclaw/logs/cache-trace.jsonl

# 保留最近 N 行（如 10000 行）
tail -10000 ~/.openclaw/logs/cache-trace.jsonl > ~/.openclaw/logs/cache-trace.jsonl.tmp
mv ~/.openclaw/logs/cache-trace.jsonl.tmp ~/.openclaw/logs/cache-trace.jsonl
```

#### session 文件清理

```bash
# 查看各日期文件大小
ls -lh ~/.openclaw/sessions/

# 删除 30 天前的文件
find ~/.openclaw/sessions/ -name "*.jsonl" -mtime +30 -delete
```

---

## 五、常见故障排查

### 5.1 网关连接失败：Pairing Required

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

### 5.2 网关超时

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

### 5.3 模型调用失败

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

### 5.4 渠道连接失败

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

## 六、高级调试

### 6.1 深度诊断

```bash
# 完整诊断报告
openclaw status --all

# 深度检查
openclaw status --deep
```

### 6.2 安全审计

```bash
# 基本审计
openclaw security audit

# 深度审计
openclaw security audit deep --fix
```

### 6.3 会话诊断

```bash
# 查看活跃会话
openclaw sessions list

# 查看会话详情
openclaw sessions show <session-id>
```

---

## 七、故障排查流程图

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

## 八、预防性维护

### 8.1 定期检查

```bash
# 每周运行
openclaw doctor

# 每月运行
openclaw security audit deep
```

### 8.2 日志监控

```bash
# 设置日志告警
openclaw logs --follow | grep -i error | mail -s "OpenClaw Error" admin@example.com
```

### 8.3 配置备份

```bash
# 备份配置
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak

# 版本控制
cd ~/.openclaw && git add . && git commit -m "backup"
```

---

## 九、获取帮助

### 9.1 生成诊断报告

```bash
# 生成完整报告
openclaw status --all > diagnostic-report.txt

# 分享给社区
# 将报告发送到 Discord 或 GitHub Issue
```

### 9.2 社区资源

| 资源 | 链接 |
|-----|------|
| Discord | https://discord.com/invite/clawd |
| GitHub Issues | https://github.com/openclaw/openclaw/issues |
| 官方文档 | https://docs.openclaw.ai/help/troubleshooting |

---

## 十、常见错误速查表

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
