# 08 Model layer

_Draft, 9 Oct 2026. Refines the "Model contract" sketch in [03 Architecture](03-architecture.md)._

The model layer has one job: take a typed request for a node, call the right model, and return a normalized result, with every call checked, costed, retried and logged. It contains no agent logic or orchestration.

## Structure

```
node invoke ─► middleware chain ─► adapter ─► provider API
               budget · redaction · retry/timeout · fallback · cost · audit
```

- **Contract (`ModelNode`):** the interface everything implements.
- **Adapters:** one per API family; translate canonical requests to provider calls and back.
- **Registry:** resolves a spec's `model:` block to a configured adapter with a pinned version and a secret reference.
- **Middleware:** cross-cutting behaviour wrapped around every call, so adapters stay simple.

## Contract: request kinds, not just "chat"

Separate request and response kinds avoid forcing non-LLMs into a chat shape.

```python
class Usage(BaseModel):
    input_tokens: int | None; output_tokens: int | None
    cached_tokens: int | None; reasoning_tokens: int | None
    cost_usd: Decimal | None          # None = unknown, never 0

class Capabilities(BaseModel):
    kinds: set[Literal["generate", "decide", "embed"]]
    tools: bool = False
    structured_output: Literal["native", "tool", "json_mode", "none"] = "none"
    vision: bool = False
    max_context: int | None = None

class ModelNode(Protocol):
    id: ModelId                       # provider + name + pinned version
    capabilities: Capabilities
    async def invoke(self, req: Request, ctx: CallContext) -> Response: ...

# Request  = GenerateRequest | DecideRequest | EmbedRequest
# Response = GenerateResponse | DecideResponse | EmbedResponse
```

`GenerateRequest` carries messages, tools, an optional output schema and params. `DecideRequest` carries state plus typed questions.

## Capabilities

Declared in a versioned catalog (YAML), not guessed at runtime, and checked at spec-validation time and again before each call. The catalog also holds the price table, with its own version so costs can be reproduced later.

## Normalizing provider differences

| Difference | Approach |
|---|---|
| Message and content formats | One canonical format of typed blocks (text, tool call, tool result, image); adapters translate |
| Tool calling | Normalize to `ToolCall(id, name, args)`; validate args against the tool schema |
| Structured output | Ladder: native JSON schema, tool-call-as-schema, JSON mode, prompt and parse. Always validate with Pydantic; bounded retries that feed the error back; log each attempt |
| Usage fields | Normalize into `Usage`; unknown stays `None` |
| Stop reasons and errors | Small enum; separate retryable (429, 5xx, timeout) from permanent errors |
| Provider-only features | `provider_options` escape hatch, always recorded in the audit event |

Store the raw provider request and response (as a hashed reference) next to the normalized version.

## Jev adapter (decision model)

- Endpoint and body: `POST https://api.typesafe.ai/v1/systemone` with `{state, model, questions}`.
- Question types: `noul` (yes/no, one probability), `choice` (2-255 labelled options), `score` (2-10 ordered levels). Answers carry probabilities and confidence.

```yaml
node: claim-risk
model: { kind: decide, provider: jev, version: jev-1.13.0 }
questions:
  risk:
    type: choice
    instructions: "How risky is this claim?"
    criteria: { low: "...", medium: "...", high: "..." }
```

Behaviours to get right:

- **Pin the version.** A moving `latest` alias exists; reject it in saved specs because audit needs the exact version.
- **No rationale.** Jev returns none, so the audit record holds probabilities, confidence, model version and request id only.
- **Probabilities are relative to the supplied labels.** One label can score high even when none fits. Add an explicit "other / none" option or a yes/no gate; set thresholds in the graph.
- **Scores are for ranking and thresholding, not measuring.**
- **Size limits.** One gateway lists a 32K context; expose `max_context` and check payload size first.
- **LLM fallback must return the same shape.** A decision node falling back to an LLM returns a `DecideResponse` with probabilities and confidence via a strict schema. An independent open-source project implements the Jev interface on top of OpenAI-compatible models, which shows the pattern works; copy the idea, not the dependency.

## Middleware order

1. Budget and policy check (per-run limits, allowed models, size)
2. Redaction (apply the audit privacy policy before logging)
3. Timeout and retry (backoff, retryable errors only)
4. Fallback (next model in the chain must satisfy the same output type; logged as its own event with the reason)
5. Cost and usage (from the versioned price table)
6. Audit and tracing (start and end events, OTel spans)

Per-provider concurrency and rate limits sit alongside these.

## Secrets, replay and testing

- **Secrets:** specs hold references such as `secret://providers/openai`, never keys. Keys never enter prompts or logs.
- **Replay:** a replay adapter returns recorded responses keyed by request hash; sampling parameters are recorded.
- **Fakes:** a scripted `FakeModel` so the runtime is testable without API calls.
- **Contract tests:** one shared suite every adapter must pass, plus recorded fixtures per provider.

## MVP scope and layout

Adapters: OpenAI-compatible (also covers Ollama and LM Studio), one hosted provider (for example Anthropic), Jev, and Fake. Defer Gemini and embeddings.

```
loomledger/models/
  contracts.py        # requests, responses, usage, capabilities
  registry.py         # spec -> adapter, secrets, pinned versions
  middleware/         # budget, redact, retry, fallback, cost, audit
  adapters/           # openai_compat.py, anthropic.py, jev_systemone.py, fake.py
  catalog/models.yaml # capabilities + prices, versioned
```

## Risks

- **Abstraction leaks:** provider features like prompt caching and extended thinking do not map cleanly; keep the escape hatch logged and rare.
- **Stale catalogs:** prices and capabilities drift; version the catalog and record which version each run used.
- **Silent degradation:** fallbacks can hide quality loss unless logged and reviewed.
