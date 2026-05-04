# 08 — CLI Reference

This document is the comprehensive reference for `hermes`, the
console-script CLI. It covers every subcommand, every slash command in
the interactive REPL, the keybindings, the autocomplete system, and the
extension points that drive them all.

Use this together with [03-tools.md](03-tools.md) (for the `/tools` and
`/skills` slash commands), [05-state-and-memory.md](05-state-and-memory.md)
(for `/insights`, `/compress`, `/usage`), and
[06-providers.md](06-providers.md) (for `/model`, `/personality`,
`/reasoning`, `/fast`).

## 1. Console scripts

`pyproject.toml` declares three entry points (`pyproject.toml:132-135`):

```toml
[project.scripts]
hermes = "hermes_cli.main:main"
hermes-agent = "run_agent:main"
hermes-acp = "acp_adapter.entry:main"
```

* **`hermes`** — the user-facing CLI / TUI orchestrator. All `hermes
  <subcommand>` invocations pass through `hermes_cli.main:main`, which
  dispatches via `hermes_cli/_parser.py`.
* **`hermes-agent`** — the headless agent runner. `run_agent.main()`
  wires up `AIAgent` with `fire`-based CLI args; primarily used by
  scripts and integrations.
* **`hermes-acp`** — the Agent Client Protocol server. Loads
  `~/.hermes/.env`, then runs `acp_adapter.server.run()`. All logging
  goes to stderr because stdout is the JSON-RPC channel.

In addition, the repo ships `./hermes` — a Bash wrapper that auto-detects
the venv, then `exec`s `hermes` with the same arguments.

## 2. `hermes` subcommand catalogue

Subcommands are routed by `hermes_cli/_parser.py` and the per-command
modules under `hermes_cli/`. Every subcommand respects the global
`--profile <name>` flag (handled by `_apply_profile_override` *before*
imports run), so `~/.hermes` becomes `~/.hermes-<name>` for the duration
of the call.

### `hermes` (no subcommand)

Starts the interactive CLI. The default flow:

1. Parse global flags (`--profile`, `--tui`, `-p/--prompt`, `--model`,
   `--quiet`, …).
2. Apply profile override (`hermes_cli/main.py:_apply_profile_override`).
3. Load `~/.hermes/config.yaml` via `hermes_cli/config.py`
   (`load_config()` deep-merges over `DEFAULT_CONFIG`).
4. Load `~/.hermes/.env` via `hermes_cli.env_loader.load_hermes_dotenv`.
5. If first run, jump to `hermes_cli/setup.py`.
6. Otherwise, instantiate `cli.HermesCLI(...)` and start the REPL.

If the user passes `-p "<prompt>"` (alias `--prompt`), `hermes_cli/oneshot.py`
takes over: it spins up a `AIAgent`, sends the single message, prints
the final response and exits. This is the non-interactive mode used by
shell pipelines and `cron` entries.

If the user passes `--tui` or sets `HERMES_TUI=1`, the Ink/React TUI
under `ui-tui/` is launched instead via `hermes_cli/main:_run_tui()`,
which `exec`s Node with the bundled JS (`hermes_cli/web_dist/`).

### `hermes setup`

The setup wizard. Implemented in `hermes_cli/setup.py`. Steps:

1. **OpenClaw migration check** — if `~/.openclaw` exists, offer to
   `hermes claw migrate` first.
2. **Provider selection** — chooses between Nous Portal, OpenRouter,
   Anthropic, OpenAI, Google, custom OpenAI-compatible, etc.
3. **Auth** — runs the relevant OAuth flow or asks for an API key.
4. **Model pick** — narrowed to those available for the chosen provider.
5. **Personality** — optional choice from `config.personalities` or
   `assets/personalities/`.
6. **Toolset** — picks an enabled toolset. Defaults to `hermes-cli`.
7. **Memory provider** — opt-in Honcho or other plugin.
8. **Cron** — opt-in install of a cron daemon.
9. **Optional extras** — skill installs, MCP server registration.
10. **Summary** — shows the final `~/.hermes/config.yaml` diff and the
    `.env` keys it has set.

