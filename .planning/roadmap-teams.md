# GBSD Roadmap: Agent Teams Architecture
**Specialist:** Agent Teams and Parallelism Rollout Across All Workflow Stages
**Granularity:** Standard (5-8 phases)
**Coverage:** 46/46 requirements mapped

---

## Governing Principle

ARCH-01/02/03 is not a phase — it is the lens through which every phase is evaluated.
Every idle API rate slot is wasted wall-clock time. Each phase below introduces
parallelism at the next workflow stage. The sequence is not arbitrary: you cannot
parallelize a stage until its isolation primitive exists and its coordination
protocol is defined.

---

## Phases

- [ ] **Phase 1: Teams Infrastructure** - Hub-and-spoke foundation, checkpoint protocol, task dependency chains, worktree isolation
- [ ] **Phase 2: API Bandwidth and Model Routing** - Key pool, circuit breaker, per-agent model selection — the rate bandwidth teams need to run at scale
- [ ] **Phase 3: Research Teams** - Parallel researchers with synthesizer, TEAM-05 end-to-end, plus Pathfinder deprecation cleanup
- [ ] **Phase 4: Planning Correctness** - File ownership, decision frontloading, autonomy guarantees — required before parallel execution is safe
- [ ] **Phase 5: Execute Teams (Wave Execution)** - Parallel wave execution wired to teams infrastructure, worktree isolation, state reconciliation, merge-back
- [ ] **Phase 6: Verification Teams and Pathfinder** - Parallel verifiers, plan-phase checker team, Pathfinder intelligence integration
- [ ] **Phase 7: Scale and Adoption** - Phase-level parallelism, dynamic agent scaling, Claude Code adoption posture, upstream sync

---

## Phase Details

### Phase 1: Teams Infrastructure

**Goal:** The hub-and-spoke coordination layer exists and is reliable. The lead can
delegate to teammates, persist state across agent boundaries, synchronize waves via
task dependency chains, and resume interrupted executors. This is the primitive every
later phase requires.

**Depends on:** Nothing — this is the foundation.

**Requirements:** TEAM-01, TEAM-02, TEAM-03, TEAM-04, PARA-01, PARA-04, PARA-07, PARA-08

**Success Criteria** (what must be TRUE when this phase completes):
  1. A team lead can spawn a teammate, pass it a worktree path and plan reference,
     and the teammate executes in that isolated worktree without touching the main
     working directory.
  2. An executor that is interrupted mid-plan writes a checkpoint file; on restart,
     the lead detects the checkpoint and can resume or retry the executor from the
     last safe boundary.
  3. Wave synchronization works: tasks in Wave 2 remain blocked until all Wave 1
     TaskUpdate(status="completed") calls have been processed, and the lead never
     spawns Wave 2 executors early.
  4. Stale worktrees from a previous interrupted run are detected on orchestrator
     startup and cleaned up (with disk space reclaimed) before new worktrees are
     created.
  5. Per-executor trace logs exist — one file per executor, written during execution,
     readable after the fact for debugging parallel runs.

**Plans:** TBD

**Teams architecture note:**
TEAM-01 establishes the lead-lean constraint: lead stays ~15% context, delegates
execution to teammates. TEAM-02 defines the file-based checkpoint protocol that
state persists across agent process boundaries. TEAM-03 establishes BlockedBy
dependency chains as the wave synchronization mechanism. TEAM-04 closes the
interruption gap — parallel execution multiplies failure surface, resume is not
optional.

PARA-01 (worktree isolation per executor) and PARA-04 (worktree manager utility)
are grouped here because teammates cannot use `isolation: "worktree"` (issue #33045
is still open) — the lead must manage worktrees manually. The worktree manager is
the workaround implementation.

PARA-07 (resource pre-flight) prevents cascading failure when N worktrees would
exceed disk or memory. PARA-08 (trace logs) is the debuggability mechanism needed
before parallel execution ships.

---

### Phase 2: API Bandwidth and Model Routing

**Goal:** The system can saturate available API rate limits across multiple orgs and
assign the right model tier to each agent role. Every executor gets a key from a
different org. Every planner uses Opus. Every researcher uses Haiku. The circuit
breaker prevents a 429 from one key from stalling a wave.

