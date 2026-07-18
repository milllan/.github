# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2026-07-18

### Changed
- **Raised default `max_diff_chars` from 60000 → 250000.** The original limit was conservative; `gemini-3.5-flash` has a 1M-token context window, and a legitimate feature PR (#82) was silently skipped by 507 bytes.
- **Skipped reviews now post a comment** explaining the skip reason (empty diff, or oversized diff with the byte count and limit) instead of posting nothing. Previously you couldn't tell a skipped review from a broken one without checking the Actions tab.

### Added
- **Manual re-review via `pr_number` input.** Caller workflows can add a `workflow_dispatch` trigger that forwards `pr_number`; the reusable workflow resolves the PR's refs via the API and re-runs the review. Useful for forcing a review after fixing review-tool issues or to bypass a one-off skip.

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
