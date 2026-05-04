# 11 — MCP and ACP

Hermes Agent integrates with the two emerging open agent protocols:

* **MCP (Model Context Protocol)** — Hermes can act as both an *MCP
  client* (consuming third-party MCP servers as tools) and an *MCP
  server* (exposing its session DB and conversation tools to other MCP
  clients like Claude Code, Cursor, Codex).
* **ACP (Agent Client Protocol)** — Hermes acts as an *ACP server* so
  IDEs and other agent-aware editors (VS Code, Zed, JetBrains) can
  drive a Hermes session over a JSON-RPC protocol.

This document covers both.

## 1. MCP — Hermes as a client

### Configuration

MCP servers are declared in `~/.hermes/mcp_servers.yaml`. Each entry is
addressable by a short name; the schema:

```yaml
servers:
  github:
    transport: stdio
    command: ["npx", "-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_TOKEN: ${GITHUB_TOKEN}
    enabled: true
    autoreconnect: true

  fly:
    transport: http
    url: https://api.fly.io/mcp
    auth:
      kind: oauth
      provider: fly
    enabled: true

  filesystem:
    transport: stdio
    command: ["mcp-server-filesystem", "/home/user/projects"]
    env: {}
    enabled: false
```

Fields:

* `transport` — `stdio` or `http`.
* `command` (stdio only) — argv to spawn.
* `env` (stdio only) — env vars to inject; supports `${VAR}` expansion.
* `url` (http only) — base URL.
* `auth` (http only) — authentication block (`oauth`, `bearer`, etc.).
* `enabled` — whether the server is active for new sessions.
* `autoreconnect` — restart on stdio EOF / HTTP disconnect.

### Lifecycle

`tools/mcp_tool.py` is the integration point. On agent startup
(`AIAgent.__init__`):

1. Load `mcp_servers.yaml`.
2. For each enabled server:
   * Spawn the stdio child or open the HTTP connection.
   * Run the MCP `initialize` handshake.
   * List tools (`tools/list`), prompts (`prompts/list`), resources
     (`resources/list`).
   * Register each tool with `tools.registry.register(toolset="mcp-<name>")`.
3. Tools are now available to the agent loop just like built-in tools.

### Reconnection

`autoreconnect: true` (default) means stdio EOF or HTTP disconnect
triggers an exponential backoff reconnect (1s, 2s, 4s, 8s, capped at
60s). Tools registered before disconnect remain registered; calls made
during the reconnect window are buffered (up to 16) and replayed.

### OAuth

`tools/mcp_oauth.py` and `tools/mcp_oauth_manager.py` implement the
MCP OAuth-2.1 / Authorization Server Discovery flow. Tokens cache at
`~/.hermes/auth/mcp/<server>.json`. The user is prompted on first
connect; `hermes mcp login <server>` re-runs the flow.

### Sampling (server-initiated LLM calls)

MCP allows servers to ask the client to run an LLM call ("sampling").
Hermes' MCP client supports this: a `sampling/createMessage` request
from the server is routed to the agent's auxiliary client (cheap/fast
model) and the response is returned. Sampling is gated by
`config.mcp.allow_sampling` (default `true`); set to `false` to make
all sampling requests fail with method-not-found.

### Credential stripping

If an MCP tool errors, the error message may contain credentials the
server received via `env`. Hermes' MCP client runs the error through
`agent/redact.py` before surfacing it to the model — never echo
raw MCP error text to the user.

### Managed MCP gateway

`tools/managed_tool_gateway.py` is the alternative to direct stdio /
HTTP — a gateway-owned Modal sandbox runs the MCP server and Hermes
talks to it via an HTTP bridge. Useful when:

* The MCP server has heavy native dependencies the user does not want
  to install locally.
* Hermes is running in a managed cloud deployment without local
  process spawning.

The gateway URL is configured under
`config.mcp.managed_gateway.base_url`.

### `/reload-mcp`

Slash command that triggers a re-read of `mcp_servers.yaml` and
reconnects any changed servers without restarting Hermes. Useful when
adding a new server mid-session.

### Failure modes

