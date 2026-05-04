# 12 — TUI and Web Dashboard

Hermes ships two non-classic user interfaces in addition to the
prompt_toolkit-based REPL:

* **The Ink TUI** (`ui-tui/`) — a TypeScript / React terminal UI
  rendered with Ink, driven by `tui_gateway/` over JSON-RPC.
* **The web dashboard** (`web/` + `hermes_cli/web_server.py`) — a
  Vite / TypeScript SPA served by FastAPI; embeds the live TUI through
  a PTY-backed WebSocket.

This document covers both surfaces and how they interact with the
agent core.

## 1. The Ink TUI

### Activation

```
hermes --tui
HERMES_TUI=1 hermes
```

The bash wrapper `./hermes` + the `hermes` console script both honour
`--tui` and `HERMES_TUI=1`. When activated, `hermes_cli/main.py`
*execs* Node with the bundled JS (`hermes_cli/web_dist/`), replacing
the Python process. The TypeScript side then spawns
`python -m tui_gateway.server` over stdio for the actual agent
backend.

### Process model

```
hermes --tui
  └─ Node (Ink)  ──stdio JSON-RPC──  Python (tui_gateway.server)
       │                                  └─ AIAgent + tools + sessions
       └─ renders transcript, composer, prompts, activity
```

* **TypeScript owns the screen** — transcript rendering, composer
  layout, status bar, prompts, key handling.
* **Python owns the agent** — sessions, tool calls, model calls,
  slash-command logic, memory, curator.

The boundary is JSON-RPC over stdio. The TS side sends *requests*
(method calls); Python sends *events* (notifications) and request
replies. See `tui_gateway/server.py` for the method/event catalog.

### Transport

`tui_gateway/transport.py` defines `Transport` (an ABC) plus a
`StdioTransport` and a `WebSocketTransport`. `bind_transport()` sets
the active transport in a `contextvars.ContextVar` so request handlers
can route their replies without explicit plumbing.

### Methods (TS → Python)

Selected methods (`tui_gateway/server.py`):

| Method | Purpose |
|--------|---------|
| `gateway.ready` | Initial handshake; returns capabilities and skin data. |
| `prompt.submit` | Submit a user message; streams `message.delta` events. |
| `prompt.cancel` | Cancel an in-flight prompt. |
| `slash.exec` | Run a slash command in the persistent `_SlashWorker` subprocess. |
| `command.dispatch` | Fallback dispatch for unknown slash commands. |
| `complete.slash` | Slash-command completions. |
| `complete.path` | Path completions. |
| `session.list` | List sessions. |
| `session.resume` | Resume a session by id. |
| `approval.respond` | Reply to an approval request. |
| `clarify.respond` / `sudo.respond` / `secret.respond` | Reply to tool-prompted inputs. |
| `state.get` | Get a state-DB record (used for the kanban side panel). |
| `tool.cancel` | Cancel a long-running tool call. |

### Events (Python → TS)

| Event | Purpose |
|-------|---------|
| `message.delta` | Streaming text delta from the model. |
| `message.complete` | End of message. |
| `tool.start` / `tool.progress` / `tool.complete` | Tool-call lifecycle. |
| `approval.request` | High-risk command needs approval. |
| `clarify.request` / `sudo.request` / `secret.request` | Prompt the user for an input. |
| `session.changed` | Session id, title, or model changed. |
| `gateway.metrics` | Periodic metrics (token counts, cost, rate-limit reset). |
| `notification.show` | Toast-style message. |
| `error.unhandled` | Unhandled exception in the gateway. |

`tui_gateway/event_publisher.py` is the central event broker; handlers
publish events through it instead of directly writing to the
transport.

### Slash workers

`tui_gateway/slash_worker.py` runs a long-lived **subprocess** that
holds the slash-command dispatch state. The reason: many slash commands
import heavy modules (e.g. `hermes_cli/commands.py` imports
prompt_toolkit; `hermes_cli/setup.py` imports the full provider
catalogue) and we do not want to pay that cost on every keystroke. The
worker stays warm for the life of the TUI session.

Communication is JSON-RPC over a pair of pipes; the worker writes
`slash.exec` results back to the gateway, which forwards them to
TypeScript.

### Key surfaces in the Ink app

