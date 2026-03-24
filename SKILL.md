---
name: update-pr
description: Incorporate feedback into a pull request. Fetches PR review comments from GitHub, accepts additional feedback from any text source (review transcripts, emails, Slack threads), categorizes each piece of feedback, applies actionable changes to local files, and generates a chronological response document. This skill is invoked explicitly by the user via /update-pr -- do not trigger it automatically.
---

# Update PR with Review Feedback

This skill drives a structured workflow for incorporating feedback into a pull request. It handles feedback from GitHub PR comments and any supplementary text the user provides (review transcripts, meeting notes, pasted messages).

The workflow has three phases: **Fetch**, **Apply**, **Summarize**. Complete them in order.

## Before You Start

The user must provide the PR number — either as an argument to `/update-pr` (e.g., `/update-pr 1234`) or in the conversation. Do not guess or auto-detect the PR number. If the user didn't provide one, ask for it before doing anything else.

Once you have the PR number, gather remaining context and confirm with the user in a single checkpoint:

1. **Repo owner/name** — run `gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"'`.
2. **Scope** — check which files the PR touches (`gh pr diff <PR> --name-only`) to suggest a default scope.
3. **Supplementary feedback** — note whether the user mentioned any non-GitHub feedback sources.

Present this to the user for confirmation:
- "PR #X in owner/repo, targeting these files: [list]. Any files out of scope for changes? Any supplementary feedback to incorporate (transcripts, pasted text, etc.)?"

Proceed only after the user confirms or adjusts.

## Phase 1: Fetch Feedback

### GitHub PR Comments

Two API calls are needed because PR feedback lives in two places.

