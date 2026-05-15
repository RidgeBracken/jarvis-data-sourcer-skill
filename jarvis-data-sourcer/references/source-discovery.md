# Jarvis HQ Source Discovery Strategy & Topic Registries

> **Document Purpose**: This file defines the systematic approach for discovering high-quality news sources for any topic, and provides concrete example registries for 14 tracked topics in the Jarvis HQ system.
>
> **Status**: All source URLs marked `needs_verification` unless explicitly confirmed.
>
> **Last Updated**: 2025-06


## Table of Contents

- Section C: Source Discovery Strategy
  - C.1 Discovery Workflow — 10-Step Process
  - C.2 Source Type Priority Matrix
  - C.3 Source Validation Checklist
  - C.4 Red Flags — Sources to Reject or Flag
- Section H: Example Source Registries
  - H.1 OpenAI / Codex
  - H.2 Anthropic / Claude
  - H.3 Google / Gemini
  - H.4 Nvidia
  - H.5 Microsoft
  - H.6 Apple
  - H.7 Tesla
  - H.8 SpaceX
  - H.9 Xbox
  - H.10 Pokemon
  - H.11 Anime
  - H.12 Gaming
  - H.13 UFC
  - H.14 World News
- Appendix: Source Verification Quick Reference
  - Confirmed RSS Feeds (Verified During Research)
  - Common Feed Discovery Patterns
  - GitHub Release API Pattern
  - YouTube Channel RSS Pattern
  - Reddit Subreddit RSS Pattern

---


---

## Section C: Source Discovery Strategy

### C.1 Discovery Workflow — 10-Step Process

A repeatable, systematic workflow for finding high-quality sources on any topic. Designed for autonomous agent execution with human review gates.

#### Step 1: Identify Official Entities
Map out the key organisations, companies, products, and governing bodies for the topic.

| Action | Details |
|--------|---------|
| Search query pattern | `"{topic} official"`, `"{topic} company"`, `"{topic} developer"` |
| What to record | Company names, product names, parent organisations, key subsidiaries |
| Example (OpenAI) | OpenAI (company), Codex (product), ChatGPT (product), OpenAI API (platform) |
| Output | List of 3-10 official entities with their relationships |

#### Step 2: Find Official Website and Blog
Locate the primary web presence for each entity.

| Action | Details |
|--------|---------|
| Search query pattern | `"{entity} official website"`, `"{entity} blog"` |
| What to look for | Root domain, `/blog` path, about page, footer links |
| Validation hint | Check WHOIS data, look for SSL certificates, verify social media bio links |
| Red flags | Unrelated domains, parked pages, missing HTTPS |
| Output | Verified root URL + blog URL for each entity |

#### Step 3: Look for RSS/Atom Feeds
Discover machine-readable feeds from official sources.

| Action | Details |
|--------|---------|
| Search query pattern | `"{entity} RSS feed"`, `"{entity} Atom feed"`, `site:{domain} rss` |
| Common feed locations | `/rss.xml`, `/feed`, `/blog/rss`, `/news/rss`, in `<head>` `<link rel="alternate" type="application/rss+xml">` |
| Tools to use | Feed readers (Feedly), browser inspect source, online RSS finders |
| Output | List of feed URLs with type (RSS 2.0, Atom, JSON Feed) |

#### Step 4: Check Developer Resources (Docs, Changelogs, GitHub)
Technical sources often update before official announcements.

| Action | Details |
|--------|---------|
| Search query pattern | `"{entity} developer docs"`, `"{entity} changelog"`, `"{entity} GitHub"` |
| What to look for | docs.{domain}, developers.{domain}, github.com/{org}, `/changelog` paths |
| Key resource types | API changelogs, release notes, GitHub releases, engineering blogs |
| Output | Developer portal URLs, GitHub org/repos, changelog URLs |

#### Step 5: Find Press/News Room
Official press releases are the most authoritative source type.

| Action | Details |
|--------|---------|
| Search query pattern | `"{entity} press room"`, `"{entity} newsroom"`, `"{entity} press releases"` |
| Common paths | `/newsroom`, `/press`, `/news`, `/press-releases`, `/media` |
| Output | Press room URL, press release archive URL |

#### Step 6: Check Official Social Channels
Social media often has the fastest updates (public posts only — no scraping requiring authentication).

| Action | Details |
|--------|---------|
| Search query pattern | `"{entity} Twitter"`, `"{entity} X"`, `"{entity} YouTube channel"` |
| Platforms to check | X/Twitter (public profiles), YouTube (public channels), LinkedIn (company pages), Instagram (public accounts) |
| What to record | Handle/username, follower count (for quality signal), posting frequency |
| ⚠️ Important | Only use publicly accessible posts. No authentication-required scraping. |
| Output | List of verified social handles per platform |

#### Step 7: Identify Reputable News Outlets Covering the Topic
Find mainstream outlets with dedicated coverage beats.

| Action | Details |
|--------|---------|
| Search query pattern | `"{topic} news site:{reputable-domain}"`, `"{topic} coverage {news-outlet}"` |
| Quality signals | Dedicated beat reporter, consistent coverage history, editorial standards |
| Cross-reference | Check the outlet's topic/tag pages for coverage frequency |
| Output | 5-15 reputable outlets with topic-specific URLs/sections |

#### Step 8: Find Specialist/Industry Publications
Trade publications and niche sites often have deeper coverage.

| Action | Details |
|--------|---------|
| Search query pattern | `"{topic} industry publication"`, `"{topic} trade magazine"`, `"{topic} specialist blog"` |
| Quality signals | Domain expertise, recognised by industry insiders, cited by mainstream outlets |
| Output | 3-10 specialist publications with RSS/feed URLs if available |

#### Step 9: Check for Public APIs
Some sources offer structured data APIs that are more reliable than scraping.

| Action | Details |
|--------|---------|
| Search query pattern | `"{entity} API"`, `"{entity} developer API"`, `site:{domain} api docs` |
| What to look for | REST APIs, GraphQL endpoints, official developer portals |
| Cost check | Verify free tier availability, rate limits, authentication requirements |
| Output | API base URL, documentation URL, free tier limits, auth type |

#### Step 10: Validate All Discovered Sources
Run every discovered source through the validation checklist (C.3).

| Action | Details |
|--------|---------|
| Automated checks | HTTP status 200, feed validity, SSL certificate, last update within 90 days |
| Manual checks | Content quality, attribution, primary vs. secondary, spam signals |
| Record keeping | Document all sources with status: `verified`, `needs_verification`, `rejected` |
| Output | Validated source registry ready for configuration |

---

### C.2 Source Type Priority Matrix

Ranked guide for selecting and implementing source types. Priority 1 = highest, 10 = lowest.

