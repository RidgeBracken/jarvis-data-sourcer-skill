# Pipeline: Processing

> Part of the Jarvis HQ pipeline reference. Originally a section of
> `pipeline.md`; split for progressive disclosure. **Stages covered:** Stage 3 (Deduplication), Stage 4 (Enrichment), Stage 5 (Scoring), Stage 6 (Summarisation).
> Previous: [$prevFile](pipeline-collection.md) · Next: [$nextFile](pipeline-distribution.md)

> **Schema authority:** `data-schema.md`. **Safety rules:** `../SKILL.md`.

### Stage 3: Deduplication

#### Purpose
Prevent the same news story from appearing multiple times in the output. The deduplication pipeline uses a **5-stage cascade** that progressively relaxes matching criteria, ensuring both exact duplicates and near-duplicates are caught.

#### Input: Validated `FeedItem[]`

#### Output: Unique `FeedItem[]` (with `duplicateGroupId` and `isPrimary` set)

#### 5-Stage Dedup Pipeline

```
  Validated FeedItem[]
          |
          v
  +-------------------------+
  | Stage 1: Exact URL      |  -- SELECT canonical_url FROM items WHERE collected_at > now() - 48h
  |         Match           |  -- If match: mark as duplicate of existing
  +-------------------------+
          | No match
          v
  +-------------------------+
  | Stage 2: Canonical URL  |  -- Normalise URL further (remove fragments, sort params)
  |         Match           |  -- Handles: m.example.com vs example.com, trailing slashes
  +-------------------------+
          | No match
          v
  +-------------------------+
  | Stage 3: Title Similar  |  -- Levenshtein distance > 80% AND same source topic
  |         (Levenshtein)   |  -- Catches: "Apple Announces iPhone 16" vs "Apple announces iPhone 16"
  +-------------------------+
          | No match
          v
  +-------------------------+
  | Stage 4: Content MinHash |  -- SimHash/MinHash Jaccard similarity > 0.7
  |         Similarity      |  -- Catches: rewritten articles, press releases
  +-------------------------+
          | No match
          v
  +-------------------------+
  | Stage 5: Time Window    |  -- Within 48h AND high title/content overlap
  |         + Heuristic     |  -- Catches: same story, different angles
  +-------------------------+
          |
          v
     UNIQUE ITEM  |  DUPLICATE --> Merge metadata, keep best source
```

#### Deduplication Engine

