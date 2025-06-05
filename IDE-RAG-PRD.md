# IDE-RAG Assistant – Product Requirements & Development Plan

## 1. Executive Summary
IDE-RAG Assistant is a plug-and-play extension + backend server that brings Retrieval-Augmented Generation (RAG) workflows directly into developers’ editors. Borrowing the end-to-end crawl-to-context pipeline of **mcp-crawl4ai-rag**, the product exposes web-crawling, knowledge-base creation, and advanced retrieval strategies to any LLM agent running inside VS Code, JetBrains IDEs, and other MCP-compatible tools.  
Key differentiator: a **WebSocket transport layer with a stdio adapter** ensures low-latency, bidirectional streaming between the IDE UI and the server while remaining compatible with existing MCP stdio clients.

## 2. Product Overview
Goal Accelerate developer productivity by surfacing precise, source-linked answers, code snippets, and API examples from both public documentation and private corpora—all without leaving the editor.

Target Users  
• Software engineers (frontend, backend, DevOps)  
• Technical writers maintaining docs  
• AI pair-programming tools that need contextual knowledge  

Value Proposition  
1. One-click ingestion of docs/sites relevant to the open file or current stack.  
2. Hybrid vector + keyword retrieval tuned for code symbols.  
3. Live context injection: active buffer, diagnostics, and selection are streamed to the LLM.  
4. Works offline/on-prem via Dockerized backend and local embeddings.

### User Journey Flow
journey
    title IDE-RAG User Journey
    section Setup
        Install IDE Extension: 5: User
        Configure Connection: 3: User
        Add Knowledge Sources: 4: User
    section Daily Usage
        Open Project File: 5: User
        Encounter Question: 3: User
        Ask IDE-RAG: 5: User, System
        View Contextual Answer: 5: User
        Apply Code Suggestion: 4: User
    section Knowledge Management
        Add New Documentation: 4: User
        Crawl External Resources: 3: User, System
        Search Across Sources: 5: User, System

## 3. Technical Architecture
```
┌──────────────────────┐
│   IDE Extension      │
│  (VS Code, JetBrains)│
└─────────┬────────────┘ ❶ WebSocket (JSON-RPC over WS)
          │
┌─────────▼────────────┐
│   Transport Bridge   │  (stdio <-> WS adapter)
└─────────┬────────────┘
          │ ❷ stdio (MCP)
┌─────────▼────────────┐
│  RAG Server (FastAPI)│
│  • Crawl4AI Engine   │
│  • Retrieval layer   │
│  • Tool registry     │
└─────────┬────────────┘
          │ ❸ SQL & Vector
┌─────────▼────────────┐
│ Supabase Postgres    │
│   + pgvector         │
└──────────────────────┘
```

### System Context Diagram
flowchart TB
    User["Software Developer"]:::user
    IDE["IDE Environment<br>VS Code / JetBrains"]:::ide
    Bridge["WebSocket Bridge"]:::bridge
    RAG["RAG Server"]:::server
    DB["Vector Database<br>Supabase + pgvector"]:::db
    Docs["Documentation Sites"]:::ext
    API["OpenAI / Ollama API"]:::ext
    
    User -->|"Uses"| IDE
    IDE -->|"WebSocket<br>JSON-RPC"| Bridge
    Bridge -->|"stdio<br>MCP Protocol"| RAG
    RAG -->|"SQL + Vector<br>Queries"| DB
    RAG -->|"Crawl"| Docs
    RAG -->|"Embeddings<br>Request"| API
    
    classDef user fill:#de953e,color:white,stroke:#333,stroke-width:1px
    classDef ide fill:#4672b4,color:white,stroke:#333,stroke-width:1px
    classDef bridge fill:#4672b4,color:white,stroke:#333,stroke-width:1px
    classDef server fill:#47956f,color:white,stroke:#333,stroke-width:1px
    classDef db fill:#8b251e,color:white,stroke:#333,stroke-width:1px
    classDef ext fill:#de953e,color:white,stroke:#333,stroke-width:1px

