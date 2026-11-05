# 23 — Task Breakdown

Small, independently executable tasks for OpenHands. **One task = one branch (`feat/<task-id>-<slug>`) = one PR** (21 §6). Pick a task only when all its dependencies are merged and its phase gate (22) is open.

**Format:** `ID — title` · **Phase** · **Deps** · **Refs** (docs/rules) · **Do** · **Accept** (every item must be demonstrable with command output or a screenshot; tests named in 18).

Common acceptance (applies to **every** task, not repeated): `make lint` and `make test` pass; tests for the new behaviour added; audit/permissions/i18n/redaction considered; traceability tags (`# BR-0xx`, `# FR-xxx`, `# DG-xx`, `# DEC-xx`) in code; no client-specific literal (DG-01); docs 11/12/20 updated if tables/endpoints/env change.

Task IDs by module: FND (M00) · CFG (M01) · XIN (M02) · NEWS (M03) · OCR (M04) · LBL (M05) · WGT (M06) · STY (M07) · CMP (M08) · ALR (M09) · DSH (M10) · COR (M11) · COW (M12) · OPS (M13) · INF (infrastructure) · REL (release).

---

## Phase 0 — Foundation (M00)

**T-FND-01 — Monorepo skeleton** · P0 · Deps: — · Refs: 15 §1, 20
Do: create the layout of 15 §1, `Makefile` (targets in 20 §3), `docker-compose.yml`/`.test.yml` with postgres(+pgvector), pgbouncer, redis, minio, mailhog, `.env.example`, pre-commit config, `ci.yml` skeleton.
Accept: `make dev` starts empty stack; `make lint` runs (even if trivial); CI workflow runs on a PR.

**T-FND-02 — Settings, clock, logging, ids, error envelope** · P0 · Deps: FND-01 · Refs: 12 §14, 16 §2/§9, DG-15
Do: `settings.py` (only reader of env), `common/clock.py` (`Clock`, `FakeClock`), structlog JSON with sanitiser, request-id propagation, `WarRoomError(code, http_status, message, details)` and global handler with the error codes of 12 §14.
Accept: unit tests: sanitiser strips keys matching `password|secret|token|authorization|whatsapp_number|api_key`; error envelope shape matches 12; no `datetime.now()` outside `clock.py` (lint rule).

**T-FND-03 — Schemas, models, migrations 0001–0006** · P0 · Deps: FND-02 · Refs: 11, 15 §3, 21 §3.1 (R-3, R-4, R-5, R-7)
Do: migrations `0001_extensions_schemas` … `0006_learning_audit` in REC-02 order (`campaigns` before `stories`); SQLAlchemy models per schema; apply resolutions R-3 (labels insert-only), R-4 (`new_stance` CHECK), R-7 (extra label fields).
Accept: TC-FND-07 (apply on empty DB, down/up cycle, schema equals models).

**T-FND-04 — RLS, DB roles/grants, triggers** · P0 · Deps: FND-03 · Refs: 13 §3.1, 16 §3.1, 15 §3 (0009–0011)
Do: migrations `0009_rls_policies` (`USING (client_id = current_setting('app.client_id')::uuid)`), `0010_db_roles_grants` (app role: no superuser, no BYPASSRLS, no UPDATE/DELETE on `mention_labels`, `audit.audit_log`), `0011_triggers` (`updated_at`, label-correction guard, audit append-only).
Accept: TC-SEC-05; raw SQL as client B returns zero client-A rows; app role cannot UPDATE labels (DB test).

**T-FND-05 — Tenancy context and repository base** · P0 · Deps: FND-04 · Refs: 16 §3.1, DG-02
Do: `ClientContext`, API dependency that opens a transaction with `set_config('app.client_id', :cid, true)`, repository base requiring `ClientContext`, Celery system-context builder, after-commit callback list (16 §3.3).
Accept: unit/integration tests; attempting a query without context fails; events publish only after commit.

**T-FND-06 — Auth: login, refresh rotation, logout, lockout, password reset** · P0 · Deps: FND-05 · Refs: 13 §1, 12 §1
Do: bcrypt cost ≥ 12; access 15 min; single-use rotating refresh tokens in `accounts.sessions` (reuse revokes the family); lockout after 5 failures; time-limited single-use reset link via SMTP.
Accept: TC-SEC-01; Mailhog receives reset email; admin can list/revoke sessions.

**T-FND-07 — PIN login, CM View session, device tokens** · P0 · Deps: FND-06 · Refs: 13 §1.5–1.6, 03 §5
Do: PIN login for CM View (30 min inactivity timeout), device tokens scoped to `/bigscreen/*` with optional CIDR allow-list, admin endpoints to create/revoke devices.
Accept: TC-SEC-02 (claims, expiry, tampering, device scope, bad CIDR rejected, CM timeout).

**T-FND-08 — RBAC guards and permission-matrix test generator** · P0 · Deps: FND-06 · Refs: 03 §2, 13 §2, 16 §7
Do: `@requires(Role.…)` route decorator, role hierarchy, matrix-driven test that iterates every route × role from 03 (+ reconciliation).
Accept: TC-FND-02; adding a route without a guard fails the test.

**T-FND-09 — Audit service and audit-coverage test** · P0 · Deps: FND-05 · Refs: 13 §5, DG-11, NFR-05
Do: `audit.record(ctx, action, entity_type, entity_id, old, new, reason)` writing in the caller's transaction; test that enumerates every non-GET route and fails if none writes audit.
Accept: TC-FND-03 (rolled back together with the mutation).

**T-FND-10 — WebSocket manager** · P0 · Deps: FND-06, FND-07 · Refs: 09 §2.7, 12 §13, 16 §8
Do: authenticated WS endpoint, per-client rooms, Redis pub/sub bridge, 20 s server heartbeat, event constants/payload models (`ws/events.py`).
Accept: two clients receive the same event; a client of another tenant does not; reconnect works.

**T-FND-11 — Celery app, queues, dead letters** · P0 · Deps: FND-02 · Refs: 15 §2, 16 §3.4, 19 §2.1
Do: `celery_app.py` with all queues of 15 §2, `acks_late=True`, `task_reject_on_worker_lost=True`, retry with backoff+jitter, `ops.dead_letters` (migration part of FND-15 if not yet present), beat schedule scaffold.
Accept: a failing task retries then lands in dead letters and raises a health warning; tasks are idempotent (run twice = no duplicates).

**T-FND-12 — Health endpoints and metrics** · P0 · Deps: FND-02 · Refs: 16 §9, 19 §6
Do: `/health`, Prometheus metrics (names in 16 §9), `source_health` writer helper.
Accept: metrics endpoint exposes the named series; health reflects DB/Redis.

