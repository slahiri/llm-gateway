# Tier 0 — Core Proxy Plumbing

Detailed specification for Tier 0 of the LLM Gateway. Parent
roadmap: [`features.md`](features.md).

## Goal

A working multi-provider proxy that any OpenAI- or Anthropic-
compatible client (Codex CLI, Claude Code, Aider, Cursor, Continue,
Cline, Windsurf, OpenWebUI, LibreChat, Anthropic SDK consumers) can
be pointed at and use immediately. Native wire-protocol endpoints,
faithful tool-call and streaming translation across provider shapes,
normalized errors, and a model registry with capability tags.

Tier 0 does **not** include caching, virtual keys, budgets, smart
routing, governance / PII, observability beyond per-request log
lines, or account management. Those land in later tiers.

## Definition of Done

Tier 0 is complete when all of the following are true:

- Three provider drivers connected and tested end-to-end: **OpenAI
  direct**, **Anthropic direct**, **AWS Bedrock** (Anthropic models
  on AWS). Bedrock is in the MVP set because it is the deployment
  shape regulated buyers prefer.
- Three wire-protocol endpoints live and behaviorally correct:
  `/v1/chat/completions`, `/v1/messages`, `/v1/embeddings`.
  `/v1/responses` is a stretch goal; firm Tier 1 deliverable.
- Streaming works end-to-end on all three including the cross-
  protocol pairs (OpenAI client + Claude backend, Anthropic client
  + GPT backend).
- Tool calls round-trip faithfully between OpenAI and Anthropic
  shapes — request, response, parallel tool calls, and streamed
  argument accumulation.
- Error taxonomy unified across providers; each endpoint renders
  errors in its native wire-format shape.
- Model registry loaded from YAML with capability tags. Cross-
  provider feature mismatches fail-fast with a clear error
  identifying the missing capability.
- Bearer-token auth against a static gateway-key store (in-memory
  map from YAML config for Tier 0; Postgres-backed virtual keys
  arrive in Tier 2).
- Per-request structured log line emitted on every request:
  `request_id`, `org_id`, `model`, `provider`, `endpoint`,
  `in_tokens`, `out_tokens`, `status`, `duration_ms`,
  `error_type`.
- `docker compose up` brings up the gateway and a bundled Postgres
  (Postgres unused in Tier 0 logic, but the infrastructure is in
  place for Tier 2).

## Wire-protocol endpoints

The gateway exposes multiple native wire-protocol endpoints in
parallel. Each is faithful to one provider's API shape. Tools point
their existing `BASE_URL` at the gateway and otherwise don't change.

| Endpoint | Native shape | Tier 0 status | Target tools |
|---|---|---|---|
| `POST /v1/chat/completions` | OpenAI Chat Completions | Required | Aider, Cline, Cursor, Continue, Windsurf, LibreChat, OpenWebUI |
| `POST /v1/messages` | Anthropic Messages | Required | Claude Code, Anthropic SDK |
| `POST /v1/embeddings` | OpenAI Embeddings | Required | Most retrieval pipelines |
| `POST /v1/responses` | OpenAI Responses | Stretch | Codex CLI, OpenAI Agents SDK |
| `POST /v1/images/generations` | OpenAI Images | Deferred to Tier 1+ | Image-gen clients |
| `POST /v1beta/models/*` | Google Gemini | Deferred | Vertex / Gemini SDK |
| `POST /api/chat` | Ollama | Deferred | Continue (Ollama mode), local-first tools |

### `/v1/chat/completions` (OpenAI Chat Completions)

- Faithful to the OpenAI Chat Completions spec.
- Supported request fields: `model`, `messages`, `temperature`,
  `top_p`, `max_tokens`, `stop`, `tools`, `tool_choice`,
  `response_format` (json_mode), `seed`, `stream`,
  `stream_options.include_usage`.
- `n` greater than 1 is rejected with `bad_request` in v0.1.
- `logit_bias`, `logprobs`, `top_logprobs` are passed through for
  native OpenAI routes; rejected for cross-provider routes that
  don't support them, with capability-tag-aware error.
- Streaming: SSE in OpenAI's chunked-delta format.

