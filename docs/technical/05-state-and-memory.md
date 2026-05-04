# 05 — State, Memory, Skills and Context

This document covers the durable subsystems that give Hermes its
"learning loop": the SQLite state store, the memory subsystem, the
context engine, and the skill curator.

## 1. SQLite session store (`hermes_state.py`)

Hermes uses **one SQLite database** for all conversational state:
`~/.hermes/state.db` (or the profile-scoped equivalent). The schema is
defined in `hermes_state.py` and centred on `SessionDB`
(`hermes_state.py:159`).

### Schema

| Table | Purpose |
|-------|---------|
| `sessions` | One row per conversation. Columns include: `session_id` (PK), `source` (`cli`/`telegram`/…), `model`, `parent_session_id` (used to chain compressed sessions), `started_at`, `ended_at`, `message_count`, `tool_call_count`, token totals (`input`, `output`, `cache_read`, `cache_write`, `reasoning`), and a billing snapshot (provider, base_url, cost estimates). |
| `messages` | Full message history. One row per message: `id` (PK), `session_id` (FK), `role`, `content`, `tool_call_id`, `tool_calls` (JSON), `tool_name`, `timestamp`, `token_count`, `finish_reason`, `reasoning`, `codex_reasoning_items`. |
| `state_meta` | KV store (`key TEXT PK`, `value TEXT`) for small persistent flags (last-curator-run, last-update-check, …). |
| `messages_fts` | FTS5 virtual table over `messages.content` with the **Unicode61** tokeniser — fast English / romance-language search. |
| `messages_fts_trigram` | FTS5 virtual table with a **3-byte trigram** tokeniser — substring search for CJK and other languages where Unicode61 word boundaries break down. |

### Concurrency model

* SQLite is opened in **WAL mode** so multiple readers can run alongside
  one writer.
* Each `SessionDB` method opens its own cursor (no long-lived transactions
  outside batch helpers).
* Writers use a short SQLite timeout (1 s) plus an application-level
  jitter retry loop: `_WRITE_MAX_RETRIES = 15`,
  `_WRITE_RETRY_MIN_S = 0.020`, `_WRITE_RETRY_MAX_S = 0.150`
  (`hermes_state.py:167-180`). This handles the case where the gateway,
  the CLI, the cron scheduler and the curator all want to write at the
  same time.

### Why one database?

Replacing the historical mix of JSONL transcripts with one relational
store gives:

* Cross-session search via FTS5.
* Constant-time session lookup by id.
* Aggregate queries (token totals, cost per session, tool-call stats)
  that would otherwise need a full directory walk.
* A single mutex (the SQLite WAL) instead of per-file locks.

### Compressed sessions

When `ContextCompressor` decides to compress, it materialises the
summary as a *new* session whose `parent_session_id` points back to the
original. This preserves the original verbatim history (so the user can
always reconstruct it) while letting the agent run on the compact form.

## 2. Memory subsystem

### `MemoryManager` (`agent/memory_manager.py`)

The orchestrator. There is always **one** `BuiltinMemoryProvider`
(`MEMORY.md` + `USER.md`) plus optionally **one** external
`MemoryProvider` (Honcho, mem0, supermemory, holographic, …). The
single-external-provider rule is deliberate: it prevents
schema/concept drift between providers from polluting the system
prompt.

Lifecycle methods:

| Method | When |
|--------|------|
| `add_provider(provider)` | Startup — registers a provider. |
| `build_system_prompt()` | Per-turn — assembles the memory section of the system prompt from the providers. |
| `prefetch_all()` | Pre-turn — providers can fetch fresh context. |
| `queue_prefetch_all()` | Pre-turn (async) — non-blocking variant. |
| `sync_all()` | Post-turn — providers reflect on the turn and may persist memory. |
| `sanitize_context(text)` | Strips memory-context tags from outbound text (`agent/memory_manager.py:57`). |
| `StreamingContextScrubber` | Stateful scrubber for streaming chunks where a tag may straddle two chunks (`agent/memory_manager.py:65`). |

### `MemoryProvider` ABC (`agent/memory_provider.py`)

```python
class MemoryProvider(ABC):
    def is_available(self) -> bool: ...
    def initialize(self) -> None: ...
    def system_prompt_block(self) -> str: ...
    def prefetch(self) -> None: ...
    def sync_turn(self, transcript) -> None: ...
    def get_tool_schemas(self) -> list[dict]: ...        # provider-specific tools
    def handle_tool_call(self, name, args) -> str: ...
    def shutdown(self) -> None: ...
    # Optional hooks
    def on_turn_start(self) -> None: ...
    def on_session_end(self) -> None: ...
    def on_session_switch(self, old_id, new_id) -> None: ...
    def on_pre_compress(self, messages) -> None: ...
    def on_memory_write(self, key, value) -> None: ...
    def on_delegation(self, sub_session_id) -> None: ...
```

### Built-in memory provider

