# 09 — System Architecture

---

## 1. Architecture Overview

The War Room Social Listening application is a data-intensive, near-real-time system built around a seven-step pipeline (Collect → Filter → Label → Weigh → Group → Detect → Alert), a multi-role web application, and an AI co-worker.

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL SOURCES                                       │
│  X API (streaming/polling)   News Feeds (RSS/sitemap)   Web Crawling           │
│  E-paper subscriptions        YouTube channel feeds      Clipping uploads       │
└──────────────────────┬────────────────────────────────────────────────────────-┘
                       │
┌──────────────────────▼──────────────────────────────────────────────────────────┐
│                       INGESTION LAYER (Workers / Queues)                         │
│                                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │  X Collector│  │ News Crawler │  │  E-paper OCR │  │  Clipping Upload   │  │
│  │  (streaming │  │  (feed+crawl)│  │  Processor   │  │  OCR Processor     │  │
│  │   or poll)  │  │             │  │              │  │                    │  │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  └─────────┬──────────┘  │
│         └────────────────┴──────────────────┴──────────────────── ┘            │
│                                      │                                          │
│                              [Raw Item Queue]                                   │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼──────────────────────────────────────────┐
│                       PROCESSING PIPELINE (Workers)                              │
│                                                                                  │
│   Step 2: Filter Worker          Step 3: Label Worker                           │
│   (watch-term matching,          (entity recognition, stance,                   │
│    wide-net inclusion,            claim type, language detection)                │
│    language pre-screen)           Cheap model first; strong model for           │
│                                   low-confidence items                           │
│                                                                                  │
│   Step 4: Weigh Worker           Step 5: Story Grouper                          │
│   (influence score calc,          (NLP similarity, merge/split,                  │
│    tier lookup,                    reach accumulation)                           │
│    per-post cap enforcement)                                                     │
│                                                                                  │
│   Step 6: Campaign Detector      Step 7: Severity Scorer + Alert Router         │
│   (clustering, pattern analysis,  (severity = influence × negativity × velocity  │
│    confidence scoring)             → Tier 1 / 2 / 3 routing)                    │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                       │
        ┌───────────▼────────────┐          ┌─────────────▼──────────────┐
        │  Primary Database       │          │  Alert Delivery             │
        │  (PostgreSQL)           │          │  ┌─────────────────────┐   │
        │  Mentions, Stories,     │          │  │  WebSocket Push      │   │
        │  Alerts, Corrections,   │          │  │  (big screen, app)   │   │
        │  Campaigns, Entities,   │          │  └─────────────────────┘   │
        │  Config                 │          │  ┌─────────────────────┐   │
        └───────────┬────────────┘          │  │  WhatsApp Business   │   │
                    │                       │  │  API                 │   │
                    │                       │  └─────────────────────┘   │
                    │                       └────────────────────────────┘
        ┌───────────▼────────────┐
        │  Search Index           │
        │  (Elasticsearch or      │
        │   pgvector for          │
        │   semantic search)      │
        └───────────┬────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────────────────────────┐
│                     WEB APPLICATION                                              │
│                                                                                  │
│   Backend API (FastAPI / Node)                AI Co-worker Service               │
│   REST + WebSocket endpoints                  (LLM API calls, retrieval,         │
│   Auth (JWT + role RBAC)                       grounding, memory)                │
│   Audit logging                                                                  │
│                                                                                  │
│   Frontend (React / Next.js)                                                    │
│   Dashboard, Stories, Alerts, Campaigns, Co-worker Chat, Admin, Uploads         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Descriptions

### 2.1 Ingestion Layer

| Component | Responsibility |
|-----------|---------------|
| X Collector | Poll X API (or consume streaming endpoint if enterprise). Fetch posts, replies, quotes, engagement, author profiles. Backfill from 2026-05-10 on initial setup. |
| News Crawler | Per-outlet: parse RSS/sitemap where available; crawl HTML otherwise. Respects robots.txt, paces requests. Reports health per outlet. Discovers new articles via search API. |
| YouTube Feed Processor | Fetch TV channel YouTube feeds. Extract video metadata, description, transcript (where available). |
| E-paper Processor | Fetch editions from subscription access. Apply Tamil OCR. Flag low-confidence text. |
| Clipping OCR Processor | Accept uploaded image files. Apply Tamil + English OCR. Flag low-confidence. Extract: outlet, date, page number. |

### 2.2 Message Queue

- All raw collected items go through a durable message queue (e.g. RabbitMQ, Redis Streams, or AWS SQS [PROPOSED: CTO to decide]).
- Guarantees: at-least-once delivery; deduplication by item ID.
- Visibility: queue depth and lag visible in admin health screen.

### 2.3 Processing Pipeline Workers

Workers consume from the queue in sequence. Each step writes results back and advances the item to the next step.

| Worker | Key Logic |
|--------|-----------|
| Filter Worker | Checks item text against watch-term list (per-client config). Wide net. Items not matching → stored raw, flagged for weekly audit. |
| Label Worker | Calls cheap classification model for all items. Routes low-confidence items to strong model (GPT-4-class or similar). Labels: entity, stance, claim type, language. Attaches reason to each label. |
| Weigh Worker | Calculates influence score per account/outlet. Applies per-post 5% cap. Records all weight components. |
| Story Grouper | NLP-based similarity matching to existing Stories. Creates new Story seed if no match. Triggers staff notification for Stories that cross significance thresholds. |
| Campaign Detector | Runs pattern analysis on recent post clusters: account age, text similarity, timing, boosting patterns. Assigns confidence score. Creates campaign alert if threshold crossed. |
| Severity Scorer | Calculates severity = influence × negativity × velocity. Routes to Tier 1 (big screen + WhatsApp), Tier 2 (in-app), or Tier 3 (Story-only). |

