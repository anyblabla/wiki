---
title: Déploiement de Picsur (Image Hosting) avec Docker Compose
description: Ce guide explique comment déployer rapidement Picsur, une application d'hébergement d'images, en utilisant une pile Docker (stack) dans Portainer à partir d'un fichier compose YAML.
published: true
date: 2025-10-28T13:14:51.107Z
tags: docker, picsur
editor: markdown
dateCreated: 2024-05-18T22:22:24.229Z
---

## 1\. Fichier `docker-compose.yml`

Picsur utilise une base de données **PostgreSQL** pour stocker ses métadonnées. Le fichier Compose définit deux services (`picsur` et `picsur_postgres`) et un volume persistant pour les données de la base de données.

```plaintext
version: '3'
services:
  picsur:
    image: ghcr.io/caramelfur/picsur:latest
    container_name: picsur
    ports:
      - '8088:8080'
    environment:
      PICSUR_HOST: '0.0.0.0'
      PICSUR_PORT: 8080

      PICSUR_DB_HOST: picsur_postgres
      PICSUR_DB_PORT: 5432
      PICSUR_DB_USERNAME: picsur
      PICSUR_DB_PASSWORD: blablalinux
      PICSUR_DB_DATABASE: picsur

      ## The default username is admin, this is not modifyable
      PICSUR_ADMIN_PASSWORD: blablalinux

      ## Optional, random secret will be generated if not set
      # PICSUR_JWT_SECRET: CHANGE_ME
      # PICSUR_JWT_EXPIRY: 7d

      ## Maximum accepted size for uploads in bytes
      PICSUR_MAX_FILE_SIZE: 10485760 # 10 MB
      ## No need to touch this, unless you use a custom frontend
      # PICSUR_STATIC_FRONTEND_ROOT: "/picsur/frontend/dist"

      ## Warning: Verbose mode might log sensitive data
      # PICSUR_VERBOSE: "true"
    restart: always
  picsur_postgres:
    image: postgres:14-alpine
    container_name: picsur_postgres
    environment:
      POSTGRES_DB: picsur
      POSTGRES_PASSWORD: blablalinux
      POSTGRES_USER: picsur
    restart: always
    volumes:
      - picsur-data:/var/lib/postgresql/data
volumes:
  picsur-data:
```

Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).

-----

## 2\. Configuration des Variables Cruciales

Le déploiement nécessite de personnaliser trois variables clés, de préférence avec la **même valeur** pour simplifier la gestion.

| Variable | Service | Rôle |
| :--- | :--- | :--- |
| **`PICSUR_DB_PASSWORD`** | `picsur` | Mot de passe utilisé par l'application pour se connecter à la DB. |
| **`POSTGRES_PASSWORD`** | `picsur_postgres` | Mot de passe de l'utilisateur `picsur` dans la base de données. |
| **`PICSUR_ADMIN_PASSWORD`** | `picsur` | Mot de passe de l'utilisateur **`admin`** de Picsur (le nom d'utilisateur par défaut n'est pas modifiable). |

> **Il est préférable que ces identifiants correspondent** pour garantir que les services communiquent correctement entre eux et pour faciliter votre accès administrateur.

### Autres variables importantes

| Variable | Valeur | Description |
| :--- | :--- | :--- |
| **Ports** | `8088:8080` | L'application Picsur tourne sur le port interne **8080**. Elle est exposée sur le port **8088** de votre hôte. |
| **Taille Max.** | `PICSUR_MAX_FILE_SIZE: 10485760` | Taille maximale des fichiers acceptés en **octets** (ici **10 Mo**). |
| **Base de données** | `PICSUR_DB_HOST: picsur_postgres` | Nom d'hôte de la DB, qui correspond au nom du service Postgres dans le fichier Compose. |

-----

## 3\. Lancement

Une fois le fichier `docker-compose.yml` copié dans une pile Portainer ou exécuté via Docker Compose, le déploiement lancera les deux conteneurs.

```plaintext
docker compose up -d
```

Vous pourrez accéder à l'interface de Picsur à l'adresse **`http://<Votre_IP_Hôte>:8088`**.

> Mon installation : Picsur Blabla Linux : [https://picsur.blablalinux.be](https://picsur.blablalinux.be)

Souhaitez-vous que je vous donne le guide de déploiement pour une autre application ?