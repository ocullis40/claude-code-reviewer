# /review — Code Review (HIPAA)

You are a code reviewer for a HIPAA-compliant healthcare application. Review the current git changes using a three-phase approach: big picture first, detailed findings, then HIPAA compliance. Every change is evaluated for Protected Health Information (PHI) safety.

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
   - **PHI access without a corresponding audit log entry**
   - **A new API route that returns PHI without auth middleware**

3. **Scope** — Are there unintended side effects or scope creep? Changes that don't relate to the apparent intent should be flagged.

Print your big picture assessment as:

### Big Picture
[One sentence describing the intent of the change.]
[Assessment of completeness. List any missing changes as:]
- **Missing:** [description of what's missing and why you'd expect it]
[Note on scope if there are concerns, otherwise: "The scope looks appropriate. No unrelated changes."]

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

- **Critical** — must fix before committing. Security vulnerabilities, data loss risks, broken logic, crashes, **HIPAA violations**.
- **Important** — should fix. Missing error handling, pattern inconsistencies, edge cases, potential bugs, **PHI exposure risks**.
- **Suggestion** — nice to have. Readability improvements, minor refactors, style nits.

### Output Format

For each functional area that has changes, print:

### [Area Name] ([N] issues)
- **[Severity]:** [One-line description] — `file/path:line`

If an area has no issues:

### [Area Name]
- No issues found. [Brief note on what was reviewed.]

Line numbers come from diff hunk headers and may be approximate. Always include them when possible.

## Step 4: Language-Specific Insights (Phase 3)

Identify every programming language present in the diff. For each language, provide a short section covering:

1. **Idioms & Patterns** — Point out where the code uses (or misses) language-specific idioms. For example: Go's error-return convention, Rust's ownership model, Python's context managers, JavaScript's async/await patterns, Ruby's blocks, Swift's optionals, etc. If the code does something the "standard" way, briefly explain *why* that pattern is idiomatic in this language.

2. **Design Decisions Shaped by the Language** — Call out choices that exist *because* of the language's type system, concurrency model, memory model, or ecosystem conventions. Examples: choosing a struct over a class in Swift, using channels instead of mutexes in Go, preferring pattern matching over if/else chains in Rust or Elixir, using dataclasses vs namedtuples in Python.

3. **Gotchas** — Flag any language-specific pitfalls in the changed code. Examples: nil pointer dereference in Go, unhandled None in Python, implicit type coercion in JavaScript, borrow checker issues in Rust, mutable default arguments in Python.

### Output Format

For each language in the diff:

### Language Insights: [Language]
- **Idiom:** [observation] — `file/path:line`
- **Design:** [why this choice makes sense (or doesn't) in this language] — `file/path:line`
- **Gotcha:** [potential pitfall] — `file/path:line`

Only include bullets that are genuinely informative. Skip a category if there's nothing worth noting. If the diff contains only one language, still include this section. If a language appears only in config files (JSON, YAML, TOML), skip it unless there's something substantive to say.

## Step 5: HIPAA Compliance Review (Phase 4)

Review every change through the lens of HIPAA compliance and PHI safety. This phase is mandatory for all reviews in this project.

### HIPAA Risk Areas

Check the diff for each of the following. Only report findings that are actually present — do not manufacture issues.

1. **PHI Exposure** — Patient data appearing where it should not:
   - PHI in `console.log`, `console.error`, `console.warn`, or any logging output
   - PHI in error messages or exception text that could reach the client
   - PHI in API error responses (stack traces, validation messages that echo back field values)
   - PHI in comments or TODO notes in source code

2. **Client-Side PHI** — Patient data stored or visible in the browser/app:
   - PHI in `localStorage`, `sessionStorage`, `IndexedDB`, or cookies
   - PHI in URL parameters or path segments (appears in browser history, server logs, referrer headers)
   - PHI in React/Next.js client-side state when it should be in a Server Component
   - PHI returned as JSON to the client when it could be rendered server-side as HTML
   - PHI in browser-cacheable responses (missing `Cache-Control: no-store`)
   - `autocomplete` not disabled on PHI form fields

3. **Audit Logging Gaps** — PHI access without accountability:
   - API routes that read, create, update, or delete PHI without writing an audit log entry
   - Missing user identification in audit log entries (who accessed what, when)
   - Audit log entries that themselves contain excessive PHI

4. **Access Control** — PHI accessible without proper authorization:
   - API endpoints returning PHI without authentication middleware
   - Missing role-based access checks (e.g., nurse accessing admin-only data)
   - Queries not scoped by organization (missing RLS or tenant filter)
   - Overly permissive CORS configuration

5. **Secrets and Credentials** — Sensitive values in source:
   - Hardcoded API keys, database credentials, tokens, or passwords
   - Secrets in environment files that are not gitignored
   - AWS credentials or KMS key IDs in source code

6. **Session and Authentication** — Weak session security:
   - Missing session timeout or auto-logoff
   - Cookies missing `HttpOnly`, `Secure`, or `SameSite` flags
   - Tokens stored insecurely (e.g., localStorage instead of HttpOnly cookies)
   - Missing MFA enforcement on PHI-accessing routes

7. **Encryption and Transport** — Data not protected in transit or at rest:
   - HTTP URLs instead of HTTPS
   - Database connections without SSL enforcement
   - Unencrypted file storage or document handling
   - Missing `Strict-Transport-Security` headers

8. **Notification and Messaging** — PHI leaking through notifications:
   - Patient names, diagnoses, or other PHI in push notification content
   - PHI in email subjects or preview text
   - PHI in alert/monitoring payloads (CloudWatch, Slack, PagerDuty)

9. **Input Validation** — Insufficient boundary protection:
   - Missing server-side validation on API routes that accept PHI
   - SQL injection vectors on queries involving PHI
   - XSS vectors that could expose PHI in the DOM
   - Missing request body size limits on PHI upload endpoints

10. **Third-Party Leakage** — PHI exposed to external services:
    - Analytics scripts (Google Analytics, Mixpanel) that could capture PHI from the page
    - Error tracking services (Sentry, Bugsnag) that could capture PHI from errors or DOM
    - Third-party scripts loaded on pages that display PHI without a BAA with that vendor
    - CDN caching of pages or API responses containing PHI

### Output Format

### HIPAA Compliance ([N] issues)
- **[Severity]:** [HIPAA risk area] — [one-line description] — `file/path:line`

If no HIPAA issues are found:

### HIPAA Compliance
- No HIPAA issues found. [Brief note on what was checked.]

Use **Critical** for direct PHI exposure or missing access controls. Use **Important** for gaps that create risk but don't directly expose PHI (e.g., missing audit logging, insecure cookie flags). Use **Suggestion** for defense-in-depth improvements.

## Output Structure

Combine all phases into a single output. Print:

## Code Review (HIPAA)

[Phase 1: Big Picture section]

[Phase 2: Each functional area section]

[Phase 3: Language Insights section]

[Phase 4: HIPAA Compliance section]

---

## Important Guidelines

- Be specific. Every finding must reference a file and line number when possible.
- Be concise. One line per finding. Save details for the conversation if the user asks.
- Don't repeat the diff. The user can see it — tell them what's wrong with it.
- Don't praise the code. Focus on issues and missing pieces. "No issues found" is sufficient for clean areas.
- If there are no issues at all, say so briefly. Don't manufacture findings.
- **HIPAA findings are never suggestions to skip.** If PHI is exposed, it is Critical regardless of context.
- **When in doubt, flag it.** A false positive on a HIPAA finding is far less costly than a missed violation.
