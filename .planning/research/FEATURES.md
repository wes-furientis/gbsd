# Feature Research

**Domain:** AI agent orchestration framework — parallel autonomous code execution
**Researched:** 2026-03-15
**Confidence:** HIGH (primary sources from project docs, MEDIUM from ecosystem web research)

---

## Feature Landscape

### Table Stakes (Users Expect These)

These are non-negotiable. An orchestration framework missing any of these feels broken or unprofessional. Developers evaluating alternatives (Superset IDE, Dagger container-use, manual git worktree workflows) assume all of these exist.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Sequential fallback mode** | Parallel is opt-in; users expect a working single-agent path that behaves identically to upstream GSD | LOW | Already exists; must not regress when parallelism is added |
| **Git worktree isolation per agent** | The ecosystem (Cursor, Codex, Container-Use, Superset) has converged on worktrees as the standard isolation primitive for parallel coding agents | MEDIUM | Branch naming convention: `gsd/exec/{phase}-{plan}-w{wave}` |
| **Wave-based parallel execution** | Plans with no shared file ownership execute simultaneously; serial execution with 5+ independent plans is visibly wasteful | MEDIUM | Core PARA-01/02 requirements |
| **Merge-back after each wave** | Agents have no value until their work lands on main; users expect completed waves to integrate before starting the next | MEDIUM | Already partially designed in merge-back-protocol.md |
| **Interruption detection on restart** | Agent sessions crash; the system must detect stale worktrees from a previous run and offer resume/retry/skip choices | MEDIUM | Standard LangGraph and GSD upstream pattern; see PARA-04 |
| **Checkpoint-to-file protocol** | Executor progress must survive session death; file-based progress (SUMMARY.md, task-status files) is the standard pattern across all agent frameworks | LOW | Already exists in GSD; TEAM-02 extends for parallel context |
| **Config-controlled parallelism depth** | Power users need `max_concurrent_agents`, min thresholds, and timeout controls to tune for their API plan and hardware | LOW | Already partially designed in config schema |
| **Backward-compatible config migration** | Existing GSD/GBSD projects must not break when the new parallel config fields are added | LOW | Auto-populate defaults from existing `parallelization: true` field |
| **Atomic commits per task** | Users need a clean git history to understand what each agent did; each task = one commit is the standard pattern | LOW | Already exists in GSD |
| **Resource pre-flight checks** | Disk space and memory checks before spawning N worktrees prevent silent failures mid-execution | LOW | Designed in GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md section 8 |
| **Per-agent execution trace logs** | When a parallel agent fails, users need a trace of what it did; one log file per agent is the minimum debuggability requirement | LOW | Designed in architecture doc section 11 |

---

### Differentiators (Competitive Advantage)

