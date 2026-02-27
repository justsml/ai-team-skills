---
name: pr-review-swarm
description: Orchestrate a parallel team of code review specialists — correctness, performance, maintainability, and domain logic — to thoroughly review a PR or branch, then synthesize findings into a unified, actionable review.
---

# PR Review Swarm

Orchestrate a parallel team of code reviewers who each evaluate a PR or branch through a distinct specialist lens, then synthesize findings into a single prioritized review with a clear verdict. Use this skill when the user wants a thorough code review, PR review, or multi-perspective feedback on their changes before merging.

## When to Use

- User asks for a code review, PR review, or review of recent changes
- User wants thorough feedback on a branch before merging
- User says "review my PR", "review this branch", "code review"
- User wants multiple perspectives on their changes
- User asks "is this ready to merge?" or "what did I miss?"

## When NOT to Use

- PR is < 3 files changed or < 50 lines — a single review pass is sufficient, skip the team
- Change is config-only (CI pipeline, linting rules, dependency version bumps) with no application logic
- User wants a pre-PR shape check, not a correctness review — use `sanity-check` instead
- The PR is a revert, a typo fix, or a documentation-only change
- There's no base branch to diff against (initial commit / new repo)

## The Specialists

Each specialist is a `general-purpose` subagent focused on a distinct review dimension. They work in parallel, reviewing the same changes independently, then findings are synthesized.

### 1. Correctness Reviewer

**Focus:** Logic errors, bugs, and broken assumptions introduced by the PR.

**Reviews for:**
- **Off-by-one errors** — incorrect boundary conditions, wrong comparison operators (`<` vs `<=`), fencepost errors in loops and slicing.
- **Null/undefined paths** — unguarded property access on potentially null values, missing optional chaining, unchecked array indexing, destructuring without defaults.
- **Race conditions** — concurrent state mutations, TOCTOU (time-of-check-to-time-of-use) bugs, async operations without proper ordering, missing locks on shared resources.
- **Unhandled errors** — missing try/catch on I/O operations, unhandled promise rejections, swallowed errors that hide failures, catch blocks that silently continue.
- **Edge cases** — empty inputs, maximum/minimum values, negative numbers, unicode strings, empty strings vs null vs undefined, zero-length arrays, single-element arrays.
- **State management bugs** — stale closures in React hooks, missing dependency arrays in useEffect/useMemo/useCallback, state updates on unmounted components, derived state going stale.
- **Type coercion traps** — loose equality (`==`), truthy/falsy gotchas (`0`, `""`, `NaN`), implicit type conversions in comparisons or arithmetic.
- **Regression risk** — does this change break existing behavior? Are all callers of modified functions updated? Are default parameter changes backwards-compatible?

**Methodology:** Traces every code path introduced or modified by the PR. For each branch or condition, asks "what happens if this is false, null, empty, or concurrent?" Reads the full file — not just the diff — to understand how changes interact with surrounding code. Checks both the happy path and every deviation.

---

### 2. Performance Reviewer

**Focus:** Runtime efficiency, resource usage, and scalability of the changes.

**Reviews for:**
- **N+1 queries** — database calls inside loops, sequential fetches that could be batched, ORM lazy-loading triggered in iteration.
- **Missing indexes** — new WHERE, ORDER BY, or JOIN clauses on columns without database indexes. Cross-references query patterns with schema/migration files.
- **O(n^2) or worse** — nested loops over the same collection, array `.includes()` or `.find()` inside a loop, repeated `.filter()` chains, sorting inside iterations.
- **Unnecessary work** — re-computing values that could be cached, re-rendering components that didn't change, fetching data that's already available in scope.
- **Unbounded growth** — arrays/maps that grow without limits, missing pagination on list endpoints, event listeners accumulating without cleanup, caches without eviction.
- **Blocking operations** — synchronous file I/O on request paths, CPU-intensive computation on the main thread, large JSON.parse/stringify in hot paths.
- **Missing parallelism** — sequential `await` calls to independent resources that could be `Promise.all`, serial HTTP calls that could be concurrent.
- **Frontend specifics** — unnecessary re-renders from missing memo/useMemo, large bundle imports for small features (`import _ from 'lodash'`), layout thrashing from DOM reads between writes.

