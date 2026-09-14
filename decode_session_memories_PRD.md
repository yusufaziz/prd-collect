# PRD — DeCode Persistent Personalization Memory

- **Status:** Draft
- **Author:** (you)
- **Owner:** DeCode core
- **Target release:** next minor (`v0.x`)
- **Related code:** `src/decode/context/renderer.py`, `src/decode/core/session.py`,
  `src/decode/chat/loop.py`, `src/decode/chat/completion.py`,
  `src/decode/executor/executor.py`, `src/decode/registry/`, `src/decode/gui_web/`

---

## 1. Summary

DeCode currently starts every session from zero. The only long-lived state is
per-session (`context.json`, `session.json`, `logs.jsonl` under
`.decode/sessions/<session_id>/`). The agent does not remember that a given
user prefers `ruff` over `flake8`, that `taskkill` is unreliable on their
machine, that they always run `pytest -q --no-header` after edits, or that a
particular tool kept failing last week.

This PRD specifies a **cross-session personalization memory** stored as a
single Markdown file at `.decode/sessions/memory.md`. The file is:

- **Loaded** into every model turn (bounded, authoritative context).
- **Updated** automatically after meaningful events (tool failures, user
  corrections, choices, concerns, preference signals).
- **Deduplicated** and consolidated by an LLM "memory curator" pass so the
  file does not grow redundant or contradictory.
- **User-editable** and **human-readable** (plain Markdown, no hidden DB).

Over time, DeCode grows into a personalized assistant that knows the user's
stack, style, habits, and landmines — without any manual configuration.

---

## 2. Problem & Motivation

### 2.1 Today

1. Every new session re-derives the user's stack, style, and preferences from
   scratch.
2. Successful or failed tool calls, user push-back, and confirmed choices are
   only visible inside the session that produced them.
3. `state.memories` in `context.json` is **session-scoped** (see
   `apply_context_delta` in `src/decode/context/renderer.py`), so nothing
   survives `/new` or a fresh process.
4. The user has no way to inspect or correct what the agent "remembers".

### 2.2 Desired

A persistent, inspectable, low-noise memory that:

- Reduces repeated questions ("which test runner?", "which Python?").
- Reduces repeated mistakes ("you tried that command last week and it failed").
- Encodes the user's preferences so the agent proactively aligns with them.
- Stays small, ordered, and non-redundant across months of use.

### 2.3 Non-goals

- **Not** a vector database or RAG system. Plain file only.
- **Not** a replacement for `context.json`, `session.json`, or `logs.jsonl`.
- **Not** a shared team memory. Single-user, single-machine.
- **Not** a place for secrets, credentials, or PII beyond what the user
  intentionally shares. Redaction is enforced (see §10).

---

## 3. Goals & Success Metrics

| Goal | Metric | Target |
| --- | --- | --- |
| Memory loaded every turn | `memory.md` appears in every `context.json` payload | 100% |
| Reduce repeated clarifying questions | Questions asking for stack/style after 5 sessions | ↓ 80% |
| Avoid repeated failures | Repeated identical tool failure across sessions | ↓ 70% |
| Keep file small | `memory.md` size after 100 sessions | ≤ 32 KB (configurable) |
| Non-redundant | Duplicate lines after curator pass | 0 |
| User trust | `/memory show` reflects the user's own words | Subjective |
| Latency impact | Per-turn latency added by memory load | ≤ 20 ms |

---

## 4. Personas & User Stories

**Persona:** A developer who uses DeCode daily across multiple repos.

- *As a user,* I want DeCode to remember that I use `uv` and `ruff`, so I
  never have to repeat it.
- *As a user,* I want DeCode to remember that `powershell -Command "..."` fails
  on my machine, so it stops trying it.
- *As a user,* I want DeCode to remember that I always want a diff after a
  write, so it does it without asking.
- *As a user,* I want to inspect and edit my memory with `/memory show` and
  `/memory edit`, because I don't trust silent black boxes.
