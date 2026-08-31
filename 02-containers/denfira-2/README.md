# Denfira-2 Infrastructure Runbook

**Sidecar Mesh · BitLocker · Disaster Recovery**

## Architecture Overview

- **OS Drive (C:)**: NVMe SSD (BitLocker Encrypted via TPM). Hosts Docker engine, Compose files, and low-latency databases.
- **Primary HDD (H:)**: Miri-Da (1TB, BitLocker Encrypted + Auto-Unlock). Active payload storage for Nextcloud (`H:\NextcloudData`) and Vaultwarden nightly backups (`H:\VaultwardenBackup`).
- **Secondary Mirror HDD (I:)**: Miri-Da-Backup (1TB, BitLocker Encrypted + Auto-Unlock). Async mirror target.
- **Networking Pattern**: **Tailscale Sidecar Container**. All application containers (npm, vaultwarden, planka, planka-db, nextcloud, nextcloud-db) bind to `network_mode: "service:tailscale"`, routing through node IP `100.64.0.3` and communicating internally over `127.0.0.1`.

| Component / Tier | Drive / Resource | Encryption / Security | Role & Stored Services |
|---|---|---|---|
| **System & Engine** | C: (NVMe SSD) | TPM BitLocker | Docker Engine, docker-compose, Compose files, and low-latency database engines (MariaDB, PostgreSQL, SQLite). |
| **Primary Data Storage** | H: (1TB HDD) | BitLocker + Auto-Unlock | Active payload storage: Nextcloud user data (`H:\NextcloudData`) and Vaultwarden nightly backup target (`H:\VaultwardenBackup`). |
| **Mirror / Secondary** | I: (1TB HDD) | BitLocker + Auto-Unlock | Asynchronous offline/mirror target for automated hourly Nextcloud sync and secondary backup retention. |
| **Networking & Mesh** | denfira-2 (Sidecar) | WireGuard + Headscale Control Plane | Shared network namespace (`100.64.0.3`) isolating all container traffic to `127.0.0.1` with zero exposed gateway router ports. |
| **Ingress & TLS** | NPM (:80, :443, :81) | mkcert Root CA | Reverse proxy, SSL/TLS termination, and WebSocket routing for vault, cloud, and planka subdomains. |

## Core System & Security Benefits

- **Zero WAN Attack Surface:** No open ports on the physical gateway router. Ingress is strictly mediated by WireGuard cryptographic handshakes managed through Headscale.
- **Network Namespace Isolation (Sidecar Pattern):** By locking container communication into `network_mode: "service:tailscale"`, application traffic between databases, backends, and the reverse proxy never crosses external interfaces — all database ports listen strictly on localhost loopback (`127.0.0.1`).
- **Hardware-Bound Cryptographic At-Rest Protection:**
  - System partition (C:) is safeguarded against offline extraction via TPM-enforced BitLocker.
  - Secondary HDDs (H: and I:) use BitLocker Auto-Unlock tied to the host TPM, ensuring automated boot recovery while protecting drives if physically detached.
- **Crash-Consistent Database Snapshots:** Utilising `docker pause` guarantees that SQLite and WAL files are flushed to disk before Robocopy executions, eliminating fragmented snapshot corruption during backup routines.
- **Private PKI Trust Architecture:** End-to-end SSL/TLS termination via mkcert root CA prevents internal domain lookups from leaking into public Certificate Transparency logs while ensuring strict HSTS compliance across mobile and desktop clients.

| Source | Entry Point | Resolution Target | Internal Routing | Termination / Storage |
|---|---|---|---|---|
| Mobile / Remote Client | Headscale WireGuard | vault.denfira-2.local.net | NPM (443) → 127.0.0.1:10000 | SQLite on C: (Nightly staged to H: & I:) |
| Mobile / Remote Client | Headscale WireGuard | cloud.denfira-2.local.net | NPM (443) → 127.0.0.1:8880 | MariaDB (C:) / Mass Data on `H:\NextcloudData` |
| Mobile / Remote Client | Headscale WireGuard | planka.denfira-2.local.net | NPM (443) → 127.0.0.1:1337 | Postgres on 127.0.0.1:5432 |
| Host System | Task Scheduler | H:\ Primary HDD | Local Robocopy Mirror | Async Mirror on I:\ (Hourly Delta) |

## 1. Complete Docker Compose Stack (`C:\dev\container\docker-compose.yml`)

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
      - "80:80"   # HTTP traffic (NPM)
      - "443:443" # HTTPS traffic (NPM)
      - "81:81"   # NPM Admin UI
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
      - BASE_URL=https://planka.denfira-2.local.net
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
```

## 2. Headscale Configuration (`C:\dev\container\headscale\config\config.yaml`)

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
    - https://controlplane.tailscale.com/derpmap/default
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
```

