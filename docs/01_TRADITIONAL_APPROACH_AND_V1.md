# From Figma to Code: The Traditional Approach & The V1 Problem

> **Context:** Aura2 — AI-Powered Figma → React Converter
> **Samsung PRISM @ IIIT Naya Raipur**
> This document covers: how design-to-code was done before AI, the V1 pain points, and why a new approach was needed.

---

## Table of Contents

1. [The Design-to-Code Problem](#1-the-design-to-code-problem)
2. [The Traditional Manual Workflow](#2-the-traditional-manual-workflow)
3. [Design Handoff Tools (Pre-AI Era)](#3-design-handoff-tools-pre-ai-era)
4. [First-Generation Automation Tools](#4-first-generation-automation-tools)
5. [The Figma REST API: How It Works and Its Limits](#5-the-figma-rest-api-how-it-works-and-its-limits)
6. [First-Gen AI Tools and Their Failures](#6-first-gen-ai-tools-and-their-failures)
7. [The V1 Architecture: What We Built First](#7-the-v1-architecture-what-we-built-first)
8. [Why V1 Was Not Enough](#8-why-v1-was-not-enough)
9. [The Core Unsolved Challenges](#9-the-core-unsolved-challenges)

---

## 1. The Design-to-Code Problem

Converting a designer's vision (in Figma) into working, production-quality code has always been one of software development's most painful bottlenecks.

### The Gap

```
Designer (Figma)          Developer (Code)
      |                         |
   Pixels                    Bytes
  Hex colors                CSS values
  Auto-layout               Flexbox/Grid
  Components                React components
  Fonts                     @font-face
  Effects                   box-shadow, blur
      |_________________________|
              GAP: hours/days of manual work
```

### Why It's Hard

| Design Concept | Code Equivalent | Complexity |
|---|---|---|
| Fill color (RGBA 0-1) | hex or rgba CSS | Requires conversion |
| Auto-layout (HORIZONTAL) | `display: flex; flex-direction: row` | Requires mapping |
| Item spacing | `gap: Xpx` | Direct but needs extraction |
| Padding (per side) | `padding: top right bottom left` | 4 values to extract |
| DROP_SHADOW effect | `box-shadow: X Y blur spread color` | 5 parameters to map |
| Corner radius (per corner) | `border-radius: tl tr br bl` | Individual or shorthand |
| Text style | font-family, size, weight, line-height, letter-spacing | 5+ CSS properties |
| Constraints (SCALE) | `width: %` or `flex-grow` | Requires inference |
| Image fills | `background-image: url()` | Image export needed |
| Blend modes | CSS `mix-blend-mode` | 1:1 but easy to miss |

---

## 2. The Traditional Manual Workflow

Before any automation existed, the workflow was entirely manual — and brutally slow.

### Step 1: Design Review (30-60 min per component)

A developer would open the Figma file in inspect mode and manually read:
- Each frame's dimensions and position
- Each color value (copy as hex)
- Font names and their weights
- Padding values from each side
- Whether auto-layout was used

### Step 2: Asset Export (15-30 min per page)

- Manually select each image/icon in Figma
- Export as PNG or SVG, one at a time
- Rename files to match code conventions
- Optimize/compress if needed

### Step 3: HTML/CSS Writing (2-8 hours per page)

```css
/* Developer manually translating Figma "Hero Section" */
.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 80px 120px; /* manually read from Figma */
  background-color: #1a1a2e; /* manually copied hex */
  gap: 24px; /* manually read itemSpacing */
}

.hero h1 {
  font-family: 'Inter', sans-serif; /* manually noted */
  font-size: 56px; /* manually read fontSize */
  font-weight: 700; /* manually read fontWeight */
  color: #ffffff;
  line-height: 1.2; /* calculated from lineHeightPx / fontSize */
}
```

### Step 4: The Pixel-Matching Game (4-16 hours)

After writing code, the developer would:
1. Open browser side by side with Figma
2. Adjust values pixel by pixel
3. Compare every section visually
4. Repeat until "close enough"

**Result:** A typical landing page took **2-5 business days** to convert.

### The Human Error Rate

Typical errors in manual conversion:
- Wrong color: `#1a1a2e` typed as `#1a1a3e` (one character)
- Wrong font weight: 600 vs 700
- Missing padding on one side (e.g., `padding-right` forgotten)
- Incorrect line-height (px vs unitless ratio)
- Auto-layout direction wrong (column vs row)
- Z-index stacking wrong

---

## 3. Design Handoff Tools (Pre-AI Era)

Several tools emerged to bridge the design-to-developer gap, but none solved the problem completely.

### 3.1 Zeplin (2015)

**What it did:**
- Connected to Figma (and Sketch) to display design specs
- Developers could click any element to see CSS values
- Automatically showed: colors, fonts, spacing, sizes
- Generated basic CSS snippets

**Limitations:**
- Still required developer to manually write all code
- CSS snippets were incomplete (no layout, no responsive)
- No component concept — just flat CSS
- No code generation, just spec display

```
Zeplin output for a button:
----------------------------
Background: #6200EE
Border Radius: 4px
Padding: 14px 24px
Font: Roboto 500 14px
Color: #FFFFFF
----------------------------
Developer still had to write all React code manually.
```

### 3.2 InVision Inspect (2017)

Similar to Zeplin — showed design specs to developers. Added:
- Comment threads between designers and developers
- Version comparison

Still: zero code generation.

### 3.3 Figma Dev Mode (2023)

Figma's own developer-facing feature:
- Built-in CSS/iOS/Android code snippets
- Design tokens export
- Component inspection
- Link to specific code sections

**Still lacked:**
- No full component generation
- Snippets were only for individual properties, not complete components
- No React/TypeScript awareness
- No layout code generation

---

## 4. First-Generation Automation Tools

### 4.1 Anima (2018-2022)

**Approach:** Plugin inside Figma that generated HTML/CSS.

**What it did well:**
- Extracted Figma node tree
- Generated static HTML/CSS files
- Preserved visual appearance somewhat

**Critical failures:**
```html
<!-- Anima-generated code (actual example style) -->
<div style="position: absolute; left: 120px; top: 240px; width: 1200px; ...">
  <div style="position: absolute; left: 0px; top: 0px; width: 100%;">
    <p style="font-size: 56px; font-family: Inter;">Title</p>
  </div>
</div>
```

Problems:
- **Absolute positioning everywhere** — not responsive at all
- Hard-coded pixel values for every element
- No semantic HTML (no `<header>`, `<nav>`, `<button>` — just `<div>`)
- No React components — just inline-styled HTML
- No maintainability — 3000 lines of spaghetti CSS
- No component reuse

### 4.2 html.to.design / Locofy.ai (2021-2023)

**Approach:** AI-assisted but still largely rule-based.

Locofy tried to be smarter:
- Recognized button patterns
- Could generate basic React/Vue components
- Allowed developers to "tag" design elements

**Remaining problems:**
- Required extensive manual tagging
- Code still unmaintainable at scale
- No context awareness across components
- Poor handling of complex layouts
- No verification that generated code matched design

### 4.3 Builder.io Visual Copilot (2023)

More advanced: used LLMs + a custom trained model.

**Three-stage pipeline:**
1. Initial model trained on **2M+ data points** transforms flat Figma structures into code hierarchies
2. **Mitosis** (open-source compiler) compiles the structured hierarchy into framework-specific code
3. Fine-tuned LLM refines the output for your specific framework

**Better:**
- Used Figma's API to extract design data
- Generated React + Tailwind code
- Understood component boundaries
- Multi-framework: React, Angular, Svelte, Vue, Qwik, HTML

**Still limited:**
- Basic Figma API calls (rate limited)
- No iterative visual verification loop (no screenshot comparison)
- Partial component reuse (maps to existing repo, but no cross-project RAG)
- No multi-page project management from scratch
- Hit Figma API rate limits constantly for heavy designs

---

## 5. The Figma REST API: How It Works and Its Limits

Understanding the API is key to understanding why V1 was constrained.

### 5.1 Figma API Architecture

```
Developer                    Figma Servers
    |                              |
    |-- GET /v1/files/{key} ------>|
    |                              |-- Read file JSON
    |<-- 200 OK (JSON) ------------|
    |
    |-- GET /v1/images/{key} ----->|
    |   (export request)           |-- Render images
    |<-- 200 OK {images: {}}  -----|
    |
    |-- GET (s3 URL from above) -->|  (AWS S3)
    |<-- PNG/JPG binary -----------|
```

### 5.2 The JSON Data Structure

A Figma file JSON looks like:

```json
{
  "document": {
    "id": "0:0",
    "name": "Document",
    "type": "DOCUMENT",
    "children": [
      {
        "id": "1:2",
        "name": "Page 1",
        "type": "CANVAS",
        "children": [
          {
            "id": "5:10",
            "name": "Hero Section",
            "type": "FRAME",
            "absoluteBoundingBox": {"x": 0, "y": 0, "width": 1440, "height": 900},
            "layoutMode": "VERTICAL",
            "paddingTop": 80, "paddingBottom": 80,
            "paddingLeft": 120, "paddingRight": 120,
            "itemSpacing": 24,
            "fills": [{"type": "SOLID", "color": {"r": 0.1, "g": 0.1, "b": 0.18, "a": 1}}],
            "children": [...]
          }
        ]
      }
    ]
  },
  "styles": {...},
  "components": {...}
}
```

The JSON can be **10-50MB** for complex designs.

### 5.3 Rate Limits — The Core Problem

Figma's REST API rate limits by seat type:

Figma uses a **leaky bucket algorithm** for rate limiting. The actual tier rates by seat type:

| Tier | View/Collab Seat | Dev/Full Starter | Dev/Full Professional | Dev/Full Org |
|---|---|---|---|---|
| Tier 1 | Up to 6/month | 10/min | 15/min | 20/min |
| Tier 2 | Up to 5/min | 25/min | 50/min | 100/min |
| Tier 3 | Up to 10/min | 50/min | 100/min | 150/min |

When exceeded: **HTTP 429** with headers: `Retry-After`, `X-Figma-Plan-Tier`, `X-Figma-Rate-Limit-Type`, `X-Figma-Upgrade-Link`.

> **2025 Update:** Figma introduced **stricter rate limits effective November 2025**, further tightening API access for View/Collab seats. This made the Plugin API bypass even more critical.

**Real-world impact:**
- A complex Samsung design with 50+ frames triggered 429 errors
- Image export (separate API call per image) exhausted quota immediately
- Retry-After headers sometimes specified 60+ seconds
- A single conversion attempt could consume the month's quota

```python
# What developers had to build just to handle rate limits:
class FigmaRateLimiter:
    def __init__(self):
        self.request_times = []
        self.max_requests = 10  # per minute

    async def make_request(self, url, headers):
        # Exponential backoff
        for attempt in range(5):
            try:
                response = await httpx.get(url, headers=headers)
                if response.status_code == 429:
                    wait = int(response.headers.get('Retry-After', 60))
                    await asyncio.sleep(wait)
                    continue
                return response
            except Exception:
                await asyncio.sleep(2 ** attempt)
```

This was built in Aura2's `backend/utils/figma_rate_limiter.py`.

---

## 6. First-Gen AI Tools and Their Failures

### 6.1 The "Screenshot to Code" Approach (screenshot-to-code, 2023)

Tools like `screenshot-to-code` (open source) would:
1. Take a screenshot of a website
2. Feed it to GPT-4V or Claude
3. Ask the AI to reproduce it in HTML/Tailwind

**Fatal flaw:** The AI had NO design data. It was guessing:
- Colors from visual pixels (imprecise)
- Font sizes by estimation
- Spacing by visual inspection
- No component structure

Output: visually approximated but completely wrong CSS values.

### 6.2 Plain Claude/GPT with Design Description

Early prompting approach:
```
"Here's my Figma design. The hero section has a dark navy background,
big white heading, some body text, and a CTA button..."
```

Problems:
- AI had no actual data — only verbal description
- Hallucinated colors, sizes, fonts
- No access to actual Figma node tree
- No way to verify the output matched design

### 6.3 The V1 Naïve Agentic Approach

An early V1 of Aura2 would:
1. Take Figma URL
2. Call Figma REST API (one big GET request)
3. Pass raw JSON to Claude API
4. Ask Claude to write React code

**Problems discovered:**
- **Context window overflow**: 50MB JSON > 200K token limit
- **No design data preprocessing**: Raw Figma JSON is noisy with internal IDs
- **Rate limits hit immediately**: Image exports broke quota
- **No verification**: Generated code was never tested
- **No component reuse**: Every project generated from scratch
- **Hallucination on complex data**: Claude would misread nested structures

---

## 7. The V1 Architecture: What We Actually Built First

V1 (at `C:\Manas\code\AI\Aura`) was far more sophisticated than a naive API wrapper — it was a full multi-agent system with LangGraph, local LLMs, and advanced RAG. But it had fundamental architectural mismatches with the Samsung PRISM requirements.

### 7.1 V1 Tech Stack

```
Local LLMs (Ollama):
├── Qwen 2.5 7B       — Reasoning (extraction, layout, responsive)
├── Llama 3.1 8B      — Creative generation (styling, consensus)
└── Qwen2.5-Coder 32B — Code generation (also 7B/14B options)

Cloud LLM (Fallback):
└── Gemini 2.5-Flash  — Highest-quality code generation

Orchestration:
└── LangGraph StateGraph — 10-node workflow graph

RAG/Knowledge Base:
└── ChromaDB + Sentence Transformers (all-MiniLM-L6-v2)
    13-module advanced RAG system

Figma Data:
└── REST API only (no plugin, no rate limit bypass)
```

### 7.2 The LangGraph 10-Node Workflow

V1's conversion pipeline was a LangGraph StateGraph with 10 sequential + conditional nodes:

```
[Figma URL]
    ↓
Node 1: Design Extraction (Qwen 2.5 7B)
    — Extract ALL elements, map to semantic JSX tags
    — Batch processing for >30 elements (batches of 15-30)
    — Handled 196+ element components via batching
    ↓
Node 2: Reuse Check (ChromaDB RAG)
    — Filter by figma_file_key (project isolation)
    — 0.98 threshold: exact match → skip to Node 10
    — 0.70 threshold: partial match → adapt
    ↓
Node 3: Layout Analysis (Qwen 2.5 7B)
    — Detect flex/grid/stack layout patterns
    ↓
Node 4: Responsive Analysis
    — Detect mobile (375px) / tablet (768px) / desktop (1280px) breakpoints
    ↓
Node 5: Styling Generation (Llama 3.1 8B)
    — Map Figma design tokens → Tailwind classes
    ↓
Node 6: Consensus Builder (470 lines)
    — Up to 3 voting rounds between agents
    — LLM arbitration for conflicts
    ↓
Node 7: Style Integration
    — Merge base + state (hover/focus/active) + responsive styles
    ↓
Node 8: Element Synthesis (Qwen2.5-Coder 32B)
    — Hierarchical bottom-up generation
    — Children generated before parents
    — Prevents nested empty divs
    ↓
Node 9: Confidence Scoring
    — 0.90+: Auto-approve
    — 0.75-0.90: Soft review
    — 0.65-0.75: Requires review
    — <0.65: Escalate to human
    ↓
Node 10: Finalization
```

### 7.3 V1 Post-Workflow Processing

After the LangGraph workflow, V1 had additional processing stages:

```
Generated components
    ↓
Component Organizer — Professional folder structure
    src/components/ui/       (buttons, inputs, cards)
    src/components/layout/   (headers, footers, navbars)
    src/pages/               (page-level components)
    ↓
Import Path Fixing — Regex-based after reorganization
    ↓
Test Generation — Vitest test files per component
    ↓
Story Generation — Storybook stories
    ↓
App.tsx Integration — Main entry point
    ↓
Build Validation — npm run build, error parsing
    ↓
Build Fixer — Auto-fix 80%+ of TypeScript/import errors
    (Up to 3 fix attempts, deterministic + AI-powered fixes)
```

### 7.4 V1 RAG System (13 Modules)

V1 had a **more advanced RAG system** than Aura2:

```
backend/rag/
├── knowledge_base.py          — Main KB interface
├── hybrid_retriever.py        — Multi-strategy retrieval
├── multi_perspective_scorer.py — 4D similarity scoring:
│                                 1. Semantic similarity
│                                 2. Structural features
│                                 3. Visual style matching
│                                 4. Behavioral patterns
├── semantic_chunker.py        — Intelligent code chunking
├── structured_features.py     — Code structure extraction
├── visual_style_matcher.py    — Design token matching
└── reranker.py                — Result reranking
```

**Project Isolation:** Every KB query filtered by `figma_file_key` — Project A's components never appeared in Project B's results.

**Variant Necessity Scoring:**
- Score < 0.3: Reuse existing directly
- Score 0.3-0.7: Create a variant
- Score > 0.7: Create new component

### 7.5 V1 Observability System

V1 had observability that Aura2 doesn't have:

- **Phoenix tracer** — Full agent execution tracing
- **RAGAS evaluator** — RAG quality evaluation
- **Visual regression** framework (Playwright + PIL + pixelmatch) — 4 breakpoints tested (375px, 768px, 1280px, 1920px)
- **NOTE:** The visual regression framework existed but was NOT integrated into the main workflow

### 7.6 V1 Code Generation Output

```tsx
// V1 generated code — typical output (better structure than naive tools)
// But still had issues:
export default function HeroSection() {
  return (
    <div className="w-full bg-[#1a1a2e] flex flex-col items-center">
      <h1 className="text-white text-5xl font-bold">
        Samsung Galaxy S24
      </h1>
      <p className="text-gray-300 text-lg">
        The Next Generation
      </p>
      <button className="bg-blue-600 text-white px-8 py-3 rounded">
        Shop Now
      </button>
    </div>
  );
}
```

Issues that persisted even in V1:
- Generic Tailwind colors (`bg-blue-600`) instead of exact hex (`bg-[#1428A0]`)
- No font imports (browser defaults)
- No hover/focus states
- No accessibility attributes (ARIA)
- No MCP tool integration
- Visual regression framework existed but not connected

### 7.7 V1 Performance Metrics

- **Single component generation**: 15-20 seconds
- **5 components**: 1.5-2 minutes
- **Large design (162 components)**: ~60 seconds
- **Build success rate**: 95%+ (with auto-fixer)
- **Confidence scores**: 0.68-1.0 range
- **Test files**: 39 test files, multiple categories
- **Frontend**: Basic — only 2 component files, no project dashboard

---

## 8. Why V1 Was Not Enough

### 8.1 V1 vs Aura2 — Complete Comparison

| Feature | V1 (Aura) | Aura2 |
|---|---|---|
| **Orchestration** | LangGraph 10-node StateGraph | Claude Agent SDK (agentic loop) |
| **LLM** | Ollama (Qwen/Llama local) + Gemini fallback | Claude Opus 4.6 via LiteLLM proxy |
| **Figma access** | REST API only (rate limited) | Plugin API (no rate limits) + REST fallback |
| **Rate limit handling** | No explicit logic (60 req/min basic) | Exponential backoff + Plugin bypass |
| **RAG system** | 13-module multi-perspective (4D scoring) | ChromaDB with simpler semantic search |
| **Component reuse** | Variant necessity scoring, 3 thresholds | >90% reuse, 70-90% adapt, <60% new |
| **Visual verification** | Framework existed, NOT connected | Fully integrated Playwright + Claude Vision loop |
| **MCP tools** | None (Playwright commented out only) | Playwright + GitHub + Vercel + component library |
| **Template system** | Generated boilerplate from scratch | Pre-built templates copied (react-tailwind/mui/chakra) |
| **Multi-page** | None | Parent-child project with React Router |
| **Frontend UI** | 2 files, no dashboard | Full dashboard with React Query + routing |
| **CI/CD** | None | GitHub Actions pre-baked in template |
| **Deployment** | None | Vercel MCP auto-deploy |
| **Build fixing** | AI-powered auto-fixer (80%+ success) | ESLint auto-fix + agent fixes |
| **Test generation** | Vitest tests + Storybook stories | None (planned) |
| **Observability** | Phoenix tracer + RAGAS evaluator | Conversion logger |
| **Batch processing** | Yes (196+ element components) | Prompt-based (no explicit batching) |
| **Consensus building** | Multi-agent voting (3 rounds) | Single agent with tool access |

### 8.2 The Samsung PRISM Requirements That Drove V2

The Samsung PRISM academic project had strict requirements V1 didn't satisfy:

1. **Bypass Figma rate limits** — V1 used REST API only, hit 429 after ~3 conversions/day
2. **Multi-page websites** — V1 had no way to add Page 2 to an existing project
3. **Exact color/font/spacing fidelity** — V1 used approximate Tailwind classes, not arbitrary values
4. **Visual verification loop** — V1's Playwright framework existed but wasn't connected
5. **Deployment pipeline** — V1 had no GitHub/Vercel integration
6. **Dashboard for project management** — V1 had only 2 frontend files

### 8.3 The Key Missing Pieces

**1. Plugin API for rate limit bypass:**
V1 had no Figma Plugin. Every conversion hit the REST API directly. After 3-4 conversions, 429 errors blocked all work.

**2. Verified visual output:**
V1 generated code and stopped. The visual regression module existed (Playwright + pixelmatch at 4 breakpoints) but was never wired into the conversion workflow. Users had to manually check if output matched design.

**3. MCP tool integration:**
V1 mentioned Playwright MCP in a comment but never implemented it. No GitHub push, no Vercel deploy — everything was manual after code generation.

**4. Exact design values:**
V1 mapped colors to nearest Tailwind preset (`bg-blue-600`). Aura2 uses arbitrary values (`bg-[#1428A0]`) for pixel-perfect accuracy.

**5. Pre-built templates:**
V1 generated the Vite project structure from scratch, leading to inconsistencies. Aura2 copies a tested template that includes ESLint, Prettier, TypeScript configs, and CI/CD workflows pre-configured.

---

## 9. The Core Unsolved Challenges

After V1, these were the concrete problems Aura2 needed to solve:

| Problem | V1 Status | Required Solution |
|---|---|---|
| Figma API rate limits | Constantly hit 429 | Plugin API bypass |
| Context window limits | 50MB JSON crashed | Smart data extraction |
| Color accuracy | ~50% correct | RGBA→hex pipeline |
| Font accuracy | ~60% correct | Complete text style extraction |
| Layout accuracy | ~45% correct | Auto-layout → Flexbox mapping |
| Component reuse | 0% | RAG vector store |
| Visual verification | None | Playwright + Claude Vision |
| Multi-page support | None | Page management system |
| Responsive design | 0% | Mobile-first generation |
| Build errors | ~50 errors/project | ESLint + build verifier |
| Semantic HTML | All divs | Semantic type inference |
| Accessibility | None | WCAG + ARIA system |

These 12 challenges drove the complete redesign into Aura2.

---

## 10. Figma Node Types & Why They're Complex

The Figma Plugin API defines **37 distinct node types**, each with unique properties:

**Document hierarchy:**
```
DocumentNode → PageNode → SceneNode subtypes (35 types)
```

**Core design nodes:**
- `FrameNode` — the workhorse container (like `<div>`)
- `GroupNode` — logical grouping, no layout impact
- `ComponentNode` — master component definition
- `ComponentSetNode` — group of variant components
- `InstanceNode` — component instance
- `TextNode`, `RectangleNode`, `EllipseNode`, `VectorNode`, `BooleanOperationNode`, etc.

**Properties that create CSS translation complexity:**

| Figma Property | CSS Problem |
|---|---|
| Fill type: Image/Gradient | Requires coordinate transform matrix conversion |
| Blend modes (27 types) | Maps to `mix-blend-mode` but 27 values to handle |
| "Corner smoothing" (squircle) | **No CSS equivalent** — iOS-style rounded corners |
| Layer opacity vs. Fill opacity | Two separate values, one CSS `opacity` |
| Auto-layout wrapping | Figma v3 wrapping ≠ CSS flexbox wrap exactly |
| Constraint types (SCALE, CENTER) | Requires relative/percentage CSS generation |
| Text `lineHeight: AUTO` | Browser default (≠ Figma's auto calculation) |
| Typography `ORIGINAL` casing | Different from CSS `normal` |

**Corner Smoothing — The Unsolvable Problem:**
Figma supports "smooth corners" (squircle rounding used in iOS design). This creates corners that are mathematically different from CSS `border-radius`. No CSS property reproduces this — it requires SVG clip-path or a workaround library.

## 11. Design-to-Code Timeline

| Year | Event |
|---|---|
| 2016 | Figma launches with Inspect panel; Zeplin gains widespread adoption |
| 2017 | Anima launches as first automated Figma-to-HTML converter |
| 2019 | Figma REST API goes public; Auto-Layout v1 introduced; Builder.io releases figma-html (later abandoned, 3.6k GitHub stars) |
| 2021 | Auto-Layout v3 with wrapping released — significantly improved design-to-code mapping |
| 2022 | Locofy.ai launches AI-powered approach; Builder.io launches Visual Copilot (using Mitosis compiler) |
| 2023 (June) | Figma Dev Mode launched at Config 2023 — dedicated developer interface |
| 2023 | Figma Variables introduced — native design token support in Figma |
| 2024 | Figma Code Connect released — real production code snippets in Dev Mode (Org/Enterprise only) |
| 2024 | Claude Computer Use public beta (Oct 22); Figma MCP server launched |
| 2025 (Nov) | Stricter Figma API rate limits; MCP donated to Agentic AI Foundation |

---

## Summary

The journey from "paste Figma URL → get code" to production-quality AI-powered conversion required solving fundamental problems:

1. **Data access**: REST API rate limits made automation impractical → needed Plugin API
2. **Data quality**: Raw Figma JSON was too noisy for LLMs → needed preprocessing pipeline
3. **Code quality**: Naive prompting produced wrong colors, fonts, layouts → needed structured prompt engineering
4. **Verification**: No feedback loop → needed visual comparison system
5. **Scale**: Each page rebuilt from scratch → needed component reuse with RAG
6. **Enterprise features**: No CI/CD, no versioning, no deployment → needed full pipeline

Aura2 was built to solve all of these. See `02_AURA2_HOW_WE_SOLVE_IT_NOW.md` for the full solution.
