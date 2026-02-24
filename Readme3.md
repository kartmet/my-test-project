# MONITORTIMEOUT Implementation Guide

This document lists all code changes with **before/after** snippets and **line numbers** for adding the MONITORTIMEOUT status (navy color) to the monitoring platform.

---

## 1. `src/lib/server/constants.js`

### Add constant (after line 12)
**Before:**
```javascript
const NO_DATA = "NO_DATA";
const API_TIMEOUT = 10 * 1000;
```
**After:**
```javascript
const NO_DATA = "NO_DATA";
const MONITORTIMEOUT = "MONITORTIMEOUT";
const API_TIMEOUT = 10 * 1000;
```

### Export (lines 108–127)
**Before:** Export list does not include `MAINTENANCE` or `MONITORTIMEOUT`.
**After:** Add `MAINTENANCE,` and `MONITORTIMEOUT,` to the export list (e.g. after `NO_DATA,`).

---

## 2. `src/lib/server/tool.js`

### StatusObj (lines 148–154)
**Before:**
```javascript
const StatusObj = {
  UP: "api-up",
  DEGRADED: "api-degraded",
  DOWN: "api-down",
  MAINTENANCE: "api-maintenance",
  NO_DATA: "api-nodata",
};
```
**After:**
```javascript
const StatusObj = {
  UP: "api-up",
  DEGRADED: "api-degraded",
  DOWN: "api-down",
  MAINTENANCE: "api-maintenance",
  NO_DATA: "api-nodata",
  MONITORTIMEOUT: "api-monitor-timeout",
};
```

---

## 3. `src/lib/server/services/apiCall.js`

### Import (line 4)
**Before:**
```javascript
import { UP, DOWN, DEGRADED, REALTIME, TIMEOUT, ERROR, MANUAL } from "../constants.js";
```
**After:**
```javascript
import { UP, DOWN, DEGRADED, MONITORTIMEOUT, REALTIME, TIMEOUT, ERROR, MANUAL } from "../constants.js";
```

### Eval validation (lines 132–135)
**Before:**
```javascript
} else if (
  evalResp.status === undefined ||
  evalResp.status === null ||
  [UP, DOWN, DEGRADED].indexOf(evalResp.status) === -1
) {
```
**After:**
```javascript
} else if (
  evalResp.status === undefined ||
  evalResp.status === null ||
  [UP, DOWN, DEGRADED, MONITORTIMEOUT].indexOf(evalResp.status) === -1
) {
```

### Optional: copy CustomTooltipMessage / CustomMonitorLink (after line 158)
**Before:**
```javascript
    if (evalResp.type !== undefined && evalResp.type !== null) {
      toWrite.type = evalResp.type;
    }
    if (timeoutError) {
```
**After:**
```javascript
    if (evalResp.type !== undefined && evalResp.type !== null) {
      toWrite.type = evalResp.type;
    }
    if (evalResp.CustomTooltipMessage !== undefined && evalResp.CustomTooltipMessage !== null) {
      toWrite.custom_tooltip_message = evalResp.CustomTooltipMessage;
    }
    if (evalResp.CustomMonitorLink !== undefined && evalResp.CustomMonitorLink !== null) {
      toWrite.custom_monitor_link = evalResp.CustomMonitorLink;
    }
    if (timeoutError) {
```
*(Note: Persisting these requires DB columns and updates to insertMonitoringData/InsertMonitoringData.)*

---

## 4. `src/lib/color.js`

### Returned object (lines 6–11)
**Before:**
```javascript
	return {
		UP: site.colors?.UP || "#00dfa2",
		DEGRADED: site.colors?.DEGRADED || "#e6ca61",
		DOWN: site.colors?.DOWN || "#ca3038",
		MAINTENANCE: site.colors?.MAINTENANCE || "#6679cc",
		NO_DATA: "#f1f5f8"
	};
```
**After:**
```javascript
	return {
		UP: site.colors?.UP || "#00dfa2",
		DEGRADED: site.colors?.DEGRADED || "#e6ca61",
		DOWN: site.colors?.DOWN || "#ca3038",
		MAINTENANCE: site.colors?.MAINTENANCE || "#6679cc",
		MONITORTIMEOUT: site.colors?.MONITORTIMEOUT || "#000080",
		NO_DATA: "#f1f5f8"
	};
```

---

## 5. `src/kener.css`

