# claude-code-reviewer

A Claude Code `/review` slash command that performs two-phase code review on your git changes.

## What it does

1. **Big Picture** — Assesses the intent of your changes, checks for completeness, and flags missing pieces (e.g. "you changed the data model but didn't update validation").
2. **Detailed Review** — Groups findings by functional area (Data Flow, Request Handling, Business Logic, etc.) with severity labels (Critical, Important, Suggestion).

## Installation

### Option A: Install as a global skill (recommended)

Copy the command into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/code-review
cp /path/to/claude-code-reviewer/commands/diff-review.md ~/.claude/skills/code-review/SKILL.md
```

This makes `/code-review` available in all projects.

### Option B: Install per-project

Symlink or copy into a specific project's commands directory:

```bash
mkdir -p .claude/commands
ln -s /path/to/claude-code-reviewer/commands/diff-review.md .claude/commands/code-review.md
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