**Depends on:** Phase 1 (teams infrastructure defines the agent roles that need keys
and models assigned)

**Requirements:** API-01, API-02, API-03, API-04, API-05, MODEL-01, MODEL-02,
MODEL-03, MODEL-04

**Success Criteria** (what must be TRUE when this phase completes):
  1. When three executors are spawned simultaneously, each receives an API key from
     a different Anthropic org; no two parallel executors share an org's rate limit
     pool.
  2. When a 429 response is received on one key, that key is blacklisted in memory
     with exponential backoff, the next key in the pool is used, and parallel
     execution continues without stalling.
  3. The planner agent runs Opus 4.6, executor agents run Sonnet 4.6, and researcher
     agents run Haiku 4.5 — verified by inspecting the model field in each agent's
     CLAUDE.md or session config.
  4. A user can override the model for any agent type via a CLI flag at invocation
     time without editing config files.
  5. The system validates that a selected model meets minimum capability requirements
     before assigning it (e.g., rejects Haiku for the planner role if capability
     validation is enabled).

**Plans:** TBD

**Teams architecture note:**
MODEL-01/02/03/04 are grouped with API because per-agent model selection and
per-agent key assignment are two sides of the same "agent identity" configuration.
A research team of 4 Haiku agents each on a different org key achieves 4× the ITPM
headroom of a single Opus session. Model routing and rate routing must be defined
together or the capacity math is wrong.

API-05 (cost tracking via LiteLLM) ships here because cost visibility is required
to validate that the Haiku-for-research / Sonnet-for-execution / Opus-for-planning
tier strategy actually delivers the projected 40-60% cost reduction.

ARCH-03 (dynamic agent count scaled to available bandwidth) depends on knowing the
available bandwidth — which requires the key pool to be in place first.

---

### Phase 3: Research Teams

**Goal:** The research phase runs as a parallel team. Four researcher agents write
to independent files simultaneously; a synthesizer agent is unblocked once all four
complete. Research wall-clock time drops from sequential to the time of the slowest
researcher. Pathfinder replaces the old map-codebase workflow.

**Depends on:** Phase 1 (task dependency chains, teammate spawn), Phase 2
(model routing so researchers use Haiku, synthesizer uses Sonnet)

**Requirements:** TEAM-05, DEPR-01, SCALE-01, SCALE-02, PATH-04

**Success Criteria** (what must be TRUE when this phase completes):
  1. Running the research phase spawns four parallel researcher teammates writing to
     STACK.md, FEATURES.md, ARCHITECTURE.md, and PITFALLS.md simultaneously; the
     synthesizer does not start until all four have signaled completion.
  2. The research phase wall-clock time is bounded by the slowest single researcher,
     not by the sum of all four — verified by comparing timestamps across research
     output files.
  3. The /gbsd:map-codebase command is deprecated and replaced by a thin wrapper or
     removed; users who invoke it receive a migration notice pointing to Pathfinder
     index generation.
  4. When a Pathfinder index is present, research agents use it for codebase queries
     instead of exploration; context budget for codebase investigation is under 5%.
  5. Pathfinder index freshness is checked before any research-phase invocation;
     stale indexes trigger a refresh prompt rather than silently producing wrong
     dependency data.

**Plans:** TBD

**Teams architecture note:**
TEAM-05 is the first full team rollout across a workflow stage. Research is the
ideal first target because it has no merge conflicts (each researcher writes to a
unique file), no worktrees needed, and a clean fan-out/fan-in shape. This validates
the hub-and-spoke coordination protocol under real conditions before it handles the
harder execution-phase merge complexity.

DEPR-01 lands here because Pathfinder replaces map-codebase, and research is the
first stage to use Pathfinder queries. PATH-04 (index freshness management) must
ship before research teams consume the index — a stale index in a parallel research
run produces four researchers with wrong assumptions.

SCALE-01/02 are measurable at research phase first: the <5% exploration overhead
target can be verified by checking how much of a research session is spent on
directory listing vs actual research.

---

### Phase 4: Planning Correctness

