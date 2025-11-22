---
title: Bonnes Pratiques avec Docker Compose
description: Configurez vos applications Docker avec des bonnes pratiques. Découvrez les meilleures méthodes pour structurer vos fichiers Compose, gérer les secrets, persister les données (volumes) et optimiser la sécurité en production.
published: true
date: 2025-11-22T15:26:50.765Z
tags: docker, compose, conteneurisation, déploiement
editor: markdown
dateCreated: 2025-11-22T15:18:58.075Z
---

Docker Compose est un outil puissant pour définir et exécuter des applications multi-conteneurs. Suivre ces bonnes pratiques vous aidera à créer des configurations **maintenables**, **sécurisées** et **portables**.

### 1\. Organisation du fichier `docker-compose.yml`

  * **Version précise :** Toujours spécifier la version de la spécification Compose en haut du fichier pour assurer la compatibilité et l'accès aux dernières fonctionnalités.

    > **Exemple :**

    > ```yaml
    > version: '3.8'
    > services:
    >   # ...
    > ```

  * **Nommage clair des services :** Utilisez des noms de services courts, descriptifs et en minuscules (ex: `app`, `db`, `cache`, `api`).

  * **Séparation des environnements :** Utilisez des fichiers Compose multiples pour gérer les différences entre les environnements (**développement**, **test**, **production**).

      * `docker-compose.yml` : Configuration de base commune.
      * `docker-compose.override.yml` : Surcharge pour le développement (montages de volumes, ports exposés, etc.).
      * `docker-compose.prod.yml` : Surcharge pour la production (réseaux dédiés, secrets, limites de ressources).

    > **Exemple d'utilisation :**

    > ```bash
    > # Démarrer en développement
    > docker compose up
    > ```

    > # Démarrer en production

    > docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

    > ```
    > ```

-----

