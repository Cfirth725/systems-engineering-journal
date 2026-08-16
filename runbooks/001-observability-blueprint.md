# Infrastructure Runbook: Observability & Telemetry Blueprint

* **Status:** Proposed (Design Draft)
* **Date:** 2026-08-16
* **Scope:** Host Telemetry, ZFS Pool Health, Container Metrics, and Log Aggregation across Proxmox VE & TrueNAS
* **Classification:** Internal Platform Runbook

---

## 1. System Architecture & Observability Objectives

This blueprint defines the telemetry pipeline planned across compute and storage infrastructure. The goal is to provide unified visibility into CPU/DRAM utilization, ZFS storage pool health, thermal envelopes, network throughput, and container runtime states, while keeping monitoring daemon overhead below 1% of total system resources.

```
+-------------------------------------------------------------------+
|                      Compute & Storage Hosts                      |
|                                                                   |
|  +--------------------------+     +--------------------------+    |
|  |   Proxmox VE (x86_64)    |     |      TrueNAS SCALE       |    |
|  | Node Exporter / cAdvisor |     |  Node Exporter / ZFS API |    |
|  +-------------+------------+     +-------------+------------+    |
+----------------|--------------------------------|-----------------+
                 | (Metrics Scrape / Pull)        | (Metrics Scrape / Pull)
                 v                                v
+-------------------------------------------------------------------+
|                    Telemetry & Ingestion Layer                    |
|                                                                   |
|  +-------------------------------------------------------------+  |
|  |       Prometheus Time-Series Database & Metric Scrapers     |  |
|  +------------------------------+------------------------------+  |
|                                 |                                 |
|                                 v                                 |
|  +-------------------------------------------------------------+  |
|  |        Grafana Dashboards & Alertmanager Alert Routing      |  |
|  +-------------------------------------------------------------+  |
+-------------------------------------------------------------------+
```

---

## 2. Monitored Metrics & Health Thresholds

### A. Compute & Memory (Proxmox VE)
* **CPU Core Load & Thermal Metrics:** Continuous monitoring of core package temperatures; warnings trigger if package temperatures exceed 80°C under sustained Go compilation workloads.
* **RAM Pressure & OOM Tracking:** Continuous validation of available host memory to prevent Linux OOM-killer invocations on container workloads and virtual machines.

### B. ZFS Storage Array Health (TrueNAS SCALE)
* **Pool Status & Error Counters:** Immediate critical alerts on any non-zero ZFS checksum or I/O errors (`zpool status`).
* **Disk Temperature Thresholds:** Active tracking of NVMe and mechanical HDD operating thermals (target: HDDs < 40°C, NVMe < 60°C).
* **Automated Scrub Schedules:** Bi-weekly background parity scans with automated notification upon completion.

### C. Container Runtimes (Docker)
* **Container Health & Restart Counts:** Tracking container uptime, unhandled exit codes, restart loops, and memory limit saturation for background services.

---

## 3. Routine Verification Procedures

### Check 1: Hypervisor Node Health & Thermals
Access the Proxmox host via SSH to verify local load and thermal baselines:
```bash
# Check system load averages and active core frequencies
uptime
lscpu | grep "MHz"

# Check active memory utilization
free -h

# Review thermal package status
sensors
```

### Check 2: ZFS Pool Integrity
Inspect the state of all configured mirror pools on the TrueNAS node:

```bash
# Check pool health and error counters
zpool status -x

# Review dataset capacity and compression ratios
zfs list -o name,used,avail,refer,compressratio,mountpoint
```

### Check 3: Container Engine Health Check
Verify that all running microservices and background agents are healthy:

```bash
# Check running container statuses and resource usage
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker stats --no-stream
```

---

## 4. Incident Triage Workflow
When a telemetry alert fires (e.g., degraded ZFS pool, thermal spike, or container restart loop):
1. **Acknowledge Alert:** Identify the source host and failing metric from the Alertmanager notification.
2. **Isolate Workload:** If CPU or thermals spike, identify runaway processes:
    ```bash
    top -b -n 1 -o %CPU | head -n 15
    ```
3. **Inspect Logs:**
    - Host / system services:
        ```bash
        sudo journalctl -xeu <service_name> --no-pager -n 50
        ```

    - Container services:
        ```bash
        docker logs --tail 100 <container_name>
        ```

4. **Remediate:** Apply targeted mitigation (e.g., restart service, clear stuck lock, replace failing drive).
5. **Document:** If service interruption occurred, open a new post-mortem file using `.github/post-mortem-template.md`.