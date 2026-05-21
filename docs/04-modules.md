# Permacite — Module Definitions & Bounded Context Detail
## Per-Context Responsibilities, APIs, and Domain Services · May 2026

---

## Module Design Principles

Each module maps 1:1 to a bounded context from the ontology. Modules enforce strict boundaries:

- **No direct function calls between modules** — all cross-module communication via domain events on the message bus
- **Each module owns its own database tables** — no shared tables between modules (enforced via schema prefixes)
- **Each module has its own Repository interfaces** — infrastructure implementations are injected at startup
- **Shared Kernel is minimal** — only `ResearcherId`, `WorkspaceId`, and core error types are shared

```
src/
├── modules/
│   ├── capture/          # Capture Context
│   ├── archival/         # Archival Context
│   ├── citation/         # Citation Context
│   ├── library/          # Library Context
│   ├── integration/      # Integration Context
│   ├── identity/         # Identity & Access Context
│   └── billing/          # Billing & Subscription Context
├── shared/               # Shared Kernel only
│   ├── domain/
│   │   ├── ResearcherId.ts
│   │   ├── WorkspaceId.ts
│   │   └── DomainEvent.ts
│   └── infrastructure/
│       ├── MessageBus.ts
│       └── EventStore.ts
└── app.ts                # Fastify application bootstrap
```

---

## Module 1: Capture Module

### Responsibility
The Capture Module is the **entry point** for all researcher-initiated captures. It handles the browser extension API, manages the capture lifecycle, and publishes events that trigger the rest of the pipeline.

### Directory Structure
```
modules/capture/
├── domain/
│   ├── Capture.ts              # Aggregate root
│   ├── CaptureStatus.ts        # Enum: PENDING | ARCHIVING | ARCHIVED | FAILED
│   ├── CaptureType.ts          # Enum: LIVE | PAYWALLED | JS_HEAVY | LOCAL_FALLBACK
│   ├── URL.ts                  # Value object
│   ├── HighlightedQuote.ts     # Value object
│   ├── BinaryRef.ts            # Value object (S3 key + hash)
│   ├── events/
│   │   ├── CaptureCreated.ts
│   │   ├── CaptureFailed.ts
│   │   └── CaptureRetryRequested.ts
│   └── repositories/
│       └── ICaptureRepository.ts   # Interface only
├── application/
│   ├── commands/
│   │   ├── CreateCaptureCommand.ts
│   │   ├── CreateCaptureHandler.ts
│   │   └── RetryCapturCommand.ts
│   └── queries/
│       ├── GetCaptureQuery.ts
│       └── ListCapturesQuery.ts
├── infrastructure/
│   ├── PostgresCaptureRepository.ts
│   ├── S3CaptureStorage.ts
│   └── CaptureEventPublisher.ts
└── interfaces/
    ├── CaptureRoutes.ts            # POST /v1/captures, GET /v1/captures/:id
    └── CaptureSchema.ts            # TypeBox schemas for request validation
```

### API Endpoints

```
POST   /api/v1/captures
  Body: { url, screenshotBase64, rawHtmlGzipped, highlightedText?, extensionVersion }
  Response: { captureId, status: "PENDING" }
  Auth: Bearer JWT (researcherId extracted from claims)
  Rate limit: 10 requests/minute per researcher (Free), 100/minute (Paid)

GET    /api/v1/captures/:captureId
  Response: { captureId, status, captureType, createdAt, citationId? }

GET    /api/v1/captures
  Query: { page, limit, status?, since? }
  Response: { items: Capture[], total, page }

POST   /api/v1/captures/:captureId/retry
  Response: { captureId, status: "PENDING" }
```

### Domain Service: `CapturePolicyService`
- Validates researcher has not exceeded monthly capture quota (delegates to Billing module via anti-corruption layer)
- Determines `CaptureType` from URL pattern (paywall detection heuristics)
- Generates `BinaryRef` for storage keys