**T-FND-13 — Seed loader, demo client, TVK placeholders, hard-coding check** · P0 · Deps: FND-03 · Refs: 15 §1, 16 DG-01, 18 §2
Do: `scripts/seed_client.py` (loads `backend/seeds/<client>/` YAML/CSV: client, entities, watch terms, tiers, outlets, thresholds, weights, recipients, model routes); `seeds/demo` ("Acme Beverages": competitor/partner/leadership summary vocabulary); `seeds/tvk` with `PLACEHOLDER` rows; `scripts/check_hardcoding.py` with banned-literal list.
Accept: TC-FND-05, TC-FND-06 (meta-test: introducing a banned literal fails CI); onboarding `demo` needs zero code diffs.

**T-FND-14 — Architectural CI gates and cross-client attack suite** · P0 · Deps: FND-08, FND-09, FND-13 · Refs: 15 §4, 16 §12, 18 §10
Do: `importlinter.ini` contracts (domain purity; `coworker` cannot import whatsapp/x/alerts-write; api not importing repositories; only `settings.py` reads env; only `llm/router.py` knows provider names; SQL only in repositories/migrations), OpenAPI drift check, i18n parity check, TC-FND-01 attack suite across all routes.
Accept: TC-FND-01 green; violating each contract in a scratch commit fails CI.

**T-FND-15 — Reconciliation addendum and migration 0007** · P0 · Deps: FND-03 · Refs: 21 §3.1–3.2
Do: write `docs/DOC-RECONCILIATION.md` (Part A permission additions/route list; Part B table/column additions; Part C decisions DEC-xx and gaps G-xx as used in docs 14–18); implement `0007_reconciliation_addendum` creating every table/column in 21 §3.2; update 11/12 accordingly.
Accept: migration applies/downgrades in CI; every table cited in docs 14–18 exists; the doc lists each DEC/G id used anywhere in the package with source.

## Phase 1 — Configuration and first ingestion

**T-CFG-01 — ConfigProvider** · P1 · Deps: FND-05 · Refs: 16 §3.5, DG-10
Do: typed snapshot accessors (`threshold`, `weights`, `vocab`), 30 s cache, invalidation via Redis pub/sub, `version_hash`; unknown key raises.
Accept: TC-FND-04 (change a threshold via API → used ≤ 30 s later; audit row).

**T-CFG-02 — Entities, relationship tags and history** · P1 · Deps: CFG-01 · Refs: FR-E14, BR-007/008, 17 M01
Do: CRUD + audit; tag change writes `entity_relationship_history` (`effective_from=now()`), applies to new Mentions only; `domain.relationship_at(entity, ts)`.
Accept: TC-US14 (old Mentions keep old tag).

**T-CFG-03 — Watch terms** · P1 · Deps: CFG-01 · Refs: BR-009, FR-B06
Do: CRUD with term types (name, nickname, hashtag, tanglish, tamil), instant effect, audit.
Accept: add/remove via API; takes effect without deploy; audit rows.

**T-CFG-04 — Standing tiers and members** · P1 · Deps: CFG-01 · Refs: FR-B06, story 6
Do: CRUD for the four tiers and members (platform + handle), immediate effect for new Mentions, audit.
Accept: TC-US06.

**T-CFG-05 — Outlets and candidates** · P1 · Deps: CFG-01 · Refs: 14 §3.4, BR-003, story 18
Do: outlet CRUD (partisan flag + note, feed/crawl URL, YouTube id, tier, reach, `poll_seconds`, `extraction_rules`, `requires_js`, `licensed_full_text`), candidate review list.
Accept: edits audited; partisan flag visible in API.

**T-CFG-06 — Entity accounts** · P1 · Deps: CFG-02 · Refs: 14 §2.3, 17 M10
Do: `config.entity_accounts` CRUD (official handles, MPs, MLAs, spokespeople per entity; client-side accounts for TVK-response detection).
Accept: accounts linked to entities; used by query builder (XIN-02) and Ally Mood (DSH-02).

**T-CFG-07 — Thresholds (typed)** · P1 · Deps: CFG-01 · Refs: 11 `alert_thresholds`, 16 §4, 07 Screen 7E
Do: typed/validated threshold keys with default shown; seeds for keys in 11/16 (`spike_z_score`, `spike_min_mentions`, `spike_window_minutes`, `spike_min_multiplier`, `big_screen_max_alerts`, `min_sample_size`, `per_post_cap`, `ally_threshold_*`, `response_trigger_*`, `severity_tier1_min`, `severity_tier2_min`, etc.).
Accept: out-of-range value rejected; change logged who/when/old/new.

**T-CFG-08 — Alert recipients** · P1 · Deps: CFG-01, FND-06 · Refs: 11, 07 Screen 7F, 03 §3
Do: routing by `alert_type` + `entity_id` + channel; `response_opportunity` routed to the configured lead.
Accept: routing resolved by data only; no phone number in code.

**T-CFG-09 — Event calendar** · P1 · Deps: CFG-01 · Refs: FR-C09, 07 Screen 7G
Do: CRUD for events (rally, assembly session, results day, other) with entity and optional baseline note.
Accept: event window queryable by the spike evaluator.

**T-CFG-10 — Weight profiles and model routes** · P1 · Deps: CFG-01 · Refs: 16 §4.1, 14 §8
Do: versioned `weight_profiles` (activate one), `model_routes` (capability → provider/model/prompt version/timeouts), seeds with the 16 §4.1 starting parameters.
Accept: weight change creates a new version; no model-name literal in code (CI grep).

**T-CFG-11 — Client settings, users, devices admin** · P1 · Deps: FND-07 · Refs: 03, 13 §1
Do: vocabulary, shifts, retention, flags, timezone; user and device management endpoints (system_admin).
Accept: vocabulary change visible via `t('vocab.…')`; user management role-guarded.

**T-CFG-12 — [OPTIONAL] CSV import endpoint** · P1 · Deps: CFG-03, CFG-04, CFG-05 · Refs: 17 M01
Do: CSV import for watch terms/tiers/outlets. **Optional**; seeds cover Phase 1. Build only if requested.
Accept: row-level validation errors; audit entry per import.

**T-CFG-13 — Admin configuration screens 7A–7G** · P1 · Deps: CFG-02…CFG-09 · Refs: 07 Screen 7, 08
Do: Next.js screens for each tab with loading/empty/error/saved states, inline "Saved" with timestamp, audit trail visible inline.
Accept: Playwright flow per tab; en and ta; permission: War Room Admin/System Admin only.

