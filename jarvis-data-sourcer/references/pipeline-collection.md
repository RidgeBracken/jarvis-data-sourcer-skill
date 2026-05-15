# Pipeline: Collection & Validation

> Part of the Jarvis HQ pipeline reference. Originally a section of
> `pipeline.md`; split for progressive disclosure. **Stages covered:** Stage 1 (Collection), Stage 2 (Validation).
> Next: [$nextFile](pipeline-processing.md)

> **Schema authority:** `data-schema.md`. **Safety rules:** `../SKILL.md`.

### Stage 1: Collection

#### Purpose
Fetch raw content from all configured sources using source-specific adapters. The Collection stage is the **entry point** of the pipeline and is responsible for transforming heterogeneous external formats into the canonical `FeedItem` representation.

#### Architecture

```
  Scheduler (node-cron)        Webhook Receiver (breaking news)
         |                               |
         v                               v
  +------------------+          +------------------+
  | Collection Queue |          | Priority Queue   |
  | (bullmq/redis)   |          | (high-priority)  |
  +--------+---------+          +--------+---------+
           |                             |
           +-------------+---------------+
                         |
              +----------v-----------+
              |   Worker Pool        |
              |   (piscina/threads)  |
              +----------+-----------+
                         |
         +---------------+---------------+
         |               |               |
    +----v----+    +-----v----+   +-----v-----+
    | RSS     |    | API      |   | HTML      |
    | Adapter |    | Adapter  |   | Scraper   |
    +---------+    +----------+   +-----------+
    +---------+    +----------+   +-----------+
    | YouTube |    | GitHub   |   | Social    |
    | Adapter |    | Adapter  |   | Adapter   |
    +---------+    +----------+   +-----------+
```

#### Input: `SourceConfig[]`

```typescript
interface SourceConfig {
  id: string;                    // 'hackernews-rss', 'arxiv-ai'
  name: string;                  // Human-readable name
  type: SourceType;              // 'rss' | 'api' | 'html' | 'youtube' | 'github' | 'social'
  url: string;                   // Feed URL / API endpoint
  topic: Topic;                  // Primary topic classification
  qualityScore: number;          // 0 -- 100, editorial rating
  collectionInterval: number;    // Minutes between collections
  rateLimitRpm: number;          // Requests per minute
  adapter: string;               // Adapter class name
  auth: AuthConfig | null;       // API keys, OAuth, etc.
  selectors?: HtmlSelectors;     // For HTML scrapers
  filters?: SourceFilter[];      // URL/title/content filters
  enabled: boolean;
  lastCollectedAt: string | null;
}

type SourceType = 'rss' | 'api' | 'html' | 'youtube' | 'github' | 'social';
```

#### Output: Raw `FeedItem[]`

Each adapter produces `FeedItem` objects with the following fields populated:
- `id` (UUID v4)
- `sourceId` (from SourceConfig)
- `rawUrl` (as collected)
- `canonicalUrl` (normalised)
- `contentHash` (null -- computed later)
- `title` (from source)
- `summary` (from source or null)
- `content` (from source or null)
- `contentText` (null -- extracted later)
- `author` (from source or null)
- `language` ('unknown' -- detected later)
- `imageUrl` (from source or null)
- `videoUrl` (from source or null)
- `readingTimeMinutes` (null)
- `publishedAt` (parsed or current time)
- `collectedAt` (ISO 8601 now)
- `processedAt` (null)
- `freshnessScore` (1.0 -- computed later)
- `topics` (empty -- classified later)
- `tags` (empty -- classified later)
- `contentType` ('article' -- refined later)
- `importance` (0 -- scored later)
- `sourceQualityScore` (from SourceConfig)
- `contentQualityScore` (0 -- scored later)
- `compositeScore` (0 -- scored later)
- `tier` (5 -- assigned later)
- `duplicateGroupId` (null)
- `isPrimary` (true -- may change)
- `pipelineStage` ('collected')
- `processingAttempts` (0)
- `lastError` (null)
- `metadata` (adapter-specific data)

