# Section D: Source Quality Scoring Rubric

> **System**: Jarvis HQ — Personal AI News Scraping Platform
> **Purpose**: Distinguish high-quality primary sources from SEO spam, rumours, and reposts; automate ranking and filtering with minimal human intervention.


## Table of Contents

- D1. Source-Level Scoring (out of 100)
  - Scoring Dimensions
  - D1.1 Source Type (15 points)
  - D1.2 Authority (15 points)
  - D1.3 Freshness (15 points)
  - D1.4 Attribution (10 points)
  - D1.5 Citation Quality (10 points)
  - D1.6 Update Frequency (10 points)
  - D1.7 Spam/SEO Risk (10 points, inverted)
  - D1.8 Commercial Bias (10 points, inverted)
  - D1.9 Rumour Risk (10 points, inverted)
  - D1.10 RSS/API Available (5 points)
  - Source Score Calculation
- D2. Content-Level Scoring (out of 100)
  - Scoring Dimensions
  - D2.1 Source Quality (20 points)
  - D2.2 Freshness (20 points)
  - D2.3 Primary Source (15 points)
  - D2.4 Factual Confidence (15 points)
  - D2.5 Relevance (15 points)
  - D2.6 Engagement Signal (10 points)
  - D2.7 Duplication (5 points)
  - Content Score Calculation
- D3. Composite Score Calculation
  - Formula
  - Why 40/60?
  - Pseudocode
  - Score Caching and Recalculation
- D4. Quality Tiers
  - Tier Determination
  - Score Distribution Target
- D5. Automatic Actions per Tier
  - Tier 1: Must-Show (90-100)
  - Tier 2: High Quality (75-89)
  - Tier 3: Medium (60-74)
  - Tier 4: Low (40-59)
  - Tier 5: Rejected (0-39)
- D6. Deduplication Strategy
  - Detection Pipeline
  - Stage 1: Exact URL Match
  - Stage 2: Canonical URL Match
  - Stage 3: Title Similarity (Levenshtein)
  - Stage 4: Content Hash Similarity (MinHash + LSH)
  - Stage 5: Semantic Similarity (Embedding Cosine)
  - Time Window Filter
  - Source Hierarchy: Which Version to Keep?
  - Duplicate Group Management
- D7. Freshness Decay
  - Decay Schedule
  - Decay Formula
  - Re-scoring Schedule
  - Special Handling: Evergreen Content
- D8. Quality Override Rules
  - D8.1 Whitelist Override
  - D8.2 Blacklist Override
  - D8.3 Manual Review Override
  - D8.4 Breaking News Boost
  - D8.5 Topic Urgency Boost
  - D8.6 Source Penalty Override
  - Override Application Order
- D9. Implementation Notes
  - Database Schema (Core Tables)
  - Score Recalculation Summary
  - Performance Considerations
- D10. Complete Example: End-to-End Scoring
  - Input: A new article
  - Step 1: Source Score Calculation
  - Step 2: Content Score Calculation
  - Step 3: Final Score
  - Step 4: Tier Determination
  - Step 5: Override Check
- Appendix A: Quick Reference Tables
  - Score Ranges by Tier
  - Decay Schedule at a Glance
  - Source Type Score Reference

---


---

## D1. Source-Level Scoring (out of 100)

Every source is scored once at registration and re-evaluated quarterly. The score is a weighted sum across 10 dimensions.

### Scoring Dimensions

| Dimension | Weight | Range | Description |
|-----------|--------|-------|-------------|
| Source Type | 15 | 0-15 | Nature of the publishing entity |
| Authority | 15 | 0-15 | Credibility and domain authority |
| Freshness | 15 | 0-15 | How often the source is updated |
| Attribution | 10 | 0-10 | Named authors, transparent ownership |
| Citation Quality | 10 | 0-10 | Links to primary sources, data backing |
| Update Frequency | 10 | 0-10 | Publishing cadence |
| Spam/SEO Risk | 10 | 0-10 | Inverted: 10 = low risk, 0 = high risk |
| Commercial Bias | 10 | 0-10 | Inverted: 10 = editorial independence, 0 = sponsored |
| Rumour Risk | 10 | 0-10 | Inverted: 10 = factual, 0 = speculative |
| RSS/API Available | 5 | 0-5 | Structured feed availability |

**Total**: 15 + 15 + 15 + 10 + 10 + 10 + 10 + 10 + 10 + 5 = **100**

---

### D1.1 Source Type (15 points)

Classification of the publishing entity's primary role.

| Score | Criteria | Examples |
|-------|----------|----------|
| **15** | Official company blog, press room, engineering blog, or changelog | `blog.google`, `cloudflare.com/blog`, `github.blog/changelog`, `react.dev/blog` |
| **12** | Established reputable news outlet with editorial standards | Reuters, AP, BBC, Financial Times, The Economist |
| **10** | Major tech/general news outlet with good editorial practices | TechCrunch, Ars Technica, The Verge, Bloomberg |
| **8** | Specialist industry publication with domain expertise | IEEE Spectrum, Nature, ACM Queue, O'Reilly Radar |
| **6** | Quality independent blog or newsletter with consistent output | Substack newsletters with named authors, Matt Levine, Benedict Evans |
| **5** | Aggregator with human curation and commentary | Hacker News featured links, Lobste.rs, TLDR Newsletter |
| **3** | Community-driven source (mixed quality, needs filtering) | Reddit threads, Stack Overflow blog, Dev.to |
| **2** | Syndicated content platform with variable quality | Medium (general), LinkedIn articles, self-published content |
| **0** | Unknown, pure aggregator, content farm, or SEO site | Auto-generated roundup sites, thin-content blogs, unnamed scrapers |

**Detection heuristics**:
- Check domain against known entity lists (corporate blog domains often match the product domain).
- Look for `/blog`, `/news`, `/press`, `/changelog`, `/releases` patterns.
- Inspect `og:type`, structured data (`@type: NewsArticle` vs `@type: BlogPosting`), and author metadata.
- Query WHOIS registration; corporate blogs often have matching registrant organization.

---

### D1.2 Authority (15 points)

The source's established credibility, measured by domain reputation, longevity, and recognition.

| Score | Criteria | Detection Method |
|-------|----------|-----------------|
| **15** | Official entity (the company/organization itself) | Domain matches known entity, verified by TLS cert org name, or listed on official website |
| **13** | Government or intergovernmental agency | `.gov`, `.gov.uk`, `.gc.ca`, `.gov.au`, `.eu`, `.govt.nz`; WHO, UN, EU institutions, OECD, IMF, World Bank |
| **12** | Tier-1 established outlet (>20 years, internationally recognized editorial standards) | Reuters, AP, AFP, Bloomberg, BBC, NHK, ARD/ZDF, ABC.au, CBC, RTÉ, NYT, WaPo, FT, The Economist, Le Monde, Der Spiegel, El País, Folha de S.Paulo, The Times of India, SCMP — cross-referenced with national press councils and award databases (Pulitzer/Peabody/Walkley/RTDNA/EPP/SOPA, regional equivalents) |
| **10** | Tier-2 established outlet (10-20 years, strong editorial reputation, regional or specialist) | Ars Technica (est. 1998), The Verge (est. 2011), AnandTech (est. 1997), 9to5Mac, MacRumors, The Register, Politico, Axios, Stratechery, Crikey, El Confidencial, Toyo Keizai |
| **8** | Tier-3 outlet (5-10 years, known in niche) | Dev.to, CSS-Tricks, Smashing Magazine |
| **6** | Recognized individual expert (consistent high-quality output >2 years) | Check byline frequency across articles, social following, citations from Tier 1/2 sources |
| **4** | Emerging outlet or individual (<2 years, good signals) | Domain age <2 years, but positive signals: original reporting, named authors, community engagement |
| **2** | Unknown or new source with no verifiable signals | Domain age <6 months, no author info, no backlinks from known sources |
| **0** | Known disreputable source | Listed in fact-checker databases (Snopes, PolitiFact flagged), or pattern-matches known clickbait domains |

**Implementation notes**:
```python
authority_signals = {
    "domain_age_years": float,          # via WHOIS
    "https_valid": bool,                # TLS cert validity
    "tls_org_match": bool,              # TLS cert org matches publisher claim
    "backlinks_from_tier1": int,        # Count of backlinks from score >=12 sources
    "factchecker_flagged": bool,        # In known disreputable list
    "awards_recognized": bool,          # Matches award database
    "first_seen": datetime,             # When Jarvis first indexed this source
}
```

---

### D1.3 Freshness (15 points)

How recently the source has published new content — an indicator of active curation.

| Score | Criteria |
|-------|----------|
| **15** | Published within the last 12 hours (live, active source) |
| **13** | Published within the last 24 hours |
| **11** | Published within the last 3 days |
| **9** | Published within the last 7 days |
| **7** | Published within the last 14 days |
| **5** | Published within the last 30 days |
| **3** | Published within the last 90 days |
| **1** | Published within the last 180 days |
| **0** | No content in the last 180 days (stale source) |

