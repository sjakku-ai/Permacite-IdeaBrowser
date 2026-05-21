# Permacite — Technology Stack Recommendations
## Justified Choices by Layer · May 2026

---

## 1. Stack Selection Principles

Technology choices are driven by four constraints specific to Permacite:

1. **Solo/micro-team velocity** — Technologies that have excellent documentation, strong community, and minimal boilerplate
2. **Domain boundary enforcement** — Frameworks that don't bleed into domain logic (avoid ORM-in-domain-model anti-pattern)
3. **Async-first** — Every major workflow (capture → archive → citation) is async; the stack must make this natural
4. **Cost-efficient at scale** — At $15/mo individual plans, gross margins must exceed 80%; this constrains storage and compute choices

---

## 2. Full Stack Recommendation Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Browser Extension | TypeScript + WXT Framework | Cross-browser (Chrome/Firefox) with modern build tooling |
| Frontend / Dashboard | Next.js 15 (App Router) | SSR for SEO, React ecosystem, Vercel-native deployment |
| API Gateway | Cloudflare Workers | Global edge, sub-ms latency for extension calls, built-in rate limiting |
| Backend (Core API) | Node.js + Fastify + TypeScript | Fast, low-overhead, excellent TypeScript support |
| Domain Layer | Pure TypeScript (no framework) | Framework-free domain model — the DDD golden rule |
| Message Bus | Redis Streams (MVP) → SQS/SNS (scale) | Low-latency in-process events; upgradeable |
| AI / OCR | Anthropic Claude API (primary) + OpenAI GPT-4o (fallback) | Multimodal OCR + semantic extraction; 95%+ accuracy |
| Citation Validation | CrossRef API + PubMed API | Free, authoritative academic metadata cross-check |
| Archival (Primary) | Wayback Machine Save API | Institutional trust; independent from Permacite infra |
| Archival (Fallback) | AWS S3 + WACZ format | Local capture fallback for paywalled/JS-heavy pages |
| Database (Primary) | PostgreSQL 16 (Supabase-hosted) | ACID, JSONB for flexible metadata, full-text search |
| Search | PostgreSQL full-text (MVP) → Typesense (scale) | Avoid Elasticsearch complexity at small scale |
| Object Storage | Cloudflare R2 | Zero egress cost; S3-compatible; screenshots + HTML |
| Auth | Clerk | Best-in-class DX; ORCID + Google SSO built-in; B2B workspace support |
| Payments | Stripe | Subscription metering, institutional invoicing, Stripe Tax |
| Email | Resend | Developer-first; React Email templates |
| Observability | OpenTelemetry + Axiom (logs) + Sentry (errors) | Full trace through async pipeline |
| Infrastructure | Vercel (frontend + API) + Railway (PostgreSQL managed) | Zero-ops managed hosting for MVP |
| CI/CD | GitHub Actions | Standard; integrates with Vercel preview deployments |
| Feature Flags | Growthbook (OSS) | Control confidence thresholds, rollout new AI models |

---

## 3. Browser Extension

### Technology: WXT + TypeScript + React (popup UI)

**Why WXT:**
- Cross-browser (Chrome Manifest V3 + Firefox MV2) from single codebase
- Hot Module Replacement during development
- TypeScript-first with excellent type safety for WebExtension APIs
- Built-in background service worker patterns for MV3

**Extension Architecture:**

```
extension/
├── entrypoints/
│   ├── background.ts        # Service worker: sends captures to API
│   ├── content.ts           # Page injection: text selection, highlight capture
│   └── popup/               # React popup: status, quick settings
├── lib/
│   ├── capture.ts           # Screenshot + HTML serialization logic
│   ├── api-client.ts        # Type-safe API client to Core API
│   └── auth.ts              # JWT token management
└── wxt.config.ts
```

**Capture Pipeline (In Extension):**
1. User clicks extension icon or invokes keyboard shortcut
2. Content script captures: `window.location.href`, `document.documentElement.outerHTML`, selected text
3. Background service worker takes `chrome.tabs.captureVisibleTab()` screenshot (PNG, compressed)
4. Payload sent to Core API via authenticated POST (< 5MB per capture)
5. Extension shows optimistic UI confirmation (citation arrives via WebSocket when ready)

