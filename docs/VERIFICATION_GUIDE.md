# Verification Guide — How Aura2 Checks Figma-to-Code Accuracy

> **Written for anyone** — backend engineers, PMs, mentors, paper reviewers,
> first-time readers. No prior UI/design knowledge assumed.
>
> If you only read one section, read **[§1 The Problem](#1-the-problem)** and
> **[§7 Reading the Report](#7-reading-the-report)**.

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [The Solution in 60 Seconds](#2-the-solution-in-60-seconds)
3. [What Gets Checked](#3-what-gets-checked)
4. [ΔE2000 — Why Not Raw RGB Diff](#4-e2000--why-not-raw-rgb-diff)
5. [Element-Level Screenshots — How Figma IDs Connect to the Browser](#5-element-level-screenshots--how-figma-ids-connect-to-the-browser)
6. [The Self-Correcting Loop](#6-the-self-correcting-loop)
7. [Reading the Report](#7-reading-the-report)
8. [Tuning Parameters](#8-tuning-parameters)
9. [Limitations — What We Do Not Check](#9-limitations--what-we-do-not-check)
10. [Glossary](#10-glossary)

---

## 1. The Problem

A designer draws a webpage in Figma. An AI system generates React code from it.
**How do we know the generated code looks like the design?**

This sounds trivial. It is not. Here is a concrete example taken from a real
conversion run:

> In Figma, the text **"Mobile"** appears in a navigation bar.
> The design spec says: font size **16 px**, color **#000000** (black),
> width **42 px**.
>
> The generated React code produces a navigation bar. When rendered in Chrome,
> the **"Mobile"** text measures: font size **16 px**, color
> **rgb(0, 0, 0)**, width **52.6 px**.
>
> **Is this a bug?**

The font size matches. The color matches. But the width is off by 10 pixels.

**Answer:** no, it is not a bug. Figma and Chrome use slightly different text
layout engines. Font glyph widths, letter-spacing, and rounding rules differ.
A 10-pixel variance on rendered text is normal and unavoidable.

But a naive checker that compares "width in Figma" to "width in browser" would
flag this as a failure — every time, on every text element. It would then
ask the AI to "fix" the code, the AI would tweak something at random, the new
width would differ again, and the loop would never terminate.

**The problem has three parts:**

1. **What to check:** not every property is meaningful. Text widths vary.
   Box widths don't. Background color is exact. Rendered color is not.
2. **How to compare:** pixel-level RGB subtraction is too strict. Human
   perception is tolerant of small color shifts — the comparison must be too.
3. **When to stop:** if the AI keeps making "fixes" that don't improve the
   score, the loop has plateaued. Keep going and you spend API credits for
   nothing.

Aura2 solves all three.

---

## 2. The Solution in 60 Seconds

Aura2 runs three independent checks in parallel, then combines them. If the
combined confidence is too low, it sends the discrepancies back to the fix
agent and re-verifies. It stops when confidence is high enough or when the
scores stop improving (a plateau).

```
         ┌─────────────────────┐
         │   Figma Design JSON │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │   Code Generator    │  (Claude agent)
         │     (React + CSS)   │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │   Build + Dev Server │  (Vite + Playwright)
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────────────────────────┐
         │        VERIFICATION (3 TIERS)           │
         │                                         │
         │  Tier 1: Vision      — image vs image   │
         │  Tier 2: Structural  — JSON vs DOM      │
         │  Tier 3: Element     — Figma node vs    │
         │                         rendered box    │
         └──────────┬──────────────────────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │  Confidence Score   │
         │  + Discrepancies    │
         └──────────┬──────────┘
                    │
           confidence high?
          ┌─────────┴─────────┐
         YES                 NO
          │                   │
          ▼                   ▼
      ✅ Done         ┌──────────────┐
                      │   Fix Agent  │
                      │ (Claude)     │
                      └──────┬───────┘
                             │
                             └──── back to Build + Verify
```

**Tier 1 — Vision.** Take a PNG of the design, a PNG of the rendered page,
ask a vision model "do these look the same?" It catches visual drift that
property checks miss.

**Tier 2 — Structural.** Parse the Figma JSON to get the *intended* CSS
properties (color, padding, font-size, box-shadow, etc.). Inspect the
rendered DOM using Playwright to get the *actual* computed styles. Compare
field by field, with perceptual tolerance where appropriate.

**Tier 3 — Element-level.** Every Figma node has an ID. We tag every React
element with `data-figma-id="<that id>"`. After render, we screenshot each
tagged element, export the same node from Figma, and compare them pixel by
pixel (ΔE2000). This gives per-component accuracy, not just per-page.

Scores are weighted (50% structural + 25% vision + 25% content by default)
and the combined number is the **overall confidence**.

---

## 3. What Gets Checked

Every property Aura2 verifies maps back to one source in Figma and one source
in the rendered DOM. This table is the ground truth for what the system can
and cannot catch.

| Property           | Figma Source                   | DOM Source (computed style)   | Comparison            | Tolerance               |
|--------------------|--------------------------------|-------------------------------|-----------------------|-------------------------|
| Box width          | `layout.bounds.width`          | `getBoundingClientRect().width` | Numeric diff         | ±4 px                   |
| Box height         | `layout.bounds.height`         | `getBoundingClientRect().height` | Numeric diff        | ±4 px                   |
| Background color   | `fills[].color` (SOLID)        | `background-color`            | ΔE2000                | ≤ 3.0                   |
| Text color         | `fills[].color` on TEXT node   | `color`                       | ΔE2000                | ≤ 3.0                   |
| Border color       | `strokes[].color`              | `border-color`                | ΔE2000                | ≤ 3.0                   |
| Padding (t/r/b/l)  | `layout.padding.*`             | `padding-top/right/bottom/left` | Numeric diff         | ±2 px                   |
| Gap                | `layout.itemSpacing`           | `gap`                         | Numeric diff          | ±2 px                   |
| Flex direction     | `layout.mode` (VERTICAL/HORIZONTAL) | `flex-direction`        | Exact string match    | —                       |
| Font size          | `style.fontSize`               | `font-size`                   | Numeric diff          | ±1 px                   |
| Font weight        | `style.fontWeight`             | `font-weight`                 | Exact numeric         | —                       |
| Font family        | `style.fontFamily`             | `font-family` (first token)   | Exact string          | —                       |
| Border radius      | `cornerRadius`                 | `border-radius`               | Numeric diff          | ±2 px                   |
| Box shadow         | `effects[] DROP_SHADOW`        | `box-shadow`                  | Parsed + numeric       | ±2 px per offset/blur  |
| Opacity            | `opacity`                      | `opacity`                     | Numeric diff          | ±0.05                   |
| Pixel match (per element) | Figma REST API PNG export | Playwright element screenshot | ΔE2000 per pixel      | ≤ 3.0 per pixel         |

**Properties deliberately NOT checked** (see [§9](#9-limitations--what-we-do-not-check)):
text widths, responsive breakpoints, hover/focus states, cross-browser
rendering, animation timing.

---

## 4. ΔE2000 — Why Not Raw RGB Diff

**TL;DR:** raw RGB comparison punishes imperceptible color differences and
ignores ones humans actually see. ΔE2000 fixes this.

### The problem with RGB diff

A naive color check might do:

```python
def colors_match(a, b):
    return abs(a.r - b.r) <= 10 and abs(a.g - b.g) <= 10 and abs(a.b - b.b) <= 10
```

This fails in two directions:

**Too lenient for reds:**
`rgb(255, 0, 0)` vs `rgb(245, 10, 10)` → "match" (all channels within 10)
but these two reds look visibly different side-by-side.

**Too strict for blues:**
`rgb(20, 40, 200)` vs `rgb(20, 45, 215)` → "no match" (blue channel off by 15)
but they are indistinguishable to almost every human observer.

The root cause: RGB is a device-oriented color space. It does not model how
humans *perceive* color. The eye is very sensitive to small changes in blue-
violet hues but much less sensitive to equivalent numerical changes in green.

### What ΔE2000 does

ΔE2000 (CIEDE2000, Sharma et al. 2005) is a formula that takes two colors,
converts them to CIE Lab color space (a perceptually uniform space), then
computes a scalar distance that correlates with human-perceived difference.

- **ΔE < 1.0** → colors are literally indistinguishable to a trained observer.
- **ΔE 1.0 – 2.3** → noticeable only under ideal side-by-side conditions.
- **ΔE 2.3 – 5.0** → visible difference but similar colors.
- **ΔE > 5.0** → clearly different colors.

Aura2 uses a threshold of **ΔE ≤ 3.0** for color and per-pixel comparison —
inside the "indistinguishable to untrained observer" band.

### Worked example

| Comparison                     | ΔE00 | RGB channel diff | Verdict         |
|--------------------------------|------|------------------|-----------------|
| `#1a73e8` vs `#1b75e9`         | 0.34 | (1, 2, 1)        | ✅ match        |
| `#1a73e8` vs `#2088ff`         | 4.12 | (6, 21, 23)      | ❌ no match     |
| `#ff0000` vs `#f50a0a`         | 3.78 | (10, 10, 10)     | ❌ no match (RGB would say ✓) |
| `#202080` vs `#203098`         | 2.18 | (0, 16, 24)      | ✅ match (RGB would say ✗)   |

The last two rows are the key insight: ΔE2000 is *stricter* on reds where the
eye is sensitive, and *more lenient* on dark blues where the eye is not.

Implementation: `backend/utils/structural_comparison.py:_color_delta_e`.

---

## 5. Element-Level Screenshots — How Figma IDs Connect to the Browser

Figma and the browser don't speak the same language. Figma has **nodes**
with IDs like `34:892`. The browser has **DOM elements** with CSS selectors.
To compare "this Figma node" with "this rendered box" we need a bridge.

### The `data-figma-id` bridge

During code generation, every React component that corresponds to a Figma
node is tagged with a data attribute:

```tsx
// Generated React code
<div data-figma-id="34:892" className="nav-bar">
  <span data-figma-id="34:893">Mobile</span>
</div>
```

After the page renders in the dev server, Playwright runs
`document.querySelectorAll("[data-figma-id]")`. For each tagged element it:

1. Reads the `data-figma-id` attribute → knows the Figma node ID.
2. Screenshots the element's bounding box.
3. Saves the PNG to `screenshots/dom_elements/<figma_id>.png`.

In parallel, the Figma REST API exports the same node IDs as PNGs via
`images/<file_key>?ids=34:892,34:893,…&format=png`. These go to
`screenshots/figma_elements/<figma_id>.png`.

Now we have pairs. For each pair we run dimension check (width/height match)
and pixel comparison (ΔE2000 per pixel).

### What gets a pixel diff

Not every Figma node is a good candidate for pixel comparison. We filter to:

- **Node types:** `FRAME`, `GROUP`, `COMPONENT`, `INSTANCE`, `RECTANGLE`.
  Skip `TEXT` — text widths vary across renderers even when the content
  is identical (font fallback, subpixel antialiasing).
- **Minimum size:** at least 40 × 40 px. Below that, icons and badges
  produce noisy pixel diffs that don't carry real signal.

This filter lives in `_collect_node_ids()` in
`backend/utils/element_screenshot_comparison.py`.

Implementation: `backend/utils/element_screenshot_comparison.py:compare_element_pixels`.

---

## 6. The Self-Correcting Loop

Verification by itself doesn't fix anything. It produces a confidence score
and a list of discrepancies. The **fix agent** consumes those discrepancies
and edits the code. Then we re-verify.

### Loop structure

```
ITERATION 1
   build code ──► run dev server ──► verify
                                        │
                                        ▼
                          confidence=0.78, 12 discrepancies
                                        │
                                        ▼
                          fix agent edits code (apply top discrepancies)
                                        │
                                        ▼
ITERATION 2
   rebuild ──► re-verify
                                        │
                                        ▼
                          confidence=0.89, 4 discrepancies
                                        │
                                        ▼
                          fix agent edits code
                                        │
                                        ▼
ITERATION 3
   rebuild ──► re-verify
                                        │
                                        ▼
                          confidence=0.95 ──► ✅ DONE
```

### When it stops

Three exit conditions:

1. **Success** — confidence ≥ `confidence_threshold` (default 0.90).
2. **Max iterations** — reached `max_verification_iterations`
   (default 10). Emits a `completed_with_warnings` status.
3. **Plateau** — confidence didn't improve by at least 1% across two
   consecutive iterations. Prevents infinite loops when the fix agent
   is just reshuffling code without improving scores.

### Why this works

Individual discrepancies are concrete. "Button at `node 34:912` has
background `#2563eb` in Figma but `#3b82f6` in DOM" is something the fix
agent can act on. It finds the button in the code, changes the color,
rebuilds. The scoring reveals whether the change actually helped.

The key insight: **each iteration is cheap compared to a fresh generation**.
We keep the existing code and the existing build cache; we only patch the
affected files.

Implementation: `backend/agents/_figma_to_react/verification.py` +
`backend/agents/_figma_to_react/fix_agent.py`.

---

## 7. Reading the Report

After verification completes, the dashboard shows five panels. Here is what
each one means.

### Panel 1 — Overall Confidence

```
┌─────────────────────────┐
│  OVERALL CONFIDENCE     │
│                         │
│        95 %             │
│                         │
│  [SUCCESS]  structural+vision
└─────────────────────────┘
```

The headline number. Weighted average of all checks.

Status values:
- `success` — confidence ≥ 0.90.
- `completed_with_warnings` — hit max iterations before threshold.
- `needs_review` — below 0.75, human should look.
- `failed` — verification crashed or the build broke.

### Panel 2 — Quick Stats

- **Iterations:** how many fix cycles ran.
- **Elements:** how many `data-figma-id` nodes were compared.
- **Discrepancies:** unresolved mismatches at the end.
- **Dim Accuracy:** fraction of elements whose width AND height matched.

### Panel 3 — Category Scores

Per-property-category breakdown:

```
Colors          ████████████████████████████ 98 %
Spacing         ████████████████████████████ 100 %
Typography      ████████████████████████████ 100 %
Effects         ████████████████████████████ 100 %
Dimensions      █████████████████░░░░░░░░░░░  74 %
Pixel Match     █████████████████████░░░░░░░  91 %
```

Read this as a diagnostic: if Dimensions is low but Colors is high, the
styling is right but the layout containers are sized wrong — usually a
flex/grid or explicit width issue.

### Panel 4 — Per-Element Grid

Each card shows:
- A thumbnail screenshot of the rendered component.
- The component name and overall element accuracy %.
- W / H badges — ✓ if width/height matched Figma.
- `px:%` — pixel match ratio if Figma PNG was available for this node.

Filter: text nodes and noisy elements are hidden by default. Click
"Show all" to see everything.

### Panel 5 — Discrepancies

A flat list of every property that failed. Each row has:
- Severity (high/medium/low).
- Type (`color`, `dimension`, `padding`, etc.).
- Location (component name or figma_id).
- Expected → Actual.

This is what the fix agent consumed during the loop. What you see in this
list is what remained *unresolved*.

### Panel 6 — Iteration Timeline

Bar chart of confidence per iteration. Good conversions show monotonic
increase. A flat or wobbly line signals a plateau — the fix agent couldn't
make progress. Look at the final discrepancies for what's stuck.

Implementation: `frontend/src/components/VerificationDashboard.tsx`.

---

## 8. Tuning Parameters

All knobs live in `backend/config.py` (`VerificationConfig` and related
settings). Key ones:

| Knob                              | Default | What raising it does                     | What lowering it does                        |
|-----------------------------------|---------|------------------------------------------|----------------------------------------------|
| `confidence_threshold`            | 0.90    | More strict → more iterations             | Lets weaker conversions pass                  |
| `max_verification_iterations`     | 10      | More fix attempts, higher API cost        | Faster fail-fast                              |
| `dimension_tolerance_px`          | 4       | Forgives more layout drift                | Catches smaller layout issues                 |
| `padding_tolerance_px`            | 2       | Forgives more spacing drift               | Catches subtle spacing issues                 |
| `color_delta_e_threshold`         | 3.0     | Accepts more color variance               | Only accepts near-identical colors            |
| `pixel_delta_e_threshold`         | 3.0     | Accepts more per-pixel noise              | Stricter pixel match                          |
| `enable_vision_comparison`        | true    | Include LLM vision tier in confidence     | Structural + pixel only                       |
| `plateau_improvement_threshold`   | 0.01    | Exits sooner if stuck                     | Keeps retrying longer                         |

**Recommended presets:**

- **Tight (design system work):** threshold 0.95, dim tolerance 2 px,
  ΔE 2.0. More iterations, fewer false passes.
- **Relaxed (content pages):** threshold 0.85, dim tolerance 6 px,
  ΔE 4.0. Passes real designs that are "close enough" without thrashing.
- **Debug:** `max_verification_iterations=1`, `enable_vision_comparison=false`.
  Runs structural only, stops after one pass, gives a fast signal.

---

## 9. Limitations — What We Do Not Check

Being explicit about what the system *cannot* verify is as important as
describing what it can. Reviewers should know these before interpreting
the 95% accuracy number.

### Not checked

1. **Text widths.** Text layout differs between Figma and browsers.
   Width variance of 5–15 px on text is expected and filtered out. A
   generated text that happens to be 50% wider than Figma will still pass.

2. **Responsive breakpoints.** We verify the design at ONE viewport
   (dynamically picked from the Figma root frame — typically 1440×900
   for desktop or 390×844 for mobile). Media queries at smaller/larger
   breakpoints are not exercised.

3. **Interaction states.** Hover, focus, active, disabled — we only see
   the rest state. A hover color being wrong would not be caught.

4. **Animation and transitions.** Not inspected. `transition: all 0.3s`
   is never verified because we only sample a static frame.

5. **Cross-browser rendering.** We test in Chromium only (Playwright
   default). Safari and Firefox differences are not tested.

6. **Font loading.** If a CSS font doesn't load (e.g., Figma uses a Google
   Font not included in the generated CSS), the browser falls back and
   font metrics diverge. The verifier may report typography matches even
   if the fallback font is visually different.

7. **Accessibility (a11y).** Color contrast, alt text, ARIA labels,
   keyboard navigation — none of this is part of the accuracy score.

8. **Semantic HTML.** We check what it *looks* like, not whether it is
   a `<nav>` vs a `<div>`.

### Known failure modes

- **Absolutely positioned SVG connectors** (arrows, lines in diagrams):
  dimension checks fail because the SVG's bounding box may include
  invisible padding. C4 diagrams especially hit this — see
  `docs/C4_TEST_PROTOCOL.md`.

- **CSS grid vs flex:** if the code uses grid but Figma auto-layout
  implies flex, structural checks may mis-attribute gaps.

- **Nested transforms:** `transform: scale()` on a parent makes
  `getBoundingClientRect` return post-scale dimensions while Figma
  bounds are pre-scale. We skip transform-wrapped elements.

---

## 10. Glossary

**CSS property** — a visual rule attached to an HTML element, e.g.,
`background-color: #1a73e8`. The browser applies these when rendering.

**DOM** — the live tree of elements the browser actually renders. Different
from the source HTML because React mutates it at runtime.

**Computed style** — the final value of a CSS property after the browser
resolves cascades, variables, and inheritance. What you get from
`window.getComputedStyle(element)`.

**Bounding box** — the smallest rectangle that contains the rendered
element, reported by `getBoundingClientRect()` in pixels.

**Figma node** — every frame, group, text, rectangle, etc. in a Figma file.
Has an ID like `34:892` and a set of properties (fills, strokes, layout,
bounds).

**Figma REST API** — HTTPS endpoint that returns file structure and can
export any node as a PNG. Docs: https://www.figma.com/developers/api.

**Auto-layout (Figma)** — Figma's flex-like layout engine. Maps directly
to CSS `flex-direction` + `gap` + `padding`.

**ΔE (Delta E)** — a scalar distance between two colors in a perceptually
uniform color space. Smaller = more similar to the human eye. ΔE2000
is the 2000 revision of the CIE formula (Sharma et al. 2005).

**CIE Lab** — a color space where equal numerical distance ≈ equal
perceived difference. RGB is not uniform this way; Lab is the foundation
of ΔE.

**Playwright** — headless browser automation library. Aura2 uses it to
spin up the generated page, inject `getComputedStyle` queries, and
screenshot individual elements.

**Dev server** — a local HTTP server (Vite in our case) that serves the
generated React app on `localhost:<port>`. Verification targets this URL.

**Structural comparison** — the tier that compares Figma JSON properties
against DOM computed styles, field by field.

**Vision comparison** — the tier that asks a vision-capable LLM "do these
two images match?" given the design screenshot and the rendered
screenshot.

**Content comparison** — the tier that compares plain text — headings,
button labels, paragraph text — between Figma and the DOM. Fallback for
when vision and structural are disabled.

**Confidence** — the weighted combination of tier scores. Range [0, 1].
Headline number on the dashboard.

**Discrepancy** — a single property-level mismatch. Each has a severity,
type, location, expected value, actual value. Fix agent consumes these.

**Fix agent** — the Claude subprocess that receives a list of discrepancies
and a pointer to the generated code, and edits files to resolve them.

**Plateau** — when confidence stops improving across iterations. Triggers
an early exit from the self-correcting loop.

---

## See Also

- `docs/05_VALIDATION_PIPELINE.md` — deep technical reference on tier
  internals, API schemas, weighting math.
- `docs/06_HOW_TO_VERIFY_ACCURACY.md` — hands-on: where files live,
  how to re-run verification, how to spot-check manually.
- `docs/C4_TEST_PROTOCOL.md` — step-by-step protocol for running against
  the C4 architecture diagram pattern.
- `backend/agents/_figma_to_react/verification.py` — the code.
- `backend/utils/structural_comparison.py` — field-by-field checks.
- `backend/utils/element_screenshot_comparison.py` — element capture and
  pixel comparison.
- `frontend/src/components/VerificationDashboard.tsx` — the UI.
