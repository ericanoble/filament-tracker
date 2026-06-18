# 🧵 Filament Tracker

A private web app for tracking 3D printing filament inventory. No server, no login, no subscription — your data lives in your own private GitHub Gist.

## How it works

```
GitHub Pages (hosts the app) ⟷ Your browser (runs everything) ⟷ GitHub Gist (stores your data, private)
```

Your filament data goes directly from your browser to your private Gist using a personal access token you enter once. Nothing touches a third-party server.

---

## Data stored (3 tables in your Gist)

### Inventory — one row per filament
| Field | Notes |
|---|---|
| id | unique key |
| colour | supplier's colour name e.g. "Fire Engine Red" |
| brand | e.g. eSUN, Bambu |
| type | e.g. PLA+, PETG-CF |
| family | broad group e.g. PLA, PETG, TPU |
| spool_weight_g | weight of one spool in grams |
| low_stock_threshold | alert when qty falls below this (grams, default 500g) |
| hex_colour | for the colour swatch on the card |
| colour_family | *(Phase 2 — auto-detected from hex)* |

### Purchases — one row per purchase event
| Field | Notes |
|---|---|
| id | unique key |
| inventory_id | links to Inventory |
| purchase_date | |
| qty_purchased | number of spools bought |
| price_paid | |
| supplier | |
| order_id | |
| notes | |

### Stocktakes — one row per manual count
| Field | Notes |
|---|---|
| id | unique key |
| inventory_id | links to Inventory |
| stocktake_date | |
| qty_counted | number of spools physically counted on that date |
| notes | |

**Quantity on hand** is calculated, not stored: `(latest stocktake + purchases since then) × spool weight in grams`

---

## Feature status

### ✅ Done
- Core inventory — add, edit, view filament cards with colour swatch
- Private storage via GitHub Gist (PAT login, auto-save)
- Quantity tracked in grams — spool weight dropdown, calculated from stocktakes + purchases
- Low stock alerts — 500g threshold, Critical / Low badges on cards
- Mobile layout — bottom tab bar, responsive cards
- Brand & spool weight dropdowns — built dynamically from your inventory, learns as you add filaments
- Filter bar Phase 1 — Status dropdown (All / Low stock / Critical), Family dropdown (dynamic), search with ✕ clear

### 🔜 Phase 2
- Colour chip filter — activate the greyed-out Colour dropdown by adding `colour_family` to inventory, auto-detected from hex value

### 📋 Planned (Phase 3)
- Add filament form rebuild — predictive colour names by brand, automatic hex assignment, multi-spool qty entry
- Colour picker moves from Add form to Edit card only
- Supplier & order ID dropdowns — predictive from purchase history

---

## Tech stack

| | |
|---|---|
| App | Single `index.html` file, no build step |
| Hosting | GitHub Pages |
| Storage | GitHub Gist API (CSV files) |
| CSV parsing | PapaParse 5.4.1 |
| ZIP export | JSZip 3.10.1 |
| Icons | Tabler Icons webfont |
| Fonts | Space Grotesk, JetBrains Mono |
