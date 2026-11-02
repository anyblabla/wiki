---
title: Bonnes pratiques avec Docker Compose
description: >-
  Configurez vos applications Docker avec des bonnes pratiques. Découvrez les
  meilleures méthodes pour structurer vos fichiers Compose, gérer les secrets,
  persister les données (volumes) et optimiser la sécurité en production.
published: true
date: '2026-11-02T23:24:03.995Z'
tags:
  - docker
  - compose
  - conteneurisation
  - déploiement
editor: markdown
dateCreated: '2025-11-22T15:18:58.075Z'
---

Docker Compose est un outil puissant pour définir et exécuter des applications multi-conteneurs. Suivre ces bonnes pratiques vous aidera à créer des configurations **maintenables**, **sécurisées** et **portables**.

-----

## 1\. Organisation du fichier Compose

  * **Nom de fichier recommandé :** La nouvelle convention est d'utiliser **`compose.yaml`** (ou `.yml`) pour simplifier. L'ancien `docker-compose.yaml` fonctionne, mais le nouveau format est préféré.
  * **Version implicite (recommandée) :** La clé `version` (ex: `version: '3.8'`) est considérée comme **obsolète** et n'est plus nécessaire. Il est recommandé de l'omettre pour que Docker Compose (V2 et suivants) utilise automatiquement la dernière spécification.
    **Exemple de fichier moderne (sans `version`):**
    ```yaml
    services:
      app:
        # ...
    ```
  * **Nommage clair des services :** Utilisez des noms de services courts, descriptifs et en minuscules (ex: `app`, `db`, `cache`, `api`).
  * **Séparation des environnements :** Utilisez des fichiers Compose multiples pour gérer les différences entre les environnements (**développement**, **test**, **production**).
      * `compose.yaml` : Configuration de base commune.
      * `compose.override.yaml` : Surcharge pour le développement (montages de volumes, ports exposés, etc.).
      * `compose.prod.yaml` : Surcharge pour la production (réseaux dédiés, secrets, limites de ressources).
    > **Exemple d'utilisation :**
    > **Démarrer en développement** (utilise `compose.yaml` et `compose.override.yaml` par défaut) :
    > ```bash
    > docker compose up
    > ```
    > **Démarrer en production** :
    > ```bash
    > docker compose -f compose.yaml -f compose.prod.yaml up -d
    > ```

-----

