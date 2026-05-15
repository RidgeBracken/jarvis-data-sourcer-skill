# Section G: MVP Build Plan

> **Project**: Jarvis HQ - Personal AI News Scraping System
> **Scope**: Phased implementation plan for a personal, single-user news aggregation and distribution platform
> **Deployment**: Local Docker, personal use only
> **Total Estimated Timeline**: 8 weeks (evenings/weekends, ~10-15 hrs/week)


## Table of Contents

- 1. Technology Stack (Final Decisions)
  - Stack Compatibility Notes
- 2. Directory Structure
- 3. Phased Implementation
  - Phase 1: Static Source Registry + RSS Ingestion (Week 1)
  - Phase 2: HTML Article Extraction (Week 2)
  - Phase 3: Source Scoring + Deduplication (Week 3)
  - Phase 4: Summarisation + Categorisation (Week 4)
  - Phase 5: Website Feed Integration (Week 5)
  - Phase 6: Telegram + n8n Automation (Week 6)
  - Phase 7: Advanced Source Discovery (Week 7-8)
- 4. Per-Phase Detailed Tasks
  - Phase 1: Static Source Registry + RSS Ingestion — Day-by-Day Breakdown
  - Phase 2-7 High-Level Task Lists
- 5. Dependencies Between Phases
  - Dependency Graph
  - Detailed Dependency Matrix
  - Parallelisation Opportunities
  - Minimal Viable Integration Points
- 6. Technology Decisions Log
  - Core Technology Decisions
  - Decisions Requiring Re-evaluation
  - Cost Projections (Personal Use)
- Appendix A: Environment Variables Reference
- Appendix B: Quick Start Commands
- Appendix C: Test Commands
- Appendix D: npm Scripts Reference

---


---

## 1. Technology Stack (Final Decisions)

| Layer | Technology | Version | Justification |
|-------|-----------|---------|---------------|
| **Language** | Node.js | 20 LTS | Rich scraping ecosystem, native async/await, alignment with n8n (Node-based automation), single-language full stack |
| **Framework** | Fastify | 5.x | 3x faster than Express, native TypeScript support, excellent plugin system, built-in schema validation, low overhead |
| **Database** | PostgreSQL | 16 | Full-text search (`tsvector`/`tsquery`), JSONB for flexible metadata, rock-solid reliability, excellent Docker support |
| **Queue** | BullMQ | 5.x + Redis 7 | Native TypeScript API, job priorities, rate limiting, dead-letter queues, delayed jobs, excellent observability |
| **RSS Parsing** | feedparser + rss-parser | latest | `feedparser` handles most edge cases (Atom/RSS variants); `rss-parser` as fallback for malformed feeds; dual-parser = maximum compatibility |
| **HTML Extraction** | cheerio + @mozilla/readability + trafilatura-js | latest | **Tiered approach**: (1) `readability` for article body, (2) `cheerio` for custom selectors/author extraction, (3) `trafilatura` as deep-extraction fallback |
| **Browser** | puppeteer | 23.x | Headless Chromium for JavaScript-heavy sites, page automation, screenshot capture for validation, proven at scale |
| **HTTP Client** | axios | 1.7.x | Configurable interceptors, request/response transformation, automatic retries, browser-like headers, timeout handling |
| **Validation** | zod | 3.23.x | TypeScript-first schema validation, runtime type checking, excellent error messages, composable schemas |
| **ORM/Query** | knex + objection | 3.x | SQL-first query builder (knex) + lightweight ORM (objection); explicit queries, migrations, TypeScript model definitions |
| **Scheduler** | node-cron + BullMQ | latest | `node-cron` for cron expressions; BullMQ for actual job queuing and execution; separation of scheduling from execution |
| **Testing** | vitest | 2.x | Native ESM support, TypeScript out of the box, 10x faster than Jest, built-in mocking, coverage reporting |
| **API Docs** | @fastify/swagger | 9.x | Auto-generated OpenAPI specs from Zod schemas + Fastify route definitions, interactive Swagger UI |
| **Container** | Docker + Docker Compose | latest | Service isolation, reproducible environments, one-command deployment, volume persistence for PostgreSQL |
| **AI/LLM** | OpenAI API | gpt-4o-mini | Cost-effective summarisation, categorisation, quality scoring; local inference fallback (future) |
| **Bot** | node-telegram-bot-api | latest | Official Telegram Bot API wrapper, webhook + polling support, Markdown formatting |
| **Config** | yaml + dotenv | latest | YAML for human-readable source registries; `.env` for secrets and environment variables |
| **Logging** | pino | 9.x | Fast JSON logging, structured output, child loggers, pretty-print in dev, file rotation |
| **Monitoring** | bull-board | 5.x | Web UI for BullMQ queue inspection, job status, retries, DLQ management |

### Stack Compatibility Notes

```
Node.js 20 LTS
├── Fastify 5.x (plugin system)
│   ├── @fastify/swagger (API docs)
│   └── @fastify/cors / @fastify/helmet (security)
├── BullMQ 5.x ──── Redis 7 (job queues)
├── Knex + Objection ──── PostgreSQL 16 (data layer)
├── Puppeteer ──── Chromium (browser automation)
├── Axios (HTTP client)
├── Zod (validation)
├── Pino (logging)
└── Vitest (testing)
```

---

## 2. Directory Structure

