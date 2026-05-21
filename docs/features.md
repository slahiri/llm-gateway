# LLM Gateway — Feature Roadmap

A Go-based LLM gateway **platform**: proxy + governance + multi-tenant
account management + metered billing + admin frontend. Distributed as
a hosted SaaS, with the same binary self-hostable for OSS adopters.

Tiers 0–3 are MVP for the proxy engine; Tiers 4–9 round out the engine
(observability, compliance, routing intelligence, DX, agentic, polish).
Tiers 10–12 are the platform layer (accounts, billing, frontend) — not
optional for the SaaS but gated by config flag so a regulated self-host
buyer can run the gateway alone.

The differentiation wedge is **compliance-first for regulated industries**
— see the wedge section below the tiers.

---

## Tier 0 — Core proxy plumbing (week 1–2)

**Detailed spec: [`features-tier-0.md`](features-tier-0.md).** Summary
below; that doc is the source of truth for Tier 0.

- **Wire-protocol endpoints in parallel** — native surfaces, not one
  canonical shape. Tier 0 ships four required endpoints:
  `/v1/chat/completions` (OpenAI Chat Completions),
  `/v1/responses` (OpenAI Responses — for Codex CLI and the OpenAI
  Agents SDK), `/v1/messages` (Anthropic Messages), and
  `/v1/embeddings` (OpenAI Embeddings). Gemini, Ollama, and image
  endpoints land in Tier 1+.
- **Agent-tool and IDE compatibility is a Tier 0 acceptance
  criterion** — Codex CLI, Claude Code, Aider, Cursor, Continue,
  Cline, Windsurf, OpenWebUI, LibreChat, and the OpenAI / Anthropic
  SDKs all work against the gateway with only a `BASE_URL` change.
  Full setup matrix in [`features-tier-0.md`](features-tier-0.md).
- **Provider drivers, dual-mode** — one driver per upstream provider;
  each driver implements every wire-protocol method. Native shape
  matches → passthrough fast path; shape differs → in-driver
  translation. MVP set in Tier 0: OpenAI direct, Anthropic direct,
  AWS Bedrock (Anthropic on AWS). Vertex / Azure OpenAI / Gemini
  direct / Ollama in Tier 1.
- **No invented internal canonical format** — requests are routed as
  their input wire-format type and translated on demand. Translation
  packages live in `internal/wire/translate/`, one per
  (from-format, to-format) pair.
- **Streaming (SSE)** — explicit state machine per (input, output)
  pair. Backpressure via `io.Pipe`. Client disconnect cancels the
  upstream context. Default mid-stream failure policy is **fail**;
  restart-on-fallback is wired through but off by default until
  Tier 1 lands fallback chains.
- **Tool / function calling normalization** — OpenAI `tool_calls`
  arrays ↔ Anthropic `tool_use` content blocks, including parallel
  tool calls and streamed argument-JSON accumulation. `tool_choice`
  semantics translated per a documented table.
- **Error normalization** — single canonical taxonomy:
  `auth_failed`, `permission_denied`, `model_not_found`,
  `bad_request`, `unsupported_capability`, `context_length_exceeded`,
  `content_filtered`, `rate_limited`, `provider_overloaded`,
  `provider_timeout`, `network_error`, `internal_error`,
  `client_disconnected`. Each endpoint renders errors in its
  wire-format-native envelope.
- **Model registry with capability tags** — YAML-loaded for Tier 0;
  every model carries `supports[]`, `max_context`, `max_output`.
  Cross-provider feature mismatches fail-fast with
  `unsupported_capability` and the missing tag named. Silent
  downgrades are forbidden.
- **Tier 0 auth** — bearer tokens against an in-memory store loaded
  from YAML (SHA-256 + pepper hashes). Postgres-backed virtual keys
  arrive in Tier 2.
- **Per-request log line** — one structured log per request with
  stable field names: `request_id`, `org_id`, `endpoint`, `model`,
  `provider`, `in_tokens`, `out_tokens`, `status`, `error_type`,
  `duration_ms`, `ttfb_ms`. No prompt or completion content.

