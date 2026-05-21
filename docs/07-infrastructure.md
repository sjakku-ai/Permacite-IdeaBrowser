# Permacite — Infrastructure & Deployment
## Cloud Architecture, CI/CD, Security & Scaling · May 2026

---

## 1. Infrastructure Philosophy

Permacite uses a **serverless-first, managed-services** infrastructure approach. The rationale:

- **Solo/micro-team:** Zero ops overhead — no VMs to patch, no Kubernetes clusters to manage
- **Cost-proportional:** Pay per request at MVP scale; auto-scales without pre-provisioning
- **Global edge:** Browser extension calls must be low-latency globally (researchers are worldwide)
- **Compliance-ready:** Managed services (Supabase, Clerk, Stripe) handle SOC 2, GDPR, FERPA compliance for their domains

---

## 2. Environment Strategy

| Environment | Purpose | Deployment Trigger |
|-------------|---------|-------------------|
| `local` | Developer machines with Docker Compose | Manual |
| `preview` | Per-PR preview deployments | Every GitHub PR |
| `staging` | Pre-production validation, shares no data with prod | Merge to `main` |
| `production` | Live system | Tagged release (`v*`) |

### Environment Isolation

```
production    → Separate Supabase project, R2 bucket, Upstash Redis instance
staging       → Separate Supabase project, R2 bucket, Upstash Redis instance
preview       → Shared staging database (read-only seed data) + isolated Vercel preview URL
local         → Docker Compose: postgres:16 + redis:7 + localstack (S3 emulation)
```

---

## 3. Deployment Architecture

### Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Global Edge Layer                            │
│                                                                       │
│   ┌───────────────────────────────────────────────────────────────┐  │
│   │              Cloudflare (CDN + WAF + DDoS protection)         │  │
│   │         Browser Extension assets, static content, routing     │  │
│   └───────────────────────────────────────────────────────────────┘  │
│                              │                                        │
│           ┌──────────────────┼──────────────────┐                    │
│           │                  │                  │                     │
│           ▼                  ▼                  ▼                     │
│  ┌─────────────────┐ ┌──────────────┐ ┌──────────────────────────┐  │
│  │  Vercel Edge    │ │ Vercel Edge  │ │  Cloudflare Workers       │  │
│  │  Functions      │ │  Functions   │ │  (Extension API Gateway)  │  │
│  │  (Next.js SSR)  │ │  (Core API)  │ │  Rate limiting + auth     │  │
│  └─────────────────┘ └──────────────┘ └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                       Managed Services Layer                         │
│                                                                       │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────────────┐ │
│  │  Supabase    │  │ Cloudflare R2 │  │  Upstash Redis            │ │
│  │  PostgreSQL  │  │  Object Store │  │  (Streams + Cache)        │ │
│  │  + Realtime  │  │               │  │                           │ │
│  └──────────────┘  └───────────────┘  └───────────────────────────┘ │
│                                                                       │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────────────┐ │
│  │   Clerk      │  │    Stripe     │  │  Resend (Email)           │ │
│  │   Auth       │  │    Billing    │  │                           │ │
│  └──────────────┘  └───────────────┘  └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                     Background Processing Layer                       │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Vercel Cron Jobs / QStash (scheduled tasks)                  │   │
│  │  - Archive health checks (weekly)                             │   │
│  │  - Storage tier migration (nightly)                           │   │
│  │  - Usage period reconciliation (hourly)                       │   │
│  │  - Export job retry worker (every 5 minutes)                  │   │
│  └───────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Vercel Configuration

### Project Structure (Monorepo)

```
permacite/                          # pnpm workspace root
├── apps/
│   ├── web/                        # Next.js dashboard → vercel.com/permacite/web
│   └── api/                        # Fastify API → vercel.com/permacite/api
├── packages/
│   ├── domain/                     # Shared domain types (no deps)
│   ├── ui/                         # shadcn/ui components
│   └── config/                     # ESLint, TypeScript, Tailwind configs
└── extensions/
    └── browser/                    # WXT browser extension (separate build)
```

