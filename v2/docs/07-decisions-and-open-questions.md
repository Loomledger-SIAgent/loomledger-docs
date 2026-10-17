# 07 Decisions and open questions

_Updated 17 Oct 2026._

## Decisions

| ID | Decision | Status |
|---|---|---|
| D1 | Project name: **loomledger** (loom for weaving agents into graphs, ledger for the audit trail) | Decided |
| D2 | Agents and nodes are **model-agnostic**; LLMs and non-LLMs (System One / Decisions API models, embeddings, tools) are both first-class; local vs API-hosted is not a constraint | Decided |
| D3 | The **spec is the source of truth**; the canvas is a view of it | Decided |
| D4 | **Typed ports** on every node; canvas validates connections; agent-to-agent messages are typed payloads | Decided |
| D5 | **Audit and tracing core before the UI** (Phase 2 before Phase 4) | Decided |
| D6 | **Own thin executor**, not built on LangGraph or Pydantic Graph; frameworks are for learning and comparison | Decided |
| D7 | The meta-agent orchestrator produces a **draft for user review**, never an autonomous orchestrator | Decided |
| D8 | Backend in **Python (FastAPI, Pydantic v2, asyncio, httpx)**, frontend in **React + TypeScript + React Flow**, storage in **PostgreSQL** (Java alternative dropped) | Decided |
| D9 | **Own thin model adapters** (`httpx` + Pydantic); no LiteLLM dependency after the March 2026 PyPI compromise; `uv` hash-pinned lockfile | Decided |
| D10 | **Audit log is separate from telemetry**: own append-only, RFC 8785 hash-chained and Ed25519-signed store is the source of truth; OpenTelemetry (`semantic-conventions-genai`) is exported alongside | Decided |
| D11 | **Durable execution deferred** past the MVP; design for idempotent steps and serializable state; DBOS (`dbos-transact-py`) on Postgres is the post-MVP choice | Decided |
| D12 | **PostgreSQL over MongoDB** for specs, runs and audit events (JSONB for flexible metadata, relational tables for runs/costs, content-addressed blob store for large payloads) | Decided |

## Proposed resolutions for open questions

| # | Question | Recommended resolution |
|---|---|---|
| 1 | **Stack** | **Resolved (D8):** Python (`FastAPI`, `Pydantic v2`, `asyncio`, `httpx`, `uv`) + PostgreSQL + React/React Flow. Java option removed. |
| 2 | **Schema strictness** | **Typed output (`output_schema`) is the enforced default** for all nodes so graph edges and the meta-agent stay type-safe. Free-text is supported by explicitly declaring the built-in `TextPayload(text: str)` schema. |
| 3 | **Audit privacy & redaction** | **Content-addressed payload separation + pre-hash redaction:** Apply configurable regex/NER redaction middleware before persistence, store full inputs/outputs in `payload_blobs` keyed by `sha256`, and place only `input_hash` and `output_hash` inside the signed `audit_events` row so purging sensitive blobs later never breaks the hash chain. |
| 4 | **Audit event schema** | **Specified in [09 Audit and tracing](09-audit-and-tracing.md):** RFC 8785 (JCS) canonical JSON, `run_seq` + `ledger_seq` counters (including `DENY` events), SHA-256 hash chaining, Ed25519 signatures with `key_id`, and isolated OTel GenAI `v1.42.0` span/metric mapping. |
| 5 | **Terminology** | Everything in a graph is a **`Node`** (`kind: agent | llm | decide | embed | tool`). Reserve **`Agent`** strictly for an LLM node configured with a tool loop (`max_steps > 1`). Single-shot LLM calls are `llm` nodes. |
| 6 | **Deployment** | **Local-first single-user via `docker compose`** (Postgres + API + local UI/CLI) for Phases 0–4; multi-tenant hosted mode deferred past MVP. |
| 7 | **License** | **Apache-2.0** (recommended for open-source portfolio and enterprise adoption with patent grant) or **MIT**. |
| 8 | **External anchoring** | Pluggable `ChainAnchor` interface: default to local signed checkpoint files in dev, with **S3-compatible Object Lock (WORM)** and **Sigstore Rekor** adapters for production compliance. |
| 9 | **MVP model providers** | Four adapters: `openai_compat` (covers OpenAI, Ollama, LM Studio, OpenRouter chat), `anthropic` (native messages API), `decisions` (TypeSafe System One API + OpenRouter `/api/alpha/decisions` supporting `typesafe/jev-1.14` and 21 other decision models), and `fake` (scripted deterministic testing). |
