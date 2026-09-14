# Product Requirements Document: Playwright → DrissionPage Migration for `decode`

**Version:** 2.1 — Codebase-Aware, Clean-Break Revision
**Status:** Draft
**Target Release:** TBD
**Author:** Automation Team
**Last Updated:** 2026-09-14
**Supersedes:** v1.0 (generic), v2.0 (codebase-aware)

---

## 0. Migration Policy — Clean Break, No Legacy Code

This migration is a **clean break**. The following rules apply to every file, config key, CLI flag, and dependency touched by this PRD:

1. **No legacy aliases.** Every renamed config key, CLI flag, environment variable, and function is renamed **outright**. No `old_name = new_name` shims.
2. **No backward-compatibility branches.** No `if PLAYWRIGHT: ... else: ...` runtime switches. The Playwright code path is **deleted**, not retained.
3. **No deprecated config keys.** `CDP_PORT`, `COPILOT_NAV_TIMEOUT_MS`, and every other Playwright-era key are **removed** from `config.py`, `_CONFIG_SPEC`, and any `config.yaml` loader path. Loading a stale `config.yaml` that contains them must fail loudly (see §7.11).
4. **No dual-mode operation.** There is no "Playwright fallback" if DrissionPage fails. Failure is failure.
5. **No migration helpers.** No `migrate_config_legacy()`, no `--compat` flag, no `LEGACY_*` module.
6. **No `playwright`, `greenlet`, or `pyee`** in `pyproject.toml`, `requirements.txt`, the frozen build, or any import path.
7. **Renamed public APIs are renamed without aliases.** If a caller outside this repo used `BrowserBridge.bind_shared_browser`, they must update to `bind_shared_page`. This is documented in the release notes as a **breaking change**.
8. **`config.yaml` schema is versioned.** A new top-level key `schema_version: 2` is required. Schema v1 files are rejected at load time with an actionable error message pointing to the migration guide.

