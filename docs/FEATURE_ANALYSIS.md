# Feature Analysis & Implementation Plans

**Date:** 2025-11-07
**Project:** ccstatusline
**Analysis:** Missing Feature Investigation

## Executive Summary

This document analyzes 4 requested features for ccstatusline and their current implementation status. All 4 features are currently **NOT IMPLEMENTED**. This document provides detailed findings and comprehensive implementation plans for each feature.

### Features Analyzed

| # | Feature | Status | Complexity | Priority Recommendation |
|---|---------|--------|------------|------------------------|
| 1 | Threshold-based color/bold formatting | ❌ Not Implemented | High | High - Enhances user awareness |
| 2 | Weekly limit tracking with % and reset | ❌ Not Implemented | Very High | Medium - Requires external data |
| 3 | Dirty file count in git status | ❌ Not Implemented | Low | High - Simple, valuable addition |
| 4 | `--no-optional-locks` flag for git | ❌ Not Implemented | Very Low | High - Prevents lock contention |

---

## Feature 1: Threshold-Based Color/Bold Formatting

### Current Status: NOT IMPLEMENTED

#### Investigation Findings

**What was searched:**
- All widget render methods in `src/widgets/`
- Rendering pipeline in `src/utils/renderer.ts`
- Settings schema in `src/types/Settings.ts`
- Color application logic in `src/utils/colors.ts`

**What was found:**
- Widgets use static default colors via `getDefaultColor()` method
- No conditional color logic based on metric values
- Only threshold found: `compactThreshold` (used for layout switching, not colors)
- No widget metadata for threshold configuration

**Code Evidence:**
```typescript
// Example from ContextPercentage.ts
render(item: WidgetItem, context: RenderContext) {
    const percentage = (contextLength / 200000) * 100;
    // Always returns same formatted string, no color change logic
    return item.rawValue ? `${percentage.toFixed(1)}%` : `Ctx: ${percentage.toFixed(1)}%`;
}
```

### Implementation Plan

#### 1.1 Architecture Design

**New Types** (`src/types/Threshold.ts`):
```typescript
export interface ThresholdConfig {
    enabled: boolean;
    warningThreshold: number;  // e.g., 60 for 60%
    criticalThreshold: number; // e.g., 80 for 80%
    warningColor?: string;     // Optional color override
    criticalColor?: string;    // Optional color override
    warningBold?: boolean;     // Optional bold override
    criticalBold?: boolean;    // Optional bold override
}

export interface WidgetThresholds {
    'context-percentage'?: ThresholdConfig;
    'context-percentage-usable'?: ThresholdConfig;
    'tokens-total'?: ThresholdConfig;
    'session-cost'?: ThresholdConfig;
    // Extensible for other numeric widgets
}
```

**Settings Schema Update** (`src/types/Settings.ts`):
```typescript
export const SettingsSchema = z.object({
    // ... existing fields
    thresholds: z.record(z.string(), z.object({
        enabled: z.boolean().default(false),
        warningThreshold: z.number().default(60),
        criticalThreshold: z.number().default(80),
        warningColor: z.string().optional(),
        criticalColor: z.string().optional(),
        warningBold: z.boolean().optional(),
        criticalBold: z.boolean().optional()
    })).optional().default({})
});
```

#### 1.2 Widget Interface Extension

**Update Widget Interface** (`src/types/Widget.ts`):
```typescript
export interface Widget {
    // ... existing methods

    // New optional methods for threshold support
    supportsThresholds?(): boolean;
    getCurrentValue?(item: WidgetItem, context: RenderContext): number | null;
    getThresholdRange?(): { min: number; max: number };
}
```

#### 1.3 Threshold Evaluation System

**New Utility** (`src/utils/thresholds.ts`):
```typescript
import type { ThresholdConfig, WidgetItem, RenderContext, Settings } from '../types';

export interface ThresholdResult {
    level: 'normal' | 'warning' | 'critical';
    color?: string;
    bold?: boolean;
}

export function evaluateThreshold(
    widgetType: string,
    value: number,
    settings: Settings
): ThresholdResult {
    const config = settings.thresholds?.[widgetType];

    if (!config || !config.enabled) {
        return { level: 'normal' };
    }

    if (value >= config.criticalThreshold) {
        return {
            level: 'critical',
            color: config.criticalColor,
            bold: config.criticalBold
        };
    }

    if (value >= config.warningThreshold) {
        return {
            level: 'warning',
            color: config.warningColor,
            bold: config.warningBold
        };
    }

    return { level: 'normal' };
}
```

#### 1.4 Renderer Integration

**Update Renderer** (`src/utils/renderer.ts`):
```typescript
import { evaluateThreshold } from './thresholds';

// In renderStatusLine function, when applying colors:
const widget = getWidget(item.type);
let finalColor = item.color ?? widget.getDefaultColor();
let finalBold = item.bold;

// Apply threshold-based overrides if widget supports it
if (widget.supportsThresholds?.() && widget.getCurrentValue) {
    const currentValue = widget.getCurrentValue(item, context);
    if (currentValue !== null) {
        const threshold = evaluateThreshold(item.type, currentValue, settings);
        if (threshold.color) finalColor = threshold.color;
        if (threshold.bold !== undefined) finalBold = threshold.bold;
    }
}

// Apply colors with potential threshold overrides
const colored = applyColorsWithOverride(widgetText, finalColor, item.backgroundColor, finalBold);
```

