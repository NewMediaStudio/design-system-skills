---
description: Launch the Storybook dev server for component development
---

# Storybook

Launch the Storybook development server for previewing and developing design system components.

## Steps

1. Check if Storybook is already running on port 6006 using `preview_list`
2. If not running, start it using `preview_start` with the `storybook` configuration from `.claude/launch.json`
3. Confirm the server is accessible at `http://localhost:6006`
4. Report the URL to the user

## Notes

- If port 6006 is already in use, inform the user and suggest checking what's running on that port
- The launch configuration should be defined in `.claude/launch.json` — see `templates/launch.example.json` for the format
- Storybook must be installed in the project (`@storybook/react` or equivalent)

## Usage

```bash
/storybook
```
