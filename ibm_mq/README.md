# Rôle Ansible — `ibm_mq`

Installation relocalisée et industrialisée d'**IBM MQ 9.4** sur **RHEL 8 / 9** :
idempotente, sécurisée (TLS + checksum), compatible multi-versions, prête pour
la création ultérieure de Queue Managers.

> Ce rôle couvre **l'installation et la préparation**. La création des Queue
> Managers (`crtmqm`, paramètres de logs `lp/ls/lf`, démarrage systemd) relève
> d'un rôle distinct `ibm_mq_qmgr`, par souci de responsabilité unique.

## Périmètre

- Détection idempotente de l'état (`package_facts`).
- Création des utilisateurs/groupes `mqm` et `admmq` (UID/GID fixes).
- Prérequis système : dépendances OS, `ulimits`, contextes SELinux.
- Téléchargement sécurisé depuis Nexus (TLS activé + vérification SHA-256).
- Installation relocalisée via `rpm --prefix` (composants paramétrables).
- Installation primaire optionnelle (`setmqinst`) + lien `WebSphere-MQ-latest`.
- Configuration de l'environnement (`setmqenv -s`) par template.
- Nettoyage sûr des fichiers temporaires.

## Prérequis

```bash
ansible-galaxy collection install -r roles/ibm_mq/requirements.yml
```

- Ansible >= 2.14
- Collections : `ansible.posix`, `community.general`
- Accès HTTPS au dépôt Nexus avec sa CA dans le magasin de confiance RHEL
  (`/etc/pki/ca-trust/source/anchors/` + `update-ca-trust`).

## Variables principales (`defaults/main.yml`)

| Variable | Défaut | Rôle |
|----------|--------|------|
| `ibm_mq_version` | `9.4.0.5` | Version cible |
| `ibm_mq_fs_root` | `/data` | Racine FS dédiée MQ |
| `ibm_mq_install_dir` | `…/WebSphere-MQ-<version>` | Emplacement relocalisé |
| `ibm_mq_symlink_latest` | `…/WebSphere-MQ-latest` | Lien de bascule de version |
| `ibm_mq_nexus_base_url` | URL Nexus | Source des binaires |
| `ibm_mq_archive_sha256` | *(à renseigner)* | **SHA-256 obligatoire** |
| `ibm_mq_components` | Runtime/Server/GSKit/Client/Msg_fr | Composants installés |
| `ibm_mq_mqm_uid/gid`, `ibm_mq_admmq_uid` | 1010/1010/1011 | Identifiants fixes |
| `ibm_mq_set_primary` | `false` | `setmqinst -i -p` (multi-versions = false) |
| `ibm_mq_nofile_*`, `ibm_mq_nproc_*` | 10240 / 4096 | Limites système mqm |
| `ibm_mq_manage_selinux` | `true` | Gestion des contextes SELinux |
| `ibm_mq_cleanup` | `true` | Suppression du tmp d'install |

> `ibm_mq_archive_sha256` **doit** être renseigné (le rôle échoue sinon, par
> design). Récupérez-le depuis votre pipeline Nexus / la publication IBM.

## Exemple d'utilisation

```yaml
- hosts: mq_servers
  become: true
  roles:
    - role: ibm_mq
      vars:
        ibm_mq_version: "9.4.0.5"
        ibm_mq_archive_sha256: "a1b2c3...<64 hex>"
        ibm_mq_set_primary: false
```

## Tags

`ibm_mq_assert`, `ibm_mq_preflight`, `ibm_mq_users`, `ibm_mq_system`,
`ibm_mq_directories`, `ibm_mq_download`, `ibm_mq_install`, `ibm_mq_primary`,
`ibm_mq_environment`, `ibm_mq_cleanup`.

```bash
# Reconfigurer uniquement l'environnement utilisateur :
ansible-playbook site.yml --tags ibm_mq_environment
```

## Tests (Molecule)

```bash
cd roles/ibm_mq
molecule test           # syntaxe + convergence + idempotence + verify (RHEL 8 & 9)
molecule converge       # première application
molecule idempotence    # doit retourner changed=0
molecule verify         # assertions post-installation
```

> Les images UBI en conteneur ne permettent pas de valider SELinux/ulimits de
> façon représentative ; `ibm_mq_manage_selinux` est désactivé dans le scénario
> par défaut. Pour un test d'installation complet, fournissez un binaire et un
> SHA-256 réels.

## Montée de version (multi-versions)

1. Installer la nouvelle version dans une arborescence `--prefix` distincte.
2. Valider (`dspmqver`).
3. Basculer le lien `WebSphere-MQ-latest` (tâche `primary.yml`).
4. Rollback = repointer le lien vers l'ancienne installation.

Aucune modification des profils n'est nécessaire : ils pointent vers le lien stable.
