# 14 — Integrations and Dependencies

**Reads with:** 09 (architecture), 10 (stack), 13 (security), `DOC-RECONCILIATION.md` (binding addenda).
**Rule for every integration:** it sits behind an **interface in `backend/app/integrations/<name>/base.py`**, has a **health record** in `ops.source_health`, has a **visible fallback**, and a **failure never looks like "no new items"** (PRD p.20; FR-I28).

Items marked `[VERIFY]` depend on third-party terms or APIs that change; check the provider's current documentation when the task starts. Items marked `[PROPOSED]` are not in the PRD.

---

## 1. Dependency map

| # | Dependency | Used for | Blocks which stories | Who supplies access | Phase-1 status |
|---|-----------|----------|----------------------|---------------------|----------------|
| I-01 | X API (or licensed vendor) | Collect X posts, profiles, TVK-response detection | 1, 5, 9, 10, 13, 31, 32 | Client/CTO (contract) | 🔴 access tier undecided (G-01) |
| I-02 | News feeds (RSS/Atom/sitemap) | Collect articles | 15, 19, 33 | Public | ✅ |
| I-03 | News site crawler | Collect where no feed | 15, 19 | Public, robots.txt-bound | ✅ build |
| I-04 | Search-discovery API | Find stories on untracked sites | 15 | Vendor key | 🟡 optional, off by default |
| I-05 | YouTube Data API | TV channel video feeds | 33 | Google API key | 🟡 transcript limits (G-20) |
| I-06 | E-paper subscriptions | Collect print editions | 17 | Client | 🔴 terms unknown (G-02) |
| I-07 | OCR engine | Clippings, e-paper | 16, 17 | Open-source / cloud | 🔴 build-or-buy open (G-05) |
| I-08 | WhatsApp Business Cloud API | Tier-1 alerts, response opportunities | 5, 29, 31 | Client + Meta | 🔴 templates/recipients (G-03, G-14) |
| I-09 | LLM provider(s) | Labels, explanations, Q&A, briefs | 2, 20–23, labelling | Minsky | 🟡 provider choice open (DEC-16) |
| I-10 | Embedding model | Story matching, campaign similarity, retrieval | 10, 15, 20 | Open-source (self-hosted) | ✅ DEC-13 |
| I-11 | Object storage (S3-compatible) | Raw payloads, scans, page images | 16, 17 | Minsky | ✅ |
| I-12 | PostgreSQL 16 + pgvector | Primary store | all | Minsky | ✅ |
| I-13 | Redis | Celery broker, pub/sub, cache | all | Minsky | ✅ |
| I-14 | Email (SMTP/SES) | Password reset | auth | Minsky | ✅ |
| I-15 | Metrics/logging stack | Observability | 28, 30 | Minsky | ✅ |

---

## 2. X (I-01)

### 2.1 Interface (DEC-18)
```python
class XSourceAdapter(Protocol):
    async def fetch_recent(self, query: XQuery, since_cursor: str | None) -> XPage: ...
    async def fetch_archive(self, query: XQuery, start: datetime, end: datetime, page_token: str | None) -> XPage: ...  # backfill
    async def stream(self, rules: list[XRule]) -> AsyncIterator[XPost]: ...   # raises NotSupported on PollingAdapter
    async def lookup_users(self, ids: list[str]) -> list[XUser]: ...
    async def user_timeline(self, user_id: str, since_id: str | None) -> XPage: ...  # TVK-response detection
    def usage(self) -> XUsageSnapshot: ...  # reads consumed so far, for ops.usage_counters
```
Implementations: `PollingAdapter` (default), `StreamAdapter` (needs a tier that offers streaming), `VendorAdapter` (stub raising `NotConfigured`). Selected by `X_ADAPTER` env (20 §env).

### 2.2 What is collected
Posts, replies, quotes, reposts counts; author profile fields (id, handle, name, verified, followers, following, post count, **account_created_at**); engagement (reposts, likes, replies, views); `referenced_tweets` for reply/quote linking; timestamps. Mapped into `core.mentions` + `core.authors` (B1). Raw JSON → object storage (`raw_ref`).

