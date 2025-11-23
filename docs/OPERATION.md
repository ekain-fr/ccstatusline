# Operating a Forked ccstatusline

This guide explains how to install, build, and use your forked version of ccstatusline instead of the upstream npm package.

## Table of Contents

- [Understanding `bun` vs `bunx`](#understanding-bun-vs-bunx)
- [Initial Setup](#initial-setup)
- [Development Operations](#development-operations)
- [Installation Options](#installation-options)
- [Testing Your Changes](#testing-your-changes)
- [Troubleshooting](#troubleshooting)

---

## Understanding `bun` vs `bunx`

**Important:** The upstream documentation uses `bunx ccstatusline@latest`, but for your forked version, you'll use `bun` instead. Here's why:

### `bunx` - For Published Packages

```bash
bunx ccstatusline@latest
```

**What it does:**
- Fetches the package from the **npm registry**
- Downloads and caches it temporarily (like `npx` for npm)
- Runs the package's entry point
- Used for published packages

**When to use:**
- Installing the upstream version from npm
- Installing your published scoped package: `bunx @yourusername/ccstatusline@latest`
- Running any package from npm without installing it

### `bun` - For Local Files

```bash
bun /path/to/ccstatusline/src/ccstatusline.ts
```

**What it does:**
- Directly executes a **local file** on your filesystem
- Runs TypeScript or JavaScript files directly
- No package resolution or downloading
- Used for local development

**When to use:**
- Running your forked/local version during development
- Executing scripts in your project
- Testing changes before publishing

### Why This Guide Uses `bun`

Since you're working with a **local fork**, you're not fetching from npm - you're running files directly from your filesystem. That's why all the examples in this guide use:

```bash
bun /Users/eugene/VS_Code/GitHub_Repos/_claude-code/ccstatusline/src/ccstatusline.ts
```

Instead of:

```bash
bunx ccstatusline@latest  # ❌ This fetches from npm, not your fork
```

### When Would You Use `bunx` with Your Fork?

Only after publishing your fork to npm:

```bash
# 1. Publish your fork with a scoped name
npm publish --access public

# 2. Then you can use bunx
bunx @yourusername/ccstatusline@latest
```

---

## Initial Setup

### 1. Clone Your Fork

```bash
# Clone your forked repository
git clone https://github.com/yourusername/ccstatusline.git
cd ccstatusline

# Or if you already have it locally
cd /path/to/your/fork/ccstatusline
```

### 2. Install Dependencies

```bash
# Install all dependencies
bun install
```

**What this does:**
- Installs all packages from `package.json`
- Applies Bun's patchedDependencies (fixes ink@6.2.0 compatibility)
- Sets up development environment

**Expected output:**
```
bun install v1.2.22
+ @eslint/js@9.33.0
+ @stylistic/eslint-plugin@5.2.3
...
386 packages installed [16.32s]
```

---

## Development Operations

### Build for Distribution

```bash
# Build the production JavaScript bundle
bun run build
```

**What this does:**
- Removes old build artifacts (`rm -rf dist/*`)
- Compiles TypeScript to JavaScript
- Bundles all dependencies
- Targets Node.js 14+ for compatibility
- Replaces version placeholder with package.json version
- Creates `dist/ccstatusline.js` (~2.5 MB)

**Expected output:**
```
Bundled 690 modules in 45ms
  ccstatusline.js  2.50 MB  (entry point)
✓ Replaced version placeholder with 2.0.21
```

**When to run:**
- Before testing the built version
- Before publishing to npm
- When using Option B (see below)

### Lint and Type Check

```bash
# Run TypeScript type checking and ESLint
bun run lint
```

**What this does:**
- Runs `bun tsc --noEmit` for type checking
- Runs ESLint with auto-fix enabled
- Enforces code style and import ordering

**When to run:**
- Before committing changes
- After making code changes
- As part of CI/CD pipeline

### Run Tests

```bash
# Run all tests with vitest
bun test
```

**What this does:**
- Executes unit tests in `src/**/__tests__/`
- Reports coverage (if configured)

**When to run:**
- After making changes to utilities or widgets
- Before committing
- Before publishing

### Generate API Documentation

```bash
# Generate TypeDoc documentation
bun run docs

# Remove generated docs
bun run docs:clean
```

**What this does:**
- Creates `api-docs/` directory with HTML documentation
- Documents all types, interfaces, and functions
- Generates browsable API reference

---

## Installation Options

The upstream documentation says to use `bunx ccstatusline@latest`, but for your fork, you have better options.

### Option A: Development Mode (Recommended for Active Development)

**Use the TypeScript source directly - no build step required.**

#### Configuration

Edit `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bun /Users/eugene/VS_Code/GitHub_Repos/_claude-code/ccstatusline/src/ccstatusline.ts",
    "padding": 0
  }
}
```

**Update the path** to match your fork's location.

#### Pros
✅ No build step required - instant feedback
✅ Changes take effect immediately
✅ Fastest development workflow
✅ Full TypeScript error messages
✅ Easy to debug with source maps

#### Cons
❌ Slightly slower startup (~100-200ms)
❌ Requires Bun runtime
❌ Not representative of production bundle

#### When to Use
- Active feature development
- Debugging issues
- Rapid iteration
- Testing small changes

#### Example Workflow

```bash
# 1. Edit a widget
vim src/widgets/ContextPercentage.ts

# 2. Test immediately (no build needed)
echo '{"model":{"display_name":"Test"}}' | bun src/ccstatusline.ts

# 3. Changes are live in Claude Code immediately
# Just run a new command in Claude Code to see the updated status line
```

---

### Option B: Production Mode (Recommended for Testing Built Version)

**Use the built JavaScript bundle - representative of npm package.**

#### Configuration

1. **Build the distribution:**
```bash
bun run build
```

2. **Edit `~/.claude/settings.json`:**
```json
{
  "statusLine": {
    "type": "command",
    "command": "bun /Users/eugene/VS_Code/GitHub_Repos/_claude-code/ccstatusline/dist/ccstatusline.js",
    "padding": 0
  }
}
```

**Update the path** to match your fork's location.

#### Pros
✅ Faster startup (bundled, optimized)
✅ Representative of production
✅ Tests the actual build output
✅ Same as npm package behavior
✅ Works with Node.js (not just Bun)

#### Cons
❌ Must rebuild after every change
❌ Extra build step
❌ Slower iteration cycle
❌ Harder to debug (bundled code)

#### When to Use
- Testing before publishing to npm
- Verifying build output
- Performance testing
- Final validation before release
- When you need Node.js compatibility

#### Example Workflow

```bash
# 1. Edit a widget
vim src/widgets/ContextPercentage.ts

# 2. Rebuild
bun run build

# 3. Test built version
echo '{"model":{"display_name":"Test"}}' | bun dist/ccstatusline.js

# 4. Changes are now live in Claude Code
```

---

## Testing Your Changes

### Quick Test (Command Line)

Test without installing to Claude Code:

```bash
# Test with minimal JSON
echo '{"model":{"display_name":"Claude 3.5 Sonnet"},"transcript_path":"test.jsonl"}' | bun src/ccstatusline.ts

# Expected output (with ANSI colors):
# Model: Claude 3.5 Sonnet | Ctx: 0 | ⎇ main
```

### Use Example Payload

```bash
# Use the provided example payload
bun run example

# This runs:
# cat scripts/payload.example.json | bun start
```

### Interactive TUI Test

```bash
# Launch the configuration TUI
bun run start

# Navigate menus:
# - Arrow keys to move
# - Enter to select
# - Escape to go back
# - Ctrl+C to exit
```

### Test in Claude Code

After updating `~/.claude/settings.json`, run any Claude Code command to see the status line:

```bash
# Start Claude Code
claude

# The status line appears at the bottom with your changes
```

---

## Alternative Installation Methods

### Option C: Use `bun link` (Global Symlink)

Good for treating your fork like an installed package.

```bash
# 1. Build your fork
cd /path/to/your/fork/ccstatusline
bun run build

# 2. Link globally
bun link

# 3. Update Claude Code settings
# ~/.claude/settings.json
{
  "statusLine": {
    "type": "command",
    "command": "ccstatusline",
    "padding": 0
  }
}
```

**Note:** Must rebuild after changes. The `bun link` creates a global symlink to your local package.

---

### Option D: Install from GitHub

Install directly from your GitHub fork (requires pushing changes first).

```bash
# Install from your fork's main branch
bun add github:yourusername/ccstatusline

# Or from a specific branch
bun add github:yourusername/ccstatusline#feature-branch

# Or from a specific commit
bun add github:yourusername/ccstatusline#abc1234
```

Then use in Claude Code:
```json
{
  "statusLine": {
    "type": "command",
    "command": "bunx ccstatusline",
    "padding": 0
  }
}
```

**Pros:** Tests actual package installation
**Cons:** Must push to GitHub first, slower iteration

---

### Option E: Publish to npm (For Sharing)

Publish your fork under a scoped package name.

```bash
# 1. Update package.json
{
  "name": "@yourusername/ccstatusline",
  "version": "2.0.21-fork.1"
}

# 2. Build
bun run build

# 3. Publish to npm
npm publish --access public

# 4. Use your package
# ~/.claude/settings.json
{
  "statusLine": {
    "type": "command",
    "command": "bunx @yourusername/ccstatusline@latest",
    "padding": 0
  }
}
```

**Pros:** Shareable, versioned, works like upstream
**Cons:** Requires npm account, public registry

---

## Troubleshooting

### "Cannot find package 'react'"

**Problem:** Dependencies not installed.

**Solution:**
```bash
bun install
```

### "Permission denied" when running status line

**Problem:** Script not executable or wrong path.

**Solution:**
```bash
# Verify file exists
ls -l /path/to/ccstatusline/src/ccstatusline.ts

# Check Bun is installed
bun --version

# Use absolute paths in settings.json
```

### Status line not updating after changes

**Problem:** Using built version without rebuilding.

**Solution:**
```bash
# If using Option B, rebuild:
bun run build

# Or switch to Option A for instant updates:
# Use src/ccstatusline.ts instead of dist/ccstatusline.js
```

### Git widgets showing "no git"

**Problem:** Not in a git repository or git not in PATH.

**Solution:**
```bash
# Check git is installed
git --version

# Verify current directory is a git repo
git status

# Git widgets show "no git" when not in a repository (expected behavior)
```

### Colors not displaying correctly

**Problem:** Terminal doesn't support 256 colors or truecolor.

**Solution:**
```bash
# Check terminal color support
echo $COLORTERM  # Should show "truecolor" for 24-bit color

# Configure color level in TUI:
bun run start
# Navigate to: Terminal Options → Color Level
```

### Build fails with TypeScript errors

**Problem:** Type errors in code.

**Solution:**
```bash
# Check types first
bun run lint

# Fix errors shown in output
# Then rebuild
bun run build
```

### Status line truncated or wrapping

**Problem:** Terminal width detection or flex mode setting.

**Solution:**
```bash
# Configure in TUI
bun run start
# Navigate to: Terminal Options → Terminal Width Options

# Options:
# - Full width always
# - Full width minus 40 (recommended)
# - Full width until compact
```

---

## File Paths Quick Reference

### Source Files
- Main entry: `src/ccstatusline.ts`
- Widgets: `src/widgets/`
- Utilities: `src/utils/`
- TUI: `src/tui/`
- Types: `src/types/`

### Configuration Files
- ccstatusline config: `~/.config/ccstatusline/settings.json`
- Claude Code config: `~/.claude/settings.json`
- Usage data: `~/.config/ccstatusline/usage-data.json`

### Build Artifacts
- Built bundle: `dist/ccstatusline.js`
- API docs: `api-docs/` (after running `bun run docs`)

### Environment Variables
- `CLAUDE_CONFIG_DIR`: Custom Claude config directory (optional)

---

## Recommended Workflow

For most development work, use **Option A**:

```bash
# 1. Initial setup (once)
git clone https://github.com/yourusername/ccstatusline.git
cd ccstatusline
bun install

# 2. Configure Claude Code (once)
# Edit ~/.claude/settings.json to use:
# "command": "bun /path/to/ccstatusline/src/ccstatusline.ts"

# 3. Develop
vim src/widgets/SomeWidget.ts

# 4. Test immediately
echo '{"model":{"display_name":"Test"}}' | bun src/ccstatusline.ts

# 5. See changes live in Claude Code (no restart needed)

# 6. Before committing
bun run lint
bun test

# 7. Before releasing
bun run build
echo '{"model":{"display_name":"Test"}}' | bun dist/ccstatusline.js
```

For final validation before publishing, switch to **Option B** to test the built bundle.

---

## Summary

| Method | Command | Best For | Rebuild Required |
|--------|---------|----------|------------------|
| **Option A** | `bun src/ccstatusline.ts` | Active development | ❌ No |
| **Option B** | `bun dist/ccstatusline.js` | Pre-release testing | ✅ Yes |
| Option C | `ccstatusline` (via bun link) | Global package testing | ✅ Yes |
| Option D | `bunx ccstatusline` (from GitHub) | CI/testing | ✅ Yes |
| Option E | `bunx @you/ccstatusline` (npm) | Public distribution | ✅ Yes |

**Recommendation:** Use **Option A** for development, **Option B** before publishing.
