---
name: review-memory
description: "Distill PR review comments into reusable lessons and save them to Claude Code memory for automatic recall in future sessions."
license: MIT
allowed-tools: "Bash, Read, Write, AskUserQuestion"
---

> **Note:** This skill is designed for **Claude Code**. The memory written here is surfaced automatically by the Claude Code harness as context in future sessions. Other agents will not benefit from the memory step.

You are an expert at distilling PR review feedback into generalized lessons that prevent repeat mistakes.

---

## Step 1 — Resolve the Target PR

Determine which PR to process:

- **`/review-memory <PR number or URL>`** — use that PR directly
- **`/review-memory` (no args)** — detect the PR for the current branch:
  ```bash
  git branch --show-current
  gh pr list --head <branch> --json number,url,state
  ```
  If no PR is found, use `AskUserQuestion` to ask which PR to process.
- **Pasted comment text** — skip Steps 2–3, treat the pasted text as the sole raw comment.

Resolve `{owner}/{repo}` from the PR URL, or fall back to:
```bash
gh repo view --json nameWithOwner --jq .nameWithOwner
```

---

## Step 2 — Fetch Review Comments

Run these in parallel:

```bash
# Inline review comments (carry body / path / line / diff_hunk)
gh api repos/{owner}/{repo}/pulls/{n}/comments --paginate

# Review-level summaries (body + state: APPROVED / CHANGES_REQUESTED / COMMENTED)
gh pr view {n} --json reviews --repo {owner}/{repo}
```

Also resolve your own login for filtering:
```bash
gh api user --jq .login
```

---

## Step 3 — Filter

Keep only comments that meet **all** of these criteria:

- `user.login` ≠ your own login (exclude your replies)
- Not a pure approval (`state == "APPROVED"` with empty `body`)
- `body` is non-empty and not just "LGTM", "Looks good", ":+1:", "👍", or similar
- Contains a substantive suggestion, critique, or explanation

Discard the rest silently.

If **no comments pass the filter**, report that and stop.

---

## Step 4 — Distill into Generalized Lessons

For each filtered comment, produce a draft lesson. Apply these rules:

**Keep** (generalizable patterns):
- Recurring code style or architectural preferences
- Misuse of a language feature, API, or tool that could happen again
- Missing error handling, edge case, or test coverage the reviewer expects
- Communication or documentation norms

**Discard** (one-off nits):
- Renaming a specific variable/function in a specific file
- Typo fixes with no broader implication
- Formatting changes already handled by the formatter

Each lesson draft must include:
- A short slug (kebab-case, ≤ 40 chars) for the filename
- A one-line `description` (used by the harness to decide relevance at recall time — make it specific enough to trigger on related work, not so broad it fires on everything)
- The lesson body
- `**Why:**` — what the reviewer's concern was (quote or paraphrase the original comment with `PR #{n}, {path}`)
- `**How to apply:**` — concrete action to take in future code

---

## Step 5 — Confirm with the User

Present the draft lessons via `AskUserQuestion` with `multiSelect: true`:

- List each lesson's slug + description so the user can identify them
- Allow deselecting one-off nits that slipped through
- Allow skipping all if none are worth saving

If the user selects nothing, stop cleanly.

---

## Step 6 — Write to Claude Code Memory

For each selected lesson, write a memory file following the Claude Code memory protocol.

**Determine the memory directory** from the `<system-reminder>` context (the harness injects the path at session start — look for the Memory section describing `~/.claude/projects/<encoded>/memory/`).

**Check for duplicates first**: read `MEMORY.md` in the memory directory. If an existing entry covers the same lesson, update that file rather than creating a new one.

For each new lesson, create `<slug>.md`:

```markdown
---
name: <slug>
description: <one-line summary — used to decide relevance during recall>
metadata:
  type: feedback
---

<lesson body>

**Why:** <reviewer's concern; cite PR #n and file path>

**How to apply:** <concrete action for future code>
```

Then append one line to `MEMORY.md`:

```
- [<Title>](<slug>.md) — <description>
```

---

## Edge Cases

- **PR not found / access denied**: report the error and stop. No partial writes.
- **All comments filtered out**: "No actionable review comments found in PR #{n}." Stop.
- **User selects nothing**: "No lessons saved." Stop.
- **Duplicate detection**: prefer updating an existing memory over creating a near-duplicate; when in doubt, ask the user which to do via `AskUserQuestion`.
