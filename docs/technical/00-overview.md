# Hermes Agent — Technical Documentation

> Generated technical reference for the `hermes-agent` codebase (v0.12.0).
> For user-facing docs, see [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/).

This documentation set describes the internal architecture, modules and
subsystems of Hermes Agent. It is intended for contributors, packagers,
plugin authors and anyone integrating with or extending the codebase.

## Table of Contents

| Document | Topic |
|----------|-------|
| [01-architecture.md](01-architecture.md) | High-level architecture, agent loop, request lifecycle |
| [02-modules.md](02-modules.md) | Module reference — every top-level package and file |
| [03-tools.md](03-tools.md) | Tool registry, builtin tools, terminal backends |
| [04-gateway.md](04-gateway.md) | Messaging gateway, platform adapters, sessions |
| [05-state-and-memory.md](05-state-and-memory.md) | SQLite session store, memory, skills, context |
| [06-providers.md](06-providers.md) | Provider transports, adapters, credentials, rate limits |
| [07-developer-guide.md](07-developer-guide.md) | Build, test, contribute, release |

## What is Hermes Agent?

Hermes Agent is a self-improving AI agent built by [Nous Research](https://nousresearch.com).
It is distributed as a Python 3.11+ package (`hermes-agent`) and exposes three console
scripts (`pyproject.toml:132-135`):

* `hermes`     — `hermes_cli.main:main` (the user-facing CLI / TUI orchestrator)
* `hermes-agent` — `run_agent:main` (the headless agent runner / library entry)
* `hermes-acp` — `acp_adapter.entry:main` (Agent Client Protocol server for
  VS Code / Zed / JetBrains)

Functionally it is:

1. A **multi-provider tool-calling agent** (Anthropic, OpenAI-compatible,
   Bedrock, Codex, Gemini native, Gemini CloudCode, Copilot, Mistral, …)
   with a unified `ProviderTransport` abstraction.
2. A **rich terminal CLI** (prompt_toolkit-based REPL plus an optional Ink/React
   TUI in `ui-tui/`).
3. A **messaging gateway** that bridges the same agent to Telegram, Discord,
   Slack, WhatsApp, Signal, Matrix, Email, SMS, Home Assistant, DingTalk,
   WeCom, Weixin, Feishu, QQ Bot, BlueBubbles and a generic Webhook/API server.
4. A **skills + memory subsystem** with autonomous curation (the agent
   creates, refines, archives and consolidates skills in the background).
5. A **batch / RL trajectory generator** (`batch_runner.py`,
   `trajectory_compressor.py`, `mini_swe_runner.py`, the `environments/`
   Atropos package) for producing training data and running RL loops.
6. An **MCP server** (`mcp_serve.py`) that exposes Hermes' session DB and
   conversation tools to external MCP clients (Claude Code, Cursor, Codex).

## Repository at a Glance

```
hermes-agent/
├── run_agent.py            # AIAgent — core conversation loop (~14k LOC)
├── cli.py                  # HermesCLI — interactive REPL (~12k LOC)
├── model_tools.py          # Tool orchestration façade
├── toolsets.py             # Toolset definitions
├── toolset_distributions.py# Probability-weighted toolset sampling
├── hermes_state.py         # SessionDB — SQLite + FTS5
├── hermes_constants.py     # Profile-aware paths (~/.hermes)
├── hermes_logging.py       # Centralised logging
├── batch_runner.py         # Parallel trajectory generation
├── trajectory_compressor.py# Training-time trajectory compression
├── mini_swe_runner.py      # SWE-style task runner
├── mcp_serve.py            # MCP server exposing Hermes tools
├── rl_cli.py               # CLI for RL workflows
│
├── agent/                  # Agent internals (transports, adapters, memory…)
├── tools/                  # Builtin tools (40+)
│   └── environments/       # Terminal backends (local, docker, ssh, modal, …)
├── gateway/                # Messaging gateway + platform adapters
│   └── platforms/          # One module per messaging platform
├── hermes_cli/             # CLI subcommands, setup wizard, kanban, …
├── plugins/                # First-party plugins (memory, dashboard, …)
├── skills/                 # Built-in skills (active by default)
├── optional-skills/        # Heavier/niche skills (opt-in)
├── cron/                   # Scheduler (jobs.py, scheduler.py)
├── acp_adapter/            # Agent Client Protocol server
├── acp_registry/           # ACP discovery/registry
├── tui_gateway/            # Python JSON-RPC backend for the TUI
├── ui-tui/                 # Ink (React) terminal UI — `hermes --tui`
├── web/                    # Hermes dashboard SPA + API
├── website/                # Docusaurus docs site
├── environments/           # RL training environments (Atropos)
├── tests/                  # Pytest suite
└── scripts/                # run_tests.sh, release.py, install.sh, …
```

User-visible state lives under `~/.hermes/`:

* `~/.hermes/config.yaml`   — settings
* `~/.hermes/.env`          — API keys (env-file format)
* `~/.hermes/state.db`      — SQLite session store (FTS5 enabled)
* `~/.hermes/logs/`         — `agent.log`, `errors.log`, `gateway.log`
* `~/.hermes/skills/`       — agent-created and user-pinned skills
* `~/.hermes/rate_limits/`  — cross-process rate-limit state files

The location of `~/.hermes` is resolved by `hermes_constants.get_hermes_home()`
and may be overridden by `HERMES_HOME` or by an active profile (multiple
profiles share a common parent).

## How To Read This Documentation

If you want to **understand the codebase**, read the documents in order:
overview → architecture → modules → subsystem-specific docs.

If you want to **contribute a new tool**, jump to
[03-tools.md](03-tools.md) and the section on `tools/registry.py`.

If you want to **add a messaging platform**, read
[04-gateway.md](04-gateway.md) and the on-disk
`gateway/platforms/ADDING_A_PLATFORM.md`.

If you want to **add a model provider**, read the "Transports & Adapters"
chapter in [06-providers.md](06-providers.md).

If you want to **build a memory provider** or a context engine, read
[05-state-and-memory.md](05-state-and-memory.md).
