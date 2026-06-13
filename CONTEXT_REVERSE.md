# CONTEXT_REVERSE.md — Domain Context for Repository Generation

> **Purpose:** This file defines the *what* and *why* of the project. Change the values below to regenerate an equivalent repository for a different domain.

---

## 1. Project Identity

| Field | Value |
|-------|-------|
| **Name** | Headroom |
| **Package name (Python)** | `headroom-ai` |
| **Package name (npm)** | `headroom-ai` |
| **Tagline** | The context compression layer for AI agents |
| **One-liner** | 60–95% fewer tokens · library · proxy · MCP · 6 algorithms · local-first · reversible |
| **License** | Apache-2.0 |
| **Primary language** | Python 3.10+ with Rust extension (pyo3/maturin) |
| **Secondary languages** | TypeScript (SDK), Rust (core + proxy) |
| **Original repo** | `github.com/chopratejas/headroom` |

---

## 2. Problem Domain

| Aspect | Description |
|--------|-------------|
| **Problem** | LLM context windows are expensive and limited. AI agents send massive tool outputs, logs, RAG chunks, and files to LLMs, wasting tokens and money. |
| **Solution** | Intercept and compress all context before it reaches the LLM using content-type-aware compression algorithms, preserving accuracy while reducing tokens by 60–95%. |
| **Key insight** | Different content types (JSON, code, logs, prose) need different compression strategies. A router + specialized compressors outperforms generic approaches. |
| **Differentiators** | Local-first (no data leaves the machine), reversible (CCR — originals cached for retrieval), works with any LLM provider, zero code changes via proxy mode. |

---

## 3. Core Algorithms & Components

### 3.1 Compression Pipeline

| Component | Purpose | Content Type |
|-----------|---------|--------------|
| **ContentRouter** | Detects content type, selects compressor | All |
| **SmartCrusher** | Statistical compression of JSON arrays/objects — keeps anomalies, errors, relevant items | JSON, tool outputs |
| **CodeCompressor** | AST-aware compression using tree-sitter — preserves imports, signatures, types | Code (Python, JS, Go, Rust, Java, C++) |
| **Kompress-base** | HuggingFace ModernBERT model trained on agentic traces | Prose, text |
| **LogCompressor** | Pattern-based log deduplication | Build logs, CLI output |
| **SearchCompressor** | Relevance-scored result filtering | Search results |
| **DiffCompressor** | Diff-aware compression | Git diffs |
| **CacheAligner** | Stabilizes prefixes for provider KV cache hits | All (pre-processing) |
| **CCR (Compress-Cache-Retrieve)** | Stores originals locally; LLM retrieves on demand via tool call | All (post-processing) |

### 3.2 Delivery Modes

| Mode | Description |
|------|-------------|
| **Library** | `compress(messages)` — inline in any Python/TS app |
| **Proxy** | HTTP proxy (`headroom proxy --port 8787`) — zero code changes |
| **Agent wrap** | `headroom wrap claude\|codex\|cursor\|aider\|copilot` — one command |
| **MCP server** | Tools: `headroom_compress`, `headroom_retrieve`, `headroom_stats` |

### 3.3 Extended Features

| Feature | Description |
|---------|-------------|
| **Cross-agent memory** | Shared vector store (SQLite + HNSW/sqlite-vec) across agents with auto-dedup |
| **SharedContext** | Compressed context passing between multi-agent workflows |
| **headroom learn** | Plugin-based failure mining — writes corrections to CLAUDE.md / AGENTS.md |
| **Image compression** | ML-based routing + OCR for image content |
| **IntelligentContext (TOIN)** | Score-based context fitting with learned importance |
| **Telemetry/Dashboard** | HTML dashboard, Prometheus metrics, OpenTelemetry export |

---

## 4. Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│  CLI Layer (click-based)                                     │
│  Commands: proxy, wrap, learn, mcp, perf, install, capture  │
├─────────────────────────────────────────────────────────────┤
│  Provider Layer                                              │
│  claude, codex, copilot, cursor, aider, openclaw, gemini    │
├─────────────────────────────────────────────────────────────┤
│  Pipeline Layer (canonical lifecycle)                        │
│  Setup → Pre-Start → Post-Start → Input Received →          │
│  Input Cached → Input Routed → Input Compressed →            │
│  Input Remembered → Pre-Send → Post-Send → Response Received│
├─────────────────────────────────────────────────────────────┤
│  Transform Layer                                             │
│  ContentRouter, SmartCrusher, CodeCompressor, Kompress,      │
│  CacheAligner, LogCompressor, SearchCompressor, DiffCompressor│
├─────────────────────────────────────────────────────────────┤
│  Storage Layer                                               │
│  CCR store, Memory (SQLite+HNSW), SharedContext              │
├─────────────────────────────────────────────────────────────┤
│  Rust Core (pyo3 extension)                                  │
│  headroom-core: transforms, tokenizer, CCR, relevance        │
│  headroom-proxy: standalone Rust proxy (axum)                │
│  headroom-py: Python bindings                                │
│  headroom-parity: test harness for Rust/Python equivalence   │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Integration Ecosystem

