# Model Profiles

Model profiles control which Claude model each GBSD agent uses. This allows balancing quality vs token spend.

## Profile Definitions

| Agent | `quality` | `balanced` | `budget` |
|-------|-----------|------------|----------|
| gbsd-planner | opus | opus | sonnet |
| gbsd-roadmapper | opus | sonnet | sonnet |
| gbsd-executor | opus | sonnet | sonnet |
| gbsd-phase-researcher | opus | sonnet | haiku |
| gbsd-project-researcher | opus | sonnet | haiku |
| gbsd-research-synthesizer | sonnet | sonnet | haiku |
| gbsd-debugger | opus | sonnet | sonnet |
| gbsd-codebase-mapper | sonnet | haiku | haiku |
| gbsd-verifier | sonnet | sonnet | haiku |
| gbsd-plan-checker | sonnet | sonnet | haiku |
| gbsd-integration-checker | sonnet | sonnet | haiku |
| gbsd-nyquist-auditor | sonnet | sonnet | haiku |

## Profile Philosophy

**quality** - Maximum reasoning power
- Opus for all decision-making agents
- Sonnet for read-only verification
- Use when: quota available, critical architecture work

**balanced** (default) - Smart allocation
- Opus only for planning (where architecture decisions happen)
- Sonnet for execution and research (follows explicit instructions)
- Sonnet for verification (needs reasoning, not just pattern matching)
- Use when: normal development, good balance of quality and cost

**budget** - Minimal Opus usage
- Sonnet for anything that writes code
- Haiku for research and verification
- Use when: conserving quota, high-volume work, less critical phases

## Resolution Logic

Orchestrators resolve model before spawning:

```
1. Read .planning/config.json
2. Check model_overrides for agent-specific override
3. If no override, look up agent in profile table
4. Pass model parameter to Task call
```

## Per-Agent Overrides

Override specific agents without changing the entire profile:

```json
{
  "model_profile": "balanced",
  "model_overrides": {
    "gbsd-executor": "opus",
    "gbsd-planner": "haiku"
  }
}
```

Overrides take precedence over the profile. Valid values: `opus`, `sonnet`, `haiku`.

## Switching Profiles

Runtime: `/gbsd:set-profile <profile>`

Per-project default: Set in `.planning/config.json`:
```json
{
  "model_profile": "balanced"
}
```

## Design Rationale

**Why Opus for gbsd-planner?**
Planning involves architecture decisions, goal decomposition, and task design. This is where model quality has the highest impact.

**Why Sonnet for gbsd-executor?**
Executors follow explicit PLAN.md instructions. The plan already contains the reasoning; execution is implementation.

**Why Sonnet (not Haiku) for verifiers in balanced?**
Verification requires goal-backward reasoning - checking if code *delivers* what the phase promised, not just pattern matching. Sonnet handles this well; Haiku may miss subtle gaps.

**Why Haiku for gbsd-codebase-mapper?**
Read-only exploration and pattern extraction. No reasoning required, just structured output from file contents.

**Why `inherit` instead of passing `opus` directly?**
Claude Code's `"opus"` alias maps to a specific model version. Organizations may block older opus versions while allowing newer ones. GBSD returns `"inherit"` for opus-tier agents, causing them to use whatever opus version the user has configured in their session. This avoids version conflicts and silent fallbacks to Sonnet.
