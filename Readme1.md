# Kener - Status Page & Monitoring Platform

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Feature Breakdown](#feature-breakdown)
4. [Deployment Strategy](#deployment-strategy)
5. [File-Level Deployment Instructions](#file-level-deployment-instructions)
6. [Database Considerations](#database-considerations)
7. [Testing & Validation](#testing--validation)
8. [Performance Considerations](#performance-considerations)
9. [Troubleshooting](#troubleshooting)
10. [Quick Reference](#quick-reference)
11. [Code Locations Reference](#code-locations-reference)

---

## System Overview

**Kener** is a modern, open-source status page application built with Node.js, SvelteKit, and PostgreSQL/MySQL/SQLite. It provides real-time monitoring, uptime tracking, incident management, and hierarchical monitor grouping.

### Core Components

- **Frontend**: SvelteKit with Tailwind CSS
- **Backend**: Node.js with Express
- **Database**: PostgreSQL (CloudSQL), MySQL, or SQLite
- **Monitoring**: Real-time status aggregation and propagation

### Monitor Types

1. **Leaf Monitors**: API, PING, TCP, DNS, SSL, SQL, HEARTBEAT, GAMEDIG
2. **GROUP Monitors**: Aggregate status from leaf monitors and child GROUPs
3. **SUPERGROUP Monitors**: Aggregate status from GROUP and SUPERGROUP monitors (multi-layer hierarchy)

---

## Architecture

### Monitor Hierarchy

```
SUPERGROUP (Business Service)
  └── SUPERGROUP (Region/Zone)
      └── SUPERGROUP (Foundation/Platform)
          └── GROUP (Application Group)
              └── Leaf Monitors (API, PING, etc.)
```

### Status Propagation

- **Priority**: DOWN > DEGRADED > UP
- **Recursive**: Status changes propagate up the hierarchy
- **Cycle Protection**: Prevents infinite loops (max depth: 10)
- **Real-time**: Status updates trigger immediate parent recalculation

### Data Flow

1. Leaf monitor reports status → `InsertMonitoringData()`
2. Status queued → `ProcessGroupUpdate()` triggered
3. Parent monitors recalculate → `calculateAggregatedStatus()`
4. Status propagated recursively → `ProcessGroupUpdate()` (recursive)
5. Database updated → `monitoring_data` table

---

## Feature Breakdown

### ✅ Feature 1: SUPERGROUP Multi-Layer Support

**Status**: ✅ **DEPLOYED IN PRODUCTION**

**Description**: Enables hierarchical monitor grouping with unlimited nesting depth (safety limit: 10 levels).

**Key Capabilities**:
- SUPERGROUP monitors can contain GROUP and SUPERGROUP children
- Recursive status propagation with cycle protection
- UI validation prevents self-selection and invalid hierarchies
- Backward compatible with existing GROUP monitors

**Files Modified**:
- `src/lib/components/manage/monitorsAdd.svelte` (Lines 197-211)
- `src/lib/components/manage/monitorSheet.svelte` (Lines 330-343, 352-401)
- `src/lib/server/controllers/controller.js` (Lines 794-892, 921-968)
- `src/lib/server/services/superGroupCall.js` (Service implementation)

**Database**: No schema changes required (uses existing `monitor_type` and `type_data` fields)

**Dependencies**: None (standalone feature)

---

### ✅ Feature 2: Custom Tooltip Message

**Status**: ✅ **DEPLOYED IN PRODUCTION**

**Description**: User-defined tooltip messages displayed on heatmap hover (optional, max 2000 chars).

**Key Capabilities**:
- Overrides default tooltip (date + status)
- Character limit: 2000 (enforced in UI and backend)
- Works for all monitor types
- Backward compatible (empty = default behavior)

**Files Modified**:
- `migrations/20260115100000_add_custom_tooltip_and_link_fields.js` (NEW)
- `src/lib/server/controllers/controller.js` (Lines 203-248, uses `monitorValidators.js`)
- `src/lib/server/controllers/monitorValidators.js` (NEW - shared validation)
- `src/lib/server/db/dbimpl.js` (Lines 439, 488)
- `src/lib/components/manage/monitorSheet.svelte` (Lines 143-152, 770-806)
- `src/lib/components/manage/monitorsAdd.svelte` (Line 95)
- `src/lib/components/monitor.svelte` (Lines 407-413)

**Database**: 
- Migration: `20260115100000_add_custom_tooltip_and_link_fields.js`
- Field: `customtooltip_message TEXT NULL` (consistent naming across all files)

**Dependencies**: None (standalone feature)

---

### ✅ Feature 3: Custom Monitor Link

**Status**: ✅ **DEPLOYED IN PRODUCTION**

**Description**: Custom click-through URLs with timeline parameters (optional, http/https only).

**Key Capabilities**:
- Overrides default daily data modal
- URL parameters: `start_time`, `end_time`, `clicked_time`
- Smart time handling (rounds down by 1 minute, prevents future time errors)
- Opens in new tab with security flags (`noopener,noreferrer`)
- URL validation (http/https only, prevents XSS)

**Files Modified**:
- `migrations/20260115100000_add_custom_tooltip_and_link_fields.js` (NEW)
- `src/lib/server/controllers/controller.js` (Lines 203-248, uses `monitorValidators.js`)
- `src/lib/server/controllers/monitorValidators.js` (NEW - shared validation)
- `src/lib/server/db/dbimpl.js` (Lines 440, 489)
- `src/lib/components/manage/monitorSheet.svelte` (Lines 154-166, 770-806)
- `src/lib/components/manage/monitorsAdd.svelte` (Line 96)
- `src/lib/components/monitor.svelte` (Lines 192-228)

**Database**: 
- Migration: `20260115100000_add_custom_tooltip_and_link_fields.js`
- Field: `custommonitor_link TEXT NULL` (consistent naming across all files)

**Dependencies**: None (standalone feature)

---

### 🚧 Feature 4: SLI (Service Level Indicator)

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Service Level Indicator - calculates availability percentage with DEGRADED weighting.

**Key Capabilities**:
- Calculates SLI: `(UP_count + degraded_weight * DEGRADED_count) / total_count * 100`
- Configurable `degraded_weight` (0.1-0.9, default 0.5)
- Stores calculated `sli_value` in database
- Historical tracking via `sli_history` table
- Supports both leaf monitors and aggregated monitors (GROUP/SUPERGROUP)

**EPIC: SLI (Service Level Indicator) Calculation System**

**Stories**:

1. **Story 4.1: Database Schema for SLI Fields**
   - **Description**: Add database fields for SLI calculation and storage
   - **Acceptance Criteria**:
     - `degraded_weight` field added (DECIMAL(3,2), default 0.5)
     - `sli_value` field added (DECIMAL(5,4), nullable)
     - `evaluation_period_start` and `evaluation_period_end` fields added (INTEGER, nullable)
     - Migration includes rollback function
   - **Files**: `migrations/20250115120000_add_sla_slo_sli_fields.js` (NEW)
   - **Before**: No SLI-related fields in `monitors` table
   - **After**: Lines 28-49 - Added `degraded_weight`, `sli_value`, `evaluation_period_start`, `evaluation_period_end` fields
   - **Justification**: Stores SLI configuration and calculated values for quick access

2. **Story 4.2: SLI History Table**
   - **Description**: Create table for historical SLI tracking
   - **Acceptance Criteria**:
     - Table `sli_history` created with fields: monitor_tag, evaluation_period_start/end, sli_value, slo_target, sla_violated, status counts
     - Indexes on monitor_tag, evaluation_period_start, sla_violated
     - Foreign key constraint on monitor_tag
     - Migration includes rollback function
   - **Files**: `migrations/20250115120001_create_sli_history_table.js` (NEW)
   - **Before**: No historical SLI tracking
   - **After**: Lines 25-53 - Created `sli_history` table with all required fields and indexes
   - **Justification**: Enables trend analysis and historical SLA compliance reporting

3. **Story 4.3: SLI Calculation Utility Functions**
   - **Description**: Create utility functions for SLI calculation
   - **Acceptance Criteria**:
     - `calculateSLI()` function calculates SLI from status counts
     - `calculateSLIFromData()` function calculates SLI from monitoring data array
     - Validates degraded_weight range (0.1-0.9)
     - Handles edge cases (no data, division by zero)
     - Returns SLI as percentage (0-100)
   - **Files**: `src/lib/server/slaUtils.js` (NEW)
   - **Before**: No SLI calculation functions
   - **After**: 
     - Lines 43-91 - `calculateSLI()` function with validation and error handling
     - Lines 211-266 - `calculateSLIFromData()` function for historical data
   - **Justification**: Centralizes SLI calculation logic for reuse across the platform

4. **Story 4.4: SLI Calculation Integration for GROUP/SUPERGROUP Monitors**
   - **Description**: Integrate SLI calculation into status aggregation workflow
   - **Acceptance Criteria**:
     - SLI calculated when GROUP/SUPERGROUP status changes
     - Uses status counts from aggregated children
     - Updates `sli_value` in database
     - Logs SLI calculation results
   - **Files**: `src/lib/server/controllers/controller.js`
   - **Before**: No SLI calculation in `ProcessGroupUpdate()`
   - **After**: 
     - Line 22 - Import `calculateSLI`, `calculateSLIFromData`, `validateSLOTarget`, `evaluateSLA` from `slaUtils.js`
     - Lines 835-839 - Call `updateSLIAndSLA()` after status aggregation
     - Lines 2259-2326 - `updateSLIAndSLA()` function calculates SLI and updates database
   - **Justification**: Automatically calculates SLI when aggregated status changes

5. **Story 4.5: SLI Calculation Integration for Leaf Monitors**
   - **Description**: Integrate SLI calculation for leaf monitors from monitoring data
   - **Acceptance Criteria**:
     - SLI calculated when leaf monitor data is inserted
     - Uses last 24 hours of monitoring data
     - Updates `sli_value` in database
     - Handles monitors with insufficient data gracefully
   - **Files**: `src/lib/server/controllers/controller.js`
   - **Before**: No SLI calculation for leaf monitors
   - **After**:
     - Lines 954-964 - Call `updateSLIForLeafMonitor()` after data insertion for leaf monitors
     - Lines 2344-2421 - `updateSLIForLeafMonitor()` function calculates SLI from monitoring data
   - **Justification**: Provides SLI for leaf monitors based on their status history

6. **Story 4.6: Database Layer Support for SLI Fields**
   - **Description**: Update database layer to handle SLI fields in insert/update operations
   - **Acceptance Criteria**:
     - `insertMonitor()` accepts `degraded_weight`, `sli_value`, `evaluation_period_start/end`
     - `updateMonitor()` accepts SLI fields for updates
     - Fields are nullable (backward compatible)
   - **Files**: `src/lib/server/db/dbimpl.js`
   - **Before**: No SLI fields in insertMonitor/updateMonitor
   - **After**: 
     - Lines 420-425 (approximate) - Added SLI fields to `insertMonitor()` data mapping
     - Lines 470-475 (approximate) - Added SLI fields to `updateMonitor()` conditional updates
   - **Justification**: Enables persistence of SLI configuration and calculated values

**Database**:
- Fields: `degraded_weight DECIMAL(3,2) DEFAULT 0.5`, `sli_value DECIMAL(5,4) NULL`, `evaluation_period_start INTEGER NULL`, `evaluation_period_end INTEGER NULL`
- Table: `sli_history` (historical SLI values for trend analysis)

**Dependencies**: 
- Requires Feature 1 (SUPERGROUP) for aggregated SLI calculation
- Can be deployed independently after SUPERGROUP

**Deployment Order**: Phase 2 (after Phase 1)

---

### 🚧 Feature 5: SLO (Service Level Objective)

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Service Level Objective - defines target availability threshold (90-100%).

**Key Capabilities**:
- Configurable SLO target: 90-100% (validated)
- Stores `slo_target` in database
- Validation prevents invalid targets (< 90% or > 100%)
- Primarily for GROUP and SUPERGROUP monitors
- Can be set for leaf monitors (optional)

**EPIC: SLO (Service Level Objective) Configuration System**

**Stories**:

1. **Story 5.1: Database Schema for SLO Target**
   - **Description**: Add `slo_target` field to monitors table
   - **Acceptance Criteria**:
     - `slo_target` field added (DECIMAL(5,4), nullable)
     - Field allows values 90-100%
     - Migration includes rollback function
   - **Files**: `migrations/20250115120000_add_sla_slo_sli_fields.js`
   - **Before**: No SLO target field in `monitors` table
   - **After**: Line 33 - Added `slo_target DECIMAL(5,4) NULL` field
   - **Justification**: Stores SLO target for SLA compliance evaluation

2. **Story 5.2: SLO Target Validation Utility**
   - **Description**: Create validation function for SLO targets
   - **Acceptance Criteria**:
     - `validateSLOTarget()` function validates target is 90-100%
     - Returns validation result with error message
     - Handles null/undefined values
     - Validates numeric type
   - **Files**: `src/lib/server/slaUtils.js`
   - **Before**: No SLO validation function
   - **After**: Lines 104-134 - `validateSLOTarget()` function with range validation (90-100%)
   - **Justification**: Ensures SLO targets are within valid range before storage

3. **Story 5.3: SLO Target Validation in Controller**
   - **Description**: Validate SLO target when creating/updating monitors
   - **Acceptance Criteria**:
     - Validates SLO target in `CreateMonitor()` and `UpdateMonitor()`
     - Returns error if target < 90% or > 100%
     - Allows null/undefined (optional field)
   - **Files**: `src/lib/server/controllers/controller.js`
   - **Before**: No SLO validation in monitor creation/update
   - **After**: 
     - Line 22 - Import `validateSLOTarget` from `slaUtils.js`
     - Lines 2475-2491 (approximate) - SLO target validation in `UpdateMonitor()` when `slo_target` is provided
   - **Justification**: Prevents invalid SLO targets from being stored

4. **Story 5.4: Database Layer Support for SLO Target**
   - **Description**: Update database layer to handle `slo_target` field
   - **Acceptance Criteria**:
     - `insertMonitor()` accepts `slo_target`
     - `updateMonitor()` accepts `slo_target` for updates
     - Field is nullable (backward compatible)
   - **Files**: `src/lib/server/db/dbimpl.js`
   - **Before**: No `slo_target` field in insertMonitor/updateMonitor
   - **After**: 
     - Lines 420-425 (approximate) - Added `slo_target` to `insertMonitor()` data mapping
     - Lines 470-475 (approximate) - Added `slo_target` to `updateMonitor()` conditional updates
   - **Justification**: Enables persistence of SLO target configuration

**Database**:
- Field: `slo_target DECIMAL(5,4) NULL` (target percentage, 90-100%)

**Dependencies**: 
- Requires Feature 4 (SLI) for SLA evaluation
- Can reference SLI values for target validation

**Deployment Order**: Phase 3 (after SLI)

---

### 🚧 Feature 6: SLA (Service Level Agreement)

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Service Level Agreement - evaluates compliance (SLI >= SLO).

**Key Capabilities**:
- Evaluates SLA compliance: `sla_violated = (sli_value < slo_target)`
- Stores `sla_violated` boolean flag
- Automatic evaluation on status updates
- Indexed for fast violation queries
- Historical tracking in `sli_history` table

**EPIC: SLA (Service Level Agreement) Compliance Evaluation System**

**Stories**:

1. **Story 6.1: Database Schema for SLA Violation Flag**
   - **Description**: Add `sla_violated` field and indexes to monitors table
   - **Acceptance Criteria**:
     - `sla_violated` field added (BOOLEAN, default false)
     - Indexes created on `sla_violated` and `(monitor_type, sla_violated)`
     - Migration includes rollback function
   - **Files**: `migrations/20250115120000_add_sla_slo_sli_fields.js`
   - **Before**: No SLA violation tracking
   - **After**: 
     - Line 43 - Added `sla_violated BOOLEAN DEFAULT false` field
     - Lines 52-53 - Added indexes `idx_monitors_sla_violated` and `idx_monitors_type_sla`
   - **Justification**: Enables fast queries for SLA violations and UI filtering

2. **Story 6.2: SLA Evaluation Utility Function**
   - **Description**: Create function to evaluate SLA compliance
   - **Acceptance Criteria**:
     - `evaluateSLA()` function compares SLI value to SLO target
     - Returns `violated: true` if `sli_value < slo_target`
     - Returns margin (difference between SLI and SLO)
     - Validates inputs and handles edge cases
   - **Files**: `src/lib/server/slaUtils.js`
   - **Before**: No SLA evaluation function
   - **After**: Lines 147-197 - `evaluateSLA()` function with validation and compliance logic
   - **Justification**: Centralizes SLA compliance evaluation logic

3. **Story 6.3: SLA Evaluation Integration for GROUP/SUPERGROUP Monitors**
   - **Description**: Integrate SLA evaluation into status aggregation workflow
   - **Acceptance Criteria**:
     - SLA evaluated when SLI is calculated for GROUP/SUPERGROUP monitors
     - Updates `sla_violated` flag in database
     - Only evaluates if SLO target is set
     - Logs SLA evaluation results
   - **Files**: `src/lib/server/controllers/controller.js`
   - **Before**: No SLA evaluation in `updateSLIAndSLA()`
   - **After**: 
     - Line 22 - Import `evaluateSLA` from `slaUtils.js`
     - Lines 2287-2292 - SLA evaluation in `updateSLIAndSLA()` when SLO target exists
     - Lines 2295-2299 - Update `sla_violated` flag in database
   - **Justification**: Automatically evaluates SLA compliance when status changes

4. **Story 6.4: SLA Evaluation Integration for Leaf Monitors**
   - **Description**: Integrate SLA evaluation for leaf monitors
   - **Acceptance Criteria**:
     - SLA evaluated when SLI is calculated for leaf monitors
     - Updates `sla_violated` flag in database
     - Only evaluates if SLO target is set
   - **Files**: `src/lib/server/controllers/controller.js`
   - **Before**: No SLA evaluation in `updateSLIForLeafMonitor()`
   - **After**: 
     - Lines 2380-2385 - SLA evaluation in `updateSLIForLeafMonitor()` when SLO target exists
     - Lines 2388-2392 - Update `sla_violated` flag in database
   - **Justification**: Provides SLA compliance for leaf monitors with SLO targets

5. **Story 6.5: SLA Re-evaluation on SLO Target Change**
   - **Description**: Re-evaluate SLA when SLO target is updated
   - **Acceptance Criteria**:
     - When SLO target is updated, SLA is re-evaluated with current SLI value
     - Updates `sla_violated` flag if compliance status changes
     - Handles case where SLI value is null
   - **Files**: `src/lib/server/controllers/controller.js`
   - **Before**: No SLA re-evaluation on SLO target update
   - **After**: Lines 2484-2491 - Re-evaluate SLA when SLO target is updated in `UpdateMonitor()`
   - **Justification**: Ensures SLA compliance is current when SLO target changes

6. **Story 6.6: Database Layer Support for SLA Violation Flag**
   - **Description**: Update database layer to handle `sla_violated` field
   - **Acceptance Criteria**:
     - `insertMonitor()` accepts `sla_violated`
     - `updateMonitor()` accepts `sla_violated` for updates
     - Field defaults to false (backward compatible)
   - **Files**: `src/lib/server/db/dbimpl.js`
   - **Before**: No `sla_violated` field in insertMonitor/updateMonitor
   - **After**: 
     - Lines 420-425 (approximate) - Added `sla_violated` to `insertMonitor()` data mapping
     - Lines 470-475 (approximate) - Added `sla_violated` to `updateMonitor()` conditional updates
   - **Justification**: Enables persistence of SLA compliance status

**Database**:
- Field: `sla_violated BOOLEAN DEFAULT false`
- Indexes: `idx_monitors_sla_violated`, `idx_monitors_type_sla`

**Dependencies**: 
- Requires Feature 4 (SLI) for SLI calculation
- Requires Feature 5 (SLO) for target comparison
- Cannot be deployed without both SLI and SLO

**Deployment Order**: Phase 4 (after SLI and SLO)

---

### 🚧 Feature 7: KPI (Key Performance Indicators)

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Configurable KPIs (MTTR, MTBF, Error Rate, Uptime, Incident Count, SLA Violation Count).

**Key Capabilities**:
- Multiple KPI types: MTTR, MTBF, ERROR_RATE, UPTIME, ERROR_BUDGET_BURN_RATE, INCIDENT_COUNT, SLA_VIOLATION_COUNT
- JSON-based storage for multiple KPIs per monitor
- Automatic calculation based on monitoring data
- Evaluation periods: DAILY, WEEKLY, MONTHLY, ROLLING_30D
- Target values for comparison

**Files**:
- `migrations/20250115120003_add_kpi_fields.js`
- `src/lib/server/kpiUtils.js` (Utility functions: `validateKPI()`, `evaluateKPIs()`, `getMonitoringDataForKPI()`)
- `src/lib/server/controllers/controller.js` (KPI evaluation integration)

**Database**:
- Fields: `kpi_definitions TEXT NULL` (JSON array), `kpi_last_evaluated_at TIMESTAMP NULL`
- Index: `idx_monitors_kpi_evaluated`

**Dependencies**: 
- Requires Feature 6 (SLA) for SLA violation count KPI
- Can reference Feature 4 (SLI) for uptime KPI
- Can be deployed independently after SLA

**Deployment Order**: Phase 5 (after SLA)

---

### 🚧 Feature 8: OKR (Objectives and Key Results)

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Business objectives tracking with measurable key results (popularized by Google).

**Key Capabilities**:
- Define high-level objectives at GROUP/SUPERGROUP level
- Multiple Key Results per objective
- Key Results can reference: SLI, SLO, SLA_COMPLIANCE, MTTR, MTBF, ERROR_BUDGET_BURN_RATE, INCIDENT_COUNT, KPI
- Automatic status evaluation: ON_TRACK, AT_RISK, OFF_TRACK, NOT_SET
- Status based on Key Results achievement percentage

**EPIC: OKR (Objectives and Key Results) Evaluation System**

**Stories**:

1. **Story 8.1: Database Schema for OKR Fields**
   - **Description**: Add database fields for OKR objective, key results, and status tracking
   - **Acceptance Criteria**:
     - `objective` field added (TEXT, nullable)
     - `key_results` field added (TEXT, nullable, stores JSON array)
     - `okr_status` field added (VARCHAR(20), default 'NOT_SET')
     - `okr_last_evaluated_at` field added (TIMESTAMP, nullable)
     - Indexes on `okr_status` and `(monitor_type, okr_status)` for performance
     - Migration includes rollback function
   - **Files**: `migrations/20250115120002_add_okr_fields.js` (NEW)
   - **Before**: No OKR fields in `monitors` table
   - **After**: 
     - Line 21 - Added `objective TEXT NULL` field
     - Line 22 - Added `key_results TEXT NULL` field
     - Line 23 - Added `okr_status VARCHAR(20) DEFAULT 'NOT_SET'` field
     - Line 24 - Added `okr_last_evaluated_at TIMESTAMP NULL` field
     - Lines 27-28 - Added indexes `idx_monitors_okr_status` and `idx_monitors_type_okr`
   - **Justification**: Stores OKR data and status for business objective tracking

2. **Story 8.2: OKR Utility Functions**
   - **Description**: Create utility functions for OKR validation and evaluation
   - **Acceptance Criteria**:
     - `validateOKR()` function validates OKR data structure
     - `evaluateOKR()` function evaluates OKR status based on Key Results
     - `evaluateKeyResult()` function evaluates individual Key Results
     - `getCurrentMetricsForOKR()` function retrieves current metric values for Key Results
     - Supports Key Results referencing: SLI, SLO, SLA_COMPLIANCE, MTTR, MTBF, ERROR_BUDGET_BURN_RATE, INCIDENT_COUNT, KPI
   - **Files**: `src/lib/server/okrUtils.js` (NEW)
   - **Before**: No OKR evaluation functions
   - **After**: 
     - Lines 46-150 (approximate) - `validateOKR()` function with comprehensive validation
     - Lines 200-300 (approximate) - `evaluateOKR()` function with status calculation
     - Lines 350-450 (approximate) - `evaluateKeyResult()` function for individual Key Results
     - Lines 500-600 (approximate) - `getCurrentMetricsForOKR()` function for metric retrieval
   - **Justification**: Centralizes OKR evaluation logic for reuse across the platform

3. **Story 8.3: OKR Evaluation Integration**
   - **Description**: Integrate OKR evaluation into status update workflow
   - **Acceptance Criteria**:
     - OKRs evaluated after SLI/SLA updates for GROUP/SUPERGROUP monitors
     - Updates `okr_status` based on Key Results achievement
     - Updates `okr_last_evaluated_at` timestamp
     - Logs OKR evaluation results
   - **Files**: `src/lib/server/controllers/controller.js`
   - **Before**: No OKR evaluation in workflow
   - **After**: 
     - Line 23 - Import `validateOKR`, `evaluateOKR`, `getCurrentMetricsForOKR`, `OKR_STATUS` from `okrUtils.js`
     - Lines 2312-2321 - Call `evaluateAndUpdateOKR()` after SLI/SLA update in `updateSLIAndSLA()`
     - Lines 2405-2416 - Call `evaluateAndUpdateOKR()` after SLI/SLA update in `updateSLIForLeafMonitor()` (for GROUP/SUPERGROUP only)
     - Lines 2560-2640 - `evaluateAndUpdateOKR()` function evaluates OKRs and updates database
   - **Justification**: Automatically evaluates OKR status when metrics change

4. **Story 8.4: Database Layer Support for OKR Fields**
   - **Description**: Update database layer to handle OKR fields in insert/update operations
   - **Acceptance Criteria**:
     - `insertMonitor()` accepts `objective`, `key_results`, `okr_status`, `okr_last_evaluated_at`
     - `updateMonitor()` accepts OKR fields for updates
     - Fields are nullable (backward compatible)
   - **Files**: `src/lib/server/db/dbimpl.js`
   - **Before**: No OKR fields in insertMonitor/updateMonitor
   - **After**: 
     - Lines 420-425 (approximate) - Added OKR fields to `insertMonitor()` data mapping
     - Lines 470-475 (approximate) - Added OKR fields to `updateMonitor()` conditional updates
   - **Justification**: Enables persistence of OKR data and status

**Database**:
- Fields: `objective TEXT NULL`, `key_results TEXT NULL` (JSON array), `okr_status VARCHAR(20) DEFAULT 'NOT_SET'`, `okr_last_evaluated_at TIMESTAMP NULL`
- Indexes: `idx_monitors_okr_status`, `idx_monitors_type_okr`

**Dependencies**: 
- Requires Feature 4 (SLI) for SLI-based Key Results
- Requires Feature 5 (SLO) for SLO-based Key Results
- Requires Feature 6 (SLA) for SLA compliance Key Results
- Requires Feature 7 (KPI) for KPI-based Key Results
- Primarily for GROUP/SUPERGROUP monitors

**Deployment Order**: Phase 6 (after KPI - last feature)

---

## Email Alert Features (Phase 7-11)

### 🚧 Feature 9: SLI Email Alerts

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Email notifications when SLI (Service Level Indicator) falls below configured threshold.

**Key Capabilities**:
- Configurable SLI threshold (0-100%)
- Multiple email recipients per monitor
- Cooldown period to prevent alert spam (default: 60 minutes)
- HTML email templates with monitor details
- Automatic alert triggering on SLI calculation
- Alert state tracking to prevent duplicates

**EPIC: SLI Email Alert System**

**Stories**:

1. **Story 9.1: Database Schema for SLI Alerts**
   - **Description**: Add `sli_alert_config` field to monitors table
   - **Acceptance Criteria**:
     - Field stores JSON: `{enabled: boolean, threshold: number, recipients: string[], cooldown_minutes: number, last_alert_at: timestamp}`
     - Field is nullable (backward compatible)
     - Migration includes rollback function
   - **Files**: `migrations/20250115120004_add_metric_alert_fields.js` (partial)

2. **Story 9.2: SLI Alert Configuration Validation**
   - **Description**: Validate SLI alert configuration on monitor create/update
   - **Acceptance Criteria**:
     - Validates threshold is 0-100
     - Validates recipients are valid email addresses
     - Validates cooldown_minutes is non-negative
     - Returns clear error messages
   - **Files**: `src/lib/server/metricAlertUtils.js` (`validateMetricAlertConfig()`)

3. **Story 9.3: SLI Alert Triggering Logic**
   - **Description**: Check if SLI alert should be sent based on current SLI value
   - **Acceptance Criteria**:
     - Triggers when `sli_value < threshold`
     - Respects cooldown period
     - Checks if alerts are enabled
     - Handles missing SLI values gracefully
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendSLIAlert()`, `isCooldownPassed()`)

4. **Story 9.4: SLI Email Template**
   - **Description**: Create HTML email template for SLI alerts
   - **Acceptance Criteria**:
     - Displays monitor name, current SLI, threshold, margin
     - Includes link to monitor detail page
     - Responsive design for mobile
     - Matches existing email template style
   - **Files**: `src/lib/server/templates/metric_alert_sli.html` (NEW)

5. **Story 9.5: SLI Alert Integration**
   - **Description**: Integrate SLI alerting into SLI calculation workflow
   - **Acceptance Criteria**:
     - Alerts triggered after SLI calculation in `controller.js`
     - Updates `metric_alert_last_sent` timestamp
     - Logs alert attempts for debugging
     - Handles email sending errors gracefully
   - **Files**: `src/lib/server/controllers/controller.js` (SLI calculation section)

6. **Story 9.6: UI for SLI Alert Configuration**
   - **Description**: Add UI fields in monitor sheet for SLI alert configuration
   - **Acceptance Criteria**:
     - Toggle to enable/disable SLI alerts
     - Input field for threshold (0-100)
     - Multi-email input for recipients
     - Cooldown minutes input
     - Validation feedback
   - **Files**: `src/lib/components/manage/monitorSheet.svelte` (SLI alert section)

**Database**:
- Field: `sli_alert_config TEXT NULL` (JSON configuration)
- Field: `metric_alert_last_sent TIMESTAMP NULL` (shared across all metric alerts)

**Dependencies**: 
- Requires Feature 4 (SLI) - cannot deploy without SLI calculation
- Requires existing email infrastructure (Resend/SMTP)

**Deployment Order**: Phase 7 (after SLI - Feature 4)

---

### 🚧 Feature 10: SLO Email Alerts

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Email notifications when SLO (Service Level Objective) target falls below configured threshold.

**Key Capabilities**:
- Configurable SLO threshold (0-100%)
- Multiple email recipients per monitor
- Cooldown period to prevent alert spam
- HTML email templates with SLO details
- Automatic alert triggering on SLO evaluation

**EPIC: SLO Email Alert System**

**Stories**:

1. **Story 10.1: Database Schema for SLO Alerts**
   - **Description**: Add `slo_alert_config` field to monitors table
   - **Acceptance Criteria**:
     - Field stores JSON: `{enabled: boolean, threshold: number, recipients: string[], cooldown_minutes: number, last_alert_at: timestamp}`
     - Field is nullable
   - **Files**: `migrations/20250115120004_add_metric_alert_fields.js` (partial)

2. **Story 10.2: SLO Alert Configuration Validation**
   - **Description**: Validate SLO alert configuration
   - **Acceptance Criteria**:
     - Validates threshold is 0-100
     - Validates recipients are valid email addresses
     - Validates cooldown_minutes
   - **Files**: `src/lib/server/metricAlertUtils.js` (`validateMetricAlertConfig()`)

3. **Story 10.3: SLO Alert Triggering Logic**
   - **Description**: Check if SLO alert should be sent
   - **Acceptance Criteria**:
     - Triggers when `slo_target < threshold`
     - Respects cooldown period
     - Checks if alerts are enabled
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendSLOAlert()`)

4. **Story 10.4: SLO Email Template**
   - **Description**: Create HTML email template for SLO alerts
   - **Acceptance Criteria**:
     - Displays monitor name, current SLO target, threshold
     - Includes link to monitor detail page
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_slo.html` (NEW)

5. **Story 10.5: SLO Alert Integration**
   - **Description**: Integrate SLO alerting into SLO evaluation workflow
   - **Acceptance Criteria**:
     - Alerts triggered after SLO target validation
     - Updates `metric_alert_last_sent` timestamp
     - Logs alert attempts
   - **Files**: `src/lib/server/controllers/controller.js` (SLO evaluation section)

6. **Story 10.6: UI for SLO Alert Configuration**
   - **Description**: Add UI fields for SLO alert configuration
   - **Acceptance Criteria**:
     - Toggle to enable/disable SLO alerts
     - Input field for threshold
     - Multi-email input for recipients
     - Cooldown minutes input
   - **Files**: `src/lib/components/manage/monitorSheet.svelte` (SLO alert section)

**Database**:
- Field: `slo_alert_config TEXT NULL` (JSON configuration)

**Dependencies**: 
- Requires Feature 5 (SLO) - cannot deploy without SLO configuration
- Requires Feature 9 (SLI Alerts) for shared infrastructure

**Deployment Order**: Phase 8 (after SLO - Feature 5)

---

### 🚧 Feature 11: SLA Email Alerts

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Email notifications when SLA (Service Level Agreement) is violated.

**Key Capabilities**:
- Automatic triggering on SLA violation (`sla_violated = true`)
- Multiple email recipients per monitor
- Cooldown period to prevent alert spam
- HTML email templates with SLI/SLO comparison
- No threshold needed (violation is binary)

**EPIC: SLA Email Alert System**

**Stories**:

1. **Story 11.1: Database Schema for SLA Alerts**
   - **Description**: Add `sla_alert_config` field to monitors table
   - **Acceptance Criteria**:
     - Field stores JSON: `{enabled: boolean, recipients: string[], cooldown_minutes: number, last_alert_at: timestamp}`
     - No threshold field (violation is binary)
   - **Files**: `migrations/20250115120004_add_metric_alert_fields.js` (partial)

2. **Story 11.2: SLA Alert Configuration Validation**
   - **Description**: Validate SLA alert configuration
   - **Acceptance Criteria**:
     - Validates recipients are valid email addresses
     - Validates cooldown_minutes
     - No threshold validation needed
   - **Files**: `src/lib/server/metricAlertUtils.js` (`validateMetricAlertConfig()`)

3. **Story 11.3: SLA Alert Triggering Logic**
   - **Description**: Check if SLA alert should be sent
   - **Acceptance Criteria**:
     - Triggers when `sla_violated = true`
     - Respects cooldown period
     - Checks if alerts are enabled
     - Does not trigger if SLA is compliant
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendSLAAlert()`)

4. **Story 11.4: SLA Email Template**
   - **Description**: Create HTML email template for SLA violation alerts
   - **Acceptance Criteria**:
     - Displays monitor name, SLI value, SLO target, margin
     - Highlights violation status
     - Includes link to monitor detail page
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_sla.html` (NEW)

5. **Story 11.5: SLA Alert Integration**
   - **Description**: Integrate SLA alerting into SLA evaluation workflow
   - **Acceptance Criteria**:
     - Alerts triggered after SLA evaluation in `controller.js`
     - Updates `metric_alert_last_sent` timestamp
     - Only triggers on violation (not on compliance)
     - Logs alert attempts
   - **Files**: `src/lib/server/controllers/controller.js` (SLA evaluation section)

6. **Story 11.6: UI for SLA Alert Configuration**
   - **Description**: Add UI fields for SLA alert configuration
   - **Acceptance Criteria**:
     - Toggle to enable/disable SLA alerts
     - Multi-email input for recipients
     - Cooldown minutes input
     - No threshold input (violation is binary)
   - **Files**: `src/lib/components/manage/monitorSheet.svelte` (SLA alert section)

**Database**:
- Field: `sla_alert_config TEXT NULL` (JSON configuration)

**Dependencies**: 
- Requires Feature 6 (SLA) - cannot deploy without SLA evaluation
- Requires Feature 9 (SLI Alerts) for shared infrastructure

**Deployment Order**: Phase 9 (after SLA - Feature 6)

---

### 🚧 Feature 12: KPI Email Alerts

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Email notifications when KPI (Key Performance Indicator) values cross configured thresholds.

**Key Capabilities**:
- Per-KPI alert configuration (multiple KPIs per monitor)
- Threshold-based alerts (above/below comparison)
- Multiple email recipients per KPI alert
- Cooldown period to prevent alert spam
- HTML email templates with KPI details

**EPIC: KPI Email Alert System**

**Stories**:

1. **Story 12.1: Database Schema for KPI Alerts**
   - **Description**: Add `kpi_alert_config` field to monitors table
   - **Acceptance Criteria**:
     - Field stores JSON: `{enabled: boolean, kpi_alerts: [{kpi_name, threshold, comparison: 'above'|'below', recipients}], cooldown_minutes: number, last_alert_at: timestamp}`
     - Supports multiple KPI alerts per monitor
   - **Files**: `migrations/20250115120004_add_metric_alert_fields.js` (partial)

2. **Story 12.2: KPI Alert Configuration Validation**
   - **Description**: Validate KPI alert configuration
   - **Acceptance Criteria**:
     - Validates each KPI alert has `kpi_name`, `threshold`, `comparison`
     - Validates `comparison` is 'above' or 'below'
     - Validates recipients are valid email addresses
     - Validates threshold is numeric
   - **Files**: `src/lib/server/metricAlertUtils.js` (`validateMetricAlertConfig()`)

3. **Story 12.3: KPI Alert Triggering Logic**
   - **Description**: Check if KPI alert should be sent for each configured KPI
   - **Acceptance Criteria**:
     - Triggers when `current_value > threshold` (if comparison='above')
     - Triggers when `current_value < threshold` (if comparison='below')
     - Respects cooldown period
     - Checks if alerts are enabled
     - Handles missing KPI values gracefully
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendKPIAlert()`)

4. **Story 12.4: KPI Email Template**
   - **Description**: Create HTML email template for KPI alerts
   - **Acceptance Criteria**:
     - Displays monitor name, KPI name, current value, threshold, comparison
     - Includes link to monitor detail page
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_kpi.html` (NEW)

5. **Story 12.5: KPI Alert Integration**
   - **Description**: Integrate KPI alerting into KPI evaluation workflow
   - **Acceptance Criteria**:
     - Alerts triggered after KPI calculation in `controller.js`
     - Updates `metric_alert_last_sent` timestamp
     - Handles multiple KPI alerts per monitor
     - Logs alert attempts
   - **Files**: `src/lib/server/controllers/controller.js` (KPI evaluation section)

6. **Story 12.6: UI for KPI Alert Configuration**
   - **Description**: Add UI fields for KPI alert configuration
   - **Acceptance Criteria**:
     - Toggle to enable/disable KPI alerts
     - Dynamic list of KPI alerts (add/remove)
     - For each KPI alert: KPI name dropdown, threshold input, comparison dropdown (above/below), recipients input
     - Cooldown minutes input
     - Validation feedback
   - **Files**: `src/lib/components/manage/monitorSheet.svelte` (KPI alert section)

**Database**:
- Field: `kpi_alert_config TEXT NULL` (JSON configuration with array of KPI alerts)

**Dependencies**: 
- Requires Feature 7 (KPI) - cannot deploy without KPI calculation
- Requires Feature 9 (SLI Alerts) for shared infrastructure

**Deployment Order**: Phase 10 (after KPI - Feature 7)

---

### 🚧 Feature 13: OKR Email Alerts

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Email notifications when OKR (Objectives and Key Results) status changes to AT_RISK or OFF_TRACK.

**Key Capabilities**:
- Status-based alerts (AT_RISK, OFF_TRACK)
- Multiple email recipients per monitor
- Cooldown period to prevent alert spam
- HTML email templates with OKR details and Key Results
- Automatic alert triggering on OKR status change

**EPIC: OKR Email Alert System**

**Stories**:

1. **Story 13.1: Database Schema for OKR Alerts**
   - **Description**: Add `okr_alert_config` field to monitors table
   - **Acceptance Criteria**:
     - Field stores JSON: `{enabled: boolean, status_alerts: ['AT_RISK'|'OFF_TRACK'], recipients: string[], cooldown_minutes: number, last_alert_at: timestamp}`
     - Supports multiple status types in alert list
   - **Files**: `migrations/20250115120004_add_metric_alert_fields.js` (partial)

2. **Story 13.2: OKR Alert Configuration Validation**
   - **Description**: Validate OKR alert configuration
   - **Acceptance Criteria**:
     - Validates `status_alerts` array contains only 'AT_RISK' or 'OFF_TRACK'
     - Validates recipients are valid email addresses
     - Validates cooldown_minutes
   - **Files**: `src/lib/server/metricAlertUtils.js` (`validateMetricAlertConfig()`)

3. **Story 13.3: OKR Alert Triggering Logic**
   - **Description**: Check if OKR alert should be sent based on OKR status
   - **Acceptance Criteria**:
     - Triggers when `okr_status` is in `status_alerts` array
     - Does not trigger for ON_TRACK or NOT_SET
     - Respects cooldown period
     - Checks if alerts are enabled
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendOKRAlert()`)

4. **Story 13.4: OKR Email Template**
   - **Description**: Create HTML email template for OKR alerts
   - **Acceptance Criteria**:
     - Displays monitor name, OKR status, objective, Key Results with achievement status
     - Highlights which Key Results are achieved/not achieved
     - Includes link to monitor detail page
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_okr.html` (NEW)

5. **Story 13.5: OKR Alert Integration**
   - **Description**: Integrate OKR alerting into OKR evaluation workflow
   - **Acceptance Criteria**:
     - Alerts triggered after OKR evaluation in `controller.js`
     - Updates `metric_alert_last_sent` timestamp
     - Only triggers on status change to AT_RISK or OFF_TRACK
     - Logs alert attempts
   - **Files**: `src/lib/server/controllers/controller.js` (OKR evaluation section)

6. **Story 13.6: UI for OKR Alert Configuration**
   - **Description**: Add UI fields for OKR alert configuration
   - **Acceptance Criteria**:
     - Toggle to enable/disable OKR alerts
     - Checkboxes for status alerts (AT_RISK, OFF_TRACK)
     - Multi-email input for recipients
     - Cooldown minutes input
     - Validation feedback
   - **Files**: `src/lib/components/manage/monitorSheet.svelte` (OKR alert section)

**Database**:
- Field: `okr_alert_config TEXT NULL` (JSON configuration)

**Dependencies**: 
- Requires Feature 8 (OKR) - cannot deploy without OKR evaluation
- Requires Feature 9 (SLI Alerts) for shared infrastructure

**Deployment Order**: Phase 11 (after OKR - Feature 8, last feature)

---

### 🚧 Feature 14: Email Templates for Metric Alerts

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Professional HTML email templates for all metric alert types with responsive design and branding.

**Key Capabilities**:
- Responsive HTML email templates for all alert types
- Consistent branding and styling
- Color-coded headers (red for SLI/SLA, orange for SLO/OKR, purple for KPI)
- Metric details displayed clearly
- Direct links to monitor detail pages
- Mobile-friendly design

**EPIC: Professional Email Templates for Metric Alerts**

**Stories**:

1. **Story 14.1: SLI Email Template**
   - **Description**: Create HTML email template for SLI alerts
   - **Acceptance Criteria**:
     - Responsive design (works on mobile and desktop)
     - Red header indicating alert severity
     - Displays monitor name, current SLI, threshold, margin
     - Includes "View Monitor Details" button
     - Matches existing email template style
   - **Files**: `src/lib/server/templates/metric_alert_sli.html` (NEW)

2. **Story 14.2: SLO Email Template**
   - **Description**: Create HTML email template for SLO alerts
   - **Acceptance Criteria**:
     - Orange header for warning level
     - Displays monitor name, current SLO target, threshold, margin
     - Includes "View Monitor Details" button
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_slo.html` (NEW)

3. **Story 14.3: SLA Email Template**
   - **Description**: Create HTML email template for SLA violation alerts
   - **Acceptance Criteria**:
     - Red header indicating critical violation
     - Displays monitor name, SLI value, SLO target, margin
     - Highlights "SLA Compliance: VIOLATED" status
     - Includes "View Monitor Details" button
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_sla.html` (NEW)

4. **Story 14.4: KPI Email Template**
   - **Description**: Create HTML email template for KPI alerts
   - **Acceptance Criteria**:
     - Purple header for KPI alerts
     - Displays monitor name, KPI name, current value, threshold, comparison
     - Includes "View Monitor Details" button
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_kpi.html` (NEW)

5. **Story 14.5: OKR Email Template**
   - **Description**: Create HTML email template for OKR status alerts
   - **Acceptance Criteria**:
     - Orange header for OKR alerts
     - Displays monitor name, OKR status, objective
     - Shows Key Results status (if available)
     - Includes "View Monitor Details" button
     - Responsive design
   - **Files**: `src/lib/server/templates/metric_alert_okr.html` (NEW)

**Files**:
- `src/lib/server/templates/metric_alert_sli.html` (NEW)
- `src/lib/server/templates/metric_alert_slo.html` (NEW)
- `src/lib/server/templates/metric_alert_sla.html` (NEW)
- `src/lib/server/templates/metric_alert_kpi.html` (NEW)
- `src/lib/server/templates/metric_alert_okr.html` (NEW)
- `src/lib/server/metricAlertUtils.js` (Updated to use templates)

**Dependencies**: 
- Requires Feature 9-13 (Email Alert Features) - templates are used by alert functions

**Deployment Order**: Phase 12 (can be deployed with Phase 7-11 or separately)

---

### 🚧 Feature 15: Alert History Tracking

**Status**: 🚧 **CODE COMPLETE, READY FOR DEPLOYMENT**

**Description**: Comprehensive alert history tracking for audit, analysis, and troubleshooting.

**Key Capabilities**:
- Tracks all alert attempts (SENT, FAILED, SKIPPED)
- Stores metric values, thresholds, recipients
- Records error messages for failed alerts
- Supports filtering by monitor, alert type, status
- Paginated history queries
- Foreign key constraints for data integrity

**EPIC: Metric Alert History System**

**Stories**:

1. **Story 15.1: Database Schema for Alert History**
   - **Description**: Create `metric_alert_history` table
   - **Acceptance Criteria**:
     - Table stores: monitor_tag, alert_type, alert_status, metric_value, threshold, recipients, error_message, alert_details, created_at
     - Indexes on monitor_tag, alert_type, alert_status, created_at
     - Foreign key constraint on monitor_tag (CASCADE delete)
     - Migration includes rollback function
   - **Files**: `migrations/20250115120005_create_metric_alert_history_table.js` (NEW)

2. **Story 15.2: Alert History Database Functions**
   - **Description**: Add database functions for alert history CRUD operations
   - **Acceptance Criteria**:
     - `insertMetricAlertHistory()` - Insert new alert history record
     - `getMetricAlertHistory()` - Get history for a monitor
     - `getMetricAlertHistoryPaginated()` - Paginated history with filters
     - `getMetricAlertHistoryCount()` - Count with filters
     - All functions handle JSON parsing for recipients and alert_details
   - **Files**: `src/lib/server/db/dbimpl.js` (Alert history functions)

3. **Story 15.3: Alert History Recording in SLI Alerts**
   - **Description**: Record alert history when SLI alerts are triggered
   - **Acceptance Criteria**:
     - Records SENT status when emails sent successfully
     - Records FAILED status when all emails fail
     - Records SKIPPED status when threshold not crossed or cooldown active
     - Stores metric_value, threshold, recipients, error_message
     - Stores alert_details with sent_count, failed_count
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendSLIAlert()`)

4. **Story 15.4: Alert History Recording in SLO Alerts**
   - **Description**: Record alert history when SLO alerts are triggered
   - **Acceptance Criteria**:
     - Records SENT/FAILED/SKIPPED statuses
     - Stores SLO target, threshold, recipients
     - Records error messages for failures
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendSLOAlert()`)

5. **Story 15.5: Alert History Recording in SLA Alerts**
   - **Description**: Record alert history when SLA alerts are triggered
   - **Acceptance Criteria**:
     - Records SENT/FAILED/SKIPPED statuses
     - Stores SLI value, SLO target, recipients
     - Records error messages for failures
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendSLAAlert()`)

6. **Story 15.6: Alert History Recording in KPI Alerts**
   - **Description**: Record alert history when KPI alerts are triggered
   - **Acceptance Criteria**:
     - Records history for each triggered KPI separately
     - Stores KPI name, current value, threshold, comparison
     - Records SENT/FAILED/SKIPPED statuses per KPI
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendKPIAlert()`)

7. **Story 15.7: Alert History Recording in OKR Alerts**
   - **Description**: Record alert history when OKR alerts are triggered
   - **Acceptance Criteria**:
     - Records SENT/FAILED/SKIPPED statuses
     - Stores OKR status, objective, key_results_count in alert_details
     - Records error messages for failures
   - **Files**: `src/lib/server/metricAlertUtils.js` (`sendOKRAlert()`)

8. **Story 15.8: Alert History API Endpoints (Future)**
   - **Description**: Create API endpoints for querying alert history
   - **Acceptance Criteria**:
     - GET endpoint for alert history (paginated)
     - Filter by monitor_tag, alert_type, alert_status
     - Returns JSON with history records
     - Includes pagination metadata
   - **Files**: `src/routes/(manage)/manage/(app)/app/api/+server.js` (Future - not yet implemented)

9. **Story 15.9: Alert History UI (Future)**
   - **Description**: Create UI page for viewing alert history
   - **Acceptance Criteria**:
     - Table view of alert history
     - Filter by monitor, alert type, status
     - Pagination controls
     - Display alert details (metric value, threshold, recipients, errors)
   - **Files**: `src/routes/(manage)/manage/(app)/app/alert-history/+page.svelte` (Future - not yet implemented)

**Database**:
- Table: `metric_alert_history`
- Fields: `id`, `monitor_tag`, `alert_type`, `alert_status`, `metric_value`, `threshold`, `recipients` (JSON), `error_message`, `alert_details` (JSON), `created_at`
- Indexes: `idx_metric_alert_history_monitor_tag`, `idx_metric_alert_history_monitor_type`, `idx_metric_alert_history_status`, `idx_metric_alert_history_created_at`

**Dependencies**: 
- Requires Feature 9-13 (Email Alert Features) - history is recorded by alert functions

**Deployment Order**: Phase 13 (can be deployed with Phase 7-11 or separately)

---

## Deployment Strategy

### Incremental Rollout Plan

**Phase 1: Core Features** ✅ **DEPLOYED IN PRODUCTION**
1. ✅ SUPERGROUP Multi-Layer Support
2. ✅ Custom Tooltip Message (`customtooltip_message`)
3. ✅ Custom Monitor Link (`custommonitor_link`)

**Phase 2: SLI (Service Level Indicator)** 🚧 **READY FOR DEPLOYMENT**
4. 🚧 SLI calculation and storage
   - `degraded_weight`, `sli_value`, `evaluation_period_start/end`
   - `sli_history` table for historical tracking

**Phase 3: SLO (Service Level Objective)** 🚧 **READY FOR DEPLOYMENT**
5. 🚧 SLO target configuration
   - `slo_target` field (90-100% validation)

**Phase 4: SLA (Service Level Agreement)** 🚧 **READY FOR DEPLOYMENT**
6. 🚧 SLA compliance evaluation
   - `sla_violated` boolean flag
   - Automatic evaluation: `sla_violated = (sli_value < slo_target)`

**Phase 5: KPI (Key Performance Indicators)** 🚧 **READY FOR DEPLOYMENT**
7. 🚧 KPI definitions and calculation
   - `kpi_definitions` (JSON), `kpi_last_evaluated_at`

**Phase 6: OKR (Objectives and Key Results)** 🚧 **READY FOR DEPLOYMENT**
8. 🚧 OKR tracking and evaluation
   - `objective`, `key_results` (JSON), `okr_status`, `okr_last_evaluated_at`

**Phase 7: SLI Email Alerts** 🚧 **READY FOR DEPLOYMENT**
9. 🚧 SLI email alert system
   - `sli_alert_config` (JSON), email templates, alert triggering

**Phase 8: SLO Email Alerts** 🚧 **READY FOR DEPLOYMENT**
10. 🚧 SLO email alert system
    - `slo_alert_config` (JSON), email templates, alert triggering

**Phase 9: SLA Email Alerts** 🚧 **READY FOR DEPLOYMENT**
11. 🚧 SLA email alert system
    - `sla_alert_config` (JSON), email templates, violation alerts

**Phase 10: KPI Email Alerts** 🚧 **READY FOR DEPLOYMENT**
12. 🚧 KPI email alert system
    - `kpi_alert_config` (JSON), per-KPI alerts, email templates

**Phase 11: OKR Email Alerts** 🚧 **READY FOR DEPLOYMENT**
13. 🚧 OKR email alert system
    - `okr_alert_config` (JSON), status-based alerts, email templates

**Phase 12: Email Templates** 🚧 **READY FOR DEPLOYMENT**
14. 🚧 Professional HTML email templates
    - Responsive templates for all 5 alert types
    - Color-coded headers and clear metric display

**Phase 13: Alert History Tracking** 🚧 **READY FOR DEPLOYMENT**
15. 🚧 Alert history system
    - `metric_alert_history` table
    - Tracks all alert attempts (SENT, FAILED, SKIPPED)
    - Audit trail for troubleshooting

### Merge Strategy

#### Phase 1: Already Deployed ✅

Phase 1 features are already deployed and in production use:
- ✅ SUPERGROUP Multi-Layer Support
- ✅ Custom Tooltip Message (`customtooltip_message`)
- ✅ Custom Monitor Link (`custommonitor_link`)

#### Incremental Merge (For Future Phases)

**Each phase must be deployed separately and tested before moving to the next:**

1. **Phase 2: SLI** → Test → Deploy → Verify
2. **Phase 3: SLO** → Test → Deploy → Verify (requires SLI)
3. **Phase 4: SLA** → Test → Deploy → Verify (requires SLI + SLO)
4. **Phase 5: KPI** → Test → Deploy → Verify (requires SLA)
5. **Phase 6: OKR** → Test → Deploy → Verify (requires SLI + SLO + SLA + KPI)
6. **Phase 7: SLI Email Alerts** → Test → Deploy → Verify (requires SLI)
7. **Phase 8: SLO Email Alerts** → Test → Deploy → Verify (requires SLO + SLI Alerts)
8. **Phase 9: SLA Email Alerts** → Test → Deploy → Verify (requires SLA + SLI Alerts)
9. **Phase 10: KPI Email Alerts** → Test → Deploy → Verify (requires KPI + SLI Alerts)
10. **Phase 11: OKR Email Alerts** → Test → Deploy → Verify (requires OKR + SLI Alerts)
11. **Phase 12: Email Templates** → Test → Deploy → Verify (can be deployed with Phase 7-11 or separately)
12. **Phase 13: Alert History Tracking** → Test → Deploy → Verify (can be deployed with Phase 7-11 or separately)

**Important**: Each phase builds on previous phases. Do not skip phases. Email alert phases can be deployed after their corresponding metric phases. Email templates and alert history can be deployed together with alert phases or separately.

### Feature Flags (Optional)

For future features (Phase 2-11), consider feature flags:

```javascript
// Example feature flag (not implemented)
const FEATURES = {
  SLI: process.env.ENABLE_SLI === 'true',
  SLO: process.env.ENABLE_SLO === 'true',
  SLA: process.env.ENABLE_SLA === 'true',
  KPI: process.env.ENABLE_KPI === 'true',
  OKR: process.env.ENABLE_OKR === 'true',
  SLI_ALERTS: process.env.ENABLE_SLI_ALERTS === 'true',
  SLO_ALERTS: process.env.ENABLE_SLO_ALERTS === 'true',
  SLA_ALERTS: process.env.ENABLE_SLA_ALERTS === 'true',
  KPI_ALERTS: process.env.ENABLE_KPI_ALERTS === 'true',
  OKR_ALERTS: process.env.ENABLE_OKR_ALERTS === 'true'
};
```

**Current Status**: Phase 1 features are always enabled (no flags needed). Future phases can use feature flags for gradual rollout.

---

## File-Level Deployment Instructions

### Phase 1: Core Features (SUPERGROUP + Custom Tooltip/Link)

#### Step 1: Database Migration

**File**: `migrations/20260115100000_add_custom_tooltip_and_link_fields.js`

**Action**: Run migration
```bash
npm run migrate
```

**Verification**:
```sql
-- PostgreSQL/MySQL
SELECT column_name, data_type, is_nullable 
FROM information_schema.columns 
WHERE table_name = 'monitors' 
AND column_name IN ('customtooltip_message', 'custommonitor_link');

-- SQLite
.schema monitors
```

**Rollback** (if needed):
```bash
npx knex migrate:rollback
```

#### Step 2: Backend Code

**Files to Deploy** (in order):

1. **`src/lib/server/controllers/monitorValidators.js`** (NEW)
   - **Purpose**: Shared validation utilities
   - **Dependencies**: None
   - **Impact**: Removes duplicate validation code

2. **`src/lib/server/controllers/controller.js`**
   - **Changes**: 
     - Import `monitorValidators.js` (Line 24)
     - `CreateMonitor()` uses shared validation (Lines 203-248)
     - `UpdateMonitor()` uses shared validation (Lines 250-299)
     - `ProcessGroupUpdate()` has cycle protection (Lines 921-968)
   - **Dependencies**: `monitorValidators.js`
   - **Impact**: Validation refactored, no behavior change

3. **`src/lib/server/db/dbimpl.js`**
   - **Changes**:
     - `insertMonitor()` includes `customtooltip_message`, `custommonitor_link` (Lines 439-440)
     - `updateMonitor()` includes `customtooltip_message`, `custommonitor_link` (Lines 488-489)
   - **Dependencies**: None
   - **Impact**: Database operations support new fields

**Deployment Order**:
1. Deploy `monitorValidators.js` first
2. Deploy `controller.js` (depends on `monitorValidators.js`)
3. Deploy `dbimpl.js` (independent)

#### Step 3: Frontend Code

**Files to Deploy** (in order):

1. **`src/lib/components/manage/monitorsAdd.svelte`**
   - **Changes**:
     - Initializes `customtooltip_message: ""`, `custommonitor_link: ""` (Lines 95-96)
     - `createSuperGroupConfig()` allows GROUP and SUPERGROUP children (Lines 197-211)
   - **Dependencies**: None
   - **Impact**: UI supports new fields and SUPERGROUP multi-layer

2. **`src/lib/components/manage/monitorSheet.svelte`**
   - **Changes**:
     - Custom tooltip validation (Lines 143-152)
     - Custom link validation (Lines 154-166)
     - SUPERGROUP validation allows GROUP/SUPERGROUP children (Lines 330-343, 352-401)
     - UI fields for customization (Lines 770-806)
   - **Dependencies**: `monitorsAdd.svelte` (for initialization)
   - **Impact**: Form validation and UI for new features

3. **`src/lib/components/monitor.svelte`**
   - **Changes**:
     - Custom tooltip display (Lines 407-413)
     - Custom link click handler (Lines 192-228)
   - **Dependencies**: None
   - **Impact**: Heatmap displays custom tooltip and handles custom link clicks

**Deployment Order**:
1. Deploy `monitorsAdd.svelte` first (initialization)
2. Deploy `monitorSheet.svelte` (depends on initialization)
3. Deploy `monitor.svelte` (independent)

#### Step 4: Build and Deploy

```bash
# Build frontend
npm run build

# Start application (migrations run automatically)
npm start
```

---

### Phase 2: SLI (Service Level Indicator)

**Files to Deploy**:
1. `migrations/20250115120000_add_sla_slo_sli_fields.js` (partial: only `degraded_weight`, `sli_value`, `evaluation_period_start/end` fields)
2. `migrations/20250115120001_create_sli_history_table.js` (SLI history table)
3. `src/lib/server/slaUtils.js` (`calculateSLI()`, `calculateSLIFromData()` functions)
4. `src/lib/server/controllers/controller.js` (SLI calculation integration)

**Dependencies**: Requires Phase 1 (SUPERGROUP) for aggregated SLI calculation

**Deployment Order**:
1. Run migrations (partial fields from `20250115120000_add_sla_slo_sli_fields.js`)
2. Deploy backend code (SLI calculation functions)
3. Test SLI calculation for leaf monitors and GROUP/SUPERGROUP monitors
4. Verify `sli_history` table is populated correctly
5. Deploy frontend (if UI changes exist)

**Note**: The migration `20250115120000_add_sla_slo_sli_fields.js` adds all SLI/SLO/SLA fields. For Phase 2, only use the SLI-related fields. SLO and SLA fields will be used in later phases.

---

### Phase 3: SLO (Service Level Objective)

**Files to Deploy**:
1. `migrations/20250115120000_add_sla_slo_sli_fields.js` (partial: only `slo_target` field - already exists from Phase 2)
2. `src/lib/server/slaUtils.js` (`validateSLOTarget()` function)
3. `src/lib/components/manage/monitorSheet.svelte` (SLO target validation in UI)
4. `src/lib/server/db/dbimpl.js` (SLO field in insertMonitor/updateMonitor)

**Dependencies**: Requires Phase 2 (SLI) for future SLA evaluation

**Deployment Order**:
1. Verify `slo_target` field exists (from Phase 2 migration)
2. Deploy backend validation code
3. Deploy frontend UI for SLO target configuration
4. Test SLO target validation (90-100% range)
5. Verify SLO targets can be set for monitors

**Note**: The `slo_target` field was added in Phase 2 migration but not used. Phase 3 enables SLO target configuration.

---

### Phase 4: SLA (Service Level Agreement)

**Files to Deploy**:
1. `migrations/20250115120000_add_sla_slo_sli_fields.js` (partial: only `sla_violated` field and indexes - already exists from Phase 2)
2. `src/lib/server/slaUtils.js` (`evaluateSLA()` function)
3. `src/lib/server/controllers/controller.js` (Lines 868-875 - SLA evaluation on status change)
4. `src/lib/server/db/dbimpl.js` (SLA field in insertMonitor/updateMonitor)

**Dependencies**: Requires Phase 2 (SLI) and Phase 3 (SLO) - cannot deploy without both

**Deployment Order**:
1. Verify `sla_violated` field and indexes exist (from Phase 2 migration)
2. Deploy backend SLA evaluation code
3. Test SLA evaluation: `sla_violated = (sli_value < slo_target)`
4. Verify SLA violations are detected and stored
5. Deploy frontend (if UI changes exist for SLA display)

**Note**: The `sla_violated` field and indexes were added in Phase 2 migration but not used. Phase 4 enables SLA compliance evaluation.

---

### Phase 5: KPI (Key Performance Indicators)

**Files to Deploy**:
1. `migrations/20250115120003_add_kpi_fields.js`
2. `src/lib/server/kpiUtils.js` (Utility functions: `validateKPI()`, `evaluateKPIs()`, `getMonitoringDataForKPI()`)
3. `src/lib/server/controllers/controller.js` (KPI evaluation integration)
4. `src/lib/server/db/dbimpl.js` (KPI fields in insertMonitor/updateMonitor)

**Dependencies**: Requires Phase 4 (SLA) for SLA violation count KPI

**Deployment Order**:
1. Run migration: `20250115120003_add_kpi_fields.js`
2. Deploy backend KPI calculation code
3. Test KPI calculation for different types (MTTR, MTBF, ERROR_RATE, UPTIME, etc.)
4. Verify KPI definitions can be stored and evaluated
5. Deploy frontend (if UI changes exist for KPI configuration)

---

### Phase 6: OKR (Objectives and Key Results)

**Files to Deploy**:
1. `migrations/20250115120002_add_okr_fields.js`
2. `src/lib/server/okrUtils.js` (Utility functions: `validateOKR()`, `evaluateOKR()`, `evaluateKeyResult()`, `getCurrentMetricsForOKR()`)
3. `src/lib/server/controllers/controller.js` (OKR evaluation integration)
4. `src/lib/server/db/dbimpl.js` (OKR fields in insertMonitor/updateMonitor)

**Dependencies**: Requires Phase 2 (SLI), Phase 3 (SLO), Phase 4 (SLA), and Phase 5 (KPI) - cannot deploy without all previous phases

**Deployment Order**:
1. Run migration: `20250115120002_add_okr_fields.js`
2. Deploy backend OKR evaluation code
3. Test OKR evaluation with Key Results referencing SLI, SLO, SLA, KPI
4. Verify OKR status calculation (ON_TRACK, AT_RISK, OFF_TRACK, NOT_SET)
5. Deploy frontend (if UI changes exist for OKR configuration)

---

## Database Considerations

### CloudSQL PostgreSQL Performance

#### Indexes

**Existing Indexes** (from migrations):
- `idx_monitors_sla_violated` - Fast SLA violation queries
- `idx_monitors_type_sla` - Monitor type + SLA status queries
- `idx_monitors_okr_status` - OKR status queries
- `idx_monitors_type_okr` - Monitor type + OKR status queries

**Recommended Additional Indexes** (if needed):
```sql
-- For custom tooltip/link queries (if filtering by these fields)
CREATE INDEX idx_monitors_custom_link ON monitors(custommonitor_link) WHERE custommonitor_link IS NOT NULL;

-- For SUPERGROUP hierarchy queries (if needed)
CREATE INDEX idx_monitors_type ON monitors(monitor_type) WHERE monitor_type IN ('GROUP', 'SUPERGROUP');
```

#### Query Performance

**Status Propagation**:
- `ProcessGroupUpdate()` queries all active GROUP/SUPERGROUP monitors
- **Optimization**: Index on `monitor_type` + `status` (if not exists)
- **Impact**: Minimal (only runs on status changes, not on every request)

**Custom Fields**:
- `customtooltip_message` and `custommonitor_link` are nullable
- **Impact**: No performance impact (only loaded when monitor is fetched)

#### Migration Performance

**Phase 1 Migration** (`20260115100000_add_custom_tooltip_and_link_fields.js`):
- Adds 2 nullable TEXT columns
- **Impact**: Minimal (ALTER TABLE is fast for nullable columns)
- **Downtime**: None (nullable columns can be added online)

**Future Migrations** (Phase 2-3):
- SLI/SLO/SLA: Adds 6 columns + 1 table
- KPI: Adds 2 columns
- OKR: Adds 4 columns
- **Impact**: Minimal (all nullable/default columns)
- **Downtime**: None

### Database Backup

**Before Migration**:
```bash
# PostgreSQL
pg_dump -U username -d kener > backup-$(date +%Y%m%d).sql

# MySQL
mysqldump -u username -p kener > backup-$(date +%Y%m%d).sql

# SQLite
cp database/kener.sqlite.db database/kener.sqlite.db.backup-$(date +%Y%m%d)
```

**Rollback Plan**:
- All migrations have `down()` functions
- Run `npx knex migrate:rollback` to revert
- Restore backup if needed

---

## Testing & Validation

### Pre-Deployment Checklist

- [ ] Database backup created
- [ ] Migration tested on staging
- [ ] Code review completed
- [ ] All linter errors resolved
- [ ] No console errors in browser/server logs

### Functional Testing

#### Test 1: SUPERGROUP Multi-Layer

**Steps**:
1. Create SUPERGROUP monitor "Level 1"
2. Create SUPERGROUP monitor "Level 2"
3. Create GROUP monitor "Level 3"
4. Create leaf monitor (API/PING)
5. Add hierarchy: Level 1 → Level 2 → Level 3 → Leaf
6. Set leaf monitor to DOWN
7. Verify: Status propagates through all levels

**Expected**: All parent monitors show DOWN status

**Validation**:
- Check `monitoring_data` table for parent statuses
- Check console logs for `[ProcessGroupUpdate]` messages
- Verify no infinite loops (check depth in logs)

#### Test 2: Custom Tooltip Message

**Steps**:
1. Create/edit monitor
2. Enter custom tooltip: "Test tooltip message"
3. Save monitor
4. Navigate to monitor detail page
5. Hover over heatmap cell
6. Verify: Custom tooltip displays

**Expected**: Custom tooltip shows instead of default

**Validation**:
- Check database: `customtooltip_message` field saved
- Check UI: Tooltip displays custom message
- Test edge cases: Empty, null, > 2000 chars (should fail validation)

#### Test 3: Custom Monitor Link

**Steps**:
1. Create/edit monitor
2. Enter custom URL: "https://example.com/dashboard"
3. Save monitor
4. Navigate to monitor detail page
5. Click heatmap cell
6. Verify: URL opens with `start_time`, `end_time`, `clicked_time` parameters

**Expected**: Custom URL opens in new tab with time parameters

**Validation**:
- Check database: `custommonitor_link` field saved
- Check URL: Parameters present and valid
- Test edge cases: Invalid URL (should fail validation), empty (should use default)

#### Test 4: Cycle Prevention

**Steps**:
1. Create SUPERGROUP monitor "Test SuperGroup"
2. Edit "Test SuperGroup"
3. Verify: "Test SuperGroup" does NOT appear in selection list
4. Try to add via API (if possible)
5. Verify: Backend cycle protection prevents infinite loops

**Expected**: Cannot select itself, cycle protection works

#### Test 5: Backward Compatibility

**Steps**:
1. Check existing GROUP monitors
2. Verify: All existing monitors still work
3. Verify: Existing monitors have NULL for new fields
4. Verify: Default tooltip and click behavior work

**Expected**: Existing monitors unaffected

### Performance Testing

#### Database Query Performance

**Test**: Status propagation with 100 monitors
```bash
# Monitor query time
EXPLAIN ANALYZE SELECT * FROM monitors WHERE monitor_type IN ('GROUP', 'SUPERGROUP') AND status = 'ACTIVE';
```

**Expected**: Query time < 100ms (with index)

#### UI Rendering Performance

**Test**: Heatmap with 365 days of data
- **Expected**: Render time < 500ms
- **Tooltip**: Hover latency < 50ms

#### Status Propagation Performance

**Test**: Update leaf monitor in deep hierarchy (10 levels)
- **Expected**: Propagation completes in < 2 seconds
- **Logs**: Check for depth warnings (should not exceed 10)

### Regression Testing

**Critical Paths** (must not break):
1. ✅ GROUP monitor creation/update/delete
2. ✅ Leaf monitor creation/update/delete
3. ✅ Status aggregation for GROUP monitors
4. ✅ Heatmap rendering
5. ✅ Daily data modal (when custom link not set)
6. ✅ Default tooltip (when custom tooltip not set)

**Test Matrix**:

| Feature | Create | Update | Delete | Status Propagation | UI Display |
|---------|--------|--------|--------|-------------------|------------|
| Leaf Monitor | ✅ | ✅ | ✅ | ✅ | ✅ |
| GROUP Monitor | ✅ | ✅ | ✅ | ✅ | ✅ |
| SUPERGROUP Monitor | ✅ | ✅ | ✅ | ✅ | ✅ |
| Custom Tooltip | ✅ | ✅ | N/A | N/A | ✅ |
| Custom Link | ✅ | ✅ | N/A | N/A | ✅ |

---

## Performance Considerations

### Database Queries

**Optimization**:
- Indexes on `monitor_type`, `status` for GROUP/SUPERGROUP queries
- Indexes on `sla_violated`, `okr_status` for metrics queries
- No N+1 queries introduced

**Monitoring**:
- Track query execution time in logs
- Monitor database connection pool usage
- Alert on slow queries (> 1 second)

### Frontend Performance

**Optimization**:
- Tooltip rendering: Debounced hover (50ms)
- Heatmap rendering: Virtual scrolling for large datasets
- Custom link: URL parsing cached

**Monitoring**:
- Track page load time
- Monitor JavaScript errors
- Alert on slow renders (> 1 second)

### Status Propagation

**Optimization**:
- Only propagates when status actually changes
- Cycle protection prevents infinite loops
- Depth limit (10) prevents excessive recursion

**Monitoring**:
- Track propagation depth in logs
- Alert on depth > 8 (approaching limit)
- Monitor propagation time

---

## Troubleshooting

### Common Issues

#### Issue 1: Migration Fails

**Symptoms**: Error when running `npm run migrate`

**Solutions**:
1. Check database connection
2. Verify migration file syntax
3. Check for conflicting migrations
4. Review error message for specific issue

**Rollback**:
```bash
npx knex migrate:rollback
```

#### Issue 2: SUPERGROUP Not Showing Children

**Symptoms**: SUPERGROUP created but no status aggregation

**Solutions**:
1. Check console logs for `[ProcessGroupUpdate]` messages
2. Verify child monitors are ACTIVE
3. Verify child monitors have status in `monitoring_data`
4. Check for ignored leaf monitors in logs (SUPERGROUP ignores leaf monitors)

#### Issue 3: Custom Tooltip Not Showing

**Symptoms**: Custom tooltip message saved but not displaying

**Solutions**:
1. Verify field is saved in database
2. Check browser console for errors
3. Verify monitor object includes `customtooltip_message`
4. Check tooltip display code (line 407-413 in monitor.svelte)

#### Issue 4: Custom Link Not Opening

**Symptoms**: Custom link saved but click doesn't open URL

**Solutions**:
1. Verify URL is valid (http:// or https://)
2. Check browser console for errors
3. Verify `dailyDataGetter()` function is called
4. Check URL parsing logic (line 192-228 in monitor.svelte)

#### Issue 5: Validation Errors

**Symptoms**: Form validation fails unexpectedly

**Solutions**:
1. Check `monitorValidators.js` for validation logic
2. Verify frontend and backend validation match
3. Check error messages in console
4. Test with valid/invalid inputs

### Debugging

**Enable Debug Logging**:
```javascript
// In controller.js, add:
console.log('[DEBUG] Monitor data:', monitorData);
console.log('[DEBUG] Validation result:', validationResult);
```

**Check Database**:
```sql
-- Verify custom fields
SELECT id, name, customtooltip_message, custommonitor_link 
FROM monitors 
WHERE customtooltip_message IS NOT NULL OR custommonitor_link IS NOT NULL;

-- Verify SUPERGROUP hierarchy
SELECT tag, name, monitor_type, type_data 
FROM monitors 
WHERE monitor_type IN ('GROUP', 'SUPERGROUP');
```

---

## Code Quality & Refactoring

### Duplicate Code Removed

**Before**: Duplicate validation logic in `CreateMonitor()` and `UpdateMonitor()`

**After**: Shared validation in `src/lib/server/controllers/monitorValidators.js`

**Impact**: 
- ✅ Reduced code duplication
- ✅ Consistent validation logic
- ✅ Easier maintenance

### Error Handling

**Centralized**: All validation errors use consistent format
- Format: `IllegalArgumentException: <message>`
- Database errors: `Database access exception: <message>`

**Logging**: Comprehensive console logging for debugging
- `[CreateMonitor]`, `[UpdateMonitor]`, `[ProcessGroupUpdate]`
- Error messages include context (monitor tag, field name, etc.)

---

## Acceptance Criteria

### Phase 1 (Current)

- ✅ All new features validated and stable
- ✅ No regression in existing monitors
- ✅ Performance remains equal or improved
- ✅ Duplicate code removed
- ✅ Clear, developer-friendly README
- ✅ Feature rollout can be done incrementally
- ✅ Any developer can deploy by following README only

### Future Phases (2-3)

- 🚧 SLI/SLO/SLA metrics functional
- 🚧 KPI calculations accurate
- 🚧 OKR evaluation working
- 🚧 All features documented
- 🚧 Performance acceptable
- 🚧 No breaking changes

---

## Support & Documentation

### Single Source of Truth

**All documentation is now consolidated in this README.md file**, including:
- Feature implementation details
- Deployment instructions
- Testing checklists
- Troubleshooting guides
- Code location references

### Code Comments

All new code includes:
- **Justification**: Why the code exists
- **Error Handling**: How errors are handled
- **Dependencies**: What the code depends on

---

## Version History

- **v3.2.19** (Deployed): Phase 1 features (SUPERGROUP, Custom Tooltip/Link)
- **v3.3.0** (Planned): Phase 2 features (SLI)
- **v3.3.1** (Planned): Phase 3 features (SLO)
- **v3.3.2** (Planned): Phase 4 features (SLA)
- **v3.4.0** (Planned): Phase 5 features (KPI)
- **v3.5.0** (Planned): Phase 6 features (OKR)
- **v3.5.1** (Planned): Phase 7 features (SLI Email Alerts)
- **v3.5.2** (Planned): Phase 8 features (SLO Email Alerts)
- **v3.5.3** (Planned): Phase 9 features (SLA Email Alerts)
- **v3.6.0** (Planned): Phase 10 features (KPI Email Alerts)
- **v3.6.1** (Planned): Phase 11 features (OKR Email Alerts)
- **v3.6.2** (Planned): Phase 12 features (Email Templates)
- **v3.6.3** (Planned): Phase 13 features (Alert History Tracking)

---

**Last Updated**: 2025-01-15  
**Status**: ✅ Phase 1 Deployed in Production  
**Next Steps**: 
1. Deploy Phase 2 (SLI), then Phase 3 (SLO), then Phase 4 (SLA), followed by Phase 5 (KPI) and Phase 6 (OKR)
2. After metric features are deployed, deploy email alert phases:
   - Phase 7 (SLI Alerts) after Phase 2
   - Phase 8 (SLO Alerts) after Phase 3
   - Phase 9 (SLA Alerts) after Phase 4
   - Phase 10 (KPI Alerts) after Phase 5
   - Phase 11 (OKR Alerts) after Phase 6
3. Deploy supporting features:
   - Phase 12 (Email Templates) - can be deployed with Phase 7-11 or separately
   - Phase 13 (Alert History Tracking) - can be deployed with Phase 7-11 or separately

---

## Field Naming Convention

**Custom Fields** (consistent across all files):
- `customtooltip_message` - Custom tooltip message (no underscores, lowercase)
- `custommonitor_link` - Custom monitor link (no underscores, lowercase)

**All database fields, code variables, and UI bindings use these exact names for consistency.**
