# 24 — Provider Adapters: Deep Dive

This chapter is a per-adapter deep dive. Where
[06-providers.md](06-providers.md) covers the abstract contract and
[24] (this file) goes file by file: what each adapter does, the public
helpers it exposes, the quirks it handles, and the tests that pin
its behaviour.

The codebase distinguishes:

* **Transports** in `agent/transports/` — message-format translation
  for one wire protocol.
* **Adapters** in `agent/<provider>_adapter.py` — provider-specific
  glue for thinking budgets, max-output limits, OAuth quirks,
  fingerprinting, region selection, custom auth.

A provider may use a transport directly (no adapter) or be wired
through both.

## 1. `agent/transports/base.py` — `ProviderTransport`

The base class:

```python
class ProviderTransport(ABC):
    api_mode: str

    @abstractmethod
    def convert_messages(self, messages: list[dict]) -> Any: ...

    @abstractmethod
    def convert_tools(self, tools: list[dict]) -> Any: ...

    @abstractmethod
    def build_kwargs(self, *, messages, tools, model, **kw) -> dict: ...

    @abstractmethod
    def normalize_response(self, raw: Any) -> NormalizedResponse: ...

    # Optional:
    def validate_response(self, raw: Any) -> None: ...
    def extract_cache_stats(self, raw: Any) -> dict: ...
    def map_finish_reason(self, raw: str) -> str: ...
    def stream_chunks(self, raw_stream) -> Iterable[NormalizedChunk]: ...
```

`api_mode` is the user-visible identifier (`"anthropic_messages"`,
`"chat_completions"`, `"codex_responses"`, `"bedrock_converse"`). It
appears in `~/.hermes/config.yaml` provider blocks and in
`SessionDB.sessions.api_mode`.

## 2. `agent/transports/types.py`

Shared dataclasses every transport uses:

### `ToolCall`

```python
@dataclass
class ToolCall:
    id: Optional[str]                  # canonical id; may be None
    name: str
    arguments: str                     # JSON string
    provider_data: Optional[Dict[str, Any]] = None  # per-protocol metadata
```

`provider_data` carries protocol-specific extras the rest of the
agent never reads but that the transport *re-emits* on the next API
call:

* Codex: `{"call_id": "call_XXX", "response_item_id": "fc_XXX"}`.
* Gemini: `{"extra_content": {"google": {"thought_signature": "..."}}}`.
  Required — Gemini 3 thinking models reject the next request without
  a re-played `thought_signature`.

`ToolCall` also exposes legacy properties (`type`, `function`,
`call_id`, `response_item_id`, `extra_content`) so the agent loop's
historic `tc.function.name` access pattern still works.

### `Usage`

```python
@dataclass
class Usage:
    input_tokens: int = 0
    output_tokens: int = 0
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    reasoning_tokens: int = 0
    request_count: int = 1
```

Adapters fill cache fields when the provider reports them; otherwise
they remain 0.

### `NormalizedResponse`

```python
@dataclass
class NormalizedResponse:
    text: str = ""
    tool_calls: list[ToolCall] = field(default_factory=list)
    usage: Usage = field(default_factory=Usage)
    finish_reason: str = "stop"
    reasoning: str = ""
    provider_data: dict[str, Any] | None = None
```

`finish_reason` is the OpenAI-style canonical value (`"stop"`,
`"length"`, `"tool_calls"`, `"content_filter"`, `"error"`).

### Helpers

* `build_tool_call(id, name, arguments, **provider_data)` — fold
  protocol-specific kwargs into `provider_data`.
* `map_finish_reason(reason, mapping)` — translate provider-specific
  stop reasons to canonical ones.

## 3. `agent/transports/chat_completions.py`

The default OpenAI-compatible transport. Powers OpenAI itself,
OpenRouter, llama.cpp, LM Studio, vLLM, NVIDIA NIM, MiMo, MiniMax,
Z.AI / GLM, Kimi, Mistral (when used in OpenAI-compat mode), and a
catch-all `provider: "custom"` shape.

### Class

