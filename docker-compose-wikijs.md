---
title: Déploiement de Wiki.js avec Docker Compose
description: Ce guide explique comment déployer rapidement Wiki.js (une solution Wiki moderne, puissante et extensible) en utilisant une pile Docker (stack) dans Portainer à partir d'un fichier compose YAML.
published: true
date: 2025-10-28T13:35:32.211Z
tags: docker, wiki
editor: markdown
dateCreated: 2024-05-18T20:22:28.247Z
---

## 1\. Fichier `docker-compose.yml`

Ce fichier définit deux services : **`db`** (la base de données PostgreSQL) et **`wiki`** (l'application Wiki.js), ainsi que les volumes persistants nécessaires.

```yaml
version: "3"
services:

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: wiki
      POSTGRES_PASSWORD: blablalinux
      POSTGRES_USER: anyblabla
    logging:
      driver: "none"
    restart: always
    volumes:
      - db-data:/var/lib/postgresql/data

  wiki:
    image: ghcr.io/requarks/wiki:2
    depends_on:
      - db
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: anyblabla
      DB_PASS: blablalinux
      DB_NAME: wiki
    restart: always
    volumes:
      - data:/wiki
    ports:
      - "80:3000"

volumes:
  db-data:
  data:
```

Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).

-----

## 2\. Configuration des Identifiants (Crucial)

Pour que l'application Wiki.js (`wiki`) puisse se connecter à la base de données (`db`), les variables d'environnement doivent correspondre entre les deux services.

Vous devez personnaliser les identifiants ci-dessous :

| Service | Variable | Rôle |
| :--- | :--- | :--- |
| **`db`** | `POSTGRES_USER: anyblabla` | Nom d'utilisateur créé dans PostgreSQL. |
| **`wiki`** | `DB_USER: anyblabla` | Nom d'utilisateur utilisé par Wiki.js pour se connecter. |
| **`db`** | `POSTGRES_PASSWORD: blablalinux` | Mot de passe de l'utilisateur PostgreSQL. |
| **`wiki`** | `DB_PASS: blablalinux` | Mot de passe utilisé par Wiki.js pour se connecter. |

> **ATTENTION : Les identifiants `POSTGRES_USER` et `DB_USER` D'UNE PART, et `POSTGRES_PASSWORD` et `DB_PASS` D'AUTRE PART, doivent obligatoirement être identiques.**

### Autres Paramètres Clés

| Paramètre | Ligne | Explication |
| :--- | :--- | :--- |
| **Port d'accès** | `- "80:3000"` | L'application Wiki.js tourne sur le port interne **3000**. Elle est exposée sur le port **80** de votre hôte, idéal pour une intégration directe avec un Proxy inverse. |
| **Base de données** | `DB_HOST: db` | Le nom du service de base de données dans le fichier Compose est utilisé comme hôte pour la connexion. |
| **Volumes** | `db-data` et `data` | Les volumes sont utilisés pour la persistance des données PostgreSQL (`db-data`) et des configurations/fichiers de Wiki.js (`data`). |

-----

## 3\. Lancement du Déploiement

Une fois les identifiants personnalisés, vous pouvez lancer la pile :

```bash
docker compose up -d
```

L'application sera accessible via votre navigateur à l'adresse **`http://<Votre_IP_Hôte>`** ou **`http://<Votre_Domaine>`**. La première connexion vous guidera à travers le processus de configuration initiale de Wiki.js.

Souhaitez-vous que je vous assiste avec la configuration de votre Proxy Inverse (comme Nginx Proxy Manager) pour diriger le trafic vers ce service ?