| Failure | Behaviour |
|---------|-----------|
| stdio child exits with non-zero | logged to `errors.log`, autoreconnect (if enabled) |
| HTTP 401 / 403 | refresh OAuth token, retry once; on second failure mark server unauth |
| schema mismatch | tool is rejected at registration; agent never sees it |
| timeout (default 30s) | tool call returns `{"error": "timeout"}` |
| oversized response (>1MB) | tool call truncated and the truncation noted |

## 2. MCP — Hermes as a server

`mcp_serve.py` (top-level, ~30 KB) is a stdio MCP server that exposes
Hermes' session database and conversation tools to other MCP clients.

### Run

```
hermes mcp-serve              # convenience wrapper
python mcp_serve.py           # direct invocation
```

The process speaks JSON-RPC over stdio and is intended to be spawned by
a host MCP client (Claude Code, Cursor, Codex, VS Code MCP extension).

### Capabilities

| Tool | Purpose |
|------|---------|
| `hermes_session_list` | List sessions, paginated, filtered by source / model / time range. |
| `hermes_session_view` | Return a session's full transcript (ordered messages). |
| `hermes_session_search` | FTS5-backed full-text search over messages. |
| `hermes_session_delete` | Remove a session. |
| `hermes_session_export` | Export a session to Markdown / JSONL. |
| `hermes_skill_list` | List installed skills. |
| `hermes_skill_view` | Return a skill's body. |
| `hermes_kanban_view` | Return Kanban boards / tasks. |
| `hermes_memory_view` | Return memory entries. |

### Resources

`mcp_serve.py` advertises `mcp://hermes/sessions/<id>` resources so
clients can subscribe to a session and receive new messages as they
land.

### Auth

The stdio server inherits the launching user's `~/.hermes/`. There is
no extra auth layer — anyone who can spawn the process has full read
access to the session DB.

### Implementation notes

* Lazy `FastMCP` import (`mcp_serve.py:49-55`) guarded by
  `_MCP_SERVER_AVAILABLE` so the file is importable without the `mcp`
  extra installed (the import only triggers on actual `serve`).
* `_get_session_db()` (`mcp_serve.py:71-78`) opens `SessionDB` lazily.
* `_extract_message_content()` / `_extract_attachments()` (lines
  118-150) normalise message rows into the MCP message shape (string
  content + attachment blobs).
* The channels-directory loader (`mcp_serve.py:98-116`) reads
  `gateway/channel_directory.py` data so search results can be
  scoped by platform/channel.

### Sampling support

When clients ask Hermes' MCP server to do sampling, the request is
routed through Hermes' auxiliary client (cheap/fast model). This keeps
Hermes' session-as-context capability aligned with the MCP spec.

## 3. ACP — Hermes as an ACP server

ACP (Agent Client Protocol) is the protocol used by editor-side agents
(VS Code chat panels, Zed agents, JetBrains AI Assistant). Hermes ships
a compliant ACP server so any ACP-aware editor can drive a Hermes
session.

### Run

```
hermes acp                # via the dispatcher
hermes-acp                # direct console-script
python -m acp_adapter.entry
```

stdout is the JSON-RPC channel; everything else (logs, tracebacks)
goes to stderr.

### Files

| File | Purpose |
|------|---------|
| `acp_adapter/entry.py` | CLI entry; loads `~/.hermes/.env`; configures stderr-only logging; starts the server. |
| `acp_adapter/server.py` | The JSON-RPC server. ~1k lines. Handles ACP methods (initialize, sessions/new, sessions/prompt, sessions/cancel, …). |
| `acp_adapter/auth.py` | Auth-provider detection (API key vs OAuth) and capability advertisement. |
| `acp_adapter/events.py` | ACP event callbacks (message, thinking, step, tool progress). |
| `acp_adapter/permissions.py` | Approval gates for high-risk operations — bridged to ACP's permission requests. |
| `acp_adapter/session.py` | `SessionManager` tracks ACP sessions and their underlying `AIAgent` instances. |
| `acp_adapter/tools.py` | Tool start/complete message construction matching the ACP schema. |
| `acp_registry/agent.json` | Static manifest read by editors that scan for installed ACP agents. |

### `agent.json`

A minimal manifest:

```json
{
  "name": "Hermes Agent",
  "version": "0.12.0",
  "icon": "icon.svg",
  "command": ["hermes-acp"],
  "description": "Self-improving AI agent with built-in skills, memory, and a tool sandbox.",
  "homepage": "https://hermes-agent.nousresearch.com",
  "capabilities": {
    "session": true,
    "tools": true,
    "permissions": true
  }
}
```