### 2.3 Query strategy `[PROPOSED]`
The PRD's constraint is the **3M reads/month cap (≈100k/day)** and that "tracking all parties plus campaign detection will likely exceed it" (p.16–17). Design so the cap is spent where it matters:

1. **Query set is generated from `config.watch_terms` + `config.entity_accounts`**, not hand-written. The generator (`XQueryBuilder`) lives in `backend/app/integrations/x/queries.py` and emits several queries, each ≤ X's query length limit `[VERIFY]`.
2. **Priority tiers** (data in `config.entities`, new column not required: derive from `is_primary` and `relationship_tag`):
   - P0: client-primary entities (TVK, CM Vijay, AadhavArjuna) – polled every `x_poll_seconds_p0` (default 60).
   - P1: ally/supporting parties – default 120.
   - P2: opposition and neutral parties – default 300.
   - Client-side accounts (TVK-response detection): `tvk_poll_seconds` (120).
3. **Cheap counts before expensive reads** (PRD cost rule): where the tier offers a counts endpoint `[VERIFY]`, call it to decide whether a full read is warranted for P2 queries.
4. **No item paid for twice**: `since_id` cursor per query in `ops.ingestion_cursors`; DB unique key `(client_id, source_type, external_id)`; in-memory Bloom/Redis set of seen IDs for the last 24 h to avoid even writing.
5. **Budget guard**: `ops.usage_counters` incremented by the adapter after every response with the number of posts returned. At 50/80/95 % a notification + `source_health` annotation is raised (FR-I30). Above the **hard budget guard** (`X_MONTHLY_READ_CAP` env, default 3 000 000) the poller switches to *sampling mode*: P2 queries are throttled first, then P1; P0 and client-side accounts are never throttled. Every aggregate computed while sampling mode was active in its window carries `is_sampled=true` (BR-040) — the dashboard SQL joins `ops.sampling_windows` (see §2.6).
6. **No multi-app/multi-account workaround** (BR-054): adapter code uses **one** credential set per client. A unit test (`test_single_credential_set`) asserts the config schema rejects a list of X apps.

### 2.4 Backfill from `config.clients.history_start_date` (2026-05-10 for TVK)
- Needs **archive-capable access** (G-15). With only recent search the backfill is partial; the system records `ops.backfill_state` as `partial` and the dashboard shows "History available from <date>" instead of pretending.
- Backfill runs as low-priority Celery jobs in 6-hour windows per query, resumable via cursor, rate-limited to a share of the daily budget (`X_BACKFILL_DAILY_SHARE`, default 0.3). Results go through the **same pipeline** but flagged `ingest_mode='backfill'` so no live alerts fire for historical items (alerts only for items with `published_at` within the last `live_alert_window_minutes`=30).
- Backfill feeds the spike baseline (G-16) and the gold-set candidate pool.

### 2.5 Freshness
- P0 poll 60 s + processing ≈ target. A p95 post-to-screen ≤ 300 s is measured via `ops.pipeline_latency`. If polling cannot meet it on spike days the stack switches to `StreamAdapter` (a configuration change, not a code change).

### 2.6 `ops.sampling_windows` (addition)
```sql
CREATE TABLE ops.sampling_windows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(), client_id UUID NOT NULL, provider TEXT NOT NULL,
  starts_at TIMESTAMPTZ NOT NULL, ends_at TIMESTAMPTZ, reason TEXT NOT NULL, throttled_tiers TEXT[] NOT NULL
);
```

### 2.7 Display and storage rules
- X Developer Policy and display requirements apply `[VERIFY]`. Until G-06 is closed, DEC-12 applies (no post text on the big screen).
- Do not store more author data than 11/B1 lists. Honour post/account deletions: if the API reports a post deleted/compliance event `[VERIFY]`, the `compliance_sync` job marks the mention `retention_class='deleted_by_source'` and removes `snippet`/`full_text`.

