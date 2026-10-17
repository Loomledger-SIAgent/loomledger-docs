# 07 Decisions and open questions

## Decisions

| ID | Decision | Status |
|---|---|---|
| D1 | Project name: **loomledger** (loom for weaving agents into graphs, ledger for the audit trail) | Decided |
| D2 | Agents and nodes are **model-agnostic**; LLMs and non-LLMs are both first-class; local vs API-hosted is not a constraint | Decided |
| D3 | The **spec is the source of truth**; the canvas is a view of it | Proposed |
| D4 | **Typed ports** on every node; canvas validates connections; agent-to-agent messages are typed payloads | Proposed |
| D5 | **Audit and tracing core before the UI** (Phase 2 before Phase 4) | Proposed |
| D6 | **Own thin executor**, not built on LangGraph; frameworks are for learning and comparison | Proposed |
| D7 | The meta-agent orchestrator produces a **draft for user review**, never an autonomous orchestrator | Proposed |
| D8 | Backend in **Python (FastAPI, Pydantic)**, frontend in **React + React Flow**, storage in **Postgres**. A Java/Quarkus core is the main alternative. | Proposed, awaiting confirmation |
| D9 | **Own thin model adapters** (httpx + Pydantic); no LiteLLM dependency after the March 2026 PyPI compromise; hash-pinned lockfile | Proposed |
| D10 | **Audit log is separate from telemetry**: own append-only hash-chained store is the source of truth; OpenTelemetry (GenAI conventions, still Development status) is exported alongside | Proposed |
| D11 | **Durable execution deferred** past the MVP; design for idempotent steps and serializable state; DBOS is the first candidate | Proposed |
| D12 | **PostgreSQL over MongoDB** for specs, runs and audit events (JSONB for flexible parts, relational tables for runs and costs, object storage for large payloads). Do not run both databases | Proposed |

## Open questions

1. **Stack:** confirm Python backend plus React/React Flow frontend, or keep more of the system in Java?
2. **Schema strictness:** should typed (schema-enforced) output be the default for LLM nodes, with free-text as an opt-out?
3. **Audit privacy:** what is the redaction and retention policy for sensitive prompts and outputs? Which fields are hashed and stored separately?
4. **Audit event schema:** exact fields, hash-chain format, and how events map to OpenTelemetry spans (next design task).
5. **Terminology:** call everything a "node" and reserve "agent" for LLM nodes with a tool loop?
6. **Deployment:** local-first single user, or hosted service later?
7. **License:** TBD.
8. **External anchoring for the audit chain:** how and where to publish periodic signed chain heads (for example object storage with retention locks), since no database alone makes logs tamper-evident.
9. **Which providers for the MVP?** Suggested: two or three LLM providers, one local model, and one non-LLM (Jev) to prove the abstraction.
