# Plan: Per-Project CLAUDE.md + Token Optimization + main.py Refactor

## Problem Statement

Three related inefficiencies found during audit:

1. **Token waste on "Add Page"** — When adding a page to an existing project, the agent runs 6-10
   Read/Bash/Glob tools to understand the project structure (components, routes, design tokens) before
   writing a single line. This exploration costs ~3,000-5,000 tokens and 10-15 seconds every time.
   Fix: generate a `CLAUDE.md` inside each project at creation time so the agent reads one file
   instead of exploring.

2. **main.py is 1,395 lines** — One file handles: background tasks, 20+ API routes, Pydantic models,
   Vercel deploy logic, GitHub push logic, dev server proxy, and static file serving. Hard to navigate,
   hard to extend, and causes repeated full-file reads in every session.
   Fix: split into FastAPI routers by domain.

3. **System prompt is 21,000+ chars** — The same full system prompt is sent for both new projects and
   page additions, even though adding a page needs only a small fraction of those instructions.
   Fix: build a lean `add_page` system prompt variant that omits irrelevant sections.

---

## Part 1: Per-Project CLAUDE.md

### What it contains (generated at project creation)

```markdown
# Project: {name}

## Stack
- UI Library: {tailwind | mui | chakra}
- Router: React Router v6
- Build: Vite + TypeScript

## Structure
src/
  components/     # Shared components (reuse these first)
  pages/          # Route-level page components
  App.tsx         # Route definitions

## Existing Routes
- / → src/App.tsx (root)
{additional routes added as pages are created}

## Existing Components
{list of component files with one-line description each}

## Design Tokens
- Primary font: {font-family}
- Colors: {top 5 hex values with role}
- Tailwind config: tailwind.config.ts

## Adding a New Page
1. Create src/pages/{PageName}.tsx
2. Add route in src/App.tsx: <Route path="/{route}" element={<PageName />} />
3. Reuse components from src/components/ before creating new ones
```

### When it's generated
- **New project**: written by `setup_project_from_template` after copying template files
- **After page added**: updated by `_run_agent_conversion` to append the new route

### How it reduces tokens
Agent system prompt gains one instruction:
> "Read CLAUDE.md at the project root before exploring any files. It contains the complete project
> map. Do not re-read files that are already described there."

This replaces 6-10 tool calls with 1 Read call.

### Implementation steps
1. In `backend/agents/_figma_to_react/project_setup.py`: after template copy, call
   `_write_project_claude_md(project_path, project_name, ui_library)` — generates initial CLAUDE.md
   with empty routes/components sections.
2. In `backend/agents/figma_to_react.py` `_run_agent_conversion`: after agent finishes,
   call `_update_project_claude_md(project_path, add_as, project_name, route_path)` to append
   new page info.
3. In `backend/agents/_figma_to_react/prompt_generation.py` `get_system_prompt`:
   add instruction to read CLAUDE.md first when `add_as == "new_page"`.

---

## Part 2: main.py Split into Routers

### Current state
`backend/main.py` — 1,395 lines, 20 routes, background tasks, models, and utilities all mixed.

### Target structure
```
backend/
  main.py                    # ~150 lines: app init, lifespan, CORS, static files, health
  routers/
    projects.py              # CRUD: create, list, get, delete, cleanup, stats
    conversion.py            # /api/projects/create, /api/figma/plugin-upload, /api/projects/add-website
    dev_server.py            # start/stop dev server, preview URL, build
    deployment.py            # Vercel deploy, deployment status
    github.py                # GitHub push
  tasks/
    conversion_tasks.py      # run_conversion_sync, run_plugin_conversion_sync (moved from main.py)
    deployment_tasks.py      # _deploy_to_vercel_task (moved from main.py)
```

### Implementation steps
1. Create `backend/routers/` directory with `__init__.py`
2. Move routes grouped by domain into router files using `APIRouter`
3. Move background task functions into `backend/tasks/`
4. Update `main.py` to `include_router(...)` for each router
5. No API surface changes — all URLs stay the same

---

## Part 3: Lean System Prompt for Add-Page

### Current state
`get_system_prompt()` returns the same 21,000-char prompt regardless of whether we're creating a
new project or adding a page. When adding a page:
- "Project setup" instructions → irrelevant (project already exists)
- "Create tailwind.config.ts" → irrelevant
- "Initialize package.json" → irrelevant
- ~40% of the prompt is dead weight

### Target
`get_system_prompt(add_as="new_page")` returns a ~9,000-char prompt with only:
- Component reuse instructions (highest priority)
- Read CLAUDE.md first
- Route registration pattern
- Visual verification instructions
- Quality check instructions

### Implementation steps
1. In `prompt_generation.py`, split `get_system_prompt` to branch on `add_as`:
   - `add_as == "new_page"` → return `_build_add_page_prompt(...)` (~9k chars)
   - else → return existing full prompt
2. `_build_add_page_prompt` includes: reuse, CLAUDE.md read, routing, verification
3. Similarly, trim `build_conversion_prompt` for add_page: skip design token preamble
   since CLAUDE.md already has it.

---

## Estimated Impact

| Change | Token savings per add-page run | Time savings |
|--------|-------------------------------|--------------|
| Per-project CLAUDE.md | ~4,000 tokens (10 fewer tool calls) | ~12s |
| Lean add-page prompt | ~12,000 tokens (system prompt reduction) | ~0s (just cost) |
| **Total** | **~16,000 tokens per add-page** | **~12s faster** |

main.py split has no runtime impact — pure developer experience improvement.

---

## Execution Order

1. **Part 1** (CLAUDE.md) — highest ROI, implement first
2. **Part 3** (lean prompt) — implement alongside Part 1, shares same context
3. **Part 2** (main.py split) — do last, purely structural, no user-visible change
