# 16 — Development Guidelines

**Audience:** OpenHands and any human engineer. These rules turn the PRD's non-negotiables into code habits. Numbered rules (`DG-xx`) are cited in task acceptance criteria (23) and the checklist (24).
**Formulas in §4 are binding defaults** so the coding agent does not invent them. Their numeric parameters are **data** (weight profile or thresholds), tunable in calibration without a deploy.

---

## 1. Non-negotiable rules

| ID | Rule | Origin |
|----|------|--------|
| DG-01 | **Nothing client-specific in code.** No literal party, person, outlet, hashtag, phone number, tier name or threshold value appears in `backend/app`, `frontend/src`, or migrations. Client facts live in `seeds/<client>/` (data) and DB rows. CI runs `scripts/check_hardcoding.py` with a banned-literal list. | BR-050, NFR-07 |
| DG-02 | **Every query is tenant-scoped.** All repository functions take a `ClientContext`; the DB session sets `app.client_id` for RLS; `client_id` is never read from request bodies/params. | BR-051, 13 §3.1 |
| DG-03 | **Nothing posts.** No code path exists that publishes/replies/messages externally except `services/whatsapp_sender.py` to configured recipients. The co-worker package cannot import it (import-linter). | BR-043 |
| DG-04 | **Originals are immutable.** `mention_labels` is insert-only; corrections are rows; effective values come from views. No `UPDATE` on labels anywhere. | BR-033 |
| DG-05 | **Every number carries its denominator;** below `min_sample_size` the UI shows "not enough data". Use the shared UI components (§8), not ad-hoc formatting. | BR-037/038 |
| DG-06 | **Weighted always beside raw.** API payloads that include a weighted figure also include the raw count and the top-3 contributors. | BR-019 |
| DG-07 | **Every label has a reason** (non-empty, ≤ 240 chars). A label without a reason fails validation and is not written. | BR-011 |
| DG-08 | **A failing source is never "quiet".** A function that returns "no items" must distinguish `Ok(empty)` from `Err(...)`. Health is updated on every attempt. | FR-I28 |
| DG-09 | **Ingested text is data.** It is only ever passed to an LLM in a delimited content block; never formatted into a system prompt or tool arguments. | BR-042 |
| DG-10 | **Settings are rows.** Thresholds, weights, tiers, terms, recipients, vocabulary are read through `ConfigProvider` (30 s cache, invalidated by Redis pub/sub on change). Changing one never needs a deploy and always writes an audit row. | NFR-05/06 |
| DG-11 | **Every mutation writes audit in the same DB transaction.** `audit.record(ctx, action, entity_type, entity_id, old, new, reason)`. A test enumerates all non-GET routes and fails if one lacks an audit write. | NFR-05 |
| DG-12 | **Reproducibility.** Every model/derived output stores `model_version`, `prompt_version` (labels, explanations, briefs, answers) or `weight_version` / threshold snapshot (scores, alerts). | NFR-05 |
| DG-13 | **Naming a party behind a campaign and ally blame need a human.** Enforced in service layer + DB check (B5) + API role guard; there is no code path that sets `attribution_status='confirmed'` without a `confirmed_by` user of role ≥ lead. | BR-030/044/045 |
| DG-14 | **Config-driven vocabulary.** UI words "party/ally/CM summary" come from `clients.vocab_*` through `t('vocab.…')`. | BR-052 |
| DG-15 | **No secret in code, logs, prompts or error messages.** | 13 §3.5, §5.3 |

---

## 2. Coding standards