These are what make GBSD stand out from the emerging ecosystem of parallel coding agent tools. The current landscape (Superset IDE, Dagger container-use, manual worktree scripts) handles isolation but not orchestration — they give you N parallel sessions but no planner, no wave structure, no decision frontloading, and no state reconciliation.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Frontloaded decision capture** | All human decisions gathered during discuss/plan phases rather than as execution-time interrupts. Parallel agents can't pause mid-wave to ask a question — the only way to achieve autonomous wave execution is to have answered all questions before wave start. This is the architectural insight that makes autonomy viable. | MEDIUM | AUTON-01/02/04; requires discuss-phase enhancement and plan-checker validation |
| **Decision surface sizing** | Plans sized by decision count (how many tasks require human input) not token budget. With 1M context windows, the old 2-3 task / 50% context cap is obsolete. This removes the arbitrary cap that caused excessive plan fragmentation. | MEDIUM | AUTON-04; requires planner rewrite |
| **File ownership as first-class planning signal** | Planner assigns exclusive file ownership per plan before wave assignment. Two plans owning the same file get serialized automatically. This is the root cause fix for merge conflicts — prevent them at planning time rather than resolving them at merge time. | HIGH | PLAN-01/02; requires planner redesign; the Clash tool (from ecosystem research) detects conflicts after the fact — GBSD prevents them before the fact |
| **Pathfinder-driven dependency decomposition** | Task boundaries derived from the import graph and call graph (module coupling analysis) rather than arbitrary manual splitting. Produces natural task units that are already maximally independent. | HIGH | PATH-01/02/03; requires Pathfinder integration; enables 70-85% wave-1 parallelizability vs current 40-60% |
| **State reconciliation algorithm** | After a wave, STATE.md and agent-history.json are merged from N executor copies using typed merge strategies (union decisions, sum metrics, last-write-wins for immutable sections). No other tool in the ecosystem handles this — they stop at code merges and leave planning state in an undefined condition. | MEDIUM | PARA-05; designed in architecture doc section 2 |
| **Merge conflict auto-resolution for known patterns** | Lock files (package-lock.json), barrel files (index.ts), and config files (package.json dependencies) have deterministic merge strategies (union, regenerate, jq-merge). Targeting 80%+ auto-resolution rate for these categories eliminates the manual conflict resolution bottleneck. | MEDIUM | PARA-06; requires per-file-type merge handlers in merge-back protocol |
| **Multi-org API key pool with circuit breaker** | Round-robin distribution of API keys across parallel executor sessions with exponential backoff and key blacklisting on 429 responses. Each Anthropic org has independent rate limits, so 3 orgs = 3x effective throughput. | MEDIUM | API-01/02/03; well-established pattern in LLM gateway literature; straightforward per-process env var distribution |
| **Phase-level parallelism with dependency analysis** | Independent phases (non-overlapping file sets, different requirement categories) execute in separate worktrees simultaneously, reducing project duration from sum-of-phases to critical-path. Dependency detection algorithm uses file path overlap scoring + requirement category analysis. | HIGH | PARA-03; opt-in via `--parallel-phases` flag; higher risk than wave-level due to cross-phase merge surface |
| **Hub-and-spoke agent teams with lean lead** | Lead agent stays at ~15% context budget (coordination only), delegates implementation to specialist subagents. Lead reads index summaries, not file contents. This is the scalability pattern for 100k+ LOC codebases where exploration tax dominates. | HIGH | TEAM-01/02/03; requires Pathfinder index as navigation layer |
| **Node repair with contextual auto-recovery** | When a task fails, the orchestrator automatically spawns a repair agent with the failure context, task history, and a constrained budget for retries before escalating to human. Reduces execution-time interruptions for recoverable failures. | MEDIUM | Already exists as upstream GSD node-repair; needs enhancement for parallel context |
| **Parallelizability score in plan metadata** | Each plan carries a computed score (ratio of wave-1 eligible tasks to total) so users can see whether their plan structure is actually going to benefit from parallelism before execution starts. | LOW | PLAN-04; a plan with 0.3 parallelizability score should trigger a warning before wave execution |
| **Multi-provider model routing** | Opus for planning/verification (high reasoning), Sonnet for execution (throughput), Haiku/Gemini for research (cost). Each session class uses the appropriate cost/quality trade-off. Targeting 40-60% cost reduction vs all-Opus baseline. | MEDIUM | API-04; requires per-agent-type model config; multi-provider adds capability variance risk |

---

### Anti-Features (Deliberately Not Building)