### `hermes model [provider:model]`

Switches the active model. Without arguments, opens an interactive
picker driven by `hermes_cli/model_switch.py` and
`hermes_cli/model_catalog.py`. The catalog is precomputed by
`scripts/build_model_catalog.py` and bundled with the wheel; runtime
lookups use `hermes_cli/model_normalize.py` to resolve aliases (e.g.
`opus` → `anthropic:claude-opus-4-7`).

### `hermes tools`

Opens an interactive toolset / per-tool toggle UI
(`hermes_cli/tools_config.py`). Stores the result in
`config.toolsets` (list of enabled toolsets) and
`config.disabled_tools` (per-tool overrides).

### `hermes config <get|set|unset|edit> <key> [value]`

Read or modify `~/.hermes/config.yaml`. `set` accepts dotted paths
(`agent.max_turns 120`). `edit` opens the file in `$EDITOR`.

### `hermes gateway <subcommand>`

Implemented in `hermes_cli/gateway.py`. Subcommands:

| Command | Purpose |
|---------|---------|
| `gateway setup` | Interactive wizard: pick which platforms to enable and configure secrets per platform. |
| `gateway start` | Launch `gateway.run.GatewayRunner` in the foreground. |
| `gateway run`   | Alias for `start`. |
| `gateway stop`  | Send SIGTERM to a running gateway (looks up the PID file). |
| `gateway status` | Query the running gateway's `/status` endpoint. |
| `gateway logs`  | Tail `gateway.log`. |
| `gateway restart` | Graceful drain + restart (`gateway/restart.py`). |

### `hermes claw <subcommand>`

OpenClaw migration (`hermes_cli/claw.py`). Subcommands:

| Command | Purpose |
|---------|---------|
| `claw migrate` | Interactive migration. Default preset = full. |
| `claw migrate --dry-run` | Preview what would be migrated. |
| `claw migrate --preset user-data` | Skip secrets. |
| `claw migrate --preset secrets-only` | Only API keys / tokens. |
| `claw migrate --overwrite` | Overwrite conflicts. |
| `claw migrate --workspace-target <dir>` | Where to put `AGENTS.md`. |

### `hermes update`

Implemented in `hermes_cli/relaunch.py`. Detects how Hermes was
installed (pip, uv, brew, nix, …) and runs the appropriate upgrade
command. After upgrade, gracefully relaunches the CLI in place so the
session continues with the new code.

### `hermes doctor`

Health check (`hermes_cli/doctor.py`). Categories:

* Python version, OS, architecture.
* `~/.hermes` layout: config readable, state.db exists and is WAL.
* Provider auth: which credentials are loaded, which are exhausted.
* Tool availability: which `check_fn` results are passing/failing.
* Terminal backends: which are usable on this host.
* Logs: tail of `errors.log` (last 5 lines).

### `hermes status`

Compact one-screen status (`hermes_cli/status.py`):

```
Hermes Agent v0.12.0
Profile: default (~/.hermes)
Model:   anthropic:claude-opus-4-7
Tools:   hermes-cli (47 tools)
Gateway: running (telegram, discord)  PID 12345
Cron:    3 jobs (1 due in 4m)
```

### `hermes logs [--follow] [--level <l>] [--session <id>]`

Tail-and-grep helper (`hermes_cli/logs.py`). Reads from
`~/.hermes/logs/agent.log` plus `errors.log` and `gateway.log` if
present.

### `hermes session <list|show|delete|export>`

Lightweight session manager (`hermes_cli/dump.py` for export). Uses
`SessionDB`. `export` writes a Markdown transcript.

### `hermes mcp <add|remove|list|enable|disable>`

MCP server management (`hermes_cli/mcp_config.py`). Stores entries in
`~/.hermes/mcp_servers.yaml`. Each entry has:

```yaml
servers:
  github:
    transport: stdio
    command: ["npx", "-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_TOKEN: ${GITHUB_TOKEN}
    enabled: true
  fly:
    transport: http
    url: https://api.fly.io/mcp
    auth:
      kind: oauth
      provider: fly
```

`tools/mcp_tool.py` reads this file at agent startup and connects the
servers, registering their tools under `toolset = "mcp-<name>"`.

### `hermes plugins <list|enable|disable|install|remove>`

Plugin lifecycle (`hermes_cli/plugins_cmd.py`):

| Command | Purpose |
|---------|---------|
| `plugins list` | Show installed plugins with status (enabled / disabled / broken). |
| `plugins enable <name>` | Set `plugins.enabled.<name> = true`. |
| `plugins disable <name>` | Set `plugins.enabled.<name> = false`. |
| `plugins install <name>` | Run the plugin's `pip_dependencies`, then enable. |
| `plugins remove <name>` | Disable; offer to remove the package. |
| `plugins reload` | Re-scan `plugins/` and `~/.hermes/plugins/`. |

### `hermes skills <subcommand>`

The skill manager (`hermes_cli/skills_config.py`,
`hermes_cli/skills_hub.py`). Subcommands:

| Command | Purpose |
|---------|---------|
| `skills list` | Print all installed skills, grouped by source. |
| `skills enable <name>` / `skills disable <name>` | Toggle. |
| `skills install <ref>` | Install from Hub or `gh:owner/repo[/path]`. |
| `skills uninstall <name>` | Remove an installed skill. |
| `skills browse` | Interactive browser for discoverable skills. |
| `skills tap add <url>` / `skills tap remove <url>` | Add/remove a Hub source URL. |
| `skills sync` | Push agent-curated skills to the configured Hub. |
| `skills audit` | Print the `audit.log` entries. |
| `skills index <rebuild>` | Rebuild the local index cache. |

### `hermes cron <list|add|edit|delete|run>`

`hermes_cli/cron.py`. Wraps `cron.jobs` to manipulate
`~/.hermes/cron/jobs.json`. `cron run <id>` triggers a single execution
out of band (useful for testing).

### `hermes curator <status|run|pause|resume|pin <skill>|archive <skill>>`

`hermes_cli/curator.py`. The agent's autonomous skill maintenance.
`curator status` shows the curator state file
(`~/.hermes/skills/.curator_state`); `curator run` forces a manual pass.

### `hermes kanban <board|task|comment|...>`

`hermes_cli/kanban.py`. Multi-profile collaboration board
backed by `hermes_cli/kanban_db.py` (its own SQLite DB at
`~/.hermes/kanban.db`).

### `hermes pairing`

`hermes_cli/pairing.py`. Device pairing for platforms that need it
(Signal, WhatsApp). Renders QR codes via `qrcode` (pulled in by the
`messaging` extra).

### `hermes voice <enable|disable>`

`hermes_cli/voice.py`. Toggles voice mode integration. The actual STT
runs via `tools/transcription_tools.py` (faster-whisper) and TTS via
`tools/tts_tool.py` (Edge TTS by default; ElevenLabs with `tts-premium`).

### `hermes dashboard`

`hermes_cli/web_server.py`. Launches the FastAPI + Vite dashboard. It
serves `web/` on the configured port (`config.dashboard.port`, default
9119) and embeds the live TUI through a PTY-backed WebSocket. See
[12-tui-and-dashboard.md](12-tui-and-dashboard.md).

### `hermes acp`

`hermes_cli/main.py` proxy to `acp_adapter.entry:main`. Used by editors
that load Hermes through ACP. `hermes-acp` is the same entry point as a
top-level console script.

### `hermes uninstall`

`hermes_cli/uninstall.py`. Walks the user through:

1. Backing up `~/.hermes`.
2. Removing the installed package (via the detected manager).
3. Optionally deleting `~/.hermes`.

### Misc subcommands

