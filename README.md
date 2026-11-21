# AdGuard Home Deployment
This guide covers deploying **AdGuard Home** in a **Proxmox LXC container** and integrating it with **Tailscale** for remote DNS filtering.

**AdGuard Home** is a self-hosted DNS filtering solution that blocks ads, trackers, phishing and malicious domains across your entire network. I initially considered both **Pi-hole** and **AdGuard Home**. After reading feedback from other users and researching their features, I chose **AdGuard** because it seemed more flexible and interesting for my needs.

---

## Table of Contents

1. [Deploying the LXC Container](#1-deploying-the-lxc-container)
   - [1.1 Download the Debian Template](#11-download-the-debian-template)
   - [1.2 Create the LXC Container](#12-create-the-lxc-container)
   - [1.3 Configure Automatic Startup](#13-configure-automatic-startup)
2. [Update Debian](#2-update-debian)
   - [2.1 DNS Issue](#21-dns-issue)
   - [2.2 Temporary DNS Solution](#22-temporary-dns-solution)
   - [2.3 Update Debian Packages](#23-update-debian-packages)
   - [2.4 Install cURL](#24-install-curl)
3. [Install AdGuard Home](#3-install-adguard-home)
4. [Initial AdGuard Configuration](#4-initial-adguard-configuration)
5. [Router DNS Configuration](#5-router-dns-configuration)
6. [Install Tailscale](#6-install-tailscale)
   - [6.1 Enable TUN Device Access](#61-enable-tun-device-access)
   - [6.2 Install Tailscale](#62-install-tailscale)
   - [6.3 Verify the Connection](#63-verify-the-connection)
7. [Access AdGuard Outside Your LAN](#7-access-adguard-outside-your-lan)
8. [Additional DNS Security Configs](#8-additional-dns-security-configs)
9. [Add DNS Blocklists](#9-add-dns-blocklists)
10. [Restrict DNS Access](#10-restrict-dns-access)

---

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

**Template**
Select the downloaded **Debian 13** template.

**Disks**
* Storage: `local-lvm` in my case
* Disk size: `4 GB`

**CPU**
* Cores: `1`

**Memory**
* Memory: `512 MB`
* Swap: `512 MB`

**Network**
* Bridge: `vmbr0`
* IPv4: `Static`
* IPv4/CIDR: `192.168.1.10/24`
* Gateway: `192.168.1.1`

**A few words about network configuration**

In the **Network** section, I assigned a static IP address to the future AdGuard server: `192.168.1.10`

Before choosing the address, I opened my router's web interface and checked its DHCP range. My router assigns addresses from `192.168.1.100` to `192.168.1.249`. Addresses from `192.168.1.2` to `192.168.1.99` are outside this range and are not automatically assigned by DHCP.

Why didn't I use DHCP Reservation?

**Why didn't I use DHCP Reservation?**

Since I chose an IP address outside my router's DHCP range, I assigned it manually instead of using DHCP reservation. This prevents the router from accidentally assigning the same IP address to another device.

Finish the wizard and start the container.
  
## 1.3 Configure Automatic Startup

Select your Container → **Options** → **Start at boot** → **Enable**

This ensures the AdGuard container starts automatically whenever the Proxmox host boots or restarts.

---

## 2. Update Debian
Start by updating the package lists and installing available updates:

```bash
apt update && apt upgrade -y
```

However, the container couldn't resolve the Debian repository addresses, so the update process failed with _Temporary failure resolving_.

### 2.1 DNS Issue

After investigating, I noticed that the Container had inherited the DNS settings from the Proxmox host. Its DNS server was `100.100.100.100`, which belongs to **Tailscale**.

Tailscale was already installed and configured on the Proxmox host, but not inside our newly created container. Therefore, the container couldn't properly use the host's Tailscale DNS configuration.

### 2.2 Temporary DNS Solution

In the Proxmox web interface → Select Container → **DNS** → **DNS server** → **Edit** → change the DNS server address
The Cloudflare DNS server I chose is `1.1.1.1`

This gives the container an independent DNS server, allowing it to resolve domain names and access Debian repositories.

We can change it back later once Tailscale is installed and configured inside the container.

> P.S. If you don't have Tailscale or another VPN configured on your Proxmox host, you shouldn't encounter this issue.

## 2.3 Update Debian Packages

Let's update Debian packages using the new DNS settings.

```bash
apt update && apt upgrade -y
```

## 2.4 Install cURL

We need to download the official **AdGuard Home** installation script directly from GitHub and run it in our Debian container. For this, we need **cURL** - a CLI tool for downloading files.
```bash
apt install curl -y
```

---

## 3. Install AdGuard Home
Run the [**official installation script**](https://github.com/AdguardTeam/AdGuardHome/blob/69ee3d389910d5d30701cbe867280cf1809853ac/README.md?plain=1#L82-L84):

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

After the installation is complete, it will display the server's IP address and port that you can use to access its web interface through your browser.

---

## 4. Initial AdGuard Configuration

Open the IP address provided by the installer. `http://192.168.1.10:3000` in my case. Click Get Started to begin the configuration.

**Admin Web Interface**

This controls how you access the **AdGuard** dashboard through your browser.
* Listen interface: `All interfaces`
* Port: `80`

**DNS Server**

This determines how **AdGuard** receives DNS queries from your devices.
* Listen interface: `All interfaces`
* Port: `53`

Keep these default values for a typical home network installation.

Then create your **administrator account**.

AdGuard Home will then show instructions explaining how to connect your devices to the DNS server. At this stage, we don't need to configure each device individually. Instead, we can configure our router (network gateway) to use AdGuard Home as its DNS server. This way, all devices connected to our local network (LAN) will automatically use AdGuard for DNS queries without requiring individual configuration.

---

## 5. Router DNS Configuration

I opened my router's web interface and set `192.168.1.10` as the primary DNS server, leaving the secondary DNS field empty. This ensures that devices receiving their network configuration through DHCP use AdGuard Home as their DNS server.

However, if you prefer to have a backup DNS server, you can keep your router's default DNS server or use an alternative such as Cloudflare's `1.1.1.2`, which provides malware protection.

> Keep in mind that devices may use the secondary DNS server without waiting for the primary one to become unavailable. This means some DNS queries could bypass AdGuard Home filtering.

After updating the DNS settings, I was prompted to **renew the DHCP leases** on my devices to apply the new network configuration.

On **Windows**, this can be done using two commands in Command Prompt:

```cmd
ipconfig /release
ipconfig /renew
```

If you're using **Linux (Ubuntu/Debian)** NetworkManager, the easiest way is to restart the network connection.

```bash
sudo nmcli networking off
sudo nmcli networking on
```

On **macOS**, I renewed the DHCP lease through the network settings:

**System Settings** → **Network**→ **Details...** → **TCP/IP** → **Renew DHCP Lease**

---

## 6. Install Tailscale

Here I want to install **Tailscale** directly in my **AdGuard LXC container**. This will allow me to access AdGuard Home remotely and use its DNS filtering even when I'm outside my home network.

### 6.1 Enable TUN Device Access

First, we need to allow our LXC container to access the TUN (Network Tunnel) device. Unlike a full VM, an LXC container shares the Proxmox host's Linux kernel and has restricted access to system devices. Tailscale needs access to `/dev/net/tun` to create its virtual network interface.

This is why we need to enable TUN device access manually in our LXC container. On regular VMs, this usually works without any additional configuration.

In Proxmox:

select your **container** → **Resources** → **Add** → **Device Passthrough** → set the device path to `/dev/net/tun`

Restart the container.

## 6.2 Install Tailscale

Inside the container, run the [**official installation command**](https://tailscale.com/docs/install/linux) to download and install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Once the installation is complete, start Tailscale:

```bash
tailscale up
```

The terminal will display an authentication URL. Open it in your browser and authorize the device.

## 6.3 Verify the connection

To check whether Tailscale is running, execute:

```bash
tailscale status
```

To get the container's Tailscale IPv4 address, run:

```bash
tailscale ip -4
```

Tailscale IPv4 address: 100.x.x.x

---

## 7. Access AdGuard Outside Your LAN

At this point, AdGuard Home is configured to work within our local network (LAN).

But what if we want to use AdGuard while we're away from home? How can we access it remotely through the Tailscale network?

**Open the configuration file:**

```bash
nano /opt/AdGuardHome/AdGuardHome.yaml
```

Find dns → bind_hosts and add both IP addresses:
```bash
dns:
  bind_hosts:
    - 192.168.1.10
    - <your tailscale ip>
  port: 53
```

Press Ctrl + O, then Enter to save, and Ctrl + X to exit Nano.

Save the changes and restart AdGuard Home:

```bash
systemctl restart AdGuardHome
```

We also need to add AdGuard's Tailscale IP address to **Global Nameservers** in the Tailscale Admin Console. This allows devices connected to our Tailscale network to use AdGuard as their DNS server even when we're away from home.

Go to **Tailscale Admin Console** (I sign in using my GitHub account) → **DNS** → **Nameservers** → **Add nameserver** → **Custom...** and enter AdGuard's Tailscale IP address.

* Override local DNS `Enabled`
* MagicDNS `Enabled`

Make sure **"Use Tailscale DNS"** settings is enabled on each client device.
This allows remote devices to use AdGuard even when connected through mobile data or another Wi-Fi network.

**AdGuard** will now listen for DNS queries on both network interfaces. Moreover, the **AdGuard** web interface is also accessible through its Tailscale IP address! We can now see DNS queries from our Tailscale clients.

---

## 8. Additional DNS Security Configs

The following are the recommended settings used in this deployment.

Open the AdGuard Home dashboard and navigate to **Settings** → **DNS**

**Upstream DNS servers**
* Upstream Mode `Load balancing`

**DNS server configuration**
* Rate limit: `20 req/s`
* EDNS Client Subnet: `Disabled`

**DNS Cache**
* DNSSEC: `Enabled`
* DNS cache: `Enabled`
* Cache size: `16 MB` (16777216)
* Minimum TTL override: `0`
* Maximum TTL override: `0`
* Optimistic caching: `Enabled`

---

## 9. Add DNS Blocklists

The blocklists below reflect my personal preferences. You can choose different lists depending on your needs and how aggressively you want to block ads, trackers, and other domains.

Navigate to the AdGuard Home dashboard:

**Filters** → **DNS blocklists**

You can find more DNS blocklists in the official [AdGuard Hostlists Registry](https://github.com/AdguardTeam/HostlistsRegistry.git).

---

## 10. Restrict DNS Access

AdGuard Home allows us to restrict which devices can send DNS queries to our server. You can specify allowed devices using their IP addresses or client identifiers.

Go to AdGuard dashboard:

**Settings** → **DNS** → **Access settings** → **Allowed clients**

After testing this feature, I decided it was overkill for my setup. My home network is already reasonably isolated, and maintaining a list of allowed devices would add unnecessary complexity, especially since devices use both LAN and Tailscale IP addresses to reach AdGuard Home. This could also require DHCP reservations and ongoing IP address management. For my setup, the extra maintenance isn't worth it.

However, this is a useful feature if you want to control or separate DNS access for specific devices or groups.
