# jarvis-data-sourcer: Skill Overview & Agent Usage Interface

## Table of Contents

- **Section A: Skill Overview** — High-level description of what the skill does
  - A.1 Purpose Statement
  - A.2 Core Capabilities
  - A.3 How Agents Use It (command-call examples)
  - A.4 Integration Points (n8n, Telegram, 1Password, etc. — *installation-specific*)
  - A.5 Scope Boundaries (what the skill does NOT do)
  - A.6 Value Proposition
- **Section B: Agent Usage Interface** — Concrete TypeScript-typed command contract
  - Shared Types (`SourceType`, `SourceConfig`, `FeedItem`, `SourceHealth`)
  - Command 1: `discover_sources`
  - Command 2: `register_source`
  - Command 3: `create_topic_registry`
  - Command 4: `scrape_topic`
  - Command 5: `score_sources`
  - Command 6: `get_feed_items`
  - Command 7: `summarise_items`
  - Command 8: `detect_duplicates`
  - Command 9: `create_digest`
  - Error model & shared `SkillResponse<T>` envelope
  - Example: blocked-by-anti-bot response

> **Note on TypeScript surface**: The contract in Section B is expressed in
> TypeScript because the original Jarvis HQ deployment is Node.js. Treat it as
> ONE possible representation of the agent interface — agents in other
> runtimes (Python/Go/Rust/etc.) should produce semantically equivalent
> contracts, not literal TypeScript imports. See `architecture.md` §4 for the
> reference-stack rationale and `SKILL.md` for the model-agnostic guarantees.

---

## Section A: Skill Overview

### 1. Purpose Statement

`jarvis-data-sourcer` is a reusable **scraping intelligence layer** for the Jarvis HQ personal AI news ecosystem. It bridges raw public web content — from news sites, RSS feeds, Reddit, YouTube, GitHub, and developer blogs — and structured intelligence that agents can consume. It sits between upstream sources and downstream components: the Next.js news website, Telegram briefings, Google Calendar alerts, and n8n workflow triggers.

The skill automates the full pipeline from source discovery to formatted briefing: agents invoke a command, and the skill handles discovery, registration, scraping, extraction, deduplication, scoring, summarisation, and export. Built for Docker/WSL2 on Windows 11, it persists data to SQLite (or Postgres) and integrates via n8n webhooks.

### 2. Core Capabilities

- **Source discovery** — Find high-quality public sources for any topic, ranked with reliability metadata.
- **Registry management** — SQLite-backed registry storing source config, health status, and quality scores.
- **Adaptive scraping** — Per-site strategies: RSS/Atom, JSON API, HTML selectors, sitemap enumeration, with graceful fallback.
- **Scoring & deduplication** — Relevance, freshness, and reliability scoring with semantic near-duplicate detection.
- **Summarisation** — Bullet-point briefings for Telegram and news website cards.
- **Multi-format export** — JSON, Markdown, HTML, and Telegram-formatted output.
- **Health monitoring** — Per-source uptime tracking, consecutive failure alerts, and degradation reporting.
- **Topic filtering** — Include/exclude keyword filters per topic.
- **Review queue** — Flags low-confidence or filter-hit items for manual review.

### 3. How Agents Use It

Agents invoke named commands with typed parameters; the skill handles all execution. No scraping code, HTTP retries, or selector maintenance required.

```
Agent: discover_sources("AI", depth=2)
       → Returns ranked source list with metadata

Agent: register_source({ name: "OpenAI Blog", url: "...", type: "rss" })
       → Validates, tests parsing, writes to registry

Agent: scrape_topic("AI", since="24h", limit=20)
       → Scrapes all AI sources, deduplicates, scores, returns feed items

Agent: create_telegram_briefing("AI", format="bullets")
       → Returns Telegram-ready message text
```

### 4. Integration Points

| Component | Integration Method | Direction |
|---|---|---|
| **n8n** | Webhooks (HTTP POST/GET) | Bidirectional — n8n triggers scrape jobs; skill sends results back to n8n for routing to Telegram, Calendar, etc. |
| **Telegram Bot** | n8n webhook payload | Outbound — `create_telegram_briefing` returns text that n8n forwards to the Telegram Bot API. |
| **Next.js News Website** | SQLite read (via API route) or JSON export | Outbound — `export_feed` generates JSON that the website API consumes to render news cards. |
| **Google Calendar** | n8n workflow | Outbound — n8n receives `create_digest` output and creates calendar events for scheduled briefings. |
| **1Password** | Service Account credential retrieval | Inbound — skill reads API keys (e.g., for paid sources) via 1Password CLI, never hardcodes secrets. |
| **OneDrive** | Optional backup sync via n8n | Outbound — n8n can archive exported feeds to OneDrive. |
| **SQLite / Postgres** | Direct connection via ORM/knex | Internal — registry, feed items, scores, and health state all persist here. |