```
jarvis-scraper/
├── docker-compose.yml              # Multi-service orchestration (app, db, redis, puppeteer)
├── Dockerfile                      # Multi-stage build: deps → build → production
├── .env.example                    # Template for environment variables
├── .env                            # Local secrets (gitignored)
├── package.json                    # Dependencies, scripts, engines
├── package-lock.json               # Locked dependency tree
├── tsconfig.json                   # TypeScript strict configuration
├── vitest.config.ts                # Test configuration + coverage settings
├── knexfile.ts                     # Database migration configuration
├── .dockerignore                   # Exclude from Docker context
├── .gitignore                      # Exclude from version control
├── .prettierrc                     # Code formatting
├── eslint.config.mjs               # Linting rules
│
├── src/
│   ├── index.ts                    # Entry point: bootstraps Fastify + workers + scheduler
│   ├── app.ts                      # Fastify app factory: plugins, routes, error handling
│   │
│   ├── config/                     # Configuration layer
│   │   ├── index.ts                # Config loader: merges .env + YAML files
│   │   ├── sources/                # YAML source registries, one per topic
│   │   │   ├── ai-llms.yaml        # LLM news sources (OpenAI, Anthropic, etc.)
│   │   │   ├── ai-robotics.yaml    # Robotics sources
│   │   │   ├── ai-research.yaml    # AI research (arXiv, Papers With Code)
│   │   │   ├── tech-startups.yaml  # Startup/VC sources
│   │   │   ├── dev-tools.yaml      # Developer tools (GitHub releases, HN)
│   │   │   └── cybersecurity.yaml  # Security news sources
│   │   ├── topics.yaml             # Topic definitions: keywords, relevance rules
│   │   ├── rate-limits.yaml        # Per-domain rate limiting configuration
│   │   └── quality-rules.yaml      # Content quality scoring weights and thresholds
│   │
│   ├── adapters/                   # Source-specific scraping adapters
│   │   ├── base.adapter.ts         # Abstract base class: interface + shared logic
│   │   ├── rss.adapter.ts          # RSS/Atom feed parser (feedparser + rss-parser)
│   │   ├── api.adapter.ts          # REST API adapters (GitHub API, etc.)
│   │   ├── html.adapter.ts         # Static HTML scraping (cheerio + readability)
│   │   ├── browser.adapter.ts      # JS-rendered sites (puppeteer fallback)
│   │   ├── github.adapter.ts       # GitHub releases, commits, issues
│   │   ├── youtube.adapter.ts      # YouTube channel/video metadata
│   │   ├── sitemap.adapter.ts      # XML sitemap discovery and parsing
│   │   └── index.ts                # Adapter registry + factory function
│   │
│   ├── core/                       # Core data pipeline (extract → score → rank → deliver)
│   │   ├── scheduler.ts            # Cron-based job scheduling (node-cron + BullMQ)
│   │   ├── queue.ts                # BullMQ queue/worker initialisation + DLQ config
│   │   ├── collector.ts            # Orchestrates adapter calls, collects raw items
│   │   ├── extractor.ts            # HTML content extraction (readability → cheerio → trafilatura)
│   │   ├── quality-engine.ts       # Multi-signal quality scoring algorithm
│   │   ├── deduplicator.ts         # 5-stage deduplication: URL → hash → content → semantic → manual
│   │   ├── summariser.ts           # Extractive + abstractive summarisation
│   │   ├── categoriser.ts          # Topic matching, keyword classification, tag assignment
│   │   └── distributor.ts          # Output routing: API feed, Telegram, digest, webhook
│   │
│   ├── models/                     # Objection.js database models
│   │   ├── base.model.ts           # Base model: timestamps, soft-delete, JSON helpers
│   │   ├── source.model.ts         # News source: URL, adapter type, last fetch, health status
│   │   ├── feed-item.model.ts      # Scraped article: title, content, author, url, scores, metadata
│   │   ├── scrape-job.model.ts     # BullMQ job persistence: status, retries, logs
│   │   ├── topic.model.ts          # Topic definitions and keyword associations
│   │   ├── quality-score.model.ts  # Quality score history per article
│   │   ├── duplicate-log.model.ts  # Deduplication audit trail
│   │   └── digest.model.ts         # Daily digest records: items, delivery status
│   │
│   ├── services/                   # Business logic services
│   │   ├── source-registry.ts      # YAML source loading, validation, registration
│   │   ├── source-discovery.ts     # Automatic source discovery for new topics
│   │   ├── source-health.ts        # Health monitoring: success rates, response times
│   │   ├── feed-service.ts         # Feed retrieval: pagination, filtering, sorting
│   │   ├── search-service.ts       # Full-text search across articles
│   │   ├── digest-service.ts       # Daily digest compilation and formatting
│   │   ├── breaking-news.ts        # Breaking news detection algorithm
│   │   ├── rate-limiter.ts         # Per-domain rate limiting with Redis backing
│   │   └── health-monitor.ts       # System health: queue depths, DB size, error rates
│   │
│   ├── api/                        # Fastify REST API layer
│   │   ├── routes/
│   │   │   ├── sources.ts          # GET/POST/PUT/DELETE /sources (CRUD)
│   │   │   ├── feed.ts             # GET /feed, GET /feed/:topic, GET /feed/search
│   │   │   ├── topics.ts           # GET /topics, GET /topics/:id/articles
│   │   │   ├── digest.ts           # GET /digest/today, POST /digest/generate
│   │   │   ├── health.ts           # GET /health, GET /health/queues, GET /health/sources
│   │   │   └── admin.ts            # GET /admin/stats, POST /admin/jobs/trigger
│   │   ├── schemas.ts              # Zod schemas: request validation + response types
│   │   ├── errors.ts               # Centralised error handling + HTTP status mapping
│   │   └── hooks.ts                # PreHandler hooks: auth, logging, rate limiting
│   │
│   ├── utils/                      # Shared utilities
│   │   ├── logger.ts               # Pino logger factory (structured JSON logging)
│   │   ├── rate-limiter.ts         # Token-bucket rate limiter (Redis-backed)
│   │   ├── retry-handler.ts        # Exponential backoff + circuit breaker pattern
│   │   ├── url-validator.ts        # URL normalization, domain extraction, safety checks
│   │   ├── date-normalizer.ts      # Date format parsing: RSS dates, meta tags, relative dates
│   │   ├── author-extractor.ts     # Author name extraction from meta tags, bylines
│   │   ├── html-cleaner.ts         # HTML sanitisation: remove ads, nav, scripts
│   │   ├── text-stats.ts           # Word count, reading time, sentence extraction
│   │   ├── hash-utils.ts           # Content hashing for deduplication (simhash, minhash)
│   │   └── telegram-formatter.ts   # MarkdownV2 formatting for Telegram messages
│   │
│   ├── types/                      # TypeScript type definitions
│   │   ├── index.ts                # Shared types: FeedItem, SourceConfig, QualityScore, etc.
│   │   ├── adapters.ts             # Adapter interfaces + union types
│   │   ├── api.ts                  # API request/response types
│   │   └── queue.ts                # BullMQ job data types
│   │
│   ├── jobs/                       # BullMQ worker definitions
│   │   ├── scraper.worker.ts       # Main scraping worker: adapter dispatch
│   │   ├── extractor.worker.ts     # Content extraction worker: HTML → clean text
│   │   ├── scorer.worker.ts        # Quality scoring worker: multi-signal scoring
│   │   ├── digest.worker.ts        # Digest compilation worker
│   │   └── notifier.worker.ts      # Telegram/webhook notification worker
│   │
│   └── tests/                      # Test suite
│       ├── fixtures/               # Test data: sample RSS feeds, HTML pages, API responses
│       │   ├── rss/
│       │   ├── html/
│       │   └── api/
│       ├── unit/                   # Unit tests (individual functions, 1:1 with src)
│       │   ├── adapters/
│       │   ├── core/
│       │   ├── utils/
│       │   └── services/
│       ├── integration/            # Integration tests (database + API interactions)
│       │   ├── api.test.ts
│       │   ├── database.test.ts
│       │   └── queue.test.ts
│       └── e2e/                    # End-to-end tests (full pipeline)
│           └── full-pipeline.test.ts
│
├── migrations/                     # Knex database migrations (timestamped)
│   ├── 20250101000001_create_sources.ts
│   ├── 20250101000002_create_feed_items.ts
│   ├── 20250101000003_create_scrape_jobs.ts
│   ├── 20250101000004_create_topics.ts
│   ├── 20250101000005_create_quality_scores.ts
│   ├── 20250101000006_create_duplicate_logs.ts
│   └── 20250101000007_create_digests.ts
│
├── seeds/                          # Database seed data for development
│   └── dev-seeds.ts
│
└── docs/                           # Project documentation
    ├── api.md                      # API endpoint reference
    ├── sources.md                  # Source registry format guide
    ├── quality.md                  # Quality scoring algorithm documentation
    ├── deployment.md               # Docker deployment guide
    └── architecture.md             # System architecture diagrams
```



---

## 3. Phased Implementation

> **Schedule Philosophy**: This is a personal project built on evenings and weekends (~10-15 hrs/week). Each phase is designed as a **vertical slice** - a complete, working increment that delivers end-to-end value. Phases build on each other but are independently testable.

### Phase 1: Static Source Registry + RSS Ingestion (Week 1)
> **Goal**: A working system that can scrape RSS feeds, store articles, and serve them via REST API.
> **MVP Deliverable**: `docker compose up` → RSS feeds ingested → articles queryable via API

| Deliverable | Files | Details |
|------------|-------|---------|
| [x] Project scaffolding | `Dockerfile`, `docker-compose.yml`, `tsconfig.json`, `package.json` | Multi-stage Docker build, TypeScript strict mode, dev/prod compose profiles |
| [x] Database schema + migrations | `migrations/*.ts`, `models/*.ts` | 7 tables: sources, feed_items, scrape_jobs, topics, quality_scores, duplicate_logs, digests |
| [x] Source registry YAML format | `config/sources/*.yaml`, `config/topics.yaml` | Human-readable source definitions per topic; validated on load |
| [x] RSS adapter | `adapters/rss.adapter.ts`, `adapters/base.adapter.ts` | Dual-parser (feedparser primary, rss-parser fallback); handles Atom + RSS 1.0/2.0 |
| [x] Source discovery command | `services/source-registry.ts` | CLI command: loads YAML, validates URLs, registers sources in DB |
| [x] Basic API endpoints | `api/routes/sources.ts`, `api/routes/feed.ts` | GET /sources (list), POST /sources (register), GET /feed (paginated), GET /feed/:id |
| [x] Docker compose setup | `docker-compose.yml` | App + PostgreSQL 16 + Redis 7 services, volume persistence, healthchecks |
| [x] Test suite foundation | `vitest.config.ts`, `tests/**/*.test.ts` | 80%+ coverage target for Phase 1 code |

