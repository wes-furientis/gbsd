# Agent Teams Communication Patterns for GSD Workflows

## Design Principles

- **Lean orchestrator:** Lead stays <20% context; delegates heavily
- **Fresh context per agent:** Each executor/researcher gets full 200k context
- **Worktrees for isolation:** Parallel executors on separate branches
- **Explicit signaling:** No polling; TaskUpdate + SendMessage for state
- **File-based checkpoints:** State persists across agent boundaries
- **Task dependency chains:** BlockedBy determines ready-to-start

---

## A. Execute-Phase (Wave-Based Parallel Executors)

### Overview
Wave-based execution: orchestrator groups plans by dependencies → spawns executor teammates per wave → waits for all to complete before next wave. Each executor works in a worktree on a dedicated branch.

### A1. Team Structure
- **Lead:** Orchestrator, spawns executor teammates, merges results
- **Executors (N per wave):** Each gets worktree + branch, executes 1 plan, signals completion
- **No synthesizer:** Results live on branches; lead merges and inspects

### A2. Worktree Communication

#### Create Worktree & Spawn Executor

**Lead action:**
1. Create worktree for executor:
   ```bash
   git worktree add ~/.claude/worktrees/wave1-exec1 -b feature/wave-1-plan-01
   WORKTREE_PATH="$HOME/.claude/worktrees/wave1-exec1"
   ```
2. Spawn executor teammate:
   ```json
   {
     "to": "executor-1",
     "message": "Execute plan 01-01 in worktree: /home/wes/.claude/worktrees/wave1-exec1\nPhase: 1\nPlan ID: 01-01\nBranch: feature/wave-1-plan-01\nInstructions: Read /path/to/plan/01-01-PLAN.md",
     "summary": "Assign plan 01-01 to executor-1 with worktree path"
   }
   ```

**Executor action:**
1. Change to worktree directory
2. Execute plan (read PLAN.md, run tasks, create SUMMARY.md)
3. Commit work on branch
4. Signal completion:
   ```json
   {
     "to": "team-lead",
     "message": {
       "type": "plan_complete",
       "plan_id": "01-01",
       "branch": "feature/wave-1-plan-01",
       "worktree": "/home/wes/.claude/worktrees/wave1-exec1",
       "summary_path": ".planning/phases/01-foundation/01-01-SUMMARY.md",
       "commit_hash": "abc123d",
       "status": "success"
     },
     "summary": "Plan 01-01 complete on feature/wave-1-plan-01"
   }
   ```

#### Mark Executor Task Complete

**Lead action (after message):**
```bash
# Find executor task (created when team spawned)
EXEC_TASK=$(TaskList | grep "Execute plan 01-01" | extract-id)
TaskUpdate(taskId="$EXEC_TASK", status="completed")
```

This removes the task from the executor's load, signals to task system that executor is idle.

### A3. Wave Synchronization

**Lead waits for all executors to signal:**
- Poll TaskList after spawning wave: `TaskList()`
- All wave tasks must reach `completed` status
- Alternative: Lead can peek at messages (non-blocking)

**Pseudo-code:**
```bash
# Spawn all executors in wave
for plan in $(echo "$PLANS_IN_WAVE"); do
  spawn_executor_for_plan "$plan"
done

# Wait for all to complete
while true; do
  COMPLETED=$(TaskList | grep "Wave 1" | grep -c "status: completed")
  TOTAL=$(echo "$PLANS_IN_WAVE" | wc -l)
  if [[ $COMPLETED -eq $TOTAL ]]; then
    break
  fi
  sleep 5
done
```

### A4. Result Collection & Merge

**Lead action after wave completes:**
1. Merge all branches back to main:
   ```bash
   # For each executor branch
   for branch in feature/wave-1-*; do
     git checkout main
     git merge --no-ff "$branch" -m "Merge $branch into main"
   done
   ```
2. Read SUMMARY.md from merged state:
   ```bash
   # SUMMARY files now exist on main after merge
   cat .planning/phases/01-foundation/01-01-SUMMARY.md
   cat .planning/phases/01-foundation/01-02-SUMMARY.md
   ```
3. Spot-check (verify files exist, commits present)
4. Clean up worktrees:
   ```bash
   git worktree remove ~/.claude/worktrees/wave1-exec1
   git worktree remove ~/.claude/worktrees/wave1-exec2
   ```

