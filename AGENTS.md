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
   - `OPENCODE_API_KEY` if using the OpenCode Zen job (get from https://opencode.ai/auth) — forwarded as `OPENCODE_API_KEY` in the caller. Free models (e.g. `muse-spark-1.3-contributor-free`, `mimo-v2.5-free`) need no billing.
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
    with: { provider: nim, model: z-ai/glm-5.3-flash }
    secrets: { NVIDIA_API_KEY: ${{ secrets.NVIDIA_API_KEY }} }
  zen-muse-review:
    uses: milllan/.github/workflows/gemini-reviewer.yml@<SHA>
    with: { provider: zen, model: muse-spark-1.3-contributor-free }
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
| `z-ai/glm-5.3-flash` | `chat_template_kwargs: { enable_thinking: true, clear_thinking: true }` | Verified 2026-09-12 (~24s with thinking). Only GLM left in the NIM catalog — `z-ai/glm-5.2` went 410 Gone upstream 2026-09-01. Same schema as 5.2. |
| `deepseek-ai/deepseek-v4-flash-0731` | `chat_template_kwargs: { thinking: true }` | Verified 2026-09-12 (0.6s; returns `reasoning_content` alongside `content`). First hits can cold-start slow — don't mistake a slow first request for a hang. |
| `minimaxai/minimax-m3` | `chat_template_kwargs: { thinking_mode: "enabled" }` | Documented at [docs.api.nvidia.com/nim/reference/minimaxai-minimax-m3-infer](https://docs.api.nvidia.com/nim/reference/minimaxai-minimax-m3-infer). Plain body also works (adaptive mode). |
| `thinkingmachines/inkling` | top-level `reasoning_effort: "high"` | OpenAI o1-style. Plain body also works. |
| `deepseek-ai/deepseek-v4-pro` | `chat_template_kwargs: { thinking: true }` | Plain body also works. |
| `deepseek-ai/deepseek-v4-flash` | `chat_template_kwargs: { thinking: true }` | Plain body also works. |
| `stepfun-ai/step-3.7-flash` | `chat_template_kwargs: { thinking: true }` | **Required** — plain body hangs. The only model in the lineup that REQUIRES a thinking flag to respond at all. |

**Verified broken (as of 2026-09-12):**

| Model | Failure | Cause |
|-------|---------|-------|
| `moonshotai/kimi-k2.6` | HTTP 404 `Function '...': Not found for account '9WY0...'` | Account entitlement — this account doesn't have kimi access. Not fixable without changing the NVIDIA account tier. |
| `z-ai/glm-5.3` | HTTP 504 (NIM gateway timeout) on real-diff review requests from GH runners — 2/2 attempts, and `glm-5.3-flash` fallback 504'd identically (2026-09-16, PR #25). Direct probes 200 (81s trivial) — region/backend lottery. Schema mapping (`enable_thinking`) stays in build_body for re-testing. |
| `moonshotai/kimi-k3` | Hangs (HTTP 000, 0 bytes) — plain body, `reasoning_effort:"max"`, and `"high"`, from residential AND datacenter IPs (2026-09-16) | Not entitlement (that 404s instantly like k2.6) — the same hang class as `deepseek-v4-pro-0813`. Do not wire; re-probe later. |
| `z-ai/glm-5.2` | HTTP 410 Gone (empty body) | Removed from NIM upstream 2026-09-01. Successor: `z-ai/glm-5.3-flash`. |
| `deepseek-ai/deepseek-v4-pro`, `deepseek-v4-flash` | HTTP 410 "reached its end of life on 2026-08-07" | Dated rebuilds exist: `deepseek-v4-flash-0731` (works, see above), `deepseek-v4-pro-0813` (hangs on every param combo from non-CI IPs — excluded pending a CI-proven run). |

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
- **inferx**: `{inferx_endpoint}` (default `https://model.inferx.net/endpoints/v1/chat/completions`, InferX gateway) with `Authorization: Bearer ${INFERX_API_KEY}`, same response shape as openai. Verified 2026-09-12: `deepseek-v4.1-flash` 200 (3s trivial, ~66s review-shaped probe with solid findings; reasoning runs internally but only final text is returned); unknown model -> HTTP 404, which the fallback chain treats as skip-to-next-model. The gateway also serves `deepseek-v4-flash-0731` and `glm-5.3-flash` (same builds as NIM) plus Qwen/Devstral/gemma variants.
- **zen**: `{zen_endpoint}` (default `https://opencode.ai/zen/v1/chat/completions`, OpenCode Zen gateway) with `Authorization: Bearer ${OPENCODE_API_KEY}`, same response shape as openai. Free models include `muse-spark-1.2-contributor-free` and `mimo-v2.5-free`. Reasoning-only models (e.g. `mimo-v2.5-free`) return `content:null`; the workflow falls back to `.choices[0].message.reasoning` so they still post a review.

### Zen muse-spark models are Responses-API only (per-model)

Like the NIM thinking schemas, the zen provider dispatches per model: **`muse-spark*` models do not serve chat-completions**. Verified by direct curl probes against `opencode.ai/zen` with a real key (2026-09-01):

- `muse-spark-1.2-contributor-free` on `/zen/v1/chat/completions` → instant HTTP 500 `Internal server error` (6/6 attempts: plain body, with `max_tokens`, with `stream:true`). Not rate limiting — the gateway returns a structured `FreeUsageLimitError` 429 when quota is the issue.
- `muse-spark-1.2-contributor-free` on `/zen/v1/responses` (OpenAI **Responses API** shape: `input[]` with role/content parts, text under `.output[]` message items) → HTTP 200. `reasoning.effort` `high` (~15s) and `xhigh` (~18s) both verified. The workflow sends `high`.
- `muse-spark-1.3-contributor-free` verified 2026-09-12: `/zen/v1/responses` → 200 (2.9s trivial probe; 11s review-quality probe at `reasoning.effort: high` — 4/4 valid findings on a toy diff, on par with 1.2). Now the default muse lane, with 1.2 as in-family fallback; the `muse-spark*` pattern covers both with no code change.
- The model entry in opencode's own model registry declares `api: "openai-responses"` — the chat-completions 500 is a routing gap, not a transient outage.
- Paid `muse-spark-1.2` / `muse-spark-1.3` (and `muse-spark-1.2-contributor` on the `/zen/go/v1` gateway) need a payment method (`CreditsError` 401) — not usable with a free key. The free tier is the `-contributor-free` variants only.
- `deepseek-v4-flash-free` (the previous zen reviewer) went down upstream on 2026-09-01: HTTP 400 `Model is unavailable` — the fallback list or the muse swap covers it.

To add another zen Responses-API model: add its name pattern to the two `muse-spark*` cases in the workflow's zen branches (`build_body()` and the curl endpoint switch + extraction), and probe `/zen/v1/responses` directly first — the `/v1/models` catalog listing is not proof a model serves a given API family.

**Zen client-identification headers (updated 2026-09-08):** OpenCode requires external tools to send `x-opencode-session: <stable-id-per-conversation>` and an explicit `User-Agent` on the `/zen/go/v1` gateway (pi's `opencode-go` provider — enforced per opencode's announcement; see r/opencode "OpenCode Go is no longer general API access"). Enforcement on the free `/zen/v1` gateway is **IP-dependent**: residential IPs got identical responses with and without the headers (probe matrix 2026-09-08), but datacenter IPs get **HTTP 400 `MissingSessionID`** — confirmed from a VPS and inferred for GitHub runners, where the muse reviews were failing with exactly that signature until the headers shipped. The workflow sends `-A milllan-github-reviewer/1.0` and a per-run UUID session header on all zen requests. A 400 `MissingSessionID` on any zen call means headers were stripped en route (proxy, fork without the fix).

**Zen free-tier caps:** the free models have tight per-model caps — `muse-spark-1.2-contributor-free` exhausted after ~10 requests in a session on 2026-09-08 (429 `FreeUsageLimitError` "Rate limit exceeded. Please try again later."). The workflow classifies that 429 type as skip-to-next-model (not retry, not fatal) so a `models` fallback list can land on a model with remaining headroom. Expect zen free reviewers to intermittently post `:warning: unavailable` comments on heavy days; that is the quota, not a bug.

The **model fallback list** works for every provider, including gemini: the `models` input (space/comma-separated) is tried in order; if a model returns HTTP 400/404/410/422 (removed/deprecated) the next is used. When `models` is set it fully overrides the single `model` input (which is only used when `models` is empty). Permanent 401/403 or balance/quota 429 fail fast (shared key). The comment heading names the model that actually reviewed, e.g. `## Gemini Code Review (gemini-flash-latest)`, `## NVIDIA NIM Code Review (z-ai/glm-5.3-flash)`, or `## OpenCode Zen Code Review (muse-spark-1.3-contributor-free)`. The fallback list may mix API families — verified 2026-09-01: `models: deepseek-v4-flash-free muse-spark-1.2-contributor-free` skips the down deepseek (400) and reviews via muse (200).

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

## Reviewer scorecard

Each PR gets multiple review comments from different models. They vary wildly in quality — some consistently find real bugs, others mostly hallucinate. **Track which is which** so you (and any agent working in this repo) know which reviews to trust and which to discount.

### Methodology (apply on every PR batch)

For every concrete technical claim in every review comment, bucket it:

- **VALID** — factually correct AND actionable (would improve the code if fixed). Verify by reading the actual code; don't take the reviewer's word.
- **NIT** — technically correct but cosmetic (whitespace, naming, doc phrasing). Not worth fixing standalone.
- **INVALID** — factually wrong: misreads the code, hallucinates APIs/model IDs/GitHub Actions behavior, or relies on a stale training-cutoff notion of what models exist.

Attribute each claim to the model that made it. A single comment can yield many claims. Compute `score = VALID − INVALID` per model (NITs are neutral). Common INVALID patterns to watch for:

- **"Model X doesn't exist / looks like a future version"** — every reviewer hallucinates this about NIM models they haven't seen in training. Discount unless verified by direct probe.
- **Hallucinated code structures** — "duplicate `secrets:` block", "no default `*)` branch", "missing `timeout-minutes`" — all real examples where reviewers invented problems from diff context that don't exist in the actual file.
- **Generic good-practice filler** — "verify the SHA is trusted", "consider adding tests" — true but contentless.

### Per-repo file

Each consuming repo should keep a `reviewer-scorecard.md` at its root (not in `.github/` — it's a project artifact, not a workflow file). Format:

```markdown
# AI reviewer quality scorecard

Tracks which review models produce useful feedback on <repo name>.
Updated after each review cycle. Score = VALID − INVALID.

## Scores (from YYYY-MM-DD cycle, N claims triaged)

| Rank | Model | VALID | NIT | INVALID | Score | Notes |
|------|-------|-------|-----|---------|-------|-------|
| 1    | ...   | ...   | ... | ...     | ...   | ...   |

## Patterns observed
- ...

## Action items implied
- ...
```

The reference implementation lives at [`milllan/.github/reviewer-scorecard.md`](./reviewer-scorecard.md) — copy its structure, then maintain your own data. Per-repo matters because model quality varies by language (a model may be great at PHP reviews but weak on TypeScript); aggregate scores in `milllan/.github` average across all callers and obscure that.

**Update practice (owner rule, 2026-08-18):** scorecard updates are committed directly to the consuming repo's default branch — never opened as PRs. Bot-reviewing pure bookkeeping wastes the review pipeline, and the models being scored must not review their own scorecard. This is a standing exception to any branch+PR-everything rule in consuming repos, scoped to `reviewer-scorecard.md` only.

### How to use the scorecard
- **When triaging a PR's reviews:** sort claims by VALID-first, weight models by historical score. A claim from a +5 model is more likely real than the same claim from a −3 model.
- **When choosing the lineup:** if a model sits below −5 across multiple cycles, replace it. If a model sits above +3, keep it even if it's slow or intermittent.
- **When adding a new model:** start it in parallel with the existing lineup for 2-3 cycles before deciding to keep it. The scorecard will tell you if it earns its slot.

## Files

- `.github/workflows/gemini-reviewer.yml` — the reusable workflow itself
- `README.md` — human-facing reference (same content, leaner)
- `CHANGELOG.md` — version history; check this before bumping a caller SHA
- `reviewer-scorecard.md` — reference implementation of the per-repo scorecard (this repo's own data)
