# 19 — Testing and Observability

This chapter covers the test infrastructure, the categories of test
that live in the repo, and how Hermes Agent emits telemetry, logs and
crash reports at runtime. Read this in conjunction with the developer
guide ([07-developer-guide.md](07-developer-guide.md)).

## 1. Test infrastructure

### Layout

```
tests/
├── conftest.py             # shared fixtures + env scrubbing
├── agent/                  # tests for agent/ internals
├── cli/                    # tests for cli.py and hermes_cli/
├── gateway/                # gateway + platform adapter tests
├── hermes_state/           # SessionDB tests
├── hermes_cli/             # subcommand tests
├── tools/                  # tool-by-tool tests
├── skills/                 # skill loading + curator tests
├── plugins/                # plugin loader tests
├── environments/           # RL env tests
├── run_agent/              # AIAgent loop tests
├── tui_gateway/            # TUI gateway JSON-RPC tests
├── acp/                    # ACP adapter tests
├── acp_adapter/            # Lower-level ACP tests
├── e2e/                    # end-to-end tests (mostly integration-marked)
├── integration/            # live-service tests
├── stress/                 # load + concurrency
├── fakes/                  # in-memory fakes for SDK clients
├── honcho_plugin/          # Honcho plugin tests
├── openviking_plugin/      # OpenViking plugin tests
├── website/                # link-check / docs-site tests
├── cron/                   # cron scheduler tests
├── conftest.py             # repo-root conftest
└── test_<topic>.py         # cross-cutting top-level tests
```

### Pytest config

`pyproject.toml` (`[tool.pytest.ini_options]`):

```
testpaths = ["tests"]
markers   = ["integration: marks tests requiring external services"]
addopts   = "-m 'not integration' -n auto"
```

* `pytest-xdist` (`-n auto`) parallelises across cores.
* The `integration` marker is excluded by default; CI sets it to
  `not integration` too. Live-service tests therefore never run
  unless someone deliberately invokes them.

### Canonical entrypoint

```
scripts/run_tests.sh                                   # full suite
scripts/run_tests.sh tests/agent/                      # subset
scripts/run_tests.sh tests/agent/test_foo.py::TestX    # single test
scripts/run_tests.sh --tb=long -v                      # pytest pass-through
```

`scripts/run_tests.sh` enforces:

* `-n 4` xdist workers (matches CI's 4-core budget; `-n auto` diverges
  locally on big machines).
* `TZ=UTC LANG=C.UTF-8 PYTHONHASHSEED=0` for determinism.
* Credential env vars cleared (so a real `OPENAI_API_KEY` cannot leak
  into a test).
* venv activation auto-detected (`.venv` → `venv` →
  `~/.hermes/hermes-agent/venv`).
* `pytest-split` installed lazily (used by CI for shard parity).

### `conftest.py` highlights

`tests/conftest.py`:

* Blanks credential env vars (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `OPENROUTER_API_KEY`, `GOOGLE_API_KEY`, …) at session start.
* Sets `HERMES_HOME` to a per-test temp dir so tests cannot pollute
  the developer's `~/.hermes`.
* Installs a `freeze_time`-compatible monkey-patch on
  `hermes_time.now()`.
* Provides shared `tmp_path`-style fixtures for SessionDB,
  `AIAgent`, fake provider clients, fake gateway adapters.

Per-package `conftest.py` files add domain-specific fixtures
(e.g. `tests/gateway/conftest.py` builds a fake `MessageEvent`).

### Fakes

`tests/fakes/` is the pile of in-memory fakes that let tests run
deterministically without hitting any network:

| Fake | Use |
|------|-----|
| `fake_chat_completions_client.py` | OpenAI-compatible client driven by a scripted response queue. |
| `fake_anthropic_client.py` | Anthropic Messages client. |
| `fake_codex_client.py` | Responses / Codex client. |
| `fake_session_db.py` | In-memory SQLite for SessionDB tests. |
| `fake_platform_adapter.py` | Gateway adapter that records sent messages. |
| `fake_terminal_backend.py` | Stub `BaseExecutionEnvironment`. |
| `fake_mcp_client.py` | Drives `mcp_serve.py` over a pipe. |

Fakes intentionally implement the *minimum* surface a test needs.
Adding a method to a fake requires a corresponding method on the real
class — the static type-check (`ty`) flags drift.

### Markers

| Marker | Meaning |
|--------|---------|
| `integration` | Requires external services (skipped by default). |
| `slow` | Takes >5s; CI may mask in PR runs. |
| `tui` | Drives the Ink TUI; only runs when Node is on `PATH`. |
| `live_provider` | Talks to a real provider (with a real key); never runs in CI. |

