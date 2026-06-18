# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Aura2 is an AI-powered platform that converts Figma designs into React + Tailwind CSS websites. It uses Claude Agent SDK for intelligent code generation and ChromaDB for semantic component reuse across projects.

## Implementation Status

### Fully Implemented (Production-Ready)

| Feature | Status | Details |
|---------|--------|---------|
| **Backend API** | ✅ Complete | FastAPI with all endpoints, background tasks, CORS |
| **React Frontend** | ✅ Complete | 22 components, React Router, React Query, shadcn/ui |
| **Figma Plugin** | ✅ Complete | Full design extraction (~750 LOC), bypasses rate limits |
| **Component Library** | ✅ Complete | ChromaDB + RAG with semantic search |
| **Code Quality Tools** | ✅ Complete | ESLint, Prettier, bundle analysis, TypeScript checks |
| **Dev Server Management** | ✅ Complete | Vite dev servers with auto port allocation (5173-6000) |
| **Accessibility Utils** | ✅ Complete | WCAG contrast, color suggestions, alt text generation |
| **Multi-Page Support** | ✅ Complete | Add pages to existing projects with routing |
| **Visual Verification** | ✅ Complete | Playwright screenshots, text/color/font/layout comparison, auto-fix loop |
| **GitHub Integration** | ✅ Complete | GitHub MCP Server integration for repo/branch/PR creation |
| **Vercel Deployment** | ✅ Complete | Vercel MCP Server for CI/CD and auto-deployment |
| **Testing** | ✅ Complete | 20 tests passing (database, MCP tools, RAG store) |

### Partially Implemented

| Feature | Status | Details |
|---------|--------|---------|
| **Figma REST API** | ⚠️ Partial | Works but has rate limits; plugin path preferred |
| **Figma MCP Server** | ⚠️ Partial | Config exists but optional; plugin path preferred |

### Not Implemented

| Feature | Status | Details |
|---------|--------|---------|
| **User Authentication** | ❌ Missing | No auth system |
| **Pixel-Perfect Diff** | ❌ Missing | Visual verification uses content-based comparison |

## Common Commands

```bash
# Activate virtual environment
.venv\Scripts\activate  # Windows
source .venv/bin/activate  # Linux/Mac

# Run all tests
python -m pytest tests/ -v

# Run a single test file
python -m pytest tests/test_rag_store.py -v

# Run a specific test
python -m pytest tests/test_rag_store.py::test_add_component -v

# Start FastAPI backend (port 8000)
python -m uvicorn backend.main:app --reload --port 8000

# Start React frontend (port 5173)
npm run dev

# Install Python dependencies (using uv)
uv pip install -r requirements.txt

# Install frontend dependencies
npm install

# Build Figma plugin
cd figma-plugin && npm run build
```

## Architecture

### Core Data Flow

1. **Input Sources**:
   - Figma REST API (via `figma_url`) - standard path, subject to rate limits
   - Figma Plugin upload (via `plugin_data`) - bypasses API rate limits, recommended

2. **Conversion Pipeline** (`backend/agents/figma_to_react.py`):
   - `FigmaToReactAgent` orchestrates conversion using Claude Agent SDK
   - Extracts design data (layout, colors, typography, effects)
   - Generates React components with Tailwind CSS
   - Runs visual verification comparing output to Figma design

3. **Component Reuse** (`backend/rag/component_store.py`):
   - ChromaDB-backed vector store for semantic component search
   - Components saved with embeddings enable similarity matching
   - Recommendations: "reuse_directly" (>90%), "adapt" (70-90%), "create_new" (<60%)

### Key Modules

