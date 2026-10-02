# Guide A à Z — Supervision sécurisée (Ansible + WireGuard + Prometheus/Grafana) sous VMware

Ce guide suppose **VMware Workstation** sur un PC **Windows**. Les playbooks du dossier `ansible/` restent identiques ; seules la création des VMs et la configuration réseau changent.

## Architecture

| VM | Rôle | IP LAN (host-only) | IP VPN |
|---|---|---|---|
| `monitoring` | WireGuard (serveur), Prometheus, Grafana, contrôle Ansible | 192.168.56.10 | 10.8.0.1 |
| `node1` | WireGuard (client), Node Exporter | 192.168.56.11 | 10.8.0.2 |

Flux : Prometheus (sur `monitoring`) collecte `node1` via `10.8.0.2:9100`, exclusivement dans le tunnel WireGuard.

---

## Phase 1 — Fixer le réseau host-only de VMware

VMware attribue par défaut un sous-réseau aléatoire à VMnet1. On le fixe pour garder les IP du projet.

1. Workstation > **Edit > Virtual Network Editor** > **Change Settings** (droits admin)
2. Sélectionne **VMnet1 (Host-only)** :
   - Subnet IP : `192.168.56.0`, Subnet mask : `255.255.255.0`
   - **Décoche** « Use local DHCP service to distribute IP address to VMs » (IP statiques)
3. **Apply / OK**. Ton PC Windows reçoit l'IP `192.168.56.1` sur cette carte.

> Si tu préfères garder le sous-réseau proposé par VMware, adapte uniquement `ansible_host` et `lan_cidr` dans `ansible/inventory.ini` ainsi que les IP netplan. Les IP VPN (10.8.0.x) et `prometheus.yml` ne changent pas.

## Phase 2 — Créer la VM `monitoring`

