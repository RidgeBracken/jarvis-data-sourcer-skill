# Patches Applied — v4 Codex-Reviewed-Patched-Polished

**Date:** 2026-05-14
**Base archive:** `jarvis-data-sourcer-v2-codex-reviewed.skill`
**v3 archive (must-fix only):** `jarvis-data-sourcer-v3-patched.skill`
**v4 archive (must-fix + polish):** `jarvis-data-sourcer-v4-polished.skill`

**Post-validation repo fix (2026-05-15):** restored the missing comment marker
on the `needs_verification` explanation line in 14 topic registry YAML files so
all bundled registries parse correctly.

This document records the 30 patches (18 §A must-fix + 12 §B polish) applied
to the v2 archive. All edits preserve the original structure and intent of
the skill. The skill remains backward-compatible — only safety holes, internal
inconsistencies, portability bugs, and structural rough edges were addressed.

---

## §A — Must-fix safety, consistency, and portability (S1–S18)

Applied in v3 and carried forward to v4.

| # | Title | Files touched |
|---|---|---|
| S1 | Reconcile User-Agent across the skill | `pipeline.md`, `references/phase-1-build-prompt.md`, `references/testing.md` |
| S2 | Strip residential-proxy example | `references/data-schema.md` |
| S3 | Replace DDG HTML scraping with API/Brave | `references/architecture.md` |
| S4 | Close R027 residential-proxy loophole | `references/security.md` |
| S5 | Remove `cookie` from `auth_type` enum | `references/data-schema.md` |
| S6 | Remove `twitter` / `twitter_api` from source enums | `references/data-schema.md` |
| S7 | Reconcile Reddit/Twitter blocklist with registries | `references/security.md` |
| S8 | Replace PII auto-block with redact-and-continue | `references/security.md` |
| S9 | Fix correction-history reverse-logic in rumour-risk rubric | `references/quality-scoring.md` |
| S10 | Add computer-use clause to Hard Safety Rules | `SKILL.md` |
| S11 | Mark architecture.md Section 4 as reference implementation (not verdict) | `references/architecture.md` |
| S12 | Add table-of-contents to large reference files (round 1) | `references/phase-1-build-prompt.md`, `references/skill-overview.md` |
| S13 | Add platform-access guide to all 14 registries | `registries/*.yaml` (×14) |
| S14 | Fix stale OpenAI URLs in build prompt | `references/phase-1-build-prompt.md` |
| S15 | Rename `codex-handoff.md` → `phase-1-build-prompt.md` + neutralize Codex-specific language | `references/`, `SKILL.md` |
| S16 | Reconcile rate-limit defaults to match SKILL.md (1 req/s/domain) | `references/phase-1-build-prompt.md` |
| S17 | Add robots.txt fetcher as a mandatory Phase-1 component | `references/phase-1-build-prompt.md` |
| S18 | Add `license_notes` field to FeedItem schema | `references/data-schema.md` |

## §B — Polish, portability, and structure (S19–S30)

New in v4.

| # | Title | Files touched |
|---|---|---|
| S19 | Split `pipeline.md` into 5 files (index + 4 sub-files) for progressive disclosure | `references/pipeline.md`, new: `pipeline-collection.md`, `pipeline-processing.md`, `pipeline-distribution.md`, `pipeline-orchestration.md`; updated `SKILL.md` Operating Modes + Bundled Resources |
| S20 | Add table-of-contents to all remaining reference files >300 lines | `references/quality-scoring.md`, `references/data-schema.md`, `references/security.md`, `references/source-discovery.md`, `references/implementation.md` |
| S21 | Designate `data-schema.md` as the canonical schema; add cross-reference notes from `architecture.md` §4.6 and `phase-1-build-prompt.md` §32 | `references/architecture.md`, `references/phase-1-build-prompt.md` |
| S22 | Reframe `skill-overview.md` "DOES NOT" bullets so out-of-context quotes can't be misread (each bullet now starts with the verb "Attempt to…", with structured detail) | `references/skill-overview.md` |
| S23 | Add `last_verified: "<ISO date>"` field to every registry | `registries/*.yaml` (×14) |
| S24 | Add `excluded: []` template to every registry for known dead-ends | `registries/*.yaml` (×14) |
| S25 | Externalize whitelist/blacklist to `registries/whitelist.yaml` and `registries/blacklist.yaml`; update references in `quality-scoring.md` D8.1/D8.2 and `security.md` §8.2/§8.3 to load from those files | `registries/whitelist.yaml` (new), `registries/blacklist.yaml` (new), `references/quality-scoring.md`, `references/security.md`, `SKILL.md` |
| S26 | Add international authority cues to `quality-scoring.md` D1.2 (.gc.ca, .gov.au, .eu, .govt.nz, NHK, ARD/ZDF, ABC.au, CBC, Le Monde, Der Spiegel, NHK, SCMP, Toyo Keizai, etc.) | `references/quality-scoring.md` |
| S27 | Downweight Twitter engagement signals (caps 3→1 and 2→1) and add Lobsters / Bluesky signals to `quality-scoring.md` D2.6 | `references/quality-scoring.md` |
| S28 | Add `verified` flag semantics block to every registry (clarifies what `verified: true` / `false` / absent means and ties it to `last_verified`) | `registries/*.yaml` (×14) |
| S29 | Name a default embedding model (`sentence-transformers/all-MiniLM-L6-v2`) for the relevance score and mark it implementation-dependent | `references/quality-scoring.md` |
| S30 | Already done as part of S1 (UA test alignment) | (no-op) |

