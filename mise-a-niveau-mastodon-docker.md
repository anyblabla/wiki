---
title: Mettre à niveau Mastodon avec Docker Compose
description: Procédure générale pour mettre à niveau une instance Mastodon auto-hébergée via Docker Compose : sauvegarde, migrations pre/post-deployment, bascule des conteneurs et vérifications.
published: true
date: 2026-08-20T17:37:22.325Z
tags: mastodon, docker, upgrade, fediverse, docker-compose, mise-a-niveau, selfhosting, activitypub
editor: markdown
dateCreated: 2026-08-20T17:37:22.325Z
---

# Mettre à niveau Mastodon (Docker Compose)

Cette page décrit la procédure générale pour mettre à niveau une instance Mastodon auto-hébergée qui tourne via `docker compose`. Elle a été rédigée et validée à partir d'une mise à niveau réelle **v4.6.6 → v4.7.0**, mais s'applique globalement à toute montée de version mineure ou majeure.

> ⚠️ **Adapte cette procédure à ton installation.** Les noms de service (`web`, `sidekiq`, `streaming`, `db`...), les noms de conteneurs, l'organisation du réseau Docker (bridge unique, réseaux séparés internal/external...) et le nom de la base de données peuvent varier d'une instance à l'autre selon comment le `docker-compose.yml` a été écrit. Les commandes ci-dessous utilisent les noms de service standards du `docker-compose.yml` officiel Mastodon (`web`, `sidekiq`, `streaming`, `db`, `redis`) — vérifie les tiens avec `docker compose config --services` avant de commencer.

## Avant de commencer

1. **Toujours lire les notes de version officielles** avant de mettre à niveau : https://github.com/mastodon/mastodon/releases
   Certaines versions imposent des étapes particulières (migrations longues, recompilation d'assets, changement de dépendances, cookies invalidés, etc.). Si tu sautes plusieurs versions d'un coup, lis aussi les notes des versions intermédiaires.
2. **Faire une sauvegarde de la base de données.** Peu importe l'outil (pg_dump manuel, [Databasus](https://github.com/), un snapshot Proxmox de la VM/LXC, etc.) — assure-toi simplement d'avoir un backup récent et restaurable avant de toucher à quoi que ce soit.
   Exemple avec `pg_dump` si tu n'as pas d'outil dédié :
   ```bash
   docker exec <nom_conteneur_postgres> pg_dump -Fc -U postgres <nom_de_la_base> > mastodon_backup_$(date +%Y%m%d).dump
   ```
3. **Sauvegarder ton `docker-compose.yml`** avant modification :
   ```bash
   cp docker-compose.yml docker-compose.yml.bkp-<version_actuelle>
   ```
4. **Mettre en pause Watchtower** (ou tout outil de mise à jour automatique de conteneurs) le temps de la manip, pour éviter qu'il n'interfère pendant la bascule manuelle :
   ```bash
   docker stop watchtower
   ```

## Étapes de mise à niveau

### 1. Identifier les services concernés

Dans un `docker-compose.yml` standard Mastodon, les services qui utilisent l'image `mastodon/mastodon` ou `mastodon/mastodon-streaming` sont typiquement nommés `web`, `sidekiq` et `streaming`. Vérifie les tiens :

```bash
docker compose config --services
grep -B2 "image:.*mastodon" docker-compose.yml
```

### 2. Mettre à jour les tags d'image

Édite le `docker-compose.yml` et remplace l'ancien tag de version par le nouveau sur **tous** les services concernés (web, sidekiq, streaming). Exemple avec `sed` (à adapter aux versions réelles) :

```bash
sed -i 's/mastodon:v4.6.6/mastodon:v4.7.0/g; s/mastodon-streaming:v4.6.6/mastodon-streaming:v4.7.0/g' docker-compose.yml
grep image docker-compose.yml   # vérification visuelle
```

### 3. Récupérer les nouvelles images

```bash
docker compose pull web sidekiq streaming
```

### 4. Migrations pre-deployment

Certaines migrations de base de données peuvent être effectuées **avant** de basculer les nouveaux conteneurs en production, sans downtime, en excluant les migrations "post-deployment" (celles qui nécessitent que l'ancien code ne tourne plus) :

```bash
docker compose run --rm -e SKIP_POST_DEPLOYMENT_MIGRATIONS=true web bundle exec rails db:migrate
```

Adapte `web` si ton service s'appelle différemment. Cette commande recrée un conteneur éphémère pour lancer la migration ; elle peut redémarrer `db`/`redis` si besoin, c'est normal.

### 5. Basculer les conteneurs sur la nouvelle version

```bash
docker compose up -d --force-recreate web sidekiq streaming
```

Vérifie que tout repasse `healthy` avant de continuer :

```bash
docker ps
```

### 6. Migrations post-deployment

Une fois les nouveaux conteneurs up et sains, termine les migrations restantes :

```bash
docker compose run --rm web bundle exec rails db:migrate
```

### 7. Vérifications

```bash
docker ps                     # tous les conteneurs "healthy"
docker compose logs -f web    # surveiller les logs quelques minutes (Ctrl+C pour sortir)
```

Puis un contrôle fonctionnel côté navigateur : chargement du timeline, notifications, publication d'un post de test, réception de fédération (les `POST /inbox` dans les logs sont bon signe).

### 8. Relancer Watchtower

```bash
docker start watchtower
```

### 9. Nettoyage (optionnel)

Une fois la stabilité confirmée (quelques heures à quelques jours selon ta tolérance au risque) :

```bash
docker image prune -f
```

## Notes et pièges connus

- **Nom de la base de données** : dans le `.env.production`, vérifie la variable `DB_NAME` — ce n'est pas toujours `postgres` par défaut (souvent `mastodon`). Adapte tes commandes `pg_dump` en conséquence si tu en fais un manuellement.
- **Migrations longues** : certaines versions majeures convertissent des vues matérialisées en tables ou ajoutent des index sur de grosses tables (`accounts`, `statuses`...). Sur une instance avec beaucoup de contenu fédéré, ça peut prendre plusieurs minutes — c'est normal, laisse tourner.
- **Réseaux Docker séparés** (internal/external) : si tes services `web`/`sidekiq` tournent sur plusieurs réseaux Docker Compose (cas fréquent pour isoler la base de données d'Internet), les commandes `docker compose run` reprennent automatiquement la configuration réseau du service — pas besoin de préciser `--network` manuellement si tu passes bien par `docker compose run` (contrairement à `docker run` brut).
- **Watchtower** : s'il n'est pas mis en pause, il peut repull l'image officielle en tâche de fond et recréer un conteneur en plein milieu de la manip, provoquant des incohérences entre le code applicatif et l'état des migrations. Toujours le stopper avant une mise à niveau manuelle.

## Sources

- [Notes de version officielles Mastodon](https://github.com/mastodon/mastodon/releases)
- [Documentation d'installation Docker officielle](https://docs.joinmastodon.org/admin/install/)
