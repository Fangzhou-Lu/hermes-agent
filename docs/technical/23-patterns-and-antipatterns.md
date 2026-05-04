# 23 — Patterns and Anti-patterns

This chapter is a consolidation of the coding patterns Hermes Agent
relies on, and the anti-patterns reviewers reject. It is meant as the
"how do we write code in this repo" reference — read it once, then
refer back when you are about to introduce a new abstraction.

Patterns are split by topic.

## 1. Module-level side effects

### Pattern: register-on-import for tools

`tools/<x>.py` modules call `tools.registry.register(...)` at module
top level. Discovery (`tools/registry.discover_builtin_tools`) finds
every file with such a call and imports just those files.

```python
# tools/my_tool.py
from tools.registry import registry

def my_handler(args, **kw): ...

registry.register(name="my_tool", toolset="...", schema={...},
                  handler=my_handler, ...)
```

Why: a single file is enough; no manual import list to keep in sync.

### Anti-pattern: registering inside a function

```python
def initialise():
    registry.register(...)
```

Discovery skips this file because the AST parser does not "see" the
register call at module level. The tool will not appear unless
something else imports the module first. Don't do it.

### Anti-pattern: top-level network / disk side effects

A tool file is imported during agent startup; expensive work at import
time slows every cold start. Defer expensive work to inside the
handler or to an explicit `initialise()` called once.

## 2. Lazy imports for heavy SDKs

### Pattern: lazy `_OpenAIProxy` style

Top of `run_agent.py` (`run_agent.py:75-90`) defines a tiny proxy that
imports `openai` only when an attribute is accessed:

```python
class _OpenAIProxy:
    def __getattr__(self, name):
        import openai
        return getattr(openai, name)
openai = _OpenAIProxy()
```

This shaves ~240 ms off cold start. Apply to: `openai`, `anthropic`,
`fire`, `mcp.server.fastmcp`, `account_usage`.

### Anti-pattern: blanket top-level `import openai`

A new contributor adds `import openai` at the top of `run_agent.py`
or `cli.py` to use one helper. The next user `hermes` invocation now
pays 240 ms more. Reviewers will reject the PR; use a lazy proxy or
local import inside the function that needs it.

## 3. Persistent paths

### Pattern: `get_hermes_home()` everywhere

```python
from hermes_constants import get_hermes_home

state_file = get_hermes_home() / "myfeature" / "state.json"
state_file.parent.mkdir(parents=True, exist_ok=True)
```

`get_hermes_home()` honours `HERMES_HOME`, the active profile, and
the managed-install marker. It is the only correct way to compute
`~/.hermes/...`.

### Pattern: `display_hermes_home()` in user-visible strings

When a path is shown to the user (tool schema description, log line,
prompt block), use `display_hermes_home()` so the rendered string is
`~/.hermes/...` regardless of the active profile.

### Anti-pattern: hardcoded `Path.home() / ".hermes"`

Breaks profile isolation. Breaks managed installs. Breaks anyone who
sets `HERMES_HOME`. Reviewers reject.

### Anti-pattern: `os.path.expanduser("~/.hermes/...")`

Same problem.

### Anti-pattern: storing config-shaped data in `.env`

`.env` is for secrets only. Settings go in `config.yaml`. `hermes
setup` will not write non-secret keys to `.env`.

## 4. Atomic writes

### Pattern: `utils.atomic_*` helpers

```python
from utils import atomic_replace, atomic_json_write, atomic_yaml_write

atomic_json_write(path, data)
atomic_yaml_write(path, config)
atomic_replace(temp_path, target_path)  # preserves symlinks
```

Why: durable state can be corrupted by a half-applied write
(`SIGKILL` mid-write, container OOM, …). Atomic helpers use temp +
fsync + os.replace.

### Anti-pattern: direct `path.write_text(json.dumps(...))`

A crash mid-write leaves a truncated JSON file. Next startup fails to
parse and the user is left with a broken `~/.hermes`.

## 5. Time

### Pattern: `hermes_time.now()`

```python
from hermes_time import now as hermes_now
ts = hermes_now()
```

Honours `config.timezone`, is freezable in tests via the conftest
fixture, and produces tz-aware datetimes.

### Anti-pattern: `datetime.now()`

Naive (no tz). Cannot be frozen. Tests become flaky in foreign
timezones.