| Priority | Source Type | When to Use | Quality | Technical Approach | Rate Limit Notes |
|----------|-------------|-------------|---------|-------------------|------------------|
| **1** | **Official Blog** | Always the first source to configure for any topic | Very High | RSS/Atom feed preferred; sitemap fallback; HTML scrape last resort | Respect `robots.txt`; 1 request/minute typical |
| **2** | **Press/News Room** | Official announcements, product launches, financial results | Very High | RSS if available; sitemap; structured HTML | Usually no rate limits; static pages |
| **3** | **GitHub Releases** | Software products, open-source projects, developer tools | Very High | GitHub API (`/repos/{owner}/{repo}/releases`); RSS feed; Atom | 60 req/hour unauthenticated; 5000/hour authenticated |
| **4** | **RSS/Atom Feed** | Any source that publishes a feed — highest signal-to-noise ratio | High (source-dependent) | Direct feed polling; FeedBin/Feedly aggregation | Poll every 15-60 min; check `Cache-Control` headers |
| **5** | **Documentation Changelog** | API changes, feature rollouts, version updates | High | RSS if available; structured HTML scraping; sitemap | Static pages; low frequency |
| **6** | **Official Social Media (public)** | Breaking news, real-time updates, event coverage | Medium-High | Official APIs, platform-provided embeds/RSS, or explicitly permitted exports; YouTube RSS | Social APIs often require auth; skip sources when terms disallow collection |
| **7** | **Official YouTube Channel** | Video announcements, keynotes, demos, earnings calls | Medium-High | YouTube channel RSS (via `https://www.youtube.com/feeds/videos.xml?channel_id={ID}`) | Generous rate limits; use channel ID not username |
| **8** | **Newsletter (public archives)** | Deep analysis, curated content not published elsewhere | Medium | Web archive scraping; RSS from newsletter platforms (Substack, Mailchimp RSS) | Check robots.txt; respect archive rate limits |
| **9** | **Reputable News Outlet** | Context, analysis, third-party perspective | Medium-High | RSS feeds; official APIs (NewsAPI, etc.); sitemaps | Varies by outlet; some block scrapers |
| **10** | **Specialist Industry Site** | Niche details, technical depth, community sentiment | Medium | RSS; sitemap; curated HTML scraping | Usually permissive; 1 req/minute |
| **11** | **Public API** | Structured data, bulk updates, programmatic access | High | REST/GraphQL polling; webhook if available | Strict rate limits; check docs |
| **12** | **Community Forum (curated)** | User sentiment, bug reports, unofficial announcements | Medium-Low | RSS for subreddits (`reddit.com/r/{sub}/.rss`); Stack Exchange RSS | Reddit: 30 req/minute; use OAuth for higher |
| **13** | **Official Podcasts** | Interviews, deep dives, earnings call audio | Medium | RSS feed (podcast feeds are standard RSS) | Standard RSS polling |
| **14** | **Search Results (secondary)** | Discovery of new sources only — not for ongoing monitoring | Low | Search API (SerpAPI, Bing Web Search API, Google Custom Search) | Strict API quotas; paid tiers required |

#### Source Type Selection Quick Reference

```
Is there an official entity?
  ├─ Yes → Start with: Official Blog (1) → Press Room (2) → GitHub (3)
  │         → Social Media (6) → YouTube (7)
  │
  ├─ Is it a software/tech product?
  │   ├─ Yes → Add: Documentation Changelog (5) → GitHub Releases (3) → Public API (11)
  │
  ├─ Is it a hardware/consumer product?
  │   ├─ Yes → Add: News Outlets (9) → Specialist Sites (10) → YouTube (7)
  │
  ├─ Is it a sports/entertainment topic?
  │   ├─ Yes → Add: Community Forums (12) → News Outlets (9) → YouTube (7)
  │
  └─ For all topics:
            → Validate all sources (Step 10)
            → Monitor for feed changes monthly
```

---

### C.3 Source Validation Checklist

Run every discovered source through these 12 criteria before adding it to the system.

| # | Criterion | Pass Criteria | How to Check |
|---|-----------|---------------|--------------|
| 1 | **Feed/API Available** | Source has RSS, Atom, JSON Feed, or public API | Visit `/rss.xml`, `/feed`, check `<head>` for alternate links, search `{domain} rss` |
| 2 | **Actively Updated** | At least one new item within the last 90 days | Check feed `lastBuildDate`, latest post date, or sitemap `lastmod` |
| 3 | **Primary Source** | Content originates here, not republished from elsewhere | Check bylines, "via" attributions, cross-reference with other sources |
| 4 | **Attribution Present** | Articles have clear author names, dates, and source labels | Sample 3-5 articles; verify dates are real, authors exist |
| 5 | **Spam-Free** | No auto-generated filler content, no excessive ads, no SEO spam | Read 3 full articles; check for coherent structure; look for AI-generated slop signals |
| 6 | **HTTPS Enabled** | Site uses valid SSL certificate | Browser padlock icon or `curl -I https://{url}` |
| 7 | **Legitimate Domain** | Domain is owned by the claimed entity | WHOIS lookup, match with official social media bio links, press kit |
| 8 | **Consistent Formatting** | Feed items have predictable structure (title, date, content/summary) | Parse 5 feed items; check all required fields present |
| 9 | **No Paywall on Feed** | Feed content is readable without authentication | Subscribe to feed in reader; check if full content or truncated |
| 10 | **Reasonable Rate Limit** | Source allows polling every 15-60 minutes without blocking | Start conservative (1 req/hour); monitor for 403/429 responses |
| 11 | **Content Relevance** | At least 70% of items are on-topic for the registry topic | Sample 20 items; count on-topic vs. off-topic |
| 12 | **No Legal/ToS Issues** | Source explicitly permits access, or is clearly public | Check `robots.txt`, Terms of Service, public RSS availability implies consent |

**Scoring:**
- Score 12/12: **Gold tier** — Add immediately
- Score 10-11/12: **Silver tier** — Add with monitoring
- Score 7-9/12: **Bronze tier** — Add as secondary only
- Score < 7/12: **Reject** — Do not add

---

### C.4 Red Flags — Sources to Reject or Flag

Reject sources that exhibit any of these characteristics:

| # | Red Flag | Why It Matters | Action |
|---|----------|---------------|--------|
| 1 | **No publication dates** on articles | Cannot determine recency or relevance | **Reject** |
| 2 | **No author attribution** ("Staff Writer" consistently) | Likely content farm or AI-generated slop | **Reject** |
| 3 | **Requires authentication** for basic access | Violates the public-source principle | **Skip unless API key freely available** |
| 4 | **Aggressive anti-bot measures** (CAPTCHA on every request, heavy Cloudflare) | Technical barrier; potential ToS violation | **Skip** |
| 5 | **Content copied verbatim** from other sources | Not a primary source; duplicate content | **Reject** |
| 6 | **More ads than content** / auto-playing video | Low-quality signal; scraping noise | **Reject** |
| 7 | **Domain registered < 6 months ago** with high volume | Potential spam farm | **Flag for review** |
| 8 | **HTTPS not available** (HTTP only) | Security risk; sign of unmaintained site | **Flag for review** |
| 9 | **Feed returns errors** or invalid XML for > 7 days | Source is unmaintained or broken | **Mark broken; retry in 30 days** |
| 10 | **Rate limits at > 1 hour intervals** | Impractical for timely news | **Deprioritise; use as backup only** |
| 11 | **Only republishes press releases** with no original content | Duplicate signal; no added value | **Use only if original source unavailable** |
| 12 | **Conspiracy theories, unsubstantiated claims, no fact-checking** | Misinformation risk | **Reject** |
| 13 | **Paywall on feed** or feed redirects to paywalled content | Cannot access content | **Skip unless public archive available** |

---


## Section H: Example Source Registries

> **⚠️ IMPORTANT**: All sources in this section are marked `needs_verification` unless explicitly confirmed through the research process. URLs listed as `expected_url_pattern` should be validated by visiting them before use. Some URLs have been verified during research and are noted as such. Never blindly trust URLs — always verify.

---

### H.1 OpenAI / Codex

