---
title: Maintenance et nettoyage de Mastodon sous Docker
description: Maintenance de Mastodon sous Docker : nettoyage automatique du cache média, des comptes inactifs et des vieux messages avec notifications Gotify optionnelles.
published: true
date: 2025-12-28T21:08:33.766Z
tags: mastodon, docker, lxc, proxmox, cron, crontab, script, bash, pve, gotify, maintenance, automatisation
editor: markdown
dateCreated: 2025-12-25T13:00:52.896Z
---

> ⚠️ Ce guide est spécifiquement conçu pour une installation de Mastodon tournant sous **Docker**. Si vous utilisez une installation "classique", consultez la page dédiée : [Maintenance Mastodon (installation classique)](https://wiki.blablalinux.be/fr/mastodon-cache).

---

## 1. Pourquoi utiliser un script plutôt que les réglages natifs ?

Le nettoyage régulier est indispensable pour éviter la saturation du disque. Mon script apporte :

* **La performance :** utilisation du multi-threading (`--concurrency`).
* **Le nettoyage complet :** suppression des cartes de prévisualisation et des comptes distants inactifs (non gérés par l'interface).

---

## 2. Automatisation par script (avec Gotify optionnel)

Ce script automatise le nettoyage et peut vous informer via **Gotify**.

> [!IMPORTANT]
> **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides ou avec leurs valeurs par défaut. Le script détectera qu'elles ne sont pas configurées et ignorera l'envoi des notifications sans générer d'erreur.

### Contenu du script : `mastodon-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance Mastodon pour Docker
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
# Laissez vide si vous n'utilisez pas Gotify
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE NETTOYAGE ---
CONTAINER_NAME="mastodon-web-1"
LOGFILE="/var/log/mastodon-cleanup.log"
HOSTNAME=$(hostname)
DAYS_MEDIA=7
THREADS=4 

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
send_gotify_notification() {
    # On vérifie si Gotify est configuré
    if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
        local title="$1"
        local message="$2"
        local priority="$3"

        curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
            -F "title=$title" \
            -F "message=$message" \
            -F "priority=$priority" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance Mastodon sur $HOSTNAME : $(date)"
echo "======================================================"

# 1. Vérification de la présence du conteneur
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
    echo "Erreur : Le conteneur $CONTAINER_NAME est introuvable."
    send_gotify_notification "❌ Mastodon Cleanup ÉCHEC" "Le conteneur $CONTAINER_NAME est introuvable sur $HOSTNAME." 8
    exit 1
fi

# 2. Nettoyage des médias et miniatures de liens
echo "--- Étape 1 : Nettoyage des médias et vignettes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
docker exec -u mastodon $CONTAINER_NAME bin/tootctl preview_cards remove --days=$DAYS_MEDIA

# 3. Nettoyage des anciens statuts et comptes inactifs
echo "--- Étape 2 : Nettoyage statuts et comptes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune

# 4. Optimisation RAM : libérer le cache système
echo "--- Étape 3 : Libération du cache RAM ---"
sync; echo 3 > /proc/sys/vm/drop_caches

echo "======================================================"
echo "Maintenance terminée avec succès : $(date)"
echo "======================================================"

# 5. Envoi de la notification de succès
send_gotify_notification "✅ Mastodon Cleanup SUCCÈS" "La maintenance hebdomadaire sur $HOSTNAME s'est terminée correctement." 4

exit 0

```

---

## 3. Planification avec Crontab

1. Rendre le script exécutable : `chmod 700 /root/scripts/mastodon-cleanup.sh`
2. Ajouter à la crontab (`crontab -e`) pour une exécution le dimanche à 3h00 :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

```

---

## 4. Conseils supplémentaires

**Libération de la RAM :** la commande `drop_caches` est cruciale pour récupérer la mémoire vive utilisée par le cache de la base de données pendant le nettoyage.

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)