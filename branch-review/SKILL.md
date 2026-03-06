---
name: branch-review
description: Combined pre-PR shape check and merge-ready review for a branch or PR. Use when the user asks for a code review, PR review, sanity check, 10x engineer perspective, how to split a PR, or whether changes are ready to merge.
---

# Change Review

Use this skill to evaluate a branch or PR. Pick a mode, scope the diff, run specialists in parallel, and synthesize a single actionable report.

## Prerequisites

Before running, verify the environment has the tools this skill depends on. Run these checks at the start and report any that fail — don't silently skip them.

**Required:**
- `git` — must be in a git repository with a clean working tree (or at least committed changes to review)
- `gh` (GitHub CLI) — needed for PR metadata, posting reviews, and checking CI status
  - Install: `brew install gh` (macOS) / `sudo apt install gh` (Linux) / `winget install GitHub.cli` (Windows)
  - Auth: `gh auth login` — follow the prompts to authenticate with GitHub
  - Verify: `gh auth status` should show "Logged in to github.com"

**Recommended:**
- `jq` — useful for parsing JSON output from `gh` and `git` commands
  - Install: `brew install jq` / `sudo apt install jq`

If `gh` is not installed or not authenticated, warn the user and skip GitHub-dependent steps (PR description fetching, review posting). The core review still works with just `git`.

## Modes (pick one)
- Shape: 10x Engineer + Code Organizer + PR Splitter + Ruthless Reviewer. Use for "sanity check", "10x", or "how should I split".
- Merge: Correctness + Performance + Maintainability + Domain Logic. Use for "review", "ready to merge", "what did I miss".
- Full: Run both for large or risky changes.
- Solo: For branches < 100 LOC, skip the team. The lead agent does all reviews inline: quick 10x take, file organization scan, ruthless deletion pass, and "split or ship as one PR?" — takes minutes instead of a full team run.

## Workflow
1. Gather context from all available sources
   Try each source below. Use what's available, skip what isn't. Record which sources you found in the review header so readers know what informed the review.

   **Git (always available):**
   - `git branch --show-current`
   - `git merge-base main HEAD`
   - `git log --oneline main..HEAD`
   - `git diff --stat main...HEAD`
   - `git diff main...HEAD`

   **GitHub PR (via `gh` CLI):**
   - `gh pr view --json title,body,labels,milestone,assignees,reviewRequests 2>/dev/null`
   - `gh pr checks 2>/dev/null` — CI status for context on known failures
   - `gh pr view --json comments --jq '.comments[].body' 2>/dev/null` — prior review comments

   **Linked tickets (via MCP or `gh`):**
   - Check the PR body and commit messages for ticket references (e.g. `PROJ-123`, `#456`, Jira/Linear/Shortcut URLs)
   - If an MCP server provides issue/ticket tools (e.g. Jira, Linear, GitHub Issues), use them to fetch the ticket description, acceptance criteria, and status
   - If no MCP ticket tool is available, try `gh issue view <number> --json title,body 2>/dev/null` for GitHub Issues
   - Ticket context helps the Domain Logic reviewer validate intent vs. implementation

   **Report which sources were used** at the top of the synthesizer output:
   ```
   **Sources:** git diff, gh pr (title, body, 2 comments), PROJ-123 (via Linear MCP)
   ```
   If a source was unavailable, note it so gaps are visible:
   ```
   **Sources:** git diff, gh pr (no PR found — reviewing branch only), no linked tickets found
   ```

2. Identify tech stack and key directories.
3. Decide mode
   - < 100 LOC and < 5 files: Solo mode — lead agent does all reviews inline, no team needed.
   - 100-500 LOC: Shape or Merge based on user request.
   - > 500 LOC: Full; scope 10x + Organizer + Ruthless Reviewer to most-changed files.
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
- Produce at least 2 decomposition strategies from: Fan-Out (parallel independent PRs), Stacked (sequential dependency chain), or Blended (shared foundation then fan-out).
- For each strategy: branch diagram, files per PR with line counts, one-sentence purpose, merge order, review size (small/medium/large), and risk level.
- If the changeset is small enough for one PR, say so explicitly — don't split for the sake of splitting.

