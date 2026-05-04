# 13 — Security Model

This document describes Hermes Agent's security posture: the threat
model, the layers of defence, the user-visible controls, and the
implementation primitives that contributors need to know about.

The official policy is in [SECURITY.md](../../SECURITY.md). This file
is the engineering counterpart.

## 1. Threat model

The primary actors and risks:

| Actor | Risk |
|-------|------|
| **The model** | Could be jailbroken into running destructive shell commands, exfiltrating secrets, or deleting user data. |
| **The user** (deliberate) | Can run anything they want; we do not add friction to legitimate use. |
| **The user** (accidental) | Could approve a destructive command without understanding it. |
| **A skill author** (third-party) | Could ship a skill containing prompt injection or a curl-pipe-bash invocation. |
| **An MCP server** (third-party) | Could attempt to manipulate the agent via tool descriptions or tool outputs. |
| **A messaging-platform user** (gateway-only) | Could try to talk to the bot when not authorised. |
| **A compromised credential** | Could leak tokens, pollute training data, or run unauthorised provider calls. |

In short: **the model is untrusted**, **third-party skills/MCP are
untrusted**, **non-paired users are untrusted**, and **only the
local human user is trusted** (subject to the approval system).

## 2. Defence in depth

Hermes layers multiple controls; each is best-effort but together they
materially raise the bar. Layers, in order from outermost to
innermost:

1. **Allow-listing** — gateway platforms, command allow-list, URL
   allow-list, MCP server allow-list.
2. **Approval gates** — high-risk actions require explicit human
   confirmation.
3. **Path security** — file tools refuse to escape sandbox roots.
4. **Container isolation** — terminal backends (`docker`, `modal`,
   `ssh`) keep the agent off the host.
5. **Prompt-injection scanning** — context files and Hub-installed
   skills are scanned before they reach the model.
6. **Output redaction** — secrets are stripped from logs and from MCP
   error surfaces.
7. **Atomic writes** — persistent state cannot be corrupted by
   half-applied changes.

## 3. Approval system (`tools/approval.py`)

The approval module gates high-risk tool calls. A call is "high risk"
when it matches one or more rules:

* Pattern-based: e.g. `rm -rf`, `git push --force`, anything starting
  with `sudo`, `chmod -R`, `chown`, `mkfs`, `dd`, `apt-get`, …
* Tool-based: `terminal`, `file_write`, `file_patch` outside the
  sandbox root, `send_message_*`, `delegate` with broader-than-parent
  toolset.
* User-defined: `config.command_allowlist` rules in
  `~/.hermes/config.yaml`.

When a high-risk call fires, the user sees an approval prompt:

```
⚠ Hermes wants to run:
    rm -rf node_modules

Approve? [y]es / [a]lways for this command / [s]ession / [d]eny
```

* `y` — once.
* `a` — write a rule to `config.command_allowlist`.
* `s` — approve for the rest of this session.
* `d` — deny; the tool returns `{"error": "denied by user"}` and the
  agent must adapt.

In gateway mode (Telegram, Discord, …), approval is solicited via the
slash interaction (`tools/slash_confirm.py`). The user types
`/approve` or `/deny` in the chat.

In ACP mode, approval is forwarded to the editor via
`session/request_permission` (see [11-mcp-and-acp.md](11-mcp-and-acp.md)).

### `yolo` mode

`/yolo` (or `config.approvals.yolo: true`) disables the approval
prompts. **Only meaningful in a sandboxed environment.** The CLI
prints a red banner whenever yolo is active.

### Allow-list expression

`config.command_allowlist` accepts entries like:

```yaml
command_allowlist:
  - "git status"             # exact match
  - "git diff *"             # glob (one segment)
  - "/^npm (run|test|ci)$/"  # regex
  - "yarn"                   # any yarn command
```

Entries are applied in order; first match wins. The matcher lives in
`tools/approval.py:command_matches_allowlist()`.

## 4. Path security (`tools/path_security.py`)

The file tools refuse to read or write paths outside the
"sandbox root". The sandbox root is determined per session:

* **Local backend** — the launch directory (or the
  user-configured `terminal.cwd`).
* **Docker / Modal / Singularity** — the container's working dir.
* **SSH** — the remote home directory (or `terminal.cwd`).

Refusal is hard: even if the model produces an absolute path that
escapes the root (e.g. `/etc/passwd`, `/home/user/.ssh/id_rsa`), the
tool returns an error and does not touch the path.

`tools/path_security.is_within(root, path)` is the canonical helper.
It uses `os.path.realpath()` so symlink-out-of-sandbox tricks are
caught.

### Exceptions

A few paths are *always* readable regardless of the sandbox root:

* `~/.hermes/skills/` — read-only, so the agent can `read_file` a
  skill's `references/`.
