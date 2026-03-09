# ETASKULATOR — Meeting Agenda App

## Overview

Single-file web application (`task_manager_v1.0.html`) for tracking engineering team tasks and meeting agendas. No build step, no server, no dependencies — open directly in a browser. Default app name is "ETASKULATOR" (customizable via settings). Version tracked via `APP_VERSION` constant; displayed in header and included in default filename (`task_manager_v{version}.html`).

## Tech Stack

- Vanilla HTML / CSS / JavaScript (no frameworks)
- Data embedded as JSON inside the HTML file itself (self-contained, portable)
- `localStorage` (key: `eng-agenda-v1`) kept as secondary backup
- External font: IBM Plex (Google Fonts) — only external dependency

## How to Run

Open `task_manager_v1.0.html` in any modern browser. No install or build required.

## Companion Files

- `WATCHLIST.md` — Known limitations and edge cases worth monitoring (17 items across 5 categories)
- `test_checklist.html` — Interactive test checklist (149 tests across 17 sections). Self-contained HTML, dark theme, localStorage persistence, export to clipboard.

## Architecture

**Single state object** drives the entire app:

```
state = {
  projects: [{ id, name }],
  tasks:    [{ id, projectId, desc, note, owner, status, createdAt, dueDate, progress, subtasks }],
  onhold:   [{ id, projectId, projectName, desc, note, owner, priorStatus, heldAt, createdAt, dueDate, progress, subtasks }],
  archive:  [{ id, projectId, projectName, desc, note, owner, doneAt }],
  trash:    [{ id, source, data, deletedAt }],
  settings: { theme, fontSize, compact, showNotes, title }
}
```

**Data flow:** User action → mutate `state` → `saveState()` (localStorage backup + auto-save to file if linked) → `render()` rebuilds DOM.

**Self-contained save:** State is embedded as JSON inside the HTML via sentinel comments (`<!--DATA_BLOCK_START-->` / `<!--DATA_BLOCK_END-->`). Uses the File System Access API (Chrome/Edge) for auto-save — one-time permission grant, then every edit writes silently to the file. Falls back to `downloadFile()` browser download for Firefox/Safari.

**Load priority:** embedded `<script id="embedded-data">` → localStorage fallback → demo seed data.

**Code sections** (marked with comment banners in the JS):
- STATE — state shape and globals
- PERSISTENCE — load/save, auto-save via File System Access API (`writeToFile()`, `handleSaveClick()`, `handleSaveAs()`, `unlinkFile()`, `dismissBrowserWarning()`), download fallback (`downloadFile()`), save indicators, demo seed data, `exportMarkdown()`, `buildMarkdown()`, `showToast()`
- RENDER — DOM construction (`render()`, `renderActive()`, `renderOnHold()`, `renderArchive()`, `renderTrash()`, `renderTask()`, `updateStats()`)
- DRAG AND DROP — `initDragAndDrop()`, `cleanupDrag()` — HTML5 drag-and-drop via event delegation on `#main-content`
- UNDO — `pushUndo()`, `undo()`, `updateUndoBtn()` — in-memory undo stack (not persisted)
- ACTIONS — status cycling, task CRUD, project CRUD, hold/resume, trash (soft delete, restore, purge), new template
- SUBTASKS — subtask helpers (`getEffectiveProgress`, `toggleSubtaskExpand`, `renderSubtaskBadgeHeld`), subtask CRUD (`quickAddSubtask`, `editSubtask`, `cycleSubtaskStatus`, `deleteSubtask`)
- MODAL — add/edit task dialog (shared between tasks and subtasks), two-column layout with live preview, `cycleModalStatus()`, `updateModalPreview()`, `updateModalBreadcrumb()`, `openReadOnlyModal()`, `closeReadOnlyModal()`
- SETTINGS — settings panel (themes, font size, compact mode, notes toggle, custom title)
- TABS — active / on-hold / archive / trash view switching
- UTILS — `esc()` (HTML escaping), `uid()`, `validateDateInput()`, `cycleTime()`, `doneVsDueLabel()`, `doneVsDueClass()`
- BOOT — `beforeunload` warning, `loadState()`, `purgeOldTrash()`, `applySettings()`, `updateSaveIndicator()`