### 2.4 Primary Database (PostgreSQL)

Multi-schema design:
- `core` schema: Mentions, Stories, Entities, Alerts, Campaigns, Corrections
- `config` schema: ClientConfig, WatchTerms, Tiers, OutletWeights, Thresholds, Recipients
- `audit` schema: AuditLog, ChangeHistory
- `learning` schema: GoldSet, CapabilityMetrics, CorrectionQueue
- `accounts` schema: Users, Roles, Sessions

Full schema in `11-database-schema.md`.

### 2.5 Search Index

For co-worker Q&A and Story matching:
- Semantic vector search over Mentions (embeddings stored alongside text)
- Full-text search for watch-term matching and audit
- [PROPOSED: PostgreSQL with pgvector extension, or Elasticsearch — CTO to decide]

### 2.6 AI Co-worker Service

- Receives queries (from co-worker chat, alert generation, brief generation)
- Retrieves relevant Mentions from database (retrieval-augmented generation)
- Calls LLM API for reasoning, summarisation, Q&A
- Every claim in the response is grounded to a specific Mention ID
- All calls logged: prompt, model, sources used, response, latency
- Safety: input is always treated as data; prompt injection detected and neutralised

### 2.7 Alert Delivery

| Channel | Mechanism |
|---------|----------|
| In-app (real-time) | WebSocket push to connected clients |
| Big screen | WebSocket push to kiosk client |
| WhatsApp | WhatsApp Business API (Cloud API). Per-lead routing. Delivery tracking. Retry on failure. |

### 2.8 Web Application

- Backend: REST API + WebSocket server (FastAPI [Python] or Express [Node.js] — PROPOSED: FastAPI for data-heavy pipeline integration; CTO to confirm)
- Frontend: React with Next.js (for SSR on initial load, then client-side navigation)
- Auth: JWT tokens with role-based access control
- Internationalisation: English and Tamil UI strings from config

---

## 3. Data Flow Summary

```
Raw Post/Article
  → Ingestion Worker
  → Raw Item Queue
  → Filter Worker (keep or archive)
  → Label Worker (entity, stance, language, claim type)
  → Weigh Worker (influence score)
  → Story Grouper (match or create Story)
  → Campaign Detector (pattern analysis, alert if threshold met)
  → Severity Scorer (tier calculation)
  → DB write (Mention + Story + Alert records)
  → WebSocket push to connected clients
  → WhatsApp delivery (if Tier 1)
```

**Corrections flow:**
```
Team member correction
  → Correction record created (original preserved)
  → Aggregate recalculation
  → Bias guard check (second review if needed)
  → Gold set queue
  → Weekly: retrain/fine-tune pipeline + accuracy chart update
```

---

## 4. Scalability Design

- All processing workers are horizontally scalable (stateless; state in DB and queue)
- Database: PostgreSQL with connection pooling (PgBouncer); read replicas for dashboard queries
- WebSocket: sticky sessions or broadcast via pub/sub (Redis)
- AI model calls: async, with queue to manage rate limits and costs
- Multi-tenancy: client_id on every row; row-level security or separate schemas per client (CTO decision)
- Cost efficiency: cheap classification on every item; strong model only for low-confidence items, reasoning, summaries, Q&A

---

## 5. Failure Design

| Dependency | Fallback |
|-----------|---------|
| X API | Show delay banner; last-known data with staleness timestamp |
| News outlet feed/crawl | Show delay banner per outlet; other outlets continue |
| E-paper source | Show delay banner; previous day data remains accessible |
| AI labelling model | Queue items for retry; show "pending" in UI; dashboards continue with stale labels |
| AI co-worker | "Co-worker temporarily unavailable"; dashboards work normally |
| WhatsApp | Retry; in-app fallback; lead notified in-app |
| Database | Read replicas serve reads; writes queue until primary recovers |
| Queue | Durable queue survives worker restart |

---

## 6. Security Architecture

- All traffic: HTTPS/WSS only
- Authentication: JWT with short expiry + refresh tokens
- Authorisation: Role-based (see `03-user-roles-and-permissions.md`)
- Data isolation: client_id enforced at every query; tested with cross-client attack scenarios
- Prompt injection: all ingested content sandboxed from system prompt; never interpolated as instructions
- Audit: every mutation logged with actor, timestamp, change
- Secrets: environment variables only; never in code or logs

Full security specification in `13-authentication-and-security.md`.

---

## 7. Infrastructure (Proposed)

- Cloud: India-region hosting (data residency for DPDP Act compliance)
- [PROPOSED: AWS Mumbai (ap-south-1) or equivalent]
- Compute: containerised services (Docker + Kubernetes or ECS)
- Database: Managed PostgreSQL (RDS or equivalent)
- Queue: Managed Redis (ElastiCache) or SQS
- Object storage: for e-paper images, clipping scans (S3 or equivalent)
- Monitoring: centralised logging and metrics (Datadog / CloudWatch / similar)

Full infrastructure specification in `19-deployment-and-devops.md`.