#### 1.5 Widget Implementations

**Update ContextPercentage** (`src/widgets/ContextPercentage.ts`):
```typescript
export const ContextPercentageWidget: Widget = {
    // ... existing methods

    supportsThresholds: () => true,

    getCurrentValue: (item: WidgetItem, context: RenderContext): number | null => {
        if (!context.tokenMetrics) return null;
        const percentage = (context.tokenMetrics.contextLength / 200000) * 100;
        return Math.min(100, percentage);
    },

    getThresholdRange: () => ({ min: 0, max: 100 })
};
```

Similarly update:
- `ContextPercentageUsable.ts`
- `TokensTotal.ts` (threshold on total token count)
- `SessionCost.ts` (threshold on cost amount)

#### 1.6 TUI Configuration Screen

**New Component** (`src/tui/components/ThresholdMenu.tsx`):
```typescript
interface Props {
    settings: Settings;
    onSave: (settings: Settings) => void;
    onBack: () => void;
}

export const ThresholdMenu: React.FC<Props> = ({ settings, onSave, onBack }) => {
    // UI for configuring thresholds per widget
    // - Select widget type
    // - Enable/disable thresholds
    // - Set warning/critical values
    // - Set warning/critical colors (optional)
    // - Set warning/critical bold (optional)
    // - Preview with sample values
};
```

**Update MainMenu** (`src/tui/components/MainMenu.tsx`):
Add "Configure Thresholds" menu option.

#### 1.7 Testing Strategy

**Unit Tests** (`src/utils/__tests__/thresholds.test.ts`):
```typescript
describe('evaluateThreshold', () => {
    it('returns normal when below warning threshold', () => {
        const result = evaluateThreshold('context-percentage', 50, settings);
        expect(result.level).toBe('normal');
    });

    it('returns warning when above warning threshold', () => {
        const result = evaluateThreshold('context-percentage', 70, settings);
        expect(result.level).toBe('warning');
    });

    it('returns critical when above critical threshold', () => {
        const result = evaluateThreshold('context-percentage', 85, settings);
        expect(result.level).toBe('critical');
    });

    it('applies custom colors', () => {
        const result = evaluateThreshold('context-percentage', 85, settingsWithColors);
        expect(result.color).toBe('red');
    });
});
```

**Widget Tests** (`src/widgets/__tests__/ContextPercentage.test.ts`):
```typescript
describe('ContextPercentageWidget.getCurrentValue', () => {
    it('returns percentage value for threshold evaluation', () => {
        const value = ContextPercentageWidget.getCurrentValue!(item, context);
        expect(value).toBe(75.5);
    });
});
```

#### 1.8 Migration Strategy

**Settings Migration** (`src/utils/migrations.ts`):
- No breaking changes - thresholds are opt-in
- Default: `thresholds: {}` (disabled for all widgets)
- Existing configurations continue to work unchanged

#### 1.9 Documentation Updates

**CLAUDE.md:**
```markdown
### Threshold-Based Formatting
Widgets can change colors and bold formatting based on configurable thresholds:
- Warning threshold: Changes to warning color/bold at specified value
- Critical threshold: Changes to critical color/bold at specified value
- Supported widgets: ContextPercentage, ContextPercentageUsable, TokensTotal, SessionCost
```

**README.md:**
Add section explaining threshold configuration in TUI.

---

## Feature 2: Weekly Limit Tracking

### Current Status: NOT IMPLEMENTED

#### Investigation Findings

**What was searched:**
- All widget implementations for limit tracking
- Token and cost calculation in `src/utils/jsonl.ts`
- Settings schema for limit configuration
- Search for "weekly", "limit", "reset" keywords

**What was found:**
- Only session-based metrics exist (SessionCost, TokensTotal, etc.)
- No weekly aggregation or limit tracking
- No reset date/time functionality
- No external API integration for usage data

**Code Evidence:**
```typescript
// SessionCost.ts only calculates current session cost
render(item: WidgetItem, context: RenderContext) {
    if (!context.data.session_cost) return null;
    const cost = context.data.session_cost.toFixed(2);
    return item.rawValue ? `$${cost}` : `Cost: $${cost}`;
}
// No weekly tracking or limit comparison
```

### Implementation Plan

#### 2.1 Architecture Design

**Challenge: Data Source**
Claude's weekly limit data is not available in the transcript JSON. Options:
1. **Manual Configuration:** User enters their weekly limit, app tracks usage from transcripts
2. **API Integration:** Call Claude API for usage data (requires auth/API key)
3. **Hybrid:** Manual limit + local tracking with periodic API sync

