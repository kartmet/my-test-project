# MONITORTIMEOUT Grid Fix – Changes with Line Numbers and Justifications

This document lists every code change made to fix the MONITORTIMEOUT heatmap grid (white/invisible cell, grid disappearing, never recovering). Each section has **BEFORE** (original code), **AFTER** (current code with line numbers), and **JUSTIFICATION**.

---

## File 1: `src/lib/components/monitor.svelte`

### Change 1.1: Safe pageData and reactive _90Day, uptime90Day, homeDataMaxDays (lines 37–50)

**BEFORE (original lines 37–50):**
```javascript
  let _0Day = {};
  let _90Day = monitor.pageData._90Day;
  let uptime90Day = monitor.pageData.uptime90Day;
  let incidents = {};
  let dayIncidentsFull = [];
  let homeDataMaxDays = monitor.pageData.homeDataMaxDays;

  let dimension = {
    x1: 6,
    x2: 4
  };

  dimension.x1 = ($page.data.isMobile ? 346 : 546) / homeDataMaxDays.maxDays;
  dimension.x2 = (4 / 6) * dimension.x1;
```

**AFTER (lines 37–50):**
```javascript
  let _0Day = {};
  // Safe pageData so grid never crashes when data is missing or still loading.
  $: pageData = monitor?.pageData ?? {};
  // Reactive so grid uses latest pageData when monitor updates (fixes "never recovers" after MONITORTIMEOUT).
  $: _90Day = pageData._90Day ?? {};
  $: uptime90Day = pageData.uptime90Day ?? "-";
  let incidents = {};
  let dayIncidentsFull = [];
  $: homeDataMaxDays = pageData.homeDataMaxDays ?? { maxDays: 90, selectableDays: [90, 30, 7, 1] };

  $: dimension = (() => {
    const x1 = Math.max(1, ($page.data.isMobile ? 346 : 546) / (homeDataMaxDays?.maxDays ?? 90));
    return { x1, x2: (4 / 6) * x1 };
  })();
```

**JUSTIFICATION:**  
Accessing `monitor.pageData._90Day` and similar when `pageData` was undefined caused a runtime error and the grid never rendered. Using `pageData = monitor?.pageData ?? {}` and deriving `_90Day`, `uptime90Day`, and `homeDataMaxDays` from it with fallbacks prevents crashes. Making these reactive (`$:`) ensures the grid updates when the server sends new data (e.g. after MONITORTIMEOUT → UP). Making `dimension` reactive and using `Math.max(1, …)` and `?? 90` avoids NaN or zero width when `homeDataMaxDays` is missing.

---

### Change 1.2: getBarBackgroundStyle with var() fallbacks (lines 52–66)

**BEFORE:**  
No helper; grid bar had only class-based background (e.g. `bg-{bar.cssClass}`).

**AFTER (lines 52–66):**
```javascript
  /** Inline background with var() fallback so bar is never transparent when variable is out of scope or invalid (fixes grid disappearing for MONITORTIMEOUT). */
  const STATUS_COLOR_FALLBACKS = {
    up: "#4ead94",
    down: "#ca3038",
    degraded: "#e6ca61",
    maintenance: "#6679cc",
    monitortimeout: "#000080"
  };
  function getBarBackgroundStyle(cssClass) {
    const c = cssClass || "";
    const m = c.match(/^api-(up|down|degraded|maintenance|monitortimeout)$/);
    if (!m) return "";
    const name = m[1];
    const fallback = STATUS_COLOR_FALLBACKS[name] || "#000080";
    return `background-color: var(--${name}-color, ${fallback});`;
  }
```

**JUSTIFICATION:**  
Using only `var(--monitortimeout-color)` with no fallback made the bar transparent when the variable was out of scope or invalid, so the grid looked like it disappeared (only hover showed it). Adding the second argument to `var()`, e.g. `var(--monitortimeout-color, #000080)`, guarantees a visible background. Fallbacks for all five statuses keep behavior consistent and avoid invisible bars for any status.

---

### Change 1.3: Use pageData instead of monitor.pageData (lines 71–73, 87, 131–132, 137–138)