## 3. Nginx Proxy Manager (NPM) Forwarding Rules

Because all services share the tailscale container network stack, all forward targets resolve over loopback:

| Domain Name | Forward Hostname / IP | Forward Port | Key Toggles |
|---|---|---|---|
| vault.denfira-2.local.net | 127.0.0.1 | 10000 | Websockets Support, Block Common Exploits, Custom SSL (mkcert) |
| planka.denfira-2.local.net | 127.0.0.1 | 1337 | Websockets Support, Custom SSL (mkcert) |
| cloud.denfira-2.local.net | 127.0.0.1 | 8880 | Websockets Support, HSTS, Block Common Exploits, Custom SSL (mkcert) |

**Key Configuration Notes to Include**

- **SSL / TLS Certificate:** Ensure each host entry has **Force SSL** and **HTTP/2 Support** enabled, referencing your custom wildcard / domain certificate generated by mkcert.
- **Nextcloud WebDAV / CalDAV Calibrations (Optional):** In NPM under the **Advanced** tab for cloud.denfira-2.local.net, you can add the standard rewrite rules if Nextcloud throws discovery warnings:

```nginx
client_max_body_size 0;
proxy_buffers 16 4k;
proxy_buffer_size 2k;
proxy_read_timeout 600s;
proxy_send_timeout 600s;

location /.well-known/carddav {
  return 301 $scheme://$host/remote.php/dav;
}
location /.well-known/caldav {
  return 301 $scheme://$host/remote.php/dav;
}
```

## 4. BitLocker Setup & Drive Automation

```powershell
# 1. Enable BitLocker encryption on storage volumes
manage-bde -on H: -UsedSpaceOnly -Password
manage-bde -on I: -UsedSpaceOnly -Password

# 2. Append 48-digit numerical recovery passwords
manage-bde -protectors -add H: -RecoveryPassword
manage-bde -protectors -add I: -RecoveryPassword

# 3. Enable silent auto-unlock bound to the host TPM/OS volume
manage-bde -autounlock -enable H:
manage-bde -autounlock -enable I:
```

## 5. Storage Replication & Scheduled Tasks

### A. Hourly Nextcloud Mirror Task (`Miri-Da-Mirror`)

Registers a scheduled task under the **SYSTEM** account to mirror Nextcloud data across storage drives every hour:

```powershell
$Action = New-ScheduledTaskAction -Execute "robocopy.exe" -Argument "H:\NextcloudData I:\NextcloudData /MIR /FFT /R:1 /W:2 /NP /NFL /NDL"
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Hours 1)
Register-ScheduledTask -TaskName "Miri-Da-Mirror" -Action $Action -Trigger $Trigger -User "SYSTEM" -RunLevel Highest -Force
```

### B. Nightly Atomic Vaultwarden Backup Script (`C:\dev\container\vaultwarden\backup-vaultwarden.ps1`)

Pauses the running Vaultwarden container to flush SQLite WAL memory and ensure zero database corruption during snapshot staging:

```powershell
$SourceDir  = "C:\dev\container\vaultwarden\data"
$StagingDir = "C:\dev\container\vaultwarden-staging"
$DestH      = "H:\VaultwardenBackup"
$DestI      = "I:\VaultwardenBackup"

# 1. Ensure target directories exist
New-Item -ItemType Directory -Force -Path $StagingDir, $DestH, $DestI | Out-Null

# 2. Pause container to flush WAL memory and guarantee zero SQLite corruption
docker pause vaultwarden
try {
    # 3. Mirror live database, WAL logs, RSA keys, and attachments to staging
    robocopy $SourceDir $StagingDir /MIR /NP /NFL /NDL /R:1 /W:1 | Out-Null
}
finally {
    # 4. Guarantee container unpauses immediately
    docker unpause vaultwarden
}

# 5. Mirror Staging -> Primary Encrypted Drive (H:)
robocopy $StagingDir $DestH /MIR /FFT /NP /NFL /NDL /R:1 /W:2 | Out-Null

# 6. Mirror Primary Encrypted Drive (H:) -> Secondary Encrypted Drive (I:)
robocopy $DestH $DestI /MIR /FFT /NP /NFL /NDL /R:1 /W:2 | Out-Null

Write-Host "Vaultwarden backup completed successfully to H: and I:" -ForegroundColor Green
```

### C. Register Nightly Vaultwarden Backup Task (2:00 AM)

Registers the automated backup script to run every night at 2:00 AM:

```powershell
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\dev\container\vaultwarden\backup-vaultwarden.ps1"
$Trigger = New-ScheduledTaskTrigger -Daily -At 2:00AM
Register-ScheduledTask -TaskName "Vaultwarden-Backup" -Action $Action -Trigger $Trigger -User "SYSTEM" -RunLevel Highest -Force
```

