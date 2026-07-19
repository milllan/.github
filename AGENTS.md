# AGENTS.md

This repo (`milllan/.github`) hosts a reusable GitHub Actions workflow for AI code review. This file is the reference an AI agent should use to add the reviewer to a new repository.

## The product

`/.github/workflows/gemini-reviewer.yml` — a `workflow_call`-only reusable workflow. It does NOT run on its own; it is invoked by a caller workflow in another repo. Despite the filename, it supports three providers: Gemini, any OpenAI-compatible endpoint (default Z.ai GLM), and OpenRouter (OpenAI-compatible, with an automatic model fallback list).

Current pinned HEAD: see [`commits/main`](https://github.com/milllan/.github/commits/main). Always pin callers to a specific SHA.

## How to add the reviewer to a repo

1. Create `.github/workflows/code-review.yml` in the target repo using the caller template in [`README.md`](./README.md) (or below). Pin to a SHA from `milllan/.github/commits/main`.
2. Add the required secret(s) to the repo (Settings → Secrets and variables → Actions):
   - `GEMINI_API_KEY` if using the Gemini job (get from https://aistudio.google.com/apikey)
   - `ZAI_API_KEY` if using the GLM job (get from https://z.ai/apikey) — forwarded as `OPENAI_API_KEY` in the caller
   - `OPENROUTER_API_KEY` if using the OpenRouter job (get from https://openrouter.ai/keys) — forwarded as `OPENROUTER_API_KEY` in the caller
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

### Multi-reviewer caller (Gemini + GLM + OpenRouter)

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
  openrouter-review:
    uses: milllan/.github/.github/workflows/gemini-reviewer.yml@<SHA>
    with:
      provider: openrouter
      model: tencent/hy3:free
      models: tencent/hy3:free anthropic/claude-3.5-haiku google/gemini-flash-1.5
    secrets: { OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }} }
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
- **openai**: `{openai_endpoint}` (default Z.ai Coding Plan) with `Authorization: Bearer`, response `.choices[0].message.content`
- **openrouter**: `{openai_endpoint}` (default `https://openrouter.ai/api/v1/chat/completions`) with `Authorization: Bearer ${OPENROUTER_API_KEY}`, same response shape as openai

The OpenAI-compatible branch (`openai`/`openrouter`) supports a **model fallback list**: the `models` input (space/comma-separated) is tried in order; if a model returns HTTP 400/404/422 (removed/deprecated) the next is used. Permanent 401/403 or balance/quota 429 fail fast (shared key). The comment heading names the model that actually reviewed, e.g. `## OpenRouter Code Review (tencent/hy3:free)`.

To add a provider whose API differs from both (e.g. direct Anthropic), add a new branch to the `case $PROVIDER` in the "Run Review" step and a new input default + secret. OpenAI-compatible providers (OpenRouter, DeepSeek, Mistral, Groq) need no code change — just a different `openai_endpoint` and key.

## Known limitations / gotchas

- Reusable workflows cannot be triggered by `workflow_dispatch` directly (`startup_failure`). Manual re-review works by pushing an empty commit to the PR branch via a separate `runs-on` job in the caller.
- Branch names with slashes (e.g. `feat/foo`) must be URL-encoded (`%2F`) when calling the `git/refs/heads/` API.
- `max_diff_chars` defaults to 250000 — large but finite. Very large PRs are skipped with a visible comment.
- The Gemini free-tier API key has `limit: 0` quota for Pro models (2.5-pro, 3.x-pro-preview) — only Flash models work without billing.
- **Z.ai has TWO endpoints.** The **Coding Plan** (subscription, what most users have) is at `https://api.z.ai/api/coding/paas/v4/chat/completions` — this is the default. The **pay-per-token API** is at `https://api.z.ai/api/paas/v4/chat/completions` and requires a positive credit balance ($0 by default → `429 insufficient balance`). The Coding Plan key works on both endpoints, but the API-credits key only works on the second. The default `openai_endpoint` is the Coding Plan one.
- The retry logic distinguishes transient `429`/`5xx` (retried with backoff) from permanent `429`s like "insufficient balance" / "quota exceeded" (fail fast).

## Files

- `.github/workflows/gemini-reviewer.yml` — the reusable workflow itself
- `README.md` — human-facing reference (same content, leaner)
- `CHANGELOG.md` — version history; check this before bumping a caller SHA
