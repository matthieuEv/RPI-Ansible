# RPi Ansible

Automated configuration of a secure Raspberry Pi with VPN, reverse proxy, and Docker services. This Ansible project transforms a Raspberry Pi (or Debian server) into a secure personal server with WireGuard VPN access, container management via Portainer, and Nginx reverse proxy to securely expose your services.

**Main features:**
- 🔐 WireGuard VPN for secure remote access
- 🐳 Docker + Portainer for container management
- 🌐 Nginx Proxy Manager for reverse proxy with SSL
- 🛡️ SSH hardening, UFW firewall, CrowdSec, Fail2ban
- 🔀 Split DNS for private subdomains accessible only via VPN

---

## Table of Contents

1. [Ansible Installation](#1-ansible-installation)
2. [WireGuard Configuration](#2-wireguard-configuration)
3. [Portainer Configuration](#3-portainer-configuration)
4. [Nginx Proxy Manager Configuration](#4-nginx-proxy-manager-configuration)
5. [Split DNS for Private Services](#5-split-dns-for-private-services)
6. [Diagnostic Commands](#6-diagnostic-commands)

---

## 1. Ansible Installation

### Prerequisites

- Ansible 2.9+
- A Raspberry Pi with Raspberry Pi OS (or Debian)
- SSH access to the Pi

### Step 1: Configure the inventory

```bash
cp inventories/prod/hosts.ini.example inventories/prod/hosts.ini
cp inventories/prod/group_vars/all/secrets.yml.example inventories/prod/group_vars/all/secrets.yml
```

Edit `inventories/prod/hosts.ini` with your Pi's IP address you can get using `ip a | grep 192` on the Pi.

### Step 2: Configure the secrets

Edit `inventories/prod/group_vars/all/secrets.yml` and add your SSH public key:

```yaml
ssh_authorized_key: "ssh-ed25519 AAAA... your-email@example.com"
```

> **Tip**: To generate an SSH key:
> ```bash
> ssh-keygen -t ed25519 -C "your-email@example.com"
> cat ~/.ssh/id_ed25519.pub
> ```

### Step 3: Configure the network interface

Find your Pi's network interface:

```bash
ip route | grep default
# Example: default via 192.168.x.x dev eth0 proto dhcp src 192.168.x.x metric 100
#   here its "eth0"
```

Update `inventories/prod/group_vars/all/secrets.yml`:

```yaml
wireguard:
  external_interface: eth0  # Adapt to your machine
```

### Step 4: Configure WireGuard clients (before running the playbook)

Before running Ansible, set up your WireGuard client(s) to avoid running the playbook twice.

#### Install the WireGuard app

| Platform | Installation |
|----------|--------------|
| **macOS** | [App Store](https://apps.apple.com/app/wireguard/id1451685025) |
| **Windows** | [wireguard.com/install](https://www.wireguard.com/install/) |
| **iOS** | App Store → "WireGuard" |
| **Android** | Play Store → "WireGuard" |

#### Create a tunnel and get the public key

1. Open the WireGuard app
2. Click **"+"** → **"Add empty tunnel"**
3. The app automatically generates a **private key** and displays the **public key**
4. **Copy the displayed public key**

#### Add the client in secrets.yml

Edit `inventories/prod/group_vars/all/secrets.yml`:

```yaml
wireguard_peers:
  - name: my-device
    public_key: "PASTE_PUBLIC_KEY_HERE"
    allowed_ips: "10.8.0.2/32"
```

> For multiple devices, add more peers with unique IPs (`10.8.0.3/32`, `10.8.0.4/32`, etc.)

### Step 5: Run the playbook

```bash
ansible-playbook playbooks/site.yml -K
```

> **Important**: Note the WireGuard public key displayed in the step `Display server public key`, you'll need it to finish your clients configuration.

---

## 2. WireGuard Configuration

After running the playbook, complete your WireGuard client configuration.

### Configure the tunnel in the app

Complete the configuration in the WireGuard app:

```ini
[Interface]
PrivateKey = (already filled automatically)
Address = 10.8.0.2/32
DNS = 10.8.0.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = PI_PUBLIC_IP:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

Replace:
- `SERVER_PUBLIC_KEY`: key displayed during playbook execution
- `PI_PUBLIC_IP`: public IP of your router (or local IP if on same network)

### Test the connection

Activate the tunnel and test:

```bash
ping 10.8.0.1
ssh user@10.8.0.1
```

### Add more devices later

For each new device:
1. Create an empty tunnel in the WireGuard app → copy the public key
2. Add a peer in `secrets.yml` with a unique IP (`10.8.0.3/32`, `10.8.0.4/32`, etc.)
3. Re-run the playbook
4. Complete the tunnel configuration in the app

---

## 3. Portainer Configuration

After running the playbook, Portainer is accessible via VPN.

### First login

> ⚠️ **Important**: Create your admin account within minutes of first launch, otherwise Portainer will disable registration for security reasons.

1. Connect to the VPN
2. Access `https://10.8.0.1:9443/`
3. Create your administrator account

---

## 4. Nginx Proxy Manager Configuration

Nginx Proxy Manager allows you to create reverse proxies with automatic SSL certificates.

### First login

1. Connect to the VPN
2. Access `http://10.8.0.1:81`
3. Create your administrator account

### Create a VPN-Only Access List

To restrict access to VPN users only:

1. Go to **Access Lists** → **Add Access List**
2. **Details**: Name = `Wireguard` or `VPN Only` or any name you want
3. **Options**: Check the `Satisfy any` box
4. **Rules**:
   - Add `10.8.0.0/24` in the `Allow` section
   - Let the default `all` in the `Deny` section
   
5. Click **Save**

### Configure Portainer as Proxy Host

1. Go to **Proxy Hosts** → **Add Proxy Host**
2. **Details**:
   - Domain Names: `portainer.<your-domain>.fr`
   - Scheme: `https` ← **Important!**
   - Forward Hostname / IP: `portainer`
   - Forward Port: `9443`
   - Enable **Websockets Support**
   - Enable **Block Common Exploits**
3. **SSL**:
   - Request a new SSL Certificate (Let's Encrypt)
   - Enable **Force SSL**
   - ✅ Check **"Ignore Invalid SSL"**
4. **Access List**: Select the access list you created earlier
5. Click **Save**

### Configure NPM as Proxy Host (optional)

To access NPM via a clean URL instead of `10.8.0.1:81`:

1. Go to **Proxy Hosts** → **Add Proxy Host**
2. **Details**:
   - Domain Names: `nginx.<your-domain>.fr`
   - Scheme: `http`
   - Forward Hostname / IP: `nginx-proxy-manager`
   - Forward Port: `81`
   - Enable **Block Common Exploits**
3. **SSL**: Request a Let's Encrypt certificate if desired
4. **Access List**: Select the access list you created earlier
5. Click **Save**

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| **403 Forbidden** | Access List misconfigured | Ensure `allow 10.8.0.0/24` comes BEFORE `deny all` |
| **502 Bad Gateway** | Container unreachable | Use the container name (`portainer`), not IP |
| **HTTP to HTTPS error** | Wrong scheme | Set Scheme to `https` for Portainer (port 9443) |

---

## 5. Split DNS for Private Services

Split DNS allows you to have public and private subdomains:
- **Public**: `<your-domain>.fr` → accessible to everyone
- **Private**: `portainer.<your-domain>.fr`, `nginx.<your-domain>.fr` → accessible only via VPN

### How it works

```
┌─────────────────────────────────────────────────────────────────┐
│                       Internet User                             │
│                              │                                  │
│                              ▼                                  │
│                    ┌──────────────────────┐                     │
│                    │   <your-domain>.fr   │ ✅ Public           │
│                    └──────────────────────┘                     │
│                              │                                  │
│              ┌───────────────┴───────────────┐                  │
│              ▼                               ▼                  │
│    portainer.<your-domain>.fr          nginx.<your-domain>.fr   │
│         ❌ Blocked                    ❌ Blocked                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    VPN User (WireGuard)                         │
│                              │                                  │
│                    DNS: 10.8.0.1 (dnsmasq)                      │
│                              │                                  │
│              ┌───────────────┴───────────────┐                  │
│              ▼                               ▼                  │
│    portainer.<your-domain>.fr         nginx.<your-domain>.fr    │
│     → 10.8.0.1 ✅                   → 10.8.0.1 ✅                │
└─────────────────────────────────────────────────────────────────┘
```

### Add a subdomain manually

You can add a new private subdomain without re-running Ansible:

```bash
# Connect to the server
ssh user@10.8.0.1

# Add the DNS entry
sudo nano /etc/dnsmasq.d/split-dns.conf
# Add: address=/new.<your-domain>.fr/10.8.0.1

# Restart dnsmasq
sudo systemctl restart dnsmasq
```

Then configure the Proxy Host in Nginx Proxy Manager.

Alternatively, add the subdomain to `secrets.yml` and re-run the playbook for a reproducible setup.

### Test

1. **Without VPN**: `portainer.<your-domain>.fr` → 403 Forbidden ❌
2. **With VPN**: `portainer.<your-domain>.fr` → Works ✅

```bash
# Verify DNS resolution (with VPN)
nslookup portainer.<your-domain>.fr 10.8.0.1
# Should return 10.8.0.1
```

---

## 6. Diagnostic Commands

```bash
# Service status
sudo systemctl status dnsmasq
sudo systemctl status wg-quick@wg0

# Docker containers
docker ps
docker logs nginx-proxy-manager
docker logs portainer

# Docker networks
docker network ls
docker network inspect proxy_network

# DNS test (with VPN)
nslookup portainer.<your-domain>.fr 10.8.0.1
```
