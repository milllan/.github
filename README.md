# AI Code Reviewer (.github)

Reusable GitHub Actions workflow that runs AI code review on pull requests and posts the result as a PR comment. Supports independent reviewers per PR: **Google Gemini**, any **OpenAI-compatible** model (defaults to Z.ai GLM), and **OpenRouter** (OpenAI-compatible, with a model fallback list).

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
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: gemini, model: gemini-3.5-flash }
    secrets: { GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }} }
  glm-review:
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: openai, model: glm-5.2 }
    secrets: { OPENAI_API_KEY: ${{ secrets.ZAI_API_KEY }} }
  openrouter-review:
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with:
      provider: openrouter
      model: tencent/hy3:free
      models: tencent/hy3:free anthropic/claude-3.5-haiku google/gemini-flash-1.5
    secrets: { OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }} }
  nim-review:
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with:
      provider: nim
      model: z-ai/glm-5.2
    secrets: { NVIDIA_API_KEY: ${{ secrets.NVIDIA_API_KEY }} }
  zen-deepseek-review:
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: zen, model: deepseek-v4-flash-free }
    secrets: { OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }} }
  zen-mimo-review:
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: zen, model: mimo-v2.5-free }
    secrets: { OPENCODE_API_KEY: ${{ secrets.OPENCODE_API_KEY }} }
```

Replace `<SHA>` with a pinned commit from [`milllan/.github/commits/main`](https://github.com/milllan/.github/commits/main) — do **not** use `@main`. Bump the SHA deliberately after reviewing changes.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `provider` | `gemini` | `gemini`, `openai` (OpenAI-compatible endpoint), `openrouter` (OpenRouter, OpenAI-compatible), `nim` (NVIDIA NIM, OpenAI-compatible, e.g. `z-ai/glm-5.2`), or `zen` (OpenCode Zen gateway, OpenAI-compatible, e.g. `deepseek-v4-flash-free`, `mimo-v2.5-free`). |
| `model` | `gemini-3.5-flash` | Model name for the chosen provider (e.g. `glm-5.2`, `tencent/hy3:free`, `z-ai/glm-5.2`, `deepseek-v4-flash-free`). For `openai`/`openrouter`/`nim`/`zen`, this is used **only when `models` is empty** — if `models` is set, it fully overrides `model`. |
| `models` | `""` | Space- or comma-separated fallback list for `openai`/`openrouter`/`nim`/`zen`. Tried in order; the next is used if one is removed/deprecated (HTTP 400/404/422) or all retries fail. If it is **non-empty after splitting**, it fully replaces `model` as the ordered list to try; if it parses to nothing (e.g. only separators), `model` is used instead. Empty = only `model` is used. Ignored for `gemini`. |
| `openai_endpoint` | `https://api.z.ai/api/coding/paas/v4/chat/completions` | OpenAI-compatible endpoint for `openai`. Defaults to Z.ai's **Coding Plan** (subscription). Use `https://api.z.ai/api/paas/v4/chat/completions` for pay-per-token API credits. |
| `openrouter_endpoint` | `https://openrouter.ai/api/v1/chat/completions` | Chat-completions endpoint for `openrouter`. Only override if you self-host or proxy OpenRouter. |
| `nim_endpoint` | `https://integrate.api.nvidia.com/v1/chat/completions` | Chat-completions endpoint for `nim` (NVIDIA NIM). Only override if you self-host or proxy NIM. |
| `zen_endpoint` | `https://opencode.ai/zen/v1/chat/completions` | Chat-completions endpoint for `zen` (OpenCode Zen). Only override if you self-host or proxy Zen. |
| `max_diff_chars` | `250000` | Skip review if the raw diff exceeds this. `0` disables the limit. |

## Secrets

| Secret | Required when | Notes |
|--------|---------------|-------|
| `GEMINI_API_KEY` | `provider=gemini` | Create at https://aistudio.google.com/apikey |
| `OPENAI_API_KEY` | `provider=openai` | For Z.ai, create at https://z.ai/apikey |
| `OPENROUTER_API_KEY` | `provider=openrouter` | Create at https://openrouter.ai/keys |
| `NVIDIA_API_KEY` | `provider=nim` | Create at https://build.nvidia.com/settings/api-keys |
| `OPENCODE_API_KEY` | `provider=zen` | Create at https://opencode.ai/auth (OpenCode Zen). Free models need no billing. |
| `GITHUB_TOKEN` | always | Auto-provided; used to post the PR comment. |

The `GITHUB_TOKEN` is provided automatically by Actions — don't add it as a secret.

## Behavior notes

- **Triggers on** `pull_request` (`opened`, `synchronize`, `reopened`) — every commit gets re-reviewed.
- **Four comments per PR** (or more) when all jobs are enabled, headed `## Gemini Code Review`, `## GLM Code Review`, `## OpenRouter Code Review (model)`, `## NVIDIA NIM Code Review (model)`, and `## OpenCode Zen Code Review (model)`.
- **Model fallback** (OpenRouter/OpenAI/NIM/Zen): if a model in `models` is removed/deprecated, the next one is tried automatically; the comment heading names the model that actually reviewed.
- **Reasoning-model fallback**: OpenAI-compatible providers fall back to `.choices[0].message.reasoning` when `content` is empty (e.g. `mimo-v2.5-free`), so reasoning-only models still produce a review comment.
- **NIM thinking modes** (per-model, not a global flag): each model has its own thinking param schema, verified by direct probes — `z-ai/glm-5.2` uses `chat_template_kwargs.enable_thinking`, `minimaxai/minimax-m3` uses `chat_template_kwargs.thinking_mode: "enabled"`, `thinkingmachines/inkling` uses top-level `reasoning_effort: "high"`, `deepseek-ai/deepseek-v4-{pro,flash}` and `stepfun-ai/step-3.7-flash` use `chat_template_kwargs.thinking: true` (step-3.7-flash *requires* it). See [AGENTS.md](./AGENTS.md#nim-thinking-schemas-per-model) for the full verified table and how to add a new model.
- **Cost guard**: diffs over `max_diff_chars` are skipped with a visible comment (not a silent no-op).
- **Retries** transient API errors (429/502/503/504) with backoff before giving up.
- **Graceful degradation**: if a provider key is missing or its API errors, that job posts an error comment; the other reviewer still runs.

## Manual re-review

Add a `workflow_dispatch` job to the caller that pushes an empty commit to the PR branch (re-fires `synchronize`). See the [upwork-callback caller](https://github.com/milllan/upwork-callback/blob/master/.github/workflows/gemini-review.yml) for a working example. GitHub Actions does not allow `workflow_dispatch` to invoke a reusable-workflow job directly.

## License

Apache License 2.0.
