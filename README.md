# 1st FP Alarm - Field Operations Platform

Internal and client-facing platform for fire alarm inspection, installation, and service operations. Built to replace Fieldwire with a fire-protection-specific tool.

## Goals

**Internal**: Eliminate duplicate data entry across field/office/accounting. Capture work once on site, flow it automatically to ServiceTrade, Sage Intacct, and BambooHR. 5,300 hours/year in estimated recovered capacity.

**Revenue lever #1**: 1,171 aged unquoted deficiencies = ~$761K repair opportunity. Deficiency-to-quote automation pushes findings directly to ServiceTrade service opportunities.

**Revenue lever #2**: Compliance and inspection portal sold to property owners at $12K-$50K/yr recurring. Target: 54%+ of revenue from inspection/service/monitoring (APi Group benchmark).

## Current Tech Stack at 1st FP

| System | Purpose |
|---|---|
| ServiceTrade | Jobs, customers, invoicing (source of truth) |
| Sage Intacct | Accounting |
| BambooHR | HR/employees |
| Microsoft 365 | Email, calendar |
| Fieldwire | Field capture (being replaced by this app) |

This app plugs into all of the above. It does not replace ServiceTrade or Sage.

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
