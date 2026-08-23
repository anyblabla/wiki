---
title: Maintenance PeerTube (installation classique)
description: Maintenance de PeerTube en installation classique (bare-metal) : automatisation du nettoyage du stockage, des fichiers distants et optimisation RAM avec notifications Gotify.
published: true
date: 2026-08-23T13:43:15.180Z
tags: serveur, debian, script, gotify, administration système, maintenance, peertube, logiciel libre
editor: markdown
dateCreated: 2025-12-28T18:39:33.530Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations manuelles sont nécessaires pour supprimer les résidus de transcodage ou les fichiers temporaires qui finissent par saturer l'espace disque.

> 💡 **Pourquoi ce script ?**
> Sur une installation classique, il est crucial de lancer les commandes avec l'utilisateur `peertube` et de charger les bonnes variables d'environnement. Ce script automatise ces tâches répétitives pour maintenir votre instance propre sans risque d'erreur de permissions ou d'oubli de paramètres.

---

## 1. Prérequis : chemins par défaut

Ce guide considère que votre installation suit la structure standard de la documentation officielle :

* **Dossier de l'instance :** `/var/www/peertube/peertube-latest`
* **Dossier de configuration :** `/var/www/peertube/config`
* **Dossier de stockage/cache :** `/var/www/peertube/storage/cache`
* **Utilisateur système :** `peertube`
* **Service systemd :** `peertube`

---

## 2. Commandes de nettoyage manuel

Ces commandes doivent être lancées depuis le dossier `peertube-latest` pour fonctionner correctement.

> 🔴 **Piège à connaître : la confirmation interactive**
> `prune-storage` et `house-keeping --delete-remote-files` demandent tous les deux une confirmation `(y/N)` avant de supprimer quoi que ce soit. En ligne de commande manuelle, ce n'est pas un problème : vous répondez vous-même. Mais **en tâche cron**, il n'y a pas de terminal (TTY) : la prompt ne reçoit jamais de réponse et rien n'est supprimé — sans le moindre message d'erreur, et avec un code de sortie 0 (succès) malgré tout. C'est un piège qui peut passer inaperçu pendant des mois si vous ne vérifiez jamais le détail des logs.
>
> Pour une exécution automatisée, il faut piper une réponse automatique avec `yes`.

| Action | Commande officielle (manuelle, interactive) |
| --- | --- |
| **Prune (fichiers orphelins)** | `sudo -u peertube NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run prune-storage` |
| **Nettoyage fichiers distants** | `sudo -u peertube NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run house-keeping -- --delete-remote-files` |
| **Régénérer les miniatures** | `sudo -u peertube NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run regenerate-thumbnails` |

> ℹ️ Après un `house-keeping --delete-remote-files`, PeerTube recommande explicitement de redémarrer le service : `sudo systemctl restart peertube`.

---

## 3. Automatisation par script (avec Gotify optionnel)

Ce script centralise les commandes de maintenance, répond automatiquement à leurs confirmations interactives, redémarre le service si nécessaire, et peut vous informer via **Gotify** avec un résumé détaillé (nombre de fichiers supprimés + espace disque libéré).

> 🔴 **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides. Le script détectera l'absence de configuration et ignorera l'envoi des notifications sans générer d'erreur.

### Contenu du script : `peertube-cleanup-classic.sh`

