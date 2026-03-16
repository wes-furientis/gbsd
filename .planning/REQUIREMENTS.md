# Requirements: GBSD Supercharge

**Defined:** 2026-03-15
**Core Value:** Phases execute autonomously end-to-end — hit execute and walk away. 5-8x faster, 40-60% cost reduction, 100k+ LOC capable.

## v1 Requirements

All features ship in v1. No deferral.

### Architecture Principle

- [ ] **ARCH-01**: Maximize parallelism at every workflow stage — every idle API rate slot is wasted wall-clock time. Default to N agents where N = available bandwidth, not 1 agent because it's simpler. Total context consumed is roughly the same; parallelism is free speed.
- [ ] **ARCH-02**: Every workflow stage (roadmap, plan, check, verify, research, synthesize, debug, audit, integration-check) must support multi-agent parallel execution, coordinated by a team lead
- [ ] **ARCH-03**: Dynamic agent count — scale concurrent agents to available API rate bandwidth, not fixed counts

### Autonomy

- [ ] **AUTON-01**: Frontloaded decision capture — all human decisions gathered during discuss/plan phases, not during execution
- [ ] **AUTON-02**: Checkpoint classification — auto-resolvable vs must-ask-human, with auto-resolve as default
- [ ] **AUTON-03**: Extended autonomous execution sessions leveraging 1M token context windows
- [ ] **AUTON-04**: Decision surface sizing — plans sized by decision count, not token budget

### Pathfinder Integration

- [ ] **PATH-01**: Planner uses Pathfinder blast-radius for change impact analysis before task decomposition
- [ ] **PATH-02**: Dependency-graph-driven task decomposition via Pathfinder call graphs and import analysis
- [ ] **PATH-03**: Module coupling analysis for automatic task clustering (tightly coupled modules = same task)
- [ ] **PATH-04**: Index freshness management — content-based staleness detection, auto-refresh when stale

### Planning Redesign

- [ ] **PLAN-01**: File ownership as first-class planning signal — identify file sets before task decomposition
- [ ] **PLAN-02**: Distinguish hard vs soft dependencies (read-only vs read-write file conflicts)
- [ ] **PLAN-03**: Auto-derive task dependencies from imports/API usage via Pathfinder
- [ ] **PLAN-04**: Target 70-85% wave-1 parallelizability (up from 40-60%)
- [ ] **PLAN-05**: Granular plans — 5-10 plans × 1-2 tasks each (vs 3-5 × 2-3)

### Parallel Execution

- [ ] **PARA-01**: Git worktree isolation per executor — each parallel plan gets its own worktree
- [ ] **PARA-02**: Wave-level parallel execution with merge-back after each wave
- [ ] **PARA-03**: Phase-level parallelism for independent phases (opt-in via --parallel-phases flag)
- [ ] **PARA-04**: Worktree manager utility — create/clean/merge worktrees, stale detection on restart, try/finally cleanup
- [ ] **PARA-05**: State reconciliation algorithm for STATE.md and agent-history.json after parallel merges
- [ ] **PARA-06**: Merge conflict auto-resolution for lock files, config, barrel files (80%+ auto-resolution)
- [ ] **PARA-07**: Resource pre-flight checks — disk space and memory validation before spawning N worktrees
- [ ] **PARA-08**: Per-agent execution trace logs — one log file per executor for debuggability

### Agent Teams

- [ ] **TEAM-01**: Hub-and-spoke agent teams — lead stays lean (~15% context), delegates to specialists
- [ ] **TEAM-02**: File-based checkpoint protocol for state persistence across agent boundaries
- [ ] **TEAM-03**: Task dependency chains (BlockedBy) for wave synchronization
- [ ] **TEAM-04**: Executor checkpoint/resume — interrupted executors can be continued or retried
- [ ] **TEAM-05**: Agent teams for research phase — lead coordinates parallel researchers, redirects based on findings, iterative synthesis rather than one-shot fire-and-forget
- [ ] **TEAM-06**: Agent teams for plan phase — planner + checker in team-based revision loop with message-based iteration
- [ ] **TEAM-07**: Agent teams for verification phase — lead coordinates parallel verifiers, aggregates results, triggers follow-up verification on failures

### Deprecations

- [ ] **DEPR-01**: Deprecate /gbsd:map-codebase — replaced by Pathfinder index generation; existing codebase mapping workflow becomes a thin wrapper or is removed entirely

### API Rate Management

- [ ] **API-01**: Multi-org API key pool — 2-3 separate Anthropic orgs for independent rate limits
- [ ] **API-02**: Round-robin key distribution across parallel executor sessions
- [ ] **API-03**: Circuit breaker failover — detect 429s, blacklist keys temporarily, exponential backoff
- [ ] **API-04**: Multi-provider model routing — per-agent-type model assignment with failover chains
- [ ] **API-05**: Cost tracking via unified gateway (LiteLLM or equivalent)

