# 01 - Virtualisation (Proxmox VE Host Setup)

## 🎯 Purpose & Scope
The start section to documents the deployment and configuration of **Proxmox Virtual Environment (PVE)** as a bare-metal Type-1 hypervisor. Following the Rabbit Hole methodology, this initial section establishes the core virtualization foundation to host isolated virtual machines (KVM) and containers (LXC).
---

## 🛠️ Bare-Metal Installation Blueprint

### Step 1: Media Preparation
* **Source:** Downloaded official ISO from the [Proxmox Get Started Guide](https://www.proxmox.com/en/products/proxmox-virtual-environment/get-started).
* **Flashing Tool:** Used BalenaEtcher / Rufus (DD Mode) to write the ISO to a bootable USB drive. For me, I used Ventoy as it is easier to see your .iso file name in GUI bootup.

### Step 2: Host Installation & Network Configuration
* Booted target hardware into USB installation wizard.
* **Target Storage Pool:** Selected primary drive (ext4 / ZFS single-disk setup).
* **Network Parameters:**
  * **FQDN:** `pve-node01.home.lab` # This can be changed later on using host and hostname file. An example: pve-node01.rabbit-hole.home
  * **Static IP:** `192.168.x.x/24` # Up to your preference, private IP address are recommended, optional for DHCP
  * **Gateway:** `192.168.x.1` # Your router/modem gateway and always your gateway, remember it
  * **DNS:** `192.168.x.1` # It is mostly your domain server but it is an internet server like cloudflare or google

### Step 3: Web GUI Management
* Accessed management portal via browser: `https://<STATIC-IP>:8006/` # Later on, you can create your own URL (hint: NGINX!) if you don't like IP address. I promised later on you will like your own proxmox server, I struggle and desparately find the shortcuts and methods of this.
* Disables non-subscription repository warnings and updated base OS packages (`pve-no-subscription`). # No more annoying pop-ups of "Must buy subscriptions", lets keep it free and away from it

---

## 📌 References & Documentation
* [Proxmox VE Official Installation Guide](https://www.proxmox.com/en/products/proxmox-virtual-environment/get-started)
