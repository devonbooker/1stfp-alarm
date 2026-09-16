# System Architecture

---

## Overview

Single-tenant SaaS initially (1st FP Alarm internal), multi-tenant later. Two-app monorepo: Next.js frontend + Fastify API backend. Tigris for both database and object storage. Deployed on Fly.io.

```
┌────────────────────────────────────────────────────────────────┐
│                         Fly.io                                 │
│                                                                │
│   ┌─────────────────┐        ┌──────────────────────────┐     │
│   │  Next.js (web)  │ ──────▶│  Fastify API (api)       │     │
│   │  Port 3000      │        │  Port 8080               │     │
│   │  Fly Machine    │        │  Fly Machine (2x min)    │     │
│   └─────────────────┘        └──────────┬───────────────┘     │
│                                         │                      │
│                               ┌─────────▼───────────┐         │
│                               │  Tigris              │         │
│                               │  MongoDB + S3        │         │
│                               └─────────────────────┘         │
└────────────────────────────────────────────────────────────────┘
```

---

## Services

### `apps/web` — Next.js Frontend

- App Router (Next.js 14+)
- Server components for initial page loads (SEO, performance)
- Client components for interactive elements (map/plan viewer, forms)
- React Query for API state management
- Deployed as a Fly.io machine (not Vercel — keep everything in Fly)

**Key pages:**
```
/                          → Dashboard (compliance overview)
/projects                  → Project list
/projects/[id]             → Project detail (plans, tasks, forms)
/projects/[id]/plans       → Plan viewer
/projects/[id]/tasks       → Task list + kanban
/projects/[id]/forms       → Form list
/projects/[id]/forms/new   → Start new inspection
/projects/[id]/forms/[id]  → Fill out / view form
/projects/[id]/devices     → Device inventory
/projects/[id]/panel       → Panel configuration
/settings                  → Account settings, users, AHJs, billing
/portal                    → Client portal entry point
```

### `apps/api` — Fastify API

- Fastify (Node.js) — faster than Express, better TypeScript support
- JWT auth (access token: 1hr TTL, refresh token: 90 days)
- Tigris client (`@tigrisdata/core`) for MongoDB ops
- Tigris S3 client (`@aws-sdk/client-s3`) for object storage
- Websocket server for real-time (Fastify WebSocket plugin)
- PDF generation (`puppeteer` — headless Chrome on Fly)
- Background jobs via Fly Machines (trigger a machine for heavy tasks)

**Mounted at:** `https://api.1stfpalarm.com/api/v1`

### Background Jobs

Run as separate Fly machines (short-lived, triggered via HTTP).

| Job | Trigger | What it does |
|---|---|---|
| `pdf-generator` | Form submitted | Generate PDF inspection report |
| `compliance-checker` | Daily cron | Check upcoming/overdue inspection dates, create tasks |
| `webhook-dispatcher` | Entity change | Fan out webhook payloads to subscriber URLs |
| `sheet-processor` | Sheet uploaded | Process PDF → images for plan viewer |
| `notification-sender` | Entity change | Push notifications to users |

Fly cron syntax in `fly.toml`:
```toml
[[statics]]
# compliance checker runs daily at 6am UTC
[processes]
  compliance_checker = "node jobs/compliance-checker.js"
```

---

## Authentication Flow

```
1. POST /api/v1/auth/login
   Body: { email, password }
   Response: { access_token (JWT, 1hr), refresh_token (90d) }

2. All API requests:
   Header: Authorization: Bearer <access_token>
   Header: X-Api-Version: 2024-01-01

3. Token refresh:
   POST /api/v1/auth/refresh
   Body: { refresh_token }
   Response: { access_token }

4. API key (for webhooks / integrations):
   POST /api/v1/auth/api-keys → returns long-lived key
   Header: Authorization: ApiKey <key>
```

JWT payload:
```json
{
  "sub": "user_id",
  "account_id": "account_id",
  "role": "technician",
  "exp": 1700000000
}
```

---

## Real-time Updates

WebSocket connection per authenticated session. Used for:
- Task assignment notifications
- Form submission events
- Compliance status changes

```
wss://api.1stfpalarm.com/ws?token=<access_token>

Server pushes:
{ type: "task.updated", data: { task_id, project_id, changes: {} } }
{ type: "notification.new", data: { id, title, body } }
{ type: "form.submitted", data: { form_id, project_id } }
```

Fly.io handles sticky sessions for WebSocket connections via `[http_service] stickiness = true`.

---

## File Upload Flow

### Plan upload (PDF)
```
1. Client: POST /api/v1/projects/:id/sheets/upload-url
   Response: { upload_url (presigned S3 PUT), sheet_id }

2. Client: PUT <upload_url> with PDF binary

3. Client: POST /api/v1/projects/:id/sheets/:sheet_id/process
   (triggers sheet-processor background job)

4. Processor:
   - Splits PDF pages
   - Renders pages to PNG at 2x resolution
   - Uploads PNGs to Tigris
   - Updates sheet status to "ready"
   - Sends WS event to client

5. Client renders sheet using PNG from Tigris (or via proxy endpoint)
```

### Photo/attachment upload
```
1. POST /api/v1/projects/:id/attachments/upload-url
   Response: { upload_url, attachment_id }

2. PUT <upload_url> with file binary

3. PATCH /api/v1/projects/:id/attachments/:id
   Body: { upload_status: "uploaded" }
```

---

## Plan Viewer Architecture

The plan viewer is the core UI component — equivalent to Fieldwire's sheet viewer.

**Approach:** Canvas-based renderer using Fabric.js or Konva.js.

