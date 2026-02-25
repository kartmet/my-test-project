# MONITORTIMEOUT Heatmap Grid Fix – README

## 1. Root cause

Two issues caused the MONITORTIMEOUT grid bug (white cell that never recovers):

1. **Stale grid data (no reactivity)**  
   `_90Day` was set once at component init: `let _90Day = monitor.pageData._90Day`. When the user navigated away and back or when the parent refetched, `monitor.pageData` was updated but `_90Day` was never reassigned (Svelte does not re-run initializations when props change). The grid kept showing the old object, so once a cell was in a bad state it never updated and never “recovered” when status later changed to UP/DOWN/DEGRADED.

2. **Invalid or missing `cssClass`**  
   - If `bar.cssClass` was ever `undefined` or `""`, the template produced `bg-undefined` or `bg-`, which have no CSS rule, so the cell looked white/blank.  
   - The server could write `_90Day[ts].cssClass = cssClass` when `cssClass` was undefined (e.g. edge case or unhandled status), so the client received an invalid value and rendered an invalid class.

3. **MONITORTIMEOUT color not set in layout**  
   The (kener) layout set `--up-color`, `--down-color`, `--degraded-color`, and `--maintenance-color` on `<main>` from `data.site.colors`, but not `--monitortimeout-color`. The navy color is defined in `kener.css` on `:root`; if the cascade or theme didn’t apply it correctly, the MONITORTIMEOUT bar could have no visible background.

---

## 2. Why previous fixes did not work

- **If reactivity was not applied:** The grid kept using the initial `_90Day` reference, so updated data (e.g. status back to UP) never appeared and the cell never recovered.  
- **If fallbacks were not applied:** Any `undefined` or empty `bar.cssClass` still produced `bg-undefined` or `bg-`, so the cell stayed white.  
- **If server defensive assignment was missing:** The payload could still contain `cssClass: undefined` for some days, so the client continued to render an invalid class.  
- **If the layout did not set `--monitortimeout-color`:** The navy variable might not be applied where the bar is rendered, so the MONITORTIMEOUT class could exist but have no visible color.

All four changes (reactive `_90Day`, client fallbacks, server `?? StatusObj.NO_DATA`, layout `--monitortimeout-color`) are required for MONITORTIMEOUT to show navy and for the grid to recover after status changes.

---

## 3. What was changed

| File | Change |
|------|--------|
| **monitor.svelte** | (1) Make `_90Day` reactive so it updates when `monitor.pageData` changes. (2) Use `bar.cssClass \|\| 'api-nodata'` and `bar.textClass \|\| 'api-nodata'` everywhere the grid and day modal use bar classes so no invalid class is ever rendered. |
| **page.js** | Set `_90Day[ts].cssClass = cssClass ?? StatusObj.NO_DATA` and derive `textClass` from the same value so the server never sends undefined. Removed temporary debug instrumentation. |
| **(kener)/+layout.svelte** | Add `--monitortimeout-color: {data.site.colors?.MONITORTIMEOUT ?? '#000080'}` to the `<main>` style so MONITORTIMEOUT uses navy (and is themeable). |

---

## 4. How the fix avoids breaking existing behavior

- **Reactive `_90Day`:** Only the source of `_90Day` changes (from one-time init to reactive assignment). The rest of the component still reads and mutates `_90Day` the same way (e.g. `border`, `showDetails`). When `monitor` does not change, behavior is unchanged.  
- **Fallbacks `|| 'api-nodata'`:** They only apply when `bar.cssClass` or `bar.textClass` is falsy. For all existing valid statuses (UP, DOWN, DEGRADED, MAINTENANCE, NO_DATA) the server sends a non-empty string, so the fallback is never used and display is unchanged.  
- **Server `?? StatusObj.NO_DATA`:** Only affects the case where `cssClass` would have been undefined; all existing code paths set `cssClass` to a string, so payloads that were already correct are unchanged.  
- **Layout variable:** Only adds a new CSS variable with a default. Existing status colors and theme behavior are unchanged.

