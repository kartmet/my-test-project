# MONITORTIMEOUT Propagation – Root Cause, Recommendations, and Fixes

This document analyzes why MONITORTIMEOUT may not appear on the heatmap grid (leaf, group, supergroup) and provides minimal code fixes. **MONITORTIMEOUT is a normal status returned by the user-defined eval**; the application must not hardcode timeout logic or override the eval’s decision.

---

## 1. Clarification: Eval and MONITORTIMEOUT

- The **eval function is user-defined**. It decides monitor status from the API payload/response.
- **MONITORTIMEOUT** is just another valid status (like UP, DOWN, DEGRADED). The app must:
  - Accept MONITORTIMEOUT when eval returns it.
  - Persist it in the same table/column as other statuses.
  - Propagate it to groups and supergroups.
  - Render it (e.g. navy) in the heatmap.
  - **Not** reinterpret it as DOWN, DEGRADED, or UNKNOWN.
  - **Not** use hardcoded HTTP status codes or timeout logic to force MONITORTIMEOUT.

The platform must support MONITORTIMEOUT end-to-end whenever eval returns it.

---

## 2. Root Cause Analysis

### A. Why the heatmap might not show navy for MONITORTIMEOUT

Possible causes:

1. **Grouping bug in `GetDataGroupByDayAlternative`**  
   The reducer uses `group[row.status]++`. In JavaScript, if `row.status` is missing from the initial group (e.g. typo or legacy data with different casing), or if `row.status` is `undefined`, then `group[row.status]` is `undefined` and `undefined++` becomes **NaN**. The return uses `group.MONITORTIMEOUT || 0`, so **NaN || 0 === 0**. So the day’s MONITORTIMEOUT count would be lost and the heatmap would not show navy for that day.

2. **Priority order**  
   In two places MONITORTIMEOUT is evaluated **after** DEGRADED. Required order is DOWN > MONITORTIMEOUT > DEGRADED > MAINTENANCE > UP. If both DEGRADED and MONITORTIMEOUT appear, the code currently picks DEGRADED; it should pick MONITORTIMEOUT.

3. **Persistence**  
   If eval returns MONITORTIMEOUT, the current code path accepts it and persists it (no override). So persistence is correct as long as eval actually returns `"MONITORTIMEOUT"` and the status column is used consistently.

### B. Database: where status is stored

| Item | Detail |
|------|--------|
| **Table** | `monitoring_data` |
| **Column** | `status` (`table.text("status")` in `migrations/20250111153517_init.js` line 10) |
| **Type** | Free text (no enum). Any string, including `"MONITORTIMEOUT"`, is valid. |
| **Write path** | `db.insertMonitoringData(data)` in `dbimpl.js` (lines 17–22). Inserts/merges `status` as provided. |
| **Read path** | `db.getDataGroupByDayAlternative(monitor_tag, start, end)` returns rows with `timestamp`, `status`, `latency`. |

So MONITORTIMEOUT is persisted correctly whenever the caller (e.g. cron-minute after eval) passes `status: "MONITORTIMEOUT"`. No schema change is required.

### C. Eval logic (no overrides)

- **apiCall.js (lines 127–153):** If eval returns `status: "MONITORTIMEOUT"`, it is in the allowed list `[UP, DOWN, DEGRADED, MONITORTIMEOUT]`, so it is **not** replaced with DOWN. `toWrite.status = evalResp.status` keeps MONITORTIMEOUT.
- There is **no** hardcoded timeout logic that forces status to MONITORTIMEOUT or DOWN based on HTTP status or `timeoutError`. Timeout behavior is left to the user-defined eval.
- Conclusion: **Eval → MONITORTIMEOUT handling is correct.** The app treats MONITORTIMEOUT like any other status and does not override it.

### D. Where the app can fail to support MONITORTIMEOUT

1. **Controller – day grouping**  
   `group[row.status]++` can produce NaN for unknown or undefined `row.status`, so `group.MONITORTIMEOUT || 0` can be 0 even when there are MONITORTIMEOUT rows. Fix: use a defensive increment so counts are never NaN and MONITORTIMEOUT is always counted when present.

2. **Controller – priority**  
   In `GetLatestStatusActiveAll` and `calculateAggregatedStatus`, MONITORTIMEOUT is currently after DEGRADED. So when both exist, the result is DEGRADED instead of MONITORTIMEOUT. Fix: evaluate MONITORTIMEOUT **before** DEGRADED so MONITORTIMEOUT takes precedence.

---

## 3. Status Flow Summary

| Layer | MONITORTIMEOUT support |
|-------|-------------------------|
| Eval (user-defined) | Returns `status: "MONITORTIMEOUT"` when user logic says so. |
| apiCall.js | Accepts MONITORTIMEOUT; does not override. |
| Persistence (monitoring_data) | `status` column (text); stores whatever is passed. |
| GetDataGroupByDayAlternative | Groups by `row.status`; **risk:** `group[row.status]++` → NaN if key missing/undefined. |
| calculateAggregatedStatus | Has MONITORTIMEOUT in statusCounts; **issue:** priority DEGRADED before MONITORTIMEOUT. |
| GetLatestStatusActiveAll | **Issue:** priority DEGRADED before MONITORTIMEOUT. |
| page.js (90-day heatmap) | Uses `dayData.MONITORTIMEOUT` and `StatusObj.MONITORTIMEOUT`; correct when dayData has count. |
| Frontend (monitor.svelte) | Uses `bar.cssClass` from pageData._90Day; navy class exists. |