**T-XIN-01 — XSourceAdapter, PollingAdapter, mock X server** · P1 · Deps: CFG-01 · Refs: 14 §2.1, DEC-18, 18 §2
Do: protocol with `fetch_recent`, `fetch_archive`, `stream` (raises `NotSupported` on polling), `lookup_users`, `user_timeline`, `usage`; `StreamAdapter` and `VendorAdapter` stubs (`NotConfigured`); `backend/tests/mocks/x_server.py` with 429/401/5xx injection and reads accounting; config schema rejects a list of X credential sets.
Accept: `test_single_credential_set` (BR-054); adapter returns `Ok(empty)` distinct from `Err`.

**T-XIN-02 — X query builder** · P1 · Deps: XIN-01, CFG-03, CFG-06 · Refs: 14 §2.3
Do: generate queries from watch terms and entity accounts (each ≤ query length limit `[VERIFY]`), priority tiers P0/P1/P2 derived from `is_primary` and `relationship_tag`.
Accept: query set regenerates on config change; no hand-written query strings.

**T-XIN-03 — Poll scheduler, cursors, dedupe** · P1 · Deps: XIN-02, FND-11 · Refs: 14 §2.3, BR-002
Do: per-priority poll intervals (defaults 60/120/300 s; client-side accounts 120 s), `since_id` cursors in `ops.ingestion_cursors`, unique key `(client_id, source_type, external_id)`, 24 h seen-ID set in Redis.
Accept: TC-PIPE-01 (same post twice → one Mention, no second cost recorded).

**T-XIN-04 — Authors upsert and raw archive** · P1 · Deps: XIN-03 · Refs: 14 §2.2, 11/B1
Do: upsert `core.authors` (id, handle, name, verified, followers, following, post count, `account_created_at`), raw JSON to `wr-raw` with `raw_ref`, no more author data than listed (BR-047).
Accept: raw payload retrievable; no private-citizen-centred view exists.

**T-XIN-05 — Usage counters, 50/80/95 % alerts, sampling mode** · P1 · Deps: XIN-03 · Refs: FR-I30, BR-040, BR-053, 14 §2.3
Do: `ops.usage_counters` increments, admin notifications once each at 50/80/95 %, sampling mode throttles P2 then P1 (never P0 or client-side accounts), `ops.sampling_windows`, `is_sampled` propagation.
Accept: TC-US30, TC-PIPE-12.

**T-XIN-06 — Client-side (TVK) response account polling** · P1 · Deps: XIN-03, CFG-06 · Refs: story 31/32, 14 §2.1
Do: poll `user_timeline` of configured client-side accounts every 120 s; store their posts as Mentions flagged client-side.
Accept: a client-side post appears within the interval; not counted as "others reacting".

**T-XIN-07 — Engagement refresh snapshots** · P1 · Deps: XIN-03 · Refs: 16 §4.1
Do: for X Mentions with `account_influence_without_velocity ≥ engagement_refresh_min_influence` (0.5) re-read at +5, +10, +20 min into `core.engagement_snapshots`.
Accept: snapshots written only for candidates; velocity derivable; read cost counted.

**T-XIN-08 — Backfill job** · P1 · Deps: XIN-03, XIN-05 · Refs: 14 §2.4, G-15
Do: 6-hour windows per query, resumable, limited by `X_BACKFILL_DAILY_SHARE`, `ingest_mode='backfill'` (never fires live alerts; only items within `live_alert_window_minutes` alert), `ops.backfill_state` partial/complete, dashboard text "History available from …" when partial.
Accept: TC-PIPE-09.

**T-XIN-09 — X source health and compliance sync** · P1 · Deps: XIN-03, FND-12 · Refs: FR-I28, 14 §2.7–2.8
Do: failure table behaviour (401/403, 429, network/5xx, cap, empty-vs-error), `compliance_sync` marking `retention_class='deleted_by_source'` and clearing snippet/full text.
Accept: TC-US28 (X part): break mock → "data delayed since HH:MM"; empty healthy run is not "delayed".

**T-XIN-10 — Replay harness** · P1 · Deps: XIN-01, FND-11 · Refs: 18 §5
Do: `scripts/replay_scenario.py --scenario <file> [--real-llm] [--real-wa-sandbox]` driving the real pipeline under `FakeClock`/real time with timestamp assertions; scenario YAML schema.
Accept: a trivial scenario replays; timestamps asserted.

**T-LBL-01 — Text normalisation and Tamil numerals** · P1 · Deps: FND-02 · Refs: BR-014, 14 §3.6
Do: NFC normalisation, `text_hash`, `to_mention_text()` (snippet ≤ 300 chars, no full text unless licensed), `app/nlp/tamil_numbers.py` (௧௨௩, "௫ கோடி").
Accept: TC-PIPE-04, TC-PIPE-11 unit tests.

**T-LBL-02 — Language detector** · P1 · Deps: LBL-01 · Refs: BR-013, DEC-14, R-2
Do: own detector for Tamil script, English, Tanglish, mixed; X language tag ignored; per-language accuracy report on the Tamil corpus.
Accept: TC-PIPE-03 (fixture where X says `en` for Tanglish is still detected as Tanglish).

**T-LBL-03 — Watch-term matcher, wide-net filter, raw storage** · P1 · Deps: LBL-02, CFG-03, XIN-03 · Refs: BR-001, 17 M05
Do: Aho-Corasick over normalised terms (Tamil, Tanglish variants, hashtags; fuzzy only for listed Tanglish variants); topic-similarity keep (`topic_similarity_min` 0.6); unmatched stored raw with `filter_matched=false` and eligible for weekly audit; `entities_matched` stored.
Accept: TC-PIPE-02.

**T-LBL-04 — Embeddings worker** · P1 · Deps: FND-11 · Refs: 14 §9, DEC-13
Do: self-hosted multilingual sentence-transformers in the `embed` queue; batch 32, max 512 tokens; `embedding_model` recorded; HNSW index; model change = re-embed job; no mixed-model comparison.
Accept: embeddings written for kept items; queries filter by identical `embedding_model`.

## Phase 2 — Label → weigh → influence alert

**T-LBL-05 — Cheap labeller, prompts, schema** · P2 · Deps: LBL-03, CFG-10 · Refs: 16 §5, BR-010/011, DG-07/09
Do: prompt files `app/prompts/label/v1.md` (hash suffix in `prompt_version`); Pydantic schema `{entities[…], language, quotes[…]}`; candidate entities from `entities_matched`; reason required (≤ 240 chars); unknown `entity_id` dropped and logged; content passed only in delimited content blocks; JSON repair retry once; insert-only label writes.
Accept: TC-PIPE-05; TC-SEC-03 (injection corpus through the labeller).

**T-LBL-06 — Escalation and human review queue** · P2 · Deps: LBL-05 · Refs: BR-012, 16 §5
Do: below `label_confidence_escalate_below` → strong route; below `label_confidence_human_below` → review queue (Stories filter "needs review").
Accept: TC-PIPE-06; TC-PIPE-08 (strong route used only in listed cases).

