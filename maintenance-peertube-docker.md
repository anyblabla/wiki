---
title: Maintenance et nettoyage de PeerTube sous Docker
description: Maintenance de PeerTube sous Docker : automatisation du nettoyage du stockage, des fichiers distants et optimisation RAM avec notifications Gotify optionnelles.
published: true
date: 2025-12-29T13:20:54.909Z
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

| Action | Commande officielle |
| --- | --- |
| **Prune (fichiers orphelins)** | `docker exec -u peertube peertube-peertube-1 npm run prune-storage` |
| **Nettoyage fichiers distants** | `docker exec -u peertube peertube-peertube-1 npm run house-keeping -- --delete-remote-files` |
| **Régénérer les miniatures** | `docker exec -u peertube peertube-peertube-1 npm run regenerate-thumbnails` |

---

## 3. Automatisation par script (avec Gotify optionnel)

Ce script automatise les tâches de nettoyage recommandées et peut vous envoyer une notification via **Gotify**.

> 🔴 **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides. Le script détectera l'absence de configuration et ignorera l'envoi des notifications sans générer d'erreur.

### Contenu du script : `peertube-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance PeerTube pour Docker
# S'aligne sur les outils officiels (Server tools) de PeerTube >= 6.2
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
CONTAINER_NAME="peertube-peertube-1"
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
    send_gotify_notification "⚠️ PeerTube Cleanup ALERTE" "Échec partiel du nettoyage (Prune) sur $HOSTNAME." 5
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
    echo "Note : Droits insuffisants pour drop_caches (LXC), l'hôte gérera le cache."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 5. Envoi de la notification de succès final
send_gotify_notification "✅ PeerTube Cleanup TERMINÉ" "La maintenance PeerTube sur $HOSTNAME est terminée avec succès." 4

exit 0

```

---

## 4. Planification et Conseils

1. **Sécuriser le script :** `chmod 700 /root/scripts/peertube-cleanup.sh`
2. **Automatiser (Crontab) :** Ajoutez cette ligne à votre `crontab -e` :
`00 04 * * 0 /bin/bash /root/scripts/peertube-cleanup.sh >> /var/log/peertube-cleanup.log 2>&1`

* **Stockage :** Si vous fédérez beaucoup d'instances, surveillez votre dossier `./docker-volume/data`.
* **Redondance :** Pensez à limiter l'espace alloué aux vidéos des autres instances dans l'interface d'administration de PeerTube (Configuration > VOD > Redondance).
* **Installation classique :** Pour ceux qui n'utilisent pas Docker, consultez la page dédiée : [Maintenance PeerTube (installation classique)](https://wiki.blablalinux.be/fr/maintenance-peertube-installation-classique).

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)