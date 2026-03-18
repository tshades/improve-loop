---
name: improve-loop
description: "Iterative plan improvement loop. Chains plan improvement, validation, and code review in cycles until convergence. Each iteration: improves the plan, validates changes, discovers code issues across the full domain, validates those, and loops. Usage: /improve-loop <plan-file> [max=N]. Requires a plan file path — if omitted, asks for one. max is optional iteration limit (default 3). Use when: improve loop, iterate on plan, loop improve, refine plan, keep improving, convergence loop, full review loop."
allowed_tools: "Read,Glob,Grep,Agent,Bash,Edit,Write"
---

# Improve Loop — Orchestrator

You are the **loop controller**. You launch Agent() sub-agents, apply their results, manage git checkpoints, and enforce the loop. You NEVER delegate to other skills via Skill().

```
┌─────────────────────────────────────────────────────────────┐
│  MANDATORY LOOP — YOU MUST NOT STOP UNTIL AN EXIT GATE.     │
│                                                             │
│  • Do NOT stop after any single phase                       │
│  • Do NOT ask "should I continue?" — ALWAYS continue        │
│  • Do NOT use Skill() — use Agent() for all sub-work        │
│  • After EVERY phase, pass through the GATE to the next     │
│  • The ONLY exit is Phase F when an exit condition is met    │
│                                                             │
│  Agent instructions live in the agents/ subdirectory:       │
│  • agents/plan-review.md   — plan improvement agents        │
│  • agents/validation.md    — change validation agents       │
│  • agents/code-review.md   — code review agents             │
│  Read each file ONCE at the start, then reuse the content.  │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 0: Setup

### Parse arguments
1. **Plan file (required):** First token that looks like a file path. Read it to confirm it exists.
2. **Max iterations (optional):** Token matching `max=N` or a bare number. Default: `3`.
3. If no plan file: ask the user and stop.

### Load agent instructions
Read these files now — you'll use their contents as prompts for sub-agents throughout the loop:
```
.claude/skills/improve-loop/agents/plan-review.md
.claude/skills/improve-loop/agents/validation.md
.claude/skills/improve-loop/agents/code-review.md
```

### Load project context
Read these files — you'll pass summaries to sub-agents:
- `PROJECT_CONTEXT.md`
- `DECISIONS.md`
- `tasks/lessons.md`

### Git checkpoint

Save ORIG_HEAD to a temp file so it survives context compression:
```bash
git rev-parse HEAD > /tmp/improve-loop-orig-head.txt
cat /tmp/improve-loop-orig-head.txt
```

If working tree is dirty, checkpoint ONLY tracked files:
```bash
if ! git diff --quiet HEAD 2>/dev/null || ! git diff --cached --quiet 2>/dev/null; then
    git add -u && git commit -m "[improve-loop] checkpoint 0: pre-loop state"
