# PRD: Asynchronous Multi-Agent Subagent Orchestration

**Status:** Draft
**Owner:** DeCode core
**Component:** `src/decode/tools/subagents/`, `src/decode/executor/`, `src/decode/chat/`, `src/decode/core/`, `src/decode/gui_web/`
**Related files:** `backend/backend.py`, `backend/dummy.py`, `backend/managed_session.py`, `chat/tool_workflow.py`, `chat/completion.py`, `context/renderer.py`, `gui_web/workspace.py`

---

## 1. Problem

The current subagent implementation is **synchronous and terminal**:

1. `subagent(instruction=...)` is a blocking tool call. The executor thread that
   runs it stays busy until the child agent produces its final answer.
2. The child agent, on producing an answer, is torn down in `finally:
   backend.shutdown()`. Its tab is closed and its session becomes unusable.
3. The main agent waits for **all** subagents in a tool batch to return
   (`executor.execute_actions` waits on the whole future set) before it can
   reason about their results.
4. There is no channel for:
   - the main agent to send a follow-up to a completed subagent,
   - the main agent to close an idle subagent on its own terms,
   - two subagents to exchange findings,
   - queued user input to reach an in-flight subagent mid-turn.

Consequences:

- One slow or failing subagent blocks the entire main-agent turn.
- A subagent that produced a partial answer cannot be asked to continue; it
  must be re-spawned from scratch, losing its context and loaded skills.
- Parallel subagents cannot cooperate; each works in isolation and the main
  agent alone must reconcile their outputs.
- Interactive users cannot inject new instructions into a running subagent
  (e.g. "stop searching for X, look at Y instead").

## 2. Goals

1. **Non-blocking dispatch.** The main agent can spawn one or more subagents
   and continue its own turn (reasoning, more tool calls, more spawns, or an
   answer) without waiting for any of them to finish.
2. **Persistent subagent lifecycle.** A subagent that produces a final answer
   enters an `idle` state and keeps its session, tab, loaded skills, and
   history intact until the main agent (or user) explicitly closes it.
3. **Follow-up continuation.** The main agent can send a follow-up instruction
   to an `idle` subagent, which resumes work using the same session/tab and
   returns a new answer.
4. **Explicit close.** Only the main agent (or user) can request that a
   subagent close. A close request transitions the subagent to `closing` and
   then `closed`, releasing the tab and persistent session.
5. **Inter-agent messaging.** Subagents can send messages to sibling subagents
   and to the main agent. Delivered messages appear in the recipient's next
   turn context.
6. **Queued inbox per agent.** Every agent (main and subagent) has a message
   inbox. Messages arriving mid-turn — from the user, from the main agent,
   from a sibling subagent, or as an async completion notification — are
   queued and merged into the next `context.json` under a dedicated `inbox`
   section, with explicit re-planning guidance.
7. **Per-agent isolation of failure.** A subagent that fails only fails itself.
   The main agent receives a failure notification and can re-plan.

## 3. Non-goals

- Distributed multi-process agents over the network.
- Autonomous subagent spawning (subagents remain leaves — they may not spawn
  their own subagents unless we later add an explicit allow flag; see §14).
- Replacing the existing `subagent` tool name in one shot; we will provide a
  compatibility shim (§12).
- Changing the Copilot browser contract or the `_response.json` attachment
  protocol.

## 4. Personas & user stories

### Main agent
- **M1** As the main agent, I dispatch three subagents and immediately continue
  writing code while they run, so I am not idle.
- **M2** As the main agent, when subagent A returns an answer, I can ask it a
  follow-up question without restarting it.
- **M3** As the main agent, when subagent B is finished with me, I close it to
  free the tab and its persistent session.
- **M4** As the main agent, when subagent C fails, I can re-plan and continue
  with A and B, without losing their work.
- **M5** As the main agent, I can ask subagent A to message subagent B with a
  shared finding, so B stops searching the same thing.

### Subagent
- **S1** As a subagent, after delivering my answer I stay alive in `idle` so
  the main agent can follow up.
- **S2** As a subagent, while I am running, I can receive a message from a
  sibling or from the user; I see it in my next context and can re-plan.
- **S3** As a subagent, I can send a message to the main agent or a sibling.

