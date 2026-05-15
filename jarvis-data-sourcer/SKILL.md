---
name: jarvis-data-sourcer
description: >
  Use for public-source discovery, lawful web/RSS/API scraping design, source
  quality scoring, deduplication, normalized feed schemas, and agent handoff for
  data collection systems. Applies to Jarvis HQ and to model-agnostic agents that
  need to gather, rank, and package public web data without bypassing access
  controls or collecting unnecessary personal data.
---

# Jarvis Data Sourcer

## Mission

Build or operate a public-data sourcing layer that finds reliable sources,
collects allowed public updates, scores quality, deduplicates repeated stories,
normalizes records, and prepares clean outputs for agents, dashboards, digests,
or downstream automation.

This skill is model-agnostic. Do not assume a specific agent runtime, browser
tool, scraping library, database, or deployment platform unless the user or the
target repository already provides one.

## Use This Skill For

- Discovering sources for a topic, company, product, beat, or event stream.
- Designing scraping/RSS/API pipelines for news, changelogs, releases, forums,
  public videos, documentation updates, or public datasets.
- Scoring sources and extracted items for authority, freshness, relevance,
  attribution, and extraction confidence.
- Defining normalized schemas for sources, feed items, scrape jobs, digests, and
  audit logs.
- Building Jarvis HQ-style personal news or briefing systems.
- Reviewing scraping implementations for safety, compliance, and quality risks.

Do not use this skill to bypass paywalls, CAPTCHAs, login gates, anti-bot
systems, private APIs, or platform terms.

## Operating Modes

Choose the smallest mode that fits the request:

| Mode | Use When | Load |
|---|---|---|
| Source discovery | Need source lists or topic expansion | `references/source-discovery.md`, relevant `registries/*.yaml` |
| Quality review | Need to rank or reject sources/items | `references/quality-scoring.md` |
| Schema/API design | Need database/API/contracts | `references/data-schema.md` |
| Implementation | Need to build the service | `references/implementation.md`, then `architecture.md`, plus the relevant `pipeline-*.md` files (`pipeline.md` index; `pipeline-collection.md`, `pipeline-processing.md`, `pipeline-distribution.md`, `pipeline-orchestration.md` for the affected stages) |
| Security review | Need legal/safety/privacy checks | `references/security.md` |
| Testing | Need test plans or fixtures | `references/testing.md` |
| Phase 1 reference build | User wants to scaffold the Phase 1 Node.js/TypeScript reference implementation | `references/phase-1-build-prompt.md` |

Avoid loading every reference at once. The reference set is intentionally deep.

## Default Workflow

1. Restate the data objective, target topic, freshness window, output format,
   and known constraints.
2. Prefer official APIs, RSS/Atom/JSON feeds, sitemaps, public changelogs,
   GitHub releases, and public documentation before raw HTML scraping.
3. Discover candidate sources using the 10-step workflow in
   `references/source-discovery.md`.
4. Reject or flag sources that require authentication, bypassing, heavy
   anti-bot workarounds, unclear ownership, missing dates, copied content,
   excessive ads, or weak attribution.
5. Score remaining sources and items using `references/quality-scoring.md`.
6. Normalize every kept item to a schema from `references/data-schema.md`.
7. Deduplicate by canonical URL, normalized title, source cluster, and
   high-similarity content fingerprint.
8. Preserve provenance: source URL, canonical URL, source name, author, publish
   date, discovered date, extraction method, license/terms notes, and confidence.
9. Produce a short audit trail: what was searched, what was accepted, what was
   rejected, why, and what should be manually verified.

## Hard Safety Rules

- Respect `robots.txt`, crawl-delay, rate limits, copyright notices, and site
  terms. When sources conflict, use the most restrictive limit.
- Never bypass paywalls, CAPTCHAs, login requirements, anti-bot protections, or
  API access controls.
- Do not collect private or unnecessary personal data. If public content
  contains incidental PII, keep only what is required for attribution and apply
  minimization.
- Do not scrape private social media, DMs, closed groups, paid communities, or
  authenticated pages unless the user owns the data and explicitly authorizes
  export through an official mechanism.
- Use conservative defaults: one request per domain per second at most, lower
  for small sites, and back off on 403, 429, 5xx, or robots changes.
