<purpose>
Display the complete GBSD command reference. Output ONLY the reference content. Do NOT add project-specific analysis, git status, next-step suggestions, or any commentary beyond the reference.
</purpose>

<reference>
# GBSD Command Reference

**GBSD** (Go Build Something Dangerous) creates hierarchical project plans optimized for solo agentic development with Claude Code.

## Quick Start

1. `/gbsd:new-project` - Initialize project (includes research, requirements, roadmap)
2. `/gbsd:plan-phase 1` - Create detailed plan for first phase
3. `/gbsd:execute-phase 1` - Execute the phase

## Staying Updated

GBSD evolves fast. Update periodically:

```bash
npx @wes-furientis/gbsd@latest
```

## Core Workflow

```
/gbsd:new-project → /gbsd:plan-phase → /gbsd:execute-phase → repeat
```

### Project Initialization

**`/gbsd:new-project`**
Initialize new project through unified flow.

One command takes you from idea to ready-for-planning:
- Deep questioning to understand what you're building
- Optional domain research (spawns 4 parallel researcher agents)
- Requirements definition with v1/v2/out-of-scope scoping
- Roadmap creation with phase breakdown and success criteria

Creates all `.planning/` artifacts:
- `PROJECT.md` — vision and requirements
- `config.json` — workflow mode (interactive/yolo)
- `research/` — domain research (if selected)
- `REQUIREMENTS.md` — scoped requirements with REQ-IDs
- `ROADMAP.md` — phases mapped to requirements
- `STATE.md` — project memory

Usage: `/gbsd:new-project`

**`/gbsd:map-codebase`**
Map an existing codebase for brownfield projects.

- Analyzes codebase with parallel Explore agents
- Creates `.planning/codebase/` with 7 focused documents
- Covers stack, architecture, structure, conventions, testing, integrations, concerns
- Use before `/gbsd:new-project` on existing codebases

Usage: `/gbsd:map-codebase`

### Phase Planning

**`/gbsd:discuss-phase <number>`**
Help articulate your vision for a phase before planning.

- Captures how you imagine this phase working
- Creates CONTEXT.md with your vision, essentials, and boundaries
- Use when you have ideas about how something should look/feel
- Optional `--batch` asks 2-5 related questions at a time instead of one-by-one

Usage: `/gbsd:discuss-phase 2`
Usage: `/gbsd:discuss-phase 2 --batch`
Usage: `/gbsd:discuss-phase 2 --batch=3`

**`/gbsd:research-phase <number>`**
Comprehensive ecosystem research for niche/complex domains.

- Discovers standard stack, architecture patterns, pitfalls
- Creates RESEARCH.md with "how experts build this" knowledge
- Use for 3D, games, audio, shaders, ML, and other specialized domains
- Goes beyond "which library" to ecosystem knowledge

Usage: `/gbsd:research-phase 3`

**`/gbsd:list-phase-assumptions <number>`**
See what Claude is planning to do before it starts.

- Shows Claude's intended approach for a phase
- Lets you course-correct if Claude misunderstood your vision
- No files created - conversational output only

Usage: `/gbsd:list-phase-assumptions 3`

**`/gbsd:plan-phase <number>`**
Create detailed execution plan for a specific phase.

- Generates `.planning/phases/XX-phase-name/XX-YY-PLAN.md`
- Breaks phase into concrete, actionable tasks
- Includes verification criteria and success measures
- Multiple plans per phase supported (XX-01, XX-02, etc.)

Usage: `/gbsd:plan-phase 1`
Result: Creates `.planning/phases/01-foundation/01-01-PLAN.md`

**PRD Express Path:** Pass `--prd path/to/requirements.md` to skip discuss-phase entirely. Your PRD becomes locked decisions in CONTEXT.md. Useful when you already have clear acceptance criteria.

### Execution

**`/gbsd:execute-phase <phase-number>`**
Execute all plans in a phase.

- Groups plans by wave (from frontmatter), executes waves sequentially
- Plans within each wave run in parallel via Task tool
- Verifies phase goal after all plans complete
- Updates REQUIREMENTS.md, ROADMAP.md, STATE.md

Usage: `/gbsd:execute-phase 5`

### Quick Mode

**`/gbsd:quick`**
Execute small, ad-hoc tasks with GBSD guarantees but skip optional agents.

Quick mode uses the same system with a shorter path:
- Spawns planner + executor (skips researcher, checker, verifier)
- Quick tasks live in `.planning/quick/` separate from planned phases
- Updates STATE.md tracking (not ROADMAP.md)

Use when you know exactly what to do and the task is small enough to not need research or verification.

Usage: `/gbsd:quick`
Result: Creates `.planning/quick/NNN-slug/PLAN.md`, `.planning/quick/NNN-slug/SUMMARY.md`

### Roadmap Management

**`/gbsd:add-phase <description>`**
Add new phase to end of current milestone.

- Appends to ROADMAP.md
- Uses next sequential number
- Updates phase directory structure

Usage: `/gbsd:add-phase "Add admin dashboard"`

**`/gbsd:insert-phase <after> <description>`**
Insert urgent work as decimal phase between existing phases.