CI runs the default pytest invocation (`-m 'not integration'`). The
nightly job runs `-m integration`.

## 2. Test categories

### Unit tests

Most of the suite. They exercise one module at a time with fakes for
its dependencies. Conventions:

* One `test_<module>.py` per source module. Tests for
  `tools/foo_tool.py` go in `tests/tools/test_foo_tool.py`.
* A test starts with `def test_<behaviour>():`.
* Side effects (filesystem, sockets) go through monkey-patched
  helpers; raw `tmp_path` is fine for filesystem.
* Avoid timing-dependent assertions — use the clock fixture.

### Async tests

`pytest-asyncio` is in the `dev` extra. The convention:

```python
import pytest

@pytest.mark.asyncio
async def test_does_thing():
    ...
```

`tests/test_model_tools_async_bridge.py` is the canonical reference
for the `_run_async` plumbing in `model_tools.py`.

### End-to-end tests

`tests/e2e/` and `tests/run_agent/`:

* Spin up an `AIAgent` with fake transports and walk a
  whole conversation through its lifecycle.
* Verify side effects (SessionDB, trajectory files, log lines) —
  not just return values.

### Integration tests

`tests/integration/`:

* Talk to real services with the `integration` marker.
* Each test is responsible for skipping cleanly if its credential is
  missing (`pytest.skip("no NOUS_API_KEY")`).
* CI invocation: `pytest -m integration` (separate workflow).

### Stress tests

`tests/stress/`:

* Concurrency: many simultaneous SessionDB writers, gateway message
  bursts, batch runner with N workers.
* Memory: ensure the LRU cache eviction in `gateway/run.py` actually
  bounds memory.
* Long-running: 1000-turn conversations to exercise compression edge
  cases.

`pytest-xdist` is *not* used for stress tests; they run serially under
`-n 0` to keep numbers comparable across runs.

### TUI tests

`ui-tui/` has its own Vitest-based suite:

```
cd ui-tui
npm test                   # vitest run
npm run type-check         # tsc --noEmit
npm run lint
npm run fmt
```

Python-side TUI tests live in `tests/tui_gateway/`. They drive the
gateway over a paired stdio pipe and assert on emitted events.

### Plugin tests

Each plugin can have its own test directory. The repo runs the test
suites for shipped plugins as part of `scripts/run_tests.sh`:

```
tests/honcho_plugin/        # plugins/memory/honcho/
tests/openviking_plugin/    # plugins/memory/openviking/
tests/plugins/              # generic plugin loader tests
```

### Yuanbao integration tests

`tests/test_yuanbao_*.py` exercise the proprietary Yuanbao protocol
and pipeline, including `test_yuanbao_proto.py` (wire format),
`test_yuanbao_markdown.py` (markdown rendering),
`test_yuanbao_pipeline.py` (full inbound→outbound flow), and
`test_yuanbao_integration.py` (end-to-end with a fake Alipay server).

## 3. Test patterns

### Driving the full agent loop

```python
def test_agent_completes_simple_turn(monkeypatch):
    fake = fake_chat_completions_client.script(
        responses=[{"role": "assistant", "content": "Hello!"}],
    )
    monkeypatch.setattr("run_agent._build_client", lambda *a, **kw: fake)

    agent = AIAgent(
        provider="openrouter",
        model="anthropic/claude-haiku-4-5",
        api_key="sk-fake",
        max_iterations=4,
        skip_memory=True,
        skip_context_files=True,
    )

    result = agent.chat("hi")
    assert result == "Hello!"
```

### Testing a tool

```python
def test_weather_tool_handles_bad_city():
    from tools import weather_tool   # registers on import
    out = json.loads(weather_tool.weather(""))
    assert out == {"success": False, "error": "city is required"}
```

### Testing the SessionDB

```python
def test_session_search_returns_results(tmp_path, monkeypatch):
    monkeypatch.setenv("HERMES_HOME", str(tmp_path))
    db = SessionDB()
    sid = db.create_session(source="cli", model="test:test")
    db.persist_turn(sid, [{"role":"user","content":"hello world"}])
    hits = db.search("hello", limit=10)
    assert any(h["session_id"] == sid for h in hits)
```

### Testing slash commands

```python
def test_yolo_toggle_persists(monkeypatch, tmp_path):
    monkeypatch.setenv("HERMES_HOME", str(tmp_path))
    cli = HermesCLI(quiet=True)
    cli.process_command("/yolo")
    assert load_config()["approvals"]["yolo"] is True
```

