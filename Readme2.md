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
   - **Description**: Add UI for creating and managing SLI alert configurations in the Alerts section
   - **Acceptance Criteria**:
     - "New Alert" button in the alerts page header
     - Modal form to create SLI alerts with monitor selection
     - Input field for threshold (0-100)
     - Multi-email input for recipients
     - Cooldown minutes input
     - Validation feedback
     - Saves alert configuration to selected monitor
   - **Files**: 
     - `src/routes/(manage)/manage/(app)/app/alerts/+page.svelte` (New Alert button)
     - `src/lib/components/manage/metricAlertForm.svelte` (SLI alert configuration form)

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
   - **Description**: Add UI for creating and managing SLO alert configurations in the Alerts section
   - **Acceptance Criteria**:
     - SLO alert configuration option in metric alert form
     - Input field for threshold (0-100)
     - Multi-email input for recipients
     - Cooldown minutes input
     - Validation feedback
     - Saves alert configuration to selected monitor
   - **Files**: 
     - `src/lib/components/manage/metricAlertForm.svelte` (SLO alert configuration form)

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
   - **Description**: Add UI for creating and managing SLA alert configurations in the Alerts section
   - **Acceptance Criteria**:
     - SLA alert configuration option in metric alert form
     - Multi-email input for recipients
     - Cooldown minutes input
     - No threshold input (violation is binary)
     - Validation feedback
     - Saves alert configuration to selected monitor
   - **Files**: 
     - `src/lib/components/manage/metricAlertForm.svelte` (SLA alert configuration form)

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
   - **Description**: Add UI for creating and managing KPI alert configurations in the Alerts section
   - **Acceptance Criteria**:
     - KPI alert configuration option in metric alert form
     - Dynamic list of KPI alerts (add/remove)
     - For each KPI alert: KPI name input, threshold input, comparison dropdown (above/below), recipients input
     - Cooldown minutes input
     - Validation feedback for all KPI alerts
     - Saves alert configuration to selected monitor
   - **Files**: 
     - `src/lib/components/manage/metricAlertForm.svelte` (KPI alert configuration form)

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
   - **Description**: Add UI for creating and managing OKR alert configurations in the Alerts section
   - **Acceptance Criteria**:
     - OKR alert configuration option in metric alert form
     - Checkboxes for status alerts (AT_RISK, OFF_TRACK)
     - Multi-email input for recipients
     - Cooldown minutes input
     - Validation feedback
     - Saves alert configuration to selected monitor
   - **Files**: 
     - `src/lib/components/manage/metricAlertForm.svelte` (OKR alert configuration form)

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

### ✅ Feature 16: Metric Alert Creation UI Enhancement

**Status**: ✅ **DEPLOYED IN PRODUCTION**

**Description**: Centralized UI in the Alerts section for creating and managing metric email alerts (SLI, SLO, SLA, KPI, OKR) instead of configuring them in the monitor form.

**Key Capabilities**:
- "New Alert" button in the Alerts section
- Unified modal form for all metric alert types
- Monitor selection dropdown (all active monitors)
- Alert type selection (SLI, SLO, SLA, KPI, OKR)
- Dynamic form fields based on alert type
- Comprehensive validation
- Saves alert configuration directly to the selected monitor

**EPIC: Metric Alert Creation UI Enhancement**

**Stories**:

1. **Story 16.1: New Alert Button in Alerts Section**
   - **Description**: Add "New Alert" button to the alerts page header
   - **Acceptance Criteria**:
     - "New Alert" button visible in alerts page header
     - Button opens modal form for metric alert creation
     - Button loads list of active monitors when clicked
     - Button is positioned next to documentation link
   - **Files**: `src/routes/(manage)/manage/(app)/app/alerts/+page.svelte` (NEW - Lines 11-45)
   - **Before**: No button to create alerts in alerts section
   - **After**: "New Alert" button opens metric alert creation form
   - **Justification**: Provides centralized location for alert management, separating alert configuration from monitor configuration

