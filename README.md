# update-pr

A Claude Code skill for incorporating PR review feedback into local files and generating a response document.

## What It Does

When invoked via `/update-pr`, this skill:

1. **Fetches** all PR comments from GitHub (top-level reviews and inline code comments)
2. **Accepts** supplementary feedback from any text source (review transcripts, emails, Slack threads)
3. **Categorizes** each piece of feedback (actionable, deferred, future work, retracted, non-actionable)
4. **Applies** actionable changes to local files within a confirmed scope
5. **Generates** `pr-comment-response.md` — a chronological reference document for posting responses back to the PR

The skill stops before committing. You decide when and how to commit.

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
/update-pr
```

The skill will ask for the PR number (if not inferrable from the branch) and confirm which files are in scope before making changes.

## Requirements

- `gh` CLI authenticated with access to the target repository
- Claude Code with skill loading enabled
