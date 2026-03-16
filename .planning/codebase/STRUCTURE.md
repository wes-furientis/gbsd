# Codebase Structure

**Analysis Date:** 2026-03-15

## Directory Layout

```
gbsd/
├── bin/                            # NPM entry point (installer)
│   └── install.js                  # Interactive installer for all runtimes
├── gbsd/                           # Core system files (deployed to runtime config dirs)
│   ├── bin/
│   │   ├── gbsd-tools.cjs          # CLI dispatcher for all atomic operations
│   │   └── lib/                    # Node.js modules (11 files)
│   ├── workflows/                  # Orchestrator workflow specs (35+ .md files)
│   ├── agents/                     # Agent role definitions (12 .md files)
│   ├── references/                 # Shared reference files (e.g., ui-brand.md)
│   └── templates/
│       ├── codebase/               # Document templates for codebase analysis
│       └── research-project/       # Templates for project research workflows
├── commands/gbsd/                  # Command metadata files (32 .md files, consumed by installer)
├── hooks/                          # Git hook implementations (src before build)
├── scripts/                        # Build scripts (build-hooks.js, run-tests.cjs)
├── tests/                          # Test suite (17 .cjs files)
├── assets/                         # Static assets (terminal.svg, logo, etc.)
├── docs/                           # User-facing documentation and analysis docs
├── .planning/                      # Project planning (STATE.md, ROADMAP.md created during use)
└── package.json                    # NPM manifest
```

## Directory Purposes

**`bin/`:**
- Purpose: Standalone installer for end users
- Contains: Single JavaScript file `install.js` that runs `npx @wes-furientis/gbsd@latest`
- Key files: `install.js` (89KB) — prompts user for runtime + location, then deploys workflows and agents to user's system
- Deployed: Users do NOT run this directory directly; instead they run `npx @wes-furientis/gbsd` which downloads package and runs `bin/install.js`

**`gbsd/bin/`:**
- Purpose: Production Node.js tools for workflow orchestration
- Contains: CLI dispatcher and 11 library modules
- Key files:
  - `gbsd-tools.cjs` (23KB): Main CLI router, called by all workflows via `node gbsd-tools.cjs <command>`
  - `lib/core.cjs` (492 lines): Shared utilities, model profiles, config loading, git helpers
  - `lib/state.cjs` (721 lines): STATE.md read/write/update operations, progression engine
  - `lib/phase.cjs` (901 lines): Phase CRUD (create, list, find, complete, remove), numbering logic
  - `lib/roadmap.cjs` (305 lines): ROADMAP.md parsing, phase metadata extraction
  - `lib/frontmatter.cjs` (299 lines): YAML frontmatter parser, serializer, validator
  - `lib/commands.cjs` (548 lines): Utilities (slug generation, timestamps, git commits, todo mgmt, history digest)
  - `lib/verify.cjs` (820 lines): Consistency validation, health checks, repair operations
  - `lib/template.cjs` (222 lines): PLAN/SUMMARY/VERIFICATION scaffold generation
  - `lib/config.cjs` (183 lines): config.json loading, defaults, field access
  - `lib/init.cjs` (710 lines): Compound init commands that load full context for workflows
  - `lib/milestone.cjs` (241 lines): Milestone archival, version management

**`gbsd/workflows/`:**
- Purpose: Orchestrator workflow specifications that run in Claude Code and other runtimes
- Contains: 35+ markdown files defining all GBSD commands
- Key workflows:
  - `new-project.md`: Initial project setup (research, roadmap creation, first phase planning)
  - `plan-phase.md`: Create PLAN.md files for a phase (research → planning → verification)
  - `execute-phase.md`: Run all plans in a phase (parallelized with wave dependencies)
  - `research-phase.md`: Targeted research for a specific phase context
  - `quick.md`: Express path for rapid iteration (minimal planning, direct execution)
  - `verify-work.md`: Validate phase execution and mark complete
  - `add-phase.md`, `insert-phase.md`, `remove-phase.md`: Phase management
  - `complete-milestone.md`: Archive phases and create MILESTONES.md
  - `pause-work.md`, `resume-project.md`: Session management
  - Others: health checks, todo management, progress tracking, debugging

**`gbsd/agents/`:**
- Purpose: Agent role specifications consumed by orchestrator to spawn subagents
- Contains: 12 markdown files defining specialized AI agents
- Key agents:
  - `gbsd-planner.md` (43KB): Creates detailed PLAN.md with task breakdown and wave dependencies
  - `gbsd-executor.md` (18KB): Implements tasks from PLAN.md, produces SUMMARY.md
  - `gbsd-verifier.md` (19KB): Validates SUMMARY against PLAN acceptance criteria
  - `gbsd-phase-researcher.md` (18KB): Gathers technical context for phase planning
  - `gbsd-project-researcher.md` (16KB): Discovers project scope, existing code, tech stack
  - `gbsd-codebase-mapper.md` (17KB): Analyzes codebases, writes ARCHITECTURE/STRUCTURE/CONVENTIONS/TESTING/STACK/INTEGRATIONS/CONCERNS.md
  - `gbsd-roadmapper.md` (17KB): Generates phase breakdown from research findings
  - `gbsd-debugger.md` (38KB): Diagnoses execution failures, proposes fixes
  - `gbsd-plan-checker.md` (23KB): Validates PLAN.md via Nyquist validation (dependencies, wave grouping)
  - `gbsd-integration-checker.md` (13KB): Validates external API integrations
  - `gbsd-research-synthesizer.md` (7KB): Aggregates research findings into CONTEXT.md
  - `gbsd-nyquist-auditor.md` (5KB): Audits wave grouping respects dependencies