* `~/.hermes/SOUL.md`, `~/.hermes/MEMORY.md`, `~/.hermes/USER.md` —
  managed by memory tools; not readable by `read_file`.

Writes are *never* permitted outside the sandbox root.

## 5. URL safety (`tools/url_safety.py`)

URLs that the agent fetches go through a small allow-list /
deny-list pipeline:

* **Block list** — `localhost`, `127.0.0.0/8`, `169.254.169.254`
  (cloud metadata), `192.168.0.0/16` (private IPs), `10.0.0.0/8`,
  `172.16.0.0/12`, `*.internal`, custom user blocks.
* **Allow list** — explicit `config.security.url_allowlist` entries
  override the block list.
* **Per-tool overrides** — `web_extract`, `browser`, `web_search` can
  override per-call (e.g. `web_extract(url="http://localhost:8080",
  allow_local=True)`) but only when the agent is running in a
  sandboxed terminal backend.

Cloud-metadata URLs are blocked unconditionally, even with
`allow_local=True`. There is no flag to disable that block.

## 6. Website policy (`tools/website_policy.py`)

A higher-level "what categories of website can the agent visit" filter,
configured under `config.security.web_categories`. Default is
permissive (`*: allow`); deployments that ship Hermes to end users
typically restrict it to a curated allow-list.

## 7. Command-injection guards

Every shell-tool path quotes user input via the relevant backend's
helpers:

* `tools/terminal_tool.py` uses the backend's `execute(cmd_list)`
  variant where possible (no shell parsing).
* When a shell *is* needed, arguments are passed through `shlex.quote`.

There is no `shell=True` with f-string interpolation anywhere in the
codebase. PR reviewers reject any new instance of that pattern.

## 8. Prompt-injection scanning

Two scanners:

* **Context-file scanner** (`agent/prompt_builder.py:36-47`) — runs
  on `.hermes.md`, `AGENTS.md`, `SOUL.md`, `.cursorrules` before
  injecting them into the system prompt. Looks for invisible Unicode,
  HTML comments containing prompt-injection trigger phrases, and
  exfil-shaped URLs.

* **Skill scanner** (`tools/skills_guard.py`) — runs on every
  Hub-installed skill. Same heuristics, plus shell-command sanity
  checks (no `curl | bash`, no fork bombs, no `~/.ssh` access).

Failing scans yield a logged warning; the offending content is
*not* injected into the prompt or installed.

### Tirith integration

`tools/tirith_security.py` integrates with Nous's Tirith security
scanner when configured. Tirith does deeper analysis (taint tracking
across multiple files) and is invoked on demand via
`/skills install --tirith`.

### OSV check

`tools/osv_check.py` queries the OSV (Open Source Vulnerabilities)
database before installing a skill that declares Python-package
prerequisites. Vulnerable packages produce a warning the user must
acknowledge.

## 9. Secret redaction

`agent/redact.py` is the canonical redaction helper. It replaces:

* Anything matching `sk-[a-z0-9]{32,}` (Anthropic / OpenAI keys).
* `xoxb-`, `xoxp-`, `xoxa-` (Slack tokens).
* `gh[ps]_[A-Za-z0-9]{36,}` (GitHub tokens).
* `AKIA[0-9A-Z]{16}` (AWS access keys).
* `Bearer <token>` headers in HTTP capture.
* Any value that matches an `OPTIONAL_ENV_VARS` key marked `password:
  True`.

Usage:

```python
from agent.redact import redact
logger.info("Outbound request: %s", redact(request_dict))
```

`hermes_logging.RedactingFormatter` applies `redact()` automatically to
every log record so accidentally `print`-ing a request still ends up
redacted on disk.

## 10. Atomic writes

State files (`config.yaml`, `state.db` companions, `MEMORY.md`,
`SOUL.md`, `auth.json`, …) are written atomically via:

* `utils.atomic_replace(src, dst)` — preserves symlinks (issue #16743).
* `utils.atomic_json_write(path, obj)` — temp file + fsync + os.replace.
* `utils.atomic_yaml_write(path, obj)` — yaml flavour of the same.

`SessionDB` is SQLite, which already gives us atomicity (transaction +
WAL). The companion files (`*.db-wal`, `*.db-shm`) are managed by
SQLite; do not move them by hand.

## 11. Container isolation

The recommended way to run Hermes in production (or with untrusted
inputs) is to use one of the sandboxed terminal backends:

| Backend | Isolation |
|---------|-----------|
| `docker` | Process namespace + filesystem namespace; can be locked down further with `--read-only`, seccomp profiles. |
| `modal` | Modal sandbox: ephemeral container per execution. |
| `managed_modal` | Same, but the sandbox is owned by a trusted gateway. |
| `singularity` | HPC-style container (rootless, immutable). |
| `daytona` | Daytona development environment. |
| `ssh` | A dedicated VM. |

Even with isolation, the agent can still:

* Send messages via the gateway (gated by per-platform allow-lists).
* Make HTTP requests to allowed URLs.
* Spend tokens on the configured providers.

Treat sandbox isolation as a *blast-radius reducer*, not as a
"can't escape" guarantee.

## 12. Auth & credential handling

* Secrets are loaded from `~/.hermes/.env` only. They are not stored
  in `config.yaml` (which is meant to be shareable). The setup wizard
  refuses to write secrets into `config.yaml`.
* Credentials in memory live in the `CredentialPool` and are never
  serialised to disk in cleartext. OAuth tokens are stored as JSON in
  `~/.hermes/auth/`; on supported platforms (macOS Keychain, Windows
  Credential Manager, Linux Secret Service) Hermes will use the OS
  store instead.
* Credential rotation: a credential that hits a 401 / 403 is moved to
  `STATUS_EXHAUSTED` for `EXHAUSTED_TTL_DEFAULT_SECONDS = 3600s`
  before being retried.

## 13. Gateway-specific controls

Per-platform allow-lists live in `~/.hermes/.env`:

```
TELEGRAM_ALLOWED_USERS=123456789,987654321
DISCORD_ALLOWED_USERS=11111111111,22222222222
SIGNAL_ALLOWED_USERS=+15551234567
```

A user not in the list is silently ignored. Platforms that support
group chats also have `<PLATFORM>_GROUP_ALLOWED_USERS` and
`<PLATFORM>_ALLOW_ALL_USERS` (for opening to a public group).

DM pairing (Signal, WhatsApp) requires an explicit handshake before
the bot binds to a chat. This stops a leaked phone number from being
used to message Hermes by random parties.

## 14. Privacy

`config.privacy` controls what telemetry leaves the host:

```yaml
privacy:
  send_diagnostics: false      # `hermes debug`
  redact_logs: true            # apply RedactingFormatter
  share_anonymous_metrics: false
```

By default Hermes sends no telemetry. `hermes debug` only uploads when
the user explicitly invokes it.

## 15. SOUL.md and MEMORY.md handling

Personality and memory files live under `~/.hermes/`:

* `SOUL.md` — persona; included in the system prompt.
* `MEMORY.md` — agent memory.
* `USER.md` — durable user profile facts.

These files are loaded *with* the prompt-injection scanner. If a third
party (or the model itself) has scribbled an injection into the file,
the scanner will refuse to inject it and log a warning.

### Manual edit hook

`hermes_cli/memory_setup.py` exposes a `hermes memory edit` command
that opens `MEMORY.md` in `$EDITOR`. Edits go through
`utils.atomic_yaml_write()`, so a partial save cannot corrupt the
file.

## 16. Approval-state persistence

Session-scoped approvals live in memory only. "Always" approvals are
appended to `config.command_allowlist` so they survive restarts. The
"always" choice can be revoked by editing the config file.

## 17. Common security mistakes contributors make

* **Bypassing path_security in tools** — never accept an absolute
  path from the model and pass it directly to `open()`. Always go
  through `path_security.is_within()` first.
* **Logging the request body** — outgoing API request bodies often
  contain the system prompt, which often contains secrets. Use
  `redact()` first.
* **Adding a "skip approval" flag to a tool** — the approval system
  is a security boundary. New tools should default to *requiring*
  approval for destructive operations and let the user opt out via
  the allow-list.
* **String-interpolating shell commands** — always use
  `shlex.quote()` or pass argv as a list.
* **Hard-coding `~/.hermes`** — always `get_hermes_home()`. The
  difference matters when users run with a profile.
* **Trusting MCP server output** — surface it to the model as a tool
  result (which is treated with appropriate scepticism), but never
  paste it into the system prompt.

## 18. Vulnerability reports

Reports go to the address in [SECURITY.md](../../SECURITY.md). We aim
to acknowledge within 48 hours and ship a fix within 30 days for any
high-severity issue.

The CI pipeline runs `tirith` on every PR; failing scans block merge.

## 19. Where to look when…

| Scenario | Where |
|----------|-------|
| Investigating an unauthorised platform user | `gateway/platforms/<name>.py` allow-list check |
| Tightening an approval rule | `tools/approval.py:command_matches_allowlist` |
| Debugging a failed url safety check | `tools/url_safety.py:check_url` |
| Adding a redaction pattern | `agent/redact.py:_PATTERNS` |
| Auditing a Hub-installed skill | `tools/skills_guard.py:scan` plus `audit.log` |
| Investigating a credential leak | `agent/credential_pool.py:STATUS_EXHAUSTED` rows + `~/.hermes/logs/errors.log` |
