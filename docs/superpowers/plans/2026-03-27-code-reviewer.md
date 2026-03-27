# Code Reviewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code `/review` slash command that performs two-phase code review (big picture + detailed) on git changes.

**Architecture:** A single markdown prompt file (`commands/review.md`) that Claude Code loads when the user types `/review`. The prompt instructs Claude to gather git context, assess intent and completeness, then produce grouped findings with severity labels. No runtime dependencies.

**Tech Stack:** Markdown (Claude Code custom command format), Git CLI

---

## File Structure

```
claude-code-reviewer/
  commands/
    review.md          # The slash command prompt template (the entire product)
  README.md            # Installation and usage instructions
  docs/
    superpowers/
      specs/            # (already exists)
      plans/            # (this file)
```

---

### Task 1: Create the slash command prompt — context gathering

**Files:**
- Create: `commands/review.md`

This task creates the prompt file with the context-gathering instructions. The prompt will be built incrementally across tasks 1-4.

- [ ] **Step 1: Create the commands directory and start the prompt file**

```markdown
# /review — Code Review

You are a code reviewer. Review the current git changes using a two-phase approach: big picture first, then detailed findings.

## Step 1: Gather Context

**Important:** Only review tracked changes from `git diff` and `git diff --cached`. Do not review untracked files. Do not run `git status` to find additional files to review.

Run these commands to understand what changed and why:

1. Check diff sizes first to handle large diffs:
   - Run: `git diff --cached | wc -l` and `git diff | wc -l`
   - If combined count exceeds 1,500 lines, skip unstaged changes. Note this in your output: "Unstaged changes were too large to include. Stage your changes and re-run, or review in smaller batches."

2. If no diff exists (both return 0), print "No changes to review." and stop.

3. Get the diffs:
   - Run: `git diff --cached` (staged changes — review these first)
   - Run: `git diff` (unstaged changes — skip if too large per above)

4. Get intent context:
   - Run: `git log --oneline -5` (recent commits)
   - Run: `git branch --show-current` (current branch name)
   - Run: `git diff --cached --name-only` and `git diff --name-only` (changed file list)
```

- [ ] **Step 2: Verify the file exists at the correct path**

Run: `cat commands/review.md | head -5`
Expected: Shows the first lines of the prompt file.

- [ ] **Step 3: Commit**

```bash
git add commands/review.md
git commit -m "feat: add review command — context gathering step"
```

---

### Task 2: Add Phase 1 — Big Picture Review

**Files:**
- Modify: `commands/review.md`

Append the big picture review instructions to the prompt.

- [ ] **Step 1: Add Phase 1 instructions to the prompt**

Append to `commands/review.md`:

```markdown
## Step 2: Big Picture Review (Phase 1)

Before reviewing individual lines, assess the change as a whole.

Based on the branch name, recent commit messages, and the set of files changed, determine:

1. **Intent** — What is this change trying to accomplish? State it in one sentence.

2. **Completeness** — Does the change accomplish its apparent goal? Look for:
   - A data model change without updated validation or API handling
   - A new feature without tests
   - A renamed or moved function without updated callers or imports
   - A new API endpoint without error handling
   - A UI change without corresponding state management updates
   - Changes that touch multiple areas but miss one (e.g. updated the service layer but not the controller)

3. **Scope** — Are there unintended side effects or scope creep? Changes that don't relate to the apparent intent should be flagged.

Print your big picture assessment as:

### Big Picture
[One sentence describing the intent of the change.]
[Assessment of completeness. List any missing changes as:]
- **Missing:** [description of what's missing and why you'd expect it]
[Note on scope if there are concerns, otherwise: "The scope looks appropriate. No unrelated changes."]
```

- [ ] **Step 2: Verify the addition**

Run: `grep -c "Big Picture" commands/review.md`
Expected: At least 2 matches (heading + output format).

- [ ] **Step 3: Commit**

```bash
git add commands/review.md
git commit -m "feat: add Phase 1 big picture review instructions"
```

---

### Task 3: Add Phase 2 — Detailed Review

**Files:**
- Modify: `commands/review.md`

Append the detailed review instructions to the prompt.

- [ ] **Step 1: Add Phase 2 instructions to the prompt**

Append to `commands/review.md`:

```markdown
## Step 3: Detailed Review (Phase 2)

Now review the diff line by line. Group your findings by **functional area** based on what the code does, not where the file lives.

### Functional Areas

Assign each change to the most appropriate area:

- **Data Flow** — changes to how data is stored, queried, or transformed (models, migrations, queries, schemas, serialization)
- **Request Handling** — changes to API endpoints, routing, middleware, request/response logic, auth flows
- **Business Logic** — changes to core domain rules, validation, calculations, algorithms
- **User Interface** — changes to components, layouts, styling, client-side behavior, state management
- **Configuration** — changes to env files, build config, dependencies, tooling, CI/CD
- **Tests** — changes to test files, unless they clearly belong to a single area above (in which case group them with that area)

A single file may contain changes that span multiple areas. Group by what the change does, not the file it's in.

### Severity Levels

Label each finding:

- **Critical** — must fix before committing. Security vulnerabilities, data loss risks, broken logic, crashes.
- **Important** — should fix. Missing error handling, pattern inconsistencies, edge cases, potential bugs.
- **Suggestion** — nice to have. Readability improvements, minor refactors, style nits.

### Output Format

For each functional area that has changes, print:

### [Area Name] ([N] issues)
- **[Severity]:** [One-line description] — `file/path:line`

If an area has no issues:

### [Area Name]
- No issues found. [Brief note on what was reviewed.]

Line numbers come from diff hunk headers and may be approximate. Always include them when possible.
```

- [ ] **Step 2: Verify the addition**

Run: `grep -c "Severity" commands/review.md`
Expected: At least 4 matches (Critical, Important, Suggestion, heading).

- [ ] **Step 3: Commit**

```bash
git add commands/review.md
git commit -m "feat: add Phase 2 detailed review with functional grouping and severity"
```

---

### Task 4: Add output wrapper and final formatting

**Files:**
- Modify: `commands/review.md`

Add the final output wrapper that ties both phases together.

- [ ] **Step 1: Add output wrapper to the prompt**

Append to `commands/review.md`:

```markdown
## Output Structure

Combine both phases into a single output. Print:

## Code Review

[Phase 1: Big Picture section]

[Phase 2: Each functional area section]

---

## Important Guidelines

- Be specific. Every finding must reference a file and line number when possible.
- Be concise. One line per finding. Save details for the conversation if the user asks.
- Don't repeat the diff. The user can see it — tell them what's wrong with it.
- Don't praise the code. Focus on issues and missing pieces. "No issues found" is sufficient for clean areas.
- If there are no issues at all, say so briefly. Don't manufacture findings.
```

- [ ] **Step 2: Read through the complete prompt file to verify it's coherent**

Run: `cat commands/review.md`
Verify: The file reads as a coherent set of instructions from top to bottom, with no duplicated sections or contradictions.

- [ ] **Step 3: Commit**

```bash
git add commands/review.md
git commit -m "feat: add output wrapper and review guidelines"
```

---

### Task 5: Create README with installation instructions

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write the README**

```markdown
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
```

- [ ] **Step 2: Verify**

Run: `cat README.md | head -3`
Expected: Shows the title and description.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README with installation and usage instructions"
```

---

### Task 6: Test the command in a real project

**Files:**
- No files created or modified. This is a manual verification task.

- [ ] **Step 1: Install the command in the individual_learning project**

Add the command path to Claude Code settings, or symlink:

```bash
mkdir -p /Users/olivercullis/individual_learning/.claude/commands
ln -s /Users/olivercullis/claude-code-reviewer/commands/review.md /Users/olivercullis/individual_learning/.claude/commands/review.md
```

- [ ] **Step 2: Make a small test change in the individual_learning project**

Make a trivial edit to any file (e.g. add a comment) so there's a diff to review.

- [ ] **Step 3: Run `/review` in a Claude Code session**

Type `/review` and verify:
- It runs the git commands
- It produces a Big Picture section
- It produces detailed findings grouped by functional area
- Severity labels are present
- File:line references are included

- [ ] **Step 4: Test the empty diff case**

Revert the test change and run `/review` again.
Expected: "No changes to review."

- [ ] **Step 5: Test the large diff case**

Generate a large unstaged diff (e.g. `for i in $(seq 1 200); do echo "// line $i" >> src/tmp-large.ts; done`) and stage a small change separately. Run `/review`.
Expected: Output includes "Unstaged changes were too large to include." and reviews only the staged change. Clean up the temp file after.

- [ ] **Step 6: Push the repo**

```bash
cd /Users/olivercullis/claude-code-reviewer
git push -u origin main
```