### 5. Scope Boundaries

The skill **does NOT** and **will NOT**:

- **Attempt to bypass paywalls.** Sources behind paywalls are marked
  `auth_required`. The skill only accesses them with credentials the operator
  has explicitly provided via a secrets manager (1Password in the reference
  deployment). No cookie/session manipulation. No archive-site indirection.
  No screenshot-OCR of paywalled pages.
- **Attempt to defeat anti-bot protections.** Cloudflare challenges, CAPTCHAs,
  and per-domain rate limits are respected. Failed fetches trigger exponential
  backoff; persistent blocks mark the source `blocked` and remove it from the
  active set. No residential proxies, no IP rotation, no User-Agent
  randomization, no browser-fingerprint masking.
- **Scrape private data.** Every source must be publicly accessible without
  authentication, or accessed via an official API with explicit operator
  consent. No DMs, no closed groups, no paid communities.
- **Run autonomously.** Executes only when invoked by agents or webhooks.
  Scheduling is delegated to the orchestrator (n8n in the reference
  deployment); the skill never self-schedules.
- **Store raw HTML permanently.** Raw content is transient (24h cache for
  re-extraction); only structured fields and excerpts (≤150 words) persist.

### 6. Value Proposition

Before this skill, maintaining a personal news site meant manually checking dozens of sources, copying links, writing summaries, and formatting posts — a repetitive process that falls behind quickly. `jarvis-data-sourcer` replaces that workflow with a single command interface: discover sources, scrape updates, deduplicate, score, summarise, and push to Telegram or the website in minutes instead of hours. It is topic-agnostic — the same pipeline serves AI news, gaming, UFC, or world news with no code changes. It prioritises free public sources for near-zero cost, with paid APIs as an optional upgrade.

---

## Section B: Agent Usage Interface

### Shared Types

```typescript
// ── Shared Types ──────────────────────────────────────────────

type SourceType = "rss" | "atom" | "json_api" | "html" | "sitemap" | "reddit" | "youtube" | "github";

type ScrapeStrategy = "feed" | "api" | "html_selectors" | "sitemap" | "reddit_api" | "youtube_api" | "github_api";

interface SourceConfig {
  name: string;                          // Human-readable name
  url: string;                           // Canonical source URL
  type: SourceType;                      // How the skill should treat this source
  strategy: ScrapeStrategy;              // Extraction strategy to use
  selectors?: Record<string, string>;    // CSS selectors for HTML strategy
  apiKeyRef?: string;                    // 1Password reference if API key needed
  headers?: Record<string, string>;      // Custom HTTP headers
  rateLimitMs?: number;                  // Min delay between requests (default: 1000)
  topicTags: string[];                   // Topics this source covers
  reliabilityScore?: number;             // 0.0–1.0, set by scoring (default: 0.5)
}

interface FeedItem {
  id: string;                            // UUID v4
  sourceId: string;                      // Reference to registered source
  sourceName: string;                    // Human-readable source name
  topic: string;                         // Topic tag
  title: string;
  url: string;                           // Canonical URL (dedup-normalised)
  summary?: string;                      // Auto-generated summary
  contentSnippet?: string;               // First N characters or excerpt
  publishedAt: string;                   // ISO 8601 datetime
  scrapedAt: string;                     // ISO 8601 datetime
  relevanceScore: number;                // 0.0–1.0
  duplicateOf?: string;                  // ID of canonical item if duplicate
  isFlagged: boolean;                    // True if needs manual review
  metadata?: Record<string, unknown>;    // Source-specific extras
}

interface SourceHealth {
  sourceId: string;
  sourceName: string;
  status: "healthy" | "degraded" | "failing" | "blocked" | "auth_required";
  lastSuccessAt?: string;                // ISO 8601
  lastFailureAt?: string;                // ISO 8601
  lastFailureReason?: string;
  consecutiveFailures: number;
  avgResponseTimeMs: number;
  successRate7d: number;                 // 0.0–1.0
}

interface ReviewQueueItem {
  itemId: string;
  sourceName: string;
  topic: string;
  title: string;
  url: string;
  flagReason: "low_confidence" | "content_filter" | "ambiguous_score" | "manual_rule";
  addedAt: string;                       // ISO 8601
}

type ErrorCode =
  | "SOURCE_NOT_FOUND"
  | "SOURCE_UNREACHABLE"
  | "PARSE_ERROR"
  | "RATE_LIMITED"
  | "TOPIC_NOT_FOUND"
  | "ITEM_NOT_FOUND"
  | "AUTH_REQUIRED"
  | "BLOCKED_BY_ANTI_BOT"
  | "INVALID_PARAMETERS"
  | "INTERNAL_ERROR";

interface SkillError {
  code: ErrorCode;
  message: string;
  sourceUrl?: string;
  retryable: boolean;
}

// ── Generic Response Wrapper ────────────────────────────────────

interface SkillResponse<T> {
  success: boolean;
  data?: T;
  error?: SkillError;
  meta?: {
    executedAt: string;                  // ISO 8601
    durationMs: number;
    queryParams?: Record<string, unknown>;
  };
}
```

