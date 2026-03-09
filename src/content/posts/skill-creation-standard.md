---
title: 适用于大模型的 Skill 创作标准
published: 2026-03-09
description: 一套面向大模型代理的 Skill 创作标准，覆盖最佳实践、描述结构规范与可直接复用的示例代码。
image: ''
tags: ['LLM', 'Skill', 'Agent', 'Prompt Engineering']
category: 'AI Engineering'
draft: false
lang: 'zh-CN'
---

# 适用于大模型的 Skill 创作标准

Skill 的本质不是“文档合集”，而是给大模型的“可执行操作系统”。
一个高质量 Skill 需要做到三件事：

1. **能被准确触发**（该用时一定被用到）
2. **能稳定执行**（步骤可重复、结果可验证）
3. **不浪费上下文**（只加载当前任务需要的信息）

本文给出一套实操标准，适用于在 Codex/Agent/LLM 工作流中构建可长期复用的 Skill。

## 1. Skill 创作最佳实践

### 1.1 先定义触发场景，再写实现

先写清楚“什么时候必须用这个 Skill”，再写内部步骤。
如果触发边界模糊，大模型就可能在该用时不用，不该用时误用。

建议先用下面三个问题约束范围：

1. 用户会说出哪些典型请求？
2. 这些请求的输入是什么（文件、命令、系统上下文）？
3. 哪些情况不应该触发这个 Skill？

### 1.2 保持 `SKILL.md` 精简，把细节下沉

`SKILL.md` 只放：

- 核心流程
- 关键约束
- 资源导航

把大段参考资料放到 `references/`，把可重复执行逻辑放到 `scripts/`，把模板和素材放到 `assets/`。
这就是“渐进加载”：先加载最小信息，按需再读细节。

### 1.3 把“容易出错的步骤”脚本化

凡是满足以下任意条件，都建议落成脚本：

- 需要稳定、可重复输出
- 手写容易漏参数或顺序
- 每次任务都在重复写同一段代码

经验上，脚本比长文本说明更可靠，也更节省上下文 token。

### 1.4 指令必须可执行、可检查

正文尽量使用祈使句，避免空泛描述。

- 好示例：`读取 references/schema.md 并生成字段映射表`
- 差示例：`理解一下 schema 后给出结果`

每个关键步骤最好包含“完成判定”或“失败处理”，例如：

- 完成判定：输出中必须包含 `summary` 与 `risk_items`
- 失败处理：解析失败时返回样例行号与原因，不可静默忽略

### 1.5 建立验证闭环

Skill 不是一次性交付，而是迭代资产。
建议每次更新都执行：

1. 运行验证脚本（如 frontmatter/命名规则检查）
2. 用 2 到 3 个真实任务回放
3. 记录失败案例并补充到 Skill 约束中

## 2. Skill 描述结构规范

下面是推荐的结构分层。

### 2.1 目录结构规范

```text
my-skill/
├── SKILL.md                    # 必需：触发 + 工作流
├── agents/
│   └── openai.yaml             # 推荐：UI 展示元数据
├── scripts/                    # 可选：确定性逻辑
│   └── run_task.py
├── references/                 # 可选：按需加载知识
│   ├── schema.md
│   └── policies.md
└── assets/                     # 可选：模板或输出素材
    └── report-template.md
```

### 2.2 `SKILL.md` 的最小规范

#### A. Frontmatter（强约束）

仅保留两个字段：

- `name`
- `description`

不要添加其他 frontmatter 字段，避免触发机制噪声。

#### B. Description（触发核心）

`description` 必须同时表达：

1. **做什么**（能力边界）
2. **什么时候用**（触发上下文）
3. **处理什么输入**（文件类型/任务类型）

可复用模板：

```text
<能力范围>。当用户需要 <任务A/任务B/任务C>，且输入为 <文件类型或环境> 时使用。
```

#### C. Body（执行说明）

正文建议分为四段：

1. 快速开始（最短路径）
2. 标准工作流（步骤 + 完成判定）
3. 异常处理（失败分支）
4. 资源导航（何时读取哪个 references 文件）

### 2.3 反例与修正

反例（触发弱）：

```yaml
description: 帮助处理文档。
```

修正（触发强）：

