---
name: sanity-check
description: Pre-PR collaborative review — get a 10x engineer perspective, explore code organization alternatives, generate PR decomposition strategies (fan-out, stacked, blended), and draft follow-up issues via MCP ticket trackers.
---

# Sanity Check

A pre-PR collaborative review that asks the questions you'd ask yourself after sleeping on it. Before you open the PR, a small team of specialists looks at your branch and tells you: how a senior engineer would approach it differently, whether the code is organized well, and how to slice the changeset into reviewable PRs.

This is not a bug-finder. It's a shape-checker. "Is this the right structure to ship?"

## When to Use

- Branch is ready (or nearly ready) and you want a gut-check before opening a PR
- Changeset touches 5+ files and you're unsure how to decompose it for review
- You want a "what would a 10x engineer do?" perspective on your approach
- Code works but you suspect the file organization could be cleaner
- You're about to open a large PR and want to split it into a reviewable sequence
- User says "sanity check", "pre-PR review", "how should I split this", or "is this the right shape"

## When NOT to Use

- Branch has < 3 files changed and < 50 lines — just open the PR
- You want bug-finding or correctness review — use `pr-review-swarm` instead
- You want security analysis — use `deep-audit` instead
- The change is a single-file bug fix, config change, or typo — no decomposition needed
- You haven't started coding yet — this reviews existing changes, not plans
- The branch is already merged — this is a pre-PR skill

## The Specialists

Three specialists work in parallel, each looking at the same changeset through a different lens. A synthesizer combines their findings and drafts actionable issues.

### 1. The 10x Engineer

**Focus:** Would a senior/staff engineer approach this differently? Not "is this correct?" but "is this the best shape?"

**Reviews for:**

- **Simpler alternatives** — Is there a fundamentally simpler way to achieve the same result? Could 200 lines become 40 by using an existing library, framework feature, or codebase utility that was overlooked?
- **Existing abstractions ignored** — Does the codebase already have patterns, helpers, or conventions that do what this code is manually reimplementing? Check for duplicated logic, reinvented wheels, or missed base classes.
- **Over-engineering** — Unnecessary indirection, premature abstraction, interfaces with one implementation, config-driven behavior that will never be reconfigured, patterns imported from other codebases that don't fit here.
- **Under-engineering** — Inline logic that should be extracted because it will clearly be needed elsewhere. Missing error boundaries. Shared state without a clear owner. Technical shortcuts that will become blockers within weeks.
- **Architecture fit** — Does this change work _with_ the existing architecture or fight against it? Would a staff engineer suggest a different integration point, a different module boundary, or a different data flow?
- **Naming and mental model** — Do the names (files, functions, variables, types) communicate the right mental model? Would a new team member understand the intent from the names alone?
- **Future-proofing vs. YAGNI** — Is this change flexible where it needs to be and rigid where it should be? The 10x engineer avoids both "this will never need to change" and "this might need to handle 47 future cases."
- **What would I do differently?** — The single most valuable output. A brief, opinionated sketch of how a senior engineer would approach the same problem if starting from scratch.

**Methodology:** Reads the full changeset and the surrounding codebase context (imports, callers, related modules). Thinks about the problem being solved, not just the solution presented. Asks: "If I were pairing with this engineer, what would I suggest?" Produces concrete alternatives, not vague advice.

---

### 2. The Code Organizer

**Focus:** File structure, module boundaries, co-location, and naming. Is the code in the right place?

**Reviews for:**

