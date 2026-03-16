---
name: Agent Teams Worktree Limitation
description: isolation worktree parameter is silently ignored for Agent Teams teammates — known bug #33045. Use independent claude CLI processes instead.
type: reference
---

Agent Teams teammates do NOT support `isolation: "worktree"` — the parameter is silently accepted but has no effect. Confirmed via spike test (2026-03-15) and tracked as GitHub issue #33045.

**Workaround:** Spawn independent Claude Code CLI processes, each in a manually-created git worktree:
```bash
git worktree add .claude/worktrees/exec-01 -b gsd/exec/03-01-w1
ANTHROPIC_API_KEY=sk-ant-... claude -p "Execute plan..." --cwd .claude/worktrees/exec-01 &
```

**Additional teammate restrictions (issue #32731):**
- Teammates lack the Agent tool entirely (can't spawn subagents)
- Teammates lack TeamCreate, TeamDelete, Cron tools
- Architecture is strictly hub-and-spoke

**Official docs:** https://code.claude.com/docs/en/agent-teams
**Bug tracking:** https://github.com/anthropics/claude-code/issues/33045
**Tool restrictions:** https://github.com/anthropics/claude-code/issues/32731
