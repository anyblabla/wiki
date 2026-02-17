---
title: Migration Listmonk - Du binaire natif vers Docker (LXC)
description: Ce guide technique détaille la migration d'une instance Listmonk d'un binaire natif vers Docker Compose en environnement LXC. Il couvre l'extraction, la restauration et l'ordre de démarrage.
published: true
date: 2026-02-17T18:31:41.456Z
tags: docker, lxc, proxmox, debian, migration, listmonk, binaire, postgresql
editor: markdown
dateCreated: 2026-02-17T18:31:41.456Z
---

Ce guide détaille la procédure technique pour migrer une instance **Listmonk** d'une installation en binaire autonome vers une architecture **Docker Compose** sous **LXC**, en garantissant la continuité des données.

## 1. Extraction des données (source)

Sur l'ancien LXC (binaire) :

1. **Générer le dump SQL** : `pg_dump -U listmonk -d listmonk > listmonk_dump.sql`
2. **Copier les médias** : récupérer l'intégralité du répertoire `uploads`.

## 2. Configuration du nouveau LXC (cible)

Sur le nouveau conteneur, les fichiers sont organisés dans `/root/listmonk`.

### Stratégie des identifiants

**Important :** pour une migration sans conflit, le fichier `docker-compose.yml` doit impérativement utiliser les mêmes `USER`, `PASSWORD` et `DATABASE` que l'ancienne installation.

### Le fichier docker-compose.yml

```yaml
services:
  listmonk_db:
    image: postgres:17-alpine
    container_name: listmonk_db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=listmonk        
      - POSTGRES_PASSWORD=votre_mdp    
      - POSTGRES_DB=listmonk          
    volumes:
      - ./listmonk-data:/var/lib/postgresql/data 
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U listmonk -d listmonk"]
      interval: 10s
      timeout: 5s
      retries: 5

  listmonk_app:
    image: listmonk/listmonk:latest
    container_name: listmonk_app
    restart: unless-stopped
    depends_on:
      listmonk_db:
        condition: service_healthy
    ports:
      - "9000:9000"
    environment:
      - LISTMONK_db__host=listmonk_db
      - LISTMONK_db__user=listmonk
      - LISTMONK_db__password=votre_mdp
      - LISTMONK_db__database=listmonk
      - LISTMONK_db__port=5432
      - LISTMONK_app__address=0.0.0.0:9000
    volumes:
      - ./config.toml:/listmonk/config.toml
      - ./uploads:/listmonk/uploads
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9000/healthlib"]
      interval: 30s
      timeout: 10s
      retries: 3

```

## 3. Restauration (ordre critique)

Au départ, le dossier `/root/listmonk` contient le `docker-compose.yml`, le dump SQL et le dossier `uploads`. **Le dossier `listmonk-data` sera créé par Docker lors de l'étape 1 ci-dessous.**

1. **Démarrer uniquement la base de données :**
```bash
docker compose up -d listmonk_db

```


2. **Injecter le dump SQL :**
```bash
docker cp /root/listmonk/listmonk_dump.sql listmonk_db:/backup.sql
docker exec -it listmonk_db psql -U listmonk -d listmonk -f /backup.sql

```


3. **Démarrer l'application Listmonk :**
```bash
docker compose up -d listmonk_app

```