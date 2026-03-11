---
description: WCAG 2.1 Level AA accessibility audit for design system components
---

# Accessibility Checker (WCAG 2.1 AA)

Audit design system components for WCAG 2.1 Level AA compliance. Checks semantic HTML, ARIA attributes, keyboard navigation, color contrast, and screen reader support.

## Prerequisites

- Storybook dev server running on `http://localhost:6006` (launch with `/storybook` if not running)
- `.claude/ds-registry.json` — unified component registry (optional, enables fast path)
- `design-system-manifest.json` in project root (component inventory, fallback)
- `.claude/ds-story-figma-map.json` — for Storybook story IDs and source file paths (fallback)
- `.claude/rules/accessibility.md` — the full WCAG 2.1 AA criteria checklist

## Arguments

- `$ARGUMENTS` — optional component name filter. Examples:
  - `Button` — audit Button only
  - `"Forms"` — audit all form components
  - (empty) — audit ALL components

---

## Phase 1: Setup

### 1.0 Load the DS Registry (Fast Path)

If `.claude/ds-registry.json` exists, load it as the primary component inventory. The registry provides component names, source file paths, variants, props, tokens, and story IDs — replacing separate reads of the manifest and mapping file for phases 1.1–1.2.

If the registry does not exist, fall back to the multi-file approach in 1.1.

### 1.1 Load Rules & Component Inventory

1. **Read `.claude/rules/accessibility.md`** — the full WCAG 2.1 AA criteria checklist
2. **Read `design-system-manifest.json`** — component inventory with props and argTypes (skip if registry loaded)
3. **Read `.claude/ds-story-figma-map.json`** — for Storybook story IDs and source file paths (skip if registry loaded)

### 1.2 Filter Components

If `$ARGUMENTS` is provided, filter to matching components (case-insensitive partial match on component name or section name). Otherwise audit all components.

### 1.3 Ensure Storybook

Verify Storybook is running on `http://localhost:6006`. If not, start it with `preview_start`.

---

## Phase 2: Per-Component Audit

For each component, run a four-part audit:

### 2.1 Source Code Analysis (Static)

Read the component source file and verify:

**Semantic HTML:**
- Interactive elements use correct native elements (`<button>`, `<a>`, `<input>`, not `<div onClick>`)
- Form inputs have associated labels (via `htmlFor`/`id` or `aria-label`)
- Error messages linked via `aria-describedby`
- Required fields marked with `required` or `aria-required`
- Tables use proper `<table>`, `<th scope>`, `<caption>` structure

**ARIA Attributes:**
- Custom widgets have explicit ARIA roles
- Dynamic state communicated via `aria-expanded`, `aria-selected`, `aria-checked`, etc.
- Live regions for async updates (`aria-live`, `role="alert"`, `role="status"`)
- No redundant or abstract roles
- Radix-based components: verify primitive used correctly, consumer labels provided

**Keyboard:**
- All click handlers have keyboard equivalents (or use native `<button>`)
- Focus ring present via `focus-visible:ring-*` classes
- No positive `tabIndex` values
- Dialog/popover: focus trap and restoration logic present

**Motion:**
- Animations use `motion-safe:` prefix or check `prefers-reduced-motion`

### 2.2 Rendered Audit (Storybook + axe-core)

Navigate to the component's default story in Storybook and run axe-core programmatically:

```js
// In preview_eval:
const results = await new Promise((resolve) => {
  const el = document.querySelector('#storybook-root') || document.querySelector('.sb-show-main');
  if (window.axe) {
    window.axe.run(el).then(resolve);
  } else {
    resolve({ violations: [], passes: [], incomplete: [] });
  }
});
JSON.stringify({
  violations: results.violations.map(v => ({
    id: v.id,
    impact: v.impact,
    description: v.description,
    nodes: v.nodes.length,
    help: v.helpUrl
  })),
  passCount: results.passes.length,
  incompleteCount: results.incomplete.length
});
```

If `axe` is not available on `window`, inject it first:
```js
if (!window.axe) {
  const script = document.createElement('script');
  script.src = 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.11.1/axe.min.js';
  document.head.appendChild(script);
  await new Promise(r => script.onload = r);
}
```

### 2.3 Keyboard Navigation Test

For each component, test keyboard interactions:

1. **Tab to component** — verify focus ring appears
2. **Activate** — Enter/Space triggers the expected action
3. **Component-specific keys** — check against the keyboard interaction table in `.claude/rules/accessibility.md` section 3.2
4. **Tab away** — verify focus moves to next element, no focus trap (except modals)

