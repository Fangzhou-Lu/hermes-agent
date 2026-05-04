# 01 — Architecture

This document describes the high-level architecture of Hermes Agent: how the
pieces fit together and how a single user message is processed end-to-end.

## 1. Process topology

A running Hermes deployment can take three shapes:

| Shape            | Entry point                | Typical use                                  |
|------------------|----------------------------|----------------------------------------------|
| **Interactive CLI** | `hermes` → `hermes_cli.main:main` → `cli.HermesCLI` | A human at a terminal |
| **Gateway daemon**  | `hermes gateway start` → `gateway.run` → spawns `AIAgent` per session | Telegram / Discord / Slack / … bot |
| **Headless / batch**| `hermes-agent` or `python run_agent.py` / `batch_runner.py` / `mini_swe_runner.py` | RL data, automation, MCP server, ACP server |

All three shapes share the **same core**: an `AIAgent` instance from
`run_agent.py`. The CLI and gateway are surfaces that drive the agent
loop and route input/output; they do **not** re-implement it.

```
                ┌────────────────────────┐
   user ───►    │    Surface layer       │     ─── CLI │ Gateway │ Batch │ MCP │ ACP
                ├────────────────────────┤
                │      AIAgent           │     run_agent.py — the conversation loop
                ├────────────────────────┤
   ┌────────────┤  Provider transports   ├────────────┐
   ▼            └────────────────────────┘            ▼
 Anthropic      OpenAI/chat_completions   Bedrock   Codex   Gemini   …
   │
   ▼
   Tool calls ──►  model_tools.handle_function_call()
                    │
                    ▼
                  tools/registry  ── 40+ tools, each registered at import time
```

## 2. The agent loop

The conversation loop is implemented in `AIAgent.run_conversation()` inside
`run_agent.py`. The shape (paraphrased from `AGENTS.md`) is:

```python
while api_call_count < self.max_iterations and self.iteration_budget.remaining > 0:
    interrupt.check()                       # raise if user cancelled
    response = transport.complete(messages, tools)
    api_call_count += 1
    self.iteration_budget.consume(response.usage)

    if response.tool_calls:
        for call in response.tool_calls:
            result = model_tools.handle_function_call(
                call.name, call.arguments, task_id=..., user_task=...
            )
            messages.append(tool_result_message(call, result))
        continue

    if response.text:
        messages.append(assistant_message(response.text))
        break
```

Around this naive shape, `run_conversation` adds:

* **Context compression** — before each turn, `agent.context_engine` is
  consulted (`should_compress_preflight`); after each response,
  `update_from_response` records token usage and may trigger
  `context_compressor.ContextCompressor.compress()` to summarise the middle
  of the message history.
* **Memory pre/post hooks** — `MemoryManager.queue_prefetch_all()` runs at
  the start of a turn; `sync_all()` runs after.
* **Skill preprocessing** — `agent/skill_preprocessing.expand_inline_shell()`
  and `agent/skill_commands._build_skill_message()` rewrite slash-command
  messages (`/skill-name args`) into structured user messages.
* **Trajectory capture** — when `save_trajectories=True`, every turn is
  appended to `trajectory_samples.jsonl` (or `failed_trajectories.jsonl`
  on error) by `agent.trajectory.save_trajectory()`.
* **Credential failover** — on rate-limit / auth errors, the
  `CredentialPool` (`agent/credential_pool.py`) rotates to the next
  `PooledCredential` according to its strategy
  (`FILL_FIRST`, `ROUND_ROBIN`, `RANDOM`, `LEAST_USED`).
* **Interrupts** — `tools.interrupt.check()` is called at safe boundaries.
  In the CLI, `Ctrl+C` and out-of-band user input both raise to break out
  of the loop and let the user redirect.
* **Grace turn** — when `max_iterations` is reached, the agent makes a
  final "wrap up" call so the user gets a coherent answer rather than a
  truncation.

`AIAgent.__init__` accepts ~60 parameters; the practical subset is documented
in `AGENTS.md`. Important ones:

| Parameter             | Meaning |
|-----------------------|---------|
| `base_url`, `api_key` | Provider endpoint and credential |
| `provider`, `api_mode`| Selects the transport (`anthropic`, `chat_completions`, `codex_responses`, …) |
| `model`               | Model id; resolved against catalog if empty |
| `max_iterations`      | Tool-calling budget (default 90) shared with subagents |
| `enabled_toolsets` / `disabled_toolsets` | Filter the registered tool set |
| `quiet_mode`          | Suppresses streaming UI for headless callers |
| `save_trajectories`   | Emit JSONL for training data |
| `platform`            | `"cli"`, `"telegram"`, … — affects prompt hints and delivery |
| `session_id`          | If present, resume the conversation from `SessionDB` |
| `skip_context_files`  | Skip `.hermes.md`, `AGENTS.md`, `SOUL.md` injection |
| `skip_memory`         | Disable the memory subsystem for this run |
| `credential_pool`     | A pre-built `CredentialPool` for failover |

## 3. Request lifecycle (CLI example)

A `hermes` user typing a message goes through the following phases:

1. **Input capture** — `cli.HermesCLI` reads a multi-line input via
   prompt_toolkit. Slash commands (`/model`, `/personality`, `/usage`,
   `/compress`, `/skills`, `/<skill-name>`, …) are intercepted in the
   surface layer. Anything else is treated as a user turn.
2. **Skill expansion** — if the message starts with `/<skill-name>`,
   `agent/skill_commands._load_skill_payload()` and `_inject_skill_config()`
   rewrite it into the canonical user message.
3. **Pre-turn hooks** — `MemoryManager.queue_prefetch_all()` runs in the
   background. The `ContextEngine.should_compress_preflight()` check may
   trigger an early compression pass.
4. **Provider call** — `AIAgent` builds an API request via the active
   `ProviderTransport.build_kwargs()`, then streams the response.
   Streaming chunks are normalised (`NormalizedResponse` from
   `agent/transports/types.py`) and fed to the surface layer.
5. **Tool calls** — the normalised response may contain `tool_calls`. Each
   call is dispatched through `model_tools.handle_function_call()`,
   which looks up the tool in `tools.registry` and runs it. Tool output
   is appended to the message history as a `tool` role message.
6. **Iteration** — steps 4–5 repeat until the model emits a terminal
   text message or the iteration budget is exhausted.
7. **Post-turn hooks** — `MemoryManager.sync_all()` reflects on the turn
   and may write to memory; `Curator` may schedule itself if the agent
   has been idle long enough; `SessionDB` persists the new messages,
   token counts and tool-call stats.

The same lifecycle applies to gateway sessions, except step 1 is replaced
by an inbound platform message (Telegram update, Discord event, …) and
step 7 also dispatches the assistant reply back through
`gateway.delivery` to the originating platform.

## 4. File dependency chain

The bottom of the stack is the tool registry; everything above it depends
on it transitively (`AGENTS.md`):

```
tools/registry.py        (no deps — imported by every tool file)
       ↑
tools/*.py               (each calls registry.register() at import time)
       ↑
model_tools.py           (imports tools.registry + triggers discovery)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/, mcp_serve.py
```

`model_tools.discover_builtin_tools()` is what causes the
side-effect-driven imports in `tools/` to run, populating the registry
before the first `get_tool_definitions()` call.

## 5. Async strategy

The codebase is mostly synchronous on purpose: the agent loop is
synchronous, tool calls are synchronous, and the CLI surface is
synchronous (prompt_toolkit handles its own event loop). Async only
appears where a third-party SDK requires it:

* HTTP clients (`httpx.AsyncClient`, `AsyncOpenAI`, `AsyncAnthropic`) used
  inside specific tools.
* Gateway platform adapters (Telegram, Discord, Matrix, Slack Bolt) are
  async-native.
* The MCP server (`mcp_serve.py`) and ACP adapter (`acp_adapter/`).

`model_tools._run_async()` (in `model_tools.py`, lines 40-170) bridges
sync tool handlers to async clients **without** spinning up a fresh
event loop per call — that would cause "Event loop is closed" errors on
cached clients. It maintains a per-thread persistent loop:

* `_get_tool_loop()` — a single loop on the main thread.
* `_get_worker_loop()` — a loop per worker thread, kept in
  thread-local storage (used by `batch_runner.py` workers).

If a running loop is detected at call time, dispatch raises rather than
nesting loops.

## 6. Configuration & profiles

User configuration is layered:

1. **Defaults** baked into the code (`agent/*`, `tools/*` constants).
2. **`~/.hermes/config.yaml`** — top-level keys for `model`, `provider`,
   `personality`, `tools`, `gateway`, `cron`, `curator`, `context`,
   `memory`, `skills`, `pricing`, etc.
3. **`~/.hermes/.env`** — API keys only (loaded with `python-dotenv`).
4. **Profile overrides** — `--profile <name>` swaps the home directory
   to `~/.hermes-<name>/`. `hermes_cli.main._apply_profile_override()`
   pre-parses this flag *before* any imports that depend on
   `get_hermes_home()`, so that logging, state and credentials all see
   the profile-scoped paths.
5. **Environment variables** — most config keys can be overridden via
   `HERMES_*` env vars. `HERMES_HOME` overrides the root entirely.
6. **Per-call kwargs** — `AIAgent(**kwargs)` always wins.

The `display_hermes_home()` helper in `hermes_constants.py` returns a
user-friendly path (`~/.hermes` form) for log messages and prompts.

## 7. State & persistence

All conversational state lives in **`~/.hermes/state.db`**, a SQLite
database with WAL mode. The schema is defined in `hermes_state.py` and
covers:

* `sessions` — one row per session: id, source, model, parent
  session id (used to chain compressed sessions), timestamps,
  message/tool counts, token totals, billing snapshot.
* `messages` — full message history; one row per message with role,
  content, tool-call payloads, finish reason, reasoning text, token count.
* `state_meta` — small key/value store for persistent flags
  (last-curator-run, last-update check, etc.).
* `messages_fts` — FTS5 virtual table over message content (Unicode61).
* `messages_fts_trigram` — trigram FTS5 table for CJK substring search.

The `SessionDB` class is thread-safe via short SQLite timeouts plus
application-level jitter retry (15 retries, 20-150 ms each). All other
caches (`agent-created skills`, `image cache`, `cron state`, …) live in
sibling directories under `~/.hermes/`.

## 8. Extension points

The architecture exposes a small number of well-defined extension points
so that contributors do not need to touch the core loop:

| Extension     | Mechanism | See |
|---------------|-----------|-----|
| **Tool**      | Add a file under `tools/` and call `registry.register()` | [03-tools.md](03-tools.md) |
| **Provider**  | Subclass `ProviderTransport` in `agent/transports/` and add a model_tools mapping | [06-providers.md](06-providers.md) |
| **Platform**  | Subclass `BaseGatewayPlatform` in `gateway/platforms/` | [04-gateway.md](04-gateway.md) |
| **Memory provider** | Subclass `MemoryProvider` in a `plugins/memory/<name>/` package | [05-state-and-memory.md](05-state-and-memory.md) |
| **Context engine** | Subclass `ContextEngine` in `plugins/context_engine/<name>/` | [05-state-and-memory.md](05-state-and-memory.md) |
| **Skill**     | Drop a directory under `skills/` or `optional-skills/` with a `SKILL.md` | [03-tools.md](03-tools.md) |
| **Plugin**    | Drop a Python package under `plugins/<name>/` with the plugin manifest | [02-modules.md](02-modules.md) |
| **MCP server**| Configure in `~/.hermes/mcp_servers.yaml`; `tools/mcp_tool.py` exposes the calls | [03-tools.md](03-tools.md) |
| **Cron job**  | `cron/jobs.py` definitions; delivered via gateway `delivery.py` | [04-gateway.md](04-gateway.md) |

## 9. Lifecycles

### Process lifecycle

