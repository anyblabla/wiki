---
title: Migration d'un bind mount vers un volume nommé Docker
description: >-
  Guide étape par étape pour migrer vos données de persistance Docker d'un Bind
  Mount (dossier local) vers un Volume Nommé, garantissant la portabilité et la
  sécurité de votre configuration.
published: true
date: '2026-11-02T23:53:50.674Z'
tags:
  - docker
  - mount
  - compose
  - volume
  - bind
  - migration
  - stockage
editor: markdown
dateCreated: '2025-11-22T20:48:15.823Z'
---

Ce guide documente la procédure standard pour transférer des données existantes d'un **bind mount** (dossier hôte local, ex: `./postgres14`) vers un **volume nommé** géré par Docker (ex: `mastodon_postgres_data`). Cette méthode garantit la persistance, la sécurité, et la portabilité.

### Prérequis : Mise à jour du fichier docker-compose.yml

Avant de commencer, modifiez votre fichier `docker-compose.yml` en deux endroits pour déclarer et utiliser le volume nommé.

#### 1\. Déclaration du volume dans le service

Remplacez le chemin local par le nom du volume dans la section `volumes:` du service concerné.

| Ancien bind mount | Nouveau volume nommé |
| :--- | :--- |
| `volumes: - ./postgres14:/var/lib/postgresql/data` | `volumes: - mastodon_postgres_data:/var/lib/postgresql/data` |

#### 2\. Déclaration du volume à la racine du fichier (obligatoire)

Ajoutez la section `volumes:` à la fin du fichier Compose :

```yaml
# ... après la section networks:

volumes:
  mastodon_postgres_data:
```

-----

## 🛠️ Étapes de migration

### Étape 1 : Arrêt du service

Arrêtez tous les conteneurs pour libérer l'accès aux dossiers de données sources.

```bash
docker compose down
```

### Étape 2 : Copie des données (migration)

Utilisez un conteneur temporaire basé sur l'image légère `alpine` pour monter l'ancien dossier et le nouveau volume, puis copier les données. Le drapeau **`-a` (archive)** est essentiel pour conserver les permissions des fichiers (UID/GID), ce qui est vital pour les bases de données.

Exécutez la commande suivante depuis le répertoire contenant votre `docker-compose.yml` :

```bash
# Migration des données de PostgreSQL (du dossier local './postgres14' vers le volume nommé)
docker run --rm \
  -v "$(pwd)/postgres14":/from \
  -v mastodon_postgres_data:/to \
  alpine \
  sh -c "cp -a /from/. /to/"
```

### Étape 3 : Lancement du service

Une fois que la commande de copie est terminée, lancez votre stack. Elle utilisera désormais les volumes nommés qui contiennent vos données migrées.

```bash
docker compose up -d
```

### Étape 4 : Nettoyage (optionnel)

Après avoir vérifié que l'application fonctionne parfaitement avec le volume nommé, vous pouvez supprimer l'ancien dossier de données bind mount.

```bash
# Suppression de l'ancien dossier bind mount
rm -rf ./postgres14
```
