# OpenClaw 工具权限配置

OpenClaw 的工具权限系统控制智能体能执行哪些操作。正确配置权限是安全使用的关键。

---

## 一、工具权限体系

### 1.1 核心概念

| 概念 | 说明 |
|-----|------|
| **Tool** | 单个能力（如 read、write、exec） |
| **allow** | 允许使用的工具列表 |
| **deny** | 禁止使用的工具列表 |
| **requireConfirmation** | 需要确认才能执行的工具 |

### 1.2 权限优先级

```
deny > allow > 默认行为
```

---

## 二、tools.allow 配置

### 2.1 基本配置

```json
{
  "tools": {
    "allow": {
      "read": true,
      "write": true,
      "edit": true,
      "exec": ["npm", "git", "ls", "cat"],
      "web_search": true,
      "web_fetch": true
    }
  }
}
```

### 2.2 exec 白名单

`exec` 工具支持命令白名单：

```json
{
  "tools": {
    "allow": {
      "exec": [
        "npm",
        "git",
        "ls",
        "cat",
        "mkdir",
        "rm"
      ]
    }
  }
}
```

**白名单规则**：

| 配置 | 含义 |
|-----|------|
| `"exec": true` | 允许所有命令（危险） |
| `"exec": ["npm", "git"]` | 只允许 npm 和 git |
| `"exec": false` | 禁止所有命令执行 |

---

## 三、tools.deny 配置

### 3.1 禁止特定工具

```json
{
  "tools": {
    "allow": {
      "exec": true
    },
    "deny": {
      "exec": ["rm", "sudo", "chmod"]
    }
  }
}
```

### 3.2 禁止模式

即使 `allow: true`，`deny` 中的命令也会被阻止：

```json
{
  "tools": {
    "allow": {
      "exec": true
    },
    "deny": {
      "exec": [
        "rm -rf",
        "sudo",
        "chmod 777",
        "> /etc/"
      ]
    }
  }
}
```

---

## 四、requireConfirmation 配置

### 4.1 需要确认的工具

敏感操作需要用户确认：

```json
{
  "tools": {
    "allow": {
      "exec": true,
      "write": true
    },
    "requireConfirmation": {
      "exec": ["rm", "sudo"],
      "write": ["/etc/", "~/.ssh/"]
    }
  }
}
```

### 4.2 确认流程

```
Agent 请求执行 rm -rf /tmp/test
    ↓
OpenClaw 检测到 requireConfirmation
    ↓
向用户发送确认请求
    ↓
用户确认 → 执行
用户拒绝 → 取消
```

---

## 五、内置安全机制

### 5.1 Shell 结构阻止

OpenClaw 会解析 Shell 结构，阻止危险模式：

| 模式 | 说明 | 状态 |
|-----|------|------|
| `>` | 重定向（防止覆盖文件） | 阻止 |
| `$(...)` | 命令替换（防止嵌套攻击） | 阻止 |
| `(...)` | 子 Shell（防止逃逸） | 阻止 |
| `&&` `||` | 链式执行（防止复杂攻击） | 阻止 |

### 5.2 示例

```bash
# 即使在白名单中，这些也会被阻止：
echo "test" > /etc/passwd     # 重定向被阻止
echo $(rm -rf /)              # 命令替换被阻止
ls && rm -rf /                # 链式执行被阻止
```

---

## 六、按场景配置

### 6.1 只读模式

```json
{
  "tools": {
    "allow": {
      "read": true,
      "web_search": true,
      "web_fetch": true
    },
    "deny": {
      "write": true,
      "edit": true,
      "exec": true
    }
  }
}
```

### 6.2 开发模式

```json
{
  "tools": {
    "allow": {
      "read": true,
      "write": true,
      "edit": true,
      "exec": ["npm", "git", "node", "ls", "cat", "mkdir"],
      "web_search": true
    },
    "requireConfirmation": {
      "exec": ["rm"]
    }
  }
}
```

### 6.3 完全信任模式（危险）

```json
{
  "tools": {
    "allow": {
      "read": true,
      "write": true,
      "edit": true,
      "exec": true,
      "browser": true
    }
  }
}
```

**警告**：只在隔离环境中使用。

---

## 七、CLI 命令

### 7.1 查看权限

```bash
# 查看当前工具权限
openclaw config get tools.allow

# 查看禁止列表
openclaw config get tools.deny
```

### 7.2 设置权限

```bash
# 允许特定工具
openclaw config set tools.allow.read true
openclaw config set tools.allow.exec '["npm", "git"]'

# 禁止特定工具
openclaw config set tools.deny.exec '["rm", "sudo"]'

# 设置需要确认的工具
openclaw config set tools.requireConfirmation.exec '["rm"]'
```

### 7.3 删除权限配置

```bash
# 删除配置，恢复默认
openclaw config unset tools.deny.exec
```

---

## 八、多 Agent 权限隔离

### 8.1 全局默认权限

```json
{
  "agents": {
    "defaults": {
      "tools": {
        "allow": {
          "read": true,
          "web_search": true
        }
      }
    }
  }
}
```

### 8.2 单个 Agent 权限

```json
{
  "agents": {
    "list": [
      {
        "id": "coder",
        "tools": {
          "allow": {
            "exec": ["npm", "git", "node"]
          },
          "deny": {
            "exec": ["rm -rf"]
          }
        }
      },
      {
        "id": "researcher",
        "tools": {
          "allow": {
            "web_search": true,
            "web_fetch": true
          },
          "deny": {
            "exec": true
          }
        }
      }
    ]
  }
}
```

---

## 九、权限审计

### 9.1 检查权限配置

```bash
# 查看完整工具配置
openclaw config get tools

# 诊断权限问题
openclaw doctor
```

### 9.2 日志审计

```bash
# 查看工具调用日志
openclaw logs --follow | grep -i tool

# 查看被拒绝的操作
openclaw logs | grep -i denied
```

---

## 十、最佳实践

### 10.1 最小权限原则

只开启需要的工具：

| 场景 | 需要的工具 |
|-----|-----------|
| 信息查询 | read, web_search |
| 文档处理 | read, write, edit |
| 代码开发 | read, write, exec（白名单） |
| 系统管理 | 需要严格限制 |

### 10.2 分层权限

```
默认权限（最小）
    ↓
Agent 特定权限（覆盖默认）
    ↓
requireConfirmation（最后防线）
```

### 10.3 定期审计

```bash
# 检查权限配置
openclaw config get tools

# 检查执行历史
cat ~/.openclaw/agents/main/sessions/*.jsonl | grep exec
```

---

## 十一、常见问题

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| 命令被拒绝 | 不在白名单 | 添加到 allow.exec |
| 命令需要确认 | 在 requireConfirmation | 确认或移除配置 |
| 操作被阻止 | Shell 结构阻止 | 使用安全替代方案 |

---

*本文基于 OpenClaw 官方文档整理。*