Stores `MEMORY.md` (agent's own notes) and `USER.md` (durable user
profile facts) under `~/.hermes/`. Exposes `memory` tools
(`tools/memory_tool.py`) for the agent to add, edit and search entries.
Periodic nudges in the system prompt encourage the agent to capture
durable facts rather than letting them disappear with the session.

### External providers (`plugins/memory/`)

Each lives in its own package with a `plugin.yaml`. Examples:

* **honcho** — Honcho AI-native memory with cross-session user
  modelling, dialectic Q&A, semantic search, persistent conclusions.
* **mem0** — mem0 graph memory.
* **supermemory** — Supermemory hosted store.
* **byterover**, **hindsight**, **holographic**, **openviking**,
  **retaindb** — alternative providers.

The `[honcho]` extra brings in `honcho-ai`. Other providers list their
own `pip_dependencies` in `plugin.yaml` and are installed on demand
(`hermes plugins install <name>`).

## 3. Context engine

### `ContextEngine` ABC (`agent/context_engine.py`)

A pluggable component that owns context-window management. Methods:

| Method | Role |
|--------|------|
| `update_from_response(response)` | Tracks tokens after each provider call. |
| `should_compress() / should_compress_preflight()` | Compression triggers (post-turn vs pre-turn). |
| `compress(messages)` | Returns the new compressed message list. |
| `has_content_to_compress()` | Quick check before invoking the LLM. |
| `get_tool_schemas() / handle_tool_call()` | Optional engine-specific tools (e.g. `lcm_grep` from the LCM engine). |
| `on_session_start() / on_session_end()` | Lifecycle hooks. |

Properties: `last_prompt_tokens`, `last_completion_tokens`,
`threshold_tokens`, `context_length`, `compression_count`.

The engine is selected via `context.engine` in `~/.hermes/config.yaml`.
Default = `compressor` (the in-tree implementation). The `lcm` engine
ships in `plugins/context_engine/`.

### `ContextCompressor` (`agent/context_compressor.py`)

The default engine. Behaviour:

* Compresses when token usage approaches the configured threshold.
* **Protects** the first system message, the first human turn, the
  first GPT turn, and the first tool result (these define the task).
* **Protects** the last N turns (configurable; default keeps the most
  recent dialogue intact).
* Summarises the **middle** of the conversation through
  `agent/auxiliary_client.call_llm()` — a cheap/fast LLM, typically
  via OpenRouter.
* Replaces pruned tool outputs with `_PRUNED_TOOL_PLACEHOLDER`
  (`agent/context_compressor.py:60`) before sending to the
  summariser, so giant blobs of stdout do not fill the summariser
  context.
* Emits a summary that follows the `SUMMARY_PREFIX` template
  (`agent/context_compressor.py:38-49`), which tracks Resolved /
  Pending questions across compactions.

### `prompt_caching.py`

Anthropic-specific. `apply_anthropic_cache_control(messages)` places up
to **four** ephemeral `cache_control` markers — by default on the
system prompt and the last three non-system messages. This produces the
"system_and_3" caching pattern that maximises cache hits without
hitting Anthropic's per-request cap.

### `prompt_builder.py`

Assembles the system prompt at the start of every session:

* Identity (`DEFAULT_AGENT_IDENTITY`) and platform-specific hints
  (`PLATFORM_HINTS`).
* Memory guidance (`MEMORY_GUIDANCE`), session-search guidance
  (`SESSION_SEARCH_GUIDANCE`), skills guidance (`SKILLS_GUIDANCE`),
  Hermes self-help guidance (`HERMES_AGENT_HELP_GUIDANCE`),
  kanban guidance (`KANBAN_GUIDANCE`).
* Project-context files: `.hermes.md`, `AGENTS.md`, `SOUL.md`,
  `.cursorrules`. These are loaded with **threat scanning** —
  `prompt_builder` looks for prompt-injection patterns, invisible
  Unicode, HTML comments and exfil-shaped URLs before injecting them.
  YAML frontmatter is stripped (`_strip_yaml_frontmatter`).
* Git-root discovery (`_find_git_root`) is used to scope which context
  files are picked up.

## 4. The skill curator

`agent/curator.py` is the autonomous side of the learning loop. It is
not a daemon — it is invoked from inside the agent at a safe boundary
when the user has been idle.

State file: `~/.hermes/skills/.curator_state` (JSON). Knobs in
`config.yaml`:

```yaml
curator:
  enabled: true
  paused: false
  interval_hours: 24
  idle_threshold_seconds: 600
  stale_skill_age_days: 30
```

Trigger logic (paraphrased):

```
if curator.enabled and not curator.paused:
    if idle_for >= idle_threshold_seconds and last_run_age >= interval_hours:
        spawn_forked_agent(curator_skill_review_prompt)
```

The forked agent reviews the agent-created skills under
`~/.hermes/skills/`, then uses skill-management tools to:

* **Pin** skills that have proven useful.
* **Archive** skills that have not been touched.
* **Consolidate** several near-duplicate skills into one.
* **Patch** skills whose instructions have been falsified by recent
  experience.

`agent/curator_backup.py` makes a backup of `~/.hermes/skills/` before
each curator run so destructive moves are reversible.

## 5. Insights

`agent/insights.py` mines the SessionDB for cross-session patterns —
recurring tasks, preferred personalities, frequently used tools — and
exposes them via `/insights [--days N]`. Insights also feed the
Honcho user-modelling provider when configured.

## 6. Title generator

`agent/title_generator.py` uses the auxiliary client to give a session a
short human-readable title once it has accumulated enough content.
Titles surface in the CLI's session browser, the dashboard and the
gateway's `/sessions` slash command.

## 7. Onboarding

`agent/onboarding.py` is the first-run flow that walks the user through
provider selection, model pick, memory setup, optional Honcho enrol,
and tool-allow-list configuration. It is called from
`hermes_cli/setup.py` (the `hermes setup` wizard) and on the very first
`hermes` invocation in a fresh `~/.hermes`.

## 8. SessionDB query patterns

`SessionDB` exposes a small, focused API. The patterns most commonly
used:

```python
# Read a session
db.get_session(session_id) -> dict
db.get_messages(session_id, limit=None, offset=0) -> list[dict]

# Write
sid = db.create_session(source="cli", model="anthropic:claude-opus-4-7",
                         parent_session_id=None)
db.persist_turn(sid, messages_added, usage_delta=None)
db.update_session_meta(sid, title=..., ended_at=..., billing=...)

# Search
db.search(query, limit=50, source=None, since=None) -> list[dict]
db.fts_search_cjk(query, limit=50) -> list[dict]   # trigram

# Maintenance
db.vacuum()
db.rebuild_fts()
```

All methods open their own short cursor and commit before returning.
Long transactions are intentionally absent — the agent loop is "many
small writes", which suits SQLite's WAL.

## 9. `state_meta` keys

Some keys you will see in `state_meta`:

| Key | Purpose |
|-----|---------|
| `last_curator_run` | ISO timestamp; read by curator inactivity check. |
| `last_update_check` | ISO timestamp; read by `hermes update`. |
| `last_setup_run` | First-run flag. |
| `pinned_skills` | JSON array of skill names auto-loaded across sessions. |
| `default_delivery_target` | `/sethome` value. |

`state_meta` is intended for *small* values; anything larger should
live in its own file or table.

## 10. The compression summary template

The exact `SUMMARY_PREFIX` (lightly paraphrased — see
`agent/context_compressor.py:38-49`):

```
SUMMARY OF EARLIER CONVERSATION
================================

Resolved questions:
- <bullet list>

Pending questions:
- <bullet list>

Files touched:
- <path: short note>

Plan / next steps:
- <bullet list>
```

Why these four sections:

1. **Resolved** — facts the agent should treat as established.
2. **Pending** — open threads that may require follow-up.
3. **Files touched** — short audit trail so the model can re-read
   any file via `read_file` if details are missing.
4. **Plan / next steps** — preserves the medium-term plan across
   compactions.

The auxiliary client is instructed to keep this scaffolding even
when there is nothing to put under a section ("- (none)" entries are
preserved). This keeps successive compactions consistent.

## 11. Curator + memory interplay

The curator pulls signals from three sources:

* `tools/skill_usage.py` — usage tracking for each skill.
* `agent/insights.py` — cross-session patterns.
* The active memory provider's transcript — the curator may decide
  that a long-running pattern in memory should be canonicalised as a
  skill.

Curator output therefore can affect memory: a successful
"consolidate" may produce a skill that supersedes a free-text memory
entry, in which case the curator can ask the memory provider to
remove the obsolete entry (`MemoryProvider.on_memory_write` hook).

## 12. Session search internals

The `session_search` tool uses both FTS5 tables:

* The default tokeniser (Unicode61) handles word-boundary searches
  for English / Romance / Cyrillic / Greek.
* The trigram tokeniser handles substring searches for CJK and other
  scripts where Unicode61 word boundaries are too coarse.

Per-query routing:

```python
if any(ord(c) > 0x2E80 for c in query):   # CJK / Hangul
    use messages_fts_trigram
else:
    use messages_fts
```

Both tables are kept in sync via SQLite triggers on `messages`
(insert / update / delete). Rebuilding either table is `db.rebuild_fts()`.

## 13. Token accounting

Per-session totals live on the `sessions` row. Each `persist_turn`
update also folds in:

* `input_tokens`
* `output_tokens`
* `cache_read_tokens`
* `cache_write_tokens`
* `reasoning_tokens`

These are fed by the transport's `extract_cache_stats()` method —
adapters that don't report cache stats simply leave the cache columns
at 0.

`/usage` reads these aggregates plus `agent/usage_pricing.py` to show
a per-session cost (status: actual / estimated / included / unknown).

