# 05 Learning roadmap

For a backend developer who knows Python and wants to build agentic AI systems. The field is usually called **agentic AI engineering** (a subset of AI engineering). Related terms: harness engineering (the system around the model), context engineering (what goes into the model's context at each step) and LLMOps/evals.

## Steps

1. **Python refresh (about 1 week).** Type hints, `pydantic` v2 (validation and schema generation), `asyncio`, `httpx`, `pytest`, `uv`. Skip frameworks for now.
2. **LLM API fundamentals (about 1 week).** Chat message format, tool/function calling, structured output, streaming, context windows, prompt caching. Practise against a local Ollama endpoint and a hosted API.
3. **Build a bare agent loop yourself (about 2 weeks).** The most important step. Loop: model, tool call, execute, feed result back, repeat. Start with `read_file`, `edit_file`, `grep`, `run_shell`, plus a permission layer (allow / ask / deny). Skip LangChain-style frameworks at first so the mechanics are visible.
4. **Context management (1-2 weeks).** Repo maps, retrieving only relevant files, truncating tool output, history compaction, prompt caching. This is where most token savings come from, more than model choice.
5. **Decision-model routing (1-2 weeks).** Define decision points as `noul` / `choice` / `score` / `box` questions, set confidence thresholds, escalate when unsure, and log every decision with its probability and outcome.
6. **Evals and observability (ongoing).** Build a small task set from real repos; measure success rate, tokens and cost per task.
7. **Safety.** Prompt injection, sandboxed execution (containers or Bubblewrap), command allowlists, and two-layer gating for destructive tools.
8. **MCP (Model Context Protocol).** Once the loop works, so the agent can use external tools inside a sandboxed boundary.

## Frameworks to study and compare

- **Pydantic AI (first).** Type-safe, model-agnostic, and as of late 2026 includes `pydantic_graph` (steps, joins, reducers, decisions), `pydantic_evals`, native `system_one` / `typesafe` (Jev) and `openai_decisions` model adapters, and first-class durable execution (`RuntimeCapability`) with DBOS, Temporal and Restate. Best reference architecture to benchmark against.
- **LangGraph (second).** Stateful graphs with checkpointing; a natural fit for classify, route, execute, test, retry or escalate. Steeper learning curve.
- **Claude Agent SDK (as a reference).** Built for coding agents with OS access and deep MCP support. Tied to Claude models, but good study material for tools, subagents and permissions.
- **OpenAI Agents SDK (optional).** Simple to learn; avoid using it with in-process LiteLLM after the March 2026 supply-chain compromise.
- **Skip CrewAI for now.** Role-based multi-agent setups add token cost.

## Suggested order

1. Python refresher, then the bare agent loop (about 3 weeks)
2. Rebuild the same agent in Pydantic AI pointed at Ollama; compare code size and complexity
3. Add a decision-model router (using Jev 1.14 or the OpenRouter Decisions API) as plain code with thresholds
4. Compare `pydantic_graph` and LangGraph once retries, escalation and resumable state are needed
5. Study Claude Agent SDK and Pydantic AI harness examples to refine tool and permission design

Note: loomledger itself uses its **own thin executor** (see [07](07-decisions-and-open-questions.md)). Frameworks are for learning and comparison, not the core of the product.

## Cautions

- Framework versions change quickly; pin versions and read current docs instead of older tutorials.
- Keep your eval set and token/cost logging framework-independent so frameworks can be compared fairly.
