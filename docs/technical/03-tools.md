# 03 — Tools, Toolsets and Terminal Backends

This document describes the tool subsystem: how tools are defined and
discovered, how toolsets group them, the eight execution backends, and
the skills mechanism that lives on top of tools.

## 1. The tool registry

The bottom of the stack is `tools/registry.py`. Every tool file imports
this module and calls `registry.register()` at module top level — the
side effect of importing the file is registration. There is **one
singleton** `ToolRegistry` (the module-level `registry`) whose job is to
hold tool metadata and dispatch tool calls.

### `ToolEntry` (`tools/registry.py:77-98`)

```python
@dataclass
class ToolEntry:
    name: str                  # tool identifier (matches function call name)
    toolset: str               # category: "memory", "discord", "mcp-<server>", ...
    schema: dict               # OpenAI-format JSON schema
    handler: Callable          # invoked with parsed args
    check_fn: Callable | None  # optional availability check (cached 30s)
    requires_env: list[str]    # required env vars
    is_async: bool             # async handler?
    description: str           # human description
    emoji: str                 # display emoji
    max_result_size_chars: int # per-tool truncation
```

### Discovery

`discover_builtin_tools()` (`tools/registry.py:57`) AST-parses every
file in `tools/` to find module-level `registry.register()` calls. It
**only imports** files that actually register a tool — modules that
register from inside a function are skipped. This avoids paying import
cost (and side-effects) for tools that are not used.

### Dynamic availability

`get_definitions()` (`tools/registry.py:310`) honours each tool's
`check_fn` so tools become available or unavailable at runtime
(e.g. Docker daemon up, Modal SDK installed, Playwright binaries
present). Results are cached for 30 seconds (`tools/registry.py:113`)
so the check does not run on every turn.

### Shadow protection

A non-MCP tool cannot register a name already taken by another non-MCP
tool — registration raises (`tools/registry.py:241-263`). MCP tools are
exempt because MCP servers can legitimately re-register the same name
across reconnects.

### Dispatch

`model_tools.handle_function_call(name, args, task_id, user_task)` looks
the tool up in the registry, validates `requires_env`, runs the handler
synchronously, and runs any registered output-truncation hook. Async
handlers are bridged via `model_tools._run_async()` using a
per-thread persistent event loop so cached `httpx.AsyncClient` and
`AsyncOpenAI` instances do not see "Event loop is closed" errors.

## 2. Tool inventory

Tools fall into a small number of practical categories. The list below
groups every file under `tools/`.

### Filesystem operations

| Tool | File | Notes |
|------|------|-------|
| `read_file` / `write_file` / `patch_file` / `search_files` | `tools/file_tools.py` + `tools/file_operations.py` | Backed by the active terminal backend's `execute()`; works on every backend including SSH/Modal/Docker. Path safety via `tools/path_security.py` and binary detection via `tools/binary_extensions.py`. |
| File-state tracking | `tools/file_state.py` | mtime/size cache used by `file_sync.py`. |
| Diff parsing | `tools/patch_parser.py` | Unified-diff parser used by `patch_file`. |
| Fuzzy match | `tools/fuzzy_match.py` | Used by `patch_file` to recover from drifted line numbers. |

### Code execution

| Tool | File | Notes |
|------|------|-------|
| `terminal` | `tools/terminal_tool.py` | Shell command execution via the selected backend. |
| `code_execution` | `tools/code_execution_tool.py` | Stateful Jupyter-style REPL with `force_cpu` and other runtime flags. |
| Background processes | `tools/process_registry.py` | Tracks long-running jobs across turns. |

### Browser & web

| Tool | File |
|------|------|
| `browser` | `tools/browser_tool.py` (CDP-based Chromium) |
| Low-level CDP | `tools/browser_cdp_tool.py` |
| Dialog handler | `tools/browser_dialog_tool.py` |
| Anti-bot variants | `tools/browser_camofox.py`, `tools/browser_camofox_state.py` |
| Process supervisor | `tools/browser_supervisor.py` |
| Web fetch | `tools/web_tools.py` |
| URL safety | `tools/url_safety.py` |
| Site allow-list | `tools/website_policy.py` |

