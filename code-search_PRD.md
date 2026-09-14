# code-search Skill — Product Requirements Document

**Skill name:** `code-search`
**Version:** 1.0
**Target toolchain:** CMake + Ninja + armcc / armlink
**Index:** SQLite at `<workspace>/build/index.sqlite`
**Runtime:** Python 3.8+ (stdlib only; `pyelftools` optional)
**Integration:** DeCode skill (loaded via `.decode/skills/code-search/`)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Tools](#3-tools)
4. [SKILLS.md](#4-skillsmd)
5. [tools/registry.json](#5-toolsregistryjson)
6. [tools/code_search.py](#6-toolscode_searchpy)
7. [Installation](#7-installation)
8. [Usage Examples](#8-usage-examples)
9. [Reference Architecture Notes](#9-reference-architecture-notes)

---

## 1. Overview

### 1.1 Purpose

Provide a **build-aware code search** across one or more firmware models. Only
functions that actually shipped in the linked image are returned. Each function
carries its caller/callee context, its exact `-D` build defines, and the source
range in the workspace. Multiple models share one SQLite index.

### 1.2 Goals

- Return only functions present in the linked image (verified via `map.txt` +
  `.axf`).
- Record the exact `-D` defines each translation unit was compiled with.
- Provide caller / callee traversal with configurable depth.
- Support multiple firmware models in one shared index.
- Auto-refresh the index when any source or build artifact changes.
- Store all paths **relative to the workspace root** for portability.
- Conform to the DeCode skill architecture used by `kanboard` and other
  reference skills.

### 1.3 Non-Goals

- No CMake File API usage (assumed unavailable in target projects).
- No preprocessed-body indexing (extension point only).
- No vector / embedding search by default (extension point only).
- No source-level parsing beyond brace matching (no `tree-sitter` dependency).
- No persistence of index state outside `build/index.sqlite`.

---

## 2. Architecture

### 2.1 Directory Layout

```
<workspace>/
├── .decode/
│   └── skills/
│       └── code-search/
│           ├── SKILLS.md
│           └── tools/
│               ├── registry.json
│               └── code_search.py
└── build/
    ├── MODEL-ABC/
    │   ├── compile_commands.json
    │   ├── map.txt
    │   ├── *.axf
    │   └── **/*.d
    ├── MODEL-XYZ/
    │   └── ...
    └── index.sqlite           ← single shared index
```

### 2.2 Conventions

- The skill's tools always run with `cwd = workspace root` (the external-tool
  executor sets this) so relative paths under `build/` resolve naturally.
- All paths stored in the index are **relative to the workspace root**.
- Every action prints **markdown** (never raw JSON).
- Errors print with an `ERROR:` prefix and exit code `1`.

### 2.3 Data Flow

```
build/MODEL-*/compile_commands.json  ──► per-TU sources + -D flags
build/MODEL-*/map.txt                ──► linked symbols + discarded sections
build/MODEL-*/*.axf                  ──► shipped functions (symtab + DWARF)
                                          │
                                          ▼
                                   build/index.sqlite
                                          │
                                          ▼
                     code-search-* tools (list/index/find/callers/callees)
                                          │
                                          ▼
                                    markdown output
```

---

## 3. Tools

| Tool | Purpose |
|---|---|
| `code-search-list-models` | Enumerate every model present in the shared index |
| `code-search-index` | Build or refresh the index for one model or all models |
| `code-search-find` | Main search: pattern → matching functions with caller/callee trees |
| `code-search-callers` | Every function that calls the named function, up to N levels |
| `code-search-callees` | Every function called by the named function, up to N levels |

All tools are registered under the `code-search-` prefix so the skill loader
(`src/decode/registry/external_skills.py`) automatically associates them with
the `code-search` skill defined in `SKILLS.md`.

---

## 4. SKILLS.md

Save the following as `.decode/skills/code-search/SKILLS.md`:

````markdown
---
name: code-search
description: Build-aware code search for CMake/Ninja/armcc firmware. Only returns functions that shipped in the linked image. Supports multiple models, caller/callee traversal, and per-TU define filters. Use when the user asks about firmware code, function callers, dependencies, or "what compiled".
---
# Code Search Skill

This skill searches firmware source code across one or more models. Tools
return **structured markdown** (not JSON). The index is a single SQLite
database at `<workspace>/build/index.sqlite` shared by all models.

## Quick Start

When user asks about firmware code:

1. **Discover models first**: `code-search-list-models`
2. **Ensure the index is fresh**: `code-search-index model=MODEL-ABC`
3. **Search**: `code-search-find model=MODEL-ABC content="*memory*|*_init"`
4. **Return the markdown** to the user

## Tools

### `code-search-list-models`
List every model that has been indexed. Call this first when you don't know
which model the user means.

**Params:** none

**Returns:**
```
# Indexed Models

- **MODEL-ABC** — 342 files, 1,204 functions (indexed 2026-09-14T09:12:04Z)
- **MODEL-XYZ** — 318 files, 1,158 functions (indexed 2026-09-14T09:10:51Z)
```

### `code-search-index`
Build or refresh the index for one model, or every model under `build/`. On
every `code-search-find` / `code-search-callers` / `code-search-callees`
invocation the index is automatically checked and refreshed if stale — you
only need this tool when you want to pre-warm it, or index a new model.

**Params:**
- `model` (optional) — model name (e.g. `MODEL-ABC`). Omit to index every model
  found under `build/`.

**Returns:**
```
# Index Updated

- **MODEL-ABC** — 342 files, 1,204 functions (rebuilt)
```

### `code-search-find`
Search for functions whose **name or body** matches one or more glob patterns.
Only functions present in the linked image (`map.txt` + `.axf`) are returned.

**Params:**
- `model` (required) — model name
- `content` (required) — pipe-separated glob patterns, e.g. `"*memory*|*_func"`
- `depth` (optional, default 2) — caller/callee tree depth
- `line` (optional, default 3) — lines of source context around each hit
- `include_comment` (optional, `true`/`false`, default `false`) — also match
  inside comments
- `define` (optional) — restrict to functions whose TU was compiled with this
  `-D` (accepts `NAME` or `NAME=VAL`)

**Returns:** see the "Output Format" section below.

### `code-search-callers`
List every function that calls the named function, up to `depth` levels.

**Params:**
- `model` (required)
- `function` (required) — exact function name
- `depth` (optional, default 2)

**Returns:**
```
# Callers of `sensor_read_memory` — MODEL-ABC (depth 2)

- sensor_poll (src/sensors/sensor.c:14)
  - app_task (src/main.c:88)
- temp_task (src/sensors/temp.c:120)
  - scheduler_tick (src/rtos.c:42)
```

### `code-search-callees`
List every function called by the named function, up to `depth` levels.

**Params:** same as `code-search-callers`

**Returns:** symmetrical to `code-search-callers`.

## Output Format

`code-search-find` returns the following markdown, one block per hit:

```c
# code-search result="*memory*|*_func" model="MODEL-ABC" depth=2 line=3 include_comment=false

## src/sensors/temp.c : func sensor_read_memory() : line 42-78
   39 | static int adc_ready(void) {
   40 |     return (ADC_SR & ADC_SR_EOC) != 0;
   41 | }
   42 | int sensor_read_memory(uint8_t ch, uint16_t *out) {
   43 |     if (!adc_ready()) return -1;
   ...
   78 | }
### function caller (depth 2)
- sensor_poll (src/sensors/sensor.c:14)
  - app_task (src/main.c:88)
- temp_task (src/sensors/temp.c:120)
### related function (depth 2)
- adc_ready (src/sensors/temp.c:39)
- i2c_write (src/i2c.c:55)
  - i2c_init (src/i2c.c:10)
```

## Workflow Examples

### Example 1: "What calls sensor_read_memory?"
1. `code-search-list-models` (if model unknown)
2. `code-search-callers model=MODEL-ABC function=sensor_read_memory depth=3`
3. Return the markdown tree to the user

### Example 2: "Search for memory functions that only compiled in FEATURE_A builds"
1. `code-search-index model=MODEL-ABC` (ensures fresh)
2. `code-search-find model=MODEL-ABC content="*memory*" define=FEATURE_A`
3. Return the markdown result

### Example 3: "Find every calibration routine across all models"
1. `code-search-list-models`
2. For each model: `code-search-find model=<m> content="*calib*"`
3. Combine results, return to user

## Rules

- **Never print raw JSON** — tools return formatted markdown
- **Never invent models** — always call `code-search-list-models` first if the
  user did not name a model
- **Only shipped code is returned** — the tool filters via `map.txt` and `.axf`
  automatically; do not bypass the filter
- **Index is shared** — one `build/index.sqlite` for all models
- **Index refreshes automatically** — every `find`/`callers`/`callees` call
  checks the fingerprint and rebuilds if source or build artifacts changed

## Error Handling

- **No models indexed:** Tell the user "No models have been indexed yet. Run
  `code-search-index` first, or verify the `build/` folder contains at least
  one subdirectory with `compile_commands.json`."
- **Unknown model:** Tell the user "Model `MODEL-X` was not found under
  `build/`. Available models: ..." and list them.
- **Empty result:** Tell the user "No matching functions in the linked image
  for this model." — the source may exist but was not compiled.
- **Index failure:** Show the error message verbatim; suggest checking
  `map.txt` / `.axf` presence.