## 4. Core Features & Capabilities
| Category | Feature | Notes |
|----------|---------|-------|
| Ingestion | Smart crawl URL / sitemap / local folder | Re-uses Crawl4AI cleaners & chunking |
| Retrieval | Vector search • BM25 • hybrid combine • cross-encoder rerank | Toggle via env flags |
| IDE Context | Auto-embed active file, selection, diagnostics | Sent as `editor_context` param in queries |
| Agent Tools | `crawl_single_page`, `smart_crawl_url`, `perform_rag_query`, `search_code_examples`, `list_sources` | Exposed via MCP |
| Transport | WebSocket (protobuf frames) + stdio adapter | ≤50 ms round-trip |
| Security | Domain whitelist, robots.txt compliance, per-workspace Supabase schema | Row-Level Security enabled |
| Extensibility | Plug-in retrieval strategies, embed models (OpenAI, Ollama) | Strategy registry pattern |

### Feature Dependency Graph
flowchart TD
    Core["Core RAG<br>Functionality"]:::core
    
    WS["WebSocket<br>Transport"]:::transport
    Stdio["stdio<br>Adapter"]:::transport
    
    VS["VS Code<br>Extension"]:::client
    JB["JetBrains<br>Plugin"]:::client
    
    Crawl["Web Crawling<br>Engine"]:::ingestion
    Parser["HTML → Markdown<br>Processor"]:::ingestion
    
    Hybrid["Hybrid<br>Search"]:::retrieval
    Context["Contextual<br>Embeddings"]:::retrieval
    Rerank["Cross-Encoder<br>Reranking"]:::retrieval
    
    EditorSync["Editor Context<br>Sync"]:::context
    LiveQuery["Live Query<br>Enhancement"]:::context
    
    Auth["JWT Auth +<br>RLS"]:::security
    
    Core --> WS
    Core --> Stdio
    
    WS --> VS
    WS --> JB
    
    Core --> Crawl
    Crawl --> Parser
    
    Core --> Hybrid
    Core --> Context
    Core --> Rerank
    
    VS --> EditorSync
    JB --> EditorSync
    EditorSync --> LiveQuery
    
    WS --> Auth
    
    classDef core fill:#4672b4,color:white,stroke:#333,stroke-width:1px
    classDef transport fill:#47956f,color:white,stroke:#333,stroke-width:1px
    classDef client fill:#de953e,color:white,stroke:#333,stroke-width:1px
    classDef ingestion fill:#8b251e,color:white,stroke:#333,stroke-width:1px
    classDef retrieval fill:#47956f,color:white,stroke:#333,stroke-width:1px
    classDef context fill:#de953e,color:white,stroke:#333,stroke-width:1px
    classDef security fill:#8b251e,color:white,stroke:#333,stroke-width:1px

## 5. Value Stream Map
flowchart LR
    subgraph Problem
        P1["Context Switching<br>to Browsers"]:::problem
        P2["Documentation<br>Fragmentation"]:::problem
        P3["Imprecise Code<br>Examples"]:::problem
    end
    
    subgraph Solution
        S1["IDE-Integrated<br>RAG"]:::solution
        S2["Multi-Source<br>Knowledge Base"]:::solution
        S3["Code-Specific<br>Retrieval"]:::solution
    end
    
    subgraph Value
        V1["30 % Reduction in<br>External Searches"]:::value
        V2["Faster Onboarding<br>to New Codebases"]:::value
        V3["Improved Code<br>Quality & Consistency"]:::value
    end
    
    P1 --> S1
    P2 --> S2
    P3 --> S3
    
    S1 --> V1
    S2 --> V2
    S3 --> V3
    
    classDef problem fill:#8b251e,color:white,stroke:#333,stroke-width:1px
    classDef solution fill:#47956f,color:white,stroke:#333,stroke-width:1px
    classDef value fill:#4672b4,color:white,stroke:#333,stroke-width:1px

