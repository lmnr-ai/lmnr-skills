# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project Overview

Source of the unified `laminar` agent skill, distributed via `lmnr-cli` (`setup` / `skill add` fetch `github:lmnr-ai/lmnr-skills/skills/laminar#main` with giget and install it into `.claude/` / `.cursor/` / `.codex/` / `.agents/`). One skill, one `SKILL.md` router, one-level-deep `references/*.md` files.

## Conventions

- Every new reference file must be added to three places: the task-router table in `skills/laminar/SKILL.md`, the `description:` frontmatter of `SKILL.md` (it drives skill-selection in agents), and the Contents list in `README.md`.
- Reference files are self-contained per task; cross-link related references with relative links instead of duplicating content.
- Verify every CLI flag and SDK signature against the actual source before documenting: `lmnr-ts/packages/lmnr-cli/src/index.ts` (command surface), `lmnr-ts/packages/{lmnr,client}/src` and `lmnr-python/src/lmnr` (SDKs), `lmnr/app-server/src/query_engine/schema.rs` (SQL-queryable tables/columns).

## Gotchas verified against source

- `lmnr-cli dataset create` REQUIRES `-o/--output-file` (`.requiredOption` in the CLI); examples without it fail.
- The dataset file-push path (CLI and `/v1/datasets/datapoints` API) requires an explicit `data` field per row and drops unknown top-level keys — only the UI file upload folds unrecognized keys into `data`.
- `dataset_datapoints` is SQL-queryable (columns: `id`, `created_at`, `dataset_id`, `data`, `target`, `metadata`); the `datasets` table is NOT — get dataset ids from `lmnr-cli dataset list --json`.