**T-LBL-07 — Quote and speaker extraction** · P2 · Deps: LBL-05 · Refs: story 33, FR-J33
Do: extract quoted lines and speaker where known into `core.mention_quotes`; hook for NEWS-08.
Accept: TC-US33 (quote part).

**T-LBL-08 — Gold-set few-shot selection** · P2 · Deps: LBL-05 · Refs: 16 §5
Do: select ≤ 6 examples from `learning.gold_set` (same language, per entity, most recent corrections first) into the label prompt; seed corpus ≥ 300 Tamil items until the real gold set arrives.
Accept: TC-PIPE-14 inputs; selection deterministic.

**T-LBL-09 — Rules-only influence fallback** · P2 · Deps: LBL-05, WGT-01 · Refs: 14 §12, 18 §5
Do: when the cheap LLM route is down, verified + tier + keyword stance heuristic still triggers the influence alert, flagged "unlabelled by AI".
Accept: TC-MP2 variant (LLM down).

**T-LBL-10 — Pipeline latency stamps** · P2 · Deps: FND-11 · Refs: 18 TC-NFR-01
Do: write `ops.pipeline_latency` at each stage; metrics `post_to_screen_seconds`.
Accept: p95 queryable per stage.

**T-WGT-01 — Influence formula** · P2 · Deps: CFG-10 · Refs: 16 §4.1, BR-016/018
Do: `domain/influence.py`: account and outlet influence, bot down-weight, all parameters from the active weight profile; stored with `weight_version`.
Accept: unit tests table-driven; partisan outlet weighted lower than independent equivalent (TC-US18 part).

**T-WGT-02 — Per-post cap** · P2 · Deps: — · Refs: 16 §4.2, DEC-07, BR-017
Do: `domain/capping.py` iterative cap (≤ 10 passes); `InsufficientSample` when N·c < 1 / N < 20.
Accept: property tests (shares sum to 1; none exceeds cap) — TC-US07.

**T-WGT-03 — Sentiment aggregation and change detection** · P2 · Deps: WGT-02 · Refs: 16 §4.3, DEC-01/06, BR-037…040
Do: polarity (sarcastic uses `sarcasm_target_stance`), weighted/raw shares, `has_enough_data`, top-3 contributors, `is_sampled`, Wilson 95 % interval, two-proportion z-test (|z| ≥ 1.96, both windows ≥ min sample) for "change".
Accept: TC-NFR-06 (vs scipy reference).

**T-WGT-04 — Bot suspicion heuristic** · P2 · Deps: XIN-04 · Refs: 16 §4.8 `[PROPOSED]`, BR-018
Do: weighted score from account age, follower ratio, post rate, default profile, handle pattern, language mismatches; weights are data; UI text "suspected bot" only.
Accept: unit tests; never labelled as fact.

**T-WGT-05 — Weights writer and author resolution** · P2 · Deps: WGT-01, CFG-04, CFG-05 · Refs: 11 `mention_weights`, 14 §2.2
Do: resolve tier via `tier_members` (platform + handle), outlet tier via `outlets`; write `core.mention_weights` with components and `capped_contribution`.
Accept: weights reproducible by `scripts/reproduce_score.py` (TC-NFR-05).

**T-WGT-06 — Rollup and effective/model-only views** · P2 · Deps: WGT-05 · Refs: 17 M06, 18 §7
Do: materialised `core.sentiment_rollup_5m` refreshed every minute; `labels_scope(view)` implementing `view=effective|model_only` once in repository query builders.
Accept: dashboard summary p95 < 400 ms on large fixture; both views return correct counts after a correction.

**T-WGT-07 — Weighted/raw/top-3 payloads** · P2 · Deps: WGT-03, WGT-06 · Refs: DG-06, FR-B07
Do: API payloads with weighted, raw, mention_count, top contributors, `corrected_count`.
Accept: property test: every percentage payload carries denominator (TC-NFR-06).

**T-ALR-01 — Severity scorer** · P2 · Deps: WGT-01 · Refs: 16 §4.4, DEC-05
Do: `domain/severity.py` = influence × negativity × velocity; tier thresholds from config; tier 3 produces no alert row.
Accept: table-driven unit tests.

**T-ALR-02 — Alert repository, dedupe and tiering** · P2 · Deps: ALR-01, FND-05 · Refs: 11 `alerts`, FR-B08, BR-023/024
Do: alert persistence with `dedupe_key` grouping, tier assignment, big-screen cap query (`big_screen_max_alerts`), `alert_events`.
Accept: TC-US08 (6 active → ≤ 3 on big screen; related posts grouped).

**T-ALR-03 — Influence trigger evaluator (incl. instant path)** · P2 · Deps: ALR-02, LBL-05, WGT-05 · Refs: FR-B05, BR-027, DEC-20, 16 §4.4, G-23
Do: single negative post from a top-tier verified account fires Tier 1 at ingestion when `account_influence ≥ 0.85`, negative about a client-primary entity and not satire (velocity term = 1 for first evaluation); reason string "verified, 1.2M followers, national media tier, 4,000 reposts in 20 min" with engagement known so far; `trigger_at = published_at`.
Accept: TC-MP2 assertions 1–4, variant "40-follower account ⇒ no Tier 1", no spike alert in window.

## Phase 3 — Delivery

**T-ALR-08 — Acknowledge and assign** · P3 · Deps: ALR-02, FND-10 · Refs: FR-B08, BR-025, 12 §6
Do: endpoints, audit, WS events `alert.acknowledged`/`alert.assigned`, flash cleared for all clients, "taken by".
Accept: TC-US08 (2 WS clients).

**T-ALR-09 — WhatsApp sender, templates, webhook, retry** · P3 · Deps: ALR-02, CFG-08 · Refs: 14 §7, FR-I29, BR-043, DEC-04
Do: `services/whatsapp_sender.py` as the only module with credentials; mock `wa_server.py`; templates `wr_alert_tier1`, `wr_response_opportunity`, `wr_overnight_digest` (English + Tamil; no raw post text); `POST /webhooks/whatsapp` signature-verified, duplicate/out-of-order tolerant; statuses accepted→sent→delivered→read|failed; retries with `WHATSAPP_RETRY_BACKOFF_SECONDS × 2^n`; failure ⇒ `alert_events('whatsapp_failed')`, lead notification, in-app badge.
Accept: TC-US29, TC-SEC-08, SLA clock `accepted_at − trigger_at ≤ 300 s` measurable; co-worker package cannot import the sender (import-linter).

**T-DSH-01 — Dashboard and big-screen summary endpoints** · P3 · Deps: WGT-07, ALR-02 · Refs: 12 §2, 07 Screens 1/10
Do: `/dashboard/summary`, `/bigscreen/summary`, entity drill-down; staleness (`is_stale`), source health list.
Accept: payloads match 12 §2; "not enough data" at N=29, percentage at N=30 (TC-US01).

