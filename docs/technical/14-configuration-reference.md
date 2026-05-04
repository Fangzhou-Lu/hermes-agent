# 14 — Configuration Reference

This document is the complete reference for every knob Hermes Agent
exposes. The two stores are:

* **`~/.hermes/config.yaml`** — settings, behavioural knobs, choices.
  Loaded by `hermes_cli/config.py:load_config()`. Deep-merged over
  `DEFAULT_CONFIG` (`hermes_cli/config.py:386`). User edits override
  defaults; missing keys inherit defaults.
* **`~/.hermes/.env`** — secrets only. Loaded by
  `hermes_cli/env_loader.py:load_hermes_dotenv()`. The allow-list of
  recognised variables lives in `OPTIONAL_ENV_VARS`
  (`hermes_cli/config.py:1319`, ~137 entries) plus `_EXTRA_ENV_KEYS`.

The `hermes setup` wizard writes both. `hermes config <get|set|edit>`
manipulates `config.yaml`. `hermes auth <provider>` and the various
`*_auth.py` modules manipulate `.env`.

## 1. `config.yaml` — top-level keys

### `_config_version: 23`

Schema version for active migrations. Bumped only when an existing key
needs to be renamed or restructured. Adding a new key inside a section
does **not** require a bump (handled by deep-merge).

### `model: ""`

Active model identifier. Examples:

```
""                                  # resolved from provider on first run
"anthropic:claude-opus-4-7"
"openrouter:anthropic/claude-haiku-4-5"
"openai:gpt-5.2"
"custom:my-endpoint:my-model"
```

### `providers: {}`

Per-provider config, keyed by provider name. Example:

```yaml
providers:
  openai:
    base_url: ""               # leave empty for default
    timeout: 60
  openrouter:
    timeout: 90
    site_url: ""
    app_name: ""
  custom:
    my-endpoint:
      base_url: "https://my-host.example/v1"
      api_mode: "chat_completions"
      auth: "bearer"
```

### `fallback_providers: []`

Ordered list of provider:model identifiers used when the primary
provider returns an unrecoverable error. The agent rotates through
this list before surfacing the error to the user.

### `credential_pool_strategies: {}`

Per-provider strategy override (`FILL_FIRST`, `ROUND_ROBIN`, `RANDOM`,
`LEAST_USED`). Default: `FILL_FIRST`.

### `toolsets: ["hermes-cli"]`

List of enabled toolsets. See [03-tools.md](03-tools.md) for the
canonical set.

### `agent: { ... }`

Agent-level behaviour:

| Key | Default | Notes |
|-----|---------|-------|
| `max_turns` | `90` | Tool-calling iteration budget. |
| `gateway_timeout` | `1800` | Inactivity timeout (seconds) for gateway agent runs. `0` = unlimited. |
| `restart_drain_timeout` | `180` | Graceful drain timeout for `/restart`. |
| `api_max_retries` | `3` | Hermes-level retry loop wrapping the SDK retry. |
| `service_tier` | `""` | Provider-specific tier override (`"priority"`, `"flex"`, …). |
| `tool_use_enforcement` | `"auto"` | Inject system-prompt guidance for models that need a nudge to actually call tools. `"auto"` applies to gpt/codex; can also be `true`/`false` or a list of model substrings. |
| `gateway_timeout_warning` | `900` | Soft timeout: warn the user before escalating to the hard timeout. `0` = disable. |
| `gateway_notify_interval` | `180` | "Still working" interval. `0` = disable. |
| `gateway_auto_continue_freshness` | `3600` | Max age of an interrupted transcript for which we still inject the auto-continue note. |
| `vision_image_attach_mode` | `"auto"` | `auto` / `native` / `text`. |
| `prefill_messages_file` | `""` | Path to a JSONL of messages to prefill on session start. |

### `terminal: { ... }`

Terminal backend configuration:

```yaml
terminal:
  backend: "local"            # local | docker | ssh | modal | managed_modal | singularity | daytona | vercel_sandbox
  cwd: ""                     # default cwd; "" = launch directory
  timeout: 120                # default per-command timeout (seconds)
  command_timeout_max: 1800   # ceiling
  env: {}                     # extra env vars exported into commands
  docker:
    image: "nousresearch/hermes-sandbox:latest"
    network: "bridge"
  ssh:
    host: ""
    user: ""
    port: 22
  modal:
    app: "hermes-sandbox"
    region: ""
  daytona:
    workspace: ""
```

### `browser: { ... }`

