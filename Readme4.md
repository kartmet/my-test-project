# MONITORTIMEOUT Showing as DOWN on Heatmap – Root Cause and Fixes

This document explains why the heatmap grid shows **DOWN** instead of **MONITORTIMEOUT** for leaf monitors, groups, and supergroups, and gives minimal code changes to fix it. **MONITORTIMEOUT is a normal status from the user-defined eval**; the application must not override it or map it to DOWN.

---

## 1. Root Cause: Why the Grid Shows DOWN Instead of MONITORTIMEOUT

### Primary cause: interpolation fills the day with the previous period’s status (e.g. DOWN)

The heatmap’s 90‑day data is built in **GetDataGroupByDayAlternative**:

1. **Raw data** from `db.getDataGroupByDayAlternative(monitor_tag, start, end)` – only rows that exist in `monitoring_data` (e.g. 29 minutes of MONITORTIMEOUT at the end of today).
2. **InterpolateData(rawData, start, anchorStatus, end)** – fills **every minute** from `start` to `end`.  
   `anchorStatus` is **GetLastStatusBefore(monitor_tag, start)** = last status **before** the range start (e.g. last status from **yesterday**).
3. So from **midnight today** up to the first real data point, **every minute is filled with that previous status**. If yesterday ended in **DOWN**, that’s hundreds of minutes of **DOWN**.
4. The reducer then does **group[row.status]++** for every row (real + interpolated). So **group.DOWN** is large and **group.MONITORTIMEOUT** is 29.
5. In **page.js**, the day’s worst status is chosen by a sequence of `if` checks; **DOWN** is applied **after** MONITORTIMEOUT, so **DOWN overwrites** and the cell shows DOWN (red).

So the application is **not** overwriting the eval result in the DB; it is **reusing the previous period’s status for missing minutes**, which pollutes the day’s counts and makes the heatmap show DOWN instead of MONITORTIMEOUT.

**Fix:** When building day-grouped data for the heatmap, treat “no data” as **NO_DATA**, not as the previous period’s status. Use **NO_DATA** as the initial status in **InterpolateData** for this path so only **real** data (e.g. MONITORTIMEOUT) and explicit NO_DATA drive the day’s status.

### Secondary: grouping can drop MONITORTIMEOUT (NaN)

**group[row.status]++** is used in the same reducer. If `row.status` is missing, wrong case, or undefined, **group[row.status]** is undefined and **undefined++** is **NaN**. The return uses **group.MONITORTIMEOUT || 0**, so **NaN || 0 === 0** and MONITORTIMEOUT can disappear for that day. A defensive increment avoids that.

### Priority order

In **GetLatestStatusActiveAll** and **calculateAggregatedStatus**, MONITORTIMEOUT is currently evaluated **after** DEGRADED. Required order is **DOWN > MONITORTIMEOUT > DEGRADED > …** so MONITORTIMEOUT is not hidden by DEGRADED when both appear.

---

## 2. Database and Eval (No Override)

| Item | Detail |
|------|--------|
| **Table** | `monitoring_data` |
| **Column** | `status` (text, no enum) |
| **Persistence** | Whatever is passed to `insertMonitoringData` is stored. Eval returns MONITORTIMEOUT → we persist MONITORTIMEOUT. |
| **Eval** | apiCall.js accepts MONITORTIMEOUT in the allowed list and sets `toWrite.status = evalResp.status`. No mapping of MONITORTIMEOUT to DOWN. |

So MONITORTIMEOUT is stored correctly when eval returns it. The “DOWN” on the heatmap comes from **interpolation and day aggregation**, not from overwriting the stored status.

---

## 3. Recommended Fixes (Summary)

| # | File | Change |
|---|------|--------|
| 1 | **controller.js** | In **GetDataGroupByDayAlternative**, call **InterpolateData** with initial status **NO_DATA** instead of **anchorStatus**, so missing minutes are NO_DATA and do not inject DOWN (or any other previous status) into the day. |
| 2 | **controller.js** | In the same reducer, use a **defensive increment** for **group[row.status]** so MONITORTIMEOUT (and other statuses) are never lost to NaN. |
| 3 | **controller.js** | In **GetLatestStatusActiveAll**, check **MONITORTIMEOUT** before **DEGRADED** (priority: DOWN > MONITORTIMEOUT > DEGRADED > UP). |
| 4 | **controller.js** | In **calculateAggregatedStatus**, check **MONITORTIMEOUT** before **DEGRADED** (priority: DOWN > MONITORTIMEOUT > DEGRADED > MAINTENANCE > UP > NO_DATA). |

---

