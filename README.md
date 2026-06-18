# 🧵 Filament Tracker

A private web app for tracking 3D printing filament inventory. No server, no login, no subscription — your data lives in your own private GitHub Gist.

## How it works

```
GitHub Pages (hosts the app) ⟷ Your browser (runs everything) ⟷ GitHub Gist (stores your data, private)
```

Your filament data goes directly from your browser to your private Gist using a personal access token you enter once. Nothing touches a third-party server.

---

## The filament lifecycle

```
Wishlist → Shopping list → Order placed → Inventory → Stocktake
```

- **Wishlist** — things you want to try someday (don't own yet)
- **Shopping list** — decided to buy; shows both low stock items and wishlist items, grouped by supplier
- **Inventory** — what you own; added via the + button (small orders) or CSV import (bulk)
- **Stocktake** — update quantity on hand when something prompts it

---

## Who uses the app

| Role | Can do |
|---|---|
| Owner (you) | View, add, edit, stocktake, manage wishlist, record purchases |
| Read-only (partner/kids) | View inventory, filter by colour — can't change anything |
| Contributor *(future)* | View inventory + add to wishlist |

---

## Data stored (4 tables in your Gist)

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
| colour_family | *(Phase 2 — auto-detected from hex: Red, Yellow, Blue…)* |

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

### Wishlist — one row per item you want to try *(Phase 3 — not built yet)*
| Field | Notes |
|---|---|
| id | unique key |
| name | e.g. "foaming wood filament" |
| family | e.g. PLA, specialty |
| supplier | e.g. Cubic Technology |
| approx_cost | |
| notes | why you want it |
| status | wishlist / on order |

---

## Feature status

### ✅ Done
- Core inventory — add, edit, view filament cards with colour swatch
- Private storage via GitHub Gist (PAT login, auto-save)
- Quantity tracked in grams — spool weight dropdown, calculated from stocktakes + purchases
- Low stock alerts — 500g threshold, Critical / Low badges on cards
- Mobile layout — bottom tab bar, responsive cards
- Brand & spool weight dropdowns — built dynamically from your inventory
- Filter bar — Status dropdown (All / Low stock / Critical), Family dropdown (dynamic), search with ✕ clear
- Result count only shows when a filter or search is active

### 🔜 Phase 2
- Colour chip filter — add `colour_family` to inventory (auto-detected from hex), activate Colour dropdown

### 📋 Phase 3
- Wishlist — new table; add/edit entries, move to shopping list when ready to buy
- Shopping list redesign — shows low stock + wishlist together, grouped by supplier
- Add filament form — full screen (not overlay on cards), buttons always visible, predictive colour names
- CSV import — downloadable blank template with headers + one example row

### 🔮 Future
- Read-only sharing URL — for partner/kids to check what's in stock
- Contributor access — wishlist-only write permission for family

### ❌ Not building
- Push notifications — seeing low stock when you open the app is enough
- Print job planning — once the app is working properly you'll know what you have

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
