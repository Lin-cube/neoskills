# neoskills

[![PyPI version](https://img.shields.io/pypi/v/neoskills.svg)](https://pypi.org/project/neoskills/)
[![Python 3.13+](https://img.shields.io/badge/python-3.13+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![CI](https://github.com/neolaf2/neoskills/actions/workflows/ci.yml/badge.svg)](https://github.com/neolaf2/neoskills/actions/workflows/ci.yml)

> 📖 本文档是 [README.md](README.md) 的简体中文翻译（节选浓缩版），供中文社区快速上手。
> 完整、最新的内容以英文原版为准。

**面向 AI 编程智能体的 Homebrew 式技能管理器。**

neoskills 在多个智能体生态（Claude Code、OpenCode、OpenClaw）之间管理可移植的技能定义。你可以在一个地方浏览技能、同步到 GitHub、通过符号链接按需部署，并通过内置的本体（Ontology）图谱发现技能之间的关系。

---

## 目录

- [安装](#安装)
- [快速上手](#快速上手)
- [工作原理](#工作原理)
- [本体层（Ontology）](#本体层ontology)
- [智能体目标（Agent Targets）](#智能体目标agent-targets)
- [运行模式](#运行模式)
- [CLI 参考](#cli-参考)
- [开发](#开发)

---

## 安装

### 从 PyPI 安装（推荐）

```bash
pip install neoskills
```

或使用 [uv](https://github.com/astral-sh/uv)：

```bash
uv tool install neoskills
```

### 从源码安装

```bash
git clone https://github.com/neolaf2/neoskills
cd neoskills
uv sync --dev
uv run neoskills --help
```

### 环境要求

- **Python 3.13+**（必需）
- **Git**（tap 操作需要）
- **uv**（开发推荐）

---

## 快速上手

```bash
# 1. 初始化工作区
neoskills init

# 2. 添加 tap（一个由 Git 托管的技能仓库）
neoskills tap https://github.com/your-org/my-skills

# 3. 浏览与搜索
neoskills list                          # 列出所有 tap 中的技能
neoskills list --linked                 # 查看已部署到智能体的技能
neoskills search "document processing"  # 跨 tap 搜索

# 4. 部署一个技能到你的智能体
neoskills install kstar-loop            # 复制到 tap 并创建符号链接

# 5. 创建你自己的技能
neoskills create my-new-skill -d "What it does" --type task

# 6. 检查系统健康状态
neoskills doctor
```

### 刚刚发生了什么？

`neoskills init` 创建了 `~/.neoskills/` 工作区和默认 tap。`install` 在 tap 中找到技能，并在智能体的技能目录（如 `~/.claude/skills/`）中创建符号链接——你的智能体现在可以使用该技能了。`create` 则生成了带有 `SKILL.md` + `ontology.yaml` 的新技能骨架。

---

## 工作原理

### 架构

```
~/.neoskills/                         # 工作区根目录
├── config.yaml                       # 目标、tap、默认配置
├── taps/                             # Git 克隆的技能仓库
│   ├── mySkills/                     # 默认 tap
│   │   └── skills/
│   │       └── <skill-id>/
│   │           ├── SKILL.md          # 技能定义（frontmatter + 正文）
│   │           ├── ontology.yaml     # 图谱元数据（可选，推荐）
│   │           ├── scripts/          # 可执行代码（可选）
│   │           ├── references/       # 辅助文档（可选）
│   │           └── assets/           # 媒体/模板（可选）
│   └── <other-taps>/                 # 其他技能来源
└── cache/                            # 备份与临时存储
```

### 部署模型

技能通过**逐技能符号链接**部署到智能体（零拷贝、可即时撤销）：

```
~/.claude/skills/kstar-loop  -->  ~/.neoskills/taps/mySkills/skills/kstar-loop
```

这意味着：

- **单一事实来源**：技能保存在 tap 中（用 Git 做版本控制）
- **多智能体共享**：多个智能体可同时链接同一技能
- **无需拷贝**：tap 中的修改立即生效
- **可撤销**：`neoskills unlink` 只删除符号链接，不影响其他内容

### 技能的组成

每个技能至少是一个包含 `SKILL.md` 的目录：

```yaml
# SKILL.md frontmatter
---
name: my-skill
description: "What this skill does"
author: "Your Name"
tags: [productivity, automation]
targets: [claude-code]
---

# My Skill

智能体将读取的指令、提示词与文档。
```

可选的 `ontology.yaml` 边车文件提供图谱元数据（类型、领域、生命周期、关系边、版本），参见[本体层](#本体层ontology)。

---

## 本体层（Ontology）

本体为技能叠加了一层**属性图**——节点和带类型的边存储在 `ontology.yaml` 边车文件中，无需外部数据库，图谱在运行时从文件系统物化生成。

### 渐进式增强（Progressive Enrichment）

技能不必在第一天就拥有完整元数据：

| 级别 | 已有内容 | 如何到达 |
|------|---------|---------|
| **L0 — 裸技能** | 仅 SKILL.md | 所有既有技能的默认状态 |
| **L1 — 已标记** | + ontology.yaml（type、domain、tags） | `neoskills ontology enrich <id>` |
| **L2 — 已连接** | + 关系边（requires、extends、composes、conflicts） | 作者声明关系 |
| **L3 — 已治理** | + 生命周期状态、版本、能力清单 | 持续维护 |

### 发现

```bash
# 分面搜索
neoskills ontology discover --domain agent-architecture --state operational
neoskills ontology discover --type meta --text "compiler"

# 加载并查看完整图谱
neoskills ontology load     # 打印摘要：技能数、边数、领域
neoskills ontology stats    # JSON 统计
```

### 依赖关系

```bash
neoskills ontology deps kstar-loop --transitive    # 它依赖什么
neoskills ontology rdeps kstar-planner --tree       # 谁依赖它
neoskills ontology add-edge skill-a skill-b -t requires
```

边类型：`requires`、`extends`、`composes_with`、`conflicts_with`、`supersedes`、`derived_from`。

### 生命周期

技能在状态机中流转：

```
candidate --> validated --> operational --> refined --> deprecated --> archived
```

```bash
neoskills ontology lifecycle                                          # 按状态查看所有技能
neoskills ontology transition my-skill validated --reason "tested"    # 变更状态
```

### 组合与拆分

```bash
# 将技能组合为流水线
neoskills ontology compose source-text-to-markdown research-md-to-latex \
  --mode pipeline --name md-to-paper

# 规划拆分方案
neoskills ontology split monolithic-skill sub-a sub-b sub-c
```

### 版本管理

```bash
neoskills ontology version kstar-loop --bump minor   # 0.1.0 -> 0.2.0
```

### 可视化与导出

```bash
neoskills ontology graph kstar-loop --depth 2 --format mermaid
neoskills ontology export --format json --output graph.json
neoskills ontology export --format dot                              # Graphviz
```

### 校验

```bash
neoskills ontology validate   # 检查断裂边、环、冲突
```

### 自动增强

```bash
neoskills ontology enrich my-skill                     # 单个技能 L0 -> L1
neoskills ontology enrich --all --level L1 --dry-run   # 批量预览
neoskills ontology enrich --all --level L1             # 批量执行
```

### 领域分类法

技能被组织成两级层次结构：agent-architecture、education、document-processing、business、knowledge-work、meta 等。领域可在可能的情况下从技能名称自动推断。

完整设计文档见 [docs/ontology-design.md](docs/ontology-design.md)。

---

## 智能体目标（Agent Targets）

内置目标：

| 目标 | 智能体 | 技能路径 |
|------|--------|---------|
| `claude-code` | Claude Code | `~/.claude/skills` |
| `opencode` | OpenCode | `~/.config/opencode/skills` |

添加自定义目标：

```bash
neoskills config set targets.my-agent.skill_path /path/to/skills
```

---

## 运行模式

1. **CLI**（默认）——`neoskills` 作为独立命令行工具运行
2. **智能体调用工具**——Claude Code 或 OpenCode 以编程方式调用 neoskills
3. **嵌入式 MCP 插件**——neoskills 作为 MCP 服务器运行在 Claude Code 内部，暴露 12+ 个工具（含本体操作）

### 认证（Claude 驱动的功能）

neoskills 自动解析认证方式：

1. **.env API key**——从 `./`、`.neoskills/` 或 `~/.neoskills/.env` 加载
2. **复用 SDK 订阅**——在 Claude Code/Desktop 内无需单独的 key
3. **禁用**——非 LLM 功能无需任何 key（tap、link、list、ontology 等）

---

## CLI 参考

### 工作区

| 命令 | 说明 |
|------|------|
| `neoskills init` | 创建 `~/.neoskills/` 工作区 |
| `neoskills config set\|get\|show` | 管理配置 |
| `neoskills doctor` | 健康检查（符号链接、配置、tap） |
| `neoskills migrate` | 从 v0.2 结构迁移到 v0.3+ |

### Tap

| 命令 | 说明 |
|------|------|
| `neoskills tap <url>` | 添加 tap（git clone） |
| `neoskills untap <name>` | 移除 tap |
| `neoskills update [name]` | 拉取 tap 最新内容 |
| `neoskills upgrade` | 更新所有 tap 并刷新链接 |
| `neoskills push` | 提交并推送 tap 到 GitHub |

### 技能

| 命令 | 说明 |
|------|------|
| `neoskills list [--linked\|--available]` | 列出技能 |
| `neoskills search <query>` | 跨 tap 搜索 |
| `neoskills info <skill_id>` | 技能详细信息 |
| `neoskills create <skill_id>` | 生成技能骨架（SKILL.md + ontology.yaml） |
| `neoskills install <skill_id>` | 一步部署（复制 + 链接） |
| `neoskills uninstall <skill_id>` | 卸载（取消链接并可选删除） |
| `neoskills link <skill_id>` | 创建符号链接（tap -> 目标） |
| `neoskills unlink <skill_id>` | 移除符号链接 |

### 本体

| 命令 | 说明 |
|------|------|
| `neoskills ontology load` | 构建图谱并打印摘要 |
| `neoskills ontology stats` | 图谱统计（JSON） |
| `neoskills ontology validate` | 完整性检查 |
| `neoskills ontology discover` | 分面搜索（--domain、--type、--state、--tag、--text） |
| `neoskills ontology deps <id>` | 依赖树 |
| `neoskills ontology rdeps <id>` | 反向依赖 |
| `neoskills ontology graph <id>` | 邻域图（Mermaid/DOT/JSON） |
| `neoskills ontology lifecycle` | 按生命周期状态查看技能 |
| `neoskills ontology transition <id> <state>` | 变更状态 |
| `neoskills ontology add-edge / remove-edge` | 增删关系边 |
| `neoskills ontology version <id> --bump <level>` | 版本号升级（major/minor/patch） |
| `neoskills ontology compose <ids...>` | 创建组合技能 |
| `neoskills ontology split <id> <names...>` | 拆分规划 |
| `neoskills ontology enrich [<id>\|--all]` | 自动增强元数据 |
| `neoskills ontology export --format <fmt>` | 导出图谱（json/mermaid/dot） |
| `neoskills ontology conflicts` | 报告冲突边 |

### 高级

| 命令 | 说明 |
|------|------|
| `neoskills enhance audit\|normalize\|add-docs\|add-tests` | Claude 驱动的技能增强 |
| `neoskills agent list\|run` | 自主智能体操作 |
| `neoskills plugin create\|validate` | 插件脚手架与校验 |
| `neoskills schedule daily` | 记忆增强的日程规划 |

---

## 开发

### 环境搭建

```bash
git clone https://github.com/neolaf2/neoskills
cd neoskills
uv sync --dev
```

### 常用命令

```bash
uv run pytest -v              # 运行全部测试（172 个）
uv run ruff check src/        # 代码检查
uv run neoskills --help        # 从源码运行
```

### 项目结构

```
src/neoskills/
├── cli/              # Click CLI 命令
│   ├── main.py       # 入口与命令注册
│   ├── create_cmd.py # 技能脚手架
│   ├── ontology_cmd.py # 17 个本体子命令
│   └── ...           # tap、link、list、doctor 等
├── core/             # Cellar、config、checksum、frontmatter、linker
├── ontology/         # 属性图层（v0.4）
│   ├── models.py     # 枚举 + 数据类（SkillNode、OntologyEdge 等）
│   ├── graph.py      # SkillGraph —— 内存属性图
│   ├── loader.py     # 文件系统 -> 图
│   ├── writer.py     # 图 -> 文件系统（ontology.yaml）
│   ├── engine.py     # 高层 API（OntologyEngine）
│   ├── taxonomy.py   # 领域分类法 + 推断
│   ├── lifecycle.py  # 状态机流转
│   ├── versioning.py # 语义化版本操作
│   ├── composition.py # 组合/拆分技能
│   ├── export.py     # Mermaid、DOT、JSON、ASCII 树
│   └── scaffold.py   # 基于模板的技能创建
├── runtime/          # 智能体运行时集成
│   └── claude/
│       └── plugin.py # MCP 插件（12+ 工具）
└── __init__.py
tests/
├── unit/             # 12 个测试模块共 172 个测试
└── integration/      # 端到端工作流测试
docs/
└── ontology-design.md  # 完整本体设计文档
```

### 发布流程

```bash
# 1. 更新 pyproject.toml 和 src/neoskills/__init__.py 中的版本号
# 2. 构建并上传
uv build
uv run twine upload dist/*

# 3. 验证
pip install --upgrade neoskills && neoskills --version
```

---

## 许可证

MIT —— 见 [LICENSE](LICENSE)

## 作者

Richard Tong

---

> 翻译说明：本中文文档由 WangChunYan 于 2026-11-22 基于 README.md（v0.4.x）翻译整理，
> 以帮助中文读者快速理解项目。如有错漏欢迎提交 Pull Request 修正。