**Goal:** Plans carry file ownership metadata. All human decisions are captured
before execution starts. Plans are sized by decision surface, not token budget.
These are the correctness guarantees — without them, parallel execution produces
merge conflicts and mid-wave interruptions.

**Depends on:** Phase 1 (worktree infrastructure defines what file ownership needs
to prevent), Phase 3 (research teams validate context; planner reads synthesis)

**Requirements:** PLAN-01, PLAN-02, PLAN-03, PLAN-04, PLAN-05, AUTON-01, AUTON-02,
AUTON-03, AUTON-04, PATH-01, PATH-02, PATH-03

**Success Criteria** (what must be TRUE when this phase completes):
  1. Every plan file contains `files_owned`, `files_created`, and `files_read`
     frontmatter fields; the planner refuses to emit a plan without them.
  2. The discuss phase gathers all ambiguous decisions before plan creation;
     execution runs to completion without pausing to ask the user a question that
     could have been answered during planning.
  3. Plans target 70-85% of tasks in Wave 1 (verified via the parallelizability
     score in plan metadata); the planner flags plans below 60% for review.
  4. When a Pathfinder index is present, task dependencies are derived from import
     graphs rather than manually specified; the planner outputs a dependency rationale
     showing which imports drove each wave boundary.
  5. Plans are 5-10 plans × 1-2 tasks each; the planner checker rejects plans with
     more than 3 tasks per plan as over-bundled.

**Plans:** TBD

**Teams architecture note:**
TEAM-06 (plan-phase as a team) is the implementation vehicle here: planner and
checker run as a team in a revision loop (researcher → planner → checker, max 3
iterations). The plan-checker validates file ownership completeness, wave
consistency, and decision surface size. AUTON-04 (decision-surface sizing) replaces
the old token-budget cap — this is only safe to remove after frontloaded decision
capture (AUTON-01/02) guarantees that plan execution will not stall for human input.

PATH-01/02/03 (blast-radius analysis, dependency-graph decomposition, coupling
analysis) are the intelligence that makes 70-85% wave-1 parallelizability achievable.
Without Pathfinder, planners achieve 40-60% through manual dependency estimation.

AUTON-03 (extended autonomous sessions via 1M context) is unblocked here because
the 1M context window GA (March 13, 2026) removes pricing friction, and
decision-surface sizing (AUTON-04) replaces the old context-budget cap.

---

### Phase 5: Execute Teams (Wave Execution)

**Goal:** Parallel wave execution runs end-to-end. The execute phase is a team:
lead manages worktrees, spawns executor teammates, waits for wave completion,
merges branches, reconciles state, and advances to the next wave. The system runs
autonomously from plan to completion without human intervention.

**Depends on:** Phase 1 (worktree manager, checkpoint protocol, task chains),
Phase 2 (key pool so each executor gets its own org key), Phase 4 (file ownership
prevents merge conflicts in parallel waves)

**Requirements:** PARA-02, PARA-03, PARA-05, PARA-06, TEAM-04

**Success Criteria** (what must be TRUE when this phase completes):
  1. A five-plan phase executes as two parallel waves: Wave 1 spawns all independent
     executor teammates simultaneously; Wave 2 does not start until all Wave 1
     TaskUpdate(status="completed") calls are received.
  2. After wave completion, branch merges succeed without manual intervention for
     lock files, config files, and barrel files (80%+ of conflict types
     auto-resolved).
  3. STATE.md and agent-history.json reflect the correct merged state after parallel
     waves — no executor's decisions are silently dropped by last-write-wins
     overwrite.
  4. An interrupted executor (process killed mid-task) is detected on the next
     orchestrator invocation; the lead can resume from the last checkpoint boundary
     without re-executing completed tasks.
  5. With --parallel-phases flag set, independent phases (verified via ROADMAP.md
     dependency declarations) execute simultaneously with separate team leads and
     branch merges.

**Plans:** TBD

**Teams architecture note:**
This is where ARCH-01/02/03 becomes fully operational: the execute phase runs
N executor teammates where N = available rate bandwidth / per-executor token rate.
PARA-02 (wave execution with merge-back) is the core deliverable. PARA-03
(phase-level parallelism) is opt-in via --parallel-phases and is gated behind
proven wave-level execution.