---

### Command 1: `discover_sources`

Find high-quality, publicly accessible sources for a given topic.

```typescript
interface DiscoverSourcesInput {
  topic: string;                         // e.g. "AI", "gaming", "UFC"
  depth?: number;                        // Discovery depth: 1=basic, 2=deep (default: 1)
}

interface DiscoveredSource {
  name: string;
  url: string;
  type: SourceType;
  estimatedReliability: number;          // 0.0–1.0
  discoveryPath: string;                 // e.g. "reddit/r/artificial -> blogroll"
}

type DiscoverSourcesOutput = SkillResponse<{
  topic: string;
  sources: DiscoveredSource[];
  totalFound: number;
}>;
```

**Example invocation:**
```json
{
  "command": "discover_sources",
  "params": { "topic": "AI", "depth": 2 }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "topic": "AI",
    "sources": [
      {
        "name": "OpenAI Blog",
        "url": "https://openai.com/blog",
        "type": "rss",
        "estimatedReliability": 0.95,
        "discoveryPath": "direct_search"
      },
      {
        "name": "r/artificial",
        "url": "https://reddit.com/r/artificial.json",
        "type": "reddit",
        "estimatedReliability": 0.65,
        "discoveryPath": "reddit_search"
      }
    ],
    "totalFound": 12
  },
  "meta": { "executedAt": "2025-01-15T09:00:00Z", "durationMs": 3400 }
}
```

---

### Command 2: `register_source`

Add a source to the persistent registry. Validates reachability and parseability before committing.

```typescript
interface RegisterSourceInput {
  sourceConfig: SourceConfig;
  skipValidation?: boolean;              // If true, skip reachability test (default: false)
}

interface RegisterSourceOutputData {
  sourceId: string;                      // UUID assigned by skill
  sourceConfig: SourceConfig;
  validationResult: {
    reachable: boolean;
    parseable: boolean;
    sampleItemTitle?: string;            // Title of first item extracted during validation
    responseTimeMs: number;
  };
}

type RegisterSourceOutput = SkillResponse<RegisterSourceOutputData>;
```

**Example invocation:**
```json
{
  "command": "register_source",
  "params": {
    "sourceConfig": {
      "name": "OpenAI Blog",
      "url": "https://openai.com/blog/rss.xml",
      "type": "rss",
      "strategy": "feed",
      "topicTags": ["AI", "OpenAI"],
      "rateLimitMs": 2000
    }
  }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "sourceId": "src-uuid-001",
    "sourceConfig": { /* echoed */ },
    "validationResult": {
      "reachable": true,
      "parseable": true,
      "sampleItemTitle": "Announcing GPT-5",
      "responseTimeMs": 340
    }
  },
  "meta": { "executedAt": "2025-01-15T09:05:00Z", "durationMs": 890 }
}
```

---

### Command 3: `create_topic_registry`

Create a complete registry for a topic from scratch, optionally seeding it with a pre-built source list.

```typescript
interface CreateTopicRegistryInput {
  topic: string;
  sourceList?: SourceConfig[];           // Pre-defined sources to register; if omitted, auto-discovers
  autoDiscover?: boolean;                // Run discover_sources if sourceList omitted (default: true)
  autoDiscoverDepth?: number;            // Depth for auto-discovery (default: 1)
}

interface CreateTopicRegistryOutputData {
  topic: string;
  sourcesRegistered: number;
  sourcesFailed: number;
  sourceIds: string[];
  failures: Array<{ url: string; reason: string }>;
}

type CreateTopicRegistryOutput = SkillResponse<CreateTopicRegistryOutputData>;
```

