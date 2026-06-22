# Hermes Agent - Development Guide

Instructions for AI coding assistants and developers working on the hermes-agent codebase.

**Never give up on the right solution.**

## What Hermes Is

Hermes is a personal AI agent that runs the same core across a CLI, messaging gateway (20+ platforms), TUI, and Electron desktop app. It learns across sessions (memory + skills), delegates to subagents, runs scheduled jobs, and drives a terminal/browser. Extended via **plugins and skills**, not by growing the core.

Two design principles:

1. **Prompt caching is sacred.** Anything that mutates past context, swaps toolsets, or rebuilds the system prompt mid-conversation invalidates the cache. We don't do it (exception: context compression).
2. **Core is narrow; capability lives at edges.** Every model tool ships on every API call. Prefer: CLI command + skill → service-gated tool → plugin → MCP server → new core tool (last resort).

---

## Contribution Rubric

### What we want

- **Fix real bugs, well.** Reproduce on current `main`, point to the exact line, fix the whole bug class (sibling call paths included).
- **Expand reach at the edges.** New platforms, channels, providers, models, desktop/TUI features land routinely. Integrate via `hermes tools`/`hermes setup`/auto-install, not raw env vars.
- **Refactor god-files.** Extracting multi-thousand-line clusters from `cli.py`/`run_agent.py`/`gateway/run.py` into focused mixins/modules is wanted work.
- **Keep the core narrow.** See "The Footprint Ladder."
- **Extend, don't duplicate.** When several PRs integrate the same category, design one shared interface (ABC + orchestrator).
- **Behavior contracts over snapshots.** Tests assert invariants, not frozen values (model lists, config versions, enumeration counts).
- **E2E validation.** Exercise real resolution chains with actual imports against a temp `HERMES_HOME`.
- **Cache-, alternation-, invariant-safe.** Preserve prompt caching, strict role alternation (no two same-role messages), byte-stable system prompt.
- **Contributor credit preserved.** Cherry-pick external work to preserve authorship.

### What we don't want

- **Speculative infrastructure.** Hooks/callbacks with no concrete consumer.
- **New `HERMES_*` env vars for non-secret config.** `.env` is for secrets only. Behavioral settings go in `config.yaml`.
- **A new core tool when terminal + file already do the job.**
- **Lazy-reading escape hatches on instructional tools.** No `offset`/`limit` pagination on skills/prompts/playbooks.
- **"Fixes" that destroy the feature they secure.**
- **Outbound telemetry without opt-in gating.**
- **Change-detector tests, cache-breaking mid-conversation, dead code without E2E proof, plugins that touch core files.**

### Verify the premise

Before "fixing" something, verify:
1. **"Intentional design, not a gap."** Read the original commit's intent (`git log -p -S "<symbol>"`).
2. **"The premise doesn't hold."** Trace the real code/runtime. Point to the exact line where the bug manifests.
3. **"Absence was deliberate."** Adding the obvious-looking missing piece can break things.
4. **"Overreached."** Keep the change to the narrow piece agreed; offer the rest as follow-up.

---

## The Footprint Ladder

Each rung adds more permanent surface. Choose the highest (least-footprint) rung that solves the problem:

1. **Extend existing code** — Zero new surface.
2. **CLI command + skill** — Zero model-tool footprint. Default for subscriptions, scheduled tasks, service setup.
3. **Service-gated tool (`check_fn`)** — Zero footprint otherwise (appears only when prerequisite configured).
4. **Plugin** — Third-party/niche capability in `~/.hermes/plugins/` or pip package.
5. **MCP server (in catalog)** — Zero permanent core-schema footprint.
6. **New core tool** — Only when fundamental, broadly useful, and unreachable via terminal + file.

When 3+ open PRs try to integrate the same category, design one ABC + orchestrator.

---

## Development Environment

```bash
source .venv/bin/activate   # or: source venv/bin/activate
scripts/run_tests.sh        # full suite, CI-parity
```

---

## Project Structure