**T-DSH-05 — WebSocket event wiring** · P3 · Deps: FND-10, DSH-01 · Refs: 12 §13
Do: publish after-commit events: `alert.new`, `sentiment.update`, `story.new/rising`, `source.health`, `presentation_mode.toggle`, `coworker.answer`, `upload.status`.
Accept: each event emitted once per change; no publish inside uncommitted transactions.

**T-DSH-08 — Number components and lint rule** · P3 · Deps: FND-01 · Refs: 16 §8
Do: `StatWithDenominator`, `NotEnoughData`, `WeightedRaw`, `SampledBadge`, `StaleStamp`; ESLint rule `warroom/no-bare-percent`; `evidenceQuery` prop on every number.
Accept: component tests for denominator, not-enough-data, sampled, stale (18 §8).

**T-DSH-09 — i18n, vocabulary, Tamil rendering** · P3 · Deps: FND-01, CFG-11 · Refs: 16 §8, DG-14
Do: `en.json`/`ta.json` parity check, `vocab.*` placeholders from client config, Noto Sans Tamil via `next/font`, `lang="ta"`, Intl formatting with client timezone.
Accept: TC-FND-05 UI part; TC-NFR-10 (key parity, Tamil snapshots).

**T-DSH-06 — Dashboard screen** · P3 · Deps: DSH-01, DSH-05, DSH-08 · Refs: 07 Screen 1, 08
Do: source-health banner, entity cards (weighted + "of N"), Ally Mood panel (empty state until DSH-02), top-3 alerts with flash, rising Stories; loading/empty/error/success states; reconnect banner.
Accept: Playwright flow; en/ta; TC-US03 (every number opens its evidence).

**T-ALR-12 — Alert screens** · P3 · Deps: ALR-08, ALR-02, DSH-08 · Refs: 07 Screen 3, 06 §2
Do: alert list/detail (type badge, tier, explanation slot, evidence, acknowledge/assign, history).
Accept: TC-US08 E2E.

**T-DSH-07 — Big-screen kiosk page** · P3 · Deps: DSH-01, DSH-05, FND-07 · Refs: 07 Screen 10, 16 §8, 19 §10
Do: `/bigscreen` with device-token auth, no pointer UI, 60 s polling fallback, 3-minute watchdog with "data delayed since HH:MM", Tier-1 flash tied to `unacknowledgedTier1Ids`, presentation badge slot.
Accept: TC-NFR-09 (WS kill → recovery; 24 h soak memory growth ≤ 20 %).

**T-OPS-01 — Source health aggregator and endpoints** · P3 · Deps: FND-12 · Refs: FR-I28, 12 §12, 14 §12
Do: `ops.source_health` upserts from every integration; admin endpoints for sources, API usage, WhatsApp log.
Accept: TC-US28 for each source type; error never renders as "no new items".

## Phase 4 — Breadth of sources and Stories

**T-NEWS-01 — Feed and sitemap source** · P4 · Deps: CFG-05, FND-11 · Refs: 14 §3.1–3.2
Do: `NewsSource` protocol, `FeedSource` (feedparser; ETag/Last-Modified; news sitemaps), outlet polling default 300 s.
Accept: mock news site feeds ingested; cursor in `ops.ingestion_cursors`.

**T-NEWS-02 — Crawler** · P4 · Deps: NEWS-01 · Refs: BR-005, 14 §3.3
Do: `robots.txt` (cached 24 h; disallow disables outlet and notifies admin), UA `WarRoomBot/1.0 (+CRAWLER_CONTACT_URL)`, per-host token bucket (1 req/5 s, concurrency 1, jitter, `Retry-After`, `Crawl-delay`), blocked site ⇒ `down` reason `blocked` (no evasion).
Accept: TC-PIPE-10.

**T-NEWS-03 — Extraction and storage limits** · P4 · Deps: NEWS-02, LBL-01 · Refs: BR-006, 14 §3.3
Do: trafilatura extraction, per-outlet CSS selector overrides, snippet ≤ 300 chars, full text only with `licensed_full_text`; layout-change detection.
Accept: TC-PIPE-11; selector break ⇒ `delayed/down` with reason.

**T-NEWS-04 — [OPTIONAL BY OUTLET] Playwright pool** · P4 · Deps: NEWS-03 · Refs: 14 §3.3
Do: separate worker image/pool for `requires_js=true` outlets.
Accept: JS-only mock outlet ingested; main workers unaffected.

**T-NEWS-05 — YouTube source** · P4 · Deps: CFG-05 · Refs: 14 §4, G-20
Do: Data API v3 uploads playlist every 5 min; Mention from title+description+time; captions only where permitted; `text_source='metadata_only'` otherwise; quota tracking with 80 % warning.
Accept: TC-US33 (video part).

**T-NEWS-06 — Search discovery (off by default)** · P4 · Deps: NEWS-01 · Refs: 14 §3.4
Do: `SearchDiscovery` adapters (google_cse, serpapi), every 30 min from top watch terms, new domains create `outlets_candidates` rows for staff review.
Accept: disabled unless `SEARCH_PROVIDER` set; candidate list shows new domains.

**T-NEWS-07 — Outlet health and syndication dedupe** · P4 · Deps: NEWS-01 · Refs: 14 §3.5, FR-I28
Do: health per outlet (error vs healthy-empty distinct); `text_hash` + embedding ≥ 0.95 within 24 h linked as duplicate but still counted as an outlet in reach.
Accept: TC-US28 outlet part; "9 outlets" counts syndicated copies once per outlet.

**T-NEWS-08 — Quote extraction hook** · P4 · Deps: NEWS-03, LBL-07 · Refs: story 33
Do: pass news/YouTube text through quote/speaker extraction; link Mention to article/video.
Accept: TC-US33.

**T-OCR-01 — OCR engine and Tesseract** · P4 · Deps: FND-11 · Refs: 14 §6, I-07
Do: `OcrEngine` protocol (`read(image, langs) → text, per-line boxes, per-line confidence`), `TesseractEngine` (tam+eng; OpenCV deskew/denoise/binarise; upscale to 300 dpi minimum), `CloudOcrEngine` slot.
Accept: TC-PIPE-13 (confidence flag).

**T-OCR-02 — OCR bake-off (decision task)** · P4 · Deps: OCR-01 · Refs: 14 §6, G-05
Do: run Tesseract and (optionally) cloud OCR on 50 real/representative Tamil newsprint pages; report character error rate and headline recall against hand-typed ground truth; recommend engine in `docs/OPEN-QUESTIONS.md`.
Accept: report committed; `OCR_ENGINE` default set from the result; **the decision belongs to the CTO**.

