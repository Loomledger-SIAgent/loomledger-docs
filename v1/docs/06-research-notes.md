# 06 Research notes

_Collected 9 Oct 2026 from web searches. Most figures are vendor- or blog-reported and this space moves fast. Verify before relying on them._

## Jev (decision model)

- A decision model from the startup TypeSafe AI. It reads language like an LLM but cannot write replies, code or explanations; it picks from options you supply and returns probabilities.
- Reported pricing: $0.042 per million input tokens, with free output.
- Reported release date: 15 September 2026. Offered as a hosted API; no downloadable weights were found.
- Useful pattern: LLM writes, Jev decides (typed Choice/Score with probabilities), plain code acts.
- Good fits: model routing, guard/permission checks, intent routing, confidence-gated escalation.
- A paper on using JEV as a judge reports that accepting confident verdicts and escalating uncertain ones retained about 99% of a frontier LLM judge's accuracy at a tiny fraction of the cost.

**Caveats**

- Savings apply to decision calls, not to code generation itself.
- Weaker on reasoning-heavy judgements: reported 14.6 points behind on JudgeBench; one comparison showed 62.6% against 81.3% for a small Claude model on a broad phishing question.
- New and mostly vendor-reported: test on your own data.
- Documented limitation: injected instructions, or text arguing for its own classification, can move the answer. Do not make it the only gate for destructive actions.
- It is not a drop-in model swap; it lives inside your harness.

## Local models (Apple Silicon)

RAM (unified memory) is what limits local models, not SSD size. M4 MacBooks go up to 32GB (M4), 48GB (M4 Pro) and 128GB (M4 Max).

| RAM | Reported pick |
|---|---|
| 16GB | Qwen 3.5-9B (Q4) |
| 24GB | Qwen 3.6-27B |
| 32GB+ | Qwen 3.6-35B-A3B or Qwen3-Coder-30B-A3B (about 19 GB) |
| 64GB+ | Qwen3-Coder-Next (Q4) |

- Runtimes: Ollama (MLX preview backend in recent versions), LM Studio or `mlx_lm`; all expose OpenAI-compatible endpoints.
- Tool-calling reliability varies by runtime; test it yourself.
- Local 27-35B models handle routine edits well but trail frontier models on hard multi-file agentic work, which is what a router is for.

## Agent frameworks (summary)

| Framework | Notes |
|---|---|
| Pydantic AI | Type-safe, model-agnostic, durable execution; younger multi-agent story |
| LangGraph | Stateful graphs, checkpointing; steeper curve |
| Claude Agent SDK | Coding-agent primitives, strong MCP support; Claude-only |
| OpenAI Agents SDK | Simple; 100+ models via LiteLLM |
| CrewAI | Role-based multi-agent; adds token cost |

## Visual builder landscape

- Flowise, Langflow, n8n and Dify offer drag-and-drop agent builders; all four are open-source with free self-hosting.
- Dify reports 130K+ GitHub stars and 100+ LLM providers.
- OpenAI's AgentKit includes a visual Agent Builder.
- Google Vertex AI Agent Builder and Microsoft Copilot Studio cover the enterprise side.

## Stack research (9 Oct 2026)

**LiteLLM supply-chain incident.** In March 2026, PyPI versions 1.82.7 and 1.82.8 of LiteLLM were compromised with credential-stealing malware. Anyone who installed those versions should rotate provider keys. This is why loomledger uses its own thin adapters instead of LiteLLM. Alternatives people moved to include compiled gateways run as separate processes (Go-based Bifrost and GoModel, Rust-based TensorZero), managed gateways (OpenRouter, Cloudflare AI Gateway, Portkey, Kong AI Gateway) and small auditable libraries.