### 2.8 Failure and fallback
| Failure | Detection | Visible behaviour | Recovery |
|---------|-----------|-------------------|----------|
| 401/403 | HTTP status | `source_health(x)=down`, banner "X data delayed since HH:MM", admin notification | Credentials rotation runbook (19) |
| 429 rate limit | HTTP status + headers | `delayed`; backoff to reset time; P2 throttled | Auto-resume |
| Network/5xx | exception | `delayed` after 2 consecutive failures | Exponential backoff, max 5 min |
| Cap reached | usage counters | Sampling mode; "sampled" label on numbers | Next month or tier upgrade |
| Empty response | — | **Never** counted as health failure and **never** shown as "no new items" when status ≠ healthy | — |

---

## 3. News (I-02, I-03, I-04)

### 3.1 Interface
```python
class NewsSource(Protocol):
    kind: Literal["feed","crawl","youtube","search"]
    async def discover(self, outlet: Outlet, cursor: str | None) -> list[ArticleStub]: ...
    async def fetch(self, stub: ArticleStub) -> Article: ...   # headline, snippet, published_at, author?, url
```
Dispatch by `config.outlets`: `feed_url` present → `FeedSource`; else `crawl_url` → `CrawlSource`; YouTube → `YouTubeSource`; `SearchDiscoverySource` runs only for outlets **not** in the register.

### 3.2 Feeds
`feedparser` (RSS/Atom) and a sitemap parser for **news sitemaps** (`<news:news>` publication dates). Honour `ETag`/`Last-Modified`; cursor in `ops.ingestion_cursors`. Poll interval per outlet: `poll_seconds` default 300 (new column on `config.outlets`: `poll_seconds INT DEFAULT 300`; added in migration 0007 with this doc). "As fast as the outlet's feed allows" — the **floor per source type** the PRD asks the CTO to state is proposed here: feed 5 min, crawl 10 min, e-paper same morning, clipping within 1 h of upload.

### 3.3 Crawling (BR-005)
- Obey `robots.txt` (cached 24 h; a change that disallows the path disables the outlet and notifies admin).
- User-Agent: `WarRoomBot/1.0 (+<contact URL>)`; contact URL from env `CRAWLER_CONTACT_URL`. **Never put a personal email in the UA.**
- Pacing: per-host token bucket, default 1 request / 5 s, concurrency 1 per host; jittered; honour `Retry-After`; respect `Crawl-delay`.
- Extraction: `trafilatura` first; per-outlet CSS selector override in `config.outlets.extraction_rules JSONB` (new column); headless browser (Playwright) only when `requires_js=true` (new column), run in a separate worker pool because it is slower and costlier (PRD open decision).
- **Store metadata, headline, snippet (≤ 300 chars), and link — not full text** (BR-006). `full_text` stays NULL unless `outlets.licensed_full_text=true` (new column, default false). The snippet cap and no-full-text rule are enforced in one function `to_mention_text()` with a unit test.
- Crawl health: if a crawl returns an HTTP error, a parse that yields zero articles on a page that previously yielded some, or a layout change (selector returns nothing), record `delayed`/`down` with the reason. A **successful run that finds zero new articles is healthy and distinct** from an errored run (FR-I28).
- Sites that block bots: record `down` with reason `blocked`; do **not** evade (no rotating proxies, no UA spoofing).

### 3.4 Search discovery (I-04) `[PROPOSED]`
Provider adapter (`GoogleCseAdapter`, `SerpApiAdapter`) behind `SearchDiscovery` interface; disabled unless `SEARCH_PROVIDER` is set. Runs every 30 min with queries from top watch terms; any URL on an outlet **not** in the register creates a `config.outlets_candidates` row for staff review (G-… outlet addition workflow) and a Mention is created only if the domain is approved or `auto_ingest_unknown=true`.

### 3.5 Dedup across outlets
Syndicated copies: `text_hash` (B9) + embedding similarity ≥ 0.95 within 24 h link as `duplicate_of` in `story_mentions` metadata but still count as an outlet in reach ("9 outlets").

### 3.6 Language and Tamil numerals
Tamil text is normalised (NFC), Tamil numerals (௧௨௩…) and amounts ("௫ கோடி") are parsed by `app/nlp/tamil_numbers.py` (BR-014) before labelling; unit tests in 18 §unit.

