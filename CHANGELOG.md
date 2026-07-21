# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.9.0] - 2026-07-21

### Changed
- **Per-model NIM dispatch expanded and verified by direct probes.** All thinking-param schemas are now backed by direct `curl` tests against `integrate.api.nvidia.com` with a real key (2026-07-21), not by reading NVIDIA docs (which disagree with the deployed runtime on several models). The `nim)` branch of `build_body()` now handles six verified-working models:
  - `z-ai/glm-5.2` → `chat_template_kwargs: { enable_thinking: true, clear_thinking: true }`
  - `minimaxai/minimax-m3` → `chat_template_kwargs: { thinking_mode: "enabled" }`
  - `thinkingmachines/inkling` → top-level `reasoning_effort: "high"`
  - `deepseek-ai/deepseek-v4-pro` and `deepseek-ai/deepseek-v4-flash` → `chat_template_kwargs: { thinking: true }`
  - `stepfun-ai/step-3.7-flash` → `chat_template_kwargs: { thinking: true }` (**required** — plain body hangs)
- **Per-request curl `--max-time` raised from 120s to 900s.** Thinking-mode responses are much slower than non-thinking: a small "say hi" prompt takes 0.8-20s, but a real PR-diff review with thinking on can take several minutes — the old 2-minute cutoff was killing legitimate responses mid-stream (HTTP 000, looked like a transient network error). 900s = 15min matches the recommended per-job `timeout-minutes: 15`, so a single attempt can complete a large-diff thinking review while still leaving the job budget to fail the run if the model genuinely hangs. Earlier draft of this change used 300s; direct timing measurements on real diffs showed that was still too short for thinking mode.
- **Removed dead language-detection steps.** The `Detect Languages` + four `Setup X` (Node/Python/Go/Rust) steps were inherited from the original `agy` CLI design, which needed runtimes installed. The current direct-curl approach uses neither; the steps were installing toolchains that no later step reads. Removes ~25s per job and 4 unused actions from the dependency graph.
- **Job name now reflects provider + model.** Was hardcoded `Gemini PR Review` for every job, which made a multi-reviewer PR's Action run page unreadable. Now `${{ inputs.provider }} review (${{ inputs.model }})` — e.g. `nim review (z-ai/glm-5.2)`.

### Corrected (earlier wrong claims)
- `deepseek-ai/deepseek-v4-pro` is **not** broken. Earlier changelogs (1.8.0 notes) called it a "404 in every PR run, never produced a review" based on CI-only evidence. Direct probes show it returns HTTP 200 in ~10s with a real key. The CI 404s were transient backend flakes, not a model or param issue. Re-added to the per-model dispatch with `chat_template_kwargs.thinking: true`.
- `minimaxai/minimax-m3` thinking mode **is** achievable. An earlier draft of 1.8.1 claimed `thinking_mode: "enabled"` hangs on the deployed runtime; that conclusion came from a CI test that was itself flaky (HTTP 000 from runner→NIM networking). Direct probes confirm `thinking_mode: "enabled"` returns HTTP 200 in ~20s.

