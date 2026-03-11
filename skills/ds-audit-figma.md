---
description: Figma-to-Storybook visual parity audit (read-only spot-check)
---

# Figma–Storybook Parity Audit

Run a comprehensive audit comparing the Figma Design System file against the Storybook component library to ensure visual and structural parity.

## Prerequisites

- Figma Desktop Bridge plugin must be running
- Your Figma design system file must be open
- `.claude/ds-story-figma-map.json` — pre-built Storybook↔Figma ID mapping
- `design-system-manifest.json` — component inventory (optional, enriches the audit)

## Arguments

- `$ARGUMENTS` — optional section or component filter. Examples:
  - `Button` — audit Button only
  - `"Actions"` — audit the Actions section
  - (empty) — audit ALL sections

---

## Steps

### 1. Load the Mapping File & Manifest

1. **Read `.claude/ds-story-figma-map.json`** — section frame IDs, verify screen IDs, component Figma node IDs, Storybook story IDs, and key variable IDs.
2. **Read `design-system-manifest.json`** for argTypes, prop definitions, and variant metadata.

### 2. Query Figma Inventory

Use `figma_execute` to enumerate all sections and their children on the Design System page. Cross-reference with section frame IDs from the mapping:

```js
const page = figma.currentPage;
const sections = page.children.filter(c => c.type === 'SECTION' || c.type === 'FRAME');
const inventory = sections.map(s => ({
  name: s.name,
  id: s.id,
  children: s.children.map(c => ({
    name: c.name,
    id: c.id,
    type: c.type,
    childCount: 'children' in c ? c.children.length : 0
  }))
}));
return inventory;
```

### 3. Cross-Reference

For each story in the manifest:
- Find the matching Figma section (by section name)
- Find the matching component within that section (by component name)
- Check if all story variants exist as Figma variants
- Flag any mismatches in naming

### 4. Check Icons

Compare the manifest's icon lists against Figma's Icons section (if it exists). Flag any missing icons.

### 5. Check Variables

Use `figma_get_variables` to verify all semantic colour tokens exist and have both Light and Dark mode values.

### 6. Visual Spot-Check

For each component:
- Use `figma_capture_screenshot` to take a screenshot
- Compare key visual properties (colours, spacing, typography) against the source code's styling classes

### 7. Generate Report

Output a markdown parity report with:
- Components that match
- Components with minor discrepancies
- Missing components (in one system but not the other)
- Variable/token coverage
- Summary statistics (% parity)

```markdown
# Figma–Storybook Parity Audit
**Date:** [ISO date]
**Components audited:** N

## Summary
- **Full parity:** X/N (Y%)
- **Minor drift:** X components
- **Missing from Figma:** X components
- **Missing from Storybook:** X components

## Component Matrix

| Component | Section | In Storybook | In Figma | Variants Match | Visual Match | Status |
|-----------|---------|-------------|----------|---------------|-------------|--------|
| Button    | Actions | ✅          | ✅       | ✅ (12/12)    | ✅          | MATCH  |
| Input     | Forms   | ✅          | ✅       | ⚠️ (5/7)      | ⚠️          | DRIFT  |
| DatePicker| Forms   | ✅          | ❌       | —             | —           | MISSING|

## Action Items
1. [Prioritised list of fixes]
```

---

## Guidelines

- **Read-only** — this skill never modifies code or Figma
- **Visual-first** — use screenshots for comparison, not just metadata
- **Token-aware** — check that Figma components use variable bindings, not hardcoded values
- **Both themes** — spot-check in both light and dark modes

## Usage

```bash
# Full audit
/ds-audit-figma

# Audit a section
/ds-audit-figma "Actions"

# Audit a specific component
/ds-audit-figma Button
```