**Python (backend)**
- 3.11+, full type hints; `mypy --strict` on `app/domain` and `app/coworker`, standard mypy elsewhere. `ruff` (lint + format). Line length 100.
- `async` end-to-end in API; Celery tasks are sync wrappers calling `asyncio.run` on an async service function **or** use sync SQLAlchemy sessions in workers — pick one per worker image and keep it consistent: **default = sync SQLAlchemy in workers, async in API** (simpler Celery semantics).
- Functions ≤ 60 lines; modules ≤ 500 lines; no wildcard imports; no mutable default args.
- Pure business logic goes in `app/domain` (no I/O, deterministic, takes a `Clock` and parameters). Services orchestrate I/O around it.
- Pydantic v2 models for all API I/O and all LLM structured outputs. No dict-shaped data crossing module boundaries.
- Exceptions: raise typed `WarRoomError(code, http_status, message, details)`; global handler emits the envelope of 12. Never `except Exception: pass`; if you catch broad, log with `exc_info` and re-raise or convert.
- Time: always timezone-aware UTC in storage and logic; convert to `clients.timezone` only at the edge (UI, briefs, WhatsApp text). Never call `datetime.now()` outside `common/clock.py`.
- Money-like or percentage maths uses `Decimal` or numpy floats with explicit rounding at the edge only.
- Docstring tag on code implementing a rule: `# BR-017` / `# FR-B07` / `# DEC-07` so traceability can be grepped (24 §traceability checks).
- Any `TODO` must carry a task ID (`TODO(T-XXX-NN)`); CI rejects bare TODOs.

**TypeScript (frontend)**
- `strict: true`; no `any` (eslint). Server data via React Query only; UI state via Zustand; WebSocket events update the React Query cache through one adapter.
- Components are presentational + hooks; no data fetching in leaf components that are reused (pass props).
- Accessibility: Radix primitives, keyboard operable, visible focus, colour is never the only signal (alert tiers use icon + text + colour).
- Tamil rendering: Noto Sans Tamil loaded via `next/font`, `lang="ta"` on Tamil strings, no letter-spacing on Tamil text, tested with a Tamil visual snapshot.

**SQL**
- Only in `repositories/` and `migrations/`. Use SQLAlchemy Core/ORM with bound parameters; never string-format values.
- Every new table: `client_id`, RLS policy, `created_at`, indexes per query pattern, migration + model + test.

---

## 3. Patterns

### 3.1 Tenancy context
```python
@dataclass(frozen=True)
class ClientContext:
    client_id: UUID
    user_id: UUID | None      # None for system/worker
    role: Role | None
    actor_type: Literal["user","system","coworker","device"]
```
API dependency builds it from the JWT or device token and opens a transaction with `SELECT set_config('app.client_id', :cid, true)`. Celery tasks receive `client_id` in their payload and build a `system` context. RLS policy: `USING (client_id = current_setting('app.client_id')::uuid)`. The app DB role is **not** a superuser and has no `BYPASSRLS`.

### 3.2 Repository/service/domain split
Repository = SQL only. Domain = maths and rules. Service = transaction boundary + audit + events. Example: `services/corrections.submit()` loads the label (repo), asks `domain/bias_guard.requires_second_review(...)`, inserts the correction (repo), writes audit (same txn), publishes `notification.new` after commit.

### 3.3 Events after commit
WebSocket/Redis publishes happen **after** the DB transaction commits (outbox-lite): services append to `ctx.after_commit` callbacks executed by the session wrapper. Never publish inside an uncommitted transaction.

### 3.4 Idempotent tasks
Each Celery task is safe to run twice (unique keys, `INSERT … ON CONFLICT DO NOTHING`, state checks). `acks_late=True`, `task_reject_on_worker_lost=True`, retry with exponential backoff and jitter, max retries per stage in `pipeline/celery_app.py`. Poison messages go to a `dead_letter` table (`ops.dead_letters`) and raise a health warning.

### 3.5 Config provider
```python
cfg = config.get(ctx)                 # cached snapshot
cfg.threshold("spike_z_score")        # typed accessors; unknown key raises
cfg.weights()                         # active weight profile
cfg.vocab("ally")                     # for text generation
```
Snapshots carry a `version_hash` stored on alerts (`evidence.config_hash`) for reproducibility.

---

## 4. Binding formulas (parameters = data)

