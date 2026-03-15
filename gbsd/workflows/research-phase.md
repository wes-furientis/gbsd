<purpose>
Research how to implement a phase. Spawns gbsd-phase-researcher with phase context.

Standalone research command. For most workflows, use `/gbsd:plan-phase` which integrates research automatically.
</purpose>

<pathfinder_navigation>
## Pathfinder Navigation (when .code-intel/ exists)

Check if Pathfinder index is available:
```bash
ls .code-intel/INDEX.md 2>/dev/null && echo "PATHFINDER_AVAILABLE=true" || echo "PATHFINDER_AVAILABLE=false"
```

**If PATHFINDER_AVAILABLE=false:** Skip this section entirely. Use standard glob/grep exploration.

**If PATHFINDER_AVAILABLE=true:**
- If any pathfinder command shows a stale index warning, run `pathfinder generate .` first
- Read `.code-intel/INDEX.md` for codebase context
- Use `pathfinder deps <entity>` to understand dependencies of modules relevant to the phase
- Use `pathfinder query "<question>"` for semantic questions about the codebase
- Read `interfaces/<module>.yaml` files for API surfaces of relevant modules
- Use `pathfinder path <from> <to>` to understand how components connect
- This replaces manual grep/glob exploration for understanding codebase structure
</pathfinder_navigation>

<process>

## Step 0: Resolve Model Profile

@~/.claude/gbsd/references/model-profile-resolution.md

Resolve model for:
- `gbsd-phase-researcher`

## Step 1: Normalize and Validate Phase

@~/.claude/gbsd/references/phase-argument-parsing.md

```bash
PHASE_INFO=$(node "$HOME/.claude/gbsd/bin/gbsd-tools.cjs" roadmap get-phase "${PHASE}")
```

If `found` is false: Error and exit.

## Step 2: Check Existing Research

```bash
ls .planning/phases/${PHASE}-*/RESEARCH.md 2>/dev/null
```

If exists: Offer update/view/skip options.

## Step 3: Gather Phase Context

```bash
INIT=$(node "$HOME/.claude/gbsd/bin/gbsd-tools.cjs" init phase-op "${PHASE}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
# Extract: phase_dir, padded_phase, phase_number, state_path, requirements_path, context_path
```

## Step 4: Spawn Researcher

```
Task(
  prompt="<objective>
Research implementation approach for Phase {phase}: {name}
</objective>

<files_to_read>
- {context_path} (USER DECISIONS from /gbsd:discuss-phase)
- {requirements_path} (Project requirements)
- {state_path} (Project decisions and history)
</files_to_read>

<additional_context>
Phase description: {description}
</additional_context>

<output>
Write to: .planning/phases/${PHASE}-{slug}/${PHASE}-RESEARCH.md
</output>",
  subagent_type="gbsd-phase-researcher",
  model="{researcher_model}"
)
```

## Step 5: Handle Return

- `## RESEARCH COMPLETE` — Display summary, offer: Plan/Dig deeper/Review/Done
- `## CHECKPOINT REACHED` — Present to user, spawn continuation
- `## RESEARCH INCONCLUSIVE` — Show attempts, offer: Add context/Try different mode/Manual

</process>