#### Adapter Interface

```typescript
abstract class SourceAdapter {
  constructor(protected config: SourceConfig) {}

  // Main collection method
  abstract collect(): Promise<FeedItem[]>;

  // Parse raw response into FeedItems
  abstract parse(raw: unknown): FeedItem[];

  // Normalise a URL to canonical form
  normaliseUrl(url: string): string {
    try {
      const parsed = new URL(url);
      // Remove tracking parameters
      const trackingParams = [
        'utm_source', 'utm_medium', 'utm_campaign', 'utm_term', 'utm_content',
        'fbclid', 'gclid', 'ref', 'source', 'medium'
      ];
      trackingParams.forEach(p => parsed.searchParams.delete(p));
      // Remove fragment unless it's a hash-route
      if (!parsed.hash.startsWith('#!/')) parsed.hash = '';
      return parsed.toString().toLowerCase().replace(/\/+$/, '');
    } catch {
      return url.toLowerCase().trim();
    }
  }

  // Rate limiting helper
  protected async rateLimit<T>(fn: () => Promise<T>): Promise<T> {
    const minIntervalMs = 60000 / this.config.rateLimitRpm;
    await this.rateLimitDelay(minIntervalMs);
    return fn();
  }

  private rateLimitDelay(ms: number): Promise<void> {
    return new Promise(r => setTimeout(r, ms));
  }
}
```

#### RSS Adapter (Pseudocode)

```typescript
class RssAdapter extends SourceAdapter {
  async collect(): Promise<FeedItem[]> {
    const response = await fetch(this.config.url, {
      headers: {
        'User-Agent': 'JarvisHQ-Bot/1.0 (+https://example.invalid/about-bot; contact=operator@example.invalid; purpose=personal-news-aggregation; respects-robots-txt=true)',
        'Accept': 'application/rss+xml, application/xml, text/xml'
      },
      signal: AbortSignal.timeout(30000)
    });

    if (!response.ok) {
      throw new CollectionError(
        `RSS fetch failed: ${response.status}`,
        response.status >= 500 || response.status === 429 ? 'transient' : 'permanent',
        { status: response.status, url: this.config.url }
      );
    }

    const xmlText = await response.text();
    return this.parse(xmlText);
  }

  parse(xmlText: string): FeedItem[] {
    const feed = parseXml(xmlText);
    const items = feed.channel?.item ?? feed.item ?? [];

    return items.map((entry: RssEntry) => {
      const rawUrl = entry.link?.[0] ?? entry.guid?.[0]?._ ?? entry.guid?.[0];
      const pubDate = entry.pubDate?.[0] ?? entry.dcDate?.[0];
      
      return this.createFeedItem({
        rawUrl,
        title: decodeHtmlEntities(entry.title?.[0] ?? 'Untitled'),
        summary: decodeHtmlEntities(entry.description?.[0] ?? null),
        content: entry['content:encoded']?.[0] ?? entry.description?.[0] ?? null,
        author: entry.author?.[0] ?? entry['dc:creator']?.[0] ?? null,
        publishedAt: parseRfc2822Date(pubDate) ?? new Date().toISOString(),
        imageUrl: extractImageFromContent(entry['content:encoded']?.[0]) 
               ?? entry.enclosure?.[0]?.url 
               ?? null,
        metadata: {
          feedType: 'rss',
          feedTitle: feed.channel?.title?.[0],
          categories: entry.category ?? []
        }
      });
    }).filter(item => item.rawUrl); // Reject items without URLs
  }

  private createFeedItem(partial: Partial<FeedItem>): FeedItem {
    const canonicalUrl = this.normaliseUrl(partial.rawUrl!);
    return {
      id: crypto.randomUUID(),
      sourceId: this.config.id,
      canonicalUrl,
      rawUrl: partial.rawUrl!,
      contentHash: '', // Computed in Validation stage
      title: partial.title!,
      summary: partial.summary ?? null,
      content: partial.content ?? null,
      contentText: null,
      author: partial.author ?? null,
      language: 'unknown',
      imageUrl: partial.imageUrl ?? null,
      videoUrl: null,
      readingTimeMinutes: null,
      publishedAt: partial.publishedAt!,
      collectedAt: new Date().toISOString(),
      processedAt: null,
      freshnessScore: 1.0,
      topics: [],
      tags: [],
      contentType: 'article',
      importance: 0,
      sourceQualityScore: this.config.qualityScore,
      contentQualityScore: 0,
      compositeScore: 0,
      tier: 5,
      duplicateGroupId: null,
      isPrimary: true,
      pipelineStage: 'collected',
      processingAttempts: 0,
      lastError: null,
      metadata: partial.metadata ?? {}
    };
  }
}
```

