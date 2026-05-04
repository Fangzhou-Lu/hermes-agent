# 04 — Messaging Gateway and Platforms

The gateway is what turns Hermes from a CLI tool into a deployed agent
that lives on Telegram, Discord, Slack, WhatsApp, Signal, Matrix,
Email, SMS, Home Assistant, DingTalk, WeCom, Weixin, Feishu, QQ Bot,
BlueBubbles or any HTTP webhook. A single `hermes gateway` process can
host many platforms at once.

## 1. Process model

`hermes gateway start` launches `gateway.run.GatewayRunner`. The runner
loads `~/.hermes/config.yaml` (`gateway.config.GatewayConfig`),
discovers enabled platforms via `gateway.platform_registry`, instantiates
one `BasePlatformAdapter` per platform, and runs all of their event
loops on the same asyncio event loop.

Per-platform inbound messages are normalised into a `MessageEvent` and
funneled through `GatewayRunner._handle_message()`, which:

1. Resolves the **session** (one per `(platform, chat_id)` pair) via
   `gateway.session.SessionSource`.
2. Walks **plugin hooks** (`before_message`, `after_dispatch`).
3. Intercepts **slash commands** (`/reset`, `/new`, `/status`,
   `/sethome`, `/model`, `/personality`, `/<skill>`, …).
4. Calls into an **`AIAgent`** instance from a per-session LRU cache
   (default capacity 128, idle TTL 1 hour). New sessions instantiate a
   fresh agent; resumed sessions reuse the cached one.
5. Routes the assistant response back through `gateway.delivery.DeliveryRouter`
   to the originating platform — or to one or more explicit targets
   (`platform:chat_id`).

The cron scheduler (`cron.scheduler.tick()`) runs on a 60 s interval
inside the gateway's background thread. Cron output is delivered via
the same `DeliveryRouter`, so a cron job can post its result to any
platform regardless of where it was created.

## 2. Sessions

| File | Purpose |
|------|---------|
| `gateway/session.py` | `SessionSource` (chat → session id), persistence to `~/.hermes/conversations/{session_id}.jsonl`, reset-policy evaluation. |
| `gateway/session_context.py` | Dynamic context injection into the system prompt (per-session memory, persona, channel description). |
| `gateway/config.py` | `Platform` enum, `SessionResetPolicy` (time-based, message-count-based, platform-specific). |

A session is auto-reset based on its `SessionResetPolicy`:

* **Time-based** — reset after N minutes of inactivity.
* **Count-based** — reset after N messages.
* **Manual** — only `/reset` clears the session.

The persistent transcript on disk is one JSONL event per line; the
canonical transcript also lives in `SessionDB` so that
`session_search_tool` and the dashboard can index it.

### LRU agent cache

`gateway/run.py` keeps a bounded LRU cache of `AIAgent` instances keyed
by session id. The bounds (capacity = 128, idle TTL = 1 h) keep
memory in check on multi-tenant gateways while letting active
conversations stay warm.

## 3. Delivery routing

`gateway/delivery.py` implements `DeliveryRouter`. A delivery target is
written as a string:

```
origin                     # back to the platform/chat that initiated the request
local                      # write to a file on disk
telegram:123456789         # explicit platform + chat id
discord:guild/channel
slack:C0123456789
matrix:!room:server.tld
email:user@example.com
```

The CLI's `/sethome <target>` slash command sets a default delivery
target for a session, which cron jobs and async tools can then use.

## 4. Plugin hooks

`gateway/hooks.py` defines the hook interface. A plugin's
`plugin.yaml` lists which hooks it wants to register
(`on_session_end`, `before_message`, `after_dispatch`, …) and the
corresponding callable in its `__init__.py` is invoked at the right
moment.

`gateway/builtin_hooks/` is reserved for hooks that ship in the package
itself; nothing is registered there by default.

## 5. Platform adapters

### Base class (`gateway/platforms/base.py`)

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self): ...
    @abstractmethod
    async def disconnect(self): ...
    @abstractmethod
    async def send_message(self, chat_id, text, **kwargs): ...
    @abstractmethod
    async def send_file(self, chat_id, path, **kwargs): ...
    async def send_audio(self, chat_id, path, **kwargs): ...
