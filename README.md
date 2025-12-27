# RPi Ansible

Ansible playbooks to configure and secure a Raspberry Pi.

## Prerequisites

- Ansible 2.9+
- A Raspberry Pi with Raspberry Pi OS (or Debian)
- SSH access to the Pi

## Quick Installation

### 1. Configure the inventory

```bash
cp inventories/prod/hosts.ini.example inventories/prod/hosts.ini
cp inventories/prod/group_vars/all/secrets.yml.example inventories/prod/group_vars/all/secrets.yml
```

Edit `inventories/prod/hosts.ini` with your Pi's IP address.

### 2. Configure the secrets

Edit `inventories/prod/group_vars/all/secrets.yml` and add your SSH public key:

```yaml
# SSH public key for remote access
ssh_authorized_key: "ssh-ed25519 AAAA... your-email@example.com"
```

> **Tip**: If you don't have an SSH key yet, generate one with:
> ```bash
> ssh-keygen -t ed25519 -C "your-email@example.com"
> cat ~/.ssh/id_ed25519.pub  # Copy this public key
> ```

This key will be added to the server's `authorized_keys`, allowing you to connect via SSH through WireGuard without a password.

### 3. Configure the variables

Edit `inventories/prod/group_vars/all/main.yml` and verify the **external network interface**:

```bash
# Connect to the Pi/server and find the network interface
ip route | grep default
# Example output: default via 192.168.1.1 dev eth0
```

Update `external_interface` with the interface name (e.g., `eth0`, `end0`, `ens160`, `wlan0`):

```yaml
wireguard:
  external_interface: eth0  # ← Adapt according to your machine
```

### 4. Run the playbook

```bash
ansible-playbook playbooks/site.yml -K
```

**Note the WireGuard server public key** displayed at the end - you'll need it for the client.

---

## VPN Client Configuration (WireGuard)

### Step 1: Install the WireGuard app

