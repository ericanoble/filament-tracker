# Maker Hub — Architecture Document

**Status:** Confirmed — ready for Phase 1 implementation  
**Last updated:** 2026-06-21  
**Scope:** Full Maker Hub (Phase 1 = 3D printing + infrastructure for all activities)

---

## 1. Product Scope

**Maker Hub** is a personal inventory management app for multi-discipline maker workflows. It replaces the Phase 3 filament tracker with a generalised system designed to support all current and planned maker activities without requiring structural changes as activities are added.

**Current activities:** 3D printing (Bambu A1 mini + AMS Lite), textiles/sewing (7 machines), Cricut cutting  
**Planned activities:** Bambu H2D (laser engraving/cutting/3D printing)  
**Phase 1 deliverable:** Full Maker Hub infrastructure with existing 3D printing data migrated. All 6 categories available in UI. Other activity data added by user as needed post-launch.

**Technical constraints:**
- Single `index.html`, no build step, GitHub Pages hosting
- Data stored in a private GitHub Gist via API (PATCH/GET)
- GitHub PAT stored in localStorage
- No backend, no user accounts

---

## 2. Design Principles

### Schema-first, UI-second
The data model is designed for the full Maker Hub scope on day one. Phase 1 UI exposes all 6 categories regardless of which are populated. Adding a new phase (Cricut, textiles, H2D) adds user data, not new data structures.

### Categories defined by management behaviour
Each category is defined by what the app *does differently* with items in it — not by what the item physically is. The defining question for each category determines app behaviour (stock alerts, thresholds, maintenance, qty tracking). The category field — not the ID number — is the sole determinant of how an item is managed.

### One question per category
Each category has a single deciding question. If the answer is yes, the item belongs there. No judgment calls.

### Opaque IDs
IDs are globally unique integers with no semantic meaning. They are never reassigned or reused. The category field alone determines what something is. (Replaces the Phase 3 ID-range-encodes-category approach.)

### DRY — one system per concept
When two features are structurally identical (e.g., Equipment maintenance task vs Tool care reminder), build one system, not two. The universal task system (`tasks.csv`) applies to all item types without exception.

### Progressive disclosure
Category chips only appear when a category has ≥ 1 item. Library, Tools, and other empty categories are hidden until first use. Settings toggles let the user force-show a category during a bulk entry session.

### Destructive actions always confirm
No destructive action executes without a confirmation step. Item count or status affects the *copy* of the confirmation, not whether the confirmation appears.

### Sticker-nickname identity anchor
The physical sticker label on a piece of equipment must match the `nickname` field in the app. This pairing is the source of truth for physical identity. Maintenance logs, notes, and compatibility data stay correctly attributed to the right physical unit regardless of where it is currently located or what it is connected to.

---

## 3. Technology Stack

- **Runtime:** Vanilla JS, single `index.html`
- **Libraries:** PapaParse 5.4.1 (CSV), JSZip 3.10.1 (ZIP export), Tabler Icons webfont, Space Grotesk + JetBrains Mono
- **Storage:** GitHub Gist API — 7 CSV files in one private Gist
- **Hosting:** GitHub Pages

---

## 4. Information Architecture

### Tab bar (4 tabs)

```
[ Inventory ]  [ To Do ]  [ Stocktake ]  [ History ]
```

**Inventory tab**
- Navigation bar (right): [★ Wishlist toggle] [⚙ Settings]
- Below nav bar: search field
- Chip row: category chips (progressive — chip hidden until ≥ 1 item in category)
- Filter panel: accessible via "Filters" button in header — covers location, status, compatibility
- Item list: tapping a card → detail view (push navigation)
- Wishlist toggle: switches between All Items and Wishlist views
  - Wishlist view: two sections — "Needs replacing" (urgent, first) and "Want to buy"

**To Do tab**
- Unified task inbox — all outstanding tasks from all item types across all categories
- iOS badge on tab icon shows count of tasks due or overdue
- Sections: Overdue · Due today · Upcoming
- Tapping a task → item detail view for that item

