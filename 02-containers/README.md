# 02 - Containers (Orchestration)

## 🎯 Purpose & Scope
This section documents containerization strategies, environment isolates, and service orchestrations running on top of Linux hosts or within Proxmox LXC containers.

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
