# Kanban Board — Feature Plan (Hermes hand-off, partial-progress)

**Target file:** `C:\Users\cosmi\Projects\kanban-board\index.html` (single-file vanilla JS + Firebase Realtime DB)
**Constraint:** Keep single-file architecture. Synced state lives under `shared` (Firebase). Per-browser state lives in `local` (localStorage).

> Status legend: ✅ done · 🟡 partial · 🔲 not yet started

---

## Phase 1 — Column floating + rename + card archive

### 1.1 Move columns into per-project synced state — ✅ DONE

- `COLUMNS` renamed to `DEFAULT_COLUMNS` (used only for seeding/migration).
- `normalise()` now seeds `p.columns` from `DEFAULT_COLUMNS` when missing, and defaults `c.archived = false`.
- `defaultShared()` seeds `columns` + `crm: []` on the first project.
- `addProject()` seeds `columns` on new projects.

### 1.2 Floating/draggable columns with snap-to-slot — ✅ DONE

- Drag handle (`⋮⋮`) in each column header is the only `draggable` element for columns (cards still drag independently).
- `dataTransfer` key is `text/colId`; card drops check `e.dataTransfer.types.includes("text/colid")` to disambiguate (note: types lowercase).
- `wireBoardColumnDnD()` computes target index from cursor X vs each column's midpoint and shows a vertical `.col-drop-indicator` strip. On drop, reorders `proj.columns` and `saveState()`.
- "+ Add column" tile at the end of the board lets users append columns.

### 1.3 Editable column names — ✅ DONE

- Double-click column title swaps it for a `.column-title-input`. Enter or blur saves; Escape cancels.
- `beginColumnRename(colId, titleEl)` handles the swap.

### 1.4 Card archive — ✅ DONE

- Schema: `archived` field defaults to `false` (migration + new-card paths).
- `.card-archive` 📦 button (hover-visible) on each card with confirm → sets `archived = true`.
- `renderBoard()` filters archived cards from `visible`.
- Card modal's column dropdown reads from `proj.columns`.
- "📦 View Archived" button in sidebar; `openArchivedModal()` shows all archived cards with Restore / Delete.

---

## Phase 2 — CRM module — ✅ DONE

✅ **Built:**
- Sidebar Board ↔ CRM segmented toggle (`.view-switch`).
- `<main class="crm-main">` with toolbar (search, status filter, + New Contact) and `<table id="crmTable">`.
- `renderCRM()` populates thead (sortable headers, ↑/↓ indicator) and tbody (rows via `buildCRMRow`).
- `buildCRMRow()` produces editable text cells, color-coded status dropdown, date picker, and per-row delete.
- `addCRMContact()` pushes blank row, persists, focuses first cell.
- `applyViewMode()` toggles which `<main>` is visible based on `local.view` and updates the segmented control.

✅ **Originally already in place:**
- `shared.crm = []` is created, persisted, and migrated.
- `CRM_STATUSES = ["sent","interested","replied","skipped","planned","bounced"]` and `CRM_STATUS_COLORS` constants exist.
- `local.view = "board" | "crm"` and `local.crmFilters = { text, status, sortBy, sortDir }` exist on the local prefs object.

🔲 **To build:**

### 2.1 Sidebar Board ↔ CRM toggle

In the sidebar HTML (currently starts around the `<aside class="sidebar">` block), add a row above "Projects":

```html
<div class="view-switch">
  <button class="view-btn" data-view="board">Board</button>
  <button class="view-btn" data-view="crm">CRM</button>
</div>
```

Style `.view-switch` as a two-button segmented control. The active button gets `background: var(--primary); color: white;` based on `local.view`.

Wire in `wireEvents()`: clicks call `local.view = e.target.dataset.view; saveLocalPrefs(); renderAll();`

### 2.2 CRM main view

Add a second `<main class="crm-main">` block parallel to the existing `<main class="main">`. In `renderAll()`, toggle which is visible based on `local.view`:

```js
$(".main").style.display     = local.view === "board" ? "flex" : "none";
$(".crm-main").style.display = local.view === "crm"   ? "flex" : "none";
if (local.view === "crm") renderCRM();
```

### 2.3 Editable table

Row schema:
```js
{
  id: uid(),
  organization: "",
  contactName:  "",
  email:        "",
  phone:        "",
  status:       "planned",
  lastInteraction: null,
  nextAction:      "",
  createdAt:    Date.now()
}
```

Columns: Organization · Contact · Email · Phone · Status · Last Interaction · Next Action · ✕

- Text cells: `contenteditable="true"` divs; on `blur`, write to row + `saveState()`. Skip `saveState()` if value unchanged (avoids Firebase write storms on focus-changes).
- Status: `<select>` with the 6 options; `background-color: CRM_STATUS_COLORS[value]; color: white;`
- Last Interaction: `<input type="date">`.
- ✕: confirm → `shared.crm = shared.crm.filter(r => r.id !== row.id); saveState(); renderCRM();`

Top bar: `+ New Contact` (pushes blank row, focuses Organization), text filter input, status filter `<select>`.

### 2.4 Sort + filter

Local-only state (`local.crmFilters`). Clicking a column header toggles `sortBy` and `sortDir`. Filter applies the search text across `organization|contactName|email` and the status equality before sort.

---

## Out of scope / explicit non-goals

- No auth changes (keep anonymous Firebase auth).
- No CRM CSV import/export.
- No row drag-reorder in CRM (sort-only).
- Don't use absolute X/Y positions for column dragging — the insertion-index pattern is already in place and is the intended behavior for "designated spots."

---

## Acceptance checklist (full feature)

- [x] Columns drag horizontally with vertical snap indicator; drop persists order.
- [x] Double-click column title → inline rename; Enter/blur saves; Esc cancels.
- [x] Existing boards without `project.columns` load with the 4 defaults.
- [x] Card archive button hides card from board; data flag persists.
- [x] Archived modal lists hidden cards with Restore / Delete forever.
- [x] Sidebar Board ↔ CRM toggle; selected view persists per browser.
- [x] CRM rows can be added, edited inline, deleted; all fields sync via Firebase.
- [x] Status dropdown is color-coded.
- [x] Sort by column header click; text + status filter.
- [ ] (Manual) verify existing card drag-between-columns still works (regression check).

---

## File touch list

Only `C:\Users\cosmi\Projects\kanban-board\index.html`.

---

## Notes for Hermes

- The function `renderBoard()` at the section commented `RENDER` is the model to follow — same patterns for filter → render → wire.
- `saveState()` is `db.ref(DB_PATH).set(shared)`; calling it after any mutation persists + broadcasts.
- `uid()` is the shared id helper. `escapeHtml()` exists for any innerHTML use.
- The Firebase listener (`db.ref(DB_PATH).on("value", ...)`) re-runs `renderAll()` on every change, including your own writes — meaning if you mutate `shared.crm` and call `saveState()`, the CRM view will re-render via that path. Don't double-render unless something feels stale.
- When adding new render paths (renderCRM, renderArchived), don't forget to call them from `renderAll()` if their data may have changed.
