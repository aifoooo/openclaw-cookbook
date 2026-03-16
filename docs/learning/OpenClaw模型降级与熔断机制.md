# OpenClaw 模型降级与熔断机制

> 当主模型失败时，OpenClaw 如何自动切换到备用模型？熔断机制如何防止雪崩效应？本文深入解析。

---

## 背景

在生产环境中，LLM 服务可能遇到：
- **认证失败**：Token 过期或无效
- **账单问题**：余额不足
- **服务过载**：上游服务返回 529 错误
- **超时**：网络或服务响应慢
- **速率限制**：API 调用超过限额

OpenClaw 的模型降级机制可以自动切换到备用模型，熔断机制可以防止故障扩散。

---

## 一、降级架构

OpenClaw 采用**两阶段故障转移机制**：

```
阶段一：Auth Profile 轮换（同一 Provider 内）
    ↓ 所有 Profile 失败
阶段二：Model Fallback（跨 Provider）
```

### 1.1 阶段一：Auth Profile 轮换

当 Provider 有多个认证配置（API Key 或 OAuth）时：
1. 按优先级尝试每个 Profile
2. 失败的 Profile 进入冷却期
3. 所有 Profile 失败后，进入阶段二

### 1.2 阶段二：Model Fallback

跨 Provider 切换到备用模型：
1. 尝试配置的 fallbacks 列表中的模型
2. 直到成功或全部失败

---

## 二、配置方式

### 2.1 多 Auth Profile 配置（阶段一）

当同一个 Provider 有多个账号（API Key 或 OAuth）时，可以配置轮换顺序。

**配置位置**：`~/.openclaw/openclaw.json`

```json
{
  "auth": {
    "order": {
      "tencentcodingplan": [
        "tencentcodingplan:account1@qq.com",
        "tencentcodingplan:account2@qq.com"
      ]
    }
  }
}
```

**认证信息存储位置**：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`

### 2.2 Model Fallback 配置（阶段二）

在 `~/.openclaw/openclaw.json` 中配置主模型和降级链：

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "tencentcodingplan/glm-5",
        "fallbacks": [
          "deepseek/deepseek-chat",
          "openai/gpt-4o-mini"
        ]
      }
    }
  }
}
```

### 2.3 配置方式对比

| 场景 | 配置方式 |
|------|---------|
| **单账号** | 直接在 `models.providers.*.apiKey` 配置 |
| **多账号** | 需要 `auth.order` + `auth-profiles.json` |

**注意**：如果只有一个 API Key，不需要配置 `auth.order`，OpenClaw 会自动使用 `models.providers.*.apiKey`。

---

## 三、熔断机制（Cooldown）

### 3.1 冷却时间

当 Profile 失败时，进入冷却期，使用**指数退避策略**：

| 失败次数 | 冷却时间 |
|---------|---------|
| 第 1 次 | 1 分钟 |
| 第 2 次 | 5 分钟 |
| 第 3 次 | 25 分钟 |
| 第 4 次+ | 1 小时（上限） |

### 3.2 失败类型分类

OpenClaw 将失败分为不同类型，优先级从高到低：

| 失败类型 | 说明 | 严重程度 |
|---------|------|---------|
| `auth_permanent` | 永久认证失败 | 最高 |
| `auth` | 认证失败 | 高 |
| `billing` | 账单问题 | 高 |
| `format` | 格式错误 | 中 |
| `model_not_found` | 模型未找到 | 中 |
| `overloaded` | 过载 | 中 |
| `timeout` | 超时 | 低 |
| `rate_limit` | 速率限制 | 低 |
| `unknown` | 未知错误 | 最低 |

### 3.3 账单特殊处理

账单失败（`billing`）使用更长的退避时间：

| 失败次数 | 退避时间 |
|---------|---------|
| 第 1 次 | 5 小时 |
| 第 2 次 | 10 小时 |
| 第 3 次 | 20 小时 |
| 第 4 次+ | 24 小时（上限） |

### 3.4 冷却状态存储

冷却状态存储在 `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`：

```json
{
  "usageStats": {
    "tencentcodingplan:account1@qq.com": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2,
      "failureCounts": {
        "rate_limit": 1,
        "timeout": 1
      }
    }
  }
}
```

---

## 四、降级策略

### 4.1 候选模型列表

OpenClaw 按以下顺序构建候选模型列表：

