# 18 — Testing and QA Plan

**Goal:** every PRD acceptance criterion has an automated test that fails when the behaviour is wrong, and the two must-pass scenarios are replayable on demand. Test IDs here are the ones used in `REQUIREMENTS-TRACEABILITY.md`.

**Levels:** `U` unit (pure, fast) · `I` integration (API+DB+Redis in containers) · `W` worker/pipeline integration · `E` end-to-end (Playwright against the full stack with mocks) · `P` performance · `S` security · `V` eval (LLM-in-loop) · `M` manual/UAT with the war room.

---

## 1. Test pyramid and tooling

| Level | Tooling | Runs | Target size |
|-------|---------|------|-------------|
| U | pytest, hypothesis (property tests for capping, thresholds, spike maths), jest | every commit, < 2 min | ~60 % of tests |
| I/W | pytest + testcontainers (Postgres 16 + pgvector, Redis), respx for HTTP mocks, Celery in eager-with-real-broker mode | every PR, < 12 min | ~30 % |
| E | Playwright against docker-compose stack with mock X, mock WhatsApp, mock LLM, mock news server | every PR (smoke subset), nightly (full) | ~8 % |
| P | locust + pipeline replay timing | nightly + pre-release | — |
| S | pytest security suite, ZAP baseline, dependency scan, secret scan | every PR (suite), weekly (ZAP) | — |
| V | `evals/` runner with real LLM routes | on prompt/route change + nightly | — |
| M | UAT scripts (§9) | before launch and after calibration | — |

**Determinism:** all time through `Clock`; tests use a `FrozenClock`/`FakeClock`. LLM mock returns canned JSON keyed by content hash. Real-LLM tests are only in `V`.

---

## 2. Test data and mocks

| Asset | Location | Notes |
|-------|----------|-------|
| Synthetic client "Acme Beverages" | `backend/seeds/demo/` | Proves generalisation: different vocabulary (`competitor`, `partner`, `leadership summary`), no political terms. Used in tenant-isolation and vocabulary tests |
| TVK seed (structure only until real lists arrive) | `backend/seeds/tvk/` | Placeholder entities/terms clearly marked `PLACEHOLDER`; real lists replace them (G-07..G-10) |
| X mock server | `backend/tests/mocks/x_server.py` | Serves recorded/generated payloads, supports rate limit/429/401/5xx injection, tracks "reads" for budget tests |
| WhatsApp mock | `backend/tests/mocks/wa_server.py` | Accepts templates, emits status webhooks with configurable delays/failures |
| LLM mock | `backend/tests/mocks/llm.py` | Deterministic; supports latency/failure injection |
| News mock site | `backend/tests/mocks/news_site/` | Feeds, sitemaps, HTML pages, robots.txt, slow/blocked/JS-only variants |
| Tamil corpus | `backend/tests/data/tamil/` | ≥ 300 hand-written items: Tamil script, English, Tanglish, mixed, sarcasm, numerals (௧௨௩, "௫ கோடி"), with expected labels. Seed for the **gold set** until real data arrives |
| Scenario fixtures | `backend/tests/scenarios/councillor_incident.yaml`, `one_post_mattered.yaml`, `surrogate_wave.yaml`, `ally_discontent.yaml`, `print_only_story.yaml` | One per PRD problem example; each is a timeline of events with timestamps (§5) |
| Scan images | `backend/tests/data/scans/` | Real-looking Tamil newsprint samples (public-domain/synthetic), good/poor quality |
| Injection corpus | `backend/tests/data/injection/` | Posts/articles containing instruction-like text in Tamil/English |

---

## 3. Acceptance tests per user story (TC-US01 … TC-US35)

Each test encodes the PRD Given/When/Then. Level in brackets.