**Recommended Approach:** Manual Configuration (simplest, no auth required)

**New Types** (`src/types/UsageLimit.ts`):
```typescript
export interface UsageLimit {
    enabled: boolean;
    weeklyLimit: number;  // In USD
    resetDay: 0 | 1 | 2 | 3 | 4 | 5 | 6;  // 0=Sunday, 1=Monday, etc.
    resetTime: string;  // HH:MM in local time
    timezone: string;  // IANA timezone (e.g., 'America/New_York')
}

export interface UsageData {
    weekStart: string;  // ISO date string
    weekEnd: string;    // ISO date string
    totalCost: number;  // Accumulated cost
    percentage: number; // Percentage of limit used
}
```

**Settings Schema Update** (`src/types/Settings.ts`):
```typescript
export const SettingsSchema = z.object({
    // ... existing fields
    usageLimit: z.object({
        enabled: z.boolean().default(false),
        weeklyLimit: z.number().default(100),  // $100 default
        resetDay: z.number().min(0).max(6).default(1),  // Monday
        resetTime: z.string().default('00:00'),
        timezone: z.string().default('UTC')
    }).optional()
});
```

#### 2.2 Usage Tracking System

**New Utility** (`src/utils/usage-tracker.ts`):
```typescript
import * as fs from 'fs';
import * as os from 'os';
import * as path from 'path';

const USAGE_DATA_PATH = path.join(os.homedir(), '.config', 'ccstatusline', 'usage-data.json');

export interface WeeklyUsageRecord {
    weekStart: string;
    totalCost: number;
    sessions: Array<{
        date: string;
        cost: number;
        transcriptPath: string;
    }>;
}

export async function loadUsageData(): Promise<WeeklyUsageRecord | null> {
    try {
        if (!fs.existsSync(USAGE_DATA_PATH)) return null;
        const content = await fs.promises.readFile(USAGE_DATA_PATH, 'utf-8');
        return JSON.parse(content) as WeeklyUsageRecord;
    } catch {
        return null;
    }
}

export async function saveUsageData(data: WeeklyUsageRecord): Promise<void> {
    await fs.promises.writeFile(USAGE_DATA_PATH, JSON.stringify(data, null, 2), 'utf-8');
}

export function getWeekBounds(settings: Settings): { start: Date; end: Date } {
    const now = new Date();
    const config = settings.usageLimit!;

    // Calculate most recent reset day/time
    const currentDay = now.getDay();
    const targetDay = config.resetDay;
    const daysAgo = (currentDay - targetDay + 7) % 7;

    const weekStart = new Date(now);
    weekStart.setDate(now.getDate() - daysAgo);
    const [hours, minutes] = config.resetTime.split(':').map(Number);
    weekStart.setHours(hours!, minutes!, 0, 0);

    // If we haven't reached reset time today and today is reset day, go back one week
    if (daysAgo === 0 && now < weekStart) {
        weekStart.setDate(weekStart.getDate() - 7);
    }

    const weekEnd = new Date(weekStart);
    weekEnd.setDate(weekStart.getDate() + 7);

    return { start: weekStart, end: weekEnd };
}

export async function calculateWeeklyUsage(
    settings: Settings,
    currentSessionCost?: number
): Promise<UsageData | null> {
    if (!settings.usageLimit?.enabled) return null;

    const { start, end } = getWeekBounds(settings);
    const usageData = await loadUsageData();

    // Check if current week
    const isCurrentWeek = usageData && new Date(usageData.weekStart).getTime() === start.getTime();

    let totalCost = 0;
    if (isCurrentWeek && usageData) {
        totalCost = usageData.totalCost;
    }

    // Add current session cost if provided
    if (currentSessionCost) {
        totalCost += currentSessionCost;
    }

    const percentage = (totalCost / settings.usageLimit.weeklyLimit) * 100;

    return {
        weekStart: start.toISOString(),
        weekEnd: end.toISOString(),
        totalCost,
        percentage: Math.min(100, percentage)
    };
}

export async function updateUsageWithSession(
    settings: Settings,
    sessionCost: number,
    transcriptPath: string
): Promise<void> {
    if (!settings.usageLimit?.enabled) return;

    const { start } = getWeekBounds(settings);
    const usageData = await loadUsageData();

    const isCurrentWeek = usageData && new Date(usageData.weekStart).getTime() === start.getTime();

    if (isCurrentWeek && usageData) {
        // Update existing week
        usageData.totalCost += sessionCost;
        usageData.sessions.push({
            date: new Date().toISOString(),
            cost: sessionCost,
            transcriptPath
        });
    } else {
        // Start new week
        const newData: WeeklyUsageRecord = {
            weekStart: start.toISOString(),
            totalCost: sessionCost,
            sessions: [{
                date: new Date().toISOString(),
                cost: sessionCost,
                transcriptPath
            }]
        };
        await saveUsageData(newData);
    }
}
```

#### 2.3 Widget Implementation

