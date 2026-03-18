---
description: Generate structured component specs — anatomy, API, tokens, structure, and accessibility
---

# Design System Spec Generator

Generate comprehensive, structured specification sheets for design system components. Inspired by Uber's uSpec approach — each spec covers anatomy, API surface, token usage, structural measurements, and accessibility semantics in a single pass.

## Prerequisites

- Storybook dev server running on `http://localhost:6006` (launch with `/storybook` if not running)
- `.claude/ds-registry.json` — unified component registry (optional, enables fast path)
- `design-system-manifest.json` in project root (component inventory, fallback)
- `.claude/ds-story-figma-map.json` — Storybook ↔ Figma ID mapping (fallback)
- `.claude/ds-token-map.json` — CSS property ↔ Figma variable mapping
- `.claude/rules/accessibility.md` — WCAG 2.1 AA criteria checklist
- **For `--figma` flag:** Figma Console MCP connected with Desktop Bridge running

## Arguments

- `$ARGUMENTS` — component name filter and/or flags. Examples:
  - `Button` — generate spec for Button only
  - `"Forms"` — generate specs for all form components
  - `--render` — output specs as Storybook MDX docs (default: markdown report)
  - `--figma` — also write spec annotations into the Figma file (requires Figma Console MCP)
  - `--a11y-only` — generate only the accessibility spec section
  - (empty) — generate specs for ALL components

---

## Phase 0: Setup

### 0.1 Registry Fast-Path

1. **Check if `.claude/ds-registry.json` exists.**
2. **If YES:** load it as the primary inventory — component names, source files, sections, variants, props, tokens, Figma node IDs, story paths. Skip redundant file reads.
3. **If NO:** fall back to reading `design-system-manifest.json`, your barrel export file, `.claude/ds-story-figma-map.json`, and `.claude/ds-token-map.json` individually.

### 0.2 Load Supporting Data

1. **Read `.claude/ds-token-map.json`** — CSS property ↔ Figma variable ID mapping with hex values
2. **Read `.claude/rules/accessibility.md`** — WCAG 2.1 AA checklist for the accessibility spec section
3. **Read your token/CSS source files** — semantic token definitions for both themes
4. **If `--figma` flag:** verify Figma Console MCP is connected by listing available tools. If not connected, warn and fall back to markdown-only output.

### 0.3 Filter Components

If `$ARGUMENTS` specifies a component or section name, filter the inventory (case-insensitive partial match). Otherwise process all components.

### 0.4 Ensure Storybook

Verify Storybook is running on `http://localhost:6006`. If not, start it with `/storybook`.

---

## Phase 1: Component Analysis

For each component, perform a deep read of the source code and rendered output.

### 1.1 Source Code Read

Read the component's source file (from registry `sourceFile` or manifest). Extract:

- **Element structure** — the JSX tree: which HTML elements and sub-components are rendered
- **Props interface** — all typed props with defaults, including `React.ComponentPropsWithoutRef` extensions
- **Variants** — CVA variant definitions (names, values, defaults), compound variants
- **Forwarded ref** — whether `forwardRef` is used and what element type it wraps
- **Slots** — `children`, `asChild`, render props, named slots
- **Radix primitives** — which Radix UI primitives are composed (if any)
- **Hooks** — custom hooks used internally
- **Token usage** — Tailwind classes referencing design tokens (e.g., `bg-surface-background`, `text-primary-foreground`)

### 1.2 Story Read

Read the component's story file. Extract:

- **Story variants** — each named export and its args/render function
- **Decorators** — layout wrappers, theme providers
- **ArgTypes** — control definitions with options, default values

### 1.3 Rendered Inspection

Navigate to the component's default story in Storybook. Capture:

- **Computed dimensions** — height, width, padding, margin, gap, border-radius for the root element and key children
- **Computed colours** — foreground, background, border colours in both light and dark themes
- **Typography** — font-family, font-size, font-weight, line-height, letter-spacing for all text nodes
- **Interactive states** — hover, focus, active, disabled appearance changes (via story variants or direct inspection)

Use DOM inspection to extract computed styles:

```js
const root = document.querySelector('#storybook-root > *');
const styles = window.getComputedStyle(root);
JSON.stringify({
  height: styles.height,
  padding: styles.padding,
  gap: styles.gap,
  borderRadius: styles.borderRadius,
  backgroundColor: styles.backgroundColor,
  color: styles.color,
  fontFamily: styles.fontFamily,
  fontSize: styles.fontSize,
  fontWeight: styles.fontWeight,
  lineHeight: styles.lineHeight
});
```

Repeat in dark theme (reload with `&globals=theme:dark`).

---

## Phase 2: Spec Generation

For each component, generate six spec sections. All sections are generated in a single pass per component — do not re-read the source between sections.

### 2.1 Anatomy

Map the component's internal structure with numbered markers:

```markdown
### Anatomy

| # | Element | HTML | Role | Description |
|---|---------|------|------|-------------|
| 1 | Root | `<button>` | — | Clickable container |
| 2 | Icon slot | `<span>` | `aria-hidden="true"` | Optional leading/trailing icon |
| 3 | Label | text node | — | Button text content |
| 4 | Loading spinner | `<LoadingSpinner>` | `aria-hidden="true"` | Shown when `loading={true}` |
```