**Example invocation:**
```json
{
  "command": "create_topic_registry",
  "params": {
    "topic": "pokemon",
    "autoDiscover": true,
    "autoDiscoverDepth": 1
  }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "topic": "pokemon",
    "sourcesRegistered": 6,
    "sourcesFailed": 1,
    "sourceIds": ["src-pkm-001", "src-pkm-002", "src-pkm-003", "src-pkm-004", "src-pkm-005", "src-pkm-006"],
    "failures": [
      { "url": "https://serebii.net/rss", "reason": "Feed format not recognised" }
    ]
  },
  "meta": { "executedAt": "2025-01-15T09:10:00Z", "durationMs": 12800 }
}
```

---

### Command 4: `scrape_topic`

Scrape the latest updates for all registered sources under a topic.

```typescript
interface ScrapeTopicInput {
  topic: string;
  since?: string;                        // ISO 8601 duration or absolute datetime, e.g. "24h" or "2025-01-01T00:00:00Z" (default: "24h")
  limit?: number;                        // Max items to return (default: 50)
  includeDuplicates?: boolean;           // If true, include items marked as duplicates (default: false)
}

interface ScrapeTopicOutputData {
  topic: string;
  itemsScraped: number;
  itemsNew: number;
  itemsDuplicate: number;
  itemsFlagged: number;
  sourcesTouched: number;
  sourcesFailed: number;
  feedItems: FeedItem[];                 // Sorted by publishedAt desc, capped at limit
}

type ScrapeTopicOutput = SkillResponse<ScrapeTopicOutputData>;
```

**Example invocation:**
```json
{
  "command": "scrape_topic",
  "params": { "topic": "AI", "since": "24h", "limit": 20 }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "topic": "AI",
    "itemsScraped": 34,
    "itemsNew": 18,
    "itemsDuplicate": 12,
    "itemsFlagged": 2,
    "sourcesTouched": 5,
    "sourcesFailed": 0,
    "feedItems": [
      {
        "id": "item-uuid-001",
        "sourceId": "src-uuid-001",
        "sourceName": "OpenAI Blog",
        "topic": "AI",
        "title": "Introducing o3: A New Era of Reasoning",
        "url": "https://openai.com/blog/o3",
        "summary": "OpenAI announces its o3 model series, claiming breakthroughs in mathematical reasoning and coding benchmarks...",
        "contentSnippet": "Today we are introducing o3 and o3-mini...",
        "publishedAt": "2025-01-14T14:00:00Z",
        "scrapedAt": "2025-01-15T09:00:00Z",
        "relevanceScore": 0.98,
        "duplicateOf": null,
        "isFlagged": false
      }
    ]
  },
  "meta": { "executedAt": "2025-01-15T09:00:00Z", "durationMs": 6700 }
}
```

---

### Command 5: `score_sources`

Score all registered sources for quality based on historical scrape data.

```typescript
interface ScoreSourcesInput {
  topic?: string;                        // If provided, score only sources for this topic; if omitted, score all
}

interface SourceScoreResult {
  sourceId: string;
  sourceName: string;
  previousScore: number;
  newScore: number;
  change: number;                        // Delta
  factors: {
    successRate: number;
    freshnessScore: number;              // How recent is the content
    contentQualityScore: number;         // Based on item length, engagement signals
    duplicateRate: number;               // Lower is better
  };
}

interface ScoreSourcesOutputData {
  sourcesScored: number;
  improvements: number;
  declines: number;
  results: SourceScoreResult[];
}

type ScoreSourcesOutput = SkillResponse<ScoreSourcesOutputData>;
```

**Example invocation:**
```json
{
  "command": "score_sources",
  "params": { "topic": "AI" }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "sourcesScored": 8,
    "improvements": 2,
    "declines": 1,
    "results": [
      {
        "sourceId": "src-uuid-001",
        "sourceName": "OpenAI Blog",
        "previousScore": 0.92,
        "newScore": 0.95,
        "change": 0.03,
        "factors": {
          "successRate": 1.0,
          "freshnessScore": 0.95,
          "contentQualityScore": 0.92,
          "duplicateRate": 0.02
        }
      }
    ]
  },
  "meta": { "executedAt": "2025-01-15T09:30:00Z", "durationMs": 1200 }
}
```

---

### Command 6: `get_feed_items`

Retrieve processed feed items from the database.

