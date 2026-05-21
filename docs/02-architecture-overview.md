# Permacite — Architecture Overview & Context Map
## DDD Strategic Design · May 2026

---

## 1. Architectural Philosophy

Permacite is built on **Domain-Driven Design (DDD)** strategic and tactical patterns. The architecture reflects three guiding principles:

1. **Core Domain First** — Archival + AI extraction is the moat. All architectural decisions prioritize these over generic concerns.
2. **Eventual Consistency as a Feature** — The capture → archive → citation pipeline is inherently async. Embrace event-driven patterns rather than fight them.
3. **Integration Over Replacement** — Zotero, Notion, and Obsidian are distribution channels, not competitors. The architecture is designed to push INTO them, not pull users away.

---

## 2. Architectural Style

Permacite uses a **Modular Monolith with Event-Driven Seams** for MVP, designed to decompose into independent services as each bounded context scales independently.

### Why Modular Monolith (Not Microservices) at Launch

| Factor | Rationale |
|--------|-----------|
| Team size | Solo founder / micro-team → microservices add ops overhead with no benefit |
| Domain maturity | Bounded context boundaries are clear in theory but will shift early; monolith boundaries are cheaper to move |
| Latency | In-process calls between Citation and Archival reduce capture-to-citation latency well below the 10-second SLA |
| Deployment simplicity | Single deployment unit → faster iteration on a 1–2 week MVP timeline |
| Decomposition path | Event-driven seams (domain events) allow clean extraction of any context into a standalone service later |

### Evolution Path

```
Phase 1 (MVP)           Phase 2 (Scale)          Phase 3 (Enterprise)
─────────────────       ────────────────────      ─────────────────────────
Modular Monolith    →   Extract Archival &     →  Full Service Mesh
+ Message Bus           Citation as services       Per-context autoscaling
+ Serverless Edge       Keep Identity/Billing      CQRS + Event Sourcing
  (browser ext)         as shared services         in Citation Context
```

---

## 3. High-Level Architecture Diagram

```
                          ┌──────────────────────────────┐
                          │       Browser Extension       │
                          │  (Chrome/Firefox, TypeScript) │
                          └──────────────┬───────────────┘
                                         │ HTTPS / WebSocket
                                         ▼
                          ┌──────────────────────────────┐
                          │        API Gateway            │
                          │  (Cloudflare / AWS API GW)    │
                          │  Rate limiting, auth, routing │
                          └──────┬───────────────┬────────┘
                                 │               │
                    ┌────────────▼──┐     ┌──────▼──────────┐
                    │  Web App      │     │  Core API        │
                    │  (Next.js)    │     │  (Fastify/NestJS) │
                    │  Dashboard    │     │  REST + Events    │
                    └───────────────┘     └──────┬──────────┘
                                                 │
                    ┌────────────────────────────▼──────────────────────────┐
                    │                    Application Core                    │
                    │                  (Modular Monolith)                   │
                    │                                                        │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐│
                    │  │ Capture  │  │ Archival │  │     Citation         ││
                    │  │ Module   │  │ Module   │  │     Module           ││
                    │  └────┬─────┘  └────┬─────┘  └──────────┬───────────┘│
                    │       │             │                    │            │
                    │  ┌────▼─────┐  ┌────▼──────┐  ┌────────▼──────────┐ │
                    │  │ Library  │  │Integration│  │  Identity &       │ │
                    │  │ Module   │  │ Module    │  │  Billing Modules  │ │
                    │  └──────────┘  └───────────┘  └───────────────────┘ │
                    │                                                        │
                    │              Internal Event Bus                        │
                    │         (In-process / Redis Streams)                  │
                    └────────────────────────────────────────────────────────┘
                                         │
                    ┌────────────────────▼──────────────────────────────────┐
                    │                   Infrastructure Layer                  │
                    │                                                         │
                    │  ┌──────────┐  ┌───────────┐  ┌──────────────────┐   │
                    │  │PostgreSQL│  │  S3 / R2  │  │  Redis (cache +  │   │
                    │  │(primary  │  │  (archive  │  │  event bus)      │   │
                    │  │ store)   │  │  storage)  │  └──────────────────┘   │
                    │  └──────────┘  └───────────┘                          │
                    └────────────────────────────────────────────────────────┘
                                         │
                    ┌────────────────────▼──────────────────────────────────┐
                    │                   External Services                    │
                    │                                                         │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
                    │  │ Wayback  │  │ OpenAI / │  │ Zotero / Notion  │    │
                    │  │ Machine  │  │ Anthropic │  │ / Obsidian APIs  │    │
                    │  │   API    │  │   APIs   │  └──────────────────┘    │
                    │  └──────────┘  └──────────┘                           │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
                    │  │CrossRef/ │  │  Stripe  │  │  Auth0 / Clerk   │    │
                    │  │ PubMed   │  │ Billing  │  │  (Identity)      │    │
                    │  └──────────┘  └──────────┘  └──────────────────┘    │
                    └────────────────────────────────────────────────────────┘
```

