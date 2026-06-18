# Aura2 — Session Changelog (April 8, 2026)

> Comprehensive record of all changes made during the April 8, 2026 development session.
> Commit: `8e11962` on `refined` branch. Pushed to both `origin` (GitHub) and `samsung`.

---

## Table of Contents

1. [Overview](#overview)
2. [WS1: Figma JSON Persistence](#ws1-figma-json-persistence)
3. [WS2: Structural Validation](#ws2-structural-validation-beyond-screenshots)
4. [WS3: CSS Modules Support](#ws3-css-modules-support)
5. [WS4: RAG & Component Reuse](#ws4-rag--component-reuse-strengthening)
6. [WS5: Documentation](#ws5-documentation)
7. [WS6: Stability & Deployment Polish](#ws6-stability--deployment-polish)
8. [WS7: Demo Preparation](#ws7-demo-preparation)
9. [Figma JSON Viewer (Frontend)](#figma-json-viewer-frontend)
10. [Conversion Tracing & Langfuse](#conversion-tracing--langfuse-integration)
11. [CSS Modules Prompt Overhaul](#css-modules-prompt-overhaul)
12. [Figma Plugin Fixes](#figma-plugin-fixes)
13. [Preview Bug Fix](#preview-bug-fix)
14. [Bug Fixes (21 total)](#bug-fixes-21-total)
15. [Files Changed Summary](#files-changed-summary)

---

## Overview

**83 files changed, 11,555 lines added, 416 lines removed.**

This session implemented 7 planned workstreams plus additional bug fixes, a frontend Figma JSON viewer, conversion tracing with Langfuse integration, a CSS Modules prompt overhaul, and a preview bug fix. Five parallel code review agents identified 21 bugs across the codebase, all of which were fixed.

---

## WS1: Figma JSON Persistence

**Problem:** Figma design data was in-memory only during conversion — Samsung devs couldn't inspect it, and it couldn't be used for post-conversion comparison.

**Solution:** Save raw Figma JSON into each project directory for every export.

### New Files
- `backend/agents/_figma_to_react/figma_json_persistence.py`
  - `save_figma_json(project_path, raw_data, design_data, source, file_key)` → creates `{project}/figma_data/`
  - Writes 3 files: `raw_figma_response.json`, `design_data.json`, `design_metadata.json`

### Modified Files
- `backend/agents/figma_to_react.py` — calls `save_figma_json` in both REST API and plugin conversion paths
- `backend/main.py` — new endpoints:
  - `GET /api/projects/{id}/figma-json` → returns design_data.json
  - `GET /api/projects/{id}/figma-json/raw` → returns raw_figma_response.json

### Where to Find Data
```
generated_projects/{name}/figma_data/
├── design_data.json         # Normalized design data
├── raw_figma_response.json  # Raw API/plugin payload
└── design_metadata.json     # Timestamp, source, stats
```

---

## WS2: Structural Validation (Beyond Screenshots)

**Problem:** Validation was vision-only (screenshot comparison). No structured checks for dimensions, padding, colors, fonts.

**Solution:** Compare Figma JSON properties against rendered DOM computed styles via `data-figma-id` attribute mapping.

### New Files
- `backend/utils/structural_comparison.py`
  - `compare_structural_properties(port, design_data, project_path)` → structured report
  - Uses Playwright to inject JS that extracts `getComputedStyle()` from DOM elements with `data-figma-id`
  - Compares: width, height, padding, margin, gap, background-color, color, font-size, font-weight, border-radius, box-shadow, flex-direction
  - Tolerance: ±2px for dimensions, ±5 per RGB channel for colors

### Modified Files
- `backend/agents/_figma_to_react/verification.py` — structural comparison as 3rd check alongside vision and content
  - Updated confidence: 50% vision + 30% structural + 20% content
- `backend/agents/_figma_to_react/prompt_generation.py` — instructs agent to add `data-figma-id` attributes
- `backend/config.py` — new settings: `enable_structural_comparison`, `structural_comparison_tolerance_px`, `color_comparison_tolerance`

---

## WS3: CSS Modules Support

**Problem:** Samsung teams needed CSS Modules as an alternative to inline Tailwind, configurable per project.

**Solution:** Added CSS Modules as a fourth UI library option with a full template and detailed prompt instructions.

### New Files
- `templates/react-css-modules/` — 17 files
  - Cloned from react-tailwind, removed Tailwind deps, clean CSS reset in index.css
  - Vite supports CSS Modules natively for `*.module.css`

### Modified Files
- `backend/config.py` — `CSS_MODULES = "css-modules"` in UILibrary enum
- `backend/agents/_figma_to_react/project_setup.py` — `"css-modules": "react-css-modules"` in template_map
- `backend/agents/_figma_to_react/prompt_generation.py` — detailed CSS Modules branch with:
  - Complete Figma→CSS property mapping table (22 properties)
  - Responsive layout patterns with `@media` queries
  - Design tokens via CSS custom properties
  - Shadow/effect translation to CSS syntax
- `frontend/src/components/ProjectForm.tsx` — "CSS Modules" dropdown option
- `figma-plugin/src/ui.html` — "CSS Modules" dropdown option
- `frontend/src/types/index.ts` — `'css-modules'` in union type

---

## WS4: RAG & Component Reuse Strengthening

**Problem:** Reuse thresholds too strict, no clear stats per page, no reuse tracking API.

### Modified Files
- `backend/rag/component_store.py`
  - Lowered "reuse_directly" threshold: 0.9 → 0.85
  - Added `get_reuse_stats()` — total components, reuse counts, top-reused
  - Added `track_decision(component_id, decision, project_id)` — records reuse/adapt/new
  - Added guard for empty ChromaDB collections
- `backend/mcp_tools/component_library.py`
  - New MCP tool: `get_reuse_report` — returns library reuse stats
  - Threshold aligned to 0.85
  - Fixed `_reset_store()` — was using invalid `collection.delete(where={})`
- `backend/agents/figma_to_react.py` — collects reuse_stats in result dict
- `backend/storage/project_store.py` — new fields: `components_adapted`, `reuse_stats`
- `backend/main.py` — `GET /api/projects/{id}/reuse-stats` endpoint

---

## WS5: Documentation

### New Files
- `docs/04_V2_ARCHITECTURE.md` — Claude Agent SDK integration, MCP topology, agent flow, data flow
- `docs/05_VALIDATION_PIPELINE.md` — Three-tier validation, confidence scoring, auto-fix loop
- `docs/06_HOW_TO_VERIFY_ACCURACY.md` — Developer guide: where to find JSON, generated code, how to spot-check

---

## WS6: Stability & Deployment Polish

### New Files
- `backend/utils/image_optimizer.py`
  - `optimize_project_images(project_path, max_size_kb=500)` — Pillow-based
  - JPEG quality reduction, PNG→WebP conversion, safe temp-file writes
  - Called automatically before GitHub push and Vercel deploy

### Modified Files
- `backend/main.py`
  - Image optimization before GitHub push and Vercel deploy
  - Improved error messages: 401→"Token expired", 403→"Permission denied", 413→"Payload too large"
  - `GET /api/figma/rate-limit-status` endpoint
- `backend/agents/_figma_to_react/figma_api.py` — actionable rate-limit messages mentioning plugin alternative
- `backend/utils/figma_rate_limiter.py` — `get_rate_limit_status()` method
- `templates/react-tailwind/.github/workflows/ci.yml` — env variables, comment block
- `requirements.txt` — added `Pillow>=10.0.0`, `langfuse>=4.0.0`

---

## WS7: Demo Preparation

### New Files
- `demos/example-conversion/` — 9 files
  - Synthetic Figma JSON (design_data, raw_response, metadata)
  - Generated React components (Header, Hero, Footer)
  - Property-by-property comparison report
- `demos/cross-page-reuse/` — 8 files
  - Page 1: creates Header, Footer, Button (saved to library)
  - Page 2: reuses Header, Footer; creates ContactForm
  - Reuse report JSON
- `demos/README.md` — overview

---

## Figma JSON Viewer (Frontend)

### New Files
- `frontend/src/components/FigmaJsonViewer.tsx`
  - Two tabs: "Design Data" / "Raw Figma Response"
  - Lazy-loaded (only fetches on click)
  - Stats badges (pages, frames, colors, fonts, images)
  - Collapsible JSON sections (pages, colors, fonts, images, full JSON)
  - Copy to clipboard + Download JSON buttons

### Modified Files
- `frontend/src/api/client.ts` — `getFigmaJson()`, `getFigmaJsonRaw()` methods
- `frontend/src/pages/ProjectDetailDistinctive.tsx` — integrated FigmaJsonViewer between Component Architecture and Project Information sections

---

## Conversion Tracing & Langfuse Integration

### New Files
- `backend/utils/trace_logger.py`
  - `ConversionTrace` class — captures per-conversion: system prompt, conversion prompt, agent messages (500-char previews), tool calls (200-char previews), token counts, cost
  - Saves to `{project}/trace/conversion_trace.json`
  - **Langfuse integration:** each conversion = one Langfuse trace with nested spans:
    - `generation: system-prompt` — full system prompt
    - `generation: conversion-prompt` — full design data prompt
    - `generation: agent-turn-N` — each assistant response
    - `span: tool:ToolName` — each tool call with args/result
    - `generation: total-usage` — final token counts + cost
  - Token usage captured from Claude Agent SDK's `ResultMessage.usage` and `total_cost_usd`
- `frontend/src/components/ConversionTraceViewer.tsx`
  - Collapsible card showing model, tokens, duration, tool count
  - Expandable sections: System Prompt, Conversion Prompt, Tool Calls
  - Copy/Download full trace JSON

### Modified Files
- `backend/config.py` — `enable_trace_logging`, `langfuse_public_key`, `langfuse_secret_key`, `langfuse_host`
- `backend/agents/figma_to_react.py` — trace hooks in `_run_agent_conversion()`:
  - Creates trace at start, logs prompts, messages, tool calls, tokens from ResultMessage
  - Saves trace on success and failure
- `backend/main.py` — `GET /api/projects/{id}/trace` endpoint
- `frontend/src/api/client.ts` — `getTrace()` method
- `frontend/src/pages/ProjectDetailDistinctive.tsx` — integrated ConversionTraceViewer

### Langfuse Setup
```env
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
```
Package: `langfuse>=4.0.0` (installed via `uv pip install langfuse`)

---

## CSS Modules Prompt Overhaul

**Problem:** The CSS Modules prompt produced less accurate output than Tailwind because "universal" sections in the system prompt used Tailwind-specific syntax (classes, hover: prefixes, gap-6, shadow-[...]), which confused the agent when generating CSS Modules code.

**Root Cause:** 6 sections in the system prompt were Tailwind-specific but treated as library-agnostic:
1. "RESPONSIVE DESIGN" — used `className="w-full max-w-7xl"`
2. "PIXEL-PERFECT SPACING" — mapped pixels to Tailwind scale
3. "SHADOW & EFFECT TRANSLATION" — Tailwind shadow syntax
4. "INTERACTIVE STATES" — JSX with `hover:opacity-90`
5. "ENHANCED LAYOUT PATTERNS" — `grid grid-cols-12 gap-4`
6. `format_frame_for_prompt()` — per-node hints like `hover:shadow-lg`

**Fix:**
- Made all 6 universal sections **library-agnostic** (describe *what* not *how*)
- Massively expanded CSS Modules prompt to match Tailwind's detail level:
  - 22-property Figma→CSS mapping table
  - Full responsive layout patterns with real CSS code
  - Container, header, hero, grid, navigation, footer patterns
  - Responsive typography with `@media` queries
  - Shadow/effect/gradient translation to CSS
  - Animation keyframes patterns
  - Design tokens system with CSS custom properties
- Fixed `format_frame_for_prompt()` to use generic descriptions instead of Tailwind classes

---

## Figma Plugin Fixes

### Modified Files
- `figma-plugin/src/ui.html`
  - Header: "Export design to React components" (was "React + Tailwind CSS")
  - XSS fix: `showStatus()` uses `textContent` by default, `innerHTML` only when spinner HTML needed
  - Safe success message: `msg.result?.project_id` with nullish fallback
- `figma-plugin/src/code.ts`
  - Double-submit guard: `isExporting` flag prevents concurrent exports
  - Global state reset: clean reassignment (`collectedColors = {}`) instead of error-prone manual delete loop
  - Per-page progress: shows "Processing page 1/3: Home" during extraction
- Plugin rebuilt with `npm run build`

---

## Preview Bug Fix

**Problem:** When clicking Preview on project A then navigating to project B, the old preview stayed visible.

**Root Cause:** `ProjectPreview` component kept old `previewUrl` in state. No state reset on project switch, no `key` prop to force iframe remount.

**Fix:**
- `frontend/src/components/ProjectPreview.tsx`
  - Added `useEffect` that resets all state when `project.id` changes
  - Added `key={project.id}-${previewUrl}` on iframe for forced remount
- `frontend/src/pages/ProjectDetail.tsx`
  - Added `key={project.id}` on `<ProjectPreview>` for full component remount

---

## Bug Fixes (21 total)

### Critical (would crash at runtime)

| # | File | Bug | Fix |
|---|------|-----|-----|
| 1 | `vision_comparison.py` | Sync `anthropic.Anthropic` blocked async event loop | Changed to `AsyncAnthropic` + `await` |
| 2 | `build_verifier.py` (x2) | `asyncio.wait_for` wrapped subprocess creation, not `.communicate()` — timeout never fired | Moved to wrap `communicate()` with kill on timeout |
| 3 | `dev_server_manager.py` | `process.terminate()` on Windows only killed npm wrapper, Vite child kept running → port leak | `_kill_process_tree()` with `taskkill /F /T /PID` |
| 4 | `image_optimizer.py` | Pillow `Image.open()` held file handle → `PermissionError` on Windows | `_safe_save()` with `img.load()` + temp file |
| 5 | `image_optimizer.py` | RGBA images crashed when saving as JPEG | Auto-convert RGBA→RGB in `_safe_save()` |
| 6 | `design_styles.py` | `effect["type"]` KeyError if Figma returned effect without type | Changed to `effect.get("type")` stored in local var |
| 7 | `design_styles.py` | `extract_strokes()` returned dict but callers iterated as list → `AttributeError` | Changed to return plain list |
| 8 | `structural_comparison.py` | f-string with JS braces → `ValueError` on every call | Rebuilt as string concatenation |
| 9 | `structural_comparison.py` | Missing `return` in `page.evaluate` → result always `undefined` | Added `return` before IIFE |
| 10 | `structural_comparison.py` | `cornerRadius` can be int → `TypeError: 'int' is not iterable` | `isinstance` guard |

### Important (correctness / edge cases)

| # | File | Bug | Fix |
|---|------|-----|-----|
| 11 | `dev_server_manager.py` | `_lock` held during `npm install` (up to 5 min) blocked all server ops | Moved install outside the lock |
| 12 | `dev_server_manager.py` | `_current_port` never reset → port range exhaustion | `find_free_port()` wraps around |
| 13 | `component_store.py` | ChromaDB `collection.query()` crashed on empty collection | Guard with `count() == 0` |
| 14 | `component_library.py` | `_reset_store()` passed empty `where={}` to ChromaDB | Use `_store.reset()` |
| 15 | `component_library.py` | Threshold mismatch (0.9 vs 0.85) | Aligned to 0.85 |
| 16 | `figma_rate_limiter.py` | Fallback `raise httpx.HTTPStatusError(request=None)` broke callers | Changed to `RuntimeError` |
| 17 | `plugin_conversion.py` | `width=0` treated as falsy → zero-dimension nodes lost bounds | `is not None` check |
| 18 | `prompt_generation.py` | Invalid CSS `box-shadow: ... offset` in focus example | Changed to `white` |
| 19 | `main.py` | `asyncio.run()` in background thread → `RuntimeError` if event loop running | `asyncio.new_event_loop()` |
| 20 | `useProjectStatus.ts` | Stale data kept polling forever when server returned errors | Added `query.state.status === "error"` check |
| 21 | `ProjectDetail.tsx` | `new Date(undefined)` threw `RangeError` if `created_at` missing | Guard with fallback text |

---

## Files Changed Summary

### New Files (32)
```
backend/agents/_figma_to_react/figma_json_persistence.py
backend/utils/image_optimizer.py
backend/utils/structural_comparison.py
backend/utils/trace_logger.py
frontend/src/components/FigmaJsonViewer.tsx
frontend/src/components/ConversionTraceViewer.tsx
templates/react-css-modules/ (17 files)
demos/ (18 files)
docs/04_V2_ARCHITECTURE.md
docs/05_VALIDATION_PIPELINE.md
docs/06_HOW_TO_VERIFY_ACCURACY.md
```

### Modified Files (32)
```
backend/agents/_figma_to_react/design_styles.py
backend/agents/_figma_to_react/figma_api.py
backend/agents/_figma_to_react/plugin_conversion.py
backend/agents/_figma_to_react/project_setup.py
backend/agents/_figma_to_react/prompt_generation.py
backend/agents/_figma_to_react/verification.py
backend/agents/figma_to_react.py
backend/config.py
backend/dev_server_manager.py
backend/main.py
backend/mcp_tools/component_library.py
backend/rag/component_store.py
backend/storage/project_store.py
backend/utils/build_verifier.py
backend/utils/figma_rate_limiter.py
backend/utils/vision_comparison.py
data/projects.json
figma-plugin/src/code.ts
figma-plugin/src/ui.html
frontend/src/api/client.ts
frontend/src/components/ProjectForm.tsx
frontend/src/components/ProjectPreview.tsx
frontend/src/hooks/useProjectStatus.ts
frontend/src/pages/ProjectDetail.tsx
frontend/src/pages/ProjectDetailDistinctive.tsx
frontend/src/types/index.ts
requirements.txt
templates/react-tailwind/.github/workflows/ci.yml
templates/react-tailwind/package.json
templates/react-chakra/package.json
templates/react-mui/package.json
```

### New API Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/projects/{id}/figma-json` | Returns processed design_data.json |
| GET | `/api/projects/{id}/figma-json/raw` | Returns raw Figma API response |
| GET | `/api/projects/{id}/reuse-stats` | Returns component reuse statistics |
| GET | `/api/projects/{id}/trace` | Returns conversion trace JSON |
| GET | `/api/figma/rate-limit-status` | Returns current Figma API rate limit status |

### New MCP Tools
| Tool | Purpose |
|------|---------|
| `get_reuse_report` | Returns component library reuse statistics |

### New Config Settings
```python
# Structural Comparison
enable_structural_comparison: bool = True
structural_comparison_tolerance_px: int = 2
color_comparison_tolerance: int = 5

# Tracing / Observability
enable_trace_logging: bool = True
langfuse_public_key: str = ""
langfuse_secret_key: str = ""
langfuse_host: str = "https://cloud.langfuse.com"
```
