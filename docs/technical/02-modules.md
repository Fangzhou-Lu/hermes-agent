# 02 — Module Reference

This is a per-package and per-module reference for `hermes-agent`. Sizes
and short descriptions reflect the v0.12.0 layout. For end-user behaviour
see the user guide; this document is for code-level navigation.

## Top-level Python modules

| File | LOC | Purpose |
|------|----:|---------|
| `run_agent.py` | 14k | `AIAgent` class — the conversation loop, used by every surface. |
| `cli.py` | 12k | `HermesCLI` — prompt_toolkit-based interactive REPL, slash commands, streaming UI. |
| `model_tools.py` | 0.8k | Façade over `tools.registry`; exposes `get_tool_definitions`, `handle_function_call`, persistent async loops. |
| `toolsets.py` | 0.8k | `_HERMES_CORE_TOOLS` plus the `TOOLSETS` dict and `resolve_toolset()`. |
| `toolset_distributions.py` | 0.4k | Probability-weighted toolset sampling for batch-runner data generation. |
| `hermes_state.py` | 2.2k | `SessionDB` — SQLite + FTS5 conversation store. |
| `hermes_constants.py` | — | `get_hermes_home()`, profile-aware paths. |
| `hermes_logging.py` | — | `setup_logging()` with `RedactingFormatter`, session-context tagging. |
| `hermes_time.py` | — | Timezone-aware `now()` helper used everywhere. |
| `utils.py` | — | `is_truthy_value`, `env_var_enabled`, `atomic_replace`, `atomic_json_write`, `atomic_yaml_write`. |
| `batch_runner.py` | 1.3k | Parallel trajectory generation from JSONL datasets, checkpointable. |
| `trajectory_compressor.py` | 1.5k | Post-hoc trajectory compression for training-time context budgets. |
| `mini_swe_runner.py` | 0.7k | SWE-bench-style task runner using terminal backends. |
| `mcp_serve.py` | 0.9k | MCP server (stdio) exposing Hermes session tools to MCP clients. |
| `rl_cli.py` | — | CLI entry for RL/Atropos workflows. |

## `agent/` — agent internals

Pure utility functions and self-contained classes extracted from the
historical monolithic `run_agent.py`. Selected modules:

### Provider transports & adapters

| File | Purpose |
|------|---------|
| `agent/transports/base.py` | `ProviderTransport` ABC: `convert_messages`, `convert_tools`, `build_kwargs`, `normalize_response`. |
| `agent/transports/types.py` | `NormalizedResponse` and shared dataclasses across transports. |
| `agent/transports/anthropic.py` | Anthropic Messages API transport. |
| `agent/transports/chat_completions.py` | OpenAI-compatible (OpenAI, OpenRouter, llama.cpp, …). |
| `agent/transports/codex.py` | OpenAI Responses / Codex streaming transport. |
| `agent/transports/bedrock.py` | AWS Bedrock Converse API. |
| `agent/anthropic_adapter.py` | Anthropic-specific quirks: thinking budget map, max-output table, OAuth setup-tokens, Claude Code creds. |
| `agent/bedrock_adapter.py` | Bedrock Converse format translation, region resolution, client pooling. |
| `agent/codex_responses_adapter.py` | Reasoning step tracking, function-call translation, preflight validation, response normalization. |
| `agent/gemini_native_adapter.py` | Google Gemini API (free/paid tier probing, quota-error detection). |
| `agent/gemini_cloudcode_adapter.py` | Gemini via Google CloudCode endpoint (different streaming format). |
| `agent/gemini_schema.py` | Gemini-specific JSON-schema sanitisation. |
| `agent/copilot_acp_client.py` | GitHub Copilot for VS Code client. |
| `agent/google_oauth.py` | OAuth flow for Google APIs. |
| `agent/google_code_assist.py` | Google Code Assist (CloudCode) integration. |
| `agent/moonshot_schema.py` | Moonshot/Kimi schema adjustments. |
| `agent/lmstudio_reasoning.py` | LM Studio reasoning-block parsing. |

