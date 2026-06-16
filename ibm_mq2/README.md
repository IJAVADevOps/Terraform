# Rôle Ansible : ibm_mq

Installe **IBM MQ Server** sur Red Hat / RHEL (8 et 9) à partir des binaires
hébergés sur Nexus.

## Ce que fait le rôle

1. Vérifie l'OS et détecte une installation MQ existante (idempotence).
2. Crée le groupe `mqm` et les utilisateurs `mqm` / `admmq`.
3. Télécharge et décompresse l'archive depuis Nexus.
4. Accepte la licence puis installe les RPM avec `--prefix`.
5. Déclare l'installation comme primaire, crée le lien `WebSphere-MQ-latest`,
   les répertoires data/conf/SSL/logs et configure les `.profile`.

## Utilisation

```yaml
- hosts: serveurs_mq
  become: true
  roles:
    - role: ibm_mq
      vars:
        version_MQ: "9.4.0.5"
        fsmq: data
```

> Le rôle nécessite les privilèges root : exécutez-le avec `become: true`.

## Tags disponibles

`mq_users`, `mq_download`, `mq_install`, `mq_configure`

## Variables principales

Voir `defaults/main.yml`. Les plus utiles : `version_MQ`, `fsmq`,
`mq_nexus_base_url`, `mq_validate_certs`, `mq_users`.

## Corrections apportées par rapport à la version d'origine

- **Variable `fsmq` non définie** : elle était utilisée partout dans les tasks
  mais absente des vars. Elle est maintenant définie (`fsmq: data`) et tous les
  chemins en dérivent de façon cohérente.
- **Doublons** : `repInstall`/`repInstall_linux` et `repLOG`/`repLog`
  faisaient double emploi. Consolidés.
- **Détection d'installation** : `rpm -qa | grep ... | wc -l` avec
  `failed_when: rc == 2` était inopérant (le code retour venait de `wc`, pas de
  `grep`). Remplacé par `grep -c` avec `set -o pipefail` et un fait booléen
  `mq_already_installed`.
- **Idempotence** : `setmqinst` ne s'exécute plus que si l'installation n'est
  pas déjà primaire ; `unarchive` utilise `creates` ; le téléchargement/
  l'installation sont entièrement ignorés si MQ est déjà présent.
- **Bug de propriété** : le `.profile` de `admmq` était créé avec
  `owner: mqm`. Chaque profil appartient désormais au bon utilisateur.
- **Incohérence de chemin** : les `.profile` pointaient vers
  `/data/WebSphere-MQ-latest`, qui n'existait pas. Un lien symbolique
  `WebSphere-MQ-latest` vers le répertoire d'installation réel est maintenant
  créé.
- **`lineinfile` x5** remplacé par un seul `blockinfile` (bloc géré, ré-exécution
  propre).
- **Utilisateurs/groupes** `mqm` et `admmq` (référencés mais jamais créés) sont
  désormais provisionnés par le rôle.
- **Création des répertoires manquants** (tmp, conf, SSL, logs) avant usage.
- **Modules FQCN** (`ansible.builtin.*`) et liste de RPM factorisée.
