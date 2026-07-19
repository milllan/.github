# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.4.0] - 2026-07-19

### Changed
- **Default `openai_endpoint` switched to Z.ai's Coding Plan endpoint** (`api.z.ai/api/coding/paas/v4/chat/completions`). Z.ai has two endpoints: the Coding Plan (subscription, what most users have) and the pay-per-token API (`/api/paas/v4/`, requires a positive credit balance which defaults to $0). The Coding Plan endpoint works with subscription keys; the old default returned `429 insufficient balance` for users without API credits.
- **Smart retry for 429s.** The retry loop now inspects the error body: permanent 429s (`insufficient balance`, `quota exceeded`, `invalid key`) fail fast instead of burning ~100s on backoff. Only transient 429s (rate-limit) and 502/503/504 are retried.

## [1.3.0] - 2026-07-18

### Added
- **Multi-provider support.** New `provider` input (`gemini` | `openai`). The `openai` branch uses the OpenAI-compatible chat-completions format and works with any compatible endpoint (Z.ai GLM, OpenRouter, DeepSeek, OpenAI) via the `openai_endpoint` input (defaults to Z.ai). Callers can now run two review jobs in parallel — e.g. Gemini + GLM — for two genuinely independent reviews per PR.
- `openai_endpoint` input and `OPENAI_API_KEY` secret for the OpenAI-compatible branch.
- Comment headings now name the provider (`## Gemini Code Review` / `## GLM Code Review`) so multiple reviews on one PR are distinguishable.

### Changed
- `GEMINI_API_KEY` is now optional (`required: false`) since either provider key may be present depending on `provider`.

## [1.2.0] - 2026-07-18

### Changed
- **Raised default `max_diff_chars` from 60000 → 250000.** The original limit was conservative; `gemini-3.5-flash` has a 1M-token context window, and a legitimate feature PR (#82) was silently skipped by 507 bytes.
- **Skipped reviews now post a comment** explaining the skip reason (empty diff, or oversized diff with the byte count and limit) instead of posting nothing. Previously you couldn't tell a skipped review from a broken one without checking the Actions tab.

### Added
- **Manual re-review support.** Caller workflows can add a `workflow_dispatch` job that pushes an empty commit to a PR's branch (given a `pr_number` input), re-firing the `pull_request: synchronize` trigger. The reusable workflow itself stays `workflow_call`-only — GitHub Actions does not allow a `workflow_dispatch` event to invoke a reusable-workflow caller job directly, so the trigger lives in each caller.

### Fixed
- Diff-step env now reads `BASE_REF` from the resolved context step instead of `github.base_ref` (which is empty on manual triggers).

## [1.1.0] - 2026-07-17

### Changed
- **Replaced the review engine**: swapped the Google Antigravity CLI (`agy`) for a direct call to the **Gemini REST API**. The `agy` CLI authenticates via browser OAuth only and fails fast on headless hosts, so it cannot run unattended in GitHub Actions. The workflow now calls `generativelanguage.googleapis.com` with a `GEMINI_API_KEY` secret.
- **Renamed** `agy-reviewer.yml` → `gemini-reviewer.yml`. The secret changed from `AGY_API_KEY` → `GEMINI_API_KEY`.
- **Hardened prompt assembly**: the request JSON is now built with `jq --rawfile` so diff contents (quotes, backticks, `$`) cannot break parsing or inject into the prompt.
- Removed the `pull_request` trigger from the reusable workflow — it was dead (it only fires on PRs to this `.github` repo, which has no code to review). The file is now `workflow_call`-only.

### Added
- Configurable `model` input (default `gemini-3.5-flash`).
- `max_diff_chars` cost guard that skips review on oversized diffs (default 60000; `0` disables).
- `git fetch` of the base ref so the diff is accurate even on shallow checkouts.
- Graceful error/empty/429 handling that still posts an informative comment.

### Fixed
- `README.md` no longer claims GitHub auto-inherits the workflow across all repos — it now documents the reusable-workflow caller pattern honestly.
- Removed the leftover `<YOUR_GITHUB_ORG_OR_USER>` placeholder note.

## [1.0.1] - 2026-07-08

### Changed
- Customized GitHub Actions reference URLs in `README.md` to target `milllan` instead of placeholders.

## [1.0.0] - 2026-07-08

### Added
- Created the directory structure for global GitHub Actions workflows.
- Implemented `agy-reviewer.yml` reusable GitHub Actions workflow template.
- Added comprehensive `README.md` documentation explaining how to consume the reusable workflow.
- Created basic `.gitignore` rules.
