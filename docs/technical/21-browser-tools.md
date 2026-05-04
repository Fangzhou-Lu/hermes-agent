# 21 — Browser Tools (deep dive)

The browser-tool family is the largest single tool surface in Hermes
(~120 KB of code in `tools/browser_tool.py` alone). It lets the agent
drive a real Chromium instance over CDP, scrape a page, take a
screenshot, fill forms, click links, and recover from anti-bot pages.

This chapter is the canonical reference for the browser stack.

## 1. The pieces

| File | Role |
|------|------|
| `tools/browser_tool.py` | Top-level browser tool that the model calls (`browser_open`, `browser_click`, `browser_extract`, `browser_screenshot`, `browser_back`, `browser_navigate`, `browser_close`). |
| `tools/browser_cdp_tool.py` | Lower-level CDP wrapper that `browser_tool.py` builds on. Exposes raw CDP commands for advanced scenarios. |
| `tools/browser_dialog_tool.py` | Handles JS dialogs (`alert`, `confirm`, `prompt`, `beforeunload`). |
| `tools/browser_supervisor.py` | Process supervisor for the headless browser child(ren). Handles spawn, restart, idle eviction. |
| `tools/browser_camofox.py` | Camoufox flavour — a Firefox-based stealth browser. |
| `tools/browser_camofox_state.py` | Camoufox session state (cookies, fingerprint). |
| `tools/browser_providers/base.py` | `BrowserProvider` ABC. |
| `tools/browser_providers/browser_use.py` | BrowserUse provider. |
| `tools/browser_providers/browserbase.py` | Browserbase cloud-browser provider. |
| `tools/browser_providers/firecrawl.py` | Firecrawl scraper provider. |
| `tools/web_tools.py` | URL fetch utilities the model uses for non-interactive scraping. |
| `tools/url_safety.py` | URL allow/deny gating. |
| `tools/website_policy.py` | Higher-level "what categories of site is the agent allowed to visit". |

## 2. Provider selection

`config.browser.provider` picks one of:

* `browser_use` — the default. Local Chromium driven over CDP.
* `browserbase` — cloud-hosted Chromium; requires
  `BROWSERBASE_API_KEY` and `BROWSERBASE_PROJECT_ID`.
* `firecrawl` — a static scraper provider; does not support live
  interaction (no clicks, no form fills). Useful when you only need
  HTML.
* `camofox` — Camoufox (Firefox stealth). Rarely the best choice
  unless you are debugging anti-bot heuristics that target Chromium.

Selection happens once per agent. To switch mid-session:
`/config set browser.provider <name>` then `/reload`.

## 3. Process model

Local providers (`browser_use`, `camofox`) spawn a long-lived
browser subprocess managed by `tools/browser_supervisor.py`:

```
agent
  └─► browser_supervisor (Python)
        └─► chromium (or firefox) child
              └─► one tab per "page" the agent opens
```

The supervisor:

* Spawns lazily on first use.
* Recycles the browser after `config.browser.idle_seconds` of
  inactivity (default 600).
* Restarts the browser if the child crashes.
* Bounds memory: `config.browser.max_pages` (default 8); evicts
  least-recently-used tab when exceeded.
* Cleans up on agent shutdown.

The cloud providers (`browserbase`) use the same supervisor shape
but route every operation over HTTP.

## 4. Tool surfaces

### `browser_open(url, ...)`

Open a URL in a fresh tab. Returns:

```json
{
  "page_id": "p-7c4e",
  "url": "https://example.com/...",
  "title": "Example Domain",
  "status": 200,
  "snippet": "...first 2 KB of rendered text..."
}
```

Options:

* `wait_until: "load" | "networkidle" | "domcontentloaded"`
* `timeout_ms: 30000`
* `headers: {...}` (request headers)
* `cookies: [{...}, ...]`
* `viewport: { width, height }`

### `browser_extract(page_id)`

Returns the rendered text content of the page. Has two modes:

* **Plain text** (default) — `document.innerText`-style with
  whitespace normalisation.
* **Markdown** (`mode: "markdown"`) — preserves headings, links,
  images.
* **Structured** (`mode: "structured"`) — JSON of headings + links +
  forms, useful for action planning.

Result is auto-truncated to `max_result_size_chars` (default for this
tool: 200 KB).

### `browser_click(page_id, selector | text)`

