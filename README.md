# Satellite — Playbooks Ansible de maintenance

Playbooks Ansible pour la gestion et la maintenance des hôtes enregistrés
auprès de **Red Hat Satellite 6.11+** (jusqu'à 6.18).

**Compatibilite :** RHEL 7 / 8 / 9 / 10 · AlmaLinux 8 / 9 · Rocky Linux 8 / 9

---

## Structure du projet

```
Satellite/
├── ansible.cfg                  # Configuration Ansible locale
├── setup_ansible_rex.yml        # Playbook : utilisateur REX
├── update_satellite_ca.yml      # Playbook : certificat CA Satellite
├── group_vars/
│   └── all.yml                  # Variables globales (à adapter)
├── inventory/
│   └── hosts.example            # Exemple d'inventaire
└── README.md
```

---

## Prerequis

| Prerequis | Detail |
|---|---|
| Ansible | >= 2.9 (>= 2.14 recommande pour RHEL 9/10) |
| Collection `ansible.posix` | `ansible-galaxy collection install ansible.posix` |
| Acces root ou sudo | Sur chaque hote cible |
| Cle publique foreman-proxy | Presente sur le Satellite/Capsule (pour `setup_ansible_rex.yml`) |
| Connectivite HTTPS | Vers le Satellite/Capsule (pour `update_satellite_ca.yml`) |

---

## Mise en place

```bash
# 1. Copier et adapter l'inventaire
cp inventory/hosts.example inventory/hosts
vi inventory/hosts

# 2. Adapter les variables globales
vi group_vars/all.yml

# 3. Verifier sans appliquer (dry-run)
ansible-playbook -i inventory/hosts <playbook>.yml --check --diff
```

---

## Playbooks

### 1. `setup_ansible_rex.yml` — Deploiement de l'utilisateur REX

Configure l'utilisateur `ansible-rex` requis par la fonctionnalite
**Remote Execution (REX)** de Red Hat Satellite sur les hotes geres.

#### Ce que fait le playbook

| Etape | Action | Condition |
|---|---|---|
| **PRE** | Verifie que l'OS est de la famille RedHat v7-10 | Toujours |
| **Bloc 1** | Cree le compte systeme `ansible-rex` (verrouille, sans mot de passe) | Seulement si l'utilisateur n'existe pas |
| **Bloc 3** | Deploie `/etc/sudoers.d/ansible-rex` valide par `visudo` | Idempotent |
| **Bloc 4** | Cree `~/.ssh/`, lit la pubkey depuis le Satellite/Capsule, deploie `authorized_keys` | Seulement si la cle n'est pas deja presente |
| **Bloc 5** | Applique les parametres anti-expiration PAM (`chage`) | Toujours |
| **Bloc 6** | Verifie et affiche un rapport : statut LK, params chage, permissions SSH | Toujours |

#### Utilisation

```bash
# Execution standard
ansible-playbook -i inventory/hosts setup_ansible_rex.yml

# Cibler un sous-groupe
ansible-playbook -i inventory/hosts setup_ansible_rex.yml --limit rhel9

# Fournir la cle publique directement
ansible-playbook -i inventory/hosts setup_ansible_rex.yml \
  -e 'rex_pubkey="ssh-rsa AAAAB3Nza... foreman-proxy@satellite"'
```

#### Variables

| Variable | Defaut | Description |
|---|---|---|
| `rex_user` | `ansible-rex` | Nom du compte systeme |
| `rex_home` | `/home/ansible-rex` | Repertoire home |
| `rex_shell` | `/bin/bash` | Shell (requis par Satellite REX) |
| `satellite_host` | `localhost` | Hote depuis lequel lire la cle publique |
| `satellite_pubkey_path` | `/usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub` | Chemin de la cle publique |
| `rex_pubkey` | *(non defini)* | Cle publique fournie directement (prioritaire sur la lecture distante) |

#### Idempotence

Le playbook est concu pour etre rejoue sans effet de bord :

- **Utilisateur** : la creation est conditionnee par `getent passwd`. Si l'utilisateur existe deja, la tache est skippee.
- **Cle SSH** : le contenu de `authorized_keys` est lu et compare avec la cle attendue. Le deploiement n'a lieu que si la cle est absente ou differente.
- **Sudoers** : le module `copy` est nativement idempotent (comparaison de checksum).

#### Detail du fichier sudoers

```sudoers
# Managed by Satellite — Remote Execution user
# Do NOT edit manually
ansible-rex ALL=(ALL) NOPASSWD: ALL
Defaults:ansible-rex !requiretty
```

| Directive | Signification |
|---|---|
| `ansible-rex ALL=(ALL) NOPASSWD: ALL` | Autorise `ansible-rex` a executer toute commande en tant que tout utilisateur, sans mot de passe. Requis car le compte est verrouille. |
| `Defaults:ansible-rex !requiretty` | Desactive l'exigence de TTY pour sudo. Requis pour les jobs REX via SSH et le pipelining Ansible. |

#### Resultat attendu

```
RAPPORT | Afficher le resume de configuration
  Hote : rhel9-client01.example.com
  OS   : RedHat 9

  > Statut passwd (doit contenir 'LK') :
      ansible-rex LK ...

  > Parametres chage :
      Password expires     : never
      Account expires      : never

  > Permissions .ssh    : 0700  (attendu: 0700)
  > Permissions authkeys: 0600  (attendu: 0600)
```

---

### 2. `update_satellite_ca.yml` — Mise a jour du certificat CA Satellite

Verifie et met a jour le certificat CA du Satellite/Capsule sur les hotes
enregistres. Gere automatiquement les deux modes d'enregistrement.

#### Contexte

Lors d'un changement de certificat SSL sur le Satellite (renouvellement,
changement de CA, migration), les hotes enregistres ne peuvent plus
communiquer avec le Satellite si leur copie locale du CA n'est pas mise a jour.

