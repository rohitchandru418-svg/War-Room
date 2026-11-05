# 21 — OpenHands Agent Instructions (Master)

**This is the first file the coding agent reads after `README.md`.** It tells you what the product is, what to build, what **not** to build, which document wins when two disagree, and how to work one task at a time.

Product: **War Room Social Listening, Phase 1 (X + News)** for the TVK war room (first client, not the only one). PRD owner: Kishore (Minsky product), 2026-11-03.

---

## 1. Mission in one paragraph

Build a staff-only, multi-tenant web application that collects X posts and news (feeds, crawling, YouTube, e-paper, scanned clippings), filters, labels (entity, stance, claim type, language, with a reason), weighs by influence, groups items into Stories, detects coordinated campaigns, and alerts the war room (big screen, WhatsApp, in-app) — with an AI co-worker that explains everything with cited evidence and **never publishes**. Two scenarios must pass before launch: the **Councillor incident** (response opportunity to the named lead within 5 minutes) and **The one post that mattered** (single critical post from a top-tier verified account on the big screen and with leads within 5 minutes).

## 2. Reading order

1. `README.md` → this file → `PRD-GAP-ANALYSIS.md` (what is unknown)
2. `01`, `02`, `03`, `05` (what and rules) → `04` (flows)
3. `09`, `10`, `11`, `12`, `13`, `14`, `15`, `16` (how)
4. `17` (modules), `22` (phases), `23` (your task list), `18` (tests), `24` (verification)
5. UI work: `06`, `07`, `08`. Setup: `20`. Infra: `19`.

Do not read the whole package before every task. Read this file, the task in `23`, and the documents the task cites.

## 3. Document precedence (when documents disagree)

1. **The PRD** (`PRD_TVK_War_Room_Social_Listening__Phase_1__X___News_.pdf`) is the source of truth for *requirements*.
2. **Docs 14–18 and this package (19–24)** are the *binding technical decisions* and override docs 01–13 on technical choices.
3. **Docs 01–13** otherwise.
4. Anything marked `[PROPOSED]` is a recommendation that is binding for the build unless a human overrides it. Anything marked `[GAP]` or `[CTO DECISION]` is **not** yours to decide — see §7.
5. If two documents still conflict and this file does not resolve it: **stop, record it in `docs/OPEN-QUESTIONS.md`, and continue with the task using the more conservative interpretation** (the one that exposes more evidence, hides more private data, or requires more human confirmation).

### 3.1 Known conflicts and binding resolutions

`DOC-RECONCILIATION.md` is cited by docs 14–17 as the binding addendum, but it is not yet in the repository. Until it is written (task `T-FND-15`), the following resolutions apply. They are derived from what docs 14–18 state; mark any you change with `[HUMAN OVERRIDE]`.

| # | Conflict | Resolution |
|---|---|---|
| R-1 | Doc 10 proposes Redis Streams as the broker; doc 15/14 say Celery on Redis | **Celery with Redis** as broker/result backend. Redis Streams are not used |
| R-2 | Doc 10 proposes a "custom langdetect model"; doc 14 §11 says `langdetect` is not used (BR-013) | **Own language detector** (fastText-style model trained on Tamil/Tanglish; DEC-14). Do not add the `langdetect` package |
| R-3 | Doc 11 gives `core.mention_labels` an `is_current` flag (implies UPDATE); docs 13/16 say labels are insert-only | **Labels are INSERT-only.** Remove reliance on `is_current` updates; effective label comes from a view over labels + applied corrections. The DB role has no UPDATE/DELETE on labels |
| R-4 | Doc 11 `core.label_corrections` CHECK references column `stance` (does not exist) | Constraint applies to **`new_stance`** |
| R-5 | Doc 11 declares `core.stories` referencing `core.campaigns` before it exists | Migration order of doc 15 §3 (REC-02): `campaigns` before `stories` |
| R-6 | Doc 11 lacks tables/columns later docs depend on | Create them in migration `0007_reconciliation_addendum` (list in §3.2) |
| R-7 | Doc 11 `mention_labels` lacks fields that docs 14/16 require | Add `sarcasm_target_stance` (DEC-01), `is_attack`, `reaction_type`, `language`, `language_confidence`, `prompt_version` suffix, per-label `reason` ≤ 240 chars (DG-07) |