**Methodology:** Identifies hot paths affected by the PR. For each, estimates computational complexity and I/O cost. Compares against what existed before. Flags anything that scales worse than linearly with input size. For database changes, considers whether new query patterns degrade at production data volumes.

---

### 3. Maintainability Reviewer

**Focus:** Will this code be easy to understand, modify, and extend six months from now?

**Reviews for:**
- **Naming clarity** — do variable/function/class names reveal intent? Misleading names? Single-letter variables outside tiny scopes? Inconsistent naming within the PR?
- **Abstraction level** — over-abstraction (premature frameworks, unnecessary indirection, interfaces with one implementation) or under-abstraction (copy-pasted logic, 200-line functions doing multiple things).
- **Coupling introduced** — does this change create hidden dependencies between modules? Shared mutable state? Import cycles? Reaching into another module's internals?
- **Consistency with repo patterns** — does this follow the codebase's conventions for error handling, file organization, naming, and testing? Or does it introduce a competing pattern?
- **Dead code** — removed features with leftover artifacts, unused imports, commented-out code, unreachable branches, TODO comments for things already done.
- **Comments** — complex logic without explanation, outdated comments that describe old behavior, obvious comments that add noise rather than clarity.
- **Test coverage** — are new code paths tested? Do tests verify behavior (not implementation details)? Are edge cases covered?
- **Readability** — deeply nested conditionals, long parameter lists, boolean parameters without named constants, magic numbers/strings.

**Methodology:** Reads the PR as if encountering this code for the first time six months from now, without access to the PR description. Asks "would I understand this?" Checks every new file/function against existing patterns in the repo. Flags deviations from established conventions even when the new code is technically correct.

---

### 4. Domain Logic Reviewer

**Focus:** Does the implementation actually match the intended behavior?

**Reviews for:**
- **Spec adherence** — does the code do what the PR title/description says it should? Are there gaps between stated intent and implementation?
- **Incomplete state machines** — are all valid state transitions handled? What about invalid transitions — are they rejected or silently ignored? Can the system get into a state the UI doesn't handle?
- **Authorization gaps** — do new endpoints/features check permissions? Can users access or modify resources they shouldn't? Are admin-only operations properly guarded?
- **User-facing edge cases** — empty form submission, double-click on submit, back button after mutation, browser refresh mid-operation, expired session during long form fill.
- **Data integrity** — can this change leave the database in an inconsistent state? Partial writes without transactions? Missing cascading updates? Orphaned records?
- **Error UX** — when things fail, does the user see a helpful message or a blank screen / generic 500 / raw error object? Are loading and empty states handled?
- **Backwards compatibility** — do existing API consumers, stored data, persisted sessions, or saved configurations still work after this change?
- **Missing feature cases** — are there scenarios the feature description implies but the code doesn't handle? (e.g., "support file upload" but only .jpg/.png, not .pdf or .heic)

**Methodology:** Reads the PR description as a specification. Traces each stated requirement to its implementation. Identifies requirements that are partially or incorrectly implemented. Considers the end-user's perspective — not just "does the code run" but "does the feature work as intended in all the ways a user would actually use it."

---

### 5. Senior Review Synthesizer ("The Reviewer")

**Focus:** Independent cross-cutting review PLUS deduplication and synthesis of all specialist findings.

**Independent review (done BEFORE reading specialist reports):**
- **Integration risk** — Do the changed files interact with each other in ways that no single specialist would catch? A new API endpoint + a new UI component + a new database migration may each look fine individually but create an inconsistent whole.
- **Change coherence** — Does the PR tell a coherent story? Or are there unrelated changes bundled together that should be separate PRs?
- **Test adequacy** — Stepping back from individual findings: does the PR's test coverage match its risk level? A high-risk change (auth, payment, data mutation) with no tests is a "Must Fix" regardless of what other specialists found.

