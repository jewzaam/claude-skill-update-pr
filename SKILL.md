---
name: update-pr
description: Incorporate feedback into a pull request. Fetches PR review comments from GitHub, accepts additional feedback from any text source (review transcripts, emails, Slack threads), walks through each comment one at a time with before/after context, and generates an actionable summary document with draft replies and a re-request review checklist. This skill is invoked explicitly by the user via /update-pr -- do not trigger it automatically.
---

# Update PR with Review Feedback

This skill performs NO write operations on GitHub. No comments, no pushes, no PR updates, no review submissions, no thread resolution. All GitHub interactions are read-only fetches. The user posts replies and pushes commits themselves.

## Overview

The workflow has three phases:

1. **Fetch** — collect all PR comments and supplementary feedback
2. **Review** — walk through each comment one at a time with the user, showing before/after context
3. **Apply & Summarize** — apply accepted changes (GitHub suggestions first, then local edits), generate an actionable summary document

## Before You Start

The user must provide the PR number — either as an argument (e.g., `/update-pr 1234`) or in the conversation. If they didn't provide one, ask before doing anything else.

Once you have the PR number, derive the repo owner/name from the git remote (do not run `gh repo view` — it's unnecessary). Then confirm scope with the user before fetching comments:

1. **Scope** — check which files the PR touches (`gh pr diff <PR> --name-only`) to suggest a default scope.
2. **Supplementary feedback** — note whether the user mentioned any non-GitHub feedback sources (transcripts, pasted text, files).

Present this for confirmation:
- "PR #X in owner/repo, targeting these files: [list]. Any files out of scope for changes? Any supplementary feedback to incorporate?"

Proceed only after the user confirms or adjusts. Do not report comment counts here — that information isn't available until after the Phase 1 fetch. The resolution filtering report comes at the end of Phase 1, after all comments are fetched, grouped, and filtered.

## Phase 1: Fetch Feedback

### GitHub PR Comments

Three API calls are needed. They are independent — run all three in parallel to minimize latency.

**Top-level reviews** (approve/request-changes/comment left via GitHub's Review button):

```bash
gh pr view <PR> --json comments,reviews,title,body
```

Returns:
- `reviews[]` — `.author.login`, `.body`, `.state`, `.submittedAt`
- `comments[]` — general PR conversation comments (not inline)

**Inline code review comments** (attached to specific lines in the diff):

```bash
gh api repos/<owner>/<repo>/pulls/<PR>/comments
```

Each comment includes:
- `.user.login`, `.path`, `.original_line`, `.body`, `.created_at`
- `.in_reply_to_id` — parent comment ID if this is a reply in a thread

This endpoint can return large payloads (64KB+ for active PRs). If the tool harness persists the output to a file, read that file.

**Review thread resolution status** (which threads are resolved on GitHub):

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100) {
          nodes {
            isResolved
            comments(first: 1) {
              nodes { id databaseId }
            }
          }
        }
      }
    }
  }