**Tests**: RSS parsing (fixture-based), database CRUD operations, API endpoint responses, adapter error handling.

---

### Phase 2: HTML Article Extraction (Week 2)
> **Goal**: Full-text article extraction from HTML content, not just RSS summaries.

| Deliverable | Files | Details |
|------------|-------|---------|
| [x] HTML extraction pipeline | `core/extractor.ts`, `utils/html-cleaner.ts` | 3-tier fallback: readability → cheerio → trafilatura |
| [x] HTML article adapter | `adapters/html.adapter.ts` | Scrapes article pages, extracts body, cleans HTML |
| [x] Browser fallback adapter | `adapters/browser.adapter.ts` | Puppeteer for JS-rendered sites |
| [x] Date normalizer | `utils/date-normalizer.ts` | Handles RSS pubDate, meta tags, ISO 8601, relative dates ("2 hours ago") |
| [x] URL validator | `utils/url-validator.ts` | Normalization, domain extraction, fragment removal, safety checks |
| [x] Author extractor | `utils/author-extractor.ts` | Meta tags (author, byline), schema.org, ld+json, regex fallback |
| [x] Content type classifier | `core/categoriser.ts` | Identifies article type: news, blog post, press release, research paper |
| [x] Enhanced feed items | `models/feed-item.model.ts` | Full article text, reading time, word count, extracted author |

**Tests**: HTML extraction accuracy (fixture HTML pages), date parsing edge cases, author detection across 50+ test pages.

---

### Phase 3: Source Scoring + Deduplication (Week 3)
> **Goal**: Quality scoring system + deduplication to filter noise and remove duplicates.

| Deliverable | Files | Details |
|------------|-------|---------|
| [x] Source quality scorer | `core/quality-engine.ts` | Success rate, response time, article freshness, error rate per source |
| [x] Content quality scorer | `core/quality-engine.ts` | Word count, readability score, author presence, date freshness, image count |
| [x] Freshness decay engine | `core/quality-engine.ts` | Exponential decay based on age: score = base * e^(-age/half_life) |
| [x] Deduplication engine | `core/deduplicator.ts` | 5-stage pipeline: exact URL → content hash → near-duplicate (simhash) → semantic → manual |
| [x] Quality tier assignment | `models/quality-score.model.ts` | Tier 1 (must-read), Tier 2 (worth-reading), Tier 3 (background), Tier 4 (filtered) |
| [x] Override rules | `config/quality-rules.yaml` | Domain-specific boost/penalty rules, topic relevance weights |
| [x] Updated API with quality filters | `api/routes/feed.ts` | Query params: `?tier=1`, `?minScore=70`, `?topic=ai-llms` |

**Tests**: Scoring accuracy (fixture articles with known scores), deduplication detection rates, freshness decay formula verification.

---

### Phase 4: Summarisation + Categorisation (Week 4)
> **Goal**: Automatic summarisation, topic classification, and content ranking.

| Deliverable | Files | Details |
|------------|-------|---------|
| [x] Text summariser | `core/summariser.ts` | Extractive (sentence ranking by TF-IDF) + abstractive (LLM API for Tier 1 articles) |
| [x] Topic keyword matching | `core/categoriser.ts` | Multi-keyword matching per topic with weighted relevance scores |
| [x] Auto-tag assignment | `core/categoriser.ts` | Automatic tag extraction: company names, technologies, people mentioned |
| [x] Importance scoring | `core/quality-engine.ts` | Combined score: quality * relevance * freshness * source authority |
| [x] Content ranking | `services/feed-service.ts` | Ranked feed output: highest-importance articles first, within tiers |
| [x] Search endpoint | `services/search-service.ts` | Full-text search: `GET /feed/search?q=openai&topic=ai-llms` |

**Tests**: Summary quality evaluation (ROUGE scores vs human summaries), topic matching precision/recall, search relevance.

---

### Phase 5: Website Feed Integration (Week 5)
> **Goal**: Production-ready API with caching, pagination, and performance.

| Deliverable | Files | Details |
|------------|-------|---------|
| [x] Full-featured API | `api/routes/feed.ts`, `api/routes/topics.ts` | Pagination (cursor-based), sorting (score, date, relevance), compound filtering |
| [x] Feed endpoint with quality tiers | `services/feed-service.ts` | Return articles by tier, with tier distribution metadata |
| [x] Topic-specific feeds | `api/routes/topics.ts` | `GET /topics/ai-llms/feed` with topic-relevant ranking |
| [x] Search endpoint | `api/routes/feed.ts` | PostgreSQL full-text search with highlighting, typo tolerance |
| [x] Caching layer | Redis caching middleware | Response caching: 5-min for feeds, 1-min for search, cache invalidation on new articles |
| [x] API rate limiting | `api/hooks.ts` | Per-IP rate limiting: 100 req/min for personal use |
| [x] Response formatting | `api/schemas.ts` | Consistent JSON:API-style responses with metadata |

**Tests**: API response format compliance, pagination boundary cases, cache hit/miss behaviour, search latency <100ms.

---

### Phase 6: Telegram + n8n Automation (Week 6)
> **Goal**: Personal distribution via Telegram bot and n8n workflow triggers.

| Deliverable | Files | Details |
|------------|-------|---------|
| [x] Telegram bot integration | `utils/telegram-formatter.ts`, `jobs/notifier.worker.ts` | `/digest` command, breaking news alerts, inline article preview |
| [x] Digest generation service | `services/digest-service.ts` | Compiles top N articles per topic into formatted digest |
| [x] Daily digest cron job | `core/scheduler.ts` | 8:00 AM daily: trigger digest compilation → send via Telegram |
| [x] Breaking news detection | `services/breaking-news.ts` | High-velocity topic detection: multiple sources covering same story within 1 hour |
| [x] n8n webhook triggers | `api/routes/admin.ts` | Outbound webhooks for n8n workflow integration: new article, digest ready, breaking news |
| [x] Digest templates | `services/digest-service.ts` | MarkdownV2 formatting: emoji headers, ranked list, article summaries, source attribution |
| [x] Notification preferences | config layer | Per-topic notification settings: immediate, digest-only, muted |

**Tests**: Telegram message formatting (character limits, MarkdownV2 escaping), digest generation time, breaking news detection accuracy.

---

### Phase 7: Advanced Source Discovery (Week 7-8)
> **Goal**: Expand beyond RSS to GitHub, YouTube, sitemaps, and auto-discovery.

| Deliverable | Files | Details |
|------------|-------|---------|
| [x] GitHub release adapter | `adapters/github.adapter.ts` | Polls GitHub releases API, extracts release notes, changelog |
| [x] YouTube adapter | `adapters/youtube.adapter.ts` | Channel video feeds via RSS (YouTube provides RSS!), metadata extraction |
| [x] Sitemap adapter | `adapters/sitemap.adapter.ts` | Parses XML sitemaps, discovers article URLs, incremental crawling |
| [x] Search discovery adapter | `services/source-discovery.ts` | Google Custom Search / SerpAPI for discovering new sources per topic |
| [x] Source health monitoring | `services/source-health.ts` | Per-source dashboards: uptime, latency, error rate, article volume trends |
| [x] Auto-discovery for new topics | `services/source-discovery.ts` | Given a topic keyword, auto-find and rank potential RSS/HTML sources |
| [x] Admin dashboard API | `api/routes/admin.ts` | GET /admin/stats, /admin/sources/health, /admin/jobs, /admin/queue |
| [x] Bull Board integration | External (bull-board) | Web UI for queue inspection mounted at `/admin/queues` |

