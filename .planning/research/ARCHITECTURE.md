# Architecture Research

**Domain:** Parallel AI Orchestration Framework (GBSD Enhancement)
**Researched:** 2026-03-15
**Confidence:** HIGH — Based on first-party design documents and existing codebase analysis

---

## Standard Architecture

### System Overview

The enhanced GBSD architecture adds four new subsystems on top of the existing layered orchestration model. The existing system is a three-layer stack (Orchestrator → Tools CLI → Agents). The new components slot in as middleware between those layers.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        USER / SLASH COMMANDS                         │
│   /gbsd:execute-phase   /gbsd:plan-phase   /gbsd:merge-back          │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│                    ORCHESTRATOR LAYER (Workflows)                     │
│  execute-phase.md   plan-phase.md   merge-back.md   progress.md      │
│                                                                       │
│   Reads config.json and STATE.md → groups plans into waves →          │
│   delegates to Worktree Manager → spawns Agent Teams                 │
└──────┬───────────────┬──────────────────┬──────────────────┬─────────┘
       │               │                  │                  │
       ▼               ▼                  ▼                  ▼
┌──────────────┐ ┌────────────────┐ ┌───────────────┐ ┌────────────────┐
│  WORKTREE    │ │  KEY POOL      │ │   PATHFINDER  │ │   STATE        │
│  MANAGER    │ │  MANAGER       │ │   ADAPTER     │ │  RECONCILER    │
│             │ │                │ │               │ │                │
│ create/     │ │ round-robin    │ │ index queries │ │ merge STATE.md │
│ destroy/    │ │ key selection  │ │ blast-radius  │ │ merge history  │
│ merge       │ │ circuit-break  │ │ dep-graph     │ │ dedup entries  │
│ worktrees   │ │ failover       │ │ read-only     │ │ after waves    │
└──────┬───────┘ └───────┬────────┘ └──────┬────────┘ └──────┬─────────┘
       │                 │                  │                  │
       └────────┬────────┘                  │                  │
                │                           │                  │
┌───────────────▼───────────────────────────▼──────────────────▼───────┐
│                         TOOLS CLI (gbsd-tools.cjs)                    │
│  state  phase  roadmap  frontmatter  config  verify  template  init   │
│                                                                       │
│  Single Node.js CLI — all state mutations are atomic through here    │
└──────────────────────────────┬────────────────────────────────────────┘
                               │