```yaml
description: 面向技术文档的创建、改写与结构化提取。当用户请求涉及 Markdown/Docx 文档的章节重组、术语统一、格式修复、摘要生成时使用；支持输入为 .md、.docx 与纯文本片段。
```

## 3. 实际 Skill 示例代码

下面给出一个可直接落地的示例 Skill：`skill-quality-gate`。

### 3.1 示例目录

```text
skill-quality-gate/
├── SKILL.md
├── scripts/
│   └── check_frontmatter.py
└── references/
    └── style-guide.md
```

### 3.2 `SKILL.md`

```markdown
---
name: skill-quality-gate
description: 校验并修复 Markdown 文章的元数据与结构一致性。当用户请求“检查文章规范”“修复 frontmatter”“统一发布格式”或需要批量审查博客文章时使用；输入通常为 .md 文件。
---

# Skill Quality Gate

## Quick Start

1. 读取目标 Markdown 文件的 frontmatter。
2. 用 `scripts/check_frontmatter.py` 执行规范检查。
3. 根据检查结果修复字段缺失、类型不匹配和日期格式问题。

## Workflow

1. 收集输入文件路径并逐个检查。
2. 若 `title/published/description/tags/category/draft` 缺失，则补全。
3. 若 `published` 不是 `YYYY-MM-DD`，转换为标准格式。
4. 输出修复摘要，包含：`fixed_files`、`warnings`、`manual_review_items`。

## Failure Handling

1. 解析 frontmatter 失败时，返回文件名和失败行号。
2. 禁止跳过错误继续写入，必须先报告再等待用户确认。

## References

- 当需要字段命名规范时，读取 `references/style-guide.md`。
```

### 3.3 `scripts/check_frontmatter.py`

```python
#!/usr/bin/env python3
import re
import sys
from pathlib import Path

REQUIRED_KEYS = {"title", "published", "description", "tags", "category", "draft"}
DATE_RE = re.compile(r"^\d{4}-\d{2}-\d{2}$")


def read_frontmatter(text: str) -> str:
    if not text.startswith("---\n"):
        raise ValueError("missing frontmatter start")
    end = text.find("\n---\n", 4)
    if end == -1:
        raise ValueError("missing frontmatter end")
    return text[4:end]


def parse_keys(frontmatter: str) -> dict:
    result = {}
    for line in frontmatter.splitlines():
        if ":" not in line:
            continue
        key, value = line.split(":", 1)
        result[key.strip()] = value.strip()
    return result


def validate(path: Path) -> list[str]:
    errors = []
    content = path.read_text(encoding="utf-8")
    fm = read_frontmatter(content)
    data = parse_keys(fm)

    missing = sorted(REQUIRED_KEYS - data.keys())
    if missing:
        errors.append(f"missing keys: {', '.join(missing)}")

    if "published" in data and not DATE_RE.match(data["published"].strip("'\"")):
        errors.append("published must be YYYY-MM-DD")

    if "draft" in data and data["draft"] not in {"true", "false"}:
        errors.append("draft must be true/false")

    return errors


def main() -> int:
    if len(sys.argv) < 2:
        print("usage: check_frontmatter.py <file1> [file2 ...]")
        return 1

    has_error = False
    for item in sys.argv[1:]:
        path = Path(item)
        try:
            errs = validate(path)
        except Exception as exc:
            print(f"{path}: parse error: {exc}")
            has_error = True
            continue

        if errs:
            has_error = True
            print(f"{path}:")
            for err in errs:
                print(f"  - {err}")
        else:
            print(f"{path}: ok")

    return 1 if has_error else 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### 3.4 `references/style-guide.md`

```markdown
# Style Guide

## Required Frontmatter Keys

- title: string
- published: YYYY-MM-DD
- description: string
- tags: array
- category: string
- draft: boolean

## Writing Rules

1. 标题控制在 8 到 28 个字。
2. 首段必须说明文章目标与适用对象。
3. 命令示例统一使用代码块并标注语言。
```

## 结语

如果你只做一件事，请先把 `description` 写到“可触发、可判定、可排除误触发”。
它决定了 Skill 会不会在正确的时刻出现。

在此基础上，再用“精简 `SKILL.md` + 下沉资源 + 脚本化关键步骤”的方式迭代，你的 Skill 才会从“能用”走向“稳定可复用”。