### A5. Task Structure for Wave Execution

**Before spawning executors:**
```bash
# Create N tasks, one per executor
# Each task blocked by previous wave (if any)
for plan in $(echo "$PLANS_IN_WAVE"); do
  TaskCreate(
    subject="Execute plan $plan",
    description="Execute $plan in worktree, commit to branch, signal lead",
    blockedBy=["$PREV_WAVE_TASK_ID"]  # if previous wave exists
  )
done
```

**Task dependency chain:**
```
Wave 1 (all tasks in parallel):
  - Execute plan 01-01
  - Execute plan 01-02

Wave 2 (all blocked by Wave 1):
  - Execute plan 01-03 (blockedBy: [Wave 1 tasks])
```

### A6. Checkpoint Handling (In-Wave)

**Executor hits checkpoint:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "checkpoint",
    "plan_id": "01-03",
    "checkpoint_type": "human-verify",
    "completed_tasks": ["task 1", "task 2"],
    "current_task": "task 3",
    "what_needs_verification": "Deploy to staging at https://...",
    "verification_steps": "1) Visit URL\n2) Log in with test@example.com\n3) Confirm form submits"
  },
  "summary": "Plan 01-03 hit checkpoint at task 3"
}
```

**Lead action:**
1. Present checkpoint to user
2. Get user response
3. Message executor with response:
   ```json
   {
     "to": "executor-3",
     "message": "Checkpoint response for 01-03: approved. Continue from task 3.",
     "summary": "Checkpoint approved for plan 01-03"
   }
   ```
4. Executor continues (fresh agent with continuation state)
5. Other executors in wave continue in parallel

---

## B. Research Phase (4 Parallel Researchers + Synthesizer)

### Overview
4 independent researchers write to separate files (.planning/research/STACK.md, FEATURES.md, ARCHITECTURE.md, PITFALLS.md). Synthesizer task blocked by all 4. No worktrees needed (different files, no merge conflicts).

### B1. Team Structure
- **Lead:** Orchestrator, spawns 4 researchers + synthesizer
- **Researchers (4):** Each writes to own file, signals completion
- **Synthesizer (1):** Waits for all 4, creates SUMMARY.md

### B2. Researcher Task Structure

**Lead action:**
```bash
# Create 4 independent research tasks
STACK_TASK=$(TaskCreate(
  subject="Research technology stack",
  description="Write to .planning/research/STACK.md"
))

FEATURES_TASK=$(TaskCreate(
  subject="Research features",
  description="Write to .planning/research/FEATURES.md"
))

ARCH_TASK=$(TaskCreate(
  subject="Research architecture",
  description="Write to .planning/research/ARCHITECTURE.md"
))

PITFALLS_TASK=$(TaskCreate(
  subject="Research pitfalls",
  description="Write to .planning/research/PITFALLS.md"
))

# Synthesizer blocked by all 4
SYNTH_TASK=$(TaskCreate(
  subject="Synthesize research",
  description="Read all 4 research files, create SUMMARY.md",
  blockedBy=["$STACK_TASK", "$FEATURES_TASK", "$ARCH_TASK", "$PITFALLS_TASK"]
))
```

### B3. Researcher Message Protocol

**Lead spawns researcher:**
```json
{
  "to": "researcher-stack",
  "message": "Research technology stack for Phase 1\nWrite to: .planning/research/STACK.md\nProject context: [project goal]\nComplete research, commit to git, signal lead",
  "summary": "Spawn stack researcher for phase 1"
}
```

**Researcher signals completion:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "research_complete",
    "research_type": "stack",
    "file_path": ".planning/research/STACK.md",
    "commit_hash": "xyz789a"
  },
  "summary": "Stack research complete"
}
```

**Lead marks task complete:**
```bash
TaskUpdate(taskId="$STACK_TASK", status="completed")
```

(Repeat for all 4 researchers. Tasks auto-trigger when TaskList polls; all 4 run in parallel because they have no blockedBy.)

### B4. Synthesizer Triggered Automatically

**Once all 4 research tasks complete:**
1. TaskList shows `SYNTH_TASK` is now unblocked
2. Lead spawns synthesizer:
   ```json
   {
     "to": "synthesizer",
     "message": "Synthesize research findings\nRead: .planning/research/STACK.md, FEATURES.md, ARCHITECTURE.md, PITFALLS.md\nWrite: .planning/research/SUMMARY.md\nCommit and signal",
     "summary": "Synthesize research phase"
   }
   ```