┌──────────────────────────────▼────────────────────────────────────────┐
│                          AGENT LAYER                                   │
│                                                                        │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐              │
│  │ gbsd-executor │  │ gbsd-planner  │  │ gbsd-verifier │              │
│  │ (in worktree) │  │ (Opus model)  │  │ (Sonnet)      │  + 9 more   │
│  └───────────────┘  └───────────────┘  └───────────────┘              │
└───────────────────────────────────────────────────────────────────────┘
```

The four new subsystems are:

1. **Worktree Manager** — Creates, assigns, and tears down git worktrees; manages branch lifecycle
2. **Key Pool Manager** — Distributes API keys (round-robin + circuit-breaker) across parallel agents
3. **Pathfinder Adapter** — Queries the `.code-intel/` index for dependency and file-ownership data; feeds the planner
4. **State Reconciler** — Merges diverged STATE.md and agent-history.json after parallel waves complete

---

### Component Responsibilities

| Component | Responsibility | Communicates With |
|-----------|----------------|-------------------|
| Orchestrator (execute-phase.md) | Choreograph wave execution; spawn/join agents; surface checkpoints to user | Worktree Manager, Key Pool, Tools CLI, Agent Layer |
| Worktree Manager | git worktree add/remove; branch creation/deletion; stale worktree detection on restart | Orchestrator, State Reconciler |
| Key Pool Manager | Select API key for each spawned agent; track 429 failures; circuit-break exhausted keys | Orchestrator only; each agent receives one key at spawn time |
| Pathfinder Adapter | Load `.code-intel/` index; answer file-ownership and dependency queries; signal when index is stale | Planner (plan-phase.md); read-only |
| State Reconciler | After merge-back: diff multiple STATE.md copies; union Decisions sections; merge agent-history.json | Orchestrator, Tools CLI (writes final merged state) |
| gbsd-executor (in worktree) | Execute one plan in its assigned worktree; commit atomically; write SUMMARY.md; signal completion | Only its own worktree filesystem; no peer contact |
| gbsd-planner | Decompose phase goal into file-ownership-first plans; assign waves; emit PLAN.md with frontmatter | Pathfinder Adapter (optional); Checker agent |
| Tools CLI (gbsd-tools.cjs) | Atomic state mutations; config reads; frontmatter parsing; health checks | Everything — all state changes route through here |

---

## Recommended File Structure for New Components

The existing codebase stores tools logic in `gbsd/bin/lib/*.cjs`. New subsystem logic follows the same pattern.

```
gbsd/
├── bin/
│   └── lib/
│       ├── core.cjs             # existing — shared utils, model profiles
│       ├── state.cjs            # existing — STATE.md read/write
│       ├── phase.cjs            # existing — phase CRUD
│       ├── config.cjs           # existing — config.json operations
│       ├── worktree.cjs         # NEW — worktree create/remove/list/detect-stale
│       ├── keypool.cjs          # NEW — key selection, circuit-breaker state
│       ├── reconcile.cjs        # NEW — STATE.md + agent-history merge logic
│       └── pathfinder.cjs       # NEW — .code-intel/ query adapter
├── workflows/
│   ├── execute-phase.md         # existing — update to use worktree + keypool
│   ├── plan-phase.md            # existing — update to call pathfinder adapter
│   ├── merge-back.md            # NEW — manual worktree merge command
│   └── progress.md              # existing — update for multi-agent visibility
└── agents/
    └── gbsd-planner.md          # existing — update decomposition algorithm
```

---

## Architectural Patterns

### Pattern 1: Hub-and-Spoke Orchestration with File-Based State

**What:** The orchestrator (lead agent) is the single coordination point. Parallel executor agents communicate only by writing files and completing tasks — never by messaging each other directly. The orchestrator reads results after agents complete rather than subscribing to in-flight updates.

**When to use:** The only viable pattern given Claude Code Agent Teams limitations: worktree isolation is not natively enforced (GitHub issue #33045), and peer-to-peer messaging between teammates is not supported.

**Trade-offs:** Lead context stays lean (~15% usage). No real-time visibility into executor progress. Checkpoints pause the lead until user responds; other executors continue independently.

**Canonical flow:**
```
Lead: git worktree add .claude/worktrees/exec-01 -b gsd/exec/01-01
Lead: Task(subagent_type="gbsd-executor", prompt="Execute plan 01-01 in worktree...")
  → Executor runs independently in its worktree, commits, writes SUMMARY.md
Lead: [waits for Task completion via TaskList polling]
Lead: git merge --no-ff gsd/exec/01-01
Lead: git worktree remove .claude/worktrees/exec-01
Lead: State Reconciler runs → merges STATE.md diffs
```

### Pattern 2: Ephemeral Worktrees as Execution Environments

**What:** Each parallel plan execution receives one dedicated git worktree — a full-copy working directory on a separate branch. Worktrees are created immediately before spawning the executor and destroyed immediately after successful merge-back. Branches persist only long enough for the merge.

**When to use:** Any plan-level parallel execution (PARA-01, PARA-02). Also required for any phase-level parallelism (PARA-03).

**Trade-offs:** Disk usage scales with worktree count (estimated 150–700 MB each). Three concurrent executors require approximately 1.5 GB free beyond the base project. Worktrees survive process interruptions — the orchestrator must detect stale worktrees on startup and offer resume/retry/skip.

**Naming convention:**
```
Worktree path:  .claude/worktrees/gsd-exec-{phase}-{plan-id}-w{wave}
Branch name:    gsd/exec/{phase}-{plan-id}-w{wave}

Example:
  .claude/worktrees/gsd-exec-03-01-01-w1
  branch: gsd/exec/03-01-01-w1
```

### Pattern 3: File-Ownership-First Plan Decomposition

**What:** The planner identifies which files each task will create or modify before decomposing into plans. A file may only have one writer per wave. Tasks that need the same file get serialized into different waves. Tasks that touch disjoint file sets can run in wave 1 in parallel.

**When to use:** Every planning invocation once PLAN-01 through PLAN-03 are implemented.

**Trade-offs:** Produces more granular plans (5–10 vs the current 3–5). Each plan is smaller and faster, but there is more orchestration overhead. With worktree parallelism, wall-clock time improves despite the overhead.

**Impact on frontmatter:** `files_modified` stays as is. The planner's reasoning process changes — it identifies the file set first, clusters by owner, then names plans. Optionally adds `files_created` / `files_read` to frontmatter for machine-readable ownership.

### Pattern 4: Append-Only State Reconciliation

**What:** After all executors in a wave complete and their branches are merged, the State Reconciler performs three operations: (1) union of `## Decisions` entries across all worktree STATE.md copies; (2) union of `agent-history.json` entries by `agent_id` (no duplicates); (3) update of `## Session Continuity` fields to reflect the most recent completed action.

**When to use:** After every wave merge-back. Required before the next wave can start.

**Trade-offs:** STATE.md can only grow (append-only decisions). No compaction. This is acceptable because GBSD plans are relatively short-lived. The reconciler is a small deterministic script — not an LLM operation.

---

## Data Flow

### Plan-Phase Flow (with Pathfinder)

```
User: /gbsd:plan-phase 3
  │
  ▼
Orchestrator loads STATE.md, ROADMAP.md, config.json (Tools CLI: init)
  │
  ▼
Pathfinder Adapter (if .code-intel/ present):
  - Queries call graph for phase-relevant files
  - Returns: file dependency map, existing exports, stale-index flag
  │
  ▼
gbsd-phase-researcher spawned → writes .planning/phases/03-*/RESEARCH.md
  │
  ▼