```python
class ChatCompletionsTransport(ProviderTransport):
    api_mode = "chat_completions"
```

### `build_kwargs`

Builds the `chat.completions.create` payload. Notable bits:

* **Tool choice** — defaults to `auto`; switches to `required` when
  the agent has hinted that a tool *must* be used (rare).
* **Temperature** — pulled from `config.providers.<provider>.temperature`
  with model-specific overrides for known reasoning families.
* **Reasoning effort** — passed through for OpenAI-style providers
  that accept it; otherwise translated by adapters.
* **Stream** — always `True` in interactive use.

### Gemini OpenAI-compat detection

`_is_gemini_openai_compat_base_url(base_url)` (line 93) recognises
Google's OpenAI-compat endpoint. When matched, the transport applies:

* `_build_gemini_thinking_config(model, reasoning_config)` (line 22)
  — thinking budget translation for the Google variants.
* `_snake_case_gemini_thinking_config(config)` (line 78) — Google
  expects snake_case keys; OpenRouter accepts camelCase. The
  conversion is per-host.

These helpers are split out so they can be unit-tested in isolation
(`tests/test_ollama_num_ctx.py`, `tests/test_minimax_model_validation.py`,
etc., follow the same pattern for other providers).

### Streaming

`stream_chunks(raw_stream)` yields `NormalizedChunk` items:

* Text deltas.
* Tool-call deltas (constructed incrementally; tool args streamed by
  fragment).
* Reasoning deltas (when the provider streams them).

The OpenAI Python SDK exposes deltas as `ChatCompletionChunk` objects;
the transport peels them apart and emits `NormalizedChunk` values.

### Cache stats

OpenAI's `usage.prompt_tokens_details.cached_tokens` is mapped onto
`Usage.cache_read_tokens`. Other providers that report cache stats
through this transport (e.g. OpenRouter when using Anthropic models)
populate the same field.

## 4. `agent/transports/anthropic.py`

The Anthropic Messages transport.

### Class

```python
class AnthropicMessagesTransport(ProviderTransport):
    api_mode = "anthropic_messages"
```

### `convert_messages`

Translates the Hermes shared format into Anthropic's:

* `role: "system"` → top-level `system` parameter (Anthropic requires
  it outside the messages list).
* `role: "tool"` → user message with `content: [{type: "tool_result",
  tool_use_id: ..., content: ...}]`.
* `role: "assistant"` with `tool_calls` → assistant message with
  `content: [{type: "tool_use", id: ..., name: ..., input: ...}, ...]`.

### `convert_tools`

OpenAI-style `function` blocks → Anthropic's `input_schema` field.

### `extract_cache_stats`

Anthropic returns `usage.cache_creation_input_tokens` and
`usage.cache_read_input_tokens`. Both are mapped onto
`Usage.cache_write_tokens` and `Usage.cache_read_tokens`.

### Working with Anthropic's pagination

Anthropic's stream uses `event: message_start`, `event: content_block_*`,
`event: message_delta`, `event: message_stop`. The transport's
`stream_chunks` understands all of them, including the
`thinking_block` events.

## 5. `agent/anthropic_adapter.py` (~82 KB)

The Anthropic-specific quirk shop. Organised by topic:

### Auth

Three auth flows are supported:

* **API keys** — `sk-ant-api*`.
* **OAuth setup-tokens** — `sk-ant-oat*`. These are short-lived
  tokens issued by Claude.ai's developer console; the adapter knows
  to refresh them.
* **Claude Code credentials** — when Claude Code is installed
  locally, its credential store can be reused (`~/Library/Application
  Support/Claude/...` on macOS, etc.).

`detect_auth_kind(token)` classifies each candidate string. The
credential pool then routes to the right refresh / validation
path.

### Thinking budgets

```python
THINKING_BUDGET = {
    "xhigh": 32000,
    "high": 16000,
    "medium": 8000,
    "low": 4000,
}
```

`map_reasoning_effort(model, requested) -> int | None` (lines 56-63)
applies model-aware adjustments:

* On Claude 4.7+, `xhigh` and `max` map to 32 k.
* On Claude 4.6, `xhigh`/`max` downgrade to `max` (16 k).
* On older models that do not support thinking, returns `None` (the
  transport then drops the field).

### Output limits

`_ANTHROPIC_OUTPUT_LIMITS` (lines 84-108) is a per-model cap table.
Substring matching handles dated model ids (e.g.
`claude-sonnet-4-6-20251015` inherits from `claude-sonnet-4-6`).

`_get_anthropic_max_output(model)` (lines 115-133) is the public
helper. `_resolve_positive_anthropic_max_tokens(model, requested)`
(lines 136-150+) applies the user's request inside the cap.

### Prompt caching

`apply_anthropic_cache_control(messages)` (in
`agent/prompt_caching.py`, called from this adapter) places up to
four `cache_control` markers — system + last 3 non-system messages.
This produces the "system_and_3" pattern.

### Service tier

Anthropic's `service_tier` parameter (`"priority"`, `"flex"`) is
plumbed through. `/fast` toggles between `priority` and the default.

### Known issues handled

* Anthropic 4xx with `tools.*.input_schema` errors when a tool's
  schema includes JSON-Schema features Anthropic does not understand
  — `agent.gemini_schema.sanitize_for_anthropic(schema)` normalises
  before sending.
* Anthropic's "tool_use_id required" error if a tool result is sent
  without the matching `tool_use` id — the transport's
  `convert_messages` keeps a stack of in-flight ids.

## 6. `agent/transports/codex.py` and `agent/codex_responses_adapter.py`

The Codex / OpenAI Responses transport pair.

### Transport (`agent/transports/codex.py`)

Implements the OpenAI Responses API protocol. `api_mode =
"codex_responses"`. Key methods are minimal — most logic lives in the
adapter.

### Adapter (~44 KB)

#### Message translation

`_chat_content_to_responses_parts(content)` (line 47) — translates
Hermes-shared message parts to the Responses API "parts" shape.
Handles text, tool_use, tool_result, image_url, and the Responses-
specific reasoning items.

`_responses_tools(tools)` (line 205) — translates tools.

#### Reasoning extraction

`_extract_responses_reasoning_text(item)` (line 768) — Codex emits
reasoning as nested `reasoning` items inside the response. The helper
flattens the tree into a single string.

#### Pre-flight validation

* `_preflight_codex_input_items(items)` (line 426) — catches malformed
  parts before they reach the API.
* `_preflight_codex_api_kwargs(kwargs)` (line 604) — validates the
  full kwargs payload (e.g. ensures `model` is on the supported
  list).

These exist because Codex 4xx errors are particularly opaque; pre-flight
is cheaper than parsing the resulting error message.

#### Response normalisation

`_normalize_codex_response(raw)` (line 789) — produces the shared
`NormalizedResponse`.

#### Tool-call propagation

Codex tool calls carry both `call_id` and `response_item_id`. Both
are stored in `ToolCall.provider_data` so the next request can
include them — Codex requires the original ids when continuing the
conversation.

## 7. `agent/transports/bedrock.py` and `agent/bedrock_adapter.py`

AWS Bedrock Converse API.

### Transport

`api_mode = "bedrock_converse"`. Lightweight wrapper.

### Adapter (~49 KB)

#### Region resolution

`_resolve_region(model, config)` walks:

1. Explicit `BEDROCK_REGION` env.
2. `config.bedrock.region`.
3. `AWS_REGION` env.
4. boto3's profile-based default.

A wrong region produces an immediate, clear error rather than a 403
deep in boto3.

#### Client pool

`get_bedrock_client(region, profile)` is keyed by `(region, profile)`
so multiple regions can run concurrently without re-creating the
client. `boto3` clients are not thread-safe in general; the pool
stores per-thread instances.

#### Format translation

* `convert_tools_to_converse(tools)` (line 397) — OpenAI →
  Converse tool spec.
* `convert_messages_to_converse(messages)` (line 480) — message
  translation. Bedrock's per-message structure differs from
  Anthropic's; this is the larger of the two helpers.