**New Widget** (`src/widgets/UsageLimit.ts`):
```typescript
import { calculateWeeklyUsage } from '../utils/usage-tracker';
import type { Widget, WidgetItem, RenderContext, Settings } from '../types';

export const UsageLimitWidget: Widget = {
    getDefaultColor: () => 'cyan',

    getDescription: () => 'Shows weekly usage vs limit with % and reset time',

    getDisplayName: () => 'Usage Limit',

    getEditorDisplay: (item) => ({
        displayText: item.rawValue ? 'Usage Limit (raw)' : 'Usage Limit'
    }),

    render: async (item: WidgetItem, context: RenderContext, settings: Settings) => {
        if (!settings.usageLimit?.enabled) {
            return null;
        }

        const currentSessionCost = context.data.session_cost ?? 0;
        const usageData = await calculateWeeklyUsage(settings, currentSessionCost);

        if (!usageData) return null;

        const resetDate = new Date(usageData.weekEnd);
        const now = new Date();
        const daysUntil = Math.ceil((resetDate.getTime() - now.getTime()) / (1000 * 60 * 60 * 24));

        if (item.rawValue) {
            // Raw format: $45.67/100 (45.7%) 3d
            return `$${usageData.totalCost.toFixed(2)}/${settings.usageLimit.weeklyLimit} (${usageData.percentage.toFixed(1)}%) ${daysUntil}d`;
        } else {
            // Full format: Usage: $45.67/$100 (45.7%) • Reset: 3d
            return `Usage: $${usageData.totalCost.toFixed(2)}/$${settings.usageLimit.weeklyLimit} (${usageData.percentage.toFixed(1)}%) • Reset: ${daysUntil}d`;
        }
    },

    supportsRawValue: () => true,

    supportsColors: () => true,

    supportsThresholds: () => true,  // Works with Feature 1

    getCurrentValue: async (item: WidgetItem, context: RenderContext, settings: Settings) => {
        const currentSessionCost = context.data.session_cost ?? 0;
        const usageData = await calculateWeeklyUsage(settings, currentSessionCost);
        return usageData?.percentage ?? null;
    },

    getThresholdRange: () => ({ min: 0, max: 100 })
};
```

**Register Widget** (`src/utils/widgets.ts`):
```typescript
import { UsageLimitWidget } from '../widgets/UsageLimit';
registerWidget('usage-limit', UsageLimitWidget);
```

#### 2.4 Session Cost Tracking Hook

**Update Main Entry Point** (`src/ccstatusline.ts`):
```typescript
import { updateUsageWithSession } from './utils/usage-tracker';

async function renderMultipleLines(data: StatusJSON) {
    // ... existing rendering code

    // After rendering, update usage tracking if session cost is available
    if (data.session_cost && data.transcript_path && settings.usageLimit?.enabled) {
        // Only update once per render to avoid double-counting
        // (This runs each time status line updates, so track last update)
        await updateUsageWithSession(settings, data.session_cost, data.transcript_path);
    }
}
```

#### 2.5 TUI Configuration Screen

**New Component** (`src/tui/components/UsageLimitMenu.tsx`):
```typescript
interface Props {
    settings: Settings;
    onSave: (settings: Settings) => void;
    onBack: () => void;
}

export const UsageLimitMenu: React.FC<Props> = ({ settings, onSave, onBack }) => {
    // UI for configuring usage limits:
    // - Enable/disable tracking (ENTER)
    // - Set weekly limit in USD (numeric input)
    // - Set reset day (0-6, dropdown)
    // - Set reset time (HH:MM input)
    // - Set timezone (text input with validation)
    // - Show current usage preview
    // - Button to reset current week's data
};
```

#### 2.6 Testing Strategy

**Unit Tests** (`src/utils/__tests__/usage-tracker.test.ts`):
```typescript
describe('getWeekBounds', () => {
    it('calculates correct week start for Monday reset', () => {
        const bounds = getWeekBounds(mondaySettings);
        expect(bounds.start.getDay()).toBe(1);
    });

    it('handles timezone correctly', () => {
        // Test various timezones
    });
});

describe('calculateWeeklyUsage', () => {
    it('returns null when disabled', async () => {
        const result = await calculateWeeklyUsage(disabledSettings);
        expect(result).toBeNull();
    });

    it('calculates percentage correctly', async () => {
        const result = await calculateWeeklyUsage(settings, 50);
        expect(result.percentage).toBe(50);
    });

    it('caps percentage at 100%', async () => {
        const result = await calculateWeeklyUsage(settings, 150);
        expect(result.percentage).toBe(100);
    });
});
```

#### 2.7 Considerations & Edge Cases

**Data Persistence:**
- Store usage data in `~/.config/ccstatusline/usage-data.json`
- Handle file corruption gracefully (backup on error)

**Timezone Handling:**
- Use system timezone by default
- Validate IANA timezone strings
- Handle DST transitions correctly

**Session Tracking:**
- Avoid double-counting same session
- Track by transcript path + timestamp
- Handle sessions spanning week boundary

