# Aura1 Architecture - 10-Node LangGraph System

## Complete System Diagram

```mermaid
graph TB
    subgraph INPUT["INPUT LAYER"]
        A1[Figma REST API<br/>2 req/s limit]
        A2[Figma File JSON]
    end

    subgraph ORCHESTRATOR["LANGGRAPH ORCHESTRATOR"]
        B1[StateGraph<br/>23 State Fields<br/>Complex State Management]
    end

    subgraph PIPELINE["10-NODE AGENT PIPELINE"]
        C1["1. Design Extraction<br/>Qwen 2.5 7B<br/>⏱️ 5-8s<br/>Extract Figma → JSX mapping"]
        C2["2. Reuse Check<br/>ChromaDB Query<br/>⏱️ 3-5s<br/>⚠️ 60% accuracy"]
        C3["3. Layout Analysis<br/>Llama 3.1 8B<br/>⏱️ 4-7s<br/>Flex/Grid detection"]
        C4["4. Responsive Analysis<br/>Llama 3.1 8B<br/>⏱️ 3-6s<br/>Breakpoint generation"]
        C5["5. Styling Generation<br/>Llama 3.1 8B<br/>⏱️ 5-9s<br/>Tailwind mapping"]
        C6["6. Consensus Builder<br/>Llama 3.1 8B<br/>⏱️ 15-30s<br/>🔴 CRITICAL BOTTLENECK<br/>Multi-turn voting"]
        C7["7. Element Synthesis<br/>Qwen2.5-Coder 32B<br/>⏱️ 10-20s<br/>⚠️ Code quality issues"]
        C8["8. Confidence Scorer<br/>Rule-based + LLM<br/>⏱️ 2-4s<br/>Quality gating"]
        C9["9. Human Review Package<br/>Triggers if &lt; 0.75<br/>⏱️ 5-8s<br/>70% of components"]
        C10["10. Component File Generator<br/>File I/O<br/>⏱️ 1-2s<br/>4 files per component"]
    end

    subgraph INFRA["LOCAL AI INFRASTRUCTURE"]
        D1["Ollama Server<br/>🔴 32GB VRAM Required<br/>$907/month AWS"]
        D2[Qwen 2.5 7B<br/>Extraction]
        D3[Llama 3.1 8B<br/>Analysis]
        D4[Qwen2.5-Coder 32B<br/>Code Gen]
        D5[Gemini 2.5-Flash<br/>Fallback Only]
    end

    subgraph KB["KNOWLEDGE BASE"]
        E1[ChromaDB<br/>SQLite-backed]
        E2[Embeddings<br/>all-MiniLM-L6-v2]
        E3[".mb Metadata<br/>Per-component JSON"]
    end

    subgraph OUTPUT["OUTPUT LAYER"]
        F1[React Component<br/>.tsx file]
        F2[Metadata<br/>.mb JSON]
        F3[Unit Tests<br/>.test.tsx]
        F4[Storybook<br/>.stories.tsx]
    end

    A1 --> B1
    A2 --> B1
    B1 --> C1
    C1 --> C2
    C2 --> C3
    C3 --> C4
    C4 --> C5
    C5 --> C6
    C6 --> C7
    C7 --> C8
    C8 --> C9
    C8 --> C10
    C9 --> C10
    C10 --> F1
    C10 --> F2
    C10 --> F3
    C10 --> F4

    C1 -.->|uses| D2
    C2 -.->|queries| E1
    C3 -.->|uses| D3
    C4 -.->|uses| D3
    C5 -.->|uses| D3
    C6 -.->|uses| D3
    C7 -.->|uses| D4
    C7 -.->|fallback| D5

    D2 --> D1
    D3 --> D1
    D4 --> D1

    E1 --> E2
    E1 --> E3

    style C6 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style C7 fill:#ffd93d,stroke:#ffa94d
    style C9 fill:#ffd93d,stroke:#ffa94d
    style D1 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style C2 fill:#ffd93d,stroke:#ffa94d
```

## Critical Problems

### 🔴 Problem 1: Consensus Builder Bottleneck
- **Time:** 15-30 seconds per conflict
- **Impact:** 40% failure rate, requires 2-3 voting rounds
- **Root Cause:** Poor reasoning in local models

### 🔴 Problem 2: Code Quality Issues
- **Issue:** Generated JSX valid but semantically wrong
- **Impact:** Empty components, missing props, no accessibility
- **Root Cause:** Qwen2.5-Coder 32B lacks design understanding

### 🔴 Problem 3: Component Reuse Accuracy
- **Accuracy:** 60% (target: 85%)
- **Impact:** Irrelevant component recommendations
- **Root Cause:** Poor similarity scoring by local models

### 🔴 Problem 4: Monolithic Output
- **Issue:** 87 components → single 15,247-line file
- **Impact:** Build failures, unmaintainable code
- **Root Cause:** No hierarchical generation strategy

### 🔴 Problem 5: Infrastructure Cost
- **Monthly Cost:** $907 (GPU server)
- **Additional:** $2,000/month developer time (manual fixes)
- **Total:** $2,907/month

## Performance Metrics (50 components)

| Stage | Time | Issues |
|-------|------|--------|
| Design Extraction | 5-8s | 68% accuracy |
| Reuse Check | 3-5s | 60% relevance |
| Layout Analysis | 4-7s | Misses nested structures |
| Responsive | 3-6s | Inconsistent breakpoints |
| Styling | 5-9s | Color mapping errors |
| **Consensus** | **15-30s** | **🔴 BOTTLENECK** |
| Synthesis | 10-20s | Poor code quality |
| Scoring | 2-4s | 70% need review |
| **Total** | **48-91s** | **58 min for 50 components** |

## Why Local Models Failed

1. **Limited Reasoning:** Multi-turn voting required for simple decisions
2. **Context Window:** 8K-32K tokens insufficient for large designs
3. **Code Understanding:** Lacks semantic awareness of React patterns
4. **Cost:** $907/month GPU + $2,000/month manual fixes
5. **Accessibility:** Required 32GB VRAM, inaccessible to most developers

## Solution: Aura2 with Claude

See AURA2_ARCHITECTURE.md for how Claude Opus 4.6 solves all these issues.
