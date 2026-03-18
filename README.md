# Code Quality Skills Suite

AI reviewing AI — then reviewing the review.

Most AI-generated plans and code changes have a dirty secret: ~20-40% of "improvements" are hallucinated fixes for problems that don't exist, incorrect API usage confidently presented as best practice, or over-engineered solutions that add complexity without value. You wouldn't ship code without code review. Why would you ship an AI-generated plan without one?

This suite solves that with a multi-layered approach:

1. **Parallel expert agents** attack your plan and code from 6 different angles simultaneously — architecture, security, API compliance, performance, edge cases, and code quality. Each agent is a specialist, not a generalist trying to do everything.

2. **Hallucination validation** reviews every change the experts proposed, going hunk-by-hunk to verify: does this problem actually exist? Is the fix correct per the real docs (checked via Context7)? Would a senior engineer actually make this change? Bad changes get surgically reverted.

3. **The convergence loop** chains it all together — improve, validate, review code, validate again, repeat — until there's nothing left to improve or strip. Each iteration catches issues the previous one missed, while validation prevents the review from drifting into fiction.

The result: plans that have been stress-tested by multiple expert perspectives, with every suggestion verified against reality. No hallucination theater. No thoroughness padding. Just real improvements that survive scrutiny.

## Skills Overview

| Skill | Command | Purpose |
|-------|---------|---------|
| **Improve Plan** | `/improve-plan [source] [focus]` | Multi-agent plan review — architecture, research, API audit, edge cases |
| **Code Review** | `/code-review [target]` | Find bugs, security holes, API misuse, and quality issues |
| **Improve Check** | `/improve-check [ref]` | Validate AI-generated changes — catch hallucinations |
| **Improve Loop** | `/improve-loop <plan-file>` | Iterative convergence loop chaining all skills |

All four skills work standalone. The loop orchestrates them via Agent() sub-agents using embedded instructions from `agents/` — it does not call the standalone skills directly (Skill() breaks loop control).

---

## `/improve-plan` — Multi-Agent Plan Review

Launches **4-5 parallel expert agents** to deeply review and improve a plan:

| Agent | Focus | Key Checks |
|-------|-------|------------|
| Architecture & Design | Principal engineer review | Step clarity, sequencing, simplicity, rollback paths, scope |
| Prior Art & Industry | Real-world research | How top companies solve this, open-source patterns, common pitfalls |
| API & Library Audit | Best practices | Context7 docs verification, deprecated patterns, rate limits, auth |
| Edge Cases & Robustness | Chaos engineering | Dependency failures, race conditions, data consistency, graceful degradation |
| Focus Deep Dive | User-specified | Security, performance, UX, architecture, cost (only if focus provided) |

### Usage

```
/improve-plan                              # Auto-detect from plan mode or tasks/todo.md
/improve-plan tasks/todo.md                # Review specific plan file
/improve-plan tasks/todo.md security       # Review with security focus
/improve-plan https://example.com/plan     # Review plan from URL
```

### Output

Severity-rated findings (Critical > Important > Suggestions) plus a "Revised Plan (proposed)" showing changes with ~~strikethrough~~ for removals and **bold** for additions.

---

## `/code-review` — Comprehensive Code Review

Launches **6 parallel expert agents**, each with a focused review mission:

| Agent | Focus | Key Checks |
|-------|-------|------------|
| Bugs & Logic | Correctness | Null refs, race conditions, async errors, stale closures, resource leaks |
| API/SDK Compliance | Library usage | Context7 docs verification, deprecated patterns, auth handling |
| Security | OWASP-informed | Injection, XSS, secrets exposure, webhook verification |
| Performance | Efficiency | React re-renders, memory leaks, N+1 queries, bundle size |
| Code Quality | Maintainability | DRY violations, complexity, dead code, test gaps |
| Architecture | System fitness | Layer violations, coupling, state ownership, error propagation |

### Usage

```
/code-review                              # Review git-changed files
/code-review src/hooks/useTTS.js          # Review specific file(s)
/code-review my Twilio integration        # Search codebase, review all related files
/code-review tasks/todo.md               # Plan mode: review code plan touches, update plan
```

### Output

Severity-rated findings: Critical > Important > Minor. Each finding includes `file:line`, current code, suggested fix, and confidence level.

---

## `/improve-check` — Hallucination Validator

Runs **after** `/code-review` or `/improve-plan` to validate their changes aren't hallucinated, unnecessary, or incorrect.

### What it checks (per change)

1. **Does the problem actually exist?** Traces data flow, checks if null/error/race is actually possible
2. **Is the fix correct?** Verifies against Context7 docs — the previous reviewer may have hallucinated API behavior
3. **Is it necessary?** Would a senior engineer make this change in a real PR?
4. **Does it preserve behavior?** Catches hidden side effects in "improvements"

### Verdicts

- **KEEP** — Valid change, leave it
- **REVERT** — Hallucinated or unnecessary, surgically restore original code
- **MODIFY** — Right idea, wrong execution, apply corrected version