```yaml
topic: "OpenAI / Codex"
description: "Official OpenAI updates, Codex releases, API changes, model announcements"
official_entity: "OpenAI"
source_types_to_discover:
  - type: "official_blog"
    search_queries: ["OpenAI blog", "OpenAI news"]
    expected_url_pattern: "openai.com/news"
    rss_feed: "openai.com/news/rss.xml"
    verified: true
    notes: "Official RSS feed confirmed active at openai.com/news/rss.xml"

  - type: "github_releases"
    repos: ["openai/openai-python", "openai/codex"]
    expected_url_pattern: "github.com/openai"
    verified: true
    notes: "OpenAI has multiple public repos; openai-python and codex are key ones"

  - type: "changelog"
    expected_url_pattern: "platform.openai.com/docs/changelog"
    search_queries: ["OpenAI API changelog"]
    notes: "API changelog for developers; may require login for full access"

  - type: "press_room"
    search_queries: ["OpenAI press", "OpenAI newsroom"]
    notes: "OpenAI does not appear to have a separate press room; /news serves both purposes"

  - type: "youtube"
    channels: ["OpenAI"]
    search_queries: ["OpenAI YouTube channel"]
    expected_url_pattern: "youtube.com/@OpenAI"
    verified: true

  - type: "twitter/x"
    handles: ["@OpenAI"]
    verified: true
    notes: "Official account; collect only through official APIs, permitted embeds/exports, or manual verification when platform terms allow"

  - type: "developer_docs"
    expected_url_pattern: "platform.openai.com/docs"
    search_queries: ["OpenAI developer documentation"]
    verified: true

  - type: "news_outlets"
    outlets: ["TechCrunch AI", "The Verge AI", "Ars Technica AI"]
    rss_feeds: ["techcrunch.com/category/ai/feed", "theverge.com/ai-artificial-intelligence/rss/index.xml"]
    notes: "Secondary sources for context and analysis"

  - type: "reddit_community"
    subreddits: ["r/OpenAI", "r/LocalLLaMA"]
    rss_available: true
    notes: "reddit.com/r/OpenAI/.rss for public feed"

  - type: "newsletter"
    newsletters: ["Import AI", "The Batch (Deeplearning.ai)", "TLDR AI"]
    notes: "Third-party newsletters with strong OpenAI coverage"

  - type: "research"
    expected_url_pattern: "openai.com/news/research"
    search_queries: ["OpenAI research RSS"]
    notes: "Research publications may not have a dedicated RSS; community-maintained feed exists at openrss.org/openai.com/news"

  - type: "community_rss"
    source: "OpenRSS (community-maintained)"
    url: "openrss.org/openai.com/news"
    notes: "Community-maintained RSS feed; useful if official feed goes down"

status: "partially_verified"
notes: |
  OpenAI RSS at openai.com/news/rss.xml is CONFIRMED active (was down in mid-2024, restored late 2024).
  OpenAI removed their old blog/rss.xml during a site redesign; the new /news/rss.xml is the canonical feed.
  GitHub repos are verified public. docs.anthropic.com pattern is confirmed.
  Research page RSS may need a community workaround (see Olshansk/rss-feeds on GitHub).
```

---

### H.2 Anthropic / Claude

```yaml
topic: "Anthropic / Claude"
description: "Anthropic AI updates, Claude model releases, API changes, safety research"
official_entity: "Anthropic"
source_types_to_discover:
  - type: "official_blog"
    search_queries: ["Anthropic blog", "Anthropic news"]
    expected_url_pattern: "anthropic.com/news"
    verified: true
    notes: "Canonical source — every model release and policy update posted here"

  - type: "rss_feed"
    search_queries: ["Anthropic RSS feed", "Anthropic news RSS"]
    expected_url_pattern: "anthropic.com/news/rss"
    notes: "Feed exists for /news section; verify exact URL"

  - type: "engineering_blog"
    search_queries: ["Anthropic engineering blog"]
    expected_url_pattern: "anthropic.com/engineering"
    notes: "Engineering blog exists at /engineering; may have separate RSS from /news"

  - type: "changelog"
    expected_url_pattern: "docs.anthropic.com/en/release-notes"
    search_queries: ["Anthropic API changelog", "Claude changelog"]
    verified: true
    notes: "API release notes at docs.anthropic.com — covers API changes, new parameters, deprecations"

  - type: "github_releases"
    repos: ["anthropics/anthropic-sdk-python", "anthropics/anthropic-sdk-typescript"]
    expected_url_pattern: "github.com/anthropics"
    verified: true

  - type: "developer_docs"
    expected_url_pattern: "docs.anthropic.com"
    verified: true
    notes: "Comprehensive API documentation with changelogs"

  - type: "youtube"
    channels: ["Anthropic"]
    search_queries: ["Anthropic YouTube"]

  - type: "twitter/x"
    handles: ["@AnthropicAI"]
    verified: true
    notes: "Fast signal for releases; model announcements get threads here"

  - type: "discord"
    server: "Anthropic"
    channels: ["#announcements", "#api-and-sdk"]
    notes: "Discord announcements often come before news page; requires Discord account"

  - type: "news_outlets"
    outlets: ["TechCrunch AI", "The Verge AI", "Ars Technica AI"]

  - type: "reddit_community"
    subreddits: ["r/ClaudeAI"]
    rss_available: true

  - type: "newsletter"
    newsletters: ["Import AI", "The Batch", "TLDR AI"]

status: "partially_verified"
notes: |
  Four primary sources cover nearly all Anthropic updates:
  1. anthropic.com/news (canonical news RSS)
  2. docs.anthropic.com (API changelog)
  3. Discord #announcements channel
  4. @AnthropicAI on X/Twitter
  Engineering blog at /engineering is separate from /news and may have its own feed.
  GitHub org "anthropics" confirmed with public SDK repos.
```

---

### H.3 Google / Gemini

```yaml
topic: "Google / Gemini"
description: "Google AI updates, Gemini model releases, DeepMind research, Google Cloud AI"
official_entity: "Google (Alphabet)"
subsidiaries: ["Google DeepMind", "Google AI", "Google Cloud"]
source_types_to_discover:
  - type: "official_blog"
    search_queries: ["Google AI blog", "Gemini updates"]
    expected_url_pattern: "blog.google/innovation-and-ai"
    rss_feed: "blog.google/rss"
    verified: true
    notes: "Main Google blog at blog.google with AI section; RSS confirmed active"

  - type: "deepmind_blog"
    search_queries: ["Google DeepMind blog RSS"]
    expected_url_pattern: "deepmind.google/discover/blog"
    verified: true
    notes: "DeepMind has its own blog separate from main Google blog"

  - type: "research_blog"
    search_queries: ["Google Research blog RSS"]
    expected_url_pattern: "research.google/blog"
    rss_feed: "research.google/blog/rss"
    verified: true
    notes: "Academic research blog with RSS"

  - type: "developers_blog"
    search_queries: ["Google Developers blog"]
    expected_url_pattern: "developers.googleblog.com"
    rss_feed: null
    verified: true
    notes: "Developer-focused announcements; feed URL needs verification because the original draft had a truncated FeedBurner pattern"

  - type: "google_cloud_blog"
    search_queries: ["Google Cloud blog AI"]
    expected_url_pattern: "cloud.google.com/blog"
    rss_feed: "cloudblog.withgoogle.com/rss"
    verified: true

  - type: "changelog"
    expected_url_pattern: "ai.google.dev/gemini-api/docs/changelog"
    search_queries: ["Gemini API changelog"]
    notes: "Gemini API changelog for developers"

  - type: "github_releases"
    repos: ["google/generative-ai-sdk-python", "google/gemma"]
    expected_url_pattern: "github.com/google"
    verified: true

  - type: "youtube"
    channels: ["Google", "Google DeepMind"]
    search_queries: ["Google AI YouTube channel"]
    verified: true

  - type: "twitter/x"
    handles: ["@GoogleAI", "@DeepMind", "@GeminiApp"]
    verified: true

  - type: "news_outlets"
    outlets: ["TechCrunch Google AI", "The Verge Google", "9to5Google"]
    rss_feeds: ["9to5google.com/feed"]

  - type: "reddit_community"
    subreddits: ["r/GoogleGeminiAI"]

  - type: "newsletter"
    newsletters: ["Import AI", "The Batch", "Last Week in AI"]

status: "partially_verified"
notes: |
  Google has MULTIPLE official blogs — key ones:
  - blog.google (main, with AI/Gemini section)
  - deepmind.google/discover/blog (DeepMind specific)
  - research.google/blog (research publications)
  - developers.googleblog.com (developer tools)
  - cloud.google.com/blog (cloud/enterprise AI)
  Each has its own RSS feed. No single unified Google AI RSS exists.
  GitHub org "google" is massive; focus on generative-ai-sdk-python and gemma repos.
```