### 4.1 Influence score (account) — `domain/influence.py`
`weight_profiles.params` keys (seed values are **starting points, calibrated later**):
```json
{
  "account": {"w_tier": 0.35, "w_verified": 0.15, "w_followers": 0.30, "w_velocity": 0.20,
              "tier_factor_by_rank": {"1": 1.0, "2": 0.85, "3": 0.8, "4": 0.6, "none": 0.1},
              "followers_log_ref": 10000000, "velocity_ref_per_min": 100},
  "outlet":  {"w_tier": 0.6, "w_reach": 0.4, "reach_factor": {"high": 1.0, "medium": 0.7, "low": 0.4},
              "partisan_multiplier": 0.6},
  "bot": {"downweight": 0.2},
  "engagement_weights": {"repost": 1.0, "reply": 0.5, "like": 0.2, "view": 0.0},
  "sentiment_window_hours": 24,
  "campaign": {"w_text": 0.30, "w_age": 0.20, "w_time": 0.20, "w_boost": 0.15, "w_hashtag": 0.15},
  "severity": {"w_instant_floor": 0.85}
}
```
```
followers_factor = min(1, log10(1+followers) / log10(1+followers_log_ref))
velocity_factor  = min(1, engagement_velocity_per_min / velocity_ref_per_min)
account_influence = clamp01( w_tier*tier_factor + w_verified*verified
                           + w_followers*followers_factor + w_velocity*velocity_factor )
if author.bot_suspected: account_influence *= bot.downweight   # BR-018, shown in UI
outlet_influence = clamp01(w_tier*tier_factor + w_reach*reach_factor) * (partisan_multiplier if partisan else 1)
```
`engagement_velocity_per_min` = Δ(weighted engagement) between the two most recent snapshots in `core.engagement_snapshots` ÷ minutes between them (weighted using `engagement_weights`). First observation of a post: velocity = weighted engagement ÷ minutes since publish.

**Engagement refresh:** for X mentions where `account_influence_without_velocity ≥ 0.5` (config key `engagement_refresh_min_influence`) the system re-reads the post at +5, +10, +20 minutes. This spends X reads deliberately and only on candidates (cost rule). Snapshots stored in `core.engagement_snapshots(mention_id, captured_at, reposts, likes, replies, views)` (B11).

### 4.2 Per-post cap — `domain/capping.py` (DEC-07)
Given raw weights `w_i` for N ≥ 20 mentions, cap `c = per_post_cap`: iterate — compute shares `s_i = w_i/Σw`; for `s_i > c` set `s_i = c` and rescale the uncapped so Σ=1; repeat until no share exceeds `c` (≤ 10 passes). Property tests: shares sum to 1; none exceeds `c` (or, if N·c < 1, function raises `InsufficientSample`).

### 4.3 Sentiment aggregation — `domain/sentiment.py`
Polarity of a mention for entity e: `stance` unless `stance='sarcastic'`, then `sarcasm_target_stance` (DEC-01). Window: `sentiment_window_hours` (default 24, rolling). For each entity:
- `raw_pct_pos = positives / N`.
- `weighted_pct_pos = Σ s_i·[pos]` with capped shares from 4.2; similarly negative/neutral.
- API returns weighted and raw side by side, `mention_count = N`, `has_enough_data = N ≥ min_sample_size`, `top_contributors` (3 largest capped shares), `is_sampled` (overlaps `ops.sampling_windows`), `last_updated_at`, `corrected_count` (identifiable corrected counts, FR-H24), `view` (`effective` or `model_only`).
- **Change detection (DEC-06):** compare current window vs previous equal window: two-proportion z-test on negative share; call it a change only if both windows ≥ min sample and |z| ≥ 1.96. Otherwise the UI shows no trend arrow. Wilson 95 % interval returned with each share.

