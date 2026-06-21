# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

SkillSpector is a **security scanner for AI agent skills** (Claude Code, Cursor, Codex CLI, Gemini CLI, etc.). It detects vulnerabilities in skill code before installation using a two-stage pipeline: fast static/behavioral analysis followed by optional LLM semantic evaluation. Python 3.12+, built on LangGraph.

## Setup & Common Commands

All commands assume a venv is created and activated first. Prefer `uv` if available.

```bash
# Install dev dependencies
make install-dev          # uv sync --all-extras, or pip install -e ".[dev]"

# Run tests (unit only — no LLM calls, this is the default)
make test-unit            # pytest -m "not integration" tests/
pytest tests/unit/test_cli.py::test_cli_version -v  # single test

# Run integration tests (invoke full graph, may call LLMs)
make test-integration     # pytest -m integration tests/

# Lint and format
make lint                 # ruff check src/ tests/
make format               # ruff check --fix + ruff format

# Run the scanner
skillspector scan ./my-skill/
skillspector scan https://github.com/user/repo --no-llm
skillspector scan ./skill.zip --format json -o report.json

# LangGraph dev server (graph visualization at smith.langchain.com)
make langgraph-dev
```

## Architecture

### LangGraph Workflow (`src/skillspector/graph.py`)

The pipeline is a compiled `StateGraph` on `SkillspectorState`:

```
START → resolve_input → build_context → [20 PARALLEL ANALYZERS] → meta_analyzer → report → END
```

All 20 analyzer nodes fan out from `build_context` and fan in to `meta_analyzer`. The graph is exported as the module-level `graph` object and is the public Python API:

```python
from skillspector import graph
result = graph.invoke({"input_path": "/path/to/skill", "output_format": "json", "use_llm": True})
```

### Shared State (`src/skillspector/state.py`)

`SkillspectorState` (TypedDict) is passed between all nodes. Key fields:
- `skill_path` — resolved local path to the skill being analyzed
- `file_cache` / `ast_cache` — file contents and AST representations, populated by `build_context`
- `components` — list of relative file paths within the skill
- `findings` — uses `operator.add` reducer so each analyzer node simply returns `{"findings": [...]}` and results are automatically accumulated
- `filtered_findings` — post-LLM-filter findings from `meta_analyzer`
- `use_llm` — when `False`, LLM-based nodes (`meta_analyzer`, `mcp_tool_poisoning`'s TP4, `semantic_*`) return immediately without calling the LLM; each node checks this flag itself
- `risk_score` (0–100), `risk_severity`, `sarif_report`, `report_body`

### Analyzer Nodes (`src/skillspector/nodes/analyzers/`)

The 20 analyzers registered in `ANALYZER_NODE_IDS` / `ANALYZER_NODES` fall into four families:

| Family | Node names | Mechanism |
|--------|-----------|-----------|
| **Static patterns** | `static_patterns_*` (11 nodes) | Regex/pattern matching per file via `run_static_patterns()` |
| **Static YARA** | `static_yara` | YARA rule matching against file bytes |
| **Behavioral** | `behavioral_ast`, `behavioral_taint_tracking` | Python AST analysis (exec/eval/subprocess/taint flow) |
| **MCP-specific** | `mcp_least_privilege`, `mcp_tool_poisoning`, `mcp_rug_pull` | MCP manifest analysis |
| **Semantic** | `semantic_security_discovery`, `semantic_developer_intent`, `semantic_quality_policy` | LLM-powered (respect `use_llm`) |

**Adding a new analyzer**: implement `node(state: SkillspectorState) -> AnalyzerNodeResponse`, return `{"findings": [...]}`, register in `ANALYZER_NODE_IDS` and `ANALYZER_NODES` in `nodes/analyzers/__init__.py`. The graph auto-wires it into the fan-out/fan-in.

**Pattern-based analyzers** share `static_runner.py`:
- `run_static_patterns(state, pattern_modules)` iterates `components`, loads from `file_cache`, infers file type, calls `module.analyze(content, file_path, file_type)` returning `list[AnalyzerFinding]`, then converts to `Finding` via `analyzer_finding_to_finding()`
- Files >1 MB and eval datasets (`evals/evals.json` etc.) are skipped
- Pattern metadata (name, category, explanation, remediation) for all 64 rule IDs lives in `pattern_defaults.py`

### LLM Provider System (`src/skillspector/providers/`)

Active provider is selected by `SKILLSPECTOR_PROVIDER` env var (default: `nv_build`). Each provider subpackage exposes credentials, an OpenAI-compatible base URL, token-budget metadata, and per-slot default models via a `model_registry.yaml`.

Model selection waterfall: `SKILLSPECTOR_MODEL` env → per-slot default in provider's registry → provider `DEFAULT_MODEL`.

LLM-using nodes call through `llm_utils.py` which resolves credentials from the active provider and falls back to `OPENAI_API_KEY` as a second tier.

### Risk Scoring

`report.py` computes `risk_score` (0–100) from filtered findings: CRITICAL +50, HIGH +25, MEDIUM +10, LOW +5. Executable scripts apply a 1.3× multiplier. Score bands: 0–20 (LOW/SAFE), 21–50 (MEDIUM/CAUTION), 51–80 (HIGH), 81–100 (CRITICAL).

### Key Configuration (`src/skillspector/constants.py`)

- `MODEL_CONFIG` — dict of model IDs per slot, resolved at import time from the active provider
- `MAX_INPUT_TOKENS_PCT = 0.75` — 75% of context used for input, 25% reserved for output
- `SKILLSPECTOR_LOG_LEVEL` — from env, defaults to `WARNING`

## Environment Variables

Copy `.env.example` to `.env`. Key variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `SKILLSPECTOR_PROVIDER` | `nv_build` | Active LLM provider (`openai`, `anthropic`, `nv_build`) |
| `NVIDIA_INFERENCE_KEY` | — | Credential for `nv_build` (default provider) |
| `OPENAI_API_KEY` | — | Credential for `openai`; also tier-2 fallback for other providers |
| `OPENAI_BASE_URL` | api.openai.com | Override to use Ollama/vLLM/other OpenAI-compatible endpoints |
| `ANTHROPIC_API_KEY` | — | Credential for `anthropic` |
| `SKILLSPECTOR_MODEL` | provider default | Override the model for all slots |
| `SKILLSPECTOR_LOG_LEVEL` | `WARNING` | Internal log level |
| `LANGCHAIN_TRACING_V2` | `false` | Enable LangSmith tracing |

## Testing Conventions

- Tests are split into `tests/unit/` (no LLM), `tests/integration/` (full graph), `tests/nodes/` (node-level), and `tests/nodes/analyzers/` (analyzer-level)
- Integration tests are marked `@pytest.mark.integration`; `addopts = "-m 'not integration'"` in `pyproject.toml` means `pytest` runs unit tests only by default
- `conftest.py` provides `safe_skill_dir` / `malicious_skill_dir` fixtures (tmp_path-based) and an **autouse** `mock_resolve_context_length` fixture that patches `skillspector.model_info._resolve_context_length` to `131_072` so no network calls are made in any test
- Async tests use `asyncio_mode = "auto"` — no manual `@pytest.mark.asyncio` needed

## Code Style

- Ruff (line length 100, target py312) is the sole linter/formatter. Rules: `E, F, W, I, N, UP, B, C4`; E501 (line-too-long) is ignored
- mypy strict: `disallow_untyped_defs = true`, `warn_return_any = true`
- Pre-commit hooks run ruff check + ruff format automatically
- All source files carry the NVIDIA SPDX copyright header
