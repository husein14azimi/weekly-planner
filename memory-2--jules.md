# Chronos — Weekly Planner: Project Memory & Architecture (by Jules)

This document records the architectural decisions, design system, technical specifications, and implementation details for the modernized **Chronos Weekly Planner** web application. It serves as an authoritative source of truth for the codebase.

---

## 1. Project Overview & Philosophy

- **Product**: **Chronos** — A modern, date-independent template weekly planner web application.
- **Design Philosophy**:
  - Inspired by modern high-performance productivity tools (Linear, Notion Calendar, Cron).
  - Modern information architecture: high-density scheduling grid with breathing room, micro-interactions, soft elevation shadows, and rounded task cards.
  - Contextual floating action dock for rapid multi-cell actions without obscuring the canvas.
  - Slide-over inspector drawer for in-depth task creation, color management, and note editing.
  - Real-time weekly allocation statistics bar with categorized time chips.
  - First-class Dark Mode and Light Mode support with smooth theme transitions.
  - Zero-build, single-file static architecture (`index.html`) using vanilla JavaScript, modern CSS variables, and native Web APIs.
- **Languages**: Full bi-directional support:
  - **English (LTR)**
  - **Persian / Farsi (RTL)** with Iranian calendar week structure (Saturday to Friday) and numeral formatting toggle (Persian digits vs. Latin digits).

---

## 2. Product Specifications & Rules

### 2.1 Calendar & Grid Structure
- **Template Week**: Date-independent template week (Sat–Fri), eliminating calendar conversion issues.
- **Columns (7 Days)**: `Sat`, `Sun`, `Mon`, `Tue`, `Wed`, `Thu`, `Fri` (indices 0 through 6).
- **Rows (18 Hours)**: `5:00` to `22:00` (indices 5 through 22). Each row represents an hour slot; the last slot is 22:00–23:00.
- **Grid Total**: 126 addressable cells (`d-h` key format).
- **Sticky Headers**: Sticky top day headers (`top: 0`) and sticky inline hour headers (`inset-inline-start: 0`).

### 2.2 Data Model & Task Reference Architecture
- **Tasks (`S.tasks`)**:
  - Properties: `id` (string), `name` (string), `color` (hex string `#rrggbb`).
  - Tasks are referenced by ID (`c.t`), not stored as copies.
  - Modifying a task's name or color automatically updates all cells referencing that task.
  - Task names do not require uniqueness; distinct tasks can share names, differentiated by color swatches.
- **Cells (`S.cells[key]`)**:
  - `t`: Task reference ID (if assigned).
  - `x`: Ad-hoc free-text title (used when no task is linked).
  - `c`: Caption (secondary free text displayed below the title).
  - `k`: Manual background color (used only when no task is assigned; assigning a task clears `k`).
- **Cascade Deletion**:
  - Deleting an unused task occurs immediately (covered by undo).
  - Deleting a task assigned to $N$ cells (with $M$ non-empty captions) triggers a confirmation dialog: *"'{task}' is used in N cells (N hours), with M captions. Delete it together with those captions?"*.
  - Confirming empties the cells completely (captions included). Reversible in one step via Undo.

### 2.3 Tinted Text Contrast Engine (OKLCH)
- Accessible text readability against any arbitrary background color:
  - Background hex is converted through Linear sRGB to OKLCH (Lightness, Chroma, Hue).
  - Text hue is matched to the background's hue, adjusting lightness iteratively until contrast reaches at least **4.8:1** (exceeding WCAG AA 4.5:1).
  - Fallback to `#0f172a` or `#ffffff` if chromatic search bounds cannot achieve adequate contrast.
  - Cell captions inherit the tinted color with 88% opacity.
  - Results are memoized in `ic[bg]` for instantaneous grid repaint.

