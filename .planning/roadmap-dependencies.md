# GBSD Supercharge — Dependency-Ordered Roadmap

**Perspective:** Dependency Ordering
**Granularity:** Standard (5-8 phases)
**Coverage:** 46/46 requirements mapped
**Created:** 2026-03-15

---

## Dependency Ordering Rationale

The phase sequence is not arbitrary. Each phase unlocks the next:

```
Phase 1 (Security + Deprecation)
  └─► Clean execution substrate; no shell injection in worktree code

Phase 2 (Worktree Foundation)
  └─► Isolation primitive everything else rests on
  └─► Prerequisite for: State Reconciler, execute-phase integration

Phase 3 (Key Pool + State Reconciliation)
  └─► Can be built in parallel with Phase 2 design, but depends on Phase 2 interfaces
  └─► Prerequisite for: execute-phase integration

Phase 4 (File Ownership + Decision Frontloading)  ← CORRECTNESS GUARANTEES
  └─► Must land BEFORE the first parallel wave ships
  └─► Without PLAN-01: merge conflict cascade is near-certain
  └─► Without AUTON-01/02: parallel agents pause for human questions mid-wave

Phase 5 (Execute-Phase Integration)
  └─► All prerequisites (worktree, key pool, reconciler, file ownership) are in place
  └─► Wires everything into end-to-end parallel wave execution

Phase 6 (Pathfinder Integration)  ← INDEPENDENT TRACK
  └─► Enhances plan quality but does not block execution (Phase 5 works without it)
  └─► Can start development during Phase 5

Phase 7 (Agent Teams + Model Routing)
  └─► Builds on proven parallel execution
  └─► Multi-provider routing only makes sense after execution validates cost patterns
```

**Critical ordering rule:** File ownership (PLAN-01) and decision frontloading (AUTON-01/02) are the correctness guarantees. Parallel execution that ships without them will produce merge conflicts and mid-wave human interruptions. These must complete before Phase 5 (execute-phase integration) ships.

---

## Phases

- [ ] **Phase 1: Security Foundation and Deprecation** - Fix shell injection, remove /gbsd:map-codebase, establish safe substrate for parallel execution
- [ ] **Phase 2: Worktree Foundation** - Isolation primitive: worktree.cjs with full lifecycle management, stale detection, try/finally cleanup
- [ ] **Phase 3: Key Pool and State Reconciliation** - keypool.cjs (circuit breaker + round-robin) and reconcile.cjs (typed merge strategies), parallel track
- [ ] **Phase 4: File Ownership and Decision Frontloading** - Correctness guarantees: planner file-ownership-first decomposition + discuss-phase decision capture before any parallel wave ships
- [ ] **Phase 5: Execute-Phase Integration** - Wire all components into end-to-end parallel wave execution with merge-back, resume, and pre-flight checks
- [ ] **Phase 6: Pathfinder Integration** - Independent track: pathfinder.cjs adapter, graceful degradation, content-based staleness, 70-85% wave-1 parallelizability
- [ ] **Phase 7: Agent Teams, Model Routing, and Adoption** - Hub-and-spoke team coordination, per-agent model assignment, upstream sync, and adoption infrastructure

---

## Phase Details

### Phase 1: Security Foundation and Deprecation
**Goal:** Establish a safe, clean substrate before parallel execution multiplies the attack surface
**Depends on:** Nothing — this is Phase 1
**Requirements:** DEPR-01, ADOPT-01, ADOPT-02, ADOPT-03
**Dependency reasoning:**
- The existing codebase has a documented shell injection vulnerability in `execSync` git operations (CONCERNS.md). Parallel execution with N worktrees multiplies this attack surface by N. Fix must land first.
- /gbsd:map-codebase must be deprecated before Pathfinder integration (Phase 6) ships — otherwise two competing codebase-mapping systems coexist and documentation/prompts are ambiguous.
- ADOPT-01/02/03 must establish their abstraction layers (especially `worktreeManager.spawnExecutor()`) before any worktree code is written in Phase 2. These are design constraints, not afterthoughts.
**Success Criteria** (what must be TRUE):
  1. All `execSync` git operations in gbsd-tools.cjs are replaced with `execFile()` equivalents — verified by grep finding zero `execSync` calls with git arguments
  2. `/gbsd:map-codebase` is removed or emits a deprecation notice redirecting to Pathfinder; existing projects that call it get a clear migration message
  3. `worktreeManager.spawnExecutor()` abstraction exists as a stub/interface before any worktree implementation — one-line change to adapt when GitHub issue #33045 is closed
  4. ADOPT-01 tracking mechanism is in place: a documented "upgrade checklist" and Claude Code version pin in prerequisites that must be verified before any worktree-related update
  5. Upstream GSD sync mechanism is documented: cherry-pick process defined, which file categories produce acceptable rebrand conflicts vs. which must not conflict
