---
name: GBSD Improvement Plan
description: Planned improvements to GBSD — Pathfinder integration, worktree parallelism, API augmentation, planner redesign for large codebase efficiency
type: project
---

GBSD (Get Big Shit Done) is evolving from sequential agent orchestration to parallel, knowledge-indexed execution for large codebases.

**Why:** Large codebases (100K+ LOC) suffer from exploration tax (20-40% of agent context) and sequential bottlenecks. GBSD + Pathfinder + worktree parallelism addresses both.

**How to apply:** When working on GBSD or Pathfinder, reference these planned improvements. Implementation follows the layer stack: Pathfinder integration → planner redesign → wave parallelism → API augmentation → phase parallelism.

## Key Findings (2026-03-15 research session)

### Architecture: 4-Layer Stack
1. **Pathfinder Integration** (Layer 1) — all agents read .code-intel/ index instead of re-exploring
2. **Parallelism-Aware Planner** (Layer 2) — file ownership, module-boundary decomposition, 70-85% parallelizability
3. **Worktree Parallel Execution** (Layer 3) — independent `claude -p` sessions per plan, wave and phase level
4. **API Rate Augmentation** (Layer 4) — multi-org key pools, multi-provider routing

### Critical Discovery: Agent Teams Worktree Bug
- `isolation: "worktree"` is silently ignored for Agent Teams teammates (GitHub issue #33045)
- Workaround: spawn independent `claude` CLI processes via Bash, each in a manual `git worktree`
- This is actually cleaner for GBSD since executors are independent by design

### Parallelism Model
- **Wave-level**: Plans in same wave → separate worktrees, merge back per wave
- **Phase-level**: Independent phases → separate worktrees running full pipelines, merge back per phase
- Both composable (parallel phases with parallel waves inside)
- Branch naming: `gsd/exec/{phase}-{plan}-w{wave}`

### Research Artifacts
- GBSD fork analysis: ~/.claude/GBSD_FORK_ANALYSIS.md
- Orchestration architecture: ~/.claude/GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md
- Merge-back protocol: ~/.claude/get-shit-done/workflows/merge-back-protocol.md
- Pathfinder integration: /home/wes/furientis/dev/tools/pathfinder/.planning/GBSD_INTEGRATION_ANALYSIS.md
- API augmentation: /home/wes/api-rate-augmentation-analysis.md
- Planner redesign: ~/.claude/projects/-home-wes/planner-redesign-analysis.md
- Planner+Pathfinder synthesis: ~/.claude/projects/-home-wes/planner-pathfinder-synthesis.md
- Phase parallelism: ~/.claude/projects/-home-wes/memory/phase-parallel-analysis.md
- Comms design: ~/.claude/design-teams-comms.md