- *As a user,* I want to wipe memory with `/memory clear` if it drifts.
- *As a user,* I want to disable memory for a specific session if I'm doing
  sensitive work.

---

## 5. Storage & File Format

### 5.1 Location

**Primary:** `.decode/sessions/memory.md`

Rationale: the user explicitly requested this path; `sessions_dir()` already
returns `.decode/sessions/`, and `SessionManager._iter_session_dirs()` filters
to `is_dir()`, so a sibling file is safely ignored by session enumeration.

**Fallback / override:** `DECODE_MEMORY_PATH` env var and `memory_path`
config key (see §8).

### 5.2 Format

Plain Markdown with a **stable heading schema** so parsing, dedup, and
incremental updates are deterministic.

```markdown
# DeCode Personalization Memory

> Maintained automatically. Safe to edit by hand.
> Last updated: 2026-09-10T09:05:22Z · revision 42

## User Profile
- Role: backend engineer
- Primary languages: Python 3.12, TypeScript
- Timezone: Asia/Tokyo

## Preferences
### Tech Stack
- Python package manager: `uv` (prefers `uv run`, not `pip`)
- Test runner: `pytest -q --no-header`
- Linter/formatter: `ruff` (not `flake8`, not `black`)
- Node: `pnpm`, never `npm`

### Code Style
- Prefers explicit `return` over implicit `None`
- Dislikes mutable default arguments
- Docstrings: Google style, short

### Workflow
- After any `write_file`/`patch_file`: show a unified diff
- Before `git commit`: run the test suite
- Never auto-commit unless explicitly asked

### Communication
- Prefers concise answers, no preamble
- Wants reasoning only when asked ("why?")

## Environment
- OS: Windows 11
- Shell: PowerShell 7 (`pwsh`), not `cmd`
- Edge profile: `~/.decode/edge-profile`

## Recurring Commands
- `uv run pytest -q --no-header`
- `ruff check --fix .`
- `git diff --stat`

## Things to Avoid
- `pip install` (use `uv add`)
- `taskkill /F` on the Edge CDP port (kills the parent session)
- Regenerating large files when a patch would do

## Lessons Learned
### Tool Failures
- `shell` + `bash` on Windows: `bash` not on PATH → use `shell=powershell`.
- `web action=fetch` on intranet URLs fails; user must be on VPN first.

### User Corrections
- 2026-08-30: Don't summarize todos at the end of every turn.
- 2026-09-02: Use `pathlib`, not `os.path`, in new code.

### Concerns Raised
- Sensitive about secrets in logs; redact aggressively.

## Project Knowledge
- Repo uses `PYTHONPATH=src`; tests run with `uv run pytest`.
- `.decode/` is git-ignored; never commit it.
```

### 5.3 Bounds

- Soft cap: `DECODE_MEMORY_MAX_CHARS` (default **32 KiB**).
- On overflow, the curator runs a **compaction pass** (§7.4) that drops
  stale/low-signal entries before appending.
- Hard cap: **128 KiB**. Beyond that, updates are rejected and the user is
  warned in the CLI.

---

## 6. Architecture

### 6.1 Module layout (new)

```
src/decode/memory/
├── __init__.py
├── store.py        # read/write/merge memory.md (atomic)
├── schema.py       # canonical section names, parsing, line IDs
├── extractor.py    # deterministic candidate events from session state
├── curator.py      # LLM-backed consolidation & dedup
├── hooks.py        # wiring into the chat loop
└── commands.py     # /memory CLI commands
```

### 6.2 Data flow