**Backend:**
| Path | Purpose |
|------|---------|
| `backend/main.py` | FastAPI application with all API endpoints |
| `backend/agents/figma_to_react.py` | Main conversion agent using Claude Agent SDK |
| `backend/rag/component_store.py` | ChromaDB component library with semantic search |
| `backend/storage/project_store.py` | JSON-based project persistence |
| `backend/dev_server_manager.py` | Manages Vite dev servers for previewing projects |
| `backend/config.py` | Pydantic settings loaded from `.env` |
| `backend/utils/visual_comparison.py` | Screenshot comparison for verification |
| `backend/utils/code_quality.py` | ESLint/Prettier integration |
| `backend/utils/accessibility.py` | WCAG contrast, color suggestions, alt text |
| `backend/utils/git_manager.py` | Git branch/commit/PR utilities |

**Frontend** (`src/`):
| Path | Purpose |
|------|---------|
| `src/App.tsx` | Main app with React Router |
| `src/pages/` | Dashboard, ProjectDetail, ComponentLibrary pages |
| `src/components/` | UI components (ProjectCard, StatusBadge, forms) |
| `src/api/client.ts` | Axios HTTP client for backend API |
| `src/hooks/` | React Query hooks (useProjects, useComponents, useStats) |

**Figma Plugin** (`figma-plugin/`):
| Path | Purpose |
|------|---------|
| `figma-plugin/src/code.ts` | Plugin logic (~750 LOC) - extracts design data |
| `figma-plugin/src/ui.html` | Plugin UI for project selection |

### API Endpoints

**Project Management:**
- `POST /api/projects/create` - Create project from Figma URL or plugin data
- `POST /api/figma/plugin-upload` - Direct endpoint for Figma plugin uploads
- `POST /api/projects/add-website` - Add website with component reuse
- `GET /api/projects` - List all projects
- `GET /api/projects/available` - List successful projects (for multi-page dropdown)
- `GET /api/projects/{id}/status` - Get project status
- `DELETE /api/projects/{id}` - Delete a project

**Dev Server:**
- `GET /api/projects/{id}/preview-url` - Get/start dev server preview
- `POST /api/projects/{id}/start-dev-server` - Start Vite dev server
- `POST /api/projects/{id}/stop-dev-server` - Stop dev server
- `POST /api/projects/{id}/build` - Run npm build

**Component Library:**
- `GET /api/components` - List all components (optional `category` filter)
- `GET /api/stats` - Platform statistics (projects, components, reuse count)

### Storage

- **Projects**: JSON file at `./data/projects.json` (thread-safe, auto-persisted)
- **Components**: ChromaDB at `./component_library/chroma/` with code files in `./component_library/chroma/codes/`
- **Generated Projects**: Output to `./generated_projects/{project_name}/`

## Environment Variables

Copy `.env.example` to `.env` and configure:

```bash
# Core APIs
FIGMA_TOKEN=...              # Figma personal access token
LITELLM_API_KEY=...          # LiteLLM proxy API key
LITELLM_BASE_URL=  # LiteLLM proxy URL

# GitHub Integration (for CI/CD)
GITHUB_PERSONAL_ACCESS_TOKEN=...  # GitHub personal access token
GITHUB_OWNER=your-username        # GitHub username or organization
AUTO_CREATE_REPO=true             # Auto-create repo on project generation
AUTO_CREATE_PR=true               # Auto-create PR after generation

# Vercel Deployment (for CI/CD)
VERCEL_TOKEN=...                  # Vercel API token
VERCEL_ORG_ID=...                 # Vercel organization ID (optional)
AUTO_DEPLOY_VERCEL=true           # Auto-deploy to Vercel
```

The system uses LiteLLM proxy for all LLM calls, auto-configuring `ANTHROPIC_BASE_URL` and `ANTHROPIC_API_KEY`.

## Testing

Tests use pytest with async support. Key fixtures in `tests/conftest.py` configure Windows event loop policy.

```bash
# Run with verbose output
python -m pytest tests/ -v

# Run specific test categories
python -m pytest tests/test_rag_store.py -v      # Component store tests
python -m pytest tests/test_mcp_tools.py -v      # MCP tools tests
python -m pytest tests/test_database.py -v       # Database model tests
```

