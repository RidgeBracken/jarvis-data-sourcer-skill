# Section J: Testing Checklist and Strategy

## Table of Contents
1. [Test Categories](#1-test-categories)
2. [Unit Test Specifications](#2-unit-test-specifications)
3. [Integration Test Specifications](#3-integration-test-specifications)
4. [End-to-End Test Specifications](#4-end-to-end-test-specifications)
5. [Quality Test Specifications](#5-quality-test-specifications)
6. [Compliance Test Specifications](#6-compliance-test-specifications)
7. [Performance Test Specifications](#7-performance-test-specifications)
8. [Test Fixtures](#8-test-fixtures)
9. [Test Infrastructure](#9-test-infrastructure)
10. [Testing Checklist for Codex](#10-testing-checklist-for-codex)

---

## 1. Test Categories

| Category | Test Types | Count Target | Framework |
|----------|-----------|--------------|-----------|
| Unit Tests | Individual functions, adapters, utilities | 50+ | Vitest |
| Integration Tests | Adapter + database, pipeline stages | 20+ | Vitest + testcontainers |
| End-to-End Tests | Full pipeline run | 5+ | Vitest + Docker Compose |
| Quality Tests | Scoring accuracy, deduplication | 10+ | Vitest |
| Compliance Tests | robots.txt, rate limits | 5+ | Vitest |
| Performance Tests | Speed, memory, throughput | 5+ | Vitest + benchmark.js |

### Test Distribution by Component

```
                    Test Distribution
    Unit Tests        [████████████████████████] 50
    Integration Tests [████████████] 20
    Quality Tests     [██████] 10
    E2E Tests         [███] 5
    Compliance Tests  [███] 5
    Performance Tests [███] 5
                      +------------------------+
                      0    10    20    30    40    50
```

---

## 2. Unit Test Specifications

### 2.1 RSS Adapter Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should parse valid RSS 2.0 feed` | Full RSS 2.0 parsing | `valid-rss-2.0.xml` | Array of 3 items with title, link, pubDate | `expect(items).toHaveLength(3); expect(items[0].title).toBe('...')` |
| 2 | `should parse valid Atom feed` | Atom format parsing | `valid-atom.xml` | Array of 2 items with title, link, updated | `expect(items[0].title).toBeDefined(); expect(items[0].link).toMatch(/^http/)` |
| 3 | `should handle empty feed` | Graceful empty handling | `empty-feed.xml` | Empty array, no errors | `expect(items).toHaveLength(0); expect(console.warn).toHaveBeenCalledWith('Empty feed')` |
| 4 | `should handle invalid XML` | XML parsing error recovery | `invalid-xml.xml` | Empty array + error logged | `expect(items).toHaveLength(0); expect(result.error).toBeDefined()` |
| 5 | `should handle 404 response` | HTTP error handling | HTTP 404 status | Empty array + source marked unreachable | `expect(items).toHaveLength(0); expect(source.status).toBe('unreachable')` |
| 6 | `should handle 500 response` | Server error recovery | HTTP 500 status | Empty array + retry scheduled | `expect(items).toHaveLength(0); expect(scheduleRetry).toHaveBeenCalled()` |
| 7 | `should handle large feed (1000+ items)` | Memory-efficient parsing | `large-feed-1000.xml` | All items parsed, memory < 50MB | `expect(items).toHaveLength(1000); expect(heapUsed).toBeLessThan(50 * 1024 * 1024)` |
| 8 | `should extract all standard fields` | Field completeness | `full-rss-item.xml` | Object with title, link, description, author, guid, pubDate, enclosure | `expect(item).toMatchObject(expectedFields)` |
| 9 | `should handle missing pubDate` | Optional field handling | `rss-no-pubdate.xml` | Item with fallback date (now) | `expect(item.pubDate).toBeInstanceOf(Date); expect(item.dateSource).toBe('fallback')` |
| 10 | `should handle relative URLs in links` | URL normalization | `rss-relative-urls.xml` | Absolute URLs resolved against feed URL | `expect(item.link).toBe('https://example.com/full/article.html')` |

### 2.2 API Adapter Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should handle API key authentication` | Bearer token auth | API config with key | Headers include `Authorization: Bearer token123` | `expect(fetch).toHaveBeenCalledWith(url, expect.objectContaining({headers}))` |
| 2 | `should handle pagination (page-based)` | Multi-page fetching | API config with `pageSize: 10` | All pages fetched, items concatenated | `expect(items).toHaveLength(25); expect(fetch).toHaveBeenCalledTimes(3)` |
| 3 | `should handle cursor-based pagination` | Cursor pagination | API with `nextCursor` field | Follows cursors until exhausted | `expect(items).toHaveLength(50); expect(apiCalls).toBeGreaterThan(1)` |
| 4 | `should handle rate limit (429 response)` | Backoff on 429 | HTTP 429 + `Retry-After: 60` | Waits 60 seconds, retries | `expect(setTimeout).toHaveBeenCalledWith(expect.any(Function), 60000)` |
| 5 | `should handle server error (500)` | 500 recovery | HTTP 500 response | Retries with backoff, eventually fails | `expect(retryCount).toBe(3); expect(result.error).toBe('Max retries exceeded')` |
| 6 | `should parse JSON response` | JSON parsing | Valid JSON body | Parsed object with correct types | `expect(item.id).toBeTypeOf('number'); expect(item.title).toBeTypeOf('string')` |
| 7 | `should handle malformed JSON` | JSON error recovery | Invalid JSON body | Error caught, source logged | `expect(result.error).toContain('JSON'); expect(logger.error).toHaveBeenCalled()` |
| 8 | `should apply request timeout` | Timeout handling | Slow API response | Request aborted after timeout | `expect(controller.abort).toHaveBeenCalled(); expect(result.error).toBe('Timeout')` |
| 9 | `should handle OAuth2 flow` | Token refresh | Expired access token | Refreshes token, retries request | `expect(tokenRefresh).toHaveBeenCalled(); expect(items).toHaveLength(5)` |
| 10 | `should validate response schema` | Schema validation | JSON not matching schema | Error with schema violations listed | `expect(result.errors).toContain('Missing required field: title')` |

### 2.3 HTML Article Adapter Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should extract article title` | Title extraction | `article-full.html` | Correct title string | `expect(title).toBe('The Future of AI in Healthcare')` |
| 2 | `should extract author name` | Author extraction | `article-full.html` | Author name or null | `expect(author).toBe('Jane Smith')` |
| 3 | `should extract publication date` | Date extraction | `article-full.html` | ISO date string | `expect(date).toBe('2024-03-15T10:30:00Z')` |
| 4 | `should handle paywall content` | Paywall detection | `article-paywall.html` | `isPaywalled: true`, limited content | `expect(result.isPaywalled).toBe(true); expect(result.content.length).toBeLessThan(500)` |
| 5 | `should handle missing author` | Optional author | `article-no-author.html` | `author: null`, no error | `expect(author).toBeNull(); expect(logger.warn).not.toHaveBeenCalled()` |
| 6 | `should handle missing date` | Optional date | `article-no-date.html` | `date: null`, fallback used | `expect(date).toBeNull(); expect(result.dateSource).toBe('fallback')` |
| 7 | `should extract main content` | Content extraction | `article-full.html` | Clean article text | `expect(content).toContain('artificial intelligence'); expect(content.length).toBeGreaterThan(1000)` |
| 8 | `should remove ads and navigation` | Content cleaning | `article-with-ads.html` | No ad text in content | `expect(content).not.toContain('Advertisement'); expect(content).not.toContain('Subscribe Now')` |
| 9 | `should handle JavaScript-rendered pages` | SPA rendering | `article-spa.html` | Content after JS execution | `expect(content.length).toBeGreaterThan(200); expect(title).toBeDefined()` |
| 10 | `should handle encoding issues` | Encoding detection | `article-utf8.html`, `article-latin1.html` | Correctly decoded text | `expect(content).toContain('café'); expect(content).toContain('naïve')` |

### 2.4 GitHub Release Adapter Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should parse release list` | Release parsing | `github-releases.json` | Array of releases with version, notes, date | `expect(releases[0].version).toBe('v2.1.0'); expect(releases).toHaveLength(3)` |
| 2 | `should handle repository with no releases` | Empty releases | `[]` (empty array) | Empty array + info log | `expect(releases).toHaveLength(0); expect(logger.info).toHaveBeenCalledWith('No releases')` |
| 3 | `should handle API rate limit (403)` | Rate limit detection | HTTP 403 + `X-RateLimit-Remaining: 0` | Error + retry scheduled with reset time | `expect(error.code).toBe('RATE_LIMITED'); expect(retryAfter).toBe(timestamp)` |
| 4 | `should parse release assets` | Asset extraction | Release with assets | Asset names, sizes, download URLs | `expect(assets[0].name).toBe('binary-linux-amd64'); expect(assets[0].size).toBe(1024000)` |
| 5 | `should extract changelog from release body` | Changelog parsing | Markdown release body | Structured changelog sections | `expect(changelog.features).toContain('Added dark mode'); expect(changelog.fixes).toHaveLength(2)` |
| 6 | `should handle pre-release flag` | Pre-release detection | `prerelease: true` | `isPreRelease: true` in output | `expect(result.isPreRelease).toBe(true); expect(result.stability).toBe('beta')` |
| 7 | `should handle GraphQL API errors` | GraphQL error handling | `{"errors": [...]}` | Error extracted from response | `expect(error.message).toContain('Field not found'); expect(result.data).toBeUndefined()` |
| 8 | `should handle non-existent repository` | 404 for repo | HTTP 404 | Empty result + source marked bad | `expect(result.items).toHaveLength(0); expect(source.status).toBe('invalid')` |

### 2.5 Quality Scorer Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should calculate source score for tier-1 source` | High-tier source | Source with `tier: 'tier-1'`, high reputation | Score >= 80 | `expect(score).toBeGreaterThanOrEqual(80); expect(score).toBeLessThanOrEqual(100)` |
| 2 | `should calculate source score for tier-3 source` | Low-tier source | Source with `tier: 'tier-3'`, low reputation | Score between 20-50 | `expect(score).toBeGreaterThanOrEqual(20); expect(score).toBeLessThanOrEqual(50)` |
| 3 | `should calculate content score for long article` | Content length bonus | Article with 5000+ words | Length bonus applied | `expect(score).toBeGreaterThan(baseScore); expect(bonus).toBe(10)` |
| 4 | `should calculate content score for short snippet` | Content length penalty | Article with < 100 words | Length penalty applied | `expect(score).toBeLessThan(baseScore); expect(penalty).toBe(-15)` |
| 5 | `should apply freshness decay for 1-day-old item` | Recent freshness | Item from 1 day ago | Minimal decay | `expect(decayFactor).toBeCloseTo(0.95, 1); expect(scored.freshness).toBe('high')` |
| 6 | `should apply freshness decay for 7-day-old item` | Week-old item | Item from 7 days ago | Moderate decay | `expect(decayFactor).toBeCloseTo(0.60, 1); expect(scored.freshness).toBe('medium')` |
| 7 | `should apply freshness decay for 30-day-old item` | Month-old item | Item from 30 days ago | Heavy decay | `expect(decayFactor).toBeCloseTo(0.20, 1); expect(scored.freshness).toBe('low')` |
| 8 | `should apply breaking news boost` | Breaking news detection | Item with breaking keywords | +25 boost applied | `expect(scored.score).toBe(baseScore + 25); expect(scored.isBreaking).toBe(true)` |
| 9 | `should apply primary source boost` | Primary source bonus | Official blog post vs re-report | +15 boost | `expect(scored.score).toBeGreaterThan(reportedScore); expect(scored.sourceType).toBe('primary')` |
| 10 | `should apply expert author bonus` | Author reputation | Article by known expert | +10 boost | `expect(scored.score).toBe(baseScore + 10); expect(scored.authorTier).toBe('expert')` |
| 11 | `should cap score at 100` | Maximum score | Item with all bonuses exceeding 100 | Score capped at 100 | `expect(scored.score).toBe(100); expect(scored.score).not.toBeGreaterThan(100)` |
| 12 | `should floor score at 0` | Minimum score | Item with all penalties below 0 | Score floored at 0 | `expect(scored.score).toBe(0); expect(scored.score).not.toBeLessThan(0)` |

### 2.6 Deduplication Engine Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should match exact URL duplicates` | Exact URL match | Two items with same URL | Second marked as duplicate | `expect(result.isDuplicate).toBe(true); expect(result.primaryId).toBe(item1.id)` |
| 2 | `should match similar titles (>90%)` | Title similarity | Two items with 95% title match | Marked as duplicate | `expect(similarity).toBeGreaterThan(0.90); expect(result.isDuplicate).toBe(true)` |
| 3 | `should not flag different titles (<70%)` | Title dissimilarity | Two items with 50% title match | Not duplicate | `expect(similarity).toBeLessThan(0.70); expect(result.isDuplicate).toBe(false)` |
| 4 | `should match similar content (>85%)` | Content similarity | Two items with 90% content overlap | Marked as duplicate | `expect(result.isDuplicate).toBe(true); expect(result.matchType).toBe('content')` |
| 5 | `should respect time window (within 24h)` | Time window check | Two items 12 hours apart, same content | Duplicate within window | `expect(result.isDuplicate).toBe(true); expect(result.hoursApart).toBe(12)` |
| 6 | `should ignore old items outside window` | Time window exclusion | Two items 72 hours apart | Not duplicate (outside window) | `expect(result.isDuplicate).toBe(false); expect(result.reason).toBe('outside_time_window')` |
| 7 | `should keep higher-quality item as primary` | Quality-based primary | Two duplicates, one higher quality | Higher quality kept as primary | `expect(primary.qualityScore).toBeGreaterThan(duplicate.qualityScore)` |
| 8 | `should merge metadata from duplicates` | Metadata merge | Two duplicates with different metadata | Merged metadata in primary | `expect(primary.tags).toContain('tag-from-item2'); expect(primary.sources).toHaveLength(2)` |
| 9 | `should handle near-duplicate detection` | Near-duplicate fuzzy match | Items with minor text changes | Detected with confidence score | `expect(result.confidence).toBeGreaterThan(0.80); expect(result.confidence).toBeLessThan(1.0)` |
| 10 | `should not flag different articles on same topic` | False positive prevention | Two distinct articles about same topic | Not duplicate | `expect(result.isDuplicate).toBe(false); expect(result.similarity).toBeLessThan(0.60)` |

### 2.7 Date Normalizer Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should parse ISO 8601 date` | ISO format | `'2024-03-15T10:30:00Z'` | Date object at correct time | `expect(date.toISOString()).toBe('2024-03-15T10:30:00.000Z')` |
| 2 | `should parse RFC 2822 date` | RFC 2822 format | `'Fri, 15 Mar 2024 10:30:00 GMT'` | Date object | `expect(date.getTime()).toBe(1710498600000)` |
| 3 | `should parse RSS pubDate` | RSS format | `'Mon, 15 Mar 2024 10:30:00 +0000'` | Date object | `expect(date.getUTCDate()).toBe(15); expect(date.getUTCMonth()).toBe(2)` |
| 4 | `should parse relative date strings` | Relative format | `'2 hours ago'`, `'yesterday'` | Date object relative to now | `expect(hoursAgo).toBeWithinRange(now - 7200000, now - 7100000)` |
| 5 | `should handle timezone conversion` | TZ handling | `'2024-03-15T10:30:00-05:00'` | UTC normalized date | `expect(date.toISOString()).toBe('2024-03-15T15:30:00.000Z')` |
| 6 | `should handle invalid date string` | Invalid date | `'not a date'` | `null` with warning | `expect(date).toBeNull(); expect(logger.warn).toHaveBeenCalled()` |
| 7 | `should handle ambiguous US/EU formats` | Ambiguous format | `'03/04/2024'` | Config-preferential parsing | `expect(date).toEqual(expectedDate); expect(config.dateFormat).toBe('US')` |
| 8 | `should handle Unix timestamp (seconds)` | Unix timestamp | `1710498600` | Date object | `expect(date.toISOString()).toBe('2024-03-15T10:30:00.000Z')` |
| 9 | `should handle Unix timestamp (milliseconds)` | Millisecond timestamp | `1710498600000` | Date object | `expect(date.toISOString()).toBe('2024-03-15T10:30:00.000Z')` |
| 10 | `should return null for empty input` | Empty handling | `''`, `null`, `undefined` | `null` | `expect(date).toBeNull(); expect(logger.warn).not.toHaveBeenCalled()` |

### 2.8 URL Validator Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should validate HTTPS URL` | Valid HTTPS | `'https://example.com/article'` | Valid result | `expect(result.valid).toBe(true); expect(result.protocol).toBe('https')` |
| 2 | `should validate HTTP URL` | Valid HTTP | `'http://example.com/article'` | Valid with warning | `expect(result.valid).toBe(true); expect(result.warning).toBe('Insecure protocol')` |
| 3 | `should reject invalid protocol` | Bad protocol | `'ftp://example.com/file'` | Invalid | `expect(result.valid).toBe(false); expect(result.error).toBe('Unsupported protocol')` |
| 4 | `should reject malformed URL` | Malformed | `'not a url at all'` | Invalid | `expect(result.valid).toBe(false); expect(result.error).toBe('Malformed URL')` |
| 5 | `should follow redirects` | Redirect handling | URL returning 301 | Final URL returned | `expect(result.url).toBe('https://final-url.com/article'); expect(result.redirects).toBe(1)` |
| 6 | `should detect redirect loops` | Loop detection | Circular redirects | Error | `expect(result.valid).toBe(false); expect(result.error).toBe('Redirect loop detected')` |
| 7 | `should resolve relative URL` | Relative resolution | `'/article/123'` with base | Absolute URL | `expect(result.url).toBe('https://example.com/article/123')` |
| 8 | `should reject URLs on blocklist` | Blocklist check | URL matching blocklist pattern | Invalid | `expect(result.valid).toBe(false); expect(result.error).toBe('Domain on blocklist')` |
| 9 | `should normalize URL` | URL normalization | URL with query params, fragments | Clean normalized URL | `expect(result.url).toBe('https://example.com/article'); expect(result.normalized).toBe(true)` |
| 10 | `should extract domain` | Domain extraction | Any valid URL | Domain string | `expect(result.domain).toBe('example.com'); expect(result.tld).toBe('.com')` |

### 2.9 Rate Limiter Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should respect requests per minute limit` | RPM compliance | Config: 10 req/min, 15 requests | First 10 succeed, 11th+ delayed | `expect(delayMs).toBeGreaterThan(0); expect(await Promise.all(results)).toHaveLength(15)` |
| 2 | `should respect requests per second limit` | RPS compliance | Config: 2 req/sec, 5 requests | Batches of 2 per second | `expect(timeElapsed).toBeGreaterThanOrEqual(2000)` |
| 3 | `should queue requests when limit reached` | Queueing | 20 requests with 10 req/min limit | All processed over 2+ minutes | `expect(queueLength).toBe(0); expect(processedCount).toBe(20)` |
| 4 | `should handle burst with token bucket` | Token bucket | Config: bucket size 5, 10 requests | First 5 immediate, rest throttled | `expect(immediateCount).toBe(5); expect(throttledCount).toBe(5)` |
| 5 | `should handle different limits per domain` | Per-domain limits | 2 domains with different limits | Correct limit applied per domain | `expect(domainA.delay).not.toBe(domainB.delay)` |
| 6 | `should reset limit after window expires` | Window reset | Request after window passes | No delay after window | `expect(delayAfterReset).toBe(0)` |
| 7 | `should cancel queued requests on abort` | Cancellation | Abort signal triggered | Pending requests rejected | `expect(rejected).toBe(true); expect(error.name).toBe('AbortError')` |
| 8 | `should track concurrent request count` | Concurrency limit | Config: max 3 concurrent | Max 3 active at any time | `expect(concurrentMax).toBeLessThanOrEqual(3)` |

### 2.10 Retry Handler Tests

| # | Test Name | What It Tests | Input Fixture | Expected Output | Assertion |
|---|-----------|---------------|---------------|-----------------|-----------|
| 1 | `should succeed on first attempt` | No retry needed | Function returning success | Result on first call | `expect(attempts).toBe(1); expect(result).toBe('success')` |
| 2 | `should retry with exponential backoff` | Exponential backoff | Function failing twice, then succeeding | 3 attempts with increasing delays | `expect(attempts).toBe(3); expect(delays).toEqual([1000, 2000])` |
| 3 | `should respect max retries limit` | Max retries | Function always failing | Fails after max retries | `expect(attempts).toBe(4); expect(error).toBe('Max retries exceeded')` |
| 4 | `should send to dead letter queue` | Dead letter | Function failing permanently | Item in dead letter queue | `expect(dlq).toContain(item); expect(dlq[0].error).toBeDefined()` |
| 5 | `should not retry on non-retryable error` | Non-retryable | 404 Not Found | Fails immediately | `expect(attempts).toBe(1); expect(error).toContain('Not Found')` |
| 6 | `should apply jitter to backoff` | Jitter | Multiple retries | Delays vary (not exact multiples) | `expect(hasJitter).toBe(true); expect(maxDelay).toBeLessThan(expectedMax * 1.5)` |
| 7 | `should reset backoff on success` | Backoff reset | Intermittent failures | Backoff resets between unrelated calls | `expect(retryDelays[2]).toBe(baseDelay); // reset after success` |
| 8 | `should handle async errors` | Async rejection | Promise rejection | Caught and retried | `expect(attempts).toBeGreaterThan(1); expect(result).toBe('success')` |

---

## 3. Integration Test Specifications

### 3.1 RSS → Database Integration

```javascript
describe('RSS → Database Integration', () => {
  it('should store all items from RSS feed', async () => {
    // Setup
    const feedUrl = 'https://test-feed.com/rss.xml';
    const mockFeed = await loadFixture('valid-rss-2.0.xml');
    server.use(http.get(feedUrl, () => HttpResponse.xml(mockFeed)));

    // Execute
    const result = await rssAdapter.collect({ url: feedUrl, topic: 'ai' });
    await storageService.saveItems(result.items);

    // Assert
    const stored = await db.select().from(items).where(eq(items.sourceUrl, feedUrl));
    expect(stored).toHaveLength(3);
    expect(stored[0].title).toBe('AI Breakthrough in 2024');
    expect(stored[0].status).toBe('raw');
  });

  it('should handle duplicate items idempotently', async () => {
    const feedUrl = 'https://test-feed.com/rss.xml';
    const mockFeed = await loadFixture('valid-rss-2.0.xml');
    server.use(http.get(feedUrl, () => HttpResponse.xml(mockFeed)));

    // Run twice
    const r1 = await rssAdapter.collect({ url: feedUrl, topic: 'ai' });
    await storageService.saveItems(r1.items);
    const r2 = await rssAdapter.collect({ url: feedUrl, topic: 'ai' });
    await storageService.saveItems(r2.items);

    const stored = await db.select().from(items);
    expect(stored).toHaveLength(3); // No duplicates
  });

  it('should update existing items on re-scrape', async () => {
    // Store initial
    const initial = createItem({ title: 'Old Title', updatedAt: yesterday });
    await storageService.saveItems([initial]);

    // Updated feed
    const updatedFeed = await loadFixture('rss-updated-item.xml');
    server.use(http.get(feedUrl, () => HttpResponse.xml(updatedFeed)));

    const result = await rssAdapter.collect({ url: feedUrl, topic: 'ai' });
    await storageService.saveItems(result.items);

    const stored = await db.select().from(items).where(eq(items.guid, initial.guid));
    expect(stored[0].title).toBe('Updated Title');
    expect(stored[0].version).toBe(2);
  });
});
```

### 3.2 HTML → Database Integration

```javascript
describe('HTML → Database Integration', () => {
  it('should extract and store full article', async () => {
    const articleUrl = 'https://tech-blog.com/ai-article';
    const mockHtml = await loadFixture('article-full.html');
    server.use(http.get(articleUrl, () => HttpResponse.html(mockHtml)));

    const result = await htmlAdapter.collect({ url: articleUrl, topic: 'ai' });
    await storageService.saveItems([result]);

    const stored = await db.select().from(items).where(eq(items.url, articleUrl));
    expect(stored[0].title).toBe('The Future of AI in Healthcare');
    expect(stored[0].author).toBe('Jane Smith');
    expect(stored[0].content.length).toBeGreaterThan(1000);
    expect(stored[0].status).toBe('raw');
  });

  it('should handle paywall articles gracefully', async () => {
    const articleUrl = 'https://paywalled-site.com/premium';
    const mockHtml = await loadFixture('article-paywall.html');
    server.use(http.get(articleUrl, () => HttpResponse.html(mockHtml)));

    const result = await htmlAdapter.collect({ url: articleUrl, topic: 'ai' });
    expect(result.isPaywalled).toBe(true);
    expect(result.content.length).toBeLessThan(500);
    expect(result.status).toBe('paywalled');
  });
});
```

### 3.3 Quality Scoring Pipeline Integration

```javascript
describe('Quality Scoring Pipeline', () => {
  it('should score raw item and update database', async () => {
    // Insert raw item
    const rawItem = await insertRawItem({
      title: 'Major AI Announcement',
      source: 'official-blog.google.com',
      content: 'Google today announced a major breakthrough...'.repeat(50),
      publishedAt: new Date(),
      url: 'https://blog.google/ai-breakthrough'
    });

    // Run scoring pipeline
    await qualityPipeline.process([rawItem]);

    // Verify scored
    const scored = await db.select().from(items).where(eq(items.id, rawItem.id));
    expect(scored[0].qualityScore).toBeGreaterThanOrEqual(70);
    expect(scored[0].sourceScore).toBeGreaterThanOrEqual(80);
    expect(scored[0].contentScore).toBeGreaterThanOrEqual(60);
    expect(scored[0].status).toBe('scored');
  });

  it('should apply breaking news boost correctly', async () => {
    const rawItem = await insertRawItem({
      title: 'BREAKING: AI Regulation Passed by Congress',
      source: 'reuters.com',
      publishedAt: new Date(),
    });

    await qualityPipeline.process([rawItem]);

    const scored = await db.select().from(items).where(eq(items.id, rawItem.id));
    expect(scored[0].isBreaking).toBe(true);
    expect(scored[0].breakingBoost).toBe(25);
    expect(scored[0].qualityScore).toBeGreaterThan(80);
  });

  it('should decay scores for old items', async () => {
    const oldItem = await insertRawItem({
      title: 'Old AI News',
      publishedAt: new Date(Date.now() - 14 * 86400000), // 14 days ago
    });

    const baseScore = 80;
    const scored = await qualityPipeline.score(oldItem);
    expect(scored.freshnessDecay).toBeLessThan(1);
    expect(scored.qualityScore).toBeLessThan(baseScore);
  });
});
```

### 3.4 Deduplication Pipeline Integration

```javascript
describe('Deduplication Pipeline', () => {
  it('should mark duplicate and keep primary', async () => {
    // Insert primary item
    const primary = await insertItem({
      url: 'https://example.com/ai-news',
      title: 'AI Breakthrough Announced Today',
      content: 'Researchers have made a significant breakthrough...',
      qualityScore: 85,
      publishedAt: new Date(),
    });

    // Insert duplicate (lower quality)
    const duplicate = await insertItem({
      url: 'https://aggregator.com/same-story',
      title: 'AI Breakthrough Announced Today',
      content: 'Researchers have made a significant breakthrough...',
      qualityScore: 45,
      publishedAt: new Date(Date.now() + 3600000), // 1 hour later
    });

    await dedupPipeline.process([primary, duplicate]);

    const updatedPrimary = await db.select().from(items).where(eq(items.id, primary.id));
    const updatedDuplicate = await db.select().from(items).where(eq(items.id, duplicate.id));

    expect(updatedDuplicate[0].status).toBe('duplicate');
    expect(updatedDuplicate[0].primaryId).toBe(primary.id);
    expect(updatedPrimary[0].status).toBe('active');
  });

  it('should merge sources list for duplicates', async () => {
    const primary = await insertItem({ sources: ['reuters.com'] });
    const duplicate = await insertItem({
      sources: ['techcrunch.com'],
      title: primary.title,
      content: primary.content,
    });

    await dedupPipeline.process([primary, duplicate]);

    const merged = await db.select().from(items).where(eq(items.id, primary.id));
    expect(merged[0].sources).toContain('reuters.com');
    expect(merged[0].sources).toContain('techcrunch.com');
  });
});
```

### 3.5 Digest Generation Integration

```javascript
describe('Digest Generation', () => {
  it('should generate formatted digest from items', async () => {
    const items = await insertScoredItems([
      { title: 'AI News 1', score: 95, topic: 'ai' },
      { title: 'AI News 2', score: 80, topic: 'ai' },
      { title: 'Non-AI News', score: 70, topic: 'crypto' },
    ]);

    const digest = await digestService.generate({
      topic: 'ai',
      items: items.filter(i => i.topic === 'ai'),
      format: 'markdown',
    });

    expect(digest).toContain('# AI Daily Digest');
    expect(digest).toContain('AI News 1');
    expect(digest).toContain('AI News 2');
    expect(digest).not.toContain('Non-AI News');
    expect(digest).toContain('## Top Stories');
  });

  it('should sort items by score descending', async () => {
    const items = [
      { title: 'Medium', score: 70 },
      { title: 'Highest', score: 95 },
      { title: 'Lowest', score: 50 },
    ];

    const digest = await digestService.generate({ items, format: 'markdown' });
    const highestIndex = digest.indexOf('Highest');
    const mediumIndex = digest.indexOf('Medium');
    const lowestIndex = digest.indexOf('Lowest');

    expect(highestIndex).toBeLessThan(mediumIndex);
    expect(mediumIndex).toBeLessThan(lowestIndex);
  });
});
```

### 3.6 Telegram Formatting Integration

```javascript
describe('Telegram Formatting', () => {
  it('should format items for Telegram', async () => {
    const item = {
      title: 'AI Breakthrough!',
      url: 'https://example.com/ai',
      summary: 'A major breakthrough in AI research.',
      score: 92,
    };

    const formatted = await telegramFormatter.format(item);

    expect(formatted).toContain('*AI Breakthrough!*');
    expect(formatted).toContain('[Read more](https://example.com/ai)');
    expect(formatted).toContain('A major breakthrough');
    expect(formatted).toContain('Score: 92');
  });

  it('should escape Markdown special characters', async () => {
    const item = {
      title: 'AI *Special* Report [v2.0]',
      summary: 'Cost: $100+ per user.',
    };

    const formatted = await telegramFormatter.format(item);

    expect(formatted).toContain('AI \\*Special\\* Report');
    expect(formatted).toContain('\\[v2\\.0\\]');
    expect(formatted).toContain('\\$100\\+');
  });

  it('should respect Telegram message length limit', async () => {
    const longItem = {
      title: 'Very Long Title',
      summary: 'A'.repeat(5000),
    };

    const formatted = await telegramFormatter.format(longItem);

    expect(formatted.length).toBeLessThanOrEqual(4096);
  });
});
```

---

## 4. End-to-End Test Specifications

### 4.1 Full Scrape of Test Topic

```javascript
describe('E2E: Full Topic Scrape', () => {
  it('should collect, score, dedup, and store items', async () => {
    // Mock RSS feed
    server.use(
      http.get('https://test-rss.com/feed.xml', () =>
        HttpResponse.xml(await loadFixture('valid-rss-2.0.xml'))
      )
    );

    // Mock HTML articles
    server.use(
      http.get('https://test-blog.com/*', () =>
        HttpResponse.html(await loadFixture('article-full.html'))
      )
    );

    // Run full pipeline for topic 'ai'
    await pipeline.run({ topic: 'ai' });

    // Verify database state
    const storedItems = await db.select().from(items).where(eq(items.topic, 'ai'));
    expect(storedItems.length).toBeGreaterThan(0);

    // All items scored
    expect(storedItems.every(i => i.qualityScore > 0)).toBe(true);

    // No duplicate active items
    const activeItems = storedItems.filter(i => i.status === 'active');
    const urls = activeItems.map(i => i.url);
    expect(new Set(urls).size).toBe(urls.length);

    // All items have required fields
    expect(storedItems.every(i => i.title && i.url && i.publishedAt)).toBe(true);
  }, 30000); // 30 second timeout
});
```

### 4.2 Full Pipeline: Collection → Storage → API

```javascript
describe('E2E: Collection to API', () => {
  it('should serve collected items via API', async () => {
    // Run collection
    const collected = await collectionService.collectTopic('ai');
    expect(collected.length).toBeGreaterThan(0);

    // Process pipeline
    await pipeline.process(collected);

    // Query API
    const response = await request(app)
      .get('/api/v1/items?topic=ai')
      .expect(200);

    expect(response.body.items.length).toBeGreaterThan(0);
    expect(response.body.items[0]).toHaveProperty('title');
    expect(response.body.items[0]).toHaveProperty('qualityScore');
    expect(response.body.items[0]).toHaveProperty('summary');
  });
});
```

### 4.3 Error Recovery Flow

```javascript
describe('E2E: Error Recovery', () => {
  it('should retry failed source and eventually succeed', async () => {
    let callCount = 0;
    const feedUrl = 'https://flaky-source.com/feed.xml';

    server.use(
      http.get(feedUrl, () => {
        callCount++;
        if (callCount < 3) {
          return new HttpResponse(null, { status: 500 });
        }
        return HttpResponse.xml(await loadFixture('valid-rss-2.0.xml'));
      })
    );

    const result = await retryHandler.execute(
      () => rssAdapter.collect({ url: feedUrl, topic: 'ai' }),
      { maxRetries: 3, backoff: 'exponential' }
    );

    expect(callCount).toBe(3); // Two failures + one success
    expect(result.items).toHaveLength(3);
    expect(result.error).toBeUndefined();
  });

  it('should send permanently failed items to dead letter queue', async () => {
    server.use(
      http.get('https://broken-source.com/feed.xml', () =>
        new HttpResponse(null, { status: 404 })
      )
    );

    await expect(
      retryHandler.execute(
        () => rssAdapter.collect({ url: 'https://broken-source.com/feed.xml', topic: 'ai' }),
        { maxRetries: 3 }
      )
    ).rejects.toThrow('Max retries exceeded');

    // Verify dead letter queue
    const dlqItems = await redis.lrange('dlq:collection', 0, -1);
    expect(dlqItems.length).toBe(1);
    expect(JSON.parse(dlqItems[0]).url).toBe('https://broken-source.com/feed.xml');
  });
});
```

### 4.4 Daily Digest Generation and Delivery

```javascript
describe('E2E: Daily Digest', () => {
  it('should generate and deliver daily digest', async () => {
    // Seed database with scored items
    await seedItems([
      { topic: 'ai', score: 95, title: 'Top AI News' },
      { topic: 'ai', score: 85, title: 'Good AI News' },
      { topic: 'crypto', score: 90, title: 'Crypto News' },
    ]);

    // Run digest job
    const digest = await digestService.generateDaily({ topic: 'ai' });

    // Verify digest content
    expect(digest.title).toContain('AI Daily Digest');
    expect(digest.items).toHaveLength(2);
    expect(digest.items[0].title).toBe('Top AI News');

    // Mock Telegram API
    const telegramCalls: any[] = [];
    server.use(
      http.post('https://api.telegram.org/*/sendMessage', async ({ request }) => {
        telegramCalls.push(await request.json());
        return HttpResponse.json({ ok: true });
      })
    );

    // Deliver digest
    await telegramService.sendDigest(digest);

    // Verify delivery
    expect(telegramCalls).toHaveLength(1);
    expect(telegramCalls[0].text).toContain('Top AI News');
  });
});
```

---

## 5. Quality Test Specifications

### 5.1 Source Scoring Accuracy

```javascript
describe('Source Scoring Accuracy', () => {
  const testCases = [
    { source: 'blog.google', tier: 'tier-1', expectedRange: [85, 100] },
    { source: 'openai.com', tier: 'tier-1', expectedRange: [85, 100] },
    { source: 'techcrunch.com', tier: 'tier-2', expectedRange: [60, 84] },
    { source: 'reddit.com', tier: 'tier-3', expectedRange: [20, 50] },
    { source: 'unknown-blog.xyz', tier: 'unranked', expectedRange: [10, 40] },
  ];

  it.each(testCases)(
    'should score $source in range $expectedRange',
    ({ source, expectedRange }) => {
      const score = qualityScorer.scoreSource({ domain: source });
      expect(score).toBeGreaterThanOrEqual(expectedRange[0]);
      expect(score).toBeLessThanOrEqual(expectedRange[1]);
    }
  );
});
```

### 5.2 Content Scoring Accuracy

```javascript
describe('Content Scoring Accuracy', () => {
  it('should detect primary source content', () => {
    const primary = {
      source: 'blog.google',
      content: 'We are announcing Google Bard...'.repeat(20),
      author: 'Sundar Pichai',
    };
    const secondary = {
      source: 'techcrunch.com',
      content: 'Google announced today that...'.repeat(10),
      author: 'Reporter Name',
    };

    const primaryScore = qualityScorer.scoreContent(primary);
    const secondaryScore = qualityScorer.scoreContent(secondary);

    expect(primaryScore.sourceType).toBe('primary');
    expect(primaryScore).toBeGreaterThan(secondaryScore);
  });

  it('should score long-form content higher', () => {
    const longArticle = { content: 'word '.repeat(5000) };
    const shortArticle = { content: 'word '.repeat(100) };

    const longScore = qualityScorer.scoreContent(longArticle);
    const shortScore = qualityScorer.scoreContent(shortArticle);

    expect(longScore.lengthBonus).toBeGreaterThan(shortScore.lengthBonus);
  });
});
```

### 5.3 Deduplication Accuracy

```javascript
describe('Deduplication Accuracy', () => {
  // False positive test: articles that should NOT be marked as duplicates
  const falsePositiveCases = [
    {
      name: 'Different articles on same topic',
      a: { title: 'Apple releases iPhone 16', content: 'Apple today announced...' },
      b: { title: 'Samsung Galaxy S25 revealed', content: 'Samsung has unveiled...' },
    },
    {
      name: 'Update to original story',
      a: { title: 'Tesla stock rises 5%', content: 'Tesla stock rose today...' },
      b: { title: 'Tesla stock rises 10% after earnings', content: 'Tesla stock surged...' },
    },
  ];

  // False negative test: articles that SHOULD be marked as duplicates
  const falseNegativeCases = [
    {
      name: 'Identical content different URLs',
      a: { title: 'AI Breakthrough', content: 'Researchers found...', url: 'a.com' },
      b: { title: 'AI Breakthrough', content: 'Researchers found...', url: 'b.com' },
    },
    {
      name: 'Slightly rewritten same story',
      a: { title: 'Google launches new AI model', content: 'Google today launched...' },
      b: { title: 'New AI model launched by Google', content: 'A new AI model was launched today by Google...' },
    },
  ];

  it.each(falsePositiveCases)('should not flag: $name', ({ a, b }) => {
    const result = dedupEngine.check(a, b);
    expect(result.isDuplicate).toBe(false);
  });

  it.each(falseNegativeCases)('should flag: $name', ({ a, b }) => {
    const result = dedupEngine.check(a, b);
    expect(result.isDuplicate).toBe(true);
  });
});
```

### 5.4 Freshness Decay Accuracy

```javascript
describe('Freshness Decay', () => {
  const testCases = [
    { age: 0, expectedDecay: 1.0 },      // Now
    { age: 1, expectedDecay: 0.95 },     // 1 day
    { age: 3, expectedDecay: 0.78 },     // 3 days
    { age: 7, expectedDecay: 0.55 },     // 1 week
    { age: 14, expectedDecay: 0.30 },    // 2 weeks
    { age: 30, expectedDecay: 0.10 },    // 1 month
  ];

  it.each(testCases)(
    'should apply $expectedDecay decay for $age day old item',
    ({ age, expectedDecay }) => {
      const item = { publishedAt: new Date(Date.now() - age * 86400000) };
      const decay = qualityScorer.calculateFreshnessDecay(item);
      expect(decay).toBeCloseTo(expectedDecay, 1);
    }
  );
});
```

### 5.5 Breaking News Detection

```javascript
describe('Breaking News Detection', () => {
  const breakingKeywords = [
    'BREAKING: Major earthquake hits',
    'URGENT: Cybersecurity breach at',
    'JUST IN: Supreme Court rules',
    'ALERT: Stock market crashes',
    'DEVELOPING: Nuclear agreement reached',
  ];

  const normalHeadlines = [
    'Weekly roundup of AI news',
    'Analysis: The future of cloud computing',
    'Interview with CTO of TechCorp',
    '10 best practices for DevOps',
    'Review: New MacBook Pro 2024',
  ];

  it.each(breakingKeywords)('should detect breaking: "%s"', (title) => {
    const result = qualityScorer.detectBreaking({ title });
    expect(result.isBreaking).toBe(true);
    expect(result.boost).toBe(25);
  });

  it.each(normalHeadlines)('should not flag normal: "%s"', (title) => {
    const result = qualityScorer.detectBreaking({ title });
    expect(result.isBreaking).toBe(false);
    expect(result.boost).toBe(0);
  });
});
```

---

## 6. Compliance Test Specifications

### 6.1 robots.txt Parsing

```javascript
describe('robots.txt Compliance', () => {
  it('should allow URLs not in Disallow', async () => {
    server.use(
      http.get('https://example.com/robots.txt', () =>
        HttpResponse.text(`
          User-agent: *
          Disallow: /admin/
          Disallow: /private/
        `)
      )
    );

    const allowed = await robotsChecker.isAllowed('https://example.com/article/123');
    expect(allowed).toBe(true);
  });

  it('should disallow URLs matching Disallow pattern', async () => {
    server.use(
      http.get('https://example.com/robots.txt', () =>
        HttpResponse.text(`
          User-agent: *
          Disallow: /admin/
        `)
      )
    );

    const allowed = await robotsChecker.isAllowed('https://example.com/admin/config');
    expect(allowed).toBe(false);
  });

  it('should respect crawl-delay directive', async () => {
    server.use(
      http.get('https://example.com/robots.txt', () =>
        HttpResponse.text(`
          User-agent: *
          Crawl-delay: 5
        `)
      )
    );

    const delay = await robotsChecker.getCrawlDelay('https://example.com');
    expect(delay).toBe(5000); // 5 seconds in ms
  });

  it('should handle missing robots.txt gracefully', async () => {
    server.use(
      http.get('https://example.com/robots.txt', () =>
        new HttpResponse(null, { status: 404 })
      )
    );

    const allowed = await robotsChecker.isAllowed('https://example.com/page');
    expect(allowed).toBe(true); // Default allow when no robots.txt
  });
});
```

### 6.2 Rate Limit Compliance

```javascript
describe('Rate Limit Compliance', () => {
  it('should not exceed configured requests per second', async () => {
    const timestamps: number[] = [];
    server.use(
      http.get('https://api.example.com/*', () => {
        timestamps.push(Date.now());
        return HttpResponse.json({ data: 'ok' });
      })
    );

    const requests = Array(10).fill(null).map(() =>
      rateLimiter.throttledFetch('https://api.example.com/data', { rps: 2 })
    );

    await Promise.all(requests);

    // Verify no two requests in same 500ms window
    for (let i = 1; i < timestamps.length; i++) {
      const diff = timestamps[i] - timestamps[i - 1];
      expect(diff).toBeGreaterThanOrEqual(450); // Allow 50ms tolerance
    }
  });

  it('should honor Retry-After header', async () => {
    server.use(
      http.get('https://api.example.com/data', () =>
        new HttpResponse(null, {
          status: 429,
          headers: { 'Retry-After': '2' },
        })
      )
    );

    const start = Date.now();
    await rateLimiter.throttledFetch('https://api.example.com/data', { retryAfter: true });
    const elapsed = Date.now() - start;

    expect(elapsed).toBeGreaterThanOrEqual(1900); // ~2 seconds
  });
});
```

### 6.3 User-Agent Compliance

Tests that the canonical User-Agent format defined in `security.md` Section 9.1 is
used unmodified for every outbound request. The canonical UA is operator-customized
at deployment but is required to follow this shape:

```
<BotName>/<Version> (+<about-bot URL>; contact=<contact>; purpose=<purpose>; respects-robots-txt=true)
```

```javascript
describe('User-Agent Compliance', () => {
  it('should send a structured, identifiable, robots-respecting User-Agent', async () => {
    const requests: Request[] = [];
    server.use(
      http.get('*', ({ request }) => {
        requests.push(request);
        return HttpResponse.json({});
      })
    );

    await httpClient.get('https://example.com/page');

    const ua = requests[0].headers.get('User-Agent');
    // Must identify as a bot with version
    expect(ua).toMatch(/^[A-Za-z][A-Za-z0-9_-]*-?Bot\/\d+\.\d+/);
    // Must include the about-bot URL marker
    expect(ua).toMatch(/\(\+https?:\/\/[^\s)]+/);
    // Must include a contact directive
    expect(ua).toMatch(/contact=[^\s)]+/);
    // Must explicitly signal robots.txt respect
    expect(ua).toContain('respects-robots-txt=true');
    // Must NOT impersonate a browser
    expect(ua).not.toMatch(/Mozilla\/|Chrome\/|Safari\/|Firefox\/|Edge\//);
  });
});
```

### 6.4 PII Sanitization

```javascript
describe('PII Sanitization', () => {
  it('should remove email addresses from content', () => {
    const dirty = 'Contact john.doe@email.com for details.';
    const clean = sanitizer.sanitize(dirty);
    expect(clean).not.toMatch(/[\w.-]+@[\w.-]+\.\w+/);
    expect(clean).toContain('[EMAIL REMOVED]');
  });

  it('should remove phone numbers from content', () => {
    const dirty = 'Call me at +1-555-123-4567.';
    const clean = sanitizer.sanitize(dirty);
    expect(clean).not.toMatch(/\+?\d{1,3}[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}/);
  });

  it('should not remove non-PII numbers', () => {
    const clean = 'Version 2.1.0 has 5000+ downloads.';
    const result = sanitizer.sanitize(clean);
    expect(result).toBe(clean);
  });

  it('should hash IP addresses if present', () => {
    const dirty = 'Server at 192.168.1.1 reported...';
    const clean = sanitizer.sanitize(dirty);
    expect(clean).toContain('[IP HASHED]');
    expect(clean).not.toContain('192.168.1.1');
  });
});
```

### 6.5 Attribution Preservation

```javascript
describe('Attribution Preservation', () => {
  it('should preserve original source URL', async () => {
    const item = {
      url: 'https://original-source.com/article',
      title: 'Original Article',
    };

    const processed = await pipeline.process(item);
    expect(processed.originalUrl).toBe('https://original-source.com/article');
    expect(processed.sourceDomain).toBe('original-source.com');
  });

  it('should maintain attribution chain through dedup', async () => {
    const primary = await insertItem({
      url: 'https://reuters.com/article',
      sources: ['reuters.com'],
    });
    const duplicate = await insertItem({
      url: 'https://aggregator.com/same',
      sources: ['aggregator.com'],
      title: primary.title,
    });

    await dedupPipeline.process([primary, duplicate]);

    const merged = await db.select().from(items).where(eq(items.id, primary.id));
    expect(merged[0].attributionChain).toEqual([
      { source: 'reuters.com', url: 'https://reuters.com/article', date: expect.any(Date) },
      { source: 'aggregator.com', url: 'https://aggregator.com/same', date: expect.any(Date) },
    ]);
  });

  it('should include source attribution in output', async () => {
    const item = await insertItem({
      url: 'https://example.com/article',
      source: 'Tech Blog',
      author: 'Jane Smith',
    });

    const digest = await digestService.generate({ items: [item] });
    expect(digest).toContain('via Tech Blog');
    expect(digest).toContain('Jane Smith');
  });
});
```

---

## 7. Performance Test Specifications

### 7.1 Scrape 100 Items in Under 30 Seconds

```javascript
describe('Performance: Scrape Speed', () => {
  it('should scrape 100 items in under 30 seconds', async () => {
    // Generate 100-item feed
    const largeFeed = generateRSSFeed(100);
    server.use(
      http.get('https://perf-test.com/feed.xml', () =>
        HttpResponse.xml(largeFeed)
    ));

    const start = Date.now();
    const result = await rssAdapter.collect({
      url: 'https://perf-test.com/feed.xml',
      topic: 'performance-test',
    });
    const elapsed = Date.now() - start;

    expect(result.items).toHaveLength(100);
    expect(elapsed).toBeLessThan(30000);
  }, 35000);
});
```

### 7.2 Handle Feeds with 1000+ Items

```javascript
describe('Performance: Large Feed Handling', () => {
  it('should process 1000-item feed without OOM', async () => {
    const hugeFeed = generateRSSFeed(1000);
    server.use(
      http.get('https://huge-feed.com/feed.xml', () =>
        HttpResponse.xml(hugeFeed)
    ));

    const memBefore = process.memoryUsage().heapUsed;
    const result = await rssAdapter.collect({
      url: 'https://huge-feed.com/feed.xml',
      topic: 'huge-test',
    });
    const memAfter = process.memoryUsage().heapUsed;
    const memUsed = (memAfter - memBefore) / 1024 / 1024;

    expect(result.items).toHaveLength(1000);
    expect(memUsed).toBeLessThan(50); // Less than 50MB overhead
  }, 60000);
});
```

### 7.3 Memory Usage Under 256MB Per Worker

```javascript
describe('Performance: Memory Usage', () => {
  it('should keep worker memory under 256MB', async () => {
    const worker = new Worker('test-queue', async (job) => {
      return await pipeline.process(job.data);
    });

    // Process 500 items
    for (let i = 0; i < 500; i++) {
      await testQueue.add('process', { itemId: i });
    }

    await waitForQueueDrain(testQueue);

    const maxMem = worker.maxMemoryUsage / 1024 / 1024;
    expect(maxMem).toBeLessThan(256);

    await worker.close();
  }, 120000);
});
```

### 7.4 Database Query Response Under 100ms

```javascript
describe('Performance: Database Queries', () => {
  it('should query items in under 100ms', async () => {
    // Seed 10,000 items
    await seedItems(10000);

    const start = Date.now();
    const results = await db.select()
      .from(items)
      .where(eq(items.topic, 'ai'))
      .limit(50)
      .orderBy(desc(items.qualityScore));
    const elapsed = Date.now() - start;

    expect(elapsed).toBeLessThan(100);
    expect(results.length).toBeLessThanOrEqual(50);
  });

  it('should query deduplication candidates in under 100ms', async () => {
    await seedItems(5000);

    const start = Date.now();
    const candidates = await db.select()
      .from(items)
      .where(
        and(
          gte(items.publishedAt, new Date(Date.now() - 86400000)),
          eq(items.status, 'active')
        )
      );
    const elapsed = Date.now() - start;

    expect(elapsed).toBeLessThan(100);
  });
});
```

### 7.5 Pipeline Throughput: 50 Items/Minute

```javascript
describe('Performance: Pipeline Throughput', () => {
  it('should process 50 items per minute', async () => {
    const items = generateItems(50);

    const start = Date.now();
    await pipeline.processBatch(items);
    const elapsed = Date.now() - start;

    const itemsPerMinute = (50 / elapsed) * 60000;
    expect(itemsPerMinute).toBeGreaterThanOrEqual(50);
    expect(elapsed).toBeLessThan(60000);
  }, 65000);
});
```

---

## 8. Test Fixtures

### 8.1 Sample RSS Feed XML (`fixtures/valid-rss-2.0.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom"
     xmlns:content="http://purl.org/rss/1.0/modules/content/">
  <channel>
    <title>AI News Daily</title>
    <link>https://ai-news.example.com</link>
    <description>Latest news in artificial intelligence</description>
    <language>en</language>
    <lastBuildDate>Mon, 15 Mar 2024 12:00:00 GMT</lastBuildDate>
    <atom:link href="https://ai-news.example.com/feed.xml" rel="self" type="application/rss+xml"/>
    <item>
      <title>AI Breakthrough in 2024: New Model Surpasses GPT-4</title>
      <link>https://ai-news.example.com/breakthrough-2024</link>
      <guid isPermaLink="true">https://ai-news.example.com/breakthrough-2024</guid>
      <description>Researchers have developed a new AI model that significantly outperforms existing systems.</description>
      <content:encoded><![CDATA[<p>Researchers at Example Labs have developed a new AI model that significantly outperforms GPT-4 on benchmark tests. The model, called ExampleNet, achieved state-of-the-art results on multiple natural language processing tasks.</p>]]></content:encoded>
      <author>jane.smith@example.com (Jane Smith)</author>
      <pubDate>Mon, 15 Mar 2024 10:30:00 GMT</pubDate>
      <category>Artificial Intelligence</category>
      <enclosure url="https://ai-news.example.com/images/breakthrough.jpg" length="12345" type="image/jpeg"/>
    </item>
    <item>
      <title>Open Source AI Tools Gain Enterprise Adoption</title>
      <link>https://ai-news.example.com/enterprise-adoption</link>
      <guid isPermaLink="true">https://ai-news.example.com/enterprise-adoption</guid>
      <description>Major corporations are increasingly adopting open-source AI tools for production use.</description>
      <author>john.doe@example.com (John Doe)</author>
      <pubDate>Mon, 15 Mar 2024 09:00:00 GMT</pubDate>
      <category>Enterprise</category>
    </item>
    <item>
      <title>EU AI Act: Compliance Requirements Explained</title>
      <link>https://ai-news.example.com/eu-ai-act</link>
      <guid>item-003</guid>
      <description>A detailed breakdown of the EU AI Act and what it means for developers.</description>
      <author>sarah.jones@example.com (Sarah Jones)</author>
      <pubDate>Sun, 14 Mar 2024 18:00:00 GMT</pubDate>
      <category>Regulation</category>
    </item>
  </channel>
</rss>
```

### 8.2 Sample Atom Feed XML (`fixtures/valid-atom.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>Tech Blog Atom Feed</title>
  <link href="https://tech-blog.example.com/"/>
  <updated>2024-03-15T12:00:00Z</updated>
  <author>
    <name>Tech Blog Team</name>
    <email>team@tech-blog.example.com</email>
  </author>
  <id>urn:uuid:60a76c80-d399-11d9-b93C-0003939e0af6</id>
  <entry>
    <title>WebAssembly: The Future of Web Development</title>
    <link href="https://tech-blog.example.com/webassembly-future"/>
    <id>urn:uuid:1225c695-cfb8-4ebb-aaaa-80da344efa6a</id>
    <updated>2024-03-15T11:00:00Z</updated>
    <summary>WebAssembly is changing how we build web applications with near-native performance.</summary>
    <content type="html">&lt;p&gt;WebAssembly (Wasm) has emerged as a powerful technology...&lt;/p&gt;</content>
    <author>
      <name>Alex Chen</name>
      <uri>https://tech-blog.example.com/authors/alex-chen</uri>
    </author>
    <category term="WebAssembly"/>
    <category term="Performance"/>
  </entry>
  <entry>
    <title>Rust 2024 Edition Preview</title>
    <link href="https://tech-blog.example.com/rust-2024"/>
    <id>urn:uuid:1225c695-cfb8-4ebb-bbbb-80da344efa6b</id>
    <updated>2024-03-14T15:30:00Z</updated>
    <summary>The Rust team announces the upcoming 2024 edition with exciting new features.</summary>
    <author>
      <name>María García</name>
    </author>
    <category term="Rust"/>
    <category term="Programming Languages"/>
  </entry>
</feed>
```

### 8.3 Sample HTML Article (`fixtures/article-full.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="author" content="Jane Smith">
  <meta property="article:published_time" content="2024-03-15T10:30:00Z">
  <meta property="og:title" content="The Future of AI in Healthcare">
  <title>The Future of AI in Healthcare - Tech Blog</title>
  <link rel="canonical" href="https://tech-blog.example.com/ai-healthcare-future">
</head>
<body>
  <header>
    <nav>Home | Articles | About</nav>
  </header>
  <article class="article-content" data-article-id="12345">
    <h1>The Future of AI in Healthcare</h1>
    <div class="article-meta">
      <span class="author">By <a href="/authors/jane-smith">Jane Smith</a></span>
      <span class="publish-date">March 15, 2024</span>
      <span class="read-time">8 min read</span>
    </div>
    <div class="article-body">
      <p class="lead">Artificial intelligence is poised to transform healthcare in ways we're only beginning to understand. From diagnostic imaging to drug discovery, AI tools are already making a measurable impact on patient outcomes.</p>
      <p>Over the past decade, machine learning algorithms have achieved remarkable accuracy in detecting conditions from medical imaging. A recent study published in Nature Medicine found that AI systems could identify certain cancers with greater accuracy than human radiologists, reducing false positives by 5.7% and false negatives by 9.4%.</p>
      <h2>Current Applications</h2>
      <p>Today, AI is being deployed across the healthcare spectrum. Some of the most prominent applications include:</p>
      <ul>
        <li>Medical imaging analysis and anomaly detection</li>
        <li>Drug discovery and molecular modeling</li>
        <li>Personalized treatment recommendations</li>
        <li>Administrative workflow optimization</li>
        <li>Predictive analytics for patient risk stratification</li>
      </ul>
      <h2>Challenges Ahead</h2>
      <p>Despite these advances, significant challenges remain. Data privacy concerns, regulatory approval processes, and the need for diverse training datasets all present obstacles to widespread adoption. Furthermore, questions about liability and the "black box" nature of some AI models continue to generate debate among policymakers and practitioners.</p>
      <p>Healthcare organizations must also address the digital infrastructure gap. Many hospitals, particularly in rural and underserved areas, lack the technical capacity to implement and maintain AI systems. This digital divide threatens to exacerbate existing health disparities.</p>
      <h2>The Road Forward</h2>
      <p>Looking ahead, the integration of AI into healthcare will likely accelerate. The FDA has established a framework for AI/ML-based medical device approval, and several major health systems have announced significant investments in AI infrastructure. As these technologies mature, we can expect to see more seamless integration into clinical workflows, with AI serving as a decision support tool rather than a replacement for human judgment.</p>
      <p>The ultimate measure of success will be whether these technologies can deliver on their promise of improved patient outcomes while maintaining the trust and confidence of both healthcare providers and the patients they serve.</p>
    </div>
    <div class="article-tags">
      <a href="/tags/ai">Artificial Intelligence</a>
      <a href="/tags/healthcare">Healthcare</a>
      <a href="/tags/technology">Technology</a>
    </div>
  </article>
  <aside class="sidebar">
    <div class="ad">Advertisement</div>
    <div class="newsletter-signup">
      <h3>Subscribe to our newsletter</h3>
      <input type="email" placeholder="Enter your email">
      <button>Subscribe</button>
    </div>
  </aside>
  <footer>
    <p>&copy; 2024 Tech Blog. All rights reserved.</p>
  </footer>
</body>
</html>
```

### 8.4 Sample API Response JSON (`fixtures/api-response.json`)

```json
{
  "status": "ok",
  "totalResults": 25,
  "page": 1,
  "pageSize": 10,
  "articles": [
    {
      "id": "article-001",
      "title": "Neural Networks Revolutionize Climate Modeling",
      "description": "New neural network architectures are enabling more accurate climate predictions than ever before.",
      "url": "https://science-daily.example.com/neural-climate-modeling",
      "urlToImage": "https://cdn.example.com/images/climate-model.jpg",
      "publishedAt": "2024-03-15T08:00:00Z",
      "author": {
        "name": "Dr. Emily Watson",
        "id": "author-42"
      },
      "source": {
        "id": "science-daily",
        "name": "Science Daily"
      },
      "category": ["Science", "AI"],
      "content": "A team of researchers at the Climate Science Institute has developed a novel neural network architecture...",
      "readingTime": 6,
      "engagement": {
        "views": 15420,
        "shares": 892,
        "comments": 156
      }
    },
    {
      "id": "article-002",
      "title": "Edge AI: Processing at the Speed of Reality",
      "description": "How edge computing is bringing AI inference closer to where data is generated.",
      "url": "https://tech-weekly.example.com/edge-ai-processing",
      "urlToImage": null,
      "publishedAt": "2024-03-14T16:45:00Z",
      "author": {
        "name": "Michael Torres",
        "id": "author-88"
      },
      "source": {
        "id": "tech-weekly",
        "name": "Tech Weekly"
      },
      "category": ["Technology", "Hardware"],
      "content": "Edge AI represents a fundamental shift in how we deploy machine learning models...",
      "readingTime": 8,
      "engagement": {
        "views": 8930,
        "shares": 445,
        "comments": 89
      }
    }
  ],
  "pagination": {
    "currentPage": 1,
    "totalPages": 3,
    "nextPage": 2,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}
```

### 8.5 Sample GitHub Release Response (`fixtures/github-release.json`)

```json
[
  {
    "url": "https://api.github.com/repos/octocat/Hello-World/releases/1",
    "html_url": "https://github.com/octocat/Hello-World/releases/v1.0.0",
    "id": 1,
    "tag_name": "v1.0.0",
    "name": "Hello World v1.0.0",
    "body": "## Release Notes\n\n### Features\n- Added dark mode support\n- New dashboard widgets\n- Improved search functionality\n\n### Bug Fixes\n- Fixed login timeout issue (#123)\n- Resolved memory leak in data processing (#124)\n\n### Breaking Changes\n- API v1 deprecated, migrate to v2",
    "draft": false,
    "prerelease": false,
    "created_at": "2024-03-15T10:00:00Z",
    "published_at": "2024-03-15T12:00:00Z",
    "author": {
      "login": "octocat",
      "id": 1,
      "avatar_url": "https://github.com/images/error/octocat_happy.gif"
    },
    "assets": [
      {
        "id": 1,
        "name": "hello-world-linux-amd64.tar.gz",
        "content_type": "application/gzip",
        "size": 1024000,
        "download_count": 1250,
        "browser_download_url": "https://github.com/octocat/Hello-World/releases/download/v1.0.0/hello-world-linux-amd64.tar.gz"
      },
      {
        "id": 2,
        "name": "hello-world-darwin-arm64.tar.gz",
        "content_type": "application/gzip",
        "size": 980000,
        "download_count": 890,
        "browser_download_url": "https://github.com/octocat/Hello-World/releases/download/v1.0.0/hello-world-darwin-arm64.tar.gz"
      }
    ]
  },
  {
    "url": "https://api.github.com/repos/octocat/Hello-World/releases/2",
    "html_url": "https://github.com/octocat/Hello-World/releases/v1.0.0-rc1",
    "id": 2,
    "tag_name": "v1.0.0-rc1",
    "name": "Release Candidate 1",
    "body": "## Pre-release\n\nThis is a release candidate for testing purposes.",
    "draft": false,
    "prerelease": true,
    "created_at": "2024-03-10T08:00:00Z",
    "published_at": "2024-03-10T10:00:00Z",
    "author": {
      "login": "octocat",
      "id": 1,
      "avatar_url": "https://github.com/images/error/octocat_happy.gif"
    },
    "assets": []
  }
]
```

### 8.6 Sample Source Configs (`fixtures/source-configs.json`)

```json
{
  "sources": [
    {
      "id": "official-google-ai",
      "name": "Google AI Blog",
      "url": "https://blog.google/technology/ai/",
      "type": "rss",
      "tier": "tier-1",
      "topics": ["ai"],
      "weight": 1.0,
      "rateLimit": { "requestsPerMinute": 10 },
      "selectors": {
        "title": "entry > title",
        "content": "entry > content",
        "date": "entry > published"
      },
      "options": {
        "followRedirects": true,
        "timeout": 30000,
        "userAgent": "JarvisHQ/1.0"
      }
    },
    {
      "id": "techcrunch-ai",
      "name": "TechCrunch AI",
      "url": "https://techcrunch.com/category/artificial-intelligence/feed/",
      "type": "rss",
      "tier": "tier-2",
      "topics": ["ai"],
      "weight": 0.8,
      "rateLimit": { "requestsPerMinute": 30 },
      "selectors": {
        "title": "item > title",
        "content": "item > description",
        "date": "item > pubDate"
      }
    },
    {
      "id": "github-releases-tensorflow",
      "name": "TensorFlow GitHub Releases",
      "url": "https://api.github.com/repos/tensorflow/tensorflow/releases",
      "type": "github",
      "tier": "tier-1",
      "topics": ["ai", "open-source"],
      "weight": 0.9,
      "rateLimit": { "requestsPerHour": 60 },
      "auth": {
        "type": "token",
        "envVar": "GITHUB_TOKEN"
      }
    },
    {
      "id": "hackernews",
      "name": "Hacker News",
      "url": "https://hn.algolia.com/api/v1/search_by_date",
      "type": "api",
      "tier": "tier-2",
      "topics": ["ai", "programming", "startup"],
      "weight": 0.6,
      "rateLimit": { "requestsPerSecond": 1 },
      "pagination": {
        "type": "page",
        "pageParam": "page",
        "pageSize": 20
      }
    }
  ]
}
```

### 8.7 Sample Topic Configs (`fixtures/topic-configs.json`)

```json
{
  "topics": [
    {
      "id": "artificial-intelligence",
      "name": "Artificial Intelligence",
      "slug": "ai",
      "keywords": [
        "artificial intelligence",
        "machine learning",
        "deep learning",
        "neural network",
        "LLM",
        "large language model",
        "GPT",
        "transformer",
        "computer vision",
        "NLP",
        "reinforcement learning"
      ],
      "excludedKeywords": [
        "AI generated art contest",
        "AI boyfriend",
        "AI girlfriend"
      ],
      "sources": ["official-google-ai", "openai-blog", "anthropic-blog"],
      "qualityThreshold": 60,
      "maxItemsPerDigest": 20,
      "schedule": {
        "collection": "*/30 * * * *",
        "digest": "0 9 * * *"
      },
      "outputChannels": ["telegram", "web", "api"]
    },
    {
      "id": "crypto-blockchain",
      "name": "Crypto & Blockchain",
      "slug": "crypto",
      "keywords": [
        "cryptocurrency",
        "blockchain",
        "bitcoin",
        "ethereum",
        "DeFi",
        "smart contract",
        "NFT",
        "web3",
        "layer 2",
        "zero knowledge"
      ],
      "excludedKeywords": [
        "crypto scam",
        "pump and dump"
      ],
      "sources": ["coindesk", "the-block", "decrypt"],
      "qualityThreshold": 55,
      "maxItemsPerDigest": 15,
      "schedule": {
        "collection": "*/60 * * * *",
        "digest": "0 10 * * *"
      },
      "outputChannels": ["telegram", "web"]
    }
  ]
}
```

---

## 9. Test Infrastructure

### 9.1 Framework and Tools

| Layer | Tool | Purpose |
|-------|------|---------|
| Test Runner | Vitest | Unit and integration test execution |
| Mocking | MSW (Mock Service Worker) | HTTP request interception |
| Fixtures | @faker-js/faker | Test data generation |
| DB Testing | testcontainers (PostgreSQL) | Isolated database per test suite |
| Redis Testing | ioredis-mock | In-memory Redis mock |
| BullMQ Testing | bullmq-test | Queue testing utilities |
| Coverage | @vitest/coverage-v8 | Code coverage reporting |
| E2E | Supertest + Docker Compose | API and full pipeline testing |

### 9.2 Vitest Configuration (`vitest.config.ts`)

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
      exclude: [
        'test/**',
        '**/*.d.ts',
        '**/node_modules/**',
        '**/dist/**',
        '**/*.config.*',
      ],
    },
    testTimeout: 30000,
    hookTimeout: 30000,
    poolOptions: {
      threads: {
        singleThread: false,
        maxThreads: 4,
      },
    },
  },
});
```

### 9.3 Test Setup (`test/setup.ts`)

```typescript
import { beforeAll, afterAll, afterEach } from 'vitest';
import { setupServer } from 'msw/node';
import { db } from '../src/db';
import { redis } from '../src/redis';

// MSW server for HTTP mocking
export const server = setupServer();

beforeAll(async () => {
  server.listen({ onUnhandledRequest: 'error' });
  await db.migrate.latest();
  await redis.flushall();
});

afterEach(() => {
  server.resetHandlers();
});

afterAll(async () => {
  server.close();
  await db.destroy();
  await redis.quit();
});
```

### 9.4 Factory Functions (`test/factories.ts`)

```typescript
import { faker } from '@faker-js/faker';
import { db } from '../src/db';
import { items } from '../src/db/schema';

export function createItem(overrides: Partial<typeof items.$inferInsert> = {}) {
  return {
    id: faker.string.uuid(),
    title: faker.lorem.sentence(),
    url: faker.internet.url(),
    content: faker.lorem.paragraphs(3),
    author: faker.person.fullName(),
    source: faker.internet.domainName(),
    publishedAt: faker.date.recent(),
    topic: faker.helpers.arrayElement(['ai', 'crypto', 'programming']),
    qualityScore: faker.number.int({ min: 0, max: 100 }),
    status: 'raw' as const,
    ...overrides,
  };
}

export function createSource(overrides: Partial<SourceConfig> = {}) {
  return {
    id: faker.string.uuid(),
    name: faker.company.name(),
    url: faker.internet.url(),
    type: faker.helpers.arrayElement(['rss', 'api', 'html', 'github']),
    tier: faker.helpers.arrayElement(['tier-1', 'tier-2', 'tier-3']),
    topics: [faker.helpers.arrayElement(['ai', 'crypto', 'programming'])],
    weight: faker.number.float({ min: 0.1, max: 1.0 }),
    ...overrides,
  };
}

export function createScoredItem(overrides: Partial<typeof items.$inferInsert> = {}) {
  return createItem({
    status: 'scored',
    qualityScore: faker.number.int({ min: 50, max: 100 }),
    sourceScore: faker.number.int({ min: 40, max: 100 }),
    contentScore: faker.number.int({ min: 40, max: 100 }),
    ...overrides,
  });
}

export async function seedItems(count: number, overrides = {}) {
  const data = Array.from({ length: count }, () => createItem(overrides));
  await db.insert(items).values(data);
  return data;
}

export async function insertScoredItems(itemsList: Partial<typeof items.$inferInsert>[] = []) {
  const data = itemsList.length > 0
    ? itemsList.map(i => createScoredItem(i))
    : [createScoredItem(), createScoredItem(), createScoredItem()];
  await db.insert(items).values(data);
  return data;
}
```

### 9.5 GitHub Actions CI Workflow (`.github/workflows/test.yml`)

```yaml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: jarvis_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint

      - name: Run unit tests
        run: npm run test:unit -- --coverage
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/jarvis_test
          REDIS_URL: redis://localhost:6379

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/jarvis_test
          REDIS_URL: redis://localhost:6379

      - name: Run performance tests
        run: npm run test:perf
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/jarvis_test
          REDIS_URL: redis://localhost:6379

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-umbrella

      - name: Coverage report
        run: |
          echo "Coverage Summary:"
          cat ./coverage/coverage-summary.json

  e2e:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Docker
        uses: docker/setup-buildx-action@v3

      - name: Run E2E tests
        run: |
          docker compose -f docker-compose.test.yml up -d
          npm run test:e2e
          docker compose -f docker-compose.test.yml down
        env:
          NODE_ENV: test

  compliance:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run compliance tests
        run: npm run test:compliance

      - name: Run quality tests
        run: npm run test:quality
```

### 9.6 Coverage Targets

| Component | Line Coverage | Branch Coverage | Function Coverage |
|-----------|--------------|-----------------|-------------------|
| Adapters (all) | >= 85% | >= 80% | >= 85% |
| Quality Scorer | >= 90% | >= 85% | >= 90% |
| Deduplication Engine | >= 90% | >= 85% | >= 90% |
| Storage/Database | >= 80% | >= 75% | >= 80% |
| Pipeline Orchestrator | >= 85% | >= 80% | >= 85% |
| Output Formatters | >= 80% | >= 75% | >= 80% |
| Utilities (date, URL, etc.) | >= 90% | >= 85% | >= 90% |
| **Overall** | **>= 80%** | **>= 75%** | **>= 80%** |

---

## 10. Testing Checklist for Codex

### Complete Component Checklist

```markdown
## Testing Checklist

### Unit Tests

#### RSS Adapter (10 tests)
- [ ] `should parse valid RSS 2.0 feed`
  - Input: `fixtures/valid-rss-2.0.xml` (3 items)
  - Expected: Array of 3 parsed items with title, link, pubDate
- [ ] `should parse valid Atom feed`
  - Input: `fixtures/valid-atom.xml` (2 entries)
  - Expected: Array of 2 parsed items with title, link, updated date
- [ ] `should handle empty feed`
  - Input: RSS with 0 `<item>` elements
  - Expected: Empty array + warning log
- [ ] `should handle invalid XML`
  - Input: Malformed/unclosed tags
  - Expected: Empty array, error captured
- [ ] `should handle 404 response`
  - Input: HTTP 404 status
  - Expected: Empty array, source marked unreachable
- [ ] `should handle 500 response`
  - Input: HTTP 500 status
  - Expected: Empty array, retry scheduled
- [ ] `should handle large feed (1000+ items)`
  - Input: Generated 1000-item feed
  - Expected: All parsed, memory < 50MB overhead
- [ ] `should extract all standard fields`
  - Input: Full RSS item with all optional elements
  - Expected: Object with title, link, description, author, guid, pubDate, enclosure
- [ ] `should handle missing pubDate`
  - Input: RSS item without `<pubDate>`
  - Expected: Item with fallback date, `dateSource: 'fallback'`
- [ ] `should handle relative URLs in links`
  - Input: RSS with relative `<link>` paths
  - Expected: Absolute URLs resolved against base feed URL

#### API Adapter (10 tests)
- [ ] `should handle API key authentication`
  - Input: Config with `auth: { type: 'bearer', token: 'tkn123' }`
  - Expected: Request headers include `Authorization: Bearer tkn123`
- [ ] `should handle pagination (page-based)`
  - Input: API with 25 results, pageSize=10
  - Expected: 3 API calls, all 25 items returned
- [ ] `should handle cursor-based pagination`
  - Input: API with `nextCursor` field
  - Expected: Follows all cursors, returns combined results
- [ ] `should handle rate limit (429 response)`
  - Input: HTTP 429 + `Retry-After: 60`
  - Expected: Waits 60 seconds, retries request
- [ ] `should handle server error (500)`
  - Input: HTTP 500 response
  - Expected: Retries 3x with backoff, then fails
- [ ] `should parse JSON response`
  - Input: Valid JSON body
  - Expected: Correctly typed parsed object
- [ ] `should handle malformed JSON`
  - Input: Invalid JSON body
  - Expected: Error captured, logged
- [ ] `should apply request timeout`
  - Input: Slow response (> configured timeout)
  - Expected: Request aborted, timeout error
- [ ] `should handle OAuth2 flow`
  - Input: Expired token, refresh endpoint available
  - Expected: Token refreshed, original request retried
- [ ] `should validate response schema`
  - Input: JSON missing required fields
  - Expected: Schema error with field names listed

#### HTML Article Adapter (10 tests)
- [ ] `should extract article title`
  - Input: `fixtures/article-full.html`
  - Expected: Title text without site suffix
- [ ] `should extract author name`
  - Input: `fixtures/article-full.html`
  - Expected: Author name from meta or byline
- [ ] `should extract publication date`
  - Input: `fixtures/article-full.html`
  - Expected: ISO date string from meta tag
- [ ] `should handle paywall content`
  - Input: `fixtures/article-paywall.html`
  - Expected: `isPaywalled: true`, truncated content
- [ ] `should handle missing author`
  - Input: HTML without author information
  - Expected: `author: null`, no error
- [ ] `should handle missing date`
  - Input: HTML without date information
  - Expected: `date: null`, fallback used
- [ ] `should extract main content`
  - Input: `fixtures/article-full.html`
  - Expected: Clean article text > 1000 chars
- [ ] `should remove ads and navigation`
  - Input: HTML with ad containers and nav
  - Expected: No ad text in extracted content
- [ ] `should handle JavaScript-rendered pages`
  - Input: SPA-style page
  - Expected: Content after JS execution
- [ ] `should handle encoding issues`
  - Input: UTF-8 and Latin-1 encoded pages
  - Expected: Correctly decoded special characters

#### GitHub Release Adapter (8 tests)
- [ ] `should parse release list`
  - Input: `fixtures/github-release.json`
  - Expected: Array of releases with version, notes, date
- [ ] `should handle repository with no releases`
  - Input: Empty `[]` response
  - Expected: Empty array, info log
- [ ] `should handle API rate limit (403)`
  - Input: HTTP 403 + `X-RateLimit-Remaining: 0`
  - Expected: Rate limit error, retry after reset
- [ ] `should parse release assets`
  - Input: Release with binary assets
  - Expected: Asset names, sizes, download URLs
- [ ] `should extract changelog from release body`
  - Input: Markdown release notes
  - Expected: Structured sections (features, fixes, breaking)
- [ ] `should handle pre-release flag`
  - Input: `prerelease: true`
  - Expected: `isPreRelease: true`, `stability: 'beta'`
- [ ] `should handle GraphQL API errors`
  - Input: `{ errors: [...] }` response
  - Expected: Error extracted, no data
- [ ] `should handle non-existent repository`
  - Input: HTTP 404 for repository
  - Expected: Empty result, source marked invalid

#### Quality Scorer (12 tests)
- [ ] `should calculate source score for tier-1 source`
  - Input: `tier: 'tier-1'`, high reputation domain
  - Expected: Score 85-100
- [ ] `should calculate source score for tier-3 source`
  - Input: `tier: 'tier-3'`, low reputation
  - Expected: Score 20-50
- [ ] `should calculate content score for long article`
  - Input: 5000+ word article
  - Expected: Length bonus applied
- [ ] `should calculate content score for short snippet`
  - Input: < 100 words
  - Expected: Length penalty applied
- [ ] `should apply freshness decay for 1-day-old item`
  - Input: Item from 24h ago
  - Expected: Decay ~0.95, freshness: 'high'
- [ ] `should apply freshness decay for 7-day-old item`
  - Input: Item from 7 days ago
  - Expected: Decay ~0.55, freshness: 'medium'
- [ ] `should apply freshness decay for 30-day-old item`
  - Input: Item from 30 days ago
  - Expected: Decay ~0.10, freshness: 'low'
- [ ] `should apply breaking news boost`
  - Input: Title with breaking keywords
  - Expected: +25 boost, `isBreaking: true`
- [ ] `should apply primary source boost`
  - Input: Official blog post
  - Expected: +15 over secondary reporting
- [ ] `should apply expert author bonus`
  - Input: Known expert author
  - Expected: +10 bonus, `authorTier: 'expert'`
- [ ] `should cap score at 100`
  - Input: All bonuses exceeding 100
  - Expected: Score = 100
- [ ] `should floor score at 0`
  - Input: All penalties below 0
  - Expected: Score = 0

#### Deduplication Engine (10 tests)
- [ ] `should match exact URL duplicates`
  - Input: Two items with identical URL
  - Expected: Second marked duplicate
- [ ] `should match similar titles (>90%)`
  - Input: Two items with 95% title similarity
  - Expected: Marked as duplicate
- [ ] `should not flag different titles (<70%)`
  - Input: Two items with 50% title similarity
  - Expected: Not duplicate
- [ ] `should match similar content (>85%)`
  - Input: Two items with 90% content overlap
  - Expected: Marked as duplicate
- [ ] `should respect time window (within 24h)`
  - Input: Two items 12h apart, same content
  - Expected: Duplicate (within window)
- [ ] `should ignore old items outside window`
  - Input: Two items 72h apart
  - Expected: Not duplicate (outside window)
- [ ] `should keep higher-quality item as primary`
  - Input: Two duplicates, different scores
  - Expected: Higher score = primary
- [ ] `should merge metadata from duplicates`
  - Input: Two duplicates with different tags
  - Expected: Merged tags in primary
- [ ] `should handle near-duplicate detection`
  - Input: Items with minor text changes
  - Expected: Detected with confidence score
- [ ] `should not flag different articles on same topic`
  - Input: Distinct articles about same topic
  - Expected: Not duplicate

#### Date Normalizer (10 tests)
- [ ] `should parse ISO 8601 date`
  - Input: `'2024-03-15T10:30:00Z'`
  - Expected: Correct Date object
- [ ] `should parse RFC 2822 date`
  - Input: `'Fri, 15 Mar 2024 10:30:00 GMT'`
  - Expected: Correct Date object
- [ ] `should parse RSS pubDate`
  - Input: `'Mon, 15 Mar 2024 10:30:00 +0000'`
  - Expected: Correct Date object
- [ ] `should parse relative date strings`
  - Input: `'2 hours ago'`, `'yesterday'`
  - Expected: Date relative to now
- [ ] `should handle timezone conversion`
  - Input: `'2024-03-15T10:30:00-05:00'`
  - Expected: UTC-normalized Date
- [ ] `should handle invalid date string`
  - Input: `'not a date'`
  - Expected: `null`, warning logged
- [ ] `should handle ambiguous US/EU formats`
  - Input: `'03/04/2024'`
  - Expected: Correct based on config preference
- [ ] `should handle Unix timestamp (seconds)`
  - Input: `1710498600`
  - Expected: Correct Date object
- [ ] `should handle Unix timestamp (milliseconds)`
  - Input: `1710498600000`
  - Expected: Correct Date object
- [ ] `should return null for empty input`
  - Input: `''`, `null`, `undefined`
  - Expected: `null`

#### URL Validator (10 tests)
- [ ] `should validate HTTPS URL`
  - Input: `'https://example.com/article'`
  - Expected: `valid: true`
- [ ] `should validate HTTP URL`
  - Input: `'http://example.com/article'`
  - Expected: `valid: true`, warning
- [ ] `should reject invalid protocol`
  - Input: `'ftp://example.com/file'`
  - Expected: `valid: false`
- [ ] `should reject malformed URL`
  - Input: `'not a url'`
  - Expected: `valid: false`
- [ ] `should follow redirects`
  - Input: URL returning 301
  - Expected: Final URL returned
- [ ] `should detect redirect loops`
  - Input: Circular redirects
  - Expected: Error
- [ ] `should resolve relative URL`
  - Input: `'/article/123'` with base
  - Expected: Absolute URL
- [ ] `should reject URLs on blocklist`
  - Input: Blocklisted domain
  - Expected: `valid: false`
- [ ] `should normalize URL`
  - Input: URL with tracking params
  - Expected: Clean normalized URL
- [ ] `should extract domain`
  - Input: Any valid URL
  - Expected: Domain + TLD

#### Rate Limiter (8 tests)
- [ ] `should respect requests per minute limit`
- [ ] `should respect requests per second limit`
- [ ] `should queue requests when limit reached`
- [ ] `should handle burst with token bucket`
- [ ] `should handle different limits per domain`
- [ ] `should reset limit after window expires`
- [ ] `should cancel queued requests on abort`
- [ ] `should track concurrent request count`

#### Retry Handler (8 tests)
- [ ] `should succeed on first attempt`
- [ ] `should retry with exponential backoff`
- [ ] `should respect max retries limit`
- [ ] `should send to dead letter queue`
- [ ] `should not retry on non-retryable error`
- [ ] `should apply jitter to backoff`
- [ ] `should reset backoff on success`
- [ ] `should handle async errors`

### Integration Tests

- [ ] **RSS -> Database**: Full flow from feed URL to stored items
- [ ] **RSS -> Database (idempotency)**: Duplicate run produces same result
- [ ] **RSS -> Database (update)**: Updated feed modifies existing items
- [ ] **HTML -> Database**: Full flow from article URL to stored item
- [ ] **HTML -> Database (paywall)**: Paywall detection and handling
- [ ] **Quality Scoring Pipeline**: Raw item -> scored item in DB
- [ ] **Quality Scoring (breaking news)**: Breaking boost applied
- [ ] **Quality Scoring (freshness decay)**: Score decays over time
- [ ] **Deduplication Pipeline**: Two similar items -> one primary
- [ ] **Deduplication (metadata merge)**: Sources merged correctly
- [ ] **Digest Generation**: Items -> formatted markdown digest
- [ ] **Digest (sorting)**: Items sorted by score descending
- [ ] **Telegram Formatting**: Items -> Telegram markdown
- [ ] **Telegram (escaping)**: Special chars escaped
- [ ] **Telegram (length limit)**: Message <= 4096 chars

### End-to-End Tests

- [ ] **Full Topic Scrape**: RSS feed -> collection -> scoring -> dedup -> storage
- [ ] **Collection to API**: Full pipeline through REST API
- [ ] **Error Recovery**: Failed source -> retry -> success
- [ ] **Error Recovery (dead letter)**: Permanent failure -> DLQ
- [ ] **Daily Digest**: Seed DB -> generate -> deliver to Telegram

### Quality Tests

- [ ] **Source scoring (tier-1)**: Score 85-100
- [ ] **Source scoring (tier-2)**: Score 60-84
- [ ] **Source scoring (tier-3)**: Score 20-50
- [ ] **Content scoring (primary vs secondary)**: Primary scores higher
- [ ] **Deduplication (false positives)**: Different articles not flagged
- [ ] **Deduplication (false negatives)**: Actual duplicates flagged
- [ ] **Freshness decay (1 day)**: ~0.95 factor
- [ ] **Freshness decay (7 days)**: ~0.55 factor
- [ ] **Freshness decay (30 days)**: ~0.10 factor
- [ ] **Breaking news detection**: Breaking keywords trigger boost

### Compliance Tests

- [ ] **robots.txt (allowed URL)**: URL not in Disallow -> allowed
- [ ] **robots.txt (disallowed URL)**: URL in Disallow -> blocked
- [ ] **robots.txt (crawl-delay)**: Directive parsed and respected
- [ ] **robots.txt (missing file)**: Default allow
- [ ] **Rate limit (RPS compliance)**: No two requests in same window
- [ ] **Rate limit (Retry-After)**: Header honored
- [ ] **User-Agent (identifiable)**: Contains JarvisHQ + contact
- [ ] **User-Agent (bot ID)**: Contains NewsAggregatorBot
- [ ] **PII sanitization (email)**: Emails replaced with [EMAIL REMOVED]
- [ ] **PII sanitization (phone)**: Phone numbers removed
- [ ] **PII sanitization (IP)**: IPs hashed
- [ ] **Attribution (URL preserved)**: Original URL maintained
- [ ] **Attribution (chain)**: Multi-source attribution chain
- [ ] **Attribution (output)**: Source credited in digest

### Performance Tests

- [ ] **Scrape 100 items**: Under 30 seconds
- [ ] **Handle 1000+ item feed**: No OOM, < 50MB overhead
- [ ] **Worker memory**: Under 256MB per worker
- [ ] **DB query speed**: Under 100ms for typical queries
- [ ] **Pipeline throughput**: 50 items/minute minimum
```

---

## Test File Organization

```
test/
├── setup.ts                    # Global test setup (MSW, DB, Redis)
├── factories.ts                # Test data factories
├── fixtures/
│   ├── valid-rss-2.0.xml       # RSS 2.0 feed sample
│   ├── valid-atom.xml          # Atom feed sample
│   ├── empty-feed.xml          # Empty RSS feed
│   ├── invalid-xml.xml         # Malformed XML
│   ├── large-feed-1000.xml     # 1000-item feed (generated)
│   ├── article-full.html       # Full article HTML
│   ├── article-paywall.html    # Paywalled article
│   ├── article-no-author.html  # Missing author
│   ├── article-no-date.html    # Missing date
│   ├── article-with-ads.html   # Article with ad containers
│   ├── article-spa.html        # JS-rendered article
│   ├── api-response.json       # Sample API response
│   ├── github-release.json     # GitHub releases API
│   ├── source-configs.json     # Source configurations
│   └── topic-configs.json      # Topic configurations
├── unit/
│   ├── adapters/
│   │   ├── rss.test.ts         # RSS adapter tests (10)
│   │   ├── api.test.ts         # API adapter tests (10)
│   │   ├── html.test.ts        # HTML adapter tests (10)
│   │   └── github.test.ts      # GitHub adapter tests (8)
│   ├── pipeline/
│   │   ├── quality-scorer.test.ts      # Quality tests (12)
│   │   ├── deduplication.test.ts       # Deduplication tests (10)
│   │   └── digest-generator.test.ts    # Digest tests (6)
│   └── utils/
│       ├── date-normalizer.test.ts     # Date tests (10)
│       ├── url-validator.test.ts       # URL tests (10)
│       ├── rate-limiter.test.ts        # Rate limit tests (8)
│       └── retry-handler.test.ts       # Retry tests (8)
├── integration/
│   ├── rss-database.test.ts           # RSS -> DB flow (3)
│   ├── html-database.test.ts          # HTML -> DB flow (2)
│   ├── quality-pipeline.test.ts       # Scoring pipeline (3)
│   ├── dedup-pipeline.test.ts         # Deduplication flow (2)
│   ├── digest-generation.test.ts      # Digest generation (2)
│   └── telegram-formatting.test.ts    # Telegram output (3)
├── e2e/
│   ├── full-pipeline.test.ts          # Complete pipeline (1)
│   ├── collection-to-api.test.ts      # API serving (1)
│   ├── error-recovery.test.ts         # Retry + DLQ (2)
│   └── daily-digest.test.ts           # Digest delivery (1)
├── quality/
│   ├── source-scoring.test.ts         # Source accuracy (3)
│   ├── content-scoring.test.ts        # Content detection (2)
│   ├── dedup-accuracy.test.ts         # False pos/neg (2)
│   ├── freshness-decay.test.ts        # Decay accuracy (3)
│   └── breaking-news.test.ts          # Breaking detection (2)
├── compliance/
│   ├── robots-txt.test.ts             # robots.txt parsing (4)
│   ├── rate-limit.test.ts             # Rate compliance (2)
│   ├── user-agent.test.ts             # UA compliance (2)
│   ├── pii-sanitization.test.ts       # PII removal (3)
│   └── attribution.test.ts            # Attribution (3)
└── performance/
    ├── scrape-speed.test.ts           # 100 items in 30s
    ├── large-feed.test.ts             # 1000+ items
    ├── memory-usage.test.ts           # < 256MB per worker
    ├── db-query-speed.test.ts         # < 100ms queries
    └── pipeline-throughput.test.ts    # 50 items/min
```

## Test Count Summary

| Category | Files | Tests | Target |
|----------|-------|-------|--------|
| Unit - Adapters | 4 | 38 | 38 |
| Unit - Pipeline | 3 | 28 | 28 |
| Unit - Utils | 4 | 36 | 36 |
| Integration | 6 | 15 | 15 |
| E2E | 4 | 5 | 5 |
| Quality | 5 | 12 | 10 |
| Compliance | 5 | 14 | 5 |
| Performance | 5 | 5 | 5 |
| **Total** | **36** | **153** | **142** |
