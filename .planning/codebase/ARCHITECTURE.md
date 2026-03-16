# Architecture

**Analysis Date:** 2026-03-15

## Pattern Overview

**Overall:** Layered orchestration with subagent delegation — GBSD is a meta-prompting framework that coordinates specialized AI agents through a modular, state-driven workflow system.

**Key Characteristics:**
- **Declarative workflow specs:** Each workflow (plan-phase, execute-phase, research-phase) is defined as markdown with embedded bash orchestration, not code
- **Subagent orchestration:** The main orchestrator (Claude Code or other runtime) spawns specialized agents via context prompts (`agents/gbsd-*.md`), delegating specific responsibilities to reduce context bleed
- **State machine progression:** Project state (`STATE.md`), roadmap structure (`ROADMAP.md`), and phase directories drive all progression; tools read this state and advance it atomically
- **Centralized tools API:** A single CLI entrypoint (`gbsd-tools.cjs`) provides atomic operations for all state mutations and queries, called from orchestrator bash scripts
- **Phase-based structuring:** Work is organized into numbered phases (1, 1.1, 2, etc.) with each phase containing PLAN.md files → SUMMARY.md files (pair pattern) and supporting docs (CONTEXT.md, RESEARCH.md, VERIFICATION.md)

## Layers

**Orchestrator Layer:**
- Purpose: Choreograph workflows, make decisions, spawn agents, handle UI/UX
- Location: `gbsd/workflows/*.md` (35+ markdown files with embedded bash)
- Contains: Workflow specifications with process steps, bash orchestration logic, checkpoint handling
- Depends on: `gbsd-tools.cjs`, agent specs, runtime environment (Claude Code, OpenCode, Gemini, Codex)
- Used by: End users via `/gbsd:command` invocations

**Tools Layer (Node.js CLI):**
- Purpose: Provide atomic operations for state queries, mutations, and validations
- Location: `gbsd/bin/gbsd-tools.cjs` (dispatcher) + `gbsd/bin/lib/*.cjs` (11 modules)
- Contains: Command router, state machine operations, file I/O, git operations, validation logic
- Core modules:
  - `core.cjs`: Shared utilities, model profiles, path helpers, config loading
  - `state.cjs`: STATE.md read/write, progression operations
  - `phase.cjs`: Phase CRUD, lifecycle (create, list, find, remove, complete)
  - `roadmap.cjs`: ROADMAP.md parsing and phase metadata extraction
  - `frontmatter.cjs`: YAML frontmatter parsing/serialization for PLAN/SUMMARY frontmatter
  - `commands.cjs`: Standalone utilities (slug generation, timestamp, git commit, todo mgmt)
  - `verify.cjs`: Consistency validation, disk state verification, health checks
  - `template.cjs`: PLAN/SUMMARY/VERIFICATION scaffold generation
  - `config.cjs`: Config.json operations and migrations
  - `milestone.cjs`: Milestone archival and version management
  - `init.cjs`: Context loading for workflow initialization (compound commands)
- Depends on: Node.js fs, path, child_process; git for version control
- Used by: All orchestrator workflows via bash subprocesses

**Agent Layer:**
- Purpose: Specialized AI agents handle specific tasks (research, planning, execution, verification)
- Location: `agents/gbsd-*.md` (12 agent specs)
- Contains: Role definitions, task instructions, tool access permissions, error handling strategies
- Agent types:
  - **gbsd-planner**: Creates executable PLAN.md files with task breakdown
  - **gbsd-executor**: Implements code from PLAN.md, produces SUMMARY.md
  - **gbsd-verifier**: Validates SUMMARY.md against PLAN.md acceptance criteria
  - **gbsd-phase-researcher**: Gathers technical context for phase planning
  - **gbsd-project-researcher**: Initial discovery research for new projects
  - **gbsd-research-synthesizer**: Aggregates research findings into CONTEXT.md
  - **gbsd-debugger**: Diagnoses failures and produces fix strategies
  - **gbsd-codebase-mapper**: Analyzes codebases, writes to `.planning/codebase/`
  - **gbsd-roadmapper**: Generates phase breakdowns and ROADMAP.md
  - **gbsd-plan-checker**: Validates PLAN.md before execution (Nyquist validation)
  - **gbsd-integration-checker**: Verifies external API integrations
  - **gbsd-nyquist-auditor**: Ensures wave grouping respects dependencies
- Depends on: Orchestrator context (workflow state, phase info), git operations
- Used by: Orchestrator workflows via context prompts