| TC | Story | Scenario encoded |
|----|-------|------------------|
| TC-US01 | 1 | [W+E] Insert a mention at t0; within 5 min (fake clock) `sentiment.update` emitted; big screen shows "of N mentions, last updated HH:MM". With N=29 shows "not enough data"; N=30 shows percentage |
| TC-US02 | 2 | [I+E] CM summary has ≤ 3 changes; each has one-line cause + evidence link; Tamil and English variants; change blaming an ally is withheld (409 `REVIEW_REQUIRED`) until reviewed, then shows "reviewed by <name>" |
| TC-US03 | 3 | [E] Click each number/Story: opens Mentions with author/outlet, engagement, time, label, reason, original link |
| TC-US04 | 4 | [I+E+S] Presentation mode on: big screen payload has only aggregates, public-actor names, Story titles; private handles/internal notes/correction details absent (property test over all Mention-bearing responses); toggle reaches all displays ≤ 2 s |
| TC-US05 | 5 | [W] Replay `one_post_mattered` (also TC-MP2); asserts instant path, reason string, within 5 min |
| TC-US06 | 6 | [I] Add handle to tier → next Mention weighted by tier; audit row with name+time; `tier_audit` visible |
| TC-US07 | 7 | [U+I] Weighted beside raw; top-3 contributors; no item share > 5 % (property test); `InsufficientSample` below 20 |
| TC-US08 | 8 | [I+E] 6 active alerts → big screen shows ≤ 3, related posts grouped by `dedupe_key`; ack clears flash for all clients (2 WS clients), "taken by <name>" |
| TC-US09 | 9 | [U+W] Spike maths: table-driven cases (below each of the 4 conditions → no alert; all satisfied for ≥ 15 min → alert with baseline/current/top posts); event window uses event baseline; cold start rule |
| TC-US10 | 10 | [W] Replay `surrogate_wave`: 300 accounts, 70 % < 90 days, near-identical Tanglish → alert with accounts, ages, phrasing, timing series, confidence |
| TC-US11 | 11 | [I+S] Campaign reads "possible coordination" and names no party; confirm requires lead; team_member → 403; confirmation logs name/reason; DB check blocks bypass |
| TC-US12 | 12 | [I+E] Confirmed campaign page shows mentions/hour, new accounts/hour, response Stories on one timeline |
| TC-US13 | 13 | [W] Replay `ally_discontent`: ally MLAs negative toward client → Ally Mood shows people+posts; alert at `ally_threshold_*` |
| TC-US14 | 14 | [I] Change ally→opposition: new tag effective immediately for new alerts/panels; old Mentions retain old tag (`relationship_at`) |
| TC-US15 | 15 | [W] 9 outlets + 1,200 X posts about one statement → one Story "9 outlets, 1,200 posts"; both lists; merge/split work and are audited |
| TC-US16 | 16 | [W+E] Upload scans → Mention with outlet/date/page; low-confidence marked "check scan"; duplicate upload rejected |
| TC-US17 | 17 | [W] Edition ingested → Mentions tagged `epaper`, page image URL signed; automation disabled ⇒ provider refuses to run (no terms breach) |
| TC-US18 | 18 | [I] News Mention exposes outlet tier + reason; partisan outlet reason text; weight lower than independent equivalent |
| TC-US19 | 19 | [W] Single `anonymous_insinuation` item → Story of one, "unattributed", not suppressed |
| TC-US20 | 20 | [V+I] Tamil question → Tamil answer; every claim cited; counts match tool queries; N<30 ⇒ "not enough data"; first token ≤ 10 s |
| TC-US21 | 21 | [I+V] Every fired alert gets a two-line explanation + evidence links; unavailable LLM ⇒ `explanation_status='unavailable'` and UI text per 08 §3.2 |
| TC-US22 | 22 | [I+E] Shift end → handover draft (open alerts, owners, watch); editable; send notifies incoming shift; CM summary on demand/schedule |
| TC-US23 | 23 | [I+S] Recommendations shown as suggestions with reason; import-linter + runtime test prove no posting/messaging code reachable from `coworker` |
| TC-US24 | 24 | [I+E] Correct a label: shows "corrected by <name>", original in history, aggregates update, corrected counts identifiable; label rows never UPDATEd (DB test) |
| TC-US25 | 25 | [W+I] Applied corrections enter gold set and few-shot pool; weekly accuracy chart data model-only vs corrected |
| TC-US26 | 26 | [U+I] Softening correction on client-primary entity or tier-1 story ⇒ pending second review, aggregates unchanged until approved; reviewer ≠ corrector; model-only view still available |
| TC-US27 | 27 | [I] Weekly sample stratified by source/language/entity; reviewer verdicts → missed-item rate on admin page |
| TC-US28 | 28 | [W+E] Break X / one outlet / e-paper / LLM: affected panels show "data delayed since HH:MM" + named source; no number silently stops; empty-but-healthy run does **not** show delayed |
| TC-US29 | 29 | [W] WhatsApp failure → retries (backoff), in-app "WhatsApp not delivered", lead notified |
| TC-US30 | 30 | [I] Reads at 50/80/95 % → admin notifications once each; sampling labelled on every affected number |
| TC-US31 | 31 | [W] Replay `councillor_incident` (also TC-MP1); response opportunity to the configured lead ≤ 5 min with evidence, who is reacting, "no TVK response yet"; later TVK post closes it "responded by <name> at HH:MM" |
| TC-US32 | 32 | [I+E] Queue shows suggested kind, who reacted, time since start, TVK status read from client-side accounts; acted/not-needed recorded |
| TC-US33 | 33 | [W] News report/YouTube item quoting a remark → Mention linked to source with quoted line and speaker where known |
| TC-US34 | 34 | [U+I] Capability above bar for `stage_promote_weeks` ⇒ promoted; admin page shows stage + error trend; NULL bar ⇒ stays Assist |
| TC-US35 | 35 | [U+I] Accuracy below bar ⇒ demoted one stage, human review resumes, lead notification with reason |

