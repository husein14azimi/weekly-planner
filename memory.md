# Weekly Planner: Project Memory (checkpoint)

Checkpoint of everything agreed and built so far in this chat. The source of truth for the code is the single HTML file (`weekly-planner.html`, to be renamed `index.html` for GitHub Pages).

---

## 1. Working agreement

- Claude acts as a professional frontend developer.
- The project runs in **phases**: vision/knowledge phase first, then the dev phase.
- **Claude must not start coding or move to the next phase until the user explicitly says so** ("proceed" / "go with defaults" + permission).
- The user likes: pointing out catches, gotchas and missing vision *before* coding; each open question comes with a recommended default so the user can answer "go with defaults".
- Stack: **one HTML file, vanilla JS, no build step, no database**. The goal is to test the frontend only; backend and persistence come later.
- Languages: **English (default) and Persian (Farsi, RTL)**.
- Target devices: desktop, plus **Android phones** (Chrome). The user will test via a private hosted link and later via GitHub (Pages).

---

## 2. Product vision

A **weekly planner** (template week, no dates):

- Columns: 7 days in this order: **Sat, Sun, Mon, Tue, Wed, Thu, Fri** (week starts Saturday).
- Rows: **hourly slots from 5:00 to 22:00** (18 rows, each labeled by its start hour; the last slot is 22:00–23:00).
- Each cell has editable **text** and **color**.
- **Pre-defined tasks** (name + color) can be chosen when editing a cell, to avoid typing and misspelling. A task's color is applied to the cell.

---

## 3. Decisions log (by topic)

### 3.1 Data and task rules
- **Template week**, not a real calendar week. No dates, so no Jalali/Gregorian problem.
- **Tasks are references, not snapshots.** Renaming or recoloring a task updates every cell that uses it.
- **Every task has a color.** Task names do **not** need to be unique.
- **Task owns the cell color.** When a cell has a task, the cell's manual color control is **inactive** (dimmed, non-interactive), with a visible note "Color comes from task 'X'". Assigning a task clears any manual color.
- A cell can hold a **task + a caption** (free text). The caption is independent of the task.
- The title field is **not strict**: a cell title is either a task link or free text.
- **Deleting a task:**
  - Unused task → deleted immediately (undo covers it).
  - Used task → a dialog says "used in N cells, with M captions. Delete it together with those captions?" On Yes the cells are emptied completely, **captions included**. M counts only non-empty captions. Undo restores everything in one step.

### 3.2 Title field and suggestions
- The field is an ARIA combobox with **loose matching** of existing tasks (no uniqueness needed; the goal is to remind users of tasks they defined and forgot).
- Matching: character bigram/trigram (Dice) overlap, plus a "contains" bonus and a subsequence bonus, with usage frequency as a tiebreaker. Threshold about 0.12 on the base score. Top 5 shown, each with a color swatch. An empty field lists all tasks, sorted with `Intl.Collator`.
- **Persian normalization (matching only; stored and displayed text stays as typed):** unify Arabic ي/ك with Persian ی/ک, ignore ZWNJ and diacritics, treat Persian/Arabic/Latin digits as equal.
- **Exact name match** (trim, case-insensitive) links to the existing task and hides "Save as task".
- **Free text:** shows a **"Save as task" checkbox, pre-checked**. A new task takes the cell's manual color if any, otherwise the next unused palette color. On a multi-cell selection all cells link to the new task.
- Enter or blur commits, Esc reverts. Each commit is one undo step.

### 3.3 Color and text contrast
- Curated 12-color palette plus a custom `<input type=color>`, and a "×" to clear the color.
- **Tinted text:** text color follows the background's hue. It is computed in OKLCH by keeping the hue and moving lightness until contrast is at least 4.8:1 (WCAG-style luminance). If no tinted color reaches the contrast, it falls back to near-black or near-white. It is computed, never stored. The caption uses the same ink at 85% opacity.

### 3.4 Selection, editing, history
- Selection: click, drag, Shift+click, click a day header (whole column) or an hour header (whole row).
- Mixed selections: manual color applies to free cells only (note: "applied to X of N"); differing values show "Mixed".
- Copy/paste: carries title, caption and color; pasted task cells stay linked. One copied cell fills the whole selection; a block pastes from the selection's top-left (first by day/hour index), clipped at the grid edge. The clipboard is internal to the page.
- Tasks spanning several hours: **separate cells** (the user copies/pastes), no merged blocks.
- **Undo/redo** covers everything (snapshot history, 100 steps, not persisted across reloads).
- Keyboard (grid focused): arrows (follow visual direction, flipped in RTL), Shift+arrows, Enter/F2 (edit title), Delete/Backspace (clear), Ctrl/Cmd+C/V/Z/Y. Shortcuts only act when the grid has focus, so text fields keep native behavior.