### :root (lines 9–14)
**Before:**
```css
:root {
  --up-color: #4ead94;
  --down-color: #ca3038;
  --degraded-color: #e6ca61;
  --maintenance-color: #6679cc;
}
```
**After:**
```css
:root {
  --up-color: #4ead94;
  --down-color: #ca3038;
  --degraded-color: #e6ca61;
  --maintenance-color: #6679cc;
  --monitor-timeout-color: #000080;
}
```

### New classes (after line 109, after `.bg-api-maintenance-90`)
**Add:**
```css
/* MONITORTIMEOUT - navy */
.bg-api-monitor-timeout {
  background-color: var(--monitor-timeout-color);
}
.text-api-monitor-timeout {
  color: var(--monitor-timeout-color);
}
```

---

## 6. `src/lib/server/controllers/controller.js`

### Import (line 21)
**Before:**
```javascript
import { DEGRADED, DOWN, NO_DATA, SIGNAL, UP, REALTIME } from "../constants.js";
```
**After:**
```javascript
import { DEGRADED, DOWN, MAINTENANCE, MONITORTIMEOUT, NO_DATA, SIGNAL, UP, REALTIME } from "../constants.js";
```

### System message – add counter (after line 268)
**Before:**
```javascript
  let upsCount = 0;
  let degradedCount = 0;
  let downCount = 0;
  let maintenanceCount = 0;
  for (let i = 0; ...
```
**After:**
```javascript
  let upsCount = 0;
  let degradedCount = 0;
  let downCount = 0;
  let maintenanceCount = 0;
  let monitorTimeoutCount = 0;
  for (let i = 0; ...
```

### System message – count MONITORTIMEOUT in loop (after line 279)
**Before:**
```javascript
      } else if (status.status === "MAINTENANCE") {
        maintenanceCount++;
      }
    }
  }

  const total = ...
```
**After:**
```javascript
      } else if (status.status === "MAINTENANCE") {
        maintenanceCount++;
      } else if (status.status === "MONITORTIMEOUT") {
        monitorTimeoutCount++;
      }
    }
  }

  const total = ...
```

### System message – total (line 285)
**Before:**
```javascript
  const total = upsCount + degradedCount + downCount + maintenanceCount;
```
**After:**
```javascript
  const total = upsCount + degradedCount + downCount + maintenanceCount + monitorTimeoutCount;
```

### System message – percentage and return (after line 299, and return object)
**Before:**
```javascript
  let maintenancePercentage = Math.round((maintenanceCount / total) * 100);

  let message = "";
```
**After:**
```javascript
  let maintenancePercentage = Math.round((maintenanceCount / total) * 100);
  let monitorTimeoutPercentage = total > 0 ? Math.round((monitorTimeoutCount / total) * 100) : 0;

  let message = "";
```

Add message branches for MONITORTIMEOUT (e.g. "All Systems Timeout", "Some Systems Timeout") and in the percentage normalization (line 333) include `monitorTimeoutPercentage`. In the return object (lines 336–342) add `monitorTimeoutPercentage`.

**Before (return):**
```javascript
  return {
    text: message,
    upsPercentage,
    degradedPercentage,
    downsPercentage,
    maintenancePercentage,
  };
```
**After:**
```javascript
  return {
    text: message,
    upsPercentage,
    degradedPercentage,
    downsPercentage,
    maintenancePercentage,
    monitorTimeoutPercentage,
  };
```

### GetLatestStatusActiveAll (lines 449–456)
**Before:**
```javascript
    if (latestData[i].status === "DOWN") {
      status = "DOWN";
    } else if (latestData[i].status === "DEGRADED" && status !== "DOWN") {
      status = "DEGRADED";
    } else if (latestData[i].status === "UP" && status !== "DOWN" && status !== "DEGRADED") {
      status = "UP";
    }
```
**After:**
```javascript
    if (latestData[i].status === "DOWN") {
      status = "DOWN";
    } else if (latestData[i].status === "DEGRADED" && status !== "DOWN") {
      status = "DEGRADED";
    } else if (latestData[i].status === "MONITORTIMEOUT" && status !== "DOWN" && status !== "DEGRADED") {
      status = "MONITORTIMEOUT";
    } else if (latestData[i].status === "UP" && status !== "DOWN" && status !== "DEGRADED" && status !== "MONITORTIMEOUT") {
      status = "UP";
    }
```

