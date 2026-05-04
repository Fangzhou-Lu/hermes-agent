# 16 — Extension Recipes

This chapter is a cookbook. Each recipe is a complete, copy-pastable
walkthrough for a common extension: a new tool, a new provider, a new
platform, a new memory provider, a new context engine, a new plugin, a
new skill, a new slash command, a new cron-delivered job. The recipes
assume a clean dev checkout per
[07-developer-guide.md](07-developer-guide.md).

## Recipe 1 — A new tool

**Goal:** add a `weather` tool that fetches a city's current weather
from a public API.

### Step 1. Create the tool file

`tools/weather_tool.py`:

```python
"""Weather tool — fetches current weather via wttr.in."""
from __future__ import annotations

import json
import logging
import os
import urllib.parse
import urllib.request

from tools.registry import registry

logger = logging.getLogger(__name__)


def _check() -> bool:
    # No API key needed for wttr.in. Return True unconditionally,
    # but pretend a key gate exists so the pattern is visible.
    return True


def _fetch_weather(city: str) -> dict:
    url = "https://wttr.in/" + urllib.parse.quote(city) + "?format=j1"
    with urllib.request.urlopen(url, timeout=10) as resp:
        return json.loads(resp.read().decode())


def weather(city: str, units: str = "metric", task_id: str | None = None) -> str:
    if not city.strip():
        return json.dumps({"success": False, "error": "city is required"})

    try:
        data = _fetch_weather(city)
    except Exception as e:
        logger.exception("weather fetch failed")
        return json.dumps({"success": False, "error": str(e)})

    current = data.get("current_condition", [{}])[0]
    if units == "imperial":
        temp = current.get("temp_F", "?")
        unit = "F"
    else:
        temp = current.get("temp_C", "?")
        unit = "C"

    return json.dumps({
        "success": True,
        "city": city,
        "temp": f"{temp}{unit}",
        "feels_like": current.get("FeelsLikeC", "?") + "C",
        "summary": current.get("weatherDesc", [{}])[0].get("value", ""),
        "humidity": current.get("humidity", "?"),
    })


registry.register(
    name="weather",
    toolset="weather",
    schema={
        "name": "weather",
        "description": (
            "Fetch the current weather for a city. Use this when the user "
            "asks about the weather conditions in a specific location."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "City name, e.g. 'Berlin' or 'New York, NY'.",
                },
                "units": {
                    "type": "string",
                    "enum": ["metric", "imperial"],
                    "default": "metric",
                },
            },
            "required": ["city"],
        },
    },
    handler=lambda args, **kw: weather(
        city=args.get("city", ""),
        units=args.get("units", "metric"),
        task_id=kw.get("task_id"),
    ),
    check_fn=_check,
    requires_env=[],
    description="Fetch current weather for a city",
    emoji="🌦️",
    max_result_size_chars=8000,
)
```

### Step 2. Register the toolset

In `toolsets.py`, add `"weather"` to `_HERMES_CORE_TOOLS` (if it should
be available everywhere) **or** add a new entry to `TOOLSETS`:

```python
TOOLSETS["weather"] = {
    "description": "Weather lookup",
    "tools": ["weather"],
}
```

### Step 3. Test

`tests/tools/test_weather_tool.py`:

```python
import json

from tools import weather_tool  # noqa: F401 — register on import


def test_weather_returns_error_for_empty_city():
    out = weather_tool.weather("")
    assert json.loads(out) == {"success": False, "error": "city is required"}


def test_registry_carries_weather():
    from tools.registry import registry
    schemas = {t.name: t for t in registry.entries()}
    assert "weather" in schemas
    assert schemas["weather"].toolset == "weather"
```

### Step 4. Use it

```
$ hermes
> /tools enable weather
> What's the weather in Tokyo?
```

## Recipe 2 — A new provider (OpenAI-compatible)

**Goal:** add support for a hypothetical "Acme AI" service whose API is
OpenAI-compatible at `https://api.acme.example/v1`.

