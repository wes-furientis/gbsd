# Stack Research

**Domain:** AI orchestration framework (Claude Code meta-prompting / spec-driven development)
**Researched:** 2026-03-15
**Confidence:** HIGH — all findings verified against official docs, live GitHub issues, and official changelog

---

## Executive Summary

This research covers the current state of Claude Code's Agent Teams API, worktree support, model
context windows, and Anthropic rate limits as of March 2026. The key findings are:

1. Agent Teams shipped February 5, 2026 (Claude Code v2.1.32) — still experimental, still requires
   `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.
2. `isolation: "worktree"` for Agent Teams **teammates** is **still broken** (GitHub issue #33045,
   open as of 2026-03-13, confirmed on v2.1.72). The workaround documented in
   `docs/reference_agent_teams_worktree.md` remains valid.
3. Worktree isolation for **subagents** (Task tool) works correctly and is fully supported as of
   v2.1.49 (February 2026).
4. The CLI `--worktree` flag was added in v2.1.49 (February 20, 2026) — parallel independent
   sessions with automatic worktree creation.
5. Opus 4.6 and Sonnet 4.6 have a 1M context window at standard pricing — no long-context premium
   (GA announced March 13, 2026).
6. Rate limits are org-level, not key-level. Multi-org is the correct strategy for rate limit
   augmentation.

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Claude Code (Anthropic) | v2.1.76+ | Primary runtime — Agent Teams, worktrees, tasks | Latest stable; includes all 2026 agent and worktree features |
| Claude Opus 4.6 | Current | Lead orchestrator model | 1M context, best reasoning, $5/$25 per MTok — affordable at 1M thanks to GA pricing |
| Claude Sonnet 4.6 | Current | Executor model | 1M context, faster/cheaper than Opus, $3/$15 per MTok — right tier for implementation work |
| Claude Haiku 4.5 | Current | Research/verification tasks | Lowest cost ($1/$5 per MTok), 4M ITPM at Tier 4 — ideal for high-throughput parallel work |
| Node.js (CommonJS) | 18+ LTS | GBSD runtime | Already the GBSD runtime — zero new dependencies needed |
| git worktrees | (git builtin) | Executor isolation | Filesystem isolation for parallel executors; git-native, no extra tooling |

### Agent Teams Architecture

| Component | Tool / Mechanism | Status | Notes |
|-----------|-----------------|--------|-------|
| Team creation | `TeamCreate` (lead only) | Stable | Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| Teammate spawn | `TeammateTool` / natural language | Stable | Lead spawns; no nesting |
| Task list | `TaskCreate`, `TaskUpdate`, `TaskList` | Stable | Supports `blockedBy` dependency chains |
| Mailbox messaging | `SendMessage`, `broadcast` | Stable | Async delivery; no polling required |
| Worktree isolation (subagents) | `isolation: "worktree"` in Task tool | WORKING — v2.1.49+ | Correctly creates isolated worktrees for subagents |
| Worktree isolation (teammates) | `isolation: "worktree"` in agent definition | BROKEN — issue #33045 | Silently ignored; use lead-managed worktree workaround |
| Lead CLI worktrees | `claude --worktree` / `git worktree add` | WORKING — v2.1.49+ | `--worktree` flag or manual `git worktree add` for lead to manage teammate isolation |

### Tool Inventory (Teammate Restrictions)

As of v2.1.72, teammates have a restricted tool set compared to the lead or standalone subagents.
This is confirmed by GitHub issue #32731 (verified March 9, 2026).

| Tool Category | Lead / Standalone | Teammate | Notes |
|---------------|------------------|----------|-------|
| Agent (spawn subagents) | YES | NO | Architecture is hub-and-spoke only |
| TeamCreate / TeamDelete | YES | NO | Only lead can manage teams |
| CronCreate / CronDelete / CronList | YES | NO | Not available to teammates |
| AskUserQuestion | YES | NO | Must route through lead |
| EnterPlanMode / ExitPlanMode | YES | YES (lead-controlled) | Lead can require plan approval before teammate implements |
| TaskCreate / TaskUpdate / TaskList | YES | YES | Shared task list — teammates can claim/update tasks |
| SendMessage | YES | YES | Teammates can message lead and each other |
| Bash, Read, Write, Edit tools | YES | YES | Full filesystem access in their working directory |
| All standard coding tools | YES | YES | WebFetch, WebSearch, Glob, Grep, etc. |

**Implication for GBSD:** The hub-and-spoke constraint is permanent by design. GBSD's planned
`TEAM-01` architecture (lead stays lean, delegates to specialists) is the correct model. The lead
must NOT delegate orchestration logic to teammates — they cannot re-delegate.

### Supporting Libraries / Tools

| Library | Purpose | When to Use |
|---------|---------|-------------|
| LiteLLM (optional) | Unified gateway for multi-provider routing, cost tracking | Needed for `API-04` (multi-provider) and `API-05` (cost tracking). Self-hosted or proxy mode. |
| tmux | Split-pane display for Agent Teams | Optional; Agent Teams work in in-process mode without tmux. Use for observability during development. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| Claude Code v2.1.76+ | Primary development environment | `--worktree` flag, Agent Teams, hooks |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | Enable Agent Teams | Must be set in environment or `settings.json` |
| `claude --worktree` | Spawn isolated parallel session | Creates git worktree + starts Claude in it; use for lead-managed executor isolation |

---

## Model Context Windows and Pricing

### Current Model Tiers (March 2026)

| Model | Context Window | Input ($/MTok) | Output ($/MTok) | Batch Input | Batch Output |
|-------|---------------|----------------|-----------------|-------------|--------------|
| Claude Opus 4.6 | 1,000,000 tokens | $5.00 | $25.00 | $2.50 | $12.50 |
| Claude Sonnet 4.6 | 1,000,000 tokens | $3.00 | $15.00 | $1.50 | $7.50 |
| Claude Haiku 4.5 | (standard) | $1.00 | $5.00 | $0.50 | $2.50 |

**Critical pricing note:** As of March 13, 2026, the 1M context window is GA for Opus 4.6 and
Sonnet 4.6 at **standard per-token pricing** — a 900K-token request costs the same per-token rate
as a 9K request. There is no long-context premium for these models.

**Implication for GBSD:** Plans should be sized by decision surface (number of human-input points),
not by token budget. Sessions with 500K tokens of codebase context are economically viable. The
`AUTON-03` and `AUTON-04` requirements are now unblocked by pricing.

### Prompt Caching

Prompt caching reduces effective rate limit consumption significantly. Cache reads do NOT count
toward ITPM limits (for Opus 4.6 and Sonnet 4.6). With 80% cache hit rate, effective throughput
is 5x the stated ITPM limit.

| Cache operation | Cost multiplier | Duration |
|-----------------|-----------------|----------|
| 5-minute cache write | 1.25x base input | 5 minutes |
| 1-hour cache write | 2x base input | 1 hour |
| Cache read (hit) | 0.1x base input | Same as preceding write |

GBSD should instruct executors to use the 1-hour cache write for static context (CLAUDE.md, plan
files, project context) to maximize cache hit rates across long parallel execution sessions.

---

## Rate Limits

### Limit Structure

Rate limits are **org-level**, not key-level. Multiple API keys in the same org share one limit
pool. The multi-org strategy (requirement `API-01`) is the correct approach for rate limit
augmentation — separate Anthropic orgs have fully independent limits.

Rate limiting uses the **token bucket algorithm** — capacity replenishes continuously, not at fixed
intervals. Burst limits apply: a 4,000 RPM limit effectively caps at ~66 RPS.

Rate limits are applied **separately per model class** — Opus and Sonnet limits are independent.

### Tier 4 Limits (Standard Self-Service Maximum)

Tier 4 requires $400 cumulative API credit purchase.

| Model | RPM | ITPM | OTPM |
|-------|-----|------|------|
| Claude Opus 4.x (combined across 4.6/4.5/4.1/4) | 4,000 | 2,000,000 | 400,000 |
| Claude Sonnet 4.x (combined across 4.6/4.5/4) | 4,000 | 2,000,000 | 400,000 |
| Claude Haiku 4.5 | 4,000 | 4,000,000 | 800,000 |

**Note:** Opus 4.x is a combined rate limit across all Opus 4.x models. Running Opus 4.6 and
Opus 4.5 in the same org shares this pool.

### Tier Thresholds

| Tier | Required Credit Purchase | Monthly Spend Cap |
|------|--------------------------|-------------------|
| Tier 1 | $5 | $100 |
| Tier 2 | $40 | $500 |
| Tier 3 | $200 | $1,000 |
| Tier 4 | $400 | $200,000 |
| Enterprise / Monthly Invoicing | Contact sales | No cap |

### Multi-Org Rate Augmentation (API-01 Strategy)

With 3 separate Anthropic orgs at Tier 4, effective limits are:

| Model | Effective RPM | Effective ITPM | Effective OTPM |
|-------|--------------|----------------|----------------|
| Opus 4.6 (3 orgs) | 12,000 | 6,000,000 | 1,200,000 |
| Sonnet 4.6 (3 orgs) | 12,000 | 6,000,000 | 1,200,000 |
| Haiku 4.5 (3 orgs) | 12,000 | 12,000,000 | 2,400,000 |

With prompt caching (80% hit rate), effective Sonnet ITPM across 3 orgs approaches 30M tokens/min.
This is more than sufficient for GBSD's parallel executor model.

### Cache-Aware ITPM Calculation

For Opus 4.6 and Sonnet 4.6, `cache_read_input_tokens` do NOT count toward ITPM. This means:

- 2M ITPM limit + 80% cache hit rate = effectively 10M total input tokens/min
- GBSD's static context (CLAUDE.md, plan files) should always be cached
- The active constraint for parallel execution shifts from tokens to RPM

---

## Worktree Isolation: Current State and Recommended Approach

### What Works (as of March 2026)

| Mechanism | Status | Notes |
|-----------|--------|-------|
| `git worktree add` (manual) | WORKING | Always reliable — git builtin |
| `claude --worktree` CLI flag | WORKING (v2.1.49+) | Auto-creates worktree, starts Claude in it |
| `isolation: "worktree"` in Task tool (subagents) | WORKING (v2.1.49+) | Declarative; correct for subagent patterns |
| `isolation: "worktree"` in agent definition (teammates) | BROKEN — issue #33045 | Silently ignored as of v2.1.72 (March 2026) |
| `worktree.sparsePaths` setting | WORKING (v2.1.76+) | Monorepo optimization — checkout only needed dirs |
| Stale worktree cleanup | WORKING (v2.1.76+) | Auto-cleanup after interrupted parallel runs |
| `WorktreeCreate` / `WorktreeRemove` hooks | WORKING (v2.1.50+) | Custom VCS setup/teardown |

### Recommended Pattern for GBSD Executor Isolation

Since `isolation: "worktree"` for teammates is still broken, GBSD should use the
**lead-managed worktree pattern** (already documented in `docs/reference_agent_teams_worktree.md`):

```bash
# Lead creates worktrees before spawning executors
git worktree add .claude/worktrees/exec-01 -b gsd/exec/03-01-w1
# Lead passes worktree path in spawn message to executor teammate
# Executor runs: cd .claude/worktrees/exec-01 && work
# Lead merges and cleans up after executor signals completion
```

This workaround is architecturally sound and does not depend on the broken `isolation: "worktree"`
parameter. Monitor issue #33045 for a fix — if it lands, GBSD can simplify to declarative
teammate isolation.

**ADOPT-01/02 trigger:** When issue #33045 is closed with fix, update agent definitions to use
`isolation: "worktree"` declaratively and remove manual worktree management from lead logic.

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Parallel isolation | Lead-managed `git worktree add` | `isolation: "worktree"` in teammate def | Broken as of v2.1.72 (issue #33045) |
| Rate augmentation | Multiple Anthropic orgs (multi-org) | Multiple API keys in one org | Keys in same org share the same rate limit pool — no benefit |
| Cost tracking | LiteLLM self-hosted proxy | Custom accounting in GBSD | LiteLLM provides vendor-neutral gateway with usage tracking; GBSD stays zero-dependency |
| Executor model | Claude Sonnet 4.6 | Claude Opus 4.6 | Opus is 1.67x more expensive per token with no benefit for implementation tasks; use Opus only for planning |
| Display mode | In-process (default) | tmux split panes | In-process works universally; tmux has known WSL2 limitations that matter for this project's target environment |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `isolation: "worktree"` in teammate definitions | Silently ignored — issue #33045 open March 2026 | Manual `git worktree add` + path-in-message pattern |
| Multiple API keys in one Anthropic org | Keys share the same org-level rate limit pool | Separate orgs per `API-01` requirement |
| Opus 4.6 for all tasks | $5/$25 vs $3/$15 — Sonnet is sufficient for execution and research | Opus for planning/orchestration, Sonnet for execution, Haiku for research |
| Agent Teams without `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | Feature is disabled by default | Set env var or add to settings.json |
| Peer-to-peer agent messaging | Not supported — teammates cannot create teams or spawn subagents (issue #32731) | Hub-and-spoke via lead; all orchestration through team lead |

---

## Stack Patterns by Workload

**If running parallel executors in wave-based mode (`PARA-01` / `PARA-02`):**
- Lead at Opus 4.6 (orchestration and merge logic)
- Executors at Sonnet 4.6 (implementation tasks)
- Lead manages worktrees manually via `git worktree add`
- Each executor gets its own API key from a different org (round-robin per `API-02`)

**If running parallel research phase (`TEAM-01`, 4 researchers + synthesizer):**
- All researchers at Haiku 4.5 (lower cost, writes to separate files — no worktrees needed)
- Synthesizer at Sonnet 4.6 (synthesis requires more reasoning)
- No worktrees needed — separate output files, no merge conflicts

**If running plan-phase pipeline (researcher → planner → checker):**
- Researcher at Haiku 4.5
- Planner at Opus 4.6 (highest-stakes reasoning — plan quality drives all downstream work)
- Checker at Sonnet 4.6
- No worktrees needed — sequential, single-file output

**If rate-limiting is the bottleneck:**
- Enable multi-org round-robin (`API-02`)
- Add prompt caching for static context (1-hour writes for CLAUDE.md + plan files)
- Promote Haiku for research tasks (4x more ITPM headroom vs Sonnet/Opus)

---

## Version Compatibility

| Feature | Minimum Claude Code Version | Notes |
|---------|----------------------------|-------|
| Agent Teams | v2.1.32 | Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| `isolation: "worktree"` (subagents) | v2.1.49 | Works for Task tool; NOT for teammate definitions |
| `claude --worktree` CLI flag | v2.1.49 | Lead session worktree creation |
| `WorktreeCreate` / `WorktreeRemove` hooks | v2.1.50 | Custom VCS hook events |
| `TeammateIdle` / `TaskCompleted` hooks | v2.1.33 | Quality gate hooks |
| `worktree.sparsePaths` | v2.1.76 | Monorepo sparse checkout optimization |
| Stale worktree auto-cleanup | v2.1.76 | After interrupted parallel runs |
| 1M context at standard pricing (Opus 4.6) | v2.1.32+ | GA as of March 13, 2026 |
| `TaskUpdate`, `TaskStop` tools | v2.1.16+ | Task lifecycle management |

---

## Sources

- [Claude Code Agent Teams docs](https://code.claude.com/docs/en/agent-teams) — Official docs, tool inventory, architecture, limitations (HIGH confidence)
- [Claude Code Changelog](https://code.claude.com/docs/en/changelog) — Version history for all worktree and agent features (HIGH confidence)
- [Anthropic API Rate Limits](https://platform.claude.com/docs/en/api/rate-limits) — Tier tables with exact RPM/ITPM/OTPM numbers (HIGH confidence)
- [Anthropic Pricing](https://platform.claude.com/docs/en/about-claude/pricing) — Current model pricing including 1M context GA (HIGH confidence)
- [GitHub issue #33045](https://github.com/anthropics/claude-code/issues/33045) — `isolation: "worktree"` broken for teammates, open as of 2026-03-13 (HIGH confidence — direct issue read)
- [GitHub issue #32731](https://github.com/anthropics/claude-code/issues/32731) — Teammates tool restrictions (Agent, TeamCreate, CronCreate absent), confirmed March 9, 2026 (HIGH confidence)
- [GitHub issue #28175](https://github.com/anthropics/claude-code/issues/28175) — Agent teams worktree bug, closed as duplicate of #23715, February 28, 2026 (HIGH confidence)
- [Boris Cherny Threads post](https://www.threads.com/@boris_cherny/post/DVAAnexgRUj/) — `--worktree` CLI flag announcement (February 20, 2026) (MEDIUM confidence — social post from Anthropic engineer)
- Anthropic pricing blog / 1M context GA announcement (March 13, 2026) — via WebSearch cross-confirmed with official pricing page (HIGH confidence)

---

*Stack research for: GBSD parallel execution, Agent Teams integration, rate limit management*
*Researched: 2026-03-15*