---

## 4. YouTube (I-05)
- **Official Data API v3 only.** Channel feed = uploads playlist of each `config.outlets.youtube_channel_id`, polled every 5 min; `search.list` is **not** used for routine polling (quota-expensive `[VERIFY]`).
- A new video becomes a Mention (`source_type='youtube'`) from title + description + published time. Captions are fetched only where the API permits for that video (G-20). If no transcript is available the Mention is flagged `text_source='metadata_only'`; story 33's "quoted line and speaker" is then filled from news reports quoting the video, and the pipeline links the two by embedding similarity.
- Quota tracking in `ops.usage_counters(provider='youtube')`; at 80 % warn admin.
- **Decision needed (G-20):** whether to add licensed speech-to-text on audio the war room is entitled to use. Out of build scope until approved.

---

## 5. E-paper (I-06)
```python
class EpaperProvider(Protocol):
    publisher: str
    async def list_editions(self, date: date) -> list[EditionRef]: ...
    async def fetch_edition(self, ref: EditionRef) -> list[PageImage]: ...   # page images and/or PDF
```
- **Per-publisher provider classes**, enabled per row in `config.epaper_subscriptions` (new): `publisher`, `automation_permitted BOOLEAN` (**default false** — flips only when legal confirms G-02), `credentials_secret_name`, `fetch_time_local`, `method` (`download`/`portal`/`manual_upload`).
- Where `automation_permitted=false` the publisher's pages are brought in **via the clipping upload path** (screen 6) so the war room still gets print coverage without a terms breach.
- Pipeline: page images → `OcrEngine` → article segmentation (layout heuristics: columns, headline font size via OCR box heights) → one Mention per story (`source_type='epaper'`, page number, edition date, `page_image_url` signed on demand) → standard pipeline. Low-confidence text flagged "check scan" (`ocr_confidence < ocr_confidence_flag_below`).
- Schedule: daily at `fetch_time_local` with retries every 15 min until success or noon; a missing expected edition by `expected_by` creates `source_health(epaper)=delayed`.
- Store page images and extracted text only for subscribed editions (BR-006, BR-055).

---

## 6. OCR (I-07) and clippings
```python
class OcrEngine(Protocol):
    name: str
    async def read(self, image: bytes, langs: tuple[str,...]=("tam","eng")) -> OcrResult  # text, per-line boxes, per-line confidence
```
- Default `TesseractEngine` (`tesseract-ocr`, `tesseract-ocr-tam`, `tesseract-ocr-eng`; pre-processing: deskew, denoise, binarise via OpenCV; 300 dpi minimum, upscale smaller images).
- `CloudOcrEngine` slot (Google Vision / AWS Textract adapters) — **built only if** the accuracy bake-off on 50 real pages (task T-OCR-02) shows Tesseract below the agreed bar. Metric: character error rate and headline-recall on a hand-typed ground truth.
- Clipping upload flow: file → virus scan (ClamAV or provider scan) → type check (JPG/PNG/PDF; max 15 MB; ≤ 20 pages) → sha-256 dedupe (`DUPLICATE_UPLOAD`) → store → Celery OCR task → `core.uploads.status` → Mention. Auto-filled outlet/date/page are **suggestions**; the form values the user submitted win.
- Failure: OCR engine down → upload stays `processing` for up to 10 min with a visible "OCR delayed" state, then `error` with a retry button; `source_health(clipping_ocr)` updated.

---

## 7. WhatsApp (I-08)

### 7.1 Interface
```python
class WhatsAppClient(Protocol):
    async def send_template(self, to_e164: str, template: str, params: list[str], lang: str) -> SendResult  # returns provider_message_id
    def parse_webhook(self, payload: dict, signature: str) -> list[StatusUpdate]
```
Implementation: Meta **WhatsApp Business Cloud API** `[VERIFY current API version]`.

### 7.2 Rules that shape the build
- **Business-initiated messages need approved templates** and the recipient must have opted in (G-14). Define templates now so approval can start in week 1:

| Template name | Purpose | Params |
|---------------|---------|--------|
| `wr_alert_tier1` | Influence trigger, spike, campaign, ally | alert type, entity, reason (≤ 200 chars), link |
| `wr_response_opportunity` | Story 31 | title, who is reacting (≤ 120 chars), "no TVK response yet", link |
| `wr_overnight_digest` | 06:30 overnight summary (typical day) | count, top alert, link |
| `wr_delivery_failed_lead` | (in-app only; no template) | — |

  Languages: English and Tamil variants for each. Template text is client-reviewed; **message bodies contain no raw post text** (privacy + prompt-injection surface); they carry a deep link to the alert.
- **Routing** is data: `config.alert_recipients` by `alert_type` + `entity_id`. Response opportunities go to the recipient configured for `alert_type='response_opportunity'` (Sajan or shift equivalent).
- Sending code lives in **`app/services/whatsapp_sender.py`** and is the *only* module with WhatsApp credentials; the co-worker service has no import path to it (BR-043; enforced by an import-linter contract, 16 §boundaries).
- **Delivery tracking:** webhook (`POST /webhooks/whatsapp`, signature-verified, unauthenticated otherwise) updates `ops.whatsapp_deliveries`; statuses `accepted → sent → delivered → read | failed`.
- **Retry:** on API error or `failed` status: up to `whatsapp_max_retries` with `whatsapp_retry_backoff_seconds` × 2^n. After the last failure: `alert_events('whatsapp_failed')`, notification `whatsapp_undelivered` to all `war_room_lead` users, WS `alert.whatsapp_failed`, in-app badge "WhatsApp not delivered" (story 29).
- **SLA clock:** DEC-04 — `accepted_at − trigger_at ≤ 300 s`.
- Webhook endpoint must tolerate duplicates and out-of-order status events.

### 7.3 Failure
Meta outage ⇒ every Tier-1 alert still fires on the big screen and in-app (WS), `source_health(whatsapp)=down`, banner shown to leads.

---

## 8. LLM provider (I-09)

```python
class LLMClient(Protocol):
    async def complete(self, route: ModelRoute, system: str, user: str, *, content_blocks: list[ContentBlock] = (), json_schema: dict | None = None, timeout_s: float) -> LLMResult
    # LLMResult: text/json, input_tokens, output_tokens, model_name, latency_ms
```
- Providers are adapters (`anthropic`, `openai`, `google`…); model name and prompt version come from `config.model_routes` (B3). **No model name literal appears in application code**; `grep` check in CI.
- **Cost rule (PRD p.17):** cheap route classifies every kept item; strong route only for (a) items under `label_confidence_escalate_below`, (b) alert explanations, (c) briefs, (d) co-worker Q&A, (e) language fallback. Token usage → `ops.usage_counters(provider='llm')`.
- **Content-as-data (BR-042):** all ingested text is passed in `content_blocks`, delimited and **never concatenated into the system prompt**. Prompt templates live in `backend/app/prompts/<capability>/<version>.md`. See 16 §prompts.
- **Structured output:** labelling returns JSON validated by Pydantic; invalid JSON → one repair retry on the same route → escalate.
- **Timeouts/fallback:** cheap 15 s, strong 45 s, chat first-token 10 s. On failure: item stays in `processing_stage='filtered'` and is retried with backoff; UI shows "pending label"; `source_health(llm_*)` updated; co-worker answers "temporarily unavailable" (08 §3.2) — it **never** answers from stale data.
- **Evals:** `evals/` suite (18 §evals) runs on every change to a prompt file or `model_routes` seed.
- Data handling: prompts contain public posts and articles, not staff personal data. WhatsApp numbers and passwords are never sent to a provider. Provider region and data-retention terms to be confirmed (G-19 / DPDP).

---

## 9. Embeddings (I-10)
Self-hosted `sentence-transformers` model per DEC-13, run in a dedicated worker pool (`embed` queue, CPU; GPU optional). Batch size 32, max 512 tokens, truncation allowed. Stored in `core.mentions.embedding`. Used for Story matching, campaign text similarity (secondary to n-gram MinHash), TVK-response matching, co-worker retrieval, memory recall. A model change = new `embedding_model` value + re-embed job; mixed-model comparisons are forbidden (queries filter by the same `embedding_model`).