### Anti-pattern: `datetime.utcnow()`

Deprecated since Python 3.12. Use `datetime.now(UTC)` if you really
need raw UTC, but prefer `hermes_time.now()`.

## 6. Logging

### Pattern: `logger = logging.getLogger(__name__)`

```python
logger = logging.getLogger(__name__)

logger.info("starting widget %s", widget_id)
logger.warning("widget %s degraded: %s", widget_id, reason)
logger.exception("widget %s crashed", widget_id)  # in except blocks
```

Why: `hermes_logging.RedactingFormatter` strips secrets from every
log record. The session-context filter tags lines with the active
session id.

### Anti-pattern: `print()`

Goes to stdout, never reaches the redacting formatter, never gets
session-tagged, breaks the JSON-RPC channel for ACP/MCP.

### Anti-pattern: f-string in `logger.info(f"...{secret}...")`

The f-string is evaluated *before* the logging system sees it; the
redactor only runs on the final formatted message, which is too late
if a custom handler captured it. Use `%`-style placeholders for
anything that might contain secrets:

```python
logger.info("api request: %s", redact(payload))
```

### Anti-pattern: logging the raw request body

API request bodies often include the system prompt, which often
includes user-injected memory blocks, which sometimes include
secrets. Always pass through `agent.redact.redact()` first.

## 7. Async ↔ sync bridging

### Pattern: `model_tools._run_async(coro)`

```python
from model_tools import _run_async

def my_sync_tool(args, **kw):
    return _run_async(my_async_helper(args))
```

`_run_async` keeps a per-thread persistent event loop so `httpx`
`AsyncOpenAI`, `AsyncAnthropic` clients survive multiple calls.

### Anti-pattern: `asyncio.run()` inside a tool

Creates a fresh loop per call. Cached async clients raise "Event loop
is closed" on the second call. Tests in
`tests/test_model_tools_async_bridge.py` cover this.

### Anti-pattern: `asyncio.get_event_loop().run_until_complete(...)`

Behaviour changes in Python 3.12+; raises a deprecation warning if no
loop is running, then crashes once the warning becomes an error. Use
`_run_async`.

## 8. Tool handlers

### Pattern: handlers return JSON strings

```python
def handler(args, **kw) -> str:
    try:
        result = do_work(args)
        return json.dumps({"success": True, "data": result})
    except Exception as e:
        logger.exception("tool failed")
        return json.dumps({"success": False, "error": str(e)})
```

### Anti-pattern: handler returns dict / list

The agent loop assumes a string. Returning a dict crashes the
serialisation layer.

### Anti-pattern: handler raises on user-visible errors

A model misuse should surface as `{"success": False, "error": ...}`,
not a Python exception. Save raises for genuine bugs and explicit
interrupts (`tools.interrupt.InterruptException`).

### Pattern: register `max_result_size_chars`

```python
registry.register(..., max_result_size_chars=80_000)
```

The default 50 KB is enough for most tools; bump for tools that
return long but useful data (terminal stdout, web scrapes).

### Anti-pattern: returning multi-megabyte tool results

Tokens are expensive and the truncation will silently lose context.
Either truncate in the handler with a clear "...truncated..." marker,
or write to `tool_result_storage` and return a handle.

## 9. Path security

### Pattern: `path_security.is_within(root, candidate)`

```python
from tools.path_security import is_within, sandbox_root

if not is_within(sandbox_root(), Path(user_provided)):
    return json.dumps({"success": False, "error": "path outside sandbox"})
```

Uses `realpath` so symlink-out-of-sandbox tricks are blocked.

### Anti-pattern: `path.startswith(str(root))`

Doesn't follow symlinks. Doesn't normalise. Trivially defeated.

## 10. Fixed file formats

### Pattern: stable JSONL one-event-per-line

Trajectory files (`trajectory_samples.jsonl`,
`failed_trajectories.jsonl`, `~/.hermes/conversations/<id>.jsonl`) are
all one JSON event per line. Append-only. Safe to tail with `jq`.

### Anti-pattern: serialising a list as one big JSON object that
grows with each turn

A 30-MB JSON object is slow to write (full rewrite per append) and
catastrophic if a write is interrupted. JSONL is append-only and
crash-safe.

## 11. Configuration

### Pattern: deep-merge over `DEFAULT_CONFIG`