3. Synthesizer completes, marks SYNTH_TASK as completed

### B5. No Worktrees Needed

**Why:** Each researcher writes to a unique file. No merge conflicts. No branch coordination. Standard git workflow:
```bash
researcher-stack:
  git add .planning/research/STACK.md
  git commit -m "docs(research): technology stack"

researcher-features:
  git add .planning/research/FEATURES.md
  git commit -m "docs(research): features research"

# All 4 can commit to main in parallel — different files, no conflicts
```

---

## C. Plan-Phase Pipeline (Researcher → Planner → Checker)

### Overview
Sequential: researcher → planner → checker. Revision loop if issues found (max 3 iterations). Can be teams-based OR main-context (planner may be too heavyweight for subagent).

### C1. Decision: Subagent or Main Context?

- **Subagent route:** Research, Plan, Verify each spawn separate teammates
  - Pros: Parallel in theory (actually serial here), fresh context per agent
  - Cons: Adds messaging overhead for sequential workflow
- **Main-context route:** Planner + checker in same context
  - Pros: Tighter iteration loop, simpler state passing
  - Cons: Planner may exceed context limits

**Recommendation:** Use subagent route for clarity and reusability. Planner + checker are teammates.

### C2. Task Dependency Chain

```bash
RESEARCH_TASK=$(TaskCreate(
  subject="Research phase 1",
  description="Write RESEARCH.md"
))

PLAN_TASK=$(TaskCreate(
  subject="Plan phase 1",
  description="Read RESEARCH.md, write PLAN.md files",
  blockedBy=["$RESEARCH_TASK"]
))

CHECK_TASK=$(TaskCreate(
  subject="Verify phase 1 plans",
  description="Read PLAN.md files, check against requirements",
  blockedBy=["$PLAN_TASK"]
))
```

### C3. Researcher → Planner Handoff

**Researcher signals completion:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "research_complete",
    "phase": "1",
    "file": ".planning/phases/01-foundation/01-RESEARCH.md"
  },
  "summary": "Phase 1 research complete"
}
```

**Lead marks RESEARCH_TASK complete, planner auto-starts:**
```bash
TaskUpdate(taskId="$RESEARCH_TASK", status="completed")

# PLAN_TASK now unblocked; lead spawns planner
# No message needed if using TaskList polling
```

**Lead spawns planner:**
```json
{
  "to": "planner",
  "message": "Plan Phase 1\nRead: .planning/phases/01-foundation/01-RESEARCH.md\nRead: .planning/CONTEXT.md, REQUIREMENTS.md, STATE.md\nWrite: PLAN-*.md files in phase directory\nCommit and signal completion",
  "summary": "Spawn planner for phase 1"
}
```

### C4. Planner → Checker Handoff

**Planner signals completion:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "plan_complete",
    "phase": "1",
    "plan_count": 3,
    "plans": ["01-01-PLAN.md", "01-02-PLAN.md", "01-03-PLAN.md"]
  },
  "summary": "Phase 1 planning complete with 3 plans"
}
```

**Lead marks PLAN_TASK complete, checker auto-starts:**
```bash
TaskUpdate(taskId="$PLAN_TASK", status="completed")

# CHECK_TASK now unblocked; lead spawns checker
```

**Lead spawns checker:**
```json
{
  "to": "checker",
  "message": "Verify Phase 1 plans\nRead: .planning/phases/01-foundation/01-*.md\nRead: .planning/CONTEXT.md, REQUIREMENTS.md, STATE.md\nCheck: coverage, goal alignment, feasibility\nReturn: VERIFICATION PASSED or ISSUES FOUND",
  "summary": "Spawn checker for phase 1 plans"
}
```

### C5. Revision Loop (Max 3 Iterations)

**Checker finds issues:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "verification_failed",
    "phase": "1",
    "iteration": 1,
    "issues": [
      "Plan 01-03 doesn't address REQ-15",
      "Waves 1 and 2 have circular dependency"
    ]
  },
  "summary": "Verification iteration 1: 2 issues found"
}
```

**Lead spawns planner for revision (if iteration < 3):**
```json
{
  "to": "planner",
  "message": {
    "type": "revision_request",
    "phase": "1",
    "iteration": 2,
    "issues": [
      "Plan 01-03 doesn't address REQ-15",
      "Waves 1 and 2 have circular dependency"
    ]
  },
  "summary": "Revision 1: 2 issues to fix"
}
```

**Planner updates PLAN.md files, signals completion:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "plan_revised",
    "phase": "1",
    "iteration": 2,
    "changes": "Added task to 01-03 for REQ-15, reordered waves"
  },
  "summary": "Phase 1 planning revised, iteration 2"
}
```

