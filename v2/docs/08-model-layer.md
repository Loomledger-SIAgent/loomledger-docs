# 08 Model layer

_Updated 17 Oct 2026. Refines the "Model contract" sketch in [03 Architecture](03-architecture.md) and incorporates Jev 1.14 and the multi-provider Decisions API._

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
    input_tokens: int | None = None
    output_tokens: int | None = None
    cached_tokens: int | None = None
    reasoning_tokens: int | None = None
    cost_usd: Decimal | None = None   # None = unknown, never 0

class Capabilities(BaseModel):
    kinds: set[Literal["generate", "decide", "embed"]]
    decision_primitives: set[Literal["noul", "choice", "score", "box"]] = set()
    tools: bool = False
    structured_output: Literal["native", "tool", "json_mode", "none"] = "none"
    vision: bool = False
    default_context: int | None = None # e.g. ~8K tokens / 32KB fast mode for Jev
    max_context: int | None = None     # e.g. 262,144 tokens for Jev 1.14

class ModelNode(Protocol):
    id: ModelId                       # provider + name + pinned version
    capabilities: Capabilities
    async def invoke(self, req: Request, ctx: CallContext) -> Response: ...

# Request  = GenerateRequest | DecideRequest | EmbedRequest
# Response = GenerateResponse | DecideResponse | EmbedResponse
```

`GenerateRequest` carries messages, tools, an optional output schema and params. `DecideRequest` carries `state` (strings, JSON objects, arrays, or image content blocks) plus typed `questions`.

## Capabilities

Declared in a versioned catalog (`models.yaml`), not guessed at runtime, and checked at spec-validation time and again before each call. The catalog also holds the price table, with its own version (`catalog_version`) recorded on every audit event so costs can be reproduced later.

## Normalizing provider differences

| Difference | Approach |
|---|---|
| Message and content formats | One canonical format of typed blocks (text, tool call, tool result, image); adapters translate |
| Tool calling | Normalize to `ToolCall(id, name, args)`; validate args against the tool schema |
| Structured output | Ladder: native JSON schema, tool-call-as-schema, JSON mode, prompt and parse. Always validate with Pydantic; bounded retries that feed the error back; log each attempt |
| Usage fields | Normalize into `Usage`; unknown stays `None`; capture provider-reported `usage.cost` when available and verify against the catalog |
| Stop reasons and errors | Small enum; separate retryable (429, 5xx, timeout) from permanent errors |
| Provider-only features | `provider_options` escape hatch, always recorded in the audit event |

Store the raw provider request and response (as a content-addressed SHA-256 reference) next to the normalized version.

## Decisions / System One adapter (Jev 1.14 & OpenRouter Decisions API)

- **Two supported surfaces ([OpenRouter Jev Guide](https://openrouter.ai/docs/guides/community/jev)):**
  1. **System One API:** `POST https://api.typesafe.ai/v1/systemone` or `POST https://openrouter.ai/api/v1/systemone` with `{state, model, questions}`.
  2. **OpenRouter Decisions API:** `POST https://openrouter.ai/api/alpha/decisions`, which normalizes `typesafe/jev-1.14`, `typesafe/jev-1.13`, and 20+ other decision models (`openai/gpt-6-luna-decisions`, `microsoft/microsoft-decision-1`, `perplexity/pplx-decider-v1.2`, `cloudflare/clef`, etc.).
- **Question types:**
  - `noul`: yes/no condition, returns `noul` probability (0 to 1).
  - `choice`: 2–255 labelled options, returns selected `choice`, `probabilities` map, and `confidence`.
  - `score`: 2–10 ordered rubric levels, returns probability-weighted `score`, level `probabilities`, and `confidence`.
  - `box` (`typesafe/jev-1.14` only, requires `vision: true` and `box` in `decision_primitives`): locates a target in an image in `state`, returning normalized `box: {top, bottom, left, right}` and `confidence`.

```yaml
node: claim-risk
model: { kind: decide, provider: openrouter_decisions, name: typesafe/jev-1.14, version: "1.14" }
questions:
  risk:
    type: choice
    instructions: "How risky is this claim?"
    criteria: { low: "...", medium: "...", high: "...", other: "None of the above" }
```

Behaviours to enforce in the adapter and spec validator:

- **Pin the version.** Reject moving aliases (`jev-latest`, `~typesafe/jev-latest`) in saved specs because audit requires the exact version (`typesafe/jev-1.14`).
- **Capability-check `box` and images.** Reject `box` questions at spec-validation time unless the chosen model declares `box` in `decision_primitives` (Jev 1.13 and non-TypeSafe decision models reject `box` with HTTP 400). When `box` is used, warn or auto-pair with a `noul` presence check since `box` always returns four coordinates even if the object is absent.
- **Fast-mode vs. long-context warning.** Jev 1.14 supports 262,144 tokens, but payloads over ~32KB of text or containing images switch into TypeSafe's slower, less accurate long-context mode. Emit a trace attribute (`loomledger.decide.long_context_mode = true`) when state exceeds `default_context` (32KB).
- **No rationale.** Decision models return none, so the audit record holds probabilities, confidence, model version and request id only.
- **Probabilities are relative to the supplied labels.** Encourage an explicit `other / none` option or a `noul` gate; set thresholds in the graph.
- **LLM fallback must return the same shape.** A decision node falling back to an LLM returns a `DecideResponse` with probabilities and confidence via a strict Pydantic schema.

## Middleware order

1. Budget and policy check (per-run limits, allowed models, payload size vs `default_context` / `max_context`)
2. Redaction (apply the audit privacy policy before hashing/storing blobs)
3. Timeout and retry (exponential backoff + jitter, retryable errors only)
4. Fallback (next model in the chain must satisfy the same output type and capabilities; logged as its own audit event with the reason)
5. Cost and usage (from the versioned price table in `models.yaml`)
6. Audit and tracing (RFC 8785 hash-chained + Ed25519-signed event in `audit_events`, plus OTel GenAI spans)

Per-provider concurrency and rate limits sit alongside these.

## Secrets, replay and testing

- **Secrets:** specs hold references such as `secret://providers/openrouter`, never keys. Keys never enter prompts or logs.
- **Replay:** a replay adapter returns recorded responses keyed by canonical request hash; sampling parameters are recorded.
- **Fakes:** a scripted `FakeModel` supporting `generate`, `decide`, and `embed` so the runtime is testable without API calls.
- **Contract tests:** one shared suite every adapter must pass, plus recorded HTTP fixtures (`vcrpy` / `respx`) per provider.

## MVP scope and layout

Adapters: `openai_compat` (covers OpenAI, Ollama, LM Studio, OpenRouter chat), `anthropic`, `decisions` (TypeSafe System One + OpenRouter Decisions API), and `fake`. Defer Gemini native and embeddings to Phase 3.

```
loomledger/models/
  contracts.py        # requests, responses, usage, capabilities (noul/choice/score/box)
  registry.py         # spec -> adapter, secrets, pinned versions
  middleware/         # budget, redact, retry, fallback, cost, audit
  adapters/           # openai_compat.py, anthropic.py, decisions.py, fake.py
  catalog/models.yaml # capabilities + prices, versioned
```

## Risks

- **Abstraction leaks:** provider features like prompt caching and extended thinking do not map cleanly; keep the escape hatch logged and rare.
- **Stale catalogs:** prices and capabilities drift; version the catalog and record which version each run used.
- **Cross-model probability drift:** decision-model probabilities are not interchangeable across providers (e.g., `typesafe/jev-1.14` vs `openai/gpt-6-luna-decisions`); bind confidence thresholds to the pinned model ID.