#### HTML Scraper Adapter (Pseudocode)

```typescript
class HtmlScraperAdapter extends SourceAdapter {
  async collect(): Promise<FeedItem[]> {
    // For HTML sources, the "collection" is typically:
    // 1. Fetch listing page
    // 2. Extract article links
    // 3. Fetch individual articles

    const listingHtml = await this.rateLimit(() =>
      fetchHtml(this.config.url, { timeout: 30000 })
    );

    const articleUrls = extractLinks(listingHtml, this.config.selectors!.articleLink);

    // Apply URL filters
    const filteredUrls = this.applyFilters(articleUrls);

    // Fetch articles in parallel with concurrency limit
    const concurrencyLimit = Math.min(5, Math.ceil(this.config.rateLimitRpm / 2));
    const articles = await pMap(filteredUrls, async (url) => {
      try {
        const articleHtml = await this.rateLimit(() =>
          fetchHtml(url, { timeout: 30000 })
        );
        return this.parseArticle(articleHtml, url);
      } catch (err) {
        logger.warn(`Failed to fetch article: ${url}`, { error: err.message });
        return null;
      }
    }, { concurrency: concurrencyLimit });

    return articles.filter((a): a is FeedItem => a !== null);
  }

  private parseArticle(html: string, url: string): FeedItem {
    const dom = new JSDOM(html);
    const doc = dom.window.document;
    const s = this.config.selectors!;

    const title = doc.querySelector(s.title)?.textContent?.trim() ?? 'Untitled';
    const content = doc.querySelector(s.content)?.innerHTML ?? null;
    const author = doc.querySelector(s.author)?.textContent?.trim() ?? null;
    const pubDate = doc.querySelector(s.date)?.getAttribute('datetime')
               ?? doc.querySelector(s.date)?.textContent
               ?? null;
    const image = doc.querySelector(s.image)?.getAttribute('src')
              ?? doc.querySelector('meta[property="og:image"]')?.getAttribute('content')
              ?? null;

    // Extract JSON-LD metadata if present
    const jsonLd = this.extractJsonLd(doc);

    return this.createFeedItem({
      rawUrl: url,
      title: sanitizeText(title),
      summary: null, // Will be generated in Summarisation stage
      content: content,
      author: author ?? jsonLd?.author?.name ?? null,
      publishedAt: parseDate(pubDate) ?? jsonLd?.datePublished ?? new Date().toISOString(),
      imageUrl: image,
      metadata: { scraped: true, jsonLd: jsonLd !== null }
    });
  }

  private extractJsonLd(doc: Document): Record<string, unknown> | null {
    const script = doc.querySelector('script[type="application/ld+json"]');
    if (!script) return null;
    try { return JSON.parse(script.textContent ?? '{}'); }
    catch { return null; }
  }
}
```

#### Scheduler Design

