# Product Requirements Document: Playwright → DrissionPage Migration

**Version:** 1.0
**Status:** Draft
**Target Release:** TBD
**Author:** Automation Team
**Last Updated:** 2026-09-14

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Background & Problem Statement](#2-background--problem-statement)
3. [Goals & Non-Goals](#3-goals--non-goals)
4. [Architecture Comparison](#4-architecture-comparison)
5. [Feature Requirements](#5-feature-requirements)
6. [Migration Implementation Plan](#6-migration-implementation-plan)
7. [Code Architecture](#7-code-architecture)
8. [Acceptance Criteria](#8-acceptance-criteria)
9. [Risks & Mitigations](#9-risks--mitigations)
10. [Timeline](#10-timeline)
11. [Appendix A: API Mapping](#appendix-a-playwright--drissionpage-api-mapping)
12. [Appendix B: Dependencies](#appendix-b-dependency-list)

---

## 1. Executive Summary

This PRD defines the migration of the existing web automation system from **Playwright + Edge** to **DrissionPage** on Windows. The migration addresses three critical pain points:

1. **Large PyInstaller build size** caused by Playwright's bundled browser binaries and Node.js dependencies.
2. **Slow startup and execution overhead** from Playwright's three-layer process architecture.
3. **Limited native multithreading support** for independent tab control.

The target system will use DrissionPage's native Python CDP implementation, achieving a packaged executable of approximately **14 MB versus Playwright's 200+ MB**, while gaining native tab-level thread safety and preserving all existing automation workflows including profile management, tab automation, screenshot capture, and Markdown conversion.

---

## 2. Background & Problem Statement

### 2.1 Current Architecture

The existing system uses Playwright with Microsoft Edge on Windows. Playwright's architecture relies on a Node.js driver process that communicates with the browser via the Chrome DevTools Protocol (CDP). While Playwright provides excellent cross-browser support and modern async APIs, it introduces significant overhead for a Windows-only Edge automation scenario:

- **Build size**: Playwright bundles Chromium, Firefox, and WebKit browser binaries, plus a Node.js runtime, resulting in PyInstaller executables exceeding 200 MB.
- **Startup latency**: The Node.js driver adds process spawn overhead on every browser launch.
- **Tab management**: Playwright's `browser.new_page()` model does not natively support concurrent operations on multiple tabs from separate Python threads due to its async event loop architecture.
- **Dependency footprint**: Playwright requires `greenlet`, `pyee`, and other transitive dependencies that inflate the frozen build.

### 2.2 Why DrissionPage

DrissionPage is a Python-native web automation library that controls Chromium-based browsers directly via CDP without an intermediate driver process. Key advantages for this migration:

| Dimension | Playwright | DrissionPage |
|---|---|---|
| **Runtime** | Node.js driver + Python bindings | Pure Python + CDP WebSocket |
| **PyInstaller size** | ~200+ MB | ~14 MB (per official docs) |
| **Startup speed** | Slower (driver spawn) | Faster (direct CDP connection) |
| **Tab concurrency** | Async event loop constraints | Native `get_tab()` per thread |
| **HTTP fallback** | Not available | `WebPage` supports SessionPage mode |
| **Screenshot** | `page.screenshot()` | `get_screenshot(full_page=True)` |
| **Profile management** | `launch_persistent_context()` | `ChromiumOptions.set_user_data_path()` |

The official DrissionPage packaging guide confirms that a minimal environment with only DrissionPage and PyInstaller produces an executable around 14 MB.

### 2.3 Migration Constraints

- **Target browser**: Microsoft Edge (Chromium) on Windows 10/11.
- **Must preserve**: User profile creation and management, tab lifecycle automation, screenshot capture (full-page and element-level), HTML-to-Markdown conversion.
- **Must improve**: Multithreaded tab independence, PyInstaller build size, overall execution speed.
- **Library preferences**: DrissionPage for automation; `markdownify` or `html2text` for Markdown conversion (both must be evaluated).

---

## 3. Goals & Non-Goals

### 3.1 Goals

1. Replace all Playwright browser automation calls with DrissionPage equivalents.
2. Reduce PyInstaller `--onefile` executable size from 200+ MB to under 30 MB.
3. Enable true multithreaded tab management where each thread operates on an independent tab without cross-thread interference.
4. Preserve and enhance profile management (create, list, select, persist Edge profiles).
5. Implement HTML-to-Markdown conversion using `markdownify` or `html2text`, with a configurable adapter.
6. Maintain or improve screenshot capture quality (full-page, viewport, element-level).
7. Reduce browser launch and page interaction latency.

### 3.2 Non-Goals

- Cross-browser support (Firefox, WebKit) is explicitly out of scope.
- Migration to DrissionPage's `SessionPage` (HTTP-only) mode is optional and will be treated as a future enhancement.
- Playwright compatibility shim or dual-mode operation is not required.
- Mobile device emulation is not required.

---

## 4. Architecture Comparison

### 4.1 Playwright Architecture (Current)
