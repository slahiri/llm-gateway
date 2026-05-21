# LLM Gateway

A Go-based LLM gateway platform built for regulated industries.
Proxy, governance, multi-tenant accounts, metered billing, and an
admin frontend — in a single binary.

**Status:** pre-MVP. Design phase. No code yet. The architecture and
roadmap live in [`docs/features.md`](docs/features.md).

## Why

The existing LLM gateways trade off in ways regulated buyers cannot
accept:

- The broad Python aggregator: structural perf and memory issues,
  paywalled governance, no jurisdiction-aware compliance.
- The Go performance leader: best raw latency, but enterprise audit
  / RBAC / SSO is paywalled and "designed for SOC 2" with no attested
  certifications.
- The governance-focused TypeScript player: closest on compliance,
  but hybrid VPC that phones home every 60 seconds, no WORM audit,
  no jurisdiction-specific PII detectors, closed-source control plane.
- The hosted SaaS aggregator: disqualifies anyone with data
  residency or audit requirements.

This project ships compliance-first OSS for banks, wealth managers,
and fintechs operating under DFSA, MAS, SEBI, RBI, HIPAA, or GDPR.

## Differentiators

- **Jurisdiction-aware PII detection out of the box** — PAN, Aadhaar,
  UPI VPA, IFSC, GSTIN, Emirates ID, NRIC, IBAN, SWIFT, US SSN, PCI
  PAN. Shipped with regression tests, not config burden on a
  third-party library.
- **WORM-by-default audit log** with S3 Object Lock adapter, HMAC
  chained signatures, native SIEM / syslog export. Not paywalled.
- **Evidence packs** mapped to SEBI Cybersecurity Framework, DFSA
  Module GEN, MAS TRM, MAS FEAT, RBI IT Framework, HIPAA §164.
- **True air-gapped deploy mode** — no telemetry, no license check,
  no config sync.
- **Data residency as enforcement** — `eu_only`, `in_only`,
  `gcc_only` modes with default-deny on cross-region.
- **MCP-scoped governance** — per-agent-run budgets, per-tool-call
  audit lines, guardrails on MCP tool inputs and outputs.

## Quick start

Once code lands:

```
docker compose up
```

This brings up the gateway and a bundled Postgres. Open
`http://localhost:8080/admin/` for the management UI.

For production, point `DATABASE_URL` at a managed Postgres
(RDS / Cloud SQL / Supabase).

## Architecture

- **Language**: Go.
- **Database**: Postgres (only). `pgx/v5` + `golang-migrate`.
- **Frontend**: Vite + React 19 + TypeScript, embedded into the Go
  binary via `embed.FS` and served from `/admin/`.
- **Cache**: in-memory (`ristretto`) single-node; Redis when
  distributed.
- **Compatibility**: OpenAI-compatible `/v1/chat/completions`,
  `/v1/embeddings`, `/v1/images/generations`. Anthropic-compatible
  `/v1/messages`.
- **Distribution**: single static binary. Hosted SaaS is the primary
  product; the same binary self-hosts for OSS adopters with the
  billing module disabled by config.

See [`docs/features.md`](docs/features.md) for the full tiered
roadmap (Tiers 0–12).

## License

Source-available under the Business Source License 1.1. Self-hosting
and internal use are permitted; offering the software as a hosted or
managed service to third parties is not. Converts to Apache 2.0 four
years after each release. See [`LICENSE`](LICENSE).

## Contributing

Contributions welcome via pull request. A Contributor License
Agreement is required — see [`CLA.md`](CLA.md). Issues and design
discussion are open.
