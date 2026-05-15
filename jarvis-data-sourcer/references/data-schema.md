# Section E: Data Schema


## Table of Contents

- Overview
- 1. Source Schema
- 2. FeedItem Schema
- 3. ScrapeJob Schema
- 4. Topic Schema
- 5. QualityScore Schema
- 6. DigestEntry Schema
- 7. Database Schema (SQL)
- 8. PostgreSQL-Specific Enhancements
- 9. Score Tier Mapping
- 10. Status State Machines
  - Source Status Transitions
  - FeedItem Status Transitions
  - ScrapeJob Status Transitions
  - Digest Status Transitions
- 11. Data Retention & Archival
- 12. ER Diagram (Text)

---


## Overview

Jarvis HQ uses a relational core (SQLite/Postgres) to store all metadata, quality scores, job logs, and digest entries. Extracted full-text and raw HTML are stored on disk and referenced by path to keep the database lean.

**Design Principles**
- UUIDv7 primary keys everywhere (sortable, collision-free across nodes)
- All timestamps stored as `TIMESTAMPTZ` (Postgres) / `TEXT` ISO-8601 (SQLite)
- Scores stored as `DECIMAL(5,2)` (Postgres) / `REAL` (SQLite) with explicit ranges
- JSON columns use `JSONB` (Postgres) / `TEXT` (SQLite) — no migrations needed for config changes
- Full-text search via Postgres `tsvector` columns OR SQLite `FTS5` virtual tables (recommended for large corpuses)
- File references (`full_text_path`, `raw_html_path`) point to disk paths rather than BLOB storage

---

## 1. Source Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Source",
  "description": "A registered data source for a topic in Jarvis HQ",
  "type": "object",
  "required": ["id", "name", "url", "type", "topic", "status"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier (UUIDv7)",
      "example": "018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12"
    },
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 255,
      "description": "Human-readable name for the source",
      "example": "Anthropic Research Blog"
    },
    "url": {
      "type": "string",
      "format": "uri",
      "maxLength": 2048,
      "description": "Primary URL of the source",
      "example": "https://www.anthropic.com/research"
    },
    "feed_url": {
      "type": ["string", "null"],
      "format": "uri",
      "maxLength": 2048,
      "description": "RSS/Atom feed URL, if different from primary URL",
      "example": "https://www.anthropic.com/research/rss.xml"
    },
    "type": {
      "type": "string",
      "enum": [
        "blog", "rss_feed", "api", "github", "youtube",
        "newsletter", "news_site", "forum",
        "docs", "changelog", "press_room", "sitemap", "search"
      ],
      "description": "Classification of the source type. Note: 'twitter' / 'x' is intentionally absent — X HTML scraping violates ToS and the official paid API is treated as a special-case 'api' type, not a primary source category.",
      "example": "blog"
    },
    "topic": {
      "type": "string",
      "description": "Foreign key to topics.slug — the primary topic this source belongs to",
      "example": "ai"
    },
    "status": {
      "type": "string",
      "enum": ["active", "paused", "error", "pending_verification", "rejected"],
      "default": "pending_verification",
      "description": "Current operational status of the source",
      "example": "active"
    },
    "priority": {
      "type": "integer",
      "minimum": 1,
      "maximum": 10,
      "default": 5,
      "description": "Scraping priority (1 = lowest, 10 = highest)",
      "example": 8
    },
    "schedule": {
      "type": ["string", "null"],
      "description": "Cron expression for scheduled scraping. Overrides topic-level schedule when set.",
      "example": "*/15 * * * *"
    },
    "adapter": {
      "type": ["string", "null"],
      "description": "Name of the adapter module that handles this source",
      "example": "anthropic_blog_adapter"
    },
    "config": {
      "type": ["object", "null"],
      "description": "Adapter-specific configuration as free-form JSON",
      "example": {
        "article_selector": "article.post",
        "date_format": "%B %d, %Y",
        "pagination": { "enabled": true, "max_pages": 5 }
      }
    },
    "quality_score": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "description": "Computed quality score for this source (0-100)",
      "example": 92.5
    },
    "quality_tier": {
      "type": ["integer", "null"],
      "minimum": 1,
      "maximum": 5,
      "description": "Quality tier derived from quality_score (1 = highest)",
      "example": 1
    },
    "last_scraped": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "Timestamp of the last scraping attempt (ISO 8601)",
      "example": "2024-05-15T14:30:00Z"
    },
    "last_success": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "Timestamp of the last successful scrape (ISO 8601)",
      "example": "2024-05-15T14:30:00Z"
    },
    "last_error": {
      "type": ["string", "null"],
      "maxLength": 2000,
      "description": "Error message from the most recent failed scrape",
      "example": "HTTP 403: Forbidden — rate limited after 120 requests"
    },
    "error_count": {
      "type": "integer",
      "minimum": 0,
      "default": 0,
      "description": "Consecutive error count; auto-pauses source at threshold",
      "example": 3
    },
    "success_count": {
      "type": "integer",
      "minimum": 0,
      "default": 0,
      "description": "Total successful scrapes (lifetime counter)",
      "example": 847
    },
    "rate_limit": {
      "type": ["integer", "null"],
      "minimum": 1,
      "maximum": 10000,
      "description": "Maximum requests per minute allowed for this source",
      "example": 30
    },
    "robots_txt_allowed": {
      "type": ["boolean", "null"],
      "description": "Whether robots.txt permits scraping (null = not checked yet)",
      "example": true
    },
    "auth_required": {
      "type": "boolean",
      "default": false,
      "description": "Whether authentication is needed to access this source",
      "example": true
    },
    "auth_type": {
      "type": ["string", "null"],
      "enum": ["none", "api_key", "oauth", "bearer", "basic"],
      "default": "none",
      "description": "Authentication method required",
      "example": "bearer"
    },
    "auth_config": {
      "type": ["object", "null"],
      "description": "Authentication configuration. Stores env var references or encrypted tokens. Never store plaintext secrets.",
      "example": {
        "env_var": "ANTHROPIC_API_KEY",
        "header_name": "Authorization"
      }
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record creation timestamp (ISO 8601)",
      "example": "2024-01-15T09:00:00Z"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record last update timestamp (ISO 8601)",
      "example": "2024-05-15T14:30:00Z"
    },
    "notes": {
      "type": ["string", "null"],
      "maxLength": 5000,
      "description": "Free-form notes for human operators",
      "example": "Site has aggressive anti-bot. Reduce crawl rate; if still blocked, mark `blocked` and remove from active set. Do not use residential proxies or IP rotation."
    },
    "tags": {
      "type": ["array", "null"],
      "items": { "type": "string", "maxLength": 50 },
      "description": "Arbitrary tags for grouping and filtering",
      "example": ["frontier-model", "safety-research", "english"]
    }
  }
}
```

**Example Record:**
```json
{
  "id": "018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12",
  "name": "Anthropic Research Blog",
  "url": "https://www.anthropic.com/research",
  "feed_url": "https://www.anthropic.com/research/rss.xml",
  "type": "blog",
  "topic": "ai",
  "status": "active",
  "priority": 9,
  "schedule": "*/15 * * * *",
  "adapter": "anthropic_blog_adapter",
  "config": {
    "article_selector": "article.post",
    "date_format": "%B %d, %Y",
    "extraction": "readability",
    "follow_pagination": false
  },
  "quality_score": 94.0,
  "quality_tier": 1,
  "last_scraped": "2024-05-15T14:30:00Z",
  "last_success": "2024-05-15T14:30:00Z",
  "last_error": null,
  "error_count": 0,
  "success_count": 1247,
  "rate_limit": 30,
  "robots_txt_allowed": true,
  "auth_required": false,
  "auth_type": "none",
  "auth_config": null,
  "created_at": "2024-01-15T09:00:00Z",
  "updated_at": "2024-05-15T14:30:00Z",
  "notes": "Primary source for Anthropic research updates. High signal-to-noise.",
  "tags": ["frontier-model", "safety-research", "english", "us"]
}
```

---

## 2. FeedItem Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "FeedItem",
  "description": "A single scraped content item — the core unit of Jarvis HQ",
  "type": "object",
  "required": ["id", "source_id", "topic", "title", "url", "content_type", "status"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier (UUIDv7)",
      "example": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23"
    },
    "source_id": {
      "type": "string",
      "format": "uuid",
      "description": "Foreign key to sources.id",
      "example": "018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12"
    },
    "topic": {
      "type": "string",
      "description": "Topic slug — denormalized for fast filtering",
      "example": "ai"
    },
    "title": {
      "type": "string",
      "maxLength": 500,
      "description": "Article or item title",
      "example": "Claude 3 Opus: Advancing the Frontier of AI Capability"
    },
    "summary": {
      "type": ["string", "null"],
      "maxLength": 10000,
      "description": "Auto-generated or extracted summary of the content",
      "example": "Anthropic introduces Claude 3 Opus, a new frontier model demonstrating significant improvements in reasoning, coding, and multi-step task completion."
    },
    "full_text": {
      "type": ["string", "null"],
      "description": "Extracted full article text. Stored on disk; this field contains the file path reference.",
      "example": "/data/fulltext/2024/05/15/018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23.txt"
    },
    "url": {
      "type": "string",
      "format": "uri",
      "maxLength": 2048,
      "description": "Original URL of the item",
      "example": "https://www.anthropic.com/research/claude-3-opus"
    },
    "canonical_url": {
      "type": ["string", "null"],
      "format": "uri",
      "maxLength": 2048,
      "description": "Resolved canonical URL after redirects and meta tags",
      "example": "https://www.anthropic.com/research/claude-3-opus"
    },
    "author": {
      "type": ["string", "null"],
      "maxLength": 255,
      "description": "Author name or handle",
      "example": "Anthropic Research Team"
    },
    "author_url": {
      "type": ["string", "null"],
      "format": "uri",
      "maxLength": 2048,
      "description": "URL to author profile, if available",
      "example": "https://www.anthropic.com/team"
    },
    "published_date": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "Original publication date from the source (ISO 8601)",
      "example": "2024-03-04T10:00:00Z"
    },
    "discovered_date": {
      "type": "string",
      "format": "date-time",
      "description": "When this item was first scraped (ISO 8601)",
      "example": "2024-05-15T14:30:00Z"
    },
    "updated_date": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "Last updated date, if the source provides one (ISO 8601)",
      "example": "2024-03-05T16:20:00Z"
    },
    "language": {
      "type": ["string", "null"],
      "minLength": 2,
      "maxLength": 10,
      "description": "ISO 639-1 language code",
      "example": "en"
    },
    "region": {
      "type": ["string", "null"],
      "maxLength": 10,
      "description": "Geographic region code (ISO 3166-1 alpha-2 or custom)",
      "example": "US"
    },
    "tags": {
      "type": ["array", "null"],
      "items": { "type": "string", "maxLength": 50 },
      "description": "Extracted or assigned tags",
      "example": ["claude", "llm", "frontier-model", "multimodal"]
    },
    "content_type": {
      "type": "string",
      "enum": [
        "article", "release", "video", "tweet", "changelog",
        "paper", "podcast", "documentation", "press_release",
        "blog_post", "forum_post", "commentary", "tutorial",
        "review", "announcement", "research", "news", "unknown"
      ],
      "description": "Classification of the content type",
      "example": "announcement"
    },
    "media_urls": {
      "type": ["array", "null"],
      "items": {
        "type": "object",
        "properties": {
          "url": { "type": "string", "format": "uri" },
          "type": { "type": "string", "enum": ["image", "video", "audio", "embed"] },
          "alt": { "type": ["string", "null"] },
          "width": { "type": ["integer", "null"] },
          "height": { "type": ["integer", "null"] }
        }
      },
      "description": "Attached media assets (images, videos, etc.)",
      "example": [
        {
          "url": "https://cdn.anthropic.com/claude-3-opus-hero.png",
          "type": "image",
          "alt": "Claude 3 Opus benchmark results",
          "width": 1200,
          "height": 630
        }
      ]
    },
    "quality_score": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "description": "Overall computed quality score (0-100)",
      "example": 91.75
    },
    "quality_tier": {
      "type": ["integer", "null"],
      "minimum": 1,
      "maximum": 5,
      "description": "Quality tier derived from quality_score (1 = best)",
      "example": 1
    },
    "freshness_score": {
      "type": ["number", "null"], 
      "minimum": 0,
      "maximum": 100,
      "description": "Freshness score — decays with age (0-100)",
      "example": 88.5
    },
    "source_quality_score": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "description": "Inherited quality score from the source at scrape time",
      "example": 94.0
    },
    "confidence_score": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "description": "Confidence in extraction accuracy (0-100)",
      "example": 96.0
    },
    "sentiment_score": {
      "type": ["number", "null"],
      "minimum": -1,
      "maximum": 1,
      "description": "Sentiment polarity: -1 (negative) to +1 (positive). Optional.",
      "example": 0.35
    },
    "importance_score": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "description": "Estimated importance/relevance for the topic (0-100)",
      "example": 92.0
    },
    "duplicate_group_id": {
      "type": ["string", "null"],
      "format": "uuid",
      "description": "UUID of the duplicate group this item belongs to, if any",
      "example": "018f5a4c-bd3e-9c5f-b061-4e8f0c2d6a34"
    },
    "is_primary": {
      "type": ["boolean", "null"],
      "description": "True if this is the original/canonical source of the content",
      "example": true
    },
    "related_item_ids": {
      "type": ["array", "null"],
      "items": { "type": "string", "format": "uuid" },
      "description": "UUIDs of semantically related feed items",
      "example": ["018f5a3d-ac4f-7d60-c172-5f9a1d3e7b45"]
    },
    "extraction_method": {
      "type": ["string", "null"],
      "enum": [
        "rss", "api", "html_scrape", "html_readability",
        "youtube_api", "github_api",
        "newsletter_parse", "pdf_extract", "manual_entry"
      ],
      "description": "Method used to extract this item. Note: 'twitter_api' is intentionally absent — X scraping is excluded by skill safety rules; if an operator has paid X API access they should record it as method='api' with extraction_method='api' and document the official endpoint in raw_metadata.",
      "example": "rss"
    },
    "raw_metadata": {
      "type": ["object", "null"],
      "description": "Original raw data from the source as JSON (for debugging/re-scoring)",
      "example": {
        "rss_guid": "https://www.anthropic.com/research/claude-3-opus",
        "rss_categories": ["Research", "Claude"],
        "rss_comments": null,
        "rss_enclosure": null
      }
    },
    "agent_notes": {
      "type": ["string", "null"],
      "maxLength": 5000,
      "description": "Notes added by the scraping agent or review pipeline",
      "example": "High-priority item — frontier model announcement. Flagged for immediate digest inclusion."
    },
    "license_notes": {
      "type": ["string", "null"],
      "maxLength": 500,
      "description": "License or terms-of-use metadata extracted from or asserted about the source content. Used to preserve provenance per the skill's hard safety rules. Free-form short string (e.g. SPDX-like identifier, license name, or 'all rights reserved').",
      "example": "all rights reserved; excerpt fair use (<150 words)"
    },
    "status": {
      "type": "string",
      "enum": ["new", "processed", "reviewed", "rejected", "published", "archived"],
      "default": "new",
      "description": "Processing lifecycle status",
      "example": "published"
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record creation timestamp (ISO 8601)",
      "example": "2024-05-15T14:30:00Z"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record last update timestamp (ISO 8601)",
      "example": "2024-05-15T15:00:00Z"
    }
  }
}
```

