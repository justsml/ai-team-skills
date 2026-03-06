---
name: security-audit
description: Deep security audit for a codebase using OWASP, data privacy, exploit research, LLM risk, and holistic review. Use when the user asks for a security audit, vulnerability assessment, penetration-style review, or hardening guidance.
---

# Security Audit

Use this skill to run a multi-lens security review and synthesize a severity-ranked report.

## Workflow
1. Scope the audit
   - Identify the tech stack, auth, data stores, and external integrations.
   - Identify the attack surface: web app, API, CLI, background jobs, LLM features.
2. Create output dir `.reports/security-audit/` (add `.reports/` to `.gitignore` if missing).
3. Create team: `TeamCreate: team_name = "security-audit"`.
4. Create tasks per specialist and run in parallel.
5. Run the Holistic Reviewer after specialists finish; write `.reports/security-audit/SECURITY-AUDIT.md`.

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
Use one file per specialist in `.reports/security-audit/` as `SEC-<specialist>.md`.

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

Synthesizer output: `.reports/security-audit/SECURITY-AUDIT.md`

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