**Tests**: GitHub release parsing, YouTube RSS feed handling, sitemap parsing (large sitemaps), source discovery relevance scoring.



---

## 4. Per-Phase Detailed Tasks

### Phase 1: Static Source Registry + RSS Ingestion — Day-by-Day Breakdown

> **Phase 1 is the MVP foundation.** Everything builds on this phase. The goal is a **working system in 7 days** that can ingest RSS feeds and serve articles via API.

---

#### Day 1 (Saturday) — Project Scaffolding
**Commit milestone**: `feat: project scaffolding`

| # | Task | File(s) | Details |
|---|------|---------|---------|
| 1 | Initialize Node.js project | `package.json` | `npm init`, set `"type": "module"`, `"engines": { "node": ">=20.0.0" }` |
| 2 | Install core dependencies | `package.json` | `npm install fastify @fastify/swagger bullmq ioredis knex objection pg js-yaml dotenv zod pino puppeteer axios cheerio feedparser rss-parser` |
| 3 | Install dev dependencies | `package.json` | `npm install -D typescript @types/node vitest @vitest/coverage-v8 tsx eslint prettier @types/feedparser @types/js-yaml` |
| 4 | Configure TypeScript | `tsconfig.json` | `strict: true`, `target: ES2023`, `moduleResolution: bundler`, `outDir: ./dist`, `rootDir: ./src` |
| 5 | Configure Vitest | `vitest.config.ts` | `globals: true`, `environment: node`, coverage thresholds: `80%` statements |
| 6 | Create `.env.example` | `.env.example` | `DATABASE_URL`, `REDIS_URL`, `PORT`, `LOG_LEVEL`, `NODE_ENV` |
| 7 | Create `.gitignore` | `.gitignore` | `node_modules/`, `dist/`, `.env`, `*.log`, `coverage/` |
| 8 | Write `README.md` | `README.md` | Quickstart: `docker compose up` instructions |

**Function signatures (Day 1):**
```typescript
// src/config/index.ts
export function loadConfig(): AppConfig;
export function getDatabaseUrl(): string;
export function getRedisUrl(): string;

// src/utils/logger.ts
import { pino } from 'pino';
export function createLogger(name: string): pino.Logger;
export const defaultLogger: pino.Logger;
```

---

#### Day 2 (Sunday) — Docker + Database + Models
**Commit milestone**: `feat: docker compose + database schema`

| # | Task | File(s) | Details |
|---|------|---------|---------|
| 1 | Write `Dockerfile` | `Dockerfile` | Multi-stage: `node:20-alpine` → deps → build → runtime. `USER node`, `EXPOSE 3000` |
| 2 | Write `docker-compose.yml` | `docker-compose.yml` | Services: `app` (build: .), `postgres` (16-alpine), `redis` (7-alpine). Volumes + healthchecks |
| 3 | Write Knexfile | `knexfile.ts` | Client: `pg`, migrations directory, connection from env |
| 4 | Create migration: sources table | `migrations/20250101000001_create_sources.ts` | Columns: `id SERIAL PK`, `name`, `url`, `adapter_type`, `topic`, `config JSONB`, `is_active`, `last_fetched_at`, `created_at`, `updated_at`, `health_score` |
| 5 | Create migration: feed_items table | `migrations/20250101000002_create_feed_items.ts` | Columns: `id SERIAL PK`, `source_id FK`, `title`, `url UNIQUE`, `content TEXT`, `summary`, `author`, `published_at`, `scraped_at`, `raw_data JSONB`, `content_hash`, `is_duplicate BOOLEAN DEFAULT false` |
| 6 | Create migration: scrape_jobs | `migrations/20250101000003_create_scrape_jobs.ts` | Columns: `id SERIAL PK`, `source_id FK`, `bull_job_id`, `status`, `started_at`, `completed_at`, `error_message`, `items_count`, `metadata JSONB` |
| 7 | Create migration: topics | `migrations/20250101000004_create_topics.ts` | Columns: `id SERIAL PK`, `slug UNIQUE`, `name`, `description`, `keywords TEXT[]`, `is_active`, `priority` |
| 8 | Write base model | `models/base.model.ts` | `class BaseModel extends Model`: `$beforeInsert`, `$beforeUpdate` timestamps, soft delete |
| 9 | Write Source model | `models/source.model.ts` | Relations: `hasMany('FeedItem')`. Methods: `markFetched()`, `updateHealth(score)` |
| 10 | Write FeedItem model | `models/feed-item.model.ts` | Relations: `belongsTo('Source')`. Static methods: `findRecent(limit)`, `findByTopic(topic)` |
| 11 | Write Topic model | `models/topic.model.ts` | Static method: `findActive()` returns active topics ordered by priority |
| 12 | Test all models | `tests/unit/models/` | CRUD operations, relations, static methods (≥80% coverage) |

**Function signatures (Day 2):**
```typescript
// models/base.model.ts
export class BaseModel extends Model {
  id!: number;
  createdAt!: Date;
  updatedAt!: Date;
  $beforeInsert(): void;
  $beforeUpdate(): void;
}

// models/source.model.ts
export class Source extends BaseModel {
  static tableName = 'sources';
  static relationMappings = { ... };
  static async findActive(): Promise<Source[]>;
  static async findByTopic(topic: string): Promise<Source[]>;
  async markFetched(): Promise<void>;
  async updateHealth(score: number): Promise<void>;
  feedItems?: FeedItem[];
}

// models/feed-item.model.ts
export class FeedItem extends BaseModel {
  static tableName = 'feed_items';
  static async findRecent(limit: number, offset?: number): Promise<FeedItem[]>;
  static async findByTopic(topic: string, limit: number): Promise<FeedItem[]>;
  static async existsByUrl(url: string): Promise<boolean>;
  source?: Source;
}
```

---

#### Day 3 (Monday evening) — Source Registry YAML + Validation
**Commit milestone**: `feat: source registry YAML format`

| # | Task | File(s) | Details |
|---|------|---------|---------|
| 1 | Define YAML schema | `types/index.ts` | Zod schema for source config validation |
| 2 | Write registry loader | `services/source-registry.ts` | Load YAML → validate with Zod → insert/update DB |
| 3 | Create sample topic file | `config/topics.yaml` | AI-LLMs topic definition with keywords |
| 4 | Create sample source registry | `config/sources/ai-llms.yaml` | 5-10 real RSS sources (OpenAI blog, Anthropic, etc.) |
| 5 | Write registry sync command | `services/source-registry.ts` | `syncFromYaml(yamlPath)` — upserts sources, marks removed as inactive |
| 6 | Add validation logic | `services/source-registry.ts` | URL format check, feed reachability test, adapter type validation |
| 7 | Unit test registry loader | `tests/unit/services/source-registry.test.ts` | 10+ test cases: valid YAML, invalid URL, missing field, duplicate URL |

**Function signatures (Day 3):**
```typescript
// types/index.ts
export const SourceConfigSchema = z.object({
  name: z.string().min(1),
  url: z.string().url(),
  adapter: z.enum(['rss', 'html', 'api', 'github', 'youtube', 'browser']),
  topic: z.string(),
  priority: z.number().min(1).max(10).default(5),
  rateLimitMs: z.number().default(5000),
  selectors: z.record(z.string()).optional(),
  headers: z.record(z.string()).optional(),
});
export type SourceConfig = z.infer<typeof SourceConfigSchema>;

export const TopicConfigSchema = z.object({
  slug: z.string().regex(/^[a-z0-9-]+$/),
  name: z.string(),
  description: z.string(),
  keywords: z.array(z.string()),
  priority: z.number().min(1).max(10).default(5),
});
export type TopicConfig = z.infer<typeof TopicConfigSchema>;

// services/source-registry.ts
export class SourceRegistry {
  constructor(private db: typeof Source, private logger: Logger);
  async loadFromYaml(filepath: string): Promise<SyncResult>;
  async syncSources(configs: SourceConfig[]): Promise<{ inserted: number; updated: number; unchanged: number }>;
  async validateFeedUrl(url: string): Promise<{ valid: boolean; reachable: boolean; feedType?: string }>;
  async discoverRssFeed(baseUrl: string): Promise<string[]>; // probes common RSS paths
}
```