**Example Record:**
```json
{
  "id": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23",
  "source_id": "018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12",
  "topic": "ai",
  "title": "Claude 3 Opus: Advancing the Frontier of AI Capability",
  "summary": "Anthropic introduces Claude 3 Opus, demonstrating state-of-the-art performance on reasoning benchmarks, coding tasks, and multi-step problem solving. The model shows particular strength in graduate-level reasoning and mathematics.",
  "full_text": "/data/fulltext/2024/05/15/018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23.txt",
  "url": "https://www.anthropic.com/research/claude-3-opus",
  "canonical_url": "https://www.anthropic.com/research/claude-3-opus",
  "author": "Anthropic Research Team",
  "author_url": null,
  "published_date": "2024-03-04T10:00:00Z",
  "discovered_date": "2024-05-15T14:30:00Z",
  "updated_date": null,
  "language": "en",
  "region": "US",
  "tags": ["claude", "llm", "frontier-model", "multimodal", "benchmarks"],
  "content_type": "announcement",
  "media_urls": [
    {
      "url": "https://cdn.anthropic.com/claude-3-opus-hero.png",
      "type": "image",
      "alt": "Claude 3 Opus benchmark results",
      "width": 1200,
      "height": 630
    }
  ],
  "quality_score": 91.75,
  "quality_tier": 1,
  "freshness_score": 88.5,
  "source_quality_score": 94.0,
  "confidence_score": 96.0,
  "sentiment_score": 0.42,
  "importance_score": 92.0,
  "duplicate_group_id": null,
  "is_primary": true,
  "related_item_ids": ["018f5a3d-ac4f-7d60-c172-5f9a1d3e7b45"],
  "extraction_method": "rss",
  "raw_metadata": {
    "rss_guid": "https://www.anthropic.com/research/claude-3-opus",
    "rss_categories": ["Research", "Claude"],
    "rss_source_feed": "https://www.anthropic.com/research/rss.xml"
  },
  "agent_notes": "Frontier model release — highest priority for AI digest. Auto-flagged for breaking digest.",
  "status": "published",
  "created_at": "2024-05-15T14:30:00Z",
  "updated_at": "2024-05-15T15:00:00Z"
}
```

---