```typescript
interface GetFeedItemsInput {
  topic?: string;                        // Filter by topic; if omitted, all topics
  since?: string;                        // ISO 8601 duration or absolute datetime (default: "24h")
  limit?: number;                        // Max items to return (default: 50)
  minRelevance?: number;                 // Minimum relevance score filter (default: 0.0)
  includeSummaries?: boolean;            // Include summary field in response (default: true)
  sortBy?: "publishedAt" | "relevanceScore" | "scrapedAt"; // (default: "publishedAt")
}

type GetFeedItemsOutput = SkillResponse<{
  total: number;
  returned: number;
  feedItems: FeedItem[];
}>;
```

**Example invocation:**
```json
{
  "command": "get_feed_items",
  "params": { "topic": "AI", "since": "24h", "limit": 10, "minRelevance": 0.7 }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "total": 18,
    "returned": 10,
    "feedItems": [ /* FeedItem[] */ ]
  },
  "meta": { "executedAt": "2025-01-15T09:45:00Z", "durationMs": 45 }
}
```

---

### Command 7: `summarise_items`

Summarise one or more feed items into a coherent briefing.

```typescript
interface SummariseItemsInput {
  itemIds: string[];                     // UUIDs of items to summarise (1–20)
  style?: "neutral" | "analytical" | "casual"; // Writing tone (default: "neutral")
  maxLength?: number;                    // Max characters per summary (default: 500)
  language?: string;                     // e.g. "en", "zh" (default: "en")
}

interface SummariseItemsOutputData {
  summaries: Array<{
    itemId: string;
    title: string;
    summary: string;
  }>;
  combinedBriefing?: string;             // Single combined briefing if multiple items
}

type SummariseItemsOutput = SkillResponse<SummariseItemsOutputData>;
```

**Example invocation:**
```json
{
  "command": "summarise_items",
  "params": {
    "itemIds": ["item-uuid-001", "item-uuid-002"],
    "style": "analytical",
    "maxLength": 300
  }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "summaries": [
      {
        "itemId": "item-uuid-001",
        "title": "Introducing o3: A New Era of Reasoning",
        "summary": "OpenAI's o3 model achieves SOTA on ARC-AGI and math benchmarks, but inference costs are significantly higher than prior models. Key takeaway: reasoning capabilities are improving rapidly, though cost-efficiency remains a concern."
      }
    ],
    "combinedBriefing": "OpenAI pushes reasoning boundaries with o3, while Anthropic emphasises constitutional safety improvements in Claude 3.5. Both signal a competitive race in frontier model capabilities."
  },
  "meta": { "executedAt": "2025-01-15T09:50:00Z", "durationMs": 2100 }
}
```

---

### Command 8: `detect_duplicates`

Find duplicate or near-duplicate stories within a topic.

```typescript
interface DetectDuplicatesInput {
  topic?: string;                        // Topic to scan; if omitted, scans all topics
  since?: string;                        // Time window (default: "7d")
  similarityThreshold?: number;          // 0.0–1.0, semantic similarity floor (default: 0.85)
  autoResolve?: boolean;                 // If true, mark lower-scored item as duplicate (default: false)
}

interface DuplicateGroup {
  canonicalItemId: string;
  canonicalTitle: string;
  canonicalUrl: string;
  duplicates: Array<{
    itemId: string;
    title: string;
    url: string;
    similarityScore: number;
    sourceName: string;
  }>;
}

interface DetectDuplicatesOutputData {
  groupsFound: number;
  totalDuplicates: number;
  groups: DuplicateGroup[];
  autoResolved: number;                  // Count of items auto-marked as duplicate
}

type DetectDuplicatesOutput = SkillResponse<DetectDuplicatesOutputData>;
```

**Example invocation:**
```json
{
  "command": "detect_duplicates",
  "params": { "topic": "AI", "since": "24h", "similarityThreshold": 0.85, "autoResolve": true }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "groupsFound": 3,
    "totalDuplicates": 5,
    "autoResolved": 3,
    "groups": [
      {
        "canonicalItemId": "item-uuid-001",
        "canonicalTitle": "OpenAI Announces o3 Model",
        "canonicalUrl": "https://openai.com/blog/o3",
        "duplicates": [
          {
            "itemId": "item-uuid-007",
            "title": "OpenAI's o3 Breaks ARC-AGI Record",
            "url": "https://techcrunch.com/openai-o3",
            "similarityScore": 0.92,
            "sourceName": "TechCrunch"
          }
        ]
      }
    ]
  },
  "meta": { "executedAt": "2025-01-15T10:00:00Z", "durationMs": 3400 }
}
```

---

### Command 9: `create_digest`

Create a formatted briefing digest for a topic.

