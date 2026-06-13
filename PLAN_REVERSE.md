# PLAN_REVERSE.md — Step-by-Step Plan to Regenerate the Repository

> **Purpose:** Execute these steps (using CONTEXT_REVERSE.md for domain values) to regenerate the entire Headroom repository from scratch. Each phase produces a working, testable increment.

---

## Phase 0: Project Scaffold (Day 1)

### 0.1 Repository Setup
- [ ] Create GitHub repository with Apache-2.0 license
- [ ] Initialize Cargo workspace with `resolver = "2"`
- [ ] Initialize Python project with `pyproject.toml` (maturin build-backend)
- [ ] Create `.gitignore`, `.gitattributes`, `.dockerignore`
- [ ] Set up `rust-toolchain.toml` (edition 2021, rust-version 1.80)
- [ ] Create `Makefile` with standard targets (test, build, lint, fmt)

### 0.2 Workspace Members
- [ ] `crates/headroom-core/` — shared Rust library (transforms, tokenizer, CCR)
- [ ] `crates/headroom-proxy/` — standalone Rust proxy binary (axum)
- [ ] `crates/headroom-py/` — pyo3 Python bindings (cdylib)
- [ ] `crates/headroom-parity/` — Rust/Python output equivalence test harness

### 0.3 Python Package Structure
- [ ] `headroom/__init__.py` — public API (`HeadroomClient`, `compress`, etc.)
- [ ] `headroom/_version.py` — version string
- [ ] `headroom/config.py` — pydantic-based configuration
- [ ] `headroom/exceptions.py` — custom exception hierarchy
- [ ] `headroom/utils.py` — shared utilities
- [ ] `headroom/py.typed` — PEP 561 marker

### 0.4 Development Environment
- [ ] `.devcontainer/devcontainer.json` — VS Code devcontainer
- [ ] `.devcontainer/Dockerfile` — devcontainer image
- [ ] `.devcontainer/post-create.sh` — post-create setup script
- [ ] `.pre-commit-config.yaml` — pre-commit hooks (ruff, clippy, commitlint)
- [ ] `.commitlintrc.json` — conventional commits enforcement

---

## Phase 1: Core Compression Pipeline (Days 2–5)

### 1.1 Content Detection & Routing
- [ ] `headroom/transforms/content_detector.py` — detect content type (JSON, code, log, prose, diff)
- [ ] `headroom/transforms/content_router.py` — route to appropriate compressor
- [ ] `headroom/transforms/base.py` — abstract `Transform` base class

### 1.2 SmartCrusher (JSON Compression)
- [ ] `headroom/transforms/smart_crusher.py` — statistical JSON compression
  - Array deduplication (keep anomalies, errors, representative samples)
  - Nested object flattening
  - Key/value frequency analysis
  - Configurable retention policies
- [ ] Rust port: `crates/headroom-core/src/transforms/smart_crusher.rs`

### 1.3 CodeCompressor (AST-Aware)
- [ ] `headroom/transforms/code_compressor.py` — tree-sitter based
  - Preserve: imports, function signatures, type annotations, error paths
  - Remove: function bodies (summarize), comments, whitespace
  - Languages: Python, JavaScript, Go, Rust, Java, C++

### 1.4 Text/Log/Search Compression
- [ ] `headroom/transforms/kompress_compressor.py` — HuggingFace model integration
- [ ] `headroom/transforms/log_compressor.py` — pattern dedup for build logs
- [ ] `headroom/transforms/search_compressor.py` — relevance-scored filtering
- [ ] `headroom/transforms/diff_compressor.py` — diff-aware compression

### 1.5 Pipeline Orchestration
- [ ] `headroom/pipeline.py` — canonical lifecycle stages enum + event protocol
- [ ] `headroom/transforms/pipeline.py` — transform chain execution
- [ ] `headroom/transforms/cache_aligner.py` — prefix stabilization for KV cache hits
- [ ] `headroom/transforms/adaptive_sizer.py` — dynamic token budget allocation
- [ ] `headroom/transforms/tag_protector.py` — protect XML/special tags from compression

