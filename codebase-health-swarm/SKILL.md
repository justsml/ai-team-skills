---
name: codebase-health-swarm
description: Combined architecture mapping, tech debt inventory, and performance profiling for a codebase. Use when the user asks for codebase health, architecture overview, refactor planning, tech debt assessment, or performance audit.
---

# Codebase Health

Use this skill to assess a codebase at the system level. Pick one or more tracks, run specialists in parallel, and synthesize a prioritized roadmap.

## Tracks (pick one or more)
- Architecture: map dependencies, data flow, boundaries, and patterns.
- Debt: surface complexity, dead code, dependency risk, inconsistencies, and test gaps.
- Performance: find algorithmic, I/O, memory, and payload bottlenecks.
- Full: run all tracks.

## Report Output

Detect a pre-existing ignored output folder by checking in order:
1. `temp/` in the project root
2. `tmp/` in the project root
3. `.cache/` in the project root
4. OS temp directory (`/tmp` on macOS/Linux, `$TEMP` on Windows)

Use the first folder that exists. **NEVER modify `.gitignore`** without explicit user approval. If none of the project folders exist, use the OS temp directory.

Organize all report artifacts under a dated subfolder:
```
{REPORT_PATH}/{YYYY-MM-DDTHH-MM-SS}__{codebase-health-swarm}/
```
Example: `tmp/2026-03-06T14-30-00__codebase-health-swarm/`

## Project Discovery via CLI

Use common CLI tools to build a quick project profile before starting the assessment. Check if each binary exists before using it (`command -v <tool> >/dev/null 2>&1`), and fall back to alternatives when unavailable.

- **Directory structure:** `tree -L 2 -I node_modules` — fallback: `find . -maxdepth 2 -print | sed -e 's;[^/]*/;|____;g;s;____|; |;g'`
- **Project size:** `cloc .` or `scc .` — fallback: `find . -type f \( -name '*.ts' -o -name '*.py' -o -name '*.go' \) | xargs wc -l`
- **Disk usage:** `du -sh .` — fallback: `find . -type f -exec ls -l {} + | awk '{total += $5} END {printf "%.1f MB\n", total/1048576}'`
- **Git stats:** `git log --oneline -20`, `git shortlog -sn --no-merges`, `git diff --stat`
- **Dependencies:** Check for `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, `Gemfile`, `pom.xml`, etc.
- **Search tools:** Prefer `rg` (ripgrep) or `ag` (silver searcher) over `grep` when available for faster codebase searches

## Branch / PR / Ticket Context

Spend 1-3 minutes gathering project context from available integrations. If nothing is found, proceed with just the codebase.

- **Git:** `git branch --show-current`, `git log --oneline -5`
- **GitHub PR:** `gh pr view --json headRefName,baseRefName,number,title,body 2>/dev/null`
- **Linked tickets:** Scan PR body and recent commits for ticket references (e.g. `PROJ-123`, `#456`, Jira/Linear/Shortcut URLs)
- **MCP integrations:** Discover ticket trackers via `ToolSearch: "issue"` or `ToolSearch: "ticket"` — use Linear, Jira, or other MCP tools if available
- **Documentation:** Check for relevant docs in Google Drive, Notion, or Confluence MCP if available
- **CI status:** `gh pr checks 2>/dev/null` or `gh run list --limit 3 2>/dev/null`

Record what was found (and what wasn't) so the assessment has clear provenance.

## Workflow
1. Scope the repo
   - Read directory layout and entry points.
   - Identify primary languages, frameworks, databases, and key services.
   - Identify tests, CI, and linting.
2. Choose track(s).
3. Create the report output directory (see **Report Output** above).
4. Create team: `TeamCreate: team_name = "codebase-health-swarm"`.
5. Create tasks per track and run in parallel.
6. Run the Health Lead synthesizer after specialists finish; write `CODEBASE-HEALTH.md` in the report directory.

### Monitoring & Status Updates

While agents are working, output status updates to the user every 1-5 minutes:

- Which agents have completed and which are still running
- Any early outputs from completed agents (read the files if available)
- Estimated progress based on task completion count

**Do NOT go silent while waiting for background agents.** The user should always know work is happening. Use `SendMessage` or direct text output to keep them informed. If all agents are still running and there's nothing new to report, a brief "Still waiting on X agents..." is sufficient.

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
Use one file per specialist in the report directory:
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

Synthesizer output: `CODEBASE-HEALTH.md` (in report directory)

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
Use `competitive-swarm` when multiple refactor or optimization approaches are viable.