**Stocktake tab**
- Phase 1: filament / weight-based only
- Phase 2: extended to fabric metres, vinyl metres when textiles phase launches

**History tab**
- Purchase log, all item types, grouped by order
- Phase 1: no filtering
- Phase 2: filtering by date, category, supplier

### Navigation model
- Primary: tab bar (peer sections)
- Within tabs: push navigation (list → detail → edit)
- Settings: push from Inventory tab nav bar gear icon
- Modals: Add item flow, confirmation sheets, date pickers

### Settings information architecture

```
Settings
├── Locations          → managed list (add, rename, reorder, delete with confirmation)
├── Stock thresholds   → sub-category Low / Critical defaults, "Reset to app defaults"
└── App setup
    ├── GitHub Gist    → PAT token, Gist ID, Test connection
    └── Disconnect     → destructive — clears stored credentials (confirmation required)
```

Settings is accessed by tapping the gear icon in the Inventory tab nav bar. Opens as a push (not modal) to support sub-screen navigation.

**First launch / Gist not configured:** No separate onboarding screen. If the user triggers an action requiring Gist access before setup is complete, the app prompts inline.

---

## 5. Category System

Six categories, each defined by one question:

| Category | Defining question | Key behaviours |
|---|---|---|
| **Equipment** | Does it need maintenance and never run out? | Maintenance tasks · nickname · attached_to · no stock alerts · no qty |
| **Consumables** | Does it get depleted and need restocking? | Stock levels · thresholds · stocktake · purchase history · unit type |
| **Accessories** | Does it attach to or work with specific equipment? | `compatible_with` required · stock count · want_to_buy · purchase history |
| **Makers Supply** | Is it small hardware bought in bulk and counted? | Count · bulk purchase → individual qty · low threshold alerts |
| **Tools** | Is it a hand or powered tool used across activities? | Location essential · no stock alerts · no thresholds · tasks optional |
| **Library** | Is it reference material you consult, not consume? | Catalogue only · no stock management · title/author/format/URL/notes |

### Category classification: borderline items

| Item | Category | Reasoning |
|---|---|---|
| 3D printer, sewing machine, Cricut, Bambu H2D | Equipment | Scheduled maintenance, significant purchase, service log needed |
| AMS Lite, filament dryer (eBox Lite) | Equipment | Powered, maintenance-worthy, attached to primary Equipment |
| Filament, resin, fabric, thread, vinyl, IPA | Consumables | Depleted and restocked, stock alerts apply |
| Nozzles, build plates, Cricut blades, Cricut mats, needles, presser feet, bobbins | Accessories | Machine-specific, compatible_with required |
| Iron, EasyPress, heat gun, soldering iron | Tools | No scheduled maintenance, ad hoc care only |
| Fabric scissors (sharpening schedule) | Tools | Sharpening reminder = a task, not a maintenance schedule. Category = Tools; schedule via task system. |
| Rotary cutter handle | Tools | Never consumed; companion blades are Accessories |
| Rotary blades | Accessories | Replaceable/sharpenable; stock count applies |
| Craft knife, tweezers, ruler, pliers, cutting mat | Tools | Location is the key attribute |
| M3 screws, magnets, springs, buttons, zippers | Makers Supply | Bought in bulk, counted individually |
| Quilting books, sewing patterns, STL files, SVG files, PDF patterns | Library | Consulted, not consumed |
| Self-healing cutting mat (non-machine-specific) | Tools | Not machine-specific; location is the key attribute |
| Cricut cutting mat (LightGrip, StandardGrip, etc.) | Accessories | Cricut-specific, compatible_with required, wears out and is replaced |

### Compatible_with

`compatible_with` references `inventory_id` values of Equipment records, or the literal string `universal`.

- Items spanning all activities: `compatible_with = universal` (IPA, adhesives, calipers, ruler, reference books)
- Items specific to machines: comma-separated inventory IDs (e.g. `"1,50"`)
- Compatibility picker: shows Equipment records as "Model — Nickname". Includes inline search for lists of 7+ items.

