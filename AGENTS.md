# AGENTS.md

This repo (`milllan/.github`) hosts a reusable GitHub Actions workflow for AI code review. This file is the reference an AI agent should use to add the reviewer to a new repository.

## The product

`/.github/workflows/gemini-reviewer.yml` — a `workflow_call`-only reusable workflow. It does NOT run on its own; it is invoked by a caller workflow in another repo. Despite the filename, it supports two providers: Gemini and any OpenAI-compatible endpoint (default Z.ai GLM).

Current pinned HEAD: see [`commits/main`](https://github.com/milllan/.github/commits/main). Always pin callers to a specific SHA.

## How to add the reviewer to a repo

1. Create `.github/workflows/code-review.yml` in the target repo using the caller template in [`README.md`](./README.md) (or below). Pin to a SHA from `milllan/.github/commits/main`.
2. Add the required secret(s) to the repo (Settings → Secrets and variables → Actions):
   - `GEMINI_API_KEY` if using the Gemini job (get from https://aistudio.google.com/apikey)
   - `ZAI_API_KEY` if using the GLM job (get from https://z.ai/apikey) — forwarded as `OPENAI_API_KEY` in the caller
3. The repo's default branch must allow Actions to post comments (`pull-requests: write` is set in the caller).
4. The reusable workflow's repo (`milllan/.github`) must be **public** — GitHub requires this for reusable workflows called across repos.

### Minimal caller (single reviewer, Gemini only)

```yaml
name: Code Review
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
  pull-requests: write
jobs:
  review:
    uses: milllan/.github/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: gemini, model: gemini-3.5-flash }
    secrets: { GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }} }
```

### Two-reviewer caller (Gemini + GLM)

```yaml
jobs:
  gemini-review:
    uses: milllan/.github/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: gemini, model: gemini-3.5-flash }
    secrets: { GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }} }
  glm-review:
    uses: milllan/.github/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: openai, model: glm-5.2 }
    secrets: { OPENAI_API_KEY: ${{ secrets.ZAI_API_KEY }} }
```

## Bumping the SHA

When `milllan/.github` publishes a workflow update (check `commits/main` and [`CHANGELOG.md`](./CHANGELOG.md)):
1. Open a PR in the consuming repo that updates the `@<SHA>` in the caller.
2. Review the changelog entry for the new SHA.
3. Merge after the review-bot on that PR confirms the new version works.

Never use `@main` — a mutable reference lets any future commit silently change what every PR review runs.

## Provider mechanics

The provider coupling is isolated to the "Run Review" step:
- **gemini**: `generativelanguage.googleapis.com/.../models/{model}:generateContent?key=...`, response `.candidates[0].content.parts[0].text`
- **openai**: `{openai_endpoint}` with `Authorization: Bearer`, response `.choices[0].message.content`

To add a third provider whose API differs from both (e.g. direct Anthropic), add a new branch to the `case $PROVIDER` in the "Run Review" step and a new input default. OpenAI-compatible providers (OpenRouter, DeepSeek, Mistral, Groq) need no code change — just a different `openai_endpoint` and key.

## Known limitations / gotchas

- Reusable workflows cannot be triggered by `workflow_dispatch` directly (`startup_failure`). Manual re-review works by pushing an empty commit to the PR branch via a separate `runs-on` job in the caller.
- Branch names with slashes (e.g. `feat/foo`) must be URL-encoded (`%2F`) when calling the `git/refs/heads/` API.
- `max_diff_chars` defaults to 250000 — large but finite. Very large PRs are skipped with a visible comment.
- The Gemini free-tier API key has `limit: 0` quota for Pro models (2.5-pro, 3.x-pro-preview) — only Flash models work without billing. GLM via Z.ai has its own quota model.

## Files

- `.github/workflows/gemini-reviewer.yml` — the reusable workflow itself
- `README.md` — human-facing reference (same content, leaner)
- `CHANGELOG.md` — version history; check this before bumping a caller SHA