```
hermes_cli.main:main()
  ├─► _apply_profile_override()  ── before any state-touching imports
  ├─► load .env                    via hermes_cli.env_loader
  ├─► load config.yaml             via hermes_cli.config
  ├─► setup_logging()
  ├─► dispatch subcommand
  │     ├── interactive REPL  →  HermesCLI(...)
  │     ├── one-shot          →  hermes_cli.oneshot.run(...)
  │     ├── gateway           →  gateway.run.GatewayRunner(...)
  │     ├── batch             →  batch_runner.main(...)
  │     ├── ACP / MCP         →  acp_adapter / mcp_serve
  │     └── ...               →  one of ~30 hermes_cli/<sub>.py files
  └─► clean shutdown          via atexit hooks (close DB, flush logs,
                                kill background processes)
```

### Session lifecycle

```
session_id = uuid4()
  ├── created_at   = now()
  ├── source       = "cli" | "telegram" | "discord" | ...
  ├── model        = "anthropic:claude-opus-4-7"
  ├── parent_id    = None | <prev session id when branched/compressed>
  ▼
N turns of:
  ├── pre-turn:  memory.queue_prefetch_all()
  │              context_engine.should_compress_preflight()
  ├── transport call(s) (with credential rotation on errors)
  ├── tool call dispatch
  ├── post-turn: memory.sync_all()
  │              SessionDB.persist_turn()
  ▼
session ends when:
  ├── /quit, gateway shutdown, or LRU eviction (1h idle)
  ├── /reset — old session is closed; new session created
  ├── /branch — new session created with parent_session_id
  └── compression — new session created with parent_session_id
```

### Agent lifecycle within a turn

```
AIAgent.run_conversation(user_message)
  ├── interrupt.check()
  ├── messages.append({"role": "user", "content": user_message})
  ├── for each iteration ≤ max_iterations:
  │     ├── interrupt.check()
  │     ├── transport.build_kwargs(messages, tools)
  │     ├── stream_response = provider.complete(**kwargs)
  │     ├── for chunk in stream_response:
  │     │     ├── text_delta  → callback.on_text(chunk)
  │     │     ├── tool_delta  → callback.on_tool_delta(chunk)
  │     │     └── reasoning   → callback.on_reasoning(chunk)
  │     ├── nr = transport.normalize_response(stream)
  │     ├── if nr.tool_calls:
  │     │     for tc in nr.tool_calls:
  │     │         result = handle_function_call(tc)
  │     │         messages.append(tool_result(tc, result))
  │     │     continue
  │     └── if nr.text:
  │           messages.append(assistant(nr.text))
  │           break
  ├── memory.sync_all(messages)
  ├── SessionDB.persist_turn()
  └── return final_response
```

## 10. Cross-cutting concerns

### Interrupts

`tools/interrupt.py` is the central interrupt registry. The agent
loop checks `interrupt.is_set()` at safe boundaries:

* Before each provider call.
* Before each tool dispatch.
* Between streaming chunks.

In the CLI, `Ctrl+C` raises a `KeyboardInterrupt` that the prompt-toolkit
event loop translates into an interrupt. The TUI sends the interrupt
via `tool.cancel` JSON-RPC. The gateway handles `/stop` by setting the
flag.

### Determinism

The codebase is intentionally deterministic where it can be:

* `hermes_time.now()` is stubbed in tests.
* `os.environ` is normalised by `hermes_cli.env_loader.load_hermes_dotenv()`.
* SQLite WAL gives serialisability per row.
* The credential pool's randomised strategy uses a seeded PRNG when
  testing.

This makes test failures reproducible.

### Idempotency

Most operations are idempotent or close to it:

* `SessionDB.persist_turn` — writes new rows; never modifies prior
  rows.
* `MemoryManager.sync_all` — providers decide whether to write.
* `Curator.run` — backs up before any destructive change; safe to
  retry.
* `cron.scheduler.tick` — file lock prevents double-execution; jobs
  may be retried by their natural schedule.

This is crucial for crash recovery: a half-applied operation cannot
corrupt state.

### Cancellation safety

Long-running tools must accept and respect cancellation. The pattern:

```python
def long_running_tool(args, **kw):
    interrupt = kw.get("interrupt")
    for chunk in produce_chunks():
        if interrupt and interrupt.is_set():
            raise InterruptException()
        process(chunk)
```

Without this, `Ctrl+C` cannot abort a runaway tool. CI tests cover the
common offenders (`terminal_tool`, `web_extract`, `browser`).