**Top-level reviews** (approve/request-changes/comment left via GitHub's Review button):

```bash
gh pr view <PR> --json comments,reviews,title,body
```

This returns:
- `reviews[]` — each has `.author.login`, `.body`, `.state` (APPROVED/COMMENTED/CHANGES_REQUESTED), `.submittedAt`
- `comments[]` — general PR conversation comments (not inline)

**Inline code review comments** (attached to specific lines in the diff):

```bash
gh api repos/<owner>/<repo>/pulls/<PR>/comments
```

Each comment includes:
- `.user.login` — reviewer
- `.path` — file path
- `.original_line` — line number in the diff
- `.body` — the comment text
- `.created_at` — timestamp
- `.in_reply_to_id` — parent comment ID if this is a reply in a thread

This endpoint can return large payloads (64KB+ for active PRs). If the tool harness persists the output to a file, read that file.

### Supplementary Feedback

If the user indicated supplementary feedback sources during the checkpoint:
- **Review transcripts** — files like `Review-transcript.md` containing notes from verbal reviews or recorded walkthroughs
- **Pasted text** — feedback copied from email, Slack, or other channels
- **Referenced files** — the user may point to files in the repo containing feedback

Read and extract actionable items from these sources.

### Build a Unified Timeline

Merge all fetched feedback into a single chronological list using the timestamps already retrieved. Sort GitHub comments by their timestamps (`.created_at` for inline, `.submittedAt` for reviews). Place supplementary feedback without timestamps at the end, noting the source.

### Resolve Comment Threads

Before categorizing, resolve comment threads. PR comments often have reply chains where feedback evolves — a reviewer may clarify, soften, or retract their original point in a follow-up reply. Use `in_reply_to_id` to group threaded comments together. When a thread exists:

- Read the full thread to understand the final position
- If a later reply retracts or supersedes the original feedback, treat the thread as a single unit reflecting the final state
- Attribute the thread to the original commenter unless a different reviewer's reply changes the nature of the feedback

## Phase 2: Apply Feedback

### Categorize Each Comment

For every piece of feedback (or resolved thread), assign exactly one category:

| Category | Meaning | Action |
|---|---|---|
| **Actionable** | Can be addressed in the current changeset within the confirmed scope | Apply the change |
| **Deferred** | Valid feedback but out of scope (e.g., references files not in this PR) | Note in response doc |
| **Future work** | Acknowledged by the commenter as follow-up, not blocking | Note in response doc |
| **Retracted** | Commenter withdrew the feedback in a later comment or thread reply | Note in response doc |
| **Non-actionable** | Approval, praise, questions already answered, or informational | Note in response doc |

### Present Categorization for Review

Before applying any changes, present the categorization to the user using grouped lists so they can verify and adjust:

```
**Actionable (N)**
- <author> on <file:line> — <summary>
- <author> on <file:line> — <summary>

**Deferred (N)**
- <author> on <file:line> — <summary> (<reason>)

**Future work (N)**
- <author> on <file:line> — <summary>

**Retracted (N)**
- <author> on <file:line> — <summary>

**Non-actionable (N)**
- <author> (<location>) — <summary>
```

This is a checkpoint — the user may recategorize items or adjust scope. Wait for confirmation before proceeding.

### Interactive Review of Actionable Items

The user is the PR author — they may disagree with feedback, want to modify the approach, or have context the reviewer didn't. Before applying any changes, walk through each **Actionable** item one at a time so the user can decide what to do.

For each actionable comment, present:
- The reviewer, location, and quoted feedback
- A suggested change (what you would do to address it)

Then use `AskUserQuestion` to get the user's decision. Provide these options:
- **Accept** — agree with the feedback, apply the suggested change
- **Reject** — disagree with the feedback
- **Modify** — partially agree, do something different

The user can also select "Other" to type any response they want. Whatever the user says — their chosen option, notes, or free-form text — becomes the basis for the Resolution in the response document. Capture what they say faithfully. If they reject or modify, their reasoning is what they'll later post on the PR, so accuracy matters.

After walking through all actionable items:
- Apply changes only for items the user accepted or modified
- For modified items, implement the user's stated approach, not the original reviewer suggestion
- Rejected items get no code changes but their reasoning is recorded for the response doc

### Apply Changes

For each accepted or modified item:
1. Edit the referenced files
2. Verify the changes make sense in context (read surrounding code)

After all changes are applied, run the project's test command if one is discoverable (e.g., `make test`, `make test-unit`, or similar from Makefile targets). Report any test failures — do not silently proceed past broken tests.

The user controls commits. Changes are applied to the working tree and left uncommitted so the user can review diffs, stage selectively, and commit with their preferred message and workflow.

## Phase 3: Response Document

Generate `pr-comment-response.md` in the repository root. The user posts PR responses themselves — this file is their reference for what was done and why.

### Format

Use this exact structure:

```markdown
# PR <number> Comment Responses

PR: <title>
Generated: <date>

## Comment Responses

### <timestamp> — <author> on <file:line> (or "top-level review") [<Decision>]

> <quoted feedback, condensed to essential point>

**Resolution:** <the user's own words from the interactive review, or brief explanation for non-actionable categories>

---

(repeat for each comment, in chronological order)

## Additional Changes

Changes made during this update that were not triggered by specific comments:

- <description of change>
```

Key format rules:
- **Chronological order** — entries ordered by timestamp so the user can correlate with the PR's comment timeline while scrolling through GitHub
- **Every comment gets an entry** — even non-actionable ones (approvals, praise) get a brief entry so the user knows nothing was missed
- **Decision tag in heading** — each entry's heading includes the decision in brackets. For actionable items: `[Accepted]`, `[Rejected]`, or `[Modified]`. For other items: `[Deferred]`, `[Future Work]`, `[Retracted]`, or `[Non-actionable]`
- **User's voice in resolutions** — for actionable items, the Resolution reflects the user's own words from the interactive review (accepted, rejected with reasoning, or modified approach). This is the text the user will adapt when posting responses on the PR, so preserve their intent and tone
- **Additional changes section** — any changes made during the update that weren't triggered by a specific comment (e.g., fixing something discovered while applying feedback)

### After Generating the Response Document

Report to the user:
- How many comments were fetched
- Breakdown by category (actionable, deferred, future work, retracted, non-actionable)
- Any test results from the verification step
