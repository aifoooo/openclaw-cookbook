# OpenClaw 从零搭建实战

在新机器上完整配置 OpenClaw，从系统准备到正常运行。

---

## 一、系统要求

### 1.1 软件依赖

| 依赖 | 要求 |
|-----|------|
| **Node.js** | v22 LTS 或 v24（推荐） |
| **npm/pnpm** | npm v9+ 或 pnpm |
| **Git** | v2.30+ |
| **Docker** | v20+（可选，用于沙盒） |

### 1.2 硬件要求

| 场景 | 最低配置 |
|-----|---------|
| 云 API 模式 | 1核 1G |
| 本地模型（7B） | 4核 16G |
| 本地模型（70B） | 8核 48G |

### 1.3 操作系统

| 系统 | 支持 |
|-----|------|
| Linux | Ubuntu/Debian/CentOS/Arch |
| macOS | Intel & Apple Silicon |
| Windows | **WSL2**（强烈推荐） |

---

## 二、安装方式选择

### 2.1 方式对比

| 方式 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| **自动脚本** | 最简单 | 不够灵活 | 新手、快速上手 |
| **npm 安装** | 标准方式 | 需手动管理 Node | 开发者 |
| **Docker** | 隔离安全 | 需要容器知识 | 生产环境 |
| **源码构建** | 最新功能 | 复杂 | 贡献者 |

---

## 三、方式一：自动脚本安装

### 3.1 Linux/macOS/WSL2

```bash
# 一键安装
curl -fsSL https://openclaw.ai/install.sh | bash

# 跳过引导向导
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

### 3.2 Windows PowerShell

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

### 3.3 安装后验证

```bash
# 检查版本
openclaw --version

# 运行诊断
openclaw doctor

# 查看状态
openclaw status
```

---

## 四、方式二：npm 安装

### 4.1 安装 Node.js

```bash
# 使用 nvm 安装
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc

# 安装 Node 24
nvm install 24
nvm use 24
```

### 4.2 安装 OpenClaw

```bash
# npm 安装
npm install -g openclaw@latest

# 或使用 pnpm
pnpm add -g openclaw@latest
pnpm approve-builds -g
```

### 4.3 运行引导向导

```bash
# 交互式配置
openclaw onboard

# 安装为系统服务
openclaw onboard --install-daemon
```

---

## 五、方式三：Docker 部署

### 5.1 拉取镜像

```bash
# 拉取官方镜像
docker pull openclaw/openclaw:latest
```

### 5.2 创建配置目录

```bash
# 创建数据目录
mkdir -p ~/.openclaw
mkdir -p ~/openclaw/workspace
```

### 5.3 运行容器

```bash
docker run -d \
  --name openclaw \
  -p 18789:18789 \
  -v ~/.openclaw:/root/.openclaw \
  -v ~/openclaw/workspace:/root/.openclaw/workspace \
  -e API_KEY=your-api-key \
  --restart unless-stopped \
  openclaw/openclaw:latest
```

### 5.4 Docker Compose 方式

```yaml
# docker-compose.yml
version: '3'
services:
  openclaw:
    image: openclaw/openclaw:latest
    container_name: openclaw
    ports:
      - "18789:18789"
    volumes:
      - ~/.openclaw:/root/.openclaw
      - ~/openclaw/workspace:/root/.openclaw/workspace
    environment:
      - API_KEY=your-api-key
    restart: unless-stopped
```

```bash
docker-compose up -d
```

---

## 六、初始配置

### 6.1 配置 API Key

```bash
# 交互式配置
openclaw configure

# 或直接设置
openclaw config set model.primary "anthropic/claude-opus-4-5"
```

### 6.2 配置文件位置

```
~/.openclaw/
├── openclaw.json      # 主配置文件
├── agents/            # Agent 数据
├── sessions/          # 会话数据
└── credentials/       # 凭证存储
```

### 6.3 最小配置示例

```json
{
  "model": {
    "primary": "anthropic/claude-opus-4-5"
  },
  "tools": {
    "allow": {
      "read": true,
      "write": true,
      "exec": ["npm", "git"]
    }
  }
}
```

---

## 七、启动与验证

### 7.1 启动 Gateway

```bash
# 前台运行
openclaw gateway run

# 后台运行
openclaw gateway start

# 查看状态
openclaw gateway status
```

### 7.2 验证安装

```bash
# 诊断检查
openclaw doctor

# 查看状态
openclaw status

# 打开控制台
openclaw dashboard
```

### 7.3 测试对话

访问 `http://127.0.0.1:18789`，发送测试消息。

---

## 八、常见安装问题

### 8.1 Node.js 版本过低

**症状**：
```
Error: Node.js version 18.x is not supported
```

**解决**：
```bash
# 升级 Node.js
nvm install 24
nvm use 24
```

### 8.2 权限问题

**症状**：
```
Error: EACCES: permission denied
```

**解决**：
```bash
# 修复权限
sudo chown -R $(whoami) ~/.openclaw
```

### 8.3 端口被占用

**症状**：
```
Error: Port 18789 is already in use
```

**解决**：
```bash
# 查看端口占用
netstat -tlnp | grep 18789

# 修改端口
openclaw config set gateway.port 18790
```

### 8.4 Gateway 连接失败

**症状**：
```
Error: Gateway connect failed
```

**解决**：
```bash
# 检查 Gateway 状态
openclaw gateway status

# 重启 Gateway
openclaw gateway restart

# 运行诊断
openclaw doctor --fix
```

---

## 九、安全加固

### 9.1 配置认证

```json
{
  "gateway": {
    "auth": {
      "token": "生成一个强密码"
    }
  }
}
```

### 9.2 限制工具权限

```json
{
  "tools": {
    "allow": {
      "read": true,
      "write": true,
      "exec": ["npm", "git", "ls"]
    },
    "requireConfirmation": {
      "exec": ["rm"]
    }
  }
}
```

### 9.3 使用 Docker 沙盒

```json
{
  "sandbox": {
    "mode": "docker"
  }
}
```

---

## 十、安装检查清单

```bash
# 1. 检查 Node.js 版本
node --version  # 应为 v22+

# 2. 检查 OpenClaw 版本
openclaw --version

# 3. 运行诊断
openclaw doctor

# 4. 检查 Gateway 状态
openclaw gateway status

# 5. 检查配置
openclaw config get model

# 6. 测试对话
# 访问 http://127.0.0.1:18789
```

---

## 十一、下一步

安装完成后：

1. 配置渠道（Telegram/Discord/Slack）
2. 安装技能（Skills）
3. 配置记忆系统（memU）
4. 设置定时任务

---

*本文基于 OpenClaw 官方文档和社区经验整理。*
