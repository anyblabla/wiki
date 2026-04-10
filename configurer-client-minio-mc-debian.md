---
title: Configurer le client MinIO (mc) sous Linux
description: Apprenez à installer et configurer le client MinIO (mc) sur Debian. Ce guide détaille la création d'alias, la gestion des buckets et l'automatisation de vos sauvegardes via la ligne de commande.
published: true
date: 2026-04-10T21:34:26.585Z
tags: debian, stockage, cli, backup, minio, s3, objet
editor: markdown
dateCreated: 2026-04-10T21:34:26.585Z
---

Ce guide détaille l’installation et l’utilisation du client **MinIO (mc)**. Cet outil en ligne de commande permet de gérer votre stockage objet (S3) aussi facilement que vos fichiers locaux.

---

## 🏗️ 1. Installation du binaire

Le client MinIO est un binaire statique. Il ne s’installe pas via les dépôts classiques (`apt`), mais se télécharge directement.

1.  **Téléchargement** :
    ```bash
    wget https://dl.min.io/client/mc/release/linux-amd64/mc
    ```
2.  **Permissions d’exécution** :
    ```bash
    chmod +x mc
    ```
3.  **Installation globale** :
    Pour éviter tout conflit avec l’utilitaire *Midnight Commander* (qui utilise aussi la commande `mc`), nous le renommons `myminio` :
    ```bash
    sudo mv mc /usr/local/bin/myminio
    ```

---

## 🔐 2. Configuration de l’accès (alias)

Pour vous connecter à votre serveur, vous devez créer un **alias**. C’est un profil qui enregistre l’adresse et vos clés de sécurité.

**Commande générique :**
```bash
myminio alias set NOM_DE_VOTRE_PROFIL URL_DU_SERVEUR CLE_ACCES CLE_SECRETE
```

**Exemple concret à adapter :**
* **NOM_DE_VOTRE_PROFIL** : Un nom court (ex : `mon-cloud`)
* **URL_DU_SERVEUR** : L’adresse de votre instance (ex : `https://minio.exemple.com`)
* **CLE_ACCES** : Votre *Access key*
* **CLE_SECRETE** : Votre *Secret key*

> [!IMPORTANT]
> Cette configuration est stockée localement dans `~/.myminio/config.json`. Ne partagez jamais ce fichier.

---

## 📂 3. Commandes essentielles

Une fois l’alias configuré (appelons-le `mon-cloud`), vous pouvez utiliser ces commandes :

* **Lister les buckets** : `myminio ls mon-cloud/`
* **Créer un bucket** : `myminio mb mon-cloud/nom-du-bucket`
* **Envoyer un fichier** : `myminio cp mon-fichier.zip mon-cloud/nom-du-bucket/`
* **Synchroniser un dossier local vers le cloud** :
  ```bash
  myminio mirror ~/Documents mon-cloud/nom-du-bucket/documents
  ```

---

## ⚡ 4. Astuce : créer des raccourcis (alias bash)

Pour automatiser vos sauvegardes, vous pouvez ajouter des alias dans votre fichier `~/.bash_aliases` ou `~/.bashrc`.

**Exemples de raccourcis :**
```bash
# Lister rapidement un bucket spécifique
alias ls-cloud='myminio ls mon-cloud/nom-du-bucket/'

# Sauvegarde rapide du dossier Scripts
alias bkp-scripts='myminio mirror ~/Scripts mon-cloud/backup-bucket/scripts'
```

---

## ❓ Pourquoi préférer le client « mc » ?

* **Vitesse** : Bien plus rapide que l’interface web pour les gros transferts.
* **Automatisation** : Utilisable dans des scripts de sauvegarde (Cron).
* **Reprise** : En cas de coupure réseau, le client peut reprendre les transferts là où ils s’étaient arrêtés.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l’ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).