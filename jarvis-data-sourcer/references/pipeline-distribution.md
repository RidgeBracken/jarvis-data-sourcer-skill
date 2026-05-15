# Pipeline: Distribution

> Part of the Jarvis HQ pipeline reference. Originally a section of
> `pipeline.md`; split for progressive disclosure. **Stages covered:** Stage 7 (Categorisation), Stage 8 (Ranking), Stage 9 (Storage), Stage 10 (Distribution).
> Previous: [$prevFile](pipeline-processing.md) · Next: [$nextFile](pipeline-orchestration.md)

> **Schema authority:** `data-schema.md`. **Safety rules:** `../SKILL.md`.

### Stage 7: Categorisation

#### Purpose
Assign topics, tags, content type, and importance score to each item. This stage ensures items appear in the correct feeds and channels.

#### Input: Summarised `FeedItem[]`

#### Output: Categorised `FeedItem[]` (with `topics`, `tags`, `contentType`, `importance`)

#### Categorisation Pipeline

```typescript
class CategorisationPipeline {
  private topicClassifier: TopicClassifier;
  private contentTypeClassifier: ContentTypeClassifier;

  constructor(private db: Knex) {
    this.topicClassifier = new TopicClassifier();
    this.contentTypeClassifier = new ContentTypeClassifier();
  }

  async categorise(items: FeedItem[]): Promise<FeedItem[]> {
    return items.map(item => this.categoriseItem(item));
  }

  private categoriseItem(item: FeedItem): FeedItem {
    // 1. Topic classification (may have been pre-populated in Enrichment)
    if (item.topics.length === 0) {
      item.topics = this.topicClassifier.classify(item);
    }

    // 2. Tag assignment
    item.tags = this.assignTags(item);

    // 3. Content type classification
    item.contentType = this.contentTypeClassifier.classify(item);

    // 4. Importance scoring
    item.importance = this.calculateImportance(item);

    item.pipelineStage = 'categorised';
    return item;
  }

  private assignTags(item: FeedItem): string[] {
    const tags = new Set<string>(item.tags);
    const text = `${item.title} ${item.summary ?? ''}`.toLowerCase();

    // Auto-generate tags from content
    const tagRules: Record<string, string[]> = {
      'breaking-news': ['breaking', 'just in', 'alert', 'urgent'],
      'exclusive': ['exclusive', 'first look', 'behind the scenes'],
      'interview': ['interview', 'q&a', 'says', 'told'],
      'review': ['review', 'hands-on', 'impressions', 'score', 'rating'],
      'tutorial': ['how to', 'guide', 'tutorial', 'learn', 'getting started'],
      'opinion': ['opinion', 'editorial', 'why', 'what if', 'should'],
      'analysis': ['analysis', 'deep dive', 'explained', 'breakdown'],
      'release': ['announced', 'launch', 'released', 'available now', 'coming soon'],
      'update': ['update', 'patch', 'version', 'changelog'],
      'security': ['vulnerability', 'exploit', 'security', 'breach', 'cve'],
      'funding': ['funding', 'investment', 'million', 'billion', 'valuation'],
      'hardware': ['cpu', 'gpu', 'chip', 'processor', 'graphics card', 'console'],
    };

    for (const [tag, keywords] of Object.entries(tagRules)) {
      if (keywords.some(kw => text.includes(kw))) {
        tags.add(tag);
      }
    }

    // Company/brand tags
    const brandPatterns = [
      'openai', 'anthropic', 'google', 'microsoft', 'apple', 'meta',
      'nvidia', 'amd', 'intel', 'tsmc', 'samsung', 'sony',
      'nintendo', 'steam', 'epic', 'unity', 'roblox',
      'spacex', 'nasa', 'tesla', 'amazon', 'netflix',
    ];
    
    for (const brand of brandPatterns) {
      if (text.includes(brand)) tags.add(brand);
    }

    return [...tags];
  }

  private calculateImportance(item: FeedItem): number {
    let score = item.compositeScore * 0.5; // Base on composite score

    // Boost for certain content types
    if (item.tags.includes('breaking-news')) score += 20;
    if (item.tags.includes('exclusive')) score += 15;
    if (item.contentType === 'release') score += 10;

    // Boost for tier 1 sources
    if (item.sourceQualityScore >= 85) score += 10;

    // Boost for high-engagement topics
    if (item.topics.includes('ai') && item.tags.some(t => ['openai', 'anthropic'].includes(t))) {
      score += 5;
    }

    return Math.min(100, Math.round(score));
  }
}

class ContentTypeClassifier {
  classify(item: FeedItem): ContentType {
    const text = `${item.title} ${item.summary ?? ''}`.toLowerCase();
    const url = item.canonicalUrl.toLowerCase();

    // Video detection
    if (item.videoUrl || 
        url.includes('youtube.com') || 
        url.includes('vimeo.com') ||
        text.includes('watch:') ||
        item.tags.includes('video')) {
      return 'video';
    }

    // GitHub release detection
    if (url.includes('github.com') && 
        (url.includes('/releases/') || text.includes('release'))) {
      return 'release';
    }

    // Social media detection
    if (url.includes('twitter.com') || 
        url.includes('x.com') ||
        url.includes('mastodon') ||
        url.includes('bsky.app')) {
      return 'social';
    }

    // Podcast detection
    if (text.includes('podcast') || 
        text.includes('episode') ||
        url.includes('podcast') ||
        item.tags.includes('podcast')) {
      return 'podcast';
    }

    // Default
    return 'article';
  }
}
```

