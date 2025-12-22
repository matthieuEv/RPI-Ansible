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
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = PI_PUBLIC_IP:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

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
    ├── ssh_hardening/
    ├── ufw/
    ├── wireguard/
    └── crowdsec/
```

## Available Roles

| Role | Description |
|------|-------------|
| common | Basic configuration (timezone, packages) |
| ssh_hardening | SSH hardening |
| ufw | Firewall |
| wireguard | WireGuard VPN |
| crowdsec | Intrusion protection |
