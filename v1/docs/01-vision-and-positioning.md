# 01 Vision and positioning

_Written 9 Oct 2026. Market claims come from third-party articles (see [06 Research notes](06-research-notes.md)); re-check before relying on them._

## The idea

A tool for creating custom agents, with these capabilities:

1. **Agent communication through an orchestrator.** Users can define a custom orchestrator. If they want default behaviour, an agent that understands the custom agents (role, context, limitations, operations, task) builds the orchestrator for them.
2. **Drag-and-drop wiring.** Users connect agents on a canvas first; the orchestrator-building step comes after.
3. **Logging and traceability for every agent.** Audit logging is a very important factor.
4. **Model-agnostic agents.** Agents are not tied to one AI model. Each agent can use a different model, and the models can be LLMs or non-LLMs.

## Honest assessment

**As a commercial product, the generic version is a hard sell.** Drag-and-drop agent builders are a crowded space. Flowise, Langflow, n8n and Dify all offer visual builders, and the open-source ones can be self-hosted for free. Dify has a very large community, OpenAI ships a visual Agent Builder, and Google Vertex AI Agent Builder and Microsoft Copilot Studio cover the enterprise side. A solo developer will not beat these on feature breadth. Model-agnosticism alone is not a differentiator either: Dify supports 100+ LLM providers, and frameworks like Pydantic AI are model-agnostic.

**As a learning and portfolio project it is excellent.** It touches the whole agent stack: runtime, tool calling, orchestration, state, a graph UI, tracing, evals and security.

### Strengths

- **Audit and traceability as a core feature** (not an add-on) is a real angle for regulated settings such as finance, insurance and healthcare.
- **Per-agent model choice** enables cost control: cheap or local models for simple steps, frontier models for hard ones.
- **Non-LLM nodes** (classifiers, embeddings, vision, speech) used alongside LLMs make cost and reliability trade-offs explicit.
- **Typed ports** make the canvas meaningful and the auto-orchestrator more reliable (see [03 Architecture](03-architecture.md)).

### Weaknesses and risks

- **The auto-built orchestrator is the riskiest feature.** Feasible version: a meta-agent drafts a graph that the user reviews and edits. Unrealistic version: a fully autonomous orchestrator with no review. Generated orchestration is often subtly wrong.
- **Multi-agent is not automatically better.** More agents mean more tokens and compounding errors; many production systems prefer one agent with good tools. The platform should make single-agent workflows easy and show the cost of each topology.
- **Drag-and-drop is the cheapest part.** The hard parts are the runtime, state, failure handling and logs behind the canvas.
- **Scope creep.** The canvas and meta-agent are tempting but the value is in the runtime and audit core.

## Positioning

> A governed agent builder: model-agnostic nodes, per-node cost attribution, and audit trails you can replay.

Differentiating capabilities:

- Agent definitions as **versioned, typed specs** (role, context, limits, allowed tools, model)
- **Enforced** limits and permissions per agent, not just words in a prompt
- **Append-only, tamper-evident** audit log tied to OpenTelemetry-style traces
- **Human approval gates** for risky actions
- **Replay** of any run for debugging and compliance
- **Cost attribution** ("this workflow cost X; agent B on model Y caused 70% of it")

## Success criteria for the MVP

- A single traced, auditable agent run is demonstrable end to end (CLI is enough).
- A 2-3 node workflow mixing at least one LLM and one non-LLM node runs with a full cross-node trace.
- Any run can be replayed from recorded responses.
- The canvas, when it arrives, reads and writes the same spec files the CLI uses.
