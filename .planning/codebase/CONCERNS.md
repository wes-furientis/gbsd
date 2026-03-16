# Codebase Concerns

**Analysis Date:** 2026-03-15

## Tech Debt

### Monolithic Plan Creation for Multi-File Audits

- **Issue:** The planner creates single audit/verification plans covering 3+ large files (1000+ lines total), resulting in superficial work because one agent cannot thoroughly read and cross-reference multiple dense files within its context budget.
- **Files:** `agents/gbsd-planner.md`, `gbsd/bin/lib/phase.cjs` (plan creation logic)
- **Impact:** Audit plans produce low-quality results (incomplete cross-references, missed consistency violations, shallow pattern analysis). Example: Phase 6 consistency audit created one plan for 2200+ lines across 3 documents when it should have been 3 parallel plans.
- **Fix approach:**
  1. Detect when task involves full file read-through (not just grepping or spot-checking)
  2. Add logic to split plans by file ownership when total line count exceeds 500+ lines
  3. Allow multiple files to be "read-only" context while one file is owned/audited per plan
  4. Ensure wave assignment allows parallel execution of split plans

### String Manipulation for Markdown Parsing

- **Issue:** Heavy reliance on regex + string replace for ROADMAP.md, STATE.md, and REQUIREMENTS.md manipulation instead of structured parsing. Over 70 regex patterns across phase.cjs, verify.cjs, roadmap.cjs, and state.cjs.
- **Files:**
  - `gbsd/bin/lib/phase.cjs` (lines 328-869, 30+ replace calls)
  - `gbsd/bin/lib/state.cjs` (lines 78-857, field extraction and patching via regex)
  - `gbsd/bin/lib/verify.cjs` (lines 64-100, commit hash and section extraction)
  - `gbsd/bin/lib/roadmap.cjs` (lines 109-277, phase/milestone parsing)
- **Impact:**
  - Brittle to markdown formatting changes (extra whitespace, different header styles)
  - Difficult to debug when patterns fail silently
  - Hard to extend with new document structures
  - Example: `replace(/^\d+(?:\.\d+)*-?/, '')` assumes specific phase number format; breaks if format changes
- **Fix approach:**
  1. Create a Markdown AST parser (lightweight, handles frontmatter + headings + tables)
  2. Build dedicated frontmatter class with typed field access (not regex)
  3. Replace low-level replace() calls with structured tree mutations
  4. Add test fixtures for each markdown document type

### Shell Command Escaping for Git Operations

- **Issue:** Manual shell escaping logic in `execGit()` uses allowlist regex `/^[a-zA-Z0-9._\-/=:@]+$/` to detect "safe" arguments, falls back to single-quote wrapping otherwise. Pattern is overly restrictive and adds complexity.
- **Files:** `gbsd/bin/lib/core.cjs` (lines 150-169), `isGitIgnored()` (lines 140-148)
- **Impact:** Potential for argument injection if pattern is incomplete; maintenance burden if git arguments expand beyond current safe set
- **Fix approach:**
  1. Use `child_process.execFile()` instead of `execSync()` + shell escaping (avoids shell entirely)
  2. Pass git args as array, not concatenated string
  3. Removes need for custom escaping logic

### Cross-Platform Path Handling Inconsistency

