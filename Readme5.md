# MONITORTIMEOUT and Heatmap Refresh Behavior – Analysis

This document analyzes whether the MONITORTIMEOUT implementation affects the existing heatmap refresh behavior (show previous status until next scheduled poll). **No code changes are implemented here**; the analysis and any recommended fixes are documented only.

---

## Current Expected Behavior (Must Not Change)

- Each monitor runs on a **fixed schedule** (e.g. every 5 minutes via `monitor.cron`).
- If a monitor last ran at 9:05, the heatmap grid must continue showing the **9:05 status** until the next scheduled run at 9:10.
- The heatmap grid must **not** update in real time.
- The heatmap grid must only update when the next poll completes and new data is persisted, and the user subsequently loads or refreshes the page.

---

## 1. Will MONITORTIMEOUT Changes Affect “Show Previous Status Until Next Poll”?

**Answer: No.**

**Reasoning:**

- **When heatmap data is produced:** The heatmap is built **only at page load**. In `src/routes/(kener)/+page.server.js`, the `load` function runs when the user navigates to the status page (or refreshes). It calls `FetchData(siteData, monitors[i], ...)` for each monitor and assigns `monitors[i].pageData = data`. There is **no** client-side polling, WebSocket, or live refetch of `pageData` or `_90Day`.
- **Where the heatmap reads from:** `src/lib/components/monitor.svelte` uses `let _90Day = monitor.pageData._90Day` (line 38). That value comes from the server-rendered payload and does not change until the user triggers a new load (navigate or refresh).
- **When new status appears:** New status (including MONITORTIMEOUT) is written to `monitoring_data` when the **cron** runs `Minuter(monitor)` at the next scheduled time. The heatmap will show that new status only on the **next** page load/refresh, which is exactly “show previous status until next poll” (the “next poll” has completed and new data is in the DB when the user next loads).

MONITORTIMEOUT is just another status value in the same persistence and read path as UP/DOWN/DEGRADED/MAINTENANCE. It does not introduce any real-time or more frequent update mechanism. So the existing “show previous status until next poll” behavior is **unchanged**.

---

## 2. Will MONITORTIMEOUT Changes Interfere with the Cron-Based Refresh Cycle?

**Answer: No.**

**Reasoning:**

- **Scheduler:** `src/lib/server/startup.js` uses `Cron(monitor.cron, async (job) => { await Minuter(monitor); })`. The schedule is driven only by `monitor.cron`; MONITORTIMEOUT does not alter cron or when `Minuter` runs.
- **Minuter:** `src/lib/server/cron-minute.js` runs at `startOfMinute = GetMinuteStartNowTimestampUTC()`, calls `serviceClient.execute()`, then writes whatever `element.status` is via `InsertMonitoringData({ ..., status: element.status, ... })`. MONITORTIMEOUT is just one more allowed value for `status`; it is not special and does not change the timing of the job.
- **InsertMonitoringData:** Pushes to a queue, calls `ProcessGroupUpdate(data)`, then `db.insertMonitoringData(data)` with `data.status` unchanged. No schedule or cron logic is touched.

So the cron-based refresh cycle is **unchanged**.

---

## 3. Will MONITORTIMEOUT Changes Cause the Heatmap to Update Prematurely or Incorrectly?

**Answer: No (for timing). Incorrect display (e.g. DOWN instead of MONITORTIMEOUT) can occur only if persistence or aggregation is wrong.**

**Reasoning:**

- **Timing:** The heatmap still updates only when the page is loaded/refreshed. There is no new mechanism that would update the grid before the next poll or before the user reloads. So no **premature** update.
- **Correctness:** The heatmap will show MONITORTIMEOUT (navy) only if:
  1. Eval returns MONITORTIMEOUT and it is **persisted** as `status: 'MONITORTIMEOUT'` in `monitoring_data`.
  2. When building the 90-day data, MONITORTIMEOUT is **not** overwritten or lost (e.g. interpolation using previous period’s status, grouping producing NaN, or priority order putting DEGRADED above MONITORTIMEOUT).

If the application previously showed DOWN instead of MONITORTIMEOUT, the cause was in (2); the fixes in MONITORTIMEOUT_DOWN_OVERRIDE_README.md (NO_DATA for interpolation, defensive grouping, priority MONITORTIMEOUT before DEGRADED) address that. They do not change **when** the heatmap updates.

