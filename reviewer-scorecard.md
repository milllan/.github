# AI reviewer quality scorecard

Tracks which review models produce useful feedback on the `milllan/.github` workflow.
Updated after each review cycle. Score = VALID − INVALID (NITs are neutral).

## Methodology
- Every concrete technical claim in every review comment is bucketed:
  - **VALID** — factually correct AND actionable (would improve the code if fixed).
  - **NIT** — technically correct but cosmetic (whitespace, naming, doc phrasing). Not worth fixing standalone.
  - **INVALID** — factually wrong (misreads the code, hallucinates APIs/models, false assumptions about GitHub Actions/NIM).
- Claims are attributed to the model that made them. A "models don't exist" hallucination counts against whichever model(s) made it.
- The unit is the claim, not the comment — one comment can yield several claims.

## Scores (from 2026-07-21 review cycle, 33 claims triaged across PRs #17/#22/#23)

| Rank | Model | VALID | NIT | INVALID | Score | Notes |
|------|-------|-------|-----|---------|-------|-------|
| 1 | `mimo-v2.5-free` (zen) | 3 | 2 | 2 | **+1** | Found the real #14 timeout-math bug alongside GLM. Low volume but high signal-to-noise. |
| 2 | `deepseek-v4-pro` | 2 | 2 | 2 | **0** | Co-flagged the branch-protection risk. No hallucinations on model existence. |
| 2 | `thinkingmachines/inkling` | 1 | 0 | 1 | **0** | Lower volume (only verified working intermittently). |
| 4 | `z-ai/glm-5.2` | 7 | 5 | 8 | **−1** | Highest raw volume of valid findings (including the #14 timeout bug, the empty-`model` job-name footgun, README drift). But also high hallucination rate: claimed a duplicate `secrets:` block that doesn't exist, missed a 10-line `timeout-minutes` comment while complaining about its absence. Net-neutral. |
| 4 | `minimaxai/minimax-m3` | 3 | 4 | 4 | **−1** | Co-flagged concurrency gap and the timeout math. Also flagged matrix consolidation as a "suggestion." Mid-pack. |
| 6 | `deepseek-ai/deepseek-v4-flash` | 4 | 7 | 7 | **−3** | Verbose. Many NITs, many hallucinations (glob-char FUD, `set -u` false claim, `nim_extras` "may be unset" when default branch exists). Not worth the noise vs the pro variant. *(Dropped from caller lineup for this reason — pro covers the family.)* |
| 7 | `stepfun-ai/step-3.7-flash` | 1 | 3 | 7 | **−6** | Worst signal-to-noise. Multiple "model doesn't exist" hallucinations about models it was running alongside, plus speculation about `needs:` deps that don't exist. Keep for diversity but discount heavily. |

## Patterns observed
- **"Model X doesn't exist / looks like a future version"** is the most common hallucination across ALL models — they all rely on training-cutoff NIM catalogs. Discount any such claim unless verified by direct probe.
- **GLM (with thinking on)** produces the most volume and surfaces the most real bugs, but also the most confident hallucinations. Read its reviews, don't trust them blindly.
- **mimo-v2.5-free** is the surprise standout — small but accurate. Worth keeping even though it's the "free tier" model.
- **deepseek-v4-flash** is strictly worse than **deepseek-v4-pro** from the same family. Confirms the user's call to drop flash.
- **step-3.7-flash** is currently the weakest. Its fast response time is the only thing going for it.

## Action items implied
- Keep GLM, deepseek-v4-pro, minimax-m3, mimo — they each contributed unique valid findings.
- step-3.7-flash is on thin ice; if a future cycle confirms low signal, consider replacing it.
- inkling is too intermittent (NIM DEGRADED status) to evaluate fairly — re-score when NIM stabilizes.

## Raw claim log (2026-07-21 cycle)

33 claims triaged across PRs #17 (workflow rewrite), #22 (caller config bump), #23 (drop flash).
Full table in session notes; key VALID findings:
1. **#14** — `--max-time 900` × 4 retries exceeds `timeout-minutes: 15` budget. (Found by mimo, GLM, minimax, deepseek-v4-flash.)
2. **#12** — Job name `${{ inputs.model }}` renders `()` if a caller sets `models` without `model`. (Found by GLM, mimo.)
3. **#15** — Gemini inherits thinking-mode `--max-time`; should be provider-scoped. (Found by deepseek-v4-flash, GLM.)
4. **#8** — Callers lack `concurrency:` group; rapid pushes fan out redundant reviews. (Found by GLM, minimax.)
5. **#22** — README "Four comments per PR" while listing 5 heading types. (Found by GLM.)
6. **#7** — Deleting caller jobs can break required-status-check branch protection. (Found by GLM, deepseek-v4-pro, deepseek-v4-flash.)
