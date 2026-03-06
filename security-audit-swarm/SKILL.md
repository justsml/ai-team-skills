---
name: security-audit-swarm
description: Deep security audit for a codebase using OWASP, data privacy, exploit research, LLM risk, and holistic review. Use when the user asks for a security audit, vulnerability assessment, penetration-style review, or hardening guidance.
---

# Security Audit

Use this skill to run a multi-lens security review and synthesize a severity-ranked report.

## Report Output

Detect a pre-existing ignored output folder by checking in order:
1. `temp/` in the project root
2. `tmp/` in the project root
3. `.cache/` in the project root
4. OS temp directory (`/tmp` on macOS/Linux, `$TEMP` on Windows)

Use the first folder that exists. **NEVER modify `.gitignore`** without explicit user approval. If none of the project folders exist, use the OS temp directory.

Organize all report artifacts under a dated subfolder:
```
{REPORT_PATH}/{YYYY-MM-DDTHH-MM-SS}__{security-audit-swarm}/
```
Example: `tmp/2026-03-06T14-30-00__security-audit-swarm/`

## Project Discovery via CLI

Use common CLI tools to build a quick project profile before starting the audit. Check if each binary exists before using it (`command -v <tool> >/dev/null 2>&1`), and fall back to alternatives when unavailable.

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

Record what was found (and what wasn't) so the audit has clear provenance.

## Workflow
1. Scope the audit
   - Identify the tech stack, auth, data stores, and external integrations.
   - Identify the attack surface: web app, API, CLI, background jobs, LLM features.
2. Create the report output directory (see **Report Output** above).
3. Create team: `TeamCreate: team_name = "security-audit-swarm"`.
4. Create tasks per specialist and run in parallel.
5. Run the Holistic Reviewer after specialists finish; write `SECURITY-AUDIT.md` in the report directory.

### Monitoring & Status Updates

While agents are working, output status updates to the user every 1-5 minutes:

- Which agents have completed and which are still running
- Any early outputs from completed agents (read the files if available)
- Estimated progress based on task completion count

**Do NOT go silent while waiting for background agents.** The user should always know work is happening. Use `SendMessage` or direct text output to keep them informed. If all agents are still running and there's nothing new to report, a brief "Still waiting on X agents..." is sufficient.

## Specialists
- OWASP AppSec
- Data Privacy and Exfiltration
- Exploit Research and Attack Chains
- LLM Vulnerability (if no LLM features, act as second reviewer)
- Holistic Security Reviewer (synthesizer)

## Findings Format
Use one file per specialist in the report directory as `SEC-<specialist>.md`.

```markdown
# <Specialist> Security Findings

**Scope:** <files or dirs>

## Critical
### [CRIT-01] <Title>
- Location: `path:line`
- Impact: <what attacker gains>
- Evidence: <why it is real>
- Fix: <action>

## High
### [HIGH-01] <Title>
- Location: `path:line`
- Impact: <impact>
- Fix: <action>

## Medium
- <items>

## Low
- <items>
```

Synthesizer output: `SECURITY-AUDIT.md` (in report directory)

```markdown
# Security Audit - <Repo>

**Scope:** <dirs>

## Overall Risk: <Low | Medium | High>

## Must Fix
1. <critical or high>
2. <critical or high>

## Fix Soon
1. <medium>
2. <medium>

## Defense-in-Depth Improvements
- <systemic improvements>
```