## Conventions

- **ID prefixes:** projects `p{timestamp}`, tasks `t{timestamp}`, subtasks `st{timestamp}`, on-hold `h{timestamp}`, archive `a{timestamp}`, trash `x{timestamp}`
- **Status cycle:** TODO → WORKING → DONE (then auto-archived)
- **On hold:** any active task can be paused via ⏸ button; resumes to its prior status
- **Functions:** camelCase (`cycleStatus`, `saveTask`)
- **CSS classes:** kebab-case (`.task-status`, `.project-header`)
- **CSS theming:** custom properties on `:root` (dark theme by default, print overrides at end of `<style>`)
- **CSS themes:** `body.theme-light`, `body.theme-high-contrast`, `body.theme-soft-dark`, `body.theme-cyberpunk` override `:root` variables; default (Midnight) has no body class
- **CSS display classes:** `body.font-small`, `body.font-large`, `body.font-presentation` for font sizing; `body.compact` for tighter rows; `body.hide-notes` hides `.task-note` elements
- **`.task-meta` vs `.task-note`:** both styled identically, but `.task-meta` is NOT hidden by `hide-notes` — use for system metadata (held dates, etc.)
- **HTML escaping:** always use `esc()` when inserting user text into HTML
- **No semicolons** are not enforced — the code uses semicolons

## Key Behaviors