### 3.5 Persistence
- Autosave to **localStorage**, key `planner.v1`, payload `{v:1, tasks, cells, lang, latin}`.
- **JSON export/import:** export is `{v:1, tasks, cells}`. Import is validated and sanitized, with confirmation before replacing data, and text is always rendered as text.
- If storage fails, the app falls back to memory and shows a warning banner.
- First run seeds sample tasks (Work, Study, Exercise, Meals, Rest) and a few cells, in the active language. "Reset samples" is in the menu.

### 3.6 Language and RTL
- `lang` and `dir` are set at runtime; CSS uses logical properties; the side panel flips to the inline end.
- User content uses `dir="auto"`; interpolated names in messages are wrapped with Unicode isolates (FSI/PDI).
- All UI strings live in one table `L.en` / `L.fa`. Messages with numbers are functions (so word order can differ per language).
- Persian UI numerals via `Intl.NumberFormat('fa-IR')` by default, with a toggle for Latin digits (button shown only in Persian mode). User text is never converted.
- Font: **Vazirmatn** from Google Fonts with a system-font fallback (embedding is possible later for offline use).
- **Icons are NOT mirrored in RTL** (user decision: Persian users are used to the same icons).
- Out of scope: Finglish input, Jalali dates, wrong-keyboard-layout rescue.

### 3.7 Mobile decisions (Android Chrome target)
- **Superseded (see section 8):** originally "stay vanilla JS, no UI library". After phone testing the hand-built sheet/tabs/menu felt non-native, so app chrome now uses Ionic web components via CDN. The grid and all data logic stay vanilla.
- Layout: full week grid with horizontal scroll and sticky headers, **about 4 day columns visible** on phones.
- **Tap selects** one cell and does not open the keyboard. **Double-tap/double-click** starts editing the title.
- **"Range" toggle button** (mobile only): turns on range mode, where touch-dragging selects a range (grid stops scrolling by touch) and tapping a second cell extends the range. Edge auto-scroll while dragging. Header taps still select a whole day/hour.
- **Bottom sheet** (≤760px width) with states **hidden / peek / full** and **two tabs**:
  - **Cell** tab: selection count, title field with suggestions, "Save as task", Copy/Paste/Clear, then (full only) caption, color and the color note.
  - **Tasks** tab: task list (rename, recolor, delete, usage count) and "Add task". Opening it on mobile expands the sheet to full.
  - Selecting a cell switches back to the Cell tab; a hidden sheet opens to peek.
  - The same two tabs are used on desktop (side panel at the inline end).
- Toolbar: Undo/Redo as SVG icon buttons, "Range" toggle (mobile), "⋯" menu (Export, Import, Reset samples, language, digits).
- Touch basics: targets at least 44px, inputs at 16px font size, `touch-action: manipulation`, `viewport-fit=cover` and safe-area padding, `interactive-widget=resizes-content` (Chrome Android keeps the sheet above the keyboard), `dvh` for the sheet's max height, pinch-zoom left enabled.
- Grid gets bottom padding equal to the sheet's height (ResizeObserver) so the sheet never covers the selected cell.
- Responsive web page only. PWA/installable (manifest + service worker) is deferred.

---

## 4. Current build

### 4.1 Files and links
- Source: `weekly-planner.html` (single file, about 27 KB). Rename to `index.html` for GitHub Pages.
- Private hosted preview (publishing link, visible only to the user): https://claude.ai/artifact/4wMiSwCV6pfUGZSWzbp9bR
- **Export (file download) is inert on the preview link** (hosted pages block script-started downloads). It should work on GitHub Pages or a local file. Import uses a file input.

