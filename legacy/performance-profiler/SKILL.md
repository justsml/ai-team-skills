---
name: performance-profiler
description: Analyze codebase performance using a parallel team of specialists — algorithmic complexity, I/O and database patterns, memory and lifecycle management, and frontend payload optimization — to identify bottlenecks and produce a ranked optimization plan.
---

# Performance Profiler

Analyze a codebase's performance characteristics through four specialist lenses in parallel, then synthesize findings into a ranked optimization plan with estimated impact. Use this skill when the user wants to find bottlenecks, prepare for scale, or systematically improve performance rather than guessing.

## When to Use

- User asks "where are the performance bottlenecks?" or "why is this slow?"
- User wants a performance review or optimization analysis
- User is preparing for a load test or scale-up and wants to find issues proactively
- User asks about database query performance, memory leaks, or bundle size
- User reports sluggish behavior and wants systematic analysis, not guessing
- User says "performance profiler", "performance audit", or "performance analysis"

## When NOT to Use

- The system has no production traffic and no plans for scale — optimization is premature
- User's actual problem is a bug (wrong behavior), not speed — use `pr-review-swarm` instead
- Codebase is a rarely-run script or batch job where execution time doesn't matter
- User already has profiling data and knows the specific bottleneck — just fix it directly
- The codebase is < 20 files with no database, no network calls, and no frontend — a single agent can cover it

## The Specialists

Each specialist is a `general-purpose` subagent focused on a distinct performance dimension. They work in parallel, analyzing the same codebase independently, then findings are synthesized.

### 1. Algorithmic Analyst

**Focus:** Computational complexity — the O(n) of it all.

**Reviews for:**
- **Hidden quadratic behavior** — Nested loops over the same dataset, array `.includes()` or `.find()` inside a loop (O(n^2) when a Set/Map lookup would be O(1)), repeated `.filter()` chains, sorting inside iterations.
- **Repeated computation** — Expensive operations called multiple times with the same inputs that should be memoized/cached. Recalculating derived values on every render/request. Computing aggregates on every call instead of maintaining running counts.
- **Inefficient data structure choices** — Arrays used for frequent lookups (should be Maps/Sets), linear scans where binary search or indexed access would work, linked lists where arrays would have better cache locality.
- **Recursive algorithms without memoization** — Tree traversals, graph algorithms, dynamic programming patterns that recompute subproblems. Fibonacci-style recursion in disguise.
- **String manipulation in hot paths** — Repeated string concatenation in loops (vs. array join), regex compilation inside loops (should be pre-compiled), unnecessary serialization/deserialization cycles (JSON.parse/stringify roundtrips).
- **Sorting overkill** — Sorting entire arrays to find min/max (use a single pass), sorting before filtering (filter first, sort fewer items), sorting when only top-k items are needed (use a partial sort or heap).
- **Context-specific anti-patterns:**
  - **API handlers:** Linear scans over all records, missing pagination, unbounded query results returned to client.
  - **React components:** Re-rendering lists without stable keys, computing derived state in render, filtering/sorting large arrays on every render.
  - **Background jobs:** Processing items one-by-one that could be batched, no chunking for large datasets.

**Methodology:** Identifies hot paths (request handlers, event loops, frequently-called functions, render paths). For each, traces the computational complexity — counting loops, branching, and data structure operations. Annotates with Big-O notation. Ranks by severity (worst complexity) multiplied by frequency (how often the path runs).

---

### 2. I/O & Query Analyst

**Focus:** Database queries, network calls, file I/O — the slowest operations in any system.