```typescript
interface DedupResult {
  unique: FeedItem[];
  duplicates: DuplicateGroup[];
}

interface DuplicateGroup {
  groupId: string;
  primaryItemId: string;
  duplicateItemIds: string[];
  mergeMetadata: Record<string, unknown>;
}

class DeduplicationPipeline {
  private readonly LEVENSHTEIN_THRESHOLD = 0.80;
  private readonly MINHASH_THRESHOLD = 0.70;
  private readonly TIME_WINDOW_HOURS = 48;
  private readonly lsh: MinHashLSH;

  constructor(private db: Knex, private redis: Redis) {
    // LSH (Locality-Sensitive Hashing) for efficient similarity search
    this.lsh = new MinHashLSH({ numBands: 20, rowsPerBand: 5 });
  }

  async deduplicate(items: FeedItem[]): Promise<DedupResult> {
    const unique: FeedItem[] = [];
    const duplicateGroups: DuplicateGroup[] = [];

    // Fetch recent items from database for comparison
    const recentItems = await this.getRecentItems(this.TIME_WINDOW_HOURS);
    
    // Build LSH index from recent items
    await this.buildLshIndex(recentItems);

    for (const item of items) {
      let duplicateOf: FeedItem | null = null;
      let matchStage = 0;

      // Stage 1: Exact URL match
      duplicateOf = this.exactUrlMatch(item, recentItems);
      matchStage = duplicateOf ? 1 : 0;

      // Stage 2: Canonical URL match
      if (!duplicateOf) {
        duplicateOf = this.canonicalUrlMatch(item, recentItems);
        matchStage = duplicateOf ? 2 : 0;
      }

      // Stage 3: Title Levenshtein similarity
      if (!duplicateOf) {
        duplicateOf = await this.titleSimilarityMatch(item, recentItems);
        matchStage = duplicateOf ? 3 : 0;
      }

      // Stage 4: MinHash content similarity
      if (!duplicateOf) {
        duplicateOf = await this.minHashMatch(item, recentItems);
        matchStage = duplicateOf ? 4 : 0;
      }

      // Stage 5: Time window + heuristic match
      if (!duplicateOf) {
        duplicateOf = this.timeWindowHeuristicMatch(item, recentItems);
        matchStage = duplicateOf ? 5 : 0;
      }

      if (duplicateOf) {
        // This is a duplicate -- merge and keep primary
        const group = await this.handleDuplicate(item, duplicateOf, matchStage);
        if (group) duplicateGroups.push(group);
      } else {
        // Unique item -- add to results and index
        item.duplicateGroupId = null;
        item.isPrimary = true;
        item.pipelineStage = 'deduplicated';
        unique.push(item);
        await this.addToLshIndex(item);
      }
    }

    // Metrics
    metrics.gauge('dedup.unique', unique.length);
    metrics.gauge('dedup.duplicates', duplicateGroups.length);
    metrics.counter('dedup.stage_hits', matchStage);

    return { unique, duplicates: duplicateGroups };
  }

  // Stage 1: Exact URL match
  private exactUrlMatch(item: FeedItem, recentItems: FeedItem[]): FeedItem | null {
    return recentItems.find(r => r.canonicalUrl === item.canonicalUrl) ?? null;
  }

  // Stage 2: Canonical URL match (further normalised)
  private canonicalUrlMatch(item: FeedItem, recentItems: FeedItem[]): FeedItem | null {
    const normalised = this.furtherNormaliseUrl(item.canonicalUrl);
    return recentItems.find(r => this.furtherNormaliseUrl(r.canonicalUrl) === normalised) ?? null;
  }

  private furtherNormaliseUrl(url: string): string {
    try {
      const parsed = new URL(url);
      // Remove www.
      parsed.hostname = parsed.hostname.replace(/^www\./, '');
      // Remove mobile subdomains
      parsed.hostname = parsed.hostname.replace(/^m\./, '');
      // Sort query parameters
      const sortedParams = new URLSearchParams([...parsed.searchParams.entries()].sort());
      parsed.search = sortedParams.toString() ? '?' + sortedParams.toString() : '';
      return parsed.toString();
    } catch {
      return url;
    }
  }

  // Stage 3: Title Levenshtein similarity
  private async titleSimilarityMatch(item: FeedItem, recentItems: FeedItem[]): Promise<FeedItem | null> {
    const itemTitle = item.title.toLowerCase().trim();
    
    // Only compare with items in same topic and within time window
    const candidates = recentItems.filter(r => {
      const hoursDiff = Math.abs(
        new Date(item.publishedAt).getTime() - new Date(r.publishedAt).getTime()
      ) / 3600000;
      return hoursDiff <= this.TIME_WINDOW_HOURS;
    });

    for (const candidate of candidates) {
      const candidateTitle = candidate.title.toLowerCase().trim();
      const similarity = this.levenshteinSimilarity(itemTitle, candidateTitle);
      
      if (similarity >= this.LEVENSHTEIN_THRESHOLD) {
        logger.debug(`Title match: "${item.title}" ~ "${candidate.title}" = ${similarity.toFixed(2)}`);
        metrics.counter('dedup.title_match', 1);
        return candidate;
      }
    }
    return null;
  }

  private levenshteinSimilarity(a: string, b: string): number {
    if (a === b) return 1.0;
    const distance = levenshtein(a, b);
    const maxLen = Math.max(a.length, b.length);
    return 1 - (distance / maxLen);
  }

  // Stage 4: MinHash content similarity
  private async minHashMatch(item: FeedItem, recentItems: FeedItem[]): Promise<FeedItem | null> {
    const text = (item.contentText ?? item.content ?? item.title).slice(0, 5000);
    
    // Use LSH for efficient candidate retrieval
    const candidateIds = await this.lsh.query(text);
    const candidates = recentItems.filter(r => candidateIds.includes(r.id));

    for (const candidate of candidates) {
      const candidateText = (candidate.contentText ?? candidate.content ?? candidate.title).slice(0, 5000);
      const similarity = await this.minHashSimilarity(text, candidateText);
      
      if (similarity >= this.MINHASH_THRESHOLD) {
        logger.debug(`MinHash match: ${item.id} ~ ${candidate.id} = ${similarity.toFixed(2)}`);
        metrics.counter('dedup.minhash_match', 1);
        return candidate;
      }
    }
    return null;
  }

  private async minHashSimilarity(textA: string, textB: string): Promise<number> {
    // Use MinHash with k-shingles
    const k = 5; // 5-gram shingles
    const numHashFunctions = 128;
    
    const shinglesA = this.getShingles(textA.toLowerCase(), k);
    const shinglesB = this.getShingles(textB.toLowerCase(), k);
    
    const minHashA = this.computeMinHash(shinglesA, numHashFunctions);
    const minHashB = this.computeMinHash(shinglesB, numHashFunctions);
    
    // Jaccard similarity estimate
    let matches = 0;
    for (let i = 0; i < numHashFunctions; i++) {
      if (minHashA[i] === minHashB[i]) matches++;
    }
    return matches / numHashFunctions;
  }

  private getShingles(text: string, k: number): Set<string> {
    const shingles = new Set<string>();
    // Simple word-level shingles
    const words = text.split(/\s+/).filter(w => w.length > 2);
    for (let i = 0; i <= words.length - k; i++) {
      shingles.add(words.slice(i, i + k).join(' '));
    }
    return shingles;
  }

  private computeMinHash(shingles: Set<string>, numHash: number): number[] {
    const shinglesArray = Array.from(shingles);
    const signatures: number[] = [];
    
    for (let h = 0; h < numHash; h++) {
      let minHash = Infinity;
      for (const shingle of shinglesArray) {
        const hash = this.hashString(`${h}:${shingle}`);
        if (hash < minHash) minHash = hash;
      }
      signatures.push(minHash);
    }
    return signatures;
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash |= 0; // Convert to 32bit integer
    }
    return Math.abs(hash);
  }

  // Stage 5: Time window + heuristic match
  private timeWindowHeuristicMatch(item: FeedItem, recentItems: FeedItem[]): FeedItem | null {
    const itemDate = new Date(item.publishedAt);
    const itemTokens = new Set(this.tokenize(item.title));
    
    const candidates = recentItems.filter(r => {
      const hoursDiff = Math.abs(itemDate.getTime() - new Date(r.publishedAt).getTime()) / 3600000;
      return hoursDiff <= this.TIME_WINDOW_HOURS;
    });

    for (const candidate of candidates) {
      const candidateTokens = new Set(this.tokenize(candidate.title));
      
      // Jaccard similarity of title token sets
      const intersection = new Set([...itemTokens].filter(x => candidateTokens.has(x)));
      const union = new Set([...itemTokens, ...candidateTokens]);
      const tokenJaccard = intersection.size / union.size;
      
      // Require high token overlap + significant token count
      if (tokenJaccard > 0.6 && intersection.size >= 4) {
        logger.debug(`Heuristic match: "${item.title}" ~ "${candidate.title}" Jaccard=${tokenJaccard.toFixed(2)}`);
        metrics.counter('dedup.heuristic_match', 1);
        return candidate;
      }
    }
    return null;
  }

  private tokenize(text: string): string[] {
    return text.toLowerCase()
      .replace(/[^a-z0-9\s]/g, '')
      .split(/\s+/)
      .filter(w => w.length > 2 && !this.stopWords.has(w));
  }

  private stopWords = new Set([
    'the', 'and', 'for', 'are', 'but', 'not', 'you', 'all', 'can', 'had',
    'her', 'was', 'one', 'our', 'out', 'day', 'get', 'has', 'him', 'his'
  ]);

  // Handle duplicate: merge metadata, keep primary
  private async handleDuplicate(
    duplicate: FeedItem, 
    primary: FeedItem,
    matchStage: number
  ): Promise<DuplicateGroup> {
    const groupId = primary.duplicateGroupId ?? crypto.randomUUID();
    
    // Ensure primary has group ID
    if (!primary.duplicateGroupId) {
      primary.duplicateGroupId = groupId;
      primary.isPrimary = true;
    }

    // Merge metadata from duplicate into primary (keep best)
    const merged = this.mergeItems(primary, duplicate);
    
    // Store duplicate record
    await this.db('duplicate_items').insert({
      group_id: groupId,
      primary_item_id: primary.id,
      duplicate_item_id: duplicate.id,
      match_stage: matchStage,
      duplicate_source_id: duplicate.sourceId,
      duplicate_url: duplicate.rawUrl,
      merged_metadata: JSON.stringify(merged),
      created_at: new Date().toISOString(),
    });

    return {
      groupId,
      primaryItemId: primary.id,
      duplicateItemIds: [duplicate.id],
      mergeMetadata: merged,
    };
  }

  // Source hierarchy for choosing primary: prefer higher quality sources
  private mergeItems(primary: FeedItem, duplicate: FeedItem): Record<string, unknown> {
    const sourceHierarchy = this.getSourceQualityScore(primary.sourceId);
    const dupHierarchy = this.getSourceQualityScore(duplicate.sourceId);

    // Keep the item from the higher-quality source as primary
    if (dupHierarchy > sourceHierarchy) {
      // Swap: duplicate has better source
      duplicate.isPrimary = true;
      duplicate.duplicateGroupId = primary.duplicateGroupId;
      primary.isPrimary = false;
    }

    // Merge: keep best fields from both
    return {
      title: primary.title.length >= duplicate.title.length ? primary.title : duplicate.title,
      summary: primary.summary ?? duplicate.summary,
      content: primary.content ?? duplicate.content,
      imageUrl: primary.imageUrl ?? duplicate.imageUrl,
      author: primary.author ?? duplicate.author,
      sources: [primary.sourceId, duplicate.sourceId],
    };
  }

  private getSourceQualityScore(sourceId: string): number {
    // This would look up from the source registry
    // Higher score = more preferred as primary
    return 50; // Placeholder
  }

  private async getRecentItems(hours: number): Promise<FeedItem[]> {
    const cutoff = new Date(Date.now() - hours * 3600000).toISOString();
    return this.db('feed_items')
      .where('collected_at', '>', cutoff)
      .where('is_primary', true)
      .select('*');
  }

  private async buildLshIndex(items: FeedItem[]): Promise<void> {
    for (const item of items) {
      await this.addToLshIndex(item);
    }
  }

  private async addToLshIndex(item: FeedItem): Promise<void> {
    const text = (item.contentText ?? item.content ?? item.title).slice(0, 5000);
    await this.lsh.insert(item.id, text);
  }
}
```