---

## 5. How the fix ensures MONITORTIMEOUT works consistently

- **Navy displays:** The layout sets `--monitortimeout-color` (default `#000080`), and `kener.css` already defines `.bg-api-monitortimeout { background-color: var(--monitortimeout-color); }`. The server sends `cssClass: "api-monitortimeout"` (or a partial variant) for MONITORTIMEOUT days, so the bar gets the navy background.  
- **Grid recovers:** Because `$: _90Day = monitor.pageData._90Day`, when the user navigates back or the parent refetches and passes new `pageData`, the grid switches to the new data and shows the new status (e.g. UP/DOWN/DEGRADED) instead of the old MONITORTIMEOUT state.  
- **No persistent invalid state:** (1) New data replaces the previous grid data via reactivity. (2) The server never sends undefined `cssClass`. (3) If the client ever sees a missing class, it uses `api-nodata` (gray) instead of an invalid class, so no permanent “blank” state is stored or rendered.

---

## 6. BEFORE and AFTER code blocks (with line numbers)

------------------ CHANGE START ------------------

### monitor.svelte – Reactive _90Day

# BEFORE (lines 37–39):

```37:39:src/lib/components/monitor.svelte
  let _0Day = {};
  let _90Day = monitor.pageData._90Day;
  let uptime90Day = monitor.pageData.uptime90Day;
```

# AFTER (lines 37–40):

```37:40:src/lib/components/monitor.svelte
  let _0Day = {};
  // Reactive so grid uses latest pageData when monitor updates (fixes "never recovers" after MONITORTIMEOUT).
  $: _90Day = monitor.pageData._90Day;
  let uptime90Day = monitor.pageData.uptime90Day;
```

# JUSTIFICATION:
_90Day was assigned once, so when `monitor.pageData` changed (e.g. after navigation or refetch), the grid kept showing the old data and the cell never recovered. Making _90Day reactive ensures the grid always uses the latest `pageData`. Existing logic that reads/mutates _90Day is unchanged; only the source of _90Day is different.

------------------ CHANGE END --------------------

------------------ CHANGE START ------------------

### monitor.svelte – Grid bar class fallback

# BEFORE (lines 393–397):

```393:397:src/lib/components/monitor.svelte
              <div
                class="oneline-in h-[30px] bg-{bar.cssClass} mx-auto rounded-{monitor.pageData.barRoundness.toUpperCase() ==
                'SHARP'
                  ? 'none'
                  : 'sm'}"
```

# AFTER (lines 393–397):

```393:397:src/lib/components/monitor.svelte
              <div
                class="oneline-in h-[30px] bg-{bar.cssClass || 'api-nodata'} mx-auto rounded-{monitor.pageData.barRoundness.toUpperCase() ==
                'SHARP'
                  ? 'none'
                  : 'sm'}"
```

# JUSTIFICATION:
If `bar.cssClass` was undefined or empty, the class became `bg-undefined` or `bg-`, which have no style and look white. Using `bar.cssClass || 'api-nodata'` ensures a valid class; existing bars with a valid cssClass are unchanged.

------------------ CHANGE END --------------------

------------------ CHANGE START ------------------

### monitor.svelte – Tooltip text class fallback

# BEFORE (line 412):

```412:412:src/lib/components/monitor.svelte
                <div class="text-{bar.textClass} text-xs font-semibold">
```

# AFTER (line 412):

```412:412:src/lib/components/monitor.svelte
                <div class="text-{bar.textClass || 'api-nodata'} text-xs font-semibold">
```

# JUSTIFICATION:
Same as grid bar: avoid `text-undefined` when `bar.textClass` is missing. Valid values are unchanged.

------------------ CHANGE END --------------------

------------------ CHANGE START ------------------

### monitor.svelte – Day modal square and dot fallbacks

# BEFORE (lines 474, 483):