### Key Business Rules
1. Captures are **immutable** once created — no update endpoints except for status transitions
2. Screenshots must be < 5MB after compression
3. HTML is stored compressed (GZIP) — original encoding preserved
4. A researcher may not create new captures if plan limit is exceeded (returns HTTP 402)

---

## Module 2: Archival Module

### Responsibility
Listens for `CaptureCreated` events and orchestrates permanent storage of captured content. Manages Wayback Machine submission, polling, and local WACZ fallback. Ensures every capture has a permanent reference before citation can proceed.

### Directory Structure
```
modules/archival/
├── domain/
│   ├── Archive.ts                   # Aggregate root
│   ├── ArchiveStatus.ts             # Enum: PENDING | IN_PROGRESS | CONFIRMED | FAILED | DEGRADED
│   ├── ArchiveProvider.ts           # Enum: WAYBACK_MACHINE | LOCAL_WACZ | HYBRID
│   ├── StorageClass.ts              # Enum: HOT | WARM | COLD
│   ├── ArchiveReference.ts          # Value object: permanent URL + provider
│   ├── ContentHash.ts               # Value object: SHA-256
│   ├── events/
│   │   ├── ArchiveConfirmed.ts
│   │   ├── ArchiveDegraded.ts
│   │   ├── ArchiveFailed.ts
│   │   └── StorageTierMoved.ts
│   ├── services/
│   │   ├── StorageTierPolicy.ts     # Domain service: when to move tiers
│   │   └── ArchiveIntegrityChecker.ts # Periodic re-verification of archive URLs
│   └── repositories/
│       └── IArchiveRepository.ts
├── application/
│   ├── handlers/
│   │   └── OnCaptureCreated.ts      # Event handler: triggers archiving
│   ├── commands/
│   │   └── ArchiveCaptureCommand.ts
│   └── queries/
│       └── GetArchiveQuery.ts
├── infrastructure/
│   ├── WaybackMachineClient.ts      # Wayback Save API wrapper
│   ├── LocalWACZCapture.ts          # Puppeteer-based WACZ creation
│   ├── R2ArchiveStorage.ts          # Cloudflare R2 adapter
│   └── PostgresArchiveRepository.ts
└── interfaces/
    └── ArchiveRoutes.ts             # GET /v1/archives/:captureId
```

### Event Handling

```typescript
// OnCaptureCreated Handler
class OnCaptureCreated {
  async handle(event: CaptureCreated): Promise<void> {
    const archive = Archive.create({ captureId: event.captureId });

    // 1. Attempt Wayback Machine (with 5s timeout)
    const waybackResult = await this.waybackClient.submit(event.sourceUrl);

    if (waybackResult.success) {
      archive.confirmWithWayback(waybackResult.permanentUrl);
      await this.archiveRepo.save(archive);
      await this.eventBus.publish(new ArchiveConfirmed({ ... }));
    } else {
      // 2. Fall back to local WACZ
      const waczResult = await this.localWACZ.capture(event.sourceUrl, event.screenshotRef, event.htmlRef);
      archive.confirmWithLocalWACZ(waczResult.storageKey);
      await this.archiveRepo.save(archive);
      await this.eventBus.publish(new ArchiveDegraded({ ... })); // Still usable, but degraded
    }
  }
}
```

### Domain Service: `StorageTierPolicy`
- Evaluates archive age and last-access timestamp
- Emits `StorageTierMoved` when archive should transition: HOT → WARM (90 days), WARM → COLD (1 year)
- Triggered by scheduled task (daily `pg_cron` job)

### Archive Health Check Service
- Weekly job re-validates all HOT and WARM Wayback Machine URLs
- If URL returns 404/5xx: attempts re-archiving or elevates to `DEGRADED` status
- Notifies researcher if their cited source's archive degrades

---

## Module 3: Citation Module

### Responsibility
The **core intelligence module**. Listens for `ArchiveConfirmed`, calls the AI API for metadata extraction, validates against academic databases, formats citations in all styles, assigns confidence scores, and manages human review workflow.

