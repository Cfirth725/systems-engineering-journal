# Infrastructure Runbook: ZFS Storage Pool Maintenance & Disk Degradation

- **Status:** Active / Operational
- **Last Verified:** 2026-08-17
- **Target Environment:** TrueNAS Storage Node / Proxmox VE Storage Subsystem
- **Service Scope:** ZFS Pool Integrity, Data Scrubbing, Disk Degradation Triage, Drive Replacement
- **Classification:** Internal Operations & Incident Response

## 1. System Overview & Architecture

This runbook defines operational procedures for monitoring, verifying, and maintaining ZFS storage pools across both high-IOPS NVMe flash tiers and high-capacity mechanical HDD mirror arrays. The primary objective is to detect silent data corruption, govern parity scrubbing schedules, and execute deterministic drive replacements without risking data loss.

- **Primary Storage Engine:** OpenZFS on Linux / TrueNAS SCALE
- **Storage Tiers:**
    - `flash`: NVMe SSD Mirror (Hot Ingestion, Application State, VM/LXC Disks, Databases)
    - `tank`: Mechanical HDD Mirror (Media Archives, Persistent Storage, Automated Snapshots)
- **Parity & Redundancy:** Two-Way Mirrors (RAID 1 equivalents per VDEV)
- **Upstream Dependencies:** None (Foundational Persistence Layer)

## 2. Prerequisites & Access Requirements
- **Authentication:** SSH key access via the local management subnet or secure mesh network.
- **Privileges:** `root` or a user with full `sudo` privileges to execute `zpool`, `zfs`, and `smartctl` binaries.
- **Device Identification:** All ZFS operations must reference persistent identifiers in `/dev/disk/by-id/` rather than volatile `/dev/sdX` or `/dev/nvmeXn1` device nodes.
- **Physical Access:** Required for hot-swap chassis access or hardware servicing during drive replacements.

## 3. Routine Health Checks & Verification

### Check 1: Overall Pool Health & Error Counters
```bash
# Check status across all pools (alerts on DEGRADED, FAULTED, or non-zero READ/WRITE/CKSUM errors)
zpool status -x
```
_Expected Output:_ `all pools are healthy`

### Check 2: Detailed Pool Inspection & VDEV Topology
```bash
# Display full topology, scrub progression, resilver rate, and error counters
zpool status -v
```

### Check 3: Dataset Capacity & Compression Metrics
```bash
# Verify available space, quotas, and operational compression ratios across datasets
zfs list -o name,used,avail,refer,compressratio,mountpoint
```

### Check 4: Disk Health & SMART Telemetry
```bash
# SATA/SAS HDDs and SSDs
sudo smartctl -A /dev/disk/by-id/<drive_id> | grep -E "Temperature_Celsius|Reallocated_Sector|Current_Pending_Sector|Offline_Uncorrectable|UDMA_CRC_Error_Count"

# NVMe Flash Devices
sudo smartctl -a /dev/disk/by-id/<nvme_id> | grep -E "Critical Warning|Temperature:|Percentage Used|Media and Data Integrity Errors|Available Spare"
```

## 4. Standard Operational Procedures

### Procedure A: Manual Parity Scrub Execution
ZFS scrubs systematically read all data blocks and compare them against cryptographic checksums to heal silent bit rot.
1. **Initiate background scrub on target pool:**
    ```bash
    zpool scrub <pool_name>
    ```
2. **Monitor scrub progression and estimated completion time:**
    ```bash
    zpool status <pool_name>
    ```
3. **Pause or stop scrub if I/O impact degrades high-priority services:**
    ```bash
    zpool scrub -p <pool_name>  # Pause scrub
    zpool scrub -s <pool_name>  # Stop/Cancel scrub
    ```

### Procedure B: Manual Dataset Snapshot Management
1. **Create an atomic snapshot prior to migrations or major updates:**
    ```bash
    zfs snapshot <pool_name>/<dataset>@manual-pre-maintenance-$(date +%Y%m%d)
    ```
