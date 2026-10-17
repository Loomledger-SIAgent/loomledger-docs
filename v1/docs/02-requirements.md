# 02 Requirements

Items marked **(proposed)** came out of the analysis rather than the original idea and should be confirmed.

## Functional requirements

| ID | Requirement |
|---|---|
| FR1 | **Agent definition.** A user can create a custom agent with a role, context, limitations, operations (tools) and a task. Definitions are versioned specs. |
| FR2 | **Model-agnostic nodes.** Each node chooses its own model. Models can be LLMs or non-LLMs (classifiers, embeddings, vision, speech, etc.). Local and API-hosted models are both supported. |
| FR3 | **Orchestration by user.** A user can define a custom orchestrator that governs how agents communicate. |
| FR4 | **Orchestration by default.** When the user wants default behaviour, a meta-agent reads the agent specs (role, context, limits, operations, task) and drafts an orchestrator. |
| FR5 | **Drag-and-drop wiring.** Users connect agents on a canvas first; the orchestrator-building journey follows. Connections are type-validated. |
| FR6 | **Audit logging and traceability.** Every node invocation and agent action is logged with enough detail to explain what happened and why. |
| FR7 | **Cost attribution (proposed).** Tokens and cost are attributed per node and per run. |
| FR8 | **Run controls (proposed).** Per-run budgets, timeouts, retries, model fallbacks, and human approval gates for risky actions. |
| FR9 | **Replay (proposed).** Any run can be replayed from recorded responses for debugging and compliance. |

## Non-functional requirements

- **Tamper-evident audit trail.** Append-only events, hash-chained so edits are detectable.
- **Privacy-aware logging.** Policy for redaction and retention of sensitive prompts and outputs (see open questions).
- **Secrets handling.** API keys stored encrypted; never in specs, logs or traces.
- **Sandboxed tools.** Shell and file tools run in a container or sandbox.
- **Testability.** The whole runtime can be tested with fake models, with no API calls.
- **Pinned model versions.** Specs pin the exact model version so logs can state which model ran.

## Constraints from multi-model design

These cautions apply to any design decision (details in [03 Architecture](03-architecture.md)):

1. Tool calling and structured output differ across providers.
2. Prompts are not portable between models.
3. Weaker or local models struggle in agent loops.
4. Agent-to-agent messages use neutral typed payloads, not raw chat history.
5. Token counting, rate limits and error behaviour vary by provider.

## Non-goals for the MVP

- Fully autonomous orchestrator generation without user review
- Multi-tenancy, RBAC and SSO
- Supporting every model provider
- Training or fine-tuning models
- Competing on canvas features with existing builders
