# Pipeline Reference (Index)

> The full pipeline reference was split into per-section files for progressive
> disclosure. This file is now a thin INDEX — it lists each stage, points to
> the file that contains the full design, and keeps the cross-cutting
> appendices (env config, source registry example, topic config) inline.
>
> **For the full overview + flow diagram + data model**, read Section 1 below.
> **For a specific stage's design**, jump to the listed file.

## Table of Contents

- Section 1: Pipeline Stages Overview (inline below)
- Section 2: Stage-by-Stage Design — see split files:
  - [pipeline-collection.md](./pipeline-collection.md) — Stages 1–2
  - [pipeline-processing.md](./pipeline-processing.md) — Stages 3–6
  - [pipeline-distribution.md](./pipeline-distribution.md) — Stages 7–10
  - [pipeline-orchestration.md](./pipeline-orchestration.md) — Section 3 (Orchestration) and Section 4 (Output Formats)
- Appendix A: Environment Configuration (inline below)
- Appendix B: Source Registry Example (inline below)
- Appendix C: Topic Configuration (inline below)

---
## 1. Pipeline Stages Overview

### Architecture Philosophy

The Jarvis HQ pipeline follows a **multi-modal processing pattern**: it supports both **real-time streaming** (for high-priority breaking news) and **batch processing** (for scheduled RSS/API collection). Each stage is idempotent, independently scalable, and fully observable. The design prioritises data integrity, fault tolerance, and horizontal scalability.

### Core Data Model: FeedItem

Every item flowing through the pipeline is represented as a `FeedItem` -- a normalised, schema-validated object that accumulates metadata as it progresses through stages.

```typescript
interface FeedItem {
  // === Core Identity (set at Collection) ===
  id: string;                    // UUID v4, generated on first ingestion
  sourceId: string;              // Reference to source registry entry
  canonicalUrl: string;          // Normalised, deduplicated URL
  rawUrl: string;                // Original URL as collected
  contentHash: string;           // SHA-256 of normalised content

  // === Content Fields (set at Collection/Enrichment) ===
  title: string;
  summary: string | null;
  content: string | null;        // Full article body (HTML or plain text)
  contentText: string | null;    // Plain-text extraction for processing
  author: string | null;
  language: string;              // ISO 639-1 code (e.g., 'en', 'de')

  // === Media (set at Enrichment) ===
  imageUrl: string | null;
  videoUrl: string | null;
  readingTimeMinutes: number | null;

  // === Temporal (set at Collection/Enrichment) ===
  publishedAt: string;           // ISO 8601 UTC
  collectedAt: string;           // ISO 8601 UTC
  processedAt: string | null;    // ISO 8601 UTC -- set at Storage
  freshnessScore: number;        // 0.0 -- 1.0, recalculated hourly

  // === Classification (set at Categorisation) ===
  topics: Topic[];               // ['ai', 'gaming', 'tech'] etc.
  tags: string[];                // Auto + manual tags
  contentType: ContentType;      // 'article' | 'release' | 'video' | 'social' | 'podcast'
  importance: number;            // 0 -- 100

  // === Scoring (set at Scoring/Ranking) ===
  sourceQualityScore: number;    // 0 -- 100, from source registry
  contentQualityScore: number;   // 0 -- 100, from content analysis
  compositeScore: number;        // 0 -- 100, weighted formula
  tier: 1 | 2 | 3 | 4 | 5;      // Derived from composite score

  // === Deduplication (set at Deduplication) ===
  duplicateGroupId: string | null;
  isPrimary: boolean;            // True if this is the canonical copy

  // === Pipeline State ===
  pipelineStage: PipelineStage;  // Current stage in pipeline
  processingAttempts: number;    // Retry counter
  lastError: string | null;
  metadata: Record<string, unknown>;
}

type Topic = 'ai' | 'gaming' | 'tech' | 'sports' | 'world' | 'science' | 'business';
type ContentType = 'article' | 'release' | 'video' | 'social' | 'podcast';
type PipelineStage = 
  | 'collected' | 'validated' | 'deduplicated' | 'enriched' 
  | 'scored' | 'summarised' | 'categorised' | 'ranked' | 'stored' | 'distributed';
```

### Pipeline Stages Summary