1. **File > New Virtual Machine > Custom** > Workstation (dernière version)
2. Choisis **I will install the operating system later** (évite l'installation « Easy Install » qui pré-configure le système)
3. Guest OS : **Linux > Ubuntu 64-bit**, nom : `monitoring`
4. **2 processeurs, 4 Go RAM**, réseau **NAT**, disque **20 Go**
5. Avant de démarrer : **Edit virtual machine settings** :
   - **CD/DVD** > Use ISO image file > ISO Ubuntu Server 22.04 LTS
   - **Add > Network Adapter > Host-only** (2e carte)
6. Démarre et installe : coche **Install OpenSSH server**, crée l'utilisateur `sysadmin`, nom de machine `monitoring`.

## Phase 3 — Créer `node1`

**Option A (rapide) : clone.** Éteins `monitoring` (avant tout playbook), puis clic droit > **Manage > Clone > Full clone** > nom `node1`. Sur `node1`, régénère les identifiants uniques :
```bash
sudo hostnamectl set-hostname node1
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server
```
Baisse la RAM de `node1` à 2 Go si besoin.

**Option B :** répète la phase 2 (2 Go RAM, nom `node1`).

## Phase 4 — IP statiques sur la carte host-only

Sur chaque VM, identifie la 2e carte (celle **sans IPv4**, DHCP étant désactivé sur VMnet1) :
```bash
ip -br a
ls /etc/netplan
```
Les noms VMware sont souvent `ens33` (NAT) et `ens37` ou `ens38` (host-only). Édite le fichier netplan existant :
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
    ens37:                          # adapte au nom relevé
      addresses: [192.168.56.10/24] # .11 sur node1
```
```bash
sudo netplan apply
ip -br a
```
Tests : depuis `monitoring`, `ping 192.168.56.11` ; depuis PowerShell Windows, `ping 192.168.56.10`.

## Phase 5 — Ansible et clés SSH (sur `monitoring`)

```bash
sudo apt update && sudo apt install -y ansible
ssh-keygen -t ed25519
ssh-copy-id sysadmin@192.168.56.10
ssh-copy-id sysadmin@192.168.56.11
```

## Phase 6 — Envoyer le projet sur `monitoring`

Depuis **PowerShell** sur Windows (dans le dossier qui contient le projet décompressé) :
```powershell
scp -r .\monitoring-sysadmin-project sysadmin@192.168.56.10:~/
```
Puis sur `monitoring` :
```bash
cd ~/monitoring-sysadmin-project/ansible
ansible all -m ping
```
Attendu : `pong` pour `monitoring` et `node1`. **Ne continue pas sans ce résultat** (le durcissement désactive les mots de passe SSH).

**Prends un snapshot VMware des deux VMs** (VM > Snapshot > Take Snapshot, nom `avant-durcissement`) : en cas d'erreur, tu reviens en arrière en 1 clic.

## Phase 7 — Exécuter les playbooks (dans l'ordre)

```bash
ansible-playbook 01-hardening.yml -K    # mises à jour, SSH, pare-feu UFW
ansible-playbook 02-wireguard.yml -K    # clés + tunnel VPN
ansible-playbook 03-monitoring.yml -K   # Node Exporter, Docker, Prometheus, Grafana
```
`-K` demande le mot de passe sudo. Le dernier playbook demande aussi le mot de passe admin Grafana.

## Phase 8 — Vérifications

**Tunnel VPN** (sur `monitoring`) :
```bash
sudo wg show                 # "latest handshake" doit apparaître
ping -c 3 10.8.0.2
```
**Prometheus** (local uniquement, par sécurité) — depuis PowerShell :
```powershell
ssh -L 9090:localhost:9090 sysadmin@192.168.56.10
```
puis ouvre `http://localhost:9090/targets` : les 3 cibles doivent être **UP**.

**Grafana** : `http://192.168.56.10:3000` (admin + ton mot de passe). La source Prometheus est déjà configurée. **Dashboards > Import**, ID **1860**, source Prometheus, instance `node1`.

## Phase 9 — Tests de sécurité (à documenter avec captures)

1. **Le trafic passe par le VPN** : sur `monitoring`, `sudo systemctl stop wg-quick@wg0` → la cible `node1` passe **DOWN** ; `sudo systemctl start wg-quick@wg0` → elle repasse **UP**.
2. **Node Exporter absent du LAN** : depuis PowerShell, `Test-NetConnection 192.168.56.11 -Port 9100` doit afficher `TcpTestSucceeded : False`.
3. **Pare-feu** : `sudo ufw status verbose` sur chaque VM.
4. **SSH sans mot de passe** : `ssh -o PubkeyAuthentication=no sysadmin@192.168.56.11` doit être refusé.
5. **Conteneur non-root** : `docker exec prometheus id` → uid 65534.

## Phase 10 — Publier sur GitHub

Sur `monitoring` :
```bash
cd ~/monitoring-sysadmin-project
git init && git add . && git commit -m "Infrastructure de supervision sécurisée"
```
Crée le dépôt sur GitHub, puis `git remote add origin <url>` et `git push -u origin main`. Le `.gitignore` exclut déjà `.env` et les clés. Ajoute un dossier `docs/` avec tes captures (cibles UP, dashboard Grafana, test de coupure du VPN).

Le `README.md` doit décrire **uniquement ce que tu as réellement réalisé**. Envoie-moi tes captures et sorties de commandes, je le rédigerai avec toi.

---

## Limites connues (à mentionner dans le README)
- Docker publie le port 3000 en contournant UFW ; Grafana reste exposé au réseau host-only, acceptable en lab, mais à placer derrière un proxy HTTPS en production.
- Pas d'alertes ni de proxy Nginx dans cette version.

## Pour aller plus loin
Alertmanager (alertes disque/CPU), Nginx + certificat auto-signé devant Grafana, un 2e nœud dans `[managed_nodes]` (ligne avec `wg_ip=10.8.0.3` + cible dans `prometheus.yml`).
