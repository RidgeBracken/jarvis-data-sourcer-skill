# Section F: Architecture Plan — Jarvis HQ

## Table of Contents
1. [System Overview Diagram](#1-system-overview-diagram)
2. [Component Specifications](#2-component-specifications)
3. [Data Flow](#3-data-flow)
4. [Tech Stack Decision](#4-tech-stack-decision)
5. [Docker Architecture](#5-docker-architecture)
6. [Configuration](#6-configuration)
7. [Extensibility](#7-extensibility)

---

## 1. System Overview Diagram

### Full System Architecture (Text-Based)

```
+===============================================================================+
|                              JARVIS HQ                                        |
|                    Personal AI News Scraper System                            |
+===============================================================================+

┌─────────────────────────────────────────────────────────────────────────────┐
│                            SOURCE LAYER                                      │
│                                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │   RSS    │ │    API   │ │ Sitemap  │ │  GitHub  │ │     YouTube      │  │
│  │  Feeds   │ │ Endpoints│ │   XML    │ │ Releases │ │    Channels      │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────────┬─────────┘  │
│       │            │            │            │                │            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │  HTML    │ │  Social  │ │  Search  │ │  JSON    │ │    Other         │  │
│  │  Pages   │ │  Posts   │ │Discovery │ │  Feeds   │ │    Sources       │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CONTROL LAYER                                      │
│                                                                             │
│  ┌────────────────────┐    ┌────────────────────┐   ┌──────────────────┐  │
│  │  Source Registry   │    │     Scheduler      │   │   Job Queue      │  │
│  │  (YAML Configs)    │───▶│  (node-cron)       │──▶│  (Bull + Redis)  │  │
│  │                    │    │  Cron Expressions  │   │  Priority + DLQ  │  │
│  └────────────────────┘    └────────────────────┘   └──────────────────┘  │
└────────────────────────┬──────────────────────────────┬────────────────────┘
                         │                              │
                         │        ┌──────────────┐     │
                         │        │  Rate Limiter │◀────┘
                         │        │  (per-domain) │
                         │        └──────┬───────┘
                         │               │
                         ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PROCESSING LAYER                                    │
│                                                                             │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐       │
│  │          │          │          │          │          │          │       │
│  │  RSS     │   API    │ Sitemap  │  HTML    │ GitHub   │ YouTube  │       │
│  │ Adapter  │ Adapter  │ Adapter  │ Adapter  │ Adapter  │ Adapter  │       │
│  │          │          │          │          │          │          │       │
│  └────┬─────┘────┬─────┘────┬─────┘────┬─────┘────┬─────┘────┬─────┘       │
│       │          │          │          │          │          │              │
│       └──────────┴──────────┴──────────┼──────────┴──────────┴──────────────┘
│                                        │
│                                        ▼
│                          ┌─────────────────────────┐
│                          │     Retry Handler       │
│                          │  (exponential backoff)  │
│                          └───────────┬─────────────┘
│                                      │
│                                      ▼
│                          ┌─────────────────────────┐
│                          │      Extractors         │
│                          │ (cheerio/trafilatura)   │
│                          └───────────┬─────────────┘
│                                      │
│                                      ▼
│                          ┌─────────────────────────┐
│                          │     Quality Engine      │
│                          │  (scoring + dedup)      │
│                          └───────────┬─────────────┘
│                                      │
│                                      ▼
│                          ┌─────────────────────────┐
│                          │    Deduplication        │
│                          │    (hash-based)         │
│                          └───────────┬─────────────┘
└──────────────────────────────────────┼──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STORAGE LAYER                                      │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    PostgreSQL 16 (single container)                  │    │
│  │                                                                    │    │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │   sources   │  │    articles  │  │   raw_data   │               │    │
│  │  │   config    │  │    (parsed)  │  │  (caches)    │               │    │
│  │  └─────────────┘  └──────────────┘  └──────────────┘               │    │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │    jobs     │  │    errors    │  │   metrics    │               │    │
│  │  │   (queue)   │  │    (DLQ)     │  │  (analytics) │               │    │
│  │  └─────────────┘  └──────────────┘  └──────────────┘               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└────────────────────────┬────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API LAYER                                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              Fastify REST API (Node.js 20)                           │    │
│  │                                                                    │    │
│  │  GET    /api/articles              - List articles (paginated)     │    │
│  │  GET    /api/articles/:id          - Single article                │    │
│  │  GET    /api/articles/search       - Full-text search              │    │
│  │  GET    /api/articles/topic/:topic - Filter by topic               │    │
│  │  GET    /api/sources               - List all sources              │    │
│  │  GET    /api/jobs                  - Job status / queue state      │    │
│  │  GET    /api/metrics               - Scraping metrics              │    │
│  │  POST   /api/jobs/trigger          - Manual trigger                │    │
│  │  GET    /api/health                - Health check                  │    │
│  │  GET    /feed/rss                  - RSS output feed               │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
└──────────────────────────────────┼──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CONSUMER LAYER                                      │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────┐    │
│  │  Next.js News  │  │ Telegram Bot   │  │      n8n Webhooks          │    │
│  │  Website       │  │ (grammY)       │  │  (workflow automation)     │    │
│  │                │  │                │  │                            │    │
│  │  - Topic pages │  │  - Daily digest│  │  - New article triggers    │    │
│  │  - Search      │  │  - On-demand   │  │  - Zapier-style flows      │    │
│  │  - Admin panel │  │  - Topic subs  │  │  - Conditional routing     │    │
│  └────────────────┘  └────────────────┘  └────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Interaction Detail

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                              DETAIL FLOW                                            │
│                                                                                     │
│   Cron Tick (every N minutes)                                                       │
│       │                                                                             │
│       ▼                                                                             │
│   ┌──────────────┐   Reads     ┌──────────────┐                                    │
│   │  Scheduler   │◀────────────│   Source     │                                    │
│   │              │             │  Registry    │                                    │
│   └──────┬───────┘             │  (YAML)      │                                    │
│          │                      └──────────────┘                                    │
│          │ "Create job for source X                                                  │
│          │  with priority, adapter_type"                                             │
│          ▼                                                                          │
│   ┌──────────────┐                    ┌──────────────┐                             │
│   │  Job Queue   │───────────────────▶│ Rate Limiter │                             │
│   │  (Bull)      │ "Check domain      │  (per-domain │                             │
│   │              │  rate limit"       │  token bucket│                             │
│   └──────┬───────┘                    └──────┬───────┘                             │
│          │                                    │ "OK to proceed"                     │
│          │                                    ▼                                     │
│          │                           ┌──────────────┐                               │
│          │                           │   Adapter    │                               │
│          │                           │  Factory     │                               │
│          │                           └──────┬───────┘                               │
│          │                                  │ "Route to correct adapter"             │
│          │                    ┌─────────────┼─────────────┐                          │
│          │                    ▼             ▼             ▼                          │
│          │              ┌────────┐   ┌──────────┐  ┌──────────┐                     │
│          │              │  RSS   │   │   API    │  │  HTML    │                     │
│          │              │ Adapter│   │ Adapter  │  │ Adapter  │                     │
│          │              └────┬───┘   └────┬─────┘  └────┬─────┘                     │
│          │                   │            │             │                            │
│          │                   └────────────┴─────┬───────┘                            │
│          │                                      │ "Raw content"                       │
│          │                                      ▼                                    │
│          │                             ┌──────────────┐                              │
│          │                             │  Extractor   │                              │
│          │                             │  (parsing)   │                              │
│          │                             └──────┬───────┘                              │
│          │                                    │ "Structured article"                  │
│          │                                    ▼                                     │
│          │                             ┌──────────────┐                             │
│          │                             │   Quality    │                             │
│          │                             │   Engine     │                             │
│          │                             └──────┬───────┘                             │
│          │                                    │ "Score + check duplicate"           │
│          │                                    ▼                                     │
│          │                             ┌──────────────┐                             │
│          │                             │ Deduplicator │                             │
│          │                             │  (SHA-256)   │                             │
│          │                             └──────┬───────┘                             │
│          │                                    │                                     │
│          │                       ┌────────────┴────────────┐                         │
│          │                       ▼                         ▼                         │
│          │              ┌──────────────┐          ┌──────────────┐                   │
│          │              │  Store New   │          │ Skip (dup)   │                   │
│          │              │  Article     │          │ Update stats │                   │
│          │              └──────┬───────┘          └──────────────┘                   │
│          │                     │ "Saved to DB"                                       │
│          │                     ▼                                                     │
│          │            ┌──────────────────┐                                           │
│          │            │  PostgreSQL 16   │                                           │
│          │            │                  │                                           │
│          │            │  articles table  │                                           │
│          │            └──────────────────┘                                           │
│          │                     │                                                     │
│          │                     ▼                                                     │
│          │            ┌──────────────────┐                                           │
│          │            │  API (Fastify)   │◀── Consumers query here                  │
│          │            │  serves data     │                                           │
│          │            └──────────────────┘                                           │
│          │                                                                           │
│          └────── Job completed / failed ──────────────────────────────────────────▶│
│                        ▲                                                          │
│                        │                                                          │
│                   ┌────┴────┐  On failure: retry 3x, then move to DLQ              │
│                   │  Retry  │                                                      │
│                   │ Handler │                                                      │
│                   └─────────┘                                                      │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Specifications


### 2.1 Source Registry

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Central configuration store defining every source Jarvis HQ scrapes — URLs, schedules, selectors, and metadata |
| **Responsibilities** | Load & validate YAML configs; expose typed config objects to scheduler; validate against JSON Schema; hot-reload on file change |
| **Input** | YAML files in `config/sources/*.yaml` |
| **Output** | Validated `SourceConfig[]` array; per-source: `id`, `name`, `url`, `adapter`, `schedule`, `topics`, `selectors`, `rateLimit`, `headers`, `auth` |
| **Tech Choice** | `js-yaml@4.1` + `zod@3.23` for schema validation |
| **Key Design Decisions** | One file per source (not one monolithic file) so adding a source is just `cp template.yaml ai-blog.yaml`; Zod schemas give compile-time + runtime validation; configs mounted as Docker volume for hot reload without rebuild |

**Example source config:**
```yaml
# config/sources/techcrunch-ai.yaml
id: techcrunch-ai
name: "TechCrunch AI"
enabled: true
url: "https://techcrunch.com/category/artificial-intelligence/feed/"
adapter: rss
topics:
  - ai
schedule: "*/30 * * * *"   # Every 30 minutes
rateLimit:
  requestsPerMinute: 10
  respectRobotsTxt: true
selectors:
  # RSS-specific: which fields to extract
  titleField: title
  linkField: link
  pubDateField: pubDate
  contentField: description
tags:
  - tech
  - ai
  - news
```

**Zod schema for validation:**
```typescript
// src/registry/schemas.ts
import { z } from 'zod';

export const SourceConfigSchema = z.object({
  id: z.string().regex(/^[a-z0-9-]+$/),
  name: z.string().min(1),
  enabled: z.boolean().default(true),
  url: z.string().url(),
  adapter: z.enum([
    'rss', 'api', 'sitemap', 'html_article', 'github_release',
    'youtube', 'social', 'search_discovery'
  ]),
  topics: z.array(z.string()).min(1),
  schedule: z.string(),                    // cron expression
  rateLimit: z.object({
    requestsPerMinute: z.number().min(1).max(1000).default(10),
    respectRobotsTxt: z.boolean().default(true),
    crawlDelay: z.number().optional(),     // additional configured delay; never lower than robots.txt
  }).default({}),
  selectors: z.record(z.string()).optional(),
  headers: z.record(z.string()).optional(),
  auth: z.object({
    type: z.enum(['bearer', 'api_key', 'basic', 'none']).default('none'),
    envKey: z.string().optional(),         // env var name holding the secret
  }).optional(),
  tags: z.array(z.string()).default([]),
});

export type SourceConfig = z.infer<typeof SourceConfigSchema>;
```

---

### 2.2 Scheduler

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Create scrape jobs on time-based triggers per source |
| **Responsibilities** | Parse cron expressions; on tick, construct a `ScrapeJob` and push to queue; skip disabled sources; log all scheduled runs; expose manual trigger endpoint |
| **Input** | `SourceConfig[]` from registry; cron expressions per source |
| **Output** | `ScrapeJob` objects: `{ id, sourceId, adapterType, url, priority, createdAt, attemptCount }` |
| **Tech Choice** | `node-cron@3.0` + custom `SchedulerService` class |
| **Key Design Decisions** | One cron job per enabled source (not one global ticker); jobs include `priority` based on topic freshness needs (breaking news = high priority); manual trigger via API bypasses cron for on-demand scraping |

**Job type definition:**
```typescript
// src/queue/types.ts
interface ScrapeJob {
  id: string;                    // uuid v4
  sourceId: string;              // references SourceConfig.id
  adapterType: string;           // which adapter handles this
  url: string;                   // target URL to fetch
  priority: number;              // 1-10 (1 = highest)
  topic: string;                 // primary topic for routing
  createdAt: Date;
  scheduledFor?: Date;
  attemptCount: number;          // incremented on each retry
  maxAttempts: number;           // default: 3
}
```

**Scheduler implementation sketch:**
```typescript
// src/scheduler/scheduler.ts
import cron from 'node-cron';
import { v4 as uuidv4 } from 'uuid';
import { SourceConfig } from '../registry/schemas';
import { JobQueue } from '../queue/job-queue';

export class SchedulerService {
  private tasks: Map<string, cron.ScheduledTask> = new Map();
  
  constructor(
    private registry: SourceConfig[],
    private queue: JobQueue
  ) {}

  start(): void {
    for (const source of this.registry.filter(s => s.enabled)) {
      const task = cron.schedule(source.schedule, async () => {
        const job = {
          id: uuidv4(),
          sourceId: source.id,
          adapterType: source.adapter,
          url: source.url,
          priority: this.computePriority(source),
          topic: source.topics[0],
          createdAt: new Date(),
          attemptCount: 0,
          maxAttempts: 3,
        };
        await this.queue.add(job);
      }, { scheduled: true });
      this.tasks.set(source.id, task);
    }
  }

  private computePriority(source: SourceConfig): number {
    // Breaking news sources = 1, daily roundups = 5, weekly = 10
    if (source.tags.includes('breaking')) return 1;
    if (source.schedule.includes('*/15')) return 2;
    if (source.schedule.includes('*/30')) return 3;
    return 5;
  }

  async triggerManual(sourceId: string): Promise<string> {
    const source = this.registry.find(s => s.id === sourceId);
    if (!source) throw new Error(`Source ${sourceId} not found`);
    const job = { /* ... same as above ... */ };
    await this.queue.add(job, { priority: 1 });
    return job.id;
  }

  stop(): void {
    for (const [_, task] of this.tasks) task.stop();
    this.tasks.clear();
  }
}
```

---

### 2.3 Job Queue

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Reliable job distribution with priority, retries, persistence, and dead-letter handling |
| **Responsibilities** | Accept jobs from scheduler; dispatch to workers; retry failed jobs (exponential backoff); move exhausted jobs to DLQ; expose queue depth metrics; handle graceful shutdown (finish in-flight jobs) |
| **Input** | `ScrapeJob` objects |
| **Output** | Job picked up by adapter workers; completion/failure events |
| **Tech Choice** | `bullmq@5.x` (Redis-backed, Bull successor, native TypeScript) |
| **Key Design Decisions** | Redis via Docker — already needed for rate limiting, so shared instance is fine; BullMQ supports priorities natively; separate queues per adapter type (`queue:rss`, `queue:html`) for parallel processing; stalled jobs auto-recovered; DLQ queue for manual inspection of permanent failures |

**Queue architecture:**
```
Redis 7
├── bull:rss:jobs           (RSS feed scraping)
├── bull:api:jobs           (API endpoint calls)
├── bull:sitemap:jobs       (Sitemap parsing)
├── bull:html:jobs          (HTML article extraction)
├── bull:github:jobs        (GitHub release checks)
├── bull:youtube:jobs       (YouTube channel checks)
├── bull:search:jobs        (Search discovery)
├── bull:dlq:jobs           (Dead letter queue — permanent failures)
└── bull:metrics            (Queue depth, processing times)
```

**Worker implementation:**
```typescript
// src/queue/worker.ts
import { Worker, Job } from 'bullmq';
import { AdapterFactory } from '../adapters/adapter-factory';

export function createAdapterWorker(queueName: string, adapterFactory: AdapterFactory) {
  const worker = new Worker(
    queueName,
    async (job: Job<ScrapeJob>) => {
      const adapter = adapterFactory.get(job.data.adapterType);
      const result = await adapter.execute(job.data);
      return result;
    },
    {
      connection: redisConnection,
      concurrency: 3,                    // Process 3 jobs in parallel per worker
      lockDuration: 30000,               // 30s lock before considering stalled
      stalledInterval: 30000,
    }
  );

  worker.on('completed', (job) => {
    logger.info(`Job ${job.id} completed`, { sourceId: job.data.sourceId });
  });

  worker.on('failed', (job, err) => {
    logger.error(`Job ${job?.id} failed`, { error: err.message, attempt: job?.attemptsMade });
  });

  return worker;
}
```

---

### 2.4 RSS Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Fetch and parse RSS, Atom, and JSON Feed formats |
| **Responsibilities** | HTTP GET feed URL; parse XML/JSON response; extract articles with title, link, pubDate, content, author; handle feed pagination (rare); normalize dates to ISO 8601; respect HTTP caching headers (ETag, Last-Modified) |
| **Input** | `ScrapeJob` with adapterType `'rss'` |
| **Output** | `RawArticle[]`: `{ title, url, publishedAt, content, author, sourceId, rawHtml? }` |
| **Tech Choice** | `feedparser@2.2` (streaming XML parser, battle-tested, handles RSS 0.9x through 2.0 + Atom) + `fast-xml-parser@4.4` (fallback for edge cases) |
| **Key Design Decisions** | Streaming parse for memory efficiency on large feeds; store raw XML in `raw_data` table for debugging; normalize all date formats to UTC; follow redirects (max 5); timeout after 15s |

```typescript
// src/adapters/rss-adapter.ts
import FeedParser from 'feedparser';
import { fetchWithTimeout } from '../utils/fetch';

export class RssAdapter implements SourceAdapter {
  async execute(job: ScrapeJob): Promise<RawArticle[]> {
    const response = await fetchWithTimeout(job.url, {
      timeout: 15000,
      headers: { 'User-Agent': 'JarvisHQ/1.0 (Personal News Bot)' },
      maxRedirects: 5,
    });

    const feedparser = new FeedParser({ addmeta: false });
    const articles: RawArticle[] = [];

    return new Promise((resolve, reject) => {
      feedparser.on('readable', () => {
        let item;
        while (item = feedparser.read()) {
          articles.push({
            title: item.title?.trim(),
            url: item.link,
            publishedAt: new Date(item.date || item.pubdate || Date.now()),
            content: item.description || item.summary,
            author: item.author,
            sourceId: job.sourceId,
          });
        }
      });

      feedparser.on('end', () => resolve(articles));
      feedparser.on('error', reject);

      response.body.pipe(feedparser);
    });
  }
}
```

---

### 2.5 API Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Generic REST/GraphQL adapter for APIs (NewsAPI, Hacker News Algolia, GitHub, etc.) |
| **Responsibilities** | Build HTTP requests from config; inject authentication (API keys from env/1Password); handle JSON responses; map response fields to article schema via JMESPath/jsonpath selectors; paginate if API supports it; handle rate limit responses (429) |
| **Input** | `ScrapeJob` with adapterType `'api'` + source config with `selectors` mapping |
| **Output** | `RawArticle[]` |
| **Tech Choice** | Native `fetch` (Node.js 20+) + `jmespath@0.16` for JSON path extraction |
| **Key Design Decisions** | Config-driven field mapping — no code changes when adding new API sources; secrets injected via env vars (e.g., `NEWSAPI_KEY`), never hardcoded; 429 responses trigger rate limiter update; response bodies cached in raw_data for 24h |

```yaml
# Example API source config
id: hackernews-top
name: "Hacker News Top Stories"
adapter: api
url: "https://hacker-news.firebaseio.com/v0/topstories.json"
topics: [tech, ai]
schedule: "*/15 * * * *"
selectors:
  # HN requires a two-step fetch: get IDs, then get each story
  itemIdsPath: "@[0:30]"                          # JMESPath: first 30 IDs
  itemDetailUrl: "https://hacker-news.firebaseio.com/v0/item/{id}.json"
  titlePath: "title"
  urlPath: "url"
  scorePath: "score"
  publishedAtPath: "time"                          # Unix timestamp
```

---

### 2.6 Sitemap Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Parse XML sitemaps to discover article URLs, then queue individual article extractions |
| **Responsibilities** | Fetch sitemap.xml/sitemap_index.xml; parse `<url>` or `<sitemap>` entries; filter URLs by patterns (e.g., only `/articles/` paths); check `lastmod` to skip already-processed URLs; emit individual article jobs for HTML extraction |
| **Input** | `ScrapeJob` with adapterType `'sitemap'` |
| **Output** | Array of discovered URLs (written back to queue as new `ScrapeJob`s for the HTML adapter) |
| **Tech Choice** | `fast-xml-parser@4.4` + native `fetch` |
| **Key Design Decisions** | Sitemap adapter is a "producer" adapter — it doesn't produce articles directly, it produces more jobs; only follows sitemap links 1 level deep for sitemap indexes; filters URLs by date (skip articles older than last scrape); respects `<priority>` hints |

```typescript
// src/adapters/sitemap-adapter.ts
import { XMLParser } from 'fast-xml-parser';

export class SitemapAdapter implements SourceAdapter {
  async execute(job: ScrapeJob): Promise<DiscoveredUrl[]> {
    const response = await fetchWithTimeout(job.url, { timeout: 15000 });
    const xml = await response.text();
    
    const parser = new XMLParser({ ignoreAttributes: false });
    const parsed = parser.parse(xml);

    // Handle sitemap index (nested sitemaps)
    if (parsed.sitemapindex) {
      return this.handleSitemapIndex(parsed.sitemapindex);
    }

    // Handle regular URL set
    const urls = parsed.urlset?.url || [];
    const urlArray = Array.isArray(urls) ? urls : [urls];

    return urlArray
      .filter((u: any) => this.matchesPattern(u.loc, job.selectors?.urlPattern))
      .filter((u: any) => this.isNewerThan(u.lastmod, job.selectors?.lastScraped))
      .map((u: any) => ({
        url: u.loc,
        lastmod: u.lastmod ? new Date(u.lastmod) : undefined,
      }));
  }
}
```

---

### 2.7 HTML Article Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Extract article content from raw HTML pages — the most complex adapter |
| **Responsibilities** | Fetch HTML page; extract article content (title, body, author, publish date, images); clean boilerplate (nav, ads, footers); convert relative URLs to absolute; handle JavaScript-rendered pages (fallback to headless browser); extract structured data (JSON-LD) |
| **Input** | `ScrapeJob` with adapterType `'html_article'` |
| **Output** | `RawArticle` with full `contentHtml` and plain-text `contentText` |
| **Tech Choice** | Primary: `mozilla-readability@0.5` (Mozilla's Firefox reader mode engine); Fallback: `trafilatura` via Python subprocess if Node libraries fail; Headless: `puppeteer@21` for JS-rendered sites only |
| **Key Design Decisions** | Tiered approach for speed: (1) Readability on raw HTML, (2) JSON-LD structured data extraction, (3) Python trafilatura for stubborn pages, (4) Puppeteer only for SPAs — most sites work with #1 or #2; cache raw HTML for 24h to allow re-extraction if parser improves; image extraction limits to first 3 images per article |

```typescript
// src/adapters/html-article-adapter.ts
import { Readability } from '@mozilla/readability';
import { JSDOM } from 'jsdom';

export class HtmlArticleAdapter implements SourceAdapter {
  async execute(job: ScrapeJob): Promise<RawArticle> {
    const html = await this.fetchHtml(job.url);
    
    // Try JSON-LD structured data first (most accurate for news sites)
    const structured = this.extractJsonLd(html);
    
    // Fall back to Readability
    const dom = new JSDOM(html, { url: job.url });
    const reader = new Readability(dom.window.document);
    const article = reader.parse();

    // Extract publish date from meta tags
    const publishDate = this.extractPublishDate(dom, structured);

    return {
      title: structured?.headline || article?.title || this.extractTitle(dom),
      url: job.url,
      publishedAt: publishDate,
      content: article?.textContent || '',
      contentHtml: article?.content || '',
      author: structured?.author?.name || this.extractAuthor(dom),
      sourceId: job.sourceId,
      images: this.extractImages(article?.content || html, job.url).slice(0, 3),
    };
  }

  private extractJsonLd(html: string): any {
    const match = html.match(/<script type="application\/ld\+json">(.*?)<\/script>/s);
    if (!match) return null;
    try {
      const data = JSON.parse(match[1]);
      return data['@type'] === 'NewsArticle' ? data : null;
    } catch { return null; }
  }
}
```

---

### 2.8 GitHub Release Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Monitor GitHub repositories for new releases and tags |
| **Responsibilities** | Call GitHub REST API `/repos/{owner}/{repo}/releases`; paginate through releases; extract version, release notes, publish date, assets; support both releases API and tags (for repos that don't use GitHub releases); respect GitHub's 60 req/hour unauthenticated / 5000 req/hour authenticated limits |
| **Input** | `ScrapeJob` with adapterType `'github_release'` |
| **Output** | `RawArticle[]` with release info formatted as readable articles |
| **Tech Choice** | Native `fetch` to GitHub REST API v3; `octokit@3.x` if more complex GitHub operations needed later |
| **Key Design Decisions** | Unauthenticated for public repos by default; `GITHUB_TOKEN` env var for higher rate limits and private repos; releases treated as articles with `type: 'github_release'` for special rendering; tag diffs linked automatically |

```typescript
// src/adapters/github-release-adapter.ts
export class GitHubReleaseAdapter implements SourceAdapter {
  async execute(job: ScrapeJob): Promise<RawArticle[]> {
    const { owner, repo } = this.parseRepoUrl(job.url);
    const token = process.env.GITHUB_TOKEN;
    
    const headers: Record<string, string> = {
      'Accept': 'application/vnd.github.v3+json',
      'User-Agent': 'JarvisHQ/1.0',
    };
    if (token) headers['Authorization'] = `Bearer ${token}`;

    const response = await fetch(
      `https://api.github.com/repos/${owner}/${repo}/releases`,
      { headers }
    );

    if (response.status === 403) {
      const resetTime = response.headers.get('x-ratelimit-reset');
      throw new RateLimitError('GitHub rate limit exceeded', { resetTime });
    }

    const releases = await response.json();
    return releases.map((rel: any) => ({
      title: `${owner}/${repo} ${rel.tag_name}`,
      url: rel.html_url,
      publishedAt: new Date(rel.published_at),
      content: rel.body || `Release ${rel.tag_name}`,
      author: rel.author?.login,
      sourceId: job.sourceId,
      metadata: {
        type: 'github_release',
        version: rel.tag_name,
        isPrerelease: rel.prerelease,
        assets: rel.assets?.map((a: any) => a.name),
      },
    }));
  }
}
```

---

### 2.9 YouTube Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Monitor YouTube channels/playlists for new videos |
| **Responsibilities** | Prefer RSS feeds over API (no API key needed); parse channel RSS to get video entries; extract title, description, publish date, video ID; build embed URLs; fallback to YouTube Data API v3 for channels without RSS |
| **Input** | `ScrapeJob` with adapterType `'youtube'` |
| **Output** | `RawArticle[]` with YouTube video entries |
| **Tech Choice** | RSS-first approach using channel RSS feeds; `youtubei.js@9.x` as fallback for metadata enrichment |
| **Key Design Decisions** | **RSS first** — `https://www.youtube.com/feeds/videos.xml?channel_id={ID}` works without API key and gives same data; YouTube Data API (`YOUTUBE_API_KEY` env var) only as fallback; videos stored with `type: 'youtube'` for special card rendering; thumbnails extracted from RSS media:group |

```typescript
// src/adapters/youtube-adapter.ts
export class YouTubeAdapter extends RssAdapter {
  // YouTube RSS feeds are standard RSS — we inherit from RssAdapter
  // but override to add YouTube-specific fields
  
  async execute(job: ScrapeJob): Promise<RawArticle[]> {
    const articles = await super.execute(job);
    return articles.map(article => {
      const videoId = this.extractVideoId(article.url);
      return {
        ...article,
        metadata: {
          type: 'youtube',
          videoId,
          embedUrl: `https://www.youtube.com/embed/${videoId}`,
          thumbnailUrl: `https://img.youtube.com/vi/${videoId}/mqdefault.jpg`,
        },
      };
    });
  }

  private extractVideoId(url: string): string {
    const match = url.match(/(?:v=|\/)([a-zA-Z0-9_-]{11})/);
    return match?.[1] || '';
  }
}
```

---

### 2.10 Social Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Fetch public posts from social platforms (Bluesky, Mastodon, Reddit, Twitter/X) |
| **Responsibilities** | Platform-specific API integration; public post fetching (no auth where possible); normalize post format to article schema; handle rate limits and API changes; respect robots.txt and terms of service |
| **Input** | `ScrapeJob` with adapterType `'social'` |
| **Output** | `RawArticle[]` with social posts |
| **Tech Choice** | Bluesky: `@atproto/api@0.12`; Mastodon: native REST; Reddit: public JSON/RSS endpoints when allowed; Twitter/X: NOT supported except through official API access or an explicitly permitted export path |
| **Key Design Decisions** | **Bluesky and Mastodon prioritized** — open APIs, no auth required for public posts; Reddit `.json` trick for public subreddits; **Twitter/X explicitly excluded** from core adapters — API costs $100/month minimum, scraping violates ToS; each platform has its own sub-adapter registered with the adapter factory; respect each platform's robots.txt |

```typescript
// src/adapters/social/bluesky-adapter.ts
import { AtpAgent } from '@atproto/api';

export class BlueskyAdapter implements SocialSubAdapter {
  private agent = new AtpAgent({ service: 'https://public.api.bsky.app' });

  async fetchPosts(handle: string, limit: number = 30): Promise<RawArticle[]> {
    const profile = await this.agent.getProfile({ actor: handle });
    const feed = await this.agent.getAuthorFeed({
      actor: profile.data.did,
      limit,
    });

    return feed.data.feed.map(item => ({
      title: item.post.record.text.substring(0, 100),
      url: `https://bsky.app/profile/${handle}/post/${item.post.uri.split('/').pop()}`,
      publishedAt: new Date(item.post.indexedAt),
      content: item.post.record.text,
      author: profile.data.displayName || handle,
      sourceId: `bluesky-${handle}`,
      metadata: {
        type: 'social',
        platform: 'bluesky',
        likes: item.post.likeCount,
        reposts: item.post.repostCount,
        replies: item.post.replyCount,
      },
    }));
  }
}
```

---

### 2.11 Search Discovery Adapter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Discover new sources and trending articles via search engines |
| **Responsibilities** | Execute search queries against configured search API; parse results into discovered URLs; filter and rank by relevance; optionally auto-queue discovered URLs for follow-up scraping |
| **Input** | `ScrapeJob` with adapterType `'search_discovery'` |
| **Output** | `DiscoveredUrl[]` (new jobs queued for HTML adapter) |
| **Tech Choice** | Primary: Google Custom Search API (100 queries/day free); Fallback: Brave Search API; Tertiary: DuckDuckGo Instant Answer API. Do not scrape DuckDuckGo HTML — their ToS prohibits automated scraping. |
| **Key Design Decisions** | Heavily rate-limited by design — 10-50 queries/day max; used for "trending" discovery, not regular scraping; results stored in `discovered_urls` table with `pending_review` status; manual approval required before auto-adding as new sources; query templates per topic (e.g., `"AI news" after:2024-01-01`) |

---

### 2.12 Rate Limiter

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Prevent overwhelming source servers; enforce per-domain rate limits; respect robots.txt |
| **Responsibilities** | Fetch and parse robots.txt for each domain; extract Crawl-delay directive; maintain per-domain token buckets; delay requests that exceed limits; expose current rate limit status via API |
| **Input** | Domain name + configured rate limit; robots.txt content |
| **Output** | `RateLimitStatus`: `{ domain, requestsRemaining, resetTime, crawlDelay }` |
| **Tech Choice** | Custom implementation using `ioredis@5` for distributed token buckets; `robots-parser@3.x` for robots.txt parsing |
| **Key Design Decisions** | **Respect-first architecture** — always fetch and cache robots.txt; cache robots.txt for 24h; token bucket algorithm (not fixed window) for smooth request distribution; per-domain buckets stored in Redis; configurable global max concurrency (default: 5 concurrent fetches across all domains); queue job if rate limit exceeded rather than dropping |

```typescript
// src/rate-limiter/rate-limiter.ts
import Redis from 'ioredis';
import { RobotsParser } from 'robots-parser';

interface TokenBucket {
  tokens: number;
  lastRefill: number;
}

export class RateLimiter {
  private redis: Redis;
  private robotsCache: Map<string, RobotsParser> = new Map();
  
  constructor(
    private defaultRpm: number = 10,
    private globalConcurrency: number = 5
  ) {
    this.redis = new Redis({ host: process.env.REDIS_HOST || 'redis' });
  }

  async checkLimit(url: string, sourceConfig?: SourceConfig): Promise<void> {
    const domain = new URL(url).hostname;
    const key = `ratelimit:${domain}`;
    
    // Get effective rate limit: config > robots.txt > default
    const limit = await this.getEffectiveLimit(domain, sourceConfig);
    
    // Token bucket check via Redis Lua script (atomic)
    const lua = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refillRate = tonumber(ARGV[2])  -- tokens per second
      local now = tonumber(ARGV[3])
      
      local bucket = redis.call('hmget', key, 'tokens', 'lastRefill')
      local tokens = tonumber(bucket[1]) or capacity
      local lastRefill = tonumber(bucket[2]) or now
      
      local elapsed = now - lastRefill
      tokens = math.min(capacity, tokens + elapsed * refillRate)
      
      if tokens < 1 then
        return {-1, math.ceil((1 - tokens) / refillRate)}
      end
      
      tokens = tokens - 1
      redis.call('hmset', key, 'tokens', tokens, 'lastRefill', now)
      redis.call('expire', key, 86400)
      return {tokens, 0}
    `;

    const now = Date.now() / 1000;
    const result = await this.redis.eval(lua, 1, key, limit, limit / 60, now);
    
    if (result[0] === -1) {
      const waitSeconds = result[1];
      throw new RateLimitExceededError(domain, waitSeconds);
    }
  }

  private async getEffectiveLimit(domain: string, config?: SourceConfig): Promise<number> {
    if (config?.rateLimit?.requestsPerMinute) return config.rateLimit.requestsPerMinute;
    
    const robots = await this.getRobotsParser(domain);
    const crawlDelay = robots.getCrawlDelay('JarvisHQ') || robots.getCrawlDelay('*');
    if (crawlDelay) return Math.floor(60 / crawlDelay);
    
    return this.defaultRpm;
  }

  private async getRobotsParser(domain: string): Promise<RobotsParser> {
    if (this.robotsCache.has(domain)) return this.robotsCache.get(domain)!;
    
    try {
      const res = await fetch(`https://${domain}/robots.txt`, { timeout: 10000 });
      const text = await res.text();
      const parser = new RobotsParser(`https://${domain}/robots.txt`, text);
      this.robotsCache.set(domain, parser);
      return parser;
    } catch {
      return new RobotsParser('', ''); // Allow all on failure
    }
  }
}
```

---

### 2.13 Retry Handler

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Handle transient failures gracefully; retry with backoff; classify errors |
| **Responsibilities** | Classify errors as transient (retryable) or permanent; apply exponential backoff with jitter; cap max retries; move exhausted jobs to DLQ; log retry attempts |
| **Input** | Failed job + error object |
| **Output** | Retried job (re-queued) or DLQ entry |
| **Tech Choice** | Built into BullMQ (configurable per-queue); custom `ErrorClassifier` for error categorization |
| **Key Design Decisions** | **3 retry tiers**: (1) Immediate (network blip — retry in 5s), (2) Short backoff (429 rate limit — retry after Retry-After header), (3) Long backoff (5xx server error — retry in 60s, 120s, 240s); jitter added to prevent thundering herd; permanent errors (404, invalid config, parse failure) go straight to DLQ; DLQ entries include full context for debugging |

```typescript
// src/retry/error-classifier.ts
export enum ErrorCategory {
  TRANSIENT_NETWORK = 'transient_network',    // ECONNRESET, ETIMEDOUT, DNS failure
  TRANSIENT_RATE_LIMIT = 'transient_rate_limit', // HTTP 429
  TRANSIENT_SERVER = 'transient_server',      // HTTP 5xx
  PERMANENT_NOT_FOUND = 'permanent_not_found',// HTTP 404
  PERMANENT_AUTH = 'permanent_auth',          // HTTP 401/403 (config issue)
  PERMANENT_PARSE = 'permanent_parse',        // Unexpected HTML structure
  PERMANENT_CONFIG = 'permanent_config',      // Invalid source config
}