**Key Libraries:**
- `@wxt-dev/module-react` — React in popup
- `dompurify` — Sanitize HTML before sending
- `pako` — GZIP compress HTML before transmission
- `webextension-polyfill` — Normalize Chrome/Firefox APIs

---

## 4. Frontend / Dashboard

### Technology: Next.js 15 (App Router) + TypeScript + Tailwind CSS

**Why Next.js:**
- App Router enables server components for fast initial load (important for library/search pages)
- Built-in API routes for BFF (Backend for Frontend) pattern
- Vercel-native → zero deployment configuration
- React Server Components reduce citation library bundle size significantly

**Key Libraries:**
- `@tanstack/react-query` — Server state management for citations, captures
- `@tanstack/react-table` — Virtualized library table (1000s of citations)
- `nuqs` — URL search params as state (shareable library filters)
- `zod` — Schema validation shared with backend
- `shadcn/ui` — Accessible component library (Radix primitives)
- `framer-motion` — Capture confirmation animation (delightful UX)

**Pages Structure:**
```
/                    → Landing page (marketing, static)
/dashboard           → Research Library overview
/library             → Full citation library + search
/library/[id]        → Individual citation detail + formats
/captures            → Capture history + status
/integrations        → Connector management
/settings            → Account, subscription, plan
/admin               → Institution admin dashboard (enterprise)
```

---

## 5. API Layer

### Technology: Node.js + Fastify 5 + TypeScript

**Why Fastify over Express or NestJS:**
- Fastest Node.js framework by throughput (critical for capture endpoint)
- First-class TypeScript support with type-safe schemas via TypeBox
- Plugin ecosystem is DDD-friendly (no controller-as-god-object patterns)
- NestJS adds too much magic/coupling for a domain-driven design

**Why NOT NestJS:** NestJS encourages injecting repositories into controllers, which is antithetical to DDD's application service layer pattern. Fastify's explicit plugin model matches DDD's port/adapter pattern better.

**API Structure:**
```
src/
├── interfaces/           # Fastify route handlers (Interface Layer)
│   ├── capture.routes.ts
│   ├── citation.routes.ts
│   ├── library.routes.ts
│   └── integration.routes.ts
├── application/          # Command/Query handlers (Application Layer)
│   ├── capture/
│   │   ├── commands/     # CreateCapture, RetryCapture
│   │   └── queries/      # GetCapture, ListCaptures
│   ├── citation/
│   │   ├── commands/     # ApproveCitation, RegenerateCitation
│   │   └── queries/      # GetCitation, SearchCitations, ExportBibliography
│   └── ...
├── domain/               # Pure domain — no framework imports
│   ├── capture/
│   ├── citation/
│   ├── archival/
│   └── shared/
├── infrastructure/       # DB, external APIs, message bus
│   ├── persistence/
│   ├── ai/
│   ├── archival/
│   └── messaging/
└── plugins/              # Fastify plugins: auth, rate-limit, tracing
```