```typescript
interface CreateDigestInput {
  topic: string;
  format?: "markdown" | "html" | "json" | "text"; // (default: "markdown")
  period?: string;                       // e.g. "24h", "7d", "1h" (default: "24h")
  maxItems?: number;                     // Max stories in digest (default: 10)
  includeScores?: boolean;               // Include relevance scores (default: false)
  minRelevance?: number;                 // (default: 0.6)
}

interface CreateDigestOutputData {
  topic: string;
  period: string;
  generatedAt: string;                   // ISO 8601
  storiesIncluded: number;
  digest: string;                        // Formatted digest string (Markdown/HTML/Text)
}

type CreateDigestOutput = SkillResponse<CreateDigestOutputData>;
```

**Example invocation:**
```json
{
  "command": "create_digest",
  "params": { "topic": "AI", "format": "markdown", "period": "24h", "maxItems": 5 }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "topic": "AI",
    "period": "24h",
    "generatedAt": "2025-01-15T10:00:00Z",
    "storiesIncluded": 5,
    "digest": "# AI News Digest (24h)\n\n## 1. OpenAI Announces o3 Reasoning Model\nOpenAI's new o3 model achieves breakthrough results on ARC-AGI, pushing the frontier of AI reasoning. [Read more](https://openai.com/blog/o3)\n\n## 2. Anthropic Updates Claude 3.5 with Enhanced Safety..."
  },
  "meta": { "executedAt": "2025-01-15T10:00:00Z", "durationMs": 890 }
}
```

---

### Command 10: `create_telegram_briefing`

Create a Telegram-formatted briefing, optimised for the platform's message constraints.

```typescript
interface CreateTelegramBriefingInput {
  topic: string;
  format?: "bullets" | "numbered" | "cards"; // (default: "bullets")
  maxItems?: number;                     // (default: 5)
  period?: string;                       // (default: "24h")
  includeLinks?: boolean;                // (default: true)
  maxMessageLength?: number;             // Telegram limit is ~4096 (default: 4000)
}

interface CreateTelegramBriefingOutputData {
  topic: string;
  message: string;                       // Telegram-ready message text
  messageLength: number;
  itemsIncluded: number;
  truncated: boolean;                    // True if message hit length limit
}

type CreateTelegramBriefingOutput = SkillResponse<CreateTelegramBriefingOutputData>;
```

**Example invocation:**
```json
{
  "command": "create_telegram_briefing",
  "params": { "topic": "AI", "format": "bullets", "maxItems": 5 }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "topic": "AI",
    "message": "*AI Briefing — Last 24h*\n\n• *OpenAI o3 Model Announced* — SOTA on ARC-AGI benchmarks. New reasoning architecture shows promise but higher inference costs. [openai.com/blog/o3]\n\n• *Claude 3.5 Safety Update* — Anthropic introduces new constitutional safeguards. Reduced jailbreak success rate by 73%. [anthropic.com]\n\n——\n_5 stories | jarvis-data-sourcer_",
    "messageLength": 384,
    "itemsIncluded": 5,
    "truncated": false
  },
  "meta": { "executedAt": "2025-01-15T10:05:00Z", "durationMs": 650 }
}
```

---

### Command 11: `validate_source`

Test if a source URL is reachable and parseable by the skill.

```typescript
interface ValidateSourceInput {
  url: string;
  type?: SourceType;                     // Hint for parsing strategy; if omitted, auto-detects
  timeoutMs?: number;                    // Request timeout (default: 10000)
}

interface ValidateSourceOutputData {
  url: string;
  reachable: boolean;
  statusCode?: number;
  detectedType?: SourceType;
  parseable: boolean;
  sampleItem?: {
    title?: string;
    url?: string;
    publishedAt?: string;
  };
  responseTimeMs: number;
  errors?: string[];
}

type ValidateSourceOutput = SkillResponse<ValidateSourceOutputData>;
```

**Example invocation:**
```json
{
  "command": "validate_source",
  "params": { "url": "https://openai.com/blog/rss.xml", "type": "rss" }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "url": "https://openai.com/blog/rss.xml",
    "reachable": true,
    "statusCode": 200,
    "detectedType": "rss",
    "parseable": true,
    "sampleItem": {
      "title": "Introducing o3: A New Era of Reasoning",
      "url": "https://openai.com/blog/o3",
      "publishedAt": "2025-01-14T14:00:00Z"
    },
    "responseTimeMs": 340
  },
  "meta": { "executedAt": "2025-01-15T10:10:00Z", "durationMs": 340 }
}
```

---

### Command 12: `get_source_health`