### 2\. Gestion des images et de la construction

  * **Préférer les images officielles/minimales :** Utilisez des images de base **officielles** (ex: `postgres:16-alpine`, `node:20-slim`) et privilégiez les variantes minimales (comme `alpine` ou `slim`) pour réduire la taille des images et la surface d'attaque.
  * **Utiliser le `build` avec un `Dockerfile` :** Si vous construisez votre propre image, utilisez toujours l'instruction `build` pour pointer vers un `Dockerfile` dans un répertoire spécifique.
    > **Exemple :**
    > ```yaml
    > services:
    >   app:
    >     build:
    >       context: ./app-code
    >       dockerfile: Dockerfile
    >     image: monapp-custom:latest
    > ```
  * **Images pré-construites pour la production :** En production, il est souvent préférable d'utiliser l'`image` (tirer une image depuis un registre) plutôt que le `build` (construire l'image localement) pour garantir la reproductibilité et la rapidité du déploiement.

-----

### 3\. Sécurité et gestion des secrets

  * **Ne jamais mettre les secrets en clair :** N'ajoutez **jamais** de mots de passe, clés API ou autres données sensibles directement dans le fichier `docker-compose.yml`.
  * **Utiliser les variables d'environnement :** Utilisez les variables d'environnement (`environment`) et faites-les charger depuis un fichier `.env` externe (qui doit être ignoré par Git \!).
    > **Fichier `.env` :**
    > ```ini
    > POSTGRES_USER=myuser
    > POSTGRES_PASSWORD=mysupersecret
    > ```
    > **Dans `docker-compose.yml` :**
    > ```yaml
    > services:
    >   db:
    >     image: postgres
    >     environment:
    >       - POSTGRES_USER
    >       - POSTGRES_PASSWORD
    >       # Ou bien
    >       # POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    > ```
  * **Secrets Docker (pour la production) :** Pour les déploiements plus complexes (comme en Docker Swarm), utilisez l'objet `secrets` de Docker.

-----

### 4\. Gestion des données et du stockage

  * **Utiliser des volumes nommés (`volumes`) :** Les volumes nommés sont le moyen **recommandé** de persister les données des conteneurs (bases de données, fichiers uploadés, etc.) car ils sont gérés par Docker et plus performants/sûrs que les montages de *bind* (liens vers le système de fichiers hôte).

    > **Exemple :**

    > ```yaml
    > services:
    >   db:
    >     image: postgres
    >     volumes:
    >       - db-data:/var/lib/postgresql/data
    > ```

    > volumes:
    > db-data:

    > ```
    > ```

  * **Utiliser les montages de *bind* pour le développement :** Les montages de *bind* (`bind mounts`) sont parfaits pour le **développement** car ils permettent d'appliquer les modifications de code instantanément sans reconstruire l'image.

-----

### 5\. Réseautage et communication

  * **Réseau par défaut :** Laissez Docker Compose créer le réseau par défaut (il est nommé d'après le nom du répertoire). Les services dans ce réseau peuvent communiquer entre eux simplement par leur **nom de service**.

    > **Exemple :** Le service `app` peut accéder au service `db` en utilisant l'hôte `db` (ex: `jdbc:postgresql://db:5432/mydb`).

  * **Éviter d'exposer les ports inutiles :** N'exposez les ports (`ports`) à l'hôte **que** pour les services qui doivent être accessibles de l'extérieur (ex: le service web). Ne pas exposer les ports de la base de données ou du cache.

    > **Mauvaise pratique :**

    > ```yaml
    > services:
    >   db:
    >     # ...
    >     ports:
    >       - "5432:5432" # Le port DB est ouvert sur l'hôte, ce qui est inutile et dangereux.
    > ```

    > **Bonne pratique :**

    > ```yaml
    > services:
    >   web:
    >     # ...
    >     ports:
    >       - "80:80" # Seul le service web est exposé.
    > ```

-----

### 6\. Robustesse et santé

  * **Checks de santé (`healthcheck`) :** Définissez des *checks* de santé pour que Docker puisse déterminer si un conteneur est réellement prêt à servir du trafic (et non juste en cours d'exécution).
    > **Exemple pour un service web simple :**
    > ```yaml
    > services:
    >   web:
    >     # ...
    >     healthcheck:
    >       test: ["CMD", "curl", "-f", "http://localhost/health"]
    >       interval: 30s
    >       timeout: 10s
    >       retries: 5
    > ```
  * **Redémarrage automatique (`restart`) :** Utilisez toujours une politique de redémarrage pour garantir que les services se relancent après une panne ou un redémarrage du système hôte.
      * `restart: always` (toujours redémarrer) est le plus courant.
      * `restart: unless-stopped` (sauf si vous l'avez arrêté manuellement).

-----

## 💡 Exemple complet (développement)

```yaml
version: '3.8'

services:
  web:
    # On construit l'image depuis le Dockerfile situé dans le répertoire './app'
    build:
      context: ./app
    # On monte le code source pour permettre le rechargement à chaud en développement
    volumes:
      - ./app:/usr/src/app
    ports:
      - "8000:8000"
    environment:
      # On référence les variables définies dans le fichier .env
      DATABASE_HOST: db
      DATABASE_NAME: ${DB_NAME}
    depends_on:
      db:
        condition: service_healthy # Attendre que la DB soit saine
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      # Volume nommé pour la persistance des données
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  # Déclaration des volumes nommés
  postgres_data:
```

-----

## 📚 Ressources et documentation

Pour approfondir les concepts et consulter la référence officielle, voici quelques liens essentiels :

### 🇫🇷 Tutoriels et guides en français

  * **Guide Docker Compose : Simplifiez le développement de conteneurs multiples**

    > Un guide complet qui explique les principes fondamentaux, la syntaxe YAML et les éléments clés (services, réseaux, volumes).

    >   * [Lien DataCamp (en français)](https://www.datacamp.com/fr/tutorial/docker-compose-guide)

  * **Docker Compose : tout ce qu'il faut savoir**

    > Un aperçu détaillé des commandes de base, de l'utilisation des volumes persistants et des variables d'environnement.

    >   * [Lien DataScientest (en français)](https://datascientest.com/docker-compose-tout-savoir)

  * **Introduction à Docker Compose**

    > Un article se concentrant sur la structure du fichier, l'utilisation et la gestion des secrets, souvent pertinent pour la transition vers la production.

    >   * [Lien Stéphane ROBERT (en français)](https://blog.stephane-robert.info/docs/conteneurs/orchestrateurs/docker-compose/)

  * **Tutoriel Docker Compose (vidéo)**

    > Pour ceux qui préfèrent un support vidéo, ce tutoriel complet couvre les bases et la création d'un premier fichier Compose.

    >   * [Lien YouTube - Docker Compose va vous ÉBLOUIR \!\!](https://www.youtube.com/watch?v=04TkqB5WdL0)

### 🇬🇧 Référence officielle (en anglais, incontournable)

  * **Référence du fichier Compose (Le guide technique ultime)**

    > C'est la référence technique complète de toutes les clés (`services`, `volumes`, `networks`, `healthcheck`, etc.) que vous pouvez utiliser.

    >   * [Lien de référence de Docker Compose (en anglais)](https://docs.docker.com/compose/compose-file/compose-file-v3/)

  * **Bonnes pratiques pour la construction d'images**

    > Crucial pour garantir la sécurité et la petite taille de vos images Docker.

    >   * [Bonnes pratiques pour la construction d'images (en anglais)](https://docs.docker.com/build/building/best-practices/)