**Reset Behavior:**
- Auto-archive old week's data
- Keep last 12 weeks for historical view (future enhancement)

**Accuracy Limitations:**
- Manual tracking may miss sessions outside ccstatusline
- No way to sync with Claude's actual billing data
- User must configure limit manually

#### 2.8 Future Enhancements

1. **API Integration:** Optional Claude API sync for accurate data
2. **Historical View:** Show past weeks' usage in TUI
3. **Notifications:** Alert when approaching limit (80%, 90%, 100%)
4. **Multiple Limits:** Support monthly limits, token limits, etc.

---

## Feature 3: Dirty File Count Display

### Current Status: NOT IMPLEMENTED

#### Investigation Findings

**What was searched:**
- GitChanges widget implementation
- Git command usage in `src/widgets/GitChanges.ts`
- GitBranch and GitWorktree widgets for file tracking

**What was found:**
- GitChanges widget uses `git diff --shortstat` which reports **line counts** only
- No file counting logic anywhere in codebase
- Format: `(+42,-10)` showing line insertions/deletions

**Code Evidence:**
```typescript
// From GitChanges.ts lines 63-87
const unstagedStat = execSync('git diff --shortstat', { cwd, encoding: 'utf-8' });
const stagedStat = execSync('git diff --cached --shortstat', { cwd, encoding: 'utf-8' });

// Parse patterns extract numbers from:
// " 3 files changed, 42 insertions(+), 10 deletions(-)"
// But only insertMatch and deleteMatch are used, not fileMatch
```

### Implementation Plan

#### 3.1 Architecture Design

**Options:**
1. **Replace line counts with file counts:** `(5 files)` instead of `(+42,-10)`
2. **Show both:** `(5 files: +42,-10 lines)`
3. **Separate widgets:** Keep GitChanges for lines, add GitFileCount widget
4. **Configurable mode:** User chooses what to display

**Recommended Approach:** Configurable mode (backwards compatible)

**New Types** (`src/types/Widget.ts` - extend WidgetItem):
```typescript
export const WidgetItemSchema = z.object({
    // ... existing fields
    gitDisplayMode: z.enum(['files', 'lines', 'both']).optional()  // New field for GitChanges
});
```

#### 3.2 Widget Implementation Update

**Update GitChanges Widget** (`src/widgets/GitChanges.ts`):
```typescript
import { execSync } from 'child_process';
import type { Widget, WidgetItem, RenderContext } from '../types';

interface GitFileStatus {
    modified: number;   // Modified unstaged files
    staged: number;     // Staged files
    untracked: number;  // Untracked files
}

function getFileCount(cwd: string): GitFileStatus | null {
    try {
        // Use git status --porcelain for file counts
        const output = execSync('git status --porcelain', {
            cwd,
            encoding: 'utf-8',
            stdio: ['pipe', 'pipe', 'ignore']
        });

        const lines = output.trim().split('\n').filter(l => l.trim());
        let modified = 0;
        let staged = 0;
        let untracked = 0;

        for (const line of lines) {
            const status = line.substring(0, 2);

            // First char: staged status, second char: unstaged status
            // ' M' = modified unstaged
            // 'M ' = modified staged
            // 'MM' = modified both
            // '??' = untracked

            if (status === '??') {
                untracked++;
            } else {
                if (status[0] !== ' ' && status[0] !== '?') staged++;
                if (status[1] !== ' ' && status[1] !== '?') modified++;
            }
        }

        return { modified, staged, untracked };
    } catch {
        return null;
    }
}

export const GitChangesWidget: Widget = {
    getDefaultColor: () => 'yellow',

    getDescription: () => 'Shows git changes (files or lines)',

    getDisplayName: () => 'Git Changes',

    getEditorDisplay: (item) => {
        const mode = item.gitDisplayMode ?? 'lines';
        return {
            displayText: `Git Changes (${mode})`,
            modifierText: item.hideNoGitMessage ? 'hide no-git' : undefined
        };
    },

    render: (item: WidgetItem, context: RenderContext) => {
        const cwd = context.data.cwd ?? process.cwd();

        // Check if in git repo
        try {
            execSync('git rev-parse --git-dir', {
                cwd,
                stdio: 'ignore'
            });
        } catch {
            return item.hideNoGitMessage ? null : 'no git';
        }

        const mode = item.gitDisplayMode ?? 'lines';

        if (mode === 'files' || mode === 'both') {
            const fileStatus = getFileCount(cwd);
            if (!fileStatus) return null;

            const total = fileStatus.modified + fileStatus.staged + fileStatus.untracked;
            if (total === 0) return null;

            if (mode === 'files') {
                // File count only: (5 files)
                const parts: string[] = [];
                if (fileStatus.staged > 0) parts.push(`${fileStatus.staged} staged`);
                if (fileStatus.modified > 0) parts.push(`${fileStatus.modified} modified`);
                if (fileStatus.untracked > 0) parts.push(`${fileStatus.untracked} untracked`);

                if (item.rawValue) {
                    return `${total}`;
                } else {
                    return `(${parts.join(', ')})`;
                }
            } else {
                // Both: (5 files: +42,-10)
                const lineStats = getLineStats(cwd);  // Existing function
                if (!lineStats) return `(${total} files)`;

                const { insertions, deletions } = lineStats;
                if (item.rawValue) {
                    return `${total} +${insertions},-${deletions}`;
                } else {
                    return `(${total} files: +${insertions},-${deletions})`;
                }
            }
        } else {
            // Lines mode: +42,-10 (existing behavior)
            const lineStats = getLineStats(cwd);
            if (!lineStats) return null;

            const { insertions, deletions } = lineStats;
            if (insertions === 0 && deletions === 0) return null;

            return `(+${insertions},-${deletions})`;
        }
    },

    supportsRawValue: () => true,

    supportsColors: () => true,

    getCustomKeybinds: () => [
        { key: 'd', label: 'display mode', action: 'cycle-display-mode' },
        { key: 'h', label: 'hide no-git', action: 'toggle-hide-no-git' }
    ],

    handleEditorAction: (action: string, item: WidgetItem) => {
        if (action === 'cycle-display-mode') {
            const modes: Array<'files' | 'lines' | 'both'> = ['files', 'lines', 'both'];
            const current = item.gitDisplayMode ?? 'lines';
            const currentIndex = modes.indexOf(current);
            const nextIndex = (currentIndex + 1) % modes.length;
            return { ...item, gitDisplayMode: modes[nextIndex] };
        }

        if (action === 'toggle-hide-no-git') {
            return { ...item, hideNoGitMessage: !item.hideNoGitMessage };
        }

        return null;
    }
};

// Helper function for existing line stats
function getLineStats(cwd: string): { insertions: number; deletions: number } | null {
    // ... existing implementation from lines 63-87
}
```