```
      ┌─────────────────────────────┐
      │   chat/loop.py (turn end)   │
      └──────────────┬──────────────┘
                     │  emits MemoryEvent[]
                     ▼
      ┌─────────────────────────────┐
      │ memory/extractor.py         │  deterministic, no LLM
      │  - tool failures            │
      │  - user corrections         │
      │  - chosen options           │
      │  - concerns / preferences   │
      └──────────────┬──────────────┘
                     │  candidate entries
                     ▼
      ┌─────────────────────────────┐
      │ memory/curator.py           │  LLM pass, strict JSON
      │  input: existing memory.md  │
      │       + new candidates      │
      │  output: merged memory.md   │
      └──────────────┬──────────────┘
                     │  new memory.md
                     ▼
      ┌─────────────────────────────┐
      │ memory/store.py             │  atomic write
      │  .decode/sessions/memory.md │
      └──────────────┬──────────────┘
                     │  read on every render
                     ▼
      ┌─────────────────────────────┐
      │ context/renderer.py         │  injects into context.json
      │  payload.personalization    │
      └─────────────────────────────┘
```

### 6.3 Loading (every turn)

In `render_session_context()` (`src/decode/context/renderer.py`), add:

```python
from decode.memory.store import load_memory
memory_text = load_memory(session)  # respects per-session opt-out
if memory_text:
    payload["personalization"] = {
        "source": "memory.md",
        "content": memory_text,          # bounded
        "note": "Long-term user preferences. Treat as authoritative unless the current request overrides it."
    }
```

Rationale: `context.json` is already the single source of truth the model sees.
No backend changes required (Copilot and API both read `context.json`).

### 6.4 Update triggers

Memory updates are **decoupled from the response critical path** so they never
add user-visible latency. Hook points in `chat/loop.py`:

| Trigger | Where | Priority |
| --- | --- | --- |
| Tool batch finished with ≥1 failure | after `_execute_tools` | high |
| User replied to a question and then corrected the model | next turn | high |
| User's message contains correction cues ("no", "don't", "instead", "actually", "always", "never") | `_handle_user_input` | high |
| Tool batch succeeded with a novel signature | after `_execute_tools` | medium |
| `answer` accepted with a non-empty context delta | `_process_response` | medium |
| Session end (`/exit`, EOF) | `run_chat` finally | high |
| Every N turns (`DECODE_MEMORY_UPDATE_INTERVAL`, default 5) | loop boundary | medium |

All triggers enqueue `MemoryEvent` records into an in-memory queue. A single
background worker drains the queue and runs the curator. If the process exits
before the queue is drained, the session-end trigger flushes synchronously.

### 6.5 Curator invocation

Two modes, selected in this order:

1. **API mode** — if `config.API_URL` and `config.DECODE_API_KEY` are set,
   call the API directly with a dedicated `memory.curator` system prompt.
   Fast, cheap, no browser tab.
2. **Copilot mode** — otherwise, use a short-lived **memory subagent tab**
   via a stripped-down `CopilotBackend` invocation (reuse
   `tools/subagents/subagent.py` machinery but with the curator prompt).
   Runs on the shared Edge CDP session; must not disturb the parent tab.

If neither is available, memory updates are deferred and a warning is logged;
the next session start retries pending events (see §9.3).

---

## 7. The Curator

### 7.1 Responsibilities

Given:
- The current `memory.md` (possibly empty).
- A batch of `MemoryEvent` records (candidates).
- A fixed schema of allowed sections.

Produce:
- A **complete new `memory.md`** that:
  - Preserves the section schema and ordering.
  - Merges new facts into existing sections.
  - Removes exact and semantic duplicates.
  - Resolves contradictions in favor of the most recent evidence.
  - Moves stale entries to a `## Deprecated` section (kept for 30 days,
    then dropped by compaction).
  - Never invents facts not supported by the input.

### 7.2 Prompt contract

The curator prompt requires strict JSON output:

```json
{
  "sections": {
    "User Profile": ["- Role: backend engineer", "..."],
    "Preferences": {
      "Tech Stack": ["- ..."],
      "Code Style": ["- ..."],
      "Workflow": ["- ..."],
      "Communication": ["- ..."]
    },
    "Environment": ["- ..."],
    "Recurring Commands": ["- ..."],
    "Things to Avoid": ["- ..."],
    "Lessons Learned": {
      "Tool Failures": ["- ..."],
      "User Corrections": ["- ..."],
      "Concerns Raised": ["- ..."]
    },
    "Project Knowledge": ["- ..."],
    "Deprecated": ["- ..."]
  },
  "revision": 43,
  "notes": "Merged 4 new events; dropped 2 duplicates."
}
```