Click by CSS selector, accessibility selector, or visible text. The
implementation auto-falls-back from selector → role → text → fuzzy
match. Returns the post-click state (URL, title, snippet).

### `browser_navigate(page_id, action)`

Action is one of `"back"`, `"forward"`, `"reload"`,
`"go_to:<url>"`. Equivalent to the user pressing the corresponding
browser button.

### `browser_screenshot(page_id, ...)`

Captures a PNG. Returns:

```json
{
  "path": "~/.hermes/cache/browser/<session>/<page>-<ts>.png",
  "width": 1280,
  "height": 800,
  "size_bytes": 142318
}
```

Options:

* `full_page: true` — capture the entire scroll height.
* `selector: "<css>"` — clip to the bounding box of the matched
  element.
* `quality: 90` — JPEG quality (when `format: "jpeg"`).

### `browser_close(page_id)`

Closes the tab. The supervisor evicts the least-recently-used tab
automatically when `max_pages` is exceeded, so explicit close is
optional.

### `browser_dialog_*`

`browser_dialog_tool.py` exposes:

* `browser_dialog_handle(page_id, action, prompt_text)` — accept,
  dismiss, or fill+accept the next dialog.
* `browser_dialog_set_default(page_id, action)` — set the default
  for all subsequent dialogs (useful for `beforeunload` storms).

### `browser_cdp(page_id, method, params)`

Raw CDP escape hatch (`tools/browser_cdp_tool.py`). For when the
high-level API does not cover the need (e.g. emulating a specific
device, capturing a HAR, intercepting network requests).

## 5. Page lifecycle

Pages have a finite lifetime:

```
agent calls browser_open       → page_id allocated
agent calls *_page_id          → page resolved from supervisor's pool
page idle for >5 min           → evicted (eviction warning logged)
agent calls *_evicted_page_id  → tool returns {"error": "page evicted"}
agent calls browser_close      → tab closed
agent ends, supervisor idle    → browser child shut down after idle_seconds
```

Best practice for the model: open → do work → close. The supervisor
keeps the browser warm across opens, so the cost of close+reopen is
low.

## 6. Form filling

`browser_tool.py` exposes `browser_fill(page_id, selector | label,
value)` for text fields, plus higher-level helpers:

* `browser_fill_form(page_id, fields)` — fills multiple fields at
  once.
* `browser_select(page_id, selector | label, value)` — for `<select>`
  elements.
* `browser_check(page_id, selector | label, checked)` — for
  checkboxes / radios.
* `browser_upload(page_id, selector, paths)` — for file inputs.

Each of these returns the post-fill state so the agent can verify.

## 7. Anti-bot handling

Pages that show anti-bot challenges (Cloudflare, hCaptcha, …) are a
common failure mode. The browser stack has multiple knobs:

* **Stealth defaults** — `browser_use` ships a stealth profile that
  hides `webdriver` flag, randomises a few navigator properties.
* **Camoufox** — switch `provider: "camofox"` for a stronger stealth
  profile. Trades raw speed for evasion.
* **Browserbase** — cloud browsers come from residential IPs with
  rotating fingerprints; the best option for sites that aggressively
  block headless Chromium.
* **Cookie persistence** — `tools/browser_camofox_state.py` saves a
  per-domain cookie + storage profile so subsequent visits look
  "returning user" rather than fresh.

