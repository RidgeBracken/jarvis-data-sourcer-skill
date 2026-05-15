# Jarvis Data Sourcer Skill

Installable Codex skill for public-source discovery, lawful web/RSS/API scraping
design, source quality scoring, deduplication, normalized feed schemas, and
agent handoff for data collection systems.

## Install

From Codex, install the skill folder from this repository:

```bash
python ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py --repo RidgeBracken/jarvis-data-sourcer-skill --path jarvis-data-sourcer
```

Restart Codex after installation so the skill is discovered.

The repository also includes `jarvis-data-sourcer-v4.1-fixed.skill`, a packaged
tar.gz skill archive for agents or tools that import `.skill` files directly.

## Validation

Validated on 2026-05-15:

- Extracted from `jarvis-data-sourcer-v4-polished.skill`.
- Ran the Codex `skill-creator` quick validator: passed.
- Parsed every bundled YAML file: passed after fixing malformed registry comment
  lines in 14 topic registries.
- Checked internal skill references and local Markdown links.
- Smoke-tested the OpenAI/Codex registry against public endpoints:
  OpenAI News RSS returned HTTP 200 and valid RSS; GitHub Codex releases Atom
  returned HTTP 200 and valid Atom; OpenAI developer docs returned HTTP 403 to a
  simple HTTP client and should be manually/browser-verified rather than bypassed.

## Notes

The installable skill is the `jarvis-data-sourcer/` directory. Repo-level notes
and patch history live outside that folder so installed agents only load the
skill resources they need.