## 3. ScrapeJob Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ScrapeJob",
  "description": "A single scraping operation/execution record",
  "type": "object",
  "required": ["id", "source_id", "topic", "status"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier (UUIDv7)",
      "example": "018f5a5d-ce4f-ad70-d283-6f0b1e4f8c56"
    },
    "source_id": {
      "type": "string",
      "format": "uuid",
      "description": "Foreign key to sources.id",
      "example": "018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12"
    },
    "topic": {
      "type": "string",
      "description": "Topic slug — denormalized for fast lookup",
      "example": "ai"
    },
    "status": {
      "type": "string",
      "enum": ["queued", "running", "success", "failed", "retrying", "cancelled", "timeout"],
      "description": "Current execution status",
      "example": "success"
    },
    "scheduled_at": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "When the job was scheduled to run (ISO 8601)",
      "example": "2024-05-15T14:15:00Z"
    },
    "started_at": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "When the job actually started (ISO 8601)",
      "example": "2024-05-15T14:15:02Z"
    },
    "completed_at": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "When the job finished (ISO 8601)",
      "example": "2024-05-15T14:15:08Z"
    },
    "items_found": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Total items discovered during scraping",
      "example": 12
    },
    "items_new": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Items that were newly inserted (not duplicates)",
      "example": 3
    },
    "items_duplicated": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Items that matched existing entries (duplicates)",
      "example": 9
    },
    "items_failed": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Items that failed extraction/parsing",
      "example": 0
    },
    "error_message": {
      "type": ["string", "null"],
      "maxLength": 4000,
      "description": "Error message if the job failed",
      "example": null
    },
    "error_traceback": {
      "type": ["string", "null"],
      "description": "Full exception traceback for debugging",
      "example": null
    },
    "retry_count": {
      "type": "integer",
      "minimum": 0,
      "default": 0,
      "description": "Number of retry attempts so far",
      "example": 0
    },
    "max_retries": {
      "type": "integer",
      "minimum": 0,
      "default": 3,
      "description": "Maximum retry attempts allowed",
      "example": 3
    },
    "adapter_used": {
      "type": ["string", "null"],
      "description": "Adapter module that executed this job",
      "example": "anthropic_blog_adapter"
    },
    "execution_time_ms": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Total execution time in milliseconds",
      "example": 6123
    },
    "network_time_ms": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Total network wait time in milliseconds",
      "example": 4800
    },
    "requests_made": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Number of HTTP requests made",
      "example": 4
    },
    "raw_log": {
      "type": ["string", "null"],
      "description": "Full execution log text for debugging",
      "example": "[2024-05-15T14:15:02Z] INFO: Starting scrape of anthropic_blog...\n[2024-05-15T14:15:04Z] INFO: Fetched 12 items from RSS feed\n[2024-05-15T14:15:08Z] INFO: Job completed successfully"
    },
    "triggered_by": {
      "type": ["string", "null"],
      "enum": ["schedule", "manual", "webhook", "retry_queue", "backfill"],
      "description": "What triggered this job",
      "example": "schedule"
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record creation timestamp (ISO 8601)",
      "example": "2024-05-15T14:15:00Z"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record last update timestamp (ISO 8601)",
      "example": "2024-05-15T14:15:08Z"
    }
  }
}
```

**Example Record:**
```json
{
  "id": "018f5a5d-ce4f-ad70-d283-6f0b1e4f8c56",
  "source_id": "018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12",
  "topic": "ai",
  "status": "success",
  "scheduled_at": "2024-05-15T14:15:00Z",
  "started_at": "2024-05-15T14:15:02Z",
  "completed_at": "2024-05-15T14:15:08Z",
  "items_found": 12,
  "items_new": 3,
  "items_duplicated": 9,
  "items_failed": 0,
  "error_message": null,
  "error_traceback": null,
  "retry_count": 0,
  "max_retries": 3,
  "adapter_used": "anthropic_blog_adapter",
  "execution_time_ms": 6123,
  "network_time_ms": 4800,
  "requests_made": 4,
  "raw_log": "[2024-05-15T14:15:02Z] INFO: Starting scrape of anthropic_blog\n[2024-05-15T14:15:03Z] INFO: Fetching RSS feed...\n[2024-05-15T14:15:04Z] INFO: Fetched 12 items from RSS feed\n[2024-05-15T14:15:05Z] INFO: Processing item 1/12: 'Claude 3 Opus: Advancing...'\n[2024-05-15T14:15:08Z] INFO: Job completed: 3 new, 9 dupes, 0 failed",
  "triggered_by": "schedule",
  "created_at": "2024-05-15T14:15:00Z",
  "updated_at": "2024-05-15T14:15:08Z"
}
```

---

## 4. Topic Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Topic",
  "description": "A content topic/category in Jarvis HQ",
  "type": "object",
  "required": ["id", "name"],
  "properties": {
    "id": {
      "type": "string",
      "maxLength": 50,
      "pattern": "^[a-z0-9_]+$",
      "description": "URL-safe slug identifier (primary key)",
      "example": "ai"
    },
    "name": {
      "type": "string",
      "maxLength": 100,
      "description": "Human-readable display name",
      "example": "Artificial Intelligence"
    },
    "description": {
      "type": ["string", "null"],
      "maxLength": 1000,
      "description": "Description of what this topic covers",
      "example": "Frontier AI research, LLM releases, AI safety, and machine learning breakthroughs"
    },
    "keywords": {
      "type": ["array", "null"],
      "items": { "type": "string", "maxLength": 100 },
      "description": "Keywords used for automatic content matching and tagging",
      "example": ["artificial intelligence", "machine learning", "LLM", "transformer", "neural network", "GPT", "Claude", "Gemini"]
    },
    "include_filters": {
      "type": ["array", "null"],
      "items": { "type": "string", "maxLength": 500 },
      "description": "Regex patterns or keywords — item must match at least one to be included",
      "example": ["AI", "artificial intelligence", "machine learning", "LLM", "language model", "neural network"]
    },
    "exclude_filters": {
      "type": ["array", "null"],
      "items": { "type": "string", "maxLength": 500 },
      "description": "Regex patterns or keywords — items matching any are excluded",
      "example": ["AI washing", "crypto.*AI", "AI-generated spam"]
    },
    "sources": {
      "type": ["array", "null"],
      "items": { "type": "string", "format": "uuid" },
      "description": "UUIDs of sources assigned to this topic",
      "example": ["018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12", "018f5a6f-df50-be81-e394-7a1c2f5g9d67"]
    },
    "active": {
      "type": "boolean",
      "default": true,
      "description": "Whether this topic is actively being scraped",
      "example": true
    },
    "schedule": {
      "type": ["string", "null"],
      "description": "Default cron expression for all sources in this topic",
      "example": "*/30 * * * *"
    },
    "last_scraped": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "Timestamp of the most recent scrape across all sources (ISO 8601)",
      "example": "2024-05-15T14:30:00Z"
    },
    "item_count": {
      "type": "integer",
      "minimum": 0,
      "default": 0,
      "description": "Total items collected for this topic (approximate, updated periodically)",
      "example": 15420
    },
    "quality_threshold": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "default": 30,
      "description": "Minimum quality score for items to be included in digests (0-100)",
      "example": 50
    },
    "digest_config": {
      "type": ["object", "null"],
      "description": "Digest generation configuration for this topic",
      "example": {
        "daily_digest": { "enabled": true, "max_items": 20, "time": "09:00" },
        "breaking_digest": { "enabled": true, "threshold_score": 85 },
        "weekly_digest": { "enabled": true, "day": "sunday", "time": "10:00" }
      }
    },
    "output_channels": {
      "type": ["array", "null"],
      "items": { "type": "string", "enum": ["website", "telegram", "n8n", "email", "webhook"] },
      "description": "Default output channels for this topic's digests",
      "example": ["website", "telegram"]
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record creation timestamp (ISO 8601)",
      "example": "2024-01-10T08:00:00Z"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record last update timestamp (ISO 8601)",
      "example": "2024-05-15T14:30:00Z"
    }
  }
}
```

**Example Record:**
```json
{
  "id": "ai",
  "name": "Artificial Intelligence",
  "description": "Frontier AI research, LLM releases, AI safety, and machine learning breakthroughs",
  "keywords": ["artificial intelligence", "machine learning", "LLM", "transformer", "neural network", "GPT", "Claude", "Gemini", "Mistral", "Llama"],
  "include_filters": ["AI", "artificial intelligence", "machine learning", "LLM", "language model", "neural network", "foundation model", "frontier model"],
  "exclude_filters": ["AI washing", "crypto.*AI", "AI-generated spam", "fortnite ai"],
  "sources": ["018f5a2e-8b1c-7a3d-9e4f-2c6d8a0b4e12", "018f5a6f-df50-be81-e394-7a1c2f5g9d67"],
  "active": true,
  "schedule": "*/30 * * * *",
  "last_scraped": "2024-05-15T14:30:00Z",
  "item_count": 15420,
  "quality_threshold": 50,
  "digest_config": {
    "daily_digest": { "enabled": true, "max_items": 20, "time": "09:00", "timezone": "UTC" },
    "breaking_digest": { "enabled": true, "threshold_score": 85, "min_items": 1 },
    "weekly_digest": { "enabled": true, "day": "sunday", "time": "10:00", "timezone": "UTC" }
  },
  "output_channels": ["website", "telegram"],
  "created_at": "2024-01-10T08:00:00Z",
  "updated_at": "2024-05-15T14:30:00Z"
}
```

---