#### 3.3 TUI Integration

**Update ItemsEditor** (`src/tui/components/ItemsEditor.tsx`):
- Display mode indicator in editor: `Git Changes (files)`, `Git Changes (lines)`, `Git Changes (both)`
- Show keyboard shortcut: `(d) display mode`
- Handle custom keybind action

#### 3.4 Testing Strategy

**Unit Tests** (`src/widgets/__tests__/GitChanges.test.ts`):
```typescript
describe('getFileCount', () => {
    it('counts modified files correctly', () => {
        // Mock git status output
        const status = getFileCount('/test/repo');
        expect(status.modified).toBe(3);
        expect(status.staged).toBe(2);
        expect(status.untracked).toBe(1);
    });

    it('handles empty repo', () => {
        const status = getFileCount('/empty/repo');
        expect(status).toEqual({ modified: 0, staged: 0, untracked: 0 });
    });
});

describe('GitChangesWidget.render', () => {
    it('renders file count in files mode', () => {
        const result = widget.render({ gitDisplayMode: 'files' }, context);
        expect(result).toBe('(2 staged, 3 modified, 1 untracked)');
    });

    it('renders line count in lines mode', () => {
        const result = widget.render({ gitDisplayMode: 'lines' }, context);
        expect(result).toBe('(+42,-10)');
    });

    it('renders both in both mode', () => {
        const result = widget.render({ gitDisplayMode: 'both' }, context);
        expect(result).toBe('(6 files: +42,-10)');
    });
});
```

**Manual Testing:**
```bash
# Test in various git states
cd /tmp/test-repo
git init
echo "test" > file1.txt
# Test untracked files
git add file1.txt
# Test staged files
echo "change" > file1.txt
# Test modified files
```

#### 3.5 Migration Strategy

**Default Behavior:**
- Existing configurations default to `gitDisplayMode: 'lines'` (preserve current behavior)
- New installations can choose mode in TUI
- No breaking changes

#### 3.6 Documentation Updates

**CLAUDE.md:**
```markdown
### Git Widgets
- **GitChanges**: Configurable display modes:
  - `files`: Show file counts (2 staged, 3 modified, 1 untracked)
  - `lines`: Show line counts (+42,-10) [default]
  - `both`: Show both (6 files: +42,-10)
```

**README.md:**
Add explanation of display modes with screenshots.

---

## Feature 4: --no-optional-locks Flag for Git Commands

### Current Status: NOT IMPLEMENTED

#### Investigation Findings

**What was searched:**
- All git command executions in codebase
- GitChanges, GitBranch, GitWorktree widgets

**What was found:**
- 4 git commands executed without `--no-optional-locks`:
  1. `git diff --shortstat` (GitChanges.ts:63)
  2. `git diff --cached --shortstat` (GitChanges.ts:68)
  3. `git branch --show-current` (GitBranch.ts:60)
  4. `git rev-parse --git-dir` (GitWorktree.ts:58)

**Code Evidence:**
```typescript
// GitChanges.ts:63
const unstagedStat = execSync('git diff --shortstat', {
    cwd,
    encoding: 'utf-8',
    stdio: ['pipe', 'pipe', 'ignore']
});
// Missing --no-optional-locks flag
```