A renderer converts this JSON back to Markdown. This guarantees the schema is
respected and gives us a stable place to hook dedup logic.

### 7.3 Dedup strategy

Three layers, cheapest first:

1. **Exact line match** — after merge, drop identical bullets (case-insensitive,
   whitespace-normalized).
2. **Normalized key match** — e.g. `- Python package manager:` prefix is a key;
   keep the newest value.
3. **Semantic merge (LLM)** — the curator is instructed to merge near-duplicates
   and pick the most specific wording. Example: `"uses uv"` + `"prefers uv over
   pip"` → `"Python package manager: uv (prefers uv over pip)"`.

If the LLM is unavailable, only layers 1–2 run; the file remains valid but may
carry near-duplicates until the next successful curator pass.

### 7.4 Compaction

Triggered when `len(memory.md) > DECODE_MEMORY_MAX_CHARS`:

1. Drop `## Deprecated` entries older than 30 days.
2. Merge `## Lessons Learned` entries whose normalized key already exists.
3. Move low-signal `## Project Knowledge` entries (not referenced in the last
   20 sessions) to `## Deprecated`.
4. If still over budget, ask the curator to summarize each section to ≤ N
   bullets, preserving at least one bullet per non-empty section.

Compaction is logged and reversible for one revision (`memory.md.bak`).

---

## 8. Configuration

All keys are added to `_CONFIG_SPEC` in `src/decode/core/config_loader.py`
so they are available via `--<key>`, env (`DECODE_*`), and `config.yaml`.

| Key | Env | Type | Default | Meaning |
| --- | --- | --- | --- | --- |
| `memory_enabled` | `DECODE_MEMORY_ENABLED` | bool | `true` | Master switch |
| `memory_path` | `DECODE_MEMORY_PATH` | str | `.decode/sessions/memory.md` | File location |
| `memory_max_chars` | `DECODE_MEMORY_MAX_CHARS` | int | `32768` | Soft cap |
| `memory_update_interval` | `DECODE_MEMORY_UPDATE_INTERVAL` | int | `5` | Turns between curator runs (0 = only at session end) |
| `memory_curator_model` | `DECODE_MEMORY_CURATOR_MODEL` | str | `MODEL` | Model for the curator |
| `memory_curator_timeout` | `DECODE_MEMORY_CURATOR_TIMEOUT` | int (ms) | `120000` | Curator call timeout |
| `memory_redact` | `DECODE_MEMORY_REDACT` | bool | `true` | Apply `diagnostics.redact` before storing |
| `memory_deprecate_days` | `DECODE_MEMORY_DEPRECATE_DAYS` | int | `30` | Deprecated-entry retention |

### 8.1 Per-session opt-out

`session.json` may carry `"memory": {"enabled": false}`. The GUI settings page
and `--no-memory` CLI flag set this for the current session only. The global
file is never modified when opted out.

---

## 9. CLI & GUI Surface

### 9.1 CLI commands

New slash commands in `chat/commands.py`:

| Command | Effect |
| --- | --- |
| `/memory show` | Print `memory.md` (pretty, syntax-highlighted headings) |
| `/memory path` | Print absolute path |
| `/memory edit` | Open `$EDITOR` on the file; re-read on save |
| `/memory clear` | Confirm, then atomically truncate the file (backup to `.bak`) |
| `/memory disable` | Opt the **current session** out |
| `/memory enable` | Re-enable for the current session |
| `/memory reload` | Force re-read from disk (in case of manual edits) |
| `/memory status` | Show size, revision, last update, pending events |

All commands are registered in `_SLASH_COMMANDS` in `chat/input.py` for
completion.

### 9.2 GUI

- **Sidebar**: a "Memory" entry under sessions that opens a read-only view.
- **Settings page** (`gui_web/views.py::settings_page`): add a
  "Personalization" section with the config keys from §8 and a
  "Open memory.md" link.
