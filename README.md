# Gemini Code Reviewer (.github)

This repository hosts a **reusable GitHub Actions workflow** that runs AI-powered code review on pull requests using the **Google Gemini API**. It posts the review directly as a PR comment.

It is designed to be referenced from any other repository in your account — add a small caller workflow there and every PR gets reviewed automatically.

---

## 🚀 Reusable Workflow (`gemini-reviewer.yml`)

Located at `milllan/.github/.github/workflows/gemini-reviewer.yml`.

### Key Features
- **Diff-driven**: pipes only the PR's actual `git diff` to Gemini, keeping token usage and cost low.
- **API-key auth**: a single `GEMINI_API_KEY` secret. No browser OAuth, no service account — works in headless CI.
- **Safe prompt assembly**: the request JSON is built with `jq --rawfile`, so diff contents containing quotes, backticks, or `$` cannot break parsing.
- **Cost guard**: skips review when the diff exceeds a configurable size (`max_diff_chars`, default 60000).
- **Language detection**: detects Node.js / Python / Go / Rust and sets up the matching runtime (retained for future tooling).
- **PR comments**: posts the review via the GitHub CLI (`gh`).

> [!NOTE]
> GitHub does **not** auto-inherit a workflow across all your repos. This is a *reusable* workflow — each repo that wants reviews adds the small caller shown below. There is no silent account-wide trigger.

---

## 🛠️ Usage in Other Repositories

1. **Get a Gemini API key** at https://aistudio.google.com/apikey.

2. **Add it as a repository secret** named `GEMINI_API_KEY`:
   **Settings → Secrets and variables → Actions → New repository secret**.

3. **Add a caller workflow** (e.g. `.github/workflows/gemini-review.yml`) in the target repository:

   ```yaml
   name: Gemini Code Review

   on:
     pull_request:
       types: [opened, synchronize, reopened]

   permissions:
     contents: read
     pull-requests: write

   jobs:
     review:
       uses: milllan/.github/.github/workflows/gemini-reviewer.yml@main
       secrets:
         GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
   ```

### Optional inputs
| Input | Default | Description |
|-------|---------|-------------|
| `model` | `gemini-3.5-flash` | Gemini model name (e.g. `gemini-3.1-pro` for deeper reasoning). |
| `max_diff_chars` | `60000` | Skip the review if the raw diff is larger (cost guard). `0` disables the limit. |

Example with a different model:

```yaml
    uses: milllan/.github/.github/workflows/gemini-reviewer.yml@main
    with:
      model: gemini-3.1-pro
    secrets:
      GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
```

---

## 🔑 Setup & Configuration

1. **Gemini API key**: create one at https://aistudio.google.com/apikey.
2. **Repository secret**: add `GEMINI_API_KEY` to the *calling* repository (the one with PRs), not this `.github` repo.
3. **Permissions**: the caller workflow must grant `pull-requests: write` so the action can post comments.

---

## 📝 License

This project is licensed under the Apache License 2.0.
