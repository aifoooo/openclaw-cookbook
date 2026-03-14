# openclaw.json 核心配置入门

OpenClaw 的核心配置文件，控制智能体的行为、内存管理、模型选择等。

> **说明**：本文只覆盖最常用的核心配置。完整配置请参考 [官方配置文档](https://docs.openclaw.ai/gateway/configuration)。

---

## 一、配置文件基础

### 1.1 文件位置

```
~/.openclaw/openclaw.json
```

支持 **JSON5** 格式（允许注释、尾逗号）。

### 1.2 配置验证

OpenClaw 对配置进行**严格验证**：
- 未知键、类型错误、无效值会导致 Gateway **拒绝启动**
- 运行 `openclaw doctor` 查看具体问题
- 运行 `openclaw doctor --fix` 自动修复

### 1.3 配置编辑方式

```bash
# 交互式向导
openclaw onboard
openclaw configure

# CLI 命令
openclaw config get session.reset.mode
openclaw config set session.reset.idleMinutes 60

# 直接编辑（热加载）
# 直接修改 openclaw.json，Gateway 自动监听并应用
```

---

## 二、会话配置（session）

控制会话生命周期和重置策略。

### 2.1 基本配置

```json
{
  "session": {
    "reset": {
      "mode": "idle",
      "idleMinutes": 60
    }
  }
}
```

| 参数 | 说明 | 官方默认 |
|------|------|---------|
| `mode` | 重置模式：`idle`（空闲重置）、`schedule`（定时重置） | `idle` |
| `idleMinutes` | 空闲多少分钟后重置会话 | `60` |

### 2.2 重置模式详解

**idle 模式**：滑动窗口重置
- 会话空闲超过 `idleMinutes` 后，下次消息会创建新会话
- 适合：不定期对话，保持灵活性

**schedule 模式**：定时重置
- 每天固定时间重置（配合 `atHour` 使用）
- 适合：需要定期"清空"的场景

### 2.3 使用场景

| 场景 | 推荐配置 |
|------|---------|
| 日常助手，希望长期记忆 | `idleMinutes: 10080`（7天）或更长 |
| 测试/调试，频繁重置 | `idleMinutes: 60`（默认） |
| 每日固定重置 | `mode: "schedule", atHour: 4` |

---

## 三、模型配置（model）

控制模型选择和降级链。

### 3.1 基本配置

```json
{
  "model": {
    "primary": "anthropic/claude-opus-4-5",
    "fallbacks": [
      "anthropic/claude-sonnet-4-5",
      "openai/gpt-4o"
    ]
  }
}
```

| 参数 | 说明 |
|------|------|
| `primary` | 主模型，格式为 `provider/model` |
| `fallbacks` | 降级链，主模型失败时依次尝试 |

### 3.2 降级链工作原理

```
主模型请求失败
    ↓
尝试 fallbacks[0]
    ↓
失败则尝试 fallbacks[1]
    ↓
全部失败则返回错误
```

### 3.3 常用模型提供商

| 提供商 | 格式 | 环境变量 |
|-------|------|---------|
| Anthropic | `anthropic/claude-opus-4-5` | `ANTHROPIC_API_KEY` |
| OpenAI | `openai/gpt-4o` | `OPENAI_API_KEY` |
| Google | `google/gemini-2.0-flash` | `GEMINI_API_KEY` |
| DeepSeek | `deepseek/deepseek-chat` | `DEEPSEEK_API_KEY` |
| 腾讯云 | `tencentcodingplan/glm-5` | 腾讯云 API Key |

---

## 四、内存管理配置（重要）

### 4.1 核心原则

**如果未写入文件，即不存在。**

智能体的长期记忆完全依赖磁盘文件：
- `MEMORY.md` — 长期记忆
- `AGENTS.md` — 行为规则
- `memory/YYYY-MM-DD.md` — 每日日志

对话历史在压缩后会**丢失细节**，只保留摘要。

---

### 4.2 上下文压缩（compaction）

当对话历史填满上下文窗口时触发。

```json
{
  "compaction": {
    "mode": "safeguard",
    "reserveTokens": 16384,
    "keepRecentTokens": 20000,
    "memoryFlush": {
      "enabled": true,
      "softThresholdTokens": 4000
    }
  }
}
```

#### 关键参数详解

| 参数 | 说明 | 官方默认 |
|------|------|---------|
| `mode` | 压缩模式 | `default` |
| `reserveTokens` | 给提示词+输出预留的空间 | `16384` |
| `keepRecentTokens` | 压缩时保留最近多少 token 不压缩 | `20000` |
| `memoryFlush.enabled` | 是否启用预压缩内存刷新 | `true` |
| `memoryFlush.softThresholdTokens` | 触发内存刷新的阈值 | `4000` |

#### reserveTokens vs keepRecentTokens

**容易混淆，必须区分**：

| 参数 | 用途 | 类比 |
|------|------|------|
| `reserveTokens` | 给**未来**预留空间（提示词 + 模型输出） | 停车场预留车位 |
| `keepRecentTokens` | 压缩时**保留**最近的对话 | 只打包旧衣服，新衣服留着 |

**示例**：
- 模型上下文窗口：128,000 tokens
- `reserveTokens: 16384` → 可用空间 = 128,000 - 16,384 = 111,616
- `keepRecentTokens: 20000` → 压缩时保留最近 20,000 tokens 不压缩

#### reserveTokens vs softThresholdTokens

这两个也容易混淆：

| 参数 | 作用 | 触发什么？ |
|------|------|-----------|
| `reserveTokens` | 给输出预留空间 | 决定压缩**何时发生** |
| `softThresholdTokens` | 提前触发记忆保存 | 决定 memoryFlush **何时触发** |

**时间线**：

```
对话增长中...
    │
    ▼ 达到（压缩阈值 - softThresholdTokens）
    │   → memoryFlush：保存记忆到文件
    │
    ▼ 达到（上下文窗口 - reserveTokens）
    │   → 执行压缩
    │
    ▼ 压缩完成，腾出空间
        → 继续对话
```

**类比**：

| 参数 | 类比 |
|------|------|
| `reserveTokens` | 停车场预留车位（给新车留位置） |
| `softThresholdTokens` | 提前 5 分钟提醒（给你时间收拾东西） |

#### 压缩模式：default vs safeguard

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| `default` | 普通压缩，一次摘要 | 一般对话 |
| `safeguard` | 超长历史时分块摘要，防止信息丢失 | 长期运行、超长对话 |

#### memoryFlush 工作流程

```
上下文接近压缩阈值
    ↓
还剩 softThresholdTokens 时
    ↓
触发预压缩内存刷新（静默轮次）
    ↓
智能体将重要信息写入 memory/YYYY-MM-DD.md
    ↓
执行压缩
    ↓
智能体继续工作
```

**为什么重要**：压缩会丢失对话细节，memoryFlush 给智能体一个"抢救"记忆的机会。

---

### 4.3 上下文修剪（contextPruning）

**与压缩不同**：只在内存中裁剪，不修改历史文件。

```json
{
  "contextPruning": {
    "mode": "adaptive",
    "keepLastAssistants": 3,
    "softTrimRatio": 0.3,
    "hardClearRatio": 0.5
  }
}
```

#### 压缩 vs 修剪

| 特性 | 压缩（compaction） | 修剪（contextPruning） |
|-----|-------------------|----------------------|
| 操作对象 | 整个对话历史 | 仅工具调用结果 |
| 持久化 | 摘要写入 JSONL | 不修改文件 |
| 目的 | 控制上下文窗口大小 | 减少工具输出堆积 |

#### 修剪模式

| 模式 | 说明 |
|------|------|
| `off` | 禁用修剪 |
| `adaptive` | 根据上下文比率动态修剪 |
| `aggressive` | 始终替换旧工具结果为占位符 |

#### 参数说明

| 参数 | 说明 |
|------|------|
| `keepLastAssistants` | 保留最近 N 条助手消息（之后的工具结果可被修剪） |
| `softTrimRatio` | 软修剪阈值（裁剪过大工具结果的中间部分） |
| `hardClearRatio` | 硬清除阈值（直接替换整个工具结果） |

---

## 五、压缩生命周期

### 5.1 好路径（维护性压缩）

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

### 5.2 坏路径（溢出恢复）

```
上下文过大导致 API 拒绝
    ↓
进入损害控制模式
    ↓
强制压缩所有内容
    ↓
无内存刷新，关键信息丢失
```

**目标**：通过配置 memoryFlush，始终走"好路径"。

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
| `hot` | 安全更改立即生效，关键更改忽略（记日志但不生效） |
| `restart` | 任何更改均重启网关 |
| `off` | 关闭热加载 |

---

## 七、完整配置示例

```json
{
  "session": {
    "reset": {
      "mode": "idle",
      "idleMinutes": 60
    }
  },
  "model": {
    "primary": "anthropic/claude-opus-4-5",
    "fallbacks": [
      "anthropic/claude-sonnet-4-5"
    ]
  },
  "compaction": {
    "mode": "safeguard",
    "reserveTokens": 16384,
    "keepRecentTokens": 20000,
    "memoryFlush": {
      "enabled": true,
      "softThresholdTokens": 4000
    }
  },
  "contextPruning": {
    "mode": "adaptive",
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
2. **启用内存刷新**：配置 `memoryFlush.enabled: true`，防止关键信息丢失
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

## 九、更多配置

本文只覆盖核心配置。更多配置请参考：

- [官方配置文档](https://docs.openclaw.ai/gateway/configuration) — 完整配置参考
- [渠道配置](https://docs.openclaw.ai/gateway/configuration#channels) — WhatsApp/Telegram/Discord 等
- [沙箱配置](https://docs.openclaw.ai/gateway/sandboxing) — Docker 隔离
- [模型提供商](https://docs.openclaw.ai/concepts/model-providers) — 自定义 API

---

*本文基于 OpenClaw 官方文档整理。*