PARA-05 (state reconciliation) and PARA-06 (merge conflict auto-resolution) are
execute-phase requirements — they only matter when executors are actually running
in parallel worktrees. Both land here rather than Phase 1 because Phase 1 defines
the interfaces and Phase 5 implements the full merge pipeline.

TEAM-04 (executor checkpoint/resume) is included again here as the execute-phase
integration of the checkpoint protocol defined in Phase 1 — Phase 1 defines the
file format, Phase 5 wires it into the execute-phase workflow.

---

### Phase 6: Verification Teams and Plan-Phase Team

**Goal:** Verification runs as a parallel team. The verify phase spawns multiple
verifier agents simultaneously; a coordinator aggregates results and triggers
follow-up verification on failures. The plan-phase team (TEAM-06) is fully
operational with the message-based revision loop.

**Depends on:** Phase 1 (task chains, teammate spawn), Phase 2 (model routing for
verifiers), Phase 5 (verification has real parallel output to inspect)

**Requirements:** TEAM-06, TEAM-07, ARCH-01, ARCH-02

**Success Criteria** (what must be TRUE when this phase completes):
  1. The verify phase spawns parallel verifier agents — one per plan completed in
     the wave — and aggregates their findings before presenting a consolidated
     verification report to the user.
  2. When a verifier finds failures, the coordinator triggers targeted follow-up
     verification on only the affected plans rather than re-running full verification
     across all plans.
  3. The plan-checker runs as a teammate in a message-based revision loop (not a
     monolithic planner session); issues from the checker are communicated via
     SendMessage and the planner receives a structured revision request.
  4. Every workflow stage (roadmap, plan, research, execute, verify, debug, audit,
     integration-check) has a documented team structure showing which roles run in
     parallel, which are sequential, and what the coordination mechanism is.

**Plans:** TBD

**Teams architecture note:**
TEAM-07 (verification teams) and TEAM-06 (plan-phase team) land here rather than
earlier because they require the execution output to verify. Verification teams are
the last major stage to receive the parallel treatment. ARCH-01/02 are explicitly
assigned here as the completion gate: when every workflow stage (including verify
and plan-check) runs via coordinated agent teams, ARCH-02's requirement that "every
workflow stage must support multi-agent parallel execution, coordinated by a team
lead" is satisfied.

ARCH-01 (maximize parallelism at every stage) is also assigned here — it is
verifiable only once all stages have team structures. The success criterion for
ARCH-01 is the Phase 6 criterion 4: every stage has a documented parallel team
structure.

---

### Phase 7: Scale and Adoption

**Goal:** GBSD operates reliably on 100k+ LOC codebases. Dynamic agent scaling
adjusts concurrent agents based on measured API bandwidth. The adoption posture
for Claude Code API changes is systematized, and upstream GSD sync is available.

**Depends on:** Phases 1-6 (full system validated at scale)

**Requirements:** ARCH-03, ADOPT-01, ADOPT-02, ADOPT-03

**Success Criteria** (what must be TRUE when this phase completes):
  1. Running GBSD against a 100k+ LOC codebase completes planning and execution
     without exploration-based context exhaustion; Pathfinder index provides all
     file ownership and dependency data.
  2. The concurrent agent count scales dynamically with measured API headroom: if
     available ITPM is 500K and each executor consumes 50K/min, the system spawns
     10 executors, not a hardcoded N.
  3. When Claude Code ships a fix for issue #33045 (teammate worktree isolation),
     GBSD can adopt declarative `isolation: "worktree"` by updating one abstraction
     layer (`worktreeManager.spawnExecutor()`), not by hunting worktree creation
     calls across the codebase.
  4. A user can cherry-pick upstream GSD improvements into GBSD via a documented
     sync process without losing GBSD-specific changes (rebranding and Pathfinder
     integration conflicts are acceptable; structural conflicts are not).

**Plans:** TBD

**Teams architecture note:**
ARCH-03 (dynamic agent count) is the final ARCH requirement to land. It requires
observed, measured bandwidth data from running phases 5-6 in parallel before
the scaling formula can be calibrated. Premature dynamic scaling (before wave
execution is proven) would produce instability rather than efficiency.