---

### Stage 8: Ranking

#### Purpose
Sort and filter items to produce the final ranked feed. Apply diversity injection, breaking news boosts, and topic-specific tier thresholds.

#### Input: Categorised `FeedItem[]`

#### Output: Ranked `FeedItem[]`

#### Ranking Algorithm

```typescript
class RankingPipeline {
  private readonly DIVERSITY_WINDOW = 5; // No 2 similar items within last 5
  private readonly BREAKING_BOOST = 15;  // Score points for breaking news
  private readonly MAX_PER_TOPIC = 20;   // Max items per topic in final feed
  private readonly MIN_TIER_THRESHOLD: Record<Topic, number> = {
    ai: 3,       // Only tiers 1-3 for AI
    tech: 3,
    gaming: 3,
    sports: 4,   // Tiers 1-4 for sports
    world: 3,
    science: 4,
    business: 3,
  };

  async rank(items: FeedItem[]): Promise<FeedItem[]> {
    // 1. Apply tier filtering
    const filtered = this.filterByTier(items);

    // 2. Apply breaking news boost
    const boosted = this.applyBreakingBoost(filtered);

    // 3. Sort by composite score
    const sorted = boosted.sort((a, b) => b.compositeScore - a.compositeScore);

    // 4. Apply diversity injection
    const diverse = this.applyDiversity(sorted);

    // 5. Balance recency vs relevance
    const balanced = this.balanceRecencyRelevance(diverse);

    // 6. Apply per-topic caps
    const capped = this.applyTopicCaps(balanced);

    // 7. Final sort
    const ranked = capped.sort((a, b) => b.compositeScore - a.compositeScore);

    // Update items
    for (const item of ranked) {
      item.pipelineStage = 'ranked';
    }

    return ranked;
  }

  private filterByTier(items: FeedItem[]): FeedItem[] {
    return items.filter(item => {
      // Use the strictest threshold across all item topics
      const maxTier = Math.min(...item.topics.map(t => this.MIN_TIER_THRESHOLD[t] ?? 5));
      return item.tier <= maxTier;
    });
  }

  private applyBreakingBoost(items: FeedItem[]): FeedItem[] {
    const now = Date.now();
    
    return items.map(item => {
      const ageHours = (now - new Date(item.publishedAt).getTime()) / 3600000;
      
      // Breaking news: items < 2 hours old with high importance
      if (ageHours <= 2 && item.importance >= 70) {
        item.compositeScore = Math.min(100, item.compositeScore + this.BREAKING_BOOST);
        item.tags.push('breaking-news');
        logger.info(`Breaking boost applied to ${item.id}: +${this.BREAKING_BOOST}`);
      }
      
      // Recent boost: items < 6 hours old
      if (ageHours <= 6 && ageHours > 2) {
        item.compositeScore = Math.min(100, item.compositeScore + 5);
      }

      return item;
    });
  }

  private applyDiversity(items: FeedItem[]): FeedItem[] {
    const selected: FeedItem[] = [];
    const topicWindow: Map<Topic, number> = new Map();

    for (const item of items) {
      // Check if any of this item's topics appear too frequently in recent window
      let tooSimilar = false;
      const recentItems = selected.slice(-this.DIVERSITY_WINDOW);
      
      for (const recent of recentItems) {
        const topicOverlap = item.topics.filter(t => recent.topics.includes(t));
        // If >50% topic overlap, consider it similar
        if (topicOverlap.length / Math.max(item.topics.length, recent.topics.length) > 0.5) {
          // Allow if scores are very different (one is significantly better)
          if (Math.abs(item.compositeScore - recent.compositeScore) < 10) {
            tooSimilar = true;
            break;
          }
        }
      }

      if (!tooSimilar) {
        selected.push(item);
      } else {
        // De-duplicate: keep the higher-scored version
        metrics.counter('ranking.diversity_filtered', 1);
      }
    }

    return selected;
  }

  private balanceRecencyRelevance(items: FeedItem[]): FeedItem[] {
    // Split into recent (< 24h) and older
    const now = Date.now();
    const recent: FeedItem[] = [];
    const older: FeedItem[] = [];

    for (const item of items) {
      const ageHours = (now - new Date(item.publishedAt).getTime()) / 3600000;
      if (ageHours <= 24) recent.push(item);
      else older.push(item);
    }

    // Ensure at least 30% of feed is recent
    const minRecentCount = Math.floor(items.length * 0.3);
    if (recent.length < minRecentCount) {
      // Pull more recent items by lowering the threshold
      const additionalRecent = older
        .filter(item => (now - new Date(item.publishedAt).getTime()) / 3600000 <= 48)
        .sort((a, b) => b.compositeScore - a.compositeScore)
        .slice(0, minRecentCount - recent.length);
      
      recent.push(...additionalRecent);
      // Remove from older
      for (const item of additionalRecent) {
        const idx = older.indexOf(item);
        if (idx >= 0) older.splice(idx, 1);
      }
    }

    // Blend: interleave recent and older
    const blended: FeedItem[] = [];
    const recentRatio = 0.6; // 60% recent, 40% older
    
    let r = 0, o = 0;
    while (r < recent.length || o < older.length) {
      // Add recent items with higher probability
      if (r < recent.length && (o >= older.length || Math.random() < recentRatio)) {
        blended.push(recent[r++]);
      } else if (o < older.length) {
        blended.push(older[o++]);
      }
    }

    return blended;
  }

  private applyTopicCaps(items: FeedItem[]): FeedItem[] {
    const topicCounts: Map<Topic, number> = new Map();
    const capped: FeedItem[] = [];

    for (const item of items) {
      // Check if any topic has reached its cap
      const canAdd = item.topics.every(topic => {
        const count = topicCounts.get(topic) ?? 0;
        return count < this.MAX_PER_TOPIC;
      });

      if (canAdd) {
        capped.push(item);
        for (const topic of item.topics) {
          topicCounts.set(topic, (topicCounts.get(topic) ?? 0) + 1);
        }
      }
    }

    return capped;
  }
}
```

