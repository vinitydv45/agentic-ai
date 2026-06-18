# V2 Architecture: Claude Agent SDK, MCP Servers & Full Pipeline

> **Project:** Aura2 — AI-Powered Figma → React Converter
> **Stack:** FastAPI · Claude Agent SDK (claude-sonnet-4-6) · React 18 · TypeScript · ChromaDB · SQLite
> **Samsung PRISM @ IIIT Naya Raipur**
> This document covers the full V2 system architecture: agent SDK integration, MCP server topology, conversation flow, data flow, and key configuration.

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Two Input Paths](#2-the-two-input-paths)
3. [Claude Agent SDK Integration](#3-claude-agent-sdk-integration)
4. [MCP Server Topology](#4-mcp-server-topology)
5. [Agent Conversation Flow](#5-agent-conversation-flow)
6. [Data Flow Diagram](#6-data-flow-diagram)
7. [Key Configuration](#7-key-configuration)

---

## 1. Overview

Aura2 converts Figma designs into production-ready React + Tailwind CSS projects. The core engine is the `FigmaToReactAgent` class (`backend/agents/figma_to_react.py`), which:

1. Accepts a Figma design (via REST API URL *or* Figma Plugin payload).
2. Extracts comprehensive design data — pages, frames, colors, fonts, images, layout, effects.
3. Persists the raw Figma JSON and processed design data to `{project}/figma_data/`.
4. Sends the design data to a Claude agent session as a structured prompt.
5. The agent generates React + Tailwind components, using MCP tools for file I/O, browser testing, component library search, GitHub, and Vercel.
6. After generation, the build is verified and fixed in a loop (up to 2 fix attempts).
7. Visual verification runs a multi-tier confidence scoring pipeline.
8. Code quality checks (ESLint, Prettier) run automatically.

**Supported UI libraries:** Tailwind CSS, Material UI (MUI), Chakra UI, CSS Modules.

---

## 2. The Two Input Paths

### Path A: Figma REST API

```
Figma URL
  → extract_figma_file_key(url)
  → fetch_figma_data(file_key, token)      # Figma REST API
  → extract_complete_design_data(figma_data)
  → download_figma_images(file_key, ...)
  → save_figma_json(project_path, raw_data, design_data, source="api")
```

The REST API path requires a `figma_token` and extracts the file key from URLs like `https://www.figma.com/file/{file_key}/...`.

### Path B: Figma Plugin (Preferred)

```
Plugin Data (JSON payload from Figma Plugin UI)
  → convert_plugin_data_to_design_data(plugin_data)
  → save_plugin_images(plugin_images, images_dir)
  → save_figma_json(project_path, raw_data, design_data, source="plugin")
```

The plugin path **bypasses the Figma REST API entirely**. The Figma Plugin extracts design data client-side and sends it directly to the backend, avoiding rate limits and API quotas. It can also capture a **design screenshot** for visual verification.

### Both Paths Converge

Both paths produce the same normalized `design_data` dict and call the shared `_run_agent_conversion()` method, which handles the Claude agent session, build verification, and visual verification.

---

## 3. Claude Agent SDK Integration

### ClaudeSDKClient and ClaudeAgentOptions

The conversion uses the Claude Agent SDK (`claude_agent_sdk` package) with two key classes:

- **`ClaudeAgentOptions`** — configures the agent session.
- **`ClaudeSDKClient`** — manages the connection to the Claude agent.

```python
# From backend/agents/figma_to_react.py

options = ClaudeAgentOptions(
    model=settings.default_model,           # "claude-sonnet-4-6"
    system_prompt=system_prompt,
    max_turns=settings.max_agent_turns,     # 500
    cwd=str(project_path),                  # Agent works inside the project dir
    max_buffer_size=20 * 1024 * 1024,       # 20MB for large base64 screenshots
    allowed_tools=[...],                    # Explicit tool allowlist
    mcp_servers=mcp_servers,                # MCP server configuration
    permission_mode="acceptEdits",          # Auto-accept file edits
)

async with ClaudeSDKClient(options=options) as client:
    await client.query(conversion_prompt)
    async for message in client.receive_response():
        # Process AssistantMessage, ToolUseBlock, ResultMessage
```

### Model Configuration

| Setting | Value | Purpose |
|---|---|---|
| `default_model` | `claude-sonnet-4-6` | Main conversion and fix agent |
| `fast_model` | `claude-haiku-4-5-20251001` | Lighter tasks (not used in main pipeline) |
| `vision_comparison_model` | `claude-sonnet-4-6` | Vision API for screenshot comparison |
| `max_agent_turns` | 500 | Maximum conversation turns for conversion |
| `max_fix_turns` | 15 | Maximum turns for the fix agent |

### Message Processing

The agent streams responses as typed messages:

- **`AssistantMessage`** — contains `TextBlock` (reasoning) and `ToolUseBlock` (tool calls).
- **`ResultMessage`** — final result or error.
- **`ToolUseBlock`** — each tool call is logged with contextual details (file paths, URLs, component names).

The agent has access to standard code tools (`Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep`, `TodoWrite`, `KillShell`, `BashOutput`, `TaskOutput`) plus MCP tools from the configured servers.

---

## 4. MCP Server Topology

All MCP servers run as **stdio** subprocesses launched via `npx`. The agent communicates with them through the Claude Agent SDK's MCP integration.

### 4.1 Playwright (Visual Testing)

```python
"playwright": {
    "type": "stdio",
    "command": npx_command,
    "args": ["@playwright/mcp@latest"],
}
```

**Purpose:** Browser automation for visual testing during conversion. The agent uses Playwright to:
- Navigate to the dev server (`browser_navigate`)
- Take screenshots (`browser_take_screenshot`)
- Inspect the DOM (`browser_snapshot`, `browser_evaluate`)
- Test interactions (`browser_click`, `browser_hover`)
- Check console errors (`browser_console_messages`)
- Test responsive layouts (`browser_resize`)

**Allowed tools:**
`browser_navigate`, `browser_take_screenshot`, `browser_snapshot`, `browser_click`, `browser_hover`, `browser_scroll`, `browser_resize`, `browser_evaluate`, `browser_wait_for`, `browser_console_messages`, `browser_network_requests`

### 4.2 Component Library (RAG Search/Save)

```python
"component_library": create_component_library_server()
```

**Purpose:** ChromaDB-backed semantic search for component reuse. Components generated in previous projects are saved to the library and can be retrieved for reuse in new projects.

**Allowed tools:**
- `search_components` — semantic search by description/category
- `save_component` — store a new component with metadata
- `get_component` — retrieve a specific component by ID

The library uses ChromaDB with sentence-transformer embeddings, persisted to `./component_library/chroma/`.

### 4.3 GitHub (Repo/PR Creation) — Optional

```python
"github": {
    "type": "stdio",
    "command": npx_command,
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": token},
}
```

**Enabled when:** `settings.effective_github_token` is set AND `settings.auto_create_repo` is `True`.

**Allowed tools:**
`create_repository`, `create_branch`, `push_files`, `create_pull_request`, `create_or_update_file`, `get_file_contents`, `list_branches`

### 4.4 Vercel (Deployment) — Optional

```python
"vercel": {
    "type": "stdio",
    "command": npx_command,
    "args": ["-y", "@vercel/mcp"],
    "env": {"VERCEL_TOKEN": token, ...},
}
```

**Enabled when:** `settings.vercel_token` is set AND `settings.auto_deploy_vercel` is `True`.

**Allowed tools:**
`deploy`, `list_projects`, `get_project`, `create_project`, `list_deployments`, `get_deployment`

### MCP Server Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Agent SDK Session                       │
│                                                                   │
│   System Prompt + Conversion Prompt                               │
│        │                                                          │
│        ▼                                                          │
│   ┌─────────────────────┐                                        │
│   │   Claude Model       │                                        │
│   │   (claude-sonnet-4-6)│                                        │
│   └──────┬──────────────┘                                        │
│          │ Tool Calls                                             │
│          ▼                                                        │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │  Built-in Tools: Read, Write, Edit, Bash, Glob, Grep     │   │
│   └──────────────────────────────────────────────────────────┘   │
│          │                                                        │
│          ▼                                                        │
│   ┌───────────────────────────────────────────────────────────┐  │
│   │  MCP Servers (stdio subprocesses)                          │  │
│   │                                                            │  │
│   │  ┌──────────────┐  ┌───────────────────┐                  │  │
│   │  │  Playwright   │  │ Component Library │                  │  │
│   │  │  (always on)  │  │  (always on)      │                  │  │
│   │  │  Screenshots  │  │  ChromaDB RAG     │                  │  │
│   │  │  DOM inspect  │  │  Search/Save      │                  │  │
│   │  └──────────────┘  └───────────────────┘                  │  │
│   │                                                            │  │
│   │  ┌──────────────┐  ┌───────────────────┐                  │  │
│   │  │  GitHub       │  │  Vercel           │                  │  │
│   │  │  (optional)   │  │  (optional)       │                  │  │
│   │  │  Repo/PR ops  │  │  Deploy           │                  │  │
│   │  └──────────────┘  └───────────────────┘                  │  │
│   └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Agent Conversation Flow

The agent session follows this sequence:

### Phase 1: System Prompt

Generated by `get_system_prompt()` in `backend/agents/_figma_to_react/prompt_generation.py`. It configures the agent as a React + Tailwind expert and includes:
- UI library instructions (Tailwind, MUI, Chakra, CSS Modules)
- Project mode (`new_project` vs `new_page`)
- GitHub/Vercel availability flags
- Semantic HTML requirements (uses inferred semantic types like `navigation`, `hero`, `card`)
- Accessibility requirements (ARIA roles from `get_aria_role()`)

### Phase 2: Conversion Prompt

Built by `build_conversion_prompt()`. Contains:
- Project path and name
- Complete design data formatted as text (colors, fonts, image paths, frame hierarchy)
- Design screenshot path (if available from plugin)
- Instructions to use `data-figma-id` attributes on elements for structural verification

### Phase 3: Tool Use Loop

The agent iterates autonomously:
1. Reads the design data from the prompt.
2. Searches the component library for reusable components (`search_components`).
3. Generates React components (`Write` tool to `src/components/*.tsx`).
4. Saves new components to the library (`save_component`).
5. Creates pages, routes, and app layout.
6. Uses Playwright to preview and verify visually.
7. Continues until satisfied or `max_agent_turns` is reached.

### Phase 4: Build Verification (Build-Fix Loop)

After the agent finishes, the build is verified:
```
for fix_attempt in range(max_fix_attempts + 1):   # max_fix_attempts = 2
    build_result = await verify_build(project_path)
    if build_ok:
        break
    # Feed errors back to agent → agent fixes → re-verify
```

The agent receives formatted build errors (TypeScript issues, missing imports, etc.) and fixes them. Up to 2 fix rounds.

### Phase 5: Visual Verification Loop

Runs after build succeeds (see `docs/05_VALIDATION_PIPELINE.md` for full details):
- Starts a dev server
- Captures screenshots via Playwright
- Runs vision, structural, and content comparisons
- Generates and applies fixes iteratively

---

## 6. Data Flow Diagram

```
┌──────────────────────┐     ┌──────────────────────┐
│   Figma Plugin       │     │   Figma REST API     │
│   (direct extraction)│     │   (URL + token)      │
└─────────┬────────────┘     └─────────┬────────────┘
          │                            │
          ▼                            ▼
   convert_plugin_data_        fetch_figma_data()
   to_design_data()            extract_complete_design_data()
          │                            │
          └──────────┬─────────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │   design_data {}   │  Normalized design data
            │   - pages[]        │  (pages, frames, colors,
            │   - colors[]       │   fonts, images, layout)
            │   - fonts[]        │
            │   - imageRefs{}    │
            │   - stats{}        │
            └────────┬───────────┘
                     │
        ┌────────────┼────────────────────┐
        │            │                    │
        ▼            ▼                    ▼
  save_figma_json()  download_images()    build_conversion_prompt()
  {project}/         {project}/public/    │
  figma_data/        images/              │
  ├─ raw_figma_                           ▼
  │  response.json            ┌────────────────────────┐
  ├─ design_data.json         │   Claude Agent Session  │
  └─ design_metadata.json     │   (claude-sonnet-4-6)   │
                              └────────────┬───────────┘
                                           │
                              ┌────────────┼────────────┐
                              │            │            │
                              ▼            ▼            ▼
                        Write files   Playwright    Component
                        src/          screenshots   Library
                        components/   & DOM check   search/save
                        pages/
                              │
                              ▼
                     ┌─────────────────┐
                     │  Build Verify   │ ← up to 2 fix rounds
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Visual Verify  │ ← up to 10 iterations
                     │  (Vision +      │   confidence threshold: 95%
                     │   Structural +  │   early stop: 98%
                     │   Content)      │
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Code Quality   │  ESLint + Prettier
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Result JSON    │  status, path, stats
                     │  + Deploy       │  (GitHub, Vercel if enabled)
                     └─────────────────┘
```

---

## 7. Key Configuration

All settings live in `backend/config.py` as a Pydantic `Settings` class, loaded from environment variables and `.env`.

### API Keys & LiteLLM

| Setting | Default | Description |
|---|---|---|
| `figma_token` | `""` | Figma personal access token |
| `litellm_api_key` | `""` | API key for LLM provider |
| `litellm_base_url` | `""` | Custom LLM endpoint (LiteLLM proxy) |
| `litellm_provider` | `""` | LLM provider identifier |

### Agent Settings

| Setting | Default | Description |
|---|---|---|
| `default_model` | `"claude-sonnet-4-6"` | Primary model for conversion |
| `fast_model` | `"claude-haiku-4-5-20251001"` | Lightweight model for quick tasks |
| `max_agent_turns` | `500` | Max conversation turns |
| `max_fix_turns` | `15` | Max turns for fix agent |

### Directories

| Setting | Default | Description |
|---|---|---|
| `generated_projects_dir` | `./generated_projects` | Where generated projects are saved |
| `component_library_dir` | `./component_library` | ChromaDB persistence directory |

### Code Quality

| Setting | Default | Description |
|---|---|---|
| `auto_run_lint` | `True` | Run ESLint after generation |
| `auto_run_format` | `True` | Run Prettier after generation |
| `auto_fix_lint` | `True` | Auto-fix ESLint issues |
| `verify_build` | `True` | Verify build after generation |

### Visual Verification

| Setting | Default | Description |
|---|---|---|
| `enable_vision_comparison` | `True` | Use Claude Vision API |
| `vision_comparison_model` | `"claude-sonnet-4-6"` | Model for vision comparison |
| `max_verification_iterations` | `10` | Max verification loop iterations |
| `verification_confidence_threshold` | `0.95` | Minimum confidence to pass (95%) |
| `verification_early_stop_threshold` | `0.98` | Confidence for early stop (98%) |
| `screenshot_scale` | `1` | Screenshot scale (1x to reduce token cost) |
| `screenshot_viewport_width` | `1440` | Desktop viewport width |
| `screenshot_viewport_height` | `900` | Desktop viewport height |

### Structural Comparison

| Setting | Default | Description |
|---|---|---|
| `enable_structural_comparison` | `True` | Compare Figma JSON vs DOM styles |
| `structural_comparison_tolerance_px` | `2` | Pixel tolerance for dimensions |
| `color_comparison_tolerance` | `5` | Per-channel RGB tolerance (0-255) |

### Fix Application

| Setting | Default | Description |
|---|---|---|
| `max_fixes_per_iteration` | `5` | Max fixes applied per verification iteration |
| `auto_apply_high_priority_fixes` | `True` | Auto-apply high severity fixes |
| `require_manual_review_for_low_confidence` | `True` | Require manual review below threshold |

### GitHub & Vercel (Optional)

| Setting | Default | Description |
|---|---|---|
| `github_token` | `""` | GitHub personal access token |
| `github_owner` | `""` | GitHub org or username |
| `auto_create_repo` | `False` | Auto-create repo on conversion |
| `auto_create_pr` | `False` | Auto-create PR after generation |
| `vercel_token` | `""` | Vercel API token |
| `auto_deploy_vercel` | `False` | Auto-deploy to Vercel |

---

## Key Source Files

| File | Purpose |
|---|---|
| `backend/agents/figma_to_react.py` | Main `FigmaToReactAgent` class and pipeline orchestration |
| `backend/agents/_figma_to_react/prompt_generation.py` | System prompt and conversion prompt generation |
| `backend/agents/_figma_to_react/verification.py` | Visual verification loop and fix application |
| `backend/agents/_figma_to_react/figma_extraction.py` | Figma data extraction and normalization |
| `backend/agents/_figma_to_react/figma_json_persistence.py` | Raw JSON and design data persistence |
| `backend/agents/_figma_to_react/plugin_conversion.py` | Plugin data format conversion |
| `backend/agents/_figma_to_react/project_setup.py` | Template-based project scaffolding |
| `backend/config.py` | All configuration settings |
| `backend/rag/component_store.py` | ChromaDB component store |
| `backend/mcp_tools/component_library.py` | Component library MCP server |
| `backend/utils/vision_comparison.py` | Claude Vision API comparison |
| `backend/utils/structural_comparison.py` | Figma JSON vs DOM computed styles |
| `backend/utils/visual_comparison.py` | Content-based comparison and screenshot capture |
| `backend/utils/build_verifier.py` | Build verification |
| `backend/utils/auto_fix_agent.py` | Fix generation from discrepancies |
| `backend/main.py` | FastAPI application and API endpoints |
