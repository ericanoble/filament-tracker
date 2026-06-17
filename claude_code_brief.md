# Spool Tracker — Project Brief for Claude Code

## What we're building
A filament inventory tracking web app for a 3D printing enthusiast. Deployed as a **single `index.html` file on GitHub Pages**, with data stored in **three CSV files** in the same repo. No backend, no login, no localStorage — the CSVs are the source of truth.

---

## Repo structure
```
/
├── index.html          ← the entire app (HTML + CSS + JS)
├── inventory.csv       ← filament types catalogue + current qty
├── purchases.csv       ← every purchase event (40 rows to date)
└── stocktakes.csv      ← physical count overrides (currently header-only)
```

---

## Data schemas

### inventory.csv
```
id, brand, family, type, colour, hex, spool_weight_g, qty_on_hand, low_stock_threshold, on_shopping_list, notes
```
- 30 rows of real data already generated
- `id` is the primary key (integer, 1–30)
- `family` = broad category (PLA, PETG, TPU, Silk PLA)
- `type` = specific variant (PLA+ HS, Matte, standard, 95A, Glow-In-Dark, Mystic Triple Colour)
- `hex` = colour code for the UI swatch
- `qty_on_hand` = current calculated stock (see logic below)
- `low_stock_threshold` = alert when qty drops to this level (default 1)
- `on_shopping_list` = true/false

### purchases.csv
```
id, inventory_id, purchase_date, qty_purchased, order_id, supplier, style, notes
```
- 40 rows of real data already generated
- `inventory_id` is the foreign key to inventory.csv
- `style` = "spool" or "refilament" (paper tube refill, no plastic hub)
- `purchase_date` = YYYY-MM-DD
- Each row represents one physical spool purchased (qty_purchased = 1 always so far)

### stocktakes.csv
```
id, inventory_id, stocktake_date, qty_counted, notes
```
- Currently header-only, ready to populate
- Used as anchor points for qty calculation (see logic below)

---

## qty_on_hand calculation logic
This is the core business logic. For each inventory item:

1. Find the **most recent stocktake** for that `inventory_id` → gives `baseline_qty` and `baseline_date`
2. Sum `qty_purchased` from `purchases.csv` WHERE `inventory_id` matches AND `purchase_date > baseline_date`
3. Sum `qty_used` from usage events WHERE `inventory_id` matches AND date > `baseline_date`
4. **If no stocktake exists**: baseline_qty = 0, baseline_date = epoch (count all purchases ever)

`qty_on_hand = baseline_qty + purchased_since_stocktake - used_since_stocktake`

**Important**: `qty_on_hand` in `inventory.csv` is a pre-calculated cache/display value. The app should recalculate it from purchases + stocktakes on every load and update the field when exporting.

---

## Status logic
| Status   | Condition                        | Badge colour |
|----------|----------------------------------|-------------|
| Critical | qty_on_hand = 0                  | Red         |
| Low      | 0 < qty_on_hand ≤ low_stock_threshold | Amber  |
| Good     | qty_on_hand > low_stock_threshold | Blue      |

---

## Features to build

### Confirmed features

**Inventory view (main screen)**
- Card grid, one card per filament type
- Each card shows: large colour band at top (using `hex` value) with a subtle spool icon silhouette, brand (small caps), colour name (prominent), family + type (small), qty on hand, status badge
- Light-coloured filaments (luminance > 155) get a thin border-bottom on the colour band and a dark icon; dark filaments get a light icon
- A "LIST" pill appears on the colour band if `on_shopping_list = true`
- Hover: slight lift + shadow

**Filters and search**
- Filter chips: All | PLA | PETG | TPU | Silk PLA | ⚠ Low stock
  - PLA filter catches all `family = PLA` variants (PLA+ HS, Matte, standard, Glow-In-Dark)
- Free text search across colour, brand, type
- Searching/filtering updates the card grid in real time

**Card action buttons** (two small icon buttons bottom-right of each card)
- 🛒 Cart icon: toggle `on_shopping_list`
- ✏️ Edit icon: open inline edit form for that item

**Add new filament type**
- Inline panel that expands below the tab bar (not a modal — modals have positioning issues in GitHub Pages iframes)
- Fields: Brand, Family (dropdown), Type, Colour name, Colour (colour picker), Spool weight (default 1000g), Low stock threshold (default 1), Notes
- Saving adds a row to inventory.csv and a row to purchases.csv (first purchase)

