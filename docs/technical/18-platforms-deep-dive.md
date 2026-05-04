# 18 — Platform Adapters: Deep Dive

This document is the per-platform reference for the gateway adapters
shipped under `gateway/platforms/`. Each section covers:

* The SDK / transport the adapter uses.
* The required env vars and config keys.
* Identity model (user id, channel id) and how Hermes maps it to
  sessions.
* Notable quirks (markdown rendering, attachments, threads, polls).
* Allow-list semantics.

For the abstract adapter contract, see [04-gateway.md](04-gateway.md).
For env-var details, see [14-configuration-reference.md](14-configuration-reference.md).

## 1. Telegram

`gateway/platforms/telegram.py` (+ `telegram_network.py`).

* **SDK:** `python-telegram-bot[webhooks]` (≥22.6, <23). The
  `messaging` extra installs it.
* **Transport:** long-polling by default; webhook mode via
  `telegram.webhook_url`.
* **Required env:** `TELEGRAM_BOT_TOKEN`.
* **Allow-list:** `TELEGRAM_ALLOWED_USERS` (comma-sep user ids),
  `TELEGRAM_GROUP_ALLOWED_USERS` (comma-sep group ids).
* **Home channel:** `TELEGRAM_HOME_CHANNEL` and `TELEGRAM_HOME_CHANNEL_NAME`.
* **Identity:** Telegram user id is a 64-bit integer; Hermes stores it
  as a string (e.g. `"12345678"`). Sessions are keyed by
  `(telegram, chat_id)` — chat id == user id for DMs, group id (negative)
  for groups.
* **Markdown:** the adapter sends with `parse_mode="MarkdownV2"`. It
  uses `_prefix_within_utf16_limit()` to truncate long messages
  safely (Telegram counts entity offsets in UTF-16 code units, not
  Python characters).
* **Auto-split:** messages over 4096 chars are split on whitespace
  boundaries.
* **Voice:** voice notes (`.ogg`) are auto-transcribed via
  `tools/transcription_tools.py` if voice mode is enabled.
* **BotCommands menu:** auto-populated from `COMMAND_REGISTRY` via
  `commands.telegram_bot_commands()` so `/<command>` works in the chat
  client autocompletion.
* **Stickers:** transcoded to PNG via `gateway/sticker_cache.py` and
  sent as photos.
* **Polls / pinned messages / forwards:** processed as their text
  representation (`/poll`, `[Pinned]`, `[Forwarded from X]`).

## 2. Discord

`gateway/platforms/discord.py`.

* **SDK:** `discord.py[voice]` (≥2.7, <3). Requires a Discord
  application + bot token + privileged intents (`MESSAGE_CONTENT`).
* **Required env:** `DISCORD_BOT_TOKEN`.
* **Allow-list:** `DISCORD_ALLOWED_USERS`, `DISCORD_GUILD_ALLOWED_USERS`.
* **Home channel:** `DISCORD_HOME_CHANNEL` (channel id),
  `DISCORD_HOME_CHANNEL_NAME` (display).
* **Identity:** Discord ids are 64-bit snowflakes. Sessions are keyed
  by channel id.
* **Markdown:** native Discord Markdown plus code blocks. The adapter
  also exposes `tools/discord_tool.py` for richer features (threads,
  reactions, embeds).
* **Voice:** voice channel join/leave is supported; voice message
  attachments work with the optional `voice` extra.
* **Reactions:** can be added to assistant replies via the discord
  tool.
* **Threads:** when the bot is mentioned in a thread, the conversation
  stays scoped to that thread.

## 3. Slack

`gateway/platforms/slack.py`.

* **SDK:** `slack-bolt` + `slack-sdk` (`messaging` or `slack` extra).
* **Required env:** `SLACK_BOT_TOKEN`, `SLACK_SIGNING_SECRET`,
  optionally `SLACK_APP_TOKEN` for socket mode.
* **Home channel:** `SLACK_HOME_CHANNEL`, `SLACK_HOME_CHANNEL_NAME`.
* **Modes:** Events API (HTTP) or Socket Mode. Auto-detected based on
  `SLACK_APP_TOKEN`.
* **Identity:** Slack uses workspace-scoped ids (`U…`) and channel
  ids (`C…`). Sessions are `(slack, channel_id)`.
* **Markdown:** Slack-flavoured markdown (`*bold*`, `~strike~`,
  triple-backtick code).
* **Slash routing:** `/hermes <subcommand>` dispatches; the subcommand
  list is auto-built from `COMMAND_REGISTRY` via
  `commands.slack_subcommand_map()`.