**Synthesis (done AFTER reading specialist reports):**
- Same issue found by multiple specialists — merge and credit all discoverers.
- Cross-cutting patterns — the same class of issue appearing in multiple places indicates a systemic gap, not just individual mistakes.
- Severity calibration — ensure "Must Fix" items genuinely block merge and "Nits" aren't inflated.
- Gaps between specialists — issues that fall between domains (e.g., a performance bug caused by a domain logic misunderstanding).
- **What everyone else missed** — Reviews all specialist findings for gaps in coverage and false negatives. Asks: "if I were reviewing this PR from scratch, what would I flag that nobody mentioned?"

**Methodology:** First does a brief independent review of the changed files, focusing on integration and coherence. Then reads all four specialist reports. Deduplicates. Adds cross-cutting observations and anything the specialists missed. Determines the overall verdict. Produces the final review.

---

## Workflow

### Step 1: Scope the Review

Before spawning the team, understand the changes:

1. **Identify the branch and base:**
   ```bash
   git branch --show-current
   git merge-base main HEAD
   ```

2. **Get the commit history:**
   ```bash
   git log --oneline main..HEAD
   ```

3. **Get the changeset summary:**
   ```bash
   git diff --stat main...HEAD
   ```

4. **Check for a PR description** (if a PR exists):
   ```bash
   gh pr view --json title,body
   ```

5. **Identify the tech stack and key directories affected**

### Step 2: Create the Team

```
TeamCreate: team_name = "pr-review"
```

Create one task per specialist:

```
TaskCreate: "Correctness Review"
TaskCreate: "Performance Review"
TaskCreate: "Maintainability Review"
TaskCreate: "Domain Logic Review"
TaskCreate: "Review Synthesis"  ← blocked by the above 4
```

### Step 3: Spawn Specialists in Parallel

Launch 4 specialists simultaneously as `general-purpose` subagents with `team_name = "pr-review"`. Each gets:

1. The scoping context from Step 1 (diff summary, commit list, affected files, tech stack)
2. Their specialist role description (copy from the relevant section above)
3. The PR description (if available) or commit messages as intent signal
4. Where to write findings (a unique file per specialist)
5. The findings format (see below)

The 5th agent (Senior Synthesizer) is spawned after the first 4 complete, or at the same time with a `blockedBy` dependency.

### Handling Failures

- **Agent timeout or empty report:** The synthesizer should note the uncovered dimension in the final review and recommend a follow-up focused review. Do not block the entire review because one specialist failed.
- **Contradictory findings:** When two specialists disagree (e.g., Correctness says "this is fine", Domain Logic says "this is broken"), the synthesizer should present both perspectives with evidence and let the user decide.
- **Large PR (30+ files):** Scope each specialist to a subset of changed files rather than the full diff. Divide by domain layer. Include the scoping in each prompt so the synthesizer knows the coverage.

### Step 4: Findings Format

Create the output directory: `.reports/pr-review/` (add `.reports/` to `.gitignore` if not already there).

Each specialist writes findings to `.reports/pr-review/PR-REVIEW-<specialist>.md`. Use this format:

```markdown
# <Specialist Name> — PR Review Findings

**Reviewer:** <Role name>
**Branch:** <branch name>
**Commits Reviewed:** <count>
**Files Reviewed:** <count>

## Must Fix (Blocks Merge)

### [MUST-01] <Title>
- **File:** `path/to/file.ts:42-58`
- **Category:** <e.g., Race Condition, N+1 Query, Missing Auth Check>
- **What:** <What the issue is>
- **Why:** <Why this matters — impact if merged as-is>
- **Suggestion:** <Specific fix, ideally with a code snippet>

## Should Fix (Merge, But Address Soon)

### [SHOULD-01] <Title>
- **File:** `path/to/file.ts:100`
- **Category:** <category>
- **What:** <issue>
- **Suggestion:** <fix>

## Nits (Optional / Stylistic)

### [NIT-01] <Title>
- **File:** `path/to/file.ts:15`
- **What:** <observation>
- **Suggestion:** <alternative>

## Looks Good
<Specific things the PR does well from this specialist's perspective>
```