**T-OCR-03 — Upload API** · P4 · Deps: OCR-01, FND-05 · Refs: 14 §6, 12 §9
Do: JPG/PNG/PDF, ≤ 15 MB, ≤ 20 pages, virus scan, sha-256 dedupe (`DUPLICATE_UPLOAD`), `core.uploads` status, storage in `wr-uploads`, signed URLs.
Accept: TC-SEC-06 (type/size/virus/path traversal), TC-US16 (duplicate rejected).

**T-OCR-04 — Clipping flow and OCR edit** · P4 · Deps: OCR-03, LBL-03 · Refs: 04 §2D, FR-F16, 08 §3.6
Do: OCR task → Mention (outlet/date/page from form win over auto-fill), low-confidence words highlighted and "check scan", manual correction before confirm, hand-off to `s2_filter`.
Accept: TC-US16, TC-PIPE-13.

**T-OCR-05 — E-paper provider framework** · P4 · Deps: OCR-04 · Refs: 14 §5, FR-F17, G-02
Do: `EpaperProvider` per publisher, `config.epaper_subscriptions` (`automation_permitted` default false), manual-upload path first, article segmentation, Mention tagged `epaper` with signed page image, delay detection.
Accept: TC-US17 (provider refuses to run when `automation_permitted=false`).

**T-OCR-06 — Upload screen 6** · P4 · Deps: OCR-03, DSH-09 · Refs: 07 Screen 6, 08 §1.4
Do: drag-drop/camera, per-upload form, per-file status (processing / OCR complete / Mention created / check scan / error).
Accept: Playwright upload flow; mobile camera option present.

**T-STY-01 — Story matcher** · P4 · Deps: LBL-04, WGT-05 · Refs: BR-020, 17 M07
Do: match by embedding + entity overlap + time window + hashtag/URL overlap, centroid update, per-client advisory lock around match-or-create.
Accept: TC-US15 (9 outlets + 1,200 posts → one Story); concurrent ingest creates no duplicate Story.

**T-STY-02 — Reach counters and auto-title** · P4 · Deps: STY-01 · Refs: BR-021, FR-F15
Do: counters by source type, "N outlets, N posts" strings, auto-title.
Accept: TC-US15 text.

**T-STY-03 — Unattributed single-item Stories** · P4 · Deps: STY-01, LBL-05 · Refs: BR-015/022, FR-F19
Do: anonymous insinuation and rumour create a Story of one marked "unattributed", never suppressed.
Accept: TC-US19.

**T-STY-04 — Merge and split** · P4 · Deps: STY-01 · Refs: 12 §5, FR-F15
Do: merge/split endpoints with role guard and audit, `merged_into` handling.
Accept: TC-US15 (merge/split audited).

**T-STY-06 — Story screens** · P4 · Deps: STY-02, STY-04, DSH-08 · Refs: 07 Screen 2
Do: story list/detail (header, timeline, mentions list with label+reason+correct button, staff actions).
Accept: Playwright flow; TC-US03.

**T-OPS-02 — Admin health screen** · P4 · Deps: OPS-01 · Refs: 07 Screen 8
Do: source status table, X usage bar with 50/80/95 markers, WhatsApp delivery log.
Accept: TC-US30 UI part.

## Phase 5 — Detection

**T-ALR-04 — Spike evaluator** · P5 · Deps: ALR-02, CFG-09, WGT-03 · Refs: FR-C09, BR-026, 16 §4.5
Do: `domain/spike.py`: baseline over `spike_baseline_days`, `σ_eff = max(σ, sqrt(max(μ,1)), 1)`, four conditions held ≥ `spike_window_minutes`, event baseline from the last ≤ 5 events of the same type (fallback marked `standard_fallback`), cold-start rule (< 7 days) with "provisional baseline".
Accept: TC-US09 (table-driven; each missing condition ⇒ no alert).

**T-ALR-05 — Ally mood evaluator** · P5 · Deps: ALR-02, CFG-06, CFG-02 · Refs: FR-E13, BR-028
Do: ally/supporting parties' public figures' stance toward the client; alert at `ally_threshold_*`; uses `relationship_at`.
Accept: TC-US13 (`ally_discontent` replay).

**T-ALR-06 — Response-opportunity evaluator** · P5 · Deps: ALR-02, STY-01, XIN-06, ALR-09 · Refs: FR-J31, BR-029, DEC-08/09, 16 §4.7
Do: `domain/response_trigger.py`; fire when attack + condemnation from others + no client-side response and threshold crossed; WhatsApp `wr_response_opportunity` to the configured recipient with title, ≥ 3 who-is-reacting names, "no TVK response yet"; close on client-side post ("responded by … at HH:MM") and stop alerting; 60 s sweep of open candidates; rules-only classification when LLM down.
Accept: **TC-MP1 / TC-US31** all assertions and variants.

**T-ALR-07 — Campaign alert creation** · P5 · Deps: CMP-02, ALR-02 · Refs: FR-D10, DG-13
Do: alert from campaign confidence ≥ `campaign_confidence_alert_min`; text never names an acting party.
Accept: TC-US11 (reads "possible coordination").

**T-ALR-11 — Alert auto-resolution** · P5 · Deps: ALR-04…ALR-06 · Refs: 17 M09
Do: resolve when conditions subside or the opportunity is answered; history preserved.
Accept: resolved alerts leave active views; audit trail intact.

**T-CMP-01 — Candidate clustering** · P5 · Deps: LBL-04, XIN-04 · Refs: 16 §4.6
Do: ≥ `campaign_min_accounts` distinct authors with MinHash (char 5-grams) ≥ `campaign_text_similarity_min`, confirmed by embedding cosine, within `campaign_window_minutes`; or shared hashtag where ≥ 50 % of authors have no other posts about the entity.
Accept: TC-US10 clustering part (`surrogate_wave`).

**T-CMP-02 — Campaign scoring and persistence** · P5 · Deps: CMP-01, WGT-04 · Refs: 16 §4.6, BR-032
Do: signals `S_text, S_age, S_time, S_boost, S_hashtag`, weighted confidence (low < 0.5, medium 0.5–0.75, high > 0.75), persist `core.campaigns` and `campaign_accounts` (handle, age, followers, bot flag), link to Story.
Accept: TC-US10 (accounts, ages, phrasing, timing series, confidence).

**T-CMP-03 — Confirm and dismiss** · P5 · Deps: CMP-02, FND-08 · Refs: FR-D11, BR-030/031/045, DG-13
Do: confirm requires `war_room_lead`+, logs name/reason/timestamp; DB CHECK prevents `confirmed` without `confirmed_by` of role ≥ lead; error `CAMPAIGN_NOT_CONFIRMED`; dismiss with reason.
Accept: TC-US11 (team_member ⇒ 403; DB check blocks bypass).

