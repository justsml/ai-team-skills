---
name: architecture-forensics
description: Map and analyze a codebase's actual architecture using a parallel team of specialists — dependency mapping, data flow tracing, boundary analysis, and pattern detection — to produce a comprehensive architecture map and health assessment.
---

# Architecture Forensics

Map and analyze a codebase's real architecture — not what the README claims, but what the code actually does. A parallel team of specialists each investigates a different structural dimension, then a cartographer synthesizes findings into a unified architecture map and health assessment. Use this skill when the user needs to understand an unfamiliar codebase, assess architectural health, or prepare for a major refactor.

## When to Use

- User wants to understand an unfamiliar codebase's architecture
- User asks "how is this codebase structured?" or "map the architecture"
- User is onboarding to a new project and needs the big picture
- User wants to assess architectural health, coupling, or modularity
- User is considering a refactor and needs to understand current state first
- User inherits a codebase and needs to get productive quickly
- User says "architecture forensics" or references this skill

## When NOT to Use

- Codebase has < 50 source files — structure is discoverable by reading directly without a team
- User already knows the architecture and is asking about a specific module — answer directly
- The repo is a single script, small CLI tool, or library with obvious structure
- User wants to find bugs or security issues — use `deep-audit` or `pr-review-swarm` instead
- User wants to fix something specific — this skill maps and documents, it doesn't remediate

## The Specialists

Each specialist is a `general-purpose` subagent with deep expertise in their structural dimension. They work in parallel, analyzing the same codebase independently, then findings are synthesized.

### 1. Dependency Mapper

**Focus:** Module relationships and the import/dependency graph.

**Reviews for:**
- **Import graph structure** — Builds the module dependency graph from actual import/require/use statements. Identifies God modules (imported by everything), kitchen-sink modules (import everything), and orphan modules (imported by nothing).
- **Circular dependencies** — Direct cycles (A imports B, B imports A) and transitive cycles (A->B->C->A). Classifies by severity: type-only cycles (usually harmless) vs. runtime cycles (can cause initialization bugs, undefined imports, and subtle ordering issues).
- **Layering violations** — Identifies implicit architectural layers (e.g., routes -> services -> repositories -> models) and finds violations where lower layers import from higher layers, UI components import database code, or utility modules import business logic.
- **Fan-in/fan-out analysis** — Modules with extremely high fan-in (many dependents — risky to change, high blast radius) or fan-out (many dependencies — hard to test in isolation, fragile). These are the structural bottlenecks of the codebase.
- **External dependency usage** — Which third-party packages are actually used, by how many modules, and for what purpose. Identifies over-reliance on single packages, packages used for a single function that could be inlined, and whether externals are wrapped (good for swappability) or used directly everywhere.
- **Package/module boundaries** — If the repo uses workspaces, packages, or directory-based modules: are the boundaries real (clean interfaces, explicit exports) or porous (reaching into `../other-package/src/internal/helper`)?

**Methodology:** Parses import/require statements across all source files. Builds an adjacency list. Computes fan-in, fan-out, and cycle detection. Groups modules by directory to infer layers, then checks for cross-layer violations.

---

### 2. Data Flow Archaeologist

**Focus:** How data enters, transforms, persists, and exits the system.

**Reviews for:**
- **Entry points** — Every way data enters the system: HTTP routes/handlers, WebSocket handlers, CLI argument parsers, message queue consumers, cron/scheduled jobs, file watchers, event listeners. Maps the full inventory with their input shapes.
- **Transformation chains** — How data is validated, parsed, transformed, and enriched as it flows through the system. Middleware stacks, service layers, mapper/transformer functions, serialization/deserialization boundaries.
- **Persistence layer** — Where and how data is written: database models/schemas, cache operations, file writes, session stores, queue publishing. Maps the data model from actual code usage (not just schema files, which may be outdated).
- **Exit points** — Every way data leaves the system: HTTP responses, WebSocket emissions, email/notification sends, webhook calls, third-party API calls, log output, file exports, analytics events.
- **Implicit data flows** — Data that flows through side channels: global/module-level state, environment variables read at runtime, shared caches, database triggers, event buses, pub/sub systems. These are the flows that don't show up in function signatures.
- **Data shape evolution** — How the shape of data changes as it flows through layers. Where are transformations lossy? Where is data re-fetched instead of passed through? Where does the same entity have multiple conflicting representations (e.g., `User`, `UserDTO`, `UserResponse` with different field sets)?