These represent over-engineering traps that look valuable but create more problems than they solve in the context of GBSD's architecture.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Real-time inter-agent communication** | Agents could share discoveries mid-execution (e.g., "I found the auth module is here") | Claude Code's Agent Teams does not support peer-to-peer messaging; simulating it requires a shared message bus (file or network) that introduces race conditions, adds coordination overhead, and makes agents dependent on each other — destroying the isolation guarantee that makes parallel execution safe | Frontloaded context: put shared knowledge in `files_to_read` before execution; post-execution state reconciliation for knowledge discovered during execution |
| **Task-level parallelism within a plan** | More granular parallelism = more speedup | Within a plan, tasks are typically sequential by design (task 2 uses the output of task 1). Forcing task-level parallelism requires complex dependency graph analysis at the task level, a much harder problem than plan-level. The complexity/benefit ratio is poor. | Plan-level parallelism first; task-level deferred to after planner redesign when file ownership analysis makes intra-plan dependencies explicit |
| **Automatic phase parallelism detection** | Eliminate the need for explicit `--parallel-phases` flag; auto-parallelize anything that looks independent | False positives (incorrectly identifying phases as independent) cause schema conflicts, API contract mismatches, and cross-phase integration failures that surface at merge time with no clear attribution. The cost of a false positive is high. | Explicit opt-in annotation: `Parallel OK with: [Phase N]` in ROADMAP.md; user controls when phase parallelism is worth the merge surface |
| **Custom UI / dashboard** | Visual progress tracking across parallel agents is easier in a GUI | GBSD targets developers who live in the terminal. A UI introduces a web server dependency, requires distribution, and becomes the main maintenance burden. The value is marginal for the target user. | `/gsd:progress` command with ASCII progress table; text output is grep-able, scriptable, and works over SSH |
| **Distributed transaction semantics for merges** | Guarantee all-or-nothing wave merges; rollback if any plan fails | Implementing distributed transactions over git branches requires a transaction coordinator, two-phase commit, and compensating transactions for partial merges — enterprise infrastructure complexity. The simpler model (merge what succeeded, retry/skip what failed) is sufficient and keeps the system understandable. | Merge completed plans, offer retry/skip for failed ones; keep branches until user confirms resolution |
| **Shared database across worktrees** | Agents working on related features need to share data state | Race conditions, port conflicts, and dependency on specific local services are the #1 failure mode for multi-agent parallel execution (confirmed by Upsun research). Shared services destroy isolation. | Filesystem-only isolation: agents share git history but not running services; integration tests run at merge time on the unified codebase |
| **Long-term agent memory / vector store** | Agents could learn from past executions and accumulate project knowledge | GSD's file-based context (Pathfinder index, SUMMARY.md files, STATE.md) already provides persistent project knowledge without a separate memory infrastructure. A vector store adds operational complexity for marginal benefit in a per-project tool. | Pathfinder index as the code intelligence layer; SUMMARY.md as execution history; STATE.md as decision history |
| **Automatic upstream GSD sync** | Stay current with GSD improvements without manual effort | GBSD has intentional structural divergences from upstream. Automatic merges will create silent conflicts in agent prompts, workflow ordering, and command definitions that are hard to detect and debug. | Documented process for reviewing upstream commits and cherry-picking selectively; rebrand conflicts are expected and acceptable |

---

## Feature Dependencies

```
[File Ownership Planning] (PLAN-01/02)
    └──enables──> [Wave-Level Parallel Execution] (PARA-01/02)
                      └──requires──> [Merge-Back Protocol] (PARA-06)
                      └──requires──> [State Reconciliation] (PARA-05)

[Pathfinder Integration] (PATH-01)
    └──enables──> [File Ownership Planning] (PLAN-01/02/03)
    └──enables──> [Hub-and-Spoke Agent Teams] (TEAM-01)
    └──enables──> [Parallelizability Score] (PLAN-04)

[Wave-Level Parallel Execution] (PARA-01/02)
    └──requires──> [API Key Pool] (API-01/02/03) for rate headroom
    └──enables──> [Phase-Level Parallelism] (PARA-03)

[Frontloaded Decision Capture] (AUTON-01/02)
    └──enables──> [Session Autonomy / skip_checkpoints] (AUTON-03)
    └──requires──> [Decision Surface Sizing] (AUTON-04)

[Checkpoint-to-File Protocol] (TEAM-02)
    └──enables──> [Interruption Detection / Resume] (TEAM-04)

[API Key Pool] (API-01/02/03)
    └──enables──> [Multi-Provider Model Routing] (API-04)

[Phase-Level Parallelism] (PARA-03)
    └──requires──> [Wave-Level Parallel Execution] (PARA-01/02)
    └──requires──> [API Key Pool] (API-01/02) for concurrent phase budgets
```

### Dependency Notes

- **File Ownership requires Pathfinder:** Without the import graph and call graph, the planner must manually assign file ownership. This works but produces suboptimal results — Pathfinder enables automatic ownership derivation from actual dependency data, which is what drives the 70-85% wave-1 parallelizability target.

- **Wave Execution requires Merge-Back:** Parallel agents producing N branches only have value after merge-back; these features ship together as a unit, not independently.

- **Frontloaded Decisions requires Discussion-Phase Enhancement:** The discuss phase must ask more questions upfront. Without this, the "skip_checkpoints" flag just causes failures when agents hit unresolved decision points.

- **Phase Parallelism conflicts with Sequential Migration Strategy:** Parallel phases cannot safely share database migration files. Phase-level parallelism requires upfront schema definition in Phase 1 before any parallel phases begin. This is not an implementation dependency but an execution constraint.

- **API Key Pool is a prerequisite for more than 3 concurrent agents:** At the default Anthropic rate limits, 3+ concurrent executor sessions will starve each other on a single key. API-01/02/03 must land before setting `max_concurrent_agents > 3`.

