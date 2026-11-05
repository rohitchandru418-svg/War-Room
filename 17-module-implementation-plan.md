# 17 — Module Implementation Plan

Feature-wise breakdown of what to build, in which module, in what order, with dependencies and explicit **"do not build"** lines. Tasks are in 23; phases in 22. Binding addenda: `DOC-RECONCILIATION.md`.

---

## 1. Module catalogue

| Mod | Name | PRD stories | Main FR/BR | Screens (07) | Primary tables |
|-----|------|-------------|-----------|---------------|----------------|
| M00 | Foundation: repo, tenancy, auth, RBAC, audit, WS, error envelope, migrations, seeds | — (NFR-05/06/07) | NFR-05/06/07, BR-050/051 | Login, PIN | accounts.*, config.clients, audit.audit_log |
| M01 | Configuration & admin | 6, 14 | FR-B06, FR-E14, BR-007–009, BR-016 | 7A–7G | config.* |
| M02 | X ingestion & backfill | 1, 5, 30 (inputs) | FR-I28, FR-I30, BR-002, BR-053/054 | 8A, 8B | core.mentions, core.authors, ops.* |
| M03 | News ingestion (feed, crawl, YouTube, search) | 15, 19, 33 (inputs) | BR-003, BR-005, BR-006, FR-J33 | 8A | core.mentions, config.outlets |
| M04 | E-paper & clippings (OCR) | 16, 17 | FR-F16, FR-F17 | 6 | core.uploads, core.mentions |
| M05 | Filter, language detection, labelling | 19 (labels), 24 (inputs) | BR-001, BR-010–015 | Mention rows | core.mention_labels, core.mention_quotes |
| M06 | Weighing & sentiment aggregation | 1, 7, 18 | FR-B07, BR-016–019, BR-037–040 | 1B, 2C | core.mention_weights, config.weight_profiles |
| M07 | Stories & response queue | 15, 19, 32 | FR-F15, FR-F18, FR-J32, BR-020–022 | 2, 1E, Response queue | core.stories, core.story_mentions |
| M08 | Campaign detection & attribution | 10, 11, 12 | FR-D10–D12, BR-030–032 | 4 | core.campaigns, core.campaign_accounts |
| M09 | Alerting, severity, routing, WhatsApp | 5, 8, 9, 13, 29, 31 | FR-B05, B08, C09, E13, I29, J31, BR-023–029 | 1D, 3 | core.alerts, alert_events, ops.whatsapp_deliveries |
| M10 | Dashboard, big screen, CM view, presentation mode, realtime | 1, 2, 3, 4, 13 | FR-A01–A04, FR-E13 | 1, 10, 11 | views over core.* |
| M11 | Corrections, gold set, bias guard, learning metrics, staging | 24, 25, 26, 34, 35 | FR-H24–H26, FR-K34/35, BR-033–036, BR-046 | 9, correction panel | core.label_corrections, learning.* |
| M12 | AI co-worker (Q&A, explanations, briefs, memory, evals) | 2, 20, 21, 22, 23 | FR-G20–G23, BR-041–045 | 5, briefs | core.briefs, learning.coworker_* |
| M13 | Platform health, usage, filter audit, audit-log viewer | 27, 28, 29, 30 | FR-I27–I30 | 8, 9D, audit | ops.*, learning.filter_audit_samples |

---

## 2. Dependency graph (build order)

```
M00 ──► M01 ──┬─► M02 ──┐
              ├─► M03 ──┤
              └─► M04 ──┴─► M05 ─► M06 ─► M07 ─► M09 ─► M10
                                    │       │      ▲
                                    │       └► M08 ┘
                                    └─────────────► M11 ─► (staging uses M05..M09 outputs)
M05 + M06 + M07 + M09 ─► M12
M02..M09 ─► M13 (health writes begin in M02; UI in M13)
```
**Vertical-slice rule:** after M00+M01, the first slice that goes end-to-end is *X post → filter → label → weigh → influence alert → big screen + WhatsApp* (must-pass test 2). Do not complete a module's breadth before this slice works (22 §phase 2–3).

