# Permacite — Data Architecture
## Storage Strategy, Schema Design & Data Flows · May 2026

---

## 1. Data Storage Strategy

Permacite uses a **polyglot persistence** model — different storage technologies optimized for each data access pattern:

| Store | Technology | Purpose | Data Owned |
|-------|-----------|---------|-----------|
| **Primary DB** | PostgreSQL 16 (Supabase) | Structured relational data, transactions, full-text search | Captures, archives, citations, library, subscriptions |
| **Object Storage** | Cloudflare R2 | Binary blobs — screenshots, HTML, WACZ files | Capture screenshots, raw HTML, archived WACZ |
| **Cache** | Upstash Redis | Session cache, usage counters, event bus | Usage quotas, citation format cache, message queues |
| **Search Index** | PostgreSQL FTS (MVP) → Typesense (scale) | Citation full-text search | Citation text, metadata, tags |
| **Audit Store** | PostgreSQL (append-only partitioned table) | Immutable AI decision audit trail | AI extraction decisions, confidence scores |
| **Event Store** | Redis Streams (MVP) → SQS (scale) | Domain event messaging | All domain events |

---

## 2. Database Schema

### Schema Namespace Conventions

Each module owns its schema namespace — no cross-namespace table joins:

```
capture.*        — Capture Module tables
archival.*       — Archival Module tables
citation.*       — Citation Module tables
library.*        — Library Module tables
integration.*    — Integration Module tables
identity.*       — Identity & Access tables
billing.*        — Billing & Subscription tables
audit.*          — Audit log (append-only)
```

---

### 2.1 Capture Schema

```sql
-- capture.captures
CREATE TABLE capture.captures (
    capture_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    researcher_id       UUID NOT NULL,
    workspace_id        UUID,
    source_url          TEXT NOT NULL,
    source_url_domain   TEXT NOT NULL GENERATED ALWAYS AS (
                            regexp_replace(source_url, '^https?://([^/]+).*', '\1')
                        ) STORED,
    capture_type        TEXT NOT NULL CHECK (capture_type IN ('LIVE', 'PAYWALLED', 'JS_HEAVY', 'LOCAL_FALLBACK')),
    status              TEXT NOT NULL DEFAULT 'PENDING'
                            CHECK (status IN ('PENDING', 'ARCHIVING', 'ARCHIVED', 'FAILED')),
    screenshot_ref      JSONB NOT NULL,   -- { bucket, key, sizeBytes, contentHash }
    html_ref            JSONB NOT NULL,   -- { bucket, key, sizeBytes, contentHash, encoding }
    highlighted_text    TEXT,
    highlight_offset    JSONB,            -- { start, end }
    extension_version   TEXT NOT NULL,
    page_title_raw      TEXT,             -- Pre-AI extraction raw title
    captured_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX captures_researcher_id_idx ON capture.captures (researcher_id, captured_at DESC);
CREATE INDEX captures_status_idx ON capture.captures (status) WHERE status != 'ARCHIVED';
CREATE INDEX captures_domain_idx ON capture.captures (source_url_domain);
```

---

### 2.2 Archival Schema

```sql
-- archival.archives
CREATE TABLE archival.archives (
    archive_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capture_id              UUID NOT NULL UNIQUE,  -- 1:1 with capture
    primary_archive_url     TEXT,                  -- Wayback Machine permanent URL
    primary_archive_provider TEXT CHECK (primary_archive_provider IN ('WAYBACK_MACHINE', 'LOCAL_WACZ')),
    fallback_archive_url    TEXT,                  -- Local WACZ URL if primary failed
    fallback_storage_ref    JSONB,                 -- R2 reference for local WACZ
    archive_provider        TEXT NOT NULL CHECK (archive_provider IN ('WAYBACK_MACHINE', 'LOCAL_WACZ', 'HYBRID')),
    storage_class           TEXT NOT NULL DEFAULT 'HOT'
                                CHECK (storage_class IN ('HOT', 'WARM', 'COLD')),
    content_hash            TEXT NOT NULL,         -- SHA-256 of archived content
    page_size_bytes         BIGINT,
    status                  TEXT NOT NULL DEFAULT 'PENDING'
                                CHECK (status IN ('PENDING', 'IN_PROGRESS', 'CONFIRMED', 'FAILED', 'DEGRADED')),
    archived_at             TIMESTAMPTZ,
    last_verified_at        TIMESTAMPTZ,           -- Last time archive URL was confirmed alive
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX archives_capture_id_idx ON archival.archives (capture_id);
CREATE INDEX archives_storage_class_idx ON archival.archives (storage_class, last_verified_at)
    WHERE status = 'CONFIRMED';

-- Storage tier transition log
CREATE TABLE archival.storage_tier_transitions (
    transition_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    archive_id      UUID NOT NULL REFERENCES archival.archives(archive_id),
    from_class      TEXT NOT NULL,
    to_class        TEXT NOT NULL,
    reason          TEXT,
    transitioned_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### 2.3 Citation Schema

```sql
-- citation.citations
CREATE TABLE citation.citations (
    citation_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capture_id          UUID NOT NULL,
    archive_id          UUID NOT NULL,
    researcher_id       UUID NOT NULL,
    workspace_id        UUID,
    status              TEXT NOT NULL DEFAULT 'EXTRACTING'
                            CHECK (status IN ('EXTRACTING', 'FORMATTING', 'REVIEW_PENDING', 'AUTO_APPROVED', 'REVIEWED', 'REJECTED')),
    overall_confidence  NUMERIC(4, 3) CHECK (overall_confidence BETWEEN 0 AND 1),
    permanent_link      TEXT,          -- Final archive URL attached to citation
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    approved_at         TIMESTAMPTZ,
    reviewed_by         UUID           -- researcher_id of reviewer (if human review)
);