### User
- **U1** As the user, while a subagent is running, I can type a new
  instruction into the chat that reaches that subagent's next turn.
- **U2** As the user, I can see the live state of every subagent (running,
  idle, closing, closed, failed) in the GUI.

## 5. Definitions

- **Agent**: an LLM-driven actor with its own `Session`, backend, and tab.
  Two kinds: `main` and `subagent`.
- **Idle subagent**: a subagent that has produced a final answer and is
  waiting for a follow-up or close. Still owns its tab and session.
- **Inbox**: an ordered list of pending messages for one agent.
- **Envelope**: one message in an inbox — `{kind, source, content, timestamp,
  meta}`.
- **Supervisor**: the process-level registry that owns all live subagents,
  their worker threads, and their inboxes.

## 6. Functional requirements

### FR-1 — Non-blocking subagent dispatch
- `subagent` (new semantics) returns **immediately** with a handle:
  ```json
  {"name":"subagent","arguments":{"instruction":"...", "subagent_id":"..."}}
  ```
  → tool result: `{"id":"001_review-parser","state":"starting"}`.
- The actual child work runs on a dedicated supervisor worker thread.
- The tool result is delivered to the main agent's current turn as soon as
  the child has been registered and its tab opened — not when it finishes.

### FR-2 — Persistent subagent state machine

States: `starting → running ⇄ idle → closing → closed`, plus terminal `failed`.

| Transition | Trigger |
|---|---|
| `starting → running` | Backend initialized, first turn submitted |
| `running → idle` | Agent produced a final `answer` with no not-done todos |
| `running → failed` | Unrecoverable error / turn limit exceeded |
| `idle → running` | Follow-up instruction delivered |
| `idle → closing` | Close requested |
| `running → closing` | Close requested (cooperative cancel) |
| `closing → closed` | Backend shutdown complete |
| `failed → closing` | Cleanup after failure notification |

A subagent is **not** shut down when it reaches `idle`. Its `SessionManager`
directory, loaded skills, message history, and Playwright page remain alive.

### FR-3 — Main-agent control tools

New built-in tools (registered in `builtin_tools.py`):

| Tool | Required | Optional | Returns |
|---|---|---|---|
| `subagent` (async) | `instruction` | `subagent_id`, `skills` | `{id, state}` |
| `subagent_wait` | `agent_ids` (list or CSV) | `timeout` | list of `{id, state, result?}` |
| `subagent_followup` | `agent_id`, `instruction` | `skills` | `{id, state}` (queued) |
| `subagent_close` | `agent_id` | `reason` | `{id, state}` |
| `subagent_status` | `agent_id` (or `*`) | — | state + last answer + inbox size |
| `subagent_message` | `agent_id`, `message` | `kind` (`note`\|`request`) | `{id, queued}` |

- `subagent_followup` on an `idle` agent transitions it to `running` and
  enqueues `instruction` into its inbox.
- `subagent_close` on a `running` agent first sets a cooperative cancel flag
  (mirrors `cancel_current_response`), then shuts down.
- `subagent_status` with `agent_id="*"` returns a bounded list of all live
  subagents owned by the requesting main agent.

### FR-4 — Subagent-to-subagent messaging
- `subagent_message` accepts any agent id visible to the sender:
  - subagent → main
  - subagent → sibling subagent
  - main → subagent
- Cross-session authorization:
  - The sender must share the same `parent_session_id` as the recipient, or
    be the main agent that owns the recipient.
  - Messages to unknown/closed agents return an error envelope, not a crash.

### FR-5 — Queued inbox on every agent
Add to `Session`:
```python
inbox: list[dict]           # pending envelopes
inbox_archive: list[dict]   # delivered, bounded to N (config)
```
An envelope:
```json
{
  "id": "inb_...",
  "kind": "request" | "note" | "completion" | "failure" | "user",
  "source": "user" | "main" | "subagent:001_review-parser",
  "content": "…",
  "meta": {"agent_id": "...", "ok": true, "todos": [...]},
  "timestamp": "..."
}
```

Delivery rules:
- Envelopes are **rendered into `context.json`** under a new top-level
  `inbox` section (see §7) and left in `inbox` until the agent's next
  successful turn completes; then they are moved to `inbox_archive`.
