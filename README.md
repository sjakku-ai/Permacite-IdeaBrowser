# Permacite
### Citation tool for researchers losing sources to link rot

> **Archive first. Cite second.** — One click captures, archives, and formats any web source into a permanent, verifiable citation.

---

## The Problem

Link rot is getting worse. A 2026 study found that web citation accessibility drops from 87% for sources under 5 years old to just 38% for sources over 10 years old. If you cite 100 web sources today, roughly 20 will be broken within 5 years and 60+ will be gone within a decade.

Existing tools (Zotero, Mendeley, EndNote) format citations — but they do nothing to preserve the source. Permacite flips the order: **archive the source first, then format the citation.**

---

## What It Does

1. Researcher encounters a source → clicks the browser extension
2. The extension captures a screenshot, full-page HTML, and the URL
3. The source is archived via Wayback Machine (with local fallback for paywalled/JS-heavy pages)
4. AI extracts title, author, date, and any highlighted quote
5. Citation is auto-formatted in APA, MLA, Chicago, or BibTeX
6. Citation + archived link are pushed to Zotero, Notion, or Obsidian
7. Future readers click the citation → see the archived page, not a dead link

**Target end-to-end latency:** < 10 seconds from click to formatted citation.

---

## Architecture Overview

Permacite is a **monorepo** built as a solo/micro-SaaS. Bounded contexts from Domain-Driven Design define the module boundaries. All modules live in a single codebase deployed to Vercel — no microservices overhead.

```
permacite/
├── apps/
│   ├── web/                  # Next.js web dashboard (researcher portal)
│   └── extension/            # Browser extension (Chrome + Firefox)
├── packages/
│   ├── capture/              # Capture Context
│   ├── archive/              # Archive Context
│   ├── extraction/           # AI Extraction Context
│   ├── citation/             # Citation Context
│   ├── library/              # Library & Organisation Context
│   ├── integration/          # Integration Context
│   ├── identity/             # Identity & Subscription Context
│   └── shared/               # Shared kernel: types, event bus, DB client
├── workers/
│   ├── archive-worker/       # Async archive job processor
│   ├── extraction-worker/    # Async AI extraction job processor
│   └── integration-worker/   # Async export job processor
└── infra/
    └── supabase/             # DB migrations, RLS policies, seed data
```

---

## Domain Model (Ontology)

The core entities and how they relate:

```
Researcher ──initiates──► Capture ──triggers──► ArchiveJob ──produces──► Archive
                              │
                              └──triggers──► ExtractionJob ──produces──► SourceMetadata
                                                                               │
                                                                               ▼
                                                                           Citation
                                                                               │
                                                                    ┌──────────┴──────────┐
                                                                    ▼                     ▼
                                                              CitationRecord         Integration
                                                           (APA/MLA/BibTeX)    (Zotero/Notion/Obsidian)
```

**Key entities:**

| Entity | What it is |
|--------|-----------|
| `Source` | A normalised URL in the live web |
| `Capture` | The timestamped event of clicking capture (screenshot + HTML + URL) |
| `PageSnapshot` | The stored bytes — the proof the page existed |
| `ArchiveJob` | The task of producing a permanent archive via the fallback chain |
| `Archive` | The completed result with an immutable `permanentUrl` |
| `ExtractionJob` | The AI task of pulling structured metadata from the capture |
| `SourceMetadata` | Extracted title, authors, date, publisher, DOI |
| `Citation` | The researcher's editable scholarly reference |
| `CitationRecord` | A rendered citation string in a specific style (cached) |
| `Quote` | A highlighted passage attached to a citation |
| `Library` | A personal or team collection of citations |
| `ResearchProject` | A thematic folder inside a Library |
| `Integration` | A configured connection to Zotero, Notion, or Obsidian |
| `Subscription` | The access tier governing usage limits |

---

## Bounded Contexts

Seven DDD bounded contexts, each owning its data and communicating only via domain events.

### 1 — Capture Context
Owns the raw evidence. Handles the `InitiateCapture` command, stores screenshot + HTML to R2, and emits `CaptureInitiated` to the async queue. Knows nothing about citations, archives, or AI.

### 2 — Archive Context
Owns the "survive the web" promise. Executes the archive fallback chain:
```
Wayback Machine API → Local WACZ (Playwright) → Screenshot-only fallback
```
Emits `ArchiveCompleted` or `ArchiveFailed`. Archives are immutable once created.

