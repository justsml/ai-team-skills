---
name: tech-debt-inventory
description: Catalog and quantify technical debt using a parallel team of specialists — complexity analysis, dead code detection, dependency health, codebase inconsistencies, and test gap analysis — to produce a prioritized remediation roadmap.
---

# Tech Debt Inventory

Catalog and quantify technical debt across five dimensions using a parallel team of specialists, then synthesize into a prioritized remediation roadmap with effort estimates. Use this skill when the user wants to understand how much tech debt exists, what to clean up first, or how to plan a debt reduction initiative.

## When to Use

- User wants to understand how much tech debt exists in their codebase
- User asks "where should we focus cleanup?" or "what's the state of our code?"
- User is planning a tech debt sprint or cleanup initiative
- User wants to prioritize what to fix first
- User asks about code quality, code health, or codebase maintainability
- User says "tech debt inventory", "tech debt audit", or "debt assessment"

## When NOT to Use

- Codebase is < 1 month old or < 1000 LoC — not enough history to accumulate meaningful debt
- User wants security findings — use `deep-audit` instead (security issues aren't tech debt)
- User wants performance bottlenecks — use `performance-profiler` instead (slow code isn't debt, it's an optimization problem)
- The repo is a prototype, spike, or throwaway experiment — debt is irrelevant if it's disposable
- User is asking about a specific file or function — answer directly instead of orchestrating a team
- The codebase was recently inventoried and no significant work has landed since

## The Specialists

Each specialist is a `general-purpose` subagent focused on a distinct category of technical debt. They work in parallel, analyzing the same codebase independently, then findings are synthesized.

### 1. Complexity Analyst

**Focus:** Functions, files, and modules that are disproportionately complex — the code that everyone dreads touching.

**Reviews for:**
- **Long functions** — Functions over 50 lines (warning), over 100 lines (critical). Identifies what they're doing and whether they could be decomposed into focused pieces.
- **Deep nesting** — Functions with 4+ levels of nesting (if/for/try/if). Each level exponentially increases cognitive load and bug likelihood.
- **High cyclomatic complexity** — Functions with many branching paths: switch statements with 10+ cases, long if/else chains, multiple early returns with different logic in each.
- **Large files** — Files over 300 lines (warning), over 500 lines (critical). Usually a sign of a module doing too many things that should be separated.
- **God classes/modules** — Modules that handle HTTP, business logic, database access, and output formatting all in one place. Signs: many unrelated public methods, high import count, imported by everything.
- **Complex conditionals** — Boolean expressions with 3+ terms, nested ternaries, conditionals that require a comment to explain what they're checking.
- **Deep call chains** — Functions where understanding behavior requires tracing 5+ levels of function calls. Makes debugging and reasoning extremely difficult.
- **Parameter lists** — Functions with 5+ parameters (especially when several are booleans or have the same type). Indicates the function does too many things or needs a config/options object.

**Methodology:** Scans all source files. For each function, estimates complexity using nesting depth, line count, branch count, and parameter count. Produces a ranked "complexity hotspot" list. Top 10 most complex functions get detailed breakdowns with specific decomposition suggestions.

---

### 2. Dead Code Hunter

**Focus:** Code that exists but serves no purpose — candidates for deletion.

**Reviews for:**
- **Unused exports** — Functions, classes, constants, and types exported but never imported anywhere else in the codebase. Cross-references all export statements with all import statements.
- **Orphaned files** — Source files not imported by anything in the module graph. May be leftover from deleted features, renamed files, or abandoned experiments.
- **Unreachable code** — Code after unconditional returns/throws, branches that can never execute (conditions already eliminated by earlier checks), disabled feature flags with dead branches.
- **Commented-out code** — Code commented out rather than deleted. Lives forever, confuses readers, and is always available in git history anyway. Count total commented-out lines.
- **TODO/FIXME/HACK inventory** — Cataloged with age via git blame. TODOs older than 6 months are effectively dead — they'll never be done. HACK annotations flag known, acknowledged debt.
- **Unused dependencies** — Packages declared in package.json/requirements.txt/Gemfile that are never imported in source code.
- **Dead feature remnants** — Routes with no corresponding UI, database tables/columns with no reads, config options nothing checks, API endpoints nothing calls, feature flags permanently on or off.
- **Vestigial abstractions** — Interfaces with exactly one implementation, factory functions that always create the same type, middleware that passes through without acting, wrapper functions that add no behavior.

**Methodology:** Cross-references all exports with all imports to find unused exports. Identifies files not in any import chain starting from entry points. Greps for TODO/FIXME/HACK and enriches with git blame dates. Checks package manifests against actual imports. Estimates total deletable lines of code.

---

### 3. Dependency Auditor

**Focus:** Third-party dependency health and risk.

**Reviews for:**
- **Outdated major versions** — Dependencies 1+ major versions behind. Major bumps often contain breaking changes, security fixes, and deprecated API removals.
- **Unmaintained packages** — Dependencies with no release in 2+ years, archived repositories, or no response to issues. These are liabilities that only get worse.
- **Duplicate functionality** — Two or more packages that do the same thing (e.g., `moment` and `dayjs`, `lodash` and `underscore`, `axios` and `node-fetch`). Pick one and standardize.
- **Oversized dependencies** — Large packages imported for a single function (e.g., all of `lodash` for `_.get`). Could be replaced with a small utility or native API.
- **Dependency depth risk** — Packages with deep transitive dependency trees. More dependencies = more supply chain risk, more breakage surface, and larger install/bundle size.
- **Missing lockfile** — No lockfile, or lockfile out of sync with manifest. Leads to "works on my machine" inconsistencies.
- **Pinning issues** — Version ranges too loose (accepting any major) or too tight (blocking security patches). Missing resolution overrides for known transitive vulnerabilities.
- **Framework version** — Is the core framework (React, Next.js, Rails, Django, etc.) current? Are plugins/extensions compatible with the current framework version? How far behind is it?

**Methodology:** Reads package manifests and lockfiles. Cross-references actual imports against declared dependencies to find unused deps. Checks for version currency. Identifies the dependency tree shape and highlights the heaviest/deepest subtrees.

---

### 4. Inconsistency Detector

**Focus:** Same concept implemented multiple different ways — the entropy of a growing codebase.

**Reviews for:**
- **Naming convention inconsistencies** — camelCase vs snake_case in the same layer, inconsistent file naming (PascalCase components mixed with kebab-case), abbreviation inconsistencies (btn vs button, usr vs user, msg vs message).
- **Error handling inconsistencies** — Some modules throw exceptions, some return null, some return error objects, some use Result types. No consistent strategy means every call site has to guess.
- **API response format inconsistencies** — Different endpoints return errors differently, use different envelope shapes, inconsistent pagination, mixed casing in JSON response fields.
- **Data fetching inconsistencies** — Some components fetch in useEffect, some use React Query, some use SWR, some use getServerSideProps. Multiple strategies for the same problem.
- **Configuration inconsistencies** — Some settings from env vars, some from config files, some hardcoded. Different modules read configuration in different ways.
- **Duplicated utilities** — Multiple implementations of the same helper: date formatting, string sanitization, array grouping, URL building. Often slightly different, causing subtle bugs when the "wrong" one is used.
- **Import style inconsistencies** — Relative imports mixed with path aliases, barrel files in some directories but not others, inconsistent use of index files.
- **Testing inconsistencies** — Different mocking strategies, inconsistent test file naming/structure, some modules with extensive tests and others with none, mixed test runners or assertion libraries.

**Methodology:** Samples code across all major directories and layers. For each cross-cutting concern (error handling, config, fetching, naming, etc.), catalogs every variant found. Produces a consistency score per concern. Identifies the dominant pattern vs. outliers. Estimates the effort to standardize each concern.

---

### 5. Test Gap Analyst

**Focus:** Where the test suite fails to protect against regressions — not just coverage percentage, but actual risk.

**Reviews for:**
- **Untested critical paths** — Authentication, authorization, payment processing, data mutation, user-facing workflows with no test coverage. These are the highest-risk gaps.
- **Module coverage distribution** — Which modules/directories have tests and which don't? Is coverage concentrated in easy-to-test utilities while complex business logic goes untested?
- **Missing edge case tests** — Functions that handle edge cases (empty inputs, null, boundary values, concurrent access) in their implementation but those edges aren't tested.
- **Happy-path-only testing** — Tests that only verify the success case. No tests for error handling, invalid input, timeout, rate limiting, or partial failure.
- **Missing integration tests** — Unit tests exist but nothing tests the interaction between modules. API -> service -> database flows only tested in isolation.
- **Test-to-code ratio by module** — Some modules may have 2:1 test-to-code ratio while others have 0. Uneven coverage indicates concentrated risk.
- **Assertion-free tests** — Tests that run code but don't assert anything meaningful — only `expect(result).toBeDefined()`, snapshot tests that auto-update without review, or tests with no assertions at all.
- **Flakiness indicators** — Tests with hardcoded timeouts, sleep/delay calls, `Date.now()` dependency, shared mutable state between tests, or order-dependent test execution.

**Methodology:** Identifies all test files and maps them to source files. Reads critical code paths (auth, data mutation, API handlers) and checks for corresponding tests. Examines test quality — not just existence but whether assertions are meaningful. Produces a risk-ranked gap list.

---

### 6. Debt Accountant (Synthesizer)

**Focus:** Independent cross-category assessment PLUS deduplication, quantification, and prioritization of all debt into an actionable roadmap.

**Independent review (done BEFORE reading specialist reports):**
- **Debt interaction effects** — Individual debt items often compound. A God module (complexity) that's also inconsistently tested (test gaps) AND uses an outdated dependency (dependency debt) is far worse than the sum of its parts. Look for these compounding clusters.
- **Velocity indicators** — Read recent git history for signals of debt friction: frequent reverts, long-lived branches, files that are modified in every PR (shotgun surgery), and commit messages containing "hack", "workaround", or "temporary".
- **Missing automation** — Is there a linter? A formatter? Pre-commit hooks? CI/CD? Type checking? Each missing automation layer is a category of debt that accumulates silently.

**Synthesis (done AFTER reading specialist reports):**
- For each debt item: estimates effort to remediate (hours/days), risk if ignored (probability and impact), and value if fixed (velocity improvement, bug reduction, onboarding speed).
- **What everyone else missed** — Reviews all specialist findings for debt that falls between categories. Asks: "what's slowing this team down that none of the specialists explicitly flagged?"
- Deduplicates across specialists (the Inconsistency Detector and Complexity Analyst may flag the same God module from different angles).
- Produces the unified inventory, dashboard, and prioritized roadmap.

**Methodology:** First does a brief independent assessment — checks git history for velocity signals and scans for missing automation. Then reads all specialist reports, deduplicates, quantifies, and produces the prioritized roadmap.

---

## Workflow

### Step 1: Scope the Inventory

Before spawning the team, understand the codebase:

1. **Read the codebase structure** — directory layout, tech stack, entry points
2. **Identify primary language(s) and framework(s)**
3. **Check for existing tooling** — linting config, test config, CI/CD pipeline, code quality tools
4. **Determine scope** — full repo or specific areas the user wants to focus on
5. **Understand the context** — is this for a cleanup sprint? A quarterly assessment? Planning for a major migration?

### Step 2: Create the Team

```
TeamCreate: team_name = "tech-debt-inventory"
```

Create one task per specialist:

```
TaskCreate: "Complexity Analysis"
TaskCreate: "Dead Code Detection"
TaskCreate: "Dependency Audit"
TaskCreate: "Inconsistency Detection"
TaskCreate: "Test Gap Analysis"
TaskCreate: "Debt Synthesis & Roadmap"  ← blocked by the above 5
```

### Step 3: Spawn Specialists in Parallel

Launch 5 specialists simultaneously as `general-purpose` subagents with `team_name = "tech-debt-inventory"`. Each gets:

1. The scoping context from Step 1 (tech stack, key directories, existing tooling)
2. Their specialist role description (copy from the relevant section above)
3. Key directories and files to analyze
4. Where to write findings (a unique file per specialist)
5. The findings format (see below)

The 6th agent (Debt Accountant) is spawned after the first 5 complete, or at the same time with a `blockedBy` dependency.

### Handling Failures

- **Agent timeout or empty report:** The Debt Accountant should note the uncovered category in the final inventory and recommend a follow-up. A partial inventory is still actionable — don't block synthesis because one specialist failed.
- **Contradictory findings:** When specialists disagree (e.g., Dead Code Hunter flags a function as unused but Test Gap Analyst references it as untested critical code), the Debt Accountant should investigate — it often reveals misunderstanding of the module graph.
- **Large codebase (500+ source files):** Scope each specialist to the most active/changing areas of the codebase (use `git log --format='%H' -- <dir> | wc -l` to find hot directories). Include scoping in each prompt so the Debt Accountant knows coverage.

### Step 4: Findings Format

Create the output directory: `.reports/tech-debt-inventory/` (add `.reports/` to `.gitignore` if not already there).

Each specialist writes findings to `.reports/tech-debt-inventory/TECH-DEBT-<specialist>.md`. Use this format:

```markdown
# <Specialist Name> — Tech Debt Findings

**Analyst:** <Role name>
**Repository:** <repo name>
**Scope:** <What was analyzed>
**Date:** <Date>

## Summary Statistics
<Specialist-specific metrics: e.g., "12 functions over 100 lines",
"3,200 lines of dead code", "8 outdated major dependencies">

## Critical Debt

### [CRITICAL-01] <Title>
- **Location:** `file/path.ts` (or module/package)
- **Category:** <e.g., God Module, Circular Dependency, Unmaintained Dependency>
- **Description:** <What the debt is>
- **Impact:** <How this slows development or risks reliability>
- **Evidence:** <Metrics, code snippets, dependency chains>
- **Estimated Effort:** <Hours or days to remediate>
- **Remediation:** <Specific approach>

## Significant Debt
### [SIGNIFICANT-01] ...

## Moderate Debt
### [MODERATE-01] ...

## Quick Wins
<Items that take < 1 hour to fix and measurably improve the codebase.
These are the most valuable output — list every one found.>

## Metrics Summary
<Specialist-specific metrics table or inventory>
```

**Severity definitions:**

| Severity | Meaning |
|----------|---------|
| **Critical** | Actively causing bugs, blocking development, or creating significant risk. Must address soon. |
| **Significant** | Meaningfully slowing development velocity or degrading reliability. Should address this quarter. |
| **Moderate** | Annoying but manageable. Address when working in the area. |
| **Quick Win** | Trivial to fix (< 1 hour), disproportionate improvement in code quality or clarity. |

### Step 5: Synthesis

The Debt Accountant reads all findings files, then produces the final `.reports/tech-debt-inventory/TECH-DEBT-INVENTORY.md`:

```markdown
# Tech Debt Inventory — <Repository Name>

**Repository:** <repo name>
**Date:** <date>
**Analysts:** Complexity, Dead Code, Dependencies, Inconsistencies, Test Gaps

## Executive Summary
<3-5 sentences: overall debt load, most concerning area, recommended focus>

## Debt Dashboard

| Category | Critical | Significant | Moderate | Quick Wins |
|----------|----------|-------------|----------|------------|
| Complexity | X | X | X | X |
| Dead Code | X | X | X | X |
| Dependencies | X | X | X | X |
| Inconsistencies | X | X | X | X |
| Test Gaps | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** |

## Estimated Deletable Code
<Total lines of dead, unreachable, and orphaned code that can be safely removed.
Less code = fewer bugs, faster onboarding, easier navigation.>

## Quick Wins (Do First)
<All items across all specialists that take < 1 hour. Ideal for a single-day
cleanup sprint. Sorted by effort (easiest first).>

| # | Category | Item | Effort | File(s) |
|---|----------|------|--------|---------|
| QW-01 | Dead Code | Remove unused export `formatLegacyDate` | 15 min | `utils/date.ts` |
| QW-02 | Dependency | Remove unused package `left-pad` | 5 min | `package.json` |
| QW-03 | ... | ... | ... | ... |

## Critical & Significant Debt (Ranked)
<Merged findings from all specialists, deduplicated, ranked by:
impact on velocity > risk if ignored > effort to fix>

### [D-01] <Title>
- **Found by:** <Specialist(s)>
- **Category:** <category>
- **Impact:** <description>
- **Effort:** <estimate>
- **Approach:** <remediation strategy>

## Remediation Roadmap

### Week 1: Quick Wins & Dead Code
<Achievable in a single cleanup sprint. Delete dead code, remove unused
dependencies, fix trivial inconsistencies. Produces visible improvement
with minimal risk.>

### Month 1: Critical Debt
<Address the highest-impact items. Decompose God modules, fix circular
dependencies, update critically outdated dependencies, add tests for
untested critical paths.>

### Quarter: Strategic Improvements
<Larger efforts: standardize competing patterns, fill systematic test gaps,
address architectural inconsistencies, complete stalled migrations.>

### Ongoing: Maintenance Habits
<Habits to adopt going forward: dependency update cadence, complexity
limits in CI, dead code detection in PR reviews, test coverage gates
on critical paths.>

## Positive Observations
<What the codebase does well. Areas with low debt. Good patterns that
should be preserved and expanded to the rest of the codebase.>

## Appendix: Specialist Reports
- [Complexity Analysis](./TECH-DEBT-complexity.md)
- [Dead Code Report](./TECH-DEBT-deadcode.md)
- [Dependency Audit](./TECH-DEBT-dependencies.md)
- [Inconsistency Report](./TECH-DEBT-inconsistencies.md)
- [Test Gap Analysis](./TECH-DEBT-testgaps.md)
```

### Step 6: Present & Clean Up

- Present the Executive Summary, Debt Dashboard, and Quick Wins to the user
- Highlight Quick Wins as immediate value — low effort, high visibility
- Present the Remediation Roadmap as the long-term plan
- Offer to start fixing specific items (especially Quick Wins or critical debt)
- Clean up the team with `TeamDelete`
- Leave all files in `.reports/tech-debt-inventory/` for the user

## Agent Prompt Template

Use this when spawning each specialist:

```
You are a technical debt analyst on a parallel assessment team. Your role is:

<ROLE NAME>

<Paste the full specialist description from the relevant section above>

## Codebase Context
- Tech stack: <stack>
- Key directories: <dirs>
- Existing CI/linting: <what exists>
- Scope: <full repo / specific dirs>

## Your Task
1. Claim your task via TaskUpdate (set to in_progress)
2. Systematically analyze the codebase through your specialist lens
3. For every finding: identify the location, assess severity, estimate
   remediation effort, and provide a specific approach to fix it
4. Write your findings to: .reports/tech-debt-inventory/TECH-DEBT-<your-slug>.md
5. Include summary statistics and metrics specific to your specialty
6. Identify Quick Wins (< 1 hour, clear positive impact) — these are
   the most valuable output
7. When complete, mark your task as completed

## Important
- Be quantitative. Count lines, functions, files. "Some functions are long"
  is useless; "14 functions exceed 100 lines, concentrated in src/services/"
  is actionable.
- Include evidence: metrics, code snippets, specific file paths.
- Estimate remediation effort for every finding — even a rough estimate
  helps prioritize.
- Quick Wins are the most valuable output — they let the team start making
  progress immediately.
- Not everything is debt. Only flag things that actually slow development
  or risk reliability. Working code that's slightly inelegant is not debt.
```

## Customization

### Adjusting Scope

- For large monorepos, scope each specialist to specific packages/services.
- For frontend-heavy codebases, add a **Bundle Debt Analyst** (unused CSS, oversized imports, missing code splitting, unoptimized assets, duplicate polyfills).
- For legacy codebases, add a **Migration Debt Analyst** (deprecated API usage, EOL runtime versions, upgrade blockers, stalled framework migrations).
- For small codebases (< 100 files), run only 3 specialists: Complexity + Dead Code + Test Gaps (most immediately actionable).

### Quick Mode

Spawn only the Dead Code Hunter and Complexity Analyst — they produce the most immediately actionable findings with the least ambiguity. The lead synthesizes directly.

### Adding Specialists

- **Bundle Debt Analyst** — For web apps: CSS/JS bundle bloat, unused styles, duplicate polyfills, missing tree-shaking, unoptimized images.
- **Migration Debt Analyst** — Deprecated API usage, EOL runtime/language versions, deprecated dependency features, stalled framework upgrades.
- **Documentation Debt Analyst** — Missing/outdated README sections, undocumented public APIs, stale architecture docs, missing onboarding instructions.
- **Infrastructure Debt Analyst** — Dockerfile anti-patterns, CI/CD pipeline issues, missing health checks, manual deployment steps, environment drift.

## Model Selection

| Role | Recommended Model | Rationale |
|------|-------------------|-----------|
| Complexity Analyst | `sonnet` | Mechanical metrics: line counts, nesting depth, parameter counts |
| Dead Code Hunter | `sonnet` | Cross-referencing imports/exports is systematic pattern matching |
| Dependency Auditor | `sonnet` | Version checking and manifest analysis are mechanical |
| Inconsistency Detector | `sonnet` | Sampling patterns across directories is systematic |
| Test Gap Analyst | `opus` | Judging whether assertions are meaningful and identifying risk requires nuance |
| Debt Accountant | `opus` | Prioritization, effort estimation, and roadmap creation require judgment |

This is the most cost-efficient skill to run — 4 of 5 specialists can use `sonnet` since their work is primarily mechanical counting and cross-referencing. Only the Test Gap Analyst and synthesizer benefit from `opus`.

## Skill Chains

| Chain | Skills (in order) | Use when |
|-------|-------------------|----------|
| **Full Health Check** | `architecture-forensics` → `tech-debt-inventory` | Map structure first, then quantify what's wrong within it |
| **Debt Sprint Planning** | `tech-debt-inventory` → `competitive-swarm` (fix top debt items with different approaches) | Planning a cleanup sprint and want to explore remediation options |
| **Quality + Security** | `tech-debt-inventory` + `deep-audit` (parallel) | Comprehensive codebase assessment covering both quality and security |
| **Quarterly Review** | `tech-debt-inventory` (run quarterly, compare reports over time) | Track debt trends and measure cleanup progress |

The Inconsistency Detector and `architecture-forensics`'s Pattern Detector overlap ~70% — if running both, scope the Inconsistency Detector to naming/formatting/duplication and let the Pattern Detector handle architectural patterns.

## Tips

- Run this skill quarterly — tech debt accumulates silently and periodic inventories keep it visible and manageable.
- Quick Wins are the gateway to debt cleanup. Start there to build momentum, demonstrate value, and get the team excited about maintenance work.
- The Dead Code Hunter's "total deletable LoC" metric is surprisingly motivating — teams love deleting code, and the number gives permission to do it.
- Pair with `competitive-swarm` for remediation: when fixing complex debt, explore multiple refactoring approaches (e.g., Minimalist vs. Best Practices decomposition of a God module).
- The Inconsistency Detector often reveals stalled migrations — half the codebase uses the old pattern, half uses the new pattern, and nobody finished the migration. This is valuable organizational signal.
- Don't try to fix everything. The roadmap is a prioritized guide, not a mandate. Focus on the debt that's actively slowing the team.
- Always include Positive Observations — it's important to know what's working so it doesn't regress during cleanup efforts.
- Pair with `architecture-forensics` for a complete codebase health picture: forensics maps the structure, this skill quantifies what's wrong within it.