### Ruthless Reviewer (Shape)
- Default position: every file, type, function, test, and abstraction must prove it deserves to exist. If it can't, it should be deleted or consolidated.
- **Unnecessary code** — Functions that wrap without adding value, abstractions with one consumer (inline them), "helper" files that help no one, dead code, unused exports, orphaned types, defensive code guarding impossible states.
- **Type bloat** — Custom types duplicating existing ones, intermediate shuttle types, overly specific types where a generic works, type aliases adding a name but no semantic value, redundant `Pick<>`/`Omit<>` wrappers.
- **Pointless tests** — Tests that only assert mocks return what they were told, tests re-testing framework behavior, tests with no meaningful assertions, snapshot tests no one reviews, integration tests that mock everything.
- **Over-engineering** — Premature abstractions, factory-of-a-factory, strategy pattern for 2 cases, plugin systems no one plugs into, generic utilities used once, stale feature flags.
- **Missed reuse** — Code reimplementing something in project dependencies, local utilities duplicating stdlib/lodash, thin wrapper hooks, hand-rolled validation when schema libraries exist.
- Each finding gets a verdict: `DELETE` | `INLINE` | `CONSOLIDATE` | `SIMPLIFY` | `REPLACE` | `KEEP` (with reluctance).
- End with a tally: lines deletable, files deletable, types eliminable, tests removable, estimated complexity reduction.

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
- Perform an independent assessment BEFORE reading specialist reports: overall shape (coherent unit of work or bundled unrelated changes?), commit hygiene (do messages tell the story? fixup commits to squash?), and missing pieces (tests, migrations, docs, type exports).
- Then read all specialist reports: deduplicate, resolve conflicts (e.g. Ruthless Reviewer says "delete" but 10x Engineer says "extract to shared module" — pick one with rationale), set verdict, and choose a split strategy if needed.
- Identify the single highest-leverage improvement: "If you do one thing before opening the PR, do this."
- Draft follow-up issues for work that improves the code but shouldn't block shipping.

## Model Selection

Default to `sonnet` for speed and cost. Use `opus` only where deeper reasoning justifies it.

| Role | Model | Rationale |
|------|-------|-----------|
| 10x Engineer | `sonnet` | Pattern recognition, not deep tracing |
| Code Organizer | `sonnet` | Structural analysis is systematic |
| PR Splitter | `sonnet` | Dependency graphs are mechanical |
| Ruthless Reviewer | `sonnet` | Deletion analysis is systematic pattern-matching against known bloat signals |
| Correctness | `opus` | Bug detection requires tracing complex code paths and edge-case reasoning |
| Performance | `sonnet` | Matching known anti-patterns (N+1, O(n²), missing parallelism) |
| Maintainability | `sonnet` | Convention checking is systematic |
| Domain Logic | `opus` | Validating intent vs. implementation requires nuanced judgment |
| Synthesizer | `opus` | Must reason across all reports, calibrate severity, and determine verdict |

**Extended context:** For large codebases or PRs touching many files, consider using extended-thinking models with 1M token context windows (e.g. `claude-sonnet-4-6` with `--max-context 1000000`). This lets specialists read full files instead of just diffs, which improves accuracy on integration and regression checks. Use when the diff + surrounding context exceeds ~100K tokens, or when the codebase has deep call chains that require tracing across many files.

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

**Mode:** <Shape | Merge | Full | Solo>
**Sources:** <list sources used — e.g. git diff, gh pr, PROJ-123 via Linear MCP>
**Files changed:** <count>
**Lines:** +<add> / -<del>

## Verdict: <Ready | Needs Work | Blocked>

## Do this before merge
1. <highest priority — "if you do one thing, do this">
2. <next fix>
3. <next fix>

## Commit hygiene (if Shape/Full)
<Do commits tell the story? Fixup commits to squash? Missing pieces?>

## Deletion candidates (if Shape/Full)
<Summary from Ruthless Reviewer: lines deletable, files deletable, types removable, tests removable, estimated complexity reduction>

## Split recommendation (if Shape/Full)
<Selected strategy (Fan-Out / Stacked / Blended) with branch diagram and merge order>

## 10x alternative (if Shape/Full)
<brief "what would I do differently" summary>

## Follow-up issues
<Issues to create after merge — improvements that shouldn't block shipping>
### Issue 1: <Title>
- **Type:** enhancement | tech-debt | follow-up
- **Description:** <what to do>
- **Context:** <why this came up>
```

### Step 8: Draft Issues (optional)

After presenting the synthesis, offer to create follow-up issues in the user's ticket tracker.

**Discover available ticket trackers** via `ToolSearch`:
- `ToolSearch: "create issue"` — Linear, GitHub Issues, Jira, etc.
- Fall back to `gh issue create` via Bash (always available in git repos)

**Before creating any issues:**
1. Present drafted issues as a numbered list
2. Ask which to create and in which tracker
3. Ask for title/description edits
4. Only then create the issues

### Monitoring & Status Updates

While agents are working, output status updates to the user every 1-5 minutes:

- Which agents have completed and which are still running
- Any early outputs from completed agents (read the files if available)
- Estimated progress based on task completion count

**Do NOT go silent while waiting for background agents.** The user should always know work is happening. Use `SendMessage` or direct text output to keep them informed. If all agents are still running and there's nothing new to report, a brief "Still waiting on X agents..." is sufficient.
