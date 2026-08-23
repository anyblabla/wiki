---
title: Maintenance et nettoyage de PeerTube sous Docker
description: Maintenance de PeerTube sous Docker : automatisation du nettoyage du stockage, des fichiers distants et optimisation RAM avec notifications Gotify optionnelles.
published: true
date: 2026-08-23T13:37:52.734Z
tags: docker, lxc, proxmox, gotify, linux, maintenance, peertube
editor: markdown
dateCreated: 2025-12-26T16:55:44.444Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations manuelles sont nécessaires pour supprimer les résidus de transcodage ou les fichiers temporaires qui finissent par saturer l'espace disque.

> 💡 **Pourquoi ce script ?**
> PeerTube possède son propre système de rétention, mais il fournit également des outils de maintenance officiels (accessibles via `npm run`) pour les tâches lourdes ou spécifiques. Mon script ne remplace pas le code des développeurs : il automatise simplement le lancement de ces outils internes à des heures creuses. C'est un complément d'administration pour garder une machine propre et réactive sans intervention manuelle.

---

## 1. Identifier votre conteneur PeerTube

Dans la configuration Docker par défaut, PeerTube n'a pas toujours un nom fixe. Il est souvent nommé selon votre dossier (généralement `peertube-peertube-1`).

Pour connaître le nom exact sur votre système, lancez :

```bash
docker ps --format "{{.Names}}" | grep peertube
```

> ⚠️ Dans la suite de ce guide et dans le script, j'utiliserai le nom **`peertube-peertube-1`**. Si vous avez personnalisé votre fichier `docker-compose.yml` avec un `container_name: peertube`, pensez à adapter ce nom dans le script ci-dessous.

---

## 2. Commandes de nettoyage manuel

Voici les commandes officielles pour un nettoyage ponctuel. Elles s'exécutent via `docker exec`.

> 🔴 **Piège à connaître : la confirmation interactive**
> `prune-storage` et `house-keeping --delete-remote-files` demandent tous les deux une confirmation `(y/N)` avant de supprimer quoi que ce soit. Si vous lancez ces commandes **sans le flag `-i`** sur `docker exec`, le stdin n'est pas connecté au terminal : la prompt ne reçoit jamais de réponse, et rien n'est supprimé — sans le moindre message d'erreur. C'est un piège classique qui passe totalement inaperçu si vous ne vérifiez jamais la sortie en détail, puisque les scripts se terminent avec un code de sortie 0 (succès) malgré tout.
>
> En exécution manuelle, `-i` suffit : vous répondez `y` vous-même. En exécution automatisée (cron), il faut en plus piper une réponse automatique avec `yes`.

| Action | Commande officielle (manuelle, interactive) |
| --- | --- |
| **Prune (fichiers orphelins)** | `docker exec -i -u peertube peertube-peertube-1 npm run prune-storage` |
| **Nettoyage fichiers distants** | `docker exec -i -u peertube peertube-peertube-1 npm run house-keeping -- --delete-remote-files` |
| **Régénérer les miniatures** | `docker exec -u peertube peertube-peertube-1 npm run regenerate-thumbnails` |

> ℹ️ Après un `house-keeping --delete-remote-files`, PeerTube recommande explicitement de redémarrer le conteneur : `docker restart peertube-peertube-1`.

---

## 3. Automatisation par script (avec Gotify optionnel)

Ce script automatise les tâches de nettoyage recommandées, répond automatiquement à leurs confirmations interactives, redémarre le conteneur si nécessaire, et peut vous envoyer une notification détaillée via **Gotify** (nombre de fichiers supprimés + espace disque libéré).

> 🔴 **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides. Le script détectera l'absence de configuration et ignorera l'envoi des notifications sans générer d'erreur.

### Contenu du script : `peertube-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance PeerTube pour Docker
# S'aligne sur les outils officiels (Server tools) de PeerTube >= 6.2
# Auteur : Amaury aka BlablaLinux
# v1.2 : notification Gotify enrichie (nb de fichiers supprimés + espace libéré)
# v1.1 : fix confirmation interactive (y/N) jamais reçue par les scripts
#        prune-storage / house-keeping quand lancés via docker exec sans -i

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
CONTAINER_NAME="peertube-peertube-1"
LOGFILE="/var/log/peertube-cleanup.log"
HOSTNAME=$(hostname)
# Dossier cache PeerTube, pour mesurer l'espace libéré avant/après
# Adaptez ce chemin à l'emplacement réel de votre docker-compose.yml
CACHE_DIR="/root/peertube/docker-volume/data/cache"

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

# 1. Vérification de la présence du conteneur
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
    MSG="Erreur : Le conteneur $CONTAINER_NAME est introuvable."
    echo "$MSG"
    send_gotify_notification "❌ PeerTube Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
fi

