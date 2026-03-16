# Roadmap: GBSD Supercharge

## Overview

GBSD evolves from a sequential agent orchestration framework into a parallel, knowledge-indexed, autonomy-first execution engine. The journey proceeds in dependency order: first the hub-and-spoke coordination substrate with worktree isolation (the primitive everything else rests on), then the API bandwidth and model routing layer that makes large-scale parallel execution sustainable, then research teams and Pathfinder as the code intelligence track, then planning correctness guarantees that prevent merge conflicts before parallel waves ever run, then end-to-end wave execution wiring all prior work together, then verification and plan-phase teams as the final workflow stages to receive parallel treatment, and finally the adoption and scaling infrastructure that keeps GBSD current as Claude Code evolves. The result: phases execute autonomously end-to-end at 5-8x faster wall-clock time with 40-60% cost reduction, capable of handling 100k+ LOC codebases.

## Architecture Principle

CRITICAL: Parallel execution uses INDEPENDENT CLAUDE CODE INSTANCES in separate git worktrees — not Agent Teams teammates with `isolation: "worktree"` (GitHub issue #33045 is open and that feature is silently broken). Each worktree gets its own `claude` CLI process with its own API key, and each instance runs its own internal team to work on its assigned plan's tasks. The orchestrator creates worktrees, spawns processes, waits for completion, and merges branches back. This architecture sidesteps issue #33045 entirely and must be preserved at the `worktreeManager.spawnExecutor()` abstraction boundary.

## Phases

- [ ] **Phase 1: Teams and Worktree Foundation** - Hub-and-spoke coordination substrate, worktree lifecycle management, wave synchronization, checkpoint/resume, and trace logging
- [ ] **Phase 2: API Bandwidth and Model Selection** - Multi-org key pool with circuit breaker, per-agent model config, cost tracking — the rate bandwidth required to run N independent claude instances in parallel
- [ ] **Phase 3: Research Teams and Pathfinder Foundation** - Research as parallel team, Pathfinder replaces map-codebase, index freshness management, scale guarantees at 100k+ LOC
- [ ] **Phase 4: Planning Correctness and Pathfinder Intelligence** - File ownership frontmatter, frontloaded decisions, blast-radius analysis, Pathfinder-driven dependency decomposition — correctness guarantees before execution
- [ ] **Phase 5: Execute Teams and Wave Execution** - End-to-end parallel wave execution: N independent claude instances in N worktrees, merge-back, state reconciliation, interrupt resume
- [ ] **Phase 6: Verification and Plan-Phase Teams** - Parallel verifiers per plan, planner-checker revision loop, ARCH completion gate confirming every workflow stage supports multi-agent parallel execution
- [ ] **Phase 7: Scale and Adoption** - Dynamic agent scaling to measured API headroom, worktreeManager abstraction finalized, upstream GSD sync, Claude Code feature adoption posture

## Phase Details

### Phase 1: Teams and Worktree Foundation
**Goal**: The hub-and-spoke coordination substrate exists and is reliable — the orchestrator can create N git worktrees, spawn N independent claude CLI processes (each with its own API key and internal team), wait for wave completion, and merge branches back. Interrupted executors write checkpoints; waves synchronize via dependency chains; trace logs exist for every executor.
**Depends on**: Nothing (first phase)
**Requirements**: TEAM-01, TEAM-02, TEAM-03, TEAM-04, PARA-01, PARA-04, PARA-07, PARA-08
**Success Criteria** (what must be TRUE):
  1. Orchestrator creates N worktrees, spawns N independent claude CLI processes each with its own API key and internal team, waits for completion, and merges branches back — the worktrees are not Agent Teams teammates using `isolation: "worktree"` but separate CLI processes managed by the orchestrator
  2. An interrupted executor writes a checkpoint file at a predictable path; on next orchestrator invocation the lead detects the checkpoint and offers resume or retry from the last safe task boundary without re-executing completed tasks
  3. Wave 2 remains blocked until all Wave 1 executors have confirmed completion — the orchestrator never spawns Wave 2 processes early
  4. Stale worktrees from a previous interrupted run are detected on orchestrator startup and surfaced for resume/retry/skip before new worktrees are created; disk space is reclaimed on cleanup
  5. Each executor writes a per-agent trace log to `.planning/phases/{phase}/exec-{plan-id}.log` — when a parallel agent fails, its complete execution trace is readable without re-running
**Plans**: TBD

### Phase 2: API Bandwidth and Model Selection
**Goal**: Each independent claude instance gets an API key from a different Anthropic org, eliminating shared rate limit contention. The circuit breaker prevents a 429 from one org from stalling a wave. Every agent type uses the model tier appropriate to its task — Opus for planning, Sonnet for execution, Haiku for research — producing the projected 40-60% cost reduction.
**Depends on**: Phase 1 (agent roles defined by teams infrastructure; executor count defines key pool sizing)
**Requirements**: API-01, API-02, API-03, API-04, API-05, MODEL-01, MODEL-02, MODEL-03, MODEL-04
**Success Criteria** (what must be TRUE):
  1. Three parallel independent claude instances each receive a key from a different Anthropic org — no two parallel executors share an org's rate limit pool; verified by inspecting anthropic-organization-id headers in trace logs
  2. When one org returns a 429, the circuit breaker blacklists it for the retry-after interval, routes remaining executors to other orgs automatically, and execution continues without stalling or requiring user intervention
  3. The planner agent uses Opus 4.6 by default, executor agents use Sonnet 4.6, and researcher agents use Haiku 4.5 — this default configuration requires no user setup and produces 40-60% cost reduction versus an all-Opus baseline
  4. Any agent's model can be overridden at three levels — per-agent-type in config, per-workflow-stage in config, or per-invocation via CLI flag — with each level taking precedence over the previous
  5. Selecting a model below minimum capability for an agent type produces a clear error naming the minimum requirement before the phase runs, not discovered mid-execution
**Plans**: TBD

### Phase 3: Research Teams and Pathfinder Foundation
**Goal**: The research phase runs as a parallel team of researchers coordinated by a synthesizer lead. Pathfinder replaces /gbsd:map-codebase as the codebase intelligence layer. Index freshness is enforced before research consumes it. GBSD operates efficiently on 100k+ LOC codebases with under 5% exploration overhead.
**Depends on**: Phase 1 (task dependency chains, teammate spawn mechanism), Phase 2 (Haiku for researchers, Sonnet for synthesizer)
**Requirements**: TEAM-05, DEPR-01, PATH-04, SCALE-01, SCALE-02
**Success Criteria** (what must be TRUE):
  1. Running the research phase spawns parallel researcher instances writing to independent output files simultaneously; the synthesizer does not start until all researchers have signaled completion — wall-clock time is bounded by the slowest single researcher, not by the sum
  2. /gbsd:map-codebase is removed or emits a deprecation notice redirecting to Pathfinder index generation; users who invoke it receive a clear migration message
  3. When a Pathfinder index is present, research agents use it for codebase navigation instead of file exploration — context budget for exploration is under 5% of total tokens consumed
  4. Pathfinder index freshness is checked before each research-phase invocation using content-based staleness detection (comparing git diff output against indexed files, not timestamps) — a stale index triggers a refresh prompt rather than silently producing wrong dependency data
**Plans**: TBD

### Phase 4: Planning Correctness and Pathfinder Intelligence
**Goal**: Plans carry file ownership metadata that prevents merge conflict cascade. All human decisions are captured before execution starts, enabling skip_checkpoints autonomy. Pathfinder provides blast-radius analysis and import-graph-driven dependency derivation. Plans are sized by decision surface and structured for 70-85% wave-1 parallelizability. These are the correctness guarantees — they must exist before any parallel wave ships.
**Depends on**: Phase 1 (worktree infrastructure defines what file ownership must prevent), Phase 3 (research synthesis available to planner; Pathfinder index freshness established)
**Requirements**: PLAN-01, PLAN-02, PLAN-03, PLAN-04, PLAN-05, AUTON-01, AUTON-02, AUTON-03, AUTON-04, PATH-01, PATH-02, PATH-03
**Success Criteria** (what must be TRUE):
  1. Every plan produced by /gbsd:plan-phase includes `files_owned`, `files_created`, and `files_read` frontmatter fields — the planner's decomposition algorithm identifies file sets before naming plans; plans without these fields are rejected by the plan-checker
  2. The plan-checker serializes any two plans in the same wave that both declare ownership of the same file into different waves — the hard conflict is caught before execution, not discovered during merge-back
  3. Running /gbsd:discuss-phase exhausts all ambiguous decision points before plan creation; a phase that previously required human prompts mid-execution completes with `skip_checkpoints: true` without pausing — the 1M context window is used without the old 2-3 task token-budget cap
  4. Plans target 70-85% of tasks in Wave 1 when a Pathfinder index is available (versus 50%+ without Pathfinder); the parallelizability score is visible in plan metadata before execution starts
  5. Plans are structured as 5-10 plans of 1-2 tasks each; task dependencies within plans are auto-derived from the import graph when a Pathfinder index is present, and use manual BlockedBy fields when it is absent — both paths produce valid plans
**Plans**: TBD

### Phase 5: Execute Teams and Wave Execution
**Goal**: Multiple plans execute in parallel waves — each wave spawns N independent claude instances in N worktrees with N internal teams, merges branches back automatically, reconciles planning state from all executors, and the system can resume a crashed wave without losing completed work. This is the core value delivery of the project.
**Depends on**: Phase 1 (worktree manager, checkpoint protocol, wave synchronization), Phase 2 (key pool so each instance gets its own org key), Phase 4 (file ownership prevents merge conflict cascade; frontloaded decisions prevent mid-wave interruptions)
**Requirements**: PARA-02, PARA-03, PARA-05, PARA-06, TEAM-04
**Success Criteria** (what must be TRUE):
  1. A phase with N independent plans launches N independent claude CLI instances in N git worktrees simultaneously — wall-clock time scales as (slowest plan + merge overhead), not as the sum of all plans; verified by timing a multi-plan wave
  2. After wave completion, lock files (package-lock.json), barrel files (index.ts exports), and package.json dependency fields are merged automatically without user intervention — 80%+ of conflicts in these categories auto-resolve
  3. STATE.md and agent-history.json after a wave merge reflect decisions and metrics from all executors — no executor's work is silently discarded by last-write-wins; agent_id values in agent-history are globally unique (worktree name + timestamp)
  4. An interrupted executor is detected on next orchestrator invocation; the lead resumes the executor from the last checkpoint boundary without re-executing completed tasks; successful executors' merged work is not re-run
  5. With `--parallel-phases` flag set, phases with no declared dependency between them execute simultaneously with separate orchestrator processes and branch merges
**Plans**: TBD

### Phase 6: Verification and Plan-Phase Teams
**Goal**: Verification runs as a parallel team — one verifier per completed plan, aggregated by a coordinator, with targeted follow-up on failures. The plan-phase team runs a message-based planner-checker revision loop. When this phase completes, every workflow stage supports multi-agent parallel execution (ARCH-01 and ARCH-02 completion gate).
**Depends on**: Phase 1 (task dependency chains, teammate spawn), Phase 2 (model routing for verifiers), Phase 5 (parallel execution produces real output to verify; plan team has proven parallel plan output to check)
**Requirements**: TEAM-06, TEAM-07, ARCH-01, ARCH-02
**Success Criteria** (what must be TRUE):
  1. The verify phase spawns parallel verifier instances — one per plan completed in the wave — and aggregates their findings into a consolidated verification report; the user does not wait for sequential single-verifier execution
  2. When a verifier finds failures, the coordinator triggers follow-up verification targeting only the affected plans — full re-verification across all plans is not required for isolated failures
  3. The plan-checker runs as a message-based teammate in a revision loop with the planner — issues are communicated via structured messages and the planner receives a revision request; the loop runs at most 3 rounds before the plan is committed
  4. Every workflow stage (roadmap, plan, research, execute, verify, debug, audit, integration-check) has a documented parallel team structure showing which roles run in parallel, which are sequential, and what the coordination mechanism is — this criterion verifies ARCH-01 and ARCH-02 are satisfied
**Plans**: TBD

### Phase 7: Scale and Adoption
**Goal**: GBSD scales concurrent agents to measured API headroom rather than a hardcoded count. The worktreeManager.spawnExecutor() abstraction is the single point to update when Claude Code ships a fix for issue #33045. A cherry-pick workflow exists for upstream GSD sync. New Claude Code features are adopted within one release cycle.
**Depends on**: Phases 1-6 (full parallel execution system must be operational and validated before scaling formula can be calibrated on real bandwidth data)
**Requirements**: ARCH-03, ADOPT-01, ADOPT-02, ADOPT-03
**Success Criteria** (what must be TRUE):
  1. Concurrent agent count scales dynamically with measured API headroom — if available ITPM is 500K and each executor consumes 50K/min, the system spawns 10 executors, not a hardcoded value
  2. All worktree spawning logic is behind the `worktreeManager.spawnExecutor()` abstraction — when Claude Code ships the fix for issue #33045, there is exactly one place to update in the GBSD codebase; scattered inline worktree calls do not exist
  3. A cherry-pick workflow exists for pulling specific commits from upstream `glittercowboy/get-shit-done` — running the sync command shows which upstream commits are available and which will conflict with GBSD structural divergences; rebranding conflicts are acceptable, structural conflicts are not
  4. There is a documented GBSD release checklist that includes checking the Claude Code changelog before any worktree-related GBSD update ships — when Claude Code ships a new tool or feature relevant to GBSD, the abstraction layer means adoption is a small diff, not a rewrite
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Teams and Worktree Foundation | 0/? | Not started | - |
| 2. API Bandwidth and Model Selection | 0/? | Not started | - |
| 3. Research Teams and Pathfinder Foundation | 0/? | Not started | - |
| 4. Planning Correctness and Pathfinder Intelligence | 0/? | Not started | - |
| 5. Execute Teams and Wave Execution | 0/? | Not started | - |
| 6. Verification and Plan-Phase Teams | 0/? | Not started | - |
| 7. Scale and Adoption | 0/? | Not started | - |

## Coverage Map

All 46 v1 requirements mapped. No orphans. No duplicates.

### Full Requirement Traceability

| Requirement | Phase | Rationale |
|-------------|-------|-----------|
| TEAM-01 | Phase 1 | Hub-and-spoke foundation — the lead-lean coordination architecture that all later team rollouts depend on |
| TEAM-02 | Phase 1 | File-based checkpoint protocol — state persistence across agent process boundaries, required before any executor runs |
| TEAM-03 | Phase 1 | BlockedBy dependency chains — the wave synchronization mechanism; Wave 2 blocked until Wave 1 completions confirmed |
| TEAM-04 | Phase 1 | Executor checkpoint/resume — parallel execution multiplies failure surface; resume is not optional from day one |
| PARA-01 | Phase 1 | Git worktree isolation per executor — the isolation primitive; each independent claude CLI process gets its own worktree |
| PARA-04 | Phase 1 | Worktree manager utility — PARA-01 and PARA-04 are the same deliverable; the manager is how isolation is implemented |
| PARA-07 | Phase 1 | Resource pre-flight checks — disk space and memory validation before spawning N worktrees; entry guard for the worktree manager |
| PARA-08 | Phase 1 | Per-agent execution trace logs — debuggability requirement before any parallel execution ships |
| API-01 | Phase 2 | Multi-org API key pool — rate limits are org-scoped; multiple keys in one org provide zero independent headroom |
| API-02 | Phase 2 | Round-robin key distribution — requires circuit breaker (API-03) to be safe; both ship together |
| API-03 | Phase 2 | Circuit breaker failover — must be built before round-robin; the safety valve that makes key rotation usable |
| API-04 | Phase 2 | Multi-provider model routing — per-agent-type model assignment; coupled with key pool (capacity math requires both) |
| API-05 | Phase 2 | Cost tracking — required to validate 40-60% cost reduction claim from tiered model assignment |
| MODEL-01 | Phase 2 | Per-agent-type model config — the mechanism API-04 routing policy operates through |
| MODEL-02 | Phase 2 | Per-workflow-stage model overrides — second override level in the three-tier override hierarchy |
| MODEL-03 | Phase 2 | Runtime model selection via CLI flag — third override level; per-invocation takes precedence |
| MODEL-04 | Phase 2 | Model capability validation — prevents planner from being assigned Haiku silently |
| TEAM-05 | Phase 3 | Agent teams for research phase — first stage to receive parallel treatment; no merge conflicts (unique output files) |
| DEPR-01 | Phase 3 | Deprecate /gbsd:map-codebase — research is the first stage to use Pathfinder queries; natural deprecation point |
| PATH-04 | Phase 3 | Index freshness management — must ship before research teams consume the index; stale index in parallel run is silent corruption |
| SCALE-01 | Phase 3 | Efficient 100k+ LOC operation — first measurable at research phase via Pathfinder index navigation |
| SCALE-02 | Phase 3 | Context budget under 5% exploration — verifiable once research phase uses Pathfinder for navigation |
| PLAN-01 | Phase 4 | File ownership as first-class planning signal — must exist before any parallel wave ships to prevent merge conflict cascade |
| PLAN-02 | Phase 4 | Hard vs soft dependency distinction — part of file ownership implementation; read-only vs read-write conflict classification |
| PLAN-03 | Phase 4 | Auto-derive dependencies via Pathfinder — Pathfinder-optional enhancement to PLAN-01; manual fallback required |
| PLAN-04 | Phase 4 | Target 70-85% wave-1 parallelizability — output metric of file-ownership-first decomposition with Pathfinder |
| PLAN-05 | Phase 4 | Granular plans 5-10 x 1-2 tasks — output of decision-surface sizing; replaces 3-5 x 2-3 task structure |
| AUTON-01 | Phase 4 | Frontloaded decision capture — must exist before any parallel wave ships; parallel agents cannot pause mid-wave |
| AUTON-02 | Phase 4 | Checkpoint classification — auto-resolvable vs must-ask-human; paired with AUTON-01 as the same correctness guarantee |
| AUTON-03 | Phase 4 | Extended autonomous sessions — unblocked by decision-surface sizing (AUTON-04) and 1M context GA (March 13, 2026) |
| AUTON-04 | Phase 4 | Decision surface sizing — replaces old 200k-era token-budget cap; must be defined before AUTON-03 removes the cap |
| PATH-01 | Phase 4 | Planner blast-radius analysis — drives wave-1 parallelizability from 40-60% to 70-85% |
| PATH-02 | Phase 4 | Dependency-graph task decomposition — Pathfinder call graph queries drive automatic task wave assignment |
| PATH-03 | Phase 4 | Module coupling analysis — tightly coupled modules clustered into same task via Pathfinder |
| PARA-02 | Phase 5 | Wave-level parallel execution with merge-back — the core execute-phase deliverable; requires all Phase 1-4 prerequisites |
| PARA-03 | Phase 5 | Phase-level parallelism opt-in — gated behind proven wave-level execution; --parallel-phases flag |
| PARA-05 | Phase 5 | State reconciliation algorithm — only needed when executors actually run in parallel; belongs with the execution integration |
| PARA-06 | Phase 5 | Merge conflict auto-resolution — lock files, barrel files, config; part of the merge-back pipeline |
| TEAM-04 | Phase 5 | Executor checkpoint/resume — Phase 1 defines the protocol; Phase 5 wires it into the execute-phase workflow |
| TEAM-06 | Phase 6 | Plan-phase team (planner + checker revision loop) — requires execution output to check; message-based iteration |
| TEAM-07 | Phase 6 | Verification phase as parallel team — final stage to receive parallel treatment; requires real execution output |
| ARCH-01 | Phase 6 | Maximize parallelism at every stage — verifiable only once all workflow stages have team structures (completion gate) |
| ARCH-02 | Phase 6 | Every workflow stage supports multi-agent parallel execution — completion gate; verified by Phase 6 criterion 4 |
| ARCH-03 | Phase 7 | Dynamic agent count scaled to API bandwidth — requires measured bandwidth data from Phases 5-6 to calibrate |
| ADOPT-01 | Phase 7 | Track Claude Code Agent Teams API changes — worktreeManager.spawnExecutor() abstraction is the adoption mechanism |
| ADOPT-02 | Phase 7 | Leverage new Claude Code features as they land — adoption posture systematized after full system is stable |
| ADOPT-03 | Phase 7 | Upstream GSD sync — cherry-pick workflow defined once structural divergences are stable |

### Coverage by Group

| Group | Requirements | Phase(s) | All Mapped |
|-------|-------------|---------|------------|
| TEAM | TEAM-01/02/03/04/05/06/07 (7) | 1, 3, 5, 6 | Yes |
| PARA | PARA-01/02/03/04/05/06/07/08 (8) | 1, 5 | Yes |
| API | API-01/02/03/04/05 (5) | 2 | Yes |
| MODEL | MODEL-01/02/03/04 (4) | 2 | Yes |
| PATH | PATH-01/02/03/04 (4) | 3, 4 | Yes |
| PLAN | PLAN-01/02/03/04/05 (5) | 4 | Yes |
| AUTON | AUTON-01/02/03/04 (4) | 4 | Yes |
| DEPR | DEPR-01 (1) | 3 | Yes |
| SCALE | SCALE-01/02 (2) | 3 | Yes |
| ARCH | ARCH-01/02/03 (3) | 6, 7 | Yes |
| ADOPT | ADOPT-01/02/03 (3) | 7 | Yes |
| **Total** | **46** | **7 phases** | **46/46** |

## Research Flags

Phases likely needing `/gbsd:research-phase` during plan-phase:

- **Phase 4 (Planning Correctness):** File-ownership-first decomposition algorithm and decision-surface sizing metric are novel patterns with limited documented prior art; planner redesign has high blast radius on all downstream agents; blast-radius query semantics for PATH-01/02/03 need verification against actual Pathfinder index format

- **Phase 3 (Pathfinder Foundation):** Pathfinder index format and query API need verification; content-based staleness detection algorithm (git diff approach) needs testing against edge cases; PATH-04 interacts with research phase timing

Phases with standard patterns (skip research-phase):

- **Phase 1 (Teams and Worktree Foundation):** git worktree API is stable and well-documented; lead-managed worktree lifecycle patterns are established; checkpoint/resume follows proven patterns
- **Phase 2 (API Bandwidth and Model Selection):** Circuit-breaker patterns are well-documented in LLM gateway literature; multi-org key pool design is specified in architecture docs; model routing policy is defined
- **Phase 5 (Execute Teams and Wave Execution):** All components are tested in Phases 1-4; wiring execute-phase.md to use them is standard orchestration work
- **Phase 6 (Verification and Plan-Phase Teams):** Hub-and-spoke team patterns are established by Phase 1; verification is the same pattern as research (fan-out/fan-in with unique output files per verifier)
- **Phase 7 (Scale and Adoption):** Dynamic scaling formula is defined; abstraction layer placement is clear; cherry-pick workflow is standard git practice

---
*Roadmap created: 2026-03-15*
*Requirements: 46/46 mapped — zero orphans, zero duplicates*
*Granularity: standard (7 phases)*
*Architecture: independent claude CLI processes per worktree, not Agent Teams isolation:worktree (issue #33045)*
