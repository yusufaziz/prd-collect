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
```

---

5. tools/registry.json

Save the following as .decode/skills/code-search/tools/registry.json:

```json
[
  {
    "tool_name": "code-search-list-models",
    "description": "List every firmware model that has been indexed in build/index.sqlite. Required: none. Optional: none. Returns each model name with file/function counts and its last index timestamp. Use this first when you don't know which model to search.",
    "args_schema": {},
    "execution": "python code_search.py list-models",
    "example": {
      "good": ["code-search-list-models"],
      "bad": ["code-search-list-models model=MODEL-ABC"]
    }
  },
  {
    "tool_name": "code-search-index",
    "description": "Build or refresh the SQLite index for one or more firmware models. Required: none. Optional: model (single model name; omit to index every model under build/). Rebuilds only when the source or build-artifact fingerprint changed. Returns the number of files and functions indexed per model.",
    "args_schema": {
      "model": {"type": "string", "required": false, "description": "Single model name (e.g. MODEL-ABC). Omit to index all models under build/."}
    },
    "execution": "python code_search.py index",
    "example": {
      "good": ["code-search-index model=MODEL-ABC", "code-search-index"],
      "bad": ["code-search-index model=../etc"]
    }
  },
  {
    "tool_name": "code-search-find",
    "description": "Search firmware functions by name or body glob pattern. Only functions present in the linked image (map.txt + .axf) are returned. Required: model, content. Optional: depth (caller/callee tree depth, default 2), line (source context lines, default 3), include_comment (true/false, default false), define (restrict to functions whose translation unit was compiled with this -D). Returns each hit with its source range, caller tree, and callee tree as markdown.",
    "args_schema": {
      "model": {"type": "string", "required": true, "description": "Model name (e.g. MODEL-ABC)."},
      "content": {"type": "string", "required": true, "description": "Pipe-separated glob patterns, e.g. '*memory*|*_func'."},
      "depth": {"type": "integer", "required": false, "description": "Caller/callee traversal depth (default 2)."},
      "line": {"type": "integer", "required": false, "description": "Lines of source context around each hit (default 3)."},
      "include_comment": {"type": "string", "required": false, "description": "Search inside comments too (true/false, default false)."},
      "define": {"type": "string", "required": false, "description": "Only functions whose TU was compiled with this -D (NAME or NAME=VAL)."}
    },
    "execution": "python code_search.py find",
    "example": {
      "good": [
        "code-search-find model=MODEL-ABC content=*memory*",
        "code-search-find model=MODEL-ABC content='*memory*|*_func' depth=3 line=5",
        "code-search-find model=MODEL-ABC content=calibrate define=FEATURE_A"
      ],
      "bad": ["code-search-find model=MODEL-ABC"]
    }
  },
  {
    "tool_name": "code-search-callers",
    "description": "List every function that calls the named function, up to depth levels. Required: model, function. Optional: depth (default 2). Returns an indented tree of caller functions with workspace-relative file:line for each.",
    "args_schema": {
      "model": {"type": "string", "required": true, "description": "Model name (e.g. MODEL-ABC)."},
      "function": {"type": "string", "required": true, "description": "Exact function name to look up."},
      "depth": {"type": "integer", "required": false, "description": "Traversal depth (default 2)."}
    },
    "execution": "python code_search.py callers",
    "example": {
      "good": ["code-search-callers model=MODEL-ABC function=sensor_read_memory depth=3"],
      "bad": ["code-search-callers model=MODEL-ABC"]
    }
  },
  {
    "tool_name": "code-search-callees",
    "description": "List every function called by the named function, up to depth levels. Required: model, function. Optional: depth (default 2). Returns an indented tree of callee functions with workspace-relative file:line for each.",
    "args_schema": {
      "model": {"type": "string", "required": true, "description": "Model name (e.g. MODEL-ABC)."},
      "function": {"type": "string", "required": true, "description": "Exact function name to look up."},
      "depth": {"type": "integer", "required": false, "description": "Traversal depth (default 2)."}
    },
    "execution": "python code_search.py callees",
    "example": {
      "good": ["code-search-callees model=MODEL-ABC function=sensor_read_memory depth=3"],
      "bad": ["code-search-callees model=MODEL-ABC"]
    }
  }
]
```

---

6. tools/code_search.py

Save the following as .decode/skills/code-search/tools/code_search.py:

```python
 #!/usr/bin/env python3
