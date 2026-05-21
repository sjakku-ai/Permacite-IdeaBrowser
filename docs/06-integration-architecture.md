# Permacite — Integration Architecture
## External Service Contracts, Patterns & Failure Handling · May 2026

---

## 1. Integration Philosophy

Permacite is fundamentally an **integration product** — it sits between a researcher's browser and their existing research toolchain. Every external integration must be:

1. **Resilient** — A Zotero API outage must not break the capture → citation pipeline
2. **Isolated** — Failures in one connector must not affect others
3. **Transparent** — Researchers must always know the status of their exports
4. **Opt-in** — Citations are complete without any integration; integrations are additive

The Integration Context uses the **Ports and Adapters (Hexagonal Architecture)** pattern — every external service is behind an interface that the domain never calls directly.

---

## 2. Integration Map

```
                              Permacite Core
                                    │
        ┌───────────────────────────┼────────────────────────────────┐
        │               │           │           │                    │
        ▼               ▼           ▼           ▼                    ▼
┌─────────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  ┌────────────────────┐
│  Internet   │  │ Anthropic│  │ Zotero │  │ Notion │  │     Stripe         │
│  Archive    │  │  Claude  │  │  API   │  │  API   │  │   Billing API      │
│  Wayback    │  │ (OpenAI  │  │ v3     │  │ v1     │  └────────────────────┘
│  Save API   │  │ fallback)│  └────────┘  └────────┘
└─────────────┘  └──────────┘
        │               │                                ┌────────────────────┐
        │               │                                │     Clerk          │
        │               │  ┌──────────────┐              │   Auth API         │
        │               │  │  CrossRef    │              └────────────────────┘
        │               │  │  PubMed      │
        │               │  │  OpenLibrary │              ┌────────────────────┐
        │               │  └──────────────┘              │    Obsidian        │
        │               │                                │  Local REST API    │
        │               │                                └────────────────────┘
        │               │
        ▼               ▼
  [Archival Context]  [Citation Context]
```

---

## 3. Internet Archive (Wayback Machine) Integration

### 3.1 Overview

The Wayback Machine is Permacite's **primary archival provider**. It provides institutional credibility that a proprietary archive cannot — an archived URL on `web.archive.org` is universally trusted by academics and legal scholars.

### 3.2 API Contract

```http
# Save a URL to the Wayback Machine
POST https://web.archive.org/save/{url}
Authorization: LOW {ia_s3_access_key}:{ia_s3_secret_key}
Content-Type: application/x-www-form-urlencoded

# Response: 200 OK
{
  "url": "https://example.com/article",
  "job_id": "spn2-a1b2c3d4..."
}

# Check save status
GET https://web.archive.org/save/status/{job_id}
# Response:
{
  "status": "success",
  "original_url": "https://example.com/article",
  "timestamp": "20260522143022",
  "duration_sec": 4.2
}
# Permanent URL: https://web.archive.org/web/20260522143022/https://example.com/article
```

### 3.3 Rate Limits & Quotas

| Account Type | Rate Limit | Daily Limit |
|-------------|-----------|-------------|
| Anonymous | ~5/min | ~100/day |
| S3-like key (free) | ~15/min | ~5,000/day |
| Partner account | Negotiable | Negotiable |

**Strategy:** Free S3-like keys are sufficient for MVP (5K captures/day = ~150K/month). At scale, contact Internet Archive for a partner relationship (they actively support academic tools).

### 3.4 Wayback Machine Adapter

```typescript
class WaybackMachineClient {
  private readonly BASE_URL = 'https://web.archive.org';
  private readonly POLL_INTERVAL_MS = 1000;
  private readonly MAX_POLL_ATTEMPTS = 15; // 15 seconds max wait

  async save(url: string): Promise<WaybackSaveResult> {
    const response = await this.httpClient.post(
      `${this.BASE_URL}/save/${encodeURIComponent(url)}`,
      { headers: { Authorization: `LOW ${this.apiKey}` } }
    );
    const jobId = response.data.job_id;
    return await this.pollForCompletion(jobId);
  }

  private async pollForCompletion(jobId: string): Promise<WaybackSaveResult> {
    for (let attempt = 0; attempt < this.MAX_POLL_ATTEMPTS; attempt++) {
      await sleep(this.POLL_INTERVAL_MS);
      const status = await this.checkStatus(jobId);
      if (status.status === 'success') {
        return {
          success: true,
          permanentUrl: `${this.BASE_URL}/web/${status.timestamp}/${status.original_url}`,
          archivedAt: parseTimestamp(status.timestamp)
        };
      }
      if (status.status === 'error') {
        return { success: false, reason: status.message };
      }
    }
    return { success: false, reason: 'TIMEOUT' };
  }
}
```

