# 22 — Delegation, Sub-agents, and Mixture of Agents

This chapter covers the three Hermes mechanisms for spawning extra
agents: classic delegation (`tools/delegate_tool.py`), mixture of
agents (`tools/mixture_of_agents_tool.py`), and clarification
(`tools/clarify_tool.py`). All three let a single root conversation
fan out work; they differ in topology and reward.

## 1. Why delegate?

Three motivations:

1. **Context window economy** — a sub-agent can do 30 turns of
   exploration and return one paragraph; the parent's context only
   pays for the paragraph.
2. **Tool restriction** — a sub-agent can be locked to a narrow tool
   set (e.g. read-only web tools) so an untrusted task cannot escalate.
3. **Parallelism** — multiple sub-agents can run concurrently when
   the parent is using the mixture-of-agents tool.

## 2. Classic delegation

`tools/delegate_tool.py` — ~107 KB. Exposes `delegate(...)` to the
agent.

### Schema

```json
{
  "name": "delegate",
  "description": "Spawn a sub-agent to handle a focused sub-task.",
  "parameters": {
    "type": "object",
    "properties": {
      "task": {"type": "string", "description": "The instruction the sub-agent receives."},
      "model": {"type": "string", "description": "Override model for the sub-agent."},
      "enabled_toolsets": {"type": "array", "items": {"type": "string"}},
      "disabled_tools":   {"type": "array", "items": {"type": "string"}},
      "max_iterations":   {"type": "integer", "default": 30},
      "personality":      {"type": "string"},
      "scope":            {"type": "string", "enum": ["isolated", "share_files", "share_session"]}
    },
    "required": ["task"]
  }
}
```

### Mechanics

1. The parent spawns a fresh `AIAgent` with the requested model,
   toolset, and personality.
2. The parent's `CredentialPool` is shared by reference (no
   re-auth).
3. A new `session_id` is allocated with `parent_session_id =
   <parent's session id>`.
4. The sub-agent runs to completion (or `max_iterations`).
5. Its final text is returned to the parent as the `delegate` tool
   result.
6. Both sessions persist to `SessionDB`.

### Scope levels

* `isolated` (default) — the sub-agent gets a fresh terminal cwd,
  fresh memory, no inherited state.
* `share_files` — the sub-agent sees the parent's terminal cwd
  (working dir, env). Files written by the sub-agent are visible to
  the parent.
* `share_session` — the sub-agent shares the parent's session id.
  Rare; mostly for "let a smaller model handle this turn for me"
  patterns. Be aware that the parent will see the sub-agent's tool
  calls in its own transcript.

### Configuration

```yaml
delegation:
  enabled: true
  max_depth: 3                  # cap on nested delegation chains
  inherit_credentials: true
  default_max_iterations: 30
  scope_to_parent_toolset: true # sub-agent's toolset is a subset of the parent's
```

`max_depth: 3` prevents pathological recursive delegation
(grand-grand-grandchild). The check is enforced at delegate-tool
dispatch time.

`scope_to_parent_toolset: true` ensures a sub-agent never gets a tool
the parent could not access — a cheap defence against "the model uses
delegation to escape its sandbox".

### Trace propagation

The sub-agent's logger uses
`set_session_context(<sub_session_id>, source="delegated")`, so log
lines are filterable by sub-session. The Langfuse plugin (if enabled)
emits a span hierarchy: delegate → sub-agent → tool calls.

### Worked example

```
parent: "Find the GitHub repo for the open-source library 'foo' and
         summarise its latest release."

parent calls delegate(
    task="Find github.com/foo and summarise its latest release",
    enabled_toolsets=["web", "search"],
    max_iterations=20,
)

sub-agent runs:
  web_search → "github foo"
  web_extract on the top result
  read the changelog
  produce summary
sub-agent returns: "<summary>"

parent calls delegate(
    task="Verify the summary against the README of the same repo",
    enabled_toolsets=["web"],
    max_iterations=10,
)

sub-agent verifies → "<correction or confirmation>"

parent assembles final answer.
```

The parent's context only paid for the two summaries plus its own
reasoning — not the 20 tool calls each sub-agent made.

## 3. Mixture of Agents (MoA)

`tools/mixture_of_agents_tool.py` is the multi-agent ensemble.

### Idea

Run **N** independent agents on the same prompt, then either:

* **Vote** — pick the answer most agents agree on.
* **Aggregate** — feed all N answers to an aggregator agent that
  produces a synthesised final answer.