#### Streaming

`normalize_converse_stream_events(stream, callback)` (line 688) —
the Converse stream uses `messageStart`, `contentBlockStart`,
`contentBlockDelta`, `contentBlockStop`, `messageStop`,
`metadata`. The helper consumes them callback-style (line 704+) so
the transport can yield as soon as deltas arrive.

#### Stop reasons

`_converse_stop_reason_to_openai(stop_reason)` (line 603) — maps
Converse's `tool_use`, `end_turn`, `max_tokens`, `stop_sequence`,
`guardrail_intervened` to canonical values.

#### Region quirks

Some regions only support certain models; the adapter maintains a
small allow-list mapping that `hermes doctor` reads to warn the user
when their `BEDROCK_REGION` cannot serve the active model.

## 8. `agent/gemini_native_adapter.py` (~33 KB)

Google Gemini API (the native shape, not the OpenAI-compat or
CloudCode endpoints).

### Tier detection

`probe_gemini_tier(api_key)` (lines 47-120) issues a cheap
`/v1beta/models` call and inspects the response headers and quota
errors to detect free vs paid tier. The result is cached for 24 h to
avoid re-probing every cold start.

### Quota errors

`is_free_tier_quota_error(err)` (lines 121-136) classifies
provider-side errors so the credential pool can mark the credential
exhausted with the right reset time (Gemini free-tier resets at
midnight UTC; paid resets per minute / per hour).

### Content translation

`_build_gemini_contents(messages)` (line 276) — translates the
Hermes-shared format to Gemini's `contents` array. Handles text,
images, tool calls, and the Gemini-specific `thoughtSignature`
roundtrip.

### Tool translation

`_translate_tools_to_gemini(tools)` (line 330) — translates tool
schemas. Gemini wants `function_declarations` rather than `functions`;
the parameter names also differ subtly.

### Thinking config

`_normalize_thinking_config(config)` (lines 372-387) — maps Hermes's
unified `reasoning_effort` to Gemini's `thinkingConfig` shape with
`thinkingBudget` (token count) and `includeThoughts` (boolean).

### Response translation

`translate_gemini_response(raw) -> NormalizedResponse` (line 474) —
folds Gemini's per-part response (`functionCalls`, `inlineData`,
`text`) into the shared shape.

### Streaming

`_GeminiStreamChunk` (line 543+) is the per-chunk dataclass. Streaming
is line-delimited JSON; the chunker buffers across newlines and emits
chunks as soon as a full JSON object is parsed.

## 9. `agent/gemini_cloudcode_adapter.py`

