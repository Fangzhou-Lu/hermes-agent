# 09 — Skills System (deep dive)

This document describes the Hermes skills subsystem in depth: the skill
file format, the index, the discovery / load pipeline, the Hub, the
guard, the sync subsystem, the curator, and the relationship between
agent-created skills and human-authored skills.

For context, [03-tools.md](03-tools.md) gives the high-level overview;
this document is the canonical reference.

## 1. What a skill *is*

A skill is **procedural memory** — a directory containing instructions
plus optional supporting files that the agent loads into the
conversation when it decides the task matches. Compared to a tool, a
skill is:

* **Data, not code** — written in Markdown, no Python class to register.
* **LLM-interpreted** — the model decides how to follow the
  instructions; nothing is executed unless the skill itself contains
  shell commands the model invokes via `terminal`.
* **Composable** — a skill can reference other skills (`related_skills`
  in the frontmatter).
* **Versioned** — the frontmatter carries `version` and `author`.

The difference from a system-prompt blob is that skills are
**lazy-loaded**: they only enter the conversation when explicitly
invoked, so they do not eat the context budget by default.

## 2. Layout

```
skills/<category>/<skill-name>/
├── SKILL.md            # required
├── references/         # optional — extra files (tier 3)
│   └── ...
├── templates/          # optional — file templates the agent can copy
│   └── ...
└── assets/             # optional — images, fixtures, sample data
    └── ...
```

Categories under `skills/` (24 in v0.12.0):

```
apple, autonomous-ai-agents, creative, data-science, devops, diagramming,
dogfood, domain, email, gaming, gifs, github, index-cache, inference-sh,
mcp, media, mlops, note-taking, productivity, red-teaming, research,
smart-home, social-media, software-development, yuanbao
```

`index-cache/` is *not* a skill category — it holds the precomputed
Skills-Hub index.

`optional-skills/` mirrors the same layout but is opt-in. Categories:
`autonomous-ai-agents, blockchain, communication, creative, devops,
dogfood, email, health, mcp, migration, mlops, productivity, research,
security, web-development`. Each carries a top-level
`DESCRIPTION.md`.

User skills live under `~/.hermes/skills/`. They are discoverable in
exactly the same way; the only difference is provenance and write
permission.

## 3. The `SKILL.md` format

YAML frontmatter + Markdown body. A real example
(`skills/software-development/test-driven-development/SKILL.md`):

```markdown
---
name: test-driven-development
description: "TDD: enforce RED-GREEN-REFACTOR, tests before code."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
metadata:
  hermes:
    tags: [testing, tdd, development, quality, red-green-refactor]
    related_skills: [systematic-debugging, writing-plans, subagent-driven-development]
---

# Test-Driven Development (TDD)

## Overview
Write the test first. Watch it fail. Write minimal code to pass.
...
```

### Frontmatter fields

| Key | Required? | Notes |
|-----|-----------|-------|
| `name` | yes | Lower-kebab-case. ≤64 chars. Must match the directory name. |
| `description` | yes | ≤1024 chars. Shown in `/skills` listing. Used by the model to decide whether to load the skill. |
| `version` | yes | Semver-ish. Curator uses this for upgrade decisions. |
| `author` | recommended | Free-text. |
| `license` | recommended | SPDX id. Hub-installed skills with no license are flagged. |
| `platforms` | optional | List of OS targets, e.g. `[macos, linux]`. Skill is filtered out elsewhere. |
| `compatibility` | optional | agentskills.io compatibility string. |
| `metadata.hermes.tags` | optional | Tags for `/skills search`. |
| `metadata.hermes.related_skills` | optional | Names of related skills. |
| `prerequisites.commands` | optional | Advisory list of CLIs that must be on `PATH`. |
| `prerequisites.python_packages` | optional | Advisory list of Python packages. |
| `config_vars` | optional | Names of `~/.hermes/config.yaml` keys the skill reads (used by `/skills view <name>` to surface them). |
| `conditions` | optional | Boolean expressions evaluated at load time, e.g. `os == "linux"` or `has_env("GITHUB_TOKEN")`. |
| `disabled` | optional | If `true`, the skill is hidden by default. |