## Tier 1 — Reliability (week 2–4)

- **Retries** with exponential backoff + jitter, configurable per error
  class.
- **Fallback chains** — `primary: claude-opus-4.7 → fallback: gpt-4 →
  fallback: gemini-pro`. Stream-safe: define policy when primary fails
  mid-stream (fail vs. restart on fallback).
- **Circuit breakers** per provider/region with half-open probing.
- **Timeout management** — separate connect, first-byte, total
  timeouts.
- **Multi-key rotation** — round-robin or weighted across multiple keys
  per provider to dodge per-key rate limits.
- **Health checks** — passive (track 5xx rates) and active (cheap
  synthetic pings).

## Tier 2 — Cost & rate governance (week 3–5)

- **Virtual keys** — gateway-issued keys mapped to real provider keys +
  a policy bundle (budget, rate, model allowlist, team).
- **Budget caps** — hard and soft, per virtual key / team / user /
  time window.
- **Rate limits** — RPM and TPM, token-bucket with Redis backend in
  distributed mode.
- **Token counting** — pre-flight estimation (tiktoken-go, Anthropic
  tokenizer) and post-flight actuals from response.
- **Cost ledger** — per-request cost computed from a maintained price
  catalog; query API exposed.

## Tier 3 — Performance (week 4–6)

- **Exact-match response cache** — keyed on hash of
  `(model, messages, params)`. TTL configurable.
- **Semantic cache** — embedding-based; pluggable embedding backend;
  similarity threshold + invalidation rules. Hard to get right for
  streaming.
- **Connection pooling + HTTP/2** — keep-alive to provider endpoints.
- **Goroutine fan-out for fallback racing** — optionally fire top-2
  providers in parallel; use whichever streams first. Premium feature,
  high cost.

## Tier 4 — Observability (week 5–7)

- **OpenTelemetry** traces and metrics out of the box. Span per
  request, child spans per provider attempt.
- **Prometheus metrics** — latency histograms (per provider, per
  model), token counters, cost gauges, cache hit rates, error counts.
- **Structured logs** with provider request/response capture
  (redaction toggle).
- **Web UI** — see Tier 12 for the full frontend spec. Embedded via
  `embed.FS` into the Go binary so the gateway ships as one artifact.

## Tier 5 — Governance & compliance (the wedge)

- **PII detection + redaction** — pluggable detectors (regex,
  rule-based, LLM-based); redact on request, optionally on
  response.
- **Audit log** — immutable, append-only, signed entries. WORM-friendly
  storage adapter (S3 Object Lock).
- **RBAC** — teams, roles, scopes on virtual keys.
- **SSO** — OIDC and SAML.
- **Data residency rules** — route enforcement so EU traffic never hits
  US endpoints, IN traffic stays on Indian-region providers, etc.
- **Content moderation hooks** — pre- and post-call hooks to invoke
  moderation models or rules engines.

## Tier 6 — Routing intelligence

- **Model aliasing** — `model: "smart"` resolves per-config to a real
  model. Org-wide swaps without client changes.
- **Cost-aware routing** — pick the cheapest model meeting a capability
  tag (vision, json_mode, ctx_length).
- **Latency-aware routing** — track p95 per provider, route
  accordingly.
- **A/B and shadow traffic** — split N% to a candidate model, log both
  for comparison without affecting the user.
- **Capability tags** on models so routers can filter:
  `supports: [vision, tools, json_mode, 200k_ctx]`.

## Tier 7 — Developer experience

- **OpenAI-compatible endpoint** — `/v1/chat/completions`,
  `/v1/embeddings`, `/v1/images/generations`. Existing SDKs work
  unchanged. Biggest adoption lever.
- **Anthropic-compatible endpoint** — `/v1/messages` with the same
  passthrough story.
- **Config as code** — YAML with schema, hot reload via SIGHUP or
  fsnotify. No required DB for basic mode (file + in-memory).