### Per-Component Model Selection

- [ ] **MODEL-01**: Per-agent-type model configuration — individually configurable model for every agent (planner, executor, researcher, verifier, checker, mapper, synthesizer, debugger, roadmapper, auditor, integration-checker)
- [ ] **MODEL-02**: Per-workflow-stage model overrides — ability to set model at the workflow level (research stage, plan stage, execute stage, verify stage)
- [ ] **MODEL-03**: Runtime model selection — ability to override model per invocation via CLI flag
- [ ] **MODEL-04**: Model capability validation — verify selected model meets minimum requirements for agent type (e.g., planner needs high reasoning, mapper can use lower tier)

### Scale

- [ ] **SCALE-01**: Efficient operation on 100k+ LOC codebases via Pathfinder index (not exploration)
- [ ] **SCALE-02**: Context budget optimization — <5% exploration overhead (down from 20-40%)

### Claude Code Adoption

- [ ] **ADOPT-01**: Track and adopt Claude Code Agent Teams API changes rapidly — abstraction layer for worktree spawning
- [ ] **ADOPT-02**: Leverage new Claude Code features (worktree support, new tools) as they land
- [ ] **ADOPT-03**: Optional upstream GSD sync — ability to cherry-pick upstream improvements

## Out of Scope

| Feature | Reason |
|---------|--------|
| Rewriting GSD workflow order | The question → research → plan → execute → verify pipeline stays; improvements are about parallelism and autonomy |
| Custom UI / dashboard | GBSD is CLI-first; ASCII progress tables are sufficient and work over SSH |
| Real-time inter-agent communication | Race conditions, destroys isolation guarantee; hub-and-spoke via task lists instead |
| Task-level parallelism within plans | Complexity/benefit ratio too poor; plan-level parallelism first |
| Automatic phase parallelism detection | False positive cost too high; opt-in annotation instead |
| Shared database across worktrees | Race conditions, port conflicts; filesystem-only isolation |
| Non-AI runtime support | GBSD targets AI coding assistants only |
| Distributed transaction semantics for merges | Enterprise complexity; merge-what-succeeded + retry/skip is sufficient |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| ADOPT-01 | Phase 7 | Pending |
| ADOPT-02 | Phase 7 | Pending |
| ADOPT-03 | Phase 7 | Pending |
| API-01 | Phase 2 | Pending |
| API-02 | Phase 2 | Pending |
| API-03 | Phase 2 | Pending |
| API-04 | Phase 2 | Pending |
| API-05 | Phase 2 | Pending |
| ARCH-01 | Phase 6 | Pending |
| ARCH-02 | Phase 6 | Pending |
| ARCH-03 | Phase 7 | Pending |
| AUTON-01 | Phase 4 | Pending |
| AUTON-02 | Phase 4 | Pending |
| AUTON-03 | Phase 4 | Pending |
| AUTON-04 | Phase 4 | Pending |
| DEPR-01 | Phase 3 | Pending |
| MODEL-01 | Phase 2 | Pending |
| MODEL-02 | Phase 2 | Pending |
| MODEL-03 | Phase 2 | Pending |
| MODEL-04 | Phase 2 | Pending |
| PARA-01 | Phase 1 | Pending |
| PARA-02 | Phase 5 | Pending |
| PARA-03 | Phase 5 | Pending |
| PARA-04 | Phase 1 | Pending |
| PARA-05 | Phase 5 | Pending |
| PARA-06 | Phase 5 | Pending |
| PARA-07 | Phase 1 | Pending |
| PARA-08 | Phase 1 | Pending |
| PATH-01 | Phase 4 | Pending |
| PATH-02 | Phase 4 | Pending |
| PATH-03 | Phase 4 | Pending |
| PATH-04 | Phase 3 | Pending |
| PLAN-01 | Phase 4 | Pending |
| PLAN-02 | Phase 4 | Pending |
| PLAN-03 | Phase 4 | Pending |
| PLAN-04 | Phase 4 | Pending |
| PLAN-05 | Phase 4 | Pending |
| SCALE-01 | Phase 3 | Pending |
| SCALE-02 | Phase 3 | Pending |
| TEAM-01 | Phase 1 | Pending |
| TEAM-02 | Phase 1 | Pending |
| TEAM-03 | Phase 1 | Pending |
| TEAM-04 | Phase 1 | Pending |
| TEAM-05 | Phase 3 | Pending |
| TEAM-06 | Phase 6 | Pending |
| TEAM-07 | Phase 6 | Pending |

**Coverage:**
- v1 requirements: 46 total (45 active + 1 deprecation)
- Mapped to phases: 46
- Unmapped: 0

---
*Requirements defined: 2026-03-15*
*Last updated: 2026-03-15 after initial definition*