### calculateAggregatedStatus – statusCounts (line 499)
**Before:**
```javascript
  let statusCounts = { UP: 0, DEGRADED: 0, DOWN: 0, MAINTENANCE: 0, NO_DATA: 0 };
```
**After:**
```javascript
  let statusCounts = { UP: 0, DEGRADED: 0, DOWN: 0, MAINTENANCE: 0, MONITORTIMEOUT: 0, NO_DATA: 0 };
```

### calculateAggregatedStatus – priority (lines 569–575)
**Before:**
```javascript
  if (statusCounts.DOWN > 0) {
    aggregatedStatus = DOWN;
  } else if (statusCounts.DEGRADED > 0) {
    aggregatedStatus = DEGRADED;
  } else if (statusCounts.MAINTENANCE > 0) {
```
**After:**
```javascript
  if (statusCounts.DOWN > 0) {
    aggregatedStatus = DOWN;
  } else if (statusCounts.DEGRADED > 0) {
    aggregatedStatus = DEGRADED;
  } else if (statusCounts.MONITORTIMEOUT > 0) {
    aggregatedStatus = MONITORTIMEOUT;
  } else if (statusCounts.MAINTENANCE > 0) {
```

### calculateAggregatedStatus – log (line 587)
**Before:** Log string lists UP, DEGRADED, DOWN, NO_DATA.
**After:** Add `, MONITORTIMEOUT: ${statusCounts.MONITORTIMEOUT}` to the log string.

### GetDataGroupByDayAlternative – initial group (lines 1006–1014)
**Before:**
```javascript
        acc[dayGroup] = {
          timestamp: dayGroup * 86400 - offsetSeconds,
          total: 0,
          UP: 0,
          DOWN: 0,
          DEGRADED: 0,
          MAINTENANCE: 0,
          NO_DATA: 0,
        };
```
**After:**
```javascript
        acc[dayGroup] = {
          timestamp: dayGroup * 86400 - offsetSeconds,
          total: 0,
          UP: 0,
          DOWN: 0,
          DEGRADED: 0,
          MAINTENANCE: 0,
          MONITORTIMEOUT: 0,
          NO_DATA: 0,
        };
```

### GetDataGroupByDayAlternative – return (lines 1025–1033)
**Before:**
```javascript
  return Object.values(groupedData).map((group) => ({
    timestamp: group.timestamp,
    total: group.total,
    UP: group.UP,
    DOWN: group.DOWN,
    DEGRADED: group.DEGRADED,
    MAINTENANCE: group.MAINTENANCE,
    NO_DATA: group.NO_DATA,
  }));
```
**After:**
```javascript
  return Object.values(groupedData).map((group) => ({
    timestamp: group.timestamp,
    total: group.total,
    UP: group.UP,
    DOWN: group.DOWN,
    DEGRADED: group.DEGRADED,
    MAINTENANCE: group.MAINTENANCE,
    MONITORTIMEOUT: group.MONITORTIMEOUT ?? 0,
    NO_DATA: group.NO_DATA,
  }));
```

---

## 7. `src/lib/server/db/dbimpl.js`

### getAggregatedMonitoringData (lines 114–119)
**Before:**
```javascript
      .select(
        this.knex.raw("COUNT(CASE WHEN status = 'DEGRADED' THEN 1 END) as DEGRADED"),
        this.knex.raw("COUNT(CASE WHEN status = 'UP' THEN 1 END) as UP"),
        this.knex.raw("COUNT(CASE WHEN status = 'DOWN' THEN 1 END) as DOWN"),
        this.knex.raw("AVG(latency) as avg_latency"),
```
**After:**
```javascript
      .select(
        this.knex.raw("COUNT(CASE WHEN status = 'DEGRADED' THEN 1 END) as DEGRADED"),
        this.knex.raw("COUNT(CASE WHEN status = 'UP' THEN 1 END) as UP"),
        this.knex.raw("COUNT(CASE WHEN status = 'DOWN' THEN 1 END) as DOWN"),
        this.knex.raw("COUNT(CASE WHEN status = 'MONITORTIMEOUT' THEN 1 END) as MONITORTIMEOUT"),
        this.knex.raw("AVG(latency) as avg_latency"),
```

