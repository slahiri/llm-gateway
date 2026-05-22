# LLM Gateway

[![CI](https://github.com/slahiri/llm-gateway/actions/workflows/ci.yml/badge.svg)](https://github.com/slahiri/llm-gateway/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Go](https://img.shields.io/badge/go-1.23%2B-00ADD8?logo=go)](https://go.dev/)
[![Postgres](https://img.shields.io/badge/postgres-15%2B-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Status](https://img.shields.io/badge/status-pre--MVP-orange)](docs/features.md)

A Go-based LLM gateway platform built for enterprises with
compliance, audit, and data-sovereignty requirements. Proxy,
governance, multi-tenant accounts, metered billing, and an admin
frontend — in a single binary.

**Status:** pre-MVP. Design phase. No code yet. The architecture and
roadmap live in [`docs/features.md`](docs/features.md).

## Why

The existing LLM gateways trade off in ways enterprise buyers
cannot accept:

- The broad Python aggregator: structural perf and memory issues,
  paywalled governance, no jurisdiction-aware compliance.
- The Go performance leader: best raw latency, but enterprise audit
  / RBAC / SSO is paywalled and "designed for SOC 2" with no
  attested certifications.
- The governance-focused TypeScript player: closest on compliance,
  but hybrid VPC that phones home every 60 seconds, no WORM audit,
  no jurisdiction-specific PII detectors, closed-source control
  plane.
- The hosted SaaS aggregator: disqualifies anyone with data
  residency or audit requirements.

This project ships compliance-first OSS for any enterprise that
needs auditability, data sovereignty, and tenant isolation under
SOC 2, ISO 27001, HIPAA, GDPR, or sector-specific frameworks.

## Differentiators

- **Jurisdiction-aware PII detection out of the box** — PAN, Aadhaar,
  UPI VPA, IFSC, GSTIN, Emirates ID, NRIC, IBAN, SWIFT, US SSN, PCI
  PAN. Shipped with regression tests, not config burden on a
  third-party library. Designed for enterprises operating across
  multiple jurisdictions.
- **WORM-by-default audit log** with S3 Object Lock adapter, HMAC
  chained signatures, native SIEM / syslog export. Not paywalled.
- **Evidence packs** mapped to SOC 2, ISO 27001, HIPAA §164, GDPR,
  and sector-specific frameworks (SEBI Cybersecurity, DFSA Module
  GEN, MAS TRM / FEAT, RBI IT Framework).
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

MIT. See [`LICENSE`](LICENSE).

## Contributing

Contributions welcome via pull request. Issues and design discussion
are open.
