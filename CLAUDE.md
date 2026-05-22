# CLAUDE.md

Project-scoped instructions for Claude when working on this codebase.
Loaded automatically by Claude Code. Keep this file as the operating
contract — `docs/features.md` is the design / roadmap document.

## Project in one paragraph

A Go-based LLM gateway platform targeting enterprises with
compliance, audit, and data-sovereignty requirements — under SOC 2,
ISO 27001, HIPAA, GDPR, or sector-specific frameworks. Proxy with
multi-tenant accounts, pure usage-based metered billing, WORM-grade
audit, jurisdiction-aware PII detection, and an **MCP-native
management interface** (no web frontend). Hosted SaaS is the primary
distribution; the same binary self-hosts with the billing module
disabled.

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
- **Management interface**: MCP (Model Context Protocol). All admin
  and customer operations are exposed as MCP tools over an
  authenticated HTTP+SSE transport served at `/v1/mcp`. **No web
  frontend, no SPA, no Vite, no React.** The only HTML the gateway
  serves is a minimal Stripe-redirect landing page at
  `/billing/return`, rendered via `html/template` (stdlib only).
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
- **License**: MIT. No CLA on contributions.

## Project guardrails

Meta-rules about how code lands in this repo. These exist to keep the
codebase auditable for the enterprise compliance buyer and stable as
we grow.

- **One logical change per commit.** Small, focused commits. If a
  commit message needs "and" twice, split it.
- **All errors must be handled.** Discarding with `_ = ` is rare and
  needs a single-line comment explaining why. Never `_ = err` after
  an IO call without justification.
- **Exported identifiers need doc comments.** Comment starts with
  the identifier name and is a complete sentence: `// Store
  persists usage events. ...`. Unexported identifiers do not need
  comments unless the *why* is non-obvious.
- **No `panic` outside `cmd/`**. Library and handler code returns
  errors. `panic` is reserved for fatal startup failures in
  `main.go`.
- **No `os.Exit` outside `cmd/`**. Same reason.
- **No `TODO` or `FIXME` without a tracking issue.** Format:
  `// TODO(#42): brief description.` A floating TODO with no issue
  is treated as tech debt that compounds invisibly.
- **No skipped tests without justification.** `t.Skip("reason and
  #issue")`. A bare `t.Skip()` is a bug.
- **Security: redact secrets and PII from logs.** Never log API
  keys, bearer tokens, customer prompt content, customer response
  content, or detected PII payloads. Log structural metadata only
  (request ID, org ID, model, token counts, status codes,
  durations).
- **Dependency hygiene**:
  - Minimize. Every new dep needs a one-line justification in the
    PR description.
  - No GPL or AGPL dependencies — incompatible with the MIT we
    ship.
  - No CGO. Keep the build pure Go so cross-compilation and
    container builds stay clean.
  - Prefer maintained, single-purpose libraries over kitchen-sink
    frameworks.
- **Run before push**: `go test ./... -race -count=1`, `go vet
  ./...`, `gofmt -l .` clean, `staticcheck ./...` clean.
- **Migrations**: never edit a merged migration file. A change is
  always a new migration. Migration files are append-only history.
- **No commented-out code in commits.** Delete it — git remembers.
- **No license headers in source files.** MIT does not require them
  and they add noise. The repo-root LICENSE is sufficient.

## Branching and PR workflow

`main` is the only long-lived branch and it is always deployable.
Never commit directly to `main`. Every change lands via a pull
request from a short-lived feature branch, even when working solo —
the discipline matters more than the audience.

### Branch naming

Format: `<type>/<short-kebab-description>`. Lowercase, kebab-case, no
underscores, no personal names. Aim for ≤ 40 characters. Include the
issue number when one exists: `feat/42-anthropic-driver`.

Allowed types:

- `feat/` — new feature or capability.
- `fix/` — bug fix.
- `chore/` — maintenance: dependency bumps, tooling, repo
  housekeeping.
- `docs/` — documentation only, no code change.
- `refactor/` — code reshape with no behavior change.
- `test/` — test additions or fixes.
- `ci/` — CI / build / release pipeline changes.
- `security/` — security-relevant changes (handle carefully; may
  warrant private discussion before opening a public PR).