## 6. Technical Implementation Details
* **Python 3.12**, Poetry/`pyproject.toml`.
* **FastAPI** + `uvicorn` for SSE transport; `json-rpc` for stdio.
* **Supabase-py** client uses `match_documents` SQL fn (see `crawled_pages.sql`).
* Embeddings default: `text-embedding-3-small`; override via `MODEL_CHOICE`.
* Environment flags toggle strategies (`USE_HYBRID_SEARCH=true`, etc.).
* Dockerfile installs system dependencies, Playwright browsers, pgvector client libs.
* Retry & concurrency logic handled in `src/crawler_utils.py` (uses `asyncio.gather` with exponential back-off).

## 7. Integration Patterns
| Environment | How to Connect |
|-------------|----------------|
| **IDE/Editor (VS Code, JetBrains, Cursor)** | Run Bridge on `localhost:3589`; extension auto-connects via WebSocket. |
| **Local agents (LangGraph, CrewAI)** | Use `mcpadapt` to call tools directly over stdio. |
| **n8n / Node-RED** | HTTP fallback endpoints with SSE for streaming answers. |
| **Cloud workloads** | Deploy container on Render/Fly; secure Supabase creds with env vars & Row-Level Security. |

### Deployment Architecture
flowchart TB
    subgraph "Developer Environment"
        IDE["IDE<br>VS Code / JetBrains"]:::client
        Bridge["WebSocket<br>Bridge"]:::bridge
    end
    
    subgraph "Server Infrastructure"
        RAG["RAG Server<br>FastAPI"]:::server
        VDB["Vector Database<br>Supabase"]:::db
    end
    
    subgraph "External Services"
        Embed["Embedding API<br>OpenAI / Ollama"]:::ext
        Crawl["Web Crawler<br>Crawl4AI"]:::ext
    end
    
    IDE <-->|"WebSocket<br>JSON-RPC"| Bridge
    Bridge <-->|"stdio<br>MCP"| RAG
    RAG <-->|"SQL<br>Vector Queries"| VDB
    RAG -->|"Embedding<br>Requests"| Embed
    RAG -->|"Crawl<br>Requests"| Crawl
    
    classDef client fill:#de953e,color:white,stroke:#333,stroke-width:1px
    classDef bridge fill:#4672b4,color:white,stroke:#333,stroke-width:1px
    classDef server fill:#47956f,color:white,stroke:#333,stroke-width:1px
    classDef db fill:#8b251e,color:white,stroke:#333,stroke-width:1px
    classDef ext fill:#de953e,color:white,stroke:#333,stroke-width:1px

## 8. Strengths and Limitations
### Strengths
+ Turn-key crawl-to-RAG pipeline.
+ MCP gives plug-and-play integration with 650+ compliant hosts.
+ Multiple retrieval strategies outperform naïve vector search.
+ Docker image simplifies reproducibility.

### Limitations
− Supabase vector search latency vs. dedicated vector DBs at large scale.  
− Heavy use of OpenAI embeds can be costly; local Ollama support is only “roadmap”.  
− Agentic RAG increases crawl time significantly.  
− Security model relies on host consent but server tools could still scrape sensitive sites.

## 9. Use Cases and Applications
* **AI Coding Assistants** – index API docs, surface code examples quickly.  
* **Enterprise Knowledge Hubs** – crawl internal Confluence/SharePoint, provide chat interface.  
* **Competitive Intelligence** – agents auto-crawl competitor sites and summarize changes.  
* **Research Bots** – scrape academic sites, store papers, hybrid search + rerank for Q&A.  

## 10. Project Structure Overview
```
ide-rag/
│
├─ src/
│  ├─ crawl4ai_mcp.py          # FastAPI entry & MCP tool declarations
│  ├─ strategies/
│  │   ├─ contextual.py
│  │   ├─ hybrid.py
│  │   ├─ agentic.py
│  │   └─ rerank.py
│  ├─ crawler_utils.py
│  └─ db_supabase.py
├─ crawled_pages.sql           # pgvector schema & SQL functions
├─ Dockerfile
└─ pyproject.toml
```

## 11. Future Roadmap (from author)
1. Tight integration with **Archon** knowledge engine.  
2. **Pluggable embedding models** incl. local Ollama runtime.  
3. Context-7-style **late chunking** & example-centric chunk boundaries.  
4. Performance optimizations for live indexing within a single prompt session.

---
© 2025 IDE-RAG Assistant Team