`tools/browser_providers/` is a tiny factory:

* `base.py`        — `BrowserProvider` ABC.
* `browser_use.py` — BrowserUse integration.
* `browserbase.py` — Browserbase cloud browser.
* `firecrawl.py`   — Firecrawl scraper.

### Skills

| Tool | File |
|------|------|
| `skills` (list) | `tools/skills_tool.py` |
| Skill view | `tools/skills_tool.py` (`skill_view` action) |
| Create / edit / delete | `tools/skill_manager_tool.py` |
| Hub install / sync | `tools/skills_hub.py` + `tools/skills_sync.py` |
| Hub security guard | `tools/skills_guard.py` |
| Usage tracking | `tools/skill_usage.py` |

### MCP integration

| Tool | File |
|------|------|
| MCP client (stdio + HTTP transports) | `tools/mcp_tool.py` |
| OAuth flows | `tools/mcp_oauth.py`, `tools/mcp_oauth_manager.py` |
| Managed Modal-hosted MCP gateway | `tools/managed_tool_gateway.py` |

Each connected MCP server registers its tools with the registry under
`toolset = "mcp-<server-name>"`. Errors from the MCP server are
returned to the agent with credentials stripped.

### Memory & notes

| Tool | File |
|------|------|
| Memory ledger (`MEMORY.md`) | `tools/memory_tool.py` |
| Todo list (`TODO.md`) | `tools/todo_tool.py` |
| Kanban boards (`BOARDS.md`) | `tools/kanban_tools.py` |
| Conversation transcript search | `tools/session_search_tool.py` |

### Voice & media

| Tool | File |
|------|------|
| Voice mode | `tools/voice_mode.py` |
| TTS | `tools/tts_tool.py` |
| Local NeuTTS sample synth | `tools/neutts_synth.py` |
| Speech-to-text | `tools/transcription_tools.py` |
| Vision (image analysis) | `tools/vision_tools.py` |
| Image generation | `tools/image_generation_tool.py` |

### Communication & integrations

| Tool | File |
|------|------|
| Cross-platform send | `tools/send_message_tool.py` |
| Discord-specific | `tools/discord_tool.py` |
| Home Assistant | `tools/homeassistant_tool.py` |
| Feishu (Lark) docs / drive | `tools/feishu_doc_tool.py`, `tools/feishu_drive_tool.py` |
| Yuanbao (Alipay) | `tools/yuanbao_tools.py` |

### Delegation & concurrency

| Tool | File |
|------|------|
| `delegate` | `tools/delegate_tool.py` — spawns sub-agents with restricted tool scopes |
| `mixture_of_agents` | `tools/mixture_of_agents_tool.py` — multi-agent voting/ensembling |
| `clarify` | `tools/clarify_tool.py` — pause and ask a clarifying question |
| Interrupts | `tools/interrupt.py` — central interrupt registry |

### Cron

| Tool | File |
|------|------|
| Cron job CRUD | `tools/cronjob_tools.py` |

### Safety, approvals, output limits

| Tool / helper | File |
|---------------|------|
| Approval gates | `tools/approval.py` |
| Tirith security scanning | `tools/tirith_security.py` |
| OSV vulnerability check | `tools/osv_check.py` |
| Slash-command confirmations | `tools/slash_confirm.py` |
| Output truncation | `tools/tool_output_limits.py` |
| Result artifact storage | `tools/tool_result_storage.py` |
| JSON-schema sanitisation | `tools/schema_sanitizer.py` |
| Path safety | `tools/path_security.py` |
| Binary detection | `tools/binary_extensions.py` |
| Token / cost budget | `tools/budget_config.py` |
| Checkpoint snapshots | `tools/checkpoint_manager.py` |
| Credential file pull | `tools/credential_files.py` |
| Backend helpers | `tools/tool_backend_helpers.py` |
| ANSI strip | `tools/ansi_strip.py` |
| xAI HTTP helper | `tools/xai_http.py` |
| OpenRouter HTTP helper | `tools/openrouter_client.py` |
| Env passthrough policy | `tools/env_passthrough.py` |
| Debug helpers | `tools/debug_helpers.py` |