**Algorithm**:
```python
from datetime import datetime, timedelta

def freshness_score(last_publish: datetime) -> int:
    hours = (datetime.now() - last_publish).total_seconds() / 3600
    if hours <= 12:   return 15
    if hours <= 24:   return 13
    if hours <= 72:   return 11
    if hours <= 168:  return 9   # 7 days
    if hours <= 336:  return 7   # 14 days
    if hours <= 720:  return 5   # 30 days
    if hours <= 2160: return 3   # 90 days
    if hours <= 4320: return 1   # 180 days
    return 0
```

---

### D1.4 Attribution (10 points)

Whether the source clearly names authors and discloses ownership.

| Score | Criteria | Signals |
|-------|----------|---------|
| **10** | Named authors on every article; ownership clearly stated in About page; editorial policy published | `author.name` in structured data; About page lists parent company and editor-in-chief; editorial guidelines linked in footer |
| **8** | Named authors on most articles; ownership discoverable | Author bylines present; no explicit editorial policy but ownership in WHOIS or footer |
| **6** | Some articles have named authors; ownership implied | Mixed byline usage; copyright notice in footer |
| **4** | Rarely names authors; ownership unclear | Generic "Staff" or "Editorial Team" bylines; no ownership disclosure |
| **2** | No named authors; ownership unknown | No bylines at all; WHOIS privacy protection; no contact info |
| **0** | Deliberately anonymous with no verifiable identity | No author info, no ownership, no contact; possible front for undisclosed interests |

**Implementation**:
```python
attribution_signals = {
    "author_named": bool,               # Byline found
    "author_consistent": bool,          # Same authors recur across articles
    "author_verifiable": bool,          # Author has profile page, social presence, or external citations
    "about_page_exists": bool,          # /about or similar page
    "ownership_stated": bool,           # Parent company or owner named
    "editorial_policy_linked": bool,    # Editorial guidelines or ethics policy
    "contact_info_present": bool,       # Contact email/address verifiable
}
```

---

### D1.5 Citation Quality (10 points)

The extent to which the source links to and cites primary sources, data, and references.

| Score | Criteria | Signals |
|-------|----------|---------|
| **10** | Every claim backed by links to primary sources, official docs, or original data; FOIA or original research cited | Hyperlinks to `.gov`, `.edu`, official company docs, GitHub repos, arXiv papers; data tables with source URLs |
| **8** | Most claims have supporting links; references to original reporting | Multiple outbound links per article; links to press releases, official announcements |
| **6** | Some citations but incomplete; references to secondary sources | A few outbound links; mostly links to other news articles rather than primary sources |
| **4** | Sparse citations; relies on general knowledge without sourcing | 1-2 links per article; mostly internal links |
| **2** | Almost no external citations; mostly opinion/assertion | Zero or near-zero outbound links |
| **0** | Makes claims without any sourcing; known to fabricate sources | No links; claims contradict known facts; flagged by fact-checkers |

**Automated detection**:
```python
def citation_quality_score(article_html: str, metadata: dict) -> int:
    outbound_links = extract_external_links(article_html)
    primary_links = count_links_matching(outbound_links, primary_domains)
    citation_density = len(primary_links) / word_count(article_html)

    if citation_density >= 0.02 and primary_links >= 3:   return 10  # 1 link per 50 words, 3+ primary
    if citation_density >= 0.01 and primary_links >= 2:   return 8
    if primary_links >= 1:                                return 6
    if len(outbound_links) >= 2:                          return 4
    if len(outbound_links) >= 1:                          return 2
    return 0
```

---

### D1.6 Update Frequency (10 points)

How often the source publishes new content — distinguishes active newsrooms from dormant blogs.

| Score | Criteria | Posts per month (estimated) |
|-------|----------|----------------------------|
| **10** | Multiple times daily | 45+ |
| **8** | Daily | 20-44 |
| **7** | 4-6 times per week | 16-19 |
| **6** | 2-3 times per week | 8-15 |
| **4** | Weekly | 3-7 |
| **3** | Bi-weekly | 2 |
| **2** | Monthly | 1 |
| **1** | Quarterly or less | <1 |
| **0** | No regular pattern or sporadic (gap >90 days between posts) | 0 |

**Algorithm**: Compute the median gap between the last 20 published articles.
```python
def update_frequency_score(publish_dates: list[datetime]) -> int:
    if len(publish_dates) < 3:
        return 0  # insufficient data

    gaps = sorted(
        (publish_dates[i] - publish_dates[i+1]).days
        for i in range(len(publish_dates) - 1)
    )
    median_gap = gaps[len(gaps) // 2]
    posts_per_month = 30 / max(median_gap, 1)

    if posts_per_month >= 45: return 10
    if posts_per_month >= 20: return 8
    if posts_per_month >= 16: return 7
    if posts_per_month >= 8:  return 6
    if posts_per_month >= 3:  return 4
    if posts_per_month >= 2:  return 3
    if posts_per_month >= 1:  return 2
    if posts_per_month >= 0.33: return 1
    return 0
```

---

### D1.7 Spam/SEO Risk (10 points, inverted)

Assesses the likelihood that content is optimized for search engines rather than human readers. Higher score = lower spam risk.

| Score | Criteria | Detection Signals |
|-------|----------|-------------------|
| **10** | Purely editorial content with no SEO manipulation | No keyword stuffing; natural language; no clickbait headlines; original reporting |
| **8** | Slight SEO awareness but content-first approach | Proper meta descriptions, clean URL structure; headlines descriptive, not sensational |
| **6** | Moderate SEO optimization, still readable | Keyword-rich headings; some listicle formatting; affiliate links present but labeled |
| **4** | Heavy SEO optimization; content quality suffers | Keyword stuffing detected (>3% keyword density); thin content (<300 words); auto-generated sections |
| **2** | Primarily SEO-driven; minimal original value | Clickbait headlines (pattern: "X things you won't believe", "This ONE trick"); scraped/rewritten content |
| **0** | Pure SEO spam | AI-generated gibberish with keywords; doorway pages; no original content; domain blocked by uBlock filters |

**Spam detection heuristics**:
```python
spam_signals = {
    "keyword_density": float,           # >3% suspicious
    "clickbait_score": float,           # 0-1 from headline pattern matching
    "word_count": int,                  # <300 suspicious
    "affiliate_links": int,             # count; >5 suspicious
    "ai_generated_likelihood": float,   # from stylometry classifier
    "duplicate_content_ratio": float,   # % matching known sources via MinHash
    "headline_caps_ratio": float,       # >30% caps suspicious
    "exclamation_count": int,           # >3 in headline suspicious
    "superlative_density": float,       # "best", "ultimate", "shocking" density
}

# Pseudocode
def spam_seo_score(html, text, headline) -> int:
    risk = 0
    if keyword_density(text) > 0.03:        risk += 2
    if len(text.split()) < 300:             risk += 2
    if is_clickbait(headline):              risk += 2
    if ai_likelihood(text) > 0.7:           risk += 2
    if duplicate_ratio(text) > 0.6:         risk += 2
    if affiliate_count(html) > 5:           risk += 1
    return max(0, 10 - risk)                # invert: higher = lower risk
```

---

### D1.8 Commercial Bias (10 points, inverted)

Assesses editorial independence from commercial interests. Higher score = less bias.

| Score | Criteria | Detection Signals |
|-------|----------|-------------------|
| **10** | No commercial relationship disclosed; nonprofit or publicly funded | BBC (UK license fee), NPR, government sources, `.edu` research; no ads, no sponsored content |
| **8** | Ad-supported but strong firewall between editorial and ads | Clear "Sponsored" labels; editorial independence statement; ad content separated |
| **6** | Ad-supported with minor commercial influence | Native advertising present but labeled; some sponsored content |
| **4** | Noticeable commercial influence | Frequent sponsored content; product reviews with undisclosed affiliate relationships |
| **2** | Heavy commercial influence | Pay-for-play coverage common; content appears driven by advertiser interests; "partner content" dominant |
| **0** | Pure commercial publication or disguised advertising | All content sponsored; no editorial independence; brand-owned "media" that only promotes products; multi-level marketing content |

**Detection signals**:
```python
commercial_signals = {
    "sponsored_content_present": bool,
    "sponsored_ratio": float,           # sponsored / total articles
    "affiliate_disclosure": bool,       # FTC disclosure present
    "advertiser_wall": bool,            # clear separation statement
    "ownership_by_commercial_entity": bool,  # Owned by reviewed companies
    "paywall_type": str,                # "none", "soft", "hard", "ad-supported"
    "sponsored_labels_used": bool,      # "Sponsored", "Paid Content", "Partner"
}
```

---

### D1.9 Rumour Risk (10 points, inverted)

Assesses likelihood of publishing unverified claims, speculation, or rumours. Higher score = more factual.

