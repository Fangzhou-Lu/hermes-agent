# Hermes Agent — Technical Documentation

> Generated technical reference for the `hermes-agent` codebase (v0.12.0).
> For user-facing docs, see [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/).

This documentation set describes the internal architecture, modules and
subsystems of Hermes Agent. It is intended for contributors, packagers,
plugin authors and anyone integrating with or extending the codebase.

## Table of Contents

| # | Document | Topic |
|---|----------|-------|
| 00 | [00-overview.md](00-overview.md) | This file. Map of the documentation set. |
| 01 | [01-architecture.md](01-architecture.md) | High-level architecture, agent loop, request lifecycle |
| 02 | [02-modules.md](02-modules.md) | Module reference — every top-level package and file |
| 03 | [03-tools.md](03-tools.md) | Tool registry, builtin tools, terminal backends, skills (overview) |
| 04 | [04-gateway.md](04-gateway.md) | Messaging gateway, platform adapters, sessions |
| 05 | [05-state-and-memory.md](05-state-and-memory.md) | SQLite session store, memory, skills, context engine, curator |
| 06 | [06-providers.md](06-providers.md) | Provider transports, adapters, credentials, rate limits, billing |
| 07 | [07-developer-guide.md](07-developer-guide.md) | Build, test, contribute, release |
| 08 | [08-cli-reference.md](08-cli-reference.md) | Every CLI subcommand, every slash command, keybindings, autocomplete |
| 09 | [09-skills-system.md](09-skills-system.md) | Skill format, index, Hub, guard, sync, curator (deep dive) |
| 10 | [10-batch-and-rl.md](10-batch-and-rl.md) | Batch runner, trajectory compressor, mini SWE runner, Atropos envs |
| 11 | [11-mcp-and-acp.md](11-mcp-and-acp.md) | MCP client + server, ACP server, comparison |
| 12 | [12-tui-and-dashboard.md](12-tui-and-dashboard.md) | Ink TUI, `tui_gateway`, web dashboard, PTY bridge |
| 13 | [13-security.md](13-security.md) | Threat model, approvals, path safety, prompt-injection scanning, redaction |
| 14 | [14-configuration-reference.md](14-configuration-reference.md) | Complete `config.yaml` and `.env` reference |
| 15 | [15-data-flow.md](15-data-flow.md) | ASCII sequence diagrams for the most common flows |
| 16 | [16-extension-recipes.md](16-extension-recipes.md) | 25 cookbook recipes (new tool, new provider, new platform, etc.) |
| 17 | [17-glossary-and-troubleshooting.md](17-glossary-and-troubleshooting.md) | Terminology + common failure modes + quick locate index |
| 18 | [18-platforms-deep-dive.md](18-platforms-deep-dive.md) | Per-platform reference for all 22 adapters |
| 19 | [19-testing-and-observability.md](19-testing-and-observability.md) | Test infrastructure, test categories, metrics, logging, tracing |
| 20 | [20-voice-and-image.md](20-voice-and-image.md) | Voice mode (STT/TTS/coordination) and image generation pipeline |
| 21 | [21-browser-tools.md](21-browser-tools.md) | Browser tools, CDP, providers, anti-bot, lifecycle |
| 22 | [22-delegation-and-subagents.md](22-delegation-and-subagents.md) | Delegation, mixture of agents, clarify, sub-agent observability |
| 23 | [23-patterns-and-antipatterns.md](23-patterns-and-antipatterns.md) | Coding patterns / anti-patterns the codebase enforces |
| 24 | [24-adapters-deep-dive.md](24-adapters-deep-dive.md) | Per-adapter (Anthropic / Codex / Bedrock / Gemini / …) deep dive |
| 25 | [25-fakes-and-test-harness.md](25-fakes-and-test-harness.md) | `tests/fakes/` manual + fixture composition patterns |
| zh | [README-zh.md](README-zh.md) | 中文导航索引 |

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

Read in numerical order if you are new to the codebase: 00 → 01 → 02 →
… 19. Most documents stand alone, but the architecture and module-reference
chapters provide the vocabulary the rest of the docs use.

If you are diving into a specific subsystem:

* **Adding a tool** → [03-tools.md](03-tools.md), [16-extension-recipes.md](16-extension-recipes.md) (Recipe 1).
* **Adding a provider** → [06-providers.md](06-providers.md), [16-extension-recipes.md](16-extension-recipes.md) (Recipes 2-3).
* **Adding a platform** → [04-gateway.md](04-gateway.md), [18-platforms-deep-dive.md](18-platforms-deep-dive.md), [16-extension-recipes.md](16-extension-recipes.md) (Recipe 4).
* **Memory provider / context engine** → [05-state-and-memory.md](05-state-and-memory.md), [16-extension-recipes.md](16-extension-recipes.md) (Recipes 5-6).
* **CLI / slash command work** → [08-cli-reference.md](08-cli-reference.md), [16-extension-recipes.md](16-extension-recipes.md) (Recipe 9).
* **Cron / scheduling** → [04-gateway.md](04-gateway.md) §9, [16-extension-recipes.md](16-extension-recipes.md) (Recipe 10).
* **MCP server work (client or server)** → [11-mcp-and-acp.md](11-mcp-and-acp.md).
* **Editor integration via ACP** → [11-mcp-and-acp.md](11-mcp-and-acp.md), [12-tui-and-dashboard.md](12-tui-and-dashboard.md).
* **Trajectory pipelines** → [10-batch-and-rl.md](10-batch-and-rl.md).
* **Diagnosing a problem** → [17-glossary-and-troubleshooting.md](17-glossary-and-troubleshooting.md).
* **Test / CI / metrics work** → [19-testing-and-observability.md](19-testing-and-observability.md).
* **Auditing security** → [13-security.md](13-security.md).
* **Anything else** → [02-modules.md](02-modules.md) is the index; jump
  from the relevant package or file.

## Conventions used in this documentation

* **Path references** use the form `<package>/<file>.py:<line>` (e.g.
  `tools/registry.py:143`). Lines were correct at the time of
  generation; large refactors may shift them.
* **Code samples** are illustrative rather than copy-pastable unless
  marked otherwise. They reflect the project's style (no top-level
  comments, prefer dataclasses, all tool handlers return JSON strings).
* **CLI samples** assume the user has activated the venv (or is using
  the `./hermes` wrapper).
* **Diagrams** are ASCII so they render in any viewer including
  terminal `less`.

## Document size

This documentation set is intentionally large — it is meant to be a
*reference* you grep, not a tutorial you read end-to-end. The full
set covers ~10 000 lines across 20 files. If you only have time for
one chapter, read [01-architecture.md](01-architecture.md). If you
only need a quick reminder of where something lives, jump straight to
[17-glossary-and-troubleshooting.md](17-glossary-and-troubleshooting.md)
("Quick locate index" at the end).

## Feedback

Found something wrong, out of date, or missing? Open an issue or PR
on GitHub. Contributors welcome — the entire technical doc set lives
under `docs/technical/` and is plain Markdown.