---

### Stage 9: Storage

#### Purpose
Persist ranked items to PostgreSQL with upsert logic, related item linking, index maintenance, and archival of old items.

#### Input: Ranked `FeedItem[]`

#### Output: Stored `FeedItem[]` (with `processedAt` set)

#### Storage Pipeline

```typescript
class StoragePipeline {
  constructor(private db: Knex, private redis: Redis) {}

  async store(items: FeedItem[]): Promise<FeedItem[]> {
    const stored: FeedItem[] = [];

    for (const item of items) {
      try {
        const existing = await this.findExisting(item);
        
        if (existing) {
          // Update existing item
          await this.updateItem(existing, item);
        } else {
          // Insert new item
          await this.insertItem(item);
        }

        // Link related items
        await this.linkRelatedItems(item);

        item.processedAt = new Date().toISOString();
        item.pipelineStage = 'stored';
        stored.push(item);

        metrics.counter('storage.stored', 1, { operation: existing ? 'update' : 'insert' });
      } catch (err) {
        logger.error(`Storage failed for ${item.id}: ${err.message}`);
        item.lastError = `Storage: ${err.message}`;
        metrics.counter('storage.errors', 1);
      }
    }

    // Update cache indices
    await this.updateIndices(stored);

    // Trigger archival
    await this.archiveOldItems();

    return stored;
  }

  private async findExisting(item: FeedItem): Promise<{ id: string } | null> {
    // Check by content hash (most reliable)
    const byHash = await this.db('feed_items')
      .where('content_hash', item.contentHash)
      .first('id');
    if (byHash) return byHash;

    // Check by canonical URL
    const byUrl = await this.db('feed_items')
      .where('canonical_url', item.canonicalUrl)
      .first('id');
    if (byUrl) return byUrl;

    return null;
  }

  private async insertItem(item: FeedItem): Promise<void> {
    await this.db('feed_items').insert({
      id: item.id,
      source_id: item.sourceId,
      canonical_url: item.canonicalUrl,
      raw_url: item.rawUrl,
      content_hash: item.contentHash,
      title: item.title,
      summary: item.summary,
      content: item.content,
      content_text: item.contentText,
      author: item.author,
      language: item.language,
      image_url: item.imageUrl,
      video_url: item.videoUrl,
      reading_time_minutes: item.readingTimeMinutes,
      published_at: item.publishedAt,
      collected_at: item.collectedAt,
      processed_at: new Date().toISOString(),
      freshness_score: item.freshnessScore,
      topics: JSON.stringify(item.topics),
      tags: JSON.stringify(item.tags),
      content_type: item.contentType,
      importance: item.importance,
      source_quality_score: item.sourceQualityScore,
      content_quality_score: item.contentQualityScore,
      composite_score: item.compositeScore,
      tier: item.tier,
      duplicate_group_id: item.duplicateGroupId,
      is_primary: item.isPrimary,
      metadata: JSON.stringify(item.metadata),
    });
  }

  private async updateItem(existing: { id: string }, item: FeedItem): Promise<void> {
    // Only update fields that may have improved
    await this.db('feed_items')
      .where('id', existing.id)
      .update({
        title: item.title, // May have been enriched
        summary: this.db.raw('COALESCE(?, summary)', [item.summary]),
        content: this.db.raw('COALESCE(?, content)', [item.content]),
        content_text: this.db.raw('COALESCE(?, content_text)', [item.contentText]),
        author: this.db.raw('COALESCE(?, author)', [item.author]),
        image_url: this.db.raw('COALESCE(?, image_url)', [item.imageUrl]),
        video_url: this.db.raw('COALESCE(?, video_url)', [item.videoUrl]),
        reading_time_minutes: item.readingTimeMinutes,
        freshness_score: item.freshnessScore,
        topics: JSON.stringify(item.topics),
        tags: JSON.stringify(item.tags),
        content_type: item.contentType,
        importance: item.importance,
        composite_score: item.compositeScore,
        tier: item.tier,
        processed_at: new Date().toISOString(),
        metadata: JSON.stringify(item.metadata),
        updated_at: new Date().toISOString(),
      });

    // Merge metadata
    item.id = existing.id; // Use existing ID
  }

  private async linkRelatedItems(item: FeedItem): Promise<void> {
    // Find related items (same topic, within 7 days, similar score)
    const related = await this.db('feed_items')
      .whereRaw('topics ?| ?', [item.topics])
      .where('id', '!=', item.id)
      .where('published_at', '>', new Date(Date.now() - 7 * 86400000).toISOString())
      .whereBetween('composite_score', [item.compositeScore - 15, item.compositeScore + 15])
      .limit(5)
      .select('id', 'title');

    for (const rel of related) {
      await this.db('related_items')
        .insert({
          item_id: item.id,
          related_item_id: rel.id,
          relationship_type: 'topic_similarity',
          score: item.compositeScore,
        })
        .onConflict(['item_id', 'related_item_id'])
        .ignore();
    }
  }

  private async updateIndices(items: FeedItem[]): Promise<void> {
    // Update Redis sorted sets for fast feed queries
    const pipeline = this.redis.pipeline();

    for (const item of items) {
      for (const topic of item.topics) {
        // Add to topic sorted set (score = composite score)
        pipeline.zadd(`feed:${topic}`, item.compositeScore, item.id);
        
        // Add to tier sorted set
        pipeline.zadd(`feed:${topic}:tier${item.tier}`, item.compositeScore, item.id);
        
        // Set expiry on sorted set members (7 days)
        pipeline.expire(`feed:${topic}`, 7 * 86400);
      }

      // Store item hash for quick lookup
      pipeline.hset(`item:${item.id}`, {
        title: item.title,
        url: item.canonicalUrl,
        score: item.compositeScore,
        tier: item.tier,
        topics: JSON.stringify(item.topics),
        published_at: item.publishedAt,
      });
      pipeline.expire(`item:${item.id}`, 7 * 86400);
    }

    await pipeline.exec();
  }

  private async archiveOldItems(): Promise<number> {
    const cutoff = new Date(Date.now() - 90 * 86400000).toISOString(); // 90 days

    // Find old items
    const oldItems = await this.db('feed_items')
      .where('published_at', '<', cutoff)
      .where('archived', false)
      .limit(1000)
      .select('id', 'metadata');

    if (oldItems.length === 0) return 0;

    // Move to archive table
    await this.db('feed_items_archive').insert(
      oldItems.map(item => ({
        item_id: item.id,
        item_data: item.metadata,
        archived_at: new Date().toISOString(),
      }))
    );

    // Mark as archived (soft delete from main table)
    await this.db('feed_items')
      .whereIn('id', oldItems.map(i => i.id))
      .update({ archived: true });

    logger.info(`Archived ${oldItems.length} old items`);
    metrics.counter('storage.archived', oldItems.length);

    return oldItems.length;
  }
}
```