- `release/v<version>` — release-prep branches (`release/v0.2.0`).

Examples:

- `feat/openai-driver`
- `fix/57-streaming-fallback`
- `chore/bump-pgx`
- `ci/govulncheck`
- `security/redact-bearer-tokens`

### Pull requests

- One logical change per PR (mirrors the per-commit rule). Small PRs
  merge fast; large PRs rot.
- PR title is the commit title that will land on `main`. Use
  imperative mood: "Add Anthropic provider driver", not "Added" or
  "Adds".
- PR body explains *why* and what was considered, not what (the diff
  shows what). Link the issue.
- CI must be green before merge. No exceptions.
- **Squash-merge** to `main` so the history is one commit per
  feature. Keeps `git log --oneline` readable and `git bisect`
  meaningful.
- Delete the feature branch after merge (the repo setting
  `delete-branch-on-merge` is on).
- Don't rebase a branch that someone else has based work on. For
  solo work this is moot; once contributors exist, treat shared
  branches as immutable.

### Main branch protection

The `main` branch is protected on GitHub. The rules:

- **No force-push.** History on `main` is append-only.
- **No deletion.**
- **Linear history required.** Merge commits are rejected; PRs
  squash-merge or rebase-merge.
- **Pull request required** before merging. Even when working solo,
  the PR is the gate for CI to run and for a moment of "do I really
  want to ship this?"
- **Status checks must pass** before merge — the `CI` workflow's
  `Go build and test` job is required once it has Go code to test.
- **Conversation resolution required** — review comments must be
  resolved before merge.
- **Admin bypass available** for genuine break-glass situations.
  Use it deliberately, not as a default.

If a future prompt asks you to push directly to `main`, surface the
conflict — branch protection should reject the push, but you should
flag it before trying.

## Conventions

### Naming conventions

Standard Go naming, plus a few project-specific rules.

**Packages**
- Short, single word, lowercase, no underscores or camelCase.
  `proxy`, not `proxy_handler` or `proxyHandler`.
- Package name matches the directory name.
- Don't repeat the package name in identifiers: `proxy.Handler`,
  not `proxy.ProxyHandler`. The caller already sees `proxy.`.

**Files**
- `snake_case.go` (`request_handler.go`, `chat_completions.go`).
- Tests live in `_test.go` files next to the code they test.
- One concept per file, but don't over-split — a 200-line file with
  three related types is fine.

**Variables and functions**
- `camelCase` unexported, `PascalCase` exported.
- Short names for short scopes (`i`, `r`, `err`, `ctx`). Descriptive
  names for longer scopes and exported APIs.
- Method receivers: 1–3 letters, consistent across every method on
  a type. If one method on `*Store` uses `s`, all do.

**Types**
- `PascalCase` exported, `camelCase` unexported.
- Single-method interfaces often use the `-er` suffix: `Reader`,
  `Closer`, `BillingProvider`.
- Multi-method interfaces describe the role: `Store`, `Cache`.
- Define interfaces in the package that *consumes* them, not the
  package that implements them. The implementing package returns a
  concrete type.

**Constants**
- `camelCase` or `PascalCase` based on visibility. **Never**
  `SCREAMING_SNAKE_CASE` — that's a C convention.
- Group related constants in `const (...)` blocks; use `iota` for
  ordered enums.

**Errors**
- Sentinel errors: `var ErrNotFound = errors.New("not found")`.
- Error types: `PascalCase` ending in `Error` (`ValidationError`).
- Wrap with context: `fmt.Errorf("get user %q: %w", id, err)`.
- Use `errors.Is` and `errors.As` at call sites. Never string-match
  error messages.

**Acronyms**
- Preserve case as a unit, don't title-case them. `HTTPServer` not
  `HttpServer`. `ID`, `URL`, `API`, `JSON`, `SQL`, `IP`.
- Lowercase the whole unit when the identifier is unexported:
  `httpServer`, `userID`, `apiKey`, `jsonPayload`.

**Booleans**
- Prefix with `is`, `has`, `should`, `can`: `isReady`,
  `hasPermission`, `shouldRetry`, `canStream`. Avoid bare nouns like
  `ready` or `streamable` that read ambiguously at call sites.

