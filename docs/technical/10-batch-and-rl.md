# 10 — Batch Generation, Trajectory Compression, and RL

This document covers the **research-facing** side of Hermes Agent: the
batch trajectory generator (`batch_runner.py`), the trajectory
compressor (`trajectory_compressor.py`), the SWE-style task runner
(`mini_swe_runner.py`), and the Atropos-based RL environments under
`environments/`.

These pieces let researchers produce large quantities of high-quality
tool-calling trajectories for fine-tuning and RL training without
relying on any cloud-only proprietary harness.

## 1. Mental model

```
                     ┌──────────────────┐
                     │   prompts.jsonl  │   # input dataset
                     └────────┬─────────┘
                              │
                              ▼
                  ┌──────────────────────┐
                  │   batch_runner.py    │   # runs N agents in parallel
                  │   (multiprocessing)  │
                  └────────┬─────────────┘
                              │
                              ▼
                  ┌──────────────────────┐
                  │ trajectories.jsonl   │   # full transcripts
                  └────────┬─────────────┘
                              │
                              ▼
                  ┌──────────────────────┐
                  │ trajectory_compressor│   # post-processes for training
                  └────────┬─────────────┘
                              │
                              ▼
                  ┌──────────────────────┐
                  │ training-ready jsonl │
                  └──────────────────────┘
```

For RL, the chain is similar but the upstream is an **env** in
`environments/`, which produces trajectories online (during rollout)
rather than from a static dataset.

## 2. `batch_runner.py`

`batch_runner.py` is a parallel agent runner using `multiprocessing.Pool`.
Each worker process spawns a fresh `AIAgent` instance, runs a single
prompt to completion, and emits a trajectory.

### CLI surface

```
python batch_runner.py \
    --dataset_file=data.jsonl \
    --batch_size=10 \
    --run_name=my_run \
    --provider=openrouter \
    --model="anthropic/claude-haiku-4-5" \
    --max_iterations=60 \
    --enabled_toolsets="['hermes-cli']" \
    --output_format="hermes" \
    --checkpoint_path="/tmp/my_run.ckpt"
```

`fire` parses these CLI args; the same flags map directly to
`run_one_task()` kwargs.

### Input format

Each line of `--dataset_file` is a JSON object with at minimum:

```json
{
  "task_id": "abc-123",
  "prompt": "Build me a simple TODO app in Python",
  "system": "You are a helpful coding assistant.",
  "metadata": { "difficulty": "easy", "tags": ["python"] }
}
```

Optional fields:

* `model` / `provider` — override the run-level default per row.
* `enabled_toolsets` / `disabled_toolsets` — per-row toolset filter.
* `seed` — used for sampling toolset distributions.

### Distribution sampling

When `--toolset_distribution=research`, `batch_runner` calls
`toolset_distributions.sample_toolsets_from_distribution("research")`
once per row. The returned set is passed as `enabled_toolsets` to the
spawned agent. This produces datasets that mix tool exposure
realistically (e.g. "research" rows use web 90% of the time, terminal
10%).

### Output formats

`--output_format` selects one of three trajectory shapes:

| Format | Use |
|--------|-----|
| `hermes` | Native Hermes JSONL with `from`/`value` pairs and `<tool_call>` / `<tool_response>` XML tags. The shape consumed by `trajectory_compressor.py`. |
| `openai` | OpenAI-style `messages` list with `role` and `tool_calls`. |
| `messages` | Plain `messages` list with `tool_call` events folded into `assistant` messages. |

`hermes` is the canonical shape for RL training and for
`trajectory_compressor.py`. The XML wrapping makes it stable across
provider quirks (Anthropic `<tool_use>` vs OpenAI `tool_calls` vs
Codex `function_call`).

### Worker model

`batch_runner._worker_main()` is the entry. Each worker:

1. Inherits a fresh `AIAgent` factory (instantiated lazily — workers
   don't share state).
2. Pulls a row off the multiprocessing queue.
3. Runs `agent.run_conversation(prompt, system_message=system, ...)`.
4. On success, writes the trajectory to the output file via the
   atomic file lock (`fcntl.flock`) — multiple workers can write
   safely.
5. Updates the checkpoint file (see below).

A failure (exception, provider error, hit `max_iterations`) writes to
`failed_trajectories.jsonl` with the exception class and message.

### Checkpointing

`--checkpoint_path` enables resumable runs. The checkpoint is a JSON
file:

```json
{
  "completed_task_ids": ["abc-123", "abc-124", ...],
  "failed_task_ids": [...],
  "started_at": "2026-04-12T01:00:00Z",
  "version": 1
}
```

On restart with the same `--run_name` + `--checkpoint_path`,
`batch_runner` skips any `task_id` already in `completed_task_ids` so
the run picks up where it left off. The file is updated atomically
after each row using `utils.atomic_json_write()`.

Tests for this are in `tests/test_batch_runner_checkpoint.py`.

### Tool usage stats

Every trajectory carries `tool_call_count` and a `tool_stats` dict.
`batch_runner._extract_tool_stats()` (`batch_runner.py:114`) and
`_normalize_tool_stats()` (`batch_runner.py:60`) compute:

* per-tool count,
* per-tool success/error count,
* per-toolset aggregated count,
* per-tool average runtime,
* per-tool error categories (rate-limit, auth, transient, bad request).

`_extract_tool_stats` is also exposed so external pipelines can use
the same logic on their trajectories.

### Concurrency knobs

| Flag | Default | Notes |
|------|---------|-------|
| `--batch_size` | 10 | Number of worker processes. |
| `--queue_max` | `batch_size * 4` | Max prompts queued ahead of workers. |
| `--per_worker_max_iterations` | `max_iterations` | Per-task tool-call ceiling. |
| `--per_worker_token_budget` | unbounded | Per-task token ceiling. |
| `--retry_failed` | `False` | Retry rows that ended in `failed_trajectories.jsonl`. |
| `--shuffle` | `False` | Shuffle the dataset before queuing. |
| `--max_dataset_rows` | `None` | Cap rows for smoke tests. |

The pool uses `spawn` start method (not `fork`) for cross-platform
parity and to avoid sharing the parent's loaded SDK clients (which
would otherwise corrupt async state).

## 3. `trajectory_compressor.py`

Once raw trajectories exist, they are typically too long to
fine-tune on. The compressor produces shorter, training-ready versions
that preserve the task and the final answer.

### Strategy

* **Protect the boundaries**:
  * first system message,
  * first user message,
  * first assistant turn,
  * first tool result,
  * last `protect_last_n_turns` turns.
* **Summarise the middle** through an auxiliary LLM (typically a
  cheap model like Claude Haiku or Mistral Small via OpenRouter).
* **Re-frame** the summary as if a different assistant produced it
  (the "different assistant" handoff frame from Codex).
* **Iteratively** update the summary across compactions so that
  information from earlier compressions does not get lost in later
  ones.

### `CompressionConfig`

```python
@dataclass
class CompressionConfig:
    target_max_tokens: int = 16000
    summary_target_tokens: int = 1500
    protect_first_system: bool = True
    protect_first_user: bool = True
    protect_first_assistant: bool = True
    protect_first_tool: bool = True
    protect_last_n_turns: int = 4
    summarizer_model: str = "anthropic/claude-haiku-4-5"
    summarizer_provider: str = "openrouter"
    handoff_framing: bool = True
    no_response_preamble: bool = True
    chars_per_token: int = 4
```

`chars_per_token=4` is a coarse approximation used for budgeting
without tokenising — fast and good enough for compression decisions.

### Tool-output pruning

Before sending a chunk to the summariser, very long tool outputs are
replaced with `_PRUNED_TOOL_PLACEHOLDER` (e.g. "[tool output of 12.4
KB pruned for summarisation]"). This prevents giant `cat`-style outputs
from blowing the summariser's context.

### Summary template

```
SUMMARY OF EARLIER CONVERSATION
================================

Resolved questions:
- ...

Pending questions:
- ...

Files touched:
- ...

Plan / next steps:
- ...
```

The compressor generates this format every time so successive
compactions can extend rather than rewrite the prior summary.

### CLI

```
python trajectory_compressor.py \
    --input=data/run_1/ \
    --sample_percent=15 \
    --target_max_tokens=16000 \
    --output=data/run_1.compressed.jsonl
```

`--sample_percent=15` tells the compressor to compress only 15% of
trajectories that exceed the target length (useful for quickly
producing a mixed short+long training set).

### Async path

`tests/test_trajectory_compressor_async.py` exercises the async-flavour
of the compressor that is used by RL environments (which already run
in an asyncio loop). The sync path is for CLI / batch processing.

