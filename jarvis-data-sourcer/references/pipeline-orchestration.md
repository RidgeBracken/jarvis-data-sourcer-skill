# Pipeline: Orchestration & Output

> Part of the Jarvis HQ pipeline reference. Originally a section of
> `pipeline.md`; split for progressive disclosure. **Stages covered:** Pipeline Orchestration, Output Formats.
> Previous: [$prevFile](pipeline-distribution.md)

> **Schema authority:** `data-schema.md`. **Safety rules:** `../SKILL.md`.

## 3. Pipeline Orchestration

### Orchestration Architecture

The pipeline uses a **hybrid event-driven + sequential** architecture:

```
  Sources          Redis Streams          Workers           Outputs
     |                  |                   |                 |
     v                  v                   v                 v
  +--------+     +--------------+     +-------------+     +--------+
  | RSS    |---->| collection   |---->| Validation  |---->| API    |
  | API    |     | stream       |     | Worker      |     | Cache  |
  | HTML   |     +--------------+     +-------------+     +--------+
  | GitHub |            |                   |
  | YouTube|            v                   v
  | Social |     +--------------+     +-------------+     +--------+
  +--------+     | priority     |---->| Dedup       |---->| Telegram
                 | queue        |     | Worker      |     +--------+
                 +--------------+     +-------------+
                                             |
                                             v
                                      +-------------+     +--------+
                                      | Enrichment  |---->| Digest |
                                      | Worker      |     +--------+
                                      +-------------+
                                             |
                                             v
                                      +-------------+     +--------+
                                      | Scoring     |---->| n8n    |
                                      | Worker      |     | Webhook|
                                      +-------------+     +--------+
                                             |
                                             v
                                      +-------------+     +--------+
                                      | Remaining   |---->| Website|
                                      | Stages      |     | Cache  |
                                      +-------------+     +--------+
```

### Stage Connection Strategy

| Transition | Pattern | Mechanism | Retry |
|------------|---------|-----------|-------|
| Collection -> Validation | Event-driven | Redis Stream push | 3x exponential |
| Validation -> Deduplication | Streaming | In-memory pipeline | 2x immediate |
| Deduplication -> Enrichment | Streaming | In-memory pipeline | 2x immediate |
| Enrichment -> Scoring | Streaming | In-memory pipeline | 2x immediate |
| Scoring -> Summarisation | Streaming | In-memory pipeline | 2x immediate |
| Summarisation -> Categorisation | Streaming | In-memory pipeline | 2x immediate |
| Categorisation -> Ranking | Batch | Cron-triggered queue | 3x exponential |
| Ranking -> Storage | Streaming | PostgreSQL transaction | 5x with backoff |
| Storage -> Distribution | Event-driven | Redis pub/sub | 3x exponential |

### Error Handling Between Stages

```typescript
class PipelineOrchestrator {
  private stages: Map<PipelineStage, PipelineStageExecutor> = new Map();
  private dlq: DeadLetterQueue;
  private metrics: MetricsCollector;

  constructor(config: PipelineConfig) {
    this.dlq = new DeadLetterQueue(config.db, config.redis);
    this.metrics = new MetricsCollector(config.redis);
  }

  async executePipeline(items: FeedItem[]): Promise<void> {
    let currentItems = items;

    for (const [stageName, executor] of this.stages) {
      const startTime = Date.now();
      
      try {
        currentItems = await this.executeStage(stageName, executor, currentItems);
        
        this.metrics.recordStageMetrics(stageName, {
          duration: Date.now() - startTime,
          itemsIn: items.length,
          itemsOut: currentItems.length,
          errors: 0,
        });

      } catch (err) {
        logger.error(`Stage ${stageName} failed: ${err.message}`);
        
        this.metrics.recordStageMetrics(stageName, {
          duration: Date.now() - startTime,
          itemsIn: currentItems.length,
          itemsOut: 0,
          errors: currentItems.length,
        });

        // Send to DLQ
        for (const item of currentItems) {
          await this.dlq.storeRejected({
            item,
            reason: `Stage ${stageName} failed: ${err.message}`,
            rule: 'pipeline_stage_failure',
            timestamp: new Date().toISOString(),
          });
        }

        // Alert on repeated failures
        await this.alertOnRepeatedFailures(stageName);
        
        // Don't continue with failed items
        break;
      }

      // If no items left, stop pipeline
      if (currentItems.length === 0) {
        logger.info(`Pipeline stopped at ${stageName}: no items to process`);
        break;
      }
    }
  }

  private async executeStage(
    stageName: PipelineStage,
    executor: PipelineStageExecutor,
    items: FeedItem[]
  ): Promise<FeedItem[]> {
    const maxRetries = executor.retryCount;
    let lastError: Error | null = null;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const result = await executor.execute(items);
        
        if (attempt > 1) {
          logger.info(`Stage ${stageName} succeeded on attempt ${attempt}`);
        }
        
        return result;
      } catch (err) {
        lastError = err as Error;
        logger.warn(`Stage ${stageName} attempt ${attempt}/${maxRetries} failed: ${err.message}`);
        
        if (attempt < maxRetries) {
          const delay = executor.getBackoffDelay(attempt);
          await new Promise(r => setTimeout(r, delay));
        }
      }
    }

    throw lastError;
  }

  private async alertOnRepeatedFailures(stageName: PipelineStage): Promise<void> {
    const key = `pipeline:failure:${stageName}`;
    const count = await this.metrics.redis.incr(key);
    await this.metrics.redis.expire(key, 3600); // 1 hour window

    if (count >= 5) {
      await this.sendAlert(`Pipeline stage ${stageName} has failed ${count} times in the last hour`);
      await this.metrics.redis.del(key);
    }
  }

  private async sendAlert(message: string): Promise<void> {
    // Send via configured alert channels
    logger.error(`ALERT: ${message}`);
    // Could also send to Telegram, PagerDuty, etc.
  }
}

interface PipelineStageExecutor {
  execute(items: FeedItem[]): Promise<FeedItem[]>;
  retryCount: number;
  getBackoffDelay(attempt: number): number;
}
```

