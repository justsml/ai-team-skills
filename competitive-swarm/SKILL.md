---
name: competitive-swarm
description: Orchestrate Claude Code teams where multiple agents tackle the same task with different priorities (e.g. minimal vs. best-practices vs. performance), then compare and select the best result. Supports competitive, collaborative, and hybrid modes.
---

# Competitive & Collaborative Claude Code Swarms

Orchestrate multi-agent teams where members approach the same problem through different lenses — then compare, critique, and synthesize the best result. Use this skill when the user wants a team/swarm, wants to explore multiple approaches in parallel, or wants agents competing or collaborating on a task.

## When to Use

- User asks for a "swarm", "team", "competitive", or "compare approaches" workflow
- User wants to see multiple implementations of the same feature side-by-side
- User wants agents with different expertise or priorities working in parallel
- User wants a bake-off, shootout, or A/B comparison of solutions
- User describes priorities like "minimal vs. robust" or "fast vs. readable"

## When NOT to Use

- Task has a single obvious correct solution with no meaningful trade-offs — just implement it
- Task is trivial (< 30 minutes for one agent) — team setup overhead exceeds the value of comparison
- User wants speed, not exploration — they need one good answer, not three
- The deliverable has no subjective dimension (e.g., "add this field to the database schema")
- User is asking for code review — use `pr-review-swarm` or `sanity-check` instead
- User is asking for analysis (security, performance, architecture) — use the dedicated analysis skill instead

## Modes

### Competitive Mode (Bake-Off)
Multiple agents solve the **same task** independently with different priority lenses. A judge agent (or the lead) compares outputs and selects/synthesizes the best result. Use when there's one clear deliverable and you want to explore the solution space.

### Collaborative Mode (Specialists)
Agents own **different parts** of the task based on their expertise. Work is divided, not duplicated. Use when the task has naturally separable concerns (e.g. API + UI + tests).

### Hybrid Mode (Compete Then Collaborate)
Start competitive on the core design decision, pick a winner, then fan out collaboratively to finish the implementation. Use for larger tasks where the initial approach matters most.

## Priority Lenses

Assign each agent a distinct priority lens that shapes how they make trade-off decisions. Below are reusable archetypes — pick 2-4 that create productive tension for the task at hand.

### Code Style & Philosophy
| Lens | Priority | Trade-off Bias |
|------|----------|----------------|
| **Minimalist** | Fewest lines, least abstraction, no dependencies | Favors deletion, inline code, simple control flow. Avoids helpers, wrappers, config. |
| **Best Practices** | Idiomatic patterns, established conventions, readability | Favors named abstractions, separation of concerns, consistent style. |
| **Repo Conventions** | Match the existing codebase exactly — its patterns, naming, structure | Studies the repo first. Mimics existing style even if imperfect. Consistency > correctness of style. |
| **Defensive** | Robust error handling, edge cases, validation, fail-safe defaults | Favors guard clauses, exhaustive checks, graceful degradation. Tolerates verbosity. |

### Performance & Efficiency
| Lens | Priority | Trade-off Bias |
|------|----------|----------------|
| **Performance** | Runtime speed, minimal re-renders, efficient algorithms, lazy loading | Favors memoization, virtualization, code splitting. Tolerates complexity for speed. |
| **Bundle Size** | Smallest payload, tree-shaking, zero/minimal dependencies | Avoids libraries when native APIs exist. Inlines small utilities. |
| **Memory** | Low allocation, object reuse, streaming, no leaks | Favors pooling, iterators over arrays, cleanup on unmount/close. |

### User-Facing Quality
| Lens | Priority | Trade-off Bias |
|------|----------|----------------|
| **Accessibility** | WCAG compliance, keyboard navigation, screen reader support, semantic HTML | Favors ARIA, focus management, contrast ratios. Will add markup for a11y even if verbose. |
| **UX Polish** | Animations, transitions, loading states, micro-interactions, responsive | Favors progressive disclosure, skeleton screens, optimistic UI. |
| **Design Fidelity** | Pixel-perfect, consistent spacing/typography, design-system alignment | Favors tokens/variables, explicit sizing, visual QA. |

### Architecture & Maintainability
| Lens | Priority | Trade-off Bias |
|------|----------|----------------|
| **Testability** | Easy to unit test, mockable boundaries, pure functions, clear contracts | Favors dependency injection, small modules, deterministic outputs. |
| **Type Safety** | Strict types, no `any`, exhaustive discriminated unions, runtime validation | Favors Zod/io-ts, branded types, compile-time guarantees. |
| **Scalability** | Will this work at 10x scale? Separation of concerns, extensibility | Favors interfaces, plugin patterns, decoupled modules. |
| **Security** | Input sanitization, CSP, XSS/CSRF prevention, least privilege | Favors allowlists, parameterized queries, escaping, auth checks. |