#### Database Schema

```sql
-- Main items table
CREATE TABLE feed_items (
    id UUID PRIMARY KEY,
    source_id VARCHAR(64) NOT NULL REFERENCES sources(id),
    canonical_url TEXT NOT NULL UNIQUE,
    raw_url TEXT NOT NULL,
    content_hash VARCHAR(64) NOT NULL,
    title TEXT NOT NULL,
    summary TEXT,
    content TEXT,
    content_text TEXT,
    author VARCHAR(255),
    language VARCHAR(10),
    image_url TEXT,
    video_url TEXT,
    reading_time_minutes INTEGER,
    published_at TIMESTAMPTZ NOT NULL,
    collected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMPTZ,
    freshness_score DECIMAL(5,2) DEFAULT 100.00,
    topics JSONB NOT NULL DEFAULT '[]',
    tags JSONB NOT NULL DEFAULT '[]',
    content_type VARCHAR(20) DEFAULT 'article',
    importance INTEGER DEFAULT 0,
    source_quality_score INTEGER DEFAULT 0,
    content_quality_score INTEGER DEFAULT 0,
    composite_score INTEGER DEFAULT 0,
    tier INTEGER DEFAULT 5,
    duplicate_group_id UUID,
    is_primary BOOLEAN DEFAULT true,
    metadata JSONB DEFAULT '{}',
    archived BOOLEAN DEFAULT false,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_feed_items_published_at ON feed_items(published_at DESC);
CREATE INDEX idx_feed_items_composite_score ON feed_items(composite_score DESC);
CREATE INDEX idx_feed_items_tier ON feed_items(tier);
CREATE INDEX idx_feed_items_topics ON feed_items USING GIN(topics);
CREATE INDEX idx_feed_items_tags ON feed_items USING GIN(tags);
CREATE INDEX idx_feed_items_content_hash ON feed_items(content_hash);
CREATE INDEX idx_feed_items_collected_at ON feed_items(collected_at DESC);
CREATE INDEX idx_feed_items_archived ON feed_items(archived) WHERE archived = false;

-- Scores table
CREATE TABLE item_scores (
    item_id UUID PRIMARY KEY REFERENCES feed_items(id),
    source_quality INTEGER,
    content_quality INTEGER,
    freshness_score DECIMAL(5,2),
    composite_score INTEGER,
    tier INTEGER,
    components JSONB,
    computed_at TIMESTAMPTZ
);

-- Related items
CREATE TABLE related_items (
    item_id UUID REFERENCES feed_items(id),
    related_item_id UUID REFERENCES feed_items(id),
    relationship_type VARCHAR(50),
    score INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (item_id, related_item_id)
);

-- Archive table
CREATE TABLE feed_items_archive (
    id SERIAL PRIMARY KEY,
    item_id UUID NOT NULL,
    item_data JSONB,
    archived_at TIMESTAMPTZ DEFAULT NOW()
);

-- Rejected items (dead letter queue)
CREATE TABLE rejected_items (
    id SERIAL PRIMARY KEY,
    item_id UUID NOT NULL,
    source_id VARCHAR(64),
    title TEXT,
    url TEXT,
    rejection_reason TEXT NOT NULL,
    rejection_rule VARCHAR(100),
    rejection_field VARCHAR(100),
    item_data JSONB,
    reviewed BOOLEAN DEFAULT false,
    reprocessed BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

### Stage 10: Distribution

#### Purpose
Deliver stored items to all configured output channels: API endpoints, Telegram notifications, daily digest generation, n8n webhooks, and website cache invalidation.

#### Input: Stored `FeedItem[]`

#### Output: Multi-channel delivery confirmations

#### Distribution Pipeline

```typescript
class DistributionPipeline {
  private channels: DistributionChannel[] = [];