#### Source Hierarchy for Primary Selection

When two items are duplicates, the primary is chosen using this hierarchy:

| Priority | Factor | Rule |
|----------|--------|------|
| 1 | Source quality score | Higher score wins |
| 2 | Content completeness | Item with more fields populated wins |
| 3 | Content length | Longer content wins |
| 4 | Collection time | Earlier collection wins (first seen) |
| 5 | URL permanence | Items from permalink URLs win over redirect URLs |

---

### Stage 4: Enrichment

#### Purpose
Enrich each unique item with additional metadata extracted from the content: author identification, date normalisation, language detection, image extraction, reading time estimation, and topic keyword matching.

#### Input: Unique `FeedItem[]`

#### Output: Enriched `FeedItem[]`

#### Enrichment Pipeline

```
  Unique FeedItem[]
       |
       v
+------------------------+
| 1. Content Extraction  |  -- HTML -> plain text, strip boilerplate
|    (if content present)|
+------------------------+
       |
       v
+------------------------+
| 2. Author Extraction   |  -- Meta tags, JSON-LD, byline selectors
|    (if missing)        |
+------------------------+
       |
       v
+------------------------+
| 3. Date Normalisation  |  -- Parse any format -> ISO 8601 UTC
|    (if needed)         |
+------------------------+
       |
       v
+------------------------+
| 4. Language Detection  |  -- franc.js on content or title
|    (if unknown)        |
+------------------------+
       |
       v
+------------------------+
| 5. Image Extraction    |  -- OG tags, Twitter cards, first image
|    (if missing)        |
+------------------------+
       |
       v
+------------------------+
| 6. Reading Time        |  -- Words / 200 WPM average
|                        |
+------------------------+
       |
       v
+------------------------+
| 7. Topic Keywords      |  -- Match title/content against keyword lists
|                        |
+------------------------+
       |
       v
  Enriched FeedItem[]
```