See [06-providers.md](06-providers.md) for the transport contract.

### Context, memory & curation

| File | Purpose |
|------|---------|
| `agent/context_engine.py` | `ContextEngine` ABC. Pluggable; default = `compressor`. |
| `agent/context_compressor.py` | Default engine: protect first/last N turns, summarise the middle via auxiliary model. |
| `agent/context_references.py` | Reference extraction (URLs, file paths, IDs) from history. |
| `agent/manual_compression_feedback.py` | Captures user feedback on compression quality. |
| `agent/prompt_builder.py` | System-prompt assembly: identity, platform hints, memory guidance, skills guidance, kanban, threat scanning of `.hermes.md`/`AGENTS.md`/`SOUL.md`/`.cursorrules`. |
| `agent/prompt_caching.py` | `apply_anthropic_cache_control()` — placement of up to 4 cache breakpoints. |
| `agent/auxiliary_client.py` | Cheap/fast LLM client used by curator, compressor, title generator. |
| `agent/memory_manager.py` | Orchestrator: built-in memory + at most one external provider; pre/post-turn hooks. |
| `agent/memory_provider.py` | `MemoryProvider` ABC and `BuiltinMemoryProvider`. |
| `agent/curator.py` | Background skill maintenance: pin/archive/consolidate/patch agent-created skills. |
| `agent/curator_backup.py` | Backup/restore for curator-managed state. |
| `agent/insights.py` | Insight extraction from past conversations. |
| `agent/title_generator.py` | Auto-titles for sessions. |
| `agent/onboarding.py` | First-run onboarding flow. |

### Skill plumbing

| File | Purpose |
|------|---------|
| `agent/skill_commands.py` | `/skill-name` slash-command resolution and message rewriting. |
| `agent/skill_preprocessing.py` | Inline-shell expansion, template substitution, config loading. |
| `agent/skill_utils.py` | Frontmatter parsing, condition checking, skill-index iteration, disabled filtering. |

### Auth, credentials, billing

| File | Purpose |
|------|---------|
| `agent/credential_pool.py` | `PooledCredential`, pool strategies (FILL_FIRST, ROUND_ROBIN, RANDOM, LEAST_USED), exhausted cooldown. |
| `agent/credential_sources.py` | Discovery of credentials from env vars, OAuth, keychain, config. |
| `agent/account_usage.py` | Provider account usage queries, RPM/TPM/RPH/TPH windows. |
| `agent/usage_pricing.py` | `CanonicalUsage`, `BillingRoute`, `PricingEntry`, `CostResult`. Official pricing snapshots. |
| `agent/nous_rate_guard.py` | Cross-process rate-limit guard for Nous Portal 429s. |
| `agent/rate_limit_tracker.py` | Parses `x-ratelimit-*` response headers. |

### Other agent utilities

| File | Purpose |
|------|---------|
| `agent/display.py` | Streaming-render helpers used by CLI and gateway. |
| `agent/error_classifier.py` | Maps provider errors to retry / fail-over decisions. |
| `agent/file_safety.py` | Allow-/deny-listed paths, write-guard helpers. |
| `agent/image_gen_provider.py` / `image_gen_registry.py` / `image_routing.py` | Image generation provider abstraction. |
| `agent/redact.py` | Secret redaction for logs. |
| `agent/retry_utils.py` | Tenacity-based retry decorators with provider-aware classification. |
| `agent/shell_hooks.py` | `bash`/`zsh`/`fish` hook generation for `hermes` integration. |
| `agent/subdirectory_hints.py` | Adds project-aware hints to the prompt. |
| `agent/tool_guardrails.py` | Pre-execution checks that gate certain tools. |
| `agent/trajectory.py` | `save_trajectory()`, scratchpad/think-tag conversion. |
| `agent/model_metadata.py` / `agent/models_dev.py` | Model catalog metadata. |

## `tools/` — built-in tools

