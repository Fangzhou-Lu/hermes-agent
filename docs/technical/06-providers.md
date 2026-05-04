# 06 — Provider Transports, Adapters, Credentials, Billing

This document describes how Hermes talks to LLM providers. Adding a new
provider — and reasoning about which one is in use, how its credentials
rotate, and how its quota is tracked — both happen here.

## 1. Transport layer

`agent/transports/` contains the format-translation layer between
Hermes' internal OpenAI-style message format and each provider's wire
protocol.

### `ProviderTransport` ABC (`agent/transports/base.py`)

```python
class ProviderTransport(ABC):
    @property
    def api_mode(self) -> str: ...           # e.g. "anthropic_messages"

    def convert_messages(self, messages): ...      # OpenAI → provider-native
    def convert_tools(self, tools): ...            # OpenAI → provider-native
    def build_kwargs(self, messages, tools, **kw): # full API kwargs
        ...
    def normalize_response(self, response) -> NormalizedResponse: ...

    # Optional:
    def validate_response(self, response): ...
    def extract_cache_stats(self, response): ...
    def map_finish_reason(self, raw): ...
```

`NormalizedResponse` lives in `agent/transports/types.py` and is the
shape the rest of the agent (the loop, the renderer, billing) consumes.

### Shipped transports

| api_mode                  | File | Notes |
|---------------------------|------|-------|
| `anthropic_messages`      | `agent/transports/anthropic.py` | Anthropic Messages API. Pairs with `agent/anthropic_adapter.py` for thinking budgets, OAuth setup-tokens, output limits. |
| `chat_completions`        | `agent/transports/chat_completions.py` | OpenAI-compatible: OpenAI itself, OpenRouter, llama.cpp, LM Studio, vLLM, MiMo, Moonshot, MiniMax, etc. |
| `codex_responses`         | `agent/transports/codex.py` | OpenAI Responses / Codex. Pairs with `agent/codex_responses_adapter.py` for reasoning step tracking. |
| `bedrock_converse`        | `agent/transports/bedrock.py` | AWS Bedrock Converse API. Pairs with `agent/bedrock_adapter.py` for region resolution and client pooling. |

The Gemini family is implemented as adapters rather than transports
because they can be reached through both the native Gemini API and the
Google CloudCode endpoint, and both behave more like a pre-existing
SDK that we wrap (`agent/gemini_native_adapter.py`,
`agent/gemini_cloudcode_adapter.py`, `agent/gemini_schema.py`).

### Anthropic specifics (`agent/anthropic_adapter.py`)

* **Thinking budgets** (`THINKING_BUDGET` dict, `agent/anthropic_adapter.py:47-49`):
  `xhigh → 32k`, `high → 16k`, `medium → 8k`, `low → 4k`.
* **Adaptive effort mapping** (`agent/anthropic_adapter.py:56-63`):
  `xhigh`/`max` on Claude 4.7+, downgrades to `max` on 4.6.
* **Per-model output limits** (`_ANTHROPIC_OUTPUT_LIMITS`,
  `agent/anthropic_adapter.py:84-108`) — substring matching so
  date-stamped model ids inherit the right cap.
* **Auth**: API keys (`sk-ant-api*`), OAuth setup-tokens
  (`sk-ant-oat*`), Claude Code credentials (loaded from the local
  Claude Code install if present).

### Bedrock specifics (`agent/bedrock_adapter.py`)

* `convert_tools_to_converse()` (`agent/bedrock_adapter.py:397`) and
  `convert_messages_to_converse()` (line 480) — Converse format
  translation.
* `normalize_converse_stream_events()` (line 688) — callback-based
  streaming normaliser.
* Region pooling — one `boto3` client per (region, profile)
  combination.

### Codex / Responses specifics (`agent/codex_responses_adapter.py`)

* `_chat_content_to_responses_parts()` (line 47) — translates each
  message part to the Responses API "part" shape.
* `_responses_tools()` (line 205) — tool translation.
* `_extract_responses_reasoning_text()` (line 768) — pulls the
  reasoning text out of nested response items.
* Preflight validators (`_preflight_codex_input_items` line 426,
  `_preflight_codex_api_kwargs` line 604) — catch malformed payloads
  before sending.
* `_normalize_codex_response()` (line 789) — produce the shared
  `NormalizedResponse`.

### Gemini specifics

* **Tier probing** (`probe_gemini_tier()`,
  `agent/gemini_native_adapter.py:47-120`) detects free vs paid tier so
  the agent can pick the right rate-limit policy.
* **Quota errors** (`is_free_tier_quota_error()`,
  `agent/gemini_native_adapter.py:121-136`) get reclassified for the
  retry / failover layer.
* `_build_gemini_contents()` (line 276) — content translation.
* `_translate_tools_to_gemini()` (line 330) — tool translation.
* `_normalize_thinking_config()` (lines 372-387) — maps Hermes
  thinking knobs to Gemini's config fields.
* `translate_gemini_response()` (line 474) and the
  `_GeminiStreamChunk` dataclass (line 543+).

### Other adapters

* `agent/copilot_acp_client.py` — GitHub Copilot for VS Code, talks the
  Copilot Editor Protocol over a child-process pipe.
* `agent/google_oauth.py` — OAuth flow for Google services
  (Gemini API, Drive, Calendar, …).
* `agent/google_code_assist.py` — Google Code Assist (CloudCode)
  integration.
* `agent/moonshot_schema.py`, `agent/lmstudio_reasoning.py` — small
  per-provider schema/parsing fixes.

## 2. Adding a provider

The recipe (high-level):

1. Pick or create a transport class.
   * If the provider speaks an OpenAI-compatible API, **reuse**
     `chat_completions` and only set `base_url`.
   * If it has its own protocol, subclass `ProviderTransport` in a new
     file under `agent/transports/`.
