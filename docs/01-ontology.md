# Permacite — Domain Ontology & Ubiquitous Language
## DDD Strategic Design · May 2026

---

## 1. Product Vision (Domain Statement)

Permacite solves **link rot** — the degradation of cited URLs over time — by inverting the traditional citation workflow: **archive first, cite second**. A researcher captures a source in a single browser click; the page is permanently preserved before a citation is ever formatted.

---

## 2. Ubiquitous Language

The following terms have precise meanings within the Permacite domain. Every developer, designer, and stakeholder must use these terms consistently across code, documentation, and conversation.

| Term | Definition |
|------|-----------|
| **Capture** | The act of saving a web page's screenshot, HTML, and URL at a specific point in time via the browser extension |
| **Archive** | A permanently stored copy of a captured page, either in the Wayback Machine or a local WACZ file |
| **Archive Reference** | A stable, permanent URL pointing to the archived copy of a page |
| **Citation** | A formatted bibliographic reference derived from a capture, conforming to a specific citation style |
| **Citation Style** | A formatting standard for citations (APA 7th, MLA 9th, Chicago 17th, BibTeX, Bluebook) |
| **Capture Metadata** | AI-extracted structured data from a capture: title, author, publication date, publisher, URL, access date |
| **Highlighted Quote** | A passage of text selected by the researcher during capture, attached to the citation as supporting evidence |
| **Research Library** | A researcher's personal or team-shared collection of captures and citations |
| **Collection** | A named sub-group within a Research Library, analogous to a folder or project |
| **Export** | The act of pushing a formatted citation to an external tool (Zotero, Notion, Obsidian) |
| **Link Rot** | The degradation of a URL to an inaccessible or altered state over time |
| **Permanent Link** | An Archive Reference that remains resolvable indefinitely, replacing the original live URL in a citation |
| **Confidence Score** | A numeric (0–1) measure of AI certainty in extracted metadata fields |
| **Fallback Capture** | A local screenshot + HTML snapshot taken when Wayback Machine archiving is unavailable (paywall, JS-heavy page) |
| **Researcher** | Any end user of Permacite (graduate student, academic, journalist, legal scholar) |
| **Institution** | A university, law library, or research organization holding a site license |
| **Workspace** | A multi-user shared environment under an institutional or team subscription |
| **Citation Authority Flywheel** | The network effect where early citation adoption increases discoverability and institutional lock-in |
| **Audit Log** | An immutable record of all AI-generated citation decisions, required for regulatory compliance |
| **Integration Connector** | A configured connection between Permacite and an external tool (Zotero, Notion, Obsidian) |

---

## 3. Bounded Contexts

Permacite decomposes into **seven bounded contexts**. Each context owns its domain model, has independent deployment boundaries, and communicates with others through well-defined contracts.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Permacite Platform                        │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐ │
│  │   Capture    │──▶│   Archival   │──▶│     Citation         │ │
│  │   Context    │   │   Context    │   │     Context          │ │
│  └──────────────┘   └──────────────┘   └──────────────────────┘ │
│          │                                        │              │
│          ▼                                        ▼              │
│  ┌──────────────┐                     ┌──────────────────────┐  │
│  │   Library    │                     │   Integration        │  │
│  │   Context    │                     │   Context            │  │
│  └──────────────┘                     └──────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────┐   ┌──────────────────────────┐    │
│  │  Identity & Access       │   │  Billing & Subscription  │    │
│  │  Context                 │   │  Context                 │    │
│  └──────────────────────────┘   └──────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3.1 Capture Context

**Purpose:** Manages the browser extension's capture lifecycle — from user-initiated page capture through to a structured capture record ready for archiving.

**Core Responsibilities:**
- Orchestrate browser extension interactions
- Receive screenshot, raw HTML, URL, and optional highlight from the extension
- Validate and sanitize incoming capture payload
- Detect capture type (live page, paywalled page, JS-heavy page)
- Emit `CaptureCreated` domain event for downstream consumption

**Aggregate Root: `Capture`**