### Directory Structure
```
modules/citation/
├── domain/
│   ├── Citation.ts                   # Aggregate root
│   ├── CitationStatus.ts             # Enum: EXTRACTING | FORMATTING | REVIEW_PENDING | AUTO_APPROVED | REVIEWED
│   ├── CitationMetadata.ts           # Value object (title, authors, date, doi, etc.)
│   ├── CitationStyle.ts              # Enum: APA_7 | MLA_9 | CHICAGO_17 | BIBTEX | BLUEBOOK
│   ├── FormattedCitation.ts          # Value object: rendered string per style
│   ├── ConfidenceScore.ts            # Value object: per-field scores + overall
│   ├── AuditEntry.ts                 # Value object: append-only AI decision record
│   ├── HighlightedQuote.ts           # Value object (imported from shared)
│   ├── events/
│   │   ├── CitationCreated.ts
│   │   ├── CitationReviewRequested.ts
│   │   ├── CitationApproved.ts
│   │   └── CitationExportReady.ts
│   ├── services/
│   │   ├── MetadataExtractionService.ts   # AI API calls
│   │   ├── CrossRefValidationService.ts   # CrossRef API validation
│   │   ├── PubMedValidationService.ts     # PubMed validation
│   │   ├── CitationFormatterService.ts    # Style-specific formatting
│   │   └── ConfidenceScoringService.ts    # Aggregates per-field scores
│   └── repositories/
│       └── ICitationRepository.ts
├── application/
│   ├── handlers/
│   │   └── OnArchiveConfirmed.ts         # Main event handler
│   ├── commands/
│   │   ├── ApproveCitationCommand.ts
│   │   ├── RejectCitationCommand.ts
│   │   └── RegenerateCitationCommand.ts
│   └── queries/
│       ├── GetCitationQuery.ts
│       ├── SearchCitationsQuery.ts        # Full-text search
│       └── ExportBibliographyQuery.ts     # Bulk export
├── infrastructure/
│   ├── AnthropicExtractionClient.ts      # Claude API adapter
│   ├── OpenAIExtractionClient.ts         # GPT-4o fallback
│   ├── CrossRefApiClient.ts
│   ├── PubMedApiClient.ts
│   └── PostgresCitationRepository.ts
└── interfaces/
    ├── CitationRoutes.ts
    └── CitationSchema.ts
```

### Citation Extraction Pipeline (Detail)

```
OnArchiveConfirmed handler:
  1. Load Capture (screenshot + HTML from R2)
  2. Call MetadataExtractionService
     a. Prepare multimodal prompt (screenshot + HTML + URL context)
     b. Call Claude API (primary)
     c. If Claude fails → Circuit breaker opens → Call GPT-4o
     d. Parse structured JSON response
     e. Record model, version, raw response in AuditEntry
  3. Call ConfidenceScoringService
     a. Per-field confidence from AI response
     b. CrossRef validation (if DOI present): boosts score
     c. PubMed validation (if PMID present): boosts score
     d. Overall score = weighted average of field scores
  4. Create Citation aggregate with metadata + confidence
  5. Call CitationFormatterService for all styles
     a. APA 7th: "[Author, A.] ([Year]). [Title]. [Publisher]. [PermanentLink]"
     b. MLA 9th, Chicago 17th, BibTeX, Bluebook variants
  6. If overallConfidence >= 0.85:
     → status = AUTO_APPROVED
     → publish CitationCreated + CitationExportReady
  7. If overallConfidence < 0.85:
     → status = REVIEW_PENDING
     → publish CitationCreated + CitationReviewRequested
     → NOT published to Integration module until researcher approves
```

### API Endpoints

```
GET    /api/v1/citations/:citationId
  Response: { citation, metadata, formattedVariants, confidenceScore, permanentLink }

GET    /api/v1/citations
  Query: { q (full-text), style, collectionId, since, page, limit }
  Response: { items, total, page }

PATCH  /api/v1/citations/:citationId/approve
  Body: { correctedMetadata? }   # Optional metadata correction by researcher
  Response: { citationId, status: "REVIEWED" }

POST   /api/v1/citations/:citationId/regenerate
  Response: { citationId, status: "EXTRACTING" }  # Re-triggers AI pipeline

GET    /api/v1/citations/export
  Query: { style: "bibtex"|"ris"|"csv", collectionId?, ids[]? }
  Response: text/plain (BibTeX) | text/ris | text/csv
```

