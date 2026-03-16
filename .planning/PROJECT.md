# GBSD Supercharge

## What This Is

GBSD is a fork of GSD (Get Shit Done) — a meta-prompting, context engineering, and spec-driven development system for Claude Code. This project supercharges GBSD into a parallel, knowledge-indexed, autonomy-first execution engine capable of handling 100k+ LOC codebases efficiently. The target user is a solo developer or small team who wants reliable, fast, autonomous AI-driven development at scale.

## Core Value

Phases execute autonomously end-to-end — hit execute and walk away. The system frontloads all decisions, uses code intelligence to eliminate wasted context, runs agents in parallel across isolated worktrees, and delivers 5-8x faster project completion with 40-60% cost reduction.

## Requirements

### Validated

- ✓ Phase-based development workflow (question → discuss → research → plan → execute → verify) — existing
- ✓ 12 specialized agents (planner, executor, researcher, verifier, debugger, etc.) — existing
- ✓ Atomic git commits per task — existing
- ✓ Wave-based parallel plan execution — existing
- ✓ Node repair operator for autonomous task recovery — existing (upstream GSD)
- ✓ Mandatory read_first and acceptance_criteria on tasks — existing (upstream GSD)
- ✓ Pathfinder navigation in 4 workflows (mapper, researcher, planner, executor) — existing
- ✓ Full rebrand from GSD to GBSD — existing
- ✓ Migration detection for existing GSD installations — existing
- ✓ Multi-runtime support (Claude Code, OpenCode, Gemini CLI, Codex) — existing

### Active

- [ ] **AUTON-01**: Frontloaded decision capture — all human decisions gathered during discuss/plan phases, not during execution
- [ ] **AUTON-02**: Checkpoint classification — auto-resolvable vs must-ask-human, with auto-resolve as default
- [ ] **AUTON-03**: Extended autonomous execution sessions leveraging 1M token context windows
- [ ] **AUTON-04**: Decision surface sizing — plans sized by decision count, not token budget
- [ ] **PATH-01**: Deep Pathfinder integration — planner uses blast-radius for impact analysis
- [ ] **PATH-02**: Dependency-graph-driven task decomposition via Pathfinder call graphs
- [ ] **PATH-03**: Module coupling analysis for automatic task clustering
- [ ] **PATH-04**: Pathfinder index freshness management (auto-refresh when stale)
- [ ] **PLAN-01**: File ownership as first-class planning signal — identify file sets before task decomposition
- [ ] **PLAN-02**: Distinguish hard vs soft dependencies (read-only vs read-write conflicts)
- [ ] **PLAN-03**: Auto-derive dependencies from imports/API usage via Pathfinder
- [ ] **PLAN-04**: Target 70-85% wave-1 parallelizability (up from 40-60%)
- [ ] **PLAN-05**: Granular plans — 5-10 plans × 1-2 tasks each (vs 3-5 × 2-3)
- [ ] **PARA-01**: Git worktree isolation per executor — each parallel plan gets its own worktree
- [ ] **PARA-02**: Wave-level parallel execution with merge-back after each wave
- [ ] **PARA-03**: Phase-level parallelism for independent phases (opt-in)
- [ ] **PARA-04**: Worktree manager utility for creating/cleaning/merging worktrees
- [ ] **PARA-05**: State reconciliation algorithm for STATE.md and agent-history.json after merges
- [ ] **PARA-06**: Merge conflict auto-resolution for lock files, config, barrel files (80%+ auto)
- [ ] **TEAM-01**: Hub-and-spoke agent teams — lead stays lean (~15% context), delegates to specialists
- [ ] **TEAM-02**: File-based checkpoint protocol for state persistence across agent boundaries
- [ ] **TEAM-03**: Task dependency chains (BlockedBy) for wave synchronization
- [ ] **TEAM-04**: Executor checkpoint/resume — interrupted executors can be continued or retried
- [ ] **API-01**: Multi-org API key pool — 2-3 separate Anthropic orgs for independent rate limits
- [ ] **API-02**: Round-robin key distribution across parallel executor sessions
- [ ] **API-03**: Circuit breaker failover — detect 429s, blacklist keys temporarily, exponential backoff
- [ ] **API-04**: Multi-provider routing — Opus for planning, Sonnet for execution, Haiku for research
- [ ] **API-05**: Cost tracking via unified gateway (LiteLLM or equivalent)
- [ ] **SCALE-01**: Efficient operation on 100k+ LOC codebases via Pathfinder index (not exploration)
- [ ] **SCALE-02**: Context budget optimization — <5% exploration overhead (down from 20-40%)
- [ ] **ADOPT-01**: Track and adopt Claude Code Agent Teams API changes rapidly
- [ ] **ADOPT-02**: Leverage new Claude Code features (worktree support, new tools) as they land
- [ ] **ADOPT-03**: Optional upstream GSD sync — ability to cherry-pick upstream improvements

### Out of Scope

- Rewriting GSD's core workflow order of operations — the question → research → plan → execute → verify pipeline stays
- Building a custom UI or dashboard — GBSD remains CLI-first
- Supporting non-AI runtimes — GBSD targets AI coding assistants only
- Real-time inter-agent communication — hub-and-spoke via task lists, not peer-to-peer messaging

## Context

GBSD is forked from glittercowboy/get-shit-done at v1.22.4 (commit 0f38e34). The fork has 2 additional commits: full rebrand (661dbb2) and Pathfinder workflow integration (c5453bc). The codebase is ~5,400 LOC Node.js (CommonJS), 12 agent definitions in markdown, 32 commands, and 35+ workflow files. Zero runtime dependencies beyond Node.js builtins.

The companion Pathfinder project provides code intelligence indexing (.code-intel/) with module maps, dependency graphs, call graphs, and blast-radius analysis. Pathfinder is a separate tool that GBSD can leverage when available.

Claude Code's Agent Teams feature is actively evolving. Key known limitation: teammates silently ignore `isolation: "worktree"` (GitHub issue #33045). Workaround: lead-managed worktrees passed as paths. This may be fixed upstream, so the architecture should be adaptable.

The shift from 200k to 1M token context windows (Opus 4.6) fundamentally changes session sizing assumptions. Plans should no longer be constrained by token budget — they should be sized by decision surface (number of points requiring human input).

## Constraints

- **Codebase size**: GBSD itself should remain small and lean — complexity in architecture, not LOC
- **Upstream optionality**: Changes should be structured so upstream GSD updates can optionally be merged (rebrand conflicts are acceptable, structural conflicts are not)
- **Claude Code dependency**: Must work with current Claude Code capabilities; design for workarounds where features are missing (e.g., worktree isolation)
- **Pathfinder coupling**: Deep integration but graceful degradation — everything must work (slower) without .code-intel/

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Keep GSD workflow order | User values the question → plan → execute pipeline; improvements are about parallelism and autonomy, not process redesign | — Pending |
| Hub-and-spoke, not peer-to-peer | Agent Teams doesn't support peer communication; lead-managed coordination is simpler and proven | — Pending |
| Opt-in phase-level parallelism | Risk of schema/API conflicts too high for auto-detection; wave-level is safe by default | — Pending |
| Multi-org over multi-key | Anthropic rate limits are org-level; multiple keys in one org share limits | — Pending |
| Pathfinder required for 100k+ LOC | Exploration tax at scale makes large codebases impractical without indexing | — Pending |

---
*Last updated: 2026-03-15 after initialization*