Any pull request that reintroduces a deprecated key, alias, or dual-path branch must be rejected at review.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Codebase Scope Analysis](#2-codebase-scope-analysis)
3. [Current Playwright Footprint](#3-current-playwright-footprint)
4. [Goals & Non-Goals](#4-goals--non-goals)
5. [Architecture Comparison](#5-architecture-comparison)
6. [Module-by-Module Migration Plan](#6-module-by-module-migration-plan)
7. [Feature Requirements](#7-feature-requirements)
8. [Migration Implementation Phases](#8-migration-implementation-phases)
9. [Acceptance Criteria](#9-acceptance-criteria)
10. [Risks & Mitigations](#10-risks--mitigations)
11. [Timeline](#11-timeline)
12. [Appendix A: Playwright → DrissionPage API Mapping](#appendix-a-playwright--drissionpage-api-mapping)
13. [Appendix B: File Migration Checklist](#appendix-b-file-migration-checklist)
14. [Appendix C: Dependency & Packaging Changes](#appendix-c-dependency--packaging-changes)

---

## 1. Executive Summary

The `decode` codebase (HEAD `d7e1d3e`, branch `dev`) is a CLI agent that drives a **Microsoft Edge browser via Playwright's CDP transport** to operate the M365 Copilot web UI as its primary LLM backend. It also exposes browser automation as a **tool** (`web`) to external skills and subagents, and ships a web GUI built on `pywebview` (which uses the Edge WebView2 runtime but is unrelated to the automation path).

This PRD defines the migration of the browser automation layer — comprising **8 core files in `src/decode/backend/` and 2 files in `src/decode/tools/browser/`** — from Playwright + Edge to DrissionPage, with a **clean break**: no legacy aliases, no deprecated keys, no dual-mode operation.

The migration must preserve the following non-trivial behaviors that the current code implements against Playwright specifically:

- Multi-tab ownership via `window.name` markers (`ManagedCopilotSession`)
- Shared CDP transport between the Copilot backend and external web tools (`BrowserBridge`)
- Response extraction via downloadable JSON attachments
- Response extraction via clipboard read of the Copy Response button
- Transport health checks and multi-tier recovery (`_transport_healthy`, `_recover_transport`)
- Profile preference injection (locale + clipboard permissions) into Edge's `Preferences` JSON
- Stale Edge process cleanup by profile path (PowerShell `Get-CimInstance` + `taskkill`)
- CDP port fallback across 6 candidate ports
- 42 KB of UI selector logic in `CopilotClient`

**Expected outcome**: PyInstaller `--onefile` size reduced from ~200 MB to **≤ 30 MB**, native multithreaded tab operation, and preserved behavior across all 91 tracked files in `src/`.

---

## 2. Codebase Scope Analysis

### 2.1 Repository Structure

The tracked tree under `src/decode/` contains 91 files across 12 packages:

| Package | Purpose | Playwright Coupling |
|---|---|---|
| `backend/` | Copilot backend, Edge launcher, client, parser, protocol | **High** (`backend.py`, `client.py`, `edge.py`, `managed_session.py`) |
| `backend/__init__.py` | Backend factory | None (indirect) |
| `backend/api.py` | OpenAI-compatible API backend | None |
| `backend/dummy.py` | Simulated backend for GUI tests | None |
| `backend/parser.py` | Response JSON schema validation | None |
| `backend/prompts.py` | Prompt construction | None |
| `backend/protocol.py` | `Backend` Protocol interface | None |
| `backend/response_contract.py` | Attachment filename validation | None |
| `chat/` | Chat loop, tools workflow, repair, questions | **Indirect** (calls backend methods) |
| `context/` | Context renderer + memory scan | None |
| `core/` | Config loader, messages, session manager | None |
| `executor/` | Tool execution thread pool | None |
| `gui_web/` | Browser GUI (pywebview + htmx) | None (uses WebView2, not automation) |
| `integrations/` | Redmine read-only client | None |
| `registry/` | Tool/skill registry, external loaders | None |
| `tools/` | Builtin tools (fileops, grep, shell, vcs, lsp, subagent) | None |
| `tools/browser/` | Web tool + CDP bridge | **High** (`bridge_browser.py`, `web.py`) |
| `utils/` | Atomic writes, diagnostics, path helpers | None |
| `config.py`, `cli.py`, `python_worker.py` | Top-level entry | **Indirect** (configures CDP port, headless, etc.) |

### 2.2 Files Requiring Migration

**Primary (contains Playwright imports and/or CDP logic):**

| File | LOC (approx) | Size | Role |
|---|---|---|---|
| `src/decode/backend/backend.py` | ~640 | 25 KB | `CopilotBackend` — lifecycle, transport recovery, turn orchestration |
| `src/decode/backend/client.py` | ~1050 | 42 KB | `CopilotClient` — Copilot UI selectors, prompt, response extraction |
| `src/decode/backend/edge.py` | ~430 | 15 KB | `EdgeLauncher` — process launch, port fallback, stale process kill, profile prefs |
| `src/decode/backend/managed_session.py` | ~90 | 2 KB | `ManagedCopilotSession` — `window.name` marker ownership |
| `src/decode/tools/browser/bridge_browser.py` | ~400 | 15 KB | `BrowserBridge` — shared CDP connection, fetch/markdown extraction |
| `src/decode/tools/browser/web.py` | ~130 | 4.5 KB | `web` tool — fetch and tab control |

**Secondary (imports `_BRIDGE` or interacts with transport):**

| File | Change Required |
|---|---|
| `src/decode/backend/__init__.py` | Unchanged public API |
| `src/decode/backend/dummy.py` | Unchanged |
| `src/decode/cli.py` | Rename `--cdp-port` → `--local-port` (no alias) |
| `src/decode/config.py` | Rename `CDP_PORT` → `DRISSION_LOCAL_PORT`; add DrissionPage-specific keys; remove all Playwright-era keys |
| `src/decode/core/config_loader.py` | Update `_CONFIG_SPEC`; enforce `schema_version: 2` |
| `src/decode/registry/external_tools.py` | External tools may import `bridge_browser` |
| `src/decode/tools/subagents/subagent.py` | Subagent backend instantiation stays the same |

**Out of scope:**

- `gui_web/` — uses `pywebview` with `gui="edgechromium"` (WebView2). This is an **embedded browser host**, not Playwright automation. No change.
- `backend/api.py`, `backend/dummy.py`, `backend/parser.py` — backend-neutral.
- All `tools/*` except `tools/browser/`.

### 2.3 Threading Model (As-Built)

The current architecture is **single-threaded at the browser layer**:

- One Playwright `page` per `CopilotBackend` instance.
- Each backend instance creates its own Playwright `sync_playwright()` transport, but *connects to the same Edge CDP endpoint* as the parent via `connect_over_cdp()`.
- Subagents instantiate a **second** `CopilotBackend` bound to a **different tab** of the same Edge process.
- Tab ownership is claimed via `window.name` marker `decode:{role}:{session_id}:{instance_id}`.
- The `BrowserBridge` **borrows** the parent's `browser` object and opens *additional* tabs on the same transport.

This model must be preserved in DrissionPage. DrissionPage's `ChromiumPage.get_tab()` and `new_tab()` return `ChromiumTab` handles that are *independently* usable across threads, which is a **strict improvement** over Playwright's thread-affine synchronous API.

---

## 3. Current Playwright Footprint

### 3.1 Playwright API Surface in Use

Extracted from `backend.py`, `client.py`, `edge.py`, `bridge_browser.py`, `web.py`:

| Playwright API | Where Used | DrissionPage Equivalent |
|---|---|---|
| `sync_playwright().start()` | `backend._connect_transport`, `bridge._ensure_browser` | Not needed |
| `chromium.connect_over_cdp(url, headers, timeout)` | `backend._connect_transport` | `ChromiumOptions.set_local_port()` + `ChromiumPage(co)` |
| `browser.contexts` | `backend._connect_transport`, `bridge._browser_context` | `page.tab_ids` |
| `browser.new_context()` | `backend._connect_transport` | N/A — DrissionPage uses browser context implicitly |
| `context.pages` | `backend._connect_transport`, `_reset_context_tabs` | `page.get_tabs()` |
| `context.new_page()` | `backend._connect_transport`, `_reset_context_tabs`, `bridge` | `page.new_tab()` |
| `page.goto(url, timeout, wait_until)` | `client.open`, `client.new_chat`, `web.web` | `tab.get(url, timeout=N)` |
| `page.wait_for_load_state(state, timeout)` | `client.open`, `client.new_chat` | `tab.wait.load_start()` |
| `page.wait_for_timeout(ms)` | Throughout `client.py` | `tab.wait(N)` |
| `page.wait_for_function(js, arg, timeout)` | `client.attach_files` | Poll via `tab.run_js()` |
| `page.evaluate(js)` | `client._failure_diagnostics`, `client.response_snapshot`, `client._generation_status`, `managed_session.read_marker`, `web._page_text` | `tab.run_js(js)` |
| `page.locator(selector).first` | Throughout `client.py` | `tab.ele(selector, timeout=N)` |
| `page.locator(selector).count()` | `client.reply_count`, `_copy_response_candidates` | `len(tab.eles(selector))` |
| `page.locator(selector).nth(i)` | `_copy_response_candidates`, `_download_latest_reply_attachments` | `tab.eles(selector)[i]` |
| `locator.wait_for(state="visible", timeout)` | `client._activate_without_hover`, `set_model`, `activate_agent` | `tab.ele(selector, timeout=N)` returns `None` on timeout |
| `locator.focus()` + `locator.press("Enter")` | `client._activate_without_hover` | `ele.focus()` + `ele.input(Keys.ENTER)` |
| `locator.click(force=True)` | `client._get_text_via_copy_response_button` | `ele.click(by_js=False)` |
| `locator.scroll_into_view_if_needed()` | `client._get_text_via_copy_response_button` | `ele.scroll.to_see()` |
| `locator.is_visible()` | Throughout | `ele.states.is_displayed` |
| `locator.is_closed()` | `backend.shutdown`, `_health_snapshot` | `tab.states.is_alive` |
| `locator.fill(text)` | `client.send_prompt` | `ele.clear()` + `ele.input(text)` |
| `locator.set_input_files(paths)` | `client.attach_files` | `ele.input(paths)` (see §7.3) |
| `locator.inner_text()` | `client._model_switcher_value`, `web._page_text` | `ele.text` |
| `locator.get_attribute(name)` | `client._model_switcher_value`, `_select_model_once` | `ele.attr(name)` |
| `page.keyboard.press("Enter")` / `insert_text` / `type` | `client.send_prompt`, `activate_agent` | `tab.actions.type()` / `tab.actions.key()` |
| `page.get_by_role(role, name=...)` | `client._activate_without_hover`, `_temporary_chat_is_active`, `_agent_is_active` | Build equivalent XPath/CSS or use `tab.ele('@role=...')` |
| `page.get_by_role(...).filter(has_text=...)` | `client.activate_agent` | `tab.eles('@role=menuitem')` + filter by text |
| `page.screenshot(path, full_page)` | `client._failure_diagnostics` | `tab.get_screenshot(path=..., full_page=True)` |
| `page.expect_download()` context manager | `client._download_latest_reply_attachments` | **No direct equivalent — see §7.4** |
| `download.save_as(target)` | `client._download_latest_reply_attachments` | `tab.download(url, save_path)` or cookie-based fetch |
| `download.suggested_filename` | `client._download_latest_reply_attachments` | Parse from `Content-Disposition` or `download` event |
| `browser.is_connected()` | `backend._health_snapshot` | `tab.states.is_alive` + CDP ping via `tab.run_js('1')` |
| `browser.close()` | `bridge.shutdown` | `page.quit()` (but see caveat §7.7) |
| `playwright.stop()` | `backend._dispose_transport`, `bridge.shutdown` | No equivalent — DrissionPage keeps the connection alive |

### 3.2 Non-Standard Behaviors That Must Be Reimplemented

These are **not** covered by Playwright→DrissionPage line-by-line translation and require explicit design:

1. **`window.name` marker ownership** (`managed_session.py`) — Playwright reads/writes `window.name` via `page.evaluate("() => window.name")`. DrissionPage's `tab.run_js("return window.name")` works equivalently. **Low risk.**
2. **`expect_download()` context manager** (`client._download_latest_reply_attachments`) — Playwright intercepts the CDP `Browser.downloadWillBegin` event. DrissionPage exposes downloads via `page.set.download_path()` and `page.wait.download_begin()`. **Medium risk.**
3. **`set_input_files()` for file attachments** (`client.attach_files`) — Playwright injects file paths into `<input type="file">` via CDP `DOM.setFileInputFiles`. DrissionPage supports the same via `ele.input(files)`. **Low risk.**
4. **Clipboard read** (`client._get_text_via_copy_response_button`) — Playwright calls `navigator.clipboard.readText()` via `page.evaluate()`. DrissionPage's `run_js` does the same. **Low risk**, but clipboard permissions must still be granted in Edge's profile (`edge._configure_profile_language`).
5. **Stale Edge process cleanup** (`edge._kill_stale_processes`) — Windows-native PowerShell + `taskkill`, not Playwright. **No change required.**
6. **Port fallback across 6 candidates** (`edge.start`) — `socket`-based, not Playwright. **No change required**, but the resolved port must be passed to `ChromiumOptions.set_local_port()` *before* `ChromiumPage(co)` is constructed.
7. **`BrowserBridge.bind_shared_browser`** — Borrowing Playwright's `browser` object across modules. DrissionPage's shared browser is a `ChromiumPage` singleton — sharing is via `ChromiumPage(addr_or_opts)` reconnect, not object passing. **Medium risk** — the bridge's semantics change from "reuse this object" to "reconnect to the same CDP endpoint."
8. **Subagent tab ownership** — Playwright `context.new_page()` creates a fresh page. DrissionPage `page.new_tab()` does the same. Markers via `window.name` still work. **Low risk** but each subagent must *not* call `ChromiumPage(co)` again with a fresh `local_port` — that would spawn a new Edge.

---

## 4. Goals & Non-Goals

### 4.1 Goals

1. Replace all Playwright imports in `backend/` and `tools/browser/` with DrissionPage.
2. Preserve the existing `Backend` Protocol (`protocol.py`) — no changes to `chat/`, `executor/`, or `registry/` public interfaces.
3. Reduce PyInstaller `--onefile` executable from ~200 MB to ≤ 30 MB.
4. Enable true multithreaded tab operation where subagents and the parent operate on independent `ChromiumTab` handles without `sync_playwright()` contention.
5. Preserve all 8 non-standard behaviors listed in §3.2.
6. Keep the `web` tool (`tools/browser/web.py`) API identical: `web(action=..., url=..., format=..., selector=..., value=..., wait=..., screenshot_mode=...)`.
7. Preserve `DummyEdgeBackend` for GUI tests (no browser dependency).
8. Preserve the API backend (`ApiBackend`) unchanged.
9. **Delete** all Playwright-era config keys, CLI flags, env vars, and deprecated aliases (see §0).

### 4.2 Non-Goals

- Migrating `gui_web/` off `pywebview` — that's WebView2, not automation.
- Migrating the Copilot web UI selectors (`SELECTORS` dict in `client.py`) — they target M365 Copilot's DOM, not Playwright.
- Adding new browser automation features (screenshots to Markdown, etc.).
- Cross-browser support — Edge only.
- Refactoring the chat loop, tool workflow, or context renderer.
- Migrating `tools/subagents/subagent.py`'s isolation model.

---

## 5. Architecture Comparison

### 5.1 Current (Playwright) — Parent + Subagent + Bridge
