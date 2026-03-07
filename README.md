# AI Team Skills for Claude Code

[![GitHub stars](https://img.shields.io/github/stars/justsml/ai-team-skills?style=flat-square)](https://github.com/justsml/ai-team-skills/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/justsml/ai-team-skills?style=flat-square)](https://github.com/justsml/ai-team-skills/network/members)
[![GitHub issues](https://img.shields.io/github/issues/justsml/ai-team-skills?style=flat-square)](https://github.com/justsml/ai-team-skills/issues)
[![Last commit](https://img.shields.io/github/last-commit/justsml/ai-team-skills?style=flat-square)](https://github.com/justsml/ai-team-skills/commits)
![Skills](https://img.shields.io/badge/skills-4-blue?style=flat-square)
![Agent teams required](https://img.shields.io/badge/Claude%20agent%20teams-required-orange?style=flat-square)

Multi-agent skills for faster, higher-signal engineering workflows.

Use these skills to run parallel specialist teams for:
- PR and branch reviews
- Codebase health assessments
- Security audits
- Solution bake-offs with explicit tradeoffs

## Included Skills

| Skill | Best for | Output |
| --- | --- | --- |
| [`branch-review-swarm`](./branch-review-swarm/SKILL.md) | "Is this PR merge-ready?", "How should I split this?" | A synthesized review verdict with prioritized fixes and split strategy |
| [`codebase-health-swarm`](./codebase-health-swarm/SKILL.md) | Architecture mapping, debt inventory, performance opportunities | A ranked roadmap: now / next / later |
| [`security-audit-swarm`](./security-audit-swarm/SKILL.md) | Vulnerability assessment and hardening guidance | Severity-ranked security report with must-fix items |
| [`competitive-swarm`](./competitive-swarm/SKILL.md) | Multiple implementation approaches and tradeoff comparison | Bake-off results with winner and adopted plan |

## Quick Start

### 1. Enable Agent Teams

Add to `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Or in shell config (`~/.zshrc`, `~/.bashrc`, etc.):

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

### 2. Install Skills

Install for all projects (user-level):

```bash
cp -rfv ./branch-review-swarm ./codebase-health-swarm ./security-audit-swarm ./competitive-swarm \
  ~/.claude/skills/
```

Or symlink for easier updates:

```bash
ln -s "$(pwd)/branch-review-swarm" ~/.claude/skills/branch-review-swarm
ln -s "$(pwd)/codebase-health-swarm" ~/.claude/skills/codebase-health-swarm
ln -s "$(pwd)/security-audit-swarm" ~/.claude/skills/security-audit-swarm
ln -s "$(pwd)/competitive-swarm" ~/.claude/skills/competitive-swarm
```

Install per-project:

```bash
mkdir -p .claude/skills
cp -rfv ./branch-review-swarm ./codebase-health-swarm ./security-audit-swarm ./competitive-swarm \
  .claude/skills/
```

## Which Skill Should I Use?

| Goal | Recommended skill |
| --- | --- |
| Pre-PR sanity check, 10x pass, split advice | `branch-review-swarm` (Shape mode) |
| Merge readiness review | `branch-review-swarm` (Merge mode) |
| Whole-repo architecture + debt + perf | `codebase-health-swarm` (Full track) |
| Security posture and exploit paths | `security-audit-swarm` |
| Build the best possible features | `competitive-swarm` |

## Prompt Starters

You can use the skills via slash commands `/branch-review-swarm`, `/codebase-health-swarm`, etc.) or you can add some details, tune the focus, set the model, or constrain the scope with natural language.

Use prompts like these in Claude Code/Codex/OpenCode/etc. to get started:

```text
/branch-review-swarm using Opus and up to 3 specialists.
```

```text
/codebase-health-swarm
produce a ranked remediation roadmap.
```

```text
/security-audit-swarm run on auth, API, and data access layers first.
```

```text
/competitive-swarm add a custom emoji picker with upload support.
```

## High-Value Skill Chains

| Objective | Chain |
| --- | --- |
| End-to-end code quality gate | `branch-review-swarm` + `security-audit-swarm` |
| Incident follow-up hardening | `security-audit-swarm` -> `competitive-swarm` |
| Large PR cleanup and decomposition | `branch-review-swarm` (Shape) -> fix -> `branch-review-swarm` (Merge) |

## What You’ll Get

Each skill writes artifacts into a timestamped report folder under `temp/`, `tmp/`, `.cache/`, or OS temp when those folders are unavailable.

You get:
- Specialist reports per lens
- A synthesized summary with prioritized actions
- Explicit tradeoffs and implementation guidance

## Repository Layout

```text
.
├── branch-review-swarm/
│   └── SKILL.md
├── codebase-health-swarm/
│   └── SKILL.md
├── security-audit-swarm/
│   └── SKILL.md
└── competitive-swarm/
    └── SKILL.md
```

## Notes

- These skills are designed to compose; run them independently or chain them for stronger outcomes.
- For exact workflow details (modes, report format, specialist definitions), open each skill’s `SKILL.md`.
