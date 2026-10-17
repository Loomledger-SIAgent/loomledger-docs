# loomledger

> Build model-agnostic multi-agent workflows with audit-grade traces.

**Status:** planning (Phase 0). Nothing is implemented yet; this repo currently holds the analysis and design docs.

## The idea

loomledger is a platform for creating custom agents and wiring them into workflows:

- **Custom agents:** each agent has a role, context, limitations, operations (tools) and a task.
- **Model-agnostic:** every node picks its own model, and a model can be an LLM *or* a non-LLM (for example a decision classifier such as Jev).
- **Orchestration, two ways:** define your own orchestrator, or let a meta-agent read the agent definitions and draft one for you to review.
- **Drag-and-drop canvas:** connect agents visually; connections are type-checked.
- **Audit and traceability first:** every model call and agent action is logged in a tamper-evident trail, with cost attributed per node.

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
| [06 Research notes](docs/06-research-notes.md) | Jev, local models, agent frameworks, competitor landscape, sources |
| [07 Decisions and open questions](docs/07-decisions-and-open-questions.md) | Decision log and what is still undecided |
| [08 Model layer](docs/08-model-layer.md) | Model contract, adapters, middleware, Jev adapter, testing |

## Roadmap at a glance

1. **Phase 0:** repo setup, spec schemas
2. **Phase 1:** single-node runtime (envelope, model adapters, tool loop)
3. **Phase 2:** audit and tracing core (the differentiator)
4. **Phase 3:** multi-node execution (typed graph, routing, budgets, approval gates)
5. **Phase 4:** API and drag-and-drop canvas
6. **Phase 5:** meta-agent that drafts orchestrators

Details in [docs/04-implementation-plan.md](docs/04-implementation-plan.md).

## License

TBD.