### RL training

| Tool | File |
|------|------|
| RL training loop | `tools/rl_training_tool.py` |

## 3. Toolsets

A **toolset** is a named group of tools. `toolsets.py` defines:

* `_HERMES_CORE_TOOLS` (`toolsets.py:31-68`) — ~50 tools that are always
  on for every Hermes deployment (web, terminal, file, vision,
  image_gen, browser, skills, todo, memory, session_search, cronjob,
  messaging, kanban, home-assistant, …).
* `TOOLSETS` (`toolsets.py:73+`) — declared groups, each with
  `description` and `tools` keys, plus an optional `includes` field that
  composes from other toolsets:

  ```python
  "research": {
      "description": "Research-oriented stack",
      "includes": ["web", "browser", "vision"],
      "tools": [...],
  }
  ```

Helpers:

* `resolve_toolset(name)` — flatten `includes` recursively.
* `validate_toolset(name)` — schema check.

`AIAgent(enabled_toolsets=[...], disabled_toolsets=[...])` filters the
registered tool set. The CLI's `/tools` command and `hermes tools` use
this to scope which tools the agent sees.

### Distributions

`toolset_distributions.py` adds a probability layer on top of toolsets,
used during dataset generation to vary which tools are exposed to the
model:

```python
DISTRIBUTIONS = {
    "default":     {"_all_": 100},
    "image_gen":   {"image_gen": 90, "vision": 90, "web": 55, "terminal": 45, "moa": 10},
    "research":    {"web": 90, "browser": 70, "vision": 50, "moa": 40, "terminal": 10},
    "science":     {"web": 94, "terminal": 94, "file": 94, "vision": 65, "browser": 50,
                    "image_gen": 15, "moa": 10},
    "development": {"terminal": 80, "file": 80, "moa": 60, "web": 30, "vision": 10},
    "safe":        {"terminal": 0, "_all_": 100},  # no terminal
}
```

`sample_toolsets_from_distribution(name)` samples a random toolset
combination per row, and `validate_distribution(name)` does a schema
check at load.

## 4. Terminal backends

`tools/environments/` holds the terminal backends. Every tool that runs
shell commands (`terminal`, `code_execution`, file ops, patch, search,
…) goes through the active backend's `execute()` method. This is what
lets the same tool calls work transparently locally, in Docker, on a
remote SSH host, or in a Modal sandbox.

### Base interface (`tools/environments/base.py`)

```python
class BaseExecutionEnvironment(ABC):
    @abstractmethod
    def execute(self, cmd, cwd=None, timeout=None, stdin_data=None)
        -> tuple[str, str, int]:  # stdout, stderr, returncode
        ...
    def cleanup(self) -> None: ...
```

The model is **spawn-per-call**: every command runs a fresh `bash -c`
process with the previously persisted session snapshot sourced first.
This gives the model a stateless API while preserving cwd, env vars and
shell aliases between calls.

### Backends