* **Threads:** replying to a message starts a thread; subsequent
  messages in the thread share the same session.
* **File uploads:** supported as inputs (auto-attached) and as outputs
  (the agent can `send_file`).

## 4. Mattermost

`gateway/platforms/mattermost.py`.

* **SDK:** internal HTTP client.
* **Required env:** `MATTERMOST_HOME_CHANNEL`,
  `MATTERMOST_HOME_CHANNEL_NAME`, the bot token (configured per
  deployment).
* **Reply mode:** `MATTERMOST_REPLY_MODE` controls whether replies
  thread off the inbound message or post at the channel root.
* **Identity:** Mattermost user ids are 26-char strings.

## 5. Matrix

`gateway/platforms/matrix.py`.

* **SDK:** `mautrix[encryption]` (≥0.20). The `matrix` extra brings
  it in (Linux only — `python-olm` is broken on macOS).
* **Required env:** `MATRIX_USER_ID`, `MATRIX_PASSWORD`,
  `MATRIX_SERVER`, optionally `MATRIX_DEVICE_ID`,
  `MATRIX_ENCRYPTION=true`, `MATRIX_RECOVERY_KEY`.
* **Home room:** `MATRIX_HOME_ROOM`.
* **Encryption:** end-to-end via olm. Requires
  `~/.hermes/matrix.db` (managed by `mautrix`).
* **Mention rule:** `MATRIX_REQUIRE_MENTION=true` makes the bot ignore
  messages that don't @-mention it (useful in busy public rooms).
* **Auto-thread:** `MATRIX_AUTO_THREAD=true` puts replies in threads.
* **Free-response rooms:** `MATRIX_FREE_RESPONSE_ROOMS` lists rooms
  where the bot replies without requiring a mention.

## 6. Signal

`gateway/platforms/signal.py` (+ `signal_rate_limit.py`).

* **SDK:** internal HTTP client against `signal-cli-rest-api` or
  similar.
* **Required env:** `SIGNAL_ACCOUNT` (your phone number),
  `SIGNAL_HTTP_URL` (the rest-api endpoint).
* **Allow-list:** `SIGNAL_ALLOWED_USERS`,
  `SIGNAL_GROUP_ALLOWED_USERS`.
* **Pairing:** `hermes pairing` walks through QR-pairing if the
  account has not been registered with the bridge yet.
* **Rate limits:** `signal_rate_limit.py` adds explicit pacing so
  Signal does not blacklist the account for spammy bursts.

## 7. WhatsApp

`gateway/platforms/whatsapp.py` (+ `gateway/whatsapp_identity.py`).

* **Modes:** `bridge` (via the WhatsApp Business API or whatsapp-web.js
  bridge in `scripts/whatsapp-bridge/`) or `webhook`.
* **Required env:** `WHATSAPP_MODE`, `WHATSAPP_ENABLED=true`.
* **Identity:** phone-number-based JIDs.
* **Sticker cache:** stickers are transcoded via
  `gateway/sticker_cache.py`.

## 8. BlueBubbles (iMessage)

`gateway/platforms/bluebubbles.py`.

