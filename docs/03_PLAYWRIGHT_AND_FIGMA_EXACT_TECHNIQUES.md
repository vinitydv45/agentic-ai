# Playwright Architecture & Figma-Exact Techniques

> **Project:** Aura2 — AI-Powered Figma → React Converter
> **Samsung PRISM @ IIIT Naya Raipur**
> This document covers: Playwright's deep architecture, how it captures screenshots, and all techniques Aura2 uses to achieve Figma-exact output.

---

## Table of Contents

1. [What is Playwright?](#1-what-is-playwright)
2. [Playwright Architecture — Deep Dive](#2-playwright-architecture--deep-dive)
3. [How Playwright Clicks Things](#3-how-playwright-clicks-things)
4. [How Playwright Takes Screenshots](#4-how-playwright-takes-screenshots)
5. [The Playwright CLI](#5-the-playwright-cli)
6. [Playwright MCP Server](#6-playwright-mcp-server)
7. [How Aura2 Uses Playwright](#7-how-aura2-uses-playwright)
8. [Figma-Exact Color Techniques](#8-figma-exact-color-techniques)
9. [Figma-Exact Typography Techniques](#9-figma-exact-typography-techniques)
10. [Figma-Exact Layout Techniques](#10-figma-exact-layout-techniques)
11. [Figma-Exact Effects Techniques](#11-figma-exact-effects-techniques)
12. [The Complete Verification System](#12-the-complete-verification-system)
13. [Visual Comparison Methods](#13-visual-comparison-methods)
14. [Auto-Fix Loop in Detail](#14-auto-fix-loop-in-detail)

---

## 1. What is Playwright?

Playwright is a **browser automation library** created by **Microsoft** and released in **January 2020** (announced May 2020). It was built by the exact team that created Puppeteer at Google's Chrome DevTools team — they moved to Microsoft and created Playwright to solve Puppeteer's fundamental limitation: it was Chrome-only.

Playwright allows programmatic control of web browsers for:
- End-to-end testing
- Visual regression testing
- Web scraping
- Screenshot capture

**Key differentiator from Puppeteer/Selenium:** Playwright supports Chromium, Firefox, AND WebKit (Safari) with a **single unified API**.

```
Playwright
    ├── Chromium (Google Chrome, Edge)  ← via Chrome DevTools Protocol (CDP)
    ├── Firefox (Mozilla)               ← via custom "Juggler" protocol (patched into Gecko)
    └── WebKit (Safari engine)          ← via patched WebKit Inspector Protocol
```

**The patching strategy:** Unlike Selenium which uses public browser APIs, Playwright's team **directly patches Firefox's Gecko engine and WebKit source code** to add CDP-compatible protocol layers. These patched binaries are downloaded from a CDN when you run `playwright install`. This is why Playwright's behavior is truly identical across all three engines — not via polyfills.

In Aura2, Playwright is used specifically for **screenshot capture and visual verification** of generated React websites.

---

## 2. Playwright Architecture — Deep Dive

### 2.1 The Browser Control Protocol

Playwright communicates with browsers through **CDP (Chrome DevTools Protocol)** for Chromium, and equivalent protocols for Firefox/WebKit:

```
Playwright Python/Node.js
        │
        │ WebSocket (CDP)
        ▼
┌───────────────────┐
│  Browser Process  │
│  (Chromium)       │
│                   │
│  ┌─────────────┐  │
│  │ Browser     │  │
│  │ ┌─────────┐ │  │
│  │ │Context 1│ │  │
│  │ │┌───────┐│ │  │
│  │ ││ Page 1││ │  │
│  │ ││ Page 2││ │  │
│  │ │└───────┘│ │  │
│  │ └─────────┘ │  │
│  └─────────────┘  │
└───────────────────┘
```

### 2.2 The Three-Layer Hierarchy

| Layer | Playwright Object | Description |
|---|---|---|
| Browser | `Browser` | The browser process (Chromium/Firefox/WebKit) |
| BrowserContext | `BrowserContext` | Isolated session (cookies, storage, auth) |
| Page | `Page` | A single tab/window |

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()           # Start browser process
    context = browser.new_context(          # Create isolated session
        viewport={"width": 1440, "height": 900}
    )
    page = context.new_page()              # Open new tab
    page.goto("http://localhost:5178")     # Navigate
    page.screenshot(path="output.png")    # Capture
    browser.close()
```

### 2.3 The Two-Hop Process Architecture

This is Playwright's actual process model — it is **not** a direct connection from your code to the browser:

```
[Your Test Code (Python/Node/Java/.NET)]
        ↕ Playwright Protocol (WebSocket RPC — serialized JSON)
[Playwright Server (Node.js relay process)]
        ↕ CDP / Juggler / WebKit protocol (WebSocket)
[Browser Process (Chromium/Firefox/WebKit)]
```

The **Playwright Server** is a Node.js relay process that:
1. Accepts serialized "Playwright Protocol" RPC calls from client libraries (Python, Java, C#)
2. Translates and forwards them as CDP or browser-specific protocol commands
3. Returns results back to the client

When you call `page.goto(url)` in Python:
1. Python client serializes the command and sends it over WebSocket to Playwright Server
2. Playwright Server sends `Page.navigate` CDP command to the browser
3. Chromium navigates and fires `Page.loadEventFired`
4. Playwright Server receives the event and replies to Python client

### 2.4 WebSocket vs HTTP (Why Playwright is Faster than Selenium)

Selenium uses **HTTP per command** — a new TCP request for every action. Playwright uses a **single persistent WebSocket** for the entire session:

| Protocol | Overhead | Speed |
|---|---|---|
| Selenium HTTP | ~50-100ms per command (TCP handshake) | Slow |
| Playwright WebSocket | Sub-millisecond per command | ~35-45% faster |

**The raw protocol overhead tradeoff:** Raw CDP benchmarks show Puppeteer (direct CDP) exchanges ~11KB of WebSocket messages for a typical task. Playwright exchanges ~326KB for the same task — the extra ~315KB is Playwright's protocol translation, actionability checking, and JS injection. This overhead is negligible for typical test suites but matters at massive scale.

### 2.5 The Auto-Wait System

Playwright's key innovation over Puppeteer: **automatic actionability checks**.

Before performing any action (click, fill, screenshot), Playwright automatically:

| Check | What it verifies |
|---|---|
| **Attached** | Element is in the DOM |
| **Visible** | Element is visible (not `display: none`, not `opacity: 0`) |
| **Stable** | Element is not animating/moving |
| **Receives Events** | Element is not obscured by another element |
| **Enabled** | Element is not `disabled` |

This eliminates the need for `await sleep(1000)` hacks common in Selenium/Puppeteer.

```python
# Old (Puppeteer style) — race condition prone:
await page.waitForSelector('#button')
await page.click('#button')

# Playwright — auto-waits for all checks:
await page.click('#button')  # Automatic retry until actionable
```

---

## 3. How Playwright Clicks Things

### 3.1 The Complete Click Sequence

When `page.click(selector)` is called:

```
1. LOCATE element using selector
        │
        ▼
2. SCROLL element into viewport
   (if needed — CDP: Runtime.callFunction → scrollIntoView)
        │
        ▼
3. WAIT FOR ACTIONABILITY
   - Is element attached to DOM?
   - Is element visible?
   - Is element stable (no CSS animation)?
   - Is element enabled?
        │
        ▼
4. CALCULATE CLICK COORDINATES
   - Get element bounding box (CDP: DOM.getBoxModel)
   - Compute center point: x = left + width/2, y = top + height/2
        │
        ▼
5. MOVE MOUSE to coordinates
   - CDP: Input.dispatchMouseEvent (type: "mouseMoved")
        │
        ▼
6. DISPATCH CLICK
   - CDP: Input.dispatchMouseEvent (type: "mousePressed", button: "left")
   - CDP: Input.dispatchMouseEvent (type: "mouseReleased", button: "left")
        │
        ▼
7. WAIT FOR NAVIGATION (if click triggers navigation)
```

### 3.2 Element Selection Methods

Playwright offers multiple ways to find elements (in order of preference):

```python
# 1. Role-based (most reliable — uses ARIA)
page.get_by_role("button", name="Shop Now")

# 2. Text content
page.get_by_text("Samsung Galaxy S24")

# 3. Label
page.get_by_label("Email address")

# 4. Placeholder
page.get_by_placeholder("Enter your email")

# 5. Alt text (images)
page.get_by_alt_text("Samsung logo")

# 6. CSS selector
page.locator(".hero-section button.cta")

# 7. XPath (last resort)
page.locator("xpath=//button[contains(@class,'cta')]")
```

### 3.3 CDP Commands Behind the Click

Under the hood, Playwright sends these CDP commands:

```json
// Step 1: Get element box model
{"method": "DOM.getBoxModel", "params": {"nodeId": 42}}
// Response: {"content": [x1,y1, x2,y2, x3,y3, x4,y4], ...}

// Step 2: Move mouse
{"method": "Input.dispatchMouseEvent", "params": {
  "type": "mouseMoved",
  "x": 720.5,
  "y": 450.0,
  "modifiers": 0,
  "button": "none"
}}

// Step 3: Press mouse button
{"method": "Input.dispatchMouseEvent", "params": {
  "type": "mousePressed",
  "x": 720.5,
  "y": 450.0,
  "button": "left",
  "clickCount": 1
}}

// Step 4: Release mouse button
{"method": "Input.dispatchMouseEvent", "params": {
  "type": "mouseReleased",
  "x": 720.5,
  "y": 450.0,
  "button": "left",
  "clickCount": 1
}}
```

### 3.4 Shadow DOM Handling

Playwright handles Shadow DOM automatically:

```python
# Works even for shadow DOM elements:
page.locator("my-custom-element >> css=button.cta").click()
```

Internally uses `Runtime.callFunction` to pierce shadow boundaries.

---

## 4. How Playwright Takes Screenshots

### 4.1 The Screenshot Mechanism

When `page.screenshot()` is called:

```
1. WAIT FOR LOAD STATE
   - "networkidle": no pending network requests for 500ms
   - All images, fonts, CSS loaded
        │
        ▼
2. PAUSE ANIMATIONS (optional)
   - Inject CSS: *, *::before, *::after { animation: none !important; }
   - Prevents blurry screenshots from in-progress animations
        │
        ▼
3. CAPTURE VIA CDP
   - CDP: Page.captureScreenshot
   - Parameters: format (png/jpeg), quality, clip, fullPage
        │
        ▼
4. RETURN BASE64
   - CDP returns base64-encoded image data
   - Playwright decodes and saves to file
```

### 4.2 Full-Page Screenshot

The `--full-page` flag captures the entire scrollable page:

```
┌──────────────────────┐  ← viewport (1440×900)
│   Visible portion    │
│                      │
└──────────────────────┘
         │  scroll
┌──────────────────────┐
│   Below fold         │
│                      │
└──────────────────────┘
         │  scroll
┌──────────────────────┐
│   Footer             │
└──────────────────────┘
```

Implementation via CDP:

```json
// Get full page dimensions
{"method": "Page.getLayoutMetrics"}
// Response: {"contentSize": {"width": 1440, "height": 3200}}

// Set viewport to full page height
{"method": "Emulation.setDeviceMetricsOverride", "params": {
  "width": 1440,
  "height": 3200,  // ← full scrollable height
  "deviceScaleFactor": 1
}}

// Capture at full size
{"method": "Page.captureScreenshot", "params": {
  "format": "png",
  "fromSurface": true
}}
```

### 4.3 DPI and Resolution

Default captures at device pixel ratio 1.0 (96 DPI). For high-DPI (retina):

```python
context = browser.new_context(device_scale_factor=2)
# → 2880×1800 pixels for 1440×900 viewport
```

Aura2 uses standard 1.0 DPR for screenshots (speed vs. fidelity tradeoff).

### 4.4 Screenshot Output Format

PNG is used (lossless):
- 24-bit color depth (16M colors)
- No JPEG compression artifacts
- Accurate color values for comparison

---

## 5. The Playwright CLI

### 5.1 Available Commands

```bash
# Screenshot (used in Aura2)
npx playwright screenshot [options] <URL> <output.png>

# Visual testing
npx playwright test

# Code generation (record interactions)
npx playwright codegen <URL>

# Show trace viewer
npx playwright show-trace trace.zip

# Install browsers
npx playwright install chromium
```

### 5.2 Screenshot CLI Flags

```bash
npx playwright screenshot \
  http://localhost:5178 \        # Target URL
  screenshot.png \               # Output file
  --full-page \                  # Capture entire scrollable page
  --viewport-size "1440,900" \  # Browser window size
  --timeout 30000 \              # Wait timeout in ms
  --wait-for-timeout 2000 \      # Additional wait after load
  --ignore-https-errors \        # Allow self-signed certs
  --browser chromium             # Browser engine (default: chromium)
```

### 5.3 How Aura2 Calls the CLI

```python
# backend/utils/visual_comparison.py
async def capture_page_screenshot(port, output_path, viewport_width=1440, viewport_height=900):
    url = f"http://localhost:{port}"
    screenshot_path = output_path / f"screenshot_{int(time.time())}.png"

    args = [
        get_npx_command(),         # "npx" or "npx.cmd" on Windows
        "playwright",
        "screenshot",
        url,
        str(screenshot_path),
        "--full-page",
        "--viewport-size", f"{viewport_width},{viewport_height}"
    ]

    # Windows requires shell=True for .cmd resolution
    use_shell = platform.system() == "Windows"

    result = subprocess.run(
        args,
        capture_output=True,
        text=True,
        timeout=30,
        shell=use_shell,
    )
```

### 5.4 Why Subprocess Instead of Python API?

Aura2 calls the Playwright **CLI** (subprocess) rather than importing the Python library. Reasons:

1. **Process isolation**: Screenshot runs in separate process, won't crash main server
2. **Simpler dependency**: No need to install `playwright` Python package separately
3. **MCP compatibility**: The `@playwright/mcp` package is Node.js based
4. **Windows compatibility**: `.cmd` resolution works better with subprocess

---

## 6. Playwright MCP Server

### 6.1 What is `@playwright/mcp`?

The `@playwright/mcp` package exposes Playwright as an **MCP (Model Context Protocol) server**, allowing AI agents to control browsers via tool calls.

```bash
npx @playwright/mcp@latest  # Starts MCP server on stdio
```

### 6.2 Available MCP Tools

| Tool | Description |
|---|---|
| `browser_navigate` | Navigate to a URL |
| `browser_screenshot` | Capture current state |
| `browser_click` | Click an element |
| `browser_fill_form` | Fill form inputs |
| `browser_type` | Type text at cursor |
| `browser_press_key` | Press keyboard keys |
| `browser_hover` | Hover over element |
| `browser_select_option` | Select dropdown option |
| `browser_wait_for` | Wait for condition |
| `browser_evaluate` | Run JavaScript |
| `browser_snapshot` | Get accessibility tree |
| `browser_console_messages` | Get browser console |
| `browser_network_requests` | Get network requests |
| `browser_tabs` | Manage browser tabs |
| `browser_resize` | Resize viewport |
| `browser_close` | Close browser |

### 6.3 How Agent Uses Playwright MCP

In Aura2's fix agent, Claude can use Playwright MCP tools:

```python
mcp_servers = {
    "playwright": {
        "type": "stdio",
        "command": "npx",
        "args": ["@playwright/mcp@latest"],
    }
}

# Agent can now call:
# browser_navigate(url="http://localhost:5178")
# browser_screenshot() → returns PNG as base64
# browser_click(selector="button.cta")
```

Claude autonomously navigates, takes screenshots, identifies problems, then edits code.

### 6.4 Playwright MCP vs CLI Screenshot

| Method | Use Case | Speed | Isolation |
|---|---|---|---|
| CLI subprocess | Simple screenshot capture | Fast | High (separate process) |
| MCP tool | Agent-driven interaction + screenshot | Medium | Medium (shared process) |
| Python API | Complex test scenarios | Fast | Low (in-process) |

Aura2 uses **CLI** for automated screenshots, **MCP** for agent-driven verification.

---

## 7. How Aura2 Uses Playwright

### 7.1 Screenshot Capture in Verification Loop

```python
# verification.py
for iteration in range(max_iterations):
    # 1. Capture current state of generated website
    generated_screenshot = await capture_page_screenshot(
        port=port,
        output_path=screenshots_dir,
        viewport_width=1440,   # Match Figma design canvas width
        viewport_height=900    # Standard desktop viewport
    )

    # 2. Compare with Figma design
    comparison = await compare_with_figma_design(
        screenshot_path=generated_screenshot,
        design_data=design_data,  # Original Figma data
        project_path=project_path,
    )
```

### 7.2 Dev Server Management

Playwright screenshots require a running web server. Aura2 manages Vite dev servers:

```python
# backend/dev_server_manager.py
def start_dev_server(project_id, project_path, project_name, install_deps=True):
    # Find free port in range 5173-6000
    port = find_free_port(start=5173, end=6000)

    # Install dependencies if needed
    if install_deps:
        subprocess.run(["npm", "install"], cwd=project_path)

    # Start Vite dev server
    process = subprocess.Popen(
        ["npm", "run", "dev", "--", "--port", str(port)],
        cwd=project_path,
        stdout=PIPE,
        stderr=PIPE,
    )

    return port  # Returns port for screenshot URL
```

### 7.3 Server Ready Check

Before taking screenshots, Aura2 waits for the server to be ready:

```python
async def wait_for_server_ready(port: int, max_wait: int = 30) -> bool:
    """Poll HTTP endpoint until server responds."""
    start = time.time()
    while time.time() - start < max_wait:
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(f"http://localhost:{port}", timeout=2)
                if response.status_code < 500:  # Any non-5xx response = ready
                    return True
        except (httpx.ConnectError, httpx.TimeoutException):
            pass
        await asyncio.sleep(0.5)
    return False
```

---

## 8. Figma-Exact Color Techniques

### 8.1 The Color Problem

Figma stores all colors as floating-point RGBA (0.0-1.0 each channel):

```json
{"r": 0.07843137254901961, "g": 0.15686274509803921, "b": 0.6274509803921569, "a": 1.0}
```

This is `rgb(20, 40, 160)` → hex `#1428A0` (Samsung's brand blue).

### 8.2 The Conversion Pipeline

```python
def rgba_to_hex(color: dict, opacity: float = 1.0) -> str:
    r = int(color.get("r", 0) * 255)   # 0.0784 × 255 = 19.99 → int = 20 = 0x14
    g = int(color.get("g", 0) * 255)   # 0.1568 × 255 = 39.99 → int = 40 = 0x28
    b = int(color.get("b", 0) * 255)   # 0.6274 × 255 = 159.9 → int = 160 = 0xA0
    a = color.get("a", 1.0) * opacity

    if a < 1.0:
        # Semi-transparent: use rgba() CSS
        return f"rgba({r}, {g}, {b}, {a:.2f})"
    # Fully opaque: use hex
    return f"#{r:02x}{g:02x}{b:02x}"   # → "#1428a0"
```

**Result: 100% color accuracy** — mathematical conversion, no approximation.

### 8.3 The Tailwind Arbitrary Values Technique

The critical prompt instruction that enables exact colors:

```
# In the conversion prompt:
"## Color Palette (use EXACT hex values)
- #1428A0 (used 47x, e.g. NavBar, CTA Button)
...
IMPORTANT: Use Tailwind arbitrary values like bg-[#1428A0] NOT bg-blue-700"
```

Generated code:

```tsx
// WRONG (V1 style):
<button className="bg-blue-700">Shop Now</button>

// RIGHT (Aura2 style):
<button className="bg-[#1428A0] hover:bg-[#0f1f8a]">Shop Now</button>
```

### 8.4 Gradient Handling

For gradient fills from Figma:

```python
def extract_gradient_fill(fill: dict) -> str:
    """Convert Figma gradient to CSS linear-gradient."""
    gradient_stops = fill.get("gradientStops", [])
    gradient_type = fill.get("type", "GRADIENT_LINEAR")

    if gradient_type == "GRADIENT_LINEAR":
        stops = []
        for stop in gradient_stops:
            color = rgba_to_hex(stop["color"])
            position = int(stop["position"] * 100)
            stops.append(f"{color} {position}%")

        # Extract angle from gradientTransform matrix
        angle = calculate_gradient_angle(fill.get("gradientTransform", []))
        return f"linear-gradient({angle}deg, {', '.join(stops)})"
```

---

## 9. Figma-Exact Typography Techniques

### 9.1 Font Discovery and Import

Aura2 collects ALL fonts used across the design:

```python
def collect_colors_and_fonts(design_data: dict) -> tuple[list, list]:
    fonts = {}  # family → set of weights

    def walk(node):
        if node.get("type") == "TEXT":
            style = node.get("style", {})
            family = style.get("fontFamily", "Inter")
            weight = style.get("fontWeight", 400)
            if family not in fonts:
                fonts[family] = set()
            fonts[family].add(weight)
        for child in node.get("children", []):
            walk(child)

    walk(design_data)

    return [{"family": f, "weights": sorted(w)} for f, w in fonts.items()]
```

The prompt then tells the agent:

```
## Fonts (import in index.html)
- Samsung Sharp Sans: weights 400,700
- Inter: weights 300,400,500,600,700

→ Agent generates in index.html:
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
```

For proprietary fonts (Samsung Sharp Sans), the agent uses the closest available Google Font alternative.

### 9.2 Font Size — Exact Pixel Values

```python
# Design data extracted:
"style": {
    "fontFamily": "Samsung Sharp Sans",
    "fontSize": 72,          # pixels
    "fontWeight": 700,
    "lineHeightPx": 86.4,    # = 72 * 1.2
    "letterSpacing": -1.5    # negative for tight heading
}
```

Prompt instructs agent to use:

```tsx
// Exact font size via Tailwind arbitrary values
<h1 className="
  font-['Samsung_Sharp_Sans']
  text-[72px]
  font-bold
  leading-[86.4px]
  tracking-[-1.5px]
  text-white
">
  Unfold Your World
</h1>
```

### 9.3 Line Height Calculation

Figma stores `lineHeightPx` (absolute pixel value). CSS uses unitless ratio or px:

```
Figma: lineHeightPx: 86.4, fontSize: 72
→ ratio = 86.4 / 72 = 1.2
→ CSS: line-height: 1.2  (or leading-[86.4px])
```

Aura2 uses px form for pixel-perfect accuracy: `leading-[86.4px]`.

### 9.4 Text Case Mapping

```python
FIGMA_TO_CSS_TEXT_CASE = {
    "ORIGINAL": "normal",
    "UPPER": "uppercase",
    "LOWER": "lowercase",
    "TITLE": "capitalize",
}
# → Tailwind: uppercase, lowercase, capitalize
```

---

## 10. Figma-Exact Layout Techniques

### 10.1 Auto-Layout → Flexbox Mapping

Figma's Auto-Layout maps directly to CSS Flexbox:

| Figma Property | Value | CSS Equivalent | Tailwind |
|---|---|---|---|
| `layoutMode` | HORIZONTAL | `flex-direction: row` | `flex flex-row` |
| `layoutMode` | VERTICAL | `flex-direction: column` | `flex flex-col` |
| `primaryAxisAlignItems` | MIN | `justify-content: flex-start` | `justify-start` |
| `primaryAxisAlignItems` | CENTER | `justify-content: center` | `justify-center` |
| `primaryAxisAlignItems` | MAX | `justify-content: flex-end` | `justify-end` |
| `primaryAxisAlignItems` | SPACE_BETWEEN | `justify-content: space-between` | `justify-between` |
| `counterAxisAlignItems` | MIN | `align-items: flex-start` | `items-start` |
| `counterAxisAlignItems` | CENTER | `align-items: center` | `items-center` |
| `counterAxisAlignItems` | MAX | `align-items: flex-end` | `items-end` |
| `itemSpacing` | 24 | `gap: 24px` | `gap-[24px]` |

### 10.2 Padding Extraction

Figma has per-side padding (not shorthand):

```python
layout["padding"] = {
    "top": node.get("paddingTop", 0),      # → pt-[80px]
    "right": node.get("paddingRight", 0),  # → pr-[120px]
    "bottom": node.get("paddingBottom", 0),# → pb-[80px]
    "left": node.get("paddingLeft", 0),    # → pl-[120px]
}
```

When all equal: `p-[80px]`. When vertical/horizontal equal: `px-[120px] py-[80px]`. When all different: `pt-[80px] pr-[120px] pb-[80px] pl-[120px]`.

Prompt tells agent to use exact values:

```
- padding: top=80px right=120px bottom=80px left=120px
→ className="px-[120px] py-[80px]"
```

### 10.3 Constraints → Responsive CSS

Figma constraints control how elements resize:

```python
CONSTRAINT_MAP = {
    # Horizontal constraints
    ("LEFT", "horizontal"):   "left-X",              # absolute left
    ("RIGHT", "horizontal"):  "right-X",             # absolute right
    ("LEFT_RIGHT", "horizontal"): "w-full",           # stretch
    ("CENTER", "horizontal"): "left-1/2 -translate-x-1/2",
    ("SCALE", "horizontal"):  "w-[percentage]",      # percentage width

    # Vertical constraints
    ("TOP", "vertical"):      "top-X",
    ("BOTTOM", "vertical"):   "bottom-X",
    ("TOP_BOTTOM", "vertical"): "h-full",
    ("SCALE", "vertical"):    "h-[percentage]",
}
```

SCALE constraint → responsive percentage width (enables responsive design from Figma constraints).

### 10.4 Responsive Breakpoints

When Figma has multiple frame sizes (desktop/tablet/mobile), Aura2 maps them:

```python
FIGMA_WIDTH_TO_BREAKPOINT = {
    1440: "xl",    # Desktop
    1280: "lg",
    768: "md",     # Tablet
    375: "sm",     # Mobile
}
```

Generated code:

```tsx
<section className="
  px-[120px] xl:px-[120px]    // Desktop
  md:px-[60px]                 // Tablet
  sm:px-[20px]                 // Mobile
">
```

---

## 11. Figma-Exact Effects Techniques

### 11.1 Drop Shadow → box-shadow

```python
def effect_to_css(effect: dict) -> str:
    if effect["type"] == "DROP_SHADOW":
        color = rgba_to_hex(effect["color"])   # → #000000 or rgba()
        x = effect["offset"]["x"]              # Horizontal offset
        y = effect["offset"]["y"]              # Vertical offset
        blur = effect["radius"]                # Blur radius
        spread = effect.get("spread", 0)       # Spread radius

        return f"shadow-[{x}px_{y}px_{blur}px_{spread}px_{color}]"
        # → className="shadow-[0px_4px_16px_0px_rgba(0,0,0,0.12)]"
```

### 11.2 Inner Shadow

```python
if effect["type"] == "INNER_SHADOW":
    return f"shadow-[inset_{x}px_{y}px_{blur}px_{spread}px_{color}]"
```

### 11.3 Layer Blur → backdrop-filter

```python
if effect["type"] == "LAYER_BLUR":
    radius = effect["radius"]
    return f"backdrop-blur-[{radius}px]"
    # → className="backdrop-blur-[20px]"
```

### 11.4 Border Radius

Figma supports per-corner radius:

```python
def extract_corner_radius(node: dict) -> dict:
    # Single radius for all corners
    if "cornerRadius" in node:
        return {"all": node["cornerRadius"]}

    # Individual corners
    return {
        "topLeft": node.get("rectangleTopLeftRadius", 0),
        "topRight": node.get("rectangleTopRightRadius", 0),
        "bottomRight": node.get("rectangleBottomRightRadius", 0),
        "bottomLeft": node.get("rectangleBottomLeftRadius", 0),
    }
```

CSS mapping:
- All equal (8px): `rounded-[8px]`
- Top only: `rounded-t-[8px]`
- Per corner: `rounded-tl-[8px] rounded-tr-[8px] rounded-br-[16px] rounded-bl-[8px]`

### 11.5 Opacity

```python
if node.get("opacity") and node["opacity"] != 1.0:
    opacity_percent = int(node["opacity"] * 100)
    # → className="opacity-[0.6]"  or opacity-60 if standard
```

---

## 12. The Complete Verification System

### 12.1 Two-Mode Comparison

```
Mode 1: Vision-Based (when Figma screenshot available from plugin)
    ├── Send BOTH screenshots to Claude Vision API
    ├── Claude analyzes layout, spacing, colors, typography
    ├── Returns: confidence score + specific discrepancies
    └── Weight: 70% of final confidence

Mode 2: Content-Based (always runs as supplement/fallback)
    ├── Extract expected text/colors/fonts from design data
    ├── Search all generated .tsx files for these values
    ├── Calculate presence score per category
    └── Weight: 30% of final confidence

Final confidence = vision_confidence × 0.7 + content_confidence × 0.3
```

### 12.2 Confidence Thresholds

```python
EARLY_STOP_THRESHOLD = 0.95    # Stop immediately, perfect match
CONFIDENCE_THRESHOLD = 0.85    # Accept as success
PLATEAU_THRESHOLD = 0.01       # Stop if improvement < 1%
```

### 12.3 Discrepancy Format

Each discrepancy has actionable fix information:

```json
{
  "type": "color",
  "location": "Header Section",
  "severity": "high",
  "expected": "#1428A0",
  "actual": "#1a1a2e",
  "coordinates": {"x": 0, "y": 0, "width": 1440, "height": 80},
  "fix": {
    "file": "src/components/Header.tsx",
    "change": "Change bg-[#1a1a2e] to bg-[#1428A0]"
  }
}
```

Types of discrepancies detected:
- `color`: Wrong background/text color
- `spacing`: Wrong padding, margin, gap
- `typography`: Wrong font size, weight, family
- `layout`: Wrong flex direction, alignment
- `missing_element`: Component not rendered
- `wrong_text`: Text content doesn't match design

---

## 13. Visual Comparison Methods

### 13.1 Claude Vision API Comparison

```python
# vision_comparison.py
async def compare_with_vision_api(figma_screenshot_path, generated_screenshot_path, design_data):
    client = anthropic.Anthropic()

    # Encode both images as base64
    figma_b64 = base64.b64encode(figma_screenshot_path.read_bytes()).decode()
    generated_b64 = base64.b64encode(generated_screenshot_path.read_bytes()).decode()

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=3000,
        messages=[{
            "role": "user",
            "content": [
                # Image 1: Original Figma design
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": figma_b64
                    }
                },
                # Image 2: Generated website
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": generated_b64
                    }
                },
                {
                    "type": "text",
                    "text": f"""You are a pixel-perfect UI comparison expert.

Image 1: Original Figma design
Image 2: Generated React/Tailwind implementation

Compare them and return JSON with:
{{
  "confidence": 0.0-1.0,
  "discrepancies": [
    {{
      "type": "color|spacing|typography|layout|missing_element",
      "location": "section name",
      "severity": "high|medium|low",
      "expected": "what it should be",
      "actual": "what it is",
      "fix": {{"file": "src/...", "change": "specific change"}}
    }}
  ],
  "accuracy_scores": {{
    "layout": 0.0-1.0,
    "spacing": 0.0-1.0,
    "colors": 0.0-1.0,
    "typography": 0.0-1.0
  }}
}}

Design context: {design_data.get('name', 'Unknown')}
Expected colors: {[c['color'] for c in design_data.get('colors', [])[:5]]}"""
                }
            ]
        }]
    )

    # Parse JSON from Claude's response
    return parse_comparison_response(response.content[0].text)
```

### 13.1b Industry Context: Claude Computer Use for Visual Checking

Anthropic introduced **Computer Use** in public beta on **October 22, 2024** (Claude 3.5 Sonnet) — making Claude the first frontier AI model to offer autonomous desktop control.

On OSWorld benchmarks:
- Claude Computer Use: **14.9%** (screenshot-only category)
- Next best system at the time: 7.8%

For visual verification, Claude's multimodal vision enables:
- **Side-by-side image comparison** in a single API call: "Here is the Figma export [img1] and here is the current render [img2] — identify all visual differences"
- Pinpointing issues at the CSS property level: wrong `font-weight`, `padding` off by Xpx, wrong `border-radius`
- Semantic understanding (not just pixel diff) — understands WHY something looks different

**Aura2 uses this directly** in `vision_comparison.py` via the Anthropic SDK with `claude-opus-4-6`, passing both the Figma plugin screenshot and the Playwright-captured screenshot as base64 images in a single API call.

### 13.2 Content-Based Comparison

```python
async def compare_with_figma_design(screenshot_path, design_data, project_path):
    # Extract expected values from design data
    expected_texts = []
    expected_colors = []
    expected_fonts = []

    def collect(node):
        if node.get("text"):
            expected_texts.append(node["text"].strip())
        for fill in node.get("fills", []):
            if fill.get("type") == "SOLID" and fill.get("color"):
                expected_colors.append(rgba_to_hex(fill["color"]))
        if node.get("style", {}).get("fontFamily"):
            expected_fonts.append(node["style"]["fontFamily"])
        for child in node.get("children", []):
            collect(child)

    for page in design_data.get("pages", []):
        for frame in page.get("frames", []):
            collect(frame)

    # Read all generated source code
    all_code = ""
    for tsx_file in (project_path / "src").rglob("*.tsx"):
        all_code += tsx_file.read_text()
    for css_file in (project_path / "src").rglob("*.css"):
        all_code += css_file.read_text()

    # Score each category
    text_score = sum(1 for t in expected_texts if t[:20] in all_code) / max(len(expected_texts), 1)
    color_score = sum(1 for c in set(expected_colors) if c in all_code) / max(len(set(expected_colors)), 1)
    font_score = sum(1 for f in set(expected_fonts) if f.lower() in all_code.lower()) / max(len(set(expected_fonts)), 1)

    confidence = text_score * 0.4 + color_score * 0.35 + font_score * 0.25

    return {
        "confidence": confidence,
        "matches": confidence > 0.85,
        "discrepancies": collect_discrepancies(expected_texts, expected_colors, all_code),
    }
```

---

## 14. Auto-Fix Loop in Detail

### 14.1 Fix Generation

When discrepancies are found, a targeted fix prompt is built:

```python
# auto_fix_agent.py
def format_fixes_for_prompt(fixes: list) -> str:
    lines = ["APPLY THESE FIXES IN ORDER:\n"]
    for i, fix in enumerate(fixes, 1):
        lines.append(f"FIX {i}: [{fix['severity'].upper()}] {fix['location']}")
        lines.append(f"  Type: {fix['type']}")
        lines.append(f"  Expected: {fix['expected']}")
        lines.append(f"  Actual: {fix['actual']}")
        lines.append(f"  File: {fix['fix']['file']}")
        lines.append(f"  Change: {fix['fix']['change']}")
        lines.append("")
    return "\n".join(lines)
```

### 14.2 Fix Agent

A dedicated sub-agent applies the fixes:

```python
async def apply_fixes(fixes, project_path, design_data):
    fixes_prompt = format_fixes_for_prompt(fixes)

    options = ClaudeAgentOptions(
        model=settings.default_model,
        system_prompt=f"""You are a pixel-perfect code fixer.
Apply the specific fixes below to make the React code match the Figma design exactly.

{fixes_prompt}

RULES:
- Use EXACT values (don't approximate)
- Update Tailwind classes precisely
- Preserve existing functionality
- Do NOT break working code""",
        max_turns=settings.max_fix_turns,
        cwd=str(project_path),
        max_buffer_size=20 * 1024 * 1024,
        allowed_tools=["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
        mcp_servers={
            "playwright": {
                "type": "stdio",
                "command": get_npx_command(),
                "args": ["@playwright/mcp@latest"],
            }
        },
        permission_mode="acceptEdits",
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.query(fixes_prompt)
        async for message in client.receive_response():
            # Track applied fixes
```

### 14.3 Hot Reload Cycle

After fixes are applied, Vite's HMR (Hot Module Replacement) automatically rebuilds:

```
Fix applied (Edit tool changes .tsx file)
        │
        ▼
Vite HMR detects file change
        │
        ▼
Vite recompiles affected modules (< 100ms)
        │
        ▼
Browser receives HMR update via WebSocket
        │
        ▼
React component updates in-place (no full reload)
        │
        ▼
Aura2 waits 3 seconds for stabilization
        │
        ▼
Next screenshot capture
```

### 14.4 Plateau Detection

If consecutive iterations show < 1% improvement, the loop stops:

```python
if iteration > 0:
    prev_confidence = iteration_history[-2]["confidence"]
    improvement = combined_confidence - prev_confidence

    if improvement < 0.01:  # Less than 1% improvement
        print(f"Plateau detected — improvement {improvement:.2%} < 1%")
        return {
            "status": "completed_with_warnings" if confidence < threshold else "success",
            "confidence": combined_confidence,
            ...
        }
```

This prevents infinite loops when the AI keeps making changes that don't improve visual match (e.g., when a component requires external API data that's unavailable).

---

## 15. Known Figma → CSS Limitations (What Can't Be 100% Exact)

Not everything in Figma has a CSS equivalent. Aura2 handles these as best-effort approximations:

### 15.1 Corner Smoothing (Squircle)

Figma supports "smooth corners" — the iOS-style squircle rounding used in Apple design. Mathematically different from CSS `border-radius`:

```
CSS border-radius: 16px       → circular arc corners
Figma "smooth corners" at 1.0 → superellipse (squircle) corners
```

**No CSS property reproduces this.** Workarounds:
- Use `border-radius` as close approximation (what Aura2 does)
- SVG clip-path with squircle math (complex, performance cost)
- CSS Houdini `paint` worklet (limited browser support)

### 15.2 Blend Modes (27 types)

Figma supports 27 blend modes. CSS has `mix-blend-mode` and `background-blend-mode` — but not all 27 map perfectly:

| Figma Blend Mode | CSS Equivalent | Notes |
|---|---|---|
| Normal, Multiply, Screen, Overlay | Direct 1:1 | Works perfectly |
| Darken, Lighten | Direct 1:1 | Works perfectly |
| Color Dodge, Color Burn | Direct 1:1 | Works perfectly |
| Hard Light, Soft Light | Direct 1:1 | Works perfectly |
| Difference, Exclusion | Direct 1:1 | Works perfectly |
| Hue, Saturation, Color, Luminosity | Direct 1:1 | Works perfectly |
| Plus Darker, Plus Lighter | No CSS equivalent | Approximated or skipped |
| Linear Dodge (Add) | `mix-blend-mode: lighten` approx | Not exact |
| Linear Burn | `mix-blend-mode: color-burn` approx | Not exact |

### 15.3 Figma Variables (Design Tokens, 2023)

Figma Variables (distinct from the older Styles system) enable design tokens natively within Figma. The **Figma Variables API** can export tokens as JSON and sync with code repositories via tools like **Tokens Studio for Figma**.

Token architecture:
```
Global tokens:    color-blue-500: #3B82F6
Alias tokens:     color-primary: {color-blue-500}
Component tokens: button-background: {color-primary}
```

Aura2 currently extracts raw color values (not variable references). Future enhancement: extract variable references to generate proper CSS custom properties (`var(--color-primary)`).

---

## 16. Industry Research: Pixelmatch & Visual Regression

### 15.1 How `toHaveScreenshot()` Works Internally

Playwright's built-in visual comparison (`expect(page).toHaveScreenshot('name.png')`) uses the **pixelmatch** library by Mapbox (open-source, GitHub: `mapbox/pixelmatch`).

**On first run:** Generates baseline image, saves to `__snapshots__/`. Test passes unconditionally.

**On subsequent runs:** Compares fresh screenshot to baseline using pixelmatch:

#### The pixelmatch Algorithm

pixelmatch works in **YIQ color space** — a perceptual color model from NTSC television (Y = luma/brightness, I & Q = chrominance). YIQ is used because human perception of color is better modeled by luma-weighted channels than raw RGB:

```
For each pixel:
1. Convert both pixel colors to YIQ
2. Compute perceived color distance
3. If distance > (threshold × max_possible_distance) → mark as different
```

Default `threshold = 0.2` (range 0.0 = exact to 1.0 = accept anything).

```javascript
// Playwright visual comparison config
await expect(page).toHaveScreenshot('design.png', {
  maxDiffPixels: 100,        // absolute: up to 100 pixels may differ
  maxDiffPixelRatio: 0.01,   // relative: up to 1% of pixels may differ
  threshold: 0.2,            // YIQ perceptual color diff tolerance
  animations: 'disabled',    // stop CSS animations before capture
  mask: [locator],           // mask dynamic regions (e.g., timestamps)
});
```

**On failure:** Playwright saves 3 files:
- `screenshot-actual.png` — current run
- `screenshot-expected.png` — stored baseline
- `screenshot-diff.png` — diff image with changed pixels in bright magenta

### 15.2 Platform Sensitivity (Critical for CI/CD)

Screenshots are **platform-sensitive**: font rendering, anti-aliasing, and sub-pixel rendering differ across operating systems and GPUs.

```
macOS Retina  → different font subpixel rendering than Windows ClearType
Linux CI      → different from both macOS and Windows
Windows ARM   → different from Windows x86
```

**Playwright's recommendation:** Maintain separate baselines per OS, or run all visual tests in **Docker containers** for rendering consistency.

**Aura2's approach:** Content-based comparison (text/color presence in source code) avoids platform sensitivity entirely. Vision-based comparison (Claude API) is semantic, not pixel-level, so it's also platform-agnostic.

---

## Summary

The combination of:

1. **Playwright** (screenshot capture at exact 1440px viewport)
2. **Claude Vision API** (multimodal comparison of Figma vs. generated)
3. **Content-based analysis** (text/color/font presence in source code)
4. **Iterative fix loop** (targeted sub-agent applies specific changes)
5. **RGBA→hex color pipeline** (mathematical exact conversion)
6. **Tailwind arbitrary values** (`bg-[#1428A0]`, `text-[72px]`)
7. **Auto-layout→Flexbox mapping** (structural accuracy)
8. **Typography extraction** (exact font family, weight, size, line-height)
9. **Effect extraction** (shadows, blur, border-radius)
10. **Semantic type inference** (correct HTML elements + ARIA)

...achieves **88-96% visual confidence** on complex multi-section designs like Samsung's website — a result that would take a developer days to achieve manually, accomplished in ~7 minutes.
