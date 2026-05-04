# 07 — Developer Guide

This guide is the operational handbook for working on the
`hermes-agent` codebase: setting up an environment, running tests,
adding features, and shipping a release.

For deep architectural context, read [01-architecture.md](01-architecture.md)
first. The canonical contributor document is the in-tree
[CONTRIBUTING.md](../../CONTRIBUTING.md); this document layers on
implementation detail.

## 1. Environment

### Prerequisites

| Requirement | Notes |
|-------------|-------|
| Git         | With `--recurse-submodules` and `git-lfs`. |
| Python 3.11+| `uv` will install it if missing. |
| `uv`        | Fast Python package manager — [install](https://docs.astral.sh/uv/). |
| Node.js 20+ | Optional. Needed for the Ink TUI, browser tools, the WhatsApp bridge. Matches `package.json` engines. |

### Setup

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

uv venv venv --python 3.11
source venv/bin/activate

uv pip install -e ".[all,dev]"

# Optional: browser tools
npm install
```

`scripts/run_tests.sh` will probe `.venv`, then `venv`, then
`$HOME/.hermes/hermes-agent/venv` (the last is for worktrees that
share a venv with the main checkout).

`./hermes` (in the repo root) is a wrapper that auto-detects the venv
so you do not need to `source venv/bin/activate` first.

### Profiles for development

When experimenting on changes, isolate state with a profile:

```bash
hermes --profile dev
```

This switches `~/.hermes` → `~/.hermes-dev`, so your config, sessions,
skills and cron jobs do not bleed into your daily install. The
profile-override is applied **before any other import** that reads
`get_hermes_home()` (`hermes_cli/main._apply_profile_override`).

## 2. Running tests

The canonical entry is `scripts/run_tests.sh`. **Always use it** rather
than calling `pytest` directly — it enforces the same environment that
CI uses (deterministic `TZ=UTC`, `LANG=C.UTF-8`, `PYTHONHASHSEED=0`,
blanked credential env vars, `pytest-split` installed, `-n 4` xdist
workers matching CI cores).

```bash
scripts/run_tests.sh                                  # full suite
scripts/run_tests.sh tests/agent/                     # one directory
scripts/run_tests.sh tests/agent/test_foo.py::TestX   # one test
scripts/run_tests.sh --tb=long -v                     # pytest pass-through
```

Pytest configuration lives in `pyproject.toml` (`[tool.pytest.ini_options]`):

```
testpaths = ["tests"]
markers   = ["integration: marks tests requiring external services"]
addopts   = "-m 'not integration' -n auto"
```

The `integration` marker is excluded by default. Tests that talk to a
live provider, Modal, Daytona, etc. should be marked
`@pytest.mark.integration`.

### Test layout

```
tests/
├── conftest.py             # shared fixtures, env scrubbing
├── agent/                  # agent/* unit tests
├── cli/                    # cli.py + hermes_cli unit tests
├── gateway/                # gateway + platform adapters
├── tools/                  # tools/*
├── skills/                 # skills + curator
├── plugins/                # plugin loader + sample plugins
├── e2e/                    # end-to-end (mostly integration-marked)
├── integration/            # integration suite
├── stress/                 # load / concurrency
├── fakes/                  # in-memory fakes for SDKs
├── run_agent/              # AIAgent loop tests
├── hermes_state/           # SessionDB tests
├── tui_gateway/            # TUI gateway JSON-RPC tests
├── website/                # docs-site link checks
└── test_<topic>.py         # cross-cutting top-level tests
```

`tests/conftest.py` blanks credential env vars (`OPENAI_API_KEY`,
`ANTHROPIC_API_KEY`, …) so accidentally-real keys do not leak into a
test run.

## 3. Adding code

### Add a slash command

The single source of truth is `COMMAND_REGISTRY` in
`hermes_cli/commands.py`. Every consumer (CLI dispatch, gateway
dispatch, Telegram BotCommand menu, Slack subcommand routing,
autocomplete, help) derives from this list.

1. Add a `CommandDef` to `COMMAND_REGISTRY`:

   ```python
   CommandDef(
       "mycommand", "Description", "Session",
       aliases=("mc",), args_hint="[arg]",
   )
   ```

2. Add the handler in `cli.HermesCLI.process_command()`:

   ```python
   elif canonical == "mycommand":
       self._handle_mycommand(cmd_original)
   ```

3. If the command is also available via the gateway, add a handler in
   `gateway/run.py`:

   ```python
   if canonical == "mycommand":
       return await self._handle_mycommand(event)
   ```

4. Persistent settings should use `save_config_value()` in `cli.py`.

`CommandDef` fields: `name`, `description`, `category`
(`"Session"` / `"Configuration"` / `"Tools & Skills"` / `"Info"` /
`"Exit"`), `aliases`, `args_hint`, `cli_only`, `gateway_only`,
`gateway_config_gate`. An alias is just a tuple entry; no other file
needs updating.

### Add a tool

Two files only.

**1. Create `tools/your_tool.py`:**

```python
import json
import os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={
        "name": "example_tool",
        "description": "...",
        "parameters": {...},  # JSON Schema
    },
    handler=lambda args, **kw: example_tool(
        param=args.get("param", ""),
        task_id=kw.get("task_id"),
    ),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

**2. Add the tool name to `toolsets.py`** — either to
`_HERMES_CORE_TOOLS` (everywhere) or to a new entry in `TOOLSETS`.

That is enough; auto-discovery (`tools.registry.discover_builtin_tools`)
will import the file because it has a top-level
`registry.register()` call. The handler **must** return a JSON string.

Use `display_hermes_home()` and `get_hermes_home()` for any path that
the user might see or that stores state — never hardcode
`Path.home() / ".hermes"`. This keeps the tool profile-aware.

### Add a config option

Settings live in `~/.hermes/config.yaml`:

1. Add to `DEFAULT_CONFIG` in `hermes_cli/config.py`.
2. Bump `_config_version` **only** if you need an active migration
   (renaming or restructuring keys). Adding a new key inside an
   existing section is handled automatically by the deep-merge.

Secrets live in `~/.hermes/.env`:

1. Add to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py`:

   ```python
   "NEW_API_KEY": {
       "description": "What it's for",
       "prompt": "Display name",
       "url": "https://...",
       "password": True,
       "category": "tool",   # provider | tool | messaging | setting
   },
   ```

`hermes setup` automatically picks new entries up.

### Add a provider

See [06-providers.md](06-providers.md). In short:

* OpenAI-compatible? Reuse the `chat_completions` transport — only
  configuration needed (a `base_url` and a credential).
* Custom protocol? Subclass `ProviderTransport` in
  `agent/transports/<provider>.py`; add a `<provider>_adapter.py` for
  quirks; register in the model catalog
  (`scripts/build_model_catalog.py` and/or `hermes_cli/model_catalog.py`).

### Add a platform

See [04-gateway.md](04-gateway.md) and the in-tree
`gateway/platforms/ADDING_A_PLATFORM.md`. New platforms either:

* live in-tree under `gateway/platforms/<name>.py`, or
* live in a plugin under `plugins/platforms/<name>/` (mirrors of
  `gateway/platforms/`).

### Add a memory provider

See [05-state-and-memory.md](05-state-and-memory.md):

1. Create `plugins/memory/<name>/` with a `plugin.yaml`, a
   `__init__.py` exporting your `MemoryProvider` subclass, and any
   helper modules.
2. The plugin manifest declares `pip_dependencies` so
   `hermes plugins install <name>` knows what to install.
3. Existing plugins (`plugins/memory/honcho`, `mem0`, `supermemory`,
   …) are good templates.

### Add a context engine

Mirror the memory-provider story, but under
`plugins/context_engine/<name>/`. Subclass `ContextEngine` from
`agent/context_engine.py` and select it from `~/.hermes/config.yaml`
under `context.engine`.

### Add a plugin

The minimal plugin layout:

```
plugins/<name>/
├── __init__.py     # plugin entry; registers hooks / tools
├── plugin.yaml     # manifest
└── README.md       # user-facing description
```

`plugin.yaml`:

```yaml
name: <name>
version: 1.0.0
description: "..."
pip_dependencies: [...]
hooks:
  - on_session_end
provides_tools:
  - some_tool
```

The loader (`plugins/__init__.py`) discovers any subdirectory whose
`plugin.yaml` is parseable. The plugin's `__init__.py` is imported and
expected to call into `tools.registry.register()` for any tools it
ships and to register hook callbacks.

## 4. Code style and linting

The project uses **`ruff`** (configured in `pyproject.toml`) and
**`ty`** (Astral's type checker, configured under `[tool.ty.*]`). The
`pyproject.toml` excludes `*` for ruff — there is no project-wide
linting; instead style and quality are enforced per-PR via reviewers
and the CI test suite.

`ty` is configured to **ignore** unresolved imports, invalid method
overrides, invalid assignments and not-iterable diagnostics by default
(see `[tool.ty.overrides.rules]`). It is intentionally permissive
during the migration to typed code; do not fight it.

### General conventions

* Prefer plain `dict`/`dataclass`-shaped state over class hierarchies.
* Tools must return **JSON strings** — never Python objects.
* All persistent paths must go through `get_hermes_home()`.
* All clocks must use `hermes_time.now()` so `freeze_time` works in
  tests.
* All file writes that are durable must use the `atomic_*` helpers
  from `utils.py`.
* Logs must use the `logging` module (not `print`); secrets are
  redacted by `RedactingFormatter` in `hermes_logging.py`.

## 5. Building & releasing

### Local build

The project is a standard PEP-517 setuptools build:

```bash
uv pip install build
python -m build
```

This produces `dist/hermes_agent-<version>-py3-none-any.whl` and a
matching `.tar.gz`. The `tool.setuptools.packages.find` and
`tool.setuptools.py-modules` sections in `pyproject.toml` enumerate
exactly what ships in the wheel.

### Release script

`scripts/release.py` automates the version bump, changelog scaffold
(`RELEASE_v<version>.md`), git tag, GitHub release, and PyPI upload.
Run it from a clean checkout on `main`.

The TUI ships in the wheel as **prebuilt JS** under
`hermes_cli/web_dist/` (declared in `[tool.setuptools.package-data]`).
`scripts/release.py` runs the Ink and dashboard builds before packaging.

### Container image

`Dockerfile` produces a single image with the `[all]` extra installed
and `hermes` on `PATH`. `docker-compose.yml` wires it up against a
named volume mounted at `~/.hermes` for persistence.

```bash
docker compose up -d
docker compose exec hermes hermes
```

### Nix

`flake.nix` (+ `flake.lock`) builds a Nix-friendly derivation. Useful
for reproducible CI and for users on NixOS / nix-darwin who want
declarative installs. The Nix derivation explicitly pins the `[google]`
extra so packaged installs do not run `pip install` at first launch.

## 6. Continuous integration

The repo's CI (under `.github/workflows/`) runs `scripts/run_tests.sh`
across the supported Python versions and operating systems. Test
parallelism uses `pytest-split` with `pytest-xdist` (`-n 4`) — the same
shape `scripts/run_tests.sh` enforces locally.

## 7. Performance & profiling

* `scripts/profile-tui.py` is a profiler harness for the Ink TUI; use
  it when investigating typing-latency issues.
* `cli.py` uses lazy imports for the heaviest SDKs (OpenAI, Anthropic,
  Fire, MCP, `account_usage`) so cold-start times stay under ~250 ms.
  When adding new top-level imports, measure the cost first.
* The agent loop is synchronous on purpose — async appears only inside
  tools that need it. Do not "asyncify" the loop.

## 8. Security considerations

The threat surface is documented in [SECURITY.md](../../SECURITY.md). The
high-impact areas:

* **Shell injection** — every shell tool quotes via the relevant
  backend's helpers; never string-concat untrusted input into shell
  commands.
* **Prompt injection** — `prompt_builder.py` scans context files for
  injection patterns, invisible Unicode and exfil-shaped URLs.
  `tools/skills_guard.py` runs the same heuristics on Hub-installed
  skills.
* **Path traversal** — `tools/path_security.py` provides the canonical
  guards used by file tools.
* **Privilege escalation** — `tools/approval.py` gates high-risk
  actions; the gateway layer adds per-platform allow-lists.
* **Container isolation** — production gateway deployments should run
  the agent on `docker` or `managed_modal`, not `local`.

## 9. Where to look when…

| Scenario | Where |
|----------|-------|
| Diagnosing a hang in the agent loop | `run_agent.py:run_conversation()` and `tools/interrupt.py` |
| Diagnosing a tool error | `tools/registry.py:get_definitions/dispatch`, then the tool file |
| A provider request looks wrong | `agent/transports/<api_mode>.py` and `agent/<provider>_adapter.py` |
| The CLI is rendering badly | `cli.py` (HermesCLI), `agent/display.py`, `hermes_cli/skin_engine.py` |
| Slash command not dispatched | `hermes_cli/commands.py:COMMAND_REGISTRY` and `cli.process_command()` / `gateway/run.py` |
| Memory not persisting | `agent/memory_manager.py` and the relevant `plugins/memory/<name>/` |
| Context not compressing | `agent/context_engine.py`, `agent/context_compressor.py`, config `context.*` |
| Skill not loading | `agent/skill_commands.py`, `agent/skill_utils.py`, `tools/skills_tool.py` |
| Cron not firing | `cron/scheduler.py:tick()`, lock file, gateway must be running |
| Gateway not forwarding | `gateway/run.py:_handle_message()` and `gateway/platforms/<name>.py` |
| Sessions not searchable | `hermes_state.py:SessionDB` (FTS tables), `tools/session_search_tool.py` |

## 10. House rules

These are de-facto conventions you will see throughout the codebase
and that reviewers will ask you to follow:

* **No `print`** — always use `logging`.
* **No hardcoded `~/.hermes`** — always `get_hermes_home()`.
* **No naive `datetime.now()`** — always `hermes_time.now()`.
* **No raw `time.sleep` in tests** — use the test fixtures.
* **No new top-level Python files** unless adding a new entry-point
  module; everything else lives under a package.
* **No new docs without an in-product entry point** — if you write a
  user-facing doc, also link it from `hermes doctor`, `hermes setup`
  or `hermes help`.
* **All handlers return JSON strings**, including tool handlers and
  gateway slash-command handlers.
* **Atomic writes** for any persistent state file.
* **Tests for every bug fix** — minimum bar.