### Testing gateway message dispatch

```python
async def test_telegram_message_routed_to_agent(fake_agent_factory):
    runner = GatewayRunner(config=...)
    runner.platforms["telegram"] = FakeTelegramAdapter()
    runner.platforms["telegram"]._inject(text="hello", user_id="u-1")
    await runner._drain()
    fake_agent_factory.assert_called_once()
```

## 4. Coverage philosophy

Hermes does not enforce a coverage percentage. The bar is "every
public function has at least one test". PRs are expected to add a test
when adding a function, and to extend an existing test when changing
behaviour.

`coverage.py` is in the dev extra; running `coverage run -m pytest
&& coverage html` produces a local report.

## 5. Linting / type checking

`ruff` is configured in `pyproject.toml` but applied per-PR by
reviewers — there is no project-wide auto-lint. `ty` (Astral) is
opt-in:

```
ty check                   # after `uv pip install -e ".[dev]"`
```

`ty` is intentionally permissive at the moment (`unresolved-import =
ignore`, `invalid-method-override = ignore`, `not-iterable = ignore`)
because the codebase is mid-migration to typed code. Reviewers reject
new untyped public APIs; existing untyped code is grandfathered.

## 6. CI

The `.github/workflows/` directory carries the CI config. The default
flow:

1. Set up Python 3.11 (and 3.12, 3.13 in matrix runs).
2. `uv pip install -e ".[all,dev]"`.
3. `scripts/run_tests.sh`.
4. `cd ui-tui && npm install && npm run type-check && npm test`.
5. Upload artifacts (test reports, coverage, TUI bundle).

Nightly:

* `pytest -m integration` against a service-account that has live
  credentials.
* SBOM / vuln scan via `osv-scanner`.
* Tirith security scan (in repo).

## 7. Logging

`hermes_logging.setup_logging()` is the canonical entrypoint. It sets
up:

* **agent.log** — INFO+ across all subsystems.
* **errors.log** — WARNING+ duplicated for fast triage.
* **gateway.log** — gateway-only (only when running the gateway).
* **tui-crash-<ts>.log** — TUI panic dumps.

All formatters wrap a `RedactingFormatter`
(`hermes_logging.py`) so secrets never reach disk.

### Loggers by name

```
agent          # agent loop, transports, adapters
agent.context  # context engine
agent.curator  # curator runs
agent.memory   # memory subsystem
gateway        # gateway runner + platforms
gateway.<name> # per-platform
tools          # tool dispatch
tools.<tool>   # per-tool
hermes_state   # SessionDB
hermes_cli     # CLI subcommands
cron           # scheduler
```

The default level is `INFO`; the user can override per-logger via
`logging.<logger>` in `config.yaml`:

```yaml
logging:
  level: "INFO"
  per_logger:
    "tools.terminal_tool": "DEBUG"
    "agent.curator": "WARNING"
```

`HERMES_DEBUG=1` overrides everything to `DEBUG`.

### Session-context tagging

`hermes_logging.set_session_context(session_id, source)` injects the
session id into every log record produced from the calling thread (via
a custom `LogRecord` factory, lines 90-102 of `hermes_logging.py`).
Combined with the JSON formatter (when `logging.format = "json"`),
this makes it trivial to filter logs by session in a log viewer.

### Rotating files

The default rotates at 50 MB and keeps 5 backups. Configurable via
`logging.rotate_max_mb` and `logging.rotate_backups`.

## 8. Metrics

Hermes does not bake in a metrics framework — observability is
provided by plugins. The first-party plugin is
`plugins/observability/langfuse/`.

### Langfuse plugin

* Sample rate: `HERMES_LANGFUSE_SAMPLE_RATE` (0.0-1.0).
* Max payload chars: `HERMES_LANGFUSE_MAX_CHARS` (truncates large
  prompts/responses).
* Env / release tagging: `HERMES_LANGFUSE_ENV`,
  `HERMES_LANGFUSE_RELEASE`.
* Debug: `HERMES_LANGFUSE_DEBUG=1`.
* Standard SDK vars: `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`,
  `LANGFUSE_BASE_URL`.

The plugin hooks into:

* Every provider call (records latency, prompt, response, tokens).
* Every tool call (records arguments, result, runtime).
* Session start/end (records session-level stats).

Activate with `hermes plugins enable observability/langfuse` after
the credentials are set in `.env`.

### Building your own observability plugin

Implement `gateway/hooks.py` callbacks (`on_session_end`,
`on_turn_complete`, `on_tool_call`, `on_provider_call`) and emit your
own metrics. The plugin manifest declares which hooks to register;
the loader subscribes them when the plugin is enabled.

