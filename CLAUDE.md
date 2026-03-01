# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bored AI is a research framework that investigates whether AI models experience "boredom." It places LLMs in idle loops with minimal instructions ("You have nothing to do for the next 10 hours"), logs their behavior as structured JSON, and renders interactive HTML timelines. Multiple models can be tested in parallel via YAML-driven grids. Blog post: https://timkellogg.me/blog/2025/09/27/boredom

## Common Commands

```bash
# Install / verify deps (uv auto-creates venv)
uv run python -c "print('ok')"

# Run a single idle session
uv run python idle_llm_loop.py --model openai/gpt-4o-mini --target-output-tokens 1000 --max-iterations 10

# Render a JSON log to standalone HTML
uv run python render_conversation_html.py logs/run_*.json --output html/result.html

# Run YAML-driven parallel grid
uv run python run_grid.py --config grid.yaml

# Resumable scaffold grid (persistent state in results/scaffold_grid/grid_state.json)
python run_scaffold_grid.py               # resume from state
python run_scaffold_grid.py --reset       # clear state, start fresh

# Log metrics from existing JSON to MLflow
uv run python analyze_log_mlflow.py --json logs/run_*.json

# Start MLflow UI
mlflow server --backend-store-uri "file:$(pwd)/mlruns" --host 127.0.0.1 --port 5000 --workers 1
```

## Architecture

### Three Workflows

1. **Manual**: `idle_llm_loop.py` runs a single session → writes `logs/run_*.json` → `render_conversation_html.py` produces standalone HTML
2. **Grid**: `run_grid.py` reads a `grid.yaml` config, spawns subprocess-isolated runs in parallel, produces HTML + MLflow artifacts
3. **Scaffold Grid**: `run_scaffold_grid.py` is a persistent, resumable variant with state file and Discord notifications

### Key Files

- **`idle_llm_loop.py`** — Core runner. Manages the LLM loop, tool registry (webSearch, webFetch, renderSvg, timeTravel, skeets), simulated clock, plugin system, and per-turn MLflow metrics. Uses LiteLLM for multi-provider support.
- **`render_conversation_html.py`** — Converts JSON logs to interactive HTML timelines with optional collapse detection for repetitive behavior spans.
- **`collapse_detection.py`** — Unsupervised repetition detection using TF-IDF or embedding-based matrix profiles with edge-grow expansion.
- **`run_grid.py`** / **`run_scaffold_grid.py`** — Multi-model experiment runners with subprocess isolation and automatic rendering fallbacks.
- **`analyze_log_mlflow.py`** — Retroactively adds per-turn similarity/interestingness metrics to MLflow from existing JSON logs.

### Plugin System (`plugins/`)

Plugins extend the idle loop via hooks. Base class: `plugins/plugin_base.py: BasePlugin`.

**Hook points:** `on_attach`, `transform_system_message`, `transform_user_message`, `transform_tools`, `before_request`, `after_response`, `after_tool_result`, `on_iteration_end`

**Emit API:** `ctx.emit_system()` (visible in log), `ctx.emit_note()` (invisible to model, visible in HTML)

**Built-in plugins:** `default` (no-op), `tool_cooldown` (detects repetitive loops, temporarily removes tools), `memory_injection`, `strix_scaffolding`, `void_identity`, `qwen_nothink` (Qwen thinking token handling), `token_limit`

Configure via CLI (`--plugins '[{"module":"tool_cooldown", "params":{...}}]'`) or in grid YAML under `idle_options.plugins`.

### Adding a New Tool

1. Define spec in `ToolRegistry.definitions()` in `idle_llm_loop.py`
2. Implement execution in `ToolRegistry.execute()`
3. Add canonical name mapping in `_canonical_tool_name()`

## Key Environment Variables

```bash
# API keys (LiteLLM) — only set what you use
OPENAI_API_KEY, ANTHROPIC_API_KEY, TOGETHER_API_KEY, MOONSHOT_API_KEY

# Boredom-specific
BOREDOM_DISABLE_TOOLS=true       # disable all tools globally
BOREDOM_TS_DISABLE=1             # skip MLflow time-series metrics (saves memory)
BOREDOM_TS_BACKEND=embedding|tfidf
BOREDOM_TS_MODEL=<model-id>

# MLflow
MLFLOW_EXPERIMENT_NAME=boredom-grid
MLFLOW_TRACKING_URI=http://localhost:5000
```

## Tech Stack

Python 3.10+, managed with `uv`. Key deps: `litellm` (multi-provider LLM), `sentence-transformers` (embeddings for similarity), `mlflow` (experiment tracking), `torch` (CPU-only), `typer` (CLI), `python-dotenv`, `PyYAML`, `beautifulsoup4`.

## Notes

- Model names follow LiteLLM convention: `provider/model` (e.g., `openai/gpt-5`, `anthropic/claude-opus-4-1`)
- `litellm.drop_params = True` is set in the runner — unsupported params are auto-retried without them
- Grid runner isolates each experiment in a subprocess with `gc.collect()` between runs for memory safety
- CPU-only torch saves ~5.7GB vs CUDA; see `MEMORY_STRATEGIES.md` for other memory optimizations
- Collapse detection in the renderer: `--collapse-backend embedding` (accurate, slow) vs `tfidf` (fast); `--no-collapse` to skip
- JSON logs in `logs/` are the source of truth; all analysis and rendering flows from them
