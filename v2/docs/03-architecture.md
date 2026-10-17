# 03 Architecture

_Updated 17 Oct 2026. Stack and core architecture decisions are confirmed (see [07 Decisions](07-decisions-and-open-questions.md))._

## Overview

```
 Canvas (React Flow) ──reads/writes──► Specs (YAML/JSON, versioned)
                                           │
                                           ▼
 API (FastAPI) ──► Executor ──► Middleware ──► Adapters ──► LLMs / Decision models / Tools
                      │
                      ▼
          Audit store (RFC 8785 hash-chained + Ed25519-signed)
          + Content-addressed payload blobs + OpenTelemetry GenAI traces
```

**The spec is the source of truth. The UI is a view of it.** If the canvas and the spec ever disagree, the spec wins.

## Terminology: Nodes vs. Agents

In `loomledger`, everything wired into a graph is a **Node** with typed input and output schemas:

| Node kind (`kind`) | Description | Example | Typical output |
|---|---|---|---|
| `agent` | LLM node running a permissioned multi-step tool loop (`max_steps > 1`) | Claude, GPT, local Qwen with tools | Schema-validated JSON (`output_schema`) + tool audit trail |
| `llm` | Single-shot generative LLM call (no tool loop) | Summarizer, extractor | Schema-validated JSON or explicit `TextPayload` |
| `decide` | System One / Decisions API model returning calibrated probabilities | `typesafe/jev-1.14`, `openai/gpt-6-luna-decisions` | `noul` probability, `choice` + distribution, `score`, or normalized `box` + `confidence` |
| `embed` | Embedding model | Local or API embeddings | Typed vector payload |
| `tool` | Deterministic code or sandboxed MCP tool node | Plain Python function, MCP tool | Schema-validated output |

A non-LLM such as `typesafe/jev-1.14` is a `decide` node (single structured judgment, no loop, does not generate text). Reserve the term **agent** strictly for LLM nodes that run a tool loop.

### Why typed ports

1. **The canvas validates connections.** If Node A outputs `ClaimSummary` and Node B expects `RiskQuestion`, the link is refused or a schema adapter is suggested.
2. **The meta-agent orchestrator becomes reliable.** It matches declared Pydantic schemas rather than guessing from prose role descriptions.
3. **Node-to-node messages are neutral typed payloads.** Raw chat histories never cross node boundaries.
4. **Audit logging falls out of the envelope.** One canonical event shape covers `agent`, `llm`, `decide`, `embed`, and `tool` nodes.

## Model contract

See [08 Model layer](08-model-layer.md) for the complete protocol (`GenerateRequest`, `DecideRequest`, `EmbedRequest`, capability checks, middleware pipeline, and the System One / Decisions API adapter).

```python
class ModelNode(Protocol):
    id: ModelId                       # provider + name + pinned version
    capabilities: Capabilities        # kinds, decision_primitives, tools, structured_output, vision, default_context, max_context

    async def invoke(self, req: Request, ctx: CallContext) -> Response: ...
```

Capabilities are validated against `catalog/models.yaml` at spec-load time and before every invocation, so the runtime immediately flags invalid pairings (such as assigning tools to a model with `tools: false` or assigning a `box` question to a decision model that only supports `noul`/`choice`/`score`).

## Node & agent spec (sketch)

```yaml
node: claims-triage
kind: decide
role: classify incoming insurance claims by risk tier
model:
  provider: openrouter_decisions
  name: typesafe/jev-1.14
  version: "1.14"
  fallback:
    provider: ollama
    name: qwen3.6-27b
    version: "27b-q4_K_M"
questions:
  risk:
    type: choice
    instructions: "How risky is this claim based on policy exclusions?"
    criteria:
      low: "Routine claim within standard coverage"
      medium: "Ambiguous coverage requiring adjuster review"
      high: "High fraud indicators or excluded peril"
      other: "Insufficient information to classify"
inputs:
  schema: ClaimText
outputs:
  schema: RiskDecision
```

## Graph spec

Nodes, typed edges, and conditions only—no custom workflow DSL. Conditional edges can branch directly on `decide` node outputs and `confidence` thresholds (for example, routing `confidence >= 0.85` directly to automated processing while escalating `confidence < 0.85` to a frontier `agent` node or a human approval gate).

## Output modes for LLM nodes

**Typed output (`output_schema`) is enforced by default.** Every node declares a Pydantic output schema, and the model layer uses its structured-output ladder (native JSON schema → tool-call-as-schema → JSON mode → bounded prompt-and-parse retry) to guarantee conformance. When free-form prose is genuinely desired, the node explicitly specifies the built-in `TextPayload(text: str)` schema.

## Audit and tracing model

Detailed in [09 Audit and tracing](09-audit-and-tracing.md):

- **Append-only PostgreSQL table (`audit_events`)** protected by `UPDATE`/`DELETE` prevention triggers.
- **RFC 8785 (JSON Canonicalization Scheme)** serialization + **SHA-256 hash chaining** + **Ed25519 signatures** (`key_id` rotation-aware).
- **Dual monotonic counters (`run_seq` + `ledger_seq`)** recording both permitted (`ALLOW`) and blocked (`DENY`) steps so deleted records leave a provable gap.
- **Content-addressed payload blobs (`payload_blobs`)** storing redacted inputs, outputs, and raw provider envelopes by `sha256` digest so privacy erasure never invalidates historical hash chains.
- **OpenTelemetry GenAI (`v1.42.0`) exporter** isolated in `otel_mapping.py`, emitting `invoke_workflow`, `invoke_agent`, `chat`/`decide`/`embeddings`, and `execute_tool` spans plus loop-detection metrics (`gen_ai.invoke_agent.inference_calls` vs. `tool_calls`).
- **Pluggable external chain-head anchoring** (local signed checkpoints in dev; S3 Object Lock WORM or Sigstore Rekor in production).

## Confirmed stack

- **Backend:** Python 3.12+ (`FastAPI`, `Pydantic v2`, `asyncio`, `httpx`, `cryptography`)
- **Model access:** Thin, auditable in-house adapters (`openai_compat.py`, `anthropic.py`, `decisions.py`, `fake.py`); zero LiteLLM dependency
- **Supply chain:** `uv` lockfile with `--require-hashes`, pinned versions, minimal dependency surface
- **Storage:** PostgreSQL 17 (`specs`, `runs`, `audit_events`, `payload_blobs`)
- **Durable execution (post-MVP):** DBOS (`dbos-transact-py`) checkpointing directly to PostgreSQL
- **Frontend (Phase 4):** React + TypeScript + React Flow
- **Executor:** Thin custom graph executor with full control over step boundaries, retries, budgets, and cryptographic receipts

## Security architecture

- **Sandboxed tools & MCP servers:** Execute shell, file, and external MCP tools inside rootless containers or OS sandboxes (Bubblewrap) with explicit per-node allowlists.
- **Encrypted secrets:** Specs store `secret://...` references only; keys never enter prompts, audit blobs, or OTel spans.
- **Two-layer gating on destructive actions:** Decision models (`typesafe/jev-1.14`) can pre-screen tool calls for reversibility, but any destructive action or low-confidence check requires a deterministic policy check or human approval gate.