### Domain-Specific
| Lens | Priority | Trade-off Bias |
|------|----------|----------------|
| **React Best Practices** | Hooks rules, component composition, proper state management, React Server Components | Favors colocation, lifting state minimally, suspense boundaries. |
| **CSS Efficiency** | Minimal specificity, no duplication, logical properties, modern CSS features | Favors `has()`, container queries, custom properties, grid. Avoids utility-class sprawl. |
| **API Design** | RESTful conventions, consistent error format, pagination, versioning | Favors resource-oriented routes, proper status codes, HATEOAS where useful. |
| **Data Modeling** | Normalized schemas, proper indexes, migration safety, query efficiency | Favors foreign keys, partial indexes, incremental migrations. |

## Workflow Steps

### Step 1: Understand the Task

Before forming a team, fully understand what the user wants built. Read relevant files, clarify ambiguities. Determine which mode fits:
- Single deliverable with subjective trade-offs → **Competitive**
- Multi-part task with clear boundaries → **Collaborative**
- Important architectural decision + implementation → **Hybrid**

### Step 2: Select Lenses

Choose 2-4 lenses that create **productive tension** for this specific task. Good pairings push agents toward meaningfully different solutions:

| Task Type | Suggested Lenses |
|-----------|-----------------|
| New UI component | Minimalist, React Best Practices, Accessibility |
| API endpoint | Performance, Defensive, API Design |
| Refactoring | Repo Conventions, Minimalist, Testability |
| Bug fix | Defensive, Minimalist, Performance |
| Full feature | Best Practices, Performance, Accessibility, UX Polish |
| Data layer | Data Modeling, Type Safety, Performance |
| CSS/styling task | CSS Efficiency, Minimalist, Design Fidelity |
| Security-sensitive | Security, Defensive, Type Safety |

The user may also specify custom lenses — always honor those.

### Step 3: Set Up the Team

Use a git worktree (if `git gtr` is available) or the `EnterWorktree` tool so agents work in isolated directories without conflicts.

Create the team and task list:

```
TeamCreate: team_name = "<task>-swarm"
```

Create one task per agent with their lens baked into the description:

```
TaskCreate:
  subject: "Implement <feature> — Minimalist lens"
  description: |
    <Full task requirements here>

    YOUR PRIORITY LENS: Minimalist
    - Fewest lines of code, least abstraction, no unnecessary dependencies
    - Favor inline code over helpers, simple control flow over patterns
    - When facing a trade-off, choose the simpler option
    - Still must be correct and functional

    Deliver your implementation to: <path>
```

Then create a final evaluation task that is blocked by all implementation tasks.

### Step 4: Spawn Agents

Launch one agent per lens as a `general-purpose` subagent with `team_name` set. Each agent's prompt should include:

1. The full task requirements (what to build)
2. Their assigned priority lens and what it means
3. Where to write their output (a unique path per agent — e.g. a worktree, a suffixed file, or a subdirectory)
4. Instruction to claim their task via `TaskUpdate`, complete the work, then mark it done

### Handling Failures

- **Agent timeout or empty output:** The judge/lead should note the missing agent's lens and evaluate only the completed implementations. Mention the gap in the comparison.
- **All agents produce similar output:** The lenses weren't different enough — note this and pick one. Don't force artificial distinctions.
- **Agent ignores its lens:** If an implementation doesn't reflect its assigned priority, the judge should flag this and weight that agent's output lower in evaluation.
- **Build/test failures:** If an agent's implementation doesn't pass linting or tests, the judge should note this as a significant negative — correctness is non-negotiable regardless of lens.

For **competitive mode**, each agent writes to a separate location:
- Separate worktrees (preferred for full features)
- Suffixed files, e.g. `Component.minimal.tsx`, `Component.bestpractices.tsx`
- Subdirectories, e.g. `_swarm/minimal/`, `_swarm/bestpractices/`

For **collaborative mode**, agents write to the same codebase but own different files.

### Step 5: Evaluate & Select

Once all agents finish, evaluate the results. Approaches:

**Lead-as-judge (default for 2-3 agents):**
The team lead reads all outputs, compares them against the lenses, and presents a summary to the user with a recommendation.

**Dedicated judge agent (for 3+ agents or complex tasks):**
Spawn a judge agent that reads all implementations and scores them on:
- Correctness — does it work?
- Lens adherence — did it honor its priority?
- Code quality — readability, maintainability
- Trade-off cost — what did the lens sacrifice?

Present the evaluation as a comparison table:

```
| Criterion       | Minimalist | Best Practices | Accessibility |
|-----------------|------------|----------------|---------------|
| Lines of code   | 45         | 120            | 95            |
| Dependencies    | 0          | 2              | 1             |
| Error handling  | Basic      | Comprehensive  | Moderate      |
| A11y coverage   | None       | Partial        | Full          |
| Matches repo    | Somewhat   | Yes            | Yes           |
| Recommendation  |            | Runner-up      | Winner        |
```