**Reviews for:**
- **N+1 queries** — Fetching a list, then looping to fetch related data one-by-one. The classic ORM anti-pattern. Detects by looking for database calls inside loops, `.map()`, `.forEach()`.
- **Missing database indexes** — Queries with WHERE, ORDER BY, or JOIN conditions on columns that aren't indexed. Cross-references query patterns with schema/migration files.
- **Sequential I/O that could be parallel** — Sequential `await` calls to independent resources. `await fetch(a); await fetch(b);` should be `await Promise.all([fetch(a), fetch(b)])`.
- **Unbatched operations** — Multiple individual inserts/updates that could be a single batch operation. Multiple individual API calls that could use a batch endpoint.
- **Over-fetching** — `SELECT *` when only 2 columns are needed, fetching entire objects to check a boolean field, loading full related objects when only IDs are needed.
- **Missing connection pooling** — Creating new database connections per request instead of using a pool. Repeated TCP handshakes and TLS negotiation.
- **No caching layer** — Identical expensive queries or API calls made repeatedly with no caching. Missing HTTP cache headers on static or rarely-changing responses.
- **Synchronous I/O on request paths** — File reads/writes using synchronous APIs (`readFileSync`), blocking the event loop. DNS lookups, certificate loading, or config file reads on every request instead of at startup.
- **Chatty service communication** — Multiple fine-grained calls between services that could be a single coarse-grained call. Lack of aggregation or BFF pattern.
- **Missing pagination** — Endpoints returning unbounded result sets. Database queries without LIMIT. Streaming without backpressure.
- **Connection/resource leaks** — Database connections not released, file handles not closed, HTTP connections not terminated. Often hidden in error paths (the connection is released in the happy path but not in the catch block).

**Methodology:** Identifies all database access, HTTP client calls, file I/O, and cache operations. Traces call patterns — especially inside loops. Reads database schema/migrations to check for index coverage. Maps the I/O dependency graph for each request handler to identify parallelism opportunities.

---

### 3. Memory & Lifecycle Analyst

**Focus:** Memory usage, object lifecycle, cleanup, and leaks.

**Reviews for:**
- **Event listener leaks** — Listeners added but never removed. Common in: WebSocket handlers without cleanup, DOM event listeners in React without useEffect cleanup, EventEmitter subscriptions in long-lived processes.
- **Growing collections without bounds** — In-memory caches without TTL or LRU eviction, maps/arrays that accumulate entries over the lifetime of the process, Set collections that only grow.
- **Closure retention** — Closures capturing large objects that keep them in memory long after needed. Common in: callbacks that reference entire request/response objects, timers that close over large datasets, event handlers that capture parent scope.
- **Missing cleanup on teardown** — Components that set up intervals/subscriptions without clearing them in useEffect cleanup. Server handlers that allocate resources without release. Worker threads that aren't terminated.
- **Large objects in memory** — Entire files loaded into memory instead of streamed, large JSON blobs parsed and held, entire database result sets materialized when only a few rows are needed.
- **Buffer/stream misuse** — Buffering entire streams into memory before processing, not using pipe/transform streams, allocating new buffers for every operation instead of reusing.
- **Object allocation in hot paths** — Creating new objects, arrays, or closures on every function call in performance-critical loops. Could use object pooling, pre-allocated buffers, or avoid allocation entirely.
- **Weak reference opportunities** — Caches that should use WeakRef/WeakMap to allow garbage collection but instead use strong references that prevent it.
- **Global state accumulation** — Module-level variables that accumulate data over time in long-running processes (servers, workers). Memory usage grows linearly with uptime.
- **Circular references** — Objects referencing each other in ways that can prevent garbage collection or cause issues with serialization and debugging tools.

**Methodology:** Identifies all long-lived objects (module-level state, singletons, caches, stores). For each: is it bounded? Is there cleanup? Can it grow without limit? Then examines component/handler lifecycle: setup matched with teardown? Subscriptions matched with unsubscriptions? Resources acquired matched with released?

---

### 4. Frontend Payload Analyst

**Focus:** What's sent to the browser — bundle size, loading performance, render efficiency.

**Note:** Only applicable if the codebase has a frontend. If the codebase is purely backend/CLI/library, this specialist should pivot to analyzing response payload sizes, serialization efficiency, and compression. They can also serve as a second pair of eyes on I/O patterns.

