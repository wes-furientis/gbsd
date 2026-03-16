# Implementation Roadmap: GBSD Agent Teams Integration

**Status:** Ready for implementation
**Last Updated:** 2026-03-15
**Dependencies Resolved:** Worktree research (#2), Communications design (#4), Merge protocol (#5)

---

## Executive Summary

This roadmap details the phased implementation of Agent Teams for GSD, enabling:
- **Wave-based parallel execution** with isolated worktrees
- **Sequential pipeline workflows** (research → plan → verify)
- **Multi-researcher discovery** (4 parallel researchers + synthesizer)
- **Checkpoint-based user interaction** with file-based state persistence

**Key Insight from Research:** Lead orchestrator must create worktrees and pass paths to executor teammates. EnterWorktree/ExitWorktree tools available for agent isolation within worktrees.

**Timeline:** 4 phases, ~4-6 weeks of implementation

---

## Phase 1: Foundation (Week 1-1.5)

### Goal
Establish core infrastructure: worktree management, messaging patterns, task coordination.

### Tasks

#### 1.1 Create Worktree Manager Utility
**Objective:** Encapsulate worktree creation/cleanup, branch naming, path passing.

**Deliverables:**
- Bash script: `bin/gsd-worktree-manager.sh`
  - `create <phase> <plan> <wave>` → creates worktree, returns path
  - `list <phase>` → lists all worktrees for phase
  - `cleanup <phase> <wave>` → removes worktrees, deletes local branches
  - Branch naming: `gsd/exec/{phase}-{plan}-w{wave}`

**Implementation Notes:**
- Use `git worktree add` with computed paths
- Store worktree paths in `.planning/worktrees.json` for tracking
- Fail gracefully if worktree exists (allow resume)

**Files:**
- `/home/wes/.claude/get-shit-done/bin/gsd-worktree-manager.sh`
- `/home/wes/.claude/get-shit-done/bin/gsd-tools.cjs` (add worktree subcommands)

**Test:**
```bash
./gsd-worktree-manager.sh create 3 1 1
# Output: /home/wes/.claude/worktrees/gsd-exec-3-1-w1
# Branch: gsd/exec/3-1-w1
```

---

#### 1.2 Standardize Message Schemas
**Objective:** Define JSON schemas for all message types (documented in design-teams-comms.md).

**Deliverables:**
- JSON schema file: `.planning/schemas/messages.json`
  - `plan_complete` (executor → lead)
  - `research_complete` (researcher → lead)
  - `checkpoint_reached` (executor → lead)
  - `checkpoint_response` (lead → executor)
  - `verification_failed` (checker → lead)
  - `revision_request` (lead → planner)

**Implementation Notes:**
- Validate messages before SendMessage
- Provide type-checking helper function in Node/Bash

**Files:**
- `.planning/schemas/messages.json`

**Test:**
```bash
# Validate a message
node ~/.claude/get-shit-done/bin/validate-message.js < message.json
```

---

#### 1.3 Implement Checkpoint File Protocol
**Objective:** Standardize checkpoint state file format and reading.

**Deliverables:**
- Checkpoint file template: `.planning/templates/checkpoint.json`
- Utility to write checkpoint: `gsd-tools write-checkpoint {phase} {plan} {type} {details}`
- Utility to read checkpoint: `gsd-tools read-checkpoint {phase} {plan}`

**Implementation Notes:**
- Checkpoint files stored at: `.planning/checkpoints/{phase}-{plan}-checkpoint.json`
- Atomic writes (write to temp, then rename)
- Include schema validation

**Files:**
- `/home/wes/.claude/get-shit-done/templates/checkpoint.json`
- `bin/gsd-tools.cjs` (add checkpoint commands)

**Test:**
```bash
gsd-tools write-checkpoint 3 2 human-verify "{\...}"
cat .planning/checkpoints/3-2-checkpoint.json | jq .checkpoint_type
```

---

### Outcomes
- ✓ Worktree infrastructure ready
- ✓ Message schemas standardized
- ✓ Checkpoint protocol implemented
- ✓ All foundation utilities tested

---

## Phase 2: Execute-Phase Integration (Week 1.5-2.5)

### Goal
Implement wave-based parallel execution with worktrees and merge-back.

### Tasks

#### 2.1 Modify Execute-Phase Workflow
**Objective:** Update execute-phase.md to spawn executor teammates and coordinate waves.

**Deliverables:**
- Updated `execute-phase.md` section: "execute_waves" step
  - Replace inline execution with teammate spawning
  - Pass worktree path and branch to each executor teammate
  - Use TaskCreate for wave tasks with proper blockedBy chains
  - Poll TaskList to detect wave completion

**Key Changes:**
```markdown
<step name="execute_waves">
  For each wave:
    1. Create wave task group (all tasks in parallel, blocked by previous wave)
    2. For each plan:
       a. Create worktree: gsd-worktree-manager.sh create {phase} {plan} {wave}
       b. Spawn executor teammate with worktree path
       c. Create plan task, add to wave group
    3. Wait for all plan tasks to complete (TaskList polling)
    4. Merge worktree branches back (section 2.2)
    5. Spot-check merged code
    6. Proceed to next wave
```

**Implementation Notes:**
- Lead stays in orchestrator context (~15% usage)
- Executors get fresh 200k context each
- Use SendMessage for executor spawning, not inline Task (to avoid nesting)
- Handle interrupted executors (resume vs restart)

**Files:**
- `/home/wes/.claude/get-shit-done/workflows/execute-phase.md` (modified)

**Test:**
- Manual: `/gsd:execute-phase 1` on small 2-plan phase
- Verify: 2 worktrees created, 2 executor teammates spawned, tasks complete

---

#### 2.2 Implement Merge-Back in Execute-Phase
**Objective:** Call merge protocol after each wave completes.

**Deliverables:**
- Bash function: `merge_back_wave() { ... }`
  - Collect executor branches for wave
  - Call merge-back protocol (documented in merge-back-protocol.md)
  - Auto-resolve conflicts (lock files, dependencies, barrels)
  - Escalate source file conflicts
  - Run tests, cleanup branches on success

**Implementation Notes:**
- Source the merge-back protocol directly from workflow
- Integrate into execute-phase.md step: "execute_waves" → after step 3b

**Files:**
- Integration into `execute-phase.md`
- Bash implementation in `bin/gsd-execute-wave-merge.sh` (helper)

**Test:**
- 2-plan wave with independent files → merges cleanly
- 2-plan wave with lock file conflict → auto-resolves
- 2-plan wave with source file conflict → escalates to user

---

#### 2.3 Create Executor Teammate Prompts
**Objective:** Define prompts for executor teammates executing plans.

**Deliverables:**
- Prompt template: `.planning/templates/executor-prompt.md`
  - Input: plan path, worktree path, phase, plan ID
  - Output: SUMMARY.md, commits to branch
  - Include: deviation rules, TDD handling, checkpoint protocol
  - Pass: execute-plan.md context inline

**Implementation Notes:**
- Executor runs full execute-plan.md workflow
- Creates SUMMARY.md with self-check
- Commits work atomically per task
- Signals lead with checkpoint or completion message

**Files:**
- `.planning/templates/executor-prompt.md`

**Test:**
- Spawn executor for simple 2-task plan
- Verify: SUMMARY.md created, commits present, lead receives completion message

---

#### 2.4 Implement Executor Task Tracking
**Objective:** Track executor agents to enable resume/retry.

**Deliverables:**
- Agent tracking file: `.planning/agent-history.json`
  - Schema: array of {agent_id, task_description, phase, plan, status, timestamp}
  - Limit: 50 entries (prune oldest completed)
- Detection of interrupted executors (task spawned but never marked complete)
- Resume offer: "Executor for plan X was interrupted. Resume or start fresh?"

**Implementation Notes:**
- Write agent_id to `.planning/current-agent-id.txt` on spawn
- Delete on completion
- Check on init; offer resume if interrupted

**Files:**
- Implementation in `bin/gsd-tools.cjs` (agent-history commands)
- Integration into execute-phase.md init step

**Test:**
- Spawn executor, kill process
- Re-run execute-phase, verify interrupt detected and resume offered

---

### Outcomes
- ✓ Execute-phase spawns executor teammates
- ✓ Worktrees created per plan
- ✓ Waves execute in parallel
- ✓ Merge-back runs after each wave
- ✓ Conflicts auto-resolved or escalated
- ✓ Executor tracking and resume working

---

## Phase 3: Research & Planning Integration (Week 2.5-3.5)

### Goal
Implement sequential research → planning → verification pipeline.

### Tasks

#### 3.1 Implement Research Phase (4 Researchers)
**Objective:** Spawn 4 parallel researchers, block synthesizer.

**Deliverables:**
- Modified `plan-phase.md` step 5 (Handle Research)
  - Create 4 independent research tasks (no blockedBy)
  - Create synthesizer task (blockedBy all 4)
  - Spawn 4 researcher teammates (messages, not Tasks)
  - Wait for all 4 to complete
  - Spawn synthesizer

**Key Changes:**
```markdown
<step name="handle_research">
  If research needed:
    1. Create 4 research tasks (independent)
    2. Spawn 4 researcher teammates:
       - researcher-stack → STACK.md
       - researcher-features → FEATURES.md
       - researcher-architecture → ARCHITECTURE.md
       - researcher-pitfalls → PITFALLS.md
    3. Each messages lead on completion
    4. TaskUpdate to mark tasks complete
    5. When all 4 complete, spawn synthesizer
    6. Synthesizer creates SUMMARY.md
```

**Implementation Notes:**
- No worktrees needed (different files, no conflicts)
- Standard git workflow (each researcher commits their file)
- Use TaskList polling to detect when all 4 complete

**Files:**
- `.planning/templates/researcher-prompts.md` (4 prompts)
- Modified `plan-phase.md`

**Test:**
- `/gsd:plan-phase 1 --research`
- Verify: 4 research tasks created, researchers spawned, synthesizer blocks until all 4 complete

---

#### 3.2 Implement Planning Task
**Objective:** Planner teammate reads research, writes PLAN.md files.

**Deliverables:**
- Planner prompt template: `.planning/templates/planner-prompt.md`
  - Input: RESEARCH.md (if exists), CONTEXT.md, REQUIREMENTS.md, STATE.md
  - Output: PLAN-*.md files in phase directory
  - Include: frontmatter (wave, dependencies, files_modified, requirements)
  - Commit and signal lead

**Implementation Notes:**
- Planner creates N PLAN.md files
- Each plan has tasks with TDD/checkpoint annotations
- Frontmatter: files_modified for conflict detection during planning

**Files:**
- `.planning/templates/planner-prompt.md`

**Test:**
- Spawn planner with research + requirements
- Verify: 3+ PLAN.md files created, frontmatter valid, signal sent

---

#### 3.3 Implement Verification Loop (Max 3 Iterations)
**Objective:** Checker verifies plans, revision loop on issues.

**Deliverables:**
- Checker prompt template: `.planning/templates/checker-prompt.md`
  - Input: PLAN-*.md files, REQUIREMENTS.md, CONTEXT.md, RESEARCH.md
  - Output: VERIFICATION PASSED or ISSUES FOUND
- Revision logic in plan-phase.md step 12
  - On issues: spawn planner with revision context
  - Loop up to 3 times
  - Max iterations: offer user options (force proceed / abandon / get guidance)

**Implementation Notes:**
- Checker reads all PLAN files
- Verifies: requirement coverage, goal alignment, feasibility
- Returns structured issue list (plan ID, issue, severity)

**Files:**
- `.planning/templates/checker-prompt.md`
- Modified `plan-phase.md` step 12 (revision loop)

**Test:**
- Spawn checker on intentionally flawed plan
- Verify: issues found, planner spawned for revision, loop runs

---

#### 3.4 Integrate Plan-Phase Pipeline with Teams
**Objective:** Chain researcher → planner → checker using tasks + messaging.

**Deliverables:**
- Modified `plan-phase.md`:
  - Step 5 (research): spawn 4 researchers, synthesizer
  - Step 8 (plan): spawn planner (blocked by research)
  - Step 10 (verify): spawn checker (blocked by planner)
  - Step 12 (revise): loop back to planner if issues (blocked by checker)

**Implementation Notes:**
- Use TaskCreate for each stage
- Each stage blocked by previous (blockedBy chains)
- Messaging between lead and teammates for handoffs
- No worktrees (single main branch)

**Files:**
- Modified `plan-phase.md`

**Test:**
- `/gsd:plan-phase 1` end-to-end
- Verify: research completes, planner spawned, checker runs, revision loop works

---

### Outcomes
- ✓ 4 researchers spawn in parallel
- ✓ Synthesizer waits for all 4
- ✓ Planner reads research, writes plans
- ✓ Checker verifies plans
- ✓ Revision loop (max 3x) on issues
- ✓ Full pipeline: research → plan → verify → done

---

## Phase 4: Checkpoint & Polish (Week 3.5-4.5)

### Goal
Complete checkpoint handling, edge cases, testing.

### Tasks

#### 4.1 Implement Checkpoint Handling in Executor
**Objective:** Executors write checkpoint files, lead reads and responds.

**Deliverables:**
- Checkpoint writing: gsd-tools write-checkpoint {phase} {plan} {type} {details}
- Executor prompts updated to call gsd-tools on checkpoint
- Lead messaging: read checkpoint file, present to user, get response
- Executor continuation: fresh agent spawned with resume state

**Implementation Notes:**
- Checkpoint types: human-verify, decision, human-action
- Write to `.planning/checkpoints/{phase}-{plan}-checkpoint.json`
- Lead reads file, presents checkpoint box to user
- Message executor with response (or spawn continuation)

**Files:**
- Modified executor prompt template
- Modified execute-phase.md (checkpoint handling section)

**Test:**
- Executor hits human-verify checkpoint
- Lead reads checkpoint file, presents to user
- Lead messages executor, executor continues or lead spawns continuation

---

#### 4.2 Handle Interrupted Executors
**Objective:** Resume functionality for failed/interrupted executor agents.

**Deliverables:**
- Detect interrupted executors: `.planning/current-agent-id.txt` exists
- Offer options: "Resume {plan} or start fresh?"
- Resume: Task `resume: true` parameter
- Fresh start: cleanup worktree, create new one

**Implementation Notes:**
- Check on execute-phase init
- Preserve worktree if resuming
- Validate SUMMARY.md exists before marking complete

**Files:**
- Implementation in execute-phase.md init step
- Bash helper: `bin/gsd-detect-interrupted-executor.sh`

**Test:**
- Simulate interrupted executor (kill subprocess)
- Re-run execute-phase, verify interrupt detected, resume offered

---

#### 4.3 Merge Conflict Resolution Automation
**Objective:** Auto-resolve 80%+ of conflicts (lock files, dependencies, barrels).

**Deliverables:**
- Bash script: `bin/gsd-merge-conflicts-auto-resolve.sh`
  - Detect conflict file types
  - Lock files: regenerate (npm install, uv pip compile)
  - Dependency files: union, regenerate lock
  - Barrel/index files: union exports
  - Config files: take latest
  - Escalate source files to user

**Implementation Notes:**
- Called from merge-back protocol
- Log all auto-resolved conflicts
- Return list of unresolved conflicts for user

**Files:**
- `bin/gsd-merge-conflicts-auto-resolve.sh`
- Integration into merge-back-protocol.md section 4

**Test:**
- 2 executors add different deps → auto-resolves package.json and lock
- 2 executors modify same source function → escalates to user

---

#### 4.4 Result Aggregation Report
**Objective:** Lead generates summary of phase execution.

**Deliverables:**
- After all waves merge: generate aggregation report
  - Per-plan summary (status, files, deviations, commit hash)
  - Total stats (plans complete, issues, deviations)
  - Next steps (execute next phase / verify / abort)

**Implementation Notes:**
- Read SUMMARY.md from each plan (post-merge)
- Read ROADMAP.md for wave info
- Create `.planning/phases/{phase}/AGGREGATION.md`

**Files:**
- Bash function: `aggregate_phase_results() { ... }`
- Integration into execute-phase.md step "aggregate_results"

**Test:**
- 2-wave phase execution completes
- Verify aggregation report created, contains all plan data

---

#### 4.5 Testing & Documentation
**Objective:** Full test coverage, user docs, troubleshooting guide.

**Deliverables:**
- Test suite: `tests/teams-integration.test.sh`
  - Unit: worktree manager, message schemas, checkpoint files
  - Integration: execute-phase with 2 parallel plans, merge-back
  - End-to-end: full pipeline from new-project → execute-phase → verify
- User guide: `.planning/TEAMS-USER-GUIDE.md`
  - How to use Agent Teams
  - When parallel execution is appropriate
  - How to monitor executors
  - How to resume interrupted executors
  - How to resolve merge conflicts
- Troubleshooting: `.planning/TEAMS-TROUBLESHOOTING.md`
  - Executor didn't complete
  - Merge conflict can't auto-resolve
  - Checkpoint not reaching user
  - Task dependency deadlock

**Files:**
- `tests/teams-integration.test.sh`
- `.planning/TEAMS-USER-GUIDE.md`
- `.planning/TEAMS-TROUBLESHOOTING.md`

**Test:**
- Run full test suite: all tests pass
- Docs reviewed by user

---

### Outcomes
- ✓ Checkpoints fully functional
- ✓ Interrupted executor recovery working
- ✓ Merge conflicts auto-resolved
- ✓ Aggregation reports generated
- ✓ Full test coverage
- ✓ User documentation complete

---

## Phase 5: Deployment & Iteration (Week 4.5-6)

### Goal
Deploy to GBSD, gather feedback, iterate.

### Tasks

#### 5.1 Deploy to GBSD
**Objective:** Roll out Agent Teams integration to all GSD workflows.

**Deliverables:**
- All Phase 1-4 artifacts committed and tested
- Workflows updated: execute-phase.md, plan-phase.md, new-project.md
- Utilities integrated: gsd-tools.cjs, bin/ scripts
- Configuration: `.planning/config.json` with `workflow.agent_teams` flag (opt-in initially)

**Implementation Notes:**
- Opt-in via config: `"agent_teams": true` to spawn teammate executors
- Default (false) runs inline (existing behavior)
- User can toggle anytime

**Test:**
- Deploy to staging environment
- `/gsd:new-project` with teams enabled
- Execute phase with 3 parallel plans
- Verify all workflows functional

---

#### 5.2 Collect User Feedback
**Objective:** Identify pain points, unexpected behaviors.

**Deliverables:**
- Feedback template: `.planning/TEAMS-FEEDBACK.md`
- Log: what worked, what broke, edge cases found
- Iteration backlog: issues to address in next phase

---

#### 5.3 Iterate on Feedback
**Objective:** Fix issues, optimize performance.

**Deliverables:**
- Address top 5 user issues
- Performance optimization (e.g., worktree cleanup, message batching)
- Enhanced error messages
- Updated docs

---

### Outcomes
- ✓ Agent Teams deployed and functional
- ✓ User feedback collected
- ✓ Iteration backlog created
- ✓ Ready for production use

---

## Risk Mitigation & Checkpoints

### Critical Risks

| Risk | Mitigation |
|------|-----------|
| Executor context not fresh enough (plan too large) | Cap plan size; split large plans into smaller ones |
| Merge conflicts block wave completion | Auto-resolve 80%+ of conflicts; escalate complex ones early |
| Checkpoint file not reaching user | Test messaging path; add fallback log file in .planning/checkpoints/ |
| Task dependency deadlock | Validate task graph before spawning; use TaskList to detect stuck tasks |
| Worktree cleanup fails, leaves stale dirs | Implement cleanup verification; add manual cleanup command |

### Phase Checkpoints

**Before Phase 2:** All Phase 1 utilities tested
**Before Phase 3:** Execute-phase spawning executors successfully
**Before Phase 4:** Research → plan → verify pipeline working
**Before Phase 5:** All edge cases tested, docs complete

---

## Success Criteria

- **Execute-Phase:** 3+ plans in parallel per wave, all merge cleanly, SUMMARY files aggregated
- **Research Phase:** 4 researchers finish in parallel, synthesizer produces SUMMARY.md
- **Plan-Phase:** Planner reads research, checker verifies, revision loop (max 3x) works
- **Checkpoints:** User receives checkpoint, responds, executor continues or fresh agent spawns
- **Documentation:** User guide complete, troubleshooting addresses top issues
- **Testing:** 100% coverage of core workflows, 90% of edge cases

---

## Timeline Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| 1 | Week 1-1.5 | Worktree manager, message schemas, checkpoint protocol |
| 2 | Week 1.5-2.5 | Execute-phase integration, merge-back, executor prompts |
| 3 | Week 2.5-3.5 | Research (4 parallel), planner, checker, revision loop |
| 4 | Week 3.5-4.5 | Checkpoints, interrupt handling, conflict resolution, testing |
| 5 | Week 4.5-6 | Deployment, feedback, iteration |

**Total: ~4-6 weeks**

---

## File Structure After Implementation

```
.planning/
├── config.json                          # "agent_teams": true/false
├── templates/
│   ├── checkpoint.json
│   ├── executor-prompt.md
│   ├── researcher-prompts.md
│   ├── planner-prompt.md
│   ├── checker-prompt.md
│   └── merge-back-helpers.sh
├── schemas/
│   └── messages.json
├── checkpoints/                         # Created at runtime
│   └── {phase}-{plan}-checkpoint.json
├── worktrees.json                       # Track worktrees
├── agent-history.json                   # Track executors
├── TEAMS-USER-GUIDE.md
├── TEAMS-TROUBLESHOOTING.md
└── TEAMS-FEEDBACK.md

bin/
├── gsd-worktree-manager.sh
├── gsd-merge-conflicts-auto-resolve.sh
├── gsd-detect-interrupted-executor.sh
└── gsd-tools.cjs (extended)

workflows/
├── execute-phase.md                     # Modified
├── plan-phase.md                        # Modified
├── merge-back-protocol.md               # Already exists
└── new-project.md                       # Minor updates

tests/
└── teams-integration.test.sh

design/
├── design-teams-comms.md                # Already exists
└── implementation-roadmap-agent-teams.md (this file)
```

---

## Next Steps

1. **Review & Approval:** Stakeholder review of this roadmap
2. **Phase 1 Kickoff:** Create worktree manager, message schemas, checkpoint protocol
3. **Weekly Sync:** Status updates on phase progress, blockers
4. **User Feedback Loop:** Continuous iteration based on user experience

---

## Appendix: Key Design Decisions

### Decision 1: Lead Creates Worktrees, Passes Paths
**Rationale:** Teammates cannot reliably create isolated worktrees without lead orchestration. Lead is already coordinating; centralized worktree creation is safer.

**Impact:** Lead context ~15% for orchestration; executors get fresh 200k each.

### Decision 2: File-Based Checkpoint State
**Rationale:** JSON files are simpler than in-memory state and persist across agent boundaries. Lead can read and present checkpoint details without re-running executor.

**Impact:** Checkpoint files live in `.planning/checkpoints/`; lead polls TaskList to detect checkpoint condition.

### Decision 3: Auto-Resolve 80% of Conflicts, Escalate 20%
**Rationale:** Most conflicts (lock files, dependencies, barrels) have deterministic resolutions. Source file conflicts require semantic understanding; escalate to user.

**Impact:** Merge-back usually succeeds without user intervention. Complex merges blocked until user intervenes.

### Decision 4: Tasks with Explicit BlockedBy for Waves
**Rationale:** Task dependency chains make wave synchronization explicit. No polling loops; TaskList tells lead when wave is complete.

**Impact:** Clean task hierarchy; lead knows when to trigger next wave without manual checks.

### Decision 5: Regular Merge Commits (No Squash/Rebase)
**Rationale:** Preserves atomic per-task commits, maintains clear history for blame/debugging, safer for shared repos.

**Impact:** Merge commits included in final history; executor work traceable per task.

---

## References

- Design: `/home/wes/.claude/design-teams-comms.md`
- Merge Protocol: `/home/wes/.claude/get-shit-done/workflows/merge-back-protocol.md`
- Worktree Research: Task #2 findings
- Core Workflows: execute-phase.md, plan-phase.md, new-project.md
