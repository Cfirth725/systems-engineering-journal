# Architecture Decision Record (ADR): Decoupled TrueNAS Storage Array

* **Status:** Accepted
* **Date:** 2026-07-25
* **Decision:** Deployment of a Dedicated TrueNAS Network-Attached Storage (NAS) Mirror Array
* **Context:** Separating Persistent Data Storage from Virtualized Compute (Proxmox VE)

---

### 1. Decision Overview

As local workloads expanded to include containerized Go services, local database persistence, creative novel drafts, and multi-gigabyte game development assets (Godot), relying on host-local OS disks created a single point of failure. To achieve high availability, storage redundancy, and seamless multi-client access, storage was decoupled from compute and centralized onto a dedicated 4-bay TrueNAS network storage pool.

* **Target Compute Node:** Proxmox VE Virtualization Host (Personal Workspaces & Learning Playgrounds)
* **Storage Platform:** TrueNAS Core/SCALE (4-Bay Enclosure)
* **Storage Topology:** ZFS Mirror Pool (RAID-1 Redundancy)
* **Access Protocols:** SMB / NFS / Container Volume Mounts
* **Status:** Operational & Integrated.

---

### 2. Technical & Architectural Drivers

**Factor 1: Separation of Compute and State**
* Storing persistent data directly on virtual machine OS disks binds application state to the hypervisor host. Decoupled NAS storage allows Proxmox virtual machines or LXC containers to be destroyed, rebuilt, or migrated without risking underlying application data loss.

**Factor 2: Data Integrity & Fault Tolerance (ZFS)**
* Traditional filesystems are vulnerable to silent data corruption (bit rot). Deploying TrueNAS with a ZFS mirror pool guarantees drive-failure redundancy, background scrub validation, and atomic dataset snapshotting for instant point-in-time recovery.

**Factor 3: Multi-User & Cross-Platform Asset Syncing**
* The home lab ecosystem required high-throughput network file access across varied operating systems and client workflows:
  * **Development:** Persistent network volumes for containerized Go background services (e.g., `Parcel Herder & The Quest Log`).
  * **Creative & Game Dev:** High-speed SMB network shares enabling real-time asset syncing for Godot engine artwork and novel repository drafts.

---

### 3. Evaluated Alternatives

* **Alternative A: Local Host-Direct Storage (Proxmox Local ZFS)**
  * *Rejected:* Keeps storage tied to the physical compute node. Maintenance on the Proxmox host creates downtime for network file shares.
* **Alternative B: Consumer Cloud-Only Backups**
  * *Rejected:* High latency for large game assets, bandwidth limitations, lack of direct block/NFS mounting capabilities for local Docker containers, privacy concerns, and recurring costs associated with cloud-only backups.
* **Selected Architecture: Dedicated TrueNAS Appliance**
  * *Accepted:* Delivers dedicated hardware resources for storage I/O, independent uptime, native ZFS snapshotting, centralized network access, and complete local governance over data access policies.

---

### 4. Key Engineering Insights & Outcomes

* **Disaster Recovery Readiness:** Hypervisor compute nodes are now fully stateless; OS configurations can be restored in minutes while persistent data remains secure on the TrueNAS pool.
* **Unified Data Management:** Streamlined local network backups into a single target with automated ZFS snapshots protecting critical project repositories and media assets.
* **Scalable Service Foundation:** Established a production-like storage topology capable of serving persistent NFS/iSCSI volumes to future container orchestration clusters.
* **Security & Access Control:** Enforced strict SMB/NFS share permission boundaries (ACLs) to ensure containers and client devices operate under least-privilege access, prohibiting unauthorized root mounting across the storage network.