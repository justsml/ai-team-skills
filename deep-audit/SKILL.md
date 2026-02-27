---
name: deep-audit
description: Deep security audit using a parallel multi-agent team of specialists — OWASP, data privacy, exploit research, LLM vulnerabilities, and a senior holistic reviewer — to comprehensively analyze a codebase for security issues.
---

# Deep Security Audit Swarm

Orchestrate a parallel team of security specialists who each audit a codebase through their unique lens, then synthesize findings into a unified, severity-ranked report. Use this skill when the user wants a thorough security review, penetration-style analysis, or vulnerability assessment of their codebase.

## When to Use

- User asks for a security audit, security review, or vulnerability assessment
- User wants to find security issues, holes, or weaknesses in their code
- User mentions OWASP, penetration testing, threat modeling, or security hardening
- User asks about data privacy, exfiltration risks, or LLM security
- User says "deep audit" or references this skill

## When NOT to Use

- Codebase has fewer than ~20 source files — a single security-focused agent can cover everything without team overhead
- User wants a quick spot-check of a specific vulnerability — answer directly instead of spawning a team
- The code is a static site or pure documentation with no backend, auth, or data handling
- User is asking about a single file or function — do an inline review instead of orchestrating a team
- User wants a pre-PR code quality review, not security — use `sanity-check` or `pr-review-swarm` instead
- The codebase has already been audited recently and no significant changes have been made

## The Specialists

Each specialist is a `general-purpose` subagent with deep expertise in their domain. They work in parallel, reviewing the same codebase independently, then findings are synthesized.

### 1. OWASP Application Security Expert

**Focus:** The OWASP Top 10 and beyond — the bread and butter of web/app security.

**Reviews for:**
- **Injection** — SQL injection, NoSQL injection, command injection, LDAP injection, expression language injection. Traces all paths from user input to query/command execution.
- **Broken Authentication** — Weak session management, credential storage, brute-force exposure, missing MFA considerations, insecure password reset flows, JWT misuse (weak algorithms, missing expiration, no audience validation).
- **Broken Access Control** — Missing authorization checks, IDOR (insecure direct object references), privilege escalation paths, missing function-level access control, CORS misconfiguration.
- **Security Misconfiguration** — Verbose error messages leaking stack traces, default credentials, unnecessary HTTP methods enabled, missing security headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options), debug mode in production, open admin panels.
- **Cryptographic Failures** — Weak hashing (MD5, SHA1 for passwords), missing encryption at rest/in transit, hardcoded secrets/keys/tokens, insufficient key rotation, use of deprecated TLS versions.
- **Insecure Design** — Missing rate limiting, no account lockout, lack of input validation at trust boundaries, missing CSRF protection, insecure deserialization.
- **Vulnerable Dependencies** — Known CVEs in dependencies, outdated packages with security patches available, dependency confusion risks.
- **SSRF** — Server-side request forgery via user-controlled URLs, DNS rebinding, internal service exposure.
- **Logging & Monitoring** — Sensitive data in logs, missing audit trails for security events, insufficient logging for auth failures.

**Methodology:** Systematically walks each OWASP Top 10 category against the codebase. Checks every HTTP handler, middleware chain, and database query. Maps trust boundaries.

---

### 2. Data Privacy & Exfiltration Specialist

**Focus:** How data flows, where it leaks, and what leaves the system — intended or not.

**Reviews for:**
- **PII Exposure** — Personal data in logs, error messages, URLs, query strings, local storage, cookies, client-side state. Traces PII fields (email, name, phone, SSN, IP, device fingerprints) through the entire stack.
- **Data Flow Mapping** — Where sensitive data enters the system, how it transforms, where it's stored, and where it exits. Identifies every outbound channel: API responses, webhooks, emails, analytics, third-party SDKs, client bundles.
- **Exfiltration Vectors** — DNS exfiltration via controlled domains, timing side-channels, error-based data extraction, breadcrumb trails in metadata, file upload/download paths that bypass access controls.
- **Third-Party Data Leakage** — Data sent to analytics (Google Analytics, Mixpanel, Segment), error tracking (Sentry, Datadog), CDNs, and other third-party services. What fields are included? Are PII fields scrubbed?
- **Storage Security** — Unencrypted sensitive fields in databases, PII in search indexes, cached sensitive data without TTL, session data in client-accessible storage, backup exposure.
- **Regulatory Compliance Surface** — GDPR data subject rights (right to deletion, export), CCPA/CPRA consent flows, HIPAA PHI handling, data retention policies (or lack thereof), cross-border data transfer risks.
- **Client-Side Leakage** — Sensitive data in Redux/Zustand stores visible via devtools, API responses returning more fields than the UI needs (over-fetching), secrets in client bundles, source maps in production.
- **Timing & Side Channels** — Enumeration via response time differences (e.g. "user not found" faster than "wrong password"), cache timing attacks, behavioral differences that leak information.