#### Enrichment Engine

```typescript
class EnrichmentPipeline {
  constructor(
    private readability: ReadabilityService,
    private franc: typeof import('franc'),
    private db: Knex
  ) {}

  async enrich(items: FeedItem[]): Promise<FeedItem[]> {
    return Promise.all(items.map(item => this.enrichItem(item)));
  }

  private async enrichItem(item: FeedItem): Promise<FeedItem> {
    try {
      // 1. Content extraction (HTML -> plain text)
      if (item.content && !item.contentText) {
        item.contentText = this.extractPlainText(item.content);
      }

      // 2. Author extraction
      if (!item.author) {
        item.author = await this.extractAuthor(item);
      }

      // 3. Date normalisation
      item.publishedAt = this.normaliseDate(item.publishedAt);

      // 4. Language detection (if still unknown)
      if (item.language === 'unknown') {
        const text = (item.contentText ?? item.title).slice(0, 1000);
        item.language = this.franc.franc(text) || 'en';
      }

      // 5. Image extraction
      if (!item.imageUrl) {
        item.imageUrl = await this.extractImage(item);
      }

      // 6. Reading time
      const wordCount = (item.contentText ?? item.title).split(/\s+/).length;
      item.readingTimeMinutes = Math.max(1, Math.ceil(wordCount / 200));

      // 7. Topic keyword matching
      const topicMatch = this.matchTopics(item);
      if (topicMatch.length > 0 && item.topics.length === 0) {
        item.topics = topicMatch;
      }

      item.pipelineStage = 'enriched';
      
      metrics.counter('enrichment.success', 1);
      return item;
    } catch (err) {
      logger.error(`Enrichment failed for ${item.id}: ${err.message}`);
      metrics.counter('enrichment.errors', 1);
      // Continue with partially enriched item
      item.lastError = `Enrichment: ${err.message}`;
      item.pipelineStage = 'enriched';
      return item;
    }
  }

  // 1. Content Extraction
  private extractPlainText(html: string): string {
    // Use Mozilla Readability or similar for boilerplate removal
    const dom = new JSDOM(html);
    const reader = new Readability(dom.window.document);
    const article = reader.parse();
    
    if (article?.textContent) {
      return this.cleanText(article.textContent);
    }
    
    // Fallback: simple HTML stripping
    return this.cleanText(html.replace(/<[^>]+>/g, ' '));
  }

  private cleanText(text: string): string {
    return text
      .replace(/\s+/g, ' ')
      .replace(/\n+/g, '\n')
      .trim();
  }

  // 2. Author Extraction
  private async extractAuthor(item: FeedItem): Promise<string | null> {
    // Try metadata from collection
    if (item.metadata?.jsonLd?.author?.name) {
      return item.metadata.jsonLd.author.name;
    }

    // If we have full HTML content, parse for author
    if (item.content) {
      const dom = new JSDOM(item.content);
      const doc = dom.window.document;

      // Try common author selectors
      const selectors = [
        'meta[name="author"]',
        'meta[property="article:author"]',
        'meta[name="twitter:creator"]',
        '[rel="author"]',
        '.author',
        '.byline',
        '[class*="author"]',
        '[class*="byline"]',
      ];

      for (const selector of selectors) {
        const el = doc.querySelector(selector);
        if (el) {
          const author = el.getAttribute('content') 
                      ?? el.textContent?.trim() 
                      ?? null;
          if (author && author.length > 1 && author.length < 100) {
            return this.cleanAuthorName(author);
          }
        }
      }
    }

    // Try to extract from source metadata
    const source = await this.db('sources').where('id', item.sourceId).first();
    if (source?.default_author) {
      return source.default_author;
    }

    return null;
  }

  private cleanAuthorName(name: string): string {
    return name
      .replace(/^\s*by\s+/i, '')           // Remove leading "by"
      .replace(/^\s*written\s+by\s+/i, '') // Remove "written by"
      .replace(/\s+/g, ' ')
      .trim();
  }

  // 3. Date Normalisation
  private normaliseDate(dateInput: string): string {
    const date = new Date(dateInput);
    if (!isNaN(date.getTime())) {
      return date.toISOString();
    }
    // Try common formats
    const formats = [
      // RFC 2822: "Mon, 15 Aug 2025 14:30:00 GMT"
      /^\w{3},\s+\d{1,2}\s+\w{3}\s+\d{4}/,
      // ISO-like: "2025-08-15T14:30:00"
      /^\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}/,
      // Relative: "2 hours ago", "Yesterday"
      // (handled by date-fns parse)
    ];
    
    for (const fmt of formats) {
      if (fmt.test(dateInput)) {
        const parsed = new Date(dateInput);
        if (!isNaN(parsed.getTime())) return parsed.toISOString();
      }
    }
    
    // Fallback: current time (with warning)
    logger.warn(`Could not parse date: ${dateInput}, using current time`);
    return new Date().toISOString();
  }

  // 4. Image Extraction
  private async extractImage(item: FeedItem): Promise<string | null> {
    if (!item.content) return null;

    const dom = new JSDOM(item.content);
    const doc = dom.window.document;

    // Priority order for image extraction
    const imageSources = [
      // Open Graph
      doc.querySelector('meta[property="og:image"]')?.getAttribute('content'),
      // Twitter Card
      doc.querySelector('meta[name="twitter:image"]')?.getAttribute('content'),
      // First meaningful image in content
      doc.querySelector('img[src]:not([src*="icon"]):not([src*="logo"]):not([width="1"])')?.getAttribute('src'),
    ];

    for (const src of imageSources) {
      if (src && src.length > 10) {
        // Resolve relative URLs
        if (src.startsWith('/')) {
          try {
            const base = new URL(item.canonicalUrl);
            return base.origin + src;
          } catch { return src; }
        }
        if (!src.startsWith('http')) {
          try {
            return new URL(src, item.canonicalUrl).toString();
          } catch { return src; }
        }
        return src;
      }
    }

    return null;
  }

  // 7. Topic Keyword Matching
  private matchTopics(item: FeedItem): Topic[] {
    const text = `${item.title} ${item.summary ?? ''} ${item.contentText ?? ''}`.toLowerCase();
    const topics: Topic[] = [];

    const topicKeywords: Record<Topic, string[]> = {
      ai: [
        'artificial intelligence', 'machine learning', 'deep learning', 'neural network',
        'llm', 'large language model', 'gpt', 'chatgpt', 'claude', 'gemini',
        'openai', 'anthropic', 'hugging face', 'transformer', 'fine-tuning',
        'generative ai', 'agent', 'prompt engineering', 'mcp', 'vector database',
        'rag', 'retrieval augmented', 'diffusion model', 'stable diffusion',
        'computer vision', 'nlp', 'natural language', 'multimodal', 'foundation model',
        'ai model', 'inference', 'training run', 'parameter', 'token',
      ],
      gaming: [
        'video game', 'gameplay', 'gamer', 'gaming', 'playstation', 'xbox',
        'nintendo', 'switch', 'steam', 'epic games', 'unity', 'unreal engine',
        'speedrun', 'esports', 'tournament', 'multiplayer', 'mmorpg',
        'open world', 'fps', 'rpg', 'indie game', 'aaa game', 'dlc',
        'game pass', 'geforce now', 'ray tracing', 'vrr', 'dlss',
        'achievement', 'walkthrough', 'remaster', 'remake', 'sequel',
      ],
      tech: [
        'startup', 'funding', 'series a', 'series b', 'ipo', 'acquisition',
        'programming', 'developer', 'software engineer', 'open source',
        'cloud computing', 'aws', 'azure', 'gcp', 'kubernetes', 'docker',
        'linux', 'windows', 'macos', 'ios', 'android', 'react', 'vue',
        'security', 'vulnerability', 'breach', 'encryption', 'zero-day',
        'semiconductor', 'chip', 'cpu', 'gpu', 'tsmc', 'nvidia', 'amd',
        'smartphone', 'laptop', 'tablet', 'wearable', 'smart home',
        'cryptocurrency', 'bitcoin', 'ethereum', 'blockchain', 'web3',
        '5g', 'wifi', 'router', 'network', 'bandwidth', 'latency',
      ],
      sports: [
        'premier league', 'bundesliga', 'la liga', 'serie a', 'ligue 1',
        'champions league', 'world cup', 'euros', 'olympics', 'transfer',
        'formula 1', 'f1', 'nfl', 'nba', 'nhl', 'mlb', 'ufc', 'boxing',
        'grand slam', 'wimbledon', 'roland garros', 'us open', 'australian open',
        'tour de france', 'marathon', 'world championship', 'score',
        'match', 'goal', 'assist', 'red card', 'penalty', 'overtime',
      ],
      world: [
        'election', 'parliament', 'government', 'president', 'prime minister',
        'sanctions', 'trade war', 'diplomacy', 'summit', 'g7', 'g20',
        'conflict', 'war', 'ceasefire', 'peace talks', 'humanitarian',
        'climate change', 'cop', 'paris agreement', 'emission',
        'inflation', 'recession', 'interest rate', 'central bank',
        'migration', 'refugee', 'border', 'treaty', 'alliance',
        'protest', 'demonstration', 'revolution', 'coup',
      ],
      science: [
        'study finds', 'researchers', 'scientists', 'journal', 'peer-reviewed',
        'nasa', 'spacex', 'mars', 'moon landing', 'telescope', 'james webb',
        'crispr', 'gene editing', 'vaccine', 'clinical trial', 'breakthrough',
        'quantum', 'particle', 'black hole', 'exoplanet', 'supernova',
        'renewable energy', 'solar', 'wind power', 'battery', 'fusion',
      ],
      business: [
        'earnings', 'revenue', 'profit', 'quarterly', 'fiscal', 'guidance',
        'merger', 'acquisition', 'takeover', 'buyout', 'valuation',
        'stock', 'shares', 'dividend', 'market cap', 'trading',
        'layoff', 'hiring freeze', 'restructuring', 'bankruptcy',
        'regulation', 'antitrust', 'fine', 'lawsuit', 'settlement',
      ],
    };

    for (const [topic, keywords] of Object.entries(topicKeywords)) {
      const matches = keywords.filter(kw => text.includes(kw));
      const matchScore = matches.length / keywords.length;
      
      // Topic matched if at least 2 keywords found or 5% match rate
      if (matches.length >= 2 || matchScore > 0.05) {
        topics.push(topic as Topic);
        item.tags.push(...matches);
      }
    }

    // Deduplicate tags
    item.tags = [...new Set(item.tags)];

    return topics;
  }
}
```

