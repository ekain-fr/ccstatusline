# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ccstatusline is a customizable status line formatter for Claude Code CLI that displays model info, git branch, token usage, and other metrics. It functions as both:
1. A piped command processor for Claude Code status lines
2. An interactive TUI configuration tool when run without input

## Development Commands

```bash
# Install dependencies
bun install

# Run in TUI mode
bun run start

# Test with piped input
echo '{"model":{"display_name":"Claude 3.5 Sonnet"},"transcript_path":"test.jsonl"}' | bun run src/ccstatusline.ts

# Or use the example payload
bun run example

# Build for npm distribution
bun run build   # Creates dist/ccstatusline.js with Node.js 14+ compatibility

# Lint and type check
bun run lint   # Runs TypeScript type checking and ESLint with auto-fix

# Run tests
bun test   # Uses vitest

# Generate API documentation
bun run docs   # Creates docs/ directory with TypeDoc output
bun run docs:clean   # Remove generated docs
```

## Architecture

The project has dual runtime compatibility - works with both Bun and Node.js:

### Core Structure
- **src/ccstatusline.ts**: Main entry point that detects piped vs interactive mode
  - Piped mode: Parses JSON from stdin and renders formatted status line
  - Interactive mode: Launches React/Ink TUI for configuration

### TUI Components (src/tui/)
- **index.tsx**: Main TUI entry point that handles React/Ink initialization
- **App.tsx**: Root component managing navigation and state
- **components/**: Modular UI components for different configuration screens
  - MainMenu, LineSelector, ItemsEditor, ColorMenu, GlobalOverridesMenu
  - PowerlineSetup, TerminalOptionsMenu, StatusLinePreview

### Utilities (src/utils/)
- **config.ts**: Settings management
  - Loads from `~/.config/ccstatusline/settings.json`
  - Handles migration from old settings format
  - Default configuration if no settings exist
- **renderer.ts**: Core rendering logic for status lines
  - Handles terminal width detection and truncation
  - Applies colors, padding, and separators
  - Manages flex separator expansion
- **powerline.ts**: Powerline font detection and installation
- **claude-settings.ts**: Integration with Claude Code settings.json
  - Respects `CLAUDE_CONFIG_DIR` environment variable with fallback to `~/.claude`
  - Provides installation command constants (NPM, BUNX, self-managed)
  - Detects installation status and manages settings.json updates
  - Validates config directory paths with proper error handling
- **colors.ts**: Color definitions and ANSI code mapping

### Widgets (src/widgets/)
Each widget implements the `Widget` interface defined in src/types/Widget.ts. All widgets must provide:
- `render()`: Generates the widget's display text
- `getDefaultColor()`: Returns the default color for the widget
- `getDescription()`: Returns a description for the TUI
- `getDisplayName()`: Returns the display name for the TUI
- `getEditorDisplay()`: Returns formatted text for the editor view
- `supportsRawValue()`: Whether the widget supports raw value mode (no label)
- `supportsColors()`: Whether the widget supports custom colors
- Optional: `getCustomKeybinds()`, `renderEditor()`, `handleEditorAction()`

Available widgets:
- Model, Version, OutputStyle, SessionCost - Claude Code metadata
- GitBranch, GitChanges, GitWorktree - Git repository status
- TokensInput, TokensOutput, TokensCached, TokensTotal - Token usage metrics
- ContextLength, ContextPercentage, ContextPercentageUsable - Context window metrics
- BlockTimer, SessionClock - Time tracking
- CurrentWorkingDir, TerminalWidth - Environment info
- CustomText, CustomCommand - User-defined content

### Rendering Pipeline (src/utils/renderer.ts)
The rendering system uses a two-pass approach for efficiency:
1. **Pre-rendering** (`preRenderAllWidgets`): Calls each widget's render() once and caches results with plaintext length
2. **Alignment calculation** (`calculateMaxWidthsFromPreRendered`): Determines max widths for auto-alignment in Powerline mode
3. **Final rendering** (`renderStatusLine`): Applies colors, padding, separators, and truncation using cached pre-rendered content

Two rendering modes:
- **Regular mode**: Standard text with separators, supports flex separators for right-alignment
- **Powerline mode** (`renderPowerlineStatusLine`): Arrow separators, caps, themes, and auto-alignment across multiple lines

## Configuration Management

Settings are stored in `~/.config/ccstatusline/settings.json` with automatic versioning and migration:
- **Version tracking**: Settings include a `version` field for migration support (src/types/Settings.ts)
- **Migrations**: Handled by src/utils/migrations.ts - automatically upgrades old config formats
- **Validation**: Uses Zod schemas (SettingsSchema) for type-safe parsing with defaults
- **Backup on error**: Bad settings backed up to settings.bak before reset to defaults
- **Settings structure**:
  - `lines`: Array of widget arrays (supports unlimited status lines)
  - `colorLevel`: Color mode (0=none, 1=basic, 2=256, 3=truecolor)
  - `flexMode`: Terminal width handling mode
  - `powerline`: Powerline configuration (enabled, theme, separators, caps, autoAlign)
  - `defaultPadding`, `defaultSeparator`: Global formatting options
  - `overrideForegroundColor`, `overrideBackgroundColor`: Global color overrides
  - `updatemessage`: Temporary update notifications (auto-decrements)

## Key Implementation Details

- **Cross-platform stdin reading**: Detects Bun vs Node.js environment and uses appropriate stdin API (src/ccstatusline.ts:28-55)
- **Token metrics**: Parses Claude Code transcript files (JSONL format) to calculate token usage (src/utils/jsonl.ts)
- **Git integration**: Uses child_process.execSync to get current branch and changes
- **Terminal width management**: Three flex modes (full, full-minus-40, full-until-compact) with context-aware switching
- **Flex separators**: Special separator type that expands to fill available space, enabling right-aligned content
- **Powerline mode**: Optional Powerline-style rendering with arrow separators, caps, themes, and auto-alignment
- **Custom commands**: Execute shell commands and display output in status line (receives JSON via stdin)
- **Mergeable items**: Items can be merged together with or without padding for seamless visual grouping
- **Pre-rendering optimization**: All widgets render once per update, results cached for multiple uses (alignment, truncation, final output)
- **Non-breaking spaces**: Output uses \u00A0 instead of regular spaces to prevent VSCode terminal trimming

## Creating New Widgets

To add a new widget:
1. Create a new file in src/widgets/ implementing the Widget interface
2. Export the widget from src/widgets/index.ts
3. Register it in src/utils/widgets.ts using `registerWidget(type, implementation)`
4. Add the widget type to the WidgetItemSchema in src/types/Widget.ts if needed
5. Widget must handle null/undefined data gracefully (return null to skip rendering)
6. Use RenderContext to access Claude Code data, token metrics, session info
7. For interactive configuration, implement `getCustomKeybinds()` and `handleEditorAction()`

Example widget structure:
```typescript
export const MyWidget: Widget = {
    getDefaultColor: () => 'cyan',
    getDescription: () => 'Shows something useful',
    getDisplayName: () => 'My Widget',
    getEditorDisplay: (item) => ({ displayText: 'My Widget' }),
    render: (item, context, settings) => {
        if (!context.data) return null;
        return 'Hello World';
    },
    supportsRawValue: () => true,
    supportsColors: () => true
};
```

## Bun Usage Preferences

Default to using Bun instead of Node.js:
- Use `bun <file>` instead of `node <file>` or `ts-node <file>`
- Use `bun install` instead of `npm install`
- Use `bun run <script>` instead of `npm run <script>`
- Use `bun build` with appropriate options for building
- Bun automatically loads .env, so don't use dotenv

## Important Notes

- **Patches**: The project uses Bun's patchedDependencies to fix ink@6.2.0 compatibility (patches/ink@6.2.0.patch). Patches are automatically applied on `bun install`
- **ESLint configuration**: Uses flat config format (eslint.config.js) with TypeScript and React plugins
  - Import ordering enforced: builtin/external → internal → parent → sibling → index
  - Single-item import newlines enforced
  - Strict TypeScript checking with consistent type imports
- **Build target**: When building for distribution, target Node.js 14+ for maximum compatibility
- **Type checking and linting**: Only run via `bun run lint` command, never using `npx eslint`, `eslint`, `tsx`, `bun tsc` or any other variation directly
- **Lint rules**: Never disable a lint rule via a comment, no matter how benign the lint warning or error may seem
- **Testing**: Uses vitest for testing (vitest.config.ts). Run tests with `bun test`. Currently has minimal test coverage with tests in src/utils/__tests__/ and src/widgets/__tests__/
- **Color handling**:
  - Chalk level set globally in ccstatusline.ts and tui/index.tsx based on settings.colorLevel
  - Three color modes: ansi16 (basic), ansi256 (256 colors), truecolor (24-bit)
  - ANSI codes manually constructed in Powerline mode to avoid reset interference
- **ANSI regex**: The pattern `\\x1b\\[[0-9;]*m` is intentionally used throughout to strip/preserve color codes (no-control-regex disabled in ESLint)