## 5. QualityScore Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "QualityScore",
  "description": "Detailed quality scoring breakdown for a feed item",
  "type": "object",
  "required": ["item_id", "final_score", "tier", "scored_at"],
  "properties": {
    "item_id": {
      "type": "string",
      "format": "uuid",
      "description": "Foreign key to feed_items.id",
      "example": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23"
    },
    "source_score": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Source quality score (0-100) — inherited from source + history",
      "example": 94.0
    },
    "source_score_breakdown": {
      "type": ["object", "null"],
      "description": "Per-dimension source quality breakdown",
      "example": {
        "domain_authority": 95,
        "historical_accuracy": 93,
        "update_frequency": 90,
        "content_depth": 96,
        "citation_practice": 92,
        "bias_neutrality": 88
      }
    },
    "content_score": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Content quality score (0-100) — computed from the article itself",
      "example": 89.5
    },
    "content_score_breakdown": {
      "type": ["object", "null"],
      "description": "Per-dimension content quality breakdown",
      "example": {
        "readability": 88,
        "originality": 92,
        "factual_density": 85,
        "structural_quality": 91,
        "media_richness": 87,
        "length_appropriateness": 90,
        "keyword_relevance": 94
      }
    },
    "final_score": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Final composite quality score (0-100) — weighted combination of all dimensions",
      "example": 91.75
    },
    "tier": {
      "type": "integer",
      "minimum": 1,
      "maximum": 5,
      "description": "Quality tier: 1 = excellent (80-100), 2 = good (60-79), 3 = average (40-59), 4 = below average (20-39), 5 = poor (0-19)",
      "example": 1
    },
    "freshness_score": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 100,
      "description": "Freshness score with time decay applied (0-100)",
      "example": 88.5
    },
    "freshness_decay_applied": {
      "type": "boolean",
      "default": true,
      "description": "Whether time-decay has been applied to the freshness score",
      "example": true
    },
    "freshness_half_life_hours": {
      "type": ["number", "null"],
      "description": "Half-life in hours used for decay calculation. Varies by topic (breaking news = 6h, research = 168h).",
      "example": 48
    },
    "deduplication_status": {
      "type": ["string", "null"],
      "enum": ["original", "duplicate", "canonical_unknown", "near_duplicate"],
      "description": "Deduplication classification",
      "example": "original"
    },
    "duplicate_of": {
      "type": ["string", "null"],
      "format": "uuid",
      "description": "If duplicate, the UUID of the canonical/original item",
      "example": null
    },
    "duplicate_similarity": {
      "type": ["number", "null"],
      "minimum": 0,
      "maximum": 1,
      "description": "Similarity score to the canonical item (0-1, Jaccard or cosine)",
      "example": null
    },
    "scored_at": {
      "type": "string",
      "format": "date-time",
      "description": "When the scoring was performed (ISO 8601)",
      "example": "2024-05-15T14:31:00Z"
    },
    "scorer_version": {
      "type": "string",
      "description": "Version of the scoring algorithm/module used",
      "example": "quality-v2.3.1"
    },
    "scoring_metadata": {
      "type": ["object", "null"],
      "description": "Additional scoring metadata and debug info",
      "example": {
        "freshness_formula": "exp(-0.0144 * hours_since_publish)",
        "content_weight": 0.6,
        "source_weight": 0.3,
        "freshness_weight": 0.1,
        "topic_boost": 1.0
      }
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record creation timestamp (ISO 8601)",
      "example": "2024-05-15T14:31:00Z"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record last update timestamp (ISO 8601)",
      "example": "2024-05-15T14:31:00Z"
    }
  }
}
```

**Example Record:**
```json
{
  "item_id": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23",
  "source_score": 94.0,
  "source_score_breakdown": {
    "domain_authority": 95,
    "historical_accuracy": 93,
    "update_frequency": 90,
    "content_depth": 96,
    "citation_practice": 92,
    "bias_neutrality": 88
  },
  "content_score": 89.5,
  "content_score_breakdown": {
    "readability": 88,
    "originality": 92,
    "factual_density": 85,
    "structural_quality": 91,
    "media_richness": 87,
    "length_appropriateness": 90,
    "keyword_relevance": 94
  },
  "final_score": 91.75,
  "tier": 1,
  "freshness_score": 88.5,
  "freshness_decay_applied": true,
  "freshness_half_life_hours": 48,
  "deduplication_status": "original",
  "duplicate_of": null,
  "duplicate_similarity": null,
  "scored_at": "2024-05-15T14:31:00Z",
  "scorer_version": "quality-v2.3.1",
  "scoring_metadata": {
    "freshness_formula": "exp(-0.0144 * hours_since_publish)",
    "content_weight": 0.6,
    "source_weight": 0.3,
    "freshness_weight": 0.1,
    "topic_boost": 1.0
  },
  "created_at": "2024-05-15T14:31:00Z",
  "updated_at": "2024-05-15T14:31:00Z"
}
```

---

## 6. DigestEntry Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DigestEntry",
  "description": "A prepared briefing/digest ready for distribution",
  "type": "object",
  "required": ["id", "topic", "type", "title", "format", "generated_at"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier (UUIDv7)",
      "example": "018f5a7e-eg61-cf92-f4a5-8b2d3g6h0e78"
    },
    "topic": {
      "type": "string",
      "description": "Topic slug this digest covers",
      "example": "ai"
    },
    "type": {
      "type": "string",
      "enum": ["daily", "breaking", "weekly", "on_demand", "topic_summary", "trend_alert"],
      "description": "Type of digest",
      "example": "daily"
    },
    "title": {
      "type": "string",
      "maxLength": 300,
      "description": "Digest title/headline",
      "example": "AI Daily Digest — May 15, 2024"
    },
    "summary": {
      "type": ["string", "null"],
      "maxLength": 10000,
      "description": "Auto-generated overview/summary of the digest contents",
      "example": "Today's AI digest covers Anthropic's Claude 3 Opus benchmarks, OpenAI's new API pricing changes, and Google's Gemini 1.5 Pro research preview. Three high-impact stories with strong signal across all quality dimensions."
    },
    "items": {
      "type": ["array", "null"],
      "items": {
        "type": "object",
        "properties": {
          "item_id": { "type": "string", "format": "uuid" },
          "title": { "type": "string" },
          "summary": { "type": "string" },
          "url": { "type": "string", "format": "uri" },
          "source_name": { "type": "string" },
          "quality_score": { "type": "number" },
          "published_date": { "type": "string", "format": "date-time" },
          "importance_note": { "type": ["string", "null"] },
          "media_url": { "type": ["string", "null"], "format": "uri" }
        }
      },
      "description": "Array of included feed item summaries",
      "example": [
        {
          "item_id": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23",
          "title": "Claude 3 Opus: Advancing the Frontier of AI Capability",
          "summary": "Anthropic's latest frontier model achieves SOTA on graduate-level reasoning benchmarks...",
          "url": "https://www.anthropic.com/research/claude-3-opus",
          "source_name": "Anthropic Research Blog",
          "quality_score": 91.75,
          "published_date": "2024-03-04T10:00:00Z",
          "importance_note": "Frontier model release — highest tier",
          "media_url": "https://cdn.anthropic.com/claude-3-opus-hero.png"
        }
      ]
    },
    "item_count": {
      "type": ["integer", "null"],
      "minimum": 0,
      "description": "Total number of items in this digest",
      "example": 15
    },
    "top_items": {
      "type": ["array", "null"],
      "items": {
        "type": "object",
        "properties": {
          "item_id": { "type": "string", "format": "uuid" },
          "title": { "type": "string" },
          "url": { "type": "string", "format": "uri" },
          "score": { "type": "number" },
          "highlight": { "type": ["string", "null"] }
        }
      },
      "description": "Top N highest-scored items for quick scanning",
      "example": [
        {
          "item_id": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23",
          "title": "Claude 3 Opus: Advancing the Frontier of AI Capability",
          "url": "https://www.anthropic.com/research/claude-3-opus",
          "score": 91.75,
          "highlight": "New SOTA on MMLU and HumanEval benchmarks"
        }
      ]
    },
    "format": {
      "type": "string",
      "enum": ["markdown", "html", "telegram_markdown", "json", "plain_text"],
      "description": "Output format of the digest",
      "example": "telegram_markdown"
    },
    "rendered_content": {
      "type": ["string", "null"],
      "description": "Pre-rendered digest content in the chosen format. Stored as file path reference for large digests.",
      "example": "/data/digests/2024/05/15/018f5a7e-eg61-cf92-f4a5-8b2d3g6h0e78.md"
    },
    "generated_at": {
      "type": "string",
      "format": "date-time",
      "description": "When the digest was generated (ISO 8601)",
      "example": "2024-05-15T09:00:00Z"
    },
    "sent_to": {
      "type": ["array", "null"],
      "items": { "type": "string", "enum": ["telegram", "website", "email", "n8n", "webhook", "rss"] },
      "description": "Channels this digest was sent to",
      "example": ["telegram", "website"]
    },
    "delivery_status": {
      "type": ["object", "null"],
      "description": "Per-channel delivery status",
      "example": {
        "telegram": { "status": "sent", "sent_at": "2024-05-15T09:00:05Z", "message_id": "123456789" },
        "website": { "status": "sent", "sent_at": "2024-05-15T09:00:03Z", "post_id": "ai-daily-2024-05-15" },
        "n8n": { "status": "sent", "sent_at": "2024-05-15T09:00:04Z", "workflow_id": "wf_abc123" }
      }
    },
    "status": {
      "type": "string",
      "enum": ["draft", "queued", "sending", "sent", "partially_sent", "failed", "cancelled"],
      "default": "draft",
      "description": "Digest lifecycle status",
      "example": "sent"
    },
    "quality_stats": {
      "type": ["object", "null"],
      "description": "Aggregate quality statistics for items in this digest",
      "example": {
        "avg_score": 78.5,
        "min_score": 52.0,
        "max_score": 91.75,
        "tier_1_count": 3,
        "tier_2_count": 8,
        "tier_3_count": 4
      }
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record creation timestamp (ISO 8601)",
      "example": "2024-05-15T09:00:00Z"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time",
      "description": "Record last update timestamp (ISO 8601)",
      "example": "2024-05-15T09:00:05Z"
    }
  }
}
```