### 3.5 Fallback Strategy

When Wayback Machine fails (timeout, rate limit, server error):
1. Local WACZ capture via Puppeteer/`@webrecorder/warcio`
2. WACZ stored in Cloudflare R2 with content hash key
3. `ArchiveDegraded` event published (citation proceeds with local archive)
4. Background retry job attempts Wayback Machine re-submission after 1 hour
5. If Wayback succeeds on retry, update Archive record with `primary_archive_url`, demote WACZ to fallback

---

## 4. AI Metadata Extraction — Anthropic Claude

### 4.1 Overview

Claude is called by the Citation Context to extract structured metadata from a capture's screenshot and HTML. This is the **most latency-sensitive** external call in the pipeline.

### 4.2 API Contract

```typescript
// Anthropic Messages API
POST https://api.anthropic.com/v1/messages
x-api-key: {ANTHROPIC_API_KEY}
anthropic-version: 2023-06-01
Content-Type: application/json

{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "messages": [{
    "role": "user",
    "content": [
      {
        "type": "image",
        "source": {
          "type": "base64",
          "media_type": "image/png",
          "data": "{screenshot_base64}"
        }
      },
      {
        "type": "text",
        "text": "{extraction_prompt} + {html_excerpt} + {url}"
      }
    ]
  }]
}
```

### 4.3 Extraction Prompt (Production Version)

```
You are a precise academic citation metadata extractor.
Extract citation metadata from the provided web page screenshot and HTML excerpt.

Return ONLY a valid JSON object. No prose, no explanation.

Schema:
{
  "title": { "value": string | null, "confidence": number },
  "authors": { "value": Array<{ "last": string, "first": string, "orcid": string | null }>, "confidence": number },
  "publicationDate": { "value": string | null, "confidence": number },  // ISO 8601
  "publisher": { "value": string | null, "confidence": number },
  "journalName": { "value": string | null, "confidence": number },
  "volume": { "value": string | null, "confidence": number },
  "issue": { "value": string | null, "confidence": number },
  "pages": { "value": string | null, "confidence": number },
  "doi": { "value": string | null, "confidence": number },
  "isbn": { "value": string | null, "confidence": number },
  "sourceType": { "value": "journal-article" | "book" | "website" | "report" | "other", "confidence": number }
}

Rules:
- Confidence is 0.0–1.0. Use 0.0 if the field is not present.
- Return null for any field not clearly visible on the page.
- Do not hallucinate author names or DOIs.
- If a DOI is present as a URL (https://doi.org/10.xxx), extract just the DOI identifier.
- For websites without a clear author, return empty array for authors with confidence 0.3.

Page URL: {url}
HTML excerpt (first 5000 chars): {html_excerpt}
```

### 4.4 Circuit Breaker & Fallback

```typescript
class MetadataExtractionService {
  private claudeCircuitBreaker = new CircuitBreaker({
    threshold: 3,        // Open after 3 consecutive failures
    timeout: 30_000,     // 30 seconds
    resetTimeout: 60_000 // Try again after 1 minute
  });

  async extract(screenshot: Buffer, html: string, url: string): Promise<ExtractionResult> {
    try {
      if (this.claudeCircuitBreaker.isOpen()) {
        throw new CircuitOpenError('Claude circuit open');
      }
      const result = await this.claudeClient.extract(screenshot, html, url);
      this.claudeCircuitBreaker.recordSuccess();
      return result;
    } catch (error) {
      this.claudeCircuitBreaker.recordFailure();
      // Fall back to GPT-4o
      return await this.openAIClient.extract(screenshot, html, url);
    }
  }
}
```

