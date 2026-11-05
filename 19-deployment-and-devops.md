# 19 — Deployment and DevOps

**Reads with:** 09 (architecture), 10 (stack), 13 (security), 14 (integrations), 15 (structure), 16 (guidelines), 20 (setup). Failure behaviour for each dependency is defined in 14 §12 and is not repeated here.

**Status markers:** `[PRD]` stated in the PRD · `[PROPOSED]` technical recommendation, not in the PRD · `[CTO DECISION]` explicitly left to the CTO · `[VERIFY]` depends on a provider's current terms or versions · `[GAP]` information the client or CTO must supply.

The PRD leaves infrastructure to the CTO (cloud choice, multi-tenancy model). Everything below is a proposal sized for the PRD's constraints: p95 post-to-screen ≤ 5 min `[PRD]`, India data residency for DPDP `[PRD: DPDP applies; region is PROPOSED]`, no monthly budget cap but cost efficiency designed in `[PRD]`.

---

## 1. Environments

| Env | Purpose | Where | Data | External services |
|---|---|---|---|---|
| `local` | Developer / OpenHands work | Docker Compose (20) | Seeds only (`demo`, `tvk` placeholders) | All mocked (X, WhatsApp, LLM, news site) |
| `ci` | Per-PR automated tests | GitHub Actions runners + testcontainers | Seeds + fixtures | All mocked; real LLM only in `evals.yml` |
| `staging` | Integration, UAT rehearsal, replays | AWS `ap-south-1` | Synthetic only. **No production data ever copied down.** | Mocks by default; WhatsApp sandbox numbers for must-pass rehearsal; real X only after G-01 closes |
| `production` | The TVK war room | AWS `ap-south-1` | Real | Real X (tier per G-01), real WhatsApp Business account, real LLM routes |

Rules:
- One Terraform root per env (`infra/terraform/envs/staging`, `envs/prod`) sharing modules; no manual console changes (drift is a defect).
- Staging mirrors production topology at smaller size so queue-priority and RLS behaviour are tested for real.
- A new client in production is a seed/config operation (`scripts/seed_client.py`), never a deploy.

## 2. Runtime topology `[PROPOSED]`

Cloud: AWS Mumbai (`ap-south-1`). GCP/Azure Mumbai are equivalent options `[CTO DECISION]`; Terraform modules isolate provider specifics.

```
Internet ── CloudFront (static assets) ──┐
        └─ ALB (HTTPS, WAF) ─────────────┴─► ECS: api (REST + WebSocket)
                                              ECS: frontend (Next.js)
                     private subnets:         ECS: workers (per queue group) + beat
                                              RDS PostgreSQL 16 (+pgvector) Multi-AZ
                                              ElastiCache Redis (AOF on)
                                              S3: wr-raw, wr-uploads, wr-pages
                                              Secrets Manager
Egress (NAT) ─► X API, news sites, YouTube, WhatsApp Cloud API, LLM providers, search API
```

### 2.1 Services (images from 15 §1 `infra/docker/`)

| Service | Image | Command / queues | Min tasks | Scale on |
|---|---|---|---|---|
| `api` | `backend.Dockerfile` | uvicorn app (REST + WS) | 2 | CPU, WS connection count |
| `frontend` | `frontend.Dockerfile` | `next start` | 2 | CPU |
| `worker-alerts` | `worker.Dockerfile` | queues `alerts`, `whatsapp` (**dedicated; never scaled to zero; highest priority**) | 2 | queue lag |
| `worker-collect-x` | `worker.Dockerfile` | `collect_x` | 2 | — (poll-driven) |
| `worker-collect-news` | `worker.Dockerfile` | `collect_news`, `collect_epaper` | 2 | queue depth |
| `worker-pipeline` | `worker.Dockerfile` | `filter`, `weigh`, `stories`, `campaigns` | 2 | queue lag |
| `worker-label` | `worker.Dockerfile` | `label_cheap`, `label_strong` | 3 | queue lag, LLM rate limits |
| `worker-embed` | `worker.Dockerfile` | `embed` (CPU; GPU optional) | 1 | queue depth |
| `worker-ocr` | `worker.Dockerfile` | `ocr` | 1 | queue depth |
| `worker-coworker` | `worker.Dockerfile` | `coworker` | 2 | queue lag |
| `worker-periodic` | `worker.Dockerfile` | `periodic` | 1 | — |
| `worker-backfill` | `worker.Dockerfile` | `backfill` (low priority, own workers so it never delays live paths) | 0–1 | on demand |
| `beat` | `worker.Dockerfile` | Celery beat (`pipeline/schedules.py`) — **exactly one instance** | 1 | — |
| `worker-playwright` | `playwright-worker.Dockerfile` | JS-heavy crawl only (`requires_js=true`) | 0–1 | on demand |