**T-CMP-04 — Spread timeline** · P5 · Deps: CMP-03 · Refs: FR-D12
Do: mentions/hour, new accounts joining, client response Stories on one timeline.
Accept: TC-US12.

**T-CMP-05 — Campaign screens** · P5 · Deps: CMP-03, CMP-04 · Refs: 07 Screen 4
Do: header status badge, evidence panel, spread timeline, lead-only attribution form (reason required).
Accept: TC-US11/12 E2E; button hidden for non-leads.

**T-STY-05 — Response queue** · P5 · Deps: ALR-06, STY-01 · Refs: FR-J32, 12 §10
Do: queue computation, suggested response type, who has reacted, time since start, client response status; mark acted/not needed (logged; feeds prioritisation).
Accept: TC-US32.

**T-DSH-02 — Ally Mood panel endpoint** · P5 · Deps: ALR-05 · Refs: FR-E13, 07 §1C
Do: per ally/supporting party mood, people and posts, alert link.
Accept: TC-US13 UI.

## Phase 6 — Co-worker, corrections, CM view

**T-COR-01 — Correction submission and bias guard** · P6 · Deps: LBL-05, WGT-06 · Refs: FR-H24/H26, BR-033/034, DEC-02/03
Do: `domain/bias_guard.requires_second_review` (softening a negative label on a client-primary entity, or touching a high-severity Story); correction rows with reason ≥ 10 chars; original retained; notification to second reviewer.
Accept: TC-US24, TC-US26 (aggregates unchanged until approved).

**T-COR-02 — Second review** · P6 · Deps: COR-01 · Refs: 12 §4, 08 §3.7
Do: pending-review list, approve/reject with reason; reviewer ≠ corrector (DB CHECK + service).
Accept: TC-US26.

**T-COR-03 — Effective labels and aggregate invalidation** · P6 · Deps: COR-01, WGT-06 · Refs: BR-035
Do: views for effective labels; rollup recompute for affected buckets on applied correction; model-only view always available; "corrected by" display.
Accept: TC-US24 (aggregates update; corrected counts identifiable; label rows never updated).

**T-COR-04 — Gold-set ingestion** · P6 · Deps: COR-03 · Refs: FR-H25, BR-036
Do: every applied correction + manual/imported labels into `learning.gold_set`.
Accept: TC-US25.

**T-COR-05 — Weekly metrics job** · P6 · Deps: COR-04 · Refs: 16 §4.9, FR-H25
Do: F1 per language/entity, correction rate per 1,000 labels, model-only vs corrected, per-person/per-entity correction-rate tracking.
Accept: TC-PIPE-14 (hand-computed fixture).

**T-COR-06 — Staging engine and StagePolicy** · P6 · Deps: COR-05 · Refs: FR-K34/K35, BR-046, 17 §4.1
Do: promote after `stage_promote_weeks`; demote one stage immediately below bar with reason and notification; NULL bar ⇒ stays Assist; `StagePolicy` consumed by labelling, alerting, briefs.
Accept: TC-US34, TC-US35.

**T-COR-07 — Output corrections** · P6 · Deps: COR-01 · Refs: PRD "Human correction", 15 `output_corrections.py`
Do: correct co-worker answers, alerts, severity, campaign flags with reason; original kept.
Accept: original visible in history; audit rows.

**T-COR-08 — Correction panel and accuracy screens** · P6 · Deps: COR-03, COR-06 · Refs: 07 Screen 9, 08 §8–9
Do: correction panel (min 10-char reason, second-review notice text of 08 §3.7), stage dashboard, weekly accuracy chart, correction-rate chart.
Accept: Playwright flow; TC-US24/34/35 UI.

**T-COW-01 — Co-worker tools (read-only)** · P6 · Deps: WGT-06, STY-01, ALR-02 · Refs: 16 §6, BR-043
Do: `search_mentions`, `count_mentions`, `compare_periods`, `get_story`, `list_alerts`, `get_alert`, `list_open_opportunities`, `entity_info`, `recall_memory`, `remember`; each logs its query id; no tool can send, post, change config, correct labels or confirm.
Accept: tool-surface test; import-linter contract.