```typescript
class CollectionScheduler {
  private cronJobs: Map<string, CronJob> = new Map();
  private queue: Queue; // bullmq Queue

  constructor(private sourceRegistry: SourceRegistry) {
    this.queue = new Queue('collection', { connection: redis });
  }

  async start(): Promise<void> {
    const sources = await this.sourceRegistry.getEnabledSources();
    
    for (const source of sources) {
      // Validate source can be scheduled
      if (!source.collectionInterval || source.collectionInterval < 5) {
        logger.warn(`Source ${source.id} has invalid interval, using 60min default`);
        source.collectionInterval = 60;
      }

      const job = new CronJob(
        this.intervalToCron(source.collectionInterval),
        () => this.scheduleCollection(source),
        null, // onComplete
        false, // start immediately -- we start after setup
        'UTC',
        null, // context
        true, // runOnInit -- collect once on startup
      );

      this.cronJobs.set(source.id, job);
      job.start();

      logger.info(`Scheduled ${source.id}: every ${source.collectionInterval}min`);
    }
  }

  private scheduleCollection(source: SourceConfig): void {
    this.queue.add('collect', { sourceId: source.id }, {
      priority: this.getTopicPriority(source.topic),
      attempts: 3,
      backoff: { type: 'exponential', delay: 5000 },
      removeOnComplete: { age: 86400 }, // Keep 24h
      removeOnFail: { age: 604800 },    // Keep 7d
    });
  }

  private getTopicPriority(topic: Topic): number {
    // Higher priority = processed first
    const priorities: Record<Topic, number> = {
      ai: 10,       // AI news is highest priority
      tech: 20,
      world: 30,
      gaming: 40,
      sports: 50,
      science: 60,
      business: 70,
    };
    return priorities[topic] ?? 50;
  }

  private intervalToCron(minutes: number): string {
    if (minutes <= 60) return `*/${minutes} * * * *`;
    const hours = Math.floor(minutes / 60);
    return `0 */${hours} * * *`;
  }

  // Trigger immediate collection (for breaking news / webhooks)
  async triggerImmediate(sourceId: string): Promise<void> {
    await this.queue.add('collect', { sourceId }, {
      priority: 1, // Highest priority
      attempts: 3,
      backoff: { type: 'fixed', delay: 2000 },
    });
  }
}
```

#### Parallel Collection Strategy

```typescript
class ParallelCollector {
  // Per-domain concurrency limit to respect rate limits
  private domainSemaphores: Map<string, Semaphore> = new Map();
  private readonly DEFAULT_CONCURRENCY = 3;

  async collectAll(sources: SourceConfig[]): Promise<FeedItem[]> {
    // Group sources by domain to enforce per-domain limits
    const byDomain = groupBy(sources, s => new URL(s.url).hostname);

    const results = await Promise.allSettled(
      sources.map(source => this.collectWithLimit(source))
    );

    const items: FeedItem[] = [];
    const errors: CollectionError[] = [];

    for (const [i, result] of results.entries()) {
      if (result.status === 'fulfilled') {
        items.push(...result.value);
      } else {
        errors.push(result.reason);
        await this.classifyAndHandleError(result.reason, sources[i]);
      }
    }

    // Report metrics
    metrics.gauge('collection.items_total', items.length);
    metrics.gauge('collection.errors_total', errors.length);
    metrics.gauge('collection.sources_success', 
      results.filter(r => r.status === 'fulfilled').length);

    return items;
  }

  private async collectWithLimit(source: SourceConfig): Promise<FeedItem[]> {
    const domain = new URL(source.url).hostname;
    const semaphore = this.getDomainSemaphore(domain, source.rateLimitRpm);
    
    return semaphore.acquire(async () => {
      const adapter = AdapterFactory.create(source);
      const startTime = Date.now();
      
      try {
        const items = await adapter.collect();
        const duration = Date.now() - startTime;
        
        metrics.histogram('collection.duration_ms', duration, { source: source.id });
        metrics.gauge('collection.items_per_source', items.length, { source: source.id });
        
        logger.info(`Collected ${items.length} items from ${source.id} in ${duration}ms`);
        return items;
      } catch (err) {
        metrics.counter('collection.errors', 1, { 
          source: source.id, 
          type: err instanceof CollectionError ? err.classification : 'unknown'
        });
        throw err;
      }
    });
  }

  private getDomainSemaphore(domain: string, rpm: number): Semaphore {
    if (!this.domainSemaphores.has(domain)) {
      const concurrency = Math.min(this.DEFAULT_CONCURRENCY, Math.max(1, Math.floor(rpm / 10)));
      this.domainSemaphores.set(domain, new Semaphore(concurrency));
    }
    return this.domainSemaphores.get(domain)!;
  }

  private async classifyAndHandleError(error: CollectionError, source: SourceConfig): Promise<void> {
    if (error.classification === 'transient') {
      // Will be retried by bullmq
      logger.warn(`Transient error for ${source.id}: ${error.message}`);
    } else {
      // Permanent error -- disable source after N consecutive failures
      const failCount = await this.incrementFailCount(source.id);
      if (failCount >= 5) {
        await this.sourceRegistry.disableSource(source.id, `Permanent failures: ${error.message}`);
        logger.error(`Disabled source ${source.id} after ${failCount} failures`);
        await this.sendAlert(`Source ${source.name} disabled due to repeated errors`);
      }
    }
  }
}
```