```

Plus shared helpers: `MessageType` enum (`TEXT`, `IMAGE`, `AUDIO`,
`FILE`, `COMMAND`), `utf16_len()` (Telegram counts entity offsets in
UTF-16 code units), `_prefix_within_utf16_limit()` (safe truncation),
`should_send_media_as_audio()` (per-platform audio routing).

### Shipped platforms

| Adapter | File |
|---------|------|
| Telegram | `gateway/platforms/telegram.py` (+ `telegram_network.py`) |
| Discord | `gateway/platforms/discord.py` |
| Slack | `gateway/platforms/slack.py` |
| Matrix | `gateway/platforms/matrix.py` |
| Mattermost | `gateway/platforms/mattermost.py` |
| Signal | `gateway/platforms/signal.py` (+ `signal_rate_limit.py`) |
| WhatsApp | `gateway/platforms/whatsapp.py` |
| BlueBubbles (iMessage) | `gateway/platforms/bluebubbles.py` |
| Email (IMAP/SMTP) | `gateway/platforms/email.py` |
| SMS | `gateway/platforms/sms.py` |
| Home Assistant | `gateway/platforms/homeassistant.py` |
| Webhook (generic) | `gateway/platforms/webhook.py` |
| HTTP API server | `gateway/platforms/api_server.py` (with TLS + auth) |
| Feishu (Lark) | `gateway/platforms/feishu.py` |
| Feishu Comments | `gateway/platforms/feishu_comment.py` (+ `feishu_comment_rules.py`) |
| WeCom (WeChat Work) | `gateway/platforms/wecom.py` (+ `wecom_callback.py`, `wecom_crypto.py`) |
| Weixin (WeChat) | `gateway/platforms/weixin.py` |
| DingTalk | `gateway/platforms/dingtalk.py` |
| QQ Bot | `gateway/platforms/qqbot/` (subpackage) |
| Yuanbao (Alipay Mini Program) | `gateway/platforms/yuanbao.py` (+ `yuanbao_media.py`, `yuanbao_proto.py`, `yuanbao_sticker.py`) |

`gateway/platforms/_http_client_limits.py` centralises connection-limit
tuning that several HTTP-based adapters share.

`gateway/platforms/helpers.py` carries the cross-platform helpers used
by adapters (entity-aware text truncation, MIME guessing, sticker
transcoding, etc.).

### Adding a platform

The repo includes `gateway/platforms/ADDING_A_PLATFORM.md`, which is
the canonical step-by-step guide. The short version:

1. Create `gateway/platforms/<name>.py`.
2. Subclass `BasePlatformAdapter`; implement `connect`, `disconnect`,
   `send_message`, `send_file` (and `send_audio` if relevant).
3. Add entries in `gateway/config.py` (`Platform` enum + config
   schema).
4. Register the adapter in `gateway/platform_registry.py`.
5. Add tests under `tests/gateway/`.

For platforms that need to live outside the main repo (vendor-specific
SDKs, licence constraints, …) the same shape works as a plugin
(`plugins/platforms/<name>/`). See the in-tree `plugins/platforms/irc/`
and `plugins/platforms/teams/` examples.

## 6. Pairing & identity

* `gateway/pairing.py` — device pairing flows for platforms that need
  one (Signal, WhatsApp). The CLI surface is `hermes pairing`.
* `gateway/whatsapp_identity.py` — WhatsApp identity & key management.
* `gateway/sticker_cache.py` — stores transcoded stickers per platform
  in `~/.hermes/cache/stickers/`.

## 7. Mirror & status

* `gateway/mirror.py` — duplicates an inbound message to a secondary
  target (useful for audit or training capture).
* `gateway/status.py` — `/status` slash command implementation; also
  exposes a small JSON over the gateway's local socket so the CLI's
  `hermes status` can show platform liveness.
* `gateway/restart.py` — graceful shutdown + relaunch coordination
  (used by `hermes update` and by panic handlers).

## 8. Security

The gateway enforces a few security defaults:

* **Allow-list per platform** — `gateway.allowed_<platform>_users`
  in `config.yaml` lists who is permitted to interact with the bot.
* **DM pairing** — for platforms that support it, the agent will only
  bind to a chat after a pairing handshake.
* **Approval gates** — `tools/approval.py` integrates with the gateway
  so that high-risk tool calls (terminal commands, file writes,
  send-message) are confirmed via a slash interaction
  (`tools/slash_confirm.py`).
* **Container isolation** — by default a gateway-deployed agent uses
  the `local` backend on the gateway host; production deployments
  typically switch to `docker` or `managed_modal` so a shell exploit
  cannot escape to the host.

## 9. Cron delivery loop (end-to-end)

```
~/.hermes/cron/jobs.json
        ▼
cron.scheduler.tick()  ── every 60 s, file-locked
        ▼
spawns AIAgent for the job (toolset, model, prompt)
        ▼
agent runs, returns transcript
        ▼
gateway.delivery.DeliveryRouter.deliver(target, message)
        ▼
