---
title: Stratégie d'automatisation globale
description: Centralisation des mécanismes d'automatisation des clusters BlablaLinux. Ce guide détaille la maintenance des hôtes Proxmox, le cycle de vie Docker et les scripts de build pour BentoPDF et Mastodon.
published: false
date: 2026-04-08T18:02:59.242Z
tags: docker, proxmox, script, bash, pve, pbs, sysadmin, fediverse, maintenance, automatisation, devops, blablalinux
editor: markdown
dateCreated: 2026-04-08T18:02:59.241Z
---

Cette page centralise les mécanismes d'automatisation des clusters BlablaLinux. L'objectif est de garantir la continuité de service et la mise à jour constante des applicatifs avec une intervention humaine minimale.

> **Note importante :** Les scripts proposés sur cette page sont **complets et fonctionnels**. Ils sont présentés ici de manière **anonymisée** (adresses IP, jetons API et URLs génériques) pour vous permettre de les adapter facilement à votre propre infrastructure.

-----

## 1\. Architecture de maintenance (hôtes et invités)

L'intelligence de maintenance est décentralisée : chaque nœud possède ses propres scripts et planifications pour éviter tout point unique de défaillance.

  * **Mise à jour des invités** : [Script de mise à jour des invités (LXC et VM) sous Proxmox](/fr/script-update-lxc-vm-proxmox)
  * **Maintenance centralisée du parc** : [Maintenance centralisée d'un cluster Proxmox en Bash](/fr/maintenance-centralisee-cluster-proxmox-bash)
  * **Automatisation par tâche Cron** : [Mise à jour automatique de Proxmox via script et Cron](/fr/update-pve-script-cron)

-----

## 2\. Cycle de vie Docker et Watchtower

Automatisation du déploiement et de la mise à jour des conteneurs basée sur les métadonnées Proxmox.

  * **Lien utile :** [Script de gestion Watchtower par tags Proxmox](/fr/script-gestion-watchtower)

-----

## 3\. Pipelines de build personnalisés (persistance du branding)

Pour les services dont le code source est hébergé sur GitHub, une mise à jour (`git pull`) peut écraser vos modifications locales. Ces scripts automatisent la récupération du code tout en préservant votre personnalisation.

### A. Exemple BentoPDF (injection d'assets et méta-tags)

Ce script est utilisé lorsque le service nécessite une reconstruction (`build`) pour intégrer des logos ou modifier des balises méta dans le code source avant la compilation.

```bash
#!/bin/bash

# Auteur : Amaury aka BlablaLinux
PROJECT_DIR="/root/bentopdf"
CUSTOM_DIR="/root/bentopdf_custom"

echo "--- Mise à jour BentoPDF (Version BlablaLinux) ---"

cd "$PROJECT_DIR" || exit

# 1. Nettoyage Git
echo "[1/4] Récupération de la version officielle..."
git fetch origin main
git reset --hard origin/main

# 2. Restauration du branding (injection dans public/)
echo "[2/4] Réinjection de vos fichiers personnalisés..."
if [ -d "$CUSTOM_DIR" ]; then
    # Le compose va à la racine
    cp "$CUSTOM_DIR/docker-compose.yml" "$PROJECT_DIR/"

    # Injection des assets et config dans le dossier public/
    cp "$CUSTOM_DIR/logo-blablalinux.png" "$PROJECT_DIR/public/"
    cp "$CUSTOM_DIR/favicon.ico" "$PROJECT_DIR/public/"
    cp "$CUSTOM_DIR/favicon.svg" "$PROJECT_DIR/public/"
    cp "$CUSTOM_DIR/robots.txt" "$PROJECT_DIR/public/"
    cp "$CUSTOM_DIR/sitemap.xml" "$PROJECT_DIR/public/"

    # --- CORRECTION DES BALISES OPEN GRAPH (Facebook/Twitter) ---
    echo "-> Correction des balises méta pour les réseaux sociaux..."
    # On remplace l'image og par ton logo local dans les sources avant le build
    find "$PROJECT_DIR/src" -type f -name "*.tsx" -exec sed -i 's|https://www.bentopdf.com/images/og-home.png|https://bentopdf.example.com/logo-blablalinux.png|g' {} +
    find "$PROJECT_DIR/src" -type f -name "*.tsx" -exec sed -i 's|https://www.bentopdf.com/images/twitter-home.png|https://bentopdf.example.com/logo-blablalinux.png|g' {} +

    echo "-> Configuration, assets et méta-tags réinjectés."
else
    echo "-> [!] Erreur : Dossier $CUSTOM_DIR introuvable."
    exit 1
fi

# 3. Optimisation RAM pour le build
sed -i '/RUN npm run build/i ENV NODE_OPTIONS="--max-old-space-size=4096"' Dockerfile

# 4. Build (SANS CACHE) et nettoyage
echo "[3/4] Reconstruction de l'image (No-Cache pour forcer les modifs)..."
# On force le build sans cache pour que le sed soit bien compilé dans l'app
if docker compose build --no-cache && docker compose up -d; then
    echo "[4/4] Nettoyage des couches inutiles..."
    docker system prune -f
    echo "--- Terminé ! BentoPDF est à jour et optimisé. ---"
else
    echo "Erreur lors du build."
    exit 1
fi
```

### B. Exemple SnoShare (persistance du Docker-Compose)

Ce script montre une méthode de sauvegarde/restauration du fichier `docker-compose.yml` local avant d'aligner le dossier sur le dépôt GitHub.

```bash
#!/bin/bash

# Auteur : Amaury aka BlablaLinux
PROJECT_DIR="$HOME/snowshare"
BACKUP_PATH="$HOME/docker-compose.yml.bak"

echo "--- Début de la mise à jour de SnoShare ---"

if [ -d "$PROJECT_DIR" ]; then
    cd "$PROJECT_DIR" || exit
    echo "Dossier cible : $(pwd)"
else
    echo "Erreur : Le dossier $PROJECT_DIR n'existe pas."
    exit 1
fi

# 1. Sauvegarde de ton compose personnalisé
echo "[1/5] Sauvegarde de ton docker-compose.yml vers $BACKUP_PATH..."
cp docker-compose.yml "$BACKUP_PATH"

# 2. Nettoyage et mise à jour forcée (alignement sur GitHub)
echo "[2/5] Récupération de la version GitHub (Force Update)..."
git fetch origin
git reset --hard origin/main

# 3. Restauration de tes réglages personnalisés
echo "[3/5] Restauration de tes réglages (depuis le backup)..."
cp "$BACKUP_PATH" docker-compose.yml

# 4. Relance de Docker
echo "[4/5] Reconstruction et redémarrage des conteneurs..."
docker compose up -d --build

# 5. Nettoyage de l'espace disque
echo "[5/5] Nettoyage complet (System + Builder + Images)..."
docker system prune -f
docker builder prune -f
docker image prune -f

echo "--- Mise à jour terminée avec succès ! ---"
```

-----

## 4\. Maintenance des services critiques (Fediverse)

La maintenance du Fediverse se concentre sur l'hygiène du stockage pour éviter la saturation des disques par les médias distants et les caches.

### A. Script pour Mastodon (Docker & Gotify)

```bash
#!/bin/bash
# Script de maintenance Mastodon pour Docker (Version sécurisée avec alertes)
# Auteur : Amaury Libert (Blabla Linux)

# --- PARAMÈTRES DE GOTIFY ---
GOTIFY_URL="https://gotify.example.com"
GOTIFY_TOKEN="VOTRE_TOKEN_ICI"

# --- PARAMÈTRES DE NETTOYAGE ---
CONTAINER_NAME="mastodon-web"
LOGFILE="/var/log/mastodon-cleanup.log"
HOSTNAME=$(hostname)
DAYS_MEDIA=7
THREADS=4

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
send_gotify_notification() {
    if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
        local title="$1"
        local message="$2"
        local priority="$3"

        curl -k -s -X POST "${GOTIFY_URL}/message?token=${GOTIFY_TOKEN}" \
            -F "title=${title}" \
            -F "message=${message}" \
            -F "priority=${priority}" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance Mastodon sur $HOSTNAME : $(date)"
echo "======================================================"

# 1. Vérification de la présence du conteneur
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
    MSG="Erreur : Le conteneur $CONTAINER_NAME est introuvable."
    echo "$MSG"
    send_gotify_notification "❌ Mastodon Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
fi

# 2. Nettoyage des médias et miniatures de liens
echo "--- Étape 1 : Nettoyage des médias et vignettes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
docker exec -u mastodon $CONTAINER_NAME bin/tootctl preview_cards remove --days=$DAYS_MEDIA
if [ ${PIPESTATUS[0]} -ne 0 ]; then
    send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Le nettoyage des médias a rencontré une erreur sur $HOSTNAME." 5
fi

# 3. Nettoyage des anciens statuts et comptes inactifs
echo "--- Étape 2 : Nettoyage statuts et comptes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune

# 4. Optimisation RAM (Protection LXC)
echo "--- Étape 3 : Libération du cache RAM ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
    echo 3 > /proc/sys/vm/drop_caches
else
    echo "Note : Droits insuffisants pour drop_caches (LXC), ignoré."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 5. Envoi de la notification de succès final
send_gotify_notification "✅ Mastodon Cleanup TERMINÉ" "La maintenance sur $HOSTNAME est finie." 4

exit 0
```

### B. Script pour PeerTube (Docker & Gotify)

```bash
#!/bin/bash
# Script de maintenance PeerTube pour Docker
# S'aligne sur les outils officiels (Server tools) de PeerTube >= 6.2
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY ---
GOTIFY_URL="https://gotify.example.com"
GOTIFY_TOKEN="VOTRE_TOKEN_ICI"

# --- PARAMÈTRES DE MAINTENANCE ---
CONTAINER_NAME="peertube"
LOGFILE="/var/log/peertube-cleanup.log"
HOSTNAME=$(hostname)

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
send_gotify_notification() {
    if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
        local title="$1"
        local message="$2"
        local priority="$3"

        curl -k -s -X POST "${GOTIFY_URL}/message?token=${GOTIFY_TOKEN}" \
            -F "title=${title}" \
            -F "message=${message}" \
            -F "priority=${priority}" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance PeerTube sur $HOSTNAME : $(date)"
echo "======================================================"

# 1. Vérification de la présence du conteneur
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
    MSG="Erreur : Le conteneur $CONTAINER_NAME est introuvable."
    echo "$MSG"
    send_gotify_notification "❌ PeerTube Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
fi

# 2. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
echo "--- Étape 1 : Nettoyage du stockage (Prune) ---"
docker exec -u peertube $CONTAINER_NAME npm run prune-storage
if [ $? -ne 0 ]; then
    send_gotify_notification "⚠️ PeerTube Cleanup ALERTE" "Échec du nettoyage du stockage sur $HOSTNAME." 5
fi

# 3. Suppression des fichiers distants (vignettes, avatars d'autres instances)
echo "--- Étape 2 : Nettoyage des fichiers distants ---"
docker exec -u peertube $CONTAINER_NAME npm run house-keeping -- --delete-remote-files
if [ $? -ne 0 ]; then
    send_gotify_notification "⚠️ PeerTube Cleanup ALERTE" "Échec du nettoyage des fichiers distants sur $HOSTNAME." 5
fi

# 4. Optimisation RAM (Protection LXC)
echo "--- Étape 3 : Libération du cache RAM ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
    echo 3 > /proc/sys/vm/drop_caches
    echo "Cache RAM libéré avec succès."
else
    echo "Note : Droits insuffisants pour drop_caches (LXC), ignoré."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 5. Envoi de la notification de succès final
send_gotify_notification "✅ PeerTube Cleanup TERMINÉ" "La maintenance PeerTube sur $HOSTNAME s'est terminée." 4

exit 0
```

-----

## 5\. Résilience et miroirs de données

  * **AdGuard Home Sync** : [AdGuard Home - Installation et configuration complète](/fr/adguard-home-installation-configuration-complete)
  * **Gitea Mirroring** : J'utilise le projet [Gitea Mirror](https://github.com/RayLabsHQ/gitea-mirror) pour synchroniser automatiquement l'ensemble de mes dépôts distants (GitHub/GitLab) vers mon instance locale, garantissant ainsi la souveraineté totale de mes sources.

-----

## 6\. Pilotage à distance (station de commandement)

  * **Gestion de l'alimentation** :
      * [Gestion de l'alimentation à distance Proxmox (WOL)](/fr/gestion-alimentation-distance-proxmox-wol)
      * [Automatisation du cycle électrique d'un serveur Linux](/fr/automatisation-cycle-electrique-serveur-linux)
  * **Administration logicielle en terminal** :
      * [Gestion de WordPress en terminal avec WP-CLI](/fr/gestion-wordpress-terminal-wp-cli)
      * [Conception d'un outil CLI en Python pour Wiki.js](/fr/python-conception-cli-wikijs)

-----

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).