**Tests**
- `TestThing` for top-level test functions matching the
  function/type under test.
- Table-driven tests: the `name string` field is first in the case
  struct; subtests via `t.Run(tc.name, func(t *testing.T) { ... })`.
- Use `t.Parallel()` at the top of any test that doesn't touch
  shared state. Run with `-race` always.

**Project-specific**
- Storage package functions are noun-based: `store.GetUser`,
  `store.InsertUsageEvent`. No `XxxRepository` or `XxxDAO`
  patterns — that's Java-isms.
- HTTP handlers: `proxy.HandleChatCompletions`, not
  `proxy.ChatCompletionsHandler` (the function-name shape mirrors
  `net/http.HandlerFunc`).
- Migration files: `NNNN_snake_case_description.up.sql` and
  `NNNN_snake_case_description.down.sql`. Zero-padded sequence.
- DB column names: `snake_case` (`org_id`, `created_at`,
  `provider_cost_usd`).
- API JSON fields: `snake_case` to match the OpenAI convention
  (`prompt_tokens`, `completion_tokens`). Struct tags pin this:
  `` `json:"prompt_tokens"` ``.

### Go style

**Defaults**
- Standard library first. Reach for third-party packages only when
  there's a concrete reason.
- One package per directory; package name matches directory name.
- `gofmt`, `go vet`, and `staticcheck` clean before commit.

**Interfaces**
- Accept interfaces, return concrete types. Callers gain
  flexibility; implementers stay predictable.
- Keep interfaces small — 1–3 methods is the sweet spot. Large
  interfaces couple consumers to internals.
- Define interfaces where they're *used*, not where they're
  implemented. The `store` package returns `*Store`; the `proxy`
  package that consumes it defines a `proxy.userLookup` interface
  if it only needs `GetUser`.

**Errors**
- Wrap with `fmt.Errorf("context: %w", err)` whenever crossing a
  layer or adding context.
- Sentinel errors via `var ErrFoo = errors.New("foo")`. Match with
  `errors.Is`.
- Error types via struct + `Error()` method. Match with
  `errors.As`.
- Never string-compare error messages.
- Don't swallow errors. If an error is truly ignorable, comment
  why: `_ = f.Close() // best-effort; flush already succeeded`.

**Control flow**
- Early return on errors. Avoid `else` after `return`, `continue`,
  or `break` — invert the condition and exit early.
- Pointer vs value receivers: pick one per type and be consistent
  across every method. Use pointer receivers if the type contains
  a mutex, must be mutated, or is large. Use value receivers for
  small read-only types.
- No `init()` functions except for cases like driver registration
  where the language requires it. They make program order opaque.
- No global state outside `cmd/`. Pass dependencies explicitly
  through constructors.

**Generics and `any`**
- Avoid `any` / `interface{}`. If you must use it at a boundary
  (JSON decoding, third-party APIs), type-assert immediately at
  the entry point so downstream code sees a concrete type.
- Use generics when they eliminate real duplication, not as a
  reflex. A `Set[T]` is fine; a generic "Service" type is usually
  a smell.

**Logging**
- Use `slog` everywhere. Structured fields, never `fmt.Sprintf` into
  log messages.
- Standard fields: `org_id`, `request_id`, `provider`, `model`,
  `duration_ms`, `status`. Be consistent — observability dashboards
  depend on field stability.
- Levels: `Debug` for hot-path detail, `Info` for normal events,
  `Warn` for recoverable problems, `Error` for failed requests
  that need attention. Don't log at `Error` for expected user
  errors (4xx) — those are `Info` with status fields.

**Context**
- `context.Context` is the first argument of any function that
  performs IO or might block.
- Never store `context.Context` in a struct field.
- Don't use `context.Value` for required parameters. Reserve it
  for cross-cutting concerns: request ID, auth subject, OTel
  span.
- Always pass the inbound `ctx`; only derive a new one
  (`context.WithTimeout`) when you genuinely need different
  semantics.

