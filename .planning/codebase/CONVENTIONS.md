# Coding Conventions

**Analysis Date:** 2026-03-15

## Naming Patterns

**Files:**
- CommonJS files: `.cjs` extension (e.g., `core.cjs`, `phase.cjs`, `frontmatter.cjs`)
- Test files: `[name].test.cjs` pattern (e.g., `core.test.cjs`, `frontmatter.test.cjs`)
- Scripts: `.js` extension for Node executable scripts (e.g., `build-hooks.js`, `run-tests.cjs`)
- Directories: lowercase with hyphens (e.g., `gbsd`, `bin`, `lib`, `hooks`, `tests`, `commands`)

**Functions:**
- Use `camelCase` for all function names: `extractFrontmatter`, `cmdPhasesList`, `safeReadFile`
- Prefix internal/helper functions with underscore or use `Internal` suffix: `findPhaseInternal`, `getRoadmapPhaseInternal`, `resolveModelInternal`
- Prefix command implementations with `cmd`: `cmdConfigSet`, `cmdPhaseAdd`, `cmdHistoryDigest`, `cmdCommit`
- Use descriptive names reflecting both action and scope: `comparePhaseNum`, `normalizePhaseName`, `escapeRegex`

**Variables:**
- Use `camelCase` for all variables: `tmpDir`, `phasesDir`, `configPath`
- Use CONSTANT_CASE for module-level constants: `MODEL_PROFILES`, `VALID_CONFIG_KEYS`, `TOOLS_PATH`
- Boolean variables clearly indicate state: `hasResearch`, `hasBraveSearch`, `found`, `success`
- Iterators use convention names: `i`, `j` for numeric loops; descriptive names for collection iteration: `phase`, `dir`, `file`, `entry`

**Types:**
- Objects returned from functions use snake_case keys: `current_phase`, `phase_number`, `total_phases`, `last_activity`
- Frontmatter keys in YAML/JSON use snake_case: `must_haves`, `tech-stack`, `key-decisions`
- Arrays have plural names: `phases`, `decisions`, `patterns`, `files`, `directories`
- Booleans in return objects clearly indicate state: `found`, `success`, `archived`, `is_complete`

## Code Style

**Formatting:**
- No external formatter (eslint/prettier) in use — style follows Node.js conventions
- Indentation: 2 spaces (consistent across `.cjs` and `.js` files)
- Line length: Practical limit ~100 characters; longer lines acceptable for readability
- Semicolons: Omitted at end of statements (matches Node.js module patterns)
- Curly brace style: Opening brace on same line (function declarations, if/else)

**Spacing:**
- One blank line between function definitions
- Section dividers using comment blocks: `// ─── Section Name ─────────────────────────────────────────────────────────`
- No blank lines between related variable declarations

**Example:**
```javascript
const fs = require('fs');
const path = require('path');
const { output, error, toPosixPath } = require('./core.cjs');

// ─── Parsing engine ───────────────────────────────────────────────────────────

function extractFrontmatter(content) {
  const frontmatter = {};
  const match = content.match(/^---\n([\s\S]+?)\n---/);
  if (!match) return frontmatter;

  // Parse logic here
  return frontmatter;
}

function reconstructFrontmatter(obj) {
  const lines = [];
  // Reconstruct logic
  return lines.join('\n');
}
```

## Import Organization

**Order:**
1. Node.js built-in modules (`fs`, `path`, `child_process`)
2. Local module imports (relative paths with `./` or `../`)
3. Destructured imports grouped logically by source

**Path Aliases:**
- No aliases in use — imports use relative paths or require() statements
- All imports are direct: `require('./core.cjs')`, `require('fs')`

**Module Pattern:**
- CommonJS `require()` for imports
- `module.exports` object with named exports (e.g., `{ output, error, safeReadFile }`)
- No barrel files (index.js) for re-exporting

**Example:**
```javascript
const fs = require('fs');
const path = require('path');
const { output, error } = require('./core.cjs');
const { extractFrontmatter } = require('./frontmatter.cjs');
```

## Error Handling

**Patterns:**
- **Throw early for validation:** Check input validity first, call `error(message)` to exit immediately with stderr + exit code 1
  ```javascript
  if (!keyPath) {
    error('Usage: config-set <key.path> <value>');
  }
  ```

- **Try-catch for file operations:** Wrap filesystem and git commands; catch silently or return safe default
  ```javascript
  try {
    const content = fs.readFileSync(filePath, 'utf-8');
    return content;
  } catch {
    return null;  // safeReadFile pattern
  }
  ```

