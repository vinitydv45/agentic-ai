# C4 Pattern Test Protocol

> **Goal:** ready-to-run protocol for validating Aura2 against a C4
> architecture diagram (mentor-supplied Figma file).
>
> Aura2's pipeline is generic — no C4-specific code exists today.
> This doc captures the steps, expected outputs, and known risks so
> we can execute the test the moment the Figma file lands.
>
> **Prerequisites:** read [VERIFICATION_GUIDE.md](./VERIFICATION_GUIDE.md)
> first if new to the verifier.

---

## 1. What C4 Is (One Paragraph)

The **C4 model** is a notation for describing software architecture at
four zoom levels — **Context**, **Container**, **Component**,
**Code**. Diagrams look like labeled boxes connected by labeled arrows.
Visually they are simple: rectangles with title + subtitle text, lines
between them, and sometimes dashed lines for asynchronous flows.

Why this is interesting as a test case:
- The shapes are trivial (rectangles, lines, text) — the pipeline
  should hit them easily.
- Connectors (arrows, lines) are SVG elements, which the current
  verifier doesn't handle gracefully.
- Nested hierarchy (System → Container → Component) stresses our
  FRAME/GROUP recursion.

---

## 2. Pre-Flight Checklist

Before starting, confirm:

- [ ] Mentor's C4 Figma file URL is available.
- [ ] Figma personal access token is in `.env` (`FIGMA_TOKEN=…`).
- [ ] `backend/config.py` has `confidence_threshold` at the intended
      value (default 0.90 is fine for a first run).
- [ ] Node 18+ and `uv` installed.
- [ ] Dev database / storage has space for a fresh project.

---

## 3. Run the Test

### Step 1 — Start backend

```bash
uv run python -m uvicorn backend.main:app --reload --port 8000
```

Wait for `Application startup complete.`

### Step 2 — Start frontend

```bash
cd frontend && npm run dev
```

Open `http://localhost:5173`.

### Step 3 — Submit the C4 file

Two paths:

**Path A — via plugin** (recommended):
1. Open the C4 Figma file.
2. Run the Aura2 Figma plugin.
3. Plugin uploads design JSON + images.
4. Note the **project ID** returned (`projects/<id>`).

**Path B — via API** (if plugin is unavailable):
```bash
curl -X POST http://localhost:8000/api/projects/from-figma \
  -H "Content-Type: application/json" \
  -d '{"figma_url": "<C4_FIGMA_URL>", "figma_token": "$FIGMA_TOKEN"}'
```

### Step 4 — Watch conversion

Dashboard shows pipeline phases:
```
SETUP → CONVERSION → BUILD → QUALITY → VERIFICATION → FIX → DONE
```

Each phase logs to `logs/project_<id>.log`. If CONVERSION stalls past
5 minutes, check the log.

### Step 5 — Inspect the verification report

Navigate to `/projects/<id>/verification`. See:
- Overall confidence.
- Per-category scores (Colors, Spacing, Typography, Effects, Dimensions, Pixel).
- Per-element grid — look for boxes labeled with C4 container/component
  names.
- Discrepancies list.

---

## 4. Expected Output

A successful C4 conversion should show:

| Metric                 | Expected Range      | Notes                                 |
|------------------------|---------------------|---------------------------------------|
| Overall confidence     | ≥ 0.85              | Lower than a webpage due to SVG lines |
| Colors                 | ≥ 0.95              | C4 uses a small, solid palette        |
| Typography             | ≥ 0.95              | Box labels are simple text            |
| Dimensions (boxes)     | ≥ 0.80              | FRAMES should measure correctly       |
| Dimensions (arrows)    | ≤ 0.70 (expected)   | SVG connector bounds are fuzzy        |
| Element count          | ~ 5–20 per diagram  | One per container / component         |

**Per-element grid** should show one card per labeled box (System,
Container, Component). Arrow-only elements may appear with low scores
— that is a known limitation (see §6).

---

## 5. What to Manually Spot-Check

Automated verification is not enough for C4. Open the generated page
alongside the Figma file and check:

1. **Box labels read correctly.** The text inside each box matches
   Figma exactly (including `<technology>` subtitles in angle brackets
   if present).
2. **Arrows point the right way.** The verifier doesn't check arrow
   direction; only that *something* is drawn there. Confirm the
   generated arrows terminate on the correct boxes.
3. **Dashed vs solid lines.** If the Figma file uses dashed lines for
   async flows, verify the CSS output uses `border-style: dashed` or
   the equivalent SVG `stroke-dasharray`.