You **don't** need a new transport — `chat_completions` handles
OpenAI-compatible APIs. You only need configuration.

### Step 1. Add the credential

`hermes_cli/config.py`, in `OPTIONAL_ENV_VARS`:

```python
"ACME_API_KEY": {
    "description": "Acme AI API key",
    "prompt": "Acme AI API key",
    "url": "https://acme.example/keys",
    "password": True,
    "category": "provider",
    "advanced": True,
},
"ACME_BASE_URL": {
    "description": "Acme AI base URL override",
    "prompt": "Acme AI base URL (leave empty for default)",
    "url": None,
    "password": False,
    "category": "provider",
    "advanced": True,
},
```

### Step 2. Add catalog entry

`hermes_cli/model_catalog.py` — append a `ProviderEntry`:

```python
ProviderEntry(
    id="acme",
    label="Acme AI",
    base_url="https://api.acme.example/v1",
    api_mode="chat_completions",
    auth_env="ACME_API_KEY",
    base_url_env="ACME_BASE_URL",
    models=[
        ModelEntry(id="acme-large", display="Acme Large", context_length=128_000),
        ModelEntry(id="acme-fast",  display="Acme Fast",  context_length=32_000),
    ],
),
```

### Step 3. Test

```
$ hermes auth acme
$ hermes model acme:acme-large
$ hermes -p "hello"
```

That's it. The `chat_completions` transport runs the rest.

## Recipe 3 — A new provider (custom protocol)

**Goal:** add a non-OpenAI provider, e.g. one that returns SSE events
in a different shape.

### Step 1. Subclass `ProviderTransport`

`agent/transports/myprovider.py`:

```python
from __future__ import annotations

from typing import Any, Iterable

from agent.transports.base import ProviderTransport
from agent.transports.types import (
    NormalizedResponse, ToolCall, Usage, build_tool_call, map_finish_reason,
)


class MyProviderTransport(ProviderTransport):
    api_mode = "myprovider"

    def convert_messages(self, messages):
        # OpenAI → MyProvider message shape
        return [{"role": m["role"], "text": m["content"]} for m in messages]

    def convert_tools(self, tools):
        return [{"id": t["function"]["name"], "schema": t["function"]["parameters"]}
                for t in tools]

    def build_kwargs(self, *, messages, tools, model, **kw):
        return {
            "model": model,
            "input": self.convert_messages(messages),
            "tools": self.convert_tools(tools) if tools else None,
            "stream": True,
        }

    def normalize_response(self, raw) -> NormalizedResponse:
        text = raw.get("output", {}).get("text", "")
        tool_calls: list[ToolCall] = []
        for tc in raw.get("output", {}).get("tools", []):
            tool_calls.append(build_tool_call(
                id=tc["id"], name=tc["name"], arguments=tc["args"],
            ))
        return NormalizedResponse(
            text=text,
            tool_calls=tool_calls,
            usage=Usage(
                input_tokens=raw["usage"]["in"],
                output_tokens=raw["usage"]["out"],
            ),
            finish_reason=map_finish_reason(
                raw.get("stop_reason"),
                {"end": "stop", "tool": "tool_calls", "limit": "length"},
            ),
        )
```

### Step 2. Add an adapter (optional)

If the provider has quirks (max-output table, OAuth setup-tokens,
custom headers), put them in `agent/myprovider_adapter.py` and wire
them into the transport.

### Step 3. Wire up

* Register in `hermes_cli/model_catalog.py`.
* Map `provider="myprovider"` to `api_mode="myprovider"` in
  `run_agent.py:_select_transport()`.

### Step 4. Test

`tests/agent/test_myprovider_transport.py` — feed canned wire-format
data through `normalize_response` and assert the shape.

## Recipe 4 — A new platform

**Goal:** integrate a fictional chat platform "Yapper" into the
gateway.

### Step 1. Create the adapter

`gateway/platforms/yapper.py`:

```python
from __future__ import annotations

import asyncio
import logging

from gateway.platforms.base import (
    BasePlatformAdapter, MessageEvent, MessageType,
)

logger = logging.getLogger(__name__)


class YapperAdapter(BasePlatformAdapter):
    name = "yapper"

    def __init__(self, config: dict, on_message):
        self._config = config
        self._on_message = on_message
        self._task: asyncio.Task | None = None

    async def connect(self):
        self._task = asyncio.create_task(self._run_event_loop())

    async def disconnect(self):
        if self._task:
            self._task.cancel()

    async def send_message(self, chat_id: str, text: str, **kw):
        # Call the Yapper SDK or HTTP endpoint
        ...

    async def send_file(self, chat_id: str, path: str, **kw):
        ...

    async def _run_event_loop(self):
        async for raw in self._yapper_stream():
            event = MessageEvent(
                source={"platform": "yapper", "chat_id": raw["chat"]},
                text=raw["text"],
                user_id=raw["user_id"],
                channel_id=raw["chat"],
                message_type=MessageType.TEXT,
                attachments=[],
                timestamp=raw["ts"],
            )
            await self._on_message(event)
```

### Step 2. Register

`gateway/platforms/__init__.py` — import the module so it self-registers
(or wire it up explicitly in `gateway/platform_registry.py`).

### Step 3. Config + secrets

* Add `yapper` block to `DEFAULT_CONFIG` in `hermes_cli/config.py`.
* Add `YAPPER_TOKEN`, `YAPPER_HOME_CHANNEL`, `YAPPER_ALLOWED_USERS`
  to `OPTIONAL_ENV_VARS`.

### Step 4. Tests

`tests/gateway/platforms/test_yapper_adapter.py` — drive the adapter
with a fake SDK stream and assert `MessageEvent` shapes.

### Step 5. `gateway/platforms/ADDING_A_PLATFORM.md`

Update the docs to note the new platform.

## Recipe 5 — A new memory provider

**Goal:** add a simple "tags-based" memory provider that stores `tag →
value` pairs.

### Step 1. Plugin scaffolding

`plugins/memory/tagstore/`:

```
plugins/memory/tagstore/
├── __init__.py
├── plugin.yaml
├── provider.py
└── README.md
```

`plugin.yaml`:

```yaml
name: tagstore
version: 1.0.0
description: "Tag-based memory provider — store and retrieve key/value pairs."
pip_dependencies: []
hooks: []
```

### Step 2. Provider implementation

`provider.py`:

```python
from __future__ import annotations

import json
from pathlib import Path

from agent.memory_provider import MemoryProvider
from hermes_constants import get_hermes_home


class TagStoreProvider(MemoryProvider):
    name = "tagstore"
    file_path = lambda self: get_hermes_home() / "memory" / "tagstore.json"

    def is_available(self) -> bool:
        return True

    def initialize(self) -> None:
        self.file_path().parent.mkdir(parents=True, exist_ok=True)
        if not self.file_path().exists():
            self.file_path().write_text("{}")
        self._cache = json.loads(self.file_path().read_text())

    def system_prompt_block(self) -> str:
        if not self._cache:
            return ""
        items = "\n".join(f"  - {k} = {v}" for k, v in sorted(self._cache.items()))
        return f"<TAG_MEMORY>\n{items}\n</TAG_MEMORY>"

    def get_tool_schemas(self) -> list[dict]:
        return [{
            "name": "tag_set",
            "description": "Store a key/value tag.",
            "parameters": {
                "type": "object",
                "properties": {
                    "key": {"type": "string"},
                    "value": {"type": "string"},
                },
                "required": ["key", "value"],
            },
        }]

    def handle_tool_call(self, name: str, args: dict) -> str:
        if name == "tag_set":
            self._cache[args["key"]] = args["value"]
            self.file_path().write_text(json.dumps(self._cache, indent=2))
            return json.dumps({"success": True})
        return json.dumps({"success": False, "error": f"unknown tool: {name}"})

    def shutdown(self) -> None:
        pass
```