### CQRS Read Model
The citation search uses a denormalized read model in PostgreSQL:
```sql
-- citations_search_view (materialized, refreshed on CitationCreated/Approved)
CREATE MATERIALIZED VIEW citations_search AS
SELECT
  c.citation_id,
  c.researcher_id,
  c.created_at,
  c.overall_confidence,
  c.status,
  cm.title,
  cm.authors,
  cm.publication_date,
  cm.doi,
  fc_apa.formatted_text AS apa_formatted,
  to_tsvector('english', cm.title || ' ' || array_to_string(cm.authors, ' ')) AS search_vector
FROM citations c
JOIN citation_metadata cm USING (citation_id)
LEFT JOIN formatted_citations fc_apa ON fc_apa.citation_id = c.citation_id AND fc_apa.style = 'APA_7';
```

---

## Module 4: Library Module

### Responsibility
Manages a researcher's organized collection of citations. Handles library CRUD, collection management, tagging, bulk export, and workspace sharing. This is the primary persistence layer researchers interact with day-to-day.

### Directory Structure
```
modules/library/
├── domain/
│   ├── ResearchLibrary.ts        # Aggregate root
│   ├── Collection.ts             # Entity
│   ├── LibraryItem.ts            # Aggregate root
│   ├── Tag.ts                    # Value object
│   ├── ResearcherNote.ts         # Value object
│   ├── events/
│   │   ├── LibraryItemAdded.ts
│   │   ├── CollectionCreated.ts
│   │   ├── LibraryShared.ts
│   │   └── BulkExportRequested.ts
│   └── repositories/
│       ├── ILibraryRepository.ts
│       └── ILibraryItemRepository.ts
├── application/
│   ├── handlers/
│   │   └── OnCitationCreated.ts      # Auto-add to default collection
│   ├── commands/
│   │   ├── AddToLibraryCommand.ts
│   │   ├── MoveToCollectionCommand.ts
│   │   ├── TagLibraryItemCommand.ts
│   │   └── ShareLibraryCommand.ts
│   └── queries/
│       ├── GetLibraryQuery.ts
│       └── GetCollectionItemsQuery.ts
└── interfaces/
    └── LibraryRoutes.ts
```

### API Endpoints

```
GET    /api/v1/library
  Response: { library, collections[], recentItems[] }

POST   /api/v1/library/collections
  Body: { name, description? }

GET    /api/v1/library/collections/:collectionId
  Query: { page, limit, sort, tags[] }

POST   /api/v1/library/items/:itemId/tags
  Body: { tags: string[] }

POST   /api/v1/library/items/:itemId/notes
  Body: { content: string }

GET    /api/v1/library/export
  Query: { format: "bibtex"|"ris", collectionId? }
  → delegates to Citation Module's ExportBibliographyQuery
```

---

## Module 5: Integration Module

### Responsibility
Manages all outbound connections to external research tools. Handles connector registration, OAuth token management, export job queuing with retry logic, and dead-letter handling.

