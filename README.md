# update-pr

A Claude Code skill for incorporating PR review feedback. Walks through each comment one at a time with before/after context, prioritizes GitHub suggestion attribution, and generates an actionable summary with draft replies and a re-request review checklist.

All GitHub interactions are read-only. The user posts replies and pushes commits themselves.

## What It Does

When invoked via `/update-pr <PR-number>`, this skill:

1. **Fetches** all PR comments from GitHub (top-level reviews and inline code comments)
2. **Accepts** supplementary feedback from any text source (review transcripts, emails, Slack threads)
3. **Reviews** each comment one at a time — shows before/after context, asks for your decision (accept, reject, revise, defer)
4. **Applies GitHub suggestions first** via the PR UI for reviewer attribution, then applies remaining changes locally
5. **Cross-checks** that every comment has a resolution path (code change or draft reply)
6. **Generates** `PR-<number>-Comment-Summary.md` with checkboxed action items: draft replies to post and reviewers to re-request

Changes are left uncommitted. You control staging, diffs, and commit messages.

## Installation

Clone or check out this repository into your Claude Code skills directory:

```bash
git clone <repo-url> ~/.claude/skills/update-pr
```

Or symlink if you prefer to keep the repo elsewhere:

```bash
ln -s /path/to/claude-skill-update-pr ~/.claude/skills/update-pr
```

## Usage

From a branch with an open PR:

```
/update-pr 1234
```

The PR number is required. The skill will confirm which files are in scope before making changes.

## Requirements

- `gh` CLI authenticated with access to the target repository
- Claude Code with skill loading enabled