| Stage | Input | Output | Description | Mode |
|-------|-------|--------|-------------|------|
| **Collection** | Source registry (`SourceConfig[]`) | Raw `FeedItem[]` | Fetch from RSS/API/HTML/YouTube/GitHub with per-source adapters | Batch + Real-time |
| **Validation** | Raw `FeedItem[]` | Validated `FeedItem[]` | Schema check, URL validation, date parsing, reject malformed items | Streaming |
| **Deduplication** | Validated `FeedItem[]` | Unique `FeedItem[]` | 5-stage dedup: URL, canonical URL, title Levenshtein, MinHash, time window | Streaming |
| **Enrichment** | Unique `FeedItem[]` | Enriched `FeedItem[]` | Author extraction, date normalisation, language detection, image extraction | Streaming |
| **Scoring** | Enriched `FeedItem[]` | Scored `FeedItem[]` | Source quality + content quality + freshness decay + tier assignment | Streaming |
| **Summarisation** | Scored `FeedItem[]` | Summarised `FeedItem[]` | Extractive summary (first N sentences) + optional abstractive LLM summary | Streaming |
| **Categorisation** | Summarised `FeedItem[]` | Categorised `FeedItem[]` | Topic keyword matching, tag assignment, content type classification | Streaming |
| **Ranking** | Categorised `FeedItem[]` | Ranked `FeedItem[]` | Composite sort, breaking news boost, diversity injection, tier filtering | Batch (post-collection) |
| **Storage** | Ranked `FeedItem[]` | Stored `FeedItem[]` | PostgreSQL upsert, related item linking, index update, archival trigger | Streaming |
| **Distribution** | Stored `FeedItem[]` | Multi-channel output | API update, Telegram notification, digest queue, n8n webhook, cache purge | Event-driven |

### Pipeline Flow Diagram

```
  Source Registry          Scheduler (cron + webhook)
        |                           |
        v                           v
  [ ADAPTERS ]  -->  RSS | API | HTML | YouTube | GitHub | Social
        |
        v
+------------------+
| 1. COLLECTION    |  <-- Rate limiting, parallel workers, retry logic
|    (FeedItem[])  |
+------------------+
        |
        v
+------------------+
| 2. VALIDATION    |  <-- Schema check, URL validation, language detect
|    (FeedItem[])  |      [REJECT] --> Dead Letter Queue
+------------------+
        |
        v
+------------------+
| 3. DEDUPLICATION |  <-- 5-stage hash/similarity pipeline
|    (FeedItem[])  |      [DUPLICATE] --> Merge / Archive
+------------------+
        |
        v
+------------------+
| 4. ENRICHMENT    |  <-- Author, date normalise, reading time, images
|    (FeedItem[])  |
+------------------+
        |
        v
+------------------+
| 5. SCORING       |  <-- Source Q + Content Q + Freshness = Composite
|    (FeedItem[])  |
+------------------+
        |
        v
+------------------+
| 6. SUMMARISATION |  <-- Extractive (default) / Abstractive (LLM)
|    (FeedItem[])  |
+------------------+
        |
        v
+------------------+
| 7. CATEGORISATION|  <-- Topic matching, tags, content type, importance
|    (FeedItem[])  |
+------------------+
        |
        v
+------------------+
| 8. RANKING       |  <-- Sort, diversity, breaking boost, tier filter
|    (FeedItem[])  |
+------------------+
        |
        v
+------------------+
| 9. STORAGE       |  <-- PostgreSQL upsert, related links, index
|    (FeedItem[])  |      [ARCHIVE] --> Cold storage (>90 days)
+------------------+
        |
        v
+------------------+
| 10. DISTRIBUTION |  <-- API, Telegram, Digest, n8n, Website cache
|    (multi-output)|
+------------------+
```

---

## 2. Stage-by-Stage Design (split across files)

The full design for each stage moved to its own file. Load only what you need:

| Stages | File | Approx. size |
|---|---|---|
| Stage 1 (Collection) · Stage 2 (Validation) | [pipeline-collection.md](./pipeline-collection.md) | ~30 KB |
| Stage 3 (Deduplication) · Stage 4 (Enrichment) · Stage 5 (Scoring) · Stage 6 (Summarisation) | [pipeline-processing.md](./pipeline-processing.md) | ~40 KB |
| Stage 7 (Categorisation) · Stage 8 (Ranking) · Stage 9 (Storage) · Stage 10 (Distribution) | [pipeline-distribution.md](./pipeline-distribution.md) | ~35 KB |
| Section 3 (Pipeline Orchestration) · Section 4 (Output Formats) | [pipeline-orchestration.md](./pipeline-orchestration.md) | ~25 KB |

If you are implementing the pipeline end-to-end, load all four split files
plus this index. If you are implementing or debugging a single stage, load
only the file containing that stage plus this index.

---
## Appendix A: Environment Configuration

