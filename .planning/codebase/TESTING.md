# Testing Patterns

**Analysis Date:** 2026-03-15

## Test Framework

**Runner:**
- Node.js built-in `node:test` module (no external test runner)
- Version: Node >=16.7.0 (based on `package.json` engines requirement)
- Config: None (uses Node defaults)

**Assertion Library:**
- Node.js built-in `node:assert` module (strict mode)
- `assert.strictEqual()` for equality
- `assert.deepStrictEqual()` for object/array comparison
- `assert.ok()` for boolean assertions

**Run Commands:**
```bash
npm test                    # Run all tests
npm run test:coverage       # Run tests with coverage check (70% threshold)
node scripts/run-tests.cjs  # Direct test runner (used by npm test)
```

**Coverage Tool:**
- `c8` v11.0.0 — coverage reporter
- Threshold: 70% line coverage
- Includes: `gbsd/bin/lib/*.cjs`
- Excludes: `tests/**`

## Test File Organization

**Location:**
- All test files co-located in `/home/wes/furientis/dev/tools/gbsd/tests/` directory
- Tests are separate from source code (not co-located)

**Naming:**
- Pattern: `[module-name].test.cjs`
- Examples: `core.test.cjs`, `frontmatter.test.cjs`, `phase.test.cjs`, `state.test.cjs`
- One test file per module in `gbsd/bin/lib/`

**Structure:**
```
tests/
├── helpers.cjs                          # Shared test utilities
├── agent-frontmatter.test.cjs
├── codex-config.test.cjs
├── commands.test.cjs
├── config.test.cjs
├── core.test.cjs
├── dispatcher.test.cjs
├── frontmatter-cli.test.cjs
├── frontmatter.test.cjs
├── gemini-config.test.cjs
├── init.test.cjs
├── milestone.test.cjs
├── phase.test.cjs
├── roadmap.test.cjs
├── state.test.cjs
├── verify-health.test.cjs
└── verify.test.cjs
```

## Test Structure

**Suite Organization:**
```javascript
const { test, describe, beforeEach, afterEach } = require('node:test');
const assert = require('node:assert');

describe('loadConfig', () => {
  let tmpDir;
  let originalCwd;

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'gbsd-core-test-'));
    fs.mkdirSync(path.join(tmpDir, '.planning'), { recursive: true });
    originalCwd = process.cwd();
  });

  afterEach(() => {
    process.chdir(originalCwd);
    fs.rmSync(tmpDir, { recursive: true, force: true });
  });

  test('returns defaults when config.json is missing', () => {
    const config = loadConfig(tmpDir);
    assert.strictEqual(config.model_profile, 'balanced');
  });
});
```

**Patterns:**
- **Setup/Teardown:** `beforeEach()` creates temp directories; `afterEach()` cleans up
- **Isolation:** Each test runs in isolated temporary directory
- **Naming:** Test names describe behavior, not implementation (e.g., "returns defaults when config.json is missing")
- **Assertions:** Each test typically 2-5 assertions focused on single behavior

## Mocking

**Framework:** No external mocking library (e.g., sinon, jest) — uses native Node.js file/exec APIs

**Patterns:**

**Filesystem mocking:**
- Create temporary directories with `fs.mkdtempSync()` for isolated test environments
- Write test data files directly: `fs.writeFileSync(path.join(tmpDir, '.planning', 'config.json'), JSON.stringify(...))`
- Clean up with `fs.rmSync(tmpDir, { recursive: true, force: true })` in afterEach

```javascript
function createTempProject() {
  const tmpDir = fs.mkdtempSync(path.join(require('os').tmpdir(), 'gbsd-test-'));
  fs.mkdirSync(path.join(tmpDir, '.planning', 'phases'), { recursive: true });
  return tmpDir;
}
```

**Command execution mocking:**
- Use `runGbsdTools()` helper to execute CLI commands in isolation
- Accepts array of arguments (safer, shell-bypassed) or shell string

```javascript
function runGbsdTools(args, cwd = process.cwd()) {
  try {
    let result;
    if (Array.isArray(args)) {
      result = execFileSync(process.execPath, [TOOLS_PATH, ...args], {
        cwd,
        encoding: 'utf-8',
        stdio: ['pipe', 'pipe', 'pipe'],
      });
    } else {
      result = execSync(`node "${TOOLS_PATH}" ${args}`, {
        cwd,
        encoding: 'utf-8',
        stdio: ['pipe', 'pipe', 'pipe'],
      });
    }
    return { success: true, output: result.trim() };
  } catch (err) {
    return {
      success: false,
      output: err.stdout?.toString().trim() || '',
      error: err.stderr?.toString().trim() || err.message,
    };
  }
}
```

**Git repository mocking:**
- Initialize temporary git repos with `git init` + `git config`
- Make at least one commit before running state-dependent tests

```javascript
function createTempGitProject() {
  const tmpDir = fs.mkdtempSync(path.join(require('os').tmpdir(), 'gbsd-test-'));
  fs.mkdirSync(path.join(tmpDir, '.planning', 'phases'), { recursive: true });

  execSync('git init', { cwd: tmpDir, stdio: 'pipe' });
  execSync('git config user.email "test@test.com"', { cwd: tmpDir, stdio: 'pipe' });
  execSync('git config user.name "Test"', { cwd: tmpDir, stdio: 'pipe' });

  fs.writeFileSync(path.join(tmpDir, '.planning', 'PROJECT.md'), '# Project\n\nTest project.\n');
  execSync('git add -A', { cwd: tmpDir, stdio: 'pipe' });
  execSync('git commit -m "initial commit"', { cwd: tmpDir, stdio: 'pipe' });

  return tmpDir;
}
```