- Clicking a status badge cycles TODO → WORKING → DONE
- Marking DONE auto-archives the task (moves from `tasks` to `archive` with timestamp)
- Clicking ⏸ on a task moves it to the On Hold tab, preserving its prior status
- On Hold tab groups held tasks by project; ▶ Resume returns them to active with their prior status
- If a project was deleted while a task was on hold, resuming re-creates the project
- Deleting a task from any tab (active, on-hold, archive) moves it to Trash (soft delete, no confirmation needed)
- ✦ New button resets the document to a completely blank template (with confirmation)
- Archive view groups completed tasks by project (with project name as group header)
- All delete buttons, New Template button hidden in read-only snapshots
- Print layout hides UI controls and uses light theme
- First load seeds demo data if no embedded data and no localStorage
- Traffic-light save button: Red "↓ Save" (no file linked), Amber "⟳ Reconnect" (stored handle needs re-grant), Green "✓ Auto-Save" (linked and active)
- Red state: first-time use or after unlinking — clicking opens file picker or downloads (non-Chrome)
- Amber state: stored handle found but permission needs user gesture — clicking requests permission, then goes green
- Green state: auto-save active — clicking triggers immediate save (bypasses debounce)
- `hasPendingHandle` flag tracks amber/reconnect state (stored handle exists but `perm === 'prompt'`)
- Auto-save writes are debounced at 1000ms (`writeDebounceTimer`) to reduce disk I/O on Dropbox
- localStorage backup still writes immediately on every `saveState()` call (no debounce — it's the safety net)
- Manual save (clicking button) bypasses debounce via `clearTimeout(writeDebounceTimer)` + direct `writeToFile()`
- Auto-save uses File System Access API (Chrome/Edge); falls back to `downloadFile()` for Firefox/Safari
- Save indicator shows linked filename: "auto-save → filename.html" / "saving..." / "✓ saved" / "● unsaved"
- After "✓ saved" display (1.5s timeout), indicator reverts to "auto-save → filename.html"
- File picker `startIn` option uses stored/current handle as directory hint — picker opens in same directory from second save onward
- Save As button (`⤓ Save As`): always opens file picker, saves to new location, switches `fileHandle` to new file
- Save As button falls back to `downloadFile()` on Firefox/Safari
- Unlink button (`✕`): small button next to save button, visible only when green (linked) — disconnects file handle, clears IndexedDB, reverts to red state
- `unlinkFile()` clears `fileHandle`, `hasPendingHandle`, sets `isDirty = true`, deletes stored handle from IndexedDB
- `beforeunload` warns only if there are unsaved changes AND no file handle is linked
- Firefox/Safari warning banner: amber dismissible banner shown on first load when `hasFileAccess` is false
- Warning text explains auto-save unavailability and advises using Chrome/Edge or saving regularly
- Dismissing sets `localStorage('dismissed-browser-warning')` — banner only shows once per browser
- Save As button, Unlink button, browser warning hidden in read-only snapshots and print
- Escape key closes modals (task and settings); clicking overlay also closes them
- `</script>` in user text is escaped when embedding JSON to prevent HTML breakage
- ⚙ Settings button opens settings panel; hidden in read-only snapshots and print
- Settings persist in `state.settings` and travel with the file (embedded in saved HTML)
- Themes: Midnight (default), Light, High Contrast, Soft Dark, Cyberpunk — applied via CSS custom property overrides
- Font sizes: S (small), M (default), L (large), XL (presentation) — presentation mode enlarges all UI elements
- Compact mode: tighter padding on task rows for dense agendas
- Show Notes toggle: hides/shows `.task-note` elements; `.task-meta` elements always visible
- Custom Title: replaces "ETASKULATOR" logo text in header AND `document.title` (used by browser tab and print header); version tag always appended
- `APP_VERSION` constant: displayed as "v1.0" next to logo, included in `document.title`, default filename, and markdown export header
- Reset Defaults button restores all settings to factory defaults
- Stats bar below tabs shows live counts: TODO, WORKING, ON HOLD, DONE, TOTAL — updates on every render
- Stats bar visible in all views, snapshots, and print; respects compact and presentation font sizes
- Task age indicators show time since creation: "today", "1d", "3d", "1w", "3w", "2mo" — displayed as small badges in the owner column
- Age color coding: fresh (< 7d, subtle), warn (7–20d, amber), stale (21d+, red) — immediately highlights neglected tasks
- Age badges appear on Active and On Hold tabs; `createdAt` is preserved through hold/resume cycle
- Tasks without `createdAt` (pre-existing data) gracefully show no badge
- Optional due dates on tasks — set via the edit modal (not quick-add row)
- Due date badges: 📅 with label ("due today", "due in 3d", "due Mar 5", "2d overdue")
- Due date color coding: normal (> 3d, subtle), soon (1–3d, amber), today (blue/accent), overdue (red)
- Due dates preserved through hold/resume cycle
- Tasks without `dueDate` show no badge (backward compatible)
- Manual task progress (0–100%) set via range slider in edit modal (step 5)
- Progress bar displayed below task description in Active and On Hold views
- Progress color coding: low (< 40%, subtle), mid (40–74%, amber), high (75%+, green)
- Tasks with 0% progress show no bar (backward compatible)
- Progress preserved through hold/resume cycle
- Progress included in markdown export as `[N%]` suffix
- Subtasks: optional `subtasks` array on tasks — `[{ id, desc, note, owner, status, createdAt, dueDate, progress, doneAt }]`
- Subtasks are a full hierarchy layer: project → task → subtask — subtasks behave like tasks with all the same properties
- Subtask ID prefix: `st{timestamp}`
- Subtask status cycles: TODO → WORKING → DONE (no auto-archive — subtasks stay in place when DONE)
- Subtasks render as indented task rows below the parent (`.subtask-container` with left border, 28px indent)
- Collapsed by default: `▸ 2/4 subtasks` toggle badge (click to expand/collapse)
- `expandedSubtasks` Set tracks which tasks have subtasks expanded (session-only, not persisted)
- Expanded subtask rows have full task controls: status cycling, click-to-edit, delete
- Subtask rows ARE draggable within the same parent (grip handle visible, smaller font)
- Subtask rows do NOT have a hold (⏸) button — only parent tasks can be held
- Quick-add subtask bar appears at bottom of expanded subtask list (inline desc + owner + button)
- Inline `+` button on the right edge of each task row (`.task-add-sub`); hidden by default, appears on hover; clicking it opens the quick-add bar below
- Clicking a subtask row opens the same task modal but hides the project selector (inherits from parent)
- `editingSubtaskParentId` and `editingSubtaskId` globals track which subtask is being edited
- `saveTask()` handles both regular task saves and subtask saves (checks `editingSubtaskId`)
- `closeModal()` resets subtask editing state and re-shows project selector
- Auto-progress: when subtasks exist, progress auto-calculates from subtask completion (`getEffectiveProgress()`); manual slider disabled in modal
- Manual slider remains active for tasks without subtasks and for subtasks themselves
- Parent gating: task cannot be marked DONE (via status click or modal) unless all subtasks have `status === 'DONE'`
- Subtasks preserved through hold/resume cycle (deep-copied in `holdTask()`/`returnToActive()`)
- Subtasks included in markdown export as nested checkboxes with owner, due date, progress, notes
- On Hold tab shows subtask count badge (non-interactive, via `renderSubtaskBadgeHeld()`)
- DONE subtask rows show reduced opacity + strikethrough
- All-done subtask badge shows green accent color (`.subtask-toggle.all-done`)
- Tasks without `subtasks` (or empty array) show no badge (backward compatible)
- Subtask interactive elements (`.add-subtask-bar`, `.add-subtask-btn`) hidden in print and read-only snapshots
- Task modal: two-column layout (`.modal.task-modal`, width 720px) — form fields on left, live preview panel on right
- Modal preview panel (`.modal-preview`): shows live-updating status pill, description, notes preview, age/due badges, owner, progress bar
- Clickable status pill in preview panel: cycles TODO → WORKING → DONE → TODO via `cycleModalStatus()`; replaces the old `<select>` dropdown with `<input type="hidden" id="m-status">`
- `modalStatus` global: canonical source for status in the modal; synced to hidden input for `saveTask()` compat
- `updateModalPreview()`: central function called by all form `oninput`/`onchange` handlers; recomputes all preview elements
- Project/parent breadcrumb (`#modalBreadcrumb`): shows project name for tasks, "Project › Parent Task" for subtasks
- `updateModalBreadcrumb()`: reads project from `<select>` for tasks, looks up parent for subtasks
- Creation date override: checkbox `#m-created-override` + date field `#m-created-date`; unchecked by default
- Override checked + date entered → `saveTask()` writes that date as `createdAt` (new tasks) or updates existing `createdAt` (edit)
- Override unchecked → existing behavior preserved (today for new, unchanged for edit)
- Age badge in preview updates live when override date changes (via `taskAge()`/`ageClass()`)
- Notes textarea min-height: 120px (up from 64px) for better meeting note capture
- Modal responsive: `@media (max-width: 700px)` collapses to single column with preview above form
- Modal scales with font size: `body.font-large` → 760px, `body.font-presentation` → 840px
- All modal preview badges reuse existing helper functions (`taskAge`, `ageClass`, `dueDateLabel`, `dueDateClass`, `renderProgressBar`)
- `openAddTask()`, `editTask()`, `editSubtask()`: all reset/populate created date override and call `updateModalPreview()`
- `closeModal()`: resets `modalStatus`, created date override UI
- Date validation: `validateDateInput()` clears invalid date input values with red-flash animation (`.date-invalid-flash`)
- Sticky header/tabs/stats/search: `position: sticky` with cumulative `top` offsets; static in print; offsets adjust for font-small (52px header) and font-presentation (72px header)
- Read-only modal: clicking on-hold or archive rows opens `openReadOnlyModal(item, source)` — shows task details, badges, subtasks in a non-editable view; `closeReadOnlyModal()` restores original modal HTML
- `readOnlyModalOrigHTML` global: caches original modal innerHTML while read-only modal is open
- Escape key and overlay click route to `closeReadOnlyModal()` when read-only modal is active
- Subtask `doneAt`: set to today's date when subtask is marked DONE (via `cycleSubtaskStatus()` or `saveTask()` modal); cleared when status changes away from DONE
- Transformed badges on DONE subtasks: age badge becomes cycle time ("⏱ 3d" via `cycleTime()`), due badge becomes performance indicator ("✓ 2d early" / "✓ on time" / "✗ 1d late" via `doneVsDueLabel()`)
- Badge transform CSS classes: `.cycle-time` (neutral), `.done-early`/`.done-ontime` (green), `.done-late` (red)
- Old DONE subtasks without `doneAt` gracefully fall through to live badges (backward compat)
- Subtask delete confirmation: `deleteSubtask()` shows `confirm()` dialog before deleting
- Larger + subtask button: font-size 18px (up from 13px), font-weight 600, hover opacity 0.7
- Search/filter bar between stats bar and main content — live-filters tasks by description, notes, owner, or project name
- Search works across Active, On Hold, and Archive tabs
- Search clears automatically on tab switch
- ✕ clear button appears when search has text
- "No tasks matching..." message shown when filter yields zero results
- Search bar hidden in read-only snapshots and print
- Undo button (↩) appears in header after any data-modifying action; hidden when stack is empty
- Undo restores previous state snapshot (projects, tasks, onhold, archive, trash — NOT settings)
- Up to 10 undo levels stored in-memory (`undoStack` array, `MAX_UNDO = 10`)
- Undo stack is NOT persisted — resets on page reload (intentional: undo is session-only)
- `pushUndo()` called before every data-modifying action; NOT called for settings changes
- Ctrl+Z (Cmd+Z on Mac) keyboard shortcut triggers undo
- Undo button hidden in read-only snapshots and print
- 📋 Export button copies agenda as Markdown to clipboard; falls back to .md file download if clipboard fails
- Export includes: title, date, active tasks grouped by project (with checkboxes, 🔶 for WORKING, owner, due dates, notes), on hold tasks, and summary stats
- Export button visible in read-only snapshots (non-destructive, useful for recipients)
- Toast notification appears at bottom of screen for 2 seconds on export
- Drag-and-drop reordering for tasks (within same project), subtasks (within same parent), and projects (reorder project groups)
- Drag initiated only via ⠿ grip handle — prevents accidental drags from other interactions
- Task grip: thin column (24px) on the left edge of each task row
- Project grip: ⠿ icon in the project header, before the project name
- Blue accent line shows drop position (above or below target)
- Dragged element shows reduced opacity while in flight
- Drag reorder pushes undo before committing — fully undoable
- Drag grips, draggable attributes hidden in read-only snapshots
- Drag grips hidden in print; print grid overrides to remove grip column
- Collapsible project groups: click ▾ arrow or project name to collapse/expand
- Collapsed state is session-only (`collapsedProjects` Set, not persisted)
- Task count shown in project header parenthetically, e.g. "MY PROJECT (3)"
- Print always expands collapsed projects; collapse toggle hidden in print
- Project actions (rename/remove) hidden in print
- "▸ Collapse All" / "▾ Expand All" toggle button above project list (only when 2+ projects exist)
- Collapse All button hidden in print
- Trash tab: soft-delete destination for tasks, held tasks, archived tasks, and projects
- Trash items store `source` ('active'/'onhold'/'archive'/'project'), `data` (original item), `deletedAt` date
- Deleting a project moves the project AND all its active/held tasks to trash individually
- Delete project still shows a confirmation dialog (since it's multi-item); individual task deletes do not
- ↩ Restore button returns items to their original location (active/onhold/archive/projects)
- Restoring an active or held task re-creates its project if the project was deleted
- ✕ Permanent delete on individual trash items (with "cannot be undone" confirmation)
- 🗑 Empty Trash button permanently deletes all trash items (with confirmation)
- Auto-purge: `purgeOldTrash()` runs on boot, removes items older than 30 days
- Trash items show: description, owner, deleted date, days remaining before purge
- Trash view groups items by source type (Tasks, On Hold, Archived, Projects)
- Search works in trash tab (filters by desc, note, owner, projectName, name)
- Trash tab hidden in read-only snapshots
- Trash actions (restore, delete, empty) hidden in print
- Trash badge shows count on tab
- `state.trash` included in undo snapshots
- `newTemplate()` clears trash along with other data
- Backward compat: `if (!state.trash) state.trash = [];` in loadState
- `applySettings()` called at boot and after every settings change
- `newTemplate()` preserves current settings when clearing data
- `DEFAULT_SETTINGS`: `{ theme: 'midnight', fontSize: 'default', compact: false, showNotes: true, title: '' }`
