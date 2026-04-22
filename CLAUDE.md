# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NutriTrack is a client-side-only Progressive Web App (PWA) for nutrition and fitness tracking, written in Italian. It is a **single-file application**: all HTML, CSS, and JavaScript live in `index.html`. There is no build step, no package manager, and no framework — just vanilla JS, HTML, and CSS served statically.

**Target device**: Android smartphone (Chrome). All UX decisions (touch targets, modal-from-bottom, max-width 430px) are optimized for mobile. The app is installed as a PWA from the browser's "Aggiungi alla schermata Home" option.

## Running the App

Serve the directory over HTTP — opening `index.html` directly as a `file://` URL will break OPFS and Service Worker APIs:

```bash
python -m http.server 8080
# or
npx serve .
```

There are no build, test, or lint commands configured.

## Architecture

Everything lives in `index.html` (~1,300 lines). The logical sections are:

| Section | Approx. Lines | Purpose |
|---|---|---|
| HTML structure | 1–100 | Shell, bottom nav, modals |
| CSS | 100–510 | All styling, dark/light theme via CSS vars |
| DB init & schema | 515–640 | sql.js SQLite setup, OPFS/localStorage persistence, `seedFoodDB()` |
| Global state & config | ~680–710 | `CFG` object (user settings), `curDate`, `editingRecipeId` |
| Profile page | ~730–785 | BMR/TDEE calculation (Mifflin-St Jeor) |
| Dashboard (Oggi) | ~785–835 | Daily calorie/macro/water summary |
| Water tracking | ~835–920 | Glass counter, Notification API, `checkWaterOnOpen()` |
| Activities | ~920–960 | Exercise logging |
| Food search / Meals | ~960–1060 | Local DB search (`food_db`), add-to-DB modal, meals CRUD |
| Recipes | ~1060–1140 | Custom recipe library with create/edit/delete |
| Reports | ~1140–1185 | 7-day and monthly analytics |
| Page router & init | ~1185–end | Navigation, SW registration, app bootstrap |

### Data Layer

- **Engine**: [sql.js](https://sql.js.org/) v1.10.3 (SQLite compiled to WebAssembly), loaded from CDN.
- **Persistence**: Writes to **OPFS** (Origin Private File System) on every `dbRun()` call via `persistDB()`. Falls back to base64-encoded `localStorage` when OPFS is unavailable.
- **Schema**: Six tables — `settings`, `foods`, `activities`, `water`, `recipes`, `food_db`. Date keys are `YYYY-MM-DD` strings.
- **Backup**: Full JSON export/import (versioned, `v:4`) via the Profile page. Custom foods (`food_db WHERE custom=1`) are included; seed foods are not.

### External APIs

- **OpenFoodFacts**: removed — food search is now fully local (SQLite `food_db` table).
- **Notification API**: water-reminder push notifications (time-gated 07:00–22:00, configurable interval). Runs only while the app is open (`setInterval`). On Android a PWA cannot receive background push without a server backend (Push API + VAPID), so notifications fire only when the app is in foreground. A one-shot notification also fires on app open if >1 hour has passed since the last launch and the water goal isn't met.

### State Management

There is no reactive framework. State is a global `CFG` object (hydrated from the `settings` DB table on startup). UI updates are imperative DOM manipulation triggered by event handlers. Every write goes directly to SQLite then re-renders the relevant section.

### Service Worker (`sw.js`)

Cache name `nutritrack-v5`. Caches all static assets. OpenFoodFacts requests are explicitly excluded from caching so food data stays fresh. To force clients to reload after a deploy, bump the cache version string — the activate handler automatically deletes old caches.

## Key Calculations

```
BMR (Mifflin-St Jeor):
  Male:   10*weight + 6.25*height - 5*age + 5
  Female: 10*weight + 6.25*height - 5*age - 161

TDEE = BMR × activity_factor   (1.2 → 1.9)
Daily deficit target = desiredWeightLoss(kg) * 7700 / 7
TargetKcal = TDEE - deficit
```

Default macro split: protein 1.8 g/kg body weight; carbs 45% of target kcal; fats 25% of target kcal.

## PWA Manifest

`manifest.json` sets scope to `/nutritrack/`. If you move the app to a different path, update `start_url` and `scope` accordingly. The service worker path in `index.html` must also match.

## Food Database (`food_db` table)

The app uses a local SQLite food database instead of any external API.

- **Seed**: ~130 common Italian foods loaded by `seedFoodDB()` at first boot (runs once, checks `COUNT(*)`). Seed rows have `custom=0`.
- **User additions**: when a user adds a new food during search, it's inserted with `custom=1`. Custom foods appear first in search results (`ORDER BY custom DESC`).
- **Search**: `LIKE '%term%'` query, minimum 2 characters, 200 ms debounce, max 20 results. Always shows an "➕ Aggiungi al database" option at the bottom of the list.
- **CSV import**: Profile → "📥 Importa alimenti da CSV". The user first selects the separator (`;` or `,`) via a modal, then picks the file. Parser (`parseCSVLine`) handles RFC 4180 quoted fields (commas inside names, `""` escaping). Columns: `nome, kcal, proteine, carboidrati, grassi` — header row auto-detected. Imported rows are inserted as `custom=1`. `tabella-alimenti-completa.csv` (900 Italian foods from CREA) is included in the repo as a ready-to-import dataset.
- **Management**: Profile → "⚙ Gestione alimenti DB" opens a searchable modal listing all `food_db` entries. Each row has ✎ (edit) and ✕ (delete). Edit reuses `food-db-modal` with `editingFoodId` state (same pattern as `editingRecipeId`); the qty field is hidden when editing.
- **Backup**: only `custom=1` rows are included in JSON exports (seed data is re-generated on first boot on any device).

To add more seed foods, edit the `SEED` array inside `seedFoodDB()`. The array format is `[name, kcal/100g, prot, carb, fat]`. After editing, increment the seed version or clear `food_db` manually to re-seed (the function is a no-op if the table is non-empty).

## Notifications (Android PWA limitations)

- In-app reminders (`setInterval`) work only while the app is open in Chrome. Chrome on Android suspends JS when the app is backgrounded.
- `checkWaterOnOpen()` fires a one-shot notification on launch if: permission granted, reminder interval > 0, >1 hour since last open, current time within the configured window, and today's water goal not yet met. The last-open timestamp is stored in the `settings` table under key `lastOpen`.
- Full background push requires a server with VAPID keys (Push API) — not implemented.

## Recipes

Recipes support create, **edit** (pencil button `✎` in the list), and delete. The modal title and save behaviour are controlled by `editingRecipeId`: `null` = insert, non-null = update.

## Localization

The UI is fully in Italian. All user-facing strings are hard-coded inline — there is no i18n library or translation file.