#### Error Classification

```typescript
class CollectionError extends Error {
  constructor(
    message: string,
    public classification: 'transient' | 'permanent',
    public context: Record<string, unknown>
  ) {
    super(message);
    this.name = 'CollectionError';
  }
}

function classifyHttpError(status: number, errorMessage: string): 'transient' | 'permanent' {
  // Transient: server errors, rate limits, timeouts, network issues
  if (status >= 500 && status < 600) return 'transient';
  if (status === 429) return 'transient'; // Rate limited -- retry with backoff
  if (status === 0) return 'transient';   // Network error / timeout
  
  // Permanent: client errors (except 429), auth failures, not found
  if (status === 401 || status === 403) return 'permanent'; // Auth issues
  if (status === 404) return 'permanent'; // Endpoint gone
  if (status === 410) return 'permanent'; // Gone permanently
  if (status >= 400 && status < 500) return 'permanent';
  
  // DNS errors, SSL errors
  if (errorMessage.includes('ENOTFOUND')) return 'transient'; // DNS may recover
  if (errorMessage.includes('ECONNREFUSED')) return 'transient';
  if (errorMessage.includes('ETIMEDOUT')) return 'transient';
  if (errorMessage.includes('EAI_AGAIN')) return 'transient';
  
  return 'transient'; // Default: retry once
}
```

#### Performance Considerations

| Metric | Target | Strategy |
|--------|--------|----------|
| Collection throughput | 1000 items/minute | Parallel workers, connection pooling |
| Per-source latency | < 30 seconds | Timeout enforcement, HTTP/2 where possible |
| Memory per worker | < 256MB | Streaming XML parsing, chunked HTML fetch |
| Retry delay (transient) | 5s, 10s, 20s | Exponential backoff with jitter |
| Concurrency per domain | 1 -- 5 | Adaptive based on rate limit headers |

---

### Stage 2: Validation

#### Purpose
Validate every collected item against the `FeedItem` schema. Reject malformed, incomplete, or suspicious items before they enter the expensive downstream stages. Validation is a **data quality gate**.

#### Input: Raw `FeedItem[]`

#### Output: Validated `FeedItem[]` + Rejection Log

#### Validation Pipeline