---

#### Day 4 (Tuesday evening) — RSS Adapter
**Commit milestone**: `feat: RSS adapter with dual-parser`

| # | Task | File(s) | Details |
|---|------|---------|---------|
| 1 | Write abstract base adapter | `adapters/base.adapter.ts` | Interface: `fetch()`, `parse()`, `validate()`. Shared: `createLogger`, rate limit decorator |
| 2 | Write RSS adapter (feedparser) | `adapters/rss.adapter.ts` | Primary parser using `feedparser` stream. Handles RSS 1.0/2.0, Atom, Dublin Core |
| 3 | Write RSS fallback (rss-parser) | `adapters/rss.adapter.ts` | When feedparser fails, retry with `rss-parser` which handles malformed feeds |
| 4 | Implement adapter factory | `adapters/index.ts` | `createAdapter(type, config)` returns correct adapter instance |
| 5 | Add rate limiting per source | `adapters/base.adapter.ts` | Token bucket rate limiter using in-memory queue (Redis-backed in Phase 5) |
| 6 | Add retry logic | `utils/retry-handler.ts` | Exponential backoff: 1s, 2s, 4s, 8s, 16s max 5 retries. Circuit breaker after 10 consecutive failures |
| 7 | Write unit tests for RSS adapter | `tests/unit/adapters/rss.adapter.test.ts` | 15+ tests: valid RSS, Atom, malformed XML, HTTP error, timeout, retry success |
| 8 | Create test fixtures | `tests/fixtures/rss/` | `valid-rss.xml`, `valid-atom.xml`, `malformed.xml`, `empty.xml` |

**Function signatures (Day 4):**
```typescript
// adapters/base.adapter.ts
export interface AdapterConfig {
  name: string;
  url: string;
  topic: string;
  priority: number;
  rateLimitMs: number;
  selectors?: Record<string, string>;
  headers?: Record<string, string>;
}

export abstract class BaseAdapter {
  protected config: AdapterConfig;
  protected logger: Logger;
  constructor(config: AdapterConfig, logger: Logger);
  abstract fetch(): Promise<RawFeedItem[]>;
  protected normalizeItem(raw: unknown): RawFeedItem;
  protected applyRateLimit(): Promise<void>;
  protected withRetry<T>(fn: () => Promise<T>, context: string): Promise<T>;
}

// adapters/rss.adapter.ts
export class RssAdapter extends BaseAdapter {
  async fetch(): Promise<RawFeedItem[]>;
  private parseWithFeedParser(stream: Readable): Promise<RawFeedItem[]>;
  private parseWithRssParser(xml: string): Promise<RawFeedItem[]>;
  private normalizeFeedparserItem(item: any): RawFeedItem;
  private normalizeRssParserItem(item: any): RawFeedItem;
}

// adapters/index.ts
export function createAdapter(config: AdapterConfig): BaseAdapter;
export const ADAPTER_REGISTRY: Record<string, new (config: AdapterConfig, logger: Logger) => BaseAdapter>;

// utils/retry-handler.ts
export interface RetryOptions {
  maxRetries: number;
  baseDelayMs: number;
  maxDelayMs: number;
  retryableStatuses: number[];
}
export async function withRetry<T>(
  operation: () => Promise<T>,
  options: RetryOptions,
  logger: Logger
): Promise<T>;
export class CircuitBreaker {
  constructor(threshold: number, resetTimeoutMs: number);
  async execute<T>(fn: () => Promise<T>): Promise<T>;
  getState(): 'CLOSED' | 'OPEN' | 'HALF_OPEN';
}
```

---

#### Day 5 (Wednesday evening) — BullMQ Queue + Collector
**Commit milestone**: `feat: BullMQ queue + scraping pipeline`

| # | Task | File(s) | Details |
|---|------|---------|---------|
| 1 | Configure BullMQ queue | `core/queue.ts` | `new Queue('scrape')`, Redis connection from env, job options: `removeOnComplete: 100`, `removeOnFail: 50` |
| 2 | Write scraper worker | `jobs/scraper.worker.ts` | Processes jobs from queue: fetches via adapter → normalizes → stores in DB |
| 3 | Write collector | `core/collector.ts` | Orchestrator: loads active sources → enqueues scrape jobs per source with priority |
| 4 | Write queue initialisation | `core/queue.ts` | `Queue`, `Worker`, `QueueScheduler` (for delayed retries) setup. DLQ config |
| 5 | Integrate Objection/Knex | `app.ts` | Database connection setup, model binding, migration runner on startup |
| 6 | Add source discovery CLI | `services/source-registry.ts` | `npm run sources:sync` command to load YAML and sync to database |
| 7 | Test queue operations | `tests/integration/queue.test.ts` | Enqueue job, worker processes, DLQ on failure, retry behaviour |

**Function signatures (Day 5):**
```typescript
// core/queue.ts
import { Queue, Worker, Job } from 'bullmq';

export const scrapeQueue = new Queue('scrape', { connection: redisConnection });
export const ScrapeWorker = new Worker(
  'scrape',
  async (job: Job<ScrapeJobData>) => { ... },
  { concurrency: 3, connection: redisConnection }
);

export interface ScrapeJobData {
  sourceId: number;
  sourceUrl: string;
  adapterType: string;
  topic: string;
  priority: number;
  retryCount: number;
}

// core/collector.ts
export class Collector {
  constructor(queue: Queue, sourceModel: typeof Source, logger: Logger);
  async collectAll(): Promise<{ enqueued: number; skipped: number }>;
  async collectTopic(topicSlug: string): Promise<{ enqueued: number }>;
  async collectSource(sourceId: number): Promise<void>;
  private async loadActiveSources(): Promise<Source[]>;
  private async enqueueSource(source: Source): Promise<void>;
}

// jobs/scraper.worker.ts
export async function processScrapeJob(job: Job<ScrapeJobData>): Promise<ScrapeResult>;
export interface ScrapeResult {
  itemsScraped: number;
  itemsNew: number;
  errors: string[];
  durationMs: number;
}
```

---

#### Day 6 (Thursday evening) — Fastify API Routes
**Commit milestone**: `feat: REST API endpoints`

| # | Task | File(s) | Details |
|---|------|---------|---------|
| 1 | Create Fastify app factory | `app.ts` | Plugin registration: swagger, CORS, helmet, error handler. Knex/Objection binding |
| 2 | Write Zod schemas | `api/schemas.ts` | Request/response schemas for all endpoints, typed with `z.infer<>` |
| 3 | Write sources routes | `api/routes/sources.ts` | `GET /sources` (list, filter by topic), `POST /sources` (register), `GET /sources/:id`, `DELETE /sources/:id` |
| 4 | Write feed routes | `api/routes/feed.ts` | `GET /feed` (paginated list), `GET /feed/:id`, `GET /feed/recent?limit=20`, query params: topic, since |
| 5 | Write health route | `api/routes/health.ts` | `GET /health` (DB + Redis connectivity), `GET /health/db`, `GET /health/redis` |
| 6 | Set up swagger | `app.ts` | `@fastify/swagger` + `@fastify/swagger-ui` at `/docs` |
| 7 | Write error handler | `api/errors.ts` | Zod validation errors → 400, DB errors → 500, custom error classes |
| 8 | Write API tests | `tests/integration/api.test.ts` | 20+ tests: GET/POST/DELETE, pagination, filtering, error cases, 404 handling |