# Mesure de la taille du cache avant nettoyage (en octets)
CACHE_BEFORE=$(du -sb "$CACHE_DIR" 2>/dev/null | cut -f1)

# 2. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
# -i sur docker exec + "yes" pour répondre automatiquement "y" à la confirmation
# interactive du script (sans -i, le stdin recevait un EOF immédiat => réponse
# implicite "non" => rien n'était jamais supprimé, silencieusement)
echo "--- Étape 1 : Nettoyage du stockage (Prune) ---"
OUTPUT_PRUNE=$(yes | docker exec -i -u peertube $CONTAINER_NAME npm run prune-storage 2>&1)
PRUNE_EXIT=$?
echo "$OUTPUT_PRUNE"
if [ $PRUNE_EXIT -ne 0 ]; then
    send_gotify_notification "⚠️ PeerTube Cleanup ALERTE" "Échec du nettoyage du stockage sur $HOSTNAME." 5
fi

# Extraction du nombre de fichiers filesystem supprimés
# -m 1 : ne garder que la première occurrence (la ligne peut apparaître
# deux fois dans la sortie : une fois en "?" question, une fois en "✔" réponse)
PRUNE_FS_COUNT=$(echo "$OUTPUT_PRUNE" | grep -m 1 -oP '^\d+(?= filesystem files deleted)')
[ -z "$PRUNE_FS_COUNT" ] && PRUNE_FS_COUNT="0"

# 3. Suppression des fichiers distants (vignettes, avatars d'autres instances)
echo "--- Étape 2 : Nettoyage des fichiers distants ---"
OUTPUT_HK=$(yes | docker exec -i -u peertube $CONTAINER_NAME npm run house-keeping -- --delete-remote-files 2>&1)
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
    send_gotify_notification "⚠️ PeerTube Cleanup ALERTE" "Échec du nettoyage des fichiers distants sur $HOSTNAME." 5
else
    # PeerTube recommande de redémarrer le conteneur après un run de
    # house-keeping --delete-remote-files
    echo "Redémarrage du conteneur $CONTAINER_NAME suite au house-keeping..."
    docker restart $CONTAINER_NAME > /dev/null 2>&1
    if [ $? -ne 0 ]; then
        send_gotify_notification "⚠️ PeerTube Cleanup ALERTE" "Échec du redémarrage de $CONTAINER_NAME après house-keeping sur $HOSTNAME." 5
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

# 4. Optimisation RAM (Protection LXC)
echo "--- Étape 3 : Libération du cache RAM ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
    echo 3 > /proc/sys/vm/drop_caches
    echo "Cache RAM libéré avec succès."
else
    echo "Note : Droits insuffisants pour drop_caches (LXC), l'hôte gérera le cache."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 5. Envoi de la notification de succès final, enrichie de statistiques
if [ "$HK_SUCCESS" -ge 1 ]; then
    HK_SUMMARY="${HK_THUMBS:-0} thumbnails, ${HK_AVATARS:-0} avatars/banners, ${HK_CAPTIONS:-0} captions, ${HK_STORYBOARDS:-0} storyboards"
else
    HK_SUMMARY="aucun fichier distant supprimé (ou déjà à jour)"
fi

DETAIL_MSG="Prune : ${PRUNE_FS_COUNT} fichiers orphelins supprimés.
House-keeping : ${HK_SUMMARY}.
Espace cache libéré : ${CACHE_FREED_HUMAN}."

send_gotify_notification "✅ PeerTube Cleanup TERMINÉ" "$DETAIL_MSG" 4

exit 0
```

---

## 4. Planification et Conseils

1. **Sécuriser le script :** `chmod 700 /root/scripts/peertube-cleanup.sh`
2. **Automatiser (Crontab) :** Ajoutez cette ligne à votre `crontab -e` :
`00 04 * * 0 /bin/bash /root/scripts/peertube-cleanup.sh >> /var/log/peertube-cleanup.log 2>&1`
3. **Premier lancement :** faites toujours un premier test manuel (`sudo bash /root/scripts/peertube-cleanup.sh`) avant de compter sur le cron. Le tout premier run peut supprimer un très grand nombre de fichiers si votre instance tourne depuis longtemps sans nettoyage régulier — c'est normal, les runs suivants seront bien plus légers.

* **Stockage :** Si vous fédérez beaucoup d'instances, surveillez votre dossier `./docker-volume/data`.
* **Redondance :** Pensez à limiter l'espace alloué aux vidéos des autres instances dans l'interface d'administration de PeerTube (Configuration > VOD > Redondance).
* **Installation classique :** Pour ceux qui n'utilisent pas Docker, consultez la page dédiée : [Maintenance PeerTube (installation classique)](https://wiki.blablalinux.be/fr/maintenance-peertube-installation-classique).

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)