```python
from hermes_cli.config import load_config
config = load_config()
my_value = config.get("my_section", {}).get("my_key", default)
```

`load_config()` already deep-merges over `DEFAULT_CONFIG`. Adding a
new key inside an existing section requires no migration —
`_config_version` only bumps for active migrations.

### Pattern: optional env var declared in `OPTIONAL_ENV_VARS`

Users configure secrets via `hermes setup` or by editing `.env`
directly. The wizard reads `OPTIONAL_ENV_VARS` so the new key gets
surfaced automatically.

### Anti-pattern: reading `os.environ.get("MY_KEY")` without
declaring it

`hermes doctor` cannot detect missing keys. `hermes setup` cannot
prompt for them. The user gets a runtime error nobody can predict.

### Anti-pattern: hardcoding a per-provider endpoint URL

Make it overridable via `MY_PROVIDER_BASE_URL` so users can point at
a self-hosted compatible endpoint.

## 12. Provider transports

### Pattern: subclass `ProviderTransport`

Format translation lives in `agent/transports/<provider>.py`. Quirks
live in `agent/<provider>_adapter.py`. Keep the boundary clean.

### Anti-pattern: putting Anthropic-specific behaviour in
`chat_completions.py`

The `chat_completions` transport is supposed to be vendor-neutral.
Anthropic-flavoured logic goes in `agent/transports/anthropic.py` or
`agent/anthropic_adapter.py`.

### Anti-pattern: silently dropping fields the provider does not
support

Better: warn once and degrade to the closest equivalent. The user
should know if their `--reasoning-effort high` request fell back
because the provider does not support it.

## 13. Credential handling

### Pattern: route through `CredentialPool`

```python
cred = pool.next(provider)
client = build_client(api_key=cred.access_token, base_url=...)
try:
    resp = client.chat.completions.create(...)
except RateLimitError as e:
    pool.mark_exhausted(cred, reset_at=parse_reset(e))
    raise
```

The pool handles rotation, retry-after, and custom-endpoint pool
keys.

### Anti-pattern: caching a single client at module level

```python
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])  # module level
```

Defeats the credential pool. Defeats fallback providers. Defeats
profile isolation. Always build the client per request through the
pool.

## 14. Memory providers

### Pattern: implement the full lifecycle

Every method on `MemoryProvider` has a default no-op implementation.
Override only the ones you need:

```python
class TagstoreProvider(MemoryProvider):
    def is_available(self) -> bool: ...
    def initialize(self) -> None: ...
    def system_prompt_block(self) -> str: ...
    def get_tool_schemas(self) -> list[dict]: ...
    def handle_tool_call(self, name, args) -> str: ...
    def shutdown(self) -> None: ...
```

### Pattern: short, focused `system_prompt_block`

Memory blocks are injected on every turn. A long block costs tokens
on every turn and trains the model to ignore it. Aim for ≤300 tokens.

### Anti-pattern: sync_turn doing heavy work synchronously

`sync_turn` runs in the agent loop's hot path. Move expensive work
(embedding, network) into `prefetch_all` (pre-turn, can be async) or
a background thread.

## 15. Context engine

### Pattern: respect the protect-first / protect-last invariants

A custom `ContextEngine.compress()` must not delete:

* The first system prompt.
* The first user message.
* The first assistant message.
* The first tool result.
* The last `protect_last_n` turns.

These are what let the model maintain a coherent task across
compactions. A compressor that drops them produces gibberish.

### Anti-pattern: replacing every middle turn with the same summary

Successive compactions then keep flattening the summary, losing
detail each pass. Use the iterative-summary pattern (extend the prior
summary with the new middle, do not regenerate from scratch).

## 16. Skills

### Pattern: small `SKILL.md`, references for detail

```
skill-name/
├── SKILL.md         # ≤200 lines: when, why, procedure
├── references/
│   └── advanced.md  # only loaded if needed
└── templates/
    └── starter.py   # copied into the workspace
```

Progressive disclosure keeps the system-prompt cost low.

### Pattern: conditions for OS / env gates

```yaml
conditions:
  - os == "macos"
  - has_command("git")
```

Hard gates beat soft `prerequisites` for capability filtering.

### Anti-pattern: secrets in the body

The skill body is fed to the model and may end up in trajectories.
Use `{{ env.X }}` substitution instead.