' -f owner='<owner>' -f repo='<repo>' -F pr=<PR>
```

Each thread's `isResolved` flag maps to its first comment's `databaseId`, which corresponds to the `id` field from the REST inline comments endpoint. Build a set of resolved comment IDs from this data.

The `first: 100` limit covers most PRs. If the response includes `pageInfo.hasNextPage: true`, paginate to fetch all threads — missing a resolved thread means its comments will incorrectly appear as unresolved. Add `pageInfo { hasNextPage endCursor }` to the query and pass `after: <endCursor>` on subsequent requests.

**Data integrity:** Comment bodies — especially suggestion blocks — must never be truncated. Read every comment body in full. Suggestion blocks contain the exact proposed replacement text and must be compared character-by-character against the current file to determine if they've already been applied.

### Supplementary Feedback

If the user indicated supplementary feedback sources:
- **Review transcripts** — files containing notes from verbal reviews or recorded walkthroughs
- **Pasted text** — feedback from email, Slack, or other channels
- **Referenced files** — files in the repo containing feedback

Read and extract feedback items from these sources.

### Group Reply Chains

Before reviewing, group threaded comments using `in_reply_to_id`. PR comments often have reply chains where feedback evolves — a reviewer may clarify, soften, or retract their original point in a follow-up. When a thread exists:

- Read the full thread to understand the final position
- If a later reply retracts or supersedes the original feedback, treat the thread as a single unit reflecting the final state
- Attribute the thread to the original commenter unless a different reviewer's reply changes the nature of the feedback

### Cross-Check Comment Count

After grouping, verify that every fetched comment appears in exactly one group (standalone or threaded). The total number of individual comments across all groups must equal the total number of fetched comments. Flag any discrepancy before proceeding — a missing comment means a reviewer's feedback could be silently ignored. This check is about grouping completeness, not review count — the number of groups to review will be smaller after resolved threads are filtered in the next step.

If the PR has no GitHub comments and no supplementary feedback was provided, report this to the user and stop — there is nothing to review.

### Filter Out Resolved Comments

Using the resolution status from the GraphQL query, remove all comments belonging to resolved threads. Resolved threads are always skipped — this is not optional and should never be presented as a choice to the user. Only unresolved threads and supplementary feedback items proceed to Phase 2.

Report the filtering result before starting Phase 2: "N comments fetched, M already resolved on GitHub, K unresolved comments to review." If all comments are resolved and there's no supplementary feedback, report this and stop — there is nothing to review.

## Phase 2: Review Each Comment

Walk through each **unresolved** comment (or consolidated thread) one at a time. This is a single pass — categorization and the user's decision happen together. Never batch multiple items into one question. Never present resolved comments — they were filtered out in Phase 1.

### Before/After Context Is Mandatory

For every comment that references a code or text change, read the actual file content at the referenced location. The user cannot evaluate a change without seeing the concrete diff. Do not describe changes in prose when you can show the text.

**Line mapping caveat:** After force-pushes, the `original_line` from the API refers to a line in the original diff, not the current file. Use the comment body and surrounding context (quoted code, suggestion blocks) to locate the correct position in the current file rather than trusting the line number blindly. Search for the quoted text in the file to find the actual location.

### Determine Comment Type

Each comment falls into one of two types, which determines the decision flow:

**Change comments** — the reviewer proposes a specific modification (suggestion block, requested edit, or rewrite). These need a code decision:

- **Accept** — apply the change as proposed
- **Reject** — don't apply; user provides reasoning for their reply
- **Already applied** — the change was already made in a prior commit

The user can always select "Other" to provide free-form text — covering cases like applying with modifications, applying an alternative change, or any nuanced response.

**Discussion comments** — questions, observations, or feedback that don't propose a specific change. These need a reply decision:

- **Answer** — user provides the answer to include in their draft reply
- **Acknowledge** — user agrees or notes the feedback
- **Defer** — valid but out of scope; user provides reasoning
- **No action needed** — approval, praise, or already-answered questions

### Mandatory Question Format

Use `AskUserQuestion` for each item. The question text must follow this structure — do not deviate:

For **change comments**, include before/after context:

```
**[reviewer_handle] on [file]:[line]:** [condensed feedback]