```
Capture
├── captureId: UUID                     (identity)
├── researcherId: ResearcherId          (owner)
├── sourceUrl: URL                      (value object)
├── capturedAt: Timestamp
├── pageTitle: String?                  (raw, pre-AI-extraction)
├── screenshotData: BinaryRef           (S3/R2 reference)
├── rawHtml: BinaryRef                  (S3/R2 reference)
├── highlightedText: HighlightedQuote?  (value object)
├── captureType: Enum[LIVE, PAYWALLED, JS_HEAVY, LOCAL_FALLBACK]
├── status: Enum[PENDING, ARCHIVING, ARCHIVED, FAILED]
└── extensionVersion: String
```

**Value Objects:**
- `URL` — validated web address with scheme, domain, path, query string
- `HighlightedQuote` — text content + character offset range
- `BinaryRef` — storage key + bucket + content hash

**Domain Events Emitted:**
| Event | Trigger |
|-------|---------|
| `CaptureCreated` | Successful capture payload received |
| `CaptureFailed` | Validation or storage failure |
| `CaptureRetryRequested` | User manually retries a failed capture |

---

### 3.2 Archival Context

**Purpose:** Ensures every capture has a permanent, stable, retrievable copy. Orchestrates Wayback Machine API submission and manages local WACZ fallback storage.

**Core Responsibilities:**
- Submit URLs to Wayback Machine Save API
- Poll for archiving confirmation
- Fall back to local WACZ storage for paywalled/JS-heavy pages
- Manage tiered storage lifecycle (hot → warm → cold)
- Ensure archive permanence SLA: captured pages remain accessible indefinitely

**Aggregate Root: `Archive`**

```
Archive
├── archiveId: UUID
├── captureId: UUID                     (reference to Capture Context)
├── primaryArchiveRef: ArchiveReference (Wayback Machine URL)
├── fallbackArchiveRef: ArchiveReference? (local WACZ)
├── archivedAt: Timestamp
├── archiveProvider: Enum[WAYBACK_MACHINE, LOCAL_WACZ, HYBRID]
├── storageClass: Enum[HOT, WARM, COLD]
├── contentHash: SHA256
├── pageSizeBytes: Long
└── status: Enum[PENDING, IN_PROGRESS, CONFIRMED, FAILED, DEGRADED]
```

**Value Objects:**
- `ArchiveReference` — permanent URL (e.g. `https://web.archive.org/web/2026*/https://...`) with provider tag
- `ContentHash` — SHA-256 fingerprint for tamper detection

**Domain Events Emitted:**
| Event | Trigger |
|-------|---------|
| `ArchiveConfirmed` | Wayback Machine or local storage confirms permanence |
| `ArchiveDegraded` | Primary archive failed; fallback used |
| `ArchiveFailed` | Both primary and fallback archiving failed |
| `StorageTierMoved` | Page moved from hot to warm/cold storage |

**Domain Services:**
- `WaybackSubmissionService` — wraps Wayback Machine Save API
- `LocalArchiveService` — manages WACZ capture using Scoop-like engine
- `StorageTierPolicy` — decides when to move archives between tiers

---

### 3.3 Citation Context

**Purpose:** The core intelligence layer. Extracts structured metadata from captures via AI/OCR, formats citations in multiple styles, and assigns confidence scores.

**Core Responsibilities:**
- AI-powered extraction of title, author, date, publisher from screenshot + HTML
- Cross-validation of extracted metadata against DOI, PubMed, and CrossRef APIs
- Citation formatting in APA 7th, MLA 9th, Chicago 17th, BibTeX, Bluebook
- Confidence scoring for every extracted field
- Human-review flagging for citations below confidence threshold
- Immutable audit log of all AI extraction decisions

**Aggregate Root: `Citation`**

