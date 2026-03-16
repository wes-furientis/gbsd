# Project Research Summary

**Project:** GBSD (Get Better Shit Done) — Parallel Agent Orchestration Enhancement
**Domain:** AI agent orchestration framework — parallel autonomous code execution with Claude Code
**Researched:** 2026-03-15
**Confidence:** HIGH

## Executive Summary

GBSD is a spec-driven AI coding orchestration framework being enhanced to support parallel autonomous execution across multiple git worktrees. The core value proposition — and the architectural challenge — is that isolation-without-orchestration is a commodity (every competitor provides it), but orchestration-with-isolation is not. The research shows that every competing tool (Superset IDE, Dagger container-use, manual worktree scripts) gives users N parallel sessions without a planner, wave structure, decision frontloading, or state reconciliation. GBSD's differentiation lives entirely in those orchestration layers. Building them in the correct order is what makes parallel execution safe rather than just fast.

The recommended approach is a dependency-ordered build: Worktree Manager first (the isolation primitive everything else rests on), then Key Pool Manager and State Reconciler (parallel with each other), then execute-phase integration to wire them together, then Pathfinder integration as an independent parallel track that enhances planning quality without blocking execution. File-ownership-first plan decomposition must land in the planner before the first parallel wave ships — without it, merge conflict cascade is nearly certain. Frontloaded decision capture must land simultaneously with wave execution — parallel agents cannot pause for human questions mid-wave. These two features (file ownership + decision frontloading) are the correctness guarantees; the worktrees and key pool are the performance mechanism.

