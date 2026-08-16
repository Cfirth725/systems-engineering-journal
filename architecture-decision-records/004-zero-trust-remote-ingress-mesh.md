# Architecture Decision Record (ADR): Zero-Trust Remote Ingress via Overlay Mesh

* **Status:** Accepted
* **Date:** 2026-08-15
* **Decision:** Deploy a Tailscale (WireGuard) point-to-point mesh network for remote management, telemetry, and workspace ingress.
* **Context:** Securing remote administrative and service access to virtualized home lab infrastructure without opening inbound NAT ports.

---

### 1. Decision Overview

Managing virtualized environments, Proxmox hypervisors, and storage arrays from remote networks traditionally relies on port forwarding, dynamic DNS, or perimeter-bound SSL VPN gateways. To eliminate external attack surfaces, protect against bot scanning, and bypass carrier-grade NAT constraints, we deployed a peer-to-peer encrypted overlay network.

* **Underlying Protocol:** WireGuard (Noise Protocol Framework)
* **Mesh Orchestration:** Tailscale
* **Ingress Scope:** Proxmox VE Management, TrueNAS WebUI, SSH Bastions, Internal Telemetry/APIs
* **Security Model:** Zero-Trust Network Architecture (ZTNA) with Node-Level ACLs
* **Status:** Accepted (Implementation In Progress)

---

### 2. Architectural Drivers

* **Inbound Attack Surface Elimination:** Exposing administrative interfaces (SSH, Web GUIs) directly to the public internet creates continuous exposure to brute-force attempts, automated scanning, and unauthenticated daemon vulnerabilities.
* **CGNAT & Dynamic IP Traversal:** Residential internet connections frequently operate behind Carrier-Grade NAT (CGNAT) or dynamic public IP assignments, making traditional static site-to-site tunnels fragile and high-maintenance.
* **Identity-Centric Access & Granular ACLs:** Access must bind to strong identity providers (IdP) enforcing Multi-Factor Authentication (MFA), role-based node tagging, and granular least-privilege ACL rules rather than blanket network-level trust.
* **Low Latency & High Throughput:** Remote management and data syncing require a lightweight protocol with minimal CPU overhead and direct peer-to-peer routing to avoid bandwidth bottlenecks.

---

### 3. Evaluated Alternatives

#### **Alternative A: Direct Port Forwarding & Dynamic DNS**
* *Pros:* Simple configuration; requires no third-party agent or management layer.
* *Cons:* Exposes critical management interfaces directly to the internet; vulnerable to zero-day daemon exploits; requires brittle dynamic DNS update daemons.
* *Verdict:* **Rejected (Unacceptable Security Risk)**.

#### **Alternative B: Self-Hosted OpenVPN / Traditional Gateway**
* *Pros:* Full infrastructure ownership; mature ecosystem.
* *Cons:* Requires open inbound firewall ports; hub-and-spoke topology routes all traffic through a single bottleneck; higher CPU overhead and handshake latency compared to modern crypto primitives.
* *Verdict:* Rejected in favor of point-to-point WireGuard mesh architecture.

#### **Selected Protocol: Tailscale (WireGuard Overlay Mesh)**
* *Pros:* Zero inbound firewall ports required; reliable NAT traversal via STUN with DERP relay fallback; end-to-end ChaCha20-Poly1305 encryption; seamless multi-device key management and automatic rotation.
* *Trade-offs:* Introduces an external control plane dependency for network coordination and key distribution (data plane remains strictly end-to-end encrypted).
* *Verdict:* **Accepted**

---

### 4. Security & Operational Hardening

* **Default-Deny Perimeter:** Edge firewall maintains a strict `DROP ALL` inbound policy on WAN interfaces, completely eliminating listening external ports.
* **Isolated Data Plane:** WireGuard private keys are generated locally and never leave the node. The coordination plane handles node discovery and public key exchange without access to unencrypted payload traffic.
* **Least-Privilege Tailscale ACLs:** Access rules enforce role-based tags (e.g., `tag:admin`, `tag:compute`, `tag:storage`). Workstations can reach hypervisor management ports, while unprivileged application containers are isolated from infrastructure control planes.
* **Subnet Router Controls:** Where direct agent installation is impractical, dedicated subnet routers expose isolated internal ranges governed by strict destination IP and port restrictions.

---

### 5. Implementation Notes & Results

* **Frictionless Ingress:** Encrypted, authenticated access to Proxmox VE and internal service endpoints operates transparently across remote devices without manual port re-mapping.
* **Hardened Edge Profile:** External WAN port audits report 100% closed/filtered states across all standard management ports (22, 80, 443, 8006).
* **Deterministic Configuration:** Node onboarding is automated via short-lived auth keys and infrastructure tags, removing the need to manage manual per-client certificate bundles.