### 2.4 Loose Matching & Normalization (Persian / English)
- **Persian Normalization**:
  - Normalizes Arabic letters: `ي` $\to$ `ی`, `ك` $\to$ `ک`.
  - Strips zero-width non-joiners (`\u200c`) and Arabic diacritics/tanween (`[\u064b-\u065f\u0670]`).
  - Normalizes Eastern Arabic and Persian digits (`۰`-`۹`, `٠`-`٩`) to Latin digits (`0`-`9`) for search indexing.
  - Normalizes multiple spaces and trims strings.
- **Loose Matching**:
  - Bigram and trigram (Dice coefficient) similarity score with bonus for substrings and character subsequences.
  - Usage frequency in cells acts as a tie-breaker.
  - Threshold set to $> 0.12$.
  - Displays top 5 matching tasks with color swatches. When the field is empty, displays all defined tasks sorted using `Intl.Collator`.
- **"Save as reusable task"**:
  - Shown when typing a non-empty name that does not match an existing task name.
  - Pre-checked by default. When committed, creates a new task using the cell's manual color or the next available palette color.

### 2.5 Selection & Interaction Mechanics
- **Pointer & Touch**:
  - Desktop: Click, drag-to-select range, Shift+click range expansion, column/row header click for full day or hour selection.
  - Mobile Touch: Single tap selects a cell without triggering the virtual keyboard. Dragging scrolls the grid smoothly.
  - **Range Mode** (mobile button toggle): In Range mode, touch dragging selects a multi-cell range and edge auto-scrolls the viewport.
- **Keyboard Shortcuts** (active when grid has focus):
  - `Arrow keys`: Navigate active cell (automatically flipped horizontally in RTL mode).
  - `Shift + Arrow keys`: Expand selection range.
  - `Enter` / `F2`: Edit title (opens drawer and focuses combobox).
  - `Delete` / `Backspace`: Clear selected cells.
  - `Ctrl/Cmd + C`: Copy selected block to internal clipboard.
  - `Ctrl/Cmd + V`: Paste clipboard into selected range.
  - `Ctrl/Cmd + Z`: Undo.
  - `Ctrl/Cmd + Y` or `Ctrl/Cmd + Shift + Z`: Redo.

### 2.6 Persistence & Schema
- **Storage**:
  - Saved automatically to `localStorage` under key `planner.v1`.
  - Schema: `{ v: 1, tasks: [{ id, name, color }], cells: { "d-h": { t, x, c, k } }, lang: "en"|"fa", latin: boolean, theme: "light"|"dark" }`.
  - Graceful fallback: If `localStorage` is disabled or throws an exception, changes persist in memory with a warning banner prompting the user to export.
- **Export / Import**:
  - Export generates a formatted `.json` file (`weekly-planner.json`).
  - Import validates task IDs, cell coordinate bounds (`d: 0..6`, `h: 5..22`), and valid hex colors before confirming and sanitizing the dataset.

---

## 3. Modernized UI Components

1. **Floating Action Dock (`#floatingDock`)**:
   - Appears anchored above the bottom viewport when cells are selected.
   - Shows live selected cell count, Edit trigger button, Copy, Paste, and Clear buttons.
2. **Slide-Over Drawer (`#sideDrawer`)**:
   - Right-side panel (left-side in RTL) with backdrop blur overlay.
   - Segmented into `Selection` (slot title, combobox, captions, palette) and `Manage Tasks` (add/rename/delete tasks, usage counter).
3. **Weekly Allocation Stats Bar (`#statsBar`)**:
   - Live horizontal scroll strip rendering colored chips with task names and total weekly scheduled hours (`{count}h`).
4. **Theme Switcher (`#themeBtn`)**:
   - Smooth instant transition between Dark Mode (`data-theme="dark"`) and Light Mode (`data-theme="light"`), persisted in `localStorage`.
5. **Modern Card Cells (`.c-card`)**:
   - Cells display discrete rounded event cards with elevation shadows, clean typography, and tinted OKLCH colors.