---

## 4. Other requirement test groups

### 4.1 Foundation and security (TC-FND, TC-SEC)
| TC | Purpose |
|----|---------|
| TC-FND-01 | **Cross-client attack suite:** for every GET/POST/PATCH/DELETE route, authenticate as client B, try client A resource IDs → 404/403, never data; raw SQL `SELECT` with RLS set to B returns zero A rows; app role has no BYPASSRLS |
| TC-FND-02 | Permission matrix generated from 03 + reconciliation: for every (route, role) the expected 200/403 |
| TC-FND-03 | Audit coverage: every non-GET route writes an audit row in the same transaction (rolled back together) |
| TC-FND-04 | Config without deploy: change a threshold via API → next evaluation uses it within 30 s; audit row exists |
| TC-FND-05 | Vocabulary: demo client renders "competitor/partner/leadership summary" in API labels and UI; no TVK literal in the demo run |
| TC-FND-06 | `check_hardcoding.py` fails when a banned literal is introduced (meta-test) |
| TC-FND-07 | Migrations apply on empty DB, downgrade/upgrade cycle, FK order (REC-02), schema equals models |
| TC-SEC-01 | Auth: password hashing bcrypt cost ≥ 12, lockout, refresh rotation (reuse of old token revokes family), logout invalidates |
| TC-SEC-02 | JWT claims, expiry, tampering; device token scope limited to `/bigscreen/*`, bad CIDR rejected; CM PIN session timeout 30 min |
| TC-SEC-03 | Prompt injection corpus through labeller, explainer, co-worker, briefs: no instruction followed; output schema intact |
| TC-SEC-04 | No secrets in logs/errors (log scraper); no credentials reachable from co-worker process (env/secret mount check) |
| TC-SEC-05 | `mention_labels` UPDATE/DELETE denied for app role; `audit.audit_log` append-only |
| TC-SEC-06 | Signed URL expiry; S3 not public; uploads: type/size/virus/path traversal |
| TC-SEC-07 | Rate limiting and input validation fuzz (schemathesis against OpenAPI) |
| TC-SEC-08 | Webhook signature verification; replay/duplicate tolerance |

### 4.2 Pipeline and data (TC-PIPE)
| TC | Purpose |
|----|---------|
| TC-PIPE-01 | Dedupe: same X post/URL/clipping hash twice ⇒ one Mention, no second LLM/X cost recorded |
| TC-PIPE-02 | Wide-net filter: watch-term hit kept; Tamil/Tanglish variant hit kept; topic-similarity keep (criticism without naming); unmatched stored raw with `filter_matched=false` and eligible for audit |
| TC-PIPE-03 | Language detection on the Tamil corpus: per-language accuracy report; X tag ignored (fixture where X says `en` for Tanglish) |
| TC-PIPE-04 | Tamil numerals/amounts parse cases (BR-014) |
| TC-PIPE-05 | Label schema validity, reason required (DG-07), unknown entity dropped, sarcasm requires target stance (DB CHECK) |
| TC-PIPE-06 | Escalation: confidence below thresholds routes cheap→strong→human queue |
| TC-PIPE-07 | Idempotent stages: replay same message twice ⇒ no duplicates; crash mid-stage resumes |
| TC-PIPE-08 | Cost rule: strong route invoked only for the listed cases (count calls in a 1,000-item replay) |
| TC-PIPE-09 | Backfill items never raise live alerts; resumable; budget share respected |
| TC-PIPE-10 | Crawler: robots.txt disallow honoured; pacing; UA string; blocked site ⇒ `down` not "no articles"; selector break detected |
| TC-PIPE-11 | Snippet cap and no full text stored (BR-006) |
| TC-PIPE-12 | X budget: sampling mode throttles P2 first; `is_sampled` flag propagates to dashboard payload |
| TC-PIPE-13 | OCR flow: confidence flag, edit text, confirm, dedupe |
| TC-PIPE-14 | Gold-set metrics script computes F1 per language and matches a hand-computed fixture |