### Verified broken (unchanged, but now with confirmed cause)
- `moonshotai/kimi-k2.6` → HTTP 404 `Function '...': Not found for account '9WY0...'`. The error body explicitly names the account, confirming this is an entitlement issue (this NVIDIA account doesn't have kimi access), not a model-deployment issue. Both available keys map to the same account, so swapping keys doesn't help.

### Docs
- AGENTS.md and README.md updated with a verified NIM thinking-schemas table (replaces the earlier "three schemas" / "four schemas" drafts that mixed correct and incorrect claims).
- New "NIM availability is region/account-dependent" gotcha documents the IP-vs-CI divergence on GLM-5.2 and the entitlement-gated kimi-k2.6, with explicit instructions to **probe directly before wiring a new model** (CI-only verification is unreliable).
- Stale "three providers" count fixed (now five: gemini, openai, openrouter, nim, zen).

## [1.8.0] - 2026-07-21

### Changed
- **Per-model NIM dispatch.** NIM models use one of three thinking schemas, and there is no single magic flag. The earlier blanket `chat_template_kwargs: { thinking: true }` (from v1.7.1) was wrong twice over: the correct GLM key is `enable_thinking` (so GLM had silently been running non-thinking), and most NIM models take no thinking params at all. The `nim` body builder now dispatches per model name:
  - `z-ai/glm-5.2` → `chat_template_kwargs: { enable_thinking: true, clear_thinking: true }` (thorough reviews, reasoning trace stripped from the comment)
  - `thinkingmachines/inkling` → `reasoning_effort: "high"` (OpenAI o1-style top-level field)
  - everything else (`moonshotai/kimi-k2.6`, `minimaxai/minimax-m3`, `deepseek-ai/*`) → plain body, no extra fields
  Verified locally against `jq`'s `*` merge: each schema produces exactly the right keys, no null injections.

### Notes
- `deepseek-ai/deepseek-v4-pro` is in the NIM catalog but the runtime endpoint has consistently returned HTTP 404 with empty body across every PR run on every repo that tried it — never produced a review. The 404 is unrelated to request params (account-entitlement or backend-deployment issue). Recommended replacement: `moonshotai/kimi-k2.6` (works today, plain body, no thinking params).

## [1.7.1] - 2026-07-20

### Fixed
- Retried HTTP `000` (curl-level failures: DNS, connection refused, timeout) the same way as `502/503/504`, instead of failing fast. Affected all providers intermittently.

### Changed
- **(Superseded by 1.8.0)** Sent `chat_template_kwargs: { thinking: true }` for all NIM requests per the build.nvidia.com example. This was based on a misreading — the correct GLM key is `enable_thinking`, and the blanket application was wrong. Replaced by per-model dispatch in 1.8.0.

## [1.7.0] - 2026-07-19

### Added
- **`zen` provider (OpenCode Zen gateway).** A fifth first-class provider, OpenAI-compatible, defaulting to `https://opencode.ai/zen/v1/chat/completions`. Uses the new `OPENCODE_API_KEY` secret and its own `zen_endpoint` input. Free models (`deepseek-v4-flash-free`, `mimo-v2.5-free`) need no billing. Comment heading becomes `## OpenCode Zen Code Review (deepseek-v4-flash-free)` so you can see which model reviewed. Shares the OpenAI-compatible request/response path and the `models` fallback list.
- **Reasoning-model fallback.** OpenAI-compatible providers now fall back to `.choices[0].message.reasoning` when `content` is empty (e.g. `mimo-v2.5-free`), so reasoning-only models still produce a review comment instead of a "no content" error.
- Callers can now run up to six parallel reviewers (Gemini + GLM + OpenRouter + NVIDIA NIM + two OpenCode Zen free models).

## [1.6.0] - 2026-07-19

### Added
- **`nim` provider (NVIDIA NIM).** A fourth first-class provider, OpenAI-compatible, defaulting to `https://integrate.api.nvidia.com/v1/chat/completions`. Uses the new `NVIDIA_API_KEY` secret and its own `nim_endpoint` input (separate from `openai_endpoint`/`openrouter_endpoint`). Model names follow NIM's `owner/model` scheme (e.g. `z-ai/glm-5.2`). Comment heading becomes `## NVIDIA NIM Code Review (z-ai/glm-5.2)` so you can see which model reviewed. Shares the OpenAI-compatible request/response path and the `models` fallback list.
- Callers can now run four parallel reviewers (Gemini + GLM + OpenRouter + NVIDIA NIM) for four independent reviews per PR.

## [1.5.0] - 2026-07-19

### Added
- **`openrouter` provider.** A third first-class provider, OpenAI-compatible, defaulting to `https://openrouter.ai/api/v1/chat/completions`. Uses the new `OPENROUTER_API_KEY` secret and its own `openrouter_endpoint` input (separate from `openai_endpoint`, which stays Z.ai's default). Comment heading becomes `## OpenRouter Code Review (model)` so you can see which model reviewed.
- **Model fallback list.** New `models` input (space/comma-separated) for `openai`/`openrouter`. Tried in order; if a model is removed/deprecated (HTTP 400/404/422) the next is used automatically. The comment heading names the model that actually succeeded.
- **Fail-fast on shared-key errors.** `401`/`403` and permanent `429` (balance/quota/key) now abort the whole run instead of burning the fallback list — the key is shared, so other models would fail identically.

### Changed
- The "Run Review" step now wraps the per-model call in an outer model loop; the existing 4-attempt transient-retry loop is the inner loop. Response parsing for `openai`/`openrouter` is shared.

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
