# CLAUDE.md -- Second Brain

## Project Purpose

Documentation repo for the Second Brain system -- a personal knowledge capture and retrieval pipeline. Contains the setup guide and companion prompts. No application code lives here; the actual Supabase edge functions are deployed separately (see Related Systems below).

## Directory Structure

```text
second-brain/
  docs/
    second-brain-guide.md     # Step-by-step setup guide for capture and retrieval
    second-brain-prompts.md   # Companion prompts for migration, discovery, reviews
  README.md                   # Project overview and architecture diagram
  CLAUDE.md                   # This file
  LICENSE                     # MIT
```

## CI

- **Workflow:** `.github/workflows/ci.yml` runs `markdownlint-cli` on Node.js 20
- **Lint config:** `.markdownlint.json` -- relaxed line length (200), allows inline HTML, allows duplicate sibling headings
- **Required status check:** `lint-check` (summary job that gates PR merges)
- **Lint errors are advisory only** (`|| true` in CI step). For a docs-only repo, strict lint enforcement is not worth blocking PRs. `lint-check` always passes regardless of lint results.
- **GitHub Actions are SHA-pinned** per root security standards.

## Workflow

- PRs target `main`; `lint-check` is the required status check
- `.github/workflows/auto-merge.yml` enables auto-merge with squash on every PR to `main`
- Branches are automatically deleted after merge

## Secrets

- `PAT_TOKEN` -- personal access token used by the auto-merge workflow (`gh pr merge --auto`). Required because the default `GITHUB_TOKEN` cannot trigger downstream workflows or satisfy branch protection rules that require a PAT.

## Content Rules

- No company-specific information in any document
- No emojis in content or documentation
- Date format: YYYY-MM-DD
- Markdown-first; code blocks for configs and commands

## Related Systems

- **Edge functions:** `~/supabase/functions/` -- `ingest-thought` (Slack capture) and `second-brain-mcp` (MCP retrieval)
- **MCP server:** Configured globally in `~/.claude.json` under `mcp__second-brain__*` prefix
- **Slack app:** Configured in user's Slack workspace (webhook points to `ingest-thought` function)