Check the health status of all sources for a topic (or globally).

```typescript
interface GetSourceHealthInput {
  topic?: string;                        // Filter by topic; if omitted, returns all sources
  minStatus?: "healthy" | "degraded" | "failing" | "blocked"; // Filter by minimum status severity
}

type GetSourceHealthOutput = SkillResponse<{
  sourcesChecked: number;
  healthy: number;
  degraded: number;
  failing: number;
  blocked: number;
  authRequired: number;
  sources: SourceHealth[];
}>;
```

**Example invocation:**
```json
{
  "command": "get_source_health",
  "params": { "topic": "AI" }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "sourcesChecked": 8,
    "healthy": 6,
    "degraded": 1,
    "failing": 0,
    "blocked": 1,
    "authRequired": 0,
    "sources": [
      {
        "sourceId": "src-uuid-001",
        "sourceName": "OpenAI Blog",
        "status": "healthy",
        "lastSuccessAt": "2025-01-15T09:00:00Z",
        "consecutiveFailures": 0,
        "avgResponseTimeMs": 340,
        "successRate7d": 1.0
      },
      {
        "sourceId": "src-uuid-005",
        "sourceName": "ArXiv AI RSS",
        "status": "degraded",
        "lastSuccessAt": "2025-01-14T08:00:00Z",
        "lastFailureAt": "2025-01-15T09:00:00Z",
        "lastFailureReason": "504 Gateway Timeout",
        "consecutiveFailures": 2,
        "avgResponseTimeMs": 8200,
        "successRate7d": 0.78
      }
    ]
  },
  "meta": { "executedAt": "2025-01-15T10:15:00Z", "durationMs": 120 }
}
```

---

### Command 13: `export_feed`

Export processed feed items in a format suitable for the Next.js news website or other consumers.

```typescript
interface ExportFeedInput {
  format?: "json" | "ndjson" | "csv" | "rss"; // (default: "json")
  topic?: string;                        // Filter by topic; if omitted, all topics
  since?: string;                        // (default: "24h")
  limit?: number;                        // (default: 100)
  minRelevance?: number;                 // (default: 0.5)
  includeSummaries?: boolean;            // (default: true)
  outputPath?: string;                   // File path to write to; if omitted, returns inline
}

interface ExportFeedOutputData {
  format: string;
  recordsExported: number;
  filePath?: string;                     // Only present if outputPath was specified
  inlineData?: string;                   // Only present if outputPath omitted
  topics: string[];
  generatedAt: string;
}

type ExportFeedOutput = SkillResponse<ExportFeedOutputData>;
```

**Example invocation:**
```json
{
  "command": "export_feed",
  "params": { "format": "json", "topic": "AI", "since": "24h", "minRelevance": 0.7 }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "format": "json",
    "recordsExported": 18,
    "inlineData": "[{...}, {...}]",
    "topics": ["AI"],
    "generatedAt": "2025-01-15T10:20:00Z"
  },
  "meta": { "executedAt": "2025-01-15T10:20:00Z", "durationMs": 230 }
}
```

---

### Command 14: `add_topic_filter`

Add content include/exclude keyword filters for a topic.

```typescript
interface AddTopicFilterInput {
  topic: string;
  include?: string[];                    // Keywords/phrases that must be present for inclusion
  exclude?: string[];                    // Keywords/phrases that exclude an item if present
  caseSensitive?: boolean;               // (default: false)
  matchMode?: "any" | "all";             // For include filters: match any or all (default: "any")
}

interface AddTopicFilterOutputData {
  topic: string;
  filters: {
    include: string[];
    exclude: string[];
    caseSensitive: boolean;
    matchMode: "any" | "all";
  };
  affectedExistingItems: number;         // Number of existing items newly filtered
}

type AddTopicFilterOutput = SkillResponse<AddTopicFilterOutputData>;
```

**Example invocation:**
```json
{
  "command": "add_topic_filter",
  "params": {
    "topic": "gaming",
    "include": ["Xbox", "Game Pass", "Starfield", "Halo"],
    "exclude": ["mobile", "gacha"]
  }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "topic": "gaming",
    "filters": {
      "include": ["Xbox", "Game Pass", "Starfield", "Halo"],
      "exclude": ["mobile", "gacha"],
      "caseSensitive": false,
      "matchMode": "any"
    },
    "affectedExistingItems": 3
  },
  "meta": { "executedAt": "2025-01-15T10:25:00Z", "durationMs": 180 }
}
```

---

### Command 15: `review_queue`

Show items flagged for manual review.