Ce playbook detecte automatiquement le mode d'enregistrement de chaque hote
et applique la procedure de mise a jour appropriee.

#### Les deux modes d'enregistrement

##### Cas 1 : Global Registration (Satellite 6.11+)

- **Detection** : le RPM `katello-ca-consumer` n'est **pas** installe
- **Stockage du CA** : fichier unique `/etc/rhsm/ca/katello-server-ca.pem`
- **Enregistrement initial** : `curl -k 'https://satellite/register?token=...' | bash`
- **Mise a jour** : remplacement direct du fichier `.pem`

##### Cas 2 : katello-ca-consumer (workflow legacy)

- **Detection** : le RPM `katello-ca-consumer` **est** installe
- **Stockage du CA** : gere par le RPM dans `/etc/rhsm/ca/`
- **Enregistrement initial** : `rpm -Uvh https://satellite/pub/katello-ca-consumer-latest.noarch.rpm`
- **Mise a jour** : desinstallation de l'ancien RPM puis reinstallation depuis le Satellite

#### Ce que fait le playbook

| Etape | Action | Condition |
|---|---|---|
| **PRE** | Verifie que l'OS est de la famille RedHat v7-10 | Toujours |
| **Bloc 1** | Detecte le mode d'enregistrement (`global` ou `legacy`) via `rpm -q` | Toujours |
| **Bloc 2** | Telecharge le CA de reference depuis `https://satellite/pub/katello-server-ca.crt` | Une seule fois (`run_once`) |
| **Bloc 3** | Compare le CA local avec la reference (byte-a-byte) | Toujours |
| **Bloc 4** | **Global** : backup horodate de l'ancien CA + deploiement du nouveau | Seulement si different et mode `global` |
| **Bloc 5** | **Legacy** : `rpm -e katello-ca-consumer-*` + reinstallation du RPM | Seulement si different et mode `legacy` |
| **Bloc 6** | Verifie le CA deploye + teste `subscription-manager identity` | Toujours |

#### Utilisation

```bash
# Execution standard
ansible-playbook -i inventory/hosts update_satellite_ca.yml

# Cibler un sous-groupe
ansible-playbook -i inventory/hosts update_satellite_ca.yml --limit rhel9

# Surcharger le FQDN du Satellite
ansible-playbook -i inventory/hosts update_satellite_ca.yml \
  -e satellite_fqdn=satellite.prod.example.com

# Dry-run : verifier sans appliquer
ansible-playbook -i inventory/hosts update_satellite_ca.yml --check --diff
```

#### Variables

| Variable | Defaut | Description |
|---|---|---|
| `satellite_fqdn` | `satellite.example.com` | FQDN du Satellite ou de la Capsule |
| `ca_cert_local_path` | `/etc/rhsm/ca/katello-server-ca.pem` | Chemin du CA sur les hotes |
| `ca_cert_remote_url` | `https://{{ satellite_fqdn }}/pub/katello-server-ca.crt` | URL de telechargement du CA |
| `katello_consumer_rpm_url` | `https://{{ satellite_fqdn }}/pub/katello-ca-consumer-latest.noarch.rpm` | URL du RPM (mode legacy) |

#### Idempotence

- Le CA de reference est telecharge depuis le Satellite et compare avec le fichier local.
- Si les certificats sont **identiques**, toutes les taches de mise a jour sont **skippees**.
- En mode **global**, un backup horodate (`katello-server-ca.pem.bak.20260320T...`) est cree avant remplacement.
- En mode **legacy**, le RPM est desinstalle puis reinstalle pour garantir la coherence.