### Directory Structure
```
modules/integration/
├── domain/
│   ├── IntegrationConnector.ts       # Aggregate root
│   ├── ExportJob.ts                  # Aggregate root
│   ├── ConnectorType.ts              # Enum: ZOTERO | NOTION | OBSIDIAN | MENDELEY
│   ├── ExportStatus.ts               # Enum: QUEUED | IN_PROGRESS | SUCCEEDED | FAILED | DEAD_LETTERED
│   ├── EncryptedCredential.ts        # Value object: encrypted OAuth tokens
│   ├── ExportPayload.ts              # Value object: connector-specific export data
│   ├── events/
│   │   ├── ConnectorRegistered.ts
│   │   ├── ExportSucceeded.ts
│   │   ├── ExportFailed.ts
│   │   └── ExportDeadLettered.ts
│   ├── services/
│   │   └── ExportRetryPolicy.ts      # Exponential backoff: 1s, 5s, 30s (3 attempts)
│   └── repositories/
│       └── IConnectorRepository.ts
├── application/
│   ├── handlers/
│   │   └── OnCitationExportReady.ts  # Creates ExportJob per configured connector
│   ├── commands/
│   │   ├── RegisterConnectorCommand.ts
│   │   ├── ProcessExportJobCommand.ts
│   │   └── RevokeConnectorCommand.ts
│   └── queries/
│       └── GetConnectorStatusQuery.ts
├── infrastructure/
│   ├── ZoteroExportAdapter.ts        # Zotero Web API v3
│   ├── NotionExportAdapter.ts        # Notion API
│   ├── ObsidianExportAdapter.ts      # Obsidian Local REST API plugin
│   ├── CredentialVault.ts            # AES-256-GCM encrypted storage
│   └── ExportJobWorker.ts            # Redis queue consumer
└── interfaces/
    ├── IntegrationRoutes.ts
    └── OAuthCallbackRoutes.ts        # Handles OAuth redirects from Zotero/Notion
```

### API Endpoints

```
GET    /api/v1/integrations
  Response: { connectors: [{ type, status, lastSyncAt }] }

POST   /api/v1/integrations/zotero/connect
  Response: { oauthRedirectUrl }   # Initiates Zotero OAuth flow

GET    /api/v1/integrations/zotero/callback
  Query: { code, state }           # OAuth callback

DELETE /api/v1/integrations/:connectorId
  Response: 204

GET    /api/v1/integrations/jobs
  Query: { status?, since? }
  Response: { jobs: ExportJob[] }
```

### Connector Adapters (Interface Contract)

Each connector implements:
```typescript
interface IExportAdapter {
  export(citation: ExportPayload, credentials: EncryptedCredential): Promise<ExportResult>;
  testConnection(credentials: EncryptedCredential): Promise<boolean>;
  formatPayload(citation: FormattedCitation): ExportPayload;
}
```

**Zotero Adapter:** Maps to Zotero API item type, pushes to researcher's default library or specified collection, attaches permanent link as `extra` field with `Archive URL:` prefix.

**Notion Adapter:** Creates a new database entry in researcher's chosen Notion citation database. Maps to configurable property schema. Attaches permanent link as a URL property.

**Obsidian Adapter:** Calls the Obsidian Local REST API plugin (researcher runs on their machine). Creates a new markdown note with citation in frontmatter + BibTeX block.

---

## Module 6: Identity & Access Module

### Responsibility
Thin wrapper around Clerk. Manages workspace creation, membership, and institutional license seat allocation. All authentication is delegated to Clerk — this module handles Permacite-specific authorization logic.

### Directory Structure
```
modules/identity/
├── domain/
│   ├── Researcher.ts              # Aggregate root (shadow of Clerk user)
│   ├── Workspace.ts               # Aggregate root
│   ├── WorkspaceMember.ts         # Entity
│   ├── WorkspaceRole.ts           # Enum: OWNER | ADMIN | MEMBER | VIEWER
│   ├── EmailAddress.ts            # Value object
│   └── events/
│       ├── ResearcherCreated.ts
│       ├── WorkspaceCreated.ts
│       └── MemberInvited.ts
├── application/
│   ├── handlers/
│   │   └── OnClerkWebhook.ts      # Clerk → domain event translator
│   ├── commands/
│   │   ├── CreateWorkspaceCommand.ts
│   │   └── InviteMemberCommand.ts
│   └── queries/
│       └── GetWorkspaceQuery.ts
└── interfaces/
    ├── WorkspaceRoutes.ts
    └── ClerkWebhookRoutes.ts
```

### Permission Model

