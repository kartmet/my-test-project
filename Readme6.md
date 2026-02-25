# MONITORTIMEOUT Grid Color Fix – Persistent White Cell

## Problem

1. When status was MONITORTIMEOUT, the heatmap grid cell showed white/blank/gray instead of navy.
2. After MONITORTIMEOUT occurred once, the cell **never** showed color again, even when status later changed to UP, DOWN, or DEGRADED.
3. The cell stayed clickable and the tooltip showed the correct status, but the bar color stayed invisible.

## Root Causes

### 1. Stale grid data (no reactivity)

- In `monitor.svelte`, `_90Day` was set once: `let _90Day = monitor.pageData._90Day`.
- When the user navigated away and back (or the parent re-fetched), `monitor` was updated with new `pageData`, but `_90Day` was **not** reassigned (Svelte only runs initializations once).
- The grid kept using the old `_90Day` object, so it never showed updated statuses and never “recovered” after MONITORTIMEOUT.

### 2. Invalid class when `cssClass` was missing

- The bar uses `class="bg-{bar.cssClass}"`. If `bar.cssClass` was `undefined` (e.g. serialization or a missing status path), the class became `bg-undefined`, which has no CSS rule → transparent/white cell.
- Once that invalid state was in the grid, without reactivity it stayed until a full reload.

### 3. Server could send undefined

- If `cssClass` was ever undefined in the page.js loop (e.g. an unhandled status or edge case), it was written to the payload and the client rendered `bg-undefined`.

## Changes Made

### 1. `src/lib/components/monitor.svelte`

- **Reactive `_90Day`:** Replaced `let _90Day = monitor.pageData._90Day` with `$: _90Day = monitor.pageData._90Day` so the grid always uses the latest `pageData` when `monitor` changes. The cell can show UP/DOWN/DEGRADED again after MONITORTIMEOUT.
- **Fallbacks for class usage:** Use `bar.cssClass || 'api-nodata'` and `bar.textClass || 'api-nodata'` everywhere the bar/tooltip classes are applied:
  - Main grid bar: `bg-{bar.cssClass || 'api-nodata'}`.
  - Tooltip text color: `text-{bar.textClass || 'api-nodata'}`.
  - Day modal squares: `bg-{bar.cssClass || 'api-nodata'}` and `text-{bar.cssClass || 'api-nodata'}` for the dot.
  - If `cssClass`/`textClass` is ever missing, the cell shows gray (NO_DATA) instead of an invalid class and white.

### 2. `src/lib/server/page.js`

- **Defensive `cssClass`:** Set `_90Day[ts].cssClass = cssClass ?? StatusObj.NO_DATA` so the server never sends `undefined`. The client always receives a valid class name.

### 3. `src/routes/(kener)/+layout.svelte`

- **Theme variable for MONITORTIMEOUT:** Added `--monitortimeout-color: {data.site.colors?.MONITORTIMEOUT ?? '#000080'}` to the `<main>` style so the navy color is set (and themeable) like the other status colors.

## Result

- MONITORTIMEOUT cells use `api-monitortimeout` (navy) when the server sends it; the layout variable ensures the color is applied.
- If `cssClass` is ever missing, the cell shows gray (`api-nodata`) instead of white.
- When status changes (e.g. MONITORTIMEOUT → UP), the grid updates because `_90Day` is reactive, so the cell recovers and shows the new color.
- No persistent invalid state: new data replaces the previous grid data, and invalid classes are avoided by fallbacks and server-side default.

## Verification

1. Set a monitor to MONITORTIMEOUT (eval returns MONITORTIMEOUT); load the status page → current day cell should be navy.
2. Change status back to UP (or DOWN/DEGRADED) and reload or re-navigate to the status page → same cell should show the new color (e.g. green for UP).
3. In DevTools, confirm the bar’s class is `bg-api-monitortimeout` or `bg-api-up` etc., never `bg-undefined`.
