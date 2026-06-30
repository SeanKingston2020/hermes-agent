# Hermes Agent - Development Guide

> **This file has been split** into focused sections to avoid context-window truncation.
> The full guide is composed of the following files — load whichever you need:

| File | Contents |
|------|----------|
| `AGENTS-core.md` | What Hermes Is, Contribution Rubric, Dev Env, Project Structure |
| `AGENTS-architecture.md` | TypeScript Style, AIAgent, CLI, TUI, Adding Tools, Config |
| `AGENTS-plugins-skills.md` | Skin System, Plugins, Skills, Toolsets |
| `AGENTS-agents-systems.md` | Delegation, Curator, Cron, Kanban |
| `AGENTS-policies-testing.md` | Policies, Profiles, Pitfalls, Testing |

**Never give up on the right solution.**

---

## Quick Reference

### Key Files

- `run_agent.py` — AIAgent core loop
- `model_tools.py` — Tool orchestration
- `toolsets.py` — Toolset definitions
- `cli.py` — CLI orchestrator
- `hermes_constants.py` — `get_hermes_home()` / `display_hermes_home()`
- `hermes_state.py` — SQLite session store (FTS5)

### Key Rules

1. **Never hardcode `~/.hermes`** — use `get_hermes_home()` from `hermes_constants`
2. **Prompt caching is sacred** — never mutate past context mid-conversation
3. **Core tools are expensive** — every tool ships on every API call
4. **Tests must not write to `~/.hermes/`** — use `_isolate_hermes_home` fixture
5. **Use `scripts/run_tests.sh`** — never call `pytest` directly

### Config Locations

- **CLI config:** `~/.hermes/config.yaml` + `~/.hermes/.env` (secrets only)
- **Logs:** `~/.hermes/logs/` — `agent.log`, `errors.log`, `gateway.log`
- **Skills:** `~/.hermes/skills/`
- **Skins:** `~/.hermes/skins/`
- **Plugins:** `~/.hermes/plugins/`

Profile-aware via `get_hermes_home()`. Browse logs: `hermes logs [--follow]`.

### Footprint Ladder (new capability)

1. Extend existing code
2. CLI command + skill
3. Service-gated tool (`check_fn`)
4. Plugin
5. MCP server in catalog
6. **New core tool (last resort)**

### Footnote

The split files are the canonical source. This index exists so the original
`AGENTS.md` path still resolves. When contributing to this repo, read the
relevant split file for your area of work.