### Step 3. Register

`__init__.py`:

```python
from .provider import TagStoreProvider

def get_provider(config):
    return TagStoreProvider()
```

### Step 4. Enable

```bash
hermes plugins enable memory/tagstore
```

`config.memory.provider: "tagstore"` will start using it after
restart.

## Recipe 6 — A new context engine

**Goal:** a "no-op" engine that never compresses (for testing or
research).

`plugins/context_engine/passthrough/`:

```
plugins/context_engine/passthrough/
├── __init__.py
├── plugin.yaml
└── engine.py
```

`engine.py`:

```python
from agent.context_engine import ContextEngine


class PassthroughEngine(ContextEngine):
    name = "passthrough"

    def __init__(self, *, threshold_tokens: int, context_length: int, **kw):
        self.threshold_tokens = threshold_tokens
        self.context_length = context_length
        self.last_prompt_tokens = 0
        self.last_completion_tokens = 0
        self.compression_count = 0

    def update_from_response(self, response):
        self.last_prompt_tokens = response.usage.input_tokens
        self.last_completion_tokens = response.usage.output_tokens

    def should_compress(self) -> bool:
        return False

    def should_compress_preflight(self) -> bool:
        return False

    def has_content_to_compress(self) -> bool:
        return False

    def compress(self, messages):
        return messages
```

`plugin.yaml`:

```yaml
name: passthrough
version: 1.0.0
description: "Context engine that never compresses."
pip_dependencies: []
```

`hermes plugins enable context_engine/passthrough`, then set
`context.engine: "passthrough"` in `config.yaml`.

## Recipe 7 — A new plugin (with hook)

**Goal:** a plugin that logs every session end to a `~/.hermes/logs/sessions.csv`.

`plugins/session-logger/`:

```
plugins/session-logger/
├── __init__.py
└── plugin.yaml
```

`plugin.yaml`:

```yaml
name: session-logger
version: 1.0.0
description: "Append every session_end to ~/.hermes/logs/sessions.csv."
hooks:
  - on_session_end
```

`__init__.py`:

```python
import csv
from datetime import datetime, timezone
from pathlib import Path

from gateway.hooks import register
from hermes_constants import get_hermes_home


@register("on_session_end")
def append_csv(session_id: str, summary: dict):
    path = get_hermes_home() / "logs" / "sessions.csv"
    path.parent.mkdir(parents=True, exist_ok=True)
    new = not path.exists()
    with path.open("a", newline="") as f:
        writer = csv.writer(f)
        if new:
            writer.writerow(["timestamp", "session_id", "messages", "tools", "tokens_in", "tokens_out"])
        writer.writerow([
            datetime.now(timezone.utc).isoformat(),
            session_id,
            summary.get("message_count", 0),
            summary.get("tool_call_count", 0),
            summary.get("input_tokens", 0),
            summary.get("output_tokens", 0),
        ])
```

`hermes plugins enable session-logger` — done. Every gateway session
end appends a row.

## Recipe 8 — A new skill

**Goal:** a skill that runs a code-format pre-commit pass.

`skills/software-development/format-and-fix/SKILL.md`:

```markdown
---
name: format-and-fix
description: "Run the project formatters (ruff, prettier, etc.) and stage the changes."
version: 1.0.0
prerequisites:
  commands: [git]
metadata:
  hermes:
    tags: [git, formatting, pre-commit]
---

# Format and fix

## When to use
- Before committing.
- After accepting a multi-file diff.

## Procedure

1. Detect formatters in this project:

```bash
ls -1 pyproject.toml package.json .pre-commit-config.yaml 2>/dev/null
```

2. Run each that exists:

```bash
[ -f .pre-commit-config.yaml ] && pre-commit run --all-files || true
[ -f pyproject.toml ] && command -v ruff >/dev/null && ruff check --fix . || true
[ -f package.json ] && command -v prettier >/dev/null && npx prettier --write . || true
```

3. Stage and report:

```bash
git add -A
git status
```
```