Rules:
- Number elements in DOM order (depth-first)
- Include the actual HTML element rendered (not the React component name)
- Note ARIA roles assigned to each element
- Flag conditional elements (shown/hidden based on props)
- For compound components (Dialog, Tabs), show the full assembled structure

### 2.2 API Surface

Document every prop the component accepts:

```markdown
### API

| Prop | Type | Default | Required | Description |
|------|------|---------|----------|-------------|
| `intent` | `"primary" \| "secondary" \| "positive" \| "negative"` | `"primary"` | No | Visual style variant |
| `size` | `"small" \| "medium" \| "large"` | `"medium"` | No | Size variant |
| `icon` | `ReactNode` | — | No | Icon element |
| `loading` | `boolean` | `false` | No | Shows loading spinner, disables interaction |
| `disabled` | `boolean` | `false` | No | Disables the button |
| `asChild` | `boolean` | `false` | No | Merges props onto child via Radix Slot |
```

Rules:
- Include inherited HTML attributes — list only notable ones, add a note about the spread
- Show actual TypeScript union types, not simplified descriptions
- Mark required props explicitly
- Document compound component sub-parts separately (e.g., `Dialog.Trigger`, `Dialog.Content`)

### 2.3 Token Mapping

Map every design token used by the component to its CSS property and Figma variable:

```markdown
### Tokens

| Element | Property | Token (CSS) | Light Value | Dark Value | Figma Variable |
|---------|----------|-------------|-------------|------------|----------------|
| Root | background | `--interactive-primary` | `#1B1B1B` | `#FFFFFF` | `interactive-primary` |
| Root | color | `--interactive-primary-foreground` | `#FFFFFF` | `#1B1B1B` | `interactive-primary-foreground` |
| Root | border-radius | `rounded-md` (6px) | — | — | — |
| Root:hover | background | `--interactive-primary-hover` | `#353535` | `#EAEAEA` | `interactive-primary-hover` |
| Root:focus | ring | `--interactive-focus-ring` | `#0080FF` | `#78B6ED` | `interactive-focus-ring` |
```

Rules:
- Cross-reference each Tailwind class against your CSS source to find the CSS custom property
- Use `.claude/ds-token-map.json` to look up the corresponding Figma Variable ID and resolved hex value
- Include state-specific tokens (hover, focus, active, disabled)
- Flag any hardcoded values (not using tokens) as warnings
- Verify light/dark values match between CSS and Figma — flag mismatches

### 2.4 Structure & Measurements

Document the component's dimensions and spacing:

```markdown
### Structure

| Variant | Height | Padding (h) | Padding (v) | Gap | Border | Border Radius |
|---------|--------|-------------|-------------|-----|--------|---------------|
| small | 32px | 12px | 6px | 6px | none | 6px |
| medium | 40px | 16px | 8px | 8px | none | 6px |
| large | 48px | 24px | 12px | 8px | none | 6px |
```

Rules:
- Use computed values from Phase 1.3, not Tailwind class approximations
- Document all size variants
- Include min-width, max-width if constrained
- Note touch target compliance (>= 24px min, 44px recommended)
- Flag spacing values that don't align to the 8px grid

### 2.5 Accessibility Spec

Generate a complete accessibility specification. This covers web (ARIA/HTML) semantics and is structured to inform implementation across platforms.

```markdown
### Accessibility

#### Roles & Semantics
| Element | Role | Semantic Element | Notes |
|---------|------|-----------------|-------|
| Root | `button` | `<button>` | Native semantics, no explicit role needed |

#### States & Properties
| State | ARIA Attribute | Values | Trigger |
|-------|---------------|--------|---------|
| Disabled | `aria-disabled` / `disabled` | `true` / `false` | `disabled` prop |
| Loading | `aria-busy` | `true` / `false` | `loading` prop |
| Pressed (toggle) | `aria-pressed` | `true` / `false` | Toggle variant click |

#### Keyboard Interactions
| Key | Action | Condition |
|-----|--------|-----------|
| `Enter` | Activate button | Always |
| `Space` | Activate button | Always |
| `Tab` | Move focus to next element | Always |

#### Focus Management
- Focus ring: describe the focus ring implementation (e.g., `focus-visible:ring-2`)
- Tab order: natural DOM order / managed
- Focus trap: Yes/No (for containers like dialogs)
- Focus restoration: describe if focus returns to trigger on close

#### Screen Reader Announcements
| Scenario | Announcement | Implementation |
|----------|-------------|----------------|
| Default | "[Label text], button" | Native `<button>` semantics |
| Icon-only | "[aria-label value], button" | Requires `aria-label` prop |
| Loading | "[Label text], button, busy" | `aria-busy="true"` |
| Disabled | "[Label text], button, dimmed" | `disabled` attribute |

