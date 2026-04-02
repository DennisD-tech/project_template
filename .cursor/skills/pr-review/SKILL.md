---
name: pr-review
description: >
  Process GitHub PR review comments: categorize and respond comment-by-comment.
  Trigger when the user mentions "PR review", "review message", "review 意见",
  "review 留言", "code review", "审查意见", "PR 评论", "review comments",
  "处理 review", or "review 反馈".
  Also trigger when the user pastes review text, provides a PR URL and asks to handle
  the review, or says "help me handle these review comments" / "处理这个 PR 里的 review".
---

# PR Review Handling Skill

This skill helps you systematically handle GitHub PR review comments by:
1. Categorizing priorities.
2. Skipping low-value suggestions by default (with explicit reasons).
3. Understanding root causes before making code changes.
4. Responding to the reviewer’s comments in a way that prevents repeat feedback.

## Prerequisites — Read Before You Touch Anything

1. Read the repo’s `AI_GUIDELINES.md` first (it is the source of truth for doc split,
   workflow order, code conventions, and security expectations).
2. If you plan to run or add tests, treat `TESTING.md` as authoritative for:
   - install prerequisites (if any)
   - standard test/typecheck/build commands
   - existing test directory structure and naming conventions
   Detailed strategies live under `docs/06-quality/` (when present).
3. When you need to update temporary/repo planning state, follow the minimal workflow
   described in `AI_GUIDELINES.md`: `WIP.md` → `TODO.md` → `PLAN.md`.

## Core Principles

1. Understand first, change later — do not modify code immediately on seeing a single comment. Read all comments to identify the reviewer’s overall concerns, then plan a modification strategy.
2. Fix the root cause — if a reviewer points at a symptom, find what causes it upstream.
3. Language-only edits are lowest priority — skip pure doc/comment wording suggestions unless they contain factual errors or are seriously misleading.
4. Respect without blind follow-through — treat reasonable suggestions seriously, and for questionable ones explain why you skip.

## Phase 1: Acquire Review Comments

Choose the acquisition mode based on what the user provides:

### Mode A: User provides a PR URL

1. Use the `gh` CLI to fetch PR data:
   - `gh pr view <PR_NUMBER> --json number,title,body,reviews,comments`
   - `gh api repos/<OWNER>/<REPO>/pulls/<PR_NUMBER>/comments`
   - `gh api repos/<OWNER>/<REPO>/pulls/<PR_NUMBER>/reviews`
2. Parse all review comments (including inline review comments and summary reviews).
3. Associate each comment with the target file and code location (line when available).
4. Prepare an internal list including the minimum identifiers needed to reply (e.g. `comment_id`).

### Mode B: User pastes review text

1. Ingest the pasted review text as-is.
2. If the format is unclear, ask the user to clarify at least:
   - Which file(s) the comments refer to
   - The relevant code snippet/line range (or the PR diff context)
3. If needed, ask for the PR URL to retrieve exact inline positions.

### Verification Step (Required)

After you collect comments, output a concise “comment summary list” and ask the user to confirm there is no omission before proceeding.

## Phase 2: Categorize & Prioritize

Categorize each review comment into one of four priorities:

| Priority | Tag | Meaning | Handling |
|---|---|---|---|
| P0 | Must Fix | Logic error, security vulnerability, data-loss risk, API contract violation | Fix immediately |
| P1 | Should Fix | Performance risk, maintainability issue, project-norm mismatch, potential bug | Usually fix unless there is a strong reason |
| P2 | Can Improve | Style preference, nicer structure, minor naming improvements | Discuss before applying |
| SKIP | Skip | Pure language polishing or subjective preferences without clear superiority | Skip and explain why |

### Output Format

Use the following table format:

```
### Review Comment Categorization

| # | Comment Summary | File | Priority | Reason |
|---|---|---|---|---|
| 1 | Missing null check | src/api/user.ts:42 | P0 Must Fix | Production crash risk |
| 2 | Use Map instead of Object | src/utils/cache.ts:15 | P1 Should Fix | Better semantics & performance |
| 3 | Rename `d` to `data` | src/api/user.ts:50 | P2 Can Improve | Small readability gain |
| 4 | Typo in comment | src/api/user.ts:38 | SKIP | Pure comment wording |
```

For every `SKIP` item, you must provide a one-sentence reason.

After showing the categorization table, wait for the user to confirm or adjust:
- Promote `SKIP` → `P1/P0`
- Demote `P1` → `SKIP`
- Provide missing context
- If the user indicates disagreement on a specific comment’s priority, ask a clarifying question and wait for the user’s decision.

## Phase 3: Deep Analysis (No Code Changes Yet)

Once the user confirms categorization, do deep analysis for all `P0` and `P1` items before editing code.

For each item:
1. Read full context — not only the indicated line; read the whole function/module to understand intent.
2. Trace root cause — if a bug is claimed, find the underlying reason (it may not be the exact line called out).
3. Evaluate impact scope — identify other places that share the same problem pattern.
4. Choose a best repair approach — prefer correctness and maintainability over “smallest diff”.

### Output Repair Plan

Output a plan like:

```
### Repair Plan

#### #1 Missing null check (P0)

**Root cause:** The return value from `fetchUser` is not handled as nullable across the call chain; the same issue exists at lines 67 and 89.

**Proposed fix:** Centralize handling at `fetchUser` and return a `Result` type so callers must handle error cases explicitly.

**Impact scope:** Update 3 callers (user.ts, profile.ts, admin.ts).
```

Wait for user confirmation before making changes.

## Phase 4: Execute Fixes

Apply changes after the user approves the repair plan, in priority order:
1. All P0 items
2. Then all P1 items
3. Only if the user agrees: P2 items
4. For all SKIP items: reply to the reviewer comment with the reason for not applying

### Replying to SKIP Items (Required)

Replying is important because the AI reviewer may repeat the same issue in later reviews if you don’t explain why it was skipped.

Use `gh api` to reply (example endpoint):
```bash
gh api repos/<OWNER>/<REPO>/pulls/comments/<COMMENT_ID>/replies \
  -f body="<Reply Content>"
```

### Change Discipline

Follow the project’s existing style and boundaries:
- Do not modify unrelated code “just because”.
- Do not do refactors unless the reviewer comment requires it or the root cause makes it necessary.
- After code changes, verify with tests and type checks as defined by `TESTING.md`.

## Phase 5: Summarize & Close the Loop

After completing all changes:

1. Produce a final summary:

```
### PR Review Handling Summary

**Handled:**
| # | Comment | Priority | Action | Files |
|---|---|---|---|---|
| 1 | Missing null check | P0 | Refactor to Result | user.ts, profile.ts |
| 2 | Map vs Object | P1 | Replaced | cache.ts |

**Skipped:**
| # | Comment | Reason |
|---|---|---|
| 4 | Typo in comment | Pure comment wording |

**Reviewer replies (draft):**
> <Short, professional replies for each handled/skipped group>
```

2. Documentation & workflow updates (follow `AI_GUIDELINES.md` “minimal workflow”):
   - Update `WIP.md` / `TODO.md` when maintenance was driven by PR review tasks.
   - Record key decisions in `DEV_NOTE.md` (decision basis + final conclusion).
   - If maintenance changes contracts/interfaces, update `docs/03-architecture/`.
   - Run checks as the project defines in `TESTING.md` (format/typecheck/build/tests as applicable).

## Communication & Safety Notes

- User-facing dialogue: default to Chinese with the user (unless the user explicitly prefers English).
- Code/commit messages: follow the project’s existing language conventions.
- Security: do not read or copy secrets (e.g. `.env` or production credentials). If credentials are required for verification, instruct the user to run commands locally and provide only sanitized output.
- If there are many comments, process them in batches: finish Phase 2 first, then execute P0/P1 in groups to keep decisions auditable.