"""code_search.py — build-aware firmware code search skill.

Actions:
  list-models
  index       [model=MODEL]
  find        model=MODEL content=PATTERN [depth=N] [line=N] [include_comment=BOOL] [define=NAME]
  callers     model=MODEL function=NAME [depth=N]
  callees     model=MODEL function=NAME [depth=N]

All actions print markdown. Errors are printed with an "ERROR:" prefix.
"""

import hashlib
import json
import re
import sqlite3
import sys
import time
from collections import defaultdict
from pathlib import Path


# ---------------------------------------------------------------------------
# Constants
# ---------------------------------------------------------------------------
SQLITE_NAME = "index.sqlite"
CC_NAME     = "compile_commands.json"
MAP_NAME    = "map.txt"
BUILD_DIR   = "build"

C_KEYWORDS = {
    "if", "for", "while", "switch", "return", "sizeof", "do", "else",
    "case", "break", "continue", "goto", "default", "defined",
}
CALL_RE = re.compile(r'\b([A-Za-z_]\w*)\s*\(')

SCHEMA = """
CREATE TABLE IF NOT EXISTS models (
    name         TEXT PRIMARY KEY,
    build_dir    TEXT NOT NULL,
    fingerprint  TEXT NOT NULL,
    indexed_at   INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS files (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    model     TEXT NOT NULL,
    rel_path  TEXT NOT NULL,
    abs_path  TEXT NOT NULL,
    UNIQUE(model, rel_path)
);
CREATE TABLE IF NOT EXISTS functions (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    model       TEXT NOT NULL,
    file_id     INTEGER NOT NULL,
    name        TEXT NOT NULL,
    start_line  INTEGER NOT NULL,
    end_line    INTEGER NOT NULL,
    body        TEXT NOT NULL,
    built       INTEGER NOT NULL,
    in_axf      INTEGER NOT NULL,
    in_map      INTEGER NOT NULL,
    discarded   INTEGER NOT NULL,
    addr        INTEGER,
    tu_defines  TEXT,
    FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE,
    UNIQUE(model, file_id, name, start_line)
);
CREATE TABLE IF NOT EXISTS calls (
    model        TEXT NOT NULL,
    caller_id    INTEGER NOT NULL,
    callee_name  TEXT NOT NULL,
    PRIMARY KEY (model, caller_id, callee_name),
    FOREIGN KEY(caller_id) REFERENCES functions(id) ON DELETE CASCADE
);
CREATE INDEX IF NOT EXISTS idx_func_name    ON functions(model, name);
CREATE INDEX IF NOT EXISTS idx_func_file    ON functions(model, file_id);
CREATE INDEX IF NOT EXISTS idx_calls_callee ON calls(model, callee_name);
CREATE INDEX IF NOT EXISTS idx_files_rel    ON files(model, rel_path);
"""


class CodeSearchError(Exception):
    """Expected tool/runtime error."""


# ---------------------------------------------------------------------------
# Arg parsing (same style as kanboard.py)
# ---------------------------------------------------------------------------
def _get_args() -> dict:
    args = {}
    for a in sys.argv[2:]:
        if "=" in a:
            k, v = a.split("=", 1)
            args[k.lstrip("-")] = v
    return args


def _as_bool(value, default=False) -> bool:
    if value is None:
        return default
    return str(value).strip().lower() in {"true", "1", "yes", "on"}


def _as_int(value, default: int) -> int:
    try:
        return int(value)
    except (TypeError, ValueError):
        return default


# ---------------------------------------------------------------------------
# Workspace paths
# ---------------------------------------------------------------------------
def _workspace_root() -> Path:
    # The external-tool executor always runs with cwd = workspace root.
    return Path.cwd().resolve()


def _open_db() -> sqlite3.Connection:
    db_path = _workspace_root() / BUILD_DIR / SQLITE_NAME
    db_path.parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(str(db_path))
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    conn.executescript(SCHEMA)
    return conn