1. **当前请求的模型**（如果有覆盖）
2. **配置的 fallbacks 列表**
3. **配置的主模型**

### 4.2 降级流程

```
尝试候选模型 1
    ↓ 失败
尝试候选模型 2
    ↓ 失败
尝试候选模型 3
    ↓ 全部失败
抛出错误
```

**关键点**：
- ✅ **所有候选模型都有效**
- ✅ **按顺序尝试**
- ✅ **直到成功或全部失败**

### 4.3 冷却期内的跳过逻辑

在冷却期内，会跳过某些候选：

| 失败类型 | 行为 |
|---------|------|
| `auth_permanent` | 跳过该 Provider 的所有模型 |
| `auth` | 跳过该 Provider 的所有模型 |
| `billing` | 单 Provider 时探测，多 Provider 时跳过 |
| `rate_limit` | 可能尝试（探测机制） |
| `overloaded` | 可能尝试（探测机制） |

---

## 五、探测机制

### 5.1 探测频率限制

为避免频繁探测失败的服务：

```
最小探测间隔：30 秒
探测提前量：2 分钟
```

### 5.2 探测逻辑

- 每个 Provider 每 30 秒最多探测一次
- 主模型且有 fallback 时，在冷却期快结束时探测
- 探测失败会标记 Provider，避免重复探测

---

## 六、会话粘性

### 6.1 Profile 固定

OpenClaw 为每个会话固定选择的 Auth Profile，以保持缓存热度：

```
会话开始 → 选择 Profile → 固定使用
    ↓ 以下情况会重新选择
会话重置（/new /reset）
压缩完成
Profile 进入冷却期
```

### 6.2 用户覆盖

通过 `/model ...@<profileId>` 手动选择 Profile：

- 锁定到该 Profile
- 失败时不轮换 Profile，直接进入 Model Fallback
- 新会话开始前保持锁定

---

## 七、实际示例

### 示例 1：主模型失败，降级到第一个 fallback

```
请求：tencentcodingplan/glm-5
尝试 1：tencentcodingplan/glm-5 → 失败（rate_limit）
       ↓ 进入冷却期，尝试下一个
尝试 2：deepseek/deepseek-chat → 成功 ✓
```

### 示例 2：所有候选都失败

```
请求：tencentcodingplan/glm-5
尝试 1：tencentcodingplan/glm-5 → 失败（auth）
尝试 2：deepseek/deepseek-chat → 失败（billing）
尝试 3：openai/gpt-4o-mini → 失败（timeout）
结果：抛出错误 "All models failed (3): ..."
```

### 示例 3：冷却期内跳过

```
请求：tencentcodingplan/glm-5
冷却状态：tencentcodingplan 所有 Profile 在冷却期
尝试 1：tencentcodingplan/glm-5 → 跳过（冷却期）
尝试 2：deepseek/deepseek-chat → 成功 ✓
```

---

## 八、监控与调试

### 8.1 日志关键词

```
model fallback decision      # 降级决策
probe_cooldown_candidate     # 冷却期探测
candidate_failed             # 候选失败
candidate_succeeded          # 候选成功
```

### 8.2 状态文件

| 文件 | 内容 |
|------|------|
| `auth-profiles.json` | Profile 状态、冷却时间、失败计数 |
| `openclaw-*.log` | 降级决策日志 |

---

## 九、最佳实践

### 9.1 配置建议

1. **至少配置 2 个 Provider**：跨 Provider 降级更可靠
2. **Fallbacks 顺序**：成本低的模型优先
3. **多 Auth Profile**：提高可用性

### 9.2 监控建议

1. 监控 `usageStats` 中的冷却状态
2. 关注降级决策日志
3. 定期检查 Auth Profile 健康状态

---

## 十、总结

| 特性 | 说明 |
|------|------|
| **两阶段降级** | Auth Profile 轮换 → Model Fallback |
| **熔断机制** | ✅ 有，指数退避冷却 |
| **候选列表** | ✅ 全部有效，顺序尝试 |
| **探测机制** | ✅ 有，30 秒最小间隔 |
| **会话粘性** | ✅ 有，固定 Auth Profile |
| **账单特殊处理** | ✅ 有，5-24 小时退避 |

---

## 参考资料

- [OpenClaw 源代码](https://github.com/openclaw/openclaw)
- [Model Failover 文档](https://docs.openclaw.ai/concepts/model-failover)

---

*本文基于 OpenClaw 2026.3.x 版本编写*