**Plans:** TBD

---

### Phase 2: Worktree Foundation
**Goal:** Reliable worktree creation and teardown — the isolation primitive everything else depends on
**Depends on:** Phase 1 (security fix must be in place before any shell invocations in worktree code)
**Requirements:** PARA-01, PARA-04, PARA-07
**Dependency reasoning:**
- PARA-01 (worktree isolation per executor) and PARA-04 (worktree manager utility) are the same deliverable — the manager is how isolation is implemented. They cannot be split.
- PARA-07 (resource pre-flight checks) belongs here because the pre-flight check is the first thing the Worktree Manager runs. Building the manager without its entry guard is a partial implementation.
- State Reconciler (Phase 3) needs to know the worktree branch naming convention — that interface is defined here.
- Key Pool Manager (Phase 3) is meaningless without executors to assign keys to — the executor lifecycle is defined here.
**Success Criteria** (what must be TRUE):
  1. `git worktree list` shows exactly N entries during a wave of N parallel plans, and 0 entries after successful wave completion — worktrees are created and destroyed cleanly
  2. Orchestrator startup scans for orphaned `gsd/exec/*` branches and worktrees not in the active wave manifest — stale worktrees are surfaced for resume/retry/skip, not silently inherited
  3. Pre-flight check before spawning N worktrees validates available disk space equals `max_concurrent_agents × 700 MB` — wave fails early with a clear message rather than mid-wave at agent 4/5
  4. try/finally cleanup runs on every worktree creation code path — a failure during executor spawn does not leave an orphaned worktree or branch consuming disk
  5. Worktrees are created outside the project directory (`~/.claude/worktrees/` or sibling path) — executor agents cannot accidentally traverse into sibling worktrees via filesystem exploration
**Plans:** TBD

---

### Phase 3: Key Pool and State Reconciliation
**Goal:** Rate limit distribution and correct STATE.md merge — prerequisites for safe parallel wave execution
**Depends on:** Phase 2 (worktree branch naming convention must be defined before reconciler can reference branches)
**Requirements:** API-01, API-02, API-03, PARA-05, PARA-06, PARA-08
**Dependency reasoning:**
- API-03 (circuit breaker) must be built before API-02 (round-robin): round-robin across a pool with dead keys causes starvation; the circuit breaker is the safety valve that makes round-robin usable.
- API-01 (multi-org key pool) is only meaningful at the org level — multi-key within one org provides zero independent rate limit benefit. This architectural constraint must be baked into keypool.cjs from the start.
- PARA-05 (state reconciliation) is a prerequisite for Phase 5 execute-phase integration — without it, parallel waves silently discard decisions from non-winning executors. Must be fully tested before Phase 5.
- PARA-06 (auto-resolve merge conflicts for lock files, barrel files) belongs here because it is part of the reconciliation layer — resolving file-level git conflicts is the filesystem analog of STATE.md reconciliation.
- PARA-08 (per-agent execution trace logs) belongs here as a testability prerequisite — the reconciler needs verifiable input; per-agent logs make the divergence visible and the tests trustworthy.
**Success Criteria** (what must be TRUE):
  1. Three parallel executors each hitting different Anthropic orgs produce no 429 errors during a test wave — verified by checking `anthropic-organization-id` response headers show three distinct values
  2. When one org returns a 429, the circuit breaker blacklists it for the retry-after interval and routes remaining executors to the other orgs — the wave continues without stalling
  3. After a three-executor wave completes, `STATE.md` Decisions section contains entries from all three executors — no decisions silently discarded by last-write-wins
  4. `agent-history.json` after reconciliation contains unique entries with globally unique `agent_id` values (worktree name + timestamp) — no duplicate executor histories
  5. Lock file and barrel file conflicts from parallel executors auto-resolve at 80%+ rate — a three-executor wave touching `package.json` and `src/index.ts` produces zero manual conflict resolution steps
  6. Each executor writes a trace log at `.planning/phases/{phase}/exec-{plan-id}.log` — interruption diagnosis does not require re-running the executor
