# Trae IDE 功能全景介绍

> 本文系统介绍 Trae（基于 VS Code 内核的 AI IDE）中常用的 **8 大核心能力**：MCP、Skills、Agents、Sub Agents、Hooks、Command、Rules 以及与之配套的 `.trae` 目录约定，帮助你把 Trae 用到极致。

---

## 目录

- [1. MCP（Model Context Protocol）](#1-mcp-model-context-protocol)
- [2. Skills（技能）](#2-skills技能)
- [3. Agents（智能体）](#3-agents智能体)
- [4. Sub Agents（子智能体）](#4-sub-agents子智能体)
- [5. Hooks（钩子）](#5-hooks钩子)
- [6. Command（命令面板 / 斜杠命令）](#6-command命令面板--斜杠命令)
- [7. Rules（项目规则）](#7-rules项目规则)
- [8. `.trae` 工作目录约定](#8-trae-工作目录约定)
- [综合使用建议](#综合使用建议)

---

## 1. MCP（Model Context Protocol）

**MCP（Model Context Protocol）** 是 Trae 用来"**给大模型插上工具手**"的统一协议。通过 MCP，AI Agent 可以调用外部工具（数据库、浏览器、文件系统、GitHub、CI/CD…）而不必把逻辑硬编码进模型。

### 1.1 核心概念

- **MCP Server**：提供工具的服务进程，可以是本地可执行文件，也可以是远程 HTTP/SSE 服务。
- **MCP Client**：Trae 内置，负责发现、连接、调用 MCP Server。
- **Tool**：MCP Server 暴露的原子能力（带 schema 描述），LLM 通过 `tool_calls` 调用。

### 1.2 在 Trae 中的使用

在 Trae 的设置中：
1. 打开 `Settings → MCP`（或编辑 `.trae/mcp.json`）。
2. 添加 Server，例如：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_xxx" }
    }
  }
}
```

3. 重启 Trae 后，AI Agent 在对话中会自动获得这些工具。

### 1.3 典型场景

- 让 AI **直接读写数据库**（PostgreSQL / SQLite / MySQL MCP Server）。
- 让 AI **执行 GitHub Issue / PR**。
- 让 AI **浏览网页并抓取信息**（Playwright MCP、Chrome DevTools MCP）。
- 让 AI **调用本项目**的 Python 工具，例如 `python -m tools.init_db`。

### 1.4 调试技巧

- 使用 `MCP: List Servers` 查看当前已注册的 Server 与状态。
- 在 Agent 对话框输入 `使用 mcp 工具...`，观察工具调用 trace。

---

## 2. Skills（技能）

**Skill** 是 Trae 中"**面向特定任务的提示词 + 工作流封装**"，相当于给 AI 一份"操作手册"。

### 2.1 Skill 的组成

```text
skill-name/
├── SKILL.md          # 技能描述 + 触发条件 + 使用说明
├── scripts/          # 可选：技能执行所需的脚本
├── references/       # 可选：参考文档
└── assets/           # 可选：模板、图片等资源
```

`SKILL.md` 是入口，Trae 会读取它来决定"什么时候该调用这个 Skill"。

### 2.2 何时使用

- 任务**有固定流程**（例如：代码审查、安全扫描、生成 PDF、生成小程序）。
- 需要**加载额外的领域知识**或**专用工具**。
- 希望**减少重复 prompt**——把常用提示词模板化。

### 2.3 内置示例

Trae 内置技能包括但不限于：

| 技能 | 用途 |
| --- | --- |
| `pdf` | 解析、操作 PDF |
| `xlsx` | 处理 Excel 文件 |
| `ms-office-suite:pdf` | Office 套件相关 |
| `TRAE-browseruse` | 浏览器自动化 |
| `TRAE-code-review` | MR/PR 代码审查 |
| `TRAE-debugger` | 复杂 Bug 调试 |
| `TRAE-generate-mini-app` | 微信/支付宝小程序生成 |
| `TRAE-security-review` | 安全漏洞扫描 |
| `skill-creator` | 创建自定义 Skill |

### 2.4 如何调用

直接在对话中描述任务，Trae 会自动匹配；也可以用命令面板 `Skills: Run Skill` 手动触发。

---

## 3. Agents（智能体）

**Agent** 是 Trae 的"**主执行单元**"。一个 Agent = **大模型 + 上下文 + 工具集 + 行为规则**。

### 3.1 默认 Agent

- `general_purpose_task`：通用任务 Agent（Tools: Skill/SearchCodebase/Read/Edit/...）。
- `search`：检索型 Agent（适合跨多文件的语义搜索）。
- `statusline-setup`：状态栏配置 Agent。

### 3.2 何时主动使用 Agent

- 复杂、多步骤任务（**自动分解 → 子任务 → 合并**）。
- 跨层改动（前后端、API、数据库、CI 同步修改）。
- 不想让主对话被长输出撑爆（Agent 的返回是"摘要"，不污染上下文）。

### 3.3 调用方式

在 Agent 模式下用 `Task` 工具或 `@agent-name` 触发，例如：

```
@search 帮我找本项目所有调用 MySQL 的地方
```

或显式：

```python
Task(description="扫描 SQL 注入风险", subagent_type="general_purpose_task", query="...")
```

---

## 4. Sub Agents（子智能体）

**Sub Agent** 是一种"**临时雇佣的专家 Agent**"，主 Agent 在任务过程中按需生成。

### 4.1 特点

- **临时性**：每次 `Task` 调用都创建一个全新的 Sub Agent。
- **隔离上下文**：Sub Agent 的工具输出不会回到主对话，只返回最终摘要。
- **专业分工**：可选 `subagent_type`（如 `general_purpose_task`、`search`）。

### 4.2 与 Agent 的区别

| 维度 | Agent | Sub Agent |
| --- | --- | --- |
| 生命周期 | 持续 | 一次性 |
| 上下文共享 | 与主对话共享 | 独立 |
| 适用 | 通用交互 | 复杂子任务 |

### 4.3 实战示例

```python
# 让 Sub Agent 并行调研两个独立问题
[
    Task(subagent_type="search",      query="本项目使用了哪些大模型？"),
    Task(subagent_type="search",      query="本项目数据库表结构是什么？"),
    Task(subagent_type="general_purpose_task", query="修复 db/__init__.py 中可能存在的连接泄漏问题"),
]
```

---

## 5. Hooks（钩子）

**Hook** 是 Trae 在 **Agent 工具调用前后** 自动触发的"拦截器"，用于做安全校验、自动化注入、日志审计等。

### 5.1 Hook 类型

| Hook 事件 | 触发时机 |
| --- | --- |
| `PreToolUse` | 工具调用前（可拒绝/修改参数） |
| `PostToolUse` | 工具调用后（可读 result、写日志） |
| `Stop` | Agent 停止时（清理、通知） |
| `SubagentStop` | Sub Agent 停止时 |
| `SessionStart` / `SessionEnd` | 会话开始/结束 |

### 5.2 配置位置

`.trae/settings.json` 或 `~/.trae/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Write|Edit",
        "hooks": [
          { "type": "command", "command": "echo '即将执行危险操作' >> ~/.trae/audit.log" }
        ]
      }
    ]
  }
}
```

### 5.3 典型用法

- **强制走 lint / type-check**：在 `PostToolUse` 跑 `mypy` / `ruff`。
- **安全护栏**：`PreToolUse` 中拦截 `rm -rf`、`git push --force`。
- **审计日志**：所有工具调用记录到文件。
- **自动 commit**：完成一段工作后自动 `git commit`。

---

## 6. Command（命令面板 / 斜杠命令）

Trae 内置 **VS Code 命令面板** + **斜杠命令（Slash Commands）** 两套命令系统。

### 6.1 命令面板

- 快捷键：`Ctrl + Shift + P`（macOS: `Cmd + Shift + P`）。
- 可以搜索并执行 Trae / VS Code 的所有命令。
- 例如：`Trae: New Agent Session`、`Trae: Toggle MCP Server`。

### 6.2 斜杠命令

直接在 Trae 对话框输入：

| 命令 | 作用 |
| --- | --- |
| `/help` | 显示可用命令 |
| `/clear` | 清空当前对话 |
| `/compact` | 压缩历史上下文（节省 token） |
| `/cost` | 查看当前会话消耗 |
| `/agents` | 列出可用的 Agent / Sub Agent |
| `/skills` | 列出 Skill |
| `/mcp` | 查看 MCP Server 状态 |
| `/init` | 为当前项目生成 `AGENTS.md` |
| `/init-rule` | 生成项目级 Rules |

### 6.3 自定义命令

在 `.trae/commands/` 下放置 Markdown 文件，例如 `.trae/commands/deploy.md`：

```markdown
---
description: 一键部署到测试环境
---
请按以下步骤部署：
1. 运行 `pytest`
2. 打包 Docker 镜像
3. kubectl apply -f k8s/test/
```

在对话中输入 `/deploy` 即可触发。

---

## 7. Rules（项目规则）

**Rules** 是 Trae 中最容易被低估的功能：它决定 **AI 在这个项目中"应该/不应该"做什么**。

### 7.1 优先级

| 层级 | 文件 | 作用 |
| --- | --- | --- |
| 全局规则 | `~/.trae/CLAUDE.md` 或 `~/.trae/rules.md` | 所有项目生效 |
| 项目规则 | `.trae/rules.md` 或 `.trae/AGENTS.md` | 当前项目生效 |
| 局部规则 | 任意子目录的 `AGENTS.md` | 子目录及以下 |

> 越靠"叶子"目录优先级越高。

### 7.2 典型 Rules 内容

```markdown
# Project Rules

## 语言
- 所有注释、文档、日志必须使用中文。
- 公共 API 函数必须写 docstring。

## 编码风格
- Python 使用 ruff + black 格式化。
- TypeScript 使用 ESLint + Prettier。
- 不要使用 emoji（除非用户明确要求）。

## 安全
- 不要把 API key 写进代码或注释。
- 数据库迁移必须经过 review。

## 协作
- 涉及 *.sql 改动必须 @DBA。
- 提交前必须跑 `pytest`。
```

### 7.3 配合 Sub Agent 使用

在 `AGENTS.md` 中可以专门为某个子目录声明"专家规则"，例如 `db/AGENTS.md`：

```markdown
# 数据库模块规则
- 所有 ORM 模型必须继承 `DBModelBase`。
- 迁移脚本只允许放在 `db/migrations/`，不要直接修改表结构。
```

---

## 8. `.trae` 工作目录约定

```text
<project>/
├── .trae/
│   ├── mcp.json          # MCP Server 配置
│   ├── settings.json     # Hooks、UI、快捷键等
│   ├── rules.md          # 项目级 Rules
│   ├── AGENTS.md         # 项目级 Agent 行为约束
│   ├── commands/         # 自定义斜杠命令
│   │   └── deploy.md
│   ├── skills/           # 自定义 Skill
│   │   └── my-skill/
│   │       └── SKILL.md
│   └── agents/           # 自定义 Agent 配置
│       └── data-analyst.md
└── ... 项目源码
```

### 8.1 建议

- 把 `mcp.json`、`rules.md`、`AGENTS.md` 都 **commit 到仓库**，让团队共享。
- `settings.json` 中个人相关的（键位、主题）放到 `~/.trae/settings.json`，避免污染仓库。

---

## 综合使用建议

| 场景 | 推荐组合 |
| --- | --- |
| **日常开发** | Rules + MCP（filesystem、git）+ Skills（pdf/xlsx） |
| **复杂 Bug 调试** | TRAE-debugger + Hooks（自动跑测试） + Sub Agent |
| **代码审查** | TRAE-code-review / TRAE-security-review + Rules |
| **CI/CD 改造** | Custom Command `/deploy` + MCP（kubectl） |
| **团队协作** | 共享 `.trae/rules.md` + 共享 `AGENTS.md` |
| **小程序开发** | TRAE-generate-mini-app + Skills |

> 💡 **Tip**：在 Trae 对话框输入 `/init` 可以一键为项目生成 `AGENTS.md` 和推荐规则，再针对性修改即可。

---

> 文档持续更新中，如需补充某项能力的实战示例，请在对话中直接提问。