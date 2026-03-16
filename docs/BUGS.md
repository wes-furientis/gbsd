# GSD Known Bugs

## BUG-001: Planner creates monolithic audit/verification plans instead of splitting by file

**Severity:** Medium
**Component:** gsd-planner
**Discovered:** 2026-03-15 (Phase 6: Polish, foveated seeker project)

### Problem

When a phase includes a read-through or audit task covering multiple large files, the planner creates a single plan with one agent responsible for all files. For document suites or multi-file verification, this produces superficial audits because one agent cannot thoroughly read 2000+ lines across 3-4 files within its context and attention budget.

### Example

Phase 6 (Polish) had plans 06-01 and 06-02 in Wave 1 (correctly split by file ownership for em-dash/acronym work). But the planner created a single 06-03 plan for the consistency audit covering all 3 documents (711 + 470 + 1050 lines) plus concordance.md. One agent reading 2200+ lines of dense technical prose and checking cross-references, parameter consistency, notation tables, and glossary accuracy across all of them cannot do a thorough job.

### Expected Behavior

The planner should split audit/verification plans by file ownership when:
1. The task involves reading full files (not just grepping or spot-checking)
2. Multiple files are independent enough to audit in parallel
3. Total line count across files exceeds a threshold (e.g., 500+ lines)

Each audit plan should own one file (or a small related set) and read concordance/reference files as shared context.

### Workaround

Manually intervene during execute-phase, delete the monolithic plan, and create per-file plans. This is what was done for Phase 6: 06-03 was split into 06-03 (seeker), 06-04 (primer), 06-05 (roadmap), all in Wave 2 running in parallel.

### Fix Suggestion

Add a heuristic to the planner: if a plan's `files_modified` list has 3+ files AND the task type involves full read-through/audit/verification (not just mechanical find-replace), split into per-file plans within the same wave. The planner already does this correctly for edit-focused plans (06-01 and 06-02 were split by file), so the logic just needs to extend to read-focused plans.
