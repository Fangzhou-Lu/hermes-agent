# 15 — Data Flow Diagrams

This document collects ASCII sequence diagrams for the most common
data flows in Hermes Agent. Each diagram is annotated with the
relevant module / file paths so you can jump to the code.

## 1. CLI turn (happy path)

```
user types message ↓
                    │
hermes_cli/        ┌──┴───────────────┐
  main.py:main()   │  HermesCLI       │
                   │  (cli.py)         │
                   │                   │
                   │  on input          │
                   │   ▼                │
                   │  resolve_command   │
                   │   ▼                │
                   │  process_command   │
                   │   │                │
                   │   ▼ (regular text) │
                   │  AIAgent.chat()    │
                   │                   │
                   └──┬────────────────┘
                      │
                      ▼
   ┌──────────────────────────────────────────────────┐
   │   AIAgent.run_conversation() (run_agent.py)      │
   │                                                   │
   │   1. memory.queue_prefetch_all()                 │
   │   2. context_engine.should_compress_preflight()  │
   │   3. transport.build_kwargs(messages, tools)     │
   │   4. provider.complete()  ──────────────────► provider API
   │   5. transport.normalize_response(raw)           │
   │   6. for tc in nr.tool_calls:                    │
   │        result = handle_function_call(...)        │
   │        messages.append(tool_result(...))         │
   │   7. context_engine.update_from_response(nr)     │
   │   8. if not nr.text: goto 3                      │
   │   9. memory.sync_all(transcript)                 │
   │  10. SessionDB.persist(messages, usage)          │
   │                                                   │
   └────────────────────┬──────────────────────────────┘
                        │
                        ▼ assistant text
                ┌───────┴────────┐
                │  callbacks.py  │ render to terminal
                └────────────────┘
```

## 2. Gateway message (Telegram)

```
Telegram update ↓
                │
          ┌─────┴────────────────────────┐
          │ TelegramAdapter              │
          │ (gateway/platforms/telegram) │
          │                              │
          │ on_update(...)               │
          │  └─▶ MessageEvent            │
          └─────┬────────────────────────┘
                │
                ▼
      ┌─────────────────────────────────┐
      │ GatewayRunner._handle_message() │ gateway/run.py
      │                                  │
      │ 1. plugin hooks: before_message  │
      │ 2. SessionSource.resolve(event)  │  (chat_id → session_id)
      │ 3. dispatch slash if any         │
      │ 4. fetch / spawn AIAgent         │  (LRU cache, 128 max, 1h TTL)
      │ 5. agent.chat(event.text)        │
      │ 6. plugin hooks: after_dispatch  │
      └─────┬──────────────────────────────┘
                │
                ▼ assistant reply
      ┌────────────────────────┐
      │ DeliveryRouter.deliver │ gateway/delivery.py
      │   target = "origin"    │
      │   → TelegramAdapter    │
      │      .send_message()   │
      └────────────────────────┘
                │
                ▼
        Telegram outbound API
```

## 3. Tool call dispatch

```
provider response with tool_call ↓
                                 │
                ┌────────────────┘
                │
                ▼
     ┌──────────────────────────────┐
     │ AIAgent loop body            │ run_agent.py
     │                               │
     │ for tc in nr.tool_calls:      │
     │   ↓                            │
     │ model_tools.handle_function_  │
     │   call(name, args, task_id…) │ model_tools.py
     │   ↓                            │
     │ tools.registry.lookup(name)   │ tools/registry.py
     │   ↓                            │
     │ check_fn() guard              │
     │ requires_env guard            │
     │   ↓                            │
     │ guardrails (tool_guardrails)  │
     │   ↓                            │
     │ approval gate                 │ tools/approval.py
     │   ↓ (approved)                 │
     │ handler(args, **kw)           │ tools/<tool>.py
     │   ↓                            │
     │ output truncation             │ tools/tool_output_limits
     │   ↓                            │
     │ result JSON string            │
     │   ↓                            │
     │ messages.append(tool result)  │
     └──────────────────────────────┘
```

## 4. Async tool from sync agent loop

```
agent loop (sync)  ──►  model_tools._run_async(coro)
                            │
                            ▼
              _get_tool_loop() → main thread persistent loop
              _get_worker_loop() → thread-local persistent loop (workers)
                            │
                            ▼
              loop.run_until_complete(coro)
                            │
                            ▼
                     async tool returns
                            │
                            ▼
              return value as JSON string
```

The persistent-loop trick is what stops `httpx.AsyncClient` and
`AsyncOpenAI` from raising `Event loop is closed` on the next call —
the loop is reused across all tool dispatches on that thread.

## 5. Context compression

