# PRD: Async Subagent Orchestration

**Status:** Draft · **Owner:** DeCode core · **Branch:** dev

---

## 1. Problem

Today `subagent` is a blocking tool call that shuts the child down when it
answers. The main agent waits for **all** subagents in a batch before it can
continue. As a result:

- One slow/failed subagent stalls the whole turn.
- A finished subagent cannot be followed up; it must be re-spawned.
- Subagents cannot cooperate; the main agent is the only router.
- Users cannot inject new instructions into a running subagent.

## 2. Goals

1. `subagent` returns immediately; the main agent keeps working.
2. A finished subagent stays **idle** (tab, session, skills, history intact).
3. Main agent can **follow up**, **close**, **message**, or **wait** on any subagent.
4. Subagents can message siblings and the main agent.
5. Every agent has a queued **inbox**; messages arriving mid-turn are merged
   into the next `context.json` and the agent re-plans.
6. A failed subagent fails alone — the main agent is notified and continues.

## 3. Subagent lifecycle

```
spawn → starting → running ⇄ idle → closing → closed
                     │
                     └─→ failed → closing → closed
```

- `running → idle`: final `answer` produced, no not-done todos.
- `idle → running`: follow-up instruction delivered.
- `idle`/`running → closing`: close requested (cooperative cancel if running).
- Only the main agent (or user) may close a subagent.
- `idle` subagents persist until explicitly closed or `SUBAGENT_IDLE_TTL_S` elapses.

## 4. New / changed tools

| Tool | Required | Optional | Returns |
|---|---|---|---|
| `subagent` (now async) | `instruction` | `subagent_id`, `skills` | `{id, state}` |
| `subagent_followup` | `agent_id`, `instruction` | — | `{id, state}` |
| `subagent_close` | `agent_id` | `reason` | `{id, state}` |
| `subagent_message` | `agent_id`, `message` | `kind` (`note`\|`request`) | `{id, queued}` |
| `subagent_status` | `agent_id` (`*` for all) | — | state + last answer + inbox size |
| `subagent_wait` | `agent_ids` | `timeout` | list of `{id, state, result?}` |

- `subagent` yields a handle as soon as the child registers; no waiting.
- `subagent_wait` is the **only** blocking subagent tool (used only when the
  main agent explicitly wants to synchronize).
- `subagent_message` allows subagent→sibling and subagent→main, but only
  within the same `parent_session_id`.

## 5. Inbox

Every agent (main and subagent) gets an ordered **inbox** on `Session`:

```python
inbox: list[dict]           # pending envelopes
inbox_archive: list[dict]   # delivered, bounded
```

Envelope:
```json
{"id":"inb_…","kind":"request|note|completion|failure|user",
 "source":"user|main|subagent:001_review-parser",
 "content":"…","meta":{...},"timestamp":"…"}
```

Delivery rules:

- Envelopes are rendered into `context.json` under a new top-level `inbox`
  section; they stay in `inbox` until the agent's next successful turn, then
  move to `inbox_archive`.
- Delivery is **not** a hard interrupt — the running turn completes first.
- A new envelope delivered to an `idle` subagent auto-transitions it to
  `running` and submits a new turn.
- The main agent receives `completion` / `failure` envelopes whenever a
  subagent changes state, so it can react without polling.

## 6. Non-blocking dispatch mechanics

- `SubagentSupervisor` (new, in `tools/subagents/supervisor.py`) owns one
  worker thread + queue per subagent runtime.
- `executor.execute_actions` calls `supervisor.spawn(...)` for `subagent`
  actions and immediately returns a `PendingToolResult` with the handle.
- Completion/failure events are surfaced to the chat loop as synthetic
  `PendingToolResult`s (`tool="subagent_completion"`), which the existing
  auto-continuation logic in `chat/loop.py` already handles — no new loop
  primitive needed.
- Per-runtime serialization is required because Playwright's sync API is
  thread-affine; one thread per subagent.

## 7. Context payload

Add one section to `render_session_context`:

```jsonc
{
  "request":  { ... },
  "inbox":    [ {id, kind, source, content, meta, timestamp}, ... ],
  "state":    { ... },
  "evidence": { ... },
  "capabilities": { ... },
  "history":  { ... }
}
```

Bounded by `MAX_INBOX_IN_CONTEXT` (20) and `MAX_INBOX_CONTENT_CHARS` (4000).

`build_turn_prompt` gains one rule: *if `inbox` has new entries, re-plan the
todos before continuing.*

## 8. Config additions

| Key | Default | Meaning |
|---|---|---|
| `MAX_LIVE_SUBAGENTS` | `8` | Cap on non-closed subagents |
| `SUBAGENT_IDLE_TTL_S` | `1800` | Auto-close idle TTL (0 disables) |
| `SUBAGENT_WAIT_DEFAULT_S` | `120` | `subagent_wait` default |
| `SUBAGENT_WAIT_MAX_S` | `1800` | `subagent_wait` cap |
| `MAX_INBOX_IN_CONTEXT` | `20` | Envelopes per turn |
| `MAX_INBOX_CONTENT_CHARS` | `4000` | Per-envelope cap |
| `ALLOW_SUBAGENT_TO_SUBAGENT_MESSAGES` | `true` | Sibling messaging |
| `ALLOW_SUBAGENT_SPAWN` | `false` | Subagents stay leaves |

## 9. GUI

- One tab per live subagent, including `idle`.
- State badge: running / idle / closing / failed.
- Idle tabs get a distinct style and increment `unread_count` on completion.
- `subagent_close` removes the tab after `closed`.
- Activity panel gains a per-agent **Inbox** count and last-answer preview.

## 10. Compatibility

- `subagent` name stays; semantics change from sync to async.
- `DECODE_SUBAGENT_SYNC=1` restores the legacy blocking behavior for one release.
- Old sessions without `inbox` load as `inbox=[]`.
- `AgentWorkspace.phase="completed"` kept as a read alias for `idle`.

## 11. Testing

- **Unit** — every state transition in §3; inbox merge/bounds; close on
  running triggers cooperative cancel; `subagent_message` authorization.
- **Integration (DummyEdgeBackend)** — main spawns 3, keeps working, receives
  completions, follows up on one, closes another, answers. Two subagents
  exchange a `subagent_message` and one re-plans. User input injected
  mid-turn appears in the target subagent's next `context.json`.
- **Regression** — `DECODE_SUBAGENT_SYNC=1` reproduces legacy behavior;
  `tool_workflow.py` still parses `{id, prompt, result}` completions.

## 12. Rollout

1. Land supervisor + async `subagent` behind `DECODE_SUBAGENT_ASYNC=1`.
2. Land inbox renderer + control tools.
3. Land GUI idle/inbox rendering.
4. Flip default on `dev`; keep `DECODE_SUBAGENT_SYNC=1` escape hatch.
5. Remove legacy path next release.

## 13. Open questions

1. Auto-close idle subagents after `SUBAGENT_IDLE_TTL_S`, or explicit only?
   *(Proposal: auto-close after TTL, with a warning event.)*
2. Expose `subagent_wait` to the model, or reserve it for the GUI?
   *(Proposal: expose it; the default `subagent` stays async.)*
3. Free-form text vs structured envelopes for inter-agent messages?
   *(Proposal: free-form with `kind`, structured `meta` for future use.)*

walkthrough, and expanded user stories were dropped.