4. **Nested containers.** In a Container diagram, the outer system
   boundary should wrap all containers. If the generated code
   flattens the hierarchy, the visual enclosure is lost even though
   child boxes look right.

---

## 6. Known Risks Specific to C4

### Risk A — SVG connectors fail dimension checks

**Symptom:** elements with names like `Arrow 12`, `Line 3` score
poorly on width/height match.

**Why:** SVG bounding boxes include invisible padding from stroke
width and endpoint markers. Figma reports the geometric bounds; the
browser reports the bounding rect including markers. These diverge.

**Mitigation:** treat connector dimension scores as non-diagnostic.
Look at the pixel match instead — if pixel match ≥ 0.85 on a
connector the arrow is visually correct even if the dim score is 0.
Future fix: detect `VECTOR`/`LINE` node types and skip dimension
check for them.

### Risk B — Nested System-Container-Component hierarchy

**Symptom:** child boxes render in correct positions but the outer
System frame is missing or has wrong background color.

**Why:** our `_extract_figma_properties` walks the tree depth-first
but flattens children. Parent frames may lose their background fill
if their only job is to group children.

**Mitigation:** inspect `figma_data/design_data.json` — confirm the
outermost FRAME has `fills` preserved. If it does and the DOM doesn't
render it, the issue is in the code generator, not extraction.

### Risk C — Label text vs technology annotation

**Symptom:** "Web App `<TypeScript>`" renders as two elements
instead of one, with the technology tag mis-styled.

**Why:** Figma typically draws these as a single TEXT node with
mixed styles. Our pipeline may split them. Check the generated JSX
for whether the label is one `<p>` or two.

**Mitigation:** manual fix in the code, document the pattern for
future.

### Risk D — C4 color palette ambiguity

**Symptom:** all containers render in the same color even though
Figma uses distinct palette entries.

**Why:** C4 conventionally uses blue for System, cyan for Container,
green for Component. If the Figma file uses Figma Variables to
reference these, the extraction may resolve to the same value or
lose the variable binding.

**Mitigation:** inspect `figma_data/raw_figma_response.json` — search
for `boundVariables` on each container to confirm distinct color
refs.

---

## 7. Decision Tree After the Test

```
Confidence ≥ 0.85?
  │
  ├── YES ──► Report success. Document which risks materialized.
  │          Ship a video/screenshot demo to mentor.
  │
  └── NO ──► Identify the dominant failure:
               │
               ├── Dimensions low on boxes
               │     → issue is code generator, not verifier.
               │       Open a generator-side bug.
               │
               ├── Colors low
               │     → check fills extraction in design_data.json.
               │
               ├── Pixel match low, dims fine
               │     → visual drift (wrong fonts, missing shadows).
               │       Look at side-by-side screenshots.
               │
               └── Typography low
                     → font family not loaded. Check CSS @import.
```

---

## 8. What NOT to Do

- **Don't add C4-specific logic to the pipeline unless testing
  proves generic code can't handle it.** The whole point of the
  test is to see what breaks with a real-world non-webpage design.
  Premature specialization defeats the experiment.
- **Don't tune `confidence_threshold` down just to make C4 pass.**
  If it fails at 0.90, the answer is to fix the underlying
  discrepancy or accept the gap — not to lower the bar.
- **Don't skip the plugin export** and hand-build design JSON. We
  want to exercise the plugin path too.

---

## 9. After the Test — Reporting

Capture and hand back to mentor:

1. **Confidence number** — headline.
2. **Per-category scores** — screenshot of dashboard.
3. **Side-by-side** — Figma vs rendered, full page.
4. **Per-element scores** — where boxes passed, where arrows failed.
5. **Top-5 discrepancies** — what remained unresolved.
6. **Failure mode narrative** — which risks from §6 triggered.
7. **Wall-clock time** — how long CONVERSION + VERIFICATION took.

Store all artifacts under `generated_projects/<c4_project_name>/` —
same location as other projects. No special handling.

---

## See Also

- [VERIFICATION_GUIDE.md](./VERIFICATION_GUIDE.md) — background on
  the 3-tier verifier.
- [06_HOW_TO_VERIFY_ACCURACY.md](./06_HOW_TO_VERIFY_ACCURACY.md) —
  where files live, how to re-run verification.
- [C4 model official reference](https://c4model.com/) — for anyone
  unfamiliar with the notation.
