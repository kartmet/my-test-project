# MONITORTIMEOUT Heatmap Fix – Root Cause, Fixes, and Before/After

This document describes why MONITORTIMEOUT does not appear on the heatmap grid (leaf, group, supergroup), the exact code changes to fix it, and before/after snippets with line numbers.

---

## 1. Root Cause

### Why the heatmap does not show navy for MONITORTIMEOUT

- **Single cause:** On API timeout, the backend never persists `status = 'MONITORTIMEOUT'`. It only sets `type = 'timeout'` and leaves `status` as whatever the eval returned (often `DOWN`) or the default `DOWN`.
- **Table:** `monitoring_data` (column `status`, free text). Rows for timeout minutes are stored with `status: 'DOWN'` (or similar), so the heatmap pipeline never sees `MONITORTIMEOUT`.
- **Data flow:** Heatmap 90-day strip uses `GetDataGroupByDayAlternative(monitor_tag, ...)` → raw rows from `monitoring_data` → grouped by `row.status`. With no rows having `status === 'MONITORTIMEOUT'` for those minutes, `dayData.MONITORTIMEOUT` stays 0 and the UI never applies the navy class (`api-monitortimeout`).
- **Same for groups/supergroups:** They use the same `GetDataGroupByDayAlternative` and aggregation from `monitoring_data`; once leaf rows store MONITORTIMEOUT, aggregation and group heatmaps will show it.

### Database

- **Table:** `monitoring_data`  
- **Column:** `status` (text, no enum) – see `migrations/20250111153517_init.js` line 10.  
- MONITORTIMEOUT is not written on timeout because the API timeout path in `apiCall.js` never sets `toWrite.status = MONITORTIMEOUT`.

### Priority order (required)

1. DOWN  
2. MONITORTIMEOUT  
3. DEGRADED  
4. MAINTENANCE  
5. UP  

In two places MONITORTIMEOUT was evaluated after DEGRADED, so MONITORTIMEOUT did not override DEGRADED when aggregating or computing latest status. Those branches need to be reordered.

---

## 2. Suggested Fixes (summary)

| # | File | Change |
|---|------|--------|
| 1 | `src/lib/server/services/apiCall.js` | When `timeoutError` is true, set `toWrite.status = MONITORTIMEOUT` in addition to `toWrite.type = TIMEOUT`. |
| 2 | `src/lib/server/controllers/controller.js` | In `GetLatestStatusActiveAll`, check MONITORTIMEOUT before DEGRADED (priority: DOWN > MONITORTIMEOUT > DEGRADED > UP). |
| 3 | `src/lib/server/controllers/controller.js` | In `calculateAggregatedStatus`, check MONITORTIMEOUT before DEGRADED (priority: DOWN > MONITORTIMEOUT > DEGRADED > MAINTENANCE > UP > NO_DATA). |

---

## 3. Fix 1 – apiCall.js: Persist MONITORTIMEOUT on timeout

**File:** `src/lib/server/services/apiCall.js`

**Reason:** Timeout must be stored as MONITORTIMEOUT so the heatmap and aggregation see it. Currently only `type` is set to TIMEOUT; status stays DOWN (or eval result).

### BEFORE (lines 159–164)

```javascript
    if (timeoutError) {
      toWrite.type = TIMEOUT;
    }

    return toWrite;
```

### AFTER

```javascript
    if (timeoutError) {
      toWrite.status = MONITORTIMEOUT;
      toWrite.type = TIMEOUT;
    }

    return toWrite;
```

**Justification:** When a timeout occurs, we force `status` to MONITORTIMEOUT so the row written to `monitoring_data` is MONITORTIMEOUT. The eval result is overridden only for timeout; non-timeout behavior is unchanged.

---

## 4. Fix 2 – controller.js: GetLatestStatusActiveAll priority

**File:** `src/lib/server/controllers/controller.js`

**Reason:** Priority must be DOWN > MONITORTIMEOUT > DEGRADED > UP. Previously MONITORTIMEOUT was after DEGRADED, so a mix of DEGRADED and MONITORTIMEOUT yielded DEGRADED.

### BEFORE (lines 456–473)

```javascript
export const GetLatestStatusActiveAll = async (monitor_tags) => {
  let latestData = await db.getLatestMonitoringDataAllActive(monitor_tags);
  let status = "NO_DATA";
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
  return {
    status: status,
  };
};
```

### AFTER

```javascript
export const GetLatestStatusActiveAll = async (monitor_tags) => {
  let latestData = await db.getLatestMonitoringDataAllActive(monitor_tags);
  let status = "NO_DATA";
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
  return {
    status: status,
  };
};
```

**Justification:** Only the order and conditions of the checks change so MONITORTIMEOUT beats DEGRADED and UP; DOWN still wins. No new statuses or other callers affected.

---

## 5. Fix 3 – controller.js: calculateAggregatedStatus priority

**File:** `src/lib/server/controllers/controller.js`

**Reason:** Group/supergroup aggregation must use DOWN > MONITORTIMEOUT > DEGRADED > MAINTENANCE > UP > NO_DATA so that if any child is MONITORTIMEOUT (and none DOWN), the parent shows MONITORTIMEOUT.

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
    // ...
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
    // ... (unchanged)
  }
```

**Justification:** MONITORTIMEOUT is evaluated before DEGRADED so that one MONITORTIMEOUT child (with no DOWN) makes the parent MONITORTIMEOUT. All other branches and NO_DATA/UP logic stay the same.

---

## 6. Impact and validation

### Impact

- **Leaf monitors:** Timeout minutes are stored with `status = 'MONITORTIMEOUT'` → heatmap 90-day strip shows navy for those minutes.
- **Groups/supergroups:** Children with MONITORTIMEOUT are counted in `statusCounts.MONITORTIMEOUT`; aggregation and stored group/supergroup status show MONITORTIMEOUT when appropriate → their heatmaps show navy.
- **Existing behavior:** Non-timeout evals and DOWN/DEGRADED/UP/MAINTENANCE handling are unchanged. Only the timeout path and the relative order of MONITORTIMEOUT vs DEGRADED are updated.

### Validation steps

1. Trigger an API monitor timeout (e.g. short timeout, slow or unresponsive target).
2. Check `monitoring_data`: there should be a row with `status = 'MONITORTIMEOUT'` and `type = 'timeout'` for that minute.
3. Reload the status page: the leaf monitor’s 90-day strip should show navy for the timeout minute.
4. With a group that contains that leaf, confirm the group’s heatmap and summary show MONITORTIMEOUT (navy) when the leaf is in MONITORTIMEOUT.
5. Confirm DOWN still overrides MONITORTIMEOUT and DEGRADED in aggregation and latest-status.

### No-breaking-changes

- Only the timeout branch in `apiCall.js` and the order of status checks in two controller functions are changed.
- No new status values, no DB schema changes, no removal of existing status handling.
- UP/DOWN/DEGRADED/MAINTENANCE and heatmap logic for non-timeout data are unchanged.
