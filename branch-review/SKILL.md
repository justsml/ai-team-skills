---
name: branch-review
description: Combined pre-PR shape check and merge-ready review for a branch or PR. Use when the user asks for a code review, PR review, sanity check, 10x engineer perspective, how to split a PR, or whether changes are ready to merge.
---

# Change Review

Use this skill to evaluate a branch or PR. Pick a mode, scope the diff, run specialists in parallel, and synthesize a single actionable report.

## Modes (pick one)
- Shape: 10x Engineer + Code Organizer + PR Splitter. Use for "sanity check", "10x", or "how should I split".
- Merge: Correctness + Performance + Maintainability + Domain Logic. Use for "review", "ready to merge", "what did I miss".
- Full: Run both for large or risky changes.

## Workflow
1. Scope changes
   - `git branch --show-current`
   - `git merge-base main HEAD`
   - `git log --oneline main..HEAD`
   - `git diff --stat main...HEAD`
   - `git diff main...HEAD`
   - `gh pr view --json title,body 2>/dev/null`
2. Identify tech stack and key directories.
3. Decide mode
   - < 100 LOC: Merge only, skip PR Splitter.
   - 100-500 LOC: Shape or Merge based on user request.
   - > 500 LOC: Full; scope 10x + Organizer to most-changed files.
4. Create output dir `.reports/branch-review/` (add `.reports/` to `.gitignore` if missing).
5. Create team: `TeamCreate: team_name = "branch-review"`.
6. Create tasks per specialist and run them in parallel.
7. Run Synthesizer after specialists finish; write `.reports/branch-review/branch-review.md`.

## Specialists
### 10x Engineer (Shape)
- Look for simpler alternatives, architecture fit, under/over-engineering, and naming clarity.

### Code Organizer (Shape)
- Check file placement, module boundaries, co-location, and naming consistency.

### PR Splitter (Shape)
- Build a dependency graph and propose split strategies with merge order.

### Correctness (Merge)
- Trace code paths, edge cases, error handling, and regression risk.
- Verify errors are both logged and handled — no swallowed exceptions or silent failures.
- Confirm new code paths have unit tests covering happy path and edge cases.

### Performance (Merge)
- Check algorithmic complexity, I/O patterns, missing batching, and hot paths.
- Flag sequential awaits on independent async work — use `Promise.all` or equivalent.
- Suggest omitting, splitting, or refactoring code that adds unnecessary complexity.

### Maintainability (Merge)
- Review naming clarity, abstraction level, consistency, and test coverage.
- Prefer minimal typing: reuse, derive, or rely on implicit types over redundant annotations.
- Verify new dependencies are justified and don't introduce unnecessary bloat.
- Ensure public APIs have appropriate documentation and comments.

### Domain Logic (Merge)
- Verify spec alignment, auth/permission checks, data integrity, and UX edge cases.
- Higher-risk features should use established Feature Flagging systems for controlled rollout. (Amplitude, LaunchDarkly, Env Vars, etc.)
- Check for missing translation keys — all user-facing text must be properly localized.
- Confirm changes integrate cleanly with the existing codebase without breaking existing functionality.

### Synthesizer
- Deduplicate findings, set verdict, and choose a split strategy if needed.

## Findings Format
Use one file per specialist in `.reports/branch-review/` as `CR-<specialist>.md`.

```markdown
# <Specialist> Findings

**Branch:** <branch>
**Scope:** <files or dirs>

## Top issues
1. [SEV-01] <Title>
- File: `path:line`
- Why: <impact>
- Fix: <action>

## Other notes
- <observations>

## Looks good
- <what works well>
```

Synthesizer output: `.reports/branch-review/branch-review.md`

```markdown
# Change Review - <Branch>

**Mode:** <Shape | Merge | Full>
**Files changed:** <count>
**Lines:** +<add> / -<del>

## Verdict: <Ready | Needs Work | Blocked>

## Do this before merge
1. <highest priority fix>
2. <next fix>
3. <next fix>

## Split recommendation (if Shape/Full)
<strategy and merge order>

## 10x alternative (if Shape/Full)
<brief "what would I do differently" summary>
```