```
  FeedItem[]
      |
      v
+--------------------------+
| 1. Schema Validation     |  -- Check required fields exist and are correct types
|    [REJECT if fails]     |
+--------------------------+
      |
      v
+--------------------------+
| 2. URL Validation        |  -- Parseable URL, reachable, not blocked
|    [WARN if unreachable] |
+--------------------------+
      |
      v
+--------------------------+
| 3. Title Validation      |  -- Non-empty, reasonable length, not spam
|    [REJECT if fails]     |
+--------------------------+
      |
      v
+--------------------------+
| 4. Date Validation       |  -- Parseable, not in future, not ancient
|    [WARN if suspicious]  |
+--------------------------+
      |
      v
+--------------------------+
| 5. Content Hash          |  -- SHA-256 of normalised content for dedup
|    [COMPUTE]             |
+--------------------------+
      |
      v
+--------------------------+
| 6. Language Detection    |  -- Detect language if not set
|    [SET if unknown]      |
+--------------------------+
      |
      v
  Validated FeedItem[]  |  Rejected Items --> Dead Letter Queue
```

#### Validation Rules

```typescript
interface ValidationRule {
  name: string;
  validate(item: FeedItem): ValidationResult;
}

interface ValidationResult {
  valid: boolean;
  severity: 'error' | 'warn';
  message: string;
  field?: string;
}

class ValidationPipeline {
  private rules: ValidationRule[] = [
    new RequiredFieldsRule(),
    new UrlValidationRule(),
    new TitleValidationRule(),
    new DateValidationRule(),
    new ContentQualityRule(),
    new SpamDetectionRule(),
  ];

  async validate(items: FeedItem[]): Promise<ValidationOutput> {
    const validated: FeedItem[] = [];
    const rejected: RejectedItem[] = [];

    for (const item of items) {
      const results: ValidationResult[] = [];
      
      for (const rule of this.rules) {
        const result = rule.validate(item);
        results.push(result);
        
        if (!result.valid && result.severity === 'error') {
          rejected.push({
            item,
            reason: result.message,
            rule: rule.name,
            field: result.field,
            timestamp: new Date().toISOString()
          });
          break; // Stop checking this item
        }
      }

      const warnings = results.filter(r => r.valid && r.severity === 'warn');
      
      // If passed all rules (no errors)
      if (!rejected.find(r => r.item.id === item.id)) {
        // Compute content hash
        item.contentHash = await this.computeContentHash(item);
        
        // Detect language if unknown
        if (item.language === 'unknown') {
          item.language = await this.detectLanguage(item);
        }

        // Log warnings but accept
        for (const w of warnings) {
          logger.warn(`Validation warning for ${item.id}: ${w.message}`);
        }

        item.pipelineStage = 'validated';
        validated.push(item);
      }
    }

    // Metrics
    metrics.gauge('validation.passed', validated.length);
    metrics.gauge('validation.rejected', rejected.length);
    metrics.gauge('validation.reject_rate', rejected.length / (validated.length + rejected.length));

    return { validated, rejected };
  }

  private async computeContentHash(item: FeedItem): Promise<string> {
    const normalisedContent = [
      item.title.toLowerCase().trim(),
      (item.contentText ?? item.content ?? item.summary ?? '').toLowerCase().trim().slice(0, 2000),
    ].join(' || ');
    
    return crypto.subtle.digest('SHA-256', new TextEncoder().encode(normalisedContent))
      .then(buf => Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2, '0')).join(''));
  }

  private async detectLanguage(item: FeedItem): Promise<string> {
    const text = item.contentText ?? item.content ?? item.title;
    try {
      const { franc } = await import('franc');
      const lang = franc(text.slice(0, 1000));
      return lang === 'und' ? 'en' : lang; // Default to English if undetermined
    } catch {
      return 'en';
    }
  }
}
```

#### Validation Rules Implementation