So the only required fixes are in the controller: safe grouping and priority order.

---

## 4. Recommended Fixes (minimal)

| # | File | Change |
|---|------|--------|
| 1 | `src/lib/server/controllers/controller.js` | In `GetDataGroupByDayAlternative`, replace `group[row.status]++` with a defensive increment so MONITORTIMEOUT (and any other known status) is never lost to NaN. |
| 2 | `src/lib/server/controllers/controller.js` | In `GetLatestStatusActiveAll`, check MONITORTIMEOUT **before** DEGRADED (priority: DOWN > MONITORTIMEOUT > DEGRADED > UP). |
| 3 | `src/lib/server/controllers/controller.js` | In `calculateAggregatedStatus`, check MONITORTIMEOUT **before** DEGRADED (priority: DOWN > MONITORTIMEOUT > DEGRADED > MAINTENANCE > UP > NO_DATA). |

**No change** to apiCall.js timeout logic: eval alone determines MONITORTIMEOUT; the app must not force it.

---

## 5. Fix 1 – GetDataGroupByDayAlternative: defensive group increment

**File:** `src/lib/server/controllers/controller.js`  
**Location:** Reducer inside `GetDataGroupByDayAlternative` (around lines 1032–1036).

**Reason:** `group[row.status]++` is undefined when `row.status` is not a key of `group`, and `undefined++` is NaN. Then `group.MONITORTIMEOUT || 0` is 0. Using a defensive increment ensures MONITORTIMEOUT (and other statuses) are counted correctly even with missing or undefined status, and avoids NaN.

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

**Justification:**  
- `(group[statusKey] ?? 0) + 1` never produces NaN and counts MONITORTIMEOUT (and any other status) even if that key were ever missing from the initial object.  
- Unknown or empty status is treated as NO_DATA so it does not create NaN and the rest of the pipeline stays consistent.  
- Only the grouping logic is changed; initial group shape and return shape are unchanged.

---

## 6. Fix 2 – GetLatestStatusActiveAll: priority DOWN > MONITORTIMEOUT > DEGRADED > UP

**File:** `src/lib/server/controllers/controller.js`  
**Location:** Loop in `GetLatestStatusActiveAll` (around lines 456–473).

**Reason:** MONITORTIMEOUT must override DEGRADED and UP. Currently MONITORTIMEOUT is checked after DEGRADED, so a mix of DEGRADED and MONITORTIMEOUT yields DEGRADED.

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

**Justification:** Only the order and conditions of the checks change so that MONITORTIMEOUT beats DEGRADED and UP; DOWN still wins. No new statuses; other callers unchanged.

---

## 7. Fix 3 – calculateAggregatedStatus: priority DOWN > MONITORTIMEOUT > DEGRADED

**File:** `src/lib/server/controllers/controller.js`  
**Location:** Priority chain in `calculateAggregatedStatus` (around lines 579–598).

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

**Justification:** MONITORTIMEOUT is evaluated before DEGRADED so one MONITORTIMEOUT child (with no DOWN) makes the parent MONITORTIMEOUT. All other branches and NO_DATA/UP logic unchanged.

---

## 8. What is not changed

- **apiCall.js**  
  No hardcoded timeout handling. Eval alone decides status (including MONITORTIMEOUT). The app already accepts and passes through MONITORTIMEOUT.

- **Database**  
  No schema or migration changes. `monitoring_data.status` remains free text.

- **Frontend**  
  No change. MONITORTIMEOUT is already mapped to navy; heatmap uses `bar.cssClass` from server-built `_90Day`.

- **Eval contract**  
  No new status values or special cases. MONITORTIMEOUT is supported the same way as UP, DOWN, DEGRADED.

---

## 9. Impact and Validation

### Impact

- **Leaf heatmap:** Day-level MONITORTIMEOUT counts are no longer lost to NaN, so the 90-day strip can show navy when eval has returned MONITORTIMEOUT.
- **Group/supergroup:** Aggregation and latest-status use DOWN > MONITORTIMEOUT > DEGRADED, so MONITORTIMEOUT appears correctly in summaries and heatmaps when any child is MONITORTIMEOUT (and none DOWN).
- **Existing behavior:** UP, DOWN, DEGRADED, MAINTENANCE, NO_DATA logic and persistence are unchanged. Only grouping safety and priority order are updated.

### Validation steps

1. Use an eval that returns `status: "MONITORTIMEOUT"` for some responses (e.g. based on response time or payload).
2. Confirm `monitoring_data` has rows with `status = 'MONITORTIMEOUT'` for that monitor.
3. Reload the status page: the leaf’s 90-day heatmap should show navy for days that have MONITORTIMEOUT minutes.
4. Put that monitor in a group; confirm the group’s heatmap and summary show MONITORTIMEOUT (navy) when the leaf is MONITORTIMEOUT.
5. Confirm DOWN still overrides MONITORTIMEOUT and DEGRADED in aggregation and latest-status.

### No-breaking-changes

- Only the controller is changed: one reducer (defensive increment) and two priority orders (GetLatestStatusActiveAll and calculateAggregatedStatus).
- No new statuses, no DB changes, no eval or timeout overrides. Existing status handling and heatmap behavior for other statuses remain the same.