```bash
#!/bin/bash
# Script de maintenance PeerTube (installation classique)
# S'aligne sur les outils officiels (Server tools) de PeerTube >= 6.2
# Auteur : Amaury aka BlablaLinux
# v1.2 : notification Gotify enrichie (nb de fichiers supprimés + espace libéré)
# v1.1 : fix confirmation interactive (y/N) jamais reçue par les scripts
#        prune-storage / house-keeping quand lancés en cron (pas de TTY)

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
PT_DIR="/var/www/peertube/peertube-latest"
export NODE_CONFIG_DIR="/var/www/peertube/config"
export NODE_ENV="production"
LOGFILE="/var/log/peertube-cleanup-classic.log"
HOSTNAME=$(hostname)
# Dossier cache PeerTube, pour mesurer l'espace libéré avant/après
# Adaptez ce chemin si votre storage.cache pointe ailleurs dans production.yaml
CACHE_DIR="/var/www/peertube/storage/cache"
# Nom du service systemd, pour le redémarrage post house-keeping
SERVICE_NAME="peertube"

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

# --- FONCTION : conversion octets -> format humain (Ko/Mo/Go) ---
human_size() {
    local bytes="$1"
    if [ -z "$bytes" ] || ! [[ "$bytes" =~ ^[0-9]+$ ]]; then
        echo "N/A"
        return
    fi
    numfmt --to=iec --suffix=B "$bytes" 2>/dev/null || echo "${bytes} octets"
}

echo "======================================================"
echo "Début de la maintenance PeerTube sur $HOSTNAME : $(date)"
echo "======================================================"

# On se déplace dans le dossier de l'instance
cd $PT_DIR || {
    MSG="Erreur : dossier PeerTube introuvable."
    echo "$MSG"
    send_gotify_notification "❌ PeerTube Classic Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
}

# Mesure de la taille du cache avant nettoyage (en octets)
CACHE_BEFORE=$(du -sb "$CACHE_DIR" 2>/dev/null | cut -f1)

# 1. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
# "yes" pour répondre automatiquement "y" à la confirmation interactive du
# script (sans ça, cron n'a pas de TTY, la prompt reçoit un EOF immédiat
# => réponse implicite "non" => rien n'était jamais supprimé, silencieusement)
echo "--- Étape 1 : Nettoyage du stockage (Prune) ---"
OUTPUT_PRUNE=$(yes | sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run prune-storage 2>&1)
PRUNE_EXIT=$?
echo "$OUTPUT_PRUNE"
if [ $PRUNE_EXIT -ne 0 ]; then
    send_gotify_notification "⚠️ PeerTube Classic ALERTE" "Échec partiel du nettoyage Prune sur $HOSTNAME." 5
fi

# Extraction du nombre de fichiers filesystem supprimés
# -m 1 : ne garder que la première occurrence (la ligne peut apparaître
# deux fois dans la sortie : une fois en "?" question, une fois en "✔" réponse)
PRUNE_FS_COUNT=$(echo "$OUTPUT_PRUNE" | grep -m 1 -oP '^\d+(?= filesystem files deleted)')
[ -z "$PRUNE_FS_COUNT" ] && PRUNE_FS_COUNT="0"

# 2. Suppression des fichiers distants (vignettes, avatars d'instances distantes)
echo "--- Étape 2 : Nettoyage des fichiers distants ---"
OUTPUT_HK=$(yes | sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run house-keeping -- --delete-remote-files 2>&1)
HOUSEKEEPING_EXIT=$?
echo "$OUTPUT_HK"

# Extraction des chiffres annoncés par house-keeping
# Ligne type : "31,951 thumbnails, 5,064 avatars/banners, 46 captions and 4,556 storyboards can be locally deleted."
HK_DETECT_LINE=$(echo "$OUTPUT_HK" | grep -m 1 -oP '[\d,]+ thumbnails, [\d,]+ avatars/banners, [\d,]+ captions and [\d,]+ storyboards')
HK_THUMBS=$(echo "$HK_DETECT_LINE" | grep -m 1 -oP '^[\d,]+')
HK_AVATARS=$(echo "$HK_DETECT_LINE" | grep -m 1 -oP '[\d,]+(?= avatars/banners)')
HK_CAPTIONS=$(echo "$HK_DETECT_LINE" | grep -m 1 -oP '[\d,]+(?= captions)')
HK_STORYBOARDS=$(echo "$HK_DETECT_LINE" | grep -m 1 -oP '[\d,]+(?= storyboards)')
HK_SUCCESS=$(echo "$OUTPUT_HK" | grep -c "All remote files deleted!")

if [ $HOUSEKEEPING_EXIT -ne 0 ]; then
    send_gotify_notification "⚠️ PeerTube Classic ALERTE" "Échec du nettoyage des fichiers distants sur $HOSTNAME." 5
else
    # PeerTube recommande de redémarrer le service après un run de
    # house-keeping --delete-remote-files
    echo "Redémarrage du service $SERVICE_NAME suite au house-keeping..."
    systemctl restart $SERVICE_NAME > /dev/null 2>&1
    if [ $? -ne 0 ]; then
        send_gotify_notification "⚠️ PeerTube Classic ALERTE" "Échec du redémarrage du service $SERVICE_NAME après house-keeping sur $HOSTNAME." 5
    fi
fi

# Mesure de la taille du cache après nettoyage
CACHE_AFTER=$(du -sb "$CACHE_DIR" 2>/dev/null | cut -f1)
if [[ "$CACHE_BEFORE" =~ ^[0-9]+$ ]] && [[ "$CACHE_AFTER" =~ ^[0-9]+$ ]]; then
    CACHE_FREED=$((CACHE_BEFORE - CACHE_AFTER))
    CACHE_FREED_HUMAN=$(human_size "$CACHE_FREED")
else
    CACHE_FREED_HUMAN="N/A"
fi

# 3. Optimisation RAM
echo "--- Étape 3 : Libération du cache RAM ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
    echo 3 > /proc/sys/vm/drop_caches
    echo "Cache RAM libéré avec succès."
else
    echo "Note : Droits insuffisants pour drop_caches, ignoré."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 4. Envoi de la notification de succès final, enrichie de statistiques
if [ "$HK_SUCCESS" -ge 1 ]; then
    HK_SUMMARY="${HK_THUMBS:-0} thumbnails, ${HK_AVATARS:-0} avatars/banners, ${HK_CAPTIONS:-0} captions, ${HK_STORYBOARDS:-0} storyboards"
else
    HK_SUMMARY="aucun fichier distant supprimé (ou déjà à jour)"
fi

DETAIL_MSG="Prune : ${PRUNE_FS_COUNT} fichiers orphelins supprimés.
House-keeping : ${HK_SUMMARY}.
Espace cache libéré : ${CACHE_FREED_HUMAN}."

send_gotify_notification "✅ PeerTube Classic Cleanup TERMINÉ" "$DETAIL_MSG" 4

exit 0
```