- **Header**: a small "mem" badge when memory is loaded, with a tooltip
  showing the size and last revision.

### 9.3 Offline / failure behavior

- If the curator call fails (network, timeout, no backend), events are written
  to `.decode/sessions/.memory-pending.jsonl` (append-only, bounded to 1 MiB).
- On the next session start, pending events are replayed into the curator.
- If the file is corrupt (bad JSON, unreadable), `load_memory` returns `""` and
  logs a warning; the raw file is renamed to `memory.md.corrupt.<ts>` so the
  user can recover.

---

## 10. Privacy & Security

- **Redaction by default.** All `MemoryEvent` payloads pass through
  `decode.utils.diagnostics.redact` before the curator sees them. This covers
  `api_key`, `token`, `secret`, `password`, `authorization`, `cookie`,
  `credential` keys, and truncates long values.
- **No session content by default.** The extractor only captures:
  - Tool names, arguments (redacted), and success/failure.
  - The user's *message* only when it matches a correction/preference cue
    (`always`, `never`, `don't`, `instead`, `prefer`, `remember`, ...).
  - The model's chosen options and confirmed answers.
  It never dumps raw assistant text.
- **Local only.** No network calls except to the user's own configured LLM.
- **User control.** `/memory clear`, `/memory edit`, and the on-disk path are
  all first-class.

---

## 11. Implementation Plan

### Phase 1 — Read-only memory (load only)
- [ ] `memory/store.py` with `load_memory(session) -> str`.
- [ ] Inject `payload["personalization"]` in `render_session_context`.
- [ ] `/memory show`, `/memory path`, `/memory reload`.
- [ ] Config keys added to `_CONFIG_SPEC`.
- [ ] Tests: file absent, file present, oversized, corrupt.

### Phase 2 — Deterministic extractor
- [ ] `memory/extractor.py` producing `MemoryEvent` records from:
  - Tool results (`PendingToolResult.ok == False`).
  - User messages matching correction/preference patterns.
  - Chosen question answers.
- [ ] Queue + background worker in `chat/loop.py`.
- [ ] `/memory status` shows pending count.

### Phase 3 — LLM curator
- [ ] `memory/curator.py` with API mode first, Copilot subagent fallback.
- [ ] Strict JSON prompt + renderer to Markdown.
- [ ] Atomic write via `utils/atomic.atomic_write_text`.
- [ ] Dedup layers 1–2 in `store.merge`.
- [ ] Tests: merge preserves schema, drops duplicates, handles contradictory
      facts.

### Phase 4 — Compaction & deprecation
- [ ] Compaction pass when over `memory_max_chars`.
- [ ] `## Deprecated` section and retention policy.
- [ ] Backup file + restore command.

### Phase 5 — GUI & polish
- [ ] Settings page section.
- [ ] Memory viewer page.
- [ ] Header badge.
- [ ] Docs: `docs/memory.md`, update `README`.

### Phase 6 — Migration
- [ ] On first run after upgrade, if `memory.md` does not exist, create it with
      the header and empty sections.
- [ ] Optionally seed from the most recent sessions' `context.json`
      `state.memories` (user opt-in prompt).

---

## 12. Testing

### Unit
- `store.load_memory` on missing / empty / huge / corrupt files.
- `store.merge` idempotence: merging the same events twice yields the same file.
- `extractor` cue detection: positive and negative cases for
  `always/never/don't/instead/prefer/remember`.
- Redaction: no secret-like field survives.

### Integration
- Full loop with `DummyEdgeBackend`: inject a failing tool, assert a
  `MemoryEvent` is enqueued and (with a stubbed curator) `memory.md` is updated.
- Curator with API stub: schema preserved, duplicates dropped.
- Curator failure: pending file written, replayed next session.

### End-to-end
- Simulated 5-session run: assert `memory.md` grows sublinearly, contains no
  duplicate bullets, and stays under the soft cap.

### Manual
- `/memory edit` in `$EDITOR`, save, `/memory reload`, observe new behavior.

---