**Why This Matters:**
- Status line runs frequently (on every Claude Code update)
- Without `--no-optional-locks`, git may create lock files
- Can cause lock contention with concurrent git operations
- Can fail if index.lock already exists

### Implementation Plan

#### 4.1 Architecture Decision

**Approach:** Add `--no-optional-locks` to all read-only git commands

**Git Version Compatibility:**
- Flag added in Git 2.15.0 (October 2017)
- As of 2025, this is widely available (7+ years old)
- Node.js 14+ required by project (released 2020), so git 2.15+ is reasonable assumption

**Error Handling Strategy:**
1. Add flag to all commands
2. If command fails with "unknown option" error, retry without flag
3. Cache result to avoid repeated retries

#### 4.2 Implementation

**New Utility** (`src/utils/git.ts`):
```typescript
import { execSync } from 'child_process';

let supportsNoOptionalLocks: boolean | null = null;

export function checkGitSupportsNoOptionalLocks(cwd: string): boolean {
    if (supportsNoOptionalLocks !== null) {
        return supportsNoOptionalLocks;
    }

    try {
        execSync('git --no-optional-locks status', {
            cwd,
            stdio: 'ignore'
        });
        supportsNoOptionalLocks = true;
    } catch {
        supportsNoOptionalLocks = false;
    }

    return supportsNoOptionalLocks;
}

export function execGitCommand(
    command: string,
    cwd: string,
    options?: { encoding?: 'utf-8'; stdio?: any }
): string {
    const supportsFlag = checkGitSupportsNoOptionalLocks(cwd);

    // Insert --no-optional-locks after 'git' if supported
    const finalCommand = supportsFlag
        ? command.replace(/^git\s/, 'git --no-optional-locks ')
        : command;

    return execSync(finalCommand, {
        cwd,
        encoding: options?.encoding ?? 'utf-8',
        stdio: options?.stdio ?? ['pipe', 'pipe', 'ignore']
    }) as string;
}
```

**Update GitChanges** (`src/widgets/GitChanges.ts`):
```typescript
import { execGitCommand } from '../utils/git';

// Line 63
const unstagedStat = execGitCommand('git diff --shortstat', cwd);

// Line 68
const stagedStat = execGitCommand('git diff --cached --shortstat', cwd);

// Also in Feature 3 implementation (git status --porcelain)
const output = execGitCommand('git status --porcelain', cwd);
```

**Update GitBranch** (`src/widgets/GitBranch.ts`):
```typescript
import { execGitCommand } from '../utils/git';

// Line 60
const branch = execGitCommand('git branch --show-current', cwd).trim();
```

**Update GitWorktree** (`src/widgets/GitWorktree.ts`):
```typescript
import { execGitCommand } from '../utils/git';

// Line 58
const gitDir = execGitCommand('git rev-parse --git-dir', cwd).trim();
```

**Update git repo detection** (multiple files):
```typescript
// Wherever 'git rev-parse --git-dir' is used for detection
try {
    execGitCommand('git rev-parse --git-dir', cwd, { stdio: 'ignore' });
    // In git repo
} catch {
    // Not in git repo
}
```

#### 4.3 Testing Strategy

**Unit Tests** (`src/utils/__tests__/git.test.ts`):
```typescript
describe('checkGitSupportsNoOptionalLocks', () => {
    it('detects support correctly', () => {
        const supports = checkGitSupportsNoOptionalLocks('/test/repo');
        expect(typeof supports).toBe('boolean');
    });

    it('caches result', () => {
        checkGitSupportsNoOptionalLocks('/test/repo');
        // Call again, should use cached value
        const result = checkGitSupportsNoOptionalLocks('/test/repo');
        expect(result).toBeDefined();
    });
});

describe('execGitCommand', () => {
    it('adds --no-optional-locks when supported', () => {
        const spy = jest.spyOn(child_process, 'execSync');
        execGitCommand('git status', '/test/repo');
        expect(spy).toHaveBeenCalledWith(
            expect.stringContaining('--no-optional-locks'),
            expect.any(Object)
        );
    });

    it('omits flag when not supported', () => {
        // Mock old git version
        supportsNoOptionalLocks = false;
        const spy = jest.spyOn(child_process, 'execSync');
        execGitCommand('git status', '/test/repo');
        expect(spy).toHaveBeenCalledWith(
            expect.not.stringContaining('--no-optional-locks'),
            expect.any(Object)
        );
    });
});
```

**Integration Testing:**
```bash
# Test with concurrent git operations
# Terminal 1: Run ccstatusline in watch mode
while true; do echo '{"model":{"display_name":"Test"}}' | bun run src/ccstatusline.ts; sleep 0.1; done

# Terminal 2: Run git operations
cd /test/repo
while true; do git status; git add .; git commit -m "test"; sleep 0.5; done

# Verify no "index.lock" errors
```

#### 4.4 Edge Cases

**Old Git Versions:**
- Gracefully fallback to commands without flag
- Cache detection result to avoid overhead

