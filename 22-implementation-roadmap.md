# 22 — Implementation Roadmap

Phases, milestones, gates and exit criteria. **Tasks** are in 23; **modules** in 17; **tests** in 18; **release checklist** in 24. The PRD does not set dates, so this roadmap is ordered by dependency and exit criteria, not calendar time.

**Principle (17 §2, vertical-slice rule):** after the foundation and configuration, the first thing that works end-to-end is *X post → filter → label → weigh → influence alert → big screen + WhatsApp* (must-pass test 2). Do not finish a module's breadth before that slice works.

Status markers: `[PRD]` stated · `[PROPOSED]` recommendation · `[GAP]` client/CTO input owed · `[GATE]` blocks the next phase.

---

## 1. Phase overview

| Phase | Name | Modules | Outcome you can demo/test |
|---|---|---|---|
| 0 | Foundation | M00 | Login, roles, tenancy, audit, WebSocket, Celery, seeds, CI gates. Cross-client isolation proven |
| 1 | Configuration and first ingestion | M01, M02 (core), M05 (filter/language) | Admin edits watch terms/tiers; X mock posts are collected, deduplicated, filtered, language-detected |
| 2 | Slice part 1: label → weigh → influence alert | M05 (labelling), M06, M09 (influence path) | A critical post from a top-tier account produces a Tier-1 alert with a reason |
| 3 | Slice part 2: delivery | M09 (WhatsApp, ack/assign), M10 (dashboard, big screen core) | **TC-MP2 passes**: alert on big screen and WhatsApp mock ≤ 300 s |
| 4 | Breadth of sources and Stories | M03, M04, M07, M13 (health) | News, YouTube, e-paper, clippings → Mentions → Stories; source health visible |
| 5 | Detection | M08, M09 (spike, ally, campaign, response opportunity), M07 (response queue) | **TC-MP1 passes**; spike, campaign, ally mood alerts |
| 6 | Co-worker, corrections, CM view | M11, M12, M10 (CM view, presentation mode, notifications) | Typical day (04 §1) runs end-to-end; corrections, bias guard, staging, Q&A, briefs |
| 7 | Hardening and launch readiness | M13 (rest), infra, performance, security, UAT | 20 consecutive nightly must-pass runs; release gate (24 §2) |
| 8 | Calibration (live) | — | Two weeks of live use; thresholds and accuracy bars set; stage engine armed |

Parallelism `[PROPOSED]`: infra (T-INF-*) runs alongside phases 1–6; frontend tasks for a module follow its backend endpoints; M03/M04 can start once M02's interfaces and M05's filter are stable.

---

## 2. Phase details

### Phase 0 — Foundation (M00)

**Scope:** monorepo skeleton, settings, clock, logging, error envelope; migrations 0001–0006 and models; RLS, DB roles, triggers; tenancy context; auth (login, refresh rotation, logout, lockout, password reset, PIN, device tokens); RBAC guards generated from the permission matrix; audit service; WebSocket manager; Celery app/queues/dead letters; health endpoints and metrics; seed loader (`demo`, `tvk` placeholders); `check_hardcoding.py`; reconciliation addendum and migration `0007`.

**Exit criteria:** TC-FND-01…07 and TC-SEC-01, 02, 05 pass; cross-client attack suite green; permission-matrix and audit-coverage tests green; `make dev` / `migrate` / `seed` work on a clean checkout (20 §10).

**Do not build:** public sign-up, social login, multi-client switching UI.

### Phase 1 — Configuration and first ingestion (M01, M02, M05 part)

**Scope:** `ConfigProvider` with cache and invalidation; CRUD + audit for entities/tags (+history), watch terms, tiers/members, outlets/candidates, entity accounts, thresholds, recipients, event calendar, weight profiles, model routes, users, devices; admin config screens 7A–7G; X source adapter (`PollingAdapter`), mock X server, query builder from watch terms, priority polling, cursors, dedupe, authors, raw archive, usage counters and sampling mode; text normalisation, Tamil numerals, language detector, watch-term matcher, wide-net filter, raw storage of unmatched items, embeddings.

**Exit criteria:** TC-FND-04, TC-FND-05; TC-PIPE-01, 02, 03, 04, 12; threshold change takes effect ≤ 30 s with audit row; ingestion refuses to start while retention is unset (TC-NFR-04 part).