**Example Record:**
```json
{
  "id": "018f5a7e-eg61-cf92-f4a5-8b2d3g6h0e78",
  "topic": "ai",
  "type": "daily",
  "title": "AI Daily Digest — May 15, 2024",
  "summary": "Today's AI digest covers Anthropic's Claude 3 Opus benchmarks, OpenAI's new API pricing changes, and Google's Gemini 1.5 Pro research preview. Three high-impact stories with strong signal across all quality dimensions.",
  "items": [
    {
      "item_id": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23",
      "title": "Claude 3 Opus: Advancing the Frontier of AI Capability",
      "summary": "Anthropic's latest frontier model achieves SOTA on graduate-level reasoning benchmarks...",
      "url": "https://www.anthropic.com/research/claude-3-opus",
      "source_name": "Anthropic Research Blog",
      "quality_score": 91.75,
      "published_date": "2024-03-04T10:00:00Z",
      "importance_note": "Frontier model release — highest tier",
      "media_url": "https://cdn.anthropic.com/claude-3-opus-hero.png"
    }
  ],
  "item_count": 15,
  "top_items": [
    {
      "item_id": "018f5a3b-9c2d-8b4e-af50-3d7e9b1c5f23",
      "title": "Claude 3 Opus: Advancing the Frontier of AI Capability",
      "url": "https://www.anthropic.com/research/claude-3-opus",
      "score": 91.75,
      "highlight": "New SOTA on MMLU and HumanEval benchmarks"
    }
  ],
  "format": "telegram_markdown",
  "rendered_content": "/data/digests/2024/05/15/018f5a7e-eg61-cf92-f4a5-8b2d3g6h0e78.md",
  "generated_at": "2024-05-15T09:00:00Z",
  "sent_to": ["telegram", "website", "n8n"],
  "delivery_status": {
    "telegram": { "status": "sent", "sent_at": "2024-05-15T09:00:05Z", "message_id": "123456789" },
    "website": { "status": "sent", "sent_at": "2024-05-15T09:00:03Z", "post_id": "ai-daily-2024-05-15" },
    "n8n": { "status": "sent", "sent_at": "2024-05-15T09:00:04Z", "workflow_id": "wf_abc123" }
  },
  "status": "sent",
  "quality_stats": {
    "avg_score": 78.5,
    "min_score": 52.0,
    "max_score": 91.75,
    "tier_1_count": 3,
    "tier_2_count": 8,
    "tier_3_count": 4
  },
  "created_at": "2024-05-15T09:00:00Z",
  "updated_at": "2024-05-15T09:00:05Z"
}
```

---

## 7. Database Schema (SQL)

**Notes on SQLite vs Postgres:**
- SQLite uses `TEXT` for JSON, `INTEGER` for booleans (0/1), `REAL` for decimals
- Postgres uses `JSONB` for JSON, `BOOLEAN`, `TIMESTAMPTZ`, `DECIMAL(5,2)` for scores
- Both support UUID primary keys (Postgres native, SQLite via `TEXT` with validation)
- Postgres-specific enhancements noted in comments below each table