---

### H.4 Nvidia

```yaml
topic: "Nvidia"
description: "Nvidia GPU releases, AI hardware, data center products, GTC announcements, DLSS, drivers"
official_entity: "NVIDIA Corporation"
source_types_to_discover:
  - type: "official_blog"
    search_queries: ["Nvidia blog RSS"]
    expected_url_pattern: "blogs.nvidia.com"
    rss_feed: "feeds.feedburner.com/nvidiablog"
    verified: true
    notes: "Main NVIDIA blog uses FeedBurner; confirmed active"

  - type: "newsroom"
    search_queries: ["Nvidia newsroom RSS"]
    expected_url_pattern: "nvidianews.nvidia.com"
    rss_feed: "nvidianews.nvidia.com/rss.xml"
    verified: true
    notes: "Press releases/newsroom with dedicated RSS"

  - type: "developer_blog"
    search_queries: ["Nvidia developer blog RSS"]
    expected_url_pattern: "developer.nvidia.com/blog"
    rss_feed: "developer.nvidia.com/blog/feed"
    verified: true
    notes: "Developer-focused technical blog with RSS"

  - type: "changelog"
    expected_url_pattern: "nvidia.com/drivers"
    search_queries: ["Nvidia driver release notes", "Nvidia DLSS changelog"]
    notes: "Driver release notes; may not have RSS"

  - type: "github_releases"
    repos: ["NVIDIA/cuda-samples", "NVIDIA/TensorRT"]
    expected_url_pattern: "github.com/NVIDIA"
    verified: true

  - type: "youtube"
    channels: ["NVIDIA", "NVIDIA Developers"]
    verified: true

  - type: "twitter/x"
    handles: ["@NVIDIA", "@NVIDIAAI", "@NVIDIAGeForce"]
    verified: true

  - type: "rss_feeds_directory"
    url: "nvidia.com/en-us/about-nvidia/rss"
    verified: true
    notes: "NVIDIA maintains an RSS directory page listing all their feeds (press room, blog, developer blog)"

  - type: "news_outlets"
    outlets: ["Tom's Hardware Nvidia", "AnandTech", "VideoCardz", "Wccftech"]
    rss_feeds: ["tomshardware.com/news/nvidia/rss"]

  - type: "reddit_community"
    subreddits: ["r/nvidia", "r/NVIDIAHelp"]
    rss_available: true

  - type: "investor_relations"
    expected_url_pattern: "investor.nvidia.com"
    search_queries: ["Nvidia investor relations RSS"]
    notes: "Earnings calls, financial results; may have RSS or email alerts"

  - type: "newsletter"
    newsletters: ["NVIDIA AI Newsletter"]
    notes: "Nvidia may offer newsletter subscriptions"

status: "partially_verified"
notes: |
  NVIDIA has an exceptionally well-organised RSS ecosystem:
  - Main blog: feeds.feedburner.com/nvidiablog (CONFIRMED)
  - Press room: nvidianews.nvidia.com/rss.xml (CONFIRMED)
  - Developer blog: developer.nvidia.com/blog/feed (CONFIRMED)
  - RSS directory page: nvidia.com/en-us/about-nvidia/rss (CONFIRMED)
  Multiple YouTube channels and Twitter accounts for different product lines.
  GitHub org "NVIDIA" confirmed with many public repos.
```

---

### H.5 Microsoft

```yaml
topic: "Microsoft"
description: "Microsoft product updates, AI/Copilot news, Azure, Windows, Xbox, Surface, earnings"
official_entity: "Microsoft Corporation"
source_types_to_discover:
  - type: "official_blog"
    search_queries: ["Microsoft blog RSS", "Microsoft official blog"]
    expected_url_pattern: "blogs.microsoft.com"
    verified: true
    notes: "Main Microsoft blog; covers company-wide announcements"

  - type: "ai_blog"
    search_queries: ["Microsoft AI blog RSS"]
    expected_url_pattern: "blogs.microsoft.com/ai"
    rss_feed: "blogs.microsoft.com/ai/feed"
    verified: true
    notes: "Dedicated AI blog with WordPress RSS feed"

  - type: "news_hub"
    search_queries: ["Microsoft news hub"]
    expected_url_pattern: "news.microsoft.com"
    verified: true
    notes: "Official news aggregation hub"

  - type: "tech_community_blog"
    search_queries: ["Microsoft TechCommunity blog"]
    expected_url_pattern: "techcommunity.microsoft.com"
    notes: "Community + official blogs; may have RSS"

  - type: "azure_blog"
    search_queries: ["Azure blog RSS"]
    expected_url_pattern: "azure.microsoft.com/blog"
    notes: "Azure-specific updates; check for RSS"

  - type: "developer_docs"
    expected_url_pattern: "learn.microsoft.com"
    search_queries: ["Microsoft Learn what's new"]
    verified: true

  - type: "github_releases"
    repos: ["microsoft/vscode", "microsoft/TypeScript", "microsoft/DeepSpeed"]
    expected_url_pattern: "github.com/microsoft"
    verified: true
    notes: "Microsoft GitHub org is one of the largest on the platform"

  - type: "youtube"
    channels: ["Microsoft"]
    verified: true

  - type: "twitter/x"
    handles: ["@Microsoft", "@MSFTAzure", "@MicrosoftAI"]
    verified: true

  - type: "news_outlets"
    outlets: ["The Verge Microsoft", "Ars Technica Microsoft", "Windows Central", "Thurrott.com"]
    rss_feeds: ["theverge.com/microsoft/rss", "windowscentral.com/rss"]

  - type: "reddit_community"
    subreddits: ["r/microsoft", "r/Windows11"]
    rss_available: true

  - type: "newsletter"
    newsletters: ["Microsoft Weekly (Thurrott)", "Windows Intelligence"]

  - type: "source_blog"
    search_queries: ["Microsoft Source blog"]
    expected_url_pattern: "news.microsoft.com/source"
    rss_feed: "news.microsoft.com/source/topics/ai/feed"
    verified: true
    notes: "Microsoft Source is the enterprise/IT pro news hub with topic-specific RSS feeds"

status: "partially_verified"
notes: |
  Microsoft has a distributed blog ecosystem:
  - blogs.microsoft.com (main corporate blog)
  - blogs.microsoft.com/ai (AI-specific, with RSS)
  - news.microsoft.com/source (enterprise news with topic RSS feeds)
  - techcommunity.microsoft.com (community + official)
  - azure.microsoft.com/blog (Azure specific)
  GitHub org "microsoft" confirmed — thousands of public repos.
  X/Twitter handles verified for @Microsoft, @MSFTAzure, @MicrosoftAI.
```

---

### H.6 Apple

