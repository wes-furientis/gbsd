# GBSD Supercharge — Coverage Roadmap

**Specialty:** Requirement coverage and phase sizing
**Roadmapper:** Coverage specialist (of 3 parallel roadmappers)
**Granularity:** standard (5-8 phases)
**Requirements total:** 46 (ALL v1, no deferral)
**Generated:** 2026-03-15

---

## Coverage Map (All 46 Requirements)

| Requirement | Phase | Category |
|-------------|-------|----------|
| ARCH-01 | Phase 1 | Architecture |
| ARCH-02 | Phase 1 | Architecture |
| ARCH-03 | Phase 1 | Architecture |
| PARA-01 | Phase 2 | Parallel Execution |
| PARA-04 | Phase 2 | Parallel Execution |
| PARA-07 | Phase 2 | Parallel Execution |
| PARA-08 | Phase 2 | Parallel Execution |
| AUTON-01 | Phase 3 | Autonomy |
| AUTON-02 | Phase 3 | Autonomy |
| AUTON-03 | Phase 3 | Autonomy |
| AUTON-04 | Phase 3 | Autonomy |
| PLAN-01 | Phase 3 | Planning |
| PLAN-02 | Phase 3 | Planning |
| PLAN-03 | Phase 3 | Planning |
| PLAN-04 | Phase 3 | Planning |
| PLAN-05 | Phase 3 | Planning |
| PARA-02 | Phase 4 | Parallel Execution |
| PARA-03 | Phase 4 | Parallel Execution |
| PARA-05 | Phase 4 | Parallel Execution |
| PARA-06 | Phase 4 | Parallel Execution |
| TEAM-02 | Phase 4 | Agent Teams |
| TEAM-03 | Phase 4 | Agent Teams |
| TEAM-04 | Phase 4 | Agent Teams |
| DEPR-01 | Phase 4 | Deprecations |
| PATH-01 | Phase 5 | Pathfinder |
| PATH-02 | Phase 5 | Pathfinder |
| PATH-03 | Phase 5 | Pathfinder |
| PATH-04 | Phase 5 | Pathfinder |
| SCALE-01 | Phase 5 | Scale |
| SCALE-02 | Phase 5 | Scale |
| TEAM-05 | Phase 5 | Agent Teams |
| TEAM-06 | Phase 5 | Agent Teams |
| TEAM-07 | Phase 5 | Agent Teams |
| TEAM-01 | Phase 6 | Agent Teams |
| API-01 | Phase 6 | API Rate |
| API-02 | Phase 6 | API Rate |
| API-03 | Phase 6 | API Rate |
| API-04 | Phase 6 | API Rate |
| API-05 | Phase 6 | API Rate |
| MODEL-01 | Phase 6 | Model Selection |
| MODEL-02 | Phase 6 | Model Selection |
| MODEL-03 | Phase 6 | Model Selection |
| MODEL-04 | Phase 6 | Model Selection |
| ADOPT-01 | Phase 7 | Adoption |
| ADOPT-02 | Phase 7 | Adoption |
| ADOPT-03 | Phase 7 | Adoption |

**Coverage validation:** 46/46 requirements mapped. No orphans. No duplicates.

---

## Phase Sizes (Balance Check)

| Phase | Requirement Count | Size Assessment |
|-------|-------------------|-----------------|
| Phase 1 | 3 (ARCH-01/02/03) | Small — but foundational; architecture decisions expressed as principles not code |
| Phase 2 | 4 (PARA-01/04/07/08) | Small — targeted infrastructure primitives |
| Phase 3 | 9 (AUTON-01/02/03/04, PLAN-01/02/03/04/05) | Largest — justified: these are tightly coupled correctness guarantees |
| Phase 4 | 8 (PARA-02/03/05/06, TEAM-02/03/04, DEPR-01) | Large — the wave execution integration wave |
| Phase 5 | 9 (PATH-01/02/03/04, SCALE-01/02, TEAM-05/06/07) | Large — Pathfinder-dependent agent teams |
| Phase 6 | 9 (TEAM-01, API-01/02/03/04/05, MODEL-01/02/03/04) | Large — API infrastructure and model routing |
| Phase 7 | 3 (ADOPT-01/02/03) | Small — ongoing practices, not shipped features |

