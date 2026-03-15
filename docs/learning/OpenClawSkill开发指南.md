# OpenClaw Skill 开发指南

Skill 是 OpenClaw 的能力扩展模块。通过编写 Skill，可以教会智能体新的工作流程。

---

## 一、Skill 基础

### 1.1 什么是 Skill？

| 概念 | 类比 | 作用 |
|-----|------|------|
| **Tool** | 器官 | 决定"能不能做" |
| **Skill** | 教科书 | 教会"怎么做" |

Skill 不改变权限，只是告诉智能体如何组合工具完成任务。

### 1.2 Skill 目录结构

```
~/.openclaw/workspace/skills/
├── my-skill/
│   ├── SKILL.md          # 必需：Skill 定义
│   ├── scripts/          # 可选：脚本
│   └── references/       # 可选：参考资料
```

---

## 二、SKILL.md 编写

### 2.1 基本格式

```markdown
---
name: my-skill
description: "技能描述，用于智能体理解何时使用"
---

# Skill 标题

技能的详细说明和使用方法。
```

### 2.2 必需字段

| 字段 | 说明 | 示例 |
|-----|------|------|
| `name` | Skill 名称 | `pdf-convert` |
| `description` | 描述（智能体可见） | "将 Markdown 转换为 PDF" |

### 2.3 完整示例

```markdown
---
name: daily-report
description: "生成每日工作简报。触发词：日报、每日简报、今天做了什么"
---

# 每日简报生成器

## 功能

从 memory/YYYY-MM-DD.md 和 Git 提交记录生成每日简报。

## 使用方式

当用户说"生成日报"或"今天做了什么"时：

1. 读取今日记忆文件
2. 查询今日 Git 提交
3. 生成结构化简报

## 输出格式

```markdown
# 每日简报 - YYYY-MM-DD

## 完成的任务
- [ ] 任务1
- [ ] 任务2

## Git 提交
- commit1: 描述
- commit2: 描述

## 明日计划
- [ ] 计划1
```

## 注意事项

- 如果 memory 文件不存在，说明"今天还没有记录"
- Git 提交只显示当前目录的提交
```

---

## 三、触发词配置

### 3.1 在 description 中定义

```markdown
---
description: "PDF 转换工具。触发词：转PDF、转成PDF、md转pdf、生成PDF"
---
```

### 3.2 常见触发词模式

| 模式 | 示例 |
|-----|------|
| 动作+对象 | 转PDF、写文章、发邮件 |
| 对象+动作 | PDF转换、文章优化 |
| 功能描述 | 深度调研、联网搜索 |

---

## 四、脚本集成

### 4.1 目录结构

```
my-skill/
├── SKILL.md
├── scripts/
│   ├── main.py          # 主脚本
│   └── utils.py         # 工具函数
└── references/
    └── template.md      # 模板文件
```

### 4.2 在 SKILL.md 中引用脚本

```markdown
---
name: pdf-convert
description: "将 Markdown 转换为 PDF。触发词：转PDF、转成PDF"
---

# PDF 转换

## 使用方式

执行脚本：

\`\`\`bash
python3 scripts/convert.py <input.md> [output.pdf]
\`\`\`

## 参数

| 参数 | 说明 |
|-----|------|
| input.md | 输入的 Markdown 文件 |
| output.pdf | 输出的 PDF 文件（可选） |

## 示例

\`\`\`bash
python3 scripts/convert.py docs/report.md
\`\`\`
```

### 4.3 脚本示例

```python
#!/usr/bin/env python3
import sys
import subprocess

def convert_md_to_pdf(input_path, output_path=None):
    if output_path is None:
        output_path = input_path.replace('.md', '.pdf')
    
    cmd = [
        'pandoc', input_path,
        '-o', output_path,
        '--pdf-engine=xelatex',
        '-V', 'CJKmainfont=Noto Sans CJK SC'
    ]
    
    subprocess.run(cmd, check=True)
    return output_path

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python convert.py <input.md> [output.pdf]")
        sys.exit(1)
    
    input_path = sys.argv[1]
    output_path = sys.argv[2] if len(sys.argv) > 2 else None
    
    result = convert_md_to_pdf(input_path, output_path)
    print(f"PDF 已生成: {result}")
```

---