- **File placement** — Are new files in the right directory? Do they follow the repo's existing organizational patterns? Would someone looking for this code find it where it lives?
- **Module boundary violations** — Does code in module A reach into module B's internals? Are there imports that cross boundaries that shouldn't be crossed? Is shared code actually shared (in a shared location) or duplicated?
- **Co-location opportunities** — Are related things close together? A component and its styles, a route and its handler, a type and its validator — are these co-located or scattered?
- **Separation opportunities** — Are unrelated things tangled together? A utility function buried in a component file, business logic in a UI handler, configuration mixed with implementation.
- **Naming consistency** — Do file names, export names, and directory names follow the repo's conventions? Are there competing naming patterns introduced?
- **Index files and barrel exports** — Are re-exports needed? Are barrel files being used where they help (public API) and avoided where they hurt (tree-shaking, circular deps)?
- **Alternative layouts** — Concrete suggestions for how the same code could be organized differently. Not "you should reorganize this" but "here are 2-3 specific alternative structures and the trade-offs of each."

**Methodology:** Maps the directory structure of changed files. Reads existing organizational patterns in the repo (how are similar features organized?). Produces specific file-move and file-rename suggestions, not abstract principles. Every suggestion includes a "before → after" layout.

---

### 3. The PR Splitter

**Focus:** How to decompose this changeset into reviewable, shippable units. The most valuable specialist for large branches.

**Reviews for:**

- **Dependency graph** — Which changed files depend on which? Which changes are foundational (depended on by others) and which are leaf changes (depend on foundations but nothing depends on them)?
- **Independent clusters** — Groups of files that changed together but are independent of other groups. These are natural candidates for parallel PRs.
- **Sequencing requirements** — Which changes MUST come before others? Database migrations before code that uses new columns. Types/interfaces before implementations. Shared utilities before consumers.
- **Reviewability** — Each proposed PR should be independently reviewable. A reviewer should be able to understand what a PR does without needing to see the others. Each PR should have a clear, one-sentence purpose.
- **Shippability** — Each PR should leave the codebase in a working state. No PR should introduce dead code, unused imports, or half-finished features without feature flags.
- **Risk isolation** — High-risk changes (auth, data mutations, migrations) should be in their own PR so they get focused review. Don't bundle risky changes with safe ones.

**Produces multiple decomposition strategies:**

#### Strategy A: Fan-Out (Parallel Independent PRs)

```
main ← PR 1 (database migration)
main ← PR 2 (new utility functions)
main ← PR 3 (UI component)
main ← PR 4 (API endpoint)
```

All PRs can merge in any order. No dependencies between them. Each reviewer sees a small, focused changeset. Best when changes are naturally independent.

#### Strategy B: Stacked Branches (Sequential)

```
main ← PR 1 (types + schema) ← PR 2 (business logic) ← PR 3 (API + UI)
```

Each PR builds on the previous. Review in order, merge in order. Best when there's a clear foundational → consumer dependency chain.

#### Strategy C: Blended (Fan-Out + Stack)

```
main ← PR 1 (foundation: types, schema, migration)
  then fan out:
    PR 1 ← PR 2a (API endpoint)
    PR 1 ← PR 2b (UI component)
    PR 1 ← PR 2c (background worker)
```

One foundational PR, then parallel PRs that build on it. Best for features with a shared base and independent consumers. This is usually the right answer for medium-to-large features.

**For each proposed strategy, the Splitter provides:**

- A diagram showing the branch/PR relationships
- Files in each PR with line counts
- One-sentence description of each PR's purpose
- Merge order (strict order for stacked, any order for fan-out sections)
- Estimated review complexity per PR (small / medium / large)

**Methodology:** Builds the file dependency graph from imports and usage. Identifies natural seams. Tests each proposed split against: "Can each PR be reviewed independently? Does each leave the repo green? Is any single PR > 400 lines?" Prefers fan-out where possible. Falls back to stacking only when dependencies require it. Always suggests at least 2 strategies.

---

### 4. The Sanity Synthesizer

**Focus:** Combine all findings into a single actionable report, then draft follow-up issues.

**Independent assessment (done BEFORE reading specialist reports):**

- **Overall shape assessment** — Is this changeset a coherent unit of work, or is it actually 2-3 unrelated changes bundled together? Does the branch tell a clear story?
- **Commit hygiene** — Do the commit messages tell the story of the change? Are there fixup commits that should be squashed? Is the commit history helpful for future archaeologists?
- **Missing pieces** — Are there obvious gaps? Tests not written, migrations not created, documentation not updated, types not exported?