```typescript
interface ReviewQueueInput {
  topic?: string;                        // Filter by topic; if omitted, all topics
  flagReason?: "low_confidence" | "content_filter" | "ambiguous_score" | "manual_rule"; // Filter by reason
  limit?: number;                        // (default: 50)
  offset?: number;                       // For pagination (default: 0)
}

interface ReviewQueueOutputData {
  total: number;
  returned: number;
  offset: number;
  items: ReviewQueueItem[];
}

type ReviewQueueOutput = SkillResponse<ReviewQueueOutputData>;
```

**Example invocation:**
```json
{
  "command": "review_queue",
  "params": { "topic": "AI", "limit": 10 }
}
```

**Example response:**
```json
{
  "success": true,
  "data": {
    "total": 3,
    "returned": 3,
    "offset": 0,
    "items": [
      {
        "itemId": "item-uuid-042",
        "sourceName": "Unknown Blog",
        "topic": "AI",
        "title": "Revolutionary AI Chip Design (Sponsored)",
        "url": "https://sponsored-blog.example/ai-chip",
        "flagReason": "low_confidence",
        "addedAt": "2025-01-15T09:30:00Z"
      },
      {
        "itemId": "item-uuid-043",
        "sourceName": "r/artificial",
        "topic": "AI",
        "title": "Click here for free AI courses!!!",
        "url": "https://spam.example",
        "flagReason": "content_filter",
        "addedAt": "2025-01-15T09:45:00Z"
      }
    ]
  },
  "meta": { "executedAt": "2025-01-15T10:30:00Z", "durationMs": 90 }
}
```

---

## Appendix: Error Response Examples

### Example: Source Unreachable
```json
{
  "success": false,
  "error": {
    "code": "SOURCE_UNREACHABLE",
    "message": "Source returned HTTP 502 after 3 retries",
    "sourceUrl": "https://example.com/feed.xml",
    "retryable": true
  },
  "meta": { "executedAt": "2025-01-15T10:00:00Z", "durationMs": 15400 }
}
```

### Example: Rate Limited
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Reddit API rate limit exceeded. Retry after 3600s.",
    "retryable": true
  },
  "meta": { "executedAt": "2025-01-15T10:05:00Z", "durationMs": 1200 }
}
```

### Example: Topic Not Found
```json
{
  "success": false,
  "error": {
    "code": "TOPIC_NOT_FOUND",
    "message": "No sources registered for topic 'blockchain'. Use create_topic_registry or register_source first.",
    "retryable": false
  },
  "meta": { "executedAt": "2025-01-15T10:10:00Z", "durationMs": 15 }
}
```

### Example: Blocked by Anti-Bot
```json
{
  "success": false,
  "error": {
    "code": "BLOCKED_BY_ANTI_BOT",
    "message": "Cloudflare challenge detected. Source has been marked as 'blocked' in registry.",
    "sourceUrl": "https://protected-site.com/news",
    "retryable": false
  },
  "meta": { "executedAt": "2025-01-15T10:15:00Z", "durationMs": 8200 }
}
```

---

## Appendix: Command Quick Reference

| # | Command | Key Input | Key Output |
|---|---|---|---|
| 1 | `discover_sources` | `topic`, `depth?` | `DiscoveredSource[]` |
| 2 | `register_source` | `sourceConfig` | `sourceId`, validation result |
| 3 | `create_topic_registry` | `topic`, `sourceList?` | Registration summary |
| 4 | `scrape_topic` | `topic`, `since?`, `limit?` | `FeedItem[]` with stats |
| 5 | `score_sources` | `topic?` | `SourceScoreResult[]` |
| 6 | `get_feed_items` | `topic?`, `since?`, `limit?` | `FeedItem[]` |
| 7 | `summarise_items` | `itemIds`, `style?` | Summaries + combined briefing |
| 8 | `detect_duplicates` | `topic?`, `similarityThreshold?` | `DuplicateGroup[]` |
| 9 | `create_digest` | `topic`, `format?`, `period?` | Formatted digest string |
| 10 | `create_telegram_briefing` | `topic`, `format?` | Telegram-ready message |
| 11 | `validate_source` | `url`, `type?` | Reachability + parseability report |
| 12 | `get_source_health` | `topic?` | `SourceHealth[]` |
| 13 | `export_feed` | `format?`, `topic?`, `outputPath?` | File or inline data |
| 14 | `add_topic_filter` | `topic`, `include?`, `exclude?` | Filter config + affected count |
| 15 | `review_queue` | `topic?`, `flagReason?` | `ReviewQueueItem[]` |
