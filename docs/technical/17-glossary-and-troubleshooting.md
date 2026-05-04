# 17 — Glossary and Troubleshooting

This document is a quick-reference companion to the rest of the
technical docs. Part 1 is a glossary of the terms used throughout the
codebase. Part 2 is a triage guide for the most common failure modes.

## Part 1 — Glossary

### `AIAgent`
The core conversation-loop class (`run_agent.py`). Every surface (CLI,
gateway, batch, ACP, MCP) instantiates `AIAgent` to do real work.
~60 constructor parameters. See [01-architecture.md](01-architecture.md).

### Adapter
Provider-specific code that handles quirks not solvable in the
transport layer (e.g. thinking budgets, max-output tables, OAuth
setup-tokens). Lives in `agent/<provider>_adapter.py`.

### ACP (Agent Client Protocol)
A JSON-RPC protocol that lets editors / IDEs drive a conversational
agent. Hermes ships an ACP server in `acp_adapter/`. See
[11-mcp-and-acp.md](11-mcp-and-acp.md).

### Adapter (gateway platform)
`gateway/platforms/<name>.py` — implements a specific messaging
platform on top of `BasePlatformAdapter`.

### Allow-list (URL / command)
Configurable list of permitted URLs (`tools/url_safety.py`) or shell
commands (`tools/approval.py`). Anything not on the allow-list is
blocked or requires explicit user approval.

### Approval gate
The system that prompts the user for confirmation before executing a
high-risk tool call (`tools/approval.py`). Slash interactions in the
gateway and `request_permission` events in ACP.

### Atropos
Open RL framework Hermes' `environments/` package targets. Pulled in
by the `[rl]` extra (`atroposlib`). See
[10-batch-and-rl.md](10-batch-and-rl.md).

### Auxiliary client
Cheap/fast LLM client (`agent/auxiliary_client.py`) used by curator,
compressor, title generator and insights. Configured under
`config.auxiliary.*`.

### Backend
Synonymous with "terminal backend" — the execution environment the
agent's shell tools target (`local`, `docker`, `ssh`, `modal`,
`managed_modal`, `singularity`, `daytona`, `vercel_sandbox`).

### Batch runner
`batch_runner.py` — parallel agent runner that produces JSONL
trajectories from a dataset of prompts.

### Branch (session)
A copy of the current session that can be steered in a different
direction without affecting the original. Implemented by writing a new
session row in `SessionDB` with the current message list copied. The
slash command is `/branch` or `/fork`.

### Cache control (Anthropic)
Markers placed on messages so the provider caches the prefix.
Hermes uses the `system_and_3` strategy
(`agent/prompt_caching.apply_anthropic_cache_control`): system
prompt + last 3 non-system messages.

### Catalog (model)
The bundled `hermes_cli/model_catalog.py` data + the snapshot built
by `scripts/build_model_catalog.py`. Drives `hermes model` and
`/model`.

### Channel
A messaging-platform conversation (Telegram chat, Discord channel,
Slack channel). The gateway maps `(platform, channel_id)` → session.

### Checkpoint
A filesystem snapshot taken before risky operations
(`tools/checkpoint_manager.py`). Restorable via `/rollback`.

### Compressed session
A session whose `parent_session_id` points at an earlier session, used
by `ContextCompressor` to keep the *original* verbatim history while
the active session runs on a compact summary.

### Compressor
The default `ContextEngine` implementation
(`agent/context_compressor.py`). Summarises the middle of a long
conversation to free context budget.

### `CommandDef`
The dataclass declaring a slash command (`hermes_cli/commands.py`).
Single source of truth for CLI dispatch, gateway dispatch, Telegram
BotCommand list, Slack subcommand routing, autocomplete and help.

### Context engine
A pluggable component that owns context-window management
(`agent/context_engine.py`). Default = `compressor`. Alternative ships
in `plugins/context_engine/`.

### Context file
One of `.hermes.md`, `AGENTS.md`, `SOUL.md`, `.cursorrules` —
project-level instructions injected into the system prompt by
`agent/prompt_builder.py`. Subject to threat scanning.

### Cred(ential) pool
`agent/credential_pool.py:CredentialPool`. Holds multiple credentials
per provider with rotation strategies (`FILL_FIRST`, `ROUND_ROBIN`,
`RANDOM`, `LEAST_USED`).