### 4.4 Severity and tier — `domain/severity.py`
```
influence_term = max influence among the Story/alert's contributing mentions, or the mention's own influence
negativity     = weighted share of negative polarity (for a single post: 1.0 if negative else 0.0)
velocity_term  = min(1, max( volume_velocity / volume_velocity_ref,  engagement_velocity_per_min / velocity_ref_per_min ))
severity       = influence_term × negativity × velocity_term              # PRD formula
tier = 1 if severity ≥ severity_tier1_min
       2 if severity ≥ severity_tier2_min
       3 otherwise                                                        # DEC-05: tier 3 = no alert row
```
**Instant path (DEC-20, resolves the PRD's "within 5 minutes of the post"):** if the author's `account_influence ≥ severity.w_instant_floor` (0.85) **and** the mention is negative about a client-primary entity **and** claim_type ≠ `satire`, set `velocity_term = 1` for the first evaluation so the alert fires at ingestion; reason text shows engagement *so far* ("verified, 1.2M followers, national media tier, 4,000 reposts in 20 min" when available, else "…, posted 3 min ago"). Without this, a brand-new post has zero velocity and cannot reach Tier 1 in time. Flagged for client confirmation (G-23).

### 4.5 Spike detection — `domain/spike.py`
Every `spike_eval_interval_minutes` for each client-primary entity and each ally/supporting party:
```
x      = negative mentions in the last 60 min (effective polarity)
for d in 1..spike_baseline_days: c_d = negatives in the same clock hour on day -d    # or event baseline
μ, σ   = mean(c_d), sd(c_d);  σ_eff = max(σ, sqrt(max(μ,1)), 1)
cond   = x ≥ μ + spike_z_score·σ_eff  AND x ≥ spike_min_multiplier·μ  AND x ≥ spike_min_mentions
fire when cond held on evaluations spanning ≥ spike_window_minutes
```
Event baseline: if `now` is inside a calendar event for that entity (or event with NULL entity), use the baseline from the most recent past events of the **same `event_type`** (up to 5), same offset-from-start window; if none exist use standard baseline and mark `evidence.baseline_kind='standard_fallback'`. Cold start (< 7 days data): use `spike_cold_start_min_mentions` instead of the z-score test and label the alert "provisional baseline". Alert payload: baseline μ, current x, top contributing mentions by capped weight.

### 4.6 Campaign confidence — `domain/campaign_score.py`
Candidate cluster = ≥ `campaign_min_accounts` distinct authors posting text with pairwise similarity ≥ `campaign_text_similarity_min` (MinHash on character 5-grams, confirmed by embedding cosine) within `campaign_window_minutes`, **or** ≥ that many authors posting the same hashtag where ≥ 50 % of those authors have no other posts about the entity. Signals (each 0..1):
- `S_text` = clustered accounts ÷ all accounts posting about the entity in window (capped 1);
- `S_age` = share of cluster accounts younger than `campaign_new_account_days`;
- `S_time` = 1 − (post-time spread ÷ window), i.e. how tightly posts are timed;
- `S_boost` = density of repost/reply/quote edges inside the cluster;
- `S_hashtag` = Herfindahl concentration of hashtags used by the cluster.
`confidence = Σ w_k·S_k` (weights in the weight profile). Level text: `< 0.5 low`, `0.5–0.75 medium`, `> 0.75 high`. Alert when `confidence ≥ campaign_confidence_alert_min`. **The alert text and API never contain a party/entity as the actor** — only the *target* entity (DG-13). Evidence payload: account list (handle, age, followers, bot flag), the shared phrasing with variants, per-minute timing series, signal values.

### 4.7 Response opportunity — `domain/response_trigger.py` (DEC-08, DEC-09)
Evaluated on every Story update touching a client-primary entity and on a 60 s sweep of open candidates. Output: `OpportunityState(open | none)`, `who_reacting`, `suggested_response_type`. Suggested type from the story's dominant `reaction_type` and entity polarity: attack→`condemn`/`rebut`, positive milestone→`congratulate`, death/accident keywords by classifier→`condole`. "Who is reacting" = distinct entity names via `entity_accounts` ∪ outlet names, non-client-side only.

### 4.8 Bot suspicion heuristic — `nlp/bot_heuristics.py` `[PROPOSED]`
Score = weighted sum of: account age < 30 d, followers/following < 0.1, post_count/age > 200/day, default-profile flag, numeric-suffix handle pattern, `lang` mismatches. Never labelled "bot" as fact — UI text is "suspected bot". Weights are data. Calibrated on campaign confirmations by leads.

### 4.9 Staging — `domain/staging.py`
Weekly job (and nightly check): for each capability with `n ≥ stage_min_sample` corrections-evaluated items in the window, compute metric (F1 vs gold for labels; precision/accept rate for alerts/flags/suggestions/briefs). Promote if metric ≥ bar for `stage_promote_weeks` consecutive weeks; demote one stage immediately when below bar (story 35), writing `learning.stage_transitions` with a human-readable reason and a `stage_demoted` notification. Bars are NULL until calibration — **with a NULL bar a capability stays at `assist`** and the admin page says "bar not set".

---

## 5. Labelling and prompts

- **Prompts are files** `app/prompts/<capability>/<version>.md`; the file's SHA-256 is stored as `prompt_version` suffix (`label-v1+ab12cd`). A changed prompt without a version bump fails CI.
- **Label output schema (Pydantic):**
  `{entities:[{entity_id, stance, sarcasm_target_stance?, claim_type, is_attack, reaction_type, confidence, reason}], language, language_confidence, quotes:[{text, speaker?}]}`. `entity_id` must be one of the candidate entities passed in; anything else is dropped and logged.
- Candidate entities come from `entities_matched` (watch-term hits), so the model chooses stance **per entity**, not entity discovery.
- Few-shot examples are selected from `learning.gold_set` (same language, per entity, most recent corrections first, ≤ 6 examples). This is how "corrections teach the system" works without fine-tuning in Phase 1; fine-tuning is out of scope.
- Escalation: confidence `< label_confidence_escalate_below` → strong route; still `< label_confidence_human_below` → human review queue (a view over mentions needing review, surfaced in the Stories page filter "needs review").
- Reason language: English (staff-facing); Tamil reason translation on demand is a later enhancement (gap G-24).

---

## 6. Co-worker rules (`app/coworker`)

**Tools (read-only; each logs its query id):**
`search_mentions(filters)`, `count_mentions(filters, group_by)`, `compare_periods(entity, a, b)`, `get_story(id)`, `list_alerts(filters)`, `get_alert(id)`, `list_open_opportunities()`, `entity_info(id)`, `recall_memory(query)`, `remember(memory_type, summary, refs)` (only writes `learning.coworker_memory`, rejects text naming private individuals — see below).
No tool can send, post, change config, correct labels, or confirm anything.

**Grounding contract (BR-041, 13 §4.2):** the model must return JSON `{answer_text, claims:[{text, mention_ids[], numbers:[{value, query_id}]}], language}`. `grounding.validate()` checks (a) every `mention_id` was returned by a tool call in this run **for this client**; (b) every number equals the result of its `query_id` (exact for counts, ±0.1 pt for percentages); (c) no claim without citations unless marked `is_opinion=false` and purely procedural. Failure → one automatic retry with the validator's error → else the answer is replaced by "I can't verify this from the data" and logged. Counts are **never** typed by the model from memory.
**Insufficient data:** if a relevant query has N < `min_sample_size`, the answer must say "not enough data (N=…)".
**Language:** answer in the question's language; for Tanglish questions default to Tamil script (08 §7.2). Detect with `nlp/language.py`.
**Memory:** facts about events, narratives, actions and their outcomes. **Never store** handles or personal details of private individuals: `remember()` runs `redaction.contains_private_author()` and refuses. Memory rows are correctable (soft-delete + `superseded_by`) via output corrections.
**Latency:** first token ≤ 10 s typical. Stream via WS event `coworker.answer`; tool calls parallelised; cap 6 tool rounds.
**Ally blame gate:** `briefs.generate_cm_summary()` runs `domain.ally_blame.detect()` (rules: a change whose cause names an ally-tagged entity **as actor of negativity**, using entity relationship at generation time) and sets `needs_ally_review`. The detector is deliberately over-inclusive; false positives only cost a lead one click.

**Eval datasets** live in `backend/evals/datasets/*.jsonl` (question, expected facts, expected citations, `must_say_unknown`). They start synthetic (from the demo seed) and are replaced by real war-room questions when supplied (G-gold).

---

## 7. API conventions
- Pagination `page`/`page_size` (max 200). Lists return `total`. Sort params validated against an allow-list.
- Timestamps ISO-8601 UTC with `Z`. IDs UUID strings.
- Idempotency: `POST` that creates (uploads, corrections) accept `Idempotency-Key` header; duplicate → original response.
- Role guards declared on the route decorator: `@requires(Role.war_room_lead)`; the permission matrix test (18) iterates routes against 03 + `DOC-RECONCILIATION` A1.
- Presentation-safe: every endpoint returning Mention/author/user data passes the response model through `redaction.apply(ctx, model)`; when `presentation_mode_active` **or** the caller is a device/CM View, redaction runs. Unit tests assert that private handles, internal notes, correction details never appear.
- Rate limiting: 120 req/min per user (429 `RATE_LIMIT`), co-worker 20/min.
- OpenAPI generated; frontend client generated from it in CI; drift fails CI.

---

## 8. Frontend rules
- **Number components** in `components/numbers/` are mandatory for any statistic: `StatWithDenominator` (value + "of N"), `NotEnoughData`, `WeightedRaw` (weighted with raw beside it and tap-to-top-3), `SampledBadge`, `StaleStamp` ("as of HH:MM", amber when stale). A custom ESLint rule `warroom/no-bare-percent` flags `%` formatting outside them.
- Every number is a link/button to its evidence (06 principle 1): the components take an `evidenceQuery` prop.
- Tier-1 flash: CSS animation driven by store state `unacknowledgedTier1Ids`; clears on WS `alert.acknowledged`.
- Realtime store: single `useRealtime()` hook subscribes, applies events to React Query caches, shows "Reconnecting…" banner after 5 s without heartbeat (server pings every 20 s).
- Big screen: no pointer-dependent UI; auto-recovers from WS loss; polls `/bigscreen/summary` every 60 s as fallback; **page-level watchdog** reloads if no successful data in 3 minutes and shows "data delayed since HH:MM" meanwhile (never a blank or stale-without-stamp screen).
- i18n: `en.json`, `ta.json`; keys required in both (CI diff check); vocabulary via `vocab.*` placeholders; date/number formatting via `Intl` with the user's locale and the client timezone.
- CM View: no navigation chrome except language switch; tap-to-expand evidence loads from `/cm-summary/:id/evidence/:idx`.

---

## 9. Logging, errors, observability
- `structlog` JSON: `ts, level, request_id, client_id, user_id, task, duration_ms`. Sanitiser removes keys matching `password|secret|token|authorization|whatsapp_number|api_key`.
- Metrics (Prometheus): `pipeline_stage_duration_seconds{stage}`, `pipeline_queue_depth{queue}`, `post_to_screen_seconds` (histogram, from `ops.pipeline_latency`), `alerts_fired_total{type,tier}`, `whatsapp_accepted_seconds`, `llm_tokens_total{route}`, `x_reads_total`, `source_health{kind,ref}`, `coworker_answer_seconds`.
- Error handling at the edge: users see the `message` from the envelope; internal details only in logs. Co-worker failures show the exact strings in 08 §3.2/§3.5.
- Tracing IDs propagate API → Celery headers → logs.

## 10. Security checklist for every PR
Input validated by Pydantic; role guard present; tenant scoping via repository; audit write present for mutations; no raw SQL string formatting; no secrets; presentation redaction applied for Mention-bearing responses; new dependency license/CVE check passes; prompt-injection test added for any new LLM call.

## 11. Dependencies
Pin exact versions (`uv.lock`/`package-lock.json`); weekly Dependabot; license allow-list (MIT/BSD/Apache-2.0/ISC/MPL-2.0; no AGPL/SSPL in the runtime path). New heavy ML dependency requires an entry in 14 §11.

## 12. Git and review workflow
One task per branch/PR; PR description lists task ID, acceptance criteria checked, BR/FR/DG ids touched, screenshots for UI, and the test commands run. CI gates: lint, types, unit, integration, import-linter, hard-coding check, OpenAPI drift, i18n parity, migrations up/down. Coverage: `domain` ≥ 90 %, `services` ≥ 80 %, overall ≥ 80 %.

## 13. Definition of done (per task)
Code + tests + docs touched + audit/permissions/i18n/redaction considered + acceptance criteria demonstrably met (command output or screenshot in PR) + no new lint/type errors + traceability tag in code. See 24 for the release-level checklist.