**Log a purchase**
- Accessible from each card (e.g. a "+" button) or from a dedicated "Log purchase" form
- Fields: inventory_id (pre-filled from card), date, qty_purchased, order_id, supplier, style (spool/refilament), notes
- On save: appends to purchases.csv, recalculates and updates `qty_on_hand`

**"Used a spool" button**
- On each card, a clearly labelled button (e.g. "– Used")
- Shows confirmation: "Mark 1 spool of [Colour] as used? (qty will drop from X to X-1)"
- Decrements `qty_on_hand` by 1
- Records the event so the qty calculation stays accurate after a stocktake
- Suggest storing usage events as negative-qty rows in purchases.csv (e.g. `qty_purchased = -1, style = "used"`) to keep everything in one file

**Shopping list tab**
- Two sections: "Auto-flagged — Low or Critical stock" (not manually added) and "Manually added" (on_shopping_list = true)
- Each item shows: colour swatch, colour name, brand · type · qty remaining, status badge
- Manually added items have an ✕ remove button
- "Copy list" button: copies plain text list to clipboard
  ```
  Filament shopping list
  • eSUN PLA+ HS – White (Warm White) — 3 on hand (reorder)
  • eSUN PLA+ HS – Black — 0 on hand (critical)
  ```

**Purchase history tab**
- All purchases.csv rows sorted by date descending
- Shows: date, colour swatch + name, brand · family · type, qty, supplier, order_id, style
- Grouped by order_id where possible

**Stocktake mode**
- Accessible from a button in the header or settings
- Shows all inventory items in a list with current calculated qty
- For each item: "Calculated: X spools — enter actual count if different" (number input, blank = no change)
- Only items where the user changes the value write a new row to stocktakes.csv
- Can also upload a pre-filled stocktakes.csv to bypass the UI
- After confirming stocktake, recalculates all qty_on_hand values

**Import / Export**
- "Export CSVs" button: downloads all three CSVs as a zip, or downloads them individually
- "Import" file picker: lets user load a CSV from disk to replace/merge data (useful after direct GitHub edits)
- On app load: `fetch('./inventory.csv')`, `fetch('./purchases.csv')`, `fetch('./stocktakes.csv')` — graceful 404 handling (start empty if files not found)

### Deferred features (not in v1 — user confirmed not needed yet)
- Print job log
- Cost tracking
- Usage & spend charts
- QR label generator
- AMS slot notation
- Weight tracking for open spools

---

## UI design

### Colour palette
```
Header background:  #1C1C2E  (dark navy)
Accent / CTA:       #D85A30  (coral-red — used for Add button, tab underline, active chips)
Page background:    var(--color-background-tertiary)  — use CSS variables for light/dark compat
Card background:    var(--color-background-primary)
Text primary:       var(--color-text-primary)
Text secondary:     var(--color-text-secondary)
Border:             var(--color-border-tertiary)
Status critical:    var(--color-text-danger) / var(--color-background-danger)
Status low:         var(--color-text-warning) / var(--color-background-warning)
Status good:        var(--color-text-info) / var(--color-background-info)
```

### Typography
- Load from Google Fonts: `Space Grotesk` (400, 500, 600, 700) for all UI text
- Load from Google Fonts: `JetBrains Mono` (400, 600) for qty numbers and weight values

### Layout
- Header (dark): logo left, low-stock badge + Add button right
- Tab bar (white/primary bg): Inventory | Shopping List | Purchase History | Stocktake
- Notification strip: single inline line below tabs for toast messages (no position:fixed — causes issues in some contexts)
- Inline add/edit form: expands below tab bar when active; "Add spool" button in header hides while form is open
- Card grid: `repeat(auto-fill, minmax(165px, 1fr))` with 12px gap

### Spool card anatomy
```
┌─────────────────────┐
│  [COLOUR BAND 64px] │  ← hex background + SVG spool icon silhouette
│   ○──○              │     + "LIST" pill top-right if on_shopping_list
│     ○               │
├─────────────────────┤
│ BRAND (9px caps)    │
│ Colour Name (13px)  │
│ family · type (11px)│
│                     │
│ Remaining  2 spools │  ← JetBrains Mono
│ [████░░░░░░░░░░░]   │  ← optional qty bar (max = some reference)
│                     │
│ [Good]    [🛒][✏️]  │
└─────────────────────┘
```