**Synthesis (done AFTER reading specialist reports):**

- Merges and deduplicates findings across all three specialists
- Resolves conflicts (10x Engineer says "simplify" but Organizer says "add a new module" — which is right?)
- Selects the best PR decomposition strategy (or blends multiple) with rationale
- Identifies the single highest-leverage improvement: "If you do one thing before opening the PR, do this"
- Drafts follow-up issues for work that should happen but doesn't block this PR

**Methodology:** Reads all specialist reports. Asks: "What's the minimum set of changes that would make this branch excellent?" Avoids suggesting everything — focuses on the 3-5 most impactful improvements. Presents a clear recommendation, not a menu of options.

---

## Workflow

### Step 1: Scope the Changes

Before spawning the team, understand the branch:

1. **Identify the branch and base:**
   ```bash
   git branch --show-current
   git merge-base main HEAD
   ```

2. **Get the commit history:**
   ```bash
   git log --oneline main..HEAD
   ```

3. **Get the changeset stats:**
   ```bash
   git diff --stat main...HEAD
   ```

4. **Get the full diff for context:**
   ```bash
   git diff main...HEAD
   ```

5. **Identify the tech stack and key directories affected**

6. **Check if a PR already exists** (if so, include its description):
   ```bash
   gh pr view --json title,body 2>/dev/null
   ```

### Step 2: Create the Team

```
TeamCreate: team_name = "sanity-check"
```

Create one task per specialist:

```
TaskCreate: "10x Engineer Review"
TaskCreate: "Code Organization Review"
TaskCreate: "PR Decomposition Analysis"
TaskCreate: "Sanity Synthesis & Issue Drafting"  ← blocked by the above 3
```

### Step 3: Spawn Specialists in Parallel

Launch 3 specialists simultaneously as `general-purpose` subagents with `team_name = "sanity-check"`. Each gets:

1. The scoping context from Step 1 (diff stats, commit list, affected files, tech stack)
2. Their specialist role description (copy from the relevant section above)
3. The full diff or a summary for large branches
4. Where to write findings (a unique file per specialist)
5. The findings format (see below)

The 4th agent (Synthesizer) is spawned after the first 3 complete, or at the same time with a `blockedBy` dependency.

### Handling Failures

- **Agent timeout or empty report:** The synthesizer should note the gap and proceed with available findings. A sanity check with 2 of 3 specialists is still valuable.
- **Small changeset (< 100 lines):** The PR Splitter may correctly determine that splitting isn't needed. That's a valid finding — "this changeset is already the right size for a single PR."
- **Large changeset (500+ lines of diff):** Scope the 10x Engineer and Code Organizer to the most-changed files. The PR Splitter should still see the full changeset to recommend decomposition.
- **Contradictory recommendations:** The synthesizer resolves these explicitly. "The 10x Engineer suggests inlining this helper, but the Organizer suggests extracting it to a shared module. Given this repo's patterns, the Organizer is right because [reason]."

### Step 4: Findings Format

Create the output directory: `.reports/sanity-check/` (add `.reports/` to `.gitignore` if not already there).

Each specialist writes findings to `.reports/sanity-check/SANITY-<specialist>.md`.

**10x Engineer format:**

```markdown
# 10x Engineer Review

**Branch:** <branch name>
**Files Changed:** <count>

## What Would I Do Differently?
<The single most valuable section. A brief, opinionated sketch of how a senior
engineer would approach the same problem. Concrete, not abstract.>

## Simplification Opportunities
### [SIMPLE-01] <Title>
- **Files:** `path/to/file.ts`
- **Current approach:** <what the code does now>
- **Simpler alternative:** <what a senior engineer would do instead>
- **Why:** <trade-off explanation>

## Over-Engineering Flags
### [OVER-01] <Title>
- **File:** `path:line`
- **What:** <what's more complex than needed>
- **Suggestion:** <simpler alternative>

## Under-Engineering Flags
### [UNDER-01] <Title>
- **File:** `path:line`
- **What:** <what's too simple / will become a problem>
- **Suggestion:** <what to add>

## Architecture Fit
<How well this change integrates with the existing codebase architecture>

## Looks Good
<What the branch does well — good decisions, clean patterns>
```

