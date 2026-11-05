# 15 — Project Structure

**Decision (binding):** one **monorepo**, one Python backend package used by the API and the workers, one Next.js frontend. Technology as in 10 and `DOC-RECONCILIATION.md` (Celery, not Redis Streams).

---

## 1. Repository layout

```
warroom/
├── README.md                      # points to /docs/README.md
├── Makefile                       # make dev | test | lint | migrate | seed | e2e  (20 §commands)
├── docker-compose.yml             # local stack (20)
├── docker-compose.test.yml
├── .env.example                   # every variable documented (20 §env)
├── .pre-commit-config.yaml
├── .github/workflows/             # ci.yml, deploy-staging.yml, deploy-prod.yml, evals.yml
├── docs/                          # this documentation package (source of truth)
│
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── importlinter.ini           # architectural contracts (16 §boundaries)
│   ├── migrations/versions/       # ordered migrations (§3)
│   ├── app/
│   │   ├── main.py                # FastAPI app factory
│   │   ├── settings.py            # pydantic-settings; the ONLY reader of env vars
│   │   ├── api/
│   │   │   ├── deps.py            # auth, role guards, db session with RLS, pagination
│   │   │   ├── errors.py          # error envelope + codes (12 §14, DOC-RECONCILIATION C13)
│   │   │   └── v1/
│   │   │       ├── auth.py  me.py  dashboard.py  bigscreen.py  mentions.py
│   │   │       ├── corrections.py  output_corrections.py  stories.py  alerts.py
│   │   │       ├── campaigns.py  coworker.py  briefs.py  cm_summary.py
│   │   │       ├── uploads.py  response_queue.py  parties.py  events.py
│   │   │       ├── notifications.py  presentation_mode.py  health.py
│   │   │       ├── admin_config.py  admin_users.py  admin_devices.py
│   │   │       ├── admin_health.py  admin_accuracy.py  admin_audit.py
│   │   │       ├── admin_filter_audit.py  admin_gold_set.py
│   │   │       └── webhooks.py    # whatsapp
│   │   ├── ws/
│   │   │   ├── manager.py         # connections per client; Redis pub/sub bridge
│   │   │   └── events.py          # event name constants + payload models
│   │   ├── domain/                # PURE logic, no I/O. Heavily unit-tested.
│   │   │   ├── influence.py       # weight formula (§16 guidelines)
│   │   │   ├── capping.py         # per-post cap (DEC-07)
│   │   │   ├── sentiment.py       # polarity, weighted/raw, Wilson CI, z-test (DEC-06)
│   │   │   ├── severity.py        # severity = influence × negativity × velocity → tier
│   │   │   ├── spike.py           # baseline, z-score, event baseline
│   │   │   ├── campaign_score.py  # coordination signals → confidence
│   │   │   ├── response_trigger.py# DEC-08
│   │   │   ├── bias_guard.py      # DEC-02, DEC-03
│   │   │   ├── staging.py         # promote/demote rules (story 34/35)
│   │   │   ├── redaction.py       # redact_for_presentation() (DEC-11)
│   │   │   └── text.py            # normalise, text_hash, snippet cap, tamil_numbers
│   │   ├── models/                # SQLAlchemy models, one file per schema
│   │   │   ├── accounts.py config.py core.py learning.py audit.py ops.py
│   │   ├── schemas/               # Pydantic request/response models (mirror 12 + addendum)
│   │   ├── repositories/          # all SQL; every function takes ClientContext
│   │   ├── services/              # orchestration, transactions, audit writes
│   │   │   ├── alerts.py corrections.py stories.py campaigns.py briefs.py
│   │   │   ├── dashboard.py presentation.py whatsapp_sender.py notifications.py
│   │   │   ├── config_admin.py usage.py staging_service.py filter_audit.py
│   │   ├── pipeline/              # Celery tasks, one module per stage (§2)
│   │   │   ├── celery_app.py  schedules.py
│   │   │   ├── s1_collect.py  s2_filter.py  s3_label.py  s4_weigh.py
│   │   │   ├── s5_stories.py  s6_campaigns.py  s7_alerting.py
│   │   │   ├── periodic_spike.py  periodic_ally.py  periodic_tvk_response.py
│   │   │   ├── periodic_staging.py  periodic_metrics.py  periodic_audit_sample.py
│   │   │   ├── periodic_retention.py  periodic_health.py
│   │   │   └── backfill.py
│   │   ├── integrations/          # one package per external system (14)
│   │   │   ├── x/ (base.py polling.py stream.py vendor.py queries.py)
│   │   │   ├── news/ (feed.py sitemap.py crawl.py search.py base.py)
│   │   │   ├── youtube/  epaper/  ocr/ (base.py tesseract.py cloud.py)
│   │   │   ├── whatsapp/ (base.py meta_cloud.py)
│   │   │   └── llm/ (base.py anthropic.py openai.py google.py router.py)
│   │   ├── nlp/                   # language detection, embeddings, tamil_numbers, bot heuristics
│   │   ├── coworker/
│   │   │   ├── agent.py           # tool-calling loop
│   │   │   ├── tools.py           # read-only data tools (§ co-worker tools in 16)
│   │   │   ├── grounding.py       # citation + number validation (13 §4.2)
│   │   │   ├── memory.py  briefs.py  explain.py  language.py
│   │   │   └── safety.py          # content-as-data wrapping, injection tests hook
│   │   ├── prompts/               # versioned prompt files (§ prompts)
│   │   │   ├── label/v1.md  explain_alert/v1.md  coworker_chat/v1.md
│   │   │   ├── cm_summary/v1.md  handover/v1.md  language_fallback/v1.md
│   │   └── common/                # clock, ids, logging, metrics, tenancy context
│   ├── seeds/
│   │   ├── tvk/                   # client profile seed = DATA (YAML/CSV), not code
│   │   │   ├── client.yaml  entities.csv  watch_terms.csv  tiers.yaml
│   │   │   ├── outlets.csv  thresholds.yaml  weights.yaml  recipients.csv  model_routes.yaml
│   │   └── demo/                  # fictional client "Acme Beverages" proving generalisation
│   ├── tests/                     # structure in 18
│   └── evals/                     # co-worker eval harness + datasets (18 §evals)
│
├── frontend/
│   ├── package.json  next.config.js  tailwind.config.ts  tsconfig.json
│   ├── src/
│   │   ├── app/                   # Next.js App Router
│   │   │   ├── (auth)/login/  (auth)/pin/
│   │   │   ├── (team)/dashboard/  stories/ stories/[id]/  alerts/ alerts/[id]/
│   │   │   │   campaigns/ campaigns/[id]/  parties/ parties/[id]/  co-worker/
│   │   │   │   uploads/  response-queue/  briefs/ notifications/  settings/
│   │   │   ├── (admin)/admin/config/ admin/health/ admin/accuracy/ admin/audit/ admin/filter-audit/ admin/users/
│   │   │   ├── bigscreen/         # kiosk, device-token auth
│   │   │   └── cm-summary/        # CM View
│   │   ├── components/            # ui primitives (Radix wrappers), charts, feature components
│   │   │   ├── evidence/ (MentionRow, LabelWithReason, CorrectionPanel, OriginalHistory)
│   │   │   ├── numbers/  (StatWithDenominator, WeightedRaw, NotEnoughData, SampledBadge, StaleStamp)
│   │   │   ├── alerts/  stories/  campaigns/  coworker/  health/
│   │   ├── lib/
│   │   │   ├── api/               # generated client from OpenAPI + thin wrappers
│   │   │   ├── ws/                # WebSocket hook, reconnect, event store
│   │   │   ├── i18n/              # en.json, ta.json, vocabulary(client) substitution
│   │   │   ├── auth/  format/ (time zone, numbers)  guards/
│   │   ├── stores/                # zustand
│   │   └── styles/
│   ├── public/                    # no client artwork committed; avatar served from object storage URL in config
│   ├── tests/                     # jest + RTL
│   └── e2e/                       # playwright
│
├── infra/
│   ├── docker/ (backend.Dockerfile worker.Dockerfile frontend.Dockerfile playwright-worker.Dockerfile)
│   ├── terraform/ (modules: network, rds, redis, s3, ecs, secrets, monitoring; envs: staging, prod)
│   └── runbooks/
└── scripts/
    ├── seed_client.py             # loads seeds/<client>/ into DB (no code change per client)
    ├── replay_scenario.py         # must-pass replays (18 §must-pass)
    ├── gen_openapi_client.sh
    └── check_hardcoding.py        # fails CI on client-specific literals (16 §no hard-coding)
```

