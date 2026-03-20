# Déploiement de l'utilisateur ansible-rex — Satellite 6.18 REX

Ce playbook configure l'utilisateur `ansible-rex` requis par la fonctionnalité
**Remote Execution (REX)** de Red Hat Satellite 6.18 sur les hôtes gérés.

**Compatibilité :** RHEL 7 / 8 / 9 / 10 · AlmaLinux 8 / 9 · Rocky Linux 8 / 9

---

## Structure du projet

```
Satellite/
├── ansible.cfg                  # Configuration Ansible locale
├── setup_ansible_rex.yml        # Playbook principal
├── group_vars/
│   └── all.yml                  # Variables globales (à adapter)
└── inventory/
    └── hosts.example            # Exemple d'inventaire
```

---

## Prérequis

| Prérequis | Détail |
|---|---|
| Ansible | ≥ 2.9 (≥ 2.14 recommandé pour RHEL 9/10) |
| Collection `ansible.posix` | `ansible-galaxy collection install ansible.posix` |
| Accès root ou sudo | Sur chaque hôte cible |
| Clé publique foreman-proxy | Présente sur le Satellite/Capsule |

---

## Utilisation rapide

```bash
# 1. Copier et adapter l'inventaire
cp inventory/hosts.example inventory/hosts
vi inventory/hosts

# 2. Adapter les variables si besoin
vi group_vars/all.yml

# 3. Lancer le playbook (depuis le Satellite, lecture automatique de la pubkey)
ansible-playbook -i inventory/hosts setup_ansible_rex.yml

# 4. Cibler un sous-groupe uniquement
ansible-playbook -i inventory/hosts setup_ansible_rex.yml --limit rhel9

# 5. Vérifier sans appliquer (dry-run)
ansible-playbook -i inventory/hosts setup_ansible_rex.yml --check --diff
```

---

## Variables — `group_vars/all.yml`

Ce fichier est le point central de configuration. Toutes les valeurs peuvent
être surchargées en ligne de commande avec `-e nom=valeur`.

### Identité de l'utilisateur REX

