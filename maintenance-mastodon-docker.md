---
title: Maintenance et nettoyage de Mastodon sous Docker
description: Maintenance de Mastodon sous Docker : nettoyage automatique du cache média, des comptes inactifs et des vieux messages avec notifications Gotify optionnelles.
published: true
date: 2025-12-29T21:46:28.301Z
tags: mastodon, docker, lxc, proxmox, cron, crontab, script, bash, pve, gotify, maintenance, automatisation
editor: markdown
dateCreated: 2025-12-25T13:00:52.896Z
---

> ⚠️ Ce guide est spécifiquement conçu pour une installation de Mastodon tournant sous **Docker** (souvent via un conteneur LXC sur Proxmox). Si vous utilisez une installation "classique", consultez la page dédiée : [Maintenance Mastodon (installation classique)](https://wiki.blablalinux.be/fr/mastodon-cache).

---

## 1. Pourquoi utiliser un script plutôt que les réglages natifs ?

Le nettoyage régulier est indispensable pour éviter la saturation du disque. Mon script apporte :

* **La performance :** utilisation du multi-threading (`--concurrency`) pour un traitement plus rapide.
* **Le nettoyage complet :** suppression des cartes de prévisualisation et des comptes distants inactifs, des éléments souvent oubliés par les tâches de fond natives.
* **La surveillance :** une notification immédiate sur votre téléphone via Gotify en cas de succès ou d'alerte.

---

## 2. Automatisation par script (avec Gotify optionnel)

Ce script automatise le nettoyage et peut vous informer via **Gotify**.

> 🔴 **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides. Le script détectera l'absence de configuration et ignorera l'envoi des notifications sans générer d'erreur.

### Contenu du script : `mastodon-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance Mastodon pour Docker
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
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

# 3. Nettoyage des anciens statuts et comptes inactifs
echo "--- Étape 2 : Nettoyage statuts et comptes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
if [ $? -ne 0 ]; then
    send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Le nettoyage des statuts a rencontré une erreur sur $HOSTNAME." 5
fi
docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune

# 4. Optimisation RAM (Vérification des droits LXC)
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
send_gotify_notification "✅ Mastodon Cleanup TERMINÉ" "La maintenance sur $HOSTNAME est terminée avec succès." 4

exit 0

```

---

## 3. Planification avec Crontab

1. Rendre le script exécutable (uniquement pour l'utilisateur root pour protéger vos tokens) :
`chmod 700 /root/scripts/mastodon-cleanup.sh`
2. Ajouter à la crontab (`crontab -e`) pour une exécution automatique le dimanche à 3h00 :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

```

---

## 4. Conseils supplémentaires

**Libération de la RAM :** La commande `drop_caches` est incluse dans un test de permission. Si votre Mastodon tourne dans un conteneur LXC non privilégié sur Proxmox, le script ignorera simplement cette étape sans planter, ce qui permet de garantir l'envoi de la notification Gotify finale.

## 5. Captures

![maintenance-mastodon-docker.jpg](/maintenance-mastodon-docker/maintenance-mastodon-docker.jpg)

![maintenance-mastodon-docker-02.jpg](/maintenance-mastodon-docker/maintenance-mastodon-docker-02.jpg)

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)