**Lead respawns checker** (loop back to C4)

**If iteration >= 3 and issues remain:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "max_iterations",
    "phase": "1",
    "issues_remaining": 1,
    "issue_text": "Plan 01-04 scope ambiguous"
  },
  "summary": "Max iterations reached, 1 issue remains"
}
```

Lead presents to user: "Force proceed? / Provide guidance? / Abandon?"

### C6. Message Flow Diagram

```
Researcher
  ↓ (signal)
Lead: Check RESEARCH_TASK complete
  ↓
Planner (spawn)
  ↓ (signal)
Lead: Check PLAN_TASK complete
  ↓
Checker (spawn)
  ↓ (issues found?)
  ├─ Yes → Revision loop (max 3x)
  │  ├─ Planner (spawn)
  │  ├─ Signal complete
  │  └─ Checker (spawn)
  └─ No → VERIFICATION PASSED
```

---

## D. Checkpoint Protocol

### Overview
Executor hits checkpoint → writes state file → messages lead → lead presents to user → lead messages executor → executor continues (or lead spawns fresh agent).

### D1. Checkpoint State File

**Executor writes to `.planning/checkpoints/{phase}-{plan}-checkpoint.json`:**

```json
{
  "phase": "3",
  "plan": "03-02",
  "checkpoint_type": "human-verify",
  "task_number": 5,
  "task_name": "Deploy to staging",
  "completed_tasks": [
    { "num": 1, "name": "Setup database", "hash": "abc123" },
    { "num": 2, "name": "Create API routes", "hash": "def456" },
    { "num": 3, "name": "Test endpoints", "hash": "ghi789" },
    { "num": 4, "name": "Deploy to Vercel", "hash": "jkl012" }
  ],
  "current_task": {
    "num": 5,
    "name": "Verify deployment",
    "details": "Check staging site works end-to-end"
  },
  "verification_steps": [
    "1. Visit https://staging.example.com",
    "2. Create test account",
    "3. Create test post",
    "4. Verify database saved data"
  ],
  "checkpoint_details": {
    "what_to_test": "Full user flow (signup → create post → verify)",
    "verification_url": "https://staging.example.com"
  },
  "awaiting": "Human verification that staging works"
}
```

### D2. Executor Signals Lead

**Executor message:**
```json
{
  "to": "team-lead",
  "message": {
    "type": "checkpoint_reached",
    "plan_id": "03-02",
    "checkpoint_file": ".planning/checkpoints/03-02-checkpoint.json",
    "progress": "4/5 tasks complete"
  },
  "summary": "Plan 03-02 at checkpoint: human verification needed"
}
```

### D3. Lead Presents to User

Lead reads checkpoint file, displays nicely:

```
## Checkpoint: Verification

**Plan:** 03-02 Deploy User API
**Progress:** 4/5 tasks complete

**What's Been Built:**
1. Database setup ✓
2. API routes ✓
3. Endpoint tests ✓
4. Vercel deployment ✓

**Current Task:** Verify deployment
**What to Do:** Test the staging site

### Verification Steps:
1. Visit https://staging.example.com
2. Create test account
3. Create test post
4. Check database saved data

**Your Action:** Perform steps above, then respond:
  - "approved" — everything works
  - describe issues — what broke
  - "skip" — move to next task