```yaml
topic: "Apple"
description: "Apple product announcements, iOS/macOS updates, AI/ML features, earnings, events"
official_entity: "Apple Inc."
source_types_to_discover:
  - type: "newsroom"
    search_queries: ["Apple newsroom RSS"]
    expected_url_pattern: "apple.com/newsroom"
    rss_feed: "apple.com/newsroom/rss-feed.rss"
    verified: true
    notes: "Official newsroom with RSS; primary source for product announcements"

  - type: "developer_news"
    search_queries: ["Apple developer news RSS"]
    expected_url_pattern: "developer.apple.com/news"
    rss_feed: "developer.apple.com/news/rss"
    verified: true
    notes: "Developer-focused news and updates with RSS"

  - type: "security_blog"
    search_queries: ["Apple security blog"]
    expected_url_pattern: "security.apple.com/blog"
    notes: "Security research blog; check for RSS"

  - type: "rss_directory"
    expected_url_pattern: "apple.com/rss"
    verified: true
    notes: "Apple maintains an RSS directory page listing all available feeds"

  - type: "youtube"
    channels: ["Apple"]
    verified: true
    notes: "Official Apple YouTube channel for keynotes and ads"

  - type: "twitter/x"
    handles: ["@Apple"]
    verified: true

  - type: "investor_relations"
    expected_url_pattern: "investor.apple.com"
    search_queries: ["Apple investor relations RSS"]
    notes: "Earnings calls, SEC filings; may have email alerts"

  - type: "news_outlets"
    outlets: ["MacRumors", "9to5Mac", "AppleInsider", "Daring Fireball", "Ars Technica Apple"]
    rss_feeds: ["macrumors.com/feed", "9to5mac.com/feed", "appleinsider.com/rss"]
    verified: true
    notes: "Apple specialist press; all have confirmed RSS feeds"

  - type: "reddit_community"
    subreddits: ["r/apple", "r/ios", "r/mac"]
    rss_available: true

  - type: "newsletter"
    newsletters: ["MacStories", "Daring Fireball"]
    notes: "MacStories and Daring Fireball are prominent Apple-focused newsletters with RSS"

status: "partially_verified"
notes: |
  Apple's RSS ecosystem is well-organised:
  - apple.com/newsroom/rss-feed.rss (CONFIRMED — main newsroom feed)
  - developer.apple.com/news/rss (CONFIRMED — developer news)
  - apple.com/rss (CONFIRMED — RSS directory page listing all feeds)
  Apple does NOT have a traditional blog. The Newsroom IS their blog equivalent.
  Security blog at security.apple.com/blog may be separate.
  Third-party Apple press (MacRumors, 9to5Mac, AppleInsider) all have RSS and are
  extremely reliable — sometimes more timely than Apple's official channels.
  YouTube channel confirmed for keynotes and promotional videos.
```

---

### H.7 Tesla

```yaml
topic: "Tesla"
description: "Tesla vehicle updates, FSD, energy products, earnings, Elon Musk statements"
official_entity: "Tesla, Inc."
source_types_to_discover:
  - type: "official_blog"
    search_queries: ["Tesla blog RSS"]
    expected_url_pattern: "tesla.com/blog"
    notes: "Tesla has a blog at /blog; RSS availability unknown — verify"

  - type: "news_releases"
    search_queries: ["Tesla news releases RSS", "Tesla IR RSS"]
    expected_url_pattern: "ir.tesla.com"
    notes: "Investor relations page; may have RSS or email alerts"

  - type: "investor_relations"
    expected_url_pattern: "ir.tesla.com/news"
    search_queries: ["Tesla investor relations news"]
    notes: "Earnings calls, shareholder letters, SEC filings"

  - type: "youtube"
    channels: ["Tesla"]
    verified: true

  - type: "twitter/x"
    handles: ["@Tesla", "@ElonMusk"]
    verified: true
    notes: "Elon Musk's personal account often has Tesla news before official channels"

  - type: "github_releases"
    repos: ["teslamotors"]
    expected_url_pattern: "github.com/teslamotors"
    notes: "Tesla has some open-source projects; not a primary news source"

  - type: "news_outlets"
    outlets: ["Electrek Tesla", "Teslarati", "InsideEVs Tesla", "Not a Tesla App"]
    rss_feeds: ["electrek.co/guides/tesla/feed", "teslarati.com/feed"]
    verified: true
    notes: "Third-party Tesla press; Electrek and Teslarati are primary specialist sources"

  - type: "reddit_community"
    subreddits: ["r/teslamotors", "r/TeslaLounge"]
    rss_available: true

  - type: "software_update_tracker"
    source: "Not a Tesla App"
    url: "notateslaapp.com"
    rss_feed: "notateslaapp.com/rss"
    verified: true
    notes: "Excellent third-party tracker for Tesla software updates and release notes"

  - type: "newsletter"
    newsletters: ["Electrek Daily"]
    notes: "Electrek has a daily newsletter with Tesla coverage"

status: "needs_verification"
notes: |
  Tesla does NOT have a strong first-party RSS ecosystem:
  - /blog exists but RSS is unconfirmed
  - IR site (ir.tesla.com) has news but may not have RSS
  - Primary news flow is via X/Twitter (@Tesla and @ElonMusk)
  - Third-party sources (Electrek, Teslarati, Not a Tesla App) are ESSENTIAL
    for comprehensive Tesla coverage — they often break news first.
  Not a Tesla App (notateslaapp.com) has RSS and tracks software updates.
  Must verify: tesla.com/blog RSS, ir.tesla.com RSS availability.
```

---

### H.8 SpaceX

```yaml
topic: "SpaceX"
description: "SpaceX launches, Starship development, Starlink, NASA partnerships, Elon Musk statements"
official_entity: "SpaceX"
source_types_to_discover:
  - type: "updates_page"
    search_queries: ["SpaceX updates RSS"]
    expected_url_pattern: "spacex.com/updates"
    notes: "SpaceX has an /updates page; RSS availability unknown — verify"

  - type: "launches_page"
    search_queries: ["SpaceX launches"]
    expected_url_pattern: "spacex.com/launches"
    notes: "Launch schedule and mission details; may not have RSS"

  - type: "youtube"
    channels: ["SpaceX"]
    verified: true
    notes: "Major source for launch livestreams and mission videos"

  - type: "twitter/x"
    handles: ["@SpaceX", "@elonmusk"]
    verified: true
    notes: "Primary real-time update channel for SpaceX"

  - type: "news_outlets"
    outlets: ["SpaceNews SpaceX", "NASASpaceFlight", "Teslarati SpaceX", "SpacePolicyOnline"]
    rss_feeds: ["spacenews.com/tag/spacex/feed", "nasaspaceflight.com/?feed=spacex"]
    verified: true
    notes: "NASASpaceFlight and SpaceNews are essential specialist sources"

  - type: "reddit_community"
    subreddits: ["r/spacex", "r/SpaceXLounge"]
    rss_available: true

  - type: "nasa_source"
    search_queries: ["NASA SpaceX blog RSS"]
    expected_url_pattern: "blogs.nasa.gov/spacex"
    rss_feed: "blogs.nasa.gov/spacex/feed"
    verified: true
    notes: "NASA blog for SpaceX partnership missions (Commercial Crew, HLS); RSS confirmed"

  - type: "newsletter"
    newsletters: ["SpaceNews Newsletter", "NASASpaceFlight Updates"]

status: "needs_verification"
notes: |
  SpaceX has a MINIMAL first-party RSS ecosystem:
  - /updates page exists but RSS is unconfirmed
  - No obvious blog or press room with RSS
  - PRIMARY sources are YouTube (launch streams) and X/Twitter (@SpaceX)
  - NASA partnership blog at blogs.nasa.gov/spacex has RSS (CONFIRMED)
  - Third-party specialist press is ESSENTIAL:
    - NASASpaceFlight (leading technical source)
    - SpaceNews (industry trade publication)
    - Teslarati (broader coverage)
  Must verify: spacex.com/updates RSS, spacex.com/launches RSS.
  r/spacex has strict quality standards and is a good community signal.
```

---

### H.9 Xbox