### getLastStatusBeforeCombined (lines 168–173)
**Before:**
```javascript
        this.knex.raw(`
				CASE 
				WHEN SUM(CASE WHEN status = 'DOWN' THEN 1 ELSE 0 END) > 0 THEN 'DOWN'
				WHEN SUM(CASE WHEN status = 'DEGRADED' THEN 1 ELSE 0 END) > 0 THEN 'DEGRADED'
				ELSE 'UP'
				END as status
			`),
```
**After:**
```javascript
        this.knex.raw(`
				CASE 
				WHEN SUM(CASE WHEN status = 'DOWN' THEN 1 ELSE 0 END) > 0 THEN 'DOWN'
				WHEN SUM(CASE WHEN status = 'DEGRADED' THEN 1 ELSE 0 END) > 0 THEN 'DEGRADED'
				WHEN SUM(CASE WHEN status = 'MONITORTIMEOUT' THEN 1 ELSE 0 END) > 0 THEN 'MONITORTIMEOUT'
				ELSE 'UP'
				END as status
			`),
```

---

## 8. `src/lib/server/page.js`

### Day loop – total and MONITORTIMEOUT block (after line 104, and between MAINTENANCE and NO_DATA)
**Before:**
```javascript
  let totalDegradedCount = 0;
  let totalDownCount = 0;
  let totalUpCount = 0;
  ...
    if (dayData.MAINTENANCE > 0) {
      cssClass = returnStatusClass(dayData.MAINTENANCE, dayData.total, StatusObj.MAINTENANCE, site.barStyle);
      ...
    }
    if (dayData.NO_DATA === dayData.total) {
```
**After:** Add `let totalMonitorTimeoutCount = 0;` and in the loop `totalMonitorTimeoutCount += dayData.MONITORTIMEOUT || 0;`. Then add after the MAINTENANCE block:
```javascript
    if ((dayData.MONITORTIMEOUT || 0) > 0) {
      cssClass = returnStatusClass(dayData.MONITORTIMEOUT, dayData.total, StatusObj.MONITORTIMEOUT, site.barStyle);
      summaryDuration = getSummaryDuration(dayData.MONITORTIMEOUT, selectedLang);
      summaryStatus = "MONITORTIMEOUT";
    }
    if (dayData.NO_DATA === dayData.total) {
```

### CURRENT summary – MONITORTIMEOUT (after line 188, before NO_DATA check)
**Before:**
```javascript
    if (!!lastRow && lastRow.status == "MAINTENANCE") {
      ...
      summaryColorClass = "api-maintenance";
    }
    if (lastRow.status === "NO_DATA") {
```
**After:**
```javascript
    if (!!lastRow && lastRow.status == "MAINTENANCE") {
      ...
      summaryColorClass = "api-maintenance";
    }
    if (!!lastRow && lastRow.status == "MONITORTIMEOUT") {
      summaryDuration = getSummaryDuration(getCountOfSimilarStatuesEnd(todayDataDb, "MONITORTIMEOUT"), selectedLang);
      summaryStatus = "MONITORTIMEOUT";
      summaryColorClass = "api-monitor-timeout";
    }
    if (lastRow && lastRow.status === "NO_DATA") {
```

*(Optional: add `totalMonitorTimeoutCount` to `uptime90DayDenominator` so MONITORTIMEOUT counts as non-up.)*

---

## 9. `src/routes/(kener)/api/today/+server.js`

### Bug fix (line 64)
**Before:**
```javascript
    if (status == "DOWN") {
      cssClass = StatusObj;
    }
```
**After:**
```javascript
    if (status == "DOWN") {
      cssClass = StatusObj.DOWN;
    }
```

### Reducer initial value (lines 39–48)
**Before:**
```javascript
    {
      UP: 0,
      DOWN: 0,
      DEGRADED: 0,
    },
```
**After:**
```javascript
    {
      UP: 0,
      DOWN: 0,
      DEGRADED: 0,
      MAINTENANCE: 0,
      MONITORTIMEOUT: 0,
    },
```

---

## 10. `src/routes/(kener)/api/today/aggregated/+server.js`

### Reducer (lines 20–27)
**Before:**
```javascript
    {
      UP: 0,
      DOWN: 0,
      DEGRADED: 0,
    },
```
**After:**
```javascript
    {
      UP: 0,
      DOWN: 0,
      DEGRADED: 0,
      MAINTENANCE: 0,
      MONITORTIMEOUT: 0,
    },
```
*(Adjust total and uptime if MONITORTIMEOUT should not count as up.)*