# ---------------------------------------------------------------------------
# Source parsing helpers
# ---------------------------------------------------------------------------
def find_functions(text: str):
    """Yield (name, start_line, end_line); 1-indexed, brace-matched."""
    lines = text.split("\n")
    n = len(lines)
    i = 0
    while i < n:
        s = lines[i].strip()
        if not s or s.startswith(("#", "//", "*", "/*")):
            i += 1
            continue
        m = re.search(r'\b([A-Za-z_]\w*)\s*\(', lines[i])
        if not m or m.group(1) in C_KEYWORDS:
            i += 1
            continue
        if ";" in lines[i][m.end():].split("{")[0]:
            i += 1
            continue
        j = i
        while j < n and "{" not in lines[j] and ";" not in lines[j]:
            j += 1
        if j >= n or "{" not in lines[j] or ";" in lines[j].split("{")[0]:
            i += 1
            continue
        depth, started, k = 0, False, j
        while k < n:
            for ch in lines[k]:
                if ch == "{":
                    depth += 1
                    started = True
                elif ch == "}":
                    depth -= 1
            if started and depth == 0:
                break
            k += 1
        if started and depth == 0:
            yield m.group(1), i + 1, k + 1
            i = k + 1
        else:
            i += 1


def parse_compile_db(cc_path: Path):
    out = {}
    if not cc_path.exists():
        return out
    try:
        entries = json.loads(cc_path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError):
        return out
    for e in entries:
        f = Path(e["file"])
        if not f.is_absolute():
            f = (Path(e["directory"]) / f).resolve()
        argv = e.get("arguments")
        if argv is None:
            import shlex
            argv = shlex.split(e.get("command", ""), posix=False)
        ds = set()
        for idx, a in enumerate(argv):
            if a == "-D" and idx + 1 < len(argv):
                ds.add(argv[idx + 1])
            elif a.startswith("-D") and len(a) > 2:
                ds.add(a[2:])
        out[f] = {"entry": e, "defines": ds}
    return out


def parse_map(map_path: Path):
    linked, discarded = set(), set()
    if not map_path.exists():
        return linked, discarded
    removed_sec = re.compile(r'^\s*\.\w+\.(\w+)\s+0x')
    symbol_re   = re.compile(r'^\s+(\w+)\s+0x[0-9a-fA-F]+')
    in_removed  = False
    try:
        text = map_path.read_text(encoding="utf-8", errors="ignore")
    except OSError:
        return linked, discarded
    for raw in text.splitlines():
        line = raw.rstrip()
        if "Removed Sections" in line or "Discarded input sections" in line:
            in_removed = True
            continue
        if in_removed:
            m = removed_sec.match(line)
            if m:
                discarded.add(m.group(1))
            if not line.strip():
                in_removed = False
            continue
        m = symbol_re.match(line)
        if m:
            linked.add(m.group(1))
    return linked, discarded


def parse_axf(axf_path: Path):
    out = {}
    if not axf_path or not axf_path.exists():
        return out
    try:
        from elftools.elf.elffile import ELFFile
    except ImportError:
        return _parse_axf_fromelf(axf_path)
    try:
        with open(axf_path, "rb") as f:
            elf = ELFFile(f)
            symtab = elf.get_section_by_name(".symtab")
            if symtab:
                for sym in symtab.iter_symbols():
                    if sym["st_info"]["type"] in ("STT_FUNC", "STT_GNU_IFUNC"):
                        name = sym.name
                        if name:
                            out[name] = {"addr": sym["st_value"]}
            if elf.has_dwarf_info():
                dwarf = elf.get_dwarf_info()
                for cu in dwarf.iter_CUs():
                    for die in cu.iter_DIEs():
                        if die.tag != "DW_TAG_subprogram":
                            continue
                        nm = die.attributes.get("DW_AT_name")
                        if not nm:
                            continue
                        fname = nm.value.decode("utf-8", "ignore")
                        if fname not in out:
                            low = die.attributes.get("DW_AT_low_pc")
                            out[fname] = {"addr": low.value if low else None}
    except Exception:
        return {}
    return out


def _parse_axf_fromelf(axf_path: Path):
    import subprocess
    try:
        txt = subprocess.run(
            ["fromelf", "--text", "-s", str(axf_path)],
            capture_output=True, text=True, check=True,
        ).stdout
    except (FileNotFoundError, subprocess.CalledProcessError):
        return {}
    out = {}
    rx = re.compile(r'^\s+(\w+)\s+0x([0-9a-fA-F]+)')
    for line in txt.splitlines():
        m = rx.match(line)
        if m:
            out[m.group(1)] = {"addr": int(m.group(2), 16)}
    return out