### 3 — AI Extraction Context
Owns the intelligence layer. Calls Claude API (vision mode) on the screenshot + HTML, parses structured JSON, computes a confidence score (0.0–1.0). Emits `MetadataExtracted` (confidence ≥ 0.85) or `MetadataNeedsReview` (below threshold). Optionally cross-checks via CrossRef/PubMed if a DOI is detected.

### 4 — Citation Context
Owns the primary deliverable. Assembles a Citation from extracted metadata + archive link. Renders CitationRecords in APA 7, MLA 9, Chicago 17, BibTeX, and Vancouver using `citation.js`. Emits `CitationCreated` and `CitationUpdated`.

### 5 — Library & Organisation Context
Owns the researcher's knowledge structure. Manages Libraries and ResearchProjects. Holds only `citationId` references — does not own Citation content.

### 6 — Integration Context
Translates Citations into third-party data models and pushes them asynchronously. Each provider has a dedicated Translator:
- `ZoteroTranslator` → Zotero Web API v3
- `NotionTranslator` → Notion API 2022-06-28
- `ObsidianTranslator` → Markdown front matter via Local REST API plugin

### 7 — Identity & Subscription Context
Owns accounts, auth, plans, and usage enforcement. All contexts call `checkUsageLimit()` before billable actions. Subscription plans govern monthly capture allowances. Stripe webhooks keep subscription state in sync.

---

## Event Flow: One Capture, Happy Path

```
[Researcher clicks extension]
        │
        ▼
1.  CaptureInitiated ──────────────────────────────────────┐
        │                                                   │
        ▼                                                   ▼
2.  ArchiveJobQueued                               3. ExtractionJobQueued
        │                                                   │
        ▼                                                   ▼
4.  Wayback Machine API                          5. Claude API (vision)
        │                                                   │
        ▼                                                   ▼
6.  ArchiveCompleted                            7. MetadataExtracted
        │                                                   │
        └───────────────────┬───────────────────────────────┘
                            ▼
                   8. CitationCreated
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
       9. CitationFormatted    10. ExportJobQueued
          (APA/MLA/etc.)           (if auto-push on)
                                        │
                                        ▼
                                 11. Pushed to Zotero / Notion
```

Steps 2–11 are **fully asynchronous** — the extension gets its success response after step 1 and the dashboard updates as subsequent steps complete.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Browser extension | **Plasmo** + TypeScript | MV3 for Chrome & Firefox from one codebase |
| Web dashboard | **Next.js 15** (App Router) | Unified frontend + API in one deploy |
| UI components | **shadcn/ui** + Tailwind CSS | Accessible, unstyled primitives |
| Backend API | **Next.js API Routes** on Vercel | Zero-config serverless, shared types with frontend |
| Async workers | **Vercel Serverless** + **Upstash QStash** | Durable HTTP-based queue; no persistent server needed |
| Database | **Supabase Postgres** | RLS at DB level, full-text search, JSON columns |
| Auth | **Supabase Auth** | Magic link, Google OAuth, JWT — no extra service |
| File storage | **Cloudflare R2** | Zero egress fees for screenshots and HTML blobs |
| AI / OCR | **Anthropic Claude API** (vision) | ~95% metadata extraction accuracy; structured JSON output |
| Web archiving | **Wayback Machine Save API** | Free, permanent, externally verifiable |
| Citation formatting | **citation.js** | Open-source; supports 9,000+ CSL styles |
| Payments | **Stripe** | Subscriptions, invoicing, tax, customer portal |
| Email | **Resend** | Transactional email (3K/month free) |
| Monitoring | **Sentry** + **PostHog** | Errors + product analytics |

### AI Cost Model

```
Claude Sonnet 4.6 cost per capture:
  Input:  1,500 tokens × $3/1M   = $0.0045
  Output: 200 tokens   × $15/1M  = $0.003
  Total:                           ~$0.0075/capture

At $15/mo plan (200 captures/user):
  AI cost/user/month:  $1.50
  Gross margin:        ~87%
```

---

## Modules & Responsibilities