Now `/format-and-fix` is available as a slash command in CLI and
gateway.

## Recipe 9 — A new slash command

**Goal:** a `/dot-files` command that lists user dotfiles in the home
directory.

### Step 1. CommandDef

`hermes_cli/commands.py` — append to `COMMAND_REGISTRY`:

```python
CommandDef(
    "dot-files", "List dotfiles in $HOME", "Info",
    aliases=("dots",),
),
```

### Step 2. CLI handler

`cli.py`, in `HermesCLI.process_command`:

```python
elif canonical == "dot-files":
    self._handle_dotfiles()
    return
```

```python
def _handle_dotfiles(self):
    p = Path.home()
    items = sorted(item.name for item in p.iterdir() if item.name.startswith("."))
    self._print(f"Dotfiles in {p}:")
    for name in items:
        self._print(f"  {name}")
```

### Step 3. Gateway handler (optional)

`gateway/run.py`, in `_handle_message` slash dispatch:

```python
if canonical == "dot-files":
    return await self._handle_dotfiles_gateway(event)
```

### Step 4. Test

`tests/cli/test_dotfiles_command.py` — drive `process_command` with a
mocked `Path.home()` and assert output.

## Recipe 10 — A scheduled cron job

**Goal:** a daily standup summary delivered to Telegram every weekday
at 08:30.

### Via the CLI

```bash
hermes cron add \
  --name "morning-standup" \
  --schedule "30 8 * * 1-5" \
  --prompt "Generate a morning standup summary based on yesterday's commits and today's calendar." \
  --skill standup-template \
  --deliver-to "telegram:home" \
  --toolset "research"
```

### Inside `~/.hermes/cron/jobs.json`

```json
{
  "id": "job-morning-standup-7c4e",
  "name": "morning-standup",
  "schedule": "30 8 * * 1-5",
  "prompt": "Generate a morning standup summary ...",
  "skills": ["standup-template"],
  "deliver_to": ["telegram:home"],
  "enabled_toolsets": ["research"],
  "enabled": true,
  "next_run": "2026-04-13T08:30:00-04:00",
  "last_run": null,
  "last_status": null
}
```

### Behaviour

* The gateway must be running for delivery.
* Output is also written to
  `~/.hermes/cron/output/job-morning-standup-7c4e/<ts>.md`.
* `hermes cron run job-morning-standup-7c4e` triggers it manually.

## Recipe 11 — Adding a config knob

**Goal:** expose a new `agent.greeting` string used at session start.

### Step 1. Default

`hermes_cli/config.py:DEFAULT_CONFIG["agent"]`:

```python
"greeting": "Hello! How can I help today?",
```

### Step 2. Read it

In `cli.py:HermesCLI.__init__`:

```python
self._greeting = self._config.get("agent", {}).get("greeting", "Hello!")
...
self._print(self._greeting)
```

### Step 3. Bump `_config_version`?

Only if you are renaming or restructuring an existing key. Adding a
new key inside an existing section (`agent`) is handled automatically
by deep-merge. So **no bump** here.

### Step 4. Surface

`hermes config get agent.greeting` and `hermes config set
agent.greeting "..."` already work via the dotted-path resolver.

## Recipe 12 — Adding an `OPTIONAL_ENV_VARS` entry

**Goal:** support `MY_TOOL_API_KEY`.

```python
"MY_TOOL_API_KEY": {
    "description": "API key for the my_tool tool",
    "prompt": "MyTool API key",
    "url": "https://my-tool.example/keys",
    "password": True,
    "tools": ["my_tool"],
    "category": "tool",
},
```

After adding, `hermes setup` will prompt for it during the wizard, the
tool's `requires_env=["MY_TOOL_API_KEY"]` gate becomes effective, and
`hermes auth my-tool` (if you wire one) will recognise it.