- **Issue:** Code uses `path.join()` for path construction (correct) but then converts to POSIX with `toPosixPath()` for output. Mixed usage of `path.sep` and hardcoded `/` in string patterns creates subtle bugs on Windows.
- **Files:** `gbsd/bin/lib/core.cjs` (lines 12-14, `toPosixPath` function), `gbsd/bin/lib/init.cjs` (line 174, hardcoded `/` in file globbing)
- **Impact:**
  - Regex patterns may fail on Windows if they expect POSIX paths
  - Examples: `backtic-refs` regex in verify.cjs (`/[a-zA-Z]{1,10})`
  - File paths in error messages may be inconsistent format
- **Fix approach:**
  1. Always construct paths with `path.join()` and `path.resolve()`
  2. Apply `toPosixPath()` ONLY at output/JSON serialization time
  3. Never use `/` in path patterns; use `path` module instead

---

## Known Bugs

### BUG-001: Planner Creates Monolithic Audit Plans

- **Symptoms:** When a phase includes multi-file audit tasks, one plan is created with all files, resulting in shallow coverage. Agent context fills up during reading; cross-reference checking is incomplete or missed.
- **Files:** `agents/gbsd-planner.md` (discovery levels), `gbsd/bin/lib/phase.cjs` (plan creation)
- **Trigger:**
  1. Create phase with 3+ large files (500+ lines each)
  2. Assign audit/verification task covering all files
  3. Run `/gbsd:plan-phase`
  4. Observe single plan with all files listed in `files_modified`
- **Workaround:** After plan creation, manually split into per-file plans and re-assign to separate waves. Verified in Phase 6 where 06-03 was manually split into 06-03, 06-04, 06-05.

### BUG-002: Config Depth-to-Granularity Migration Silent Failure

- **Symptoms:** Projects upgraded from older GBSD versions may have `depth` key in config.json. Migration to `granularity` runs, but if the write-back fails silently (due to permissions), the old key persists and new config logic may not read it.
- **Files:** `gbsd/bin/lib/core.cjs` (lines 89-95), `gbsd/bin/lib/config.cjs` (lines 51-55)
- **Trigger:**
  1. Have config.json with `"depth": "standard"`
  2. Run any GBSD command
  3. Check if `depth` is removed and `granularity` is present
- **Workaround:** Manually delete `depth` key and re-run init or config-set commands

### BUG-003: References Verification Misses Relative Paths

- **Symptoms:** `cmdVerifyReferences()` uses regex `/`([^`]+\/[^`]+\.[a-zA-Z]{1,10})`/` which requires both a directory separator AND a file extension. Valid relative paths like `src/foo` (no extension) or `.env` (no directory separator) are not checked for existence.
- **Files:** `gbsd/bin/lib/verify.cjs` (lines 238-249)
- **Trigger:** Document references a relative path like `` `src/config` `` or `` `.env` ``; verification passes even if file doesn't exist
- **Impact:** Plans/summaries can reference non-existent files; discovered only during manual review
- **Workaround:** Always include file extensions in references; structure directories explicitly

---

## Security Considerations

### Shell Injection Risk in Git Check-Ignore

- **Risk:** `isGitIgnored()` uses allowlist-based character filtering (`replace(/[^a-zA-Z0-9._\-/]/g, '')`) on file paths before passing to `git check-ignore`. If a path contains special characters, they are silently stripped, potentially allowing unintended matches.
- **Files:** `gbsd/bin/lib/core.cjs` (lines 134-148)
- **Current mitigation:** Allowlist is permissive for valid file path characters (alphanumeric, dot, dash, slash); filenames with special chars are mangled but won't break shell
- **Recommendations:**
  1. Switch to `execFile()` to avoid shell entirely (highest priority)
  2. If sticking with `execSync()`, use `--no-index` flag (already present) which prevents argument interpretation

### Missing Input Validation on Frontmatter Fields

- **Risk:** Frontmatter parsing accepts arbitrary JSON; no schema validation. Malformed frontmatter (missing fields, wrong types) causes silent failures or crashes downstream.
- **Files:** `gbsd/bin/lib/frontmatter.cjs` (parsing), `gbsd/bin/lib/verify.cjs` (validation)
- **Current mitigation:** Verification checks for required fields post-parse; but no type checking (e.g., `wave` should be integer)
- **Recommendations:**
  1. Define JSON schema for each frontmatter type (PLAN, SUMMARY, etc.)
  2. Validate against schema on load; error if missing required fields or wrong types
  3. Provide clear error messages indicating what's wrong and where

### Config File Write Without Atomic Guarantees

- **Risk:** `cmdConfigSet()` and migration code write config.json without atomic operations. If write fails mid-operation, config is corrupted or incomplete.
- **Files:** `gbsd/bin/lib/config.cjs` (lines 135, 86), `gbsd/bin/lib/core.cjs` (lines 94)
- **Current mitigation:** Try-catch blocks catch errors but don't restore previous state
- **Recommendations:**
  1. Write to `.tmp` file first, then rename atomically
  2. On write failure, preserve original config
  3. Add changelog comments in config for auditability

---

## Performance Bottlenecks

### Regex Compilation in Loops

- **Problem:** Phase number comparison and phase filtering uses raw regex patterns in loops without pre-compilation. Pattern `/^(\d+)([A-Z])?((?:\.\d+)*)` is compiled multiple times per phase comparison.
- **Files:** `gbsd/bin/lib/core.cjs` (lines 186-189, `comparePhaseNum`)
- **Impact:** Negligible for typical projects (<50 phases), but noticeable if project has 100+ phases or regex runs in rapid succession
- **Improvement path:** Pre-compile regex patterns as module-level constants or within class constructors

### File I/O Synchronous Blocking

- **Problem:** All file operations use synchronous `fs.readFileSync()`, `fs.writeFileSync()`, `execSync()`. For large projects with many phase directories or large markdown files (>10MB), reads/writes block the CLI.
- **Files:** `gbsd/bin/lib/*.cjs` (all file operations)
- **Impact:** CLI responsiveness degrades with project size; especially noticeable on slow storage (SMB shares, cloud synced disks)
- **Improvement path:**
  1. Profile CLI startup time for large projects (100+ phases)
  2. If >1s, migrate to async fs module with Promise-based API
  3. Use promise-based `execSync` wrapper or real spawn for git operations
  4. Add `--async` flag for user opt-in

### Markdown Regex Matching on Large Files

- **Problem:** `cmdVerifyReferences()` and similar functions match all `@`-references and backtick paths in document, then resolve each one with `fs.existsSync()`. For a 2000+ line SUMMARY.md with 100+ file references, this is 100+ synchronous file stat calls.
- **Files:** `gbsd/bin/lib/verify.cjs` (lines 221-249)
- **Impact:** Verification step slows down proportionally to file reference count; noticeable for comprehensive SUMMARY.md files
- **Improvement path:**
  1. Batch file existence checks (read all paths, call `fs.existsSync()` in parallel)
  2. Or: defer detailed checks to lazy evaluation (only check when user requests detailed report)

---

## Fragile Areas

### Phase Number Normalization

- **Files:** `gbsd/bin/lib/core.cjs` (lines 177-184, `normalizePhaseName`)
- **Why fragile:** Phase names are parsed with loose regex `/^(\d+)([A-Z])?((?:\.\d+)*)/i`. Works for expected formats (01, 02A, 02.1, 02A.1), but edge cases exist:
  - Input `1a2` matches as phase 01, letter A, decimal `.2` — unexpected
  - Input `10B.2.3.4` works but deeply nested decimals are untested
  - Decimal phases with trailing zeros (01.01 vs 01.1) may have inconsistent sorting
- **Safe modification:**
  1. Add comprehensive test cases for all expected + edge case formats
  2. Consider stricter regex or explicit validation function
  3. Document assumptions: "Phases must be `\d+([A-Z])?(\\.\d+){0,2}`" (max 2 decimal levels)

### Markdown State Mutation via Replace

- **Files:** `gbsd/bin/lib/state.cjs`, `gbsd/bin/lib/phase.cjs`
- **Why fragile:** Functions like `stateReplaceField()` and roadmap field updates chain `replace()` calls. If one pattern doesn't match, subsequent patterns operate on partially-modified content, causing data corruption.
- **Example:**
  ```javascript
  content.replace(boldPattern, ...)
    .replace(plainPattern, ...)  // <-- may now match wrong location
    .replace(anotherPattern, ...)
  ```
- **Safe modification:**
  1. Refactor to single-pass AST traversal
  2. Add unit tests after each replace() to verify content structure
  3. Return early if primary pattern fails (don't chain replacements blindly)
- **Test coverage:** Only 13 tests cover state patching logic; expand to 30+

### Frontmatter Extraction and Round-Tripping

- **Files:** `gbsd/bin/lib/frontmatter.cjs` (lines 1-100)
- **Why fragile:** Code extracts YAML frontmatter from markdown, parses it, then reconstructs with `reconstructFrontmatter()`. If reconstruction logic differs from original formatting (quote style, key order, spacing), repeated read/write cycles corrupt the file.
- **Example:** Original has `phase: "01"` (quoted); after round-trip becomes `phase: 01` (unquoted) — JSON parsing same, but markdown render different
- **Safe modification:**
  1. Preserve original frontmatter text instead of round-tripping through YAML.parse/stringify
  2. Only modify fields that changed; leave others untouched
  3. Add "golden file" tests comparing round-trip output with known-good files

---

## Scaling Limits

### Phase Directory Growth

- **Current capacity:** System lists all phases with `fs.readdirSync()` on `.planning/phases/`. Scales linearly to 1000+ phases before noticeable slowdown.
- **Limit:** At 10,000+ phases, directory listing and sorting becomes slow (>500ms on older disks).
- **Scaling path:**
  1. Implement phase indexing: store phase manifest in `.planning/phases/_index.json` instead of relying on directory listing
  2. Cache phase list in memory with version file watch for invalidation
  3. Lazy-load phase details only when needed (don't read all phase directories on every CLI call)

### Large Markdown Documents (>10MB)

- **Current capacity:** Regex matching and full-text replace operations on documents <5MB complete in <1s
- **Limit:** Documents >10MB cause noticeable (>2s) CLI slowdown due to synchronous file I/O + regex compilation
- **Scaling path:**
  1. For ROADMAP.md: implement section-based loading (read/write only the section being modified, not whole file)
  2. For REQUIREMENTS.md: consider database-backed storage for large projects (convert to SQLite)
  3. Stream-based processing for verification tasks instead of full regex match

### Number of Requirements per Phase

- **Current capacity:** Phase requirements extracted from ROADMAP with regex; scales well to 50+ requirements per phase
- **Limit:** Over 100 requirements/phase, traceability table rendering and verification checking becomes O(n²) in phase count
- **Scaling path:**
  1. Implement requirement index: separate `REQUIREMENTS.md` structure with fast lookups
  2. Phase-requirement mapping as two-way references instead of one-way table
  3. Lazy-load requirement details only for current phase

---

## Dependencies at Risk

### Node.js execSync() Deprecation Warning

- **Risk:** Node.js may deprecate or change behavior of `execSync()` in future versions. Currently used for all git operations and some file scanning.
- **Impact:** Code may break on Node 22+ if deprecation is enforced
- **Migration plan:** Switch to `child_process.execFile()` for all git operations (no shell escaping needed), and Promise-based fs module for file I/O

### Markdown Parsing Brittleness

- **Risk:** No formal markdown parser used. Custom regex-based parsing is fragile to markdown variations (extra whitespace, different heading styles, code block nesting).
- **Impact:** As GBSD documents evolve, parsing may fail silently or mangle data
- **Migration plan:** Integrate lightweight markdown parser (e.g., `marked` or `gray-matter`) for robust parsing; maintain backward compatibility with current document formats

---

## Test Coverage Gaps

### State Patching Logic Under-tested

- **What's not tested:**
  - Multi-field patch operations in single call
  - Patch on malformed state file (missing fields)
  - Patch with special characters in values (quotes, newlines, `$` signs)
  - Interaction between bold (`**Field:**`) and plain (`Field:`) field formats when both exist
- **Files:** `tests/state.test.cjs` (1378 lines, but only 13 tests for `cmdStatePatch`)
- **Risk:** State mutations fail silently; users don't discover until next phase is misclassified
- **Priority:** Medium → expand state patching tests to 40+ cases

### Verify Commands Lack Integration Tests

- **What's not tested:**
  - Full phase verification workflow (plan → summary → verify-phase-completeness)
  - Cross-file reference checking with actual project structure
  - Verification on projects with missing files (broken references)
  - Commit hash validation against real git history
- **Files:** `tests/verify.test.cjs` (1013 lines, mostly unit tests; no integration tests)
- **Risk:** Verification may pass even if summaries are incomplete or references are broken
- **Priority:** Medium → add 20+ integration test scenarios

### Frontmatter Parsing Round-Trip Tests Missing

- **What's not tested:**
  - Parse → modify → stringify → parse again (data preservation)
  - Round-trip with YAML special characters (quotes, colons, dashes)
  - Whitespace preservation (preserving indentation style)
- **Files:** `tests/frontmatter.test.cjs` (not found; functionality tested indirectly)
- **Risk:** Repeated phase edits corrupt frontmatter; users hit "invalid frontmatter" errors
- **Priority:** High → implement dedicated frontmatter round-trip test suite (30+ cases)

### Phase Number Parsing Edge Cases Under-tested

- **What's not tested:**
  - Deeply nested decimals (01.1.1.1)
  - Leading zeros handling (01 vs 1)
  - Uppercase letter variants (01A vs 01a)
  - Comparison across different formats (01 vs 01.0)
- **Files:** `tests/core.test.cjs` (804 lines, `comparePhaseNum` has ~10 tests)
- **Risk:** Phase sorting becomes non-deterministic; execution order changes unpredictably
- **Priority:** Medium → expand phase parsing tests to 50+ cases covering all documented formats

---

## Missing Critical Features

### No Dry-Run Mode for Destructive Operations

- **Problem:** Commands like `phase remove --force` and `milestone complete` permanently delete/archive data with no `--dry-run` option to preview changes
- **Impact:** Users cannot verify impact before destructive action; recovery requires git history inspection
- **Blocks:** Safe rollback workflows; automated testing of GBSD against real projects

### No Undo/Rollback for State Changes

- **Problem:** Once phase is marked complete or requirements are marked satisfied, there's no easy way to revert. Git commit can be reverted, but `.planning/` state is not automatically rolled back.
- **Impact:** Accidental phase completion requires manual `.planning/` directory editing
- **Blocks:** Safe experimentation with phase workflow; recovery from user errors

### Limited Debugging Visibility into Config Resolution

- **Problem:** When `loadConfig()` falls back to defaults due to missing/corrupt config.json, users don't know why their settings aren't being applied
- **Impact:** Config issues are silent; users think settings are broken when actually they're just using defaults
- **Blocks:** Troubleshooting; config audit trails

---

## Known Limitation: Planner Context Window Assumptions

- **Issue:** Planner uses hardcoded 50% context target and 2-3 task caps, designed for Claude 3.5 Sonnet's 200k context window. Current Claude Opus 4.6 and Sonnet 4.6 have 1M context windows.
- **Files:** `agents/gbsd-planner.md` (lines 100-110, philosophy section)
- **Impact:** Plans are unnecessarily fragmented; phase execution is split across more plan files than necessary. A single executor could handle an entire phase's work in one session, but planner splits into 5-7 plans.
- **Not a bug, but design constraint:** Documented in IMPROVEMENT_SPEC.md as part of "Session Autonomy" redesign. Will be addressed in Layer 2 (Parallelism-Aware Planner) implementation.

---

*Concerns audit: 2026-03-15*
