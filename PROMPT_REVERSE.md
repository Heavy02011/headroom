# PROMPT_REVERSE.md — Prompts & Skills to Regenerate the Repository

> **Purpose:** Each prompt below is a self-contained instruction that, when executed in sequence (or independently for specific components), generates the corresponding part of the repository. Feed these to an AI coding agent along with CONTEXT_REVERSE.md.

---

## Meta-Prompt: Full Repository Generation

```
You are regenerating a complete open-source project from a reverse-engineered specification.
Read CONTEXT_REVERSE.md for domain values (project name, problem, algorithms, architecture).
Read PLAN_REVERSE.md for the execution order and file structure.
Execute the prompts below in order. Each produces a testable increment.
Use the tech stack: Python 3.10+ (primary), Rust (core extension via pyo3/maturin), TypeScript (SDK).
```

---

## Skill 1: Project Scaffold

**Prompt:**
```
Create a new project with the following structure:
- Cargo workspace (resolver = "2") with members: crates/{name}-core, crates/{name}-proxy, crates/{name}-py, crates/{name}-parity
- Python package using maturin build-backend (pyproject.toml)
- The Python package name is "{package_name}" with entry point "{name}.cli:main"
- Workspace dependencies: serde, serde_json (preserve_order + arbitrary_precision + raw_value), bytes, thiserror, tracing, anyhow, clap (derive), tokio (macros + rt-multi-thread + signal), axum 0.7, tower, reqwest (json + rustls-tls), pyo3 (abi3-py310)
- Python dependencies: tiktoken, pydantic>=2, litellm, click, rich, opentelemetry-api
- Optional dependency groups: [proxy], [code], [ml], [memory], [relevance], [image], [mcp], [evals], [all]
- Release profile: strip=symbols, lto=thin, codegen-units=1
- Development tools: ruff (line-length 100), mypy, pytest, pre-commit
- Standard files: LICENSE (Apache-2.0), README.md, CONTRIBUTING.md, CODE_OF_CONDUCT.md, SECURITY.md, .gitignore, .gitattributes, rust-toolchain.toml, Makefile
```

---

## Skill 2: Content Router & Detection

**Prompt:**
```
Implement a content routing system with these components:

1. ContentDetector (headroom/transforms/content_detector.py):
   - Detect content type from raw text: JSON, code (by language), log output, search results, diff, HTML, prose
   - Use heuristics (JSON parse attempt, language markers, log patterns) + optional ML (magika)
   - Return ContentType enum with confidence score

2. ContentRouter (headroom/transforms/content_router.py):
   - Accept detected content type
   - Route to appropriate compressor (SmartCrusher for JSON, CodeCompressor for code, etc.)
   - Support fallback chains and custom routing rules
   - Implement as a Transform subclass

3. Base Transform (headroom/transforms/base.py):
   - Abstract base class with: compress(content, config) -> CompressResult
   - CompressResult: compressed_text, tokens_before, tokens_after, metadata
   - Support async and sync execution
```

---

## Skill 3: SmartCrusher (JSON Compression)

**Prompt:**
```
Implement SmartCrusher — a statistical JSON/array compression algorithm:

1. Python (headroom/transforms/smart_crusher.py):
   - Input: JSON string (typically arrays of objects from tool outputs)
   - Algorithm:
     a. Parse JSON, detect structure (array of dicts, nested objects, mixed)
     b. For arrays of dicts: identify schema (common keys), compute value distributions
     c. Keep: error entries, anomalies (statistical outliers), first/last items, items matching query relevance
     d. Remove: redundant entries that follow dominant patterns
     e. Emit: compressed representation with summary header ("N items, showing K representative + anomalies")
   - Preserve insertion order (IndexMap semantics)
   - Target: 70-90% compression on typical tool outputs

2. Rust port (crates/headroom-core/src/transforms/smart_crusher.rs):
   - Same algorithm, using serde_json with preserve_order
   - Expose via pyo3 bindings
   - Must produce byte-identical output to Python implementation (parity tests)
```

---

## Skill 4: CodeCompressor (AST-Aware)