## 2\. Gestion des images et de la construction

  * **Préférer les images officielles/minimales :** Utilisez des images de base **officielles** (ex: `postgres:16-alpine`, `node:20-slim`) et privilégiez les variantes minimales (comme `alpine` ou `slim`) pour réduire la taille des images et la surface d'attaque.

  * **Utiliser le `build` avec un `Dockerfile` :** Si vous construisez votre propre image, utilisez toujours l'instruction `build` pour pointer vers un `Dockerfile` dans un répertoire spécifique.
    **Exemple de `build` (pour le développement) :**

    ```yaml
    services:
      app:
        build:
          context: ./app-code
          dockerfile: Dockerfile
        image: monapp-custom:latest # Nommer l'image pour la référence en production
    ```

  * **Images pré-construites pour la production :** En production, il est souvent préférable d'utiliser l'`image` (tirer une image depuis un registre) plutôt que le `build` (construire l'image localement) pour garantir la reproductibilité et la rapidité du déploiement.

    **Exemple de surcharge de production (`compose.prod.yaml`) :**

    ```yaml
    # Fichier compose.prod.yaml
    services:
      app:
        # 1. On retire l'instruction 'build' présente dans le fichier de base
        build: {} 
        
        # 2. On référence l'image déjà construite et taguée (par CI/CD par ex.)
        image: registry.blablalinux.be/monapp-custom:1.0.1
    ```

    > L'utilisation de `build: {}` dans le fichier de production permet d'annuler l'instruction `build` définie dans le fichier de base (`compose.yaml`) lors de la fusion des fichiers Compose.

-----

## 3\. Sécurité et gestion des secrets

  * **Ne jamais mettre les secrets en clair :** N'ajoutez **jamais** de mots de passe, clés API ou autres données sensibles directement dans le fichier Compose (**`compose.yaml`**).

  * **Utiliser les variables d'environnement :** Utilisez les variables d'environnement (`environment`) et faites-les charger depuis un fichier `.env` externe (qui doit être ignoré par Git \!).
    **Fichier `.env` :**

    ```ini
    POSTGRES_USER=myuser
    POSTGRES_PASSWORD=mysupersecret
    ```

    **Dans `compose.yaml` :**

    ```yaml
    services:
      db:
        image: postgres
        environment:
          - POSTGRES_USER
          - POSTGRES_PASSWORD
          # Ou bien
          # POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ```

  * **Secrets Docker (pour la production) :** Pour les déploiements plus complexes (comme en Docker Swarm), utilisez l'objet `secrets` de Docker. Les secrets sont stockés de manière sécurisée et montés dans le conteneur en lecture seule.

    **Exemple d'utilisation de `secrets` :**

    ```yaml
    services:
      db:
        image: postgres
        secrets:
          - db_password_file # Le secret sera monté dans le conteneur

    secrets:
      db_password_file:
        # Fichier local contenant le secret (à ignorer par Git !)
        file: ./db_password.txt
    ```

    > Dans cet exemple, le contenu du fichier local `db_password.txt` est accessible dans le conteneur sous `/run/secrets/db_password_file`. Votre application doit lire ce fichier pour obtenir le mot de passe.

-----

## 4\. Gestion des données et du stockage

  * **Utiliser des volumes nommés (`volumes`) :** Les volumes nommés sont le moyen **recommandé** de persister les données des conteneurs (bases de données, fichiers uploadés, etc.) car ils sont gérés par Docker et plus performants/sûrs que les montages de *bind* (liens vers le système de fichiers hôte).
    **Exemple de volume nommé :**

    ```yaml
    services:
      db:
        image: postgres
        volumes:
          - db-data:/var/lib/postgresql/data

    volumes:
      db-data:
    ```

  * **Utiliser les montages de *bind* pour le développement :** Les montages de *bind* (`bind mounts`) sont parfaits pour le **développement** car ils permettent d'appliquer les modifications de code instantanément sans reconstruire l'image.

    **Exemple de montage de *bind* pour le code source :**

    ```yaml
    services:
      web:
        # ...
        volumes:
          # Chemin sur l'hôte (répertoire courant) : Chemin dans le conteneur
          - ./app:/usr/src/app 
          
          # Optionnel : exclure le dossier node_modules de l'hôte pour utiliser celui du conteneur
          - /usr/src/app/node_modules
    ```

-----

## 5\. Réseautage et communication

  * **Réseau par défaut :** Laissez Docker Compose créer le réseau par défaut (il est nommé d'après le nom du répertoire). Les services dans ce réseau peuvent communiquer entre eux simplement par leur **nom de service**.
    **Exemple :** Le service `app` peut accéder au service `db` en utilisant l'hôte `db` (ex: `jdbc:postgresql://db:5432/mydb`).

  * **Éviter d'exposer les ports inutiles :** N'exposez les ports (`ports`) à l'hôte **que** pour les services qui doivent être accessibles de l'extérieur (ex: le service web). Ne pas exposer les ports de la base de données ou du cache.

    **Mauvaise pratique :**

    ```yaml
    services:
      db:
        # ...
        ports:
          - "5432:5432" # Le port DB est ouvert sur l'hôte, ce qui est inutile et dangereux.
    ```

    **Bonne pratique :**

    ```yaml
    services:
      web:
        # ...
        ports:
          - "80:80" # Seul le service web est exposé.
    ```

-----

## 6\. Robustesse et santé

  * **Checks de santé (`healthcheck`) :** Définissez des *checks* de santé pour que Docker puisse déterminer si un conteneur est réellement prêt à servir du trafic (et non juste en cours d'exécution).
    **Exemple pour un service web simple :**
    ```yaml
    services:
      web:
        # ...
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost/health"]
          interval: 30s
          timeout: 10s
          retries: 5
    ```
  * **Redémarrage automatique (`restart`) :** Utilisez toujours une politique de redémarrage pour garantir que les services se relancent après une panne ou un redémarrage du système hôte.
      * `restart: always` (toujours redémarrer) est le plus courant.
      * `restart: unless-stopped` (sauf si vous l'avez arrêté manuellement).

-----

## 💡 Exemple complet (développement)

Cet exemple nécessite un fichier `.env` à la racine du projet pour fonctionner.

**Exemple de fichier `.env` :**

```ini
# --- Variables d'environnement pour l'exemple Compose ---
DB_USER=devuser
DB_PASSWORD=devpassword
DB_NAME=myapp_dev
```

**Fichier `compose.yaml` :**

```yaml
# Notez l'absence de la clé 'version', conforme aux bonnes pratiques modernes.

services:
  web:
    # On construit l'image depuis le Dockerfile situé dans le répertoire './app'
    build:
      context: ./app
    # 🎯 Montage de Bind pour le code : synchronisation du code en temps réel
    volumes:
      - ./app:/usr/src/app 
      
      # Optionnel : exclure le dossier node_modules de l'hôte pour utiliser celui du conteneur
      - /usr/src/app/node_modules

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
      # Volume nommé pour la persistance des données (non synchronisé)
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

Pour approfondir les concepts et consulter la référence officielle, voici le lien essentiel :

#### 🔗 Documentation officielle Docker

  * **Documentation complète Docker**
    > Le point de départ pour toute la documentation technique relative à Docker, y compris Docker Compose, les Dockerfiles, et les bonnes pratiques de sécurité.
      * [Lien vers la documentation Docker](https://docs.docker.com/)