| Score | Criteria | Detection Signals |
|-------|----------|-------------------|
| **10** | Rigorous fact-checking; corrections published promptly; primary source verification | Correction policy linked; fact-checking methodology stated; uses "according to X" consistently; no retracted articles |
| **8** | Generally factual with rare errors; sources cited | Mostly verified claims; occasional anonymous sources but contextualized |
| **6** | Mix of fact and analysis; some speculation labeled as such | Opinion pieces clearly marked; analysis distinguished from reporting |
| **4** | Frequent use of anonymous sources; speculative framing | "Sources say" without attribution; unnamed "insiders"; hedging language dominates |
| **2** | Often publishes rumours and unverified claims | Pattern of stories later debunked; "reportedly", "allegedly" used as shields for false claims |
| **0** | Known rumour mill; disinformation or conspiracy content | Multiple fact-checker flags; articles regularly retracted; known misinformation pattern; satire not labeled as such |

**Language pattern detection**:
```python
rumour_signals = {
    "hedging_words_ratio": float,       # "reportedly", "allegedly", "sources say"
    "anonymous_source_count": int,      # per article
    "verified_claim_ratio": float,      # claims with primary source / total claims
    "correction_history": int,          # corrections published per quarter
    "factchecker_flags": int,           # negative flags from fact-check databases
    "speculative_language_ratio": float,  # "may", "might", "could", "possibly"
    "all_caps_rumour_words": int,       # "BREAKING" without verification, "EXCLUSIVE"
}

def rumour_risk_score(signals: dict) -> int:
    risk = 0
    if signals["hedging_words_ratio"] > 0.05:       risk += 1
    if signals["anonymous_source_count"] > 2:       risk += 2
    if signals["verified_claim_ratio"] < 0.5:       risk += 2
    # Corrections are evidence of editorial rigor, not sloppiness. Established
    # newsrooms (Reuters, NYT, WaPo) publish many corrections because they
    # self-correct. Only penalize uncorrected retractions or factchecker-flagged
    # patterns of false reporting.
    if signals.get("uncorrected_retractions", 0) > 3: risk += 1
    if signals["factchecker_flags"] > 0:             risk += 3
    if signals["speculative_language_ratio"] > 0.08: risk += 1
    return max(0, 10 - risk)
```

---

### D1.10 RSS/API Available (5 points)

Whether the source provides a structured, machine-readable feed.

| Score | Criteria |
|-------|----------|
| **5** | Official REST/GraphQL API with documentation (GitHub API, Reddit API, HN API) |
| **4** | RSS/Atom feed with full content and proper metadata (author, date, categories) |
| **3** | RSS/Atom feed with partial content or basic metadata |
| **2** | JSON Feed or sitemap.xml with article listings |
| **1** | Social media API bridge (Twitter/X API, Mastodon) |
| **0** | No structured feed; requires HTML scraping only |

---

### Source Score Calculation

```python
class SourceScore:
    WEIGHTS = {
        "source_type":     15,
        "authority":       15,
        "freshness":       15,
        "attribution":     10,
        "citation":        10,
        "update_freq":     10,
        "spam_risk":       10,   # inverted
        "commercial_bias": 10,   # inverted
        "rumour_risk":     10,   # inverted
        "feed_available":   5,
    }

    @staticmethod
    def calculate(scores: dict[str, int]) -> int:
        """
        Calculate weighted source score.
        Each score is 0-N as defined above.
        Returns integer 0-100.
        """
        total = 0
        for dimension, weight in SourceScore.WEIGHTS.items():
            raw = scores.get(dimension, 0)
            max_points = weight  # scores already at their natural scale
            clamped = max(0, min(raw, max_points))
            total += clamped
        return int(total)
```

**Example calculation**:
| Dimension | Raw Score | Max | Weighted |
|-----------|-----------|-----|----------|
| Source Type | 12 | 15 | 12 |
| Authority | 10 | 15 | 10 |
| Freshness | 11 | 15 | 11 |
| Attribution | 8 | 10 | 8 |
| Citation Quality | 8 | 10 | 8 |
| Update Frequency | 8 | 10 | 8 |
| Spam/SEO Risk | 9 | 10 | 9 |
| Commercial Bias | 8 | 10 | 8 |
| Rumour Risk | 9 | 10 | 9 |
| RSS/API | 4 | 5 | 4 |
| **Total** | | **100** | **87** |

---

## D2. Content-Level Scoring (out of 100)

Every individual article receives a score at ingestion time. The score combines inherited source quality with article-specific signals.

### Scoring Dimensions

| Dimension | Weight | Range | Description |
|-----------|--------|-------|-------------|
| Source Quality | 20 | 0-20 | Inherited from source score |
| Freshness | 20 | 0-20 | How recently published |
| Primary Source | 15 | 0-15 | Original reporting vs repost |
| Factual Confidence | 15 | 0-15 | Verified facts vs speculation |
| Relevance | 15 | 0-15 | Match strength against topic filters |
| Engagement Signal | 10 | 0-10 | Community interest indicators |
| Duplication | 5 | 0-5 | Originality of content |

**Total**: 20 + 20 + 15 + 15 + 15 + 10 + 5 = **100**

---

### D2.1 Source Quality (20 points)

Directly inherited from the source-level score.

| Score | Criteria |
|-------|----------|
| **20** | Source score 95-100 (Tier 1 source) |
| **18** | Source score 85-94 |
| **16** | Source score 75-84 (Tier 2 source) |
| **14** | Source score 65-74 |
| **12** | Source score 55-64 (Tier 3 source) |
| **10** | Source score 45-54 |
| **8** | Source score 35-44 (Tier 4 source) |
| **5** | Source score 20-34 |
| **2** | Source score 1-19 (Tier 5 source) |
| **0** | Source score 0 or source not rated |

```python
def source_quality_inherited(source_score: int) -> int:
    if source_score >= 95: return 20
    if source_score >= 85: return 18
    if source_score >= 75: return 16
    if source_score >= 65: return 14
    if source_score >= 55: return 12
    if source_score >= 45: return 10
    if source_score >= 35: return 8
    if source_score >= 20: return 5
    if source_score >= 1:  return 2
    return 0
```

---

### D2.2 Freshness (20 points)

Time decay since publication. The newer, the higher the score.

| Score | Age |
|-------|-----|
| **20** | < 1 hour (breaking) |
| **19** | 1-6 hours |
| **17** | 6-24 hours |
| **15** | 1-3 days |
| **12** | 3-7 days |
| **10** | 1-2 weeks |
| **7** | 2-4 weeks |
| **5** | 1-2 months |
| **3** | 2-6 months |
| **1** | 6-12 months |
| **0** | > 12 months (archival) |

```python
def freshness_score(publish_date: datetime, now: datetime = None) -> int:
    if now is None:
        now = datetime.now(publish_date.tzinfo)
    hours = max(0, (now - publish_date).total_seconds() / 3600)

    if hours < 1:    return 20
    if hours < 6:    return 19
    if hours < 24:   return 17
    if hours < 72:   return 15
    if hours < 168:  return 12
    if hours < 336:  return 10
    if hours < 720:  return 7
    if hours < 1440: return 5
    if hours < 4320: return 3
    if hours < 8760: return 1
    return 0
```

---

### D2.3 Primary Source (15 points)

Whether this article is the original reporting or a repost/summary.

| Score | Criteria | Detection Signals |
|-------|----------|-------------------|
| **15** | Original reporting: journalist on scene, FOIA request, original data analysis, exclusive interview | Bylined journalist with unique quotes; contains "Jarvis analysis" or original data; links only to background, not to same story elsewhere |
| **12** | Original synthesis or first-hand account | Editorial combining multiple sources with original framing; developer's personal blog post; official announcement posted first here |
| **9** | Significant original contribution with some aggregated content | Original commentary on news with substantial analysis; 30%+ original content |
| **6** | Syndicated from primary source with minor changes | "Originally published at X" notice; canonical URL points elsewhere; author credit retained but not original publisher |
| **4** | Summary or rewrite of primary source | Heavily quotes from another article; no new reporting; similar structure to original |
| **2** | Thin rewrite with minimal value | AI summary of another article; mostly paraphrased with no attribution to original journalist |
| **0** | Identical duplicate or scraped content | Exact text match to another article; canonical URL is different domain; no attribution |