### 4.5 Cost Control

- HTML is truncated to first 5,000 characters (sufficient for metadata in page `<head>` and above-fold content)
- Screenshot is resized to max 1024×768 before base64 encoding (reduces input tokens by ~40%)
- Results are cached in PostgreSQL — re-formatting in a different citation style reuses stored metadata (no AI call)
- Batch processing: if the same DOI is captured by multiple users, the metadata is shared (deduplicated by DOI + URL hash)

---

## 5. CrossRef & PubMed Validation

### 5.1 CrossRef API

Used to validate and enrich citation metadata when a DOI is detected.

```http
GET https://api.crossref.org/works/{doi}
User-Agent: Permacite/1.0 (mailto:support@permacite.com)

Response (simplified):
{
  "message": {
    "title": ["Deep Learning for Citation Extraction"],
    "author": [{ "given": "Jane", "family": "Smith" }],
    "published": { "date-parts": [[2025, 3, 15]] },
    "publisher": "Nature Publishing Group",
    "container-title": ["Nature Machine Intelligence"],
    "DOI": "10.1038/s42256-025-00123-4",
    "ISSN": ["2522-5839"]
  }
}
```

**Validation Logic:**
- If AI-extracted title has > 0.9 similarity to CrossRef title → boost confidence to 0.99
- If AI-extracted authors differ significantly from CrossRef → flag for human review
- CrossRef data always takes precedence for DOI, ISSN, publisher

### 5.2 PubMed API (for medical/life sciences citations)

```http
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi
  ?db=pubmed&term={title}+{author}&retmode=json&tool=permacite&email=support@permacite.com

GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi
  ?db=pubmed&id={pmid}&retmode=json
```

Both CrossRef and PubMed are **free** with polite usage (User-Agent header with contact email).

---

## 6. Zotero Integration

### 6.1 Overview

Zotero is Permacite's most important integration — it has ~7.5M users and is the incumbent tool for the academic audience. Permacite pushes citations INTO Zotero, turning the incumbent into a distribution channel.

### 6.2 OAuth Flow

Zotero uses OAuth 1.0a:

```
1. Researcher clicks "Connect Zotero" in Permacite settings
2. Permacite backend calls:
   POST https://www.zotero.org/oauth/request
   → Returns oauth_token + oauth_token_secret
3. Researcher redirected to:
   https://www.zotero.org/oauth/authorize?oauth_token={token}
4. Researcher approves → Zotero redirects to:
   https://permacite.com/integrations/zotero/callback?oauth_token={token}&oauth_verifier={verifier}
5. Permacite exchanges for access token:
   POST https://www.zotero.org/oauth/access
   → Returns userID + oauth_access_token + oauth_access_token_secret
6. Tokens encrypted (AES-256-GCM) and stored in integration.connectors
```

### 6.3 Export API Call

```typescript
// Create item in Zotero via Web API v3
async exportToZotero(citation: FormattedCitation, creds: ZoteroCredentials): Promise<ZoteroExportResult> {
  const item = this.mapToZoteroItem(citation);
  
  const response = await fetch(
    `https://api.zotero.org/users/${creds.userId}/items`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${creds.accessToken}`,
        'Zotero-API-Version': '3',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify([item])
    }
  );
  
  // Zotero returns item keys for created items
  const result = await response.json();
  return { success: true, zoteroItemKey: result.successful[0].key };
}

// Map Permacite citation to Zotero item format
mapToZoteroItem(citation: FormattedCitation): ZoteroItem {
  return {
    itemType: this.mapSourceType(citation.metadata.sourceType), // "journalArticle", "book", etc.
    title: citation.metadata.title,
    creators: citation.metadata.authors.map(a => ({
      creatorType: 'author',
      lastName: a.last,
      firstName: a.first
    })),
    date: citation.metadata.publicationDate,
    DOI: citation.metadata.doi,
    url: citation.permanentLink,           // Permanent archive URL, not original
    extra: `Original URL: ${citation.sourceUrl}\nArchive URL: ${citation.permanentLink}`,
    tags: [{ tag: 'permacite-captured' }]  // Enables filtering in Zotero
  };
}
```

---

## 7. Notion Integration

### 7.1 OAuth Flow

Notion uses OAuth 2.0:

```
1. Researcher clicks "Connect Notion"
2. Redirect to: https://api.notion.com/v1/oauth/authorize
   ?client_id={CLIENT_ID}&redirect_uri={CALLBACK}&response_type=code