**Gate `[GATE]`:** none required from the client; X tier (G-01) is only needed for production, development uses the mock.

### Phase 2 — Label → weigh → influence alert (M05, M06, M09)

**Scope:** cheap labeller with prompt files and schema validation; strong-route escalation and human review queue; gold-set few-shot selection; influence formula, per-post cap (property-tested), sentiment aggregation (weighted, raw, z-test, Wilson), bot heuristic, `mention_weights` writer, rollups, effective vs model-only views; severity scorer; alert repository/dedupe/tiering; **influence trigger evaluator including the instant path (DEC-20)**; rules-only fallback when the cheap LLM route is down.

**Exit criteria:** TC-US05 (alert part), TC-US07, TC-PIPE-05/06/07/08/14, TC-NFR-06; replay shows the alert created with a reason string and `trigger_at = published_at`.

**`[GAP]` to flag:** G-23 (instant path confirmation by client). Build with the documented default and record the assumption.

### Phase 3 — Delivery (M09, M10 core)

**Scope:** WhatsApp sender service, templates (English/Tamil), webhook with signature verification, retry/backoff, undelivered notifications; ack/assign with WebSocket events; dashboard summary and big-screen endpoints; number components; dashboard and alert screens; big-screen kiosk with watchdog; source-health banner; i18n en/ta.

**Exit criteria `[GATE]`:** **TC-MP2 (TC-US05) passes** with accepted − published ≤ 300 s, reason string present, no spike alert in the window, and the 40-follower variant produces no Tier 1. TC-US08, TC-US29 (retry/in-app badge), TC-SEC-08.

### Phase 4 — Breadth of sources and Stories (M03, M04, M07, M13 part)

**Scope:** feed/sitemap source, crawler (robots, pacing, UA), extraction and snippet cap, optional Playwright pool, YouTube source, search discovery (off by default), syndication dedupe; OCR engine, **Tesseract vs cloud bake-off (T-OCR-02)**, upload API (virus scan, dedupe), clipping flow, e-paper provider framework (manual-upload path first); story matcher with advisory lock, reach counters, unattributed single-item Stories, merge/split, outlet tier display; source health aggregator and admin health screen.

**Exit criteria:** TC-US15, 16, 17, 18, 19, 33, 28; TC-PIPE-10, 11, 13; TC-NFR-03 (chaos subset for sources).

**Gates `[GATE]` / `[GAP]`:** automated e-paper ingestion for a publisher only after legal confirms (G-02); until then `automation_permitted=false` and print enters via clipping upload. OCR engine choice recorded after the bake-off (G-05).

### Phase 5 — Detection (M08, M09, M07)

**Scope:** spike evaluator (standard and event baselines, cold start); ally mood evaluator and panel logic; campaign clustering, scoring, persistence, confirm/dismiss with guards, spread timeline; response-opportunity trigger with TVK-response closure (account polling); response queue; campaign and response-queue screens; alert auto-resolution.

**Exit criteria `[GATE]`:** **TC-MP1 (TC-US31) passes** (≤ 300 s, ≥ 3 reacting names, "no TVK response yet", later closure "responded by … at HH:MM", variants with LLM down and WhatsApp failing). TC-US09, 10, 11, 12, 13, 14, 32.

**`[GAP]`:** response-trigger numeric definition (DEC-08 default; PRD-GAP-ANALYSIS 3.4); ally threshold values come from calibration.

### Phase 6 — Co-worker, corrections, CM view (M11, M12, M10)

**Scope:** corrections with bias guard and second review; effective-label application and aggregate invalidation; gold-set ingestion; weekly metrics; staging engine (promote/demote) and `StagePolicy`; co-worker tools, agent loop and grounding validator, streaming, interaction logging, rating, suggestions; async alert explanations; CM summary and shift handover with ally-blame gate; briefs lifecycle; memory; eval harness and injection suite; presentation-safe mode and redaction; CM View pages; notifications; admin accuracy screens.

**Exit criteria `[GATE]`:** typical-day walkthrough (04 §1) completes end-to-end in Playwright (English and Tamil); TC-US02, 04, 20–26, 34, 35; TC-SEC-03; evals pass their gates (citation correctness ≥ 0.95, no ungrounded number).

### Phase 7 — Hardening and launch readiness