### 4.3 Non-functional (TC-NFR)
| TC | Requirement | Method / pass criterion |
|----|-------------|-------------------------|
| TC-NFR-01 | Freshness p95 ≤ 5 min | Replay of 100 posts/min for 30 min plus spike burst of 20 posts/s for 2 min; `ops.pipeline_latency` p95 ≤ 300 s; critical-path (influence) p95 ≤ 90 s |
| TC-NFR-02 | Accuracy per language | Gold-set F1 ≥ bars (defaults 0.80/0.75/0.72) — **reported**, not a CI gate until calibration sets final bars |
| TC-NFR-03 | Reliability/fallback | Chaos tests: kill X mock, one outlet, LLM, WhatsApp, Redis restart, Postgres failover: matrix in 14 §12 behaviours observed |
| TC-NFR-04 | Privacy | No private-citizen profile views (no endpoint filters by private author); retention jobs delete per policy; ingestion refuses to start when retention unset |
| TC-NFR-05 | Auditability | Reproduce any label/score from stored versions in a script (`scripts/reproduce_score.py`) and compare |
| TC-NFR-06 | Statistics | Property tests: denominators present in every percentage payload; min-sample gate; z-test and Wilson implementation vs scipy reference |
| TC-NFR-07 | Generalisation | Onboard "Acme" by seed only, run full pipeline replay, zero code diffs, isolation holds |
| TC-NFR-08 | Co-worker latency | 95 % of typical questions first token ≤ 10 s with real LLM in `V` |
| TC-NFR-09 | Big screen resilience | Kill WS: screen shows stamp/"reconnecting", recovers, watchdog works; 24 h soak: no memory growth > 20 % |
| TC-NFR-10 | Accessibility/i18n | axe checks; en/ta key parity; Tamil snapshot tests; no clipped Tamil glyphs at big-screen sizes |

### 4.4 Must-pass scenarios (TC-MP1, TC-MP2) — see §5.

---

## 5. Must-pass replay design (pre-launch gate, PRD p.25)

`scripts/replay_scenario.py --scenario <file> [--real-llm] [--real-wa-sandbox]` loads a scenario timeline, drives the **real pipeline** (collect adapters fed by mocks, real Celery workers, real DB) under a `FakeClock` or real time, and asserts timestamps.

**TC-MP1 — Councillor incident** (`councillor_incident.yaml`)
Timeline: t0 a news article reports a rival-party speaker's remark about CM at a DMK-type function (client seed uses placeholder party names); t0+4 min five outlets carry it; t0+6 min posts from three non-client parties and a verified journalist condemn; **no client-side account posts**. Assertions:
1. Response-trigger conditions (DEC-08) first true at `t_trigger` (computed by the test from the timeline).
2. A `response_opportunity` alert exists with `trigger_at = t_trigger`.
3. WhatsApp mock receives template `wr_response_opportunity` for the configured recipient with `accepted_at − t_trigger ≤ 300 s`.
4. Payload includes the triggering article/clip title, ≥ 3 "who is reacting" names, "no TVK response yet".
5. Inject a client-side account post at `t_trigger+20 min` → opportunity becomes `responded`, text "responded by <name> at HH:MM", no further WhatsApp.
6. Run with the LLM mock **and** with the real LLM route (nightly): both pass.
7. Variant A: LLM down ⇒ opportunity still fires using rules-only reaction classification (flagged "unlabelled by AI"). Variant B: WhatsApp fails ⇒ retries then in-app banner.