The parser is `agent/skill_utils.py:parse_frontmatter()`. It tolerates
both YAML and JSON frontmatter. Empty frontmatter is allowed; the
parser then falls back to inferring the `name` from the directory.

### Body conventions

* **Title (`# Name`)** — first line of body; mirrors `name`.
* **Overview** — one-liner of what the skill does; the model uses this
  to decide if the skill is relevant.
* **When to use** — bullet list of scenarios.
* **Procedure** — numbered steps, often interleaved with shell blocks
  fenced as ```` ```bash ````.
* **References** — at the bottom, links to files under `references/`
  the agent should `read_file` if it needs more detail.

### Inline shell expansion

`agent/skill_preprocessing.expand_inline_shell()` lets a skill embed a
command whose stdout is captured and inlined into the loaded skill
before it reaches the model. Syntax:

```markdown
Here is the current branch: $$ git rev-parse --abbrev-ref HEAD $$
```

When the skill is loaded, `$$ ... $$` blocks are evaluated against the
current shell (the active terminal backend) and replaced with their
output. This is how skills like `git-status-summary` add a fresh git
status without re-prompting.

### Template variable substitution

Skills can reference config values via `{{ config.skills.<name>.<key> }}`
or env vars via `{{ env.NAME }}`. Substitution happens in
`agent/skill_preprocessing.substitute_template_vars()` before the
skill is sent to the model. Unset values raise a clear error rather
than silently producing an empty string.

### Conditions

The `conditions:` frontmatter key is evaluated by
`agent/skill_utils.skill_matches_platform()`. A failing condition
hides the skill from `/skills` and prevents `/skill-name` from loading
it. Examples:

```yaml
conditions:
  - os == "macos"
  - has_env("GITHUB_TOKEN")
  - has_command("ffmpeg")
