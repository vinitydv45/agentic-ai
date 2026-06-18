# How to Verify Accuracy: A Practical Guide

> **Project:** Aura2 — AI-Powered Figma → React Converter
> **Samsung PRISM @ IIIT Naya Raipur**
> This document is a hands-on guide for developers: where to find Figma JSON, generated code, verification reports, and how to manually spot-check accuracy.
>
> 📘 **For conceptual background** (what the verifier checks, why ΔE2000, how the loop works),
> read [**VERIFICATION_GUIDE.md**](./VERIFICATION_GUIDE.md) first. This file is the operational
> how-to.

---

## Table of Contents

1. [Where to Find Figma JSON](#1-where-to-find-figma-json)
2. [API Endpoints for Figma Data](#2-api-endpoints-for-figma-data)
3. [Where to Find Generated Code](#3-where-to-find-generated-code)
4. [Manual Spot-Check Walkthrough](#4-manual-spot-check-walkthrough)
5. [Reading Verification Reports](#5-reading-verification-reports)
6. [Re-Running Verification](#6-re-running-verification)
7. [Component Reuse and Library Stats](#7-component-reuse-and-library-stats)

---

## 1. Where to Find Figma JSON

Every conversion persists Figma data to the project's `figma_data/` directory. This is your primary source for verifying what the system extracted from the original design.

### File Locations

```
generated_projects/{project_name}/
├── figma_data/
│   ├── design_data.json           ← Processed design data (this is what the agent sees)
│   ├── raw_figma_response.json    ← Raw Figma API response or plugin payload
│   └── design_metadata.json       ← Metadata: source, timestamp, stats
├── src/
│   ├── components/                ← Generated React components
│   └── pages/                     ← Generated page components
├── public/
│   └── images/                    ← Downloaded Figma images
└── screenshots/                   ← Design and verification screenshots
```

### design_data.json

This is the normalized design data that drives the entire conversion. It contains:

```json
{
  "name": "My Design",
  "pages": [
    {
      "name": "Page 1",
      "frames": [
        {
          "id": "123:456",
          "name": "Header",
          "type": "FRAME",
          "layout": {
            "bounds": {"x": 0, "y": 0, "width": 1440, "height": 80},
            "padding": {"top": 16, "right": 24, "bottom": 16, "left": 24},
            "mode": "HORIZONTAL",
            "itemSpacing": 12
          },
          "fills": [{"type": "SOLID", "color": {"r": 0.1, "g": 0.1, "b": 0.2}}],
          "children": [...]
        }
      ]
    }
  ],
  "colors": [
    {"color": "#1A1A33", "count": 15, "contexts": ["Header background", "Footer"]}
  ],
  "fonts": [
    {"family": "Inter", "weights": [400, 500, 600, 700]}
  ],
  "imageRefs": {
    "abc123": "public/images/hero-image.png"
  },
  "stats": {
    "pageCount": 2,
    "frameCount": 8,
    "colorCount": 12,
    "fontCount": 3,
    "imageCount": 5
  }
}
```

### raw_figma_response.json

The unprocessed payload from either:
- **Figma REST API:** Full API response including document tree, styles, components.
- **Figma Plugin:** The raw plugin extraction payload including base64 images and screenshots.

This file can be very large (10MB+ for complex designs). Use it when you need to verify that the extraction step (`extract_complete_design_data`) correctly interpreted the raw data.

### design_metadata.json

```json
{
  "saved_at": "2026-03-20T10:30:00+00:00",
  "source": "plugin",
  "file_key": "",
  "stats": {
    "pageCount": 2,
    "frameCount": 8,
    "colorCount": 12,
    "fontCount": 3,
    "imageCount": 5
  }
}
```

The `source` field tells you whether data came from the REST API (`"api"`) or the Figma Plugin (`"plugin"`).

---

## 2. API Endpoints for Figma Data

The backend exposes REST endpoints for programmatic access to Figma data. The project ID is an integer assigned when the project is created.

### Get Processed Design Data

```
GET /api/projects/{project_id}/figma-json
```

Returns the contents of `design_data.json` — the normalized design data that was fed to the agent.

**Example:**
```bash
curl http://localhost:8000/api/projects/1/figma-json | jq '.pages[0].frames[0].name'
```

### Get Raw Figma Response

```
GET /api/projects/{project_id}/figma-json/raw
```

Returns the contents of `raw_figma_response.json` — the unprocessed API response or plugin payload.

**Example:**
```bash
curl http://localhost:8000/api/projects/1/figma-json/raw | jq '.document.children | length'
```

### Get Platform Stats (Includes Reuse Counts)

```
GET /api/stats
```

Returns aggregate statistics:
```json
{
  "total_projects": 5,
  "completed_projects": 4,
  "total_components": 42,
  "total_component_reuses": 18
}
```

### List All Components in Library

```
GET /api/components?category=navigation&limit=50
```

Returns components stored in the ChromaDB library, optionally filtered by category.

### Get Individual Project Details

```
GET /api/projects/{project_id}
```

Returns project metadata including `components_generated`, `components_reused`, and `conversion_time_seconds`.

---

## 3. Where to Find Generated Code

### Components

```
generated_projects/{project_name}/src/components/
```

Each Figma frame or reusable element becomes a `.tsx` file. For example:
- `Header.tsx` — top-level navigation/header frame
- `HeroSection.tsx` — hero banner
- `Card.tsx` — reusable card component
- `Footer.tsx` — footer frame

### Pages

```
generated_projects/{project_name}/src/pages/
```

When using multi-page mode (`add_as="new_page"`), each conversion creates a new page file that imports shared components from `../components/`.

### App Entry Point

```
generated_projects/{project_name}/src/App.tsx
```

The main app component with routing (if multiple pages exist).

### Static Assets

```
generated_projects/{project_name}/public/images/
```

All images downloaded from Figma (via REST API) or decoded from base64 (via plugin).

### Screenshots

```
generated_projects/{project_name}/screenshots/
```

Contains:
- `design_screenshot.png` — the original Figma design screenshot (if captured by plugin)
- Verification screenshots captured during the verification loop

---

## 4. Manual Spot-Check Walkthrough

Use this step-by-step process to verify that a generated component faithfully reproduces the Figma design.

### Step 1: Open the Design Data

```bash
# Using jq for readable output
cat generated_projects/my-app/figma_data/design_data.json | jq '.'
```

Or via the API:
```bash
curl http://localhost:8000/api/projects/1/figma-json | jq '.'
```

### Step 2: Find a Specific Node

Pick a node to verify. For example, find the "Header" frame:

```bash
cat generated_projects/my-app/figma_data/design_data.json | \
  jq '.pages[0].frames[] | select(.name == "Header")'
```

Note the key properties:
- `id` — the Figma node ID (e.g., `"123:456"`)
- `layout.bounds` — width and height
- `layout.padding` — padding values
- `layout.mode` — `"HORIZONTAL"` or `"VERTICAL"` (maps to flex-direction)
- `layout.itemSpacing` — gap between children
- `fills` — background colors
- `style` — font size, weight, family (for text nodes)
- `cornerRadius` — border radius
- `effects` — shadows, blurs

### Step 3: Locate the Matching Component

Open the generated component:

```bash
cat generated_projects/my-app/src/components/Header.tsx
```

### Step 4: Compare Properties

Check each property from the design data against the generated code:

| Design Data Property | What to Look For in Code |
|---|---|
| `layout.bounds.width: 1440` | Container width (e.g., `max-w-screen-xl` or `w-full`) |
| `layout.bounds.height: 80` | Height (e.g., `h-20` = 80px) |
| `layout.padding.top: 16` | `pt-4` (16px) |
| `layout.padding.left: 24` | `pl-6` (24px) |
| `layout.mode: "HORIZONTAL"` | `flex flex-row` |
| `layout.itemSpacing: 12` | `gap-3` (12px) |
| `fills[0].color: {r:0.1, g:0.1, b:0.2}` | `bg-[#1A1A33]` (convert RGBA 0-1 to hex) |
| `cornerRadius.all: 8` | `rounded-lg` (8px) |
| `style.fontSize: 16` | `text-base` (16px) or `text-[16px]` |
| `style.fontWeight: 600` | `font-semibold` (600) |

### Step 5: Check data-figma-id Attributes

If structural comparison is enabled, verify that the generated component includes `data-figma-id` attributes:

```tsx
<header data-figma-id="123:456" className="flex flex-row gap-3 ...">
```

This attribute links the DOM element back to its Figma node, enabling the structural comparison tier.

### Step 6: Visual Comparison

Run the project and compare visually:

```bash
cd generated_projects/my-app
npm run dev
```

Open `http://localhost:5173` (or the assigned port) and compare side-by-side with the Figma design.

---

## 5. Reading Verification Reports

### Where Reports Are Stored

Verification results are returned as part of the conversion result and can be accessed through the project API. The verification loop returns:

```json
{
  "status": "success",
  "iterations": 3,
  "confidence": 0.96,
  "final_scores": {
    "layout": 0.92,
    "spacing": 0.88,
    "colors": 0.99,
    "typography": 0.95,
    "effects": 0.90,
    "structural": 0.94
  },
  "history": [
    {
      "iteration": 1,
      "confidence": 0.82,
      "method": "vision+structural+content",
      "accuracy_scores": {"layout": 0.78, "spacing": 0.72, "colors": 0.95},
      "discrepancies": [...],
      "fixes_applied": 5
    },
    {
      "iteration": 2,
      "confidence": 0.91,
      "method": "vision+structural+content",
      "discrepancies": [...],
      "fixes_applied": 3
    },
    {
      "iteration": 3,
      "confidence": 0.96,
      "method": "vision+structural+content",
      "discrepancies": [],
      "fixes_applied": 0
    }
  ]
}
```

### Understanding Confidence Scores

| Score Range | Meaning |
|---|---|
| **0.98 - 1.00** | Near-perfect match. Early stop triggered. |
| **0.95 - 0.98** | Passes threshold. Minor visual differences that are acceptable. |
| **0.85 - 0.95** | Good but not perfect. Some spacing, color, or layout issues remain. Returned as `"completed_with_warnings"`. |
| **Below 0.85** | Significant differences. Returned as `"needs_review"`. Manual inspection recommended. |

### Understanding the `method` Field

| Method | Meaning |
|---|---|
| `vision+structural+content` | All three tiers ran. Weights: 50/30/20. Most accurate. |
| `vision+content` | No `data-figma-id` elements found. Weights: 70/30. |
| `structural+content` | No Figma screenshot available. Weights: 60/40. |
| `content_only` | Only text/color/font matching. Least precise. |

### Understanding Accuracy Scores

| Score | What It Measures |
|---|---|
| `layout` | Element positioning, container sizes, alignment (from vision) |
| `spacing` | Padding, margins, gaps (from vision + structural) |
| `colors` | Background/text/border color accuracy (from vision + structural) |
| `typography` | Font size, weight, line height matching (from vision + structural) |
| `effects` | Shadows, gradients, border radius (from vision) |
| `structural` | Overall structural comparison confidence (from Tier 2) |
| `dimension_accuracy` | Width/height accuracy (from structural, high-severity checks) |
| `color_accuracy` | Color accuracy (from structural, medium-severity checks) |
| `spacing_accuracy` | Padding/gap accuracy (from structural, low-severity checks) |

### Interpreting Discrepancies

Each discrepancy tells you exactly what is wrong and how to fix it:

```json
{
  "type": "spacing",
  "severity": "high",
  "location": "Header section",
  "expected": "padding-left: 24.0px",
  "actual": "padding-left: 16.0px"
}
```

- **`severity: "high"`** — Dimensions, font size, font weight, flex direction. These are the most visible issues.
- **`severity: "medium"`** — Background and text colors. Noticeable but less impactful.
- **`severity: "low"`** — Padding, gap, border radius, box shadow. Subtle differences.

---

## 6. Re-Running Verification

### Option 1: Re-run the Full Conversion

Start a new conversion via the API or frontend. The verification loop runs automatically after the agent finishes and the build succeeds.

### Option 2: Programmatic Verification

You can call the verification loop directly from Python:

```python
from pathlib import Path
from backend.agents._figma_to_react.verification import visual_verification_loop
import json
import asyncio

project_path = Path("generated_projects/my-app")

# Load design data
with open(project_path / "figma_data" / "design_data.json") as f:
    design_data = json.load(f)

# Run verification
result = asyncio.run(visual_verification_loop(
    project_path=project_path,
    design_data=design_data,
    max_iterations=5,
))

print(f"Status: {result['status']}")
print(f"Confidence: {result['confidence']:.2%}")
print(f"Iterations: {result['iterations']}")
```

### Option 3: Manual Structural Check

Run just the structural comparison:

```python
from backend.utils.structural_comparison import compare_structural_properties
import asyncio

# Assumes dev server is already running on port 5173
result = asyncio.run(compare_structural_properties(
    port=5173,
    design_data=design_data,
    project_path=project_path,
))

print(f"Confidence: {result['confidence']:.2%}")
print(f"Passed: {result['passed_checks']}/{result['total_checks']}")
for check in result['property_checks']:
    if not check['match']:
        print(f"  FAIL: {check['element']}.{check['property']}: "
              f"expected {check['figma_value']}, got {check['dom_value']}")
```

---

## 7. Component Reuse and Library Stats

### How the Component Library Works

1. **During conversion**, the agent can search the ChromaDB library for existing components that match the current design:
   - `search_components` — semantic search by description or category.
   - `get_component` — retrieve a specific component by ID.

2. **After generating** a new component, the agent saves it to the library:
   - `save_component` — stores the component code, description, category, and optional Figma metadata.

3. **On subsequent conversions**, the agent finds and reuses components from the library, reducing generation time and improving consistency.

### Where to See Reuse Stats

#### Per-Project Stats

```
GET /api/projects/{project_id}
```

Response includes:
```json
{
  "components_generated": 8,
  "components_reused": 3,
  "conversion_time_seconds": 45.2
}
```

- `components_generated` — number of new `.tsx` files created in `src/components/`.
- `components_reused` — number of components retrieved from the library or reused from a parent project.

#### Platform-Wide Stats

```
GET /api/stats
```

Response includes:
```json
{
  "total_projects": 5,
  "completed_projects": 4,
  "total_components": 42,
  "total_component_reuses": 18
}
```

#### Library Contents

```
GET /api/components
GET /api/components?category=navigation
```

Returns all components in the ChromaDB library. Each entry includes name, description, category, and usage count.

### ChromaDB Storage Location

The component library is persisted at:
```
./component_library/chroma/
```

This path is configurable via `settings.component_library_dir` in `backend/config.py`.

### How Reuse Is Detected

The system detects reuse through multiple methods:

1. **Explicit `get_component` calls:** The agent retrieves a library component — counted directly.
2. **Import scanning (multi-page mode):** When adding a page to an existing project, the system scans the new page's imports for references to components that existed before the conversion.
3. **Library name matching:** For standalone projects, component file names are compared against the library's existing entries.

---

## Quick Reference

| What You Want | Where to Look |
|---|---|
| Original Figma design data | `{project}/figma_data/design_data.json` |
| Raw Figma API/plugin payload | `{project}/figma_data/raw_figma_response.json` |
| Generated components | `{project}/src/components/*.tsx` |
| Generated pages | `{project}/src/pages/*.tsx` |
| Downloaded images | `{project}/public/images/` |
| Design screenshot | `{project}/screenshots/design_screenshot.png` |
| Verification screenshots | `{project}/screenshots/` |
| Component library | `./component_library/chroma/` |
| All settings | `backend/config.py` |
| API: design data | `GET /api/projects/{id}/figma-json` |
| API: raw Figma data | `GET /api/projects/{id}/figma-json/raw` |
| API: project details | `GET /api/projects/{id}` |
| API: platform stats | `GET /api/stats` |
| API: library components | `GET /api/components` |
