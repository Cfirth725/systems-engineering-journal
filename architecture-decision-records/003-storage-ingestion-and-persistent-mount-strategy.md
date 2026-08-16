# Architecture Decision Record (ADR): Storage Ingestion & Persistent Mount Strategy

* **Status:** Accepted
* **Date:** 2026-08-15
* **Decision:** Adopt NFSv4 with POSIX permission mapping for Linux compute guests and container volumes, combined with local NVMe tiering for active transactional state.
* **Context:** Selecting a performant, resilient storage and network mount architecture to interface TrueNAS ZFS datasets with Proxmox VE compute guests and Docker container runtimes.

---

### 1. Decision Overview

With primary storage consolidated onto a dedicated TrueNAS array (ADR-001) and tiered across flash and mechanical pools (ADR-002), a standardized network mounting protocol and data persistence strategy was required. 

Compute workloads running on Proxmox VE (Ubuntu Server guests and Docker runtimes) require persistent, concurrent access to centralized ZFS datasets without protocol overhead, lock collisions, or complex target management. Simultaneously, high-IOPS transactional state must be isolated to prevent networked filesystem locking issues.

* **Target Compute:** Proxmox VE Compute Guests & Docker Runtimes
* **Storage Provider:** TrueNAS SCALE ZFS Pools (Mechanical & Flash Tiers)
* **Selected Network Protocol:** Network File System version 4 (NFSv4)
* **Active Database Storage:** Local NVMe Host/LXC Storage
* **Access Boundary:** Dedicated, isolated storage VLAN with strict export rules
* **Status:** Accepted (Implementation In Progress)

---

### 2. Architectural Drivers

* **Native Linux Kernel I/O:** Containerized Go microservices and pipeline workers generate sustained file and metadata operations. The protocol must operate natively within the Linux VFS layer rather than passing through user-space translation daemons.
* **Consistent POSIX Permissions:** Containers run as dedicated non-root users (`1000:1000` or service-specific UIDs). The storage protocol must cleanly handle Unix permissions and ownership without cumbersome translation layers.
* **Concurrent Multi-Client Access:** Microservices, ingestion pipelines, and automated backup runners require simultaneous read/write access to shared directories without lock stalls or split-brain corruption.
* **Database I/O Isolation & Media Wear Reduction:** Transactional databases generate continuous random write-ahead log (WAL) activity. These workloads require sub-millisecond latency and must be isolated from mechanical HDD arrays to prevent unnecessary disk wear and I/O bottlenecks.

---

### 3. Evaluated Alternatives

#### **Alternative A: Server Message Block (SMB / CIFS)**
* *Pros:* Great out-of-the-box Windows client support; native integration with Active Directory/ACLs.
* *Cons:* Higher CPU overhead in Linux environments; complex UID/GID permission mapping for Docker volumes; poor handling of Unix symlinks and socket files.
* *Verdict:* Rejected for container workloads. Reserved strictly for cross-platform desktop and creative asset shares.

#### **Alternative B: iSCSI (Block Storage over IP)**
* *Pros:* True block-level access, near-zero protocol overhead, direct integration for hypervisor virtual disks.
* *Cons:* Lacks multi-initiator concurrent read/write capabilities without complex clustered filesystems (e.g., GFS2/OCFS2); enforces fixed LUN allocations rather than dynamic, shared ZFS dataset quotas.
* *Verdict:* Rejected for shared application state; retained only for raw hypervisor disk images.

#### **Selected Protocol: Network File System (NFSv4)**
* *Pros:* Native kernel integration, stateful locking, minimal compute overhead, dynamic ZFS dataset quota consumption, and seamless multi-service volume sharing.
* *Trade-offs:* Requires strict network segmentation, firewall rules, and explicit ID-mapping configuration to prevent `nobody:nogroup` permission mismatch.
* *Verdict:* **Accepted**

---

### 4. Storage Tiering & State Boundary Policy

To ensure high performance and prevent locking contention over networked filesystems, storage boundaries are defined by workload profile:

* **Active Transactional State (Hot Tier):** Embedded and stateful database engines (e.g., SQLite in WAL mode) reside directly on local, high-speed NVMe flash within the compute host/LXC filesystem. This guarantees microsecond-level write latencies and avoids network lock stalls.
* **Shared Persistent Volumes (NFS Tier):** Media libraries, static application assets, shared configuration templates, and pipeline staging directories mount directly to TrueNAS datasets over NFSv4.
* **Cold Backup Retention:** Automated snapshot routines perform hot database backups (using native online snapshot APIs without requiring container downtime). Compressed archives are written over NFS to the mechanical TrueNAS storage pool, governed by a rolling retention policy (e.g., retaining the latest 5 weekly snapshots).

---

### 5. Security & Operational Hardening

* **Subnet Isolation:** TrueNAS exports are strictly restricted to the internal compute/storage subnet. All traffic across untrusted VLANs is dropped at the firewall.
* **Root Squash Policy:** Enabled `root_squash` on all standard exports so that a compromised container cannot execute privileged root operations against the raw ZFS filesystem.
* **Mount Resilience & Parameters:** Mounts are managed via systemd automount / `/etc/fstab` using `nfs4,noatime,hard,timeo=150,retrans=2,rsize=1048576,wsize=1048576`. The `hard` mount policy ensures write operations block safely during brief network interruptions rather than dropping silently.

---

### 6. Implementation Notes & Results

* **Native Docker Integration:** Docker Compose services bind persistent shared volumes directly using native NFS declarations (`type: nfs`), eliminating the need for intermediary storage sync containers.
* **Clean Permissions:** Standardizing on numeric UID/GID mappings eliminated file permission conflicts across Go compilation jobs and data staging tasks.
* **ZFS Feature Alignment:** TrueNAS automated snapshot lifecycle rules and retention schedules execute transparently at the dataset level without impacting mounted client sessions.