```yaml
browser:
  provider: "browser_use"      # browser_use | browserbase | firecrawl
  headless: true
  timeout: 30
  user_agent: ""
  viewport: { width: 1280, height: 800 }
```

### `checkpoints: { ... }`

Filesystem checkpoint behaviour for `tools/checkpoint_manager.py`:

```yaml
checkpoints:
  enabled: true
  before_terminal: true
  before_file_write: true
  retention_days: 7
  store_path: ""               # default ~/.hermes/checkpoints
```

### `file_read_max_chars: 100000`

Maximum bytes read in one `read_file` call. Larger reads are paginated.

### `tool_output: { ... }`

```yaml
tool_output:
  default_max_chars: 50000
  per_tool:
    terminal: 80000
    web_extract: 200000
```

### `tool_loop_guardrails: { ... }`

Heuristic guardrails applied during the agent loop:

```yaml
tool_loop_guardrails:
  max_consecutive_same_tool: 8
  max_consecutive_terminal_failures: 5
  max_consecutive_empty_tool_results: 3
```

### `compression: { ... }`

```yaml
compression:
  engine: "compressor"         # see context.engine
  threshold_tokens: 0          # 0 = use provider default (typically 0.85 * context_length)
  protect_first_n: 4
  protect_last_n: 6
  summarizer_provider: "openrouter"
  summarizer_model: "anthropic/claude-haiku-4-5"
```

### `prompt_caching: { ... }`

```yaml
prompt_caching:
  enabled: true
  strategy: "system_and_3"     # places cache_control on system + last 3 non-system messages
  enabled_providers: ["anthropic"]
```

### `openrouter: { ... }`

```yaml
openrouter:
  site_url: ""
  app_name: "Hermes Agent"
  fallback_to_free_when_quota: true
```

### `bedrock: { ... }`

```yaml
bedrock:
  region: ""
  profile: ""
  arn_per_model: {}
```

### `auxiliary: { ... }`

Auxiliary client (used by curator, compressor, title generator,
insights):

```yaml
auxiliary:
  provider: "openrouter"
  model: "anthropic/claude-haiku-4-5"
  vision:
    provider: ""               # "" = use main provider
    model: ""
  use_for_compression: true
  use_for_curator: true
  use_for_titles: true
  use_for_insights: true
```

### `display: { ... }`

```yaml
display:
  banner: true
  spinner: "kawaii"            # kawaii | dots | spinner | line
  reasoning: "auto"            # auto | show | hide
  tool_progress: "new"         # off | new | all | verbose
  tool_progress_command: ""    # config-gated slash command
  show_status_bar: true
  show_runtime_footer: false
  skin: ""                     # name from assets/skins/ or ~/.hermes/skins/
  busy_enter_action: "interrupt"  # interrupt | queue | newline
```

### `dashboard: { ... }`

```yaml
dashboard:
  host: "127.0.0.1"
  port: 9119
  insecure: false              # required to bind non-localhost
  auth_token_path: ""
  open_browser: true
  tui: true                    # embed the Ink TUI
```

### `privacy: { ... }`

```yaml
privacy:
  send_diagnostics: false
  redact_logs: true
  share_anonymous_metrics: false
```

### `tts: { ... }`

```yaml
tts:
  provider: "edge"             # edge (free) | elevenlabs (premium)
  voice: ""
  rate: "+0%"
  volume: "+0%"
  pitch: "+0Hz"
  output_format: "mp3"
  cache_path: ""
```

### `stt: { ... }`

```yaml
stt:
  provider: "faster-whisper"   # faster-whisper | openai | custom
  model: "small"
  device: "cpu"
  compute_type: "int8"
  language: ""                 # autodetect
```

### `voice: { ... }`

```yaml
voice:
  enabled: false
  push_to_talk: false
  hotkey: ""
  vad: true
```

### `human_delay: { ... }`

```yaml
human_delay:
  enabled: false
  min_ms: 800
  max_ms: 2400
```

### `context: { ... }`

```yaml
context:
  engine: "compressor"         # compressor | lcm | <plugin name>
  include_files: [".hermes.md", "AGENTS.md", "SOUL.md", ".cursorrules"]
  threat_scan: true
```

### `memory: { ... }`

```yaml
memory:
  enabled: true
  provider: ""                 # name of an external provider (honcho|mem0|supermemory|...)
  honcho:
    workspace: ""
  mem0:
    api_key_env: "MEM0_API_KEY"
  scrub_streaming: true
```

### `delegation: { ... }`

```yaml
delegation:
  enabled: true
  max_depth: 3
  inherit_credentials: true
  default_max_iterations: 30
  scope_to_parent_toolset: true
```