**BEFORE (examples):**
```javascript
        startTs: monitor.pageData.midnight90DaysAgo,
        endTs: monitor.pageData.maxDateTodayTimestamp,
```
```javascript
    let endTs = Math.min(startTs + 86400, monitor.pageData.maxDateTodayTimestamp);
```
```javascript
          startTs: monitor.pageData.startOfTheDay,
          endTs: monitor.pageData.maxDateTodayTimestamp
        });
      } else {
        ret.push({
          text: `${roller} ${l(lang, "Days")}`,
          startTs: monitor.pageData.startOfTheDay - 86400 * (roller - 1),
          endTs: monitor.pageData.maxDateTodayTimestamp
```

**AFTER (lines 72–73, 87, 131–132, 137–138):**
```javascript
        startTs: pageData.midnight90DaysAgo,
        endTs: pageData.maxDateTodayTimestamp,
```
```javascript
    let endTs = Math.min(startTs + 86400, pageData.maxDateTodayTimestamp);
```
```javascript
          startTs: pageData.startOfTheDay,
          endTs: pageData.maxDateTodayTimestamp
        });
      } else {
        ret.push({
          text: `${roller} ${l(lang, "Days")}`,
          startTs: pageData.startOfTheDay - 86400 * (roller - 1),
          endTs: pageData.maxDateTodayTimestamp
```

**JUSTIFICATION:**  
All reads of `monitor.pageData` were switched to the safe reactive `pageData` so the component never touches `monitor.pageData` directly and cannot throw when it is undefined. Behavior is unchanged when data is present.

---

### Change 1.4: Summary display and barRoundness from pageData (lines 362–363, 392)

**BEFORE:**
```javascript
          <div class="truncate text-xs font-semibold text-{monitor.pageData.summaryColorClass}">
            {monitor.pageData.summaryStatus}
          </div>
```
```javascript
                class="oneline-in h-[30px] bg-{bar.cssClass} mx-auto rounded-{monitor.pageData.barRoundness.toUpperCase() ==
```

**AFTER (lines 362–363, 392):**
```javascript
          <div class="truncate text-xs font-semibold text-{pageData.summaryColorClass || 'api-nodata'}">
            {pageData.summaryStatus ?? ''}
          </div>
```
```javascript
                class="oneline-in h-[30px] bg-{bar.cssClass || 'api-nodata'} mx-auto rounded-{(pageData.barRoundness || 'ROUNDED').toUpperCase() ==
```

**JUSTIFICATION:**  
Using `pageData` avoids crashes when it is missing. Adding `|| 'api-nodata'` and `?? ''` and `|| 'ROUNDED'` ensures the template never receives undefined for class or text, so the summary and bar roundness render safely.

---

### Change 1.5: Empty grid state and bar fallbacks (lines 374–377, 391–396, 410)

**BEFORE:**
```html
        <div class="daygrid90 flex min-h-[60px] justify-between overflow-x-auto overflow-y-hidden py-1">
          {#each Object.entries(_90Day) as [ts, bar]}
            ...
              <div
                class="oneline-in h-[30px] bg-{bar.cssClass} mx-auto rounded-..."
                style="width: {dimension.x2}px"
              ></div>
            ...
                <div class="text-{bar.textClass} text-xs font-semibold">
```

**AFTER (lines 374–377, 391–396, 410):**
```html
        <div class="daygrid90 flex min-h-[60px] justify-between overflow-x-auto overflow-y-hidden py-1">
          {#if Object.keys(_90Day).length === 0}
            <p class="text-muted-foreground py-2 text-xs">No status data for this period.</p>
          {:else}
          {#each Object.entries(_90Day) as [ts, bar]}
            ...
              <div
                class="oneline-in h-[30px] bg-{bar.cssClass || 'api-nodata'} mx-auto rounded-{(pageData.barRoundness || 'ROUNDED').toUpperCase() ==
                'SHARP'
                  ? 'none'
                  : 'sm'}"
                style="width: {dimension.x2}px; {getBarBackgroundStyle(bar.cssClass)}"
              ></div>
            ...
                <div class="text-{bar.textClass || 'api-nodata'} text-xs font-semibold">
```

**JUSTIFICATION:**  
When `_90Day` is empty, showing "No status data for this period." makes it clear the UI is working. Using `bar.cssClass || 'api-nodata'` and `bar.textClass || 'api-nodata'` avoids `bg-undefined` or `text-undefined`, which would leave cells unstyled or broken. Adding `style="... {getBarBackgroundStyle(bar.cssClass)}"` applies the correct background (with var fallbacks) so MONITORTIMEOUT and other statuses are always visible.

---

