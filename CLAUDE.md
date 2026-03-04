# CLAUDE.md -- Second Brain

## Project Purpose

Documentation repo for the Second Brain system -- a personal knowledge capture and retrieval pipeline. Contains the setup guide and companion prompts. No application code lives here; the actual Supabase edge functions are deployed separately (see Related Systems below).

## Directory Structure

```text
second-brain/
  .github/
    workflows/
      ci.yml                  # Markdown lint (advisory) + lint-check summary job
      auto-merge.yml          # Squash-merge PRs to main, delete branches
    dependabot.yml            # Weekly GitHub Actions version updates
  docs/
    second-brain-guide.md     # Step-by-step setup guide for capture and retrieval
    second-brain-prompts.md   # Companion prompts for migration, discovery, reviews
  .gitignore                  # Security-focused patterns (secrets, credentials, OS artifacts)
  .markdownlint.json          # Lint config (relaxed line length, allow inline HTML)
  CLAUDE.md                   # This file
  CONTRIBUTING.md             # Contribution guidelines (minimal, personal project)
  LICENSE                     # MIT
  README.md                   # Project overview, privacy notice, FAQ
  SECURITY.md                 # Vulnerability reporting process
```

## Commands

```bash
# Lint all markdown files locally
npx markdownlint '**/*.md' --ignore node_modules
```

## CI

- **Workflow:** `.github/workflows/ci.yml` runs `markdownlint-cli` on Node.js 20
- **Lint config:** `.markdownlint.json` -- relaxed line length (200), allows inline HTML, allows duplicate sibling headings
- **Required status check:** `lint-check` (summary job that gates PR merges)
- **Lint errors are advisory only** (`|| true` in CI step). For a docs-only repo, strict lint enforcement is not worth blocking PRs. `lint-check` always passes regardless of lint results.
- **GitHub Actions are SHA-pinned** per root security standards.

## Workflow

- **No direct pushes to `main`** -- all changes go through PRs
- **Ruleset:** `main-protection` -- requires `lint-check` status check to pass before merge
- `.github/workflows/auto-merge.yml` enables auto-merge with squash on every PR to `main`
- Branches are automatically deleted after merge
- **Branch prefixes:** `feature/`, `fix/`, `docs/`, `test/`, `chore/`

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