**API Versioning Strategy:**
- URL versioning: `/api/v1/captures`
- Extension API is v1 and must remain stable (users don't auto-update extensions)
- Dashboard BFF API is v2 and can evolve faster

---

## 6. AI & OCR Layer

### Primary: Anthropic Claude claude-sonnet-4-6 (claude-sonnet-4-6 API)
### Fallback: OpenAI GPT-4o

**Why Claude as Primary:**
- Superior multimodal performance on structured academic document extraction
- Lower hallucination rate on citation metadata vs. GPT-4o (based on 2025 benchmarks)
- `claude-sonnet-4-6` offers best accuracy/cost ratio for this use case
- Anthropic API reliability and latency SLAs match sub-10-second pipeline target

**Why keep GPT-4o as Fallback:**
- AI provider redundancy is a must — a single provider outage breaks the core product
- GPT-4o has strong OCR on dense PDF page screenshots
- Fallback activates automatically via circuit breaker (3 consecutive failures)

**Extraction Prompt Strategy:**

```typescript
// Simplified extraction prompt structure
const extractionPrompt = `
You are extracting citation metadata from an academic web page capture.
Return ONLY a JSON object with these fields:
- title: string (exact title from page)
- authors: string[] (each author as "Last, First")
- publicationDate: string (ISO 8601 date, best estimate)
- publisher: string | null
- doi: string | null (only if visible on page)
- url: string (the original URL provided)
- pageType: "journal-article" | "book" | "website" | "report" | "other"

For EACH field, also return a confidence score 0.0-1.0.
If you are uncertain about a field, return null rather than guessing.
`;
```

**Cross-Validation Services:**
- `CrossRef API` (free) — Validates title/author/DOI for journal articles
- `PubMed API` (free) — Validates medical/life sciences citations
- `OpenLibrary API` (free) — Validates book citations by ISBN

**Cost Model:**
- Claude claude-sonnet-4-6: ~$0.003 per capture (input ~2K tokens screenshot + HTML, output ~500 tokens)
- At $15/mo with 100 captures/month → AI cost = $0.30, margin preserved at >97%

---

## 7. Archival Services

### Primary: Internet Archive Wayback Machine Save API
### Fallback: Local WACZ capture via Browsertrix Cloud (self-hosted optional) or Scoop library

**Wayback Machine Integration:**
```typescript
// Save API endpoint (free, rate-limited)
POST https://web.archive.org/save/{url}
Authorization: Bearer {ia_api_key}

// Poll for confirmation
GET https://archive.org/wayback/available?url={url}&timestamp={yyyymmdd}
```

**Local WACZ Fallback:**
- Uses `@webrecorder/warcio` npm library for in-process WARC/WACZ creation
- Captures full page with JavaScript execution via Puppeteer (headless Chrome in Lambda)
- WACZ files stored in Cloudflare R2 with content hash as key
- This handles paywalled pages (user must be authenticated when capturing) and JS-heavy SPAs

**Storage Tier Policy:**
| Age | Storage Class | Location | Cost |
|-----|--------------|----------|------|
| 0–90 days | Hot | Cloudflare R2 Standard | $0.015/GB/mo |
| 91–365 days | Warm | R2 + Infrequent Access | $0.01/GB/mo |
| 365+ days | Cold | AWS S3 Glacier | $0.004/GB/mo |
| Research delete | Soft-delete | Retain metadata, delete content | GDPR compliance |

---

## 8. Database & Storage

### Primary Database: PostgreSQL 16

**Why PostgreSQL:**
- JSONB columns for flexible CitationMetadata (fields vary by citation type)
- Full-text search via `tsvector`/`tsquery` for MVP (avoid Elasticsearch ops overhead)
- Row-level security (RLS) for multi-tenant workspace data isolation
- ACID transactions for Subscription usage tracking
- `pg_cron` for scheduled archive health checks

**Managed Hosting: Supabase (PostgreSQL)**
- Supabase provides PostgreSQL + connection pooling (pgBouncer) + realtime subscriptions
- Realtime subscriptions push `CitationCreated` events to the Next.js dashboard without polling
- Row-level security policies map directly to workspace/institution access control

**Schema Highlights:**
- `captures` — one row per capture event
- `archives` — one row per archive record, FK to capture
- `citations` — one row per citation, FK to archive, JSONB for metadata variants
- `formatted_citations` — one row per (citation, style) pair for fast retrieval
- `library_items` — many-to-many between citations and collections
- `audit_log` — append-only, partitioned by month

**Object Storage: Cloudflare R2**
- Screenshots (PNG): ~150KB average, stored with 5-year TTL minimum
- Raw HTML: ~50KB compressed, stored with 5-year TTL minimum
- WACZ fallback archives: ~500KB average
- Zero egress fees (critical — high-frequency read of screenshots for AI processing)

---

## 9. Authentication & Identity

### Technology: Clerk

**Why Clerk over Auth0 / Cognito / custom:**
- Built-in ORCID SSO (essential for academic researchers — ORCID is their professional identity)
- Google SSO included
- B2B Organizations feature maps directly to Permacite's Workspace model
- Institutional SSO (SAML/OIDC) via Clerk Organizations Enterprise — needed for university contracts
- JWT tokens with custom claims for workspaceId, planTier
- Excellent Next.js integration (middleware-based route protection)

**Auth Flow:**
```
Researcher → Clerk → JWT (contains: researcherId, workspaceId, planTier)
                           ↓
                     API Gateway validates JWT
                           ↓
                     Application extracts ResearcherId from JWT claims
                     (never calls Clerk on hot path — JWT is self-contained)
```

---

## 10. Payments & Billing

### Technology: Stripe

**Why Stripe:**
- Subscription metering for usage-based captures-per-month
- Stripe Billing Portal for self-service plan management
- Stripe Tax handles international tax compliance (EU VAT, US sales tax)
- Institutional invoicing via Stripe Invoicing (net-30 terms for library contracts)
- Webhooks map to Billing Context domain events

**Integration Points:**
- `subscription.created` → `SubscriptionCreated` domain event
- `customer.subscription.updated` → `SubscriptionUpgraded / Downgraded`
- `invoice.payment_failed` → `SubscriptionPastDue` → grace period → `CaptureLimitEnforced`

---

## 11. Message Bus

### MVP: Redis Streams (via Upstash Redis — serverless)
### Scale: AWS SQS + SNS (fan-out to multiple consumers)

**Why Redis Streams at MVP:**
- Upstash serverless Redis — zero ops, pay-per-use
- Consumer groups enable reliable at-least-once delivery
- Native message acknowledgement prevents lost events
- Max 24-hour retention is sufficient for all pipeline events

**Event Schema (CloudEvents format):**
```json
{
  "specversion": "1.0",
  "type": "permacite.capture.created",
  "source": "https://api.permacite.com/captures",
  "id": "a3f2b1c0-...",
  "time": "2026-05-22T10:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "captureId": "...",
    "researcherId": "...",
    "sourceUrl": "...",
    "captureType": "LIVE"
  }
}
```

---

## 12. Observability Stack

| Concern | Tool | Rationale |
|---------|------|-----------|
| Distributed Tracing | OpenTelemetry SDK + Axiom | Trace capture → archive → citation pipeline; Axiom has excellent OTel support |
| Error Monitoring | Sentry | Browser extension + server-side errors; source maps; session replay |
| Application Logs | Axiom (structured JSON) | Columnar log storage; fast query; reasonable cost |
| Metrics & Uptime | Better Uptime + Vercel Analytics | External uptime check; core web vitals |
| AI Decision Audit | Custom PostgreSQL audit_log table | Regulatory requirement; immutable |
| Feature Flags | GrowthBook (OSS, self-hosted) | Control confidence thresholds, AI model rollout |

**Critical Traces to Instrument:**
1. `capture.created` → `archive.confirmed` latency (target: < 3s P95)
2. `archive.confirmed` → `citation.created` latency (target: < 7s P95)
3. AI API call duration + token usage per capture
4. Export job success rate per connector type

---

## 13. Development Tooling

| Tool | Purpose |
|------|---------|
| `pnpm` workspaces | Monorepo management (extension, dashboard, API share types) |
| `turborepo` | Incremental build caching across packages |
| `zod` | Schema validation shared between API and frontend |
| `vitest` | Unit + integration testing (fast, Vite-based) |
| `playwright` | E2E testing for extension capture flow |
| `drizzle-orm` | Type-safe SQL (not an ORM that bleeds into domain — used only in infrastructure layer) |
| `eslint-plugin-boundaries` | Enforce module import boundaries (no domain importing infrastructure) |
| `changesets` | Versioning for extension releases |
| `docker-compose` | Local PostgreSQL + Redis for development |

---

## 14. Cost Model at Scale

Estimated monthly infrastructure cost at 10,000 active users, 50 captures/user/month (500K captures/month):

| Service | Usage | Cost/Month |
|---------|-------|-----------|
| Vercel Pro | 500K function invocations | ~$200 |
| Supabase Pro | 50GB database + 10M rows | ~$100 |
| Cloudflare R2 | 500K captures × 200KB avg = 100GB | ~$15 |
| Upstash Redis | 50M operations | ~$50 |
| Anthropic API | 500K captures × $0.003 | ~$1,500 |
| Wayback Machine | Free (rate-limited, acceptable at this scale) | $0 |
| Clerk | 10K users | ~$100 |
| Stripe | 2.9% + $0.30 per transaction | Variable |
| Sentry + Axiom | Combined | ~$100 |
| **Total** | | **~$2,065/month** |

Revenue at 10K users × $15/mo = $150,000/month → **Infrastructure is ~1.4% of revenue**. 

---

*Technology choices should be revisited at 50K MAU and again at 500K MAU. The primary evolution triggers are: AI cost becoming the dominant line item, PostgreSQL full-text search hitting query-time SLA limits, and Wayback Machine rate limits constraining capture throughput.*
