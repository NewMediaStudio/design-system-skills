# The Mapping File

The mapping file (`.claude/ds-story-figma-map.json`) is the single most important artifact in this workflow. It's a machine-readable contract that connects every Storybook story to its corresponding Figma node.

Without it, Claude has to guess which Figma component matches which story. With it, every comparison is precise and every sync writes to the right place.

---

## Structure

```json
{
  "_meta": {
    "description": "Storybook story IDs to Figma node IDs mapping for ds-sync",
    "generated": "2026-03-01",
    "figmaFile": "YOUR_FIGMA_FILE_KEY",
    "storybookBase": "http://localhost:6006/iframe.html?id={storyId}&viewMode=story&globals=theme:{theme}",
    "semanticCollection": "VariableCollectionId:XXXX:YYYY",
    "lightMode": "XXXX:0",
    "darkMode": "XXXX:1"
  },
  "sections": {
    "SectionName": {
      "sectionFrameId": "FIGMA_FRAME_ID",
      "verifyLight": "FIGMA_VERIFY_FRAME_ID_LIGHT",
      "verifyDark": "FIGMA_VERIFY_FRAME_ID_DARK",
      "components": {
        "ComponentName": {
          "figmaId": "FIGMA_COMPONENT_NODE_ID",
          "figmaType": "COMPONENT_SET",
          "variantCount": 12,
          "stories": [
            "section-component--variant-name"
          ],
          "sourceFile": "src/components/component-name/component-name.tsx"
        }
      }
    }
  },
  "storybookOnly": {
    "ComponentName": {
      "reason": "Behaviour-only component, no visual Figma counterpart"
    }
  },
  "figmaOnly": {
    "ComponentName": {
      "figmaId": "NODE_ID",
      "reason": "Figma-only documentation component"
    }
  }
}
```

### `_meta` Section

| Field | Description |
|-------|-------------|
| `figmaFile` | The Figma file key from the URL (`figma.com/design/<KEY>/...`) |
| `storybookBase` | URL template for loading stories. `{storyId}` and `{theme}` are placeholders |
| `semanticCollection` | Figma Variable Collection ID for your semantic tokens |
| `lightMode` | Mode ID for the light theme in that collection |
| `darkMode` | Mode ID for the dark theme |

### `sections` Section

Group components by their Figma section/frame. Each section has:

| Field | Description |
|-------|-------------|
| `sectionFrameId` | The top-level frame in Figma containing this section's components |
| `verifyLight` / `verifyDark` | Optional "verify screen" frames that show all components arranged for visual comparison |
| `components` | Map of component name → details |

### Per-Component Fields

| Field | Description |
|-------|-------------|
| `figmaId` | The Figma node ID of the component or component set |
| `figmaType` | `COMPONENT_SET` (has variants), `COMPONENT` (single), or `FRAME` (composite) |
| `variantCount` | Number of Figma variants (children of a `COMPONENT_SET`) |
| `stories` | Array of Storybook story IDs (the `id` field from Storybook's index) |
| `sourceFile` | Relative path to the component source file |

---

## Finding IDs

### Figma Node IDs

1. **Right-click** any element in Figma
2. Select **Copy/Paste as → Copy link**
3. The URL contains `node-id=X-Y` — replace the `-` with `:` to get `X:Y`
4. Example: `node-id=5321-90928` → `5321:90928`

### Figma Variable Collection IDs

Use Claude with the Figma Console MCP:

```
> Use figma_get_variables with format: 'summary' to list all variable collections and their IDs
```

### Storybook Story IDs

1. Open Storybook and navigate to a story
2. The URL contains `?path=/story/<STORY_ID>` — that's the ID
3. Or fetch `http://localhost:6006/index.json` for all story IDs at once

### Auto-Discovery

You can ask Claude to help build the mapping:

```
> My Storybook is running on localhost:6006 and my Figma file is open.
> Help me build a ds-story-figma-map.json by:
> 1. Fetching the Storybook index to get all story IDs
> 2. Enumerating the Figma page structure to get component node IDs
> 3. Matching them by name similarity
> 4. Writing the mapping file
```

---

## Maintenance

### When to Update

- **New component added** — add its Figma ID and story IDs
- **Component renamed** — update the component key and story IDs
- **Component deleted** — remove from mapping, or move to `storybookOnly`/`figmaOnly`
- **Figma restructured** — node IDs change when components are deleted and recreated; re-enumerate
- **Story IDs changed** — happens when you rename the `title` in story meta

### Staleness Detection

The `/ds-sync` skill checks the mapping file at startup. If a Figma ID returns a null node, it logs a warning. If multiple IDs are stale, it suggests regenerating the mapping.

### Verify Screens (Optional)

Verify screens are Figma frames that contain instances of every component in a section, arranged like the Storybook layout. They're useful for visual QA:

1. Create a frame in Figma for each section (e.g., "Verify - Actions - Light")
2. Place instances of every component in that section
3. Set the frame's variable mode to Light or Dark
4. Add the frame IDs to the mapping as `verifyLight` / `verifyDark`

The `/ds-sync` skill uses these for pixel-level comparison against Storybook screenshots.

---

## Example Workflow

1. You add a new `DatePicker` component to your library
2. You create a Storybook story with variants (default, with value, disabled, error)
3. You create the corresponding Figma component with matching variants
4. You add the mapping:
   ```json
   "DatePicker": {
     "figmaId": "1234:5678",
     "figmaType": "COMPONENT_SET",
     "variantCount": 4,
     "stories": [
       "components-forms-datepicker--default",
       "components-forms-datepicker--with-value",
       "components-forms-datepicker--disabled",
       "components-forms-datepicker--error"
     ],
     "sourceFile": "src/components/date-picker/date-picker.tsx"
   }
   ```
5. Run `/ds-sync DatePicker` to verify alignment
6. Run `/ds-report` to update the drift score

---

## See Also

- **[DS Registry](./ds-registry.md)** — the mapping file is automatically incorporated into the unified `ds-registry.json` when you run the registry generator. If the registry exists, skills use it as a fast path instead of reading the mapping file and manifest separately.