| Surface | Component | Method/Event |
|---------|-----------|--------------|
| Chat streaming | `app.tsx` + `messageLine.tsx` | `prompt.submit` / `message.delta`+`message.complete` |
| Tool activity | `thinking.tsx` | `tool.start` / `tool.progress` / `tool.complete` |
| Approvals | `prompts.tsx` | `approval.respond` ← `approval.request` |
| Clarify / sudo / secret | `prompts.tsx` + `maskedPrompt.tsx` | `clarify/sudo/secret.respond` |
| Session picker | `sessionPicker.tsx` | `session.list` / `session.resume` |
| Slash commands | local handler + fallthrough | `slash.exec` → `_SlashWorker`, `command.dispatch` |
| Completions | `useCompletion` hook | `complete.slash` / `complete.path` |
| Theming | `theme.ts` + `branding.tsx` | `gateway.ready` (skin data in payload) |

### Built-in client commands

Some slash commands are handled entirely in TypeScript and never go to
Python:

* `/help`, `/quit`, `/clear`, `/resume`, `/copy`, `/paste`, `/redraw`,
  `/snapshot` (UI parts).

They map to local Ink actions (clearing the buffer, copying to system
clipboard, opening the session picker, etc.).

### Dev workflow

```
cd ui-tui
npm install
npm run dev          # watch mode (hermes-ink + tsx --watch)
npm start            # production
npm run build        # full build (hermes-ink + tsc)
npm run type-check   # tsc --noEmit
npm run lint         # eslint
npm run fmt          # prettier
npm test             # vitest
```

The build outputs to `hermes_cli/web_dist/` so `pip install -e .`
picks up the latest bundle without an extra step.

### Layout

`ui-tui/` is a monorepo (`packages/`) plus a top-level Vite project:

```
ui-tui/
├── package.json           # root
├── package-lock.json
├── babel.compiler.config.cjs
├── eslint.config.mjs
├── tsconfig.{json,build.json}
├── vitest.config.ts
├── packages/              # internal packages (e.g. hermes-ink)
├── src/                   # the main Ink application
│   ├── entry.tsx
│   ├── app.tsx
│   ├── gatewayClient.ts
│   └── app/{components,hooks,lib}/
└── scripts/
```

`packages/hermes-ink/` is the reusable Ink-component library; `src/`
is the actual TUI application.

## 2. The web dashboard

### Activation

```
hermes dashboard           # localhost:9119 by default
```

`hermes_cli/web_server.py` starts a FastAPI app powered by uvicorn (the
`[web]` extra brings them in). The app serves:

* `/` — the SPA from `web/dist/`.
* `/api/*` — REST endpoints for sessions, skills, kanban, status.
* `/api/pty` — WebSocket for the embedded TUI.

### Authentication

Two layers:

1. **Localhost-only by default** — `dashboard.host` defaults to
   `127.0.0.1`. Binding to a non-loopback address requires
   `dashboard.insecure: true` (the user is asking for it).
2. **Ephemeral session token** — generated at start, written into the
   browser via the `/login` redirect. Subsequent requests must carry
   it as `Authorization: Bearer <token>` (REST) or `?token=` (WS, since
   browsers cannot set headers on WebSocket upgrade).

The token is rotated on every dashboard restart.

### REST endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/status` | GET | Status + version + active model. |
| `/api/sessions` | GET | Paginated session list. |
| `/api/sessions/{id}` | GET | Full transcript. |
| `/api/sessions/{id}` | DELETE | Delete session. |
| `/api/sessions/search?q=...` | GET | FTS5 search. |
| `/api/skills` | GET | List skills. |
| `/api/skills/{name}` | GET | Skill body. |
| `/api/kanban/boards` | GET / POST | Boards. |
| `/api/kanban/tasks` | GET / POST / PATCH | Tasks. |
| `/api/insights` | GET | Insights summary. |
| `/api/usage` | GET | Token counts and cost (if pricing known). |
| `/api/health` | GET | Liveness probe. |
| `/api/pty?token=...` | WS | PTY-backed embedded TUI. |
| `/api/cron` | GET / POST | Cron jobs. |
| `/api/logs?file=...` | GET | Last 1000 lines of a logfile. |

### PTY bridge

`hermes_cli/pty_bridge.py` plus the `/api/pty` WebSocket endpoint
implement the embedded terminal:

```
browser ── WebSocket ── pty_bridge ── ptyprocess ── hermes --tui (real subprocess)
```

* Uses `ptyprocess` (POSIX) — WSL2 works; native Windows does not.
* Frames are raw PTY bytes in both directions.
* Resize messages from the browser are intercepted via the framed
  escape sequence `\x1b[RESIZE:<cols>;<rows>]` and applied with
  `TIOCSWINSZ`.
* Auth uses the same ephemeral token as REST, passed as `?token=`
  because browsers cannot set `Authorization` on WS upgrade.

