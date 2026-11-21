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

DHCP Reservation is used to make a device receive the same IP address every time, but the reserved address normally needs to be within the router's DHCP pool. Since I chose an address outside that range (`192.168.1.10`), a reservation wasn't necessary. The router won't accidentally assign the same address to another device.

Finish the wizard and start the container.
  
## 1.3 Configure Automatic Startup

Select your Container → **Options** → **Start at boot** → **Enable**

This ensures the AdGuard container starts automatically whenever the Proxmox host boots or restarts.

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

## 3. Install AdGuard Home
Run the [**official installation script**](https://github.com/AdguardTeam/AdGuardHome/blob/69ee3d389910d5d30701cbe867280cf1809853ac/README.md?plain=1#L82-L84):

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

After the installation is complete, it will display the server's IP address and port that you can use to access its web interface through your browser.

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

AdGuard Home will thenshow instructions explaining how to connect your devices to the DNS server. At this stage, we don't need to configure each device individually. Instead, we can configure our router (network gateway) to use AdGuard Home as its DNS server. This way, all devices connected to our local network (LAN) will automatically use AdGuard for DNS queries without requiring individual configuration.


## 5. Router DNS Configuration

I opened my router's web interface and set `192.168.1.10` as the primary DNS server, leaving the router's default DNS server as the secondary one.

This means devices on my LAN (that receive their network settings from the router) will use **AdGuard** for DNS queries. If the AdGuard server becomes unavailable, devices can fall back to the secondary DNS server and hopefully retain internet access.

After updating the DNS settings on my router, it asked me to renew the DHCP lease on my devices to apply the new network configuration.

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