- **Admin REST + gRPC API** for keys, budgets, policies.
- **Single static binary** — no runtime deps. Optional Postgres /
  ClickHouse adapters for prod-scale analytics.

## Tier 8 — Agentic-era features (2026 differentiation)

- **MCP-native routing** — proxy MCP server traffic so the gateway sees
  tool calls and their results. Most existing gateways are blind here.
- **Agent budgets** — track spend per-agent-task (parent trace ID), not
  per-request. "This agent run cost $0.42" instead of "this call cost
  $0.003."
- **Multi-step trace stitching** — auto-correlate sequential calls
  within an agent run.
- **Local model adapters** — Ollama, vLLM, llama.cpp as first-class
  providers. Important for hybrid deployments.
- **Embedding cache** — separate from response cache; embeddings are
  expensive and highly cacheable.

## Tier 9 — Enterprise

- Multi-tenancy with hard isolation (tenant ID stamped on every row;
  cross-tenant query default-denied at the store layer).
- HSM/KMS-backed provider key storage (AWS KMS, GCP KMS, Vault).
- In-VPC deploy mode (no phone-home, fully offline).
- ClickHouse adapter for analytics at scale (async read replica of
  the usage ledger; reporting queries do not touch Postgres).
- Helm chart + Terraform module.
- **Docker Compose for OSS adopters** — `docker-compose up` brings
  the gateway and a bundled Postgres up together. One command to
  try the project, even though Postgres is required.

## Tier 10 — Account management

- **Identity model**: User → Team → Organization. Personal accounts
  are a Team-of-one. All resources (keys, budgets, audit entries,
  usage events) are stamped with `org_id` for tenant isolation.
- **Auth**: email + password (Argon2id), magic-link, OAuth (Google /
  GitHub for SaaS). SSO via OIDC and SAML reuses Tier 5 for
  enterprise customers.
- **Invitations**: email-based, role-scoped, expiring tokens.
- **API key issuance**: per user, per team, per org. Naming,
  last-used tracking, rotation, revocation. These ARE the virtual
  keys from Tier 2 — the same object viewed from the account-
  management side.
- **Workspaces / projects** (optional): sub-org grouping for teams
  separating dev/staging/prod budgets and keys.
- **Self-service org settings**: name, logo, default model alias,
  default residency rule, default budget.

## Tier 11 — Billing & monetization

**Pricing model: pure usage-based.** No subscriptions, no seat fees,
no monthly minimums. Customers pay for what they consume, billed
against a prepaid credit balance. Postpaid (NET-30 invoiced) is
available as a sales-touched enterprise option.

- **Rate card**: per-(model, direction, token) pricing maintained
  centrally. Charged as pass-through provider cost × gateway markup
  (markup configurable: percentage, fixed per-token, or both).
  Customers always see the underlying provider rate plus the
  transparent gateway fee — no hidden lock-in pricing.
- **Metered usage ledger**: every successful call writes an append-
  only usage event (timestamp, org, key, model, provider, in/out
  tokens, cached flag, provider-cost-USD, gateway-fee-USD,
  total-USD). Source of truth for billing — never updated, only
  appended. Hot table in Postgres; asynchronously replicated to the
  analytics tier (Tier 9).
- **Prepaid credit balance** (default for self-serve):
  - Customer tops up via Stripe Checkout; credit balance is the
    spendable unit.
  - Every call decrements the balance in near-real-time (debit
    written in the same transaction as the usage event).
  - Hard cutoff at zero — virtual keys reject new requests.
  - **Auto-top-up rules**: configurable threshold + amount (e.g.,
    "when balance drops below $20, charge $100"). Triggers a Stripe
    charge on the saved payment method.
  - Negative-balance protection: when the call's cost can't be
    predicted upfront (streaming, unknown output length), a small
    holdback is reserved against the balance and reconciled on
    response.
  - **Free initial credit**: $5 on signup (configurable, off by
    default in self-host).