**Detection algorithm**:
```python
def primary_source_score(article: Article, all_articles: list[Article]) -> int:
    signals = {
        "has_exclusive_quote": bool,        # Contains " according to X " or direct quote from named person
        "has_original_data": bool,          # Contains data tables, charts, FOIA docs
        "canonical_is_self": bool,          # rel=canonical points to this URL
        "syndication_notice": bool,         # "Originally published" or "Reprinted with permission"
        "content_similarity_max": float,    # Max similarity to other articles via MinHash
        "first_seen_among_dups": bool,      # Earliest publish date among duplicates
        "journalist_byline": bool,          # Named journalist, not wire service
    }

    if signals["content_similarity_max"] > 0.85:
        if not signals["first_seen_among_dups"]:
            return 0  # duplicate
        return 6    # syndicated but first

    if signals["has_exclusive_quote"] or signals["has_original_data"]:
        if signals["content_similarity_max"] < 0.2:
            return 15  # original reporting
        return 12    # original synthesis

    if signals["syndication_notice"]:
        return 6

    if signals["content_similarity_max"] > 0.6:
        return 4   # summary

    if signals["content_similarity_max"] > 0.4:
        return 9   # significant contribution

    return 12  # default: treat as original if no close matches
```

---

### D2.4 Factual Confidence (15 points)

How confident the system is that the article's claims are factual.

| Score | Criteria | Detection Signals |
|-------|----------|-------------------|
| **15** | All claims verifiable against primary sources; official announcement; release notes | Links to official docs, GitHub releases, SEC filings, press releases from the entity itself |
| **13** | Strongly factual; multiple corroborating sources | 3+ independent sources confirm claims; data-driven with cited methodology |
| **11** | Factual with minor uncertainty | 2 independent sources; some claims attributed to anonymous but reasonable |
| **8** | Mix of fact and analysis | Clearly separated fact and opinion sections; analysis labeled as such |
| **5** | Significant speculation | Multiple "may", "might", "sources say" without corroboration; future predictions without basis |
| **3** | Primarily speculative | Rumour-based; no named sources; claims about future events without evidence |
| **0** | Disinformation or known false claims | Contradicted by fact-checkers; matches known misinformation patterns; logically impossible claims |

```python
def factual_confidence_score(article: Article) -> int:
    signals = extract_factual_signals(article)

    # Quick disqualifiers
    if signals["factchecker_contradicted"]:
        return 0
    if signals["known_false_pattern"]:
        return 0

    if signals["official_announcement"]:
        return 15
    if signals["corroborating_sources"] >= 3 and signals["data_driven"]:
        return 13
    if signals["corroborating_sources"] >= 2:
        return 11
    if signals["corroborating_sources"] >= 1:
        return 8
    if signals["hedging_ratio"] > 0.1 and signals["anonymous_sources"] > 1:
        return 5
    if signals["hedging_ratio"] > 0.15:
        return 3
    if signals["speculative_framing"]:
        return 5

    return 8  # default to medium confidence
```

---

### D2.5 Relevance (15 points)

How well the article matches the user's configured topic filters and keywords.

| Score | Criteria |
|-------|----------|
| **15** | Exact topic match: primary subject aligns with configured topic; title and body highly focused |
| **13** | Strong match: major section of article on target topic; title mentions key terms |
| **11** | Good match: significant coverage of target topic within article |
| **8** | Moderate match: mentions target topic but not central |
| **5** | Weak match: tangential reference to topic |
| **2** | Barely relevant: keyword match in passing |
| **0** | No relevance: does not match any configured topic |

**Algorithm**:
```python
def relevance_score(article: Article, topic_config: TopicConfig) -> int:
    # Topic keywords (user-configured)
    primary_keywords = topic_config.keywords        # e.g., ["AI", "artificial intelligence"]
    secondary_keywords = topic_config.secondary     # e.g., ["machine learning", "neural network"]
    exclusion_keywords = topic_config.exclude       # e.g., ["AI-generated art"]

    title = article.title.lower()
    body = article.body_text.lower()
    summary = article.summary.lower()

    # Exclusion check first
    for ex in exclusion_keywords:
        if ex.lower() in title or (ex.lower() in body and body.count(ex.lower()) >= 2):
            return 0

    # Title match (strong signal)
    title_primary = sum(1 for kw in primary_keywords if kw.lower() in title)
    title_secondary = sum(1 for kw in secondary_keywords if kw.lower() in title)

    # Body match
    body_primary = sum(1 for kw in primary_keywords if kw.lower() in body)
    body_secondary = sum(1 for kw in secondary_keywords if kw.lower() in body)

    # TF-IDF-style relevance
    # NOTE on embeddings: topic_semantic_similarity() is implementation-dependent.
    # The reference deployment uses sentence-transformers/all-MiniLM-L6-v2 (384-dim)
    # for both article and topic_config embeddings, but ANY consistent embedding
    # model works as long as both the article and the topic config are encoded
    # with the SAME model. Cosine similarity is the recommended distance metric.
    # If you don't have an embedding model available, set the semantic similarity
    # term to 0 and rely on the keyword counts — accuracy drops but the scorer
    # still works.
    relevance = (
        title_primary * 4.0 + title_secondary * 2.0 +
        min(body_primary, 10) * 1.0 + min(body_secondary, 10) * 0.5 +
        topic_semantic_similarity(article.embedding, topic_config.embedding) * 3.0
    )

    if relevance >= 15:   return 15
    if relevance >= 12:   return 13
    if relevance >= 9:    return 11
    if relevance >= 6:    return 8
    if relevance >= 3:    return 5
    if relevance >= 1:    return 2
    return 0
```

---

### D2.6 Engagement Signal (10 points)

Community interest indicators that suggest the content is important or worth reading.

| Score | Criteria | Signals |
|-------|----------|---------|
| **10** | Viral/high engagement | 1000+ shares on primary platform; 500+ comments; on HN front page >4 hours |
| **8** | Strong engagement | 100-999 shares; 50-499 comments; HN front page >2 hours |
| **6** | Moderate engagement | 20-99 shares; 10-49 comments; notable Reddit discussion (>50 upvotes) |
| **4** | Some engagement | 5-19 shares; 2-9 comments; any social activity |
| **2** | Minimal engagement | 1-4 shares or 1 comment |
| **0** | No engagement or unable to measure | Zero social signals; paywalled article with no public metrics |

**Platform-specific signals**:

> **Note on X/Twitter signals.** The architecture excludes Twitter/X HTML
> scraping (ToS), and most operators do not have paid X API access. Twitter
> engagement weights below are kept for completeness but should be treated as
> usually-zero. Prefer HN/Lobsters/Bluesky/Mastodon, which are accessible via
> official APIs or RSS feeds. If you need to add a new platform's signal, add
> the lambda here with a conservative cap and document the data source.

```python
engagement_weights = {
    # High-confidence, openly accessible
    "hn_score": lambda s: min(s / 50, 5),           # HN points, cap at 250 points = 5
    "hn_comment_count": lambda c: min(c / 20, 3),   # HN comments, cap at 60
    "hn_hours_frontpage": lambda h: min(h, 2),      # Hours on front page, cap at 2
    "lobsters_score": lambda s: min(s / 30, 3),     # Lobste.rs points, cap at 90
    "reddit_upvotes": lambda u: min(u / 100, 4),    # Reddit upvotes (RSS/API), cap at 400
    "reddit_comments": lambda c: min(c / 50, 3),    # Reddit comments, cap at 150
    "mastodon_boosts": lambda b: min(b / 50, 2),    # Mastodon boosts, cap at 100
    "bluesky_likes": lambda l: min(l / 100, 2),     # Bluesky likes (open API), cap at 200
    "bluesky_reposts": lambda r: min(r / 50, 2),    # Bluesky reposts, cap at 100

    # Available only via paid API; default to 0 if not configured
    "twitter_retweets": lambda r: min(r / 200, 1),  # X API only; reduced cap (was 3, now 1)
    "twitter_likes":    lambda l: min(l / 500, 1),  # X API only; reduced cap (was 2, now 1)

    # Other social
    "linkedin_shares":  lambda s: min(s / 100, 2),  # LinkedIn (often inaccessible), cap at 200
}

def engagement_score(signals: dict) -> int:
    total = 0
    for signal, weight_fn in engagement_weights.items():
        value = signals.get(signal, 0)
        total += weight_fn(value)

    if total >= 8:  return 10
    if total >= 5:  return 8
    if total >= 3:  return 6
    if total >= 1:  return 4
    if total >= 0.2: return 2
    return 0
```

---

### D2.7 Duplication (5 points)

Whether this is the original article or a duplicate/repost.

| Score | Criteria |
|-------|----------|
| **5** | Original article: no duplicates found; first instance of this content |
| **3** | Minor overlap: some shared text (quotes, press release text) but majority original |
| **1** | Known duplicate: same article published elsewhere, canonical URL different |
| **0** | Identical duplicate: exact or near-exact copy of another indexed article |

**Dedup integration**: This dimension directly ties to the deduplication system (see D6).
```python
def duplication_score(article: Article, dup_group: DuplicateGroup) -> int:
    if dup_group is None:
        return 5  # no duplicates found

    if article.canonical_url == article.url:
        return 5  # canonical is self = original

    if article == dup_group.original:
        return 5

    similarity = dup_group.similarity_to_original(article)
    if similarity > 0.95:
        return 0  # identical
    if similarity > 0.75:
        return 1  # known duplicate
    if similarity > 0.5:
        return 3  # minor overlap

    return 5  # different enough to be original
```

