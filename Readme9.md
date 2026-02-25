# MONITORTIMEOUT – Before/After Code Changes (with file names and line numbers)

Every modified file is listed with **Before** (exact existing code and line range) and **After** (updated code with inline justifications).  
**Change justification summary** (Purpose, Risk, Reasoning) follows each file.

---

## File: src/lib/server/db/dbimpl.js

### Before (lines 112–126)

```javascript
  //given monitor_tag, start and end timestamp in utc seconds return total degraded, up, down, avg(latency), max(latency), min(latency)
  async getAggregatedMonitoringData(monitor_tag, start, end) {
    return await this.knex("monitoring_data")
      .select(
        this.knex.raw("COUNT(CASE WHEN status = 'DEGRADED' THEN 1 END) as DEGRADED"),
        this.knex.raw("COUNT(CASE WHEN status = 'UP' THEN 1 END) as UP"),
        this.knex.raw("COUNT(CASE WHEN status = 'DOWN' THEN 1 END) as DOWN"),
        this.knex.raw("AVG(latency) as avg_latency"),
        this.knex.raw("MAX(latency) as max_latency"),
        this.knex.raw("MIN(latency) as min_latency"),
      )
      .where("monitor_tag", monitor_tag)
      .where("timestamp", ">=", start)
      .where("timestamp", "<=", end)
      .first();
  }
```

### After (lines 112–129)

```javascript
  //given monitor_tag, start and end timestamp in utc seconds return total degraded, up, down, monitortimeout, maintenance, avg(latency), max(latency), min(latency)
  async getAggregatedMonitoringData(monitor_tag, start, end) {
    return await this.knex("monitoring_data")
      .select(
        this.knex.raw("COUNT(CASE WHEN status = 'DEGRADED' THEN 1 END) as DEGRADED"),
        this.knex.raw("COUNT(CASE WHEN status = 'UP' THEN 1 END) as UP"),
        this.knex.raw("COUNT(CASE WHEN status = 'DOWN' THEN 1 END) as DOWN"),
        this.knex.raw("COUNT(CASE WHEN status = 'MONITORTIMEOUT' THEN 1 END) as MONITORTIMEOUT"),
        this.knex.raw("COUNT(CASE WHEN status = 'MAINTENANCE' THEN 1 END) as MAINTENANCE"),
        this.knex.raw("AVG(latency) as avg_latency"),
        this.knex.raw("MAX(latency) as max_latency"),
        this.knex.raw("MIN(latency) as min_latency"),
      )
      .where("monitor_tag", monitor_tag)
      .where("timestamp", ">=", start)
      .where("timestamp", "<=", end)
      .first();
  }
```

**Change justification:**
- **Purpose:** Ensure any caller of `getAggregatedMonitoringData` receives MONITORTIMEOUT (and MAINTENANCE) counts so MONITORTIMEOUT is a first-class status in aggregates.
- **Risk:** Low; only adds columns. Callers that ignore the new keys are unchanged.
- **Reasoning:** Minimal extension; no change to existing UP/DOWN/DEGRADED logic.

---

### Before (lines 166–174)

```javascript
        this.knex.raw(`
				CASE 
				WHEN SUM(CASE WHEN status = 'DOWN' THEN 1 ELSE 0 END) > 0 THEN 'DOWN'
				WHEN SUM(CASE WHEN status = 'DEGRADED' THEN 1 ELSE 0 END) > 0 THEN 'DEGRADED'
				ELSE 'UP'
				END as status
			`),
```

### After (lines 166–176)

```javascript
        this.knex.raw(`
				CASE 
				WHEN SUM(CASE WHEN status = 'DOWN' THEN 1 ELSE 0 END) > 0 THEN 'DOWN'
				WHEN SUM(CASE WHEN status = 'MONITORTIMEOUT' THEN 1 ELSE 0 END) > 0 THEN 'MONITORTIMEOUT'
				WHEN SUM(CASE WHEN status = 'DEGRADED' THEN 1 ELSE 0 END) > 0 THEN 'DEGRADED'
				ELSE 'UP'
				END as status
			`),
```

