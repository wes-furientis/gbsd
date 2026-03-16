# GBSD Improvement Specification

**Date:** 2026-03-15
**Status:** Research complete, ready for implementation planning
**Context:** Synthesized from 6 parallel research agents analyzing wave parallelism, phase parallelism, API augmentation, planner redesign, Pathfinder integration, and orchestration architecture.

---

## Vision

GBSD evolves from a sequential agent orchestrator into a **parallel, knowledge-indexed execution engine** for large codebases. Three pillars: **Pathfinder intelligence**, **worktree parallelism**, and **rate-augmented throughput**.

---

## Architecture: The Improvement Stack (Bottom-Up)

```
Layer 4: API Rate Augmentation
           ↑ enables more concurrent sessions
Layer 3: Parallel Execution (waves + phases in worktrees)
           ↑ enabled by independence analysis
Layer 2: Parallelism-Aware Planner (file ownership, wave optimization)
           ↑ informed by code intelligence
Layer 1: Pathfinder Integration (index → all agents)
           ↑ foundation for everything above
Layer 0: GBSD Core (existing GSD fork + node-repair, resume, diagnostics)
```

Each layer enables the one above it. Implementation order follows the stack.

---

## Layer 1: Pathfinder Integration

**What**: Every GBSD agent reads `.code-intel/` index instead of re-exploring the codebase.

| Agent | Current | With Pathfinder |
|---|---|---|
| **Mapper** | 4 parallel agents write 7 prose docs | Ensure index exists (3-5 sec), agents read it directly |
| **Researcher** | Explores codebase to understand scope | Queries blast-radius + dependency graph before exploring |
| **Planner** | Guesses task dependencies | Uses import graph for automatic wave assignment via topological sort |
| **Executor** | Globs/greps to find files (20-40% context tax) | Navigates via L0/L1/L2 index (<1% context) |
| **Verifier** | Checks files exist, tests pass | Call-graph-based coverage — verifies callers aren't broken |

**New capabilities unlocked:**
- **Merge conflict prediction**: Dependency graph identifies which modules share imports — flag for sequential execution
- **Module-boundary task decomposition**: Natural task boundaries from coupling analysis, not arbitrary splitting
- **Parallelizability score**: Ratio of independent modules to total — measurable before execution

**Impact**: 25% context savings per agent, 3-5x more parallelizable tasks, 40-60% faster phase planning.

**Complexity**: Medium
**Risk**: Low — Pathfinder v1 is complete and self-dogfooding; agent modifications are additive (check if index exists, use it if so)

---

## Layer 2: Parallelism-Aware Planner

**What**: Redesign gsd-planner to optimize for maximum independent work units.

**Key changes:**
1. **File ownership as first-class concept** — each plan declares owned files; two plans can't own the same file in the same wave
2. **Dependency-driven decomposition** — instead of "decompose then check dependencies," start from the dependency graph and decompose along module boundaries
3. **Distinguish hard vs soft dependencies** — hard: must complete first; soft: can stub and proceed
4. **More plans per phase, fewer tasks per plan** — shift from 3-5 plans to 5-10 plans, each tightly scoped to a module boundary
5. **Parallelizability score** — target 70-85% wave-1 eligibility (up from current 40-60%)

**New frontmatter fields:**
```yaml
files_owned: [src/auth/*, src/middleware/auth.ts]  # exclusive ownership
files_read: [src/types/user.ts]                     # read-only, no conflict
dependency_type: hard | soft                         # soft = can stub
parallelizability_score: 0.78                        # computed by planner
```

**Complexity**: High
**Risk**: Medium — changes core planning logic; needs validation that more granular plans don't increase coordination overhead beyond parallelism gains

---

## Layer 3: Parallel Execution

**What**: Independent `claude` CLI processes, each in its own git worktree, for wave-level and phase-level parallelism.

### Wave-Level (within a phase)
- Plans in the same wave → each gets a `git worktree` + independent `claude -p` session
- Orchestrator creates worktrees, spawns processes, waits, merges back
- Branch naming: `gsd/exec/{phase}-{plan}-w{wave}`
- Merge strategy: regular merge commits, auto-resolve locks/deps/barrels, escalate source conflicts

### Phase-Level (across phases)
- Independent phases (no shared requirements, no shared files) → separate worktrees running full pipelines
- Dependency analysis from ROADMAP.md: explicit deps, requirement categories, file path overlap scoring
- Opt-in via `--parallel-phases` flag or config
- Higher risk, bigger payoff — reduces project duration from sum to critical-path