### 1.6 Tokenizer
- [ ] `headroom/tokenizer.py` — tiktoken-based token counting
- [ ] `headroom/tokenizers/` — model-specific tokenizer configs
- [ ] Rust port: `crates/headroom-core/src/tokenizer/`

---

## Phase 2: CCR — Reversible Compression (Days 6–7)

### 2.1 Core CCR
- [ ] `headroom/ccr/__init__.py` — CCR public API
- [ ] `headroom/ccr/batch_store.py` — local storage for original content
- [ ] `headroom/ccr/batch_processor.py` — batch compression with store-back
- [ ] `headroom/ccr/context_tracker.py` — track what's been compressed in session
- [ ] `headroom/ccr/tool_injection.py` — inject `headroom_retrieve` tool into LLM toolset
- [ ] `headroom/ccr/response_handler.py` — handle retrieve tool calls from LLM
- [ ] `headroom/ccr/mcp_server.py` — MCP server for CCR operations

---

## Phase 3: Compress API & Client (Days 8–9)

### 3.1 One-Function API
- [ ] `headroom/compress.py` — `compress(messages, model=...)` → CompressResult
- [ ] `headroom/client.py` — `HeadroomClient` wrapping any LLM client
- [ ] `headroom/hooks.py` — compression hooks for SDK wrappers

### 3.2 Provider Wrappers
- [ ] `headroom/providers/claude/` — Claude Code environment shaping
- [ ] `headroom/providers/codex/` — Codex/OpenAI runtime
- [ ] `headroom/providers/copilot/` — GitHub Copilot CLI subscription mode
- [ ] `headroom/providers/cursor/` — Cursor config generation
- [ ] `headroom/providers/openclaw/` — OpenClaw ContextEngine plugin
- [ ] `headroom/providers/registry.py` — provider discovery and dispatch

---

## Phase 4: Proxy Server (Days 10–13)

### 4.1 Python Proxy (ASGI)
- [ ] `headroom/proxy/server.py` — FastAPI/uvicorn ASGI application
- [ ] `headroom/proxy/handlers/` — route handlers (chat completions, responses)
- [ ] `headroom/proxy/interceptors/` — request/response interceptors
- [ ] `headroom/proxy/models.py` — request/response models
- [ ] `headroom/proxy/auth_mode.py` — authentication mode handling
- [ ] `headroom/proxy/compression_decision.py` — should-compress logic
- [ ] `headroom/proxy/memory_handler.py` — memory integration in proxy
- [ ] `headroom/proxy/cost.py` — cost tracking and reporting
- [ ] `headroom/proxy/warmup.py` — model/tokenizer preloading

### 4.2 Streaming Support
- [ ] SSE (Server-Sent Events) forwarding with compression
- [ ] WebSocket proxy for `/v1/responses` (Codex)
- [ ] Streaming usage parsing and header forwarding

### 4.3 Rust Proxy (axum)
- [ ] `crates/headroom-proxy/src/main.rs` — binary entry point
- [ ] `crates/headroom-proxy/src/proxy.rs` — request forwarding
- [ ] `crates/headroom-proxy/src/handlers/` — route handlers
- [ ] `crates/headroom-proxy/src/compression/` — Rust compression engine
- [ ] `crates/headroom-proxy/src/sse/` — SSE stream processing
- [ ] `crates/headroom-proxy/src/websocket.rs` — WebSocket upgrade
- [ ] `crates/headroom-proxy/src/bedrock/` — AWS Bedrock SigV4 signing
- [ ] `crates/headroom-proxy/src/vertex/` — GCP Vertex AI auth
- [ ] `crates/headroom-proxy/src/observability/` — metrics + tracing

---

## Phase 5: CLI (Days 14–15)

### 5.1 Click-Based CLI
- [ ] `headroom/cli/__init__.py` — main entry point
- [ ] `headroom/cli/main.py` — top-level click group
- [ ] `headroom/cli/proxy.py` — `headroom proxy` command
- [ ] `headroom/cli/wrap.py` — `headroom wrap <agent>` command
- [ ] `headroom/cli/learn.py` — `headroom learn` failure mining
- [ ] `headroom/cli/mcp.py` — `headroom mcp install/run`
- [ ] `headroom/cli/perf.py` — `headroom perf` savings report
- [ ] `headroom/cli/install.py` — `headroom install` setup wizard
- [ ] `headroom/cli/memory.py` — `headroom memory` management
- [ ] `headroom/cli/capture.py` — `headroom capture` network recording
- [ ] `headroom/cli/copilot_auth.py` — `headroom copilot-auth login`