**`gbsd/templates/`:**
- Purpose: Markdown document templates for guided document creation
- Contains: Subdirectories with template files
- `codebase/`: 7 templates for codebase analysis (architecture.md, structure.md, conventions.md, testing.md, stack.md, integrations.md, concerns.md)
- `research-project/`: Templates for research workflow outputs

**`commands/gbsd/`:**
- Purpose: Command metadata files exposed as slash commands in Claude Code UI
- Contains: 32 markdown files matching workflows (one per command)
- Pattern: `add-phase.md`, `complete-milestone.md`, `debug.md`, `discuss-phase.md`, `execute-phase.md`, etc.
- Mechanism: Installer copies these files to `~/.claude/commands/gbsd/` so Claude Code displays `/gbsd:add-phase`, `/gbsd:execute-phase`, etc.

**`hooks/`:**
- Purpose: Git hook implementations (built to `dist/` before deployment)
- Contains: Source hook scripts
- Built: `npm run build:hooks` generates `hooks/dist/` with bundled hooks

**`scripts/`:**
- Purpose: Build and test automation
- Contains:
  - `build-hooks.js`: Bundles git hooks via esbuild
  - `run-tests.cjs`: Test runner that orchestrates all `.test.cjs` files in `tests/`

**`tests/`:**
- Purpose: Test suite for gbsd-tools and installation logic
- Contains: 17 CommonJS test files
- Key tests:
  - `core.test.cjs`: Core utilities and helpers
  - `state.test.cjs`: STATE.md operations and mutations
  - `phase.test.cjs`: Phase CRUD and lifecycle
  - `roadmap.test.cjs`: ROADMAP.md parsing and extraction
  - `frontmatter.test.cjs`: YAML parsing and serialization
  - `commands.test.cjs`: Standalone command utilities
  - `verify.test.cjs`: Validation and health checks
  - `init.test.cjs`: Initialization context loading
  - `milestone.test.cjs`: Milestone operations
  - `config.test.cjs`: Configuration loading and defaults
  - `codex-config.test.cjs`, `gemini-config.test.cjs`: Runtime-specific tests
- Test helpers: `helpers.cjs` provides `runGbsdTools()`, `createTempProject()`, `cleanup()`

**`assets/`:**
- Purpose: Static assets (logos, terminal screenshots, etc.)
- Contains: SVG files, images, visual assets for README and docs

**`docs/`:**
- Purpose: User documentation and analysis reports
- Contains: Various markdown analysis files and user guides

**.`planning/`:**
- Purpose: Project planning directory (created during first use)
- Contains: STATE.md (project state), ROADMAP.md (phase breakdown), config.json (user settings)
- Subdirectories created by users:
  - `phases/`: Phase directories (1, 1.1, 2, etc.) containing PLAN.md, SUMMARY.md, CONTEXT.md, RESEARCH.md, VERIFICATION.md, UAT.md
  - `codebase/`: Codebase analysis documents (ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, STACK.md, INTEGRATIONS.md, CONCERNS.md)
  - `todos/`: Task tracking (pending/, completed/)
  - `milestones/`: Archived phases from completed milestones

## Key File Locations

**Entry Points:**
- `bin/install.js`: NPM installer script (interactive, runs `npx @wes-furientis/gbsd@latest`)
- `gbsd/bin/gbsd-tools.cjs`: CLI dispatcher (called by all workflows via bash)
- `gbsd/workflows/*.md`: Orchestrator workflows (exposed as slash commands after installation)

**Configuration:**
- `package.json`: NPM package metadata, version, dependencies
- `.planning/config.json`: User settings (model_profile, commit_docs, research, parallelization, etc.) — created during first use
- `gbsd/bin/lib/core.cjs`: MODEL_PROFILES table mapping agents to models by profile (quality/balanced/budget)

**Core Logic:**
- `gbsd/bin/lib/state.cjs`: STATE.md mutations and progression
- `gbsd/bin/lib/phase.cjs`: Phase operations (create, find, complete)
- `gbsd/bin/lib/roadmap.cjs`: ROADMAP.md parsing
- `gbsd/bin/lib/frontmatter.cjs`: YAML frontmatter parsing
- `gbsd/bin/lib/verify.cjs`: Validation and repair logic
- `gbsd/bin/lib/init.cjs`: Context loading for workflows

**Testing:**
- `tests/`: All test files (*.test.cjs)
- `tests/helpers.cjs`: Test utilities (runGbsdTools, createTempProject, etc.)
- `scripts/run-tests.cjs`: Test runner (used by `npm test`)