- If you have browser-control or computer-use tools, the same rules apply.
  Do not log in to a site to extract content the unauthenticated visitor
  cannot see, do not solve CAPTCHAs (manually or programmatically), and do
  not simulate human input patterns to evade anti-bot detection. Computer
  use does not grant new scraping permissions.
- Store credentials only in the target project secret manager or environment
  variables. Never hardcode tokens, cookies, session IDs, or API keys.
- Attribute sources in every digest or downstream record.
- Mark unverified source registries as `needs_verification`; do not present them
  as confirmed without checking current pages.

## Source Priority

1. Official blog, newsroom, changelog, release notes, or documentation.
2. RSS/Atom/JSON feed or official API.
3. GitHub releases/tags for software projects.
4. Public YouTube/podcast feeds for official media.
5. Reputable news outlets and specialist publications.
6. Public forums or social channels only when allowed and clearly useful.
7. Search results only for discovery, not as a primary monitored source.

## Quality Tiers

| Tier | Score | Action |
|---|---:|---|
| 1 | 90-100 | Auto-include with attribution |
| 2 | 75-89 | Include normally |
| 3 | 60-74 | Include only with review/low confidence |
| 4 | 40-59 | Hold, enrich, or monitor only |
| 5 | 0-39 | Reject and log reason |

## Agent Command Interface

Agents may expose these as natural-language commands, slash commands, tool
routes, or task templates:

- `discover sources for <topic>`
- `validate source <url> for <topic>`
- `register source <url> as <type> for <topic>`
- `score sources for <topic>`
- `scrape latest public updates for <topic> since <time window>`
- `normalize items to <schema>`
- `deduplicate feed items`
- `create digest for <topic> as <markdown|json|html|telegram>`
- `review scraping plan for safety`
- `build phase <n> of the Jarvis data pipeline`

If an agent lacks browser or HTTP tools, produce a precise plan, schemas, and
queries rather than inventing results.

## Output Contract

For discovery or scraping tasks, return:

- Accepted sources with URL, type, topic, status, quality score, and rationale.
- Rejected or deferred sources with reason.
- Required manual checks.
- Normalized item records or schema mapping when items were collected.
- Compliance notes: robots/terms/rate-limit assumptions and unresolved risks.
- Next action: implement, verify, schedule, or ask for missing credentials.

## Bundled Resources

- `references/source-discovery.md`: 10-step workflow and detailed topic examples.
- `references/quality-scoring.md`: source and content scoring rubric.
- `references/data-schema.md`: JSON schemas and SQL table definitions.
- `references/architecture.md`: Jarvis HQ component and Docker architecture.
- `references/pipeline.md`: 10-stage collection-to-distribution pipeline — INDEX file. Contains the overview, data model, flow diagram, and appendices. Section-2 stage designs were split out for progressive disclosure:
  - `references/pipeline-collection.md`: Stages 1–2 (Collection, Validation)
  - `references/pipeline-processing.md`: Stages 3–6 (Deduplication, Enrichment, Scoring, Summarisation)
  - `references/pipeline-distribution.md`: Stages 7–10 (Categorisation, Ranking, Storage, Distribution)
  - `references/pipeline-orchestration.md`: Pipeline orchestration + output formats.
- `references/implementation.md`: phased MVP build plan.
- `references/security.md`: risk register and compliance checklist.
- `references/testing.md`: unit, integration, end-to-end, compliance tests.
- `references/phase-1-build-prompt.md`: model-agnostic Phase 1 build prompt
  for a Node.js/TypeScript reference stack. Originally authored for OpenAI
  Codex; reusable by any code-generating agent.
- `registries/*.yaml`: starter source registries (per topic) extracted from
  the topic examples in `source-discovery.md`. Treat all entries as starter
  candidates until verified. Each registry includes a `last_verified` date
  and an `excluded:` list for sources investigated and rejected.
- `registries/whitelist.yaml`: operator-curated quality whitelist used by
  `quality-scoring.md` D8.1 and `security.md` §8.2. Premium domains get a
  Tier-1 score floor (>=92), standard domains get Tier-2 (>=85). Review
  annually.
- `registries/blacklist.yaml`: operator-curated blocklist + auto-block rules
  used by `quality-scoring.md` D8.2 and `security.md` §8.3–§8.4. Defines
  category blocks (X HTML, Facebook, TikTok, etc.), auto-populated
  fact-checker integration, and per-incident response policy.