This is most useful for difficult reasoning tasks where ensembling
across diverse models or temperatures lifts accuracy.

### Schema

```json
{
  "name": "mixture_of_agents",
  "description": "Run a panel of agents on the same prompt.",
  "parameters": {
    "type": "object",
    "properties": {
      "prompt": {"type": "string"},
      "panel": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "model": {"type": "string"},
            "personality": {"type": "string"},
            "temperature": {"type": "number"}
          },
          "required": ["model"]
        }
      },
      "aggregator": {"type": "string"},
      "mode": {"type": "string", "enum": ["vote", "aggregate"], "default": "aggregate"}
    },
    "required": ["prompt", "panel"]
  }
}
```

Example panel:

```json
{
  "panel": [
    {"model": "anthropic:claude-opus-4-7"},
    {"model": "openrouter:openai/gpt-5.2"},
    {"model": "openrouter:moonshotai/kimi-k2"},
    {"model": "openrouter:deepseek/deepseek-r1-0528"}
  ],
  "aggregator": "anthropic:claude-opus-4-7"
}
```

### Concurrency

The panel runs in parallel via `asyncio.gather()` over individual
sub-agents. Each panel member is a real `AIAgent` instance with its
own toolset (defaults to a small subset that does not include
delegation — a panel member cannot recursively delegate). Cancellation
propagates: if the parent cancels, every panel member is interrupted.

### Cost

MoA is expensive — N model calls per turn. Use only for the hard
turns; pair with `auxiliary` config to keep each member cheap.

### Failure modes

* **Diverging answers in vote mode** — the tool returns the
  plurality; ties produce the alphabetically first answer (deterministic
  but arbitrary). Switch to `aggregate` mode if ties are common.
* **One panel member errors** — error is treated as "no vote" or
  "no contribution to aggregator". The tool succeeds as long as at
  least one member produces an answer.
* **Aggregator hits context limit** — aggregator's context is
  budgeted by the same context engine; if the inputs are too long,
  truncation happens before aggregation.

## 4. Clarify

`tools/clarify_tool.py` — `clarify(question, options=...)`.

Not a sub-agent, but conceptually similar: it pauses the agent and
asks the user a question. The user answers; the agent resumes with
the answer in its context.

### Schema

```json
{
  "name": "clarify",
  "description": "Ask the user a clarifying question and wait for a reply.",
  "parameters": {
    "type": "object",
    "properties": {
      "question": {"type": "string"},
      "options":  {"type": "array", "items": {"type": "string"}},
      "default":  {"type": "string"}
    },
    "required": ["question"]
  }
}
```

### Surfaces

| Surface | Render |
|---------|--------|
| CLI | prompt_toolkit modal with optional choices. |
| TUI | `clarify.request` event → modal in Ink. |
| Gateway | sends question over the platform; user replies in chat. |
| ACP | `session/request_user_input` event to the editor. |

The agent loop is suspended until the user answers (or the
gateway-timeout fires, in which case `clarify` returns `{"error":
"timeout"}`).

### When to use

`clarify` is the right tool when:

* The user's request is ambiguous and a wrong guess is costly.
* The agent needs a value the user has not provided (path, filename,
  threshold).
* A long-running task hits a fork in the road that needs human
  judgement.

It is **not** the right tool when:

* The agent could pick a sensible default and let the user `/undo`.
* The information is in `MEMORY.md` or a recent message — the agent
  should re-read first.

## 5. The `agentic_opd_env`

`environments/agentic_opd_env.py` is an RL environment that rewards
the model for using `clarify` early when the user's task is
under-specified. It penalises late or unnecessary clarifications.

This is one of the more interesting trainable signals in the repo —
the model has to learn the tradeoff between asking too much and
guessing wrong.

## 6. Sub-agent observability

| Surface | Where |
|---------|-------|
| Active sub-agents listing | `/agents` (alias `/tasks`) slash command. |
| Per-sub-agent transcript | `SessionDB.get_messages(sub_session_id)`. |
| Wall-clock + token cost | `SessionDB.get_session(sub_session_id)`. |
| Live stream | gateway `stream_consumer` events tagged with the sub-session id. |
| Langfuse spans | parent → child relation preserved when the plugin is enabled. |

`/agents` is interactive — each running sub-agent is listed with
its session id, model, and elapsed time. Selecting one shows the
in-flight tool calls.

## 7. Cancellation propagation

A `Ctrl+C` (or `/stop`) at the parent's level propagates to every
sub-agent and every panel member:

```
parent: tools.interrupt.set()
  → every sub-agent's loop sees interrupt.is_set() at next safe boundary
  → raises InterruptException
  → sub-agent loop terminates
  → parent's delegate tool returns {"error": "interrupted"}
```

The MoA tool surfaces this as `{"error": "interrupted",
"completed_panel": [...]}` so the model can decide whether to
retry, fall back, or surface the error to the user.

## 8. Stress patterns

`tests/stress/test_delegation_stress.py` exercises:

* 100 concurrent sub-agents.
* 3-deep nested delegation chains (within `max_depth`).
* MoA with 8 panel members.
* Cancellation under load.

These tests are slow (~30s each); they run in the nightly suite, not
the per-PR matrix.

## 9. Sub-agent tool gating

The parent decides which tools its child sees:

* `enabled_toolsets` — explicit allow-list (subset of parent's).
* `disabled_tools` — explicit deny-list within the chosen toolsets.
* `delegation.scope_to_parent_toolset: true` — implicit cap.

A common pattern for security:

```
parent toolset:        ["hermes-cli"]
delegate enabled:      ["web", "search", "vision"]   # no terminal!
```

The sub-agent cannot escalate to terminal access even if its
instructions try to.

## 10. Memory propagation

By default, sub-agents do **not** inherit the parent's memory. This
is deliberate:

* It prevents secrets in `USER.md` from leaking to a sub-agent that
  may share its trajectory in training data.
* It keeps sub-agent token counts low — memory blocks can be large.

Override with `share_memory: true` on the `delegate(...)` call. In
practice this is rarely needed.

## 11. Trajectory shape

A delegated turn appears in the parent's trajectory as:

```jsonl
{"from":"gpt","value":"<tool_call>delegate(task='...', ...)</tool_call>"}
{"from":"tool","value":"<tool_response>{ \"result\": \"<sub-agent's answer>\" }</tool_response>"}
```

The sub-agent's full transcript lives in its own session row in
`SessionDB`. `batch_runner.py --include_sub_sessions` flag includes
them in the dataset; without the flag, only the parent's row is
exported.

## 12. MoA aggregator prompt

The aggregator gets a prompt like:

```
Several agents independently produced answers to the same question.
Consolidate them into a single answer that is at least as good as the
best of them. Resolve disagreements by reasoning explicitly. Cite
which agent contributed which point.

QUESTION:
<the original prompt>

AGENT 1 (model: anthropic:claude-opus-4-7) ANSWER:
<answer 1>

AGENT 2 (model: openrouter:openai/gpt-5.2) ANSWER:
<answer 2>

...

YOUR ANSWER:
```

Reviewers can override the prompt template via
`config.delegation.moa.aggregator_prompt`.

## 13. Comparison

| Mechanism | Topology | Cost | When to use |
|-----------|----------|------|-------------|
| `delegate` | parent → child (serial) | 1 extra agent | Off-load focused sub-tasks; restrict tool surface. |
| `mixture_of_agents` | parent → fan-out → aggregator | N+1 extra agents | High-stakes reasoning where ensembling helps. |
| `clarify` | agent ↔ user | 0 extra agents | When the model needs human input. |

Roughly: delegate for "I need to do this work in isolation", MoA for
"I need a more reliable answer", clarify for "I need the human".

## 14. Antipatterns

* **Delegating trivial work** — single-tool tasks belong inline.
  Delegation has overhead.
* **Recursive `delegate` to the same model** — does not improve
  quality. Use MoA with diverse models instead.
* **MoA on every turn** — too expensive; pick selectively.
* **Clarify in a loop** — if the agent keeps asking and the user
  keeps not knowing, the agent should propose options or commit to a
  default. The `agentic_opd_env` reward shape penalises this.
* **Sharing the parent's terminal in sub-agents that should be
  isolated** — defeats the purpose of restriction.

## 15. Where to look when…

| Symptom | Where |
|---------|-------|
| `delegate` returns "max depth exceeded" | check `delegation.max_depth` |
| Sub-agent never returns | inspect `/agents`; usually a stuck tool call. Use `tool.cancel`. |
| MoA produces same answer for every panel member | panel is too homogeneous; vary models / temperatures |
| Aggregator answers ignore some panel members | aggregator hit context limit; lower per-member output budget |
| Clarify never delivers question | gateway not running, or platform is non-interactive |
| Cancellation does not propagate | tool blocking on a synchronous call without interrupt-check; fix the tool |
| Sub-agent cost surprises the user | cost panel only counts the parent by default; add `--include_sub` flag in `/usage` |