---

## 4. Conseils supplémentaires

1. **Permissions :** Ne lancez jamais ces commandes directement en tant qu'utilisateur `root` sans spécifier `sudo -u peertube`. Cela pourrait corrompre les permissions de vos fichiers de données.
2. **Sécurité du script :** Appliquez des permissions restrictives pour protéger vos tokens Gotify :
`chmod 700 /root/scripts/peertube-cleanup-classic.sh`
3. **Planification (Crontab) :** `00 04 * * 0 /bin/bash /root/scripts/peertube-cleanup-classic.sh >> /var/log/peertube-cleanup-classic.log 2>&1`
4. **Premier lancement :** faites toujours un premier test manuel (`sudo bash /root/scripts/peertube-cleanup-classic.sh`) avant de compter sur le cron. Le tout premier run peut supprimer un très grand nombre de fichiers si votre instance tourne depuis longtemps sans nettoyage régulier — c'est normal, les runs suivants seront bien plus légers.
5. **Droits sudo :** le compte qui exécute le cron (souvent `root`) doit pouvoir faire `sudo -u peertube` sans mot de passe pour que le script fonctionne de façon non-interactive (vérifiez votre configuration `/etc/sudoers` ou `visudo` si besoin).

* **Installation Docker :** Pour ceux qui utilisent Docker/docker-compose, consultez la page dédiée : [Maintenance PeerTube (Docker)](https://wiki.blablalinux.be/fr/maintenance-peertube-docker).

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)