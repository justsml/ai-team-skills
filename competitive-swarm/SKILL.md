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

## Report Output

Detect a pre-existing ignored output folder by checking in order:
1. `temp/` in the project root
2. `tmp/` in the project root
3. `.cache/` in the project root
4. OS temp directory (`/tmp` on macOS/Linux, `$TEMP` on Windows)

Use the first folder that exists. **NEVER modify `.gitignore`** without explicit user approval. If none of the project folders exist, use the OS temp directory.

Organize all report artifacts under a dated subfolder:
```
{REPORT_PATH}/{YYYY-MM-DDTHH-MM-SS}__{competitive-swarm}/
```
Example: `tmp/2026-03-06T14-30-00__competitive-swarm/`

## Excluded Paths

When scoping files or running analysis, **always exclude** these directories. They contain tool output, generated artifacts, or config — not source code under evaluation.

```
.cache/  temp/  tmp/  .reports/  .claude/  .git/
node_modules/  vendor/  dist/  build/  out/  coverage/
__pycache__/  .next/  .nuxt/  .turbo/  .vercel/
```

## Project Discovery via CLI

Use common CLI tools to build a quick project profile before starting the swarm. Check if each binary exists before using it (`command -v <tool> >/dev/null 2>&1`), and fall back to alternatives when unavailable.

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

Record what was found (and what wasn't) so the swarm has clear provenance.

## Workflow
1. Clarify the deliverable and success criteria.
2. Pick mode and lenses.
3. Create the report output directory (see **Report Output** above).
4. Create team: `TeamCreate: team_name = "competitive-swarm"`.
5. Create one task per agent with its lens baked into the prompt.
6. Run agents in parallel; each writes `BAKEOFF-<lens>.md` in the report directory.
7. Run a Judge agent to compare, pick a winner, and synthesize into a single plan.

### Monitoring & Status Updates

While agents are working, output status updates to the user every 1-5 minutes:

- Which agents have completed and which are still running
- Any early outputs from completed agents (read the files if available)
- Estimated progress based on task completion count

**Do NOT go silent while waiting for background agents.** The user should always know work is happening. Use `SendMessage` or direct text output to keep them informed. If all agents are still running and there's nothing new to report, a brief "Still waiting on X agents..." is sufficient.

## Findings Format
Agent output: `BAKEOFF-<lens>.md` (in report directory)

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

Judge output: `BAKEOFF-RESULT.md` (in report directory)

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