**Synthesis (best of all worlds):**
After picking a winner, optionally cherry-pick the best ideas from runner-ups into the final implementation. For example: take the Minimalist's lean structure but add the Accessibility agent's ARIA attributes.

### Step 6: Deliver & Clean Up

- Copy or move the winning implementation to the final location
- Remove swarm artifacts (`_swarm/` directories, suffixed files, worktrees)
- Clean up the team with `TeamDelete`
- Present the final result to the user with a brief rationale for why this approach won

## Agent Prompt Template

Use this template when spawning each competitive agent:

```
You are a team member in a competitive swarm. Your goal is to implement
the task below according to your assigned priority lens.

## Task
<paste full task requirements>

## Your Priority Lens: <Lens Name>
<paste lens description and trade-off bias from the table above>

When making ANY trade-off decision, let your lens guide you:
- If two approaches are equally correct, pick the one your lens favors
- Document (in brief code comments) where your lens influenced a non-obvious choice
- Do NOT compromise on correctness — your lens shapes HOW you solve it, not WHETHER it works

## Output Location
Write your implementation to: <unique path>

## Process
1. Read the task from TaskGet and mark it in_progress
2. Study relevant existing code in the repo to understand patterns and context
3. Implement the solution guided by your lens
4. Verify your work (run linter/tests if available)
5. Mark your task as completed
```

## Example Invocations

### "Build this component in a few different styles"
→ Competitive mode with Minimalist + Best Practices + Repo Conventions

### "I want the fastest possible version of this"
→ Competitive mode with Performance + Bundle Size + Minimalist

### "Build this feature with a team"
→ Collaborative mode: one agent for API/data layer, one for UI, one for tests. Assign relevant lenses (API Design, UX Polish, Testability).

### "Explore different approaches to this refactor"
→ Competitive mode with Minimalist + Scalability + Repo Conventions

### "Make this accessible and performant"
→ Hybrid: compete on the core approach (Accessibility vs Performance vs Balanced), then collaborate to finish with the winning approach.

## Custom Lenses

Users can define their own lenses. A custom lens needs:
1. **Name** — short label
2. **Priority** — what this agent optimizes for (one sentence)
3. **Trade-off bias** — when facing ambiguity, which direction to lean (one sentence)

Example:
```
Lens: Beginner-Friendly
Priority: Code that a junior developer can read and modify confidently
Trade-off bias: Explicit over clever, verbose over terse, flat over nested
```

## Model Selection

| Role | Recommended Model | Rationale |
|------|-------------------|-----------|
| Competitive agents | `sonnet` | Implementation work; `sonnet` is fast and capable for code generation |
| Judge / Evaluator | `opus` | Comparative judgment requires nuanced reasoning across multiple implementations |
| Collaborative agents | `sonnet` | Each owns a focused implementation task |

For most swarms, `sonnet` agents produce excellent implementations at lower cost. Reserve `opus` for the evaluation/synthesis step where subjective comparison matters. If the task is architecturally complex (the approach matters more than the implementation), consider `opus` for all agents.

## Skill Chains

Analysis skills produce findings that are natural inputs to competitive-swarm remediation:

| Chain | Skills (in order) | Use when |
|-------|-------------------|----------|
| **Explore Remediation** | `deep-audit` or `tech-debt-inventory` → `competitive-swarm` (fix critical findings with different lenses) | Complex fix with architectural trade-offs |
| **Refactor with Confidence** | `architecture-forensics` → `competitive-swarm` (refactor with Minimalist vs. Best Practices vs. Repo Conventions) | Major refactor of a module identified as problematic |
| **Performance Fix Bake-Off** | `performance-profiler` → `competitive-swarm` (optimize with Performance vs. Minimalist vs. Memory lenses) | Bottleneck has multiple possible optimization strategies |
| **Sanity → Competitive Fix** | `sanity-check` → `competitive-swarm` (implement the 10x Engineer's alternative approach) | Sanity check revealed a fundamentally different approach worth exploring |

When using findings from an analysis skill as input, include the specific finding ID and evidence in each agent's task description so they're solving a concrete problem, not re-analyzing the codebase.

## Tips

- **2-3 agents** is the sweet spot for most tasks. More agents = more cost and comparison overhead.
- **Maximize tension** between lenses. Two similar lenses produce similar outputs and waste work. Minimalist vs. Defensive creates better contrast than Minimalist vs. Bundle Size.
- **Time-box evaluation.** Don't over-analyze — if one solution is clearly better, pick it and move on.
- **Repo Conventions is a universal lens** that pairs well with anything. It keeps solutions grounded in the codebase.
- For **quick comparisons**, skip the full team setup — just spawn 2 Task agents in parallel with different prompts and compare results yourself.
- **The user decides.** Always present options and let the user pick. Never silently discard an agent's work.