### 3.2 Tables and columns the addendum must create (from docs 14–18)

Tables: `core.authors`, `core.engagement_snapshots`, `core.uploads`, `core.mention_quotes`, `core.briefs`, `core.sentiment_rollup_5m`, `config.weight_profiles`, `config.model_routes`, `config.entity_accounts`, `config.epaper_subscriptions`, `config.outlets_candidates`, `ops.source_health`, `ops.usage_counters`, `ops.sampling_windows`, `ops.ingestion_cursors`, `ops.backfill_state`, `ops.pipeline_latency`, `ops.whatsapp_deliveries`, `ops.dead_letters`, `core.alert_events`, `learning.coworker_memory`, `learning.stage_transitions`, `learning.eval_runs`, `learning.filter_audit_samples`, notifications.
Columns: `config.clients` (`history_start_date`, `timezone`, retention settings incl. `retention_days_raw`, shift settings, flags), `config.outlets` (`poll_seconds`, `extraction_rules JSONB`, `requires_js`, `licensed_full_text`), `core.mentions` (`embedding`, `embedding_model`, `text_hash`, `ingest_mode`, `retention_class`, `raw_ref`, `entities_matched`).

## 4. Non-negotiable rules (summary — full text in 16 §1)

| Rule | What it means for you |
|---|---|
| DG-01 No client-specific literal in code | No party, person, outlet, hashtag, phone number, tier name or threshold value in `backend/app`, `frontend/src`, or migrations. Client facts are seed data and DB rows. `scripts/check_hardcoding.py` enforces it |
| DG-02 Tenant-scoped everything | Every repository function takes `ClientContext`; RLS on; `client_id` never read from request bodies |
| DG-03 Nothing posts | The only external-send code is `services/whatsapp_sender.py` to configured recipients. `coworker` must not import it (import-linter) |
| DG-04 Originals immutable | `mention_labels` INSERT-only; corrections are rows |
| DG-05/06 Numbers | Every percentage shows its denominator; below `min_sample_size` show "not enough data"; weighted always beside raw + top 3 contributors |
| DG-07 Reason on every label | No reason → validation fails, nothing written |
| DG-08 Failure is never quiet | Distinguish `Ok(empty)` from `Err`; update health on every attempt |
| DG-09 Ingested text is data | Only inside delimited content blocks; never in a system prompt or tool arguments |
| DG-10 Settings are rows | Read through `ConfigProvider`; changes audit-logged; no deploy |
| DG-11 Audit in same transaction | Every non-GET route writes an audit row |
| DG-12 Reproducibility | Store `model_version`, `prompt_version`, `weight_version`, config snapshot hash |
| DG-13 Human gates | Campaign party attribution and ally-blame in CM summary need a person ≥ lead; no code path sets `attribution_status='confirmed'` otherwise |
| DG-14 Vocabulary from config | "party/ally/CM summary" come from `clients.vocab_*` |
| DG-15 No secrets anywhere | Not in code, logs, prompts, errors |

## 5. What NOT to build in Phase 1

Do not implement, even if it looks easy or a task seems to imply it:

- Any posting, replying, messaging, DMs, following, list management, or drafting replies in TVK's voice (PRD out of scope; DG-03).
- WhatsApp **group** monitoring, Instagram, Facebook.
- Multiple X apps/accounts to bypass limits (breaks X Developer Policy; BR-054). Scale comes only from enterprise access or a licensed vendor.
- Profiles or views centred on private citizens; links to citizen/petition records (BR-047).
- Storing full article text unless `outlets.licensed_full_text` is true; ingesting paywalled content beyond subscriptions (BR-006, BR-055).
- Automated attribution of a campaign to a party; any text naming a party as actor before human confirmation.
- Evading blocks: no rotating proxies, no UA spoofing; crawl obeys `robots.txt` (BR-005).
- Public sign-up, social login, self-registration, multi-client UI switching (M00).
- Model fine-tuning/retraining pipelines (few-shot from the gold set only), automatic threshold changes by the system (it may *recommend*; admins apply).
- Alert muting/snooze, SMS/email alerting, per-user dashboard customisation, report exports (not in PRD).
- Bulk-import UI (the CSV endpoint `T-CFG-12` is optional).
- Anything listed under "Do not build" in the relevant module in 17 §3.