**Document Format Layer:**
- Purpose: Standardize state representation and document schemas
- Location: `gbsd/templates/codebase/*.md` (7 template files) + actual `.planning/` structure
- Contains: Markdown document templates with YAML frontmatter schemas
- Key documents:
  - `STATE.md`: Project state, decisions, blockers, session continuity
  - `ROADMAP.md`: Phase breakdown with requirements, goals, success metrics
  - `REQUIREMENTS.md`: Cross-project requirement tracking (REQ-01, REQ-02, etc.)
  - `.planning/codebase/ARCHITECTURE.md`: Target codebase architecture (this document)
  - `.planning/codebase/STRUCTURE.md`: Target codebase file organization
  - `.planning/codebase/CONVENTIONS.md`: Coding patterns and style guidelines
  - `.planning/codebase/TESTING.md`: Testing frameworks and patterns
  - `.planning/codebase/STACK.md`: Technology dependencies
  - `.planning/codebase/INTEGRATIONS.md`: External API integrations
  - `.planning/codebase/CONCERNS.md`: Technical debt and risks
  - Phase docs: CONTEXT.md, RESEARCH.md, PLAN.md, SUMMARY.md, VERIFICATION.md, UAT.md
- Used by: Planner and executor agents to understand project constraints, current state, and target architecture

## Data Flow

**Workflow Lifecycle:**

1. **Initialization** (Orchestrator reads system state)
   - Orchestrator calls `node gbsd-tools.cjs init <workflow-type> <args>`
   - Tool loads `.planning/config.json`, `.planning/STATE.md`, `.planning/ROADMAP.md`
   - Tool returns JSON context (file paths, flags, current phase state)
   - Orchestrator parses JSON and proceeds with workflow

2. **Phase Planning** (plan-phase workflow)
   - Orchestrator spawns **gbsd-phase-researcher** with phase context → produces RESEARCH.md
   - Orchestrator spawns **gbsd-planner** with RESEARCH.md + CONTEXT.md → produces PLAN.md (with frontmatter: tasks, files_modified, wave dependencies)
   - Orchestrator spawns **gbsd-plan-checker** (if enabled) → validates PLAN.md via Nyquist validation
   - Orchestrator revises plan if checker finds issues (max 3 iterations)
   - Orchestrator calls `node gbsd-tools.cjs commit <message> --files PLAN.md` → git adds and commits
   - **Result:** Phase ready for execution

3. **Execution** (execute-phase workflow)
   - Orchestrator loads plan wave structure from PLAN.md frontmatter
   - Orchestrator groups plans into waves (respects `depends_on` in frontmatter)
   - **For each wave (parallel if `parallelization=true`, serial if `false`):**
     - For each plan: Orchestrator spawns **gbsd-executor** with PLAN.md → produces SUMMARY.md
     - Orchestrator spawns **gbsd-verifier** with PLAN.md + SUMMARY.md → validates completion
     - If summary doesn't meet criteria, spawns **gbsd-debugger** → fix strategy
   - Orchestrator updates STATE.md with execution metrics (duration, tasks completed, files modified)
   - Orchestrator calls `node gbsd-tools.cjs state advance-plan` → increments plan counter
   - **Result:** Phase execution logged, progress tracked

4. **Progression** (orchestrator triggers state machine)
   - After all phase plans execute, orchestrator calls `node gbsd-tools.cjs phase complete <phase>`
   - Tool updates ROADMAP.md phase status to "complete"
   - Tool updates STATE.md with phase completion timestamp
   - Orchestrator detects next unplanned phase from ROADMAP.md
   - **Result:** Ready for next phase planning

**State Management:**

- **Single source of truth:** `.planning/STATE.md` holds current project state (current phase, completed phases, decisions, blockers)
- **Atomic mutations:** All state changes go through `gbsd-tools.cjs state update/patch` — no direct file edits from workflows
- **Checkpoint recovery:** STATE.md includes session continuity fields (stopped_at, current_task) so workflows can resume mid-phase
- **Config-driven behavior:** `.planning/config.json` gates features (research, verification, parallelization) and sets model profiles (quality/balanced/budget)

## Key Abstractions

**Phase:**
- Purpose: Represents a unit of work in the roadmap (e.g., "1", "1.1", "2" are valid phase numbers)
- Examples: `.planning/phases/1-project-setup`, `.planning/phases/2-api-core`, `.planning/phases/2.1-auth`
- Pattern: Each phase is a directory containing PLAN.md, SUMMARY.md, CONTEXT.md, RESEARCH.md, VERIFICATION.md, UAT.md
- Operations: Create via phase add, find via phase find, complete via phase complete, remove via phase remove

**Plan & Summary Pair:**
- Purpose: PLAN.md = specification, SUMMARY.md = proof of execution
- PLAN frontmatter: `name`, `description`, `tasks`, `files_modified`, `depends_on`, `wave`, `type` (execute, tdd, research)
- SUMMARY frontmatter: `name`, `status` (completed, failed), `duration_minutes`, `tasks_completed`, `key_decisions`, `artifacts`
- Pattern: Orchestrator validates SUMMARY against PLAN acceptance criteria before marking phase complete
- Location: Both live in phase directory (e.g., `.planning/phases/1-setup/1-PLAN.md` and `1-SUMMARY.md`)