**T-COW-02 — Agent loop and grounding validator** · P6 · Deps: COW-01, CFG-10 · Refs: BR-041, 13 §4.2, 16 §6
Do: JSON `{answer_text, claims[…], language}`; `grounding.validate()` (mention ids came from this client's tool calls; numbers equal query results — exact counts, ±0.1 pt percentages); one auto-retry then "I can't verify this from the data"; "not enough data (N=…)" under min sample; answer in question language (Tanglish ⇒ Tamil); cap 6 tool rounds.
Accept: TC-US20; no ungrounded number passes.

**T-COW-03 — Streaming, logging, rating, suggestions** · P6 · Deps: COW-02, FND-10 · Refs: 12 §8, 08 §1.3
Do: `/coworker/chat` + WS streaming, `learning.coworker_interactions` logging (model/prompt version, sources, latency), rating endpoint, contextual suggestions; "Taking a moment…" after 10 s; LLM-down message per 08 §3.2.
Accept: TC-US20 first token ≤ 10 s (eval level); TC-NFR-08.

**T-COW-04 — Alert explanations (async)** · P6 · Deps: COW-02, ALR-02 · Refs: FR-G21
Do: two-line explanation on every fired alert with evidence links; `explanation_status`; unavailable text when LLM down.
Accept: TC-US21.

**T-COW-05 — CM summary, handover, ally-blame gate, briefs** · P6 · Deps: COW-02, ALR-08 · Refs: FR-G22, BR-044, 04 §6–7
Do: ≤ 3 changes with one-line cause and evidence link; `domain/ally_blame.detect()` (over-inclusive); `needs_ally_review`, send endpoint 409 `REVIEW_REQUIRED` until a lead reviews; handover draft (open alerts, owners, watch items), editable before send; briefs lifecycle; English/Tamil.
Accept: TC-US02, TC-US22.

**T-COW-06 — Memory** · P6 · Deps: COW-02 · Refs: PRD "Memory", 16 §6
Do: `remember()` rejects private individuals (`redaction.contains_private_author()`); rows correctable by soft-delete + `superseded_by`.
Accept: private-handle test refused; correction supersedes without deleting.

**T-COW-07 — Eval harness and datasets** · P6 · Deps: COW-02 · Refs: 18 §6
Do: `backend/evals/` runner, ≥ 60 Q&A items (≥ 20 Tamil, ≥ 10 Tanglish, ≥ 5 must-say-unknown) against a frozen snapshot, scores for accuracy, citation correctness, unknown-handling, language match, latency; `learning.eval_runs`; `evals.yml`.
Accept: CI fails if citation correctness < 0.95 or any ungrounded number or accuracy drop > 2 pts.

**T-COW-08 — Injection test suite** · P6 · Deps: LBL-05, COW-02 · Refs: BR-042, TC-SEC-03
Do: corpus of instruction-like posts/articles (Tamil and English) through labeller, explainer, co-worker, briefs.
Accept: TC-SEC-03; any new LLM call must add a test here.

**T-COW-09 — Co-worker chat and briefs UI** · P6 · Deps: COW-03, COW-05 · Refs: 07 Screen 5, 08
Do: chat with citations and sample-size note, thumbs, suggested questions, briefs/handover editor, persistent chat button.
Accept: Playwright; Tamil question ⇒ Tamil answer in E2E.

**T-DSH-03 — Presentation-safe mode and redaction** · P6 · Deps: FND-10, DSH-01 · Refs: FR-A04, BR-049, DEC-11, 16 §7
Do: `domain/redaction.apply(ctx, model)` on every Mention-bearing response when mode is active or caller is device/CM View; toggle reaches all displays ≤ 2 s; visible badge.
Accept: TC-US04 (property test over all Mention-bearing responses; no private handles, internal notes, correction details).

**T-DSH-04 — CM summary pages and evidence** · P6 · Deps: COW-05, DSH-03 · Refs: 07 Screen 11, FR-A02
Do: `/cm-summary` read-only, "reviewed by" badge, evidence endpoint `/cm-summary/:id/evidence/:idx`, no navigation chrome beyond language switch.
Accept: TC-US02 E2E.

**T-DSH-10 — Notifications** · P6 · Deps: COR-02, ALR-09 · Refs: 15 `notifications.py`
Do: in-app notification model/endpoints for second review, WhatsApp undelivered, stage demotion, X usage.
Accept: each trigger creates a notification once.

## Phase 7 — Hardening, infrastructure and release

**T-OPS-03 — Weekly filter audit** · P7 · Deps: LBL-03 · Refs: FR-I27, 04 §9
Do: sampler stratified by source/language/entity, reviewer UI, missed-item-rate, admin chart.
Accept: TC-US27.

**T-OPS-04 — Audit-log viewer** · P7 · Deps: FND-09 · Refs: 07 admin, 13 §5.4
Do: read-only viewer for configuration changes, corrections, campaign confirmations, alert actions.
Accept: visible only to war_room_admin/system_admin; append-only verified.

**T-OPS-05 — Retention jobs and ingestion gate** · P7 · Deps: CFG-11, XIN-03 · Refs: BR-048, 19 §8
Do: `periodic_retention`, S3 lifecycle mirroring `retention_days_raw`, **ingestion refuses to start while retention unset**.
Accept: TC-NFR-04.

**T-OPS-06 — Dead-letter view** · P7 · Deps: FND-11 · Refs: 16 §3.4
Do: admin view of `ops.dead_letters`.
Accept: poison message visible with retry/dismiss audit.

**T-INF-01 — Terraform network, RDS, Redis, S3** · P7 · Deps: — · Refs: 19 §3
Do: modules per 19 with staging env first; PITR, AOF, SSE, public-block, lifecycle.
Accept: `terraform plan` clean; no public DB endpoint.

**T-INF-02 — ECS services and autoscaling** · P7 · Deps: INF-01, FND-11 · Refs: 19 §2.1
Do: services/tasks of 19 §2.1; dedicated alert workers; task roles (co-worker without WhatsApp/X secrets).
Accept: TC-SEC-04 (no credentials reachable from the co-worker process).

**T-INF-03 — CI/CD deployment workflows** · P7 · Deps: INF-02 · Refs: 19 §5
Do: `deploy-staging.yml`, `deploy-prod.yml` (manual approval), migration job with migrator role, smoke E2E, image scanning, SHA tags.
Accept: staging deploy from `main` end-to-end; rollback by previous tag tested.

**T-INF-04 — Monitoring and paging** · P7 · Deps: INF-02, FND-12 · Refs: 19 §6.1
Do: dashboards and alarms from 19 §6.1.
Accept: forced failure of each alarm condition in staging pages the on-call channel.

**T-INF-05 — Backups, DR, runbooks** · P7 · Deps: INF-01 · Refs: 19 §7, §11
Do: confirm RPO/RTO with CTO, restore drill, write the runbooks of 19 §11.
Accept: restore drill timed and recorded; runbooks reviewed.

**T-INF-06 — Secrets and access** · P7 · Deps: INF-01 · Refs: 19 §4, 13 §6
Do: Secrets Manager entries, rotation, bastion + MFA, signed-URL-only object access, WAF rules.
Accept: no secret in repo/images/logs (secret scan green).

**T-REL-01 — Must-pass replay gate** · P7 · Deps: ALR-03, ALR-06 · Refs: 18 §5
Do: nightly job running TC-MP1 and TC-MP2 (LLM mock and real-LLM routes, variants); 20 consecutive passes tracked.
Accept: counter reaches 20 before release; any regression resets it.

**T-REL-02 — Performance and chaos** · P7 · Deps: all pipeline tasks · Refs: 18 §7, TC-NFR-01/03/09
Do: 100 posts/min for 30 min plus 20 posts/s burst for 2 min; chaos tests for X, outlet, LLM, WhatsApp, Redis, Postgres.
Accept: p95 ≤ 300 s, critical path ≤ 90 s; behaviours match 14 §12.

**T-REL-03 — Security and accessibility pass** · P7 · Deps: all API tasks · Refs: TC-SEC-01…08, TC-NFR-10
Do: security suite, schemathesis fuzz, ZAP baseline, axe checks, Tamil clipping check at big-screen sizes.
Accept: all green; findings triaged.

**T-REL-04 — UAT and calibration prep** · P7 · Deps: Phase 6 complete · Refs: 18 §9
Do: gold-set session (300–500 items, 40 % Tamil/Tanglish), typical-day walkthrough with Meena-role users, presentation-safe rehearsal, WhatsApp sandbox must-pass.
Accept: outcomes recorded; exit criteria of 18 §9 met.

**T-REL-05 — Release gate sign-off** · P7 · Deps: REL-01…REL-04, INF-* · Refs: 24 §2
Do: complete 24 §2 with evidence; confirm client inputs of 24 §7.
Accept: signed checklist.

## Phase 8 — Calibration

**T-CAL-01 — Calibration support** · P8 · Deps: release · Refs: PRD "Calibration", 22 Phase 8
Do: measure daily volume, produce threshold-review report from act/dismiss data, set accuracy bars (recorded in config), arm the stage engine, publish the guardrails dashboard.
Accept: thresholds reviewed with the war room; bars recorded; stop/rollback signals visible.
