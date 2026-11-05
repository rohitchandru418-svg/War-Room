# 10 — Technology Stack

**Note:** Items marked [PROPOSED] are technical recommendations not explicitly stated in the PRD. Items marked [CTO DECISION] are explicitly left to the CTO by the PRD. All proposed choices are justified below.

---

## 1. Frontend

| Technology | Choice | Justification |
|-----------|--------|--------------|
| Framework | **Next.js 14+ (React)** [PROPOSED] | SSR for initial load (important for big screen), then client-side navigation. Large ecosystem, good i18n support for Tamil. |
| Language | **TypeScript** [PROPOSED] | Type safety reduces bugs in complex state management |
| State management | **Zustand + React Query** [PROPOSED] | Zustand for local/global UI state; React Query for server state, caching, and background refresh |
| Real-time | **WebSocket (native browser API)** [PROPOSED] | Receives push updates for alerts, sentiment changes, source health |
| UI components | **Tailwind CSS + Radix UI** [PROPOSED] | Utility-first CSS for rapid development; Radix for accessible headless components |
| Charts | **Recharts or Chart.js** [PROPOSED] | Sentiment sparklines, campaign timing charts, volume spike charts |
| Internationalisation | **next-i18next** [PROPOSED] | English and Tamil UI strings; right-to-left not needed (Tamil is left-to-right) |
| Tamil font | **Noto Sans Tamil** (Google Fonts) | Full Unicode Tamil support, free, well-tested |

---

## 2. Backend API

| Technology | Choice | Justification |
|-----------|--------|--------------|
| Framework | **FastAPI (Python)** [PROPOSED] | Python is natural for data/ML-adjacent backend; FastAPI is async, fast, and has automatic OpenAPI docs. Integrates well with pipeline workers written in Python. |
| Language | **Python 3.11+** [PROPOSED] | Consistency with ML/NLP libraries |
| WebSocket server | **FastAPI WebSocket** [PROPOSED] | Integrated with the same FastAPI app; or separate service if scale requires |
| REST API | **FastAPI + Pydantic** [PROPOSED] | Pydantic models for validation and serialisation |
| Task queue | **Celery + Redis** [PROPOSED] [CTO DECISION: could be RabbitMQ, SQS] | Background task processing for pipeline workers |
| Auth | **JWT (python-jose)** [PROPOSED] | Short-lived access tokens + refresh tokens |
| ORM | **SQLAlchemy 2.0 + Alembic** [PROPOSED] | Mature ORM with async support; Alembic for migrations |

---

## 3. Data Pipeline (Workers)

| Technology | Choice | Justification |
|-----------|--------|--------------|
| Worker framework | **Celery workers** [PROPOSED] | Scales horizontally; retries; dead-letter queues |
| Message broker | **Redis Streams** [PROPOSED] [CTO DECISION] | Durable, ordered, consumer groups; already used for WebSocket pub/sub |
| NLP — filtering | **FastText or sentence-transformers** [PROPOSED] | Fast, cheap embedding for watch-term and topic-similarity matching |
| NLP — labelling (cheap) | **GPT-4o-mini or Gemini Flash** [PROPOSED] [CTO DECISION] | Low cost per item for classification; Tamil support available |
| NLP — labelling (strong) | **GPT-4o or Claude 3.5 Sonnet** [PROPOSED] [CTO DECISION] | Used for low-confidence items, reasoning, summaries, Q&A |
| Tamil language detection | **Custom langdetect model** [PROPOSED] [CTO DECISION] | X language tag not reliable for Tanglish; needs custom model trained on Tamil/Tanglish samples |
| Tamil OCR | **Tesseract with Tamil traineddata + post-processing** [PROPOSED] [CTO DECISION] | Open-source; Tamil support exists; may need fine-tuning on Tamil newsprint. Alternative: Google Cloud Vision or AWS Textract. |
| Story merging | **Sentence-transformers (multilingual)** [PROPOSED] [CTO DECISION] | Semantic similarity for grouping cross-source stories |
| Campaign detection | **Scikit-learn clustering + custom rule engine** [PROPOSED] | DBSCAN for account clustering; rule engine for timing and text similarity patterns |
| News crawling | **Scrapy + newspaper3k / trafilatura** [PROPOSED] | Scrapy for site-specific crawling; trafilatura for generic article extraction |
| Feed parsing | **feedparser** [PROPOSED] | Mature RSS/Atom/sitemap parser |
| YouTube data | **YouTube Data API v3** | Official API for channel feeds and video metadata |

---

## 4. Database