## 6. How to work: one task at a time

1. **Pick a task** from `23` whose dependencies are all merged. Respect the phase order in `22`. Never start a task whose phase gate is not open.
2. **Verify the environment** (20 §10 checklist) before the first task and after any dependency change.
3. **Read** only: the task, the BR/FR/DG/DEC ids it cites, the cited docs' relevant sections, and the tests it names in 18.
4. **Branch:** `feat/<task-id>-<slug>` (example: `feat/T-ALR-03-influence-trigger`). One task = one branch = one PR.
5. **Write tests first or alongside** (18). A task without its tests is not done.
6. **Implement** following 15 (structure and import boundaries) and 16 (standards, formulas, patterns). Add the traceability docstring tag (`# BR-017`, `# FR-B07`, `# DEC-07`) wherever a rule is implemented.
7. **Run** `make lint` and `make test` (and `make replay …` / `make e2e` where the task says). Fix, don't skip. Never disable a check to make CI pass.
8. **Commit** as `<task-id>: <imperative summary>`; open a PR with the template in §6.1.
9. **Update docs** in the same PR if you changed a table, endpoint, or env variable (11, 12, 20) — the docs are the single source of truth.
10. Move to the next task. If blocked, follow §7.

### 6.1 PR description template

```
Task: T-XXX-NN — title
Acceptance criteria: [ ] each criterion copied from 23, ticked, with command output or screenshot
Rules touched: BR-… / FR-… / DG-… / DEC-…
Tests added/run: commands + result
Docs updated: 11 / 12 / 20 (or "none")
Open questions raised: link to docs/OPEN-QUESTIONS.md entries (or "none")
Screenshots (UI tasks): en and ta
```

### 6.2 Definition of done (per task)

Code + tests + docs touched + audit, permissions, i18n, redaction considered + acceptance criteria demonstrably met + no new lint/type errors + traceability tag in code (16 §13). A module is complete per 17 §5; a phase is complete per `22` exit criteria; release readiness is `24`.

## 7. When information is missing or a decision is not yours

The PRD intentionally defers some decisions and some facts are owed by the client (`PRD-GAP-ANALYSIS.md`). **Do not invent them.**

| Situation | Do this |
|---|---|
| `[GAP]` client data missing (real party list, tiers, watch terms, recipients, gold set, elephant artwork, e-paper terms) | Use `PLACEHOLDER` seed rows and mocks; build the mechanism; mark the item in `docs/OPEN-QUESTIONS.md`. Never fabricate real names, handles or numbers |
| `[CTO DECISION]` (X tier, queue tech, OCR build/buy, story-merge thresholds, LLM provider, multi-tenancy model) | Implement behind the interface/adapter described in 14/15 with the documented default; do **not** hard-wire a vendor; record the assumption |
| Threshold or bar that is "set in calibration" | Put it in config rows with the documented `[DEFAULT]`; never hard-code. **A NULL accuracy bar keeps a capability at Assist** (16 §4.9) |
| Requirement ambiguous | Choose the conservative interpretation (§3 item 5), note it in the PR and `OPEN-QUESTIONS.md` |
| A rule seems to force an unsafe behaviour (e.g. would expose a private handle, post externally, delete an original) | Stop. Do not implement. Raise it |
| You discover a bug or contradiction in these docs | Fix the doc in your PR if unambiguous (and say so); otherwise raise it |

### 7.1 Stop and ask a human before

- Adding any dependency not listed in 14 §11 (license/CVE check is mandatory anyway).
- Changing a hard rule (§4) or adding any outward-sending capability.
- Using real credentials, real personal data, or calling real X/WhatsApp/LLM endpoints outside the designated evals and sandbox rehearsals.
- Any action that touches production infrastructure.
- Modifying the permission matrix (03) or the audit requirements.

