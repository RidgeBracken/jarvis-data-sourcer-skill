# Phase 1 Build Prompt

## Table of Contents

- **Preamble** — Purpose, target stack, who can run this prompt
- **Overview** — What Phase 1 delivers
- **Tech Stack** — Node.js/TypeScript reference dependencies
- **Scope — Phase 1 ONLY** — In-scope and explicitly out-of-scope items
- **Directory Structure** — File tree to create
- **Files to Create** — Section per file (#1–#41), each with purpose, requirements, and signatures
  - #1–#13: Docker, config, source registry YAML
  - #14: Shared types (`src/types/index.ts`)
  - #15–#17: Adapters (`base`, `rss`, factory)
  - #18–#20: Scheduler, queue, collector
  - #21–#23: Models (Objection.js)
  - #24–#29: API routes + schemas + error handler
  - #30: Pino logger
  - #31: Token bucket rate limiter
  - #32: Initial migration SQL
  - #33: Seed data
  - #34–#37: RSS fixtures
  - #38–#39: Unit + integration tests
  - #40: `sync-sources` CLI
  - **#40b: Robots.txt compliance** (mandatory; called by every adapter)
- **Verification Criteria** — `curl`/test-based done checklist
- **Implementation Notes** — Edge cases, env handling, error policy

> **Purpose**: A copy-paste-ready build prompt for Phase 1 of the Jarvis HQ data
> scraper. It targets a Node.js/TypeScript reference stack (per
> `architecture.md` §4) and was originally authored for OpenAI Codex, but is
> reusable by any code-generating agent (Claude Code, the Claude Agent SDK,
> Cursor, Aider, Kimi/OpenClaw, GitHub Copilot Workspace, etc.). It is
> intentionally self-contained so that the agent does not need to ask
> clarifying questions.
>
> **If your target stack is not Node.js/TypeScript**, treat this file as a
> high-level checklist (services, files, contracts, verification criteria)
> and adapt language-specific details to your stack. The safety rules in
> `../SKILL.md` and `security.md` apply regardless of stack.

---

```
# Build Jarvis HQ Data Scraper -- Phase 1

## Overview
Build a Dockerized Node.js/TypeScript service that ingests RSS feeds from configured sources, stores articles in PostgreSQL, and exposes a REST API. This is Phase 1 of a personal AI news scraping system -- it establishes the foundation that all later phases build upon.

**What Phase 1 delivers**: A working system where `docker compose up` starts all services, source registries (YAML) define RSS feeds to monitor, a BullMQ+Redis job queue triggers RSS scraping, parsed feed items are stored in PostgreSQL, and a Fastify REST API serves the data.

## Tech Stack
- Node.js 20 LTS (Alpine Linux in Docker)
- TypeScript 5.x with strict mode
- Fastify 4.x (web framework with built-in schema validation)
- PostgreSQL 16 (relational database, Alpine Linux)
- Redis 7 (job queue backing + caching, Alpine Linux)
- BullMQ 5.x (job queue with priority, retries, dead-letter support)
- Knex.js 3.x + Objection.js 3.x (query builder + lightweight ORM)
- Zod 3.23.x (TypeScript-first schema validation)
- feedparser 2.2.x + rss-parser 3.13.x (RSS/Atom parsing, dual-parser for compatibility)
- cheerio 1.0.x (HTML parsing for Phase 2 prep)
- axios 1.7.x (HTTP client with interceptors and retries)
- js-yaml 4.1.x + dotenv 16.x (configuration)
- pino 9.x (structured JSON logging)
- vitest 2.x (testing framework with coverage)

## Scope -- Phase 1 ONLY
This phase implements exactly these capabilities:

1. Docker Compose with 3 services (app, PostgreSQL, Redis)
2. Source registry in YAML (2 topic files with real RSS feeds)
3. RSS adapter using feedparser (primary) with rss-parser fallback
4. PostgreSQL database with `sources` and `feed_items` tables
5. Fastify REST API: GET /sources, GET /feed, POST /scrape, GET /health
6. BullMQ job queue for scraping with a worker
7. Basic quality scoring (placeholder -- assigns default scores)
8. Docker setup with .env configuration
9. Tests: RSS parsing (fixture-based), API endpoints, database operations
10. **Robots.txt fetcher + parser** at `src/utils/robots.ts`. Called by every adapter before any HTTP fetch. If robots disallows the URL, log a `compliance.robots_blocked` event and skip the fetch. Cache parsed robots per-domain for 24h. Respect the `Crawl-delay` directive (override the per-source `rate_limit` if robots specifies a longer delay — most-restrictive wins). This is non-optional: a working Phase 1 must enforce robots.txt before any fetch.

DO NOT implement: HTML extraction, browser automation, Telegram, n8n webhooks, deduplication, summarisation, categorisation, GitHub/YouTube adapters, sitemap parsing, or admin dashboard. Those are Phase 2+.

## Directory Structure

Create the following file tree:

```
jarvis-scraper/
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── .gitignore
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── knexfile.ts
│
├── src/
│   ├── index.ts                    # Entry point: bootstrap everything
│   ├── app.ts                      # Fastify app factory
│   │
│   ├── config/
│   │   ├── index.ts                # Config loader: merges .env + YAML
│   │   └── sources/
│   │       ├── ai-openai.yaml      # OpenAI + AI topic sources
│   │       └── tech-microsoft.yaml # Microsoft tech topic sources
│   │
│   ├── adapters/
│   │   ├── base.adapter.ts         # Abstract base adapter class
│   │   ├── rss.adapter.ts          # RSS/Atom feed parser
│   │   └── index.ts                # Adapter factory
│   │
│   ├── core/
│   │   ├── scheduler.ts            # Cron-based job scheduling
│   │   ├── queue.ts                # BullMQ queue + worker setup
│   │   └── collector.ts            # Orchestrates adapter calls
│   │
│   ├── models/
│   │   ├── base.model.ts           # Base Objection model with timestamps
│   │   ├── source.model.ts         # Source table model
│   │   └── feed-item.model.ts      # Feed items table model
│   │
│   ├── api/
│   │   ├── routes/
│   │   │   ├── sources.ts          # GET /sources, POST /sources
│   │   │   ├── feed.ts             # GET /feed, GET /feed/:id
│   │   │   ├── scrape.ts           # POST /scrape
│   │   │   └── health.ts           # GET /health
│   │   ├── schemas.ts              # Zod request/response schemas
│   │   └── errors.ts               # Central error handling
│   │
│   ├── utils/
│   │   ├── logger.ts               # Pino logger factory
│   │   └── rate-limiter.ts         # Token bucket rate limiter
│   │
│   └── types/
│       └── index.ts                # Shared TypeScript types + Zod schemas
│
├── migrations/
│   └── 001_initial.sql             # Database schema creation
│
├── seeds/
│   └── dev-seeds.ts                # Development seed data
│
├── tests/
│   ├── fixtures/
│   │   └── rss/
│   │       ├── valid-rss-2.0.xml   # Standard RSS 2.0 fixture
│   │       ├── valid-atom.xml      # Atom feed fixture
│   │       ├── malformed.xml       # Malformed XML fixture
│   │       └── empty.xml           # Empty feed fixture
│   ├── unit/
│   │   └── adapters/
│   │       └── rss.adapter.test.ts # RSS adapter unit tests
│   └── integration/
│       └── api.test.ts             # API endpoint integration tests
```

## Files to Create

For each file below, implement it completely. Every file must be production-quality TypeScript with proper types, error handling, and logging.

---

### 1. docker-compose.yml
**Purpose**: Orchestrate the 3 services (app, PostgreSQL 16, Redis 7).

Requirements:
- `app` service: build from Dockerfile, port 3000, depends_on db and redis, restart unless-stopped, healthcheck on GET /health
- `db` service: postgres:16-alpine, port 5432, environment variables for user/password/db, volume `pgdata` for persistence, healthcheck via `pg_isready`
- `redis` service: redis:7-alpine, port 6379, volume `redisdata` for persistence, healthcheck via `redis-cli ping`
- All three services on a shared network `jarvis-net`
- App service passes DATABASE_URL and REDIS_URL as environment variables
- App waits for db and redis healthchecks before starting

---

### 2. Dockerfile
**Purpose**: Multi-stage Docker build for the Node.js app.

Requirements:
- Stage 1 (deps): `node:20-alpine`, install production dependencies only
- Stage 2 (build): copy source, install dev deps, run `tsc`
- Stage 3 (runtime): copy built output + production node_modules, expose 3000, `USER node`, `CMD ["node", "dist/index.js"]`
- Use `.dockerignore` to exclude node_modules, dist, .env, coverage
- Healthcheck: `curl -f http://localhost:3000/health || exit 1`

---

### 3. .env.example
**Purpose**: Template for all environment variables.

Create with these variables (empty values for secrets):
```
NODE_ENV=development
PORT=3000
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://jarvis:jarvis@db:5432/jarvis_hq

# Redis
REDIS_URL=redis://redis:6379

# Scraper config
RATE_LIMIT_RPS=1
MAX_RETRIES=5
SCRAPE_INTERVAL_MINUTES=30

# API docs
ENABLE_SWAGGER=true
```

---

### 4. .gitignore
**Purpose**: Exclude files from version control.

Include: node_modules/, dist/, .env, *.log, coverage/, .DS_Store, *.tsbuildinfo

---

### 5. package.json
**Purpose**: Project manifest with dependencies and scripts.

Requirements:
- `name`: "jarvis-scraper", `version`: "1.0.0", `type`: "module"
- `engines`: `{ "node": ">=20.0.0" }`
- Scripts: `dev` (tsx watch), `build` (tsc), `start` (node dist/index.js), `test` (vitest run), `test:watch` (vitest), `test:coverage` (vitest run --coverage), `db:migrate` (knex migrate:latest), `db:seed` (knex seed:run), `sources:sync` (tsx src/cli/sync-sources.ts)
- Dependencies: fastify, @fastify/swagger, @fastify/swagger-ui, bullmq, ioredis, knex, objection, pg, js-yaml, dotenv, zod, pino, axios, feedparser, rss-parser, cheerio, node-cron
- Dev dependencies: typescript, @types/node, vitest, @vitest/coverage-v8, tsx, @types/feedparser

---

### 6. tsconfig.json
**Purpose**: TypeScript strict configuration.

Requirements:
- `target`: "ES2023", `module`: "NodeNext", `moduleResolution`: "NodeNext"
- `strict`: true, `esModuleInterop`: true, `skipLibCheck`: true
- `outDir`: "./dist", `rootDir`: "./src", `declaration`: true
- `resolveJsonModule`: true, `forceConsistentCasingInFileNames`: true

---

### 7. vitest.config.ts
**Purpose**: Test configuration.

Requirements:
- `globals`: true, `environment`: "node"
- Coverage: provider "v8", thresholds -- statements 80, branches 70, functions 80, lines 80
- Include: `src/**/*.ts`, `tests/**/*.test.ts`
- Exclude: `dist/`, `node_modules/`

---

### 8. knexfile.ts
**Purpose**: Knex database configuration.

Requirements:
- Default config reads DATABASE_URL from env
- Client: "pg", migrations directory: "./migrations", seeds directory: "./seeds"
- Connection pool: min 2, max 10
- Export configs for development, production, and test environments

---

### 9. src/index.ts
**Purpose**: Application entry point. Bootstraps database, Redis, queue workers, scheduler, and Fastify server.

Requirements:
- Import and call `buildApp()` from `./app`
- Set up Knex connection and bind to Objection Model.knex()
- Run pending migrations on startup via Knex
- Initialize BullMQ queue and workers
- Start the scheduler (if NODE_ENV is not test)
- Start Fastify server on port from env (default 3000)
- Graceful shutdown on SIGTERM: stop accepting requests, finish active jobs, close DB, close Redis, exit
- Log startup and shutdown events with structured logging

Function signature:
```typescript
export async function start(): Promise<void>;
export async function stop(signal: string): Promise<void>;
```

---

### 10. src/app.ts
**Purpose**: Fastify app factory. Creates and configures the Fastify instance with plugins, routes, and error handling.

Requirements:
- Create Fastify instance with logger (use Pino)
- Register @fastify/swagger with OpenAPI 3.0 spec
- Register @fastify/swagger-ui at `/docs`
- Register CORS plugin
- Register all routes from `./api/routes/*`
- Global error handler: Zod validation errors → 400 with details, DB errors → 500, generic errors → 500
- Export a `buildApp()` function that returns the configured Fastify instance
- Export the app for testing

Function signature:
```typescript
export async function buildApp(): Promise<FastifyInstance>;
```

---

### 11. src/config/index.ts
**Purpose**: Configuration loader that merges `.env` variables with YAML source registries.

Requirements:
- Load `.env` via dotenv
- Read all YAML files from `src/config/sources/*.yaml` using js-yaml
- Validate each source config with the Zod `SourceConfigSchema` (defined in types)
- Export `getConfig()` returning merged config: env vars + sources array
- Export `getDatabaseUrl()`, `getRedisUrl()`, `getPort()`, `getLogLevel()`
- Export `getSources()` returning validated SourceConfig[]

Function signatures:
```typescript
export function loadConfig(): AppConfig;
export function getDatabaseUrl(): string;
export function getRedisUrl(): string;
export function getPort(): number;
export function getLogLevel(): string;
export function getSources(): SourceConfig[];
```

---

### 12. src/config/sources/ai-openai.yaml
**Purpose**: Source registry for the "AI / OpenAI" topic. Defines RSS feeds to monitor.

Create this exact content:
```yaml
topic:
  slug: ai-openai
  name: AI & OpenAI
  description: Artificial Intelligence news from OpenAI and related AI sources
  keywords: ["AI", "artificial intelligence", "OpenAI", "GPT", "language model", "LLM"]
  active: true

sources:
  # NOTE: OpenAI removed the legacy /blog, /research, and /newsroom RSS feeds
  # during a 2024 site redesign. The current canonical feed lives at
  # openai.com/news/rss.xml. See registries/openai-codex.yaml for verified URLs.
  - id: openai-news
    name: OpenAI News
    url: https://openai.com/news
    feed_url: https://openai.com/news/rss.xml
    type: rss_feed
    adapter: rss
    priority: 9
    rate_limit: 30
    enabled: true
```

---

### 13. src/config/sources/tech-microsoft.yaml
**Purpose**: Source registry for the "Tech / Microsoft" topic.

Create this exact content:
```yaml
topic:
  slug: tech-microsoft
  name: Tech & Microsoft
  description: Microsoft technology news, Azure, Windows, and developer updates
  keywords: ["Microsoft", "Azure", "Windows", "tech", "cloud", "developer"]
  active: true

sources:
  - id: microsoft-devblog
    name: Microsoft Developer Blog
    url: https://devblogs.microsoft.com
    feed_url: https://devblogs.microsoft.com/feed
    type: rss_feed
    adapter: rss
    priority: 7
    rate_limit: 30
    enabled: true

  - id: microsoft-azure
    name: Microsoft Azure Blog
    url: https://azure.microsoft.com/blog
    feed_url: https://azure.microsoft.com/blog/feed/
    type: rss_feed
    adapter: rss
    priority: 7
    rate_limit: 30
    enabled: true

  - id: microsoft-techcommunity
    name: Microsoft Tech Community
    url: https://techcommunity.microsoft.com
    feed_url: https://techcommunity.microsoft.com/gxcuf89792/rss/board?board.id=MicrosoftDeveloperBlog
    type: rss_feed
    adapter: rss
    priority: 6
    rate_limit: 20
    enabled: true
```

---

### 14. src/types/index.ts
**Purpose**: Shared TypeScript types and Zod schemas used across the application.

Create these exact exports:

```typescript
import { z } from 'zod';

// ── Source Configuration ──────────────────────────────────────

export const SourceConfigSchema = z.object({
  id: z.string().regex(/^[a-z0-9-]+$/),
  name: z.string().min(1),
  url: z.string().url(),
  feed_url: z.string().url().optional(),
  type: z.enum(['blog', 'rss_feed', 'api', 'github', 'youtube', 'newsletter', 'news_site', 'forum', 'docs', 'changelog', 'press_room', 'sitemap', 'search']),
  adapter: z.enum(['rss', 'html', 'api', 'github', 'youtube', 'browser', 'sitemap']),
  topic: z.string(),
  priority: z.number().min(1).max(10).default(5),
  rate_limit: z.number().min(1).max(1000).default(10),
  enabled: z.boolean().default(true),
  selectors: z.record(z.string()).optional(),
  headers: z.record(z.string()).optional(),
});

export type SourceConfig = z.infer<typeof SourceConfigSchema>;

// ── Raw Feed Item (output from adapters) ─────────────────────

export const RawFeedItemSchema = z.object({
  title: z.string().min(1),
  url: z.string().url(),
  content: z.string().optional(),
  summary: z.string().optional(),
  author: z.string().optional(),
  publishedAt: z.string().datetime().optional(),
  sourceId: z.string(),
  sourceName: z.string(),
  topic: z.string(),
});

export type RawFeedItem = z.infer<typeof RawFeedItemSchema>;

// ── API Request/Response Schemas ─────────────────────────────

export const ListSourcesQuerySchema = z.object({
  topic: z.string().optional(),
  active: z.boolean().optional().default(true),
  page: z.number().int().min(1).optional().default(1),
  limit: z.number().int().min(1).max(100).optional().default(20),
});

export const CreateSourceBodySchema = z.object({
  name: z.string().min(1).max(200),
  url: z.string().url(),
  feed_url: z.string().url().optional(),
  adapter: z.enum(['rss', 'html', 'api', 'github', 'youtube']),
  topic: z.string(),
  priority: z.number().min(1).max(10).optional().default(5),
});

export const ListFeedQuerySchema = z.object({
  topic: z.string().optional(),
  since: z.string().datetime().optional(),
  page: z.number().int().min(1).optional().default(1),
  limit: z.number().int().min(1).max(100).optional().default(20),
});

// ── Scrape Job Data ──────────────────────────────────────────

export interface ScrapeJobData {
  sourceId: string;
  sourceUrl: string;
  sourceName: string;
  adapterType: string;
  topic: string;
  priority: number;
  feedUrl?: string;
}

// ── Scrape Result ────────────────────────────────────────────

export interface ScrapeResult {
  itemsScraped: number;
  itemsNew: number;
  itemsDuplicate: number;
  errors: string[];
  durationMs: number;
}

// ── App Config ───────────────────────────────────────────────

export interface AppConfig {
  nodeEnv: string;
  port: number;
  logLevel: string;
  databaseUrl: string;
  redisUrl: string;
  rateLimitRps: number;
  maxRetries: number;
  scrapeIntervalMinutes: number;
  enableSwagger: boolean;
  sources: SourceConfig[];
}
```

---

### 15. src/adapters/base.adapter.ts
**Purpose**: Abstract base class that all adapters extend. Provides shared logging, rate limiting, and retry logic.

Requirements:
- Abstract class `BaseAdapter` with constructor taking `AdapterConfig` and `Logger`
- Abstract method `fetch(): Promise<RawFeedItem[]>` that subclasses must implement
- Concrete method `execute()` that wraps fetch with rate limiting and retry
- Rate limiting: use TokenBucketRateLimiter (from utils/rate-limiter.ts) -- check limit before fetch, wait if needed
- Retry logic: exponential backoff with jitter, max 5 retries, circuit breaker after 10 consecutive failures
- Logging: log all operations with sourceId, adapter type, duration
- Graceful error handling: catch errors, log them, return empty array (never crash the worker)

```typescript
export interface AdapterConfig {
  id: string;
  name: string;
  url: string;
  feed_url?: string;
  topic: string;
  priority: number;
  rate_limit: number;
  selectors?: Record<string, string>;
  headers?: Record<string, string>;
}

export abstract class BaseAdapter {
  protected config: AdapterConfig;
  protected logger: Logger;

  constructor(config: AdapterConfig, logger: Logger);
  abstract fetch(): Promise<RawFeedItem[]>;
  async execute(): Promise<RawFeedItem[]>;
  protected withRetry<T>(fn: () => Promise<T>, context: string): Promise<T>;
}
```

---

### 16. src/adapters/rss.adapter.ts
**Purpose**: Fetch and parse RSS/Atom feeds. Primary parser: feedparser. Fallback: rss-parser.

Requirements:
- Class `RssAdapter extends BaseAdapter`
- `fetch()` method:
  1. Determine the URL to fetch: use `config.feed_url` if available, else `config.url`
  2. HTTP GET the feed URL with axios: timeout 15s, max 5 redirects, User-Agent header from the canonical UA constant defined in security.md (`JarvisHQ-Bot/1.0 (+https://example.invalid/about-bot; contact=operator@example.invalid; purpose=personal-news-aggregation; respects-robots-txt=true)`). Operators must replace the `example.invalid` placeholders with their real about-bot URL and contact address before deployment.
  3. **Primary parse**: Pass response body through `feedparser` stream parser
     - Handle RSS 0.9x, 1.0, 2.0, and Atom formats
     - Extract per item: title, link (as url), description (as content/summary), author, pubdate (as publishedAt), guid
  4. **Fallback parse**: If feedparser fails, retry with `rss-parser`
     - rss-parser handles malformed feeds better
  5. Normalize each parsed item into `RawFeedItem` with sourceId, sourceName, topic from config
  6. Normalize dates to ISO 8601 UTC strings (handle RSS pubDate, Atom updated, Dublin Core dates)
  7. Resolve relative URLs against the feed URL
  8. Return array of RawFeedItem
- Handle edge cases: empty feeds → return [], HTTP errors → log and return [], XML parse errors → try fallback parser
- Private helper methods: `parseWithFeedParser(xml: string)`, `parseWithRssParser(xml: string)`, `normalizeItem(item: any)`, `normalizeDate(date: any): string | undefined`

```typescript
export class RssAdapter extends BaseAdapter {
  async fetch(): Promise<RawFeedItem[]>;
  private async parseWithFeedParser(xml: string): Promise<RawFeedItem[]>;
  private async parseWithRssParser(xml: string): Promise<RawFeedItem[]>;
  private normalizeItem(item: any): RawFeedItem;
  private normalizeDate(date: any): string | undefined;
  private resolveRelativeUrl(url: string, baseUrl: string): string;
}
```

---

### 17. src/adapters/index.ts
**Purpose**: Adapter factory that creates the right adapter instance for a given source config.

Requirements:
- Export `ADAPTER_REGISTRY` mapping adapter names to classes
- Export `createAdapter(config: AdapterConfig, logger: Logger): BaseAdapter` factory function
- For Phase 1, only `rss` adapter is registered
- Throw clear error if unknown adapter type requested
- Future adapters (html, api, github, youtube, browser) will be registered in later phases

```typescript
export const ADAPTER_REGISTRY: Record<string, new (config: AdapterConfig, logger: Logger) => BaseAdapter>;
export function createAdapter(config: AdapterConfig, logger: Logger): BaseAdapter;
```

---

### 18. src/core/scheduler.ts
**Purpose**: Periodically trigger scrape jobs for all configured sources using node-cron.

Requirements:
- Class `SchedulerService`
- Constructor takes `Queue` (BullMQ), `SourceConfig[]`, and `Logger`
- `start()` method:
  - For each enabled source, create a cron job (default: every 30 minutes)
  - On each tick, enqueue a scrape job to BullMQ with source metadata
  - Log each scheduled job
- `stop()` method: stop all cron tasks
- `triggerManual(sourceId: string)` method: immediately enqueue a scrape job for a specific source
- Compute job priority from source priority (1 = highest, 10 = lowest)
- Skip disabled sources

```typescript
export class SchedulerService {
  constructor(queue: Queue, sources: SourceConfig[], logger: Logger);
  start(): void;
  stop(): void;
  async triggerManual(sourceId: string): Promise<string>;
  private computePriority(sourcePriority: number): number;
  private enqueueSource(source: SourceConfig): Promise<Job>;
}
```

---

### 19. src/core/queue.ts
**Purpose**: Initialize BullMQ queue and workers for scraping.

Requirements:
- Create and export `scrapeQueue = new Queue('scrape', { connection: redisConnection })`
- Create and export `ScrapeWorker = new Worker('scrape', processor, options)`
- Worker processor function:
  1. Extract job data (ScrapeJobData)
  2. Create the appropriate adapter via `createAdapter()`
  3. Call `adapter.execute()` to fetch items
  4. Store each item in the database via `FeedItem.query().insert()` with ON CONFLICT (url) DO NOTHING
  5. Update source `last_scraped` timestamp
  6. Return ScrapeResult with counts
- Worker options: concurrency 3, Redis connection
- Handle worker events: `completed` (log success), `failed` (log error with retry count)
- Export function `closeQueue()` for graceful shutdown
- QueueScheduler for delayed retries (stalled job recovery)

```typescript
export const scrapeQueue: Queue;
export const ScrapeWorker: Worker;
export async function closeQueue(): Promise<void>;
export async function processScrapeJob(job: Job<ScrapeJobData>): Promise<ScrapeResult>;
```

---

### 20. src/core/collector.ts
**Purpose**: Orchestrates scraping by loading active sources and enqueuing jobs.

Requirements:
- Class `Collector`
- Constructor takes `Queue`, Knex instance, and `Logger`
- `collectAll()` method:
  - Load all sources from YAML configs that are enabled
  - For each source, enqueue a scrape job
  - Return counts: `{ enqueued: number, skipped: number }`
- `collectTopic(topicSlug: string)` method:
  - Filter sources by topic slug
  - Enqueue scrape jobs for matching sources
- `collectSource(sourceId: string)` method:
  - Find source by ID, enqueue single job
- Use job options: `removeOnComplete: 100`, `removeOnFail: 50` to keep queue clean

```typescript
export class Collector {
  constructor(queue: Queue, knex: Knex, logger: Logger);
  async collectAll(): Promise<{ enqueued: number; skipped: number }>;
  async collectTopic(topicSlug: string): Promise<{ enqueued: number }>;
  async collectSource(sourceId: string): Promise<void>;
  private loadActiveSources(): Promise<SourceConfig[]>;
}
```

---

### 21. src/models/base.model.ts
**Purpose**: Base Objection.js model with automatic timestamp handling.

Requirements:
- Class `BaseModel extends Model` from objection
- Auto-set `created_at` and `updated_at` on insert via `$beforeInsert()`
- Auto-set `updated_at` on update via `$beforeUpdate()`
- All timestamps as ISO 8601 strings

```typescript
import { Model } from 'objection';

export class BaseModel extends Model {
  id!: string;
  created_at!: string;
  updated_at!: string;

  $beforeInsert(): void;
  $beforeUpdate(): void;
}
```

---

### 22. src/models/source.model.ts
**Purpose**: Objection model for the `sources` table.

Requirements:
- Class `Source extends BaseModel`
- `static tableName = 'sources'`
- Define relation: `hasMany('FeedItem')` via `feedItems` relation mapping
- Static methods:
  - `findActive()`: return all sources where status='active', ordered by priority DESC
  - `findByTopic(topic: string)`: return sources for a topic
  - `findById(id: string)`: return single source by ID
- Instance methods:
  - `markFetched()`: update last_scraped to now
  - `updateHealth(score: number)`: update quality_score

```typescript
export class Source extends BaseModel {
  static tableName = 'sources';
  static relationMappings: RelationMappings;
  static async findActive(): Promise<Source[]>;
  static async findByTopic(topic: string): Promise<Source[]>;
  static async findById(id: string): Promise<Source | undefined>;
  async markFetched(): Promise<void>;
  async updateHealth(score: number): Promise<void>;
}
```

---

### 23. src/models/feed-item.model.ts
**Purpose**: Objection model for the `feed_items` table.

Requirements:
- Class `FeedItem extends BaseModel`
- `static tableName = 'feed_items'`
- Define relation: `belongsTo('Source')` via `source` relation mapping
- Static methods:
  - `findRecent(limit: number, offset?: number)`: return items ordered by published_at DESC
  - `findByTopic(topic: string, limit: number)`: filter by topic
  - `findById(id: string)`: return single item
  - `existsByUrl(url: string)`: check if URL already exists (for dedup)
- Instance method `calculateQualityScore()`: placeholder scoring (returns default 50.0 for Phase 1)

```typescript
export class FeedItem extends BaseModel {
  static tableName = 'feed_items';
  static relationMappings: RelationMappings;
  static async findRecent(limit: number, offset?: number): Promise<FeedItem[]>;
  static async findByTopic(topic: string, limit: number): Promise<FeedItem[]>;
  static async findById(id: string): Promise<FeedItem | undefined>;
  static async existsByUrl(url: string): Promise<boolean>;
  calculateQualityScore(): number; // Phase 1 placeholder
}
```

---

### 24. src/api/routes/sources.ts
**Purpose**: CRUD routes for managing sources.

Requirements:
- Default export async function that takes FastifyInstance
- Register these routes:
  - `GET /sources` -- list all sources with pagination (query: page, limit, topic). Returns { sources: Source[], total: number, page: number, limit: number }
  - `POST /sources` -- register a new source (body validated with CreateSourceBodySchema). Insert into DB, return 201 with created source
  - `GET /sources/:id` -- get single source by ID. Return 404 if not found
  - `DELETE /sources/:id` -- soft-delete a source (set status to 'paused'). Return 204
- Use Zod schemas from `../schemas` for validation
- Use Source model for database queries
- Log all requests

```typescript
export default async function sourceRoutes(fastify: FastifyInstance): Promise<void>;
```

---

### 25. src/api/routes/feed.ts
**Purpose**: Feed item retrieval routes.

Requirements:
- Default export async function that takes FastifyInstance
- Register these routes:
  - `GET /feed` -- list feed items with pagination and filters (query: page, limit, topic, since). Returns { items: FeedItem[], total: number, page: number, limit: number }
  - `GET /feed/:id` -- get single feed item by UUID with its source relation loaded. Return 404 if not found
  - `GET /feed/recent` -- shortcut for latest 20 items (ordered by published_at DESC)
- Use FeedItem model with eager loading of `source` relation
- Filter by topic if provided (case-insensitive match)
- Filter by `since` date if provided

```typescript
export default async function feedRoutes(fastify: FastifyInstance): Promise<void>;
```

---

### 26. src/api/routes/scrape.ts
**Purpose**: Trigger scraping jobs manually.

Requirements:
- Default export async function that takes FastifyInstance
- Register these routes:
  - `POST /scrape` -- trigger a full scrape of all sources. Uses Collector.collectAll(). Returns { message: "Scrape initiated", enqueued: number }
  - `POST /scrape/:topic` -- trigger scrape for a specific topic. Uses Collector.collectTopic(). Returns { message: "Topic scrape initiated", topic: string, enqueued: number }
  - `POST /scrape/source/:id` -- trigger scrape for a single source. Uses Collector.collectSource(). Returns { message: "Source scrape initiated", sourceId: string }
- All routes return 202 Accepted (scrape is async, runs via queue)
- Validate topic slug and source ID parameters

```typescript
export default async function scrapeRoutes(fastify: FastifyInstance): Promise<void>;
```

---

### 27. src/api/routes/health.ts
**Purpose**: Health check endpoint for Docker and monitoring.

Requirements:
- Default export async function that takes FastifyInstance
- Register:
  - `GET /health` -- check DB connectivity (simple SELECT 1) and Redis connectivity (PING). Return `{ status: "ok" | "degraded", checks: { db: boolean, redis: boolean }, timestamp: ISO8601 }`. HTTP 200 if both OK, 503 if any check fails
  - `GET /health/db` -- DB only check
  - `GET /health/redis` -- Redis only check
- Fast response (< 100ms), no auth required

```typescript
export default async function healthRoutes(fastify: FastifyInstance): Promise<void>;
```

---

### 28. src/api/schemas.ts
**Purpose**: Zod schemas for request validation and response typing.

Requirements:
- Re-export all schemas from `../types/index.ts`
- Add API-specific schemas:
  - `ScrapeResponseSchema = z.object({ message: z.string(), enqueued: z.number(), topic: z.string().optional(), sourceId: z.string().optional() })`
  - `HealthResponseSchema = z.object({ status: z.string(), checks: z.object({ db: z.boolean(), redis: z.boolean() }), timestamp: z.string() })`
  - `ErrorResponseSchema = z.object({ success: z.literal(false), error: z.object({ code: z.string(), message: z.string() }) })`
- Export inferred TypeScript types from all schemas

---

### 29. src/api/errors.ts
**Purpose**: Centralized error handling for the API layer.

Requirements:
- Custom error classes: `ValidationError`, `NotFoundError`, `DatabaseError`
- Fastify error handler function:
  - Zod validation errors → 400 Bad Request with field-level error details
  - NotFoundError → 404
  - DatabaseError → 500
  - Generic errors → 500
- Consistent error response format: `{ success: false, error: { code, message } }`
- Log all errors with structured logging (include stack trace in development)

```typescript
export class ValidationError extends Error { code: string; details: any; }
export class NotFoundError extends Error { code: string; resource: string; }
export class DatabaseError extends Error { code: string; cause?: Error; }
export function registerErrorHandler(fastify: FastifyInstance): void;
```

---

### 30. src/utils/logger.ts
**Purpose**: Pino logger factory for structured JSON logging.

Requirements:
- Export `createLogger(name: string)` returning a Pino child logger
- Export `defaultLogger` as the root logger
- Log level from env (default: info)
- In development: use pino-pretty for readable output
- In production: raw JSON to stdout
- Redact sensitive fields: `*.apiKey`, `*.token`, `*.password`, `headers.authorization`
- Every log entry includes: service name, version, timestamp

```typescript
import { pino, Logger } from 'pino';
export function createLogger(name: string): Logger;
export const defaultLogger: Logger;
```

---

### 31. src/utils/rate-limiter.ts
**Purpose**: Token bucket rate limiter for per-source request throttling.

Requirements:
- Class `TokenBucketRateLimiter`
- Token bucket algorithm: bucket has capacity, refills at `rateLimit / 60` tokens per second
- `async checkLimit(key: string, capacity: number): Promise<void>` -- waits if bucket is empty
- `releaseToken(key: string): void` -- return a token (optional, for concurrency limiting)
- Use in-memory Map for bucket storage (Redis-backed in Phase 5)
- Thread-safe: use async/await for backpressure

```typescript
export class TokenBucketRateLimiter {
  constructor();
  async checkLimit(key: string, capacity: number): Promise<void>;
  releaseToken(key: string): void;
  private refill(key: string, capacity: number): void;
}
```

---

### 32. migrations/001_initial.sql
**Purpose**: Initial database schema. PostgreSQL-compatible.

> **Schema authority:** The canonical schema lives in
> [`../references/data-schema.md`](../references/data-schema.md). The DDL below
> is a *Phase-1 minimal subset* — enough to make `docker compose up` work and
> tests pass — not the full schema. When you implement Phase 2+ you should
> migrate toward the canonical schema (UUIDv7 keys, JSONB columns, dedicated
> `scrape_jobs` / `raw_data` / `audit_log` tables). Treat any disagreement
> between this SQL and `data-schema.md` as: `data-schema.md` wins.

Create this EXACT SQL:

```sql
-- ============================================================
-- Migration: 001_initial
-- Phase 1 schema: sources + feed_items tables
-- ============================================================

-- Sources table: registered news sources
CREATE TABLE IF NOT EXISTS sources (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    url             TEXT NOT NULL,
    feed_url        TEXT,
    type            TEXT NOT NULL,
    adapter         TEXT NOT NULL DEFAULT 'rss',
    topic           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'paused', 'error', 'pending_verification', 'rejected')),
    priority        INTEGER NOT NULL DEFAULT 5 CHECK (priority BETWEEN 1 AND 10),
    rate_limit      INTEGER NOT NULL DEFAULT 10 CHECK (rate_limit BETWEEN 1 AND 1000),
    quality_score   REAL CHECK (quality_score BETWEEN 0 AND 100),
    quality_tier    INTEGER CHECK (quality_tier BETWEEN 1 AND 5),
    last_scraped    TIMESTAMPTZ,
    last_success    TIMESTAMPTZ,
    last_error      TEXT,
    error_count     INTEGER NOT NULL DEFAULT 0,
    success_count   INTEGER NOT NULL DEFAULT 0,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    selectors       JSONB,
    headers         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Feed items table: scraped articles from all sources
CREATE TABLE IF NOT EXISTS feed_items (
    id              TEXT PRIMARY KEY,
    source_id       TEXT NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    topic           TEXT NOT NULL,
    title           TEXT NOT NULL,
    url             TEXT NOT NULL UNIQUE,
    summary         TEXT,
    content         TEXT,
    author          TEXT,
    published_at    TIMESTAMPTZ,
    scraped_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    quality_score   REAL DEFAULT 50.0 CHECK (quality_score BETWEEN 0 AND 100),
    quality_tier    INTEGER DEFAULT 3 CHECK (quality_tier BETWEEN 1 AND 5),
    is_duplicate    BOOLEAN NOT NULL DEFAULT false,
    raw_metadata    JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes for common query patterns
CREATE INDEX IF NOT EXISTS idx_sources_topic ON sources(topic);
CREATE INDEX IF NOT EXISTS idx_sources_status ON sources(status);
CREATE INDEX IF NOT EXISTS idx_sources_type ON sources(type);
CREATE INDEX IF NOT EXISTS idx_sources_enabled ON sources(enabled);
CREATE INDEX IF NOT EXISTS idx_sources_priority ON sources(priority DESC);

CREATE INDEX IF NOT EXISTS idx_feed_items_source_id ON feed_items(source_id);
CREATE INDEX IF NOT EXISTS idx_feed_items_topic ON feed_items(topic);
CREATE INDEX IF NOT EXISTS idx_feed_items_published_at ON feed_items(published_at DESC);
CREATE INDEX IF NOT EXISTS idx_feed_items_scraped_at ON feed_items(scraped_at DESC);
CREATE INDEX IF NOT EXISTS idx_feed_items_url ON feed_items(url);

-- Trigger function to auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply trigger to both tables
CREATE TRIGGER update_sources_updated_at BEFORE UPDATE ON sources
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_feed_items_updated_at BEFORE UPDATE ON feed_items
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

### 33. seeds/dev-seeds.ts
**Purpose**: Development seed data for local testing.

Requirements:
- Knex seed file that inserts 2 test sources and 5 sample feed items
- Use the ai-openai and tech-microsoft topics
- Sample items should have realistic titles/URLs but clearly be test data (use example.com domains)
- Feed items reference the seeded sources via source_id

---

### 34. tests/fixtures/rss/valid-rss-2.0.xml
**Purpose**: Valid RSS 2.0 feed for testing.

Create this EXACT content:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>OpenAI Blog</title>
    <link>https://openai.com/blog</link>
    <description>Latest news and research from OpenAI</description>
    <language>en</language>
    <lastBuildDate>Mon, 15 Jan 2025 10:00:00 GMT</lastBuildDate>
    <item>
      <title>GPT-5: The Next Generation of Language Models</title>
      <link>https://openai.com/blog/gpt-5</link>
      <description>OpenAI announces GPT-5, featuring breakthrough reasoning capabilities and multimodal understanding across text, images, and video.</description>
      <author>research@openai.com (OpenAI Research Team)</author>
      <pubDate>Mon, 15 Jan 2025 09:00:00 GMT</pubDate>
      <guid>https://openai.com/blog/gpt-5</guid>
    </item>
    <item>
      <title>Advancing AI Safety Through Constitutional AI</title>
      <link>https://openai.com/blog/constitutional-ai-safety</link>
      <description>New research on aligning AI systems with human values through constitutional principles and reinforcement learning.</description>
      <author>safety@openai.com (OpenAI Safety Team)</author>
      <pubDate>Mon, 15 Jan 2025 08:30:00 GMT</pubDate>
      <guid>https://openai.com/blog/constitutional-ai-safety</guid>
    </item>
    <item>
      <title>OpenAI API: New Features for Developers</title>
      <link>https://openai.com/blog/api-updates-jan-2025</link>
      <description>Introducing streaming improvements, function calling enhancements, and reduced latency in the latest API update.</description>
      <author>dev@openai.com (OpenAI Developer Relations)</author>
      <pubDate>Sun, 14 Jan 2025 16:00:00 GMT</pubDate>
      <guid>https://openai.com/blog/api-updates-jan-2025</guid>
    </item>
  </channel>
</rss>
```

---

### 35. tests/fixtures/rss/valid-atom.xml
**Purpose**: Valid Atom 1.0 feed for testing.

Create this EXACT content:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>Microsoft Developer Blog</title>
  <link href="https://devblogs.microsoft.com"/>
  <updated>2025-01-15T10:00:00Z</updated>
  <author>
    <name>Microsoft Developer Team</name>
  </author>
  <id>urn:uuid:devblogs-microsoft-com</id>
  <entry>
    <title>Azure AI Studio: Building Intelligent Applications</title>
    <link href="https://devblogs.microsoft.com/azure-ai-studio"/>
    <id>urn:uuid:azure-ai-studio-2025-01</id>
    <updated>2025-01-15T09:00:00Z</updated>
    <summary>A comprehensive guide to building intelligent applications with Azure AI Studio, including model deployment and prompt engineering.</summary>
    <author>
      <name>Azure AI Team</name>
    </author>
  </entry>
  <entry>
    <title>.NET 9 Performance Improvements</title>
    <link href="https://devblogs.microsoft.com/dotnet-9-performance"/>
    <id>urn:uuid:dotnet-9-perf-2025-01</id>
    <updated>2025-01-14T14:00:00Z</updated>
    <summary>Deep dive into the performance optimizations in .NET 9, including JIT compilation improvements and memory management enhancements.</summary>
    <author>
      <name>.NET Team</name>
    </author>
  </entry>
</feed>
```

---

### 36. tests/fixtures/rss/malformed.xml
**Purpose**: Malformed XML feed to test parser fallback behavior.

Create this EXACT content:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>Broken Feed</title>
    <link>https://example.com</link>
    <description>A malformed feed for testing
    <!-- Missing closing tags intentionally -->
    <item>
      <title>Valid Item in Broken Feed</title>
      <link>https://example.com/valid-item</link>
    </item>
  </channel>
</rss>
```

---

### 37. tests/fixtures/rss/empty.xml
**Purpose**: Empty but valid RSS feed.

Create this EXACT content:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>Empty Feed</title>
    <link>https://example.com</link>
    <description>An empty feed for testing</description>
    <language>en</language>
  </channel>
</rss>
```

---

### 38. tests/unit/adapters/rss.adapter.test.ts
**Purpose**: Unit tests for the RSS adapter with fixture-based testing.

Requirements -- implement ALL of these test cases:

```typescript
describe('RssAdapter', () => {
  // Test 1: Parse valid RSS 2.0 feed from fixture
  it('should parse valid RSS 2.0 feed and return all items', async () => {
    // Mock axios to return the valid-rss-2.0.xml fixture content
    // Parse and expect 3 items with correct titles, URLs, dates
  });

  // Test 2: Parse valid Atom feed from fixture
  it('should parse valid Atom 1.0 feed and return all items', async () => {
    // Mock axios to return the valid-atom.xml fixture content
    // Expect 2 items with correct structure
  });

  // Test 3: Handle empty feed gracefully
  it('should return empty array for empty feed', async () => {
    // Mock axios to return the empty.xml fixture
    // Expect empty array, no errors
  });

  // Test 4: Handle malformed XML with fallback parser
  it('should fallback to rss-parser when feedparser fails on malformed XML', async () => {
    // Mock axios to return the malformed.xml fixture
    // Expect rss-parser to successfully extract at least 1 item
  });

  // Test 5: Handle HTTP 404 error
  it('should return empty array and log error on HTTP 404', async () => {
    // Mock axios to throw 404
    // Expect empty array, error logged
  });

  // Test 6: Handle HTTP timeout
  it('should handle request timeout gracefully', async () => {
    // Mock axios to timeout
    // Expect empty array after retries exhausted
  });

  // Test 7: Normalize dates to ISO 8601
  it('should normalize RSS pubDate to ISO 8601 format', async () => {
    // Parse feed with known dates
    // Expect publishedAt to be ISO 8601 string
  });

  // Test 8: Resolve relative URLs
  it('should resolve relative URLs against feed URL', async () => {
    // Feed with relative /article/123 URLs
    // Expect absolute URLs with base feed URL
  });

  // Test 9: Include source metadata
  it('should include sourceId, sourceName, and topic in each item', async () => {
    // Parse and verify every item has source metadata
  });

  // Test 10: Handle missing optional fields
  it('should handle items missing author or description', async () => {
    // Feed with minimal items (title + link only)
    // Expect items with undefined optional fields
  });
});
```

Use Vitest (`describe`, `it`, `expect`, `vi.fn()` for mocking). Mock axios with `vi.mock('axios')`. Load fixture files using Node's `fs.readFileSync`.

---

### 39. tests/integration/api.test.ts
**Purpose**: Integration tests for API endpoints.

Requirements -- implement ALL of these test cases:

```typescript
describe('API Integration Tests', () => {
  // Setup: start Fastify app, seed test data, use test database

  it('GET /health should return 200 with ok status', async () => {
    // Request /health
    // Expect status 200, body.status === 'ok', checks.db === true
  });

  it('GET /sources should return list of sources', async () => {
    // Seed 2 sources
    // GET /sources
    // Expect 200, array of 2 sources, each has id, name, url, topic
  });

  it('GET /sources?topic=ai-openai should filter by topic', async () => {
    // Seed sources for different topics
    // GET /sources?topic=ai-openai
    // Expect only sources matching the topic
  });

  it('GET /sources/:id should return single source', async () => {
    // Seed a source
    // GET /sources/{id}
    // Expect 200 with correct source data
  });

  it('GET /sources/:id should return 404 for unknown id', async () => {
    // GET /sources/nonexistent-id
    // Expect 404
  });

  it('POST /sources should create a new source', async () => {
    // POST /sources with valid body
    // Expect 201, response contains created source with generated id
    // Verify source exists in database
  });

  it('POST /sources should return 400 for invalid body', async () => {
    // POST /sources with missing required field (e.g., no name)
    // Expect 400 with validation error details
  });

  it('GET /feed should return feed items', async () => {
    // Seed 5 feed items
    // GET /feed
    // Expect 200, array of items, has pagination metadata
  });

  it('GET /feed?topic=ai-openai should filter by topic', async () => {
    // Seed items for multiple topics
    // GET /feed?topic=ai-openai
    // Expect only items for that topic
  });

  it('GET /feed/:id should return single feed item', async () => {
    // Seed an item
    // GET /feed/{id}
    // Expect 200 with item data including source relation
  });

  it('GET /feed/recent should return latest 20 items', async () => {
    // Seed 25 items with different dates
    // GET /feed/recent
    // Expect exactly 20 items, ordered by published_at DESC
  });

  it('POST /scrape should return 202 and enqueue jobs', async () => {
    // POST /scrape
    // Expect 202, response has enqueued count
  });

  it('POST /scrape/:topic should enqueue topic-specific jobs', async () => {
    // POST /scrape/ai-openai
    // Expect 202, response has topic and enqueued count
  });
});
```

Use a test database (SQLite in-memory or separate test Postgres). Clean up data between tests. Use `beforeAll` to start app, `afterAll` to close, `beforeEach` to reset data.

---

### 40b. src/utils/robots.ts
**Purpose**: Fetch and parse `robots.txt` for each domain; enforce compliance before every outbound HTTP request.

Requirements:
- Export class `RobotsCompliance`
- Constructor takes a `Cache` (in-memory `Map<domain, ParsedRobots>` is fine for Phase 1) and the canonical bot User-Agent token (e.g., `JarvisHQ-Bot`)
- `async isAllowed(url: string): Promise<boolean>` — derives the domain, fetches `https://<domain>/robots.txt` if not cached or cache is older than 24h, parses it with `robots-parser`, and returns whether the configured User-Agent token is allowed to fetch the path. On HTTP 404 for robots.txt, default-allow.
- `async getCrawlDelayMs(domain: string): Promise<number>` — returns the per-domain crawl delay (Crawl-delay directive, in ms). Returns 0 if not specified.
- Cache TTL: 24h. Stale entries refresh on next access.
- Honest User-Agent: send the canonical UA string when fetching robots.txt itself.
- Failure mode: on network error while fetching robots.txt, return `false` (default-deny) and log a `compliance.robots_fetch_failed` event so the operator notices. Do not silently allow.

```typescript
export class RobotsCompliance {
  constructor(cache: Map<string, ParsedRobots>, userAgentToken: string, logger: Logger);
  async isAllowed(url: string): Promise<boolean>;
  async getCrawlDelayMs(domain: string): Promise<number>;
  private async fetchAndParse(domain: string): Promise<ParsedRobots>;
}
```

**Integration**: `BaseAdapter.execute()` must call `await this.robotsCompliance.isAllowed(this.config.url)` (and the per-item URL for HTML adapters in later phases) before fetching. The rate limiter must take `max(configured_rate_limit_delay, getCrawlDelayMs(domain))` as its effective per-request delay — the most-restrictive limit wins.

---

### 40. src/cli/sync-sources.ts
**Purpose**: CLI command to load YAML source registries and sync to database.

Requirements:
- Read all YAML files from `src/config/sources/*.yaml`
- For each source: upsert into database (insert if new, update if exists)
- Set status to 'active' for existing sources being re-synced
- Log summary: "Synced N sources from M topic files"
- Exit with code 0 on success, 1 on failure
- Can be run via `npm run sources:sync`

```typescript
// CLI script, run directly
// Usage: tsx src/cli/sync-sources.ts
```

---

## Verification Criteria (Done Checklist)

After implementation, the following must ALL pass. Verify each (the agent that ran this prompt should self-check, and the operator should re-verify before merging):

- [ ] `docker-compose up -d` starts all 3 services without errors
- [ ] `docker-compose ps` shows all 3 services as healthy
- [ ] `npm install` completes without errors
- [ ] `npm run build` compiles TypeScript without errors
- [ ] `npm test` passes all tests (run in Docker via `docker-compose exec app npm test`)
- [ ] `curl http://localhost:3000/health` returns `{"status":"ok","checks":{"db":true,"redis":true},"timestamp":"..."}` with HTTP 200
- [ ] `curl http://localhost:3000/sources` returns JSON array with 6+ registered sources (from the 2 YAML files)
- [ ] `curl http://localhost:3000/sources?topic=ai-openai` returns only OpenAI sources
- [ ] `curl http://localhost:3000/feed` returns feed items (may be empty initially, returns `{ items: [], total: 0 }`)
- [ ] `curl -X POST http://localhost:3000/scrape` returns `202` with `{ message: "Scrape initiated", enqueued: N }`
- [ ] After POST /scrape, `curl http://localhost:3000/feed` returns items parsed from RSS feeds
- [ ] All 2 topics (ai-openai, tech-microsoft) have sources registered
- [ ] `docker-compose logs app` shows structured JSON logs with no unhandled errors
- [ ] `npm run test:coverage` shows >= 80% statement coverage
- [ ] Robots.txt enforcement test: with a mock domain whose `robots.txt` returns `Disallow: /`, `POST /scrape` for a source in that domain MUST result in zero feed items fetched and a `compliance.robots_blocked` log entry
- [ ] Crawl-delay test: with a mock `Crawl-delay: 5` in robots.txt, repeated requests to the same domain MUST be spaced at least 5 seconds apart even if the per-source rate_limit is configured tighter

## Implementation Notes

1. **Phase 1 only** -- Do not implement any features beyond what's listed. No HTML extraction, no Telegram, no n8n, no deduplication, no summarisation.

2. **Database** -- Use Knex migrations (SQL file, not JS) for simplicity. Run migrations on app startup. The `001_initial.sql` uses PostgreSQL syntax (TIMESTAMPTZ, JSONB, plpgsql triggers).

3. **Objection.js setup** -- Bind Knex to Objection's Model class in `src/index.ts` before starting the server: `Model.knex(knexInstance)`.

4. **BullMQ connection** -- Use ioredis for the Redis connection. The connection string comes from `REDIS_URL` env var.

5. **Graceful shutdown** -- On SIGTERM: stop Fastify from accepting new connections, wait for active BullMQ jobs to complete (with timeout), close Knex pool, close Redis connection, exit process.

6. **Error handling** -- The RSS adapter must never throw uncaught exceptions. All errors are caught, logged, and result in an empty array return. This prevents worker crashes.

7. **Quality scoring** -- For Phase 1, quality scoring is a placeholder. `FeedItem.calculateQualityScore()` returns 50.0 (default). Full scoring comes in Phase 3.

8. **Feed items dedup** -- Use `ON CONFLICT (url) DO NOTHING` in the insert query to silently skip duplicates based on URL.

9. **Date handling** -- RSS pubDate can be in various formats (RFC 822, ISO 8601, relative). Normalize everything to ISO 8601 UTC strings. Use `new Date().toISOString()` as fallback.

10. **Environment** -- The app must work in 3 environments: development (tsx watch), test (in-memory or test DB), production (Docker). Use `.env` file for local dev, env vars in Docker.
```

---

*End of Phase 1 build prompt. This document is self-contained; paste it entirely into any code-generating agent (Codex, Claude Code, the Claude Agent SDK, Cursor, Aider, Kimi/OpenClaw, etc.) targeting a Node.js/TypeScript stack. For other stacks, treat it as a structured checklist.*