  constructor(config: DistributionConfig) {
    this.channels = [
      new ApiCacheChannel(config.api),
      new TelegramChannel(config.telegram),
      new DigestChannel(config.digest),
      new N8nWebhookChannel(config.n8n),
      new WebsiteCacheChannel(config.website),
    ];
  }

  async distribute(items: FeedItem[]): Promise<DistributionResult> {
    const results: DistributionResult = {
      api: 0,
      telegram: 0,
      digest: 0,
      n8n: 0,
      website: 0,
      errors: [],
    };

    for (const channel of this.channels) {
      try {
        const delivered = await channel.deliver(items);
        results[channel.name] = delivered;
        metrics.counter(`distribution.${channel.name}`, delivered);
      } catch (err) {
        logger.error(`Distribution failed for ${channel.name}: ${err.message}`);
        results.errors.push({ channel: channel.name, error: err.message });
        metrics.counter('distribution.errors', 1, { channel: channel.name });
      }
    }

    // Update item stage
    for (const item of items) {
      item.pipelineStage = 'distributed';
    }

    return results;
  }
}

// Base channel interface
interface DistributionChannel {
  name: string;
  deliver(items: FeedItem[]): Promise<number>;
}

// Channel 1: API Cache Update
class ApiCacheChannel implements DistributionChannel {
  name = 'api';

