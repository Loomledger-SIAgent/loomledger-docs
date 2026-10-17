# 09 Audit and tracing

_Updated 17 Oct 2026. Completes the pending audit schema and OpenTelemetry mapping task noted in [03 Architecture](03-architecture.md) and [07 Decisions](07-decisions-and-open-questions.md)._

## Goals

1. **Tamper-evident by construction:** Any edit, reordering, or deletion of a permitted or denied step is cryptographically detectable offline.
2. **Privacy-safe:** Prompts, completions, and tool outputs never leak into indexed telemetry attributes and can be redacted or purged without breaking the hash chain.
3. **Deterministic replay:** Any historical run can be re-executed offline using content-addressed recorded responses.
4. **Standards-aligned observability:** Export OpenTelemetry GenAI spans and metrics alongside the audit store without relying on sampled telemetry as the system of record.

## Cryptographic receipt & hash-chain design

A database hash chain alone has three failure modes: cross-language JSON serialization drift, silent deletion of denied/tail records, and full-chain rewriting by an operator holding database credentials ([AgenticRail, 2026](https://agenticrail.nz/blog/cryptographic-ai-audit-trail/); [Activepieces, 2026](https://www.activepieces.com/blog/what-a-hash-chained-audit-trail-actually-means)). `loomledger` closes all three:

1. **RFC 8785 JSON Canonicalization Scheme (JCS):** Before hashing or signing, the event payload is serialized to canonical UTF-8 bytes (lexicographically sorted keys at every nesting level, no whitespace, deterministic number formatting).
2. **Dual monotonic counters (`run_seq` + `ledger_seq`):** Every event—including blocked tool calls, failed schema validations, and budget denials—receives a strictly sequential `run_seq` (1, 2, ... N) within the run and a global `ledger_seq`. Deleting a denied action leaves a provable gap in `run_seq`.
3. **SHA-256 hash chaining + Ed25519 signatures:**
   - `event_hash = SHA256(JCS(event_body_with_prev_hash))`
   - `signature = Ed25519_Sign(private_key, event_hash)` stored alongside `key_id` (e.g., `k1_2026-10_ed25519`) so historical receipts remain verifiable across key rotations.
4. **Content-addressed payload separation (`payload_blobs`):** Raw prompts, model completions, and tool I/O are stored in a separate table (or object store) keyed by `sha256` digest. The signed audit event stores only `input_hash` and `output_hash`. If a GDPR/privacy retention policy later purges a sensitive prompt blob, the audit chain and cryptographic signature remain 100% intact.
5. **External chain-head anchoring:** When a run completes (or every N minutes via a background worker), the latest signed chain head (`run_id`, `final_run_seq`, `head_event_hash`, `signature`, `timestamp_ms`) is pushed to an independent WORM sink—either **S3-compatible Object Lock** in Compliance/Governance mode ([AWS S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)) or a transparency log such as **Sigstore Rekor** ([Sigstore Rekor](https://docs.sigstore.dev/logging/overview/)).

## Event schema (PostgreSQL)

```sql
CREATE TABLE audit_events (
    ledger_seq        BIGSERIAL PRIMARY KEY,
    event_id          UUID NOT NULL UNIQUE,
    run_id            UUID NOT NULL,
    run_seq           INTEGER NOT NULL,
    ts_ms             BIGINT NOT NULL,
    event_kind        TEXT NOT NULL, -- 'run_start' | 'node_invoke' | 'model_call' | 'tool_call' | 'policy_gate' | 'approval_gate' | 'fallback' | 'run_end'
    node_id           TEXT,
    decision          TEXT NOT NULL, -- 'ALLOW' | 'DENY' | 'ESCALATE' | 'ERROR'
    model_id          TEXT,          -- e.g. 'typesafe/jev-1.14' (always pinned)
    catalog_version   TEXT NOT NULL, -- version of models.yaml used for pricing/capabilities
    trace_id          TEXT NOT NULL, -- W3C 32-hex trace ID
    span_id           TEXT NOT NULL, -- W3C 16-hex span ID
    parent_span_id    TEXT,
    input_hash        TEXT,          -- sha256 of canonical redacted input blob
    output_hash       TEXT,          -- sha256 of canonical redacted output blob
    raw_provider_hash TEXT,          -- sha256 of raw provider request/response envelope
    usage_json        JSONB,         -- input_tokens, output_tokens, cached_tokens, reasoning_tokens, cost_usd
    confidence        NUMERIC(5,4),  -- populated for Decide nodes (e.g. Jev)
    meta_json         JSONB NOT NULL DEFAULT '{}'::jsonb,
    prev_event_hash   TEXT NOT NULL, -- 'GENESIS' for run_seq = 1
    event_hash        TEXT NOT NULL,
    key_id            TEXT NOT NULL,
    signature         TEXT NOT NULL,
    UNIQUE (run_id, run_seq)
);

CREATE TABLE payload_blobs (
    blob_hash         TEXT PRIMARY KEY, -- sha256 hex digest
    created_at_ms     BIGINT NOT NULL,
    redaction_policy  TEXT NOT NULL,
    content_json      JSONB,            -- nullable if purged under retention policy
    purged_at_ms      BIGINT
);
```

Database triggers forbid `UPDATE` and `DELETE` on `audit_events`.

## OpenTelemetry GenAI mapping

The OpenTelemetry GenAI semantic conventions moved into `semantic-conventions-genai` in `v1.42.0` (12 June 2026) and remain in `Development` status ([Dash0 OTel GenAI Guide](https://www.dash0.com/knowledge/opentelemetry-genai-semantic-conventions-explained); [OpenTelemetry GenAI Agent Spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md)). Keep all attribute names isolated in `loomledger/audit/otel_mapping.py`:

| `loomledger` scope | OTel `gen_ai.operation.name` | Span name format | Key attributes |
|---|---|---|---|
| Workflow run | `invoke_workflow` | `invoke_workflow {workflow.name}` | `gen_ai.workflow.name`, `loomledger.run_id` |
| LLM agent node | `invoke_agent` | `invoke_agent {agent.name}` | `gen_ai.agent.name`, `gen_ai.agent.id`, `gen_ai.agent.version` |
| Model call (LLM / Decide / Embed) | `chat` / `decide` / `embeddings` | `{operation} {request.model}` | `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.cache_read.input_tokens`, `gen_ai.usage.reasoning.output_tokens` |
| Tool / MCP execution | `execute_tool` | `execute_tool {tool.name}` | `gen_ai.tool.name`, `gen_ai.tool.call.id`, `gen_ai.tool.type` |

### Loop-detection & FinOps metrics

Emit the OTel GenAI metrics alongside spans:
- `gen_ai.client.token.usage` and `gen_ai.client.operation.duration`
- `gen_ai.invoke_agent.inference_calls` and `gen_ai.invoke_agent.tool_calls` — tracking the ratio of model calls to tool calls per agent run catches deliberative infinite loops early before budget exhaustion ([Dash0, 2026](https://www.dash0.com/knowledge/opentelemetry-genai-semantic-conventions-explained)).
- Never attach prompt or completion strings to span attributes; export metadata on spans by default and allow opt-in `gen_ai.client.inference.operation.details` events only in local/staging environments.
