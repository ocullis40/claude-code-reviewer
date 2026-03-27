# Code Reviewer — Design Spec

## Overview

A Claude Code custom slash command (`/review`) that reviews staged and unstaged git changes. It groups findings by functional area, labels them by severity, and prints a summary to the terminal. Conversational follow-up is handled naturally within the same Claude Code session.

## Goals

- Provide a fast, consistent code review on every set of changes
- Group findings by functional area so the developer sees the big picture
- Label findings by severity so the developer knows what to focus on
- Work on any project regardless of stack
- Replace the current "dispatch code review subagent" pattern with a simpler, reusable command

## How It Works

1. **Get context** — the prompt instructs Claude to gather:
   - `git diff` (unstaged) and `git diff --cached` (staged) for all pending changes. Untracked files (not yet `git add`ed) are excluded. Newly created files that have been staged are included via `git diff --cached`.
   - `git log --oneline -5` and the current branch name to understand recent intent.
   - The full list of changed files as a set to understand scope.

### Phase 1: Big Picture Review

2. **Assess intent and scope** — before reviewing individual lines, Claude determines what the change is trying to accomplish based on commit messages, branch name, and the set of files touched. It then considers:
   - Does the change accomplish its apparent goal?
   - **What's missing?** — are there changes you'd expect to see but don't? (e.g. a data model change without updated validation, a new feature without tests, a renamed function without updated callers)
   - Are there unintended side effects or scope creep?
3. **Print big picture summary** — a brief assessment of intent, completeness, and any "missing changes" concerns.

### Phase 2: Detailed Review

4. **Analyze and group** — Claude reads the diff and groups changes by functional area based on what the code *does*, not where the file lives. Functional areas include:
   - **Data flow** — changes to how data is stored, queried, or transformed (models, migrations, queries, schemas)
   - **Request handling** — changes to API endpoints, routing, middleware, request/response logic
   - **Business logic** — changes to core domain rules, validation, calculations
   - **User interface** — changes to components, layouts, styling, client-side behavior
   - **Configuration** — changes to env files, build config, dependencies, tooling
   - **Tests** — changes to test files (unless tightly coupled to one of the above areas)
5. **Assign severity** — each finding is labeled:
   - **Critical** — must fix before committing (security vulnerabilities, data loss risks, broken logic)
   - **Important** — should fix (missing error handling, pattern inconsistencies, edge cases)
   - **Suggestion** — nice to have (readability improvements, minor refactors)
6. **Print detailed summary** — formatted output with functional area headings, one-liner findings with severity and `file:line` references (line numbers are derived from diff hunk headers and may be approximate). Areas with no issues get a brief "looks good" note to confirm they were reviewed.

### Edge Cases

- **Empty diff** — if both `git diff` and `git diff --cached` return empty, print "No changes to review." and exit.
- **Large diffs** — the prompt instructs Claude to first check diff sizes by running `git diff --cached | wc -l` and `git diff | wc -l`. If the combined count exceeds roughly 1,500 lines, review only staged changes and print: "Unstaged changes were too large to include. Stage your changes and re-run, or review in smaller batches." This two-step approach avoids consuming context on a diff that will be discarded.

## Output Format

```
## Code Review

### Big Picture
This change adds email-based password reset to the auth system.
The implementation covers the request flow and token generation, but:
- **Missing:** No expiration check on reset tokens — tokens are generated but never validated for age.
- **Missing:** No rate limiting on the reset endpoint — could be abused for email spam.
- The scope looks appropriate otherwise. No unrelated changes.

### Data Flow (1 issue)
- **Critical:** Reset tokens are stored in plain text — `src/models/reset-token.ts:18`. Should be hashed before storage.

### Request Handling (2 issues)
- **Important:** Missing error handling for invalid email format — `src/api/reset/route.ts:15`
- **Suggestion:** Consider returning 200 even for unknown emails to prevent user enumeration — `src/api/reset/route.ts:22`

### Tests
- No issues found. Coverage looks appropriate.
```

## Installation

The slash command lives at `commands/review.md` in this repo. To use it in any project, add the repo path to your Claude Code settings under `projects` > `commandPaths`:

```json
{
  "projects": {
    "/path/to/your-project": {
      "commandPaths": ["/path/to/claude-code-reviewer/commands"]
    }
  }
}
```

Alternatively, symlink `commands/review.md` into your project's `.claude/commands/` directory.

## Usage

- Type `/review` in any Claude Code session to run the review
- To dig deeper on any finding, continue the conversation (e.g. "tell me more about the auth issue on line 42") — conversational follow-up uses normal Claude Code capabilities including reading full files
- Integrates with the Stride workflow by replacing the "dispatch code review subagent" step — same checkpoint, simpler mechanism

## What's NOT Included (v1)

- No linting or test execution
- No reading full files in the initial review (follow-up conversation can read files normally)
- No CI integration
- No configuration options (severity thresholds, ignore patterns)
- No persistent review history

These are all candidates for future versions if needed.

## Technical Details

The slash command is a single markdown file (`commands/review.md`) that contains a prompt template. When invoked via `/review`, Claude Code loads it into the current session. The prompt instructs Claude to:

1. Gather context: git diff (staged + unstaged), recent commit log, branch name
2. Phase 1: Assess intent, completeness, and missing changes
3. Phase 2: Group findings by functional area with severity labels
4. Print combined output (big picture first, then details)

No additional dependencies, no build step, no runtime. It's a prompt file in a repo.