```
Citation
├── citationId: UUID
├── captureId: UUID                     (reference to Capture Context)
├── archiveId: UUID                     (reference to Archival Context)
├── researcherId: ResearcherId
├── metadata: CitationMetadata          (value object)
├── formattedVariants: Map<CitationStyle, FormattedCitation>
├── confidenceScore: ConfidenceScore    (value object)
├── highlightedQuote: HighlightedQuote?
├── permanentLink: ArchiveReference     (from Archival Context)
├── reviewStatus: Enum[AUTO_APPROVED, PENDING_REVIEW, REVIEWED, REJECTED]
├── auditLog: List<AuditEntry>          (append-only)
└── createdAt: Timestamp
```

**Value Objects:**
- `CitationMetadata` — { title, authors[], publicationDate, publisher, volume, issue, pages, doi, isbn }
- `CitationStyle` — enum with formatting rules (APA, MLA, Chicago, BibTeX, Bluebook)
- `FormattedCitation` — rendered string + raw structured data
- `ConfidenceScore` — per-field confidence map (0.0–1.0), overall score, extraction model version
- `AuditEntry` — { timestamp, modelId, modelVersion, inputFields, outputFields, score, decision }

**Domain Events Emitted:**
| Event | Trigger |
|-------|---------|
| `CitationCreated` | Metadata extracted + citation formatted |
| `CitationReviewRequested` | Confidence below threshold (default: 0.95) |
| `CitationApproved` | Human or auto-approval confirmed |
| `CitationExportReady` | Citation ready for push to external tools |

**Domain Services:**
- `MetadataExtractionService` — calls multimodal AI API (GPT-4o / Claude)
- `CrossRefValidationService` — cross-checks DOI/author against CrossRef database
- `PubMedValidationService` — validates academic citations against PubMed
- `CitationFormatterService` — applies style rules to produce formatted strings
- `ConfidenceScoringService` — aggregates per-field scores into citation confidence

---

### 3.4 Library Context

**Purpose:** Manages a researcher's personal or team workspace — organizing captures, citations, and collections for long-term research use.

**Core Responsibilities:**
- CRUD for Research Libraries and Collections
- Tagging, search, and filtering across the library
- Shared library management for teams
- Citation linking (one capture → multiple citations in different styles)
- Export to bibliography formats (bulk BibTeX, RIS, CSV)

**Aggregate Root: `ResearchLibrary`**

```
ResearchLibrary
├── libraryId: UUID
├── ownerId: ResearcherId | WorkspaceId
├── name: String
├── collections: List<Collection>
├── totalCaptures: Int
├── createdAt: Timestamp
└── settings: LibrarySettings
```

**Aggregate Root: `LibraryItem`**

```
LibraryItem
├── itemId: UUID
├── libraryId: UUID
├── collectionId: UUID?
├── citationId: UUID                    (reference to Citation Context)
├── captureId: UUID                     (reference to Capture Context)
├── tags: List<Tag>
├── notes: ResearcherNote?
├── addedAt: Timestamp
└── lastAccessedAt: Timestamp
```

**Domain Events Emitted:**
| Event | Trigger |
|-------|---------|
| `LibraryItemAdded` | Citation saved to library |
| `CollectionCreated` | New collection created |
| `LibraryShared` | Library shared with workspace member |
| `BulkExportRequested` | Researcher requests bibliography export |

---

### 3.5 Integration Context

**Purpose:** Manages all outbound connections to external research tools — Zotero, Notion, Obsidian, and future connectors. Handles OAuth, API token management, push/sync, and failure recovery.

**Core Responsibilities:**
- Connector registration and OAuth flow management
- Push citation to Zotero library (via Zotero Web API)
- Create Notion database entry or page
- Append to Obsidian vault markdown file
- Retry and dead-letter queue for failed exports
- Webhook ingestion for bidirectional sync (future)

**Aggregate Root: `IntegrationConnector`**

```
IntegrationConnector
├── connectorId: UUID
├── researcherId: ResearcherId
├── connectorType: Enum[ZOTERO, NOTION, OBSIDIAN, MENDELEY]
├── credentials: EncryptedCredential    (value object)
├── status: Enum[ACTIVE, INACTIVE, ERROR, REVOKED]
├── lastSyncAt: Timestamp?
├── settings: ConnectorSettings
└── createdAt: Timestamp
```

**Aggregate Root: `ExportJob`**