```
Sheet Image (PNG from Tigris)
  ↓
Canvas element (Fabric.js)
  + Device pins (SVG icons, positioned by x/y %)
  + Zone boundaries (GeoJSON polygons)
  + Task bubbles (linked to tasks)
  + Markup annotations (shapes, text, arrows)
  ↑
User interactions: pan, zoom, tap pin → open task/device detail
```

**Device icon library:** SVG icons for each device type (NFPA 72 standard symbols). Stored in `apps/web/public/device-icons/`.

**Coordinate system:** All positions stored as 0.0–1.0 fractions of sheet width/height. Rendered positions = fraction × canvas pixel dimensions.

---

## PDF Report Generation

On form submission:
1. API triggers `pdf-generator` Fly machine
2. Machine launches Puppeteer
3. Renders a Next.js route `/report-template/[form_id]` (internal, not public)
4. Page pulls form data from API, renders full inspection report HTML
5. Puppeteer prints to PDF
6. PDF uploaded to Tigris (`reports/` bucket)
7. Attachment record created, form record updated with `pdf_key`
8. Notification sent to relevant users

Report templates in `apps/web/app/report-template/` — server-side rendered, print-optimized CSS.

---

## ServiceTrade Integration

ServiceTrade is the source of truth for customers and jobs. This app is the field capture layer, not the job management layer.

```
ServiceTrade                     This app
──────────────                   ──────────────────────
Customer record       ←────      Read customer info for new project
Job / Work Order      ←────      Link project to ST job
                      ────→      Push deficiency as service opportunity quote
                      ────→      Push closeout package completion (billing milestone trigger)
Asset                 ←────      Read existing assets when building device DB
```

Implementation:
- ServiceTrade REST API + API key (stored encrypted in `servicetrade_sync` collection)
- Sync runs on: job creation, deficiency created, closeout sent, manual trigger
- One-way mostly: ST → App for initial data, App → ST for deficiency quotes and billing triggers
- Error handling: failed ST pushes queue for retry, visible in sync log

## Sage Intacct Integration

Read-only for now. Billing milestone triggers via webhook from ServiceTrade (not direct Sage API).

Phase 2: Direct Sage API for project financial visibility in compliance dashboard.

## BambooHR Integration

Pull active employee list for technician assignment dropdowns. Sync on login + daily.

## Microsoft 365 Integration

Send inspection report emails via MS Graph API (uses company email domain).

---

## Multi-tenancy

Each account gets:
- Own `account_id` scoping all data
- Own Tigris object storage prefix (`{account_id}/...`)
- Own API keys
- Isolated access control (no cross-account data access)

For contractor/SaaS tier: one Fly deployment serves all accounts. Data isolation enforced at API level (every query filters by `account_id` from JWT). No per-tenant databases (MongoDB multitenancy via discriminators).

---

## Fly.io Configuration

### Machines

| Machine | Count | Size | Purpose |
|---|---|---|---|
| `web` | 2 | shared-cpu-1x / 256MB | Next.js frontend |
| `api` | 2 | shared-cpu-1x / 512MB | Fastify API |
| `pdf-generator` | 0 (on-demand) | performance-cpu-1x / 2GB | Puppeteer PDF |
| `sheet-processor` | 0 (on-demand) | performance-cpu-1x / 2GB | PDF→PNG processing |

### `fly.toml` (API)

```toml
app = "1stfpalarm-api"
primary_region = "iad"  # US East

[build]
  dockerfile = "apps/api/Dockerfile"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 1
  [http_service.concurrency]
    type = "requests"
    hard_limit = 250
    soft_limit = 200

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1
```

### Environment Variables (Fly secrets)

```
TIGRIS_URI=mongodb+srv://...
TIGRIS_ACCESS_KEY_ID=...
TIGRIS_SECRET_ACCESS_KEY=...
TIGRIS_ENDPOINT_URL=https://fly.storage.tigris.dev
TIGRIS_BUCKET_PLANS=1stfpalarm-plans
TIGRIS_BUCKET_ATTACHMENTS=1stfpalarm-attachments
TIGRIS_BUCKET_REPORTS=1stfpalarm-reports
JWT_SECRET=...
JWT_REFRESH_SECRET=...
NEXT_PUBLIC_API_URL=https://api.1stfpalarm.com
```

---

## Tigris Setup

Tigris runs natively on Fly.io. Create via:

```bash
fly storage create --name 1stfpalarm-plans
fly storage create --name 1stfpalarm-attachments
fly storage create --name 1stfpalarm-reports
```

MongoDB connection via Tigris:
```bash
# Tigris gives you a MongoDB connection string
fly tigris db create --name 1stfpalarm-db
```

---

## Monorepo Structure

```
1stfp-alarm/
├── apps/
│   ├── web/
│   │   ├── app/               # Next.js App Router
│   │   ├── components/
│   │   │   ├── plan-viewer/   # Canvas plan viewer
│   │   │   ├── forms/         # Dynamic form renderer
│   │   │   ├── tasks/
│   │   │   └── devices/
│   │   ├── public/
│   │   │   └── device-icons/  # NFPA SVG icons
│   │   ├── fly.toml
│   │   └── Dockerfile
│   └── api/
│       ├── src/
│       │   ├── routes/        # Fastify route files
│       │   ├── services/      # Business logic
│       │   ├── middleware/    # Auth, rate limit
│       │   └── jobs/          # Background job handlers
│       ├── fly.toml
│       └── Dockerfile
├── packages/
│   ├── db/
│   │   ├── schemas/           # Tigris/Mongoose schemas
│   │   └── client.ts          # Tigris client setup
│   ├── types/
│   │   └── index.ts           # Shared TS types
│   └── pdf/
│       ├── templates/         # Puppeteer report templates
│       └── generator.ts
├── infra/
│   ├── env.example
│   └── setup.sh               # Fly + Tigris provisioning
├── docs/
└── package.json               # pnpm workspace root
```