**Tests**
- Tests live in `_test.go` files alongside the code.
- Table-driven tests for input/output coverage.
- Use `t.Parallel()` for independent tests. Use `t.Cleanup()` for
  teardown — it runs in LIFO order even if a subtest fails.
- Run `go test ./... -race -count=1` before considering anything
  done. The `-count=1` defeats the test result cache.
- Don't mock what you can use directly. The compliance buyer's
  test suite will hit a real Postgres — your tests should too,
  via `testcontainers-go` or a shared dev DB. No mocked DB
  clients.

**Misc**
- Prefer struct literals with field names: `User{Name: "x"}`, not
  `User{"x"}`. Field-order changes shouldn't break call sites.
- Always handle `Close()` errors on writers (`*os.File`,
  `io.WriteCloser`, response bodies). For best-effort cleanup, use
  a named return and a deferred wrapper.
- `defer` inside a loop accumulates until the function returns.
  If cleanup must happen per iteration, wrap the loop body in a
  function.
- Prefer `[]T` over `[]*T` for value types unless you specifically
  need shared mutation. Slices of pointers are easy to misuse.

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

### SQL practices

We are multi-tenant from day one. Tenant isolation is the load-
bearing invariant of the whole product — a cross-tenant read is a
sev-1 security incident, not a bug. SQL conventions below are
written around that.

**Multi-tenancy (non-negotiable)**
- Every tenant-owned table has an `org_id` column.
- Every query on a tenant-owned table includes `org_id` in `WHERE`.
  No exceptions.
- Composite indexes lead with `org_id` — `(org_id, created_at)`,
  `(org_id, key_id, ts)`. Don't index a sub-column alone unless it
  is a globally-unique surrogate (e.g., `id`).
- Foreign keys stay within an org. A child row's parent must be in
  the same `org_id`; enforce in app code and in constraints.
- Cross-tenant joins are forbidden in transactional queries. If you
  need cross-tenant analytics, route through the analytics tier
  (Tier 9 ClickHouse), not through Postgres directly.
- All store functions take `orgID` as an explicit argument; never
  read it from a struct field or context. The signature itself
  enforces the discipline.
- Defense in depth: add a `pgx`-level helper that wraps `Query` and
  fails loudly if `org_id` is missing from the SQL text. This is
  cheap and catches reviewer slips.
- Consider Postgres Row-Level Security policies on the most
  sensitive tables (`audit_log`, `usage_event`) as a second wall
  even if the app is correct.

**Query construction**
- Parameterized queries only. Use positional `$1, $2` or
  `pgx.NamedArgs`. Never concatenate values into SQL — that's how
  injections get shipped.
- No `SELECT *`. List columns explicitly so schema changes don't
  ripple silently into broken code or wrong-shaped struct scans.
- Always `LIMIT` queries that can return multiple rows. Unbounded
  reads turn into incidents.
- Cursor pagination (`WHERE id > $last_id ORDER BY id LIMIT n`),
  not `OFFSET`, at any scale.
- Use `INSERT ... RETURNING` to round-trip the inserted row; don't
  follow with a separate `SELECT`.

**Schema design**
- Primary keys: `bigint generated always as identity` for internal
  rows; `uuid` (via `gen_random_uuid()` from `pgcrypto`) for any ID
  that leaves the system (API keys, public org IDs) — prevents
  enumeration attacks.
- Timestamps: `timestamptz`, never `timestamp`. Store UTC.
- Money: `numeric(20,6)` for USD with microcent precision. Never
  `float` / `double precision` for monetary values.
- Strings: `text`. Don't use `varchar(N)` with arbitrary limits;
  validate length in app code.
- `NOT NULL` is the default. Allow `NULL` only when the absence
  carries a distinct meaning.
- JSONB for genuinely variable shapes (audit log details, request
  metadata blobs). Don't reach for JSONB to avoid designing a
  proper schema.

**Indexes**
- Every foreign key gets an explicit index — Postgres doesn't
  create one automatically.
- Partial indexes for sparse predicates: `CREATE INDEX ... WHERE
  status = 'active'`.
- Don't over-index — every index slows writes and inflates
  storage. Verify with `EXPLAIN ANALYZE` before adding.
- `CREATE INDEX CONCURRENTLY` in production migrations to avoid
  table locks.