The key risk is building in the wrong order or skipping the correctness guarantees in favor of visible speedup. A confirmed open bug (GitHub issue #33045) means Agent Teams teammate worktree isolation is silently broken as of v2.1.72 — the manual lead-managed worktree pattern is the only reliable path. Rate limits are org-scoped not key-scoped, meaning multi-key pools within one org provide zero benefit. Context window pricing is now unblocked (1M tokens at standard pricing as of March 13, 2026), removing the old arbitrary 2-3 task plan cap. The replacement sizing mechanism (decision-surface count, not token budget) must be implemented before the old cap is removed.

## Key Findings

### Recommended Stack

Claude Code v2.1.76+ is the required runtime. Agent Teams is still experimental (requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) but the underlying tools (TaskCreate, TaskUpdate, SendMessage, Bash/Read/Write tools for teammates) are stable. The recommended model tier is: Opus 4.6 for orchestration and planning (1M context, best reasoning), Sonnet 4.6 for execution (1M context, 40% cheaper than Opus), Haiku 4.5 for research and verification tasks (4x ITPM headroom vs Sonnet/Opus at Tier 4). The Node.js CommonJS runtime already in GBSD requires no new dependencies. All new subsystem logic follows the existing `gbsd/bin/lib/*.cjs` pattern.

Rate limit management requires separate Anthropic organizations, not just separate API keys. Three orgs at Tier 4 provides 6M effective ITPM per model class. With 80% prompt cache hit rate (achievable by caching static CLAUDE.md and plan files with 1-hour cache writes), effective throughput approaches 30M tokens/min — sufficient for any realistic parallel execution workload. The active constraint shifts from ITPM to RPM.

**Core technologies:**
- Claude Code v2.1.76+: primary runtime for Agent Teams, worktrees, task management — latest stable with all 2026 features
- Opus 4.6: orchestration and planning model — 1M context, GA pricing since March 13, 2026
- Sonnet 4.6: executor model — same 1M context, 40% cheaper, sufficient quality for implementation
- Haiku 4.5: research/verification model — lowest cost, 4M ITPM at Tier 4, best for high-throughput parallel tasks
- Node.js CJS 18+ LTS: GBSD runtime — zero new dependencies needed
- git worktrees: executor isolation primitive — git-native, proven, no extra tooling

### Expected Features

**Must have (table stakes) — P1, launch blockers:**
- Worktree isolation per executor (PARA-01, PARA-04) — the isolation primitive everything rests on
- Wave-level parallel execution (PARA-02) — the primary speedup mechanism
- File ownership in plan frontmatter (PLAN-01, PLAN-02) — conflict prevention before execution
- Frontloaded decision capture in discuss-phase (AUTON-01, AUTON-02) — parallel agents cannot pause for questions
- Merge-back with auto-resolution for lock files and barrel files (PARA-06) — without this, every wave requires manual resolution
- State reconciliation for STATE.md and agent-history.json (PARA-05) — planning state must survive parallel merge
- Interruption detection and executor resume (TEAM-04) — parallel execution increases failure surface

**Should have (competitive differentiators) — P2, add after P1 validates:**
- Pathfinder integration for automatic file-ownership derivation (PATH-01/02/03) — increases wave-1 parallelizability from 40-60% to 70-85%
- API key pool with circuit breaker (API-01/02/03) — needed before `max_concurrent_agents > 3`
- Decision surface sizing / remove context cap (AUTON-04) — enables larger plans once frontloading is validated
- Multi-provider model routing (API-04) — 40-60% cost reduction via tiered model assignment
- Parallelizability score in plan metadata (PLAN-04) — signals plan quality before execution

**Defer (v2+) — P3:**
- Phase-level parallelism (PARA-03) — high merge risk, high complexity; only after wave-level is proven
- Hub-and-spoke Agent Teams with lean lead (TEAM-01/02/03) — requires Pathfinder; complex architecture change
- LiteLLM cost tracking gateway (API-05) — operational overhead; add for enterprise/multi-team use only

**Anti-features (explicitly not building):**
- Real-time inter-agent communication (race conditions, destroys isolation guarantee)
- Task-level parallelism within a plan (complexity/benefit ratio too poor)
- Automatic phase parallelism detection (false positive cost too high)
- Custom UI/dashboard (terminal-native tool; ASCII progress is sufficient)

### Architecture Approach

The enhanced GBSD architecture adds four new subsystems as middleware between the existing Orchestrator layer and Tools CLI layer. The hub-and-spoke constraint is permanent by design — teammates cannot spawn subagents or create teams, so all orchestration logic must stay in the lead. Each executor agent operates in full filesystem isolation in its assigned worktree; no peer contact is possible or required. All cross-executor coordination happens through the orchestrator after wave completion, using the State Reconciler to merge diverged planning state.

**Major components:**
1. **Worktree Manager (worktree.cjs)** — creates, assigns, and tears down git worktrees; owns all `git worktree` interactions; detects stale worktrees on restart
2. **Key Pool Manager (keypool.cjs)** — round-robin API key assignment across orgs; circuit-breaker with in-memory blacklisting; never persisted to disk
3. **State Reconciler (reconcile.cjs)** — merges diverged STATE.md and agent-history.json after parallel waves using typed merge strategies per field
4. **Pathfinder Adapter (pathfinder.cjs)** — reads `.code-intel/` index for dependency queries; gracefully degrades when index is absent

Build order is a hard constraint: worktree.cjs first, then keypool.cjs and reconcile.cjs in parallel, then execute-phase.md integration wiring all three together. Pathfinder integration is an independent parallel track. Phase-level parallelism requires worktree and reconciler to be proven stable first.

### Critical Pitfalls

1. **Worktree lifecycle leaks** — partial worktree creation on error leaves orphaned branches and dirs consuming 400-700 MB each; use try/finally cleanup in every worktree operation; scan for orphaned `gsd/exec/*` branches on orchestrator startup (PARA-04 must be built first)

2. **Merge conflict cascade from shared infrastructure files** — parallel executors independently modifying package.json, barrel files, or migration timestamps each produce a conflict; solve with PLAN-01 file ownership enforcement in the planner before any parallel wave ships; designate a single Wave 1 "infra task" for shared infrastructure files

3. **Rate limit starvation under concurrent sessions** — startup burst from N sessions simultaneously triggers 429; all keys in the same Anthropic org share one rate limit pool regardless of key count; implement circuit breaker (API-03) before round-robin (API-02); stagger executor spawning by 2-3 seconds; default `max_concurrent_agents: 3`, not 5

4. **STATE.md and agent-history divergence after parallel merge** — last-write-wins for all fields silently discards decisions from losing executors; design merge strategy field-by-field before implementing PARA-05; agent_id in agent-history.json must be globally unique (include worktree name + timestamp); test reconciler against dirty inputs

5. **Agent Teams API instability breaking workarounds** — issue #33045 (teammate worktree isolation silently ignored) is open as of March 2026; abstract all worktree spawning behind `worktreeManager.spawnExecutor()` so there is one place to update when Anthropic ships the fix; pin Claude Code version in prerequisites; monitor changelog before any worktree-related update ships

6. **Pathfinder index staleness causing wrong wave assignments** — time-based freshness check fails for recent refactors; implement content-based staleness detection (`git diff --name-only HEAD~1..HEAD` vs indexed files); run freshness check at every `plan-phase` invocation; validate all `files_owned` paths exist before spawning executors

## Implications for Roadmap

Based on combined research, the dependency graph mandates a 5-6 phase structure. The ordering is not arbitrary — each phase unlocks the next, and shipping phases out of order produces a system that appears to work but fails under real conditions.

### Phase 1: Worktree Foundation and Isolation Infrastructure
**Rationale:** Everything else in this project requires reliable worktree creation and teardown. This is the critical path item with no predecessors. Without a robust Worktree Manager, parallel execution cannot start, State Reconciler has nothing to merge, and Key Pool Manager has no executors to assign keys to. This must also include the `execSync` shell injection fix documented in CONCERNS.md — parallel execution multiplies the attack surface.
**Delivers:** `worktree.cjs` with full lifecycle management; stale worktree detection on restart; worktree path outside project directory (`~/.claude/worktrees/`); try/finally cleanup on all creation paths; branch naming convention enforced; security fix for shell injection
**Addresses:** PARA-01, PARA-04
**Avoids:** Worktree lifecycle leaks (Pitfall 1), shell injection (security mistake)
**Research flag:** Standard git patterns — skip research-phase

### Phase 2: Key Pool and State Reconciliation (Parallel Track)
**Rationale:** Key Pool Manager and State Reconciler have no dependency on each other and can be implemented in parallel. Both are prerequisites for execute-phase integration in Phase 3. Key Pool Manager must implement circuit-breaker before round-robin is useful. State Reconciler must define merge strategy field-by-field before any code is written to avoid the "last-write-wins silently discards decisions" trap.
**Delivers:** `keypool.cjs` with round-robin selection, in-memory circuit-breaker, org-level key assignment; `reconcile.cjs` with typed merge strategies for each STATE.md section and agent-history.json
**Addresses:** API-01, API-02, API-03, PARA-05
**Avoids:** Rate limit starvation (Pitfall 3), STATE.md divergence (Pitfall 4)
**Research flag:** Well-documented circuit-breaker patterns — skip research-phase

### Phase 3: File Ownership Planning and Frontloaded Decisions
**Rationale:** These two features must land before the first parallel wave ships. File ownership prevents merge conflict cascade — without it, parallel executors will write to the same files and produce unresolvable conflicts. Frontloaded decision capture prevents execution-time interruptions — without it, `skip_checkpoints: true` causes plan failures rather than autonomous completion. These are the correctness guarantees; they cannot be deferred to "after parallelism works."
**Delivers:** Updated `gbsd-planner.md` with file-ownership-first decomposition; `files_owned`/`files_created`/`files_read` frontmatter fields; enhanced discuss-phase that captures all ambiguous decisions before plan creation; plan-checker validation for wave consistency and decision surface (AUTON-04 decision-count cap replaces old token-based cap)
**Addresses:** PLAN-01, PLAN-02, PLAN-03, AUTON-01, AUTON-02, AUTON-04
**Avoids:** Merge conflict cascade (Pitfall 2), context window assumption artifacts (Pitfall 8)
**Research flag:** Needs research-phase — file-ownership decomposition algorithm and decision surface sizing are novel patterns with limited documented prior art; planner redesign has high blast radius

### Phase 4: Execute-Phase Integration (Wave Execution)
**Rationale:** All prerequisites (Worktree Manager, Key Pool, State Reconciler, File Ownership) are in place. This phase wires them together into end-to-end parallel wave execution. Executors are not modified — they run unchanged in their assigned worktrees. The orchestration logic in execute-phase.md is what changes. Interruption detection and resume must ship in this phase — parallel execution increases failure surface and a system without resume is not production-ready.
**Delivers:** Updated `execute-phase.md` with wave grouping, worktree spawning, key assignment, merge-back protocol, and State Reconciler invocation; `merge-back.md` workflow; interruption detection on startup; executor resume with checkpoint boundary enforcement; resource pre-flight checks (disk space = `max_concurrent_agents × 700 MB`)
**Addresses:** PARA-02, PARA-06, TEAM-04
**Avoids:** Checkpoint blocking parallel wave (Pitfall 9), leaked worktrees from execute-phase errors
**Research flag:** Standard patterns for this project — skip research-phase (all components tested in Phases 1-3)

### Phase 5: Pathfinder Integration
**Rationale:** Pathfinder is an independent parallel track that enhances plan quality but does not block parallel execution (which shipped in Phase 4). Graceful degradation must be implemented and tested first — the "no index" path must be verified before any Pathfinder-dependent code runs. Index freshness check must use content-based detection, not time-based.
**Delivers:** `pathfinder.cjs` adapter with graceful degradation; content-based staleness detection; blast-radius validation before executor spawn; automatic `files_owned` derivation from import graph; wave assignment from file ownership graph (targeting 70-85% wave-1 parallelizability vs current 40-60%)
**Addresses:** PATH-01, PATH-02, PATH-03, PATH-04
**Avoids:** Pathfinder index staleness causing wrong wave assignments (Pitfall 6), over-engineering via hard Pathfinder coupling (Pitfall 7)
**Research flag:** Needs research-phase — Pathfinder index format and query API; content-based staleness algorithm; blast-radius query semantics

### Phase 6: Multi-Provider Routing and Cost Optimization
**Rationale:** Add after Phase 4 validates that parallel execution works and API costs become visible. Routing policy must be defined by consequence (research/verification stay on Opus) not by task name (never "route all research to Haiku"). LiteLLM is optional and can be deferred to enterprise use.
**Delivers:** Per-agent-type model config (Opus for planner/checker, Sonnet for executors, Haiku for researcher only); `api_management` config schema extension; cost tracking; multi-provider failover
**Addresses:** API-04
**Avoids:** Multi-provider model capability variance (Pitfall 10)
**Research flag:** Well-documented routing patterns — skip research-phase; LiteLLM docs are comprehensive

### Phase Ordering Rationale

- **Phases 1-2 before Phase 3:** The planner redesign (Phase 3) requires the worktree and reconciler interfaces to be defined so it can emit the right frontmatter. The correctness guarantees depend on knowing what the execution layer needs.
- **Phase 3 before Phase 4:** File ownership enforcement in the planner must exist before parallel waves run. Shipping execution without conflict prevention produces a broken system that appears to work on simple projects and fails on any project with shared infrastructure files.
- **Phases 1-4 as the critical path:** The MVP is Phase 4 completion — a working parallel wave execution system. Pathfinder (Phase 5) and multi-provider routing (Phase 6) enhance quality and reduce cost but do not define correctness.
- **Pathfinder as independent track:** Phase 5 can begin development in parallel with Phase 4 execution integration, as long as graceful degradation is verified before any Pathfinder-dependent planner code ships.

### Research Flags

Phases likely needing `/gsd:research-phase` during planning:
- **Phase 3 (File Ownership / Decision Frontloading):** Novel algorithms with limited documented prior art; planner redesign has high blast radius on downstream agents; decision-surface sizing metric needs definition
- **Phase 5 (Pathfinder Integration):** Pathfinder index format and query semantics need verification; content-based staleness detection algorithm; blast-radius query design

Phases with standard patterns (skip research-phase):
- **Phase 1 (Worktree Manager):** git worktree API is stable and well-documented; lifecycle management patterns are well-established
- **Phase 2 (Key Pool / Reconciler):** Circuit-breaker patterns are well-documented in LLM gateway literature; STATE.md merge strategy is defined in architecture docs
- **Phase 4 (Execute-Phase Integration):** All components are defined and tested; wiring is standard orchestration work
- **Phase 6 (Multi-Provider Routing):** Routing policy is defined; LiteLLM docs are comprehensive

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All findings verified against official docs, live GitHub issues, and Anthropic changelog; 1M pricing GA confirmed March 13, 2026; issue #33045 and #32731 read directly |
| Features | HIGH | Primary sources are the project's own design documents (PROJECT.md, IMPROVEMENT_SPEC.md, GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md); ecosystem competitive analysis from current web sources |
| Architecture | HIGH | All architecture findings derived from first-party GBSD design documents; component interfaces are fully specified; build order is based on explicit dependency analysis |
| Pitfalls | HIGH | Mix of first-party (CONCERNS.md, design docs) and verified external sources (GitHub issues, arXiv 2503.13657, Upsun DevCenter); shell injection bug documented in CONCERNS.md is a confirmed existing issue |

**Overall confidence:** HIGH

### Gaps to Address

- **Agent Teams v2.2 API contract:** Agent Teams is still experimental (v2.1.32+). The `isolation: "worktree"` bug fix for issue #33045 could land at any time. The abstraction layer in worktreeManager.spawnExecutor() must be in place before this happens to avoid a conflicting double-management situation. Monitor Claude Code changelog before every GBSD release.

- **Pathfinder index format versioning:** The Pathfinder adapter design assumes the `.code-intel/` schema is stable. If Pathfinder ships a breaking schema change, `pathfinder.cjs` needs an adapter version check. Verify schema stability before Phase 5 implementation begins.

- **`max_concurrent_agents` default for non-Tier-4 users:** Research confirms 3 is the safe default for Tier 4. Users on lower tiers (Tier 2: 100K ITPM, Tier 3: 500K ITPM) will saturate at fewer concurrent agents. A pre-flight rate limit tier detection or a clear documentation warning is needed. This gap should be addressed in Phase 4 config design.

- **STATE.md regex patching fragility:** CONCERNS.md documents that the current codebase uses regex-based STATE.md patching that silently fails on formatting variations. This is flagged as acceptable for MVP but must be replaced before parallel execution multiplies the failure surface. Phase 1 or Phase 2 should assess whether the regex patcher needs replacement before the reconciler is built on top of it.

## Sources

### Primary (HIGH confidence)
- [Claude Code Agent Teams docs](https://code.claude.com/docs/en/agent-teams) — tool inventory, architecture, teammate restrictions, worktree isolation status
- [Claude Code Changelog](https://code.claude.com/docs/en/changelog) — version history for all worktree and agent features
- [Anthropic API Rate Limits](https://platform.claude.com/docs/en/api/rate-limits) — exact tier tables, org-level scoping, token bucket algorithm
- [Anthropic Pricing](https://platform.claude.com/docs/en/about-claude/pricing) — 1M context GA pricing confirmed March 13, 2026
- [GitHub issue #33045](https://github.com/anthropics/claude-code/issues/33045) — `isolation: "worktree"` broken for teammates, open March 2026
- [GitHub issue #32731](https://github.com/anthropics/claude-code/issues/32731) — teammate tool restrictions confirmed March 2026
- `docs/GBSD_PARALLEL_ORCHESTRATION_ARCHITECTURE.md` — worktree lifecycle, state reconciliation, config schema
- `docs/IMPROVEMENT_SPEC.md` — four-layer improvement stack with complexity and risk assessments
- `docs/planner-redesign-analysis.md` — file-ownership-first decomposition, wave algorithm
- `docs/api-rate-augmentation-analysis.md` — key pool design, multi-org strategy, circuit-breaker
- `.planning/codebase/CONCERNS.md` — shell injection bug, regex state patching fragility, existing known issues
- `.planning/PROJECT.md` — active requirements, Pathfinder graceful degradation requirement

### Secondary (MEDIUM confidence)
- [Upsun DevCenter: Git worktrees for parallel AI coding agents](https://devcenter.upsun.com/posts/git-worktrees-for-parallel-ai-coding-agents/) — disk space estimates, port conflict gotchas, isolation challenges
- [arXiv 2503.13657: Why Do Multi-Agent LLM Systems Fail](https://arxiv.org/abs/2503.13657) — failure mode taxonomy for multi-agent systems
- [Boris Cherny Threads post](https://www.threads.com/@boris_cherny/post/DVAAnexgRUj/) — `--worktree` CLI flag announcement (social post from Anthropic engineer)
- [anomalyco/opencode GitHub issue #14648](https://github.com/anomalyco/opencode/issues/14648) — worktree bootstrap failure orphan patterns
- [Clash conflict detection tool](https://github.com/clash-sh/clash) — confirms file ownership gap in ecosystem
- [Portkey: Retries, fallbacks, and circuit breakers in LLM apps](https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/) — circuit breaker patterns for API key pools

### Tertiary (LOW confidence)
- [Nx Blog: Git Worktrees Changed My AI Agent Workflow](https://nx.dev/blog/git-worktrees-ai-agents) — developer workflow patterns; not specific to Claude Code
- [Towards Data Science: The 17x Error Trap of Bag of Agents](https://towardsdatascience.com/why-your-multi-agent-system-is-failing-escaping-the-17x-error-trap-of-the-bag-of-agents/) — over-parallelization failure modes; general multi-agent research

---
*Research completed: 2026-03-15*
*Ready for roadmap: yes*
