---
title: Docker Compose Wiki.js
description: Déployez simplement Wiki.js grâce à une pile (stack) docker compose.
published: true
date: 2025-04-19T23:31:51.869Z
tags: docker, wiki
editor: markdown
dateCreated: 2024-05-18T20:22:28.247Z
---

Pratique pour déployer rapidement [Wiki.js](https://js.wiki) dans [Portainer](https://www.portainer.io) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/).

```plaintext
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

-   En rouge, à remplacer pour vos identifiants personnels :
    -   POSTGRES\_PASSWORD: blablalinux
    -   POSTGRES\_USER: anyblabla
    -   DB\_USER: anyblabla
    -   DB\_PASS: blablalinux

**Les identifiants _POSTGRES_ et _DB_ doivent correspondres !**