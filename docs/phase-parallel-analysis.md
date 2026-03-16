# Phase-Level Parallelism Design Analysis

**Date:** 2026-03-15
**Task:** Design phase-level parallelism for independent phases
**Status:** Complete

---

## Executive Summary

Phase-level parallelism enables independent phases to execute in parallel, reducing total project duration from the sum of all phases to the critical path. This analysis covers dependency detection, merge strategy, risk factors, and interaction with wave-level parallelism.

**Key Finding:** Phases are naturally serializable (1→2→3→4) in ROADMAP.md, but many projects have independent phase clusters that could execute in parallel. The cost of parallelism is increased merge surface and schema synchronization complexity.

---

## 1. Dependency Detection: How to Identify Independent Phases

### 1.1 Dependency Signals in ROADMAP.md

**Signal 1: Explicit "Depends on" field**
```markdown
**Depends on**: Phase 3
**Depends on**: Nothing (first phase)
**Depends on**: Phase 1, Phase 2  # Multiple deps possible
```

Reading this field gives the primary dependency graph. Example from Pathfinder ROADMAP:
- Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7
- **Linear chain:** No parallelism opportunity

**Signal 2: Requirement IDs and categories**
```markdown
**Requirements**: AUTH-01, AUTH-02, PROF-01, PROF-02
```

Requirements with different category prefixes (AUTH, PROF, CONTENT) often indicate independent subsystems. If two phases have non-overlapping requirement categories, they're likely independent.

**Signal 3: File path mentions in plans**

Within the PLAN.md files, look for:
- Schemas/databases modified: `schema/user_auth.py` vs `schema/product_catalog.py`
- API endpoints touched: `/auth/login` vs `/products/search`
- UI surfaces: authentication pages vs dashboard pages
- Worker queues: `jobs/email_queue` vs `jobs/image_queue`

Non-overlapping file paths strongly suggest independence.

**Signal 4: Success criteria coupling**

```markdown
Phase 2: "User can log in"
Phase 3: "User can create profiles"

Phase 2 success: "User can log in and stay logged in"
Phase 3 success: "User profiles persist"
```

If Phase 3 success doesn't require Phase 2 completion, they might run in parallel (though Phase 2's success criteria may need Phase 1).

### 1.2 When Phases Are NOT Independent

**Strong coupling scenarios:**
- Schema migrations: Phase 1 creates users table, Phase 2 adds auth columns → serialized
- Shared API: Phase 2 builds auth API, Phase 3 calls auth API for access control → serialized
- Database: Phase 2 stores user sessions, Phase 3 requires session lookup → serialized
- Shared services: Both phases need a shared logging/monitoring service → Phase 1 foundation required
- Event contracts: Phase 2 emits events Phase 3 consumes → explicit dependency

**Weak coupling (candidates for parallel execution):**
- Orthogonal features: auth vs product catalog (different users, no interaction)
- Subsystem silos: email notifications vs push notifications (both send alerts, no data sharing)
- UI/backend pairs: frontend for dashboard (Phase 3) vs backend for dashboard API (Phase 2) can run in parallel if API contracts are locked

### 1.3 Practical Dependency Analysis Algorithm

```
Build phase dependency graph:
  1. Parse all "Depends on" fields → create DAG
  2. For each phase pair (A, B):
     a. Extract requirement IDs from both
     b. Check category overlap (AUTH, PROF, etc)
     c. Scan both plan files for file path patterns
     d. Count: same file path mentions / (total A files + total B files)
        - > 50% → likely coupled
        - < 20% → likely independent
  3. Output: DAG with edge weights (confidence in dependency)

Examples:
  Phase 1 (Auth) → Phase 2 (Profiles)       [HIGH confidence dependency]
    - Profiles need to know "who am I" → Auth required

  Phase 2 (Product Catalog) || Phase 3 (Recommendations)  [LOW confidence]
    - Recommendations can build on empty catalog
    - Parallel OK, but Phase 3 launches with less data
```

---

## 2. Phase-Level Merge Strategy