```
ExportJob
├── jobId: UUID
├── citationId: UUID
├── connectorId: UUID
├── payload: ExportPayload              (value object)
├── status: Enum[QUEUED, IN_PROGRESS, SUCCEEDED, FAILED, DEAD_LETTERED]
├── attempts: Int
├── lastError: String?
└── completedAt: Timestamp?
```

**Domain Events Emitted:**
| Event | Trigger |
|-------|---------|
| `ConnectorRegistered` | New integration configured |
| `ExportSucceeded` | Citation pushed to external tool |
| `ExportFailed` | Push attempt failed (retryable) |
| `ExportDeadLettered` | Max retries exceeded |

---

### 3.6 Identity & Access Context

**Purpose:** Manages researcher identities, team workspaces, institutional memberships, roles, and access control.

**Core Responsibilities:**
- Researcher registration and authentication (email, Google/ORCID SSO)
- Team/Workspace creation and membership management
- Role-based access control (RBAC)
- Institutional license seat management
- Session management and token issuance

**Aggregate Root: `Researcher`**

```
Researcher
├── researcherId: UUID
├── email: EmailAddress                 (value object)
├── displayName: String
├── authProviders: List<AuthProvider>   (email, Google, ORCID)
├── workspaceMemberships: List<WorkspaceMembership>
├── createdAt: Timestamp
└── status: Enum[ACTIVE, SUSPENDED, DELETED]
```

**Aggregate Root: `Workspace`**

```
Workspace
├── workspaceId: UUID
├── name: String
├── ownerResearcherId: UUID
├── members: List<WorkspaceMember>      (with roles)
├── institutionId: UUID?                (for institutional licenses)
├── settings: WorkspaceSettings
└── createdAt: Timestamp
```

**Value Objects:**
- `WorkspaceMember` — { researcherId, role: Enum[OWNER, ADMIN, MEMBER, VIEWER] }
- `EmailAddress` — validated email with domain verification

---

### 3.7 Billing & Subscription Context

**Purpose:** Manages subscription plans, billing cycles, usage metering, and institutional licensing contracts.

**Core Responsibilities:**
- Subscription plan management (Free, Individual, Pro, Team, Institutional, Enterprise)
- Usage tracking (captures/month, storage bytes, export calls)
- Stripe integration for payment processing
- Institutional license issuance and seat allocation
- Overage handling and upgrade prompts
- Invoicing and tax compliance

**Aggregate Root: `Subscription`**

```
Subscription
├── subscriptionId: UUID
├── subscriberId: ResearcherId | WorkspaceId
├── plan: SubscriptionPlan              (value object)
├── status: Enum[ACTIVE, TRIAL, PAST_DUE, CANCELLED, PAUSED]
├── billingCycleAnchor: Date
├── currentPeriodEnd: Date
├── stripeSubscriptionId: String
└── usageThisPeriod: UsageSummary       (value object)
```

**Value Objects:**
- `SubscriptionPlan` — { planId, name, capturesPerMonth, storageGB, integrations, price, currency }
- `UsageSummary` — { capturesUsed, storageUsedBytes, exportsUsed, periodStart, periodEnd }

**Domain Events Emitted:**
| Event | Trigger |
|-------|---------|
| `SubscriptionCreated` | New subscription activated |
| `SubscriptionUpgraded` | Plan changed to higher tier |
| `SubscriptionCancelled` | Researcher cancels |
| `UsageLimitApproached` | 80% of plan limit reached |
| `UsageLimitExceeded` | 100% of plan limit reached |
| `InstitutionalLicenseIssued` | Institutional contract activated |

---

## 4. Context Relationships & Integration Patterns

### Upstream → Downstream Flow

```
Browser Extension
      │
      ▼ (REST / WebSocket)
Capture Context
      │
      ├──[CaptureCreated event]──▶ Archival Context
      │                                   │
      │                           [ArchiveConfirmed]
      │                                   │
      └──[CaptureCreated + ArchiveConfirmed]──▶ Citation Context
                                                      │
                                              [CitationCreated]
                                                      │
                                    ┌─────────────────┼─────────────────┐
                                    ▼                 ▼                 ▼
                             Library Context   Integration Context  Audit/Compliance
```