---

## 4. Will MONITORTIMEOUT Changes Cause the Heatmap to Show DOWN Instead of MONITORTIMEOUT?

**Answer: Not if the current implementation is correct. Historically, DOWN appeared because of interpolation/grouping/priority; those have been fixed.**

**Relevant code paths:**

| Layer | Behavior | MONITORTIMEOUT override? |
|-------|----------|---------------------------|
| **apiCall.js** | Eval return is validated against `[UP, DOWN, DEGRADED, MONITORTIMEOUT]`. If eval returns MONITORTIMEOUT, `toWrite.status = evalResp.status` (line 151). | **No** override to DOWN. |
| **cron-minute.js** | Writes `element.status` from mergedData to InsertMonitoringData. No mapping of status. | **No** override. |
| **InsertMonitoringData** | Passes `data.status` to `db.insertMonitoringData`. ProcessGroupUpdate uses same status for aggregation input. | **No** override. |
| **db.insertMonitoringData** | Inserts/merges `status` as provided. Column is text. | **No** override. |
| **GetDataGroupByDayAlternative** | Builds day groups from raw rows. **Previously:** interpolation used `anchorStatus` (e.g. yesterday’s DOWN), so the day could be dominated by DOWN. **Now:** interpolation uses NO_DATA, and grouping uses defensive increment. | **No** override in code; previously **display** showed DOWN due to data shape, not a direct status override. |
| **page.js (FetchData)** | Derives `cssClass` from `dayData` (DEGRADED, MONITORTIMEOUT, DOWN, MAINTENANCE, NO_DATA). Priority order (last writer wins) is DOWN after MONITORTIMEOUT, so if day has both, DOWN wins by design. | **No** override of MONITORTIMEOUT to DOWN; only priority when both exist. |
| **calculateAggregatedStatus** | **Now:** DOWN > MONITORTIMEOUT > DEGRADED > … So MONITORTIMEOUT is not hidden by DEGRADED. | **No** override. |
| **GetLatestStatusActiveAll** | **Now:** DOWN > MONITORTIMEOUT > DEGRADED > UP. | **No** override. |

So with the current implementation (including interpolation and priority fixes), the heatmap will **not** show DOWN instead of MONITORTIMEOUT for a minute that was stored as MONITORTIMEOUT, unless that day also has DOWN and the chosen priority is DOWN (intended).

---

## 5. Will MONITORTIMEOUT Changes Affect How monitoring_data Is Persisted or Retrieved?

**Answer: Persistence and retrieval logic are unchanged. MONITORTIMEOUT is just another string value.**

- **Table:** `monitoring_data` (see `migrations/20250111153517_init.js`). Column `status` is `table.text("status")` (free text). No enum; any string is valid.
- **Write:** `db.insertMonitoringData({ monitor_tag, timestamp, status, latency, type })` (dbimpl.js). Whatever is passed as `status` (including `"MONITORTIMEOUT"`) is stored. No mapping or validation of status values.
- **Read:** `db.getDataGroupByDayAlternative`, `getMonitoringData`, `getLatestMonitoringData`, etc. return rows with `status` as stored. No mapping of MONITORTIMEOUT to DOWN or anything else in the DB layer.

So persistence and retrieval are **unchanged** in behavior; they simply support one more valid status value.

---

## B. Code Paths Where MONITORTIMEOUT Might Be Overridden, Ignored, or Mapped to DOWN

### 1. Already correct (no change needed)

- **apiCall.js:** Accepts MONITORTIMEOUT; sets `toWrite.status = evalResp.status`. No override.
- **cron-minute.js:** Passes `element.status` through. No override.
- **InsertMonitoringData / db.insertMonitoringData:** Persist `data.status` as-is. No override.
- **ProcessGroupUpdate / calculateAggregatedStatus:** MONITORTIMEOUT is in `statusCounts` and in the priority chain (DOWN > MONITORTIMEOUT > DEGRADED > …). No override.
- **GetDataGroupByDayAlternative:** Uses NO_DATA for interpolation and defensive grouping; MONITORTIMEOUT is in the initial group and in the return. No override.
- **page.js:** Uses `dayData.MONITORTIMEOUT` and `StatusObj.MONITORTIMEOUT` for cssClass. No override.
- **Heatmap component:** Uses `bar.cssClass`; navy class exists in kener.css. No override.

