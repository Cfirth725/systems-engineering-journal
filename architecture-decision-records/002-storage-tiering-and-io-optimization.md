# Architecture Decision Record (ADR): Tiered Storage, Self-Hosted Media, and Drive I/O Optimization

* **Status:** Accepted
* **Date:** 2026-07-30
* **Decision:** Deployment of Immich, Jellyfin, and Syncthing with NVMe/HDD Storage Tiering & Dataset Compression
* **Context:** Solving Mobile Storage Bottlenecks while Protecting Mechanical HDD Lifespans from Random I/O Wear

---

### 1. Decision & Incident Overview

Mobile device storage limits and recurring vendor cloud subscription ceilings prompted the self-hosting of core media and document synchronization services on the UGREEN NAS. However, running continuous sync utilities (Syncthing) and database-heavy photo engines (Immich) directly on mechanical hard drives (HDDs) presents performance degradation and physical drive wear due to constant random read/write operations. 

To resolve this, a **tiered storage architecture** was implemented, decoupling active sync operations onto NVMe solid-state storage while reserving HDD arrays for long-term sequential retention.

* **Primary Storage Engine:** UGREEN NAS running TrueNAS / Docker Runtimes
* **Hot Tier (High IOPS):** NVMe SSD (Syncthing Active Ingestion, Immich Databases)
* **Cold Tier (High Capacity):** Mechanical HDD Pool (Jellyfin Media, Immich Photo Archives, Long-Term Backups)
* **Status:** Implemented & Operational.

---

### 2. Service Architecture & Stack Selection

* **Immich (Photos & Mobile Offload):** Replaces restrictive cloud photo services. Ingests raw phone photos over local network connections, automatically freeing up physical mobile device storage.
* **Jellyfin (Bulk Media Streaming):** Centralized media server managing uncompressed video/audio streaming across household devices.
* **Syncthing (Document & Repository Sync):** Real-time, peer-to-peer file synchronization across workstations, devices, and creative repositories.

---

### 3. I/O Engineering & Storage Tiering Strategy

#### **Problem: Random I/O Spindle Abuse on HDDs**
Syncthing continuously monitors directory trees, generating high volumes of small, random read/write metadata updates. Directing this activity to mechanical HDDs prevents drives from entering low-power states, elevates operating temperatures, and accelerates mechanical actuator arm fatigue.

#### **Solution: NVMe Staging & Flash Tiering**
1. **NVMe Hot Storage:** Syncthing sync targets, active project files, and application state databases (Immich PostgreSQL) are hosted exclusively on high-speed NVMe storage.
2. **HDD Cold Storage:** Mechanical drives are isolated for bulk, high-capacity sequential write workloads:
   * Periodic long-term archival backups.
   * Large static media stores for Jellyfin.
   * Permanent raw photo stores managed by Immich.

---

### 4. Dataset Compression & Filesystem Tuning

To optimize both CPU throughput and disk space utilization, custom ZFS dataset compression policies were applied based on underlying file entropy:

| Dataset / Service | Data Type | Compression Policy | Rationale |
| :--- | :--- | :--- | :--- |
| **`tank/media` (Jellyfin)** | Video (MP4/MKV), Audio | `lz4` / `off` | Video/Audio are pre-compressed; heavy compression wastes CPU cycles for 0% gain. |
| **`tank/photos` (Immich)** | JPEGs, RAW Photos | `lz4` | Minimal CPU overhead; skips uncompressible binary media automatically. |
| **`flash/documents` (Syncthing)** | Text, Code, PDFs, Configs | `zstd` | High compressibility; significantly reduces disk footprint for text-heavy assets with low CPU impact. |

---

### 5. Key Engineering Insights

* **Hardware Longevity:** Protecting mechanical HDDs from random read/write thrashing significantly extends drive lifespan and reduces ambient noise and thermal output.
* **Predictable Performance:** Offloading database indexes and continuous sync state to NVMe ensures near-zero latency responsiveness for end-user applications.
* **Storage Independence:** Successfully reclaimed mobile hardware storage without relying on proprietary, subscription-based cloud ecosystems.