3. Researcher selects workspace + pages to grant access
4. Callback: POST /integrations/notion/callback
5. Exchange code: POST https://api.notion.com/v1/oauth/token
   → Returns access_token + workspace_id + bot_id
```

### 7.2 Export to Notion Database

Permacite creates entries in a researcher's Notion citation database. On first connect, Permacite creates a template database with the right schema; researchers can also point to an existing database.

```typescript
async exportToNotion(citation: FormattedCitation, settings: NotionSettings): Promise<void> {
  const properties = {
    'Title': { title: [{ text: { content: citation.metadata.title } }] },
    'Authors': { rich_text: [{ text: { content: citation.metadata.authors.map(a => `${a.last}, ${a.first}`).join('; ') } }] },
    'Publication Date': { date: { start: citation.metadata.publicationDate } },
    'DOI': { url: citation.metadata.doi ? `https://doi.org/${citation.metadata.doi}` : null },
    'Permanent Link': { url: citation.permanentLink },
    'APA Citation': { rich_text: [{ text: { content: citation.formattedVariants.APA_7 } }] },
    'BibTeX': { rich_text: [{ text: { content: citation.formattedVariants.BIBTEX } }] },
    'Source Type': { select: { name: citation.metadata.sourceType } },
    'Captured': { date: { start: citation.capturedAt.toISOString() } }
  };

  await this.notionClient.pages.create({
    parent: { database_id: settings.databaseId },
    properties
  });
}
```

---

## 8. Obsidian Integration

### 8.1 Overview

Obsidian is a local-first tool — it stores notes as markdown files on the researcher's computer. There is no cloud API. Integration uses the **Obsidian Local REST API** community plugin.

The researcher installs the plugin in Obsidian, which starts a local HTTP server on port 27123. Permacite's browser extension communicates with this local server directly (no cloud round-trip).

### 8.2 Integration Pattern

```
Browser Extension → localhost:27123 (Obsidian REST API) → Obsidian Vault
```

This is a **browser extension → localhost** pattern, not a server-side integration. The extension handles Obsidian export, not the Permacite backend.

```typescript
// In browser extension: create markdown note in Obsidian
async exportToObsidian(citation: FormattedCitation, settings: ObsidianSettings): Promise<void> {
  const noteContent = this.buildObsidianNote(citation);
  const fileName = this.sanitizeFileName(`${citation.metadata.authors[0]?.last || 'Unknown'}_${citation.metadata.publicationDate?.slice(0,4) || 'nd'}_${citation.metadata.title?.slice(0,50) || 'Untitled'}.md`);

  await fetch(`http://localhost:27123/vault/${settings.citationFolder}/${fileName}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${settings.apiKey}`,
      'Content-Type': 'text/markdown'
    },
    body: noteContent
  });
}

buildObsidianNote(citation: FormattedCitation): string {
  return `---
title: "${citation.metadata.title}"
authors: [${citation.metadata.authors.map(a => `"${a.last}, ${a.first}"`).join(', ')}]
year: ${citation.metadata.publicationDate?.slice(0,4)}
doi: "${citation.metadata.doi || ''}"
permanent_link: "${citation.permanentLink}"
source_type: ${citation.metadata.sourceType}
captured: ${citation.capturedAt.toISOString()}
tags: [citation, permacite]
---

# ${citation.metadata.title}

## APA Citation
${citation.formattedVariants.APA_7}

## BibTeX
\`\`\`bibtex
${citation.formattedVariants.BIBTEX}
\`\`\`

## Highlighted Quote
${citation.highlightedQuote ? `> ${citation.highlightedQuote.text}` : '_No quote captured_'}

## Archived Source
[View Archived Page](${citation.permanentLink})
`;
}
```

---

## 9. Stripe Billing Integration

### 9.1 Webhook Events

```typescript
// Stripe webhook handler maps to Billing Context domain events
const STRIPE_EVENT_MAP = {
  'customer.subscription.created':    'SubscriptionCreated',
  'customer.subscription.updated':    'SubscriptionUpdated',
  'customer.subscription.deleted':    'SubscriptionCancelled',
  'invoice.payment_succeeded':        'PaymentSucceeded',
  'invoice.payment_failed':           'PaymentFailed',
  'customer.subscription.trial_will_end': 'TrialEnding',
};