**Scope:** remaining M13 (filter audit sampler and reviewer UI, audit-log viewer, retention jobs, dead-letter view); performance (TC-NFR-01 burst, 24 h big-screen soak); chaos tests; security suite and ZAP; accessibility; infrastructure for staging and production, monitoring, backups, runbooks (19); UAT with the war room; gold-set build session; WhatsApp sandbox rehearsal.

**Exit criteria `[GATE]`:** all of 24 §2 (release gate) including **20 consecutive nightly passes** of TC-MP1 and TC-MP2 (18 §5), TC-NFR-01 p95 ≤ 300 s (critical path ≤ 90 s), cross-client attack suite green, runbooks reviewed, retention configured, WhatsApp templates approved.

**Required client inputs `[GAP]` (24 §7):** real party/leader list and tags; standing tiers; watch terms; WhatsApp recipients and routing; two named attribution leads; shift times; outlet register confirmation; e-paper terms; labelled starter set; official TVK elephant artwork and co-worker name; X access decision (G-01).

### Phase 8 — Calibration (live, two weeks)

**Scope `[PRD]`:** measure daily volume (owed), tune thresholds against the gold set and act/dismiss data, set per-language accuracy bars and capability stage bars, review alert-fatigue guardrails, review the missed-item rate trend. Engage the stage engine only after bars are set (a NULL bar keeps a capability at Assist).

**Exit:** thresholds reviewed with the war room; bars recorded; stop/rollback signals dashboard live (PRD): most big-screen alerts dismissed after calibration, Tamil/Tanglish F1 below the agreed bar, or a privacy incident ⇒ stop/roll back.

---

## 3. Milestones

| ID | Milestone | Reached when |
|---|---|---|
| MS-0 | Foundation green | Phase 0 exit |
| MS-1 | Configurable ingest | Phase 1 exit |
| MS-2 | **Vertical slice** | Phase 3 exit (TC-MP2 passing) |
| MS-3 | Full source coverage | Phase 4 exit |
| MS-4 | Detection complete; **both must-pass tests pass** | Phase 5 exit |
| MS-5 | Feature complete (typical day works) | Phase 6 exit |
| MS-6 | Launch candidate | Phase 7 exit (24 §2) |
| MS-7 | Calibrated | Phase 8 exit |

## 4. Critical path and external dependencies

Critical path: M00 → M01 → M02 → M05 → M06 → M09 → M10 → (TC-MP2) → M07/M08 → M09 rest → (TC-MP1) → M11/M12 → hardening.

| Dependency | Needed by | If late |
|---|---|---|
| X access tier decision (G-01); enterprise/vendor contract lead time | Production launch, backfill completeness, streaming | Develop on mock; backfill is `partial` with an honest "History available from …" banner. **Start talks now** (PRD decision 1) |
| WhatsApp Business account, templates approved (G-03, G-14) | Production Tier-1 delivery | Develop on mock; submit template approval in week 1 |
| E-paper terms per publisher (G-02) | Automated e-paper | Print via clipping upload |
| Tamil OCR decision (G-05) | Phase 4 | Default Tesseract; cloud adapter slot ready |
| Gold set from the war room | Calibration, accuracy bars, staging | Seed Tamil corpus (≥ 300 items) for development; real gold set required before calibration |
| LLM provider choice and region terms (DEC-16, G-19) | Production | Adapters keep it configurable |
| X display rules for the wall screen (G-06) | Launch | DEC-12: no post text on the big screen until confirmed |
| Retention periods (PRD pre-mortem, BR-048) | **Before ingestion starts** | Ingestion is blocked by design |

## 5. Risks to the schedule `[from PRD pre-mortem; mitigations in design]`

1. **Confident wrong answers in Tamil/Tanglish** — build the gold set from real material before launch; per-language bars; reasons on every label; ally blame reviewed before it reaches the CM.
2. **Alert fatigue in campaign season** — severity tiers, big-screen cap, event-aware baselines, weekly tuning on act/dismiss data; the must-pass case always alerts.
3. **Coverage gap on the day it matters** — X cap telemetry, crawl health, e-paper and clippings in Phase 1, outlet register with feed status.
4. **Integration drift** — third-party terms change (`[VERIFY]` items in 14): re-check at the start of each integration task.

## 6. Change control

Any change to scope, a hard rule, or a gate is recorded in `docs/OPEN-QUESTIONS.md` and requires human approval (21 §7.1). Update this roadmap and 23/24 in the same PR.