---

## 3. Module specifications

### M00 Foundation
**Build:** monorepo skeleton (15); settings; DB session with RLS context; migrations 0001–0011; models for all tables in 11 + addendum; auth (login, refresh with rotation, logout, PIN login, password reset, lockout after 5 failures); role guards generated from the permission matrix; device tokens; audit service; error envelope; WebSocket manager (auth, per-client rooms, Redis pub/sub bridge, heartbeat); health endpoints; Celery app and queue routing; structured logging and metrics; seed loader for client profiles; CI pipeline.
**Depends on:** nothing. **Exit test:** TC-FND-* (18): cross-client isolation attack suite passes; permission matrix test passes; audit-coverage test passes.
**Do not build:** public sign-up, social login, user self-registration, multi-client UI switching (one client per user in Phase 1).

### M01 Configuration & admin
**Build:** CRUD + audit for watch terms, tiers & members, outlets (incl. candidates), entities & relationship tags (+ history table writes per FR-E14), `entity_accounts`, thresholds (typed, validated, default shown), alert recipients, event calendar, weight profiles (versioned, activate), model routes, client settings (vocabulary, shifts, retention, flags), users, devices. `ConfigProvider` with cache + invalidation. Admin screens 7A–7G.
**Rules:** every edit is audit-logged with old/new; threshold edits validate type/range; relationship-tag change inserts `entity_relationship_history` with `effective_from=now()` and is applied **only to new Mentions** (history keeps the old tag; stories 14); the tag in force at `published_at` is resolved by `domain.relationship_at(entity, ts)` — used by Ally Mood and reports.
**Do not build:** bulk import UI (seed scripts cover Phase 1; CSV import endpoint is optional T-CFG-12).
**Depends on:** M00.

### M02 X ingestion
**Build:** `XSourceAdapter` + `PollingAdapter`; query builder from watch terms/entity_accounts; poll scheduler by priority tier; cursoring; raw archive; `core.authors` upsert; usage accounting and sampling mode; TVK-response account polling; engagement refresh; backfill job; health recording; replay harness (`scripts/replay_scenario.py` feeds recorded X payloads through the adapter interface for tests).
**Depends on:** M00, M01. **Gate:** G-01 for production tier; development uses recorded fixtures and a mock X server in `backend/tests/mocks/x_server.py`.
**Do not build:** multiple apps/accounts, scraping of x.com, posting, DMs, following, list management.

### M03 News ingestion
**Build:** feed/sitemap source, crawler (robots, pacing, UA), extraction with selector overrides, optional Playwright pool, YouTube source, search-discovery adapter (off by default), outlet health recording, syndication dedupe, quote extraction hook (calls M05), outlet candidate review list.
**Do not build:** full-text storage, paywall bypass, login-based scraping of subscription sites (that's M04's per-publisher provider and only when `automation_permitted`).
**Depends on:** M00, M01.

### M04 E-paper & clippings
**Build:** upload API + storage + dedupe + virus scan; `OcrEngine` + Tesseract engine + pre-processing; OCR confidence and "check scan"; OCR text edit before confirm; page-image signed URLs; e-paper provider framework (manual-upload path first; automated providers only for publishers with `automation_permitted=true`); article segmentation; screen 6.
**Gate:** G-02, G-05. **Bake-off task** T-OCR-02 decides Tesseract vs cloud.
**Do not build:** handwriting OCR, automated e-paper fetch for any publisher without legal clearance, image storage beyond subscription editions.
**Depends on:** M00, M01, M05 (hand-off point is `s2_filter`).