**Methodology:** Maps all data flows end-to-end. Traces every sensitive field from ingestion to deletion. Checks every outbound HTTP call, log statement, and client-facing response.

---

### 3. Exploit Researcher & Tester

**Focus:** Actually exploiting weaknesses — chaining vulnerabilities, constructing attack paths, proving impact.

**Reviews for:**
- **Attack Chain Construction** — Finds sequences of individually-minor issues that chain into critical exploits. Example: information disclosure + IDOR + missing rate limit = mass data scraping.
- **Input Fuzzing Analysis** — Identifies inputs that aren't properly bounded: oversized payloads, Unicode edge cases (null bytes, RTL overrides, homoglyphs), nested objects that cause algorithmic complexity (ReDoS, prototype pollution, zip bombs).
- **Authentication Bypass** — Token forgery, session fixation, race conditions in auth flows, OAuth/OIDC misconfigurations (redirect URI manipulation, state parameter absence, token leakage via referrer).
- **Privilege Escalation Paths** — Horizontal (accessing other users' data) and vertical (user → admin). Checks every endpoint for authorization gaps, role confusion, parameter tampering.
- **Race Conditions** — TOCTOU bugs in financial transactions, double-submit on critical operations, concurrent modification without locking, check-then-act patterns without atomicity.
- **File & Upload Attacks** — Unrestricted file upload (polyglot files, web shells), path traversal in file names, MIME type confusion, zip slip, image processing vulnerabilities (ImageTragick-style).
- **Business Logic Exploitation** — Negative quantity in cart, coupon stacking, referral abuse, state machine violations (skipping payment step), replay attacks on idempotency keys.
- **Prototype Pollution & Deserialization** — `__proto__` manipulation in JavaScript, unsafe `JSON.parse` → `eval` chains, deserialization gadgets in any language.
- **Supply Chain** — Dependency typosquatting, compromised transitive dependencies, postinstall scripts, lockfile manipulation.

**Methodology:** Thinks like an attacker. For every vulnerability found, asks "how would I actually exploit this?" and "what's the worst-case impact?" Constructs proof-of-concept attack narratives. Chains findings across specialists.

---

### 4. LLM Vulnerability Specialist

**Focus:** AI/LLM-specific attack surface — prompt injection, model abuse, agent security.

**Reviews for:**
- **Prompt Injection** — Direct injection (user input reaches system prompt), indirect injection (data from external sources contains instructions), multi-turn manipulation, instruction hierarchy bypass. Checks every path where user/external content enters an LLM call.
- **System Prompt Leakage** — Can users extract system prompts via "repeat your instructions" style attacks, error messages that include prompt fragments, debugging endpoints that expose prompts.
- **Tool/Function Call Abuse** — If the LLM has tool access: can it be tricked into calling dangerous tools? Are tool inputs validated? Can tool descriptions be manipulated? Is there a human-in-the-loop for destructive actions?
- **Data Poisoning Surface** — If RAG is used: can users inject content into the knowledge base that alters future responses? Are embeddings from untrusted sources sanitized? Can retrieval be manipulated?
- **Output Safety** — LLM outputs rendered as HTML/markdown without sanitization (stored XSS via LLM), generated SQL/code executed without sandboxing, file paths from LLM used in filesystem operations.
- **Model Denial of Service** — Inputs designed to maximize token consumption, recursive tool-calling loops, context window stuffing, expensive embedding operations triggered by users.
- **Jailbreaking Surface** — How hardened are system prompts against known jailbreak techniques? DAN-style, roleplay, base64 encoding, language switching, multi-turn context shifting.
- **Agent Autonomy Risks** — Can an AI agent take irreversible actions without confirmation? Are there guardrails on financial transactions, data deletion, external communications? Can the agent be socially engineered by data it processes?
- **Credential & Context Leakage** — API keys, database credentials, or internal URLs in system prompts or tool configurations. Context windows that accumulate sensitive data across turns without cleanup.
- **Multi-Tenant Isolation** — In shared LLM systems: can one user's context leak into another's? Are conversation histories properly isolated? Can users access other users' fine-tuned models or RAG data?

**Methodology:** Traces every LLM invocation. Maps what reaches the prompt, what tools are available, what happens with the output. Evaluates defense layers (input filtering, output validation, guardrails). Considers both direct attacks and indirect injection via data the LLM processes.

---

### 5. Senior Holistic Security Reviewer ("The Closer")

**Focus:** The nuanced, cross-cutting, hard-to-categorize issues that fall between specialties. Architectural blind spots. Systemic patterns.

**Reviews for:**
- **Architectural Security Posture** — Trust boundary mapping, defense-in-depth assessment. Where are the single points of failure? What happens if one layer is breached — does the whole system fall?
- **Implicit Trust Assumptions** — Internal services that trust each other without authentication, "it's behind the firewall" assumptions, service-to-service calls without mTLS, shared secrets across environments.
- **Error Handling as an Attack Surface** — Do different error paths reveal different information? Are error messages consistent? Do stack traces leak in any environment? Can errors be induced to cause denial of service?
- **Configuration & Environment Risks** — Environment variable handling (`.env` files committed, missing from `.gitignore`), secrets management (hardcoded vs. vault), configuration drift between environments, feature flags that bypass security.
- **Deployment & Infrastructure Signals** — Dockerfile security (running as root, secrets in build args, multi-stage leakage), CI/CD pipeline permissions (over-scoped tokens, secret exposure in logs), infrastructure-as-code misconfigurations.
- **The "Clever" Bugs** — Subtle issues that require understanding the interaction between multiple systems: cache poisoning via host header injection, request smuggling setups, HTTP desync between proxy and application server, parser differential attacks.
- **Systemic Patterns** — Across the entire codebase: is authorization consistently applied or ad hoc? Is input validation centralized or scattered? Are there entire classes of bugs that a single architectural change would eliminate?
- **What Everyone Else Missed** — Reviews findings from all other specialists. Looks for gaps in coverage, false negatives, findings that were individually minor but collectively severe. Asks: "if I were a sophisticated attacker who read all these findings, what would I target?"
- **Remediation Architecture** — Not just "fix this bug" but "here's the architectural pattern that prevents this class of bug." Proposes systemic improvements: middleware-level authorization, centralized input validation, security linting rules, pre-commit hooks.

**Methodology:** Reads the codebase holistically first — architecture, data flow, trust boundaries. Reviews other specialists' findings for gaps and false negatives. Focuses on the seams between systems, the assumptions nobody questioned, and the patterns that indicate systemic risk.

---

## Workflow

### Step 1: Scope the Audit

Before spawning the team, understand what's being audited:

1. **Read the codebase structure** — `ls`, `Glob`, key config files, README, package.json/Gemfile/requirements.txt
2. **Identify the tech stack** — framework, language, database, auth provider, hosting
3. **Identify the attack surface** — web app? API? CLI? LLM-powered? Multi-tenant?
4. **Determine scope** — full repo, specific directories, recent changes, or a particular feature

If the codebase has **no LLM/AI components**, still spawn the LLM specialist but note in their task that they should focus on any AI-adjacent risks (e.g., API keys for AI services, potential future AI integration surface) and assist other specialists. If there is truly nothing AI-related, they can contribute as a second pair of eyes on the most complex findings.

### Step 2: Create the Team

```
TeamCreate: team_name = "security-audit"
```

Create one task per specialist plus a synthesis task:

```
TaskCreate: "OWASP Application Security Review"
TaskCreate: "Data Privacy & Exfiltration Review"
TaskCreate: "Exploit Research & Attack Chain Analysis"
TaskCreate: "LLM Vulnerability Assessment"
TaskCreate: "Holistic Security Review & Synthesis"  ← blocked by the above 4
```

The Senior Reviewer's task should be blocked by all other specialists so they can review everyone's findings.

### Step 3: Spawn Specialists in Parallel

Launch 4 specialists simultaneously as `general-purpose` subagents with `team_name = "security-audit"`. Each gets:

1. The scoping context from Step 1 (tech stack, attack surface, key directories)
2. Their specialist role description (copy from the relevant section above)
3. Where to write findings (a unique file per specialist)
4. The findings format (see below)

The 5th specialist (Senior Reviewer) is spawned after the first 4 complete, or can be spawned at the same time with a `blockedBy` dependency so they wait.

### Handling Failures

- **Agent timeout or empty report:** The Senior Reviewer should note the uncovered area in the final report and recommend a follow-up focused review of that dimension. Do not skip synthesis because one specialist failed.
- **Contradictory findings:** When two specialists disagree (e.g., Exploit Researcher says "exploitable" but OWASP Expert says "mitigated"), the Senior Reviewer should present both perspectives with evidence and let the user decide.
- **Large codebase (500+ source files):** Scope each specialist to their most relevant subset of files/directories rather than the full repo. Include the scoping in each agent's prompt so the Senior Reviewer knows what was and wasn't covered.

### Step 4: Findings Format

Create the output directory: `.reports/security-audit/` (add `.reports/` to `.gitignore` if not already there).

Each specialist writes findings to `.reports/security-audit/SECURITY-AUDIT-<specialist>.md`. Use this format:

```markdown
# <Specialist Name> — Security Findings

**Auditor:** <Role name>
**Scope:** <What was reviewed>
**Date:** <Date>

## Critical

### [CRITICAL-01] <Title>
- **Location:** `file/path.ts:123-145`
- **Category:** <e.g., SQL Injection, PII Exposure, Prompt Injection>
- **Description:** <What the vulnerability is>
- **Impact:** <What an attacker could achieve>
- **Attack Scenario:** <Step-by-step exploitation>
- **Evidence:** <Code snippet or trace showing the issue>
- **Remediation:** <Specific fix with code example>

## High
### [HIGH-01] ...

## Medium
### [MEDIUM-01] ...

## Low / Informational
### [LOW-01] ...

## Positive Observations
<Things the codebase does well from this specialist's perspective>
```

**Severity definitions:**
| Severity | Meaning |
|----------|---------|
| **Critical** | Actively exploitable with severe impact (data breach, RCE, full auth bypass). Fix immediately. |
| **High** | Exploitable with significant impact or requires minimal chaining. Fix before next release. |
| **Medium** | Exploitable with moderate impact or requires significant chaining/unlikely conditions. Fix soon. |
| **Low** | Minor issues, defense-in-depth improvements, or informational findings. Fix when convenient. |

### Step 5: Synthesis

The Senior Reviewer reads all findings files, then produces the final `.reports/security-audit/SECURITY-AUDIT.md`:

```markdown
# Security Audit Report

**Repository:** <repo name>
**Date:** <date>
**Specialists:** OWASP, Data Privacy, Exploit Research, LLM Security, Senior Review

## Executive Summary
<3-5 sentence overview: overall posture, critical finding count, top systemic risks>

## Risk Dashboard

| Severity | Count | Categories |
|----------|-------|------------|
| Critical | X     | ...        |
| High     | X     | ...        |
| Medium   | X     | ...        |
| Low      | X     | ...        |

## Critical & High Findings (Deduplicated & Ranked)

<Merged findings from all specialists, deduplicated, with cross-references.
 Each finding attributed to discovering specialist(s).
 Ordered by: severity → exploitability → impact.>

### [C-01] <Title>
- **Found by:** <Specialist(s)>
- **Location:** ...
- **Attack chain:** <If the Exploit Researcher connected this to other findings, include the chain>
- **Remediation:** ...

## Attack Chains
<Exploit Researcher's chained findings, showing how individually-minor issues combine>

## Systemic Patterns
<Senior Reviewer's architectural observations — classes of bugs, missing security layers>

## Remediation Roadmap

### Immediate (Critical)
1. ...

### Short-Term (High)
1. ...

### Medium-Term (Architectural)
1. ...

## Medium & Low Findings
<Remaining findings, briefly summarized with references to specialist reports>

## Positive Security Observations
<What the codebase does well — important for morale and to avoid regressing>

## Appendix: Individual Specialist Reports
- [OWASP Review](./SECURITY-AUDIT-owasp.md) <!-- all files in .reports/security-audit/ -->
- [Data Privacy Review](./SECURITY-AUDIT-privacy.md)
- [Exploit Research](./SECURITY-AUDIT-exploits.md)
- [LLM Security Review](./SECURITY-AUDIT-llm.md)
```

### Step 6: Present & Clean Up

- Present the Executive Summary and Risk Dashboard to the user
- Highlight critical findings that need immediate attention
- Offer to start remediation on specific findings
- Clean up the team with `TeamDelete`
- Leave all files in `.reports/security-audit/` for the user

## Agent Prompt Template

Use this when spawning each specialist:

```
You are a security specialist on a parallel audit team. Your role is:

<ROLE NAME>

<Paste the full specialist description from the relevant section above>

## Codebase Context
- Tech stack: <stack>
- Key directories: <dirs>
- Attack surface: <web app / API / CLI / LLM-powered / etc.>
- Scope: <full repo / specific dirs / specific feature>

## Your Task
1. Claim your task via TaskUpdate (set to in_progress)
2. Systematically review the codebase through your specialist lens
3. For every finding: identify the file and line, assess severity, describe
   the attack scenario, and provide a specific remediation
4. Write your findings to: .reports/security-audit/SECURITY-AUDIT-<your-slug>.md
5. Use the severity format defined in your task description
6. When complete, mark your task as completed

## Important
- Be thorough. Read actual code, don't just grep for patterns.
- Trace data flows end-to-end. Follow user input from entry to storage/output.
- Include code snippets as evidence for every finding.
- Note positive security observations too — what's done well.
- If you find something outside your specialty, still report it with a
  note that another specialist should verify.
```

## Customization

### Adjusting Scope
For smaller codebases or focused audits, the user can:
- Run only 2-3 specialists instead of all 5
- Scope each specialist to specific directories or features
- Skip the LLM specialist if there's no AI component
- Skip synthesis and just read individual reports

### Adding Specialists
For specific domains, additional specialists can be added:
- **Cloud/Infrastructure** — IAM policies, S3 buckets, network ACLs, Kubernetes RBAC
- **Mobile Security** — Certificate pinning, local storage, deeplink hijacking, biometric bypass
- **Cryptography** — Protocol analysis, key management, random number generation, timing attacks
- **Compliance Auditor** — SOC 2 control mapping, HIPAA technical safeguards, PCI DSS requirements

### Quick Mode
For a fast scan instead of a deep audit, spawn only 2 agents:
- OWASP Expert (broadest coverage)
- The Closer (catches what's missed, synthesizes)

Skip the full synthesis — the Senior Reviewer produces the final report directly.

## Model Selection

| Role | Recommended Model | Rationale |
|------|-------------------|-----------|
| OWASP Expert | `sonnet` | Systematic checklist-driven analysis against known categories |
| Data Privacy Specialist | `sonnet` | Data flow tracing is methodical; pattern-matching intensive |
| Exploit Researcher | `opus` | Attack chain construction requires creative reasoning and cross-referencing |
| LLM Vulnerability Specialist | `opus` | Novel attack surface; requires nuanced judgment about emerging threats |
| Senior Holistic Reviewer | `opus` | Must synthesize across all reports, identify gaps, and reason about systemic risk |

Use `opus` for the synthesizer and any specialist that requires cross-referencing or creative reasoning. Use `sonnet` for specialists doing systematic, checklist-driven analysis to reduce cost and latency.

## Skill Chains

These skills are more powerful in combination. Named compositions for common scenarios:

| Chain | Skills (in order) | Use when |
|-------|-------------------|----------|
| **Sanity + Security** | `sanity-check` + `deep-audit` (parallel, scoped to branch) | Security-sensitive feature: check shape AND vulnerabilities before PR |
| **Secure PR Review** | `pr-review-swarm` + `deep-audit` (scoped to changed files, parallel) | Security-sensitive PR — catches both bugs and vulnerabilities |
| **Pre-Launch Audit** | `performance-profiler` + `deep-audit` (parallel) | Preparing for production — covers both performance and security |
| **Full Health Check** | `architecture-forensics` → `deep-audit` + `tech-debt-inventory` + `performance-profiler` (parallel) | Inherited codebase or annual assessment |
| **Post-Incident Review** | `deep-audit` (scoped to affected area) → `competitive-swarm` (remediation approaches) | After a security incident — audit then explore fixes |

When chaining, pass the previous skill's output as context to the next. For example, `architecture-forensics`'s trust boundary map makes `deep-audit`'s specialists significantly more effective.

## Tips

- **Authorize the context.** This skill is for defensive security auditing of the user's own codebase. All specialists operate as authorized reviewers.
- **Biggest repos first.** Start specialists on the largest/most complex directories so they use their time efficiently.
- **Don't deduplicate too early.** Overlapping findings from different specialists often reveal different facets of the same systemic issue — that's valuable signal, not noise.
- **The Closer is the most important agent.** They see what nobody else sees. Give them access to all other findings.
- **Offer remediation.** After the audit, offer to fix critical/high findings. The competitive-swarm skill can be used to explore different remediation approaches if the fix involves architectural trade-offs.