### Anti-pattern: one giant skill that does many things

The model has trouble loading and using huge skills. Split into
several focused skills with `related_skills` cross-references.

### Anti-pattern: `prerequisites.commands` as a security gate

`prerequisites` is *advisory*. The skill loads even if a listed
command is missing. Use `conditions:` for hard gates.

## 17. Tests

### Pattern: one fixture per fact

```python
@pytest.fixture
def sample_session(tmp_path, monkeypatch):
    monkeypatch.setenv("HERMES_HOME", str(tmp_path))
    db = SessionDB()
    return db.create_session(source="test", model="test:test")
```

A fixture supplies a single named state. Compose them rather than
building one mega-fixture.

### Pattern: monkey-patch the build helper, not the SDK

```python
monkeypatch.setattr("run_agent._build_client", lambda *a, **kw: fake)
```

`_build_client` is the seam Hermes owns. Patching it is stable across
SDK upgrades.

### Anti-pattern: `monkeypatch.setattr("openai.OpenAI", ...)`

Couples the test to the SDK shape. Breaks on minor SDK upgrades.

### Pattern: mark integration tests

```python
@pytest.mark.integration
def test_real_anthropic_call(): ...
```

`scripts/run_tests.sh` excludes them; they run in the nightly job.

### Anti-pattern: live tests in the default suite

A flaky network call in CI blocks every PR. Mark integration; use
`tests/fakes/` for unit coverage.

## 18. Streaming

### Pattern: process chunks as they arrive

```python
for chunk in transport.stream(messages, tools):
    if chunk.kind == "text":
        callback.on_text(chunk.delta)
    elif chunk.kind == "tool_start":
        callback.on_tool_start(chunk)
    ...
```

The CLI relies on early text deltas to keep the spinner alive and the
user happy.

### Anti-pattern: collecting the whole response before yielding

Defeats streaming. The CLI shows nothing for tens of seconds.

## 19. CLI surfaces

### Pattern: one `CommandDef` is the source of truth

Add a `CommandDef`, then add the dispatch arms in CLI / gateway / ACP.
Help, autocomplete, Telegram BotCommand and Slack subcommand all
derive from `COMMAND_REGISTRY`.

### Anti-pattern: bespoke parsing of slash commands in a single
surface

The command shows up in CLI but not in Telegram, or vice versa, and
the user is confused. Always go through `resolve_command()`.

### Anti-pattern: blocking on stdin from inside a slash handler

prompt_toolkit owns stdin. Use the in-flight modal helpers
(`hermes_cli/callbacks.py` provides `prompt_for_input`, `prompt_for_choice`).

## 20. Approvals

### Pattern: classify destructive operations as such

Tools that destroy data or send messages should be on the high-risk
list (`tools/approval.py`). The user can always allow-list specific
patterns; what they cannot do is opt back *in* to a forgotten
default-safe behaviour.

### Anti-pattern: a "skip approval" flag on a destructive tool

This is an escalation. Use `command_allowlist` for granular
allow-listing. The exception is `/yolo` mode, which is global and
clearly opt-in.

## 21. Plugins

### Pattern: declare hooks in `plugin.yaml`

```yaml
hooks:
  - on_session_end
  - on_tool_call
```

The loader subscribes the plugin's callbacks automatically.

### Anti-pattern: the plugin's `__init__.py` mutates global state on
import

Plugins are loaded eagerly when enabled. Side effects at import time
mean the plugin runs even when the user has merely *enabled* it (not
necessarily used a feature). Defer to a hook callback.

## 22. The dashboard / TUI boundary

### Pattern: dashboard adds *around* the TUI, not over it

Sidebars, model pickers, kanban panels — all fine. The chat surface
itself stays in Ink + the embedded PTY.

### Anti-pattern: re-implementing the chat transcript in React

Loses streaming behaviour, slash-command parity, approval flow, voice
mode. Reviewers reject.

## 23. Tracing identifiers

### Pattern: thread session id through tool calls

`task_id` is a per-tool-call id; `session_id` is the conversation id;
`request_id` is the per-provider-request id. Logs and trajectories
should carry all three so a downstream reader can reconstruct.

### Anti-pattern: passing logger names around as identifiers

Logger names are for grouping, not identity.

## 24. Naming

### Pattern: lower-snake for Python; lower-kebab for skills/files
that the user types