fi
```

### Resolve plan file path
Convert the plan file to a **repo-relative path** for use with `git show`:
```bash
git ls-files --full-name <plan-file>
```
Store this relative path — you'll need it for `git show HEAD:<relative-path>` in validation phases.

### Initialize tracking
- `iteration = 0`, `max_iterations` from args (default 3)
- Per-iteration: `plan_proposed`, `plan_kept`, `code_proposed`, `code_kept`

Output: "Starting improve loop on `[file]` — max [N] iterations"

**GATE: Proceed to Step 1.**

---

## Step 1: Begin Iteration

Increment `iteration`. Output: "**--- Iteration [N] of [max] ---**"

---

### Phase A: Improve Plan

Read the current plan file.

Using the agent definitions from `agents/plan-review.md`, launch **4 parallel Agent() calls** in a SINGLE message. Pass each agent:
- The full plan text
- Project context summary (PROJECT_CONTEXT.md, DECISIONS.md, lessons.md)
- Its specific mission from the plan-review.md file

The agents are:
1. Architecture & Design Review (`subagent_type: Plan`)
2. Prior Art & Industry Research
3. API & Library Best Practices Audit
4. Edge Cases, Failure Modes & Robustness

Wait for all agents to return.

### Synthesize & Apply

Using the synthesis rules from `agents/plan-review.md`:
1. Deduplicate findings across agents
2. Categorize by severity: Critical > Important > Suggestions
3. Write a revised version of the plan that incorporates all critical and important fixes
4. **Save the revised plan directly to the plan file** using Edit or Write

**GATE: Plan applied. Proceed to Phase B.**

---

### Phase B: Check Plan Diff

```bash
git diff HEAD --stat
```

- **Empty** → `plan_proposed = 0`. Skip to Phase D.
- **Changes exist** → `plan_proposed = [count]`. Proceed to Phase C.

**GATE: Proceed to Phase C or D.**

---

### Phase C: Validate Plan Changes

Get the diff and originals:
```bash
git diff HEAD
git show HEAD:<plan-file-path>
```

Read the current file too.

Using the agent definitions from `agents/validation.md`, launch **1-2 parallel Agent() calls**. Pass each:
- The diff hunks for its assigned files
- The original file content (from git show)
- The current file content
- The full validation mission from validation.md

Wait for agents to return.

**Apply verdicts yourself:**
- **KEEP** → do nothing
- **REVERT** → use Edit to restore original text from `git show HEAD:<path>`
- **MODIFY** → use Edit to apply the corrected version

Record stats: `plan_kept`, `plan_reverted`, `plan_modified`

Check what survived:
```bash
git diff HEAD --stat
```

- **Empty** → `plan_kept = 0`
- **Changes remain** → checkpoint:
```bash
git add -u && git commit -m "[improve-loop] iteration [N]: plan improvements"
```

**GATE: Checkpoint done. Proceed to Phase D.**

---

### Phase D: Code Review — Full Domain Discovery

#### D1: Identify the domain

Read the plan file. Identify the primary domain/integration/feature and its keywords. Examples:
- "Twilio integration" → twilio, TwiML, phone, caller, voice, recording
- "billing flow" → stripe, subscription, payment, invoice, pricing
- "audio pipeline" → audio, TTS, STT, WebSocket, microphone, transcription

#### D2: Discover ALL related files

Search the **entire codebase** — cast a wide net beyond what the plan mentions:
```
Grep(pattern: "<keyword1>|<keyword2>|<keyword3>", output_mode: "files_with_matches")
Glob(pattern: "**/*<domain-term>*")
Glob(pattern: "api/*<domain-term>*")
```

Also find: importers/callers of discovered files, config files, test files.

#### D3: Launch code review agents

Read the discovered files (batch if >15). Using `agents/code-review.md`, launch **up to 5 parallel Agent() calls**:
1. Bugs & Logic Errors (Agent 1 from code-review.md)
2. API & SDK Compliance (Agent 2 from code-review.md)
3. Security Audit (Agent 3 from code-review.md)
4. Performance & Efficiency (Agent 4 from code-review.md)
5. Code Quality & Architecture (Agent 5 + Agent 6 missions combined into one prompt)

Pass each: file contents, project context, its mission from code-review.md.

Wait for all agents to return.

#### D4: Compare findings to plan

Read the current plan. For each finding:
- Plan already covers it → skip
- Plan misses it → include as a gap

#### D5: Append gaps

If gaps exist, use Edit to append to the plan file:

```markdown

## Code Review Gaps (Iteration [N])

_Discovered by reviewing [X] files across the [domain] domain_