## 6. SSL / TLS Setup with mkcert

### A. Generate Wildcard Certificate for Local Mesh Domains

Run on the host workstation where mkcert is installed:

```powershell
# Create the local wildcard certificate, SANs, and private key
mkcert "*.denfira-2.local.net" "denfira-2.local.net" localhost 127.0.0.1 100.64.0.3

# Locate the root CA folder on your machine
mkcert -CAROOT
```

- Rename the output files for clarity:
  - `_wildcard.denfira-2.local.net+4.pem` → `denfira-wildcard.crt`
  - `_wildcard.denfira-2.local.net+4-key.pem` → `denfira-wildcard.key`

### B. Import Certificate into Nginx Proxy Manager (NPM)

1. In the NPM Web UI (`http://localhost:81`), navigate to **SSL Certificates** → **Add SSL Certificate** → **Custom**.
2. **Name:** `denfira-wildcard`
3. **Certificate Key:** Paste the entire contents of `denfira-wildcard.key`.
4. **Certificate:** Paste the entire contents of `denfira-wildcard.crt`.
5. Apply this certificate to all Proxy Hosts (vault, cloud, planka) and enable **Force SSL**, **HTTP/2 Support**, and **HSTS**.

### C. Install Root CA on Client Endpoints

To prevent browser and app TLS validation errors over the mesh, install `rootCA.pem` onto each client device.

```powershell
# Retrieve the exact path to your root CA directory
mkcert -CAROOT
```

**Security Note:** Only distribute `rootCA.pem` (rename to `rootCA.crt` for mobile installers). **Never** share or export `rootCA-key.pem`.

- **Android (Samsung Z Fold7 / Tablets):**
  1. Transfer `rootCA.crt` to internal storage.
  2. Navigate to **Settings** → **Security & Privacy** → **More security settings** → **Encryption & credentials** (or search *"Install a certificate"*).
  3. Select **Install from storage** → **CA certificate** → **Install anyway**.
  4. Select `rootCA.crt` and authenticate with your device PIN/biometrics.

- **iOS / iPadOS:**
  1. Send `rootCA.pem` to the device via AirDrop or local file transfer.
  2. Open the file and choose **Install Profile**.
  3. Navigate to **Settings** → **Profile Downloaded** → **Install**.
  4. Navigate to **Settings** → **General** → **About** → **Certificate Trust Settings**.
  5. Under **Enable full trust for root certificates**, toggle **ON** the mkcert Root CA.

- **Secondary Windows Workstations:**
  1. Double-click `rootCA.crt` → **Install Certificate**.
  2. Select **Local Machine** → Place all certificates in the **Trusted Root Certification Authorities** store.

- **Linux (Debian / Ubuntu / Proxmox nodes):**

```bash
sudo cp rootCA.pem /usr/local/share/ca-certificates/mkcert-rootCA.crt
sudo update-ca-certificates
```

## 7. Known Environment Quirks & Troubleshooting

### A. Nextcloud: Windows NTFS Data Directory Permissions Fix

- **Symptom:** Nextcloud fails to boot or throws a setup error:
  > "Your data directory is readable by other people. Please change the permissions to 0770..."
- **Root Cause:** Host bind mounts from Windows NTFS storage (`H:/NextcloudData:/var/www/html/data`) mount into Linux containers with static `0777` permissions. Because NTFS does not implement POSIX permission bits, `chmod 0770` cannot be applied directly on the host mount.
- **Resolution:** Disable strict data directory permission checks.

**Method 1: Direct File Edit**

1. Open `C:\dev\container\nextcloud\html\config\config.php`.
2. Insert this key into the `$CONFIG` array:

```php
'check_data_directory_permissions' => false,
```

**Method 2: Dynamic CLI Configuration**

```powershell
docker exec -u www-data nextcloud php occ config:system:set check_data_directory_permissions --value=false --type=boolean
```

### B. Initialising Planka Administrator Account

Because Planka does not generate a default administrator account upon fresh database creation without predefined environment variables, seed the initial admin via the container CLI:

1. Confirm the container stack is active:

```powershell
cd C:\dev\container
docker compose ps
```

2. Run the interactive account creation script:

```powershell
docker exec -it planka npm run db:create-admin-user
```

3. Enter your administrator details at the prompt:
   - **Email:** `<admin_email>` (e.g., `admin@denfira-2.local.net`)
   - **Password:** `<strong_admin_password>`
   - **Name:** `<display_name>`
   - **Username:** `<username>`

4. Log into [https://planka.denfira-2.local.net](https://planka.denfira-2.local.net) with the newly created credentials and save them in Vaultwarden.