## Naming Conventions

**Files:**
- Workflows: kebab-case (plan-phase.md, execute-phase.md, new-project.md)
- Agents: kebab-case with `gbsd-` prefix (gbsd-planner.md, gbsd-executor.md)
- Commands: kebab-case in `commands/gbsd/` directory (add-phase.md, remove-phase.md)
- Documents in .planning/: UPPERCASE (STATE.md, ROADMAP.md, REQUIREMENTS.md, PROJECT.md)
- Phase documents: UPPERCASE (PLAN.md, SUMMARY.md, CONTEXT.md, RESEARCH.md, VERIFICATION.md, UAT.md)
- Test files: camelCase with .test.cjs suffix (state.test.cjs, phase.test.cjs)
- Library modules: camelCase with .cjs suffix (state.cjs, phase.cjs, core.cjs)

**Directories:**
- Workflow output: Numeric phase names (1, 1.1, 2, 2.1) with optional slug suffix (1-project-setup, 2.1-auth-system)
- Config directories: Dot-prefixed (.planning, .claude, .opencode, .gemini, .codex)
- Library directories: lowercase (bin, lib, workflows, agents, templates, commands, tests)
- Project-internal: `gbsd/` for core system, root level for user-facing files (README, package.json, bin/)

## Where to Add New Code

**New Workflow:**
- Primary code: Create `gbsd/workflows/my-workflow.md`
- Command metadata: Create `commands/gbsd/my-workflow.md` (used by installer)
- Pattern: Follow existing workflow structure (purpose, process steps, bash orchestration)
- Reference: Study `gbsd/workflows/plan-phase.md` or `gbsd/workflows/execute-phase.md` for conventions
- Update: Add agent invocations if needed; create agents in `gbsd/agents/` if specialized role required

**New Agent:**
- Primary code: Create `gbsd/agents/gbsd-my-agent.md`
- Pattern: Include role definition, tool access, instructions, error handling, output format
- Reference: Study `gbsd-planner.md` or `gbsd-executor.md` for comprehensive examples
- Usage: Orchestrator spawns via context prompt in workflow

**New Tool Command:**
- Primary code: Add to appropriate module in `gbsd/bin/lib/` (state.cjs for state ops, phase.cjs for phase ops, etc.)
- Function naming: `cmd<CamelCase>` (e.g., `cmdStateUpdate`, `cmdPhaseFind`)
- Dispatcher: Register in `gbsd/bin/gbsd-tools.cjs` main() switch statement
- Output: Call `output(result, raw)` for JSON or `output(result, raw, rawValue)` for raw text
- Error: Call `error('message')` which exits code 1

**New Test:**
- Location: `tests/my-feature.test.cjs`
- Helpers: Use `runGbsdTools(args, cwd)`, `createTempProject()`, `cleanup(tmpDir)` from `tests/helpers.cjs`
- Pattern: Each test uses a temporary project directory via `createTempProject()` and cleans up after
- Reference: Study `tests/state.test.cjs` or `tests/phase.test.cjs` for patterns

**Utilities:**
- Shared helpers: `gbsd/bin/lib/core.cjs` (utility functions)
- Path normalization: `toPosixPath()` in core.cjs (cross-platform path handling)
- Config access: `loadConfig(cwd)` in core.cjs (load with defaults)
- Git operations: `execGit(cwd, args)` in core.cjs (run git commands safely)

## Special Directories

**`.planning/` (Project Planning):**
- Purpose: Holds all project state, roadmap, and execution logs
- Generated: Created by `/gbsd:new-project` workflow
- Committed: YES, tracked in git (contains project specs, decisions, phase logs)
- Structure:
  - `STATE.md`: Current state (phase, decisions, blockers, session continuity)
  - `ROADMAP.md`: All phases with requirements, goals, success metrics
  - `REQUIREMENTS.md`: Cross-phase requirement tracking
  - `PROJECT.md`: Project overview (generated once, rarely updated)
  - `config.json`: User settings for this project
  - `phases/`: Phase directories (1/, 1.1/, 2/, etc.)
  - `codebase/`: Codebase analysis documents (ARCHITECTURE.md, STRUCTURE.md, etc.)
  - `todos/`: Task tracking
  - `milestones/`: Archived phases from completed milestones

**`gbsd/` (Core System):**
- Purpose: System files deployed to user's runtime config dir (~/.claude/gbsd/, ~/.config/opencode/gbsd/, etc.)
- Generated: NO (shipped with package)
- Committed: YES (source of truth in repo)

**`tests/` (Test Suite):**
- Purpose: Automated tests for all gbsd-tools commands and installation logic
- Generated: NO (source code)
- Committed: YES
- Run: `npm test` or `npm run test:coverage`
- Coverage: 70+ line coverage enforced on core lib modules

**`hooks/dist/` (Compiled Git Hooks):**
- Purpose: Pre-commit, post-commit hooks bundled by esbuild
- Generated: YES (`npm run build:hooks` creates from `hooks/src/`)
- Committed: YES (dist/ committed, src/ not committed post-build)

---

*Structure analysis: 2026-03-15*