---

### Content Score Calculation

```python
class ContentScore:
    WEIGHTS = {
        "source_quality":    20,
        "freshness":         20,
        "primary_source":    15,
        "factual_confidence": 15,
        "relevance":         15,
        "engagement":        10,
        "duplication":        5,
    }

    @staticmethod
    def calculate(scores: dict[str, int]) -> int:
        total = 0
        for dimension, weight in ContentScore.WEIGHTS.items():
            raw = scores.get(dimension, 0)
            max_points = weight
            clamped = max(0, min(raw, max_points))
            total += clamped
        return int(total)
```

---

## D3. Composite Score Calculation

The final article score combines source trustworthiness and content quality.

### Formula

```
Source_Score  = weighted_sum of source dimensions (0-100)
Content_Score = weighted_sum of content dimensions (0-100)
Final_Score   = round(Source_Score * 0.4 + Content_Score * 0.6)
```

### Why 40/60?

- **Source (40%)**: A bad source can occasionally publish good content, but the baseline trustworthiness matters. This prevents a single well-written article from a spam site from getting a high score.
- **Content (60%)**: A great source can publish low-value content (press release reprints, off-topic articles). The content-specific signals ensure relevance and freshness get proper weight.

### Pseudocode

```python
def calculate_final_score(article: Article, source: Source) -> int:
    source_score = SourceScore.calculate(source.dimension_scores)
    content_scores = {
        "source_quality":     source_quality_inherited(source_score),   # 0-20
        "freshness":          freshness_score(article.published_at),    # 0-20
        "primary_source":     primary_source_score(article),            # 0-15
        "factual_confidence": factual_confidence_score(article),        # 0-15
        "relevance":          relevance_score(article, topic_config),   # 0-15
        "engagement":         engagement_score(article.signals),        # 0-10
        "duplication":        duplication_score(article, dup_group),    # 0-5
    }
    content_score = ContentScore.calculate(content_scores)

    final = round(source_score * 0.4 + content_score * 0.6)
    return max(0, min(final, 100))
```

### Score Caching and Recalculation

| Event | Action |
|-------|--------|
| New article ingested | Calculate full score immediately |
| Source re-evaluated (quarterly) | Recalculate all recent articles (last 30 days) from that source |
| User changes topic filters | Recalculate relevance for all unarchived articles |
| Freshness threshold crossed (hourly cron) | Apply decay to all articles older than their current bracket |
| Engagement data updated | Recalculate engagement score; adjust final score |
| Deduplication match found | Recalculate duplication score for affected articles |

---

## D4. Quality Tiers

Articles are bucketed into 5 tiers based on their Final Score.

| Tier | Score Range | Label | Description |
|------|-------------|-------|-------------|
| **Tier 1** | 90-100 | Must-Show | Authoritative, fresh, primary reporting. Always include. |
| **Tier 2** | 75-89 | High Quality | Reliable, relevant, well-sourced. Standard inclusion. |
| **Tier 3** | 60-74 | Medium | Acceptable with context. Include but flag. |
| **Tier 4** | 40-59 | Low | Borderline quality. Hold for review. |
| **Tier 5** | 0-39 | Rejected | Spam, rumour, irrelevant. Reject and log. |

### Tier Determination

```python
def determine_tier(final_score: int) -> int:
    if final_score >= 90: return 1
    if final_score >= 75: return 2
    if final_score >= 60: return 3
    if final_score >= 40: return 4
    return 5
```

### Score Distribution Target

For a healthy source mix, target distribution:
- Tier 1: 5-10% of daily ingest
- Tier 2: 30-40%
- Tier 3: 25-35%
- Tier 4: 10-20%
- Tier 5: 5-15% (should be rejected at ingestion)

---

## D5. Automatic Actions per Tier

### Tier 1: Must-Show (90-100)

| Action | Detail |
|--------|--------|
| **Auto-include** | Immediately included in daily digest |
| **Priority rank** | Placed at the top of the digest, sorted by freshness within tier |
| **Notification** | Push notification if user enabled breaking news alerts |
| **Highlight** | Marked with a visual indicator (star, border, or accent) |
| **Skip review** | No human review required |
| **Archive rule** | Keep indefinitely; never auto-delete |

### Tier 2: High Quality (75-89)

| Action | Detail |
|--------|--------|
| **Standard inclusion** | Included in daily digest with standard treatment |
| **Rank position** | Below Tier 1, sorted by score then freshness |
| **Notification** | No push unless engagement signals are very high |
| **Review** | Optional spot-check (1% sample rate for quality assurance) |
| **Archive rule** | Keep for 90 days, then move to cold storage |

### Tier 3: Medium (60-74)

| Action | Detail |
|--------|--------|
| **Conditional inclusion** | Included in digest with a "flag for review" marker |
| **Visual indicator** | Subtle badge: "Review recommended" or similar |
| **Review trigger** | Added to review queue; human reviewer has 24h to approve/remove |
| **If approved** | Treated as Tier 2 going forward; score locked at 76 |
| **If rejected** | Downgraded to Tier 5; logged for source quality feedback |
| **Auto-approve** | If no reviewer action within 24h and no negative signals, auto-approve |

### Tier 4: Low (40-59)

| Action | Detail |
|--------|--------|
| **Hold in queue** | Not included in digest |
| **Review required** | Placed in manual review queue with priority indicator |
| **Reviewer options** | Approve (bump to Tier 3 at 65 points), Reject (Tier 5), or Request More Info |
| **Auto-reject** | If no reviewer action within 72h, auto-reject to Tier 5 |
| **Source feedback** | If >5 Tier 4 articles from same source in 30 days, trigger source re-evaluation |

### Tier 5: Rejected (0-39)

| Action | Detail |
|--------|--------|
| **Reject** | Not included in digest; not shown to user |
| **Log** | Full metadata logged: URL, source, score breakdown, rejection reason |
| **Deduplication** | Used as a match target for future deduplication |
| **Source impact** | Contributes to source's rolling quality average; 10+ Tier 5 in 30 days triggers source re-evaluation |
| **Blacklist check** | If source generates >50% Tier 5 over 30 days, flag for blacklist review |
| **Notification** | None (silent rejection) |

```python
class TierAction:
    ACTIONS = {
        1: {"include": True,  "priority": "top",     "review": False, "notification": True},
        2: {"include": True,  "priority": "normal",  "review": False, "notification": False},
        3: {"include": True,  "priority": "flagged", "review": True,  "notification": False},
        4: {"include": False, "priority": "queue",   "review": True,  "notification": False},
        5: {"include": False, "priority": "reject",  "review": False, "notification": False},
    }

    @staticmethod
    def apply(article: Article, tier: int):
        action = TierAction.ACTIONS[tier]
        article.included_in_digest = action["include"]
        article.priority = action["priority"]
        if action["review"]:
            ReviewQueue.add(article, deadline_hours={3: 24, 4: 72}.get(tier))
        if action["notification"]:
            NotificationService.push(article)
```

---

## D6. Deduplication Strategy

### Detection Pipeline

Dedup runs at ingestion time, before scoring. The pipeline has 5 stages, ordered from cheapest to most expensive:

```
Stage 1: URL Match (instant)      --> 99% of duplicates caught here
Stage 2: Canonical URL Match      --> catches syndicated content
Stage 3: Title Similarity         --> catches rewritten titles
Stage 4: Content Hash Similarity  --> catches near-identical rewrites
Stage 5: Semantic Similarity      --> catches substantial rewrites
```

### Stage 1: Exact URL Match

```python
def stage1_url_match(new_url: str, existing_urls: set[str]) -> bool:
    """O(1) lookup via Bloom filter + exact set."""
    normalized = normalize_url(new_url)
    # Strip tracking parameters
    normalized = remove_utm_params(normalized)
    return normalized in existing_urls
```

