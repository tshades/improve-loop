# Plan Review — Agent Instructions

These instructions are used by the improve-loop orchestrator. The orchestrator reads this file and distributes the agent missions below to parallel Agent() sub-agents, passing each the full plan text and project context.

## Context Gathering (orchestrator does this before launching agents)

Before launching agents, the orchestrator gathers:
- The full plan text
- `PROJECT_CONTEXT.md` for architecture and stack
- `DECISIONS.md` for prior architectural decisions
- `tasks/lessons.md` for known pitfalls
- All third-party APIs, libraries, and services mentioned in the plan
- All user-facing features the plan touches

---

## Agent 1: Architecture & Design Review

```
subagent_type: Plan
```

**Mission:** Review the plan as a principal engineer would during a design review.

Evaluate:
- Does each step have a clear, singular responsibility?
- Are there missing steps or implicit assumptions?
- Is the sequencing optimal? Can anything be parallelized?
- Are there simpler alternatives that achieve the same outcome?
- Does it align with existing architecture (check PROJECT_CONTEXT.md, DECISIONS.md)?
- Are rollback/failure paths considered?
- Is the scope right? Is anything over-engineered or under-specified?

Return: Ordered list of concerns (critical -> minor), concrete rewording suggestions for weak steps, and any steps that should be added/removed/reordered.

---

## Agent 2: Prior Art & Industry Research

```
subagent_type: general-purpose
```

**Mission:** Research how leading companies and cutting-edge apps solve the same problems this plan addresses. Use Firecrawl search (NOT WebSearch) for richer results.

Research:
- How do top companies (relevant to the domain) implement similar features?
- Are there well-known open-source implementations or patterns to reference?
- What are common pitfalls others have hit solving this problem?
- Are there emerging best practices or newer approaches the plan should consider?
- Search for "[feature/problem] best practices 2025/2026", "[feature] architecture at scale", "[technology] production lessons learned"

Return: 3-5 concrete insights from real-world implementations, with source URLs. For each: what they did, why it's relevant, and how the plan should adapt.

---

## Agent 3: API & Library Best Practices Audit

```
subagent_type: general-purpose
```

**Mission:** For EVERY third-party API, SDK, or library referenced in the plan, verify the plan follows current best practices. Use Context7 (resolve-library-id -> query-docs) for each library. Use Firecrawl to search for known issues and migration guides.

For each API/library, verify:
- Is the plan using the latest stable API version/patterns?
- Are there deprecated methods or endpoints being used?
- Does error handling follow the library's recommended patterns?
- Are rate limits, retries, and backoff strategies accounted for?
- Are there security advisories or known gotchas?
- Is authentication/token management done correctly?
- Are WebSocket/streaming patterns correct (lifecycle, reconnection, cleanup)?

Return: Per-library findings with code-level specifics. Flag anything that contradicts official docs.

---

## Agent 4: Edge Cases, Failure Modes & Robustness

```
subagent_type: general-purpose
```

**Mission:** Adversarially stress-test the plan. Think like a chaos engineer and a QA lead combined. Explore the codebase to understand current error handling patterns.

Probe for:
- What happens when each external dependency is down or slow?
- What are the race conditions and timing issues?
- What happens under concurrent users / rapid repeated actions?
- Are there data consistency issues (stale state, partial writes)?
- What happens on mobile / slow networks / Safari?
- Memory leaks, orphaned listeners, unclosed connections?
- What does the degraded experience look like? Is there graceful fallback?
- What would cause silent data corruption vs. loud failures?

Return: Prioritized list of failure scenarios the plan doesn't address, with suggested mitigations for each.

---

## Agent 5: Focus-Area Deep Dive (conditional)

```
subagent_type: general-purpose
```

**Only launch this agent if the user specified a focus area.** Tailor the agent's mission to that focus:

- **Security**: OWASP top 10 analysis, auth flows, input validation, secrets management, CSP, CORS
- **Performance**: Rendering bottlenecks, bundle size impact, network waterfall, caching strategy, perceived latency
- **UX**: User flow gaps, error states users see, accessibility (WCAG), loading states, mobile experience
- **Architecture**: Coupling analysis, testability, deployment complexity, future extensibility
- **Cost**: API pricing impact, compute costs, bandwidth, storage growth projections
- **[Custom]**: Deep research on whatever the user specified

Return: Detailed analysis specific to the focus area with actionable recommendations.

---

## Synthesis Rules (orchestrator applies these after agents return)

- **Deduplicate** across agents — same issue found by multiple agents = higher confidence
- **Conflict resolution** — if agents disagree, present both perspectives with your recommendation
- **Be concrete** — every issue must have a specific fix, not just "consider X"
- **Preserve the user's intent** — improve the plan, don't replace it with your own vision
- **Calibrate severity honestly** — not everything is critical; most things are suggestions
- If the plan is genuinely solid, say so. Don't manufacture issues for thoroughness theater.

### Severity Categories
- **Critical** — Issues that would cause failures, data loss, security holes, or architectural regret
- **Important** — Issues that would cause pain later, degrade UX, or miss clear best practices
- **Suggestions** — Polish, optimizations, things that would make it staff-engineer-approved

## Anti-Patterns to Avoid

- Don't just validate the plan — challenge it
- Don't suggest adding complexity without justifying the risk it mitigates
- Don't recommend technologies or patterns not already in the stack unless the benefit is overwhelming
- Don't turn a focused plan into a kitchen-sink epic
- Don't parrot generic best practices — be specific to THIS plan, THIS codebase, THIS stack