```typescript
class RequiredFieldsRule implements ValidationRule {
  name = 'required_fields';

  validate(item: FeedItem): ValidationResult {
    const required = ['id', 'sourceId', 'rawUrl', 'title', 'publishedAt', 'collectedAt'];
    const missing = required.filter(field => !item[field as keyof FeedItem]);
    
    if (missing.length > 0) {
      return {
        valid: false,
        severity: 'error',
        message: `Missing required fields: ${missing.join(', ')}`,
        field: missing[0]
      };
    }
    return { valid: true, severity: 'warn', message: 'All required fields present' };
  }
}

class UrlValidationRule implements ValidationRule {
  name = 'url_validation';

  validate(item: FeedItem): ValidationResult {
    try {
      const url = new URL(item.rawUrl);
      
      // Must be http or https
      if (!['http:', 'https:'].includes(url.protocol)) {
        return { valid: false, severity: 'error', message: `Invalid protocol: ${url.protocol}`, field: 'rawUrl' };
      }
      
      // Reject known spam/tracking domains
      const blockedDomains = ['bit.ly.spam', 'malicious.example']; // Loaded from config
      if (blockedDomains.some(d => url.hostname.includes(d))) {
        return { valid: false, severity: 'error', message: 'Domain blocked', field: 'rawUrl' };
      }
      
      // URL should not be excessively long
      if (item.rawUrl.length > 2048) {
        return { valid: false, severity: 'error', message: 'URL exceeds 2048 characters', field: 'rawUrl' };
      }

      return { valid: true, severity: 'warn', message: 'URL valid' };
    } catch {
      return { valid: false, severity: 'error', message: 'Invalid URL format', field: 'rawUrl' };
    }
  }
}

class TitleValidationRule implements ValidationRule {
  name = 'title_validation';

  validate(item: FeedItem): ValidationResult {
    const title = item.title.trim();
    
    if (title.length === 0) {
      return { valid: false, severity: 'error', message: 'Empty title', field: 'title' };
    }
    if (title.length < 5) {
      return { valid: false, severity: 'error', message: `Title too short (${title.length} chars)`, field: 'title' };
    }
    if (title.length > 300) {
      return { valid: true, severity: 'warn', message: `Title very long (${title.length} chars)`, field: 'title' };
    }
    
    // Check for spam indicators
    const spamIndicators = ['click here', 'make money fast', 'act now!!!', 'free!!!'];
    const lowerTitle = title.toLowerCase();
    const spamMatch = spamIndicators.find(ind => lowerTitle.includes(ind));
    if (spamMatch) {
      return { valid: false, severity: 'error', message: `Spam indicator detected: "${spamMatch}"`, field: 'title' };
    }
    
    // Check for excessive caps (more than 70% uppercase)
    const upperRatio = (title.match(/[A-Z]/g) ?? []).length / title.length;
    if (upperRatio > 0.7 && title.length > 20) {
      return { valid: true, severity: 'warn', message: `Excessive caps (${(upperRatio * 100).toFixed(0)}%)`, field: 'title' };
    }

    return { valid: true, severity: 'warn', message: 'Title valid' };
  }
}

class DateValidationRule implements ValidationRule {
  name = 'date_validation';

  validate(item: FeedItem): ValidationResult {
    const pubDate = new Date(item.publishedAt);
    const now = new Date();
    const age = now.getTime() - pubDate.getTime();
    const maxAge = 90 * 24 * 60 * 60 * 1000; // 90 days
    const futureThreshold = 60 * 60 * 1000; // 1 hour into future allowed (clock skew)
    
    if (isNaN(pubDate.getTime())) {
      return { valid: false, severity: 'error', message: 'Invalid date format', field: 'publishedAt' };
    }
    
    if (age < -futureThreshold) {
      return { valid: true, severity: 'warn', message: 'Date is in the future', field: 'publishedAt' };
    }
    
    if (age > maxAge) {
      return { valid: true, severity: 'warn', message: `Article is ${Math.floor(age / 86400000)} days old`, field: 'publishedAt' };
    }

    return { valid: true, severity: 'warn', message: 'Date valid' };
  }
}

class ContentQualityRule implements ValidationRule {
  name = 'content_quality';

  validate(item: FeedItem): ValidationResult {
    const text = item.contentText ?? item.content ?? item.summary ?? '';
    
    if (text.length === 0 && item.summary === null) {
      // No content at all -- acceptable for some sources (e.g., social, RSS-only)
      return { valid: true, severity: 'warn', message: 'No content body available', field: 'content' };
    }
    
    // Check for placeholder content
    const placeholders = ['lorem ipsum', 'placeholder', 'coming soon', 'tbd'];
    if (placeholders.some(p => text.toLowerCase().includes(p))) {
      return { valid: false, severity: 'error', message: 'Placeholder content detected', field: 'content' };
    }

    return { valid: true, severity: 'warn', message: 'Content valid' };
  }
}

class SpamDetectionRule implements ValidationRule {
  name = 'spam_detection';

  private spamPatterns = [
    /\b(buy now|limited time|click here|subscribe now)\b/gi,
    /\$\d+\s*(discount|off|sale)/gi,
    /(free|win|winner|prize|lottery|viagra|crypto scam)/gi,
  ];

  validate(item: FeedItem): ValidationResult {
    const text = `${item.title} ${item.summary ?? ''} ${item.content ?? ''}`.toLowerCase();
    
    let spamScore = 0;
    for (const pattern of this.spamPatterns) {
      const matches = text.match(pattern);
      if (matches) spamScore += matches.length;
    }
    
    // More than 5 spam indicators = reject
    if (spamScore > 5) {
      return { valid: false, severity: 'error', message: `Spam score ${spamScore} exceeds threshold`, field: 'content' };
    }
    // 2 -- 5 indicators = warn
    if (spamScore >= 2) {
      return { valid: true, severity: 'warn', message: `Elevated spam score: ${spamScore}`, field: 'content' };
    }

    return { valid: true, severity: 'warn', message: 'Spam check passed' };
  }
}
```

