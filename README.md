# Déploiement Automatisé et Sécurisé d'une Infrastructure de Supervision (Monitoring)

## 📌 Description du projet

Ce projet déploie, sécurise et supervise une infrastructure composée de deux serveurs Linux (Ubuntu Server 22.04 LTS), en environnement virtualisé (VMware Workstation). Il a été réalisé dans le cadre de ma Licence en Administration Système et Réseau, avec l'objectif de démontrer des compétences concrètes en **automatisation (Ansible)**, **sécurisation réseau (VPN, pare-feu)** et **supervision applicative (Prometheus/Grafana)**, directement transférables à la gestion d'infrastructures de recherche ou d'entreprise.

Chaque étape du déploiement a été testée et validée manuellement, y compris les mécanismes de sécurité (voir [Tests de sécurité](#-tests-de-sécurité-réalisés)).

## 🛠️ Technologies utilisées

* **Virtualisation :** VMware Workstation (2 VMs Ubuntu Server 22.04 LTS)
* **Automatisation / IaC :** Ansible (inventaire multi-hôtes, playbooks de durcissement, VPN et supervision)
* **Sécurité réseau :** WireGuard (VPN chiffré entre les deux serveurs), UFW (pare-feu)
* **Conteneurisation :** Docker & Docker Compose
* **Supervision :** Prometheus (collecte des métriques) + Node Exporter (exportateur système) + Grafana (visualisation)

## 📐 Architecture

| VM | Rôle | IP LAN (host-only) | IP VPN |
|---|---|---|---|
| `monitoring` | Serveur de contrôle Ansible, serveur WireGuard, Prometheus, Grafana | 192.168.56.10 | 10.8.0.1 |
| `node1` | Nœud supervisé, client WireGuard, Node Exporter | 192.168.56.11 | 10.8.0.2 |

**Principe de sécurité central :** Node Exporter (le service qui expose les métriques système de `node1`) n'écoute **que sur l'IP du tunnel VPN** (`10.8.0.2`), jamais sur l'IP LAN. Concrètement, Prometheus ne peut collecter les métriques qu'à travers le tunnel chiffré WireGuard — un test de coupure du VPN (voir plus bas) le démontre.

## 🚀 Déploiement

### Prérequis
* VMware Workstation avec 2 VMs Ubuntu Server 22.04 LTS sur un réseau host-only commun
* Ansible installé sur la VM `monitoring` (poste de contrôle)
* Accès SSH par clé entre `monitoring` et `node1`

### Étapes

```bash
git clone https://github.com/veraa237/monitoring-sysadmin-project.git
cd monitoring-sysadmin-project/ansible
```

Adapter `inventory.ini` aux IP et utilisateurs réels des deux VMs, puis vérifier la connectivité Ansible :
```bash
ansible all -m ping
```

Exécuter les playbooks, dans l'ordre :
```bash
ansible-playbook 01-hardening.yml -K    # durcissement : mises à jour, SSH, pare-feu UFW
ansible-playbook 02-wireguard.yml -K    # génération des clés et activation du tunnel VPN
ansible-playbook 03-monitoring.yml -K   # Node Exporter, Docker, Prometheus, Grafana
```

> Note : si les deux hôtes de l'inventaire ont des mots de passe sudo différents, exécuter chaque playbook séparément avec `--limit <hôte>`.

Le détail complet, pas à pas, avec les critères de réussite de chaque étape et les erreurs fréquentes, est disponible dans [`GUIDE.md`](./GUIDE.md) et [`CHECKLIST.md`](./CHECKLIST.md).

## 📊 Résultat

Une fois déployée, l'interface Grafana (`http://<IP_monitoring>:3000`) affiche un dashboard en temps réel (modèle "Node Exporter Full", ID Grafana.com 1860) avec l'usage CPU, RAM, disque et réseau de `node1`, collecté exclusivement via le tunnel VPN.

## 🔒 Bonnes pratiques de sécurité implémentées

* **Durcissement SSH :** connexion root désactivée, authentification par mot de passe désactivée (clé uniquement), limite de tentatives
* **Pare-feu UFW :** tout le trafic entrant refusé par défaut ; seuls les ports strictement nécessaires sont ouverts, et uniquement depuis le sous-réseau local (SSH limité avec anti brute-force, WireGuard, Grafana)
* **Isolation du VPN :** Node Exporter sur `node1` n'écoute que sur l'IP du tunnel WireGuard, jamais sur le LAN
* **Conteneurs non-root :** Prometheus s'exécute avec un utilisateur non privilégié (`nobody`)
* **Secrets hors dépôt :** mot de passe Grafana et clés privées exclus du suivi Git via `.gitignore`

## ✅ Tests de sécurité réalisés

| Test | Méthode | Résultat |
|---|---|---|
| Le trafic passe bien par le VPN | Coupure du service `wg-quick@wg0` puis observation du dashboard Grafana | Les métriques passent immédiatement à "No data" / N/A, confirmant qu'aucune donnée ne transite en dehors du tunnel |
| Node Exporter invisible depuis le LAN | `Test-NetConnection 192.168.56.11 -Port 9100` depuis une machine du réseau local | `TcpTestSucceeded : False` — le port n'est accessible que via l'IP VPN |
| Pare-feu actif et restrictif | `sudo ufw status verbose` | Politique par défaut "deny (incoming)", seuls 22/tcp (limité), 51820/udp et 3000/tcp autorisés depuis le sous-réseau local |

## ⚠️ Limites connues

* Grafana (port 3000) reste exposé sur le réseau local sans HTTPS — acceptable en environnement de lab, mais nécessiterait un reverse proxy Nginx avec certificat TLS en production
* Pas d'alertes automatiques configurées (Alertmanager) dans cette version
* Infrastructure à 2 nœuds ; l'ajout d'un `node2` nécessite d'ajouter une entrée dans `inventory.ini` et dans `prometheus.yml`

## 👤 Auteure

**Wandji Djoukoue Priscille Verra** — Licence en Administration Système et Réseau, Institut Universitaire des Technologies