**Methodology:** Starts from entry points (HTTP handlers, CLI parsers, event listeners) and traces data forward through the call chain to persistence and exit. Then starts from exit points and traces backward. Maps the transformation steps in between.

---

### 3. Boundary & Contract Analyst

**Focus:** Trust boundaries, module contracts, and interface design.

**Reviews for:**
- **Trust boundaries** — Where does trusted internal code meet untrusted input? Public API surface, admin vs. user boundaries, service-to-service interfaces, authenticated vs. unauthenticated zones. Maps each boundary and how it's enforced (middleware, decorators, manual checks, nothing).
- **Explicit contracts** — TypeScript interfaces/types at module boundaries, API schemas (OpenAPI, GraphQL SDL, protobuf), validation schemas (Zod, Joi, JSON Schema), database schemas. Are they comprehensive and up to date?
- **Implicit contracts** — Assumptions modules make about their inputs that aren't encoded in types or validation. Functions that assume non-null but don't check. API handlers that assume a specific body shape. Services that assume database rows exist without verifying.
- **Boundary enforcement** — Are module boundaries enforced (package.json exports field, eslint-plugin-boundaries, barrel files with explicit re-exports) or advisory (just directory conventions that anyone can violate)?
- **Shared mutable state** — Global variables, singleton instances, module-level state, shared caches without proper isolation. Maps what's shared and who mutates it.
- **Configuration boundaries** — How is configuration loaded and distributed? Single source of truth or scattered env reads? Type-safe config or stringly-typed? Consistent between environments?
- **Error boundaries** — How do errors propagate across module boundaries? Are they translated at each layer (domain errors -> HTTP errors) or do database errors bubble raw to the API response?

**Methodology:** Identifies every module/package/service boundary in the system. For each: what's the explicit interface? What implicit assumptions exist? How is the boundary enforced? What happens when a contract is violated?

---

### 4. Pattern Detector

**Focus:** The actual recurring patterns in the codebase — vs. what documentation claims.

**Reviews for:**
- **State management patterns** — How is application state managed? Redux/Zustand/Jotai/Context? Server state (React Query, SWR)? Server-side state: in-memory, Redis, database? Are there competing patterns?
- **Error handling patterns** — Try/catch, Result types, error codes, exception hierarchies, error boundaries (React). Is there a consistent strategy or does each module do it differently?
- **Async patterns** — Callbacks, promises, async/await, observables, event emitters. Which dominates? Are there transitions in progress (callbacks being migrated to async/await)?
- **Data access patterns** — Direct SQL, ORM (Prisma, TypeORM, Sequelize), repository pattern, active record, data mapper, raw queries. One pattern or several?
- **Auth patterns** — Middleware-based, decorator-based, per-route manual checks, centralized policy engine, ad-hoc. Consistent or fragmented?
- **Testing patterns** — Test structure, mocking strategy (jest.mock, dependency injection, test doubles), fixtures/factories/inline data, integration test approach.
- **Configuration patterns** — Environment variables, config files, feature flags, dependency injection containers, hardcoded values.
- **Competing paradigms** — The most important finding: where does the codebase have two or more ways of doing the same thing? Examples: half callbacks / half async-await, two HTTP client libraries, mixed validation strategies, ORM and raw SQL. This indicates either a migration in progress or accumulated inconsistency.
- **Pattern evolution** — Archaeological layers visible in the code. Older code using one pattern, newer code using another. Reveals the team's direction and where legacy cleanup is needed.

**Methodology:** Samples code across all major directories. For each cross-cutting concern (error handling, data access, auth, config, etc.), catalogs every pattern variant with file counts. Identifies the dominant pattern vs. outliers. Looks for chronological patterns via git history or code style differences.

---

### 5. Architecture Cartographer (Synthesizer)

**Focus:** Independent architectural assessment PLUS synthesis of all specialist findings into a unified map.

**Independent review (done BEFORE reading specialist reports):**
- **System-level architecture classification** — Is this a monolith, modular monolith, microservices, serverless, or hybrid? What's the deployment model? This framing shapes how all specialist findings are interpreted.
- **Missing architectural layers** — Are there expected layers for this type of system that don't exist? (e.g., a web app with no caching layer, an API with no validation middleware, a multi-tenant system with no isolation layer)
- **"Nobody owns this" zones** — Code that sits between modules, doesn't clearly belong to any team or domain, and has no clear owner. These are where bugs accumulate and where changes are most dangerous.

