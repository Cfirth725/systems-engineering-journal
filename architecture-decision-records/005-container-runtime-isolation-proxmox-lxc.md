# Architecture Decision Record (ADR): Container Runtime Isolation on Proxmox VE

- **Status:** Accepted
- **Date:** 2026-08-16
- **Decision:** Provision an Unprivileged Debian 12 LXC Container for Native Docker & Compose Workloads
- **Context:** Decoupling 24/7 Production Stacks and Microservices from Heavy Interactive Virtual Machines

---

### 1. Decision Overview

To support persistent, always-on backend microservices (Go REST APIs, telemetry daemons, background workers) and container management tooling without the heavy resource overhead of full virtual machines, I provisioned a dedicated container runtime host directly on the compute cluster. Instead of running background stacks inside my main interactive development VM or spinning up another full KVM guest, workloads are consolidated into an unprivileged Debian 12 LXC running Docker Engine.

- **Target Compute Node:** Proxmox VE Compute Host (`srv-compute-01`)
- **Execution Environment:** Debian 12 Unprivileged LXC (`nesting=1`, `keyctl=1`)
- **Container Runtime:** Docker Engine (CE) with Docker Compose v2 & Portainer CE
- **Storage Driver:** Native `overlay2`
- **Access Protocols:** SSH Public-Key Authentication (`ed25519`), TLS Web UI (Port 9443)

---

### 2. Technical & Architectural Drivers

**Factor 1: Decoupling Interactive Workspace from Always-On Workloads**
- Running persistent services inside the primary interactive development VM (`workspace`) tightly coupled application uptime to local dev sessions. Offloading production and staging stacks to a dedicated LXC means the dev workstation can be rebooted, reconfigured, or stressed during heavy compilation tasks without taking down running background services.

**Factor 2: Compute Density & Resource Efficiency**
- Full virtual machines require dedicated guest kernels, virtual memory tables, and virtual disk images—often idling at 512 MB to 1 GB of RAM before launching a single container. An unprivileged LXC shares the host Linux kernel via cgroups v2 and namespaces, idling at under 60 MB of RAM while providing near bare-metal I/O performance.

**Factor 3: Defense-in-Depth via User Namespaces (`userns`)**
- Running nested Docker inside an **unprivileged** container creates a strict security boundary: root inside the LXC (UID 0) maps to an unprivileged high UID (e.g., `100000`) on the Proxmox host. Even if an entrypoint exploit or container breakout occurs, the blast radius remains trapped within the sub-UID namespace, preventing privilege escalation to the physical hypervisor while keeping host AppArmor containment intact.

---

### 3. Evaluated Alternatives

- **Alternative A: Dedicated Full Virtual Machine (QEMU/KVM)**  
  - *Rejected:* Incurs hypervisor scheduling overhead, higher base RAM usage, and unnecessarily large backup images for standard OCI containers.

- **Alternative B: Running Production Stacks Inside Interactive Dev VM (`workspace`)**  
  - *Rejected:* Violates separation of concerns. Developer experiments, tool upgrades, and compilation spikes directly threaten service stability and create resource contention.

- **Selected Architecture: Unprivileged Proxmox LXC with Nesting**  
  - *Accepted:* Delivers near-instant boot times, minimal storage/RAM overhead, seamless integration with host ZFS/LVM storage, and strong isolation via Linux user namespaces.

---

### 4. Key Engineering Insights & Trade-Offs

- **Standardized Declarative Deployments:** All production applications are managed via version-controlled `docker-compose.yml` files and `.env` secret configurations rather than manual CLI commands.

- **Least-Privilege Non-Root Execution:** Root SSH login is disabled. Day-to-day deployments run under a dedicated service account with scoped `sudo` privileges and membership in the `docker` group.

- **Fast Disaster Recovery:** The lightweight LXC footprint allows for rapid, differential Proxmox backups with minimal storage overhead, while persistent application data stays mapped to dedicated storage volumes.

- **Observability & Management Foundation:** Deployed Portainer CE directly against the local Docker socket for real-time container health metrics, log streaming, and straightforward lifecycle management.

- **Operational Trade-Offs (Accepted):** Unprivileged LXCs cannot easily handle raw PCIe hardware passthrough or arbitrary custom kernel module loading compared to full KVM VMs. Because all target workloads are standard OCI application containers, this constraint has zero impact on operational goals.