-- citation.citation_metadata
-- Flexible JSONB for metadata that varies by source type
CREATE TABLE citation.citation_metadata (
    citation_id         UUID PRIMARY KEY REFERENCES citation.citations(citation_id),
    title               TEXT,
    authors             JSONB NOT NULL DEFAULT '[]',  -- [{ last, first, orcid? }]
    publication_date    DATE,
    publisher           TEXT,
    journal_name        TEXT,
    volume              TEXT,
    issue               TEXT,
    pages               TEXT,
    doi                 TEXT,
    isbn                TEXT,
    source_type         TEXT CHECK (source_type IN ('journal-article', 'book', 'website', 'report', 'other')),
    confidence_breakdown JSONB NOT NULL DEFAULT '{}', -- { title: 0.95, authors: 0.88, date: 0.72, ... }
    extraction_model    TEXT NOT NULL,                -- "claude-sonnet-4-6" etc.
    raw_ai_response     JSONB,                        -- Full AI response for debugging
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- citation.formatted_citations
-- Pre-computed formatted strings per style (avoid re-formatting on every read)
CREATE TABLE citation.formatted_citations (
    formatted_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citation_id         UUID NOT NULL REFERENCES citation.citations(citation_id),
    style               TEXT NOT NULL CHECK (style IN ('APA_7', 'MLA_9', 'CHICAGO_17', 'BIBTEX', 'BLUEBOOK')),
    formatted_text      TEXT NOT NULL,
    formatted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (citation_id, style)
);

CREATE INDEX formatted_citations_citation_style_idx ON citation.formatted_citations (citation_id, style);

-- Full-text search index
CREATE INDEX citations_fts_idx ON citation.citation_metadata
    USING gin(to_tsvector('english',
        coalesce(title, '') || ' ' ||
        coalesce(publisher, '') || ' ' ||
        coalesce(journal_name, '')
    ));
```

---

### 2.4 Audit Schema (Append-Only, Partitioned)

```sql
-- audit.ai_extraction_log — NEVER UPDATED OR DELETED
CREATE TABLE audit.ai_extraction_log (
    log_id              UUID NOT NULL DEFAULT gen_random_uuid(),
    citation_id         UUID NOT NULL,
    researcher_id       UUID NOT NULL,
    event_type          TEXT NOT NULL,  -- 'EXTRACTION_ATTEMPTED', 'EXTRACTION_SUCCEEDED', 'HUMAN_REVIEW', 'APPROVED', 'REJECTED'
    model_id            TEXT,
    model_version       TEXT,
    input_tokens        INT,
    output_tokens       INT,
    confidence_score    NUMERIC(4,3),
    decision            TEXT,
    metadata            JSONB,
    logged_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (log_id, logged_at)
) PARTITION BY RANGE (logged_at);

-- Monthly partitions (created by pg_cron or migration at month start)
CREATE TABLE audit.ai_extraction_log_2026_05
    PARTITION OF audit.ai_extraction_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

### 2.5 Library Schema

```sql
-- library.research_libraries
CREATE TABLE library.research_libraries (
    library_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id        UUID NOT NULL,         -- researcher_id or workspace_id
    owner_type      TEXT NOT NULL CHECK (owner_type IN ('RESEARCHER', 'WORKSPACE')),
    name            TEXT NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- library.collections
CREATE TABLE library.collections (
    collection_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES library.research_libraries(library_id),
    name            TEXT NOT NULL,
    description     TEXT,
    parent_id       UUID REFERENCES library.collections(collection_id),  -- Nested collections
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- library.library_items
CREATE TABLE library.library_items (
    item_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id          UUID NOT NULL REFERENCES library.research_libraries(library_id),
    collection_id       UUID REFERENCES library.collections(collection_id),
    citation_id         UUID NOT NULL,    -- Cross-module reference (no FK — module boundary)
    capture_id          UUID NOT NULL,    -- Cross-module reference
    tags                TEXT[] NOT NULL DEFAULT '{}',
    notes               TEXT,
    added_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_accessed_at    TIMESTAMPTZ,
    UNIQUE (library_id, citation_id)
);

CREATE INDEX library_items_tags_idx ON library.library_items USING gin(tags);
```

---

### 2.6 Integration Schema

```sql
-- integration.connectors
CREATE TABLE integration.connectors (
    connector_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    researcher_id       UUID NOT NULL,
    connector_type      TEXT NOT NULL CHECK (connector_type IN ('ZOTERO', 'NOTION', 'OBSIDIAN', 'MENDELEY')),
    status              TEXT NOT NULL DEFAULT 'ACTIVE'
                            CHECK (status IN ('ACTIVE', 'INACTIVE', 'ERROR', 'REVOKED')),
    encrypted_creds     BYTEA NOT NULL,   -- AES-256-GCM encrypted JSON blob
    settings            JSONB NOT NULL DEFAULT '{}',  -- { defaultCollection, autoExport, ... }
    last_sync_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (researcher_id, connector_type)
);

-- integration.export_jobs
CREATE TABLE integration.export_jobs (
    job_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citation_id     UUID NOT NULL,
    connector_id    UUID NOT NULL REFERENCES integration.connectors(connector_id),
    status          TEXT NOT NULL DEFAULT 'QUEUED'
                        CHECK (status IN ('QUEUED', 'IN_PROGRESS', 'SUCCEEDED', 'FAILED', 'DEAD_LETTERED')),
    payload         JSONB NOT NULL,         -- Connector-specific export data
    attempts        INT NOT NULL DEFAULT 0,
    last_error      TEXT,
    external_id     TEXT,                   -- ID assigned by Zotero/Notion after successful export
    queued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX export_jobs_status_idx ON integration.export_jobs (status, queued_at)
    WHERE status IN ('QUEUED', 'FAILED');
```

---

### 2.7 Billing Schema

```sql
-- billing.subscriptions
CREATE TABLE billing.subscriptions (
    subscription_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id           UUID NOT NULL,
    subscriber_type         TEXT NOT NULL CHECK (subscriber_type IN ('RESEARCHER', 'WORKSPACE')),
    plan_tier               TEXT NOT NULL CHECK (plan_tier IN ('FREE', 'INDIVIDUAL', 'PRO', 'TEAM', 'INSTITUTIONAL', 'ENTERPRISE')),
    status                  TEXT NOT NULL CHECK (status IN ('ACTIVE', 'TRIAL', 'PAST_DUE', 'CANCELLED', 'PAUSED')),
    stripe_subscription_id  TEXT UNIQUE,
    stripe_customer_id      TEXT,
    billing_cycle_anchor    DATE,
    current_period_start    TIMESTAMPTZ,
    current_period_end      TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- billing.usage_records (append-only, one row per billing period)
CREATE TABLE billing.usage_records (
    record_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id       UUID NOT NULL,
    period_start        TIMESTAMPTZ NOT NULL,
    period_end          TIMESTAMPTZ NOT NULL,
    captures_used       INT NOT NULL DEFAULT 0,
    exports_used        INT NOT NULL DEFAULT 0,
    storage_used_bytes  BIGINT NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (subscriber_id, period_start)
);
```

---

## 3. Object Storage Layout (Cloudflare R2)

### Bucket: `permacite-captures`

```
permacite-captures/
├── screenshots/
│   └── {year}/{month}/{researcherId}/{captureId}.png
│       Retention: 5+ years  |  Average size: 150KB
├── html/
│   └── {year}/{month}/{researcherId}/{captureId}.html.gz
│       Retention: 5+ years  |  Average size: 50KB compressed
└── wacz/
    └── {year}/{month}/{researcherId}/{captureId}.wacz
        Retention: 5+ years  |  Average size: 500KB
```

### Storage Lifecycle Policy
```
Day 0–90:   R2 Standard (hot)         → ~$0.015/GB/month
Day 91–365: R2 Infrequent Access      → ~$0.01/GB/month
Day 365+:   AWS S3 Glacier via export → ~$0.004/GB/month
```

**GDPR Right to Erasure Handling:**
- When researcher requests deletion: mark capture as `soft_deleted` in PostgreSQL
- Set R2 object expiry to 30 days (hard delete after grace period)
- Citation record retained (researcher may have shared citations) with screenshot/HTML refs nulled
- Audit log entries are NEVER deleted (required for regulatory compliance)

---

## 4. Redis Data Structures

### Usage Counter Cache
```
Key: usage:{researcherId}:{yearMonth}
Type: Hash
Fields: { captures: int, exports: int, storageBytes: int }
TTL: 3600 seconds (1 hour)
Refresh: On CaptureCreated event (increment) + hourly reconcile from PostgreSQL
```

### Citation Format Cache
```
Key: citation_format:{citationId}:{style}
Type: String (JSON)
Value: { formattedText: "...", updatedAt: "..." }
TTL: 86400 seconds (24 hours)
Invalidate: On CitationApproved (metadata corrected)
```

### Session Token Cache
```
Key: session:{jti}
Type: String
Value: { researcherId, workspaceId, planTier, expiresAt }
TTL: Matches JWT expiry (15 minutes access, 7 days refresh)
```

---

## 5. Event Store Schema (Redis Streams)

### Stream: `permacite:events`

Each event follows CloudEvents spec:
```
XADD permacite:events * \
  specversion 1.0 \
  type permacite.capture.created \
  source /captures \
  id {uuid} \
  time {iso8601} \
  data {json_encoded_payload}
```

### Consumer Groups
```
Group: archival-service     → Consumes: CaptureCreated
Group: citation-service     → Consumes: ArchiveConfirmed, ArchiveDegraded
Group: library-service      → Consumes: CitationCreated
Group: integration-service  → Consumes: CitationExportReady
Group: billing-service      → Consumes: CaptureCreated, ExportSucceeded
```

Event retention: 24 hours (all downstream processing should complete within minutes; 24h is a safety buffer for recovery).

---

## 6. Data Access Patterns

### Hot Paths (latency-critical)

| Pattern | Query | Optimization |
|---------|-------|-------------|
| Load capture status (extension polling) | `SELECT status FROM captures WHERE capture_id = $1` | PK lookup, sub-1ms |
| Get citation for dashboard | JOIN citations + metadata + formatted (APA) | Covering index on citation_id |
| Search citations by text | `WHERE search_vector @@ to_tsquery($1)` | GIN index on tsvector |
| Check usage quota | Redis cache first, PostgreSQL fallback | Redis: < 1ms |

### Cold Paths (acceptable latency)

| Pattern | Query | Optimization |
|---------|-------|-------------|
| Bulk bibliography export | SELECT all citations in collection, format as BibTeX | Async job, streamed response |
| Archive health check | SELECT archives WHERE last_verified_at < 7 days ago | Scheduled pg_cron job |
| Storage tier migration | UPDATE storage_class, copy R2 objects | Nightly batch job |
| Usage period reconciliation | SUM captures in period, compare to Redis counter | Hourly scheduled task |

---

## 7. Data Consistency Model

### Within a Bounded Context: Strong Consistency
Each module's writes to its own PostgreSQL tables are ACID-transactional. Example: creating a Citation record and all its formatted variants happens in a single transaction.

### Across Bounded Contexts: Eventual Consistency
Cross-module communication is async (domain events). The pipeline is:
- Capture → (event, ~200ms) → Archive → (event, ~3-5s) → Citation

**Compensating Transactions:**
If `ArchiveFailed` is published, the Capture status is updated to `FAILED`. No citation is created. The researcher is notified. They can retry the capture.

If a Citation is created but the export job fails, the citation remains valid in the library — only the export is retried (no rollback of citation).

### Idempotency
All event handlers are idempotent. Processing a `CaptureCreated` event twice should not create two archives. Each handler checks: "does an Archive record already exist for this captureId?" before creating one.

```sql
-- Idempotency check in archive creation
INSERT INTO archival.archives (capture_id, ...)
ON CONFLICT (capture_id) DO NOTHING;
```

---

*Schema is managed via Drizzle ORM migrations. All migrations are forward-only (no rollback scripts — only compensating migrations). Schema changes require a review from two engineers once the team grows past 2 people.*