- Delivery is **not** a hard interrupt: the running turn finishes, then the
  next render includes them.
- For an `idle` subagent, a new envelope automatically transitions it to
  `running` (a new turn is submitted).

### FR-6 — Completion & failure notifications
When a subagent reaches `idle`, `failed`, or `closed`, the supervisor enqueues
an envelope into the **main agent's** inbox:
```json
{"kind":"completion","source":"subagent:001_review-parser",
 "content":"Final answer: …", "meta":{"agent_id":"...","ok":true}}
```
The main agent's next turn sees it and can:
- `subagent_followup` to continue,
- `subagent_close` to release,
- spawn another,
- or ignore it and finish its own answer.

### FR-7 — Re-planning guidance
`context.json` gains an `inbox` section and the turn prompt
(`build_turn_prompt`) gains a rule: *if `inbox` contains new messages, re-plan
the todos before continuing; do not silently drop them.*

### FR-8 — GUI representation
- `SessionWorkspace` keeps one `AgentWorkspace` per subagent id, including
  `idle` agents.
- Tab shows a state badge: running / idle / closing / failed.
- `idle` tabs are visually distinct and unread-completion events increment
  `unread_count`.
- `subagent_close` events remove the tab after `closed`.
- The activity panel gains an **Inbox** subsection per agent.

## 7. Context payload changes

Add to `render_session_context` payload:

```jsonc
{
  "request": { ... },
  "inbox": [
    {
      "id": "inb_7f…",
      "kind": "completion",
      "source": "subagent:001_review-parser",
      "content": "Final answer: parser exposes 3 unused imports.",
      "meta": {"agent_id":"001_review-parser","ok":true,"todos":[...]},
      "timestamp": "2026-09-10T09:05:00Z"
    }
  ],
  "state": { ... },
  "evidence": { ... },
  "capabilities": { ... },
  "history": { ... }
}
```

Bounded: max `config.MAX_INBOX_IN_CONTEXT` envelopes (default 20), newest
first, each `content` truncated to `config.MAX_INBOX_CONTENT_CHARS`.

## 8. Architecture

### 8.1 New component: `SubagentSupervisor`

Location: `src/decode/tools/subagents/supervisor.py`.

Responsibilities:
- Own a singleton registry `{agent_id → SubagentRuntime}`.
- Own one long-lived worker thread + `queue.Queue` per subagent runtime.
- Serialize Playwright access per runtime (Playwright sync API is
  thread-affine — each runtime must own exactly one thread).
- Provide `spawn`, `followup`, `close`, `status`, `list`, `message`,
  `wait_for`.
- Publish state-change callbacks to the GUI adapter (reusing the existing
  `progress_callback` shape, extended with `subagent_state` events).
- Enforce `config.MAX_LIVE_SUBAGENTS` and reject spawns beyond the limit.

### 8.2 `SubagentRuntime`

Per-agent state:
```python
@dataclass
class SubagentRuntime:
    agent_id: str
    parent_session_id: str
    state: SubagentState
    inbox: queue.Queue
    thread: threading.Thread
    backend: Backend        # initialized on its thread
    session: Session
    last_answer: str
    last_error: str
    loaded_skills: list[str]
    created_at: str
    close_requested: threading.Event
    stop_requested: threading.Event
```

Worker loop (single-threaded per runtime):
1. Initialize backend (`CopilotBackend` or `DummyEdgeBackend`).
2. Loop:
   - Drain inbox → merge envelopes into `session.inbox`.
   - If `close_requested` → break.
   - If `state == idle` and inbox is empty → block on `queue.get` with
     timeout (to allow cooperative cancel / close).
   - Else run one turn (`_send_to_backend` + `_process_response` +
     `_execute_tools`) as today.
   - On final answer with no not-done todos → state `idle`, publish
     completion envelope to parent, save session.
   - On failure → state `failed`, publish failure envelope, break.
3. Shutdown backend (close tab), state `closed`.

### 8.3 Executor changes

- `execute_actions` no longer blocks on `subagent` futures. When it detects a
  `subagent` action it calls `SubagentSupervisor.spawn(...)` and immediately
  yields a `PendingToolResult(ok=True, output=json.dumps({id, state}))`.