2. **List existing snapshots sorted chronologically:**
    ```bash
    zfs list -t snapshot -o name,used,creation -s creation
    ```
3. **Prune obsolete or temporary snapshots:**
    ```bash
    zfs destroy <pool_name>/<dataset>@<snapshot_name>
    ```

## 5. Incident Triage & Emergency Response

### Scenario 1: Pool Reports Checksum (`CKSUM`) Errors
Checksum errors indicate corrupt data blocks intercepted and corrected by ZFS mirrors.

1. **Assess Error Distribution:** Check `zpool status -v` to determine if errors are isolated to a single disk (drive/cable fault) or spread across the pool (controller/memory issue).
2. **Inspect Physical Connectivity:** Verify SATA/SAS/NVMe data cables, backplane seating, and power connections.
3. **Clear Transient Error Counters (Post-Inspection):**
    ```bash
    zpool clear <pool_name>
    ```
4. **Execute Full Parity Scrub:**
    ```bash
    zpool scrub <pool_name>
    ```

### Scenario 2: Disk Degradation & Drive Replacement Procedure
Execute this procedure when a disk experiences mechanical failure, uncorrectable sector faults, or drops completely from the host bus.

1. **Identify the Degraded Device:**
    ```bash
    # Retrieve pool topology and persistent IDs
    zpool status -v <pool_name>
    
    # Confirm device model and physical serial number before pulling hardware
    sudo smartctl -i /dev/disk/by-id/<drive_id> | grep -E "Model Family|Device Model|Serial Number"
    ```
    > **Note:** If a drive has dropped off the bus entirely, ZFS will display a 64-bit numerical GUID (e.g., `1483920194857201948`) in place of the path. Note this GUID.
1. **Offline the Degraded Drive (if still responsive):**
    ```bash
    zpool offline <pool_name> /dev/disk/by-id/<failed_drive_id>
    ```
    _(Skip this step if the disk is already marked `FAULTED`, `UNAVAIL`, or represented only by a numeric GUID)._
2. **Physical Replacement:** 
    - Locate the physical drive bay using the confirmed serial number.
    - Hot-swap the failed drive with the replacement unit (must match or exceed the capacity of the original drive). If the chassis lacks hot-swap backplanes, perform a controlled host shutdown before swapping.
3. **Prepare the Replacement Drive:**
    ```bash
    # Locate the new persistent device identifier
    ls -la /dev/disk/by-id/
    
    # Wipe legacy partition tables, ZFS metadata, or foreign filesystem headers
    sudo wipefs -a /dev/disk/by-id/<new_drive_id>
    ```
4. **Initiate ZFS Replace & Resilver:**
    ```bash
    # Standard replacement (replacing existing visible ID)
    zpool replace <pool_name> /dev/disk/by-id/<failed_drive_id> /dev/disk/by-id/<new_drive_id>
    
    # GUID-based replacement (if the failed drive was dead/missing)
    zpool replace <pool_name> <failed_drive_guid> /dev/disk/by-id/<new_drive_id>
    ```
5. **Monitor Resilver Progression:**
    ```bash
    zpool status <pool_name>
    ```
    _Do not reboot the node, disconnect storage HBAs, or alter pool topology until resilvering reaches 100% and the pool state returns to `ONLINE`._
6. **Ensure Autoexpand is Enabled (for larger capacity replacement disks):**
    ```bash
    zpool set autoexpand=on <pool_name>
    ```

## 6. Escalation & Post-Incident Actions

- **Verification:** Confirm all pool VDEVs display `ONLINE` and `zpool status -x` returns `all pools are healthy`.
- **Telemetry Update:** Ensure disk serial numbers and SMART monitoring targets are updated in node exporter and dashboard telemetry.
- **Documentation:** Log any unpredicted drive failures or anomalous controller resets in the infrastructure post-mortem log.