# OpenClaw 自定义 Skill 实战

从零开发一个实用的 Skill，扩展 OpenClaw 的能力。

---

## 一、实战目标

开发一个 **"每日简报生成器"** Skill：

- 从 memory/YYYY-MM-DD.md 读取今日记录
- 从 Git 提交记录获取工作内容
- 生成结构化简报

---

## 二、Skill 目录结构

```bash
# 创建 Skill 目录
mkdir -p ~/.openclaw/workspace/skills/daily-report/scripts
mkdir -p ~/.openclaw/workspace/skills/daily-report/references
```

最终结构：

```
daily-report/
├── SKILL.md              # Skill 定义
├── scripts/
│   └── generate.py       # 生成脚本
└── references/
    └── template.md       # 简报模板
```

---

## 三、编写 SKILL.md

```markdown
---
name: daily-report
description: "生成每日工作简报。触发词：日报、每日简报、今天做了什么、生成简报"
---

# 每日简报生成器

## 功能

从 memory/YYYY-MM-DD.md 和 Git 提交记录生成每日简报。

## 使用方式

当用户说"生成日报"或"今天做了什么"时：

1. 读取今日记忆文件
2. 查询今日 Git 提交
3. 生成结构化简报

## 执行命令

\`\`\`bash
python3 scripts/generate.py
\`\`\`

## 输出格式

\`\`\`markdown
# 每日简报 - YYYY-MM-DD

## 完成的任务
- 任务1
- 任务2

## Git 提交
- commit1: 描述
- commit2: 描述

## 明日计划
（根据今日记录推断）
\`\`\`

## 注意事项

- 如果 memory 文件不存在，说明"今天还没有记录"
- Git 提交只显示当前目录的提交
- 简报保存到 reports/daily/YYYY-MM-DD.md
```

---

## 四、编写生成脚本

```python
#!/usr/bin/env python3
"""
每日简报生成器
"""
import os
import subprocess
from datetime import datetime
from pathlib import Path

def get_today_memory():
    """读取今日记忆文件"""
    today = datetime.now().strftime('%Y-%m-%d')
    memory_file = Path.home() / '.openclaw/workspace/memory' / f'{today}.md'
    
    if not memory_file.exists():
        return None, today
    
    with open(memory_file, 'r', encoding='utf-8') as f:
        return f.read(), today

def get_git_commits():
    """获取今日 Git 提交"""
    try:
        # 获取今日提交
        result = subprocess.run(
            ['git', 'log', '--since="midnight"', '--oneline', '--no-merges'],
            capture_output=True,
            text=True,
            timeout=10
        )
        
        if result.returncode == 0 and result.stdout.strip():
            return result.stdout.strip().split('\n')
        return []
    except Exception:
        return []

def generate_report(memory_content, today, commits):
    """生成简报"""
    report = f"""# 每日简报 - {today}

## 记忆记录

"""
    
    if memory_content:
        report += memory_content
    else:
        report += "今天还没有记录。\n"
    
    report += """
---

## Git 提交

"""
    
    if commits:
        for commit in commits:
            report += f"- {commit}\n"
    else:
        report += "今天没有 Git 提交。\n"
    
    return report

def main():
    # 获取数据
    memory_content, today = get_today_memory()
    commits = get_git_commits()
    
    # 生成简报
    report = generate_report(memory_content, today, commits)
    
    # 保存简报
    report_dir = Path.home() / '.openclaw/workspace/reports/daily'
    report_dir.mkdir(parents=True, exist_ok=True)
    
    report_file = report_dir / f'{today}.md'
    with open(report_file, 'w', encoding='utf-8') as f:
        f.write(report)
    
    print(f"简报已生成: {report_file}")
    print("\n" + report)

if __name__ == '__main__':
    main()
```

保存到：`scripts/generate.py`

---

## 五、创建简报模板

```markdown
# 每日简报 - {{DATE}}

## 工作内容

### 完成的任务
{{TASKS}}

### 进行中的工作
{{IN_PROGRESS}}

## Git 提交

{{COMMITS}}

## 明日计划

{{TOMORROW}}

---

*生成时间: {{TIMESTAMP}}*
```

保存到：`references/template.md`

---

## 六、测试 Skill

### 6.1 重载 Skill

```bash
# 重载所有 Skill
openclaw skills reload

# 查看是否加载
openclaw skills list | grep daily-report
```

### 6.2 测试触发

与 OpenClaw 对话：

```
用户：生成日报
```

预期输出：

```
简报已生成: /root/.openclaw/workspace/reports/daily/2026-03-15.md

# 每日简报 - 2026-03-15

## 记忆记录

今天还没有记录。

---

## Git 提交

- abc123: docs: 新增每日简报 Skill
```

---

## 七、进阶功能

### 7.1 添加参数支持

修改 `generate.py`：

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument('--date', help='指定日期 (YYYY-MM-DD)')
parser.add_argument('--output', help='输出路径')
args = parser.parse_args()

# 使用指定日期或今天
target_date = args.date or datetime.now().strftime('%Y-%m-%d')
```

### 7.2 添加邮件发送

```python
def send_email(report, to):
    """发送邮件"""
    import smtplib
    from email.mime.text import MIMEText
    
    msg = MIMEText(report)
    msg['Subject'] = f'每日简报 - {datetime.now().strftime("%Y-%m-%d")}'
    msg['From'] = 'openclaw@example.com'
    msg['To'] = to
    
    # 发送邮件
    # ...
```

### 7.3 添加定时任务

```bash
# 每天下午 6 点生成简报
openclaw cron add "0 18 * * *" "生成日报"
```

---

## 八、Skill 开发最佳实践

### 8.1 描述要清晰

```markdown
---
# 好的描述
description: "生成每日工作简报。触发词：日报、每日简报、今天做了什么"

# 不好的描述
description: "简报工具"
---
```

### 8.2 提供完整示例

```markdown
## 示例

用户：帮我生成今天的日报
执行：python3 scripts/generate.py
输出：简报已生成: reports/daily/2026-03-15.md
```

### 8.3 处理错误情况

```python
try:
    # 尝试读取文件
    content = read_file(path)
except FileNotFoundError:
    print(f"文件不存在: {path}")
    return None
```

### 8.4 添加日志

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info("开始生成简报")
logger.info(f"读取记忆文件: {memory_file}")
```

---

## 九、调试 Skill

### 9.1 直接执行脚本

```bash
# 测试脚本
python3 ~/.openclaw/workspace/skills/daily-report/scripts/generate.py
```

### 9.2 查看日志

```bash
# 查看 Skill 相关日志
openclaw logs | grep -i skill
```

### 9.3 检查 Skill 加载

```bash
# 列出已加载的 Skill
openclaw skills list --enabled
```

---

## 十、发布 Skill

### 10.1 发布到 ClawHub

```bash
# 发布 Skill
openclaw skills publish daily-report
```

### 10.2 发布到 GitHub

1. 创建 GitHub 仓库
2. 上传 Skill 文件
3. 在 ClawHub 提交

---

## 十一、完整代码

完整代码已保存到：

```
~/.openclaw/workspace/skills/daily-report/
```

---

*本文通过实战演示 Skill 开发流程。*
