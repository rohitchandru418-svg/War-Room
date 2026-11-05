# 20 — Environment and Setup

**Audience:** OpenHands and human engineers setting up a working local environment. **Reads with:** 15 (structure), 16 (guidelines), 18 (testing), 19 (deployment).

Everything here runs **without any real external account**. X, WhatsApp, the LLM provider, and news sites are replaced by mock servers (18 §2). Real credentials are only needed in staging/production (19).

> Version numbers below are **minimums or proposals** (`[VERIFY]`). The lockfiles (`uv.lock`, `package-lock.json`) are the authoritative pins (16 §11). Update this document in the same PR that changes a major version.

---

## 1. Prerequisites

| Tool | Minimum | Notes |
|---|---|---|
| Docker + Compose v2 | Docker 24+ | Runs the whole stack |
| Python | 3.11+ | Backend (10) |
| `uv` | current | Dependency management and lockfile |
| Node.js | 20 LTS `[VERIFY]` | Frontend (Next.js 14+) |
| `make` | any | Wraps all common commands |
| Tesseract with `tam` and `eng` data, poppler | system packages | Only needed to run OCR tests **outside** Docker; the worker image includes them |
| Git | any | Branch-per-task workflow (15 §5) |

Hardware: 8 GB RAM free is enough for the stack; the embedding model (sentence-transformers, CPU) is the largest consumer.

## 2. Quick start

```bash
git clone <repo-url> warroom && cd warroom
cp .env.example .env            # defaults work for local
make dev                        # builds and starts the full stack with mocks
make migrate                    # applies migrations 0001..0011
make seed CLIENT=demo           # "Acme Beverages" synthetic client (proves generalisation)
make seed CLIENT=tvk            # TVK placeholder structure (real lists replace PLACEHOLDER rows)
```

Then open:

| URL | What |
|---|---|
| `http://localhost:3000` | Frontend (team mode, admin) |
| `http://localhost:3000/bigscreen` | Big screen (needs a device token — see §6) |
| `http://localhost:8000/docs` | FastAPI OpenAPI / Swagger UI |
| `http://localhost:8025` | Mailhog (password-reset emails) |
| `http://localhost:9001` | MinIO console (object storage) |

Smoke check that proves the slice works end-to-end:

```bash
make replay SCENARIO=one_post_mattered   # must-pass test 2 on the real pipeline with mocks
```

Expected: a Tier-1 alert is created, a `alert.new` WebSocket event is emitted, the WhatsApp mock records an accepted message, and the printed latency is ≤ 300 s (fake clock) — see 18 §5.

## 3. Make targets (15 §1)

| Target | Does |
|---|---|
| `make dev` | `docker compose up --build` full stack |
| `make down` | Stop and remove containers (keeps volumes) |
| `make reset` | Stop and **delete volumes** (clean DB/Redis/object store) |
| `make migrate` | Run Alembic upgrade head with the migrator role |
| `make seed CLIENT=<name>` | `scripts/seed_client.py` loads `backend/seeds/<name>/` (data only, no code change per client) |
| `make test` | Unit + integration (testcontainers), same as CI `ci.yml` core |
| `make test-unit` | Fast unit tests only (< 2 min target, 18 §1) |
| `make lint` | ruff, mypy (strict on `domain` and `coworker`), eslint, prettier, import-linter, `check_hardcoding.py` |
| `make e2e` | Playwright against the compose stack with mocks |
| `make replay SCENARIO=<name>` | `scripts/replay_scenario.py` (add `REAL_LLM=1` for the real LLM route, `REAL_WA_SANDBOX=1` for sandbox WhatsApp) |
| `make evals` | Co-worker and label evals (`backend/evals/`), needs real LLM keys |
| `make openapi-client` | Regenerate the frontend API client from OpenAPI (`scripts/gen_openapi_client.sh`); CI fails on drift |

## 4. Environment variables

Only `backend/app/settings.py` reads the environment (15 §4 rule 4). `.env.example` documents every variable. Frontend variables are prefixed `NEXT_PUBLIC_`.

### 4.1 Core