Each Python file under `tools/` typically registers one or more tools at
import time via `tools.registry.register()`. The full inventory and
categorisation lives in [03-tools.md](03-tools.md). The registry itself
is at `tools/registry.py`:

* `ToolRegistry` (singleton, `tools/registry.py:143`)
* `ToolEntry` dataclass (`tools/registry.py:77-98`) — name, toolset,
  schema, handler, check_fn, requires_env, is_async, description, emoji,
  max_result_size_chars.
* `discover_builtin_tools()` (`tools/registry.py:57`) — AST-parses the
  `tools/` package to find module-level `register()` calls and imports
  only those modules.
* `get_definitions()` (`tools/registry.py:310`) — applies `check_fn`
  results with a 30-second TTL cache so dynamic availability checks
  (Docker daemon up? Modal SDK installed?) do not run on every turn.
* Shadow protection (`tools/registry.py:241-263`) — registering a
  non-MCP tool with a name already taken raises rather than silently
  overriding.

`tools/environments/` holds the terminal backends (one per execution
target). They share `BaseExecutionEnvironment` in `environments/base.py`.
See [03-tools.md](03-tools.md) "Terminal backends" for details.

`tools/browser_providers/` holds the browser-automation factory
(`base.py`, `browser_use.py`, `browserbase.py`, `firecrawl.py`).

## `gateway/` — messaging gateway

| File | Purpose |
|------|---------|
| `gateway/run.py` | `GatewayRunner` — main loop, message dispatch, AIAgent LRU cache (up to 128 agents, 1h idle TTL). |
| `gateway/config.py` | `GatewayConfig` (YAML), `Platform` enum, `SessionResetPolicy`. |
| `gateway/session.py` | `SessionSource`, session persistence (`~/.hermes/conversations/`), reset policy evaluation. |
| `gateway/session_context.py` | Dynamic context injection into the system prompt. |
| `gateway/delivery.py` | `DeliveryTarget` parsing, routing of cron/agent responses to platforms. |
| `gateway/hooks.py` | Plugin hooks into the message flow. |
| `gateway/builtin_hooks/` | Always-registered hooks (none shipped by default; reserved extension point). |
| `gateway/platform_registry.py` | Discovery of platform adapters. |
| `gateway/pairing.py` | Device pairing flows (Signal, WhatsApp, …). |
| `gateway/mirror.py` | Cross-platform message mirroring. |
| `gateway/restart.py` | Graceful shutdown coordination. |
| `gateway/status.py` | Health / status endpoints. |
| `gateway/stream_consumer.py` | Async consumption of streamed responses. |
| `gateway/channel_directory.py` | Maps channel ids to metadata. |
| `gateway/display_config.py` | Per-platform rendering hints. |
| `gateway/runtime_footer.py` | Persistent CLI footer (token counts, etc). |
| `gateway/whatsapp_identity.py` | WhatsApp identity helpers. |
| `gateway/sticker_cache.py` | Caches transcoded stickers per platform. |

`gateway/platforms/` holds one module per platform — see
[04-gateway.md](04-gateway.md) for the full list.

## `hermes_cli/` — CLI subcommands

This is the dispatcher and per-command code that the `hermes` console
script uses. Highlights:

| File | Purpose |
|------|---------|
| `hermes_cli/main.py` | Entry point. `_apply_profile_override()` pre-parses `--profile` before any state-touching imports. Dispatches to subcommands. |
| `hermes_cli/_parser.py` | argparse parser construction. |
| `hermes_cli/setup.py` | Setup wizard (`hermes setup`). |
| `hermes_cli/config.py` | YAML config loading. |
| `hermes_cli/env_loader.py` | `~/.hermes/.env` loader (python-dotenv). |
| `hermes_cli/auth.py`, `auth_commands.py` | Auth flows for providers. |
| `hermes_cli/copilot_auth.py`, `dingtalk_auth.py`, `vercel_auth.py` | Per-provider OAuth. |
| `hermes_cli/azure_detect.py` | Azure environment detection. |
| `hermes_cli/banner.py`, `cli_output.py`, `colors.py`, `skin_engine.py`, `tips.py` | Visual presentation. |
| `hermes_cli/callbacks.py` | Streaming callbacks routed to the renderer. |
| `hermes_cli/clipboard.py` | Cross-platform clipboard. |
| `hermes_cli/claw.py` | OpenClaw migration (`hermes claw migrate`). |
| `hermes_cli/cron.py` | Cron CLI (`hermes cron list/add/edit/delete`). |
| `hermes_cli/curator.py` | Curator CLI (`hermes curator`). |
| `hermes_cli/curses_ui.py` | Fallback curses-based menu. |
| `hermes_cli/debug.py`, `doctor.py` | Diagnostics. |
| `hermes_cli/dump.py` | Conversation dump. |
| `hermes_cli/fallback_cmd.py` | Unknown-command handler. |
| `hermes_cli/gateway.py` | `hermes gateway start/setup/stop`. |
| `hermes_cli/goals.py` | Long-running goals. |
| `hermes_cli/hooks.py` | Shell-hook installation. |
| `hermes_cli/kanban.py`, `kanban_db.py` | Kanban CLI + storage. |
| `hermes_cli/logs.py` | `hermes logs [--follow] [--level] [--session]`. |
| `hermes_cli/mcp_config.py` | MCP server configuration. |
| `hermes_cli/memory_setup.py` | Memory provider setup. |
| `hermes_cli/model_catalog.py`, `model_normalize.py`, `model_switch.py`, `models.py`, `codex_models.py` | Model catalog & switching. |
| `hermes_cli/oneshot.py` | `hermes -p "prompt"` non-interactive mode. |
| `hermes_cli/pairing.py` | `hermes pairing` device pairing UI. |
| `hermes_cli/platforms.py` | Platform listing. |
| `hermes_cli/plugins.py`, `plugins_cmd.py` | Plugin discovery and CLI. |
| `hermes_cli/profiles.py` | Profile management. |
| `hermes_cli/providers.py` | Provider enumeration. |
| `hermes_cli/pty_bridge.py` | PTY bridging for terminal multiplexing. |
| `hermes_cli/relaunch.py` | In-place relaunch after `hermes update`. |
| `hermes_cli/runtime_provider.py` | Runtime selection. |
| `hermes_cli/skills_config.py`, `skills_hub.py` | Skill management. |
| `hermes_cli/slack_cli.py` | Slack helper. |
| `hermes_cli/status.py` | `hermes status`. |
| `hermes_cli/timeouts.py` | Timeout configuration. |
| `hermes_cli/tools_config.py` | Toolset configuration. |
| `hermes_cli/uninstall.py` | `hermes uninstall`. |
| `hermes_cli/web_server.py` | Local dashboard SPA + API server. |
| `hermes_cli/webhook.py` | Webhook utilities. |
| `hermes_cli/voice.py` | Voice mode helpers. |
| `hermes_cli/default_soul.py` | Default `SOUL.md` template. |
| `hermes_cli/completion.py` | Shell completion script generation. |
| `hermes_cli/commands.py` | Top-level subcommand registry. |

## `cron/` — built-in scheduler

| File | Purpose |
|------|---------|
| `cron/scheduler.py` | `tick()` invoked every 60s by the gateway. File lock at `~/.hermes/cron/.tick.lock` prevents overlapping ticks. Cross-platform (`fcntl` on Unix, `msvcrt` on Windows). |
| `cron/jobs.py` | Job storage (`~/.hermes/cron/jobs.json`), output to `~/.hermes/cron/output/{job_id}/{timestamp}.md`. |

A job has an agent config (model, toolset, system prompt) and a delivery
target (origin, local, or `platform:chat_id`). Delivery is performed via
`gateway.delivery.DeliveryRouter` so a cron job can run inside a CLI
session and post its result to Telegram.

## `acp_adapter/` — Agent Client Protocol server