**Plans:** TBD

---

### Phase 4: File Ownership and Decision Frontloading
**Goal:** Correctness guarantees — prevent merge conflict cascade and mid-wave human interruptions before parallel execution ships
**Depends on:** Phase 3 (planner must know what the execution layer needs: worktree branch conventions, reconciler-friendly frontmatter)
**Requirements:** PLAN-01, PLAN-02, PLAN-03, PLAN-04, PLAN-05, AUTON-01, AUTON-02, AUTON-03, AUTON-04
**Dependency reasoning:**
- This phase MUST complete before Phase 5 (execute-phase integration) ships. This is a hard ordering constraint from the research: "File ownership enforcement in the planner must exist before parallel waves run. Shipping execution without conflict prevention produces a broken system that appears to work on simple projects and fails on any project with shared infrastructure files."
- PLAN-01/02 (file ownership, hard vs soft dependencies) are the conflict prevention mechanism. Without them, any two plans that touch `package.json` or barrel files produce an unresolvable conflict.
- AUTON-01/02 (frontloaded decisions, checkpoint classification) are the autonomy mechanism. Without them, `skip_checkpoints: true` causes plan failures rather than autonomous completion — parallel agents cannot pause mid-wave for human input.
- AUTON-04 (decision-surface sizing) must be defined before AUTON-03 (remove 200k-era token cap) — the old cap was a proxy for "manageable work unit." Removing it without a replacement produces unbounded plans. Decision count replaces token count as the sizing metric.
- PLAN-03 (auto-derive dependencies via Pathfinder) is listed here as a Pathfinder-optional enhancement. The planner implements manual file-ownership-first decomposition (no Pathfinder required); Pathfinder automates the analysis when available.
- PLAN-04/05 (parallelizability score, granular plans) are outputs of the redesigned planner — they emerge naturally from file-ownership-first decomposition and decision-surface sizing.
**Success Criteria** (what must be TRUE):
  1. Plans produced by the updated planner include `files_owned`, `files_created`, and `files_read` frontmatter fields — the planner's decomposition algorithm identifies file sets before naming plans
  2. The plan-checker rejects any wave assignment where two plans in the same wave both declare ownership of the same file — the hard constraint is enforced before execution, not discovered during merge
  3. The discuss-phase captures all ambiguous decisions into a structured decision log before plan creation begins — execution runs with `skip_checkpoints: true` on a project where every decision was previously made interactively, and completes without pausing
  4. Plans are sized by decision-point count (3-5 decision points per plan, not 3-5 tasks) — a plan with 10 deterministic tasks and 1 human-verify point is valid; a plan with 4 tasks all requiring human judgment is flagged
  5. Wave 1 contains 70%+ of plans from a typical phase (baseline: 40-60%) when Pathfinder index is available; 50%+ without Pathfinder — verified by examining wave assignments in a test plan
**Plans:** TBD

---