---

## 4. Layered Architecture (Per Bounded Context)

Each bounded context follows a strict four-layer architecture:

```
┌──────────────────────────────────────────────────────┐
│                  Interface Layer                      │
│  REST Controllers, Event Handlers, CLI Commands      │
│  Translates external requests to Application cmds    │
├──────────────────────────────────────────────────────┤
│                 Application Layer                     │
│  Command Handlers, Query Handlers, Event Handlers    │
│  Orchestrates domain objects — no business logic     │
├──────────────────────────────────────────────────────┤
│                   Domain Layer                        │
│  Aggregates, Entities, Value Objects, Domain Events  │
│  Domain Services, Repository Interfaces              │
│  *** NO infrastructure dependencies ***              │
├──────────────────────────────────────────────────────┤
│                Infrastructure Layer                   │
│  Repository Implementations, External API clients    │
│  Message Bus adapters, Database ORM, S3 clients      │
└──────────────────────────────────────────────────────┘
```

**Critical Rule:** The Domain Layer has zero dependencies on any framework, database, or external service. It only knows about its own aggregates and uses interfaces (ports) for everything external.

---

## 5. CQRS Application (Citation Context)

The Citation Context uses **CQRS (Command Query Responsibility Segregation)** because:
- Citation formatting is computationally expensive (AI calls)
- Read patterns (search, filter, export) are high-frequency and diverse
- Write patterns (capture → extract → format) are sequential and async

```
                        Citation Context
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   COMMANDS                         QUERIES                  │
│   ──────────────────                ─────────────────────── │
│   FormatCitation                    GetCitationById          │
│   ReviewCitation                    SearchCitations          │
│   ApproveCitation                   GetBulkBibliography      │
│   RegenerateCitation                GetLibraryExport         │
│                                                             │
│   ┌──────────────────┐             ┌──────────────────────┐ │
│   │  Command Handler │             │   Query Handler      │ │
│   │  → Domain Logic  │             │   → Read Model       │ │
│   │  → Write DB      │             │   → Search Index     │ │
│   └──────────────────┘             └──────────────────────┘ │
│            │                                  │             │
│     ┌──────▼──────┐                ┌──────────▼──────────┐  │
│     │  PostgreSQL  │                │  PostgreSQL (read)   │  │
│     │  (write)     │                │  + Full-text search  │  │
│     └─────────────┘                └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Event-Driven Pipeline (Core Domain Flow)

The capture-to-citation pipeline is the **critical path**. Every step is event-driven with compensating transactions for failures.

```
1. Researcher clicks extension
        │
        ▼
2. Extension sends POST /captures
        │
        ▼
3. Capture Module creates Capture aggregate
   → Saves screenshot + HTML to S3/R2
   → Publishes CaptureCreated event
        │
        ▼ (async, ~200ms)
4. Archival Module handles CaptureCreated
   → Submits URL to Wayback Machine API
   → If fails: local WACZ fallback
   → Publishes ArchiveConfirmed | ArchiveFailed
        │
        ▼ (async, ~2-5s)
5. Citation Module handles ArchiveConfirmed
   → Calls multimodal AI API with screenshot + HTML
   → Extracts CitationMetadata
   → Cross-validates with DOI/PubMed
   → Formats in all requested styles
   → Scores confidence
   → If score < 0.85: flags for review
   → Publishes CitationCreated
        │
        ▼ (async, ~100ms)
6. Library Module handles CitationCreated
   → Adds LibraryItem to default collection
   → Updates full-text search index

7. Integration Module handles CitationExportReady
   → Pushes to Zotero / Notion / Obsidian
   → Retry with exponential backoff
   → Dead-letter after 3 failures

End-to-end target: < 10 seconds (P95)
```

---

## 7. Context Map (Strategic Relationships)

```
                    ┌────────────────────────────────────┐
                    │         CORE DOMAIN                │
                    │                                    │
                    │   ┌──────────┐   ┌─────────────┐  │
                    │   │ Capture  │──▶│  Archival   │  │
                    │   │ Context  │   │  Context    │  │
                    │   └──────────┘   └──────┬──────┘  │
                    │                         │         │
                    │                         ▼         │
                    │              ┌───────────────────┐ │
                    │              │  Citation Context │ │
                    │              └─────────┬─────────┘ │
                    └────────────────────────┼───────────┘
                                             │
                    ┌────────────────────────▼───────────┐
                    │       SUPPORTING DOMAIN            │
                    │                                    │
                    │  ┌──────────┐   ┌──────────────┐  │
                    │  │ Library  │   │ Integration  │  │
                    │  │ Context  │   │ Context      │  │
                    │  └──────────┘   └──────────────┘  │
                    └────────────────────────────────────┘
                                             │
                    ┌────────────────────────▼───────────┐
                    │        GENERIC SUBDOMAINS          │
                    │                                    │
                    │  ┌──────────┐   ┌──────────────┐  │
                    │  │Identity &│   │  Billing &   │  │
                    │  │ Access   │   │  Subscript.  │  │
                    │  └──────────┘   └──────────────┘  │
                    └────────────────────────────────────┘