### 2.1 Branch-per-Phase vs Shared Main

**Option A: Branch-per-Phase (Recommended for Independent Phases)**

```
main (v1.0 baseline)
├─ phase-2-profiles    (Phase 2 executes here)
├─ phase-3-notifications (Phase 3 executes here)
├─ phase-4-discovery (Phase 4 executes here)
└─ (all merge back to main after completion)
```

**Advantages:**
- Isolation: Phase 2 work doesn't interfere with Phase 3
- Conflict detection: Merge conflicts surface at integration time (better than silent bugs)
- Rollback: If Phase 3 fails verification, it doesn't block Phase 2 → main
- Parallel CI: Each branch can run its own test suite in parallel

**Disadvantages:**
- Merge complexity: 3 parallel branches merging to main needs careful ordering
- Test freshness: Phase 2 tests pass on Phase 2 baseline, but may fail when Phase 3 code lands
- Schema conflicts: Both phases touch `schema/models.py` → merge conflict

**Option B: Wave-per-Phase on Same Branch (Simpler, Less Parallel)**

Phases still depend on each other, but within each phase, plans execute in waves. No branch-per-phase overhead.
- Use when phases ARE serialized anyway
- Reduces merge surface to one per phase

### 2.2 Merge Integration Strategy (Branch-per-Phase)

**Prerequisite:** All independent phases must complete verification before merging any back to main.

```
1. All phases complete execution and verification separately
   Phase 2: SUMMARY created, VERIFICATION passed
   Phase 3: SUMMARY created, VERIFICATION passed
   Phase 4: SUMMARY created, VERIFICATION passed

2. Merge-back staging (sequential to avoid main thrashing):
   git checkout main
   git merge phase-2-profiles --no-ff
   git merge phase-3-notifications --no-ff
   git merge phase-4-discovery --no-ff

3. Final integration test:
   Run full test suite on merged main to catch cross-phase interactions
   If failures: identify affected phase → gap closure for that phase
```

**Merge conflict resolution:**
- Conflicting files: Manually resolve or ask which phase "owns" the file
- Example: Both Phase 2 and Phase 3 edit `schema/models.py`
  - If they touch different tables → manual merge easy
  - If they touch same table → dependency was missed, need gap closure

### 2.3 Database Migrations

**Critical risk:** Parallel migrations from independent phases can deadlock or corrupt data.

**Safe approach:**
- Never parallelize migrations to the same database
- Instead: Phase 1 creates schema foundation, all phases read from it
- Phase 2+ add columns/tables in separate, serial migration waves
- OR: Use feature flags to tolerate schema changes mid-phase

---

## 3. Phase-Level Worktrees vs Wave-Level

### 3.1 Scope Differences

| Level | Duration | Files Touched | Merge Surface | Isolation |
|-------|----------|---------------|---------------|-----------|
| **Wave** | 1-2 hours | 5-20 files | Single branch | Low (within phase) |
| **Phase** | 1-7 days | 50-500 files | Branch → main merge | High (between phases) |

**Wave-level parallelism** (current GSD):
- Multiple plans execute in parallel (Wave 1 = [01-01, 01-02], Wave 2 = [01-03])
- All commits to same branch within phase
- Conflicts resolved inline during execution
- Risk: Low (all work on same branch, clear sequence)

**Phase-level parallelism** (proposed):
- Multiple phases execute in parallel (Phase 2, 3, 4 concurrently)
- Each phase may have internal wave parallelism too
- Commits to separate branches, merge later
- Risk: High (cross-phase interactions emerge at merge time)

### 3.2 Nested Parallelism: Phases × Waves

```
        PHASE 2            PHASE 3            PHASE 4
       (Profiles)      (Notifications)      (Discovery)
          |                  |                  |
       Wave 1            Wave 1             Wave 1
      (01-01, 01-02)   (03-01, 03-02)    (04-01, 04-02)
          |                  |                  |
       Wave 2            Wave 2             Wave 2
      (01-03)           (03-03)            (04-03)
          |                  |                  |
         [✓]               [✓]               [✓]

        All phases complete, then merge back to main
```