export interface RetryPolicy {
  maxRetries: number;
  backoffMs: number;      // Initial backoff
  maxBackoffMs: number;   // Cap
  multiplier: number;     // Exponential factor
  jitter: boolean;
}

const RETRY_POLICIES: Record<ErrorCategory, RetryPolicy> = {
  [ErrorCategory.TRANSIENT_NETWORK]: {
    maxRetries: 3,
    backoffMs: 5000,
    maxBackoffMs: 60000,
    multiplier: 2,
    jitter: true,
  },
  [ErrorCategory.TRANSIENT_RATE_LIMIT]: {
    maxRetries: 5,
    backoffMs: 60000,       // Start at 1 minute
    maxBackoffMs: 3600000,  // Cap at 1 hour
    multiplier: 2,
    jitter: true,
  },
  [ErrorCategory.TRANSIENT_SERVER]: {
    maxRetries: 3,
    backoffMs: 60000,
    maxBackoffMs: 300000,
    multiplier: 2,
    jitter: true,
  },
  // Permanent errors: maxRetries = 0 (go straight to DLQ)
  [ErrorCategory.PERMANENT_NOT_FOUND]: { maxRetries: 0, backoffMs: 0, maxBackoffMs: 0, multiplier: 0, jitter: false },
  [ErrorCategory.PERMANENT_AUTH]: { maxRetries: 0, backoffMs: 0, maxBackoffMs: 0, multiplier: 0, jitter: false },
  [ErrorCategory.PERMANENT_PARSE]: { maxRetries: 0, backoffMs: 0, maxBackoffMs: 0, multiplier: 0, jitter: false },
  [ErrorCategory.PERMANENT_CONFIG]: { maxRetries: 0, backoffMs: 0, maxBackoffMs: 0, multiplier: 0, jitter: false },
};