**Function signatures (Day 6):**
```typescript
// api/schemas.ts
export const ListSourcesQuery = z.object({
  topic: z.string().optional(),
  active: z.boolean().optional().default(true),
  page: z.number().int().min(1).optional().default(1),
  limit: z.number().int().min(1).max(100).optional().default(20),
});
export type ListSourcesQuery = z.infer<typeof ListSourcesQuery>;

export const CreateSourceBody = z.object({
  name: z.string().min(1).max(200),
  url: z.string().url(),
  adapter: z.enum(['rss', 'html', 'api', 'github', 'youtube']),
  topic: z.string(),
  priority: z.number().min(1).max(10).optional().default(5),
});
export type CreateSourceBody = z.infer<typeof CreateSourceBody>;

export const ListFeedQuery = z.object({
  topic: z.string().optional(),
  since: z.string().datetime().optional(),
  page: z.number().int().min(1).optional().default(1),
  limit: z.number().int().min(1).max(100).optional().default(20),
});

// api/routes/sources.ts
export default async function sourceRoutes(fastify: FastifyInstance): Promise<void>;
// Registers: GET /sources, POST /sources, GET /sources/:id, DELETE /sources/:id

// api/routes/feed.ts
export default async function feedRoutes(fastify: FastifyInstance): Promise<void>;
// Registers: GET /feed, GET /feed/:id, GET /feed/recent

// api/routes/health.ts
export default async function healthRoutes(fastify: FastifyInstance): Promise<void>;
// GET /health → { status: 'ok', checks: { db: boolean, redis: boolean }, timestamp: ISO }
```

---

#### Day 7 (Friday evening) — Integration, Tests, Polish
**Commit milestone**: `feat: Phase 1 complete — full integration`

| # | Task | File(s) | Details |
|---|------|---------|---------|
| 1 | Write integration test: full scrape flow | `tests/integration/full-scrape.test.ts` | Source → enqueue → worker → database → API query. End-to-end with testcontainers |
| 2 | Write e2e test | `tests/e2e/full-pipeline.test.ts` | Full pipeline: register source → trigger scrape → verify API has articles |
| 3 | Add database seed data | `seeds/dev-seeds.ts` | 5 sample sources + sample feed items for development |
| 4 | Finalize docker-compose healthchecks | `docker-compose.yml` | Postgres `pg_isready`, Redis `redis-cli ping`, app `/health` endpoint |
| 5 | Write entry point | `src/index.ts` | `start()` function: connect DB → connect Redis → run migrations → register workers → start Fastify server → start scheduler |
| 6 | Verify coverage | `vitest --coverage` | ≥80% statement coverage across all new code |
| 7 | Integration polish | `app.ts`, `api/errors.ts` | Error logging, graceful shutdown (SIGTERM handler), connection cleanup |
| 8 | Documentation | `docs/deployment.md` | `docker compose up` quickstart, environment variables reference |

**Function signatures (Day 7):**
```typescript
// src/index.ts
export async function start(): Promise<void>;
export async function stop(): Promise<void>;
// Graceful shutdown: stop accepting new requests → finish active jobs → close DB → close Redis → exit
```

**Phase 1 Test Summary:**

| Test Suite | Count | Coverage Target |
|-----------|-------|----------------|
| Unit: models | 8 test files, ~40 tests | 90% |
| Unit: adapters | 2 test files, ~20 tests | 85% |
| Unit: services | 1 test file, ~10 tests | 85% |
| Unit: utils | 2 test files, ~15 tests | 85% |
| Integration: API | 1 test file, ~25 tests | 80% |
| Integration: queue | 1 test file, ~10 tests | 80% |
| E2E: pipeline | 1 test file, ~5 tests | N/A (end-to-end) |
| **Total** | **~125 tests** | **≥80% overall** |

---

### Phase 2-7 High-Level Task Lists

#### Phase 2: HTML Article Extraction (Week 2)

| Day | Focus | Key Tasks |
|-----|-------|-----------|
| Day 8 (Sat) | Readability integration | Install `@mozilla/readability`, write `core/extractor.ts` readability tier |
| Day 9 (Sun) | Cheerio fallback | Write cheerio-based extraction, custom selector support, HTML cleaning |
| Day 10 (Mon) | Puppeteer fallback | Write `adapters/browser.adapter.ts` for JS-rendered sites, headless Chromium |
| Day 11 (Tue) | Date normalizer | `utils/date-normalizer.ts` — 20+ date format patterns, relative date parsing |
| Day 12 (Wed) | Author extractor | `utils/author-extractor.ts` — meta tags, schema.org, byline regex patterns |
| Day 13 (Thu) | URL validator | `utils/url-validator.ts` — normalization, dedup, canonical URL extraction |
| Day 14 (Fri) | Tests + polish | 30+ fixture HTML pages, extraction accuracy tests, edge cases |

#### Phase 3: Source Scoring + Deduplication (Week 3)

| Day | Focus | Key Tasks |
|-----|-------|-----------|
| Day 15 (Sat) | Quality engine design | `core/quality-engine.ts` — signal definitions, scoring formula, weights |
| Day 16 (Sun) | Source quality scorer | Success rate, response time, error rate per source over 7-day window |
| Day 17 (Mon) | Content quality scorer | Word count, readability (Flesch-Kincaid), image count, freshness score |
| Day 18 (Tue) | Deduplication engine | 5-stage pipeline: URL match → SHA-256 hash → simhash (near-dup) → TF-IDF similarity → manual |
| Day 19 (Wed) | Freshness decay | Exponential decay formula, configurable half-life per topic |
| Day 20 (Thu) | Tier assignment | `config/quality-rules.yaml`, tier thresholds, override rules |
| Day 21 (Fri) | API filters + tests | `?tier=1&minScore=70`, scoring accuracy tests, dedup benchmarks |

#### Phase 4: Summarisation + Categorisation (Week 4)

| Day | Focus | Key Tasks |
|-----|-------|-----------|
| Day 22 (Sat) | Extractive summariser | TF-IDF sentence ranking, TextRank algorithm, max summary length |
| Day 23 (Sun) | LLM abstractive summariser | OpenAI API integration for Tier 1 articles, prompt engineering, cost budget |
| Day 24 (Mon) | Topic categoriser | Keyword matching engine, multi-topic assignment, relevance scoring |
| Day 25 (Tue) | Auto-tag assignment | Named entity extraction: companies, people, technologies, frameworks |
| Day 26 (Wed) | Importance scoring | Combined formula: quality × relevance × freshness × authority |
| Day 27 (Thu) | Content ranking | `services/feed-service.ts` ranking algorithm, A/B test with manual ratings |
| Day 28 (Fri) | Search endpoint + tests | PostgreSQL full-text search, `tsvector` index, search relevance ranking |

#### Phase 5: Website Feed Integration (Week 5)

| Day | Focus | Key Tasks |
|-----|-------|-----------|
| Day 29 (Sat) | Cursor-based pagination | `?cursor=eyJpZCI6MTIzfQ==&limit=20`, stable ordering |
| Day 30 (Sun) | Advanced filtering | Compound filters: topic + tier + date range + keyword, filter validation |
| Day 31 (Mon) | Sorting options | `?sortBy=score|date|relevance`, `?sortOrder=asc\|desc` |
| Day 32 (Tue) | Response caching | Redis response cache, cache key generation, TTL per endpoint, invalidation |
| Day 33 (Wed) | Rate limiting | `@fastify/rate-limit`, Redis store, 100 req/min personal limit |
| Day 34 (Thu) | Topic feeds | `GET /topics/:slug/feed` with topic-specific ranking and filters |
| Day 35 (Fri) | API docs + performance tests | Swagger completeness, k6 load tests, <100ms p95 response time |

