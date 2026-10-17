# 01 Vision and positioning

_Updated 17 Oct 2026. Market and technical references are documented in [06 Research notes](06-research-notes.md)._

## The idea

A tool for creating custom agents, with these capabilities:

1. **Agent communication through an orchestrator.** Users can define a custom orchestrator. If they want default behaviour, a meta-agent that understands the custom agents (role, context, limitations, operations, task) drafts an orchestrator for them to review.
2. **Drag-and-drop wiring.** Users connect agents and nodes on a canvas first; the orchestrator-building step comes after.
3. **Logging and traceability for every agent.** Cryptographic audit logging is a core architectural pillar.
4. **Model-agnostic agents and nodes.** Nodes are not tied to one AI model. Each node can use a different model, and the models can be LLMs or non-LLMs (such as System One / Decisions API models like `typesafe/jev-1.14`, embeddings, or plain functions).

## Honest assessment

**As a commercial product, the generic version is a hard sell.** Drag-and-drop agent builders are a crowded space. Flowise, Langflow, n8n and Dify all offer visual builders, and the open-source ones can be self-hosted for free. Dify has a very large community, OpenAI ships a visual Agent Builder, and Google Vertex AI Agent Builder and Microsoft Copilot Studio cover the enterprise side. A solo developer will not beat these on feature breadth. Model-agnosticism alone is not a differentiator either: Dify supports 100+ LLM providers, and frameworks like Pydantic AI are model-agnostic.

**As a learning and portfolio project—and as a focused governance tool—it is excellent.** It touches the whole agent stack: runtime, tool calling, orchestration, state, a graph UI, cryptographic audit trails, OpenTelemetry tracing, evals and security.

### Strengths

- **Audit and traceability as a core feature** (not an add-on) is a real angle for regulated settings such as finance, insurance and healthcare (see [09 Audit and tracing](09-audit-and-tracing.md)).
- **Per-node model choice** enables cost control: cheap decision models or local models for routing and routine steps, frontier models for hard reasoning.
- **First-class decision nodes** (`typesafe/jev-1.14` and the 22-model OpenRouter Decisions API) alongside LLMs make cost, latency and reliability trade-offs explicit.
- **Typed ports** make the canvas meaningful and the auto-orchestrator reliable (see [03 Architecture](03-architecture.md)).

### Weaknesses and risks

- **The auto-built orchestrator is the riskiest feature.** Feasible version: a meta-agent drafts a typed graph that the user reviews and edits. Unrealistic version: a fully autonomous orchestrator with no review. Generated orchestration is often subtly wrong.
- **Multi-agent is not automatically better.** More agents mean more tokens and compounding errors; many production systems prefer one agent with good tools. The platform should make single-agent workflows easy and show the cost of each topology.
- **Drag-and-drop is the cheapest part.** The hard parts are the runtime, state, failure handling and cryptographic logs behind the canvas.
- **Scope creep.** The canvas and meta-agent are tempting, but the core value lives in the runtime and audit ledger.

## Positioning

> A governed agent builder: model-agnostic nodes, per-node cost attribution, and cryptographic audit trails you can verify and replay.

Differentiating capabilities:

- Node and agent definitions as **versioned, typed specs** (role, context, limits, allowed tools, pinned model)
- **Enforced** limits and permissions per agent, not just words in a prompt
- **Append-only, RFC 8785 hash-chained and Ed25519-signed** audit log tied to OpenTelemetry GenAI traces
- **Two-layer safety gates** (decision-model reversibility checks + human approval gates) for risky actions
- **Deterministic replay** of any run from content-addressed recorded responses
- **Cost attribution** ("this workflow cost $X; agent B on model Y caused 70% of it")

## Success criteria for the MVP

- A single traced, cryptographically verifiable agent run is demonstrable end to end via CLI.
- A 2–3 node workflow mixing at least one LLM node and one decision-model node (`typesafe/jev-1.14`) runs with a full cross-node trace and verifiable receipt chain.
- Any run can be replayed deterministically from recorded responses without live API calls.
- The canvas, when it arrives in Phase 4, reads and writes the exact same spec files the CLI uses.