```
turn N completes, large context ↓
                                │
           context_engine.update_from_response(nr)
                                │
                                ▼
           context_engine.should_compress() → True
                                │
                                ▼
     ┌────────────────────────────────────┐
     │ ContextCompressor.compress()       │ agent/context_compressor.py
     │                                     │
     │ 1. partition messages:              │
     │    head = first system + first user │
     │           + first assistant         │
     │           + first tool result       │
     │    tail = last `protect_last_n`     │
     │    middle = everything else         │
     │                                     │
     │ 2. prune large tool outputs in      │
     │    middle → _PRUNED_TOOL_PLACEHOLDER│
     │                                     │
     │ 3. summarize middle via             │
     │    auxiliary_client.call_llm()      │
     │                                     │
     │ 4. wrap summary in SUMMARY template │
     │    (Resolved / Pending / Files /    │
     │     Plan)                            │
     │                                     │
     │ 5. produce new sequence:            │
     │    head ++ [summary] ++ tail        │
     └────────────────┬───────────────────┘
                      │
                      ▼
        SessionDB writes a new session row
        with parent_session_id = original
                      │
                      ▼
        agent loop continues with the
        compressed message list
```

## 6. Credential rotation

```
provider call returns 429 / 401 ↓
                                │
                                ▼
        agent/error_classifier.classify(err)
                                │
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                  ▼
       transient         rate-limit            auth
       (retry)           (sleep then retry)    (rotate creds)
              │                 │                  │
              │                 │                  ▼
              │                 │      cred.last_status = STATUS_EXHAUSTED
              │                 │      cred.last_error_reset_at = now+TTL
              │                 │                  │
              ▼                 ▼                  ▼
        retry with same         sleep until        pool.next() rotates
        cred                    cred.reset_at      to next cred per
                                                   strategy
                                                   │
                                                   ▼
                                          all creds exhausted?
                                                   │
                                          no ──►  use it
                                          yes ──► return cred with
                                                  earliest reset_at
                                                  + log warning
```

## 7. Cron job execution

```
gateway runs in foreground
   │
   ▼  every 60s, in a background thread
cron.scheduler.tick()                   cron/scheduler.py
   │
   ├─► acquire ~/.hermes/cron/.tick.lock  (fcntl)
   │
   ├─► load jobs.json                      cron/jobs.py
   │
   ├─► for each due job:
   │      │
   │      ▼
   │    _build_job_prompt(job)
   │    spawn AIAgent with:
   │      enabled_toolsets = job.toolsets or cron.default_toolset
   │      model            = job.model     or main config
   │      personality      = job.personality
   │    agent.chat(prompt) → result
   │      │
   │      ▼
   │    _resolve_delivery_targets(job) →  [{platform: 'telegram', chat_id: '12345'}, …]
   │      │
   │      ▼
   │    DeliveryRouter.deliver(target, result)
   │      │
   │      ▼
   │    write ~/.hermes/cron/output/<job_id>/<ts>.md
   │
   └─► release lock
```

## 8. Skill invocation

```
user types /test-driven-development run-tests ↓
                                              │
                                              ▼
              hermes_cli/commands.resolve_command("test-driven-development")
                              ↓ returns None (it's a skill, not a CommandDef)
                              ▼
              cli.HermesCLI._handle_skill_command("test-driven-development", "run-tests")
                              ▼
              agent/skill_commands._load_skill_payload("test-driven-development")
                              │
                              ├─► read SKILL.md
                              ├─► parse_frontmatter()
                              ├─► expand_inline_shell()
                              ├─► substitute_template_vars()
                              └─► check conditions
                              ▼
              agent/skill_commands._inject_skill_config(payload, skill_dir)
                              │
                              └─► append "[Skill config: ...]" if config_vars
                              ▼
              agent/skill_commands._build_skill_message(payload, "run-tests")
                              │
                              └─► <SKILL name="...">…</SKILL>
                              ▼
              AIAgent.run_conversation(user_message=<SKILL>...)
                              ▼
              normal turn proceeds
```

## 9. Provider transport selection

```
AIAgent.__init__(provider="openrouter", model="anthropic/claude-haiku-4-5")
                              │
                              ▼
            agent/credential_pool.resolve(provider, model)
                              │
                              └─► PooledCredential
                              ▼
            decide api_mode based on provider:
              openai-compatible → "chat_completions"
              anthropic native  → "anthropic_messages"
              codex / openai responses → "codex_responses"
              bedrock           → "bedrock_converse"
              gemini native     → "chat_completions"
                                  + ChatCompletionsTransport detection
                                    (_is_gemini_openai_compat_base_url)
                                  → may swap to gemini_native_adapter
                              ▼
            agent/transports.<api_mode>.<Transport>()
                              │
                              ▼
            self.transport = transport_instance
                              ▼
            self.transport.build_kwargs(...)  used per turn
```

## 10. Memory provider lifecycle