### Context Map (Integration Styles)

| Upstream Context | Downstream Context | Integration Style | Notes |
|------------------|-------------------|-------------------|-------|
| Capture → Archival | Domain Event (async) | `CaptureCreated` event via message bus |
| Archival → Citation | Domain Event (async) | `ArchiveConfirmed` event triggers AI extraction |
| Citation → Library | Domain Event (async) | `CitationCreated` auto-adds to default library |
| Citation → Integration | Domain Event (async) | `CitationExportReady` triggers export jobs |
| Identity → All Contexts | Shared Kernel | `ResearcherId` is universal identity token |
| Billing → All Contexts | Anti-Corruption Layer | Usage gates enforced via middleware |
| All Contexts → Audit | Published Language | Append-only audit stream for compliance |

---

## 5. Domain Event Catalogue

| Event | Source Context | Consumers | Payload Summary |
|-------|---------------|-----------|-----------------|
| `CaptureCreated` | Capture | Archival, Billing | captureId, researcherId, url, captureType, timestamp |
| `CaptureFailed` | Capture | Notification | captureId, reason, timestamp |
| `ArchiveConfirmed` | Archival | Citation | captureId, archiveId, permanentUrl, provider |
| `ArchiveFailed` | Archival | Notification, Capture | captureId, reason |
| `CitationCreated` | Citation | Library, Integration | citationId, captureId, styles[], confidenceScore |
| `CitationReviewRequested` | Citation | Notification | citationId, fields[], scores[] |
| `CitationExportReady` | Citation | Integration | citationId, formats[], connectorIds[] |
| `ExportSucceeded` | Integration | Notification, Billing | jobId, connectorType, citationId |
| `ExportFailed` | Integration | Notification | jobId, reason, retryCount |
| `UsageLimitApproached` | Billing | Notification | researcherId, planId, percentUsed |
| `InstitutionalLicenseIssued` | Billing | Identity | institutionId, seats, features, expiry |

---

## 6. Aggregate Invariants

Critical business rules enforced within aggregates (never violated, even across contexts):

1. **A Citation must have a confirmed Archive Reference** — a citation cannot be finalized without at least one valid permanent link
2. **Confidence threshold gate** — citations with overall confidence < 0.85 must be flagged for human review before auto-export
3. **Capture immutability** — once a Capture is created, its source URL, screenshot, and HTML cannot be modified (only metadata annotations)
4. **Archive permanence** — an Archive record is never deleted; its storage class may change but the reference survives
5. **Audit log append-only** — AuditEntry records are never updated or deleted; they form the regulatory compliance trail
6. **Usage enforcement** — a Capture cannot be initiated if the researcher has exceeded their plan's monthly capture limit
7. **Workspace seat limit** — a Workspace cannot exceed the seat count of its institutional license

---

## 7. Core Domain vs. Supporting vs. Generic Subdomains

| Subdomain | Classification | Rationale |
|-----------|---------------|-----------|
| Archive-First Citation Workflow | **Core Domain** | Primary differentiator; custom build |
| AI Metadata Extraction + Confidence Scoring | **Core Domain** | Competitive moat; custom build |
| Archival Orchestration (Wayback + Local) | **Core Domain** | Technical moat; custom build |
| Research Library & Collections | **Supporting Domain** | Valuable but not differentiating; custom build |
| Integration Connectors (Zotero, Notion) | **Supporting Domain** | Critical distribution; custom build |
| Identity & Access Management | **Generic Subdomain** | Use Auth0/Clerk; do not custom-build |
| Billing & Subscription Management | **Generic Subdomain** | Use Stripe; do not custom-build |
| Email Notifications | **Generic Subdomain** | Use Resend/SendGrid |
| Audit Logging & Compliance | **Supporting Domain** | Regulatory requirement; lightweight custom build |

---

*This ontology is the source of truth for all team communication. When domain terms appear in code, they must match this document exactly.*
