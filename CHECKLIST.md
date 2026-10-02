# Checklist de validation — une phase n'est réussie que si son critère est vérifié

Ne passe jamais à la phase suivante sans avoir validé le critère de la phase en cours : les erreurs s'accumulent sinon et deviennent difficiles à localiser.

## Phase 1 — Réseau host-only VMware
**Critère de réussite :**
```
Virtual Network Editor > VMnet1 : Subnet 192.168.56.0, DHCP décoché
```
**Erreur fréquente :** VMnet1 absent de la liste → clique **Add Network**, choisis **Host-only**, puis configure le sous-réseau.

## Phase 2 & 3 — Création des VMs
**Critère de réussite :** les deux VMs démarrent, tu peux te connecter en local (écran VMware) avec `sysadmin` + mot de passe.
```bash
whoami        # doit renvoyer sysadmin
hostname      # doit renvoyer monitoring (ou node1)
```
**Erreur fréquente (clonage) :** les deux VMs partagent la même IP/clé SSH si tu oublies l'étape `machine-id` + `ssh_host_*` → `ssh` refuse ou mélange les hôtes. Reprends la Phase 3, option A.

## Phase 4 — IP statiques
**Critère de réussite :**
```bash
ip -br a
# la 2e carte affiche bien 192.168.56.10/24 (ou .11)
ping -c 2 192.168.56.11    # depuis monitoring
```
**Erreurs fréquentes :**
- `netplan apply` sans erreur mais pas d'IP → mauvais nom de carte, revérifie avec `ip -br a` avant modification.
- Erreur YAML (`netplan apply` échoue) → l'indentation est en **espaces**, jamais en tabulations.

## Phase 5 — Ansible et SSH
**Critère de réussite :**
```bash
ssh sysadmin@192.168.56.11   # doit se connecter SANS mot de passe
```
**Erreur fréquente :** `ssh-copy-id` demande encore un mot de passe à chaque connexion → la clé n'a pas été copiée (vérifie `~/.ssh/authorized_keys` sur la cible) ou mauvais utilisateur/IP.

## Phase 6 — Transfert et ping Ansible
**Critère de réussite :**
```bash
ansible all -m ping
# monitoring | SUCCESS => {"ping": "pong"}
# node1      | SUCCESS => {"ping": "pong"}
```
**Erreur fréquente :** `UNREACHABLE` → vérifie `inventory.ini` (bonnes IP), et que `ssh sysadmin@<ip>` fonctionne manuellement d'abord.

## Phase 7 — Playbooks
**Critère de réussite après chaque playbook :** la ligne de fin affiche `failed=0`.
```
PLAY RECAP *********************************************************
monitoring : ok=12  changed=8  unreachable=0  failed=0
node1      : ok=9   changed=6  unreachable=0  failed=0
```
**Erreurs fréquentes :**
- `01-hardening.yml` échoue sur `community.general.ufw` → le collection n'est pas installée : `ansible-galaxy collection install community.general`
- `02-wireguard.yml` : la clé publique d'un hôte reste vide dans le template → relance le playbook une 2e fois (les faits `wg_public_key` doivent être calculés sur **tous** les hôtes avant que le template des autres ne s'exécute ; ce playbook cible `hosts: all`, donc c'est normal si le premier passage échoue partiellement sur le template).
- `03-monitoring.yml` : `docker-compose: command not found` → reconnecte-toi en SSH pour recharger le groupe `docker`, ou ajoute `become: true` (déjà présent dans le playbook fourni).

## Phase 8 — Vérifications fonctionnelles
**Critères de réussite (les 3 doivent être vrais) :**
1. `sudo wg show` affiche un `latest handshake` récent (< 3 min)
2. `http://localhost:9090/targets` (via le tunnel SSH) : les 3 cibles en vert **UP**
3. Grafana affiche des courbes non plates sur le dashboard 1860

**Erreur fréquente :** cible `node1` en `DOWN` dans Prometheus mais `wg show` montre un handshake → vérifie que Node Exporter écoute bien sur `10.8.0.2:9100` et non `127.0.0.1` :
```bash
sudo ss -tlnp | grep 9100
```

## Phase 9 — Tests de sécurité
**Critère de réussite : les 5 tests donnent le résultat attendu, capture d'écran à l'appui.** Si un test échoue (ex. le port 9100 répond depuis le LAN), c'est une vraie faille à corriger avant de documenter le projet comme "sécurisé".

## Phase 10 — Publication GitHub
**Critère de réussite :**
```bash
git log --oneline        # au moins 1 commit
git status                # "nothing to commit, working tree clean"
```
Vérifie sur GitHub que `.env` et les fichiers `*.key` n'apparaissent **pas** dans le dépôt (le `.gitignore` doit les avoir exclus) — sinon supprime-les de l'historique avant de rendre le dépôt public.

---

## Si tu bloques
Colle-moi : la commande exacte tapée, le message d'erreur complet, et la phase concernée. Je corrigerai le fichier en cause plutôt que de deviner.