export function classifyError(error: unknown): { category: ErrorCategory; retryAfter?: number } {
  if (error instanceof RateLimitExceededError) {
    return { category: ErrorCategory.TRANSIENT_RATE_LIMIT, retryAfter: error.retryAfterSeconds };
  }
  if (error instanceof FetchError) {
    if (error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT' || error.code === 'ENOTFOUND') {
      return { category: ErrorCategory.TRANSIENT_NETWORK };
    }
    if (error.status >= 500) return { category: ErrorCategory.TRANSIENT_SERVER };
    if (error.status === 429) return { category: ErrorCategory.TRANSIENT_RATE_LIMIT };
    if (error.status === 404) return { category: ErrorCategory.PERMANENT_NOT_FOUND };
    if (error.status === 401 || error.status === 403) return { category: ErrorCategory.PERMANENT_AUTH };
  }
  if (error instanceof ParseError) {
    return { category: ErrorCategory.PERMANENT_PARSE };
  }
  return { category: ErrorCategory.TRANSIENT_NETWORK };
}
```

---

### 2.14 Logger

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Structured, queryable logging for debugging and monitoring |
| **Responsibilities** | Log all scrape operations; include job context (sourceId, jobId, adapter); support multiple levels; write to stdout + file; rotate logs; redact sensitive data |
| **Input** | Log messages + context objects |
| **Output** | JSON-formatted log lines to stdout and/or file |
| **Tech Choice** | `pino@9.x` (fast JSON logger) + `pino-pretty` for dev; `pino-rotating-file` for production file output |
| **Key Design Decisions** | **Always structured JSON** — every log line includes `jobId`, `sourceId`, `adapter` as top-level fields; child loggers per job for automatic context propagation; sensitive fields (API keys, auth tokens) automatically redacted; log levels: `trace`, `debug`, `info`, `warn`, `error`, `fatal` |

```typescript
// src/logger/logger.ts
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: {
    service: 'jarvis-hq',
    version: process.env.npm_package_version || '1.0.0',
  },
  redact: {
    paths: ['*.apiKey', '*.token', '*.password', '*.secret', 'headers.authorization'],
    remove: true,
  },
  transport: process.env.NODE_ENV === 'development' 
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
});

