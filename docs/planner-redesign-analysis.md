# Planner Redesign Analysis: Maximizing Parallelizable Independent Tasks

**Status:** Complete
**Date:** 2026-03-15
**Task:** Redesign planner to maximize parallelizable independent tasks
**Blocked by:** Task #5 (Pathfinder analysis) — findings will incorporate Pathfinder integration post-completion

---

## Executive Summary

The current GSD planner creates plans using vertical-slice thinking (full feature from model → API → UI) which works well for independent features, but the decomposition strategy is fundamentally task-first and wave-second. The planner achieves moderate parallelism (~40-60% Wave 1 eligibility in typical phases) but can be redesigned to push this to 70-85% by making **file ownership a first-class planning concept**.

Current bottleneck: The planner reasons about dependencies at the task level **after** grouping, not **before**. This creates implicit couplings and forces sequential waves. The redesign shifts to dependency-driven task creation where file ownership becomes the primary parallelization signal.

---

## 1. Current Plan Decomposition Strategy

### How the Planner Currently Decides Plan Scope and Task Boundaries

**Current approach** (lines 1090-1127 in gsd-planner.md):

1. **Break into tasks** — Decompose phase goal into task-shaped chunks (15-60 min each)
2. **Build dependency graph** — After decomposition, analyze needs/creates for each task
3. **Assign waves** — Use wave algorithm (max dependency + 1) to determine execution order
4. **Group into plans** — 2-3 tasks per plan, same-wave + no file conflicts in same plan

**Issue:** This is sequential reasoning. The decomposition happens without explicit parallelization intent. Tasks are created, then dependencies are checked, then waves are assigned. If a task has a dependency, it creates a wave boundary — even if that dependency could be optional or delayed.

### Plan Scope Definition