### Dead Letter Queue

```typescript
class DeadLetterQueue {
  constructor(private db: Knex, private redis: Redis) {}

  async storeRejected(rejected: RejectedItem): Promise<void> {
    await this.db('rejected_items').insert({
      item_id: rejected.item.id,
      source_id: rejected.item.sourceId,
      title: rejected.item.title,
      url: rejected.item.rawUrl,
      rejection_reason: rejected.reason,
      rejection_rule: rejected.rule,
      item_data: JSON.stringify(rejected.item),
      created_at: rejected.timestamp,
    });

    // Also push to Redis for monitoring
    await this.redis.xadd('dlq:stream', '*',
      'item_id', rejected.item.id,
      'reason', rejected.reason,
      'rule', rejected.rule,
      'stage', rejected.item.pipelineStage,
      'timestamp', rejected.timestamp
    );
  }

  async reprocess(itemId: string): Promise<void> {
    const record = await this.db('rejected_items')
      .where('item_id', itemId)
      .first();

    if (!record) throw new Error('Item not found in DLQ');

    const item: FeedItem = JSON.parse(record.item_data);
    
    // Reset pipeline state
    item.processingAttempts = 0;
    item.lastError = null;
    item.pipelineStage = 'collected';

    // Re-inject
    await this.redis.xadd('collection:stream', '*',
      'item', JSON.stringify(item),
      'reprocess', 'true'
    );

    await this.db('rejected_items')
      .where('item_id', itemId)
      .update({ reprocessed: true });
  }

  async getStats(): Promise<{ total: number; recent: number; byRule: Record<string, number> }> {
    const total = await this.db('rejected_items').count('id as c').first();
    const recent = await this.db('rejected_items')
      .where('created_at', '>', new Date(Date.now() - 86400000).toISOString())
      .count('id as c').first();

    const byRule = await this.db('rejected_items')
      .select('rejection_rule')
      .count('id as count')
      .groupBy('rejection_rule');

    return {
      total: Number(total?.c ?? 0),
      recent: Number(recent?.c ?? 0),
      byRule: Object.fromEntries(byRule.map(r => [r.rejection_rule, Number(r.count)])),
    };
  }
}
```

### Monitoring and Alerting