**Reviews for (frontend):**
- **Bundle size** — Total JS/CSS shipped, largest chunks, duplicate packages in the bundle, packages that could be dynamically imported instead of statically included.
- **Un-tree-shaken imports** — `import _ from 'lodash'` (imports everything) vs. `import get from 'lodash/get'` (imports one function). Default imports from barrel files that defeat tree-shaking.
- **Missing code splitting** — Route-based splitting not configured, large components not lazy-loaded, above-the-fold and below-the-fold code in the same chunk.
- **Unoptimized assets** — Images not compressed or not in modern formats (WebP/AVIF), SVGs not optimized, fonts loading unused character sets, videos without proper compression.
- **Critical render path** — Render-blocking CSS/JS, large above-the-fold content that could be inlined, missing preload/prefetch hints for critical resources.
- **CSS bloat** — Unused CSS rules, duplicate styles, overly specific selectors, Tailwind without purge configured.
- **Third-party script impact** — Analytics, ads, chat widgets, A/B testing scripts. How many? Deferred? Total size? Do they block rendering?
- **React-specific (if applicable):** Unnecessary re-renders from missing memo/useMemo/useCallback, large component trees re-rendering from small state changes, Context providers causing re-renders in unrelated consumers, missing Suspense boundaries.

**Reviews for (backend-only pivot):**
- **Response payload size** — Responses larger than necessary, unused fields returned, missing partial response support (field selection, sparse fieldsets).
- **Serialization efficiency** — JSON.stringify on hot paths, opportunities for schema-based serialization, repeated serialization of the same objects.
- **Compression** — Is gzip/brotli enabled? Are responses too small to benefit (overhead exceeds savings)?

**Methodology:** Analyzes build configuration (webpack, vite, next.config, etc.) for code splitting and tree-shaking settings. Reads import statements to identify bundle-busting patterns. Examines component structure for render efficiency. Catalogs all third-party scripts and assets.

---

### 5. Performance Architect (Synthesizer)

**Focus:** Independent critical path analysis PLUS synthesis of all specialist findings into a prioritized optimization plan.

**Independent review (done BEFORE reading specialist reports):**
- **Critical path mapping** — Traces the 3-5 most important user-facing operations end-to-end (e.g., "user loads dashboard" = HTTP request → auth middleware → 3 database queries → template render → response). This map frames all specialist findings.
- **Compounding bottlenecks** — A slow query (I/O) inside a loop (algorithmic) that allocates large objects (memory) is worse than the sum of its parts. Look for paths where multiple performance dimensions compound.
- **Missing performance infrastructure** — Is there a caching layer? Connection pooling? CDN? Query monitoring? Each missing layer is a systemic performance issue that individual specialists may not flag.

**Synthesis (done AFTER reading specialist reports):**
- Maps specialist findings onto the critical paths — which findings affect which user-facing operations?
- Deduplicates findings across specialists (the Algorithmic Analyst and I/O Analyst may both flag the same loop from different angles).
- Ranks optimizations by estimated impact multiplied by ease of implementation.
- Identifies "free wins" (trivial changes with outsized impact) and "strategic wins" (larger changes needed for scale).
- **What everyone else missed** — Reviews all specialist findings for performance issues that span multiple dimensions. Asks: "what's the single change that would improve the most critical paths?"

**Methodology:** First traces the critical user-facing paths end-to-end by reading route handlers and following the call chain. Then reads all specialist reports, maps findings onto critical paths, and produces the prioritized optimization plan.

---

## Workflow

### Step 1: Scope the Analysis

Before spawning the team, understand the codebase:

1. **Read the codebase structure** — directory layout, tech stack, build configuration
2. **Identify the system type:** web app (has frontend), API server, CLI tool, library, worker/queue processor
3. **If web app:** identify the bundler, framework, and deployment model (SSR, SSG, SPA, edge)
4. **If API:** identify the framework, database, ORM, and caching layer
5. **Identify known performance concerns** from the user (if any)
6. **Determine if the Frontend Payload Analyst applies** (frontend exists) or should pivot to response efficiency

### Step 2: Create the Team

```
TeamCreate: team_name = "performance-profiler"
```

Create one task per specialist:

```
TaskCreate: "Algorithmic Complexity Analysis"
TaskCreate: "I/O & Query Analysis"
TaskCreate: "Memory & Lifecycle Analysis"
TaskCreate: "Frontend Payload Analysis"
TaskCreate: "Performance Synthesis"  ← blocked by the above 4
```