---

## Phase 6: Memory System (Days 16–18)

### 6.1 Core Memory
- [ ] `headroom/memory/core.py` — MemoryStore interface
- [ ] `headroom/memory/models.py` — memory entry models
- [ ] `headroom/memory/storage_router.py` — route to SQLite/Qdrant/Neo4j
- [ ] `headroom/memory/factory.py` — memory backend factory
- [ ] `headroom/memory/sync.py` — cross-agent memory sync
- [ ] `headroom/memory/tracker.py` — usage tracking
- [ ] `headroom/memory/traffic_learner.py` — learn from traffic patterns

### 6.2 Backends
- [ ] SQLite + sqlite-vec (default, zero-dependency vector search)
- [ ] Qdrant (optional, via docker-compose)
- [ ] Neo4j (optional, graph relationships)
- [ ] `headroom/memory/backends/` — backend implementations

### 6.3 SharedContext
- [ ] `headroom/shared_context.py` — compressed inter-agent context

---

## Phase 7: Learn System (Days 19–20)

### 7.1 Failure Mining
- [ ] `headroom/learn/__init__.py` — public API
- [ ] `headroom/learn/scanner.py` — scan session logs for failures
- [ ] `headroom/learn/analyzer.py` — categorize failure patterns
- [ ] `headroom/learn/writer.py` — write corrections to agent config files
- [ ] `headroom/learn/plugins/` — agent-specific plugins (Claude, Codex, Gemini)
- [ ] `headroom/learn/registry.py` — plugin discovery

---

## Phase 8: TypeScript SDK (Days 21–22)

### 8.1 SDK Package
- [ ] `sdk/typescript/package.json` — npm package config
- [ ] `sdk/typescript/src/index.ts` — public exports
- [ ] `sdk/typescript/src/compress.ts` — `compress()` function
- [ ] `sdk/typescript/src/client.ts` — HeadroomClient class
- [ ] `sdk/typescript/src/hooks.ts` — SDK adapter hooks
- [ ] `sdk/typescript/src/adapters/` — Anthropic, OpenAI, Vercel AI SDK adapters
- [ ] `sdk/typescript/src/shared-context.ts` — SharedContext for multi-agent
- [ ] `sdk/typescript/src/types.ts` — TypeScript type definitions
- [ ] `sdk/typescript/tsup.config.ts` — build config
- [ ] `sdk/typescript/vitest.config.ts` — test config

---

## Phase 9: Plugins (Days 23–24)

### 9.1 Agent Hooks Plugin
- [ ] `plugins/headroom-agent-hooks/` — lifecycle hooks for agents
- [ ] `.claude-plugin` manifest for Claude Code

### 9.2 Hermes Plugin
- [ ] `plugins/hermes/` — `headroom_retrieve` MCP tool server

### 9.3 OpenClaw Plugin
- [ ] `plugins/openclaw/` — OpenClaw ContextEngine integration (TypeScript)
- [ ] `openclaw.plugin.json` — plugin manifest

### 9.4 OAuth2 Plugin
- [ ] `plugins/headroom-oauth2/` — OAuth2 authentication plugin

---

## Phase 10: Testing (Days 25–28)

### 10.1 Unit Tests
- [ ] `tests/` — comprehensive pytest test suite (300+ test files)
- [ ] `tests/conftest.py` — shared fixtures
- [ ] `tests/fixtures/` — test data (JSON, code samples, logs)
- [ ] Coverage targets: transforms, proxy, memory, CCR, CLI, providers

### 10.2 Parity Tests
- [ ] `tests/parity/` — Rust/Python output equivalence tests
- [ ] `tests/parity/fixtures/` — golden files for comparison

### 10.3 Integration & E2E
- [ ] `e2e/` — end-to-end tests (Docker-based, real providers)
- [ ] `benchmarks/` — performance benchmarks (latency, compression ratio)