### M05 Filter, language, labelling
**Build:** text normalisation; Tamil numeral parser; language detector (DEC-14); watch-term matcher (Aho-Corasick over normalised terms incl. Tamil and Tanglish variants; hashtags; fuzzy matching only for Tanglish variants listed in data, no generic fuzzy); wide-net rule: matched ⇒ keep, **plus topic-similarity keep** (embedding similarity to entity seed phrases ≥ `topic_similarity_min`, default 0.6) to catch criticism that never names the minister (PRD "keyword alerts" critique); store unmatched as raw; embeddings; cheap labeller; strong escalation; human-review queue; quote/speaker extraction; label writes (append-only); `pipeline_latency` stamps; prompts; gold-set-driven few-shot selection; rules-only influence fallback.
**Depends on:** M01 (terms/entities), M02/M03/M04 (inputs), LLM routes.
**Do not build:** fine-tuning, per-language model training (design for per-language routes only via `model_routes`), automatic watch-term expansion without staff approval (suggestions list only, optional).

### M06 Weighing & sentiment aggregation
**Build:** `domain/influence`, `capping`, `sentiment`; author influence resolution (tier lookup via `tier_members` by platform+handle; outlet tier via `outlets`); bot heuristic; `mention_weights` writer with `weight_version`; aggregate queries and a **materialised rollup** (`core.sentiment_rollup_5m`: client, entity, bucket_start, source_type, pos/neg/neu counts + weight sums, refreshed every minute) so dashboards are cheap; effective vs model-only views; weighted/raw/top-3 payloads; engagement snapshot reader.
**Depends on:** M05. **Do not build:** a blended single score without raw beside it (PRD rejected option).

### M07 Stories & response queue
**Build:** story matching (embedding + entity overlap + time window + hashtag/URL overlap), centroid update, reach counters by source type (outlets, X posts, e-paper, clippings), auto-title (cheap LLM or headline of earliest news item), single-item "unattributed" story for anonymous insinuation (story 19), merge/split with audit, response queue computation, opportunity state, suggested response type, "who has reacted" rollup, timeline endpoints, screens 2 and queue.
**Concurrency:** per-client advisory lock around match-or-create to avoid duplicate Stories.
**Depends on:** M05, M06. **Do not build:** drafting reply text (out of scope).

### M08 Campaign detection & attribution
**Build:** windowed clustering job, signals, confidence, campaign + accounts persistence, link to Story, alert creation via M09, confirm/dismiss with role guard and logging, spread timeline, TVK-response overlay, screen 4.
**Depends on:** M05, M06, M07, `core.authors`. **Do not build:** automated attribution; any text that names a party as actor before confirmation; network-graph UI (optional [PROPOSED] stretch).

### M09 Alerting, routing, WhatsApp
**Build:** severity scoring service; evaluators (influence trigger incl. instant path, volume spike, ally mood, campaign, response opportunity); grouping via `dedupe_key`; tiering; big-screen cap logic (query-level, `big_screen_max_alerts`); ack/assign with events and WS; alert reason strings; explanation hook to M12 (async, `explanation_status`); WhatsApp sender, templates, webhook, retry, undelivered notifications; resolution rules (auto-resolve when conditions subside or opportunity answered); alert history; screens 1D, 3, alerts list.
**Depends on:** M05–M08; WhatsApp account (G-03/G-14) for production, mock server for dev. **Gate for launch:** must-pass replays (18).
**Do not build:** alert muting/snooze (not in PRD; if the team asks, track as backlog), SMS/email alerting.

### M10 Dashboard, big screen, CM view, realtime
**Build:** `/dashboard/summary`, `/bigscreen/summary`, entity drill-down; Ally Mood panel logic (per ally/supporting party, using `entity_accounts` figures); freshness stamps and stale/sampled flags; source health banner; presentation-mode service + redaction; CM summary pages and evidence endpoint; WS events; screens 1, 10, 11, notifications, persistent co-worker button.
**Depends on:** M06, M07, M09 (alerts), M12 for briefs (CM page works with an empty state until M12 lands).
**Do not build:** per-user dashboard customisation, exports/reports (not in PRD).

