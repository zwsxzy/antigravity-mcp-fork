# antigravity-mcp-fork

A fork of [Kio-pon/antigravity-mcp](https://github.com/Kio-pon/antigravity-mcp) that adds
**GPT-planned native multi-parent orchestration**, **local job supervision**, and
**signed job-event webhooks** on top of the upstream Antigravity IDE bridge.

Upstream exposes 6 tools. This fork exposes **8** — the same 6 plus
`get_agent_job_status` and `supervise_agent_job`.

---

## Provenance

This is a downstream fork, not an independent project.

| | |
| :--- | :--- |
| Upstream | [Kio-pon/antigravity-mcp](https://github.com/Kio-pon/antigravity-mcp) |
| Upstream base commit | `eebf18edac29a72c113c6841d43d494df9acf823` (v1.1.0) |
| Upstream license | MIT, Copyright (c) 2026 Kio-pon |
| Local version | `1.1.0.post1` |

Five of the nine upstream modules are byte-identical to the base commit:
`__init__.py`, `__main__.py`, `discovery.py`, `paths.py`, `protocol.py`.

Four were modified to wire in the added features:

| Module | Upstream lines changed | What changed |
| :--- | ---: | :--- |
| `antigravity_local.py` | ~536 | Native tree tracking; rewrote context-size truncation |
| `server.py` | ~201 | Registers the two added tools; hooks `forest_*` and `workflow_settings` |
| `executor.py` | ~209 | `run_forest` dispatch; model resolution rework |
| `job_manager.py` | ~100 | `RLock` guard and thread-safe `_snapshot()` |

The local version string was bumped to `1.1.0.post1` specifically so this fork can
never be confused with upstream `1.1.0` — both previously reported `1.1.0` while
behaving differently, which made them indistinguishable from a client's point of view.

---

## Added modules

These eight modules do not exist upstream:

| Module | Lines | Responsibility |
| :--- | ---: | :--- |
| `native_lifecycle.py` | 382 | Tracks only native descendant trees created by a validated GPT plan |
| `job_events.py` | 302 | MCP job-attention subscriptions; signed, durable webhook delivery |
| `supervision.py` | 250 | Non-LLM 300-second supervision of registered jobs (`sqlite3`) |
| `forest_executor.py` | 210 | Runs independent native root trees against one pinned local IDE endpoint |
| `native_completion.py` | 153 | Validates a parent's final report against the observed conversation tree |
| `native_plan.py` | 119 | Validates GPT plans before any native worker tree is created |
| `forest_plan.py` | 102 | Validates independent multi-parent plans; requires sibling writes to be disjoint |
| `workflow_settings.py` | 89 | Loads persistent collaboration preferences |

---

## Tools

| Tool | Source |
| :--- | :--- |
| `dispatch_gemini_agent` | upstream |
| `list_available_models` | upstream |
| `check_agent_job` | upstream |
| `list_active_jobs` | upstream |
| `cancel_agent_job` | upstream |
| `gemini_code_search` | upstream |
| `get_agent_job_status` | **this fork** |
| `supervise_agent_job` | **this fork** |

---

## Execution model

The intended loop: **GPT plans and reviews; Gemini executes.**

`workflow_settings.json` (shipped alongside the package) pins the policy:

```json
{
  "default_model": "Gemini 3.8 Flash (High)",
  "planner": "gpt",
  "mode": "native_multi_parent",
  "min_workers_per_parent": 2,
  "min_agents_per_worker": 2,
  "single_parent_conversation": false,
  "same_ide": true,
  "subagent_model": "inherit",
  "independent_tasks_required": true,
  "require_all_terminal_results": true
}
```

Backends are tried in order: the local Antigravity language server, then the Gemini
REST API. If neither is reachable the job fails with an explicit error — there is
deliberately no fallback tier that returns an invented result.

---

## Install

Requires Python 3.9+. Zero runtime dependencies (standard library only).

```bash
pip install -e .
```

Run over stdio:

```bash
python -m antigravity_mcp
```

MCP client configuration:

```json
{
  "mcpServers": {
    "antigravity-mcp": {
      "command": "python",
      "args": ["-m", "antigravity_mcp"],
      "env": { "PYTHONPATH": "D:\\path\\to\\antigravity-mcp-fork\\src" }
    }
  }
}
```

---

## Environment variables

| Variable | Default | Purpose |
| :--- | :--- | :--- |
| `ANTIGRAVITY_STATE_DIR` | platform state dir | Where `jobs.json` and `supervision.sqlite3` live |
| `ANTIGRAVITY_MAX_CONCURRENT_JOBS` | `3` | Jobs running at once; extras queue |
| `ANTIGRAVITY_MAX_CONCURRENT_REQUESTS` | `16` | Tool calls handled in parallel |
| `ANTIGRAVITY_JOB_TIMEOUT` | `1800` | Seconds before a job is abandoned |
| `ANTIGRAVITY_DISABLE_LOCAL` | unset | Skip the local backend entirely |
| `ANTIGRAVITY_EVENT_PROFILE` | unset | Selects the job-event/webhook profile |
| `ANTIGRAVITY_LOG_LEVEL` | `INFO` | Log verbosity (written to stderr) |

---

## Shared state warning

`paths.py` resolves the state directory to `%LOCALAPPDATA%\antigravity-mcp` on Windows
(and the equivalent per-platform location elsewhere). Upstream `antigravity-mcp`
resolves to **the same directory**.

If you install both, they share one `jobs.json`. This fork writes extra
`native_forest*` metadata keys that upstream ignores, and upstream writes records
lacking those keys. Neither crashes today, but it is shared mutable state between
two different codebases. If you run both, pin a separate directory for each:

```bash
set ANTIGRAVITY_STATE_DIR=%LOCALAPPDATA%\antigravity-mcp-fork
```

---

## Status

- Verified on Windows x64 with Python 3.12: the server starts from the fork's own
  `src/` tree, completes the MCP handshake, and lists all 8 tools.
- Upstream notes that only Linux has been verified on real hardware against
  Antigravity IDE v1.107.0.
- No automated test suite is included in this repository yet; upstream's `tests/`
  directory was not carried over because the fork's modified modules diverge from it.

---

## License

MIT. Original work Copyright (c) 2026 Kio-pon, see `LICENSE`.

Fork modifications are distributed under the same MIT terms.