**Balance assessment:** Phases 1, 2, 7 are small (3-4 reqs) but justified by their nature — architectural principles, isolated infrastructure primitives, and ongoing adoption practices respectively. Phases 3-6 are 8-9 requirements each, which is balanced for this requirement density. Standard granularity with 7 phases is within the 5-8 target on the higher end; consider merging Phase 1 into Phase 2 if synthesis roadmap prefers tighter clustering (see note at bottom).

---

## Phase Details

### Phase 1: Architecture Principles
**Goal:** The parallel-by-default architecture is codified — every execution stage has a defined parallel execution model and the system scales to available API bandwidth rather than a hardcoded agent count.
**Depends on:** Nothing (first phase)
**Requirements:** ARCH-01, ARCH-02, ARCH-03
**Requirement count:** 3

**Success Criteria** (what must be TRUE when this phase completes):
1. Every workflow stage (roadmap, plan, check, verify, research, synthesize, debug, audit, integration-check) has documented parallel execution support in its agent definition — a human reviewing the agents can verify this in the workflow files
2. The system default is N-agent parallel execution where N is derived from available API bandwidth, not a hardcoded value — running `/gbsd:execute` with no flags launches parallel agents, not a single agent
3. Agent count scales up and down as API rate capacity changes mid-session — adding more API keys during a run increases concurrency without a restart
4. Architecture principles are enforced at the config schema level — any agent definition missing parallel support is flagged as invalid on schema validation

**Plans:** TBD

---

### Phase 2: Worktree Isolation Infrastructure
**Goal:** Each parallel executor gets its own git worktree with automatic lifecycle management — worktrees are created reliably, cleaned up on failure, and stale orphans from crashed sessions are detected and recovered on restart.
**Depends on:** Phase 1 (architecture principles define what worktree isolation must satisfy)
**Requirements:** PARA-01, PARA-04, PARA-07, PARA-08
**Requirement count:** 4

**Success Criteria** (what must be TRUE when this phase completes):
1. Launching an executor for a plan creates a new git worktree at `~/.claude/worktrees/{branch}` — the executor's file writes do not appear in the main working directory until merge-back
2. If an executor crashes mid-task, the worktree is cleaned up — restarting the orchestrator does not leave orphaned directories consuming disk space
3. On orchestrator startup after a crash, stale worktrees from the previous session are listed and the user is offered resume/retry/skip — no silent data loss
4. Before spawning N worktrees, the system checks available disk space (N × 700 MB) and available memory, and refuses to proceed with a clear error if resources are insufficient (PARA-07)
5. Each executor writes a per-agent trace log to a predictable path — when a parallel agent fails, the user can read its complete execution trace without searching (PARA-08)

**Plans:** TBD

---

### Phase 3: Planner Redesign and Frontloaded Decisions
**Goal:** Plans are structured so parallel execution can succeed without human interruption — every plan declares file ownership upfront, every decision that would block an executor is captured before execution begins, and plans are sized by decision surface rather than token budget.
**Depends on:** Phase 2 (worktree isolation defines what file ownership must prevent — same-file writes across worktrees)
**Requirements:** AUTON-01, AUTON-02, AUTON-03, AUTON-04, PLAN-01, PLAN-02, PLAN-03, PLAN-04, PLAN-05
**Requirement count:** 9

**Success Criteria** (what must be TRUE when this phase completes):
1. Every plan produced by `/gbsd:plan-phase` includes `files_owned`, `files_created`, and `files_read` frontmatter fields — a human reviewing a plan can see exactly which files it writes to before execution starts
2. Two plans that write to the same file are automatically serialized into different waves — a user cannot accidentally launch parallel plans that will conflict
3. Running `/gbsd:discuss-phase` exhausts all decision points before execution — the discussion session asks every question that would otherwise interrupt an executor mid-wave, and the answers are committed to the plan
4. A plan running with `skip_checkpoints: true` completes autonomously end-to-end without human prompts — the 1M context window is used, not the old 2-3 task token-budget cap
5. Each plan shows a parallelizability score (ratio of wave-1 tasks to total tasks) before execution — a score below 0.3 triggers a warning that most work will run serially
6. Plans with no shared file conflicts default to granular 5-10 sub-plans of 1-2 tasks each, not 3-5 larger plans — the plan list shows more granular entries than before the redesign
7. Task dependencies within a plan are derived from import graph data when Pathfinder index is present, and from manual `BlockedBy` fields when it is absent — both paths produce valid plans (PLAN-03 graceful degradation)

**Plans:** TBD

---