gbsd-planner spawned with RESEARCH.md + Pathfinder data:
  1. Identifies all files to create/modify
  2. Clusters tasks by file ownership
  3. Detects hard vs soft dependencies (Pathfinder-aided)
  4. Assigns wave numbers from file ownership graph
  5. Produces granular PLAN.md files (5-10 plans × 1-2 tasks)
  │
  ▼
gbsd-plan-checker validates (Nyquist validation + wave consistency)
  │
  [issues found?]
  ├─ Yes → revision loop (max 3 iterations) → back to planner
  └─ No → commit PLAN.md files
```

### Execute-Phase Flow (with Parallel Worktrees)

```
User: /gbsd:execute-phase 3
  │
  ▼
Orchestrator: init → loads plan index → groups plans into waves
  │
  ▼
For each wave:
  │
  ├─ Key Pool Manager: assign API key per executor (round-robin)
  │
  ├─ Worktree Manager: for each plan in wave
  │     git worktree add .claude/worktrees/gsd-exec-03-{N}-w{W}
  │                      -b gsd/exec/03-{N}-w{W}
  │
  ├─ Spawn N executor agents in parallel (Task() calls)
  │     Each executor: reads plan → executes tasks → commits → writes SUMMARY.md
  │     Each executor: isolated; knows nothing about siblings
  │
  ├─ [Lead polls TaskList until all N tasks complete]
  │     [If checkpoint: present to user, spawn continuation, other executors continue]
  │
  ├─ Merge-back (per completed executor):
  │     git merge --no-ff gsd/exec/03-{N}-w{W}
  │     npm test (or project test command)
  │     git worktree remove .claude/worktrees/gsd-exec-03-{N}-w{W}
  │     git branch -d gsd/exec/03-{N}-w{W}
  │
  ├─ State Reconciler:
  │     Collect STATE.md copies from merged branches
  │     Union Decisions entries
  │     Merge agent-history.json (union by agent_id)
  │     Write reconciled STATE.md via Tools CLI
  │
  └─ Continue to next wave (or verify-phase)