```typescript
// RBAC matrix per workspace role
const permissions = {
  OWNER:  ['capture', 'cite', 'export', 'manage_integrations', 'manage_members', 'manage_billing'],
  ADMIN:  ['capture', 'cite', 'export', 'manage_integrations', 'manage_members'],
  MEMBER: ['capture', 'cite', 'export'],
  VIEWER: ['cite:read', 'export:read'],
};
```

Row-level security in PostgreSQL enforces workspace isolation:
```sql
-- Row-level security example
ALTER TABLE captures ENABLE ROW LEVEL SECURITY;
CREATE POLICY captures_workspace_isolation ON captures
  USING (researcher_id = current_setting('app.current_researcher_id')::uuid
         OR workspace_id = current_setting('app.current_workspace_id')::uuid);
```

---

## Module 7: Billing Module

### Responsibility
Tracks subscription state, enforces usage limits, and provides the Anti-Corruption Layer that all other modules use to check plan entitlements. Never holds payment details — Stripe is the source of truth.

### Directory Structure
```
modules/billing/
├── domain/
│   ├── Subscription.ts           # Aggregate root
│   ├── SubscriptionPlan.ts       # Value object (planId, limits, price)
│   ├── UsageSummary.ts           # Value object (captures used, storage used)
│   ├── PlanTier.ts               # Enum: FREE | INDIVIDUAL | PRO | TEAM | INSTITUTIONAL | ENTERPRISE
│   ├── events/
│   │   ├── SubscriptionCreated.ts
│   │   ├── SubscriptionUpgraded.ts
│   │   ├── UsageLimitApproached.ts
│   │   ├── UsageLimitExceeded.ts
│   │   └── InstitutionalLicenseIssued.ts
│   └── services/
│       └── UsageEnforcementService.ts    # The Anti-Corruption Layer
├── application/
│   ├── handlers/
│   │   ├── OnStripeWebhook.ts            # Stripe → domain events
│   │   └── OnCaptureCreated.ts           # Increment usage counter
│   └── commands/
│       └── IncrementUsageCommand.ts
└── interfaces/
    ├── BillingRoutes.ts
    └── StripeWebhookRoutes.ts
```

### Anti-Corruption Layer (Usage Enforcement)

Other modules NEVER check Stripe directly. They call:

```typescript
// UsageEnforcementService — the single source of truth for entitlement checks
interface IUsageEnforcementService {
  canCapture(researcherId: string): Promise<{ allowed: boolean; remaining: number }>;
  canExport(researcherId: string, connectorType: ConnectorType): Promise<boolean>;
  canCreateWorkspace(researcherId: string): Promise<boolean>;
  getStorageQuota(researcherId: string): Promise<{ usedBytes: number; limitBytes: number }>;
}
```

Usage is cached in Redis (60-second TTL) to avoid hitting PostgreSQL on every capture.

---

## Cross-Module Communication Summary

| Source Module | Event | Consumer Modules | Purpose |
|--------------|-------|-----------------|---------|
| Capture | `CaptureCreated` | Archival, Billing | Trigger archiving; increment usage |
| Capture | `CaptureFailed` | (notification future) | Error alerting |
| Archival | `ArchiveConfirmed` | Citation | Trigger AI extraction |
| Archival | `ArchiveDegraded` | Citation, (notification) | Use fallback archive; alert researcher |
| Archival | `ArchiveFailed` | (notification future) | Capture cannot proceed |
| Citation | `CitationCreated` | Library | Auto-add to default collection |
| Citation | `CitationExportReady` | Integration | Create export jobs per connector |
| Citation | `CitationReviewRequested` | (notification future) | Request researcher review |
| Integration | `ExportSucceeded` | Billing | (future: metered exports) |
| Billing | `UsageLimitExceeded` | Capture | Block new captures (via middleware) |
| Billing | `InstitutionalLicenseIssued` | Identity | Provision workspace seats |
| Identity | `WorkspaceCreated` | Billing | Create workspace subscription |

---

*Module boundaries are enforced at build time via `eslint-plugin-boundaries`. Any import from `modules/citation` into `modules/capture` will fail the build.*