### 10.4 Rust Tests
- [ ] `crates/headroom-core/tests/` — Rust unit + integration tests
- [ ] `crates/headroom-proxy/tests/` — proxy integration tests
- [ ] `crates/headroom-core/benches/` — criterion benchmarks

---

## Phase 11: Documentation (Days 29–31)

### 11.1 Code Documentation
- [ ] `README.md` — comprehensive with badges, examples, benchmarks
- [ ] `CONTRIBUTING.md` — contribution guidelines
- [ ] `CODE_OF_CONDUCT.md` — community standards
- [ ] `SECURITY.md` — security policy
- [ ] `CHANGELOG.md` — auto-generated via release-please
- [ ] `llms.txt` — LLM-readable project index

### 11.2 Docs Site (Next.js + Fumadocs)
- [ ] `docs/` — Next.js documentation site
- [ ] `docs/content/` — MDX documentation pages
- [ ] `docs/source.config.ts` — content source config
- [ ] `mkdocs.yml` — alternative docs config

### 11.3 Wiki
- [ ] `wiki/` — detailed technical documentation
- [ ] Architecture, API, CLI, configuration, integrations, troubleshooting

### 11.4 Examples
- [ ] `examples/` — working code examples
- [ ] Jupyter notebooks, demo scripts, deployment configs

---

## Phase 12: CI/CD & Release (Days 32–33)

### 12.1 GitHub Actions
- [ ] `.github/workflows/ci.yml` — main CI (lint, test, build)
- [ ] `.github/workflows/rust.yml` — Rust-specific CI
- [ ] `.github/workflows/release-please.yml` — automated releases
- [ ] `.github/workflows/publish.yml` — PyPI + npm publishing

### 12.2 Release Automation
- [ ] `.release-please-config.json` — release-please configuration
- [ ] `.release-please-manifest.json` — version manifest
- [ ] `scripts/version-sync.py` — keep versions aligned across packages
- [ ] `scripts/sync-plugin-versions.py` — plugin version sync

### 12.3 Container
- [ ] `Dockerfile` — multi-stage production image
- [ ] `docker-compose.yml` — full stack (proxy + Qdrant + Neo4j)
- [ ] `docker-bake.hcl` — multi-platform build

---

## Phase 13: Observability & Dashboard (Day 34)

- [ ] `headroom/dashboard/` — HTML dashboard templates
- [ ] `headroom/observability/` — OpenTelemetry instrumentation
- [ ] `headroom/telemetry/` — internal telemetry collection
- [ ] `sql/` — dashboard/telemetry SQL schemas
- [ ] Prometheus metrics endpoint at proxy `/metrics`

---

## Phase 14: Polish & Hardening (Days 35–37)

### 14.1 Security
- [ ] `.gitguardian.yaml` — secret detection config
- [ ] `deny.toml` — cargo-deny (license + vulnerability audit)
- [ ] `codecov.yml` — coverage thresholds

### 14.2 Performance
- [ ] Profile release settings (LTO, strip, codegen-units=1)
- [ ] CI profile (fast compilation for test wheels)
- [ ] Benchmark suite with real-world workloads

### 14.3 Multi-platform
- [ ] macOS, Linux, Windows wheel builds
- [ ] ARM64 + x86_64 Docker images
- [ ] Platform-specific auth (Keychain, Credential Manager, Secret Service)

---

## Verification Checklist

After regeneration, verify:

- [ ] `pip install -e ".[dev]"` succeeds
- [ ] `cargo test --workspace` passes
- [ ] `pytest` passes
- [ ] `headroom proxy --port 8787` starts and responds to `/readyz`
- [ ] `headroom wrap claude` produces correct environment setup
- [ ] `from headroom import compress; compress(messages, model="gpt-4o")` works
- [ ] TypeScript SDK: `npm test` passes
- [ ] Docker: `docker compose up` starts all services
- [ ] Rust proxy: `cargo run -p headroom-proxy` binds and proxies
- [ ] CCR: compressed content is retrievable
- [ ] Memory: facts persist across sessions
- [ ] Dashboard: accessible at proxy `/dashboard`
