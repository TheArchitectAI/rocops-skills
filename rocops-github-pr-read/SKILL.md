---
name: rocops-github-pr-read
description: Read GitHub PRs across TheArchitectAI org repos using the gh CLI. Use when you need to review a PR diff, read PR description/comments, check CI status, or audit recent merges. Read-only — no comments posted, no merges executed.
---

# GitHub PR read via gh CLI

Read-only access to GitHub PRs across `TheArchitectAI` org repositories. Authenticated via the GitHub CLI on this machine.

## Auth

```bash
gh auth status 2>&1 | head -5
```

Should show `Logged in to github.com account TheArchitectAI`. If not authenticated, escalate to CTO — don't run `gh auth login` from within a skill.

## Common repos under TheArchitectAI

| Repo | Purpose |
|---|---|
| `TheArchitectAI/mortgagearchitect-ai` | Mega-monorepo (parent platform + AA Mortgage tenant) |
| `TheArchitectAI/roc-ai` | AA Mortgage tenant UI shell (ai.rochomeloans.com) |
| `TheArchitectAI/architect-os` | Personal OS + vault (private) |
| `TheArchitectAI/roc-cloud-ai` | AA Realty tenant seed |
| `TheArchitectAI/roc-publisher` | Postiz content publisher |
| `TheArchitectAI/carousel-system` | DPA carousel generator |
| `TheArchitectAI/rocops-skills` | Paperclip agent skills (this repo) |

See `[[reference_aa_repo_ledger_2026-05-17]]` for the full canonical list.

## Read a specific PR

```bash
gh pr view 15 --repo TheArchitectAI/roc-ai --json number,title,state,author,body,headRefName,baseRefName,createdAt,mergedAt,statusCheckRollup
```

## Read the diff

```bash
gh pr diff 15 --repo TheArchitectAI/roc-ai
```

For large diffs, pipe through `head` or `wc -l` first to know the size:
```bash
gh pr diff 15 --repo TheArchitectAI/roc-ai | wc -l
```

If > 500 lines, consider focusing on specific file paths:
```bash
gh pr view 15 --repo TheArchitectAI/roc-ai --json files | jq -r '.files[].path'
# Then read individual files at the PR head:
gh api repos/TheArchitectAI/roc-ai/contents/src/App.tsx?ref=feat/branch-name --jq '.content' | base64 -d
```

## Read PR comments + review threads

```bash
gh pr view 15 --repo TheArchitectAI/roc-ai --json comments,reviews
gh api repos/TheArchitectAI/roc-ai/pulls/15/comments  # line-level comments
```

## List recent PRs

```bash
# All PRs across a repo (limit 30, sort by updated)
gh pr list --repo TheArchitectAI/roc-ai --state all --limit 30 \
  --json number,title,state,author,createdAt,mergedAt,headRefName

# Just open PRs
gh pr list --repo TheArchitectAI/mortgagearchitect-ai --state open --limit 20

# PRs by a specific author (e.g., paperclip agent commits)
gh search prs --owner TheArchitectAI --author "@me" --limit 20
```

## Check CI status

```bash
gh pr checks 15 --repo TheArchitectAI/roc-ai
```

Returns pass/fail/pending per check name.

## When to use this skill

- CEO reviewing a PR for approval (e.g., ROC-9 PR #15 review)
- CTO auditing recent merges for governance compliance (Codex scaffold trace? Gemini audit attached?)
- Researcher cross-referencing what's been shipped vs what memory claims
- Identifying author identity on commits (human vs Paperclip vs Claude)

## When NOT to use this skill

- Posting comments / approving / merging — this is read-only. Mutations escalate to the human (Ivan) or to a write-capable surface
- Reading code on default branches that aren't behind a PR — use raw repo file reads instead
- Searching code across the entire org — use `gh search code` separately

## Safety constraints

- **Read-only.** Never `gh pr review`, `gh pr merge`, `gh pr comment` from this skill.
- **No token rotation.** Don't run `gh auth refresh` or `gh auth login`.
- **No NPI in output.** If a PR body contains borrower names (rare, but possible in a roc-ai loan-detail screen test fixture), redact in your comments.

## Identifying commit author identity

Useful for audits — distinguish who did what:

| Pattern in commit message | Author |
|---|---|
| `Co-Authored-By: Ivan Duarte` or `dwizy` | Human (Ivan) |
| `Co-Authored-By: Claude Opus` or `Co-Authored-By: Claude <noreply@anthropic.com>` | Claude Code (Trinity/Opus session) |
| `Co-Authored-By: Codex` | Codex scaffold |
| `Co-Authored-By: Gemini` | Gemini audit/edit |
| `Co-Authored-By: Paperclip <noreply@paperclip.ing>` | Paperclip agent |

## Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| `gh: command not found` | gh CLI not in PATH on this host | Escalate to CTO |
| `HTTP 401` from gh | Token expired | Don't retry; escalate to CTO |
| `HTTP 403` on private repo | Authenticated user lacks access | Confirm repo visibility; some repos are private |
| Rate-limited | Too many calls in short window | Wait 60s, retry; if persistent, batch reads with `gh api graphql` |

## Reference

- AA repo ledger: `[[reference_aa_repo_ledger_2026-05-17]]`
- Multi-LLM governance (when to spot the Codex/Gemini co-author footers): `[[feedback_multi_llm_governance]]`