```

### Pathfinder Data Flow into Planner

```
.code-intel/                       Pathfinder Adapter
├── module-map.json         ───►   query: "files relevant to phase goal"
├── dependency-graph.json   ───►   query: "what imports X?"
├── call-graph.json         ───►   query: "blast radius of file Y"
└── index-metadata.json     ───►   query: "is index fresh?" (stale if >24h or recent git changes)
         │
         ▼
Adapter returns to Planner:
  {
    "file_dependency_map": { "src/models/user.ts": ["src/api/users.ts", ...] },
    "existing_exports": { "src/models/user.ts": ["User", "CreateUserDto"] },
    "stale": false
  }
         │
         ▼
Planner uses this to:
  1. Detect which new files will depend on which existing files
  2. Assign hard vs soft dependency classification
  3. Auto-populate depends_on in plan frontmatter
  4. Avoid declaring "files_modified" conflicts on read-only imports
```

### API Key Flow

```
config.json: api_management.organizations[{org_id, api_key, role}]
         │
         ▼
Key Pool Manager (keypool.cjs):
  - Maintains array of {key, org_id, failures, blacklisted_until}
  - assign_key(executor_index) → key = pool[executor_index % pool.length]
  - on 429: increment failures; if failures >= 3 → blacklist for 300s
  - select_available_key() → skips blacklisted keys
         │
         ▼
Orchestrator passes key to Task() spawn:
  Task(
    subagent_type="gbsd-executor",
    env={"ANTHROPIC_API_KEY": selected_key},
    prompt="..."
  )
```

---

## Component Boundaries

### Worktree Manager (worktree.cjs)

**Owns:** All `git worktree` shell interactions. Nothing else touches worktree lifecycle.

**Interface:**
```
create(phase, planId, wave) → { path, branch }
remove(path)
list() → [{ path, branch, status }]
detectStale(phase) → [{ path, branch }]  // worktrees with no in-progress Task
```

**Does NOT own:** Merging branches (orchestrator does that), deciding which plans run in parallel (orchestrator does that).

### Key Pool Manager (keypool.cjs)

**Owns:** API key selection, failure counting, circuit-breaker state (in-memory only — not persisted to disk).

**Interface:**
```
assignKey(executorIndex) → string  // round-robin
reportFailure(key)                 // increments failure counter
reportSuccess(key)                 // resets failure counter
getAvailableKey() → string | null  // returns null if all keys blacklisted
```

**Does NOT own:** Actually calling the API (agents do that), storing keys on disk (config.json owns storage).

### Pathfinder Adapter (pathfinder.cjs)

**Owns:** Reading `.code-intel/` index files; answering dependency queries; detecting index staleness.

**Interface:**
```
isAvailable() → boolean            // .code-intel/ exists and index-metadata.json present
isStale() → boolean               // index older than 24h or behind git HEAD
getFileDeps(files[]) → Map         // dependency graph for a set of files
getExports(file) → string[]        // exported symbols from a file
getBlastRadius(file) → string[]    // files that import the given file
```

**Does NOT own:** Running Pathfinder (user runs that separately), refreshing the index (out of scope for initial integration — PATH-04 is a separate requirement).

### State Reconciler (reconcile.cjs)

**Owns:** Diffing and merging STATE.md and agent-history.json from multiple worktree branches. Writes the merged result via the existing Tools CLI.

**Interface:**
```
reconcile(phase, wave, branches[]) → void
  // Reads STATE.md from each branch
  // Unions Decisions entries (deduplication by content hash)
  // Merges agent-history.json entries (union by agent_id)
  // Writes final STATE.md via: node gbsd-tools.cjs state update ...
  // Writes final agent-history.json