---

## 2. Pipeline modules ↔ Celery queues

| Stage module | Queue | Concurrency (initial) | Idempotency key | Next |
|--------------|-------|----------------------|-----------------|------|
| `s1_collect` | `collect_x`, `collect_news`, `collect_epaper` | 2 / 4 / 1 | `(client, source_type, external_id)` unique | `s2_filter` |
| `s2_filter` | `filter` | 4 | mention id + `filter_version` | `s3_label` if matched |
| `s3_label` | `label_cheap`, `label_strong` | 8 / 2 | mention id + `prompt_version` + `model_name` | `s4_weigh` |
| `s4_weigh` | `weigh` | 4 | mention id + `weight_version` | `s5_stories` |
| `s5_stories` | `stories` | 2 (serialised per client by advisory lock) | mention id | `s6_campaigns`, `s7_alerting` |
| `s6_campaigns` | `campaigns` | 1 per client | window id | `s7_alerting` |
| `s7_alerting` | `alerts` (**highest priority, dedicated workers**) | 2 | `dedupe_key` | WS + `whatsapp` |
| WhatsApp sending | `whatsapp` | 2 | alert id + recipient + template | — |
| Embeddings | `embed` | 2 | mention id + `embedding_model` | — |
| OCR | `ocr` | 2 | upload id | `s2_filter` |
| Co-worker | `coworker` | 4 | request id | — |
| Periodic | `periodic` | 2 | — | — |

