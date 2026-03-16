# Technology Stack

**Analysis Date:** 2026-03-15

## Languages

**Primary:**
- JavaScript (Node.js) - Core CLI application, agent orchestration, GBSD tools
- CommonJS (.cjs) - Main executable binaries (`gbsd-tools.cjs`, `bin/lib/*.cjs`)

**Secondary:**
- Markdown - Configuration templates, documentation, agent prompts

## Runtime

**Environment:**
- Node.js 16.7.0+ (minimum)
- Tested on: Node.js 18, 20, 22 (via CI matrix)

**Package Manager:**
- npm (Node Package Manager)
- Lockfile: `package-lock.json` present

## Frameworks

**Core:**
- CLI Framework: Custom built with `readline` module for interactive prompts
- No third-party CLI framework (argparse/yargs)

**Testing:**
- Node.js built-in `--test` flag (native test runner, Node 18+)
- c8 v11.0.0 - Code coverage analysis

**Build/Dev:**
- esbuild v0.24.0 - JavaScript bundling and minification
- npm scripts - Custom build scripts in `scripts/` directory

## Key Dependencies

**Critical:**
- None explicitly required at runtime - codebase uses only Node.js built-in modules
- Development only: c8, esbuild

**Infrastructure:**
- Git - Version control integration via `child_process.execSync()`
- fetch (Node.js built-in) - HTTP requests to external APIs
- fs, path, os, crypto, readline - All Node.js built-in standard library modules

## Configuration

**Environment:**
- Config file: `.planning/config.json` (project-level)
- Environment variables: `BRAVE_API_KEY` (optional Brave Search API key)
- Installation: Interactive prompt or non-interactive flags (`--claude`, `--gemini`, `--opencode`, `--codex`)

**Build:**
- Build config: `scripts/build-hooks.js` - Copies hook files to `hooks/dist/`
- Publish hook: `prepublishOnly` runs `build:hooks` before npm publish

## Platform Requirements

**Development:**
- Node.js 16.7.0 or later
- npm for package management
- Cross-platform: Windows, macOS, Linux (tested via GitHub Actions matrix)
- Git repository (for version control integration)

**Production:**
- Published on npm as `@wes-furientis/gbsd`
- Installation method: `npx @wes-furientis/gbsd@latest` or global npm install
- Deployment targets: Claude Code, OpenCode, Gemini CLI, Codex
- Installed to: `~/.claude/`, `~/.opencode/`, `~/.gemini/`, `~/.codex/` depending on runtime

---

*Stack analysis: 2026-03-15*