## 9. Tracing

The codebase has a lightweight tracer in `agent/display.py` that
emits per-turn spans (start, end, type) for the streaming UI. It is
not a tracing API in the OpenTelemetry sense — only enough to drive
the activity feed.

For real tracing, the Langfuse plugin emits Langfuse traces; the
OpenTelemetry plugin (proposed; not in v0.12.0) would emit OTLP
spans against an OTLP collector.

## 10. Crash handling

* Top-level uncaught exceptions in the CLI go through
  `cli.HermesCLI._handle_unhandled_exception()`, which writes a
  redacted bundle to `~/.hermes/logs/crash-<ts>.log` and prompts the
  user to upload via `hermes debug`.
* The TUI's Python gateway installs panic hooks
  (`tui_gateway/server.py`); a panic produces
  `~/.hermes/logs/tui-crash-<ts>.log` plus a one-line "gateway died"
  banner in the Ink UI.
* The ACP adapter's logger filter
  (`acp_adapter/entry._BenignProbeMethodFilter`) silences benign
  `method_not_found` tracebacks for liveness probes only — every other
  exception still reaches stderr.
* The gateway's per-platform error handler logs the exception to
  `gateway.log` and notifies the user with a generic "something went
  wrong" message in the relevant chat. The full traceback never
  leaves the host.

## 11. `hermes debug`

`hermes_cli/debug.py` is the canonical "support bundle" command. It:

1. Collects:
   * `hermes doctor` output.
   * Last 1000 lines each of `agent.log`, `errors.log`, `gateway.log`.
   * Resolved config (with secrets redacted).
   * Versions: hermes, Python, OS, Node (if available), key Python
     packages.
2. Runs each captured blob through `agent.redact.redact()`.
3. Uploads to a configured paste endpoint (default: Hermes Cloud).
4. Returns shareable URLs to paste in an issue.

Refuses to upload anything that contains content matching any of the
redaction patterns it didn't manage to scrub — the user gets a copy on
disk and is told to redact manually.

## 12. Verifying a release

The `scripts/release.py` flow includes a smoke step that:

* Builds the wheel.
* Installs it into a fresh venv.
* Runs `hermes --version`, `hermes doctor`, `hermes -p "say hi"`
  (against a fake provider via `HERMES_FAKE_PROVIDER=1`).
* Validates that the bundled JS is present in the wheel.
* Validates that the model catalog is up to date.

Failed smoke aborts the release.

## 13. Performance baselines

Tracked baselines (used to catch regressions in PR review):

| Metric | Target |
|--------|--------|
| `hermes` cold start | <250 ms (without TUI). |
| `hermes --tui` cold start | <500 ms (Node + Python). |
| `hermes -p "..."` first token | <2 s on Anthropic Haiku. |
| Slash command dispatch | <50 ms typical. |
| Tool registry import | <100 ms with all built-ins. |
| Test suite (default) | <5 minutes on 4 cores. |
| Wheel size | <30 MB without TUI bundle, <50 MB with. |

`scripts/profile-tui.py` is the harness for instrumenting the cold-
start path.

## 14. Troubleshooting failing tests

Most common failure modes (in order of frequency):

1. **Env contamination** — a real key is set, leaking into the test
   matrix. `scripts/run_tests.sh` blanks them; running `pytest`
   directly does not.
2. **Locale** — `LANG=C` breaks Unicode-handling tests. Set
   `LANG=C.UTF-8`.
3. **TZ** — clock-dependent tests fail outside UTC. Set `TZ=UTC`.
4. **Process leakage** — a test that spawns a subprocess (e.g. a TUI
   test) failed without cleaning up. Look for orphan `python` /
   `node` PIDs from the same workspace.
5. **WAL-related flakiness** — under heavy `-n auto`, SessionDB
   writers may collide. The retry loop usually saves it; if it does
   not, lower `-n`.
6. **`pytest-xdist` shard imbalance** — without `pytest-split`, some
   shards finish minutes before others. `scripts/run_tests.sh`
   installs `pytest-split` automatically.

## 15. Where each subsystem's tests live

Quick map:

| Subsystem | Tests |
|-----------|-------|
| `run_agent.py` core loop | `tests/run_agent/`, `tests/test_ctx_halving_fix.py`, `tests/test_get_tool_definitions_cache_isolation.py` |
| Tools (registry + each tool) | `tests/tools/` |
| Toolsets / distributions | `tests/test_toolsets.py`, `tests/test_toolset_distributions.py` |
| Provider transports / adapters | `tests/agent/` |
| Memory subsystem | `tests/agent/`, `tests/honcho_plugin/`, `tests/openviking_plugin/` |
| Curator | `tests/skills/` |
| Context engine / compressor | `tests/agent/`, `tests/test_trajectory_compressor.py` |
| SessionDB | `tests/hermes_state/`, `tests/test_evidence_store.py`, `tests/test_sql_injection.py`, `tests/test_atomic_replace_symlinks.py` |
| Logging | `tests/test_hermes_logging.py` |
| Time | `tests/test_timezone.py` |
| CLI | `tests/cli/`, `tests/test_cli_*.py`, `tests/test_model_picker_scroll.py` |
| Slash commands | `tests/cli/` (per-command files) |
| Gateway | `tests/gateway/` |
| Per-platform | `tests/gateway/platforms/` |
| Cron | `tests/cron/`, `tests/test_batch_runner_checkpoint.py` |
| Batch runner | `tests/test_batch_runner_checkpoint.py` |
| Trajectory compressor | `tests/test_trajectory_compressor.py`, `tests/test_trajectory_compressor_async.py` |
| Mini SWE runner | `tests/test_mini_swe_runner.py`, `tests/test_minisweagent_path.py` |
| MCP | `tests/test_mcp_serve.py`, `tests/integration/mcp/` |
| ACP | `tests/acp/`, `tests/acp_adapter/` |
| TUI gateway | `tests/tui_gateway/`, `tests/test_tui_gateway_server.py` |
| Skills loading + sync | `tests/skills/`, `tests/test_plugin_skills.py` |
| Plugins | `tests/plugins/` |
| Packaging | `tests/test_packaging_metadata.py`, `tests/test_project_metadata.py` |
| Install scripts | `tests/test_install_sh_setup_wizard_tty_probe.py` |
| Yuanbao | `tests/test_yuanbao_*.py` |
| Network preferences | `tests/test_ipv4_preference.py`, `tests/test_base_url_hostname.py` |
| Atomic writes | `tests/test_atomic_replace_symlinks.py` |
| Subprocess isolation | `tests/test_subprocess_home_isolation.py` |
| Hermes constants / home | `tests/test_hermes_home_profile_warning.py`, `tests/test_hermes_constants.py` |
| Misc retries | `tests/test_retry_utils.py` |

## 16. Adding a test

Minimum-viable PR shape for a new feature:

1. The feature itself.
2. **One unit test** for the happy path.
3. **One unit test** for at least one error path.
4. (If user-visible) **One e2e test** that drives the full surface
   (e.g. a slash command end-to-end, a tool dispatched through the
   agent loop).
5. (If destructive) **One stress / fuzz test** if there is even a
   small chance the feature could corrupt state.

Reviewers will ask for these explicitly if missing.

## 17. Observability checklist for new features

* [ ] Log statements at INFO level for every meaningful state
      transition (not every line).
* [ ] Log statements at WARNING / ERROR with redaction-friendly text.
* [ ] No `print()` calls.
* [ ] Logger name follows the package conventions
      (`tools.<tool>`, `agent.<subsystem>`, `gateway.<platform>`).
* [ ] If the feature has counters, expose them via the gateway's
      `/api/status` endpoint or a hook callback.
* [ ] If the feature has a long-running background loop, expose its
      health to `hermes status`.
* [ ] If the feature can fail in a way the user must see, surface a
      notification — do not silently log.

## 18. Operational runbook (for deployers)

* **Logs** — tail `~/.hermes/logs/agent.log`. Errors duplicate to
  `errors.log`.
* **Status** — `hermes status` for a CLI snapshot.
* **Health** — gateway exposes `/api/health` (the FastAPI app under
  the dashboard). Returns 200 + JSON if the runner is alive.
* **Restart** — `hermes gateway restart` drains active sessions
  before stopping.
* **Backups** — `~/.hermes/state.db` is the only critical file.
  Snapshot it from outside Hermes (the WAL handles in-flight writes).
* **Upgrades** — `hermes update` for unmanaged installs;
  `brew upgrade hermes-agent` / `nixos-rebuild switch` for managed
  ones.
* **Cron** — gateway must be running for cron to fire. The lock file
  at `~/.hermes/cron/.tick.lock` ensures only one process executes
  ticks even if multiple gateways start.
* **Rate limits** — `~/.hermes/rate_limits/` carries per-provider
  cool-down state; safe to delete if a provider's rate limit changed
  on their side.
* **Secrets rotation** — edit `.env`, then `/reload` in a session, or
  restart the gateway.