```sql
-- ============================================================
-- Jarvis HQ Database Schema
-- Compatible with SQLite 3.35+ and PostgreSQL 14+
-- ============================================================

-- Enable foreign key support (SQLite; Postgres has it on by default)
PRAGMA foreign_keys = ON;

-- ============================================================
-- 1. TOPICS
-- ============================================================
CREATE TABLE topics (
    id              TEXT PRIMARY KEY,                          -- URL-safe slug: ai, gaming, tech
    name            TEXT NOT NULL,                             -- Display name
    description     TEXT,                                      -- Human-readable description
    keywords        TEXT,                                      -- JSON array of keywords
    include_filters TEXT,                                      -- JSON array of regex/keywords
    exclude_filters TEXT,                                      -- JSON array of regex/keywords
    active          INTEGER NOT NULL DEFAULT 1,               -- 0=inactive, 1=active (BOOLEAN in PG)
    schedule        TEXT,                                      -- Default cron expression
    last_scraped    TEXT,                                      -- ISO-8601 timestamp
    item_count      INTEGER NOT NULL DEFAULT 0,               -- Approximate count
    quality_threshold REAL NOT NULL DEFAULT 30.0,             -- Min score for digest inclusion
    digest_config   TEXT,                                      -- JSON: digest generation config
    output_channels TEXT,                                      -- JSON array of channel names
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),  -- ISO-8601
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))   -- ISO-8601
);

-- Indexes: topics
CREATE INDEX idx_topics_active ON topics(active);
CREATE INDEX idx_topics_last_scraped ON topics(last_scraped);

-- ============================================================
-- 2. SOURCES
-- ============================================================
CREATE TABLE sources (
    id                      TEXT PRIMARY KEY,                          -- UUIDv7
    name                    TEXT NOT NULL,                             -- Human-readable name
    url                     TEXT NOT NULL,                             -- Primary URL
    feed_url                TEXT,                                      -- RSS/Atom feed URL
    type                    TEXT NOT NULL CHECK(type IN (
                                'blog', 'rss_feed', 'api', 'github', 'youtube',
                                'twitter', 'newsletter', 'news_site', 'forum',
                                'docs', 'changelog', 'press_room', 'sitemap', 'search'
                            )),
    topic                   TEXT NOT NULL REFERENCES topics(id) ON DELETE CASCADE,
    status                  TEXT NOT NULL DEFAULT 'pending_verification'
                            CHECK(status IN ('active', 'paused', 'error',
                                'pending_verification', 'rejected')),
    priority                INTEGER NOT NULL DEFAULT 5
                            CHECK(priority BETWEEN 1 AND 10),
    schedule                TEXT,                                      -- Source-specific cron
    adapter                 TEXT,                                      -- Adapter module name
    config                  TEXT,                                      -- JSON: adapter-specific config
    quality_score           REAL CHECK(quality_score BETWEEN 0 AND 100),
    quality_tier            INTEGER CHECK(quality_tier BETWEEN 1 AND 5),
    last_scraped            TEXT,                                      -- ISO-8601
    last_success            TEXT,                                      -- ISO-8601
    last_error              TEXT,                                      -- Error message string
    error_count             INTEGER NOT NULL DEFAULT 0,
    success_count           INTEGER NOT NULL DEFAULT 0,
    rate_limit              INTEGER CHECK(rate_limit BETWEEN 1 AND 10000),
    robots_txt_allowed      INTEGER,                                 -- 0/1/null (BOOLEAN in PG)
    auth_required           INTEGER NOT NULL DEFAULT 0,              -- 0/1 (BOOLEAN in PG)
    auth_type               TEXT DEFAULT 'none'
                            CHECK(auth_type IN ('none', 'api_key', 'oauth',
                                'bearer', 'basic', 'cookie')),
    auth_config             TEXT,                                      -- JSON: env var refs / encrypted
    notes                   TEXT,                                      -- Human operator notes
    tags                    TEXT,                                      -- JSON array of strings
    created_at              TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at              TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes: sources
CREATE INDEX idx_sources_topic ON sources(topic);
CREATE INDEX idx_sources_status ON sources(status);
CREATE INDEX idx_sources_type ON sources(type);
CREATE INDEX idx_sources_status_topic ON sources(status, topic);
CREATE INDEX idx_sources_priority ON sources(priority DESC);
CREATE INDEX idx_sources_quality_tier ON sources(quality_tier);
CREATE INDEX idx_sources_last_scraped ON sources(last_scraped);
CREATE INDEX idx_sources_adapter ON sources(adapter);

-- Partial index equivalent: only active sources (Postgres: WHERE status='active')
-- SQLite doesn't support partial indexes; use a composite index instead
CREATE INDEX idx_sources_active_priority ON sources(status, priority DESC) WHERE status = 'active';

-- ============================================================
-- 3. FEED_ITEMS
-- ============================================================
CREATE TABLE feed_items (
    id                      TEXT PRIMARY KEY,                          -- UUIDv7
    source_id               TEXT NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    topic                   TEXT NOT NULL REFERENCES topics(id) ON DELETE CASCADE,
    title                   TEXT NOT NULL,                             -- Article title
    summary                 TEXT,                                      -- Extracted/generated summary
    full_text_path          TEXT,                                      -- Path to full text file on disk
    url                     TEXT NOT NULL,                             -- Original URL
    canonical_url           TEXT,                                      -- Resolved canonical URL
    author                  TEXT,                                      -- Author name/handle
    author_url              TEXT,                                      -- Author profile URL
    published_date          TEXT,                                      -- ISO-8601 from source
    discovered_date         TEXT NOT NULL DEFAULT (datetime('now')),  -- ISO-8601
    updated_date            TEXT,                                      -- ISO-8601 if updated
    language                TEXT CHECK(length(language) <= 10),       -- ISO 639-1 code
    region                  TEXT CHECK(length(region) <= 10),         -- ISO 3166-1 alpha-2
    tags                    TEXT,                                      -- JSON array of strings
    content_type            TEXT NOT NULL DEFAULT 'unknown'
                            CHECK(content_type IN (
                                'article', 'release', 'video', 'tweet', 'changelog',
                                'paper', 'podcast', 'documentation', 'press_release',
                                'blog_post', 'forum_post', 'commentary', 'tutorial',
                                'review', 'announcement', 'research', 'news', 'unknown'
                            )),
    media_urls              TEXT,                                      -- JSON array of media objects
    quality_score           REAL CHECK(quality_score BETWEEN 0 AND 100),
    quality_tier            INTEGER CHECK(quality_tier BETWEEN 1 AND 5),
    freshness_score         REAL CHECK(freshness_score BETWEEN 0 AND 100),
    source_quality_score    REAL CHECK(source_quality_score BETWEEN 0 AND 100),
    confidence_score        REAL CHECK(confidence_score BETWEEN 0 AND 100),
    sentiment_score         REAL CHECK(sentiment_score BETWEEN -1 AND 1),
    importance_score        REAL CHECK(importance_score BETWEEN 0 AND 100),
    duplicate_group_id      TEXT REFERENCES duplicate_groups(id) ON DELETE SET NULL,
    is_primary              INTEGER,                                 -- 0/1/null (BOOLEAN in PG)
    related_item_ids        TEXT,                                      -- JSON array of UUIDs
    extraction_method       TEXT CHECK(extraction_method IN (
                                'rss', 'api', 'html_scrape', 'html_readability',
                                'youtube_api', 'github_api', 'twitter_api',
                                'newsletter_parse', 'pdf_extract', 'manual_entry'
                            )),
    raw_metadata            TEXT,                                      -- JSON: original raw data
    agent_notes             TEXT,                                      -- Agent/reviewer notes
    status                  TEXT NOT NULL DEFAULT 'new'
                            CHECK(status IN ('new', 'processed', 'reviewed',
                                'rejected', 'published', 'archived')),
    created_at              TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at              TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes: feed_items (performance-critical table)
CREATE INDEX idx_feed_items_source_id ON feed_items(source_id);
CREATE INDEX idx_feed_items_topic ON feed_items(topic);
CREATE INDEX idx_feed_items_status ON feed_items(status);
CREATE INDEX idx_feed_items_topic_status ON feed_items(topic, status);
CREATE INDEX idx_feed_items_content_type ON feed_items(content_type);
CREATE INDEX idx_feed_items_published_date ON feed_items(published_date DESC);
CREATE INDEX idx_feed_items_discovered_date ON feed_items(discovered_date DESC);
CREATE INDEX idx_feed_items_quality_score ON feed_items(quality_score DESC);
CREATE INDEX idx_feed_items_quality_tier ON feed_items(quality_tier);
CREATE INDEX idx_feed_items_freshness ON feed_items(freshness_score DESC);
CREATE INDEX idx_feed_items_duplicate_group ON feed_items(duplicate_group_id);
CREATE INDEX idx_feed_items_is_primary ON feed_items(is_primary);
CREATE INDEX idx_feed_items_url ON feed_items(url);
CREATE INDEX idx_feed_items_canonical_url ON feed_items(canonical_url);
CREATE INDEX idx_feed_items_language ON feed_items(language);

-- Composite index for the most common query: topic + status + quality, sorted by freshness
CREATE INDEX idx_feed_items_topic_status_freshness
    ON feed_items(topic, status, freshness_score DESC);

-- Composite index for digest generation: topic + published, filtered by quality
CREATE INDEX idx_feed_items_topic_published
    ON feed_items(topic, published_date DESC);

-- ============================================================
-- 4. SCRAPE_JOBS
-- ============================================================
CREATE TABLE scrape_jobs (
    id                  TEXT PRIMARY KEY,                          -- UUIDv7
    source_id           TEXT NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    topic               TEXT NOT NULL REFERENCES topics(id) ON DELETE CASCADE,
    status              TEXT NOT NULL DEFAULT 'queued'
                        CHECK(status IN ('queued', 'running', 'success',
                            'failed', 'retrying', 'cancelled', 'timeout')),
    scheduled_at        TEXT,                                      -- ISO-8601
    started_at          TEXT,                                      -- ISO-8601
    completed_at        TEXT,                                      -- ISO-8601
    items_found         INTEGER DEFAULT 0 CHECK(items_found >= 0),
    items_new           INTEGER DEFAULT 0 CHECK(items_new >= 0),
    items_duplicated    INTEGER DEFAULT 0 CHECK(items_duplicated >= 0),
    items_failed        INTEGER DEFAULT 0 CHECK(items_failed >= 0),
    error_message       TEXT,                                      -- Human-readable error
    error_traceback     TEXT,                                      -- Full traceback
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 3,
    adapter_used        TEXT,                                      -- Adapter module name
    execution_time_ms   INTEGER CHECK(execution_time_ms >= 0),
    network_time_ms     INTEGER CHECK(network_time_ms >= 0),
    requests_made       INTEGER CHECK(requests_made >= 0),
    raw_log             TEXT,                                      -- Full execution log
    triggered_by        TEXT CHECK(triggered_by IN (
                            'schedule', 'manual', 'webhook', 'retry_queue', 'backfill'
                        )),
    created_at          TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at          TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes: scrape_jobs
CREATE INDEX idx_scrape_jobs_source_id ON scrape_jobs(source_id);
CREATE INDEX idx_scrape_jobs_topic ON scrape_jobs(topic);
CREATE INDEX idx_scrape_jobs_status ON scrape_jobs(status);
CREATE INDEX idx_scrape_jobs_scheduled_at ON scrape_jobs(scheduled_at);
CREATE INDEX idx_scrape_jobs_created_at ON scrape_jobs(created_at DESC);
CREATE INDEX idx_scrape_jobs_status_created ON scrape_jobs(status, created_at DESC);

-- ============================================================
-- 5. QUALITY_SCORES
-- ============================================================
CREATE TABLE quality_scores (
    item_id                     TEXT PRIMARY KEY REFERENCES feed_items(id) ON DELETE CASCADE,
    source_score                REAL NOT NULL CHECK(source_score BETWEEN 0 AND 100),
    source_score_breakdown      TEXT,                              -- JSON: per-dimension scores
    content_score               REAL NOT NULL CHECK(content_score BETWEEN 0 AND 100),
    content_score_breakdown     TEXT,                              -- JSON: per-dimension scores
    final_score                 REAL NOT NULL CHECK(final_score BETWEEN 0 AND 100),
    tier                        INTEGER NOT NULL CHECK(tier BETWEEN 1 AND 5),
    freshness_score             REAL CHECK(freshness_score BETWEEN 0 AND 100),
    freshness_decay_applied     INTEGER NOT NULL DEFAULT 1,      -- 0/1 (BOOLEAN in PG)
    freshness_half_life_hours   REAL,
    deduplication_status        TEXT CHECK(deduplication_status IN (
                                    'original', 'duplicate', 'canonical_unknown', 'near_duplicate'
                                )),
    duplicate_of                TEXT REFERENCES feed_items(id) ON DELETE SET NULL,
    duplicate_similarity        REAL CHECK(duplicate_similarity BETWEEN 0 AND 1),
    scored_at                   TEXT NOT NULL,                    -- ISO-8601
    scorer_version              TEXT NOT NULL,                    -- e.g. "quality-v2.3.1"
    scoring_metadata            TEXT,                              -- JSON: debug info
    created_at                  TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at                  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes: quality_scores
CREATE INDEX idx_quality_scores_final_score ON quality_scores(final_score DESC);
CREATE INDEX idx_quality_scores_tier ON quality_scores(tier);
CREATE INDEX idx_quality_scores_freshness ON quality_scores(freshness_score DESC);
CREATE INDEX idx_quality_scores_dedup ON quality_scores(deduplication_status);
CREATE INDEX idx_quality_scores_duplicate_of ON quality_scores(duplicate_of);
CREATE INDEX idx_quality_scores_scored_at ON quality_scores(scored_at DESC);

-- ============================================================
-- 6. DIGEST_ENTRIES
-- ============================================================
CREATE TABLE digest_entries (
    id              TEXT PRIMARY KEY,                          -- UUIDv7
    topic           TEXT NOT NULL REFERENCES topics(id) ON DELETE CASCADE,
    type            TEXT NOT NULL
                    CHECK(type IN ('daily', 'breaking', 'weekly', 'on_demand', 'topic_summary', 'trend_alert')),
    title           TEXT NOT NULL,
    summary         TEXT,                                      -- Auto-generated overview
    items           TEXT,                                      -- JSON: array of item summaries
    item_count      INTEGER CHECK(item_count >= 0),
    top_items       TEXT,                                      -- JSON: array of top N items
    format          TEXT NOT NULL
                    CHECK(format IN ('markdown', 'html', 'telegram_markdown', 'json', 'plain_text')),
    rendered_content_path TEXT,                                -- Path to rendered file on disk
    generated_at    TEXT NOT NULL,                            -- ISO-8601
    sent_to         TEXT,                                      -- JSON array of channel names
    delivery_status TEXT,                                      -- JSON: per-channel delivery status
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK(status IN ('draft', 'queued', 'sending', 'sent',
                        'partially_sent', 'failed', 'cancelled')),
    quality_stats   TEXT,                                      -- JSON: aggregate quality stats
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes: digest_entries
CREATE INDEX idx_digest_entries_topic ON digest_entries(topic);
CREATE INDEX idx_digest_entries_type ON digest_entries(type);
CREATE INDEX idx_digest_entries_status ON digest_entries(status);
CREATE INDEX idx_digest_entries_generated_at ON digest_entries(generated_at DESC);
CREATE INDEX idx_digest_entries_topic_type ON digest_entries(topic, type);

-- ============================================================
-- 7. DUPLICATE_GROUPS
-- ============================================================
CREATE TABLE duplicate_groups (
    id              TEXT PRIMARY KEY,                          -- UUIDv7
    canonical_item_id TEXT REFERENCES feed_items(id) ON DELETE SET NULL,
    title_signature TEXT,                                      -- Normalized title hash for grouping
    content_hash    TEXT,                                      -- SimHash or MinHash of content
    item_count      INTEGER NOT NULL DEFAULT 1,
    first_seen      TEXT NOT NULL DEFAULT (datetime('now')),  -- ISO-8601
    last_seen       TEXT NOT NULL DEFAULT (datetime('now')),  -- ISO-8601
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes: duplicate_groups
CREATE INDEX idx_duplicate_groups_canonical ON duplicate_groups(canonical_item_id);
CREATE INDEX idx_duplicate_groups_title_sig ON duplicate_groups(title_signature);
CREATE INDEX idx_duplicate_groups_content_hash ON duplicate_groups(content_hash);

-- ============================================================
-- 8. SOURCE_HEALTH_LOG
-- ============================================================
CREATE TABLE source_health_log (
    id              TEXT PRIMARY KEY,                          -- UUIDv7
    source_id       TEXT NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    event_type      TEXT NOT NULL
                    CHECK(event_type IN ('success', 'error', 'warning', 'info', 'status_change')),
    event_message   TEXT NOT NULL,                             -- Human-readable message
    error_code      TEXT,                                      -- Machine-readable error code
    details         TEXT,                                      -- JSON: structured event details
    items_found     INTEGER,
    items_new       INTEGER,
    execution_time_ms INTEGER,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes: source_health_log
CREATE INDEX idx_source_health_source_id ON source_health_log(source_id);
CREATE INDEX idx_source_health_event_type ON source_health_log(event_type);
CREATE INDEX idx_source_health_created_at ON source_health_log(created_at DESC);
CREATE INDEX idx_source_health_source_event ON source_health_log(source_id, event_type);

-- ============================================================
-- TRIGGER: Auto-update updated_at timestamps (SQLite)
-- ============================================================
CREATE TRIGGER trg_topics_updated_at
AFTER UPDATE ON topics
FOR EACH ROW
BEGIN
    UPDATE topics SET updated_at = datetime('now') WHERE id = NEW.id;
END;

CREATE TRIGGER trg_sources_updated_at
AFTER UPDATE ON sources
FOR EACH ROW
BEGIN
    UPDATE sources SET updated_at = datetime('now') WHERE id = NEW.id;
END;

CREATE TRIGGER trg_feed_items_updated_at
AFTER UPDATE ON feed_items
FOR EACH ROW
BEGIN
    UPDATE feed_items SET updated_at = datetime('now') WHERE id = NEW.id;
END;

CREATE TRIGGER trg_scrape_jobs_updated_at
AFTER UPDATE ON scrape_jobs
FOR EACH ROW
BEGIN
    UPDATE scrape_jobs SET updated_at = datetime('now') WHERE id = NEW.id;
END;

CREATE TRIGGER trg_quality_scores_updated_at
AFTER UPDATE ON quality_scores
FOR EACH ROW
BEGIN
    UPDATE quality_scores SET updated_at = datetime('now') WHERE item_id = NEW.item_id;
END;

CREATE TRIGGER trg_digest_entries_updated_at
AFTER UPDATE ON digest_entries
FOR EACH ROW
BEGIN
    UPDATE digest_entries SET updated_at = datetime('now') WHERE id = NEW.id;
END;

CREATE TRIGGER trg_duplicate_groups_updated_at
AFTER UPDATE ON duplicate_groups
FOR EACH ROW
BEGIN
    UPDATE duplicate_groups SET updated_at = datetime('now') WHERE id = NEW.id;
END;

-- ============================================================
-- VIEWS: Common Query Patterns
-- ============================================================

-- View: Ready-to-publish items (for website/Telegram)
CREATE VIEW v_ready_items AS
SELECT
    fi.*,
    qs.final_score AS computed_score,
    qs.tier AS computed_tier,
    s.name AS source_name,
    s.type AS source_type,
    s.quality_score AS source_current_score
FROM feed_items fi
LEFT JOIN quality_scores qs ON fi.id = qs.item_id
LEFT JOIN sources s ON fi.source_id = s.id
WHERE fi.status IN ('new', 'processed')
  AND (fi.quality_score IS NULL OR fi.quality_score >= 40)
  AND fi.is_primary IN (1, NULL)
ORDER BY fi.quality_score DESC NULLS LAST, fi.freshness_score DESC NULLS LAST;

-- View: Source health dashboard
CREATE VIEW v_source_health AS
SELECT
    s.id,
    s.name,
    s.type,
    s.topic,
    s.status,
    s.quality_score,
    s.quality_tier,
    s.error_count,
    s.success_count,
    s.last_scraped,
    s.last_success,
    s.last_error,
    CASE
        WHEN s.error_count >= 5 THEN 'critical'
        WHEN s.error_count >= 3 THEN 'warning'
        WHEN s.last_success IS NULL THEN 'unverified'
        ELSE 'healthy'
    END AS health_status,
    (SELECT COUNT(*) FROM feed_items fi WHERE fi.source_id = s.id) AS total_items,
    (SELECT COUNT(*) FROM feed_items fi WHERE fi.source_id = s.id AND fi.status = 'published') AS published_items,
    (SELECT MAX(created_at) FROM source_health_log shl WHERE shl.source_id = s.id) AS last_event_at
FROM sources s;

-- View: Daily digest candidate items
CREATE VIEW v_digest_candidates AS
SELECT
    fi.id,
    fi.title,
    fi.summary,
    fi.url,
    fi.source_id,
    s.name AS source_name,
    fi.topic,
    fi.published_date,
    fi.discovered_date,
    fi.quality_score,
    fi.freshness_score,
    fi.importance_score,
    qs.tier,
    fi.content_type,
    fi.media_urls,
    fi.tags,
    ROW_NUMBER() OVER (PARTITION BY fi.topic ORDER BY fi.quality_score DESC, fi.freshness_score DESC) AS rank_in_topic
FROM feed_items fi
LEFT JOIN quality_scores qs ON fi.id = qs.item_id
LEFT JOIN sources s ON fi.source_id = s.id
WHERE fi.status IN ('new', 'processed', 'published')
  AND fi.is_primary IN (1, NULL)
  AND qs.deduplication_status IN ('original', NULL)
ORDER BY fi.quality_score DESC, fi.freshness_score DESC;
```