### `goals: { ... }`

```yaml
goals:
  enabled: true
  store_path: ""               # default ~/.hermes/goals.json
  max_active: 5
```

### `skills: { ... }`

```yaml
skills:
  enabled: true
  enabled_optional: []         # names of opt-in skills to activate
  disabled: []                 # names hidden from /skills
  cache_seconds: 60
  sync:
    enabled: false
    target: "hermes-cloud"
    min_age_minutes: 5
```

### `curator: { ... }`

```yaml
curator:
  enabled: true
  paused: false
  interval_hours: 24
  idle_threshold_seconds: 600
  stale_skill_age_days: 30
  consolidate_min_overlap: 0.6
  max_skills_per_run: 50
  prefer_models: ["openrouter:anthropic/claude-haiku-4-5"]
```

### `honcho: {}`

Reserved (managed by `plugins/memory/honcho/`).

### `timezone: ""`

If empty, uses the OS's local timezone. Set to an IANA name (e.g.
`"America/New_York"`) to override.

### Per-platform configuration

Each messaging platform has a config block under `gateway` or as a
top-level key (legacy). Modern shape:

```yaml
discord:
  guild_id: ""
  channel_id: ""
  bot_token_env: "DISCORD_BOT_TOKEN"

telegram:
  bot_token_env: "TELEGRAM_BOT_TOKEN"
  webhook_url: ""

slack:
  bot_token_env: "SLACK_BOT_TOKEN"
  signing_secret_env: "SLACK_SIGNING_SECRET"

mattermost:
  url: ""
  bot_token_env: "MATTERMOST_BOT_TOKEN"

whatsapp:
  mode: "bridge"               # bridge | webhook
  bridge_url: ""

… (one block per supported platform)
```

The platform-specific keys are documented in
`hermes_cli/setup.py` (which prompts for them during the wizard) and
in `gateway/platforms/<name>.py` (which reads them).

### `approvals: { ... }`

```yaml
approvals:
  yolo: false
  auto_approve_session: false
  per_tool: {}                 # override per tool: { terminal: "session", file_write: "always" }
```

### `command_allowlist: []`

Allow-list rules; see [13-security.md](13-security.md).

### `quick_commands: {}`

Map of short aliases to command strings. The CLI expands them inline:

```yaml
quick_commands:
  "qstatus": "/status"
  "ml": "/model"
```

### `hooks: {}` and `hooks_auto_accept: false`

Shell-hook integration. `hermes hooks install bash` writes a hook
into `~/.bashrc` that calls `hermes` for certain triggers. Settings
control which triggers fire.

### `personalities: {}`

```yaml
personalities:
  engineer:
    system_prompt: "You are a precise software engineer..."
    model: "anthropic:claude-opus-4-7"
    toolset: "hermes-cli"
    emoji: "👷"
  helpful:
    system_prompt: "You are a helpful assistant."
```

### `security: { ... }`

```yaml
security:
  url_allowlist: []
  url_blocklist: []
  web_categories: { "*": "allow" }
  block_cloud_metadata: true   # cannot be disabled
```

### `cron: { ... }`

```yaml
cron:
  enabled: true
  output_retention_days: 30
  default_toolset: "hermes-cli"
```

### `kanban: { ... }`

```yaml
kanban:
  enabled: true
  store_path: ""               # default ~/.hermes/kanban.db
  default_board: "personal"
```

### `code_execution: { ... }`

```yaml
code_execution:
  default_kernel: "python3"
  force_cpu: false
  pre_imports: []
  packages: []
```

### `logging: { ... }`

```yaml
logging:
  level: "INFO"                # DEBUG | INFO | WARNING | ERROR
  rotate_max_mb: 50
  rotate_backups: 5
  redact: true
```

### `model_catalog: { ... }`

```yaml
model_catalog:
  refresh_days: 7
  source: "bundled"            # bundled | live
```

### `network: { ... }`

```yaml
network:
  prefer_ipv4: true
  http_proxy: ""
  https_proxy: ""
  no_proxy: []
  timeout: 60
```

### `sessions: { ... }`

```yaml
sessions:
  store_path: ""               # default ~/.hermes/state.db
  retention_days: 0            # 0 = forever
  fts_rebuild_on_open: false
  search_max_results: 100
```

### `onboarding: { ... }`

```yaml
onboarding:
  show_tips: true
```

### `updates: { ... }`

```yaml
updates:
  check_on_start: true
  channel: "stable"            # stable | beta
```

### `plugins: { ... }`