```typescript
class PipelineMonitor {
  constructor(private redis: Redis, private db: Knex) {}

  async collectMetrics(): Promise<PipelineMetrics> {
    const now = new Date();
    const oneHourAgo = new Date(now.getTime() - 3600000).toISOString();

    return {
      // Collection metrics
      itemsCollectedLastHour: await this.db('feed_items')
        .where('collected_at', '>', oneHourAgo).count('id as c').then(r => Number(r[0].c)),
      
      // Processing metrics
      itemsByStage: await this.db('feed_items')
        .select(this.db.raw('pipeline_stage, COUNT(*) as count'))
        .groupBy('pipeline_stage'),

      // Quality metrics
      avgCompositeScore: await this.db('feed_items')
        .where('collected_at', '>', oneHourAgo)
        .avg('composite_score as avg').then(r => Number(r[0].avg) || 0),

      tierDistribution: await this.db('feed_items')
        .where('collected_at', '>', oneHourAgo)
        .select('tier').count('id as count').groupBy('tier'),

      // Error metrics
      rejectionCount: await this.db('rejected_items')
        .where('created_at', '>', oneHourAgo).count('id as c').then(r => Number(r[0].c)),

      // Source health
      sourceHealth: await this.db('sources')
        .leftJoin('feed_items', 'sources.id', 'feed_items.source_id')
        .select('sources.id', 'sources.name')
        .count('feed_items.id as item_count')
        .groupBy('sources.id', 'sources.name'),

      // Latency metrics
      avgProcessingTime: await this.redis.get('metrics:avg_processing_time').then(v => Number(v) || 0),
    };
  }

  async healthCheck(): Promise<HealthStatus> {
    const checks = await Promise.all([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkSources(),
      this.checkPipelineLag(),
    ]);

    const allHealthy = checks.every(c => c.healthy);

    return {
      status: allHealthy ? 'healthy' : 'degraded',
      checks: Object.fromEntries(checks.map(c => [c.name, c])),
      timestamp: new Date().toISOString(),
    };
  }

  private async checkDatabase(): Promise<HealthCheck> {
    try {
      await this.db.raw('SELECT 1');
      return { name: 'database', healthy: true, latency: 0 };
    } catch (err) {
      return { name: 'database', healthy: false, error: err.message };
    }
  }

  private async checkRedis(): Promise<HealthCheck> {
    try {
      const start = Date.now();
      await this.redis.ping();
      return { name: 'redis', healthy: true, latency: Date.now() - start };
    } catch (err) {
      return { name: 'redis', healthy: false, error: err.message };
    }
  }

  private async checkSources(): Promise<HealthCheck> {
    const failedSources = await this.db('sources')
      .where('enabled', true)
      .where('last_error_at', '>', new Date(Date.now() - 3600000).toISOString())
      .count('id as c').then(r => Number(r[0].c));

    return {
      name: 'sources',
      healthy: failedSources < 3,
      failingSources: failedSources,
    };
  }

  private async checkPipelineLag(): Promise<HealthCheck> {
    const lagItems = await this.db('feed_items')
      .where('pipeline_stage', '!=', 'distributed')
      .where('collected_at', '<', new Date(Date.now() - 300000).toISOString()) // 5 min lag
      .count('id as c').then(r => Number(r[0].c));

    return {
      name: 'pipeline_lag',
      healthy: lagItems < 100,
      stalledItems: lagItems,
    };
  }
}

interface PipelineMetrics {
  itemsCollectedLastHour: number;
  itemsByStage: Array<{ pipeline_stage: string; count: number }>;
  avgCompositeScore: number;
  tierDistribution: Array<{ tier: number; count: number }>;
  rejectionCount: number;
  sourceHealth: Array<{ id: string; name: string; item_count: number }>;
  avgProcessingTime: number;
}

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  checks: Record<string, HealthCheck>;
  timestamp: string;
}

interface HealthCheck {
  name: string;
  healthy: boolean;
  latency?: number;
  error?: string;
  failingSources?: number;
  stalledItems?: number;
}
```

### Grafana Dashboard Metrics

The following metrics are exposed for monitoring:

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `pipeline_items_total` | Counter | `stage`, `topic` | Items processed per stage |
| `pipeline_duration_seconds` | Histogram | `stage` | Stage processing duration |
| `pipeline_errors_total` | Counter | `stage`, `error_type` | Error count per stage |
| `dlq_items_total` | Counter | `rule`, `source` | Dead letter queue inserts |
| `feed_items_by_tier` | Gauge | `topic`, `tier` | Current items per tier |
| `source_collection_duration` | Histogram | `source_id` | Per-source collection time |
| `source_errors_total` | Counter | `source_id`, `error_type` | Per-source error count |
| `freshness_score_avg` | Gauge | `topic` | Average freshness score |
| `composite_score_avg` | Gauge | `topic` | Average composite score |
| `cache_hit_ratio` | Gauge | `endpoint` | API cache hit ratio |
| `telegram_messages_sent` | Counter | `topic`, `tier` | Telegram messages sent |
| `digest_items_queued` | Gauge | -- | Items queued for next digest |