```
hermes-agent/
├── run_agent.py          # AIAgent class — core conversation loop (~12k LOC)
├── model_tools.py        # Tool orchestration, discover_builtin_tools(), handle_function_call()
├── toolsets.py           # Toolset definitions, _HERMES_CORE_TOOLS list
├── cli.py                # HermesCLI class — interactive CLI (~11k LOC)
├── hermes_state.py       # SessionDB — SQLite session store (FTS5 search)
├── hermes_constants.py   # get_hermes_home(), display_hermes_home()
├── hermes_logging.py     # setup_logging()
├── batch_runner.py       # Parallel batch processing
├── agent/                # Provider adapters, memory, caching, compression, etc.
├── hermes_cli/           # CLI subcommands, setup wizard, plugins loader, skin engine
├── tools/                # Tool implementations — auto-discovered via tools/registry.py
│   └── environments/     # Terminal backends (local, docker, ssh, modal, daytona, singularity)
├── gateway/              # Messaging gateway — run.py + session.py + platforms/
│   └── platforms/        # Adapter per platform (see ADDING_A_PLATFORM.md)
├── plugins/              # Plugin system (general, memory, model-providers, kanban, etc.)
├── optional-skills/      # Heavier/niche skills shipped but NOT active by default
├── skills/               # Built-in skills bundled with the repo
├── ui-tui/               # Ink (React) terminal UI — `hermes --tui`
├── tui_gateway/          # Python JSON-RPC backend for the TUI
├── acp_adapter/          # ACP server (VS Code / Zed / JetBrains integration)
├── cron/                 # Scheduler — jobs.py, scheduler.py
├── scripts/              # run_tests.sh, release.py
├── website/              # Docusaurus docs site
└── tests/                # Pytest suite (~17k tests across ~900 files)
```

**User config:** `~/.hermes/config.yaml` (settings), `~/.hermes/.env` (API keys).
**Logs:** `~/.hermes/logs/` — `agent.log` (INFO+), `errors.log` (WARNING+), `gateway.log`.

---

## TypeScript Style

- Prefer small nanostores over component state when state is shared/reused.
- Let each feature own its atoms. Components that render from an atom use `useStore`.
- No monolithic hooks. A hook should own one narrow job.
- Prefer colocated action modules over hidden god hooks.
- Prefer interfaces for public props. Extend React primitives: `React.ComponentProps<'button'>`.
- `src/app` owns routes/pages. `src/store` owns shared atoms. `src/lib` owns shared pure helpers.

---

## File Dependency Chain

```
tools/registry.py  (no deps — imported by all tool files)
       ↑
tools/*.py  (each calls registry.register() at import time)
       ↑
model_tools.py  (imports tools/registry + triggers tool discovery)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

---

## AIAgent Class (run_agent.py)

```python
class AIAgent:
    def __init__(self,
        base_url: str = None, api_key: str = None, provider: str = None,
        api_mode: str = None, model: str = "", max_iterations: int = 90,
        enabled_toolsets: list = None, disabled_toolsets: list = None,
        platform: str = None, session_id: str = None, ...): ...

    def chat(self, message: str) -> str: ...
    def run_conversation(self, user_message: str, system_message: str = None,
                         conversation_history: list = None, task_id: str = None) -> dict: ...
```

### Agent Loop

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) or self._budget_grace_call:
    if self._interrupt_requested: break
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

Messages follow OpenAI format: `{"role": "system/user/assistant/tool", ...}`.

---

## CLI Architecture (cli.py)

- **Rich** for banner/panels, **prompt_toolkit** for input with autocomplete
- **KawaiiSpinner** — animated faces during API calls
- `load_cli_config()` merges hardcoded defaults + user config YAML
- **Skin engine** (`hermes_cli/skin_engine.py`) — data-driven CLI theming
- Skill slash commands injected as **user message** (not system prompt) to preserve prompt caching

### Slash Command Registry (`hermes_cli/commands.py`)

All slash commands defined in `COMMAND_REGISTRY` list of `CommandDef` objects. Every downstream consumer derives from this registry automatically.

**Adding a slash command:**
1. Add `CommandDef` to `COMMAND_REGISTRY` in `hermes_cli/commands.py`
2. Add handler in `HermesCLI.process_command()` in `cli.py`
3. (Optional) Add gateway handler in `gateway/run.py`

---

## TUI Architecture (ui-tui + tui_gateway)

The TUI is a full replacement for the classic CLI, activated via `hermes --tui` or `HERMES_TUI=1`.

### Process Model

```
hermes --tui
  └─ Node (Ink)  ──stdio JSON-RPC──  Python (tui_gateway)
       │                                  └─ AIAgent + tools + sessions
       └─ renders transcript, composer, prompts, activity