```

## 4. The skills index

Hermes builds an in-memory skill index every time the agent starts.
The index is a flat `dict[str, SkillRecord]` keyed by skill name. A
`SkillRecord` carries:

* `name`, `description`, `version`, `author`, `license`.
* `tags`, `related_skills`.
* `path` — absolute path to the skill directory.
* `source` — `"builtin"`, `"optional"`, `"user"`, `"hub:<id>"`,
  `"plugin:<name>"`.
* `disabled` — whether this skill is hidden.
* `conditions_ok` — whether the conditions evaluate true on this host.

### Roots

`agent/skill_utils.get_all_skills_dirs()` returns every directory that
might contain skills:

1. **Builtin** — the package's `skills/` directory.
2. **Optional** — `optional-skills/` (only those listed in
   `config.skills.enabled_optional`).
3. **User** — `~/.hermes/skills/`.
4. **Hub** — `~/.hermes/skills/<hub>/<skill>/` directories registered
   in `~/.hermes/skills/.hub/lock.json`.
5. **Plugins** — any `plugins/<name>/skills/` declared in the plugin
   manifest's `provides_skills` field.

`HERMES_OPTIONAL_SKILLS_DIR` overrides the optional-skills root for
packagers (e.g. Nix, Homebrew) that ship the optional skills outside
the wheel.

### Index cache

The Skills Hub publishes a precomputed JSON index of available skills
under `skills/index-cache/<hub>/index.json`. Hermes loads this lazily
when the user runs `/skills browse` so we do not need to hit the
network on every startup.

### Disabled skills

`config.skills.disabled` is a list of names. Anything in it is
filtered out. `/skills disable <name>` writes to this list.

## 5. Loading a skill

The flow when the user types `/test-driven-development run-tdd-on src/foo.py`:

```
hermes_cli/commands.py:resolve_command("test-driven-development")
        │  (returns None — not a CommandDef, so it's a skill)
        ▼
agent/skill_commands._load_skill_payload("test-driven-development")
        │  ─→ reads SKILL.md, parses frontmatter
        │  ─→ runs expand_inline_shell + substitute_template_vars
        │  ─→ checks conditions; surfaces a friendly error if any fail
        ▼
agent/skill_commands._inject_skill_config(payload, skill_dir)
        │  ─→ resolves config_vars and appends a "[Skill config: ...]" block
        ▼
agent/skill_commands._build_skill_message(payload, args, skill_dir)
        │  ─→ wraps the payload in <SKILL ...> tags
        ▼
AIAgent.run_conversation(user_message=<SKILL>...)
```

The skill enters the conversation as a **user message**, not a
system prompt. This is deliberate — placing it in the user role
preserves Anthropic's prompt-cache prefix (the system prompt is the
caching key).

### "Skill view" tool

For agent-initiated invocation, `tools/skills_tool.py` exposes
`skills_list` (tier-1 metadata) and `skill_view(name)` (tier-2 body).
The model loads tier-2 only when it has decided the skill is relevant,
based on the tier-1 description.

## 6. The Skills Hub

Hermes can discover and install skills from external registries via
`tools/skills_hub.py`. The Hub is **source-agnostic** — anything that
can describe and serve a directory tree of skills works. Concrete
sources:

| Source class | Use |
|--------------|-----|
| `OptionalSkillSource` | Built-in `optional-skills/` directory. Trusted; no quarantine. |
| `GitHubSource` | Any GitHub repo via the Contents API. Quarantined and scanned. |
| `HermesCloudSource` | Hosted by Nous (sync target for `hermes skills sync`). |
| `LocalDirectorySource` | A local path the user adds as a tap. |

### Hub state on disk

```
~/.hermes/skills/.hub/
├── lock.json           # provenance for installed skills
├── audit.log           # append-only install/uninstall log
├── taps.json           # registered Hub source URLs
├── quarantine/         # staging area for in-flight installs
└── index-cache/        # cached source indices (timestamped)
```

`lock.json` example:

```json
{
  "skills": {
    "fancy-pdf-tools": {
      "source": "github://NousResearch/community-skills",
      "ref": "v1.2.0",
      "sha": "deadbeef...",
      "installed_at": "2026-04-14T13:11:02Z",
      "license": "MIT",
      "warned_no_license": false
    }
  }
}
```

### Install pipeline

```
hermes skills install gh:NousResearch/community-skills/fancy-pdf-tools
        │
        ▼  GitHubSource.fetch(ref) — clone tarball into quarantine/
        ▼  skills_guard.scan() — see Section 7
        ▼  if scan ok → move to ~/.hermes/skills/<hub>/<skill>/
        ▼  write lock.json entry, append audit.log
```

The agent never sees a Hub-installed skill until it has cleared the
guard.

### Taps

A "tap" is a registered Hub source URL. `hermes skills tap add
https://example.com/community-skills/index.json` adds it; the URL must
serve a JSON index that the loader understands (see
`tools/skills_hub.py:HubLockFile.LOAD`).

## 7. The skills guard

`tools/skills_guard.py` runs on every Hub-installed skill before it
becomes loadable. The scanner looks for:

* **Prompt-injection patterns** — invisible Unicode (zero-width spaces,
  bidi overrides), HTML comments containing `<!--`, prompt-injection
  trigger phrases ("ignore previous instructions", "you are now",
  "sudo", "reveal your system prompt", …).
* **Exfil URLs** — URL shorteners with non-printing characters,
  data-exfil-shaped query strings, fingerprinting beacons.
* **Suspicious shell commands** in inline-shell blocks: `curl |
  bash`, fork bombs, anything that touches `~/.ssh`, `~/.hermes`, or
  `~/.aws`.
* **Token grabs** — calls to `printenv`, `env`, `cat ~/.bashrc` …
* **Unbounded `eval`** — `eval "$(...)"` or Python `exec(open(...).read())`.

A scan that fails moves the skill from `quarantine/` into a
`rejected/` directory and writes a row in `audit.log` with the reason.
The user has to explicitly override (via `hermes skills install --force
<name>`) to install it.

## 8. The skills sync subsystem

`tools/skills_sync.py` is the upload side. It pushes user-curated
skills to the configured Hub (typically Hermes Cloud / Nous Portal).

Sync rules:

* Only skills under `~/.hermes/skills/` (not the bundled ones) are
  candidates.
* Skills marked `metadata.hermes.private: true` are skipped.
* Skills younger than `config.skills.sync.min_age_minutes` (default 5)
  are skipped — gives the curator time to consolidate before pushing.
* The local `version` is compared against the remote; only newer
  versions are pushed.

The sync is one-way (local → remote). Pulling from a remote source is
done via `hermes skills install` (see above).

## 9. The curator

`agent/curator.py` is the autonomous skill maintainer. Highlights:

| Feature | Detail |
|---------|--------|
| Trigger | Inactivity-based: idle ≥ `idle_threshold_seconds` AND last run age ≥ `interval_hours`. |
| Mode | Spawns a forked `AIAgent` with a curated prompt and the skill-management tools. |
| Operations | `pin`, `archive`, `consolidate`, `patch`, `delete`. |
| State | `~/.hermes/skills/.curator_state` (JSON). Load: `_state_file()` / `_default_state()`. |
| Backup | `agent/curator_backup.py` snapshots `~/.hermes/skills/` before each run. |

### State file

```json
{
  "last_run": "2026-04-12T03:00:00Z",
  "last_outcome": "success",
  "session_id": "curator-20260412-030000",
  "summary": "Pinned 2, archived 5, consolidated 3 → 1, patched 1",
  "paused_until": null
}
```

### Configuration

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

### Operations in detail

* **Pin** — add `metadata.hermes.pinned: true`. Pinned skills are
  never archived even if idle.
* **Archive** — move to `~/.hermes/skills/.archived/<name>/`. Hidden
  from `/skills` but recoverable.
* **Consolidate** — pick a representative skill from a near-duplicate
  cluster, merge unique steps from the others, archive the rest.
* **Patch** — rewrite an instruction that the curator's review found
  was contradicted by recent runs (e.g. a CLI flag changed).
* **Delete** — only for skills the curator created itself and that
  have never been used.

The curator never modifies skills shipped by the package or installed
from the Hub; it only touches `~/.hermes/skills/`.

## 10. Skill-as-tool: the `skills_tool` family

Even though skills are data, the agent reaches them through tools:

| Tool | What it does |
|------|--------------|
| `skills_list` | Tier-1 metadata for every available skill. Filtered by toolset, conditions, and the `disabled` list. |
| `skill_view(name)` | Returns the full `SKILL.md` body. |
| `skill_create(...)` | Creates a new skill under `~/.hermes/skills/`. |
| `skill_edit(...)` | Edits an existing skill. |
| `skill_patch(...)` | Targeted edit that preserves frontmatter. |
| `skill_delete(name)` | Move skill to trash. |
| `skill_pin(name)` / `skill_unpin(name)` | Toggle pin. |
| `skill_archive(name)` / `skill_unarchive(name)` | Toggle archive. |

Implementations live in `tools/skill_manager_tool.py` and
`tools/skills_tool.py`.

### Persistence rules

All write tools go through `utils.atomic_yaml_write()` so the skill's
`SKILL.md` is replaced atomically. References under `references/`,
`templates/`, `assets/` are written with `utils.atomic_replace()`.

### Skill usage tracking

`tools/skill_usage.py` records usage signals for the curator:

* When a skill is loaded (timestamp, session id, outcome).
* When a skill emits a tool call that errors.
* Heuristic "did this skill help?" inferred from whether the model
  surfaced its result.

Stored in `~/.hermes/skills/.usage.jsonl`. The curator reads this on
each pass.

## 11. Skill index iteration

`agent/skill_utils.iter_skill_index_files()` walks the index and
yields `SkillRecord` objects. Used by:

* `tools/skills_tool.py` for `skills_list`.
* `cli.py` autocomplete for `/skill-name` completions.
* `gateway/run.py` for cross-platform `/<skill-name>` dispatch.

Iteration is cached for `config.skills.cache_seconds` (default 60).

## 12. Optional skills DESCRIPTION.md

`optional-skills/DESCRIPTION.md` is a top-level overview of which
optional skills exist and what each is for. It is shown by
`hermes skills browse --optional`.

## 13. Plugin-shipped skills

A plugin can ship its own skills under `plugins/<name>/skills/` and
declare them in `plugin.yaml`:

```yaml
provides_skills:
  - relative/path/to/my-skill
```

When the plugin is loaded, its skills are added to the index with
`source = "plugin:<name>"`. They are subject to the same disabled list
and condition filters as built-in skills.

## 14. Authoring a new skill

Quick recipe (also documented in
`skills/software-development/hermes-agent-skill-authoring/SKILL.md`):

1. Pick a category. If the existing categories do not fit, propose a
   new one in the PR.
2. Create the directory: `skills/<category>/<my-skill>/`.
3. Write `SKILL.md` with required frontmatter and a clear "When to use"
   section.
4. Add `references/` files for any details that would bloat `SKILL.md`.
5. Test with `/<my-skill>` in the CLI.
6. Add a test case in `tests/skills/`.
7. PR.

For Hub-distributed skills, the same recipe applies but the skill
lives in your own repo and is published via the Hub URL.

## 15. Worked examples

### A pure-instruction skill

```markdown
---
name: arxiv-search
description: "Find papers on arXiv using Google Scholar / web_extract."
version: 1.0.0
---

# arXiv Search

## When to use
- Finding the latest paper on a topic.

## Procedure
1. Use `web_search` with the query "<topic> arxiv".
2. Pick the top 3 results that are arXiv abstracts.
3. Use `web_extract` on each to get the abstract + authors.
4. Summarise.
```

No code; the model orchestrates the existing tools.

### A skill that runs shell

```markdown
---
name: pre-commit-fixup
description: "Run pre-commit hooks and auto-fix the changes."
version: 1.0.1
prerequisites:
  commands: [git, pre-commit]
---

# Pre-commit fixup

## Procedure
```bash
pre-commit run --files $(git diff --cached --name-only)
git add -A
git commit --amend --no-edit
```
```

The model executes the block via the `terminal` tool.

### A skill that uses inline shell expansion

```markdown
---
name: branch-summary
description: "Show what's on the current branch since main."
version: 1.0.0
---

# Branch summary

Branch: $$ git rev-parse --abbrev-ref HEAD $$
Diverged from main:

```bash
git log main..HEAD --oneline
```
```

The `$$ ... $$` block is captured at load time and inlined into the
text the model sees, so the model has the branch name without needing
to call `terminal` for it.

## 16. Performance and cache implications

Skills are loaded as **user messages**. This has two consequences:

1. They do not invalidate the system-prompt cache (good).
2. They *do* invalidate the user-message cache for that turn (fine —
   that turn is unique anyway).

If a skill is large, prefer breaking it into a small `SKILL.md` plus
files under `references/` that the model can `read_file` only when
needed. This is the *progressive disclosure* pattern in practice.

## 17. Anti-patterns

* **Don't put secrets in `SKILL.md`.** The skill body is shipped to
  the provider on every load. Use `~/.hermes/.env` and `{{ env.X }}`
  substitution.
* **Don't fork an existing skill** when a small patch will do — the
  curator will eventually consolidate them, but you'll churn the
  user.
* **Don't rely on `prerequisites` for security.** The list is
  advisory; the model is free to ignore it. Use a `condition` if you
  need a hard gate.
* **Don't use one skill for many unrelated tasks.** Skills compete for
  the model's attention; small, single-purpose skills win.
* **Don't write skills that auto-`approve` dangerous commands.** The
  approval system is a security boundary; skills must respect it.

## 18. Skill review checklist (for PRs)

* [ ] `name` matches the directory.
* [ ] `description` is ≤1024 chars and accurate.
* [ ] `version` is set.
* [ ] `license` is set if the skill ships under a non-MIT license.
* [ ] No secrets in the body.
* [ ] No `eval` / `curl | bash` / `~/.ssh` access in inline shell.
* [ ] `prerequisites.commands` lists everything the skill assumes.
* [ ] `conditions` gate any platform-specific procedure.
* [ ] At least one test under `tests/skills/`.
* [ ] Documented in the relevant `README.md` if the category has one.