---

## 10. Storage, database, cache, email
- **Object storage (S3-compatible):** buckets `wr-raw` (raw payloads), `wr-uploads` (clippings), `wr-pages` (e-paper pages). SSE enabled, no public access, **signed URLs (5 min)** for any image the UI shows (13 §6.2). Lifecycle rules implement `retention_days_raw`.
- **PostgreSQL 16 + pgvector:** extension `vector` ≥ 0.7 `[VERIFY]`; HNSW index on `mentions.embedding`. PgBouncer in transaction mode — RLS uses `SET LOCAL app.client_id` inside each transaction (works with transaction pooling).
- **Redis:** Celery broker/result backend, pub/sub channel `ws:{client_id}`, rate-limit counters, seen-ID set. Persistence on (AOF) so queued tasks survive restarts.
- **Email:** SMTP/SES for password reset only. Templates in English and Tamil.

---

## 11. Runtime library dependencies (pinned in 20)

| Area | Libraries |
|------|-----------|
| Backend | fastapi, uvicorn, pydantic v2, sqlalchemy 2 (asyncio), asyncpg, alembic, python-jose, passlib[bcrypt], httpx, tenacity, structlog, prometheus-client, celery[redis], redis |
| NLP | sentence-transformers, torch (CPU), fasttext-wheel `[VERIFY]`, scikit-learn, numpy, scipy, datasketch (MinHash), langdetect **not used** (BR-013) |
| Ingestion | feedparser, trafilatura, scrapy **or** httpx+selectolax (CTO may choose; default httpx+trafilatura), playwright (optional, separate image), pytesseract, opencv-python-headless, pdf2image/pypdfium2 |
| Frontend | next, react, typescript, @tanstack/react-query, zustand, tailwindcss, @radix-ui/*, recharts, next-intl **or** next-i18next, date-fns-tz |
| Tooling | pytest, pytest-asyncio, respx, testcontainers, hypothesis, ruff, mypy, import-linter, jest, @testing-library/react, playwright, eslint, prettier |

License check for every new dependency is a CI step (16 §dependencies).

---

## 12. Third-party failure matrix (PRD "Failure behaviour", p.19–20)

| Dependency | Down state shown | Pipeline behaviour | Alerts behaviour | Owner of recovery |
|-----------|------------------|--------------------|------------------|-------------------|
| X | banner + admin health | other sources continue | influence/spike for X pause; nothing says "quiet" | on-call |
| One news outlet | banner per outlet (admins), summary count on dashboard | others continue | none | on-call |
| E-paper | banner when edition overdue | clipping upload still works | none | ops |
| OCR | "OCR delayed" on uploads | uploads queue | none | on-call |
| LLM cheap | "pending label" badge | items wait, retried | alerts needing labels delayed; **influence trigger falls back to a rules-only path** (verified + tier + keyword stance heuristic) so must-pass test 2 degrades gracefully, flagged "unlabelled by AI" | on-call |
| LLM strong | co-worker "unavailable" | low-confidence items wait for human queue | explanation `unavailable` | on-call |
| Embeddings | "story grouping delayed" | items skip story step, retried | response-opportunity detection degraded; flagged | on-call |
| WhatsApp | banner to leads | — | in-app + big screen still fire; leads notified | on-call |
| Redis | app read-only message | Celery halts; DB writes from API continue | alerts cannot be created: health check pages on-call | on-call |
| PostgreSQL | maintenance page | queue holds | — | on-call |

The rules-only fallback for the influence trigger is the **one deliberate exception** to "no AI → no label": the PRD demands a 5-minute alert for the single critical post, so a degraded alert marked clearly is better than none.

---

## 13. Open integration items (all tracked in PRD-GAP-ANALYSIS)
G-01 X tier, G-02 e-paper terms, G-03 WhatsApp recipients, G-05 OCR build/buy, G-06 X display rules, G-14 template approval, G-15 archive access, G-20 YouTube transcripts, G-21 per-party query budget.