Frontend uses `xterm.js` with the WebGL renderer, `@xterm/addon-fit`
for container-driven resize, and `@xterm/addon-unicode11` for modern
wide-character widths.

### Front-end structure

```
web/
├── package.json
├── vite.config.ts
├── eslint.config.js
├── index.html
├── public/                  # static assets
└── src/
    ├── main.ts
    ├── pages/
    │   ├── ChatPage.tsx     # embeds the live TUI via xterm
    │   ├── SessionsPage.tsx
    │   ├── SkillsPage.tsx
    │   ├── KanbanPage.tsx
    │   ├── InsightsPage.tsx
    │   ├── UsagePage.tsx
    │   └── ...
    ├── components/
    │   ├── ChatSidebar.tsx
    │   ├── ModelPickerDialog.tsx
    │   └── ToolCall.tsx
    ├── stores/              # Zustand state stores
    ├── hooks/
    └── api/                 # REST client wrappers
```

Build outputs to `web/dist/` and is shipped via the `[web]` extra
(`hermes_cli/web_dist/` is the *Ink* bundle; the dashboard SPA is
served from `web/dist/` at runtime).

### Architecture rule: don't reimplement the chat surface

The dashboard's chat page **embeds the real Ink TUI** through the PTY
bridge — it is not a React rewrite. From `AGENTS.md`:

> Do not re-implement the primary chat experience in React. The main
> transcript, composer/input flow, and PTY-backed terminal belong to
> the embedded `hermes --tui`.

Structured React UI **around** the TUI is fine — sidebars, model
pickers, tool inspectors. Anything that would duplicate the transcript
or composer goes into Ink instead.

### Health probes

Two env vars used to be supported:

* `GATEWAY_HEALTH_URL`
* `GATEWAY_HEALTH_TIMEOUT`

These are **deprecated** in v0.12.0 (block-comment in
`hermes_cli/web_server.py`). Container deployments now use the
single-container `HERMES_DASHBOARD=1` pattern (see
`Dockerfile` / `docker-compose.yml`), where the dashboard runs as a
side-process inside the gateway container.

### Container mode

The official Docker image runs as a single container:

```
docker run -d \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 -p 9119:9119 \
  -e HERMES_DASHBOARD=1 \
  nousresearch/hermes-agent gateway run
```

When `HERMES_DASHBOARD=1` is set, the entrypoint script backgrounds
`hermes dashboard` before `exec`-ing the chosen foreground command (in
the example above, `gateway run`). Defaults:

* Host: `0.0.0.0` (override with `HERMES_DASHBOARD_HOST`).
* Port: `9119`.
* Auto-adds `--insecure` when binding to non-localhost (matches the
  dashboard's own safety gate).
* Output prefixed with `[dashboard]` via `stdbuf` + `sed -u`.

If the dashboard process dies, it stays down until the container
restarts (no in-container supervision).

## 3. Tying it all together

```
            ┌─────────────────────────┐
            │     Browser (xterm)     │
            └──────────┬──────────────┘
                       │ WebSocket (PTY)
                       ▼
            ┌─────────────────────────┐
            │ FastAPI dashboard server│  hermes_cli/web_server.py
            └──────────┬──────────────┘
                       │ spawns
                       ▼
            ┌─────────────────────────┐
            │   hermes --tui          │  Node (Ink)
            └──────────┬──────────────┘
                       │ stdio JSON-RPC
                       ▼
            ┌─────────────────────────┐
            │ tui_gateway.server      │  Python
            └──────────┬──────────────┘
                       │
                       ▼
            ┌─────────────────────────┐
            │   AIAgent + tools       │  run_agent.py + tools/...
            └─────────────────────────┘
```

Each layer has a strict responsibility, and the JSON-RPC boundaries
make it easy to swap surfaces (a third-party Ink replacement, a
non-browser dashboard, etc.) without touching the agent core.

## 4. Practical notes

* **Latency** — the cold start of the Ink TUI is dominated by Node's
  startup; `npm start` keeps it under 250ms. The PTY-bridge adds
  ~1 frame of latency per direction.
* **Locale** — the TUI relies on the controlling terminal honouring
  UTF-8. `LANG=C` will break box drawing; the install script sets
  `LANG=C.UTF-8` if the user's locale is missing.
* **Resize loop** — terminal resize events propagate as
  `\x1b[RESIZE:...]` over the PTY; both Ink and any child shells in
  the dashboard pick them up.
* **Crash recovery** — `tui_gateway/server.py` installs panic hooks
  that write a crash report to `~/.hermes/logs/tui-crash-<ts>.log`.
  The TUI shows a one-line "gateway died — see crash log" message and
  exits gracefully.