---

## 4. Output Formats

### 4.1 API Response Format

The API serves feeds for each topic, supporting pagination, filtering, and sorting.

**Endpoint:** `GET /api/feed/{topic}?tier=&limit=&offset=&sort=`

```json
{
  "items": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "title": "OpenAI Announces GPT-5 with Multimodal Reasoning",
      "summary": "OpenAI has unveiled GPT-5, featuring significant improvements in multimodal reasoning, code generation, and scientific problem-solving capabilities.",
      "url": "https://example.com/openai-gpt5-announcement",
      "imageUrl": "https://example.com/images/gpt5.jpg",
      "author": "Jane Smith",
      "publishedAt": "2026-05-14T06:30:00Z",
      "collectedAt": "2026-05-14T07:00:00Z",
      "readingTime": 8,
      "language": "en",
      "score": {
        "composite": 94,
        "source": 92,
        "content": 95,
        "freshness": 100
      },
      "tier": 1,
      "topics": ["ai"],
      "tags": ["openai", "gpt", "release", "exclusive"],
      "contentType": "article",
      "source": {
        "id": "techcrunch-ai",
        "name": "TechCrunch AI",
        "qualityScore": 92
      },
      "metadata": {
        "duplicateCount": 2,
        "relatedItems": 3
      }
    }
  ],
  "meta": {
    "topic": "ai",
    "total": 150,
    "returned": 20,
    "offset": 0,
    "limit": 20,
    "new_today": 12,
    "tier_breakdown": {
      "1": 3,
      "2": 8,
      "3": 15,
      "4": 32,
      "5": 92
    },
    "sort": "composite_score_desc",
    "filters": {
      "tier_min": null,
      "date_from": null,
      "date_to": null,
      "tags": []
    },
    "last_updated": "2026-05-14T08:00:00Z",
    "response_time_ms": 45
  },
  "pagination": {
    "hasMore": true,
    "nextOffset": 20,
    "prevOffset": null
  }
}
```

### 4.2 Telegram Briefing Format

Telegram messages use MarkdownV2 formatting with emoji indicators.

#### Breaking News Format (Tier 1)

```markdown
🤖 🔴 BREAKING OpenAI Announces GPT\-5 with Multimodal Reasoning

OpenAI has unveiled GPT\-5, featuring significant improvements in 
multimodal reasoning, code generation, and scientific problem\-solving 
capabilities\. The new model scores 95% on graduate\-level reasoning 
benchmarks\.

🏷 ai, release | ⏱ 8min | ⭐ 94
📰 TechCrunch AI | 🕐 2h ago

[Read more](https://example.com/openai-gpt5-announcement)
```

#### Regular Item Format (Tier 2-3)

```markdown
🎮 🟠 HIGH Nintendo Switch 2 Pre\-Orders Break Records

Nintendo reports over 2 million pre\-orders for the Switch 2 console 
within 24 hours of opening, surpassing all previous hardware launches\.

🏷 gaming, nintendo | ⏱ 5min | ⭐ 78
📰 GameSpot | 🕐 4h ago

[Read more](https://example.com/switch2-preorders)
```

#### Compact Format (Tier 4-5 / Batch)

```markdown
📰 Quick Roundup

• 💻 Google announces Android 16 developer preview [⭐ 62]
• ⚽ Champions League semi\-final results: Real Madrid advances [⭐ 58]
• 🔬 New fusion reactor achieves 30\-minute sustained reaction [⭐ 71]
• 💼 Stripe raises $500M at $85B valuation [⭐ 65]

Full feed: https://jarvis-hq.example.com
```

### 4.3 Daily Digest Format

The daily digest is a Markdown document designed for email or messaging delivery.

