# Build Plan — Phase 3

**Date:** 2026-06-19  
**Status:** Approved — ready to build  
**Preceded by:** Phase 2 (colour family filter)

---

## Scope

### In this build

**Data model**
- Stocktake table: `qty_counted` (spools) replaced by `qty_counted_g` (grams)
- Inventory table: `low_stock_threshold` replaced by `low_stock_threshold_g` and new `critical_stock_threshold_g` (both per-filament, editable in edit form)
- Inventory table: new `archived` boolean field
- Quantity formula: `qty on hand = last stocktake (g) + (spools purchased since × spool_weight_g)`

**One-time Gist migration from stocktake_20260619.csv**
- Stocktake CSV: clear all old spool-based records, create 30 new grams-based records dated 2026-06-19
- Inventory CSV: update colour_family, hex, low_stock_threshold_g, critical_stock_threshold_g per row
- Purchases CSV: untouched

**Stocktake tab redesign**
- Layout (top to bottom): heading + instructions → action row (Download fillable CSV · Import CSV) → filter chips + search → scrollable list → data panels
- Input shows current calculated qty as greyed-out placeholder
- Supports simple maths in input — `100+200+300` resolves to `600` on Enter
- Enter key: resolve maths → save to Gist immediately → update placeholder → dismiss keyboard
- Filter chips and search row identical to inventory tab
- "Download fillable stocktake.csv" pre-fills id, colour, brand, type; user fills qty_on_hand_g and stocktake_date (pre-filled with today)
- Import validation: skip rows with unrecognised id, show warning listing skipped rows
- "Confirm stocktake" button removed — Enter key is the save action
- Data panels redesigned: label on row 1, grey buttons on row 2, orange button on row 3

**Filter bar + header chip**
- Status chip removed; replaced by Brand chip (dynamically built from inventory)
- Contextual Low/Critical chip appears in header where current badge is:
  - Only low stock items → single-tap chip, instantly filters to low stock
  - Only critical items → single-tap chip, instantly filters to critical
  - Both → chip with dropdown
  - Neither → chip hidden entirely
- Chip visible on all tabs; tapping always navigates to inventory tab, clears all active filters, shows all low/critical items

**Inventory tab changes**
- Cart icon hidden on low/critical items (badge is sufficient)
- Cart icon visible and tappable on Good items; filled when item is on the shopping list
- "+" Add button opens add filament form (new filament types only)
- "+ Buy" button removed from inventory cards
- Archive: toggle in edit form; "Show archived" toggle at bottom of inventory list reveals archived filaments greyed out with edit button still visible
- Critical and Low badges remain on inventory cards

**Edit form changes**
- `low_stock_threshold_g` and `critical_stock_threshold_g` fields replace single threshold field
- Archive toggle added

**Buy tab redesign (renamed from "List")**
- Three sections:
  1. **Running low** — Critical items first, then Low; auto-populated from stocktake calculations
  2. **Want to buy** — filaments manually added via cart icon
  3. **Wishlist** — quick capture for filaments not yet owned
- Duplicate items appear once only, in the highest priority section
- Each card shows: colour swatch, colour name, brand · type · grams on hand, status badge (Low / Critical / Good), ✕ remove button (manually added items only)
- Tapping anywhere on a card except the ✕ button opens the edit form for that filament
- Cart icon on inventory cards shows filled for manually added items only; auto items (low/critical) have no cart icon
- Empty state: "All stocked up" with brief explanation
- "Copy list to clipboard" button includes all sections
- "+ Add to wishlist" button in Wishlist section opens short form: name, filament type, notes
- Wishlist items have ✕ remove button; no "Move to buy" in this build
- When a wishlist item arrives: use + button or purchases CSV to add to inventory, then ✕ to remove from wishlist

**Navigation + general**
- App always opens on inventory tab regardless of last-used tab
- Tab rename and reorder: Filament · Buy · Stocktake · History

### Out of scope (Phase 4+)

- Full wishlist management (move to buy, link to inventory record)
- Grouping by supplier on Buy tab
- Archive usage rate and detail view
- Pull-to-refresh with green confirmation bar
- PWA / home screen icon
- Read-only sharing URL
- Contributor access

---

## Dependencies

| Dependency | Reason |
|---|---|
| Data model changes before any UI | qty_counted_g, threshold fields, archived field must exist before UI reads them |
| Quantity formula before display | New formula must be correct before inventory cards, Buy tab, and stocktake tab show quantities |
| Gist migration after formula | Importing stocktake_20260619.csv uses the new grams formula — wrong order = wrong numbers |
| status() function before chips | Low/Critical header chip and cart icon visibility both depend on updated status() using new thresholds |
| Archive field before archive UI | Toggle and "show archived" filter need the field to exist first |
| Edit form changes before Buy tab | Tapping a Buy card opens the edit form — thresholds and archive must be there first |

---

## Build Sequence

| Step | Work | Type |
|---|---|---|
| 1 | Data model + formula — add qty_counted_g, threshold fields, archived field; update quantity formula; handle old field name as fallback | Data ⚠️ |
| 2 | Gist migration — import stocktake_20260619.csv; clear old stocktake records; create 30 new grams-based records; update inventory fields; verify quantities | Data ⚠️ |
| 3 | Edit form — add low/critical threshold fields, archive toggle, remove single threshold field | UI |
| 4 | Stocktake tab redesign — filter chips, grams inputs, maths on Enter, auto-save to Gist, download fillable CSV, import validation, remove Confirm button, data panels layout | UI |
| 5 | Filter bar + header chip — replace Status chip with Brand chip; add contextual Low/Critical header chip | UI |
| 6 | Inventory tab — cart icon logic, remove + Buy button, archive toggle, show archived, app always opens on inventory tab | UI |
| 7 | Buy tab redesign — three sections, card layout, tap to edit, empty state, copy to clipboard, wishlist quick-add form | UI |
| 8 | Tab rename + reorder + data panels — Filament · Buy · Stocktake · History; data panels label/buttons/action layout | UI |
| 9 | Full test pass — end-to-end on iPhone using GitHub Pages URL; verify Gist sync, quantities, all features, edge cases | Test |

---

## Risk Flags

### 🔴 High — Gist data migration (irreversible)
Clearing old stocktake records and updating inventory fields cannot be undone once synced to Gist.  
**Mitigation:** Export a full ZIP backup before running the migration. Verify quantities in the app before proceeding to step 3.

### 🔴 High — Quantity formula change (mixed data risk)
Old stocktake records store spools; new records store grams. If any old records survive migration, the formula will produce wrong numbers.  
**Mitigation:** Clear ALL old stocktake records in step 2. No partial migration.

### 🟡 Medium — Math expression evaluation (security)
Evaluating user-typed expressions like `100+200+300` in JavaScript. Using `eval()` is a security risk.  
**Mitigation:** Use a safe parser that only allows digits and arithmetic operators (`+`, `–`, `*`, `/`). No `eval()`.

### 🟡 Medium — Cart icon state complexity
Cart icon has multiple states: hidden (low/critical), visible unfilled (good, not on list), visible filled (manually added). Easy to get conditions wrong.  
**Mitigation:** Implement as a single function with clearly ordered conditions. Test each state explicitly.

### 🟡 Medium — Threshold field rename
Renaming `low_stock_threshold` to `low_stock_threshold_g`. Existing Gist inventory.csv uses the old field name.  
**Mitigation:** Migration reads both old and new field names. Today's CSV import writes the new names.

### 🟢 Low — Import CSV validation
Unrecognised IDs in stocktake CSV import could silently skip rows.  
**Mitigation:** Show explicit warning listing any skipped rows after import.
