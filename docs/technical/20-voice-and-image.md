# 20 — Voice Mode and Image Generation

This chapter covers two adjacent media subsystems: the voice stack
(STT, TTS, voice mode coordination) and the image generation pipeline.
Both interact with multiple providers and follow the same "auxiliary
client + main provider" pattern as the core agent loop.

## 1. Voice mode

Voice mode lets the user talk to Hermes and have it talk back. It is
optional — the `voice` extra brings in `faster-whisper`, `sounddevice`
and `numpy` (which need wheel-only deps that some packagers cannot
ship; this is why `voice` is not in the default install).

### Toggling

* `/voice` slash command — toggles for the current session.
* `hermes voice <enable|disable>` — global default.
* `config.voice.enabled: true` — config-file form.
* `config.voice.push_to_talk: true` + `hotkey: "F12"` — push-to-talk
  mode.
* `config.voice.vad: true` — voice activity detection (default on).

### Coordination

`tools/voice_mode.py` is the coordination point. It:

1. Captures audio from the configured input device (sounddevice).
2. Detects voice activity (`webrtcvad`-style energy gate).
3. Buffers until silence + minimum-utterance length.
4. Hands the buffer to STT.
5. Feeds the resulting text into the agent loop as a normal user
   turn.
6. Captures the assistant text and pipes it through TTS.
7. Plays the output through the configured output device.

The implementation runs on a background thread so the CLI / TUI
remains responsive to keyboard input (the user can interrupt with
`Ctrl+C` mid-utterance).

### STT

Configured via `config.stt.*`:

```yaml
stt:
  provider: "faster-whisper"     # faster-whisper | openai | custom
  model: "small"                 # tiny | small | medium | large-v3
  device: "cpu"                  # cpu | cuda | auto
  compute_type: "int8"           # int8 | float16 | float32
  language: ""                   # autodetect when empty
```

`tools/transcription_tools.py` implements the dispatch:

* `provider: "faster-whisper"` — local model from `faster-whisper`.
  Caches model downloads to `~/.cache/huggingface/`.
* `provider: "openai"` — OpenAI's `whisper-1` over the API. Uses the
  `OPENAI_API_KEY`.
* `provider: "custom"` — any HTTP endpoint with the OpenAI-style
  `/audio/transcriptions` shape; configure `stt.base_url`.

The transcription tool is also exposed to the agent (so the model can
transcribe voice notes attached to inbound messages).

### TTS

Configured via `config.tts.*`:

```yaml
tts:
  provider: "edge"           # edge | elevenlabs
  voice: ""                  # provider-specific voice id; "" = default
  rate: "+0%"
  volume: "+0%"
  pitch: "+0Hz"
  output_format: "mp3"
  cache_path: ""             # default ~/.hermes/cache/tts
```

`tools/tts_tool.py` dispatches:

* `provider: "edge"` — Microsoft Edge TTS (free, no API key). Uses
  the `edge-tts` package; ships in core deps.
* `provider: "elevenlabs"` — ElevenLabs. Requires
  `ELEVENLABS_API_KEY`. Brought in by the `tts-premium` extra.

Both providers expose voice listing (`hermes voice voices`) and
pre-warming. Cached audio is keyed by `(provider, voice, text-hash,
rate, pitch, volume)` so re-saying the same line skips the network
round-trip.

### Local NeuTTS

`tools/neutts_synth.py` is a self-contained NeuTTS-style synthesiser
used for sample generation. It is **not** the user-facing TTS — it is
used for voice-cloning and sample generation in research workflows.
Sample voices live in `tools/neutts_samples/`.

### Voice on messaging platforms

When voice mode is enabled at the gateway level:

* Telegram voice notes (`.ogg`) → STT → text → agent.
* Discord voice messages → STT → text → agent.
* Signal voice notes → STT → text → agent.
* QQ voice → STT (via `QQ_STT_*` env vars) → text → agent.
* Outbound: assistant replies above
  `gateway.voice.audio_min_chars` are sent as voice notes; below that
  they are sent as text.

Voice channels (Discord, …) are a different surface — see the
voice-related fields on the Discord adapter.

### TUI voice integration

The Ink TUI exposes voice via:

* A `voice.start` JSON-RPC method (TS → Python).
* A `voice.delta` event (Python → TS) for live partial transcripts.
* A `voice.end` event with the final transcript.

The TUI can choose to render the partial transcript inline or in a
status pill; both ship out of the box.

