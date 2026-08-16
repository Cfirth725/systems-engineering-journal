# Infrastructure Runbook: Docker Host & Container Lifecycle Triage

* **Status:** Active / Operational
* **Last Verified:** 2026-08-16
* **Target Environment:** Proxmox LXC (`apps` / Debian 12) & Proxmox Ubuntu VM (`workspace`)
* **Service Scope:** Docker Engine, Compose Deployments, Container Health, Storage Reclamation
* **Classification:** Internal Operations & Incident Response

---

## 1. System Overview & Architecture

This runbook provides deterministic troubleshooting and lifecycle management procedures for Docker container hosts running backend microservices, persistence layers (SQLite/PostgreSQL), and support utilities. It establishes structured procedures for resolving restart loops, diagnosing out-of-memory events, resolving port and network collisions, and safely pruning stale container layers.

* **Container Runtime:** Docker Engine (CE) with Docker Compose v2
* **Storage Driver:** `overlay2` backed by LVM / ext4
* **Security Model:** Non-root runtime execution (`docker` group, dedicated service UIDs)
* **Upstream Persistence:** NFSv4 mounted datasets & local persistent Docker volumes

---

## 2. Prerequisites & Access Requirements

* **Authentication:** SSH key access to the target host.
* **User Privileges:** Standard user with membership in the `docker` security group; `sudo` privileges for Docker daemon-level restarts.
* **Working Directory Context:** Application repository roots located under `~/projects/<service_name>/` or `~/apps/<service_name>/`.

---

## 3. Routine Health Checks & Verification

### Check 1: Active Container Status & Port Bindings
```bash
# List all containers with state, status duration, and bound host ports
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Image}}\t{{.Ports}}"
````

### Check 2: Real-Time Resource Consumption

```bash
# Snapshot CPU, memory usage, limits, and network I/O per container
docker stats --no-stream
```

### Check 3: Host Storage Footprint

```bash
# Inspect disk allocation across images, containers, local volumes, and build cache
docker system df -v
```

### Check 4: Docker Daemon Service Health

```bash
# Verify systemd service status and daemon runtime logs
sudo systemctl status docker --no-pager
```

## 4. Standard Operational Procedures

### Procedure A: Application Rebuild & Deployment (Docker Compose)

1. **Navigate to the target project directory:**
    ```bash
    cd ~/apps/<service_name>
    ```
2. **Option 1: In-Place Rolling Update (Minimal Downtime):**
    ```bash
    docker compose build --pull
    docker compose up -d --remove-orphans
    ```
3. **Option 2: Clean Rebuild & State Reset:**
    ```bash
    docker compose down
    docker compose build --no-cache
    docker compose up -d
    ```
4. **Verify Startup Stream:**
    ```bash
    docker compose logs -f --tail=50
    ```

### Procedure B: Storage Reclamation & Pruning

Execute these commands sequentially to reclaim disk space without dropping stateful data.
1. **Remove stopped containers, unused networks, and untagged dangling images:**
    ```bash
    docker system prune -f
    ```
2. **Purge multi-stage build cache:**
    ```bash
    docker builder prune -f --keep-storage=5GB
    ```
3. **Prune Dangling Volumes (Run ONLY after verifying unattached volumes are safe to destroy):**
    ```bash
    docker volume prune -f
    ```

## 5. Incident Triage & Emergency Response

### Scenario 1: Container in `CrashLoopBackOff` or Restart Loop

1. **Inspect Exit Code and Explicit Error States:**
    ```bash
    docker inspect <container_name> --format 'ExitCode: {{.State.ExitCode}} | OOMKilled: {{.State.OOMKilled}} | Error: {{.State.Error}}'
    ```
    - _Common Exit Codes:_
        - `137`: Process killed by SIGKILL (OOM or force-stopped).
        - `1`: Application crash (unhandled exception, database connection error, missing config).
        - `126 / 127`: Permission denied on entrypoint or missing binary/shared library in image.
2. **Inspect Timestamped Crash Logs:**
    ```bash
    docker logs --timestamps --tail=100 <container_name>
    ```
3. **Verify Host Mount & Volume Permissions:**
    Ensure the container's execution UID/GID possesses correct read/write permissions on the host path:
    ```bash
    ls -ld /path/to/mounted/volume
    ```

### Scenario 2: Host Out-of-Memory (OOM) Invocations

1. **Search Kernel Ring Buffer for OOM Invocations:**
    ```bash
    sudo dmesg -T | grep -i -E "oom[-_]killer|killed process"
    ```
2. **Inspect Container Memory Limits:**
    Apply explicit hard and soft limits in `docker-compose.yml` to prevent runaway memory exhaustion:
    ```YAML
    services:
      app:
        deploy:
          resources:
            limits:
              memory: 512M
            reservations:
              memory: 128M
    ```
3. **Recreate Stack with Applied Constraints:**
    ```bash
    docker compose up -d --force-recreate
    ```

### Scenario 3: Port and Network Collisions

1. **Port Binding Collisions (`address already in use`):**
    ```bash
    # Identify the process/PID holding the target host port
    sudo ss -tulpn | grep :<port_number>
    ```
    - If a stale container holds the port: `docker stop <container_id>`.
    - If a host service conflicts: update the left-hand host port binding in `docker-compose.yml` (e.g., `"8081:8080"`).
2. **Bridge Subnet Allocation Exhaustion (`could not find an available, non-overlapping IPv4 pool`):**
    ```bash
    # Remove stale, orphaned Compose bridge networks
    docker network prune -f
    ```

## 6. Escalation & Post-Incident Actions

- **Endpoint Verification:** Verify HTTP endpoints return expected `200 OK` responses and health endpoints pass.
- **Volume Persistence Check:** Confirm stateful assets (SQLite database files, uploaded media) remain intact across container recreate cycles.
- **Post-Mortem:** If the outage was triggered by volume permission issues, memory exhaustion, or image layer regressions, log the incident using `.github/post-mortem-template.md`.