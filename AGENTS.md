# AGENTS.md

This repo (`milllan/.github`) hosts a reusable GitHub Actions workflow for AI code review. This file is the reference an AI agent should use to add the reviewer to a new repository.

## The product

`/.github/workflows/gemini-reviewer.yml` — a `workflow_call`-only reusable workflow. It does NOT run on its own; it is invoked by a caller workflow in another repo. Despite the filename, it supports five providers: Gemini, any OpenAI-compatible endpoint (default Z.ai GLM), OpenRouter (OpenAI-compatible, with an automatic model fallback list), NVIDIA NIM (OpenAI-compatible), and OpenCode Zen (OpenAI-compatible).

Current pinned HEAD: see [`commits/main`](https://github.com/milllan/.github/commits/main). Always pin callers to a specific SHA.

## How to add the reviewer to a repo

1. Create `.github/workflows/code-review.yml` in the target repo using the caller template in [`README.md`](./README.md) (or below). Pin to a SHA from `milllan/.github/commits/main`.
2. Add the required secret(s) to the repo (Settings → Secrets and variables → Actions):
   - `GEMINI_API_KEY` if using the Gemini job (get from https://aistudio.google.com/apikey)
   - `ZAI_API_KEY` if using the GLM job (get from https://z.ai/apikey) — forwarded as `OPENAI_API_KEY` in the caller
   - `OPENROUTER_API_KEY` if using the OpenRouter job (get from https://openrouter.ai/keys) — forwarded as `OPENROUTER_API_KEY` in the caller
   - `NVIDIA_API_KEY` if using the NVIDIA NIM job (get from https://build.nvidia.com/settings/api-keys) — forwarded as `NVIDIA_API_KEY` in the caller
   - `OPENCODE_API_KEY` if using the OpenCode Zen job (get from https://opencode.ai/auth) — forwarded as `OPENCODE_API_KEY` in the caller. Free models (e.g. `deepseek-v4-flash-free`, `mimo-v2.5-free`) need no billing.
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
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: gemini, model: gemini-3.5-flash }
    secrets: { GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }} }
```

### Multi-reviewer caller (Gemini + GLM + OpenRouter)

```yaml
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
    with: { provider: nim, model: z-ai/glm-5.2 }
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
- **openrouter**: `{openrouter_endpoint}` (default `https://openrouter.ai/api/v1/chat/completions`) with `Authorization: Bearer ${OPENROUTER_API_KEY}`, same response shape as openai
- **nim**: `{nim_endpoint}` (default `https://integrate.api.nvidia.com/v1/chat/completions`, NVIDIA NIM) with `Authorization: Bearer ${NVIDIA_API_KEY}`, same response shape as openai. Model names follow NIM's `owner/model` scheme, e.g. `z-ai/glm-5.2`, `moonshotai/kimi-k2.6`, `minimaxai/minimax-m3`, `thinkingmachines/inkling`.

### NIM thinking schemas (per-model)

NIM models do **not** share a single "thinking" flag — each model picks its own schema, and NIM silently ignores unknown keys (so the wrong flag is silently a no-op, never an error). The workflow's `nim` body builder dispatches per model name. The schemas below are **verified by direct curl probes** against `integrate.api.nvidia.com/v1/chat/completions` with a real key (2026-07-21), not just by reading docs — NIM's docs and its deployed runtime disagree on several models.

**Verified working (HTTP 200 with thinking ON):**

| Model | Schema | Notes |
|-------|--------|-------|
| `z-ai/glm-5.2` | `chat_template_kwargs: { enable_thinking: true, clear_thinking: true }` | `clear_thinking: true` strips the reasoning trace so only the final review lands in the PR comment. **Hangs intermittently from non-CI IPs** — see gotchas below. |
| `minimaxai/minimax-m3` | `chat_template_kwargs: { thinking_mode: "enabled" }` | Documented at [docs.api.nvidia.com/nim/reference/minimaxai-minimax-m3-infer](https://docs.api.nvidia.com/nim/reference/minimaxai-minimax-m3-infer). Plain body also works (adaptive mode). |
| `thinkingmachines/inkling` | top-level `reasoning_effort: "high"` | OpenAI o1-style. Plain body also works. |
| `deepseek-ai/deepseek-v4-pro` | `chat_template_kwargs: { thinking: true }` | Plain body also works. |
| `deepseek-ai/deepseek-v4-flash` | `chat_template_kwargs: { thinking: true }` | Plain body also works. |
| `stepfun-ai/step-3.7-flash` | `chat_template_kwargs: { thinking: true }` | **Required** — plain body hangs. The only model in the lineup that REQUIRES a thinking flag to respond at all. |

**Verified broken (as of 2026-07-21):**

| Model | Failure | Cause |
|-------|---------|-------|
| `moonshotai/kimi-k2.6` | HTTP 404 `Function '...': Not found for account '9WY0...'` | Account entitlement — this account doesn't have kimi access. Not fixable without changing the NVIDIA account tier. |

To add a new NIM model:
1. **Probe it directly first** (not just CI — CI verification is unreliable because GitHub runners hit different NIM backends). Save a key to `~/.config/shell/.nimrc`, then:
   ```bash
   source ~/.config/shell/.nimrc
   curl -sS --max-time 30 -X POST https://integrate.api.nvidia.com/v1/chat/completions \
     -H "Authorization: Bearer $NVIDIA_API_KEY" -H "Content-Type: application/json" \
     -d '{"model":"<owner>/<model>","messages":[{"role":"user","content":"hi"}],"chat_template_kwargs":{"thinking":true}}'
   ```
   Try `thinking`, `enable_thinking`, `thinking_mode:"enabled"`, `reasoning_effort:"high"`, and plain body — pick whichever returns HTTP 200 with content.
2. Add a `case` to the `nim)` branch of `build_body()` in the workflow with the verified schema.
3. Document the result in the tables above.

Don't blanket-apply any single flag — that was the v1.7.1 bug (wrong key name for GLM, no-op for everything else).
- **zen**: `{zen_endpoint}` (default `https://opencode.ai/zen/v1/chat/completions`, OpenCode Zen gateway) with `Authorization: Bearer ${OPENCODE_API_KEY}`, same response shape as openai. Free models include `deepseek-v4-flash-free` and `mimo-v2.5-free`. Reasoning-only models (e.g. `mimo-v2.5-free`) return `content:null`; the workflow falls back to `.choices[0].message.reasoning` so they still post a review.

The OpenAI-compatible branch (`openai`/`openrouter`/`nim`/`zen`) supports a **model fallback list**: the `models` input (space/comma-separated) is tried in order; if a model returns HTTP 400/404/422 (removed/deprecated) the next is used. When `models` is set it fully overrides the single `model` input (which is only used when `models` is empty); `models` is ignored for `gemini`. Permanent 401/403 or balance/quota 429 fail fast (shared key). The comment heading names the model that actually reviewed, e.g. `## OpenRouter Code Review (tencent/hy3:free)`, `## NVIDIA NIM Code Review (z-ai/glm-5.2)`, or `## OpenCode Zen Code Review (deepseek-v4-flash-free)`.

To add a provider whose API differs from both (e.g. direct Anthropic), add a new branch to the `case $PROVIDER` in the "Run Review" step and a new input default + secret. OpenAI-compatible providers (OpenRouter, NVIDIA NIM, OpenCode Zen, DeepSeek, Mistral, Groq) need no code change — just a different `*_endpoint` and key.

## Known limitations / gotchas

- Reusable workflows cannot be triggered by `workflow_dispatch` directly (`startup_failure`). Manual re-review works by pushing an empty commit to the PR branch via a separate `runs-on` job in the caller.
- Branch names with slashes (e.g. `feat/foo`) must be URL-encoded (`%2F`) when calling the `git/refs/heads/` API.
- `max_diff_chars` defaults to 250000 — large but finite. Very large PRs are skipped with a visible comment.
- The Gemini free-tier API key has `limit: 0` quota for Pro models (2.5-pro, 3.x-pro-preview) — only Flash models work without billing.
- **Z.ai has TWO endpoints.** The **Coding Plan** (subscription, what most users have) is at `https://api.z.ai/api/coding/paas/v4/chat/completions` — this is the default. The **pay-per-token API** is at `https://api.z.ai/api/paas/v4/chat/completions` and requires a positive credit balance ($0 by default → `429 insufficient balance`). The Coding Plan key works on both endpoints, but the API-credits key only works on the second. The default `openai_endpoint` is the Coding Plan one.
- The retry logic distinguishes transient `429`/`5xx` (retried with backoff) from permanent `429`s like "insufficient balance" / "quota exceeded" (fail fast).
- **NIM availability is region/account-dependent — verify with direct probes, not CI.** NIM's catalog (`GET /v1/models`) listing a model is not proof the runtime serves it for your account or from your IP. Two failure modes seen (2026-07-21):
  - **Account entitlement:** `moonshotai/kimi-k2.6` returns HTTP 404 `Function '...': Not found for account '9WY0...'` — the account doesn't have access. Not fixable without changing the NVIDIA account tier.
  - **IP/region hang:** `z-ai/glm-5.2` hangs (HTTP 000, curl `--max-time` cutoff) from some IPs (e.g. Serbia home/office) but responds fine from GitHub Actions runners. CI "success" on GLM is partly region lottery.
  - `deepseek-ai/deepseek-v4-pro` previously looked like a CI-only 404 (PR #16) but works perfectly from a direct probe. The CI failures were transient/backend-side, not param or entitlement issues.

  **Before wiring a new NIM model into a caller, probe it directly** with an authenticated `curl` from a non-CI machine (save a key to `~/.config/shell/.nimrc` per the "NIM thinking schemas" section above). A catalog listing + a CI run are not sufficient proof. When a model does fail in production, the job posts a visible `:warning:` comment per PR (graceful degradation) so the breakage is loud rather than silent.
- **Some NIM models REQUIRE a thinking flag to respond at all.** `stepfun-ai/step-3.7-flash` hangs on plain body but returns in <1s with `chat_template_kwargs.thinking: true`. The per-model dispatch in `build_body()` handles this, but it's a sharp edge: removing the thinking flag from a model that needs it will silently break that model.

## Files

- `.github/workflows/gemini-reviewer.yml` — the reusable workflow itself
- `README.md` — human-facing reference (same content, leaner)
- `CHANGELOG.md` — version history; check this before bumping a caller SHA