### `/v1/messages` (Anthropic Messages)

- Faithful to the Anthropic Messages API spec.
- Supported request fields: `model`, `messages`, `system` (top-
  level field), `max_tokens`, `temperature`, `top_p`, `top_k`,
  `stop_sequences`, `tools`, `tool_choice`, `stream`,
  `metadata.user_id`.
- Extended-thinking blocks supported for models with the
  `reasoning` capability tag.
- `cache_control` breakpoints supported for models with the
  `prompt_caching` capability tag; ignored with a warning on
  models that don't.
- Streaming: typed event stream — `message_start`, `content_
  block_start`, `content_block_delta`, `content_block_stop`,
  `message_delta`, `message_stop`, `ping`.

### `/v1/embeddings` (OpenAI Embeddings)

- Faithful to the OpenAI Embeddings spec.
- Supported request fields: `model`, `input` (string or array),
  `encoding_format`, `dimensions`, `user`.
- Cross-provider embedding models (Voyage, Cohere, Bedrock
  embeddings) translate response into OpenAI shape.

### `/v1/responses` (stretch)

- Faithful to OpenAI's Responses API.
- Critical for Codex CLI and the OpenAI Agents SDK.
- Different shape from Chat Completions — uses `input` instead of
  `messages`, returns `output[]` with typed items, has its own
  streaming events.
- Held as stretch for Tier 0 because the shape is larger and
  Codex specifically can also be configured against
  `/v1/chat/completions` as a fallback.

### Deferred endpoints

- `/v1/images/generations`: not on the agent-tool hot path; ship
  in Tier 1 with the rest of the multimedia surface.
- Gemini and Ollama native paths: Tier 1+. OpenAI- and Anthropic-
  shaped endpoints cover the vast majority of agent tooling.

## Provider drivers

One driver per upstream LLM provider. Each driver implements
**every wire-protocol method**, not just the ones matching its
native shape. The driver decides internally whether the call is a
passthrough (request shape matches provider's native shape — fast
path) or a translation (different shapes — translate, call, reverse-
translate).

### Interface (sketch, not fixed)

```go
type Provider interface {
    ID() string                        // "openai", "anthropic-direct", "bedrock"
    Native() WireFormat                // openai, anthropic, google, ...
    Models() []ModelDescriptor         // models this driver serves

    Chat(ctx, OpenAIChatRequest) (OpenAIChatResponse, ChatStream, error)
    Responses(ctx, OpenAIResponsesRequest) (OpenAIResponsesResponse, ResponsesStream, error)
    Messages(ctx, AnthropicMessagesRequest) (AnthropicMessagesResponse, MessagesStream, error)
    Embeddings(ctx, OpenAIEmbeddingsRequest) (OpenAIEmbeddingsResponse, error)
}
```

`ChatStream`, `MessagesStream`, `ResponsesStream` are
`io.ReadCloser`-shaped iterators yielding the appropriate event
type. The handler wraps them in the SSE format expected by the
calling endpoint.

### MVP provider set (Tier 0)

- **`openai-direct`** — `api.openai.com`. Native: openai.
- **`anthropic-direct`** — `api.anthropic.com`. Native: anthropic.
- **`bedrock`** — AWS Bedrock with SigV4 auth. Hosts Anthropic
  models (Claude on AWS) plus others. Native: bedrock-anthropic
  for Claude models; capability-tag-checked for the rest.

Three providers is intentionally small. Adding providers is
mechanical; getting the wire-format translations right is the
hard work that Tier 0 must finish first. Vertex, Azure OpenAI,
Gemini direct, and Ollama land in Tier 1.

### Driver configuration

Loaded from YAML at startup. Hot reload via SIGHUP is Tier 7; for
Tier 0 a restart is acceptable.

```yaml
providers:
  - id: openai-prod
    type: openai-direct
    base_url: https://api.openai.com
    api_keys:
      - env: OPENAI_API_KEY
  - id: anthropic-prod
    type: anthropic-direct
    base_url: https://api.anthropic.com
    api_keys:
      - env: ANTHROPIC_API_KEY
  - id: bedrock-us-east-1
    type: bedrock
    region: us-east-1
    # SigV4 via the standard AWS credential chain
```

