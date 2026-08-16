# System Troubleshooting & Incident Post-Mortem: Enterprise Schedule State Overwrite via Automated Sync

* **Status:** Resolved
* **Date:** 2026-08-15
* **Issue:** Active Weekly Production Schedule Disappearance via Automated Backend Service Overwrite
* **Resolution:** Point-in-time shift data was extracted from the transactional schedule audit logs and used to reconstruct and re-publish the daily roster for the missing week

---

### 1. Incident Overview

During routine administrative operations, a manager published the upcoming weekly schedule via the management portal. Immediately following this action, the active, published schedule for the current operational week vanished across all client-facing web and mobile applications, resulting in a blank schedule state.

* **Target System:** CrunchTime / Net-Chef Back-Office Infrastructure & TeamWorx Client Endpoints
* **Component in Scope:** Shift Scheduling Engine, Relational State Table, Automated Service Worker (`EMPLOYEEPORTAL USER`)
* **Trigger Event:** Publishing of future pay-period schedule with overlapping/unconstrained active date range context
* **Status:** Resolved - Shift state recovered via audit logs and restored to operational visibility

---

### 2. Diagnostic Steps & Systematic Isolation

#### Phase 1: Client Boundary & Presentation Layer Isolation
* Executed date-range header traversal and toggled views between individual assignments and the master roster to rule out local UI filtering resets.
* Performed client cache invalidation and application force-closure on mobile endpoints.
* Verified state persistence across independent desktop web sessions to eliminate local caching or mobile synchronization drift, confirming a centralized backend state failure.

#### Phase 2: Upstream / Downstream Data Integrity Validation
* Inspected POS back-of-house terminals and payroll timecard audit logs to confirm historical punches and prior-day labor records remained completely intact.
* **Analysis:** The persistence of POS timecard data confirmed that transactional labor history was unharmed, isolating the corruption strictly to the forward-looking scheduling entity tables.

#### Phase 3: Audit Log Telemetry & State Analysis
* Inspected the back-office modification history and identified a batch mutation event executed by the automated backend service principal (`EMPLOYEEPORTAL USER`) corresponding precisely to the timestamp of the outage.
* Queried the active schedule record in the Labor Scheduler grid; observed that the schedule contained only a single unassigned placeholder shift rather than the populated roster.
* **Analysis:** The single placeholder shift and service account timestamp confirmed an unconstrained batch synchronization had overwritten the active production record with a blank template shell.

---

### 3. Root Cause Analysis (RCA)

* **Primary Failure Mechanism:** The backend scheduling engine executed an unconstrained batch update job via the automated service account (`EMPLOYEEPORTAL USER`), replacing the active week's populated relational records with an empty template payload initialized during the next week's publish sequence.
* **Contributing Environmental Factors:** Lack of optimistic concurrency locking or date-range boundary isolation between active and draft schedules during administrative publishing workflows; absence of a direct one-click UI state rollback feature.

#### 5 Whys Summary
1. **Why was operational visibility lost?** → The active schedule displayed as blank across all mobile and web client interfaces.
2. **Why did that symptom occur?** → The active database record was overwritten with an unassigned blank scheduling template shell.
3. **Why did the immediate cause trigger?** → An automated backend service job (`EMPLOYEEPORTAL USER`) executed a batch sync across pay period boundaries without validating active target IDs.
4. **Why was that factor present?** → There were no safety checks in place to catch the mistake before the system replaced a fully staffed schedule with an empty one.
5. **Why was that unmitigated?** → The platform had no built-in undo or version restore feature, so the only way to recover the lost shifts was by digging through the audit logs by hand.

---

### 4. Key Takeaways & Engineering Insights

* **Event-Sourced Data Recovery:** When an application lacks a native rollback or undo button, immutable audit logs act as an effective event ledger to reconstruct deleted production state piece by piece.
* **Blast Radius Containment:** Checked independent data pipelines early to confirm that clock-in punches and payroll records were isolated from the presentation-layer wipe, protecting core historical data before starting recovery.
* **Cross-Functional Triage:** Diagnosed the failure mode and identified the responsible background job without direct back-office write permissions, then provided the manager with the extracted audit data so they could repopulate and republish the missing schedule without operational downtime.