- `subagent_wait` is the *only* blocking subagent tool; it waits on the
  supplied ids up to `timeout` (default `config.SUBAGENT_WAIT_DEFAULT_S`, cap
  `config.SUBAGENT_WAIT_MAX_S`). Non-terminal timeouts return the current
  state, not an error.
- Per-tool soft timeout behavior for `subagent` no longer applies to the
  whole child lifetime — only to the spawn handshake.

### 8.4 Chat loop changes

- `chat/loop.py` no longer relies on subagent tool results carrying final
  answers. Answers arrive via the inbox; the loop already supports
  auto-continuation when `pending_tool_results` is non-empty, so supervisor
  completion events are surfaced as synthetic `PendingToolResult`s with
  `tool="subagent_completion"` and `params={"agent_id": ...}`.
- A new session field `_inbox_wakeup: bool` is set by the supervisor when a
  new envelope is delivered; the loop's `_handle_user_input` returns
  `auto=True` when set (mirroring the current auto-continuation flow).
- `require_request` policy is unchanged.

### 8.5 GUI changes

- `AgentWorkspace.phase` gains `idle`/`closing` (already has `running`,
  `completed`, `failed`; rename `completed` → `idle` for subagents).
- New events from supervisor:
  - `subagent_state` `{subagent_id, state, last_answer, last_error}`
  - `subagent_inbox` `{subagent_id, count}`
- `gui_web/views.py` renders idle badges and an Inbox panel.

## 9. Tool contracts (exact JSON)

```jsonc
// subagent (async)
{"name":"subagent","arguments":{"instruction":"Review parser for unused imports","subagent_id":"review-parser","skills":["python-review"]}}
→ {"id":"001_review-parser","state":"starting"}

// subagent_followup
{"name":"subagent_followup","arguments":{"agent_id":"001_review-parser","instruction":"Also check callers in src/decode/chat"}}
→ {"id":"001_review-parser","state":"running"}

// subagent_close
{"name":"subagent_close","arguments":{"agent_id":"001_review-parser","reason":"task complete"}}
→ {"id":"001_review-parser","state":"closing"}

// subagent_status
{"name":"subagent_status","arguments":{"agent_id":"*"}}
→ {"agents":[{"id":"001_review-parser","state":"idle","last_answer_chars":812,"inbox":0}]}

// subagent_message
{"name":"subagent_message","arguments":{"agent_id":"002_scanner","message":"parser has 3 unused imports; skip that check","kind":"note"}}
→ {"id":"002_scanner","queued":true}

// subagent_wait
{"name":"subagent_wait","arguments":{"agent_ids":["001_review-parser","002_scanner"],"timeout":120}}
→ {"agents":[{"id":"001_review-parser","state":"idle","result":"…"},{"id":"002_scanner","state":"running"}]}
```

## 10. State machine (authoritative)

```
                    spawn
                      │
                      ▼
                 ┌──────────┐
                 │ starting │
                 └────┬─────┘
                      │ first turn submitted
                      ▼
      followup  ┌──────────┐  answer(no todos)
        ┌──────►│ running  ├──────────────┐
        │       └────┬─────┘              │
        │            │ error              ▼
        │            ▼              ┌──────────┐
        │       ┌──────────┐        │  idle    │
        │       │  failed  │        └────┬─────┘
        │       └────┬─────┘             │ close / user close
        │            │ cleanup           ▼
        │            │             ┌──────────┐
        └────────────┴────────────►│ closing  │
                                   └────┬─────┘
                                        ▼
                                   ┌──────────┐
                                   │  closed  │
                                   └──────────┘
```

`failed → closing` is driven by the supervisor's own cleanup, not by the
main agent.

## 11. Config additions (`config.py` + `_CONFIG_SPEC`)

| Key | Default | Meaning |
|---|---|---|
| `MAX_LIVE_SUBAGENTS` | `8` | Hard cap on non-closed subagents per main session |
| `SUBAGENT_IDLE_TTL_S` | `1800` | Idle auto-close TTL (0 disables) |
| `SUBAGENT_WAIT_DEFAULT_S` | `120` | `subagent_wait` default |
| `SUBAGENT_WAIT_MAX_S` | `1800` | `subagent_wait` cap |
| `MAX_INBOX_IN_CONTEXT` | `20` | Envelopes rendered per turn |
| `MAX_INBOX_CONTENT_CHARS` | `4000` | Per-envelope content cap |
| `INBOX_ARCHIVE_LIMIT` | `200` | Retained archived envelopes |
| `ALLOW_SUBAGENT_TO_SUBAGENT_MESSAGES` | `true` | Sibling messaging |
| `ALLOW_SUBAGENT_SPAWN` | `false` | Subagents may not spawn subagents by default |