ADOPT-01/02/03 are grouped here as the "staying current" phase. ADOPT-01 is the
most critical: the abstraction layer in `worktreeManager.spawnExecutor()` must
exist before issue #33045 is fixed upstream, or GBSD will have conflicting
double-management of worktrees when the fix lands. This phase systematizes the
monitoring and adoption posture so Claude Code API evolution is absorbed as an
operational practice, not a crisis response.

---

## Requirement Coverage Map

### Full Traceability

| Requirement | Phase | Rationale |
|-------------|-------|-----------|
| ARCH-01 | Phase 6 | Verified when all workflow stages have team structures |
| ARCH-02 | Phase 6 | Every stage documented as multi-agent — completion gate |
| ARCH-03 | Phase 7 | Dynamic scaling requires measured bandwidth from running system |
| AUTON-01 | Phase 4 | Frontloaded decision capture is a planning correctness requirement |
| AUTON-02 | Phase 4 | Checkpoint classification is part of the planning protocol |
| AUTON-03 | Phase 4 | Extended sessions are unblocked by decision-surface sizing |
| AUTON-04 | Phase 4 | Decision-surface sizing replaces token-budget cap in planner |
| PATH-01 | Phase 4 | Blast-radius analysis drives wave-1 parallelizability in planner |
| PATH-02 | Phase 4 | Dependency-graph task decomposition is a planner capability |
| PATH-03 | Phase 4 | Module coupling analysis drives automatic task clustering in planner |
| PATH-04 | Phase 3 | Index freshness required before research teams consume the index |
| PLAN-01 | Phase 4 | File ownership frontmatter is a planning correctness requirement |
| PLAN-02 | Phase 4 | Hard vs soft dependency distinction is in planner decomposition |
| PLAN-03 | Phase 4 | Auto-derive dependencies from imports is a planner Pathfinder query |
| PLAN-04 | Phase 4 | Parallelizability score ships with the planner redesign |
| PLAN-05 | Phase 4 | Granular plan structure (5-10 plans × 1-2 tasks) is planner policy |
| PARA-01 | Phase 1 | Worktree isolation per executor is the foundational isolation primitive |
| PARA-02 | Phase 5 | Wave-level parallel execution is the core execute-phase deliverable |
| PARA-03 | Phase 5 | Phase-level parallelism is opt-in, gated behind proven wave execution |
| PARA-04 | Phase 1 | Worktree manager utility is the Phase 1 core component |
| PARA-05 | Phase 5 | State reconciliation is only needed when executors actually run parallel |
| PARA-06 | Phase 5 | Merge conflict auto-resolution is part of the merge-back pipeline |
| PARA-07 | Phase 1 | Resource pre-flight prevents cascading failure before first worktree |
| PARA-08 | Phase 1 | Per-executor trace logs are required for debugging from day one |
| TEAM-01 | Phase 1 | Hub-and-spoke foundation — the coordination architecture |
| TEAM-02 | Phase 1 | File-based checkpoint protocol — state persistence across boundaries |
| TEAM-03 | Phase 1 | BlockedBy dependency chains — wave synchronization mechanism |
| TEAM-04 | Phase 1 | Checkpoint/resume — interruption recovery at execute phase |
| TEAM-05 | Phase 3 | Research phase as parallel agent team — first stage rollout |
| TEAM-06 | Phase 6 | Plan-phase team (planner + checker in message-based revision loop) |
| TEAM-07 | Phase 6 | Verification phase as parallel agent team — final stage rollout |
| DEPR-01 | Phase 3 | map-codebase deprecated when Pathfinder replaces it in research |
| API-01 | Phase 2 | Multi-org key pool — the rate bandwidth teams need to scale |
| API-02 | Phase 2 | Round-robin distribution across parallel executor sessions |
| API-03 | Phase 2 | Circuit breaker — prevents 429 from stalling a wave |
| API-04 | Phase 2 | Multi-provider model routing — per-agent-type model assignment |
| API-05 | Phase 2 | Cost tracking validates the tiered model savings claim |
| MODEL-01 | Phase 2 | Per-agent-type model config — individual model per agent role |
| MODEL-02 | Phase 2 | Per-workflow-stage model overrides — research/plan/execute/verify |
| MODEL-03 | Phase 2 | Runtime model selection via CLI flag |
| MODEL-04 | Phase 2 | Model capability validation before assignment |
| SCALE-01 | Phase 3 | Efficient 100k+ LOC operation — first measured at research phase |
| SCALE-02 | Phase 3 | <5% exploration overhead — verifiable once research uses Pathfinder |
| ADOPT-01 | Phase 7 | Abstraction layer for API changes — adoption posture systematized |
| ADOPT-02 | Phase 7 | Leverage new Claude Code features as they land |
| ADOPT-03 | Phase 7 | Upstream GSD sync capability |

