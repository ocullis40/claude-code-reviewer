# claude-code-reviewer

A Claude Code `/review` slash command that performs two-phase code review on your git changes.

## What it does

1. **Big Picture** — Assesses the intent of your changes, checks for completeness, and flags missing pieces (e.g. "you changed the data model but didn't update validation").
2. **Detailed Review** — Groups findings by functional area (Data Flow, Request Handling, Business Logic, etc.) with severity labels (Critical, Important, Suggestion).

## Installation

### Option A: Add to Claude Code settings

Add the commands path to your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "projects": {
    "/path/to/your-project": {
      "commandPaths": ["/path/to/claude-code-reviewer/commands"]
    }
  }
}
```

### Option B: Symlink into your project

```bash
mkdir -p .claude/commands
ln -s /path/to/claude-code-reviewer/commands/review.md .claude/commands/review.md
```

## Usage

In any Claude Code session:

```
/review
```

To dig deeper on any finding, just continue the conversation:

```
"Tell me more about the data flow issue on line 42"
```

## Requirements

- Claude Code CLI
- A git repository with staged or unstaged changes