### Usage

```
/improve-check                # Validate uncommitted changes vs HEAD
/improve-check HEAD~3         # Validate against a specific ref
```

---

## `/improve-loop` — The Convergence Loop

Iteratively improves a plan file by chaining plan review, validation, and code review in cycles until the plan stabilizes. Think of it as a "Ralph Wiggum loop" that keeps going around until there's nothing left to improve.

### The Loop

```
                    +------------------+
                    |  /improve-plan   |  Review plan with 4 expert agents
                    +--------+---------+
                             |
                    +--------v---------+
                    | /improve-check   |  Validate plan changes (revert hallucinations)
                    +--------+---------+
                             |
                    +--------v---------+
                    |  /code-review    |  Discover ALL domain files, review code,
                    |  (full domain)   |  find gaps the plan missed
                    +--------+---------+
                             |
                    +--------v---------+
                    | /improve-check   |  Validate code review gaps
                    +--------+---------+
                             |
                    +--------v---------+
                    |  Exit gate?      |--NO--> Loop back to /improve-plan
                    +--------+---------+
                             |
                            YES
                             |
                    +--------v---------+
                    |  Collapse & Report |
                    +------------------+
```

### Exit Conditions

The loop exits when ANY of these are true:

1. **Converged** — Neither plan review nor code review proposed any changes (plan is stable)
2. **All Stripped** — Changes were proposed but validation reverted them all (only hallucinations)
3. **Max Iterations** — Safety valve reached (default: 3)

### Usage

```
/improve-loop tasks/todo.md              # Default 3 iterations
/improve-loop tasks/todo.md max=5        # Up to 5 iterations
```

### How Code Review Works in the Loop

The code review phase doesn't just review files the plan mentions. It:

1. **Parses the plan** to identify the domain (e.g., "Twilio integration")
2. **Searches the entire codebase** for ALL related files — services, hooks, API routes, configs, tests
3. **Runs a full code review** on the discovered files with 5 parallel agents
4. **Compares findings against the plan** to identify gaps
5. **Appends only the gaps** as new action items

This means each iteration can discover issues the plan didn't know about.

### Git Checkpoint Strategy

The loop uses micro-commits to track changes between phases:

```
Start:   HEAD = abc123 (saved to /tmp/improve-loop-orig-head.txt)
         Commit existing changes as checkpoint 0

Phase A: Plan review agents run, revised plan applied to file
Phase C: Validation runs on uncommitted changes (git diff HEAD)
         Surviving changes committed as checkpoint

Phase D: Code review agents run, gaps appended to plan
Phase E: Validation runs on uncommitted changes (git diff HEAD)
         Surviving changes committed as checkpoint

... repeat ...

End:     git reset --soft abc123
         All checkpoint commits collapse to staged changes
         User reviews with `git diff --cached` and commits manually
```

Key: Each `git diff HEAD` only shows the current phase's changes because prior phases were committed as checkpoints. At the end, `git reset --soft` collapses everything back to uncommitted so the user sees one clean diff.

---

## File Structure

```
.claude/skills/
  improve-plan/
    SKILL.md                    # Standalone plan improvement skill
  code-review/
    SKILL.md                    # Standalone code review skill
  improve-check/
    SKILL.md                    # Standalone validation skill
  improve-loop/
    SKILL.md                    # Orchestrator (lean — just loop logic + gates)
    README.md                   # This file
    agents/
      plan-review.md            # Full plan improvement agent instructions
      validation.md             # Full change validation agent instructions
      code-review.md            # Full code review agent instructions
```

The loop's `SKILL.md` is lean orchestration. The full agent instructions live in `agents/` and are read once at loop start, then reused as prompts for Agent() sub-agents at each phase.

---

## Using Skills Together (Manual Pipeline)

If you prefer to run each step manually instead of using the loop:

```bash
# 1. Improve your plan
/improve-plan tasks/todo.md

# 2. Validate the improvements aren't hallucinated
/improve-check

# 3. Review the actual code
/code-review my Twilio integration

# 4. Validate the code review findings
/improve-check
```

Or let the loop do it automatically:

```bash
/improve-loop tasks/todo.md
```

---

## Design Decisions

- **Agent() not Skill()** — The loop uses Agent() sub-agents instead of Skill() calls. Skill() hands off control and the agent considers itself "done" after the sub-skill finishes. Agent() returns results to the orchestrator, which stays in control of the loop.
- **`git add -u` not `git add -A`** — Only stages tracked files to avoid accidentally committing secrets or temp files.
- **ORIG_HEAD in temp file** — Persisted to `/tmp/improve-loop-orig-head.txt` so it survives context compression during long loops.
- **Repo-relative paths** — Plan file paths are resolved to repo-relative form for `git show HEAD:<path>` compatibility.
- **Context management** — Between iterations, verbose agent output is discarded. Only stats and summaries carry forward. Git checkpoints are the source of truth.