## Request and response handling

The gateway does **not** invent a third "canonical" internal
format. Each request is routed as its input wire-format type and
translated on demand when the selected provider's native shape
differs. Translation logic lives in `internal/wire/translate/` with
one package per (from-format, to-format) pair.

This avoids the lossy-canonical problem (every translation step
discards provider-specific features) and keeps fast-path requests
truly fast. Native passthrough touches translation code zero times.

Translation pairs to implement in Tier 0:
- `openai_chat ↔ anthropic_messages` (request and response,
  including streaming)
- `openai_embeddings → cohere_embeddings` (request and response;
  one-shot, no streaming)
- `openai_embeddings → bedrock_embeddings` (request and response;
  one-shot)

Translation pairs deferred to Tier 1:
- Anything involving Gemini, Vertex, Azure, Ollama wire formats.
- `openai_responses ↔ anthropic_messages`.

## Tool / function-calling normalization

Tool-use shapes diverge across providers:

- **OpenAI**: `tools: [{type:"function", function:{name, parameters}}]`;
  response carries `tool_calls: [{id, type:"function",
  function:{name, arguments}}]`.
- **Anthropic**: `tools: [{name, description, input_schema}]`;
  response carries `content: [{type:"tool_use", id, name, input}]`
  interleaved with `text` blocks.

### Translation rules

| Concept | OpenAI | Anthropic |
|---|---|---|
| Tool definition | `function.name + function.parameters` | `name + input_schema` |
| Tool call request | `tool_calls[].function.arguments` (JSON string) | `content[].input` (JSON object) |
| Tool call result | `message{role:"tool", tool_call_id, content}` | `content[].tool_result{tool_use_id, content}` |
| Parallel tools | Multiple entries in `tool_calls[]` | Multiple `tool_use` content blocks |
| `tool_choice` | `auto` / `required` / `none` / `{type:"function", function:{name}}` | `auto` / `any` / `tool{name}` / `none` |

The translator preserves tool IDs across the round-trip so the
client's `tool_call_id` references resolve correctly when the
client passes back the tool result.

### Tool-call streaming

OpenAI streams tool-call arguments incrementally — each chunk
carries a fragment of the `arguments` JSON string. Anthropic
streams `input_json_delta` events inside a `tool_use` content
block.

Streaming translation rules:
- When the input is Anthropic-shaped and the output is OpenAI-
  shaped, accumulate `input_json_delta` events into a JSON string
  and emit one OpenAI delta per accumulated chunk boundary. Final
  emission on `content_block_stop`.
- When the input is OpenAI-shaped and the output is Anthropic-
  shaped, parse the streamed argument fragments; if not valid
  partial JSON, buffer until parseable. Emit `input_json_delta`
  events that maintain Anthropic's delta semantics.
- `tool_use.id` must be assigned at the start of the block and be
  consistent for the rest of the stream.

### `tool_choice` semantics

| Client says | OpenAI provider receives | Anthropic provider receives |
|---|---|---|
| OpenAI `auto` | `auto` | `auto` |
| OpenAI `required` | `required` | `any` |
| OpenAI `none` | `none` | `none` |
| OpenAI `{function:"x"}` | `{function:"x"}` | `tool{name:"x"}` |
| Anthropic `auto` | `auto` | `auto` |
| Anthropic `any` | `required` | `any` |
| Anthropic `none` | `none` | `none` |
| Anthropic `tool{x}` | `{function:"x"}` | `tool{name:"x"}` |

## Streaming (SSE)

Streaming is where most existing gateways quietly fall over. Tier 0
takes it seriously.

### State machines per (input, output) pair

For cross-protocol streaming, the gateway runs an explicit state
machine for each `(input wire format, output wire format)` pair.
There is no shared "canonical" stream representation — that
approach loses fidelity in subtle ways. Pairs in Tier 0:

- `openai_chat → openai_chat` — passthrough; framing transparent.
- `anthropic_messages → anthropic_messages` — passthrough.
- `anthropic_messages → openai_chat` — translate the typed event
  stream into OpenAI's `data: {delta}` chunks.