### `vercel.json` for API

```json
{
  "version": 2,
  "functions": {
    "apps/api/src/**/*.ts": {
      "maxDuration": 30
    },
    "apps/api/src/interfaces/capture.routes.ts": {
      "maxDuration": 10,
      "memory": 512
    }
  },
  "crons": [
    { "path": "/api/cron/archive-health-check", "schedule": "0 2 * * 1" },
    { "path": "/api/cron/storage-tier-migration", "schedule": "0 3 * * *" },
    { "path": "/api/cron/usage-reconciliation", "schedule": "0 * * * *" }
  ]
}
```

### Edge Runtime for Extension API Gateway (Cloudflare Workers)

The browser extension always calls `ext.permacite.com`, served by a Cloudflare Worker. This provides:
- Global latency < 50ms for the initial capture acknowledgment
- IP-based rate limiting before requests hit Vercel
- JWT verification at the edge (warm Workers avoid cold starts)

```typescript
// Cloudflare Worker: extension-gateway.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    // 1. Verify JWT at edge
    const auth = await verifyJWT(request.headers.get('Authorization'), env.CLERK_JWT_PUBLIC_KEY);
    if (!auth.valid) return new Response('Unauthorized', { status: 401 });

    // 2. Rate limit check via Cloudflare Rate Limiting API
    const rateLimit = await env.RATE_LIMITER.limit({ key: auth.researcherId });
    if (!rateLimit.success) return new Response('Rate limit exceeded', { status: 429 });

    // 3. Forward to Vercel API
    const apiUrl = new URL(request.url);
    apiUrl.hostname = 'api.permacite.vercel.app';
    return fetch(new Request(apiUrl.toString(), request));
  }
};
```

---

## 5. CI/CD Pipeline

### GitHub Actions Workflows

```yaml
# .github/workflows/ci.yml — runs on every PR
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint          # ESLint + boundary checks
      - run: pnpm type-check    # TypeScript strict mode

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
      redis:
        image: redis:7
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install --frozen-lockfile
      - run: pnpm test          # Vitest unit + integration tests
      - run: pnpm test:e2e      # Playwright extension tests (headless)

  preview-deploy:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
```

```yaml
# .github/workflows/deploy.yml — runs on tag push
name: Deploy to Production

on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Run database migrations
        run: pnpm db:migrate --env production
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
      - name: Deploy to Vercel Production
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-args: '--prod'
      - name: Build & publish extension to Chrome Web Store
        run: pnpm extension:publish
        if: startsWith(github.ref, 'refs/tags/v')
```

### Deployment Checklist (Pre-Production Gate)

Before any production deploy, GitHub Actions checks:
1. All tests pass (unit, integration, E2E)
2. No TypeScript errors (`tsc --noEmit`)
3. No module boundary violations (`eslint-plugin-boundaries`)
4. Bundle size regression check (< 10% increase without approval)
5. Database migration is backward-compatible (checked via `drizzle-kit check`)
6. Sentry source maps uploaded

---

## 6. Database Operations

### Migration Strategy

```bash
# Generate migration from schema changes
pnpm drizzle-kit generate

# Apply migrations (run before deploy, atomic per migration file)
pnpm drizzle-kit migrate

# Rollback: forward-only migrations only
# If a migration causes issues, a new compensating migration is written
```

### Connection Pooling

Supabase provides PgBouncer in transaction pooling mode. Configuration:

```
Pool mode: Transaction (not Session — Serverless functions don't hold connections)
Pool size: 15 connections per Vercel region
Supabase connection string: postgresql://...?pgbouncer=true
```

### Backup Strategy

| Backup Type | Frequency | Retention | Provided By |
|-------------|-----------|-----------|-------------|
| Continuous WAL archiving | Real-time | 7 days | Supabase (included) |
| Daily snapshot | 00:00 UTC | 30 days | Supabase Pro |
| Monthly archive | 1st of month | 1 year | Manual + S3 Glacier |
| Schema-only backup | On every migration | Unlimited | GitHub (migrations folder) |