Initial concurrency per queue is in 15 §2. Concurrency, retries and backoff are settings, not code.

**Latency rule `[PRD freshness; PROPOSED mechanism]`:** the critical path `collect_x → filter → label_cheap → weigh → alerts` must never queue behind backfill or bulk news. This is why `alerts`/`whatsapp` have dedicated workers and `backfill` has its own pool. Verified by TC-NFR-01.

## 3. Infrastructure as code

`infra/terraform/` modules (15): `network`, `rds`, `redis`, `s3`, `ecs`, `secrets`, `monitoring`; envs: `staging`, `prod`.

| Module | Key settings |
|---|---|
| `network` | VPC, 2+ AZ, public subnets (ALB, NAT only), private subnets (everything else), security groups per 13 §6.1 |
| `rds` | PostgreSQL 16, Multi-AZ, encrypted at rest, `pgvector` ≥ 0.7 `[VERIFY]`, PgBouncer sidecar/service in **transaction mode** (RLS uses `SET LOCAL app.client_id`), automated backups + PITR, no public endpoint |
| `redis` | ElastiCache, AOF persistence on (queued Celery tasks survive restart), encrypted |
| `s3` | Buckets `wr-raw`, `wr-uploads`, `wr-pages`; SSE on, public access blocked, versioning on, lifecycle rules implementing `retention_days_raw` |
| `ecs` | Fargate services of §2.1, task roles with least privilege, autoscaling policies |
| `secrets` | Secrets Manager entries (§4) with rotation config where supported |
| `monitoring` | Log groups, metric alarms (§7), dashboards, SNS/paging integration |

The **co-worker task role has no access** to the WhatsApp secret or the X secret (BR-043, 13 §4.3). Only `worker-alerts`/`whatsapp` tasks may read WhatsApp credentials. This is checked by TC-SEC-04.

## 4. Configuration and secrets

- `backend/app/settings.py` is the **only** reader of environment variables (16 / 15 §4). Env vars are listed with defaults in 20 §4.
- Secrets live in Secrets Manager and are injected as environment variables at task start. Never in code, images, logs, prompts or error messages (DG-15).
- Two DB roles: **app role** (no superuser, no `BYPASSRLS`, no UPDATE/DELETE on `core.mention_labels` or `audit.audit_log`) and **migrator role** (used only by the migration job). Role grants come from migration `0010_db_roles_grants`.

| Secret | Used by | Rotation |
|---|---|---|
| DB app password / migrator password | api, workers / migration job | Scheduled |
| `JWT_SECRET` | api | Manual, with refresh-token revocation |
| X credentials (one set per client — BR-054) | `worker-collect-x` | Manual (runbook) |
| WhatsApp access token, app secret, verify token | `worker-alerts`, api webhook | Manual |
| LLM provider keys | label, coworker workers | Manual |
| YouTube API key, search-provider key | collect workers | Manual |
| SMTP/SES credentials | api | Manual |

Staff-editable settings (thresholds, tiers, watch terms, recipients, vocabulary, weights) are **database rows, not environment config** (DG-10, PRD). Changing one never needs a deploy.

## 5. CI/CD

GitHub Actions workflows (`.github/workflows/`, 15):

| Workflow | Trigger | Does |
|---|---|---|
| `ci.yml` | every PR / push | Lint, types, unit, integration (testcontainers), import-linter, `check_hardcoding.py`, OpenAPI drift, i18n parity, migration up/down cycle, permission-matrix test, cross-client attack suite, security suite, smoke E2E, coverage gates (16 §12) |
| `evals.yml` | change to `prompts/**` or `model_routes` seed; nightly | Co-worker/label evals with real LLM routes (18 §6) |
| `deploy-staging.yml` | merge to `main` | Build images → push to registry → migrate → deploy → smoke E2E |
| `deploy-prod.yml` | manual approval after staging green | Same sequence on production |
| nightly (scheduled in `ci.yml` or separate) | cron | Full E2E, scenario replays TC-MP1/MP2 and others, performance, dependency scan; ZAP weekly (18 §10) |

### 5.1 Release sequence