### Nested Parallelism
Both composable: parallel phases, each with parallel waves inside. A 5-phase project with 3 plans/phase could run 6-8 concurrent sessions at peak.

### Critical Discovery: Agent Teams Worktree Bug
- `isolation: "worktree"` is silently ignored for Agent Teams teammates (GitHub issue #33045)
- **Workaround**: spawn independent `claude` CLI processes via Bash, each in a manual `git worktree`
- This is actually cleaner for GBSD since executors are independent by design (no inter-agent communication needed)

**Complexity**: Medium (wave), High (phase)
**Risk**: Medium — merge conflicts are the primary concern; mitigated by file ownership in planner + auto-resolution protocol

---

## Layer 4: API Rate Augmentation

**What**: Multiple API keys and providers to support high-concurrency parallel execution.

**Problem**: Anthropic rate limits are org-level. 5 parallel sessions on one org key will starve each other.

**Solutions (progressive):**
1. **Multi-org key pool** — 2-3 Anthropic orgs with separate billing, round-robin session distribution → 2-3x throughput
2. **Multi-provider specialization** — Opus for planning/execution, Sonnet for verification, Haiku/Gemini for research → 40-60% cost reduction
3. **Intelligent routing** — capacity-aware load balancing, circuit breaker failover when one key is rate-limited
4. **Cost tracking** — LiteLLM proxy for unified cost attribution across keys/providers

**Configuration:**
```json
{
  "api_key_pool": [
    { "org": "primary", "key": "sk-ant-...", "models": ["opus", "sonnet"] },
    { "org": "secondary", "key": "sk-ant-...", "models": ["sonnet", "haiku"] }
  ],
  "parallelization": {
    "max_concurrent_agents": 5,
    "key_distribution": "round-robin",
    "session_timeout_minutes": 30
  }
}
```

**Complexity**: Low (multi-key), Medium (multi-provider)
**Risk**: Low — environment variable per process is straightforward; multi-provider adds model capability variance

---

## Cross-Cutting Concern: Session Autonomy

**What**: Longer, uninterrupted executor sessions with fewer execution-time interruptions, enabled by frontloading decisions into planning.

### The Constraint Shift

GSD's current plan/phase sizing is anchored to the old 200k subagent context window:
- 2-3 tasks per plan (~50% context target)
- 15-60 min per task
- Aggressive checkpoint insertion for human-verify gates

These constraints were designed for Claude 3.5 Sonnet's 200k context. **Claude Opus 4.6 and Sonnet 4.6 have 1M token context windows** — a 5x increase. Combined with Pathfinder cutting exploration tax to <5% and worktree isolation giving each executor a fresh window, the sizing constraints are obsolete.

However, context length becomes less of a concern in the new framework for a different reason: **each parallel executor gets its own fresh 1M context.** The bottleneck is no longer "fitting work into one context window" but rather "how many independent contexts can we run simultaneously" (which is an API rate / cost question, not a context question). GBSD should:

1. **Remove hardcoded context-based sizing** — the 50% context target, the 2-3 task cap, and the "split if >5 files" rule all assumed 200k. With 1M, a single executor session can handle an entire phase's worth of tasks if they're sequential.
2. **Size plans by decision surface, not token budget** — how many tasks can run without needing human input, not how many fit in context.
3. **Use context headroom for richer execution** — with 5x more context, executors can load full file contents (not just paths), keep more codebase context active, and produce more thorough SUMMARY.md outputs without worrying about truncation.
4. **Reserve context-awareness for edge cases** — very large codebases (500K+ LOC) where even 1M context could fill up. Pathfinder's index-based navigation is the mitigation here: read the index (5K tokens) instead of the codebase (500K tokens).

### Frontloaded Checkpoints

The total number of decision points stays the same or increases — they move from execution-time interruptions to planning-time decisions. Invest more upfront in thorough planning (discuss → research → plan → verify, with more questions asked), and executors run autonomously because decisions were already made.

**Pros:**
- Executor sessions run uninterrupted — no context-breaking pauses for human input
- Parallel execution becomes viable — can't pause 5 worktree sessions to ask the user a question mid-task
- Better decisions — planning-time decisions have full research context; execution-time decisions are made under pressure with a half-consumed context window
- Batch human attention — one focused planning session, then walk away during execution
- Aligns with worktree parallelism — independent sessions can't coordinate on ad-hoc decisions

**Cons:**
- Planning sessions get longer and more demanding of user attention upfront
- Some decisions genuinely can't be made until code is running (integration issues, performance, API behavior)
- Risk of over-planning — deciding things that turn out to be irrelevant
- If the plan is wrong, longer autonomous execution = more wasted work before correction
- Node-repair partially mitigates but has a budget limit

### Changes Required

1. **Planner**: Remove the 2-3 task / 50% context cap; size plans based on decision surface (how many tasks can run without needing human input), not context budget
2. **Discuss-phase**: More thorough upfront questioning — resolve ambiguities that would otherwise become execution-time checkpoints
3. **Checkpoint classification**: Distinguish "must ask human" from "can auto-resolve with node-repair" — reduce the former, increase the latter
4. **Plan-checker**: Verify that plans are self-contained — flag any task that would require an execution-time decision that wasn't resolved in planning
5. **Auto-advance**: Extend from phase chaining to full milestone autonomy — start milestone, walk away, review results

**Complexity**: Medium (touches planner sizing + checkpoint logic)
**Risk**: Medium — over-planning and wrong-plan-runs-too-long are real risks; node-repair and verification provide guardrails

---

## Implementation Order

| Phase | What | Depends On | Complexity |
|---|---|---|---|
| **1** | Pathfinder index integration (4 agent modifications) | Pathfinder v1 complete (it is) | Medium |
| **2** | Planner redesign (file ownership, soft deps, module-boundary decomposition) | Phase 1 (needs dependency graph) | High |
| **3** | Wave-level worktree parallelism (process spawning, merge-back) | Phase 2 (needs parallelizable plans) | Medium |
| **4** | API key pool + multi-org distribution | Phase 3 (needs parallel sessions to distribute) | Low |
| **5** | Phase-level parallelism (dependency analysis, nested parallelism) | Phases 3+4 (needs wave parallelism + rate headroom) | High |
| **6** | Multi-provider routing (Gemini/OpenAI for research/verification) | Phase 4 (needs key pool infrastructure) | Medium |

---

## What Exists Today vs What Needs Building

| Component | Status | Location |
|---|---|---|
| GSD core workflows | Exists | `~/.claude/get-shit-done/` |
| GBSD fork (node-repair, resume, diagnostics) | Exists | `/home/wes/furientis/dev/tools/gbsd/` |
| Pathfinder v1 (index generation, queries, CLI) | Complete | `/home/wes/furientis/dev/tools/pathfinder/` |
| Merge-back protocol | Designed | `~/.claude/get-shit-done/workflows/merge-back-protocol.md` |
| Communication patterns | Designed | `~/.claude/design-teams-comms.md` |
| Orchestration architecture | Designed | `~/.claude/GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md` |
| Agent modifications for Pathfinder | **Not built** | Pending in GBSD |
| Worktree process management | **Not built** | Needs `gsd-worktree-manager.sh` |
| API key pool + routing | **Not built** | Needs config schema + spawning logic |
| Phase dependency analysis | **Not built** | Needs algorithm in orchestrator |
| Planner parallelism optimization | **Not built** | Needs gsd-planner.md rewrite |

---

## Projected Impact

| Metric | Current | After All Layers |
|---|---|---|
| Exploration tax per agent | 20-40% of context | <5% |
| Parallelizability (wave-1 eligible tasks) | 40-60% | 70-85% |
| Context savings per phase | Baseline | ~33% reduction (~16K tokens) |
| Project duration (5-phase) | Sum of all phases | Critical path only |
| Concurrent executor sessions | 1 | 5-8 |
| Cost per phase (with multi-provider) | Baseline | 40-60% reduction |

---

## Research Artifacts

| File | Content |
|---|---|
| `~/.claude/GBSD_FORK_ANALYSIS.md` | GBSD vs upstream GSD diff |
| `~/.claude/GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md` | Full orchestration architecture |
| `~/.claude/design-teams-comms.md` | Message schemas and task dependency structures |
| `~/.claude/implementation-roadmap-agent-teams.md` | Implementation roadmap (earlier iteration) |
| `~/.claude/get-shit-done/workflows/merge-back-protocol.md` | Merge-back spec with pseudocode |
| `/home/wes/furientis/dev/tools/pathfinder/.planning/GBSD_INTEGRATION_ANALYSIS.md` | Pathfinder integration opportunities |
| `/home/wes/api-rate-augmentation-analysis.md` | API rate augmentation analysis |
| `~/.claude/projects/-home-wes/planner-redesign-analysis.md` | Planner redesign for parallelism |
| `~/.claude/projects/-home-wes/planner-pathfinder-synthesis.md` | Planner + Pathfinder unified strategy |
| `~/.claude/projects/-home-wes/memory/phase-parallel-analysis.md` | Phase-level parallelism design |