Place this on the editor's discovery path (varies by editor; Zed reads
`~/.config/zed/agents/`, VS Code reads `~/.config/Code/User/acp/`).

### Sessions

`SessionManager` (`acp_adapter/session.py`) owns:

* A bounded LRU of `AIAgent` instances keyed by ACP `session_id`.
* Per-session permission state.
* Per-session running task handles (so `sessions/cancel` can interrupt).

### Permission flow

When a tool call hits `tools/approval.py`, ACP turns it into a
permission request:

```
1. Hermes sees terminal call: { "command": "rm -rf node_modules" }
2. tools/approval.py raises: ApprovalRequired(...)
3. acp_adapter/permissions.py catches, sends:
       method: session/request_permission
       params: { tool: "terminal", scope: "session"|"always"|"once", explanation: ... }
4. Editor displays a dialog.
5. Editor responds with user choice; permission is recorded; tool call resumes or is denied.
```

### Liveness probe noise filter

Some clients send periodic methods (`ping`, `health`, `healthcheck`)
as liveness probes. The ACP router responds with JSON-RPC -32601 (method
not found) which is the correct protocol response, but the supervisor
task that dispatches the request would otherwise emit a stack trace to
stderr on every probe. `_BenignProbeMethodFilter` (in
`acp_adapter/entry.py`) silences just that traceback for the benign
probe methods, leaving every other "method not found" trace visible.

### Streaming

Tool calls and assistant text are streamed to the editor via:

* `session/update` events for each delta.
* `session/tool_progress` for tool-call updates (start, output chunk,
  complete).

## 4. Comparison: MCP vs ACP

| Aspect | MCP | ACP |
|--------|-----|-----|
| Direction | Editor calls server tools | Editor drives a *session* with a remote agent |
| Granularity | Tool-level | Conversation-level |
| Hermes role | Both client (consume) and server (expose) | Server only |
| Streaming | Tool output streams | Whole conversation streams |
| Permissions | Server-controlled | Client-controlled (with explicit `request_permission`) |
| Spec | https://modelcontextprotocol.io | Internal-ish; close to LSP-style RPC |

In short: **MCP is a tool API; ACP is a chat API**. Hermes plugs into
both.

## 5. Hermes-specific MCP / ACP extensions

A handful of Hermes-specific behaviours appear in both protocols:

* **Skill invocation** — the server side advertises `hermes_skill_view`
  (MCP) and the ACP server accepts `/<skill-name>` slash commands as
  prompts.
* **Curator** — both protocols expose curator state as a read-only
  resource so external dashboards can observe.
* **Insights** — `hermes_session_search` accepts a date range that
  matches the `/insights --days N` slash command's behaviour.

These are surfaced via the `extensions` field in the relevant manifest
(MCP server's `tools/list` response, ACP server's `agent.json`).

## 6. Testing

| File | Test |
|------|------|
| `tests/test_mcp_serve.py` | The Hermes MCP server end-to-end. |
| `tests/integration/mcp/` | Integration tests that spawn a real `mcp` package. |
| `tests/acp/` | ACP server unit tests. |
| `tests/acp_adapter/` | Lower-level adapter tests. |

The MCP test suite uses `tests/fakes/fake_mcp_client.py` to drive
`mcp_serve.py` over a pipe so we do not need to spawn the actual
`@modelcontextprotocol/inspector` to test.

## 7. Known limitations

* **No MCP `progress` reporting** — outbound MCP tool calls do not yet
  emit `progress` updates back to the model, which means very long
  tool calls show as "still running" to the user without a percentage.
* **ACP sessions do not persist across editor restarts** — the editor
  is expected to send `sessions/resume` with a previously captured
  `session_id`; without it the agent state is fresh.
* **Single concurrent ACP session per agent instance** — running two
  separate ACP sessions sharing the same `~/.hermes` is supported, but
  they will compete for `state.db` writes (mitigated by the SessionDB
  jitter retry).

## 8. Adding a new MCP capability

1. Define the tool in `mcp_serve.py`'s `tools/list` response.
2. Implement the handler.
3. Cover with a test in `tests/test_mcp_serve.py`.
4. Document in this file under "Capabilities".
