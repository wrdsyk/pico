# Pico

Pico is a small local coding agent for repository-level development tasks. It runs from the command line, builds a compact view of the current workspace, asks a model for one next action at a time, and executes only through a fixed set of guarded tools.

This repository is my personal fork / second-development version of Pico. The focus is not to build a full IDE assistant, but to keep a clear and inspectable agent runtime that is useful for learning, experimentation, and future extensions such as RAG.

## What It Does

- Builds a workspace context from the current repository, branch, git status, recent commits, and key project files.
- Provides a small tool set for listing files, reading files, searching, running shell commands, writing files, and patching files.
- Runs an agent loop: build prompt, call model, parse tool/final output, execute tool, record result, and continue.
- Stores session memory, run traces, task state, and final reports under `.pico/`.
- Supports Ollama, OpenAI-compatible APIs, Anthropic-compatible APIs, and DeepSeek-compatible APIs.
- Includes tests and benchmark fixtures for runtime behavior, memory, safety, context reduction, and evaluation.

## Project Structure

```text
pico/
  cli.py              command-line entry and runtime assembly
  runtime.py          main agent loop, prompt flow, tool execution, checkpoints
  workspace.py        repository context collection
  tools.py            tool registry, validation, and implementations
  context_manager.py  prompt sections, budgets, and context reduction
  memory.py           working memory and durable memory helpers
  models.py           model provider adapters
  run_store.py        run artifacts: task_state, trace, report
  task_state.py       per-request state machine
  evaluator.py        benchmark runner
  metrics.py          benchmark and experiment aggregation

tests/                unit tests and fixture repositories
benchmarks/           benchmark task definitions
scripts/              experiment and metrics scripts
```

## Installation

Pico requires Python 3.10+.

```bash
pip install -e .
```

If you use `uv`:

```bash
uv sync
```

Copy the example environment file and fill only the provider you need:

```bash
cp .env.example .env
```

Do not commit real API keys. `.env` is ignored by git.

## Usage

Start an interactive session in the current repository:

```bash
pico --provider deepseek
```

Run one task directly:

```bash
pico --provider deepseek "inspect the failing tests and propose a fix"
```

Use another workspace:

```bash
pico --cwd /path/to/repo --provider openai
```

Run as a Python module:

```bash
python -m pico --provider ollama
```

## Common Commands

Inside the REPL:

```text
/help      show help
/memory    show distilled working memory
/session   show current session file
/reset     clear current session history and memory
/exit      quit
```

## Safety Model

Pico does not let the model execute arbitrary Python callbacks. The model can only request tools from the registry, and every call goes through runtime validation.

Risky tools such as shell execution and file writes are controlled by approval policy:

```bash
pico --approval ask
pico --approval auto
pico --approval never
```

File tools are anchored inside the workspace root. Tool results are written into session history and run traces so the whole execution can be inspected later.

## Development

Run tests:

```bash
python -m pytest -q
```

Run linting if Ruff is installed:

```bash
ruff check .
```

## Notes

This is a compact code-agent runtime intended for learning and extension. The next natural extension points are repository indexing, RAG-based file retrieval, richer planning state, and stronger tool policies.
