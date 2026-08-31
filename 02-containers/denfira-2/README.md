# Denfira-2 Infrastructure Runbook: Sidecar Mesh, BitLocker & Disaster Recovery

## 🎯 Architecture Overview

* **OS Drive (`C:`):** NVMe SSD (BitLocker encrypted via TPM). Hosts Docker engine, Compose files, and low-latency database engines.
* **Primary Storage (`H:` - Miri-Da):** 1TB HDD (BitLocker encrypted + Auto-Unlock). Active payload storage for Nextcloud (`H:\NextcloudData`) and Vaultwarden nightly backups (`H:\VaultwardenBackup`).
* **Secondary Mirror (`I:` - Miri-Da-Backup):** 1TB HDD (BitLocker encrypted + Auto-Unlock). Asynchronous mirror target for hourly synchronization and retention.
* **Networking Pattern:** Tailscale Sidecar Container. All application containers (`npm`, `vaultwarden`, `planka`, `planka-db`, `nextcloud`, `nextcloud-db`) bind to `network_mode: "service:tailscale"`, routing through node IP `100.64.0.3` and communicating internally over `127.0.0.1`.

---

## 📊 Component & Storage Breakdown

| Component / Tier | Drive / Resource | Encryption / Security | Role & Stored Services |
| :--- | :--- | :--- | :--- |
| **System & Engine** | `C:` (NVMe SSD) | TPM BitLocker | Docker Engine, Compose files, and database engines (MariaDB, PostgreSQL, SQLite). |
| **Primary Data Storage** | `H:` (1TB HDD) | BitLocker + Auto-Unlock | Active payload storage: Nextcloud data (`H:\NextcloudData`) and Vaultwarden nightly backup target. |
| **Mirror / Secondary** | `I:` (1TB HDD) | BitLocker + Auto-Unlock | Asynchronous mirror target for automated hourly Nextcloud sync and secondary backup retention. |
| **Networking & Mesh** | `denfira-2` (Sidecar) | WireGuard + Headscale | Shared network namespace (`100.64.0.3`) isolating all container traffic to `127.0.0.1` with zero exposed gateway router ports. |
| **Ingress & TLS** | NPM (`:80`, `:443`, `:81`) | mkcert Root CA | Reverse proxy, SSL/TLS termination, and WebSocket routing for vault, cloud, and planka subdomains. |

---

## 🔒 Security Architecture & Benefits

1. **Zero WAN Attack Surface:** No open ports forwarded on the physical gateway router. Ingress is mediated entirely through WireGuard cryptographic handshakes managed via Headscale.
2. **Network Namespace Isolation (Sidecar Pattern):** By locking container communication into `network_mode: "service:tailscale"`, traffic between databases, backends, and the reverse proxy never crosses external physical interfaces—all database ports listen strictly on loopback (`127.0.0.1`).
3. **Hardware-Bound Cryptographic Protection At-Rest:**
   - The system partition (`C:`) is safeguarded against offline extraction via TPM-enforced BitLocker.
   - Secondary HDDs (`H:` and `I:`) use BitLocker Auto-Unlock tied to the host TPM, ensuring automated reboot recovery while remaining protected if physically detached.
4. **Crash-Consistent Database Snapshots:** Utilizing `docker pause` ensures SQLite write-ahead logging (WAL) memory is flushed to disk before Robocopy staging, preventing fragmented snapshot corruption during backup tasks.
5. **Private PKI Trust Architecture:** End-to-end SSL/TLS termination via a local `mkcert` root CA prevents internal domain lookups from leaking into public Certificate Transparency logs while maintaining HSTS compliance across mobile and desktop clients.

---

## 🚦 Ingress & Network Routing Path