Soft-delete (`archived = 1`) is used instead of hard delete for all records. Compatibility references remain valid because Equipment records are never truly deleted.

### Location system

`location_id` on every item references a managed list in `locations.csv`. Location and `compatible_with` are orthogonal:
- `compatible_with` = what activities/machines this item works with
- `location_id` = where this physical instance currently lives

Current locations: 3D printing station · Sewing room · Cricut bench · Electronics station · Shed

Items with no location: appear in all views; no location badge shown on card.

---

## 6. Equipment Records

**One record per physical unit.** Two identical printers = two separate Equipment records, each with its own maintenance history.

**Fields specific to Equipment:**
- `nickname` — must match physical sticker label on the unit. Source of truth for physical identity.
- `attached_to_id` — self-referential FK to the primary Equipment record this peripheral is connected to. Last-known-state snapshot — not an authoritative live relationship. Update inline from the Equipment card; do not build logic that depends on this being current.

**Maintenance tasks:** Per-unit, from scratch. Stored in `tasks.csv` with `inventory_id` referencing the specific Equipment record. (Phase 2 consideration: copy-tasks-from-another-unit convenience feature.)

**Not tracked on Equipment:** qty, unit, thresholds, print hours, firmware version.

---

## 7. Universal Task System

Any item in any category can have tasks. One data model, one UI pattern, one inbox (To Do tab).

**Task = a title, a due date, an optional recurrence interval.**

Equipment detail view: Tasks section is prominent (machines have many tasks; service history matters).  
All other item detail views: Tasks section is lightweight — shows inline empty state "No tasks · Add task" until populated.

Mark done action sheet: "Mark done today · [date]" / "Mark done on another date…" / Cancel. Date picker defaults to yesterday for backdated completions.

---

## 8. Interaction Design Patterns

### Empty states

Three types:

**Type A — Zero data / first launch** (welcoming, always has CTA)

| Screen | Headline | Body | CTA |
|---|---|---|---|
| Inventory tab (no items) | "Your inventory is empty" | "Add items to track your maker supplies, equipment, and materials." | "Add your first item" |
| History tab | "No purchase history" | "Log a purchase against any item to track your spending here." | none |

**Type B — View-level empty** (informative, CTA only when clear next action exists)

| Screen | Headline | Body | CTA |
|---|---|---|---|
| Wishlist view (nothing flagged) | "Your wishlist is empty" | "Open any item and tap 'Want to buy' to add it here." | "Browse inventory" |
| To Do (no tasks anywhere) | "No tasks yet" | "Open any item to add a maintenance task or care reminder." | none |
| To Do (all complete) | "All caught up" | "No tasks due. Check back later." | none |
| Stocktake (no consumables) | "Nothing to stocktake" | "Add consumable items like filament or resin to your inventory." | none |
| Category chip active, no items | "No [Category] items" | "Add [category] items to your inventory and they'll appear here." | none |

**Type C — No results** (helpful guidance, no CTA button except "Clear filters")

| Screen | Headline | Body | CTA |
|---|---|---|---|
| Search, no results | "No results for '[query]'" | "Try a different search term." | none |
| Filters active, no results | "No items match your filters" | "Try adjusting or clearing your filters." | "Clear filters" |

**Inline empty states** (within item detail, not full-screen):
- Notes section: "No notes yet · Add note (optional)"
- Tasks section: "No tasks · Add task"
- Tasks section appears on ALL item types — consistent behaviour, lightweight enough not to intrude on items that rarely have tasks.

**Visual spec:** SF Symbols icon (~40pt, system secondary colour) · headline (.title3) · body (.subheadline, secondary colour) · CTA (.tinted button style, never .filled)

### Quantity management

- **Adjust button** on item card: opens –1 / +1 / Set picker. Sets `qty_on_hand` directly for count-based items.
- **Stocktake** (Stocktake tab): weight-based entry for filament. qty = last stocktake + purchases since.
- **Log purchase** (item detail or `+` action sheet): records to `purchases.csv`, updates `qty_on_hand`.

### Status flags

Two boolean flags on inventory records:

| Flag | UI location | Available on | Behaviour |
|---|---|---|---|
| `needs_replacing` | Item detail → Status section | Equipment, Tools | Auto-note on set/clear. Shows in Wishlist view "Needs replacing" section. |
| `want_to_buy` | Item detail → Status section | All item types | Auto-note on set/clear. Shows in Wishlist view "Want to buy" section. |

An item with both flags: appears in "Needs replacing" section only (not duplicated).

Running-low (threshold-based, automatic) is surfaced by the alert chip in the Inventory tab header — not in the Wishlist view.

### Destructive actions

All destructive actions require a confirmation step without exception. Item count or status affects the confirmation copy, not whether confirmation appears.

Standard confirmation buttons: "Keep it" / "Delete" (or context-appropriate equivalents).

### Notes

- Log-style: multiple timestamped entries per item, all item types
- Date defaults to today, tappable → date picker (allows backdating)
- Long-press → Edit or Delete (with confirmation)
- Auto-detected URLs rendered as tappable hyperlinks
- Auto-generated entries (`is_auto = 1`) visually distinct but fully editable/deletable
- 2–3 most recent entries shown; "Show all (N)" to expand

### Add item flow

`+` action sheet: "Log purchase" / "Import CSV" / "Add to inventory"

### CSV import/export

**Export:** Settings → Export → "All data" (ZIP of all 7 CSV files) or "By category" (single CSV). JavaScript blob download → iOS Files app → AirDrop / iCloud Drive to laptop.

**Import:** `+` action sheet ("Import CSV") or Settings → Import. Flow: category picker → "Download blank template" → file picker (iOS Files) → validation → preview ("23 items: 18 new, 5 will be updated") → confirm.

**Conflict handling — upsert by ID:**
- `id` matches existing record → UPDATE
- No `id` or `id` not found → INSERT with app-assigned auto-increment ID

**Validation:** per-row error messages. No auto-notes on bulk import. Purchases not imported via this flow.

---

## 9. Data Specification

### `inventory.csv`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | integer | NOT NULL | auto-increment | PK. Global. Phase 3 IDs preserved. |
| `name` | text | NOT NULL | — | Composite searchable string |
| `category` | text | NOT NULL | — | `Equipment` · `Consumables` · `Accessories` · `Makers Supply` · `Tools` · `Library` |
| `sub_category` | text | nullable | null | |
| `unit` | text | nullable | null | `count` · `grams` · `ml` · `metres`. Null for Equipment, Library. |
| `qty_on_hand` | real | nullable | null | Null for Equipment, Library. Filament: calculated. Others: stored. |
| `low_threshold` | real | nullable | null | Null = inherit sub-category default |
| `critical_threshold` | real | nullable | null | Null = inherit sub-category default |
| `compatible_with` | text | nullable | null | Comma-separated inventory IDs or `universal` |
| `location_id` | integer | nullable | null | FK → `locations.csv.id` |
| `colour_hex` | text | nullable | null | 6-char hex (no `#`). Filament, thread, vinyl, fabric. |
| `colour_family` | text | nullable | null | E.g. "Black", "Red" |
| `nickname` | text | nullable | null | Equipment only (UI). Matches physical sticker. |
| `attached_to_id` | integer | nullable | null | Self-ref FK → `inventory.csv.id`. Equipment peripherals. Last-known-state. |
| `needs_replacing` | integer | NOT NULL | 0 | Boolean 0/1 |
| `want_to_buy` | integer | NOT NULL | 0 | Boolean 0/1 |
| `archived` | integer | NOT NULL | 0 | Boolean 0/1. Soft delete. |
| `format` | text | nullable | null | Library only: `physical` · `digital` · `both` |
| `storage_location` | text | nullable | null | Library physical only. Free text. |
| `url` | text | nullable | null | Library digital. Tappable. |
| `author` | text | nullable | null | Library only |
| `subject` | text | nullable | null | Library only |
| `purchase_date` | text | nullable | null | ISO 8601: `YYYY-MM-DD` |
| `added_date` | text | NOT NULL | today | ISO 8601: `YYYY-MM-DD`. Auto-set, never changed. |