**Code Organizer format:**

```markdown
# Code Organization Review

**Branch:** <branch name>

## File Placement Issues
### [ORG-01] <Title>
- **Current:** `path/to/file.ts`
- **Suggested:** `better/path/to/file.ts`
- **Reason:** <why this location is better>

## Alternative Layouts

### Layout A: <Name>
```
<directory tree showing proposed structure>
```
**Trade-offs:** <pros and cons>

### Layout B: <Name>
```
<directory tree showing proposed structure>
```
**Trade-offs:** <pros and cons>

## Module Boundary Issues
### [BOUNDARY-01] <Title>
- **File:** `path:line`
- **Issue:** <what boundary is violated>
- **Fix:** <where this code should live>

## Co-location Opportunities
<Related files that should be closer together>

## Naming Suggestions
<Specific rename suggestions for files, functions, or variables>
```

**PR Splitter format:**

```markdown
# PR Decomposition Analysis

**Branch:** <branch name>
**Total Files Changed:** <count>
**Total Lines Changed:** +<add> / -<del>

## Dependency Graph
<Which files depend on which. ASCII or markdown diagram.>

## Strategy A: <Name> (<Fan-Out / Stacked / Blended>)

```
<branch diagram>
```

### PR 1: <one-sentence purpose>
- Files: <list with line counts>
- Review size: <small / medium / large>
- Risk: <low / medium / high>

### PR 2: <one-sentence purpose>
- Files: <list with line counts>
- Depends on: PR 1 (or "none — independent")
- Review size: <small / medium / large>
- Risk: <low / medium / high>

### Merge order: <description>

---

## Strategy B: <Name> (<Fan-Out / Stacked / Blended>)
<same format>

---

## Recommendation
<Which strategy to use and why. Consider: review burden, risk isolation,
merge conflict potential, CI efficiency.>

## Single-PR Option
<If the changeset is small enough, explicitly say "this is fine as one PR"
with the rationale. Don't split for the sake of splitting.>
```

### Step 5: Synthesis

The Synthesizer reads all findings files, then produces `.reports/sanity-check/SANITY-CHECK.md`:

```markdown
# Sanity Check — <Branch Name>

**Branch:** <branch>
**Base:** <base branch>
**Commits:** <count>
**Files Changed:** <count>
**Lines Changed:** +<add> / -<del>

## Overall Shape: <Good / Needs Work / Rethink>

<2-4 sentence assessment. Is this branch in good shape to become a PR?
What's the single highest-leverage improvement?>

## Do This Before Opening the PR

<The 3-5 most impactful improvements, in priority order.
Each should be specific and actionable.>

### 1. <Highest priority change>
- **From:** <10x Engineer / Organizer / Splitter>
- **What:** <specific action>
- **Why:** <impact>

### 2. ...

## PR Decomposition Recommendation

<Selected strategy from the PR Splitter, with any modifications
based on 10x Engineer or Organizer input.>

### Recommended Split
```
<branch diagram>
```

<For each PR in the sequence: title, one-sentence scope, files, review size>

## The 10x Version

<Summary of 10x Engineer's "what would I do differently" —
the aspirational version of this change>

## Code Organization

<Summary of Organizer's key suggestions, filtered to what's actionable now>

## Follow-Up Issues

<Issues to create after the PR(s) merge. NOT blockers —
things that improve the code but shouldn't delay shipping.>

### Issue 1: <Title>
- **Type:** <enhancement / tech-debt / follow-up>
- **Description:** <what to do>
- **Context:** <why this came up in the sanity check>

### Issue 2: ...

## Appendix: Specialist Reports
- [10x Engineer Review](./SANITY-10x.md)
- [Code Organization Review](./SANITY-organizer.md)
- [PR Decomposition Analysis](./SANITY-splitter.md)
```