| Source | Entry Point | Resolution Target | Internal Routing | Termination / Storage |
| :--- | :--- | :--- | :--- | :--- |
| **Remote Client** | Headscale WireGuard | `vault.denfira-2.local.net` | NPM (443) $\rightarrow$ `127.0.0.1:10000` | SQLite on `C:` (Staged to `H:` & `I:`) |
| **Remote Client** | Headscale WireGuard | `cloud.denfira-2.local.net` | NPM (443) $\rightarrow$ `127.0.0.1:8880` | MariaDB (`C:`) / Payload on `H:\NextcloudData` |
| **Remote Client** | Headscale WireGuard | `planka.denfira-2.local.net` | NPM (443) $\rightarrow$ `127.0.0.1:1337` | PostgreSQL on `127.0.0.1:5432` |
| **Host System** | Task Scheduler | `H:\` Primary HDD | Local Robocopy Mirror | Async Mirror on `I:\` (Hourly Delta) |

---

## 📦 1. Complete Docker Compose Stack

File path: `C:\dev\container\docker-compose.yml` *(or your preferred local storage directory)*

```yaml
services:
  # ---------------------------------------------------------------------------
  # HEADSCALE CONTROL SERVER
  # ---------------------------------------------------------------------------
  headscale:
    image: headscale/headscale:latest
    container_name: headscale
    volumes:
      - ./headscale/config:/etc/headscale
      - ./headscale/data:/var/lib/headscale
    ports:
      - "9500:8080"
    command: serve
    restart: unless-stopped

  # ---------------------------------------------------------------------------
  # TAILSCALE CLIENT SIDECAR
  # ---------------------------------------------------------------------------
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    hostname: denfira-2
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
      - TS_AUTHKEY=<HEADSCALE_PREAUTH_KEY>
      - TS_INSECURE_SERVER=true
      - TS_EXTRA_ARGS=--login-server=http://headscale:8080 --accept-dns=false
    ports:
      - "80:80"       # HTTP traffic (NPM)
      - "443:443"     # HTTPS traffic (NPM)
      - "81:81"       # NPM Admin UI
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    depends_on:
      - headscale
    restart: unless-stopped

  # ---------------------------------------------------------------------------
  # NGINX PROXY MANAGER
  # ---------------------------------------------------------------------------
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    network_mode: "service:tailscale"
    environment:
      - DISABLE_IPV6=true
    depends_on:
      - tailscale
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
    restart: unless-stopped

  # ---------------------------------------------------------------------------
  # VAULTWARDEN (Port: 10000)
  # ---------------------------------------------------------------------------
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    network_mode: "service:tailscale"
    depends_on:
      - tailscale
    environment:
      - WEBSOCKET_ENABLED=true
      - ROCKET_PORT=10000
    volumes:
      - ./vaultwarden/data:/data
      - ./vaultwarden/ssl:/ssl
    restart: unless-stopped

  # ---------------------------------------------------------------------------
  # PLANKA & POSTGRES (Port: 1337)
  # ---------------------------------------------------------------------------
  planka-db:
    image: postgres:15-alpine
    container_name: planka-db
    network_mode: "service:tailscale"
    environment:
      - POSTGRES_DB=planka
      - POSTGRES_USER=<SELECT_USERNAME>
      - POSTGRES_PASSWORD=<SECURE_PLANKA_DB_PASSWORD>
    volumes:
      - ./planka/db:/var/lib/postgresql/data
    restart: unless-stopped

  planka:
    image: ghcr.io/plankanban/planka:latest
    container_name: planka
    network_mode: "service:tailscale"
    depends_on:
      - tailscale
      - planka-db
    environment:
      - BASE_URL=[https://planka.denfira-2.local.net](https://planka.denfira-2.local.net)
      - DATABASE_URL=postgresql://<SELECT_USERNAME>:<SECURE_PLANKA_DB_PASSWORD>@127.0.0.1:5432/planka
      - SECRET_KEY=<RANDOM_GENERATED_SECRET_KEY>
    volumes:
      - ./planka/user-avatars:/app/public/user-avatars
      - ./planka/project-background-images:/app/public/project-background-images
      - ./planka/attachments:/app/public/attachments
    restart: unless-stopped

  # ---------------------------------------------------------------------------
  # NEXTCLOUD & MARIADB (Port: 8880)
  # ---------------------------------------------------------------------------
  nextcloud-db:
    image: mariadb:10.11
    container_name: nextcloud-db
    network_mode: "service:tailscale"
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    environment:
      - MYSQL_ROOT_PASSWORD=<SECURE_MYSQL_ROOT_PASSWORD>
      - MYSQL_PASSWORD=<SECURE_NEXTCLOUD_DB_PASSWORD>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=<SELECT_USERNAME>
    volumes:
      - ./nextcloud/db:/var/lib/mysql
    restart: unless-stopped

  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    network_mode: "service:tailscale"
    depends_on:
      - tailscale
      - nextcloud-db
    command: >
      bash -c "sed -i 's/Listen 80/Listen 8880/' /etc/apache2/ports.conf &&
               sed -i 's/:80/:8880/' /etc/apache2/sites-available/000-default.conf &&
               apache2-foreground"
    environment:
      - MYSQL_HOST=127.0.0.1
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=<SELECT_USERNAME>
      - MYSQL_PASSWORD=<SECURE_NEXTCLOUD_DB_PASSWORD>
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.denfira-2.local.net 127.0.0.1
      - OVERWRITEHOST=cloud.denfira-2.local.net
      - OVERWRITEPROTOCOL=https
      - OVERWRITEWEBROOT=/
    volumes:
      - ./nextcloud/html:/var/www/html
      - H:/NextcloudData:/var/www/html/data
    restart: unless-stopped
---

## ⚙️ 2. Headscale Configuration

File path: `C:\dev\container\headscale\config\config.yaml`

```yaml
server_url: http://<HOST_LAN_IP>:9500
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 0.0.0.0:9090

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48

derp:
  server:
    enabled: false
  urls:
    - [https://controlplane.tailscale.com/derpmap/default](https://controlplane.tailscale.com/derpmap/default)
  auto_update_enabled: true
  update_frequency: 24h

dns:
  magic_dns: true
  base_domain: denfira-2.local.net
  nameservers:
    split: {}
    global:
      - 1.1.1.1
      - 8.8.8.8
  search_domains:
    - denfira-2.local.net
  extra_records:
    - name: vault.denfira-2.local.net
      type: A
      value: <TAILSCALE_SIDECAR_IP>
    - name: planka.denfira-2.local.net
      type: A
      value: <TAILSCALE_SIDECAR_IP>
    - name: cloud.denfira-2.local.net
      type: A
      value: <TAILSCALE_SIDECAR_IP>
