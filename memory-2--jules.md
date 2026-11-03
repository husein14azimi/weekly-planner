# Weekly Planner: Project Memory & Architecture (by Jules)

This document records the architectural decisions, design patterns, technical specifications, and implementation details for the **Weekly Planner** web application. It serves as an authoritative source of truth for the codebase and a continuity reference for future iterations.

---

## 1. Project Overview & Working Agreement

- **Product**: A responsive, date-independent template weekly planner web application.
- **Core Architecture**:
  - Single self-contained HTML file (`index.html`) with zero build step.
  - No external runtime framework dependencies (removed `@ionic/core` CDN in favor of standard vanilla JavaScript, CSS logical properties, and native Web APIs).
  - Deployable directly to GitHub Pages or any static web host, fully functional offline once assets are cached.
- **Languages**: Full bi-directional support:
  - **English (LTR)**
  - **Persian / Farsi (RTL)** with Iranian calendar week structure (Saturday to Friday) and numeral formatting toggle (Persian digits vs. Latin digits).
- **Target Devices**:
  - Desktop browsers (Chrome, Firefox, Safari, Edge).
  - Mobile browsers, optimized specifically for Android Chrome and iOS Safari touch interactions.

---

## 2. Product Specifications & Rules

### 2.1 Calendar & Grid Structure
- **Template Week**: Independent of specific dates/months/years, eliminating Jalali/Gregorian calendar conversion issues.
- **Columns (7 Days)**: `Sat`, `Sun`, `Mon`, `Tue`, `Wed`, `Thu`, `Fri` (indices 0 through 6).
- **Rows (18 Hours)**: `5:00` to `22:00` (indices 5 through 22). Each row represents an hour slot; the last slot is 22:00–23:00.
- **Grid Total**: 126 addressable cells (`d-h` key format).

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
  - Deleting a task assigned to $N$ cells (with $M$ non-empty captions) triggers a confirmation dialog: *"'{task}' is used in N cells, with M captions. Delete it together with those captions?"*.
  - Confirming empties the cells completely (captions included). Reversible in one step via Undo.

### 2.3 Tinted Text Contrast Engine (OKLCH)
- Accessible text readability against any arbitrary background color:
  - Background hex is converted through Linear sRGB to OKLCH (Lightness, Chroma, Hue).
  - Text hue is matched to the background's hue, adjusting lightness iteratively until contrast reaches at least **4.8:1** (exceeding WCAG AA 4.5:1).
  - Fallback to `#111111` or `#ffffff` if chromatic search bounds cannot achieve adequate contrast.
  - Cell captions inherit the tinted color with 85% opacity.
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
- **"Save as task"**:
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
  - `Enter` / `F2`: Edit title (focuses combobox).
  - `Delete` / `Backspace`: Clear selected cells.
  - `Ctrl/Cmd + C`: Copy selected block to internal clipboard.
  - `Ctrl/Cmd + V`: Paste clipboard into selected range.
  - `Ctrl/Cmd + Z`: Undo.
  - `Ctrl/Cmd + Y` or `Ctrl/Cmd + Shift + Z`: Redo.

### 2.6 Persistence & Schema
- **Storage**:
  - Saved automatically to `localStorage` under key `planner.v1`.
  - Schema: `{ v: 1, tasks: [{ id, name, color }], cells: { "d-h": { t, x, c, k } }, lang: "en"|"fa", latin: boolean }`.
  - Graceful fallback: If `localStorage` is disabled or throws an exception, changes persist in memory with a warning banner prompting the user to export.
- **Export / Import**:
  - Export generates a formatted `.json` file (`weekly-planner.json`).
  - Import validates task IDs, cell coordinate bounds (`d: 0..6`, `h: 5..22`), and valid hex colors before confirming and sanitizing the dataset.

---

## 3. UI/UX Architecture

### 3.1 Responsive Strategy
- **Desktop (> 768px)**:
  - Sticky day headers (`top: 0`) and sticky hour headers (`inline-start: 0`).
  - Fixed right (or left in RTL) side panel (`aside#sidePanel`) with segmented tabs (`Cell` and `Tasks`).
  - Palette swatches and custom color picker.
- **Mobile ($\le$ 768px)**:
  - Responsive column widths displaying approximately 3.5 to 4 columns at a time with horizontal scroll.
  - Bottom sheet with three lifecycle states:
    - **Hidden**: Fully collapsed off-screen.
    - **Peek** (`195px` height): Displays selection count, title combobox, "Save as task" checkbox, and quick action buttons (Copy/Paste/Clear).
    - **Full** (`80dvh` height): Expands with backdrop to reveal captions, full color palette, and task manager.
  - Gesture support: Touch swipe on the handle bar (`#sheetHandle`) smoothly toggles between `peek` and `full`.
  - Dynamic scroll padding on `#gridWrapper` prevents the bottom sheet from covering selected cells.

### 3.2 Bidirectional (RTL / LTR) Design
- Document-level `dir="rtl"` and `lang="fa"`.
- Logical CSS properties throughout: `margin-inline`, `padding-inline`, `border-inline`, `inset-inline-start`.
- UI strings localized in dictionary `L.en` / `L.fa`.
- Numbers formatted via `Intl.NumberFormat('fa-IR')` with a toggle for Latin digits (`123` vs `۱۲۳`).
- User input preserves `dir="auto"`.
- Standard icons remain unmirrored per design agreement for Iranian web users.

---

## 4. Key Implementation Decisions (Jules)

1. **Elimination of Ionic Framework Overhead**:
   - Replaced CDN-based Ionic Core (`@ionic/core`) web components with native, highly performant CSS and JavaScript.
   - Eliminates custom-element registration lag, hydration delays, and modal breakpoint synchronization glitches.
   - Yields instantaneous initial page loads and zero external network latency.

2. **Dialog & Notification Architecture**:
   - Built custom modal dialogs (`ask()`) returning Promises over native backdrop overlays, providing clean confirmation flows for cascading deletions, data resets, and JSON imports.
   - Toast notification system (`toast()`) providing non-intrusive feedback for copy, paste, export, and import actions.

3. **Robust Sanitization on Import**:
   - Strict validation of JSON input schema ensures corrupted or malicious files cannot compromise the planner state or execute unauthorized operations.

---

## 5. Verified User Journeys (CUJs)

The following workflows were automated and visually verified using Playwright:

1. **Desktop Exploration & Task Assignment**:
   - Rendered 7-day $\times$ 18-hour grid.
   - Selected cell, searched task via fuzzy combobox, assigned task, and added caption.
   - Switched to Tasks tab and added custom task.
2. **Persian RTL Transition**:
   - Switched UI language to Persian. Verified full RTL layout mirroring, Vazirmatn typography, and Persian numeral formatting.
3. **Mobile Bottom Sheet Lifecycle**:
   - Tested 393px mobile viewport (Pixel / Galaxy size).
   - Tapped cell $\to$ bottom sheet entered `peek` state without obscuring cell.
   - Tapped handle $\to$ expanded to `full` state with backdrop blur.
   - Navigated between `Cell` and `Tasks` tabs seamlessly.

---

## 6. Future Scope & Recommendations

While deferred for the core release to guarantee stability and performance, future roadmap items may include:
- Current day column highlight based on real-time client date.
- Hours-per-task weekly summary statistics and progress meters.
- Print / PDF export stylesheet optimized for A4 paper.
- PWA service worker manifest for one-tap home screen installation and offline asset caching.