export function getJobLogger(jobId: string, sourceId: string, adapter: string) {
  return logger.child({ jobId, sourceId, adapter });
}

export { logger };
```

---

### 2.15 Error Handler

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Central error handling, classification, and reporting |
| **Responsibilities** | Catch unhandled errors; classify by severity; decide retry vs DLQ; emit error events; persist error details for later analysis; alert on systemic issues |
| **Input** | Any error thrown in the pipeline |
| **Output** | Classified error record; retry decision; alert (if systemic) |
| **Tech Choice** | Custom `ErrorHandler` class integrated with BullMQ event system |
| **Key Design Decisions** | All errors stored in `errors` table with full context (stack trace, job data, response body snippet); systemic error detection: >5 same-source failures in 1 hour triggers alert; errors linked to jobs for traceability; cleanup job purges errors older than 30 days |

---

## 3. Data Flow

### 3.1 Complete Pipeline Flow

```
STEP 1: SCHEDULER CREATES A SCRAPE JOB
─────────────────────────────────────────
1.1 node-cron fires per configured schedule
1.2 Scheduler reads SourceRegistry for source config
1.3 Validates source is enabled and schedule matches
1.4 Constructs ScrapeJob with UUID, priority, metadata
1.5 Pushes to appropriate BullMQ queue (based on adapter type)
1.6 Logs: "Scheduled job {id} for {source} at {timestamp}"

STEP 2: JOB ADDED TO QUEUE
─────────────────────────────────────────
2.1 BullMQ persists job to Redis (durability)
2.2 Priority queue orders jobs (lower number = higher priority)
2.3 Queue metrics updated (depth, wait time)
2.4 Worker picks up next available job (respecting concurrency limit)

STEP 3: ADAPTER FETCHES CONTENT
─────────────────────────────────────────
3.1 Worker calls AdapterFactory.get(adapterType) → returns adapter instance
3.2 RateLimiter.checkLimit(url) called before any HTTP request
3.2a  If robots.txt not cached → fetch and parse, cache for 24h
3.2b  If rate limit exceeded → job delayed (re-added to queue with delay)
3.3 Adapter performs HTTP fetch with configured headers + auth
3.4 HTTP response received (raw HTML/JSON/XML)
3.5 Raw response stored in raw_data table (cache for 24h)
3.6 Adapter-specific parsing → RawArticle[]