---

## 8. PostgreSQL-Specific Enhancements

When migrating to PostgreSQL, apply these additions for production performance:

```sql
-- ============================================================
-- POSTGRESQL MIGRATION: Type Upgrades
-- ============================================================

-- Convert JSON TEXT columns to JSONB for indexing
ALTER TABLE sources ALTER COLUMN config TYPE JSONB USING config::jsonb;
ALTER TABLE sources ALTER COLUMN auth_config TYPE JSONB USING auth_config::jsonb;
ALTER TABLE sources ALTER COLUMN tags TYPE JSONB USING tags::jsonb;
ALTER TABLE feed_items ALTER COLUMN tags TYPE JSONB USING tags::jsonb;
ALTER TABLE feed_items ALTER COLUMN media_urls TYPE JSONB USING media_urls::jsonb;
ALTER TABLE feed_items ALTER COLUMN raw_metadata TYPE JSONB USING raw_metadata::jsonb;
ALTER TABLE feed_items ALTER COLUMN related_item_ids TYPE JSONB USING related_item_ids::jsonb;
ALTER TABLE topics ALTER COLUMN keywords TYPE JSONB USING keywords::jsonb;
ALTER TABLE topics ALTER COLUMN include_filters TYPE JSONB USING include_filters::jsonb;
ALTER TABLE topics ALTER COLUMN exclude_filters TYPE JSONB USING exclude_filters::jsonb;
ALTER TABLE topics ALTER COLUMN digest_config TYPE JSONB USING digest_config::jsonb;
ALTER TABLE topics ALTER COLUMN output_channels TYPE JSONB USING output_channels::jsonb;
ALTER TABLE quality_scores ALTER COLUMN source_score_breakdown TYPE JSONB USING source_score_breakdown::jsonb;
ALTER TABLE quality_scores ALTER COLUMN content_score_breakdown TYPE JSONB USING content_score_breakdown::jsonb;
ALTER TABLE quality_scores ALTER COLUMN scoring_metadata TYPE JSONB USING scoring_metadata::jsonb;
ALTER TABLE digest_entries ALTER COLUMN items TYPE JSONB USING items::jsonb;
ALTER TABLE digest_entries ALTER COLUMN top_items TYPE JSONB USING top_items::jsonb;
ALTER TABLE digest_entries ALTER COLUMN sent_to TYPE JSONB USING sent_to::jsonb;
ALTER TABLE digest_entries ALTER COLUMN delivery_status TYPE JSONB USING delivery_status::jsonb;
ALTER TABLE digest_entries ALTER COLUMN quality_stats TYPE JSONB USING quality_stats::jsonb;
ALTER TABLE source_health_log ALTER COLUMN details TYPE JSONB USING details::jsonb;

-- Convert INTEGER booleans to BOOLEAN
ALTER TABLE sources ALTER COLUMN robots_txt_allowed TYPE BOOLEAN
    USING CASE WHEN robots_txt_allowed = 1 THEN TRUE WHEN robots_txt_allowed = 0 THEN FALSE ELSE NULL END;
ALTER TABLE sources ALTER COLUMN auth_required TYPE BOOLEAN
    USING CASE WHEN auth_required = 1 THEN TRUE ELSE FALSE END;
ALTER TABLE feed_items ALTER COLUMN is_primary TYPE BOOLEAN
    USING CASE WHEN is_primary = 1 THEN TRUE WHEN is_primary = 0 THEN FALSE ELSE NULL END;
ALTER TABLE quality_scores ALTER COLUMN freshness_decay_applied TYPE BOOLEAN
    USING CASE WHEN freshness_decay_applied = 1 THEN TRUE ELSE FALSE END;

-- Convert TEXT timestamps to TIMESTAMPTZ
ALTER TABLE topics ALTER COLUMN last_scraped TYPE TIMESTAMPTZ USING last_scraped::timestamptz;
ALTER TABLE topics ALTER COLUMN created_at TYPE TIMESTAMPTZ USING created_at::timestamptz;
ALTER TABLE topics ALTER COLUMN updated_at TYPE TIMESTAMPTZ USING updated_at::timestamptz;
-- (Repeat for all timestamp columns across all tables...)

-- UUID columns use native UUID type
ALTER TABLE sources ALTER COLUMN id TYPE UUID USING id::uuid;
ALTER TABLE feed_items ALTER COLUMN id TYPE UUID USING id::uuid;
ALTER TABLE feed_items ALTER COLUMN source_id TYPE UUID USING source_id::uuid;
ALTER TABLE feed_items ALTER COLUMN duplicate_group_id TYPE UUID USING duplicate_group_id::uuid;
-- (Repeat for all UUID columns...)

-- ============================================================
-- POSTGRESQL: Advanced Indexes
-- ============================================================

-- GIN indexes for JSONB columns (fast JSON containment queries)
CREATE INDEX idx_sources_config_gin ON sources USING GIN(config);
CREATE INDEX idx_feed_items_tags_gin ON feed_items USING GIN(tags);
CREATE INDEX idx_feed_items_raw_metadata_gin ON feed_items USING GIN(raw_metadata);

-- Partial indexes for common filtered queries
CREATE INDEX idx_sources_active_high_priority ON sources(priority DESC)
    WHERE status = 'active';
CREATE INDEX idx_feed_items_publishable ON feed_items(topic, quality_score DESC, freshness_score DESC)
    WHERE status IN ('new', 'processed') AND is_primary IS NOT FALSE;
CREATE INDEX idx_scrape_jobs_recent_failures ON scrape_jobs(source_id, created_at DESC)
    WHERE status = 'failed';

-- Full-text search (Portuguese configuration example)
ALTER TABLE feed_items ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(summary, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(array_to_string(tags::text[], ' '), '')), 'C')
    ) STORED;
CREATE INDEX idx_feed_items_fts ON feed_items USING GIN(search_vector);

-- Triggers become functions in Postgres
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_topics_updated_at BEFORE UPDATE ON topics
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER trg_sources_updated_at BEFORE UPDATE ON sources
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER trg_feed_items_updated_at BEFORE UPDATE ON feed_items
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER trg_scrape_jobs_updated_at BEFORE UPDATE ON scrape_jobs
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER trg_quality_scores_updated_at BEFORE UPDATE ON quality_scores
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER trg_digest_entries_updated_at BEFORE UPDATE ON digest_entries
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER trg_duplicate_groups_updated_at BEFORE UPDATE ON duplicate_groups
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Partitioning recommendation for feed_items at scale (> 1M rows)
-- CREATE TABLE feed_items_partitioned (LIKE feed_items INCLUDING ALL)
-- PARTITION BY RANGE (discovered_date);
```