| File | Purpose |
|------|---------|
| `acp_adapter/__main__.py`, `entry.py` | CLI entry. Loads `~/.hermes/.env`; logs go to stderr (stdout is the JSON-RPC channel). |
| `acp_adapter/server.py` | ACP server, session management, tool completion. |
| `acp_adapter/auth.py` | Auth-provider detection (API key, OAuth). |
| `acp_adapter/events.py` | ACP event callbacks (message, thinking, step, tool progress). |
| `acp_adapter/permissions.py` | Approval gates for high-risk operations. |
| `acp_adapter/session.py` | Per-ACP-session bookkeeping. |
| `acp_adapter/tools.py` | Tool start/complete message construction for ACP schema. |

## `acp_registry/` — ACP discovery manifest

Holds `agent.json` and `icon.svg` so editors that read the ACP registry
manifest can advertise Hermes as an installed agent.

## `tui_gateway/` — Python JSON-RPC backend for the Ink TUI

| File | Purpose |
|------|---------|
| `tui_gateway/server.py` | Main gateway server with crash hooks. |
| `tui_gateway/transport.py` | Stdio / WebSocket transport abstraction. |
| `tui_gateway/ws.py` | WebSocket transport. |
| `tui_gateway/event_publisher.py` | Event streaming to TUI. |
| `tui_gateway/render.py` | TUI rendering helpers. |
| `tui_gateway/slash_worker.py` | Background worker for slash commands. |
| `tui_gateway/entry.py` | CLI entry point. |

## `ui-tui/` — Ink (React) terminal UI

A monorepo (`packages/`) of TypeScript components plus a top-level Vite
project. Started by `hermes --tui`. Talks to `tui_gateway` over stdio
(or WebSocket when running detached). See `ui-tui/README.md`.

## `web/` — Hermes dashboard

A Vite/TypeScript SPA. The Python entry point is
`hermes_cli.web_server` (FastAPI + uvicorn — pulled in via the `[web]`
extra). Uses the same SessionDB.

## `website/` — Docusaurus docs site

Built and published to `https://hermes-agent.nousresearch.com/docs/`.

## `environments/` — RL training environments

Atropos-style RL environments built on Hermes Agent. Notable files:

| File | Purpose |
|------|---------|
| `environments/hermes_base_env.py` | Base RL environment using AIAgent + terminal backends. |
| `environments/agent_loop.py` | Loop that produces trajectories. |
| `environments/agentic_opd_env.py` | Open-Procedural-Dialog environment. |
| `environments/web_research_env.py` | Web-research RL environment. |
| `environments/hermes_swe_env/` | SWE-bench-style environment. |
| `environments/terminal_test_env/` | Synthetic terminal task env. |
| `environments/benchmarks/` | Benchmark drivers. |
| `environments/tool_call_parsers/` | Parsers for non-OpenAI tool-call formats. |
| `environments/tool_context.py` | Tool execution context shared across envs. |
| `environments/patches.py` | Patch utilities. |
| `environments/README.md` | Env documentation. |

The `[rl]` extra installs `atroposlib`, `tinker`, `fastapi`, `uvicorn`,
`wandb`.

## `plugins/` — first-party plugins

A plugin is a Python package under `plugins/<name>/` with a
`plugin.yaml` manifest describing its `name`, `version`, `description`,
`pip_dependencies` and `hooks`. Example
(`plugins/memory/honcho/plugin.yaml`):

```yaml
name: honcho
version: 1.0.0
description: "Honcho AI-native memory — cross-session user modeling …"
pip_dependencies:
  - honcho-ai
hooks:
  - on_session_end
```

Shipped plugin categories:

* `plugins/context_engine/` — pluggable context engines (default:
  `compressor`; alternative: `lcm`).
* `plugins/memory/` — memory providers: `byterover`, `hindsight`,
  `holographic`, `honcho`, `mem0`, `openviking`, `retaindb`, `supermemory`.
