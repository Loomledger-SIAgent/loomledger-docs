# 04 Implementation plan

Estimates assume about 10 hours per week and some learning time. Treat them as rough and optimistic.

## Phases

### Phase 0 (3-4 days): foundations
- Repo setup: `uv`, `ruff`, `pytest`, docker-compose with Postgres
- Short decision log (see [07](07-decisions-and-open-questions.md))
- Draft spec schemas for node, graph and envelope

### Phase 1 (weeks 1-2): single-node runtime
- Envelope and `ModelNode` contract with capability checks
- LLM adapter (gateway + Ollama) and a Jev adapter
- YAML spec to validated Pydantic objects
- A bare agent tool loop with permissioned tools
- Tests using a fake model so they run without API calls

### Phase 2 (weeks 2-3): audit and tracing core
Build this early because it is the differentiator.
- Append-only event table with hash chaining
- Every invocation logged: inputs, outputs, model/version, tokens, cost, latency, confidence
- Run replay from recorded responses
- CLI trace viewer

Milestone: a traced, auditable agent run. This is already a strong demo on its own.

### Phase 3 (week 4): multi-node execution
- Graph of typed nodes and edges with conditional routing (including decision nodes such as Jev)
- Typed message passing, retries, fallbacks, timeouts
- Per-run budgets (max tokens or cost)
- Human approval gate for risky actions

### Phase 4 (weeks 5-6): API and canvas
- FastAPI endpoints
- React Flow canvas that reads and writes the same graph spec
- Type-validated connections
- Run viewer: trace timeline and cost per node

### Phase 5 (weeks 7-8): meta-agent orchestrator
- Takes agent specs and drafts a graph as schema-validated output
- The same type checker validates the draft; the user reviews and approves it
- Evaluate against hand-built graphs on a small test set

### Later
Auth and multi-tenancy, durable execution (for example Temporal), RBAC, audit export, tool sandboxing improvements, more providers.

## Principles

1. **The spec is the source of truth.** The UI is just a view.
2. **Keep the graph model simple:** nodes, typed edges, conditions. No new workflow language.
3. **Audit versus privacy:** decide early on redaction and retention; hash large payloads and store them separately with access control.
4. **API keys are secrets:** encrypted storage, never in specs or logs.
5. **Tools are the main security risk:** run shell and file tools in a container or sandbox from the start.
6. **Build the spec and logs before the UI.**

## Risks

- **Scope creep** is the biggest one. The canvas and the meta-agent are tempting, but Phases 1-3 deliver the real value.
- **Timeline:** estimates slip if time is constrained. Phase 2 is the milestone worth shipping on its own.
- **Meta-agent quality:** keep it as "draft for review", and measure it against hand-built graphs before trusting it.
- **Provider churn:** model and framework versions move fast; pin versions and read current docs.