def _find_first_axf(model_dir: Path):
    for p in sorted(model_dir.glob("*.axf")):
        return p
    for p in sorted(model_dir.rglob("*.axf")):
        return p
    return None


# ---------------------------------------------------------------------------
# Fingerprint + indexer
# ---------------------------------------------------------------------------
def _fingerprint(model_dir: Path, cc_data: dict) -> str:
    h = hashlib.sha1()
    cc_path = model_dir / CC_NAME
    try:
        h.update(cc_path.read_bytes())
    except OSError:
        pass
    for src in sorted(cc_data.keys()):
        try:
            st = src.stat()
            h.update(f"{src}|{st.st_mtime_ns}|{st.st_size}\n".encode())
        except OSError:
            pass
    mp = model_dir / MAP_NAME
    if mp.exists():
        try:
            h.update(f"map|{mp.stat().st_mtime_ns}\n".encode())
        except OSError:
            pass
    axf = _find_first_axf(model_dir)
    if axf:
        try:
            h.update(f"axf|{axf}|{axf.stat().st_mtime_ns}\n".encode())
        except OSError:
            pass
    return h.hexdigest()


def _to_rel(path: Path, root: Path) -> str:
    try:
        return str(path.resolve().relative_to(root.resolve()))
    except ValueError:
        return str(path)


def _delete_model(conn: sqlite3.Connection, model: str):
    cur = conn.cursor()
    cur.execute("DELETE FROM calls WHERE model=?", (model,))
    cur.execute("DELETE FROM functions WHERE model=?", (model,))
    cur.execute("DELETE FROM files WHERE model=?", (model,))
    cur.execute("DELETE FROM models WHERE name=?", (model,))


def _index_model(conn: sqlite3.Connection, model_dir: Path,
                 root: Path, force: bool = False) -> tuple[bool, int, int]:
    """Return (rebuilt, file_count, function_count)."""
    model = model_dir.name
    cc_data = parse_compile_db(model_dir / CC_NAME)
    fp = _fingerprint(model_dir, cc_data)

    cur = conn.cursor()
    row = cur.execute("SELECT fingerprint FROM models WHERE name=?",
                      (model,)).fetchone()
    if row and row[0] == fp and not force:
        counts = cur.execute(
            "SELECT COUNT(*), (SELECT COUNT(*) FROM functions WHERE model=?) "
            "FROM files WHERE model=?",
            (model, model),
        ).fetchone()
        return False, counts[0] or 0, counts[1] or 0

    linked, discarded = parse_map(model_dir / MAP_NAME)
    axf = _find_first_axf(model_dir)
    axf_syms = parse_axf(axf) if axf else {}

    _delete_model(conn, model)

    by_name = defaultdict(list)
    pending_calls = []
    file_count = 0

    for src, info in cc_data.items():
        try:
            text = src.read_text(encoding="utf-8", errors="ignore")
        except OSError:
            continue
        rel = _to_rel(src, root)
        cur.execute(
            "INSERT OR IGNORE INTO files(model, rel_path, abs_path) VALUES (?,?,?)",
            (model, rel, str(src)),
        )
        file_id = cur.execute(
            "SELECT id FROM files WHERE model=? AND rel_path=?",
            (model, rel),
        ).fetchone()[0]
        file_count += 1
        defines_json = json.dumps(sorted(info["defines"]))

        for name, start, end in find_functions(text):
            body = "\n".join(text.split("\n")[start - 1:end])
            in_axf  = 1 if name in axf_syms else 0
            in_map  = 1 if (not linked or name in linked) else 0
            removed = 1 if name in discarded else 0
            built   = 1 if ((in_axf or in_map) and not removed) else 0
            addr    = axf_syms.get(name, {}).get("addr")

            cur.execute(
                """INSERT INTO functions
                   (model, file_id, name, start_line, end_line, body,
                    built, in_axf, in_map, discarded, addr, tu_defines)
                   VALUES (?,?,?,?,?,?,?,?,?,?,?,?)""",
                (model, file_id, name, start, end, body,
                 built, in_axf, in_map, removed, addr, defines_json),
            )
            fid = cur.lastrowid
            by_name[name].append(fid)
            callees = set()
            for m in CALL_RE.finditer(body):
                nm = m.group(1)
                if nm in C_KEYWORDS or nm == name:
                    continue
                callees.add(nm)
            pending_calls.append((fid, sorted(callees)))

    for fid, callees in pending_calls:
        for cname in callees:
            if cname not in by_name:
                continue
            cur.execute(
                "INSERT OR IGNORE INTO calls(model, caller_id, callee_name) VALUES (?,?,?)",
                (model, fid, cname),
            )

    fn_count = cur.execute(
        "SELECT COUNT(*) FROM functions WHERE model=?", (model,)
    ).fetchone()[0]

    cur.execute(
        """INSERT INTO models(name, build_dir, fingerprint, indexed_at)
           VALUES (?,?,?,?)
           ON CONFLICT(name) DO UPDATE SET
              build_dir=excluded.build_dir,
              fingerprint=excluded.fingerprint,
              indexed_at=excluded.indexed_at""",
        (model, str(model_dir), fp, int(time.time())),
    )
    conn.commit()
    return True, file_count, fn_count