- `openai_chat → anthropic_messages` — translate OpenAI deltas
  into Anthropic's typed event stream.

### Backpressure

The gateway must not buffer unboundedly when the client is slow.
Implementation:
- `io.Pipe` between the provider reader and the client writer.
- Provider read is paused (via `context.Context` propagation) if
  the client writer blocks for longer than a configurable
  threshold (default: 30 seconds).
- Client disconnect cancels the inbound context; the provider
  call is aborted by the cancellation, and any in-flight tokens
  are accounted for in the per-request log.

### Mid-stream failure policy

When the upstream provider fails mid-stream (after the first byte
has been sent to the client):

- **Default: fail.** Emit a final error event in the output wire
  format and close the stream. The client sees a clean error and
  decides whether to retry.
- **Configurable: restart-on-fallback.** Buffer the prefix
  emitted so far in memory; on failure, attempt the same request
  against the next provider in the fallback chain (Tier 1
  delivers fallback chains). Re-emit the prefix verbatim before
  the new stream's tokens. This is dangerous — the client may
  have already acted on the prefix — and is off by default.

Tier 0 ships only the "fail" mode. The restart option is wired
through but disabled until Tier 1 lands fallback chains.

### Client disconnect handling

A client disconnect during streaming cancels the upstream provider
call immediately via `context.Context`. The per-request log
captures `status=client_disconnected` and the partial token
counts.

### Streamed token accounting

Both OpenAI and Anthropic emit usage information in their final
stream events. The gateway parses these and emits them in the per-
request log even when the client disconnected (the upstream call
may still have completed before disconnect propagated).

## Error normalization

A single canonical error taxonomy maps every provider error.

### Canonical taxonomy

| Code | HTTP | Meaning |
|---|---|---|
| `auth_failed` | 401 | Gateway or upstream auth failed |
| `permission_denied` | 403 | Key valid but not authorized for resource |
| `not_found` | 404 | Endpoint or path not found |
| `model_not_found` | 404 | Model alias not in registry |
| `bad_request` | 400 | Malformed request |
| `unsupported_capability` | 400 | Model lacks a requested capability |
| `context_length_exceeded` | 400 | Prompt exceeds model's `max_context` |
| `content_filtered` | 400 | Provider content filter blocked |
| `rate_limited` | 429 | Provider or gateway rate limit hit |
| `provider_overloaded` | 503 | Upstream returned 503-equivalent |
| `provider_timeout` | 504 | Upstream timed out |
| `network_error` | 502 | Network failure reaching provider |
| `internal_error` | 500 | Gateway bug or unexpected state |
| `client_disconnected` | n/a | Client closed connection mid-stream |

### Provider error mapping table

Each driver translates provider-specific error responses into the
canonical taxonomy. Maintained as a per-driver table in
`internal/provider/<name>/errors.go`. Unknown provider errors map
to `internal_error` and are logged for triage.

### Per-format error rendering

Each wire-protocol endpoint renders a canonical error in the
format that endpoint's clients expect:

- `/v1/chat/completions`, `/v1/embeddings`, `/v1/responses`:
  OpenAI error envelope `{error: {message, type, code, param}}`.
- `/v1/messages`: Anthropic error envelope `{type: "error",
  error: {type, message}}`.

The HTTP status code is set from the canonical code's mapping.

## Model registry and capability tags

Static YAML in Tier 0; can become dynamic / DB-backed later.

```yaml
models:
  - id: gpt-4o
    provider: openai-prod
    family: openai-gpt
    supports: [tools, vision, json_mode, structured_output, streaming, parallel_tools]
    max_context: 128000
    max_output: 16384

  - id: claude-opus-4-7
    provider: anthropic-prod
    family: anthropic-claude
    supports: [tools, vision, prompt_caching, reasoning, streaming, parallel_tools, structured_output]
    max_context: 1000000
    max_output: 64000

  - id: claude-opus-4-7-bedrock
    provider: bedrock-us-east-1
    family: anthropic-claude
    supports: [tools, vision, prompt_caching, streaming, parallel_tools]
    max_context: 200000
    max_output: 64000
```

