# 25 — Fakes and Test Harness

This chapter is the working manual for `tests/fakes/` and the
shared fixtures in `tests/conftest.py`. Everything here is
test-only — none of it ships in the wheel.

The point: write deterministic tests *fast*, without spinning up
real providers, real platforms, or real subprocesses. The fake
catalogue is what makes that possible.

## 1. Layout

```
tests/
├── conftest.py             # repo-root shared fixtures + env scrubbing
├── fakes/
│   ├── __init__.py
│   ├── fake_chat_completions_client.py
│   ├── fake_anthropic_client.py
│   ├── fake_codex_client.py
│   ├── fake_session_db.py
│   ├── fake_platform_adapter.py
│   ├── fake_terminal_backend.py
│   ├── fake_mcp_client.py
│   ├── fake_browser_provider.py
│   ├── fake_credential_pool.py
│   ├── fake_memory_provider.py
│   └── fake_context_engine.py
├── conftest.py             # per-package conftests under each subdir
└── ...
```

Each fake is intentionally small — just enough to exercise the surface
under test. When a test needs a behaviour the fake does not provide,
extend the fake with the minimum addition.

## 2. The repo-root `conftest.py`

Provides session-scoped fixtures every test inherits.

### Env scrubbing

```python
@pytest.fixture(autouse=True, scope="session")
def _scrub_credentials():
    for key in (
        "OPENAI_API_KEY", "ANTHROPIC_API_KEY", "OPENROUTER_API_KEY",
        "GOOGLE_API_KEY", "GEMINI_API_KEY", "XAI_API_KEY",
        "MISTRAL_API_KEY", "MINIMAX_API_KEY", "NVIDIA_API_KEY",
        "TELEGRAM_BOT_TOKEN", "DISCORD_BOT_TOKEN", "SLACK_BOT_TOKEN",
        # ... every key likely to leak into tests
    ):
        os.environ.pop(key, None)
    yield
```

Why: a developer might have real keys in their env. Without this
fixture, tests that build a credential pool would silently use the
real key and rack up real charges (and possibly hit the real
provider). Scrubbing makes it impossible.

### `HERMES_HOME` redirect

```python
@pytest.fixture(autouse=True)
def _hermes_home_tmp(tmp_path, monkeypatch):
    monkeypatch.setenv("HERMES_HOME", str(tmp_path / "hermes-home"))
    yield
```

Every test gets its own `~/.hermes` under `tmp_path`. Tests cannot
pollute each other's state, the developer's state, or anything else.

### Frozen clock

```python
@pytest.fixture
def frozen_now(monkeypatch):
    fixed = datetime(2026, 4, 1, 12, 0, 0, tzinfo=timezone.utc)
    monkeypatch.setattr("hermes_time.now", lambda: fixed)
    yield fixed
```

Use this when assertions depend on `now()`. The clock is
deterministic; freeze it explicitly when the test cares.

### Deterministic random

`PYTHONHASHSEED=0` is set in `scripts/run_tests.sh`. Tests that need
deterministic randomness should also `random.seed(0)` explicitly —
the global seed only stabilises hashing.

### `caplog`

Pytest's standard `caplog` fixture works out of the box. Set the
level explicitly:

```python
def test_something_logs_warning(caplog):
    caplog.set_level("WARNING", logger="agent")
    do_thing()
    assert any("expected message" in r.message for r in caplog.records)
```

## 3. `fake_chat_completions_client`

### Public API

```python
from tests.fakes.fake_chat_completions_client import script

fake = script(responses=[
    {"role": "assistant", "content": "Hi!"},
    {"role": "assistant", "tool_calls": [{
        "id": "tc-1", "type": "function",
        "function": {"name": "weather", "arguments": "{\"city\":\"Tokyo\"}"},
    }]},
    {"role": "tool", "tool_call_id": "tc-1", "content": "{\"temp\":\"22C\"}"},
    {"role": "assistant", "content": "It's 22°C in Tokyo."},
])
```

The fake exposes a `chat.completions.create()` that:

* Returns the next response in the queue.
* Records the request kwargs in `fake.calls` for assertions.
* Streams when `stream=True` (yields `ChatCompletionChunk` shapes).

### Use

```python
def test_full_loop(monkeypatch):
    fake = script(...)
    monkeypatch.setattr("run_agent._build_client", lambda *a, **kw: fake)
    agent = AIAgent(provider="openrouter", model="x", api_key="sk-fake",
                    skip_memory=True, skip_context_files=True)
    out = agent.chat("hi")
    assert "Tokyo" in out
    assert fake.calls[0]["model"] == "x"
```

### Common pitfalls