- Creates intermediate phase (e.g., 7.1 between 7 and 8)
- Useful for discovered work that must happen mid-milestone
- Maintains phase ordering

Usage: `/gbsd:insert-phase 7 "Fix critical auth bug"`
Result: Creates Phase 7.1

**`/gbsd:remove-phase <number>`**
Remove a future phase and renumber subsequent phases.

- Deletes phase directory and all references
- Renumbers all subsequent phases to close the gap
- Only works on future (unstarted) phases
- Git commit preserves historical record

Usage: `/gbsd:remove-phase 17`
Result: Phase 17 deleted, phases 18-20 become 17-19

### Milestone Management

**`/gbsd:new-milestone <name>`**
Start a new milestone through unified flow.

- Deep questioning to understand what you're building next
- Optional domain research (spawns 4 parallel researcher agents)
- Requirements definition with scoping
- Roadmap creation with phase breakdown

Mirrors `/gbsd:new-project` flow for brownfield projects (existing PROJECT.md).

Usage: `/gbsd:new-milestone "v2.0 Features"`

**`/gbsd:complete-milestone <version>`**
Archive completed milestone and prepare for next version.

- Creates MILESTONES.md entry with stats
- Archives full details to milestones/ directory
- Creates git tag for the release
- Prepares workspace for next version

Usage: `/gbsd:complete-milestone 1.0.0`

### Progress Tracking

**`/gbsd:progress`**
Check project status and intelligently route to next action.

- Shows visual progress bar and completion percentage
- Summarizes recent work from SUMMARY files
- Displays current position and what's next
- Lists key decisions and open issues
- Offers to execute next plan or create it if missing
- Detects 100% milestone completion

Usage: `/gbsd:progress`

### Session Management

**`/gbsd:resume-work`**
Resume work from previous session with full context restoration.

- Reads STATE.md for project context
- Shows current position and recent progress
- Offers next actions based on project state

Usage: `/gbsd:resume-work`

**`/gbsd:pause-work`**
Create context handoff when pausing work mid-phase.

- Creates .continue-here file with current state
- Updates STATE.md session continuity section
- Captures in-progress work context

Usage: `/gbsd:pause-work`

### Debugging

**`/gbsd:debug [issue description]`**
Systematic debugging with persistent state across context resets.

- Gathers symptoms through adaptive questioning
- Creates `.planning/debug/[slug].md` to track investigation
- Investigates using scientific method (evidence → hypothesis → test)
- Survives `/clear` — run `/gbsd:debug` with no args to resume
- Archives resolved issues to `.planning/debug/resolved/`

Usage: `/gbsd:debug "login button doesn't work"`
Usage: `/gbsd:debug` (resume active session)

### Todo Management

**`/gbsd:add-todo [description]`**
Capture idea or task as todo from current conversation.

- Extracts context from conversation (or uses provided description)
- Creates structured todo file in `.planning/todos/pending/`
- Infers area from file paths for grouping
- Checks for duplicates before creating
- Updates STATE.md todo count

Usage: `/gbsd:add-todo` (infers from conversation)
Usage: `/gbsd:add-todo Add auth token refresh`

**`/gbsd:check-todos [area]`**
List pending todos and select one to work on.

- Lists all pending todos with title, area, age
- Optional area filter (e.g., `/gbsd:check-todos api`)
- Loads full context for selected todo
- Routes to appropriate action (work now, add to phase, brainstorm)
- Moves todo to done/ when work begins

Usage: `/gbsd:check-todos`
Usage: `/gbsd:check-todos api`

### User Acceptance Testing

**`/gbsd:verify-work [phase]`**
Validate built features through conversational UAT.

- Extracts testable deliverables from SUMMARY.md files
- Presents tests one at a time (yes/no responses)
- Automatically diagnoses failures and creates fix plans
- Ready for re-execution if issues found

Usage: `/gbsd:verify-work 3`

### Milestone Auditing

**`/gbsd:audit-milestone [version]`**
Audit milestone completion against original intent.

- Reads all phase VERIFICATION.md files
- Checks requirements coverage
- Spawns integration checker for cross-phase wiring
- Creates MILESTONE-AUDIT.md with gaps and tech debt

Usage: `/gbsd:audit-milestone`

**`/gbsd:plan-milestone-gaps`**
Create phases to close gaps identified by audit.

- Reads MILESTONE-AUDIT.md and groups gaps into phases
- Prioritizes by requirement priority (must/should/nice)
- Adds gap closure phases to ROADMAP.md
- Ready for `/gbsd:plan-phase` on new phases

Usage: `/gbsd:plan-milestone-gaps`

### Configuration

**`/gbsd:settings`**
Configure workflow toggles and model profile interactively.

- Toggle researcher, plan checker, verifier agents
- Select model profile (quality/balanced/budget)
- Updates `.planning/config.json`

Usage: `/gbsd:settings`

**`/gbsd:set-profile <profile>`**
Quick switch model profile for GBSD agents.

- `quality` — Opus everywhere except verification
- `balanced` — Opus for planning, Sonnet for execution (default)
- `budget` — Sonnet for writing, Haiku for research/verification