**Latency rule:** queues are ordered so the single-critical-post path (`collect_x` → `filter` → `label_cheap` → `weigh` → `alerts`) never waits behind backfill or bulk news. Backfill uses queue `backfill` with low priority and its own workers. A stage that hits the same mention twice is a no-op (idempotent).

---

## 3. Migration order (REC-02)

`0001_extensions_schemas` (pgcrypto, vector, schemas) → `0002_config_core_tables` (`config.clients`, `accounts.users`, `accounts.sessions`) → `0003_config_tables` → `0004_core_campaigns` → `0005_core_mentions_labels_stories_alerts` (**campaigns before stories**) → `0006_learning_audit` → `0007_reconciliation_addendum` (everything in `DOC-RECONCILIATION` Part B, including B11) → `0008_views` → `0009_rls_policies` → `0010_db_roles_grants` → `0011_triggers` (`updated_at`, audit guard, label-correction guard, append-only on `audit.audit_log`).

Migrations are forward-only in staging/production; each has a tested downgrade in CI on a scratch DB.

---

## 4. File ownership boundaries (who may import whom)

```
api  ─►  services  ─►  repositories  ─►  models
 │          │
 │          └─►  domain (pure)         integrations (I/O adapters)
 └─►  schemas                           ▲
pipeline ─► services / repositories / domain / integrations / nlp / coworker(explain only)
coworker ─► repositories (READ-ONLY views), domain, integrations.llm
```
Rules enforced by `import-linter` (config in `backend/importlinter.ini`):
1. `domain` imports nothing outside the standard library, `numpy`, `scipy`, `datetime`.
2. `coworker` **must not import** `integrations.whatsapp`, `services.whatsapp_sender`, `services.alerts` write paths, `integrations.x` (BR-043, 13 §4.3).
3. `api` never imports `repositories` directly (goes through `services`).
4. Only `settings.py` reads `os.environ`.
5. Only `integrations/llm/router.py` knows provider names.
6. No module outside `repositories` writes raw SQL, except migrations.

---

## 5. Naming and conventions
- Python: modules `snake_case`, classes `PascalCase`, Celery tasks `warroom.<stage>.<action>`.
- DB: as 11. Migrations named `NNNN_<what>.py`.
- API: JSON keys `snake_case` (as 12). Frontend maps to camelCase only inside `lib/api`.
- Branches: `feat/<task-id>-<slug>`; one task = one branch = one PR (21 §workflow).
- Commit message: `<task-id>: <imperative summary>` e.g. `T-ALR-03: add influence trigger evaluator`.

---

## 6. Where each doc's content lands in code

| Doc | Lands in |
|-----|----------|
| 03 permissions | `api/deps.py` role guards + `tests/security/test_permission_matrix.py` (generated from the matrix) |
| 05 business rules | `domain/*` + `services/*`; each BR id appears in a docstring tag `# BR-0xx` |
| 07/08 screens/states | `frontend/src/app/*` and `components/*` |
| 11 + addendum | `models/*`, `migrations/*` |
| 12 + addendum | `api/v1/*`, `schemas/*` |
| 13 | `api/deps.py`, `services/*`, DB grants, `coworker/safety.py` |
| 14 | `integrations/*` |
