---
name: research-problem-solving-swarm
description: Broad-but-disciplined research and problem-solving workflow for ambiguous questions, debugging, architecture decisions, vendor or tool evaluations, root-cause analysis, and "what are my options?" requests. Use when the user wants many plausible avenues explored, multiple strategies attempted or pressure-tested, tradeoffs summarized, and a written recommendation or research memo saved in the project or in Notion, Obsidian, Apple Notes, or another notes system.
---

# Research & Problem Solving

Use this skill to widen the search space first, then narrow it with evidence. Explore multiple plausible avenues, run the cheapest credible tests, and end with a recommendation that explains why it beats the alternatives.

## Polya Method

Apply Polya's "How to Solve It" method throughout the task, even when the task is research-heavy rather than a single clean math problem.

1. Understand the problem.
   - Restate the question in plain language.
   - Identify the unknowns, givens, constraints, assumptions, and success criteria.
   - Distinguish what is actually being asked from adjacent but non-essential questions.
2. Devise a plan.
   - Generate multiple plausible avenues before committing.
   - Look for analogies, known patterns, invariants, simpler subproblems, and reversible tests.
   - Prefer a sequence of cheap checks that can quickly eliminate weak paths.
3. Carry out the plan.
   - Execute the top avenues methodically.
   - Record intermediate results, failed attempts, and contradictions.
   - Adjust the plan when evidence invalidates a leading hypothesis.
4. Look back.
   - Verify the recommendation or explanation against the original question.
   - Ask whether the result is complete, whether a simpler explanation exists, and what would generalize to future cases.
   - Summarize lessons, unresolved uncertainty, and what would change the answer.

If the task contains several subproblems, run this Polya loop for each major avenue, then synthesize across them.

## Modes
- Research: map the landscape, sources, and competing explanations.
- Diagnose: investigate a bug, incident, failure, or confusing behavior.
- Solve: compare solution strategies, designs, vendors, or implementation plans.
- Full: research + diagnose/solve + synthesis.
- Solo: for narrow questions, skip the team and do the work inline.

## Exploration Bar
- Start with at least 3 materially different avenues, hypotheses, or strategies. If fewer than 3 are plausible, say why.
- Expand to 5-7 avenues when the problem spans multiple layers or when early evidence is weak.
- Pressure-test the top 2-4 avenues with evidence, experiments, or concrete checks.
- Prefer falsification over confirmation. Try to disprove the leading ideas first.
- Stop exploring only when new avenues are duplicate, dominated, blocked by missing inputs, or no longer worth the time.

## Evidence Rules
- Prefer primary sources: source code, tests, logs, official docs, specs, standards, issue trackers, benchmarks, and first-party data.
- Use secondary sources for context, not as the sole basis for core claims.
- Separate facts, inferences, and open questions.
- Record what was tried and what the outcome was, even when the attempt failed.
- If something cannot be verified, label it as tentative instead of smoothing over the gap.
- During "look back," explicitly check whether the current answer actually solves the user's stated problem or only a nearby one.

## Report Output

Detect a pre-existing ignored output folder by checking in order:
1. `temp/` in the project root
2. `tmp/` in the project root
3. `.cache/` in the project root
4. OS temp directory (`/tmp` on macOS/Linux, `$TEMP` on Windows)

Use the first folder that exists. **NEVER modify `.gitignore`** without explicit user approval. If none of the project folders exist, use the OS temp directory.

Organize all report artifacts under a dated subfolder:
```
{REPORT_PATH}/{YYYY-MM-DDTHH-MM-SS}__{research-problem-solving-swarm}/
```
Example: `tmp/2026-03-17T14-30-00__research-problem-solving-swarm/`

## Excluded Paths

When scoping files or running analysis, **always exclude** these directories. They contain tool output, generated artifacts, or config rather than source material under investigation.

```
.cache/  temp/  tmp/  .reports/  .claude/  .git/
node_modules/  vendor/  dist/  build/  out/  coverage/
__pycache__/  .next/  .nuxt/  .turbo/  .vercel/
```

## Discovery Inputs

Gather context from as many relevant sources as are reasonably available:
- User prompt, pasted artifacts, and local files
- Repo context: directory layout, dependencies, tests, recent commits, diffs, logs
- GitHub PR/issue context via `gh` when available
- Ticket and docs systems via MCP when available
- External docs, standards, benchmarks, release notes, and issue trackers when the question depends on up-to-date or niche information

Record the sources you actually used so the final memo has clear provenance.

## Workflow
1. Frame the problem.
   - Extract the question, constraints, deadline, success criteria, and desired deliverable.
   - Decide whether the task is Research, Diagnose, Solve, or Full.
   - Apply Polya step 1: restate the problem, list unknowns and givens, and note any ambiguity that could invalidate the rest of the work.
