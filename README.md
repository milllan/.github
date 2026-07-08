# Google Antigravity Code Reviewer (.github)

This repository contains the global, reusable GitHub Actions workflow for the **Google Antigravity CLI (`agy`) reviewer**. It automates pull request reviews by running static analysis, detecting bugs, security issues, performance flaws, and style violations, then posting feedback directly as PR comments.

---

## 🚀 Reusable Workflow (`agy-reviewer.yml`)

The central workflow is located at:
`milllan/.github/.github/workflows/agy-reviewer.yml` (relative to your organization/user setup).

### Key Features
- **Auto-Language Detection**: Detects Node.js, Python, Go, and Rust, setting up their respective runtimes automatically.
- **Smart Git Diff**: Generates precise diffs comparing the PR branch against the target base branch.
- **Cost & Token Optimized**: Pipes only the actual code diff to the AI engine for efficient token consumption.
- **PR Comments**: Posts feedback directly to the PR using GitHub CLI.

---

## 🛠️ Usage in Other Repositories

To use this reviewer in other repositories in your GitHub organization, create a workflow file (e.g., `.github/workflows/review.yml`) in the target repository with the following minimal setup:

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  run-reviewer:
    # Reference the reusable workflow in this repository
    uses: milllan/.github/.github/workflows/agy-reviewer.yml@main
    secrets:
      AGY_API_KEY: ${{ secrets.AGY_API_KEY }}
```

> [!IMPORTANT]
> Replace `<YOUR_GITHUB_ORG_OR_USER>` with the name of your GitHub organization or user account where the `.github` repository resides.

---

## 🔑 Setup & Configuration

1. **Get an Antigravity API Key**: Ensure you have a valid API key for Google Antigravity.
2. **Add Repository Secrets**:
   - Go to your repository **Settings** -> **Secrets and variables** -> **Actions**.
   - Create a new repository secret:
     - **Name**: `AGY_API_KEY`
     - **Value**: *Your Google Antigravity API Key*
3. **Permissions**: The calling workflow requires `pull-requests: write` permissions to allow posting comments.

---

## 📝 License

This project is licensed under the Apache License 2.0.