```

### Context Relationship Types

| Relationship | Type | Description |
|-------------|------|-------------|
| Capture ↔ Archival | **Customer/Supplier** | Capture (upstream) provides captures; Archival (downstream) processes them. Capture defines the contract. |
| Archival ↔ Citation | **Customer/Supplier** | Archival (upstream) provides archive references; Citation (downstream) enriches them. |
| Citation ↔ Library | **Published Language** | Citation publishes a standard event that Library consumes without coupling. |
| Citation ↔ Integration | **Published Language** | Standard export payload; Integration adapts to each external tool. |
| Identity → All | **Shared Kernel** | ResearcherId and permission model are shared. Changes require joint agreement. |
| Billing → All | **Anti-Corruption Layer** | Billing limits translated at API gateway middleware; core domain never checks billing directly. |
| Integration ↔ External Tools | **Conformist** | Permacite conforms to Zotero/Notion/Obsidian API contracts. They set the standard. |

---

## 8. Key Architectural Decisions (ADRs)

### ADR-001: Modular Monolith over Microservices at Launch
- **Decision:** Single deployable with logical module separation
- **Rationale:** Team size, iteration speed, domain boundary uncertainty
- **Consequence:** Must enforce module boundaries strictly via import linting; async events even in-process

### ADR-002: Event-Driven Seams Between All Bounded Contexts
- **Decision:** All cross-context communication via domain events (no direct function calls across modules)
- **Rationale:** Enables future decomposition without code changes; explicit dependency graph
- **Consequence:** Additional complexity for developers unfamiliar with event-driven patterns

### ADR-003: CQRS in Citation Context Only
- **Decision:** Apply CQRS only to Citation; other contexts use simple CRUD
- **Rationale:** Citation has the most asymmetric read/write patterns and performance requirements; premature CQRS elsewhere adds complexity without benefit
- **Consequence:** Separate read models maintained for Citation; eventual consistency on search index

### ADR-004: AI Extraction in Citation Context, Not as a Shared Service
- **Decision:** AI metadata extraction is a domain service inside Citation, not a separate AI microservice
- **Rationale:** AI extraction logic is tightly coupled to CitationMetadata domain objects; extracting it would create an anemic domain model
- **Consequence:** AI provider (OpenAI/Anthropic) coupling is inside one bounded context

### ADR-005: Wayback Machine as Primary Archive, Local WACZ as Fallback
- **Decision:** Always attempt Wayback Machine first; fall back to local storage only on failure
- **Rationale:** Wayback Machine provides institutional trust and independent permanence; local storage is under our control but less trusted by academia
- **Consequence:** Rate limiting and API availability become SLA dependencies

### ADR-006: Confidence Threshold as a Business Rule, Not Infrastructure
- **Decision:** 0.85 confidence threshold lives in the Citation domain model, not in AI API config
- **Rationale:** This is a business rule that may change by citation style (legal vs. academic), institution, or user preference
- **Consequence:** Must be configurable per tenant without code changes

---

## 9. Non-Functional Requirements & Architecture Responses

| NFR | Requirement | Architectural Response |
|-----|-------------|----------------------|
| **Latency** | End-to-end capture < 10s (P95) | Async pipeline; parallel archival + AI; CDN for extension assets |
| **Availability** | 99.9% uptime for capture API | Multi-region deployment; fallback capture path always available |
| **Scalability** | Support 1M monthly active users | Serverless compute (Vercel/Lambda); S3 for object storage; read replicas |
| **Data Durability** | Archives must survive indefinitely | Multi-region S3 replication; Wayback Machine as independent backup |
| **Compliance** | GDPR, FERPA, EU AI Act | Data residency options; audit logs; right-to-erasure (captures soft-deleted, citations retained) |
| **Security** | Sensitive research data | Encryption at rest and in transit; credential vault for integration tokens; zero-trust networking |
| **Observability** | Debug AI extraction failures | Structured logs per capture; tracing across pipeline; AI decision audit log |
| **Cost** | Margins at $15/mo | Tiered storage; AI batching; usage-based scaling; cold storage after 90 days |

---

*All architectural decisions in this document are subject to review as the domain matures. The context map is the primary instrument for resolving inter-team disagreements.*