---

## MVP Definition

The project already has a well-defined implementation stack (PROJECT.md active requirements). This MVP definition maps feature categories to that stack.

### Launch With — Phase 1 and 2 of Implementation Stack

The minimum that delivers the core value proposition (autonomous parallel execution):

- [ ] **Worktree isolation per executor** (PARA-01, PARA-04) — the isolation primitive everything else rests on; without this, parallel execution is just concurrent writes to the same files
- [ ] **Wave-level parallel execution** (PARA-02) — the primary speedup mechanism; this is what "parallel" means to the user
- [ ] **Merge-back with auto-resolution for lock files** (PARA-06) — without this, every parallel wave requires manual conflict resolution, negating the autonomy benefit
- [ ] **State reconciliation for STATE.md and agent-history.json** (PARA-05) — planning state must survive the merge; without reconciliation the system has undefined state after each wave
- [ ] **Interruption detection and resume** (TEAM-04) — parallel execution increases failure surface; resume must work before autonomous execution is reliable
- [ ] **File ownership in plan frontmatter** (PLAN-01, PLAN-02) — the conflict prevention mechanism; without this, agents will write to the same files and create unresolvable merge conflicts
- [ ] **Frontloaded decision capture in discuss-phase** (AUTON-01, AUTON-02) — parallel agents cannot pause for human questions; this must land alongside PARA-02

### Add After Validation — Phase 3 and 4

Add when core parallel execution is working and validated:

- [ ] **Pathfinder integration for all agents** (PATH-01/02/03/04) — enables automatic file ownership derivation; adds the 70-85% parallelizability target; add when manual file ownership proves insufficient
- [ ] **API key pool with circuit breaker** (API-01/02/03) — add when users report rate limit saturation from parallel execution; not needed for 2-3 concurrent agents on standard API plans
- [ ] **Decision surface sizing / remove context cap** (AUTON-04) — add after frontloaded decisions validate that plans can be larger without quality degradation
- [ ] **Multi-provider model routing** (API-04) — add when cost becomes a concern; Opus-only baseline is simpler and more predictable

### Future Consideration — Phase 5 and 6

Defer until core is proven and adopted:

- [ ] **Phase-level parallelism** (PARA-03) — high complexity, high merge risk; only valuable for projects with truly independent phases; add when wave-level parallelism is mature and users explicitly request it
- [ ] **Hub-and-spoke agent teams** (TEAM-01/02/03) — requires Pathfinder integration and significantly changes agent architecture; complexity is high relative to the wave-level execution wins
- [ ] **LiteLLM cost gateway** (API-05) — operational overhead; add when cost tracking becomes important for multi-team or enterprise usage

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Worktree isolation per executor | HIGH | MEDIUM | P1 |
| Wave-level parallel execution | HIGH | MEDIUM | P1 |
| File ownership in plan frontmatter | HIGH | HIGH | P1 |
| Frontloaded decision capture | HIGH | MEDIUM | P1 |
| Merge-back with lock file auto-resolution | HIGH | MEDIUM | P1 |
| State reconciliation (STATE.md, agent-history) | HIGH | MEDIUM | P1 |
| Interruption detection / checkpoint resume | HIGH | MEDIUM | P1 |
| Pathfinder integration (all agents) | HIGH | MEDIUM | P2 |
| API key pool + circuit breaker | MEDIUM | LOW | P2 |
| Decision surface sizing | MEDIUM | MEDIUM | P2 |
| Parallelizability score in plan metadata | MEDIUM | LOW | P2 |
| Multi-provider model routing | MEDIUM | MEDIUM | P2 |
| Phase-level parallelism | HIGH | HIGH | P3 |
| Hub-and-spoke agent teams | MEDIUM | HIGH | P3 |
| LiteLLM cost tracking gateway | LOW | MEDIUM | P3 |

**Priority key:**
- P1: Must have — delivers the core parallel autonomy value
- P2: Should have — adds efficiency; add after P1 validates
- P3: Nice to have — high complexity relative to marginal value at this stage

---

## Competitor Feature Analysis

Comparison against emerging tools in the parallel AI coding agent space as of early 2026.