**Synthesis (done AFTER reading specialist reports):**
- Cross-references dependency structure with data flows with boundary analysis with pattern inventory to build the unified map.
- Identifies the top architectural risks, the key load-bearing abstractions (highest fan-in modules), and the overall structural health.
- **What everyone else missed** — Reviews all specialist findings for gaps. Asks: "if I were a new developer reading these reports, what would I still not understand about how this system works?"
- Produces the "architecture onboarding guide" — the 10 things a new developer must know to be productive.

**Methodology:** First does a brief independent assessment of the system's overall architecture by reading entry points, top-level config, and directory structure. Then reads all specialist reports, cross-references findings across dimensions, and synthesizes into the unified map.

---

## Workflow

### Step 1: Scope the Analysis

Before spawning the team, understand the codebase:

1. **Read top-level configuration:** package.json, tsconfig.json, Makefile, Dockerfile, docker-compose.yml, README, etc.
2. **Map the directory structure:** `ls` key directories, understand the layout
3. **Identify the tech stack:** language(s), framework(s), database, key libraries
4. **Identify the system type:** web app, API server, CLI tool, library, monorepo, microservices
5. **Determine scope:** full repo, specific packages/services, or particular subsystem

### Step 2: Create the Team

```
TeamCreate: team_name = "architecture-forensics"
```

Create one task per specialist:

```
TaskCreate: "Dependency Mapping"
TaskCreate: "Data Flow Analysis"
TaskCreate: "Boundary & Contract Analysis"
TaskCreate: "Pattern Detection"
TaskCreate: "Architecture Synthesis"  ← blocked by the above 4
```

### Step 3: Spawn Specialists in Parallel

Launch 4 specialists simultaneously as `general-purpose` subagents with `team_name = "architecture-forensics"`. Each gets:

1. The scoping context from Step 1 (tech stack, directory structure, system type)
2. Their specialist role description (copy from the relevant section above)
3. Key starting points to investigate (entry points, main directories, config files)
4. Where to write findings (a unique file per specialist)
5. The findings format (see below)

The 5th agent (Architecture Cartographer) is spawned after the first 4 complete, or at the same time with a `blockedBy` dependency.

### Handling Failures

- **Agent timeout or empty report:** The Cartographer should note the uncovered dimension in the final map and recommend a follow-up. A partial architecture map is still valuable — don't block synthesis because one specialist failed.
- **Contradictory findings:** When specialists disagree (e.g., Boundary Analyst says boundaries are enforced but Pattern Detector finds cross-boundary imports), the Cartographer should investigate the discrepancy — it often reveals the most interesting architectural insight.
- **Large codebase (500+ source files):** Scope each specialist to the most architecturally significant modules. The Dependency Mapper should focus on the top-level module graph (not every utility file). Include scoping in each prompt.

### Step 4: Findings Format

Create the output directory: `.reports/architecture-forensics/` (add `.reports/` to `.gitignore` if not already there).

Each specialist writes findings to `.reports/architecture-forensics/ARCHITECTURE-<specialist>.md`. Use this format:

```markdown
# <Specialist Name> — Architecture Findings

**Analyst:** <Role name>
**Repository:** <repo name>
**Scope:** <What was analyzed>
**Date:** <Date>

## Key Findings

### [FINDING-01] <Title>
- **Category:** <e.g., Circular Dependency, Missing Boundary, Competing Patterns>
- **Severity:** Critical / Significant / Moderate / Informational
- **Location:** <files/modules involved>
- **Description:** <What was found>
- **Impact:** <Why this matters for maintainability, reliability, or development velocity>
- **Evidence:** <Code snippets, import chains, data flow traces>
- **Recommendation:** <Specific architectural improvement>

## Maps & Inventories
<Specialist-specific maps: dependency graph, data flow diagram, boundary map,
pattern inventory — whatever is most useful for this specialist's domain>

## Health Assessment
<Specialist's perspective on architectural health in their dimension>
```

**Severity definitions:**