### Spool icon SVG (colour band)
```svg
<svg width="44" height="44" viewBox="0 0 44 44" fill="none">
  <circle cx="22" cy="22" r="17" stroke="${iconColor}" stroke-width="2.5"/>
  <circle cx="22" cy="22" r="6.5" fill="${iconColor}"/>
  <circle cx="22" cy="22" r="2.6" fill="${centerColor}"/>
</svg>
```
- `iconColor`: rgba(0,0,0,0.2) for light filaments; rgba(255,255,255,0.3) for dark
- `centerColor`: rgba(0,0,0,0.15) for light; rgba(255,255,255,0.5) for dark

### Light colour detection
```javascript
function isLight(hex) {
  const r = parseInt(hex.slice(1,3),16);
  const g = parseInt(hex.slice(3,5),16);
  const b = parseInt(hex.slice(5,7),16);
  return (r*299 + g*587 + b*114) / 1000 > 155;
}
```
Light filaments also get `border-bottom: 0.5px solid var(--color-border-tertiary)` on the colour band.

---

## Technical approach

### CSV parsing
Use **Papa Parse** from CDN (already available at `https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js`). Handles quoted fields, commas in values (e.g. colour names with parentheses), missing fields.

### Data loading
```javascript
async function loadData() {
  const [inv, pur, stk] = await Promise.all([
    fetchCSV('./inventory.csv'),
    fetchCSV('./purchases.csv'),
    fetchCSV('./stocktakes.csv'),
  ]);
  // recalculate qty_on_hand from purchases + stocktakes
  // then render
}

async function fetchCSV(path) {
  try {
    const r = await fetch(path);
    if (!r.ok) return [];
    const text = await r.text();
    return Papa.parse(text, { header: true, skipEmptyLines: true }).data;
  } catch { return []; }
}
```

### Saving / exporting
GitHub Pages is read-only — the app can't write back automatically. Workflow:
1. User makes changes in app → data held in memory
2. User clicks "Export CSVs" → downloads updated inventory.csv, purchases.csv, stocktakes.csv
3. User commits files to GitHub → next page load picks up changes

Consider offering "Export all as ZIP" using a simple client-side zip library (JSZip from CDN).

### Icons
Use **Tabler Icons** via CDN: `https://cdn.jsdelivr.net/npm/@tabler/icons-webfont@latest/dist/tabler-icons.min.css`
Relevant icons: `ti-box` (inventory), `ti-shopping-cart`, `ti-history`, `ti-clipboard-check` (stocktake), `ti-plus`, `ti-edit`, `ti-trash`, `ti-download`, `ti-upload`, `ti-copy`, `ti-alert-triangle`, `ti-x`, `ti-check`

---

## Real data summary
- **30 unique filament types** across: Bambu (2), eSUN PLA+ HS (25), eSUN PLA Matte (1), eSUN PLA Glow-In-Dark (1), eSUN PLA standard Clear (1), eSUN Silk PLA Mystic Triple Colour (1), eSUN PETG standard (1), eSUN TPU 95A (1)
- **40 purchases** across 5 orders from Cubic Technology (Oct–Dec 2024), 1 Amazon order, 1 direct Bambu order (Sep 2024)
- **Highest stock**: White (Warm White) — 5 spools (inventory_id 26). Consider raising its low_stock_threshold to 2.
- **Refilament format** (paper tube, no plastic hub): 4 purchase rows have `style = refilament` — Blue, Red, White (Warm White) from order #81076SP, and PETG White from #81094SP
- **Suppliers in data**: Bambu, Amazon, Cubic Technology

---

## What's already done
- ✅ inventory.csv (30 rows, real data, hex colours from Cubic Tech page)
- ✅ purchases.csv (40 rows, inventory_id foreign keys filled in)
- ✅ stocktakes.csv (header only, ready to populate)
- ✅ UI prototyped in Claude.ai (two iterations — filter, search, shopping list, inline add form all verified working)
- ✅ Data architecture and calculation logic designed and agreed

## What needs building
- ❌ index.html — the full app (everything described above)