> [reviewer's quoted comment, or suggestion block content]

- Before: [exact current text from the file, quoted]
- After: [exact proposed replacement text, quoted]

What would you like to do?
```

Options: Accept | Reject | Already applied

For **discussion comments**:

```
**[reviewer_handle] on [file]:[line] (or "top-level review"):** [condensed feedback]

> [reviewer's quoted comment]

How would you like to handle this?
```

Options: Answer | Acknowledge | Defer | No action needed

The user can always select "Other" to provide free-form text. Whatever the user says — option, notes, or free-form — becomes the basis for the draft reply in the summary document. Capture their words faithfully; this is what they'll adapt when posting on the PR.

### Reviewer References

When referring to reviewers in questions or the output document, use their handle directly (e.g., "jane requested..." not "she requested...") or use "they/them" as default neutral pronouns. Never infer gender from names.

### Already-Applied Verification

Before categorizing any comment as "already applied," diff the suggestion or proposed change against the current file content. If they don't match exactly, it's a change comment that needs a decision. Partial matches (e.g., the suggestion includes additional text beyond what was applied) are not "already applied."

## Phase 3: Apply Changes & Generate Summary

### Step 1: Separate GitHub Suggestions from Local Edits

After the review pass, separate accepted changes into two groups:

**GitHub-mergeable suggestions** — accepted items that have a GitHub `suggestion` block. Applying these through the GitHub UI creates a commit credited to the reviewer, which is valuable for team dynamics and PR history. If the user accepted a suggestion but added modifications via free-form text, apply the suggestion on GitHub first, then apply the user's revision locally after pulling — this preserves reviewer attribution for the base change while layering the user's modifications.

**Local edits** — everything else (items without suggestion blocks, alternatives, revisions on top of merged suggestions, and cleanup work).

### Step 2: GitHub Suggestions First

Present the GitHub-mergeable suggestions to the user via `AskUserQuestion`. Include the count and a checklist:

```
N suggestions can be applied via GitHub's "Apply suggestion" button for reviewer attribution.
Which would you like to handle on GitHub?
```

For each suggestion, show the reviewer, file, and the change. Options for each:
- **I'll apply on GitHub** — user will apply it in the PR UI
- **Apply locally instead** — skip GitHub attribution, apply as a local edit
- **Already applied** — user already merged it

After the user indicates which ones they'll apply on GitHub, instruct them:

1. Apply the suggestions on the GitHub PR page
2. Stash any uncommitted local changes (`git stash`)
3. Pull the new commits (`git pull`)
4. Restore local state (`git stash pop`)

Wait for the user to confirm they've completed this. Then verify each suggestion landed by reading the file and confirming the expected text is present. Report any that didn't land before proceeding.

### Step 3: Apply Local Edits

For each remaining accepted or modified item:
1. Edit the referenced files
2. Verify changes make sense in context (read surrounding code)

After all changes are applied, run the project's test command if discoverable (e.g., `make test`, `make test-unit`, or similar from Makefile targets). Report any test failures.

Changes are left uncommitted — the user controls staging, diffs, and commit messages.

### Step 4: Cross-Check — Every Comment Needs a Resolution Path

Before generating the summary, verify every fetched comment has a clear resolution:

| Resolution type | What happened | Reply needed? |
|---|---|---|
| Applied via GitHub suggestion | Reviewer gets commit attribution | No (GitHub auto-marks) |
| Applied locally | Code changed but reviewer has no signal | Yes — draft a reply noting the change |
| Rejected | No code change | Yes — draft a reply with user's reasoning |
| Discussion answered | No code change | Yes — draft the answer |
| Deferred | Out of scope | Yes — draft a reply with reasoning |
| Already applied | Changed in a prior commit | Yes — brief reply noting which commit |
| Resolved on GitHub | Thread already resolved before this run | No — already handled |
| No action needed | Approval, praise | No |

Flag any comment that lacks a resolution path. Every reviewer comment (except pure approvals) should map to either "resolved by code change" or "needs reply to draft."

### Step 5: Generate Summary Document

Write `PR-<number>-Comment-Summary.md` (e.g., `PR-1301-Comment-Summary.md`) in the repository root.

Use this structure:

```markdown
# PR <number> Comment Summary

PR: <title>
Generated: <date>

## Draft Replies

Replies to post on the PR to acknowledge reviewer feedback and address comment threads.

- [ ] **<reviewer_handle> on <file:line>** [<Decision>]
  > <condensed feedback>
  **Draft reply:** <reply text based on user's words from the review>

(repeat for each comment needing a reply)

## Re-Request Reviews

After posting replies and pushing changes, re-request review from these reviewers:

- [ ] **<reviewer_handle>** — <brief note on what was addressed for them>

(repeat for each reviewer who left actionable feedback)

## Applied via GitHub Suggestions

These were applied through GitHub's suggestion feature. Reviewer attribution is preserved.
No action needed — GitHub auto-resolves these threads.

- <reviewer_handle> on <file:line> — <summary>

## Applied Locally

These changes were applied to local files and will be included in the next push.

- <reviewer_handle> on <file:line> — <summary>

## Already Resolved on GitHub

N threads were already resolved before this review session. No action taken.

## No Action Needed

Approvals, praise, and informational comments that require no response.

- <reviewer_handle> — <summary>

<details>
<summary>Full Comment Log</summary>

### <timestamp> — <reviewer_handle> on <file:line> [<Decision>]

> <quoted feedback>

**Resolution:** <user's own words from the review, or explanation for non-actionable categories>

---

(repeat for every comment, in chronological order)

</details>
```

Format rules:

- **Checkboxes (`- [ ]`) only on items requiring user action** — draft replies to post and reviews to re-request. Informational items (already applied, no action needed) use plain `- ` bullets. If checking the box doesn't correspond to a discrete action the user performs, it is not a checkbox.
- **Chronological order in the full log** — entries ordered by timestamp so the user can correlate with the PR's comment timeline on GitHub
- **User's voice in draft replies** — resolutions reflect the user's own words from the interactive review. This is the text they'll adapt when posting on the PR, so preserve their intent and tone.
- **Full comment log is collapsible** — wrapped in `<details>` so it doesn't dominate the output. The action items sections above are the primary working surface; the log exists as an audit trail for completeness.

### After Generating the Summary

Report to the user:
- Total comments fetched
- Breakdown: already resolved on GitHub, applied via GitHub suggestion, applied locally, rejected, deferred, no action needed
- Count of draft replies to post
- Count of reviewers to re-request
- Any test results from the verification step