* **SDK:** internal HTTP client against a [BlueBubbles](https://bluebubbles.app/)
  server (a Mac sidecar).
* **Required env:** `BLUEBUBBLES_SERVER_URL`,
  `BLUEBUBBLES_PASSWORD`.
* **Home channel:** `BLUEBUBBLES_HOME_CHANNEL`.
* **Note:** iMessage identity (`+1...`) is what the BlueBubbles server
  exposes; the adapter does not see Apple Account IDs directly.

## 9. Email

`gateway/platforms/email.py`.

* **SDK:** Python's `imaplib` for inbox and `smtplib` for outbound.
* **Threading:** preserves `In-Reply-To` and `References` headers so
  replies thread correctly in the user's mail client.
* **Inbox poll interval:** `gateway.email.poll_interval_seconds`
  (default 60).
* **Attachments:** `application/*`, `image/*`, `audio/*` are stored
  in `~/.hermes/cache/email/attachments/<msgid>/` and offered to the
  agent as `read_file`-able paths.
* **Reply quoting:** the adapter strips quoted history from the
  inbound message before passing to the agent (configurable).

## 10. SMS

`gateway/platforms/sms.py`.

* **SDK:** internal HTTP client against an SMS gateway (Twilio etc).
* **Required env:** the gateway-specific token + phone numbers.
* **Limits:** SMS is hard-capped at 160 characters per segment. The
  adapter splits and prefixes `(1/3) ...`.

## 11. Home Assistant

`gateway/platforms/homeassistant.py`.

* **SDK:** `aiohttp` (the `homeassistant` extra brings it).
* **Required env:** `HOMEASSISTANT_URL`, `HOMEASSISTANT_TOKEN`.
* **Surface:** Home Assistant's Conversation API — the bot answers
  questions and triggers automations. Pairs with
  `tools/homeassistant_tool.py` so the agent can call HA services.

## 12. DingTalk

`gateway/platforms/dingtalk.py`.

* **SDK:** `dingtalk-stream` + `alibabacloud-dingtalk` (the
  `dingtalk` extra).
* **Required env:** `DINGTALK_CLIENT_ID`,
  `DINGTALK_CLIENT_SECRET`.
* **Auth:** OAuth via `hermes dingtalk login`.
* **QR pairing:** `qrcode` (pulled in by the same extra) renders
  pairing QR codes.

## 13. WeCom (WeChat Work)

`gateway/platforms/wecom.py` (+ `wecom_callback.py`,
`wecom_crypto.py`).

* **Required env:** `WECOM_BOT_ID`, `WECOM_SECRET`. For callbacks:
  `WECOM_CALLBACK_CORP_ID`, `_CORP_SECRET`, `_AGENT_ID`, `_TOKEN`,
  `_ENCODING_AES_KEY`, `_HOST`, `_PORT`.
* **Crypto:** WeChat Work uses AES-CBC with PKCS#7 padding for
  callback payloads; `wecom_crypto.py` does the encrypt/decrypt.
* **Bot vs callback:** two integration modes — outbound bot
  (no inbox) and full callback (server reachable from the WeCom
  cloud).

## 14. Weixin (WeChat consumer)

`gateway/platforms/weixin.py`.

* **Required env:** `WEIXIN_ACCOUNT_ID`, `WEIXIN_TOKEN`,
  `WEIXIN_BASE_URL`, optionally `WEIXIN_CDN_BASE_URL`.
* **Allow-list:** `WEIXIN_ALLOWED_USERS`,
  `WEIXIN_GROUP_ALLOWED_USERS`, `WEIXIN_ALLOW_ALL_USERS`.
* **DM / group policy:** `WEIXIN_DM_POLICY`, `WEIXIN_GROUP_POLICY`
  (`open` / `mention` / `silent`).

## 15. Feishu (Lark)

`gateway/platforms/feishu.py` (+ `feishu_comment.py`,
`feishu_comment_rules.py`).

* **SDK:** `lark-oapi` (the `feishu` extra).
* **Required env:** `FEISHU_APP_ID`, `FEISHU_APP_SECRET`,
  `FEISHU_ENCRYPT_KEY`, `FEISHU_VERIFICATION_TOKEN`.
* **Comments delivery:** `feishu_comment.py` lets cron/agent results
  post as comments on Feishu docs (`feishu_comment_rules.py` defines
  who can reply where).
* **Pairs with:** `tools/feishu_doc_tool.py` and
  `tools/feishu_drive_tool.py` so the agent can read/write Feishu
  documents and Drive files.

## 16. Yuanbao (Alipay Mini Program)

`gateway/platforms/yuanbao.py` (+ `yuanbao_media.py`,
`yuanbao_proto.py`, `yuanbao_sticker.py`).

* **Identity:** Alipay user id (`u-...`).
* **Media:** the proprietary Yuanbao media protocol; `yuanbao_proto.py`
  serialises the wire shape.
* **Stickers:** dedicated cache + transcoder
  (`yuanbao_sticker.py`).
* **Pairs with:** `tools/yuanbao_tools.py`.

## 17. QQ Bot

`gateway/platforms/qqbot/` (subpackage).

* **Required env:** `QQ_APP_ID`, `QQ_CLIENT_SECRET`. Legacy aliases
  `QQ_HOME_CHANNEL` / `QQ_HOME_CHANNEL_NAME` still work.
* **Allow-list:** `QQ_ALLOWED_USERS`, `QQ_GROUP_ALLOWED_USERS`,
  `QQ_ALLOW_ALL_USERS`.
* **Markdown:** `QQ_MARKDOWN_SUPPORT=true` enables QQ Markdown (less
  capable than other platforms).
* **STT:** voice messages can be transcribed via `QQ_STT_API_KEY`,
  `QQ_STT_BASE_URL`, `QQ_STT_MODEL` — pluggable to any
  speech-to-text endpoint.

## 18. Webhook (generic)

`gateway/platforms/webhook.py`.

A generic adapter for any HTTP-callable platform that posts events to
a URL. Configurable via `gateway.webhook.{listen_host, listen_port,
secret}` plus per-route mappings. Use this when integrating an
in-house chat or a platform with no first-party adapter.

## 19. HTTP API server

`gateway/platforms/api_server.py`.

The "gateway as a REST API". Useful for service-to-service
integrations:

* `POST /api/sessions/{id}/prompt` — send a prompt; returns the
  assistant reply.
* `GET /api/sessions/{id}/messages` — get the transcript.
* `WS /api/sessions/{id}/stream` — stream new messages over a
  WebSocket.

Authentication: `gateway.api_server.{token, hmac_secret}`. TLS via the
embedded uvicorn config or a reverse proxy.

## 20. IRC (plugin)

`plugins/platforms/irc/` — community-quality adapter that ships in the
plugins tree rather than `gateway/platforms/`. Required env:
`IRC_SERVER`, `IRC_PORT`, `IRC_NICKNAME`, `IRC_CHANNEL`,
optional `IRC_USE_TLS`, `IRC_SERVER_PASSWORD`,
`IRC_NICKSERV_PASSWORD`. Enable with `hermes plugins enable
platforms/irc`.

## 21. Microsoft Teams (plugin)

`plugins/platforms/teams/` — community-quality adapter. Enable with
`hermes plugins enable platforms/teams`.

## 22. Common adapter contract

The five methods every adapter must implement
(`gateway/platforms/base.py`):

| Method | Purpose |
|--------|---------|
| `connect()` | Open the connection or start the long-poll loop. |
| `disconnect()` | Cancel any background tasks; close the connection. |
| `send_message(chat_id, text, **kw)` | Send a text message. |
| `send_file(chat_id, path, **kw)` | Send a file attachment. |
| `send_audio(chat_id, path, **kw)` | (Optional) Send a voice message. |

Plus the inbound side: each adapter is responsible for normalising its
platform-specific event shape into a `MessageEvent`
(`gateway/platforms/base.py`). The runner only ever sees `MessageEvent`
objects — adapters absorb the platform's quirks.

## 23. Identity normalisation

Every adapter produces `event.source` as a dict:

```python
{
    "platform": "telegram",
    "chat_id": "12345678",
    "user_id": "12345678",
    "user_name": "alice",
    "channel_name": "Direct chat",
}
```

`SessionSource.resolve(event)` turns this into a `session_id`:

```
sha1(f"{platform}:{chat_id}").hexdigest()[:16]
```

That gives every chat a stable, opaque session id without exposing the
raw user identifier in logs / dashboards.

## 24. Inbound media handling

Inbound media (images, audio, files) follows a shared path:

1. The adapter downloads the asset (with retry / timeout) into
   `~/.hermes/cache/inbox/<adapter>/<msgid>/`.
2. The adapter attaches `attachments=[{type, path, mime, size}]` to
   the `MessageEvent`.
3. `GatewayRunner` decides:
   * Image → attach to the prompt (vision-enabled provider) or
     pre-analyse via `vision_analyze` if the model is text-only.
   * Audio → transcribe via `tools/transcription_tools.py` (if voice
     is enabled) or pass the path to the agent.
   * Other file → save the path; the agent can `read_file` it.

All inbound media respects the active backend's path security.

## 25. Outbound formatting

Each adapter implements platform-specific formatting on top of a
shared "intermediate" representation:

* Markdown is normalised (e.g. `**bold**` → `*bold*` for Slack and
  Telegram MarkdownV2).
* Code blocks are preserved.
* URLs are wrapped in the platform's link syntax where needed.
* Long messages are split at safe boundaries (whitespace, sentence
  boundaries).

`gateway/platforms/helpers.py` carries the cross-platform helpers
(`utf16_len`, `split_message`, `mime_to_kind`).

## 26. Adding a platform: contractual checklist

When you add a platform, the contract is:

* [ ] Adapter class subclasses `BasePlatformAdapter`.
* [ ] Inbound events produce `MessageEvent` with all required fields.
* [ ] Outbound `send_message` accepts the intermediate Markdown format.
* [ ] Allow-list is honoured (env vars or config).
* [ ] DM-pairing handshake is implemented if the platform allows it.
* [ ] `gateway/config.py:Platform` enum has an entry.
* [ ] `OPTIONAL_ENV_VARS` lists every secret + every allow-list / home
      channel var.
* [ ] `tests/gateway/platforms/test_<name>.py` covers the inbound and
      outbound paths.
* [ ] Documentation entry in this file.

The in-tree `ADDING_A_PLATFORM.md` is the canonical PR checklist.