```

**Does NOT own:** Merging source code files (git merge handles that), validating that execution succeeded (verifier agent handles that).

---

## Integration Points with Existing GBSD Architecture

### execute-phase.md Integration

Current execute-phase.md spawns executors sequentially (or parallel if `parallelization=true`). The new components plug in at three points:

1. **Before wave spawn:** Worktree Manager creates worktrees; Key Pool Manager assigns keys
2. **During wave:** Executors operate in their worktrees unchanged (executors are not modified)
3. **After wave:** Worktree Manager tears down; State Reconciler merges state

The existing `parallelization` config flag gates the entire new behavior. When false, no worktrees are created and Key Pool assigns the single default key.

### plan-phase.md Integration

Current plan-phase.md spawns gbsd-phase-researcher then gbsd-planner. Pathfinder Adapter adds a step:

1. **After researcher, before planner:** Pathfinder Adapter runs if `isAvailable()` returns true
2. **Pathfinder output** is passed as an additional context block to gbsd-planner prompt
3. **If Pathfinder unavailable:** planner receives no Pathfinder block (graceful degradation — existing behavior unchanged)

The planner's internal decomposition algorithm changes (file-ownership-first), but the plan frontmatter schema changes are minimal and backward-compatible.

### config.json Integration

New configuration fields are additive — existing configs without them get safe defaults:

```json
{
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2,
    "worktree_strategy": "wave",
    "worktree_cleanup_policy": "aggressive",
    "session_timeout_minutes": 60
  },
  "api_management": {
    "strategy": "single",
    "organizations": [
      { "org_id": "default", "api_key": "", "role": "primary" }
    ],
    "load_balancing": "round-robin",
    "failover": {
      "enabled": true,
      "circuit_breaker": true,
      "max_retries": 3
    }
  }
}
```

The existing `parallelization: true/false` boolean is migrated to `parallelization.enabled` at first run.

### STATE.md Integration

STATE.md grows two new fields used by the State Reconciler:

```markdown
## Execution Metadata
Last reconciled: 2026-03-15T10:30:00Z
Reconciled waves: [phase-03-wave-1, phase-03-wave-2]
```

These are appended — all existing sections remain unchanged.

---

## Build Order (Dependency Graph for Phases)

The components must be built in this order because each unlocks the next:

```
1. worktree.cjs (Worktree Manager)
   └─► Enables: wave-parallel execution at all
       Prerequisite for: State Reconciler, Phase-level parallelism

2. keypool.cjs (Key Pool Manager)
   └─► Enables: multi-org rate limit separation
       Can be built in parallel with: worktree.cjs
       Prerequisite for: nothing else, but is meaningless without worktrees

3. reconcile.cjs (State Reconciler)
   └─► Requires: worktree.cjs (needs to know which branches to collect)
       Enables: correct STATE.md after parallel execution

4. execute-phase.md updates
   └─► Requires: worktree.cjs + keypool.cjs + reconcile.cjs all in place
       Enables: end-to-end parallel plan execution

5. pathfinder.cjs (Pathfinder Adapter) [parallel track — independent of 1-4]
   └─► Enables: accurate dependency detection in planner

6. plan-phase.md + gbsd-planner.md updates
   └─► Requires: pathfinder.cjs (optional, but needed for full benefit)
       Enables: 70-85% parallelizability through file-ownership decomposition

7. Phase-level parallelism (PARA-03)
   └─► Requires: worktree.cjs + reconcile.cjs proven stable (from step 4)
       Requires: pathfinder.cjs for cross-phase dependency analysis