| Platform | Installation |
|------------|--------------|
| **macOS** | [App Store](https://apps.apple.com/app/wireguard/id1451685025) |
| **Windows** | [wireguard.com/install](https://www.wireguard.com/install/) |
| **iOS** | App Store → "WireGuard" |
| **Android** | Play Store → "WireGuard" |

### Step 2: Create a tunnel

1. Open the WireGuard app
2. Click on **"+"** → **"Add empty tunnel"**
3. The app automatically generates a **private key** and displays the **public key**
4. **Copy the displayed public key**

### Step 3: Add the client in Ansible

Edit `inventories/prod/group_vars/all/secrets.yml` (ignored by git):

```yaml
wireguard_peers:
  - name: my-device
    public_key: "PASTE_PUBLIC_KEY_HERE"
    allowed_ips: "10.8.0.2/32"
```

> **Note**: Create this file from the example if needed:
> ```bash
> cp inventories/prod/group_vars/all/secrets.yml.example inventories/prod/group_vars/all/secrets.yml
> ```

Re-run the playbook:

```bash
ansible-playbook playbooks/site.yml -K
```

### Step 4: Complete the configuration in the app

In the WireGuard app, complete the tunnel configuration:

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

> **Important**: The DNS is set to `10.8.0.1` (WireGuard server) to enable Split DNS.
> This allows private subdomains to resolve only when connected to the VPN.

Replace:
- `SERVER_PUBLIC_KEY`: key displayed during playbook execution
- `PI_PUBLIC_IP`: public IP of your router (or local IP if on same network)

### Step 5: Connect

Activate the tunnel in the WireGuard app!

Test: `ping 10.8.0.1` or `ssh user@10.8.0.1`

---

## Adding More Devices

For each new device:
1. Install the WireGuard app
2. Create an empty tunnel → copy the public key
3. Add a peer in `all.yml` with a unique IP (`10.8.0.3/32`, `10.8.0.4/32`, etc.)
4. Re-run the playbook
5. Configure the tunnel in the app

---

## Project Structure

```
rpi-ansible/
├── ansible.cfg
├── inventories/prod/
│   ├── hosts.ini
│   └── group_vars/all.yml
├── playbooks/site.yml
└── roles/
    ├── common/
    ├── dns_split/
    ├── docker/
    ├── ssh_hardening/
    ├── ufw/
    ├── wireguard/
    └── crowdsec/
```

## Available Roles

| Role | Description |
|------|-------------|
| common | Basic configuration (timezone, packages) |
| dns_split | Split DNS for private subdomains (dnsmasq) |
| docker | Docker + Portainer + Nginx Proxy Manager |
| ssh_hardening | SSH hardening |
| ufw | Firewall |
| wireguard | WireGuard VPN |
| crowdsec | Intrusion protection |

---

## Split DNS for Private Services

This setup allows you to have public and private subdomains:
- **Public**: `example.fr` → accessible to everyone
- **Private**: `portainer.example.fr`, `nginx.example.fr` → only accessible via WireGuard VPN

### How it works

```
┌─────────────────────────────────────────────────────────────────┐
│                        Internet User                            │
│                              │                                  │
│                              ▼                                  │
│                    ┌─────────────────┐                          │
│                    │   example.fr    │ ✅ Public                │
│                    │  (Nginx NPM)    │                          │
│                    └─────────────────┘                          │
│                              │                                  │
│              ┌───────────────┴───────────────┐                  │
│              ▼                               ▼                  │
│    portainer.example.fr            nginx.example.fr             │
│         ❌ Blocked                    ❌ Blocked                │
│    (Access List: VPN only)      (Access List: VPN only)         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     VPN User (WireGuard)                        │
│                              │                                  │
│                    DNS: 10.8.0.1 (dnsmasq)                      │
│                              │                                  │
│              ┌───────────────┴───────────────┐                  │
│              ▼                               ▼                  │
│    portainer.example.fr            nginx.example.fr             │
│     → 10.8.0.1 ✅                   → 10.8.0.1 ✅               │
│    (Split DNS resolution)       (Split DNS resolution)          │
└─────────────────────────────────────────────────────────────────┘
```

### Configuration

#### 1. Configure private domains

Edit `inventories/prod/group_vars/all/secrets.yml`:

```yaml
# Your base domain
base_domain: "example.fr"

# Private subdomains accessible only via VPN
private_domains:
  - name: "Portainer"
    subdomain: "portainer"
    base_domain: "{{ base_domain }}"
  - name: "Nginx Proxy Manager"
    subdomain: "nginx"
    base_domain: "{{ base_domain }}"
```

#### 2. Run the playbook

```bash
ansible-playbook playbooks/site.yml -K
```

This installs **dnsmasq** as a local DNS server that:
- Resolves private subdomains to the WireGuard server IP (`10.8.0.1`)
- Forwards all other DNS queries to Cloudflare (`1.1.1.1`)

#### 3. Configure Nginx Proxy Manager

After running the playbook, connect to your VPN and access Nginx Proxy Manager:

**URL**: `http://10.8.0.1:81`

**Default credentials**:
- Email: `admin@example.com`
- Password: `changeme`

> You will be prompted to change these on first login.

##### Step 1: Create the VPN-Only Access List

1. Go to **Access Lists** → **Add Access List**
2. **Details** tab:
   - Name: `VPN Only`
3. **Access** tab:
   - Click **Add** and enter: `allow` → `10.8.0.0/24`
   - Click **Add** and enter: `deny` → `all`
   
   > ⚠️ **Important**: The `allow` rule MUST come before `deny all`, otherwise all access will be blocked (403 Forbidden).

4. Click **Save**

##### Step 2: Configure Portainer Proxy Host

> 💡 **First-time setup**: Before configuring the proxy, access Portainer directly at `https://10.8.0.1:9443/` to create your admin user. You must do this within a few minutes after first launch, otherwise Portainer will disable registration for security.

Portainer uses HTTPS on port 9443, so the configuration is specific:

1. Go to **Proxy Hosts** → **Add Proxy Host**
2. **Details** tab:
   - Domain Names: `portainer.example.fr`
   - Scheme: `https` ← **Important!**
   - Forward Hostname / IP: `portainer` (Docker container name)
   - Forward Port: `9443`
   - Enable **Websockets Support**
3. **SSL** tab:
   - Request a new SSL Certificate (Let's Encrypt) or use your own
   - Enable **Force SSL**
   - ✅ Check **"Ignore Invalid SSL"** (Portainer uses a self-signed certificate)
4. **Access List**: Select `VPN Only`
5. Click **Save**

> 💡 If you get "Client sent an HTTP request to an HTTPS server", you forgot to set Scheme to `https`.

##### Step 3: Configure Nginx Proxy Manager Proxy Host (optional)

To access NPM via a clean URL instead of `10.8.0.1:81`:

1. Go to **Proxy Hosts** → **Add Proxy Host**
2. **Details** tab:
   - Domain Names: `nginx.example.fr`
   - Scheme: `http`
   - Forward Hostname / IP: `nginx-proxy-manager` (or `127.0.0.1`)
   - Forward Port: `81`
3. **SSL** tab: Request a new SSL Certificate if desired
4. **Access List**: Select `VPN Only`
5. Click **Save**

##### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| **403 Forbidden** | Access List misconfigured | Ensure `allow 10.8.0.0/24` comes BEFORE `deny all` |
| **502 Bad Gateway** | Container not reachable | Use container name (`portainer`), not IP. Check both are on `proxy_network` |
| **HTTP to HTTPS error** | Wrong scheme | Set Scheme to `https` for Portainer (port 9443) |
| **SSL errors** | Self-signed certificate rejected | Enable "Ignore Invalid SSL" in SSL tab |

#### 4. Update WireGuard client configuration

Make sure your WireGuard client uses the VPN's DNS:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.8.0.2/32
DNS = 10.8.0.1    # ← Important: Use VPN DNS for split DNS

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = PI_PUBLIC_IP:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

### Testing

1. **Without VPN**: 
   - `example.fr` → Works ✅
   - `portainer.example.fr` → 403 Forbidden ❌

2. **With VPN connected**:
   - `example.fr` → Works ✅
   - `portainer.example.fr` → Works ✅ (resolves to 10.8.0.1)

3. **Verify DNS resolution** (with VPN):
   ```bash
   nslookup portainer.example.fr 10.8.0.1
   # Should return 10.8.0.1
   ```

### Adding a New Private Subdomain Manually

You can add a new private subdomain (e.g., `pocketbase.example.fr`) without re-running the Ansible playbook, though you can also add it via the playbook by updating `secrets.yml` and running `ansible-playbook playbooks/site.yml -K`.

**Manual method:**

#### 1. SSH into your server

```bash
ssh user@10.8.0.1  # or your server IP
```

#### 2. Add DNS entry to dnsmasq

```bash
sudo nano /etc/dnsmasq.d/split-dns.conf
```

Add the following line:

```
# pocketbase.example.fr - resolves to WireGuard server IP
address=/pocketbase.example.fr/10.8.0.1
```

Save and exit (Ctrl+X, Y, Enter).

#### 3. Restart dnsmasq

```bash
sudo systemctl restart dnsmasq
```

#### 4. Verify DNS resolution

From your client (with VPN connected):

```bash
nslookup pocketbase.example.fr 10.8.0.1
# Should return 10.8.0.1
```

#### 5. Configure Nginx Proxy Manager

1. Go to **Proxy Hosts** → **Add Proxy Host**
2. **Details** tab:
   - Domain Names: `pocketbase.example.fr`
   - Scheme: `http` (or `https` if your app uses SSL)
   - Forward Hostname / IP: IP of your Pocketbase container/service
   - Forward Port: Port of your Pocketbase service
   - Enable **Websockets Support** if needed
3. **SSL** tab (optional): Request Let's Encrypt certificate
4. **Access List**: Select `VPN Only`
5. Save

#### 6. Test

With VPN connected, visit `https://pocketbase.example.fr` in your browser. It should work! ✅

> **Note**: This manual method is useful for quick testing. For a permanent setup, it's recommended to add the subdomain to `secrets.yml` and re-run the playbook so the configuration is version-controlled and reproducible.

---

## Diagnostic Commands

```bash
# Check if dnsmasq is running
sudo systemctl status dnsmasq

# Test DNS resolution (from client with VPN)
nslookup portainer.example.fr 10.8.0.1

# Check Docker networks
docker network ls
docker network inspect proxy_network

# Check running containers
docker ps

# Check NPM logs
docker logs nginx-proxy-manager

# Check Portainer logs
docker logs portainer
```