**What to Mock:**
- File system state (config.json, SUMMARY.md, phase directories)
- Git state (initialized repo, commits)
- Temporary directories for isolation

**What NOT to Mock:**
- File system operations themselves (use real tmpdir)
- JSON parsing/serialization (test actual behavior)
- Path resolution logic (test actual behavior)

## Fixtures and Factories

**Test Data:**
Generated inline using helper functions that write to test file system:

```javascript
function writeConfig(obj) {
  fs.writeFileSync(
    path.join(tmpDir, '.planning', 'config.json'),
    JSON.stringify(obj, null, 2)
  );
}

// Usage in test:
test('reads model_profile from config.json', () => {
  writeConfig({ model_profile: 'quality' });
  const config = loadConfig(tmpDir);
  assert.strictEqual(config.model_profile, 'quality');
});
```

**YAML/Frontmatter fixtures:**
```javascript
const summaryContent = `---
phase: "01"
name: "Foundation Setup"
dependency-graph:
  provides:
    - "Database schema"
    - "Auth system"
tech-stack:
  added:
    - "prisma"
    - "jose"
---

# Summary content here
`;
fs.writeFileSync(path.join(phaseDir, '01-01-SUMMARY.md'), summaryContent);
```

**Location:**
- Test data created inline with `fs.writeFileSync()` in each test
- No separate fixtures directory
- Data is temporary (cleaned up after each test)

## Coverage

**Requirements:** 70% line coverage enforced on `gbsd/bin/lib/*.cjs`

**View Coverage:**
```bash
npm run test:coverage
```

**Coverage output includes:**
- Line coverage percentage
- Uncovered statement locations
- Pass/fail against 70% threshold

## Test Types

**Unit Tests:**
- **Scope:** Individual functions and modules
- **Approach:** Test pure functions with various inputs, testing logic paths
- **Examples:**
  - `core.test.cjs` tests `loadConfig`, `resolveModelInternal`, `comparePhaseNum`, etc.
  - `frontmatter.test.cjs` tests YAML parsing with simple to complex structures
  - `config.test.cjs` tests config CRUD operations
- **Isolation:** Each unit test creates its own tmpdir environment

**Integration Tests:**
- **Scope:** Multiple modules working together (e.g., config + phase + commands)
- **Approach:** Test end-to-end command execution via `runGbsdTools()`
- **Examples:**
  - `commands.test.cjs` tests `history-digest` command with full phase structure
  - `state.test.cjs` tests `state-snapshot` command reading/parsing STATE.md
  - `phase.test.cjs` tests phase lifecycle (add, complete, remove)
- **Isolation:** Use `createTempProject()` or `createTempGitProject()` to set up realistic environments

**E2E Tests:**
- Not explicitly present; integration tests via CLI commands serve this purpose
- `runGbsdTools()` invokes actual CLI entry point (`gbsd-tools.cjs`), testing real workflows

## Common Patterns

**Async Testing:**
No async/await patterns in use — all tests are synchronous. Operations use blocking APIs:
- `execSync()` for git commands
- `fs.readFileSync()` / `fs.writeFileSync()` for file operations
- `execFileSync()` for process execution

**Error Testing:**
```javascript
test('missing STATE.md returns error', () => {
  const result = runGbsdTools('state-snapshot', tmpDir);
  assert.ok(result.success, `Command should succeed: ${result.error}`);

  const output = JSON.parse(result.output);
  assert.strictEqual(output.error, 'STATE.md not found', 'should report missing file');
});
```

**Test Regression Documentation:**
Tests document known bugs with `REG-##` naming convention. Appears in both test comments and file headers:

From `tests/frontmatter.test.cjs`:
```javascript
/**
 * GBSD Tools Tests - frontmatter.cjs
 *
 * Tests for the hand-rolled YAML parser's pure function exports...
 * Includes REG-04 regression: quoted comma inline array edge case.
 */

describe('extractFrontmatter', () => {
  test('handles quoted commas in inline arrays — REG-04 known limitation', () => {
    // REG-04: The split(',') on line 53 does NOT respect quotes.
    // The parser WILL split on commas inside quotes, producing wrong results.
    // This test documents the CURRENT (buggy) behavior.
    const content = '---\nkey: ["a, b", c]\n---\n';
    const result = extractFrontmatter(content);
    // The bug produces ["a", "b", "c"] instead of ["a, b", "c"]
    assert.ok(result.key.length > 2, 'REG-04: split produces more items than intended due to quoted comma bug');
  });
});
```

From `tests/core.test.cjs`:
```javascript
// Bug: loadConfig previously omitted model_overrides from return value
test('returns model_overrides when present (REG-01)', () => {
  writeConfig({ model_overrides: { 'gbsd-executor': 'opus' } });
  const config = loadConfig(tmpDir);
  assert.deepStrictEqual(config.model_overrides, { 'gbsd-executor': 'opus' });
});
```

**Setup/Cleanup patterns:**
```javascript
describe('module functionality', () => {
  let tmpDir;

  beforeEach(() => {
    tmpDir = createTempProject();
  });

  afterEach(() => {
    cleanup(tmpDir);
  });

  test('does something', () => {
    // Test code using tmpDir
  });
});
```

## Test Helpers

**`helpers.cjs`** provides reusable test utilities:

```javascript
// Run gbsd-tools command
function runGbsdTools(args, cwd = process.cwd()) { ... }

// Create temp project structure
function createTempProject() { ... }

// Create temp git repo
function createTempGitProject() { ... }

// Cleanup temp directory
function cleanup(tmpDir) {
  fs.rmSync(tmpDir, { recursive: true, force: true });
}

module.exports = { runGbsdTools, createTempProject, createTempGitProject, cleanup, TOOLS_PATH };
```

Located at: `tests/helpers.cjs`

---

*Testing analysis: 2026-03-15*
