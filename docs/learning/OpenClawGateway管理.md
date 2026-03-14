# OpenClaw Gateway 管理

Gateway 是 OpenClaw 的核心组件，负责消息路由、会话管理和工具调度。

---

## 一、Gateway 基础

### 1.1 什么是 Gateway？

Gateway 是 OpenClaw 的控制平面：

| 功能 | 说明 |
|-----|------|
| 消息路由 | 将消息分发到正确的 Agent |
| 会话管理 | 管理会话生命周期 |
| 工具调度 | 执行工具调用 |
| 渠道适配 | 对接各消息平台 |

### 1.2 默认配置

| 配置项 | 默认值 |
|-------|-------|
| 端口 | 18789 |
| 绑定地址 | 127.0.0.1 |
| 配置文件 | ~/.openclaw/openclaw.json |

---

## 二、Gateway 生命周期管理

### 2.1 启动 Gateway

```bash
# 前台运行（调试用）
openclaw gateway run

# 强制启动（忽略警告）
openclaw gateway run --force

# 后台启动
openclaw gateway start

# 以守护进程方式启动
openclaw gateway start --daemon
```

### 2.2 停止 Gateway

```bash
# 停止 Gateway
openclaw gateway stop

# 强制停止
openclaw gateway stop --force
```

### 2.3 重启 Gateway

```bash
# 重启 Gateway
openclaw gateway restart

# 强制重启
openclaw gateway restart --force
```

### 2.4 查看状态

```bash
# 基本状态
openclaw gateway status

# 详细状态
openclaw gateway status --verbose
```

**输出示例**：

```
Gateway Status:
  Runtime: running
  PID: 12345
  Uptime: 2h 30m
  Port: 18789
  RPC probe: ok
```

---

## 三、Gateway 配置

### 3.1 端口配置

```json
{
  "gateway": {
    "port": 18789
  }
}
```

### 3.2 绑定地址

```json
{
  "gateway": {
    "bind": "127.0.0.1"
  }
}
```

| 绑定地址 | 说明 |
|---------|------|
| `127.0.0.1` | 仅本机访问（默认） |
| `0.0.0.0` | 所有接口可访问 |
| `192.168.1.100` | 特定 IP |

### 3.3 认证配置

```json
{
  "gateway": {
    "auth": {
      "token": "your-secure-token"
    }
  }
}
```

### 3.4 外部访问配置

```json
{
  "gateway": {
    "bind": "0.0.0.0",
    "auth": {
      "token": "your-secure-token"
    },
    "allowInsecureAuth": true
  }
}
```

**警告**：外部访问需要配置认证，否则有安全风险。

---

## 四、Gateway 探测

### 4.1 可达性检查

```bash
# 检查 Gateway 是否可达
openclaw gateway probe
```

**输出示例**：

```
Gateway Probe:
  Reachable: yes
  Response time: 5ms
  RPC: ok
```

### 4.2 常见探测结果

| 结果 | 说明 |
|-----|------|
| `Reachable: yes` | Gateway 正常 |
| `Reachable: no` | Gateway 未启动或端口被阻止 |
| `RPC: limited` | 认证受限（非错误） |
| `RPC: ok` | RPC 连接正常 |

---

## 五、守护进程管理

### 5.1 安装为系统服务

```bash
# 安装用户级服务
openclaw daemon install --user

# 安装系统级服务（需要 root）
sudo openclaw daemon install
```

### 5.2 服务管理

```bash
# 启动服务
systemctl --user start openclaw-gateway

# 停止服务
systemctl --user stop openclaw-gateway

# 重启服务
systemctl --user restart openclaw-gateway

# 查看状态
systemctl --user status openclaw-gateway
```

### 5.3 开机自启

```bash
# 启用开机自启
systemctl --user enable openclaw-gateway

# 禁用开机自启
systemctl --user disable openclaw-gateway
```

### 5.4 查看服务日志

```bash
# 查看服务日志
journalctl --user -u openclaw-gateway

# 实时查看
journalctl --user -u openclaw-gateway -f
```

---

## 六、Gateway 日志

### 6.1 查看日志