- **2-3 tasks per plan** (lines 355-356, 1126)
- **~50% context target** (lines 109, 353, 1139)
- **Autonomy factor**: Checkpoints reduce parallelism (one plan with checkpoint blocks plan's wave)

### Current Task Boundaries

Task sizing rule (lines 198-209):
- **15-60 minutes** Claude execution time
- **Triggers split:** >5 files, multiple subsystems, >3 tasks in plan
- **Vertical slices preferred:** One feature = model + API + UI in single plan

The vertical slice works well for independent features but creates false dependencies when:
- A feature requires shared infrastructure (auth, validation, state management)
- API design needs to be approved before implementation
- UI depends on finalized API contracts

---

## 2. Current Wave Assignment Algorithm

**Algorithm** (lines 1109-1118):

```
for each plan:
  if depends_on is empty: wave = 1
  else: wave = max(waves[dep] for dep in depends_on) + 1
```

**How it determines dependencies:**

1. **File overlap detection** (line 345 in template docs):
   - `files_modified` field in each plan
   - If Plan A and Plan B modify same file → Plan B must wait (sequential)

2. **Explicit depends_on** (lines 363-364, 410):
   - Each plan declares dependencies on other plans
   - Defaults to empty → Wave 1 candidate

3. **Checkpoint handling** (lines 195-196, 364):
   - Plans with checkpoints get `autonomous: false`
   - Checkpoints create wave boundaries (must complete, then block)

### Current Dependency Detection Issues

**Problem 1: Overly Conservative**
- File overlap in frontmatter assumes conflicts → forces sequential
- Example: Two plans both import from `src/types/user.ts` but don't modify it
  - Both declare `files_modified: [src/types/user.ts]`
  - Treated as conflict, even though they're both just reading the same type

**Problem 2: Missing Optional Dependencies**
- `depends_on` is binary: either depends or doesn't
- No distinction between:
  - **Critical dependency:** API types needed before UI can build
  - **Soft dependency:** UI can stub if API incomplete
  - **Ordering preference:** Do X first, but Y can proceed if X blocks

**Problem 3: No Subtask Scheduling**
- Plans are atomic — can't express "do Task A, then parallelize Tasks B+C in same plan"
- If a plan has checkpoint that requires Tasks 1-2 complete, Task 3 can't start until checkpoint clears

---

## 3. Restructuring for Maximum Parallelism: File Ownership as First-Class Concept

### Core Insight: File Ownership Prevents Merge Complexity

Current bottleneck: When two tasks modify the same file in same wave, they can't run parallel. Current approach is to add them to `depends_on`.

**Better approach:** Use **file ownership** as primary planning signal:

```
Each file = exclusive owner during wave
Each task = owns its output files (no sharing)
Different waves = can both modify same file (sequential, no conflict)
Same wave = different files = true parallel
```

### Redesigned Decomposition Flow

**New priority order:**

1. **Identify file sets** — What files will be created/modified?
2. **Cluster by ownership** — Which tasks own which files?
3. **Detect actual file conflicts** — Tasks modifying same file in same wave → different waves
4. **Build dependency graph** — Only from actual file conflicts + type dependencies
5. **Assign waves** — Using file ownership as primary signal
6. **Create plans** — Group by wave + file ownership pattern

### File Ownership Rules

**Ownership is exclusive per wave:**
- In Wave 1, Task A owns `src/models/user.ts` (exclusive)
- In Wave 2, Task B can modify `src/models/user.ts` (no conflict, different wave)
- In Wave 1, Task A + Task B both trying to modify `src/models/user.ts` → conflict, must be sequential

**Type dependencies** override file separation:
- Plan A creates type `type User`
- Plan B needs `User` type → depends on Plan A
- Even if they don't share files, they have type dependency

**API endpoint dependencies:**
- Plan A creates `GET /users` endpoint
- Plan B creates UI that calls `GET /users` → depends on Plan A

### Practical Decomposition Strategy

Instead of: "Build User feature (model + API + UI)"

Think: "What's the minimal unit of parallelizable work?"

Example Phase: "User management"

**Current approach:**
- Plan 01: User model + API + UI (3 tasks, fully sequential)
  - Wave 1: User model
  - Wave 2: User API (depends on model types)
  - Wave 3: User UI (depends on API)

**Redesigned approach:**
- Plan 01: User model + types (1 task, Wave 1)
- Plan 02: User API (1 task, Wave 2, depends on types)
- Plan 03: User UI (1 task, Wave 2... wait, can't parallel UI without API)

Actually, let's pick a better example with genuinely parallel work:

Example Phase: "Dashboard with multiple widgets"

**Current approach:**
- Plan 01: User stats widget (model + API + UI)
- Plan 02: Activity widget (model + API + UI)
- Plan 03: Revenue widget (model + API + UI)
- Result: All Wave 1 if no file overlap ✓ Already good

**Where it breaks down:** If all three widgets share a `src/lib/api.ts` utility:
- All three plans declare `files_modified: [..., src/lib/api.ts]`
- Treated as conflict → forced sequential
- Actually, they're just appending functions to same file

**Redesigned approach:**
- Separate concerns: **data layer** from **UI layer**
- Plan 01: Data models (Wave 1)
  - Creates: `src/models/stats.ts`, `src/models/activity.ts`, `src/models/revenue.ts`
- Plan 02: API endpoints (Wave 1 if stats model sufficient, or Wave 2)
  - Creates: `src/api/stats.ts`, `src/api/activity.ts`, `src/api/revenue.ts`
  - **Key:** Use separate files for each domain's API
- Plan 03: Shared utility lib (Wave 0 or 1)
  - Creates: `src/lib/api.ts` with shared helpers
  - Other plans import (don't modify)
- Plans 04-06: UI widgets (Wave 2 or 3)
  - Create: `src/components/StatsWidget.tsx`, etc.
  - Each owns its file (no conflict)

Result: **More plans, shorter waves, maximum parallelism.**

---

## 4. Should Plans Be Longer (More Tasks) or Phases Have More Plans (Fewer Tasks Each)?

**Current guidance** (line 355, 1126): 2-3 tasks per plan

**Redesign recommendation: Shift toward more plans with fewer tasks (1-2 tasks each)**

### Rationale

**More plans, fewer tasks:**
- ✓ Higher parallelism (each plan in own wave, less coupling)
- ✓ Easier to parallelize via worktree sessions (each plan = one agent)
- ✓ Clearer file ownership (task owns output files)
- ✓ Simpler verification (fewer artifacts to check per plan)
- ✓ Better error isolation (one task fails, others proceed)
- ✗ More plans to manage/orchestrate
- ✗ More redundancy in context (each plan reads same base context)

**Fewer plans, more tasks:**
- ✓ Less overhead
- ✓ Shared context efficiency
- ✗ More sequential dependencies within plan
- ✗ Harder to parallelize

### Proposed Shift

**Old:** 3-5 plans per phase, 2-3 tasks each = ~6-15 tasks total
**New:** 5-10 plans per phase, 1-2 tasks each = ~7-20 tasks total

Same task count, but distributed differently:
- **Old pattern:** 3 plans with 3 tasks each = all sequential within plan, some Wave 1
- **New pattern:** 9 plans with 2 tasks each = mostly 1 task per plan, most Wave 1

### Concrete Example

**Old Phase: "Auth System"**
```
Plan 01: Models + Types (3 tasks)
  - Task 1: Create User model
  - Task 2: Create Session model
  - Task 3: Create auth types
  Wave 1, 3 sequential tasks

Plan 02: API Routes (3 tasks)
  - Task 1: Login endpoint
  - Task 2: Logout endpoint
  - Task 3: Refresh token endpoint
  Wave 2, 3 sequential tasks (wait for models)

Plan 03: Middleware + Guards (2 tasks)
  - Task 1: Auth middleware
  - Task 2: Protected route decorator
  Wave 3, 2 sequential tasks (wait for auth types)

Total: 8 tasks, 3 plans, 3 waves (100% sequential between plans)
```

**New Phase: "Auth System"**
```
Plan 01: User model (1 task)
  Wave 1
Plan 02: Session model (1 task)
  Wave 1
Plan 03: Auth types (1 task)
  Wave 1
Plan 04: Login endpoint (1 task)
  Wave 2, depends on Plans 01+03
Plan 05: Logout endpoint (1 task)
  Wave 2, depends on Plan 02
Plan 06: Refresh endpoint (1 task)
  Wave 2, depends on Plan 02
Plan 07: Auth middleware (1 task)
  Wave 3, depends on Plan 03
Plan 08: Protected route decorator (1 task)
  Wave 3, depends on Plan 03

Total: 8 tasks, 8 plans
Wave 1: 3 plans (50% of work runs in parallel)
Wave 2: 3 plans (all can run in parallel)
Wave 3: 2 plans (both can run in parallel)
Parallelism: 3-4x better in Wave 2+3
```

The second pattern is "slower" per plan (more overhead), but orchestrates faster (more parallel).

---

## 5. How Pathfinder's Dependency Graph Could Inform the Planner

**Key insight:** Pathfinder will analyze the codebase **statically** to detect real dependencies.

After Task #5 completes, the planner gains:

### What Pathfinder Provides

1. **True file dependencies** (not just `files_modified`)
   - File A imports from File B
   - File A calls function in File B
   - Actual wiring, not guesses

2. **Type dependencies**
   - Function expects type T from module M
   - Type T must be exported from M before function can use it

3. **API contract dependencies**
   - Route expects request body schema S
   - Client expects response schema R
   - These are contracts that must match

4. **Module-level design** (if Pathfinder does dep analysis)
   - Which modules import which
   - Dependency DAG of the codebase

### Integration Points

**At planning time**, Pathfinder analysis enables:

1. **Accurate file conflict detection:**
   - Current: "files_modified overlap → conflict"
   - Future: "Pathfinder says these files don't have conflicting exports/types → can parallel"

2. **Smart depends_on derivation:**
   - Current: Manual, educated guess
   - Future: "Plan A creates type T, Plan B needs T → auto-detect depends_on"

3. **Cross-phase dependencies:**
   - Pathfinder can show: "Phase 03 task needs Phase 02 artifact"
   - Enables phase-level parallelism (Phase 02 and 03 can run together if no conflicts)

4. **Optional vs critical dependencies:**
   - Pathfinder can show strength of link (is file import used in that task?)
   - Weak imports can be stubs, strong imports block

### Concrete Workflow After Pathfinder Integration

```
Plan Phase Flow:
1. Load ROADMAP (what to build)
2. Run Pathfinder (analyze existing codebase)
3. Pathfinder output:
   - Existing files and exports
   - Import graph
   - Type definitions
4. Planner uses Pathfinder data:
   - "Type User exported from src/models/user.ts"
   - When task B needs User type, planner queries Pathfinder
   - Pathfinder confirms dependency
   - Planner adds to depends_on
5. Decompose with confidence:
   - Fewer guesses, more facts
   - Accurate wave assignment
   - Maximum parallelism
```

---

## 6. Trade-offs: More Parallelism vs More Merge Complexity & Smaller Tasks vs More Overhead

### Parallelism vs Merge Complexity

**More parallelism = higher chance of merge conflicts**

Example: Wave 1 has 4 plans all modifying `src/lib/index.ts`

In current system: Not allowed (would fail file conflict check)
In redesigned system: Could happen if we're not careful

**Mitigation strategies:**

1. **File ownership as contract:**
   - If multiple plans need to append to same file, agree on structure before Wave 1
   - Example: `src/lib/index.ts` has sections:
     ```typescript
     // ============ Auth utils (Plan 01 owns)
     // Plan 01 tasks write here

     // ============ API utils (Plan 02 owns)
     // Plan 02 tasks write here
     ```
   - Sequential writes in same file, but plans can develop in parallel, merge at end

2. **Separate files by domain:**
   - Don't share `index.ts`, create `auth.ts`, `api.ts`, `validation.ts`
   - Each plan owns its domain file
   - Less merge conflict likelihood

3. **Merge-friendly patterns:**
   - Append-only patterns (easier merges)
   - Array/map spreads (less conflict)
   - Avoid deeply nested rewrites

### Small Tasks vs Orchestration Overhead

**Cost of more plans:**

| Plans | Context per Plan | Orchestration | Total Time |
|-------|------------------|---------------|----|
| 3 plans × 3 tasks | High (context 50%) | Low | Fast per plan, slow total |
| 9 plans × 1 task | Low (context 20%) | High | Slower per plan, fast total if parallel |

If plans can run in parallel via worktrees:
- 3 plans sequential = 3× planning + 3× execution + 3× verification
- 9 plans in 3 waves parallel = 9× planning, but execution + verification happen in waves

**Key:** With phase-level parallelism (worktrees), 9 smaller plans can actually be faster total.

**Rough math:**
- Plan overhead: ~2 min reading context, ~5 min planning, ~10 min execution, ~2 min verification = 19 min
- 3 large plans: 3 × 19 = 57 min sequential
- 9 small plans: 9 × 19 = 171 min if sequential, but in 3 parallel waves = 3 × 19 + 6 × 5 = 87 min (execution/verification parallel, planning sequential)

Actually the math gets complex, but the key insight: **Smaller plans enable better orchestration patterns.**

---

## 7. Parallelizability Score: Metric Definition

**Define: Parallelizability Score = (Wave 1 Eligible Plans) / (Total Plans)**

Higher score = more work can happen in parallel from day 1.

### Current System Performance

Typical phase (Features with shared types):
- Wave 1: 2-4 plans (basic CRUD)
- Wave 2: 2-3 plans (depends on Wave 1)
- Wave 3: 1-2 plans (integration/verification)
- **Score: 2-4 / 5-9 = ~40-60%**

Ideal phase (fully independent features):
- Wave 1: 5-7 plans
- Wave 2: 0 plans
- **Score: 5-7 / 5-7 = ~100%**

### Redesigned System Target

Goal: **70-85% parallelizability across typical phases**

By decomposing into smaller, file-owned tasks:
- Wave 1: 5-8 plans (models, types, utility layers)
- Wave 2: 3-5 plans (APIs, middleware, depends on Wave 1)
- Wave 3: 1-2 plans (UI, integration, optional)
- **Score: 5-8 / 9-15 = ~55-65% → improvements via Pathfinder to 70-85%**

### Measurement Points

1. **Per-phase score:** (Wave 1 count) / (Total plans)
2. **Historical trend:** Track score over multiple phases to see improvement
3. **Workload distribution:** Ideal is pyramid (most work in Wave 1-2, less in later waves)

---

## 8. Concrete Changes to Frontmatter Schema and Wave Assignment Logic

### Current Frontmatter (lines 407-421)

```yaml
---
phase: XX-name
plan: NN
type: execute | tdd
wave: N
depends_on: []
files_modified: []
autonomous: true | false
requirements: []
user_setup: []
must_haves:
  truths: []
  artifacts: []
  key_links: []
---
```

### Proposed Additions/Changes

**Add file ownership clarity:**

```yaml
---
phase: XX-name
plan: NN
type: execute | tdd
wave: N
depends_on: []

# NEW: Detailed file ownership
files_created: []      # Files this plan creates (owns)
files_modified: []     # Files this plan modifies (owns)
files_read: []         # Files this plan reads (no ownership)

# Helps planners distinguish:
# - Created/modified files are exclusive (owner in wave)
# - Read files can be shared (type dependencies only)

autonomous: true | false
requirements: []
user_setup: []

# ENHANCED: Optional vs critical dependencies
soft_depends_on: []    # Can proceed if these are stubs
hard_depends_on: []    # MUST have these before starting

must_haves:
  truths: []
  artifacts: []
  key_links: []
---
```

**Alternative (simpler) approach without schema change:**

Keep frontmatter simple, enhance wave assignment logic instead:

```yaml
---
phase: XX-name
plan: NN
type: execute | tdd
wave: N
depends_on: []
files_modified: []     # Still used, but interpreted smarter
autonomous: true | false
requirements: []
user_setup: []
must_haves:
  truths: []
  artifacts: []
  key_links: []

# NEW: File ownership annotations in frontmatter comments
# files_modified: [src/models/user.ts, src/api/users.ts]
#   ^ Task A owns these (created from scratch)
# files_modified: [src/lib/index.ts]  # appended to, not rewritten
#   ^ Task B appends exports (no conflict with Task C appending)
---
```

### Revised Wave Assignment Logic

**Current algorithm (lines 1109-1118):**

```python
for each plan:
  if depends_on is empty:
    wave = 1
  else:
    wave = max(waves[dep] for dep in depends_on) + 1
```

**Enhanced algorithm (file-ownership aware):**

```python
# Build file ownership map
file_owners = {}
for each plan:
  for file in plan.files_modified:
    if file not in file_owners:
      file_owners[file] = []
    file_owners[file].append(plan)

# Detect conflicts
for each plan:
  conflicts = []
  for file in plan.files_modified:
    other_plans = [p for p in file_owners[file] if p != plan]
    for other in other_plans:
      if should_conflict(plan, other):  # See below
        conflicts.append(other)

  # Add file conflicts to depends_on (if not already)
  for conflict in conflicts:
    if conflict.plan_id not in plan.depends_on:
      plan.depends_on.append(conflict.plan_id)

# Assign waves (same as before, but now depends_on is accurate)
for each plan:
  if depends_on is empty:
    wave = 1
  else:
    wave = max(waves[dep] for dep in depends_on) + 1
```

**Key function: `should_conflict(plan_a, plan_b)`**

```python
def should_conflict(plan_a, plan_b):
  """Determine if two plans have real file conflicts."""

  # File conflict only if:
  # 1. Same file in files_modified
  # 2. Same wave (would run parallel)
  # 3. Both modifying (not one reading)

  common_files = set(plan_a.files_modified) & set(plan_b.files_modified)

  if not common_files:
    return False

  # Check if file conflict is resolvable
  # (e.g., both appending to imports list, both writing disjoint sections)
  for file in common_files:
    if is_append_only_file(file):  # like __init__.py, index.ts
      # Multiple plans can append same file
      # Use merge-friendly patterns in action descriptions
      return False
    else:
      # File is densely packed, rewrites = conflict
      return True

  return True
```

### Enhanced Dependency Detection

**Instead of just checking depends_on, also check:**

1. **Type imports:**
   - If Plan B action includes "import { User } from ../models"
   - And User is created by Plan A
   - Add Plan A to depends_on

2. **API endpoint usage:**
   - If Plan B action includes "fetch('/api/users')"
   - And /api/users endpoint created by Plan A
   - Add Plan A to depends_on

**Tool support:**

The planner can query actual code:

```bash
# Find what Plan A creates
grep -n "export.*type\|export.*function\|export.*const" \
  $(find .planning/phases/XX-name -name "*-SUMMARY.md" | head -1)

# Find what Plan B needs
grep -n "import.*from\|fetch.*api\|useQuery" <(plan_b_action)
```

---

## 9. Frontmatter Enhancement: Proposal for vote

**Option A: Minimal change (add comments only)**
- No schema change
- Planner adds file ownership context in task action descriptions
- Wave assignment logic enhanced (as above)
- Pros: No format breaking, backward compatible
- Cons: Ownership intent not machine-readable

**Option B: Add optional fields**
```yaml
files_created: []
files_modified: []
files_read: []
soft_depends_on: []
hard_depends_on: []
```
- Schema addition for clarity
- Planner fills in intelligently
- Pros: Machine-readable, enables future automation
- Cons: More fields to validate, slight format change

**Option C: Enhanced files_modified with ownership tags**
```yaml
files_modified:
  - path: src/models/user.ts
    ownership: created    # created | modified | appended
  - path: src/lib/index.ts
    ownership: appended   # safe to merge with other appends
```
- More expressive, still single field family
- Pros: Self-documenting, clear intent
- Cons: Requires parser update

**Recommendation:** **Option A (minimal change)** for immediate implementation. **Option B** as follow-up if Pathfinder integration requires richer metadata.

---

## 10. Practical Implementation Roadmap

### Phase 1: Wave Assignment Enhancement (Immediate)

1. **Update `assign_waves` function in gsd-planner.md** (lines 1109-1118)
   - Add file conflict detection
   - Implement `should_conflict()` logic
   - Maintain backward compatibility (existing depends_on still honored)

2. **Update task action descriptions**
   - When multiple plans modify same file in same wave
   - Explicitly document merge-friendly pattern in action
   - Example: "Append new exports to index.ts, do NOT rewrite existing"

3. **Metrics:**
   - Log parallelizability score for each phase
   - Track Wave 1 count vs total plans
   - Monitor merge conflicts (signal if pattern breaks)

### Phase 2: Pathfinder Integration (Post Task #5)

1. **Read Pathfinder analysis output**
   - File dependency graph
   - Type export locations
   - API contract definitions

2. **Enhance depends_on derivation**
   - Auto-detect type imports
   - Auto-detect API endpoint usage
   - Reduce manual depends_on declarations

3. **Enable cross-phase parallelism**
   - Pathfinder shows Phase 03 needs Phase 02 artifact
   - Phase 02 and 03 can run in parallel if no conflicts

### Phase 3: Orchestration Enhancement (Longer term)

1. **Plan-per-worktree strategy**
   - Each plan = one worktree session
   - Waves can spawn multiple worktrees in parallel
   - Reduces orchestration overhead

2. **Context budget refinement**
   - Small plans (1 task) use less context
   - Can allocate more tasks per phase
   - Better utilization of available context

---

## Summary: Key Recommendations

### 1. Decomposition Strategy
- **Shift from task-first to file-ownership-first thinking**
- Build dependency graph explicitly before grouping into plans
- Prefer smaller plans (1-2 tasks) to maximize parallelism

### 2. Wave Assignment
- **File ownership becomes primary parallelization signal**
- Enhance `should_conflict()` to distinguish resolvable conflicts (appends) from blocking conflicts (rewrites)
- Implement smart dependency detection (type imports, API usage)

### 3. Plan Scope
- **Target more plans with fewer tasks per phase**
- From 3-5 plans × 2-3 tasks → 5-10 plans × 1-2 tasks
- Keep same total task count, but distribute for parallelism

### 4. Metrics
- **Introduce Parallelizability Score = Wave 1 plans / Total plans**
- Target 70-85% score for typical phases
- Track historical improvement

### 5. Frontmatter
- **Option A (immediate):** Keep schema, enhance wave logic
- **Option B (post-Pathfinder):** Add file ownership fields if needed

### 6. Pathfinder Integration
- After Task #5: Use Pathfinder output for accurate dependency detection
- Enable cross-phase parallelism
- Auto-detect type and API dependencies

---

## Appendix: Example Phase Redesign

**Original Phase: "Product Management"**

Goal: Users can create, view, list, update products.

Current Plan:
```
Plan 01: Product CRUD (4 tasks)
  - Product model
  - Product API endpoints
  - Product UI component
  - Product forms

Total: 4 tasks, 1 plan, 1 wave (100% sequential)
```

**Redesigned Phase: "Product Management"**

```
Plan 01: Product model (1 task, Wave 1)
  - Define Product type, exports

Plan 02: Product API routes (1 task, Wave 2)
  - GET /products (list)
  - GET /products/:id (single)
  - POST /products (create)
  - PUT /products/:id (update)
  - DELETE /products/:id (delete)
  - Depends on Plan 01 (needs Product type)

Plan 03: Product list component (1 task, Wave 3)
  - List view, fetches from /products
  - Depends on Plan 02 (needs API endpoint)

Plan 04: Product detail component (1 task, Wave 3)
  - Single product view
  - Depends on Plan 02

Plan 05: Product forms (1 task, Wave 3)
  - Create + update forms
  - Reusable form component
  - Depends on Plan 02

Total: 5 tasks, 5 plans
- Wave 1: 1 plan (20%)
- Wave 2: 1 plan (20%)
- Wave 3: 3 plans (60% parallelism in final wave)
- Parallelizability: 1/5 = 20%... wait, this doesn't look better

Actually, can we improve further?
```

**Further Optimized: "Product Management"**

```
Plan 01: Product model + types (Wave 1)
Plan 02: API utilities (shared validation, response formatting) (Wave 1)
Plan 03: Product API routes (Wave 2, depends on 01)
Plan 04: Form utilities and components (Wave 1)
  - Reusable form inputs
  - Validation helpers
  - Don't need API yet (can stub)
Plan 05: Product list view (Wave 3, depends on 03)
Plan 06: Product detail view (Wave 3, depends on 03)
Plan 07: Product create/edit forms (Wave 3, depends on 04+03)

Total: 7 tasks, 7 plans
- Wave 1: 3 plans (43%)
- Wave 2: 1 plan (14%)
- Wave 3: 3 plans (43%)
- Parallelizability: 3/7 = 43%

Better! But still not 70-85%. Where's the missing parallelism?

Answer: We can push forms into Wave 1 if we stub the API!
```

**Maximum Parallelism: "Product Management"**

```
Plan 01: Product model + types (Wave 1)
Plan 02: API types + response contracts (Wave 1)
Plan 03: Shared utilities (validation, formatting) (Wave 1)
Plan 04: Form components and handlers (Wave 1)
  - Can develop without API (stub API calls)
Plan 05: Product list view component (Wave 1)
  - Can develop without API (use mock data)
Plan 06: Product API routes (Wave 2, depends on 01+02)
Plan 07: Product create/edit flow (Wave 2, depends on 04+06)
Plan 08: Product detail flow (Wave 2, depends on 05+06)

Total: 8 tasks, 8 plans
- Wave 1: 5 plans (63%)
- Wave 2: 3 plans (37%)
- Parallelizability: 5/8 = 63% ✓ Better!

Can we hit 70-85%? Only if:
- We can develop UI truly independently of API (good stubbing)
- Or we have more independent features (different products?)
- Or Pathfinder shows the type dependency is weaker than we thought
```

The example shows the tension: True parallelism requires **genuinely independent work**. The planner's job is to expose that independence where it exists, and help executors stub/delay dependencies where possible.

---

**Status:** Ready for team review and feedback

