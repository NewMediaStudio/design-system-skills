---
description: Scan the codebase for design system component adoption, usage patterns, and override anti-patterns
---

# Design System Usage Audit

Scan your application codebase to measure design system adoption: which components are used, where, how often, and whether consumers are overriding or wrapping them instead of using them as intended. Outputs an adoption report with health metrics per component and per team/directory.

## Prerequisites

- `.claude/ds-registry.json` — unified component registry (optional, enables fast path)
- `design-system-manifest.json` — component export list (fallback)
- Application source directories accessible (e.g., `apps/`, `src/`, `packages/app/`)

## Arguments

- `$ARGUMENTS` — optional flags and filters:
  - `Button` — report on a specific component only
  - `apps/dashboard` — scope to a specific directory
  - `--overrides` — focus on override and wrapping patterns
  - `--unused` — report only components with zero usage
  - `--teams` — break down usage by team/directory
  - (empty) — full adoption report for all components

---

## Phase 1: Build Component Export List

### 1.1 Load Registry (Fast Path)

If `.claude/ds-registry.json` exists, read `registry.components[].name` and `registry.components[].exportName` to get the full list of DS component exports.

### 1.2 Fallback: Read Barrel Export

If the registry is absent, read the DS barrel export file (e.g., `packages/ds/src/main.tsx` or `packages/ds/src/index.ts`) and extract all named exports that are PascalCase (components) and camelCase hooks.

Build a component list:

```json
[
  { "name": "Button", "exportName": "Button", "aliases": ["Btn"] },
  { "name": "Input", "exportName": "Input", "aliases": [] },
  { "name": "Dialog", "exportName": "Dialog", "aliases": ["Modal"] }
]
```

**Total DS exports:** N components, M hooks

---

## Phase 2: Codebase Scan

### 2.1 Locate Application Source

Determine which directories contain application code (not the DS package itself). Common paths:

- `apps/*/src/`
- `src/` (monorepo root app)
- `packages/*/src/` (excluding the DS package)

If scoping to a specific directory via `$ARGUMENTS`, restrict the scan to that path.

### 2.2 Import Pattern Scan

For each DS component, search for import statements across application source:

```bash
# Pattern: import { ComponentName } from '@yourorg/design-system'
# or from the local DS package path
grep -r "import.*{ *ComponentName" apps/ src/ --include="*.tsx" --include="*.ts" --include="*.jsx"
```

For each match, record:
- File path
- Import source (DS package name or relative path)
- Whether it's a named import, default import, or re-export

**Flag re-exports:** If a file imports from the DS and then re-exports the component, flag it as a potential **shadow copy** (local wrapper that consumers may import instead of the DS directly).

### 2.3 Usage Pattern Scan

For each imported component, count JSX usages in the same file:

```bash
# Count: <Button, <Button , <Button>
grep -c "<ComponentName[^a-zA-Z]" filepath
```

Record per-file usage count.

### 2.4 Override Pattern Detection

Detect common anti-patterns where consumers modify DS components in ways that bypass the design system:

**Pattern 1 — className override:**
```bash
grep -r "<Button className=" apps/ --include="*.tsx"
```
Overrides suggest the DS component lacks a needed variant. Flag the component and the className value used.

**Pattern 2 — Style prop override:**
```bash
grep -r "<Button style=" apps/ --include="*.tsx"
```
Direct style overrides are highest priority to eliminate.

**Pattern 3 — Local re-implementation:**
Search for files that define a function/component with the same name as a DS export but import from a non-DS path:
```bash
grep -rn "export.*function Button\|export.*const Button\|export default.*Button" apps/ --include="*.tsx"
```

**Pattern 4 — Nested wrapper:**
Files that import a DS component and immediately export a new component with a similar name:
```bash
grep -rn "import.*Button.*from.*design-system" apps/ --include="*.tsx" | xargs grep -l "export.*Button"
```

**Pattern 5 — Deprecated prop usage:**
If the registry has a `deprecatedProps` list, scan for usage of those props:
```bash
grep -rn "oldPropName=" apps/ --include="*.tsx"
```

---

## Phase 3: Usage Metrics

### 3.1 Per-Component Metrics

For each DS component, calculate:

| Metric | Description |
|--------|-------------|
| **Import count** | Number of files that import it |
| **Usage count** | Total JSX usages across all files |
| **Override count** | Files applying className, style, or wrapper overrides |
| **Shadow copies** | Local re-implementations found |
| **Adoption rate** | Import count / total app files (as %) |
| **Override rate** | Override count / Import count (as %) |

### 3.2 Per-Team / Per-Directory Breakdown

If `--teams` flag is set, group by top-level directory (e.g., `apps/dashboard`, `apps/settings`, `packages/mobile`):

| Team/Directory | DS Imports | Total Components Used | Override Rate | Health |
|---------------|:---:|:---:|:---:|:---:|
| apps/dashboard | 234 | 18/22 (82%) | 4% | ✅ Good |
| apps/settings | 89 | 12/22 (55%) | 22% | ⚠️ Moderate |
| apps/mobile | 45 | 6/22 (27%) | 51% | ❌ Low |

### 3.3 Component Health Score

For each component, compute a health score:

```
health = 100
  - (overrideRate × 30)        // overrides penalise heavily
  - (shadowCopies × 20)        // local re-implementations are critical
  - (zeroUsage ? 40 : 0)       // unused components score low
  - (deprecatedPropUsage × 10) // deprecated props are a debt signal
```

Cap at 0, max 100.

---

## Phase 4: Report

Write `.claude/ds-usage-report.md`:

```markdown
# Design System Usage Report
**Date:** [ISO date]
**DS components:** N
**Application files scanned:** N
**Scope:** [directory or "all"]

## Adoption Summary

| Metric | Value |
|--------|-------|
| Components with ≥ 1 import | X/N (Y%) |
| Components with zero usage | X |
| Total DS import statements | X |
| Total JSX usages | X |
| Files with overrides | X |
| Shadow copies detected | X |

## Component Adoption

| Component | Imports | Usages | Overrides | Shadow Copies | Health |
|-----------|:---:|:---:|:---:|:---:|:---:|
| Button | 87 | 312 | 3 (3%) | 0 | ✅ 97 |
| Input | 54 | 201 | 12 (22%) | 1 | ⚠️ 60 |
| DatePicker | 2 | 4 | 0 | 0 | ✅ 100 |
| LegacyAlert | 0 | 0 | 0 | 0 | ❌ 60 (unused) |

## Override Details

| File | Component | Pattern | Value | Priority |
|------|-----------|---------|-------|----------|
| apps/dashboard/Table.tsx:34 | Button | className | `"!bg-red-500"` | P0 — hardcoded color |
| apps/settings/Form.tsx:12 | Input | style | `{ border: 'none' }` | P0 — style prop |
| apps/mobile/Alert.tsx | Button | shadow copy | local re-implementation | P0 — delete |

## Shadow Copies (Local Re-implementations)

| File | Mimics | Recommendation |
|------|--------|----------------|
| apps/mobile/components/Button.tsx | DS Button | Delete, import from DS |
| shared/ui/Modal.tsx | DS Dialog | Evaluate — may have overrides |

## Unused DS Components

| Component | Last seen in git | Recommendation |
|-----------|-----------------|----------------|
| LegacyAlert | 6 months ago | Deprecate or remove from DS |
| Breadcrumbs | 3 months ago | Verify with teams; mark deprecated if no plans |

## Recommendations

### P0 — Fix immediately
1. Remove shadow copies in `apps/mobile/` — delete local re-implementations, update imports
2. Replace hardcoded `className` overrides with DS props or new DS variants

### P1 — Plan for next sprint
1. Address 12 Input overrides in `apps/settings/` — likely need a FormInput DS variant
2. Mark LegacyAlert as deprecated in the DS registry

### P2 — Ongoing
1. Add per-team adoption targets to `.claude/ds-usage-targets.json` for tracking
2. Integrate into CI to block new shadow copies (see `/ci-integration`)
```

---

## Key Rules

1. **Read-only** — this skill never modifies application code
2. **DS package is excluded** — only scan application/consumer code, not the DS package itself
3. **Override ≠ bug** — report overrides as signals, not automatic failures; some are intentional
4. **Shadow copies are critical** — local re-implementations silently bypass all DS updates

## Usage

```bash
# Full adoption report
/ds-usage

# Report on a single component
/ds-usage Button

# Scope to a directory (team)
/ds-usage apps/dashboard

# Focus on override patterns
/ds-usage --overrides

# List only unused DS components
/ds-usage --unused

# Break down by team
/ds-usage --teams
```