**TC-MP2 — The one post that mattered** (`one_post_mattered.yaml`)
Timeline: verified account in tier rank 1 with 1.2 M followers posts a sharply critical post at t0; baseline traffic otherwise quiet (no spike); engagement snapshots show 4,000 reposts by t0+20 min (the early snapshots show a few hundred). Assertions:
1. Mention labelled negative about a client-primary entity.
2. Tier-1 alert created with `trigger_at = published_at`; `big screen` WS `alert.new` emitted and WhatsApp `accepted_at − published_at ≤ 300 s` (instant path DEC-20).
3. `reason_text` contains verified, follower count, tier name, and the engagement known at alert time.
4. No volume-spike alert exists for this window (proves it was the influence path).
5. Variant: LLM cheap route down ⇒ rules-only path fires with "unlabelled by AI" marker.
6. Variant: same post from a 40-follower account ⇒ **no** Tier 1.

**Pass rule:** 20 consecutive nightly passes before launch; any regression blocks release (24 §release gate).

Other scenario replays (not launch-gating but run nightly): `surrogate_wave` (TC-US10), `ally_discontent` (TC-US13), `print_only_story` (TC-US15/16/19 combined: clipping joins an X story two days later).

---

## 6. Co-worker evals (V level) — PRD p.18

- **Dataset:** `backend/evals/datasets/qa.jsonl`: ≥ 60 items at launch (≥ 20 Tamil, ≥ 10 Tanglish, ≥ 5 "answer should be I don't know"), each with `question`, `expected_facts`, `expected_mention_ids` (against a frozen DB snapshot `evals/snapshots/*.sql`), `must_say_unknown`.
- **Scores:** answer accuracy (LLM-judge with rubric **and** programmatic check of numbers), citation correctness (cited IDs ⊆ ground truth and every claim cited), unknown-handling, language match, latency.
- **Gates:** run on every change to `prompts/**` or `model_routes` seed; fail CI if citation correctness < 0.95 or any ungrounded number appears; accuracy must not drop > 2 pts vs main. Absolute accuracy bar set in calibration (PRD "Calibration").
- **Label evals:** same runner over the Tamil corpus + gold set → F1 per language/entity, confusion matrices, sarcasm subset.
- **Injection evals:** TC-SEC-03 corpus included.
- Results stored in `learning.eval_runs` (migration with M12) and visible to admins.

---

## 7. Performance and capacity
- **Pipeline throughput target (design):** 100 k X posts/day (cap-sized) + 5 k news items/day + bursts of 10× for 30 min. Workers scaled so p95 queue wait on `alerts`/`label_cheap` < 20 s during bursts.
- **DB:** dashboard summary p95 < 400 ms at 10 M mentions (rollup tables), mention list p95 < 600 ms with filters; EXPLAIN checks in CI for the top 10 queries (no seq scans on `core.mentions`).
- **WS:** 200 concurrent clients, fan-out < 1 s.
- **LLM cost telemetry** compared against a budget forecast; not a failing gate (PRD: no cap) but trend alert on > 30 % week-over-week cost per item.

---

## 8. Frontend test coverage
Component tests for every number component (denominator, not-enough-data, sampled, stale), correction panel (reason min 10 chars, second-review notice text from 08 §3.7), alert card states (flash, taken-by), source-health banner, presentation badge. Playwright flows: typical day (04 §1) end to end with mocks at fake times; Tamil UI run of the same flows.

---

## 9. UAT and calibration (with the war room)

| Activity | When | Exit |
|----------|------|------|
| Gold-set building session (war room labels 300–500 items: 40 % Tamil/Tanglish) | before calibration | Gold set loaded; F1 baselines recorded |
| Typical-day walkthrough with Meena-role users | end of phase 6 | All steps of 04 §1 done without Minsky help |
| Presentation-safe rehearsal with a visitor-style audience | before CM visit | Nothing private appears; toggle ≤ 2 s |
| Must-pass replays with real recipients' WhatsApp (sandbox) | before launch | TC-MP1/2 pass 20× |
| Two-week calibration (PRD) | after launch | Thresholds reviewed with act/dismiss data; bars set; stage engine armed |

**Stop/rollback signals** (PRD p.23) are encoded as dashboards on the admin Accuracy page and reviewed weekly: share of big-screen alerts dismissed, Tamil/Tanglish F1 below bar, any privacy incident.

---

## 10. CI gates summary
Lint/type; unit; integration; import-linter; hard-coding check; OpenAPI drift; i18n parity; migration cycle; permission-matrix test; cross-client attack suite; security suite; coverage thresholds (16 §12); smoke E2E; evals on prompt change. Nightly: full E2E, replays, perf, ZAP (weekly).
