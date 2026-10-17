# loomledger

> Build model-agnostic multi-agent workflows with audit-grade traces.

**Status:** planning (Phase 0). Design docs updated 17 Oct 2026; ready to begin Phase 0 & Phase 1 implementation.

## The idea

loomledger is a platform for creating custom agents and wiring them into workflows:

- **Custom agents:** each agent has a role, context, limitations, operations (tools) and a task.
- **Model-agnostic:** every node picks its own model, and a model can be an LLM *or* a non-LLM (such as System One / Decisions API models like `typesafe/jev-1.14`, embeddings, or plain code).
- **Orchestration, two ways:** define your own orchestrator, or let a meta-agent read the agent definitions and draft one for you to review.
- **Drag-and-drop canvas:** connect agents visually; connections are type-checked.
- **Audit and traceability first:** every model call and agent action is logged in an RFC 8785 hash-chained, Ed25519-signed trail, with cost attributed per node.

## Why another agent builder?

Visual agent builders already exist (Dify, Langflow, Flowise, n8n, OpenAI's Agent Builder and others), so loomledger does not try to win on canvas features. Its angle is **governed, cost-aware, multi-model agents** where "which model made this decision, on what input, at what cost" is always answerable. See [docs/01-vision-and-positioning.md](docs/01-vision-and-positioning.md).

## Docs

| Doc | What it covers |
|---|---|
| [01 Vision and positioning](docs/01-vision-and-positioning.md) | The idea, honest assessment, competition, positioning |
| [02 Requirements](docs/02-requirements.md) | Functional and non-functional requirements, non-goals |
| [03 Architecture](docs/03-architecture.md) | Typed ports, node kinds, model contract, specs, audit model basics |
| [04 Implementation plan](docs/04-implementation-plan.md) | Phased plan, principles, risks |
| [05 Learning roadmap](docs/05-learning-roadmap.md) | What to learn, in what order, including frameworks |
| [06 Research notes](docs/06-research-notes.md) | Jev 1.14 & Decisions API, local models, frameworks, OTel GenAI, audit cryptography |
| [07 Decisions and open questions](docs/07-decisions-and-open-questions.md) | Confirmed stack decisions and proposed resolutions for open questions |
| [08 Model layer](docs/08-model-layer.md) | Model contract, adapters, middleware, System One / Decisions adapter, testing |
| [09 Audit and tracing](docs/09-audit-and-tracing.md) | RFC 8785 hash chain, Ed25519 receipts, content-addressed privacy blobs, OTel GenAI mapping |

## Roadmap at a glance

1. **Phase 0:** repo setup (`uv`, `ruff`, `pytest`, Postgres), spec schemas, cryptographic receipt primitives
2. **Phase 1:** single-node runtime (`Envelope`, `ModelNode` adapters for OpenAI-compat / Anthropic / Decisions API / Fake, permissioned tool loop)
3. **Phase 2:** audit and tracing core (`audit_events` hash chain + Ed25519 signing, `payload_blobs`, OTel GenAI spans, replay, CLI viewer)
4. **Phase 3:** multi-node execution (typed graph, confidence-gated decision routing, budgets, human approval gates)
5. **Phase 4:** API and drag-and-drop canvas (FastAPI + React Flow)
6. **Phase 5:** meta-agent that drafts orchestrators for review

Details in [docs/04-implementation-plan.md](docs/04-implementation-plan.md).

## License

Apache-2.0 (proposed).