**Change justification:**
- **Purpose:** Include MONITORTIMEOUT in `getLastStatusBeforeCombined` so combined status follows priority DOWN > MONITORTIMEOUT > DEGRADED > UP when this function is used.
- **Risk:** Low; only adds a branch; existing callers get correct status when MONITORTIMEOUT is present.
- **Reasoning:** Keeps DB aggregation consistent with application priority; no remap of MONITORTIMEOUT to UP.

---

## File: src/lib/server/cron-minute.js

### Before (line 4)

```javascript
import { UP, DOWN, DEGRADED, REALTIME, TIMEOUT, ERROR, MANUAL, DEFAULT_STATUS } from "./constants.js";
```

### After (line 4)

```javascript
import { UP, DOWN, DEGRADED, REALTIME, TIMEOUT, ERROR, MANUAL, DEFAULT_STATUS, MONITORTIMEOUT } from "./constants.js";
```

### Before (lines 130–132)

```javascript
    if (monitor.default_status !== undefined && monitor.default_status !== null) {
      if ([UP, DOWN, DEGRADED].indexOf(monitor.default_status) !== -1) {
```

### After (lines 130–133)

```javascript
    if (monitor.default_status !== undefined && monitor.default_status !== null) {
      // MONITORTIMEOUT included so config can set default_status to MONITORTIMEOUT (first-class status)
      if ([UP, DOWN, DEGRADED, MONITORTIMEOUT].indexOf(monitor.default_status) !== -1) {
```

**Change justification:**
- **Purpose:** Allow `default_status: "MONITORTIMEOUT"` so MONITORTIMEOUT is a valid config option like UP/DOWN/DEGRADED.
- **Risk:** Low; only extends the allowed list.
- **Reasoning:** Minimal-change; preserves behavior for existing default_status values.

---

## File: src/routes/(kener)/api/today/+server.js

### Before (lines 88–106)

```javascript
export async function GET({ request }) {
  const payload = await request.json();
  const monitor = payload.monitor;
  const now = GetMinuteStartNowTimestampUTC();
  const start = payload.startTs;
  let end = Math.min(payload.startTs + 24 * 60 * 60, now);
  let aggregatedData = db.getDataGroupByMinute(monitor.tag, start, end);
  let ups = aggregatedData.UP;
  let downs = aggregatedData.DOWN;
  let degradeds = aggregatedData.DEGRADED;

  let total = ups + downs + degradeds;
  let uptime = ParseUptime(ups + degradeds, total);
  if (monitor.include_degraded_in_downtime === "YES") {
    uptime = ParseUptime(ups, total);
  }

  return json({ uptime });
}
```

### After (lines 88–112)

```javascript
export async function GET({ request }) {
  const payload = await request.json();
  const monitor = payload.monitor;
  const now = GetMinuteStartNowTimestampUTC();
  const start = payload.startTs;
  let end = Math.min(payload.startTs + 24 * 60 * 60, now);
  const dayData = await db.getMonitoringData(monitor.tag, start, end);
  const aggregatedData = dayData.reduce(
    (acc, row) => {
      acc[row.status] = (acc[row.status] || 0) + 1;
      return acc;
    },
    { UP: 0, DOWN: 0, DEGRADED: 0, MONITORTIMEOUT: 0, MAINTENANCE: 0 },
  );
  let ups = Number(aggregatedData.UP || 0);
  let downs = Number(aggregatedData.DOWN || 0);
  let degradeds = Number(aggregatedData.DEGRADED || 0);
  let monitortimeouts = Number(aggregatedData.MONITORTIMEOUT || 0);
  let total = ups + downs + degradeds + monitortimeouts;
  let uptime = ParseUptime(ups + degradeds, total);
  if (monitor.include_degraded_in_downtime === "YES") {
    uptime = ParseUptime(ups, total);
  }
  return json({ uptime });
}
```

**Change justification:**
- **Purpose:** Fix GET handler (replace non-existent `getDataGroupByMinute` with `getMonitoringData` + reduce) and include MONITORTIMEOUT in aggregation and total so uptime denominator is correct.
- **Risk:** Low; GET now returns correct uptime when MONITORTIMEOUT (or MAINTENANCE) minutes exist.
- **Reasoning:** Aligns GET with POST and ensures MONITORTIMEOUT is first-class in today API.