### Capability tag vocabulary

| Tag | Meaning |
|---|---|
| `vision` | Accepts image inputs |
| `audio_in` | Accepts audio inputs |
| `audio_out` | Produces audio output |
| `tools` | Supports function/tool calling |
| `parallel_tools` | Can emit multiple tool calls in one response |
| `json_mode` | Has a JSON-mode flag |
| `structured_output` | Schema-bound output (JSON Schema) |
| `prompt_caching` | Honors `cache_control` breakpoints |
| `reasoning` | Produces explicit thinking / chain-of-thought output |
| `streaming` | Supports SSE streaming |

`max_context` and `max_output` are integers (tokens), not tags.

### Fail-fast on capability mismatch

When a client requests a feature a routed model doesn't have, the
gateway returns `unsupported_capability` with the missing tag
named. Examples:
- `/v1/chat/completions` request with image input parts, routed to
  a model without `vision`: 400 with
  `unsupported_capability: vision`.
- `/v1/messages` request with `cache_control` breakpoints, routed
  to a Bedrock Claude model that doesn't advertise
  `prompt_caching` in its tags: 400 with
  `unsupported_capability: prompt_caching`.

This is non-negotiable. Silent downgrades are how compliance
buyers get surprises.

## Authentication (Tier 0 scope)

- Bearer token in the standard `Authorization: Bearer <token>`
  header.
- Token format: `gw_live_<32-bytes-base64url>`.
- Token store in Tier 0: in-memory map loaded from a YAML config
  file. Tokens are stored as SHA-256 hashes with a static pepper;
  the YAML contains hashes, not raw tokens.
- Constant-time comparison on every check.
- The Tier 0 token grants full access to all configured providers
  and models. Multi-tenancy, scoping, budgets, and Postgres-
  backed virtual keys arrive in Tier 2.

Provider API keys never leave the gateway. The client sees only
gateway-issued tokens.

## Per-request observability hook

Every request, success or failure, emits exactly one structured
log line via `slog`:

```
request_id     uuid
org_id         string (placeholder "default" in Tier 0)
endpoint       "/v1/chat/completions" | "/v1/messages" | ...
model          string
provider       string
in_tokens      int
out_tokens     int
status         "ok" | "error" | "client_disconnected"
error_type     string (canonical error code, or empty)
duration_ms    int
ttfb_ms        int (time to first byte, streaming only)
```

Field names are stable from Tier 0 onward; observability dashboards
(Tier 4) will depend on them. No prompt content, no completion
content, no API keys, no PII — ever.

## Out of scope (deferred to later tiers)

- Retries with backoff, fallback chains, circuit breakers,
  multi-key rotation, health checks → **Tier 1**.
- Virtual keys with budgets, RPM/TPM rate limits, cost ledger,
  token-counting accuracy → **Tier 2**.
- Response cache (exact and semantic), connection pooling tuning
  → **Tier 3**.
- OpenTelemetry traces, Prometheus metrics, admin UI → **Tier 4**.
- PII redaction, audit log, RBAC, SSO, data residency → **Tier 5**.
- Model aliasing, cost / latency-aware routing, A/B & shadow,
  capability-tag-driven routing → **Tier 6**.
- OpenAI / Anthropic compat at the documentation level — done
  here as the API itself; tier 7 layers config-as-code, admin
  REST/gRPC, etc.
- Account management, billing, frontend → **Tiers 10–12**.

## Open questions

1. **`/v1/responses` in Tier 0 or Tier 1?** Stretch Tier 0, firm
   Tier 1. Codex CLI can be pointed at `/v1/chat/completions` as
   a fallback, but the Responses API is its real home and shipping
   it earlier is friendlier.
2. **Model aliasing in Tier 0?** Recommend Tier 1. Tier 0 uses
   real provider model IDs only.
3. **Single-provider key rotation within a wire-protocol endpoint
   (OpenAI key A → OpenAI key B on 429)** — Tier 1, since it's
   reliability, not core plumbing.
4. **Should the model registry support per-org overrides in Tier 0?**
   Recommend no. Multi-tenancy lands in Tier 2; Tier 0 is single-
   tenant.
