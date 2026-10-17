# 03 Architecture

_Draft. The stack choice is proposed and not yet confirmed (see [07 Decisions](07-decisions-and-open-questions.md))._

## Overview

```
 Canvas (React Flow) ──reads/writes──► Specs (YAML/JSON, versioned)
                                           │
                                           ▼
 API (FastAPI) ──► Executor ──► Node adapters ──► LLMs / classifiers / embeddings / tools
                      │
                      ▼
          Audit log (append-only, hash-chained) + OpenTelemetry traces
```

**The spec is the source of truth. The UI is a view of it.** If the canvas and the spec ever disagree, the spec wins.

## Node kinds and typed ports

Every node declares typed inputs and outputs.

| Node kind | Example | Typical output |
|---|---|---|
| LLM agent | Claude, GPT, local Qwen | text, tool calls, or schema-validated JSON |
| Decision model | Jev | label + probabilities |
| Embedding / vision / speech | any | vector, text, structured result |
| Code / tool node | plain function | whatever it returns |

A non-LLM such as Jev is not an agent in the strict sense (no loop, cannot write). In the canvas everything is a **node**; "agent" is reserved for LLM nodes that run a tool loop.

### Why typed ports

1. **The canvas validates connections.** If A outputs `ClaimSummary` and B expects `RiskQuestion`, the link is refused or an adapter is suggested.
2. **The meta-agent orchestrator becomes more reliable.** It matches declared schemas rather than guessing from prose role descriptions.
3. **Agent-to-agent messages are neutral typed payloads.** Raw chat histories never cross node boundaries.
4. **Audit logging falls out of the envelope.** One event shape covers LLMs and non-LLMs.

## Model contract (sketch)

See [08 Model layer](08-model-layer.md) for the refined design (request kinds, capabilities, middleware, Jev adapter).

```python
class ModelNode(Protocol):
    kind: str                     # "llm" | "classifier" | "embedding" | ...
    capabilities: Capabilities    # tools, json_mode, vision, max_context, cost
    input_schema: type[BaseModel]
    output_schema: type[BaseModel]

    async def invoke(self, request: Envelope) -> Envelope: ...
    # Envelope: payload, trace_id, parent_span, usage, model_id, confidence?
```

Providers (an OpenAI-compatible gateway, Ollama, Jev, ...) are adapters implementing this contract. Capabilities are checked before a run, so the runtime can say "this agent needs tool calling and the chosen model does not support it".

## Agent spec (sketch)

```yaml
agent: claims-triage
role: classify incoming claims
context: ...
task: ...
model:
  type: classifier          # llm | classifier | embedding | ...
  provider: jev
  version: pinned-version
  fallback: { type: llm, provider: ollama, name: qwen-local }
limits: { max_tokens: 2000, allowed_tools: [lookup_policy] }
inputs:  { schema: ClaimText }
outputs: { schema: RiskLabel }
```

## Graph spec

Nodes, typed edges and conditions only. No custom workflow language. Conditional routing can use a decision node (for example Jev) with a confidence threshold, escalating to a stronger LLM when confidence is low.

## Output modes for LLM nodes

Two modes, with **typed** as the default:

- **Typed:** output must match a schema; the runtime retries on validation failure.
- **Free-text:** output is wrapped as a simple `Text` type.

(Whether typed should be strictly enforced by default is an open question.)

## Audit model (baseline, to be designed in detail)

- **Append-only** event table; each event includes the hash of the previous one
- Every invocation records: inputs, outputs, provider, model id and pinned version, token usage, cost, latency, confidence (if the model provides one)
- Events link to OpenTelemetry spans so traces connect across nodes
- Large payloads can be hashed and stored separately with access control
- Run replay from recorded model responses

A full event schema is a pending design task.

## Hard parts of multi-model support

Keep these in mind for every design decision:

- **Provider differences:** tool calling and structured output vary. Do not design for the lowest common denominator; declare capabilities per model and check them.
- **Prompt portability:** a prompt tuned on one model often degrades on another. Treat each agent + model pair as something to evaluate.
- **Weak or local models in agent loops:** warn or block bad pairings (e.g. a small local model assigned complex multi-tool planning).
- **Neutral messages:** typed payloads between nodes; no raw histories.
- **Operational differences:** token counting, rate limits and errors vary; build retries, timeouts and fallback models.
- **Keep the provider layer thin and auditable.** Write small adapters (httpx + Pydantic) that implement the `ModelNode` contract. One OpenAI-compatible adapter covers OpenAI, Ollama, LM Studio and many others; add dedicated adapters for Anthropic, Gemini and non-LLMs such as Jev. Avoid LiteLLM as a dependency: PyPI releases 1.82.7 and 1.82.8 were compromised in March 2026 with credential-stealing malware, and a layer that sees every API key is a high-value target. If a gateway is wanted later, run it as a separate process, not as an in-process dependency.

## Proposed stack

- **Backend:** Python (FastAPI, Pydantic, asyncio)
- **Model access:** own thin adapters (see above): OpenAI-compatible adapter (covers Ollama and LM Studio), provider adapters as needed, custom adapter for Jev
- **Supply chain:** `uv` lockfile with hashes, pinned versions, few dependencies, dependency review before upgrades
- **Storage:** Postgres (specs, runs, audit events)
- **Audit vs telemetry:** the audit log is its own append-only, hash-chained store and is the source of truth. OpenTelemetry traces are exported alongside it for observability, but are not relied on for audit because telemetry can be sampled or dropped. The OpenTelemetry GenAI conventions are still in Development status, so map to them without depending on them.
- **Durable execution:** deferred past the MVP; design steps to be idempotent with serializable state. First candidate later: DBOS (runs on Postgres).
- **Frontend:** React + TypeScript + React Flow
- **Executor:** own thin executor rather than building on LangGraph, because orchestration and audit are the product and need full control of every step, retry and log

## Security notes

- Tools are the main risk: sandbox shell and file tools from the start.
- API keys are secrets: encrypted storage, never in specs or logs.
- Decision models can be steered by injected text in untrusted input; do not make one the only gate on destructive actions.