---

## File: src/lib/server/webhook.js

### Before (lines 129–151)

```javascript
  let dayDataNew = await db.getMonitoringData(tag, start, now);
  let ups = 0;
  let downs = 0;
  let degradeds = 0;
  let lastData = dayDataNew[dayDataNew.length - 1];

  for (let i = 0; i < dayDataNew.length; i++) {
    let row = dayDataNew[i];
    if (row.status == "UP") {
      ups++;
    } else if (row.status == "DEGRADED") {
      degradeds++;
    } else if (row.status == "DOWN") {
      downs++;
    }
  }
  let numerator = ups + degradeds;
  if (include_degraded_in_downtime === "YES") {
    numerator = ups;
  }

  resp.uptime = ParseUptime(numerator, ups + degradeds + downs);
```

### After (lines 129–154)

```javascript
  let dayDataNew = await db.getMonitoringData(tag, start, now);
  let ups = 0;
  let downs = 0;
  let degradeds = 0;
  let monitortimeouts = 0;
  let lastData = dayDataNew[dayDataNew.length - 1];

  for (let i = 0; i < dayDataNew.length; i++) {
    let row = dayDataNew[i];
    if (row.status == "UP") {
      ups++;
    } else if (row.status == "DEGRADED") {
      degradeds++;
    } else if (row.status == "DOWN") {
      downs++;
    } else if (row.status == "MONITORTIMEOUT") {
      monitortimeouts++;
    }
  }
  let numerator = ups + degradeds;
  if (include_degraded_in_downtime === "YES") {
    numerator = ups;
  }
  // Include MONITORTIMEOUT in total so uptime denominator reflects all status minutes (first-class status)
  const total = ups + degradeds + downs + monitortimeouts;
  resp.uptime = ParseUptime(numerator, total);
```

**Change justification:**
- **Purpose:** Count MONITORTIMEOUT minutes and include them in the uptime denominator so `GetMonitorStatusByTag` returns correct uptime when the day has MONITORTIMEOUT.
- **Risk:** Low; numerator logic unchanged; denominator grows only when MONITORTIMEOUT exists.
- **Reasoning:** MONITORTIMEOUT is not treated as “up”; it is correctly included in total minutes.

---

## File: src/lib/components/monitor.svelte

### Before (lines 37–57)

```svelte
  let _0Day = {};
  // Reactive so grid uses latest pageData when monitor updates (fixes "never recovers" after MONITORTIMEOUT).
  $: _90Day = monitor.pageData._90Day;
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

  /** Inline background from CSS variable so MONITORTIMEOUT (and others) always show even if class is purged or overridden. */
  function getBarBackgroundStyle(cssClass) {
    const c = cssClass || "";
    const m = c.match(/^api-(up|down|degraded|maintenance|monitortimeout)$/);
    return m ? `background-color: var(--${m[1]}-color);` : "";
  }
```

### After (lines 37–75)

```svelte
  let _0Day = {};
  // Reactive so grid uses latest pageData when monitor updates (fixes grid disappearing after MONITORTIMEOUT → UP/DOWN/DEGRADED).
  $: _90Day = monitor?.pageData?._90Day ?? {};
  $: uptime90Day = monitor?.pageData?.uptime90Day ?? "-";
  let incidents = {};
  let dayIncidentsFull = [];
  $: homeDataMaxDays = monitor?.pageData?.homeDataMaxDays ?? { maxDays: 90, selectableDays: [30, 90] };

  // Reactive so cell width updates when homeDataMaxDays or viewport changes; safe fallback avoids division by undefined.
  $: dimension = (() => {
    const maxDays = homeDataMaxDays?.maxDays ?? 90;
    const x1 = ($page?.data?.isMobile ? 346 : 546) / maxDays;
    return { x1, x2: (4 / 6) * x1 };
  })();

  // Fallback hex ensures MONITORTIMEOUT (and others) show when CSS variable is missing or out of scope.
  const STATUS_COLOR_FALLBACKS = {
    up: "#00dfa2",
    down: "#ca3038",
    degraded: "#e6ca61",
    maintenance: "#6679cc",
    monitortimeout: "#000080",
  };
  function getBarBackgroundStyle(cssClass) {
    const c = cssClass || "";
    const m = c.match(/^api-(up|down|degraded|maintenance|monitortimeout)$/);
    if (!m) return "";
    const name = m[1];
    const fallback = STATUS_COLOR_FALLBACKS[name] ?? "#f1f5f8";
    return `background-color: var(--${name}-color, ${fallback});`;
  }
```

