# Build Plan — Phase 4 (Maker Hub)

**Date:** 2026-06-21  
**Status:** Approved — ready to build  
**Preceded by:** Phase 3 (grams-based stocktake, dual thresholds, Buy tab redesign)  
**Architecture reference:** [docs/architecture-decisions.md](architecture-decisions.md)

---

## Scope

### In this build

**Data model — new schema (7 CSV files)**
- `inventory.csv` — expanded to 25 columns; 6-category system; Equipment, Library, Tools, location, status flags, task awareness
- `tasks.csv` — new file; maintenance tasks and care reminders for any item type
- `purchases.csv` — existing, field names updated; qty always in item's own unit (filament in grams directly)
- `notes.csv` — existing format; universal across all item types
- `stocktakes.csv` — `qty_counted_g` renamed to `qty_counted`; unit-agnostic
- `locations.csv` — new file; managed location list
- `wishlist.csv` — preserved as-is from Phase 3; new app does not write to it

**One-time Gist migration from Phase 3 schema**
- Filament (IDs 100–135): category = Consumables, unit = grams, name = composite string, colour_hex from hex
- Printer (ID 1): category = Equipment, nickname = "Printer 1"
- AMS (ID 50): category = Equipment, nickname = "AMS Lite 1", attached_to_id = 1
- eBox Lite (ID 1004): reclassify Accessories → Equipment
- All items: add location_id = null, needs_replacing = 0, want_to_buy = 0, added_date = today
- Phase 3 `notes` field on inventory → migrate to notes.csv entry
- `on_shopping_list` → `want_to_buy`; `low/critical_stock_threshold_g` → `low/critical_threshold`; `hex` → `colour_hex`
- `spool_weight_g` dropped; filament purchase qty now in grams directly
- Create empty tasks.csv, locations.csv with default locations

**6-category system (replaces filament-specific categories)**
- Equipment · Consumables · Accessories · Makers Supply · Tools · Library
- Category chips in Inventory tab (progressive disclosure — chip hidden until ≥ 1 item in category)
- All category-specific field visibility logic per category (see architecture document)

**Item detail + edit**
- Detail view: all fields, sections by category (Status · Notes · Tasks · Maintenance)
- Edit form: fields shown/hidden by category
- compatible_with picker with inline search (required for 7+ machines)
- Location picker (managed list from locations.csv)

**Notes system (universal)**
- All item types; timestamped log; add, edit, delete
- Auto-notes on flag changes (is_auto = 1, visually distinct)
- URL auto-detection → tappable hyperlinks

**Task system (universal)**
- All item types; tasks.csv; title, due date, optional recurrence
- Mark done (with optional backdating via date picker)
- Recurrence: next due = completed_date + recurrence_days

**Status flags**
- `needs_replacing` (Equipment + Tools only in UI) and `want_to_buy` (all types) as toggle chips in item detail Status section
- Auto-notes on set/clear

**Wishlist view**
- Inventory tab header toggle (★ star icon, nav bar right)
- Two sections: "Needs replacing" · "Want to buy"
- Running-low items NOT in Wishlist view (they stay in the alert chip, Inventory header)

**To Do tab**
- Unified task inbox; all item types; sections: Overdue · Due today · Upcoming
- iOS badge on tab showing count of due/overdue tasks
- Empty states: "No tasks yet" and "All caught up"

**Add item + log purchase flows**
- `+` action sheet: Log purchase / Import CSV / Add to inventory
- Category picker as first step for "Add to inventory"
- New item form (fields by category)
- Purchase form (updates purchases.csv and qty_on_hand)

**Equipment-specific features**
- Nickname field (matches physical sticker label)
- attached_to_id: inline picker in Equipment detail, not buried in edit form
- Maintenance section prominent in Equipment detail; same task system, different visual weight

**Library category**
- Catalogue-only: no qty, no thresholds, no stocktake
- Fields: format (physical/digital/both), storage_location, url, author, subject
- Format field controls field relevance: physical → storage_location; digital → url

**Settings**
- Gear icon in Inventory tab nav bar (right side, outermost)
- Locations sub-screen: managed list, add/rename/reorder/delete with confirmation always
- Stock thresholds sub-screen: ~12 sub-category Low/Critical defaults, "Reset to app defaults"
- App setup sub-screen: PAT token, Gist ID, Test connection, Disconnect (destructive)
- First launch / no Gist: inline prompt on first action, not separate onboarding

**Stocktake tab**
- Updated for new schema (unit-agnostic qty_counted, filament-only in Phase 1)
- All existing stocktake UX preserved