---

### Stage 5: Scoring

#### Purpose
Calculate a composite quality score for each item by combining three dimensions: **source quality** (trustworthiness of origin), **content quality** (readability, completeness, depth), and **freshness** (time decay since publication). Assign a tier (1--5) based on the composite score.

#### Input: Enriched `FeedItem[]`

#### Output: Scored `FeedItem[]` (with `sourceQualityScore`, `contentQualityScore`, `compositeScore`, `tier`)

#### Scoring Formula

```
Composite Score = (sourceQ * 0.30) + (contentQ * 0.40) + (freshness * 0.20) + (engagement * 0.10)

Where:
  sourceQ      = Source quality score (0 -- 100) from source registry
  contentQ     = Content quality score (0 -- 100) computed from analysis
  freshness    = Freshness score (0 -- 100) with exponential time decay
  engagement   = Predicted engagement score (0 -- 100) from heuristics

Tier Assignment:
  Tier 1: Composite >= 85  (Breaking / Must-read)
  Tier 2: Composite >= 70  (High quality)
  Tier 3: Composite >= 55  (Good quality)
  Tier 4: Composite >= 40  (Average)
  Tier 5: Composite <  40  (Low priority / filler)
```

#### Scoring Engine

```typescript
class ScoringPipeline {
  // Time decay constants
  private readonly HALF_LIFE_HOURS = 24;     // Score halves after 24 hours
  private readonly MAX_AGE_HOURS = 168;      // Maximum age for scoring (1 week)
  private readonly BREAKING_WINDOW_MINUTES = 60; // Breaking news window

  async score(items: FeedItem[]): Promise<FeedItem[]> {
    return items.map(item => this.scoreItem(item));
  }

  private scoreItem(item: FeedItem): FeedItem {
    // Dimension 1: Source Quality (30%)
    const sourceQ = this.computeSourceQuality(item);

    // Dimension 2: Content Quality (40%)
    const contentQ = this.computeContentQuality(item);

    // Dimension 3: Freshness (20%)
    const freshness = this.computeFreshness(item);

    // Dimension 4: Engagement Potential (10%)
    const engagement = this.computeEngagementScore(item);

    // Composite score
    const composite = Math.round(
      sourceQ * 0.30 +
      contentQ * 0.40 +
      freshness * 0.20 +
      engagement * 0.10
    );

    // Apply override rules
    const finalScore = this.applyOverrideRules(item, composite);

    // Assign tier
    const tier = this.scoreToTier(finalScore);

    item.sourceQualityScore = sourceQ;
    item.contentQualityScore = contentQ;
    item.compositeScore = finalScore;
    item.tier = tier;
    item.pipelineStage = 'scored';

    // Store score components for transparency
    item.metadata = {
      ...item.metadata,
      scoreComponents: {
        sourceQuality: { score: sourceQ, weight: 0.30, contribution: sourceQ * 0.30 },
        contentQuality: { score: contentQ, weight: 0.40, contribution: contentQ * 0.40 },
        freshness: { score: freshness, weight: 0.20, contribution: freshness * 0.20 },
        engagement: { score: engagement, weight: 0.10, contribution: engagement * 0.10 },
      },
      overridesApplied: finalScore !== composite ? { original: composite, final: finalScore } : null,
    };

    return item;
  }

  // Source Quality Score (0 -- 100)
  private computeSourceQuality(item: FeedItem): number {
    // Base score from source registry
    let score = item.sourceQualityScore;

    // Adjustments
    if (item.author) score += 2;           // Known author is a positive signal
    if (item.metadata?.jsonLd) score += 3;  // Structured data indicates professionalism
    score = Math.min(100, score);

    return score;
  }

  // Content Quality Score (0 -- 100)
  private computeContentQuality(item: FeedItem): number {
    let score = 50; // Start at neutral

    // Content length scoring (optimal: 500 -- 3000 words)
    const wordCount = (item.contentText ?? item.title).split(/\s+/).length;
    if (wordCount > 100) {
      score += Math.min(20, (wordCount - 100) / 100); // +1 per 100 words, max +20
    } else {
      score -= 10; // Very short content penalty
    }
    if (wordCount > 5000) score -= 5; // Excessively long penalty

    // Content completeness
    const hasTitle = item.title && item.title.length > 10;
    const hasSummary = item.summary && item.summary.length > 20;
    const hasContent = item.contentText && item.contentText.length > 100;
    const hasAuthor = !!item.author;
    const hasImage = !!item.imageUrl;
    const hasDate = !!item.publishedAt;

    const completenessScore = [hasTitle, hasSummary, hasContent, hasAuthor, hasImage, hasDate]
      .filter(Boolean).length * 3; // +3 per field, max +18
    score += completenessScore;

    // Title quality
    if (item.title.length < 30) score -= 5; // Too short
    if (item.title.length > 150) score -= 3; // Too long
    if (/\?/.test(item.title)) score += 2; // Question titles engage

    // Content structure signals
    const text = item.contentText ?? '';
    const paragraphCount = text.split(/\n\n+/).length;
    if (paragraphCount >= 3) score += 5; // Well-structured content
    if (text.includes('"') || text.includes("'")) score += 3; // Contains quotes (interview/research)
    if (/\d+/.test(text)) score += 2; // Contains statistics/data

    // Language quality (simple heuristic)
    const sentenceCount = text.split(/[.!?]+/).filter(s => s.trim().length > 10).length;
    if (sentenceCount > 0) {
      const avgSentenceLength = wordCount / sentenceCount;
      if (avgSentenceLength >= 10 && avgSentenceLength <= 25) score += 5; // Good readability range
      if (avgSentenceLength > 40) score -= 5; // Hard to read
    }

    // Penalty for spam indicators
    const spamWords = ['click here', 'subscribe now', 'limited time', 'act now'];
    if (spamWords.some(w => text.toLowerCase().includes(w))) score -= 15;

    return Math.max(0, Math.min(100, score));
  }

  // Freshness Score (0 -- 100) with exponential decay
  private computeFreshness(item: FeedItem): number {
    const now = Date.now();
    const published = new Date(item.publishedAt).getTime();
    const ageHours = (now - published) / 3600000;

    // Breaking news bonus (first hour)
    if (ageHours <= 1) {
      return 100;
    }

    // Exponential decay: score = 100 * (0.5 ^ (age / half_life))
    const halfLives = ageHours / this.HALF_LIFE_HOURS;
    const score = 100 * Math.pow(0.5, halfLives);

    // Clamp and round
    return Math.max(0, Math.round(score));
  }

  // Engagement Score (0 -- 100) -- heuristic prediction
  private computeEngagementScore(item: FeedItem): number {
    let score = 50;

    // Title patterns that drive engagement
    const title = item.title.toLowerCase();
    if (title.includes('breaking')) score += 15;
    if (title.includes('exclusive')) score += 10;
    if (title.includes('announces')) score += 5;
    if (title.includes('vs') || title.includes('versus')) score += 8; // Comparison
    if (/\d+/.test(title)) score += 5; // Numbers in title
    if (title.includes('?')) score += 3; // Questions

    // Topic-specific engagement
    if (item.topics.includes('ai')) score += 5;
    if (item.topics.includes('gaming')) score += 3;

    // Content type
    if (item.contentType === 'video') score += 8;
    if (item.contentType === 'release') score += 5;

    return Math.min(100, score);
  }

  // Override Rules
  private applyOverrideRules(item: FeedItem, score: number): number {
    // Boost breaking news
    const ageHours = (Date.now() - new Date(item.publishedAt).getTime()) / 3600000;
    if (ageHours <= 2 && item.topics.some(t => ['ai', 'tech'].includes(t))) {
      score = Math.min(100, score + 10);
    }

    // Boost high-authority sources for important topics
    if (item.sourceQualityScore >= 90 && item.importance >= 80) {
      score = Math.min(100, score + 5);
    }

    // Penalty for known low-quality patterns
    if (item.title.toLowerCase().includes('sponsored')) {
      score = Math.max(0, score - 20);
    }

    return Math.max(0, Math.min(100, score));
  }

  // Score to Tier mapping
  private scoreToTier(score: number): 1 | 2 | 3 | 4 | 5 {
    if (score >= 85) return 1;
    if (score >= 70) return 2;
    if (score >= 55) return 3;
    if (score >= 40) return 4;
    return 5;
  }
}
```