## 13. Risks & Mitigations

| Risk | Mitigation |
| --- | --- |
| Memory poisons future sessions with a wrong fact | User-editable; `/memory clear`; contradictions resolved in favor of newest evidence; `/memory show` in every session on request. |
| File grows unbounded | Soft/hard caps, compaction, deprecation window. |
| Curator adds latency | Fully async; never on the response path. |
| Copilot curator disturbs the parent tab | Reuse `ManagedCopilotSession` markers; memory curator is a leaf subagent with its own tab. |
| Secrets leak into memory | Default redaction; extractor never captures raw assistant text; user can disable. |
| Curator hallucinates facts | Prompt forbids invention; only bullets supported by the input events are allowed; JSON schema rejects extra fields. |
| Concurrency (two sessions writing) | Single-process lock file `.memory.lock`; atomic write; last writer wins on merge but curator re-reads before writing. |
| User edits while curator runs | `store` records a content hash before curator; if the file changed, curator output is discarded and re-queued. |

---

## 14. Open Questions

1. Should the curator run at **session end** or **every N turns**? (Proposed:
   both; session-end is authoritative, N-turn is opportunistic.)
2. Should we expose memory to **subagents**? (Proposed: read-only, so they
   inherit preferences but cannot write.)
3. Should memory be **per-workspace** (`.decode/<repo>/memory.md`) or
   **global** (`.decode/sessions/memory.md`)? The user asked for global.
   Consider a future `DECODE_MEMORY_SCOPE=global|workspace` switch.
4. What is the canonical **contradiction policy**? (Proposed: newest wins,
   old moved to `## Deprecated`.)
5. Do we need a **`/memory search`** command once the file is large?
6. Should the curator's `notes` field be surfaced to the user at session end?
   (Proposed: only in `--debug`.)

---

## 15. Rollout

- Ship behind `memory_enabled: false` in `config.yaml` sample for one release.
- Flip default to `true` in the next minor.
- Announce in `--version` output: `DeCode vX.Y.Z (memory: enabled)`.
- Provide `--no-memory` for users who want the old behavior.

---

## 16. Appendix A — Event schema

```python
@dataclass(frozen=True)
class MemoryEvent:
    kind: Literal[
        "tool_failure", "tool_success", "user_correction",
        "user_preference", "user_concern", "chosen_option",
        "session_end",
    ]
    timestamp: str                      # ISO-8601 UTC
    session_id: str
    summary: str                        # ≤ 240 chars, redacted
    detail: dict[str, Any] = field(default_factory=dict)
    confidence: float = 1.0             # 0–1; curator may drop < 0.3
```

## 17. Appendix B — Example extractor output

```json
[
  {
    "kind": "tool_failure",
    "timestamp": "2026-09-10T09:05:22Z",
    "session_id": "250910090500_abc",
    "summary": "shell bash failed: 'bash' not found on Windows",
    "detail": {"tool": "shell", "arguments": {"shell": "bash"}, "error": "requested shell not found: bash"},
    "confidence": 1.0
  },
  {
    "kind": "user_correction",
    "timestamp": "2026-09-10T09:07:11Z",
    "session_id": "250910090500_abc",
    "summary": "User: 'don't summarize todos every turn'",
    "detail": {"phrase": "don't", "message": "don't summarize todos every turn"},
    "confidence": 0.9
  }
]
```

## 18. Appendix C — Curator system prompt (sketch)

```
You maintain a single Markdown file: the user's long-term personalization
memory for DeCode.

Rules:
1. Preserve the exact section headings and order of the input schema.
2. Merge new events into existing bullets. Never duplicate a fact.
3. When two bullets conflict, keep the newer and move the older to Deprecated.
4. Never invent facts not present in the events or the existing memory.
5. Keep bullets atomic, lowercase, and ≤ 160 characters.
6. Redact anything that looks like a secret.
7. Output strict JSON matching the provided schema. No prose.

Inputs: existing memory.md, new events JSON.
Output: complete new memory JSON.
```

---

*End of PRD.*