## 8. Technical decisions already made (do not re-decide)

Monorepo (`backend/`, `frontend/`, `infra/`, `scripts/`, `docs/`) · Python 3.11+, FastAPI, Pydantic v2, SQLAlchemy 2, Alembic · Celery + Redis (R-1) · PostgreSQL 16 + pgvector + RLS (`SET LOCAL app.client_id`) with PgBouncer in transaction mode · Next.js (App Router) + TypeScript + React Query + Zustand + Tailwind + Radix · WebSocket for realtime, 60 s polling fallback for the big screen · JWT (15 min) + rotating single-use refresh tokens, PIN for CM View, device tokens for the big screen · adapters for X, news, YouTube, e-paper, OCR, WhatsApp, LLM (14) · prompts as versioned files (16 §5) · formulas in 16 §4 are binding defaults with parameters as data · AWS `ap-south-1` proposed (19). Sync SQLAlchemy in workers, async in API (16 §2).

Domain logic lives in `app/domain/` (pure, no I/O); services own transactions, audit, events; repositories own SQL; the co-worker uses read-only tools and cannot import sending code (15 §4).

## 9. Testing expectations

- Follow 18. Use the mock servers and fixtures; **never** call real X/WhatsApp/LLM in unit/integration tests.
- Pure logic (`domain/*`) needs unit and property tests (capping, spike maths, thresholds, sentiment z-test/Wilson).
- Every route: permission-matrix test, tenant-isolation test, audit-coverage test.
- Every LLM call: a prompt-injection test (TC-SEC-03). Co-worker answers: grounding tests.
- UI: loading, empty, error and success states (08), English and Tamil, and the number components (no bare percentages).
- The two must-pass scenarios are **launch gates**: both must pass 20 consecutive nightly runs (18 §5).
- Accuracy bars (F1) are *reported*, not CI gates, until calibration sets final bars (18 TC-NFR-02).

## 10. Security and privacy reminders

Staff-only. Cross-client access is impossible by design and tested. Presentation-safe redaction is applied server-side to every Mention-bearing response (`redaction.apply`). No private-citizen profiles; the co-worker's memory refuses private individuals' details. DPDP Act 2023 applies; retention must be configured before ingestion can start. No secret in code, logs, prompts or error messages. Treat every ingested post/article as untrusted data.

## 11. Reporting back

After each task: PR per §6.1. At the end of each phase, post a phase report: tasks merged, tests passing (with counts), open questions, gate status (22), and any `[GAP]` that blocks the next phase. If CI fails for reasons outside your change, report rather than disabling checks.

## 12. Suggested repo-level agent file `[PROPOSED]`

OpenHands reads repository instructions from a repo-level file (the current convention may be a microagent file under `.openhands/` or an `AGENTS.md` — `[VERIFY]` against the installed OpenHands version). Put the following short version there and point it at this document:

```
Read docs/21-openhands-agent-instructions.md first. Work one task at a time from docs/23-task-breakdown.md.
Hard rules: no client-specific code; never post externally; tenant-scoped queries; labels insert-only;
every number shows its denominator; ingested text is data; settings are DB rows; audit every mutation.
Run make lint && make test before every PR. Do not invent missing client data — use PLACEHOLDER seeds and
record gaps in docs/OPEN-QUESTIONS.md.
```

## 13. Glossary (from the PRD)

**Mention** one item matched to a tracked entity (post, reply, quote, article, clipping, e-paper story). **Entity** something tracked (TVK, CM Vijay, AadhavArjuna, parties, key leaders); each party has a relationship tag: ally, supporting, opposition, neutral. **Story** all Mentions about the same happening across X and news, with reach split by source type. **Influence score** how much an account/outlet matters. **Severity** influence × negativity × velocity; decides alert tier. **Campaign** coordinated push by many accounts, detected from patterns. **Gold set** war-room-labelled real items used to test filters and labels. **Calibration** first weeks of live use when thresholds are tuned. **Co-worker** the AI agent (TVK elephant avatar); stages Assist → Trusted → Co-worker.