| Technology | Choice | Justification |
|-----------|--------|--------------|
| Primary DB | **PostgreSQL 16** [PROPOSED] | Strong JSON support, row-level security, ACID, pgvector extension, proven at scale |
| Vector search | **pgvector extension** [PROPOSED] [CTO DECISION: could be Elasticsearch] | Semantic search for co-worker retrieval; keeps stack simple (no extra service) |
| Full-text search | **PostgreSQL FTS (tsvector)** [PROPOSED] | Tamil FTS may need custom dictionary; evaluate during build |
| Connection pooling | **PgBouncer** [PROPOSED] | Needed for high connection count from workers |
| Cache / pub-sub | **Redis** [PROPOSED] | WebSocket broadcast pub/sub; session cache; task queue |
| Object storage | **AWS S3 or equivalent** [PROPOSED] | E-paper page images, clipping scans |

---

## 5. AI Co-worker

| Component | Choice | Justification |
|-----------|--------|--------------|
| LLM API | **Anthropic Claude or OpenAI GPT-4o** [CTO DECISION] | Both support Tamil; grounding and citation requires careful prompting |
| RAG framework | **LangChain or LlamaIndex** [PROPOSED] | Standard retrieval-augmented generation; connects LLM to Mention database |
| Memory | **Custom PostgreSQL tables** [PROPOSED] [CTO DECISION] | Correctable, isolated per client, queryable. Alternative: vector store + structured memory |
| Eval harness | **Custom eval script** [PROPOSED] | Fixed set of real war-room questions with checked answers; run on every model/prompt change |
| Prompt management | **LangSmith or Promptflow** [PROPOSED] | Version prompts; track experiments |

---

## 6. Integrations

| Service | Technology | Notes |
|---------|-----------|-------|
| X API | **X API v2** (Basic or Enterprise) | [CTO DECISION: tier]; OAuth 2.0 app-only auth |
| WhatsApp | **WhatsApp Business Cloud API (Meta)** | Official API; requires Business Account verification |
| News feeds | **RSS, Atom, news sitemaps** | Standard formats; fallback to crawl |
| YouTube | **YouTube Data API v3** | Channel feed + video search |
| E-paper | **Per-publisher** | Web scraping of subscriber portals or file download (terms to verify) |
| Search discovery | **Google Custom Search API or SerpAPI** [PROPOSED] | Finding articles on sites not yet in outlet register |

---

## 7. Infrastructure

| Component | Choice | Notes |
|-----------|--------|-------|
| Cloud | **AWS (ap-south-1 / Mumbai)** [PROPOSED] | India data residency for DPDP; also GCP or Azure have Mumbai regions |
| Containers | **Docker + ECS (Fargate)** [PROPOSED] | Or Kubernetes (EKS) if scale demands it |
| CI/CD | **GitHub Actions** [PROPOSED] | Automated test, build, deploy pipeline |
| Managed PostgreSQL | **AWS RDS PostgreSQL** [PROPOSED] | Multi-AZ for HA |
| Managed Redis | **AWS ElastiCache Redis** [PROPOSED] | HA cluster mode |
| Object storage | **AWS S3** [PROPOSED] | Scans, e-paper images |
| CDN | **CloudFront** [PROPOSED] | Static assets for frontend |
| Monitoring | **Datadog or CloudWatch + Grafana** [PROPOSED] | Metrics, logs, alerts |
| Secrets | **AWS Secrets Manager** [PROPOSED] | API keys, DB passwords, WhatsApp credentials |

---

## 8. Development Tools

| Tool | Purpose |
|------|---------|
| Git + GitHub | Version control |
| Docker Compose | Local development environment |
| pytest (Python) | Backend unit and integration tests |
| Jest + React Testing Library | Frontend tests |
| Playwright | End-to-end tests |
| Alembic | Database migrations |
| pre-commit hooks | Linting, formatting (ruff for Python, eslint/prettier for TS) |
| OpenAPI / Swagger UI | Auto-generated API docs via FastAPI |

---

## 9. Key Decisions Left to CTO

| Decision | Options | Impact |
|---------|---------|--------|
| X API tier | Pay-as-you-go vs. Enterprise vs. Firehose vendor | Coverage, cost, contract time |
| X data fetch method | Polling vs. streaming | Latency, reliability |
| Message queue | Redis Streams vs. RabbitMQ vs. SQS | Operations, durability |
| Labelling models | One multilingual model vs. per-language models | Tanglish accuracy, cost |
| Tamil OCR | Tesseract vs. cloud OCR (Google/AWS) | Accuracy on newsprint, cost |
| Story merge algorithm | Semantic similarity thresholds | What "same story" means |
| Search | pgvector vs. Elasticsearch | Stack complexity, search quality |
| Co-worker LLM | Claude vs. GPT-4o vs. Gemini | Tamil quality, cost, latency |
| Co-worker memory | Structured DB vs. vector store | Correctability, retrieval quality |
| Infrastructure | AWS vs. GCP vs. Azure | Cost, compliance, team familiarity |
| Multi-tenancy model | Row-level security vs. schema-per-client | Isolation strength, ops complexity |