---

## 9. Score Tier Mapping

| Tier | Score Range | Label       | Action                                |
|------|-------------|-------------|---------------------------------------|
| 1    | 80-100      | Excellent   | Auto-publish, breaking digest alert   |
| 2    | 60-79       | Good        | Include in daily digest               |
| 3    | 40-59       | Average     | Include in weekly digest only         |
| 4    | 20-39       | Below Avg   | Store but exclude from digests        |
| 5    | 0-19        | Poor        | Auto-reject unless manually flagged   |

---

## 10. Status State Machines

### Source Status Transitions
```
pending_verification --> active (manual approval or auto-verify)
pending_verification --> rejected (manual rejection)
active --> paused (manual pause)
active --> error (3+ consecutive failures)
error --> paused (auto-pause after threshold)
error --> active (manual retry after fix)
paused --> active (manual resume)
paused --> error (retry fails again)
```

### FeedItem Status Transitions
```
new --> processed (scoring complete)
new --> rejected (auto-reject: low score / excluded)
processed --> reviewed (human review)
processed --> published (auto-publish: high score)
reviewed --> published (approved)
reviewed --> rejected (rejected by reviewer)
published --> archived (age > retention period)
```

### ScrapeJob Status Transitions
```
queued --> running (worker picks up)
queued --> cancelled (manual cancel)
running --> success (complete)
running --> failed (error, no retries left)
running --> retrying (error, retries remain)
retrying --> success (retry succeeds)
retrying --> failed (all retries exhausted)
```

### Digest Status Transitions
```
draft --> queued (scheduled for send)
queued --> sending (send in progress)
sending --> sent (all channels delivered)
sending --> partially_sent (some channels failed)
sending --> failed (all channels failed)
draft --> cancelled (manual cancel)
```

---

## 11. Data Retention & Archival

| Table              | Retention | Archival Action                                    |
|--------------------|-----------|----------------------------------------------------|
| feed_items         | 90 days   | Move full_text to cold storage; keep metadata      |
| quality_scores     | 90 days   | Archive to parquet; keep latest for each item      |
| scrape_jobs        | 30 days   | Summarize stats; keep last 100 per source          |
| digest_entries     | 365 days  | Keep rendered content; archive old formats         |
| source_health_log  | 30 days   | Aggregate daily; keep error events 90 days         |
| duplicate_groups   | 90 days   | Clean up resolved groups with 2+ items             |

---

## 12. ER Diagram (Text)

```
+-------------+       +--------------+       +----------------+
|   topics    |<-----+|   sources    |<-----+|   feed_items   |
+-------------+       +--------------+       +----------------+
      |                                            |    |
      |                                            |    |
      v                                            v    v
+-------------+       +------------------+    +-----------------+
|digest_entries       |  scrape_jobs     |    | quality_scores  |
+-------------+       +------------------+    +-----------------+
                                                       |
                                                       v
                                              +----------------+
                                              |duplicate_groups|
                                              +----------------+

+-------------+
|source_health|
|    _log     |
+-------------+
```

**Relationships:**
- `topics` 1:N `sources` (cascade delete)
- `topics` 1:N `feed_items` (cascade delete)
- `topics` 1:N `scrape_jobs` (cascade delete)
- `topics` 1:N `digest_entries` (cascade delete)
- `sources` 1:N `feed_items` (cascade delete)
- `sources` 1:N `scrape_jobs` (cascade delete)
- `sources` 1:N `source_health_log` (cascade delete)
- `feed_items` 1:1 `quality_scores` (cascade delete)
- `feed_items` N:1 `duplicate_groups` (set null on delete)
- `feed_items` 1:N `quality_scores.duplicate_of` (set null on delete)