target = "origin"          → posts back to the chat that created the job
target = "local"           → writes ~/.hermes/cron/output/{job}/{ts}.md
target = "telegram:12345"  → routes via TelegramAdapter.send_message
```

This is why the gateway has to be running for cron to deliver to any
remote platform: the scheduler is hosted *inside* the gateway loop.

## 10. Per-platform configuration patterns

Every platform follows the same shape: an entry under `gateway:` (or
top-level for legacy platforms) plus secret keys in `.env`. The
patterns:

### Bot-token platforms

Telegram, Discord, Slack, Mattermost — most "bot ID + secret" platforms
follow this:

```yaml
<platform>:
  bot_token_env: "<PLATFORM>_BOT_TOKEN"
  home_channel_env: "<PLATFORM>_HOME_CHANNEL"
  allowed_users_env: "<PLATFORM>_ALLOWED_USERS"
  allowed_groups_env: "<PLATFORM>_GROUP_ALLOWED_USERS"
```

`*_env` fields name the env var rather than holding the secret —
secrets must live in `.env`.

### Account-pairing platforms

Signal, WhatsApp, BlueBubbles — the bot binds to a real user account
rather than a separate bot identity:

```yaml
signal:
  account_env: "SIGNAL_ACCOUNT"          # e.g. "+15551234567"
  http_url_env: "SIGNAL_HTTP_URL"        # e.g. "http://localhost:8080"
  allowed_users_env: "SIGNAL_ALLOWED_USERS"
```

`hermes pairing` walks through the account-binding handshake.

### OAuth platforms

DingTalk, Feishu, Vercel-hosted ones — the user runs an OAuth flow:

```bash
hermes dingtalk login
hermes feishu login        # if a CLI helper exists; otherwise hermes auth feishu
```

OAuth tokens live at `~/.hermes/auth/<platform>.json`. The platform
adapter reads the access/refresh tokens from there at startup.

### Cryptography-callback platforms

WeCom (WeChat Work) and Yuanbao (Alipay) require server-side AES-CBC
verification of incoming webhooks. Hermes ships the encrypt/decrypt
utilities (`gateway/platforms/wecom_crypto.py`,
`gateway/platforms/yuanbao_proto.py`) so the adapter can run without
relying on a vendor-specific SDK at runtime.

## 11. The session lifecycle

A session is created when:

* A new chat sends its first inbound message.
* A user runs `/new` (or `/reset`).
* A `branch`/`fork` produces a child session.
* A compression pass produces a new session row whose
  `parent_session_id` points at the original.

A session is reset when:

* The user runs `/reset`.
* `SessionResetPolicy` triggers (time-based, count-based, or
  platform-specific).

A session is "ended" but not deleted when:

* The user runs `/quit` (CLI).
* The gateway shuts down the LRU cache entry (idle TTL expiration).

A session is hard-deleted when:

* The user runs `hermes session delete <id>`.
* `sessions.retention_days` elapses with no activity (if set).

The transcript is stored in two places:

1. `SessionDB` — relational, fully searchable, retains tool-call
   metadata.
2. `~/.hermes/conversations/<session_id>.jsonl` — append-only JSONL,
   one event per line. Used as the canonical replay source if
   SessionDB is rebuilt.

## 12. Reset policies

`gateway.config.SessionResetPolicy`:

```yaml
gateway:
  reset_policy:
    kind: "inactivity"     # inactivity | message_count | manual | hybrid
    inactivity_minutes: 240
    message_count: 100
    platform_overrides:
      telegram: { kind: "inactivity", inactivity_minutes: 1440 }
      slack:    { kind: "manual" }
```

* `inactivity` — idle for N minutes → `/reset` is implicit.
* `message_count` — every N exchanges, start fresh.
* `manual` — only `/reset` clears.
* `hybrid` — whichever fires first.

Reset is independent of the LRU cache. The LRU just controls memory
on the gateway host; reset controls *conversation* boundaries.

## 13. Branching and forking

`/branch` (alias `/fork`) creates a copy of the current session up to
the current message and switches the user to it. The implementation:

1. Allocate a new `session_id` (uuid).
2. Insert a new `sessions` row with `parent_session_id =
   <current>`.
3. Copy all `messages` rows for `<current>` up to `now()` into the
   new session id (relabelled).
4. Switch `AIAgent.session_id` to the new id.
5. Write a `state_meta` entry recording the branch point so
   `/insights` can show branch lineage.

The original session is unchanged.

## 14. Mirroring

`gateway/mirror.py` lets every inbound and outbound message be
duplicated to a secondary target — useful for live audit, training
capture, or "send everything I get to my second account so I see it
on phone too" workflows.

Configuration:

```yaml
gateway:
  mirror:
    enabled: true
    target: "telegram:secondary-account"
    direction: "both"        # inbound | outbound | both
    redact: true             # apply redact() before mirroring