### Step 6: Draft Issues

After presenting the synthesis, offer to create issues in the user's ticket tracker.

**Discover available ticket trackers:**

Use `ToolSearch` to find MCP tools for issue creation:

```
ToolSearch: "create issue"     → Linear, GitHub Issues, Jira, etc.
ToolSearch: "notion create"    → Notion databases
```

**Priority order for discovery:**

1. **Linear** — `mcp__claude_ai_Linear_2__create_issue` (check for team/project context)
2. **GitHub Issues** — `gh issue create` via Bash (always available in git repos)
3. **Notion** — `mcp__claude_ai_Notion__notion-create-pages`
4. **Other MCP tools** — search for any `create_issue` or `create_ticket` pattern

**Before creating any issues:**

1. Present the drafted issues to the user as a numbered list
2. Ask which to create and in which tracker
3. Ask for any title/description edits
4. Only then create the issues

**Issue template for Linear:**

```
Title: [Sanity Check] <Issue title>
Description: |
  ## Context
  Found during pre-PR sanity check of branch `<branch>`.

  ## What
  <description>

  ## Why
  <context from the sanity check finding>

  ## Suggested Approach
  <if applicable>
Labels: ["tech-debt"] or ["enhancement"]
Priority: 4 (Low) — these are follow-ups, not blockers
```

**Issue template for GitHub:**

```bash
gh issue create \
  --title "[Sanity Check] <title>" \
  --body "$(cat <<'EOF'
## Context
Found during pre-PR sanity check of branch `<branch>`.

## What
<description>

## Why
<context>
EOF
)" \
  --label "tech-debt"
```

### Step 7: Present & Clean Up

- Present the Overall Shape assessment and "Do This Before Opening the PR" list
- Show the PR decomposition recommendation with the branch diagram
- Present drafted follow-up issues and ask which to create
- If user wants to split the branch, offer to help create the branches using `git gtr` or manual git commands
- Clean up the team with `TeamDelete`
- Leave all files in `.reports/sanity-check/` for the user

## Agent Prompt Template

Use this when spawning each specialist:

```
You are a specialist on a pre-PR sanity check team. Your role is:

<ROLE NAME>

<Paste the full specialist description from the relevant section above>

## Branch Context
- Branch: <branch name> (base: <base branch>)
- Commits: <commit list>
- Files changed: <file list with stats>
- Tech stack: <stack>
- PR description: <description if available>

## Changed Files
<list of files changed with line counts>

## Your Task
1. Claim your task via TaskUpdate (set to in_progress)
2. Read the full diff: `git diff main...HEAD`
3. Read every changed file in full — not just the diff lines — to understand context
4. For key changed files, also read their importers/callers to understand impact
5. Apply your specialist lens to the entire changeset
6. Write findings to: .reports/sanity-check/SANITY-<your-slug>.md
7. Be specific and actionable — every suggestion should be something the
   engineer can do in 5-30 minutes
8. When complete, mark your task as completed

## Important
- This is a PRE-PR review. The goal is to improve the branch before it becomes
  a PR, not to find bugs. Focus on shape, organization, and strategy.
- Be opinionated. "It depends" is not useful. Make a recommendation and explain
  the trade-off.
- Every suggestion must be concrete. "Consider refactoring" is useless.
  "Move `parseConfig` from `utils/helpers.ts` to `config/parser.ts` because
  it's only used by the config module" is actionable.
- If the branch is in great shape, say so. Don't manufacture issues.
- Respect the repo's existing conventions even if you'd prefer different ones.
```

## Customization

### Adjusting Scope

- For small branches (< 5 files), skip the PR Splitter — splitting isn't needed. Run only 10x Engineer + Code Organizer.
- For very large branches (30+ files), prioritize the PR Splitter — decomposition is the highest-value output. Scope the other specialists to the most-changed files.
- For branches that are clearly one feature, focus the 10x Engineer on approach alternatives rather than splitting.