### Step 5: Synthesis

The Senior Synthesizer reads all findings files, then produces the final `.reports/pr-review/PR-REVIEW.md`:

```markdown
# PR Review — <Branch Name>

**Branch:** <branch>
**Base:** <base branch>
**Commits:** <count>
**Files Changed:** <count>
**Reviewers:** Correctness, Performance, Maintainability, Domain Logic

## Verdict: <APPROVE / REQUEST CHANGES / NEEDS DISCUSSION>

<2-3 sentence summary: overall quality, key concerns, recommendation>

## Must Fix

### [M-01] <Title>
- **Found by:** <Specialist(s)>
- **File:** `path:line`
- **Issue:** <description>
- **Fix:** <specific suggestion>

### [M-02] ...

## Should Fix

### [S-01] <Title>
- **Found by:** <Specialist(s)>
- **File:** `path:line`
- **Issue:** <description>
- **Fix:** <suggestion>

## Nits

| # | File | Issue | Suggestion |
|---|------|-------|------------|
| N-01 | `path:line` | ... | ... |

## Positive Observations
<What the PR does well — good patterns, thoughtful handling, clean code>

## Appendix: Specialist Reports
- [Correctness Review](./PR-REVIEW-correctness.md)
- [Performance Review](./PR-REVIEW-performance.md)
- [Maintainability Review](./PR-REVIEW-maintainability.md)
- [Domain Logic Review](./PR-REVIEW-domain.md)
```

### Step 6: Present & Clean Up

- Present the Verdict and Must Fix items to the user
- If APPROVE: congratulate and note any should-fix items for follow-up
- If REQUEST CHANGES: summarize what needs to change before merge
- Offer to fix specific findings automatically
- Optionally post the review to GitHub via `gh pr review` (see Customization)
- Clean up the team with `TeamDelete`
- Leave all files in `.reports/pr-review/` for the user

## Agent Prompt Template

Use this when spawning each specialist:

```
You are a code review specialist on a parallel review team. Your role is:

<ROLE NAME>

<Paste the full specialist description from the relevant section above>

## PR Context
- Branch: <branch name> (base: <base branch>)
- Commits: <commit list>
- Files changed: <file list with stats>
- Tech stack: <stack>
- PR description: <description if available, otherwise "No PR description — infer
  intent from commit messages and code">

## Changed Files
<list of files changed with line counts>

## Your Task
1. Claim your task via TaskUpdate (set to in_progress)
2. Read every file modified in this PR. Read the full file, not just the diff —
   you need context to understand how changes interact with existing code.
3. For each changed file, also read the code that calls or imports it, to
   understand downstream effects.
4. Apply your specialist lens: evaluate every change through your specific focus area
5. Write findings to: .reports/pr-review/PR-REVIEW-<your-slug>.md
6. Use the findings format: Must Fix / Should Fix / Nit / Looks Good
7. Be specific — include file paths, line numbers, and concrete suggestions
8. When complete, mark your task as completed

## Important
- Read the ACTUAL code, not just filenames. Open every changed file.
- Consider how changes interact with existing code — not just the diff in isolation.
- Include "Looks Good" observations — what the PR does well.
- If you find something outside your specialty, note it briefly.
- Be constructive, not pedantic. Prioritize issues that matter.
- For "Must Fix" items, the bar is: this will cause a bug, outage, security issue,
  or data loss if merged. Do not inflate this category.
```

## Customization

### Adjusting Scope