#### Rejection Handling

```typescript
interface RejectedItem {
  item: FeedItem;
  reason: string;
  rule: string;
  field?: string;
  timestamp: string;
}

class DeadLetterQueue {
  constructor(private db: Knex, private redis: Redis) {}

  async storeRejected(rejected: RejectedItem[]): Promise<void> {
    if (rejected.length === 0) return;

    await this.db('rejected_items').insert(rejected.map(r => ({
      item_id: r.item.id,
      source_id: r.item.sourceId,
      title: r.item.title,
      url: r.item.rawUrl,
      rejection_reason: r.reason,
      rejection_rule: r.rule,
      rejection_field: r.field,
      item_data: JSON.stringify(r.item),
      created_at: r.timestamp,
    })));

    // Alert if rejection rate is high
    const hourlyRejectionCount = await this.db('rejected_items')
      .where('created_at', '>', new Date(Date.now() - 3600000).toISOString())
      .count('id as count')
      .first();

    if (hourlyRejectionCount && Number(hourlyRejectionCount.count) > 100) {
      await this.sendAlert(`High rejection rate: ${hourlyRejectionCount.count} items in last hour`);
    }
  }

  // Manual review endpoint -- allows reprocessing
  async getRejectedForReview(limit: number = 50): Promise<RejectedItem[]> {
    return this.db('rejected_items')
      .where('reviewed', false)
      .orderBy('created_at', 'desc')
      .limit(limit);
  }

  async reprocessItem(itemId: string): Promise<void> {
    const record = await this.db('rejected_items').where('item_id', itemId).first();
    if (!record) throw new Error('Item not found in DLQ');
    
    const item: FeedItem = JSON.parse(record.item_data);
    // Re-inject into pipeline at Validation stage
    await pipeline.reinject(item, 'validation');
    await this.db('rejected_items').where('item_id', itemId).update({ reviewed: true, reprocessed: true });
  }
}
```

---