#### Score Persistence

```typescript
class ScoreStore {
  constructor(private db: Knex) {}

  async saveScoreComponents(item: FeedItem): Promise<void> {
    await this.db('item_scores').insert({
      item_id: item.id,
      source_quality: item.sourceQualityScore,
      content_quality: item.contentQualityScore,
      freshness_score: item.freshnessScore,
      composite_score: item.compositeScore,
      tier: item.tier,
      components: JSON.stringify(item.metadata?.scoreComponents),
      computed_at: new Date().toISOString(),
    }).onConflict('item_id').merge();
  }

  // Recalculate freshness scores hourly (cron job)
  async recalculateFreshness(): Promise<number> {
    const items = await this.db('feed_items')
      .where('collected_at', '>', new Date(Date.now() - 7 * 86400000).toISOString())
      .select('*');

    let updated = 0;
    for (const item of items) {
      const newFreshness = this.computeFreshness(item);
      if (Math.abs(newFreshness - item.freshness_score) > 1) {
        await this.db('feed_items').where('id', item.id).update({
          freshness_score: newFreshness,
          composite_score: this.db.raw(
            'source_quality_score * 0.30 + content_quality_score * 0.40 + ? * 0.20 + 50 * 0.10',
            [newFreshness]
          ),
        });
        updated++;
      }
    }
    return updated;
  }
}
```