```

(Lead can also offer: "done" / "issues" / "skip")

### D4. Lead Messages Executor

**After user responds:**
```json
{
  "to": "executor-3",
  "message": {
    "type": "checkpoint_response",
    "checkpoint_id": "03-02",
    "user_response": "approved"
  },
  "summary": "Checkpoint 03-02 approved, continue to task 5"
}
```

### D5. Executor Resumes or Lead Spawns Continuation

**Option A (Executor in main context):** Executor receives message, continues from task 5.

**Option B (Executor in subagent):** Executor returned already. Lead spawns fresh continuation agent:

```json
{
  "to": "executor-continuation",
  "message": {
    "type": "execute_continuation",
    "plan_id": "03-02",
    "resume_from_task": 5,
    "completed_tasks_summary": "[table from checkpoint file]",
    "user_approval": "approved"
  },
  "summary": "Continue plan 03-02 from task 5"
}
```

Continuation agent reads plan, skips tasks 1-4, executes task 5+, creates SUMMARY, commits.

---

## E. Result Aggregation (Post-Execution)

### Overview
After wave execution, lead needs to inspect all SUMMARY.md files. With worktrees on separate branches, lead merges branches, reads files from merged state. Alternative: read files from worktrees before cleanup.

### E1. Merge-After-Read Pattern (Recommended)

**Lead after all executors complete:**
```bash
# Executors have signaled completion and committed to their branches
# e.g., feature/wave-1-plan-01, feature/wave-1-plan-02, etc.

# Merge all wave branches into main
for branch in feature/wave-1-*; do
  git checkout main
  git merge --no-ff "$branch" -m "Merge $branch: $(git log $branch -1 --format=%s)"
done

