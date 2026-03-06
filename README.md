# AI Team Skills for Claude Code

A collection of Claude Code skills for multi-agent orchestration -- parallel specialist teams that analyze, review, and improve codebases.

## Skills

| Skill | What it does |
|-------|-------------|
| [`branch-review-swarm`](./branch-review-swarm/SKILL.md) | Combined pre-PR shape check and merge-ready review (10x perspective + correctness/perf/maintainability) |
| [`codebase-health-swarm`](./codebase-health-swarm/SKILL.md) | Unified architecture mapping, tech debt inventory, and performance profiling with a single roadmap |
| [`security-audit-swarm`](./security-audit-swarm/SKILL.md) | Security audit via parallel specialists: OWASP, privacy, exploit research, LLM risk, and holistic synthesis |
| [`competitive-swarm`](./competitive-swarm/SKILL.md) | Competing or collaborative agent approaches with explicit tradeoff comparison |

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

Or set the environment variable in your shell:

```bash
echo "export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1" >> ~/.zshrc
source ~/.zshrc
```

## Installation

### Install for All Projects

Copy skills to your Claude Code **user** skills directory:

```bash
cp -rfv ./branch-review-swarm ./codebase-health-swarm ./security-audit-swarm ./competitive-swarm \
      ~/.claude/skills/
```

Or symlink individual skills for easier updates:

```bash
ln -s $(pwd)/branch-review-swarm ~/.claude/skills/branch-review-swarm
ln -s $(pwd)/codebase-health-swarm ~/.claude/skills/codebase-health-swarm
ln -s $(pwd)/security-audit-swarm ~/.claude/skills/security-audit-swarm
ln -s $(pwd)/competitive-swarm ~/.claude/skills/competitive-swarm
```

### Install Per Project

Copy skills to your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -rfv ./branch-review-swarm ./codebase-health-swarm ./security-audit-swarm ./competitive-swarm \
  .claude/skills/
```

## Skill Chains

These skills are designed to compose. Common chains:

| Goal | Chain |
|------|-------|
| Full codebase health check | `codebase-health-swarm` (full) + `security-audit-swarm` (parallel) |
| Pre-PR pipeline | `branch-review-swarm` (shape) -> fix -> `branch-review-swarm` (merge) |
| Secure PR review | `branch-review-swarm` (merge) + `security-audit-swarm` (parallel, scoped to changed files) |
| Debt sprint planning | `codebase-health-swarm` (debt) -> `competitive-swarm` |
| Post-incident remediation | `security-audit-swarm` -> `competitive-swarm` |
