# 02 - Containers (Orchestration)

## 🎯 Purpose & Scope
This section documents containerisation strategies, environment isolates, and service orchestrations running on top of Linux hosts or within Proxmox LXC containers.

---

## 🛠️ Proxmox Containers (LXC) 
### Step 1: Template Storage & Image Selection
* **Storage Target:** Downloaded official Linux distribution templates directly via the local Proxmox storage engine (`local:vztmpl`).
* **Base Distro:** Go to container template and search lightweight **Debian / Ubuntu Minimal** LXC templates to keep footprint extremely small.

### Step 2: Container Provisioning Parameters
* **Container ID:** Assigned structured numbering (`CT 100`, `CT 101`, etc.) to keep host inventory grouped and organised. # Note: Proxmox naming convention are restrictive
* **Privilege Mode:** 
  * **Unprivileged (Default):** Used for standard services to ensure security isolation (root inside container is unprivileged on the host).
  * **Privileged (Optional/Isolated):** Reserved only for specific hardware passthrough or NFS/SMB storage mount requirements.
* **Resource Allocation:** Allocated minimal dynamic RAM (512MB–1GB) and thin-provisioned disk space.

### Step 3: Network & Gateway Attachment
* **Bridge:** Bound container interface directly to `vmbr0`. By default `vmbr0` is your WAN/NAT connection, can be changed based on your preference as long it doesn't derived from your gateway network and subnet.
* **IP Assignment:** Configured static LAN IP or assigned via router MAC reservation for consistent network accessibility based on network gateway and subnet.

---

## 📌 References & Documentation
* [Proxmox VE LXC Official Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_pct)
* [Proxmox VE Linux Container Wiki](https://pve.proxmox.com/wiki/Linux_Container)

### Step 4: Docker & Compose Engine Setup (Nested Containerisation)

* **Proxmox Feature Flags:** Enabled **Keyctl** and **Nesting** (`nesting=1`) in LXC Container Options to allow Docker engine execution inside an unprivileged LXC container.
* **Storage Driver:** Configured Docker to use the **overlay2** storage driver for optimal layer management and performance.
* **Stack Deployment:** Standardied container deployments using `docker-compose.yml` declarations managed via CLI (`docker compose up -d`).
* **Volume Persistence:** Mapped host-bound persistent volumes to prevent data loss across container recreations.

## 🛠️ Essential Docker & Compose Commands

### 📦 Stack Orchestration (Compose)
* **Start stack in background:** `docker compose up -d`
* **Stop and remove stack:** `docker compose down`
* **Stop stack without removing containers:** `docker compose stop`
* **Restart stack services:** `docker compose restart`
* **Rebuild/pull and force recreate containers:** `docker compose up -d --force-recreate`

### 🔍 Monitoring & Diagnostics
* **View running containers:** `docker ps`
* **View all containers (including stopped):** `docker ps -a`
* **View real-time logs for a stack:** `docker compose logs -f`
* **View logs for a specific container:** `docker logs -f <container_name>`
* **Monitor resource usage (CPU/RAM):** `docker stats`

### 💻 Container Interaction
* **Open interactive shell inside a running container:** 
  `docker exec -it <container_name> /bin/sh`  *(or `/bin/bash`)*
* **Check container health & environment config:** `docker inspect <container_name>`

### 🧹 System Cleanup (Pruning)
* **Remove unused stopped containers, networks, and dangling images:** `docker system prune`
* **Full deep clean (includes unused volumes):** `docker system prune -a --volumes`

---

## 🛡️ Featured Projects

### [Denfira-2 Infrastructure Stack](./denfira-2/README.md)
* **Architecture:** Zero-trust Tailscale sidecar container mesh governed by Headscale control plane.
* **Storage & Security:** Multi-tier NVMe OS drive paired with auto-unlocked BitLocker HDD mirrors (`H:` and `I:`).
* **Workloads:** Nextcloud (MariaDB), Vaultwarden (SQLite), Planka (PostgreSQL), and Nginx Proxy Manager (mkcert private PKI).
* **Disaster Recovery:** Crash-consistent `docker pause` nightly SQLite snapshots and automated hourly Robocopy replication tasks.

