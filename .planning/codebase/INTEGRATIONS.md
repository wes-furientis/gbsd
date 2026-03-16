# External Integrations

**Analysis Date:** 2026-03-15

## APIs & External Services

**Search:**
- Brave Search API - Optional web search integration for research agents
  - SDK/Client: Native fetch (Node.js built-in)
  - Auth: API key in `BRAVE_API_KEY` environment variable
  - Endpoint: `https://api.search.brave.com/res/v1/web/search`
  - Usage: Performed by `gbsd-phase-researcher` and `gbsd-project-researcher` agents when configured
  - Implementation: `gbsd/bin/lib/commands.cjs` - `cmdWebsearch()` function
  - Parameters: query, limit (default 10), freshness (day/week/month), country, language
  - Fallback: If `BRAVE_API_KEY` not set, agents use built-in WebSearch capability (silent skip)

**NPM Registry:**
- npm.js Registry - Package distribution and installation
  - Used for: Package installation via `npm install` and `npm ci`
  - Version management: Handled by `package.json` and `package-lock.json`

## Runtime Integrations

**AI Agents & Code Editors:**
- Claude Code - IDE integration with Claude AI model
  - Installation path: `~/.claude/gbsd/`
  - Integration: Custom prompts in `.claude/prompts/gbsd/`
  - Agents deployed: 12 custom agents (planner, executor, researcher, etc.)

- OpenCode - Open-source code editor with AI
  - Installation path: `~/.config/opencode/gbsd/`
  - Integration: Custom prompts

- Gemini CLI - Google Gemini command-line interface
  - Installation path: `~/.gemini/gbsd/`
  - Integration: Custom prompts

- Codex - Code generation platform (skills-first architecture)
  - Installation path: `~/.codex/skills/gbsd-*/`
  - Integration: Skills files (`SKILL.md`) instead of custom prompts
  - Agents deployed as skills with workspace permissions

## Version Control

**Git Integration:**
- Integration method: `child_process.execSync()` for git commands
- Operations: Commit planning documents, manage branches, fetch repository info
- Files involved:
  - `gbsd/bin/lib/core.cjs` - `execGit()` helper
  - `gbsd/bin/lib/commands.cjs` - Commit, branch, and git state operations
- Supports: Branch creation, phase branching, commit history tracking

## Data & State Management

**Local Filesystem:**
- `.planning/` directory structure (project root)
  - `.planning/config.json` - Project configuration and model profiles
  - `.planning/codebase/` - STACK.md, ARCHITECTURE.md, etc. (codebase analysis docs)
  - `.planning/phases/` - Phase directories with PLAN.md, SUMMARY.md files
  - `.planning/todos/` - Pending and completed task files
  - `.planning/references/` - External documentation references

**No External Databases:**
- All state stored locally in `.planning/` directory
- Uses JSON for structured config data
- Uses Markdown frontmatter for document metadata
- Git-tracked for version history

## Configuration & Secrets

**Environment Variables:**
- `BRAVE_API_KEY` - Optional Brave Search API key (enables enhanced web search)
- `CLAUDE_CONFIG_DIR` - Override for custom Claude config directory (supports multi-account setups)
- `NODE_V8_COVERAGE` - Set by test runner for code coverage collection

**Secrets Storage:**
- Brave API key stored in: `~/.gbsd/brave_api_key` (local file, not tracked)
- Never stored in `.env` files (GBSD doesn't use dotenv)
- Environment-based configuration for deployment scenarios

## NPM Module Distribution

**Package Registry:**
- Published to npm as `@wes-furientis/gbsd`
- Entry point: `bin/install.js` - Interactive installation script
- Distribution files:
  - `bin/` - Installation scripts
  - `commands/` - Command integrations (not currently used)
  - `gbsd/` - Core tools and templates
  - `agents/` - Agent prompt definitions
  - `hooks/dist/` - Pre-built hooks for IDE integration
  - `scripts/` - Build and test utilities

## No Third-Party External Services

**Payment:**
- Not integrated

**Email/Messaging:**
- Not integrated

**File Storage:**
- Not integrated (local filesystem only)

**Monitoring/Analytics:**
- Not integrated

**Authentication:**
- Not integrated (GBSD itself doesn't authenticate; it facilitates AI agents with their own auth)

**CDN/Hosting:**
- Not integrated (distributed via npm registry only)

---

*Integration audit: 2026-03-15*