**Execution model:**
- Orchestrator spawns 3 phase executors (one per phase)
- Each executor runs waves internally (Wave 1 → Wave 2)
- Orchestrator waits for all 3 phases to complete verification
- Then merges all branches back to main

---

## 4. Explicit Phase Independence Annotation

### 4.1 Current ROADMAP Structure

```markdown
**Depends on**: Phase 2
```

This is sufficient for serial execution but doesn't capture:
- Degree of independence (hard coupling vs soft dependency)
- Conflict zones (which files do both phases touch)
- Validation strategy (what to verify at merge time)

### 4.2 Proposed Enhancement (Optional)

Add metadata to ROADMAP.md:

```markdown
### Phase 2: Profiles
**Depends on**: Phase 1
**Parallel OK with**: Phase 3, Phase 4
**Conflict zones**: schema/models.py (adds User.profile table)

### Phase 3: Notifications
**Depends on**: Phase 1
**Parallel OK with**: Phase 2, Phase 4
**Conflict zones**: schema/models.py (adds Notification table)
```

**Benefit:** Roadmapper can auto-generate parallelization hints.
**Cost:** Extra metadata burden, only valuable for complex projects.

**Decision:** Not mandatory for MVP. Add if user workflows show frequent manual conflict resolution.

---

## 5. Risk Analysis: When Phase Parallelism Fails

### 5.1 Schema Synchronization

**Risk:** Phase 2 adds `users.auth_method`, Phase 3 adds `users.timezone`

**Failures:**
1. **Silent data loss:** Phase 2 creates users without timezone, Phase 3 expects it → gaps in data
2. **Migration conflicts:** Both write to same migration file → merge conflict
3. **Runtime type mismatches:** Phase 3 code assumes `timezone` but Phase 2 users lack it

**Mitigation:**
- Phase 1 foundation defines ALL schema changes upfront (pessimistic)
- OR: Feature flags allow Phase 3 to handle missing `timezone` gracefully
- OR: Post-merge validation checks for orphaned columns

### 5.2 Shared API Contracts

**Risk:** Phase 2 builds auth API, Phase 3 calls it before Phase 2 lands on main

