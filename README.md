# Pico

Pico 是一个轻量级本地 Code Agent 项目，面向代码仓库里的开发、排查和修改任务。它不是简单地把大模型接到几个工具上，而是围绕一次真实代码任务，组织出一条可控的执行链：构建仓库上下文、拼装提示词、调用模型、解析工具请求、执行工具、记录状态、更新记忆，并在需要时生成可追踪的运行工件。

这个仓库是我的个人二开版本，主要用于学习和展示 Code Agent 的核心设计。项目刻意保持简洁，方便理解 agent runtime 的基本结构，也方便后续继续扩展，比如加入 RAG、仓库索引、更强的任务规划和更细的工具权限控制。

## 项目能做什么

- 自动采集当前仓库的基础上下文，包括目录、分支、Git 状态、最近提交和关键项目文件。
- 向模型暴露一组受控工具，包括文件列表、文件读取、代码搜索、命令执行、文件写入和精确 patch。
- 通过主循环驱动任务推进：组装 prompt、调用模型、解析输出、执行工具、记录结果，再进入下一轮。
- 使用结构化 memory 保存当前任务、最近文件、文件摘要和少量跨轮笔记，减少重复读取。
- 将每次运行的 `task_state`、`trace`、`report` 保存到本地 `.pico/`，方便复盘和调试。
- 支持 Ollama、OpenAI-compatible、Anthropic-compatible 和 DeepSeek-compatible 模型后端。
- 提供测试和 benchmark，用于验证运行时、工具边界、上下文压缩、记忆和评测链路。

## 项目结构

```text
pico/
  cli.py              命令行入口，负责参数解析和 agent 装配
  runtime.py          核心运行时，包含主循环、工具执行、checkpoint 和 report
  workspace.py        仓库上下文采集，生成稳定的 workspace 摘要
  tools.py            工具注册、参数校验和具体工具实现
  context_manager.py  prompt 分段、预算控制和上下文压缩
  memory.py           工作记忆、文件摘要和持久记忆
  models.py           模型后端适配层
  run_store.py        运行工件落盘：task_state、trace、report
  task_state.py       单次请求的状态机
  evaluator.py        benchmark 执行框架
  metrics.py          指标统计和实验汇总

tests/                单元测试和 fixture 仓库
benchmarks/           benchmark 任务定义
scripts/              实验和指标脚本
```

## 核心设计思路

Pico 的核心不是“模型会调用工具”，而是让模型的每一步行动都进入一个可控 runtime。

一条用户请求进入 Pico 后，大致会经过这些阶段：

1. CLI 解析参数，创建模型客户端、仓库上下文、会话存储和 Pico 实例。
2. Pico 先构建稳定 prefix，包括 agent 规则、工具清单和仓库摘要。
3. 每轮模型调用前，`ContextManager` 会把 prefix、memory、相关记忆、历史记录和当前请求拼成最终 prompt。
4. 如果 prompt 超过预算，会优先压缩相关记忆和历史，当前用户请求始终保留。
5. 模型只能返回一个 `<tool>...</tool>` 或 `<final>...</final>`。
6. 工具调用会统一经过参数校验、路径边界检查、重复调用拦截和审批策略。
7. 工具结果会写入 history、trace、task_state，并把高价值信息沉淀到 memory。
8. 如果模型返回 final，Pico 写出 report 并结束本次任务；如果超过步数或重试上限，则按明确原因停止。

这套设计让一次代码任务不再是一次性聊天，而是一条可审计、可恢复、可评测的执行流程。

## 安装

需要 Python 3.10+。

```bash
pip install -e .
```

如果使用 `uv`：

```bash
uv sync
```

复制环境变量模板：

```bash
cp .env.example .env
```

把需要使用的模型服务 key 填进 `.env`。真实 key 不要提交到仓库，`.env` 已经被 `.gitignore` 忽略。

## 使用方式

在当前仓库启动交互模式：

```bash
pico --provider deepseek
```

直接执行一次任务：

```bash
pico --provider deepseek "检查测试失败原因并给出修复方案"
```

指定另一个工作目录：

```bash
pico --cwd /path/to/repo --provider openai
```

也可以用模块方式启动：

```bash
python -m pico --provider ollama
```

## REPL 内置命令

```text
/help      查看帮助
/memory    查看当前提炼后的工作记忆
/session   查看当前会话文件路径
/reset     清空当前会话历史和记忆
/exit      退出
```

## 工具与安全边界

Pico 不允许模型直接执行任意 Python 函数。模型只能申请调用工具注册表里的工具，并且所有工具调用都必须经过 runtime 校验。

高风险工具，比如 shell 命令和文件写入，会受审批策略控制：

```bash
pico --approval ask
pico --approval auto
pico --approval never
```

文件类工具会被限制在 workspace root 之内，避免路径逃逸。每次工具调用的参数、结果、状态和可能影响到的文件都会进入 trace 或 report，便于后续复盘。

## 开发与测试

运行测试：

```bash
python -m pytest -q
```

如果安装了 Ruff，可以运行：

```bash
ruff check .
```

## 后续扩展方向

- 加入 RAG 或仓库索引，让 agent 更快定位相关文件。
- 增强任务规划状态，比如 `current_plan`、`open_questions`、`confirmed_findings`。
- 增加更细粒度的工具权限策略。
- 扩展 workspace context，加入目录结构摘要、语言分布和 Git diff 摘要。
- 提供更完整的 Web UI 或可视化 trace 查看器。
