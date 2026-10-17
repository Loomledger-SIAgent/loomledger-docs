# 06 Research notes

_Updated 17 Oct 2026 from primary documentation and web research. Most figures are vendor- or blog-reported and this space moves fast. Verify before relying on them._

## Jev and the System One / Decisions API ecosystem

- **Jev 1.14 release (14 Oct 2026):** TypeSafe AI released `typesafe/jev-1.14` ([OpenRouter Jev 1.14](https://openrouter.ai/typesafe/jev-1.14); [TypeSafe System One Docs](https://docs.typesafe.ai/concepts/system-one)), following the initial 15 September 2026 launch (`jev-1.13`).
- **Pricing and latency:** $0.042 per million input tokens, $0 per million output tokens; ~0.11s–0.16s P50 latency on OpenRouter.
- **Context window and multimodal input:** Context expanded to **262,144 tokens** and now accepts both **text and images** in `state`. Note: requests over ~32,000 bytes of text or containing images run in TypeSafe's long-context mode, which is slower and slightly less accurate than the default fast mode (<32KB text).
- **Four question primitives:**
  1. `noul`: boolean condition check returning the probability of `true` (0 to 1).
  2. `choice`: 2–255 labelled options returning the selected option, full probability distribution across options, and `confidence`.
  3. `score`: 2–10 ordered rubric levels returning a probability-weighted position, per-level probabilities, and `confidence`.
  4. `box` (**new in Jev 1.14**): locates a described target in an image and returns normalized fractional bounding-box coordinates (`top`, `bottom`, `left`, `right`) plus `confidence`. Because `box` always returns four edges even when the target is absent, pair it with a `noul` presence question in the same request ([OpenRouter Jev Guide](https://openrouter.ai/docs/guides/community/jev)).
- **Two API surfaces & 22 decision models across 13 makers:** As of 16 October 2026, OpenRouter exposes both the **System One API** (`POST https://openrouter.ai/api/v1/systemone`, compatible with `POST https://api.typesafe.ai/v1/systemone`) and a standardized **Decisions API** (`POST https://openrouter.ai/api/alpha/decisions`) supported by 22 decision models from 13 providers, including OpenAI (`openai/gpt-6-luna-decisions`), Microsoft (`microsoft/microsoft-decision-1`), Perplexity (`perplexity/pplx-decider-v1.2`), Cloudflare (`cloudflare/clef`, `clef-omni`), Inception (`inception/mercury-decide`), Liquid AI (`liquid/d1`), and Upstage (`upstage/solar-decide`) ([OpenRouter Jev Guide](https://openrouter.ai/docs/guides/community/jev)).
- **Good fits:** model routing, tool-call guard/reversibility checks, intent routing, and confidence-gated cascades (draft with a cheap model, verify with Jev, escalate to a frontier LLM or human when confidence is below threshold).

**Caveats**

- Savings apply to decision calls, not to code or prose generation itself.
- Weaker on reasoning-heavy judgements than frontier LLMs; probabilities differ across decision-model providers, so thresholds must be calibrated per model.
- Documented limitation: injected instructions or self-arguing text in untrusted input can shift probabilities. Never make a single decision model the sole gate on destructive actions without deterministic policy checks or human approval.

## Local models (Apple Silicon)

RAM (unified memory) is what limits local models, not SSD size. M4 MacBooks go up to 32GB (M4), 48GB (M4 Pro) and 128GB (M4 Max).

| RAM | Reported pick |
|---|---|
| 16GB | Qwen 3.5-9B (Q4) |
| 24GB | Qwen 3.6-27B |
| 32GB+ | Qwen 3.6-35B-A3B or Qwen3-Coder-30B-A3B (about 19 GB) |
| 64GB+ | Qwen3-Coder-Next (Q4) |

- Runtimes: Ollama (MLX backend), LM Studio or `mlx_lm`; all expose OpenAI-compatible endpoints.
- Local 27-35B models handle routine steps well but trail frontier models on complex multi-tool work, which is where decision-model routing pays off.

## Agent frameworks (summary)

| Framework | Notes (as of Oct 2026) |
|---|---|
| Pydantic AI | Type-safe, model-agnostic; includes `pydantic_graph`, `pydantic_evals`, native `system_one` (Jev) & `openai_decisions` support, and co-maintained durable execution (`RuntimeCapability`) for DBOS, Temporal, and Restate ([Pydantic Docs](https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/)) |
| LangGraph | Stateful graphs, checkpointing; steeper curve |
| Claude Agent SDK | Coding-agent primitives, strong MCP support; Claude-only |
| OpenAI Agents SDK | Simple; avoid pairing with in-process LiteLLM |
| CrewAI | Role-based multi-agent; adds token cost |

## Visual builder landscape

- Flowise, Langflow, n8n and Dify offer drag-and-drop agent builders; all four are open-source with free self-hosting.
- Dify reports 130K+ GitHub stars and 100+ LLM providers.
- OpenAI's AgentKit includes a visual Agent Builder; Google Vertex AI Agent Builder and Microsoft Copilot Studio cover the enterprise side.

## Stack research (updated 17 Oct 2026)

**LiteLLM supply-chain incident.** In March 2026, PyPI versions 1.82.7 and 1.82.8 of LiteLLM were compromised with credential-stealing malware. This is why loomledger uses its own thin adapters (`httpx` + Pydantic) instead of LiteLLM. If an external gateway is needed later, run a compiled gateway as a separate process (e.g., Bifrost, GoModel, TensorZero) or use a managed gateway (OpenRouter, Cloudflare AI Gateway, Portkey).

**OpenTelemetry GenAI semantic conventions.** On 12 June 2026 (`v1.42.0`), the GenAI and MCP conventions moved into `semantic-conventions-genai` ([GitHub](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md); [Dash0 Guide](https://www.dash0.com/knowledge/opentelemetry-genai-semantic-conventions-explained)). Every `gen_ai.*` span, metric, and event remains in `Development` status with no tagged release yet. Key operations: `invoke_workflow`, `invoke_agent`, `chat`, `embeddings`, and `execute_tool` (which also covers MCP tool calls). Notably, prompts and completions are excluded from span attributes by default (`gen_ai.client.inference.operation.details` is opt-in), and the metric pair `gen_ai.invoke_agent.inference_calls` vs. `gen_ai.invoke_agent.tool_calls` provides a strong signal for catching agent loop bugs.

**Cryptographic AI audit trails & external anchoring.** A database hash chain alone fails if JSON serialization varies across languages, if tail/denied events are deleted, or if an operator with DB credentials rewrites the chain ([AgenticRail, 2026](https://agenticrail.nz/blog/cryptographic-ai-audit-trail/); [Activepieces, 2026](https://www.activepieces.com/blog/what-a-hash-chained-audit-trail-actually-means)). Best practice combines **RFC 8785 (JSON Canonicalization Scheme)**, a monotonic `run_seq` counter covering both permitted and denied steps, **Ed25519 signatures** with `key_id` rotation support, content-addressed payload hashes (so PII redaction/erasure never breaks the hash chain), and periodic chain-head anchoring to **S3 Object Lock (WORM)** ([AWS S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)) or **Sigstore Rekor** ([Sigstore Rekor](https://docs.sigstore.dev/logging/overview/)).

**Durable execution.** DBOS (`dbos-transact-py`, [DBOS GitHub](https://github.com/dbos-inc)) checkpoints workflow steps directly to PostgreSQL without requiring a separate workflow server ([DBOS Blog](https://www.dbos.dev/blog/postgres-is-all-you-need-for-durable-execution)), making it the cleanest post-MVP fit since Postgres is already in our stack.

**MCP security & sandboxing.** Because local MCP servers execute with client privileges by default ([MCP Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices); [DoD/NSA MCP Security Report, June 2026](https://media.defense.gov/2026/Jun/02/2003943289/-1/-1/0/CSI_MCP_SECURITY.PDF)), shell, file, and MCP tool calls must run in rootless containers or OS sandboxes (e.g., Bubblewrap) with per-node capability allowlists.

## Name check

"patchbay" was considered and rejected: several existing GitHub projects already use it, including AI-related ones. Always check GitHub, PyPI, npm and domains before settling on a name.

## Sources

- Jev 1.14 & Decisions API: https://openrouter.ai/typesafe/jev-1.14
- Jev on OpenRouter: https://openrouter.ai/docs/guides/community/jev
- TypeSafe System One: https://docs.typesafe.ai/concepts/system-one
- TypeSafe Introduction: https://docs.typesafe.ai/introduction
- OTel GenAI Semantic Conventions: https://www.dash0.com/knowledge/opentelemetry-genai-semantic-conventions-explained
- OTel GenAI Agent Spans: https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md
- Cryptographic AI Audit Trail: https://agenticrail.nz/blog/cryptographic-ai-audit-trail/
- Hash-Chained Audit Trails: https://www.activepieces.com/blog/what-a-hash-chained-audit-trail-actually-means
- Pydantic AI Durable Execution: https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/
- DBOS Durable Execution: https://www.dbos.dev/blog/postgres-is-all-you-need-for-durable-execution
- MCP Security Best Practices: https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices
- AWS S3 Object Lock (WORM): https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- Sigstore Rekor: https://docs.sigstore.dev/logging/overview/