**Transactions**
- Explicit `BEGIN` / `COMMIT` for any multi-statement write.
- Default isolation (`READ COMMITTED`) is fine for most paths.
- Billing-ledger writes use `SERIALIZABLE` with retry-on-
  serialization-failure — money correctness beats throughput.
- Keep transactions short. No external IO inside a transaction.
  No user input inside a transaction.
- Advisory locks (`pg_advisory_xact_lock`) for cross-instance
  coordination; not `SELECT FOR UPDATE` on hot rows.

**Migrations**
- Append-only after merge. A change is always a new migration
  file. Never edit a merged migration.
- Forward-only is OK in MVP; once we have prod data, every
  migration ships a tested `.down.sql`.
- Large-table `ALTER`s are multi-step: nullable column first, deploy
  app that handles both shapes, backfill, then drop or constrain.
  Never block production with a long `ALTER TABLE`.

**Append-only tables**
- `audit_log` and `usage_event` are append-only. No `UPDATE`, no
  `DELETE` in code. The store package exposes only `Insert` and
  `Query` for these tables; an `UPDATE` review-fails.
- `audit_log` rows include a chained HMAC over the previous row +
  the current row's content. Tampering breaks the chain.
- Partition by month using Postgres declarative partitioning. The
  cold partition drops are policy operations, not application code.

**Common pitfalls to avoid**
- Don't store enums as Postgres `ENUM` types — schema changes are
  painful. Use `text` + a `CHECK` constraint, or an integer with
  app-side mapping.
- Don't use `serial` / `bigserial` — `generated as identity` is the
  modern replacement and behaves more predictably.
- Don't trigger application logic from database triggers. Triggers
  are invisible to readers of the Go code and break local
  reproduction.

### Security practices

The compliance buyer audits this code. Treat every handler as
hostile-input territory and every log line as a potential leak.

**Input handling**
- Validate at boundaries: HTTP handlers, queue consumers, file
  readers. Trust internal callers.
- Length-limit every incoming string before storing or processing.
- Reject requests larger than a configurable max body size (10 MB
  default for chat, 1 MB for admin).
- Decode admin JSON with `DisallowUnknownFields` to catch typos
  and client drift early.
- Never accept `any` / `interface{}` shaped input without
  immediate type-assertion at the entry point.

**Output and headers**
- Set security headers globally on every response:
  `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`,
  `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`. A strict
  `Content-Security-Policy` is applied to the `/billing/return`
  Stripe-redirect page (default-src 'none' plus the minimum needed
  for that one template). No CSP needed elsewhere — the gateway
  serves no other HTML.
- Strict `Content-Type` on every response. Never let it be sniffed.
- Generic error messages to clients. Detailed error + request ID
  in the log. Return the request ID in the response so users can
  quote it in support.
- Never reflect SQL errors, stack traces, or internal paths to
  clients.

**Authentication and sessions**
- Passwords: Argon2id with sensible cost parameters (m=64MB, t=3,
  p=2). Bump as hardware improves.
- Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax`. Use
  `SameSite=Strict` for cookies that authorize destructive admin
  actions.
- CSRF tokens on every state-changing browser request. The admin
  UI is the threat model.
- API keys are prefixed with a public identifier
  (`gw_live_<random>`) so they're recognizable in logs and leaks.
  Store only the hash (SHA-256 with a peppered salt is fine for
  high-entropy random keys; Argon2id is overkill here).
- Bearer token and HMAC comparisons use
  `crypto/subtle.ConstantTimeCompare`. Never `==` on secrets.
- Aggressive rate-limit on auth endpoints: per IP, per username,
  per API key. Lockout-after-failures with backoff.

**Cryptography**
- `crypto/rand` for any token, key, nonce, or ID. **Never**
  `math/rand`.
- Use `crypto/...` and `golang.org/x/crypto/...`. Never roll your
  own crypto. Never invent a crypto protocol.
- Provider API keys encrypted at rest. Envelope encryption with a
  master key from KMS in production; AES-256-GCM with a master key
  in env var for dev.
- Audit-log HMACs use a key separate from the main encryption key.

**Secrets**
- Never commit secrets. `.env*` is gitignored.
- Configuration via environment variables or a secrets manager
  (Vault, AWS Secrets Manager). Never literals in code.
- Rotate provider API keys on a schedule. Multi-key support
  (Tier 1) is what makes rotation non-disruptive.

**Logging and telemetry (security view)**
- Log redaction is enforced at the logger level, not per-call. The
  logger refuses to print field names on a deny-list: `password`,
  `token`, `api_key`, `authorization`, `prompt`, `completion`,
  `pan`, `aadhaar`, etc.
- Log structural metadata only: request ID, org ID, model, token
  counts, status codes, durations.
- Per-org opt-in for prompt/response capture, audit-logged when
  toggled. Default-off.

**Network**
- TLS for everything, including internal service-to-service.
- Verify upstream TLS certificates — `InsecureSkipVerify` is a
  review-fail.
- Redirect HTTP → HTTPS at the edge.

**Dependencies**
- Pin versions in `go.mod`. Don't `go get` without checking.
- `govulncheck ./...` in CI; fail the build on known vulns.
- Each new dep needs a one-line justification in the PR. Prefer
  maintained, single-purpose libraries over kitchen-sink
  frameworks. No package from a one-commit-old GitHub account.
- License check on every new dep — no GPL/AGPL in the dep tree.

**Tenant isolation (defense in depth)**
- App-layer `org_id` filter (SQL practice above).
- Foreign-key constraints scoped within an org.
- Postgres RLS on `audit_log` and `usage_event`.
- Test suite includes adversarial cases: org A attempting to read
  org B's resources via every endpoint. These tests are
  non-negotiable and run in CI.

**MCP management interface**
- Bearer-token auth on every connection; tokens carry scopes that
  gate tool visibility and invocation.
- Every MCP tool invocation produces an audit log entry (Tier 5
  audit log, same WORM semantics).
- Tool inputs validated with strict JSON schemas declared by each
  tool. No `any` / `interface{}` passes through unguarded.
- Long-running tool responses stream with proper backpressure (same
  rules as proxy streaming).
- Rate-limit MCP per token, same as proxy API.

**Stripe-redirect HTML page**
- Rendered server-side via stdlib `html/template`. No inline JS, no
  external scripts, no third-party assets.
- Auto-escape on; never `template.HTML` without an explicit safety
  comment.
- Strict CSP (`default-src 'none'` + minimum allowances).

**Anti-abuse**
- Rate limits per IP, per user, per API key — already in Tier 2;
  the security stake is signup, login, and password-reset
  endpoints.
- Email verification before first paid usage (Tier 11).
- Stripe Radar for billing fraud (Tier 11).

**Concurrency hazards in security-sensitive code**
- Watch for TOCTOU patterns — check-then-use across goroutines.
- Acquire-modify-release patterns on counters use atomic ops
  (`sync/atomic`) or a single owning goroutine; never bare-read +
  bare-write.

### Project layout (target)

```
cmd/gateway/             main.go
internal/
  proxy/                 LLM wire-protocol handlers, request lifecycle
  provider/              one driver per upstream LLM provider
  mcp/                   MCP server + tool implementations
    tools/               one file per tool group (keys, billing, audit, ...)
  store/                 all DB access; nothing else imports pgx
    migrations/          golang-migrate .sql files
  billing/               BillingProvider interface + Stripe impl
  billingweb/            stdlib html/template pages for Stripe redirect flow
  auth/                  bearer tokens, scopes, MCP auth middleware
  governance/            PII, audit log, residency rules, guardrails
  routing/               fallback, load balancing, cost/latency rules
  cache/                 ristretto + Redis adapter
  config/                koanf-based config loader
  obs/                   OTel + Prometheus wiring
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
- **Do not introduce a web frontend.** No Vite, no React, no SPA,
  no Tailwind, no shadcn/ui, no embedded JS bundle. Management is
  MCP-only. Customer billing UX is via Stripe Customer Portal +
  the `/billing/return` Stripe-redirect landing page rendered by
  stdlib `html/template`. If a "small dashboard" feels tempting,
  add an MCP tool instead.
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
- `LICENSE` — MIT.

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
