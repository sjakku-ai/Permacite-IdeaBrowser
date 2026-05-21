# Permacite
### Citation tool for researchers losing sources to link rot

> **Archive first. Cite second.** — One click captures, archives, and formats any web source into a permanent, verifiable citation.

---

## The Problem

Link rot is getting worse. Web citation accessibility drops from 87% for sources under 5 years old to just 38% for sources over 10 years old. If you cite 100 web sources today, roughly 20 will be broken within 5 years.

Existing tools (Zotero, Mendeley, EndNote) format citations — but do nothing to preserve the source. Permacite flips the order: **archive first, then cite.**

---

## How It Works

1. Researcher clicks the browser extension on any web page
2. Extension captures screenshot, page HTML, and URL in one click
3. Source is archived via Wayback Machine (local WACZ fallback for paywalled/JS-heavy pages)
4. AI extracts title, author, date, and any highlighted quote
5. Citation auto-formats in APA, MLA, Chicago, or BibTeX
6. Citation + permanent archive link pushed to Zotero, Notion, or Obsidian

**Target latency:** < 10 seconds from click to formatted citation.

---

## Architecture

Built as a **Modular Monolith** following Domain-Driven Design. Seven bounded contexts own their data and communicate only through domain events — no direct cross-module calls.

```
permacite/
├── apps/
│   ├── web/              # Next.js 15 dashboard (researcher portal)
│   └── extension/        # Chrome + Firefox extension (WXT framework)
├── src/
│   └── modules/
│       ├── capture/       # Receives captures from extension
│       ├── archival/      # Wayback Machine + local WACZ fallback
│       ├── citation/      # AI extraction, formatting, confidence scoring
│       ├── library/       # Research library & collections
│       ├── integration/   # Zotero, Notion, Obsidian connectors
│       ├── identity/      # Auth, workspaces, RBAC (via Clerk)
│       └── billing/       # Subscriptions, usage enforcement (via Stripe)
└── shared/               # ResearcherId, WorkspaceId, DomainEvent base types
```

**Module boundary rule:** Modules never import each other. All cross-context communication is via events on the message bus.

---

## Event Pipeline

```
[Researcher clicks extension]
         │
         ▼
   CaptureCreated ─────────────────────────┐
         │                                 │
         ▼                                 ▼
  ArchiveConfirmed                  Usage incremented
  (Wayback / WACZ)                  (Billing module)
         │
         ▼
   CitationCreated
   (AI extraction + formatting)
         │
    ┌────┴────┐
    ▼         ▼
 Library   ExportJob
  item      queued
 added    (Zotero / Notion / Obsidian)
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Browser extension | WXT + TypeScript |
| Web dashboard | Next.js 15 (App Router) + shadcn/ui |
| API | Fastify + TypeScript (Vercel serverless) |
| AI / OCR | Anthropic Claude claude-sonnet-4-6 (GPT-4o fallback) |
| Archiving | Wayback Machine Save API + WACZ local fallback |
| Database | PostgreSQL 16 (Supabase) |
| Object storage | Cloudflare R2 (screenshots, HTML, WACZ) |
| Cache / events | Upstash Redis Streams |
| Auth | Clerk (ORCID + Google SSO, workspace support) |
| Payments | Stripe |
| Observability | OpenTelemetry + Axiom + Sentry |

---

## Pricing

| Tier | Price | Captures/month |
|------|-------|---------------|
| Free | $0 | 5 |
| Individual | $15/mo | 200 |
| Pro | $30/mo | 1,000 |
| Team | $50/mo | 5,000 |
| Institutional | $5K–$50K/yr | Unlimited |

At 10,000 active users, infrastructure cost is approximately $2,065/month (~1.4% of revenue).

---

## Local Development

```bash
# Prerequisites: Node 22+, pnpm, Docker

git clone https://github.com/your-org/permacite
cd permacite && pnpm install

# Start local services (PostgreSQL + Redis)
docker-compose up -d

# Apply migrations
pnpm db:migrate

# Copy and fill environment variables
cp .env.example .env.local
# Required: ANTHROPIC_API_KEY, CLOUDFLARE_R2_*, STRIPE_*, CLERK_*, RESEND_API_KEY

# Start web + API
pnpm dev

# Start browser extension (loads into Chrome)
pnpm extension:dev
```

---

## Architecture Documents

| File | Contents |
|------|---------|
| [01-ontology.md](./01-ontology.md) | Domain model — bounded contexts, aggregates, value objects, domain events, ubiquitous language |
| [02-architecture-overview.md](./02-architecture-overview.md) | System diagram, layered architecture, context map, CQRS, key ADRs |
| [03-tech-stack.md](./03-tech-stack.md) | Technology choices with rationale, cost model, scaling thresholds |
| [04-modules.md](./04-modules.md) | Per-module structure, API endpoints, domain services, cross-module event catalogue |
| [05-data-architecture.md](./05-data-architecture.md) | PostgreSQL schema, object storage layout, Redis structures, consistency model |
| [06-integration-architecture.md](./06-integration-architecture.md) | Wayback Machine, Claude, Zotero, Notion, Obsidian, Stripe — contracts and failure handling |
| [07-infrastructure.md](./07-infrastructure.md) | Deployment, CI/CD, security, observability, disaster recovery |

---

*Permacite · May 2026 · DDD Modular Monolith · Solo/micro-SaaS*