```

Phase 1 implementation (worktrees + key pool + reconciler + execute-phase update) is the critical path. Pathfinder integration is a parallel track that does not block parallel execution.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Executors Reading Each Other's Worktrees

**What people do:** Pass all worktree paths to every executor so they can read sibling progress.

**Why it's wrong:** Introduces race conditions. Executor A reads Executor B's partially-committed file and makes decisions based on incomplete state. This is especially dangerous with TypeScript type files — a half-written type declaration causes compilation errors.

**Do this instead:** Executors are fully isolated. All cross-executor coordination happens through the orchestrator after wave completion.

### Anti-Pattern 2: State Mutation During Parallel Execution

**What people do:** Have executors write directly to the main branch's STATE.md as they progress.

**Why it's wrong:** Parallel writes to the same file on the same branch cause git merge conflicts on a file that is supposed to be immutable during execution. The reconciler cannot distinguish intentional updates from accidental overwrites.

**Do this instead:** Each executor writes STATE.md updates only to its own worktree branch. The State Reconciler merges after all executors complete.

### Anti-Pattern 3: Storing Circuit-Breaker State in config.json

**What people do:** Persist blacklisted keys or failure counts to disk to survive restarts.

**Why it's wrong:** A key that was rate-limited 5 minutes ago is likely available again. Persisting blacklist state causes permanent starvation if the write happens at an unlucky moment. Rate limit windows reset within minutes; persisted state outlives that window.

**Do this instead:** Keep Key Pool Manager state in-memory only (process lifetime). On process restart, all keys start healthy. If a key is genuinely exhausted, the circuit-breaker will rediscover this within the first few requests.

### Anti-Pattern 4: Pathfinder Coupling (Hard Dependency)

**What people do:** Make the planner refuse to run without a valid `.code-intel/` index.

**Why it's wrong:** Pathfinder is a separate tool that users may not have installed. On a new project with no code yet, there is no index to query. This breaks the existing workflow for all users who don't use Pathfinder.

**Do this instead:** `pathfinder.cjs.isAvailable()` returns false gracefully. Planner receives no Pathfinder context block and uses manual file-ownership reasoning. Everything works — just slower and less accurate than with Pathfinder.

### Anti-Pattern 5: Worktrees in the Project Directory

**What people do:** Create worktrees inside the project directory (e.g., `./worktrees/`).

**Why it's wrong:** Claude Code's file tools will discover and read these worktrees as part of the project structure. The executor in worktree A may accidentally read files from worktree B by exploring the filesystem. Also, `.gitignore` patterns that apply to the project root may not apply correctly inside nested worktrees.

**Do this instead:** Store worktrees in `~/.claude/worktrees/` (outside the project directory). Executors receive the absolute path explicitly and only operate in their assigned path.

---

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1 concurrent agent (current) | No worktrees needed; sequential execution; single API key |
| 2-3 concurrent agents (Phase 1 target) | Wave-parallel worktrees; round-robin across 2-3 Anthropic orgs; simple reconciler |
| 5-8 concurrent agents | Add capacity-aware key selection; disk-space pre-check; memory check before spawning |
| Phase-level parallelism | Requires Pathfinder for cross-phase dep analysis; more aggressive State Reconciler |

**First bottleneck:** Anthropic API rate limits (ITPM). With 3 parallel executors each sending 4K tokens per request, a Tier 2 org (~100K ITPM) saturates at ~25 concurrent requests. Multi-org key pool resolves this.

**Second bottleneck:** Disk I/O for worktree creation. `git worktree add` on a large repo (>500 MB) takes 2-10 seconds per worktree. Not a blocker at 3 agents, but relevant at 8+.

**Third bottleneck:** Lead context growth. The orchestrator accumulates tool call results over a long wave. At 8 concurrent agents with long SUMMARY.md files, the lead context can approach 40-50% usage. Keep SUMMARY.md concise; leads read only frontmatter fields, not full content.

---

## Sources

All findings are HIGH confidence, derived from first-party design documents:

- `docs/GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md` — Worktree lifecycle, State Reconciliation, config schema
- `docs/design-teams-comms.md` — Hub-and-spoke patterns, checkpoint protocol, message schemas
- `docs/planner-redesign-analysis.md` — File-ownership-first decomposition, wave algorithm redesign
- `docs/api-rate-augmentation-analysis.md` — Key Pool Manager design, multi-org strategy, circuit-breaker
- `.planning/codebase/ARCHITECTURE.md` — Existing layers, Tools CLI structure, Agent Layer inventory
- `gbsd/workflows/execute-phase.md` — Current orchestration entry point and parallelization integration point

---

*Architecture research for: GBSD parallel agent orchestration enhancement*
*Researched: 2026-03-15*