The agent does not auto-solve CAPTCHAs. If a page presents one, the
extracted text reflects the challenge ("Please verify you are
human") and the agent can ask the user for help via `clarify`.

## 8. Sessions and cookies

By default each agent session gets a fresh browser profile. To
persist cookies across sessions:

```yaml
browser:
  profile: "default"           # named profile under ~/.hermes/cache/browser/profiles/
  persist_storage: true
```

Multiple named profiles let users isolate logged-in identities (e.g.
"work" vs "personal" Google accounts).

## 9. URL safety

Every URL passed to the browser stack runs through
`tools/url_safety.check_url()`:

* Cloud-metadata addresses (`169.254.169.254`, …) are hard-blocked;
  no override.
* Localhost is blocked by default; `allow_local=True` overrides only
  inside a sandboxed terminal backend.
* Private IP ranges are blocked by default with the same override
  rules.
* `config.security.url_blocklist` adds user-defined deny entries.
* `config.security.url_allowlist` overrides the blocklist for
  specific hosts.

`tools/website_policy.py` adds a category-based filter on top —
useful for deployments where the agent should only browse a curated
set of sites.

## 10. Performance characteristics

| Operation | Typical | Worst case |
|-----------|---------|------------|
| `browser_open` cold (first ever) | ~3 s (Chromium startup) | 15-20 s |
| `browser_open` warm | 200-400 ms | 2-3 s |
| `browser_extract` | 50-200 ms | 1-2 s |
| `browser_click` | 100-500 ms | 5 s (waiting for navigation) |
| `browser_screenshot` (viewport) | 200-500 ms | 2 s |
| `browser_screenshot` (full_page) | 1-3 s | 10 s |

Warming the browser before a long workflow is worth doing — issue a
no-op `browser_open` to a cheap URL (`https://example.com`) at the
start so the cost amortises.

## 11. Headless vs headed

`config.browser.headless: false` opens the browser with a UI. Useful
for:

* Debugging selector issues (you watch the agent click).
* Solving CAPTCHAs interactively (rare but possible).
* Live demos.

The TUI / CLI is unaffected — the headed window is an extra OS-level
window the user can ignore.

## 12. Known limitations

* **No native PDF rendering** — the browser will offer a download
  rather than render the PDF. Use `tools/web_tools.py` to fetch the
  PDF then use the appropriate skill or tool to extract.
* **No video transcription** — embedded video does not auto-transcribe.
  Use `tools/transcription_tools.py` after downloading the audio
  track.
* **No persistent session sync across machines** — if you run Hermes
  on machine A and want to share the browser cookies with machine B,
  copy `~/.hermes/cache/browser/profiles/<name>/` manually.
* **GPU acceleration off by default** — adds another set of
  fingerprintable surface area; turn on with
  `config.browser.gpu: true` if needed.

## 13. Worked example

User asks "What's on the first 5 results when I search 'hermes
agent'?":

```
agent receives prompt
  → web_search(query="hermes agent", n=5)
        → returns 5 result URLs
  → for each url:
        browser_open(url, wait_until="domcontentloaded")
        browser_extract(page_id, mode="structured")
        browser_close(page_id)
  → assemble summary
  → return text answer
```

In practice the agent uses `browser_use_results` (the higher-level
helper that wraps the loop above) so the trip is one tool call. The
flow above is what the helper expands to.

## 14. Tool dependency check

The browser tools' `check_fn` reports the tool as unavailable when:

* `browser_use`: Playwright browsers not installed (`npx playwright install`).
* `browserbase`: `BROWSERBASE_API_KEY` not set.
* `camofox`: `firefox` binary not on PATH.
* `firecrawl`: `FIRECRAWL_API_KEY` not set.

`hermes doctor` enumerates the missing prerequisites with one-line
fixes.

## 15. Adding a new browser provider

Subclass `tools/browser_providers/base.BrowserProvider`:

```python
class MyProvider(BrowserProvider):
    name = "myprovider"

    async def open_url(self, url: str, *, headless: bool = True) -> Page: ...
    async def screenshot(self, page: Page) -> bytes: ...
    async def extract(self, page: Page) -> str: ...
    async def click(self, page: Page, selector: str) -> None: ...
    # ... rest of the protocol
```

Register the class in `tools/browser_providers/__init__.py`. Then
`config.browser.provider: "myprovider"` selects it.

The `BrowserProvider` ABC is intentionally page-aware (rather than
URL-aware) so the supervisor can manage page lifecycle uniformly
across providers.

## 16. Testing

`tests/tools/browser/` carries the suite. Patterns:

* In-memory `FakeBrowserProvider` that returns canned page state for
  unit tests (no network).
* `pytest.mark.integration` tests that drive a real Chromium for full
  smoke coverage.
* Snapshot tests for `browser_extract` (compare structured output to
  golden files; update via `pytest --update-snapshots`).

## 17. Operational notes

* The `tools/browser_supervisor.py` log level is best left at INFO —
  DEBUG generates a lot of CDP traffic noise.
* Long-running gateways will accumulate browser children if the
  supervisor cannot evict them. The default `idle_seconds=600` is
  conservative; raise to keep them around longer if your workload
  is bursty.
* `~/.hermes/cache/browser/` can grow large (full-page screenshots
  are expensive); the cleanup plugin `plugins/disk-cleanup/` knows
  about this directory.
* Camoufox needs ~600 MB of disk for its browser binary; the
  Browserbase provider needs no local storage.
