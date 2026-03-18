# Validation (Improve Check) — Agent Instructions

These instructions are used by the improve-loop orchestrator. The orchestrator reads this file and distributes the validation mission to Agent() sub-agents, passing each a batch of changes to validate.

## Core Principle

The previous AI pass (plan improvement or code review) may have:
- **Hallucinated a bug** that doesn't exist, then "fixed" it
- **Misunderstood an API** and changed correct code to incorrect code
- **Over-engineered** by adding unnecessary error handling, validation, or abstraction
- **Changed behavior** unintentionally while "improving" code
- **Made correct fixes** that should be kept

Your job: separate the good from the bad, revert the bad, keep the good.

---

## Step 1: Identify & Categorize Changes (orchestrator does this)

The orchestrator will:
1. Run `git diff HEAD` to get the full diff of all changes
2. Run `git diff HEAD --stat` for a file summary
3. Run `git diff HEAD --name-only` for the changed file list
4. For each changed file, read the **full current version** AND the **original version** (via `git show HEAD:path/to/file`)
5. Parse the diff into individual **hunks** (logical change units)

For each hunk, classify it:
- **Bug fix** — Claims to fix a bug (null check, error handling, race condition, etc.)
- **API/SDK correction** — Changes how a third-party API/library is called
- **Security fix** — Addresses a security concern
- **Performance improvement** — Optimizes something
- **Refactor** — Restructures without changing behavior
- **Plan annotation** — Adds review notes/checkboxes to a plan file
- **New code** — Adds entirely new functionality

---

## Step 2: Validation Agent Mission

Split the changed files into batches and launch **parallel validation agents**. Each agent validates a batch of changes. If there are fewer than 4 changed files, use a single agent.

Each agent receives:
- The diff hunks for its assigned files
- The full original file content (via `git show HEAD:path`)
- The full current file content
- These instructions

### For EACH individual change (hunk), perform these checks:

#### Check 1: Does the "problem" actually exist?
- Read the **original code** carefully. Is there actually a bug/issue, or did the reviewer imagine one?
- For null checks: Can the value actually be null/undefined in practice? Trace the data flow.
- For error handling: Can this error actually occur? Check callers, types, and guarantees.
- For race conditions: Is there actually concurrent access? Or is this single-threaded UI code?
- **Verdict:** `real_problem` or `phantom_problem`

#### Check 2: Is the fix correct?
- Does the change actually fix what it claims to fix?
- Does it introduce new bugs? (Changed logic, different return types, broken callers)
- For API/SDK changes: Use **Context7** (`resolve-library-id` -> `query-docs`) to verify the "correct" usage is actually correct. The previous reviewer may have hallucinated API behavior.
- Does the fix match how the rest of the codebase handles the same pattern?
- **Verdict:** `correct_fix`, `incorrect_fix`, or `partially_correct`

#### Check 3: Is the change necessary?
- Does this change provide meaningful value, or is it defensive fluff?
- Would a senior engineer on this team make this change in a real PR?
- Is it adding complexity for an edge case that can't realistically happen?
- Is it changing working code to a different-but-not-better pattern?
- **Verdict:** `necessary`, `unnecessary`, or `marginal`

#### Check 4: Does it preserve behavior?
- For refactors: Does the refactored code produce identical results for all inputs?
- Are there subtle behavior changes hidden in the "improvement"? (Different falsy handling, changed execution order, new async boundaries)
- **Verdict:** `behavior_preserved` or `behavior_changed`

### Agent Output Format (per hunk)

```
### [file:lines] — Change Description
- **Category:** bug-fix | api-correction | security | performance | refactor | plan-annotation | new-code
- **Problem exists:** yes | no | uncertain
- **Fix correct:** yes | no | partially
- **Necessary:** yes | no | marginal
- **Behavior preserved:** yes | n/a | no — [what changed]
- **Verdict:** KEEP | REVERT | MODIFY — [reason]
- **If MODIFY:** [what specifically to change about the fix]
```

---

## Step 3: Act on Findings (orchestrator does this)

After all validation agents return, the orchestrator processes each change:

### KEEP — Do nothing. The change is valid.

### REVERT — Restore the original code.
- Use the Edit tool to replace the changed code with the original code (from `git show HEAD:path`).
- Be surgical — only revert the specific hunk, not the entire file.

### MODIFY — Fix the fix.
- Edit the change to correct it while preserving the valid intent.
- If the problem is real but the fix is wrong, implement the correct fix.

**IMPORTANT:** Make reverts/modifications one file at a time. After editing a file, re-read it to confirm the result is valid (no syntax errors, no orphaned code).

---

## Step 4: Report Results

The orchestrator reports:

```markdown
## Validation Results

- Changes analyzed: X hunks across Y files
- Kept: N (valid, correct changes)
- Reverted: N (hallucinated or unnecessary)
- Modified: N (right idea, wrong execution)

### Reverted Changes
[For each reverted change:]
#### [file:lines] — [description]
**Why:** [one-line reason]

### Modified Changes
[For each modified change:]
#### [file:lines] — [description]
**Original fix:** [what the reviewer did]
**Problem:** [what was wrong with the fix]
**Corrected to:** [what you changed it to]

### Kept Changes
[Brief list — one line each]
- `file.js:42` — [description]
```

---

## Anti-Patterns — Do NOT Do These

- **Don't rubber-stamp everything** — The whole point is to catch bad changes. If you keep 100% of changes, you probably weren't skeptical enough.
- **Don't revert everything** — The previous pass likely found real issues. Be fair.
- **Don't guess at API behavior** — If a change claims "this API returns X not Y", use Context7 to verify. Don't assume the reviewer was right OR wrong.
- **Don't leave broken code** — After reverting a hunk, make sure the surrounding code still works (no dangling references, missing imports, orphaned variables).
- **Don't expand scope** — You're validating existing changes, not finding new issues. That's what code review is for.
- **Don't skip the original code** — Always read the pre-change version. You can't judge a fix without seeing what it fixed.