### `tasks.csv`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | integer | NOT NULL | auto-increment | PK |
| `inventory_id` | integer | NOT NULL | — | FK → `inventory.csv.id` |
| `title` | text | NOT NULL | — | |
| `due_date` | text | nullable | null | ISO 8601: `YYYY-MM-DD` |
| `recurrence_days` | integer | nullable | null | Next due = `completed_date` + `recurrence_days` |
| `completed_date` | text | nullable | null | ISO 8601: `YYYY-MM-DD`. Most recent completion. |
| `notes` | text | nullable | null | How-to or completion notes |

### `purchases.csv`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | integer | NOT NULL | auto-increment | PK |
| `inventory_id` | integer | NOT NULL | — | FK → `inventory.csv.id` |
| `purchase_date` | text | NOT NULL | — | ISO 8601: `YYYY-MM-DD` |
| `qty_purchased` | real | NOT NULL | — | In item's own unit. Filament: grams. |
| `unit_price` | real | nullable | null | Price per unit |
| `order_id` | text | nullable | null | |
| `supplier` | text | nullable | null | |
| `notes` | text | nullable | null | |

### `notes.csv`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | integer | NOT NULL | auto-increment | PK |
| `inventory_id` | integer | NOT NULL | — | FK → `inventory.csv.id` |
| `timestamp` | text | NOT NULL | — | ISO 8601: `YYYY-MM-DDTHH:MM:SS` |
| `note_text` | text | NOT NULL | — | URLs auto-detected and rendered tappable |
| `is_auto` | integer | NOT NULL | 0 | Boolean 0/1. System-generated. |

### `stocktakes.csv`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | integer | NOT NULL | auto-increment | PK |
| `inventory_id` | integer | NOT NULL | — | FK → `inventory.csv.id` |
| `stocktake_date` | text | NOT NULL | — | ISO 8601: `YYYY-MM-DD` |
| `qty_counted` | real | NOT NULL | — | In item's own unit. Same-day tiebreak: highest `id` wins. |
| `notes` | text | nullable | null | |

### `locations.csv`

| Column | Type | Null | Default | Notes |
|---|---|---|---|---|
| `id` | integer | NOT NULL | auto-increment | PK |
| `name` | text | NOT NULL | — | E.g. "3D printing station" |
| `display_order` | integer | NOT NULL | 0 | User-reorderable in Settings |

### `wishlist.csv` (legacy)

Phase 3 file. New app does not write to it. User resolves each entry via UI (convert to Library item, flag as `want_to_buy` on existing inventory, or delete). File becomes empty once all entries resolved.

---

## 10. Phase Boundaries

### Phase 1 (this build)

All 6 categories available. 3D printing inventory migrated from Phase 3. App infrastructure complete.

Includes: full schema · migration script · all 6 categories in UI · all 4 tabs · Settings · Equipment maintenance · Library catalogue · universal task system · Notes · status flags + Wishlist view · location system · CSV import/export · empty states · compatible_with visible on cards and pickers.

### Explicitly deferred to Phase 2

- Stocktake for non-filament consumables (fabric metres, vinyl metres) — Stocktake tab remains weight/filament-only in Phase 1
- History tab filtering
- Offline handling and conflict resolution
- PWA / home screen icon
- Read-only sharing URL
- Advanced compatibility filtering ("show all accessories for machine X")
- Pull-to-refresh visual
- Copy-tasks-from-another-unit convenience (Equipment task template shortcut)

**Hooks to leave in Phase 1 code:**
- `unit` field on `stocktakes.csv` — already unit-agnostic (ready for fabric/vinyl stocktake)
- `compatible_with` field and picker — already built for multi-machine; advanced filter is a UI addition only
- Library chip — already in progressive disclosure chip row; appears when first Library item added

---

## 11. Data Migration: Phase 3 → Maker Hub

Automated migration script. Runs once on first launch if Phase 3 schema is detected in Gist.

### `inventory.csv` transformations