```bash
# 查看日志
openclaw logs

# 实时日志
openclaw logs --follow

# 最近 N 行
openclaw logs --tail 100

# 按级别过滤
openclaw logs --level error
```

### 6.2 日志配置

```json
{
  "gateway": {
    "logging": {
      "level": "info",
      "format": "json"
    }
  }
}
```

### 6.3 日志级别

| 级别 | 说明 |
|-----|------|
| `error` | 仅错误 |
| `warn` | 警告及以上 |
| `info` | 信息及以上（默认） |
| `debug` | 调试信息 |
| `trace` | 详细追踪 |

---

## 七、Gateway 性能调优

### 7.1 连接池配置

```json
{
  "gateway": {
    "connectionPool": {
      "maxConnections": 100,
      "idleTimeout": 30000
    }
  }
}
```

### 7.2 超时配置

```json
{
  "gateway": {
    "timeout": {
      "request": 30000,
      "idle": 60000
    }
  }
}
```

### 7.3 内存限制

```json
{
  "gateway": {
    "memory": {
      "maxHeapSize": "2G"
    }
  }
}
```

---

## 八、多 Gateway 实例

### 8.1 配置隔离

```bash
# 使用不同配置文件
openclaw gateway run --config /path/to/config1.json
openclaw gateway run --config /path/to/config2.json

# 使用不同端口
# config1.json: port 18789
# config2.json: port 18790
```

### 8.2 Profile 模式

```bash
# 使用不同 profile
openclaw gateway run --profile dev
openclaw gateway run --profile prod
```

---

## 九、Gateway 故障排查

### 9.1 Gateway 无法启动

**症状**：
```
Error: Gateway start failed
```

**排查步骤**：

```bash
# 1. 检查端口占用
netstat -tlnp | grep 18789

# 2. 检查配置
openclaw doctor

# 3. 查看错误日志
openclaw logs --level error

# 4. 强制启动
openclaw gateway run --force
```

### 9.2 Gateway 频繁重启

**可能原因**：
- 配置错误
- 内存不足
- 依赖服务不可用

**排查步骤**：

```bash
# 1. 检查系统资源
free -h
df -h

# 2. 检查日志
journalctl --user -u openclaw-gateway -n 100

# 3. 检查配置热加载
openclaw config get hotReload
```

### 9.3 Gateway 响应慢

**可能原因**：
- 上下文过大
- 模型响应慢
- 网络延迟

**排查步骤**：

```bash
# 1. 检查会话状态
openclaw status --deep

# 2. 检查模型延迟
openclaw models test

# 3. 检查网络
ping api.openai.com
```

---

## 十、Gateway 安全

### 10.1 认证配置

```json
{
  "gateway": {
    "auth": {
      "token": "生成一个强密码",
      "allowInsecureAuth": false
    }
  }
}
```

### 10.2 HTTPS 配置

```json
{
  "gateway": {
    "tls": {
      "enabled": true,
      "cert": "/path/to/cert.pem",
      "key": "/path/to/key.pem"
    }
  }
}
```

### 10.3 访问控制

```json
{
  "gateway": {
    "allowFrom": [
      "127.0.0.1",
      "192.168.1.0/24"
    ]
  }
}
```

---

## 十一、Gateway CLI 命令速查

| 命令 | 功能 |
|-----|------|
| `openclaw gateway run` | 前台运行 |
| `openclaw gateway start` | 后台启动 |
| `openclaw gateway stop` | 停止 |
| `openclaw gateway restart` | 重启 |
| `openclaw gateway status` | 查看状态 |
| `openclaw gateway probe` | 可达性检查 |
| `openclaw daemon install` | 安装为服务 |
| `openclaw logs` | 查看日志 |

---

## 十二、最佳实践

### 12.1 生产环境建议

| 建议 | 说明 |
|-----|------|
| 使用守护进程 | systemd 管理 |
| 配置认证 | 防止未授权访问 |
| 定期备份 | 备份配置文件 |
| 监控日志 | 设置告警 |

### 12.2 开发环境建议

| 建议 | 说明 |
|-----|------|
| 前台运行 | 方便调试 |
| 详细日志 | `--level debug` |
| 热加载 | 配置自动生效 |

---

*本文基于 OpenClaw 官方文档整理。*