  constructor(private config: { redis: Redis; db: Knex }) {}

  async deliver(items: FeedItem[]): Promise<number> {
    // Update Redis caches for API endpoints
    const pipeline = this.config.redis.pipeline();
    
    for (const item of items) {
      for (const topic of item.topics) {
        // Invalidate cache for this topic feed
        pipeline.del(`api:feed:${topic}`);
        pipeline.del(`api:feed:${topic}:meta`);
        
        // Update today's count
        const today = new Date().toISOString().split('T')[0];
        pipeline.hincrby(`stats:${topic}:${today}`, 'new_items', 1);
      }
    }

    await pipeline.exec();
    
    // Pre-warm cache for popular endpoints
    for (const topic of new Set(items.flatMap(i => i.topics))) {
      await this.prewarmCache(topic);
    }

    return items.length;
  }

  private async prewarmCache(topic: string): Promise<void> {
    const items = await this.config.db('feed_items')
      .whereRaw('topics ? ?', [topic])
      .where('archived', false)
      .orderBy('composite_score', 'desc')
      .limit(50)
      .select('*');

    const formatted = items.map(i => this.formatApiItem(i));
    
    await this.config.redis.setex(
      `api:feed:${topic}`,
      300, // 5 min TTL
      JSON.stringify(formatted)
    );
  }

  private formatApiItem(item: any) {
    return {
      id: item.id,
      title: item.title,
      summary: item.summary,
      url: item.canonical_url,
      imageUrl: item.image_url,
      author: item.author,
      publishedAt: item.published_at,
      readingTime: item.reading_time_minutes,
      score: item.composite_score,
      tier: item.tier,
      topics: item.topics,
      tags: item.tags,
      contentType: item.content_type,
    };
  }
}

