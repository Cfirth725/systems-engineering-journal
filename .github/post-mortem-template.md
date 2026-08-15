# Incident Post-Mortem & Remediation: [Brief Incident Title]

* **Status:** [ Investigating | Mitigated | Resolved | Closed ]
* **Date:** YYYY-MM-DD
* **Issue:** [One-sentence summary of the failure mode or operational interruption]
* **Resolution:** [One-sentence summary of how the system was brought back to healthy operation]
* **Severity / Impact Tier:** [ Tier 1: System Outage | Tier 2: Degraded Performance | Tier 3: Internal / Maintenance Blocker ]

---

### 1. Incident Overview

[Narrative description of what failed, how the failure was detected, and what workloads or environments were affected.]

* **Target Hardware / Environment:** [e.g., Proxmox Compute Node, TrueNAS Pool, Guest VM]
* **Component in Scope:** [e.g., Memory Controller, ZFS Pool, Network Interface, Container Engine]
* **Trigger Event:** [e.g., Initial hardware bring-up, OS kernel update, network storage migration]
* **Current Status:** [Resolved - Normal operational state restored]

---

### 2. Diagnostic Steps & Systematic Isolation

#### Phase 1: Initial Detection & Triage
* [First symptom observed, diagnostic LED, error log, or system alert]
* [Initial actions taken to stabilize the environment or prevent data corruption]

#### Phase 2: Hardware / Software Isolation
* [Systematic isolation steps taken to eliminate confounding variables]
* [Low-level diagnostics run: out-of-band firmware tools, single-channel isolation, kernel dmesg checks, network probes]

#### Phase 3: Failure State Verification
* [The precise test or command that confirmed the failure mechanism]
* [Observed vs. Expected behavior analysis]

---

### 3. Root Cause Analysis (RCA)

* **Primary Failure Mechanism:** [The foundational mechanical, firmware, configuration, or software defect]
* **Contributing Environmental Factors:** [Secondary issues like strict sub-timing parameters, permission boundaries, or MTU mismatches]
* **5 Whys Summary:**
  1. *Why did the service halt?* → [Direct symptom]
  2. *Why did that symptom occur?* → [Immediate cause]
  3. *Why did the immediate cause trigger?* → [Systemic factor]
  4. *Why was that factor present?* → [Process or configuration gap]
  5. *Why was that unmitigated?* → [Root cause]

---

### 4. Resolution & Remediation Actions

* **Immediate Fix (Hotfix):** [Action executed to restore functionality immediately]
* **Permanent Remediation:** [Architectural, hardware, or configuration changes implemented to prevent recurrence]

| Action Item | Type | Owner / Scope | Status |
| :--- | :--- | :--- | :--- |
| [Action 1: Replace non-QVL RAM with validated kit] | Hardware / Spec | [Compute Node] | Completed |
| [Action 2: Update firmware baseline in provisioning docs] | Documentation | [Runbook] | Completed |
| [Action 3: Add automated disk/temperature alerts] | Observability | [Monitoring] | Planned |

---

### 5. Key Engineering Insights & Lessons Learned

* **[Insight 1 - Diagnostic Rigor]:** [What the triage process proved regarding debugging methodology]
* **[Insight 2 - Architecture / Hardware Validation]:** [Lessons on component validation, vendor compatibility, or failure isolation]
* **[Insight 3 - Operational Hardening]:** [How this incident improved overall platform resilience]