**History tab**
- Purchase log across all categories (not filament-only)
- Grouped by order_id

**CSV import/export**
- Export: Settings → Export; "All data" (ZIP) or "By category" (single CSV)
- Import: `+` action sheet or Settings → Import; category picker → template download → file picker → validation → preview → confirm
- Conflict handling: upsert by ID (match existing → UPDATE; no match → INSERT with auto-assigned ID)
- Per-row validation errors; no auto-notes on bulk import

**Legacy wishlist resolution**
- UI to resolve Phase 3 wishlist.csv entries: convert to Library item, flag as want_to_buy, or delete
- Prompted on first launch if wishlist.csv is non-empty

**Empty states — all screens**
- Type A (first launch/zero data): Inventory tab, History tab
- Type B (view-level empty): Wishlist view, To Do tab (no tasks / all done), Stocktake tab, category chip active
- Type C (no results): search, active filters
- Inline: Notes section ("No notes yet · Add note (optional)") · Tasks section ("No tasks · Add task")

### Out of scope (Phase 5+)

- Stocktake for non-filament consumables (fabric metres, vinyl metres)
- History tab filtering
- Offline handling and sync conflict resolution
- PWA / home screen icon
- Read-only sharing URL
- Advanced compatibility filtering ("show all accessories for machine X")
- Pull-to-refresh visual
- Copy-tasks-from-another-unit convenience (Equipment task template shortcut)

---

## Dependencies

| Dependency | Reason |
|---|---|
| Data model before any UI | New schema columns must exist before UI reads or writes them |
| Migration before UI testing | Cannot test with live data until Phase 3 data is migrated to Phase 4 schema |
| Category system before item cards | Card layout, field visibility, and section logic all depend on the category value |
| Notes system before status flags | Auto-note generation is triggered by flag changes — notes must exist first |
| Tasks system before To Do tab | To Do tab reads tasks.csv — must be populated |
| Status flags before Wishlist view | Wishlist view filters by needs_replacing and want_to_buy |
| Detail view before edit form | Tap card → detail → Edit; detail must exist before edit is reachable |
| Item detail before Add flow | Add item reuses item detail/edit form components |
| Locations.csv before location picker | Picker reads from locations.csv — file must exist with default data |
| Settings Locations screen before location picker | User must be able to manage locations before assigning them to items |
| compatible_with picker before detail/edit complete | Edit form is not complete without a working equipment picker |
| Gist sync update before migration | 7-file sync layer must handle all files before migration writes to them |

---

## Build Sequence

| Step | Work | Type |
|---|---|---|
| 1 | **Gist sync update** — extend sync layer to read/write all 7 CSV files (inventory, tasks, purchases, notes, stocktakes, locations, wishlist); maintain backward compatibility with Phase 3 field names during migration | Data ⚠️ |
| 2 | **Data migration — schema only** — implement new inventory.csv column set in code; map Phase 3 field names to Phase 4; write migration function (not executed yet) | Data ⚠️ |
| 3 | **Execute migration** — export full Phase 3 ZIP backup; run migration function against live Gist; create tasks.csv and locations.csv; verify all 36 filament items, Equipment records, and Accessories present with correct fields and qty | Data ⚠️ |
| 4 | **Category system + item cards** — implement 6-category model; rebuild item card for all categories (name, sub_category, location badge, status chips, qty display by unit, colour swatch where applicable); Inventory tab chip row with progressive disclosure | UI |
| 5 | **Item detail view** — all fields, section layout by category (Status · Notes · Tasks · Maintenance for Equipment); read-only; tap card → push navigation | UI |
| 6 | **Item edit form** — all fields editable; field visibility by category; compatible_with picker (multi-select Equipment list with inline search); location picker | UI |
| 7 | **Notes system** — add/edit/delete timestamped notes on any item; date picker (defaults to today, allows backdating); URL auto-detection; auto-note generation hook; inline empty state | UI |
| 8 | **Task system** — add/edit/delete tasks on any item; recurrence_days logic; mark done with optional backdating; To Do tab (unified inbox, Overdue · Due today · Upcoming sections, iOS badge); inline empty state; tab empty states | UI |
| 9 | **Status flags + Wishlist view** — needs_replacing / want_to_buy toggles in item detail Status section; auto-notes on set/clear; Wishlist view (★ toggle in Inventory nav bar, two-section layout); Wishlist empty state | UI |
| 10 | **Add item + Log purchase flows** — `+` action sheet (Log purchase / Import CSV / Add to inventory); category picker; new item form (fields by category); purchase form; qty_on_hand update on purchase | UI |
| 11 | **Equipment-specific features** — nickname display/edit; attached_to_id inline picker in Equipment detail; Maintenance section prominent layout in Equipment detail view | UI |
| 12 | **Library category** — format / storage_location / url / author / subject fields in detail and edit; catalogue-only behaviour (no qty section, no threshold section, no stocktake) | UI |
| 13 | **Settings** — gear icon in Inventory tab nav bar; Locations sub-screen (full managed list UX); Stock thresholds sub-screen (~12 rows, tap to edit, reset to defaults); App setup sub-screen (PAT, Gist ID, test connection, disconnect) | UI |
| 14 | **Inventory tab: search + filter panel** — search field (suspends chip filter); Filters button → filter panel (location, status, compatibility); empty states (search / filter no results) | UI |
| 15 | **Stocktake tab update** — rename qty_counted_g → qty_counted; preserve all existing Stocktake UX | UI |
| 16 | **History tab update** — all categories (not filament-only); grouped by order_id | UI |
| 17 | **CSV import/export** — Export (Settings → Export, ZIP or single CSV, JS blob download); Import (action sheet + Settings → Import, category picker, template download, file picker, validation, preview, confirm, upsert by ID) | UI |
| 18 | **Legacy wishlist resolution** — display Phase 3 wishlist.csv entries on first launch if non-empty; per-entry actions: convert to Library item / flag as want_to_buy / delete | UI |
| 19 | **Empty states — all screens** — implement all Type A / B / C empty states; verify inline states in Notes and Tasks sections | UI |
| 20 | **Full test pass** — end-to-end on iPhone via GitHub Pages URL; verify Gist sync, all categories, all tabs, Settings, migration data integrity, CSV import/export round-trip, empty states, destructive action confirmations | Test |