// Channel 2: Telegram Notifications
class TelegramChannel implements DistributionChannel {
  name = 'telegram';

  constructor(private config: { botToken: string; chatId: string }) {}

  async deliver(items: FeedItem[]): Promise<number> {
    // Only send breaking/high-tier items to Telegram
    const telegramItems = items.filter(item => 
      item.tier <= 2 || item.tags.includes('breaking-news')
    );

    let sent = 0;
    for (const item of telegramItems) {
      try {
        await this.sendTelegramMessage(item);
        sent++;
        // Rate limit: max 1 message per 2 seconds
        await new Promise(r => setTimeout(r, 2000));
      } catch (err) {
        logger.error(`Telegram send failed for ${item.id}: ${err.message}`);
      }
    }

    return sent;
  }

  private async sendTelegramMessage(item: FeedItem): Promise<void> {
    const emoji = this.getTopicEmoji(item.topics);
    const tierBadge = this.getTierBadge(item.tier);
    
    const text = this.formatTelegramMessage(item, emoji, tierBadge);

    const response = await fetch(`https://api.telegram.org/bot${this.config.botToken}/sendMessage`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        chat_id: this.config.chatId,
        text,
        parse_mode: 'MarkdownV2',
        disable_web_page_preview: false,
      }),
    });

    if (!response.ok) {
      throw new Error(`Telegram API error: ${response.status}`);
    }
  }

  private formatTelegramMessage(item: FeedItem, emoji: string, badge: string): string {
    // Escape markdown characters
    const escape = (text: string) => text.replace(/[_*\[\]()~`>#+\-=|{}.!]/g, '\\$&');
    
    const lines = [
      `${emoji} ${badge} ${escape(item.title)}`,
      '',
      item.summary ? escape(item.summary.slice(0, 200)) : '',
      '',
      `🏷 ${escape(item.topics.join(', '))} | ⏱ ${item.readingTimeMinutes}min | ⭐ ${item.compositeScore}`,
      '',
      `[Read more](${item.canonicalUrl})`,
    ];

    return lines.filter(Boolean).join('\n');
  }

  private getTopicEmoji(topics: Topic[]): string {
    const emojiMap: Record<string, string> = {
      ai: '🤖', gaming: '🎮', tech: '💻', sports: '⚽',
      world: '🌍', science: '🔬', business: '💼',
    };
    return emojiMap[topics[0]] ?? '📰';
  }

  private getTierBadge(tier: number): string {
    const badges: Record<number, string> = {
      1: '🔴 BREAKING',
      2: '🟠 HIGH',
      3: '🟡 GOOD',
      4: '🔵 AVERAGE',
      5: '⚪ LOW',
    };
    return badges[tier] ?? '';
  }
}

// Channel 3: Daily Digest Queue
class DigestChannel implements DistributionChannel {
  name = 'digest';

  constructor(private config: { redis: Redis }) {}

  async deliver(items: FeedItem[]): Promise<number> {
    // Queue items for the daily digest
    // Actual digest generation happens via cron job at 7am
    const pipeline = this.config.redis.pipeline();
    
    for (const item of items) {
      const digestEntry = JSON.stringify({
        id: item.id,
        title: item.title,
        summary: item.summary,
        url: item.canonicalUrl,
        topics: item.topics,
        tier: item.tier,
        score: item.compositeScore,
        publishedAt: item.publishedAt,
        readingTime: item.readingTimeMinutes,
      });

      // Add to digest queue, scored by composite score
      pipeline.zadd('digest:pending', item.compositeScore, digestEntry);
    }

    await pipeline.exec();
    return items.length;
  }

  // Called by cron job at 7am
  static async generateDailyDigest(redis: Redis, db: Knex): Promise<string> {
    // Get all items from the last 24 hours
    const yesterday = new Date(Date.now() - 86400000).toISOString();
    
    const items = await db('feed_items')
      .where('collected_at', '>', yesterday)
      .where('tier', '<=', 3) // Only tiers 1-3 for digest
      .orderBy('composite_score', 'desc')
      .limit(30);

    if (items.length === 0) {
      return 'No notable news today. Check back tomorrow!';
    }

    // Group by topic
    const byTopic = new Map<string, typeof items>();
    for (const item of items) {
      for (const topic of item.topics) {
        if (!byTopic.has(topic)) byTopic.set(topic, []);
        byTopic.get(topic)!.push(item);
      }
    }

    // Generate markdown digest
    const date = new Date().toLocaleDateString('en-US', { 
      weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' 
    });

    const lines = [
      `# 📰 Jarvis HQ Daily Digest`,
      `## ${date}`,
      '',
      `**${items.length} stories** from ${byTopic.size} topics`,
      '',
      '---',
      '',
    ];

    for (const [topic, topicItems] of byTopic) {
      const emojiMap: Record<string, string> = {
        ai: '🤖', gaming: '🎮', tech: '💻', sports: '⚽',
        world: '🌍', science: '🔬', business: '💼',
      };
      const emoji = emojiMap[topic] ?? '📌';
      
      lines.push(`### ${emoji} ${topic.toUpperCase()}`, '');
      
      for (const item of topicItems.slice(0, 5)) {
        const stars = '⭐'.repeat(Math.min(5, Math.ceil(item.composite_score / 20)));
        lines.push(`**${item.title}** ${stars}`);
        lines.push(`${item.summary ?? ''}`);
        lines.push(`[Read more](${item.canonical_url}) | ${item.reading_time_minutes ?? '?'} min read`);
        lines.push('');
      }
      
      lines.push('---', '');
    }

    // Clear the pending queue
    await redis.del('digest:pending');

    return lines.join('\n');
  }
}

