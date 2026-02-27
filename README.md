# AI Team Skills for Claude Code

A collection of Claude Code skills for multi-agent orchestration -- parallel specialist teams that analyze, review, and improve codebases.

## Skills

| Skill | What it does |
|-------|-------------|
| [`branch-review`](./branch-review/SKILL.md) | Combined pre-PR shape check and merge-ready review (10x perspective + correctness/perf/maintainability) |
| [`codebase-health`](./codebase-health/SKILL.md) | Unified architecture mapping, tech debt inventory, and performance profiling with a single roadmap |
| [`security-audit`](./security-audit/SKILL.md) | Security audit via parallel specialists: OWASP, privacy, exploit research, LLM risk, and holistic synthesis |
| [`swarm-bakeoff`](./swarm-bakeoff/SKILL.md) | Competing or collaborative agent approaches with explicit tradeoff comparison |

## Installation

Copy skills to your Claude Code user skills directory:

```bash
cp -r ./branch-review ./codebase-health ./security-audit ./swarm-bakeoff \
      ~/.claude/skills/
```

Or symlink an individual skill:

```bash
ln -s $(pwd)/branch-review ~/.claude/skills/branch-review
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
| Full codebase health check | `codebase-health` (full) + `security-audit` (parallel) |
| Pre-PR pipeline | `branch-review` (shape) -> fix -> `branch-review` (merge) |
| Secure PR review | `branch-review` (merge) + `security-audit` (parallel, scoped to changed files) |
| Debt sprint planning | `codebase-health` (debt) -> `swarm-bakeoff` |
| Post-incident remediation | `security-audit` -> `swarm-bakeoff` |

## Legacy

Archived skills are kept in `legacy/` for reference and are not part of the active set.