- **Postpaid invoicing** (enterprise, opt-in):
  - Monthly aggregation of the usage ledger into a PDF invoice with
    line items per virtual key.
  - NET-30 terms by default, configurable per contract.
  - Manual approval flow in the admin UI; not self-serve.
- **Payment processor abstraction**: a `BillingProvider` interface
  with Stripe as the default implementation. Regional alternates
  (Razorpay for India, etc.) added when market demand appears.
  Self-host can plug in their own or run with billing disabled
  entirely.
- **Customer billing portal**: use Stripe Customer Portal for
  payment-method management, invoice history, and receipts. Do not
  build payment forms in v1.
- **Tax handling**: delegated to the processor (Stripe Tax). GST,
  VAT, sales tax computed by the processor at checkout / invoice
  time.
- **Spend caps** (critical without plans for ceiling): hard and
  soft caps at org, team, and key level — daily, weekly, monthly,
  rolling. Soft cap sends a webhook + email; hard cap stops new
  requests. These are the safety rails that subscription plans
  would have provided.
- **Dunning + payment failure**: built-in retries on top-up
  failures; configurable grace period before hard-disable; email
  notifications.
- **Refunds / credits / manual adjustments**: admin-only operations
  on the credit balance, audit-logged through Tier 5.
- **Feature entitlements** (decoupled from pricing): some features
  are gated by org-level entitlements (e.g., semantic cache,
  advanced guardrails, SSO, enterprise audit retention) that an
  admin toggles. Sales determines who gets what — entitlements are
  not tied to a plan tier, since there are no tiers.

## Tier 12 — Web frontend (Vite + React)

- **Stack**: Vite, React 19, TypeScript (strict mode), TanStack
  Query for data, TanStack Router for routing, Tailwind + shadcn/ui
  for styling, Recharts for usage graphs.
- **Build target**: embedded into the Go binary via `embed.FS` so
  the gateway ships as one artifact. Vite's `dist/` output is
  `go:embed`-ed and served from `/admin/`. OSS users get the full
  UI with no separate deployment step.
- **Routes**:
  - `/admin/dashboard` — usage, cost, latency, error rates at a
    glance.
  - `/admin/keys` — virtual key CRUD, scoping, budgets.
  - `/admin/usage` — drill-down by model, provider, key, time
    range.
  - `/admin/billing` — credit balance, top-up flow, auto-top-up
    rules, payment method, transaction history, invoices /
    receipts.
  - `/admin/teams` — org / team / user management, invitations,
    roles.
  - `/admin/audit` — compliance audit log viewer (filter by user,
    IP, action, time).
  - `/admin/requests` — live request explorer with redaction
    toggle.
  - `/admin/settings` — SSO config, residency rules, guardrail
    policies, model catalog.
- **Auth for the UI**: session cookie + CSRF token. The same admin
  REST API is also callable with API keys for programmatic admin.
- **Real-time updates**: SSE from the Go side for live request
  stream and live token counters. No WebSockets needed.
- **Frontend in dev mode**: Vite dev server runs on a separate port
  during development; the Go binary serves the API and proxies
  unknown routes to the Vite dev server when `--dev` is set.

---

## Positioning — the regulated-industries wedge

Speed and provider breadth are already taken lanes. The differentiator
is **"the LLM gateway built for regulated industries"** (DFSA / MAS /
SEBI / banking / wealth):

- Field-level redaction with jurisdiction-aware rules out of the box
  (PCI, PII, account numbers, PAN, Aadhaar).
- Data residency enforcement as a first-class concept, not a config
  afterthought.
- Audit trails that map directly to SEBI / DFSA / MAS evidence
  requirements.
- Tenant isolation strong enough for B2B SaaS sold into banks.
- Bundled compliance reports — who accessed what model with what data,
  exportable for auditors.

The market is full of "fastest" and "most providers." There is no
compliance-first OSS option today, and that is the gap regulated
buyers cite when rejecting existing gateways. Commercial story:
OSS core, paid enterprise add-ons (SSO, HSM, advanced audit).