### Phase 4: Wave Execution and Merge-Back
**Goal:** Multiple plans execute in parallel waves — each wave runs its plans concurrently in isolated worktrees, merges back automatically, and the system can resume a crashed wave without losing completed work.
**Depends on:** Phase 3 (file ownership must exist before parallel waves run; without it, conflict cascade is guaranteed)
**Requirements:** PARA-02, PARA-03, PARA-05, PARA-06, TEAM-02, TEAM-03, TEAM-04, DEPR-01
**Requirement count:** 8

**Success Criteria** (what must be TRUE when this phase completes):
1. Running `/gbsd:execute` on a phase with 4 independent plans launches all 4 concurrently — the terminal shows 4 parallel agent sessions running, not sequential execution
2. After a wave completes, all executor branches are merged back to main automatically — the user does not need to run any git commands manually; the merged result is on main before the next wave starts
3. Lock files (package-lock.json), barrel files (index.ts exports), and package.json dependency fields are merged automatically with no user intervention — 80%+ of conflicts in these categories resolve without prompting
4. STATE.md and agent-history.json after a wave merge reflect work from all executors — no executor's decisions or metrics are silently discarded (union strategy for decisions, sum for counts)
5. If one executor in a wave fails, the others complete and merge successfully — the failed executor's worktree is preserved for retry or skip, but does not block the successful executors
6. An interrupted wave (orchestrator crash mid-wave) can be resumed — the user runs `/gbsd:resume` and only incomplete executors re-run; completed executors' work is already merged and not repeated (TEAM-04)
7. Plans with explicit `parallel_with: [Phase N]` annotations execute as a phase-level parallel (PARA-03) — independent phases run concurrently when the flag is set
8. `/gbsd:map-codebase` is removed or replaced by a Pathfinder wrapper — existing codebase mapping is handled by Pathfinder index generation; the old command either routes to Pathfinder or is deprecated with a clear migration message (DEPR-01)

**Plans:** TBD

---

### Phase 5: Pathfinder Integration and Parallel Agent Teams
**Goal:** The Pathfinder code intelligence index drives automatic file ownership derivation and wave assignment, pushing wave-1 parallelizability from 40-60% to 70-85%, while parallel agent teams handle research, planning, and verification using the same hub-and-spoke coordination pattern.
**Depends on:** Phase 4 (wave execution infrastructure must work before Pathfinder enhances it; agent teams for research/plan/verify build on the team coordination patterns established in Phase 4)
**Requirements:** PATH-01, PATH-02, PATH-03, PATH-04, SCALE-01, SCALE-02, TEAM-05, TEAM-06, TEAM-07
**Requirement count:** 9

**Success Criteria** (what must be TRUE when this phase completes):
1. On a codebase with a `.code-intel/` Pathfinder index present, the planner automatically assigns `files_owned` to each plan without manual user input — the wave assignment produced by Pathfinder matches or exceeds what manual assignment would produce
2. Wave-1 parallelizability score on a typical plan is 70-85% when Pathfinder index is present, versus 40-60% without it — the score improvement is visible in the plan metadata
3. When `.code-intel/` is absent or stale, GBSD falls back to manual file ownership assignment without error — the absence of Pathfinder slows planning but does not break it
4. Before any executor is spawned, the Pathfinder index is checked for staleness using content-based detection (`git diff --name-only` vs indexed files) — a stale index triggers a refresh prompt, not silent stale data
5. On a 100k+ LOC codebase, GBSD's context budget for exploration is under 5% of total tokens consumed — the Pathfinder index is used for navigation instead of file exploration (SCALE-01, SCALE-02)
6. Research phase runs with parallel researcher agents coordinated by a team lead — each researcher handles a separate topic, findings are synthesized by the lead rather than running one sequential research session (TEAM-05)
7. Plan phase runs a planner-checker team loop — the checker reviews the plan and sends revision messages to the planner before the plan is finalized, rather than requiring a manual human review cycle (TEAM-06)
8. Verification phase runs with parallel verifier agents — each verifier covers a separate success criterion, failures are aggregated by the lead, and follow-up verification is automatically triggered on failures (TEAM-07)

**Plans:** TBD

---

