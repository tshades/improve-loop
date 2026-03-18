# Code Review — Agent Instructions

These instructions are used by the improve-loop orchestrator. The orchestrator reads this file and distributes the review missions to parallel Agent() sub-agents. In the loop context, the orchestrator handles domain discovery and gap comparison — these agents focus purely on reviewing code.

## Context (orchestrator provides this to each agent)

Each agent receives:
- The list of files to review (and their contents if feasible, otherwise instructions to read them)
- The project context summary (from PROJECT_CONTEXT.md)
- Architectural decisions (from DECISIONS.md)
- Known pitfalls (from tasks/lessons.md)
- Its specific review mission

## Finding Format

Every agent MUST return findings in this format per issue:
```
### [severity] Issue Title
- **File**: path/to/file.js:line_number
- **Category**: [bug|security|performance|api-misuse|quality|architecture]
- **Confidence**: [high|medium|low]
- **Description**: What's wrong and why it matters
- **Fix**: Concrete code change or approach (not vague advice)
```

---

## Agent 1: Bugs & Logic Errors

```
subagent_type: general-purpose
```

**Mission:** Find actual bugs, logic errors, and correctness issues. Read every target file carefully, line by line.

Hunt for:
- **Null/undefined access** — unguarded optional chains, missing null checks before `.map()`, `.length`, etc.
- **Race conditions** — concurrent state updates, stale closures in useEffect/callbacks, missing cleanup
- **Async errors** — unhandled promise rejections, missing try/catch, await in loops without need, dangling promises (missing await)
- **State management** — stale state in closures, setState on unmounted components, missing dependency arrays in useEffect/useMemo/useCallback
- **Off-by-one errors** — array indexing, pagination, boundary conditions
- **Error handling gaps** — catch blocks that swallow errors, missing error states in UI, API calls without error handling
- **Type coercion** — loose equality, string/number confusion, falsy value traps (0, "", null vs undefined)
- **Resource leaks** — unclosed WebSockets, timers not cleared, event listeners not removed, MediaStream tracks not stopped
- **Incorrect conditional logic** — inverted conditions, short-circuit evaluation side effects, missing else branches

Return: Only issues you're confident about. For each, include the exact code snippet that's wrong and the fix.

---

## Agent 2: API & SDK Compliance

```
subagent_type: general-purpose
```

**Mission:** For EVERY third-party API, SDK, or library used in the target files, verify the code follows current best practices. **Use Context7** (`resolve-library-id` then `query-docs`) for each library. Cross-reference with official docs.

For each API/SDK found, check:
- **Correct API usage** — Are methods called with correct parameters? Are return values handled properly?
- **Deprecated patterns** — Is the code using deprecated methods, endpoints, or patterns?
- **Error handling** — Does it follow the library's recommended error handling? (e.g., Supabase returns `{ data, error }` not exceptions)
- **Authentication** — Is auth done correctly? Tokens refreshed? API keys not exposed client-side?
- **Rate limits & retries** — Are rate limits respected? Is there backoff/retry logic where needed?
- **WebSocket lifecycle** — Correct open/close/reconnect/cleanup patterns? Heartbeats where required?
- **Version compatibility** — Are patterns compatible with the version in package.json?
- **Missing features** — Is the code doing something manually that the SDK provides built-in?

Specific integrations to watch for (check Context7 for each):
- **Twilio** — REST API patterns, webhook validation, TwiML construction, phone number formatting
- **Supabase** — RPC calls, auth client usage, realtime subscriptions, row-level security
- **Stripe** — Webhook signature verification, idempotency keys, error handling, test vs live mode
- **Deepgram** — WebSocket streaming, keepalive, encoding params, diarization config
- **Cartesia** — WebSocket TTS, context_id lifecycle, PCM format params
- **React** — Hooks rules, key props, memo usage, concurrent features

Return: Per-library findings with the specific doc reference that contradicts the code.

---

## Agent 3: Security Audit

```
subagent_type: general-purpose
```

**Mission:** Security review informed by OWASP Top 10 2025 and web application security best practices. Think like an attacker.