1. Build images tagged with the git SHA (never `latest`). Scan images and dependencies.
2. Run the **migration job** as a one-off task with the migrator role. Migrations are forward-only in staging/production; each has a tested downgrade in CI only (15 §3).
3. Roll the services: `api`, then workers, then `frontend`. Workers use `acks_late=True` and `task_reject_on_worker_lost=True` (16 §3.4) so in-flight tasks are retried, not lost. Graceful shutdown on SIGTERM; ECS stop timeout ≥ longest task timeout.
4. Smoke E2E against the environment (login, dashboard, replay `one_post_mattered` on staging).
5. Production only: manual approval; after deploy, verify health page shows all sources `healthy` or expectedly `delayed`.

### 5.2 Database change policy `[PROPOSED]`

Expand/contract: add columns/tables in release N (nullable or defaulted), switch code in N+1, drop in N+2. Never rename in place. Every migration ships with its model and test (16 §2 SQL).

### 5.3 Rollback

- Application: redeploy the previous image tag (workers and api independently).
- Database: roll **forward** with a fix migration; restore from PITR only for data corruption (§8).
- Config: every settings change is audit-logged with old/new values and can be reverted through the admin API.
- A bad prompt or `model_route` change is rolled back by reverting the prompt file/seed; `prompt_version` is recorded on every output (DG-12), so affected outputs are identifiable.

## 6. Observability

Logging: `structlog` JSON to CloudWatch (or equivalent); the sanitiser removes keys matching `password|secret|token|authorization|whatsapp_number|api_key` (16 §9). Request/trace IDs propagate API → Celery headers → logs.

Metrics (Prometheus naming from 16 §9): `pipeline_stage_duration_seconds{stage}`, `pipeline_queue_depth{queue}`, `post_to_screen_seconds`, `alerts_fired_total{type,tier}`, `whatsapp_accepted_seconds`, `llm_tokens_total{route}`, `x_reads_total`, `source_health{kind,ref}`, `coworker_answer_seconds`.

### 6.1 Operational alerts (paging) `[PROPOSED thresholds]`

| Condition | Why | Severity |
|---|---|---|
| `post_to_screen_seconds` p95 > 240 s for 5 min | PRD target is ≤ 300 s; page before breach | Page |
| `alerts` or `label_cheap` queue wait p95 > 20 s | 18 §7 design target | Page |
| Any `source_health` = `down` for X, WhatsApp, LLM, Redis or Postgres | PRD failure behaviour | Page |
| `whatsapp_accepted_seconds` > 300 for a Tier-1 alert | Must-pass SLA (DEC-04) | Page |
| WhatsApp failures after max retries | Story 29 | Page |
| Dead-letter count > 0 (`ops.dead_letters`) | Poison message | Ticket |
| X reads ≥ 80 % / 95 % of cap | FR-I30 (also raised in-app to admins) | Ticket / Page |
| LLM error rate or latency above budget | Co-worker/label degradation | Ticket |
| LLM cost per item +30 % week over week | 18 §7 trend alert (not a spend cap — PRD has none) | Ticket |
| Celery beat missing heartbeat | Periodic jobs (spike, audit, staging) silent | Page |
| DB connections/CPU/storage, Redis memory | Capacity | Ticket |
| TLS certificate < 14 days | Availability | Ticket |

The app's own "data delayed since HH:MM" banner (FR-I28) is **user-facing**; these alerts are **operator-facing**. Neither replaces the other.

## 7. Backup and disaster recovery

The PRD gives no RPO/RTO `[GAP — PRD-GAP-ANALYSIS §5]`. Proposed starting targets for CTO confirmation:

| Asset | Mechanism | Proposed target |
|---|---|---|
| PostgreSQL | Multi-AZ, automated backups + PITR (≥ 7 days) | RPO ≤ 5 min, RTO ≤ 1 h |
| Object storage | Versioning + cross-AZ durability | Per S3 durability |
| Redis | AOF persistence; queue is rebuildable from `collect_*` cursors and `ops.ingestion_cursors` | Rebuild on loss; dedupe keys prevent double processing (BR-002) |
| Audit log | Append-only table (13 §5.4), included in DB backups | Retention ≥ 1 year `[PROPOSED — confirm with client]` |

Restore drill: quarterly in staging from a production snapshot **after redaction of personal data** `[PROPOSED]`. Record the time taken.

Degraded-mode behaviour during a DB outage is defined in 09 §5 and 14 §12 (maintenance page; queue holds; read replicas serve reads).

## 8. Data residency, retention and privacy operations