Parallel adapter for the CloudCode endpoint (Google's "Gemini for
Code Assist" surface). Same shape as the native adapter, but:

* Uses `google_oauth.py` for auth (CloudCode credentials are tied to
  a Google account; not API keys).
* Uses a different base URL.
* Has slightly different streaming shape (CloudCode wraps responses
  in `{"v": ..., "data": ...}` envelopes).

The two adapters share most translation helpers; the CloudCode
variant is mostly a thin wrapper plus auth.

## 10. `agent/google_oauth.py`

Browser-based OAuth flow for Google APIs.

### Behaviour

* Spawns a local HTTP server on a free port.
* Opens the user's browser to Google's consent page.
* Waits for the callback; captures the auth code.
* Exchanges for access + refresh tokens.
* Stores at `~/.hermes/auth/google.json` (or
  `~/.hermes/auth/google-cloudcode.json`).

### Refresh

`refresh_token(creds)` is called automatically by the adapter when a
401 is received. The CredentialPool's exhausted state is *not* used
for OAuth refresh failures — those go through a separate path.

## 11. `agent/google_code_assist.py`

Google Code Assist client used by the CloudCode flow. Wraps the
Google API Discovery client with Hermes-shaped helpers. Independent
of the chat path — used by the `gemini-cli` integration and by skills
that read Code Assist's project state.

## 12. `agent/copilot_acp_client.py`

GitHub Copilot for VS Code client. Speaks the Copilot Editor Protocol
over a child-process pipe.

### Behaviour

* Spawns the Copilot extension binary (or the user-installed
  language server).
* Sends authentication.
* Surfaces the Copilot model list.
* Wraps Copilot's completion API in a chat-completions-compatible
  shape so the rest of Hermes does not need to know the difference.

### Limitations

Copilot's model list and capabilities are limited; not all Hermes
features (parallel tool calling, vision, deep reasoning) work
through this path. The adapter degrades gracefully.

## 13. `agent/moonshot_schema.py`

Moonshot / Kimi-specific schema shims. Moonshot's `tools` field
accepts a slightly different parameter shape than OpenAI; this module
adjusts the JSON schema before sending.

## 14. `agent/lmstudio_reasoning.py`

LM Studio (local LLM host) emits reasoning blocks inside the response
text using a custom delimiter. This module parses them out and
populates `NormalizedResponse.reasoning`.

## 15. `agent/gemini_schema.py`

Gemini-specific JSON-Schema sanitisation. Gemini does not support
some schema features (`additionalProperties`, deeply-nested
recursion). This module removes / replaces them so tools whose schema
were authored for OpenAI / Anthropic can pass Gemini validation.

Also exposes `sanitize_for_anthropic(schema)` — Anthropic shares a
subset of these constraints.

## 16. `agent/error_classifier.py`

Maps provider exceptions to one of:

* `transient` — connection drop, 5xx, timeout. Retry.
* `rate_limit` — 429. Sleep until the reset, then retry.
* `auth` — 401 / 403 / OAuth refresh failure. Mark credential
  exhausted; rotate.
* `bad_request` — 400 with non-recoverable shape. Surface to user.
* `content_filter` — provider's safety system blocked the request.
  Surface to user.

The classifier's output drives `agent/retry_utils.py`'s decisions.

## 17. `agent/retry_utils.py`

Tenacity-based decorator factory:

```python
@with_retry(provider="anthropic")
def call_anthropic(...): ...
```

The decorator:

1. Catches the exception.
2. Calls `error_classifier.classify(err)`.
3. Sleeps according to the class (`transient` → exp backoff;
   `rate_limit` → sleep until reset).
4. Retries up to `agent.api_max_retries` (default 3).
5. On final failure, marks the credential as appropriate and
   re-raises.

Per-provider rate-limit headers are extracted via
`agent/rate_limit_tracker.py` and stored in `~/.hermes/rate_limits/`
so other Hermes processes on the same host do not retry into the
same wall.

## 18. `agent/credential_pool.py`

Already covered in [06-providers.md](06-providers.md). One bit to
amplify: pools are keyed by **`(provider, custom_endpoint)`**, so the
user can have multiple OpenAI-compat endpoints without their
credentials colliding.

`PooledCredential.last_error_reset_at` is the reset time the
classifier extracted from the failure — used to know when to take the
credential out of `STATUS_EXHAUSTED`.

## 19. Provider catalog (`hermes_cli/model_catalog.py`)

The static list of providers + models. Each entry:

```python
ProviderEntry(
    id="anthropic",
    label="Anthropic",
    base_url="https://api.anthropic.com",
    api_mode="anthropic_messages",
    auth_env="ANTHROPIC_API_KEY",
    base_url_env="ANTHROPIC_BASE_URL",
    models=[ModelEntry(...), ...],
)
```

The model picker (`hermes model`, `/model`) walks this list. New
providers must add an entry; otherwise the user cannot select them
without manually editing `config.providers`.

## 20. Provider-specific tests

Each adapter has a tightly-focused test:

| Test file | What it pins |
|-----------|--------------|
| `tests/test_minimax_model_validation.py` | MiniMax model id validation |
| `tests/test_minimax_oauth.py` | MiniMax OAuth flow |
| `tests/test_ollama_num_ctx.py` | Ollama context-window kwarg name |
| `tests/test_empty_model_fallback.py` | Empty model id resolution |
| `tests/test_account_usage.py` | Account-usage parsers |
| `tests/test_base_url_hostname.py` | Base-URL parsing for credential pool keys |

Patterns used in these tests:

* Build a `transport.build_kwargs(...)` call with synthetic input.
* Assert on the resulting kwargs dict.
* No network. No fake clients. Just translation logic.

Adapter-level behaviour (token refresh, region resolution) is tested
similarly with `monkeypatch` for the underlying HTTP calls.

## 21. Adding a provider — checklist

In one place:

* [ ] `agent/transports/<api_mode>.py` (only if the protocol is new).
* [ ] `agent/<provider>_adapter.py` (only if quirks exist).
* [ ] `hermes_cli/model_catalog.py` — `ProviderEntry`.
* [ ] `hermes_cli/config.py` — `OPTIONAL_ENV_VARS` entries for
      `<PROVIDER>_API_KEY` + `<PROVIDER>_BASE_URL`.
* [ ] `agent/credential_sources.py` — discovery for the new env vars
      (usually one line).
* [ ] `agent/usage_pricing.py` — pricing entries (or accept "unknown"
      until pricing is published).
* [ ] `agent/error_classifier.py` — provider-specific 429 header
      parser if the provider does not follow the standard
      `x-ratelimit-*` shape.
* [ ] One test in `tests/agent/` exercising `build_kwargs`.
* [ ] One test exercising error classification.
* [ ] Doc entry under [06-providers.md](06-providers.md) and
      [24-adapters-deep-dive.md](24-adapters-deep-dive.md).

## 22. Adapter testing patterns

### Build-kwargs unit test

```python
def test_anthropic_build_kwargs_sets_thinking_budget():
    transport = AnthropicMessagesTransport()
    kwargs = transport.build_kwargs(
        messages=[{"role": "user", "content": "hi"}],
        tools=[],
        model="claude-opus-4-7",
        reasoning_effort="high",
    )
    assert kwargs["thinking"]["budget_tokens"] == 16000
```

### Normalisation roundtrip

```python
def test_codex_normalise_tool_call_preserves_call_id():
    raw = {"output": [{"type": "function_call",
                       "call_id": "call_abc",
                       "name": "x", "arguments": "{}"}]}
    nr = _normalize_codex_response(raw)
    assert nr.tool_calls[0].provider_data["call_id"] == "call_abc"
```

### Error classification

```python
def test_anthropic_429_reset_at_parsed():
    err = make_anthropic_429(reset_at=1714521234)
    cls = error_classifier.classify(err)
    assert cls.kind == "rate_limit"
    assert cls.reset_at == 1714521234
```

These three tests catch ~90% of regressions from SDK upgrades.

## 23. Adapter performance tips

* **Avoid per-call boto3 client** — use the cached `(region, profile)`
  client.
* **Cache schema translations** — `convert_tools` for a given list of
  tool schemas is deterministic; cache by tool-list hash to avoid
  recomputing per turn.
* **Lazy SDK imports** — `import anthropic` is ~150 ms; do it inside
  the build-client helper, not at module level.
* **Stream pre-allocation** — for large streams, pre-allocate a
  bytearray for accumulating text deltas; repeated string
  concatenation is O(n²).

## 24. Where to look when…

| Symptom | Where |
|---------|-------|
| Anthropic 400 "tool input_schema invalid" | `agent/gemini_schema.sanitize_for_anthropic` |
| Codex 400 "missing call_id" | `agent/codex_responses_adapter._preflight_codex_*` |
| Bedrock "model not in region" | `agent/bedrock_adapter._resolve_region` + region allow-list |
| Gemini 429 immediately on a fresh key | `agent/gemini_native_adapter.probe_gemini_tier` |
| OpenAI prompt-cache hits not visible | `extract_cache_stats` in `agent/transports/chat_completions.py` |
| OAuth token expired mid-session | `agent/google_oauth.refresh_token` + adapter retry |
| Streaming cuts off mid-chunk | transport's `stream_chunks` — check buffer flushing |
| Tool call repeated by the model | `provider_data` not roundtripped (Codex / Gemini) |
| Reasoning text missing | adapter's `_extract_*_reasoning_text` not implemented |