**Failures:**
1. **Branch-level API mismatch:** Phase 3 branch has no auth API (it's on Phase 2 branch)
2. **Signature changes:** Phase 2 changes `/auth/login` response format, Phase 3 branch still expects old format
3. **Test failures:** Phase 3 integration tests fail because Phase 2 API missing from main

**Mitigation:**
- Lock API contracts (Swagger/OpenAPI) before phases diverge
- Phase 2 implements contract, Phase 3 consumes contract
- Validation at merge: both branches must satisfy contract

### 5.3 Database State

**Risk:** Phase 2 adds user sessions, Phase 3 assumes sessions exist

**Failures:**
1. **Null pointer:** Phase 3 code: `session = lookup_session(user_id)` → crashes if Phase 2 not merged
2. **Migration sequence:** Phase 3 migrations depend on Phase 2 schema → ordering matters
3. **Data consistency:** Parallel Phase 2 writes sessions, Phase 3 reads them → race conditions

**Mitigation:**
- Phase 1 creates session table (even if empty)
- Phase 2 populates sessions
- Phase 3 reads from table (safe even if empty)
- Or: Phase 3 depends on Phase 2 (serialized)

### 5.4 Shared Dependencies (pip/npm packages)

**Risk:** Phase 2 adds `pandas==1.5.0`, Phase 3 adds `numpy==2.0`

**Failures:**
1. **Version conflict:** `numpy==2.0` incompatible with `pandas==1.5.0` → import errors
2. **Silent incompatibility:** Tests pass on separate branches but fail when merged
3. **Transitive conflicts:** Phase 2 needs `scipy==1.9.0`, Phase 3 needs `scipy==1.11.0`

**Mitigation:**
- Centralized dependency management (pyproject.toml locked before phases start)
- Phase 2/3 update the shared file (merge conflicts are explicit)
- Test at merge time: `pip install -e .` and run full suite

### 5.5 Architectural Decisions

**Risk:** Phase 2 picks SQLAlchemy ORM, Phase 3 picks raw SQL queries

**Failures:**
1. **Code style mismatches:** Different patterns in same codebase (cosmetic)
2. **Performance cliff:** Raw SQL Phase 3 code is 10x faster than ORM Phase 2, creates pressure to rewrite
3. **Feature parity issues:** ORM features unavailable in raw SQL, vice versa
4. **Maintenance burden:** Two ways to do database access forever

**Mitigation:**
- Phase 1 CLAUDE.md locks architectural decisions upfront
- Phase 2/3 must follow locked decisions
- Review CLAUDE.md before starting phase

---

## 6. Interaction with Wave-Level Parallelism

### 6.1 Execution Timeline

```
Timeline: Phase-level (outer) × Wave-level (inner)

Day 1-2:
  Phase 1 (Foundation)
    Wave 1: [execute-plan 01-01, 01-02 in parallel]
    Wave 2: [execute-plan 01-03]
  Phase 1 complete → merge to main

Day 3-5 (PARALLEL):
  Phase 2 (Profiles)                Phase 3 (Notifications)
    Wave 1: [02-01, 02-02 ∥]         Wave 1: [03-01 alone]
    Wave 2: [02-03]                  Wave 2: [03-02, 03-03 ∥]
  Phase 2 complete                  Phase 3 complete

Day 6:
  Merge Phase 2, Phase 3 to main (sequential, conflict resolution)
  Run integration tests

Day 7+:
  Phase 4 (Discovery) — now has Phase 2+3 baseline
```

### 6.2 Implications

**Advantage:** Nested parallelism reduces time from Sum(all phases) to critical path
- Without: 1 + 2 + 2 + 1 = 6 days
- With phase parallelism: 1 + max(2, 2) + 1 + 1 = 5 days
- Critical path = longest sequence of dependent phases

**Complexity:** Wave-level dependencies within phases must account for cross-phase merges
- Wave 2 of Phase 2 may depend on Wave 1 of Phase 3 (data dependency)
- Explicit in PLAN.md: "Blocked by Phase 3 Wave 1"

**Testing burden:** More test suites running in parallel
- Phase 2 tests on Phase 2 branch
- Phase 3 tests on Phase 3 branch
- Merge tests on main
- Risk of inconsistent test environments

---

## 7. Merge Strategy for Phase-Level Branches

### 7.1 Pre-Merge Checklist

Before merging any phase branch to main:

```
[ ] Phase execution complete (all plans done)
[ ] SUMMARY.md created for each plan
[ ] VERIFICATION.md passed (all must-haves verified)
[ ] ROADMAP.md updated (phase marked complete)
[ ] No unresolved gaps (gap-closure phase if needed)
[ ] Conflicts with main identified (git merge --no-commit)
[ ] Conflict files reviewed (understand both sides)
[ ] Unit tests pass on branch
```

### 7.2 Merge Order Strategy

**Option A: Sequential Merge (Safer)**
```bash
git checkout main
git merge phase-2-profiles       # Phase 2 lands first
  [resolve conflicts if any]
  [run tests]
git merge phase-3-notifications  # Phase 3 lands second
  [resolve conflicts if any]
  [run tests]
git merge phase-4-discovery      # Phase 4 lands third
  [resolve conflicts]
  [run full integration tests]
```

**Option B: Parallel Rebase (Riskier)**
```bash
# All phases rebase onto main first
git checkout phase-2-profiles && git rebase main
git checkout phase-3-notifications && git rebase main
git checkout phase-4-discovery && git rebase main

# Then merge all at once
git checkout main && git merge phase-2-profiles phase-3-notifications phase-4-discovery
```

**Recommendation:** Option A (sequential) because:
- Conflicts emerge one at a time
- Tests run between each merge
- Easier to blame which phase caused test failure
- If Phase 3 merge breaks tests, we know it's Phase 3, not Phase 2

### 7.3 Conflict Resolution Strategy

**Automatic resolution (safe):**
- Different files → git merge handles automatically
- Same file, different lines → git merge handles automatically

**Manual resolution (needs review):**
- Both phases edited schema/models.py (same lines)
  - Review both definitions
  - Merge with both changes (additive)
  - Re-run type checkers to validate

- Both phases edit config.json
  - Review what each phase changed
  - Manually combine (or use tool like `jq merge`)

- Dependency versions conflict
  - Phase 2: `numpy==1.24.0`
  - Phase 3: `numpy==1.25.0`
  - Pick highest compatible version, test both

---

## 8. Dependency Analysis Algorithm (Detailed)

### 8.1 Formal Dependency Detection

```python
def compute_phase_independence(phases: List[Phase]) -> Dict[int, List[int]]:
    """
    Compute which phases can run in parallel.

    Returns: phase_id → list of phase IDs it's safe to run in parallel with

    Algorithm:
    1. Parse "Depends on" field (explicit dependencies)
    2. Scan all PLAN.md files for file path mentions (implicit dependencies)
    3. Extract requirement IDs (category-based coupling)
    4. Compute conflict score between each phase pair
    5. Threshold: conflict_score < 0.3 → independent
    """

    parallelizable = {}

    for phase in phases:
        can_parallel_with = []

        for other_phase in phases:
            if phase.id == other_phase.id:
                continue

            # Check explicit dependencies
            if other_phase.id in phase.depends_on:
                continue  # Explicit blocker

            # Check file path overlap
            files_a = extract_files_from_plans(phase)
            files_b = extract_files_from_plans(other_phase)
            overlap = len(files_a & files_b) / (len(files_a) + len(files_b))

            # Check requirement categories
            cats_a = [req.split('-')[0] for req in phase.requirements]
            cats_b = [req.split('-')[0] for req in other_phase.requirements]
            cat_overlap = len(set(cats_a) & set(cats_b)) / len(set(cats_a) | set(cats_b))

            # Conflict score: weighted combination
            conflict_score = 0.6 * overlap + 0.4 * cat_overlap

            if conflict_score < 0.3:  # Threshold
                can_parallel_with.append(other_phase.id)

        parallelizable[phase.id] = can_parallel_with

    return parallelizable
```

### 8.2 Example: Pathfinder Roadmap

Applying algorithm to Pathfinder phases:

```
Phase 1 (Graph Foundation) → all others (explicit dependency)
Phase 2 (Python AST) depends on Phase 1 only
  Parallel with: Phase 3 (if Python AST foundation is sufficient)
  NO — Phase 3 needs Python AST output to query it

Phase 2 → Phase 3 (explicit dependency)
Phase 3 (CLI) parallel with Phase 4 (Markdown)?
  File overlap: Phase 3 adds CLI queries, Phase 4 adds document extraction
    - Phase 3: src/cli.py, src/query.py, tests/test_cli.py
    - Phase 4: src/extractors/markdown.py, tests/test_markdown.py
    - Overlap: ~10% (both edit src/core/graph.py? LOW)
  Requirement overlap: QURY-01-06 vs MDEX-01-12 (different categories)
    - Overlap: 0%
  Conflict score: 0.6 * 0.1 + 0.4 * 0 = 0.06
  Result: CAN RUN IN PARALLEL (conflict_score < 0.3)
```

---

## 9. Roadmapper Enhancement: Explicit Independence Annotation

### 9.1 Optional ROADMAP Extension

If roadmapper detects independent phases, optionally add:

```markdown
### Phase 2: Python AST Extractor
**Goal**: Agents can generate code intelligence index
**Depends on**: Phase 1
**Parallel OK**: None (Phase 3 depends on us)
**Conflict zones**:
  - src/core/graph.py (adds extraction method)
  - pyproject.toml (adds ast dependencies)

### Phase 3: CLI and Structural Queries
**Goal**: Agents can query the code graph
**Depends on**: Phase 2
**Parallel OK**: Phase 4 (deterministic cross-domain linking)
**Conflict zones**:
  - src/core/graph.py (adds query methods)
  - tests/ (lots of tests, but no overlaps with Phase 4)
```

### 9.2 When to Annotate

- **Do annotate** if project has clear phase clusters (e.g., Platform A + Platform B)
- **Skip annotation** if phases are strictly linear (Pathfinder case)
- **Add during revision** if user says "these phases should be parallel"

---

## 10. Risk Mitigation Summary Table

| Risk | Likelihood | Severity | Mitigation |
|------|-----------|----------|-----------|
| Schema conflicts | High | Critical | Phase 1 defines schema, phases add only; feature flags for null-safe reads |
| API contract mismatch | Medium | Critical | Lock OpenAPI spec before phases diverge; validate at merge |
| Database race conditions | High | Critical | Serialized phases for shared state; or pessimistic locking |
| Test environment divergence | High | Medium | Run integration tests at merge time; don't trust branch-level tests |
| Dependency version conflicts | Medium | Medium | Centralized lock file (pyproject.toml); update together |
| Architectural divergence | Low | Medium | CLAUDE.md locks decisions before phase start |
| Merge conflict storms | Medium | Medium | Small conflict zones (different files); sequential merge order |

---

## 11. Recommendations for GSD Roadmapper

### 11.1 Current Behavior (No Phase Parallelism)

- ROADMAP.md specifies `Depends on: Phase N`
- Planner creates sequential waves
- Execute-phase runs phases one at a time
- No merge complexity

**Cost:** If Phase 2 (auth) takes 10 days and Phase 3 (notifications) is independent but must wait, total time = 20 days instead of parallel 10-12 days.

### 11.2 Proposed Enhancement (Optional)

1. **Roadmapper detects independence:**
   - Scan file paths in plans (cache for perf)
   - Check requirement category non-overlap
   - Compute conflict score
   - Annotate `Parallel OK with: [phases]`

2. **Planner honors parallelization:**
   - Read `Parallel OK` annotation
   - If `--parallel` flag passed, create separate branches
   - Document merge order and conflict zones in STATE.md

3. **Execute-phase spawns parallel orchestrators:**
   - One executor per phase (not sequential queue)
   - Wait for all to complete
   - Merge sequentially back to main
   - Run final integration tests

### 11.3 MVP Approach

**Don't implement auto-parallelization yet.** Instead:

1. Document the dependency detection algorithm (done)
2. Expose `--parallel` flag to roadmapper (optional, not auto)
3. User explicitly annotates: `roadmap --parallelizable`
4. If user wants parallelization, they request it explicitly
5. System produces branches and merge instructions

**Rationale:** Phase parallelism is powerful but risky. Better to let users opt-in than auto-parallelize and create surprise merge conflicts.

---

## 12. Conclusion

**Key Insights:**

1. **Phases are serialized by default** — ROADMAP.md's "Depends on" field creates a linear chain
2. **But many projects have independent clusters** — auth + notifications can run in parallel
3. **Dependency detection is automatable** — file path overlap + requirement categories + explicit deps
4. **Merge complexity is the main cost** — 3 parallel branches merging to main requires careful sequencing and integration tests
5. **Wave-level parallelism is independent** — multiple waves within one phase, or multiple phases, both work
6. **Schema is the biggest risk** — parallel phases touching the same table cause data inconsistencies

**Recommendation for User's Goal:**

To achieve "longer phases with more independent tasks":

- Use wave-level parallelism within each phase (plans execute in parallel) ← Already done
- Add explicit phase independence annotation to ROADMAP.md (optional)
- Implement `--parallel` opt-in flag for users who want it
- Document merge strategy and conflict zones upfront
- Add integration test step between phase branch merges
- Don't auto-parallelize; let users control when it's worth the merge complexity

This preserves the simplicity of serial phases while enabling power users to exploit independent phase clusters.