STEP 4: EXTRACTOR PARSES CONTENT
─────────────────────────────────────────
4.1 HTML content passed to extraction pipeline
4.2 Tier 1: Readability (Mozilla) — extracts title, body, author
4.3 Tier 2: JSON-LD structured data — headline, datePublished, author
4.4 Tier 3: Meta tag extraction — og:title, og:description, article:published_time
4.5 Tier 4: Python trafilatura subprocess (if Trafilatura env enabled)
4.6 Content normalized: HTML sanitized (DOMPurify), relative URLs absolutized
4.7 Plain-text version generated for search indexing
4.8 Image URLs extracted and validated (first 3 kept)

STEP 5: QUALITY ENGINE SCORES
─────────────────────────────────────────
5.1 Quality check: title length (10-200 chars acceptable)
5.2 Quality check: content length (>100 chars minimum)
5.3 Quality check: has valid URL
5.4 Quality check: published date within last 30 days (configurable)
5.5 Quality check: URL is reachable (HEAD request, optional)
5.6 Composite quality score calculated (0-100)
5.7 Articles below quality threshold (score < 30) logged and skipped
5.8 Metadata extracted: estimated reading time, language detection

STEP 6: DEDUPLICATION CHECK
─────────────────────────────────────────
6.1 Generate content hash: SHA256 of normalized title + first 200 chars of content
6.2 Query articles table: SELECT id WHERE content_hash = ?
6.3 If duplicate found:
    6.3a If same source → skip (already scraped)
    6.3b If different source → store as duplicate with link to original
6.4 If not duplicate → proceed to storage
6.5 URL also checked: SELECT id WHERE url = ? (exact match prevention)

STEP 7: STORAGE IN POSTGRESQL
─────────────────────────────────────────
7.1 INSERT INTO articles with all extracted fields
7.2 INSERT INTO article_topics (many-to-many link)
7.3 INSERT INTO article_images if images present
7.4 INSERT INTO article_metadata for adapter-specific data
7.5 Job marked as completed in queue
7.6 Metrics updated: articles_scraped counter incremented
7.7 Search index updated (tsvector for full-text search)

STEP 8: API SERVES DATA
─────────────────────────────────────────
8.1 Fastify API serves requests from consumers
8.2 GET /api/articles — paginated, sortable, filterable
8.3 GET /api/articles/topic/:topic — filtered by topic
8.4 GET /api/articles/search — full-text search via PostgreSQL tsvector
8.5 GET /feed/rss — RSS feed output of latest articles
8.6 WebSocket push available for real-time new article notifications
8.7 n8n webhook triggered on new article (if configured)
```

### 3.2 Data Flow Diagram (Error Path)

```
┌─────────────┐    Fetch Error    ┌──────────────────┐
│   Adapter   │──────────────────▶│  Error Classifier │
│   Execute   │                   └────────┬─────────┘
└─────────────┘                            │
                                           ▼
                              ┌────────────────────────┐
                              │ Transient? (network,   │
                              │ rate limit, 5xx)       │
                              └───────────┬────────────┘
                                          │
                           ┌──────────────┴──────────────┐
                           ▼                             ▼
                    ┌──────────────┐              ┌──────────────┐
                    │  YES → Retry  │              │ NO → DLQ     │
                    │  (backoff)    │              │ (permanent)  │
                    └──────┬───────┘              └──────┬───────┘
                           │                             │
                           ▼                             ▼
                    ┌──────────────┐              ┌──────────────┐
                    │ Re-queue job  │              │ Log to       │
                    │ with delay    │              │ errors table │
                    └──────────────┘              │ Alert if     │
                                                  │ systemic     │
                                                  └──────────────┘
```

---

## 4. Reference Implementation Stack (Node.js)

> **This is one validated reference stack — not a required choice.** This skill
> is model-agnostic and runtime-agnostic; if your target environment already
> has a working stack (Python, Go, Rust, Java, etc.), use it. The "Reference
> choice" callouts below reflect the original Jarvis HQ deployment and are
> useful starting points, not mandates. Adapt freely.

### 4.1 Language: Node.js (Reference Choice)

**Reference choice: Node.js 20 LTS**

| Factor | Node.js | Python | Decision |
|--------|---------|--------|----------|
| **Native async/await** | Excellent (event loop) | Good (asyncio) | Node.js |
| **HTTP request libraries** | Native `fetch`, `undici` | `httpx`, `requests` | Tie |
| **RSS parsing** | `feedparser`, `rss-parser` | `feedparser` (py) | Node.js |
| **HTML extraction** | `readability`, `jsdom` | `trafilatura` (superior) | Python wins |
| **Headless browsing** | `puppeteer`, `playwright` | `playwright`, `selenium` | Node.js |
| **Queue library** | `bullmq` (excellent) | `celery`, `rq` (good) | Node.js |
| **API framework** | `fastify` (fastest) | `fastapi` (excellent) | Tie |
| **Docker image size** | ~200MB (node:20-alpine) | ~500MB (python:3.12-slim) | Node.js |
| **Memory footprint** | Lower | Higher | Node.js |
| **Startup time** | ~100ms | ~300ms | Node.js |
| **Your future stack** | Next.js, n8n both JS | Separate ecosystem | **Node.js** |
| **Trafilatura fallback** | Via Python subprocess | Native | Python wins |

**Decision rationale:**
- **Primary stack: Node.js** — aligns with your existing ecosystem (Next.js frontend, n8n automations), smaller Docker footprint, superior queue libraries (BullMQ), better async handling
- **Trafilatura as subprocess fallback** — for the 5-10% of sites where Readability isn't enough, shell out to a small Python microservice or CLI invocation
- **No Python rewrite needed** — the HTML extraction gap is closed by calling trafilatura as a Docker sidecar or subprocess

### 4.2 Database: PostgreSQL (Reference Choice)

**Reference choice: PostgreSQL 16**

| Factor | SQLite | PostgreSQL 16 | Decision |
|--------|--------|---------------|----------|
| **Setup complexity** | Zero (file) | Docker container | SQLite wins |
| **Concurrent writes** | Poor (WAL mode helps) | Excellent | PostgreSQL |
| **Full-text search** | FTS5 (basic) | `tsvector` + GIN (powerful) | PostgreSQL |
| **JSON operations** | JSON1 extension | Native JSONB + indexing | PostgreSQL |
| **Backup** | Copy file | `pg_dump`, WAL archiving | PostgreSQL |
| **Scaling** | Single node only | Vertical + read replicas | PostgreSQL |
| **Docker** | N/A (just a volume) | `postgres:16-alpine` | Tie |
| **Future-proof** | Limited | Handles all future needs | **PostgreSQL** |

**Decision rationale:**
- **PostgreSQL 16** — full-text search is critical for article search; JSONB for flexible metadata; handles concurrent scraper workers without lock contention
- **Still simple** — single Docker container, `postgres:16-alpine` image, no complex setup
- **Migration path** — if you outgrow it, Postgres scales up easily; SQLite would require a full migration
- **pgAdmin or Adminer** — included in docker-compose for easy data inspection

### 4.3 Queue: BullMQ + Redis (Reference Choice)

**Reference choice: BullMQ (Redis-backed)**

| Factor | In-Memory | Bull (Redis) | BullMQ (Redis) | Decision |
|--------|-----------|--------------|----------------|----------|
| **Durability** | Lost on restart | Persists to Redis | Persists to Redis | BullMQ |
| **Priorities** | Manual | Supported | Native | BullMQ |
| **TypeScript** | N/A | `@types/bull` | Native TS | BullMQ |
| **Retries** | Manual | Built-in | Built-in + backoff | BullMQ |
| **DLQ** | Manual | Manual | Native | BullMQ |
| **Observability** | Custom | Moderate | Built-in UI | BullMQ |
| **Maintenance** | None | Maintenance mode | Actively maintained | **BullMQ** |

**Decision rationale:**
- **BullMQ 5.x** — successor to Bull, native TypeScript, active development, built-in priority queues, flows, and sandboxed workers
- **Redis 7-alpine** — single container, ~30MB memory, shared between queue and rate limiter

### 4.4 Scraping Libraries

| Purpose | Library | Version | Rationale |
|---------|---------|---------|-----------|
| RSS/Atom parsing | `feedparser` | `^2.2.10` | Streaming, handles all RSS variants |
| XML parsing | `fast-xml-parser` | `^4.4.0` | Fast, handles sitemaps, namespaces |
| HTML extraction | `@mozilla/readability` | `^0.5.0` | Firefox's reader mode engine |
| DOM manipulation | `jsdom` | `^24.0.0` | Readability dependency, server-side DOM |
| HTML sanitization | `dompurify` | `^3.1.0` + `jsdom` | XSS protection for stored content |
| Headless browser | `puppeteer` | `^21.0.0` | SPA rendering fallback |
| HTTP requests | Native `fetch` | Node 20+ | Built-in, no dependency |
| JSON path queries | `jmespath` | `^0.16.0` | Config-driven API field mapping |
| Robots.txt parsing | `robots-parser` | `^3.0.1` | Respects Crawl-delay, Allow/Disallow |
| URL utilities | `normalize-url` | `^8.0.1` | Consistent URL normalization |
| Date parsing | `date-fns` | `^3.6.0` | Consistent date handling |
| Cron scheduling | `node-cron` | `^3.0.3` | Simple, reliable cron |
| UUID generation | `uuid` | `^9.0.1` | Job IDs |
| Content hashing | Node `crypto` | Built-in | SHA-256 deduplication |
| Language detection | `franc` | `^6.1.0` | Filter non-English articles (optional) |

### 4.5 API: REST (Fastify, Reference Choice)

**Reference choice: REST with Fastify**

| Factor | REST (Fastify) | GraphQL | Decision |
|--------|---------------|---------|----------|
| **Complexity** | Low | Medium | REST |
| **Consumer compatibility** | Universal | Needs client | REST |
| **n8n integration** | Native HTTP node | Needs plugin | REST |
| **Telegram bot** | Simple HTTP calls | Overkill | REST |
| **Next.js** | `fetch()` / SWR | Apollo client | REST |
| **Performance** | Excellent (Fastify) | Good | REST |
| **Caching** | HTTP native | Custom | REST |
| **Future needs** | Sufficient | Not needed | **REST** |

**Decision rationale:**
- **Fastify** — fastest Node.js framework, built-in schema validation, excellent plugin system, low overhead
- **REST** — all consumers (Next.js, Telegram bot, n8n) work natively with REST; no GraphQL complexity needed for a personal project
- **Schema validation** — use Fastify's built-in JSON schema for request/response validation

### 4.6 Database Schema

> **Canonical schema lives in [`data-schema.md`](./data-schema.md).** The DDL below
> is a *reference* table layout for the Node.js/Postgres deployment and is kept
> here so the architecture diagrams stay self-contained. If `data-schema.md` and
> this section disagree, `data-schema.md` is authoritative (it uses UUIDv7
> primary keys, JSONB columns throughout, and supports both Postgres and SQLite
> backends). Treat this section as a quick-look DDL only.

```sql
-- Core tables (reference layout — see data-schema.md for the canonical schema)
CREATE TABLE sources (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL,
  url             TEXT NOT NULL,
  adapter         TEXT NOT NULL,
  topics          TEXT[] NOT NULL,
  schedule        TEXT NOT NULL,
  enabled         BOOLEAN DEFAULT true,
  config          JSONB DEFAULT '{}',
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  last_scraped_at TIMESTAMPTZ,
  scrape_count    INTEGER DEFAULT 0,
  error_count     INTEGER DEFAULT 0
);

