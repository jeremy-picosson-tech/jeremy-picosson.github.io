
---
draft: true
date: 2025-08-01
categories:
  - INFRASTRUCTURE
  - CYBER
comments: true
---

# 🚀 Déploiement complet d’un VPN WireGuard (avec automatisation & Docker)

Ce tutoriel vous guide étape par étape pour déployer un **VPN WireGuard sécurisé** avec :

- Scripts d’installation en **Bash et Python**
- Génération automatique des clés
- Configuration serveur et client
- Génération de QR code pour mobile
- 🚀 Bonus : déploiement avec Docker

---

## 📦 Prérequis

- Serveur Linux (Ubuntu/Debian/CentOS)
- Accès root (`sudo`)
- Port UDP 51820 ouvert
- Python 3, Bash
- Docker (pour partie Docker)

---

## 1. 🔧 Installation de WireGuard (Bash)

```bash
#!/bin/bash
set -e
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard qrencode
sudo mkdir -p /etc/wireguard
sudo chmod 700 /etc/wireguard
```

---

## 2. 🔑 Génération des clés serveur et client

```bash
#!/bin/bash
set -e
cd /etc/wireguard
umask 077

wg genkey | tee privatekey | wg pubkey > publickey
mkdir -p clients
CLIENT="client1"
wg genkey | tee clients/$CLIENT-privatekey | wg pubkey > clients/$CLIENT-publickey
```

---

## 3. ⚙️ Configuration serveur `wg0.conf`

```bash
#!/bin/bash
cd /etc/wireguard
cat <<EOF | sudo tee wg0.conf
[Interface]
PrivateKey = $(cat privatekey)
Address = 10.10.10.1/24
ListenPort = 51820
SaveConfig = true

[Peer]
PublicKey = $(cat clients/client1-publickey)
AllowedIPs = 10.10.10.2/32
EOF

chmod 600 wg0.conf
```

---

## 4. 📜 Génération configuration client (Python)

```python
import os

client = "client1"
client_dir = "/etc/wireguard/clients"
priv = open(f"{client_dir}/{client}-privatekey").read().strip()
server_pub = open("/etc/wireguard/publickey").read().strip()
server_ip = "YOUR.SERVER.IP"

conf = f"""[Interface]
PrivateKey = {priv}
Address = 10.10.10.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = {server_pub}
Endpoint = {server_ip}:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
"""

with open(f"{client_dir}/{client}.conf", "w") as f:
    f.write(conf)
```

---

## 5. ✅ Activer et démarrer WireGuard

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

---

## 6. 📲 Génération QR Code client

```bash
qrencode -t ansiutf8 < /etc/wireguard/clients/client1.conf
```

---

## 7. 🔁 Script tout-en-un `deploy_wireguard.sh`

```bash
#!/bin/bash
set -e
WG_DIR="/etc/wireguard"
CLIENT="client1"
apt update && apt install -y wireguard qrencode

mkdir -p $WG_DIR/clients && cd $WG_DIR && umask 077
wg genkey | tee privatekey | wg pubkey > publickey
wg genkey | tee clients/$CLIENT-privatekey | wg pubkey > clients/$CLIENT-publickey

cat <<EOF > $WG_DIR/wg0.conf
[Interface]
PrivateKey = $(cat privatekey)
Address = 10.10.10.1/24
ListenPort = 51820
SaveConfig = true

[Peer]
PublicKey = $(cat clients/$CLIENT-publickey)
AllowedIPs = 10.10.10.2/32
EOF

cat <<EOF > $WG_DIR/clients/$CLIENT.conf
[Interface]
PrivateKey = $(cat clients/$CLIENT-privatekey)
Address = 10.10.10.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat publickey)
Endpoint = YOUR.SERVER.IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

qrencode -t ansiutf8 < $WG_DIR/clients/$CLIENT.conf
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

---

## 🐳 Déploiement avec Docker (optionnel)

### 📁 Structure du projet

```
wireguard-docker/
├── docker-compose.yml
├── config/
│   └── wg0.conf
├── entrypoint.sh
```

### 📜 `docker-compose.yml`

```yaml
version: "3.8"
services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - SERVERURL=YOUR.SERVER.IP
      - SERVERPORT=51820
      - PEERS=client1
      - PEERDNS=1.1.1.1
      - INTERNAL_SUBNET=10.10.10.0
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    restart: unless-stopped
```

### 📝 `entrypoint.sh` (optionnel si génération custom)

```bash
#!/bin/bash
docker-compose up -d
```

---

## ✅ Vérification

```bash
sudo wg show
```

---

## 🎯 Résultat

- Serveur WireGuard actif sur port UDP 51820
- Client prêt à se connecter avec fichier `.conf` ou QR code
- Déploiement automatisé en Bash et Python

---