## 4. Fix 1 – Use NO_DATA for interpolated minutes (main fix)

**File:** `src/lib/server/controllers/controller.js`  
**Location:** GetDataGroupByDayAlternative, around lines 1012–1015.

**Reason:** InterpolateData currently fills every minute from start to end with **anchorStatus** (last status before the range). That can be DOWN and dominates the day’s counts, so the heatmap shows DOWN instead of MONITORTIMEOUT. Using **NO_DATA** for the initial status makes “no data” minutes count as NO_DATA only, so the day’s color is driven by real data (e.g. MONITORTIMEOUT).

### BEFORE (lines 1012–1015)

```javascript
  let rawData = await db.getDataGroupByDayAlternative(monitor_tag, start, end);
  let anchorStatus = await GetLastStatusBefore(monitor_tag, start);
  rawData = InterpolateData(rawData, start, anchorStatus, end);
```

### AFTER

```javascript
  let rawData = await db.getDataGroupByDayAlternative(monitor_tag, start, end);
  // Use NO_DATA for interpolated minutes so the day's status is not polluted by the previous period's status (e.g. DOWN)
  rawData = InterpolateData(rawData, start, NO_DATA, end);
```

**Justification:** Only the argument passed to InterpolateData changes. InterpolateData still fills one row per minute; those without real data now get status **NO_DATA** instead of the previous period’s status. Days with only MONITORTIMEOUT (and NO_DATA for the rest) will then show MONITORTIMEOUT on the heatmap. Existing behavior for days that have real DOWN/DEGRADED/UP is unchanged.

---

## 5. Fix 2 – Defensive group increment

**File:** `src/lib/server/controllers/controller.js`  
**Location:** Reducer inside GetDataGroupByDayAlternative, around lines 1032–1036.

**Reason:** **group[row.status]++** can produce NaN when `row.status` is undefined or not a key of the initial group, so **group.MONITORTIMEOUT || 0** becomes 0 and MONITORTIMEOUT is lost. A defensive increment and handling of unknown/empty status avoid NaN and keep MONITORTIMEOUT (and others) correct.

### BEFORE (lines 1032–1036)

```javascript
    const group = acc[dayGroup];
    group.total++;
    group[row.status]++;

    return acc;
```

### AFTER

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

**Justification:** Ensures no NaN, counts MONITORTIMEOUT and other known statuses even when the key was missing from the initial object, and maps null/empty status to NO_DATA so the rest of the pipeline stays consistent.

---

## 6. Fix 3 – GetLatestStatusActiveAll priority

**File:** `src/lib/server/controllers/controller.js`  
**Location:** GetLatestStatusActiveAll loop, around lines 458–470.

**Reason:** MONITORTIMEOUT must override DEGRADED and UP. Currently MONITORTIMEOUT is checked after DEGRADED.

### BEFORE (lines 458–470)

```javascript
  for (let i = 0; i < latestData.length; i++) {
    // Priority: DOWN > DEGRADED > MONITORTIMEOUT > UP
    if (latestData[i].status === "DOWN") {
      status = "DOWN";
    } else if (latestData[i].status === "DEGRADED" && status !== "DOWN") {
      status = "DEGRADED";
    } else if (latestData[i].status === "MONITORTIMEOUT" && status !== "DOWN" && status !== "DEGRADED") {
      status = "MONITORTIMEOUT";
    } else if (latestData[i].status === "UP" && status !== "DOWN" && status !== "DEGRADED" && status !== "MONITORTIMEOUT") {
      status = "UP";
    }
  }
```

### AFTER

```javascript
  for (let i = 0; i < latestData.length; i++) {
    // Priority: DOWN > MONITORTIMEOUT > DEGRADED > UP
    if (latestData[i].status === "DOWN") {
      status = "DOWN";
    } else if (latestData[i].status === "MONITORTIMEOUT" && status !== "DOWN") {
      status = "MONITORTIMEOUT";
    } else if (latestData[i].status === "DEGRADED" && status !== "DOWN" && status !== "MONITORTIMEOUT") {
      status = "DEGRADED";
    } else if (latestData[i].status === "UP" && status !== "DOWN" && status !== "MONITORTIMEOUT" && status !== "DEGRADED") {
      status = "UP";
    }
  }
```

**Justification:** Only the order and conditions of the checks change so that MONITORTIMEOUT beats DEGRADED and UP; DOWN still wins.

---

## 7. Fix 4 – calculateAggregatedStatus priority

**File:** `src/lib/server/controllers/controller.js`  
**Location:** calculateAggregatedStatus priority chain, around lines 579–598.

