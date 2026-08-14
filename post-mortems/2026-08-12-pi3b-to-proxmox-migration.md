# Architecture Migration & Environment Post-Mortem: Pi to Proxmox

* **Status:** Resolved
* **Date:** 2026-08-12
* **Issue:** Technical Debt & Compute Bottlenecks on Legacy ARM64 Edge Node
* **Resolution:** Live Network Data Migration & Workload Re-architecture to x86_64 Virtualized Workspace

---

### 1. Incident & Migration Overview

As backend services, Docker environments, and local Go development expanded beyond simple edge workloads, the existing single-node Raspberry Pi platform reached its practical hardware ceiling. To prevent resource starvation, reduce compile times, and eliminate physical storage limitations, the entire development workspace was migrated to an x86_64 virtual machine hosted on a high-performance Proxmox VE hypervisor node.

* **Source Platform:** Raspberry Pi 3B (ARM64 / Headless Bare-Metal)
* **Target Hypervisor:** Proxmox VE running on Mini-ITX Node (Intel Core Ultra 5 250K / 32GB DDR5)
* **Target Guest Environment:** Ubuntu Server 24.04 LTS VM (`workspace` / x86_64 / 64GB LVM)
* **Status:** Resolved & Operational. Source code, repositories, and toolchains fully transitioned to x86_64.

---

### 2. Migration Execution & Systematic Implementation

#### Phase 1: Target Provisioning & OS Hardening
* Provisioned an Ubuntu Server 24.04 LTS guest on Proxmox VE and extended the LVM volume group (`/dev/mapper/ubuntu--vg-ubuntu--lv`) to allocate 64 GB of virtual disk space.
* Disabled SSH password authentication in favor of strict `ed25519` public key pairs.
* Configured passwordless sudo privileges for the primary user and appended user accounts to the `docker` security group to enforce non-root container operations.

#### Phase 2: Non-Destructive Network Transfer (`rsync`)
* Initiated a live over-the-network pull from the guest workspace targeting the active Raspberry Pi filesystem:
```bash
  rsync -avP PI_USER@PI_IP:~/projects/ ~/projects/
```
* **Exit Code 23 Evaluation:** Isolated permission blocks to root-owned runtime directories and transient container locks (e.g., Portainer data stores).
* Used dry-run error diagnostics (`rsync -avPn ...`) to verify that all primary Git repositories and project source files transferred with 100% data integrity.

#### Phase 3: Cross-Architecture Remediation (`ARM64 → x86_64`)
* Identified and removed legacy ARM binaries across all project paths to eliminate `Exec format error` exceptions under x86_64 execution.
* Deliberately discarded ARM-based Docker container layers and state; re-built application images 
    ```
    docker compose up -d --build
    ```
* Installed the official x86_64 Go toolchain (`linux/amd64`) and verified clean native compilation across all backend services.

#### Phase 4: Remote SCM & Environment Integration
* Configured global Git user identity (`user.name`, `user.email`) and set default branch initialization (`init.defaultBranch main`).
* Generated a dedicated VM-level `ed25519` keypair and established SSH trust with GitHub (`ssh -T git@github.com`).
* Connected VS Code via Remote-SSH to facilitate headless, remote development directly inside `~/projects`.

### 3. Root Cause Analysis (RCA) & Architectural Shifts
* **Instruction Set Incompatibility:** Binary assets compiled for ARM64 (`aarch64`) cannot execute on x86_64 (`amd64`) architectures without emulation overhead. 
	* _Resolution:_ Clean separation of source code from compiled artifacts. All binaries and container layers were purged and recompiled natively on the host.
* **State vs. Code Separation:** Root-owned container volumes created permission collisions during non-root network transfers.
	*  _Resolution:_ Pruned host-bound container state (`rm -rf ~/projects/portainer`). Adopted a decoupled deployment model using lightweight, remote agent containers managed via the central Proxmox cluster.
* **Storage Fragmentation:** Default OS installation templates allocated restricted sub-volumes despite larger virtual disk provisioning.
	* _Resolution:_ Executed logical volume management (`lvextend` / `resize2fs`) prior to migration to ensure unconstrained workspace growth.

### 4. Key Takeaways & Engineering Insights
* **Stateless Workspace Migration:** Treating runtime state as disposable and source code as persistent allowed seamless data migration across differing hardware architectures without data loss.
* **Least-Privilege Security Posture:** Enforcing non-root Docker group execution, key-only SSH access, and abstracting network topology metrics ensured the VM was hardened from initial boot.
* **Scalable Infrastructure Pipeline:** Transitioning from edge hardware to virtualized hypervisor compute reduced Go compilation times, eliminated RAM constraints, and established a foundation for future containerization and cluster lab expansions.