2. **Story 16.2: Metric Alert Form Component**
   - **Description**: Create reusable component for metric alert configuration
   - **Acceptance Criteria**:
     - Modal form component for creating metric alerts
     - Monitor selection dropdown (shows all active monitors)
     - Alert type selection (SLI, SLO, SLA, KPI, OKR)
     - Dynamic form fields based on selected alert type
     - Form validation with error messages
     - Save functionality that updates monitor's alert config
     - Close modal functionality
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (NEW)
   - **Before**: No centralized form for metric alert creation
   - **After**: Unified form component for all metric alert types
   - **Justification**: Centralizes alert creation logic and provides consistent UX for all metric alert types

3. **Story 16.3: SLI Alert Configuration in Form**
   - **Description**: Add SLI alert configuration fields to metric alert form
   - **Acceptance Criteria**:
     - Threshold input field (0-100, numeric)
     - Recipients input field (comma-separated emails)
     - Cooldown minutes input field
     - Validation for all fields
     - Saves to `sli_alert_config` field in selected monitor
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (SLI configuration section)
   - **Before**: SLI alerts could only be configured in monitor form
   - **After**: SLI alerts can be created from alerts section
   - **Justification**: Separates alert management from monitor configuration

4. **Story 16.4: SLO Alert Configuration in Form**
   - **Description**: Add SLO alert configuration fields to metric alert form
   - **Acceptance Criteria**:
     - Threshold input field (0-100, numeric)
     - Recipients input field (comma-separated emails)
     - Cooldown minutes input field
     - Validation for all fields
     - Saves to `slo_alert_config` field in selected monitor
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (SLO configuration section)
   - **Justification**: Consistent alert management interface

5. **Story 16.5: SLA Alert Configuration in Form**
   - **Description**: Add SLA alert configuration fields to metric alert form
   - **Acceptance Criteria**:
     - Recipients input field (comma-separated emails)
     - Cooldown minutes input field
     - No threshold field (violation is binary)
     - Validation for recipients
     - Saves to `sla_alert_config` field in selected monitor
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (SLA configuration section)
   - **Justification**: Consistent alert management interface

6. **Story 16.6: KPI Alert Configuration in Form**
   - **Description**: Add KPI alert configuration fields to metric alert form
   - **Acceptance Criteria**:
     - Dynamic list of KPI alerts (add/remove buttons)
     - For each KPI alert: KPI name input, threshold input, comparison dropdown (above/below), recipients input
     - Cooldown minutes input (shared across all KPI alerts)
     - Validation for each KPI alert
     - Saves to `kpi_alert_config` field in selected monitor
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (KPI configuration section)
   - **Justification**: Supports multiple KPI alerts per monitor with flexible configuration

7. **Story 16.7: OKR Alert Configuration in Form**
   - **Description**: Add OKR alert configuration fields to metric alert form
   - **Acceptance Criteria**:
     - Checkboxes for status alerts (AT_RISK, OFF_TRACK)
     - Recipients input field (comma-separated emails)
     - Cooldown minutes input field
     - Validation for recipients and status selection
     - Saves to `okr_alert_config` field in selected monitor
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (OKR configuration section)
   - **Justification**: Consistent alert management interface

8. **Story 16.8: Form Validation and Error Handling**
   - **Description**: Implement comprehensive validation and error handling in metric alert form
   - **Acceptance Criteria**:
     - Validates monitor selection (required)
     - Validates alert type selection (required)
     - Validates email addresses (format validation)
     - Validates threshold ranges (0-100 for SLI/SLO)
     - Validates required fields based on alert type
     - Displays clear error messages
     - Handles API errors gracefully
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (Validation logic)
   - **Justification**: Ensures data integrity and provides clear feedback to users

9. **Story 16.9: Monitor API Integration**
   - **Description**: Integrate with monitor API to fetch and update alert configurations
   - **Acceptance Criteria**:
     - Fetches list of active monitors for dropdown
     - Fetches monitor details when saving alert config
     - Updates monitor's alert configuration field
     - Handles API errors (monitor not found, update failed)
     - Provides success/error feedback
   - **Files**: `src/lib/components/manage/metricAlertForm.svelte` (API integration)
   - **Justification**: Enables seamless integration with existing monitor management system