For interactive states (dropdown open, dialog open):
```js
// Tab to element
preview_eval: document.querySelector('[data-testid="component"]')?.focus()
// Check focus visible
preview_inspect: selector for :focus-visible styles
// Trigger keyboard event
preview_eval: document.activeElement?.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', bubbles: true }))
```

### 2.4 Contrast Audit

For the component's default and key variant states:

1. Use `preview_inspect` to capture `color`, `background-color`, `border-color`
2. Calculate contrast ratios using the WCAG formula (see `.claude/rules/accessibility.md` section 4.4)
3. Verify:
   - Normal text: >= 4.5:1
   - Large text (>= 18px or >= 14px bold): >= 3.0:1
   - UI components against adjacent background: >= 3.0:1
4. Repeat for dark theme (reload story with `&globals=theme:dark`)

**Token-level check (if Figma connected):**
If the Figma Desktop Bridge is available, also cross-reference the Storybook computed colors against Figma variable bindings to ensure the Figma component uses the same tokens.

---

## Phase 3: Report

### 3.1 Per-Component Report

For each component, generate:

```markdown
## [Component Name]

### Positive Points
- [List detected good practices]

### Issues Detected

#### Critical (P0) — Blocking
- [ ] [File:Line] [Problem description]
      Solution: [How to fix]

#### Important (P1) — To fix
- [ ] [File:Line] [Problem description]
      Solution: [How to fix]

#### Minor (P2) — Improvement
- [ ] [File:Line] [Problem description]
      Solution: [How to fix]

### Scores
| Category | Score |
|----------|-------|
| Semantic HTML | X/5 |
| ARIA Attributes | X/5 |
| Keyboard / Focus | X/5 |
| Color / Contrast | X/5 |
| **Total** | **X/20** |

### Verdict: [COMPLIANT / MINOR ISSUES / TO FIX / NON-COMPLIANT]

### axe-core Results
- Violations: N (X critical, Y serious, Z moderate)
- Passes: N
- Incomplete: N
```

### 3.2 Summary Report

After auditing all components, write `.claude/ds-wcag-report.md`:

```markdown
# WCAG 2.1 AA Accessibility Audit Report
**Date:** [ISO date]
**Components audited:** N
**Filter:** [component filter or "all"]

## Summary

| Verdict | Count |
|---------|-------|
| COMPLIANT (18-20) | X |
| MINOR ISSUES (14-17) | Y |
| TO FIX (10-13) | Z |
| NON-COMPLIANT (0-9) | W |

## Component Scores

| Component | Semantic | ARIA | Keyboard | Contrast | Total | Verdict | P0 Issues |
|-----------|----------|------|----------|----------|-------|---------|-----------|
| Button | 5 | 5 | 5 | 5 | 20 | COMPLIANT | 0 |
| Input | 4 | 3 | 5 | 5 | 17 | MINOR | 0 |
| ... | | | | | | | |

## Critical Issues (P0)

| Component | Issue | File:Line | Impact |
|-----------|-------|-----------|--------|
| Select | Missing aria-expanded | select.tsx:45 | Screen readers |
| ... | | | |

## axe-core Violation Summary

| Rule ID | Impact | Components Affected | Description |
|---------|--------|-------------------|-------------|
| color-contrast | serious | Badge, Tabs | Insufficient contrast ratio |
| ... | | | |

## Recommendations

1. [Prioritised list of fixes]
2. ...
```

---

## Phase 4: Auto-Fix Suggestions

For P0 and P1 issues where the fix is clear and mechanical:

1. Present the proposed code change
2. Wait for user approval
3. Apply the fix using Edit tool
4. Re-run the axe-core check on the modified component to verify

**Auto-fixable patterns:**
- Missing `aria-label` on icon-only buttons → add `aria-label` prop
- Missing `aria-invalid` on error state → add `aria-invalid={!!error}`
- Missing `aria-describedby` for error messages → wire up `id` linkage
- Missing `aria-required` → add `aria-required={required}`
- Missing `role="alert"` on error containers → add the role
- Missing `sr-only` text for icon-only actions → add `<span className="sr-only">`

---

## Key Reference

- **Rules file:** `.claude/rules/accessibility.md`
- **Report output:** `.claude/ds-wcag-report.md`
- **Storybook a11y addon:** If configured, provides per-story axe-core panel
- **WCAG 2.1 Quick Reference:** https://www.w3.org/WAI/WCAG21/quickref/
- **ARIA Authoring Practices:** https://www.w3.org/WAI/ARIA/apg/

## Usage

```bash
# Audit all components
/ds-wcag

# Audit a specific component
/ds-wcag Button

# Audit a section
/ds-wcag "Forms"
```