def _discover_models(build_root: Path):
    if not build_root.is_dir():
        return []
    found = []
    for child in sorted(build_root.iterdir()):
        if child.is_dir() and (child / CC_NAME).exists():
            found.append(child)
    return found


# ---------------------------------------------------------------------------
# Search
# ---------------------------------------------------------------------------
def _glob_match(pattern: str, text: str) -> bool:
    rx = "^" + re.escape(pattern).replace(r"\*", ".*").replace(r"\?", ".") + "$"
    return re.match(rx, text) is not None


def _strip_comments(text: str) -> str:
    text = re.sub(r"/\*.*?\*/", "", text, flags=re.DOTALL)
    text = re.sub(r"//[^\n]*", "", text)
    return text


def _search(conn, model, content, include_comment, define_filter=None):
    patterns = [p.strip() for p in content.split("|") if p.strip()]
    cur = conn.cursor()
    rows = cur.execute(
        """SELECT f.id, f.name, f.start_line, f.end_line, f.body,
                  f.tu_defines, fl.rel_path, fl.abs_path
           FROM functions f JOIN files fl ON fl.id=f.file_id
           WHERE f.model=? AND f.built=1""",
        (model,),
    ).fetchall()
    hits = []
    for fid, name, start, end, body, defs_json, rel, abs_path in rows:
        if define_filter:
            want = define_filter.split("=", 1)[0]
            defs = json.loads(defs_json or "[]")
            if not any(d.split("=", 1)[0] == want for d in defs):
                continue
        hay = body if include_comment else _strip_comments(body)
        if any(_glob_match(p, name) or _glob_match(p, hay) for p in patterns):
            hits.append({"id": fid, "name": name, "start": start,
                         "end": end, "body": body, "rel": rel, "abs": abs_path})
    return hits


def _neighbors(conn, model, name, direction):
    cur = conn.cursor()
    if direction == "callers":
        return cur.execute(
            """SELECT DISTINCT f.name, fl.rel_path, f.start_line
               FROM calls c
               JOIN functions f ON f.id=c.caller_id
               JOIN files fl ON fl.id=f.file_id
               WHERE c.model=? AND c.callee_name=?""",
            (model, name),
        ).fetchall()
    return cur.execute(
        """SELECT DISTINCT c.callee_name, fl.rel_path, f.start_line
           FROM calls c
           JOIN functions f ON f.id=c.caller_id
           JOIN files fl ON fl.id=f.file_id
           WHERE c.model=? AND f.name=?""",
        (model, name),
    ).fetchall()


def _render_tree(conn, model, root, direction, depth, seen=None, indent=0):
    if seen is None:
        seen = set()
    if depth <= 0 or root in seen:
        return []
    seen.add(root)
    out = []
    for name, rel, line in _neighbors(conn, model, root, direction):
        out.append("  " * indent + f"- {name} ({rel}:{line})")
        out.extend(_render_tree(conn, model, name, direction, depth - 1,
                                seen, indent + 1))
    return out