## 4. `mini_swe_runner.py`

A SWE-bench-style task runner. It is conceptually a lightweight
single-task variant of `batch_runner` aimed at running real
software-engineering tasks (build a feature, fix a bug, write a
test) in an isolated environment.

### CLI

```
python mini_swe_runner.py --task "Create a Python script that prints hello world" --env local
python mini_swe_runner.py --dataset tasks.jsonl --env modal --max_concurrency 4
```

Outputs trajectories in the same Hermes format that
`trajectory_compressor.py` consumes.

### Environment factory

`mini_swe_runner.create_environment(env_name)` (line 120) dispatches
to the right backend:

| `env_name` | Backend |
|------------|---------|
| `local` | Spawn a temporary directory + `BaseExecutionEnvironment` (`local`). |
| `docker` | Per-task ephemeral container. |
| `modal` | Per-task Modal sandbox. |

### Tool definition

`TERMINAL_TOOL_DEFINITION` (line 71) is the full OpenAI tool schema
used for the runner. The schema embeds usage examples directly in the
description so the model needs no extra prompt-engineering — handy
when running across many providers.

## 5. RL environments (`environments/`)

The `environments/` directory ships Atropos-compatible RL environments
that wrap Hermes Agent. They are pulled in by the `[rl]` extra (which
also installs `atroposlib`, `tinker`, `fastapi`, `uvicorn`, `wandb`).

### Layout

```
environments/
├── README.md
├── hermes_base_env.py       # Base env: AIAgent + terminal backend per rollout
├── agent_loop.py            # Loop that drives the env
├── agentic_opd_env.py       # Open Procedural Dialog
├── web_research_env.py      # Web research env
├── hermes_swe_env/          # SWE-bench environment
├── terminal_test_env/       # Synthetic terminal tasks
├── benchmarks/              # Benchmark drivers
├── tool_call_parsers/       # Parsers for non-OpenAI tool-call formats
├── tool_context.py          # Shared tool execution context
└── patches.py               # Patch helpers (apply, revert, validate)
```

### `HermesBaseEnv`

The base class for every environment. Responsibilities:

* Spawn an `AIAgent` per rollout with a fresh terminal backend.
* Provide a `score()` method that maps a finished trajectory to a
  scalar reward.
* Expose Atropos hooks: `reset`, `step`, `done`.

Concrete envs override `score()` and `_make_task()` (which generates
the prompt for one rollout).

### Tool call parsers

Some training models emit tool calls in non-OpenAI formats (XML, JSON
embedded in plain text, custom delimiters). The parsers under
`environments/tool_call_parsers/` translate those back into the shape
that `model_tools.handle_function_call()` expects, so the env can use
the production tool dispatch path unchanged.

### `agentic_opd_env.py`

Open Procedural Dialog environment. The model is given an
under-specified user request and rewarded for asking clarifying
questions early before committing to a procedure. Backed by
`tools/clarify_tool.py`.

### `web_research_env.py`

Web research env with reward shaped by:

* whether the answer cites a primary source,
* whether it correctly answers a held-out evaluation question,
* whether it stays within a token budget.

### `hermes_swe_env/`

SWE-bench-style RL env. Each rollout is a real GitHub issue + repo
checkout in a Docker / Modal container. Reward is `tests-pass` minus a
length penalty.

### `terminal_test_env/`