---

## Storage architecture

**Postgres is the only supported database.** Hosted SaaS runs on
managed Postgres (RDS / Cloud SQL / Supabase). OSS adopters get a
`docker-compose.yml` that brings up Postgres + the gateway together,
so the install is still one command.

Design rules that keep the storage layer clean without building a
premature abstraction:

1. **All DB code lives in `internal/store/`**. Nothing outside that
   package imports a SQL driver or knows table names. When swapping
   to MySQL or pluggable backends becomes a real requirement, the
   change is scoped to one package.
2. **Driver**: `pgx/v5` directly (not `database/sql` with a `pgx`
   driver — native `pgx` is more idiomatic and exposes Postgres
   features cleanly when needed).
3. **Migrations**: `golang-migrate` with the `postgres` driver.
   Versioned `.sql` files in `internal/store/migrations/`. Run on
   boot in dev; explicit `gateway migrate up` in prod.
4. **Functions, not interfaces, until needed**. Start with
   package-level functions like `store.GetUser(ctx, db, id)`. The
   day a second backend becomes real, extract an interface from
   the existing surface — that's a one-hour refactor in Go.
5. **Use Postgres-native features where they help** — JSONB for
   request/response blobs and audit-log entries, partial indexes
   for tenant-scoped hot paths, `INSERT ... ON CONFLICT` upserts,
   `LISTEN/NOTIFY` for cross-instance cache invalidation, advisory
   locks for migration coordination. No artificial portability
   constraint.
6. **Three logical data classes, separated even when co-located**:
   - **Transactional** (users, orgs, keys, subscriptions, plans) —
     ACID requirements high, volume low. Main Postgres tables.
   - **Append-only ledger** (usage events, audit log) — never
     updated, only appended. Partition by month. Replicate to the
     analytics tier (Tier 9 ClickHouse) for reporting queries.
   - **Ephemeral** (sessions, rate-limit counters, cache) — use
     Redis once distributed; in-memory (`ristretto`) for the
     single-node case.
7. **Tenant isolation as a column, not a database**. Every
   tenant-owned table has an `org_id` column with an index. A
   middleware-level helper rejects queries that don't include a
   tenant scope. Schema-per-tenant is overkill at this stage.
8. **Connection management**: `pgxpool` for the main pool; a
   separate small pool for long-running queries (analytics, audit
   exports) so they don't starve the hot path.

## Stack

- **HTTP**: `net/http` + `chi` (or `fiber` for raw throughput).
- **Streaming**: `bufio.Scanner` for SSE; `io.Pipe` for fan-out.
- **State**: Postgres via `pgx/v5` + `pgxpool`; migrations via
  `golang-migrate`.
- **Cache**: in-memory (`ristretto`) with Redis adapter when
  multi-instance.
- **Config**: `koanf` or `viper`.
- **Observability**: `otel-go`, `prometheus/client_golang`, `slog`.
- **Tokenizers**: `tiktoken-go`; roll your own for Anthropic (or
  vendor their tokenizer JSON).
- **Payments**: `stripe-go` for the default billing provider
  implementation.
- **Frontend**: Vite + React 19 + TypeScript; embedded via
  `embed.FS`.

---

## Market scan — what existing gateways ship

Synthesized from primary-source reviews of the five most-cited
projects in the category. Names omitted intentionally; what matters is
the shape of the market, not the leaderboard.

### Archetype A — Broad Python aggregator
- Python web service; MIT core with paid Enterprise add-ons. Runs as
  a service, not a single binary.
- 100+ providers, the broadest coverage in the field. OpenAI- and
  Anthropic-compatible endpoints. Full MCP gateway with multi-auth
  modes.
- Multiple routing strategies (latency-based, usage-based,
  least-busy, cost-based, shadow), multi-key rotation, semantic +
  exact cache across multiple Redis/vector-DB/object-store backends,
  virtual keys + budgets + cost ledger, OTel + Prometheus.
