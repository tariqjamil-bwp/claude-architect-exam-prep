# CI Notes: Claude as a CI Worker

Exam-prep notes on using Claude Code in CI/CD, built around GitHub and the `gh` CLI.

## Contents

1. [Get the `gh` CLI](#1-get-the-gh-cli)
2. [CLI concepts using a `gh` project](#2-cli-concepts-using-a-gh-project)
3. [Live demo: Claude as a CI worker](#3-live-demo-claude-as-a-ci-worker)
4. [In-depth configuration of Claude Code as a CI worker](#4-in-depth-configuration-of-claude-code-as-a-ci-worker)

---

## 1. Get the `gh` CLI

`gh` is GitHub's official command-line tool. It handles issues, pull requests, workflow runs and API calls from the terminal or from a CI job.

### Install

| Platform | Command |
|----------|---------|
| macOS | `brew install gh` |
| Windows | `winget install --id GitHub.cli` |
| Debian/Ubuntu | `sudo apt install gh` (after adding the official GitHub apt repo) |
| Fedora/RHEL | `sudo dnf install gh` |
| GitHub Actions | Preinstalled on `ubuntu-latest` runners |

Check the install with `gh --version`.

### Authenticate

- Interactive: `gh auth login`, then follow the prompts.
- Non-interactive (CI): set `GH_TOKEN` (or `GITHUB_TOKEN`) in the environment.
  In Actions: `env: GH_TOKEN: ${{ github.token }}`.
- Check status with `gh auth status`.

---

## 2. CLI concepts using a `gh` project

Use a small repo as a sandbox and practise the core command groups.

### Command groups

| Group | Purpose | Examples |
|-------|---------|----------|
| `gh repo` | Create, clone, view repos | `gh repo clone owner/name`, `gh repo view` |
| `gh issue` | Manage issues | `gh issue create`, `gh issue list`, `gh issue comment 14` |
| `gh pr` | Manage pull requests | `gh pr create`, `gh pr checks`, `gh pr diff`, `gh pr merge` |
| `gh run` | Inspect workflow runs | `gh run list`, `gh run view <id> --log-failed` |
| `gh workflow` | Trigger and list workflows | `gh workflow run ci.yml` |
| `gh api` | Call any REST/GraphQL endpoint | `gh api repos/{owner}/{repo}/issues` |
| `gh secret` | Manage Actions secrets | `gh secret set NAME` |

### Concepts worth knowing

- **Scriptable output:** `--json <fields>` combined with `--jq '<expr>'` gives machine-readable output.
  Example: `gh pr list --json number,title --jq '.[].title'`
- **Context inference:** inside a clone, `gh` infers the repo from the git remote. Elsewhere, use `-R owner/name`.
- **Auth model:** a token's scopes and the workflow's `permissions:` block limit what `gh` can do.
- **Exit codes:** `gh` returns non-zero on failure, so it fits in shell conditionals in CI.
- **Why it matters for Claude:** Claude Code can run `gh` through its Bash tool, but only if the command is allowed (see section 4).

### Practice exercise

1. Create a sandbox repo: `gh repo create ci-sandbox --public --clone`.
2. Open an issue: `gh issue create --title "Test" --body "Hello"`.
3. Branch, commit, push, then run `gh pr create --fill`.
4. Watch checks with `gh pr checks --watch`.
5. If a run fails, read the logs with `gh run view --log-failed`.

---

## 3. Live demo: Claude as a CI worker

This repo is itself the demo. The flow:

1. Someone opens an issue (for example, #14).
2. A comment containing `@claude` triggers the **Claude Code** workflow (`.github/workflows/claude.yml`).
3. Claude reads the issue, works on a branch (`claude/issue-<n>-...`), commits and pushes.
4. Claude posts progress in a single updating comment and links a prefilled PR.
5. Other workflows (for example `markdown-only.yml`) run against the PR as usual.

### Demo script

1. Open an issue describing a small change.
2. Comment `@claude work on the issue and create a pr to main`.
3. Open the Actions tab and watch the **Claude Code** job.
4. Review the resulting branch and PR like any other contributor's.
5. Optionally comment `@claude` on the PR to request follow-up edits.

### What to point out

- Claude acts as a normal contributor: a branch, commits, a PR.
- A human still reviews and merges. Claude cannot approve PRs.
- Claude cannot edit `.github/workflows/` (GitHub App permissions block it).
- This repo's CI enforces Markdown-only files, so Claude's changes must pass the same check.

---

## 4. In-depth configuration of Claude Code as a CI worker

The integration is the `anthropics/claude-code-action@v1` GitHub Action.

### 4.1 Triggers

```yaml
on:
  issue_comment:        { types: [created] }
  pull_request_review_comment: { types: [created] }
  issues:               { types: [opened, assigned] }
  pull_request_review:  { types: [submitted] }
```

Gate each job with an `if:` that checks for `@claude`, so it only runs when mentioned. Without the gate, every event starts a run.

### 4.2 Authentication

| Method | Input / secret |
|--------|----------------|
| Claude subscription OAuth token | `claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}` |
| Anthropic API key | `anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}` |
| Bedrock / Vertex | Cloud credentials via OIDC (`id-token: write`) |

Always keep credentials in repository secrets, never in the workflow file.

### 4.3 Permissions (least privilege)

```yaml
permissions:
  contents: read        # raise to write if Claude should push commits
  pull-requests: read   # raise to write to comment on PRs
  issues: read
  id-token: write       # OIDC token exchange
  actions: read         # lets Claude read CI results
```

Grant only what the task needs. Review-only workflows can stay read-only.

### 4.4 Action inputs

- `prompt`: fixed instructions. If omitted, Claude follows the text of the `@claude` comment.
- `claude_args`: CLI flags passed to Claude Code, for example:
  - `--allowed-tools "Bash(gh pr *)"` allows only specific tools or commands.
  - `--max-turns 10` caps the length of a run.
  - `--model <model-id>` selects the model.
- `additional_permissions`: for example `actions: read`, so Claude can read workflow run results.
- `trigger_phrase`: change `@claude` to something else.

### 4.5 Project guidance: `CLAUDE.md`

Claude reads `CLAUDE.md` in the repo root. Put build and test commands, code style, and constraints there. In this repo, a useful rule would be "only Markdown files are allowed".

### 4.6 Tool allowlisting

Claude's tools are restricted by default in CI. Allow only what is needed, for example:

```yaml
claude_args: |
  --allowed-tools "Bash(npm test),Bash(gh pr view *),Bash(gh pr diff *)"
```

### 4.7 Common patterns

| Pattern | Setup |
|---------|-------|
| Interactive helper | `@claude` mentions on issues and PRs (this repo) |
| Automatic PR review | `pull_request` trigger plus a review `prompt` (`claude-code-review.yml`) |
| Scheduled maintenance | `schedule:` trigger plus a fixed `prompt` |
| Fix failing CI | Trigger on `workflow_run` failure, with `actions: read` so Claude can read logs |

### 4.8 Security considerations

- Treat issue and PR text as untrusted input (prompt injection risk). Restrict who can trigger the workflow.
- Use least-privilege `permissions:` and a narrow tool allowlist.
- Keep secrets out of logs and prompts.
- Require human review before merge. Use branch protection.
- Set limits (`--max-turns`, job `timeout-minutes`) to bound cost.

### 4.9 Likely exam angles

- Which trigger events start Claude, and how to gate them.
- Which permissions each task needs (read versus write).
- How `CLAUDE.md`, `prompt`, and `claude_args` each shape behavior.
- Why a tool allowlist and human review matter for safety.
- How Claude's output shows up: branch, commits, comment, PR.