---

### Stage 6: Summarisation

#### Purpose
Generate or improve item summaries. If a source provides no summary or a poor-quality one, the pipeline generates one automatically using an extractive approach (first N sentences) or optionally an abstractive approach (local LLM).

#### Input: Scored `FeedItem[]`

#### Output: Summarised `FeedItem[]` (with `summary` field populated)

#### Summarisation Strategy Decision Tree

```
  Scored FeedItem
       |
       v
  Has summary?
       |
  +----+----+
  |         |
 Yes       No
  |         |
  v         v
Good     Generate
quality?   summary
  |         |
  v         v
Keep    Select method:
        |
  +-----+-----+
  |           |
Short      Long
content    content
(<500    (>500
 words)   words)
  |           |
  v           v
Extractive  Extractive +
(3 sents)   optional
            Abstractive
```

#### Summarisation Engine

```typescript
class SummarisationPipeline {
  private readonly MIN_SUMMARY_LENGTH = 50;
  private readonly MAX_DIGEST_LENGTH = 200;
  private readonly MAX_DETAIL_LENGTH = 500;
  private llmClient: LLMClient | null = null;

  constructor(config: { useLLM: boolean; llmEndpoint?: string }) {
    if (config.useLLM && config.llmEndpoint) {
      this.llmClient = new LLMClient(config.llmEndpoint);
    }
  }

  async summarise(items: FeedItem[]): Promise<FeedItem[]> {
    return Promise.all(items.map(item => this.summariseItem(item)));
  }

  private async summariseItem(item: FeedItem): Promise<FeedItem> {
    // Check if existing summary is good enough
    if (item.summary && this.isGoodSummary(item.summary)) {
      item.pipelineStage = 'summarised';
      return item;
    }

    const content = item.contentText ?? item.content ?? item.title;
    
    if (!content || content.trim().length < 50) {
      // Too little content to summarise -- use title as summary
      item.summary = item.title;
      item.pipelineStage = 'summarised';
      return item;
    }

    // Choose summarisation method based on content length and config
    const wordCount = content.split(/\s+/).length;
    
    if (wordCount < 200) {
      // Short content: just use the first couple of sentences
      item.summary = this.extractiveSummarise(content, 2);
    } else if (!this.llmClient || item.tier >= 4) {
      // No LLM available or low-tier item: extractive only
      item.summary = this.extractiveSummarise(content, 3);
    } else {
      // Use LLM for abstractive summary on high-tier items
      const extractive = this.extractiveSummarise(content, 3);
      try {
        const abstractive = await this.abstractiveSummarise(content, item.title);
        item.summary = abstractive || extractive;
        item.metadata = { ...item.metadata, summaryMethod: abstractive ? 'abstractive' : 'extractive' };
      } catch {
        item.summary = extractive; // Fallback
        item.metadata = { ...item.metadata, summaryMethod: 'extractive' };
      }
    }

    // Ensure length limits
    item.summary = this.enforceLengthLimit(item.summary, this.MAX_DETAIL_LENGTH);

    item.pipelineStage = 'summarised';
    metrics.counter('summarisation.completed', 1, { method: item.metadata?.summaryMethod as string || 'extractive' });
    return item;
  }

  private isGoodSummary(summary: string): boolean {
    if (!summary || summary.length < this.MIN_SUMMARY_LENGTH) return false;
    if (summary === 'No summary available') return false;
    if (summary.startsWith('http')) return false; // Some feeds put URLs as summaries
    return true;
  }

  // Extractive summarisation: select most important sentences
  private extractiveSummarise(text: string, sentenceCount: number): string {
    // Split into sentences
    const sentences = text
      .replace(/([.!?])\s+/g, '$1|')
      .split('|')
      .map(s => s.trim())
      .filter(s => s.length > 20 && s.length < 500);

    if (sentences.length <= sentenceCount) {
      return sentences.join(' ').slice(0, this.MAX_DETAIL_LENGTH);
    }

    // Score sentences by position and keyword density
    const scored = sentences.map((sentence, index) => {
      const positionScore = this.positionScore(index, sentences.length);
      const keywordScore = this.keywordScore(sentence, text);
      const lengthScore = this.lengthScore(sentence);
      
      return {
        sentence,
        score: positionScore * 0.4 + keywordScore * 0.4 + lengthScore * 0.2,
      };
    });

    // Take top sentences by score, but preserve original order
    const topSentences = scored
      .map((s, i) => ({ ...s, originalIndex: i }))
      .sort((a, b) => b.score - a.score)
      .slice(0, sentenceCount)
      .sort((a, b) => a.originalIndex - b.originalIndex);

    return topSentences.map(s => s.sentence).join(' ').slice(0, this.MAX_DETAIL_LENGTH);
  }

  private positionScore(index: number, total: number): number {
    // First and last sentences are most important
    if (index === 0) return 1.0;
    if (index === total - 1) return 0.8;
    return 1 - (index / total); // Decay by position
  }

  private keywordScore(sentence: string, fullText: string): number {
    // Sentences containing frequent words from the text score higher
    const words = sentence.toLowerCase().split(/\s+/).filter(w => w.length > 3);
    if (words.length === 0) return 0;
    
    const textWords = fullText.toLowerCase().split(/\s+/);
    const wordFreq = new Map<string, number>();
    for (const w of textWords) {
      wordFreq.set(w, (wordFreq.get(w) ?? 0) + 1);
    }
    
    const score = words.reduce((sum, w) => sum + (wordFreq.get(w) ?? 0), 0);
    return Math.min(1, score / (words.length * 5)); // Normalise
  }

  private lengthScore(sentence: string): number {
    // Optimal sentence length: 80 -- 200 characters
    const len = sentence.length;
    if (len >= 80 && len <= 200) return 1.0;
    if (len < 80) return len / 80;
    return Math.max(0, 1 - (len - 200) / 300);
  }

  // Abstractive summarisation via local LLM
  private async abstractiveSummarise(content: string, title: string): Promise<string | null> {
    if (!this.llmClient) return null;

    const prompt = `Summarise the following article in 2-3 sentences. Be concise and factual.

