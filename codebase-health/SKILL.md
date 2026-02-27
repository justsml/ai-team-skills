---
name: codebase-health
description: Combined architecture mapping, tech debt inventory, and performance profiling for a codebase. Use when the user asks for codebase health, architecture overview, refactor planning, tech debt assessment, or performance audit.
---

# Codebase Health

Use this skill to assess a codebase at the system level. Pick one or more tracks, run specialists in parallel, and synthesize a prioritized roadmap.

## Tracks (pick one or more)
- Architecture: map dependencies, data flow, boundaries, and patterns.
- Debt: surface complexity, dead code, dependency risk, inconsistencies, and test gaps.
- Performance: find algorithmic, I/O, memory, and payload bottlenecks.
- Full: run all tracks.

## Workflow
1. Scope the repo
   - Read directory layout and entry points.
   - Identify primary languages, frameworks, databases, and key services.
   - Identify tests, CI, and linting.
2. Choose track(s).
3. Create output dir `.reports/codebase-health/` (add `.reports/` to `.gitignore` if missing).
4. Create team: `TeamCreate: team_name = "codebase-health"`.
5. Create tasks per track and run in parallel.
6. Run the Health Lead synthesizer after specialists finish; write `.reports/codebase-health/CODEBASE-HEALTH.md`.

## Track Specialists
### Architecture
- Dependency Mapper
- Data Flow Analyst
- Boundary Analyst
- Pattern Detector

### Debt
- Complexity Analyst
- Dead Code Hunter
- Dependency Auditor
- Inconsistency Detector
- Test Gap Analyst

### Performance
- Algorithmic Analyst
- I/O & Query Analyst
- Memory & Lifecycle Analyst
- Frontend Payload Analyst (or response payloads if backend-only)

### Health Lead (Synthesizer)
- Deduplicate, assess risk/impact, and produce a ranked roadmap.

## Scoping Guidance
- For large repos (500+ files), scope each specialist to the most active directories.
- Use `git log --format='%H' -- <dir> | wc -l` or `rg --files` to find hot areas.
- If the user only cares about one module, scope to that module.

## Findings Format
Use one file per specialist in `.reports/codebase-health/`:
- Architecture: `ARCH-<specialist>.md`
- Debt: `DEBT-<specialist>.md`
- Performance: `PERF-<specialist>.md`

```markdown
# <Specialist> Findings

**Track:** <Architecture | Debt | Performance>
**Scope:** <files or dirs>

## High impact
1. <issue>
- Evidence: `path:line`
- Risk: <why it matters>
- Fix: <action>

## Medium impact
- <items>

## Low impact
- <items>
```

Synthesizer output: `.reports/codebase-health/CODEBASE-HEALTH.md`

```markdown
# Codebase Health - <Repo>

**Tracks:** <Architecture | Debt | Performance | Full>
**Scope:** <dirs>

## Executive Summary
<2-4 sentences>

## Architecture Map (if run)
<key boundaries, dependency hotspots>

## Top Debt Items (if run)
1. <item>
2. <item>
3. <item>

## Top Performance Opportunities (if run)
1. <item>
2. <item>
3. <item>

## Roadmap
- Now: <high impact, low effort>
- Next: <medium effort>
- Later: <large refactors>
```

## Follow-on
Use `swarm-bakeoff` when multiple refactor or optimization approaches are viable.
