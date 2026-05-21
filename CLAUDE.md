# CLAUDE.md

Project-scoped instructions for Claude when working on this codebase.
Loaded automatically by Claude Code. Keep this file as the operating
contract — `docs/features.md` is the design / roadmap document.

## Project in one paragraph

A Go-based LLM gateway platform targeting regulated industries
(banking, wealth, fintech under DFSA / MAS / SEBI / RBI / HIPAA).
Proxy with multi-tenant accounts, pure usage-based metered billing,
WORM-grade audit, jurisdiction-aware PII detection, and an admin
frontend. Hosted SaaS is the primary distribution; the same binary
self-hosts with the billing module disabled.

## Architectural commitments (do not regress)

Decisions already made. Treat these as load-bearing. If a future
prompt seems to ask you to change one, surface the conflict
explicitly before acting.

- **Language**: Go. No Python, no Node services in core.
- **Database**: Postgres only. No SQLite, not even for embedded
  mode. OSS adopters get a `docker-compose.yml` that bundles
  Postgres.
- **DB driver**: `pgx/v5` directly, not `database/sql` with `pgx`
  as a driver. Use `pgxpool` for the main connection pool.
- **Migrations**: `golang-migrate` with versioned `.sql` files in
  `internal/store/migrations/`.
- **No ORM.** No GORM, no ent, no query builder. Plain SQL strings
  or `.sql` files.
- **Storage code lives in `internal/store/`.** Nothing outside that
  package imports `pgx` or knows table names.
- **Functions over interfaces** in `internal/store/` until a second
  backend is real. Don't write a `Store` interface preemptively.
- **Tenant isolation by `org_id` column**, not schema-per-tenant.
  A store-layer helper rejects queries missing a tenant scope.
- **Frontend**: Vite + React 19 + TypeScript (strict), TanStack
  Query + Router, Tailwind + shadcn/ui, Recharts. Embedded into the
  Go binary via `embed.FS` and served from `/admin/`.
- **Pricing model**: pure usage-based. Prepaid credit balance for
  self-serve (with auto-top-up), postpaid NET-30 invoicing for
  enterprise. No subscription tiers, no seat fees, no plan-gated
  features. Feature entitlements are decoupled and toggled per-org.
- **Payment processor**: Stripe as the default `BillingProvider`
  interface implementation. Regional alternates (Razorpay, etc.)
  added only when market demand appears.
- **Cache**: `ristretto` in-memory single-node; Redis when
  distributed.
- **HTTP**: `net/http` + `chi`. Skip heavy frameworks.
- **Observability**: `otel-go`, `prometheus/client_golang`, `slog`.
- **License**: Business Source License 1.1, converting to Apache
  2.0 after the Change Date. CLA required on contributions.

## Conventions

### Go style

- Standard library first. Reach for third-party packages only when
  there's a concrete reason.
- One package per directory; package name matches directory name.
- Errors: wrap with `fmt.Errorf("context: %w", err)`. Sentinel
  errors via `var ErrFoo = errors.New("foo")`. Use `errors.Is` /
  `errors.As` at call sites.
- `context.Context` is the first argument of every function that
  does IO or might block.
- Use `slog` for logging. Structured fields, never `fmt.Sprintf`
  into log messages.
- Tests live in `_test.go` files alongside the code. Use table-
  driven tests where reasonable.
- Run `go test ./... -race` before considering anything done.
- `gofmt`, `go vet`, and `staticcheck` clean before commit.

### Concurrency

The user has basic Go knowledge. Be explicit and conservative:

- Use `context.WithCancel` and `context.WithTimeout` rather than
  hand-rolling cancellation.
- For fan-out, use `errgroup.Group` from
  `golang.org/x/sync/errgroup`.
- Channels should be created and closed by the same goroutine
  whenever possible. Never close a channel from the receiver side.
- All goroutines must have a documented exit path. No goroutine
  may outlive the request that spawned it without an explicit
  reason.
- Run tests with `-race`. Fail the build on races.
- Avoid `sync.Mutex` when a channel or `sync.Once` would do.

### SQL style

- Use Postgres-native features where they help: JSONB for flexible
  blobs, partial indexes, `INSERT ... ON CONFLICT (col) DO UPDATE`
  upserts, `LISTEN / NOTIFY` for cross-instance signaling.
- Migrations are append-only after merge. Never edit a merged
  migration. New change = new migration file.
- Every tenant-owned table has an `org_id` column and an index that
  includes `org_id` as the leading column.

### Project layout (target)

```
cmd/gateway/             main.go
internal/
  proxy/                 HTTP handlers, request lifecycle
  provider/              one driver per upstream LLM provider
  store/                 all DB access; nothing else imports pgx
    migrations/          golang-migrate .sql files
  billing/               BillingProvider interface + Stripe impl
  auth/                  sessions, OAuth, OIDC/SAML
  governance/            PII, audit log, residency rules, guardrails
  routing/               fallback, load balancing, cost/latency rules
  cache/                 ristretto + Redis adapter
  config/                koanf-based config loader
  obs/                   OTel + Prometheus wiring
web/                     Vite + React frontend; build output embedded
docs/                    features.md, design notes
```

## Things to NOT do

- Do not suggest or add SQLite support, even for dev.
- Do not suggest an ORM (GORM, ent, sqlc-generated code with a
  schema-aware ORM layer, etc.). `sqlc` itself is a code generator,
  not an ORM, and would be acceptable later if hand-written SQL
  becomes painful — but only after explicit discussion.
- Do not introduce subscription tiers, seat pricing, or plan-gated
  features in the billing layer.
- Do not add backwards-compatibility shims for old/removed
  features. If something is removed, remove it cleanly.
- Do not add closed-source dependencies to the core. Optional paid
  integrations are fine if isolated behind a build tag or external
  binary.
- Do not write comments that explain what the code does. Only
  explain hidden constraints or surprising decisions.
- Do not generate planning or summary documents unless asked.

## Reference

- `docs/features.md` — full tiered roadmap (Tiers 0–12),
  anonymized competitive landscape, wedge, performance targets.
  Source of truth for "what are we building." Read it before
  proposing roadmap-level changes.
- `README.md` — public-facing project description.
- `LICENSE` — BSL 1.1 terms.
- `CLA.md` — Contributor License Agreement.

## Working with the user

- User has limited / basic Go knowledge as of project inception.
  Prefer standard library over frameworks so code is idiomatic and
  readable. Explain concurrency primitives when they appear in
  suggestions. Flag goroutine / race hazards explicitly.
- For decision-framing questions ("which library?", "which
  approach?"), respond in prose with pros / cons / recommendation.
  Do not use interactive selection UI for these.
- For exploratory questions, respond in 2–3 sentences with a
  recommendation and the main tradeoff. Don't implement until the
  user agrees.