---

## 11. `src/routes/(kener)/+layout.svelte`

### CSS variable (lines 107–114)
**Before:**
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
**After:**
```html
  style="
	--font-family: {data.site.font.family};
	--bg-custom: {data.bgc};
	--up-color: {data.site.colors.UP};
	--down-color: {data.site.colors.DOWN};
	--degraded-color: {data.site.colors.DEGRADED};
	--maintenance-color: {data.site.colors.MAINTENANCE};
	--monitor-timeout-color: {data.site.colors.MONITORTIMEOUT || '#000080'};
	"
```

---

## 12. `src/routes/(kener)/+page.svelte`

### Status bar segment (lines 117–120)
**Before:**
```svelte
      <div class="bg-api-up h-4" style="width: {data.systemDataMessage.upsPercentage}%;"></div>
      <div class="bg-api-degraded h-4" style="width: {data.systemDataMessage.degradedPercentage}%;"></div>
      <div class="bg-api-down h-4" style="width: {data.systemDataMessage.downsPercentage}%;"></div>
      <div class="bg-api-maintenance h-4" style="width: {data.systemDataMessage.maintenancePercentage}%;"></div>
```
**After:**
```svelte
      <div class="bg-api-up h-4" style="width: {data.systemDataMessage.upsPercentage}%;"></div>
      <div class="bg-api-degraded h-4" style="width: {data.systemDataMessage.degradedPercentage}%;"></div>
      <div class="bg-api-down h-4" style="width: {data.systemDataMessage.downsPercentage}%;"></div>
      <div class="bg-api-maintenance h-4" style="width: {data.systemDataMessage.maintenancePercentage}%;"></div>
      <div class="bg-api-monitor-timeout h-4" style="width: {data.systemDataMessage.monitorTimeoutPercentage || 0}%;"></div>
```

### Legend (lines 243–250)
**Before:**
```svelte
          <span class="bg-api-maintenance mr-1 inline-flex h-[8px] w-[8px] rounded-full opacity-75"></span>
          <span class="">
            {l(data.lang, "MAINTENANCE")}
          </span>
        </Badge>
```
**After:**
```svelte
          <span class="bg-api-maintenance mr-1 inline-flex h-[8px] w-[8px] rounded-full opacity-75"></span>
          <span class="mr-3">
            {l(data.lang, "MAINTENANCE")}
          </span>

          <span class="bg-api-monitor-timeout mr-1 inline-flex h-[8px] w-[8px] rounded-full opacity-75"></span>
          <span class="">
            {l(data.lang, "MONITORTIMEOUT")}
          </span>
        </Badge>
```

---

## 13. `src/lib/server/webhook.js`

### Status validation (line 57)
**Before:**
```javascript
  if (data.status === undefined || ["UP", "DOWN", "DEGRADED"].indexOf(data.status) === -1) {
```
**After:**
```javascript
  if (data.status === undefined || ["UP", "DOWN", "DEGRADED", "MONITORTIMEOUT"].indexOf(data.status) === -1) {
```

---

## 14. `src/lib/server/cron-minute.js`

### Import (line 4)
**Before:**
```javascript
import { UP, DOWN, DEGRADED, REALTIME, TIMEOUT, ERROR, MANUAL, DEFAULT_STATUS } from "./constants.js";
```
**After:**
```javascript
import { UP, DOWN, DEGRADED, MONITORTIMEOUT, REALTIME, TIMEOUT, ERROR, MANUAL, DEFAULT_STATUS } from "./constants.js";
```

### default_status (lines 131–132)
**Before:**
```javascript
    if (monitor.default_status !== undefined && monitor.default_status !== null) {
      if ([UP, DOWN, DEGRADED].indexOf(monitor.default_status) !== -1) {
```
**After:**
```javascript
    if (monitor.default_status !== undefined && monitor.default_status !== null) {
      if ([UP, DOWN, DEGRADED, MONITORTIMEOUT].indexOf(monitor.default_status) !== -1) {
```

---

## 15. `src/lib/components/manage/themeInfo.svelte`