CREATE TABLE articles (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_id       TEXT NOT NULL REFERENCES sources(id),
  title           TEXT NOT NULL,
  url             TEXT NOT NULL UNIQUE,
  content         TEXT,
  content_html    TEXT,
  summary         TEXT,
  author          TEXT,
  published_at    TIMESTAMPTZ,
  scraped_at      TIMESTAMPTZ DEFAULT NOW(),
  quality_score   INTEGER DEFAULT 0,
  content_hash    TEXT NOT NULL,
  language        TEXT DEFAULT 'en',
  read_time_minutes INTEGER,
  status          TEXT DEFAULT 'active',  -- active, archived, duplicate
  metadata        JSONB DEFAULT '{}',
  
  -- Full-text search
  search_vector   tsvector
);

CREATE INDEX idx_articles_source ON articles(source_id);
CREATE INDEX idx_articles_published ON articles(published_at DESC);
CREATE INDEX idx_articles_topics ON articles USING GIN(
  (metadata->'topics')
);
CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);

-- Topics many-to-many
CREATE TABLE article_topics (
  article_id UUID REFERENCES articles(id) ON DELETE CASCADE,
  topic      TEXT NOT NULL,
  PRIMARY KEY (article_id, topic)
);

CREATE INDEX idx_article_topics_topic ON article_topics(topic);

-- Images
CREATE TABLE article_images (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  article_id  UUID NOT NULL REFERENCES articles(id) ON DELETE CASCADE,
  url         TEXT NOT NULL,
  alt_text    TEXT,
  is_primary  BOOLEAN DEFAULT false,
  fetched_at  TIMESTAMPTZ
);

-- Raw data cache (for debugging and re-extraction)
CREATE TABLE raw_data (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_id   TEXT NOT NULL,
  job_id      TEXT NOT NULL,
  url         TEXT NOT NULL,
  content_type TEXT,
  body        TEXT,
  fetched_at  TIMESTAMPTZ DEFAULT NOW(),
  expires_at  TIMESTAMPTZ  -- auto-purge after 24h
);

CREATE INDEX idx_raw_data_source ON raw_data(source_id);
CREATE INDEX idx_raw_data_expires ON raw_data(expires_at);

-- Jobs / queue state (supplement to BullMQ Redis)
CREATE TABLE job_log (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id      TEXT NOT NULL,
  source_id   TEXT NOT NULL,
  adapter     TEXT NOT NULL,
  status      TEXT NOT NULL,  -- queued, processing, completed, failed, dead
  started_at  TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  duration_ms INTEGER,
  articles_found INTEGER DEFAULT 0,
  articles_saved INTEGER DEFAULT 0,
  error_message TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_job_log_source ON job_log(source_id);
CREATE INDEX idx_job_log_status ON job_log(status);
CREATE INDEX idx_job_log_created ON job_log(created_at DESC);

-- Errors / dead letter queue
CREATE TABLE errors (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  job_id      TEXT,
  source_id   TEXT NOT NULL,
  adapter     TEXT NOT NULL,
  category    TEXT NOT NULL,  -- transient_network, permanent_parse, etc.
  message     TEXT NOT NULL,
  stack_trace TEXT,
  context     JSONB DEFAULT '{}',
  resolved    BOOLEAN DEFAULT false,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_errors_source ON errors(source_id);
CREATE INDEX idx_errors_category ON errors(category);
CREATE INDEX idx_errors_created ON errors(created_at DESC);
CREATE INDEX idx_errors_unresolved ON errors(resolved) WHERE resolved = false;

-- Trigger for full-text search update
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector := 
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(NEW.summary, '')), 'C');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_update
  BEFORE INSERT OR UPDATE ON articles
  FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- Auto-cleanup of old raw_data
CREATE OR REPLACE FUNCTION cleanup_expired_raw_data()
RETURNS void AS $$
BEGIN
  DELETE FROM raw_data WHERE expires_at < NOW();
END;
$$ LANGUAGE plpgsql;
```


---

## 5. Docker Architecture

### 5.1 docker-compose.yml

```yaml
# docker-compose.yml — Jarvis HQ
# Usage: docker-compose up -d
#        docker-compose logs -f scraper

version: "3.9"

