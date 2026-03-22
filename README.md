# ETASKULATOR

A single-file HTML task manager for engineering team meetings. No server, no build step, no dependencies. Open in a browser and go.

## Features

- **Projects & Tasks** — organize work into projects with tasks and subtasks
- **Status Tracking** — cycle tasks through TODO → WORKING → DONE (auto-archives on completion)
- **On Hold** — pause tasks and resume later with prior status preserved
- **Archive & Trash** — completed tasks auto-archive, deleted items go to trash with configurable auto-purge
- **Auto-Save** — File System Access API (Chrome/Edge) saves directly to your file with 1-second debounce; localStorage backup on every change
- **Import/Export** — JSON backup/restore, HTML snapshot export (read-only, shareable), Markdown clipboard export
- **Print Summary** — structured 4-section printout: Active, On Hold, Recently Completed, Recently Deleted
- **Search** — live filter across descriptions, notes, owners, and project names
- **Drag & Drop** — reorder tasks, subtasks, and projects
- **Subtasks** — full hierarchy with independent status cycling, auto-progress calculation
- **Undo/Redo** — 10-level undo with Ctrl+Z / Ctrl+Y
- **5 Themes** — Midnight, Light, High Contrast, Soft Dark, Cyberpunk
- **Keyboard Shortcuts** — Ctrl+Z/Y, Ctrl+Enter, Escape, / (search)
- **Settings** — font size (S/M/L/XL), compact mode, custom title, purge intervals

## Getting Started

1. Download `task_manager_v2.0.html`
2. Open it in Chrome or Edge (recommended for auto-save)
3. Click `?` in the header for the built-in help guide

Firefox and Safari work but require manual saves (download button).

## File Format

Everything is stored as JSON embedded directly in the HTML file between `<!--DATA_BLOCK_START-->` and `<!--DATA_BLOCK_END-->` sentinels. The file modifies itself on save — no external database needed.

## Test Data

The `test_data/` folder contains sample datasets you can import via Settings → Import to try out the app with realistic data.

## Documentation

| File | Description |
|------|-------------|
| `CLAUDE.md` | Development guide and conventions |
| `COMPONENTS_MAP.md` | Full structural map of the HTML file |
| `STYLE_GUIDE.md` | CSS system, themes, component patterns |
| `DATA_MODEL.md` | Database schema and task lifecycle |
| `CHANGELOG.md` | Version history |
| `WATCHLIST.md` | Known limitations and edge cases |

## License

Personal project by 8rian 8rown. Not currently licensed for redistribution.