```yaml
plugins:
  enabled:
    honcho: true
    spotify: false
    observability/langfuse: false
```

## 2. `.env` — environment variables

The full list lives in `OPTIONAL_ENV_VARS` (`hermes_cli/config.py:1319`).
Below is the canonical inventory grouped by category.

### Provider keys

| Variable | Notes |
|----------|-------|
| `NOUS_API_KEY` | Nous Portal. |
| `NOUS_BASE_URL` | Override Nous Portal base URL. |
| `OPENAI_API_KEY` | OpenAI. |
| `OPENAI_BASE_URL` | OpenAI base URL override. |
| `OPENAI_ORG_ID` | OpenAI organisation. |
| `ANTHROPIC_API_KEY` | Anthropic. Also accepts setup-tokens (`sk-ant-oat*`) and `claude-code:<creds>`. |
| `OPENROUTER_API_KEY` | OpenRouter (vision, MoA, scraping helpers). |
| `GOOGLE_API_KEY` / `GEMINI_API_KEY` | Gemini (aliases). |
| `GEMINI_BASE_URL` | Gemini base URL override. |
| `XAI_API_KEY` | xAI. |
| `XAI_BASE_URL` | xAI base URL override. |
| `NVIDIA_API_KEY` | NVIDIA NIM. |
| `MISTRAL_API_KEY` | Mistral. |
| `KIMI_API_KEY` / `MOONSHOT_API_KEY` | Kimi/Moonshot. |
| `MINIMAX_API_KEY` | MiniMax. |
| `Z_AI_API_KEY` / `GLM_API_KEY` | z.ai / GLM. |
| `XIAOMI_API_KEY` | Xiaomi MiMo. |
| `HUGGINGFACE_API_KEY` | Hugging Face. |
| `OLLAMA_BASE_URL` | Ollama (local). |
| `LMSTUDIO_BASE_URL` | LM Studio (local). |
| `LLAMA_CPP_BASE_URL` | llama.cpp (local). |
| `BEDROCK_REGION` / `AWS_PROFILE` | AWS Bedrock. |

### Tool-specific keys

| Variable | Used by |
|----------|---------|
| `EXA_API_KEY` | `web_search`. |
| `BROWSERBASE_API_KEY`, `BROWSERBASE_PROJECT_ID` | Browserbase. |
| `BROWSER_USE_API_KEY` | BrowserUse. |
| `FIRECRAWL_API_KEY` | Firecrawl. |
| `PARALLEL_API_KEY` | Parallel Web. |
| `FAL_KEY` | fal.ai (image gen). |
| `ELEVENLABS_API_KEY` | Premium TTS. |
| `EDGE_TTS_*` | Edge TTS tuning (rare). |
| `OSV_API_BASE_URL` | OSV vuln check (rare; use default). |
| `TIRITH_API_KEY` | Nous Tirith security scanner. |

### Messaging — see also per-platform notes

| Variable | Used by |
|----------|---------|
| `TELEGRAM_BOT_TOKEN` | Telegram. |
| `TELEGRAM_ALLOWED_USERS`, `TELEGRAM_GROUP_ALLOWED_USERS`, `TELEGRAM_HOME_CHANNEL`, `TELEGRAM_HOME_CHANNEL_NAME` | Telegram allow-list / home channel. |
| `DISCORD_BOT_TOKEN` | Discord. |
| `DISCORD_*` (allowed users, home channel) | Discord. |
| `SLACK_BOT_TOKEN`, `SLACK_SIGNING_SECRET`, `SLACK_APP_TOKEN`, `SLACK_HOME_CHANNEL*` | Slack. |
| `SIGNAL_ACCOUNT`, `SIGNAL_HTTP_URL`, `SIGNAL_*` | Signal. |
| `SMS_*` | SMS gateway. |
| `MATRIX_*` | Matrix. |
| `MATTERMOST_*` | Mattermost. |
| `WHATSAPP_*` | WhatsApp bridge. |
| `BLUEBUBBLES_*` | BlueBubbles (iMessage). |
| `DINGTALK_*` | DingTalk. |
| `FEISHU_*` | Feishu / Lark. |
| `WECOM_*`, `WECOM_CALLBACK_*` | WeChat Work. |
| `WEIXIN_*` | WeChat. |
| `YUANBAO_*` | Alipay Mini Program. |
| `QQ_*`, `QQBOT_*` | QQ Bot. |
| `IRC_*` | IRC plugin. |

### Observability