## Multi-Page Projects

Projects support adding pages to existing projects:
- `add_as="new_project"` - Standalone project
- `add_as="new_page"` with `parent_project_id` - Page within existing project

Pages get auto-generated route paths and are tracked via `is_page` and `route_path` fields.

## PDF Requirements Checklist (6.40.3 Spec)

### Core Pipeline (All Complete)
- [x] Design Extractor - Figma API + Plugin extraction
- [x] AI Conversion Engine - Claude Agent SDK for React/TS generation
- [x] Code Formatter & Optimizer - ESLint, Prettier, TypeScript
- [x] Component Library - ChromaDB + RAG semantic search
- [x] Deployment Pipeline - GitHub + Vercel auto-deploy
- [x] Visual Verification - Playwright screenshots + comparison
- [x] Accessibility - WCAG contrast, ARIA, alt text
- [x] Multi-Page Support - Add pages to existing projects

### Future Enhancements (From PDF)
- [ ] Bi-directional Sync (Code ↔ Figma)
- [ ] Animations & Micro-interactions
- [ ] Multi-framework Support (Vue, Angular, React Native)
- [ ] AI-based Code Review
- [ ] Feature Flag Integration (A/B testing)
- [ ] Pixel-Perfect Visual Diff
- [ ] Auto-generated Test Stubs (Jest/RTL)
- [ ] Dark/Light Mode Auto-theming

### USP Focus
1. **Component Reuse** - RAG-based semantic matching for enterprise-scale reuse
2. **Add Pages Easily** - Extend projects without rebuilding from scratch
3. **Design Governance** - Consistent UI across all generated pages

## Distinctive Dashboard Design

### Location
- **Component**: `frontend/src/components/distinctive/DistinctiveDashboard.tsx`
- **Styles**: `frontend/src/styles/distinctive-aesthetic.css`
- **Integration**: Dashboard.tsx uses the distinctive design

### Setup
```bash
cd frontend
npm install framer-motion  # Required dependency
npm run dev
```

### Features
- **Gradient Mesh Background**: 4 animated floating gradient blobs
- **Magnetic Buttons**: Buttons with cursor attraction effect
- **3D Floating Cards**: Cards with depth, rotation, and hover effects
- **Bento Grid Layout**: Asymmetric grid (1 large + 3 small stats)
- **Bold Typography**: Large hero text with gradient accents
- **Vibrant Colors**: Violet (#8b5cf6), Cyan (#06b6d4), Emerald (#10b981)
- **Status Indicators**: Animated badges (spinning for "processing")

### Design System
```css
/* Colors */
--color-violet: #8b5cf6   /* Main accent */
--color-cyan: #06b6d4     /* Secondary */
--color-emerald: #10b981  /* Success */
--color-amber: #f59e0b    /* Warning */

/* Typography */
--font-primary: 'Space Grotesk'  /* Display font */
--font-mono: 'JetBrains Mono'    /* Code font */
```

### Key Differences from Old Design
| Old | Distinctive |
|-----|-------------|
| Regular grid | Bento box layout |
| Static cards | 3D floating with depth |
| Simple buttons | Magnetic attraction |
| Plain background | Animated gradient mesh |
| Muted colors | Vibrant gradients |
| Standard typography | Bold dramatic sizes |

### Connecting Real Data
Replace mock data in `DistinctiveDashboard.tsx`:
```tsx
import { useProjects } from '@/hooks/useProjects';
import { useStats } from '@/hooks/useStats';

export default function DistinctiveDashboard() {
  const { data: projectsData } = useProjects();
  const { data: statsData } = useStats();

  const projects = projectsData?.projects || [];
  // Use real data in JSX
}
```

### Customization
- **Colors**: Edit `distinctive-aesthetic.css` (line 15-20)
- **Animations**: Adjust `duration` in transition props
- **Fonts**: Update `@import` and `--font-primary` in CSS