**OpenTelemetry GenAI semantic conventions.** In June 2026 (v1.42.0, 12 June) they moved into a dedicated repository, `semantic-conventions-genai`. As of mid-2026 every GenAI span, metric, event and attribute is still in Development status, and none is Stable. Agent-related operations include `create_agent`, `invoke_agent`, `invoke_workflow`, `plan` and `execute_tool`, with attributes such as `gen_ai.agent.name` and `gen_ai.agent.id`; MCP conventions also exist. Use the names, but expect changes.

**Durable execution.** Options: Temporal (the market leader, multi-language, self-hosted stateful clusters), DBOS (checkpoints steps to Postgres with no separate server, MIT-licensed; released DBOSify for Temporal Python on 20 July 2026 and reports a production-ready Java v1.0), Restate and Hatchet (lower-latency or Postgres-based alternatives), Inngest (event-driven). DBOS fits loomledger best because Postgres is already in the stack.

**Jev API details.** Single endpoint `POST https://api.typesafe.ai/v1/systemone` with `{state, model, questions}`; question types `noul`, `choice` (2-255 options) and `score` (2-10 levels); answers carry probabilities and confidence; no rationale is returned; choice probabilities are relative to the supplied labels; a pinned version (for example `jev-1.13.0`) and a moving `jev-latest` alias both exist. Sources: https://docs.typesafe.ai/api, https://www.promptfoo.dev/docs/providers/typesafe/, https://opentweet.io/jev/choice-score-noul

**PostgreSQL vs MongoDB.** Postgres gained documents via JSONB and Mongo gained multi-document transactions; what remains distinct is Mongo's native sharding and change streams versus Postgres's relational depth, constraints and SQL reporting. Postgres has no native change streams (LISTEN/NOTIFY or app-level events cover live views). For audit-style workloads, constraint and trigger support favours Postgres. Neither database alone makes an audit log tamper-evident. Sources: https://swyftstack.com/blog/mongodb-vs-postgresql, https://www.kunalganglani.com/blog/mongodb-vs-postgresql-2026, https://docs.nvidia.com/nvsentinel/components/postgre-sql-provider/

Sources:
- https://dev.to/kuldeep_paul/best-litellm-alternatives-for-production-ai-in-2026-f0a
- https://getmaxim.ai/articles/top-litellm-alternatives-in-2026/
- https://dev.to/s-bandy/litellm-alternative-the-best-options-for-2026-36f4
- https://www.dash0.com/knowledge/opentelemetry-genai-semantic-conventions-explained
- https://klu.ai/glossary/llm-session-tracing-opentelemetry
- https://dev.to/mr_manushukla/durable-ai-agents-without-temporal-exactly-once-workflows-on-postgres-with-dbos-2026-5a6n
- https://dev.to/yigit-konur/serverless-workflow-engines-40-tools-ranked-by-latency-cost-and-developer-experience-19h2

## Name check

"patchbay" was considered and rejected: several existing GitHub projects already use it, including AI-related ones. Always check GitHub, PyPI, npm and domains before settling on a name.

## Sources

- Jev: https://datanorth.ai/blog/jev-what-it-is-and-why-this-new-model-matters
- Jev: https://www.respan.ai/articles/what-is-the-jev-ai-model
- Jev: https://pooyagolchian.com/blog/what-is-jev-ai-decision-model-2026/
- Jev: https://www.aibuilderclub.com/blog/jev-engineering-guide
- Jev as judge: https://aiweekly.co/alerts/jev-judge-model-matches-llm-accuracy-at-036-of-the-cost
- Local models: https://stridenote.net/best-local-llms-coding-2026/
- Local models: https://ollamaherd.com/guides/apple-silicon-models
- Frameworks: https://www.morphllm.com/ai-agent-framework
- Frameworks: https://devtoollab.com/blog/best-ai-agent-frameworks
- Builders: https://fast.io/resources/best-no-code-ai-agent-builders/
- Builders: https://rasa.com/blog/best-low-code-ai-agents-platforms-for-2026
- Builders: https://www.morphllm.com/no-code-ai-agent
- Builders: https://inkeep.com/blog/agent-frameworks-platforms-overview