```markdown
# 📰 Jarvis HQ Daily Digest

## Wednesday, May 14, 2026

**Top stories from the last 24 hours**

| Topic | Stories | Highlights |
|-------|---------|------------|
| 🤖 AI | 8 | GPT-5, Anthropic funding, EU AI Act |
| 💻 Tech | 12 | Android 16, Cloudflare outage, New chips |
| 🎮 Gaming | 6 | Switch 2, Elden Ring DLC, Xbox showcase |
| 🌍 World | 5 | G7 summit, Climate report, Trade talks |
| ⚽ Sports | 4 | Champions League, NBA playoffs |

---

### 🤖 AI — Top Stories

**🔴 OpenAI Announces GPT-5 with Multimodal Reasoning** ⭐⭐⭐⭐⭐
OpenAI has unveiled GPT-5, featuring significant improvements in 
multimodal reasoning, code generation, and scientific problem-solving. 
The model demonstrates graduate-level performance across mathematics, 
physics, and computer science benchmarks.
[Read more (8 min)](https://example.com/openai-gpt5) | TechCrunch AI

**🟠 Anthropic Raises $2B at $20B Valuation** ⭐⭐⭐⭐
Anthropic has closed a Series D funding round led by Spark Capital, 
bringing total funding to over $7 billion. The company plans to 
expand Claude's capabilities and accelerate international expansion.
[Read more (5 min)](https://example.com/anthropic-funding) | The Information

**🟡 EU AI Act Implementation Begins Today** ⭐⭐⭐
The European Union's comprehensive AI Act enters its first enforcement 
phase, requiring AI systems classified as "high-risk" to meet strict 
transparency and safety standards.
[Read more (6 min)](https://example.com/eu-ai-act) | Politico EU

---

### 💻 Tech — Top Stories

**🟠 Google Releases Android 16 Developer Preview** ⭐⭐⭐⭐
The first developer preview of Android 16 introduces a redesigned 
notification system, improved battery management, and native support 
for foldable form factors.
[Read more (7 min)](https://example.com/android16) | Android Police

**🟡 Global Cloudflare Outage Affects 15% of Internet** ⭐⭐⭐
A configuration error in Cloudflare's edge network caused widespread 
website accessibility issues for approximately 45 minutes across 
Europe and Asia-Pacific regions.
[Read more (4 min)](https://example.com/cloudflare-outage) | The Verge

---

### 📊 Today's Statistics

- **47 new stories** collected from 32 sources
- **Average quality score:** 68/100
- **Sources with issues:** 2 (Hacker News RSS, Ars Technica API)
- **Pipeline processing time:** 3m 42s

---

*Jarvis HQ — Your personal AI news curator*
*Manage preferences: https://jarvis-hq.example.com/settings*
```

### 4.4 Website Feed Format

The React/Next.js website consumes this JSON format for SSR and client-side hydration.

```typescript
// /api/feed/[topic].ts
interface FeedResponse {
  items: FeedCardItem[];
  meta: FeedMeta;
  pagination: PaginationInfo;
}

interface FeedCardItem {
  id: string;
  title: string;
  summary: string;
  url: string;
  imageUrl: string | null;
  author: string | null;
  publishedAt: string;      // ISO 8601
  relativeTime: string;     // "2h ago", "yesterday"
  readingTime: number | null; // minutes
  language: string;
  score: {
    composite: number;      // 0 -- 100
    tier: 1 | 2 | 3 | 4 | 5;
    tierLabel: string;      // "Breaking", "High", "Good", "Average", "Low"
    tierColor: string;      // "#ef4444", "#f97316", "#eab308", "#3b82f6", "#6b7280"
  };
  topics: Topic[];
  tags: string[];
  contentType: ContentType;
  source: {
    id: string;
    name: string;
    iconUrl: string | null;
  };
  engagement: {
    relatedCount: number;   // Number of related stories
    duplicateCount: number; // Number of duplicate sources
  };
}

interface FeedMeta {
  topic: Topic;
  topicLabel: string;       // Human-readable: "Artificial Intelligence"
  topicEmoji: string;       // "🤖"
  total: number;            // Total matching items
  newToday: number;         // Items collected in last 24h
  tierBreakdown: Record<string, number>;
  lastUpdated: string;      // ISO 8601
  refreshInterval: number;  // Seconds until next refresh
  filters: {
    tier?: number;
    dateFrom?: string;
    dateTo?: string;
    tags?: string[];
    contentType?: ContentType;
  };
}

interface PaginationInfo {
  hasMore: boolean;
  currentPage: number;
  totalPages: number;
  nextOffset: number | null;
  prevOffset: number | null;
  limit: number;
}
```

#### Example Website Feed JSON