* **Forgetting to script enough turns** — the agent loop iterates
  until the model returns a terminal text response. If the script
  ends mid-tool-call, the loop hangs (until the iteration budget).
* **Mismatching `tool_call_id`** — Anthropic / OpenAI both error if
  the id does not roundtrip. Use the same id you returned in the
  previous assistant message.

## 4. `fake_anthropic_client`

Same shape as `fake_chat_completions_client` but for the Anthropic
SDK. Differences:

* Returns `Message` objects with `content` blocks (text + tool_use).
* Streams `MessageStream`-shaped iterators.

Use when testing `AnthropicMessagesTransport` end-to-end.

## 5. `fake_codex_client`

Drives `agent/transports/codex.py`. Fakes the Responses API shape
(items, function_calls, reasoning blocks).

Used by `tests/agent/test_codex_responses_adapter.py` for
`_normalize_codex_response` round-trips.

## 6. `fake_session_db`

In-memory `SessionDB` replacement. Compatible with the public API:

```python
db = FakeSessionDB()
sid = db.create_session(source="cli", model="test:test")
db.persist_turn(sid, [{"role": "user", "content": "hi"}])
hits = db.search("hi")
```

Backed by Python dicts; FTS is implemented as substring scan (good
enough for tests, not a real FTS5 replacement).

When to use the *real* `SessionDB`: tests that exercise SQL, FTS5
ranking, WAL contention, or migration paths. Otherwise the fake is
faster and easier.

## 7. `fake_platform_adapter`

A `BasePlatformAdapter` that records every outbound message into
`fake.sent_messages` and lets tests inject inbound `MessageEvent`s
into `fake.inject(...)`.

```python
def test_telegram_loops_message(fake_runner):
    tg = FakePlatformAdapter(name="telegram")
    fake_runner.platforms["telegram"] = tg
    tg.inject(text="hi", user_id="u-1")
    fake_runner.run_one_step()
    assert tg.sent_messages == [{"chat_id": "u-1", "text": "Hi back!"}]
```

## 8. `fake_terminal_backend`

Replaces `BaseExecutionEnvironment`. Tests can:

```python
fake = FakeTerminalBackend()
fake.add_response("ls", stdout="foo.txt\nbar.txt", returncode=0)

stdout, stderr, code = fake.execute(["ls"], cwd=None)
assert "foo.txt" in stdout
assert fake.commands == [(["ls"], None)]
```

Two modes:

* **Scripted** (`add_response`) — match command prefixes; later
  responses match longer prefixes first.
* **Echo** — for trivial tests, returns the command itself as stdout.

## 9. `fake_mcp_client`

Drives `mcp_serve.py` over a pipe so the MCP server side can be
tested without a real MCP host:

```python
def test_mcp_session_search_returns_hit():
    server, client = spawn_pair_with_db(...)
    out = client.call("tools/call", {
        "name": "hermes_session_search",
        "arguments": {"q": "hello"},
    })
    assert out["result"]["hits"]
```

Used heavily in `tests/test_mcp_serve.py`.

## 10. `fake_browser_provider`

Implements `BrowserProvider` returning canned page state:

```python
fake = FakeBrowserProvider()
fake.add_page(url="https://example.com",
              title="Example",
              text="Hello!",
              links=["https://example.com/about"])
result = fake.open_url("https://example.com")
assert result.title == "Example"
```

For full-stack browser tests (driving real Chromium) use the
integration marker; the unit tests use this fake.

## 11. `fake_credential_pool`

Useful when the test's focus is the credential-rotation code path:

```python
pool = FakeCredentialPool([
    PooledCredential(provider="openai", id="k1", access_token="sk-1",
                     last_status=STATUS_OK),
    PooledCredential(provider="openai", id="k2", access_token="sk-2",
                     last_status=STATUS_EXHAUSTED),
])
assert pool.next("openai").id == "k1"
pool.mark_exhausted(pool.current(), reset_at=time.time() + 60)
assert pool.next("openai").id == "k2"
```

## 12. `fake_memory_provider`

Implements `MemoryProvider` with predictable side effects so tests
can verify hook ordering:

```python
fake = FakeMemoryProvider()
manager = MemoryManager()
manager.add_provider(fake)
manager.queue_prefetch_all()
manager.sync_all([{"role": "user", "content": "x"}])
assert fake.events == ["prefetch", "sync_turn"]
```

## 13. `fake_context_engine`

Implements `ContextEngine` that can be told to compress on demand:

```python
fake = FakeContextEngine()
fake.set_should_compress(True)
fake.set_compressed_messages([{"role": "system", "content": "summary"}])
```