```474:474:src/lib/components/monitor.svelte
                  <div data-index={bar.index} class="bg-{bar.cssClass} today-sq m-[1px] h-[10px] w-[10px]"></div>
```

```483:483:src/lib/components/monitor.svelte
                        <span class="text-{bar.cssClass}"> ● </span>
```

# AFTER (lines 474, 483):

```474:474:src/lib/components/monitor.svelte
                  <div data-index={bar.index} class="bg-{bar.cssClass || 'api-nodata'} today-sq m-[1px] h-[10px] w-[10px]"></div>
```

```483:483:src/lib/components/monitor.svelte
                        <span class="text-{bar.cssClass || 'api-nodata'}"> ● </span>
```

# JUSTIFICATION:
Day modal uses the same bar data; fallbacks prevent invalid classes there as well. No change for valid cssClass/textClass.

------------------ CHANGE END --------------------

------------------ CHANGE START ------------------

### page.js – Defensive cssClass and textClass

# BEFORE (lines 163–170):

```163:170:src/lib/server/page.js
    _90Day[ts].timestamp = ts;
    _90Day[ts].cssClass = cssClass;
    _90Day[ts].summaryStatus = l(lang, summaryTime(summaryStatus), {
      status: l(lang, summaryStatus),
      duration: summaryDuration,
    });

    _90Day[ts].textClass = cssClass.replace(/-\d+$/, "");
```

# AFTER (lines 163–171):

```163:171:src/lib/server/page.js
    _90Day[ts].timestamp = ts;
    _90Day[ts].cssClass = cssClass ?? StatusObj.NO_DATA;
    _90Day[ts].summaryStatus = l(lang, summaryTime(summaryStatus), {
      status: l(lang, summaryStatus),
      duration: summaryDuration,
    });

    _90Day[ts].textClass = (cssClass ?? StatusObj.NO_DATA).replace(/-\d+$/, "");
```

# JUSTIFICATION:
If `cssClass` was ever undefined, the client received an invalid value and rendered `bg-undefined`. Using `cssClass ?? StatusObj.NO_DATA` ensures the server never sends undefined; existing paths that always set `cssClass` are unchanged. `textClass` is derived from the same value so it stays in sync.

------------------ CHANGE END --------------------

------------------ CHANGE START ------------------

### (kener)/+layout.svelte – MONITORTIMEOUT theme variable

# BEFORE (lines 107–114):

```107:114:src/routes/(kener)/+layout.svelte
  style="
	--font-family: {data.site.font.family};
	--bg-custom: {data.bgc};
	--up-color: {data.site.colors.UP};
	--down-color: {data.site.colors.DOWN};
	--degraded-color: {data.site.colors.DEGRADED};
	--maintenance-color: {data.site.colors.MAINTENANCE};
	"
```

# AFTER (lines 107–115):

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

# JUSTIFICATION:
MONITORTIMEOUT bars use `var(--monitortimeout-color)` in `kener.css`. Setting this variable on `<main>` (with theme fallback) ensures navy is applied and can be themed like other status colors. Other variables and behavior are unchanged.

------------------ CHANGE END --------------------

---

## 7. Summary

- **Root cause:** Stale `_90Day` (no reactivity), possible undefined/missing `cssClass` (no server guard, no client fallback), and MONITORTIMEOUT color not set in the layout.  
- **Why previous fixes failed:** Any of the four pieces (reactivity, client fallbacks, server guard, layout variable) missing leaves the bug in place.  
- **What was changed:** Reactive `_90Day`, `bar.cssClass || 'api-nodata'` and `bar.textClass || 'api-nodata'` in the grid and day modal, `cssClass ?? StatusObj.NO_DATA` (and same for `textClass`) in page.js, and `--monitortimeout-color` in (kener) layout.  
- **No breaking changes:** Only the data source for _90Day, guards on missing values, and one new CSS variable were added; existing status handling and display are preserved.  
- **Result:** MONITORTIMEOUT shows navy, the grid updates when status changes (including recovery after MONITORTIMEOUT), and no persistent invalid state is created.