| Module | Owns | Key responsibility |
|--------|------|--------------------|
| `apps/extension` | Capture UI | One-click capture; screenshot via `captureVisibleTab`; sends payload to API |
| `apps/web` | Researcher portal | Library browser, citation editor, review queue, integration settings, billing |
| `packages/capture` | `captures`, `page_snapshots` | Validates capture, stores to R2, emits event to queue |
| `packages/archive` | `archive_jobs`, `archives` | Runs fallback chain; verifies permanentUrls on a cron |
| `packages/extraction` | `extraction_jobs`, `source_metadata` | AI extraction pipeline; confidence scoring; DOI cross-check |
| `packages/citation` | `citations`, `citation_records`, `quotes` | Assembles, formats, and manages citations |
| `packages/library` | `libraries`, `research_projects` | Organises citations into folders; bulk export |
| `packages/integration` | `integrations`, `export_jobs`, `sync_states` | Translates and pushes citations to external tools |
| `packages/identity` | `researchers`, `subscriptions`, `usage_records` | Auth, plans, usage enforcement |
| `packages/shared` | — | Shared types, event contracts, DB client, queue factory |
| `workers/archive-worker` | — | Async consumer for archive queue |
| `workers/extraction-worker` | — | Async consumer for extraction queue |
| `workers/integration-worker` | — | Async consumer for export queue |

**Module dependency rule:** No context package imports another context package. All cross-context communication is event-driven. Only `shared` is imported by all.

---

## Pricing & Subscriptions

| Tier | Price | Captures/month | Key features |
|------|-------|---------------|-------------|
| Free | $0 | 5 | Extension, basic citation preview |
| Individual | $15/mo | 200 | Full archive + AI + all integrations |
| Pro | $30/mo | 1,000 | Advanced styles, team sharing |
| Team | $50/mo | 5,000 | Shared libraries, multi-user workflows |
| Institutional | $5K–$50K/yr | Unlimited | SSO, admin dashboard, SLA |

---

## Infrastructure Cost Estimates

| Stage | Monthly infra cost | Revenue | Margin |
|-------|--------------------|---------|--------|
| Dev (0 users) | ~$5 | $0 | — |
| Early (100 users, 20 paying) | ~$143 | $300 | ~52% |
| Growth (500 paying @ $20 avg) | ~$550 | $10,000 | ~94% |

---

## Security

| Concern | Implementation |
|---------|---------------|
| Row-level security | Every Supabase table enforces `researcher_id = auth.uid()` at DB level |
| Auth tokens | Supabase JWT; 1-hour expiry + refresh |
| Credential storage | Integration API keys encrypted at rest via Supabase Vault / pgcrypto |
| File access | R2 pre-signed URLs with 15-minute expiry |
| CORS | API restricted to extension origin + web app origin |
| Prompt injection | HTML input sanitised and truncated to 50KB before AI processing |
| Confidence gate | Citations with AI confidence < 0.85 flagged for human review before use |

---

## Local Development

```bash
# Prerequisites: Node 22+, pnpm, Docker (for local Supabase)

# 1. Clone and install dependencies
git clone https://github.com/your-org/permacite
cd permacite && pnpm install

# 2. Start local Supabase
pnpm supabase start

# 3. Set environment variables
cp .env.example .env.local
# Required: ANTHROPIC_API_KEY, CLOUDFLARE_R2_*, STRIPE_*, RESEND_API_KEY

# 4. Run database migrations
pnpm supabase db push

# 5. Start the web app and API
pnpm --filter web dev

# 6. Start the extension (separate terminal)
pnpm --filter extension dev

# 7. Start async workers (simulate QStash locally)
pnpm --filter archive-worker dev
pnpm --filter extraction-worker dev
```

---

## Architecture Documents

Full detail for each architectural decision is in the following files:

| File | Contents |
|------|---------|
| [`01-domain-ontology.md`](./01-domain-ontology.md) | All domain entities, attributes, invariants, value objects, aggregates, and repository interfaces |
| [`02-ddd-bounded-contexts.md`](./02-ddd-bounded-contexts.md) | Seven bounded contexts, context map, integration patterns, async queue design, data ownership |
| [`03-module-breakdown.md`](./03-module-breakdown.md) | Each module's components, API contracts, inbound/outbound interfaces, security responsibilities |
| [`04-tech-stack.md`](./04-tech-stack.md) | Nine ADRs (Architecture Decision Records), cost models, three implementation approaches compared, risk register |

---

## MVP Timeline

**Target:** 1–2 weeks to a working demo.

**Week 1:** Extension captures screenshot + URL → backend stores to R2 → Claude extracts metadata → citation formatted and displayed in web dashboard.

**Week 2:** Wayback Machine archiving → Zotero push → Subscription + usage enforcement → Stripe billing.

**Pilot validation criteria:** End-to-end capture-to-citation in under 10 seconds for 10 thesis students, each citing a single source.

---

*Permacite · May 2026 · Solo/micro-SaaS architecture*
