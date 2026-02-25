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

#### 1a. Reactive `_90Day` (lines 37–41)

**BEFORE:**

```37:39:src/lib/components/monitor.svelte
  let _0Day = {};
  let _90Day = monitor.pageData._90Day;
  let uptime90Day = monitor.pageData.uptime90Day;
```

**AFTER:**

```37:41:src/lib/components/monitor.svelte
  let _0Day = {};
  // Reactive so grid updates when monitor/pageData changes (e.g. after navigation or refetch).
  // Without this, a cell that once showed MONITORTIMEOUT (or invalid cssClass) would never recover.
  $: _90Day = monitor.pageData._90Day;
  let uptime90Day = monitor.pageData.uptime90Day;
```

#### 1b. Main grid bar – fallback for `cssClass` (lines 374–377)

**BEFORE:**

```373:376:src/lib/components/monitor.svelte
              <div
                class="oneline-in h-[30px] bg-{bar.cssClass} mx-auto rounded-{monitor.pageData.barRoundness.toUpperCase() ==
                'SHARP'
                  ? 'none'
```

**AFTER:**

```374:377:src/lib/components/monitor.svelte
                class="oneline-in h-[30px] bg-{bar.cssClass || 'api-nodata'} mx-auto rounded-{monitor.pageData.barRoundness.toUpperCase() ==
                'SHARP'
                  ? 'none'
                  : 'sm'}"
```

#### 1c. Tooltip text color – fallback for `textClass` (line 392)

**BEFORE:**

```391:391:src/lib/components/monitor.svelte
                <div class="text-{bar.textClass} text-xs font-semibold">
```

**AFTER:**

```392:392:src/lib/components/monitor.svelte
                <div class="text-{bar.textClass || 'api-nodata'} text-xs font-semibold">
```

#### 1d. Day modal – square and dot fallbacks (lines 455, 464)

**BEFORE:**

```454:455:src/lib/components/monitor.svelte
                  <div data-index={bar.index} class="bg-{bar.cssClass} today-sq m-[1px] h-[10px] w-[10px]"></div>
```

```461:461:src/lib/components/monitor.svelte
                        <span class="text-{bar.cssClass}"> ● </span>
```

**AFTER:**

```455:455:src/lib/components/monitor.svelte
                  <div data-index={bar.index} class="bg-{bar.cssClass || 'api-nodata'} today-sq m-[1px] h-[10px] w-[10px]"></div>
```

```464:464:src/lib/components/monitor.svelte
                        <span class="text-{bar.cssClass || 'api-nodata'}"> ● </span>
```

---

### 2. `src/lib/server/page.js`

#### Defensive `cssClass` (lines 147–149)

**BEFORE:**

```146:148:src/lib/server/page.js
    _90Day[ts].timestamp = ts;
    _90Day[ts].cssClass = cssClass;
    _90Day[ts].summaryStatus = l(lang, summaryTime(summaryStatus), {
```

**AFTER:**

```147:150:src/lib/server/page.js
    _90Day[ts].timestamp = ts;
    // Ensure we never send undefined (avoids "bg-undefined" and persistent broken grid state)
    _90Day[ts].cssClass = cssClass ?? StatusObj.NO_DATA;
    _90Day[ts].summaryStatus = l(lang, summaryTime(summaryStatus), {
```

---

### 3. `src/routes/(kener)/+layout.svelte`

#### Theme variable for MONITORTIMEOUT (lines 107–115)

**BEFORE:**

```107:113:src/routes/(kener)/+layout.svelte
  style="
	--font-family: {data.site.font.family};
	--bg-custom: {data.bgc};
	--up-color: {data.site.colors.UP};
	--down-color: {data.site.colors.DOWN};
	--degraded-color: {data.site.colors.DEGRADED};
	--maintenance-color: {data.site.colors.MAINTENANCE};
	"
```

**AFTER:**

```107:115:src/routes/(kener)/+layout.svelte
  style="
	--font-family: {data.site.font.family};
	--bg-custom: {data.bgc};
	--up-color: {data.site.colors.UP};
	--down-color: {data.site.colors.DOWN};
	--degraded-color: {data.site.colors.DEGRADED};
	--maintenance-color: {data.site.colors.MAINTENANCE};
	--monitortimeout-color: {data.site.colors?.MONITORTIMEOUT ?? '#000080'};
	"
```

## Result

- MONITORTIMEOUT cells use `api-monitortimeout` (navy) when the server sends it; the layout variable ensures the color is applied.
- If `cssClass` is ever missing, the cell shows gray (`api-nodata`) instead of white.
- When status changes (e.g. MONITORTIMEOUT → UP), the grid updates because `_90Day` is reactive, so the cell recovers and shows the new color.
- No persistent invalid state: new data replaces the previous grid data, and invalid classes are avoided by fallbacks and server-side default.

## Verification

1. Set a monitor to MONITORTIMEOUT (eval returns MONITORTIMEOUT); load the status page → current day cell should be navy.
2. Change status back to UP (or DOWN/DEGRADED) and reload or re-navigate to the status page → same cell should show the new color (e.g. green for UP).
3. In DevTools, confirm the bar’s class is `bg-api-monitortimeout` or `bg-api-up` etc., never `bg-undefined`.
