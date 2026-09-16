# Deployment

Fly.io for compute. Tigris for database + object storage. Both in the same Fly.io ecosystem.

---

## Prerequisites

```bash
# Install Fly CLI
curl -L https://fly.io/install.sh | sh

# Login
fly auth login
```

---

## Tigris Setup

Tigris runs as a Fly.io extension. All setup via `fly` CLI.

### Object Storage Buckets

```bash
# Create storage buckets
fly storage create --name 1stfpalarm-plans
fly storage create --name 1stfpalarm-attachments
fly storage create --name 1stfpalarm-reports
fly storage create --name 1stfpalarm-exports
```

Each command outputs:
```
TIGRIS_ACCESS_KEY_ID
TIGRIS_SECRET_ACCESS_KEY
TIGRIS_ENDPOINT_URL=https://fly.storage.tigris.dev
TIGRIS_BUCKET_NAME
```

Save these — you'll set them as Fly secrets.

### MongoDB-compatible Database

```bash
fly tigris db create --name 1stfpalarm-db
# Outputs: TIGRIS_URI (MongoDB connection string)
```

---

## Fly App Setup

### Create the apps

```bash
# API app
cd apps/api
fly launch --name 1stfpalarm-api --region iad --no-deploy

# Web app
cd apps/web
fly launch --name 1stfpalarm-web --region iad --no-deploy
```

### Set secrets (API app)

```bash
cd apps/api

fly secrets set \
  TIGRIS_URI="mongodb+srv://..." \
  TIGRIS_ACCESS_KEY_ID="..." \
  TIGRIS_SECRET_ACCESS_KEY="..." \
  TIGRIS_ENDPOINT_URL="https://fly.storage.tigris.dev" \
  TIGRIS_BUCKET_PLANS="1stfpalarm-plans" \
  TIGRIS_BUCKET_ATTACHMENTS="1stfpalarm-attachments" \
  TIGRIS_BUCKET_REPORTS="1stfpalarm-reports" \
  TIGRIS_BUCKET_EXPORTS="1stfpalarm-exports" \
  JWT_SECRET="$(openssl rand -hex 32)" \
  JWT_REFRESH_SECRET="$(openssl rand -hex 32)" \
  NODE_ENV="production"
```

### Set secrets (web app)

```bash
cd apps/web

fly secrets set \
  NEXT_PUBLIC_API_URL="https://1stfpalarm-api.fly.dev" \
  NEXTAUTH_SECRET="$(openssl rand -hex 32)"
```

---

## `fly.toml` — API

```toml
app = "1stfpalarm-api"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[env]
  PORT = "8080"
  NODE_ENV = "production"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
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

# WebSocket support
[[services]]
  protocol = "tcp"
  internal_port = 8080
  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]
  [[services.ports]]
    port = 80
    handlers = ["http"]
```

---

## `fly.toml` — Web

```toml
app = "1stfpalarm-web"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[env]
  PORT = "3000"
  NODE_ENV = "production"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 1
  [http_service.concurrency]
    type = "requests"
    hard_limit = 200
    soft_limit = 150

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1
```

---

## Dockerfiles

### API (`apps/api/Dockerfile`)

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
RUN npm install -g pnpm

FROM base AS deps
COPY package.json pnpm-lock.yaml ./
COPY apps/api/package.json ./apps/api/
COPY packages/db/package.json ./packages/db/
COPY packages/types/package.json ./packages/types/
RUN pnpm install --frozen-lockfile

FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm --filter api build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/apps/api/dist ./dist
COPY --from=build /app/node_modules ./node_modules
EXPOSE 8080
CMD ["node", "dist/index.js"]
```

### Web (`apps/web/Dockerfile`)

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
RUN npm install -g pnpm

FROM base AS deps
COPY package.json pnpm-lock.yaml ./
COPY apps/web/package.json ./apps/web/
RUN pnpm install --frozen-lockfile

FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm --filter web build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/apps/web/.next/standalone ./
COPY --from=build /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=build /app/apps/web/public ./apps/web/public
EXPOSE 3000
CMD ["node", "apps/web/server.js"]
```

---

## Deploy

```bash
# Deploy API
cd apps/api
fly deploy

# Deploy web
cd apps/web
fly deploy
```

---

## Custom Domains

```bash
# API
fly certs create api.1stfpalarm.com --app 1stfpalarm-api

# Web
fly certs create app.1stfpalarm.com --app 1stfpalarm-web

# Portal (separate subdomain for client-facing)
fly certs create portal.1stfpalarm.com --app 1stfpalarm-web
```

Add CNAME records in your DNS pointing to `1stfpalarm-api.fly.dev` / `1stfpalarm-web.fly.dev`.

---

## PDF Generator Machine

PDF generation uses Puppeteer (headless Chrome). Needs more memory — run as an on-demand machine.

```bash
# Create a separate app for PDF generation
fly launch --name 1stfpalarm-pdf --region iad --no-deploy

fly secrets set \
  TIGRIS_ACCESS_KEY_ID="..." \
  TIGRIS_SECRET_ACCESS_KEY="..." \
  TIGRIS_ENDPOINT_URL="https://fly.storage.tigris.dev" \
  TIGRIS_BUCKET_REPORTS="1stfpalarm-reports" \
  INTERNAL_API_URL="http://1stfpalarm-api.internal:8080" \
  --app 1stfpalarm-pdf
```

`fly.toml` for PDF machine:
```toml
app = "1stfpalarm-pdf"

[http_service]
  internal_port = 8090
  auto_stop_machines = "suspend"  # suspend (not stop) to avoid cold starts
  auto_start_machines = true
  min_machines_running = 0        # scale to zero when idle

[[vm]]
  memory = "2gb"
  cpu_kind = "performance"
  cpus = 2
```

---

## Compliance Cron Job

Daily job checks upcoming and overdue inspections.

Option 1: Fly scheduled machines (Fly cron)

```bash
# Create one-off machine that runs daily
fly machine run \
  --app 1stfpalarm-api \
  --schedule daily \
  --region iad \
  node dist/jobs/compliance-checker.js
```

Option 2: External cron (cron-job.org) hitting internal endpoint

```
GET https://1stfpalarm-api.fly.dev/internal/run-compliance-check
Header: X-Internal-Secret: <secret>
Schedule: 0 6 * * *  (6am UTC daily)
```

---

## Monitoring

```bash
# View logs
fly logs --app 1stfpalarm-api

# Status
fly status --app 1stfpalarm-api
fly status --app 1stfpalarm-web

# Scale up
fly scale count 3 --app 1stfpalarm-api

# Metrics
fly dashboard  # opens Fly metrics dashboard in browser
```

---

## Local Development

```bash
# Install deps
pnpm install

# Set up local env
cp infra/env.example .env.local

# Start everything
pnpm dev
# API: http://localhost:8080
# Web: http://localhost:3000
```

For local Tigris: use a free Tigris account at https://fly.io/docs/tigris/ and set the same env vars locally.

---

## Cost Estimate (Initial)

| Service | Config | ~Monthly Cost |
|---|---|---|
| Fly API machine (2x) | shared-cpu-1x, 512MB | ~$10/mo |
| Fly web machine (2x) | shared-cpu-1x, 512MB | ~$10/mo |
| Tigris object storage | 50GB | ~$2.50/mo |
| Tigris MongoDB | 1GB | ~$0 (included in Fly) |
| **Total** | | **~$22/mo** |

Scales linearly. PDF machine only runs when generating reports (fractions of a cent per report).