- **Never throw from command functions:** Use `error()` helper to halt execution with user-friendly message
  ```javascript
  function cmdConfigSet(cwd, keyPath, value, raw) {
    if (!keyPath) {
      error('Usage: config-set <key.path> <value>');  // Exits process
    }
    // Continue if validation passes
  }
  ```

- **Silent catch for optional operations:** Non-critical file writes may fail silently
  ```javascript
  try { fs.writeFileSync(globalDefaultsPath, JSON.stringify(userDefaults, null, 2), 'utf-8'); } catch {}
  ```

- **Return error objects in JSON responses:** Command functions return structured responses with error field when appropriate
  ```javascript
  output({ error: 'STATE.md not found' }, raw, '');
  ```

## Logging

**Framework:** No logging library used — uses direct `console` methods where needed

**Patterns:**
- Errors written to `stderr` via `process.stderr.write()` in error handler
- Status/output sent to `stdout` via `process.stdout.write()` in output handler
- No INFO/DEBUG logging in production command paths; comments document intent instead

**Example:**
```javascript
function error(message) {
  process.stderr.write('Error: ' + message + '\n');
  process.exit(1);
}

function output(result, raw, rawValue) {
  if (raw && rawValue !== undefined) {
    process.stdout.write(String(rawValue));
  } else {
    process.stdout.write(JSON.stringify(result, null, 2));
  }
  process.exit(0);
}
```

## Comments

**When to Comment:**
- **Section dividers:** Use visual separators for major logical blocks
  ```javascript
  // ─── File & Config utilities ──────────────────────────────────────────────────
  ```

- **Algorithm explanation:** Complex regex patterns, sorting logic, or non-obvious operations
  ```javascript
  // Segment-by-segment decimal comparison: 12A < 12A.1 < 12A.1.2 < 12A.2
  ```

- **Known issues/workarounds:** Explain why a shortcut was taken (especially in tests with REG- prefixes)
  ```javascript
  // --no-index checks .gitignore rules regardless of whether the file is tracked.
  // Without it, git check-ignore returns "not ignored" for tracked files...
  ```

- **Avoid:** Self-evident comments that repeat code intent
  ```javascript
  // DON'T: let tmpDir = fs.mkdtempSync(...); // Create temp directory
  // DO: Direct code is clear; only comment WHY, not WHAT
  ```

**JSDoc/TSDoc:**
- Not used in this codebase — functions are documented via comments above definitions
- Inline comments explain complex logic
- Return value and parameter intent documented in function header comments

**Example:**
```javascript
/**
 * Core — Shared utilities, constants, and internal helpers
 */

/**
 * Normalize a relative path to always use forward slashes (cross-platform).
 */
function toPosixPath(p) {
  return p.split(path.sep).join('/');
}
```

## Function Design

**Size:**
- Aim for 20-50 lines per function for command handlers
- Utility functions often 5-15 lines
- No strict limit; readability and single responsibility principle guide length

**Parameters:**
- Most command functions follow pattern: `cmd[Name](cwd, [args...], raw)`
- `cwd`: Always first parameter (working directory context)
- `raw`: Always last parameter (flag for raw output format)
- Options passed as objects: `{ type, phase, includeArchived }`

**Return Values:**
- Command functions return via `output()` helper, never return directly
- Utility functions return structured objects: `{ found: true, directory: '...', error: null }`
- Failures return structured error in same object: `{ found: false, error: 'Phase not found' }`

**Example:**
```javascript
function cmdPhasesList(cwd, options, raw) {
  const phasesDir = path.join(cwd, '.planning', 'phases');

  if (!fs.existsSync(phasesDir)) {
    output({ directories: [], count: 0 }, raw, '');
    return;
  }

  try {
    const entries = fs.readdirSync(phasesDir, { withFileTypes: true });
    const dirs = entries.filter(e => e.isDirectory()).map(e => e.name);
    output({ directories: dirs, count: dirs.length }, raw, '');
  } catch (err) {
    error('Failed to list phases: ' + err.message);
  }
}
```

## Module Design

**Exports:**
- Single `module.exports` object containing all public functions
- No default exports; all exports are named

**Organization:**
- Group related functions at end of file with comment separator
- Export only functions intended for use outside module; helpers remain private

**Example:**
```javascript
module.exports = {
  output,
  error,
  safeReadFile,
  loadConfig,
  isGitIgnored,
  execGit,
  escapeRegex,
  normalizePhaseName,
  comparePhaseNum,
  // ... more exports
};
```

**File Organization:**
- Header comment describing module purpose (e.g., "Config — Planning config CRUD operations")
- Import section (builtin, then local)
- Logical function groupings with section dividers
- Single export block at end

---

*Convention analysis: 2026-03-15*