```typescript
interface PipelineConfig {
  // Database
  database: {
    host: string;
    port: number;
    name: string;
    user: string;
    password: string;
    poolSize: number;
  };

  // Redis
  redis: {
    host: string;
    port: number;
    password?: string;
    db: number;
  };

  // Collection
  collection: {
    maxConcurrentSources: number;
    defaultRateLimitRpm: number;
    requestTimeoutMs: number;
    maxRetries: number;
  };

  // Scoring
  scoring: {
    sourceWeight: number;      // 0.30
    contentWeight: number;     // 0.40
    freshnessWeight: number;   // 0.20
    engagementWeight: number;  // 0.10
    halfLifeHours: number;     // 24
    breakingWindowMinutes: number; // 60
  };

  // Summarisation
  summarisation: {
    useLLM: boolean;
    llmEndpoint?: string;
    minSummaryLength: number;   // 50
    maxDigestLength: number;    // 200
    maxDetailLength: number;    // 500
  };

  // Distribution
  telegram: {
    enabled: boolean;
    botToken: string;
    chatId: string;
    minTier: number;  // Only send tiers <= this
  };

  digest: {
    enabled: boolean;
    cronSchedule: string;  // "0 7 * * *" (7am daily)
    maxItems: number;      // 30
    minTier: number;       // 3 (tiers 1-3)
  };

  n8n: {
    enabled: boolean;
    webhookUrl: string;
    secret: string;
    batchSize: number;
  };

  website: {
    nextJsUrl: string;
    revalidateToken: string;
  };

  // Monitoring
  monitoring: {
    metricsPort: number;     // 9090 for Prometheus
    healthCheckPort: number; // 8080
    alertWebhook?: string;
  };
}
```

## Appendix B: Source Registry Example

```json
[
  {
    "id": "hackernews-top",
    "name": "Hacker News Top Stories",
    "type": "api",
    "url": "https://hacker-news.firebaseio.com/v0/topstories.json",
    "topic": "tech",
    "qualityScore": 85,
    "collectionInterval": 30,
    "rateLimitRpm": 60,
    "adapter": "HackerNewsAdapter",
    "enabled": true
  },
  {
    "id": "arxiv-ai",
    "name": "arXiv AI Papers",
    "type": "rss",
    "url": "http://export.arxiv.org/rss/cs.AI",
    "topic": "ai",
    "qualityScore": 90,
    "collectionInterval": 360,
    "rateLimitRpm": 10,
    "adapter": "ArxivAdapter",
    "enabled": true
  },
  {
    "id": "the-verge",
    "name": "The Verge",
    "type": "rss",
    "url": "https://www.theverge.com/rss/index.xml",
    "topic": "tech",
    "qualityScore": 80,
    "collectionInterval": 60,
    "rateLimitRpm": 30,
    "adapter": "RssAdapter",
    "enabled": true
  },
  {
    "id": "espn-football",
    "name": "ESPN Football",
    "type": "rss",
    "url": "https://www.espn.com/espn/rss/soccer/news",
    "topic": "sports",
    "qualityScore": 85,
    "collectionInterval": 30,
    "rateLimitRpm": 60,
    "adapter": "RssAdapter",
    "enabled": true
  },
  {
    "id": "openai-blog",
    "name": "OpenAI Blog",
    "type": "html",
    "url": "https://openai.com/blog",
    "topic": "ai",
    "qualityScore": 95,
    "collectionInterval": 360,
    "rateLimitRpm": 10,
    "adapter": "HtmlScraperAdapter",
    "selectors": {
      "articleLink": "a[href*='/blog/']",
      "title": "h1, .post-title",
      "content": "article, .post-content",
      "author": "[rel='author'], .author",
      "date": "time, [datetime]",
      "image": "meta[property='og:image']"
    },
    "enabled": true
  }
]
```

## Appendix C: Topic Configuration

```typescript
const TOPIC_CONFIG: Record<Topic, TopicSettings> = {
  ai: {
    label: 'Artificial Intelligence',
    emoji: '🤖',
    color: '#8b5cf6',
    minTier: 3,
    maxFeedItems: 50,
    telegramEnabled: true,
    digestPriority: 1,
  },
  gaming: {
    label: 'Gaming',
    emoji: '🎮',
    color: '#ec4899',
    minTier: 3,
    maxFeedItems: 40,
    telegramEnabled: true,
    digestPriority: 3,
  },
  tech: {
    label: 'Technology',
    emoji: '💻',
    color: '#06b6d4',
    minTier: 3,
    maxFeedItems: 60,
    telegramEnabled: true,
    digestPriority: 2,
  },
  sports: {
    label: 'Sports',
    emoji: '⚽',
    color: '#22c55e',
    minTier: 4,
    maxFeedItems: 30,
    telegramEnabled: false,
    digestPriority: 4,
  },
  world: {
    label: 'World News',
    emoji: '🌍',
    color: '#f59e0b',
    minTier: 3,
    maxFeedItems: 40,
    telegramEnabled: true,
    digestPriority: 2,
  },
  science: {
    label: 'Science',
    emoji: '🔬',
    color: '#14b8a6',
    minTier: 4,
    maxFeedItems: 30,
    telegramEnabled: false,
    digestPriority: 5,
  },
  business: {
    label: 'Business',
    emoji: '💼',
    color: '#6366f1',
    minTier: 3,
    maxFeedItems: 35,
    telegramEnabled: false,
    digestPriority: 3,
  },
};
```