# ---------------------------------------------------------------------------
# Actions
# ---------------------------------------------------------------------------
def _ensure_model_dir(model: str) -> Path:
    model_dir = _workspace_root() / BUILD_DIR / model
    if not (model_dir / CC_NAME).exists():
        raise CodeSearchError(
            f"ERROR: model '{model}' not found at {BUILD_DIR}/{model}/ "
            f"(no {CC_NAME})"
        )
    return model_dir


def _refresh_model(conn, model_dir: Path, root: Path) -> tuple[bool, int, int]:
    return _index_model(conn, model_dir, root, force=False)


def action_list_models(conn) -> str:
    rows = conn.execute(
        "SELECT name, indexed_at FROM models ORDER BY name"
    ).fetchall()
    if not rows:
        return ("# Indexed Models\n\n"
                "_No models have been indexed yet._\n\n"
                f"Run `code-search-index` (or point the skill at a workspace "
                f"with a `{BUILD_DIR}/` directory containing model "
                f"subfolders with `{CC_NAME}`).")
    lines = ["# Indexed Models", ""]
    for name, ts in rows:
        fc = conn.execute(
            "SELECT COUNT(*) FROM files WHERE model=?", (name,)
        ).fetchone()[0]
        fn = conn.execute(
            "SELECT COUNT(*) FROM functions WHERE model=?", (name,)
        ).fetchone()[0]
        stamp = time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime(ts))
        lines.append(f"- **{name}** — {fc} files, {fn} functions (indexed {stamp})")
    return "\n".join(lines)


def action_index(conn, args: dict) -> str:
    root = _workspace_root()
    build_root = root / BUILD_DIR
    model = args.get("model", "").strip()

    if model:
        dirs = [_ensure_model_dir(model)]
    else:
        dirs = _discover_models(build_root)
        if not dirs:
            raise CodeSearchError(
                f"ERROR: no models found under {BUILD_DIR}/ "
                f"(expected subdirs with {CC_NAME})"
            )

    lines = ["# Index Updated", ""]
    for mdir in dirs:
        rebuilt, fc, fn = _index_model(conn, mdir, root, force=True)
        tag = "rebuilt" if rebuilt else "unchanged"
        lines.append(f"- **{mdir.name}** — {fc} files, {fn} functions ({tag})")
    return "\n".join(lines)


def _refresh_if_stale(conn, model: str) -> None:
    model_dir = _ensure_model_dir(model)
    _refresh_model(conn, model_dir, _workspace_root())


def action_find(conn, args: dict) -> str:
    model = args.get("model", "").strip()
    content = args.get("content", "").strip()
    if not model or not content:
        raise CodeSearchError("ERROR: 'model' and 'content' are required")

    depth = _as_int(args.get("depth"), 2)
    line = _as_int(args.get("line"), 3)
    include_comment = _as_bool(args.get("include_comment"), False)
    define_filter = (args.get("define") or "").strip() or None

    _refresh_if_stale(conn, model)

    hits = _search(conn, model, content, include_comment, define_filter)

    header = (f'# code-search result="{content}" model="{model}" '
              f'depth={depth} line={line} '
              f'include_comment={str(include_comment).lower()}')
    if define_filter:
        header += f' define="{define_filter}"'

    if not hits:
        return header + "\n\n_no results_"

    blocks = [header]
    for h in hits:
        blocks.append("")
        blocks.append(f"## {h['rel']} : func {h['name']}() "
                      f": line {h['start']}-{h['end']}")
        try:
            full = Path(h["abs"]).read_text(encoding="utf-8",
                                            errors="ignore").split("\n")
        except OSError:
            full = h["body"].split("\n")
        lo = max(0, h["start"] - 1 - line)
        hi = min(len(full), h["end"] + line)
        blocks.append("```c")
        for i in range(lo, hi):
            blocks.append(f"{i + 1:5d} | {full[i]}")
        blocks.append("```")

        blocks.append(f"### function caller (depth {depth})")
        callers = _render_tree(conn, model, h["name"], "callers", depth)
        blocks.extend(callers if callers else ["- (none)"])

        blocks.append(f"### related function (depth {depth})")
        callees = _render_tree(conn, model, h["name"], "callees", depth)
        blocks.extend(callees if callees else ["- (none)"])

    return "\n".join(blocks)