| Variable | Used by |
|----------|---------|
| `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` / `LANGFUSE_BASE_URL` | observability/langfuse plugin. |
| `HERMES_LANGFUSE_ENV` / `_RELEASE` / `_SAMPLE_RATE` / `_MAX_CHARS` / `_DEBUG` | Langfuse tuning. |

### Operational env vars (not in `.env`)

These are read at runtime but not normally written to `.env`:

| Variable | Effect |
|----------|--------|
| `HERMES_HOME` | Override the entire `~/.hermes` root. |
| `HERMES_OPTIONAL_SKILLS` | Override the optional-skills directory (used by Nix/Brew). |
| `HERMES_TUI` | `1` to enable the Ink TUI by default. |
| `HERMES_DEBUG` | `1` to bump root logger to DEBUG. |
| `HERMES_DEV` | `1` to relax strict checks (file-permission warnings, etc). |
| `HERMES_PROFILER` | `1` to enable the cProfile harness. |
| `HERMES_MANAGED` | Set to `nix` / `brew` to indicate a managed install (suppresses `hermes update`). |
| `HERMES_DASHBOARD` | `1` to start the dashboard side-process inside the gateway container. |
| `HERMES_DASHBOARD_HOST` | Override dashboard bind host. |
| `HERMES_DASHBOARD_TUI` | `0` to disable the embedded TUI. |
| `EDITOR` | Used by `hermes config edit`, `hermes memory edit`, `hermes skills edit`. |
| `LANG` / `LC_ALL` | Must be UTF-8; the install script sets `C.UTF-8` if missing. |
| `TZ` | Used by `hermes_time.now()` if `config.timezone` is empty. |

## 3. Profiles

Profiles isolate `~/.hermes` state. They share the *parent* directory:

```
~/
├── .hermes/             # default profile
├── .hermes-dev/         # `hermes --profile dev`
└── .hermes-experiments/ # `hermes --profile experiments`
```

`hermes_cli/main._apply_profile_override()` parses `--profile <name>`
**before** any state-touching imports. `get_hermes_home()` then
returns the right path for the rest of the run.

Profile data does not auto-migrate; each profile has its own
`config.yaml`, `.env`, `state.db`, skills, etc.

### Active profile

`~/.config/hermes/active-profile` (one line, profile name) sets the
default profile when `--profile` is not passed. Writing this file is
done by `hermes profiles use <name>`.

## 4. Validation

* `_KNOWN_ROOT_KEYS` (`hermes_cli/config.py:2793`) is the set of
  recognised top-level keys. Unknown keys produce a warning at load
  time (not a hard failure — third-party plugins may add their own).
* `_VALID_CUSTOM_PROVIDER_FIELDS` (`hermes_cli/config.py:2802`) is the
  schema for entries under `providers.custom.<name>`.
* The setup wizard re-validates after writing so a corrupted
  `config.yaml` is caught at write time.

## 5. Migrations

Active migrations live in `hermes_cli/config.py` near `_config_version`.
Each migration is a function that reads the old shape and writes the
new shape, gated by version. Adding a key inside an existing section
does **not** require a migration.

Migrations log their actions to `agent.log` so a user can see what
changed.

## 6. Where settings come from at runtime

The precedence (highest first) is:

1. CLI flags (`--model`, `--profile`, `--quiet`, …).
2. Per-call kwargs to `AIAgent(**kwargs)`.
3. Environment variables (`HERMES_*`, plus tool-specific ones).
4. `~/.hermes/config.yaml`.
5. `DEFAULT_CONFIG` (in `hermes_cli/config.py`).
6. Code defaults inside individual modules.

If a setting is read in two different places, this is the order
that wins.

## 7. Editing tips

* `hermes config edit` opens `~/.hermes/config.yaml` in `$EDITOR` and
  re-validates on save.
* `hermes config get a.b.c` and `hermes config set a.b.c value`
  manipulate dotted paths.
* `hermes setup` is idempotent — running it again only changes what
  you re-confirm.
* Comments in `config.yaml` are preserved across `hermes config set`.
  (Implemented via `ruamel.yaml`-style round-tripping where possible;
  fallback `pyyaml` strips them in pathological cases. Use
  `hermes config edit` if comments matter.)

## 8. Diagnostics

`hermes doctor` reads the resolved config (after merge + migrations)
and reports anomalies:

* Unknown keys at the root.
* Mismatched `_config_version`.
* Provider entries with no credential.
* Plugins listed as enabled but not installed.
* Skills in `enabled_optional` that are not present.

`hermes config show` prints the resolved config (post-merge) for
inspection. Secrets are redacted unless `--show-secrets` is passed.