// All webhooks verified with Stripe webhook secret before processing
async handleStripeWebhook(payload: Buffer, signature: string): Promise<void> {
  const event = stripe.webhooks.constructEvent(payload, signature, WEBHOOK_SECRET);
  const domainEvent = this.mapToDomainEvent(event);
  await this.eventBus.publish(domainEvent);
}
```

### 9.2 Metered Usage Reporting

For pay-as-you-go or usage-based plans:
```typescript
// Report usage to Stripe at end of billing period
await stripe.subscriptionItems.createUsageRecord(subscriptionItemId, {
  quantity: usage.capturesUsed,
  timestamp: Math.floor(Date.now() / 1000),
  action: 'set'  // Not increment — we report the total
});
```

---

## 10. Clerk Authentication Integration

### 10.1 JWT Verification Middleware

```typescript
// Fastify middleware: verify Clerk JWT on every request
async function clerkAuthMiddleware(request: FastifyRequest, reply: FastifyReply) {
  const { sessionClaims } = await clerkClient.verifyToken(
    request.headers.authorization?.replace('Bearer ', '') ?? ''
  );

  // Inject into request context (no framework dependency in domain)
  request.researcherId = sessionClaims.sub;
  request.workspaceId = sessionClaims.org_id;
  request.planTier = sessionClaims.publicMetadata?.planTier ?? 'FREE';
}
```

### 10.2 ORCID SSO

ORCID is the academic research identity standard. Clerk supports ORCID as a social OAuth provider:
- Researchers log in with their ORCID iD
- ORCID name and affiliation pre-populate the researcher profile
- ORCID iD stored in researcher profile for potential citation authority use

---

## 11. Integration Resilience Patterns

### Retry Policy (Export Jobs)

```
Attempt 1: Immediate
Attempt 2: 1 second delay
Attempt 3: 5 seconds delay
Attempt 4: 30 seconds delay
After 4 failures: Dead-lettered → researcher notified
```

### Circuit Breaker States

```
CLOSED (normal) → OPEN (failing) → HALF-OPEN (testing recovery) → CLOSED

Threshold to OPEN: 3 consecutive failures in 60s window
Time in OPEN state: 60 seconds
Test in HALF-OPEN: 1 request — if success → CLOSED, if failure → OPEN again
```

### Dead Letter Queue

Dead-lettered export jobs are stored in PostgreSQL (`status = 'DEAD_LETTERED'`). A dashboard widget shows researchers their failed exports with a one-click "Retry All" button. Retrying creates new export jobs from the same citation payload.

---

## 12. Integration Security

| Concern | Implementation |
|---------|---------------|
| OAuth token storage | AES-256-GCM encrypted at rest in `integration.connectors.encrypted_creds` |
| Encryption key management | AWS KMS (key rotation every 90 days) |
| Webhook signature verification | Stripe: HMAC-SHA256; Clerk: RS256 JWT; Internet Archive: no signing (IP allowlist) |
| API key rotation | Users can rotate integration credentials without losing export history |
| Credential scope | Zotero: read/write items only; Notion: only databases explicitly granted; Obsidian: local only |
| SSRF prevention | All outbound HTTP goes through a proxy allowlist; researcher-controlled URLs (Obsidian localhost) are explicitly allowlisted in extension CSP |

---

*Integration contracts change frequently. The adapters in `modules/integration/infrastructure/` are the sole places where external API structures are known. The domain layer never knows what Zotero or Notion look like.*
