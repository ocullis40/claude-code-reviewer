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

1. **Get the diff** — the slash command prompt instructs Claude to run `git diff` (unstaged) and `git diff --cached` (staged) to capture all pending changes. Untracked files (not yet `git add`ed) are excluded. Newly created files that have been staged are included via `git diff --cached`.
2. **Analyze and group** — Claude reads the diff and groups changes by functional area (e.g. "Authentication", "API routes", "Database layer"). The prompt includes grouping guidance with examples: files under `src/app/api/` group as "API Routes", files under `src/components/` group as "UI Components", test files group as "Tests" unless they clearly belong to a single functional area, and related changes across files are grouped together when they serve the same purpose.
3. **Assign severity** — each finding is labeled:
   - **Critical** — must fix before committing (security vulnerabilities, data loss risks, broken logic)
   - **Important** — should fix (missing error handling, pattern inconsistencies, edge cases)
   - **Suggestion** — nice to have (readability improvements, minor refactors)
4. **Print summary** — formatted output with functional area headings, one-liner findings with severity and `file:line` references (line numbers are derived from diff hunk headers and may be approximate). Areas with no issues get a brief "looks good" note to confirm they were reviewed.

### Edge Cases

- **Empty diff** — if both `git diff` and `git diff --cached` return empty, print "No changes to review." and exit.
- **Large diffs** — the prompt instructs Claude to first check diff sizes by running `git diff --cached | wc -l` and `git diff | wc -l`. If the combined count exceeds roughly 1,500 lines, review only staged changes and print: "Unstaged changes were too large to include. Stage your changes and re-run, or review in smaller batches." This two-step approach avoids consuming context on a diff that will be discarded.

## Output Format

```
## Code Review Summary

### Authentication (2 issues)
- **Critical:** SQL injection risk in login query — `src/auth/login.ts:42`
- **Suggestion:** Consider extracting token validation to a shared util — `src/auth/login.ts:78`

### API Routes (1 issue)
- **Important:** Missing error handling for invalid topicId — `src/app/api/topics/route.ts:15`

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

1. Run the git diff commands via Bash
2. Analyze the output
3. Produce the grouped, severity-labeled summary

No additional dependencies, no build step, no runtime. It's a prompt file in a repo.
