# adguard
This guide covers deploying **AdGuard Home** in a **Proxmox LXC container** and integrating it with **Tailscale** for remote DNS filtering.

**AdGuard Home** is a self-hosted DNS filtering solution that blocks ads, trackers, phishing and malicious domains across your entire network. I initially considered both **Pi-hole** and **AdGuard Home**. After reading feedback from other users and researching their features, I chose **AdGuard** because it seemed more flexible and interesting for my needs.






## 1. Deploying the LXC Container

I chose an **LXC container** instead of a full virtual machine (VM) because AdGuard is a lightweight DNS service that does not require a separate OS kernel.
LXC containers use fewer system resources, start faster, and integrate directly with the Proxmox host. This makes them a practical choice for running lightweight network services. Moreover, an LXC container provides sufficient isolation while keeping resource consumption low.

### 1.1 Download the Debian Template

Open **pve** → select the storage that supports Container Templates (usually named **_local (pve)_** ) → **CT Templates** → **Templates** → Search for `debian-13-standard` → **Download**.

Why Debian 13? It's lightweight and stable. The Standard template provides a minimal installation without a graphical desktop or unnecessary software, making it ideal for running AdGuard in an LXC container.

### 1.2 Create the LXC Container

On the same storage page (in my case, **local (pve)**) → **Create CT**.

Configure **General** tab:
* Container ID
* Hostname
* Set a strong root password
* Enable Unprivileged Container (An unprivileged container is more secure because its root user has limited permissions on the Proxmox host).

**Template
*** Select the downloaded Debian 13 template.

**Disks**
* Storage: local-lvm in my case
* Disk size: 4 GB

**CPU**
* Cores: 1

**Memory**
* Memory: 512 MB
* Swap: 512 MB

**Network**
* Bridge: vmbr0
* IPv4: Static
* IPv4/CIDR: 192.168.1.10/24
* Gateway: 192.168.1.1

  
Finish the wizard and start the container.
