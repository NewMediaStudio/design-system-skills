---
description: Sync Storybook components to Figma via visual benchmarking, token mapping, and write-back
---

# Design System Sync

Synchronise your live Storybook component library with a Figma Design System file. The workflow renders each component, compares it to its Figma counterpart, extracts token/prop context, and writes adjustments back to Figma using only bound variables — never raw hex values.

## Prerequisites

- Figma Desktop Bridge plugin must be running
- Your Figma design system file must be open
- Storybook dev server running on `http://localhost:6006` (launch with `/storybook` if not running)
- `design-system-manifest.json` in project root (component inventory)
- `.claude/ds-story-figma-map.json` — pre-built Storybook↔Figma ID mapping (component IDs, story IDs, verify screen IDs, section frame IDs, key variable IDs)
- `.claude/rules/accessibility.md` — WCAG 2.1 Level AA criteria (auto-loaded by Phase 4.5)
- *(Optional)* [Figma Code Connect](https://github.com/figma/code-connect) — if published, provides real prop mappings for more accurate variant-level comparison

## Arguments

- `$ARGUMENTS` — optional component name filter and/or mode flag. Examples:
  - `Button` — standard sync for Button only
  - `--precision Button` — precision 1:1 per-variant audit for Button
  - `--precision` — precision audit for ALL components (slow)
  - `"Data Display"` — standard sync for a section
  - (empty) — standard sync for ALL components

---

## Phase 1: Setup & Token Map

### 1.0 Load the DS Registry (Fast Path)

If `.claude/ds-registry.json` exists, load it as the primary data source (single file read). The registry contains component metadata, variants, props, tokens, story mappings, and Figma node IDs — everything needed for Phases 1–3. Skip reading the barrel export, individual component source files, the manifest, and the token map separately.

If the registry does not exist, fall back to the individual file reads described below.

### 1.1 Load the Mapping File & Manifest

1. **Read `.claude/ds-story-figma-map.json`** first. This is the primary lookup for:
   - Section frame IDs → `sections[sectionName].sectionFrameId`
   - Verify screen IDs → `sections[sectionName].verifyLight` / `.verifyDark`
   - Component Figma node IDs → `sections[sectionName].components[name].figmaId`
   - Storybook story IDs → `sections[sectionName].components[name].stories[]`
   - Key Figma variable IDs → `keyVariableIds`
   - Storybook URL template → `_meta.storybookBase` (replace `{storyId}` and `{theme}`)
   - Semantic collection/mode IDs → `_meta.semanticCollection`, `_meta.lightMode`, `_meta.darkMode`

2. **Read `design-system-manifest.json`** for argTypes, prop definitions, and variant metadata not in the mapping file.

### 1.1.1 Mapping File Staleness Check

Before syncing, validate the mapping file is current. This prevents silent failures where write-backs target the wrong or deleted Figma nodes.

**Step 1 — Resolve all component node IDs in Figma:**

```js
// figma_execute
const page = figma.currentPage;
const liveIds = new Set();
function walk(node) {
  if (node.type === 'COMPONENT' || node.type === 'COMPONENT_SET' || node.type === 'FRAME') {
    liveIds.add(node.id);
  }
  if ('children' in node) node.children.forEach(walk);
}
walk(page);
return [...liveIds];
```

**Step 2 — Cross-reference against the mapping:**

For each `sections[section].components[name].figmaId` in the mapping file:
- If the ID is **not** in `liveIds` → flag as **STALE_ID**
- If the component is in the manifest but has no mapping entry → flag as **MISSING_MAPPING**
- If the mapping has an entry with no corresponding manifest component → flag as **ORPHAN_ENTRY**

**Step 3 — Fetch Storybook index and cross-reference story IDs:**

```bash
curl -s http://localhost:6006/index.json
```

For each `stories[]` entry in the mapping, verify the story ID exists in the Storybook index. Flag **MISSING_STORY** for any that don't.

**Step 4 — Report and decide:**

```markdown
### Mapping File Health
| Status | Count | Details |
|--------|-------|---------|
| Valid entries | X | — |
| Stale Figma IDs | X | Button (ID:123), Input (ID:456) |
| Missing mappings | X | DatePicker, Tooltip |
| Orphan entries | X | LegacyAlert (removed from manifest) |
| Missing Storybook stories | X | badge--with-icon |
```

- If stale/missing/orphan entries are **0** → proceed with sync.
- If any stale IDs exist → **pause and report.** Do NOT write to stale node IDs. Offer to attempt auto-recovery (re-scan Figma by name and update the IDs) before continuing.
- If > 25% of entries are invalid → **abort sync** and instruct the user to regenerate the mapping file using the [Mapping File Guide](../guides/mapping-file.md).

**Auto-recovery for stale IDs:** For each stale component, search Figma by name:

```js
// figma_execute
const page = figma.currentPage;
const matches = [];
function walk(node) {
  if ((node.type === 'COMPONENT' || node.type === 'COMPONENT_SET') && node.name === 'COMPONENT_NAME') {
    matches.push({ id: node.id, name: node.name, parent: node.parent?.name });
  }
  if ('children' in node) node.children.forEach(walk);
}
walk(page);
return matches;
```

If exactly one match is found, update the mapping entry and continue. If zero or multiple matches are found, require manual resolution.

**If the mapping file is absent**, regenerate it by fetching Storybook's `/index.json` and traversing the Figma page children. Save the updated mapping back to `.claude/ds-story-figma-map.json`.

### 1.2 Build the Token Translation Map

Build a JSON mapping that links CSS custom properties (Storybook) to Figma Variable IDs.

**Sources:**
- Your CSS token files (e.g., `src/styles/tokens.css`, `src/styles/colors.css`) — primitive and semantic color tokens
- Your app-level styles (e.g., `src/styles/index.css`) — semantic + Tailwind theme tokens
- Figma variables via `figma_get_variables` with `resolveAliases: true`

**Build the map:**

```json
{
  "tokenMap": {
    "--primary-foreground": { "cssValueLight": "...", "cssValueDark": "...", "figmaVarId": "VariableID:...", "figmaVarName": "primary-foreground" },
    "--surface-background": { "cssValueLight": "...", "cssValueDark": "...", "figmaVarId": "VariableID:...", "figmaVarName": "surface-background" }
  }
}
```

For each CSS variable, find the Figma variable whose resolved RGB values match. Log any unmatched tokens as warnings.

### 1.3 Filter Components

If `$ARGUMENTS` is provided, filter the mapping's `sections` to only include matching components or sections (case-insensitive partial match on component name or section name). Otherwise process all sections and components. Components listed in `map.storybookOnly` and `map.figmaOnly` are skipped from the main sync loop but logged in the report.

---

## Phase 1.5: Duplicate & Shadow Component Detection

Before any visual or token audit, scan for **duplicate components** — cases where the app re-implements a DS component locally instead of importing from the design system package. These shadow copies silently drift from the design system and bypass all DS-level changes.

### 1.5.1 App-Level Duplicate Scan

For each component in the DS manifest:

1. **Search for local copies:** Use `grep` to find files in your app source that export a function or component with the same name as a DS export.
2. **Compare implementations:** If a local file re-implements a DS component, flag it as a duplicate.
3. **Check import paths:** Verify that consuming files import from your DS package, not from local paths.

**Output:**
```
Duplicate Component Report
━━━━━━━━━━━━━━━━━━━━━━━━━
| App File                    | DS Component        | Status    | Action                    |
|-----------------------------|---------------------|-----------|---------------------------|
| shared/components/button.tsx | Button             | DUPLICATE | Delete, use DS import     |
| shared/components/table.tsx | Table               | SHADOW    | Review — may have overrides |
```

---

## Phase 2: Audit & Visual Benchmarking

### 2.0 Bulk Programmatic Audit

Before visual comparison, run a programmatic scan across ALL section frames to find every issue at once. Use `figma_execute` to traverse all component nodes and report:

1. **Hardcoded fills** — Any fill paint without a `boundVariables.color` binding
2. **Hardcoded text colors** — Same check on text node fills
3. **Hardcoded strokes** — Stroke paints without variable bindings
4. **Missing text styles** — Text nodes without `textStyleId`
5. **Wrong font family** — Text nodes using unexpected fonts
6. **Icon instances** — Verify icons are component instances (not raw vectors/groups)
7. **Missing/broken component references** — Instances with null `mainComponent`

Output a categorised table per section.

### 2.0.1 Variable Correctness Audit (Wrong Bindings)

Beyond checking for missing bindings, verify that bound variables are the **correct** ones by cross-referencing against the code's CSS tokens:

1. Extract the bound variable name from each Figma node
2. Extract the expected CSS token from the component source code for that state/variant
3. If the bound variable ≠ the expected token, flag as `WRONG_BINDING`

### 2.0.2 Component Dimension Comparison

Compare Figma component dimensions against code sizing classes:

1. Parse the component source for sizing classes (e.g., `w-8` = 32px, `h-4` = 16px)
2. Read Figma node `width` and `height`
3. Flag any dimension mismatch beyond ±2px tolerance

### 2.0.3 Structural Completeness Audit

Compare the Figma node tree against the code's render tree:

1. Extract all rendered child elements from JSX
2. Walk the Figma component's children recursively
3. Match code elements to Figma nodes by name/type/position
4. Flag missing children (e.g., counter badge, icon slot, secondary label)

### 2.0.4 Contrast & Readability Audit

Catch cases where Figma nodes have valid variable bindings but the resulting color combination produces insufficient contrast:

1. Build a token→resolved-color map for Light and Dark modes
2. For every component variant, identify foreground/background pairs
3. Calculate contrast ratios using the WCAG 2.0 formula
4. Flag violations: text < 4.5:1, large text < 3.0:1, UI components < 3.0:1
5. Check both modes

### 2.1 Verify Screens (Visual Benchmarking)

If your Figma file contains **Verify screens** (frames that mirror Storybook layouts), use them for pixel-level comparison:

1. Look up verify screen IDs from the mapping
2. Capture Storybook screenshot via `preview_screenshot`
3. Capture Figma verify frame via `figma_capture_screenshot`
4. Compare for color accuracy, spacing, typography, border radius, icon presence

### 2.2 Per-Component Visual Comparison

For each component:

1. Render the default Storybook story (light and dark)
2. Screenshot the corresponding Figma component
3. Categorise as: **MATCH** (< 1% delta), **DRIFT** (auto-correctable), or **MISMATCH** (needs manual intervention)

---

## Phase 3: Token Extraction & Comparison

### 3.1 Extract Storybook Tokens

For each component in DRIFT status:

1. Navigate to the Storybook story
2. Use `preview_inspect` to capture computed CSS properties: `color`, `background-color`, `border-color`, `font-size`, `font-weight`, `padding`, `gap`, `border-radius`
3. Map computed values back to CSS custom property names using the token translation map

### 3.2 Extract Figma Tokens

For the matching Figma component:

1. Use `figma_execute` to read fills, strokes, text styles, and their variable bindings
2. For each property, record: the bound variable ID, the resolved value, and the variable name

### 3.3 Diff

Compare Storybook tokens against Figma bindings. For each discrepancy, record:

```
| Property | Storybook Token | Figma Variable | Status |
|----------|----------------|----------------|--------|
| fill     | --button       | --button       | MATCH  |
| text     | --primary-fg   | --secondary-fg | DRIFT  |
| border   | --stroke-input | (hardcoded)    | MISSING|
```

---

## Phase 4: Write-Back to Figma

### 4.1 Binding Corrections

For each DRIFT or MISSING token:

1. Look up the correct Figma Variable ID from the token translation map
2. Use `figma_execute` to set the variable binding:
   ```js
   const node = await figma.getNodeByIdAsync('NODE_ID');
   const variable = await figma.variables.getVariableByIdAsync('VAR_ID');
   node.setBoundVariable('fills', 0, variable);
   ```
3. **NEVER set raw hex values.** Always bind to variables.

### 4.2 Spacing & Dimension Fixes

If spacing values differ:

1. Read the expected values from Storybook
2. Set padding, gap, and corner radius on the Figma node:
   ```js
   node.paddingTop = 8;
   node.paddingBottom = 8;
   node.itemSpacing = 12;
   node.cornerRadius = 8;
   ```

### 4.3 Review & Confirm

After all write-backs:

1. Re-capture the Figma component screenshot
2. Compare against Storybook to verify the fix
3. Report the result

---

## Phase 5: Report

### 5.1 Per-Component Summary

```markdown
| Component | Light | Dark | Issues Fixed | Remaining |
|-----------|-------|------|-------------|-----------|
| Button    | MATCH | MATCH| 0           | 0         |
| Input     | DRIFT | DRIFT| 3 bindings  | 1 manual  |
| Badge     | MISMATCH | — | —           | Needs rebuild |
```

### 5.2 Token Coverage

```markdown
| Metric              | Count | Status |
|---------------------|-------|--------|
| Total fills audited | 240   | —      |
| Variable-bound      | 228   | 95%    |
| Fixed this run      | 8     | +3.3%  |
| Still hardcoded     | 4     | Review |
```

### 5.3 Action Items

List remaining issues that need manual intervention, grouped by severity.

---

## Phase 6: Precision Mode (`--precision`)

When the `--precision` flag is used, run a **1:1 per-variant audit** that goes deeper than the standard sync:

### 6.1 Variant Enumeration

For each `COMPONENT_SET`, enumerate every Figma variant:
```js
const set = await figma.getNodeByIdAsync('COMPONENT_SET_ID');
return set.children.map(c => ({ name: c.name, id: c.id }));
```

Match each Figma variant to a corresponding Storybook state by parsing the variant name (e.g., `Size=Small, State=Default, Intent=Primary`).

### 6.2 Per-Variant Token Comparison

For each variant:

1. Read the Storybook source to find the CSS classes applied for that specific variant/state combination
2. Map classes to CSS tokens (e.g., `bg-surface-background` → `--surface-background`)
3. Read the Figma variant's fills/strokes/text bindings
4. Compare every binding

**If Code Connect is published:** Also verify that Figma's variant property names and values match the Code Connect prop mappings. For example, if Code Connect maps `figma.enum("Type", { Primary: "primary" })`, verify that the Figma component set actually has a variant property named "Type" with a value "Primary", and that it corresponds to the `variant="primary"` prop in code. Flag any mismatches as `PROP_MAPPING_DRIFT`.

### 6.3 Per-Variant Visual Comparison

For each variant:

1. Render the Storybook story with the matching args
2. Screenshot the Figma variant node
3. Side-by-side visual comparison

This is slow but catches every discrepancy at the variant level.

---

## Key Rules

1. **Variables only** — Every color write to Figma MUST use variable bindings, never raw hex
2. **Token map drives decisions** — The CSS↔Figma token map is the source of truth for what "correct" looks like
3. **Code is canonical** — When Storybook and Figma disagree, Storybook (code) wins
4. **Log everything** — Every change is recorded for the report
5. **No destructive changes** — Components are updated, never deleted

## Usage

```bash
# Sync all components
/ds-sync

# Sync a specific component
/ds-sync Button

# Sync a section
/ds-sync "Data Display"

# Precision 1:1 variant audit
/ds-sync --precision Button

# Precision audit for all components (slow)
/ds-sync --precision
```