### Cron job
A scheduled prompt + delivery target (`cron/jobs.py`). Executed by
`cron.scheduler.tick()` from inside the gateway. See
[04-gateway.md](04-gateway.md).

### Curator
`agent/curator.py` — the autonomous skill maintainer. Runs in the
background after periods of user inactivity to pin / archive /
consolidate / patch agent-created skills.

### Dashboard
The localhost SPA + FastAPI server (`hermes dashboard`). Embeds the
live TUI through a PTY-backed WebSocket. See
[12-tui-and-dashboard.md](12-tui-and-dashboard.md).

### Delegation
Spawning a sub-agent to handle a sub-task with a restricted toolset
(`tools/delegate_tool.py`). The subagent reuses the parent's
credential pool.

### Delivery target
A string that tells the gateway where to send a result. Forms:
`origin`, `local`, `<platform>:<chat_id>` (e.g. `telegram:123456789`).

### `DeliveryRouter`
`gateway/delivery.py` — resolves a delivery target to the right
adapter and posts the message.

### `discover_builtin_tools`
`tools/registry.py:57` — walks the `tools/` directory and imports
every file that has a top-level `registry.register()` call.

### `display_hermes_home`
`hermes_constants.py:145` — returns the user-friendly path
representation (e.g. `~/.hermes`). Use it in tool schema descriptions
that the user might see.

### Distribution (toolset)
A probability-weighted toolset combination
(`toolset_distributions.py`). Used by `batch_runner` to vary tool
exposure across rows.

### Dot-files
Project-level instruction files: `.hermes.md`, `AGENTS.md`,
`SOUL.md`, `.cursorrules`. (See "Context file".)

### `FILL_FIRST`
Default credential pool strategy: drain the highest-priority credential
until it errors, then move to the next.

### Fast mode
Provider-side priority-processing (Anthropic Fast Mode, OpenAI
Priority Processing). Toggled with `/fast`.

### `find_git_root`
`agent/prompt_builder.py:_find_git_root` — walks up the cwd looking
for a `.git/` directory; used to scope context-file discovery to the
current project.

### FTS5
SQLite full-text-search extension. Hermes uses two FTS5 tables on
`messages`: Unicode61 (English / Romance) and trigram (CJK).

### Gateway
The messaging server (`gateway/run.py`) that bridges the agent to
chat platforms. See [04-gateway.md](04-gateway.md).

### `get_hermes_home`
`hermes_constants.py:14` — single source of truth for the `~/.hermes`
path, accounting for `HERMES_HOME` overrides and active profiles.

### Goals
Standing tasks the agent works on across multiple turns
(`hermes_cli/goals.py`). Slash command: `/goal`.

### Guardrails (tool loop)
Heuristics that detect pathological loops (same tool called N times
in a row, repeated empty results) and break out. Configured under
`config.tool_loop_guardrails`.

### Handoff framing
Compression technique borrowed from Codex: present the summary as if
it were generated by a *different* assistant, encouraging the model to
treat it as authoritative external context rather than its own past
output.

### HermesCLI
The interactive REPL class (`cli.py`). prompt_toolkit-based; ~12k
LOC.