### Quick Mode

Run only the PR Splitter. Most useful when you already know the code is fine but need help decomposing a large branch for review. Skip the team setup — just spawn one agent.

### Adding Specialists

For specific contexts, consider adding:

- **Migration Safety Reviewer** — For branches with database migrations: rollback safety, data backfill, zero-downtime deployment concerns.
- **API Surface Reviewer** — For branches that change public APIs: backwards compatibility, versioning, documentation.
- **Test Strategy Advisor** — For branches with significant new logic but sparse tests: which tests to write, what coverage matters.

### Solo Mode (No Team)

For branches under 100 lines changed, skip the team entirely. The lead agent does all three reviews inline:

1. Quick 10x take ("anything I'd do differently?")
2. Quick file organization scan
3. "Split or ship as one PR?"

This takes 2 minutes instead of 10.

## Model Selection

| Role | Recommended Model | Rationale |
|------|-------------------|-----------|
| 10x Engineer | `opus` | Requires creative reasoning about alternative approaches and architecture judgment |
| Code Organizer | `sonnet` | Pattern-matching against existing conventions is systematic |
| PR Splitter | `sonnet` | Dependency analysis and clustering are methodical tasks |
| Sanity Synthesizer | `opus` | Must resolve conflicts between specialists and make prioritization judgments |

The 10x Engineer is the most judgment-heavy role — `opus` is worth it. The Organizer and Splitter do more systematic analysis where `sonnet` is efficient and accurate.

## Skill Chains

| Chain | Skills (in order) | Use when |
|-------|-------------------|----------|
| **Full Pre-PR Pipeline** | `sanity-check` → implement improvements → `pr-review-swarm` | Maximum quality: shape the code, then verify correctness |
| **Sanity + Security** | `sanity-check` + `deep-audit` (parallel, scoped to branch) | Security-sensitive feature: check shape AND vulnerabilities before PR |
| **Sanity → Competitive Fix** | `sanity-check` → `competitive-swarm` (implement 10x Engineer's alternative) | The 10x alternative is compelling enough to try side-by-side |
| **Architecture → Sanity** | `architecture-forensics` → `sanity-check` | Unfamiliar codebase: understand the architecture, then sanity-check your changes against it |
| **Sanity → Split → Review** | `sanity-check` (get split plan) → create branches → `pr-review-swarm` (per branch) | Large feature: decompose first, then review each piece |
| **Sanity → Regression** | `sanity-check` → `braintrust-eval` (add regression cases for LLM behaviors) | Branch modifies LLM features — lock in behavior with evals before merging |

## Tips

- **Run this BEFORE you think the code is done.** The best time for a sanity check is when you're 80% done — early enough to change direction, late enough to have real code to review.
- **The PR Splitter is the highest-ROI specialist.** A well-decomposed branch gets reviewed faster, merges cleaner, and has fewer conflicts. If you run only one specialist, run the Splitter.
- **"What would I do differently?" is the 10x Engineer's most valuable output.** It's not criticism — it's a senior engineer pairing with you. Read it as "here's another way to think about this."
- **Don't split for the sake of splitting.** If 5 files and 150 lines tell a coherent story, ship it as one PR. The Splitter should say so.
- **Fan-out > stacked when possible.** Independent PRs can merge in any order, don't create merge conflicts with each other, and let different reviewers work in parallel. Only stack when dependencies require it.
- **Follow-up issues are NOT blockers.** They're improvements that came up during the sanity check but shouldn't delay shipping. Create them, tag them, and move on.
- **Run `architecture-forensics` first if the codebase is unfamiliar.** Understanding the existing architecture makes all three specialists more effective.
- **This skill complements `pr-review-swarm`, it doesn't replace it.** Sanity-check improves the shape before PR; pr-review-swarm verifies correctness after.