```

TypeScript owns the screen. Python owns sessions, tools, model calls, and slash command logic.

### TUI in the Dashboard (`hermes dashboard` → `/chat`)

The dashboard embeds the real `hermes --tui` — **not** a rewrite. See `hermes_cli/pty_bridge.py`.

**Rule:** Do not re-implement the primary chat experience in React. The main transcript, composer/input flow, and PTY-backed terminal belong to the embedded `hermes --tui`.

### Electron Desktop Chat App (`apps/desktop/`)

A **separate** chat surface. Electron + React + nanostore renderer that talks to `tui_gateway` backend over JSON-RPC.

**Slash commands in the desktop app:** Curated client-side via `apps/desktop/src/lib/desktop-slash-commands.ts`, then dispatched to the backend. The curation is about hiding noise (terminal-only/messaging-only built-ins), NOT about hiding user-activated extensions. Skill commands belong in completions.

---

## Adding New Tools

**Footprint first:** Most capabilities should NOT be core tools. Use the plugin route for custom/local-only tools.

**Built-in/core tools require changes in 2 files:**

**1. Create `tools/your_tool.py`:**
```python
import json, os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(param=args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

**2. Add to `toolsets.py`** — either `_HERMES_CORE_TOOLS` or a new toolset.

Auto-discovery: any `tools/*.py` file with a top-level `registry.register()` call is imported automatically. Wiring into a toolset is still a deliberate step.

All handlers MUST return a JSON string. Use `get_hermes_home()` for paths.

---

## Dependency Pinning Policy

All dependencies must have upper bounds.

| Source type | Treatment | Example |
|---|---|---|
| PyPI package | `>=floor,<next_major` | `"httpx>=0.28.1,<1"` |
| Git URL | Commit SHA | `git+https://...@<40-char-sha>` |
| GitHub Actions | Commit SHA + comment | `uses: actions/checkout@<sha>  # v4` |

When adding a new dependency to `pyproject.toml`:
1. Pin to `>=current_version,<next_major` for post-1.0.
2. For pre-1.0 packages, use `<0.(current_minor + 2)`.
3. Run `uv lock` to regenerate `uv.lock` with hashes.

---

## Adding Configuration

### config.yaml options:
1. Add to `DEFAULT_CONFIG` in `hermes_cli/config.py`
2. Bump `_config_version` ONLY if you need to actively migrate/transform existing user config.

### .env variables (SECRETS ONLY):
1. Add to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py` with metadata.

### Config loaders (three paths):
| Loader | Used by | Location |
|--------|---------|----------|
| `load_cli_config()` | CLI mode | `cli.py` |
| `load_config()` | `hermes tools`, `hermes setup`, most CLI subcommands | `hermes_cli/config.py` |
| Direct YAML load | Gateway runtime | `gateway/run.py` + `gateway/config.py` |

### Working directory:
- **CLI** — uses the process's current directory (`os.getcwd()`).
- **Messaging** — uses `terminal.cwd` from `config.yaml` (bridged to `TERMINAL_CWD` env var).

---

## Skin/Theme System

The skin engine (`hermes_cli/skin_engine.py`) provides data-driven CLI visual customization. Skins are **pure data**.

### What skins customize
- Banner/response box borders, colors
- Spinner faces/verbs/wings
- Tool prefix, per-tool emojis
- Branding (agent name, welcome message, prompt symbol)

### Built-in skins
- `default` — Classic Hermes gold/kawaii
- `ares` — Crimson/bronze war-god theme
- `mono` — Clean grayscale monochrome
- `slate` — Cool blue developer-focused theme

Activate with `/skin <name>` or `display.skin: <name>` in config.yaml.

---

## Plugins

Hermes has two plugin surfaces. Both live under `plugins/` in the repo.

### General plugins (`hermes_cli/plugins.py` + `plugins/<name>/`)

`PluginManager` discovers plugins from `~/.hermes/plugins/`, `./.hermes/plugins/`, and pip entry points. Each plugin exposes a `register(ctx)` function that can:
- Register Python-callback lifecycle hooks: `pre_tool_call`, `post_tool_call`, `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`
- Register new tools via `ctx.register_tool(...)`
- Register CLI subcommands via `ctx.register_cli_command(...)`

**Discovery timing pitfall:** `discover_plugins()` only runs as a side effect of importing `model_tools.py`.

### Memory-provider plugins (`plugins/memory/<name>/`)

Separate discovery system for pluggable memory backends. Built-in providers: **honcho, mem0, supermemory, byterover, hindsight, holographic, openviking, retaindb**.

**Rule:** plugins MUST NOT modify core files. If a plugin needs a capability the framework doesn't expose, expand the generic plugin surface.

**No new in-tree memory providers (May 2026):** New memory backends must ship as **standalone plugin repos**. PRs that add a new directory under `plugins/memory/` will be closed.

### Model-provider plugins (`plugins/model-providers/<name>/`)

Every inference backend ships as a plugin. `providers/__init__.py._discover_providers()` is a **lazy, separate discovery system** — scanned on first `get_provider_profile()` or `list_providers()` call.

Scan order: 1. Bundled, 2. User (`$HERMES_HOME/plugins/model-providers/`), 3. Legacy (`providers/<name>.py`).

---

## Skills

Two parallel surfaces:
- **`skills/`** — built-in skills shipped and loadable by default.
- **`optional-skills/`** — heavier or niche skills shipped but NOT active by default. Installed via `hermes skills install official/<category>/<skill>`.

### SKILL.md frontmatter

Standard fields: `name`, `description`, `version`, `author`, `license`, `platforms`, `metadata.hermes.*`.

### Skill authoring standards

1. **`description` ≤ 60 characters, one sentence, ends with a period.**
2. **Tools referenced must be native Hermes tools or MCP servers.** Point at proper tools by name (`` `terminal` ``, `` `read_file` ``, `` `patch` ``, etc.).
3. **`platforms:` gating audited against actual script imports.**
4. **`author` credits the human contributor first.**
5. **Modern section order:** Title, intro, `## When to Use`, `## Prerequisites`, `## How to Run`, `## Quick Reference`, `## Procedure`, `## Pitfalls`, `## Verification`.
6. **Scripts go in `scripts/`, references in `references/`, templates in `templates/`.**
7. **Tests live at `tests/skills/test_<skill>_skill.py`.**

---

## Toolsets

All toolsets defined in `toolsets.py` as a single `TOOLSETS` dict.

Current keys: `browser`, `clarify`, `code_execution`, `cronjob`, `debugging`, `delegation`, `discord`, `discord_admin`, `feishu_doc`, `feishu_drive`, `file`, `homeassistant`, `image_gen`, `kanban`, `memory`, `messaging`, `moa`, `rl`, `safe`, `search`, `session_search`, `skills`, `spotify`, `terminal`, `todo`, `tts`, `video`, `vision`, `web`, `yuanbao`.

Enable/disable per platform via `hermes tools` or `tools.<platform>.enabled`/`disabled` in `config.yaml`.

---

## Delegation (`delegate_task`)

`tools/delegate_tool.py` spawns a subagent with isolated context + terminal session.

**Two shapes:**
- **Single:** pass `goal` (+ optional `context`, `toolsets`).
- **Batch (parallel):** pass `tasks: [...]` — concurrency capped by `delegation.max_concurrent_children` (default 3).

**Roles:**
- `role="leaf"` (default) — focused worker. Cannot call `delegate_task`, `clarify`, `memory`, `send_message`, `execute_code`.
- `role="orchestrator"` — retains `delegate_task`, gated by `delegation.orchestrator_enabled` (default true), bounded by `delegation.max_spawn_depth` (default 2).

**Durability rule:** background `delegate_task` is process-local. For work that must survive process restart, use `cronjob` or `terminal(background=True, notify_on_complete=True)`.

---

## Curator (skill lifecycle)

Background skill-maintenance system that tracks usage on agent-created skills and auto-archives stale ones.

- **Core:** `agent/curator.py` + `agent/curator_backup.py`
- **CLI:** `hermes curator <verb>` — `status`, `run`, `pause`, `resume`, `pin`, `unpin`, `archive`, `restore`, `prune`, `backup`, `rollback`
- **Telemetry:** `~/.hermes/skills/.usage.json`

**Invariants:**
- Only touches skills with `created_by: "agent"` provenance.
- Never deletes; max destructive action is archive.
- Pinned skills are exempt from every auto-transition and LLM review pass.

Config: `curator:` in `config.yaml` (`enabled`, `interval_hours`, `min_idle_hours`, `stale_after_days`, `archive_after_days`, `backup.*`).

---

## Cron (scheduled jobs)

`cron/jobs.py` (job store) + `cron/scheduler.py` (tick loop).

**Supported schedule formats:**
- Duration: `"30m"`, `"2h"`, `"1d"`
- "every" phrase: `"every 2h"`, `"every monday 9am"`
- 5-field cron: `"0 9 * * *"`
- ISO timestamp (one-shot): `"2026-06-01T09:00:00Z"`

**Hardening invariants:**
- **3-minute hard interrupt** on cron sessions.
- File lock at `~/.hermes/cron/.tick.lock` prevents duplicate ticks.
- Cron sessions pass `skip_memory=True` by default.

---

## Kanban (multi-agent work queue)

Durable SQLite-backed board for multi-profile/worker collaboration.

- **CLI:** `hermes kanban <verb>` — `init`, `create`, `list`, `show`, `assign`, `link`, `unlink`, `comment`, `complete`, `block`, `unblock`, `archive`, `tail`, `watch`, `stats`, `runs`, `log`, `dispatch`, `daemon`, `gc`
- **Worker toolset:** `kanban_show`, `kanban_complete`, `kanban_block`, `kanban_heartbeat`, `kanban_comment`, `kanban_create`, `kanban_link`
- **Dispatcher:** long-lived loop (default every 60s), runs inside the gateway by default via `kanban.dispatch_in_gateway: true`

**Isolation model:** Board is the hard boundary (workers spawned with `HERMES_KANBAN_BOARD` pinned). Tenant is a soft namespace within a board.

---

## Important Policies

### Prompt Caching Must Not Break

**Do NOT implement changes that would:**
- Alter past context mid-conversation
- Change toolsets mid-conversation
- Reload memories or rebuild system prompts mid-conversation

Slash commands that mutate system-prompt state must be **cache-aware**: default to deferred invalidation (change takes effect next session), with an opt-in `--now` flag.

### Background Process Notifications (Gateway)

Control verbosity with `display.background_process_notifications` in config.yaml:
- `all` — running-output updates + final message (default)
- `result` — only the final completion message
- `error` — only the final message when exit code != 0
- `off` — no watcher messages at all

---

## Profiles: Multi-Instance Support

Hermes supports **profiles** — multiple fully isolated instances, each with its own `HERMES_HOME` directory.

Core mechanism: `_apply_profile_override()` in `hermes_cli/main.py` sets `HERMES_HOME` before any module imports.

### Rules for profile-safe code

1. **Use `get_hermes_home()` for all HERMES_HOME paths.** Import from `hermes_constants`. NEVER hardcode `~/.hermes`.
2. **Use `display_hermes_home()` for user-facing messages.**
3. **Module-level constants are fine** — they cache `get_hermes_home()` at import time (after `_apply_profile_override()`).
4. **Tests that mock `Path.home()` must also set `HERMES_HOME`**.
5. **Gateway platform adapters should use token locks** — call `acquire_scoped_lock()` in `connect()`/`start()`, `release_scoped_lock()` in `disconnect()`/`stop()`.
6. **Profile operations are HOME-anchored, not HERMES_HOME-anchored** — `_get_profiles_root()` returns `Path.home() / ".hermes" / "profiles"`.

---

## Known Pitfalls

### DO NOT hardcode `~/.hermes` paths
Use `get_hermes_home()` for code paths. Use `display_hermes_home()` for user-facing print/log messages.

### DO NOT introduce new `simple_term_menu` usage
The preferred UI is curses (stdlib) because `simple_term_menu` has ghost-duplication rendering bugs in tmux/iTerm2.

### DO NOT use `\033[K` (ANSI erase-to-EOL) in spinner/display code
Leaves as literal `?[K` text under `prompt_toolkit`'s `patch_stdout`. Use space-padding.

### `_last_resolved_tool_names` is a process-global in `model_tools.py`
`_run_single_child()` in `delegate_tool.py` saves and restores this global around subagent execution.

### DO NOT hardcode cross-tool references in schema descriptions
Tool schema descriptions must not mention tools from other toolsets by name. If a cross-reference is needed, add it dynamically in `get_tool_definitions()` in `model_tools.py`.

### The gateway has TWO message guards
(1) **base adapter** queues messages in `_pending_messages` when agent is running. (2) **gateway runner** intercepts `/stop`, `/new`, `/queue`, `/status`, `/approve`, `/deny`. Any new command that must reach the runner while the agent is blocked MUST bypass BOTH guards and be dispatched inline.

### Squash merges from stale branches silently revert recent fixes
Before squash-merging a PR, ensure the branch is up to date with `main`. Verify with `git diff HEAD~1..HEAD` after merging.

### Don't wire in dead code without E2E validation
Before wiring an unused module into a live code path, E2E test the real resolution chain with actual imports against a temp `HERMES_HOME`.

### Tests must not write to `~/.hermes/`
The `_isolate_hermes_home` autouse fixture in `tests/conftest.py` redirects `HERMES_HOME` to a temp dir.

---

## Testing

**ALWAYS use `scripts/run_tests.sh`** — do not call `pytest` directly.

```bash
scripts/run_tests.sh                                  # full suite, CI-parity
scripts/run_tests.sh tests/gateway/                   # one directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # one test
scripts/run_tests.sh -v --tb=long                     # pass-through pytest flags
scripts/run_tests.sh --no-isolate tests/foo/          # disable subprocess isolation (faster, for debugging)
```

### Subprocess-per-test isolation

Every test runs in a freshly-spawned Python subprocess via the in-tree plugin at `tests/_isolate_plugin.py`.

- The plugin uses `multiprocessing.get_context("spawn")`.
- Per-test overhead is ~0.5–1.0s.
- `isolate_timeout` caps each test at 30s.
- Pass `--no-isolate` to disable isolation.

### Don't write change-detector tests

A test is a **change-detector** if it fails whenever data that is **expected to change** gets updated — model catalogs, config version numbers, enumeration counts.

**Do not write:**
```python
assert "gemini-2.5-pro" in _PROVIDER_MODELS["gemini"]
assert DEFAULT_CONFIG["_config_version"] == 21
assert len(_PROVIDER_MODELS["huggingface"]) == 8
```

**Do write:**
```python
assert "gemini" in _PROVIDER_MODELS
assert len(_PROVIDER_MODELS["gemini"]) >= 1
assert raw["_config_version"] == DEFAULT_CONFIG["_config_version"]
```

The rule: if the test reads like a snapshot of current data, delete it. If it reads like a contract about how two pieces of data must relate, keep it.

---

## Commit Conventions

```
type: concise subject line

Optional body.
```

Types: `fix:`, `feat:`, `refactor:`, `docs:`, `chore:`

---

## Where to Find Things

| Looking for... | Location |
|---|---|
| Config options | `hermes config edit` or [Configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) |
| Available tools | `hermes tools list` or [Tools reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) |
| Slash commands | `/help` in session or [Slash commands reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) |
| Skills catalog | `hermes skills browse` or [Skills catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog) |
| Provider setup | `hermes model` or [Providers guide](https://hermes-agent.nousresearch.com/docs/integrations/providers) |
| Platform setup | `hermes gateway setup` or [Messaging docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) |
| MCP servers | `hermes mcp list` or [MCP guide](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) |
| Profiles | `hermes profile list` or [Profiles docs](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) |
| Cron jobs | `hermes cron list` or [Cron docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) |
| Memory | `hermes memory status` or [Memory docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) |
| Source code | `~/.hermes/hermes-agent/` |