| Severity | Meaning |
|----------|---------|
| **Critical** | Actively causing bugs or blocking development. Circular dependencies causing initialization failures, missing boundaries allowing data corruption, no consistent patterns making onboarding painful. |
| **Significant** | Degrading development velocity or reliability. High coupling making changes risky, inconsistent patterns causing confusion, missing contracts leading to integration bugs. |
| **Moderate** | Worth addressing but not urgent. Minor layering violations, slightly inconsistent patterns, optimization opportunities in module structure. |
| **Informational** | Observations about the architecture that aren't problems but inform understanding. Good for the onboarding guide. |

### Step 5: Synthesis

The Architecture Cartographer reads all findings files, then produces the final `.reports/architecture-forensics/ARCHITECTURE-MAP.md`:

```markdown
# Architecture Map — <Repository Name>

**Repository:** <repo name>
**Date:** <date>
**Analysts:** Dependency Mapper, Data Flow Archaeologist, Boundary Analyst, Pattern Detector

## System Overview
<3-5 sentences: what this system is, its primary purpose, scale, and overall
architectural style (monolith, modular monolith, microservices, etc.)>

## Architecture Diagram
<Text-based representation of the high-level architecture: major components,
data flows between them, external dependencies, entry/exit points.
Use indentation, arrows (->), and grouping to make it scannable.>

## Key Components
<For each major module/package/service: what it does, what it depends on,
what depends on it, its boundary type (explicit interface / implicit / porous).
Ordered by architectural importance (highest fan-in first).>

## Data Flow Summary
<How data enters, flows through, and exits the system. Key transformation
points. Major data paths for the primary use cases.>

## Architectural Health Dashboard

| Dimension | Health | Key Issue |
|-----------|--------|-----------|
| Coupling | red/yellow/green | <one-line summary> |
| Boundaries | red/yellow/green | <one-line summary> |
| Consistency | red/yellow/green | <one-line summary> |
| Data Flow | red/yellow/green | <one-line summary> |

## Top Architectural Risks
<Ranked list of the most impactful architectural issues, cross-referencing
specialist findings>

### [RISK-01] <Title>
- **Found by:** <Specialist(s)>
- **Impact:** <Why this matters>
- **Recommendation:** <Architectural change>

## Load-Bearing Abstractions
<The modules that everything depends on — highest fan-in. These are the
riskiest to change and the most important to understand. Includes fan-in
counts and a note on their interface quality.>

## Competing Patterns & Migration State
<Where the codebase has two ways of doing the same thing. Which is the
"new way" vs. "old way" based on code age? What percentage has migrated?
What's blocking completion?>

## Onboarding Guide: 10 Things to Know
<The 10 most important things a new developer needs to understand about
this codebase's architecture. Not how-to-run — how-it-works. Ordered by
"you'll hit this first" to "you'll need this eventually.">

## Recommendations

### Quick Wins
<Low-effort improvements with high clarity/velocity impact>

### Strategic Improvements
<Larger refactors that would meaningfully improve architectural health>

## Appendix: Specialist Reports
- [Dependency Map](./ARCHITECTURE-dependencies.md)
- [Data Flow Analysis](./ARCHITECTURE-dataflow.md)
- [Boundary & Contract Analysis](./ARCHITECTURE-boundaries.md)
- [Pattern Inventory](./ARCHITECTURE-patterns.md)
```

### Step 6: Present & Clean Up

- Present the System Overview, Health Dashboard, and Top Risks to the user
- Highlight the Onboarding Guide as the fastest way to get oriented
- Offer to dive deeper into any specific finding or component
- Clean up the team with `TeamDelete`
- Leave all files in `.reports/architecture-forensics/` for the user

## Agent Prompt Template

Use this when spawning each specialist:

```
You are an architecture analysis specialist on a parallel forensics team. Your role is:

<ROLE NAME>

<Paste the full specialist description from the relevant section above>

## Codebase Context
- Tech stack: <stack>
- System type: <web app / API / CLI / library / monorepo>
- Key directories: <dirs>
- Entry points: <main files, route handlers, CLI entry>

## Your Task
1. Claim your task via TaskUpdate (set to in_progress)
2. Systematically analyze the codebase through your specialist lens
3. Read actual code — don't just look at filenames and directory names
4. For every finding: identify the affected modules, assess severity, describe
   the architectural impact, and provide a specific recommendation
5. Write your findings to: .reports/architecture-forensics/ARCHITECTURE-<your-slug>.md
6. Include maps/inventories specific to your specialty (dependency graph, data
   flow map, boundary catalog, pattern inventory)
7. When complete, mark your task as completed

## Important
- Be empirical. Base findings on actual code, not assumptions from file names.
- Include evidence: import chains, code snippets, file counts, specific paths.
- This is forensics — document what IS, not what SHOULD BE. Recommendations
  come after the findings.
- If you discover something outside your specialty, note it for other specialists.
- "Informational" findings are valuable — architecture understanding doesn't
  require everything to be a problem.
```