**Prompt:**
```
Implement CodeCompressor — AST-aware code compression:

1. headroom/transforms/code_compressor.py:
   - Use tree-sitter (via tree-sitter-language-pack) and/or ast-grep-cli
   - Supported languages: Python, JavaScript/TypeScript, Go, Rust, Java, C++
   - Compression strategy per node type:
     * KEEP: imports, function/method signatures, type annotations, class declarations, error handling
     * SUMMARIZE: function bodies (replace with "// ... N lines"), large string literals
     * REMOVE: comments (keep TODOs), excessive whitespace, redundant brackets
   - Produce valid syntax skeleton that an LLM can reason about
   - Configurable retention level (signatures-only, signatures+key-logic, full)
   - Thread-safe (no global state)
```

---

## Skill 5: CacheAligner (Prefix Stabilization)

**Prompt:**
```
Implement CacheAligner — stabilize message prefixes for LLM provider KV cache hits:

1. headroom/transforms/cache_aligner.py:
   - Problem: LLM providers cache the KV state of processed prefixes. If prefixes change between requests, cache misses occur ($$).
   - Solution: Reorder and normalize message content so that stable content (system prompt, tool definitions, recent history) forms a consistent prefix.
   - Algorithm:
     a. Identify stable vs volatile message segments
     b. Place stable content first (maximizes cache hit probability)
     c. Normalize whitespace/formatting in stable prefix zone
     d. Track prefix hash to detect drift
   - Emit cache-alignment metadata (prefix_hash, aligned_tokens, estimated_cache_savings)
```

---

## Skill 6: CCR (Compress-Cache-Retrieve)

**Prompt:**
```
Implement CCR — reversible compression with local cache:

1. Core concept: Compress content but store originals locally. Inject a tool into the LLM's toolset so it can retrieve originals on demand.

2. Components:
   - batch_store.py: SQLite-backed store for original content, keyed by hash. TTL-based expiry.
   - batch_processor.py: Compress N items, store originals, return compressed + retrieval keys.
   - context_tracker.py: Track what's been compressed in this session/conversation.
   - tool_injection.py: Add "headroom_retrieve(key)" to the tools array sent to the LLM.
   - response_handler.py: Intercept LLM tool calls to "headroom_retrieve", return the original content.
   - mcp_server.py: Expose as MCP tools: headroom_compress, headroom_retrieve, headroom_stats.

3. Protocol:
   - Compressed output includes markers: "[compressed: {hash} — call headroom_retrieve to expand]"
   - LLM sees compressed version, can request original if needed
   - Most of the time, LLM doesn't need the original (compression preserves key information)
```

---

## Skill 7: Proxy Server

**Prompt:**
```
Implement an OpenAI/Anthropic-compatible HTTP proxy:

1. Python proxy (headroom/proxy/server.py):
   - FastAPI + uvicorn ASGI app
   - Routes: POST /v1/chat/completions, POST /v1/responses, POST /messages (Anthropic)
   - Pipeline: receive request → compress messages → forward to upstream → stream response back
   - Support SSE streaming, WebSocket (for Codex /v1/responses)
   - Health endpoints: /healthz, /readyz
   - Dashboard: /dashboard (HTML stats page)
   - Metrics: /metrics (Prometheus format)
   - Auth passthrough: forward Authorization headers to upstream

2. Rust proxy (crates/headroom-proxy/):
   - axum-based HTTP server
   - Same routes and behavior as Python proxy
   - Native Bedrock support (AWS SigV4 signing)
   - Native Vertex AI support (GCP ADC bearer tokens)
   - Zero-copy SSE stream processing
   - Binary: `headroom-proxy --port 8787 --host 0.0.0.0`

3. Both proxies must:
   - Be drop-in replacements (change base URL, everything works)
   - Support all OpenAI and Anthropic API formats
   - Handle streaming and non-streaming responses
   - Inject CCR retrieve tool transparently
   - Track and report token savings
```

---

## Skill 8: CLI