| Integration | Mechanism |
|-------------|-----------|
| Anthropic SDK | `withHeadroom(new Anthropic())` wrapper |
| OpenAI SDK | `withHeadroom(new OpenAI())` wrapper |
| Vercel AI SDK | `wrapLanguageModel({ model, middleware: headroomMiddleware() })` |
| LiteLLM | `litellm.callbacks = [HeadroomCallback()]` |
| LangChain | `HeadroomChatModel(your_llm)` |
| Agno | `HeadroomAgnoModel(your_model)` |
| Strands (AWS) | Model wrapping + hook-based tool output compression |
| ASGI apps | `app.add_middleware(CompressionMiddleware)` |
| MCP clients | `headroom mcp install` |

---

## 6. Infrastructure & Deployment

| Aspect | Technology |
|--------|-----------|
| **Build system (Python)** | maturin (Rust extension + Python source in one wheel) |
| **Build system (Rust)** | Cargo workspace |
| **Build system (TS SDK)** | tsup + vitest |
| **CI/CD** | GitHub Actions (ci.yml, rust.yml, release-please) |
| **Container** | Dockerfile (multi-stage), docker-compose (proxy + Qdrant + Neo4j) |
| **Docs** | Next.js site (Fumadocs), deployed on Vercel |
| **Package registry** | PyPI, npm, ghcr.io (Docker) |
| **Linting** | ruff (Python), clippy (Rust), ESLint (TS) |
| **Testing** | pytest (Python), cargo test (Rust), vitest (TS) |
| **Release** | release-please (conventional commits → changelog → version bump) |
| **Pre-commit** | pre-commit hooks, commitlint (conventional commits) |
| **Code coverage** | codecov.io |
| **Security** | GitGuardian, cargo-deny, secret scanning |

---

## 7. Configuration & Environment

| Variable / File | Purpose |
|-----------------|---------|
| `HEADROOM_MODE` | Compression mode (optimize, passthrough, etc.) |
| `HEADROOM_HOST` / `HEADROOM_PORT` | Proxy bind address |
| `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` | Provider credentials (passed through) |
| `OPENAI_TARGET_API_URL` | Custom upstream endpoint |
| `HEADROOM_MEMORY_ENABLED` | Enable cross-agent memory |
| `HEADROOM_CONTEXT_TOOL` | CLI context tool (rtk or lean-ctx) |
| `HEADROOM_EMBEDDER_RUNTIME` | Embedding runtime (pytorch_mps for Apple GPU) |
| `.env` / `.env.example` | Local environment variables |
| `headroom.toml` (planned) | Project-level configuration |

---

## 8. Domain-Specific Terminology

| Term | Meaning |
|------|---------|
| **Token** | Smallest unit of text processed by an LLM (~4 chars) |
| **Context window** | Maximum tokens an LLM can process in one request |
| **KV cache** | Provider-side key-value cache that speeds up repeated prefixes |
| **CCR** | Compress-Cache-Retrieve — reversible compression protocol |
| **TOIN** | Token-Optimal Intelligent Narrowing — score-based context fitting |
| **SmartCrusher** | JSON array/object compression via statistical analysis |
| **Agentic trace** | Sequence of tool calls and outputs from an AI agent |
| **MCP** | Model Context Protocol — standard for LLM tool/resource serving |

---

## 9. Tweaking for Another Domain

To adapt this repository template for a different domain:

1. **Change Section 1** — New project name, packages, tagline
2. **Change Section 2** — New problem/solution/differentiators
3. **Change Section 3** — Replace compression algorithms with your domain's processing algorithms
4. **Change Section 4** — Adjust architecture layers to match your processing pipeline
5. **Change Section 5** — Replace integrations with your domain's ecosystem
6. **Keep Section 6** — Infrastructure patterns are reusable (Rust+Python+TS, maturin, GitHub Actions)
7. **Adjust Section 7** — Environment variables for your domain
8. **Change Section 8** — Your domain's terminology

### Example: Adapting for "Audio Processing Layer for AI Agents"

- Replace compression algorithms with: noise reduction, speech-to-text, speaker diarization
- Replace content types with: WAV, MP3, FLAC, real-time streams
- Keep: proxy mode, library mode, MCP server, cross-agent memory pattern
- Replace: SmartCrusher → AudioCrusher, CodeCompressor → SpeechCompressor, etc.