// Channel 4: n8n Webhook Trigger
class N8nWebhookChannel implements DistributionChannel {
  name = 'n8n';

  constructor(private config: { webhookUrl: string; secret: string }) {}

  async deliver(items: FeedItem[]): Promise<number> {
    // Batch items for n8n to avoid rate limits
    const batchSize = 10;
    let sent = 0;

    for (let i = 0; i < items.length; i += batchSize) {
      const batch = items.slice(i, i + batchSize);
      
      try {
        const response = await fetch(this.config.webhookUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Webhook-Secret': this.config.secret,
          },
          body: JSON.stringify({
            event: 'new_items',
            timestamp: new Date().toISOString(),
            count: batch.length,
            items: batch.map(item => ({
              id: item.id,
              title: item.title,
              url: item.canonicalUrl,
              topics: item.topics,
              tier: item.tier,
              score: item.compositeScore,
              publishedAt: item.publishedAt,
            })),
          }),
        });

        if (response.ok) {
          sent += batch.length;
        } else {
          throw new Error(`Webhook returned ${response.status}`);
        }
      } catch (err) {
        logger.error(`n8n webhook failed: ${err.message}`);
      }

      // Rate limit: 1 request per second
      if (i + batchSize < items.length) {
        await new Promise(r => setTimeout(r, 1000));
      }
    }

    return sent;
  }
}

// Channel 5: Website Cache Invalidation
class WebsiteCacheChannel implements DistributionChannel {
  name = 'website';

  constructor(private config: { 
    nextJsUrl: string; 
    revalidateToken: string;
    redis: Redis;
  }) {}

  async deliver(items: FeedItem[]): Promise<number> {
    // Collect unique topics that need cache invalidation
    const topicsToInvalidate = new Set<Topic>();
    
    for (const item of items) {
      for (const topic of item.topics) {
        topicsToInvalidate.add(topic);
      }
    }

    // Invalidate Next.js ISR cache for each topic page
    for (const topic of topicsToInvalidate) {
      try {
        // Method 1: Next.js on-demand revalidation
        const response = await fetch(`${this.config.nextJsUrl}/api/revalidate`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.config.revalidateToken}`,
          },
          body: JSON.stringify({
            path: `/feed/${topic}`,
            secret: this.config.revalidateToken,
          }),
        });

        if (!response.ok) {
          logger.warn(`Cache invalidation failed for ${topic}: ${response.status}`);
        }

        // Method 2: Clear Redis cache keys
        await this.config.redis.del(`page:feed:${topic}`);
        await this.config.redis.del(`page:index`);

        logger.info(`Cache invalidated for topic: ${topic}`);
      } catch (err) {
        logger.error(`Cache invalidation error for ${topic}: ${err.message}`);
      }
    }

    return items.length;
  }
}
```

---