- **Residency:** all production data in India region `[PROPOSED: ap-south-1]`. Terraform/org policy denies resource creation outside the approved region.
- **LLM provider region and retention terms:** to be confirmed before production (G-19). Prompts contain public posts/articles only; WhatsApp numbers and passwords are never sent to a provider (14 §8).
- **Retention:** retention rules **must be set before ingestion starts** (PRD pre-mortem; BR-048; **[GAP]** periods not defined). The system enforces this: ingestion refuses to start while `retention_days_raw` and related client settings are unset (TC-NFR-04). Jobs: `pipeline/periodic_retention.py`; S3 lifecycle rules mirror `retention_days_raw`.
- **Source deletions:** honour X post/account deletion events via `compliance_sync` (14 §2.7).
- **Privacy incident runbook** exists before launch: a privacy incident is a PRD stop/rollback signal.

## 9. Network and edge security

Per 13 §6: ALB public; API, DB, workers in private subnets; no public DB endpoint; S3 not public (signed URLs, 5 min); SSH only via bastion with key + MFA.

Additions `[PROPOSED]`:
- AWS WAF on the ALB with rate-based rules in front of the app-level limits (120 req/min per user, co-worker 20/min — 16 §7).
- `/webhooks/whatsapp` is the only unauthenticated route: signature verification mandatory (TC-SEC-08), duplicates and out-of-order status events tolerated.
- Big screen route (`/bigscreen`) is reachable only with a **device token** scoped to `/bigscreen/*`, optionally restricted to an allow-listed CIDR (TC-SEC-02). Tokens are managed in `admin_devices` and are revocable.

## 10. Big-screen and CM-device deployment

- The wall display runs a managed browser in kiosk mode on a device-pinned URL with a long-lived device token (13 §1.5). Operating system time sync is required; stale timestamps rely on correct clocks.
- Resilience is built into the page (16 §8): WebSocket reconnect, 60 s polling fallback to `/bigscreen/summary`, 3-minute watchdog reload with "data delayed since HH:MM" in the meantime. Soak test: 24 h with memory growth ≤ 20 % (TC-NFR-09).
- CM device: separate simplified login/PIN, 30-minute inactivity timeout (13 §1.6). Presentation-safe mode toggles reach all displays ≤ 2 s (TC-US04).
- The client supplies the official TVK elephant artwork `[GAP: PRD "Identity"]`. It is stored in object storage and referenced by URL in client config; **no client artwork is committed to the repo** (15 §1 `public/`).

## 11. Runbooks (`infra/runbooks/`)

Written before launch; each has trigger, diagnosis, action, verification, owner.

| Runbook | Trigger |
|---|---|
| `credentials-rotation` | Scheduled rotation or suspected leak (14 §2.8 references this) |
| `x-api-tier-change` | G-01 decision: switch `X_ADAPTER` polling ↔ stream ↔ vendor (configuration change, not code) |
| `x-cap-approaching` | 80 % / 95 % usage alerts; sampling mode behaviour; labelling of sampled numbers |
| `outlet-blocked-or-layout-changed` | Crawl `down` with reason `blocked` or selector break (never evade blocks) |
| `whatsapp-template-approval` | G-14: submit/track templates (English and Tamil variants) |
| `whatsapp-outage` | Meta outage; in-app and big screen still fire |
| `llm-outage-or-cost-spike` | Provider failure; rules-only influence fallback is active; co-worker says "unavailable" |
| `queue-backlog` | Queue lag alerts; scale workers; check backfill isolation |
| `db-failover-and-restore` | Failover event or corruption |
| `release-rollback` | §5.3 |
| `privacy-incident` | Any suspected exposure; PRD stop signal |
| `calibration-threshold-change` | Weekly tuning with admins; how to record the change and rationale |
| `kiosk-recovery` | Wall display frozen/blank |

## 12. Capacity and cost

Design targets `[PROPOSED]` (18 §7): 100 k X posts/day (cap-sized), 5 k news items/day, 10× bursts for 30 min, dashboard summary p95 < 400 ms at 10 M mentions, 200 concurrent WebSocket clients. Daily volume is **unknown until measured in the first two weeks** `[PRD]`, so sizing is revisited after that window.

Cost levers already in the design (PRD cost rule): cheap classification for every item, strong model only for escalations/explanations/briefs/Q&A, dedupe before processing, sampling mode under X cap pressure, engagement refresh only for high-influence candidates. Cost telemetry is a trend alert, not a gate (PRD: no cap).

## 13. Launch and cutover

The deployment-side launch checklist (infra, secrets, runbooks, backups, retention set, WhatsApp templates approved, device tokens issued) is part of 24 §2 (release gate). Production go-live also requires the client inputs listed in 24 §7.
