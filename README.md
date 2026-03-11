# Design System Skills for Claude Code

A collection of open-source [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code) for managing design systems across **Code**, **Storybook**, and **Figma**. These skills turn your design system into AI-readable infrastructure — enabling automated audits, bidirectional sync, accessibility checks, and prototyping.

## What's Included

### Skills (`.claude/commands/`)

| Skill | Command | Description |
|-------|---------|-------------|
| **[DS Sync](skills/ds-sync.md)** | `/ds-sync` | Sync Storybook components to Figma — renders each component, compares it to its Figma counterpart, and writes adjustments back using only bound variables |
| **[DS WCAG](skills/ds-wcag.md)** | `/ds-wcag` | WCAG 2.1 Level AA accessibility audit — source analysis, axe-core, keyboard testing, contrast checks in light + dark |
| **[DS Report](skills/ds-report.md)** | `/ds-report` | Cross-reference Code, Storybook, and Figma to produce a parity report with drift scores and historical benchmarks |
| **[DS Proto](skills/ds-proto.md)** | `/ds-proto` | Prototype layouts using your existing design system components with accessibility guardrails |
| **[DS Audit Figma](skills/ds-audit-figma.md)** | `/ds-audit-figma` | Lightweight Figma-to-Storybook visual parity spot-check |
| **[Storybook](skills/storybook.md)** | `/storybook` | Launch your Storybook dev server |

### Rules (`.claude/rules/`)

| File | Description |
|------|-------------|
| **[accessibility.md](rules/accessibility.md)** | Comprehensive WCAG 2.1 Level AA criteria — semantic HTML, ARIA, keyboard, contrast, touch targets, component checklists |

### Guides

| Guide | Description |
|-------|-------------|
| **[Getting Started](guides/getting-started.md)** | End-to-end setup: Figma MCP, Storybook, mapping file, and your first sync |
| **[Mapping File](guides/mapping-file.md)** | How to create and maintain the Storybook-to-Figma mapping JSON |

### Templates

| Template | Description |
|----------|-------------|
| **[ds-story-figma-map.json](templates/ds-story-figma-map.example.json)** | Example mapping file connecting Storybook stories to Figma node IDs |
| **[ds-benchmarks.json](templates/ds-benchmarks.example.json)** | Example benchmark file for tracking drift over time |
| **[launch.json](templates/launch.example.json)** | Example Claude Code server launch config for Storybook |
| **[settings.json](templates/settings.example.json)** | Example Claude Code settings with auto-formatting hooks |

---

## Quick Start

### 1. Install the skills

Copy the skills you want into your project's `.claude/commands/` directory:

```bash
# Clone this repo
git clone https://github.com/NewMediaStudio/design-system-skills.git

# Copy everything into your project
cp design-system-skills/skills/*.md your-project/.claude/commands/
cp design-system-skills/rules/*.md your-project/.claude/rules/
```

Or cherry-pick individual skills:

```bash
# Just the accessibility audit
cp design-system-skills/skills/ds-wcag.md your-project/.claude/commands/
cp design-system-skills/rules/accessibility.md your-project/.claude/rules/
```

### 2. Set up the prerequisites

See the **[Getting Started Guide](guides/getting-started.md)** for full setup instructions. The key pieces:

1. **Figma Console MCP** — install the [Figma Console MCP server](https://github.com/nicholasrq/figma-console) and the Desktop Bridge plugin
2. **Storybook** — running on `localhost:6006` (or configure your port)
3. **Mapping file** — create `.claude/ds-story-figma-map.json` linking your Storybook stories to Figma node IDs

### 3. Run a skill

```bash
# Audit accessibility on all components
/ds-wcag

# Audit a specific component
/ds-wcag Button

# Run a full parity report
/ds-report

# Sync Storybook to Figma
/ds-sync Button

# Prototype a new feature
/ds-proto "dashboard with metric cards and filters"
```

---

## Architecture

These skills assume a **three-pillar** design system:

```
Code (component library)  →  Storybook (rendered truth)  →  Figma (design mirror)
```

**Code is canonical.** Components are built in your codebase (React, Vue, Svelte — whatever your stack). Storybook documents their states and variants. Figma mirrors the components using bound variables (design tokens), not hardcoded values.

**Claude Code bridges all three.** Using MCP (Model Context Protocol) servers, Claude can:
- Read your component source code
- Render and inspect Storybook stories
- Read and write to Figma files via the Desktop Bridge
- Compare all three and report discrepancies

### The Mapping File

The key artifact is `.claude/ds-story-figma-map.json` — a machine-readable contract connecting every Storybook story to its corresponding Figma node. See the [Mapping File Guide](guides/mapping-file.md) for how to create one.

### Drift Scoring

The `/ds-report` skill tracks design system health over time:

```
Component Drift % = (1 - fullParityCount / totalUniqueComponents) × 100
Token Drift %     = (|codeTokens - figmaTokens| / max(codeTokens, figmaTokens)) × 100
Icon Drift %      = (|codeIcons - figmaIcons| / max(codeIcons, figmaIcons)) × 100
Overall Drift %   = 60% × Component + 25% × Token + 15% × Icon
```

Target: **0% drift.** Every run is benchmarked and stored for trend analysis.

---

## Customisation

These skills are designed to be adapted to your project. Common customisations:

### File Paths

Update these paths in the skills to match your project structure:

| Default | What It Is | Where to Change |
|---------|-----------|-----------------|
| `packages/ds/src/main.tsx` | DS barrel export | `ds-report.md`, `ds-proto.md` |
| `packages/ds/stories/` | Story file location | `ds-report.md`, `ds-wcag.md` |
| `packages/ds/src/styles/` | Token/CSS files | `ds-sync.md`, `ds-proto.md` |
| `apps/*/src/` | Application code | `ds-sync.md`, `ds-proto.md` |
| `design-system-manifest.json` | Component inventory | All skills |
| `.claude/ds-story-figma-map.json` | Figma mapping | All skills |

### Token Architecture

The skills assume CSS custom properties as design tokens:

```css
:root {
  --primary-foreground: #1a1a1a;
  --surface-background: #ffffff;
}
.dark {
  --primary-foreground: #f5f5f5;
  --surface-background: #1a1a1a;
}
```

If you use a different token system (Style Dictionary JSON, Tailwind config, etc.), update the token-loading sections in `ds-sync.md` and `ds-proto.md`.

### Storybook Port

Default is `localhost:6006`. If your Storybook runs on a different port, update `launch.json` and the `storybookBase` URL in your mapping file.

---

## Prerequisites

| Tool | Required For | Installation |
|------|-------------|-------------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | All skills | `npm install -g @anthropic-ai/claude-code` |
| [Figma Console MCP](https://github.com/nicholasrq/figma-console) | Figma sync, audit, report | See [Getting Started](guides/getting-started.md) |
| [Figma Desktop](https://www.figma.com/downloads/) | Figma skills | Download from Figma |
| [Storybook](https://storybook.js.org/) | All skills | Part of your project |

---

## Contributing

Contributions welcome. If you've built skills for your design system workflow, open a PR.

### Adding a Skill

1. Create a `.md` file in `skills/`
2. Use the frontmatter format: `---\ndescription: Short description\n---`
3. Keep skills generic — no product names, proprietary tokens, or internal URLs
4. Reference config files by their `.claude/` paths
5. Update this README

---

## License

MIT

---

## Author

**Valentine Makhouleen** — [New Media Studio](https://github.com/NewMediaStudio)

## Credits

Built on the work of:
- [Figma Console MCP](https://github.com/nicholasrq/figma-console) by TJ Pitre / Southleft
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) by Anthropic