Title: ${title}

Content:
${content.slice(0, 4000)}

Summary:`;

    try {
      const response = await this.llmClient.complete({
        prompt,
        max_tokens: 150,
        temperature: 0.3, // Low temperature for factual output
      });

      const summary = response.trim();
      if (summary.length >= this.MIN_SUMMARY_LENGTH) {
        return summary;
      }
      return null;
    } catch (err) {
      logger.error(`LLM summarisation failed: ${err.message}`);
      return null;
    }
  }

  private enforceLengthLimit(text: string, maxLength: number): string {
    if (text.length <= maxLength) return text;
    
    // Try to cut at sentence boundary
    const truncated = text.slice(0, maxLength);
    const lastSentence = truncated.lastIndexOf('.');
    if (lastSentence > maxLength * 0.7) {
      return truncated.slice(0, lastSentence + 1);
    }
    return truncated + '...';
  }
}

// LLM Client for local model (e.g., Ollama, llama.cpp)
class LLMClient {
  constructor(private endpoint: string) {}

  async complete(params: { prompt: string; max_tokens: number; temperature: number }): Promise<string> {
    const response = await fetch(`${this.endpoint}/api/generate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'llama3.1:8b',
        prompt: params.prompt,
        stream: false,
        options: {
          temperature: params.temperature,
          num_predict: params.max_tokens,
        },
      }),
    });

    if (!response.ok) throw new Error(`LLM request failed: ${response.status}`);
    const data = await response.json();
    return data.response;
  }
}
```

---