services:
  # ─── PostgreSQL Database ──────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    container_name: jarvis-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: jarvis_hq
      POSTGRES_USER: ${DB_USER:-jarvis}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-jarvis_dev_pass}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    ports:
      - "${DB_PORT:-5432}:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-jarvis} -d jarvis_hq"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - jarvis-network

  # ─── Redis (Queue + Rate Limiter + Cache) ────────────────────
  redis:
    image: redis:7-alpine
    container_name: jarvis-redis
    restart: unless-stopped
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    ports:
      - "${REDIS_PORT:-6379}:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - jarvis-network

  # ─── Main Scraper Application ────────────────────────────────
  scraper:
    build:
      context: .
      dockerfile: Dockerfile
      target: scraper
    container_name: jarvis-scraper
    restart: unless-stopped
    environment:
      NODE_ENV: ${NODE_ENV:-production}
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: jarvis_hq
      DB_USER: ${DB_USER:-jarvis}
      DB_PASSWORD: ${DB_PASSWORD:-jarvis_dev_pass}
      DB_POOL_SIZE: ${DB_POOL_SIZE:-10}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD:-}
      LOG_LEVEL: ${LOG_LEVEL:-info}
      GITHUB_TOKEN: ${GITHUB_TOKEN:-}
      YOUTUBE_API_KEY: ${YOUTUBE_API_KEY:-}
      NEWSAPI_KEY: ${NEWSAPI_KEY:-}
      GOOGLE_SEARCH_API_KEY: ${GOOGLE_SEARCH_API_KEY:-}
      GOOGLE_SEARCH_CX: ${GOOGLE_SEARCH_CX:-}
      MAX_CONCURRENT_FETCHES: ${MAX_CONCURRENT_FETCHES:-5}
      DEFAULT_RATE_LIMIT_RPM: ${DEFAULT_RATE_LIMIT_RPM:-10}
      ENABLE_HEADLESS: ${ENABLE_HEADLESS:-false}
      TRAFILatura_URL: ${TRAFILatura_URL:-http://trafilatura:8080}
    volumes:
      - ./config/sources:/app/config/sources:ro
      - ./logs:/app/logs
      - ./data:/app/data
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - jarvis-network
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 128M

  # ─── Trafilatura Python Microservice (optional fallback) ──────
  trafilatura:
    build:
      context: .
      dockerfile: Dockerfile.trafilatura
    container_name: jarvis-trafilatura
    restart: unless-stopped
    environment:
      PORT: 8080
    expose:
      - "8080"
    deploy:
      resources:
        limits:
          memory: 256M
    networks:
      - jarvis-network
    profiles:
      - full    # Only start with: docker-compose --profile full up

  # ─── API Server ──────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: api
    container_name: jarvis-api
    restart: unless-stopped
    ports:
      - "${API_PORT:-3000}:3000"
    environment:
      NODE_ENV: ${NODE_ENV:-production}
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: jarvis_hq
      DB_USER: ${DB_USER:-jarvis}
      DB_PASSWORD: ${DB_PASSWORD:-jarvis_dev_pass}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      API_PORT: 3000
      CORS_ORIGIN: ${CORS_ORIGIN:-*}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - jarvis-network
    deploy:
      resources:
        limits:
          memory: 256M

  # ─── Adminer (DB Management UI) ──────────────────────────────
  adminer:
    image: adminer:latest
    container_name: jarvis-adminer
    restart: unless-stopped
    ports:
      - "${ADMINER_PORT:-8080}:8080"
    depends_on:
      - postgres
    networks:
      - jarvis-network
    profiles:
      - dev    # Only in development

  # ─── Redis Insight (Redis GUI) ───────────────────────────────
  redisinsight:
    image: redis/redisinsight:latest
    container_name: jarvis-redisinsight
    restart: unless-stopped
    ports:
      - "${REDIS_INSIGHT_PORT:-5540}:5540"
    volumes:
      - redisinsight_data:/data
    depends_on:
      - redis
    networks:
      - jarvis-network
    profiles:
      - dev

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  redisinsight_data:
    driver: local

networks:
  jarvis-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### 5.2 Dockerfile (Multi-stage)

```dockerfile
# ─── Dockerfile — Jarvis HQ (Multi-stage build) ─────────────────
# Stage 1: Base dependencies
FROM node:20-alpine AS base
WORKDIR /app
RUN apk add --no-cache dumb-init
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Stage 2: Scraper (full application)
FROM base AS scraper
WORKDIR /app
COPY --from=base /app/node_modules ./node_modules
COPY . .
RUN npm run build
EXPOSE 3000
USER node
CMD ["dumb-init", "node", "dist/index.js"]

# Stage 3: API only (lightweight, no scraping workers)
FROM base AS api
WORKDIR /app
COPY --from=base /app/node_modules ./node_modules
COPY . .
RUN npm run build
EXPOSE 3000
USER node
CMD ["dumb-init", "node", "dist/api-server.js"]
```

```dockerfile
# ─── Dockerfile.trafilatura ─────────────────────────────────────
FROM python:3.12-slim
WORKDIR /app
RUN pip install --no-cache-dir trafilatura[all] uvicorn fastapi
COPY trafilatura-service.py .
EXPOSE 8080
CMD ["uvicorn", "trafilatura-service:app", "--host", "0.0.0.0", "--port", "8080"]
```

```python
# trafilatura-service.py — FastAPI wrapper for Trafilatura
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, HttpUrl
import trafilatura
import requests

app = FastAPI(title="Trafilatura Extraction Service")

class ExtractRequest(BaseModel):
    url: HttpUrl
    include_images: bool = False
    include_comments: bool = False

class ExtractResponse(BaseModel):
    title: str | None
    author: str | None
    date: str | None
    content_text: str | None
    content_html: str | None
    language: str | None
    success: bool
    error: str | None = None

@app.post("/extract", response_model=ExtractResponse)
async def extract_article(request: ExtractRequest):
    try:
        downloaded = trafilatura.fetch_url(str(request.url))
        if not downloaded:
            raise HTTPException(status_code=400, detail="Failed to fetch URL")
        
        result = trafilatura.extract(
            downloaded,
            output_format='json',
            with_metadata=True,
            include_images=request.include_images,
            include_comments=request.include_comments,
            url=str(request.url)
        )
        
        if not result:
            return ExtractResponse(success=False, error="Extraction returned empty")
        
        data = trafilatura.utils.load_json(result)
        return ExtractResponse(
            title=data.get('title'),
            author=data.get('author'),
            date=data.get('date'),
            content_text=data.get('text'),
            content_html=data.get('raw-html'),
            language=data.get('language'),
            success=True
        )
    except Exception as e:
        return ExtractResponse(success=False, error=str(e))

@app.get("/health")
async def health():
    return {"status": "ok"}
```

### 5.3 Environment Variables (.env)

```bash
# ─── Jarvis HQ Environment Configuration ────────────────────────
# Copy this file to .env and fill in your values

# ── Database ───────────────────────────────────────────────────
DB_USER=jarvis
DB_PASSWORD=change_me_in_production
DB_PORT=5432
DB_POOL_SIZE=10

# ── Redis ──────────────────────────────────────────────────────
REDIS_PORT=6379
REDIS_PASSWORD=

# ── API ────────────────────────────────────────────────────────
API_PORT=3000
NODE_ENV=production
LOG_LEVEL=info
CORS_ORIGIN=http://localhost:3001

# ── Scraping ───────────────────────────────────────────────────
MAX_CONCURRENT_FETCHES=5
DEFAULT_RATE_LIMIT_RPM=10
ENABLE_HEADLESS=false

# ── Source API Keys (stored in 1Password, referenced here) ─────
GITHUB_TOKEN=ghp_xxxxxxxxxxxx
YOUTUBE_API_KEY=AIzaSxxxxxxxxxxxx
NEWSAPI_KEY=xxxxxxxxxxxxxxxx
GOOGLE_SEARCH_API_KEY=AIzaSxxxxxxxxxxxx
GOOGLE_SEARCH_CX=xxxxxxxxxxxxxxx

# ── n8n Webhook (optional) ─────────────────────────────────────
N8N_WEBHOOK_URL=

# ── Telegram Bot (optional) ────────────────────────────────────
TELEGRAM_BOT_TOKEN=

# ── Dev Tools ──────────────────────────────────────────────────
ADMINER_PORT=8080
REDIS_INSIGHT_PORT=5540
```

### 5.4 Directory Structure

```
jarvis-hq/
├── docker-compose.yml
├── docker-compose.override.yml      # Dev overrides (gitignored)
├── Dockerfile
├── Dockerfile.trafilatura
├── .env                             # Environment variables (gitignored)
├── .env.example                     # Template
├── package.json
├── tsconfig.json
│
├── config/
│   └── sources/                     # One YAML per source
│       ├── _template.yaml
│       ├── techcrunch-ai.yaml
│       ├── hackernews-top.yaml
│       ├── github-releases.yaml
│       └── youtube-channels.yaml
│
├── src/
│   ├── index.ts                     # Main entry (scheduler + workers)
│   ├── api-server.ts                # API-only entry
│   ├── registry/
│   │   ├── source-registry.ts
│   │   └── schemas.ts
│   ├── scheduler/
│   │   └── scheduler.ts
│   ├── queue/
│   │   ├── job-queue.ts
│   │   ├── worker.ts
│   │   └── types.ts
│   ├── adapters/
│   │   ├── adapter-factory.ts
│   │   ├── adapter-interface.ts
│   │   ├── rss-adapter.ts
│   │   ├── api-adapter.ts
│   │   ├── sitemap-adapter.ts
│   │   ├── html-article-adapter.ts
│   │   ├── github-release-adapter.ts
│   │   ├── youtube-adapter.ts
│   │   ├── social/
│   │   │   ├── bluesky-adapter.ts
│   │   │   ├── mastodon-adapter.ts
│   │   │   └── reddit-adapter.ts
│   │   └── search-discovery-adapter.ts
│   ├── extractors/
│   │   ├── readability-extractor.ts
│   │   ├── jsonld-extractor.ts
│   │   └── trafilatura-extractor.ts
│   ├── quality/
│   │   ├── quality-engine.ts
│   │   └── deduplicator.ts
│   ├── rate-limiter/
│   │   └── rate-limiter.ts
│   ├── retry/
│   │   ├── error-classifier.ts
│   │   └── retry-handler.ts
│   ├── storage/
│   │   ├── database.ts
│   │   ├── article-repository.ts
│   │   └── migrations/
│   ├── api/
│   │   ├── server.ts
│   │   ├── routes/
│   │   │   ├── articles.ts
│   │   │   ├── sources.ts
│   │   │   ├── jobs.ts
│   │   │   ├── search.ts
│   │   │   ├── feed.ts
│   │   │   └── health.ts
│   │   └── plugins/
│   ├── logger/
│   │   └── logger.ts
│   └── utils/
│       ├── fetch.ts
│       ├── hash.ts
│       └── date.ts
│
├── init/                            # Database init scripts
│   └── 001-schema.sql               # Schema creation (runs on first start)
│
├── trafilatura-service.py           # Python microservice
├── logs/                            # Log output (Docker volume)
├── data/                            # Data files (Docker volume)
└── scripts/
    ├── add-source.sh                # Interactive source creation
    ├── test-source.sh               # Test a single source
    └── backup.sh                    # Database backup
```

---

## 6. Configuration

### 6.1 Source Registry YAML Schema (Complete)

```yaml
# ─── Source Configuration Specification ─────────────────────────
# Every source must follow this schema. One file per source in
# config/sources/{source-id}.yaml

# ── Required Fields ────────────────────────────────────────────
id: string                    # Unique identifier, lowercase, kebab-case
                              # Examples: techcrunch-ai, hackernews-top
                              # Pattern: ^[a-z0-9-]+$

name: string                  # Human-readable name
                              # Example: "TechCrunch AI"

enabled: boolean              # Whether this source is active
                              # Default: true

url: string                   # Primary URL to scrape
                              # For RSS: feed URL
                              # For API: endpoint URL
                              # For sitemap: sitemap URL
                              # For GitHub: repo URL
                              # For YouTube: channel URL or RSS feed
                              # Must be valid URL

adapter: enum                 # Which adapter handles this source
                              # One of:
                              #   rss, api, sitemap, html_article,
                              #   github_release, youtube, social,
                              #   search_discovery

topics: string[]              # Topics this source covers
                              # Must have at least one
                              # Valid topics: ai, gaming, tech, sports,
                              #   world_news, science, business,
                              #   entertainment, politics
                              # Custom topics allowed

schedule: string              # Cron expression for scraping frequency
                              # Examples:
                              #   "*/15 * * * *"   — Every 15 minutes
                              #   "*/30 * * * *"   — Every 30 minutes
                              #   "0 * * * *"      — Every hour
                              #   "0 */6 * * *"    — Every 6 hours
                              #   "0 0 * * *"      — Daily at midnight
                              #   "0 0 * * 1"      — Weekly on Monday

# ── Optional Fields ────────────────────────────────────────────
rateLimit:                    # Source-specific rate limiting
  requestsPerMinute: number   # Max requests per minute (1-1000)
                              # Default: 10
  respectRobotsTxt: boolean   # Whether to fetch and obey robots.txt
                              # Default: true
  crawlDelay: number          # Additional configured crawl-delay (seconds)
                              # Effective delay is max(configured delay, robots.txt crawl-delay)

selectors: map<string, string> # Adapter-specific field mappings
                              # Meaning depends on adapter:
                              #
                              # RSS adapter:
                              #   titleField: "title"
                              #   linkField: "link"
                              #   pubDateField: "pubDate"
                              #   contentField: "description"
                              #
                              # API adapter:
                              #   itemsPath: "articles"        # JSON path to array
                              #   titlePath: "title"           # Field within item
                              #   urlPath: "url"
                              #   datePath: "publishedAt"
                              #   contentPath: "body"
                              #   authorPath: "author.name"
                              #
                              # Sitemap adapter:
                              #   urlPattern: "/articles/"     # URL filter
                              #   dateField: "lastmod"
                              #
                              # HTML adapter:
                              #   titleSelector: "h1"
                              #   contentSelector: "article"
                              #   dateSelector: "time"
                              #   authorSelector: ".author"
                              #
                              # GitHub adapter:
                              #   (none needed, uses API response)
                              #
                              # YouTube adapter:
                              #   channelId: "UC..."           # Extracted from URL
                              #
                              # Social adapter:
                              #   platform: "bluesky"          # Social platform
                              #   handle: "username"           # Account handle
                              #   postLimit: 30                # Max posts per run

headers: map<string, string>  # Custom HTTP headers
                              # Example:
                              #   Accept: application/json
                              #   User-Agent: JarvisHQ/1.0

auth:                         # Authentication configuration
  type: enum                  # One of: bearer, api_key, basic, none
                              # Default: none
  envKey: string              # Environment variable name holding secret
                              # Example: "NEWSAPI_KEY"
                              # The actual value is NOT stored in this file
                              # It is read from the environment variable
                              # referenced by envKey at runtime

tags: string[]                # Arbitrary tags for organization
                              # Examples: breaking, weekly-roundup,
                              #   official, community
                              # Default: []
```

### 6.2 Topic Configuration

```yaml
# config/topics.yaml — Topic definitions and defaults

topics:
  ai:
    name: "Artificial Intelligence"
    displayOrder: 1
    defaultSchedule: "*/30 * * * *"
    keywords: ["AI", "machine learning", "LLM", "neural network", "GPT"]
    icon: "brain"
    color: "#8B5CF6"

  gaming:
    name: "Gaming"
    displayOrder: 2
    defaultSchedule: "0 * * * *"
    keywords: ["game", "gaming", "esports", "console", "steam"]
    icon: "gamepad"
    color: "#EF4444"

  tech:
    name: "Technology"
    displayOrder: 3
    defaultSchedule: "*/30 * * * *"
    keywords: ["tech", "software", "hardware", "startup"]
    icon: "cpu"
    color: "#3B82F6"

  sports:
    name: "Sports"
    displayOrder: 4
    defaultSchedule: "0 */3 * * *"
    keywords: ["football", "basketball", "soccer", "tennis", "olympics"]
    icon: "trophy"
    color: "#10B981"

  world_news:
    name: "World News"
    displayOrder: 5
    defaultSchedule: "0 */2 * * *"
    keywords: ["world", "international", "politics", "breaking"]
    icon: "globe"
    color: "#F59E0B"
```

### 6.3 Rate Limit Configuration

```yaml
# config/rate-limits.yaml — Per-domain rate limits

# Global defaults
defaults:
  requestsPerMinute: 10
  concurrentRequests: 5
  respectRobotsTxt: true
  userAgent: "JarvisHQ/1.0 (Personal News Bot; +https://yourdomain.com/about)"

# Domain-specific overrides
# These set local minimums; never use them to go faster than robots.txt
domains:
  "github.com":
    requestsPerMinute: 60          # 3600/hour with auth
    authRequired: true

  "news.ycombinator.com":
    requestsPerMinute: 30

  "api.allorigins.win":
    requestsPerMinute: 20          # Free tier limit

  "www.googleapis.com":
    requestsPerMinute: 100         # YouTube/Data API
    quotaReset: "daily"

  "reddit.com":
    requestsPerMinute: 30
    oauthRequired: true

  "bsky.social":
    requestsPerMinute: 3000        # Bluesky is generous

  "inference.fably.eu":
    requestsPerMinute: 10          # Trafilatura hosting

# Platform-specific limits
platforms:
  bluesky:
    requestsPerMinute: 100
    endpoints:
      getAuthorFeed: 100
      getProfile: 60

  mastodon:
    requestsPerMinute: 300         # Per-instance, usually generous
```

### 6.4 Retry Configuration

```yaml
# config/retry.yaml — Retry policies by error category

policies:
  transient_network:
    maxRetries: 3
    backoff:
      type: exponential
      initialMs: 5000
      maxMs: 60000
      multiplier: 2
      jitter: true               # Add randomness to prevent thundering herd

  transient_rate_limit:
    maxRetries: 5
    backoff:
      type: exponential
      initialMs: 60000           # Start at 1 minute
      maxMs: 3600000             # Cap at 1 hour
      multiplier: 2
      jitter: true
    honorRetryAfterHeader: true  # Respect HTTP Retry-After if present

  transient_server:
    maxRetries: 3
    backoff:
      type: exponential
      initialMs: 60000
      maxMs: 300000              # Cap at 5 minutes
      multiplier: 2
      jitter: true

  permanent:
    maxRetries: 0                # No retries for permanent errors
    dlqImmediately: true

# Global settings
global:
  maxTotalRetries: 10            # Hard cap across all retries
  dlqAfterMaxRetries: true       # Move to DLQ after exhausting retries
  alertOnSystemicFailure: true
  systemicFailureThreshold:      # Alert if >N failures from same source in 1 hour
    count: 5
    windowMinutes: 60
```

---

## 7. Extensibility

### 7.1 Adding a New Adapter

To add a new adapter (e.g., a "Newsletter" adapter that parses email newsletters):

**Step 1: Define the adapter interface**

```typescript
// src/adapters/adapter-interface.ts
export interface SourceAdapter {
  execute(job: ScrapeJob): Promise<RawArticle[]>;
}

export interface AdapterConstructor {
  new (): SourceAdapter;
}
```

**Step 2: Implement the adapter**

```typescript
// src/adapters/newsletter-adapter.ts
import { SourceAdapter } from './adapter-interface';

export class NewsletterAdapter implements SourceAdapter {
  async execute(job: ScrapeJob): Promise<RawArticle[]> {
    // Your custom logic here
    // e.g., fetch from IMAP, parse email content, extract article
    const articles: RawArticle[] = [];
    // ... implementation
    return articles;
  }
}
```

**Step 3: Register in the factory**

```typescript
// src/adapters/adapter-factory.ts
import { NewsletterAdapter } from './newsletter-adapter';

export class AdapterFactory {
  private adapters: Map<string, SourceAdapter> = new Map([
    ['rss', new RssAdapter()],
    ['api', new ApiAdapter()],
    ['sitemap', new SitemapAdapter()],
    ['html_article', new HtmlArticleAdapter()],
    ['github_release', new GitHubReleaseAdapter()],
    ['youtube', new YouTubeAdapter()],
    ['social', new SocialAdapter()],
    ['search_discovery', new SearchDiscoveryAdapter()],
    ['newsletter', new NewsletterAdapter()],  // ← Add here
  ]);

  get(type: string): SourceAdapter {
    const adapter = this.adapters.get(type);
    if (!adapter) throw new Error(`Unknown adapter type: ${type}`);
    return adapter;
  }
}
```

**Step 4: Update the schema to accept the new type**

```typescript
// src/registry/schemas.ts
adapter: z.enum([
  'rss', 'api', 'sitemap', 'html_article', 'github_release',
  'youtube', 'social', 'search_discovery', 'newsletter',  // ← Add here
]),
```

**Step 5: Create a source config**

```yaml
# config/sources/example-newsletter.yaml
id: example-newsletter
name: "Example Weekly Newsletter"
adapter: newsletter
url: "imap://mail.example.com/inbox"
topics: [tech, ai]
schedule: "0 9 * * 1"   # Every Monday at 9am
selectors:
  folder: "Newsletters"
  subjectFilter: "Weekly Roundup"
  maxEmails: 1
auth:
  type: basic
  envKey: NEWSLETTER_IMAP_PASSWORD
```

**No other changes needed** — the scheduler, queue, and pipeline will pick up the new adapter automatically.

### 7.2 Adding a New Source Type

Adding a source type follows the same pattern as adding an adapter (above), since each source type maps 1:1 to an adapter. However, for composite types (like "a blog that has both RSS and HTML pages"):

**Composite source pattern:**

```yaml
# config/sources/multi-format-blog.yaml
id: multi-format-blog
name: "A Blog With RSS and Direct HTML"
adapter: rss                    # Primary adapter
url: "https://example.com/feed"
topics: [ai]
schedule: "*/30 * * * *"
secondaryAdapter: html_article  # Fallback if RSS is empty/incomplete
secondaryUrl: "https://example.com/articles"
mergeStrategy: "deduplicate"    # Merge articles from both, remove dups
```

The scheduler would create two jobs per tick (one for each adapter) and the deduplicator handles the rest.

### 7.3 Adding a New Topic

**Step 1: Add topic definition**

```yaml
# config/topics.yaml
topics:
  space:                        # ← New topic
    name: "Space & Astronomy"
    displayOrder: 6
    defaultSchedule: "0 */6 * * *"
    keywords: ["space", "NASA", "rocket", "satellite", "Mars", "astronomy"]
    icon: "rocket"
    color: "#6366F1"
```

**Step 2: Create sources for the topic**

```bash
# Use the helper script
./scripts/add-source.sh
# Enter topic: space
# Enter source name: NASA Blog
# Enter URL: https://www.nasa.gov/news-release/feed/
# Select adapter: rss
# Enter schedule: 0 */6 * * *
```

**Step 3: No code changes needed** — the pipeline automatically routes articles tagged with `space` to the correct topic index.

### 7.4 Plugin Architecture

For third-party or optional extensions:

```typescript
// src/plugins/plugin-interface.ts
export interface JarvisPlugin {
  name: string;
  version: string;
  
  // Lifecycle hooks
  onInit?(context: PluginContext): Promise<void>;
  onBeforeScrape?(job: ScrapeJob): Promise<ScrapeJob | null>;   // Return null to skip
  onAfterScrape?(job: ScrapeJob, articles: RawArticle[]): Promise<RawArticle[]>;
  onBeforeStore?(article: RawArticle): Promise<RawArticle | null>;
  onShutdown?(): Promise<void>;
}

export interface PluginContext {
  registerAdapter(type: string, adapter: SourceAdapter): void;
  registerFilter(filter: ArticleFilter): void;
  registerWebhook(url: string, events: string[]): void;
  getConfig(key: string): any;
}
```

**Example plugin: Send to n8n webhook**

```typescript
// src/plugins/n8n-webhook-plugin.ts
export class N8nWebhookPlugin implements JarvisPlugin {
  name = 'n8n-webhook';
  version = '1.0.0';
  private webhookUrl: string;

  constructor() {
    this.webhookUrl = process.env.N8N_WEBHOOK_URL || '';
  }

  async onAfterScrape(_job: ScrapeJob, articles: RawArticle[]): Promise<RawArticle[]> {
    if (!this.webhookUrl) return articles;
    
    for (const article of articles) {
      await fetch(this.webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          event: 'article.found',
          data: article,
          timestamp: new Date().toISOString(),
        }),
      });
    }
    return articles;
  }
}
```

**Plugin registration:**

```typescript
// src/plugins/plugin-manager.ts
export class PluginManager {
  private plugins: JarvisPlugin[] = [];

  register(plugin: JarvisPlugin): void {
    this.plugins.push(plugin);
  }

  async init(context: PluginContext): Promise<void> {
    for (const plugin of this.plugins) {
      await plugin.onInit?.(context);
    }
  }

  async executeHook<T>(
    hookName: keyof JarvisPlugin,
    ...args: any[]
  ): Promise<T> {
    let result = args[0];
    for (const plugin of this.plugins) {
      const hook = plugin[hookName] as Function;
      if (hook) {
        result = await hook.apply(plugin, args);
        if (result === null) break; // Plugin cancelled the operation
        args[0] = result;
      }
    }
    return result;
  }
}
```

**Plugins are loaded dynamically from the `plugins/` directory** — drop a `.ts` file there, it gets auto-registered on startup.

---

## Appendix: Quick Reference

### Adapter Decision Tree

```
What kind of source?
│
├─ RSS/Atom/JSON Feed? → RSS Adapter
│
├─ REST API or GraphQL? → API Adapter
│
├─ Blog/article website (no RSS)? → HTML Article Adapter
│
├─ Sitemap with article URLs? → Sitemap Adapter → HTML Adapter
│
├─ GitHub repository releases? → GitHub Release Adapter
│
├─ YouTube channel? → YouTube Adapter (RSS-first, API fallback)
│
├─ Bluesky/Mastodon/Reddit? → Social Adapter
│
├─ Search-based discovery? → Search Discovery Adapter
│
└─ Something else? → Write a custom adapter (see 7.1)
```

### Key Metrics to Track

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| Jobs queued (depth) | BullMQ + Redis | > 1000 |
| Job processing time | BullMQ events | > 60s average |
| Failed jobs / hour | errors table | > 20 |
| Same-source failures | errors table | > 5 in 1 hour |
| Articles scraped / day | articles table | Sudden drop to 0 |
| DB connection pool usage | pg_stat_activity | > 80% |
| Redis memory usage | Redis INFO | > 200MB |
| API response time | Fastify metrics | > 500ms p95 |

### Security Checklist

- [ ] All secrets in environment variables (never in code)
- [ ] API keys stored in 1Password, referenced by name in config
- [ ] No sensitive data in logs (Pino redaction configured)
- [ ] Database password is strong and unique
- [ ] Redis has AUTH password in production
- [ ] robots.txt respected for all domains
- [ ] User-Agent header identifies the bot
- [ ] Rate limits are conservative (never aggressive)
- [ ] No credential storage in Docker layers
- [ ] `.env` file is gitignored
- [ ] CORS restricted to known origins in production

---

*Jarvis HQ Architecture Plan — Section F*
*Version 1.0 — Designed for single-user local deployment*
*Extensible to multi-user and cloud deployment*
