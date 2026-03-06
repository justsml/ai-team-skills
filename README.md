# AI Team Skills for Claude Code

A collection of Claude Code skills for multi-agent orchestration -- parallel specialist teams that analyze, review, and improve codebases.

## Skills

| Skill | What it does |
|-------|-------------|
| [`branch-review`](./branch-review/SKILL.md) | Combined pre-PR shape check and merge-ready review (10x perspective + correctness/perf/maintainability) |
| [`codebase-health`](./codebase-health/SKILL.md) | Unified architecture mapping, tech debt inventory, and performance profiling with a single roadmap |
| [`security-audit`](./security-audit/SKILL.md) | Security audit via parallel specialists: OWASP, privacy, exploit research, LLM risk, and holistic synthesis |
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
cp -r ./branch-review ./codebase-health ./security-audit ./competitive-swarm \
      ~/.claude/skills/
```

Or symlink individual skills for easier updates:

```bash
ln -s $(pwd)/branch-review ~/.claude/skills/branch-review
ln -s $(pwd)/codebase-health ~/.claude/skills/codebase-health
ln -s $(pwd)/security-audit ~/.claude/skills/security-audit
ln -s $(pwd)/competitive-swarm ~/.claude/skills/competitive-swarm
```

### Install Per Project

Copy skills to your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r ./branch-review ./codebase-health ./security-audit ./competitive-swarm \
  .claude/skills/
```

## Skill Chains

These skills are designed to compose. Common chains:

| Goal | Chain |
|------|-------|
| Full codebase health check | `codebase-health` (full) + `security-audit` (parallel) |
| Pre-PR pipeline | `branch-review` (shape) -> fix -> `branch-review` (merge) |
| Secure PR review | `branch-review` (merge) + `security-audit` (parallel, scoped to changed files) |
| Debt sprint planning | `codebase-health` (debt) -> `competitive-swarm` |
| Post-incident remediation | `security-audit` -> `competitive-swarm` |