| Class | `MyAdapterClass` |
| Function | `do_widget_thing` |
| Skill name | `test-driven-development` |
| Slash command | `/test-driven-development` (matches skill name) |
| Config key | `agent.max_turns` |
| Env var | `HERMES_FOO` / `<PROVIDER>_API_KEY` |

### Anti-pattern: shouty `MY_VAR`-style attribute names

Reserved for module-level constants. Local vars and config keys are
lower.

## 25. Comments and docstrings

### Pattern: short module docstring

```python
"""
Background curator — pin/archive/consolidate skills when idle.
"""
```

Reading the first line tells a contributor what the file is for.

### Anti-pattern: line-by-line inline comments

The code should read; comments belong where the *why* is non-obvious.

### Anti-pattern: `// removed` / `// TODO` markers in committed code

Use git history to track removals. TODOs go on the issue tracker,
not in the source tree.

## 26. Public API stability

### Pattern: function names are stable, line numbers are not

Documentation refers to `hermes_cli/commands.py:resolve_command()`
rather than `hermes_cli/commands.py:222`. Refactors shift line
numbers; function names survive.

### Anti-pattern: relying on a private helper from outside its file

If `_build_kwargs()` in `agent/transports/anthropic.py` is needed
elsewhere, lift it into a public helper in `agent/transports/types.py`
or a similar shared module before depending on it.

## 27. Testing surfaces

### Pattern: drive the full agent loop in e2e tests

For features that span many files, the e2e test that drives
`AIAgent.chat(...)` with a faked transport catches integration bugs
unit tests would miss.

### Anti-pattern: testing private helpers in isolation only

You can ship a feature where every helper test passes but the
integration is broken. Belt + braces: unit tests for helpers, e2e
tests for the surface.

## 28. Git hygiene

### Pattern: small commits, descriptive messages

```
fix(cli): preserve cwd when reloading after `hermes update`

Symlinks broke because the relaunch path normalised before
calling exec. Use Path.absolute() instead of Path.resolve().
```

### Anti-pattern: "wip", "fix bug", or 50-file commits

Reviewers cannot bisect. Squash before merging if you must work in
small WIP commits locally.

### Pattern: keep generated artifacts out

Skills index, model catalog, TUI bundle — generated by scripts; do
*not* commit them.

### Anti-pattern: amending a published commit

Once pushed, amending forces a force-push, which can clobber other
contributors' work. New commit instead.

## 29. Releases

### Pattern: every `RELEASE_v*.md` lists user-visible changes

Internal refactors do not need a line; behaviour changes do.

### Anti-pattern: bumping `_config_version` for additive changes

Reserved for renames / restructures.

## 30. Documentation

### Pattern: link to source files with `:line` references

Examples: `tools/registry.py:143`, `hermes_cli/commands.py:64`.
Readers can jump straight to the relevant code.

### Anti-pattern: paraphrasing what the code "does"

Show the function signature; paraphrase only the *why*.

### Pattern: place docs adjacent to code

Subsystem-specific notes live in the relevant package's
`README.md` or as comments inline. The technical doc set under
`docs/technical/` is the *cross-cutting* reference.

## 31. The "no half-finished" rule

If a PR adds a new tool, also:

* Add a test.
* Update `toolsets.py`.
* Update `OPTIONAL_ENV_VARS` if a secret is required.
* Add a row to the relevant doc (or update the table).
* Add a `SKILL.md` only if the feature is broadly useful — not
  speculative.

Half-applied features rot fast. Keep the working set small enough to
land complete.

## 32. Final checklist for any PR

* [ ] No `print()`.
* [ ] No `Path.home() / ".hermes"`.
* [ ] No `datetime.now()` without explicit reason.
* [ ] No top-level `import openai` / `import anthropic` / heavy SDK
      imports.
* [ ] All persistent writes are atomic.
* [ ] All tool handlers return JSON strings.
* [ ] All shell-tool inputs are quoted via the backend's helper.
* [ ] All paths in tool outputs are within the sandbox or use
      `display_hermes_home()`.
* [ ] All new env vars are listed in `OPTIONAL_ENV_VARS`.
* [ ] `_config_version` not bumped unless required.
* [ ] At least one new test.
* [ ] Docs updated.
* [ ] No commented-out code; no spurious TODOs.