```yaml
topic: "Xbox"
description: "Xbox console updates, Game Pass, Xbox Series X|S, Bethesda, Microsoft Gaming, hardware"
official_entity: "Microsoft Gaming (Xbox division)"
source_types_to_discover:
  - type: "official_blog"
    search_queries: ["Xbox Wire RSS"]
    expected_url_pattern: "news.xbox.com"
    rss_feed: "news.xbox.com/en-us/feed"
    verified: true
    notes: "Xbox Wire is the official Xbox blog; RSS should be available — verify exact URL"

  - type: "news_hub"
    search_queries: ["Xbox news"]
    expected_url_pattern: "news.xbox.com/en-us"
    verified: true
    notes: "Xbox Wire in English; also available in Japanese and other languages"

  - type: "game_pass_news"
    search_queries: ["Xbox Game Pass updates RSS"]
    expected_url_pattern: "news.xbox.com/en-us/tag/xbox-game-pass"
    notes: "Game Pass additions/removals are a key update category"

  - type: "youtube"
    channels: ["Xbox"]
    verified: true
    notes: "Official Xbox YouTube channel for showcases and trailers"

  - type: "twitter/x"
    handles: ["@Xbox", "@XboxGamePass"]
    verified: true

  - type: "developer_docs"
    expected_url_pattern: "developer.microsoft.com/games"
    search_queries: ["Xbox developer documentation"]

  - type: "news_outlets"
    outlets: ["Pure Xbox", "Windows Central Xbox", "Eurogamer Xbox", "TrueAchievements"]
    rss_feeds: ["purexbox.com/rss", "windowscentral.com/rss", "eurogamer.net/feed"]

  - type: "reddit_community"
    subreddits: ["r/xbox", "r/XboxSeriesX", "r/XboxGamePass"]
    rss_available: true

  - type: "newsletter"
    newsletters: ["Xbox Wire Newsletter"]

  - type: "events"
    search_queries: ["Xbox showcase dates"]
    notes: "Xbox holds showcases (Developer_Direct, Partner Preview, Games Showcase); dates announced on Xbox Wire"

status: "needs_verification"
notes: |
  Xbox Wire (news.xbox.com) is the primary official source.
  RSS feed is expected but the exact URL needs verification.
  Try: news.xbox.com/en-us/feed or check the page source for alternate links.
  Xbox is a Microsoft division — some updates may appear on blogs.microsoft.com.
  Microsoft Gaming CEO (Phil Spencer, Sarah Bond) statements on X are newsworthy.
  Third-party specialist Xbox press (Pure Xbox, Windows Central) has strong RSS coverage.
  YouTube showcases are major news events; subscribe to channel RSS.
```

---

### H.10 Pokemon

```yaml
topic: "Pokemon"
description: "Pokemon game releases, TCG, Pokemon GO, anime, events, merchandise"
official_entity: "The Pokemon Company International / Nintendo"
source_types_to_discover:
  - type: "press_site"
    search_queries: ["Pokemon press site"]
    expected_url_pattern: "pokemon.gamespress.com"
    verified: true
    notes: "The Pokemon Company International official press site; may have RSS — verify"

  - type: "press_site_na"
    search_queries: ["Pokemon press releases North America"]
    expected_url_pattern: "press.pokemon.com"
    verified: true
    notes: "North America-specific press site"

  - type: "official_youtube"
    channels: ["Pokemon"]
    expected_url_pattern: "youtube.com/@Pokemon"
    verified: true
    notes: "Official Pokemon YouTube channel; Pokemon Presents events streamed here"

  - type: "twitter/x"
    handles: ["@Pokemon", "@PokemonGoApp", "@PlayPokemon"]
    verified: true
    notes: "Multiple accounts for different aspects of the franchise"

  - type: "pokemon_go_blog"
    search_queries: ["Pokemon GO blog RSS"]
    expected_url_pattern: "pokemongolive.com/post"
    notes: "Pokemon GO has a dedicated blog at pokemongolive.com; may have RSS"

  - type: "nintendo_news"
    search_queries: ["Nintendo Pokemon news"]
    expected_url_pattern: "nintendo.com/whatsnew/pokemon"
    notes: "Nintendo publishes Pokemon game news on their official news channel"

  - type: "tournament_updates"
    search_queries: ["Pokemon World Championships news"]
    expected_url_pattern: "pokemon.com/us/play-pokemon"
    notes: "Play Pokemon series (TCG, VGC) tournament news"

  - type: "news_outlets"
    outlets: ["Serebii.net", "Bulbapedia", "Pokemon.com news"]
    rss_feeds: ["serebii.net/rss.xml"]
    notes: "Serebii.net is the leading Pokemon news source; check for RSS"

  - type: "reddit_community"
    subreddits: ["r/pokemon", "r/PokemonScarletViolet", "r/TheSilphRoad"]
    rss_available: true

  - type: "youtube"
    channels: ["Pokemon"]
    verified: true

  - type: "newsletter"
    newsletters: ["Pokemon Trainer Club Newsletter"]
    notes: "Official newsletter via Pokemon Trainer Club; email-based"

status: "needs_verification"
notes: |
  Pokemon's information ecosystem is distributed:
  - pokemon.gamespress.com (international press site — CONFIRMED)
  - press.pokemon.com (North America press site — CONFIRMED)
  - pokemongolive.com (Pokemon GO-specific blog)
  - nintendo.com (first-party game announcements)
  - Serebii.net is the UNOFFICIAL but extremely reliable specialist source
  Pokemon Presents video presentations are major news events.
  TheSilphRoad subreddit is the best source for Pokemon GO datamines.
  Must verify: RSS availability on press sites and pokemongolive.com.
```

---

### H.11 Anime

```yaml
topic: "Anime"
description: "Anime announcements, season releases, licensing news, simulcast schedules, industry news"
official_entity: "Multiple: Crunchyroll, Funimation (Sony), Sentai Filmworks, Aniplex, etc."
source_types_to_discover:
  - type: "industry_news_site"
    search_queries: ["Anime News Network RSS"]
    expected_url_pattern: "animenewsnetwork.com"
    rss_feed: "animenewsnetwork.com/all/rss.xml"
    verified: true
    notes: "Anime News Network (ANN) is the leading anime news site; RSS confirmed"

  - type: "streaming_service"
    search_queries: ["Crunchyroll news RSS"]
    expected_url_pattern: "crunchyroll.com/news"
    rss_feed: "feeds.feedburner.com/crunchyroll/animenews"
    notes: "Crunchyroll news feed; note — this feed may be intermittently active, verify"

  - type: "official_youtube"
    channels: ["Crunchyroll", "Funimation"]
    verified: true

  - type: "twitter/x"
    handles: ["@AnimeNewsNet", "@Crunchyroll"]

  - type: "seasonal_tracker"
    search_queries: ["LiveChart anime seasonal RSS"]
    expected_url_pattern: "livechart.me"
    notes: "LiveChart is a popular seasonal anime tracker; check for RSS"

  - type: "news_outlets"
    outlets: ["Crunchyroll News", "Anime News Network", "MyAnimeList News", "Otaku USA"]
    rss_feeds: ["animenewsnetwork.com/all/rss.xml"]
    verified: true

  - type: "reddit_community"
    subreddits: ["r/anime", "r/manga", "r/Animedubs"]
    rss_available: true

  - type: "simulcast_schedule"
    search_queries: ["anime simulcast schedule RSS"]
    notes: "Crunchyroll seasonal schedule; may not have RSS"

  - type: "newsletter"
    newsletters: ["Anime News Network Newsletter"]

status: "partially_verified"
notes: |
  Anime news ecosystem:
  - Anime News Network (ANN) is the PRIMARY source; RSS at all/rss.xml (CONFIRMED)
  - Crunchyroll has a news feed via FeedBurner; verify if still active (intermittent issues reported)
  - MyAnimeList has news section; check for RSS
  - Seasonal anime tracking: livechart.me, anichart.net
  - r/anime is a major community hub with scheduled discussion threads
  - Japanese sources (Oricon News Natalie, Mantan Web) for primary Japanese industry news
    — requires Japanese language ability
  Key news types: seasonal announcements, licensing deals, production committee news,
  VA (seiyuu) announcements, BD/DVD sales figures.
```