```json
{
  "items": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "title": "OpenAI Announces GPT-5 with Multimodal Reasoning",
      "summary": "OpenAI has unveiled GPT-5, featuring significant improvements in multimodal reasoning, code generation, and scientific problem-solving capabilities.",
      "url": "https://example.com/openai-gpt5-announcement",
      "imageUrl": "https://images.example.com/gpt5-hero.jpg",
      "author": "Jane Smith",
      "publishedAt": "2026-05-14T06:30:00Z",
      "relativeTime": "2h ago",
      "readingTime": 8,
      "language": "en",
      "score": {
        "composite": 94,
        "tier": 1,
        "tierLabel": "Breaking",
        "tierColor": "#ef4444"
      },
      "topics": ["ai"],
      "tags": ["openai", "gpt", "release", "exclusive"],
      "contentType": "article",
      "source": {
        "id": "techcrunch-ai",
        "name": "TechCrunch AI",
        "iconUrl": "https://assets.example.com/icons/techcrunch.png"
      },
      "engagement": {
        "relatedCount": 3,
        "duplicateCount": 2
      }
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "title": "Google DeepMind's New Protein Folding Breakthrough",
      "summary": "Researchers at Google DeepMind have extended AlphaFold to predict protein-protein interactions in real-time, opening new avenues for drug discovery.",
      "url": "https://example.com/deepmind-protein-folding",
      "imageUrl": "https://images.example.com/proteins.jpg",
      "author": "Dr. Alan Chen",
      "publishedAt": "2026-05-14T04:15:00Z",
      "relativeTime": "4h ago",
      "readingTime": 12,
      "language": "en",
      "score": {
        "composite": 87,
        "tier": 2,
        "tierLabel": "High",
        "tierColor": "#f97316"
      },
      "topics": ["ai", "science"],
      "tags": ["deepmind", "alphafold", "protein", "research"],
      "contentType": "article",
      "source": {
        "id": "nature-news",
        "name": "Nature News",
        "iconUrl": "https://assets.example.com/icons/nature.png"
      },
      "engagement": {
        "relatedCount": 5,
        "duplicateCount": 1
      }
    }
  ],
  "meta": {
    "topic": "ai",
    "topicLabel": "Artificial Intelligence",
    "topicEmoji": "🤖",
    "total": 150,
    "newToday": 12,
    "tierBreakdown": {
      "1": 3,
      "2": 8,
      "3": 15,
      "4": 32,
      "5": 92
    },
    "lastUpdated": "2026-05-14T08:00:00Z",
    "refreshInterval": 300,
    "filters": {
      "tier": null,
      "dateFrom": null,
      "dateTo": null,
      "tags": [],
      "contentType": null
    }
  },
  "pagination": {
    "hasMore": true,
    "currentPage": 1,
    "totalPages": 8,
    "nextOffset": 20,
    "prevOffset": null,
    "limit": 20
  }
}
```

#### Client-Side Hydration Format

For Next.js ISR (Incremental Static Regeneration), the initial page load includes pre-rendered HTML with embedded JSON:

```typescript
// pages/feed/[topic].tsx
interface PageProps {
  initialData: FeedResponse;
  topic: Topic;
  buildTime: string;
  revalidateAfter: number; // Seconds until ISR revalidation
}

// The page component uses SWR for client-side refresh
function FeedPage({ initialData, topic }: PageProps) {
  const { data, error } = useSWR<FeedResponse>(
    `/api/feed/${topic}`,
    fetcher,
    { 
      fallbackData: initialData,
      refreshInterval: 60000, // Poll every minute
      revalidateOnFocus: true,
    }
  );
  
  return <FeedLayout feed={data} topic={topic} />;
}

export async function getStaticProps({ params }: GetStaticPropsContext) {
  const topic = params?.topic as Topic;
  const feed = await fetchFeed(topic, { limit: 20 });
  
  return {
    props: {
      initialData: feed,
      topic,
      buildTime: new Date().toISOString(),
      revalidateAfter: 300, // Revalidate every 5 minutes
    },
    revalidate: 300,
  };
}
```

### 4.5 n8n Webhook Payload

```json
{
  "event": "new_items",
  "timestamp": "2026-05-14T08:00:00Z",
  "webhookId": "wh_abc123def456",
  "payload": {
    "count": 12,
    "topics": ["ai", "tech"],
    "items": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "title": "OpenAI Announces GPT-5",
        "url": "https://example.com/openai-gpt5",
        "topics": ["ai"],
        "tier": 1,
        "score": 94,
        "publishedAt": "2026-05-14T06:30:00Z"
      }
    ],
    "summary": {
      "byTopic": {
        "ai": { "count": 5, "avgScore": 82, "topTier": 1 },
        "tech": { "count": 7, "avgScore": 71, "topTier": 2 }
      },
      "tierDistribution": { "1": 1, "2": 3, "3": 5, "4": 3 }
    }
  }
}
```

---