- For small PRs (< 5 files), run only 2 specialists: Correctness + Maintainability (broadest coverage). The lead synthesizes directly.
- For performance-critical changes, add a dedicated **Database Reviewer** focused on query plans, index coverage, and migration safety.
- For API changes, add an **API Contract Reviewer** that checks backwards compatibility, response shape consistency, and deprecation handling.
- For UI changes, add an **Accessibility Reviewer** for WCAG compliance, keyboard navigation, screen reader support, and focus management.

### Quick Mode

For a fast review, skip the full team and spawn only 2 agents: Correctness Reviewer + Domain Logic Reviewer. The lead synthesizes instead of a dedicated synthesizer. Appropriate for PRs under 200 lines changed.

### Posting to GitHub

After synthesis, optionally post the review to the PR:

```bash
# As a comment
gh pr review <PR-number> --comment --body "$(cat PR-REVIEW.md)"

# As a formal review with verdict
gh pr review <PR-number> --request-changes --body "$(cat PR-REVIEW.md)"
gh pr review <PR-number> --approve --body "$(cat PR-REVIEW.md)"
```

Ask the user before posting — they may want to edit the synthesis first.

### Large PRs

For PRs touching 30+ files, scope each specialist to a subset of changed files rather than the full diff. Divide by domain (e.g., Performance Reviewer gets the database/API layer, Maintainability Reviewer gets the UI/utility layer). Note the division in each agent's prompt so the synthesizer knows the coverage.

## Model Selection

| Role | Recommended Model | Rationale |
|------|-------------------|-----------|
| Correctness Reviewer | `opus` | Bug detection requires tracing complex code paths and reasoning about edge cases |
| Performance Reviewer | `sonnet` | Pattern-matching for known anti-patterns (N+1, missing indexes, O(n^2)) |
| Maintainability Reviewer | `sonnet` | Convention checking and pattern comparison are systematic |
| Domain Logic Reviewer | `opus` | Validating intent-vs-implementation requires nuanced judgment |
| Senior Synthesizer | `opus` | Must reason across all reports, calibrate severity, and determine verdict |

For Quick Mode (2 agents), use `opus` for both since you're relying on fewer reviewers to catch more.

## Skill Chains

| Chain | Skills (in order) | Use when |
|-------|-------------------|----------|
| **Full Pre-PR Pipeline** | `sanity-check` → implement improvements → `pr-review-swarm` | Maximum pre-merge quality: shape the code, then verify correctness |
| **Secure PR Review** | `pr-review-swarm` + `deep-audit` (scoped to changed files, parallel) | Security-sensitive PR |
| **Review Then Fix** | `pr-review-swarm` → `competitive-swarm` (fix Must Fix items with different approaches) | PR has complex issues with multiple valid fixes |
| **Thorough Feature Review** | `pr-review-swarm` + `performance-profiler` (scoped to changed files, parallel) | Large feature PR touching database queries or hot paths |
| **Review → Regression** | `pr-review-swarm` → `braintrust-eval` (add regression cases for findings) | PR modifies LLM-powered features — lock in correctness with evals |

When running alongside `deep-audit`, deduplicate at the end — the Correctness Reviewer and OWASP Expert may flag the same auth gap from different angles.

## Tips

- Run `deep-audit` alongside this skill on security-sensitive PRs — they complement each other (this catches bugs and design issues, deep-audit catches security vulnerabilities).
- The Domain Logic Reviewer is most effective when a PR description exists. If there's no description, ask the user to explain the intent before spawning.
- For branches with many commits, `git log --oneline main..HEAD` may reveal fixup commits — this context helps distinguish intentional final state from work-in-progress artifacts.
- "Must Fix" should be reserved for genuine merge-blockers. Over-using it erodes trust in the review.
- "Looks Good" sections matter — purely negative reviews are demoralizing and miss the chance to reinforce good patterns. Each specialist should identify at least one thing done well.
- If the synthesizer finds the same class of issue across multiple specialists, elevate it as a systemic pattern — patterns indicate a missing convention, not just individual mistakes.
- The Correctness Reviewer is highest priority for PRs that touch auth, payment, or data mutation. If time-constrained, run just that agent.