---

### H.12 Gaming

```yaml
topic: "Gaming"
description: "Video game industry news, releases, reviews, hardware, esports, industry trends"
official_entity: "Multiple: Sony (PlayStation), Microsoft (Xbox), Nintendo, Valve, Epic Games, etc."
source_types_to_discover:
  - type: "industry_news_site"
    search_queries: ["IGN RSS feed", "Kotaku RSS", "Polygon RSS"]
    expected_url_patterns:
      - "ign.com/rss"
      - "kotaku.com/rss"
      - "polygon.com/rss"
    verified: true
    notes: "Major gaming outlets all have RSS feeds; IGN, Kotaku, Polygon confirmed"

  - type: "industry_news_site_2"
    outlets: ["GameSpot", "Eurogamer", "VG247", "PC Gamer", "Rock Paper Shotgun"]
    rss_feeds:
      - "gamespot.com/feeds/mashup"
      - "eurogamer.net/feed"
      - "vg247.com/feed"
      - "pcgamer.com/rss"
      - "rockpapershotgun.com/feed"
    notes: "All major gaming sites have RSS; verify exact URLs"

  - type: "platform_holders"
    search_queries: ["PlayStation Blog RSS", "Nintendo news RSS"]
    expected_url_patterns:
      - "blog.playstation.com/feed"
      - "nintendo.com/whatsnew"
    notes: "Platform holder blogs are primary sources for exclusives"

  - type: "game_awards"
    search_queries: ["The Game Awards news"]
    expected_url_pattern: "thegameawards.com"
    notes: "Annual awards show; significant news source for announcements"

  - type: "youtube"
    channels: ["IGN", "GameSpot", "PlayStation", "Nintendo", "Xbox"]
    verified: true

  - type: "twitter/x"
    handles: ["@IGN", "@GameSpot", "@Nibellion (defunct — find successor)"]
    notes: "Nibellion was the premier gaming news aggregator; account is no longer active. Look for successor accounts."

  - type: "reddit_community"
    subreddits: ["r/games", "r/GamingLeaksAndRumours", "r/pcgaming"]
    rss_available: true

  - type: "esports"
    search_queries: ["esports news RSS"]
    expected_url_patterns:
      - "esportsobserver.com/feed"
      - "dexerto.com/esports/rss"
    notes: "Esports is a sub-domain of gaming; may want separate tracking"

  - type: "newsletter"
    newsletters: ["GameDiscoverCo", "SuperJoost"]
    notes: "Industry analysis newsletters for gaming business trends"

status: "partially_verified"
notes: |
  Gaming news is a HIGH-VOLUME topic. Source strategy:
  1. Primary: IGN + Polygon + Kotaku RSS (high volume, need filtering)
  2. Platform-specific: PlayStation Blog, Xbox Wire, Nintendo News (lower volume, high relevance)
  3. Industry: GameSpot, Eurogamer, VG247 (regional focus)
  4. Community: r/games, r/GamingLeaksAndRumours
  5. YouTube: Subscribe to channel RSS for showcases (Summer Game Fest, State of Play, Direct)
  Nibellion (@Nibellion) on X/Twitter was the definitive gaming news aggregator
  but the account is no longer active. No clear successor has emerged.
  Consider using RSS aggregation + filtering instead.
  Steam News RSS is available for specific games/curators.
```

---

### H.13 UFC

```yaml
topic: "UFC"
description: "UFC events, fight announcements, rankings, fighter news, PPV results, Dana White statements"
official_entity: "Ultimate Fighting Championship (TKO Group Holdings)"
source_types_to_discover:
  - type: "official_news"
    search_queries: ["UFC news RSS", "UFC.com news feed"]
    expected_url_pattern: "ufc.com/news"
    notes: "UFC.com news section; RSS availability unknown — verify"

  - type: "official_event_listings"
    search_queries: ["UFC events schedule"]
    expected_url_pattern: "ufc.com/events"
    notes: "Event listings and fight cards; may have RSS — verify"

  - type: "press_releases"
    search_queries: ["UFC press releases RSS"]
    expected_url_pattern: "ufc.com/press"
    notes: "Press releases for major announcements; check for RSS"

  - type: "rankings"
    search_queries: ["UFC rankings RSS"]
    expected_url_pattern: "ufc.com/rankings"
    notes: "Updated after events; check if dynamic or RSS available"

  - type: "youtube"
    channels: ["UFC"]
    verified: true
    notes: "Fight highlights, Embedded series, weigh-ins, press conferences"

  - type: "twitter/x"
    handles: ["@ufc", "@danawhite", "@bokamotoESPN"]
    notes: "@ufc official; @danawhite for announcements; @bokamotoESPN for breaking news"

  - type: "news_outlets"
    outlets: ["ESPN MMA", "MMA Junkie", "MMA Fighting", "Bloody Elbow", "Cageside Press"]
    rss_feeds: ["espn.com/espn/rss/mma", "mmajunkie.com/news/feed", "mmafighting.com/rss.xml"]
    notes: "MMA specialist press; ESPN is highest-profile mainstream coverage"

  - type: "reddit_community"
    subreddits: ["r/MMA", "r/ufc"]
    rss_available: true

  - type: "podcast"
    source: "UFC Minute Podcast"
    rss_feed: "media.ufc.tv/iTunes_podcasts/rss/ufc_minute_itunes_feed.rss"
    verified: true
    notes: "Official UFC podcast RSS; may be legacy — verify if still updated"

  - type: "fighter_stats"
    search_queries: ["UFC stats API"]
    expected_url_pattern: "ufcstats.com"
    notes: "Fight statistics; unofficial but widely used; check for API"

  - type: "newsletter"
    search_queries: ["UFC newsletter"]
    expected_url_pattern: "ufc.com/newsletter"
    notes: "UFC may offer email newsletter"

status: "needs_verification"
notes: |
  UFC first-party RSS ecosystem is limited:
  - ufc.com/news may not have RSS — must verify
  - ufc.com/events has schedule data but likely no RSS
  - Official podcast RSS exists but may be legacy
  - YouTube is PRIMARY for video content (highlights, Embedded)
  - X/Twitter (@ufc, @danawhite) for announcements
  ESPN MMA is the highest-quality mainstream source.
  MMA Junkie, MMA Fighting, Bloody Elbow are specialist press with RSS.
  r/MMA is the largest combat sports community on Reddit.
  Fighter announcements often break on Ariel Helwani (@arielhelwani) or
  Brett Okamoto (@bokamotoESPN) X accounts before official channels.
  Verify: ufc.com/news RSS, ufc.com/events RSS, podcast feed freshness.
```

---

### H.14 World News

