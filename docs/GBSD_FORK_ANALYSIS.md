# GBSD Fork Analysis

## Overview

GBSD ("Go Build Something Dangerous") is a **minimally-invasive fork** of upstream [GSD](https://github.com/glittercowboy/get-shit-done) ("Get Shit Done"), focused on integrating [Pathfinder](https://github.com/wes-furientis/pathfinder) code intelligence into the GSD workflow system.

## Fork Point

Forked from `glittercowboy/get-shit-done` at commit `0f38e34` (v1.22.4+). The fork contains exactly **2 additional commits** on top of upstream:

1. **`661dbb2`** — Full rebrand: GSD → GBSD (177 files, 2837 insertions, 2829 deletions)
2. **`c5453bc`** — Pathfinder navigation instructions added to 4 workflow definitions (78 insertions)

## What the Fork Changes

### 1. Rebrand (commit `661dbb2`)

Systematic rename from GSD to GBSD via a 4-pass Python script (longest-patterns-first to avoid partial matches):

| Element | Original (GSD) | Fork (GBSD) |
|---------|----------------|-------------|
| Package name | `get-shit-done-cc` | `@wes-furientis/gbsd` |
| Agent files | `gsd-*.md` | `gbsd-*.md` (all 12) |
| Hook files | `gsd-*.js` | `gbsd-*.js` |
| Main directory | `get-shit-done/` | `gbsd/` |
| Commands directory | `commands/gsd/` | `commands/gbsd/` |
| Slash commands | `/gsd:` | `/gbsd:` |
| UI banners | `GSD >` | `GBSD >` |
| Environment variables | `GET_SHIT_DONE` | `GBSD` |
| Config directory | `~/.claude/get-shit-done/` | `~/.claude/gbsd/` |
| Branding | "Get Shit Done" | "Go Build Something Dangerous" |

177 files modified, 22 directory/file renames.

Migration detection added to `bin/install.js` — detects existing GSD installations and provides guidance.

Post-rebrand audit confirmed zero unintended GSD references remaining, excluding:
- External attribution URLs (discord.gg/gsd, gsd_foundation Twitter)
- Git history comments

### 2. Pathfinder Navigation (commit `c5453bc`)

Four workflow files received Pathfinder-specific navigation instructions:

| Workflow | Pathfinder Usage |
|----------|-----------------|
| `gbsd/workflows/map-codebase.md` | Reads `INDEX.md`, `module_map.yaml`, `document_map.yaml` for orientation |
| `gbsd/workflows/research-phase.md` | Uses `pathfinder deps/query` for codebase understanding |
| `gbsd/workflows/plan-phase.md` | Uses `pathfinder blast-radius` for impact analysis in `files_modified` |
| `gbsd/workflows/execute-plan.md` | Uses `pathfinder deps` for file discovery instead of globbing |

All 4 workflows include a `PATHFINDER_AVAILABLE` check:

```bash
ls .code-intel/INDEX.md 2>/dev/null && echo "PATHFINDER_AVAILABLE=true" || echo "PATHFINDER_AVAILABLE=false"
```

When `PATHFINDER_AVAILABLE=false`, agents skip Pathfinder sections entirely and fall back to standard glob/grep exploration (vanilla GSD behavior).

## What the Fork Does NOT Change

The following features exist in **upstream GSD** and were inherited, not added by the fork:

- **Node Repair Operator** — Autonomous task recovery (retry/decompose/prune/escalate, budget=2). Added upstream in commit `2411f66`.
- **Mandatory `read_first` and `acceptance_criteria`** — Task metadata fields for frontloading context and enabling automated verification. Added upstream in commit `e97851e`.
- **Pre-commit hook handling** — Guidance against `--no-verify` bypass. Part of upstream executor workflow.
- **Workflows** (diagnose-issues, resume-project, transition) — All present in upstream GSD.
- **All 12 agent definitions** — Core behavior unchanged; only renamed from `gsd-*` to `gbsd-*`.

## Upstream Sync Strategy

GBSD tracks upstream via the `upstream` remote:

```
origin    https://github.com/wes-furientis/gbsd.git
upstream  https://github.com/glittercowboy/get-shit-done.git
```

To sync upstream changes: merge `upstream/main`, resolve any rebrand conflicts (gsd→gbsd naming), and verify Pathfinder workflow sections are preserved.

## Status

- **Rebrand:** Complete and verified
- **Pathfinder integration:** Implemented (4 workflows modified with graceful degradation)
- **Upstream divergence:** 2 commits ahead of `glittercowboy/get-shit-done`