### Phase 6: API Management and Per-Agent Model Selection
**Goal:** Parallel execution at scale is sustainable — API rate limits are managed across multiple organizations, costs are minimized by routing each agent type to the appropriate model tier, and every model assignment is configurable and validated.
**Depends on:** Phase 5 (the full parallel execution system must be operational before API management optimizations are meaningful — you need load before you can optimize it)
**Requirements:** TEAM-01, API-01, API-02, API-03, API-04, API-05, MODEL-01, MODEL-02, MODEL-03, MODEL-04
**Requirement count:** 10

**Note on TEAM-01 placement:** TEAM-01 (hub-and-spoke lean lead) is placed here rather than Phase 5 because it requires both Pathfinder (Phase 5 — lead reads index summaries, not file contents) and API management (this phase — lean lead is only meaningful when multiple orgs provide sufficient rate headroom for N specialist agents). It is the capstone of the agent team architecture.

**Success Criteria** (what must be TRUE when this phase completes):
1. Configuring multiple Anthropic org API keys in `gbsd-config.json` causes parallel executors to be assigned round-robin across orgs — each executor's API calls consume a different org's rate limit, not all competing on one org's limit
2. When an org's API key returns a 429, that org is blacklisted temporarily and executors are rerouted to other orgs automatically — execution continues without user intervention; the blacklist clears after exponential backoff
3. The planner agent uses Opus 4.6 by default, executor agents use Sonnet 4.6, and research/verification agents use Haiku 4.5 — this is the default configuration, not requiring user setup, and produces 40-60% cost reduction vs all-Opus baseline
4. Any agent's model can be overridden per-agent-type in config, per-workflow-stage in config, or per-invocation via CLI flag — three levels of override, each taking precedence over the previous
5. Selecting a model below minimum capability for an agent type produces a clear error with the minimum requirement stated — the user cannot accidentally run the planner on Haiku without being warned (MODEL-04)
6. API cost is tracked and reported per session — the user can see total tokens consumed and estimated cost per workflow stage after each run (API-05)
7. The hub-and-spoke lead agent runs at approximately 15% context budget — context usage metrics show the lead's context consumption stays in coordination mode rather than loading full file contents (TEAM-01)

**Plans:** TBD

---

### Phase 7: Upstream Compatibility and Claude Code Adoption
**Goal:** GBSD remains current with Claude Code's evolving Agent Teams API and absorbs beneficial upstream GSD improvements without structural conflicts.
**Depends on:** Phase 6 (full system must be operational before adoption practices are meaningful — you need a stable codebase to sync against)
**Requirements:** ADOPT-01, ADOPT-02, ADOPT-03
**Requirement count:** 3

**Note on phase size:** This phase is intentionally small (3 requirements) because these requirements are ongoing operational practices and tooling, not a one-time deliverable. They formalize how GBSD evolves post-v1 — the abstraction layer for worktree spawning, the Claude Code feature monitoring process, and the upstream sync tooling.

**Success Criteria** (what must be TRUE when this phase completes):
1. All worktree spawning logic is behind a `worktreeManager.spawnExecutor()` abstraction — when Claude Code ships the fix for issue #33045 (teammate worktree isolation), there is exactly one place to update in the GBSD codebase, not a scattered set of inline worktree calls
2. There is a documented GBSD release checklist that includes checking the Claude Code changelog before any worktree-related GBSD update ships — the changelog review is a formal gate, not an informal practice
3. A cherry-pick workflow exists for pulling specific commits from upstream `glittercowboy/get-shit-done` — running the sync command shows which upstream commits are available to cherry-pick and which will conflict with GBSD's structural divergences
4. When Claude Code ships a new tool or feature relevant to GBSD (e.g., worktree isolation fix, new task tool), the GBSD codebase adopts it within one release cycle — the abstraction layer means the adoption is a small diff, not a rewrite

**Plans:** TBD

---

## Phases Summary