2. If the provider has quirks (thinking budgets, model-specific
   limits, custom auth), add an adapter under `agent/<provider>_adapter.py`.
3. Wire it into the model catalog: `scripts/build_model_catalog.py`
   (or `hermes_cli/model_catalog.py`) so `hermes model` can offer it.
4. Add tests under `tests/agent/`.

## 3. Credentials

### `CredentialPool` (`agent/credential_pool.py`)

Hermes can hold multiple credentials per provider and rotate between
them. `PooledCredential` (lines 91-137):

```python
@dataclass
class PooledCredential:
    provider: str
    id: str                  # stable identifier
    label: str               # display name
    auth_type: str           # "oauth" | "api_key"
    priority: int
    source: str              # where it came from (env, oauth, file)
    access_token: str
    refresh_token: str | None
    last_status: str         # STATUS_OK | STATUS_EXHAUSTED
    last_error_code: str | None
    last_error_message: str | None
    last_error_reset_at: float | None
```

Pool strategies (`agent/credential_pool.py:59-63`):

| Strategy     | Behaviour |
|--------------|-----------|
| `FILL_FIRST` | Use the highest-priority credential until it errors. |
| `ROUND_ROBIN`| Rotate through the pool turn by turn. |
| `RANDOM`     | Pick uniformly. |
| `LEAST_USED` | Pick the one with the fewest recent uses. |

When a credential hits a quota / auth error, it is parked with
`last_status = STATUS_EXHAUSTED` and a cooldown
(`EXHAUSTED_TTL_DEFAULT_SECONDS = 3600`,
`agent/credential_pool.py:73`). The pool returns one of the remaining
healthy credentials; if all are exhausted, it returns the one with the
earliest `last_error_reset_at`.

Custom endpoints get a `"custom:"` pool key prefix
(`agent/credential_pool.py:79`) so user-configured endpoints do not
collide with named providers.

### `CredentialSources` (`agent/credential_sources.py`)

Discovers credentials from:

* environment variables (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `OPENROUTER_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, …),
* `~/.hermes/.env` via `python-dotenv`,
* OS keychain (where supported),
* OAuth tokens cached at `~/.hermes/auth/<provider>.json`,
* config-file entries (rare; mostly for self-hosted endpoints).

### Why a pool?

Two concrete reasons:

1. **Free-tier farming** — multiple Gemini / OpenRouter free-tier keys
   can be combined to stretch daily quotas without paying.
2. **Reliability** — if one provider has a regional outage, the pool
   rotates to the next one transparently. The agent does not see the
   error.

## 4. Rate-limit awareness

### `RateLimitTracker` (`agent/rate_limit_tracker.py`)

Parses the standard `x-ratelimit-*` response headers (12 fields:
request/token limits per minute and per hour, remaining counts, reset
seconds). Exposes:

* `RateLimitBucket` — single window (limit, remaining, reset_at).
* `RateLimitState` — full snapshot from one response.

### `NousRateGuard` (`agent/nous_rate_guard.py`)

Cross-process guard against Nous Portal 429 amplification. The OpenAI
SDK retries 3× on 429 by default; Hermes retries 3× on top. Without
coordination that is 9 calls per turn and a permanent ban is a likely
outcome.

`NousRateGuard` writes the latest reset time to
`~/.hermes/rate_limits/nous.json` (`agent/nous_rate_guard.py:26`) so
that **all** Hermes processes on the host respect the same back-off.
Header parsing (`_parse_reset_seconds()`,
`agent/nous_rate_guard.py:39`) checks
`x-ratelimit-reset-requests-1h`, `x-ratelimit-reset-requests`, and
`retry-after` in priority order.

### Account usage (`agent/account_usage.py`)

Polls provider account-usage endpoints (Anthropic, OpenAI, etc.) to
materialise:

* `AccountUsageWindow` — RPM, TPM, RPH, TPH counters.
* `AccountUsageSnapshot` — point-in-time snapshot.
* `RateLimitBucket` — same shape as in `rate_limit_tracker`.

`render_account_usage_lines()` formats the snapshot for the CLI
(`/usage`) and dashboard.

## 5. Billing

### `usage_pricing.py`

The cost-accounting layer. Key dataclasses (lines 19-67):

```python
@dataclass
class CanonicalUsage:
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int
    cache_write_tokens: int
    reasoning_tokens: int
    request_count: int

@dataclass
class BillingRoute:
    provider: str
    model: str
    base_url: str
    billing_mode: str         # "actual" | "estimated" | "included" | "unknown"

@dataclass
class PricingEntry:
    # per-million-token costs for input, output, cache variants
    input: float
    output: float
    cache_read: float
    cache_write: float
    source: str               # where the price came from

@dataclass
class CostResult:
    cost: float
    status: str               # "actual" | "estimated" | "included" | "unknown"
    source: str
```

`_OFFICIAL_DOCS_PRICING` (`agent/usage_pricing.py:77+`) is a snapshot
of provider price sheets for stable models. Newer / dated models fall
back to a substring lookup against this table; "unknown" status means
Hermes does not have a price (the `/usage` panel will say so).

The CLI's `/usage` command, the dashboard's billing panel, and the
session-end billing snapshot all use this module.

## 6. Retry policy

`agent/retry_utils.py` provides Tenacity-based decorators that classify
provider errors via `agent/error_classifier.py`:

* **Transient** (network blip, 5xx) — retry with exponential back-off.
* **Rate-limit** (429) — sleep until `reset_at`, then retry.
* **Auth** — drop the credential into `EXHAUSTED`, rotate the pool.
* **Bad request** (4xx other than 429) — surface to the user.

The agent loop never spins on a quota-exhausted credential because of
this classification + pool rotation.
