# 🛠️ Systems Engineering Journal

Welcome to my systems engineering and infrastructure journal. This repository tracks architectural decisions, home lab engineering evolution, operational runbooks, and formal incident post-mortems for hardware bring-up, virtualization, networking, and storage systems.

The documentation here reflects a production-grade approach to systems management: prioritizing root-cause analysis (RCA), least-privilege security, stateless compute migrations, and systematic hardware isolation.

---

## 📐 Architecture Decision Records (ADRs)

| ID | Title | Date | Focus Area | Status |
| :--- | :--- | :--- | :--- | :--- |
| **ADR-001** | [Decoupled TrueNAS Storage Array](./architecture-decision-records/001-decoupled-truenas-storage-array.md) | 2026-07-25 | Storage / State Separation | Accepted |
| **ADR-002** | [Tiered Storage, ZFS Tuning, & Drive I/O Optimization](./architecture-decision-records/002-storage-tiering-and-io-optimization.md) | 2026-07-30 | Storage / Filesystem Tuning | Accepted |
| **ADR-003** | [Storage Ingestion & Persistent Mount Strategy](./architecture-decision-records/003-storage-ingestion-and-persistent-mount-strategy.md) | 2026-08-15 | Storage / NFSv4 Networking | Accepted |
| **ADR-004** | [Zero-Trust Remote Ingress via Overlay Mesh](./architecture-decision-records/004-zero-trust-remote-ingress-mesh.md) | 2026-08-15 | Networking / ZTNA Ingress | Accepted |

---

## 📋 Infrastructure & Operational Runbooks

| Doc ID | Title | Date | Target Systems | Status |
| :--- | :--- | :--- | :--- | :--- |
| **RBK-001** | [Observability & Telemetry Blueprint](./runbooks/001-observability-blueprint.md) | 2026-08-16 | Proxmox VE / TrueNAS / Docker | Proposed |

---

## 🚨 Incident Post-Mortems & Migrations

| Date | Incident / Event | Impact Area | Status |
| :--- | :--- | :--- | :--- |
| **2026-08-04** | [POST Failure on Z890 Mini-ITX Build](./post-mortems/2026-08-04-z890-dram-post-failure.md) | Hardware / Firmware | Resolved |
| **2026-08-12** | [Environment Migration: Pi 3B to Proxmox](./post-mortems/2026-08-12-pi3b-to-proxmox-migration.md) | Compute / Virtualization | Resolved |
| **2026-08-15** | [Enterprise Schedule State Overwrite via Automated Sync](./post-mortems/2026-08-15-schedule-state-overwrite-sync.md) | Application State / Data Recovery | Resolved |

---

## 🖥️ Core Infrastructure Stack

* **Compute Node (Hypervisor):** Proxmox VE running on a custom Mini-ITX build (Intel Core Ultra 5 250K / 32GB DDR5)
* **Storage Appliance:** Dedicated TrueNAS SCALE Node (ZFS Mirror Pool, NVMe Flash Cache / Ingestion Tier)
* **Remote Ingress & Overlay:** Tailscale (WireGuard P2P Mesh with Role-Based ACLs)
* **Primary Guest Workspace:** Ubuntu Server 24.04 LTS (`personal workspace` / 64GB LVM)
* **Core Runtime & Toolchains:** Go (`linux/amd64`), Docker & Docker Compose, Git, VS Code Remote-SSH

---

## 🔒 Security & Operational Standards

* **Zero Secrets Policy:** All internal IP addresses, network subnets, API tokens, and private keys are strictly redacted or abstracted in public documentation.
* **Perimeter Defense & Ingress:** Default-deny edge firewall posture; zero public inbound port forwarding, relying entirely on point-to-point WireGuard mesh routing.
* **Least Privilege:** Non-root execution enforced for container runtimes (`docker` group), strict POSIX/ACL filesystem boundaries on network shares, and key-based SSH (`ed25519`) with password authentication disabled.
* **Data Integrity & Resilience:** Decoupling persistent storage onto ZFS mirrors ensures that compute environments remain entirely disposable and rapidly rebuildable without risking application data loss.