**URL normalization rules**:
- Strip `utm_*`, `fbclid`, `gclid`, `ref` parameters
- Remove `#fragment` (unless it's a SPA route)
- Convert to lowercase
- Remove trailing slash
- Resolve redirects (follow HTTP 301/302 to final URL)

### Stage 2: Canonical URL Match

```python
def stage2_canonical_match(new_article: Article, index: ArticleIndex) -> bool:
    canonical = new_article.canonical_url or new_article.url
    canonical = normalize_url(canonical)
    matches = index.find_by_canonical(canonical)
    return len(matches) > 0
```

### Stage 3: Title Similarity (Levenshtein)

```python
from Levenshtein import ratio

def stage3_title_similarity(new_title: str, candidates: list[Article]) -> list[DuplicateMatch]:
    normalized_new = normalize_title(new_title)  # lowercase, strip punctuation
    matches = []
    for candidate in candidates:
        # Only compare against articles published within 48h window
        if hours_apart(new_title.published, candidate.published) > 48:
            continue
        sim = ratio(normalized_new, normalize_title(candidate.title))
        if sim >= 0.80:
            matches.append(DuplicateMatch(candidate, sim, method="title_levenshtein"))
    return matches
```

**Title normalization**:
- Lowercase
- Remove punctuation except hyphens
- Strip leading "BREAKING: ", "UPDATE: ", "EXCLUSIVE: " prefixes
- Remove publication name suffixes ("- TechCrunch", "| The Verge")
- Standardize whitespace

### Stage 4: Content Hash Similarity (MinHash + LSH)

```python
def stage4_content_hash(new_article: Article, index: ArticleIndex) -> list[DuplicateMatch]:
    # Generate 5-gram MinHash signature
    tokens = tokenize(new_article.body_text)
    signature = minhash_signature(tokens, n=5, num_hashes=128)

    # LSH lookup for candidates within 48h window
    candidates = index.lsh_query(signature, time_window_hours=48)

    matches = []
    for candidate in candidates:
        candidate_sig = index.get_signature(candidate.id)
        jaccard = signature.jaccard(candidate_sig)
        if jaccard >= 0.75:
            matches.append(DuplicateMatch(candidate, jaccard, method="minhash_lsh"))
    return matches
```

**MinHash parameters**:
- Shingle size: 5 words
- Hash functions: 128 (for ~0.95 accuracy)
- LSH bands: 16 bands x 8 rows
- Jaccard threshold: 0.75 (catches near-duplicates)

### Stage 5: Semantic Similarity (Embedding Cosine)

```python
import numpy as np

def stage5_semantic_similarity(new_article: Article, candidates: list[Article]) -> list[DuplicateMatch]:
    new_embedding = embedding_model.encode(new_article.title + " " + new_article.first_500_chars)

    matches = []
    for candidate in candidates:
        cand_embedding = index.get_embedding(candidate.id)
        similarity = cosine_similarity(new_embedding, cand_embedding)
        if similarity >= 0.88:  # high threshold to avoid false positives
            matches.append(DuplicateMatch(candidate, similarity, method="semantic_cosine"))
    return matches
```

### Time Window Filter

Dedup only compares articles within a **48-hour window** (configurable by source). Articles older than the window are assumed distinct.

```python
DEDUP_TIME_WINDOW_HOURS = 48

def within_dedup_window(a: datetime, b: datetime) -> bool:
    return abs((a - b).total_seconds()) / 3600 <= DEDUP_TIME_WINDOW_HOURS
```

### Source Hierarchy: Which Version to Keep?

When duplicates are detected, the system selects the **single best version** using this hierarchy:

```python
SOURCE_PRIORITY_RANK = {
    "official_blog": 1,        # Official company/entity blog
    "tier1_news": 2,           # Reuters, AP, BBC
    "tier2_news": 3,           # TechCrunch, Ars Technica
    "specialist_pub": 4,       # IEEE, ACM, O'Reilly
    "quality_indie": 5,        # Named expert newsletters
    "aggregator_curated": 6,   # HN, TLDR
    "community": 7,            # Reddit, Dev.to
    "syndication_platform": 8, # Medium, LinkedIn
}

def select_best_version(duplicates: list[Article]) -> Article:
    """Select the highest-quality version of a duplicate group."""
    scored = []
    for article in duplicates:
        score = (
            SOURCE_PRIORITY_RANK.get(article.source.source_type, 99),
            -article.source_score,          # higher source score = better
            -article.freshness_score,        # fresher = better
            -article.content_length,         # longer (more complete) = better
            article.has_full_text,           # full text > summary
        )
        scored.append((score, article))

    scored.sort()
    return scored[0][1]  # best version
```

### Duplicate Group Management

```python
@dataclass
class DuplicateGroup:
    group_id: str
    original: Article                    # best version (kept)
    duplicates: list[Article]            # inferior versions (deduped)
    canonical_url: str                   # URL of original
    similarity_scores: dict[str, float]  # article_id -> similarity
    created_at: datetime
    updated_at: datetime

    def add_duplicate(self, article: Article, similarity: float):
        self.duplicates.append(article)
        self.similarity_scores[article.id] = similarity
        self.updated_at = datetime.now()

# Dedup result actions
DEDUP_ACTIONS = {
    "original":     {"action": "keep",     "score_mult": 1.0},
    "canonical":    {"action": "redirect", "score_mult": 0.0},   # points to canonical
    "near_dup":     {"action": "dedup",    "score_mult": 0.0},   # 75-85% similar
    "exact_dup":    {"action": "discard",  "score_mult": 0.0},   # >95% similar
}
```

---

## D7. Freshness Decay

Article scores degrade over time to prioritize recent news. The decay applies to the **Freshness dimension** of the content score (20 points max).

### Decay Schedule

| Age | Freshness % | Content Score Impact |
|-----|------------|---------------------|
| < 1 hour | 100% | 20/20 points |
| < 6 hours | 95% | 19/20 points |
| < 24 hours | 85% | 17/20 points |
| < 3 days | 70% | 14/20 points |
| < 1 week | 50% | 10/20 points |
| < 2 weeks | 30% | 6/20 points |
| < 1 month | 15% | 3/20 points |
| < 3 months | 5% | 1/20 points |
| > 3 months | 1% | 0/20 points |

### Decay Formula

```python
FRESHNESS_DECAY_TABLE = [
    (1,     1.00),   # < 1 hour
    (6,     0.95),   # < 6 hours
    (24,    0.85),   # < 24 hours
    (72,    0.70),   # < 3 days
    (168,   0.50),   # < 1 week
    (336,   0.30),   # < 2 weeks
    (720,   0.15),   # < 1 month (30 days)
    (2160,  0.05),   # < 3 months (90 days)
    (8760,  0.01),   # < 1 year
]

def freshness_decay_multiplier(hours_old: float) -> float:
    for threshold, multiplier in FRESHNESS_DECAY_TABLE:
        if hours_old < threshold:
            return multiplier
    return 0.0  # > 1 year

def apply_freshness_decay(article: Article) -> int:
    hours_old = (datetime.now() - article.published_at).total_seconds() / 3600
    multiplier = freshness_decay_multiplier(hours_old)

    # Apply to the freshness dimension (max 20)
    raw_freshness = freshness_score(article.published_at)  # 0-20
    decayed = round(raw_freshness * multiplier)
    return decayed
```

### Re-scoring Schedule

Freshness decay requires periodic re-scoring. The following triggers update article scores:

| Trigger | Frequency | Action |
|---------|-----------|--------|
| **Hourly cron** | Every hour | Re-score all articles from last 7 days |
| **Daily cron** | Every 24h | Re-score all articles from last 30 days |
| **Weekly cron** | Every 7 days | Re-score all articles from last 90 days |
| **Tier migration check** | As part of daily cron | Move articles that dropped below tier thresholds |

```python
# Scheduled re-scoring pseudocode
async def hourly_rescore():
    articles = await db.articles.where(
        published_at > now() - timedelta(days=7),
        tier <= 3
    )
    for article in articles:
        new_score = recalculate_score(article)
        if new_score != article.current_score:
            await update_score(article, new_score)
            await check_tier_migration(article)

async def daily_rescore():
    articles = await db.articles.where(
        published_at > now() - timedelta(days=30),
        tier <= 4
    )
    for article in articles:
        new_score = recalculate_score(article)
        await update_score(article, new_score)
```

### Special Handling: Evergreen Content

Some content types should have **reduced decay** or **no decay**:

| Content Type | Decay Modifier | Rationale |
|-------------|---------------|-----------|
| Official documentation / changelogs | 50% decay rate | Remains relevant for reference |
| Research papers / whitepapers | No decay | Timeless academic content |
| Tutorials / how-to guides | 25% decay rate | Useful until technology changes |
| Legal / regulatory updates | 50% decay rate | Important for compliance |
| Breaking news follow-ups | Standard decay | Time-sensitive |
| Opinion / analysis | Standard decay | Context-bound |

```python
EVERGREEN_MULTIPLIERS = {
    "documentation": 0.50,
    "changelog": 0.50,
    "research_paper": 0.0,
    "whitepaper": 0.0,
    "tutorial": 0.25,
    "legal_update": 0.50,
    "news": 1.0,
    "analysis": 1.0,
    "opinion": 1.0,
}

def adjusted_decay_multiplier(article: Article) -> float:
    base = freshness_decay_multiplier(article.hours_old)
    modifier = EVERGREEN_MULTIPLIERS.get(article.content_type, 1.0)
    return base * (1 - modifier) + base * modifier  # simplified: base * (1 - mod * (1 - base))
```

---

## D8. Quality Override Rules

Automatic scores are overridden in specific, well-defined circumstances. Overrides are logged with reason, old score, new score, and timestamp.

### D8.1 Whitelist Override

**Purpose**: Known-good sources that should always score highly regardless of automated signals.

| Trigger | Action | Score Adjustment |
|---------|--------|-----------------|
| Source is on `WHITELIST` | Floor score at 85 (Tier 2 minimum) | `max(final_score, 85)` |
| Source is on `PREMIUM_WHITELIST` | Floor score at 92 (Tier 1 minimum) | `max(final_score, 92)` |

**Whitelist management**: The whitelist domains, score floors, and review
metadata live in [`../registries/whitelist.yaml`](../registries/whitelist.yaml).
That file is operator-customizable and the canonical source; the in-code data
structure is a thin loader.

```python
# Load operator-curated whitelist from registries/whitelist.yaml
import yaml
from pathlib import Path

def load_whitelist() -> dict:
    path = Path(__file__).parent.parent / "registries" / "whitelist.yaml"
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f)

WHITELIST = load_whitelist()
# WHITELIST["premium"]  -> Tier 1 floor (final_score >= 92)
# WHITELIST["standard"] -> Tier 2 floor (final_score >= 85)
```

**Annual review**: All whitelist entries must be reviewed annually. If a source's automated score drops below the whitelist floor for 30 consecutive days, it is flagged for review.

### D8.2 Blacklist Override

**Purpose**: Known-bad sources that should always score poorly.

| Trigger | Action | Score Adjustment |
|---------|--------|-----------------|
| Source is on `BLACKLIST` | Cap score at 20 (Tier 5) | `min(final_score, 20)` |
| Source is on `SPAM_BLACKLIST` | Force score to 0 | `final_score = 0` |

**Blacklist management**: The blacklist domains, category blocks, auto-block
rules, and fact-checker integrations live in
[`../registries/blacklist.yaml`](../registries/blacklist.yaml). That file is
operator-customizable and the canonical source; the in-code data structure is a
thin loader.

```python
# Load operator-curated blacklist from registries/blacklist.yaml
import yaml
from pathlib import Path

def load_blacklist() -> dict:
    path = Path(__file__).parent.parent / "registries" / "blacklist.yaml"
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f)

BLACKLIST = load_blacklist()
# BLACKLIST["blacklist"]            -> cap final_score at 20
# BLACKLIST["spam_blacklist"]       -> force final_score to 0
# BLACKLIST["category_blocks"]      -> permanent block patterns (X HTML, FB, IG, etc.)
# BLACKLIST["auto_block_rules"]     -> automatic per-incident response policy
```

**Auto-blacklist rules** (triggered automatically):
- Source generates >70% Tier 5 articles over 30 days
- Source is flagged by 2+ independent fact-checkers
- Source pattern-matches known content farm signatures (auto-generated text, zero original content)
- Source uses known clickbait headline patterns on >80% of articles

### D8.3 Manual Review Override

**Purpose**: Human reviewers can override any score.

| Action | Effect | Requirements |
|--------|--------|-------------|
| **Boost** | +10 to +30 points | Reviewer must provide justification; max 92 |
| **Reduce** | -10 to -50 points | Reviewer must provide justification; min 0 |
| **Lock** | Prevent future automatic re-scoring | Used for edge cases; reviewed quarterly |
| **Force Tier** | Directly assign tier | Overrides all automatic scoring |

```python
@dataclass
class ManualOverride:
    article_id: str
    reviewer_id: str
    old_score: int
    new_score: int
    action: str          # "boost", "reduce", "lock", "force_tier"
    justification: str   # Required free text
    timestamp: datetime
    expires_at: datetime  # Lock overrides expire after 90 days
    review_status: str   # "active", "expired", "reverted"
```

**Governance**:
- All overrides require justification (minimum 20 characters)
- Override history is visible in the article's audit log
- Boosts >20 points require a second reviewer to approve
- Locked scores are reviewed quarterly by admin

### D8.4 Breaking News Boost

**Purpose**: Time-sensitive verified news should get priority treatment.

| Condition | Boost | Requirements |
|-----------|-------|-------------|
| Verified breaking news | +20 points (capped at 100) | At least 2 Tier 1 sources confirm; published <3 hours ago |
| Probable breaking news | +10 points (capped at 95) | 1 Tier 1 source or 2 Tier 2 sources; published <6 hours ago |
| Trending topic | +5 points (capped at 90) | Engagement signals spike (>3x normal); published <12 hours ago |

**Breaking news detection**:
```python
def breaking_news_boost(article: Article) -> int:
    hours_old = article.hours_since_published

    # Check for breaking news signals
    confirmation_sources = count_confirming_sources(article, min_source_tier=2)
    is_confirmed_breaking = confirmation_sources >= 2 and hours_old < 3
    is_probable_breaking = (confirmation_sources >= 1 and hours_old < 6) or \
                           (confirmation_sources >= 2 and hours_old < 6)
    is_trending = article.engagement_signals.normalized_score > 3.0 and hours_old < 12

    if is_confirmed_breaking:
        return min(20, 100 - article.current_score)  # cap at 100
    if is_probable_breaking:
        return min(10, 95 - article.current_score)   # cap at 95
    if is_trending:
        return min(5, 90 - article.current_score)    # cap at 90

    return 0
```

**Confirmation sources**: Two or more independent sources (not syndicated copies) reporting the same core fact within the time window.

### D8.5 Topic Urgency Boost

**Purpose**: User-defined urgent topics get priority.

| Condition | Boost | Requirements |
|-----------|-------|-------------|
| Article matches "urgent" topic tag | +10 points (capped at 95) | User has marked topic as "urgent" in config |
| Article matches "follow" topic tag | +5 points (capped at 90) | User has marked topic as "follow" in config |

```python
TOPIC_URGENCY_BOOSTS = {
    "urgent": {"boost": 10, "cap": 95},
    "follow": {"boost": 5, "cap": 90},
    "interest": {"boost": 0, "cap": 100},  # no boost
}
```

### D8.6 Source Penalty Override

**Purpose**: Temporarily penalize sources that show quality degradation.

| Condition | Penalty | Duration |
|-----------|---------|----------|
| Source quality drops below 50 for 7 consecutive days | -15 points | 14 days or until quality recovers |
| Source publishes a correction/retraction | -10 points for affected article | Permanent |
| Source is acquired by entity with commercial bias flag | -10 points | Until review complete |

```python
class SourcePenalty:
    source_id: str
    penalty_points: int
    reason: str
    applied_at: datetime
    expires_at: datetime
    is_active: bool

    def apply(self, article_score: int) -> int:
        if not self.is_active or datetime.now() > self.expires_at:
            return article_score
        return max(0, article_score - self.penalty_points)
```

### Override Application Order

When multiple overrides apply, they are processed in this order:

```python
OVERRIDE_PRIORITY = [
    "blacklist",           # 1. Apply blacklist caps first (can force reject)
    "source_penalty",      # 2. Apply temporary source penalties
    "whitelist",           # 3. Apply whitelist floors (guarantees minimum)
    "breaking_news",       # 4. Apply breaking news boost
    "topic_urgency",       # 5. Apply topic urgency boost
    "manual_review",       # 6. Manual review has highest precedence
]

def apply_overrides(final_score: int, article: Article, overrides: list) -> int:
    score = final_score
    applied = []

    for override_type in OVERRIDE_PRIORITY:
        if override_type == "blacklist" and article.source.is_blacklisted:
            old = score
            score = min(score, article.source.blacklist_cap)
            applied.append(("blacklist", old, score))

        elif override_type == "source_penalty":
            for penalty in article.source.active_penalties:
                old = score
                score = penalty.apply(score)
                applied.append(("source_penalty", old, score))

        elif override_type == "whitelist" and article.source.is_whitelisted:
            old = score
            score = max(score, article.source.whitelist_floor)
            applied.append(("whitelist", old, score))

        elif override_type == "breaking_news":
            boost = breaking_news_boost(article)
            if boost > 0:
                old = score
                score = min(100, score + boost)
                applied.append(("breaking_news", old, score))

        elif override_type == "topic_urgency":
            boost = topic_urgency_boost(article)
            if boost > 0:
                old = score
                score = min(100, score + boost)
                applied.append(("topic_urgency", old, score))

        elif override_type == "manual_review":
            if article.has_active_manual_override:
                old = score
                score = article.manual_override.new_score
                applied.append(("manual_review", old, score))

    return score, applied
```

---

## D9. Implementation Notes

### Database Schema (Core Tables)

```sql
-- Source quality scores
CREATE TABLE source_scores (
    source_id           TEXT PRIMARY KEY,
    source_type_score   INT CHECK (source_type_score BETWEEN 0 AND 15),
    authority_score     INT CHECK (authority_score BETWEEN 0 AND 15),
    freshness_score     INT CHECK (freshness_score BETWEEN 0 AND 15),
    attribution_score   INT CHECK (attribution_score BETWEEN 0 AND 10),
    citation_score      INT CHECK (citation_score BETWEEN 0 AND 10),
    update_freq_score   INT CHECK (update_freq_score BETWEEN 0 AND 10),
    spam_risk_score     INT CHECK (spam_risk_score BETWEEN 0 AND 10),
    commercial_bias_score INT CHECK (commercial_bias_score BETWEEN 0 AND 10),
    rumour_risk_score   INT CHECK (rumour_risk_score BETWEEN 0 AND 10),
    feed_available_score INT CHECK (feed_available_score BETWEEN 0 AND 5),
    total_source_score  INT CHECK (total_source_score BETWEEN 0 AND 100),
    tier                INT CHECK (tier BETWEEN 1 AND 5),
    last_evaluated_at   TIMESTAMP,
    next_evaluation_at  TIMESTAMP,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Content scores
CREATE TABLE content_scores (
    article_id          TEXT PRIMARY KEY,
    source_id           TEXT REFERENCES source_scores(source_id),
    source_quality_inherited INT CHECK (source_quality_inherited BETWEEN 0 AND 20),
    freshness_raw       INT CHECK (freshness_raw BETWEEN 0 AND 20),
    freshness_decayed   INT CHECK (freshness_decayed BETWEEN 0 AND 20),
    primary_source_score INT CHECK (primary_source_score BETWEEN 0 AND 15),
    factual_confidence  INT CHECK (factual_confidence BETWEEN 0 AND 15),
    relevance_score     INT CHECK (relevance_score BETWEEN 0 AND 15),
    engagement_score    INT CHECK (engagement_score BETWEEN 0 AND 10),
    duplication_score   INT CHECK (duplication_score BETWEEN 0 AND 5),
    total_content_score INT CHECK (total_content_score BETWEEN 0 AND 100),
    final_score         INT CHECK (final_score BETWEEN 0 AND 100),
    tier                INT CHECK (tier BETWEEN 1 AND 5),
    is_override         BOOLEAN DEFAULT FALSE,
    override_reason     TEXT,
    scored_at           TIMESTAMP,
    freshness_expires_at TIMESTAMP,
    FOREIGN KEY (article_id) REFERENCES articles(id)
);

-- Override audit log
CREATE TABLE score_override_log (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    article_id          TEXT,
    source_id           TEXT,
    override_type       TEXT NOT NULL,  -- whitelist, blacklist, manual, breaking_news, etc.
    old_score           INT,
    new_score           INT,
    applied_by          TEXT,           -- user_id or "system"
    justification       TEXT NOT NULL,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Duplicate groups
CREATE TABLE duplicate_groups (
    group_id            TEXT PRIMARY KEY,
    original_article_id TEXT NOT NULL,
    canonical_url       TEXT,
    similarity_method   TEXT,           -- url, canonical, title, minhash, semantic
    max_similarity      FLOAT,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE duplicate_group_members (
    group_id            TEXT REFERENCES duplicate_groups(group_id),
    article_id          TEXT REFERENCES articles(id),
    similarity_score    FLOAT,
    is_original         BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (group_id, article_id)
);
```

### Score Recalculation Summary

```python
SCORE_RECALCULATION_TRIGGERS = {
    "new_article":        {"frequency": "event",     "scope": "single article"},
    "source_reeval":      {"frequency": "quarterly", "scope": "all articles from source (30d)"},
    "topic_filter_change": {"frequency": "event",     "scope": "all unarchived articles"},
    "hourly_freshness":   {"frequency": "hourly",    "scope": "articles <7 days old"},
    "daily_freshness":    {"frequency": "daily",     "scope": "articles <30 days old"},
    "engagement_update":  {"frequency": "event",     "scope": "single article"},
    "dedup_found":        {"frequency": "event",     "scope": "duplicate group"},
    "manual_override":    {"frequency": "event",     "scope": "single article"},
    "whitelist_update":   {"frequency": "event",     "scope": "all articles from source"},
    "blacklist_update":   {"frequency": "event",     "scope": "all articles from source"},
    "breaking_news_check": {"frequency": "every 15min", "scope": "articles <3 hours old"},
}
```

### Performance Considerations

| Concern | Strategy |
|---------|----------|
| **Scoring latency** | Score at ingestion time; freshness decay applied asynchronously via cron |
| **Bulk re-scoring** | Use batch updates (1000 articles per transaction) during source re-evaluation |
| **MinHash storage** | Store 128-bit signatures as 16-byte blobs; LSH buckets as integer arrays |
| **Embedding storage** | Use 384-dimension `all-MiniLM-L6-v2` embeddings (compact, fast); store as 1536-byte blobs |
| **Cache hot scores** | Keep last 7 days of article scores in Redis; cold storage for older |
| **Deduplication** | Bloom filter for Stage 1/2 (O(1)); LSH for Stage 4 (sub-linear) |
| **Override lookup** | In-memory cache of whitelist/blacklist domains; LRU with 1h TTL |

---

## D10. Complete Example: End-to-End Scoring

### Input: A new article

```yaml
article:
  url: "https://techcrunch.com/2024/06/15/openai-announces-gpt-5/"
  title: "OpenAI announces GPT-5 with multimodal reasoning"
  published_at: "2024-06-15T09:00:00Z"
  body: "OpenAI today announced GPT-5, its next-generation AI model..."
  source:
    domain: "techcrunch.com"
    type: "tier2_news"
    about_page: true
    author: "Kyle Wiggers"
    update_frequency: "daily"
    rss_available: true
```

### Step 1: Source Score Calculation

| Dimension | Raw Score | Reason |
|-----------|-----------|--------|
| Source Type | 10 | Major tech news outlet |
| Authority | 10 | Tier-2 established outlet (~13 years) |
| Freshness | 11 | Published within 24h of last check |
| Attribution | 8 | Named authors, ownership discoverable (Yahoo/Apollo) |
| Citation Quality | 6 | Some links to official sources, not every claim |
| Update Frequency | 8 | Daily publishing |
| Spam/SEO Risk | 8 | Some native advertising but clearly labeled |
| Commercial Bias | 6 | Ad-supported, some sponsored content |
| Rumour Risk | 7 | Uses anonymous sources occasionally, generally factual |
| RSS/API | 4 | RSS feed with partial content |
| **Total Source Score** | **78** | |

### Step 2: Content Score Calculation

| Dimension | Raw Score | Reason |
|-----------|-----------|--------|
| Source Quality | 16 | Inherited from source score 78 (Tier 2 band) |
| Freshness | 20 | Published <1 hour ago |
| Primary Source | 12 | Original reporting with on-background quotes |
| Factual Confidence | 13 | Official announcement, multiple confirmations |
| Relevance | 13 | Strong match on "AI" topic, title keyword match |
| Engagement | 6 | Moderate early engagement (HN discussion forming) |
| Duplication | 5 | Original, no duplicates found |
| **Total Content Score** | **85** | |

### Step 3: Final Score

```
Final Score = round(78 * 0.4 + 85 * 0.6)
            = round(31.2 + 51.0)
            = 82
```

### Step 4: Tier Determination

- Score 82 falls in **Tier 2 (75-89): High Quality**
- **Action**: Auto-include in digest with standard treatment

### Step 5: Override Check

- Not whitelisted (TechCrunch is not on premium whitelist)
- Not blacklisted
- Not breaking news (single source, confirmed by official blog but not independently)
- Topic not marked urgent
- No manual override

**Final result**: **Score 82, Tier 2, included in digest**

---

## Appendix A: Quick Reference Tables

### Score Ranges by Tier

| Tier | Score | Source Min | Content Min | Action |
|------|-------|-----------|-------------|--------|
| 1 | 90-100 | ~85 | ~90 | Auto-include, top priority |
| 2 | 75-89 | ~65 | ~75 | Standard inclusion |
| 3 | 60-74 | ~50 | ~60 | Include, flag for review |
| 4 | 40-59 | ~35 | ~45 | Hold in review queue |
| 5 | 0-39 | any | any | Reject and log |

### Decay Schedule at a Glance

| Age | Freshness % | Raw 20-pt | Decayed |
|-----|-------------|-----------|---------|
| 30 min | 100% | 20 | 20 |
| 3 hours | 95% | 19 | 18 |
| 12 hours | 85% | 17 | 14 |
| 2 days | 70% | 14 | 10 |
| 5 days | 50% | 10 | 5 |
| 10 days | 30% | 6 | 2 |
| 20 days | 15% | 3 | 0 |
| 60 days | 5% | 1 | 0 |
| 180 days | 1% | 0 | 0 |

### Source Type Score Reference

| Type | Score | Example |
|------|-------|---------|
| Official blog | 15 | blog.google |
| Tier-1 news | 12 | Reuters, AP, BBC |
| Major news | 10 | TechCrunch, Bloomberg |
| Specialist pub | 8 | IEEE Spectrum |
| Quality indie | 6 | Benedict Evans |
| Curated aggregator | 5 | TLDR Newsletter |
| Community | 3 | Reddit, Dev.to |
| Syndication | 2 | Medium general |
| Unknown/SEO | 0 | Content farm |

---

*End of Section D: Source Quality Scoring Rubric*