| Variable | Local default | Purpose |
|---|---|---|
| `APP_ENV` | `local` | `local`/`ci`/`staging`/`production` |
| `LOG_LEVEL` | `INFO` | structlog level |
| `DATABASE_URL` | `postgresql+psycopg://warroom_app:…@postgres:5432/warroom` | **App role** (no superuser, no BYPASSRLS) |
| `DATABASE_URL_MIGRATOR` | `…warroom_migrator…` | Migration job only |
| `REDIS_URL` | `redis://redis:6379/0` | Celery broker/result, pub/sub, rate limits, seen-ID set |
| `JWT_SECRET` | dev-only random | Signing key; rotate per 19 §4 |
| `JWT_ACCESS_TTL_SECONDS` | `900` | 15 min (13 §1.1) |
| `REFRESH_TTL_DAYS` | `7` | Refresh token lifetime |
| `CM_SESSION_TIMEOUT_MINUTES` | `30` | CM View inactivity timeout (13 §1.6) |
| `FRONTEND_URL`, `ALLOWED_HOSTS`, `CORS_ORIGINS` | localhost values | Web/CORS |
| `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_WS_URL` | `http://localhost:8000`, `ws://localhost:8000/ws` | Frontend endpoints |

### 4.2 Storage and email

| Variable | Local default | Purpose |
|---|---|---|
| `S3_ENDPOINT_URL` | `http://minio:9000` | S3-compatible store (MinIO locally) |
| `S3_BUCKET_RAW`, `S3_BUCKET_UPLOADS`, `S3_BUCKET_PAGES` | `wr-raw`, `wr-uploads`, `wr-pages` | Buckets (14 §10) |
| `S3_SIGNED_URL_TTL_SECONDS` | `300` | Signed URL lifetime (13 §6.2) |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM` | `mailhog`, `1025` | Password-reset email only |

### 4.3 Integrations (14)

| Variable | Local default | Purpose |
|---|---|---|
| `X_ADAPTER` | `polling` | `polling` / `stream` / `vendor` (14 §2.1). Switching is configuration, not code |
| `X_API_BASE_URL` | `http://mock-x:8081` | Points at the mock locally |
| `X_BEARER_TOKEN` | `mock` | **One credential set per client** (BR-054); config schema rejects a list (test `test_single_credential_set`) |
| `X_MONTHLY_READ_CAP` | `3000000` | Hard budget guard; drives 50/80/95 % alerts and sampling mode |
| `X_BACKFILL_DAILY_SHARE` | `0.3` | Share of daily read budget backfill may use |
| `CRAWLER_CONTACT_URL` | `http://localhost/contact` | Goes in the crawler User-Agent. **Never a personal email** (14 §3.3) |
| `SEARCH_PROVIDER` | *(unset = disabled)* | `google_cse` or `serpapi` |
| `SEARCH_API_KEY` | — | Key for the chosen provider |
| `YOUTUBE_API_KEY` | `mock` | YouTube Data API v3 |
| `WHATSAPP_API_BASE_URL` | `http://mock-wa:8082` | Mock locally |
| `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_APP_SECRET`, `WHATSAPP_VERIFY_TOKEN` | `mock` | Meta Cloud API credentials. Readable only by the WhatsApp sender (BR-043) |
| `WHATSAPP_MAX_RETRIES`, `WHATSAPP_RETRY_BACKOFF_SECONDS` | `5`, `30` | Retry policy (14 §7.2; 08 §3.3 "defined in deployment config") |
| `LLM_API_BASE_URL_OVERRIDE` | `http://mock-llm:8083` | Local/CI mock; unset for real providers |
| `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY` | — | Only for staging/prod and `evals` |
| `OCR_ENGINE` | `tesseract` | `tesseract` or `cloud` (decided by the T-OCR-02 bake-off) |
| `EMBEDDING_DEVICE` | `cpu` | Embedding worker device |

Model names and prompt versions are **not** environment variables: they are rows in `config.model_routes` (no model-name literal in application code — CI `grep` check, 14 §8).

### 4.4 Pipeline and worker tuning

`CELERY_CONCURRENCY_<QUEUE>` (defaults in 15 §2), `CELERY_TASK_TIME_LIMIT`, `CELERY_MAX_RETRIES`. Per-client business thresholds (spike z-score, caps, influence trigger, etc.) are **database rows** (`config.alert_thresholds`), never environment variables (DG-10).

## 5. Docker Compose services

`docker-compose.yml` (local) and `docker-compose.test.yml` (CI/E2E):

