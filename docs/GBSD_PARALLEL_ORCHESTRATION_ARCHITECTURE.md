# GBSD Parallel Orchestration Architecture

**Status:** Architecture Analysis for Task #6
**Date:** 2026-03-15
**Scope:** Wave-level and phase-level parallelism via worktree execution
**Context:** Foundation for wave-parallel (#8), phase-parallel (#9), and api-augment (#10) implementations

---

## Executive Summary

GBSD's current orchestration model (execute-phase.md) is **sequentially structured**: each wave executes plans one-at-a-time through spawned gsd-executor agents. This analysis designs a **parallel execution layer** that:

1. **Spawns executor agents in isolated worktrees** — each agent gets its own `.git worktree` and session context
2. **Manages state across N parallel sessions** — reconciles multiple `.planning/` copies, STATE.md updates, and branch merges
3. **Integrates with existing workflows** — backward-compatible with single-session mode; parallel is opt-in via config
4. **Handles interruption and resumption** — persisted worktree branches survive restarts; orchestrator detects and resumes

**Key Design Decisions:**
- Worktrees are **ephemeral** — created at wave start, deleted after merge-back completes
- Each worktree is a **fresh execution environment** — executor reads full context independently
- State reconciliation happens **after merge-back** — no real-time synchronization during parallel execution
- Configuration controls parallelism depth: `parallelization.enabled`, `parallelization.max_concurrent_agents`

---

## 1. Orchestrator Lifecycle

### 1.1 Phase Initialization → Worktree Creation

```
[User: /gsd:execute-phase 3]
  ↓
[execute-phase orchestrator loads STATE.md]
  ↓
[Discover plans in phase 3]
  ↓
[Group plans into waves via dependencies]
  ↓
[For each wave:
  - IF parallelization.enabled AND plan_count ≥ min_plans_for_parallel:
      CREATE WORKTREES FOR EACH PLAN
    ELSE:
      EXECUTE PLANS SEQUENTIALLY
```

### 1.2 Worktree Creation & Session Spawning

**Before spawning executors:**

```bash
# For each plan P in current wave:
PLAN_ID="03-01"
WAVE="1"
WORKTREE_NAME="gsd-exec-${PHASE}-${PLAN_ID}-w${WAVE}"
BRANCH_NAME="gsd/exec/${PHASE}-${PLAN_ID}-w${WAVE}"

# Create worktree directory
git worktree add ".claude/worktrees/${WORKTREE_NAME}" \
  -b "$BRANCH_NAME" origin/main

# At this point:
# - New git worktree exists with isolated .git/
# - New branch "$BRANCH_NAME" created from main
# - Executor agent will cd into this directory
```

**Executor receives:**
```bash
Task(
  prompt="<orchestration_context>
  Execute plan ${PLAN_ID} in isolated worktree.
  Worktree path: ${WORKTREE_PATH}
  Branch: ${BRANCH_NAME}

  <isolated_context>
  You have an isolated git worktree. Work here locally.
  This is YOUR session — no shared state with other executors.
  Push is BLOCKED (worktrees can't push from shallow clones).
  </isolated_context>

  <files_to_read>
  Read these files from worktree directory:
  - .planning/STATE.md
  - .planning/ROADMAP.md
  - .planning/config.json
  - {phase_dir}/{plan_file}.md
  </files_to_read>

  <success_criteria>
  - [ ] All tasks executed and committed to ${BRANCH_NAME}
  - [ ] SUMMARY.md created
  - [ ] Commits visible in: git log --oneline --all
  </success_criteria>
  ",
  subagent_type="gsd-executor",
  model=executor_model,
  isolation="worktree",  // NEW: tells Claude Code to use isolated worktree
  worktree_path="${WORKTREE_PATH}"  // NEW: passes isolation context
)
```

**Key design:** Executor doesn't know it's parallel. It just executes normally. The orchestrator manages parallelism via worktree isolation.

### 1.3 Parallel Execution & Monitoring

```
Wave 1:
  Plan 03-01 → Executor Agent A → Worktree A
  Plan 03-02 → Executor Agent B → Worktree B
  Plan 03-03 → Executor Agent C → Worktree C

  [All three execute in parallel, independent Claude Code sessions]
  [Orchestrator WAITS for all to complete (blocking join)]
```

**Monitoring:**
- Orchestrator polls each executor agent's Task status
- No communication between parallel executors (they're independent)
- Each executor writes to its own `.planning/` copy
- On completion, orchestrator collects results

### 1.4 Merge-Back & State Reconciliation

**After all executors in wave complete:**

```bash
# 1. Merge executor branches into main (per merge-back-protocol.md)
for plan in wave_plans; do
  BRANCH="gsd/exec/${PHASE}-${PLAN}-w${WAVE}"

  # Auto-resolve conflicts
  git merge --no-ff "$BRANCH"  # Handles lock files, config, state

  # Test merged code
  npm test

  # Delete worktree
  git worktree remove ".claude/worktrees/${WORKTREE_NAME}"

  # Delete branch
  git branch -d "$BRANCH"
done

# 2. Reconcile STATE.md from all executors
# (See section 2 for state reconciliation logic)

# 3. Continue to next wave or verify phase
```

### 1.5 Failure & Interruption Handling

**If executor fails mid-wave:**

```
Wave 1:
  Plan 03-01 → Executor A [✓ COMPLETE, worktree persisted]
  Plan 03-02 → Executor B [✗ FAILED, worktree branch remains]
  Plan 03-03 → Executor C [? AWAITING, not started yet]

User interrupts or failure detected:
  - Orchestrator STOPS spawning remaining executors
  - Worktrees & branches persist to disk
  - User options:
    1. Retry Plan 03-02: `git worktree remove .../03-02; git branch -d gsd/exec/03-02-w1`
       Then re-run `/gsd:execute-phase 3 --resume`
    2. Continue anyway: `git branch -d gsd/exec/03-02-w1; git worktree remove .../03-02`
       Then resume (skips 03-02)
```

**On restart detection:**

```bash
# Orchestrator discovers persisted worktrees
STALE_WORKTREES=$(git worktree list | grep detached)

if [ ! -z "$STALE_WORKTREES" ]; then
  echo "Found incomplete worktrees from previous run:"
  echo "$STALE_WORKTREES"

  user_choice=$(AskUserQuestion "Resume from interruption?")

  if "Yes":
    # Discover which wave was interrupted
    STALE_BRANCHES=$(git branch -l "gsd/exec/*" | sort)
    RESUME_WAVE=$(detect_incomplete_wave "$STALE_BRANCHES")

    # Resume that wave (already has completed branches merged)
    /gsd:execute-phase ${PHASE} --resume-wave ${RESUME_WAVE}
  else:
    # Clean up
    git worktree remove [stale worktrees]
    continue
fi
```

---

## 2. State Management Across Worktrees

### 2.1 Problem: Multiple `.planning/` Copies

Each worktree is a **full git clone** with its own working directory. When executor A reads `.planning/STATE.md`, it's reading from Worktree A's `.planning/` directory — which is **independent** of Worktree B's `.planning/`.

**Three categories of state:**

| State Type | Scope | Problem | Solution |
|-----------|-------|---------|----------|
| **Immutable input** (ROADMAP.md, REQUIREMENTS.md) | All worktrees | All executors read same files (main branch) | Git worktree isolation — no read conflicts |
| **Mutable execution** (agent-history.json) | Individual executor | Each executor appends its own history | Append-only merge after wave completes |
| **Phase tracking** (STATE.md, progress) | All worktrees | Multiple executors update position → conflicts | Last-write-wins + manual reconciliation |

### 2.2 STATE.md Reconciliation Algorithm

**Immutable sections (from ROADMAP.md — no reconciliation needed):**
```markdown
## Current Position
Phase: 3 of 8 (Phase name)
```

**Mutable sections (from executors — must reconcile):**

```markdown
## Decisions
[Each executor appends its decisions]

## Session Continuity
[Last completed action across all executors]

## Performance Metrics
[Aggregate velocity from all executors]
```

**Reconciliation process:**

```bash
# After all executors in wave complete:

# 1. Collect STATE.md copies from all worktrees
for WORKTREE in .claude/worktrees/gsd-exec-03-*-w1; do
  cp "$WORKTREE/.planning/STATE.md" "/tmp/STATE-${WORKTREE_NAME}.md"
done

# 2. Parse decision sections
# Each executor's STATE.md has:
# ## Decisions
# - [Phase 03-01]: Decision A
# - [Phase 03-02]: Decision B

# 3. Merge decisions (union)
DECISIONS=$(
  for state_file in /tmp/STATE-*.md; do
    grep "^- \[Phase" "$state_file"
  done | sort -u
)

# 4. Merge metrics (sum/average)
TOTAL_PLANS=$(grep "Total plans completed" /tmp/STATE-*.md | \
  awk '{sum+=$NF} END {print sum}')
AVG_DURATION=$(grep "Average duration" /tmp/STATE-*.md | \
  awk '{sum+=$NF; count++} END {print sum/count}')

# 5. Update main STATE.md
sed -i "s/^- \[Phase.*$/$(echo "$DECISIONS" | tr '\n' '|' | sed 's/|/\n/g')/" \
  .planning/STATE.md
sed -i "s/Total plans completed:.*/Total plans completed: $TOTAL_PLANS/" \
  .planning/STATE.md

# 6. Commit merged state
git add .planning/STATE.md
git commit -m "docs: reconcile state from wave ${WAVE} executors"
```

### 2.3 Agent History Merge (JSON Merge)

**Each executor appends to agent-history.json:**

```json
// Executor A writes to Worktree A
{
  "version": 1,
  "entries": [
    { "agent_id": "exec-03-01-w1", "status": "complete", ... }
  ]
}

// Executor B writes to Worktree B
{
  "version": 1,
  "entries": [
    { "agent_id": "exec-03-02-w1", "status": "complete", ... }
  ]
}
```

**After merge-back:**

```bash
# Merge-back protocol detects agent-history.json conflict
# Strategy: Union entries by agent_id (no duplicates)

jq -s '{
  version: .[0].version,
  entries: (.[0].entries + .[1].entries + ...) | unique_by(.agent_id) | sort_by(.agent_id)
}' \
  <(git show HEAD~1:agent-history.json) \
  <(git show gsd/exec/03-01-w1:agent-history.json) \
  <(git show gsd/exec/03-02-w1:agent-history.json) \
  > agent-history.json

git add agent-history.json
git commit -m "merge: reconcile agent history from wave ${WAVE}"
```

### 2.4 ROADMAP.md & REQUIREMENTS.md Updates

**These are RARELY updated by executors** (planner creates them, verifier consumes them). If executor needs to update:

```markdown
# In executor's worktree:
# Executor marks plan as "complete" in ROADMAP.md progress table

## Phase 3: Core Features
| Plan | Status | Date |
| 03-01 | ✓ Complete | 2026-03-15 |  ← Executor A adds this
```

**Merge strategy:** Last-write-wins (Wave 2 overwrites Wave 1 if conflict)

```bash
if merge_conflict ROADMAP.md; then
  git checkout --theirs ROADMAP.md  # Take last executor's version
  git add ROADMAP.md
  git commit -m "merge: reconcile ROADMAP progress"
fi
```

---

## 3. Configuration Schema Extension

### 3.1 Current Config Structure

```json
{
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "skip_checkpoints": true,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2
  }
}
```

### 3.2 NEW Parallel-Specific Fields

```json
{
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "max_concurrent_agents": 3,           // NEW: max worktree sessions simultaneously
    "min_plans_for_parallel": 2,           // NEW: below this, execute sequentially
    "worktree_strategy": "wave",           // NEW: "wave" | "phase" | "both"
    "worktree_cleanup_policy": "aggressive", // NEW: "aggressive" (delete after test) | "conservative" (keep branches)
    "api_key_pool": [                      // NEW: round-robin API keys across agents
      "sk-...",
      "sk-...",
      "sk-..."
    ],
    "session_timeout_minutes": 60          // NEW: max execution time per executor
  }
}
```

### 3.3 Semantics

**`worktree_strategy`:**
- `"wave"` — Parallelize within each wave (current design)
- `"phase"` — Parallelize across entire phase (all waves simultaneously)
- `"both"` — Adaptive: use phase-level if no dependencies, wave-level if dependencies exist

**`max_concurrent_agents`:**
- Default: 3 (safe for Claude API rate limits)
- Can be tuned based on API plan / rate limits
- Orchestrator caps parallelism here

**`api_key_pool`:**
- List of API keys for round-robin distribution
- Each executor agent gets key `pool[agent_index % pool.length]`
- Increases effective rate limit: `3 agents × 100 RPM = 300 RPM effective`

**`session_timeout_minutes`:**
- Max execution time for single executor
- Prevents runaway agents
- Default: 60 min (reasonable for medium plans)
- Escalates to user if exceeded: "Plan 03-01 exceeded timeout, retry or skip?"

---

## 4. Progress Reporting & Visibility

### 4.1 Current State (Sequential)

```bash
/gsd:progress

## Phase 3: Core Features
Plan: 1/5
Status: Executing plan 03-02...
  Task: 3/8 - [task name]
```

### 4.2 NEW: Multi-Session Visibility

```bash
/gsd:progress

## Phase 3: Core Features [WAVE 1 - PARALLEL]

| Plan | Task Progress | Status | ETA |
|------|---------------|--------|-----|
| 03-01 | ████░░░░░░ 40% | Running task 5/8 | 12 min |
| 03-02 | ██████░░░░ 60% | Running task 4/6 | 8 min |
| 03-03 | ██░░░░░░░░ 20% | Running task 2/8 | 18 min |
| **Aggregate** | **████░░░░░░ 40%** | **ETA: ~18 min** | |

---

Recent activity:
- 03-01: Completed database schema (task 4/8)
- 03-02: Created API routes (task 3/6)
- 03-03: Starting models (task 1/8)

Parallel efficiency: 3 agents running simultaneously
Rate limit headroom: 150 RPM available (300 RPM used, 450 RPM limit)
```

### 4.3 Implementation: `/gsd:progress` Enhancement

```bash
# Current: Sequential progress
git log --oneline -5 --all --grep="phase-03"

# NEW: Parallel progress tracking
function get_parallel_progress() {
  for WORKTREE in .claude/worktrees/gsd-exec-03-*-w${WAVE}; do
    PLAN=$(basename "$WORKTREE" | sed 's/.*-\([0-9][0-9]\)-.*/\1/')

    # Read SUMMARY.md progress from worktree
    if [ -f "$WORKTREE/.planning/phases/03-${PLAN}/*-SUMMARY.md" ]; then
      PROGRESS=$(grep "## Progress" "$WORKTREE/.planning/phases/03-${PLAN}/*-SUMMARY.md")
      echo "| $PLAN | $PROGRESS | Running |"
    fi
  done
}

get_parallel_progress
```

### 4.4 Dashboard Option (Future)

```
POST /gsd:progress --watch

[Live update dashboard in Claude Code terminal]
Refreshes every 5 seconds
Shows all parallel agents' progress
```

---

## 5. Interruption & Resume Protocol

### 5.1 Detecting Interruption on Restart

```bash
# Orchestrator detects worktrees from previous session
function detect_interrupted_waves() {
  WORKTREES=$(git worktree list | grep "gsd-exec-${PHASE}")

  if [ ! -z "$WORKTREES" ]; then
    echo "Found incomplete worktrees:"
    echo "$WORKTREES"

    # Extract wave number from worktree names
    WAVES=$(echo "$WORKTREES" | sed 's/.*-w\([0-9]*\).*/\1/' | sort -u)

    return {interrupted: true, waves: $WAVES}
  fi

  return {interrupted: false}
}
```

### 5.2 Resume Workflow

```bash
user@machine:project$ /gsd:execute-phase 3 --resume

[Orchestrator detects Wave 1 was interrupted mid-execution]

Found incomplete execution from previous session:
  Wave 1:
    - Plan 03-01: COMPLETE (merged)
    - Plan 03-02: IN PROGRESS (branch: gsd/exec/03-02-w1, worktree: .claude/worktrees/gsd-exec-03-02-w1)
    - Plan 03-03: NOT STARTED (no branch)

Options:
  1. Retry plan 03-02 (discard and start fresh)
  2. Continue with 03-02 (resume from last checkpoint)
  3. Skip to next wave (delete 03-02, merge completed plans)
```

### 5.3 Resume Implementation

**Case 1: Retry (discard worktree, start fresh)**

```bash
# User chooses "Retry 03-02"

# 1. Clean up previous attempt
git worktree remove .claude/worktrees/gsd-exec-03-02-w1
git branch -d gsd/exec/03-02-w1

# 2. Spawn fresh executor for plan 03-02
# (Same as original spawning)
git worktree add ".claude/worktrees/gsd-exec-03-02-w1" \
  -b "gsd/exec/03-02-w1" origin/main

Task(
  prompt="Resume plan 03-02 from scratch..."
  isolation="worktree"
  worktree_path=".claude/worktrees/gsd-exec-03-02-w1"
)
```

**Case 2: Continue from checkpoint (preserve worktree)**

```bash
# User chooses "Continue with 03-02"

# 1. Executor agent resumes in same worktree
# (Reads SUMMARY.md to understand prior progress)

Task(
  prompt="Resume plan 03-02 from prior session.

  Prior progress from SUMMARY.md:
  [Task 1/8: ✓ Complete]
  [Task 2/8: ✓ Complete]
  [Task 3/8: ✗ Failed - [reason]]

  Resume from task 3.
  "
  isolation="worktree"
  worktree_path=".claude/worktrees/gsd-exec-03-02-w1"  # Same path
)
```

**Case 3: Skip & continue (merge completed, delete failed)**

```bash
# User chooses "Skip plan 03-02"

# 1. Delete incomplete work
git worktree remove .claude/worktrees/gsd-exec-03-02-w1
git branch -d gsd/exec/03-02-w1

# 2. Completed plans from Wave 1 (03-01) already merged
# 3. Proceed to Wave 2
```

---

## 6. Integration with Existing Commands

### 6.1 `/gsd:progress` Enhancements

**Current behavior (sequential):**
```
Executing Phase 3, Plan 3/5: Dashboard Layout
Task: 4/8 complete
```

**NEW behavior (parallel):**
```
Executing Phase 3, Wave 1 [PARALLEL]
  Plan 03-01: ████░░░░░░ 40% (Task 4/8)
  Plan 03-02: ██████░░░░ 60% (Task 5/7)
  Plan 03-03: ██░░░░░░░░ 20% (Task 2/9)
Aggregate progress: 40% | ETA: 18 min
```

### 6.2 `/gsd:pause-work` with Parallel Support

**Current:**
```bash
/gsd:pause-work

[Pauses current agent, records STATE.md]
```

**NEW:**
```bash
/gsd:pause-work

[Detects N parallel agents running]

Pausing 3 parallel agents in Wave 1...
  ✓ Plan 03-01 paused (worktree: .../03-01-w1)
  ✓ Plan 03-02 paused (worktree: .../03-02-w1)
  ✓ Plan 03-03 paused (worktree: .../03-03-w1)

Merged completed: Plan 03-01 ✓

Worktrees persisted. Resume with:
  /gsd:execute-phase 3 --resume
```

### 6.3 `/gsd:resume-work` with Parallel Support

```bash
/gsd:resume-work

[Detects Wave 1 was paused mid-execution]
[Offers resume options per section 5.2]
```

### 6.4 NEW: `/gsd:merge-back` Manual Command

```bash
/gsd:merge-back --phase 3 --wave 1

[Manually triggers merge-back for a wave]
[Useful if auto-merge failed or user wants control]

Merging Wave 1 plans...
  ✓ Plan 03-01 merged
  ✓ Plan 03-02 merged (1 auto-resolved conflict in package.json)
  ✓ Plan 03-03 merged

Tests: ✓ All passing

Wave 1 merge complete. Worktrees cleaned up.
Ready for Wave 2.
```

---

## 7. Backward Compatibility

### 7.1 Single-Session Fallback

**When parallelization is disabled or N < min_plans_for_parallel:**

```bash
# Config: parallelization.enabled = false

/gsd:execute-phase 3

[Execute-phase falls back to sequential mode]
[For each plan:
  - Spawn single executor (no worktree)
  - Wait for completion
  - Merge to main
  - Proceed to next plan
]

[No worktrees created, behavior identical to upstream GSD]
```

### 7.2 Graceful Degradation

**If Claude API rate limits hit (too many parallel agents):**

```bash
# Config: max_concurrent_agents = 3, but user has 5 plans in wave

[Orchestrator queues executors]

Spawning agents: 03-01, 03-02, 03-03
Queueing: 03-04, 03-05

[Wait for first 3 to complete, then spawn 03-04, 03-05]

Progress:
  Wave 1 (batch 1/2): ████░░░░░░
  Wave 1 (batch 2/2): ░░░░░░░░░░ [Waiting for batch 1]
```

### 7.3 Config Migration

**Existing config.json (no parallelization fields):**

```bash
# On first run with new orchestrator, auto-populate defaults

{
  "parallelization": true,  // Existing field

  // NEW defaults added
  "parallelization": {
    "enabled": true,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2,
    "worktree_strategy": "wave",
    "api_key_pool": ["sk-..."],  // Auto-populated from existing API key
    "session_timeout_minutes": 60
  }
}
```

---

## 8. Resource Management

### 8.1 Disk Space Estimation

**Per worktree:**
- `.git/` (shallow clone): ~50-200 MB
- Working directory (copy): ~100-500 MB (varies by project size)
- **Total per worktree: 150-700 MB**

**For 3 parallel agents:**
- 3 worktrees: ~450-2100 MB
- Recommendation: Require 3 GB free before parallel execution

**Detection:**
```bash
AVAILABLE_DISK=$(df .claude/worktrees/ | awk 'NR==2 {print $4}')  # KB
REQUIRED=3000000  # 3 GB in KB
WORKTREE_ESTIMATE=$(($MAX_AGENTS * 400))  # 400 MB per worktree est.

if [ "$AVAILABLE_DISK" -lt "$WORKTREE_ESTIMATE" ]; then
  echo "⚠ Insufficient disk space for parallel execution"
  echo "Available: $((AVAILABLE_DISK / 1024)) MB"
  echo "Required: ~$((WORKTREE_ESTIMATE / 1024)) MB"
  echo "Continuing in sequential mode..."
fi
```

### 8.2 Memory per Claude Code Session

**Baseline (single agent):** ~200-400 MB
**Per parallel agent:** +200-400 MB
**For 3 agents:** 600-1200 MB (reasonable on modern laptops)

**Detection:**
```bash
AVAILABLE_MEM=$(free -m | awk 'NR==2 {print $7}')  # MB free
REQUIRED_PER_AGENT=300  # MB
TOTAL_REQUIRED=$((MAX_AGENTS * REQUIRED_PER_AGENT))

if [ "$AVAILABLE_MEM" -lt "$TOTAL_REQUIRED" ]; then
  echo "⚠ Low memory available"
  # Reduce max_concurrent_agents
fi
```

### 8.3 API Rate Limiting

**Current:** Single agent max ~100 RPM (Claude API standard)
**With API key pool:** `N agents × 100 RPM = N × 100 RPM`

Example:
- Pool: 3 API keys
- 3 agents running in parallel
- Effective rate: 300 RPM (if all keys have 100 RPM limits)

**Configuration:**
```json
{
  "api_key_pool": [
    "sk-proj-...",  // Key 1: 100 RPM
    "sk-proj-...",  // Key 2: 100 RPM
    "sk-proj-..."   // Key 3: 100 RPM
  ]
}
```

**Distribution algorithm:**
```bash
function assign_api_key() {
  AGENT_INDEX=$1
  POOL_SIZE=${#api_key_pool[@]}
  KEY_INDEX=$((AGENT_INDEX % POOL_SIZE))

  echo "${api_key_pool[$KEY_INDEX]}"
}

# Agent 0 gets key 0
# Agent 1 gets key 1
# Agent 2 gets key 2
# Agent 3 gets key 0 (wraps around)
```

---

## 9. Checkpoint Handling in Parallel

### 9.1 Checkpoint Types

```
1. human-verify: "Does X look right?"
2. human-action: "Can you run command Y?"
3. decision: "Which approach?"
```

### 9.2 Blocking Checkpoints (Pauses Wave)

**If Plan 03-01 hits a checkpoint:**

```
Wave 1:
  Plan 03-01 → Executor A [BLOCKED at checkpoint]
  Plan 03-02 → Executor B [Continue running ✓]
  Plan 03-03 → Executor C [Continue running ✓]

[Orchestrator presents checkpoint to user]
[User responds]
[Executor A resumes]
[All three continue running]
[Orchestrator waits for ALL to complete before next wave]
```

**Implementation:**

```bash
# Executor A hits checkpoint, returns to orchestrator with:
{
  "status": "checkpoint_reached",
  "plan": "03-01",
  "checkpoint_type": "human-verify",
  "details": "Does authentication flow look right?",
  "completed_tasks": [...],
  "awaiting": "User approval"
}

# Orchestrator:
# 1. Present to user
# 2. Spawn continuation agent for Plan 03-01
# 3. Continuation resumes in same worktree
# 4. Other agents (B, C) continue unblocked
```

### 9.3 Atomic Checkpoints (All Plans Must Pass)

**If any plan hits a checkpoint → PAUSE ENTIRE WAVE**

```
config.json: skip_checkpoints = true

# All executors pause on checkpoint
# (prevents one agent from completing while another is blocked)
```

---

## 10. Integration with Merge-Back Protocol

### 10.1 Merge Ordering for Parallel Executors

**Current protocol:** Merge within wave alphabetically, then by wave

**NEW: Parallel-aware merge:**

```bash
# Wave 1 completes with 3 parallel executors
BRANCHES="gsd/exec/03-01-w1 gsd/exec/03-02-w1 gsd/exec/03-03-w1"

# Merge in order: 03-01, 03-02, 03-03
# (Protocol ensures no conflicts within wave if planner assigned files correctly)

for BRANCH in $(echo $BRANCHES | tr ' ' '\n' | sort); do
  git merge --no-ff "$BRANCH"
  npm test  # Test after each merge
  git branch -d "$BRANCH"
done
```

### 10.2 Conflict Auto-Resolution (Parallel-Aware)

**Issue:** All 3 executors modified package.json

```bash
# Merge conflict in package.json (lock file)
# Strategy: Union all dependencies, regenerate lock

git merge --no-ff gsd/exec/03-01-w1  # Merges A's package.json

git merge --no-ff gsd/exec/03-02-w1  # Conflict: both modified
# Auto-resolve:
jq -s '.[0].dependencies * .[1].dependencies' \
  <(git show :2:package.json) \
  <(git show :3:package.json) > package.json.merged
mv package.json.merged package.json

npm install --package-lock-only  # Regenerate lock
git add package.json package-lock.json
git commit -m "merge: resolve dependencies from parallel executors"
```

---

## 11. Monitoring & Debugging

### 11.1 Execution Trace

**Per-agent trace file:**

```bash
# Executor writes to isolated worktree
.claude/worktrees/gsd-exec-03-01-w1/.planning/trace-03-01.log

[2026-03-15 10:00:00] Task 1: Creating database schema
[2026-03-15 10:05:30] Task 1: ✓ Complete (commit abc123)
[2026-03-15 10:06:00] Task 2: Creating API routes
[2026-03-15 10:08:15] Task 2: ✓ Complete (commit def456)
```

**Orchestrator aggregates:**

```bash
# After wave completes, collect traces
for WORKTREE in .claude/worktrees/gsd-exec-03-*-w1; do
  cat "$WORKTREE/.planning/trace-*.log" >> .planning/trace-wave-1.log
done

# Result: Chronological trace of all parallel execution
```

### 11.2 Failure Diagnosis

**If merge-back fails:**

```bash
/gsd:diagnose-merge-failure --phase 3 --wave 1 --plan 03-02

[Orchestrator outputs:]

Merge conflict in: src/api/auth.ts
Executor branch: gsd/exec/03-02-w1
Worktree: .claude/worktrees/gsd-exec-03-02-w1

Conflict markers:
<<<<<<< HEAD
  [JWT implementation from Executor 01]
=======
  [OAuth implementation from Executor 02]
>>>>>>> gsd/exec/03-02-w1

Options:
  1. Take executor's version
  2. Take main's version
  3. Manual resolution
```

---

## 12. Implementation Phases

### Phase 1: Core Worktree Orchestration (Task #8)
- Implement worktree spawning in execute-phase.md
- Add `isolation="worktree"` parameter to executor agent
- Test with 2-agent parallel execution
- Config: `max_concurrent_agents=2`

### Phase 2: State Reconciliation (Task #6 Analysis → Implementation)
- Implement STATE.md merge algorithm
- Implement agent-history.json merge
- Test with 3-agent parallel execution
- Verify no data loss across state files

### Phase 3: API Augmentation (Task #10)
- Implement `api_key_pool` distribution
- Round-robin key assignment to agents
- Rate limit monitoring

### Phase 4: Phase-Level Parallelism (Task #9)
- Extend orchestrator to parallelize across phases (not just waves)
- Requires dependency analysis before phase execution
- New config: `worktree_strategy = "phase"`

### Phase 5: Planner Redesign (Task #4)
- Maximize parallelizable tasks within plans
- Auto-assign file ownership to prevent conflicts
- Dependency graph analysis

### Phase 6: Full System Integration
- Dashboard/progress visualization
- Resume protocol testing
- Comprehensive error handling

---

## 13. Configuration Template

```json
{
  "mode": "interactive",
  "granularity": "standard",
  "model_profile": "quality",
  "commit_docs": true,

  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true,
    "auto_advance": true,
    "nyquist_validation": true
  },

  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2,
    "worktree_strategy": "wave",
    "worktree_cleanup_policy": "aggressive",
    "api_key_pool": [
      "sk-proj-xxx"
    ],
    "session_timeout_minutes": 60,
    "skip_checkpoints": true,
    "checkpoint_block_policy": "all"
  },

  "resource_limits": {
    "min_disk_mb": 3000,
    "min_memory_mb": 900,
    "max_worktree_size_mb": 700
  },

  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true,
    "worktree_validation_before_merge": true
  }
}
```

---

## 14. Known Limitations & Future Work

### 14.1 Current Limitations

1. **No shared context between agents** — Each executor is fully independent (by design for isolation)
2. **Checkpoints block entire wave** — Atomic operation (can be relaxed in Phase 2)
3. **Manual merge conflict resolution required** — For source file conflicts (non-lock files)
4. **No distributed transaction semantics** — If merge fails partway, manual cleanup required
5. **Task-level parallelism deferred** — Task #12 (planner redesign) required for within-plan parallelism

### 14.2 Future Enhancements

1. **Real-time agent communication** — Allow executors to coordinate (Phase 4+)
2. **Distributed tracing** — Full causality across agents (observability)
3. **Predictive conflict detection** — Warn before merge if conflicts detected
4. **Automatic conflict resolution for source files** — ML-based merging (research phase)
5. **Dynamic rate limit adjustment** — Monitor API headroom, spawn more agents if available

---

## 15. Glossary

| Term | Meaning |
|------|---------|
| **Worktree** | Isolated git working directory with its own `.git/` metadata |
| **Executor agent** | Claude Code agent instance running in isolated worktree |
| **Merge-back** | Process of merging executor branches back into main after wave completes |
| **Wave** | Set of independent plans that can execute in parallel |
| **Phase** | Collection of waves (1+ plans) working toward a single phase goal |
| **State reconciliation** | Process of merging mutable state (STATE.md, agent-history.json) from parallel executors |
| **API key pool** | List of API keys for distributing load across parallel agents |
| **Checkpoint** | Synchronization point where executor pauses and awaits user decision |

---

## Summary

This architecture enables **GBSD to scale from single-agent to N-agent parallel execution** while maintaining:

1. **Isolation** — Each agent works in independent worktree (no interference)
2. **Determinism** — Merge-back protocol is predictable and reproducible
3. **Debuggability** — Full branch history and trace logs preserved
4. **Backward compatibility** — Sequential mode remains default fallback
5. **Resource awareness** — Disk, memory, API rate limits all monitored

**Key insight:** Parallelism happens at the **orchestrator level** (spawning agents), not the **executor level** (each agent unaware of siblings). This keeps executor code simple while orchestrator manages complexity.

The design is **iterative**: Wave-level parallelism (Phase 1) lands first, then phase-level, then with API augmentation and planner redesign, full-system optimization becomes possible.

---

*Analysis completed: 2026-03-15 by orchestrator-arch agent*