#### Contrast Verification
| Element | Foreground | Background | Ratio | Passes | Theme |
|---------|-----------|------------|-------|--------|-------|
| Label (primary) | `#FFFFFF` | `#1B1B1B` | 15.4:1 | AA pass | Light |
| Label (primary) | `#1B1B1B` | `#FFFFFF` | 15.4:1 | AA pass | Dark |
```

Rules:
- Reference the component-specific checklist from `.claude/rules/accessibility.md` section 7 (if available)
- Calculate actual contrast ratios using the WCAG 2.0 relative luminance formula
- Check in both light and dark themes
- For Radix-based components, note which ARIA attributes are handled automatically by the primitive
- For compound components, document the ARIA relationship between parts (e.g., `aria-controls`, `aria-labelledby`)
- Flag any missing attributes as `MISSING — [fix instruction]`

### 2.6 Usage Examples

Provide 2–3 code examples showing common usage patterns:

```markdown
### Usage

#### Basic
\`\`\`tsx
<Button intent="primary" onClick={handleSave}>
  Save changes
</Button>
\`\`\`

#### With icon
\`\`\`tsx
<Button intent="secondary" icon={<RiAddLine />}>
  Add item
</Button>
\`\`\`
```

Rules:
- Use realistic domain data appropriate to your product
- Show the accessibility-correct pattern (e.g., `aria-label` on icon-only buttons)
- Include at least one example demonstrating proper accessible usage

---

## Phase 3: Output

### 3.1 Markdown Report (default)

Write all specs to `.claude/ds-specs/[component-name].md`:

```markdown
# [Component Name] — Design Spec

**Generated:** [ISO date]
**Source:** [source file path]
**Package:** [package name]
**Section:** [section name]
**Storybook:** [story URL]

---

[Anatomy section]

---

[API section]

---

[Token mapping section]

---

[Structure section]

---

[Accessibility section]

---

[Usage examples]
```

Write an index file at `.claude/ds-specs/index.md`:

```markdown
# Design System Specifications

**Generated:** [ISO date]
**Components:** [count]

| Component | Section | Source | Spec |
|-----------|---------|-------|------|
| Button | Actions | `src/components/button/button.tsx` | [View](button.md) |
| Badge | Data Display | `src/components/badge/badge.tsx` | [View](badge.md) |
```

### 3.2 Storybook MDX Docs (`--render`)

When `--render` is passed, additionally generate MDX doc files alongside the story files:

```mdx
import { Meta } from "@storybook/blocks";

<Meta title="Specs/[Section]/[ComponentName]" />

# [Component Name]

[Spec content converted to MDX-compatible markdown]
```

Place the MDX files next to the corresponding story files.

### 3.3 Figma Annotations (`--figma`)

When `--figma` is passed and Figma Console MCP is connected:

1. Navigate to the component's Figma node (from registry/mapping)
2. Create a spec frame adjacent to the component using `figma_execute`
3. Populate the frame with:
   - Anatomy table with numbered markers
   - Token table with variable references
   - Structure measurements
   - Accessibility notes
4. Use Figma text styles matching the DS typography

**Figma spec frame structure:**

```
SPEC: [ComponentName]
+-- Anatomy (section with table)
+-- API Summary (key props only)
+-- Token Map (color swatches + variable names)
+-- Structure (dimension callouts)
+-- Accessibility (ARIA roles, keyboard shortcuts, screen reader text)
```

Use `figma_execute` to create frames, text nodes, and rectangles. Bind colours to Figma variables where applicable. This is the write-back capability — spec documentation lives directly in the Figma file alongside the component.

---

## Phase 4: Summary

Present the user with:

```markdown
## Spec Generation Complete

**Components:** [N] specs generated
**Output:** `.claude/ds-specs/` ([N] files)
**Storybook docs:** [Yes/No] ([N] MDX files)
**Figma annotations:** [Yes/No] ([N] spec frames)

### Highlights

| Component | Tokens | Hardcoded Values | A11y Issues | Contrast |
|-----------|--------|-----------------|-------------|----------|
| Button | 12 | 0 | 0 | All pass |
| Badge | 8 | 1 warning | 0 | All pass |
| Input | 10 | 0 | 1 (missing aria-describedby) | Light fail: placeholder 3.8:1 |

### Token Warnings
[List any hardcoded values found]

### Accessibility Gaps
[List any P0/P1 issues found during spec generation]
```

---

## Key Rules

1. **Single pass** — read the source once, generate all six sections from that read. Do not re-read between sections.
2. **Token verification** — every colour must trace back to a CSS custom property and (if mapped) a Figma variable. Flag hardcoded values.
3. **Contrast calculation** — use the WCAG 2.0 relative luminance formula. Check both themes.
4. **Registry-first** — always prefer the registry over individual file reads.
5. **Incremental** — if a spec already exists at `.claude/ds-specs/[name].md`, update it rather than overwriting (preserve any manual annotations).

## Usage

```bash
# Generate spec for a single component
/ds-spec Button

# Generate specs for all form components
/ds-spec "Forms"

# Generate all specs
/ds-spec

# Generate specs with Storybook doc pages
/ds-spec --render Button

# Generate specs with Figma annotations (requires Figma Console MCP)
/ds-spec --figma Button

# Only the accessibility section
/ds-spec --a11y-only Dialog

# All flags
/ds-spec --render --figma
```