---

## 7. Security Architecture

### Defense in Depth Layers

```
Layer 1: Cloudflare (WAF, DDoS, Bot Management)
     ↓
Layer 2: Edge Rate Limiting (Cloudflare Workers)
     ↓
Layer 3: JWT Authentication (Clerk, verified at edge)
     ↓
Layer 4: Application Authorization (RBAC via middleware)
     ↓
Layer 5: Database Row-Level Security (PostgreSQL RLS)
     ↓
Layer 6: Data Encryption (at rest via Supabase, in transit via TLS 1.3)
```

### Secrets Management

```
Production secrets stored in: Vercel Environment Variables (encrypted)
Key management for credential encryption: AWS KMS (key rotation every 90 days)
Database password: Rotated every 30 days via Supabase service roles
AI API keys: Separate keys per environment; usage budgets set at provider level
```

### Data Privacy (GDPR/FERPA Compliance)

| Requirement | Implementation |
|-------------|---------------|
| Data residency | Supabase region: US (default); EU option for institutional contracts |
| Right to erasure | Soft-delete captures + 30-day hard-delete; citations retained (public record) |
| Data portability | GET /api/v1/researcher/export — full JSON export of all citations + metadata |
| Processing records | Audit log provides article 30 GDPR processing records |
| Consent | Clerk handles consent at registration; granular consent for ORCID |
| DPA | Supabase, Clerk, Anthropic all provide DPAs (signed for institutional contracts) |

### Extension Security

```
Manifest V3 permissions (minimal):
- "activeTab"           — capture current page only (not all tabs)
- "scripting"           — inject content script for text selection
- "storage"             — store settings locally
- "tabs"                — take screenshot of current tab

Content Security Policy:
- connect-src: https://api.permacite.com https://ext.permacite.com
  http://localhost:27123 (Obsidian only)
- No eval(), no inline scripts
```

---

## 8. Observability & Monitoring

### Logging (Axiom)

All logs are structured JSON, shipped to Axiom via OpenTelemetry SDK:

```typescript
// Standard log format
logger.info('capture.pipeline.stage', {
  captureId: 'abc-123',
  researcherId: 'usr-456',
  stage: 'ARCHIVAL_STARTED',
  durationMs: 234,
  provider: 'WAYBACK_MACHINE',
  traceId: 'span.traceId'
});
```

**Log Retention:** 30 days hot (Axiom), 90 days cold (S3 archive)

### Distributed Tracing

Every request gets an OpenTelemetry trace that spans the full pipeline:

```
Trace: capture-to-citation-pipeline
├── Span: capture.api.create (200ms)
│   ├── Span: capture.storage.upload_screenshot (150ms)
│   └── Span: capture.storage.upload_html (80ms)
├── Span: archival.wayback.submit (800ms)
│   └── Span: archival.wayback.poll (2400ms, 3 polls)
└── Span: citation.ai.extract (3200ms)
    ├── Span: citation.anthropic.call (2800ms, 1024 tokens)
    ├── Span: citation.crossref.validate (180ms)
    └── Span: citation.formatter.format_all_styles (40ms)

Total: 6.8s ✓ (target: < 10s P95)
```

### Alerting Rules

| Alert | Condition | Severity | Channel |
|-------|-----------|---------|---------|
| Capture API error rate | > 1% in 5-min window | P1 | PagerDuty |
| AI extraction timeout | > 5% timeouts in 15-min | P2 | Slack |
| Wayback API failure rate | > 20% in 10-min | P2 | Slack |
| Export dead letter queue | > 10 items | P3 | Slack |
| Database connection pool | > 80% utilized | P2 | Slack |
| Storage usage | > 70% of tier limit | P3 | Email |

### Key Dashboards