---

## Risk Flags

### 🔴 High — Gist data migration (irreversible)
Transforming Phase 3 schema to Phase 4 cannot be undone once written to Gist.  
**Mitigation:** Export a full Phase 3 ZIP backup at the start of step 3. Verify all item counts and filament quantities match Phase 3 before proceeding to step 4. Keep backup accessible.

### 🔴 High — Schema change while Phase 3 app is live
If Phase 4 migration runs and then Phase 4 build is not complete, the Phase 3 app will misread the migrated data.  
**Mitigation:** Complete steps 1–3 and step 4 (category system + item cards) before the migrated Gist is used in the browser. Do not execute migration (step 3) until the new UI is ready to read it.

### 🔴 High — Gist sync update (7 files)
Extending the sync layer from 4 to 7 CSV files. If sync is broken, the app cannot read or write any data.  
**Mitigation:** Implement step 1 (sync update) and test read/write of all files before touching any data. Maintain the Phase 3 app in a separate branch until step 4 is stable.

### 🟡 Medium — qty_on_hand dual calculation model
Filament qty = calculated from stocktake + purchases. All other stocked items: stored directly in qty_on_hand. Two models in one system — easy to apply the wrong model to the wrong item type.  
**Mitigation:** Implement a single `calcQty(item)` function that branches on item unit/category. All qty reads go through this function. No inline qty calculations in UI components.

### 🟡 Medium — compatible_with picker at 7+ machines
The compatible_with picker must include inline search when the Equipment list reaches 7+ items. Missing this makes the picker unusable with a full sewing machine inventory.  
**Mitigation:** Implement inline search in the picker from step 6. Test with 10 dummy Equipment records to verify usability.

### 🟡 Medium — Auto-note generation loop risk
Auto-notes are triggered by flag changes. If any code path triggers a flag change while processing a note save, it could create an infinite loop.  
**Mitigation:** Auto-note generation must be a fire-and-forget write to notes.csv only. It must not trigger any event that could re-invoke flag change logic.

### 🟡 Medium — CSV import upsert collision
Rows with an ID not found in existing inventory are INSERT with app-assigned ID. If a user provides an ID in the import CSV that happens to match an existing record they didn't intend to update, data will be silently overwritten.  
**Mitigation:** Show all UPDATE rows in the import preview (not just count). Give the user a chance to inspect which existing records will be modified before confirming.

### 🟢 Low — Phase 3 wishlist.csv resolution
Phase 3 wishlist entries are not automatically migrated — they require user action. If the user dismisses the resolution prompt without acting, wishlist.csv retains orphaned entries.  
**Mitigation:** Re-prompt on subsequent launches until wishlist.csv is empty or the user explicitly dismisses permanently ("Don't show again").
