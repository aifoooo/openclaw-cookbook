# OpenClaw 渠道配置

OpenClaw 支持多平台消息对接，包括 Telegram、WhatsApp、Discord、Slack、飞书、QQ 等。

---

## 一、渠道配置基础

### 1.1 配置位置

渠道配置在 `openclaw.json` 的 `channels` 字段：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "你的Bot Token"
    },
    "whatsapp": {
      "enabled": true,
      "allowFrom": ["+15555550123"]
    }
  }
}
```

### 1.2 前置条件

**必须使用 Node.js 22+ LTS**

| 渠道 | 依赖 |
|-----|------|
| WhatsApp | Node 22 的 `AsyncLocalStorage` |
| Telegram | Node 22 的 `WebCrypto` API |
| Discord | Node 22 的 WebSocket 稳定性 |

低版本会导致难以诊断的静默错误。

---

## 二、Telegram 配置

### 2.1 创建 Bot

1. 打开 Telegram，搜索 `@BotFather`
2. 发送 `/newbot`，按提示创建 Bot
3. 获取 Bot Token（格式：`123456789:ABCdefGHIjklMNOpqrsTUVwxyz`）

### 2.2 配置

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "123456789:ABCdefGHIjklMNOpqrsTUVwxyz"
    }
  }
}
```

### 2.3 CLI 命令

```bash
# 登录 Telegram
openclaw channels login telegram

# 查看状态
openclaw channels status telegram

# 查看日志
openclaw channels logs --channel telegram
```

### 2.4 常见问题

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| Bot 无响应 | Token 错误 | 检查 Token 是否正确 |
| 无法接收消息 | 隐私模式 | 群聊需关闭隐私模式或 @Bot |
| 连接断开 | 网络问题 | 检查代理设置 |

---

## 三、WhatsApp 配置

### 3.1 配置方式

WhatsApp 使用扫码登录，无需 Bot Token：

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "allowFrom": ["+15555550123"]
    }
  }
}
```

### 3.2 登录流程

```bash
# 登录 WhatsApp
openclaw channels login whatsapp

# 会显示二维码，用手机 WhatsApp 扫码
```

### 3.3 allowFrom 白名单

限制哪些号码可以与 Bot 交互：

```json
{
  "allowFrom": [
    "+15555550123",
    "+8613800138000"
  ]
}
```

---

## 四、Discord 配置

### 4.1 创建 Bot

1. 访问 [Discord Developer Portal](https://discord.com/developers/applications)
2. 创建 Application → Bot → Add Bot
3. 获取 Token
4. 配置 Intents（必须开启 Message Content Intent）

### 4.2 配置

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "botToken": "你的Discord Bot Token",
      "applicationId": "你的Application ID"
    }
  }
}
```

### 4.3 邀请 Bot

生成邀请链接：

```
https://discord.com/api/oauth2/authorize?
  client_id=你的ApplicationID
  &permissions=2048
  &scope=bot
```

### 4.4 Intents 配置

Discord Bot 需要正确的 Intents：

| Intent | 用途 |
|--------|------|
| Message Content | 读取消息内容 |
| Server Members | 获取成员列表 |
| Presence | 获取在线状态 |

```bash
# 检查 Intents 配置
openclaw channels capabilities discord
```

---

## 五、Slack 配置

### 5.1 创建 App

1. 访问 [Slack API](https://api.slack.com/apps)
2. Create New App → From scratch
3. 获取 Bot User OAuth Token

### 5.2 配置

```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "botToken": "xoxb-你的Bot Token",
      "appToken": "xapp-你的App Token"
    }
  }
}
```

### 5.3 OAuth Scopes

需要的 Scopes：

| Scope | 用途 |
|-------|------|
| `chat:write` | 发送消息 |
| `channels:history` | 读取频道历史 |
| `im:history` | 读取私信历史 |

---

## 六、CLI 配置命令

### 6.1 渠道管理

```bash
# 添加渠道账户
openclaw channels add telegram --account main

# 移除渠道
openclaw channels remove telegram --account main

# 列出所有渠道
openclaw channels list

# 查看渠道状态
openclaw channels status
```

### 6.2 登录/登出

```bash
# 登录渠道
openclaw channels login telegram
openclaw channels login whatsapp
openclaw channels login discord

# 登出渠道
openclaw channels logout telegram
```

### 6.3 故障排查

```bash
# 查看渠道日志
openclaw channels logs --channel telegram

# 探测渠道能力
openclaw channels capabilities discord

# 解析用户/频道 ID
openclaw channels resolve telegram @username
```

---

## 七、多渠道同时运行

### 7.1 配置示例

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "telegram-token"
    },
    "discord": {
      "enabled": true,
      "botToken": "discord-token",
      "applicationId": "app-id"
    },
    "whatsapp": {
      "enabled": true,
      "allowFrom": ["+15555550123"]
    }
  }
}
```

### 7.2 资源竞争问题

多渠道同时运行可能导致资源竞争：

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| 消息延迟 | 事件循环阻塞 | 使用 Node 22+ |
| 连接断开 | WebSocket 竞争 | 调整心跳间隔 |
| 内存增长 | 连接泄漏 | 定期重启 Gateway |

---

## 八、安全建议

### 8.1 白名单机制

所有渠道都支持 `allowFrom` 白名单：

```json
{
  "channels": {
    "telegram": {
      "allowFrom": ["@username", "123456789"]
    }
  }
}
```

### 8.2 Token 安全

- 不要将 Token 提交到 Git
- 使用环境变量注入：

```json
{
  "channels": {
    "telegram": {
      "botToken": "${TELEGRAM_BOT_TOKEN}"
    }
  }
}
```

```bash
export TELEGRAM_BOT_TOKEN="your-token"
```

---

## 九、常见错误

| 错误 | 原因 | 解决方案 |
|-----|------|---------|
| `401 Unauthorized` | Token 无效 | 检查 Token 是否正确 |
| `403 Forbidden` | 权限不足 | 检查 Intents/Scopes |
| `429 Too Many Requests` | 请求过快 | 降低消息频率 |
| `Connection timeout` | 网络问题 | 检查代理/防火墙 |

---

## 十、调试技巧

```bash
# 实时日志
openclaw logs --follow

# 深度诊断
openclaw doctor

# 渠道连通性测试
openclaw channels status --probe
```

---

*本文基于 OpenClaw 官方文档和社区经验整理。*
