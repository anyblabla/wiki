---
title: Déploiement de PsiTransfer avec Docker Compose
description: Ce guide pratique explique comment déployer rapidement PsiTransfer (un outil de partage de fichiers self-hosted sécurisé) en utilisant une pile Docker (stack) dans Portainer à partir d'un fichier compose YAML.
published: true
date: 2025-10-28T13:23:08.242Z
tags: docker, portainer, psitransfer, share, file
editor: markdown
dateCreated: 2024-06-30T14:00:46.077Z
---

## 1\. Liens utiles

  - [Site officiel](https://psi.cx/2017/psitransfer/)
  - [Documentation (site officiel) installation](https://psi.cx/2017/psitransfer-installation/)
  - [Page de projet GitHub](https://github.com/psi-4ward/psitransfer)
  - [Page de projet DockerHub](https://hub.docker.com/r/psitrax/psitransfer)

-----

## 2\. Fichier `docker-compose.yml`

Ce fichier Compose est minimaliste et fonctionnel, idéal pour un déploiement rapide.

```yaml
name: psitransfer
services:
    psitransfer:
        ports:
            - 0.0.0.0:3000:3000
        environment:
            #- PSITRANSFER_ADMIN_PASS=blablalinux
        volumes:
            - /data:/data
        image: psitrax/psitransfer:latest
        restart: always
```

Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).

-----

## 3\. Personnalisation du Déploiement

### Ports d'accès

  - **`- 0.0.0.0:3000:3000`** : L'application utilise le port interne **3000**. Le premier `3000` (ou `0.0.0.0:3000`) correspond au port sur votre hôte. Vous pouvez le modifier pour éviter les conflits (ex. : `3001:3000`).

### Volume de Données et Sécurité

  - **`- /data:/data`** : Cette ligne monte le volume de données de l'hôte (ici `/data`) sur le répertoire interne du conteneur (`/data`). C'est l'emplacement où les fichiers envoyés seront stockés de manière **persistante**.

> **Note de sécurité** : Les données transférées et stockées sur votre instance PsiTransfer sont **chiffrées**. Même en accédant directement au volume sur l'hôte, vous pourrez voir les fichiers chiffrés et les fichiers `.json` associés, mais pas le contenu non chiffré.

### Mot de Passe Administrateur

  - **`#- PSITRANSFER_ADMIN_PASS=blablalinux`** : Cette ligne est commentée par défaut. Pour **activer et sécuriser** la page d'administration de PsiTransfer, vous devez **dé-commenter** la ligne (supprimer le `#`) et spécifier votre mot de passe :
    ```yaml
    environment:
        - PSITRANSFER_ADMIN_PASS=votre_mot_de_passe_securise
    ```
    La page `admin` vous permet de gérer et de visualiser la liste des transferts actifs sur votre instance.

### Version de l'Image

  - **`image: psitrax/psitransfer:latest`** : Vous pouvez remplacer le tag **`latest`** par un [tag spécifique](https://hub.docker.com/r/psitrax/psitransfer/tags) pour cibler une version précise et stable du logiciel.

-----

## 4\. Lancement

Une fois le fichier `docker-compose.yml` configuré, lancez votre pile :

```bash
docker compose up -d
```

Vous accéderez à votre instance PsiTransfer via votre navigateur à l'adresse **`http://<Votre_IP_Hôte>:3000`** (ou le port que vous avez choisi).

![](/docker-compose-psitransfer/psitransfer-password-admin.png)

Page “admin” de PsiTransfer - Demande de mot de passe

![](/docker-compose-psitransfer/psitransfer-admin.png)

Page “admin” de PsiTransfer

![](/docker-compose-psitransfer/psitransfer-data.png)

PsiTranfer - Données /data

![](/docker-compose-psitransfer/psitransfer-data-json.png)

PsiTranfer - Données /data - Fichier .json

![](/docker-compose-psitransfer/psitransfer-data-2.png)

PsiTranfer - Données /data chiffrées

![](/docker-compose-psitransfer/psitransfer-data-3.png)

PsiTranfer - Données /data chiffrées

### Démonstration

Vous avez le choix de voir l'application en action :

  - [Facebook](https://www.facebook.com/blablalinux/videos/453133560991741)
  - [Twitter](https://x.com/i/status/1806849212237173113)
  - [Mastodon](https://mastodon-blablalinux.be/@blablalinux/112697108025204971)