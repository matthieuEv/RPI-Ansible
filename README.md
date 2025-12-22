# RPi Ansible

Playbooks Ansible pour configurer et sécuriser un Raspberry Pi.

## Prérequis

- Ansible 2.9+
- Un Raspberry Pi avec Raspberry Pi OS (ou Debian)
- Accès SSH au Pi

## Installation rapide

### 1. Configurer l'inventaire

```bash
cp inventories/prod/hosts.ini.example inventories/prod/hosts.ini
cp inventories/prod/group_vars/all/secrets.yml.example inventories/prod/group_vars/all/secrets.yml
```

Éditez `inventories/prod/hosts.ini` avec l'IP de votre Pi.

### 2. Configurer les variables

Éditez `inventories/prod/group_vars/all/main.yml` et vérifiez **l'interface réseau externe** :

```bash
# Connectez-vous au Pi/serveur et trouvez l'interface réseau
ip route | grep default
# Exemple de sortie : default via 192.168.1.1 dev eth0
```

Mettez à jour `external_interface` avec le nom de l'interface (ex: `eth0`, `end0`, `ens160`, `wlan0`) :

```yaml
wireguard:
  external_interface: eth0  # ← Adapter selon votre machine
```

### 3. Lancer le playbook

```bash
ansible-playbook playbooks/site.yml -K
```

**Notez la clé publique du serveur WireGuard** qui s'affiche à la fin - vous en aurez besoin pour le client.

---

## Configuration du client VPN (WireGuard)

### Étape 1 : Installer l'app WireGuard

| Plateforme | Installation |
|------------|--------------|
| **macOS** | [App Store](https://apps.apple.com/app/wireguard/id1451685025) |
| **Windows** | [wireguard.com/install](https://www.wireguard.com/install/) |
| **iOS** | App Store → "WireGuard" |
| **Android** | Play Store → "WireGuard" |

### Étape 2 : Créer un tunnel

1. Ouvrez l'app WireGuard
2. Cliquez sur **"+"** → **"Ajouter un tunnel vide"** (ou "Add empty tunnel")
3. L'app génère automatiquement une **clé privée** et affiche la **clé publique**
4. **Copiez la clé publique** affichée

### Étape 3 : Ajouter le client dans Ansible

Éditez `inventories/prod/group_vars/all/secrets.yml` (ignoré par git) :

```yaml
wireguard_peers:
  - name: mon-appareil
    public_key: "COLLEZ_LA_CLE_PUBLIQUE_ICI"
    allowed_ips: "10.8.0.2/32"
```

> **Note** : Créez ce fichier à partir de l'exemple si nécessaire :
> ```bash
> cp inventories/prod/group_vars/all/secrets.yml.example inventories/prod/group_vars/all/secrets.yml
> ```

Relancez le playbook :

```bash
ansible-playbook playbooks/site.yml -K
```

### Étape 4 : Compléter la configuration dans l'app

Dans l'app WireGuard, complétez la configuration du tunnel :

```ini
[Interface]
PrivateKey = (déjà rempli automatiquement)
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = CLE_PUBLIQUE_SERVEUR
Endpoint = IP_PUBLIQUE_DU_PI:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

Remplacez :
- `CLE_PUBLIQUE_SERVEUR` : clé affichée lors du playbook
- `IP_PUBLIQUE_DU_PI` : IP publique de votre box (ou IP locale si même réseau)

### Étape 5 : Se connecter

Activez le tunnel dans l'app WireGuard !

Test : `ping 10.8.0.1` ou `ssh user@10.8.0.1`

---

## Ajouter d'autres appareils

Pour chaque nouvel appareil :
1. Installez l'app WireGuard
2. Créez un tunnel vide → copiez la clé publique
3. Ajoutez un peer dans `all.yml` avec une IP unique (`10.8.0.3/32`, `10.8.0.4/32`, etc.)
4. Relancez le playbook
5. Configurez le tunnel dans l'app

---

## Structure du projet

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

## Rôles disponibles

| Rôle | Description |
|------|-------------|
| common | Configuration de base (timezone, paquets) |
| ssh_hardening | Sécurisation SSH |
| ufw | Firewall |
| wireguard | VPN WireGuard |
| crowdsec | Protection contre les intrusions |