#### Phase 6: Telegram + n8n Automation (Week 6)

| Day | Focus | Key Tasks |
|-----|-------|-----------|
| Day 36 (Sat) | Telegram bot setup | `node-telegram-bot-api`, webhook/polling, `/start`, `/digest` commands |
| Day 37 (Sun) | Message formatting | `utils/telegram-formatter.ts` — MarkdownV2 escaping, emoji headers, article cards |
| Day 38 (Mon) | Digest service | `services/digest-service.ts` — compile top N articles per topic, format Markdown |
| Day 39 (Tue) | Daily digest cron | `core/scheduler.ts` — 8:00 AM daily, `node-cron` expression, timezone aware |
| Day 40 (Wed) | Breaking news detection | `services/breaking-news.ts` — velocity algorithm: 3+ sources, same keywords, 1-hour window |
| Day 41 (Thu) | n8n webhooks | `api/routes/admin.ts` — outbound webhooks: `new_article`, `digest_ready`, `breaking_news` |
| Day 42 (Fri) | Notification preferences | Per-topic: immediate/digest/muted, preference storage, preference-aware routing |

#### Phase 7: Advanced Source Discovery (Week 7-8)

| Day | Focus | Key Tasks |
|-----|-------|-----------|
| Day 43 (Sat) | GitHub adapter | `adapters/github.adapter.ts` — GitHub API releases, rate limit handling, release notes |
| Day 44 (Sun) | YouTube adapter | `adapters/youtube.adapter.ts` — YouTube RSS feeds, video metadata, description extraction |
| Day 45 (Mon) | Sitemap adapter | `adapters/sitemap.adapter.ts` — XML sitemap parsing, URL discovery, incremental crawl |
| Day 46 (Tue) | Source discovery engine | `services/source-discovery.ts` — Google Custom Search for topic keywords, source ranking |
| Day 47 (Wed) | Source health dashboard | `services/source-health.ts` — 7-day uptime, latency trends, article volume tracking |
| Day 48 (Thu) | Auto-discovery | Given topic → search → rank candidate sources → suggest additions |
| Day 49-50 (Fri-Sat) | Admin API + Bull Board | `api/routes/admin.ts` stats endpoints, Bull Board at `/admin/queues` |
| Day 51-56 (Sun-Fri) | Final polish | Bug fixes, performance optimisation, documentation, README completion |



---

## 5. Dependencies Between Phases

### Dependency Graph

```
Phase 1: Static Source Registry + RSS Ingestion
│
├──► Phase 2: HTML Article Extraction
│    │  (depends on: feed items from Phase 1, models, database)
│    │
│    ├──► Phase 3: Source Scoring + Deduplication
│    │    │  (depends on: extracted content from Phase 2, feed items)
│    │    │
│    │    ├──► Phase 4: Summarisation + Categorisation
│    │    │    │  (depends on: scored + deduped content from Phase 3)
│    │    │    │
│    │    │    ├──► Phase 5: Website Feed Integration
│    │    │    │    │  (depends on: ranked, summarised content from Phase 4)
│    │    │    │    │  (independent: can use raw feed without Phase 4)
│    │    │    │    │
│    │    │    │    └──► Phase 6: Telegram + n8n Automation ◄──────┐
│    │    │    │         │  (depends on: API + feed from Phase 5)   │
│    │    │    │         │  (depends on: digest content from Phase 4)
│    │    │    │         │  (alternative: can work with Phase 3 content)
│    │    │    │         │                                         │
Phase 1 ────────────────────────────────────────────────────────────┘
│
└──► Phase 7: Advanced Source Discovery
     │  (depends on: Phase 1 scaffolding, can parallel with Phase 2-6)
     │  (soft dependency: best after Phase 3 for scoring context)
```

### Detailed Dependency Matrix

| From Phase | To Phase | Dependency Type | Description |
|-----------|----------|----------------|-------------|
| Phase 1 | Phase 2 | **Hard** | Phase 2 requires the database schema (`feed_items` table), models (`FeedItem`, `Source`), and adapter infrastructure from Phase 1 |
| Phase 1 | Phase 3 | **Hard** | Phase 3 requires feed items and source data from Phase 1; can work with RSS-only content before Phase 2 completes |
| Phase 2 | Phase 3 | **Soft** | Phase 3 works better with full-text content from Phase 2, but can score RSS summaries as a starting point |
| Phase 3 | Phase 4 | **Hard** | Phase 4 summarisation runs on quality-scored, deduplicated content from Phase 3 |
| Phase 3 | Phase 6 | **Soft** | Phase 6 digest can use Phase 3 tiered content without waiting for Phase 4 summaries |
| Phase 4 | Phase 5 | **Soft** | Phase 5 API serves content; Phase 4 adds value but API works with pre-Phase-4 data |
| Phase 4 | Phase 6 | **Soft** | Phase 6 digests benefit from Phase 4 summaries but can use excerpts |
| Phase 5 | Phase 6 | **Hard** | Phase 6 Telegram bot consumes the Phase 5 API endpoints for feed data |
| Phase 1 | Phase 7 | **Hard** | Phase 7 uses the scaffolding, models, and adapter pattern from Phase 1 |
| Phase 3 | Phase 7 | **Soft** | Phase 7 source health monitoring is enhanced by Phase 3 scoring context |

### Parallelisation Opportunities

```
Week 1:  ████████████████████████████████████████ Phase 1 (blocking)
Week 2:  ████████████████████████████████████████ Phase 2
Week 3:  ████████████████████████████████████████ Phase 3
Week 4:  ████████████████████████████████████████ Phase 4
Week 5:  ████████████████████████████████████████ Phase 5
Week 6:  ████████████████████████████████████████ Phase 6
Week 7:  ██████████████████████████████░░░░░░░░░░ Phase 7 (part 1)
Week 8:  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████ Phase 7 (part 2) + Polish
```

**Key parallelisation notes:**
- Phase 7 work (GitHub adapter, YouTube adapter) can begin in Week 5 if Phase 1 scaffolding is solid
- Phase 6 Telegram work can begin after Phase 3 completes (uses tiered content) — don't need to wait for Phase 5
- Phase 5 API enhancements can begin after Phase 3 (uses scored content) — summaries are additive

### Minimal Viable Integration Points

At the end of each phase, the system must satisfy these integration checkpoints:

| After Phase | Integration Checkpoint | Test Command |
|------------|----------------------|--------------|
| Phase 1 | `docker compose up` → RSS scrape → API serves articles | `curl http://localhost:3000/feed` returns JSON with articles |
| Phase 2 | Full-text extraction from HTML sources | `curl http://localhost:3000/feed/1` includes `content` field with article body |
| Phase 3 | Scored feed with tiers, no duplicates | `curl 'http://localhost:3000/feed?tier=1'` returns only high-quality articles, no duplicates |
| Phase 4 | Summarised, categorised, ranked feed | `curl 'http://localhost:3000/feed?topic=ai-llms'` returns ranked, summarised articles |
| Phase 5 | Cached API with search | `curl 'http://localhost:3000/feed/search?q=openai'` returns search results |
| Phase 6 | Telegram digest delivery | `/digest` command in Telegram bot returns formatted digest |
| Phase 7 | GitHub releases in feed | `curl 'http://localhost:3000/feed?source=github'` includes GitHub release notes |

---

## 6. Technology Decisions Log

### Core Technology Decisions