**Reason:** For group/supergroup aggregation, when any child has MONITORTIMEOUT (and none DOWN), the parent should be MONITORTIMEOUT. Currently DEGRADED is checked before MONITORTIMEOUT.

### BEFORE (lines 579–598)

```javascript
  // Priority: DOWN > DEGRADED > MONITORTIMEOUT > UP > MAINTENANCE > NO_DATA
  if (statusCounts.DOWN > 0) {
    aggregatedStatus = DOWN;
  } else if (statusCounts.DEGRADED > 0) {
    aggregatedStatus = DEGRADED;
  } else if (statusCounts.MONITORTIMEOUT > 0) {
    aggregatedStatus = MONITORTIMEOUT;
  } else if (statusCounts.MAINTENANCE > 0) {
    aggregatedStatus = MAINTENANCE;
  } else if (statusCounts.UP > 0 && statusCounts.NO_DATA === 0) {
    aggregatedStatus = UP;
  } else if (statusCounts.UP > 0) {
    // ... unchanged
  }
```

### AFTER

```javascript
  // Priority: DOWN > MONITORTIMEOUT > DEGRADED > MAINTENANCE > UP > NO_DATA
  if (statusCounts.DOWN > 0) {
    aggregatedStatus = DOWN;
  } else if (statusCounts.MONITORTIMEOUT > 0) {
    aggregatedStatus = MONITORTIMEOUT;
  } else if (statusCounts.DEGRADED > 0) {
    aggregatedStatus = DEGRADED;
  } else if (statusCounts.MAINTENANCE > 0) {
    aggregatedStatus = MAINTENANCE;
  } else if (statusCounts.UP > 0 && statusCounts.NO_DATA === 0) {
    aggregatedStatus = UP;
  } else if (statusCounts.UP > 0) {
    // ... unchanged
  }
```

**Justification:** MONITORTIMEOUT is evaluated before DEGRADED so one MONITORTIMEOUT child (with no DOWN) makes the parent MONITORTIMEOUT. Other branches unchanged.

---

## 8. Optional: DB layer if used later

**File:** `src/lib/server/db/dbimpl.js`  
**getLastStatusBeforeCombined** (lines 167–173) uses a **CASE** that returns only **'DOWN'**, **'DEGRADED'**, or **'UP'**; everything else (including MONITORTIMEOUT) falls into **ELSE 'UP'**. This function is not used in the heatmap path today, but if it is used for status display or aggregation, MONITORTIMEOUT would be shown as UP. For consistency, the CASE could be extended with:

- **WHEN SUM(CASE WHEN status = 'MONITORTIMEOUT' THEN 1 ELSE 0 END) > 0 THEN 'MONITORTIMEOUT'**  
  and placed between DOWN and DEGRADED to match priority. This is optional until that code path is used.

---

## 9. What is not changed

- **apiCall.js:** No change. Eval’s MONITORTIMEOUT is already accepted and persisted; no timeout overrides.
- **Database schema:** No change.
- **Frontend:** No change. MONITORTIMEOUT is already mapped to navy; heatmap uses server-built `_90Day`.
- **InterpolateData** is still used for the “today” / daily-detail path with the existing anchor (e.g. in **page.js** for todayDataDb). Only the **GetDataGroupByDayAlternative** call uses NO_DATA as the initial status.

---

## 10. Impact and Validation

### Impact

- **Leaf heatmap:** Days with only MONITORTIMEOUT (and NO_DATA for the rest) show navy; days are no longer filled with the previous period’s DOWN.
- **Group/supergroup:** Same data source and aggregation; MONITORTIMEOUT propagates with the correct priority (DOWN > MONITORTIMEOUT > DEGRADED).
- **Existing behavior:** Real DOWN/DEGRADED/UP/MAINTENANCE minutes still drive the heatmap; only **interpolated** minutes are now NO_DATA in this path.

### Validation

1. Ensure eval returns MONITORTIMEOUT for the relevant minutes; confirm `monitoring_data` has `status = 'MONITORTIMEOUT'` for those rows.
2. Reload the status page: leaf heatmap should show navy for days where the only (or worst) real status is MONITORTIMEOUT.
3. Check a group/supergroup containing that leaf: their heatmaps should show MONITORTIMEOUT when the leaf is MONITORTIMEOUT and no child is DOWN.
4. Confirm days that have real DOWN still show DOWN (red).

### No-breaking-changes

- Only **GetDataGroupByDayAlternative** (interpolation argument + reducer) and the two priority chains are changed. No new statuses, no schema changes, no eval or timeout overrides. Other callers of InterpolateData and existing status handling are unchanged.