### Honcho
The default external memory provider (Plastic Labs' Honcho). Pulled in
by the `[honcho]` extra. See `plugins/memory/honcho/`.

### Hub (skills)
External skill source (GitHub repo, Hermes Cloud, custom URL). State
under `~/.hermes/skills/.hub/`. See [09-skills-system.md](09-skills-system.md).

### Insights
Cross-session usage analytics (`agent/insights.py`). Slash command:
`/insights [--days N]`.

### Iteration budget
The maximum number of tool-call iterations per turn
(`AIAgent.max_iterations`, default 90). Shared with subagents.

### Kanban
Multi-profile collaboration board (`hermes_cli/kanban.py`,
`hermes_cli/kanban_db.py`). State at `~/.hermes/kanban.db`.

### KawaiiSpinner
The animated face that appears while the agent is "thinking"
(`agent/display.py`). Configurable via
`config.display.spinner`.

### LCM (engine)
Alternative context engine that ships in `plugins/context_engine/`.
Uses an external "LCM" system instead of Hermes' built-in summariser.

### LRU cache (gateway agents)
The bounded cache of `AIAgent` instances kept by `GatewayRunner` —
default capacity 128, idle TTL 1 hour.

### MCP
Model Context Protocol. Hermes acts as both client (`tools/mcp_tool.py`)
and server (`mcp_serve.py`).

### Memory provider
Pluggable memory backend (`agent/memory_provider.py`). Always-on
built-in plus at most one external (`plugins/memory/<name>/`).

### `MessageEvent`
The normalised inbound shape created by every gateway adapter
(`gateway/platforms/base.py`). Carries text, source, channel, user,
attachments, timestamp.

### Mini SWE runner
`mini_swe_runner.py` — single-task runner for SWE-bench-style tasks.

### Model catalog
Bundled per-provider model metadata (`hermes_cli/model_catalog.py`,
`scripts/build_model_catalog.py`).

### Modal
Modal Cloud — sandbox provider. Two backends: `modal` (direct SDK)
and `managed_modal` (gateway-owned).

### MoA — Mixture of Agents
`tools/mixture_of_agents_tool.py`. Multi-agent ensemble / voting.

### `NormalizedResponse`
Shared shape (`agent/transports/types.py`) that every transport
maps its raw provider response to.

### Nous Portal
Nous Research's hosted provider. `NOUS_API_KEY` /
`NOUS_BASE_URL`.

### One-shot
Non-interactive single-turn invocation: `hermes -p "<prompt>"`.

### OpenClaw
The predecessor project. Hermes' `hermes claw migrate` imports its
state.

### Optional skills
Skills shipped under `optional-skills/` but not active by default.
Activated via `hermes skills install` or the Hub UI.

### `parent_session_id`
Foreign key in `sessions` rows pointing to a previous session. Used
for compressed-session chaining.

### `PassiveContextEngine` (proposed)
Skeleton in [16-extension-recipes.md](16-extension-recipes.md);
example of a no-op context engine.

### Personality
Persona overlay loaded as a system prompt block. Defined in
`config.personalities`. Slash command: `/personality`.

### Plugin
Python package under `plugins/<name>/` with a `plugin.yaml` manifest.
See [02-modules.md](02-modules.md).

### `PooledCredential`
Single credential entry in `CredentialPool`
(`agent/credential_pool.py:91-137`).

### Profile
Isolated `~/.hermes` root: `--profile dev` swaps to `~/.hermes-dev/`.
See [14-configuration-reference.md](14-configuration-reference.md).

### Prompt cache
See "Cache control".

### `prompt_builder`
`agent/prompt_builder.py` — assembles the system prompt at session
start.

### `prompt_toolkit`
Python library powering the classic REPL. Optional dependency —
gateway-only and test-only environments can import
`hermes_cli.commands` without it.

### Provider
A model vendor (Anthropic, OpenAI, OpenRouter, Bedrock, Gemini, …).
Selected via `provider:` on the model identifier.

### `ProviderTransport`
Abstract base class (`agent/transports/base.py`) for the format-translation
layer between Hermes and a specific provider protocol.

### `redact`
`agent/redact.py:redact()` — secret-scrubbing helper applied to logs
and error surfaces.

### Reasoning
Provider-emitted "thinking" tokens. Hermes preserves them as
`<think>...</think>` tags in trajectories and surfaces them in the
CLI under `config.display.reasoning`.

### Registry (tool)
`tools/registry.py` — singleton holding every registered tool.

### Resolved / Pending (compression)
The two question lists carried by every compression summary. See
`agent/context_compressor.py:38-49`.

### `RL`
Reinforcement learning. The `[rl]` extra installs Atropos + tinker;
`environments/` ships envs.

### Scratchpad
Earlier name for reasoning blocks; `agent/trajectory.py` converts
`<REASONING_SCRATCHPAD>` → `<think>` for cross-provider portability.

### Session
A single conversation, identified by `session_id`, persisted in
`SessionDB`.

### `SessionDB`
`hermes_state.py:159` — the SQLite-backed session store.

### Setup wizard
`hermes setup` (`hermes_cli/setup.py`). First-run + on-demand
configuration flow.

### Skill
Procedural memory: a `SKILL.md` directory loaded into the conversation
on demand. See [09-skills-system.md](09-skills-system.md).

### Skills Guard
`tools/skills_guard.py` — security scanner run on Hub-installed
skills.

### Skills Hub
Distribution / discovery infrastructure for third-party skills. See
[09-skills-system.md](09-skills-system.md).

### Skill sync
Pushing user-curated skills to the Hub
(`tools/skills_sync.py`).

### Slash command
A command that starts with `/` in the chat surface. Defined in
`hermes_cli/commands.py:COMMAND_REGISTRY`.

### `SOUL.md`
Persona file under `~/.hermes/SOUL.md`. Included in the system prompt.

### Status bar
The bottom-of-screen line in the CLI showing model / context budget /
token counts. Toggled with `/statusbar`.

### Steer
Inject a message after the next tool call without interrupting.
Slash command: `/steer`.

### `SUMMARY_PREFIX`
The leading template used in compression summaries
(`agent/context_compressor.py:38-49`).

### Sub-agent
A nested `AIAgent` spawned via `tools/delegate_tool.py`.

### System prompt
The first message of any conversation. Built by
`agent/prompt_builder.py`.

### Telegram BotCommand menu
Auto-generated from `COMMAND_REGISTRY` so `/<x>` works on Telegram with
auto-completion in the chat input.

### Tirith
Nous's security scanner; integrated via `tools/tirith_security.py`.

### Toolset
Named group of tools (`toolsets.py`).

### Toolset distribution
Probability-weighted set of toolsets (`toolset_distributions.py`).

### Trajectory
A serialised conversation in Hermes JSONL format
(`agent/trajectory.py`).

### Trajectory compressor
`trajectory_compressor.py` — the post-hoc compactor for training
data.

### Transport
See `ProviderTransport`.

### TUI
The Ink/React terminal UI under `ui-tui/`, driven by `tui_gateway/`.

### `_run_async`
`model_tools.py:82` — bridges sync tool callers to async clients
without re-creating the event loop.

### WAL
Write-ahead logging — SQLite mode used by `SessionDB`.

### YOLO mode
`/yolo` — disables approval prompts. Use only inside a sandbox.

## Part 2 — Troubleshooting

A set of common failure modes and how to diagnose them.

### "Event loop is closed" when calling a tool

**Cause:** a previous tool call left a closed `asyncio` loop on the
calling thread.

**Fix:** the codebase routes async tool calls through
`model_tools._run_async()`, which keeps a per-thread persistent loop.
If a tool you wrote calls `asyncio.run()` directly, that breaks the
contract — replace with the persistent-loop helper.

### Provider returns 429 even after waiting

**Cause:** the OpenAI SDK has its own retry loop (default 2) that runs
inside the Hermes retry loop (default 3) — so 9 calls per "turn"
without coordination. For Nous Portal, also check
`agent/nous_rate_guard.py`.

**Fix:**

1. Check `~/.hermes/rate_limits/<provider>.json` for the latest reset
   time the guard knows about.
2. Lower `agent.api_max_retries` to `1` if you are using
   `fallback_providers` and want fast failover.
3. Add another credential to the pool so one exhaustion does not
   block the queue.

### Skills disappear after `/reload-skills`

**Cause:** `config.skills.disabled` includes them, or
`metadata.hermes.disabled: true` is set in the frontmatter, or the
skill's `conditions` block evaluates false on this host.

**Fix:** `hermes skills list --all` shows every skill including
disabled ones with the reason. `hermes skills view <name>` prints the
parsed frontmatter.

### Curator never runs

Causes (in order of likelihood):

1. `config.curator.enabled: false` or `config.curator.paused: true`.
2. `config.curator.interval_hours` not yet elapsed since last run.
3. `idle_threshold_seconds` not yet reached.
4. `prefer_models` references an unavailable provider.

`hermes curator status` reports the resolved state. `hermes curator
run` forces an immediate pass.

### Gateway crashes on startup with TLS error

Most common: `aiohttp` cannot find the system trust store. Set
`SSL_CERT_FILE` to a known bundle, or install the `certifi` package
into the venv. WSL2 users sometimes need to copy `/etc/ssl/certs/...`
manually.

### Telegram messages are sent but never received

* Check `TELEGRAM_ALLOWED_USERS` — your user id may not be on the
  list.
* Check `TELEGRAM_BOT_TOKEN` — invalid tokens silently fail.
* Check the bot's privacy mode (Telegram-side) — group chats need it
  disabled to receive plain messages.

### `hermes update` says "managed install detected"

A managed install (Nix, Homebrew, systemd unit) sets
`HERMES_MANAGED=<system>`. `hermes update` refuses to upgrade in this
case so you do not bypass the package manager. Use the system's own
upgrade command (`brew upgrade hermes-agent`,
`sudo nixos-rebuild switch`).

### Sessions search returns nothing for CJK queries

`messages_fts` (Unicode61) does not tokenise CJK on word boundaries —
that is what `messages_fts_trigram` is for. The session-search tool
already routes CJK queries to the trigram table, but custom callers
hitting `SessionDB` directly need to query the right table. See
`hermes_state.py:103-156`.

### MCP server connects then immediately disconnects

* stdio transport: the child process logged something on stderr while
  it was supposed to respond on stdout. Look at
  `~/.hermes/logs/agent.log` for the captured stderr.
* HTTP transport: check OAuth token freshness; expired tokens often
  fail with `401` masked as a connection close.
* Either: enable `autoreconnect: true` in `mcp_servers.yaml` so the
  guard re-establishes the link after transient failures.

### "Permission denied" on file write inside docker backend

The Docker backend bind-mounts the workspace; the container user
needs write permission on the mount. Either run the container as the
host user (`--user $(id -u):$(id -g)` or set `terminal.docker.user_map:
true`) or set the workspace permissions accordingly.

### TUI quits on resize

The PTY resize path (`hermes_cli/pty_bridge.py`) only works on POSIX
PTYs. On native Windows there is no `ptyprocess`. Use WSL2.

### CLI prints `?` characters instead of glyphs

Locale issue. Set `LANG=C.UTF-8` (or your locale's UTF-8 variant) in
the shell before launching `hermes`.

### `hermes config edit` strips comments

The YAML round-tripper falls back to `pyyaml` when `ruamel.yaml`
cannot represent something. As a workaround:

* Edit `config.yaml` directly with your `$EDITOR`.
* Avoid `hermes config set` for keys you have annotated with comments.

### `hermes setup` cannot detect TTY

Some non-TTY contexts (CI, sandboxed shells) make `setup` unsafe.
`tests/test_install_sh_setup_wizard_tty_probe.py` documents the probe
logic. Set `HERMES_DEV=1` to bypass the TTY check (development only).

### Trajectories saved with `<tool_call>` but not `<tool_response>`

Cause: the model returned a tool call as the final output of the turn
(probably hit `max_iterations`). Check
`failed_trajectories.jsonl` — the row will have a `finish_reason` of
`length` or a custom `iteration_budget_exhausted`.

Fix: bump `max_iterations`, or invoke `agent.run_conversation` with a
follow-up turn so the model gets to consume the tool response.

### Provider request body is missing `cache_control`

Two prereqs: `config.prompt_caching.enabled: true` and the active
provider must be in `config.prompt_caching.enabled_providers` (default
`["anthropic"]`). The transport applies `apply_anthropic_cache_control`
in `build_kwargs` only for matching providers.

### "No module named X" inside a worker spawned by `batch_runner`

`batch_runner` uses `spawn` start method — the worker does not inherit
the parent's `sys.path` modifications. Either:

* Install Hermes editably (`pip install -e .`) so the package is on
  the venv's `sys.path`, or
* Pass `--worker_setup_path=<path>` so the worker prepends it.

### `~/.hermes/state.db` is locked

Causes:

* Another `hermes` process is holding a long write transaction.
* A crashed process left a `.db-shm` / `.db-wal` pair.

Fix:

* Wait — the SessionDB jitter retry usually resolves contention
  within a few hundred ms.
* If clearly stuck: `lsof | grep state.db`, kill the offender, then
  `sqlite3 state.db .recover` if you suspect corruption.

### `hermes doctor` reports a tool as unavailable

Check the tool's `check_fn` (`tools/<tool>.py`):

* Are the `requires_env` keys set? `printenv | grep <KEY>`.
* Are the prerequisite binaries on `PATH`? `which <bin>`.
* Is the active backend able to run the tool? Some tools require
  `local` (e.g. `voice_mode`).

### `hermes claw migrate` fails halfway

Re-run with `--dry-run` to see what would be migrated. Use
`--preset user-data` to skip secrets if the secrets path is the issue.
The migration is idempotent — re-running with `--overwrite=false` will
skip anything already migrated.

### `gateway` exits silently on Linux

`systemd` users running the gateway under a unit: ensure the unit's
`StandardOutput=` / `StandardError=` does not point at a closed file
descriptor when the agent forks for delegation. Use
`StandardOutput=journal` for safety.

### `mcp_serve` blocks indefinitely

The MCP host did not send `initialize`. Check the host's logs
(Claude Code: View → Output → MCP). Hermes' MCP server expects the
spec-mandated handshake — without it, no further methods are allowed.

### `hermes -p` returns no output and exit 0

Likely the model returned an empty assistant message and a
non-error finish reason. Re-run with `HERMES_DEBUG=1` to see the
full transport response.

### Docker backend cannot find binaries

`tools/environments/docker.py` runs commands inside the configured
image. The default image is small; install whatever your tools
require:

```dockerfile
FROM nousresearch/hermes-sandbox:latest
RUN apt-get update && apt-get install -y ffmpeg pandoc imagemagick
```

Then `config.terminal.docker.image: "<your-image>"`.

### Memory provider not found

Check:

* `config.memory.provider` matches the plugin's `name` from
  `plugin.yaml`.
* `hermes plugins list` shows the plugin as enabled.
* `pip_dependencies` declared by the plugin are installed (run
  `hermes plugins install <name>`).

### Approval gate fires for safe commands

The pattern matcher in `tools/approval.py` is conservative. Add an
allow-list entry for the specific shell pattern:

```yaml
command_allowlist:
  - "git status"
  - "git diff *"
  - "/^npm (run|test|ci)$/"
```

The `/approve always` choice writes the rule for you.

### Skill conditions fail at runtime

Run `hermes skills view <name>` — the parsed `conditions` are shown.
`/skills enable <name>` will refuse to enable a skill whose conditions
are not met on this host.

### Browser tools error: "browser binary not found"

`browser_use` and `browserbase` need either Playwright browsers (run
`npx playwright install`) or a Browserbase project id. `firecrawl`
works without a local browser.

### Voice mode fails with `OSError: PortAudio not found`

`sounddevice` requires `portaudio`. macOS:
`brew install portaudio`. Linux:
`apt-get install libportaudio2`.

### Ink TUI says `gateway died`

The Python `tui_gateway.server` process exited unexpectedly. Look at
`~/.hermes/logs/tui-crash-<ts>.log` — the panic hook captured the
traceback there. Then check `~/.hermes/logs/agent.log`.

### "Unknown command" but the slash command is right

Common causes:

* The command is `cli_only` and you are in the gateway. Check
  `hermes_cli/commands.py:COMMAND_REGISTRY`.
* The command name has been migrated; check for the new name in the
  changelog.
* `aliases` was changed and your muscle memory is out of date.

`hermes_cli/commands.py:resolve_command()` is the dispatcher; if it
returns `None`, the command does not exist for the current surface.

### General debugging tips

* Set `HERMES_DEBUG=1` and re-run.
* Tail logs: `hermes logs --follow --level DEBUG`.
* Run `hermes doctor` and address red items first.
* When opening an issue, attach the redacted bundle from
  `hermes debug` — it includes versions, config, recent log
  excerpts.

## Part 3 — Quick locate index

The single biggest thing this index gets you is "given a *symptom*,
where do I look".

| Symptom | Look in |
|---------|---------|
| Agent loop hangs | `run_agent.py:run_conversation`, `tools/interrupt.py` |
| Tool errors | `model_tools.py`, `tools/registry.py`, the tool file |
| Provider request shape | `agent/transports/<api_mode>.py` and `agent/<provider>_adapter.py` |
| CLI render bug | `cli.py`, `agent/display.py`, `hermes_cli/skin_engine.py` |
| Slash dispatch | `hermes_cli/commands.py` |
| Gateway routing | `gateway/run.py`, `gateway/platforms/<name>.py` |
| Session not searchable | `hermes_state.py:SessionDB`, `tools/session_search_tool.py` |
| Compression misbehaves | `agent/context_compressor.py`, config `context.*` |
| Skill not loading | `agent/skill_commands.py`, `agent/skill_utils.py` |
| Memory not persisting | `agent/memory_manager.py`, `plugins/memory/<name>/` |
| Cron not firing | `cron/scheduler.py`, gateway must be running |
| Curator never runs | `agent/curator.py`, `~/.hermes/skills/.curator_state` |
| MCP server flaky | `tools/mcp_tool.py`, server logs |
| ACP editor disconnects | `acp_adapter/server.py`, stderr logs |
| TUI crashes | `tui_gateway/server.py`, `~/.hermes/logs/tui-crash-*.log` |
| Dashboard auth fails | `hermes_cli/web_server.py`, ephemeral token |
| Approval prompts loop | `tools/approval.py`, `command_allowlist` |