## Recipe 13 — A managed-MCP gateway adapter

**Goal:** wrap a Modal-hosted MCP server so users do not need to
install its native dependencies locally.

### Step 1. Modal app

Build a Modal image that has the MCP server's binary on `PATH`.
Expose an HTTP function that proxies the stdio transport
(read line-delimited JSON from request body, write line-delimited JSON
in response).

### Step 2. Register in `mcp_servers.yaml`

```yaml
servers:
  myserver:
    transport: managed_modal
    app: "my-mcp-server"
    region: "us-east-1"
    enabled: true
```

### Step 3. `tools/managed_tool_gateway.py`

The gateway already speaks `managed_modal` transport — no code change
needed; it dispatches to the Modal app via the configured
`config.mcp.managed_gateway.base_url`.

## Recipe 14 — A custom personality

```yaml
# ~/.hermes/config.yaml
personalities:
  greybeard:
    system_prompt: |
      You are a senior engineer with 30 years of experience. You write
      terse, accurate, opinionated answers. You name names — preferred
      libraries, anti-patterns to avoid. You assume the reader is also
      an engineer.
    model: "anthropic:claude-opus-4-7"
    toolset: "hermes-cli"
    emoji: "🧓"
```

Use it: `/personality greybeard`.

## Recipe 15 — A profile for an isolated experiment

```bash
hermes --profile experiment-2
# inside the new profile:
> /model openrouter:moonshotai/kimi-k2
> /tools enable browser
> ...
```

Everything (sessions, skills, cron, memory) lives under
`~/.hermes-experiment-2/`. Delete `~/.hermes-experiment-2/` to scrap the
experiment without affecting the default profile.

## Recipe 16 — Running a one-shot from `cron(1)`

System cron + Hermes one-shot:

```cron
30 8 * * 1-5  /usr/local/bin/hermes -p "Generate a morning standup" --quiet > /tmp/standup.txt 2>&1
```

This bypasses the gateway entirely; if you want delivery to a chat
platform, prefer `hermes cron add` (Recipe 10) so the gateway
dispatches.

## Recipe 17 — A test that drives the full agent loop

`tests/run_agent/test_full_loop_smoke.py`:

```python
import pytest

from run_agent import AIAgent
from tests.fakes import fake_chat_completions_client


@pytest.mark.parametrize("prompt", ["What is 2+2?", "Hello!"])
def test_simple_prompt_returns_text(prompt, monkeypatch):
    fake = fake_chat_completions_client.script([
        {"role": "assistant", "content": "..."},
    ])
    monkeypatch.setattr("run_agent._build_client", lambda *a, **kw: fake)

    agent = AIAgent(
        provider="openrouter",
        model="anthropic/claude-haiku-4-5",
        api_key="sk-fake",
        max_iterations=4,
        save_trajectories=False,
        skip_memory=True,
        skip_context_files=True,
    )
    out = agent.chat(prompt)
    assert isinstance(out, str)
    assert "..." in out
```

`tests/fakes/fake_chat_completions_client.py` provides a deterministic
response source.

## Recipe 18 — Plugging a custom logging sink

`logging.yaml`:

```yaml
version: 1
disable_existing_loggers: false
formatters:
  default:
    "()": "hermes_logging.RedactingFormatter"
    fmt: "%(asctime)s %(levelname)s %(name)s %(message)s"
handlers:
  console:
    class: logging.StreamHandler
    formatter: default
    level: DEBUG
  file:
    class: logging.handlers.RotatingFileHandler
    formatter: default
    filename: /tmp/hermes.log
    maxBytes: 10485760
    backupCount: 5
loggers:
  agent:
    level: DEBUG
    handlers: [console, file]
  tools:
    level: INFO
    handlers: [file]
```

```bash
LOG_CONFIG=logging.yaml hermes
```

`hermes_logging.setup_logging()` honours `LOG_CONFIG` if present.

## Recipe 19 — Custom tool with backend awareness

