# Aura2 — Complete Architecture Diagrams

> **Project:** Aura2 — AI-Powered Figma → React Converter
> **Stack:** FastAPI · Claude Agent SDK (Opus 4.6) · React 18 · TypeScript · ChromaDB · SQLite
> **Context:** Samsung PRISM @ IIIT Naya Raipur

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Tech Stack Map](#2-tech-stack-map)
3. [End-to-End Data Flow](#3-end-to-end-data-flow)
4. [Conversion Pipeline — 5 Steps](#4-conversion-pipeline--5-steps)
5. [Backend Architecture](#5-backend-architecture)
6. [Frontend Architecture](#6-frontend-architecture)
7. [MCP Server Integration](#7-mcp-server-integration)
8. [Database & Storage Schema](#8-database--storage-schema)
9. [API Endpoints Map](#9-api-endpoints-map)
10. [RAG Component Reuse System](#10-rag-component-reuse-system)
11. [Visual Verification Loop](#11-visual-verification-loop)
12. [Deployment Pipeline](#12-deployment-pipeline)
13. [Component Relationships (Class Diagram)](#13-component-relationships-class-diagram)
14. [Full Conversion Sequence Diagram](#14-full-conversion-sequence-diagram)
15. [Performance Benchmarks](#15-performance-benchmarks)
16. [Environment Configuration Map](#16-environment-configuration-map)

---

## 1. System Overview

```mermaid
graph TB
    subgraph INPUT["🎨 INPUT SOURCES"]
        FU[Figma URL]
        FP[Figma Plugin<br/>Direct Upload]
    end

    subgraph FRONTEND["🖥️ FRONTEND<br/>React 18 · TypeScript · Vite · Tailwind"]
        DASH[Dashboard Page]
        PD[Project Detail Page]
        CL[Components Library Page]
    end

    subgraph BACKEND["⚙️ BACKEND<br/>FastAPI · Python 3.11 · Uvicorn"]
        API[REST API<br/>22 endpoints]
        AGENT[FigmaToReactAgent<br/>Claude Agent SDK Orchestrator]
        CONFIG[Config Manager<br/>Pydantic Settings]
        DSM[Dev Server Manager<br/>Vite Lifecycle]
    end

    subgraph AI_LAYER["🤖 AI LAYER"]
        CLAUDE[Claude Opus 4.6<br/>Code Generation<br/>via LiteLLM Proxy]
        VISION[Claude Vision Sonnet 4<br/>Visual Verification]
        EMB[sentence-transformers<br/>all-MiniLM-L6-v2<br/>Component Embeddings]
    end

    subgraph STORAGE["💾 STORAGE LAYER"]
        SQLITE[(SQLite<br/>aura2.db<br/>Projects · Components · Usage)]
        JSON[(JSON File Store<br/>data/projects.json<br/>In-memory cache)]
        CHROMA[(ChromaDB<br/>component_library/chroma/<br/>Vector embeddings)]
    end

    subgraph OUTPUT["📦 OUTPUT"]
        REACT[Generated React Project<br/>Tailwind · MUI · Chakra]
        GH[GitHub Repository<br/>+ Pull Request]
        VERCEL[Vercel Deployment<br/>Live URL]
        PREVIEW[Local Dev Server<br/>localhost:5173+]
    end

    subgraph MCP["🔌 MCP SERVERS<br/>.mcp.json"]
        SHADCN_MCP[shadcn MCP<br/>npx shadcn@latest mcp<br/>Component Registry]
        DRAWIO_MCP[drawio MCP<br/>npx drawio-mcp-server<br/>Diagram Tools]
    end

    FU --> FRONTEND
    FP --> FRONTEND
    FRONTEND -- "Axios HTTP" --> API
    API --> AGENT
    AGENT --> CLAUDE
    AGENT --> VISION
    AGENT --> EMB
    AGENT --> SQLITE
    AGENT --> JSON
    AGENT --> CHROMA
    AGENT --> REACT
    REACT --> GH
    REACT --> VERCEL
    REACT --> PREVIEW
    BACKEND -. "MCP Protocol<br/>stdio transport" .-> MCP
    CONFIG --> BACKEND
    DSM --> PREVIEW

    style INPUT fill:#4f46e5,color:#fff
    style FRONTEND fill:#0891b2,color:#fff
    style BACKEND fill:#059669,color:#fff
    style AI_LAYER fill:#d97706,color:#fff
    style STORAGE fill:#7c3aed,color:#fff
    style OUTPUT fill:#dc2626,color:#fff
    style MCP fill:#475569,color:#fff
```

---

## 2. Tech Stack Map

```mermaid
mindmap
  root((Aura2<br/>Tech Stack))
    Frontend
      React 18.2
      TypeScript 5.2
      Vite 5.0.8
        HMR
        Fast Build
      Tailwind CSS 3.3.6
      shadcn/ui
        Primitives
        Radix UI base
      TanStack Query 5.17
        Server state
        Polling
      React Router 6.20
      Framer Motion 11.18
        Animations
      Axios 1.6
      Lucide Icons 0.294
    Backend
      FastAPI 0.109
      Python 3.11+
      Uvicorn
        ASGI server
      Pydantic v2.5
        Data validation
        Settings
      SQLAlchemy async
        ORM
      aiosqlite
        Async SQLite
      aiofiles
        Async file I/O
      httpx 0.26
      anyio 4.2
    AI & ML
      Claude Agent SDK 0.1
        Tool orchestration
        max_turns=35
      Anthropic API 0.40
        Claude Opus 4.6
          Code generation
          Design analysis
          Prompt: ~8k tokens
        Claude Sonnet 4
          Vision comparison
          Screenshot diff
      LiteLLM Proxy
        API key routing
        Model aliasing
      sentence-transformers 2.2
        all-MiniLM-L6-v2
        384-dim vectors
    Storage
      SQLite
        projects table
        components table
        component_usage table
      ChromaDB 1.4
        Persistent store
        Cosine similarity
        Threshold 60%
      JSON File Store
        projects.json
        Thread-safe Lock
        In-memory cache
    Browser Automation
      Playwright 1.57
        Full-page screenshots
        Dev server testing
        Headless Chromium
      Pillow 10
        Image processing
        Diff generation
    Code Quality
      ESLint
        TS rules
        React rules
      Prettier
        Auto-format
      npm CLI
        install
        run build
        run dev
    MCP Servers
      shadcn MCP
        list_items
        search_items
        get_add_command
        view_items
      drawio MCP
        create_diagram
        edit_diagram
    External APIs
      Figma REST API
        GET /files/{key}
        GET /images/{key}
        Rate limiting handled
      GitHub API
        Repo creation
        Branch push
        PR creation
      Vercel API
        v13/deployments
        Auto-deploy
        Live URL
    Templates
      react-tailwind
        Primary template
        Vite + Tailwind
      react-mui
        Material-UI
      react-chakra
        Chakra UI
```

---

## 3. End-to-End Data Flow

```mermaid
flowchart LR
    A([👤 User]) --> B{Input Type}

    B -- "Figma URL" --> C[figma_api.py<br/>Fetch design tree<br/>from REST API]
    B -- "Plugin Upload" --> D[plugin_conversion.py<br/>Direct plugin data<br/>bypasses rate limits]

    C --> E[figma_extraction.py<br/>Parse node hierarchy<br/>frames · groups · text]
    D --> E

    E --> F[design_styles.py<br/>Colors · Fonts<br/>Effects · Shadows]
    E --> G[responsive_extraction.py<br/>Breakpoints<br/>Responsive rules]

    F --> H[Normalized<br/>Design JSON]
    G --> H

    H --> I{RAG Search<br/>component_store.py<br/>ChromaDB}
    I -- "similarity ≥ 60%" --> J[Reuse existing<br/>component code]
    I -- "no match" --> K[Flag for new<br/>generation]

    J --> L[prompt_generation.py<br/>Build Claude prompt<br/>with design context<br/>+ reusable components]
    K --> L

    L --> M[Claude Opus 4.6<br/>Agent SDK<br/>max_agent_turns=35]
    M --> N[React + TypeScript<br/>+ Tailwind CSS code]

    N --> O[semantic_analysis.py<br/>ARIA attributes<br/>AST analysis]
    O --> P[accessibility.py<br/>WCAG 2.1 AA<br/>contrast · alt text]

    P --> Q[project_setup.py<br/>Scaffold from template<br/>react-tailwind / mui / chakra]

    Q --> R[Visual Verification Loop<br/>verification.py]
    R --> S[Playwright<br/>Capture screenshot]
    S --> T[vision_comparison.py<br/>Claude Vision API<br/>diff analysis]
    T --> U{confidence<br/>≥ 98%?}
    U -- "No<br/>max 10 iters" --> V[auto_fix_agent.py<br/>Generate targeted fixes<br/>Claude Opus]
    V --> W[fix_applicator.py<br/>Apply code patches]
    W --> R
    U -- "Yes" --> X[build_verifier.py<br/>npm run build<br/>TypeScript check]

    X --> Y[code_quality.py<br/>ESLint + Prettier<br/>auto-format]
    Y --> Z[component_store.py<br/>Store new components<br/>in ChromaDB]
    Z --> AA[project_store.py<br/>Update status: success<br/>Save to JSON + SQLite]

    AA --> AB{Deploy?}
    AB -- "GitHub" --> AC[git_manager.py<br/>Create repo + push<br/>+ optional PR]
    AB -- "Vercel" --> AD[Vercel API<br/>auto-deploy<br/>get live URL]
    AB -- "Local" --> AE[dev_server_manager.py<br/>localhost:5173+]

    AC --> AF([✅ Done])
    AD --> AF
    AE --> AF

    style A fill:#4f46e5,color:#fff
    style M fill:#d97706,color:#fff
    style T fill:#d97706,color:#fff
    style AF fill:#059669,color:#fff
```

---

## 4. Conversion Pipeline — 5 Steps

```mermaid
flowchart TD
    START([▶ FigmaToReactAgent.convert]) --> S1

    subgraph S1["📐 STEP 1 — Design Extraction"]
        S1A[figma_api.py<br/>Fetch Figma file tree<br/>via REST API token]
        S1B[figma_extraction.py<br/>Parse node hierarchy<br/>frames · groups · components · text]
        S1C[design_styles.py<br/>Extract: colors · fonts<br/>effects · box-shadows · opacity]
        S1D[responsive_extraction.py<br/>Extract breakpoints<br/>& responsive layout rules]
        S1E[figma_screenshot.py<br/>Download design screenshots<br/>for visual reference]
        S1A --> S1B --> S1C --> S1D --> S1E
    end

    subgraph S2["🔍 STEP 2 — RAG Component Reuse"]
        S2A[sentence-transformers<br/>all-MiniLM-L6-v2<br/>Generate 384-dim query embedding]
        S2B[ChromaDB cosine search<br/>collection: react_components<br/>Top-K neighbors]
        S2C{Similarity<br/>threshold?}
        S2D["reuse_directly<br/>> 90% match<br/>Return as-is"]
        S2E["adapt<br/>70–90% match<br/>Modify for context"]
        S2F["create_new<br/>< 60% match<br/>Generate fresh"]
        S2A --> S2B --> S2C
        S2C -- "> 90%" --> S2D
        S2C -- "70–90%" --> S2E
        S2C -- "< 60%" --> S2F
    end

    subgraph S3["🤖 STEP 3 — Claude Code Generation"]
        S3A[prompt_generation.py<br/>Build: system prompt<br/>+ design JSON<br/>+ reusable component context]
        S3B[Claude Agent SDK<br/>claude-opus-4-6<br/>max_turns=35<br/>max_fix_turns=15]
        S3C[semantic_analysis.py<br/>AST analysis<br/>Add ARIA roles · labels<br/>Semantic HTML mapping]
        S3D[accessibility.py<br/>WCAG 2.1 AA compliance<br/>Contrast ratios · alt text<br/>Keyboard navigation]
        S3A --> S3B --> S3C --> S3D
    end

    subgraph S4["👁️ STEP 4 — Visual Verification Loop"]
        S4A[project_setup.py<br/>Scaffold project from template<br/>npm install]
        S4B[dev_server_manager.py<br/>Start Vite: localhost:5173<br/>Wait for health check]
        S4C[Playwright<br/>Full-page screenshot<br/>headless Chromium]
        S4D[vision_comparison.py<br/>Claude Vision Sonnet 4<br/>Compare Figma vs Generated<br/>Output: confidence + discrepancies]
        S4E{Confidence<br/>≥ 98%?}
        S4F[auto_fix_agent.py<br/>Claude Opus 4.6<br/>Generate targeted code fixes<br/>for each discrepancy]
        S4G[fix_applicator.py<br/>Apply patches to<br/>component source files]
        S4H[fix_tracker.py<br/>Track fix history<br/>Prevent fix loops]
        S4I{Iter ≥ 10?}
        S4A --> S4B --> S4C --> S4D --> S4E
        S4E -- "No" --> S4F --> S4G --> S4H --> S4I
        S4I -- "No" --> S4C
        S4I -- "Yes, force stop" --> NEXT4
        S4E -- "Yes ✅" --> NEXT4
    end

    subgraph S5["🏗️ STEP 5 — Build & Quality Check"]
        S5A[build_verifier.py<br/>npm run build<br/>TypeScript compile check]
        S5B[code_quality.py<br/>ESLint auto-fix<br/>Prettier format]
        S5C[component_store.py<br/>Embed + store new components<br/>in ChromaDB for future reuse]
        S5D[project_store.py<br/>Update project: status=success<br/>Save metrics to JSON + SQLite]
        S5A --> S5B --> S5C --> S5D
    end

    S1D --> S2A
    S2D & S2E & S2F --> S3A
    S3D --> S4A
    NEXT4[→] --> S5A
    S5D --> DONE([✅ Project Ready<br/>generated_projects/])

    style S1 fill:#dbeafe,stroke:#3b82f6,color:#1e40af
    style S2 fill:#fef3c7,stroke:#f59e0b,color:#78350f
    style S3 fill:#dcfce7,stroke:#22c55e,color:#14532d
    style S4 fill:#fce7f3,stroke:#ec4899,color:#831843
    style S5 fill:#ede9fe,stroke:#8b5cf6,color:#4c1d95
    style START fill:#1e293b,color:#fff
    style DONE fill:#059669,color:#fff
```

---

## 5. Backend Architecture

```mermaid
graph TB
    subgraph ENTRY["🚪 Entry — main.py (FastAPI 0.109)"]
        MAIN["FastAPI App<br/>22 REST endpoints<br/>CORS · Background Tasks<br/>Static file mount: /projects"]
    end

    subgraph CFG["⚙️ Config"]
        CONFIG["config.py<br/>Settings (Pydantic BaseSettings)<br/>Reads from .env<br/>All tokens + thresholds"]
        DSM["dev_server_manager.py<br/>Vite dev server lifecycle<br/>Port allocation: 5173–6000<br/>Process tracking"]
    end

    subgraph AGENTS["🤖 agents/"]
        FTR["figma_to_react.py<br/>FigmaToReactAgent<br/>Main orchestrator<br/>Claude Agent SDK"]
        CA["component_analyzer.py<br/>Component analysis utilities"]

        subgraph AGENT_SUB["agents/_figma_to_react/ — Pipeline Steps"]
            FA["figma_api.py<br/>Figma REST API client<br/>File fetch · Image export"]
            FE["figma_extraction.py<br/>Node tree parser<br/>Hierarchy traversal"]
            DS["design_styles.py<br/>Style extractor<br/>Colors · Fonts · Effects"]
            RE["responsive_extraction.py<br/>Breakpoint extractor<br/>Responsive rules"]
            PC["plugin_conversion.py<br/>Plugin data → Design JSON<br/>Bypasses Figma rate limits"]
            PS["project_setup.py<br/>Template → Project scaffold<br/>react-tailwind / mui / chakra"]
            PG["prompt_generation.py<br/>Claude prompt builder<br/>System + User + Context"]
            SA["semantic_analysis.py<br/>AST-based semantic mapping<br/>ARIA role assignment"]
            VE["verification.py<br/>Verification loop controller<br/>max_iterations=10"]
        end
    end

    subgraph UTILS["🔧 utils/ — 15 Utility Modules"]
        AC["accessibility.py<br/>WCAG 2.1 AA compliance<br/>Contrast · Alt text · ARIA"]
        AF["auto_fix_agent.py<br/>AI-powered fix generator<br/>Claude Opus · targeted patches"]
        BV["build_verifier.py<br/>npm install + build<br/>TypeScript validation"]
        CQ["code_quality.py<br/>ESLint + Prettier<br/>Auto-fix + format"]
        CM["component_matcher.py<br/>Similarity matching<br/>Threshold scoring"]
        CL_LOG["conversion_logger.py<br/>Progress logging<br/>Step tracking"]
        FRL["figma_rate_limiter.py<br/>Figma API rate limiting<br/>Request throttling"]
        FSS["figma_screenshot.py<br/>Figma design screenshots<br/>Node image export"]
        FIX["fix_applicator.py<br/>Code patch applicator<br/>File-level edits"]
        FT["fix_tracker.py<br/>Fix state tracker<br/>Prevent fix loops"]
        GM["git_manager.py<br/>GitHub integration<br/>Repo · Branch · PR"]
        SM["semantic_mapping.py<br/>AST-based mapping<br/>Component type inference"]
        VC["vision_comparison.py<br/>Claude Vision API<br/>Screenshot diff analysis"]
        VISC["visual_comparison.py<br/>Image diff utilities<br/>Pillow processing"]
    end

    subgraph DATABASE["💾 database/"]
        DB["db.py<br/>SQLAlchemy async engine<br/>Session factory<br/>aiosqlite driver"]
        MDL["models.py<br/>ORM Models:<br/>Project · Component · ComponentUsage"]
    end

    subgraph STORAGE_MOD["📦 storage/"]
        PS_STORE["project_store.py<br/>ProjectStore class<br/>JSON + In-memory HashMap<br/>Thread-safe asyncio.Lock<br/>CRUD operations"]
    end

    subgraph RAG_MOD["🔍 rag/"]
        CS["component_store.py<br/>ChromaDB persistent store<br/>sentence-transformers embeddings<br/>collection: react_components<br/>cosine similarity search"]
    end

    subgraph MCP_MOD["🔌 mcp_tools/"]
        CL_MCP["component_library.py<br/>MCP Server<br/>Exposed tools:<br/>search_components<br/>get_component<br/>list_categories"]
        CLR["component_library_rag.py<br/>RAG-backed MCP search<br/>Semantic component retrieval"]
    end

    MAIN --> FTR
    MAIN --> PS_STORE
    MAIN --> CS
    MAIN --> DB
    MAIN --> DSM
    FTR --> FA & FE & DS & RE & PC & PS & PG & SA & VE
    FTR --> AF & BV & CQ & CM & GM & VC
    VE --> VC & FSS & FIX & FT
    AF --> FIX & FT
    CS --> CM
    CL_MCP --> CS
    CLR --> CS
    CONFIG --> MAIN & FTR & DSM

    style ENTRY fill:#1e40af,color:#fff
    style CFG fill:#374151,color:#fff
    style AGENTS fill:#065f46,color:#fff
    style AGENT_SUB fill:#d1fae5,stroke:#059669,color:#064e3b
    style UTILS fill:#78350f,color:#fff
    style DATABASE fill:#581c87,color:#fff
    style STORAGE_MOD fill:#164e63,color:#fff
    style RAG_MOD fill:#713f12,color:#fff
    style MCP_MOD fill:#1e3a5f,color:#fff
```

---

## 6. Frontend Architecture

```mermaid
graph TB
    subgraph ENTRY["🚪 Entry"]
        MAIN_TSX["main.tsx<br/>ReactDOM.createRoot<br/>QueryClientProvider<br/>BrowserRouter"]
        APP["App.tsx<br/>React Router v6<br/>Route definitions"]
    end

    subgraph PAGES["📄 pages/"]
        DASH_P["Dashboard.tsx<br/>Main dashboard<br/>Project list + stats<br/>Create project form"]
        PDD["ProjectDetailDistinctive.tsx<br/>Project detail view<br/>Build controls<br/>Deploy actions<br/>Dev server toggle"]
        COMP_P["Components.tsx<br/>Component library page<br/>Filtered view"]
    end

    subgraph COMPONENTS["🧩 components/"]
        NAV["Navigation.tsx<br/>Top nav bar<br/>Route links"]
        HERO["Hero.tsx<br/>Landing hero section<br/>Gradient animations"]
        FEAT["Features.tsx<br/>Feature highlights<br/>Icon grid"]
        PF["ProjectForm.tsx<br/>Create project form<br/>Figma URL input<br/>UI library select<br/>Plugin data support"]
        PC_C["ProjectCard.tsx<br/>Project tile<br/>Status · time · metrics<br/>Quick actions"]
        SP["StatsPanel.tsx<br/>Aggregated statistics<br/>Projects · Components<br/>Reuse rate"]
        PP["ProjectPreview.tsx<br/>Iframe preview<br/>Dev server URL"]
        DC["DeploymentCard.tsx<br/>GitHub + Vercel status<br/>Deploy buttons<br/>Live URL display"]
        SB["StatusBadge.tsx<br/>Animated status pill<br/>pending · generating<br/>success · failed"]
        CL_COMP["ComponentLibrary.tsx<br/>Searchable component grid<br/>Category filters<br/>Code preview"]

        subgraph UI["components/ui/ — shadcn/ui Primitives"]
            BTN["button.tsx<br/>Variants: default<br/>destructive · outline<br/>ghost · link"]
            CARD["card.tsx<br/>Card · Header · Content<br/>Footer · Description"]
            DLG["dialog.tsx<br/>Modal dialogs<br/>Radix UI portal"]
            INP["input.tsx<br/>Styled text input"]
            LBL["label.tsx<br/>Form label"]
            PRG["progress.tsx<br/>Progress bar<br/>Animated fill"]
            SEL["select.tsx<br/>Dropdown select<br/>Radix UI"]
            TABS["tabs.tsx<br/>Tab group<br/>Radix UI"]
            BADGE["badge.tsx<br/>Status badge pill"]
            TOAST["toast.tsx<br/>Notification toast<br/>Radix UI"]
            BB["BorderBeam.tsx<br/>✨ Animated gradient<br/>border effect"]
            GC["GlassCard.tsx<br/>🪟 Glass morphism<br/>backdrop-blur card"]
            SHIM["ShimmerButton.tsx<br/>✨ Shimmer hover<br/>button animation"]
        end

        subgraph PAPERCUT["components/papercut/"]
            PCD["PaperCutDashboard.tsx<br/>PaperCut design system<br/>Alternative dashboard UI"]
        end
    end

    subgraph HOOKS["🪝 hooks/ — React Query"]
        UP["useProjects.ts<br/>GET /api/projects<br/>Refetch interval: 5s"]
        UPS["useProjectStatus.ts<br/>GET /api/projects/{id}/status<br/>Refetch: 2s while active"]
        UC["useComponents.ts<br/>GET /api/components<br/>Category filter support"]
        US["useStats.ts<br/>GET /api/stats<br/>System metrics"]
    end

    subgraph API_CLIENT["🌐 api/"]
        AC["client.ts<br/>Axios instance<br/>baseURL: VITE_API_URL<br/>Default: localhost:8000"]
    end

    subgraph TYPES["📋 types/index.ts — TypeScript Interfaces"]
        TI["Project<br/>Component<br/>Stats<br/>DeploymentStatus<br/>ProjectStatus enum<br/>ProjectCreateRequest<br/>DeploymentStatusResponse"]
    end

    subgraph UTILS_FE["🛠️ utils/ + lib/"]
        CONST["utils/constants.ts<br/>API_BASE_URL<br/>ROUTE_PATHS"]
        LIB["lib/utils.ts<br/>cn() = clsx + tailwind-merge<br/>Class name helper"]
    end

    MAIN_TSX --> APP
    APP -- "/ (index)" --> DASH_P
    APP -- "/projects/:id" --> PDD
    APP -- "/components" --> COMP_P

    DASH_P --> NAV & HERO & FEAT & PF & PC_C & SP
    PDD --> DC & PP & SB
    COMP_P --> CL_COMP

    DASH_P --> UP & US
    PDD --> UPS
    COMP_P --> UC

    UP & UPS & UC & US --> AC
    AC --> TI

    style ENTRY fill:#1e40af,color:#fff
    style PAGES fill:#065f46,color:#fff
    style COMPONENTS fill:#78350f,color:#fff
    style UI fill:#fef3c7,stroke:#f59e0b,color:#78350f
    style PAPERCUT fill:#fce7f3,stroke:#ec4899,color:#831843
    style HOOKS fill:#581c87,color:#fff
    style API_CLIENT fill:#164e63,color:#fff
    style TYPES fill:#374151,color:#fff
    style UTILS_FE fill:#1e3a5f,color:#fff
```

---

## 7. MCP Server Integration

```mermaid
graph LR
    subgraph CLAUDE_CODE["Claude Code Session<br/>claude-sonnet-4-6"]
        CC[Claude Code<br/>AI Assistant]
    end

    subgraph LOCAL_MCP["Local MCP Servers<br/>.mcp.json config"]
        subgraph SHADCN["shadcn MCP"]
            S_CMD["npx -y shadcn@latest mcp"]
            S_TOOLS["Tools:<br/>• list_items_in_registries<br/>• search_items_in_registries<br/>• view_items_in_registries<br/>• get_add_command_for_items<br/>• get_item_examples_from_registries<br/>• get_project_registries<br/>• get_audit_checklist"]
        end
        subgraph DRAWIO["drawio MCP"]
            D_CMD["npx -y drawio-mcp-server"]
            D_TOOLS["Tools:<br/>• create diagrams<br/>• edit diagrams<br/>• export SVG/PNG/XML"]
        end
    end

    subgraph EXTERNAL_MCP["External MCP Servers (Claude AI Platform)"]
        subgraph PLAY["Playwright MCP"]
            PW_T["~20 tools<br/>browser_navigate<br/>browser_screenshot<br/>browser_click<br/>browser_fill_form<br/>browser_evaluate"]
        end
        subgraph CTX7["Context7 MCP"]
            C7_T["2 tools<br/>resolve-library-id<br/>query-docs<br/>Up-to-date library docs"]
        end
        subgraph SEQ["Sequential Thinking MCP"]
            ST_T["1 tool<br/>sequentialthinking<br/>Step-by-step reasoning"]
        end
        subgraph LIN["Linear MCP"]
            LIN_T["~40 tools<br/>list_issues · save_issue<br/>list_projects · save_project<br/>list_teams · get_user<br/>Project management"]
        end
        subgraph HR["HeyReach MCP"]
            HR_T["~35 tools<br/>add_leads_to_campaign<br/>send_message · get_lead<br/>LinkedIn outreach"]
        end
        subgraph LF["Langfuse MCP"]
            LF_T["3 tools<br/>getLangfuseOverview<br/>getLangfuseDocsPage<br/>searchLangfuseDocs<br/>LLM observability"]
        end
        subgraph LC["Langchain Docs MCP"]
            LC_T["1 tool<br/>search_docs_by_lang_chain<br/>LangChain documentation"]
        end
    end

    subgraph APP_MCP["Internal App MCP Tools<br/>backend/mcp_tools/"]
        subgraph CL_TOOL["component_library.py"]
            CL_T["MCP Server (internal)<br/>Tools exposed to Agent:<br/>• search_components(query)<br/>• get_component(id)<br/>• list_categories()<br/>• add_component(name, code)"]
        end
        subgraph CLR_TOOL["component_library_rag.py"]
            CLR_T["RAG-backed search<br/>Uses ChromaDB<br/>sentence-transformers<br/>Similarity threshold 60%"]
        end
    end

    CC -- "stdio transport<br/>JSON-RPC 2.0" --> SHADCN
    CC -- "stdio transport" --> DRAWIO
    CC -- "HTTP/SSE" --> PLAY
    CC -- "stdio" --> CTX7
    CC -- "stdio" --> SEQ
    CC -- "HTTP/SSE" --> LIN
    CC -- "HTTP/SSE" --> HR
    CC -- "HTTP" --> LF
    CC -- "stdio" --> LC

    subgraph AGENT_BOX["FigmaToReactAgent (Claude Agent SDK)"]
        AGENT_INNER["Claude Agent<br/>Orchestrates tools"]
    end

    AGENT_INNER --> CL_TOOL
    AGENT_INNER --> CLR_TOOL

    style CLAUDE_CODE fill:#d97706,color:#fff
    style LOCAL_MCP fill:#1e40af,color:#fff
    style EXTERNAL_MCP fill:#581c87,color:#fff
    style APP_MCP fill:#065f46,color:#fff
    style AGENT_BOX fill:#374151,color:#fff
    style SHADCN fill:#1e293b,color:#fff
    style DRAWIO fill:#1e293b,color:#fff
```

---

## 8. Database & Storage Schema

### Entity Relationship Diagram

```mermaid
erDiagram
    PROJECTS {
        INTEGER id PK "auto-increment"
        VARCHAR name UK "unique project name"
        TEXT figma_url "Figma design URL"
        VARCHAR figma_file_key "Figma file key"
        DATETIME created_at
        DATETIME updated_at
        TEXT project_path "generated_projects/name/"
        VARCHAR status "pending|generating|success|failed"
        TEXT error_message
        INTEGER components_generated
        INTEGER components_reused
        FLOAT conversion_time_seconds
        INTEGER parent_project_id FK "self-ref for pages"
        BOOLEAN is_page
        VARCHAR route_path "e.g. /about"
        BOOLEAN visual_match
        INTEGER verification_iterations
        FLOAT verification_confidence "0.0–1.0"
        JSON visual_discrepancies "list of diff areas"
        TEXT github_repo_url
        BOOLEAN github_pushed
        VARCHAR github_branch
        TEXT github_pr_url
        VARCHAR deployment_status "not_deployed|deploying|deployed|failed"
        TEXT deployment_url "live Vercel URL"
        TEXT deployment_error
        DATETIME last_deployed_at
        VARCHAR vercel_project_id
    }

    COMPONENTS {
        INTEGER id PK
        VARCHAR name "e.g. PrimaryButton"
        TEXT code "Full React TSX source"
        TEXT description "AI-generated description"
        VARCHAR category "button|form|card|layout|nav"
        JSON props_schema "TypeScript interface"
        JSON tailwind_classes "used Tailwind utilities"
        JSON figma_metadata "original Figma node data"
        TEXT embedding_vector "384-dim CSV string"
        INTEGER project_id FK "origin project"
        INTEGER reuse_count "times reused"
        DATETIME created_at
        DATETIME updated_at
    }

    COMPONENT_USAGE {
        INTEGER id PK
        INTEGER component_id FK
        INTEGER project_id FK
        BOOLEAN was_modified
        JSON modifications "what was changed"
    }

    PROJECTS ||--o{ COMPONENTS : "generates"
    PROJECTS ||--o{ COMPONENT_USAGE : "tracks usage"
    COMPONENTS ||--o{ COMPONENT_USAGE : "used in"
    PROJECTS ||--o{ PROJECTS : "has pages (parent_project_id)"
```

### Storage Layer Architecture

```mermaid
graph TB
    subgraph PRIMARY["Primary Store — JSON File"]
        J1["project_store.py<br/>ProjectStore class"]
        J2["data/projects.json<br/>All project metadata"]
        J3["asyncio.Lock<br/>Thread-safe writes"]
        J4["Dict[int, Project]<br/>In-memory cache<br/>Fast reads"]
        J1 --> J2
        J1 --> J3
        J1 --> J4
    end

    subgraph SECONDARY["Secondary Store — SQLite ORM"]
        DB1["database/db.py<br/>SQLAlchemy async engine<br/>Connection pool"]
        DB2["aura2.db<br/>SQLite file"]
        DB3["models.py<br/>Project · Component<br/>ComponentUsage ORM classes"]
        DB1 --> DB2
        DB1 --> DB3
    end

    subgraph VECTOR["Vector Store — ChromaDB"]
        C1["rag/component_store.py<br/>ComponentStore class"]
        C2["component_library/chroma/<br/>Persistent storage"]
        C3["Collection: react_components<br/>Embeddings + metadata"]
        C4["sentence-transformers<br/>all-MiniLM-L6-v2<br/>384-dim vectors"]
        C5["Cosine similarity search<br/>Threshold: 60%<br/>Top-K results"]
        C1 --> C2
        C1 --> C3
        C4 --> C3
        C3 --> C5
    end

    subgraph ACCESS["Access Patterns"]
        AP1["Project CRUD<br/>→ JSON (primary)<br/>→ SQLite (audit)"]
        AP2["Component semantic search<br/>→ ChromaDB"]
        AP3["Component metadata<br/>→ SQLite"]
        AP4["System stats<br/>→ JSON count + ChromaDB count"]
    end

    PRIMARY --> AP1
    SECONDARY --> AP1
    VECTOR --> AP2
    SECONDARY --> AP3
    PRIMARY & VECTOR --> AP4

    style PRIMARY fill:#1e40af,color:#fff
    style SECONDARY fill:#581c87,color:#fff
    style VECTOR fill:#065f46,color:#fff
    style ACCESS fill:#374151,color:#fff
```

---

## 9. API Endpoints Map

```mermaid
mindmap
  root((FastAPI<br/>/api<br/>22 endpoints))
    Project Management
      POST /projects/create
        Body: ProjectCreateRequest
          figma_url
          project_name
          ui_library default=tailwind
          data_source figma_url|plugin
          plugin_data optional
          add_as new_project|new_page
          parent_project_id optional
        Returns: project_id + status
      POST /figma/plugin-upload
        Body: PluginUploadRequest
        Direct Figma plugin flow
      POST /projects/add-website
        Multi-page project support
      GET /projects
        List all projects
        Returns: projects array
      GET /projects/available
        Successful projects only
        Used for page parent selection
      GET /projects/{id}/status
        Returns: ProjectStatusResponse
          Full project details
          Deployment info
          Verification metrics
      DELETE /projects/{id}
        Delete project + files
      POST /projects/cleanup
        Remove temp files
      DELETE /projects/clear-all
        Full system reset
    Build & Preview
      GET /projects/{id}/preview-url
        Auto-starts dev server
        Returns: localhost:PORT
      POST /projects/{id}/build
        npm install + npm run build
        Returns: BuildResult
      POST /projects/{id}/start-dev-server
        Start Vite process
        Allocate port 5173+
      POST /projects/{id}/stop-dev-server
        Kill Vite process
        Free port
    Deployment
      POST /projects/{id}/push-to-github
        Create repo if needed
        Push to main branch
        Optional PR creation
        Returns: repo_url + pr_url
      POST /projects/{id}/deploy-to-vercel
        Upload to Vercel API
        Poll until deployed
        Returns: deployment_url
      GET /projects/{id}/deployment-status
        GitHub + Vercel combined status
        Returns: DeploymentStatusResponse
    Components & Stats
      GET /components
        Query: category filter
        Returns: all components
        With metadata + code
      GET /api/stats
        Total projects
        Success rate
        Components count
        Reuse rate
        Avg conversion time
    Static Files
      GET /projects/*
        StaticFiles mount
        Serve built dist/ folders
        Direct HTML access
```

---

## 10. RAG Component Reuse System

```mermaid
flowchart TB
    subgraph INGEST["📥 Component Ingestion (After Generation)"]
        I1["New React Component<br/>Generated by Claude Agent"]
        I2["AI generates description:<br/>component type · purpose<br/>prop interface · Tailwind classes"]
        I3["sentence-transformers<br/>all-MiniLM-L6-v2<br/>Generate 384-dim embedding"]
        I4["ChromaDB.upsert()<br/>Collection: react_components<br/>Metadata: name · category · code<br/>props_schema · tailwind_classes"]
        I5["SQLite component record<br/>With reuse_count=0"]
        I1 --> I2 --> I3 --> I4
        I4 --> I5
    end

    subgraph SEARCH["🔍 Semantic Search (Before Generation)"]
        S1["Design requirements<br/>from Figma extraction:<br/>component type + style context"]
        S2["Query embedding<br/>same sentence-transformer model"]
        S3["ChromaDB query()<br/>n_results=5<br/>Return top-K by cosine similarity"]
        S4{"Similarity<br/>score?"}
        S5["reuse_directly<br/>> 90%<br/>Return code as-is"]
        S6["adapt<br/>70–90%<br/>Return with modification hints"]
        S7["create_new<br/>< 60%<br/>Return empty → generate fresh"]
        S1 --> S2 --> S3 --> S4
        S4 -- "> 90%" --> S5
        S4 -- "70–90%" --> S6
        S4 -- "< 60%" --> S7
    end

    subgraph REUSE["♻️ Reuse in Code Generation"]
        R1["Matched components<br/>injected into Claude prompt<br/>as 'existing_components' context"]
        R2["Claude Agent SDK<br/>Adapts / reuses matched components<br/>Generates only what's missing"]
        R3["component_matcher.py<br/>Final similarity scoring<br/>Post-generation validation"]
        R4["component_usage table<br/>Record: component_id + project_id<br/>was_modified + modifications"]
        R5["Increment reuse_count<br/>in components table"]
        S5 & S6 --> R1 --> R2
        R2 --> R3
        R3 --> R4 --> R5
    end

    subgraph BENEFIT["📊 Measured Benefits"]
        B1["⚡ 70% faster<br/>for similar components"]
        B2["🎨 Consistent design<br/>language across projects"]
        B3["💰 Reduced API cost<br/>Fewer generation tokens"]
        B4["📈 Self-improving<br/>Grows with each project"]
    end

    R5 --> B1 & B2 & B3 & B4

    style INGEST fill:#1e40af,color:#fff
    style SEARCH fill:#065f46,color:#fff
    style REUSE fill:#78350f,color:#fff
    style BENEFIT fill:#374151,color:#fff
```

---

## 11. Visual Verification Loop

```mermaid
stateDiagram-v2
    [*] --> ProjectSetup : Step 4 begins

    ProjectSetup : 📁 project_setup.py<br/>Scaffold from template<br/>npm install dependencies

    ProjectSetup --> StartDevServer

    StartDevServer : 🚀 dev_server_manager.py<br/>Spawn: npm run dev<br/>Wait for localhost:PORT health check

    StartDevServer --> CaptureScreenshot

    CaptureScreenshot : 📸 Playwright (headless Chromium)<br/>Navigate to localhost:PORT<br/>Full-page screenshot → screenshot.png

    CaptureScreenshot --> ClaudeVisionCompare

    ClaudeVisionCompare : 👁️ vision_comparison.py<br/>Claude Vision Sonnet 4<br/>Input: figma_design.png + screenshot.png<br/>Output:<br/>  • confidence: 0.0–1.0<br/>  • discrepancies: list of issues<br/>    (layout · colors · fonts · spacing)

    ClaudeVisionCompare --> EvalConfidence

    EvalConfidence : Evaluate confidence score

    state EvalConfidence <<choice>>

    EvalConfidence --> EarlyStop : score ≥ 0.98 ✅
    EvalConfidence --> CheckMaxIter : score < 0.98

    state CheckMaxIter <<choice>>

    CheckMaxIter --> GenerateFixes : iterations < 10
    CheckMaxIter --> ForcedStop : iterations ≥ 10 ⚠️

    GenerateFixes : 🔧 auto_fix_agent.py<br/>Claude Opus 4.6<br/>For each discrepancy → targeted fix<br/>Code patch per component file

    GenerateFixes --> TrackFixes

    TrackFixes : 📋 fix_tracker.py<br/>Record fix attempt<br/>Prevent duplicate fixes<br/>Track fix history

    TrackFixes --> ApplyFixes

    ApplyFixes : ✏️ fix_applicator.py<br/>Apply code patches<br/>to source .tsx files

    ApplyFixes --> CaptureScreenshot : Re-render with fixes

    EarlyStop : ✅ PASS<br/>Save: verification_confidence<br/>Save: verification_iterations<br/>Save: visual_match = true

    ForcedStop : ⚠️ TIMEOUT<br/>Save best confidence achieved<br/>Mark for manual review<br/>visual_match = false

    EarlyStop --> [*]
    ForcedStop --> [*]
```

---

## 12. Deployment Pipeline

```mermaid
flowchart TB
    START(["✅ npm run build<br/>succeeded"]) --> OPTIONS

    OPTIONS["Deployment options<br/>based on env vars + user action"]

    OPTIONS --> LOCAL
    OPTIONS --> GITHUB
    OPTIONS --> VERCEL
    OPTIONS --> STATIC

    subgraph LOCAL["🖥️ Local Preview (Always Available)"]
        L1["dev_server_manager.py<br/>start_dev_server(project_id)"]
        L2["Vite: npm run dev<br/>Port range: 5173–6000"]
        L3["GET /api/projects/{id}/preview-url<br/>Returns localhost:PORT"]
        L4["ProjectPreview.tsx<br/>Iframe embed in dashboard"]
        L1 --> L2 --> L3 --> L4
    end

    subgraph STATIC["📁 Static File Serving"]
        S1["FastAPI mounts:<br/>/projects → generated_projects/"]
        S2["Serve built dist/ files<br/>Direct HTML/CSS/JS access"]
        S3["No Node.js required<br/>Pure static serving"]
        S1 --> S2 --> S3
    end

    subgraph GITHUB["🐙 GitHub Integration<br/>git_manager.py"]
        G1{"GITHUB_TOKEN<br/>set?"}
        G2["Create repository<br/>via GitHub API v3"]
        G3["git init<br/>git remote add origin"]
        G4["git add . && git commit<br/>git push origin main"]
        G5{"AUTO_CREATE_PR<br/>= true?"}
        G6["Create Pull Request<br/>title: 'Generated by Aura2'<br/>body: metrics + verification"]
        G7["Save to project:<br/>github_repo_url<br/>github_branch<br/>github_pr_url<br/>github_pushed = true"]
        G1 -- Yes --> G2 --> G3 --> G4 --> G5
        G5 -- Yes --> G6 --> G7
        G5 -- No --> G7
        G1 -- No --> SKIP_GH[Skip GitHub]
    end

    subgraph VERCEL["⬆️ Vercel Deployment"]
        V1{"VERCEL_TOKEN<br/>set?"}
        V2["POST /v13/deployments<br/>Vercel API<br/>Upload project files"]
        V3["deployment_status = 'deploying'"]
        V4["Poll /v13/deployments/{id}<br/>every 5 seconds"]
        V5{"Build<br/>succeeded?"}
        V6["Save:<br/>deployment_url<br/>vercel_project_id<br/>last_deployed_at<br/>status = 'deployed'"]
        V7["Save:<br/>deployment_error<br/>status = 'failed'"]
        V1 -- Yes --> V2 --> V3 --> V4 --> V5
        V5 -- Yes --> V6
        V5 -- No --> V7
        V1 -- No --> SKIP_V[Skip Vercel]
    end

    LOCAL --> END
    STATIC --> END
    GITHUB --> END
    VERCEL --> END

    END["📊 project_store.update_project()<br/>Final status saved<br/>Frontend polling sees update<br/>→ DeploymentCard shows URLs"]

    style START fill:#059669,color:#fff
    style END fill:#1e40af,color:#fff
    style LOCAL fill:#374151,color:#fff
    style STATIC fill:#374151,color:#fff
    style GITHUB fill:#1e3a5f,color:#fff
    style VERCEL fill:#1e40af,color:#fff
```

---

## 13. Component Relationships (Class Diagram)

```mermaid
classDiagram
    class FigmaToReactAgent {
        -settings: Settings
        -project_store: ProjectStore
        -component_store: ComponentStore
        -auto_fix_agent: AutoFixAgent
        -vision_comparison: VisionComparison
        -dev_server: DevServerManager
        +convert_figma_to_react(url, name, ui_library) Task
        +convert_from_plugin_data(data, name) Task
        -_extract_design(url) DesignData
        -_setup_project(design, template) Path
        -_search_reusable_components(design) List~Component~
        -_generate_code(design, reusable) GeneratedCode
        -_verify_visually(path, original) VerificationResult
        -_build_and_validate(path) BuildResult
        -_save_components(code) void
    }

    class ProjectStore {
        -_projects: Dict~int, Project~
        -_lock: asyncio.Lock
        -_file_path: Path
        +create_project(name, url) Project
        +get_project(id) Project
        +update_project(id, kwargs) Project
        +list_projects() List~Project~
        +delete_project(id) bool
        -_save_to_file() void
        -_load_from_file() void
    }

    class ComponentStore {
        -_client: chromadb.Client
        -_collection: Collection
        -_model: SentenceTransformer
        +add_component(name, code, desc, cat) str
        +search_similar(query, threshold) List~ComponentMatch~
        +get_component(id) Component
        +list_components(category) List~Component~
        +get_stats() StoreStats
    }

    class AutoFixAgent {
        -_claude: Anthropic
        -_model: str
        -_fix_tracker: FixTracker
        +generate_fixes(code, discrepancies) List~Fix~
        +apply_and_verify(path, fixes) bool
    }

    class VisionComparison {
        -_claude: Anthropic
        -_model: str
        +compare(design_img, screenshot) ComparisonResult
        +get_confidence() float
        +get_discrepancies() List~str~
        +generate_diff_report() str
    }

    class DevServerManager {
        -_servers: Dict~str, ServerProcess~
        -_port_pool: List~int~
        +start_dev_server(project_id) int
        +stop_dev_server(project_id) bool
        +get_port(project_id) int
        +is_running(project_id) bool
        +cleanup_all() void
    }

    class BuildVerifier {
        +verify_build(project_path) BuildResult
        +run_npm_install(path) bool
        +run_npm_build(path) bool
        +get_build_errors() List~str~
    }

    class GitManager {
        -_token: str
        -_owner: str
        +create_repo(name) RepoInfo
        +push_to_github(path, repo) bool
        +create_pr(repo, title, body) PRInfo
        +get_repo_url(name) str
    }

    class Settings {
        +figma_token: str
        +litellm_api_key: str
        +litellm_base_url: str
        +database_url: str
        +generated_projects_dir: Path
        +component_library_dir: Path
        +enable_vision_comparison: bool
        +max_verification_iterations: int
        +verification_confidence_threshold: float
        +verification_early_stop_threshold: float
        +github_personal_access_token: str
        +auto_create_repo: bool
        +auto_create_pr: bool
        +vercel_token: str
        +auto_deploy_vercel: bool
        +max_agent_turns: int
        +max_fix_turns: int
    }

    class Project {
        +id: int
        +name: str
        +figma_url: str
        +status: ProjectStatus
        +project_path: str
        +components_generated: int
        +components_reused: int
        +conversion_time_seconds: float
        +verification_confidence: float
        +verification_iterations: int
        +github_repo_url: str
        +github_pr_url: str
        +deployment_url: str
        +deployment_status: DeploymentStatus
        +is_page: bool
        +parent_project_id: int
        +route_path: str
    }

    FigmaToReactAgent --> ProjectStore : reads / writes
    FigmaToReactAgent --> ComponentStore : search / store
    FigmaToReactAgent --> AutoFixAgent : delegates fixes
    FigmaToReactAgent --> VisionComparison : screenshot diff
    FigmaToReactAgent --> DevServerManager : manages server
    FigmaToReactAgent --> BuildVerifier : validates build
    FigmaToReactAgent --> GitManager : optional deploy
    ProjectStore --> Project : manages instances
    Settings --> FigmaToReactAgent : configures all
    Settings --> DevServerManager : port config
    Settings --> VisionComparison : model + thresholds
```

---

## 14. Full Conversion Sequence Diagram

```mermaid
sequenceDiagram
    actor User
    participant FE as React Frontend<br/>Vite + React Query
    participant API as FastAPI Backend<br/>:8000
    participant Agent as FigmaToReactAgent<br/>Claude Agent SDK
    participant Figma as Figma REST API<br/>api.figma.com
    participant LiteLLM as LiteLLM Proxy<br/>→ Claude Opus 4.6
    participant ChromaDB as ChromaDB<br/>Vector Store
    participant Playwright as Playwright<br/>Headless Chromium
    participant Vision as Claude Vision<br/>Sonnet 4
    participant Store as Project Store<br/>JSON + SQLite

    User->>FE: Enter Figma URL + project name<br/>click "Generate"
    FE->>API: POST /api/projects/create<br/>{ figma_url, project_name, ui_library }
    API->>Store: create_project() → id=42<br/>status = "pending"
    API-->>FE: { project_id: 42, status: "pending" }
    Note over FE: Start polling /api/projects/42/status<br/>every 2 seconds

    API->>Agent: background task:<br/>convert_figma_to_react(url, "MyProject")
    API->>Store: update status → "generating"

    rect rgb(219, 234, 254)
        Note over Agent,Figma: STEP 1 — Design Extraction
        Agent->>Figma: GET /files/{key}<br/>Authorization: Bearer FIGMA_TOKEN
        Figma-->>Agent: Design JSON<br/>(node tree, styles, components)
        Agent->>Figma: GET /images/{key}?ids=...&format=png
        Figma-->>Agent: Image export URLs
        Agent->>Agent: figma_extraction.py → parse nodes
        Agent->>Agent: design_styles.py → extract colors/fonts/effects
        Agent->>Agent: responsive_extraction.py → breakpoints
    end

    rect rgb(254, 243, 199)
        Note over Agent,ChromaDB: STEP 2 — RAG Component Reuse
        Agent->>ChromaDB: query(embedding, n_results=5)<br/>for each detected component type
        ChromaDB-->>Agent: [ {id, code, similarity: 0.87}, ... ]
        Note over Agent: Filter: ≥60% threshold<br/>3 reusable components found
    end

    rect rgb(220, 252, 231)
        Note over Agent,LiteLLM: STEP 3 — Code Generation
        Agent->>LiteLLM: Chat completion<br/>system: role + guidelines<br/>user: design JSON + reusable components<br/>model: claude-opus-4-6<br/>max_turns: 35
        LiteLLM-->>Agent: React + TypeScript + Tailwind<br/>streaming response
        Agent->>Agent: semantic_analysis.py → ARIA attrs
        Agent->>Agent: accessibility.py → WCAG 2.1 AA
    end

    rect rgb(252, 231, 243)
        Note over Agent,Vision: STEP 4 — Visual Verification Loop
        Agent->>Agent: project_setup.py → scaffold from template
        Agent->>Playwright: launch browser<br/>npm run dev → :5173
        Playwright-->>Agent: Server ready ✅

        loop Until confidence ≥ 98% (max 10 iterations)
            Agent->>Playwright: page.screenshot(fullPage=True)
            Playwright-->>Agent: screenshot.png (bytes)
            Agent->>Vision: messages.create()<br/>images: [figma_design.png, screenshot.png]<br/>prompt: "Compare and score similarity"
            Vision-->>Agent: { confidence: 0.85,<br/>discrepancies: ["button color off", "font size wrong"] }

            alt confidence < 98%
                Agent->>LiteLLM: Generate fixes for discrepancies<br/>model: claude-opus-4-6
                LiteLLM-->>Agent: Code patches per file
                Agent->>Agent: fix_applicator.py → apply patches
                Agent->>Agent: fix_tracker.py → record attempt
            end
        end

        Agent->>Playwright: close browser
    end

    rect rgb(237, 233, 254)
        Note over Agent,Store: STEP 5 — Build & Finalize
        Agent->>Agent: npm install
        Agent->>Agent: npm run build → TypeScript compile
        Agent->>Agent: ESLint + Prettier auto-fix
        Agent->>ChromaDB: Add new components with embeddings
        Agent->>Store: update_project(<br/>  status="success",<br/>  confidence=0.98,<br/>  components_generated=8,<br/>  components_reused=3,<br/>  conversion_time=87.3<br/>)
    end

    Store-->>FE: polling returns status="success"<br/>+ full project details
    FE-->>User: 🎉 Show project preview<br/>+ deploy options

    opt Deploy to GitHub
        User->>FE: Click "Push to GitHub"
        FE->>API: POST /api/projects/42/push-to-github
        API->>Agent: git_manager.create_repo() + push
        Agent-->>API: { repo_url: "github.com/...", pr_url: "..." }
        API->>Store: save github_repo_url + github_pr_url
        API-->>FE: GitHub URLs
        FE-->>User: Show repo + PR links
    end

    opt Deploy to Vercel
        User->>FE: Click "Deploy to Vercel"
        FE->>API: POST /api/projects/42/deploy-to-vercel
        API->>Agent: POST /v13/deployments (Vercel API)
        Note over Agent: Poll until deployed...
        Agent-->>API: { deployment_url: "https://myproject.vercel.app" }
        API->>Store: save deployment_url
        API-->>FE: Live URL
        FE-->>User: 🌐 Open live URL
    end
```

---

## 15. Performance Benchmarks

| Metric | Aura1 (LangGraph) | Aura2 (Claude SDK) | Improvement |
|--------|:-----------------:|:-------------------:|:-----------:|
| Conversion Time | 58 min | ~90 sec | **48× faster** |
| Build Success Rate | 20% | 100% | **5× better** |
| Visual Accuracy | 72% | 95% | **+32%** |
| Monthly API Cost | $2,907 | $262 | **91% cheaper** |
| Manual Fixes Needed | 80% | 5% | **94% reduction** |
| Component Reuse | 0% | ~70% | **New capability** |

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#1e40af'}}}%%
xychart-beta
    title "Aura1 vs Aura2 — Key Metrics Comparison"
    x-axis ["Build Success %", "Visual Accuracy %", "Manual Fix Rate %", "Component Reuse %"]
    y-axis "Percentage" 0 --> 100
    bar [20, 72, 80, 0]
    bar [100, 95, 5, 70]
```

---

## 16. Environment Configuration Map

```mermaid
graph LR
    subgraph ENV[".env — All Configuration"]
        subgraph AI["🤖 AI Core"]
            E1["FIGMA_TOKEN<br/>figd_xxxx...<br/>Figma REST API access"]
            E2["LITELLM_API_KEY<br/>Claude API access<br/>via LiteLLM proxy"]
            E3["LITELLM_BASE_URL<br/>https://proxy.example.com<br/>Proxy endpoint"]
            E4["LITELLM_PROVIDER<br/>anthropic"]
        end
        subgraph DB_CFG["💾 Storage"]
            E5["DATABASE_URL<br/>sqlite+aiosqlite:///./aura2.db"]
            E6["GENERATED_PROJECTS_DIR<br/>./generated_projects"]
            E7["COMPONENT_LIBRARY_DIR<br/>./component_library"]
        end
        subgraph GIT_CFG["🐙 GitHub"]
            E8["GITHUB_PERSONAL_ACCESS_TOKEN<br/>ghp_xxxx..."]
            E9["GITHUB_OWNER<br/>your-username"]
            E10["AUTO_CREATE_REPO<br/>false (default)"]
            E11["AUTO_CREATE_PR<br/>false (default)"]
        end
        subgraph VER_CFG["⬆️ Vercel"]
            E12["VERCEL_TOKEN<br/>xxxx..."]
            E13["VERCEL_ORG_ID<br/>team_xxxx (optional)"]
            E14["AUTO_DEPLOY_VERCEL<br/>false (default)"]
        end
        subgraph VIS_CFG["👁️ Vision Verification"]
            E15["ENABLE_VISION_COMPARISON<br/>true (default)"]
            E16["VISION_COMPARISON_MODEL<br/>claude-sonnet-4"]
            E17["MAX_VERIFICATION_ITERATIONS<br/>10"]
            E18["VERIFICATION_CONFIDENCE_THRESHOLD<br/>0.95"]
            E19["VERIFICATION_EARLY_STOP_THRESHOLD<br/>0.98"]
        end
        subgraph AGENT_CFG["🤖 Agent Tuning"]
            E20["MAX_AGENT_TURNS<br/>35 (Claude SDK)"]
            E21["MAX_FIX_TURNS<br/>15 (Auto-fix)"]
        end
    end

    subgraph READS["config.py reads via Pydantic BaseSettings"]
        SETTINGS["Settings class<br/>All fields with defaults<br/>Auto-load from .env<br/>Used everywhere in app"]
    end

    ENV --> READS

    style AI fill:#d97706,color:#fff
    style DB_CFG fill:#581c87,color:#fff
    style GIT_CFG fill:#1e3a5f,color:#fff
    style VER_CFG fill:#1e40af,color:#fff
    style VIS_CFG fill:#065f46,color:#fff
    style AGENT_CFG fill:#78350f,color:#fff
    style READS fill:#374151,color:#fff
```

---

*Aura2 Architecture Diagrams — generated by Claude Code*
*Render with any Mermaid-compatible viewer: GitHub · VS Code (Mermaid Preview extension) · [mermaid.live](https://mermaid.live)*