2. Build the avenue map.
   - Generate 3-7 materially different avenues, hypotheses, or strategies before converging.
   - Merge duplicates and mark obviously weak paths early.
   - Rank the remaining avenues by expected upside, uncertainty, reversibility, and cost to test.
   - Apply Polya step 2: look for analogous cases, simpler versions, decompositions, edge cases, and ways to falsify each avenue cheaply.
3. Choose the execution model.
   - Use Solo mode if the task is narrow or the evidence is already obvious.
   - Otherwise create team: `TeamCreate: team_name = "research-problem-solving-swarm"`.
4. Run specialists in parallel.
   - Give each specialist a distinct lens and enough raw context to work independently.
5. Attempt the top avenues.
   - Use the cheapest credible test first: read code, search docs, reproduce the issue, inspect logs, run targeted commands or tests, build a tiny prototype, or sketch a design.
   - Avoid destructive or expensive operations unless the user asked for them or approved them.
   - For debugging, try to falsify the top hypotheses rather than just accumulating supporting evidence.
   - Apply Polya step 3: work through the plan carefully, keep track of what changed, and pivot when the evidence contradicts the current lead.
6. Converge.
   - Mark each avenue `recommended`, `viable`, `rejected`, `blocked`, or `inconclusive`.
   - Explain why the winner wins and what evidence would change the recommendation.
   - Apply Polya step 4: check the answer against the original problem, consider a simpler or more general solution, and capture what was learned.
7. Persist the artifact.
   - Default: write a markdown report in the report directory.
   - If the user asked to save elsewhere, read [references/export-destinations.md](references/export-destinations.md) and write there via MCP or CLI when the destination and tool access are clear.
   - If no destination was requested, mention that the memo can also be saved to a project path or a notes tool.

### Monitoring & Status Updates

While work is in progress, keep the user informed:
- Which avenues are being tested
- Which avenues have already been ruled out
- Which experiments or checks are still pending
- Whether a save destination has been written yet

Do not go silent while background work is running.

## Specialists
### Context Mapper
- Define scope, constraints, known facts, unknowns, and success criteria.
- Own Polya step 1 and flag ambiguity early.

### Researcher
- Gather primary sources, map the avenue list, and identify missing evidence.
- Contribute analogies, prior art, and simpler related cases for Polya step 2.

### Experimenter
- Run targeted checks, reproductions, prototypes, or measurements against the top avenues.
- Own Polya step 3 with explicit attempt/result tracking.

### Skeptic
- Attack assumptions, search for counterexamples, and identify downside risk or hidden constraints.
- Stress-test the "look back" step by asking whether the proposed answer really solves the stated problem.

### Synthesizer
- Produce the final recommendation, compare alternatives, and write the saved artifact.
- Explicitly include Polya step 4: verification, simplification, and lessons learned.

## Findings Format
Use one file per specialist in the report directory as `RPS-<specialist>.md`.

```markdown
# <Specialist> Findings

**Mode:** <Research | Diagnose | Solve | Full | Solo>
**Scope:** <files, systems, or question area>

## Avenue Map
1. <avenue>
2. <avenue>
3. <avenue>

## Polya Notes
- Understand:
- Plan:
- Carry out:
- Look back:

## Evidence
- <source or observation>

## Attempts
- <check or experiment> -> <result>

## Assessment
- Recommended now:
- Worth keeping in reserve:
- Rejected or blocked:
```

Synthesizer output: `RESEARCH-SUMMARY.md` (in the report directory)

```markdown
# Research Summary - <Problem>

**Mode:** <Research | Diagnose | Solve | Full | Solo>
**Question:** <restated problem>
**Sources:** <what informed the result>
**Avenues explored:** <count>

## Executive Summary
<2-4 sentences>

## Polya Check
- Understand: <restated problem, givens, unknowns, constraints>
- Plan: <candidate avenues and why they were chosen>
- Carry out: <what was tried and what happened>
- Look back: <verification, simplification, lessons, remaining doubt>

## Recommended Path
<what to do and why>

## Summary & Discussion
1. <avenue> - <recommended | viable | rejected | blocked | inconclusive>
2. <avenue> - <recommended | viable | rejected | blocked | inconclusive>
3. <avenue> - <recommended | viable | rejected | blocked | inconclusive>

## Attempts and Results
- <attempt> -> <outcome>

## What Would Change the Decision
- <missing evidence, constraint, or trigger>

## Open Questions
- <remaining uncertainty>

## Saved To
- <report path or external destination>
```