### Change 1.6: Day modal squares and dot fallbacks (lines 474, 482)

**BEFORE:**
```html
                  <div data-index={bar.index} class="bg-{bar.cssClass} today-sq m-[1px] h-[10px] w-[10px]"></div>
                  ...
                        <span class="text-{bar.cssClass}"> ● </span>
```

**AFTER (lines 474, 482):**
```html
                  <div data-index={bar.index} class="bg-{bar.cssClass || 'api-nodata'} today-sq m-[1px] h-[10px] w-[10px]" style="{getBarBackgroundStyle(bar.cssClass)}"></div>
                  ...
                        <span class="text-{bar.cssClass || 'api-nodata'}"> ● </span>
```

**JUSTIFICATION:**  
Same as the main grid: fallbacks prevent invalid classes; the inline style from `getBarBackgroundStyle` ensures the day-detail squares (including MONITORTIMEOUT) always have a visible background.

---

## File 2: `src/lib/server/page.js`

### Change 2.1: Defensive cssClass and textClass (lines 147–148, 154)

**BEFORE (lines 147–154):**
```javascript
    _90Day[ts].timestamp = ts;
    _90Day[ts].cssClass = cssClass;
    _90Day[ts].summaryStatus = l(lang, summaryTime(summaryStatus), {
      status: l(lang, summaryStatus),
      duration: summaryDuration,
    });

    _90Day[ts].textClass = cssClass.replace(/-\d+$/, "");
```

**AFTER (lines 147–154):**
```javascript
    _90Day[ts].timestamp = ts;
    _90Day[ts].cssClass = cssClass ?? StatusObj.NO_DATA;
    _90Day[ts].summaryStatus = l(lang, summaryTime(summaryStatus), {
      status: l(lang, summaryStatus),
      duration: summaryDuration,
    });

    _90Day[ts].textClass = (cssClass ?? StatusObj.NO_DATA).replace(/-\d+$/, "");
```

**JUSTIFICATION:**  
If `cssClass` was ever undefined (e.g. unhandled status or edge case), the client received `cssClass: undefined` and rendered `bg-undefined`, producing an invisible or broken cell. Using `cssClass ?? StatusObj.NO_DATA` guarantees a valid class in the payload. Deriving `textClass` from the same value keeps it in sync and avoids sending undefined.

---

## File 3: `src/routes/(kener)/+layout.svelte`

### Change 3.1: MONITORTIMEOUT theme variable on main (lines 107–115)

**BEFORE (lines 107–114):**
```html
  style="
	--font-family: {data.site.font.family};
	--bg-custom: {data.bgc};
	--up-color: {data.site.colors.UP};
	--down-color: {data.site.colors.DOWN};
	--degraded-color: {data.site.colors.DEGRADED};
	--maintenance-color: {data.site.colors.MAINTENANCE};
	"
```

**AFTER (lines 107–115):**
```html
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

**JUSTIFICATION:**  
MONITORTIMEOUT bars use `var(--monitortimeout-color)` in inline styles and in `kener.css`. Setting this variable on `<main>` (with theme fallback `#000080`) ensures the navy color is in scope for the status page and can be themed like the other status colors. Existing variables and behavior are unchanged.

---

## Summary table

| File              | Location        | Change summary |
|-------------------|-----------------|----------------|
| monitor.svelte    | 37–50           | Safe reactive `pageData`, `_90Day`, `uptime90Day`, `homeDataMaxDays`, `dimension` |
| monitor.svelte    | 52–66           | `getBarBackgroundStyle()` with `var(--X-color, fallback)` for all statuses |
| monitor.svelte    | 72–73, 87, 131–132, 137–138 | Use `pageData` instead of `monitor.pageData` |
| monitor.svelte    | 362–363, 392    | Summary and bar roundness from `pageData` with fallbacks |
| monitor.svelte    | 374–377, 391–396, 410 | Empty-state message; `bar.cssClass`/`textClass` fallbacks; inline bar style |
| monitor.svelte    | 474, 482        | Day modal: class fallbacks and `getBarBackgroundStyle` on square |
| page.js           | 147–148, 154    | `cssClass ?? StatusObj.NO_DATA` and same for `textClass` |
| (kener)/+layout.svelte | 107–115    | Add `--monitortimeout-color` to `<main>` style |

All changes are backward-compatible: they only add guards, fallbacks, and one new CSS variable; existing behavior when data and variables are present is unchanged.
