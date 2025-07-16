---
title: Docker Compose Picsur
description: Déployez simplement Picsur grâce à une pile (stack) docker compose.
published: true
date: 2025-07-16T23:42:56.987Z
tags: docker, picsur
editor: markdown
dateCreated: 2024-05-18T22:22:24.229Z
---

Pratique pour déployer rapidement [Picsur](https://github.com/caramelfur/picsur) dans [Portainer](https://www.portainer.io) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/).

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

-   En rouge, à remplacer pour vos identifiants personnels :
    -   PICSUR\_DB\_PASSWORD: blablalinux
    -   PICSUR\_ADMIN\_PASSWORD: blablalinux
    -   POSTGRES\_PASSWORD: blablalinux

**C'est préférable que les identifiants correspondent !**

---

Picsur Blabla Linux : [https://picsur.blablalinux.be](https://picsur.blablalinux.be)