Usage: `/gbsd:set-profile budget`

### Utility Commands

**`/gbsd:cleanup`**
Archive accumulated phase directories from completed milestones.

- Identifies phases from completed milestones still in `.planning/phases/`
- Shows dry-run summary before moving anything
- Moves phase dirs to `.planning/milestones/v{X.Y}-phases/`
- Use after multiple milestones to reduce `.planning/phases/` clutter

Usage: `/gbsd:cleanup`

**`/gbsd:help`**
Show this command reference.

**`/gbsd:update`**
Update GBSD to latest version with changelog preview.

- Shows installed vs latest version comparison
- Displays changelog entries for versions you've missed
- Highlights breaking changes
- Confirms before running install
- Better than raw `npx @wes-furientis/gbsd`

Usage: `/gbsd:update`

**`/gbsd:join-discord`**
Join the GBSD Discord community.

- Get help, share what you're building, stay updated
- Connect with other GBSD users

Usage: `/gbsd:join-discord`

## Files & Structure

```
.planning/
├── PROJECT.md            # Project vision
├── ROADMAP.md            # Current phase breakdown
├── STATE.md              # Project memory & context
├── RETROSPECTIVE.md      # Living retrospective (updated per milestone)
├── config.json           # Workflow mode & gates
├── todos/                # Captured ideas and tasks
│   ├── pending/          # Todos waiting to be worked on
│   └── done/             # Completed todos
├── debug/                # Active debug sessions
│   └── resolved/         # Archived resolved issues
├── milestones/
│   ├── v1.0-ROADMAP.md       # Archived roadmap snapshot
│   ├── v1.0-REQUIREMENTS.md  # Archived requirements
│   └── v1.0-phases/          # Archived phase dirs (via /gbsd:cleanup or --archive-phases)
│       ├── 01-foundation/
│       └── 02-core-features/
├── codebase/             # Codebase map (brownfield projects)
│   ├── STACK.md          # Languages, frameworks, dependencies
│   ├── ARCHITECTURE.md   # Patterns, layers, data flow
│   ├── STRUCTURE.md      # Directory layout, key files
│   ├── CONVENTIONS.md    # Coding standards, naming
│   ├── TESTING.md        # Test setup, patterns
│   ├── INTEGRATIONS.md   # External services, APIs
│   └── CONCERNS.md       # Tech debt, known issues
└── phases/
    ├── 01-foundation/
    │   ├── 01-01-PLAN.md
    │   └── 01-01-SUMMARY.md
    └── 02-core-features/
        ├── 02-01-PLAN.md
        └── 02-01-SUMMARY.md
```

## Workflow Modes

Set during `/gbsd:new-project`:

**Interactive Mode**

- Confirms each major decision
- Pauses at checkpoints for approval
- More guidance throughout

**YOLO Mode**

- Auto-approves most decisions
- Executes plans without confirmation
- Only stops for critical checkpoints

Change anytime by editing `.planning/config.json`

## Planning Configuration

Configure how planning artifacts are managed in `.planning/config.json`:

**`planning.commit_docs`** (default: `true`)
- `true`: Planning artifacts committed to git (standard workflow)
- `false`: Planning artifacts kept local-only, not committed

When `commit_docs: false`:
- Add `.planning/` to your `.gitignore`
- Useful for OSS contributions, client projects, or keeping planning private
- All planning files still work normally, just not tracked in git

**`planning.search_gitignored`** (default: `false`)
- `true`: Add `--no-ignore` to broad ripgrep searches
- Only needed when `.planning/` is gitignored and you want project-wide searches to include it

Example config:
```json
{
  "planning": {
    "commit_docs": false,
    "search_gitignored": true
  }
}
```

## Common Workflows

**Starting a new project:**

```
/gbsd:new-project        # Unified flow: questioning → research → requirements → roadmap
/clear
/gbsd:plan-phase 1       # Create plans for first phase
/clear
/gbsd:execute-phase 1    # Execute all plans in phase
```

**Resuming work after a break:**

```
/gbsd:progress  # See where you left off and continue
```

**Adding urgent mid-milestone work:**

```
/gbsd:insert-phase 5 "Critical security fix"
/gbsd:plan-phase 5.1
/gbsd:execute-phase 5.1
```

**Completing a milestone:**

```
/gbsd:complete-milestone 1.0.0
/clear
/gbsd:new-milestone  # Start next milestone (questioning → research → requirements → roadmap)
```

**Capturing ideas during work:**

```
/gbsd:add-todo                    # Capture from conversation context
/gbsd:add-todo Fix modal z-index  # Capture with explicit description
/gbsd:check-todos                 # Review and work on todos
/gbsd:check-todos api             # Filter by area
```

**Debugging an issue:**

```
/gbsd:debug "form submission fails silently"  # Start debug session
# ... investigation happens, context fills up ...
/clear
/gbsd:debug                                    # Resume from where you left off
```

## Getting Help

- Read `.planning/PROJECT.md` for project vision
- Read `.planning/STATE.md` for current context
- Check `.planning/ROADMAP.md` for phase status
- Run `/gbsd:progress` to check where you're up to
</reference>
