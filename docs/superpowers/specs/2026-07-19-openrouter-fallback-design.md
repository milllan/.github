# Design: OpenRouter provider + model fallback list

**Date:** 2026-07-19
**Status:** Implemented (reusable workflow `.github/workflows/gemini-reviewer.yml`)
**Replaces/extends:** the `openai` provider added in v1.3.0

## Context

The reusable reviewer supported two providers: `gemini` and any OpenAI-compatible
endpoint via `provider: openai` (default Z.ai GLM). The user wanted a third
reviewer using **OpenRouter** with the free `tencent/hy3:free` model, and — critically —
an automatic **fallback list of models**, so that when a model is later removed or
deprecated by the provider, the workflow silently switches to the next model instead
of failing the review.

OpenRouter's API is OpenAI-compatible (`/v1/chat/completions`, same response shape),
so no new request/parse code was needed — only a dedicated provider branch for the
default endpoint + secret, plus a model-loop around the existing call path.

## Design decisions (confirmed with user)

1. **Secret handling (security):** `milllan/.github` is a *public* repo (required for
   cross-repo reusable workflows). The OpenRouter key must **never** be committed. It is
   stored only as a GitHub Actions repo secret (`OPENROUTER_API_KEY`), added via
   `gh secret set` from the key found in a local env file — never written to git.
2. **Provider shape:** a new first-class `provider: openrouter` value (not just reusing
   `openai`), so the comment label and default endpoint are correct. Internally it reuses
   the OpenAI-compatible call/parse path.
3. **Fallback scope:** the `models` fallback list applies to `openai` + `openrouter`
   only. Gemini stays single-model (it was deemed unnecessary there).

## Mechanism

### Inputs (added/changed)
- `provider`: now `gemini | openai | openrouter`.
- `model`: for `openai`/`openrouter`, the **first** model tried.
- `models` (new, default `""`): space- or comma-separated fallback list for
  `openai`/`openrouter`. Empty ⇒ only `model` is used.
- `openai_endpoint`: for `openai` only, defaults to Z.ai's Coding Plan endpoint.
- `openrouter_endpoint` (new, default `https://openrouter.ai/api/v1/chat/completions`):
  for `openrouter` only. `ENDPOINT` is resolved from `OPENROUTER_ENDPOINT` (not
  `OPENAI_ENDPOINT`), so the openrouter call always targets OpenRouter unless
  explicitly overridden. This was the fix for an earlier bug where openrouter
  inherited the Z.ai `openai_endpoint` default and got `401`.

### Secrets (added)
- `OPENROUTER_API_KEY` (required: false) — forwarded as env `OPENROUTER_API_KEY`.

### "Run Review" step restructure
The step now has two loops:
- **Outer loop:** iterates `MODEL_LIST` (built from `models`, else `[model]`).
- **Inner loop:** the pre-existing 4-attempt transient-retry loop
  (502/503/504 and transient `429` get backoff; permanent `429` like
  `balance`/`quota`/`invalid key` fails fast).

Per-attempt classification after a non-200:
- `502|503|504` or transient `429` → **retry same model** (inner loop).
- `400|404|422` (model removed/deprecated) → **unavailable** → break inner, try next model.
- `401|403` or permanent `429` → **fatal** → fail fast (shared key, other models would
  fail identically), post error comment, exit.
- any other code → **fatal** → post error comment, exit.

On success (`200`) the model is recorded as `used_model` and both loops break. If the
list is exhausted, an "All models failed" comment is posted.

### Comment heading
For `openai`/`openrouter` the heading includes the model that actually reviewed, e.g.
`## OpenRouter Code Review (tencent/hy3:free)`. This makes a fallback event visible.

## Testing

- **Local:** a mock OpenAI-compatible server returned `404` for `removed-model` and `200`
  for `good-model`; running the extracted "Run Review" script with
  `PROVIDER=openrouter MODELS="removed-model good-model"` correctly skipped the dead
  model and posted `## OpenRouter Code Review (good-model)`.
- **Live:** add a caller in `milllan/.github` referencing the local workflow path
  (`./.github/workflows/gemini-reviewer.yml`, no SHA needed for same-repo), open a PR,
  and confirm a third `## OpenRouter Code Review (...)` comment appears. The
  `OPENROUTER_API_KEY` secret is added via `gh secret set` (never committed).

## Rollout
Callers bump their pinned `@<SHA>` and add the `openrouter-review` job + secret to adopt
the third reviewer (see `README.md` / `AGENTS.md` multi-reviewer template).