## Notable behavioral changes (v3 + v4)

These are user-visible behavior changes since the v2 archive.

1. **Reddit RSS feeds are now explicitly allowed for titles/URLs only.** (S7)

2. **PII detection no longer permanently blocks sources.** A persistent-PII
   threshold (`>10` in 7 days) escalates to manual review instead. (S8)

3. **No residential proxies, anywhere.** (S2 + S3 + S4)

4. **`auth_type: cookie` removed from schema.** (S5)

5. **Twitter/X removed from source `type` and `extraction_method` enums.** (S6)

6. **Computer-use / browser-control rule** added to SKILL.md Hard Safety Rules. (S10)

7. **Phase-1 reference build now requires robots.txt enforcement.** (S17)

8. **`quality-scoring.md` correction-history bug fixed.** (S9)

9. **`pipeline.md` is now a slim INDEX** pointing at 4 stage-grouped sub-files.
   Agents implementing a single stage load ~30 KB instead of the original ~145 KB. (S19)

10. **Whitelist / blacklist are externalized**, operator-customizable, and
    no longer require forking the skill. (S25)

11. **Authority rubric now recognizes international outlets** beyond US/UK
    media. (S26)

12. **Embedding model named** for the relevance scorer
    (`sentence-transformers/all-MiniLM-L6-v2`), with a graceful-degrade path
    for agents without an embedding model available. (S29)

13. **Every registry now declares**: `last_verified` (when the entries were
    spot-checked), an `excluded: []` slot for known dead-ends, and a
    `verified` flag semantics block. (S23 + S24 + S28)

## Files added in v4

- `registries/whitelist.yaml` — externalized quality whitelist
- `registries/blacklist.yaml` — externalized blacklist + auto-block rules
- `references/pipeline-collection.md` — Stages 1–2 (Collection, Validation)
- `references/pipeline-processing.md` — Stages 3–6 (Dedup, Enrich, Score, Summarise)
- `references/pipeline-distribution.md` — Stages 7–10 (Categorise, Rank, Store, Distribute)
- `references/pipeline-orchestration.md` — Orchestration + Output Formats

## Files renamed in v3

- `references/codex-handoff.md` → `references/phase-1-build-prompt.md`

## Out of scope (still pending)

These items from the original deep review were intentionally left for a
future iteration:

- Per-language authority lists (e.g., separate `quality-scoring-cn.md`,
  `quality-scoring-jp.md`)
- A `compatibility:` frontmatter field in SKILL.md declaring minimum runtime
  capabilities
- Per-runtime agent contract files (`agents/claude.yaml`, `agents/codex.yaml`,
  `agents/generic.yaml`) — only `agents/openai.yaml` ships today
- A unit-test fixture for the externalized whitelist/blacklist YAML loaders
- A CI smoke check that every registry URL still 200s

## How this archive differs from v2

- v3 changes (18 §A must-fix patches) plus:
- 12 §B polish patches:
  - 6 new files (5 pipeline subdivisions + whitelist/blacklist YAML)
  - 1 file rename (codex-handoff → phase-1-build-prompt)
  - 5 reference files now have a top-of-file Table of Contents
  - 14 registries gained 3 new fields/blocks
  - International authority cues, downweighted Twitter signals, named
    embedding model in `quality-scoring.md`
- Skill structure (top-level layout, `agents/` contract) is otherwise unchanged