# Now SUMMARY.md files exist on main
ls -1 .planning/phases/01-foundation/*-SUMMARY.md

# Read and inspect each
cat .planning/phases/01-foundation/01-01-SUMMARY.md
cat .planning/phases/01-foundation/01-02-SUMMARY.md
```

**Spot-check (verify execution success):**
```bash
for summary in .planning/phases/01-foundation/*-SUMMARY.md; do
  # Check file exists and has content
  [[ -f "$summary" ]] && grep -q "Self-Check: PASSED" "$summary" \
    && echo "✓ $summary: OK" \
    || echo "✗ $summary: FAILED"
done

# Check commits
git log --oneline --grep="feat(01-" | wc -l  # Should be ≥2

# Check key-files exist
grep "^key-files:" .planning/phases/01-foundation/*-SUMMARY.md | wc -l
```

### E2. Read-Before-Cleanup Pattern (If needed)

**Alternative if merging is not possible:**
```bash
# Before cleaning up worktrees, read SUMMARY files from each branch
for worktree in ~/.claude/worktrees/wave1-*; do
  SUMMARY="$worktree/.planning/phases/01-foundation/*-SUMMARY.md"
  cat "$SUMMARY" > /tmp/aggregated-$(basename $worktree).md
done

# Clean up worktrees
git worktree remove ~/.claude/worktrees/wave1-exec1
git worktree remove ~/.claude/worktrees/wave1-exec2

# Analyze aggregated summaries
cat /tmp/aggregated-*.md
```

### E3. Aggregation Report

**Lead creates aggregation report:**

```markdown
## Wave 1 Execution Summary

| Plan | Status | Files Created | Deviations | Commit |
|------|--------|---------------|-----------|--------|
| 01-01 | ✓ | 5 files | None | abc123d |
| 01-02 | ✓ | 3 files | 1 bug fixed | def456e |
| 01-03 | ⚠️ | 2 files | Blocked on auth setup | ghi789f |

**Total:** 2/3 complete, 1 checkpoint pending

**Next:** Get user auth, spawn continuation for 01-03, then Wave 2
```

---

## F. Comprehensive Message Schemas

### Standard Message Types

#### plan_complete
```json
{
  "type": "plan_complete",
  "plan_id": "01-01",
  "status": "success|failed|checkpoint",
  "branch": "feature/wave-1-plan-01",
  "worktree": "/path/to/worktree",
  "summary_path": ".planning/phases/01-foundation/01-01-SUMMARY.md",
  "commit_hash": "abc123d",
  "files_created": ["src/api/auth.ts", "src/types/user.ts"],
  "deviations": [
    { "rule": 1, "category": "Bug", "title": "Missing null check", "fix": "Added validation" }
  ]
}
```

#### research_complete
```json
{
  "type": "research_complete",
  "research_type": "stack|features|architecture|pitfalls",
  "phase": "1",
  "file": ".planning/research/STACK.md",
  "commit_hash": "xyz789a",
  "key_findings": ["Finding 1", "Finding 2"]
}
```

#### checkpoint_reached
```json
{
  "type": "checkpoint_reached",
  "plan_id": "03-02",
  "checkpoint_type": "human-verify|decision|human-action",
  "task_number": 5,
  "task_name": "Verify deployment",
  "checkpoint_file": ".planning/checkpoints/03-02-checkpoint.json",
  "progress": "4/5 tasks complete",
  "awaiting": "User verification"
}
```

#### checkpoint_response
```json
{
  "type": "checkpoint_response",
  "checkpoint_id": "03-02",
  "user_response": "approved|issues_found|skip",
  "issues_description": "Form doesn't submit (optional)",
  "next_action": "continue|retry|skip"
}
```

#### verification_failed
```json
{
  "type": "verification_failed",
  "phase": "1",
  "iteration": 1,
  "issues": ["Issue 1", "Issue 2"],
  "can_revise": true,
  "max_iterations": 3
}
```

#### revision_request
```json
{
  "type": "revision_request",
  "phase": "1",
  "iteration": 2,
  "issues": ["Issue 1", "Issue 2"],
  "guidance": "Optional guidance from user or lead"
}
```

### Message Routing Patterns

**Executor → Lead (Completion):**
- Type: `plan_complete` with status, files, deviations
- Metadata: branch, worktree, commit hash for verification

**Lead → Executor (Checkpoint Response):**
- Type: `checkpoint_response` with user decision
- Metadata: task number to resume from

**Researcher → Lead (Completion):**
- Type: `research_complete` with file path, key findings
- Metadata: commit hash for verification

**Checker → Lead (Verification):**
- Type: `verification_failed` with issue list
- Metadata: iteration count, whether revision is possible

**Lead → Planner (Revision):**
- Type: `revision_request` with issues from checker
- Metadata: iteration count, max iteration threshold

---

## G. Task Structure Examples

### Execute-Phase Task Tree
```
Task: Execute Phase 1
├─ Task: Execute Wave 1 (parallel)
│  ├─ Task: Execute plan 01-01 (blockedBy: none)
│  ├─ Task: Execute plan 01-02 (blockedBy: none)
│  └─ Task: Execute plan 01-03 (blockedBy: none)
├─ Task: Execute Wave 2 (parallel, blockedBy: Wave 1)
│  ├─ Task: Execute plan 01-04 (blockedBy: Wave 1)
│  └─ Task: Execute plan 01-05 (blockedBy: Wave 1)
└─ Task: Aggregate results (blockedBy: Wave 2)
```

### Plan-Phase Task Tree
```
Task: Plan Phase 1
├─ Task: Research (blockedBy: none)
├─ Task: Plan (blockedBy: Research)
├─ Task: Verify (blockedBy: Plan)
├─ Task: Revise (iteration 2, blockedBy: Verify)
├─ Task: Re-verify (iteration 2, blockedBy: Revise)
└─ (repeat revisions up to 3x)
```

### Research Phase Task Tree
```
Task: Research Phase 1
├─ Task: Stack research (blockedBy: none)
├─ Task: Features research (blockedBy: none)
├─ Task: Architecture research (blockedBy: none)
├─ Task: Pitfalls research (blockedBy: none)
└─ Task: Synthesize (blockedBy: all 4 researchers)
```

---

## H. Summary: Key Decisions

| Workflow | Pattern | Worktrees? | Message Type | Blocking Mechanism |
|----------|---------|-----------|--------------|-------------------|
| Execute-Phase | Wave-based executors | YES | `plan_complete` | TaskUpdate + TaskList poll |
| Research | 4 parallel + synthesizer | NO | `research_complete` | TaskUpdate + TaskList poll |
| Plan | Seq. researcher → planner → checker | NO | `verification_failed` → `revision_request` | TaskUpdate + blockedBy chain |
| Checkpoint | Executor pause → lead interaction | Depends | `checkpoint_reached` → `checkpoint_response` | File-based state + messages |
| Aggregation | Merge worktrees + read SUMMARY | YES | (implicit) | File existence + spot-checks |

---

## Recommendations for Implementation

1. **Start with Execute-Phase:** Highest value, clearest workflow
2. **Worktree naming:** `~/.claude/worktrees/{phase}-wave-{N}-plan-{ID}`
3. **Branch naming:** `feature/phase-{X}-wave-{N}-plan-{ID}`
4. **Checkpoint files:** Always written by executor before signaling
5. **Message field for paths:** Keep absolute paths (for subprocess isolation)
6. **Spot-check strategy:** File existence + git log grep + SUMMARY parsing
7. **Task hierarchy:** One task per executor/researcher, task dependency = wave/iteration barrier
