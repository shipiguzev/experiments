---
name: doc-consistency-check
description: Verify README's "Структура репозитория" tree and numbered doc list match the actual repo files. Use before/after adding a chart, values file, or docs page, or when asked to check the repo for drift between docs and reality.
---

Catches drift like: a new values file or cluster gets added to `charts/`, but README's file-tree
block and numbered doc list never get updated to mention it.

## 1. Get the actual file list

```bash
find . -maxdepth 4 -type f -not -path '*/.git/*' | sort
```

## 2. Compare against README's tree block

README.md has a fenced code block under `## Структура репозитория` listing every top-level dir and
the notable files/values inside each (with inline `#` comments explaining each one). For every file
under `charts/`, `docs/`, `monitoring/`, `postgres/operator/`, `clickhouse/operator/`, `cnpg/operator/`
that isn't just a `templates/*.yaml` (those are summarized, not enumerated individually):

- Flag any file that exists on disk but isn't mentioned in the tree block (e.g. a new
  `values-testN.yaml`, a new dashboard JSON, a new doc).
- Flag any file mentioned in the tree block that no longer exists on disk (renamed/removed but
  README not updated).

## 3. Compare docs/ against the numbered list

README has a numbered list ("Порядок развёртывания", items 1–13+) linking every doc under `docs/`
in the order they're meant to be followed, plus the standalone quick-start link. For every
`docs/*.md` file:

- Flag any doc file not linked from either the quick-start reference or the numbered list.
- Flag any numbered list entry whose linked file doesn't exist.

## 4. Report

List findings as concrete file paths, not general statements — e.g. "`charts/clickhouse-cluster/
values-test3.yaml` exists (added in 824cb81) but isn't in the README tree block or numbered docs
list" rather than "some files are undocumented". Don't auto-fix — README wording/placement in the
sequence is an editorial call (per CLAUDE.md, this belongs in the same commit as the file that
caused the drift, but that commit already happened) — report and let the user decide where it goes.
