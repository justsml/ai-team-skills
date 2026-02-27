# AI Team Skills for Claude Code

A collection of Claude Code skills for multi-agent orchestration — parallel specialist teams that analyze, review, and improve codebases.

## Skills

| Skill | What it does |
|-------|-------------|
| [`competitive-swarm`](./competitive-swarm/SKILL.md) | Spawn multiple agents with different priority lenses (Minimalist, Best Practices, Performance…) to tackle the same task, then compare and select the best result |
| [`pr-review-swarm`](./pr-review-swarm/SKILL.md) | Parallel PR review team: Correctness, Performance, Maintainability, and Domain Logic specialists synthesized into a single verdict |
| [`sanity-check`](./sanity-check/SKILL.md) | Pre-PR shape check: 10x Engineer, Code Organizer, and PR Splitter review your branch before you open the PR |
| [`deep-audit`](./deep-audit/SKILL.md) | Security audit via parallel specialists: OWASP, Data Privacy, Exploit Research, LLM Vulnerabilities, and a Senior Holistic Reviewer |
| [`architecture-forensics`](./architecture-forensics/SKILL.md) | Map the actual architecture of a codebase: Dependency Mapper, Data Flow Archaeologist, Boundary Analyst, and Pattern Detector |
| [`tech-debt-inventory`](./tech-debt-inventory/SKILL.md) | Catalog technical debt across five dimensions (Complexity, Dead Code, Dependencies, Inconsistencies, Test Gaps) and produce a prioritized roadmap |
| [`performance-profiler`](./performance-profiler/SKILL.md) | Performance analysis via parallel specialists: Algorithmic Complexity, I/O & Queries, Memory & Lifecycle, and Frontend Payload |

## Installation

Copy skills to your Claude Code user skills directory:

```bash
cp -r ./competitive-swarm ./pr-review-swarm ./sanity-check ./deep-audit \
      ./architecture-forensics ./tech-debt-inventory ./performance-profiler \
      ~/.claude/skills/
```

Or symlink individual skills:

```bash
ln -s $(pwd)/competitive-swarm ~/.claude/skills/competitive-swarm
```

## Requirements

These skills use Claude Code's agent team features. Make sure you have agent teams enabled:

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

## Skill Chains

These skills are designed to compose. Common chains:

| Goal | Chain |
|------|-------|
| Full codebase health check | `architecture-forensics` → `deep-audit` + `tech-debt-inventory` + `performance-profiler` (parallel) |
| Pre-PR pipeline | `sanity-check` → fix → `pr-review-swarm` |
| Secure PR review | `pr-review-swarm` + `deep-audit` (parallel, scoped to changed files) |
| Debt sprint planning | `tech-debt-inventory` → `competitive-swarm` (fix top items with different approaches) |
| Post-incident remediation | `deep-audit` → `competitive-swarm` (explore fix approaches) |