```
session start
   │
   ▼
MemoryManager.add_provider(BuiltinMemoryProvider())
MemoryManager.add_provider(<external>)            (≤ 1 external)
   │
   ▼
provider.is_available() ── False ──► provider skipped
   │ True
   ▼
provider.initialize()
   │
   ▼

per turn:
   pre-turn:
     for p in providers: p.prefetch()           (sync, blocking allowed)
     OR p.queue_prefetch_all()                  (async, non-blocking)

   build system prompt:
     for p in providers: p.system_prompt_block()
     concatenate into prompt template

   model decides to call a memory tool:
     for p in providers:
       if name in p.get_tool_schemas():
         result = p.handle_tool_call(name, args)
         break

   post-turn:
     for p in providers: p.sync_turn(transcript)

session end:
   for p in providers: p.shutdown()
```

## 11. Streaming response render (CLI)

```
provider stream chunk ↓
                      │
                      ▼
   transport.normalize_chunk(raw) → NormalizedChunk
                      │
                      ▼
  AIAgent dispatches per chunk type:
        text_delta   → callbacks.on_text(...)
        tool_start   → callbacks.on_tool_start(...)
        tool_delta   → callbacks.on_tool_delta(...)
        tool_end     → callbacks.on_tool_end(...)
        reasoning    → callbacks.on_reasoning(...)
        finish       → break
                      │
                      ▼
   hermes_cli/callbacks.py
        │
        ├─► strip reasoning tags (cli._strip_reasoning_tags)
        ├─► route to display.KawaiiSpinner for indicator
        ├─► route to ┊ activity feed for tool output
        └─► prompt_toolkit print_formatted_text
                      │
                      ▼
                    terminal
```

## 12. Approval flow

```
agent decides to run terminal("rm -rf node_modules")
                      │
                      ▼
   tools/registry.dispatch("terminal", {"cmd": "rm -rf node_modules"})
                      │
                      ▼
   tools/approval.requires_approval(call) → True
                      │
                      ▼
   ┌───── CLI ─────────────────┐    ┌───── gateway ─────────────┐    ┌───── ACP ────────────────┐
   │  prompt_toolkit modal      │    │ tools/slash_confirm.py    │    │ acp_adapter/permissions  │
   │  → user types y/a/s/d      │    │ → /approve / /deny over   │    │   sends                   │
   │                            │    │   the platform             │    │   session/request_       │
   │  result returned to        │    │                            │    │   permission              │
   │  approval gate             │    │                            │    │                           │
   └─────────────┬──────────────┘    └─────────────┬──────────────┘    └───────────┬───────────────┘
                 │                                  │                              │
                 ▼                                  ▼                              ▼
                          tools/approval.record_decision(call, choice)
                                              │
                                              ▼
                  if denied → return {"error": "denied by user"}
                  else      → handler runs, result returned
```

## 13. SessionDB write

```
agent loop completes a turn
                │
                ▼
   SessionDB.persist_turn(session_id, messages_added, usage_delta)
                │
                ▼
   for msg in messages_added:
     INSERT INTO messages (session_id, role, content, …) VALUES (…)
                │
                ▼
   UPDATE sessions SET
       message_count    = message_count + N,
       tool_call_count  = tool_call_count + M,
       input_tokens     = input_tokens + …,
       output_tokens    = output_tokens + …,
       cache_read       = …,
       cache_write      = …,
       reasoning_tokens = …,
       ended_at         = ?
   WHERE session_id = ?
                │
                ▼
   FTS triggers populate messages_fts and messages_fts_trigram
                │
                ▼
   if WAL contention:
     retry with random jitter (15 retries, 20–150ms each)
                │
                ▼
   commit
```

## 14. Compressor → new session split

```
compress fires on session "abc-123"
                │
                ▼
   compressor produces N new messages (head + summary + tail)
                │
                ▼
   SessionDB.create_session(parent_session_id="abc-123",
                            source=…, model=…) → "abc-456"
                │
                ▼
   SessionDB.persist_turn("abc-456", new_messages, usage_zero)
                │
                ▼
   AIAgent.session_id = "abc-456"
   subsequent turns append to "abc-456"
                │
                ▼
   on /history, /resume, /insights:
     SessionDB walks parent chain so the user can still see the
     verbatim original turns
```

## 15. Curator inactivity wake-up

```
agent idle for `idle_threshold_seconds`
                │
                ▼
   on next safe boundary (between turns):
     curator.maybe_run()
                │
                ▼
   read state file ~/.hermes/skills/.curator_state
   check (now - last_run) >= interval_hours
                │
                ▼ (yes)
   take backup → ~/.hermes/skills/.backups/<ts>/
                │
                ▼
   spawn forked AIAgent with curator system prompt
     toolset = skills tools + memory tools
     model   = config.curator.prefer_models
                │
                ▼
   forked agent runs:
     skills_list → tier-1 metadata for everything
     for clusters of similar skills:
        consolidate → patch → archive
     write new state, exit
                │
                ▼
   curator.record_outcome(...) → state file
                │
                ▼
   resume normal user-facing agent
```