| Service | Image / source | Port |
|---|---|---|
| `postgres` | PostgreSQL 16 with pgvector | 5432 |
| `pgbouncer` | transaction pooling (matches production RLS behaviour) | 6432 |
| `redis` | Redis with AOF on | 6379 |
| `minio` | S3-compatible storage; buckets created at startup | 9000 / 9001 |
| `mailhog` | SMTP capture | 1025 / 8025 |
| `api` | `backend.Dockerfile` | 8000 |
| `worker-*`, `beat` | `worker.Dockerfile` (same queue split as 19 §2.1, smaller concurrency) | — |
| `frontend` | `frontend.Dockerfile` (dev mode with hot reload) | 3000 |
| `mock-x` | `backend/tests/mocks/x_server.py` — rate-limit/429/401/5xx injection, "reads" accounting | 8081 |
| `mock-wa` | `backend/tests/mocks/wa_server.py` — accepts templates, emits status webhooks with configurable delay/failure | 8082 |
| `mock-llm` | `backend/tests/mocks/llm.py` — deterministic canned JSON keyed by content hash | 8083 |
| `mock-news` | `backend/tests/mocks/news_site/` — feeds, sitemaps, HTML, robots.txt, slow/blocked/JS-only variants | 8084 |
| `worker-playwright` *(profile `crawl-js`)* | `playwright-worker.Dockerfile` | — |

All time in tests goes through `common/clock.py` (`FakeClock`); never call `datetime.now()` elsewhere (16 §2).

## 6. First-run data and logins

- `make seed CLIENT=demo` and `CLIENT=tvk` create the client row, entities, watch terms, tiers, outlets, thresholds, weights, recipients, model routes (files in `backend/seeds/<client>/`). TVK seeds contain **PLACEHOLDER** entries clearly marked until the war room supplies real lists (G-07…G-10, PRD "Information needed from the war room").
- Seeding also creates one user per role (`system_admin`, `war_room_admin`, `war_room_lead`, `team_member`, `cm_view`). **Dev passwords are generated at seed time and printed once**; none is committed.
- Big-screen device token: `make device-token CLIENT=demo` prints a token scoped to `/bigscreen/*`; open `/bigscreen?device_token=…` once (the page stores it).
- WhatsApp recipients in seeds point at the mock; production recipients are `[GAP]` (PRD-GAP-ANALYSIS 1.3).

## 7. Running tests

| Goal | Command |
|---|---|
| Everything CI core runs | `make test` |
| Backend unit only | `make test-unit` |
| One module | `cd backend && uv run pytest tests/<area> -k <expr>` |
| Frontend | `cd frontend && npm test` |
| E2E smoke / full | `make e2e` (smoke subset in PRs; full nightly) |
| Must-pass 1 / 2 | `make replay SCENARIO=councillor_incident` / `one_post_mattered` |
| Other replays | `surrogate_wave`, `ally_discontent`, `print_only_story` |
| Evals | `make evals` (real LLM keys) |

Coverage gates: `domain` ≥ 90 %, `services` ≥ 80 %, overall ≥ 80 % (16 §12).

## 8. Working on the frontend

- Server data through React Query only; UI state through Zustand; WebSocket events update the React Query cache through one adapter (16 §2).
- Statistics must use the shared number components (`components/numbers/`); the `warroom/no-bare-percent` ESLint rule flags raw `%` formatting (16 §8).
- Tamil: Noto Sans Tamil via `next/font`, `lang="ta"` on Tamil strings, no letter-spacing on Tamil text; `en.json` and `ta.json` must have identical keys (CI check).
- Regenerate the typed API client after backend schema changes: `make openapi-client`.

## 9. Common problems

| Symptom | Cause / fix |
|---|---|
| Migration fails on FK order | Run in the order in 15 §3 (campaigns before stories). Do not reorder files |
| Queries return nothing / permission denied | RLS: the session must `SET LOCAL app.client_id`; use repositories, not raw sessions |
| No alerts in a replay | Backfill-flagged items never alert; check `ingest_mode` and `live_alert_window_minutes` (14 §2.4) |
| Dashboard shows "not enough data" | Fewer than `min_sample_size` (30) mentions in window — expected |
| WhatsApp send fails locally | Check `mock-wa` is up and `WHATSAPP_API_BASE_URL` points at it |
| Worker import error mentioning `whatsapp` from `coworker` | import-linter contract is doing its job (15 §4 rule 2); remove the import |
| Embedding worker slow | CPU model; reduce batch size (32) or enable GPU profile |
| Tamil text renders as boxes | Missing Noto Sans Tamil in the page/screenshot environment |

## 10. Verifying the environment

Tick before starting any task (21 §6):

- [ ] `make dev`, `make migrate`, `make seed CLIENT=demo` succeed on a clean checkout
- [ ] `make lint` and `make test-unit` pass on `main`
- [ ] `make replay SCENARIO=one_post_mattered` passes
- [ ] Login works for each seeded role; permission-matrix test passes
- [ ] Frontend builds and `en.json`/`ta.json` parity check passes