- [ ] **Phase 1: Architecture Principles** — Codify the parallel-by-default execution model across all workflow stages
- [ ] **Phase 2: Worktree Isolation Infrastructure** — Build the lifecycle-managed git worktree isolation primitive
- [ ] **Phase 3: Planner Redesign and Frontloaded Decisions** — Redesign planning for parallel correctness: file ownership, decision capture, decision-surface sizing
- [ ] **Phase 4: Wave Execution and Merge-Back** — Wire isolation, planning, and state reconciliation into end-to-end parallel wave execution
- [ ] **Phase 5: Pathfinder Integration and Parallel Agent Teams** — Add code intelligence-driven planning and parallel teams for research, planning, and verification
- [ ] **Phase 6: API Management and Per-Agent Model Selection** — Add multi-org rate management, model routing, cost tracking, and lean lead agent architecture
- [ ] **Phase 7: Upstream Compatibility and Claude Code Adoption** — Formalize the abstraction and process for staying current with Claude Code and GSD upstream

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Architecture Principles | 0/? | Not started | - |
| 2. Worktree Isolation Infrastructure | 0/? | Not started | - |
| 3. Planner Redesign and Frontloaded Decisions | 0/? | Not started | - |
| 4. Wave Execution and Merge-Back | 0/? | Not started | - |
| 5. Pathfinder Integration and Parallel Agent Teams | 0/? | Not started | - |
| 6. API Management and Per-Agent Model Selection | 0/? | Not started | - |
| 7. Upstream Compatibility and Claude Code Adoption | 0/? | Not started | - |

---

## Coverage Verification

### Full Requirement-to-Phase Map (verification pass)

**ARCH group (3):**
- ARCH-01 → Phase 1 (maximize parallelism at every stage)
- ARCH-02 → Phase 1 (every workflow stage supports multi-agent parallel execution)
- ARCH-03 → Phase 1 (dynamic agent count scales to API bandwidth)

**AUTON group (4):**
- AUTON-01 → Phase 3 (frontloaded decision capture)
- AUTON-02 → Phase 3 (checkpoint classification)
- AUTON-03 → Phase 3 (extended autonomous sessions)
- AUTON-04 → Phase 3 (decision surface sizing)

**PATH group (4):**
- PATH-01 → Phase 5 (blast-radius impact analysis)
- PATH-02 → Phase 5 (dependency-graph task decomposition)
- PATH-03 → Phase 5 (module coupling analysis for clustering)
- PATH-04 → Phase 5 (index freshness management)

**PLAN group (5):**
- PLAN-01 → Phase 3 (file ownership as planning signal)
- PLAN-02 → Phase 3 (hard vs soft file dependencies)
- PLAN-03 → Phase 3 (auto-derive dependencies from Pathfinder)
- PLAN-04 → Phase 3 (target 70-85% wave-1 parallelizability score)
- PLAN-05 → Phase 3 (granular 5-10 plans × 1-2 tasks)

**PARA group (8):**
- PARA-01 → Phase 2 (git worktree isolation per executor)
- PARA-02 → Phase 4 (wave-level parallel execution with merge-back)
- PARA-03 → Phase 4 (phase-level parallelism opt-in)
- PARA-04 → Phase 2 (worktree manager utility)
- PARA-05 → Phase 4 (state reconciliation algorithm)
- PARA-06 → Phase 4 (merge conflict auto-resolution)
- PARA-07 → Phase 2 (resource pre-flight checks)
- PARA-08 → Phase 2 (per-agent execution trace logs)

**TEAM group (7):**
- TEAM-01 → Phase 6 (hub-and-spoke lean lead)
- TEAM-02 → Phase 4 (file-based checkpoint protocol)
- TEAM-03 → Phase 4 (task dependency chains BlockedBy)
- TEAM-04 → Phase 4 (executor checkpoint/resume)
- TEAM-05 → Phase 5 (agent teams for research)
- TEAM-06 → Phase 5 (agent teams for plan phase)
- TEAM-07 → Phase 5 (agent teams for verification)

**DEPR group (1):**
- DEPR-01 → Phase 4 (deprecate /gbsd:map-codebase)

**API group (5):**
- API-01 → Phase 6 (multi-org API key pool)
- API-02 → Phase 6 (round-robin key distribution)
- API-03 → Phase 6 (circuit breaker failover)
- API-04 → Phase 6 (multi-provider model routing)
- API-05 → Phase 6 (cost tracking via gateway)

**MODEL group (4):**
- MODEL-01 → Phase 6 (per-agent-type model config)
- MODEL-02 → Phase 6 (per-workflow-stage model overrides)
- MODEL-03 → Phase 6 (runtime model selection via CLI)
- MODEL-04 → Phase 6 (model capability validation)

**SCALE group (2):**
- SCALE-01 → Phase 5 (100k+ LOC via Pathfinder index)
- SCALE-02 → Phase 5 (context budget optimization <5% exploration)

**ADOPT group (3):**
- ADOPT-01 → Phase 7 (track Claude Code Agent Teams API changes)
- ADOPT-02 → Phase 7 (leverage new Claude Code features)
- ADOPT-03 → Phase 7 (optional upstream GSD sync)