| Backend | File | Notes |
|---------|------|-------|
| `local` | `tools/environments/local.py` | Subprocess on the host with `preexec_fn` for process isolation. |
| `docker` | `tools/environments/docker.py` | Bind-mounts the workspace; live host filesystem. |
| `ssh` | `tools/environments/ssh.py` | Remote execution via `ssh`. Uses `file_sync` to ship credentials, skills and cache. |
| `modal` | `tools/environments/modal.py` | Direct Modal SDK client; requires the local `modal` package. |
| `managed_modal` | `tools/environments/managed_modal.py` | Gateway-owned Modal sandbox via the tool-gateway HTTP bridge. No local Modal SDK required — useful for managed cloud deployments. |
| `singularity` | `tools/environments/singularity.py` | Singularity / Apptainer containers (HPC). |
| `daytona` | `tools/environments/daytona.py` | Daytona cloud development environment. |
| `vercel_sandbox` | `tools/environments/vercel_sandbox.py` | Vercel Functions sandbox. |

### File sync (`tools/environments/file_sync.py`)

Shared by remote backends (SSH, Modal, Daytona). Tracks `(mtime, size)`
per file, detects deletions and syncs transactionally. Docker and
Singularity skip it because bind mounts already give the container the
host's live filesystem. `iter_sync_files()` enumerates the things that
must follow the agent: credentials, skills and cache.

### Modal helper

`tools/environments/modal_utils.py` holds Modal-specific shared helpers
(image build, app naming, sandbox lifecycle).

## 5. Skills

Skills sit on top of the tool layer. They are **procedural memory**:
canned instructions and code snippets the agent can load and execute.

### Layout

A skill is a directory with at least `SKILL.md`:

```
skills/<category>/<skill-name>/
├── SKILL.md            # YAML frontmatter + Markdown
├── references/         # optional — extra files loaded on demand (tier 3)
├── templates/          # optional — file templates
└── assets/             # optional — images, fixtures
```

### `SKILL.md` frontmatter

```yaml
---
name: test-driven-development          # max 64 chars
description: "TDD: enforce RED-GREEN-REFACTOR, tests before code."  # max 1024 chars
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [macos]                     # optional OS restriction
compatibility: "Requires X"            # agentskills.io compatibility
metadata:
  hermes:
    tags: [testing, tdd, development, quality, red-green-refactor]
    related_skills: [systematic-debugging, writing-plans, subagent-driven-development]
prerequisites:
  commands: [memo]                     # advisory only — informs the user, does not block
---

# Test-Driven Development (TDD)

## Overview
Write the test first. Watch it fail. ...
```

### Progressive disclosure

To keep token usage low, skills are loaded in three tiers:

1. **Tier 1** — `name + description` only, returned by `skills_list`.
2. **Tier 2** — `SKILL.md` body, returned by `skill_view(skill_name)`.
3. **Tier 3** — files under `references/`, `templates/`, `assets/`,
   loaded on demand via `read_file`.

### Built-in vs optional

* `skills/` ships with the package and is enabled by default.
* `optional-skills/` ships but requires explicit activation via
  `hermes skills install` or the Hub UI.

### Skills Hub

`tools/skills_hub.py` is a centralised source-adapter pattern.
`SkillSource` is the ABC; concrete sources include `OptionalSkillSource`
(in-repo opt-ins), `GitHubSource` (any GitHub repo via the Contents
API), and registry adapters (Claude Marketplace, LobHub, …).

State lives at `~/.hermes/skills/.hub/`:

* `lock.json`     — provenance for installed skills.
* `quarantine/`   — sandboxed staging area for in-flight installs.
* `audit.log`     — append-only install history.
* `taps.json`     — installed Hub "taps" (extra sources).
* `index-cache/`  — cached source indices.

Hub-installed skills run through `tools/skills_guard.py`, which scans
for prompt injection patterns and known-bad shell incantations before
the skill becomes loadable.

### Curator

`agent/curator.py` runs in the background to maintain agent-created
skills. It is not a daemon — it fires when the agent has been idle for
`curator.idle_threshold_seconds` and `curator.interval_hours` has
elapsed since the last run. It spawns a forked `AIAgent` to review,
pin, archive, consolidate or patch the skills the user has been
generating, and writes its state to
`~/.hermes/skills/.curator_state`.