| Command | File |
|---------|------|
| `hermes auth <provider>` | `hermes_cli/auth_commands.py` — manual OAuth/API-key entry. |
| `hermes copilot login` | `hermes_cli/copilot_auth.py` — GitHub Copilot OAuth. |
| `hermes vercel <login|deploy>` | `hermes_cli/vercel_auth.py` |
| `hermes dingtalk login` | `hermes_cli/dingtalk_auth.py` |
| `hermes nous <subscription>` | `hermes_cli/nous_subscription.py` |
| `hermes goals <list|add|complete>` | `hermes_cli/goals.py` |
| `hermes profiles <list|create|delete|use>` | `hermes_cli/profiles.py` |
| `hermes platforms` | `hermes_cli/platforms.py` |
| `hermes providers` | `hermes_cli/providers.py` |
| `hermes models <list>` | `hermes_cli/models.py` |
| `hermes hooks install <shell>` | `hermes_cli/hooks.py` |
| `hermes completion <bash|zsh|fish>` | `hermes_cli/completion.py` |
| `hermes tips` | `hermes_cli/tips.py` |
| `hermes skin` | `hermes_cli/skin_engine.py` |
| `hermes timeouts` | `hermes_cli/timeouts.py` |
| `hermes runtime` | `hermes_cli/runtime_provider.py` |
| `hermes browser connect` | `hermes_cli/browser_connect.py` |

## 3. Slash command registry

The single source of truth is `COMMAND_REGISTRY` in
`hermes_cli/commands.py`. The dataclass:

```python
@dataclass(frozen=True)
class CommandDef:
    name: str
    description: str
    category: str                       # "Session", "Configuration",
                                        # "Tools & Skills", "Info", "Exit"
    aliases: tuple[str, ...] = ()
    args_hint: str = ""
    subcommands: tuple[str, ...] = ()
    cli_only: bool = False
    gateway_only: bool = False
    gateway_config_gate: str | None = None
```

There are **61** `CommandDef` entries today (run
`grep -c CommandDef /home/user/hermes-agent/hermes_cli/commands.py` to
verify). The list below is grouped by category.

### Category: "Session" (in-conversation actions)

| Slash | Aliases | Args | Notes |
|-------|---------|------|-------|
| `/new` | `/reset` | — | Start a new session id. |
| `/clear` | — | — | CLI-only. Clears the screen + new session. |
| `/redraw` | — | — | CLI-only. Force full UI repaint. |
| `/history` | — | — | CLI-only. Show the conversation. |
| `/save` | — | — | CLI-only. Save the conversation. |
| `/retry` | — | — | Resend the last user message. |
| `/undo` | — | — | Remove the last user/assistant exchange. |
| `/title` | — | `[name]` | Rename the session. |
| `/branch` | `/fork` | `[name]` | Branch off into a new session. |
| `/compress` | — | `[focus]` | Manual compression pass. |
| `/rollback` | — | `[number]` | Restore a filesystem checkpoint. |
| `/snapshot` | `/snap` | `[create|restore <id>|prune]` | CLI-only state snapshot. |
| `/stop` | — | — | Kill background processes. |
| `/approve` | — | `[session|always]` | Gateway-only approve flow. |
| `/deny` | — | — | Gateway-only deny flow. |
| `/background` | `/bg` `/btw` | `<prompt>` | Run a prompt in the background. |
| `/agents` | `/tasks` | — | Show active sub-agents. |
| `/queue` | `/q` | `<prompt>` | Queue for the next turn. |
| `/steer` | — | `<prompt>` | Inject a message after the next tool call. |
| `/goal` | — | `<prompt>` | Standing goal across turns. |
| `/status` | — | — | Show session info. |
| `/sethome` | — | — | Mark this chat as the default delivery target. |
| `/resume` | — | — | Resume a previously-named session. |
| `/restart` | — | — | Drain & restart the gateway. |

### Category: "Configuration"