**Roadmap Section:**
- Purpose: Centralized specification of all phases and their requirements
- Structure: ROADMAP.md contains `##` headings for each phase with metadata
- Metadata extracted: phase number, phase name, goal, description, requirements (REQ-01, REQ-02), success metrics
- Pattern: `gbsd-tools.cjs roadmap get-phase <N>` returns phase metadata for use in planning workflows

**Config Profile:**
- Purpose: Determines which Claude model to use for each agent type based on user preference
- Profiles: `quality` (opus), `balanced` (opus/sonnet mix), `budget` (sonnet/haiku mix)
- Model mappings: `gbsd/bin/lib/core.cjs` contains `MODEL_PROFILES` table mapping agent → {quality, balanced, budget}
- Pattern: Orchestrator reads `config.json` model_profile, then calls `gbsd-tools.cjs resolve-model <agent>` to get actual model name
- Example: `resolve-model gbsd-planner` with profile=balanced returns "claude-3-5-sonnet"

**Wave Dependencies:**
- Purpose: Define execution order for plans within a phase
- PLAN frontmatter field: `depends_on: [<plan-name>, ...]`
- Pattern: Orchestrator calls `gbsd-tools.cjs phase-plan-index <phase>` to get plan dependency graph, groups into waves
- Execution: All plans in wave 1 can execute in parallel; wave 2 waits for wave 1 completion
- Usage: Respects database migrations before tests, APIs before consumers, etc.

## Entry Points

**User Entry Points:**
- Location: Commands defined in `.claude/commands/gbsd/*.md` (installed by `bin/install.js`)
- Format: Each command is a markdown file that the Claude Code UI exposes as a slash command
- Example: `/gbsd:new-project`, `/gbsd:plan-phase`, `/gbsd:execute-phase`
- Mechanism: User types `/gbsd:plan-phase 2` → Claude Code loads corresponding workflow markdown → orchestrator bash starts

**NPM Installation:**
- Location: `bin/install.js` (Node.js installer script)
- Purpose: Copies workflows + agents to runtime config directory (~/.claude/, ~/.opencode/, ~/.gemini/, ~/.codex/)
- Process: `npx @wes-furientis/gbsd@latest` prompts user for runtime + location, then deploys files

**Tools Dispatcher:**
- Location: `gbsd/bin/gbsd-tools.cjs`
- Entry: `node gbsd-tools.cjs <command> [args]`
- Routing: Main function parses `argv[2]` to determine which module (state, phase, roadmap, verify, etc.) handles the command
- Output: JSON to stdout, raw values via `--raw` flag, large payloads written to `/tmp/gsd-*.json` and referenced as `@file:/tmp/...`

## Error Handling

**Strategy:** Fail fast with clear messages; tools exit code 1 on error, orchestrator detects and halts workflow.

**Patterns:**

- **Config errors:** If `.planning/config.json` missing, tools return defaults. Orchestrator prompts user to run `/gbsd:new-project`.
- **Phase not found:** `phase find <phase>` returns null, orchestrator suggests valid phases from ROADMAP.md.
- **State corruption:** `verify health --repair` attempts automated fixes (missing STATE.md sections, orphaned phase dirs).
- **Plan validation:** `gbsd-plan-checker` runs Nyquist validation, returns list of violations. Orchestrator prompts `gbsd-planner` to revise.
- **Execution failures:** If executor SUMMARY status is "failed", orchestrator spawns `gbsd-debugger` to analyze. User reviews debug output, fixes code, retries plan.

## Cross-Cutting Concerns

**Logging:**
- Tools use `console.error()` for warnings, `process.stderr.write()` for structured errors
- Workflows write checkpoint logs to STATE.md (stopped_at timestamp, current_task field)
- Agents include execution timings in SUMMARY.md (duration_minutes field)

**Validation:**
- Tools perform schema validation on frontmatter (PLAN.md, SUMMARY.md) via `frontmatter validate`
- Orchestrator calls `verify consistency` to check phase numbering, disk/roadmap sync before major operations
- `verify health` checks for missing required files, orphaned directories, incorrect phase numbering

**Authentication & Secrets:**
- `.env` files not tracked; tools read secrets from environment or `~/.gbsd/brave_api_key` for Brave Search
- Git operations use system git config (user.name, user.email)
- Agents can access secrets through environment variables injected by runtime (Claude Code, OpenCode, etc.)

**Context Engineering:**
- Workflows use `<files_to_read>` blocks in agent prompts to pass required files
- Large context (>50KB JSON) written to `/tmp/gsd-*.json` and referenced as `@file:/path` to avoid buffer overflows
- Agents load only phase-specific context to prevent context rot (CONTEXT.md, RESEARCH.md, PLAN.md, not entire codebase)

---

*Architecture analysis: 2026-03-15*