## 12. Compatibility & migration

- The `subagent` tool **name** stays; its **semantics** change from sync to
  async. A `DECODE_SUBAGENT_SYNC=1` env var restores the legacy blocking
  behavior for one release.
- `Session.subagents_result` is retained for persisted-session compatibility;
  new completions also append there when they reach `idle`.
- Old persisted sessions without `inbox` load as `inbox=[]`.
- `AgentWorkspace.phase="completed"` is kept as a read alias for `idle`.

## 13. Observability

- Structured debug events (via `utils.diagnostics.debug_event`):
  `subagent.spawned`, `subagent.state`, `subagent.followup`,
  `subagent.closed`, `subagent.message`, `subagent.inbox_delivered`,
  `subagent.wait_timeout`.
- Each subagent's `session.json` continues to be the source of truth for its
  own history; the supervisor never mutates a closed session's files.
- The GUI activity panel gains a per-agent Inbox count and last-answer preview.

## 14. Security & isolation

- Subagent tabs remain marked with `ManagedCopilotSession` markers; the
  supervisor never touches a sibling's page.
- `subagent_message` validates that sender and recipient share a
  `parent_session_id`; cross-session messaging is rejected.
- `subagent_close` is the only path that releases a subagent's tab; a
  subagent cannot close itself except by reaching a terminal failure.
- Subagent leaves cannot spawn subagents unless
  `ALLOW_SUBAGENT_SPAWN=true` (default false), preserving the current
  "subagents are leaves" guarantee.

## 15. Testing

- **Unit**
  - Supervisor state transitions for each edge in §10.
  - Inbox merge → `context.json` `inbox` section shape and bounds.
  - `subagent_close` on `running` triggers cooperative cancel.
  - `subagent_wait` returns non-terminal states without error.
  - Authorization matrix for `subagent_message`.
- **Integration (DummyEdgeBackend)**
  - Main spawns 3 subagents, continues its own turn, receives completions in
    order of arrival, follows up on one, closes another, and produces a final
    answer.
  - Two subagents exchange a `subagent_message` and one visibly re-plans.
  - User input injected mid-subagent-turn appears in that subagent's next
    `context.json`.
- **Regression**
  - `DECODE_SUBAGENT_SYNC=1` reproduces legacy behavior.
  - Existing `subagent` result consumption in `tool_workflow.py` still parses
    the completion envelope (`{id, prompt, result}`).
- **GUI**
  - Idle tab badge, unread counters, close removes tab.

## 16. Rollout

1. Land supervisor + async `subagent` behind `DECODE_SUBAGENT_ASYNC=1`
   (default on for `dev` branch, off for `main`).
2. Land inbox renderer + control tools.
3. Land GUI idle/inbox rendering.
4. Flip default, keep `DECODE_SUBAGENT_SYNC=1` escape hatch for one release.
5. Remove legacy path.

## 17. Open questions

1. Should `subagent_wait` participate in the LLM's tool schema as a blocking
   call, or be reserved for the GUI? (Proposal: expose it; the model may
   choose it, but the default `subagent` remains async.)
2. Should idle subagents auto-close after `SUBAGENT_IDLE_TTL_S`, or only on
   explicit close? (Proposal: auto-close after TTL, with a warning event.)
3. Should subagent→subagent messages be free-form text or schema-validated
   envelopes? (Proposal: free-form text with `kind: note|request`; keep the
   door open for structured `meta`.)
4. How should the main agent's own `context.json` bound `inbox` growth when
   the user pastes a large block mid-turn? (Proposal: same truncation as
   other history sections.)
5. Do we need a `subagent_pause` / `subagent_resume` pair, or is `idle` +
   `followup` sufficient? (Proposal: `idle`/`followup` is sufficient for v1.)