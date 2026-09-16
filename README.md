# 1st FP Alarm - Field Operations Platform

Internal and client-facing platform for fire alarm inspection, installation, and service operations. Built to replace Fieldwire with a fire-protection-specific tool.

## Goals

**Internal**: Cut field technician admin time, standardize inspection documentation, track device-level service history.

**Revenue**: Sell as a SaaS to other fire protection contractors and as a client portal (inspection history, compliance status) to end customers.

## Tech Stack

| Layer | Tool |
|---|---|
| Frontend | Next.js (App Router) |
| Backend API | Node.js + Fastify |
| Database | Tigris (MongoDB-compatible + S3-compatible object storage) |
| Auth | JWT (access + refresh tokens) |
| Hosting | Fly.io |
| Repo | GitHub (`devonbooker/1stfp-alarm`) |
| Real-time | WebSockets (Fly.io machines) |

## Docs

- [`docs/fieldwire-audit.md`](docs/fieldwire-audit.md) — Fieldwire feature-by-feature breakdown and what we're replicating
- [`docs/architecture.md`](docs/architecture.md) — System architecture, service layout, data flow
- [`docs/data-model.md`](docs/data-model.md) — Full Tigris collection schemas
- [`docs/api-design.md`](docs/api-design.md) — REST API endpoint reference
- [`docs/fire-alarm-specifics.md`](docs/fire-alarm-specifics.md) — Fire alarm domain features (NFPA 72, device DB, AHJ, compliance)
- [`docs/deployment.md`](docs/deployment.md) — Fly.io + Tigris setup and config

## Repo Structure

```
1stfp-alarm/
├── apps/
│   ├── web/          # Next.js frontend
│   └── api/          # Fastify backend
├── packages/
│   ├── db/           # Tigris schemas + client
│   ├── pdf/          # PDF generation (inspection reports)
│   └── types/        # Shared TypeScript types
├── docs/
└── infra/            # Fly.io configs, Tigris setup scripts
```

## Running Locally

```bash
# Install
pnpm install

# Start API + web
pnpm dev
```

## Environment Variables

See `infra/env.example` for required vars (Tigris credentials, JWT secret, Fly tokens).