Used to verify the agent loop's compression code path without running
the real compressor (which calls an LLM).

## 14. Per-package conftests

Each subdirectory often has its own conftest with domain-specific
fixtures.

### `tests/agent/conftest.py`

* `make_agent(transport, **kw)` — convenience builder.
* `script_normalized(...)` — emit a sequence of
  `NormalizedResponse`s without going through a transport.

### `tests/gateway/conftest.py`

* `fake_runner` — a `GatewayRunner` with all platforms replaced by
  `FakePlatformAdapter`.
* `make_message_event(...)` — convenience event builder.

### `tests/cli/conftest.py`

* `cli` — a `HermesCLI` instance backed by `prompt_toolkit.input.create_pipe_input`
  and `DummyOutput()`.
* `slash(input)` — driver function that types a slash command and
  returns the captured output.

### `tests/skills/conftest.py`

* `skill_dir(name, body)` — build an in-tmp skill directory.
* `skills_index([dir1, dir2, ...])` — produce an in-memory index.

### `tests/run_agent/conftest.py`

* `simple_agent` — a fully-mocked `AIAgent` ready to run a scripted
  conversation.
* `mock_provider(...)` — wire the right transport given the active
  provider.

### `tests/tools/conftest.py`

* `tool(name)` — fetch a `ToolEntry` by name from the registry.
* `dispatch(name, args)` — run a tool with the right call shape.

## 15. Patterns

### Drive the full agent loop

```python
def test_agent_makes_tool_call_then_replies(monkeypatch):
    fake = script([
        {"role": "assistant", "tool_calls": [{
            "id": "tc-1", "type": "function",
            "function": {"name": "weather", "arguments": "{}"},
        }]},
        {"role": "assistant", "content": "It's nice out."},
    ])
    monkeypatch.setattr("run_agent._build_client", lambda *a, **kw: fake)

    # Stub the weather tool to avoid network
    from tools.registry import registry
    monkeypatch.setattr(
        registry.get("weather"), "handler",
        lambda args, **kw: '{"success": true, "data": "22C"}',
    )

    agent = AIAgent(provider="openrouter", model="x", api_key="sk-fake",
                    skip_memory=True, skip_context_files=True)
    out = agent.chat("how's the weather?")
    assert "nice out" in out
```

### Verify a slash command surfaces in the gateway

```python
def test_slash_status_returns_session_info(fake_runner):
    fake_runner.platforms["telegram"].inject(text="/status", user_id="u-1")
    asyncio.run(fake_runner.run_one_step())
    sent = fake_runner.platforms["telegram"].sent_messages
    assert sent
    assert "Session" in sent[0]["text"]
```

### Test compression triggers without running an LLM

```python
def test_compressor_kicks_in(monkeypatch):
    fake_engine = FakeContextEngine()
    fake_engine.set_should_compress(True)
    monkeypatch.setattr("agent.context_engine.load_engine",
                        lambda *a, **kw: fake_engine)
    agent = AIAgent(...)
    agent.run_conversation("hi")
    assert fake_engine.compress_calls == 1
```

### Snapshot a tool's schema

```python
def test_weather_schema_matches_snapshot(snapshot):
    from tools import weather_tool       # noqa: F401 — import to register
    schema = registry.get("weather").schema
    assert schema == snapshot
```

`pytest-syrupy` (`syrupy` package) provides the `snapshot` fixture;
update goldens with `pytest --snapshot-update`.

### Drive the curator without firing an LLM

```python
def test_curator_pins_used_skills(tmp_path, monkeypatch):
    monkeypatch.setenv("HERMES_HOME", str(tmp_path))
    write_skill_usage(usage=[("test-skill", 10)])
    fake_aux = FakeAuxiliaryClient(responses=[
        '{"action": "pin", "skill": "test-skill"}',
    ])
    monkeypatch.setattr("agent.curator._make_aux_client", lambda *a, **kw: fake_aux)
    Curator().run_once()
    assert is_pinned("test-skill")
```

### Stress test SessionDB writes

```python
@pytest.mark.slow
def test_concurrent_persist(tmp_path, monkeypatch):
    monkeypatch.setenv("HERMES_HOME", str(tmp_path))
    db = SessionDB()
    sid = db.create_session(source="t", model="t:t")

    def writer():
        for _ in range(100):
            db.persist_turn(sid, [{"role": "user", "content": "x"}])

    threads = [threading.Thread(target=writer) for _ in range(8)]
    for t in threads: t.start()
    for t in threads: t.join()

    assert len(db.get_messages(sid)) == 800
```

## 16. Snapshot tests

Snapshot tests are appropriate when the output is a stable, structured
shape. Examples:

* Tool JSON schemas (`tests/tools/__snapshots__/...`).
* Trajectory shapes (`tests/test_trajectory_compressor.py` golden
  files).
* Compressed message shapes after a compression pass.

Don't snapshot:

* Free text from a model (it shifts as models update).
* Anything containing timestamps or random IDs (use freezing /
  seeding first, or scrub before snapshotting).

## 17. Driving streams

For tests that care about streaming behaviour, use the chunk-level
APIs:

```python
def test_stream_emits_text_then_tool(monkeypatch):
    fake = script_streaming([
        chunk_text("Hel"),
        chunk_text("lo "),
        chunk_text("world"),
        chunk_tool_start(name="weather"),
        chunk_tool_args("{\"city\""),
        chunk_tool_args(":\"Tokyo\"}"),
        chunk_finish("tool_calls"),
    ])
    monkeypatch.setattr("run_agent._build_client", lambda *a, **kw: fake)

    deltas = []
    agent = AIAgent(...).with_callback(text=lambda d: deltas.append(d))
    agent.chat("...")
    assert "".join(deltas) == "Hello world"
```

## 18. Asserting on logs

Two patterns:

* **`caplog`** for per-test capture.
* **A logging handler that captures records** for higher-fidelity
  assertions:

  ```python
  records = []
  handler = logging.Handler()
  handler.emit = lambda r: records.append(r)
  logging.getLogger("agent").addHandler(handler)
  ```

Always assert on the message *and* the level; a downgraded warning
hides regressions otherwise.

## 19. Fixture composition

Tests are easier to maintain when fixtures are small and composable.
Pattern:

```python
@pytest.fixture
def session_id(tmp_path, monkeypatch):
    monkeypatch.setenv("HERMES_HOME", str(tmp_path))
    return SessionDB().create_session(source="t", model="t:t")

@pytest.fixture
def session_with_messages(session_id):
    db = SessionDB()
    db.persist_turn(session_id, [{"role": "user", "content": "hi"}])
    return session_id
```

Tests pull the depth they need:

```python
def test_lookup(session_with_messages):
    db = SessionDB()
    msgs = db.get_messages(session_with_messages)
    assert msgs[0]["content"] == "hi"
```

## 20. Common test smells

* **Test depends on the developer's `~/.hermes`** — missing
  `HERMES_HOME` redirect.
* **Test passes locally but fails in CI** — probable culprits: TZ,
  locale, flaky clock, resource ordering.
* **Test hangs** — agent loop expecting more scripted responses; or a
  background thread holding a lock.
* **Test catches `Exception`** — too broad; catch the specific
  exception class you expect.
* **Test asserts on the count of log lines** — log content shifts as
  features land. Assert on shape, not count.
* **Test imports the SDK** — e.g. `from openai import OpenAI` —
  delete it; use the fake.
* **Test hits the real network** — only OK with the `integration`
  marker.

## 21. Adding a fake

If you find yourself stubbing the same method in three tests, lift
the stub into a fake. Conventions:

* File name `fake_<thing>.py`.
* Class name `Fake<Thing>` (note: not `Mock<Thing>` — that hints
  unittest.mock).
* Constructor takes the minimum scripting args; everything else has
  sensible defaults.
* `calls` / `events` lists for verification.
* `inject(...)` / `add_response(...)` for scripting.
* No network, no disk, no subprocess.

Add a one-paragraph docstring explaining what surface the fake covers
and where the real version lives.

## 22. Where to look when…

| Symptom | Where |
|---------|-------|
| Test hangs forever | check the script length; bump `max_iterations` to surface the issue |
| Test passes alone, fails in suite | likely a global state leak — check fixtures + autouse |
| Test fails on Windows only | check path separators and env var quoting |
| Test fails under `-n auto` only | shared resource (port? filesystem path?) — make it tmp-scoped |
| Test fails on Python 3.13 only | typing or asyncio API change — check stdlib release notes |
| Snapshot churn | something non-deterministic in the data — freeze the clock, seed random |
| Provider-specific test fails after SDK upgrade | upgrade the matching fake to mirror the new shape |

## 23. Recommended reading order

For a contributor onboarding to the test suite:

1. `tests/conftest.py` — top to bottom (~200 lines).
2. `tests/run_agent/test_full_loop_smoke.py` — see how fakes drive
   the loop end-to-end.
3. `tests/fakes/fake_chat_completions_client.py` — see the canonical
   fake shape.
4. `tests/cli/test_*` — see the slash-command driver pattern.
5. `tests/gateway/test_*` — see the platform-adapter pattern.

After those, the rest of the suite reads predictably.