### [Category]
- [ ] [finding] — `file:line`
```

Check:
```bash
git diff HEAD --stat
```

- **Empty** → `code_proposed = 0`. Skip to Phase F.
- **Changes** → `code_proposed = [count]`. Proceed to Phase E.

**GATE: Proceed to Phase E or F.**

---

### Phase E: Validate Code Review Gaps

Same pattern as Phase C. Get diff + originals. Using `agents/validation.md`, launch validation agent(s).

Apply verdicts: KEEP / REVERT / MODIFY. Record stats.

Check survivors:
```bash
git diff HEAD --stat
```

- **Empty** → `code_kept = 0`
- **Changes remain** → checkpoint:
```bash
git add -u && git commit -m "[improve-loop] iteration [N]: code review gaps"
```

**GATE: Checkpoint done. Proceed to Phase F.**

---

### Phase F: EXIT GATE

Evaluate exit conditions:

**1. Convergence** — `plan_proposed = 0 AND code_proposed = 0`
→ Nothing new to improve. `exit_reason = "converged"`

**2. All Stripped** — Changes were proposed (`plan_proposed > 0 OR code_proposed > 0`) but none survived (`plan_kept = 0 AND code_kept = 0`)
→ Only hallucinations produced. `exit_reason = "all-stripped"`

**3. Max Iterations** — `iteration >= max_iterations`
→ `exit_reason = "max-iterations"`

```
┌─────────────────────────────────────────────────────────┐
│  IF NO EXIT CONDITION MET:                              │
│  Output: "No exit condition met. Iteration [N+1]..."    │
│  GO BACK TO STEP 1 NOW. DO NOT STOP.                   │
└─────────────────────────────────────────────────────────┘
```

IF an exit condition IS met → proceed to Step 2.

---

## Step 2: Collapse & Report

### Collapse checkpoints

```bash
CURRENT_HEAD=$(git rev-parse HEAD)
```

If `CURRENT_HEAD` != `ORIG_HEAD`:
```bash
git reset --soft $ORIG_HEAD
```

### Summary Report

```markdown
# Improve Loop Complete

## Result
**[exit_reason]** after **[N]** iteration(s)

## Per-Iteration Stats
| Iter | Phase | Proposed | Kept | Reverted | Modified |
|------|-------------|----------|------|----------|----------|
| ... | ... | ... | ... | ... | ... |

## Domain Files Discovered
[Files code-review examined beyond what the plan mentioned]

## Key Changes
- [Major plan improvements]
- [Major code review gaps added]

## Next Steps
Review all changes: `git diff --cached`
Commit when satisfied: `git commit -m "your message"`
Or discard: `git reset HEAD`
```

**The loop is now complete. You may stop.**

---

## Context Management

Between iterations, **discard verbose agent output** from prior iterations. Only carry forward:
- The per-iteration stats (proposed/kept/reverted/modified counts)
- A 1-2 sentence summary of what changed
- The ORIG_HEAD value (also saved in `/tmp/improve-loop-orig-head.txt`)

The git checkpoints capture the actual state. You don't need old agent output — just re-read the plan file at each iteration start.

## Error Recovery

If something goes wrong mid-loop (tool failure, context overflow):
1. Check if any checkpoint commits were made: `git log --oneline ORIG_HEAD..HEAD`
2. If yes, collapse them: `git reset --soft $(cat /tmp/improve-loop-orig-head.txt)`
3. Report what completed and what failed
4. Clean up: `rm /tmp/improve-loop-orig-head.txt`

The user can then decide to re-run or continue manually.

## Anti-Patterns

- **NEVER use Skill()** — It breaks the loop. Use Agent() only.
- **NEVER stop between phases** — Gates are mandatory. Only Phase F can exit.
- **NEVER skip validation** — Unvalidated changes defeat the purpose.
- **NEVER skip checkpointing** — Without commits, validation sees mixed changes.
- **NEVER narrow code-review scope** — Search the FULL codebase for the domain.
- **NEVER forget to collapse** — `git reset --soft` at end is critical.
- **NEVER ask "should I continue?"** — Phase F exit conditions are the only decision point.
- **NEVER use `git add -A`** — Use `git add -u` to only stage tracked files. Avoid staging secrets or temp files.
