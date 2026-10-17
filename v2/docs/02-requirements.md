# 02 Requirements

_Updated 17 Oct 2026. Previously proposed requirements (FR7–FR9) are now confirmed as core scope (see [07 Decisions](07-decisions-and-open-questions.md))._

## Functional requirements

| ID | Requirement |
|---|---|
| FR1 | **Node & agent definition.** A user can create a custom agent or node with a role, context, limitations, operations (tools) and a task. Definitions are versioned YAML/JSON specs with typed input/output schemas. |
| FR2 | **Model-agnostic nodes.** Each node chooses its own model. Models can be LLMs or non-LLMs (System One / Decisions API models such as `typesafe/jev-1.14`, embeddings, vision, speech, or plain functions). Local and API-hosted models are both supported. |
| FR3 | **Orchestration by user.** A user can define a custom orchestrator graph that governs how nodes and agents communicate. |
| FR4 | **Orchestration by default.** When the user wants default behaviour, a meta-agent reads the node specs (role, context, limits, operations, task, input/output schemas) and drafts a type-validated orchestrator graph for user review. |
| FR5 | **Drag-and-drop wiring.** Users connect nodes on a canvas first; the orchestrator-building journey follows. Connections are type-validated against declared schemas. |
| FR6 | **Audit logging and traceability.** Every node invocation, policy gate, and agent action (both `ALLOW` and `DENY`) is logged in an append-only, cryptographically verifiable trail with enough detail to explain what happened and why. |
| FR7 | **Cost attribution.** Input, output, cached, and reasoning tokens plus USD cost (from a versioned price catalog) are attributed per node and per run. |
| FR8 | **Run controls.** Per-run budgets, timeouts, bounded retries, capability-checked model fallbacks, decision-model guardrails, and human approval gates for risky actions. |
| FR9 | **Deterministic replay.** Any run can be replayed offline from content-addressed recorded responses for debugging and compliance verification. |

## Non-functional requirements

- **Tamper-evident audit trail.** Append-only events serialized via RFC 8785 (JSON Canonicalization Scheme), SHA-256 hash-chained with monotonic `run_seq` counters, Ed25519-signed with `key_id`, and externally anchorable to WORM storage (see [09 Audit and tracing](09-audit-and-tracing.md)).
- **Privacy-aware logging.** Content-addressed payload separation (`payload_blobs` keyed by `sha256`) and pre-hash redaction middleware so sensitive prompts/outputs can be redacted or purged under retention policies without breaking the cryptographic hash chain.
- **Secrets handling.** API keys referenced as `secret://...` URIs and stored encrypted; never in specs, logs, or telemetry spans.
- **Sandboxed tools.** Shell, file, and external MCP server tools run inside rootless containers or OS sandboxes (e.g., Bubblewrap) with per-node capability allowlists.
- **Testability.** The entire runtime and graph executor can be tested deterministically with `FakeModel` and recorded HTTP fixtures, with zero live API calls.
- **Pinned model and catalog versions.** Saved specs pin exact model versions (rejecting moving aliases like `jev-latest`) and record the `catalog_version` used for capability and cost calculation.

## Constraints from multi-model design

These cautions apply to every design decision (details in [03 Architecture](03-architecture.md) and [08 Model layer](08-model-layer.md)):

1. Tool calling and structured output differ across providers.
2. Prompts and decision-model probability thresholds are not portable between models.
3. Weaker or local models struggle in complex multi-tool agent loops.
4. Node-to-node messages use neutral typed payloads, never raw chat history.
5. Token counting, rate limits, context modes (e.g., Jev 1.14's 32KB fast mode vs. 262K long-context mode), and error behaviour vary by provider.

## Non-goals for the MVP

- Fully autonomous orchestrator generation without user review
- Multi-tenancy, RBAC and SSO
- Supporting every model provider natively in Phase 1
- Training or fine-tuning models
- Competing on generic canvas widgets with existing builders