### M11 Corrections, learning, staging
**Build:** correction submission (label + output corrections), bias-guard evaluation, second-review queue/endpoints/notifications, effective-label application on approval, aggregate invalidation (rollup recompute for affected buckets), gold-set ingestion (every applied correction; plus manual/imported labels), weekly metrics job (F1 per language, correction rate, model-only vs corrected), per-person/per-entity correction-rate tracking for bias guard monitoring, staging engine (promote/demote with reasons and notifications), "model-only" view, stage-aware behaviour: **at Assist, alerts/labels remain reviewable; at Trusted, low-confidence/high-stakes still routed to humans** (a `StagePolicy` object consumed by M05/M09/M12), screens 9A–9C and correction panel.
**Depends on:** M05–M09. **Do not build:** model fine-tuning/retraining pipelines (few-shot selection from gold set only), automatic threshold changes by the system (it may *recommend*, admins apply).

### M12 AI co-worker
**Build:** agent loop, tools, grounding validator, language handling, streaming over WS, interaction logging, rating, suggestions, per-alert explanations (async worker; strict length: two lines), CM summary and handover generation (+ ally-blame detector), briefs lifecycle, memory write/recall/correct, eval harness and datasets, injection test suite, screen 5 + briefs UI.
**Depends on:** M05–M09 data; M11 for correction memory. **Do not build:** any posting/messaging tool, response drafts in TVK's voice, memory of private individuals, answers when LLM is down (must say unavailable).

### M13 Platform health, usage, filter audit
**Build:** health aggregator (`ops.source_health` upserts from every integration), queue/latency endpoints, usage counters UI, WhatsApp log UI, weekly filter-audit sampler (stratified), reviewer UI and missed-item-rate calculation, audit-log viewer, retention jobs (`periodic_retention`), dead-letter view.
**Depends on:** M02–M09 emit health; M00.

---

## 4. Cross-cutting implementation matrices

### 4.1 Stage-aware behaviour (stories 34/35)
| Capability (`capability_metrics.capability`) | At **Assist** | At **Trusted** | At **Co-worker** |
|------|--------------|----------------|------------------|
| `label_en/ta/tanglish` | Every label flagged "AI suggestion", appears in a review queue sample (≥ 50 % sampled until stage change) | Only low-confidence and high-severity-story labels need review | Exceptions only |
| `spike_alert`, `influence_alert` | Fire and display with "suggested" marker; leads must acknowledge | Fire without review | Fire; weekly spot check |
| `campaign_flag` | Always "possible" + lead review | Same (attribution still human always) | Same (attribution still human always) |
| `response_suggestion` | Queue shows suggestion with "AI suggestion" | Queue ordering trusted | Same |
| `brief` | Drafts always need lead edit/approve | Drafts approve-only | Auto-publish **to the war room** allowed; **ally blame still needs a lead** |
Stage never changes the hard rules: attribution and ally blame always need a person; nothing ever publishes.

### 4.2 "Effective vs model-only" parameter
Every aggregate endpoint accepts `view=effective|model_only` (default effective). Implemented once in repository query builders (`labels_scope(view)`).

### 4.3 Where each hard rule is enforced
| Rule | Layer(s) |
|------|----------|
| No posting | Package boundaries (import-linter) + no credentials in LLM service + test |
| Attribution needs lead | Service guard + DB CHECK + route role + UI hides button for lower roles |
| Ally blame review | `briefs` CHECK + `send` endpoint 409 + CM View query filter |
| Corrections never overwrite | Append-only grants + trigger + service |
| Second reviewer ≠ corrector | DB CHECK + service |
| Presentation-safe | `redaction.apply` on every Mention-bearing response + device scope + tests |
| Client isolation | RLS + repository ctx + attack tests |
| Source error ≠ quiet | `Result` types in integrations + `source_health` + UI components |

---

## 5. Definition of "module complete"
A module is complete when (a) every task in 23 for it is merged, (b) its acceptance tests in 18 pass in CI, (c) its rows in 24 §3 are ticked with evidence, (d) its screens (if any) pass the state checklist from 08 (loading, empty, error, success), and (e) docs 11/12 patches for its tables/endpoints are merged.