def _graph_action(conn, args: dict, direction: str) -> str:
    model = args.get("model", "").strip()
    fn = args.get("function", "").strip()
    if not model or not fn:
        raise CodeSearchError(
            f"ERROR: 'model' and 'function' are required for {direction}"
        )
    depth = _as_int(args.get("depth"), 2)
    _refresh_if_stale(conn, model)

    title = ("Callers of" if direction == "callers" else "Callees of")
    lines = [f"# {title} `{fn}` — {model} (depth {depth})", ""]
    tree = _render_tree(conn, model, fn, direction, depth)
    lines.extend(tree if tree else ["- (none)"])
    return "\n".join(lines)


def action_callers(conn, args: dict) -> str:
    return _graph_action(conn, args, "callers")


def action_callees(conn, args: dict) -> str:
    return _graph_action(conn, args, "callees")


# ---------------------------------------------------------------------------
# Dispatch
# ---------------------------------------------------------------------------
_ACTIONS = {
    "list-models": lambda conn, args: action_list_models(conn),
    "index":       action_index,
    "find":        action_find,
    "callers":     action_callers,
    "callees":     action_callees,
}


def main() -> None:
    action = sys.argv[1] if len(sys.argv) > 1 else ""
    handler = _ACTIONS.get(action)
    if handler is None:
        print(f"ERROR: Unknown action: {action}", flush=True)
        sys.exit(1)
    args = _get_args()
    try:
        conn = _open_db()
        try:
            print(handler(conn, args), flush=True)
        finally:
            conn.close()
    except CodeSearchError as exc:
        print(str(exc), flush=True)
        sys.exit(1)
    except Exception as exc:
        print(f"ERROR: {exc}", flush=True)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

---
7. Installation

7.1 Create the skill directory

```bash
mkdir -p .decode/skills/code-search/tools
```

7.2 Save the three files

Source section Destination path
Section 4 .decode/skills/code-search/SKILLS.md
Section 5 .decode/skills/code-search/tools/registry.json
Section 6 .decode/skills/code-search/tools/code_search.py

7.3 Optional — enable .axf DWARF parsing

```bash
pip install pyelftools
```

Fallback order if pyelftools is not installed:

1. fromelf --text -s (if fromelf is on PATH) — armcc toolchain symbol dump.
2. map.txt only — search still works, but only functions present in the
   armlink map are considered "built".

7.4 Verify registration

```bash
# from the workspace root
python .decode/skills/code-search/tools/code_search.py list-models
```

Expected output when the workspace has no indexed models yet:

```
# Indexed Models

_No models have been indexed yet._

Run `code-search-index` (or point the skill at a workspace with a `build/`
directory containing model subfolders with `compile_commands.json`).
```

If you see this message, the skill is registered correctly and the DeCode
list_skills() call will include code-search.

---

8. Usage Examples

8.1 Discovery

```
code-search-list-models
```

Output:

```
# Indexed Models

- **MODEL-ABC** — 342 files, 1204 functions (indexed 2026-09-14T09:12:04Z)
- **MODEL-XYZ** — 318 files, 1158 functions (indexed 2026-09-14T09:10:51Z)
```

8.2 Build the index

```
code-search-index
```

or a single model:

```
code-search-index model=MODEL-ABC
```

Output:

```
# Index Updated

- **MODEL-ABC** — 342 files, 1204 functions (rebuilt)
- **MODEL-XYZ** — 318 files, 1158 functions (rebuilt)
```

8.3 Search

```
code-search-find model=MODEL-ABC content="*memory*|*_func" depth=2 line=3 include_comment=false
```

Output (markdown):

```
# code-search result="*memory*|*_func" model="MODEL-ABC" depth=2 line=3 include_comment=false

## src/sensors/temp.c : func sensor_read_memory() : line 42-78
```c
   39 | static int adc_ready(void) {
   40 |     return (ADC_SR & ADC_SR_EOC) != 0;
   41 | }
   42 | int sensor_read_memory(uint8_t ch, uint16_t *out) {
   43 |     if (!adc_ready()) return -1;
   ...
   78 | }
```
### function caller (depth 2)
- sensor_poll (src/sensors/sensor.c:14)
  - app_task (src/main.c:88)
- temp_task (src/sensors/temp.c:120)
### related function (depth 2)
- adc_ready (src/sensors/temp.c:39)
- i2c_write (src/i2c.c:55)
  - i2c_init (src/i2c.c:10)
```

8.4 Restrict to a feature flag

