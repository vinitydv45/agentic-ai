# Aura2 — Complete Setup Guide

> Covers everything needed to run Aura2 from a fresh clone: system
> requirements, environment variables, Figma plugin, Figma MCP server
> (optional), and all integrations (GitHub, Vercel, Langfuse).
>
> **Stack:** Python 3.11 · FastAPI · Claude Agent SDK · React 18 ·
> TypeScript · Vite · SQLite · ChromaDB · Playwright

---

## Table of Contents

1. [System Requirements](#1-system-requirements)
2. [Clone and Install](#2-clone-and-install)
3. [Environment Variables](#3-environment-variables)
4. [Run the App](#4-run-the-app)
5. [Figma Plugin Setup](#5-figma-plugin-setup)
6. [Figma MCP Server (Optional)](#6-figma-mcp-server-optional)
7. [GitHub Integration (Optional)](#7-github-integration-optional)
8. [Vercel Deployment (Optional)](#8-vercel-deployment-optional)
9. [Langfuse Observability (Optional)](#9-langfuse-observability-optional)
10. [Common Problems](#10-common-problems)

---

## 1. System Requirements

| Tool       | Minimum Version | Notes                                      |
|------------|-----------------|--------------------------------------------|
| Python     | 3.11            | 3.12 also works; 3.10 not tested           |
| Node.js    | 18+             | 20 or 22 LTS recommended; 24 also fine     |
| npm        | 9+              | Comes with Node                            |
| uv         | 0.4+            | Fast Python package manager — see below    |
| Git        | any             |                                            |

### Install uv

`uv` replaces `pip` + `virtualenv` in one command. Required for running the backend.

```bash
# Windows (PowerShell)
powershell -ExecutionPolicy Bypass -c "irm https://astral.sh/uv/install.ps1 | iex"

# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Verify: `uv --version`

---

## 2. Clone and Install

```bash
# 1. Clone
git clone <repo-url>
cd Aura2

# 2. Python dependencies (creates .venv automatically)
uv sync
# If uv sync fails, fallback:
uv pip install -r requirements.txt

# 3. Frontend dependencies
cd frontend && npm install && cd ..

# 4. Root-level Node deps (Playwright runner for verification)
npm install

# 5. Install Playwright Chromium browser
uv run playwright install chromium
```

Verify Playwright works:
```bash
uv run playwright --version
```

---

## 3. Environment Variables

Copy the example file and fill in values:

```bash
cp .env.example .env
```

Then edit `.env`. Every variable below corresponds to a key in that file.

---

### 3.1 Required — Without These, Nothing Works

```dotenv
# Your Anthropic API key (starts with sk-ant-)
# Get it at: https://console.anthropic.com/settings/keys
ANTHROPIC_API_KEY=sk-ant-...

# Your Figma personal access token
# Get it at: Figma → Account Settings → Personal access tokens → Generate new
FIGMA_TOKEN=figd_...
```

Both are **required**. Without `ANTHROPIC_API_KEY` the conversion agent won't run.
Without `FIGMA_TOKEN` the Figma REST API export (for element pixel-diff) won't work —
the rest of the pipeline still functions but pixel comparison is skipped.

---

### 3.2 Optional — LiteLLM Proxy

If you route Claude calls through a LiteLLM proxy (common in enterprise / Samsung
internal setups where the API key is managed centrally):

```dotenv
LITELLM_API_KEY=sk-...           # Proxy auth key
LITELLM_BASE_URL=https://...     # e.g. https://litellm.internal.company.com
LITELLM_PROVIDER=anthropic       # Usually "anthropic"
```

Leave blank to call Anthropic directly.

---

### 3.3 Verification Settings

These control the self-correcting verification loop. Defaults work well — only
tune if you know what you are changing (see
[VERIFICATION_GUIDE.md §8](./VERIFICATION_GUIDE.md#8-tuning-parameters)).

```dotenv
# Enable Claude Vision API as one verification tier (recommended: true)
ENABLE_VISION_COMPARISON=true

# Which Claude model to use for vision comparison
VISION_COMPARISON_MODEL=claude-sonnet-4-6

# Stop when confidence reaches this value (0.0–1.0)
VERIFICATION_CONFIDENCE_THRESHOLD=0.95

# Max fix-and-reverify cycles before giving up
MAX_VERIFICATION_ITERATIONS=10
```

---

### 3.4 Paths (rarely need changing)

```dotenv
# SQLite database location
DATABASE_URL=sqlite+aiosqlite:///./aura2.db

# Where generated React projects are written
GENERATED_PROJECTS_DIR=./generated_projects

# Where reusable component library is stored
COMPONENT_LIBRARY_DIR=./component_library
```

---

### 3.5 GitHub Integration (optional)

```dotenv
GITHUB_PERSONAL_ACCESS_TOKEN=ghp_...   # PAT with repo scope
GITHUB_OWNER=your-org-or-username

# Auto-create a repo for each generated project?
AUTO_CREATE_REPO=false

# Auto-open a PR after generation?
AUTO_CREATE_PR=false
```

---

### 3.6 Vercel Deployment (optional)

```dotenv
VERCEL_TOKEN=...         # Vercel API token
VERCEL_ORG_ID=...        # Team ID (optional, personal accounts leave blank)

# Auto-deploy to Vercel after successful generation?
AUTO_DEPLOY_VERCEL=false
```

---

### 3.7 Figma MCP Server (optional)

Only needed if you enable the MCP path instead of the REST API path.
See [§6](#6-figma-mcp-server-optional) for full setup.

```dotenv
USE_FIGMA_MCP=false                       # Set true to enable
FIGMA_MCP_SERVER_TYPE=remote              # "remote" or "local"
FIGMA_MCP_SERVER_URL=https://mcp.figma.com/mcp   # Remote endpoint
```

---

### 3.8 Langfuse Observability (optional)

```dotenv
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

---

## 4. Run the App

Run **two terminals** — backend and frontend simultaneously.

### Terminal 1 — Backend

```bash
uv run python -m uvicorn backend.main:app --reload --port 8000
```

Wait for:
```
INFO:     Application startup complete.
```

Backend API is at `http://localhost:8000`.
API docs (Swagger): `http://localhost:8000/docs`.

### Terminal 2 — Frontend

```bash
cd frontend
npm run dev
```

Open `http://localhost:5173` in your browser.

---

### One-liner (if you prefer)

```bash
# Backend in background, frontend in foreground
uv run python -m uvicorn backend.main:app --reload --port 8000 &
cd frontend && npm run dev
```

---

## 5. Figma Plugin Setup

The plugin is the recommended way to submit designs. It bypasses Figma REST
API rate limits and sends richer node data.

### 5.1 Load the plugin in Figma desktop

1. Open Figma desktop app (web app does not support local plugins).
2. Go to **Plugins → Development → Import plugin from manifest**.
3. Select `figma-plugin/manifest.json` from the repo root.

### 5.2 Configure the plugin

The plugin needs to know where the backend is running.

Inside `figma-plugin/src/` look for a config or constants file with the
backend URL. Set it to `http://localhost:8000` (or your server address if
running remotely).

Build the plugin after any changes:
```bash
cd figma-plugin
npm install
npm run build
```

### 5.3 Use the plugin

1. Open a Figma file.
2. Run **Plugins → Development → Aura2** (or whatever name you gave it).
3. Click **Convert** — the plugin sends design JSON + images to the backend.
4. Switch to the frontend dashboard (`http://localhost:5173`) to track
   progress.

### 5.4 REST API path (fallback, no plugin needed)

If you can't use the plugin (web-only access, CI, etc.):

```bash
curl -X POST http://localhost:8000/api/projects/from-figma \
  -H "Content-Type: application/json" \
  -d '{
    "figma_url": "https://www.figma.com/design/<file-key>/...",
    "figma_token": "<your token>"
  }'
```

This path is subject to Figma REST API rate limits on large files.

---

## 6. Figma MCP Server (Optional)

The Figma MCP server is an alternative to the REST API. It provides richer
node data and is the path Figma is pushing officially. Most teams don't need
this — the plugin + REST path works fine.

### What MCP adds

- Direct access to Figma file structure via MCP protocol.
- Slightly richer component metadata.
- No rate limit worries on the read side.

### 6.1 Remote MCP server (easiest)

Figma runs a hosted MCP endpoint. No local process needed.

1. Set in `.env`:
   ```dotenv
   USE_FIGMA_MCP=true
   FIGMA_MCP_SERVER_TYPE=remote
   FIGMA_MCP_SERVER_URL=https://mcp.figma.com/mcp
   ```

2. Your `FIGMA_TOKEN` is used to authenticate requests. No extra setup.

3. Restart the backend. Look for `[FigmaMCP] Using remote server` in logs.

### 6.2 Local MCP server

Run the MCP server as a local process (useful for debugging or offline).

```bash
# Install Figma MCP server globally
npm install -g @figma/mcp-server

# Run it
figma-mcp-server --token $FIGMA_TOKEN --port 3001
```

Then set in `.env`:
```dotenv
USE_FIGMA_MCP=true
FIGMA_MCP_SERVER_TYPE=local
FIGMA_MCP_SERVER_URL=http://localhost:3001/mcp
```

### 6.3 Verify MCP is working

After backend restart, check:
```bash
curl http://localhost:8000/api/status | python -m json.tool | grep mcp
```

Should show `"figma_mcp": true`.

### 6.4 MCP vs REST: which to use?

| Situation                               | Recommendation       |
|-----------------------------------------|----------------------|
| New setup, quick test                   | Plugin path (no MCP) |
| Large files, rate limit issues          | Remote MCP           |
| Debugging extraction, need raw protocol | Local MCP            |
| CI / headless / no Figma desktop        | REST API fallback    |

---

## 7. GitHub Integration (Optional)

Auto-creates a GitHub repo and opens a PR with the generated code.

### 7.1 Create a PAT

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic).
2. Scopes needed: `repo` (full).
3. Copy token → set `GITHUB_PERSONAL_ACCESS_TOKEN=ghp_...` in `.env`.

### 7.2 Enable auto-create

```dotenv
GITHUB_OWNER=your-username-or-org
AUTO_CREATE_REPO=true
AUTO_CREATE_PR=true
```

Each generated project will get its own repo named after the project.

---

## 8. Vercel Deployment (Optional)

Auto-deploys the generated React app to Vercel after successful generation.

### 8.1 Get a Vercel token

Vercel dashboard → Settings → Tokens → Create token (full access).

```dotenv
VERCEL_TOKEN=...
VERCEL_ORG_ID=...    # Your team slug, visible in Vercel team settings
AUTO_DEPLOY_VERCEL=true
```

### 8.2 What happens

After generation + verification pass confidence threshold, the pipeline
runs `vercel deploy --prod` in the generated project directory. The
deployment URL appears in the project detail view on the dashboard.

---

## 9. Langfuse Observability (Optional)

Traces every Claude agent call — prompts, token counts, latency, tool use.
Useful for debugging and understanding where the conversion agent spends time.

### 9.1 Sign up

Free tier at `https://cloud.langfuse.com`. Self-host also works.

### 9.2 Get keys

Langfuse → Settings → API Keys → Create new project → copy public + secret.

```dotenv
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

Traces appear in the Langfuse dashboard under "Traces" after the first
conversion run.

---

## 10. Common Problems

### Backend won't start — `ModuleNotFoundError`

```bash
# Re-sync dependencies
uv sync
```

If `uv sync` fails with lock file errors:
```bash
uv pip install -r requirements.txt
```

---

### Playwright can't find Chromium

```
Error: Executable doesn't exist at ...
```

```bash
uv run playwright install chromium
```

---

### Figma plugin says "Cannot connect to backend"

1. Backend running? Check terminal 1 shows `Application startup complete`.
2. Plugin URL correct? Should be `http://localhost:8000`.
3. Firewall blocking localhost? Unlikely but possible on some corporate
   machines.

---

### Vision comparison fails with `AuthenticationError`

`ANTHROPIC_API_KEY` not set or expired.
```bash
# Quick test
uv run python -c "import anthropic; c = anthropic.Anthropic(); print('ok')"
```

---

### Verification loop never reaches confidence threshold

Lower the threshold temporarily to see where it plateaus:
```dotenv
VERIFICATION_CONFIDENCE_THRESHOLD=0.80
```

Check the Discrepancies panel in the dashboard — the stuck properties
point to the root cause. See [VERIFICATION_GUIDE.md §9](./VERIFICATION_GUIDE.md#9-limitations--what-we-do-not-check) for known limits.

---

### ChromaDB error on first run

```
ValueError: ... chroma ... collection
```

The component library is uninitialized. Run one conversion — ChromaDB
creates the collection automatically on first use. Or:
```bash
uv run python -c "from backend.rag.component_store import init_store; import asyncio; asyncio.run(init_store())"
```

---

### `uv` not found after install

Re-open terminal. The installer adds `uv` to `PATH` but the change
only takes effect in a new shell.

---

## See Also

- [VERIFICATION_GUIDE.md](./VERIFICATION_GUIDE.md) — how the 3-tier
  verification works, what scores mean, tuning guide.
- [C4_TEST_PROTOCOL.md](./C4_TEST_PROTOCOL.md) — running against C4
  architecture diagrams.
- [02_AURA2_HOW_WE_SOLVE_IT_NOW.md](./02_AURA2_HOW_WE_SOLVE_IT_NOW.md)
  — full architecture overview.
- [05_VALIDATION_PIPELINE.md](./05_VALIDATION_PIPELINE.md) — deep
  technical reference for the verification pipeline.