### 2. Optional fix (not in heatmap path today)

- **dbimpl.js – getLastStatusBeforeCombined** (lines 161–173): The SQL `CASE` returns only `'DOWN'`, `'DEGRADED'`, or `'UP'`. Any other status (including MONITORTIMEOUT) falls into `ELSE 'UP'`. This function is **not** used in the heatmap or status-page data path (grep shows no callers). If it is used elsewhere in the future, MONITORTIMEOUT would be reported as UP. For consistency, the CASE could be extended with a branch for MONITORTIMEOUT (e.g. between DOWN and DEGRADED to match priority). This is **optional** and only needed if/when this function is used for status display or aggregation.

**BEFORE (dbimpl.js, lines 167–173):**

```javascript
        this.knex.raw(`
				CASE 
				WHEN SUM(CASE WHEN status = 'DOWN' THEN 1 ELSE 0 END) > 0 THEN 'DOWN'
				WHEN SUM(CASE WHEN status = 'DEGRADED' THEN 1 ELSE 0 END) > 0 THEN 'DEGRADED'
				ELSE 'UP'
				END as status
			`),
```

**AFTER (recommended only if getLastStatusBeforeCombined is or will be used):**

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

**Justification:** Keeps DB aggregation consistent with priority (DOWN > MONITORTIMEOUT > DEGRADED > UP) if this function is ever used. No impact on heatmap or cron until then.

---

## C. Before/After Snippets for Already-Applied Fixes

The following fixes are **already implemented** in the codebase (documented here for the analysis). They do **not** change when the heatmap refreshes; they only fix how MONITORTIMEOUT is represented in the data used for the heatmap.

### 1. GetDataGroupByDayAlternative – Use NO_DATA for interpolation

**File:** `src/lib/server/controllers/controller.js`  
**Effect:** Interpolated minutes (no real data) no longer use the previous period’s status (e.g. DOWN). So days with only MONITORTIMEOUT no longer appear as DOWN.

**BEFORE:**

```javascript
  let rawData = await db.getDataGroupByDayAlternative(monitor_tag, start, end);
  let anchorStatus = await GetLastStatusBefore(monitor_tag, start);
  rawData = InterpolateData(rawData, start, anchorStatus, end);
```

**AFTER (current):**

```javascript
  let rawData = await db.getDataGroupByDayAlternative(monitor_tag, start, end);
  // Use NO_DATA for interpolated minutes so the day's status is not polluted by the previous period's status (e.g. DOWN)
  rawData = InterpolateData(rawData, start, NO_DATA, end);
```

### 2. GetDataGroupByDayAlternative – Defensive group increment

**Effect:** Avoids losing MONITORTIMEOUT (or any status) when `row.status` is missing or invalid (e.g. NaN from `group[row.status]++`).

**BEFORE:**

```javascript
    const group = acc[dayGroup];
    group.total++;
    group[row.status]++;

    return acc;
```

**AFTER (current):**

```javascript
    const group = acc[dayGroup];
    group.total++;
    const statusKey = row.status;
    if (statusKey != null && statusKey !== "") {
      group[statusKey] = (group[statusKey] ?? 0) + 1;
    } else {
      group.NO_DATA = (group.NO_DATA ?? 0) + 1;
    }

    return acc;
```

### 3. GetLatestStatusActiveAll – Priority MONITORTIMEOUT before DEGRADED

**Effect:** When multiple statuses are present, MONITORTIMEOUT is not hidden by DEGRADED.

**BEFORE:**

```javascript
    if (latestData[i].status === "DOWN") status = "DOWN";
    } else if (latestData[i].status === "DEGRADED" && status !== "DOWN") {
      status = "DEGRADED";
    } else if (latestData[i].status === "MONITORTIMEOUT" && ...
```

**AFTER (current):**

```javascript
    if (latestData[i].status === "DOWN") status = "DOWN";
    } else if (latestData[i].status === "MONITORTIMEOUT" && status !== "DOWN") {
      status = "MONITORTIMEOUT";
    } else if (latestData[i].status === "DEGRADED" && status !== "DOWN" && status !== "MONITORTIMEOUT") {
      status = "DEGRADED";
    } else if (latestData[i].status === "UP" && ...
```

