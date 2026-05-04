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