### Final Count

| Group | Requirements | All Mapped |
|-------|--------------|------------|
| ARCH | 3 | Yes |
| AUTON | 4 | Yes |
| PATH | 4 | Yes |
| PLAN | 5 | Yes |
| PARA | 8 | Yes |
| TEAM | 7 | Yes |
| DEPR | 1 | Yes |
| API | 5 | Yes |
| MODEL | 4 | Yes |
| SCALE | 2 | Yes |
| ADOPT | 3 | Yes |
| **Total** | **46** | **46/46** |

**Coverage: 46/46 requirements mapped. Zero orphans. Zero duplicates.**

---

## Phase Balance Analysis

| Phase | Req Count | % of Total | Assessment |
|-------|-----------|------------|------------|
| Phase 1 | 3 | 6.5% | Small but correct — ARCH principles are architectural decisions, not implementation work |
| Phase 2 | 4 | 8.7% | Small but atomic — worktree infrastructure is a focused primitive |
| Phase 3 | 9 | 19.6% | Largest — justified because AUTON + PLAN are inseparable correctness guarantees |
| Phase 4 | 8 | 17.4% | Heavy but appropriate — this is the integration wave wiring all prior work |
| Phase 5 | 9 | 19.6% | Large — Pathfinder + scale + three agent team types have natural cohesion |
| Phase 6 | 10 | 21.7% | Largest — API management + model routing are distinct but both land in the "optimization after validation" window |
| Phase 7 | 3 | 6.5% | Small — ongoing practice, not a feature delivery |

**Merge option for synthesis:** If synthesis roadmap prefers 6 phases over 7, Phase 1 (ARCH principles) can merge into Phase 2. ARCH-01/02/03 are architecture decisions that manifest as config schema, agent definition structure, and dynamic scaling — all of which are implemented alongside the worktree infrastructure. The merged phase would be: "Worktree Isolation + Architecture Principles" (7 requirements). This is the recommended merge if 7 phases feels top-heavy.

**Split option for synthesis:** If synthesis prefers tighter phases, Phase 6 (10 requirements) could split: API management (API-01/02/03/04/05 + TEAM-01 = 6 reqs) as one phase, model selection (MODEL-01/02/03/04 = 4 reqs) as a separate phase. MODEL requirements are self-contained and can ship independently of the API key pool work.

---

## Dependency Chain

```
Phase 1 (ARCH principles)
    |
    v
Phase 2 (Worktree isolation)
    |
    v
Phase 3 (Planner redesign + decision frontloading)
    |
    v
Phase 4 (Wave execution + merge-back)
    |
    v
Phase 5 (Pathfinder + parallel agent teams)
    |
    v
Phase 6 (API management + model routing)
    |
    v
Phase 7 (Adoption practices)
```

Every phase delivers a coherent, independently verifiable capability. Phases 1-4 are the MVP critical path. Phases 5-7 are enhancements that increase quality, scale, and sustainability.

---

## Standalone Value Test

For each phase, can a user observe tangible value at completion?

| Phase | Standalone Value | User-Observable Outcome |
|-------|-----------------|------------------------|
| Phase 1 | Marginal alone | Architecture principles in agent definitions; parallelism-by-default configured — visible in config schema and agent files |
| Phase 2 | Yes | Running execute creates visible worktrees; crash recovery works; trace logs exist |
| Phase 3 | Yes | Plans have file ownership fields; discuss session asks all questions upfront; plans show parallelizability score |
| Phase 4 | Yes — core value | Multiple agents execute visibly in parallel; merge-back happens automatically; wave resume works |
| Phase 5 | Yes | Pathfinder drives file ownership (no manual assignment); scale to 100k+ LOC; parallel research/plan/verify teams |
| Phase 6 | Yes | API keys from multiple orgs; automatic 429 failover; cost report after each run; model per agent type |
| Phase 7 | Yes | Cherry-pick workflow for upstream sync; abstracted worktree spawning ready for API fix |

Phase 1 is the weakest standalone — but architectural decisions expressed in code (config schema, agent definitions with parallel support documented) are observable and reviewable. Combined with Phase 2 it produces the complete foundation.

---

*Coverage roadmap complete. 46/46 requirements mapped. 7 phases, standard granularity.*
*Synthesis: merge Phases 1+2 if 6-phase structure preferred; split Phase 6 if 8-phase structure preferred.*