**Prompt:**
```
Implement a CLI using click (Python):

Commands:
- `headroom proxy [--port 8787] [--host 127.0.0.1] [--memory] [--code-graph]` — start proxy
- `headroom wrap <agent>` — wrap an agent (claude, codex, cursor, aider, copilot, gemini)
  - Sets environment variables, starts proxy, launches agent
- `headroom learn [--agent claude|codex|gemini]` — mine failed sessions, write corrections
- `headroom mcp install|run` — install/run MCP server
- `headroom perf` — show savings report (table of recent requests)
- `headroom install` — interactive setup wizard
- `headroom memory list|clear|export` — memory management
- `headroom capture start|stop|replay` — network capture for debugging
- `headroom copilot-auth login` — GitHub Copilot OAuth flow

Each `wrap` subcommand:
- Knows agent-specific env vars and config file locations
- Starts proxy in background
- Sets OPENAI_BASE_URL / ANTHROPIC_BASE_URL to point at local proxy
- Launches the agent process
- Shows savings on exit
```

---

## Skill 9: Memory System

**Prompt:**
```
Implement a cross-agent memory system:

1. Core (headroom/memory/):
   - MemoryStore interface: put(fact, metadata), get(query, top_k), delete(id)
   - Vector search using sqlite-vec (SQLite extension) or HNSW
   - Embedding: fastembed (BAAI/bge-small-en-v1.5, ONNX, 384 dims)
   - Per-project isolation (memories scoped to project directory)
   - Auto-dedup: don't store near-duplicate facts
   - Agent provenance: track which agent stored which fact

2. Backends:
   - SQLite + sqlite-vec (default, zero external dependencies)
   - Qdrant (optional, for larger deployments)
   - Neo4j (optional, for graph-based relationships)

3. Integration:
   - Proxy injects relevant memories into system prompt
   - Memory budget: limit injected memory tokens
   - Traffic learner: automatically extract facts from conversations
   - Sync: share memories between agents via filesystem

4. SharedContext (headroom/shared_context.py):
   - Compressed context passing between agents in multi-agent workflows
   - Put/get interface with automatic compression
```

---

## Skill 10: TypeScript SDK

**Prompt:**
```
Create a TypeScript SDK (sdk/typescript/):

1. Package: "headroom-ai" on npm
2. Build: tsup (ESM + CJS dual output)
3. Test: vitest

4. Core API:
   - compress(messages, options) → CompressResult
   - HeadroomClient class (wraps fetch-based HTTP calls to proxy)

5. Adapters:
   - withHeadroom(anthropicClient) — wraps Anthropic SDK
   - withHeadroom(openaiClient) — wraps OpenAI SDK
   - headroomMiddleware() — Vercel AI SDK middleware
   - HeadroomCallback — LiteLLM-style callback

6. SharedContext:
   - put(key, data) / get(key) for inter-agent context

7. Type definitions:
   - Full TypeScript types for all API surfaces
   - Exported as .d.ts
```

---

## Skill 11: Learn System (Failure Mining)

**Prompt:**
```
Implement `headroom learn` — automated failure mining:

1. Scanner (headroom/learn/scanner.py):
   - Find agent session logs (Claude: ~/.claude/sessions, Codex: ~/.codex/sessions)
   - Parse session format (JSONL, markdown, etc.)
   - Identify failed sessions (error exits, user corrections, repeated attempts)

2. Analyzer (headroom/learn/analyzer.py):
   - Categorize failures: wrong tool use, missing context, hallucination, wrong file, etc.
   - Extract patterns: "when X happens, agent does Y but should do Z"
   - Score by frequency and impact

3. Writer (headroom/learn/writer.py):
   - Generate corrections in agent-native format
   - Claude: append to CLAUDE.md
   - Codex: append to AGENTS.md
   - Gemini: append to GEMINI.md
   - Format: actionable rules ("When doing X, always Y")

4. Plugin system (headroom/learn/plugins/):
   - Each agent has a plugin that knows its log format and config file
   - Registry for plugin discovery
```

---

## Skill 12: Rust Core (pyo3 Extension)

**Prompt:**
```
Implement the Rust core library with Python bindings:

1. crates/headroom-core/:
   - src/transforms/ — SmartCrusher, compression policy
   - src/tokenizer/ — fast tokenization (tiktoken-compatible)
   - src/ccr/ — CCR store and retrieval
   - src/relevance/ — embedding-based relevance scoring
   - src/signals/ — keyword/pattern signal detection
   - src/cache_control.rs — cache alignment logic
   - src/auth_mode.rs — auth mode detection
   - src/compression_policy.rs — when to compress decisions

2. crates/headroom-py/:
   - pyo3 bindings exposing Rust functions to Python
   - Module: headroom._core
   - Functions: crush_json(), compress_code(), count_tokens(), compute_relevance()
   - Must be built via maturin (extension-module feature)

3. crates/headroom-parity/:
   - Test harness that runs same inputs through Python and Rust
   - Compares outputs byte-for-byte
   - Fixture-based: tests/parity/fixtures/ contains input/expected pairs

4. Key invariants:
   - Rust output must be byte-identical to Python output (realignment invariant)
   - serde_json preserve_order + arbitrary_precision ensures numeric fidelity
   - No precision loss through f64 (numbers stay as string tokens)
```