## Customization

### Adjusting Scope

- For monorepos, scope each specialist to analyze cross-package relationships — not just within packages, but how packages depend on and communicate with each other.
- For microservices, add a **Service Topology Mapper** that maps service-to-service communication, shared databases, API contracts between services, and deployment dependencies.
- For frontend-heavy codebases, add a **Component Architecture Analyst** that maps component hierarchy, prop drilling depth, state management boundaries, render tree structure, and code splitting points.
- For small codebases (< 50 files), run only 2 specialists: Pattern Detector + Boundary Analyst (most insight for least overhead).

### Quick Mode

Spawn only the Pattern Detector and Dependency Mapper. The lead synthesizes. Good for a quick architectural orientation without the full forensics treatment.

### Adding Specialists

- **Service Topology Mapper** — For microservices: service registry, communication patterns (sync/async/events), shared databases, circuit breakers, deployment dependency graph.
- **Component Architecture Analyst** — For frontend: component tree depth, prop drilling, state management boundaries, render optimization, code splitting, route structure.
- **Infrastructure Mapper** — Docker/K8s configuration, CI/CD pipeline structure, environment topology, deployment flow, secrets management.
- **API Surface Cataloger** — REST/GraphQL/gRPC endpoints, versioning strategy, deprecation status, documentation completeness, response consistency.

## Model Selection

| Role | Recommended Model | Rationale |
|------|-------------------|-----------|
| Dependency Mapper | `sonnet` | Mechanical import graph analysis, counting, cycle detection |
| Data Flow Archaeologist | `opus` | Tracing implicit data flows requires reasoning about side effects and indirection |
| Boundary & Contract Analyst | `opus` | Identifying implicit contracts requires understanding assumptions not written in code |
| Pattern Detector | `sonnet` | Cataloging patterns is systematic sampling and comparison |
| Architecture Cartographer | `opus` | Synthesis requires cross-referencing all dimensions and producing the onboarding guide |

The Dependency Mapper and Pattern Detector are the most mechanical — `sonnet` handles them well. The Boundary Analyst and Data Flow Archaeologist need `opus` because implicit assumptions and hidden data flows require deeper reasoning.

## Skill Chains

| Chain | Skills (in order) | Use when |
|-------|-------------------|----------|
| **Full Health Check** | `architecture-forensics` → `deep-audit` + `tech-debt-inventory` + `performance-profiler` (parallel) | Inherited codebase, annual assessment |
| **Architecture → Sanity** | `architecture-forensics` → `sanity-check` | Unfamiliar codebase: map the architecture, then sanity-check your changes against it |
| **Pre-Migration Assessment** | `architecture-forensics` → `tech-debt-inventory` | Planning a major framework upgrade or monolith decomposition |
| **Informed Security Audit** | `architecture-forensics` → `deep-audit` | Feed trust boundary map and data flow map to security specialists |
| **Onboarding Package** | `architecture-forensics` (standalone) → share `ARCHITECTURE-MAP.md` with new developers | New team member joining the project |

When chaining into other skills, pass the Architecture Map's "Key Components", "Data Flow Summary", and "Onboarding Guide" sections as context to downstream specialists. This saves them from re-discovering the structure.

## Tips

- Run this skill before `deep-audit` — understanding the architecture makes security findings more actionable and helps specialists identify trust boundaries faster.
- The Pattern Detector's "competing paradigms" finding is often the most actionable — it tells you where the codebase is mid-transition and where to focus consistency efforts.
- The Onboarding Guide in the synthesis is valuable beyond onboarding — it's the living document of architectural decisions that teams usually lack.
- For inherited codebases, this skill replaces weeks of manual code exploration.
- The Dependency Mapper's fan-in analysis reveals the "load-bearing" code — modules where a bug or breaking change cascades everywhere. Handle these with care.
- Don't run this on a codebase you already know well — it's most valuable for unfamiliar territory or for documenting what's only in people's heads.
- Pair with `tech-debt-inventory` for a complete codebase health picture: architecture-forensics maps the structure, tech-debt-inventory quantifies what's wrong within it.
