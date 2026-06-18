# Validation Pipeline: Three-Tier Verification & Auto-Fix Loop

> **Project:** Aura2 — AI-Powered Figma → React Converter
> **Samsung PRISM @ IIIT Naya Raipur**
> This document covers the complete validation system: vision comparison, structural comparison, content comparison, confidence scoring, the auto-fix loop, and all related configuration.
>
> 📘 **New reader?** Start with [**VERIFICATION_GUIDE.md**](./VERIFICATION_GUIDE.md) — a plain-language
> introduction written for non-UI readers. This file is the deep technical reference.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Tier 1: Vision Comparison (Claude Vision API)](#2-tier-1-vision-comparison-claude-vision-api)
3. [Tier 2: Structural Comparison (Figma JSON vs DOM)](#3-tier-2-structural-comparison-figma-json-vs-dom)
4. [Tier 3: Content Comparison (Fallback)](#4-tier-3-content-comparison-fallback)
5. [Confidence Scoring](#5-confidence-scoring)
6. [Auto-Fix Loop](#6-auto-fix-loop)
7. [Verification Configuration](#7-verification-configuration)
8. [Figma JSON Persistence](#8-figma-json-persistence)

---

## 1. Overview

After the Claude agent generates React components and the build succeeds, the **visual verification loop** runs to ensure the generated output matches the original Figma design. This loop is implemented in `backend/agents/_figma_to_react/verification.py`.

The loop:
1. Starts a dev server for the generated project.
2. Captures a screenshot of the rendered page via Playwright.
3. Runs up to three comparison tiers (vision, structural, content).
4. Combines scores into a **confidence** value.
5. If confidence is below the threshold, generates and applies fixes, then re-verifies.
6. Repeats for up to `max_verification_iterations` (default: 10) or until confidence is met.

```
┌─────────────────────────────────────────────────────────┐
│              Visual Verification Loop                     │
│                                                           │
│   Start dev server                                        │
│        │                                                  │
│        ▼                                                  │
│   ┌─────────────────────┐                                │
│   │ Capture Screenshot   │ ◄──── Playwright               │
│   └──────────┬──────────┘                                │
│              │                                            │
│   ┌──────────▼──────────────────────────────────────┐    │
│   │  Run Comparison Tiers                            │    │
│   │                                                  │    │
│   │  Tier 1: Vision (Claude Vision API)              │    │
│   │       Figma screenshot vs Generated screenshot   │    │
│   │                                                  │    │
│   │  Tier 2: Structural (Figma JSON vs DOM)          │    │
│   │       data-figma-id elements → computed styles   │    │
│   │                                                  │    │
│   │  Tier 3: Content (text, color, font matching)    │    │
│   │       Always runs as fallback/supplement         │    │
│   └──────────┬──────────────────────────────────────┘    │
│              │                                            │
│              ▼                                            │
│   Combined Confidence Score                               │
│              │                                            │
│        ┌─────┴─────┐                                     │
│        │  >= 95%?  │──── YES ──► Return "success"        │
│        └─────┬─────┘                                     │
│              │ NO                                         │
│              ▼                                            │
│   Generate Fixes → Apply Fixes → Loop back               │
│   (up to 5 per iteration, up to 10 iterations)           │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Tier 1: Vision Comparison (Claude Vision API)

**Source:** `backend/utils/vision_comparison.py`

The vision tier sends **both images** (Figma design screenshot and generated website screenshot) to the Claude Vision API and asks it to perform a pixel-perfect comparison.

### How It Works

1. The Figma design screenshot comes from one of two sources:
   - **Plugin path:** The Figma Plugin captures a screenshot of the selected frame and sends it as base64.
   - **API path:** Exported via `export_design_screenshot()` if available.

2. The generated website screenshot is captured by Playwright from the running dev server.

3. Both images are base64-encoded and sent to the Claude Vision API:

```python
client = anthropic.Anthropic(
    api_key=settings.litellm_api_key,
    base_url=settings.litellm_base_url or None,
)

message = client.messages.create(
    model=settings.vision_comparison_model,  # "claude-sonnet-4-6"
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", ...}},  # Figma screenshot
            {"type": "image", "source": {"type": "base64", ...}},  # Generated screenshot
            {"type": "text", "text": comparison_prompt},
        ],
    }],
)
```

### What the Vision API Returns

The model returns a JSON response with:

```json
{
  "matches": false,
  "confidence": 0.85,
  "overall_assessment": "Close match but spacing issues in header",
  "discrepancies": [
    {
      "type": "spacing",
      "severity": "high",
      "location": "Header section",
      "expected": "24px padding",
      "actual": "16px padding",
      "coordinates": {"x": 100, "y": 50, "width": 200, "height": 80},
      "fix_instructions": {
        "target_file": "src/components/Header.tsx",
        "target_element": "header container",
        "current_value": "p-4",
        "new_value": "px-6 py-5",
        "explanation": "Change padding to match design exactly"
      }
    }
  ],
  "accuracy_scores": {
    "layout": 0.85,
    "spacing": 0.75,
    "colors": 0.98,
    "typography": 0.92,
    "effects": 0.88
  }
}
```

### Analysis Categories

The vision model evaluates:

| Category | What It Checks |
|---|---|
| **Layout** | Element positioning, container sizes, alignment |
| **Spacing** | Padding, margins, gaps between elements |
| **Colors** | Background colors, text colors, border colors (exact hex) |
| **Typography** | Font sizes, weights, line heights, letter spacing |
| **Effects** | Shadows, border radius, gradients, opacity |

### When Vision Is Not Available

If no Figma screenshot exists (e.g., the REST API path without export, or plugin didn't capture one), the vision tier is skipped and the system falls back to structural + content comparison with adjusted weights.

---

## 3. Tier 2: Structural Comparison (Figma JSON vs DOM)

**Source:** `backend/utils/structural_comparison.py`

This is the newest comparison tier. It performs a **property-by-property** comparison between the original Figma node properties and the actual DOM computed styles, matched by `data-figma-id` attributes.

### How It Works

#### Step 1: Extract Figma Properties

The `_extract_figma_properties()` function walks the design data tree and extracts properties for each node by its ID:

- **Dimensions:** width, height (from `layout.bounds`)
- **Padding:** top, right, bottom, left (from `layout.padding`)
- **Gap:** item spacing (from `layout.itemSpacing`)
- **Flex direction:** row or column (from `layout.mode`)
- **Background color:** from solid fills (RGBA → RGB)
- **Text color:** from fills on TEXT nodes
- **Font size, weight, family:** from `style` properties
- **Border radius:** from `cornerRadius`
- **Box shadow:** presence from effects

#### Step 2: Extract DOM Computed Styles

A Node.js script is executed via subprocess that uses Playwright to:
1. Launch a headless Chromium browser.
2. Navigate to `http://localhost:{port}`.
3. Query all elements with `[data-figma-id]` attributes.
4. For each element, read `window.getComputedStyle()` values:

```javascript
const elements = document.querySelectorAll('[data-figma-id]');
elements.forEach(el => {
    const figmaId = el.getAttribute('data-figma-id');
    const cs = window.getComputedStyle(el);
    const rect = el.getBoundingClientRect();
    results[figmaId] = {
        width: rect.width, height: rect.height,
        paddingTop, paddingRight, paddingBottom, paddingLeft,
        backgroundColor, color, fontSize, fontWeight, fontFamily,
        borderRadius, boxShadow, gap, display, flexDirection,
        alignItems, justifyContent,
    };
});
```

#### Step 3: Compare Properties

Each Figma node ID is matched to its corresponding DOM element. Properties are compared with configurable tolerances:

| Property | Severity | Tolerance |
|---|---|---|
| Width, height | High | ±2px (`structural_comparison_tolerance_px`) |
| Font size | High | ±1px |
| Font weight | High | Exact match |
| Flex direction | High | Exact match |
| Background color | Medium | ±5 per RGB channel (`color_comparison_tolerance`) |
| Text color | Medium | ±5 per RGB channel |
| Padding (all sides) | Low | ±2px |
| Gap | Low | ±2px |
| Border radius | Low | ±2px |
| Box shadow | Low | Presence check |

#### Step 4: Aggregate Scores

```python
total = len(property_checks)
passed = sum(1 for c in property_checks if c["match"])
confidence = passed / total

# Category-specific accuracy:
dimension_accuracy = passed_high / total_high     # high severity checks
color_accuracy     = passed_medium / total_medium  # medium severity checks
spacing_accuracy   = passed_low / total_low        # low severity checks
```

### The `data-figma-id` Requirement

For structural comparison to work, the agent must add `data-figma-id` attributes to generated React elements that correspond to Figma node IDs. The conversion prompt instructs the agent to do this. If no `data-figma-id` elements are found in the DOM, the structural tier is skipped (treated as unavailable).

---

## 4. Tier 3: Content Comparison (Fallback)

**Source:** `backend/utils/visual_comparison.py` — `compare_with_figma_design()`

This tier always runs as a fallback/supplement. It compares:
- **Text content** — checks whether text strings from the Figma design appear in the generated output.
- **Colors** — verifies that the design's color palette is present in the generated CSS.
- **Fonts** — checks whether specified font families are loaded.

This tier is the least precise but requires no screenshots and no `data-figma-id` mapping. It provides a baseline confidence even when the other tiers are unavailable.

---

## 5. Confidence Scoring

### Adaptive Weighting

The weights change based on which tiers are available:

| Available Tiers | Formula |
|---|---|
| Vision + Structural + Content | **50% vision + 30% structural + 20% content** |
| Vision + Content (no structural) | 70% vision + 30% content |
| Structural + Content (no vision) | 60% structural + 40% content |
| Content only | 100% content |

### Thresholds

| Threshold | Value | Action |
|---|---|---|
| `verification_confidence_threshold` | **0.95 (95%)** | Minimum to pass — returns `"success"` |
| `verification_early_stop_threshold` | **0.98 (98%)** | Early stop — returns `"success"` immediately |
| 0.85 | 85% | Below pass but above 85% → `"completed_with_warnings"` |
| Below 0.85 | < 85% | `"needs_review"` |

### Plateau Detection

If confidence improves by less than 1% between consecutive iterations, the loop stops early to avoid wasting compute:

```python
if improvement < 0.01:  # Less than 1% improvement
    # Plateau detected — stop
```

### Example Scoring

```
Vision confidence:     0.88 (layout good, spacing issues)
Structural confidence: 0.92 (most properties match)
Content confidence:    0.95 (all text and colors present)

Combined = 0.88 * 0.50 + 0.92 * 0.30 + 0.95 * 0.20
         = 0.44 + 0.276 + 0.19
         = 0.906 (90.6%)

Result: Below 95% threshold → generate fixes and loop
```

---

## 6. Auto-Fix Loop

### Fix Generation

When confidence is below the threshold, the `generate_fixes()` function (from `backend/utils/auto_fix_agent.py`) converts discrepancies into actionable fix instructions.

Fix sources:
- **Vision tier:** Provides `fix_instructions` with target file, current value, and new Tailwind class.
- **Structural tier:** High-severity mismatches (e.g., wrong width, wrong flex-direction) become fix entries with exact expected vs actual values.

### Fix Application

Fixes are applied by a **dedicated Claude agent session** (`apply_fixes()` in `backend/agents/_figma_to_react/verification.py`):

```python
options = ClaudeAgentOptions(
    model=settings.default_model,
    system_prompt=f"You are a pixel-perfect code fixer. Apply the specific fixes...",
    max_turns=settings.max_fix_turns,  # 15
    cwd=str(project_path),
    allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
    permission_mode="acceptEdits",
)
```

Key constraints:
- **Max fixes per iteration:** 5 (`max_fixes_per_iteration`)
- **Fix agent max turns:** 15 (`max_fix_turns`)
- The fix agent receives the formatted fixes and applies them precisely.

### Fix Loop Flow

```
Iteration 1:
  Capture screenshot → Compare → Confidence 85%
  Generate 8 fixes → Apply top 5 → Wait for hot reload

Iteration 2:
  Capture screenshot → Compare → Confidence 91%
  Generate 3 fixes → Apply 3 → Wait for hot reload

Iteration 3:
  Capture screenshot → Compare → Confidence 96%
  Above 95% threshold → Return "success"
```

### Discrepancy Structure

Each discrepancy has a consistent format regardless of which tier produced it:

```json
{
  "type": "spacing|color|layout|shadow|typography|structural|missing_element",
  "severity": "high|medium|low",
  "location": "Header section",
  "expected": "24px padding",
  "actual": "16px padding"
}
```

Structural discrepancies appended from Tier 2:

```json
{
  "type": "structural",
  "severity": "high",
  "location": "NavBar",
  "expected": "flex-direction: row",
  "actual": "flex-direction: (display: block)"
}
```

---

## 7. Verification Configuration

All settings from `backend/config.py`:

### Vision Settings

| Setting | Default | Description |
|---|---|---|
| `enable_vision_comparison` | `True` | Master switch for Vision API comparison |
| `vision_comparison_model` | `"claude-sonnet-4-6"` | Model with vision capabilities |
| `max_verification_iterations` | `10` | Max iterations for the entire loop |
| `verification_confidence_threshold` | `0.95` | Minimum confidence to pass |
| `verification_early_stop_threshold` | `0.98` | Confidence for early stop (skip remaining iterations) |

### Screenshot Settings

| Setting | Default | Description |
|---|---|---|
| `screenshot_scale` | `1` | Scale factor (1x reduces base64 token cost) |
| `screenshot_viewport_width` | `1440` | Viewport width in pixels |
| `screenshot_viewport_height` | `900` | Viewport height in pixels |

### Structural Comparison Settings

| Setting | Default | Description |
|---|---|---|
| `enable_structural_comparison` | `True` | Master switch for structural comparison |
| `structural_comparison_tolerance_px` | `2` | Pixel tolerance for dimension/spacing checks |
| `color_comparison_tolerance` | `5` | Per-channel RGB tolerance (0-255) |

### Fix Application Settings

| Setting | Default | Description |
|---|---|---|
| `max_fixes_per_iteration` | `5` | Maximum fixes applied in a single iteration |
| `auto_apply_high_priority_fixes` | `True` | Automatically apply high-severity fixes |
| `require_manual_review_for_low_confidence` | `True` | Flag for manual review when confidence stays low |

---

## 8. Figma JSON Persistence

**Source:** `backend/agents/_figma_to_react/figma_json_persistence.py`

Every conversion persists the Figma data to `{project}/figma_data/` for debugging, auditing, and API access.

### Files Saved

| File | Contents |
|---|---|
| `figma_data/raw_figma_response.json` | Raw Figma REST API response or raw plugin payload |
| `figma_data/design_data.json` | Normalized/extracted design data (pages, frames, colors, fonts, images) |
| `figma_data/design_metadata.json` | Metadata: timestamp, source ("api" or "plugin"), file key, stats |

### save_figma_json() Function

```python
def save_figma_json(
    project_path: Path,
    raw_data: dict,       # Raw API response or plugin payload
    design_data: dict,    # Processed design data
    source: str,          # "api" or "plugin"
    file_key: str = "",   # Figma file key (API path only)
) -> Path:
```

This function is called in both input paths:
- **REST API path:** `save_figma_json(project_path, figma_data, design_data, source="api", file_key=file_key)`
- **Plugin path:** `save_figma_json(project_path, plugin_data, design_data, source="plugin")`

### API Endpoints for Accessing Figma JSON

| Endpoint | Description |
|---|---|
| `GET /api/projects/{id}/figma-json` | Returns the processed `design_data.json` |
| `GET /api/projects/{id}/figma-json/raw` | Returns the raw `raw_figma_response.json` |

---

## Key Source Files

| File | Purpose |
|---|---|
| `backend/agents/_figma_to_react/verification.py` | Main verification loop and fix application |
| `backend/utils/vision_comparison.py` | Claude Vision API comparison |
| `backend/utils/structural_comparison.py` | Figma JSON vs DOM computed styles |
| `backend/utils/visual_comparison.py` | Content-based comparison and screenshot capture |
| `backend/utils/auto_fix_agent.py` | Fix generation from discrepancies |
| `backend/utils/figma_screenshot.py` | Design screenshot export/save |
| `backend/agents/_figma_to_react/figma_json_persistence.py` | Figma data persistence |
| `backend/config.py` | All verification settings |