| Decision | Choice | Alternatives Considered | Rationale |
|----------|--------|------------------------|-----------|
| **Runtime** | Node.js 20 LTS | Python 3.12, Deno, Bun | Node.js has the richest scraping ecosystem (`cheerio`, `puppeteer`, `feedparser`); n8n compatibility (Node-based); single-language full stack; async/await maturity |
| **Framework** | Fastify 5.x | Express, Hono, Elysia | 3x faster than Express; native TypeScript; schema-based validation; excellent plugin ecosystem; low memory footprint |
| **Database** | PostgreSQL 16 | SQLite, MongoDB, MySQL 8 | Full-text search (`tsvector`/`tsquery`) for search endpoint; JSONB for flexible metadata; proven reliability; excellent Docker tooling |
| **Queue** | BullMQ 5.x + Redis 7 | Bull, Agenda, RabbitMQ | Native TypeScript; native priority support; job scheduling; rate limiting; dead letter queues; excellent observability with bull-board |
| **RSS Parsing** | feedparser + rss-parser | rss-parser only, fast-xml-parser | `feedparser` is the gold standard but can fail on malformed feeds; `rss-parser` handles edge cases; dual-parser = maximum feed compatibility |
| **HTML Extraction** | readability + cheerio + trafilatura | JSDOM, Playwright-only, single library | Tiered approach maximises extraction success rate: readability (80% of sites) → cheerio (custom selectors) → trafilatura (deep extraction fallback) |
| **Browser** | Puppeteer 23.x | Playwright, Selenium | Puppeteer has the largest community; works well with Docker; `--no-sandbox` mode for containers; sufficient for JS-rendered fallback |
| **HTTP Client** | Axios 1.7.x | fetch, got, undici | Interceptors for request/response transformation; automatic retries; timeout handling; browser-like headers; proven in production |
| **Validation** | Zod 3.23.x | Joi, Yup, class-validator | TypeScript-first: infer types from schemas; excellent error messages; composable; no decorator magic; smaller bundle |
| **ORM/Query** | Knex + Objection | Prisma, TypeORM, Drizzle | SQL-first query builder (explicit queries, no magic); Objection adds lightweight ORM; migration support; no code generation step |
| **Scheduler** | node-cron + BullMQ | node-schedule, Agenda, Bull | Separation of concerns: `node-cron` for cron expressions, BullMQ for actual execution; BullMQ handles retries, DLQ, concurrency |
| **Testing** | Vitest 2.x | Jest, Mocha + Chai | Native ESM support; TypeScript out of the box; 10x faster; compatible with Jest API; built-in coverage; watch mode |
| **AI/LLM** | OpenAI API | Local LLM (Ollama), Claude API, Cohere | `gpt-4o-mini` is cost-effective ($0.15/$0.60 per 1M tokens); reliable API; future: add local inference fallback for sensitive data |
| **Bot Platform** | Telegram | Discord, Slack, WhatsApp | Simple API; excellent for personal use; webhook + polling; Markdown support; free; no rate limits for personal bots |
| **Config Format** | YAML + dotenv | JSON, TOML | YAML is human-readable for source registries; `.env` for secrets; `js-yaml` is mature; supports comments |
| **Logging** | Pino 9.x | Winston, Bunyan, console | Fastest JSON logger; structured logging; child loggers; pretty-print in dev; file rotation with `pino-roll` |
| **Container** | Docker Compose | Kubernetes, Podman | Single-node deployment; service orchestration; volume persistence; one-command setup; no orchestration complexity |

### Decisions Requiring Re-evaluation

| Decision | Re-evaluate At | Trigger | Alternatives |
|----------|---------------|---------|-------------|
| OpenAI API (cloud) | Phase 4 completion | Monthly cost > $20 | Ollama (local LLM), switching to gpt-4o-mini already planned |
| Puppeteer | Phase 7 | Chrome memory > 2GB | Playwright (lighter), jsdom for simple cases |
| PostgreSQL | Phase 5 | Search latency > 200ms | Meilisearch or Typesense for search, keep PG as primary |
| BullMQ | Phase 5 | Queue depth > 10,000 | Redis Streams, or add Redis Cluster for scaling |
| Knex + Objection | Phase 7 | Query complexity issues | Drizzle ORM (TypeScript-native, growing ecosystem) |

### Cost Projections (Personal Use)

| Service | Estimated Monthly Cost | Notes |
|---------|----------------------|-------|
| OpenAI API (gpt-4o-mini) | $5-15 | ~500 articles/day summarised, Tier 1 only |
| Hosting (local) | $0 | Runs on existing hardware via Docker |
| Telegram Bot API | $0 | Free for personal use |
| **Total** | **$5-15/month** | Can reduce to $0 with local LLM fallback |

---

## Appendix A: Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NODE_ENV` | Yes | `development` | `development` \| `production` \| `test` |
| `PORT` | No | `3000` | Fastify server port |
| `LOG_LEVEL` | No | `info` | `trace` \| `debug` \| `info` \| `warn` \| `error` \| `fatal` |
| `DATABASE_URL` | Yes | — | PostgreSQL connection string: `postgresql://user:pass@host:5432/dbname` |
| `REDIS_URL` | No | `redis://localhost:6379` | Redis connection string |
| `OPENAI_API_KEY` | No | — | OpenAI API key for summarisation (Phase 4+) |
| `TELEGRAM_BOT_TOKEN` | No | — | Telegram bot token from @BotFather (Phase 6+) |
| `TELEGRAM_CHAT_ID` | No | — | Your personal Telegram chat ID (Phase 6+) |
| `CRON_SCHEDULE` | No | `0 8 * * *` | Daily digest cron expression (default: 8:00 AM) |
| `N8N_WEBHOOK_URL` | No | — | n8n webhook base URL (Phase 6+) |
| `RATE_LIMIT_RPS` | No | `2` | Requests per second limit per domain |
| `MAX_RETRIES` | No | `5` | Maximum retry attempts per failed job |
| `PUPPETEER_ARGS` | No | `--no-sandbox,--disable-setuid-sandbox` | Chromium launch arguments |

## Appendix B: Quick Start Commands

```bash
# 1. Clone and setup
git clone <repo>
cd jarvis-scraper
cp .env.example .env
# Edit .env with your values

# 2. Start infrastructure
docker compose up -d postgres redis

# 3. Run migrations
npm run db:migrate

# 4. Seed development data
npm run db:seed

# 5. Sync sources from YAML
npm run sources:sync

# 6. Start development server
npm run dev
# Server runs on http://localhost:3000
# API docs at http://localhost:3000/docs

# 7. Trigger initial scrape
curl -X POST http://localhost:3000/admin/jobs/trigger

# 8. View feed
curl http://localhost:3000/feed

# Production deployment
docker compose -f docker-compose.yml up -d
```

## Appendix C: Test Commands

```bash
# Run all tests
npm test

# Run with coverage
npm run test:coverage

# Run unit tests only
npm run test:unit

# Run integration tests only
npm run test:integration

# Run e2e tests
npm run test:e2e

# Run tests in watch mode
npm run test:watch

# Run specific test file
npx vitest src/tests/unit/adapters/rss.adapter.test.ts
```

## Appendix D: npm Scripts Reference

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:unit": "vitest run src/tests/unit",
    "test:integration": "vitest run src/tests/integration",
    "test:e2e": "vitest run src/tests/e2e",
    "db:migrate": "knex migrate:latest",
    "db:migrate:rollback": "knex migrate:rollback",
    "db:migrate:make": "knex migrate:make",
    "db:seed": "knex seed:run",
    "sources:sync": "tsx src/cli/sync-sources.ts",
    "sources:discover": "tsx src/cli/discover-sources.ts",
    "sources:health": "tsx src/cli/check-health.ts",
    "queue:work": "tsx src/cli/start-worker.ts",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write src/",
    "docker:build": "docker compose build",
    "docker:up": "docker compose up -d",
    "docker:down": "docker compose down",
    "docker:logs": "docker compose logs -f app"
  }
}
```

---

*Document generated as part of the Jarvis HQ architecture specification. This build plan serves as the executable roadmap for the phased development of the personal AI news scraping system.*