## 五、Skill 类型

### 5.1 纯文档型

只有 SKILL.md，无脚本：

```markdown
---
name: git-workflow
description: "Git 工作流指南。触发词：git、提交、分支"
---

# Git 工作流

## 提交规范

- feat: 新功能
- fix: 修复
- docs: 文档
- refactor: 重构

## 分支策略

- main: 生产分支
- develop: 开发分支
- feature/*: 功能分支
```

### 5.2 脚本型

包含可执行脚本：

```
my-skill/
├── SKILL.md
└── scripts/
    └── main.py
```

### 5.3 混合型

文档 + 脚本 + 模板：

```
my-skill/
├── SKILL.md
├── scripts/
│   └── main.py
└── references/
    └── template.md
```

---

## 六、Skill 安装与管理

### 6.1 安装 Skill

```bash
# 从 ClawHub 安装
openclaw skills install skill-name

# 从 GitHub 安装
openclaw skills install github:user/repo

# 本地安装（复制到目录）
cp -r my-skill ~/.openclaw/workspace/skills/
```

### 6.2 管理 Skill

```bash
# 列出已安装的 Skill
openclaw skills list

# 启用/禁用 Skill
openclaw skills enable skill-name
openclaw skills disable skill-name

# 更新 Skill
openclaw skills update skill-name

# 卸载 Skill
openclaw skills uninstall skill-name
```

### 6.3 重载 Skill

修改 SKILL.md 后需要重载：

```bash
openclaw skills reload
```

---

## 七、最佳实践

### 7.1 描述要清晰

```markdown
---
# 好的描述
description: "将 Markdown 转换为 PDF。触发词：转PDF、转成PDF、md转pdf"

# 不好的描述
description: "PDF工具"
---
```

### 7.2 提供使用示例

```markdown
## 示例

用户：帮我把 report.md 转成 PDF
执行：python3 scripts/convert.py report.md
输出：PDF 已生成: report.pdf
```

### 7.3 说明前置条件

```markdown
## 前置条件

- 需要安装 pandoc
- 需要安装 xelatex（中文支持）

安装命令：
\`\`\`bash
apt install pandoc texlive-xetex
\`\`\`
```

### 7.4 处理错误情况

```markdown
## 错误处理

- 如果文件不存在，提示"文件未找到"
- 如果转换失败，检查 pandoc 是否安装
```

---

## 八、调试 Skill

### 8.1 查看 Skill 是否加载

```bash
openclaw skills list --enabled
```

### 8.2 测试触发词

直接与智能体对话，使用触发词测试。

### 8.3 查看日志

```bash
openclaw logs --follow | grep -i skill
```

---

## 九、Skill 安全

### 9.1 审查第三方 Skill

安装前检查：

| 检查项 | 说明 |
|-------|------|
| 脚本内容 | 是否有危险操作 |
| 网络请求 | 是否发送数据到外部 |
| 文件操作 | 是否访问敏感文件 |

### 9.2 使用安全审计

```bash
# 审计配置和本地状态
openclaw security audit

# 深度审计（包括 Gateway 检查）
openclaw security audit --deep

# 自动修复安全问题
openclaw security audit --fix
```

---

## 十、实战案例

### 10.1 简单 Skill

```markdown
---
name: timestamp
description: "获取当前时间戳。触发词：时间戳、当前时间"
---

# 时间戳工具

## 使用方式

执行命令：

\`\`\`bash
date +%s
\`\`\`

## 输出

Unix 时间戳（秒）
```

### 10.2 复杂 Skill

```markdown
---
name: project-init
description: "初始化项目结构。触发词：初始化项目、新建项目"
---

# 项目初始化

## 功能

创建标准项目目录结构。

## 使用方式

\`\`\`bash
bash scripts/init.sh <project-name>
\`\`\`

## 生成的目录结构

\`\`\`
project-name/
├── src/
├── docs/
├── tests/
├── README.md
└── .gitignore
\`\`\`
```

---

## 十一、常见问题

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| Skill 不生效 | 未重载 | `openclaw skills reload` |
| 触发词不识别 | 描述不清晰 | 优化 description |
| 脚本执行失败 | 权限或路径问题 | 检查脚本权限 |

---

*本文基于 OpenClaw 官方文档整理。*
