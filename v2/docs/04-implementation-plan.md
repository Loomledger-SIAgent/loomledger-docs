# 04 Implementation plan

_Updated 17 Oct 2026. Incorporates the confirmed Python + PostgreSQL stack, multi-provider Decisions API (`typesafe/jev-1.14`), and cryptographic audit core ([09 Audit and tracing](09-audit-and-tracing.md))._

Estimates assume about 10 hours per week. Treat them as rough milestones where Phases 0–2 already form a standalone, demonstrable product.

## Phases

### Phase 0 (3–4 days): foundations & cryptographic primitives
- Repo setup: `uv` (with hash-pinned lockfile), `ruff`, `pytest`, `docker-compose.yml` with PostgreSQL 17
- Cryptographic receipt primitives (`loomledger/audit/crypto.py`): RFC 8785 JSON canonicalization, SHA-256 content addressing, and Ed25519 key generation, signing, and verification (`key_id` support)
- Pydantic v2 schemas for `Envelope`, `Capabilities`, `NodeSpec`, `AgentSpec`, and `GraphSpec` (enforcing typed `output_schema` by default, with `TextPayload` opt-out and rejected `latest` aliases)

### Phase 1 (weeks 1–2): model layer & single-node runtime
- `ModelNode` contract (`GenerateRequest`, `DecideRequest`, `EmbedRequest`) and versioned `catalog/models.yaml` (capabilities + pricing table)
- Four MVP adapters:
  1. `fake.py` (scripted deterministic responses for `generate`, `decide`, and `embed`)
  2. `openai_compat.py` (covers OpenAI, Ollama, LM Studio, OpenRouter chat)
  3. `anthropic.py` (native Messages API)
  4. `decisions.py` (TypeSafe System One API `/v1/systemone` + OpenRouter Decisions API `/api/alpha/decisions` supporting `typesafe/jev-1.14` with `noul`, `choice`, `score`, and capability-gated `box`)
- 6-stage middleware pipeline: `budget` → `redact` → `retry/timeout` → `fallback` → `cost` → `audit`
- Bare `agent` tool loop with permissioned tools (`allow / ask / deny`) and sandboxed execution wrapper
- Shared adapter contract test suite running 100% offline with `FakeModel` and recorded HTTP fixtures (`respx`)

### Phase 2 (weeks 2–3): audit and tracing core
Build this before the UI because it is the core differentiator.
- PostgreSQL migrations for append-only `audit_events` (with `UPDATE`/`DELETE` prevention triggers and `run_seq` + `ledger_seq` counters) and content-addressed `payload_blobs`
- Pre-hash PII/secret redaction middleware
- Isolated OpenTelemetry GenAI (`v1.42.0`) span and metric exporter (`loomledger/audit/otel_mapping.py`), including loop-detection metrics (`inference_calls` vs. `tool_calls`)
- Deterministic `ReplayAdapter` keyed by canonical request hash
- Pluggable `ChainAnchor` (local signed checkpoint files first; S3 Object Lock WORM / Sigstore Rekor ready)
- CLI commands: `loomledger run`, `loomledger trace`, `loomledger verify`, and `loomledger replay`

**Milestone:** A traced, cryptographically verifiable, replayable run demonstrable end-to-end from the CLI.

### Phase 3 (week 4): multi-node graph execution
- DAG and conditional graph executor over typed nodes (`agent`, `llm`, `decide`, `embed`, `tool`)
- Confidence-gated routing using `decide` nodes (e.g., `typesafe/jev-1.14` handling fast routing or tool-call reversibility checks, escalating to a frontier LLM or human when confidence < threshold)
- Per-run token and USD cost budgets, bounded retries, and capability-checked model fallbacks
- Two-layer safety gates (deterministic policy / `decide` check + human approval gate) for destructive actions

### Phase 4 (weeks 5–6): FastAPI service & React Flow canvas
- FastAPI endpoints for specs, runs, approvals, traces, and chain verification
- React + TypeScript + React Flow canvas that reads and writes the exact YAML/JSON graph spec
- Live port type-checking and model capability warnings in the canvas
- Run inspector: trace timeline, confidence distributions, verification status badge, and per-node cost breakdown

### Phase 5 (weeks 7–8): meta-agent orchestrator
- Meta-agent reads a set of `NodeSpec` definitions and drafts a `GraphSpec` as schema-validated output
- Graph type-checker and capability validator verify the draft before presenting it to the user for review and approval
- Benchmark generated graphs against hand-built workflows on an evaluation suite

### Later (post-MVP)
- Durable execution via DBOS (`dbos-transact-py`) on the existing PostgreSQL instance
- Production S3 Object Lock (WORM) and Sigstore Rekor chain-head anchoring adapters
- Multi-tenancy, RBAC, SSO, and additional native provider adapters

## Principles

1. **The spec is the source of truth.** The UI is just a view.
2. **Keep the graph model simple:** nodes, typed edges, conditions. No new workflow DSL.
3. **Cryptographic audit + privacy by separation:** hash-chain and Ed25519-sign canonical event metadata (`audit_events`) while storing redacted payloads by `sha256` in `payload_blobs` so retention purges never break the chain.
4. **API keys are secrets:** `secret://` references only; never in specs, blobs, or spans.
5. **Tools are the main security risk:** sandbox shell, file, and MCP tools from the start, and never rely on a decision model alone as the sole gate on destructive actions.
6. **Build the spec, runtime, and audit ledger before the UI.**