* `plugins/platforms/` — additional platform adapters (`irc`, `teams`).
* `plugins/disk-cleanup/`, `plugins/example-dashboard/`,
  `plugins/google_meet/`, `plugins/hermes-achievements/`,
  `plugins/image_gen/`, `plugins/kanban/` (`dashboard/`, `systemd/`),
  `plugins/observability/`, `plugins/spotify/`,
  `plugins/strike-freedom-cockpit/`.

The minimal loader lives in `plugins/__init__.py`; each plugin declares
its tools via `plugin.yaml`, and the gateway discovers them at startup.

## `skills/` — built-in skills (active by default)

Top-level categories: `apple/`, `autonomous-ai-agents/`, `creative/`,
`data-science/`, `devops/`, `diagramming/`, `dogfood/`, `domain/`,
`email/`, `gaming/`, `gifs/`, `github/`, `inference-sh/`, `mcp/`,
`media/`, `mlops/`, `note-taking/`, `productivity/`, `red-teaming/`,
`research/`, `smart-home/`, `social-media/`, `software-development/`,
`yuanbao/`. The `index-cache/` directory holds Skills-Hub index
metadata.

A skill is a directory containing `SKILL.md` (frontmatter + Markdown)
and optional `references/`, `templates/`, `assets/` subdirectories. See
[03-tools.md](03-tools.md) and [05-state-and-memory.md](05-state-and-memory.md).

## `optional-skills/` — opt-in skills

Heavier or niche skills shipped with the repo but not active by
default. Categories: `autonomous-ai-agents/`, `blockchain/`,
`communication/`, `creative/`, `devops/`, `dogfood/`, `email/`,
`health/`, `mcp/`, `migration/`, `mlops/`, `productivity/`, `research/`,
`security/`, `web-development/`. Activated via `hermes skills install`
or the Hub UI.

## `tinker-atropos/`, `datagen-config-examples/`, `cli-config.yaml.example`

Datagen scaffolding and example configurations used by RL workflows.

## `scripts/`

| Script | Purpose |
|--------|---------|
| `scripts/install.sh`, `install.cmd`, `install.ps1` | Cross-platform installers. |
| `scripts/run_tests.sh` | Test entry — probes `.venv`, `venv`, `~/.hermes/hermes-agent/venv`. |
| `scripts/build_model_catalog.py` | Builds the bundled model catalog. |
| `scripts/build_skills_index.py` | Builds the bundled skills index. |
| `scripts/contributor_audit.py` | Contributor metrics. |
| `scripts/discord-voice-doctor.py` | Discord voice diagnostics. |
| `scripts/kill_modal.sh` | Convenience: kill stray Modal sandboxes. |
| `scripts/profile-tui.py` | Profiler harness for the TUI. |
| `scripts/release.py` | Release packaging. |
| `scripts/sample_and_compress.py` | Runs `batch_runner` + `trajectory_compressor` end-to-end. |
| `scripts/hermes-gateway/`, `whatsapp-bridge/` | Helper deployment scripts. |
| `scripts/lib/node-bootstrap.sh` | Node bootstrap helper. |

## `tests/`

Pytest suite. Organised by subsystem (`tests/agent/`, `tests/cli/`,
`tests/gateway/`, `tests/run_agent/`, `tests/tools/`, `tests/skills/`,
…) plus topic-specific test modules at the top level
(`tests/test_*.py`). Marker `integration` is excluded by default
(`pyproject.toml:147-151`); `pytest-xdist` runs in parallel
(`addopts = "-m 'not integration' -n auto"`).

## `nix/`, `flake.nix`, `flake.lock`

Nix flake for reproducible installs; `nix/` holds the derivation.

## `packaging/`

Distribution packaging assets (RPM/DEB/Homebrew formula scaffolding).

## `docker/`, `Dockerfile`, `docker-compose.yml`

Container images for self-hosted deployments. The Dockerfile installs
the `[all]` extra and the `hermes` script.

## `.github/`, `.envrc`, `.mailmap`, `.gitattributes`

Standard repo metadata. `.envrc` is for `direnv` users.
