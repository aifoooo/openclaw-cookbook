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
├── cache-trace.jsonl           # LLM 调用缓存追踪
├── commands.log                # 命令执行记录
└── config-audit.jsonl          # 配置变更审计

~/.openclaw/agents/{agent-name}/sessions/
├── sessions.json               # 会话元数据索引
└── {session-id}.jsonl          # 实际对话消息文件
```

> 📖 **日志文件字段详解**：参见 [OpenClaw日志系统详解.md](./OpenClaw日志系统详解.md)

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