| Feature | Superset IDE | Dagger container-use | Manual git worktrees | GBSD approach |
|---------|--------------|---------------------|---------------------|---------------|
| **Isolation primitive** | Git worktrees | Docker containers + git worktrees | Git worktrees (manual) | Git worktrees (orchestrator-managed) |
| **Orchestration layer** | None (user-driven) | None (agent-driven) | None (fully manual) | Structured waves with dependency analysis |
| **Merge conflict prevention** | None (Clash tool detects after-the-fact) | Container isolation prevents conflicts at runtime, not at planning | None | File ownership in plan frontmatter prevents conflicts before execution |
| **Auto merge-back** | Manual | Via git in container | Manual | Automated per wave with typed conflict resolution |
| **State reconciliation** | None | None | None | STATE.md + agent-history.json merge algorithm |
| **Checkpoint / resume** | None documented | None documented | Manual | File-based checkpoint protocol with resume workflow |
| **Decision frontloading** | None | None | None | Discuss-phase captures all decisions before execution |
| **Rate limit management** | None | None | None | API key pool with circuit breaker and round-robin distribution |
| **Pathfinder-driven planning** | None | None | None | Import graph + call graph drives automatic task decomposition |
| **Phase-level parallelism** | None | None | None | Opt-in with dependency analysis |
| **Spec-driven workflow** | None (pure execution) | None | None | Full question → discuss → research → plan → execute → verify pipeline |

**Key differentiator insight:** Every competing tool provides isolation (worktrees or containers) without orchestration. GBSD provides the full orchestration layer — the planner that prevents conflicts, the wave sequencer that respects dependencies, the state reconciler that maintains coherent project state, and the decision frontloader that enables uninterrupted execution. The isolation layer is a commodity; the orchestration layer is the value.

---

## Sources

- [GBSD PROJECT.md](/home/wes/furientis/dev/tools/gbsd/.planning/PROJECT.md) — Active requirements (PARA-01 through TEAM-04, API-01 through API-05, AUTON-01 through AUTON-04, PLAN-01 through PLAN-05, PATH-01 through PATH-04)
- [GBSD Parallel Orchestration Architecture](/home/wes/furientis/dev/tools/gbsd/docs/GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md) — Detailed design for worktree lifecycle, state reconciliation, merge-back, checkpoint handling
- [GBSD Improvement Spec](/home/wes/furientis/dev/tools/gbsd/docs/IMPROVEMENT_SPEC.md) — Four-layer improvement stack with complexity and risk assessments
- [Phase Parallel Analysis](/home/wes/furientis/dev/tools/gbsd/docs/phase-parallel-analysis.md) — Phase-level dependency detection algorithm, risk analysis, merge strategy
- [Git Worktrees for Parallel AI Coding Agents — Upsun](https://devcenter.upsun.com/posts/git-worktrees-for-parallel-ai-coding-agents/) — Ecosystem validation: challenges (port conflicts, missing node_modules, merge conflict blindspots), solutions, emerging tools
- [Git Worktrees Changed My AI Agent Workflow — Nx Blog](https://nx.dev/blog/git-worktrees-ai-agents) — Developer workflow patterns for AI agent worktree usage
- [Clash — conflict detection for parallel agents](https://github.com/clash-sh/clash) — Ecosystem tool that detects cross-worktree conflicts; confirms the gap GBSD's file ownership model fills
- [Container Use by Dagger — InfoQ](https://www.infoq.com/news/2025/08/container-use/) — Container-level isolation approach; confirms isolation-without-orchestration is the ecosystem baseline
- [How to Checkpoint Code Projects with AI Agents — HAMY](https://hamy.xyz/blog/2025-07_ai-checkpointing) — Code checkpoints (git commits) + project checkpoints (RFC, task description, task tracking files) as standard pattern
- [Claude Code Agent Teams documentation](https://code.claude.com/docs/en/agent-teams) — Hub-and-spoke vs peer model; agent team autonomy patterns; known worktree isolation bug (#33045)
- [Retries, fallbacks, and circuit breakers in LLM apps — Portkey](https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/) — Circuit breaker patterns for LLM API key pool management
- [Rate Limiting and Backpressure for LLM APIs](https://dasroot.net/posts/2026/02/rate-limiting-backpressure-llm-apis/) — API key pool round-robin and weighted load balancing patterns

---
*Feature research for: AI agent orchestration — parallel autonomous execution*
*Researched: 2026-03-15*