| Slash | Aliases | Args | Notes |
|-------|---------|------|-------|
| `/config` | — | — | Show current config. |
| `/model` | — | `[provider:model]` | Switch model. |
| `/personality` | — | `[name]` | Apply a personality. |
| `/statusbar` | — | — | Toggle the context/model bar. |
| `/verbose` | — | — | Cycle tool-progress display: off → new → all → verbose. |
| `/footer` | — | — | Toggle gateway runtime footer. |
| `/yolo` | — | — | Skip dangerous-command approvals (use with care). |
| `/reasoning` | — | `<auto|low|medium|high|xhigh|off> [<show|hide>]` | Reasoning effort and display. |
| `/fast` | — | — | Toggle Anthropic Fast Mode / OpenAI Priority Processing. |
| `/skin` | — | `[name]` | Theme. |
| `/indicator` | — | — | TUI busy-indicator style. |
| `/voice` | — | — | Toggle voice mode. |
| `/busy` | — | — | What Enter does while Hermes is working. |

### Category: "Tools & Skills"

| Slash | Args | Notes |
|-------|------|-------|
| `/tools` | `[list|disable|enable] [name...]` | Manage tools per session. |
| `/toolsets` | — | List available toolsets. |
| `/skills` | `[list|search|view|install|...]` | Skill management. |
| `/<skill-name>` | `[args]` | Invoke a skill directly (e.g. `/test-driven-development`). |
| `/cron` | `[list|add|run]` | Cron management. |
| `/curator` | `[status|run|pin|archive]` | Curator. |
| `/kanban` | `[board|task]` | Kanban. |
| `/reload` | — | Reload `.env`. |
| `/reload-mcp` | — | Reload MCP servers. |
| `/reload-skills` | — | Re-scan `~/.hermes/skills/`. |
| `/browser` | — | Connect browser tools to a live Chrome via CDP. |
| `/plugins` | — | List plugins. |

### Category: "Info"

| Slash | Args | Notes |
|-------|------|-------|
| `/profile` | — | Show active profile + home dir. |
| `/gquota` | — | Google Code Assist quota usage. |
| `/commands` | — | Browse all commands & skills. |
| `/help` | — | Show help. |
| `/usage` | — | Token usage and rate limits. |
| `/insights` | `[--days N]` | Cross-session insights. |
| `/platforms` | — | Gateway/messaging platform status. |
| `/copy` | — | Copy last response to clipboard. |
| `/paste` | — | Attach clipboard image. |
| `/image` | `[path]` | Attach a local image. |
| `/update` | — | Update Hermes. |
| `/debug` | — | Upload debug bundle. |

### Category: "Exit"

| Slash | Aliases | Notes |
|-------|---------|-------|
| `/quit` | `/exit` `/q!` | Exit the CLI. |

## 4. Slash command lifecycle

```
user types /foo bar
        │
        ▼
HermesCLI._on_input()              # cli.py
        │
        ▼
resolve_command("foo")             # hermes_cli/commands.py
        │
        ▼
HermesCLI.process_command(...)     # cli.py — dispatch on canonical name
        │
        ├── built-in handler        # one per CommandDef in cli.py
        │
        └── /<skill-name>  →  agent.skill_commands._build_skill_message()
                              and feed into AIAgent as a user turn
```

The same `resolve_command()` is used by `gateway/run.py:_handle_message()`
for slash dispatch on platforms; `GATEWAY_KNOWN_COMMANDS` is computed by
filtering `COMMAND_REGISTRY` to those that are not `cli_only` (or whose
`gateway_config_gate` is set and resolves truthy).

`resolve_command()` (`hermes_cli/commands.py:222`) walks the
`_COMMAND_LOOKUP` map (built once at import time from canonical names +
aliases). Unknown names return `None`, which the dispatcher surfaces as
"unknown command". Fuzzy suggestions are produced by
`tools.fuzzy_match.suggest()` so a typo of `/persoanlity` proposes
`/personality`.

## 5. Autocomplete

`SlashCommandCompleter` (`hermes_cli/commands.py`) is a
`prompt_toolkit.completion.Completer`. It:

* Suggests slash commands when the input starts with `/`.
* Suggests subcommands declared on the matching `CommandDef`
  (`subcommands=("list","add","run")`).
* Suggests skill names dynamically from `~/.hermes/skills/` (with
  prefix `/`) and `skills/`.

`SlashCommandAutoSuggest` is a `prompt_toolkit.auto_suggest.AutoSuggest`
that proposes a tail-completion based on history.

Path completion (for `/image <path>`, `/sethome <path>`, etc.) goes
through `hermes_cli/callbacks.py:complete_path()` rather than the
shell-style completer so it respects sandboxed paths.

## 6. Keybindings

The CLI uses `prompt_toolkit`'s keybinding system. Notable bindings:

| Keys | Action |
|------|--------|
| `Enter` | Submit the prompt (configurable: `config.display.busy_enter_action`). |
| `Alt+Enter` / `Esc, Enter` | Always insert a newline. |
| `Ctrl+C` | Interrupt the agent (raise `tools.interrupt.InterruptException`). |
| `Ctrl+D` (empty buffer) | Exit. |
| `Ctrl+L` | Repaint (`/redraw`). |
| `Ctrl+R` | Reverse history search. |
| `Ctrl+P` / `Ctrl+N` | Prev / next history. |
| `Up` / `Down` | Multi-line navigation when buffer is non-empty; otherwise history. |
| `Tab` | Trigger the active completer. |
| `Alt+B`, `Alt+F` | Word-back / word-forward. |

Custom bindings live in `cli.py:HermesCLI._build_key_bindings()`. They
are deliberately conservative — many users come from `bash` or `zsh`
and rely on default Emacs bindings.

## 7. Streaming UI

The CLI renders streaming output through `agent/display.py`'s
`KawaiiSpinner` (animated face during API calls) and a `┊` activity
feed for tool-call results. The flow:

```
Provider transport → AIAgent → callback (hermes_cli/callbacks.py)
                                        │
                                        ▼
                         HermesCLI._render_stream(...)
                                        │
                                        ▼
                          prompt_toolkit print_formatted_text
```

Reasoning blocks are filtered through `cli._strip_reasoning_tags()` so
verbose models do not bury the answer in `<think>...</think>` content.

## 8. Personalities

Defined in `config.personalities` (a dict keyed by name). Each
personality has `system_prompt`, optional `model`, optional `toolset`
overrides, and an emoji. The default personalities ship in
`hermes_cli/default_soul.py` (`DEFAULT_SOUL_MD`); user-defined ones live
in `~/.hermes/config.yaml`. Switching is `/personality <name>`.

## 9. Skin engine

`hermes_cli/skin_engine.py` is the data-driven CLI theming layer. It is
initialised from `config.display.skin` and customises:

* Banner colours (`hermes_cli/banner.py`).
* Spinner faces, verbs, wings (`agent/display.py`).
* Tool prefix glyph (`┊` by default).
* Response box style (single, double, rounded).
* Branding text.

Skins are JSON files under `assets/skins/`; users can drop their own
into `~/.hermes/skins/`. `/skin <name>` switches at runtime.

## 10. Profile system

Profiles isolate `~/.hermes` state. `--profile dev` swaps to
`~/.hermes-dev`. Implementation in `hermes_cli/profiles.py` and
`hermes_cli/main.py:_apply_profile_override()`. The override **must**
run before any other import that reads `get_hermes_home()` — otherwise
the logger, the credential pool and the SessionDB latch onto the wrong
path.

`hermes profiles list` enumerates all `~/.hermes-*` directories.
`hermes profiles use <name>` writes the chosen profile name into
`~/.config/hermes/active-profile` so subsequent `hermes` invocations
default to it without needing `--profile`.

## 11. Config file vs `.env` file

Two-file model. Both live under `~/.hermes/`:

* `config.yaml` — settings, knobs, choices. Deep-merged over
  `DEFAULT_CONFIG` in `hermes_cli/config.py:386`.