Synthetic terminal tasks (e.g. "find the file with the most lines under
`/tmp/repo`"). Cheap to generate, useful for shaping early-RL behaviour.

### Patches

`environments/patches.py` provides:

* `apply_patch(diff_text)` — apply a unified diff via `git apply`.
* `revert_patch(diff_text)` — `git apply -R`.
* `validate_patch(diff_text)` — dry-run check.

These are used by `hermes_swe_env/` to verify model-produced patches.

## 6. `rl_cli.py`

Top-level RL helper script (~16 KB). Wraps Atropos training entry
points so users can do:

```
python rl_cli.py train --env=hermes_swe_env --model=Qwen/Qwen2.5-7B --epochs=3
python rl_cli.py eval --env=web_research_env --model=mistralai/Mistral-Small-3.1
python rl_cli.py rollout --env=terminal_test_env --num_rollouts=128
```

Internally it calls `tinker.train`, `tinker.eval`, and
`atroposlib.run_rollout` with environment-specific settings pulled
from `environments/<env>/config.yaml`.

## 7. Datagen config examples

`datagen-config-examples/` and `tinker-atropos/` carry sample
configurations for common training pipelines:

| File | Purpose |
|------|---------|
| `datagen-config-examples/research-bias-balanced.yaml` | Research dataset with balanced toolset distribution. |
| `datagen-config-examples/development-heavy.yaml` | Software-development heavy distribution. |
| `datagen-config-examples/safe-no-terminal.yaml` | No-terminal subset for capability evaluation. |
| `tinker-atropos/.../*.yaml` | Tinker training configs for Hermes envs. |

## 8. Sampling and compressing in one go

`scripts/sample_and_compress.py` runs the full pipeline:

```
python scripts/sample_and_compress.py \
    --input=raw_run/ \
    --output=compressed_run/ \
    --target_max_tokens=16000 \
    --sample_percent=20
```

Internally it:

1. Selects a random `sample_percent` of trajectories.
2. Compresses each via `trajectory_compressor.py`.
3. Writes the compressed JSONL plus a sidecar `.stats.json` with
   per-trajectory original / compressed sizes.

Useful when curating a training set from an existing batch.

## 9. Running on Modal

For datagen at scale, the Modal backend is the cheapest path:

```
python batch_runner.py \
    --dataset_file=tasks.jsonl \
    --batch_size=64 \
    --terminal_env=modal \
    --provider=openrouter \
    --model="x-ai/grok-4-fast" \
    --output=runs/run_$(date +%Y%m%d).jsonl
```

Each worker spawns a Modal sandbox per task; sandboxes hibernate
between tasks. `scripts/kill_modal.sh` is the convenience cleanup
script for stray sandboxes.

## 10. Atropos compatibility

Hermes envs implement the Atropos `Environment` protocol so they
plug into Atropos-aware trainers (`tinker`, `atroposlib`) without a
custom adapter. The contract is:

```python
class Environment(Protocol):
    def reset(self, seed: int | None = None) -> Observation: ...
    def step(self, action: Action) -> tuple[Observation, float, bool, dict]: ...
```

`HermesBaseEnv` packages an `AIAgent` rollout into one `step()` per
turn — so a single-turn task is one step, multi-turn tasks expand to
many steps.

## 11. Trajectory format gotchas

* **Tool calls inside `assistant` turns** — the Hermes format puts
  `<tool_call>` blocks inside the assistant message, separated by
  newlines. Many other formats put them in their own messages.
* **Reasoning tokens** — encoded inline as `<think>...</think>` tags
  by `agent.trajectory.convert_scratchpad_to_think()`. The compressor
  preserves these blocks unless explicitly told not to.
* **Token counts** — recorded per-turn via the provider's reported
  usage; do not recompute via tokenisers in downstream code (different
  providers use different tokenisers).

## 12. Where to look when…

| Scenario | Where |
|----------|-------|
| Workers hang on cold start | check the SDK lazy-import path in `run_agent.py:75-90` |
| Trajectory output is missing tool calls | check `agent/trajectory.save_trajectory()` and the model's tool-call emission shape; the parser may be in `environments/tool_call_parsers/` |
| Compression keeps blowing the summariser | drop `summary_target_tokens` and/or `target_max_tokens`, or switch summariser model |
| Modal sandboxes leak | use `scripts/kill_modal.sh`, then `mini_swe_runner.create_environment` reuses them less |
| Atropos training accuracy plateaus | inspect `score()` distribution per env in `wandb`; rebalance with toolset distributions |

## 13. Example end-to-end

```
# 1. Generate raw trajectories
python batch_runner.py \
    --dataset_file=data/easy_tasks.jsonl \
    --batch_size=16 \
    --provider=openrouter \
    --model="anthropic/claude-haiku-4-5" \
    --output=runs/easy_raw.jsonl

# 2. Compress
python trajectory_compressor.py \
    --input=runs/easy_raw.jsonl \
    --output=runs/easy_compressed.jsonl \
    --target_max_tokens=12000

# 3. Train with tinker
python rl_cli.py train \
    --env=hermes_swe_env \
    --train_data=runs/easy_compressed.jsonl \
    --base_model=Qwen/Qwen2.5-7B \
    --epochs=3 \
    --wandb_project=hermes-rl
```