A tool that runs a heavy command — only enable it when the active
backend is sandboxed:

```python
def _check() -> bool:
    backend = (config_dict() or {}).get("terminal", {}).get("backend", "local")
    return backend in {"docker", "modal", "managed_modal", "singularity"}
```

The agent will see the tool only when the user is in a sandbox. `hermes
doctor` reflects this in its output.

## Recipe 20 — Listening for SessionDB writes

A plugin that wants to live-update a dashboard:

```python
import time
from hermes_state import SessionDB

def watch():
    db = SessionDB()
    last_id = 0
    while True:
        rows = db.fetch_messages_since_id(last_id)
        for row in rows:
            push_to_websocket(row)
            last_id = row["id"]
        time.sleep(1)
```

`fetch_messages_since_id()` is a public method on `SessionDB`; it does
a single indexed query on `messages.id`.

## Recipe 21 — Building an offline trajectory pipeline

```bash
# 1. Generate raw trajectories on a small dataset.
python batch_runner.py \
  --dataset_file=data/easy.jsonl \
  --batch_size=8 \
  --provider=openrouter \
  --model="anthropic/claude-haiku-4-5" \
  --output=runs/easy_raw.jsonl \
  --output_format=hermes \
  --checkpoint_path=runs/easy.ckpt

# 2. Compress.
python trajectory_compressor.py \
  --input=runs/easy_raw.jsonl \
  --output=runs/easy_compressed.jsonl \
  --target_max_tokens=12000 \
  --summarizer_provider=openrouter \
  --summarizer_model="anthropic/claude-haiku-4-5"

# 3. Filter (e.g. by tool stats).
python -c '
import json
with open("runs/easy_compressed.jsonl") as fin, open("runs/easy_filtered.jsonl","w") as fout:
    for line in fin:
        row = json.loads(line)
        stats = row.get("tool_stats", {})
        if stats.get("terminal", {}).get("count", 0) >= 1:
            fout.write(line)
'

# 4. Upload to your training platform.
```

## Recipe 22 — Adding a new browser provider

`tools/browser_providers/myprovider.py`:

```python
from .base import BrowserProvider


class MyProvider(BrowserProvider):
    name = "myprovider"

    async def open_url(self, url: str, *, headless: bool = True) -> str:
        ...
    async def screenshot(self, url: str) -> bytes:
        ...
    async def extract(self, url: str) -> str:
        ...
```

Register in `tools/browser_providers/__init__.py`. Then `config.browser.provider:
"myprovider"` selects it.

## Recipe 23 — Subagent delegation with restricted tools

```python
from tools.delegate_tool import delegate

result = delegate(
    task="Find a Python implementation of A* search",
    enabled_toolsets=["web", "search"],   # the subagent only sees these
    disabled_tools=["terminal"],          # belt-and-braces
    max_iterations=15,
    parent_session_id=current_session_id,
)
```

The subagent runs in its own `AIAgent` instance, inheriting the
parent's credential pool. Its trajectory is a child of the parent
session in the SessionDB.

## Recipe 24 — Replacing the prompt-injection scanner

Drop-in replacement for `agent/prompt_builder.py:_scan_for_threats`.
The contract:

```python
def _scan_for_threats(text: str, source: str) -> tuple[bool, str]:
    """Return (safe, reason). If safe=False, the text is not injected."""
```

Reasonable extensions: ML-based scoring, integration with a remote
scanner, project-specific allow/deny lists.

## Recipe 25 — Custom installation channel

`hermes update` looks at the running install (pip, brew, nix, npm,
docker) and dispatches the right upgrade command. To add a new channel:

1. Edit `hermes_cli/relaunch.py:detect_install_channel()` to recognise
   the new channel (e.g. via a marker file).
2. Edit `update_command_for_channel()` to return the upgrade incantation.
3. Document in `RELEASE_*.md`.

The channel auto-detect lets you ship Hermes through unusual paths
(e.g. enterprise package mirrors) without code changes for end users.
