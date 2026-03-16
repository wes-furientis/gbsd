# Pitfalls Research

**Domain:** Parallel AI agent orchestration with worktree isolation, autonomous phase execution, and code-intelligence-driven planning
**Researched:** 2026-03-15
**Confidence:** HIGH (primary sources: project docs, GitHub issues, Anthropic engineering blog, multi-agent failure research)

---

## Critical Pitfalls

### Pitfall 1: Worktree Lifecycle Leaks

**What goes wrong:**
When a worktree bootstrap fails partway through — `git worktree add` succeeds but executor spawn throws — the partially-created worktree and its branch persist on disk indefinitely. No cleanup runs on error paths. Over a session with a 2 GB codebase, each failed+abandoned worktree consumes 400-700 MB. Three stale worktrees = 1-2 GB gone without warning. Stale locked branches block future checkout of the same branch name, causing confusing errors on retry.

**Why it happens:**
Worktree creation is a three-step sequence (create directory, `git worktree add`, set branch). Failures at any step leave partial state. Error handling that handles the happy path but ignores cleanup on early exits is the common cause. This is documented as an active issue pattern in open-source AI orchestration tools (GitHub issue pattern confirmed in anomalyco/opencode #14648).

**How to avoid:**
Use a try/finally cleanup pattern for every worktree creation. Register each worktree for cleanup before spawning. On orchestrator startup, scan for orphaned `gsd/exec/*` branches and worktrees not in the active wave manifest — any found = leaked and must be pruned or resumed. Implement a `gsd-worktree-manager` utility that wraps all worktree operations in atomic create/register/cleanup lifecycle. Never leave cleanup as "the caller's responsibility."

**Warning signs:**
- `git worktree list` shows entries not in current wave
- `.claude/worktrees/` directory growing unexpectedly between runs
- `git branch -l 'gsd/exec/*'` shows branches with no corresponding active task
- Disk usage climbing without corresponding code commits

**Phase to address:** Phase implementing PARA-04 (Worktree Manager Utility). This must be the first worktree-adjacent code written — before any parallel execution.

---

### Pitfall 2: Merge Conflict Cascade from Shared Infrastructure Files

**What goes wrong:**
Multiple parallel executors each independently add a dependency to `package.json`, create a migration file, or append to a barrel index (`src/index.ts`). Each executor's change is clean in isolation. At merge time, every one of these files has a conflict. Three executors touching `package.json` means three-way conflicts. The auto-resolution logic — union all dependencies, regenerate lock — fails if any executor pinned a conflicting version. The cascade escalates: lock file conflict blocks tests, tests block merge validation, merge validation blocks wave completion.

**Why it happens:**
Plans decomposed by feature (auth, notifications, users) rather than by file ownership naturally converge on the same infrastructure files. The planner doesn't distinguish "this plan writes to package.json" from "this plan logically depends on a package." Without explicit file ownership signals, executors have no reason to coordinate.

**How to avoid:**
Implement PLAN-01 (file ownership as first-class planning signal) before any parallel execution lands. Each plan's frontmatter must declare `files_owned` (exclusive write access) and `files_read` (shared read-only). Two plans in the same wave cannot both own the same file — the planner must enforce this as a hard constraint, not a guideline. For truly shared infrastructure files (package.json, migration timestamps, barrel exports), designate a single "infra task" that owns them and runs in Wave 1 alone, so all parallel tasks in Wave 2 already have their dependencies resolved.

**Warning signs:**
- Plans in the same wave that both modify `package.json`, `**/index.ts`, or migration files
- Merge producing `<<<<<<< HEAD` markers in lock files after first parallel wave
- Tests failing post-merge but passing on individual executor branches

**Phase to address:** Phase implementing PLAN-01/PLAN-02 (file ownership and hard vs soft dependencies). Must be solved in planner before wave parallelism ships.

---

### Pitfall 3: Rate Limit Starvation Under Concurrent Parallel Sessions

**What goes wrong:**
Five executor sessions start simultaneously. Each fires 8+ parallel startup API requests within 20 ms (documented Claude Code behavior). The burst triggers a 429 before any session completes its first task. Even with round-robin key distribution across a pool, all sessions on the same org share the same rate limit bucket. Exponential backoff from 5 sessions compounds: session A backs off, session B backs off with an offset, but they both retry at similar times and trigger another 429 cycle. The wave stalls for 5-10 minutes despite having capacity.

**Why it happens:**
Anthropic rate limits are org-scoped, not key-scoped. Multiple keys under the same org share a single rate limit. Session startup is spike-heavy (not smooth). Round-robin key distribution assumes independent limits but doesn't provide them unless keys are from different orgs (API-01 requirement: multi-org, not multi-key within one org).

**How to avoid:**
Implement API-03 (circuit breaker) before API-02 (round-robin): detect 429 responses, blacklist the triggering key for the retry-after interval, route to another org's key. Stagger executor spawning by 2-3 seconds between sessions to avoid the startup burst. Set `max_concurrent_agents` conservatively (3, not 5) as the default, with documentation explaining why. For multi-org pools: document that keys must be from different Anthropic organizations with separate billing to actually get independent limits.

**Warning signs:**
- `anthropic-ratelimit-requests-remaining: 0` headers in executor output
- Wave ETA climbing instead of shrinking after agents spawn
- All executors reporting idle time simultaneously
- 429 errors appearing within the first 60 seconds of a wave start

**Phase to address:** Phase implementing API-01/API-02/API-03 (rate augmentation). Circuit breaker must land before increasing `max_concurrent_agents` above 3.

---

### Pitfall 4: STATE.md and Agent History Divergence After Parallel Merge

**What goes wrong:**
Three executors each update STATE.md in their worktree. The reconciliation algorithm uses `last-write-wins` for some sections and `union` for others. After merge: the "Current Position" shows the last executor's plan number (not the correct aggregate), the "Decisions" section has duplicates because two executors wrote the same ambient decision, and `agent-history.json` has entries with identical `agent_id` values from different worktrees that `unique_by(.agent_id)` silently deduplicates — losing one executor's history entirely.

**Why it happens:**
Reconciling mutable state from N independent writers requires a precise merge strategy for every field in every document. Ad-hoc `sed` + `grep` reconciliation gets the common cases right but fails on edge cases: duplicate content, non-deterministic field ordering, fields that are additive vs. last-write. The reconciliation code is tested against clean inputs, not against the messy real-world outputs agents produce.

**How to avoid:**
Design the merge strategy field-by-field before implementing PARA-05 (state reconciliation). Document which fields are: immutable (read from main, never touched by executors), append-only (union on merge, deduplicate by hash not by field value), last-write (only most recent executor's value kept), and aggregated (sum, max, or average depending on field semantics). `agent_id` in agent-history.json must be globally unique per run — include worktree name and timestamp in the ID, not just plan number. Test reconciliation against dirty inputs: out-of-order writes, duplicate decision lines, truncated JSON.

**Warning signs:**
- STATE.md showing plan count smaller than the number of completed plans
- `agent-history.json` with fewer entries after merge than before
- Decisions section containing duplicate lines after wave completion
- Progress metrics showing regression after wave completes

**Phase to address:** Phase implementing PARA-05 (state reconciliation algorithm). Must be fully tested before phase-level parallelism (which creates far larger reconciliation surface).

---

### Pitfall 5: Agent Teams API Instability Breaking Workarounds

**What goes wrong:**
GBSD adopts the `isolation: "worktree"` workaround (spawn independent `claude` CLI processes) because Agent Teams teammates silently ignore the parameter (GitHub issue #33045). Anthropic ships a fix for #33045 in a future Claude Code version. GBSD's workaround now conflicts with the official implementation: two worktree management systems fight over the same `.claude/worktrees/` directory. Alternatively, Anthropic changes the CLI spawning API in a minor version bump, breaking the subprocess invocation pattern GBSD depends on.

**Why it happens:**
Agent Teams is a research preview (introduced 2026-02-05 in v2.1.32) with explicit API instability warnings. GBSD is building production workflows on top of a moving target. The correct workaround today may become incorrect next month. There is no versioning contract for Claude Code CLI flags.

**How to avoid:**
Implement ADOPT-01 (rapid Agent Teams API change tracking) as a scheduled concern, not an afterthought. Abstract the worktree spawning behind a single `worktreeManager.spawnExecutor()` function — one place to update when the API changes. Pin Claude Code version in project prerequisites documentation and add an explicit "upgrade checklist" that verifies worktree behavior hasn't changed. Monitor the Claude Code changelog (specifically the Agent Teams section) before any worktree-related update ships.

**Warning signs:**
- Claude Code release notes mentioning "worktree", "isolation", or "agent teams" changes
- `isolation: "worktree"` starting to have an effect (which would be the bug fix, but would conflict with GBSD's manual management)
- Executor processes reporting different working directories than expected

**Phase to address:** Phase implementing PARA-01/PARA-04 (worktree isolation + manager). Design the abstraction layer first; don't let the workaround leak into every workflow that spawns executors.

---

### Pitfall 6: Pathfinder Index Staleness Causing Wrong Wave Assignments

**What goes wrong:**
The planner reads `.code-intel/module_map.yaml` and uses it to assign tasks to waves based on import dependencies. The index was generated before a recent refactor that moved `src/auth/service.ts` into `src/features/auth/service.ts`. The old index still shows the original path. The planner doesn't detect the drift, assigns two tasks to Wave 1 as "independent," but they both actually modify the same now-shared file at its new location. Wave 1 executes in parallel, both executors write to the same file in different worktrees, merge conflict on a file neither plan declared as shared.

**Why it happens:**
Index freshness is a timestamp check, but "freshness" is relative to the number of structural changes since last index, not wall-clock time. A project that had zero refactors for 3 days has a "stale" index by time but an accurate one by content. A project that moved 20 files in the last 10 minutes has a "fresh" index by time but a wrong one. Time-based freshness checks are a heuristic that can fail in both directions.

**How to avoid:**
Implement PATH-04 (Pathfinder index freshness management) with content-based staleness detection, not just time-based. Check `git diff --name-only HEAD~1..HEAD` against the set of files in the current index — if any indexed file was moved, renamed, or deleted since last indexing, force a re-index. Run freshness check at the start of every `plan-phase` invocation, not just the mapper. Add a "blast radius validation" step after planning that cross-references PLAN.md file lists against the live filesystem — if any `files_owned` path doesn't exist, abort with an error before spawning executors.

**Warning signs:**
- Module paths in PLAN.md frontmatter pointing to files that don't exist
- Wave 1 plans having overlapping files when the planner said they were independent
- Dependency graph suggesting a file has no importers when grep shows it's imported everywhere

**Phase to address:** Phase implementing PATH-04 and PLAN-01 together. File ownership validation must run at plan creation time, not at execution time.

---

## Moderate Pitfalls

### Pitfall 7: Over-Engineering the Planner with Full Pathfinder Coupling

**What goes wrong:**
The planner is redesigned to require Pathfinder index for all operations. Projects without a `.code-intel/` directory — new projects, greenfield starts, or projects where Pathfinder hasn't been run — fail entirely with "index not found." The graceful degradation path ("work without the index") is not implemented because the Pathfinder integration was built first and worked so well that the fallback was never prioritized.

**Why it happens:**
Tight coupling is the natural outcome when building integration before building fallback. Once the planner uses Pathfinder for dependency detection, task clustering, and conflict prediction, removing Pathfinder means replacing all three with manual alternatives. The fallback becomes a full reimplementation of the old planner.

**How to avoid:**
Build the Pathfinder integration as a progressive enhancement: planner works correctly without index (same as today), works better with index (automatic wave assignment), works best with index + fresh analysis (conflict prediction). The constraint from PROJECT.md is explicit: "graceful degradation — everything must work (slower) without .code-intel/". Implement and test the degraded path first, then layer in the enhanced path.

**Warning signs:**
- Planner code that throws if `module_map.yaml` is missing
- Integration tests that all require an existing `.code-intel/` directory
- No test coverage for "index not found" path

**Phase to address:** Phase implementing PATH-01/PATH-02 (Pathfinder planner integration). Write the "no index" test suite before writing any Pathfinder-dependent code.

---

### Pitfall 8: Context Window Assumption Artifacts in Hardcoded Limits

**What goes wrong:**
The planner still contains `// 50% context target` comments and `max: 3 tasks per plan` caps that were calibrated for Claude 3.5 Sonnet's 200k context. The IMPROVEMENT_SPEC.md documents this as a known limitation. When these limits are removed as part of AUTON-03/04, plans become too large — not because of context limits, but because large plans increase the decision surface, which contradicts AUTON-04 ("size by decision count, not token budget"). The 200k-era caps were a proxy for "manageable work unit." Removing them without replacing with decision-surface sizing produces plans that run for 3+ hours and fail late.

**Why it happens:**
Legacy constraints are removed without understanding why they existed. The 2-3 task cap wasn't arbitrary — it was a heuristic for "one context session worth of focused work." The correct fix is replacing the heuristic with a better one (decision count), not removing the constraint entirely.

**How to avoid:**
Before removing any sizing constraint, document: (1) what problem it was solving, (2) the new mechanism that replaces it, and (3) how to verify the replacement is working. For the task-count cap: the replacement is decision-surface sizing — count decision points (tasks requiring human input or ambiguous outcomes) per plan, cap at 3-5 decision points, not 3-5 tasks. A plan can have 10 tasks if 9 are deterministic and only 1 requires a human verify.

**Warning signs:**
- Plans with 8+ tasks all marked as "auto-resolvable"
- Executors running for more than 90 minutes without a commit
- Wave completion time increasing instead of decreasing after parallelism lands

**Phase to address:** Phase implementing AUTON-03/AUTON-04 (context window and decision surface sizing). Must define the replacement metric before removing the old cap.

---

### Pitfall 9: Checkpoint Blocking Entire Wave on a Single Agent Pause

**What goes wrong:**
Three executors are running Wave 1 in parallel. Executor B hits a `human-verify` checkpoint at task 2/8. Executor A and C are at task 6/8 and 5/8 respectively. The orchestrator's "atomic checkpoint" policy pauses all agents. Executor A has 2 tasks of uncommitted work buffered in its context. When it pauses, the state is ambiguous: did task 6 complete and commit? Was task 7 partially done? On resume, executor A restarts from task 6 — duplicating already-committed work or producing a duplicate commit.

**Why it happens:**
"Pause all agents when any hits a checkpoint" is the safe conservative policy. But "pause" for an async process that doesn't have a synchronous pause signal means "kill the process and restart later." Unlike a synchronous pausing mechanism, `claude -p` subprocesses either run to completion or crash — there's no mid-task suspend. The orchestration layer assumes suspend is clean; it isn't.

**How to avoid:**
Design checkpoints as wave-level synchronization points, not mid-task interrupts. The safe atomic unit for a checkpoint is a completed task with a commit — not a task in progress. Executors should only report a checkpoint at task boundaries (after the previous task committed, before the next task starts). The `skip_checkpoints: true` default in config is the correct setting for parallel waves — the policy for parallel execution should be "auto-resolve checkpoints or fail the plan" not "pause and wait."

**Warning signs:**
- Checkpoints reported mid-task in executor output
- Duplicate commits in executor branch after resume
- Task count in SUMMARY.md not matching commits in branch history

**Phase to address:** Phase implementing TEAM-02/TEAM-04 (checkpoint protocol and executor resume). Define "valid checkpoint positions" as a constraint on the executor agent prompt.

---

### Pitfall 10: Multi-Provider Model Capability Variance

**What goes wrong:**
API-04 routes "research" tasks to Haiku or Gemini Flash for cost reduction. A researcher agent using Haiku produces a RESEARCH.md that recommends a library not available in the target runtime. The planner (on Opus) reads the research output and faithfully implements it. The executor (on Sonnet) discovers the library doesn't exist at task runtime. The error surfaces 4 phases later, requiring a rewrite of research, plan, and implementation artifacts.

**Why it happens:**
Different models have different knowledge cutoffs, different reasoning quality on domain-specific questions, and different hallucination rates on specific topics. Haiku and Gemini Flash are cheaper because they make different tradeoffs. Using them for research — where accuracy matters most and errors are amplified downstream — is the wrong tradeoff.

**How to avoid:**
Route by consequence, not by task name. Research agents feed every downstream agent; their output errors are amplified. Verification agents must catch what executors miss; their misses become shipped bugs. These two roles should stay on the highest-quality model. Only route to cheaper models for tasks where errors are isolated and immediately verifiable: syntax checks, formatting, simple file reads, progress reporting.

**Warning signs:**
- Research outputs referencing library versions that don't exist in npm/PyPI
- Verification passing on code that obviously doesn't satisfy acceptance criteria
- Planner outputs that contain clearly hallucinated file paths

**Phase to address:** Phase implementing API-04 (multi-provider routing). Define routing policy before implementation — "cheapest available" is the wrong default.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| `last-write-wins` for STATE.md conflict resolution | No reconciliation code needed | Silently discards decisions from losing executor; history gaps | Never for Decisions section; acceptable for Current Position |
| Hardcoded `max_concurrent_agents: 3` without runtime check | Simple config | Rate limit failures on Pro plan where limits are lower | Only if there's an explicit disk/memory/rate pre-check that can lower the value |
| Skip Pathfinder freshness check for speed | Faster planning startup | Stale index produces wrong wave assignments, merge conflicts at execution | Never during plan-phase; acceptable during mapper-only runs |
| `isolation: "worktree"` passed to Agent Teams even though it's silently ignored | Future-proof when bug is fixed | Creates false confidence that worktree isolation is active when it isn't | Never; the workaround must be explicit, not cargo-culted |
| Regex-based STATE.md patching (current codebase) | Zero new dependencies | Silently fails on formatting variations, chains of replace() can corrupt state | Acceptable for MVP; must be replaced before parallel execution multiplies the failure surface |
| Storing API keys in `config.json` as plaintext | Simple round-robin distribution | Keys exposed in git history if config.json committed; leaked in worktree copies | Never; require environment variable injection per executor session |

---

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Claude Code Agent Teams | Using `isolation: "worktree"` and assuming it works | Verify against current Claude Code version; use manual `git worktree add` + `claude -p --cwd` as the primary mechanism until #33045 is confirmed closed |
| Anthropic API key pool | Putting multiple keys from one org and expecting independent rate limits | Multi-org only: keys must be from different Anthropic organizations with separate billing to get independent rate limit buckets |
| Pathfinder index | Calling `pathfinder generate` once at project init and never again | Re-index on every `plan-phase` call with content-based staleness detection (changed files since last index) |
| `git worktree add` | Creating worktrees inside the main repo directory (`.git/worktrees/`) | Create worktrees as siblings: `../repo-exec-01/` or under a dedicated `../.gbsd-worktrees/` to avoid confusing git operations |
| Executor subprocess `ANTHROPIC_API_KEY` | Inheriting parent process key for all executors | Explicitly set `ANTHROPIC_API_KEY=<pool_key>` per subprocess invocation; never rely on inheritance when round-robin distribution is needed |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Synchronous `fs.readFileSync` on all file operations (current codebase) | CLI hangs 2-3 seconds on large projects | Profile startup time; migrate to async fs for >5 file reads | Projects with 100+ phase directories or ROADMAP.md >5MB |
| `git worktree list` scan on every orchestrator tick to detect stale worktrees | Orchestrator slows as wave size grows | Cache worktree manifest; only re-scan on explicit `--resume` or startup | 10+ concurrent worktrees |
| Full regex match on ROADMAP.md for every state update | ROADMAP.md writes slow down linearly | Implement section-based loading: read/write only the modified section | ROADMAP.md > 2 MB (multi-milestone projects) |
| Spawning N executors with no disk space pre-check | Worktree creation fails at agent 4/5 mid-wave, leaving 3 leaked worktrees | Check available disk = `max_concurrent_agents × 700 MB` before spawning any | Projects with git history > 500 MB on disk < 3 GB free |
| `jq -s` union merge for large agent-history.json | Reconciliation slows as history grows | Stream-based merge or use SQLite for history storage | agent-history.json > 10 MB (projects with 1000+ plan completions) |

---

## Security Mistakes

Domain-specific security issues beyond general web security.

| Mistake | Risk | Prevention |
|---------|------|------------|
| API keys in `config.json` (plaintext, committed to git) | Keys exposed in git history; present in all worktree copies which may persist on disk | Require API keys via environment variable injection only; add `config.json` to `.gitignore` if it contains keys; document this explicitly |
| Worktree directories inheriting world-readable permissions | Other users on shared dev machines can read in-progress code and API responses | Set worktree parent directory permissions to 700 on creation |
| Shell injection in git arguments via `execSync` (current codebase: BUG noted in CONCERNS.md) | Crafted branch names or file paths could execute arbitrary shell commands | Migrate to `child_process.execFile()` for all git operations before parallel execution multiplies the attack surface; this is already documented as a fix in CONCERNS.md |
| Executor agents receiving full API key pool in context | Agents could leak keys in summaries or tool calls | Pass only the assigned key per executor; never pass the full pool |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Worktree parallelism:** "Plans are running in parallel" — verify worktrees are actually isolated branches (not just background processes in the same working directory). `git worktree list` must show N separate entries.
- [ ] **State reconciliation:** "STATE.md is updated after the wave" — verify the Decisions section contains entries from ALL executors, not just the last one merged. Count decision lines before and after reconciliation.
- [ ] **Rate limit distribution:** "We have N API keys" — verify keys are from different Anthropic orgs (different org IDs in the key prefix), not just different keys under the same billing account.
- [ ] **Pathfinder integration:** "The planner is using the index" — verify the index was regenerated after the most recent commit that changed file structure. Check `stat .code-intel/module_map.yaml` vs `git log --format=%ai -1`.
- [ ] **Auto-resolve merge conflicts:** "Lock file conflicts resolve automatically" — verify the auto-resolver handles 3-way conflicts (not just 2-way). Test with Wave 1 having 3 executors all adding different packages.
- [ ] **Graceful degradation:** "The system works without Pathfinder" — run `plan-phase` with `.code-intel/` deleted or absent. Planning must complete, just with manual dependency declarations.
- [ ] **Executor resume:** "Interrupted executors can be resumed" — simulate an interrupt by killing an executor mid-task. Verify the resume flow finds the correct checkpoint and doesn't duplicate committed work.
- [ ] **Multi-org rate limits:** "Round-robin key pool is working" — during a 3-executor wave, check that each executor is actually hitting a different org's rate limit bucket (different `anthropic-organization-id` in response headers).

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Leaked worktrees | LOW | `git worktree list` to find orphans; `git worktree remove --force <path>` then `git branch -d <branch>` for each; verify disk space recovered |
| Merge conflict cascade | MEDIUM | `git merge --abort`; manually resolve conflicting infrastructure files; re-run wave with fixed plans that separate infrastructure ownership |
| Rate limit starvation mid-wave | LOW | Wait for `retry-after` window; reduce `max_concurrent_agents` for this session; re-trigger wave with `--resume` |
| STATE.md divergence post-merge | MEDIUM | Restore last good STATE.md from git history: `git show HEAD~1:.planning/STATE.md > .planning/STATE.md`; manually apply missing decisions from executor branch logs; commit restored state |
| Pathfinder index staleness causing wrong wave | HIGH | Identify which tasks had wrong file assignment; merge the conflicting branches manually (content is correct, file ownership was wrong); re-run `pathfinder generate --force`; do not re-run the plans (data is committed, just re-assign next time) |
| Agent Teams API breaking workaround | MEDIUM | Pin Claude Code version in shell profile until fix is validated; update `worktreeManager.spawnExecutor()` abstraction; test with a single-executor wave before enabling parallel |
| Over-sized plan running autonomously too long | HIGH | Kill executor; inspect committed tasks; create gap-closure plan for remaining tasks; never restart the original plan (produces duplicates) |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Worktree lifecycle leaks | PARA-04 (Worktree Manager) | `git worktree list` shows 0 entries after wave completion; disk space check before/after |
| Merge conflict cascade | PLAN-01/PLAN-02 (File Ownership) | Wave with 3 parallel executors on shared-infrastructure project produces 0 merge conflicts |
| Rate limit starvation | API-01/API-03 (Multi-org pool + circuit breaker) | 3 concurrent executors run to completion without 429 errors |
| STATE.md divergence | PARA-05 (State Reconciliation) | Wave with 3 executors produces STATE.md with decisions from all 3 |
| Agent Teams API instability | ADOPT-01 (API change tracking) + PARA-04 abstraction | Single-point update path: changing `spawnExecutor()` is sufficient to adapt to upstream changes |
| Pathfinder index staleness | PATH-04 (Index freshness management) | `plan-phase` after a file rename produces plans with correct file paths (not stale paths) |
| Context window assumption artifacts | AUTON-03/AUTON-04 (decision-surface sizing) | Plans sized by decision count (≤5 decision points) not task count; verified via plan-checker |
| Checkpoint blocking parallel wave | TEAM-02/TEAM-04 (checkpoint protocol) | Executor checkpoints only occur at task boundaries; no duplicate commits after resume |
| Multi-provider model variance | API-04 (routing policy) | Research and verification agents stay on Opus; cost reduction only on isolated/verifiable tasks |
| API keys in plaintext config | API-01/API-02 implementation | `config.json` has no `api_key` fields; keys only in environment; config safe to commit |

---

## Sources

- `docs/reference_agent_teams_worktree.md` — Confirmed Agent Teams `isolation: "worktree"` bug (GitHub issue #33045); teammate tool restrictions (#32731)
- `docs/GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md` — Worktree lifecycle design, state reconciliation algorithm, rate limit architecture
- `docs/phase-parallel-analysis.md` — Schema synchronization risks, merge conflict cascade analysis, dependency detection failures
- `docs/planner-pathfinder-synthesis.md` — Pathfinder staleness risk (Risk 1), over-clustering, wrong module coupling analysis
- `.planning/codebase/CONCERNS.md` — Shell injection via execSync, regex-based state patching fragility, frontmatter round-trip corruption, config write without atomic guarantees
- `.planning/PROJECT.md` — Pathfinder graceful degradation requirement, Agent Teams research preview status, multi-org rate limit architecture
- [anomalyco/opencode GitHub issue #14648](https://github.com/anomalyco/opencode/issues/14648) — Worktree bootstrap failures leaving orphaned directories
- [Upsun DevCenter: Git worktrees for parallel AI coding agents](https://devcenter.upsun.com/posts/git-worktrees-for-parallel-ai-coding-agents/) — Shared database/cache isolation gotchas, disk space estimates
- [Why Do Multi-Agent LLM Systems Fail? (arXiv 2503.13657)](https://arxiv.org/abs/2503.13657) — 14 failure modes in multi-agent systems; inter-agent misalignment taxonomy
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Context window management best practices
- [Claude Code Rate Limits Explained](https://www.sitepoint.com/claude-code-rate-limits-explained/) — 429 behavior, startup burst requests, weekly limit structure
- [TechCrunch: Anthropic rate limits for Claude Code power users](https://techcrunch.com/2025/07/28/anthropic-unveils-new-rate-limits-to-curb-claude-code-power-users/) — Weekly rate limit introduction context
- [DEV: How We Built True Parallel Agents With Git Worktrees](https://dev.to/getpochi/how-we-built-true-parallel-agents-with-git-worktrees-2580) — State reconciliation patterns, agent isolation lessons
- [Towards Data Science: The 17x Error Trap of "Bag of Agents"](https://towardsdatascience.com/why-your-multi-agent-system-is-failing-escaping-the-17x-error-trap-of-the-bag-of-agents/) — Over-parallelization failure modes

---
*Pitfalls research for: GBSD parallel agent orchestration with worktree isolation*
*Researched: 2026-03-15*