```
code-search-find model=MODEL-ABC content=calibrate define=FEATURE_A
```

Only functions whose TU was compiled with -DFEATURE_A (or -DFEATURE_A=...)
are returned. This is the mechanism for answering "what code exactly compiled
for this build condition".

8.5 Caller / callee trees

```
code-search-callers model=MODEL-ABC function=sensor_read_memory depth=3
code-search-callees model=MODEL-ABC function=sensor_read_memory depth=3
```

---

9. Reference Architecture Notes

9.1 Parity with the kanboard reference skill

Reference (kanboard) This skill (code-search)
.decode/skills/kanboard/SKILLS.md .decode/skills/code-search/SKILLS.md
.decode/skills/kanboard/tools/registry.json .decode/skills/code-search/tools/registry.json
.decode/skills/kanboard/tools/kanboard.py .decode/skills/code-search/tools/code_search.py
sys.argv[1] = action, sys.argv[2:] = key=value Same
Output = markdown, never raw JSON Same
Errors printed with ERROR: prefix Same
config.json for connection settings Not required — the tool uses workspace cwd
5 tools registered under kanboard-* 5 tools registered under code-search-*

9.2 How the skill is discovered

dump.md confirms the discovery path:

· src/decode/registry/external_tools.py::load_external_tools() walks the
  directory given by registry_dir and loads every registry.json it finds.
· src/decode/registry/external_skills.py::load_external_skills() walks
  decode_dir()/skills/<name>/SKILLS.md, parses its frontmatter, and calls
  load_external_tools(<skill_dir>/tools, origin=f"skill:{name}").
· src/decode/registry/external_skills.py::register_all_external() is the
  single entry point used by the CLI at startup.

Because our tools are named code-search-* and our skill name is code-search,
the skill loader automatically associates them:

```python
skill_tools_map = {
    "code-search": [t for t in all_tools if t.name.startswith("code-search-")]
}
```

No configuration is needed beyond dropping the three files in place.

9.3 Environment assumptions

Assumption Source
Workspace root contains a build/ directory This PRD §2.1
Each build/<MODEL>/ contains compile_commands.json CMake + Ninja + armcc
Optional: build/<MODEL>/map.txt armlink output
Optional: build/<MODEL>/*.axf Linked image
python is on PATH for the external executor DeCode runtime

If map.txt and .axf are both absent, all functions from
compile_commands.json are treated as "built". This is still better than
searching the raw source tree but does not catch linker-discarded symbols.

9.4 Extension Points

Feature How to add
Vector search Add embeddings(fid INTEGER, vec BLOB) table. Replace _search body with cosine top-K; keep the built=1 filter.
Preprocessed bodies Add preprocessed_body TEXT column, populate with cached armcc -E output keyed by sha1(abs_path + tu_defines + header mtimes).
Include graph Add includes(file_id INTEGER, header_rel TEXT) table populated from *.d / .ninja_deps. Invert for "who includes this header".
Reachability from entry point After indexing, BFS from Reset_Handler/main over calls. Store reachable INTEGER on functions.
Recent-change ranking Parse .ninja_log for per-object timestamps; add built_at INTEGER column. Boost recent hits.
Doxygen enrichment Parse Doxygen XML; join on (rel_path, start_line) to fill a doc_comment column.

9.5 Known Limitations

· Function extraction is heuristic. The regex / brace matcher handles the
  vast majority of embedded C but will occasionally mis-detect a
  macro-as-function or miss a function with __attribute__((...)) in odd
  positions. Replace find_functions with a tree-sitter-c parser if higher
  accuracy is needed — the rest of the tool does not care where the boundaries
  come from.
· Callee resolution is name-based, not scope-aware. Two static void init()
  functions in different files will be conflated in the calls table. For a
  stricter graph, use the .axf DWARF DW_AT_abstract_origin linkage or
  resolve via disassembly. This is usually acceptable for search and
  documentation.
· armcc v5 vs armclang v6 differ. v5 emits DWARF 3, v6 emits DWARF 4/5.
  pyelftools handles both, but inline function attribution differs. Adjust
  parse_axf if the DWARF DW_AT_decl_file resolution needs to be exact.
· No .d / .ninja_deps parsing by default. The include graph is not
  built. This is a deliberate scope choice to keep the initial skill small.

---

End of document.

```