**Coverage:** 46/46 requirements mapped. No orphans.

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Teams Infrastructure | 0/? | Not started | - |
| 2. API Bandwidth and Model Routing | 0/? | Not started | - |
| 3. Research Teams | 0/? | Not started | - |
| 4. Planning Correctness | 0/? | Not started | - |
| 5. Execute Teams (Wave Execution) | 0/? | Not started | - |
| 6. Verification Teams and Plan-Phase Team | 0/? | Not started | - |
| 7. Scale and Adoption | 0/? | Not started | - |

---

## Phase Dependency Graph

```
Phase 1: Teams Infrastructure
  └─ Phase 2: API Bandwidth and Model Routing
       └─ Phase 3: Research Teams
            └─ Phase 4: Planning Correctness
                 └─ Phase 5: Execute Teams
                      └─ Phase 6: Verification Teams
                           └─ Phase 7: Scale and Adoption
```

All phases are on the critical path for teams rollout. Phase 3 can begin
development in parallel with Phase 2 (research team structure does not require
the key pool to be built, only defined), but research teams cannot run in
production without Phase 2 complete.

---

## Parallelism Rollout by Workflow Stage

| Workflow Stage | Teams Phase | Mechanism | Worktrees? |
|----------------|-------------|-----------|------------|
| Research | Phase 3 | 4 parallel researchers + synthesizer (TEAM-05) | No — unique output files |
| Plan/Check | Phase 6 | Planner + checker in message-based revision loop (TEAM-06) | No — sequential |
| Execute | Phase 5 | N executor teammates in parallel waves (PARA-02) | Yes — required |
| Verify | Phase 6 | Parallel verifiers per plan, coordinator aggregates (TEAM-07) | No — read-only |
| Roadmap | Phase 6 | Parallel roadmappers (per ARCH-02 coverage requirement) | No — unique output files |
| Debug | Phase 6 | Multiple debugger agents on independent issues | No — unique output files |
| Audit | Phase 6 | Parallel auditors per module | No — read-only |
| Integration-check | Phase 6 | Parallel integration checkers per integration point | No — read-only |

---

## Key Architecture Decisions

| Decision | Phase | Rationale |
|----------|-------|-----------|
| Teams infrastructure before first team rollout | Phase 1 before Phase 3 | Cannot roll out research teams without checkpoint protocol and task chains |
| Model routing before teams run at scale | Phase 2 before Phase 3 | Research team of 4 Haiku agents consumes 4x ITPM — needs multi-org key pool |
| Research teams before planning correctness | Phase 3 before Phase 4 | Planner reads research synthesis; research teams validate coordination before the higher-stakes planning stage |
| Planning correctness before execute teams | Phase 4 before Phase 5 | File ownership prevents merge conflicts; decision frontloading prevents mid-wave interruptions; shipping execution without these guarantees produces a system that fails on real projects |
| ARCH-01/02 assigned to Phase 6 | Phase 6 completion gate | "Every stage supports parallel multi-agent execution" is only verifiable when all stages have received teams treatment |
| ARCH-03 (dynamic scaling) assigned to Phase 7 | Phase 7 | Dynamic scaling requires measured bandwidth data from running parallel phases; premature implementation produces instability |
| DEPR-01 in Phase 3 | Phase 3 | Pathfinder replaces map-codebase; research is the first stage to use Pathfinder queries, making it the natural deprecation point |

---

*Roadmap specialist: Agent Teams Architecture*
*Written: 2026-03-15*
*All 46 v1 requirements mapped. No deferral.*