```

Mirrors are best-effort and never block the primary delivery path.

## 15. Status surfaces

Several places report the gateway's health:

* `hermes status` — CLI snapshot.
* `hermes platforms` — per-platform liveness.
* `/status` — slash command.
* `/api/health` — HTTP endpoint exposed by the dashboard.
* `/api/status` — full JSON status (agent count, cron counts, recent
  errors).

`gateway/status.py` produces the JSON; the other surfaces are thin
adapters over it.

## 16. Restart and drain

`/restart` triggers a graceful drain (`gateway/restart.py`):

1. New inbound messages are queued (or rejected, depending on
   `gateway.restart_drain.queue_or_reject`).
2. In-flight `AIAgent` runs are given `agent.restart_drain_timeout`
   seconds to complete.
3. Anything still running gets `tools.interrupt.InterruptException`.
4. The runner exits with code 0; `systemd` (or the container
   supervisor) restarts.

`agent.restart_drain_timeout` default is 180 seconds, calibrated for
the realistic in-flight turn (60-150 s mid-reasoning).

## 17. Notifications and footers

Two related features for long-running turns:

* **Status footer** — gateway-side, shows the current model, token
  count and elapsed time after every assistant message.
  `gateway.runtime_footer` controls the shape; `/footer` toggles per
  session.
* **Still-working notifications** — periodic "I'm still here" pings
  every `agent.gateway_notify_interval` seconds (default 180). Saves
  users from assuming the bot has died.

Both can be disabled per session and per platform.

## 18. Platform routing for outbound tools

The `send_message_*` family of tools in `tools/send_message_tool.py`
delegates to the same `DeliveryRouter` used by the gateway runner.
This means:

* Cron jobs can post to any platform.
* Tools can post to any platform.
* Skills can `/sethome` and then send to the home target without
  hardcoding a platform.

The user does not need to write platform-specific code anywhere.

## 19. Auto-continue after restart

When the gateway restarts mid-turn, the next user message gets
prepended with:

> [System note: your previous turn was interrupted — process the
> unfinished tool result(s) first]

This nudges the model to pick up where it left off rather than
treating the new user message as a fresh request. The freshness
window is `agent.gateway_auto_continue_freshness` (default 3600 s);
older transcripts skip the auto-continue note so a "stale" gateway
restart does not revive an unrelated old task.

## 20. Toolset gating per platform

Different platforms surface different toolsets by default:

```yaml
gateway:
  toolsets:
    telegram: ["hermes-cli"]
    discord:  ["hermes-cli"]
    cron:     ["research"]      # cron is a "platform" for toolset gating
```

The `cron` "platform" is a virtual one — it just gives cron jobs a
default toolset distinct from the interactive sessions.

## 21. Locale and timezone

`hermes_time.now()` honours `config.timezone` (or the host's TZ if
empty). All gateway timestamps and cron schedules use that clock.
Tests freeze `hermes_time.now()` to UTC + a fixed instant via
`tests/conftest.py`.

## 22. Channel directory

`gateway/channel_directory.py` maps `channel_id` to:

* `display_name` — what the user sees.
* `pinned_skills` — skills auto-loaded for the channel.
* `default_personality` — overrides the global default for this
  channel.
* `delivery_target` — for `/sethome`-less defaults.

Stored in `~/.hermes/channels.json` and reloaded on `/reload`. Used
heavily by per-channel automations.

## 23. Streaming consumption

`gateway/stream_consumer.py` provides a small abstraction so that the
dashboard, the TUI and external observers can subscribe to the same
streaming events the gateway produces. Subscribers register a callback
that receives:

```python
{
    "session_id": "...",
    "kind": "message_delta" | "tool_start" | "tool_end" | ...,
    "payload": { ... },
    "ts": 1714521234.567,
}
```

The dashboard's `/api/sessions/{id}/stream` endpoint plugs into this
to relay events over a WebSocket.

## 24. Per-platform display config

`gateway/display_config.py` lets each platform tune how messages are
rendered:

```yaml
gateway:
  display:
    telegram:
      footer: false
      show_tool_progress: false
      truncate_long_messages_at: 4000
    discord:
      use_embeds: true
      tool_progress_emoji: "🔧"
```

This keeps `agent/display.py` agnostic of platform conventions; the
adapters just consume the `DisplayConfig` produced from this section.