### 4. calculateAggregatedStatus – Priority MONITORTIMEOUT before DEGRADED

**Effect:** Group/supergroup aggregation shows MONITORTIMEOUT when any child has MONITORTIMEOUT (and none DOWN).

**BEFORE:**

```javascript
  } else if (statusCounts.DEGRADED > 0) {
    aggregatedStatus = DEGRADED;
  } else if (statusCounts.MONITORTIMEOUT > 0) {
    aggregatedStatus = MONITORTIMEOUT;
```

**AFTER (current):**

```javascript
  } else if (statusCounts.MONITORTIMEOUT > 0) {
    aggregatedStatus = MONITORTIMEOUT;
  } else if (statusCounts.DEGRADED > 0) {
    aggregatedStatus = DEGRADED;
```

---

## D. Justification for Each Change

- **NO_DATA for interpolation:** Ensures the heatmap reflects only real data (and explicit NO_DATA) for each day. Prevents the previous period’s status (e.g. DOWN) from filling the day and causing DOWN to be shown instead of MONITORTIMEOUT. Does not change when data is read or when the page loads.
- **Defensive group increment:** Ensures MONITORTIMEOUT (and other statuses) are never lost to NaN or missing keys. Does not change refresh timing.
- **GetLatestStatusActiveAll / calculateAggregatedStatus priority:** Ensures MONITORTIMEOUT is visible when it is the correct “worst” status relative to DEGRADED/UP. Does not change when aggregation runs (still on each InsertMonitoringData and on each FetchData at page load).
- **getLastStatusBeforeCombined (optional):** Only for consistency if that DB helper is used; not required for heatmap or cron behavior.

---

## E. Summary README

### Root cause analysis

- The heatmap does **not** refresh in real time. It is built once per page load from `monitoring_data` via `FetchData` → `GetDataGroupByDayAlternative` and `page.js`. MONITORTIMEOUT does not add any new refresh mechanism.
- If the heatmap had shown DOWN instead of MONITORTIMEOUT, the cause was **data shape** (interpolation filling with previous status, grouping issues, or priority order), not a change to when or how often the grid updates.

### Impact on existing behavior

- **Cron schedule:** Unchanged. Minuter still runs at `monitor.cron`; MONITORTIMEOUT is just another status written.
- **“Show previous status until next poll”:** Unchanged. The grid still shows server-rendered `pageData._90Day` until the user loads/refreshes the page; new status (including MONITORTIMEOUT) appears only after the next poll has persisted and the user loads again.
- **Persistence:** Unchanged. `monitoring_data.status` remains free text; MONITORTIMEOUT is stored like any other status.
- **Retrieval:** Unchanged. Reads return `status` as stored; aggregation and day-building now correctly include MONITORTIMEOUT (interpolation and priority fixes).

### Recommended fixes

- The four fixes above are **already applied** in the codebase. They ensure MONITORTIMEOUT is persisted (unchanged), correctly aggregated, and correctly displayed on the next page load after the next cron run.
- **Optional:** Extend `getLastStatusBeforeCombined` with a MONITORTIMEOUT branch if that function is or will be used for status display or aggregation.

### Validation steps

1. Set a monitor’s eval to return MONITORTIMEOUT under some condition (e.g. timeout or specific response).
2. Run one poll (wait for cron or trigger manually). Confirm `monitoring_data` has a row with `status = 'MONITORTIMEOUT'` for that minute.
3. Load the status page. Confirm the heatmap shows navy for the corresponding day/cell for that monitor.
4. Confirm the heatmap does **not** update until you reload or navigate again (no real-time update).
5. For a group containing that monitor, confirm the group’s heatmap shows MONITORTIMEOUT (navy) when appropriate after the next load.

### No-breaking-changes confirmation

- No changes to cron schedule, to when Minuter runs, or to when the heatmap data is fetched (still only at page load).
- No new real-time or client-side refresh logic.
- MONITORTIMEOUT is additive: same persistence and read path as UP/DOWN/DEGRADED/MAINTENANCE, with correct interpolation, grouping, and priority so the heatmap shows it on the next refresh after the next poll.