### Default MONITORTIMEOUT color (after lines 54–56)
**Before:**
```javascript
  if (data.siteData.colors) {
    if (data.siteData.colors.MAINTENANCE === undefined) {
      data.siteData.colors.MAINTENANCE = "#6679cc";
    }
    themeData.colorsJ = JSON.parse(JSON.stringify(data.siteData.colors));
  }
```
**After:**
```javascript
  if (data.siteData.colors) {
    if (data.siteData.colors.MAINTENANCE === undefined) {
      data.siteData.colors.MAINTENANCE = "#6679cc";
    }
    if (data.siteData.colors.MONITORTIMEOUT === undefined) {
      data.siteData.colors.MONITORTIMEOUT = "#000080";
    }
    themeData.colorsJ = JSON.parse(JSON.stringify(data.siteData.colors));
  }
```

---

## 16. `src/lib/server/notification/email.js`

### Color branch (lines 46–54)
**Before:**
```javascript
    let bgColor = "#f4f4f4";
    if (data.severity == "critical") {
      bgColor = this.siteData.colors.DOWN;
    } else if (data.severity == "warning") {
      bgColor = this.siteData.colors.DEGRADED;
    }

    if (data.status === "RESOLVED") {
      bgColor = this.siteData.colors.UP;
    }
```
**After:** Add a branch for MONITORTIMEOUT:
```javascript
    let bgColor = "#f4f4f4";
    if (data.severity == "critical") {
      bgColor = this.siteData.colors.DOWN;
    } else if (data.severity == "warning") {
      bgColor = this.siteData.colors.DEGRADED;
    } else if (data.status === "MONITORTIMEOUT" || data.severity == "timeout") {
      bgColor = this.siteData.colors.MONITORTIMEOUT || "#000080";
    }

    if (data.status === "RESOLVED") {
      bgColor = this.siteData.colors.UP;
    }
```

---

## 17. `src/routes/(embed)/+layout.svelte` (optional)

### CSS variable (lines 58–60)
**Before:**
```html
	--up-color: {data.site.colors.UP};
	--down-color: {data.site.colors.DOWN};
	--degraded-color: {data.site.colors.DEGRADED}
```
**After:**
```html
	--up-color: {data.site.colors.UP};
	--down-color: {data.site.colors.DOWN};
	--degraded-color: {data.site.colors.DEGRADED};
	--monitor-timeout-color: {data.site.colors.MONITORTIMEOUT || '#000080'}
```

---

## Summary Table

| # | File | Section | Line(s) |
|---|------|---------|---------|
| 1 | `constants.js` | Add constant, export | 12, 108–127 |
| 2 | `tool.js` | StatusObj | 148–154 |
| 3 | `apiCall.js` | Import, eval validation, optional toWrite | 4, 132–135, after 158 |
| 4 | `color.js` | MONITORTIMEOUT color | 6–11 |
| 5 | `kener.css` | :root, new classes | 9–14, after 109 |
| 6 | `controller.js` | Import, system message, GetLatestStatusActiveAll, statusCounts, priority, GetDataGroupByDayAlternative, log | 21, 268–342, 446–461, 499, 569–587, 1002–1033 |
| 7 | `dbimpl.js` | getAggregatedMonitoringData, getLastStatusBeforeCombined | 114–119, 168–173 |
| 8 | `page.js` | Day loop, CURRENT summary | 104–193 |
| 9 | `api/today/+server.js` | Bug fix, reducer | 64, 39–48 |
| 10 | `api/today/aggregated/+server.js` | Reducer | 20–27 |
| 11 | `(kener)/+layout.svelte` | --monitor-timeout-color | 107–114 |
| 12 | `(kener)/+page.svelte` | Status bar, legend | 117–120, 243–250 |
| 13 | `webhook.js` | Status validation | 57 |
| 14 | `cron-minute.js` | Import, default_status | 4, 131–132 |
| 15 | `themeInfo.svelte` | Default MONITORTIMEOUT color | 54–56 |
| 16 | `notification/email.js` | MONITORTIMEOUT branch | 46–54 |
| 17 | `(embed)/+layout.svelte` | Optional CSS var | 58–60 |

---

## Eval Return Shape

Eval can return:
```javascript
{
  status: "MONITORTIMEOUT",
  CustomTooltipMessage: "Optional custom tooltip text",
  CustomMonitorLink: "https://example.com/dashboard"
}
```
Optional: persist `CustomTooltipMessage` and `CustomMonitorLink` via new DB columns and pass them through to the UI for per-cell tooltip/link.