* `.env` — secrets only. Loaded by `python-dotenv`. Allowed keys are
  enumerated in `OPTIONAL_ENV_VARS` (`hermes_cli/config.py:1319`,
  ~137 entries) plus `_EXTRA_ENV_KEYS`.

Reading a secret from the config file is intentionally not supported.
Tools that need secrets read `os.environ`; the env loader populates
`os.environ` from `.env` early in startup.

## 12. The `hermes` Bash wrapper

Repo root `hermes`:

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
for c in "$SCRIPT_DIR/.venv" "$SCRIPT_DIR/venv"; do
    if [ -f "$c/bin/python" ]; then
        exec "$c/bin/python" -m hermes_cli.main "$@"
    fi
done
exec hermes "$@"
```

This lets contributors run `./hermes` from a checkout without
activating the venv. The `python -m hermes_cli.main` form is preferred
over `hermes` because it skips the console-script wrapper and goes
straight to the entrypoint, shaving startup cost.

## 13. Testing the CLI

The CLI is tested under `tests/cli/`. Common patterns:

* `tests/test_cli_file_drop.py` — file-drop detection.
* `tests/test_cli_manual_compress.py` — `/compress` flow.
* `tests/test_cli_skin_integration.py` — skin engine.
* `tests/test_model_picker_scroll.py` — model picker keybindings.

Tests use the `prompt_toolkit.input.create_pipe_input()` / `DummyOutput()`
test harnesses to drive the REPL deterministically. Stub clients live
in `tests/fakes/`.

## 14. Adding a CLI subcommand

Two-file change pattern (analogous to slash commands):

1. Create `hermes_cli/<name>.py` with a `run(args)` entry point.
2. Register in `hermes_cli/_parser.py` (the argparse builder) and
   `hermes_cli/main.py` (the dispatcher's `if name == "<name>":` arm).
3. Add tests under `tests/cli/`.
4. Update `hermes help`'s output (`hermes_cli/main.py` `_show_help()`)
   if the command should appear there.

For very small commands (single function, no flags), define them
inline in `hermes_cli/main.py` — there is no architectural penalty
for either choice.

## 15. Diagnostics & debug tooling

* `hermes doctor` — see above.
* `hermes debug` — uploads a redacted debug bundle (logs, config, OS
  info) and returns a shareable URL. See `hermes_cli/debug.py`.
* `HERMES_DEBUG=1` — turns on verbose logging. The exact effect varies
  by module; `hermes_logging.setup_logging()` reads this and bumps the
  level to `DEBUG` for the root logger.
* `HERMES_PROFILER=1` — enables the cProfile harness around the CLI's
  main loop (used by `scripts/profile-tui.py`).
* `HERMES_DEV=1` — relaxes some checks (e.g. permission of
  `~/.hermes/.env`). Do not set this in production.

## 16. Common CLI workflows

### Start a new conversation about a code change

```
$ cd ~/code/myproj
$ hermes
> /personality engineer
> Look at the diff in src/foo.py and propose a fix for the failing
  test in tests/test_foo.py.
```

The CLI auto-detects the git root (`agent/prompt_builder.py:_find_git_root`)
and includes `.hermes.md` / `AGENTS.md` / `SOUL.md` from the project
root in the system prompt.

### Run a one-shot prompt from a script

```
$ hermes -p "Summarise the changes since v0.11.0" --model openrouter:anthropic/claude-haiku-4-5 --quiet
```

`hermes_cli/oneshot.py` skips the REPL, runs a single
`AIAgent.chat()` turn, prints the result, and exits 0/1 on
success/failure.

### Start the gateway with multiple platforms

```
$ hermes gateway setup        # configure once
$ hermes gateway start        # foreground; or use systemd unit, see plugins/kanban/systemd/
```

### Migrate from OpenClaw

```
$ hermes claw migrate --dry-run     # preview
$ hermes claw migrate               # actually migrate
```