### Step 3: Spawn Specialists in Parallel

Launch 4 specialists simultaneously as `general-purpose` subagents with `team_name = "performance-profiler"`. Each gets:

1. The scoping context from Step 1 (tech stack, system type, known concerns)
2. Their specialist role description (copy from the relevant section above)
3. Key files to investigate (entry points, hot paths, build config, schema files)
4. Where to write findings (a unique file per specialist)
5. The findings format (see below)

If no frontend exists, tell the Frontend Payload Analyst to pivot to response efficiency analysis.

The 5th agent (Performance Architect) is spawned after the first 4 complete, or at the same time with a `blockedBy` dependency.

### Handling Failures

- **Agent timeout or empty report:** The Performance Architect should note the uncovered dimension in the final report and recommend a follow-up. Do not block synthesis because one specialist failed — the other dimensions are still valuable.
- **Contradictory findings:** When specialists disagree (e.g., Algorithmic Analyst says a function is O(n) but I/O Analyst says it's the bottleneck due to query count), the Performance Architect should reconcile — both can be true at different scales.
- **Large codebase (500+ source files):** Scope each specialist to the known hot paths and highest-traffic code. Use entry points (route handlers, queue consumers) as starting points. Include scoping in each prompt.
- **No frontend:** If the codebase has no frontend, redirect the Frontend Payload Analyst prompt to analyze response payload efficiency and serialization instead. Note this in their task description.

### Step 4: Findings Format

Create the output directory: `.reports/performance-profiler/` (add `.reports/` to `.gitignore` if not already there).

Each specialist writes findings to `.reports/performance-profiler/PERFORMANCE-<specialist>.md`. Use this format:

```markdown
# <Specialist Name> — Performance Findings

**Analyst:** <Role name>
**Repository:** <repo name>
**Scope:** <What was analyzed>
**Date:** <Date>

## Summary
<Key metrics and top findings overview>

## Critical Performance Issues

### [CRITICAL-01] <Title>
- **Location:** `file/path.ts:42-58`
- **Category:** <e.g., O(n^2) Loop, N+1 Query, Memory Leak, 2MB Bundle>
- **Hot Path:** <Which user-facing operation this affects>
- **Current Complexity/Cost:** <e.g., O(n^2), 47 queries per request, 3.2MB parsed>
- **Expected at Scale:** <What happens with 10x data/users>
- **Evidence:** <Code snippet showing the problem>
- **Optimization:** <Specific fix with estimated improvement>
- **Effort:** <Trivial / Small / Medium / Large>

## Significant Performance Issues
### [SIGNIFICANT-01] ...

## Moderate Issues
### [MODERATE-01] ...

## Free Wins
<Trivial changes (< 30 minutes) with outsized impact. List every one found.>

## Hot Path Map
<For this specialist's domain: which paths are hottest and where do they bottleneck?>
```

**Severity definitions:**

| Severity | Meaning |
|----------|---------|
| **Critical** | Will cause visible degradation at current or near-future scale. Response time > 1s, memory growth until OOM, bundle > 3MB. |
| **Significant** | Measurable impact on performance. Suboptimal but not crisis-level. Will get worse with growth. |
| **Moderate** | Minor inefficiency. Worth fixing when in the area but not a priority. |
| **Free Win** | Trivial effort (< 30 minutes), clear measurable improvement. Do these first. |

### Step 5: Synthesis

The Performance Architect reads all findings files, then produces the final `.reports/performance-profiler/PERFORMANCE-ANALYSIS.md`:

```markdown
# Performance Analysis — <Repository Name>

**Repository:** <repo name>
**Date:** <date>
**Analysts:** Algorithmic, I/O & Query, Memory & Lifecycle, Frontend Payload

## Executive Summary
<3-5 sentences: overall performance posture, biggest bottlenecks, critical scaling risks>

## Critical Path Map
<The 3-5 most important user-facing operations, with their full performance chain:
frontend -> network -> API -> database -> response.
Where does each path spend its time?>

## Performance Dashboard

| Dimension | Health | Top Issue |
|-----------|--------|-----------|
| Algorithms | red/yellow/green | <one-line> |
| I/O & Queries | red/yellow/green | <one-line> |
| Memory | red/yellow/green | <one-line> |
| Frontend/Payload | red/yellow/green | <one-line> |

## Free Wins (Do Immediately)
<All trivial optimizations from all specialists, ranked by estimated impact>

| # | Category | Fix | Effort | Estimated Impact |
|---|----------|-----|--------|-----------------|
| FW-01 | I/O | Add index on `users.email` | 10 min | 50ms -> 2ms lookup |
| FW-02 | Algorithm | Replace array.includes with Set.has in processItems | 15 min | O(n^2) -> O(n) |

## Critical & Significant Findings (Ranked)

### [P-01] <Title>
- **Found by:** <Specialist(s)>
- **Hot Path:** <affected operation>
- **Issue:** <description>
- **Scale Risk:** <what happens at 10x>
- **Fix:** <optimization>
- **Effort:** <estimate>

## Scaling Risk Assessment
<Which operations will break first as the system grows? What are the scaling
bottlenecks and at what approximate scale do they become critical?>

## Optimization Roadmap

### Immediate: Free Wins
<List from above>

### Short-Term: Critical Fixes
<The 3-5 most impactful optimizations>

### Medium-Term: Architectural Performance Improvements
<Caching layers, query optimization patterns, bundle splitting strategy,
connection pooling, streaming architecture, etc.>

### Monitoring Recommendations
<What to measure going forward: specific metrics, key queries to watch,
memory baselines, bundle size budgets, alerting thresholds>

## Positive Observations
<What performs well. Patterns that should be preserved.>

## Appendix: Specialist Reports
- [Algorithmic Analysis](./PERFORMANCE-algorithms.md)
- [I/O & Query Analysis](./PERFORMANCE-io.md)
- [Memory & Lifecycle Analysis](./PERFORMANCE-memory.md)
- [Frontend Payload Analysis](./PERFORMANCE-frontend.md)
```

### Step 6: Present & Clean Up

- Present the Executive Summary, Critical Path Map, and Free Wins to the user
- Highlight scaling risks that may not be visible at current scale but will appear with growth
- Offer to implement Free Wins immediately
- Offer to implement specific optimizations from the ranked findings
- Clean up the team with `TeamDelete`
- Leave all files in `.reports/performance-profiler/` for the user

## Agent Prompt Template

Use this when spawning each specialist:

```
You are a performance analysis specialist on a parallel profiling team. Your role is:

<ROLE NAME>

<Paste the full specialist description from the relevant section above>

## Codebase Context
- Tech stack: <stack>
- System type: <web app / API / CLI / worker>
- Key directories: <dirs>
- Known performance concerns: <if any>
- Frontend: <yes/no — framework, bundler>
- Database: <type, ORM>

## Your Task
1. Claim your task via TaskUpdate (set to in_progress)
2. Identify the hot paths — the most-executed code paths in the system
3. Analyze each hot path through your specialist lens
4. For every finding: identify the exact location, measure/estimate the cost,
   project the scaling behavior, and provide a specific optimization with
   effort estimate
5. Write your findings to: .reports/performance-profiler/PERFORMANCE-<your-slug>.md
6. Identify Free Wins — trivial changes with outsized impact
7. When complete, mark your task as completed

## Important
- Focus on hot paths. A quadratic algorithm that runs once at startup matters
  less than a linear algorithm that runs on every request.
- Be specific about costs. "This is slow" is useless. "This executes 47 queries
  per request, each taking ~5ms, totaling ~235ms on a list of 47 items and
  scaling linearly" is actionable.
- Think about scale. What happens with 10x users, 10x data, 10x concurrent requests?
- Include code snippets as evidence.
- Free Wins are the most valuable output — prioritize finding them.
- Not everything needs optimization. Only flag things that actually impact user
  experience or system stability.
```

## Customization

### Adjusting Scope

- For API-only codebases, replace the Frontend Payload Analyst with a **Response Efficiency Analyst** (serialization overhead, compression, response size, partial response support).
- For real-time systems (WebSocket, streaming), add a **Concurrency Analyst** (connection pooling, backpressure, fan-out efficiency, message batching, queue depth).
- For data-intensive systems, add a **Data Pipeline Analyst** (batch processing efficiency, ETL bottlenecks, parallel processing, checkpointing).
- For small codebases, run only 2 specialists: Algorithmic Analyst + I/O Analyst (most bang for buck).

### Quick Mode

Spawn only the I/O & Query Analyst — database and network operations are almost always the dominant bottleneck. The lead synthesizes directly.

### Adding Specialists

- **Concurrency Analyst** — Thread/worker pool sizing, lock contention, async queue depth, connection pool exhaustion, rate limiting efficiency.
- **Caching Strategist** — Cache hit rates, cache invalidation patterns, caching opportunities, multi-layer cache architecture, cache stampede risk.
- **Rendering Performance Analyst** — For SSR/SSG: server render time, hydration cost, streaming SSR opportunities, partial pre-rendering.
- **Data Pipeline Analyst** — Batch job efficiency, ETL patterns, parallel processing, checkpoint/restart, backfill performance.

## Model Selection

| Role | Recommended Model | Rationale |
|------|-------------------|-----------|
| Algorithmic Analyst | `opus` | Tracing computational complexity through call chains requires deep reasoning |
| I/O & Query Analyst | `sonnet` | N+1 detection, index checking, and sequential I/O are pattern-matching tasks |
| Memory & Lifecycle Analyst | `opus` | Leak detection requires reasoning about object lifetimes and implicit retention |
| Frontend Payload Analyst | `sonnet` | Bundle analysis, import checking, and asset cataloging are systematic |
| Performance Architect | `opus` | Must map critical paths across all dimensions and make prioritization judgments |

The I/O & Query Analyst and Frontend Payload Analyst are pattern-matching heavy — `sonnet` handles them well. The Algorithmic Analyst and Memory Analyst need `opus` because tracing complexity through indirection and reasoning about object lifetimes require deeper analysis.

## Skill Chains

| Chain | Skills (in order) | Use when |
|-------|-------------------|----------|
| **Pre-Launch Audit** | `performance-profiler` + `deep-audit` (parallel) | Preparing for production — performance bottlenecks can also be DoS vectors |
| **Informed Profiling** | `architecture-forensics` → `performance-profiler` | Unfamiliar codebase — map the data flow first, then analyze hot paths |
| **Performance Fix Bake-Off** | `performance-profiler` → `competitive-swarm` (optimize critical findings with Performance vs. Minimalist vs. Memory lenses) | Bottleneck has multiple valid optimization strategies |
| **Full Health Check** | `architecture-forensics` → `deep-audit` + `tech-debt-inventory` + `performance-profiler` (parallel) | Comprehensive codebase assessment |

When chaining from `architecture-forensics`, pass the Data Flow Summary and Key Components to the performance specialists — they'll identify hot paths faster with the architecture map.

## Tips

- I/O is almost always the biggest bottleneck. If you only have time for one specialist, run the I/O & Query Analyst.
- Free Wins have the best ROI. Adding a missing database index takes 10 minutes and can speed up queries by 100x.
- Think about scale, not just current performance. A quadratic algorithm on 100 items is fine. On 10,000 items it's not.
- Pair with `deep-audit` when analyzing externally-facing APIs — some performance issues are also denial-of-service vectors (unbounded queries, missing rate limiting, algorithmic complexity attacks).
- The Critical Path Map in the synthesis is the most valuable artifact — it tells you exactly where time is spent for each user-facing operation.
- For memory issues, focus on long-running processes (servers, workers). Short-lived processes (CLI tools, scripts) rarely have meaningful memory issues.
- Don't optimize prematurely. If the system is fast enough and the code is clear, that's better than fast code nobody can maintain. Only flag things that matter.
- Run `architecture-forensics` first if the codebase is unfamiliar — understanding the data flow and module structure makes performance analysis faster and more accurate.