---

## Skill 13: CI/CD & Release

**Prompt:**
```
Set up CI/CD:

1. GitHub Actions workflows:
   - ci.yml: lint (ruff, clippy, ESLint) + test (pytest, cargo test, vitest) + build
   - rust.yml: Rust-specific (fmt --check, clippy, test, maturin build)
   - release-please.yml: automated changelog + version bump on merge to main
   - publish.yml: triggered on release — publish to PyPI + npm + ghcr.io

2. Release automation:
   - release-please with conventional commits
   - .release-please-config.json + .release-please-manifest.json
   - scripts/version-sync.py — keep Cargo.toml, pyproject.toml, package.json aligned

3. Container:
   - Multi-stage Dockerfile (builder + runtime)
   - docker-compose.yml: proxy + Qdrant + Neo4j
   - docker-bake.hcl: multi-platform (amd64 + arm64)

4. Quality gates:
   - codecov.yml: coverage thresholds
   - deny.toml: license allowlist + vulnerability audit
   - .gitguardian.yaml: secret detection
   - .pre-commit-config.yaml: pre-commit hooks
```

---

## Skill 14: Documentation

**Prompt:**
```
Create comprehensive documentation:

1. README.md:
   - ASCII art banner
   - Badges (CI, coverage, PyPI, npm, model, license, docs)
   - What it does (bullet list of modes)
   - Architecture diagram (ASCII)
   - Quick start (3 commands)
   - Proof table (real workloads with before/after/savings)
   - Accuracy benchmarks table
   - Agent compatibility matrix
   - Integration table
   - Installation (all methods)
   - Contributing section

2. llms.txt:
   - Machine-readable project index for AI agents
   - Install commands, entry points, how-it-works, SDK integrations, operations

3. Docs site (docs/):
   - Next.js + Fumadocs framework
   - Pages: quickstart, installation, proxy, MCP, memory, architecture, API reference
   - Auto-generated from MDX content files

4. Wiki (wiki/):
   - Detailed technical docs: architecture, API, CLI, configuration, troubleshooting
   - Integration guides per framework
```

---

## Skill 15: Testing Strategy

**Prompt:**
```
Implement comprehensive testing:

1. Unit tests (tests/):
   - One test file per module (test_smart_crusher.py, test_code_compressor.py, etc.)
   - Fixtures in tests/fixtures/ (real-world JSON, code, logs)
   - Parametrized tests for multiple content types and edge cases
   - Mock external services (LLM APIs, vector DBs)

2. Parity tests (tests/parity/):
   - Golden file comparison: Python output vs Rust output
   - Fixture recording: scripts/record_fixtures.py
   - CI enforces parity on every PR

3. Integration tests:
   - Proxy end-to-end (start server, send request, verify response)
   - Memory persistence (write, restart, read)
   - CCR round-trip (compress, retrieve, verify identical)

4. Benchmarks (benchmarks/):
   - Latency: compression time per content type
   - Compression ratio: real-world workloads
   - Accuracy: benchmark suites (GSM8K, TruthfulQA, SQuAD v2, BFCL)
   - Agent cost: real session replay with token counting

5. E2E (e2e/):
   - Docker-based full-stack tests
   - Real provider calls (gated behind API key availability)
```

---

## Execution Notes

1. **Order matters:** Skills 1-6 form the core. Skills 7-8 add delivery modes. Skills 9-15 add ecosystem.
2. **Each skill is independently testable** — verify before proceeding.
3. **CONTEXT_REVERSE.md is the variable** — change it for a different domain, keep these prompts.
4. **The Rust/Python duality** is the hardest part — establish parity tests early (Skill 3) and enforce on every change.
5. **Start with Python-only** — Rust ports come after Python is proven correct.