```yaml
topic: "World News"
description: "Global breaking news, international politics, conflicts, diplomacy, major events"
official_entity: "Multiple: Reuters, AP, BBC, AFP, DW, etc."
source_types_to_discover:
  - type: "news_wire"
    search_queries: ["Reuters RSS feed", "AP News RSS", "BBC News RSS"]
    expected_url_patterns:
      - "reuters.com/world/rss"
      - "apnews.com/rss"
      - "feeds.bbci.co.uk/news/rss.xml"
    verified: true
    notes: "All major wire services have RSS feeds; these are PRIMARY sources"

  - type: "international_broadcaster"
    outlets: ["BBC News", "Al Jazeera", "France 24", "DW News", "CNA (Singapore)"]
    rss_feeds:
      - "feeds.bbci.co.uk/news/world/rss.xml"
      - "aljazeera.com/xml/rss/all.xml"
      - "france24.com/en/rss"
      - "dw.com/en/top-stories/s-9097"
    verified: true
    notes: "International broadcasters with global perspective; all have RSS"

  - type: "news_wire_detailed"
    search_queries: ["Reuters world news RSS", "AP world news RSS"]
    expected_url_patterns:
      - "reuters.com/world"
      - "apnews.com/hub/world-news"
    notes: "Reuters and AP have topic-specific RSS feeds"

  - type: "aggregators"
    search_queries: ["Google News RSS", "Apple News RSS"]
    expected_url_patterns:
      - "news.google.com/rss"
    notes: "Google News has an RSS endpoint; useful for keyword-based feeds"

  - type: "no_paywall_sources"
    search_queries: ["news organizations without paywall RSS"]
    sources:
      - "BBC News: feeds.bbci.co.uk/news/rss.xml"
      - "Reuters: reuters.com"
      - "Al Jazeera: aljazeera.com/xml/rss/all.xml"
      - "AP News: apnews.com"
      - "DW News: dw.com"
      - "France 24: france24.com/en/rss"
      - "The Guardian (registration): theguardian.com/rss"
    verified: true
    notes: "These sources do not have hard paywalls; RSS content is accessible"

  - type: "youtube"
    channels: ["BBC News", "Al Jazeera English", "DW News", "FRANCE 24 English"]
    verified: true
    notes: "News channels on YouTube for video coverage"

  - type: "twitter/x"
    handles: ["@Reuters", "@AP", "@BBCWorld", "@BBCBreaking"]
    verified: true
    notes: "Breaking news accounts; high signal-to-noise for major events"

  - type: "reddit_community"
    subreddits: ["r/worldnews", "r/news", "r/internationalnews"]
    rss_available: true

  - type: "newsletter"
    newsletters: ["Reuters Daily Briefing", "BBC News Daily"]
    notes: "Many wire services offer daily email digests"

  - type: "specialised_topics"
    search_queries: ["UN news RSS", "World Bank news RSS"]
    expected_url_patterns:
      - "un.org/rss"
      - "worldbank.org/en/news"
    notes: "International organisation news for policy/diplomatic coverage"

status: "partially_verified"
notes: |
  World news is the MOST RSS-friendly topic in this registry:
  - Reuters, AP, BBC all have well-maintained RSS feeds (CONFIRMED)
  - Al Jazeera, France 24, DW all have RSS (CONFIRMED)
  - No paywall on these sources makes them ideal for automated scraping
  - Google News RSS can be used for keyword-based custom feeds:
    news.google.com/rss/search?q={keyword}
  - Reddit r/worldnews is a major community source
  - For breaking news, @BBCBreaking and @Reuters on X are fastest
  Avoid: paywalled sources (NYT, WaPo) for automated feeds unless you
  have subscription access. Guardian RSS is free but requires free registration.
  Consider filtering: world news volume is extremely high.
  Recommend filtering by region (Americas, Europe, Asia, Middle East)
  or topic (conflict, diplomacy, economics) using feed subsets.
```

---

## Appendix: Source Verification Quick Reference

### Confirmed RSS Feeds (Verified During Research)

| Source | Feed URL | Status |
|--------|----------|--------|
| OpenAI News | `openai.com/news/rss.xml` | CONFIRMED |
| NVIDIA Blog | `feeds.feedburner.com/nvidiablog` | CONFIRMED |
| NVIDIA Press Room | `nvidianews.nvidia.com/rss.xml` | CONFIRMED |
| NVIDIA Developer Blog | `developer.nvidia.com/blog/feed` | CONFIRMED |
| NVIDIA RSS Directory | `nvidia.com/en-us/about-nvidia/rss` | CONFIRMED |
| Google Blog | `blog.google/rss` | CONFIRMED |
| Google Research Blog | `research.google/blog/rss` | CONFIRMED |
| Google Cloud Blog | `cloudblog.withgoogle.com/rss` | CONFIRMED |
| Microsoft AI Blog | `blogs.microsoft.com/ai/feed` | CONFIRMED |
| Microsoft Source (AI) | `news.microsoft.com/source/topics/ai/feed` | CONFIRMED |
| Apple Newsroom | `apple.com/newsroom/rss-feed.rss` | CONFIRMED |
| Apple Developer News | `developer.apple.com/news/rss` | CONFIRMED |
| Apple RSS Directory | `apple.com/rss` | CONFIRMED |
| Xbox Wire | `news.xbox.com/en-us/feed` | NEEDS_VERIFY |
| Anime News Network | `animenewsnetwork.com/all/rss.xml` | CONFIRMED |
| BBC News | `feeds.bbci.co.uk/news/rss.xml` | CONFIRMED |
| BBC World News | `feeds.bbci.co.uk/news/world/rss.xml` | CONFIRMED |
| Al Jazeera | `aljazeera.com/xml/rss/all.xml` | CONFIRMED |
| France 24 | `france24.com/en/rss` | CONFIRMED |
| DW News | `dw.com/en/top-stories/s-9097` | CONFIRMED |
| Reuters | `reuters.com` (RSS available) | CONFIRMED |
| AP News | `apnews.com` (RSS available) | CONFIRMED |
| Fox Sports UFC | `foxsports.com/rss-feeds` (UFC section) | CONFIRMED |
| NASA SpaceX Blog | `blogs.nasa.gov/spacex/feed` | CONFIRMED |
| UFC Minute Podcast | `media.ufc.tv/iTunes_podcasts/rss/ufc_minute_itunes_feed.rss` | VERIFY_FRESHNESS |
| SpaceX Updates | `spacex.com/updates` | NO_RSS_CONFIRMED |

### Common Feed Discovery Patterns

```
# Feed URL patterns to try (in order of likelihood)
1. {domain}/rss.xml
2. {domain}/feed
3. {domain}/feed.xml
4. {domain}/blog/rss
5. {domain}/news/rss
6. {domain}/news/feed
7. feeds.feedburner.com/{name}
8. Check <head> for: <link rel="alternate" type="application/rss+xml" href="...">
9. Check /sitemap.xml for RSS references
10. Use online tool: https://rssbox.herokuapp.com or https://openrss.org
```

### GitHub Release API Pattern

```
# For any GitHub repo, releases are available at:
https://api.github.com/repos/{owner}/{repo}/releases

# Example:
https://api.github.com/repos/openai/openai-python/releases
https://api.github.com/repos/anthropics/anthropic-sdk-python/releases

# Rate limits: 60 req/hour unauthenticated, 5000/hour with token
# RSS/Atom also available:
https://github.com/{owner}/{repo}/releases.atom
```

### YouTube Channel RSS Pattern

```
# YouTube provides RSS feeds for channels:
https://www.youtube.com/feeds/videos.xml?channel_id={CHANNEL_ID}

# To find a channel ID:
1. Visit the YouTube channel
2. View page source
3. Search for "channelId" or "browse_id"
4. Or use: https://commentpicker.com/youtube-channel-id.php
```

### Reddit Subreddit RSS Pattern

```
# Reddit provides RSS for any public subreddit:
https://reddit.com/r/{subreddit}/.rss

# Example:
https://reddit.com/r/OpenAI/.rss
https://reddit.com/r/spacex/.rss

# Rate limit: ~30 requests per minute for unauthenticated
# JSON also available: reddit.com/r/{sub}/new.json
```

---

*End of Jarvis HQ Source Discovery Strategy & Topic Registries*