### 4.2 Code structure (inside the one file)
Sections in the `<script>`:
1. **Constants:** `START=5`, `END=22`, `ND=7`, palette `PAL`.
2. **Strings:** `L.en`, `L.fa`, accessed via `tr(key, ...args)`.
3. **State:** `S = {tasks:[{id,name,color}], cells:{"d-h":{t,x,c,k}}}`. `d` is 0..6 (Sat..Fri), `h` is 5..22. Cell fields: `t` task id, `x` free-text title, `c` caption, `k` manual color. UI state: `sel={a,b}`, `clip`, `un`/`re` stacks, `lang`, `latin`, `rng`.
4. **Storage:** `LS.load/save`, `ok()` (validation), `clean()` (sanitizer).
5. **Color:** `lin`, `gam`, `lab`, `mk`, `lum`, `hex`, `ink(bg)` (tinted text, cached).
6. **Data ops:** `put`, `act` (snapshot + mutate + save + render), `undo`, `redo`, `copy`, `paste`, `clr`, `setColor`, `assign`, `seed`.
7. **Matching:** `norm`, `gr` (n-grams), `usage`, `suggest`.
8. **Rendering:** `renderGrid`, `paintSel`, `panel`, `tasks`, `render`, `applyLang`.
9. **Dialog and tasks:** `ask` (Promise over `<dialog>`), `delTask`, add task.
10. **Title combobox:** `sugg`, `commit`, key handling.
11. **Palette**, **grid interaction** (`hit`, pointer handlers, keyboard), **toolbar** (`exp`, import via file input, action map `A`).
12. **Mobile:** `setS`, `setTab`, `openSheet`, range toggle, menu, ResizeObserver for sheet padding.
13. **Init.**

Touch note: on touch in non-range mode, selection is done on `click` (so scrolling the grid does not select cells); mouse/pen and range mode select on `pointerdown`.

---

## 5. Not built / deferred

- Today's-column highlight (user deferred it).
- "Did you mean…" hint for close matches.
- Wrong-keyboard-layout rescue (off by default in plan; not built).
- Full ARIA grid structure (`role="row"` wrappers are missing; cells and headers have roles).
- Edge handles / long-press range selection (only the Range toggle exists).
- Hours-per-task summary, print stylesheet, task durations, categories, recurring rules, reminders and notifications, sharing, dark mode, 30-minute slots, PWA/offline.
- Embedding the Vazirmatn font for fully offline use.

---

## 6. Known caveats and things to verify

- **The code has not been run in a browser by Claude.** Neither the desktop build nor the mobile build has been tested. Expect to find bugs; the first test pass on a real Android phone and a desktop browser is the next real check.
- Things worth checking first: range-mode drag and edge auto-scroll; the bottom sheet states and keyboard behavior on Android Chrome; RTL arrow-key mapping; suggestion list inside the sheet; dialog styling; Persian string wording (the table is a draft and needs native review); storage behavior on GitHub Pages.
- Desktop toolbar no longer has Copy/Paste/Clear buttons (they moved into the Cell tab; shortcuts still work).
- The sheet's suggestion list opens inline below the title field on mobile (not upward, since the sheet would clip it). "Save as task" is visible in the peek state so tasks are not created silently.
- Task names are not unique; users can create duplicates deliberately (swatches tell them apart).
- Import errors and storage warnings show in the banner (the `alert` was replaced by a banner message).

---

## 7. Suggested next steps (not agreed yet; ask the user first)

1. User tests desktop + Android and reports bugs → fix pass.
2. Native-speaker review of the Persian strings.
3. Optional additions from the deferred list (today highlight, "did you mean" hint, ARIA rows, summary, print stylesheet).
4. Decide the real project's framework/backend, then port the plain-function modules (state, matcher, color, i18n) there.

---

## 8. Change: Ionic web components for app chrome (after phone test)

- User tested on phone: hand-coded bottom sheet/tabs/menu did not feel like a normal modern web app. Decision (user said "go with defaults"): use **Ionic Core web components via CDN** (`@ionic/core@8.8.5`, `css/core.css` + `ionic.esm.js`, no build step, still one HTML file).
- Replaced: bottom sheet -> `ion-modal#sheet` (swipeable, `breakpoints [0, .42, 1]`, `--height: 82dvh`, `backdropBreakpoint 1`, `focusTrap false`, `handleBehavior cycle`); Cell/Tasks tabs -> `ion-segment#seg`; menu -> `ion-popover`; confirm dialog -> `ion-alert` (`ask()`); import error flash -> `ion-toast`; toolbar and action buttons -> `ion-button`.
- Panel content lives in `div#PB`. On mobile (<=760px) it is moved into the modal's `ion-content`; on desktop it sits in `aside#P`. `place()` moves it on resize.
- Kept custom: grid, selection, title combobox, color palette, data/undo/i18n logic. The memory-storage banner `#warn` is unchanged.
- Not yet run in a browser. First checks: grid stays tappable while the sheet is at peek; swipe to 0 closes and the next cell tap reopens it; RTL; combobox inside the sheet; Tasks tab opens the sheet full.
