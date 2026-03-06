---
name: competitive-swarm
description: Orchestrate multiple agents to produce competing solutions and compare tradeoffs. Use when the user wants a bake-off, competitive swarm, multiple approaches, or explicit tradeoff exploration (minimal vs best practices vs performance).
---

# Competitive Swarm

Use this skill to run multiple approaches in parallel, then compare and select the best outcome.

## Modes
- Competitive: All agents solve the same task with different lenses.
- Collaborative: Agents own different parts of the task.
- Hybrid: Compete on approach, then collaborate to implement the winner.

## Lenses (pick 2-4)
- Minimalist: fewest lines, least abstraction.
- Best Practices: idiomatic patterns, readability.
- Repo Conventions: match existing codebase style.
- Defensive: robust validation and error handling.
- Performance: runtime speed and efficiency.
- Memory: low allocations and cleanup.
- Bundle Size: minimize payload.
- Testability: easy to test and mock.
- Type Safety: strict types and validation.
- Security: input sanitization and least privilege.
- Accessibility: WCAG, keyboard, semantics.
- UX Polish: loading states, transitions.

## Workflow
1. Clarify the deliverable and success criteria.
2. Pick mode and lenses.
3. Create output dir `.reports/competitive-swarm/` (add `.reports/` to `.gitignore` if missing).
4. Create team: `TeamCreate: team_name = "competitive-swarm"`.
5. Create one task per agent with its lens baked into the prompt.
6. Run agents in parallel; each writes `BAKEOFF-<lens>.md`.
7. Run a Judge agent to compare, pick a winner, and synthesize into a single plan.

### Monitoring & Status Updates

While agents are working, output status updates to the user every 1-5 minutes:

- Which agents have completed and which are still running
- Any early outputs from completed agents (read the files if available)
- Estimated progress based on task completion count

**Do NOT go silent while waiting for background agents.** The user should always know work is happening. Use `SendMessage` or direct text output to keep them informed. If all agents are still running and there's nothing new to report, a brief "Still waiting on X agents..." is sufficient.

## Findings Format
Agent output: `.reports/competitive-swarm/BAKEOFF-<lens>.md`

```markdown
# <Lens> Solution

## Approach
<core idea>

## Key tradeoffs
- <tradeoff>

## Risks
- <risk>

## Implementation sketch
<steps or pseudocode>
```

Judge output: `.reports/competitive-swarm/BAKEOFF-RESULT.md`

```markdown
# Bakeoff Result - <Task>

## Winner
<lens and rationale>

## Why it wins
- <reasons>

## Adopted plan
<merged best parts or single approach>

## Follow-up
- <next steps>
```