Check for:
- **Injection** — SQL injection (even with Supabase — check raw queries, RPC params), command injection, template injection
- **XSS** — Unsanitized user input rendered as HTML, `dangerouslySetInnerHTML`, URL parameter reflection
- **Broken authentication** — Missing auth checks on API routes, token exposure in logs/URLs, insecure token storage
- **Broken access control** — Missing authorization checks, IDOR vulnerabilities, privilege escalation paths
- **Secrets exposure** — API keys in client code (check for `VITE_` env vars that shouldn't be public), hardcoded credentials, secrets in error messages/logs
- **CORS/CSP** — Overly permissive CORS, missing CSP headers, wildcard origins
- **Input validation** — Missing validation on API endpoints, type checking at system boundaries, file upload validation
- **Sensitive data** — PII in logs, unencrypted sensitive data, excessive data exposure in API responses
- **Dependency risks** — Known vulnerable packages (check with `npm audit` if relevant)
- **Webhook security** — Missing signature verification on incoming webhooks (Stripe, Twilio)

Return: Each finding with exploitability assessment and concrete fix.

---

## Agent 4: Performance & Efficiency

```
subagent_type: general-purpose
```

**Mission:** Find performance issues, memory leaks, and inefficiencies. Consider both runtime performance and developer experience.

Check for:
- **React rendering** — Unnecessary re-renders (missing memo, unstable references in props, inline object/function creation in JSX), large component trees without code splitting
- **Memory leaks** — Event listeners not cleaned up, timers not cleared, growing arrays/maps without bounds, stale refs holding large objects
- **Network** — Redundant API calls, missing caching, large payloads without pagination, waterfall requests that could be parallel
- **Bundle size** — Large imports that could be tree-shaken or lazy-loaded, unused dependencies
- **Database** — N+1 queries, over-fetching, missing indexes (if SQL visible), unbounded queries
- **WebSocket efficiency** — Unnecessary reconnections, oversized messages, missing compression, idle connections without cleanup
- **Audio/Media** — AudioContext not reused, MediaStream tracks not stopped, excessive buffer allocation
- **Algorithmic** — O(n^2) where O(n) is possible, unnecessary array copies, repeated lookups that should be cached

Return: Each finding with estimated impact (high/medium/low) and the specific optimization.

---

## Agent 5: Code Quality & Patterns

```
subagent_type: general-purpose
```

**Mission:** Review code quality, maintainability, and adherence to codebase conventions. Read existing codebase patterns to understand conventions before flagging deviations.

Check for:
- **DRY violations** — Duplicated logic that should be extracted (but only if genuinely duplicated, not just similar)
- **Complexity** — Functions >50 lines, deeply nested conditionals (>3 levels), god components doing too many things
- **Naming** — Misleading variable/function names, inconsistent naming conventions within the file
- **Error boundaries** — React components that could crash the whole app without error boundaries
- **Dead code** — Unreachable branches, unused variables/imports, commented-out code blocks
- **Consistency** — Deviations from patterns established elsewhere in the codebase (check similar files)
- **Magic values** — Hardcoded numbers/strings that should be named constants
- **Missing edge cases** — Empty arrays, null inputs, network offline, concurrent calls
- **Test coverage gaps** — Critical logic paths with no test coverage (check for corresponding test files)
- **TODO/FIXME/HACK** — Flag any that look stale or urgent

Do NOT flag:
- Style preferences (semicolons, quotes, trailing commas) — that's for linters
- Missing comments on self-explanatory code
- "Could be more functional" style opinions
- Minor naming preferences that don't impact readability

Return: Focus on issues that would cause maintenance pain or bugs, not style nits.

---

## Agent 6: Architecture & Integration Patterns

```
subagent_type: general-purpose
```

**Mission:** Review the code's architectural fitness — does it fit well within the existing system? Read `PROJECT_CONTEXT.md` and `DECISIONS.md` first.

Check for:
- **Layer violations** — Business logic in components, database calls from UI, API keys accessed directly instead of through services
- **Coupling** — Components that know too much about each other's internals, circular dependencies, tight coupling to specific implementations
- **Separation of concerns** — Hooks doing too many things, components mixing data fetching + rendering + business logic
- **Error propagation** — Do errors surface correctly from services -> hooks -> components -> user-facing UI?
- **State ownership** — Is state managed at the right level? Prop drilling that should use context? Global state that should be local?
- **API contract** — Do API endpoints return consistent shapes? Are request/response schemas documented or typed?
- **Consistency with decisions** — Does the code follow patterns established in DECISIONS.md?
- **Integration boundaries** — Are third-party services properly abstracted behind service layers?

Return: Architectural concerns with specific recommendations for restructuring.

---

## Synthesis Rules (orchestrator applies these after agents return)

- **Deduplicate** — Same issue found by multiple agents = higher confidence, list it once with combined context
- **Resolve conflicts** — If agents disagree, present both perspectives with your recommendation
- **Be concrete** — Every issue MUST have a specific fix, not "consider improving"
- **Calibrate honestly** — Most issues are minor or important. Reserve critical for things that will actually break.
- **Don't manufacture issues** — If the code is solid, say so. Never pad a review for thoroughness theater.

## Anti-Patterns — Do NOT Do These

- **Don't review files you haven't read** — Always read the actual code before flagging issues
- **Don't flag style/formatting** — That's for linters, not code review
- **Don't suggest adding types to a JS codebase** — Unless the project uses TypeScript
- **Don't recommend wholesale rewrites** — Suggest incremental improvements
- **Don't parrot generic advice** — Every finding must be specific to THIS code
- **Don't flag things that work** — "This could be done differently" is not a finding unless the current approach has a concrete problem
- **Don't add unnecessary complexity** — "You should add error handling for [impossible scenario]" is noise
- **Don't skip Context7** — For any third-party library, checking current docs is mandatory
