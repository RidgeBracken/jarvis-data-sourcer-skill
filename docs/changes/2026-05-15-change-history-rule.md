# Change History Rule

## What Changed

Added a repository rule requiring every future skill update or maintenance
change to include a dated Markdown change note under `docs/changes/`.

The rule was added to `AGENTS.md` so future Codex or agent sessions see it as a
working instruction, and summarized in `README.md` so people browsing the repo
can understand the process.

## Reason For The Change

The skill is expected to evolve over time. A required Markdown change note keeps
the reason for each update close to the files, so future maintainers can
understand what changed, why it changed, how it was validated, and how to
backtrack if needed.

## Files Changed

- `AGENTS.md`
- `README.md`
- `docs/changes/2026-05-15-change-history-rule.md`

## Validation Performed

- Verified the rule did not exist before the change.
- Verified this dated change note exists.
- Verified `README.md` documents the rule.

## Rollback Notes

To remove this policy, delete `AGENTS.md` or remove its change-history section,
remove the `README.md` maintenance rule, and delete this change note. That is not
recommended because it would remove the explicit backtracking record for future
skill updates.
