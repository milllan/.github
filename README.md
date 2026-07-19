# AI Code Reviewer (.github)

Reusable GitHub Actions workflow that runs AI code review on pull requests and posts the result as a PR comment. Supports two independent reviewers per PR: **Google Gemini** and any **OpenAI-compatible** model (defaults to Z.ai GLM).

Other repos reference it via a small caller workflow — there is no account-wide auto-inheritance.

## Caller template

Drop this into `.github/workflows/code-review.yml` in any repo that wants reviews. Both jobs are optional — keep only the reviewer(s) you want.

```yaml
name: Code Review
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
  pull-requests: write
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

Replace `<SHA>` with a pinned commit from [`milllan/.github/commits/main`](https://github.com/milllan/.github/commits/main) — do **not** use `@main`. Bump the SHA deliberately after reviewing changes.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `provider` | `gemini` | `gemini` or `openai` (OpenAI-compatible endpoint). |
| `model` | `gemini-3.5-flash` | Model name for the chosen provider (e.g. `glm-5.2`). |
| `openai_endpoint` | `https://api.z.ai/api/coding/paas/v4/chat/completions` | OpenAI-compatible endpoint. Defaults to Z.ai's **Coding Plan** (subscription). Use `https://api.z.ai/api/paas/v4/chat/completions` for pay-per-token API credits. Also works for OpenRouter, DeepSeek, OpenAI. |
| `max_diff_chars` | `250000` | Skip review if the raw diff exceeds this. `0` disables the limit. |

## Secrets

| Secret | Required when | Notes |
|--------|---------------|-------|
| `GEMINI_API_KEY` | `provider=gemini` | Create at https://aistudio.google.com/apikey |
| `OPENAI_API_KEY` | `provider=openai` | For Z.ai, create at https://z.ai/apikey |
| `GITHUB_TOKEN` | always | Auto-provided; used to post the PR comment. |

The `GITHUB_TOKEN` is provided automatically by Actions — don't add it as a secret.

## Behavior notes

- **Triggers on** `pull_request` (`opened`, `synchronize`, `reopened`) — every commit gets re-reviewed.
- **Two comments per PR** when both jobs are enabled, headed `## Gemini Code Review` and `## GLM Code Review`.
- **Cost guard**: diffs over `max_diff_chars` are skipped with a visible comment (not a silent no-op).
- **Retries** transient API errors (429/502/503/504) with backoff before giving up.
- **Graceful degradation**: if a provider key is missing or its API errors, that job posts an error comment; the other reviewer still runs.

## Manual re-review

Add a `workflow_dispatch` job to the caller that pushes an empty commit to the PR branch (re-fires `synchronize`). See the [upwork-callback caller](https://github.com/milllan/upwork-callback/blob/master/.github/workflows/gemini-review.yml) for a working example. GitHub Actions does not allow `workflow_dispatch` to invoke a reusable-workflow job directly.

## License

Apache License 2.0.