### Phase 5: Execute-Phase Integration
**Goal:** End-to-end parallel wave execution — wire all components into a working system
**Depends on:** Phase 2 (worktree.cjs), Phase 3 (keypool.cjs, reconcile.cjs, PARA-06), Phase 4 (PLAN-01, AUTON-01/02 — the correctness guarantees must exist before waves run)
**Requirements:** PARA-02, PARA-03, TEAM-04, ARCH-01, ARCH-02, ARCH-03, SCALE-01, SCALE-02
**Dependency reasoning:**
- PARA-02 (wave-level parallel execution with merge-back) is the central deliverable — this is the phase where execute-phase.md gains wave grouping, worktree spawning, key assignment, merge-back, and State Reconciler invocation.
- TEAM-04 (executor checkpoint/resume) must ship in this phase — parallel execution increases failure surface. A system without resume is not production-ready.
- PARA-03 (phase-level parallelism, opt-in) ships here as a gated feature. It requires worktree and reconciler to be proven stable (from Phase 2/3). The `--parallel-phases` flag enables it; the default remains wave-level only.
- ARCH-01/02/03 (maximize parallelism, multi-agent at every stage, dynamic agent count) are architecture principles that are realized here — the execute-phase.md update is the primary realization point.
- SCALE-01/02 (100k+ LOC efficient operation, <5% exploration overhead) are consequences of correct Pathfinder integration (Phase 6) combined with the parallel execution implemented here. They are tracked in this phase because the execution layer is what drives scale behavior.
**Success Criteria** (what must be TRUE):
  1. A phase with N independent plans executes in parallel across N worktrees — wall-clock time scales as (slowest_plan + overhead), not sum(all_plans); verified by timing a 3-plan wave before and after
  2. An interrupted wave (executor killed mid-task) produces a resume prompt on next `/gbsd:execute-phase` invocation — the executor resumes from the correct task boundary without duplicating committed work
  3. Three concurrent executors each use a key from a different Anthropic org — verified by executor trace logs showing distinct org IDs; no executor inherits the lead's API key
  4. Post-wave: `git worktree list` shows 0 entries, all executor branches are deleted, STATE.md decisions section is reconciled from all executors — the system is clean for the next wave
  5. A phase with independently-implementable sub-phases can be annotated for `--parallel-phases` execution — the feature is available behind the opt-in flag and documented
  6. Exploration overhead in the execute-phase context budget is under 5% when a Pathfinder index is available — agents read only their `files_owned` set, not the broader codebase
**Plans:** TBD

---