**Change justification:**
- **Purpose:** (1) Safe reactive `_90Day`/`uptime90Day`/`homeDataMaxDays` so the grid does not throw when `pageData` is undefined. (2) Reactive `dimension` with fallback so layout is correct after data refresh. (3) Inline bar style with `var(--X-color, fallback)` so MONITORTIMEOUT (and others) always have a visible color.
- **Risk:** Low; fallbacks only apply when values are missing; existing behavior unchanged when data is present.
- **Reasoning:** Prevents grid disappearance and transparent bars; backward compatible.

---

### Before (lines 364–367)

```svelte
        <div class="daygrid90 flex min-h-[60px] justify-between overflow-x-auto overflow-y-hidden py-1">
          {#each Object.entries(_90Day) as [ts, bar]}
            <button
```

### After (lines 375–378)

```svelte
        <div class="daygrid90 flex min-h-[60px] justify-between overflow-x-auto overflow-y-hidden py-1">
          {#key _90Day}
          {#each Object.entries(_90Day) as [ts, bar] (ts)}
            <button
```

### Before (lines 412–416)

```svelte
            {/if}
          {/each}
        </div>
        {#if (monitor.monitor_type === "GROUP" || monitor.monitor_type === "SUPERGROUP") && !!!embed}
```

### After (lines 421–426)

```svelte
            {/if}
          {/each}
          {/key}
        </div>
        {#if (monitor.monitor_type === "GROUP" || monitor.monitor_type === "SUPERGROUP") && !!!embed}
```

**Change justification:**
- **Purpose:** Force full re-render of the grid when `_90Day` is replaced (e.g. after MONITORTIMEOUT → UP) and stabilize each bar by key `(ts)` so the grid does not disappear or show wrong cells.
- **Risk:** Low; only affects list reconciliation and re-creation when data reference changes.
- **Reasoning:** Minimal change to fix grid recovery; no change to data or business logic.

---

### Before (line 381)

```svelte
                class="oneline-in h-[30px] bg-{bar.cssClass || 'api-nodata'} mx-auto rounded-{monitor.pageData.barRoundness.toUpperCase() ==
```

### After (line 390)

```svelte
                class="oneline-in h-[30px] bg-{bar.cssClass || 'api-nodata'} mx-auto rounded-{(monitor?.pageData?.barRoundness ?? 'ROUND').toUpperCase() ==
```

**Change justification:**
- **Purpose:** Avoid throwing when `pageData` or `barRoundness` is undefined during load or refresh.
- **Risk:** Low; default `'ROUND'` matches typical config.
- **Reasoning:** Defensive; preserves existing behavior when data is present.

---

## Summary

| File | Sections updated |
|------|-------------------|
| `src/lib/server/db/dbimpl.js` | getAggregatedMonitoringData (MONITORTIMEOUT, MAINTENANCE); getLastStatusBeforeCombined (CASE) |
| `src/lib/server/cron-minute.js` | Import MONITORTIMEOUT; default_status allow list |
| `src/routes/(kener)/api/today/+server.js` | GET: getMonitoringData + reduce; MONITORTIMEOUT in aggregation and total |
| `src/lib/server/webhook.js` | GetMonitorStatusByTag: count MONITORTIMEOUT; total includes monitortimeouts |
| `src/lib/components/monitor.svelte` | Reactive _90Day/uptime90Day/homeDataMaxDays; reactive dimension; getBarBackgroundStyle fallbacks; {#key _90Day} and (ts); barRoundness optional chaining |

All changes are minimal, backward-compatible, and documented with inline comments where behavior is non-obvious.
