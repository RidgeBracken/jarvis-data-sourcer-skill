# Repository Instructions

## Skill Change History Rule

Every change to this skill must include a dated Markdown change note under
`docs/changes/`.

This applies to changes in:

- `jarvis-data-sourcer/`
- packaged `.skill` archives
- validation, install, or repo metadata that changes how the skill is shared or
  maintained

Use this filename pattern:

```text
docs/changes/YYYY-MM-DD-short-change-name.md
```

Each note must include:

- What changed
- Why it changed
- Files changed
- Validation performed
- Rollback/backtracking notes, if relevant

Do not bury meaningful skill history only in git commit messages. The Markdown
change note is required so future agents and humans can recover the reasoning
without reconstructing it from chat history.