## 16. ACP session lifecycle

```
editor invokes hermes-acp
                │
                ▼
   acp_adapter/entry.main()
     → load .env, set up stderr-only logging
     → server.run()
                │
                ▼
   editor sends initialize → returns capabilities
                │
                ▼
   editor sends sessions/new → SessionManager.create_session()
                │
                ▼
   editor sends sessions/prompt → AIAgent.run_conversation(...)
     stream events back:
       session/update         text deltas
       session/tool_progress  tool start/output/complete
       session/request_permission  on approval
                │
                ▼
   editor sends sessions/cancel → SessionManager.cancel(session_id)
                │
                ▼
   editor closes pipe → server.shutdown()
```

## 17. Failure → fallback provider

```
primary provider call fails (5xx after retries, or quota exhausted)
                │
                ▼
   credential_pool tries next credential for primary provider
                │
                ▼ none healthy
   fallback_providers = [openrouter:anthropic/...]
                │
                ▼
   AIAgent re-runs the same turn against the next fallback
     (transport may switch from anthropic_messages to chat_completions)
                │
                ▼
   on success → continue
   on all fallbacks failing → surface error to user; agent loop ends
                              with a clear failure message
```

## 18. Trajectory save

```
AIAgent.run_conversation() with save_trajectories=True
                │
                ▼ per turn
   agent/trajectory.save_trajectory(turn_dict, success_or_failure_path)
                │
                ▼
   convert <REASONING_SCRATCHPAD> → <think>
                │
                ▼
   serialize to Hermes JSONL format:
     {"from": "human",     "value": "..."},
     {"from": "gpt",       "value": "<tool_call>..."},
     {"from": "tool",      "value": "<tool_response>..."},
     {"from": "gpt",       "value": "..."}
                │
                ▼
   append (newline-delimited) to:
     trajectory_samples.jsonl       (success)
     failed_trajectories.jsonl      (failure)
                │
                ▼
   downstream batch_runner / trajectory_compressor consume same format
```

## 19. Plugin load

```
process startup
                │
                ▼
   plugins.discover()
     scan plugins/ and ~/.hermes/plugins/
     for each subdir:
       parse plugin.yaml
       if config.plugins.enabled.<name> is truthy:
         install pip_dependencies (lazy: only when first imported)
         import the plugin's __init__.py
         hook callbacks register themselves into gateway/hooks
         provided tools register themselves with tools.registry
         provided skills register themselves with the skill index
                │
                ▼
   on /reload:
     repeat the discovery (idempotent for already-loaded plugins)
```

## 20. Where each diagram's code lives

| Diagram | Primary file(s) |
|---------|------------------|
| 1 — CLI turn | `cli.py`, `run_agent.py`, `model_tools.py`, `hermes_cli/callbacks.py` |
| 2 — Gateway message | `gateway/run.py`, `gateway/platforms/telegram.py`, `gateway/delivery.py` |
| 3 — Tool dispatch | `model_tools.py`, `tools/registry.py`, `tools/approval.py` |
| 4 — Async bridging | `model_tools.py:_get_tool_loop`, `_get_worker_loop`, `_run_async` |
| 5 — Compression | `agent/context_compressor.py`, `agent/auxiliary_client.py` |
| 6 — Credential rotation | `agent/credential_pool.py`, `agent/error_classifier.py` |
| 7 — Cron | `cron/scheduler.py`, `cron/jobs.py`, `gateway/delivery.py` |
| 8 — Skill invocation | `agent/skill_commands.py`, `agent/skill_preprocessing.py` |
| 9 — Transport selection | `run_agent.py:AIAgent.__init__`, `agent/transports/*` |
| 10 — Memory lifecycle | `agent/memory_manager.py`, `agent/memory_provider.py` |
| 11 — Streaming render | `hermes_cli/callbacks.py`, `agent/display.py` |
| 12 — Approval | `tools/approval.py`, `tools/slash_confirm.py`, `acp_adapter/permissions.py` |
| 13 — SessionDB write | `hermes_state.py:SessionDB` |
| 14 — Compressed-session split | `hermes_state.py`, `agent/context_compressor.py` |
| 15 — Curator | `agent/curator.py`, `agent/curator_backup.py` |
| 16 — ACP lifecycle | `acp_adapter/entry.py`, `server.py`, `session.py` |
| 17 — Fallback provider | `run_agent.py`, `agent/credential_pool.py` |
| 18 — Trajectory save | `agent/trajectory.py`, `batch_runner.py`, `trajectory_compressor.py` |
| 19 — Plugin load | `plugins/__init__.py`, plugin `__init__.py` files |