### Voice mode configuration knobs

```yaml
voice:
  enabled: false
  push_to_talk: false
  hotkey: ""                  # PTT hotkey
  vad: true
  vad_aggressiveness: 2       # 0..3 (webrtcvad-style)
  silence_ms: 700             # ms of silence to end an utterance
  max_utterance_seconds: 60
  input_device: ""            # sounddevice index or name; "" = default
  output_device: ""
  pre_roll_ms: 200            # ms of audio before VAD-detected speech
  ducking: false              # lower system volume during playback
```

### Failure modes

* No audio device → fall back to text mode; CLI prints a one-line
  warning. `hermes doctor` lists the offending device.
* STT model missing on disk → `faster-whisper` auto-downloads on first
  use. The download is logged.
* Transcription returns empty → agent receives `(empty utterance)`
  and the model is instructed to ask the user to repeat.
* TTS fails → assistant reply is delivered as text; warning is logged.

## 2. Image generation pipeline

### Subsystems

The image-gen stack lives in three places:

| Layer | File | Role |
|-------|------|------|
| Provider abstraction | `agent/image_gen_provider.py` | ABC + provider-specific subclasses. |
| Provider registry | `agent/image_gen_registry.py` | Resolves provider/model identifiers to concrete classes. |
| Routing | `agent/image_routing.py` | Decides which provider to use for a given request based on user prefs / cost / availability. |
| Tool surface | `tools/image_generation_tool.py` | The OpenAI-style tool the model calls. |
| Vision counterpart | `tools/vision_tools.py` | Image *analysis* (Claude vision, GPT vision, etc.). |
| Backend providers | `tools/browser_providers/firecrawl.py` (for URL → image), `tools/web_tools.py` (for URL fetching) | Auxiliary helpers. |

### Supported providers

| Provider | Notes |
|----------|-------|
| `openai` | DALL-E 3 / GPT-image. Requires `OPENAI_API_KEY`. |
| `fal` | fal.ai (FLUX, SDXL, etc.). Requires `FAL_KEY`. |
| `bytedance` | Bytedance image API (when configured). |
| `stability` | Stability AI. |
| `together` | Together's image endpoints. |
| `gemini` | Gemini's image generation (where available). |

Adding a provider: subclass `ImageGenProvider`, implement `generate`
(returns bytes or a URL), and register in `image_gen_registry`.

### Routing rules

`agent/image_routing.py` picks a provider based on (in order):

1. Explicit `--provider` flag in the tool call.
2. `config.image_gen.preferred_provider`.
3. Whichever provider has a credential set.
4. Fall back to `openai` if everything else fails.

The `image_gen` plugin's UI exposes a model picker that mirrors this
routing.

### Tool surface

`image_generation_tool.py` registers a tool with this rough schema:

```json
{
  "name": "image_generate",
  "description": "Generate an image from a text prompt.",
  "parameters": {
    "type": "object",
    "properties": {
      "prompt": {"type": "string"},
      "negative_prompt": {"type": "string"},
      "size":     {"type": "string", "default": "1024x1024"},
      "n":        {"type": "integer", "default": 1, "maximum": 4},
      "provider": {"type": "string", "default": ""},
      "model":    {"type": "string", "default": ""},
      "style":    {"type": "string"},
      "seed":     {"type": "integer"}
    },
    "required": ["prompt"]
  }
}
```

The handler returns a JSON object with paths (or URLs) of the
generated images; the agent can then `read_file` to attach them or
post them via `send_message_*`.

### Storage

Generated images are saved to:

```
~/.hermes/cache/images/<session_id>/<ts>-<hash>.<ext>
```

A path is preferred over a URL because it survives provider-side
expiry, makes downstream tools deterministic, and avoids leaking the
provider's signed URL to the user.

### Cost tracking

`agent/usage_pricing.py` knows per-provider per-model image rates.
The `/usage` panel includes a "image generation" row when any image
calls happened in the session.

### Vision counterpart

Image *analysis* (the model "looking at" an image) is separate:

* `tools/vision_tools.py` exposes `vision_analyze(image, question)`.
* It dispatches to an auxiliary client (`agent/auxiliary_client.py`)
  configured via `config.auxiliary.vision.*`.
* Useful when the main model is text-only but the user attaches an
  image.

### `vision_image_attach_mode`

`config.agent.vision_image_attach_mode` controls whether attached
images go to the main model directly or are pre-analysed:

| Value | Behaviour |
|-------|-----------|
| `"auto"` | Attach natively when the active model reports `supports_vision=True` AND the user hasn't explicitly configured `auxiliary.vision.provider`. Otherwise pre-analyse with `vision_analyze`. |
| `"native"` | Always attach natively; non-vision models will either error at the provider or get a last-chance text fallback (see `run_agent._prepare_messages_for_api`). |
| `"text"` | Always pre-analyse with `vision_analyze` and prepend the description as text. The main model never sees pixels. |

The default `"auto"` is the right choice for most users.

### Image gen plugin

`plugins/image_gen/` adds a UI panel for image generation in the
dashboard. It wraps the `image_generate` tool with an interactive
form (prompt, size, n, seed, provider, model, style). Generated
images are inlined in the dashboard's session view.

### Plumbing for browser-supplied images

A user may paste an image URL or attach a screenshot from clipboard.
The flow:

1. CLI: `/image <path>` or `/paste` (clipboard).
2. Gateway: inbound media path → `~/.hermes/cache/inbox/...`.
3. Either form ends up as a `file://` reference in the next user
   message; the agent loop picks it up and decides whether to attach
   natively or call `vision_analyze`.

### Generation safety

* Prompts that the provider rejects (nudity, violence, etc.) return
  the provider's error verbatim.
* The agent never auto-saves images outside `~/.hermes/cache/images/`
  — the session id keeps a tenant boundary if Hermes runs multi-user.
* Provider-side watermarking is preserved (Hermes does not strip
  metadata).

## 3. Audio attachments end-to-end

A canonical voice-attachment flow:

```
Telegram user sends a voice note
  → Telegram adapter downloads .ogg → ~/.hermes/cache/inbox/telegram/<msgid>/voice.ogg
  → MessageEvent attaches it as audio
  → GatewayRunner._handle_message:
       voice mode is enabled →
            tools/transcription_tools.transcribe(voice.ogg) → text
       voice mode is disabled →
            attach as audio file path; let the agent decide
  → AIAgent.chat(text)
  → assistant reply
  → if reply length > gateway.voice.audio_min_chars and voice mode is on:
       tools/tts_tool.synthesize(reply, …) → speech.mp3
       Telegram adapter.send_audio(speech.mp3)
     else:
       Telegram adapter.send_message(reply)
```

The path through SessionDB is unchanged — voice notes are recorded as
attachments alongside their transcripts.

## 4. Image attachments end-to-end

```
User pastes a screenshot in the CLI
  → cli.HermesCLI._on_paste_event:
       PIL captures the clipboard image → ~/.hermes/cache/clipboard/<ts>.png
  → Next user message includes "[image: <path>]" reference
  → run_agent._prepare_messages_for_api decides:
       supports_vision = True (e.g. Claude 4.7) → attach natively
       supports_vision = False → vision_analyze(<path>) → attach text description
  → Provider call → assistant reply
```

If the user types `/image <path>`, step 1 is replaced by direct
copy into `~/.hermes/cache/clipboard/`.

## 5. Practical notes

* **`force_cpu`** — `config.code_execution.force_cpu: true` makes the
  REPL skip GPU-only kernels; useful when the host has flaky CUDA.
  Has no effect on TTS / STT (they have their own `device:` knob).
* **Whisper download size** — `tiny` (75 MB) → `large-v3` (3 GB).
  Pick `small` for English laptops; `medium` for multilingual; only
  go to `large-v3` if you have a beefy GPU.
* **ElevenLabs voice IDs** — pick from the dashboard at
  https://elevenlabs.io/voice-library, copy the voice id, set
  `tts.voice` accordingly.
* **fal.ai pricing** — varies wildly by model. The user-facing tool
  description does not include pricing on purpose; surface cost via
  `/usage` after the fact.

## 6. Where to look when…

| Symptom | Where |
|---------|-------|
| Voice mode does nothing | `tools/voice_mode.py:_run_loop`, `sounddevice` device list, `hermes doctor` |
| STT returns empty | `tools/transcription_tools.py`, model on disk, `language` setting |
| TTS audio is garbled | provider, sample-rate mismatch (`config.tts.output_format`) |
| Image gen 401 | wrong / missing API key for provider |
| Image gen produces low quality | provider-specific `model`/`style` knobs |
| Vision analysis is wrong | check whether `auxiliary.vision.provider` is set; verify the underlying model |
| Telegram voice not transcribed | `gateway.voice.enabled` plus the global `voice.enabled` |
