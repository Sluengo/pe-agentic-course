# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Hands-on exercise code for "Agentic AI in Platform Engineering" — a PlatformEngineering.org course. Each `moduleN/` directory is a self-contained exercise. Students implement `agent.py` (or the named script) against a scaffold, guided by the module's `README.md`.

## Running Exercises

```bash
# Verify environment before starting any module
python shared/verify_setup.py

# Every module supports mock mode (no API key required)
python moduleN/agent.py --mock
MOCK_MODE=1 python moduleN/agent.py

# Live run (requires ANTHROPIC_API_KEY)
export ANTHROPIC_API_KEY=sk-ant-...
python moduleN/agent.py

# Guided walkthrough scripts (run from repo root)
bash runners/moduleN.sh
```

Output JSON is saved to `output/output_moduleN.json` automatically by each agent.

## Architecture

### The `ask()` Function — All Claude Calls Go Here

`shared/claude_client.py` is the single API entry point. Students call `ask()` — never the Anthropic SDK directly.

```python
from shared.claude_client import ask

result = ask(
    system="...",   # agent role + output schema
    user="...",     # data to analyse
    model="claude-opus-4-5-20251101",  # default
    max_tokens=1024,
)
# Always returns a parsed dict — raises ValueError on JSON parse failure
```

`ask()` handles: markdown code fence stripping, literal-newline sanitisation inside JSON strings, and lazy client initialization.

### Output Utilities — `shared/output.py`

Three formatters used by every module agent:
- `save_json(result, module)` — writes to `output/output_moduleN.json`
- `to_step_summary(result, title)` — GitHub Actions Step Summary markdown; auto-writes to `$GITHUB_STEP_SUMMARY` in CI
- `to_github_issue(result, module)` — issue body for escalation events

### Module Structure Pattern

Every module follows the same layout:

| File | Role |
|------|------|
| `agent.py` | Exercise entry point with `MOCK_RESPONSE`, `SYSTEM_PROMPT`, `AGENT_CONFIG`, and `run_agent()` with a `TODO` block for students |
| `sample_data.json` / `sample_log.txt` | Realistic test data fed to the agent |
| `agent-config.yml` | Model, max_tokens, max_iterations, context_fields |
| `solutions/solution.py` | Reference implementation — read after your own attempt |

The `AGENT_CONFIG` dict in each module file controls the model and token budget. Override it to experiment.

### Agent Patterns by Module

| Module | Pattern | Key file(s) |
|--------|---------|------------|
| 1–2 | Single-shot: one `ask()` call, one JSON response | `agent.py` |
| 3 | ReAct loop: iterates up to `max_iterations`, accumulates `history`, breaks on `finished=true` | `agent.py` |
| 4–5 | Event-driven triage / quality gate | `triage_agent.py` |
| 6 | Conversational agent: Phase 1 routes with cheap model (Haiku, 64 tokens), Phase 2 executes with full model | `conversational_agent.py` |
| 7 | Orchestrator + two parallel specialists via `ThreadPoolExecutor`; Safety First conflict rule | `orchestrator.py` |
| 8 | Five-step pipeline combining all prior patterns; DIAGNOSE + GATE run in parallel (Steps 2–3) | `platform_agent.py` |

### Model Selection Guidelines

- Default for exercises: `claude-opus-4-5-20251101`
- Routing/classification calls: `claude-sonnet-4-5-20250929`
- High-volume or very short calls (classifier, Phase 1 route): `claude-haiku-4-5-20251001` with `max_tokens=64`
- Always use the full model string including the date suffix.

### Token Budget by Agent Type

| Agent type | max_tokens |
|-----------|-----------|
| Query classifier / route | 64 |
| Single-shot triage | 512 |
| ReAct iteration | 1024 |
| Gate / rollback agent | 1024 |
| Pipeline step with fix script | 4096 |

If you see `JSONDecodeError: Unterminated string`, raise `max_tokens` — the response was cut off.

### ReAct Loop Pattern (Module 3+)

```python
history = []
for i in range(AGENT_CONFIG["max_iterations"]):
    user_msg = f"Context:\n{context}" if i == 0 else \
               f"Context:\n{context}\n\nPrevious iterations:\n{json.dumps(history, indent=2)}"
    result = ask(system=SYSTEM_PROMPT, user=user_msg, ...)
    history.append(result)
    if result.get("finished"):
        break
```

Pass the full `history` list back on each iteration (not just the last step). The final answer is `history[-1]`.

### Multi-Agent Conflict Rule (Modules 7–8)

Safety First: when a Gate Agent says APPROVE and a Rollback Agent says IMMEDIATE, the rollback wins. This rule is encoded in `detect_conflict()` in both `module7/orchestrator.py` and `module8/platform_agent.py`.

## GitHub Actions

Each module has a workflow in `.github/workflows/moduleN-*.yml` that triggers on push to `moduleN/**` or `shared/**`. To enable CI, add `ANTHROPIC_API_KEY` as a repository secret (Settings → Secrets and variables → Actions).
