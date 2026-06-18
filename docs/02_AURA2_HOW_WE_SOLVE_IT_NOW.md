# Aura2: How We Solve It Now — Complete Architecture & Pipeline

> **Project:** Aura2 — AI-Powered Figma → React Converter
> **Stack:** FastAPI · Claude Agent SDK (claude-opus-4-6) · React 18 · TypeScript · ChromaDB · SQLite
> **Samsung PRISM @ IIIT Naya Raipur**
> This document covers the complete current architecture: every subsystem, technique, and design decision.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [The Two Input Paths](#2-the-two-input-paths)
3. [Figma Plugin API — Bypassing Rate Limits](#3-figma-plugin-api--bypassing-rate-limits)
4. [Data Extraction Pipeline](#4-data-extraction-pipeline)
5. [Claude Agent SDK — The Conversion Engine](#5-claude-agent-sdk--the-conversion-engine)
6. [Prompt Engineering for Design Fidelity](#6-prompt-engineering-for-design-fidelity)
7. [RAG Component Reuse System](#7-rag-component-reuse-system)
8. [Visual Verification Loop](#8-visual-verification-loop)
9. [Code Quality Pipeline](#9-code-quality-pipeline)
10. [Multi-Page Support](#10-multi-page-support)
11. [CI/CD and Deployment](#11-cicd-and-deployment)
12. [Backend API Architecture](#12-backend-api-architecture)
13. [Frontend Dashboard](#13-frontend-dashboard)
14. [Complete Conversion Flow](#14-complete-conversion-flow)
15. [Performance Metrics](#15-performance-metrics)

---

## 1. System Overview

Aura2 converts Figma designs into production-ready React + Tailwind projects through a multi-stage intelligent pipeline:

```
┌─────────────────────────────────────────────────────────────┐
│                      INPUT SOURCES                           │
│                                                             │
│  [Figma Plugin] ──────────────────────────────────────────► │
│  (Bypasses rate limits, direct design data)                 │
│                                                             │
│  [Figma REST API] ─────────────────────────────────────────►│
│  (Fallback, with exponential backoff)                       │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│              BACKEND: FastAPI (port 8000)                    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              FigmaToReactAgent                       │   │
│  │  (Claude Agent SDK — claude-opus-4-6)                │   │
│  │                                                     │   │
│  │  1. Data Extraction → 2. RAG Lookup → 3. LLM Gen   │   │
│  │  4. Code Quality → 5. Visual Verification           │   │
│  └────────────────────┬────────────────────────────────┘   │
│                        │                                    │
│  ┌─────────────────────▼──────────────────────────────┐   │
│  │  ChromaDB (RAG)    │  SQLite DB   │  JSON Storage  │   │
│  │  Component Library │  Project DB  │  projects.json │   │
│  └────────────────────────────────────────────────────┘   │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│              GENERATED OUTPUT                                │
│                                                             │
│  generated_projects/{name}/                                 │
│  ├── src/                                                   │
│  │   ├── components/  (React + TypeScript)                  │
│  │   ├── pages/       (Route pages)                         │
│  │   └── styles/      (Tailwind, CSS)                       │
│  ├── public/images/   (Figma-exported assets)               │
│  ├── package.json     (Vite + React + Tailwind)             │
│  └── .github/workflows/ (CI/CD)                             │
└─────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┴────────────┐
                    ▼                          ▼
              [GitHub Repo]             [Vercel Deploy]
```

---

## 2. The Two Input Paths

### Path A: Figma Plugin (Primary, Recommended)

The Figma Plugin runs **inside** the Figma application and has direct access to the design tree through the Plugin API — no REST API calls needed.

```
User opens Figma → Runs Aura Export plugin → Plugin extracts data
    → Plugin sends JSON to backend via HTTP POST
    → Backend processes without ever calling Figma REST API
```

**Key advantage:** Zero rate limit exposure.

### Path B: Figma REST API (Fallback)

```
User provides Figma URL → Backend extracts file key
    → GET /v1/files/{fileKey} (with rate limit handling)
    → GET /v1/images/{fileKey} (for image export, per-image)
    → Exponential backoff on 429
```

### Selection Logic (in `backend/main.py`)

```python
if request.plugin_data:
    # Path A: Plugin data
    design_data = convert_plugin_data_to_design_data(request.plugin_data)
    images = save_plugin_images(request.plugin_data)
    data_source = "plugin"
else:
    # Path B: REST API
    figma_data = await fetch_figma_data(file_key, figma_token)
    design_data = extract_complete_design_data(figma_data)
    images = await download_figma_images(file_key, figma_token, design_data)
    data_source = "rest_api"
```

---

## 3. Figma Plugin API — Bypassing Rate Limits

### 3.1 How the Figma Plugin Works

The plugin (`figma-plugin/src/code.ts`) runs in Figma's sandboxed TypeScript runtime:

```typescript
// Plugin entry point — runs inside Figma
figma.showUI(__html__, { width: 400, height: 650 });

// Direct access to the design tree — NO REST API calls
const designData: DesignData = {
  fileName: figma.root.name,
  pages: [],
  colors: {},
  fonts: [],
  images: {},
  stats: { pageCount: 0, frameCount: 0, colorCount: 0, fontCount: 0, imageCount: 0 }
};

// Walk the entire document tree
for (const page of figma.root.children) {
  const pageData: PageData = {
    id: page.id,
    name: page.name,
    frames: []
  };
  // Extract frames (top-level children of pages)
  for (const node of page.children) {
    pageData.frames.push(extractNode(node));
  }
  designData.pages.push(pageData);
}
```

### 3.2 Image Export from Plugin

The plugin exports images using `figma.exportAsync()` — a Plugin API method unavailable via REST:

```typescript
async function exportImages(node: SceneNode, designData: DesignData) {
  if ('exportAsync' in node) {
    const imageData = await node.exportAsync({
      format: 'PNG',
      constraint: { type: 'SCALE', value: 2 }  // 2x for retina
    });
    // Convert Uint8Array to base64
    const base64 = uint8ArrayToBase64(imageData);
    designData.images[node.id] = `data:image/png;base64,${base64}`;
  }
}
```

### 3.3 Plugin to Backend Communication

The plugin UI (running in iframe context) sends data to the backend:

```typescript
// In plugin UI (browser context, can make HTTP calls)
async function sendToBackend(designData: DesignData, settings: PluginSettings) {
  const payload = {
    project_name: settings.projectName,
    plugin_data: {
      design_data: designData,
      timestamp: Date.now(),
      plugin_version: '1.0.0'
    }
  };

  const response = await fetch(`${settings.backendUrl}/api/figma/plugin-upload`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  });
}
```

### 3.4 Plugin Data Structure

```typescript
interface DesignData {
  fileName: string;
  pages: PageData[];           // All pages in the file
  colors: Record<string, ColorInfo>;  // Color palette
  fonts: FontInfo[];           // All fonts used
  images: Record<string, string>;     // nodeId → base64 PNG
  stats: {
    pageCount: number;
    frameCount: number;
    colorCount: number;
    fontCount: number;
    imageCount: number;
  };
}

interface ExtractedNode {
  id: string;
  name: string;
  type: string;               // FRAME, TEXT, RECTANGLE, etc.
  visible: boolean;
  opacity?: number;
  layout?: LayoutInfo;        // Auto-layout / flex info
  fills?: Paint[];            // Colors, gradients, images
  strokes?: Paint[];          // Border colors
  effects?: Effect[];         // Shadows, blur
  cornerRadius?: CornerRadius;
  text?: string;              // For TEXT nodes
  style?: TextStyle;          // For TEXT nodes
  children?: ExtractedNode[]; // Recursive tree
  absoluteBoundingBox?: { x, y, width, height };
}
```

---

## 4. Data Extraction Pipeline

### 4.1 Node Data Extraction (`backend/agents/_figma_to_react/figma_extraction.py`)

The extraction pipeline recursively walks the Figma node tree, extracting everything needed for code generation:

```python
def extract_node_data(node: dict, depth: int = 0) -> dict:
    """Extract complete data from a single Figma node."""
    node_type = node.get("type", "")

    data = {
        "id": node.get("id"),
        "name": node.get("name", ""),
        "type": node_type,
        "visible": node.get("visible", True),
    }

    if node_type == "TEXT":
        data["text"] = node.get("characters", "")
        data["style"] = extract_text_style(node)   # font, size, weight, lineHeight...
        data["fills"] = extract_fills(node)

    elif node_type in ("FRAME", "GROUP", "COMPONENT", "INSTANCE", "SECTION"):
        data["layout"] = extract_layout_info(node)  # auto-layout → flexbox mapping
        data["fills"] = extract_fills(node)
        data["strokes"] = extract_strokes(node)
        data["effects"] = extract_effects(node)      # shadows, blur
        data["cornerRadius"] = extract_corner_radius(node)
        data["clipsContent"] = node.get("clipsContent", False)

        # Recursively process children (only visible)
        if node.get("children"):
            data["children"] = [
                extract_node_data(child, depth + 1)
                for child in node["children"]
                if child.get("visible", True)
            ]
    # ... RECTANGLE, VECTOR, ELLIPSE, LINE also handled
```

### 4.2 Color Conversion (`backend/agents/_figma_to_react/design_styles.py`)

Figma stores colors as RGBA with values 0-1. We convert to web-standard hex:

```python
def rgba_to_hex(color: dict, opacity: float = 1.0) -> str:
    """Convert Figma RGBA color to hex string."""
    r = int(color.get("r", 0) * 255)   # 0-1 → 0-255
    g = int(color.get("g", 0) * 255)
    b = int(color.get("b", 0) * 255)
    a = color.get("a", 1.0) * opacity
    if a < 1.0:
        return f"rgba({r}, {g}, {b}, {a:.2f})"  # Transparent colors
    return f"#{r:02x}{g:02x}{b:02x}"             # Full hex
```

Example: Figma's `{r: 0.078, g: 0.157, b: 0.627, a: 1.0}` → `#1428A0` (Samsung Blue)

### 4.3 Auto-Layout → Flexbox Mapping

```python
def extract_layout_info(node: dict) -> dict:
    layout = {}
    if node.get("layoutMode"):
        layout["mode"] = node["layoutMode"]  # HORIZONTAL → flex-direction: row
                                              # VERTICAL → flex-direction: column
        layout["itemSpacing"] = node.get("itemSpacing", 0)  # → gap
        layout["padding"] = {
            "top": node.get("paddingTop", 0),      # → pt-X
            "right": node.get("paddingRight", 0),  # → pr-X
            "bottom": node.get("paddingBottom", 0),
            "left": node.get("paddingLeft", 0),    # → pl-X
        }
        # Alignment mapping:
        # primaryAxisAlignItems: MIN → justify-start, CENTER → justify-center, MAX → justify-end, SPACE_BETWEEN → justify-between
        # counterAxisAlignItems: MIN → items-start, CENTER → items-center, MAX → items-end
```

### 4.4 Text Style Extraction

```python
def extract_text_style(node: dict) -> dict:
    style = node.get("style", {})
    return {
        "fontFamily": style.get("fontFamily", "Inter"),
        "fontSize": style.get("fontSize", 16),
        "fontWeight": style.get("fontWeight", 400),
        "lineHeight": style.get("lineHeightPx"),           # Pixel line height
        "letterSpacing": style.get("letterSpacing", 0),    # In em units
        "textAlignHorizontal": style.get("textAlignHorizontal", "LEFT"),
        "textCase": style.get("textCase", "ORIGINAL"),     # UPPER, LOWER, etc.
        "textDecoration": style.get("textDecoration", "NONE"),
    }
```

### 4.5 Effects Extraction (Shadows, Blur)

```python
def extract_effects(node: dict) -> list:
    effects = []
    for effect in node.get("effects", []):
        if not effect.get("visible", True):
            continue
        if effect["type"] == "DROP_SHADOW":
            effects.append({
                "type": "DROP_SHADOW",
                "color": rgba_to_hex(effect.get("color", {})),
                "offset": effect.get("offset", {"x": 0, "y": 0}),
                "radius": effect.get("radius", 0),
                "spread": effect.get("spread", 0),
            })
            # → CSS: box-shadow: Xpx Ypx radiuspx spreadpx color
```

### 4.6 Semantic Type Inference (`backend/agents/_figma_to_react/semantic_analysis.py`)

One of Aura2's key innovations — automatically detecting what a component IS:

```python
def infer_semantic_type(node: dict) -> str:
    """Infer semantic type from node name, type, and structure."""
    name = node.get("name", "").lower()
    node_type = node.get("type", "")

    # Name-based detection
    if any(k in name for k in ["header", "nav", "navbar", "navigation"]):
        return "header"
    if any(k in name for k in ["footer"]):
        return "footer"
    if any(k in name for k in ["hero", "banner", "jumbotron"]):
        return "hero"
    if any(k in name for k in ["btn", "button", "cta"]):
        return "button"
    if any(k in name for k in ["card", "tile", "item"]):
        return "card"
    if any(k in name for k in ["sidebar", "side-panel"]):
        return "sidebar"
    if any(k in name for k in ["input", "field", "form"]):
        return "input"
    if any(k in name for k in ["modal", "dialog", "popup"]):
        return "modal"
    # ... 15+ more patterns

def get_aria_role(semantic_type: str) -> str:
    """Map semantic type to ARIA role."""
    role_map = {
        "header": "banner",
        "footer": "contentinfo",
        "nav": "navigation",
        "main": "main",
        "button": "button",
        "input": "textbox",
        "modal": "dialog",
        "card": "article",
    }
    return role_map.get(semantic_type, "")
```

---

## 5. Claude Agent SDK — The Conversion Engine

### 5.1 What is Claude Agent SDK?

The **Claude Agent SDK** (PyPI: `claude-agent-sdk`, also `claude_agent_sdk`) is the underlying framework that powers **Claude Code**, made available as a programmable library. Anthropic renamed it from "Claude Code SDK" in 2025 to reflect broader applicability beyond coding.

It gives developers the same tools, agent loop, and context management that power Claude Code:

```python
# Minimal usage
import claude_agent_sdk

async for message in claude_agent_sdk.query(
    prompt="Convert this Figma design to React...",
    options={"allowed_tools": ["Read", "Write", "Bash"]}
):
    print(message)
```

The `query()` function creates and drives the **agentic loop**, returning an async iterator that streams messages as Claude works.

**Key differences from raw Claude API:**

| Dimension | Claude Agent SDK | Raw Anthropic API |
|---|---|---|
| Tool execution | Claude executes 14+ built-in tools directly (Read, Write, Bash, Glob, Grep...) | You define tools as JSON schema + implement execution yourself |
| State management | Session persistence, automatic context compaction as conversation grows | Stateless — pass full message history every call |
| Context compaction | Automatic — SDK handles compaction at limits, re-reads CLAUDE.md after | Manual — you must truncate/summarize history |
| Orchestration | SDK handles tool execution, retries, context management | Manual loop implementation required |
| Subagents | First-class support — spawn subagents for subtasks | Not native |
| MCP integration | Built-in — MCP servers plug in via `mcp_servers` dict | Requires manual integration |

The five core message types emitted by the SDK loop:
- `SystemMessage` — session lifecycle events
- `AssistantMessage` — Claude's response (includes TextBlock + ToolUseBlock)
- `UserMessage` — tool execution results sent back to Claude
- `ResultMessage` — final result
- `ToolResultMessage` — individual tool output

### 5.2 Agent Configuration

```python
from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient

options = ClaudeAgentOptions(
    model="claude-opus-4-6",          # Most capable model
    system_prompt=get_system_prompt(), # Expert React developer persona
    max_turns=50,                      # Max back-and-forth iterations
    cwd=str(project_path),             # Working directory = project folder
    max_buffer_size=20 * 1024 * 1024, # 20MB — handles large base64 screenshots
    allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
    mcp_servers={
        "playwright": {
            "type": "stdio",
            "command": "npx",
            "args": ["@playwright/mcp@latest"],
        },
        "component_library": create_component_library_server(),
    },
    permission_mode="acceptEdits",     # Auto-accept file writes
)

async with ClaudeSDKClient(options=options) as client:
    await client.query(conversion_prompt)
    async for message in client.receive_response():
        # Process generated code
```

### 5.3 MCP Servers Available to the Agent

Up to 4 MCP servers connect to the agent, depending on configuration:

```python
mcp_servers = {
    # Always enabled
    "playwright": {
        "type": "stdio",
        "command": "npx",
        "args": ["@playwright/mcp@latest"],
    },
    "component_library": create_component_library_server(),  # ChromaDB RAG

    # Conditional: only if GITHUB_PERSONAL_ACCESS_TOKEN set + AUTO_CREATE_REPO=true
    "github": {
        "type": "stdio",
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {"GITHUB_PERSONAL_ACCESS_TOKEN": settings.effective_github_token},
    },

    # Conditional: only if VERCEL_TOKEN set
    "vercel": {
        "type": "stdio",
        "command": "npx",
        "args": ["-y", "@vercel/mcp"],
        "env": {
            "VERCEL_TOKEN": settings.vercel_token,
            "VERCEL_ORG_ID": settings.vercel_org_id,  # optional
        },
    },
}
```

**Full tool list available to the agent:**

| MCP Server | Tools |
|---|---|
| `playwright` | `browser_navigate`, `browser_take_screenshot`, `browser_snapshot`, `browser_click`, `browser_hover`, `browser_scroll`, `browser_resize`, `browser_evaluate`, `browser_wait_for`, `browser_console_messages`, `browser_network_requests` |
| `component_library` | `search_components`, `save_component`, `get_component` |
| `github` | `create_repository`, `create_branch`, `push_files`, `create_pull_request`, `create_or_update_file`, `get_file_contents`, `list_branches` |
| `vercel` | `deploy`, `list_projects`, `get_project`, `create_project`, `list_deployments`, `get_deployment` |

### 5.4 Agentic Tool Use Loop

When the agent runs, it autonomously:

```
Agent receives design data prompt
    ↓
Agent thinks: "I need to check for similar components"
    ↓ tool_use: search_components(query="hero section")
    ↓ tool_result: [{ name: "HeroSection", similarity: 0.87 }]
    ↓
Agent thinks: "Found similar — I'll adapt it"
    ↓ tool_use: get_component(id="hero_HeroSection_5")
    ↓ tool_result: { code: "export default function HeroSection..." }
    ↓
Agent generates adapted code
    ↓ tool_use: Write("src/components/HeroSection.tsx", code)
    ↓ tool_use: Write("src/pages/Home.tsx", page_code)
    ↓
Agent thinks: "Let me verify the build works"
    ↓ tool_use: Bash("npm run build")
    ↓ tool_result: "Build successful"
    ↓
Agent: "Done. Saved component to library."
    ↓ tool_use: save_component(name="HeroSection", code=..., category="hero")
```

---

## 6. The Template System — Zero Setup, Instant Start

One of Aura2's critical design decisions: **never generate boilerplate from scratch**. Instead, every project starts from a pre-built, tested template that is simply copied.

### 6.1 Three Templates, Three UI Libraries

```
templates/
├── react-tailwind/    ← Default (most used)
├── react-mui/         ← Material UI v7
└── react-chakra/      ← Chakra UI v3
```

### 6.2 Template Selection

```python
# project_setup.py
def setup_project_from_template(project_name, output_dir, ui_library="tailwind"):
    template_map = {
        "tailwind": "react-tailwind",
        "mui":      "react-mui",
        "chakra":   "react-chakra",
    }
    template_name = template_map.get(ui_library, "react-tailwind")
    template_dir = Path("templates") / template_name

    project_path = output_dir / project_name

    # Fresh copy — remove any existing project
    if project_path.exists():
        shutil.rmtree(project_path)

    # Copy entire template tree
    shutil.copytree(template_dir, project_path, ignore=shutil.ignore_patterns('.gitkeep'))

    # Replace {{PROJECT_NAME}} placeholder in package.json + index.html
    for file_name in ["package.json", "index.html"]:
        file_path = project_path / file_name
        if file_path.exists():
            content = file_path.read_text()
            content = content.replace("{{PROJECT_NAME}}", project_name)
            file_path.write_text(content)

    return project_path
```

### 6.3 What Each Template Contains (Pre-Configured)

#### react-tailwind (Default)

```json
// package.json — pre-configured with all tools
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^7.1.3"
  },
  "devDependencies": {
    "tailwindcss": "^4.0.0",
    "@tailwindcss/postcss": "^4.0.0",
    "eslint": "^9.0.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "prettier": "^3.2.0",
    "typescript": "~5.8.0",
    "vite": "^7.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",    // TypeScript compile + Vite bundle
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,css}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,css}\"",
    "typecheck": "tsc --noEmit"          // Type check without building
  }
}
```

Pre-baked file structure:
```
react-tailwind/
├── .github/
│   └── workflows/
│       ├── ci.yml      ← CI pipeline (build, lint, typecheck)
│       └── deploy.yml  ← Deploy to GitHub Pages / Vercel
├── src/
│   ├── App.tsx         ← Minimal router setup
│   ├── main.tsx        ← React 19 entry point
│   ├── index.css       ← Tailwind v4 @import
│   ├── components/     ← Empty (agent fills this)
│   ├── pages/          ← Empty (agent fills this)
│   └── types/index.ts  ← Shared TypeScript interfaces
├── package.json        ← All deps pre-specified
├── vite.config.ts
├── tsconfig.json
├── postcss.config.js
└── .prettierrc
```

#### react-mui

```json
{
  "dependencies": {
    "@mui/material": "^7.0.0",
    "@mui/icons-material": "^7.0.0",
    "@emotion/react": "^11.14.0",
    "@emotion/styled": "^11.14.0",
    "react": "^19.0.0",
    "react-router-dom": "^7.1.3"
  }
}
```

Agent instructions for MUI:
```
- Import components from @mui/material
- Use sx prop: sx={{ backgroundColor: '#hexcode' }}
- Use Box, Stack, Grid for layouts
- Responsive: sx={{ width: { xs: '100%', md: '50%', lg: '33%' } }}
- Icons from @mui/icons-material
```

#### react-chakra

```json
{
  "dependencies": {
    "@chakra-ui/react": "^3.0.0",
    "@emotion/react": "^11.14.0",
    "react": "^19.0.0",
    "react-router-dom": "^7.1.3"
  }
}
```

### 6.4 The CI/CD Pipeline in Every Template

The Tailwind template ships with a complete GitHub Actions CI pipeline:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:    { branches: [main, develop] }
  pull_request: { branches: [main] }

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]   # Test on both Node versions
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "${{ matrix.node-version }}", cache: 'npm' }
      - run: npm ci                    # Clean install
      - run: npm run lint              # ESLint check
      - run: npm run format:check      # Prettier check (continue-on-error)
      - run: npm run typecheck         # TypeScript type check
      - run: npm run build             # Full Vite production build
      - uses: actions/upload-artifact@v4
        with: { name: "build-${{ matrix.node-version }}", path: dist/ }
```

**Why this matters:** The template ensures every generated project is already configured for CI/CD before the agent writes a single line of component code.

### 6.5 Why Templates vs. Generation

| Approach | Problem |
|---|---|
| Generate boilerplate from scratch (V1) | AI might use wrong Tailwind version, miss ESLint config, write broken package.json |
| Copy pre-built template (Aura2) | 100% correct setup, zero boilerplate errors, agent starts on component code immediately |

The agent's first action when opening the project is to find the existing template structure and build on it — not write configuration files.

---

## 7. Prompt Engineering for Design Fidelity

### 6.1 System Prompt

The agent's system prompt establishes it as an expert:

```python
def get_system_prompt() -> str:
    return """You are an expert React + TypeScript + Tailwind CSS developer specializing
in pixel-perfect conversion of Figma designs.

CRITICAL RULES:
1. Use EXACT hex colors from the design (e.g., #1428A0 not bg-blue-700)
2. Use EXACT font families specified (import from Google Fonts if needed)
3. Match exact padding/margin values using Tailwind arbitrary values [24px]
4. Auto-layout HORIZONTAL → flex flex-row, VERTICAL → flex flex-col
5. itemSpacing → gap-[Xpx]
6. Generate semantic HTML: header/nav/main/footer/button/article
7. Include ARIA attributes from semantic analysis
8. Add hover: and focus: states for interactive elements
9. Mobile-first responsive: sm: md: lg: xl: breakpoints
10. Import all fonts from Google Fonts in index.html
"""
```

### 6.2 Conversion Prompt Structure

The prompt gives the agent a structured representation of the design:

```
# Design: Samsung Website Redesign

## Overview
- Pages: 4
- Frames: 23
- Unique Colors: 18
- Fonts: 3
- Images: 12

## Fonts (import in index.html)
- Samsung Sharp Sans: weights 400,700
- Inter: weights 300,400,500,600,700

## Color Palette (use EXACT hex values)
- #1428A0 (used 47x, e.g. NavBar, CTA Button)
- #ffffff (used 312x, e.g. Background, Text)
- #000000 (used 89x, e.g. Body Text, Borders)
- #f4f4f4 (used 23x, e.g. Section Background)
...

## Complete Design Structure
Use this EXACT structure to recreate the design:

### Page: Home Page
#### Hero Section (FRAME) [id: 5:10] [y=0]
  **SEMANTIC: hero** → Use <section role="region"> element
  - Layout: VERTICAL, gap=24px, padding=80px 120px 80px 120px
  - Background: #1428A0
  - Size: 1440×900px

  #### Headline (TEXT) [id: 5:11]
  - Text: "Unfold Your World"
  - Font: Samsung Sharp Sans 700 72px
  - Color: #FFFFFF
  - Line height: 86.4px

  #### CTA Button (FRAME) [id: 5:12]
  **SEMANTIC: button** → Use <button role="button"> element
  - Layout: HORIZONTAL, gap=8px, padding=16px 32px
  - Background: #FFFFFF
  - Border radius: 8px all corners
  - On hover: Add box-shadow effect
```

### 6.3 The "EXACT values" Principle

The key insight in Aura2's prompt engineering: **tell the AI to use arbitrary Tailwind values** instead of nearest-class approximations:

```
Wrong (V1 style):  className="bg-blue-700 p-4 text-5xl"
Right (Aura2):     className="bg-[#1428A0] p-[80px] text-[72px]"
```

Tailwind CSS supports arbitrary values via bracket notation, enabling pixel-perfect design reproduction.

---

## 7. RAG Component Reuse System

### 7.1 Architecture

```
New Project Request
        │
        ▼
┌───────────────────┐
│  search_components │  ← Query: "blue primary button with shadow"
│  (ChromaDB query)  │
└────────┬──────────┘
         │
         ├── Similarity > 90%: "reuse_directly"
         ├── Similarity 70-90%: "adapt"
         └── Similarity < 60%: "create_new"
                │
                ▼
        Agent generates/adapts code
                │
                ▼
        save_component(name, code, category, description)
                │
                ▼
        ChromaDB stores with embedding
        (sentence-transformers/all-MiniLM-L6-v2)
```

### 7.2 ChromaDB Component Store (`backend/rag/component_store.py`)

```python
class ComponentStore:
    def __init__(self, persist_directory="./component_library/chroma"):
        self.client = chromadb.PersistentClient(
            path=str(persist_directory),
            settings=ChromaSettings(anonymized_telemetry=False)
        )
        # Uses default sentence-transformers embedding function
        self.collection = self.client.get_or_create_collection(
            name="react_components",
            metadata={"description": "Reusable React components with Tailwind CSS"}
        )

    def add_component(self, name, code, description, category, props_schema=None):
        # Create semantic document for embedding
        document = f"{name} {category} {description}"
        # If props exist: "accepts: variant, size, onClick, children"

        metadata = {
            "name": name,
            "category": category,
            "code_length": len(code),
            "usage_count": 0,
        }

        self.collection.add(
            documents=[document],      # What gets embedded
            metadatas=[metadata],
            ids=[component_id],
        )
        # Code stored separately in filesystem (too large for ChromaDB)
        code_path = self.codes_dir / f"{component_id}.tsx"
        code_path.write_text(code)

    def search_components(self, query, category=None, n_results=5):
        results = self.collection.query(
            query_texts=[query],
            n_results=n_results,
            where={"category": category} if category else None,
        )
        # Returns results with similarity scores (0.0 = identical, 2.0 = opposite)
        # Convert to 0-1 similarity: similarity = 1 - (distance / 2)
```

### 7.3 Reuse Thresholds

| Similarity Score | Action | Savings |
|---|---|---|
| > 90% | `reuse_directly` — use as-is | ~100% generation time saved |
| 70-90% | `adapt` — modify props/styles | ~60% time saved |
| 60-70% | `use_as_reference` — use as inspiration | ~30% time saved |
| < 60% | `create_new` — generate from scratch | 0% |

### 7.4 Component MCP Server

The RAG store is exposed as an MCP server to the agent:

```python
def create_component_library_server():
    return {
        "type": "stdio",
        "command": "python",
        "args": ["-m", "backend.mcp_tools.component_library"],
        "env": {...}
    }
```

Tools available:
- `search_components(query, category, n_results)` → returns similar components with similarity score
- `save_component(name, code, description, category, props_schema)` → adds to library
- `get_component(id)` → retrieves full component code

---

## 8. Visual Verification Loop

This is Aura2's most sophisticated subsystem — ensuring the generated code actually matches the Figma design.

### 8.1 Complete Loop Architecture

```
Generated Project
      │
      ▼
Start Vite Dev Server (port auto-allocated 5173-6000)
      │
      ▼
Wait for server ready (HTTP health check)
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                  ITERATION LOOP (max N)              │
│                                                     │
│  Capture Screenshot (Playwright CLI)                │
│         │                                           │
│         ▼                                           │
│  ┌─────────────────────────────────────────┐       │
│  │   Vision Comparison (if enabled)         │       │
│  │   Claude Vision API: 70% weight          │       │
│  │   Figma screenshot ←→ Generated          │       │
│  └──────────────────┬──────────────────────┘       │
│                     │                               │
│  ┌──────────────────▼──────────────────────┐       │
│  │   Content Comparison: 30% weight         │       │
│  │   - Text content present?                │       │
│  │   - Colors used correctly?               │       │
│  │   - Fonts applied?                       │       │
│  │   - Images loaded?                       │       │
│  └──────────────────┬──────────────────────┘       │
│                     │                               │
│         Combined Confidence Score                   │
│                     │                               │
│         ≥ 95%? → Early Stop ✓                      │
│         ≥ 85%? → Success ✓                         │
│         < 85%? → Generate Fixes                    │
│                     │                               │
│         Apply Fixes (Claude Agent SDK)              │
│         Hot reload → next iteration                 │
└─────────────────────────────────────────────────────┘
      │
      ▼
Stop Dev Server
      │
      ▼
Return: { status, confidence, iterations, history }
```

### 8.2 Screenshot Capture

Using Playwright CLI (not API) for subprocess isolation:

```python
async def capture_page_screenshot(port, output_path, viewport_width=1440, viewport_height=900):
    url = f"http://localhost:{port}"

    args = [
        "npx", "playwright", "screenshot", url, str(screenshot_path),
        "--full-page",                              # Scroll to capture entire page
        "--viewport-size", f"{viewport_width},{viewport_height}"  # 1440×900 (design canvas)
    ]

    result = subprocess.run(
        args,
        capture_output=True,
        text=True,
        timeout=30,
        shell=True   # Windows requires shell=True for .cmd files
    )
```

### 8.3 Vision-Based Comparison

When a Figma screenshot is available (from plugin), Claude Vision API compares them:

```python
async def compare_with_vision_api(figma_screenshot_path, generated_screenshot_path, design_data):
    client = anthropic.Anthropic()

    figma_b64 = base64.b64encode(figma_screenshot_path.read_bytes()).decode()
    generated_b64 = base64.b64encode(generated_screenshot_path.read_bytes()).decode()

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": [
                {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": figma_b64}},
                {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": generated_b64}},
                {"type": "text", "text": """Compare these two images:
Image 1: Original Figma design
Image 2: Generated React implementation

Rate accuracy (0.0-1.0) for:
- layout_accuracy: Overall layout and structure
- spacing_accuracy: Padding, margins, gaps
- color_accuracy: Colors and backgrounds
- typography_accuracy: Fonts and text styling

List specific discrepancies with:
- location (which section)
- expected vs actual values
- severity (high/medium/low)
- fix suggestion (which file, what change)

Return as JSON."""}
            ]
        }]
    )
    # Parse JSON response → discrepancies list + accuracy scores
```

### 8.4 Content-Based Comparison (Fallback)

When no Figma screenshot is available, text and code analysis:

```python
async def compare_with_figma_design(screenshot_path, design_data, project_path):
    # 1. Extract expected text from design data
    expected_texts = collect_all_text_content(design_data)

    # 2. Read all generated source files
    all_source_code = read_all_tsx_files(project_path / "src")

    # 3. Check text presence
    text_hits = sum(1 for text in expected_texts if text in all_source_code)
    text_score = text_hits / max(len(expected_texts), 1)

    # 4. Check color usage
    expected_colors = collect_colors(design_data)
    color_hits = sum(1 for color in expected_colors if color in all_source_code)
    color_score = color_hits / max(len(expected_colors), 1)

    # 5. Check font usage
    expected_fonts = collect_fonts(design_data)
    font_hits = sum(1 for font in expected_fonts if font.lower() in all_source_code.lower())
    font_score = font_hits / max(len(expected_fonts), 1)

    # Combined confidence (weighted)
    confidence = (text_score * 0.4 + color_score * 0.35 + font_score * 0.25)

    return {"confidence": confidence, "discrepancies": discrepancies, "matches": confidence > 0.85}
```

### 8.5 Fix Application

When discrepancies are found, a focused sub-agent applies fixes:

```python
# Fix prompt structure
"""
FIXES TO APPLY:
1. [HIGH] Header Section - Wrong background color
   Expected: #1428A0
   Actual: #1a1a2e
   Fix: In src/components/Header.tsx, change bg-[#1a1a2e] to bg-[#1428A0]

2. [MEDIUM] Hero Text - Wrong font size
   Expected: 72px (text-[72px])
   Actual: 56px (text-[56px])
   Fix: In src/components/Hero.tsx, change text-[56px] to text-[72px]
"""

# Sub-agent applies fixes using Write/Edit tools
options = ClaudeAgentOptions(
    model=settings.default_model,
    system_prompt="You are a pixel-perfect code fixer. Apply the specific fixes below...",
    max_turns=settings.max_fix_turns,
    allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
    permission_mode="acceptEdits",
)
```

---

## 9. Code Quality Pipeline

All quality tools are **pre-configured in the template** — no installation needed, agent just runs them.

### 9.1 ESLint Integration

After code generation, ESLint runs automatically:

```python
async def run_eslint(project_path: Path) -> dict:
    result = subprocess.run(
        ["npx", "eslint", "src/", "--format=json", "--fix"],
        cwd=project_path,
        capture_output=True,
        text=True
    )
    # Parse JSON output → errors, warnings per file
    # Auto-fix common issues (unused imports, missing semicolons)
```

ESLint config in template includes:
- `eslint-plugin-react-hooks` — enforce hooks rules
- `eslint-plugin-react-refresh` — Vite HMR compatibility
- `typescript-eslint` — TypeScript-aware linting

Available via npm script: `npm run lint` / `npm run lint:fix`

### 9.2 Prettier Formatting

```python
async def run_prettier(project_path: Path):
    subprocess.run(
        ["npx", "prettier", "--write", "src/"],
        cwd=project_path
    )
```

`.prettierrc` pre-configured in template. Available via: `npm run format` / `npm run format:check`

### 9.3 TypeScript Compilation Check

```python
async def check_typescript(project_path: Path) -> dict:
    result = subprocess.run(
        ["npx", "tsc", "--noEmit", "--strict"],
        cwd=project_path
    )
    return {"errors": parse_tsc_errors(result.stderr)}
```

Available via: `npm run typecheck`

### 9.4 Build Verification

```python
async def verify_build(project_path: Path) -> dict:
    result = subprocess.run(
        ["npm", "run", "build"],   # runs: tsc -b && vite build
        cwd=project_path,
        timeout=120
    )
    if result.returncode == 0:
        # Parse bundle size from stdout
        bundle_sizes = parse_vite_output(result.stdout)
        return {"success": True, "bundle_sizes": bundle_sizes}
    else:
        return {"success": False, "errors": result.stderr}
```

`npm run build` = TypeScript compile → Vite production bundle → `dist/` folder

### 9.5 The Quality Gauntlet Order

```
Generated code
    ↓
1. npm run lint:fix          (ESLint auto-fix)
    ↓
2. npm run format            (Prettier auto-format)
    ↓
3. npm run typecheck         (tsc --noEmit, no build)
    ↓
4. npm run build             (tsc -b && vite build, final proof)
    ↓
5. Visual verification loop  (Playwright + Claude Vision)
    ↓
Production-ready project ✓
```

---

## 10. Multi-Page Support

### 10.1 Page Addition API

```python
# Add a second page to existing project
POST /api/projects/add-website
{
  "project_name": "Samsung Page 2",
  "figma_url": "plugin://Samsung Website Redesign (Community) (Copy)",
  "add_as": "new_page",
  "parent_project_id": 33,
  "parent_project_name": "Samsung"
}
```

### 10.2 React Router Integration

When adding a page, the agent updates the router configuration:

```tsx
// App.tsx — updated by agent when new page added
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import HomePage from './pages/HomePage';
import ProductsPage from './pages/ProductsPage';  // ← Added by agent

export default function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/products" element={<ProductsPage />} />  {/* ← Added */}
      </Routes>
    </BrowserRouter>
  );
}
```

### 10.3 Component Reuse Across Pages

When Page 2 is added, the agent:
1. Searches component library for Header (already generated for Page 1)
2. Finds similarity ~95% → reuse directly
3. Imports existing component instead of regenerating
4. Only generates page-specific sections new

---

## 11. CI/CD and Deployment

### 11.1 GitHub Integration

```python
# Auto-create GitHub repo and push after generation
github_params = {
    "owner": settings.github_owner,
    "repo_name": project_name,
    "description": f"Generated by Aura2 from Figma design",
    "private": False,
    "auto_init": True,
}
# Via GitHub MCP Server
```

### 11.2 Vercel Deployment

```python
# Auto-deploy to Vercel
vercel_params = {
    "project_name": project_name,
    "framework": "vite",
    "build_command": "npm run build",
    "output_directory": "dist",
}
# Via Vercel MCP Server
```

### 11.3 Generated CI/CD Pipeline

Each project gets a `.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run build
      - run: npx tsc --noEmit
      - run: npx eslint src/ --format=compact
```

---

## 12. Backend API Architecture

### 12.1 FastAPI Application (`backend/main.py`)

22 REST endpoints organized by domain:

```
Project Management:
  POST   /api/projects/create          → Create new project
  POST   /api/projects/add-website     → Add page to existing project
  GET    /api/projects                 → List all projects
  GET    /api/projects/{id}/status     → Get conversion status
  DELETE /api/projects/{id}            → Delete project

Plugin Upload:
  POST   /api/figma/plugin-upload      → Direct plugin data endpoint

Dev Server:
  GET    /api/projects/{id}/preview-url  → Get/start preview
  POST   /api/projects/{id}/start-dev-server
  POST   /api/projects/{id}/stop-dev-server
  POST   /api/projects/{id}/build

Component Library:
  GET    /api/components               → List all components
  GET    /api/stats                    → Platform statistics
```

### 12.2 Background Task Processing

All conversions run as FastAPI background tasks to avoid HTTP timeout:

```python
@app.post("/api/projects/create")
async def create_project(request: CreateProjectRequest, background_tasks: BackgroundTasks):
    # Immediately return project ID
    project = project_store.create_project(request.project_name, status="processing")

    # Conversion runs in background
    background_tasks.add_task(
        run_conversion,
        project_id=project.id,
        figma_url=request.figma_url,
        plugin_data=request.plugin_data,
    )

    return {"project_id": project.id, "status": "processing"}

# Frontend polls /api/projects/{id}/status to track progress
```

### 12.3 Project Storage

Projects stored in `data/projects.json` (thread-safe JSON file):

```json
{
  "projects": {
    "33": {
      "id": 33,
      "name": "Samsung",
      "status": "success",
      "project_path": "generated_projects/Samsung",
      "components_generated": 4,
      "components_reused": 0,
      "conversion_time_seconds": 436.4,
      "visual_match": true,
      "verification_confidence": 0.92,
      "github_repo_url": "https://github.com/...",
      "deployment_url": "https://samsung-aura.vercel.app",
      "deployment_status": "deployed"
    }
  }
}
```

---

## 13. Frontend Dashboard

### 13.1 Stack

- React 18 + TypeScript + Vite
- Tailwind CSS v4
- shadcn/ui components
- React Query (data fetching + caching)
- React Router v6 (routing)
- Framer Motion (animations)

### 13.2 Key Pages

| Page | Route | Purpose |
|---|---|---|
| Dashboard | `/` | Project grid, stats, create new |
| Project Detail | `/projects/:id` | Status, preview, download |
| Component Library | `/components` | Browse reusable components |

### 13.3 Real-Time Status Polling

```typescript
// React Query hook for project status
function useProjectStatus(projectId: number) {
  return useQuery({
    queryKey: ['project', projectId],
    queryFn: () => api.getProjectStatus(projectId),
    refetchInterval: (data) => {
      // Poll every 2s while processing, stop when done
      return data?.status === 'processing' ? 2000 : false;
    },
  });
}
```

---

## 14. Complete Conversion Flow

Full sequence for a typical conversion (Samsung homepage, ~7 minutes):

```
t=0s    User opens Figma plugin → runs on Samsung design
t=5s    Plugin extracts 23 frames, 18 colors, 3 fonts, 12 images (base64)
t=15s   Plugin sends JSON payload (~8MB) to /api/figma/plugin-upload
t=16s   Backend receives data, creates project ID=33, starts background task

t=20s   FigmaToReactAgent initializes with Claude Opus 4.6
t=25s   Data preprocessing: RGBA→hex, auto-layout→flex mapping, semantic analysis
t=30s   RAG search: "samsung hero section navy blue" → 0 results (new project)
t=35s   Claude generates HeroSection.tsx (tool: Write)
t=60s   Claude generates NavBar.tsx (RAG: no match → generates)
t=120s  Claude generates Footer.tsx, ProductGrid.tsx
t=180s  Claude generates App.tsx, router, index.html (with font imports)
t=200s  npm install deps (React, Tailwind, React Router)
t=210s  ESLint auto-fix (23 warnings → 0 errors)
t=220s  npm run build → success (87KB gzip bundle)

t=230s  Visual verification starts
t=232s  Vite dev server starts on port 5178
t=235s  Playwright captures screenshot (1440×900, full-page)
t=238s  Vision API comparison: confidence 0.78 (several fixes needed)
t=240s  3 fixes identified: wrong header color, font size, CTA padding
t=260s  Fix agent applies 3 changes (Edit tool)
t=265s  Hot reload, next iteration
t=268s  New screenshot captured
t=272s  Vision comparison: confidence 0.94 → Success!

t=275s  Dev server stopped
t=280s  Components saved to ChromaDB (4 components)
t=285s  GitHub repo created, code pushed
t=290s  Vercel deployment triggered
t=320s  Deployment URL returned

t=436s  Project marked "success"
        confidence=0.94, components_generated=4, deployment_url="..."
```

---

## 15. MCP Internals — How Each Server Works

### 15.1 The MCP Protocol Foundation

MCP (Model Context Protocol) uses **JSON-RPC 2.0** over either stdio (local) or HTTP+SSE (remote). Aura2 uses **stdio transport** for all its MCP servers.

**Stdio transport model:**
```
Claude Agent SDK (Python process)
    │ stdin  ──────────── writes JSON-RPC requests ──────► MCP Server (child subprocess)
    │ stdout ◄─────────── reads JSON-RPC responses ──────  (Node.js / Python)
    │ stderr             (ignored — server logs go here)
```

The parent (Claude Agent SDK) spawns each MCP server as a **child subprocess** via `npx`. Messages are **newline-delimited**: each complete JSON object ends with `\n`. There is NO chunking — a 500KB base64 screenshot is one very long single line.

### 15.2 The Initialization Handshake (3-step)

Before any tool calls, this exact sequence occurs:

**Step 1 — Parent writes to child stdin:**
```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"claude-code","version":"1.0.0"}}}
```

**Step 2 — Child writes to stdout:**
```json
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-06-18","capabilities":{"tools":{}},"serverInfo":{"name":"github-mcp-server","version":"2025.4.8"}}}
```

**Step 3 — Parent sends `initialized` notification (no response expected):**
```json
{"jsonrpc":"2.0","method":"notifications/initialized"}
```

Session is now live. Then `tools/list` is called to discover available tools (with their full JSON Schema for inputs).

### 15.3 A Complete Tool Call on the Wire

```
# Parent → child stdin (tool call):
{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"push_files","arguments":{"owner":"alice","repo":"samsung-website","branch":"main","files":[{"path":"src/App.tsx","content":"export default..."}],"message":"feat: add Samsung hero component"}}}\n

# Child executes → makes 4 GitHub API calls
# Child → parent stdout (result):
{"jsonrpc":"2.0","id":5,"result":{"content":[{"type":"text","text":"{\"ref\":\"refs/heads/main\",\"object\":{\"sha\":\"abc123\"}}"}],"isError":false}}\n

# Error case (bad token):
{"jsonrpc":"2.0","id":5,"result":{"content":[{"type":"text","text":"Authentication Failed: Bad credentials"}],"isError":true}}\n
```

`isError: true` is a tool execution error (not a JSON-RPC protocol error) — Claude sees it and can self-correct.

### 15.4 GitHub MCP Server (`@modelcontextprotocol/server-github`)

**26 tools** — key ones used by Aura2:

#### `create_repository`

Calls **`POST https://api.github.com/user/repos`**:
```json
{"name": "samsung-website", "description": "Generated by Aura2", "private": false, "autoInit": true}
```
Returns repo JSON with `html_url`, `clone_url`, `id`.

Authentication: reads `process.env.GITHUB_PERSONAL_ACCESS_TOKEN` on every call, sends as `Authorization: Bearer <token>`.

#### `push_files` — 4-Step Git Data API

This is NOT the simple Contents API. It uses the **Git Data API** to create a real git commit atomically, regardless of how many files:

```
Step 1: GET  /repos/{owner}/{repo}/git/refs/heads/{branch}
             → gets current branch tip SHA

Step 2: POST /repos/{owner}/{repo}/git/trees
             → creates new tree with all files at once
        Body: {
          "base_tree": "<current-sha>",
          "tree": [
            {"path": "src/App.tsx", "mode": "100644", "type": "blob", "content": "..."},
            {"path": "src/components/Hero.tsx", "mode": "100644", "type": "blob", "content": "..."}
          ]
        }

Step 3: POST /repos/{owner}/{repo}/git/commits
             → creates the commit object
        Body: {"message": "feat: Samsung hero", "tree": "<new-tree-sha>", "parents": ["<old-sha>"]}

Step 4: PATCH /repos/{owner}/{repo}/git/refs/heads/{branch}
              → advances the branch pointer
        Body: {"sha": "<new-commit-sha>", "force": true}
```

All files in one atomic commit — no per-file API calls.

#### `create_pull_request`

Calls **`POST https://api.github.com/repos/{owner}/{repo}/pulls`**:
```json
{"title": "feat: Samsung Website from Figma", "head": "feature/aura2-generation", "base": "main", "body": "Generated by Aura2 in 436 seconds..."}
```

### 15.5 Vercel MCP (`@vercel/mcp`) — Important Caveat

> **Vercel's official MCP is a REMOTE server** at `https://mcp.vercel.com`, using **OAuth** — NOT a local stdio process with `VERCEL_TOKEN`.

The `@vercel/mcp` npm package and `VERCEL_TOKEN` env var refer to an **older/unofficial integration**. The official Vercel MCP:
- Uses **Streamable HTTP transport** (POST to MCP endpoint + SSE)
- Uses **OAuth** browser flow for authentication (not env vars)
- Can be scoped to a project: `https://mcp.vercel.com/{teamSlug}/{projectSlug}`

**Key Vercel MCP tools:**

| Tool | What it does |
|---|---|
| `deploy_to_vercel` | Triggers Vercel to build + deploy the project (like `git push`) |
| `list_deployments` | Lists deployments with state/time |
| `get_deployment` | Gets deployment status |
| `get_deployment_build_logs` | Fetches build logs — critical for debugging |
| `get_runtime_logs` | Fetches function/edge runtime logs |
| `list_projects` | Lists all projects |
| `get_project` | Project details (framework, domains, latest deploy) |

`deploy_to_vercel` triggers a **server-side build** on Vercel's infrastructure — it does NOT upload pre-built `dist/` files. Vercel detects the framework (Vite in Aura2's case), runs `npm run build`, and deploys.

### 15.6 Playwright MCP (`@playwright/mcp`) — Internal Architecture

**Key design:** `@playwright/mcp` is a thin wrapper — the actual implementation lives inside the `playwright` npm package itself (`playwright/lib/mcp/`).

**Browser lifecycle:**
```
Default mode:  One persistent browser context per MCP connection
               (persisted to disk across sessions)
Isolated mode: In-memory context, discarded when connection closes
               (flag: --isolated)
```

**The accessibility-first design:**
Playwright MCP is NOT vision-first. The recommended flow is:

```
1. browser_navigate(url)        → navigates, returns accessibility tree
2. browser_snapshot()           → returns textual accessibility tree with refs
   Output: "button[ref=e42] 'Shop Now'"
3. browser_click(ref="e42")     → clicks the button by ref (deterministic)
```

`browser_snapshot` returns a Markdown-like accessibility tree (hundreds of bytes). `browser_take_screenshot` returns a base64 PNG (hundreds of KB). For interactions, use snapshot. For visual verification, use screenshot.

**How `browser_take_screenshot` returns to Claude:**

The screenshot is returned as MCP `image` content type — base64 PNG embedded in the JSON-RPC response:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "content": [
      {
        "type": "image",
        "data": "<base64-encoded-PNG — could be 500KB+>",
        "mimeType": "image/png"
      }
    ]
  }
}
```

This entire multi-hundred-KB JSON blob is written as **one newline-terminated line** on stdout. The parent process buffers until `\n`. No chunking, no streaming within a single response.

The `--image-responses` flag controls this:
- `allow` — always send image data (default)
- `omit` — never send (return only text description)
- `auto` — send only if client declares image support

### 15.7 How the Claude Agent SDK Connects All MCP Servers

```python
# Claude Agent SDK internally does for each MCP server:
from mcp.client.stdio import stdio_client, StdioServerParameters

async with stdio_client(StdioServerParameters(
    command="npx",
    args=["-y", "@modelcontextprotocol/server-github"],
    env={"GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."}
)) as (read_stream, write_stream):
    async with ClientSession(read_stream, write_stream) as session:
        await session.initialize()     # 3-step handshake
        tools = await session.list_tools()  # tool discovery
        # tools now available to Claude during inference
```

**Environment variable handling:** The SDK passes the server's `env` dict merged on top of safe default environment variables (`PATH`, `HOME`, etc.). This is why `GITHUB_PERSONAL_ACCESS_TOKEN` in the `env` key reaches the GitHub MCP server process.

**Shutdown:** SDK closes the child's stdin → waits 2 seconds → SIGTERM → SIGKILL (Unix) or Job Object termination (Windows).

### 15.8 MCP Message Types

| Type | Has `id`? | Direction | Purpose |
|---|---|---|---|
| Request | Yes | Either | Expects a response |
| Response | Yes (matches request) | Opposite | Returns result or error |
| Notification | No | Either | Fire-and-forget, no response |

**Tool execution errors** use `isError: true` in result (not JSON-RPC error codes) — Claude can read and self-correct:
```json
{"jsonrpc":"2.0","id":5,"result":{"content":[{"type":"text","text":"Rate limit exceeded"}],"isError":true}}
```

**Protocol errors** use `error` object (for protocol violations, not tool failures):
```json
{"jsonrpc":"2.0","id":5,"error":{"code":-32602,"message":"Unknown tool: invalid_tool_name"}}
```

---

## 16. Aura2 vs. Industry Tools

### 15.1 Current Market Landscape (2025)

| Tool | Figma Integration | Code Quality | Framework | AI Architecture |
|---|---|---|---|---|
| **Aura2** | Native Plugin (no rate limits) | ESLint + Prettier + TS verified | React + Tailwind | Claude Agent SDK + RAG + Vision verification |
| **Builder.io Visual Copilot** | Native Plugin | High (pixel-perfect claim) | React, Vue, Angular, Svelte, Qwik | 3-stage: 2M+ dataset model → Mitosis compiler → LLM refine |
| **Vercel v0** | None (text/screenshot only) | High (React + shadcn/ui) | React, Next.js | LLM-based (not agentic), no design data |
| **Locofy.ai** | Native Plugin | High for React+Tailwind | React, Next.js, Vue, React Native | AI-assisted with manual tagging |
| **Anima** | Native Plugin | High pixel fidelity | React, HTML/CSS, Vue, TypeScript | AI-assisted |
| **Cursor + Figma MCP** | Official Figma MCP server | Depends on model | Any (IDE context) | Fully agentic |

### 15.2 Figma's Official MCP Server (2025)

In 2025, Figma launched its **Dev Mode MCP server** — the most significant development for design-to-code:
- Exposes design metadata (frames, components, design tokens, layout constraints) to AI agents
- Works with Claude Code, Cursor, Windsurf, VS Code Copilot, GitHub Copilot
- Two modes: **Remote MCP** (Figma-hosted) and **Desktop MCP** (local, via Figma desktop app)
- Rate limits still apply (same as REST API for View seats)

Aura2's Plugin approach predates this and has an advantage: **zero rate limits** (Plugin API runs inside Figma's memory space, no REST calls).

### 15.3 MCP Ecosystem Context

MCP (Model Context Protocol) was announced by Anthropic in **November 2024** as an open standard:
- More than **10,000 active public MCP servers** by late 2025
- Adopted by: Claude Code, Cursor, ChatGPT, VS Code Copilot, Windsurf, Gemini
- In **December 2025**, Anthropic donated MCP to the **Agentic AI Foundation (AAIF)** under the Linux Foundation (co-founded with Block and OpenAI)

Aura2 uses MCP for:
- `@playwright/mcp` — browser control during verification
- `component_library` MCP — custom RAG server (ChromaDB-backed)
- GitHub MCP — repo creation and PR management
- Vercel MCP — deployment automation

---

## 16. Performance Metrics

Based on real Samsung PRISM conversions:

| Metric | Value |
|---|---|
| Average conversion time | 436 seconds (~7 min) |
| Visual confidence achieved | 88-96% |
| Component reuse rate (multi-page) | 60-80% |
| Build success rate (after verification) | 95%+ |
| TypeScript errors after generation | <5 |
| ESLint warnings auto-fixed | 90%+ |
| Generated bundle size | 50-150KB gzip |
| Color accuracy (RGBA→hex) | 100% (mathematical conversion) |
| Font accuracy | 95%+ (direct extraction) |
| Layout accuracy | 85-90% (flex mapping) |