- SSO, RBAC, audit logs, secret managers, key rotation — **all
  paywalled**. PII handling is via third-party guardrail
  integrations, not built-in.
- Self-published perf: ~8 ms P95 overhead at 1k RPS on a 4-instance
  cluster.
- **Structural openings**: recurring memory-leak issues across the
  tracker, hot-path inefficiencies in spend tracking, latency-routing
  race condition known to degrade to random, mid-stream fallback
  semantics not documented. Maintainers have publicly acknowledged
  the scope-vs-quality strain. A Go implementation has room to win
  on stability and per-node throughput.

### Archetype B — Go-based performance gateway
- Go, Apache-2.0, single binary; OSS core + Enterprise tier.
- ~25 providers including Ollama and vLLM. Drop-in OpenAI, Anthropic,
  Google endpoints.
- CEL-based dynamic routing with VK > Team > Customer > Global
  scopes; weighted A/B/canary; capacity-aware (budget %, tokens %,
  request %); adaptive load balancing (Enterprise) blending error
  rate, latency, utilization with an exploration probe. Dual-layer
  cache (exact + semantic) shipped by default across multiple
  vector-store backends.
- Hierarchical Customer → Team → VK budgets; rate limits are VK-only
  (a real gap — Teams and Customers can't set RPM/TPM).
- OTel (GenAI semconv) and Prometheus native; built-in web UI.
- Enterprise: RBAC + SCIM, SSO across major IdPs, **HMAC-signed audit
  logs with 365-day default retention and native SIEM export**,
  CEL-based guardrails, PII via regex + provider integration,
  prompt-injection via provider integration.
- Compliance: marketed as "designed for" SOC 2 / GDPR / HIPAA / ISO
  27001 — **no attested certifications found in primary sources**.
- First-class MCP: client + server, tool filtering per VK, per-user
  upstream OAuth and tenant isolation (Enterprise).
- Self-published perf: **11 µs overhead at 5k RPS** on a 4-vCPU node,
  59 µs on a 2-vCPU node. This is the bar.
- **Gaps**: cost-aware routing, shadow traffic, mid-stream fallback
  semantics, agent-specific traces — claimed but not documented.

### Archetype C — Optimization/experimentation platform
- Rust, Apache-2.0, single Docker image, fully OSS. Paid product is a
  separate SaaS that closes the model-optimization loop.
- Positioned as a "unified LLMOps platform" — the gateway is a thin
  data-collection front door for a flywheel of variants + feedback
  signals + optimization recipes (SFT, RLHF, distillation, prompt
  optimization, dynamic in-context learning, LLM-judge tuning).
- ~19 native providers + OpenAI-compatible passthrough. Variant
  sampling and sticky A/B across multi-step workflows.
- **Missing as a gateway**: latency-aware routing, cost-aware
  routing, circuit breakers, multi-key rotation, capability tags,
  semantic cache, virtual-key budgets, RBAC, SSO, audit log, PII
  redaction, data residency primitives.
- Self-published perf: <1 ms P99 overhead at 10k+ QPS.
- **Bottom line**: complement, not direct competitor. They optimize
  models; this project would govern traffic.

### Archetype D — Hosted SaaS aggregator
- Hosted only, no self-host. Edge-deployed on a CDN runtime. EU
  residency is a paywalled tier — no residency on the standard tier.
- 300–400+ models, 60+ providers; unified billing with no inference
  markup (small credit-purchase / BYOK fee instead).
- OpenAI + Anthropic Messages API drop-ins. Streaming. Provider load
  balancing weighted by inverse-square price; sort-by-latency and
  sort-by-price. Auto-routing and curated-provider variants.
- Per-key credit limits with daily/weekly/monthly reset; teams + orgs
  + workspaces with shared credit pool. **No platform-level RPM/TPM
  on paid tier.**
- OTel passthrough to any collector via webhook; Prometheus only
  indirectly.
- RBAC limited to Admin/Member. SSO not publicly documented. No
  public HIPAA BAA. SOC 2 trust portal exists, no public attestation.
- **Privacy red flag**: opt-in logging grants the operator broad
  commercial rights over inputs/outputs.
- Self-published perf: ~15 ms routing overhead (recently improved
  from ~25 ms).
- **Disqualified for the wedge buyer**: hosted-only with broad rights
  over content is a non-starter for regulated workloads.

### Archetype E — Governance-focused gateway
- TypeScript on Node.js; MIT gateway only. **Control plane
  (analytics, prompt management, RBAC, audit, guardrails UI) is
  proprietary closed-source.**
- 1,600+ models, 45+ providers. OpenAI + Anthropic Messages drop-ins.
- Routing config language with MongoDB-style operators
  (`$eq`, `$gt`, `$regex`, `$and`, `$or`) on metadata / request
  params; sequential fallback; weighted LB; sticky sessions.
  **Missing**: latency-aware, cost-aware, real shadow, real A/B
  (canary is just weighted LB).
- Exact-match cache on all tiers; **semantic Enterprise-only** with
  external vector-DB backends.
- USD + token budgets, RPM/hour/day + concurrent limits, org →
  workspace → model hierarchy, custom-price overrides.
- OTel-compliant tracing; **Prometheus metrics Enterprise-only**;
  log retention is tier-gated (3 days dev / 30 days Pro / custom
  Enterprise); no documented OTel exporter to common observability
  backends.
- **Compliance — the actual comparison axis**:
  - Audit log captures CRUD across workspaces, keys, configs,
    prompts, guardrails; filterable by status / workspace / user /
    IP / country; **"indefinite retention" claimed but WORM and
    immutability NOT documented; no SIEM/syslog export**.
  - PII: LLM-based "detect PII" (configurable categories but **not
    jurisdiction-specific**). Deterministic guardrails are regex /
    JSON / word-count. **No built-in detectors for PAN, Aadhaar,
    UPI, IFSC, GSTIN, Emirates ID, NRIC.** PII detectors offloaded
    to paid third-party guardrail vendors.
  - Data residency: **no documented managed EU-only or India-only
    region**. Only path is "Private Cloud / VPC" deployment, which
    is **hybrid — data plane in customer VPC, control plane stays
    in the vendor's cloud with a 1-minute outbound heartbeat**. No
    documented air-gapped mode.
  - RBAC: org + workspace, Enterprise. SSO: limited to two major
    IdPs. **SCIM not documented.**
  - Certifications: SOC 2 Type 2, ISO 27001, GDPR, HIPAA (custom
    BAA). **No PCI-DSS, no DFSA/MAS/SEBI/RBI evidence packs.**
  - Encryption: TLS 1.2+, AES-256 at rest, BYOK/KMS on Enterprise.
    HSM/CloudHSM beyond generic BYOK not documented.
- MCP: **client only** (Responses + Messages API → remote MCP
  servers). Not an MCP server. **No per-agent budgets / audit /
  guardrails scoped to MCP tool calls.**
- Latency: OSS README cites `<1 ms`; real-world with guardrails +
  governance + observability enabled is 20–40 ms.

---

## Table-stakes vs. differentiation

What every serious player already ships — must match these to be
taken seriously, no points for shipping them:

- OpenAI- and Anthropic-compatible endpoints with streaming.
- 15+ providers covered, including Bedrock, Vertex, Azure, Ollama,
  vLLM.
- Provider-error normalization, retries with backoff+jitter, fallback
  chains.
- Virtual keys with budgets and RPM/TPM limits.
- Exact-match response cache; semantic cache as configurable add-on.
- OTel + Prometheus out of the box.
- MCP gateway in some form.

What is **actually differentiated** in 2026, ranked by buyer impact:

1. **Regulated-industries compliance done right** (see wedge below) —
   no OSS option closes this. The closest player ships audit without
   WORM or SIEM, hybrid VPC that phones home, and PII detectors that
   are not jurisdiction-aware.
2. **True air-gapped / no-phone-home deploy** — the existing VPC
   offerings either claim it without proof or phone home on a 1-min
   heartbeat. Neither is provably air-gapped.
3. **Cost-aware routing** — most players lack it entirely; one has it
   only as a shuffle-strategy hint.
4. **MCP with per-agent budgets and audit** — every player has MCP
   in some form, but none scope budgets/audit to the MCP tool-call
   level.
5. **WORM audit with SIEM export** — one Go-based player ships
   HMAC-signed audit with SIEM export (still Enterprise-paywalled);
   the closest governance-focused player has audit but no WORM and
   no SIEM; others either paywall it entirely or don't ship it.
6. **Sub-100 µs OSS overhead** — the leader sits at 11–59 µs; the
   Python aggregator sits at ~8 ms. Matching the leader is plausible;
   crushing the aggregator is trivial and worth doing publicly.

---

## Sharpened wedge — what to actually build

The "compliance-first for regulated industries" framing is correct
but vague. Concrete, defensible gaps in the market today:

1. **Jurisdiction-aware PII detection out of the box** — built-in
   detectors for PAN, Aadhaar, UPI VPA, IFSC, GSTIN (India),
   Emirates ID (DFSA/UAE), NRIC (MAS/Singapore), IBAN, SWIFT, US
   SSN, PCI PAN. Not a config burden on a third-party library —
   actually shipped with regression tests. **No competitor ships
   this**; the closest one points you at third-party paid guardrails.
2. **WORM-by-default audit log** — S3 Object Lock adapter shipped;
   HMAC chained signatures; contractually-defined retention;
   SIEM/syslog export shipped, not paywalled. The closest
   competing audit log explicitly does not claim WORM or SIEM.
3. **Evidence packs mapped to specific regulator frameworks** —
   prebuilt report exports for SEBI Cybersecurity Framework, DFSA
   Module GEN, MAS TRM Guidelines, MAS FEAT, RBI IT Framework,
   HIPAA §164 controls. Generic SOC 2 mappings are not enough; this
   is the "we can show our regulator" gap.
4. **True air-gapped mode** — fully disconnected install: no
   telemetry, no license check, no config sync. Documented and
   tested. Existing "VPC" offerings phone home.
5. **Data residency as enforcement, not config** — provider routing
   rules that hard-fail rather than fall back across jurisdictions
   (`eu_only`, `in_only`, `gcc_only` modes). Default-deny on
   cross-region.
6. **MCP-scoped governance** — per-agent-run budgets, per-tool-call
   audit lines, guardrails that can intercept tool inputs/outputs
   the same way they do completions. Industry-wide gap.
7. **Closed-source-component-free posture** — entire stack OSS so
   regulator-facing code is auditable. The closest governance-focused
   competitor's control plane is closed; this matters for procurement
   at banks.

Commercial story: OSS core. Paid add-ons for HSM-backed key storage,
managed evidence-pack updates as regulations change, 24/7 support,
and signed compliance attestations.

---

## Concrete performance targets

Numbers to beat, sourced from competitors' own published benchmarks:

- **vs. the Python aggregator**: easy. ~8 ms P95 overhead at 1k RPS
  on a 4-instance cluster. Any reasonable Go implementation will
  beat this on a single node.
- **vs. the Go performance leader**: hard. 11 µs overhead at 5k RPS
  on a 4-vCPU node, 59 µs on a 2-vCPU node. Target: stay within 2×
  on equivalent hardware, accepting that compliance features cost
  microseconds. Publish on the same hardware tier.
- **vs. the governance-focused gateway**: medium. OSS layer is
  <1 ms; with guardrails + governance enabled, real-world is
  20–40 ms. Target: stay under 5 ms with the full compliance stack
  enabled (PII + audit + residency).
- **vs. the hosted SaaS aggregator**: not applicable (they're at the
  edge of a CDN). The self-host story is the answer, not a head-to-
  head ms number.

Publish a benchmark harness in the repo from day one. The leader's
numbers are credible because their harness is reproducible — match
that openness.