| Phase 3 record | Action |
|---|---|
| Filament (IDs 100–135) | `category` = Consumables · `unit` = grams · IDs preserved · `name` = composite of brand/family/type/colour · `colour_hex` = from `hex` field |
| Printer (ID 1) | `category` = Equipment · `nickname` = "Printer 1" |
| AMS (ID 50) | `category` = Equipment · `nickname` = "AMS Lite 1" · `attached_to_id` = 1 |
| eBox Lite (ID 1004) | `category` = Equipment (was Accessories) |
| All other accessories (1000–1999) | `category` = Accessories |
| Makers Supply (3000–3003) | `category` = Makers Supply |
| All items | Add: `location_id` = null · `needs_replacing` = 0 · `want_to_buy` = 0 · `added_date` = today |
| `on_shopping_list` | → `want_to_buy` |
| `low_stock_threshold_g` | → `low_threshold` |
| `critical_stock_threshold_g` | → `critical_threshold` |
| `notes` field (if populated) | Migrate to `notes.csv` as a single entry with `timestamp` = today, `is_auto` = 0 |
| `hex` | → `colour_hex` |
| `spool_weight_g` | Dropped — purchases now record qty in grams directly |

### Other files

- `purchases.csv`: preserved, field names updated (`qty_purchased` already exists — verify unit is grams for filament)
- `notes.csv`: preserved as-is (already in new format from Phase 3)
- `stocktakes.csv`: `qty_counted_g` → `qty_counted`
- `wishlist.csv`: preserved, not transformed (user resolves manually)
- `tasks.csv`: created empty
- `locations.csv`: created with Phase 3 default locations (3D printing station · Sewing room · Cricut bench · Electronics station · Shed)

---

## 12. Standing Constraints

### Database migration readiness

The CSV/Gist data model is designed so that migrating to a relational database (e.g. Supabase/PostgreSQL) requires minimal changes.

- **One entity type per CSV file.** Each CSV maps cleanly to one future database table.
- **Consistent foreign keys.** All cross-file relationships use `inventory_id` (or explicit `_id` suffix columns). No implicit joins.
- **No logic in column names.** Category, hierarchy, and relationships expressed as row values, not encoded in column names.
- **Compatibility references use inventory IDs** (not hardcoded model names). Valid because Equipment records use soft delete (`archived`) — they are never truly deleted, so references never become invalid.
- **Thin data access layer.** No business logic coupled to the Gist transport. Swapping `fetch(gist_url)` for `fetch(supabase_url)` is the primary change required to migrate to a database backend.

### Compatible_with: inventory IDs, not model strings

Compatibility references `inventory_id` values of Equipment records (e.g. `"1,50"`), not hardcoded model name strings (e.g. `"A1 mini"`). This means:
- Adding a second A1 mini creates a new Equipment record; accessories are compatible with the specific physical unit, not the model class.
- Compatibility picker shows "Model — Nickname" (e.g. "Bambu A1 mini — Printer 1") so the user sees physical identity.
- Compatibility picker includes inline search for Equipment lists of 7+ items.

### ID scheme

IDs are opaque, globally auto-incrementing integers. Existing Phase 3 IDs (max = 3003) preserved. New items: 3004, 3005… regardless of category. The category field — not the ID range — determines how an item is managed.

### Filament qty calculation

`qty_on_hand` for filament = qty at last stocktake + sum of `purchases.qty_purchased` since that stocktake date. Same-day tiebreak on stocktakes: highest `id` wins (most recent entry). All other stocked items: `qty_on_hand` stored directly.

---

## 13. Open Questions (Phase 2)

**Maintenance task templates (Equipment):** For Phase 1, tasks are per-unit from scratch (Option B — simple to build, user only has one A1 mini). Phase 2 consideration: add a "copy tasks from another unit of the same model" shortcut (Option C approach) to reduce setup burden when adding a second machine.

**Dynamic low-stock threshold for vacuum bags (ID 1006):** Could be derived from current total filament spool count (since each spool is stored in one bag). Phase 2 consideration if threshold alerts prove imprecise for this item.