1. **Pipeline Health** — Real-time capture/archive/citation rates + latency percentiles
2. **AI Performance** — Confidence score distribution, model latency, token usage, fallback rate
3. **Integration Health** — Per-connector success/failure rates, dead letter queue depth
4. **Business Metrics** — Daily active researchers, captures/day, conversion funnel
5. **Cost Dashboard** — AI API spend/day, storage growth rate, projected monthly cost

---

## 9. Scaling Thresholds & Evolution Plan

### Current Architecture Limits

| Component | Current Limit | When to Upgrade |
|-----------|--------------|----------------|
| Vercel Serverless Functions | 1,000 concurrent | 500K MAU |
| Supabase PostgreSQL | 500 connections (pgBouncer) | 200K MAU |
| Upstash Redis Streams | 10GB/month free | 50K MAU |
| Cloudflare R2 | Unlimited (pay-per-use) | N/A |
| Wayback Machine API | ~5K req/day (free tier) | 5K daily active users |

### Scale-Out Triggers

**Trigger 1: 50K MAU**
- Upgrade Upstash Redis to pay-as-you-go plan
- Add PostgreSQL read replica for citation search queries
- Negotiate Wayback Machine partner agreement (higher rate limits)

**Trigger 2: 200K MAU**
- Extract Archival Context to standalone service (highest I/O, most independent)
- Migrate event bus from Redis Streams to SQS/SNS (higher throughput, fan-out)
- Add Typesense for citation full-text search (offload from PostgreSQL)

**Trigger 3: 500K MAU**
- Extract Citation Context to standalone service (GPU-optimized instances for AI batching)
- Implement CQRS event sourcing in Citation Context (full event history for citation lineage)
- Multi-region deployment for APAC (highest growth market per market analysis)
- Consider self-hosted AI inference for cost reduction at volume

---

## 10. Disaster Recovery

### Recovery Time & Point Objectives

| Scenario | RTO | RPO | Strategy |
|----------|-----|-----|---------|
| Vercel deployment failure | 5 min | 0 | Rollback to previous deployment (1-click) |
| Supabase outage | 30 min | 5 min | Point-in-time recovery from continuous WAL backup |
| Cloudflare R2 outage | 60 min | 0 | Archives accessible via Wayback Machine primary URLs |
| Anthropic API outage | 0 | 0 | Circuit breaker → automatic GPT-4o fallback |
| Wayback Machine outage | 0 | 0 | Circuit breaker → local WACZ fallback |
| Full region outage | 4 hours | 1 hour | Restore to alternative region from backup |

### Archive Durability

The archive is Permacite's most critical data. Multi-layer durability strategy:
1. **Primary:** Wayback Machine (Internet Archive, independent infrastructure)
2. **Secondary:** Cloudflare R2 with 11 nines durability (multi-region replication)
3. **Tertiary:** Monthly snapshot to researcher's own storage (future feature: "take your archive with you")

---

## 11. Local Development Setup

```bash
# Clone and install
git clone https://github.com/permacite/permacite
cd permacite
pnpm install

# Start local services
docker-compose up -d    # PostgreSQL + Redis + LocalStack (S3)

# Run database migrations
pnpm db:migrate

# Start all apps in dev mode
pnpm dev               # Starts web + api concurrently

# Start browser extension in dev mode (loads into Chrome)
pnpm extension:dev

# Run tests
pnpm test              # Unit + integration (Vitest)
pnpm test:watch        # Watch mode
pnpm test:e2e          # Playwright (requires running app)
```

### `docker-compose.yml`

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: permacite_dev
      POSTGRES_PASSWORD: devpassword
    ports: ["5432:5432"]
    volumes: [postgres_data:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  localstack:
    image: localstack/localstack:latest
    environment:
      SERVICES: s3
      DEFAULT_REGION: us-east-1
    ports: ["4566:4566"]

volumes:
  postgres_data:
```

---

*Infrastructure decisions are revisited quarterly. The primary drivers for change are cost (AI API spend), latency (global expansion), and compliance requirements (institutional contracts in new regions).*