### Phase 6: Pathfinder Integration
**Goal:** Code-intelligence-driven plan quality — automatic file ownership, dependency detection, and wave assignment without manual specification
**Depends on:** Phase 4 (planner's file-ownership-first decomposition is the interface Pathfinder plugs into; graceful degradation path defined before Pathfinder-dependent code is written)
**Requirements:** PATH-01, PATH-02, PATH-03, PATH-04, DEPR-01
**Dependency reasoning:**
- Pathfinder is an independent parallel track that can begin development during Phase 5. The hard dependency is on Phase 4: the planner's decomposition algorithm and frontmatter schema must be stable before the Pathfinder adapter writes into them.
- PATH-04 (index freshness) must implement content-based staleness detection (`git diff --name-only HEAD~1..HEAD` vs indexed files) rather than time-based. Time-based fails in both directions.
- Graceful degradation (`pathfinder.cjs.isAvailable()` returning false) must be implemented and tested before any Pathfinder-dependent planner code runs — the "no index" path must be verified first.
- DEPR-01 (/gbsd:map-codebase deprecation) is listed in Phase 1 for the removal/deprecation notice, but the full functional replacement by Pathfinder is complete here when the adapter ships.
**Success Criteria** (what must be TRUE):
  1. Running `/gbsd:plan-phase` with `.code-intel/` absent or deleted produces a valid plan — planning completes with manual file-ownership declarations, no errors about missing index
  2. Running `/gbsd:plan-phase` after a file rename produces plans referencing the new file paths — content-based staleness detection forces re-index when any indexed file was moved, renamed, or deleted since last indexing
  3. A plan produced with Pathfinder available has `depends_on` fields auto-populated from the import graph — planner does not require manual dependency specification for files already in the index
  4. Wave 1 assignment achieves 70%+ parallelizability on a real phase — measured by counting plans assigned to Wave 1 vs. total plans, with Pathfinder providing the dependency data
**Plans:** TBD

---

### Phase 7: Agent Teams, Model Routing, and Adoption
**Goal:** Operational excellence — hub-and-spoke team coordination, tiered model assignment, and upstream compatibility infrastructure
**Depends on:** Phase 5 (parallel execution must be proven working before building coordination layers on top; model routing costs only become visible after execution runs)
**Requirements:** TEAM-01, TEAM-02, TEAM-03, TEAM-05, TEAM-06, TEAM-07, API-04, API-05, MODEL-01, MODEL-02, MODEL-03, MODEL-04
**Dependency reasoning:**
- TEAM-01/02/03/05/06/07 (hub-and-spoke coordination, checkpoint protocol, wave synchronization, team-based research/plan/verify) all build on the execution infrastructure from Phase 5. Hub-and-spoke with lean lead requires Pathfinder (available from Phase 6) for blast-radius analysis that keeps lead context small.
- TEAM-02 (file-based checkpoint protocol) is closely related to PARA-08 (trace logs, Phase 3) — the checkpoint protocol is the structured version of what trace logs capture informally.
- API-04 (multi-provider model routing) must be defined by routing consequence (research and verification stay on Opus; execution on Sonnet; high-throughput tasks on Haiku) not by task name. This routing policy only makes sense after real cost data from Phase 5 execution.
- API-05 (cost tracking via LiteLLM or equivalent) is optional infrastructure; it is listed here and not deferred because the project requires all requirements in v1.
- MODEL-01/02/03/04 (per-agent-type model config, per-workflow overrides, CLI override, capability validation) are the mechanism that API-04's routing policy operates through. They require the same foundation of proven parallel execution.
**Success Criteria** (what must be TRUE):
  1. A research phase with 3 sub-topics spawns 3 parallel researcher agents coordinated by a lead — the lead aggregates findings iteratively rather than waiting for all three to finish before starting synthesis
  2. The planner-checker revision loop runs as a message-based team iteration — planner and checker exchange at most 3 rounds before the plan is committed, with structured feedback in each round
  3. The per-agent model config is operative: planner uses Opus 4.6, executors use Sonnet 4.6, and at least one agent type is configurable to Haiku 4.5 without modifying agent definitions
  4. A model capability validation error surfaces when an agent type is assigned a model below its minimum reasoning tier — the config is rejected before the phase runs, not discovered mid-execution
  5. Cost tracking records token usage and estimated cost per agent invocation in a structured log — post-phase cost report shows per-agent-type breakdown
**Plans:** TBD

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Security Foundation and Deprecation | 0/? | Not started | - |
| 2. Worktree Foundation | 0/? | Not started | - |
| 3. Key Pool and State Reconciliation | 0/? | Not started | - |
| 4. File Ownership and Decision Frontloading | 0/? | Not started | - |
| 5. Execute-Phase Integration | 0/? | Not started | - |
| 6. Pathfinder Integration | 0/? | Not started | - |
| 7. Agent Teams, Model Routing, and Adoption | 0/? | Not started | - |

---

## Coverage Map

All 46 requirements mapped. No orphans.

| Requirement | Phase | Dependency Rationale |
|-------------|-------|----------------------|
| DEPR-01 | Phase 1 | Deprecation notice before Pathfinder ships; full replacement complete in Phase 6 |
| ADOPT-01 | Phase 1 | Abstraction layer must exist before any worktree code is written |
| ADOPT-02 | Phase 1 | New Claude Code features must have an adoption path before worktrees depend on current behavior |
| ADOPT-03 | Phase 1 | Upstream sync mechanism documented before structural work begins |
| PARA-01 | Phase 2 | Isolation primitive — nothing else can start without this |
| PARA-04 | Phase 2 | Worktree manager is how PARA-01 is implemented; inseparable |
| PARA-07 | Phase 2 | Pre-flight resource check is the entry guard for the worktree manager |
| API-01 | Phase 3 | Multi-org pool design; prerequisite for independent rate limits |
| API-02 | Phase 3 | Round-robin requires circuit-breaker (API-03) to be safe; both ship together |
| API-03 | Phase 3 | Circuit breaker before round-robin; must be built first |
| PARA-05 | Phase 3 | State reconciler; must be tested before Phase 5 execute-phase integration |
| PARA-06 | Phase 3 | Auto-resolve lock/barrel conflicts; part of the reconciliation layer |
| PARA-08 | Phase 3 | Per-agent trace logs; testability prerequisite for reconciler |
| PLAN-01 | Phase 4 | File ownership first-class signal; must exist before any parallel wave ships |
| PLAN-02 | Phase 4 | Hard vs soft dependency classification; part of file ownership implementation |
| PLAN-03 | Phase 4 | Auto-derive dependencies via Pathfinder; Pathfinder-optional enhancement to PLAN-01 |
| PLAN-04 | Phase 4 | Parallelizability score; output of file-ownership-first decomposition |
| PLAN-05 | Phase 4 | Granular plans (5-10 × 1-2 tasks); output of decision-surface sizing |
| AUTON-01 | Phase 4 | Frontloaded decision capture; must exist before any parallel wave ships |
| AUTON-02 | Phase 4 | Checkpoint classification; paired with AUTON-01 (same correctness guarantee) |
| AUTON-03 | Phase 4 | Remove 200k-era token cap; requires AUTON-04 (replacement metric) to be defined first |
| AUTON-04 | Phase 4 | Decision-surface sizing metric; defines the replacement for AUTON-03 |
| PARA-02 | Phase 5 | Wave-level execution; requires all correctness guarantees from Phases 2-4 |
| PARA-03 | Phase 5 | Phase-level parallelism (opt-in); requires worktree + reconciler proven stable |
| TEAM-04 | Phase 5 | Executor checkpoint/resume; must ship with parallel execution (parallel increases failure surface) |
| ARCH-01 | Phase 5 | Maximize parallelism; realized in execute-phase.md update |
| ARCH-02 | Phase 5 | Multi-agent at every stage; realized in execute-phase.md update |
| ARCH-03 | Phase 5 | Dynamic agent count; realized in execute-phase.md config |
| SCALE-01 | Phase 5 | 100k+ LOC efficient operation; consequence of Pathfinder + parallel execution |
| SCALE-02 | Phase 5 | <5% exploration overhead; consequence of file ownership scoping in executors |
| PATH-01 | Phase 6 | Blast-radius impact analysis; Pathfinder adapter prerequisite |
| PATH-02 | Phase 6 | Dependency-graph task decomposition; Pathfinder adapter function |
| PATH-03 | Phase 6 | Module coupling analysis; Pathfinder adapter function |
| PATH-04 | Phase 6 | Index freshness management; must use content-based not time-based detection |
| TEAM-01 | Phase 7 | Hub-and-spoke teams; requires Pathfinder (blast-radius for lean lead) |
| TEAM-02 | Phase 7 | File-based checkpoint protocol; structured version of PARA-08 trace logs |
| TEAM-03 | Phase 7 | Task dependency chains (BlockedBy); wave synchronization enhancement |
| TEAM-05 | Phase 7 | Agent teams for research phase; requires working agent teams infrastructure |
| TEAM-06 | Phase 7 | Agent teams for plan phase; planner-checker revision loop |
| TEAM-07 | Phase 7 | Agent teams for verification phase; parallel verifier coordination |
| API-04 | Phase 7 | Multi-provider model routing; routing policy defined by cost data from Phase 5 |
| API-05 | Phase 7 | Cost tracking via LiteLLM; operational infrastructure for multi-provider routing |
| MODEL-01 | Phase 7 | Per-agent-type model config; mechanism API-04 routing operates through |
| MODEL-02 | Phase 7 | Per-workflow-stage model overrides; mechanism API-04 routing operates through |
| MODEL-03 | Phase 7 | Runtime model selection via CLI flag; mechanism API-04 routing operates through |
| MODEL-04 | Phase 7 | Model capability validation; safety check for API-04 routing assignments |

**Coverage check:**
- Total v1 requirements: 46
- Mapped: 46
- Unmapped: 0

---

## Research Flags

Phases likely needing `/gsd:research-phase` during planning:
- **Phase 4 (File Ownership / Decision Frontloading):** Novel algorithms with limited documented prior art; planner redesign has high blast radius on downstream agents; decision-surface sizing metric needs definition
- **Phase 6 (Pathfinder Integration):** Pathfinder index format and query semantics need verification; content-based staleness detection algorithm; blast-radius query design

Phases with standard patterns (skip research-phase):
- **Phase 1 (Security + Deprecation):** Shell injection fix is well-documented (`execSync` → `execFile`); deprecation pattern is straightforward
- **Phase 2 (Worktree Manager):** git worktree API is stable and well-documented; lifecycle management patterns are well-established
- **Phase 3 (Key Pool / Reconciler):** Circuit-breaker patterns are well-documented in LLM gateway literature; STATE.md merge strategy is defined in architecture docs
- **Phase 5 (Execute-Phase Integration):** All components are defined and tested in Phases 2-4; wiring is standard orchestration work
- **Phase 7 (Agent Teams / Model Routing):** Routing policy is defined; LiteLLM docs are comprehensive; Agent Teams patterns from Phase 5

---

*Roadmap perspective: Dependency Ordering*
*Created: 2026-03-15*
*Counterpart roadmaps: roadmap-features.md, roadmap-autonomy.md (synthesized into ROADMAP.md)*