**Non-Git Directories:**
- Detection check handles this (no-optional-locks not needed)

**Windows Git:**
- Flag supported in Git for Windows 2.15.0+
- Same fallback logic applies

#### 4.5 Migration Strategy

**No Breaking Changes:**
- Purely additive change
- Improves reliability without affecting functionality
- Automatic fallback for old git versions

#### 4.6 Documentation Updates

**CLAUDE.md:**
```markdown
## Key Implementation Details

- **Git commands**: All read-only git commands use `--no-optional-locks` flag to prevent lock contention (git 2.15.0+, with automatic fallback for older versions)
```

---

## Implementation Priority & Timeline

### Recommended Order

1. **Feature 4: --no-optional-locks flag** (1-2 hours)
   - Lowest risk, high value
   - Improves reliability immediately
   - No user-facing changes needed

2. **Feature 3: Dirty file count** (3-4 hours)
   - Low complexity
   - High user value
   - Clear use case

3. **Feature 1: Threshold-based formatting** (8-12 hours)
   - Medium complexity
   - High user value
   - Foundation for Feature 2

4. **Feature 2: Weekly limit tracking** (12-16 hours)
   - Highest complexity
   - Data persistence required
   - External dependency concerns

### Estimated Timeline

- **Phase 1** (Features 4 + 3): 1-2 days
- **Phase 2** (Feature 1): 2-3 days
- **Phase 3** (Feature 2): 3-4 days
- **Total**: 6-9 days with testing

---

## Testing Strategy

### Unit Tests
- All new utilities in `src/utils/__tests__/`
- All updated widgets in `src/widgets/__tests__/`
- Target coverage: >80% for new code

### Integration Tests
- Manual testing with real git repositories
- Test various git states (clean, dirty, staged, etc.)
- Test with different git versions
- Test threshold triggers
- Test weekly limit rollover

### User Acceptance Testing
- Test TUI configuration screens
- Test status line rendering in Claude Code
- Test with various terminal emulators
- Test on Windows, macOS, Linux

---

## Backward Compatibility

All features maintain backward compatibility:

1. **Feature 1**: Thresholds opt-in, default disabled
2. **Feature 2**: Usage tracking opt-in, default disabled
3. **Feature 3**: Display mode defaults to 'lines' (current behavior)
4. **Feature 4**: Automatic fallback for old git versions

Existing configurations continue to work unchanged.

---

## Documentation Requirements

### Update Files

1. **CLAUDE.md**
   - Add sections for each new feature
   - Update architecture overview
   - Document new utility files

2. **README.md**
   - Add usage examples
   - Add screenshots
   - Update widget list
   - Add configuration guide

3. **API Documentation** (TypeDoc)
   - Document new types
   - Document new utilities
   - Add examples

### New Files

1. Create `docs/THRESHOLDS.md` - Threshold configuration guide
2. Create `docs/USAGE_LIMITS.md` - Usage limit setup guide

---

## Potential Issues & Mitigations

### Feature 1: Thresholds

**Issue:** Performance overhead from threshold evaluation on every render
**Mitigation:** Cache threshold results, only re-evaluate when value changes significantly

**Issue:** Color conflicts with user-configured colors
**Mitigation:** Thresholds override widget color but respect global overrides

### Feature 2: Usage Limits

**Issue:** Inaccurate tracking if sessions run outside ccstatusline
**Mitigation:** Document limitation clearly, consider API integration in future

**Issue:** Timezone handling complexity
**Mitigation:** Default to system timezone, validate IANA strings, clear error messages

**Issue:** Week boundary edge cases
**Mitigation:** Comprehensive unit tests for all scenarios

### Feature 3: File Counts

**Issue:** Performance with large repos (many files)
**Mitigation:** `git status --porcelain` is fast, no tree traversal needed

**Issue:** Display too verbose with many file types
**Mitigation:** Offer compact mode (just total) and detailed mode

### Feature 4: Git Locks

**Issue:** Old git versions don't support flag
**Mitigation:** Automatic detection and fallback

**Issue:** Detection adds overhead
**Mitigation:** Cache detection result globally

---

## Future Enhancements

Beyond these 4 features, consider:

1. **Historical Usage View**: Chart showing past weeks' usage
2. **Budget Alerts**: Configurable notifications at threshold levels
3. **API Integration**: Sync with Claude's official usage API
4. **Multiple Limit Types**: Token limits, request limits, etc.
5. **Team Usage**: Aggregate usage across team members
6. **Export Data**: CSV/JSON export of usage history

---

## Conclusion

All 4 requested features are feasible to implement with varying levels of complexity:

- **Feature 4** (git locks) is trivial and should be done immediately
- **Feature 3** (file counts) is straightforward with high value
- **Feature 1** (thresholds) requires careful architecture but provides excellent UX
- **Feature 2** (usage limits) is most complex but fills a real need

The recommended approach is to implement in order 4 → 3 → 1 → 2, allowing early wins while building foundation for more complex features.

All features maintain backward compatibility and follow the project's existing patterns and conventions.