| Variable | Valeur par défaut | Description |
|---|---|---|
| `rex_user` | `ansible-rex` | Nom du compte système créé sur les hôtes |
| `rex_home` | `/home/ansible-rex` | Répertoire home de l'utilisateur |
| `rex_shell` | `/bin/bash` | Shell assigné au compte (`/sbin/nologin` possible si l'agent REX ne nécessite pas de shell interactif, mais `/bin/bash` est requis par Satellite REX) |

> **Note `rex_shell`** : Red Hat Satellite REX requiert `/bin/bash`. Ne pas utiliser
> `/sbin/nologin` ou `/bin/false` : l'exécution distante des jobs échouerait.

### Clé publique SSH (foreman-proxy)

Deux modes exclusifs :

#### Mode 1 — Lecture automatique depuis le Satellite/Capsule *(recommandé)*

Le playbook se connecte à `satellite_host` via `delegate_to` et lit la clé
directement sur le système de fichiers.

| Variable | Valeur par défaut | Description |
|---|---|---|
| `satellite_host` | `localhost` | Hôte depuis lequel lire la clé publique. Mettre `localhost` si le playbook tourne **sur** le Satellite, sinon le FQDN de la Capsule ou du Satellite |
| `satellite_pubkey_path` | `/usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub` | Chemin absolu de la clé publique sur `satellite_host`. Ne changer que si votre Capsule utilise un chemin non standard |

```yaml
# group_vars/all.yml — Mode 1 (par défaut)
satellite_host: localhost
satellite_pubkey_path: /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub
```

#### Mode 2 — Clé fournie directement en variable

Utile en pipeline CI/CD, ou lorsque le playbook ne peut pas atteindre le
Satellite par SSH. **Protéger ce fichier avec Ansible Vault.**

| Variable | Valeur par défaut | Description |
|---|---|---|
| `rex_pubkey` | *(non défini)* | Contenu brut de la clé publique SSH (`ssh-rsa AAAA…`). Dès qu'elle est définie, la lecture distante est ignorée |

```yaml
# group_vars/all.yml — Mode 2
rex_pubkey: "ssh-rsa AAAAB3NzaC1yc2EAAA... foreman-proxy@satellite.example.com"
```

Ou en ligne de commande (pour les tests) :

```bash
ansible-playbook -i inventory/hosts setup_ansible_rex.yml \
  -e 'rex_pubkey="ssh-rsa AAAAB3Nza... foreman-proxy@satellite"'
```

---

## Configuration — `ansible.cfg`

Ce fichier configure le comportement global d'Ansible pour ce projet.
Il est lu automatiquement depuis le répertoire courant.

### Section `[defaults]`

| Paramètre | Valeur actuelle | Description |
|---|---|---|
| `inventory` | `inventory/hosts.example` | Inventaire par défaut. **Modifier** pour pointer vers votre inventaire réel (`inventory/hosts`) |
| `remote_user` | `root` | Utilisateur SSH utilisé pour se connecter aux hôtes cibles. Remplacer par un compte sudoer si root est désactivé (ex. : `admin`, `rhel`) |
| `become` | `true` | Active l'élévation de privilèges (sudo) automatiquement |
| `become_method` | `sudo` | Méthode d'élévation. `sudo` dans la quasi-totalité des cas |
| `host_key_checking` | `false` | Désactive la vérification des clés SSH hôtes. Mettre `true` en production si les hôtes sont connus de `~/.ssh/known_hosts` |
| `gathering` | `smart` | Collecte les facts uniquement si l'hôte n'est pas déjà dans le cache |
| `fact_caching` | `jsonfile` | Active le cache des facts Ansible sur disque |
| `fact_caching_connection` | `/tmp/ansible_facts_cache` | Répertoire de stockage du cache. Modifier si `/tmp` n'est pas adapté |
| `fact_caching_timeout` | `3600` | Durée de validité du cache en secondes (1 heure). Augmenter pour des parcs stables |

### Section `[privilege_escalation]`

| Paramètre | Valeur actuelle | Description |
|---|---|---|
| `become_user` | `root` | Utilisateur cible après élévation |
| `become_ask_pass` | `false` | Ne pas demander le mot de passe sudo. Nécessite que `NOPASSWD` soit configuré pour `remote_user` |

### Section `[ssh_connection]`

| Paramètre | Valeur actuelle | Description |
|---|---|---|
| `pipelining` | `true` | Envoie plusieurs commandes dans la même connexion SSH : **performances nettement améliorées**. Incompatible avec `requiretty` dans sudoers (déjà désactivé via `Defaults:ansible-rex !requiretty`) |
| `ssh_args` | `-o ControlMaster=auto -o ControlPersist=60s` | Multiplexage SSH : réutilise les connexions pendant 60 secondes, réduit la latence sur les grands parcs |

---

## Inventaire — `inventory/hosts.example`

Adapter ce fichier à votre environnement réel.

### Structure recommandée

```ini
[rhel7]
rhel7-client01.example.com

[rhel8]
rhel8-client01.example.com
almalinux8-srv01.example.com
rocky8-srv01.example.com

[rhel9]
rhel9-client01.example.com
almalinux9-srv01.example.com
rocky9-srv01.example.com

[rhel10]
rhel10-client01.example.com

[rex_targets:children]
rhel7
rhel8
rhel9
rhel10

[rex_targets:vars]
ansible_user=root
# satellite_host=capsule.example.com
```

### Variables d'inventaire disponibles

| Variable | Exemple | Description |
|---|---|---|
| `ansible_user` | `root` | Utilisateur SSH de connexion pour ce groupe d'hôtes. Surcharge `remote_user` de `ansible.cfg` |
| `ansible_ssh_private_key_file` | `~/.ssh/id_rsa` | Clé privée SSH à utiliser si différente de la clé par défaut |
| `ansible_port` | `22` | Port SSH si non standard |
| `satellite_host` | `capsule.example.com` | Surcharge par groupe : utile si différents groupes sont rattachés à des Capsules différentes |
| `satellite_pubkey_path` | `/usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub` | Surcharge du chemin de clé par groupe |

---

## Cas d'usage fréquents

### Capsule multi-sites : clé différente par groupe

```ini
# inventory/hosts
[site_paris]
srv-paris-[01:10].example.com

[site_paris:vars]
satellite_host=capsule-paris.example.com

[site_lyon]
srv-lyon-[01:05].example.com

[site_lyon:vars]
satellite_host=capsule-lyon.example.com
```

### Connexion initiale avec un compte sudoer (root SSH désactivé)

```ini
# inventory/hosts
[rex_targets:vars]
ansible_user=admin
```

```ini
# ansible.cfg
[defaults]
remote_user = admin

[privilege_escalation]
become          = true
become_ask_pass = false
```

### Protéger la clé publique avec Ansible Vault

```bash
# Chiffrer la valeur
ansible-vault encrypt_string 'ssh-rsa AAAA... foreman-proxy@satellite' \
  --name rex_pubkey >> group_vars/all.yml

# Lancer le playbook avec le vault
ansible-playbook -i inventory/hosts setup_ansible_rex.yml --ask-vault-pass
# ou avec un fichier de mot de passe
ansible-playbook -i inventory/hosts setup_ansible_rex.yml --vault-password-file ~/.vault_pass
```

---

## Ce que fait le playbook (résumé)

| Étape | Action |
|---|---|
| **PRE** | Vérifie que l'OS est de la famille RedHat v7–10 |
| **Bloc 1** | Crée le compte système `ansible-rex` avec home et `/bin/bash` |
| **Bloc 2** | Désactive le mot de passe et toute politique d'expiration initiale |
| **Bloc 3** | Déploie `/etc/sudoers.d/ansible-rex` validé par `visudo` (voir détail ci-dessous) |
| **Bloc 4** | Crée `~/.ssh/`, lit la pubkey depuis le Satellite/Capsule, déploie `authorized_keys` avec les bonnes permissions |
| **Bloc 5** | Verrouille le compte (`usermod -L`) et applique les paramètres anti-expiration PAM finaux |
| **Bloc 6** | Vérifie et affiche un rapport : statut LK, params chage, permissions SSH |

---

## Détail du fichier sudoers — `/etc/sudoers.d/ansible-rex`

Le playbook déploie un fichier drop-in sudoers dédié à l'utilisateur `ansible-rex`.
Ce fichier est **validé automatiquement par `visudo -cf`** avant écriture afin d'éviter
toute erreur de syntaxe qui pourrait bloquer l'ensemble de sudo sur l'hôte.

### Contenu déployé

```sudoers
# Managed by Satellite — Remote Execution user
# Do NOT edit manually
ansible-rex ALL=(ALL) NOPASSWD: ALL
Defaults:ansible-rex !requiretty
```

### Explication des directives

| Directive | Signification |
|---|---|
| `ansible-rex ALL=(ALL) NOPASSWD: ALL` | Autorise l'utilisateur `ansible-rex` à exécuter **n'importe quelle commande** en tant que **n'importe quel utilisateur** (`root` inclus) **sans mot de passe**. C'est indispensable car le compte est verrouillé (pas de mot de passe) et Satellite REX doit pouvoir exécuter des tâches d'administration à distance |
| `Defaults:ansible-rex !requiretty` | Désactive l'exigence d'un terminal (TTY) pour les commandes sudo de cet utilisateur. Sans cette directive, les jobs REX lancés via SSH sans TTY alloué échoueraient. Cette option est aussi requise pour le **pipelining** Ansible (paramètre `pipelining = true` dans `ansible.cfg`) |

### Sécurité

| Mesure | Détail |
|---|---|
| **Fichier isolé** | Déployé dans `/etc/sudoers.d/` (un fichier par usage), sans toucher au fichier `/etc/sudoers` principal |
| **Permissions** | `root:root 0440` — lecture seule pour root et le groupe root, aucun accès aux autres utilisateurs |
| **Validation** | `visudo -cf %s` vérifie la syntaxe sur un fichier temporaire **avant** de l'écrire à sa destination finale. En cas d'erreur de syntaxe, la tâche échoue et le fichier existant reste inchangé |
| **Accès limité au compte** | Bien que les droits sudo soient larges (`ALL`), le compte `ansible-rex` est lui-même verrouillé (pas de mot de passe, pas de login interactif). Seule une connexion SSH par clé publique (celle du foreman-proxy) peut l'utiliser |

---

## Résultat attendu à la fin du playbook

```
RAPPORT | Afficher le résumé de configuration
  Hôte : rhel9-client01.example.com
  OS   : RedHat 9

  ► Statut passwd (doit contenir 'LK') :
      ansible-rex LK ...

  ► Paramètres chage :
      Last password change                               : never
      Password expires                                   : never
      Password inactive                                  : never
      Account expires                                    : never
      Minimum number of days between password change     : 0
      Maximum number of days between password change     : 99999
      Number of days of warning before password expires  : 0

  ► Permissions .ssh    : 0700  (attendu: 0700)
  ► Permissions authkeys: 0600  (attendu: 0600)
```