#### Resultat attendu

```
RAPPORT | Afficher le resume
  Hote     : rhel9-client01.example.com
  OS       : RedHat 9
  Mode     : GLOBAL
  Satellite: satellite.example.com

  > Mise a jour effectuee : NON (deja a jour)
  > RHSM identity :
      system identity: 12345678-abcd-...
      org ID: Default_Organization
```

---

## Variables globales — `group_vars/all.yml`

Ce fichier centralise toutes les variables. Toutes peuvent etre surchargees
en ligne de commande avec `-e nom=valeur`.

```yaml
# Identite de l'utilisateur REX
rex_user: ansible-rex
rex_home: /home/ansible-rex
rex_shell: /bin/bash

# Cle publique SSH (lecture automatique)
satellite_host: localhost
satellite_pubkey_path: /usr/share/foreman-proxy/.ssh/id_rsa_foreman_proxy.pub

# Certificat CA Satellite
satellite_fqdn: "satellite.example.com"
ca_cert_local_path: /etc/rhsm/ca/katello-server-ca.pem
```

---

## Configuration — `ansible.cfg`

| Section | Parametre | Valeur | Description |
|---|---|---|---|
| `[defaults]` | `inventory` | `inventory/hosts.example` | Inventaire par defaut |
| | `remote_user` | `root` | Utilisateur SSH |
| | `host_key_checking` | `false` | Desactive la verification des cles hotes |
| | `gathering` | `smart` | Cache des facts |
| `[ssh_connection]` | `pipelining` | `true` | Performances ameliorees |
| | `ssh_args` | `-o ControlMaster=auto -o ControlPersist=60s` | Multiplexage SSH |

---

## Inventaire — `inventory/hosts.example`

### Structure recommandee

```ini
[rhel7]
rhel7-client01.example.com

[rhel8]
rhel8-client01.example.com
almalinux8-srv01.example.com

[rhel9]
rhel9-client01.example.com
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
```

### Capsule multi-sites

```ini
[site_paris]
srv-paris-[01:10].example.com

[site_paris:vars]
satellite_host=capsule-paris.example.com
satellite_fqdn=capsule-paris.example.com

[site_lyon]
srv-lyon-[01:05].example.com

[site_lyon:vars]
satellite_host=capsule-lyon.example.com
satellite_fqdn=capsule-lyon.example.com
```

---

## Securite

| Mesure | Detail |
|---|---|
| **Compte verrouille** | `ansible-rex` n'a pas de mot de passe et est verrouille (`password_lock`). Seule l'authentification par cle SSH est possible. |
| **Sudoers isole** | Fichier `/etc/sudoers.d/ansible-rex` valide par `visudo` avant ecriture. Le fichier principal `/etc/sudoers` n'est jamais modifie. |
| **Permissions SSH** | `.ssh/` en `0700`, `authorized_keys` en `0600`. |
| **Anti-expiration PAM** | Parametres `chage` configures pour eviter le verrouillage par les politiques de mots de passe (RHEL 9+). |
| **no_log** | Les taches manipulant des cles ou certificats utilisent `no_log: true`. |
| **Backup CA** | En mode Global Registration, l'ancien certificat est sauvegarde avec horodatage avant remplacement. |
| **Validation HTTPS** | `validate_certs: false` uniquement pour le telechargement du CA (le CA local peut etre obsolete). |
| **Ansible Vault** | Utiliser Vault pour proteger `rex_pubkey` si fournie directement dans `group_vars/all.yml`. |

---

## Depannage

### Le playbook echoue sur "OS non supporte"

Verifier que l'hote cible est bien RHEL, AlmaLinux ou Rocky Linux en version 7, 8, 9 ou 10.

### La cle publique n'est pas trouvee

- Verifier que `satellite_pubkey_path` pointe vers le bon fichier sur le Satellite/Capsule.
- Si le playbook ne tourne pas sur le Satellite, configurer `satellite_host` avec le FQDN accessible.
- Alternative : fournir `rex_pubkey` directement.

### `subscription-manager identity` echoue apres mise a jour du CA

- Verifier que le FQDN dans `satellite_fqdn` correspond bien au certificat SSL du Satellite.
- Verifier que le fichier `/etc/rhsm/rhsm.conf` pointe vers le bon serveur.
- Tester manuellement : `openssl s_client -connect satellite.example.com:443 -CAfile /etc/rhsm/ca/katello-server-ca.pem`

### Mode legacy : le RPM ne s'installe pas

- Verifier la connectivite HTTPS vers le Satellite : `curl -k https://satellite.example.com/pub/`
- Verifier que le RPM est disponible : `curl -kI https://satellite.example.com/pub/katello-ca-consumer-latest.noarch.rpm`
